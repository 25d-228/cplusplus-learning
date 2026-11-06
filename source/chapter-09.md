# Chapter 9 — Memory budgets and a pinned block cache

## Part 1 — Charge allocation, not logical contents

A block size limit does not bound a process. Chapter 8 limits each table cursor,
but a hundred cursors can still allocate a hundred buffers. A cache introduces
another distinction: the cache may stop retaining a block while a reader still
needs its bytes. Eviction policy and allocation lifetime cannot be the same thing.

We will give the engine's dynamic storage a common byte ledger, then build a
synchronous cache whose readers share immutable blocks. A full cache is not an
I/O error. It may return a separately charged block without retaining it. A full
**memory budget**, however, must reject an allocation; bypass is not permission
to allocate outside the ledger.

> **Learning objectives**
> Reserve before allocating; charge vector capacity and allocation-control storage;
> implement a stateful standard-library allocator; share immutable buffers through
> owning pins; maintain explicit recency links; evict only unpinned entries; connect
> cache misses to the real table validator; diagnose logical and memory failures
> separately.

**Prerequisites:** all code through the three parts of
[Chapter 8](https://25d-228.github.io/cplusplus-learning/chapter-08.html), especially
its `TableReader::Cursor::load`, and the ownership, exception-safety, and template
material in Chapters 3–5. Keep the eleven passing CTest entries. The code remains
in the same `work/strata` tree. There is no log, database manifest, or thread pool
in this chapter.

**Implementation allocation: 3.5 hours for all three parts together.** Allocate
1.0 hour to the ledger, allocator, and existing allocation boundaries; 1.5 to
pins, recency, admission, and reader integration; and 1.0 to pressure tests,
debugging, and the deliberately incorrect recency variation. Reading is separate.

Part 1 implements the ledger and charges arenas/builders. Part 2 implements the
cache. [Part 3](https://25d-228.github.io/cplusplus-learning/chapter-09-part-3.html)
updates the table reader and supplies the complete test program and CMake changes.
Finish all three parts before rebuilding: the API migration is deliberately
explicit rather than preserving a silent, uncharged compatibility overload.

```sh title="Terminal — new memory and cache files"
mkdir -p include/strata/memory include/strata/storage src/memory src/storage tests
```

Create `include/strata/memory/budget.hpp`, `src/memory/budget.cpp`,
`include/strata/storage/block_cache.hpp`, `src/storage/block_cache.cpp`,
`tests/cache_file_ops.hpp`, and `tests/block_cache.cpp`. Repeated filename-labelled
blocks for these **new files** append in reading order. Edits to the existing
arena, memtable, and table files are separately labelled as replacements. Do not
append a second definition of an existing function.

## Decide exactly what a byte limit promises

Let $L$ be the ledger limit in bytes, $R$ the bytes reserved but not yet allocated,
and $A$ the bytes charged to live allocations. Define admitted usage $U$ by

$$
U = R + A, \qquad 0 \le U \le L.
$$

An allocation request of $n$ bytes is admitted only when $n \le L-U$. Test that
subtraction, rather than computing an unchecked $U+n$ first. Once admission
succeeds, adding $n$ is safe because the result is at most the representable limit
$L$.

A **lease** owns a reservation. Its destructor returns that reservation unless the
allocator has converted it into an allocation charge. A lease is move-only: there
must not be two destructors returning the same reserved bytes. An allocator-owned
charge instead lives until the matching `deallocate` call.

| Operation | Reserved bytes | Allocated bytes | Admitted usage |
|---|---:|---:|---:|
| Start, with a 16-byte limit | 0 | 0 | 0 |
| Reserve 12 | 12 | 0 | 12 |
| Attempt to reserve 5 | 12 | 0 | 12; rejected |
| Allocate the admitted 12 successfully | 0 | 12 | 12 |
| Deallocate those 12 | 0 | 0 | 0 |

If the underlying allocation throws, the lease returns the 12 reserved bytes.
No allocation charge was installed. The high-water mark records **admitted usage**,
including such temporary reservations, not merely successfully allocated memory.

The accounting domain is specific. We will charge every allocation made through
our allocator: full arena chunks, builder workspaces, reader metadata, cached and
bypass frame capacity, cache slot storage, and the storage requested for each shared
block's object/control state. A node inside an already charged arena chunk is not
charged again. A second pin to an existing block is not another block allocation.

This is **not** a cap on the operating system's process-memory measurement.
Allocator bookkeeping and size-class rounding outside requested object storage,
fixed stack objects, file-system/error-message internals, runtime libraries,
and the operating system's file cache are outside this ledger. Test fixtures and
caller-owned input arrays are outside it too. Later subsystems must join the
ledger explicitly; a naked allocation does not become charged by proximity.
Leave headroom for excluded storage and measure process memory separately.

## Teach the ledger to own a reservation

A budget outlives every lease and allocation that refers to it. It is neither
copyable nor movable: copied limits would create competing ledgers, and moving
its address would invalidate allocator pointers. Put a budget before its users
in an enclosing scope so reverse destruction releases their allocations first.

The error type below derives from `std::bad_alloc`, the allocation-failure family
introduced in Chapter 4. Its `what()` returns a fixed string; constructing this
particular error does not need a dynamically allocated explanatory message.
A caller may catch `BudgetExceeded` specifically, or catch `std::bad_alloc` to
handle both admission refusal and underlying allocation failure.

```cpp title="include/strata/memory/budget.hpp"
#ifndef STRATA_MEMORY_BUDGET_HPP
#define STRATA_MEMORY_BUDGET_HPP

#include <cstddef>
#include <exception>
#include <limits>
#include <memory>
#include <new>
#include <optional>
#include <stdexcept>
#include <type_traits>
#include <utility>
#include <vector>

namespace strata::memory {

template<class T> class Allocator;

class BudgetExceeded : public std::bad_alloc {
public:
    const char* what() const noexcept override { return "memory budget exhausted"; }
};
```

`Lease` is nested to keep its relationship to the ledger explicit. Its default
state owns nothing. `friend class Budget` permits the budget to create a nonempty
lease, while the template friend permits an allocator specialization to commit
one. Neither operation is public to an arbitrary caller. Friendship grants
access; it does not extend an object's lifetime.

```cpp title="include/strata/memory/budget.hpp"
class Budget {
public:
    class Lease {
    public:
        Lease() noexcept = default;
        ~Lease() noexcept { reset(); }
        Lease(const Lease&) = delete;
        Lease& operator=(const Lease&) = delete;
        Lease(Lease&& other) noexcept;
        Lease& operator=(Lease&& other) noexcept;
        void reset() noexcept;
        std::size_t bytes() const noexcept { return bytes_; }
    private:
        friend class Budget;
        template<class T> friend class Allocator;
        Lease(Budget& owner, std::size_t bytes) noexcept
            : owner_(&owner), bytes_(bytes) {}
        void commit_allocation() noexcept;
        Budget* owner_ = nullptr;
        std::size_t bytes_ = 0;
    };

    explicit Budget(std::size_t limit) noexcept : limit_(limit) {}
    ~Budget() noexcept;
    Budget(const Budget&) = delete;
    Budget& operator=(const Budget&) = delete;
    Budget(Budget&&) = delete;
    Budget& operator=(Budget&&) = delete;
    [[nodiscard]] Lease reserve(std::size_t bytes);
    std::size_t limit() const noexcept { return limit_; }
    std::size_t reserved() const noexcept { return reserved_; }
    std::size_t allocated() const noexcept { return allocated_; }
    std::size_t used() const noexcept { return reserved_ + allocated_; }
    std::size_t peak() const noexcept { return peak_; }
    void fail_allocation_on(std::size_t ordinal);
    void clear_failure() noexcept { fail_after_.reset(); }

private:
    template<class T> friend class Allocator;
    void before_allocate();
    void release_allocation(std::size_t bytes) noexcept;
    std::size_t limit_;
    std::size_t reserved_ = 0;
    std::size_t allocated_ = 0;
    std::size_t peak_ = 0;
    std::optional<std::size_t> fail_after_;
};
```

There are two different release paths. `Lease::reset` subtracts from `reserved_`;
`release_allocation` subtracts from `allocated_`. Mixing them would make a
plausible total briefly hide an invalid state. Invariant failures in nonthrowing
cleanup terminate instead of underflowing a counter or throwing a second exception.

Start the source file with the complete lease implementation:

```cpp title="src/memory/budget.cpp"
#include "strata/memory/budget.hpp"

#include <algorithm>

namespace strata::memory {

Budget::Lease::Lease(Lease&& other) noexcept
    : owner_(std::exchange(other.owner_, nullptr)),
      bytes_(std::exchange(other.bytes_, 0)) {}

Budget::Lease& Budget::Lease::operator=(Lease&& other) noexcept {
    if (this != &other) {
        reset();
        owner_ = std::exchange(other.owner_, nullptr);
        bytes_ = std::exchange(other.bytes_, 0);
    }
    return *this;
}

void Budget::Lease::reset() noexcept {
    if (owner_) {
        if (bytes_ > owner_->reserved_) { std::terminate(); }
        owner_->reserved_ -= bytes_;
        owner_ = nullptr;
        bytes_ = 0;
    }
}

void Budget::Lease::commit_allocation() noexcept {
    if (!owner_ || bytes_ > owner_->reserved_) { std::terminate(); }
    owner_->reserved_ -= bytes_;
    owner_->allocated_ += bytes_;
    owner_ = nullptr;
    bytes_ = 0;
}
```

Moving transfers both the owner pointer and the byte count with `std::exchange`.
Move assignment first returns the destination's old reservation. Its self-move
guard preserves the one lease rather than returning and then trying to reclaim
its own bytes.

`commit_allocation` changes the classification of bytes, not their total. Since
the amount is already part of the reservation, adding it to `allocated_` cannot
exceed $L$. It then empties the lease. The allocation, rather than that stack
object, is now responsible for eventual release.

Complete the source file:

```cpp title="src/memory/budget.cpp"
Budget::~Budget() noexcept {
    if (used() != 0) { std::terminate(); }
}

Budget::Lease Budget::reserve(std::size_t bytes) {
    if (bytes > limit_ - used()) { throw BudgetExceeded{}; }
    if (bytes == 0) { return {}; }
    reserved_ += bytes;
    peak_ = std::max(peak_, used());
    return Lease{*this, bytes};
}

void Budget::release_allocation(std::size_t bytes) noexcept {
    if (bytes > allocated_) { std::terminate(); }
    allocated_ -= bytes;
}

void Budget::fail_allocation_on(std::size_t ordinal) {
    if (ordinal == 0) { throw std::invalid_argument("allocation ordinal is zero"); }
    fail_after_ = ordinal;
}

void Budget::before_allocate() {
    if (fail_after_ && --*fail_after_ == 0) {
        fail_after_.reset();
        throw std::bad_alloc{};
    }
}

}  // namespace strata::memory
```

The failure seam is deliberately small. `fail_allocation_on(2)` rejects the second
nonempty, successfully admitted allocator request after arming. Rejected budget
reservations and zero-size requests do not consume an ordinal. The injected
failure clears itself, and `clear_failure()` also cancels an unconsumed ordinal.
This counts our allocator calls, not all allocations performed by a process.

A budget destructor with outstanding usage is a lifetime error. It cannot safely
"fix" the situation by freeing other objects' memory. The explicit termination
check catches that incorrect destruction order before later deallocators follow
a dangling budget pointer.

A manually held lease is **not** a credit automatically consumed by a vector.
The allocator below reserves for its own request. Holding another lease while
allocating the same planned bytes would count both reservations. Manual leases
can hold headroom for a separate operation or, as in our tests, create pressure;
only the allocator's private commit transfers its lease into its allocation.

## Put the accounting boundary where capacity is chosen

Suppose a vector has two elements but capacity ten. Charging two elements misses
eight allocated slots. Worse, charging before calling `reserve(10)` with an
assumed growth policy can disagree with the implementation's actual allocation
request. A stateful allocator intercepts that request instead.

An **allocator** is a small object describing how to acquire and return typed
storage. `Allocator<T>::allocate(n)` returns storage for `n` objects of type `T`;
it does not call their constructors. Containers perform construction separately.
`deallocate(pointer,n)` must use an allocator equal to the one that allocated the
storage and the matching count. These are caller/container contracts, not bounds
the deallocator can reconstruct from an arbitrary pointer. See the C++20 draft's
[allocator requirements](https://timsong-cpp.github.io/cppwp/n4861/allocator.requirements).

Our state is one borrowed `Budget*`. The converting constructor accepts
`Allocator<U>` and preserves that pointer when the library needs a different
internal storage type. The `value_type` alias identifies the requested element
type. `std::allocator_traits` supplies standard defaults, including rebinding
`Allocator<T>` to `Allocator<U>`; we do not specialize a standard-library template.

The `propagate_on_container_*` aliases are compile-time Boolean types.
`std::true_type` has a static `value` equal to `true`. They tell a container to
move/copy/swap allocator state along with the allocation ownership when the named
operation needs that state. `is_always_equal` is false because two different
budgets cannot deallocate each other's charges. Equality compares budget identity,
not the numeric limits. The relevant rules are in
[allocator traits](https://timsong-cpp.github.io/cppwp/n4861/allocator.traits.types)
and the [allocator-aware container requirements](https://timsong-cpp.github.io/cppwp/n4861/container.requirements.general).

```cpp title="include/strata/memory/budget.hpp"
template<class T>
class Allocator {
public:
    using value_type = T;
    using propagate_on_container_copy_assignment = std::true_type;
    using propagate_on_container_move_assignment = std::true_type;
    using propagate_on_container_swap = std::true_type;
    using is_always_equal = std::false_type;

    explicit Allocator(Budget& budget) noexcept : budget_(&budget) {}
    template<class U>
    Allocator(const Allocator<U>& other) noexcept : budget_(&other.budget()) {}

    Budget& budget() const noexcept { return *budget_; }
    [[nodiscard]] T* allocate(std::size_t count) {
        if (count > std::numeric_limits<std::size_t>::max() / sizeof(T)) {
            throw std::bad_array_new_length{};
        }
        if (count == 0) { return nullptr; }
        auto lease = budget_->reserve(count * sizeof(T));
        budget_->before_allocate();
        T* result = std::allocator<T>{}.allocate(count);
        lease.commit_allocation();
        return result;
    }
    void deallocate(T* pointer, std::size_t count) noexcept {
        if (count == 0) { return; }
        std::allocator<T>{}.deallocate(pointer, count);
        budget_->release_allocation(count * sizeof(T));
    }
    template<class U>
    bool operator==(const Allocator<U>& other) const noexcept {
        return budget_ == &other.budget();
    }

private:
    Budget* budget_;
};

template<class T>
using Vector = std::vector<T, Allocator<T>>;

}  // namespace strata::memory
#endif
```

The allocation sequence has four steps. First, reject an overflowing
`count * sizeof(T)`. Second, reserve the checked byte count. Third, run the
failure seam and ask `std::allocator<T>` for suitably aligned storage. Finally,
commit the reservation and return the pointer. If either operation in the third
step throws, stack unwinding returns the reservation. There is no effect to undo
in the container because it never received a pointer.

The wrapper delegates raw storage acquisition, not budget policy, to the standard
allocator. This also avoids accidentally losing support for over-aligned types
by calling a mismatched raw `operator new`/`operator delete` pair. The standard
allocator's [allocation interface](https://timsong-cpp.github.io/cppwp/n4861/allocator.members)
specifies storage for its element type; the test uses an `alignas(64)` object.

`Vector<T>` is only an alias for a standard vector with this allocator. It does
not implement a second vector. Construct one by passing its ledger explicitly:

```cpp title="Usage fragment — charged vector"
strata::memory::Budget budget{4096};
strata::memory::Vector<std::uint64_t> values{
    strata::memory::Allocator<std::uint64_t>{budget}};
values.reserve(10);
values.push_back(7);
values.push_back(9);
```

For observed capacity $c$ and element size $s=\operatorname{sizeof}(T)$, the
retained element-storage charge is $c s$, not `values.size() * s`. While a vector
grows, its old allocation and candidate allocation may coexist; both are charged.
Clearing elements does not itself release that capacity. The standard's
[vector capacity rules](https://timsong-cpp.github.io/cppwp/n4861/vector.capacity)
distinguish size, reserve, and reallocation; the implementation must not rely on a
particular growth factor.

There is intentionally no default `Allocator()` constructor. A default-created
charged vector without an explicit allocator therefore fails to compile. This is
useful friction: a new engine buffer should choose its budget, not escape it.

## Charge an arena chunk once, including its header

Chapter 7 already limits an individual arena by `sizeof(Chunk)`. That protects
one memtable, not the sum of two memtables and a table builder. Preserve that local
cap and add the common ledger. `Arena::reserved_bytes()` keeps its existing name:
it measures acquired chunk storage, which belongs to **allocated** bytes in the
new ledger, not to `Budget::reserved()`.

In `include/strata/memory/arena.hpp`, add
`#include "strata/memory/budget.hpp"`. Replace the old inline constructor with:

```cpp title="include/strata/memory/arena.hpp — replace constructor"
Arena(Budget& budget, std::size_t byte_cap) noexcept
    : budget_(budget), cap_(byte_cap) {}
```

Add `Budget& budget_;` in the private data members immediately before
`Chunk* head_ = nullptr;`. Keep `Chunk`, the transaction interface, and all other
members unchanged. The reference does not own the ledger.

In `src/memory/arena.cpp`, add explicit `<memory>` and `<type_traits>` includes.
Replace `drop_head` with this complete function:

```cpp title="src/memory/arena.cpp — replace drop_head"
void Arena::drop_head() noexcept {
    Chunk* old = head_;
    head_ = old->previous;
    reserved_ -= sizeof(Chunk);
    std::destroy_at(old);
    Allocator<Chunk>{budget_}.deallocate(old, 1);
}
```

Destruction ends the `Chunk` object's lifetime; deallocation returns its typed
storage and charge. The old `delete old` cannot remain: an allocation obtained
through the allocator must return through that same allocation model.

Replace the complete `Arena::allocate` function as follows. The search for space,
alignment padding, local-cap check, and existing chunk-failure seam are unchanged.
The new allocation boundary is at the bottom:

```cpp title="src/memory/arena.cpp — replace allocate"
void* Arena::allocate(std::size_t bytes, std::size_t alignment) {
    if (alignment == 0 || (alignment & (alignment - 1)) != 0 ||
        alignment > alignof(std::max_align_t)) {
        throw std::invalid_argument("unsupported arena alignment");
    }
    if (bytes == 0 || bytes > chunk_payload) {
        throw std::length_error("arena request must fit one nonempty chunk");
    }
    auto try_chunk = [this, bytes, alignment]() -> void* {
        if (!head_) {
            return nullptr;
        }
        const auto padding = (alignment - head_->used % alignment) % alignment;
        const auto available = chunk_payload - head_->used;
        if (padding > available || bytes > available - padding) {
            return nullptr;
        }
        auto* result = head_->bytes + head_->used + padding;
        head_->used += padding + bytes;
        used_ += padding + bytes;
        return result;
    };

    if (void* result = try_chunk()) {
        return result;
    }
    if (sizeof(Chunk) > cap_ - reserved_) {
        throw ArenaFull{};
    }
    if (fail_after_ && --*fail_after_ == 0) {
        fail_after_.reset();
        throw std::bad_alloc{};
    }
    static_assert(std::is_nothrow_default_constructible_v<Chunk>);
    auto* fresh = Allocator<Chunk>{budget_}.allocate(1);
    std::construct_at(fresh);
    fresh->previous = head_;
    head_ = fresh;
    reserved_ += sizeof(Chunk);
    return try_chunk();
}
```

`std::construct_at(fresh)` starts the object lifetime in acquired storage. Here
construction is statically known not to throw. That fact justifies the absence of
a catch/deallocate path between allocation and linking. A future throwing `Chunk`
constructor would require such a guard; simply removing the `static_assert` would
not be an equivalent change. Construction/destruction are separate from allocation,
as specified by [construct_at](https://timsong-cpp.github.io/cppwp/n4861/specialized.construct)
and [destroy_at](https://timsong-cpp.github.io/cppwp/n4861/specialized.destroy).

The chunk is linked only after storage acquisition and construction succeed.
A failed common-budget admission therefore leaves the list, local counters, and
transaction checkpoint unchanged. Existing transaction rollback uses `drop_head`,
so abandoning a prepared batch now returns both chunk storage and its ledger
charge. Restoring the bump offset inside a retained chunk does **not** return
that chunk's charge: the allocation still exists.

For example, give two arenas local caps of two chunks each but a shared budget
of one chunk. Once the first arena acquires a chunk, the second must fail before
linking one, despite being below its own local cap. No count of logical nodes
would capture that aggregate constraint correctly.

## Pass the ledger through memtables and builders

In `include/strata/storage/memtable.hpp`, replace only the constructor declaration:

```cpp title="include/strata/storage/memtable.hpp — replace declaration"
MemTable(memory::Budget& budget, std::size_t byte_cap, std::uint32_t seed = 1,
         std::size_t fixed_height = 0);
```

Replace the complete constructor definition in `src/storage/memtable.cpp`.
The node layout, search, preparation, publication, and destruction algorithms stay
unchanged:

```cpp title="src/storage/memtable.cpp — replace constructor"
MemTable::MemTable(memory::Budget& budget, std::size_t byte_cap, std::uint32_t seed,
                   std::size_t fixed_height)
    : arena_(budget, byte_cap), seed_(seed), fixed_height_(fixed_height) {
    static_assert(alignof(Node) <= alignof(std::max_align_t));
    static_assert(sizeof(Node) <= memory::Arena::chunk_payload);
    static_assert(std::is_nothrow_default_constructible_v<Node>);
    static_assert(std::is_trivially_destructible_v<Node>);
    if (seed == 0 || fixed_height > max_height) {
        throw std::invalid_argument("invalid height generator configuration");
    }
}
```

The table builder has two dynamic workspaces. In `include/strata/storage/table.hpp`,
add `#include "strata/storage/block_cache.hpp"`; Part 2 supplies that header and
it includes the budget definitions. Change `block_` and `metadata_` **inside
TableBuilder** from `std::vector<std::byte>` to `memory::Vector<std::byte>`.
Change the constructor and free-function declarations to:

```cpp title="include/strata/storage/table.hpp — replace builder declarations"
// Inside TableBuilder's public section:
TableBuilder(memory::Budget& budget, io::File& file, TableOptions options = {});

// At namespace scope, where write_table was already declared:
TableInfo write_table(memory::Budget& budget, io::File& file, const MemTable& table,
                      TableOptions options = {});
```

Replace the constructor in `src/storage/table.cpp` with the following complete
function. Both workspace allocations precede the first write to the output file.
Failure during the second allocation destroys the first vector automatically:

```cpp title="src/storage/table.cpp — replace builder constructor"
TableBuilder::TableBuilder(memory::Budget& budget, io::File& file, TableOptions options)
    : file_(file), options_(validate_options(options)),
      block_(options_.block_bytes, memory::Allocator<std::byte>{budget}),
      metadata_(metadata_size(options_.block_limit, options_.filter_bytes),
                memory::Allocator<std::byte>{budget}) {
    std::array<std::byte, 1> probe{};
    if (file_.read_up_to(0, probe) != 0) {
        throw std::invalid_argument("builder requires a new empty file");
    }
    std::array<std::byte, header_bytes> header{};
    std::copy(header_magic.begin(), header_magic.end(), header.begin());
    put_le<4>(std::span{header}.subspan(8, 4), 1);
    put_le<4>(std::span{header}.subspan(12, 4), options_.block_bytes);
    put_le<4>(std::span{header}.last(4), crc32(std::span{header}.first(20)));
    file_.write_all(0, header);
}
```

Finally replace the `write_table` definition. It still traverses the frozen
memtable directly; the new argument changes allocation ownership, not the format
or streaming algorithm:

```cpp title="src/storage/table.cpp — replace write_table"
TableInfo write_table(memory::Budget& budget, io::File& file,
                      const MemTable& table, TableOptions options) {
    if (!table.frozen()) { throw std::logic_error("flush input must be frozen"); }
    TableBuilder builder{budget, file, options};
    for (auto cursor = table.begin(); cursor.valid(); cursor.next()) {
        builder.append(cursor.entry());
    }
    return builder.finish();
}
```

These changes do not alter table bytes. The existing independent Python decoder
will continue to check the same format. A successful `finish` still does not
install a table durably or release the builder's workspaces early; they remain
owned until the builder is destroyed.

The important API decision is the first argument. A caller constructs one ledger
for the intended engine domain and passes it to memtables, builders, readers, and
the cache. There is no process-global fallback and no "unlimited if omitted"
mode. The fixture-only migration in Part 3 supplies a shared ledger to the older
tests without changing their assertions.

## Predict the next allocation before running it

Consider a vector that owns 80 charged bytes under a 100-byte ledger. It needs a
new 160-byte allocation to grow. The request fails: there are only 20 admitted
bytes available, and preserving the old allocation requires coexistence during
preparation. Even an eventual 90-byte replacement would fail under these exact
conditions. A final state that fits does not prove that its transition fits.

A cache has the same issue. Evicting first could free enough space, but then a
read or validation failure would have already discarded useful data. In Part 2
we deliberately choose **prepare before eviction**, account for that peak, and
make failure preserve the old cache contents.

Continue with [Part 2 — Recency, pins, and admission](https://25d-228.github.io/cplusplus-learning/chapter-09-part-2.html).

