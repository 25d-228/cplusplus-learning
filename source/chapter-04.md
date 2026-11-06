# Chapter 4 — Errors and failure-safe state changes

A replacement can fail after it has acquired **some**, but not all, of the
resources it needs. Releasing those temporary resources is only half the
problem. The caller may still need the old value, its associated metadata, and
its existing read-only view to remain usable.

We will build an owned numeric record buffer with two arrays: record values and
cached prefix totals. The initial value is `{4, 5}`, whose total is 9. A proposed
replacement is `{8, 9, 10}`, whose total is 27. We will deliberately fail the
**second allocation within that replacement**, after the new value array exists
but before its prefix array exists. The old buffer must still contain `{4, 5}`;
the incomplete candidate must release its one acquired array. A subsequent
attempt will succeed without special repair code in the caller.

The starting workspace is the restored code from
[Chapter 3](https://25d-228.github.io/cplusplus-learning/chapter-03.html), including
all earlier CMake targets. You need Chapter 2's arrays, spans, and references,
and Chapter 3's member initialization, unique ownership, and destruction rules.
This chapter adds an independent executable. It does **not** include a `.cpp`
containing another `main`, extract a hidden header, or modify `ByteBuffer`.

> **Learning objectives**
>
> Distinguish absence from invalid input and resource exhaustion; follow an
> exception through construction and stack unwinding; define the state protected
> by a failure guarantee; build and validate a candidate before nonthrowing
> commit; inject allocation failure without exhausting the machine; distinguish
> memory cleanup from completion of external I/O.

The allocation remains **2.0 implementation hours**: 0.5 for error handling,
1.0 for the replacement mechanism, and 0.5 for failure checks and diagnosis.
Reading is separate. This is a foundation exercise in compound ownership, not
an implementation of the engine's persistent records or transaction protocol.

## One file and an explicit failure boundary

Create the directory in the existing learner workspace. Retain the compatible
native compiler selected in Chapter 1; the following selects Apple's compiler
when that is your working choice:

```sh title="Terminal — prepare the exercise"
CXX="$(xcrun --find clang++)"
mkdir -p exercises/chapter-04
```

Every C++ block labelled **`exercises/chapter-04/transactional_buffer.cpp`**, in
reading order, appends to that one file. Together they form its complete
contents. The CMake block appends a target to the existing root file. The later
block labelled **Temporary diagnostic edit** replaces one function only while
testing a weaker guarantee; restore it before continuing to the next chapter.

There is no new package dependency or newer language mode. `std::optional` is
part of the C++20 library baseline, as are the exception and ownership facilities
used here. We use standard library templates, but do not yet write a generic
allocator or our own function template.

Start the file with all required headers and the active check helper:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
#include <array>
#include <cstddef>
#include <cstdint>
#include <exception>
#include <iostream>
#include <limits>
#include <memory>
#include <new>
#include <optional>
#include <span>
#include <stdexcept>
#include <utility>

namespace strata {

bool check(bool condition, const char* name) {
    if (condition) {
        return true;
    }
    std::cerr << "FAIL " << name << '\n';
    return false;
}

constexpr std::size_t max_records = 1024;
```

`<new>` declares `std::bad_alloc`; `<stdexcept>` supplies
`std::invalid_argument`; `<exception>` declares their common standard exception
interface. `<optional>` supplies a value that can be absent. These headers are
explicit: do not depend on another library header happening to include them.
`max_records` is a compile-time constant for this small fixture, not a limit on
the eventual database.

## Give each failure a meaning

A missing record is not an allocation failure. An invalid argument is not an
uncertain disk write. Begin with the semantic cases before choosing syntax:

| Situation | Meaning in this chapter | Representation |
|---|---|---|
| No record at an index | The buffer is valid; that position has no value | Empty `std::optional<std::uint64_t>` |
| Too many records, or an unrepresentable total | The proposed buffer is outside the accepted input domain | `std::invalid_argument`, translated at the replacement boundary |
| Candidate allocation cannot be obtained | The request is valid enough to reach allocation, but storage is unavailable | `std::bad_alloc`, translated at the replacement boundary |
| External I/O fails | A different boundary must report the operation and what may already have happened | Not implemented by this memory-only exercise |

The caller must still supply a valid readable span throughout its use, and must
not concurrently mutate the source or destination. Exceptions do not validate a
dangling pointer, repair an out-of-bounds access, or synchronize a data race.
Checking the span's count cannot prove that its address names live elements.

For this operation, **the old state** means the destination's values, prefix
totals, size, and owned-array identities. Failure must preserve all of them.
Diagnostic counters and the failure injector are deliberately outside that
state: they must change to record the attempt. Temporary resource acquisition is
also allowed, provided those resources are released before the failure reaches
the caller.

It helps to distinguish three guarantees:

| Guarantee | What the caller can conclude after failure |
|---|---|
| Basic guarantee | Invariants still hold and owned resources do not leak, but the value may have changed |
| Strong guarantee for this replacement | The old destination state remains intact; temporary resources are gone |
| Nonthrowing operation | No exception escapes the operation; its specified state transition still has to be correct |

The strong guarantee is not “nothing happened anywhere.” Nor is `noexcept` a
rollback mechanism. The replacement itself is allowed to throw; its commit and
cleanup steps will be nonthrowing. We prove the guarantee from their ordering.

## An exception changes the control path

`throw std::bad_alloc{};` constructs an exception object of type
`std::bad_alloc` and transfers control toward a matching handler. `throw
std::invalid_argument("too many records");` supplies a different type and a
message. The type, not a string comparison, selects a `catch` handler. These
standard exception types are specified under
[allocation errors](https://eel.is/c++draft/bad.alloc) and
[invalid arguments](https://eel.is/c++draft/invalid.argument).

The block after `try` is the region whose escaping exceptions its handlers can
receive. A handler such as `catch (const std::bad_alloc&)` receives a read-only
reference to the exception object. The omitted parameter name is intentional:
we only need the type. A named handler parameter can call `error.what()` to
obtain a diagnostic message while the exception object is alive.

A handler does not resume at the failed line. When its block finishes normally,
execution continues after the entire `try`/`catch` statement. Statements after
the throwing operation inside the `try` block were skipped. A function that
cannot handle the failure can let it propagate; it need not write a `catch`
just to perform resource cleanup.

Before entering a matching handler, C++ destroys completed automatic objects
whose scopes the exception leaves. This is **stack unwinding**. It runs their
ordinary destructors; it does not restore old values in arbitrary variables.
A separately allocated array is released only because a destroyed owner releases
it. The language does not search for every raw allocation. The
[throwing rules](https://eel.is/c++draft/except.throw),
[handler matching](https://eel.is/c++draft/except.handle), and
[unwinding rules](https://eel.is/c++draft/except.ctor) define these boundaries.

Our `RecordBuffer` will contain owning members. If construction fails, completed
members are destroyed even though the enclosing buffer was never completely
constructed. That is why responsibility must enter a member owner immediately,
not wait for the enclosing object's destructor to notice an unfinished job.

## An optional value is not a hidden error code

`std::optional<T>` stores either a `T` or no `T`. The optional itself tracks which
case holds. For `std::optional<std::uint64_t>`, the integer storage is part of the
optional; using it does not require a separate heap allocation. An optional of
some other type may contain a type that allocates its own resources. The
[optional object model](https://eel.is/c++draft/optional.general) describes that
separation.

`std::nullopt` denotes the no-value case. `has_value()` and a condition such as
`if (result)` test **presence**, not the numerical truth of the contained value.
An optional containing zero is present. `*result` accesses the contained value
and requires that it exists. Short-circuiting `result.has_value() && *result == 0`
checks that precondition first. The checked `result.value()` alternative throws
`std::bad_optional_access` when no value exists; we will not use that as normal
lookup control flow. See the
[optional observers](https://eel.is/c++draft/optional.observe).

The buffer's `value_at(index)` will return a copied optional integer. Keeping
that result does not borrow the buffer. Its `values()` method will instead
return a read-only span, with the lifetime rules from Chapter 2. The return type
makes that ownership distinction visible.

We will also use an optional in the test injector: no value means “not armed,”
and a present positive count means “fail after this many allocation attempts.”
That is absence of a configured action, not absence of a record. The type alone
does not choose the domain meaning; the enclosing interface does.

## Fail an allocation by its position, not by consuming all RAM

The failure **seam** is a small explicit call immediately before each tracked
payload allocation. Arming it with 2 permits the next attempt, then throws on
the following one. It counts from the moment it is armed, regardless of earlier
successful allocations. No global allocator is replaced and no large memory
request is needed to provoke the failure.

Append the complete probe:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
struct AllocationProbe {
    std::size_t attempts = 0;
    std::size_t acquired = 0;
    std::size_t released = 0;
    std::size_t live = 0;
    std::size_t peak_live = 0;
    std::optional<std::size_t> countdown{};

    void fail_on(std::size_t ordinal) {
        if (ordinal == 0) {
            throw std::invalid_argument("allocation ordinal starts at one");
        }
        countdown = ordinal;
    }

    void before_allocation() {
        ++attempts;
        if (!countdown) {
            return;
        }
        --*countdown;
        if (*countdown == 0) {
            countdown.reset();
            throw std::bad_alloc{};
        }
    }

    void disarm() noexcept {
        countdown.reset();
    }
};
```

`countdown = ordinal` gives the optional a value. `--*countdown` decreases the
contained integer, not the optional's address. The preceding presence check
makes dereferencing legal. `fail_on` rejects zero, so decrementing an armed
count cannot wrap from zero. Once the count reaches zero, `reset()` removes it
**before** the exception is thrown. Thus the injected failure is one-shot.

`attempts` increases even for an attempt deliberately failed before allocation.
`acquired` will increase only after an actual array allocation succeeds.
`released` will increase when such an array is relinquished. These are separate
quantities. Treating a failed attempt as an acquisition would conceal a leak or
force a fictitious release.

The probe is finite, single-threaded test instrumentation. Its counters are not
atomic, not overflow-hardened telemetry, and not a process-memory budget. It
counts neither the exception runtime's own storage nor allocations performed by
streams or exception messages. An ordinary allocation can still fail on its own;
only the explicitly injected failure automatically clears this countdown.
`disarm()` is available when ending another experiment deliberately.

The probe's lifetime is a precondition: it must outlive every array whose
release it records. Each test will declare the probe outside its owners' scopes.
Copying the probe into `const auto before` later produces a value snapshot of its
counters and countdown. Owners still point to the original probe, not that copy.

## Put accounting next to the owning pointer

Two arrays need two independent cleanup obligations. `WordArray` will own one
`std::uint64_t[]` and borrow the probe that accounts for it. It has no public
length or indexing contract; the enclosing `RecordBuffer` supplies those. This
small helper is specific to the exercise, not a replacement for `std::vector`.

We need to exchange resources, not move or copy these helper objects into other
containers. Copying and moving them are therefore explicitly disabled here.
An object can still expose a member `swap` that exchanges its private resources.
This differs from the general `std::swap(object_a, object_b)` algorithm, which
normally requires move construction and move assignment. Chapter 3's move-only
owner and all its tests remain unchanged.

Append the helper:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
class WordArray {
public:
    WordArray() noexcept = default;

    WordArray(std::size_t count, AllocationProbe& probe) : probe_(&probe) {
        if (count == 0) {
            return;
        }
        probe.before_allocation();
        data_ = std::make_unique<std::uint64_t[]>(count);
        ++probe.acquired;
        ++probe.live;
        if (probe.live > probe.peak_live) {
            probe.peak_live = probe.live;
        }
    }

    ~WordArray() noexcept {
        if (data_) {
            data_.reset();
            ++probe_->released;
            --probe_->live;
        }
    }

    WordArray(const WordArray&) = delete;
    WordArray& operator=(const WordArray&) = delete;
    WordArray(WordArray&&) = delete;
    WordArray& operator=(WordArray&&) = delete;

    std::uint64_t* data() noexcept { return data_.get(); }
    const std::uint64_t* data() const noexcept { return data_.get(); }

    void swap(WordArray& other) noexcept {
        data_.swap(other.data_);
        std::swap(probe_, other.probe_);
    }

private:
    std::unique_ptr<std::uint64_t[]> data_{};
    AllocationProbe* probe_ = nullptr;
};
```

The constructor first records which probe will account for the resource. A
zero-length request performs no allocation and consumes no failure countdown.
For a nonempty request, `before_allocation()` runs immediately before
`std::make_unique<std::uint64_t[]>(count)`. That array form creates initialized
integer elements and returns their unique owner; see
[unique-pointer creation](https://eel.is/c++draft/unique.ptr.create).

Only after the unique pointer owns the array do the acquisition counters change.
If the seam or allocation throws, the successful-acquisition counters have not
changed. The already-constructed empty unique-pointer member is still cleaned
up as construction unwinds. There is no raw pointer waiting for manual rescue.

The destructor calls `data_.reset()` to release the array, then updates its
probe. The subsequent automatic destruction of the now-empty unique pointer
releases nothing again; see the [body/member destruction order](https://eel.is/c++draft/class.dtor). There is no stream output, new allocation, or throwing
element destructor on this path. Integer-array destruction and the
[unique-pointer reset contract](https://eel.is/c++draft/unique.ptr.runtime.modifiers)
are sufficient for this cleanup policy.

`probe_->released` means access the member of the pointed-to probe, equivalently
`(*probe_).released`. A nonempty `data_` must always have its live probe attached.
The default empty object has neither. `data_.swap(other.data_)` exchanges the
owning pointers without copying their arrays. We must also swap `probe_`:
accounting responsibility follows the allocation, not the helper object's old
address. Neither pointer exchange allocates. The
[unique-pointer swap contract](https://eel.is/c++draft/unique.ptr.single.modifiers)
and [scalar swap contract](https://eel.is/c++draft/utility.swap) support the
nonthrowing declaration for these particular member types and default deleters.

These exchanges are not a concurrency protocol. Between their individual
statements, another thread must not observe the helper. The same exclusive-use
rule will apply to the buffer's larger commit.

## Values and prefix totals form one invariant

Let $n$ be the number of records and $v_i$ the unsigned 64-bit value at zero-based
index $i$, for $0 \le i < n$. For a nonempty buffer, the value array has shape
$[n]$ and its prefix array has shape $[n+1]$. Define $P_k$ as the sum of the first
$k$ values, with the empty prefix explicitly represented:

$$
P_0 = 0, \qquad P_{i+1} = P_i + v_i.
$$

For the proposed `{8, 9, 10}` input, the construction loop produces:

| Loop index $i$ | Read $v_i$ | Existing $P_i$ | Write $P_{i+1}$ |
|---|---:|---:|---:|
| 0 | 8 | 0 | 8 |
| 1 | 9 | 8 | 17 |
| 2 | 10 | 17 | 27 |

The two arrays are `{8, 9, 10}` and `{0, 8, 17, 27}`. Keeping a prefix array makes
a prefix-total request a single indexed read rather than a fresh loop. It also
creates a meaningful compound invariant: replacing the values without replacing
the prefixes could make the same object report contradictory answers.

The accepted domain is at most `max_records` entries, with every prefix total
representable in `std::uint64_t`. Let $U = 2^{64}-1$, the largest such value.
All inputs are nonnegative, so a partial total never needs to decrease. Before
an addition, rearrange the mathematical bound:

$$
P_i + v_i \le U \quad\Longleftrightarrow\quad v_i \le U - P_i.
$$

The loop invariant already gives $0 \le P_i \le U$, so the right-side
subtraction is representable. We test that right side **before** adding.
For input `{U, 1}`, the first prefix becomes $U$. At the next step, $U-P_1=0$,
so the value 1 is rejected. `{U, 0}` is valid: its second prefix is still $U$.

Unsigned addition in C++ is modular; it does not raise an arithmetic exception
when the mathematical result exceeds the type's range. See
[unsigned integer arithmetic](https://eel.is/c++draft/basic.fundamental).
A post-addition comparison against `max()` would be too late: the stored result
would already be a representable, wrapped integer. Our exception is an explicit
application-level rejection, not a hardware overflow trap.

For zero records we use **no arrays**. The logical prefix total at count zero
is still zero; the accessor will return that constant without dereferencing a
null pointer. At completed public operations, a nonempty buffer owns both
arrays with matching sizes and valid prefixes, or an empty buffer owns neither.

Append the complete buffer declaration:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
class RecordBuffer {
public:
    RecordBuffer() noexcept = default;
    RecordBuffer(std::span<const std::uint64_t> input, AllocationProbe& probe);
    ~RecordBuffer() noexcept = default;

    RecordBuffer(const RecordBuffer&) = delete;
    RecordBuffer& operator=(const RecordBuffer&) = delete;
    RecordBuffer(RecordBuffer&&) = delete;
    RecordBuffer& operator=(RecordBuffer&&) = delete;

    std::size_t size() const noexcept;
    std::span<const std::uint64_t> values() const & noexcept;
    std::span<const std::uint64_t> values() const && = delete;
    std::optional<std::uint64_t> value_at(std::size_t index) const noexcept;
    std::optional<std::uint64_t> prefix_total(std::size_t count) const noexcept;

    void swap(RecordBuffer& other) noexcept;
    void replace(std::span<const std::uint64_t> input, AllocationProbe& probe);

private:
    static std::size_t checked_size(std::size_t count);

    std::size_t size_ = 0;
    WordArray values_{};
    WordArray prefixes_{};
};
```

Only read-only access to the stored values escapes the class. A mutable span
would let a caller alter `v_i` without repairing later prefixes. `value_at` and
`prefix_total` return copied values, so they do not need borrow restrictions.
`values()` returns a borrowed span and rejects an rvalue receiver just as Chapter
3's view API did. Those ref qualifiers still do not supply full borrow checking.

`size_` is declared **before** the two owners. Its checked initialization will
therefore finish before either array requests storage, following the
[member-initialization order](https://eel.is/c++draft/class.base.init). `values_` is declared
before `prefixes_`, so it acquires the first allocation. This member order is
part of the failure experiment, not incidental formatting.

We deliberately make this aggregate owner noncopyable and nonmovable, while
providing the exact exchange operation it needs. No object is returned by value
or relocated in this exercise. Adding movement later would require preserving
the coupled empty-state invariant, as Chapter 3 taught; it is not needed to
establish replacement safety now.

## Construct and validate the candidate completely

Append the size check and constructor:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
std::size_t RecordBuffer::checked_size(std::size_t count) {
    if (count > max_records) {
        throw std::invalid_argument("too many records");
    }
    return count;
}

RecordBuffer::RecordBuffer(std::span<const std::uint64_t> input,
                           AllocationProbe& probe)
    : size_(checked_size(input.size())),
      values_(size_, probe),
      prefixes_(size_ == 0 ? 0 : size_ + 1, probe) {
    if (size_ == 0) {
        return;
    }
    auto* totals = prefixes_.data();
    totals[0] = 0;
    const auto limit = std::numeric_limits<std::uint64_t>::max();
    for (std::size_t i = 0; i < size_; ++i) {
        const auto value = input[i];
        if (value > limit - totals[i]) {
            throw std::invalid_argument("record total overflows uint64_t");
        }
        values_.data()[i] = value;
        totals[i + 1] = totals[i] + value;
    }
}
```

`checked_size(input.size())` runs as `size_` is initialized. An oversized input
throws before either member-array constructor begins. Once the count is at most
1024, `size_ + 1` and the small allocation sizes fit the native target's count
and address-size domains. This fixture cap makes that arithmetic explicit; a
future general-purpose buffer must derive its own checked size calculations.

The conditional expression `size_ == 0 ? 0 : size_ + 1` allocates no prefix
array for the empty case. Otherwise both arrays exist before the constructor
body begins. `auto* totals` is a pointer to the owned prefix elements, not a new
allocation. The loop reads one source value, checks its addition, copies it into
the owned value array, and writes the next prefix. No borrow of the input is
retained after the constructor returns.

The candidate may have partially filled arrays when the overflow check throws.
That is harmless because it has not been published as the destination. Its
member owners already know how to release the arrays, regardless of how many
integers the validation loop reached.

Follow the four possible construction boundaries:

| Failure point | Candidate resources acquired | Cleanup before a handler can run |
|---|---|---|
| `checked_size` rejects the count | None | No candidate arrays exist |
| First array's seam or allocation fails | None | Its empty smart-pointer member is cleaned up |
| Second array's seam or allocation fails | Value array only | The completed `values_` member releases that array |
| Prefix validation in the constructor body fails | Both arrays | Completed members release prefixes first, then values |

In the last two rows, **`RecordBuffer`'s destructor is not called for the
incomplete candidate**. Its completed member subobjects are destroyed instead.
Likewise, a `WordArray` whose constructor did not finish does not run its own
destructor, though its completed members are cleaned up. The ownership must
already be in those members; adding a cleanup loop only to `~RecordBuffer`
would miss construction failure.

## Return absence without losing the distinction from zero

Append the complete read interface:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
std::size_t RecordBuffer::size() const noexcept {
    return size_;
}

std::span<const std::uint64_t> RecordBuffer::values() const & noexcept {
    return {values_.data(), size_};
}

std::optional<std::uint64_t>
RecordBuffer::value_at(std::size_t index) const noexcept {
    if (index >= size_) {
        return std::nullopt;
    }
    return values_.data()[index];
}

std::optional<std::uint64_t>
RecordBuffer::prefix_total(std::size_t count) const noexcept {
    if (count > size_) {
        return std::nullopt;
    }
    if (count == 0) {
        return std::uint64_t{0};
    }
    return prefixes_.data()[count];
}
```

For `{4, 5}`, `value_at(0)` returns a present 4, `value_at(1)` a present 5, and
`value_at(2)` an empty optional. A record whose value is zero still yields a
present result. The boundary check uses `index >= size_`, because `size_` is the
first invalid index.

A prefix *count* has a different range: zero through `size_` inclusive.
`prefix_total(2)` for this buffer returns 9, and `prefix_total(3)` is absent.
Count zero returns a present zero even for an empty buffer. The special branch
prevents reading the empty buffer's nonexistent `prefixes_[0]`.

`return values_.data()[index]` copies one integer into an optional result. It
does not return a reference to the array. By contrast, the span's address still
belongs to the owner. On a failed replacement we will preserve that allocation;
on a successful replacement it will be released before the call returns.

Using `value_or(0)` at this interface would collapse “no record” and “record
with value zero.” That is acceptable only when a caller explicitly wants that
meaning. Our checks keep presence visible instead.

## Prepare first; commit with a nonthrowing exchange

The replacement algorithm is short because the candidate's constructor carries
the preparation and validation work. Append both complete functions:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
void RecordBuffer::swap(RecordBuffer& other) noexcept {
    values_.swap(other.values_);
    prefixes_.swap(other.prefixes_);
    std::swap(size_, other.size_);
}

void RecordBuffer::replace(std::span<const std::uint64_t> input,
                           AllocationProbe& probe) {
    RecordBuffer candidate(input, probe);
    swap(candidate);
}
```

There are two stages. `RecordBuffer candidate(input, probe)` may fail, but it
operates on a separate object and touches no destination member. Once it
succeeds, `swap(candidate)` exchanges the entire representation. Array owners
and their accounting probes travel together, then the scalar count is exchanged.
All these operations are nonthrowing for the concrete types used here.

This is a call to the buffer's **member** `swap`, not the generic
`std::swap(current, candidate)`. The latter would require operations we explicitly
deleted. Only pointer and integer exchanges inside our implementation use the
standard swap utility.

After the exchange, the destination owns the prepared arrays. `candidate` now
owns the destination's **old** arrays. Its ordinary destruction at the end of
`replace` releases them. No separate undo loop and no `catch` inside `replace`
are necessary.

Let O name the pair of old arrays and N the new pair. The names describe live
resources, not particular numerical pointer values:

| Point | Destination | Candidate | Can this step throw out of `replace`? |
|---|---|---|---|
| Before preparation | Owns O: `{4,5}` and `{0,4,9}` | Not constructed | Preparation can |
| Candidate fully ready | Still owns O | Owns N: `{8,9,10}` and `{0,8,17,27}` | No more throwing work remains |
| Exchange complete | Owns N | Owns O | No |
| Function returns | Still owns N | Destroyed; O released | No |

For any failure before commit, the destination was untouched and member owners
release acquired candidate resources. After commit, no operation in this
function can throw. Those two facts prove the strong guarantee for this
implementation. Merely catching an exception and returning an error would not
prove it.

A successful replacement invalidates existing spans into O. Do not compare or
read their stored pointers afterward; end their use before the success path.
A failed replacement preserves O, so such views remain valid under the original
lifetime preconditions. A copied optional integer remains valid in either case.

Input may be a subspan of the destination's old value array. Preparation reads
all its elements while O is still owned. Only afterward does commit exchange
owners and cleanup release O. The implementation does not inspect `input` after
that point. An erase-first algorithm would lose this aliasing property as well
as its failure guarantee.

This exchange is a logical commit under **exclusive access**, not an atomic
machine instruction. The temporary mismatch between individual pointer and
count swaps must not be observed by another thread or a reentrant callback.
Concurrency will require a separately taught publication boundary.

### Keeping the old state has a memory cost

Let $w$ be `sizeof(std::uint64_t)`, and $B(n)$ the bytes occupied by the two
payload arrays for a nonempty $n$-record buffer. Ignore owner objects, allocator
metadata, and alignment overhead in this payload-only calculation:

$$
B(n) = wn + w(n+1) = w(2n+1), \qquad n > 0.
$$

For the empty representation, $B(0)=0$ rather than the nonempty formula.
The target's eight-byte words give $B(2)=8(5)=40$ bytes and $B(3)=8(7)=56$ bytes.
During a successful replacement, both old and new arrays coexist:

$$
B_{\mathrm{peak}} = B(2) + B(3) = 40 + 56 = 96\ \mathrm{bytes}.
$$

On the injected second-allocation failure, only the new three-word value array
has been acquired, so that attempt reaches $40+3(8)=64$ payload bytes. Cleanup
returns to 40. The next successful attempt peaks at 96, then retains 56 after
the old arrays are released. The probe will record **array counts**, not these
byte totals: three simultaneous arrays during the failed attempt, four during
a successful one, and two afterward.

The old/new coexistence is the cost of this particular strong-guarantee design,
not a theorem that every failure-safe algorithm must double memory. A later
bounded-memory system must reserve its preparation cost before starting, use a
different representation, or reject work. Releasing the old allocation early
just to reduce this peak silently changes the operation's guarantee.

## Translate exceptions only at a boundary that can classify them

Inside the constructor, failure stops construction by throwing. At the caller's
replacement boundary we want a small status that says what happened. A scoped
enumeration, written `enum class`, gives distinct named values. Use the scope
qualifier in `ReplaceStatus::ok`; these values do not silently become booleans
or ordinary integers. The [enumeration rules](https://eel.is/c++draft/dcl.enum)
define this behavior.

Append the vocabulary and its complete adapter:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
enum class ReplaceStatus {
    ok,
    invalid_input,
    resource_exhausted
};

[[nodiscard]] ReplaceStatus try_replace(
    RecordBuffer& destination,
    std::span<const std::uint64_t> input,
    AllocationProbe& probe) {
    try {
        destination.replace(input, probe);
        return ReplaceStatus::ok;
    } catch (const std::invalid_argument&) {
        return ReplaceStatus::invalid_input;
    } catch (const std::bad_alloc&) {
        return ReplaceStatus::resource_exhausted;
    }
}
```

The [`[[nodiscard]]` attribute](https://eel.is/c++draft/dcl.attr.nodiscard) asks
the compiler to diagnose an ignored returned result. It is
a warning aid, not a substitute for handling a status. Tests will store the
status and compare it explicitly. The attribute does not change control flow.

`try_replace` returns `ok` only if `replace` completed. By the time a matching
handler is entered, any incomplete candidate has already released its resources.
The handler therefore translates a known failure; it does not perform rollback.
`const std::invalid_argument&` and `const std::bad_alloc&` avoid copying exception
objects, and neither handler needs to keep an exception alive after returning.

There is deliberately no `catch (...)` here. An unrelated exception must not be
silently relabelled “out of memory” or “bad input.” The adapter is not declared
`noexcept`, so an unclassified exception can propagate to the program's outer
failure boundary. Nor should constructors throughout a subsystem each catch and
rethrow the same error just to print it repeatedly.

The classifications describe the exception that actually occurred. The [library exception-handling rules](https://eel.is/c++draft/res.on.exception.handling)
allow storage-allocation failure during exception-object construction. An
`invalid_argument` message may itself need storage; if that fails with
`bad_alloc`, the adapter reports resource exhaustion instead. The deterministic
probe does not interfere with such message storage. A test with controlled
payload-allocation failures is not a claim that every possible global
out-of-memory condition is recoverable.

Exceptions and statuses are not competing ownership models. Either can report
failure while RAII supplies cleanup. The useful questions are where a decision
can be made, what state remains valid, and whether the returned information lets
the caller make that decision. In this example, absent lookup values do not need
an exception or a replacement status at all.

## Observe the failed second allocation

The test helpers compare both the stored values and the cached totals. A test
that only checks the status would miss a destination accidentally cleared before
failure. A test that only checks for leaks would miss the same bug.

Append two complete predicates:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
bool same_values(const RecordBuffer& buffer,
                 std::span<const std::uint64_t> expected) noexcept {
    const auto actual = buffer.values();
    if (actual.size() != expected.size()) {
        return false;
    }
    for (std::size_t i = 0; i < actual.size(); ++i) {
        if (actual[i] != expected[i]) {
            return false;
        }
    }
    return true;
}

bool has_total(const RecordBuffer& buffer, std::size_t count,
               std::uint64_t expected) noexcept {
    const auto result = buffer.prefix_total(count);
    return result.has_value() && *result == expected;
}
```

`same_values` checks the count before indexing. `has_total` tests optional
presence before dereferencing. Neither retains a view, allocates storage, or
modifies the buffer.

The first check creates the old buffer **before** arming failure. Its two
allocations therefore do not consume the requested countdown. The following
block starts a function; the next labelled block continues that same function.

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
bool check_replacement(AllocationProbe& probe) {
    const std::array<std::uint64_t, 2> old_value{4, 5};
    const std::array<std::uint64_t, 3> new_value{8, 9, 10};
    RecordBuffer current(old_value, probe);
    const auto copied_old = current.value_at(1);
    if (!check(same_values(current, old_value) && has_total(current, 2, 9),
               "initial value and prefix")) {
        return false;
    }
    std::cout << "initial: values=4 5 total=9 live=" << probe.live << '\n';

    {
        const auto borrowed_old = current.values();
        const auto before = probe;
        probe.fail_on(2);
        const auto status = try_replace(current, new_value, probe);
        if (!check(status == ReplaceStatus::resource_exhausted &&
                   same_values(current, old_value) && has_total(current, 2, 9),
                   "second-allocation failure preserves value and prefix") ||
            !check(current.values().data() == borrowed_old.data() &&
                   borrowed_old[1] == 5, "failed replacement preserves borrow") ||
            !check(probe.attempts == before.attempts + 2 &&
                   probe.acquired == before.acquired + 1 &&
                   probe.released == before.released + 1 &&
                   probe.live == before.live && !probe.countdown,
                   "partial candidate released; failure seam disarmed")) {
            return false;
        }
        std::cout << "second allocation failed: values=4 5 total=9"
                     " candidate acquired=1 released=1 live="
                  << probe.live << '\n';
    }
```

`const auto before = probe` makes an independent value snapshot. No owner is
copied, and the snapshot's counter changes are not connected to later updates.
Arming 2 is therefore relative to the two allocation calls of the candidate,
not “the second allocation since process startup.”

The state trace is now concrete:

| Event | Successful acquisitions | Releases | Live arrays | Destination total |
|---|---:|---:|---:|---:|
| Construct `{4,5}` | 2 | 0 | 2 | 9 |
| Allocate candidate values | 3 | 0 | 3 | 9 |
| Fail candidate prefixes | 3 | 0 | 3 | 9 |
| Unwind completed candidate value owner | 3 | 1 | 2 | 9 |

The failed prefix constructor never records an acquisition, so it owes no
release of a prefix array. Its empty unique-pointer member can still be destroyed
normally. The completed `values_` member owes exactly one release and performs
it before the `resource_exhausted` result reaches this check.

The first `check` verifies the status and old logical state. Only when that
passes do we inspect the retained view into the old live allocation. This
ordering also lets the later erase-first diagnostic fail without reading its
invalidated old view. The `||` chain short-circuits at a failed check.

The extra scope ends the borrow's use before the next successful replacement.
The copied `copied_old` optional intentionally survives; it owns its integer.

## Retry, replace from an alias, and clear without allocating

Continue the open `check_replacement` function:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
    const auto retried = try_replace(current, new_value, probe);
    if (!check(retried == ReplaceStatus::ok && same_values(current, new_value) &&
               has_total(current, 0, 0) && has_total(current, 1, 8) &&
               has_total(current, 2, 17) && has_total(current, 3, 27),
               "retry installs value and every prefix") ||
        !check(copied_old.has_value() && *copied_old == 5,
               "optional result owns its copied value")) {
        return false;
    }
    std::cout << "retry: values=8 9 10 total=27 copied-old=5 live="
              << probe.live << '\n';

    const std::array<std::uint64_t, 2> suffix{9, 10};
    const auto aliased = try_replace(current, current.values().subspan(1), probe);
    if (!check(aliased == ReplaceStatus::ok && same_values(current, suffix) &&
               has_total(current, 2, 19), "input may borrow the old payload")) {
        return false;
    }
    std::cout << "aliased input: values=9 10 total=19\n";

    const auto before_clear = probe.attempts;
    probe.fail_on(1);
    const auto cleared = try_replace(current, {}, probe);
    if (!check(cleared == ReplaceStatus::ok && current.size() == 0 &&
               current.values().data() == nullptr && !current.value_at(0) &&
               has_total(current, 0, 0) && !current.prefix_total(1),
               "empty is valid; zero total is not absence") ||
        !check(probe.attempts == before_clear && probe.live == 0 &&
               probe.countdown.has_value() && *probe.countdown == 1,
               "clearing consumes no allocation attempt")) {
        return false;
    }
    probe.disarm();
    std::cout << "clear: absent record; present zero total; live=0\n";
    return true;
}
```

The injected failure disarmed itself, so the retry can attempt both allocations.
The test checks all four prefixes `{0,8,17,27}`, not only the final sum. The old
copied value remains 5 after its source array is gone. That is a copied value's
lifetime, not a surviving span.

`current.values().subspan(1)` denotes the currently owned suffix `{9,10}`.
Passing it directly keeps the source alive during preparation and avoids keeping
a named old view for accidental later use. A successful return gives a new
owned suffix with prefix totals `{0,9,19}`. The input view is not a snapshot or
an owner; this is safe because of the precise read-before-commit order.

For the clear operation, `{}` constructs an empty input span. Failure is armed
on the next allocation, yet clearing succeeds because empty construction requests
no array. Swapping with an empty candidate moves the old arrays into that
candidate, whose destructor releases them. The still-armed count confirms that
no hidden tracked allocation was consumed. We explicitly disarm it afterward
so it cannot spill into another test.

The empty buffer distinguishes absent record zero from present prefix total
zero. It also has no allocation merely to represent its empty-prefix value.

## Reject before allocation and after partial construction

A good replacement boundary must also handle invalid input, not just simulated
allocation errors. The next function first checks the cheap length rule and the
first-allocation failure. Its second block checks arithmetic validation after
both member arrays exist.

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
bool check_rejection(AllocationProbe& probe) {
    const std::array<std::uint64_t, 2> old_value{4, 5};
    const std::array<std::uint64_t, 3> new_value{8, 9, 10};
    const std::array<std::uint64_t, max_records + 1> oversized{};
    RecordBuffer current(old_value, probe);
    const auto before = probe;

    probe.fail_on(1);
    const auto too_large = try_replace(current, oversized, probe);
    if (!check(too_large == ReplaceStatus::invalid_input &&
               same_values(current, old_value) && has_total(current, 2, 9) &&
               probe.attempts == before.attempts && probe.countdown.has_value() &&
               *probe.countdown == 1, "length rejected before allocation")) {
        return false;
    }

    const auto no_storage = try_replace(current, new_value, probe);
    if (!check(no_storage == ReplaceStatus::resource_exhausted &&
               same_values(current, old_value) && has_total(current, 2, 9) &&
               probe.attempts == before.attempts + 1 &&
               probe.acquired == before.acquired &&
               probe.released == before.released && probe.live == before.live,
               "first-allocation failure acquires nothing")) {
        return false;
    }
    std::cout << "early rejection: size checked first; first allocation acquired=0\n";
```

The 1025-element fixture is a real initialized `std::array`, not a falsely sized
span over a smaller object. Length validation rejects it without reaching the
probe. The armed failure therefore remains pending and rejects the very next
valid request at its **first** allocation. That request acquires no candidate
resource and leaves the old buffer intact.

Continue the same function with the late-rejection and numeric-boundary cases:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
    const auto limit = std::numeric_limits<std::uint64_t>::max();
    const std::array<std::uint64_t, 2> overflowing{limit, 1};
    const auto before_overflow = probe;
    const auto invalid = try_replace(current, overflowing, probe);
    if (!check(invalid == ReplaceStatus::invalid_input &&
               same_values(current, old_value) && has_total(current, 2, 9),
               "validation failure preserves the old value") ||
        !check(probe.attempts == before_overflow.attempts + 2 &&
               probe.acquired == before_overflow.acquired + 2 &&
               probe.released == before_overflow.released + 2 &&
               probe.live == before_overflow.live,
               "both members released after constructor-body failure")) {
        return false;
    }
    std::cout << "late rejection: candidate acquired=2 released=2; old total=9\n";

    const std::array<std::uint64_t, 2> boundary{limit, 0};
    const auto accepted = try_replace(current, boundary, probe);
    const auto zero = current.value_at(1);
    if (!check(accepted == ReplaceStatus::ok && same_values(current, boundary) &&
               has_total(current, 1, limit) && has_total(current, 2, limit) &&
               zero.has_value() && *zero == 0 && !current.value_at(2),
               "maximum plus zero is valid; absent is different from zero")) {
        return false;
    }
    std::cout << "overflow boundary: maximum plus zero accepted; no wrapped total\n";
    return true;
}
```

For `{U,1}`, the two candidate arrays were constructed before the loop detected
that the second prefix would not fit. Two acquisitions and two releases must
therefore be observed, with the destination's two old arrays still live. The
old total remains 9 because none of the candidate's partial state was committed.

For `{U,0}`, every addition is representable and commit succeeds. The stored
zero at index 1 is present, while index 2 is absent. This makes the overflow
check and optional presence test exercise their exact boundaries rather than
only typical positive values.

This constructor chooses one copy-and-prefix pass after allocation. It could
validate all sums in a separate pass before allocating, at the cost of another
input pass; that would change the failure-order trace but can be a valid design.
The important requirement is that all potentially failing work finishes before
the destination changes, not that every design must validate at the same line.

## Move the handler outward and follow the scopes

So far, `try_replace` caught the candidate failure while the caller's `current`
object remained in scope. What happens when the caller does not use that adapter
and its whole local scope is exited instead?

Append this check of the throwing interface itself:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
bool check_unwinding(AllocationProbe& probe) {
    const auto before = probe;
    const std::array<std::uint64_t, 2> old_value{4, 5};
    const std::array<std::uint64_t, 3> new_value{8, 9, 10};
    bool caught = false;
    bool reached_after_call = false;
    try {
        RecordBuffer local(old_value, probe);
        probe.fail_on(2);
        local.replace(new_value, probe);
        reached_after_call = true;
    } catch (const std::bad_alloc&) {
        caught = true;
    }
    if (!check(caught && !reached_after_call &&
               probe.attempts == before.attempts + 4 &&
               probe.acquired == before.acquired + 3 &&
               probe.released == before.released + 3 && probe.live == before.live,
               "outer handler sees candidate and local resources released")) {
        return false;
    }
    std::cout << "unwind: caught bad_alloc; continuation skipped; live="
              << probe.live << '\n';
    return true;
}
```

Here the handler is outside the scope containing `local`. On the injected
failure, the incomplete candidate first releases its acquired value array.
Unwinding then destroys the fully constructed `local`, releasing its two old
arrays. The handler sees three acquisitions and three releases relative to its
baseline. The assignment to `reached_after_call` is skipped.

There is no contradiction with the strong guarantee. `replace` did not mutate
the old buffer, but the caller chose a control path that ended that buffer's
lifetime. Preserving an object's value on operation failure does not preserve
the object beyond its enclosing scope. If a caller needs to keep and use the old
buffer after failure, its handler must be inside that buffer's lifetime, as the
earlier status-based check was.

These cleanup paths rely on ordinary exception handling reaching a handler.
Do not generalize them to forced process termination, a power failure, or every
call to `std::terminate`. An exception escaping a `noexcept` function, or an
exception escaping destruction during active unwinding, can terminate the
program rather than produce the result we tested. With no matching handler,
unwinding before termination is not universally guaranteed. See the
[termination rules](https://eel.is/c++draft/except.terminate).

## Connect the program to the existing build

Close the namespace and add the complete entry point:

```cpp title="exercises/chapter-04/transactional_buffer.cpp"
} // namespace strata

int main() {
    strata::AllocationProbe probe;
    bool ok = false;
    try {
        ok = strata::check_replacement(probe) &&
             strata::check_rejection(probe) &&
             strata::check_unwinding(probe);
    } catch (const std::exception& error) {
        std::cerr << "UNEXPECTED " << error.what() << '\n';
    } catch (...) {
        std::cerr << "UNEXPECTED nonstandard exception\n";
    }

    const bool drained = strata::check(
        probe.live == 0 && probe.acquired == probe.released,
        "all tracked arrays released");
    if (!ok || !drained) {
        return 1;
    }
    std::cout << "final: attempts=" << probe.attempts
              << " acquired=" << probe.acquired
              << " released=" << probe.released
              << " live=" << probe.live
              << " peak-live=" << probe.peak_live << '\n';
    std::cout << "foundation-04: all checks passed\n";
    return 0;
}
```

`std::exception` is a standard base interface shared by the standard exception
classes used here. Catching it by `const` reference permits a final diagnostic
without copying a base-only object and discarding more specific type information.
`what()` supplies a message; its pointer is not retained after the handler. See
[the standard exception interface](https://eel.is/c++draft/exception).

The final `catch (...)` is actual C++ syntax for a handler matching any exception
not already matched. It belongs last. Here it ends the test program with failure;
it does not convert an unknown error into a successful test or pretend to recover
the computation. The `ok` flag remains false if any exception exits the combined
checks before the assignment completes.

`probe` is outside the `try` block. Any local record buffers unwind before the
outer handlers run, while their accounting probe remains alive. The final
`drained` check runs even when a previous check returned false or an exception
escaped it. No required check or operation lives inside an `assert` that
vanishes in Release.

Append this CMake block **after** all existing targets:

```cmake title="CMakeLists.txt"
add_executable(foundation-04 exercises/chapter-04/transactional_buffer.cpp)
target_compile_features(foundation-04 PRIVATE cxx_std_20)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(foundation-04 PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion
    )
endif()
add_test(NAME foundation-04 COMMAND foundation-04)
```

The [executable target](https://cmake.org/cmake/help/latest/command/add_executable.html)
contains only this chapter's file. It does not link another foundation executable
or compile its `main`. Keep the root `enable_testing()` and language-extension
policy established in Chapter 1.

Reconfigure and run the cumulative Debug suite using the native compiler already
selected. A newly created tree uses the same single-configuration generator and
ARM64 target as the earlier chapters:

```sh title="Terminal — Debug build and cumulative tests"
cmake -S . -B build/debug -G "Unix Makefiles" \
    -DCMAKE_BUILD_TYPE=Debug \
    -DCMAKE_CXX_COMPILER="$CXX" \
    -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build/debug --parallel 4 && \
    ctest --test-dir build/debug --output-on-failure --no-tests=error
./build/debug/foundation-04
```

If the existing tree uses Ninja or another compatible generator, retain it
rather than changing its cached generator. Stop on an unexpected configuration
or build error instead of executing an old surviving binary.

Repeat with the separate Release tree:

```sh title="Terminal — Release build and cumulative tests"
cmake -S . -B build/release -G "Unix Makefiles" \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_COMPILER="$CXX" \
    -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build/release --parallel 4 && \
    ctest --test-dir build/release --output-on-failure --no-tests=error
./build/release/foundation-04
```

The cumulative suite now contains **five** tests: `foundation-01`, `cxx20-probe`,
`foundation-02`, `foundation-03`, and `foundation-04`. All should pass in both
configurations. The new executable's expected output is:

```text title="Expected foundation-04 output"
initial: values=4 5 total=9 live=2
second allocation failed: values=4 5 total=9 candidate acquired=1 released=1 live=2
retry: values=8 9 10 total=27 copied-old=5 live=2
aliased input: values=9 10 total=19
clear: absent record; present zero total; live=0
early rejection: size checked first; first allocation acquired=0
late rejection: candidate acquired=2 released=2; old total=9
overflow boundary: maximum plus zero accepted; no wrapped total
unwind: caught bad_alloc; continuation skipped; live=0
final: attempts=19 acquired=16 released=16 live=0 peak-live=4
foundation-04: all checks passed
```

The attempts exceed acquisitions by three: two injected second-allocation
failures and one injected first-allocation failure. The oversize rejection
consumes no attempt. The arithmetic rejection consumes two successful array
allocations, both released. Final live count zero means the instrumented owners
released all 16 of their acquired arrays; it is not a measurement of process
memory or a claim about every allocation in the C++ runtime.

## Make the weaker guarantee fail visibly

Temporarily replace **only** `RecordBuffer::replace` with this complete function:

```cpp title="Temporary diagnostic edit — weaker RecordBuffer::replace"
void RecordBuffer::replace(std::span<const std::uint64_t> input,
                           AllocationProbe& probe) {
    RecordBuffer discarded;
    swap(discarded);
    RecordBuffer candidate(input, probe);
    swap(candidate);
}
```

This version moves the old state into `discarded` and makes the destination
empty **before** preparation. If preparation fails, `discarded` is destroyed as
control leaves the function, releasing the old arrays. The destination remains
a valid empty buffer and no tracked array leaks, but the old value is gone.
It offers only the basic guarantee for this failure path.

Predict the first failing check, then build and run the focused test in both
configurations:

```sh title="Terminal — a leak-free implementation can still violate the contract"
cmake --build build/debug --parallel 4 && \
    ctest --test-dir build/debug -R '^foundation-04$' \
        --output-on-failure --no-tests=error
cmake --build build/release --parallel 4 && \
    ctest --test-dir build/release -R '^foundation-04$' \
        --output-on-failure --no-tests=error
```

Both must report the diagnostic:

```text title="Expected failure from the weaker replacement"
FAIL second-allocation failure preserves value and prefix
```

The executable returns nonzero. The first logical-state check stops the test
before its old-view check, so this diagnostic does not intentionally dereference
a stale borrow. A memory sanitizer can be silent for this incorrect algorithm:
it releases resources correctly while breaking a higher-level state promise.

Restore the original two-statement `replace`, rebuild, and return both full
CTest suites to passing. This controlled variation is inside the chapter's test
allocation, not a retained alternate implementation.

### Check normal memory use separately

After restoring the function, compile the standalone normal program with the
selected compatible compiler and sanitizer runtime:

```sh title="Terminal — focused address and undefined-behavior checks"
mkdir -p build/diagnostics
"$CXX" -std=c++20 -O1 -g \
    -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
    -fsanitize=address,undefined -fno-omit-frame-pointer \
    -fno-sanitize-recover=undefined \
    exercises/chapter-04/transactional_buffer.cpp \
    -o build/diagnostics/foundation-04
./build/diagnostics/foundation-04
```

Expected behavior is the normal eleven-line output and exit status zero, without
a sanitizer report. The explicit counter checks still matter: sanitizer support
and leak-detection defaults vary by platform and installed runtime. Do not turn
an unsupported diagnostic option into a requirement to change the entire
working toolchain. The official
[AddressSanitizer instructions](https://clang.llvm.org/docs/AddressSanitizer.html)
and [UndefinedBehaviorSanitizer instructions](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)
describe compiler and linking flags and their scope.

In particular, the usual `undefined` sanitizer group does not reject ordinary
unsigned modular addition. The `{U,1}` rejection is established by our explicit
range check and test, not by expecting a sanitizer to recognize an application
sum as too large.

| Symptom | Boundary to inspect |
|---|---|
| First replacement unexpectedly succeeds | Was failure armed before the call, and is the ordinal relative to the candidate's two allocations? |
| Retry fails at the same injected point | Was the countdown cleared before throwing? Is this an actual allocator failure instead? |
| Candidate count remains live after failure | Did acquired storage immediately enter a completed owning member? Are acquisition and attempt counters confused? |
| Old values remain but totals change | Did all candidate validation happen off to the side, and does swap exchange every representation member? |
| Failed replacement returns a status but the old value is empty | Was old state cleared before preparing the candidate? |
| The process terminates instead of reaching a handler | Was a throwing operation incorrectly declared `noexcept`, or did cleanup throw during unwinding? |
| Empty lookup appears identical to stored zero | Was optional presence replaced with a default-zero shortcut? |

## Cleanup is not successful completion of external I/O

The strong guarantee above applies to one in-memory destination. No persistent
file, peer, device, or other externally visible system was changed. This is why
we can prove unchanged old state simply by keeping it alive until commit.

Consider instead a storage operation that appends a record, requests persistence,
and then installs a new in-memory index entry. Suppose the file append has
modified external state, but the persistence request reports failure or the
index installation cannot finish. Destroying a local buffer releases its memory.
Destroying a file-handle owner may release a handle. Neither act is a general
undo operation for bytes already written. Even an attempted truncation would
be a new fallible I/O operation, not the automatic inverse of a C++ exception.

A useful API therefore separates cleanup from **explicit completion**. The
explicit operation, such as a checked flush or commit in a later chapter, must
report a failure while the caller can still decide what to do. A destructor
supplies nonthrowing fallback cleanup; it cannot report reliable completion by
throwing on a path that may already be unwinding.

The memory exercise also gives a precise contrast in retry behavior:

| Boundary | What a failure means | What must be established before retry |
|---|---|---|
| This `replace` before commit | Destination unchanged; candidate cleaned up | The caller may retry with a valid input; resources may still be unavailable |
| A future write rejected before any external effect | Nothing accepted under that boundary's contract | Rejection and retry rules must be explicit |
| A future failure after external effects may have occurred | Completion may be uncertain | Recovery or another protocol must resolve effects; repeating blindly may duplicate work |

This is a reasoning example, not an implemented file API or a power-loss test.
The storage chapters will need actual write ordering, record validation, recovery,
and publication rules. They cannot inherit durability from a `try` block or a
nonthrowing destructor. Likewise, a function that logs or invokes arbitrary
callbacks **after** its in-memory swap would need to reconsider what it can
promise if that later work throws.

## What carries forward

Keep `transactional_buffer.cpp`, its appended CMake target, and the restored
prepare-then-swap implementation. Earlier foundation files remain unchanged.
The completed program has a checked input domain, an explicit absence type,
a narrow exception-to-status adapter, two independently owned candidate
resources, and tests at failures before and after each acquisition boundary.

The reusable method is not “put a catch around everything.” Establish a valid
candidate privately, give each acquired resource an owner immediately, identify
the exact commit operation, and keep every operation after that commit within
the promised failure model. State what the caller retains on failure, what it
loses on success, and which external effects are outside the guarantee.

The [roadmap](https://25d-228.github.io/cplusplus-learning/index.html) next turns
to templates, concepts, and ordering contracts. That chapter can return owned
results while accepting borrowed inputs without guessing what a failed
construction does to its caller's previous state. It will not depend on an
unwritten engine subsystem or a generic error framework.

## Technical references

These are primary specifications or official tool documentation for the
facilities used here. The live working draft also contains newer features;
only the C++20-compatible operations shown in this chapter are required.

- C++ working draft: [throwing exceptions](https://eel.is/c++draft/except.throw),
  [handler matching and references](https://eel.is/c++draft/except.handle),
  [stack unwinding and incomplete construction](https://eel.is/c++draft/except.ctor),
  [member initialization](https://eel.is/c++draft/class.base.init),
  [destruction](https://eel.is/c++draft/class.dtor), and
  [termination](https://eel.is/c++draft/except.terminate).
- C++ working draft: [`bad_alloc`](https://eel.is/c++draft/bad.alloc),
  [`invalid_argument`](https://eel.is/c++draft/invalid.argument),
  [the `exception` interface](https://eel.is/c++draft/exception), and
  [library allocation-failure rules](https://eel.is/c++draft/res.on.exception.handling).
- C++ working draft: [optional storage](https://eel.is/c++draft/optional.general),
  [optional observers](https://eel.is/c++draft/optional.observe), and
  [scoped enumerations](https://eel.is/c++draft/dcl.enum), and
  [the `nodiscard` attribute](https://eel.is/c++draft/dcl.attr.nodiscard).
- C++ working draft: [unique-pointer creation](https://eel.is/c++draft/unique.ptr.create),
  [array reset](https://eel.is/c++draft/unique.ptr.runtime.modifiers),
  [pointer swap and default-owner modifiers](https://eel.is/c++draft/unique.ptr.single.modifiers),
  [scalar swap](https://eel.is/c++draft/utility.swap), and
  [unsigned arithmetic](https://eel.is/c++draft/basic.fundamental).
- CMake: [`add_executable`](https://cmake.org/cmake/help/latest/command/add_executable.html).
  Clang: [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) and
  [UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html).
