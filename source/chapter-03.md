# Chapter 3 — Lifetime, RAII, and move-only owners

A buffer holding a disk block can outlive the function that read it, but it
cannot outlive its last owner. A query can borrow the block without becoming responsible for
freeing it. Moving that responsibility to another object need not move even one
payload byte. These are different relationships, and a useful resource type
must make each one explicit.

This chapter builds a fixed-size, move-only byte buffer. It transfers `{4, 5, 6}`
without allocating another payload, releases an already-owned destination before
replacing it, leaves a distinct moved-from source empty, and makes self-move a
no-op. Allocation counters and a separate lifetime trace make those promises
observable. The same reasoning will later govern file handles, pinned blocks,
arenas, and spill-file cleanup; this exercise is not the engine's allocator.

**Starting code:** retain the complete programs and root CMake configuration
from [Chapter 1](https://25d-228.github.io/cplusplus-learning/chapter-01.html) and
[Chapter 2](https://25d-228.github.io/cplusplus-learning/chapter-02.html). Their
three passing CTest entries stay in place. You need references, `const`, arrays,
vectors, spans, and Chapter 1's active `check` pattern. No engine files or future
chapter code are required.

> **Learning objectives**
>
> Separate storage, object lifetime, ownership, and borrowing; define classes
> with constructors, destructors, and private invariants; prefer member-owned
> resources and the rule of zero; distinguish move construction from move
> assignment; implement a documented moved-from state; account for destruction,
> self-move, early return, and vector relocation without double release.

The allocation is **2.5 implementation hours**: 1.0 for the owner and lifetime
exercises, 1.0 for move paths and container integration, and 0.5 for checks and
diagnosis. Reading is separate. The fixtures are deliberately small; their
purpose is an exact state trace, not a throughput result.

## One source file, several observable lifetimes

From the learner workspace root, select the working compiler from Chapter 1
and create the exercise directory:

```sh title="Terminal — prepare this chapter"
CXX="$(xcrun --find clang++)"
mkdir -p exercises/chapter-03
```

Use a previously selected compatible native LLVM compiler instead when that is
your working toolchain. This chapter introduces no dependency beyond the C++20
standard library and the existing build tools.

Every C++ block labelled **`exercises/chapter-03/buffer_owner.cpp`**, in reading
order, appends to that one file. Together they are its complete contents. The
CMake block at the end appends one target and one test. Blocks explicitly labelled
as temporary diagnostic edits are not part of the final source.

## Storage is not an object, and an object is not its payload

*Storage* is a region of memory with a size and alignment. An *object* has a type,
initialization, and a lifetime in suitable storage. For the ordinary class
objects in this chapter, completed initialization establishes a usable object;
starting its destructor ends its lifetime in the formal language sense.
Construction and destruction have special rules that still permit their member
accesses. Merely retaining an address does not keep an object alive. The
[object-lifetime rules](https://eel.is/c++draft/basic.life) distinguish these
concepts more generally, including storage reuse cases we are not using here.

Consider a local owner that contains a pointer to three dynamically allocated
bytes. There are at least three separate things to track:

| Thing | Where its storage comes from | What ends its useful lifetime here |
|---|---|---|
| Local owner object | Automatic storage for a local variable | Leaving its scope |
| Dynamically allocated byte array | An array allocation | Its owning pointer resets or is destroyed |
| Local `std::span` value | Automatic storage for another local variable | Leaving the span's scope; its *borrow* can become invalid sooner |

The pointer member belongs to the owner object, but the pointed-to array is not
a member subobject. Destroying a bare pointer does not release its pointee.
Destroying an owning smart pointer does. A span stores access information; it
does not participate in release responsibility.

We will not use placement `new`, casts from untyped storage, or manual destructor
calls in this chapter. Those would mix the question of *who releases a resource*
with the different question of *which objects exist in raw storage*. The later
arena implementation will need the latter distinction explicitly.

## Constructors establish state; destructors finish a scope

A `class`, like a `struct`, defines a type with data and functions. Its default
member access is private; a struct's is public. We write `public:` for callable
operations and `private:` for internal state. Private access prevents a caller
from casually breaking an invariant; it does not make an incorrect constructor
or member function correct.

A constructor has the class's name and no return type. A destructor uses that
name preceded by `~`, takes no arguments, and has no return type. The expression
following a constructor's colon initializes members **before** its body runs.
A reference member must be bound there or by a member initializer; assignment
inside the body cannot go back and initialize an unbound reference.

Start with an allocation-free log and the active check helper:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
#include <array>
#include <cstddef>
#include <iostream>
#include <memory>
#include <span>
#include <type_traits>
#include <utility>
#include <vector>

namespace strata {

bool check(bool condition, const char* name) {
    if (condition) {
        return true;
    }
    std::cerr << "FAIL " << name << '\n';
    return false;
}

struct LifetimeLog {
    std::array<int, 8> events{};
    std::size_t used = 0;
    bool overflow = false;

    void record(int event) noexcept {
        if (used == events.size()) {
            overflow = true;
            return;
        }
        events[used] = event;
        ++used;
    }
};
```

The function inside `LifetimeLog` is a *member function*. A call such as
`log.record(1)` operates on that particular log's fields. `noexcept` on the
function declares that no exception may escape it. Here that promise is supported
by fixed-array indexing guarded by the bound, and scalar assignments; no memory
allocation or stream output happens inside `record`.

An exception is a separate error-control path, taught in Chapter 4. For now,
remember that `noexcept` does not turn failure into success or make throwing code
safe: an exception escaping such a function leads to termination. See the
[exception specifications](https://eel.is/c++draft/except.spec) and
[termination rules](https://eel.is/c++draft/except.terminate). We use a bounded
log rather than potentially allocating output from a destructor.

The next two small classes record construction as a positive identifier and
destruction as its negative. The identifiers in this fixture are only 1–4.
They borrow `LifetimeLog`; the log must outlive them. A declaration ending in
`= delete` forbids that operation at compile time. These probes deliberately
cannot be copied: duplicating a probe would duplicate destruction events without
matching the construction events we want to observe.

```cpp title="exercises/chapter-03/buffer_owner.cpp"
class ScopeMark {
public:
    ScopeMark(LifetimeLog& log, int id) noexcept : log_(log), id_(id) {
        log_.record(id_);
    }

    ~ScopeMark() noexcept {
        log_.record(-id_);
    }

    ScopeMark(const ScopeMark&) = delete;
    ScopeMark& operator=(const ScopeMark&) = delete;

private:
    LifetimeLog& log_;
    int id_;
};

class MarkPair {
public:
    explicit MarkPair(LifetimeLog& log) noexcept
        : log_(log), first_(log, 1), second_(log, 2) {
        log_.record(3);
    }

    ~MarkPair() noexcept {
        log_.record(-3);
    }

    MarkPair(const MarkPair&) = delete;
    MarkPair& operator=(const MarkPair&) = delete;

private:
    LifetimeLog& log_;
    ScopeMark first_;
    ScopeMark second_;
};
```

The trailing underscores are a naming convention for private fields, not special
syntax. `explicit` prevents `MarkPair`'s one-argument constructor from serving as
an implicit conversion from a log. Direct construction, `MarkPair pair(log)`,
remains valid. `operator=` names an assignment operator; its detailed role will
become concrete in the buffer's move assignment.

`MarkPair` declares `first_` before `second_`. They initialize in that declaration
order, not whichever order an author happens to write in the constructor's
initializer list. Write both in the same order to make the dependency visible.
The constructor body records 3 only after those members are ready. The
[member-initialization specification](https://eel.is/c++draft/class.base.init)
defines this ordering.

On destruction, `MarkPair`'s destructor body records -3 **before** its members are
destroyed. `second_` then records -2, followed by `first_` recording -1. The
reference member `log_` never destroys the log it names. The
[destructor rules](https://eel.is/c++draft/class.dtor) explain the body/member
sequence. Do not call a member's destructor manually: the automatic member
cleanup would otherwise encounter an already-destroyed object.

Append a complete check of both nested scopes and member order:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
bool check_scope_order() {
    LifetimeLog log;
    {
        MarkPair pair(log);
        {
            ScopeMark nested(log, 4);
        }
    }

    const std::array<int, 8> expected{1, 2, 3, 4, -4, -3, -2, -1};
    if (!check(!log.overflow && log.used == expected.size() &&
               log.events == expected, "scope and member destruction order")) {
        return false;
    }
    std::cout << "lifetime:";
    for (int event : log.events) {
        std::cout << ' ' << event;
    }
    std::cout << '\n';
    return true;
}
```

There is no heap allocation in the trace itself. `log` is declared outside the
probes' scope, so all record operations finish before the log dies. This is the
same ownership-order question a future cache will face: a pin must stop using the
cache's bookkeeping before that bookkeeping is destroyed.

## Prefer an existing owner before writing special members

**Resource Acquisition Is Initialization**, or **RAII**, binds a resource's
release responsibility to an object's lifetime. A successful construction gives
you an owner; destruction releases what it still owns. The resource might be an
allocation, file descriptor, lock, or temporary file, not just memory. RAII is
not garbage collection, and it does not require the owner itself to be on the
heap.

`std::vector` already owns its element storage. `std::unique_ptr<T>` represents
exclusive release responsibility for a separately allocated `T`; moving it
transfers that responsibility. For an array, use `std::unique_ptr<T[]>`, whose
default deletion uses the array form of deletion. Neither form is a reference
count. Copying a unique pointer is prohibited; its interfaces are specified in
[unique ownership](https://eel.is/c++draft/unique.ptr.runtime).

Before introducing our cross-member invariant, compare two small aggregates:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
struct ValueRecord {
    std::vector<std::byte> payload;
};

struct UniqueRecord {
    std::unique_ptr<std::array<std::byte, 3>> payload;
};
```

`std::byte`, from `<cstddef>`, represents a byte without treating it as an ordinary
arithmetic integer. Write a known byte value as `std::byte{4}`. For printing a
numeric value, `std::to_integer<unsigned int>(value)` converts explicitly; see
[byte operations](https://eel.is/c++draft/support.types.byteops). The fixed
three-element array here lets `UniqueRecord` avoid a second length member.

Neither aggregate declares a destructor, copy operation, or move operation.
This design is often called the **rule of zero**: let well-chosen member types
supply those operations instead of duplicating their resource management.
`ValueRecord` can copy its vector into independent storage. `UniqueRecord` cannot
copy its unique pointer, but the generated move transfers that member. The
[copy/move constructor rules](https://eel.is/c++draft/class.copy.ctor) describe
memberwise generation and when a generated copy is deleted.

### `std::move` permits a transfer; it does not perform one

A named object expression such as `source` is an *lvalue*: it designates an
existing object. `std::move(source)` produces an expression that permits selection
of an operation accepting an expiring value, conventionally a `T&&` parameter.
It does not allocate, copy payload bytes, destroy the source, or itself remove
ownership. The selected constructor or assignment operator performs the work.
The [move helper's specification](https://eel.is/c++draft/forward) describes a
cast, not a transfer algorithm.

An rvalue reference parameter is written `T&&`; it is not two levels of pointer.
The parameter still refers to an object. Once that parameter has a name inside
the function, using its name is again an lvalue expression. This is why a custom
move constructor must explicitly move the source's owning member rather than
assume every access to the parameter will move automatically.

Append the rule-of-zero exercise:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
bool check_standard_owners() {
    ValueRecord original{{std::byte{4}, std::byte{5}, std::byte{6}}};
    auto copied = original;
    copied.payload[1] = std::byte{9};
    if (!check(original.payload[1] == std::byte{5} &&
               copied.payload[1] == std::byte{9}, "value copy is independent")) {
        return false;
    }

    const std::array<std::byte, 3> fixture{
        std::byte{4}, std::byte{5}, std::byte{6}
    };
    UniqueRecord source{std::make_unique<std::array<std::byte, 3>>(fixture)};
    auto destination = std::move(source);
    if (!check(!source.payload && destination.payload &&
               *destination.payload == fixture, "generated unique-owner move")) {
        return false;
    }
    std::cout << "standard owners: independent copy; unique source empty\n";
    return true;
}
```

`std::make_unique<T>(arguments)` allocates and constructs a `T` and returns its
unique owner. Here `T` is one `std::array`, not an unknown-length array. The
parenthesized `fixture` supplies its copied initial value. Dereferencing
`*destination.payload` accesses that owned array; the boolean tests verify that
the pointer exists before the dereference. When this function returns, the
vectors and remaining unique pointer release their resources through their own
members' destruction. No handwritten destructor is needed.

The empty `source.payload` is a documented consequence of the
[unique-pointer move constructor](https://eel.is/c++draft/unique.ptr.single.ctor),
not a universal property of moving. The standard library generally permits a
valid but unspecified moved-from state unless a type gives a stronger guarantee;
see [moved-from library objects](https://eel.is/c++draft/lib.types.movedfrom).
A user-defined type does not become correct merely because it has a `T&&`
constructor. We must establish and document our own source state.

## An invariant that needs a custom move

Our `ByteBuffer` has two coupled fields: an owning pointer and a byte count.
The public boundary invariant is:

- Empty means a null pointer and count zero, with no allocation.
- Nonempty means exclusive ownership of an array of exactly the advertised
  number of accessible bytes; every byte has been initialized.
- A move between distinct owners preserves the array's address and contents,
  and leaves the source empty. A self-move preserves the original state.

Let $p$ denote the stored pointer and $n$ the byte count. The empty-state part is

$$
n = 0 \quad\Longleftrightarrow\quad p = \mathrm{nullptr}.
$$

The other half of the contract cannot be proved from those two values alone: a
nonnull pointer with count three must actually own a live three-byte array. We
arrange that by making fields private and allowing acquisition only through the
constructor. A span does not validate this ownership for us.

Why not generate every move? A defaulted move would move the unique pointer but
copy the scalar length. A source that originally held three bytes would then
contain a null pointer and length three. Each member would individually be in a
legal state, but the combined object would violate our invariant. The rule of
zero works best when the chosen representation makes memberwise operations
correct; it is not a command to accept an invalid state.

The buffer also records successful payload acquisitions and releases. Define
$A$ as acquisitions, $R$ as releases, and $L$ as currently live payload arrays.
For this finite, single-threaded fixture, at completed public operations,

$$
L = A - R.
$$

Moving to a fresh owner changes neither $A$ nor $R$. Replacing an already-owning
destination releases one array, so $R$ increases by one and $L$ decreases by one.
These counters count **arrays owned by `ByteBuffer` only**. They do not count
stream allocations, vectors' element storage, allocator bookkeeping, the earlier
standard-owner exercise, or process memory usage. They are not a thread-safe
memory budget and are not intended for an indefinitely running counter service.

### Read the interface before its implementation

A default constructor takes no arguments. `= default` asks the compiler to
generate that operation using the members' declared initializers. `= delete`
instead makes an attempted call ill-formed. We explicitly delete both copying
operations because copying this type must not silently allocate a second payload
or manufacture a second release obligation.

`ByteBuffer(ByteBuffer&&)` constructs a new object from a source.
`ByteBuffer& operator=(ByteBuffer&&)` changes an already-existing object and returns
a reference to that destination. The two operations have different cleanup
requirements. A destructor, copy constructor, copy assignment, move constructor,
and move assignment are the five special operations whose interaction you must
consider here; “consider all five” does not mean implement all five as permitted
operations. The [assignment rules](https://eel.is/c++draft/class.copy.assign)
explain generated and deleted assignments.

The view accessors have another qualifier: `bytes() &` is callable on an lvalue
owner, while `bytes() const &` supplies read-only elements when the owner is
observed as const. We delete the corresponding `&&` overloads, preventing a
call such as `ByteBuffer(3).bytes()` from borrowing a temporary that is about to
die. These are *ref-qualified member functions*; the qualifier belongs to the
object on which the method is called, not to its returned span. See
[member-function qualifiers](https://eel.is/c++draft/class.mfct.non.static).
This blocks one easy mistake, not every dangling borrow.

Append the full interface:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
struct AllocationCounts {
    std::size_t acquired = 0;
    std::size_t released = 0;
    std::size_t live = 0;
};

class ByteBuffer {
public:
    ByteBuffer() noexcept = default;
    explicit ByteBuffer(std::size_t size);
    ~ByteBuffer() noexcept;

    ByteBuffer(const ByteBuffer&) = delete;
    ByteBuffer& operator=(const ByteBuffer&) = delete;
    ByteBuffer(ByteBuffer&& other) noexcept;
    ByteBuffer& operator=(ByteBuffer&& other) noexcept;

    std::size_t size() const noexcept;
    bool empty() const noexcept;
    std::span<std::byte> bytes() & noexcept;
    std::span<const std::byte> bytes() const & noexcept;
    std::span<std::byte> bytes() && = delete;
    std::span<const std::byte> bytes() const && = delete;
    void reset() noexcept;

    static AllocationCounts allocations() noexcept;

private:
    std::unique_ptr<std::byte[]> data_{};
    std::size_t size_ = 0;
    inline static AllocationCounts counts_{};
};
```

`size() const` can inspect but not change ordinary members through that access
path. Its returned integer owns its value and needs no lifetime restriction.
A static member function, `allocations()`, has no particular buffer as its target;
call it as `ByteBuffer::allocations()`. The `inline static` data member supplies
one shared counter object, rather than a counter in every buffer. `inline` here
permits the definition in the class; it is not an instruction to inline machine
code. The [static-data-member rules](https://eel.is/c++draft/class.static.data)
cover these declarations. Returning counts by value prevents a caller from
modifying the instrumentation through the result.

An explicitly declared destructor or copy operation changes which move operations
are implicitly supplied. Do not add a logging destructor to an otherwise
rule-of-zero type and assume that its move behavior is unchanged. Here all five
choices are visible rather than left to accidental generation.

## Acquire once, and make release idempotent

Define construction outside the class. `ByteBuffer::` identifies the member
being defined, just as a namespace qualifier identified a function in Chapter 1.

```cpp title="exercises/chapter-03/buffer_owner.cpp"
ByteBuffer::ByteBuffer(std::size_t size)
    : data_(size == 0 ? std::unique_ptr<std::byte[]>{}
                      : std::make_unique<std::byte[]>(size)),
      size_(size) {
    if (data_) {
        ++counts_.acquired;
        ++counts_.live;
    }
}

ByteBuffer::~ByteBuffer() noexcept {
    reset();
}

void ByteBuffer::reset() noexcept {
    if (data_) {
        data_.reset();
        ++counts_.released;
        --counts_.live;
    }
    size_ = 0;
}

AllocationCounts ByteBuffer::allocations() noexcept {
    return counts_;
}
```

The conditional expression selects exactly one initializer. A size of zero uses
an empty unique pointer and performs no array allocation. Otherwise,
`std::make_unique<std::byte[]>(size)` value-initializes the array, so every byte
starts at zero. The [creation functions](https://eel.is/c++draft/unique.ptr.create)
specify this array initialization. `ByteBuffer(0)` and `ByteBuffer{}` therefore
reach the same empty representation; we do not depend on how a zero-length
`new[]` allocation might be represented.

Acquisition is counted only after initialization succeeds. The size-taking
constructor is deliberately **not** `noexcept`: allocation can fail. No counter
is incremented before the allocation, and the remaining constructor body only
updates the counters. If a constructor fails before completion, the complete
object's destructor is not the cleanup mechanism; its fully constructed member
subobjects are cleaned up. Making the resource a smart-pointer member puts that
release responsibility in a constructed subobject. The
[construction-failure rules](https://eel.is/c++draft/except.ctor) underpin this
behavior. Chapter 4 will add explicit handling and deterministic failure tests;
these tiny successful allocations are not an allocation-failure experiment.

`data_.reset()` releases its currently owned array and makes the pointer empty.
Our outer `reset()` also updates the length and instrumentation. Calling it twice
is safe: the second call observes no pointer, releases nothing, and leaves length
zero. This property is called **idempotence**: repeating the operation has the
same final state as doing it once.

Do not substitute `data_.release()`. The similarly named smart-pointer operation
detaches and returns a raw pointer **without deleting the resource**. Ignoring
that return value would leak the array. Nor should a borrower call `delete[]`
on `data_.get()`: the owning pointer would still believe it must delete the same
array. The [unique-pointer modifiers](https://eel.is/c++draft/unique.ptr.single.modifiers)
and [array interface](https://eel.is/c++draft/unique.ptr.runtime) distinguish
observing, releasing responsibility, and destroying a resource.

The destructor calls our reset once; after its body, member destruction still
runs. `data_` is then empty, so its destructor has no second array to release.
There is no throwing callback, allocation, or stream output in this cleanup
path. A real file's successful flush is different from merely releasing its
handle: important completion errors need an explicit operation, not a destructor
that tries to report success by throwing.

## A view exposes bytes, not release authority

Append the size and view operations:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
std::size_t ByteBuffer::size() const noexcept {
    return size_;
}

bool ByteBuffer::empty() const noexcept {
    return size_ == 0;
}

std::span<std::byte> ByteBuffer::bytes() & noexcept {
    if (empty()) {
        return {};
    }
    return {data_.get(), size_};
}

std::span<const std::byte> ByteBuffer::bytes() const & noexcept {
    if (empty()) {
        return {};
    }
    return {data_.get(), size_};
}
```

`get()` observes the stored pointer; it transfers nothing. A mutable owner gives
out a writable span. A const access path gives out a span of const bytes.
Constness of a unique pointer by itself would not make its pointee const; our
chosen return type supplies the read-only access contract.

The explicit empty branch returns a default empty span. It requires no pointer
arithmetic and no attempt to describe three elements through a null pointer.
For the nonempty case, the constructor-established pointer/count invariant makes
the range valid. This accessor does not perform a separate allocation check.

A caller may keep the span only while the particular array it refers to is
alive. Resetting or destroying its current owner invalidates that borrow.
Replacing an owning destination invalidates views into the destination's **old**
array. In contrast, moving our pointer/count pair to a different owner leaves
the same array alive, so a view into the **transferred** array remains valid.
This is a property of our implementation and contract, not a promise that any
operation named “move” preserves every kind of view.

Deleting rvalue accessors is only a guardrail. A span obtained from a named local
can still be returned from a function that then destroys that owner. A caller
can also keep a span and reset the owner. Neither mistake becomes safe because
the original `bytes()` call was permitted by the type system.

## Transfer both fields, not just the pointer

The source of a move is still a live object and will eventually have its own
destructor called. The move must leave it with no responsibility for the
transferred allocation. `std::unique_ptr` handles that obligation for its member;
we must also handle the length.

`std::exchange(object, replacement)`, from `<utility>`, stores the replacement
and returns the old value. For a length of three,
`std::exchange(other.size_, 0)` returns 3 and leaves `other.size_` equal to zero.
This is not an atomic operation; see the
[exchange specification](https://eel.is/c++draft/utility.exchange). Its role here
is to make the old-value/new-state transition explicit.

Append the complete move constructor:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
ByteBuffer::ByteBuffer(ByteBuffer&& other) noexcept
    : data_(std::move(other.data_)),
      size_(std::exchange(other.size_, 0)) {
}
```

The members initialize in declaration order: the destination's `data_` receives
the source's owning pointer, then `size_` receives the source's previous count.
The source has become `(nullptr, 0)` by the time construction completes. No
payload loop, array allocation, or counter adjustment is needed. There are
brief internal intermediate states between those initializers, but nothing in
this single-threaded code observes them or invokes a callback. Our invariant is
promised at operation boundaries, not after every machine instruction.

The `std::move` on `other.data_` is necessary. Merely writing `data_(other.data_)`
would try to copy a unique pointer, because the named expression is an lvalue.
Deleting that copy is a useful compile-time failure, not a reason to add a raw
pointer escape hatch.

Move construction has no old destination resource. Move assignment does:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
ByteBuffer& ByteBuffer::operator=(ByteBuffer&& other) noexcept {
    if (this != &other) {
        reset();
        data_ = std::move(other.data_);
        size_ = std::exchange(other.size_, 0);
    }
    return *this;
}
```

`this` is the pointer to the object receiving the member call. `&other` is the
address of the source object. `*this` names the destination object; returning it
as `ByteBuffer&` returns a reference, not a new owner. A call
`destination = std::move(source)` selects this function when both have the
appropriate types.

The identity check handles aliases: `other` may be another reference to the very
same destination. If so, we preserve the object, payload, and counts. Without
that check, our reset-first algorithm would erase the object before trying to
move it from itself. That might avoid a double deletion, yet it would still
violate our stronger no-op self-move contract.

For different objects, reset destroys the destination's previous allocation and
accounts for the release. Only then do we take the source's pointer and length.
The [unique-pointer assignment](https://eel.is/c++draft/unique.ptr.single.asgn)
already knows how to release its old pointer; our explicit reset also maintains
the enclosing buffer's counters and invariant. After reset the pointer is empty,
so the member assignment has no second old allocation to release.

A swap-based assignment is another possible policy. It would leave the source
owning the destination's old payload until later cleanup, rather than empty.
That can be valid, but it is not the policy promised here. “Moved from” describes
an operation that happened; it is not a complete state specification.

### Why these moves can honestly be `noexcept`

They transfer a default-deleter unique pointer, assign scalar fields, and possibly
release a byte array. None of those operations needs a new allocation or a
throwing element destructor. This reasoning, not the attractiveness of a faster
container path, justifies the declaration.

When a vector needs new element storage, it must construct elements there while
preserving its specified failure behavior. A nonthrowing move gives it a transfer
path that cannot fail after some old elements have been moved from. For some
copyable types a potentially throwing move motivates copying instead; a move-only
type has no such fallback. The [vector capacity contract](https://eel.is/c++draft/vector.capacity)
and [insertion guarantees](https://eel.is/c++draft/vector.modifiers) specify the
relevant exceptions rather than promising that every move-looking expression
is cheap.

Add compile-time checks using standard *type traits*: the `_v` forms are boolean
questions about a type, not function calls on a buffer. `static_assert` rejects
the translation unit when a condition is false. The
[type-property specification](https://eel.is/c++draft/meta.unary.prop) defines
these queries.

```cpp title="exercises/chapter-03/buffer_owner.cpp"
static_assert(!std::is_copy_constructible_v<ByteBuffer>);
static_assert(!std::is_copy_assignable_v<ByteBuffer>);
static_assert(std::is_nothrow_move_constructible_v<ByteBuffer>);
static_assert(std::is_nothrow_move_assignable_v<ByteBuffer>);
static_assert(std::is_nothrow_destructible_v<ByteBuffer>);
```

These checks verify the type's declared interface, not that a body never leaks or
violates an invariant. A wrongly written `noexcept` function can still terminate.
Runtime state checks are still necessary.

## Follow two allocations through three owners

Use logical allocation names rather than platform-dependent addresses. Let `Q`
be the one-byte destination's original array and `P` the three-byte source array.
The trace within one test is:

| Operation completed | `source` | `moved` | `destination` | $A$ count / $R$ count / $L$ count |
|---|---|---|---|---|
| Construct destination and source | Owns array P, length 3 | Not constructed | Owns array Q, length 1 | 2 / 0 / 2 |
| Move-construct `moved` | Empty | Owns array P, length 3 | Owns array Q, length 1 | 2 / 0 / 2 |
| Move-assign destination | Empty | Empty | Owns array P, length 3; Q released | 2 / 1 / 1 |
| Self-move destination | Empty | Empty | Still owns array P | 2 / 1 / 1 |
| Leave the inner scope | Destroyed, no release | Destroyed, no release | Still owns array P | 2 / 1 / 1 |
| Leave the outer scope | — | — | Destroyed; P released | 2 / 2 / 0 |

Here $A$, $R$, and $L$ are the acquisition, release, and live-array counters;
P and Q are only labels for the two resources. The increments are relative to
the counters before this test. This example requires no claim about an
allocator's physical address reuse.

First append two complete fixture predicates:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
bool empty_state(const ByteBuffer& buffer) noexcept {
    return buffer.empty() && buffer.size() == 0 &&
           buffer.bytes().empty() && buffer.bytes().data() == nullptr;
}

bool has_fixture(const ByteBuffer& buffer) noexcept {
    const auto view = buffer.bytes();
    return view.size() == 3 && view[0] == std::byte{4} &&
           view[1] == std::byte{5} && view[2] == std::byte{6};
}
```

Both receive read-only references to owners and retain no borrow after returning.
The size check precedes the element reads. `empty_state` checks the documented
null/zero representation, rather than assuming “safe to destroy” means empty.

Start the transfer test with an outer destination and an inner source. The
inner span expires before the destination releases the transferred allocation:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
bool check_transfers() {
    const auto baseline = ByteBuffer::allocations();
    {
        ByteBuffer destination(1);
        destination.bytes()[0] = std::byte{99};
        {
            ByteBuffer source(3);
            auto borrowed = source.bytes();
            if (!check(borrowed.size() == 3 &&
                       borrowed[0] == std::byte{0} &&
                       borrowed[1] == std::byte{0} &&
                       borrowed[2] == std::byte{0}, "new bytes are initialized")) {
                return false;
            }
            borrowed[0] = std::byte{4};
            borrowed[1] = std::byte{5};
            borrowed[2] = std::byte{6};

            const auto before_move = ByteBuffer::allocations();
            ByteBuffer moved(std::move(source));
            const auto after_move = ByteBuffer::allocations();
            if (!check(empty_state(source) && has_fixture(moved),
                       "move construction transfers value and empties source") ||
                !check(moved.bytes().data() == borrowed.data() &&
                       borrowed[1] == std::byte{5}, "borrow survives transfer") ||
                !check(after_move.acquired == before_move.acquired &&
                       after_move.released == before_move.released &&
                       after_move.live == before_move.live,
                       "move construction changes no allocation count")) {
                return false;
            }
            std::cout << "move construction: source=" << source.size()
                      << " destination=" << moved.size()
                      << " live=" << after_move.live << '\n';
```

This block intentionally leaves `check_transfers` and both scopes open. The next
block continues the same function. The address comparison is between pointers
into a **still-live array**. We never keep a pointer into the destination's old
array for use after replacement.

```cpp title="exercises/chapter-03/buffer_owner.cpp"
            destination = std::move(moved);
            const auto after_assign = ByteBuffer::allocations();
            if (!check(empty_state(moved) && has_fixture(destination),
                       "move assignment transfers value and empties source") ||
                !check(destination.bytes().data() == borrowed.data() &&
                       borrowed[2] == std::byte{6}, "borrow follows live array") ||
                !check(after_assign.acquired == after_move.acquired &&
                       after_assign.released == after_move.released + 1 &&
                       after_assign.live + 1 == after_move.live,
                       "assignment releases exactly the old destination")) {
                return false;
            }
            std::cout << "move assignment: source=" << moved.size()
                      << " destination=" << destination.size()
                      << " live=" << after_assign.live << '\n';

            auto& same_object = destination;
            destination = std::move(same_object);
            const auto after_self = ByteBuffer::allocations();
            if (!check(has_fixture(destination) &&
                       destination.bytes().data() == borrowed.data(),
                       "self-move preserves payload") ||
                !check(after_self.acquired == after_assign.acquired &&
                       after_self.released == after_assign.released &&
                       after_self.live == after_assign.live,
                       "self-move changes no allocation count")) {
                return false;
            }
            std::cout << "self-move: payload preserved; no allocation\n";
        }
        if (!check(has_fixture(destination), "new owner outlives source")) {
            return false;
        }
    }
    const auto after = ByteBuffer::allocations();
    if (!check(after.acquired == baseline.acquired + 2 &&
               after.released == baseline.released + 2 &&
               after.live == baseline.live, "both transfer-test arrays released")) {
        return false;
    }
    std::cout << "transfer cleanup: acquired=" << after.acquired - baseline.acquired
              << " released=" << after.released - baseline.released
              << " live=" << after.live << '\n';
    return true;
}
```

`same_object` is an alias, not a copied owner. Self-move through an alias is a
real possibility in a generic algorithm; the test should not depend on the
compiler noticing the spelling `x = std::move(x)`.

On any failing `check`, returning `false` still exits the active scopes. Live
local owners are destroyed along that return path. The test does not require a
handwritten cleanup label or a special branch for each failure. Main will also
inspect the final live count even when a helper reports failure.

## Empty objects and reusable moved-from sources

A valid empty state is useful only if normal operations handle it. Test default
construction, size-zero construction, moving from empty, assigning empty over a
nonempty destination, reusing a previously moved-from object, and repeated reset.
No view from this test survives an operation that releases its array.

```cpp title="exercises/chapter-03/buffer_owner.cpp"
bool check_empty_paths() {
    const auto baseline = ByteBuffer::allocations();
    {
        ByteBuffer empty;
        ByteBuffer zero(0);
        ByteBuffer moved_empty(std::move(zero));
        if (!check(empty_state(empty) && empty_state(zero) &&
                   empty_state(moved_empty), "empty construction and move") ||
            !check(ByteBuffer::allocations().acquired == baseline.acquired,
                   "empty owners do not allocate")) {
            return false;
        }

        ByteBuffer occupied(2);
        occupied = std::move(empty);
        if (!check(empty_state(occupied) && empty_state(empty),
                   "empty source clears occupied destination") ||
            !check(ByteBuffer::allocations().live == baseline.live,
                   "empty assignment releases old allocation")) {
            return false;
        }

        zero = ByteBuffer(3);
        empty = std::move(zero);
        if (!check(empty_state(zero) && empty.size() == 3 &&
                   empty.bytes()[2] == std::byte{0},
                   "moved-from owner can receive a new buffer")) {
            return false;
        }
        empty.reset();
        empty.reset();
        auto& same_empty = zero;
        zero = std::move(same_empty);
        if (!check(empty_state(empty) && empty_state(zero),
                   "repeated reset and empty self-move")) {
            return false;
        }
    }
    const auto after = ByteBuffer::allocations();
    if (!check(after.acquired == baseline.acquired + 2 &&
               after.released == baseline.released + 2 &&
               after.live == baseline.live, "empty-path allocations balance")) {
        return false;
    }
    std::cout << "empty paths: acquired=" << after.acquired - baseline.acquired
              << " released=" << after.released - baseline.released
              << " live=" << after.live << '\n';
    return true;
}
```

In `zero = ByteBuffer(3)`, the right side is a temporary owner. Its allocation is
transferred by assignment; the temporary is empty before its destruction at the
end of that full expression. The subsequent move into `empty` transfers that
same allocation again. Neither transfer creates another array.

Notice the difference between assigning from a temporary and borrowing from it.
Taking ownership before it dies is safe. Returning a span to its bytes without
transferring ownership would not keep those bytes alive; the deleted rvalue
accessors reject that particular spelling.

## A return statement still crosses the lifetime boundary

Append a function with an early-return branch and a success branch, followed by
a check of both:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
bool probe_with_early_return(bool accept) {
    ByteBuffer scratch(3);
    if (!accept) {
        return false;
    }
    scratch.bytes()[0] = std::byte{42};
    return scratch.bytes()[0] == std::byte{42};
}

bool check_early_return() {
    const auto baseline = ByteBuffer::allocations();
    const bool rejected = probe_with_early_return(false);
    const bool accepted = probe_with_early_return(true);
    const auto after = ByteBuffer::allocations();
    if (!check(!rejected && accepted, "both return paths produce their result") ||
        !check(after.acquired == baseline.acquired + 2 &&
               after.released == baseline.released + 2 &&
               after.live == baseline.live, "both return paths clean up")) {
        return false;
    }
    std::cout << "early return: rejected=" << rejected
              << " accepted=" << accepted << " live=" << after.live << '\n';
    return true;
}
```

The returned boolean is evaluated before local cleanup. It is an independent
value, not a reference to `scratch`. Both paths destroy the local owner; neither
returns its allocation. The same discipline applies when a future parser rejects
a malformed record halfway through its work.

Do not generalize scope cleanup into a crash-recovery guarantee. Normal returns
and exception unwinding toward a handler can run destructors; abrupt termination,
process killing, and power loss are different events. Nor does releasing a
buffer undo a disk write. The storage chapters will need a durable recovery
protocol rather than an assumption that cleanup always executes.

## Relocating owners is different from relocating their arrays

A `std::vector<ByteBuffer>` owns storage for its **buffer objects**. Each buffer
in turn owns a separate byte array. If vector growth relocates the buffer
objects, references to those objects are invalidated. Our buffers' move
constructors nevertheless keep their separately allocated arrays alive at the
same addresses. A view to those bytes is therefore a different dependency from
a reference to a vector element.

Append a forced relocation check. As in Chapter 2, request more than the observed
capacity rather than assuming a particular growth factor:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
bool check_vector_relocation() {
    const auto baseline = ByteBuffer::allocations();
    {
        std::vector<ByteBuffer> buffers;
        buffers.push_back(ByteBuffer(3));
        auto payload = buffers[0].bytes();
        payload[0] = std::byte{7};
        const auto before = ByteBuffer::allocations();
        const auto old_capacity = buffers.capacity();
        if (!check(old_capacity < buffers.max_size(), "room for capacity probe")) {
            return false;
        }
        buffers.reserve(old_capacity + 1);
        const auto after = ByteBuffer::allocations();
        if (!check(buffers.size() == 1 &&
                   buffers[0].bytes().data() == payload.data() &&
                   payload[0] == std::byte{7}, "payload survives owner relocation") ||
            !check(after.acquired == before.acquired &&
                   after.released == before.released && after.live == before.live,
                   "owner relocation allocates no new payload array")) {
            return false;
        }
        std::cout << "vector relocation: payload preserved; new payload arrays=0\n";
    }
    const auto after = ByteBuffer::allocations();
    return check(after.acquired == baseline.acquired + 1 &&
                 after.released == baseline.released + 1 &&
                 after.live == baseline.live, "vector-owned payload released");
}
```

The vector did allocate new **element storage**; the output deliberately says
“new payload arrays,” not “no allocation.” No reference or pointer to an old
`ByteBuffer` object is kept across reserve. `payload` is declared after `buffers`,
so its local scope ends before vector destruction releases the live array.
A successful reserve leaves our view usable. A future owner that stores bytes
inline, rather than separately allocating them, would need a different promise.

There is no timing claim here. Counting payload acquisitions tells us which
mechanism ran; it does not measure allocator overhead, cache behavior, or total
process memory. Those measurements need larger, controlled workloads later.

## Connect the checks to the existing build

Close the namespace and append the complete `main`. The separate `drained` check
runs even when the earlier `&&` chain stops at a failed helper:

```cpp title="exercises/chapter-03/buffer_owner.cpp"
} // namespace strata

int main() {
    const bool ok = strata::check_scope_order() &&
                    strata::check_standard_owners() &&
                    strata::check_transfers() &&
                    strata::check_empty_paths() &&
                    strata::check_early_return() &&
                    strata::check_vector_relocation();
    const auto counts = strata::ByteBuffer::allocations();
    const bool drained = strata::check(
        counts.live == 0 && counts.acquired == counts.released,
        "all ByteBuffer payloads released");
    if (!ok || !drained) {
        return 1;
    }
    std::cout << "final: acquired=" << counts.acquired
              << " released=" << counts.released << " live=" << counts.live << '\n';
    std::cout << "foundation-03: all checks passed\n";
    return 0;
}
```

There are seven counted nonempty arrays: two in the transfer test, two in the
empty-path test, two in the return-path test, and one in the vector test. The
lifetime markers allocate none; the standard-owner comparison is outside this
class's instrumentation. Every successful normal path returns to zero live
`ByteBuffer` payloads before the final report.

Append, rather than replace, this block in the root CMake file:

```cmake title="CMakeLists.txt"
add_executable(foundation-03 exercises/chapter-03/buffer_owner.cpp)
target_compile_features(foundation-03 PRIVATE cxx_std_20)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(foundation-03 PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion
    )
endif()
add_test(NAME foundation-03 COMMAND foundation-03)
```

This target has no dependency on the earlier statistics library. Its helpers and
`main` form one independent executable, just as Chapter 2's did. Keep the existing
`CMAKE_CXX_EXTENSIONS OFF` and `enable_testing()` settings from Chapter 1.

Run the cumulative Debug configuration:

```sh title="Terminal — Debug build and checks"
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug \
    -DCMAKE_CXX_COMPILER="$CXX"
cmake --build build/debug --parallel 4
ctest --test-dir build/debug --output-on-failure
./build/debug/foundation-03
```

Expected output of the last command, with the source exactly as written:

```text title="Expected foundation-03 output"
lifetime: 1 2 3 4 -4 -3 -2 -1
standard owners: independent copy; unique source empty
move construction: source=0 destination=3 live=2
move assignment: source=0 destination=3 live=1
self-move: payload preserved; no allocation
transfer cleanup: acquired=2 released=2 live=0
empty paths: acquired=2 released=2 live=0
early return: rejected=0 accepted=1 live=0
vector relocation: payload preserved; new payload arrays=0
final: acquired=7 released=7 live=0
foundation-03: all checks passed
```

The cumulative CTest suite now has **four** tests: `foundation-01`,
`cxx20-probe`, `foundation-02`, and `foundation-03`. Then build and test Release
separately, preserving the compiler selection and existing configuration:

```sh title="Terminal — Release build and checks"
cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_COMPILER="$CXX"
cmake --build build/release --parallel 4
ctest --test-dir build/release --output-on-failure
./build/release/foundation-03
```

The logical output should be the same. No required operation or validation is
inside an `assert` that disappears with `NDEBUG`. These are expected native
commands and output, not a claim about a particular installed macOS release.

## Diagnose an ownership mistake at the right boundary

Keep the passing program as the baseline. The next edits are separate, temporary
experiments; undo each one before trying the next. They are not additional
permanent statements in `check_transfers`.

### Ask for a forbidden copy

Immediately after `ByteBuffer destination(1);` in `check_transfers`, temporarily
insert:

```cpp title="Temporary diagnostic edit — forbidden copy"
ByteBuffer forbidden(destination);
```

Compile without linking:

```sh title="Terminal — check a temporary negative edit"
"$CXX" -std=c++20 -Wall -Wextra -Wpedantic -fsyntax-only \
    exercises/chapter-03/buffer_owner.cpp
echo "compiler exit status=$?"
```

Expected behavior: a nonzero status and a diagnostic identifying the deleted copy
constructor. Exact diagnostic wording varies by compiler. The error should name
the attempted copy, not a missing include or syntax error elsewhere. Even an
empty buffer is noncopyable; copyability is a type-level interface choice, not a
runtime branch based on its current size. Remove the inserted line.

### A const source is not permission to change ownership

At the same location, instead insert these two lines:

```cpp title="Temporary diagnostic edit — move from a const access path"
const auto& frozen = destination;
ByteBuffer forbidden(std::move(frozen));
```

Run the same compile-only command. `std::move` does not remove `const`, so the
non-const `ByteBuffer&&` move constructor cannot accept that expression. The
copy constructor can bind a const reference, but it is deleted. The expected
result is therefore another compilation failure, not a silent transfer.

Do not fix this by casting constness away. When the caller truly gives up a
mutable owner's resource, express that through the appropriate mutable object.
When it merely borrows, use the view/reference interface. Remove both lines.

### Reject a borrow from a temporary

At the same location, temporarily insert:

```cpp title="Temporary diagnostic edit — temporary owner borrow"
auto forbidden_view = ByteBuffer(3).bytes();
```

The compile-only command should reject the deleted rvalue-qualified accessor.
Without that guard, the temporary owner would normally be destroyed at the
semicolon while a copied span remained. Naming an owner whose scope covers the
use of its view is the fix; a longer-lived variable containing only the span is
not. Remove this line too.

### Test the self-move promise, not just absence of a crash

In the definition of `ByteBuffer::operator=`, temporarily change only
`if (this != &other)` to `if (true)`. Rebuild the Chapter 3 target and run its test:

```sh title="Terminal — deliberate self-move defect"
cmake --build build/debug --target foundation-03 --parallel 4
ctest --test-dir build/debug -R '^foundation-03$' --output-on-failure
echo "CTest exit status=$?"
```

Expected behavior: a failing test containing `FAIL self-move preserves payload`.
The defective algorithm resets the destination, which is also its own source,
then transfers an empty state. This is a semantic failure even if every
allocation is eventually released.

This particular diagnostic remains safe: `has_fixture(destination)` first checks
that the now-empty view has three bytes. It returns false, so `&&` does **not**
evaluate the subsequent comparison with the old borrow. The test returns without
reading freed bytes. Restore the identity condition immediately, then rebuild
and rerun both complete Debug and Release suites. Do not retain the defect as a
normal example of acceptable moved-from behavior.

### Run a focused memory diagnostic on the restored program

Use the supported sanitizer runtime associated with the selected native compiler:

```sh title="Terminal — ownership checks with sanitizers"
mkdir -p build/diagnostics
"$CXX" -std=c++20 -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
    -g -O1 -fsanitize=address,undefined -fno-omit-frame-pointer \
    -fno-sanitize-recover=undefined \
    exercises/chapter-03/buffer_owner.cpp \
    -o build/diagnostics/foundation-03-sanitized
./build/diagnostics/foundation-03-sanitized
```

Expected behavior: the same successful output and no sanitizer report.
[AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) instruments
memory accesses and can expose use-after-free or double deletion on executed
paths. It must be enabled at compilation and final linking, which this single
compiler invocation does. On macOS, its documentation notes that `dsymutil` may
be needed for source-file/line symbolization. For example, run
`dsymutil build/diagnostics/foundation-03-sanitized` when a report lacks those
locations, then rerun.

[UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)
adds selected language checks; the no-recovery option makes a diagnosed undefined
operation stop the run. Neither tool proves an ownership contract, and optional
leak-detection availability is separate from basic address instrumentation. Do
not label a missing compiler runtime as a successful sanitizer run. Use the
explicit allocation checks regardless of the platform's diagnostic support.

The examples above distinguish three boundaries: illegal interfaces are rejected
by compilation, broken state promises fail ordinary tests, and invalid memory
access needs a memory diagnostic when the path is exercised. The right response
to a failure is not always “add another smart pointer.”

| Symptom | Inspect first | Correction to the design or implementation |
|---|---|---|
| A supposed move tries to call a deleted copy | The source expression's value category and constness | Move the mutable owning member; do not remove const with a cast |
| Moved-from size is still three | The scalar field in the move operation | Transfer the count and reset the source, not just its pointer |
| Live count stays positive after replacement | Destination cleanup and any use of `release()` | Destroy and account for the old resource before accepting the new one |
| Live count wraps to a huge value | Decrement on an empty owner or repeated cleanup | Guard release by actual ownership and make reset idempotent |
| A borrow fails after vector growth | Whether it names the owner object or its separate array | Reacquire invalidated element references; preserve only borrows the actual owner contract supports |
| Destruction events are missing after abnormal exit | Whether scope exit or unwinding occurred at all | Do not substitute RAII for crash recovery or explicit completion protocols |

## Change member order and predict the trace

For one short variation, exchange `first_` and `second_` in `MarkPair`'s **member
declarations**. Before changing its constructor initializer list, compile and
inspect any member-reordering warning: the declarations, not the written
initializer-list order, determine what happens. Then put the initializer list
in the same order as those declarations:

```cpp title="Variation — MarkPair member declaration order"
LifetimeLog& log_;
ScopeMark second_;
ScopeMark first_;
```

The constructor's initializer line becomes
`: log_(log), second_(log, 2), first_(log, 1)`. Its body and destructor body stay
unchanged. Update the expected array in `check_scope_order` to:

```cpp title="Variation — expected reordered lifetime events"
const std::array<int, 8> expected{2, 1, 3, 4, -4, -3, -1, -2};
```

Predict the result before running. The containing destructor body still records
-3 before either member destructor, but member construction and destruction order
have both changed. This matters when one member borrows another: declaration
order can determine whether cleanup still has a live dependency. Here both
members only borrow the outer log, so either order is safe.

After the experiment, restore the original declarations, initializer list, and
expected array. The next chapter assumes the original passing program, not a
negative edit or reordered trace. Keep the five static assertions, all six
runtime check functions, and the appended CMake target.

## What this owner does and does not promise

The buffer has exclusive ownership, a known empty state, initialized payloads,
nonthrowing transfer and cleanup, and restricted view creation. A distinct move
empties its source; self-move preserves the value. Existing destinations release
their old arrays. Borrowers observe the array, not release authority. The checks
make each transition visible without deliberately dereferencing dead storage.

This is not a replacement for `std::vector`, an engine memory allocator, a shared
cache pin, or a transactional update. Ordinary values with independent member
invariants should continue to use the rule of zero. Here custom special members
were justified by the pointer/count relationship and accounting. Do not copy
this single-threaded instrumentation into a concurrent engine and call it a
memory budget.

The next chapter introduces error categories and exception handling, then builds
a replacement before committing it. That will answer a different question:
when acquiring a proposed new resource fails, how do we preserve the old value
without leaking either resource?

## Technical references

The links near the explanations identify the specific contracts used. These
working-draft pages also contain newer language/library features; only the
C++20 facilities used in this chapter are required.

| Topic | Primary reference and relevant passage |
|---|---|
| Object versus storage | [Lifetime](https://eel.is/c++draft/basic.life): beginning and end of lifetime, and the distinction from retained storage |
| Construction/destruction order | [Initializing members](https://eel.is/c++draft/class.base.init) and [destructors](https://eel.is/c++draft/class.dtor): declaration order, destructor body, and reverse member destruction |
| Generated special members | [Copy/move constructors](https://eel.is/c++draft/class.copy.ctor) and [assignment](https://eel.is/c++draft/class.copy.assign): generation, memberwise operations, and deletion |
| Existing unique ownership | [Array owners](https://eel.is/c++draft/unique.ptr.runtime), [creation](https://eel.is/c++draft/unique.ptr.create), and [modifiers](https://eel.is/c++draft/unique.ptr.single.modifiers): array deletion, value initialization, reset versus release |
| Transfer semantics | [Move helpers](https://eel.is/c++draft/forward), [exchange](https://eel.is/c++draft/utility.exchange), and [moved-from states](https://eel.is/c++draft/lib.types.movedfrom): expression conversion, old-value replacement, and type-specific postconditions |
| Error boundaries | [Construction failure](https://eel.is/c++draft/except.ctor) and [termination](https://eel.is/c++draft/except.terminate): subobject cleanup and nonthrowing boundaries |
| Container interaction | [Vector capacity](https://eel.is/c++draft/vector.capacity) and [modifiers](https://eel.is/c++draft/vector.modifiers): reallocation, invalidation, and exception guarantees |
| Diagnostics | Clang [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) and [UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html): compiler/linker flags, runtime checks, and symbolization |
