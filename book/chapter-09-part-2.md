# Chapter 9 — Memory budgets and a pinned block cache

## Part 2 — Recency, pins, and admission

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-09.html) introduced the
ledger and charged allocator. This part implements the cache itself. The byte
ledger answers whether an allocation may exist. The cache answers whether it
should **retain** that allocation after the current reader releases its pin.
These are different decisions.

This is the same Chapter 9, sharing its 3.5-hour implementation allocation.
Append the new-file listings in order. Do not yet rebuild; Part 3 completes the
reader migration and test call sites.

## Share an allocation, not a borrowed address

Two cursors may need the same immutable block. Giving the second cursor a raw
pointer into a cache-owned vector would make eviction a lifetime hazard. Giving
each cursor a copy would preserve lifetime but defeat sharing and multiply the
memory cost.

A **pin** is an owning handle to one shared immutable block. Its ownership is
represented by `std::shared_ptr<const Block>`. Copying the handle adds an owner;
its referent is not copied. Moving transfers a handle and empties the source.
The block is destroyed when its final owning handle disappears. We do not expose
weak pointers, raw ownership transfer, or a mutable block through the public pin
interface. The shared-pointer [destruction rules](https://timsong-cpp.github.io/cppwp/n4861/util.smartptr.shared.dest)
describe the last-owner cleanup.

The cache itself owns one handle for each resident block. Therefore a resident
block with `use_count() == 1` has no reader pins and is eligible for eviction.
That conclusion is valid **only in this exclusive, synchronous implementation**.
The standard's [observer contract](https://timsong-cpp.github.io/cppwp/n4861/util.smartptr.shared.obs)
does not turn `use_count` into synchronization. Shared lifetime does not make the
cache links, budget counters, file backend, or reader counters thread-safe.
Chapter 20 must add a deliberate synchronization design before concurrent use.

The lifetime relationships are:

```text title="Ownership and borrowing"
Budget (must outlive all charged users)
  ^ borrowed by allocators
  |
cache slot array ---- owns ----> resident shared block <---- owns ---- pin
                                         |
                                         owns
                                         |
                                charged frame-byte vector

Cursor owns a pin; its EntryView only borrows bytes from that pinned frame.
```

A pin contains no pointer back to the cache. It can outlive cache destruction,
provided its budget survives. A table cursor still borrows its `TableReader`, so
a cursor cannot outlive that reader merely because its bytes are pinned.

## Give a cached block a stable identity

A file descriptor is a temporary operating-system handle. A path is a name.
Neither is our definition of immutable file identity. Replacing a file at the same
path must not let a cached old block masquerade as the new file's bytes.

Each cache issues nonzero, monotonically increasing `FileId` values. Assign one
value to an immutable file incarnation and pass the same value to readers that
are allowed to share its bytes. Assign a **new** value to a replacement, even when
its path, descriptor, length, and first block offset match the old ones. A new
value for an unchanged file is safe but forfeits hits. Values need only be unique
within this cache's lifetime because its lookup table is not persisted or shared
with another cache. Exhaustion throws instead of wrapping.

`BlockKey` combines that identity with the file offset and encoded frame size.
`knows(id)` later rejects zero and not-yet-issued values; it is a range check, not
proof that a caller associated a token with the right file. The immutable-file
association remains an explicit caller contract. The database catalog in later
chapters will own that association; here the tests own it.

Start `include/strata/storage/block_cache.hpp`:

```cpp title="include/strata/storage/block_cache.hpp"
#ifndef STRATA_STORAGE_BLOCK_CACHE_HPP
#define STRATA_STORAGE_BLOCK_CACHE_HPP

#include "strata/memory/budget.hpp"

#include <concepts>
#include <cstddef>
#include <cstdint>
#include <limits>
#include <memory>
#include <span>
#include <stdexcept>
#include <utility>

namespace strata::storage {

using FileId = std::uint64_t;

struct BlockKey {
    FileId file = 0;
    std::uint64_t offset = 0;
    std::size_t bytes = 0;
    bool operator==(const BlockKey&) const = default;
};

template<class Load>
concept BlockLoader = requires(const Load& load, std::span<std::byte> out) {
    { load(out) } -> std::same_as<void>;
};

namespace cache_detail {
struct Block {
    Block(memory::Budget& budget, std::size_t size)
        : data(size, memory::Allocator<std::byte>{budget}) {}
    memory::Vector<std::byte> data;
    std::size_t charge = 0;
};
std::shared_ptr<Block> make_block(memory::Budget& budget, std::size_t size);
}  // namespace cache_detail
```

`BlockLoader` is a syntactic concept, like the comparator constraint in Chapter 5.
It checks that calling the supplied object with a writable byte span has result
type `void`. It cannot prove that all bytes were filled, that validation was
performed, or that no mutable alias was retained. Those are part of the loader's
contract.

A loader runs synchronously, fills **all** requested bytes, validates the
representation that will be cached, and returns only after success. It may throw.
It must not retain the writable span or reenter the same cache. The actual table
loader in Part 3 does exactly one `read_exact` followed by complete block
validation. There is no hidden background operation.

The `cache_detail::Block` vector uses our allocator. Merely charging that vector
would still omit the shared-ownership bookkeeping and the `Block` object itself.
`std::allocate_shared` accepts an allocator and rebinds it for its allocation
needs, as described in its [creation contract](https://timsong-cpp.github.io/cppwp/n4861/util.smartptr.shared.create).
We supply our allocator there as well; an ordinary `make_shared` would not use
this budget for its control storage.

Continue the header with the pin. Its bytes accessor is lvalue-only, as in the
previous ownership chapters. That rejects a view taken directly from an immediately
destroyed temporary pin, although it cannot prevent every dangling-view mistake:

```cpp title="include/strata/storage/block_cache.hpp"
class BlockPin {
public:
    BlockPin() noexcept = default;
    // Copying a pin shares one immutable block; it does not copy its bytes.
    BlockPin(const BlockPin&) noexcept = default;
    BlockPin& operator=(const BlockPin&) noexcept = default;
    BlockPin(BlockPin&&) noexcept = default;
    BlockPin& operator=(BlockPin&&) noexcept = default;
    explicit operator bool() const noexcept { return static_cast<bool>(block_); }
    void reset() noexcept { block_.reset(); }
    std::span<const std::byte> bytes() const & noexcept {
        return block_ ? std::span<const std::byte>{block_->data}
                      : std::span<const std::byte>{};
    }
    std::span<const std::byte> bytes() const && = delete;
    std::size_t capacity() const noexcept { return block_ ? block_->data.capacity() : 0; }
    std::size_t charge() const noexcept { return block_ ? block_->charge : 0; }

    template<BlockLoader Load>
    static BlockPin load(memory::Budget& budget, std::size_t size, const Load& read) {
        auto candidate = cache_detail::make_block(budget, size);
        read(candidate->data);
        return BlockPin{std::move(candidate)};
    }

private:
    friend class BlockCache;
    explicit BlockPin(std::shared_ptr<const cache_detail::Block> block) noexcept
        : block_(std::move(block)) {}
    std::shared_ptr<const cache_detail::Block> block_;
};
```

`BlockPin::load` keeps mutable ownership private while the loader runs, then
converts it to a const-element shared pointer. The conversion shares the same
allocation. After return, only read-only byte access is available through the pin.
A loader that secretly retained a writable alias would violate the contract; a
const handle cannot undo an alias already leaked by its creator.

If construction or the loader throws, the local `candidate` releases its vector
and control storage. No cache entry has been changed. If the load succeeds but
admission later declines, this same pin becomes the charged bypass result.
There is no second allocation or copy solely to perform bypass.

## Measure all storage belonging to one block

Start `src/storage/block_cache.cpp` with the block factory:

```cpp title="src/storage/block_cache.cpp"
#include "strata/storage/block_cache.hpp"

namespace strata::storage {
namespace cache_detail {
std::shared_ptr<Block> make_block(memory::Budget& budget, std::size_t size) {
    if (size == 0) { throw std::invalid_argument("empty cache block"); }
    const auto before = budget.allocated();
    auto block = std::allocate_shared<Block>(memory::Allocator<Block>{budget}, budget, size);
    // Exclusive accounting: no callback or other thread runs inside this interval.
    block->charge = budget.allocated() - before;
    return block;
}
}  // namespace cache_detail
```

`before` records allocated bytes before `allocate_shared`; the difference after
successful construction includes both the vector capacity and whatever storage
this implementation requests through the rebound allocator. It does not hard-code
a control-block size or assume that `allocate_shared` is required to use exactly
one allocation. The standard recommends combining allocations, but the accounting
follows actual requests.

This subtraction is sound here because the interval is exclusive: the constructor
only creates the block and its vector, with no user callback, unrelated release,
or other thread changing the ledger. The loader runs **after** the charge has
been measured. A concurrent implementation cannot keep this unsynchronized global
before/after method; it needs synchronized or operation-local attribution.

Let $V$ be the vector's requested capacity in bytes and $H$ all other storage
requested through the shared-object allocator. The block charge $B$ is

$$
B = V + H.
$$

`H` includes storage for the `Block` object/control representation as supplied by
the library. It is not the operating-system allocator's hidden bookkeeping.
The tests measure $B$ on the active implementation instead of asserting a
libc++/libstdc++ layout constant.

## Use a fixed slot array and explicit recency links

A cache map can accidentally become its own unbounded allocation problem. Here
lookup metadata is a fixed, charged vector of slots constructed once. A slot
contains a key, one owning handle, and two index links. Lookup scans at most $S$
slots, where $1 \le S \le 4096$ is the configured slot count. This is an explicit
$O(S)$ lookup tradeoff, not an alleged constant-time hash table.

Recency links form a doubly linked list of occupied slots. `newest_` is the most
recently accessed entry, `oldest_` the least recent. `previous` points toward the
newer end; `next` points toward the older end. `none`, the largest `size_t`, is a
sentinel, never an array index. Empty slots remain outside the list.

Complete the header:

```cpp title="include/strata/storage/block_cache.hpp"
class BlockCache {
    static constexpr std::size_t none = std::numeric_limits<std::size_t>::max();
    struct Slot {
        BlockKey key{};
        std::shared_ptr<const cache_detail::Block> block;
        std::size_t previous = none;
        std::size_t next = none;
    };
public:
    static constexpr std::size_t max_slots = 4096;
    struct Statistics {
        std::uint64_t hits = 0;
        std::uint64_t misses = 0;
        std::uint64_t evictions = 0;
        std::uint64_t bypasses = 0;
    };
    BlockCache(memory::Budget& budget, std::size_t slots, std::size_t byte_limit);
    BlockCache(const BlockCache&) = delete;
    BlockCache& operator=(const BlockCache&) = delete;
    BlockCache(BlockCache&&) = delete;
    BlockCache& operator=(BlockCache&&) = delete;

    FileId new_file_id();
    bool knows(FileId id) const noexcept { return id != 0 && id <= last_id_; }
    memory::Budget& budget() const noexcept { return budget_; }
    std::size_t metadata_charge() const noexcept { return slots_.capacity() * sizeof(Slot); }
    std::size_t resident_charge() const noexcept { return metadata_charge() + block_charge_; }
    std::size_t entries() const noexcept { return entries_; }
    Statistics statistics() const noexcept { return statistics_; }
    bool contains(BlockKey key) const noexcept { return lookup(key) != none; }
    void trim() noexcept;
    void check_structure() const;

    template<BlockLoader Load>
    BlockPin get(BlockKey key, const Load& read) {
        if (!knows(key.file) || key.bytes == 0) {
            throw std::invalid_argument("invalid cache block key");
        }
        const auto hit = lookup(key);
        if (hit != none) {
            ++statistics_.hits;
            touch(hit);
            return BlockPin{slots_[hit].block};
        }
        ++statistics_.misses;
        auto candidate = BlockPin::load(budget_, key.bytes, read);
        admit(key, candidate.block_);
        return candidate;
    }

private:
    std::size_t lookup(BlockKey key) const noexcept;
    std::size_t empty_slot() const noexcept;
    void unlink(std::size_t slot) noexcept;
    void prepend(std::size_t slot) noexcept;
    void touch(std::size_t slot) noexcept;
    void evict(std::size_t slot) noexcept;
    void admit(BlockKey key, const std::shared_ptr<const cache_detail::Block>& block) noexcept;
    memory::Budget& budget_;
    memory::Vector<Slot> slots_;
    std::size_t byte_limit_;
    std::size_t block_charge_ = 0;
    std::size_t entries_ = 0;
    std::size_t newest_ = none;
    std::size_t oldest_ = none;
    FileId last_id_ = 0;
    Statistics statistics_{};
};

}  // namespace strata::storage
#endif
```

The byte limit includes the slot vector's retained allocation **and** resident
block charges. It does not reserve that entire capacity up front. The shared
budget admits actual allocations as they occur. Reader metadata, bypass blocks,
and temporary candidates belong to the shared budget but are not resident-cache
storage.

A hit performs no allocation and never calls the loader. It moves the resident
slot to the newest position and returns another handle to the same block.
A miss increments `misses`, constructs a candidate, calls `admit`, and returns
that candidate whether it became resident or bypassed. Admission refusal and
allocation failure are intentionally different outcomes.

The following constructors and bounded scans continue the source file:

```cpp title="src/storage/block_cache.cpp"
namespace {
std::size_t checked_slots(std::size_t slots) {
    if (slots == 0 || slots > BlockCache::max_slots) {
        throw std::invalid_argument("cache slot count out of range");
    }
    return slots;
}
}  // namespace

BlockCache::BlockCache(memory::Budget& budget, std::size_t slots, std::size_t byte_limit)
    : budget_(budget), slots_(checked_slots(slots), memory::Allocator<Slot>{budget}),
      byte_limit_(byte_limit) {
    if (metadata_charge() > byte_limit_) {
        throw std::invalid_argument("cache limit cannot hold its slot array");
    }
}

FileId BlockCache::new_file_id() {
    if (last_id_ == std::numeric_limits<FileId>::max()) {
        throw std::overflow_error("cache file identifiers exhausted");
    }
    return ++last_id_;
}

std::size_t BlockCache::lookup(BlockKey key) const noexcept {
    for (std::size_t i = 0; i < slots_.size(); ++i) {
        if (slots_[i].block && slots_[i].key == key) { return i; }
    }
    return none;
}

std::size_t BlockCache::empty_slot() const noexcept {
    for (std::size_t i = 0; i < slots_.size(); ++i) {
        if (!slots_[i].block) { return i; }
    }
    return none;
}
```

The constructor checks slot count before constructing the slot array. The shared
budget checks its allocation before memory is obtained. If the resulting retained
array cannot fit the requested cache byte limit, construction throws and the
vector releases its charge during unwinding. A cache can validly have no room for
a resident block after metadata; its reads then bypass subject to the shared
budget.

The slot vector never grows. Its capacity charge is stable, so the resident limit
is not undermined by rehashing, list-node allocation, or iterator invalidation.
The cache object itself and local loop variables are fixed control objects outside
the dynamic allocation ledger.

## Update both ends of a recency list

Here are the complete link operations:

```cpp title="src/storage/block_cache.cpp"
void BlockCache::unlink(std::size_t slot) noexcept {
    const auto previous = slots_[slot].previous;
    const auto next = slots_[slot].next;
    if (previous == none) { newest_ = next; }
    else { slots_[previous].next = next; }
    if (next == none) { oldest_ = previous; }
    else { slots_[next].previous = previous; }
    slots_[slot].previous = none;
    slots_[slot].next = none;
}

void BlockCache::prepend(std::size_t slot) noexcept {
    slots_[slot].previous = none;
    slots_[slot].next = newest_;
    if (newest_ != none) { slots_[newest_].previous = slot; }
    else { oldest_ = slot; }
    newest_ = slot;
}

void BlockCache::touch(std::size_t slot) noexcept {
    unlink(slot);
    prepend(slot);
}
```

To unlink an entry, its newer neighbor must skip forward to its older neighbor,
and its older neighbor must skip back to its newer neighbor. At a boundary,
update the corresponding head or tail instead. Clearing the removed slot's links
helps make the empty-slot invariant observable.

`prepend` places a detached occupied slot at the newest end. On an empty list it
also becomes the oldest entry. `touch` composes unlink and prepend, including the
one-entry case. There is no special-case shortcut that forgets a tail update.

For two unpinned residents, access order is visible even when contents never change:

| Operation | Recency, newest to oldest |
|---|---|
| Load A | A |
| Load B | B, A |
| Hit A | A, B |
| Load C under a two-block byte limit | C, A; B is evicted |

Removing `touch(hit)` later will preserve every byte's validity while choosing
the wrong victim. That is a logical defect for our test to detect, not a memory
violation for a sanitizer to infer.

## Plan eviction before changing ownership

Define $M$ as charged slot metadata, $C$ as the configured cache byte limit,
$P=C-M$ as the capacity available for resident blocks, and $D$ as their current
charge. For a candidate charged $B$ bytes, the current free capacity is
$F=P-D$. The additional bytes needed are

$$
N = \max(0, B-F).
$$

Admission requires both enough releasable **unpinned** bytes to cover $N$ and at
least one empty or evictable slot. An empty slot alone is not enough for byte
capacity; free bytes alone are not enough for slot capacity. A candidate larger
than $P$ can never become resident.

Do not begin evicting while searching for enough space. Suppose A is pinned and
B is unpinned, but the candidate requires both blocks' worth of capacity. Evicting
B and only then discovering A is pinned would discard B without achieving
admission. Instead, a first pass plans over the recency list without mutation.
Only a feasible plan enters the nonthrowing second pass.

Append the complete eviction and admission functions:

```cpp title="src/storage/block_cache.cpp"
void BlockCache::evict(std::size_t slot) noexcept {
    if (slots_[slot].block.use_count() != 1) { std::terminate(); }
    unlink(slot);
    block_charge_ -= slots_[slot].block->charge;
    --entries_;
    slots_[slot].block.reset();
    slots_[slot].key = {};
    ++statistics_.evictions;
}

void BlockCache::admit(BlockKey key,
        const std::shared_ptr<const cache_detail::Block>& block) noexcept {
    const auto capacity = byte_limit_ - metadata_charge();
    if (block->charge > capacity) { ++statistics_.bypasses; return; }
    const auto free = capacity - block_charge_;
    const auto need = block->charge > free ? block->charge - free : 0;
    std::size_t releasable = 0;
    bool have_slot = entries_ < slots_.size();
    for (auto i = oldest_; i != none && (releasable < need || !have_slot);
         i = slots_[i].previous) {
        if (slots_[i].block.use_count() == 1) {
            releasable += slots_[i].block->charge;
            have_slot = true;
        }
    }
    if (releasable < need || !have_slot) { ++statistics_.bypasses; return; }

    // No allocation, user callback, or throwing operation remains after this point.
    for (auto i = oldest_; block->charge > capacity - block_charge_ ||
                          entries_ == slots_.size();) {
        if (i == none) { std::terminate(); }
        const auto previous = slots_[i].previous;
        if (slots_[i].block.use_count() == 1) { evict(i); }
        i = previous;
    }
    const auto destination = empty_slot();
    if (destination == none) { std::terminate(); }
    slots_[destination].key = key;
    slots_[destination].block = block;
    block_charge_ += block->charge;
    ++entries_;
    prepend(destination);
}
```

`releasable` never exceeds the charge of the existing resident blocks, so it stays
within the cache limit. `have_slot` becomes true when the plan encounters its
first unpinned victim, even when $N=0$ and the only problem is a full slot array.
If either requirement remains unsatisfied, the candidate bypasses and every old
entry stays in place.

In the commit pass, save the next index to visit **before** unlinking the current
slot. Eviction destroys only the cache's sole handle, so the block and its charge
really are released. A pinned slot is skipped, even if it is the least recent
entry. When enough capacity and a slot exist, installing the already prepared
shared pointer cannot allocate. The loader and all potentially throwing work
have finished before the first eviction.

The guarantee is about old entries, their contents, and their recency order on a
failed miss. Diagnostic counters are not rolled back: a failed miss is still a
miss, and a candidate may raise the budget's admitted high-water mark before
being destroyed. Do not test diagnostic statistics as if the entire operation
had never been attempted.

## Bypass is still memory admission

Consider this **measured layout example from Clang 17/libstdc++ on Linux**, not a
portable ABI promise: two slots occupy 112 requested bytes, and a 64-byte frame
has total shared-block charge 128 bytes. For two retained blocks, the cache limit
is therefore $112+2\times128=368$ bytes. The test calculates these values at runtime
on the native toolchain instead of assuming them.

A new 64-byte-frame candidate needs another 128 bytes before anything is evicted:

$$
U_{\text{candidate}} = 112 + 128 + 128 + 128 = 496.
$$

If both old entries are pinned, the candidate is returned without cache retention.
Its bytes remain charged while any copy of that bypass pin survives. Dropping the
last bypass pin returns 128 bytes. If B is unpinned instead, successful admission
replaces B and returns the shared-budget usage to 368 bytes after commit.

Dropping the last **reader** pin of a resident block does not immediately free
it: the cache still owns one handle. It merely makes that block evictable. This
is why an ownership count and a "currently used by a reader" count are related
but not identical.

A shared budget of exactly 368 bytes would reject the candidate before eviction,
even though its eventual resident state could fit. That is intentional. This
implementation preserves old entries on failure rather than evicting first to
make an uncertain read possible. Configure candidate headroom, reduce other live
memory, or explicitly trim unpinned entries before another attempt. Do not hide
an unbounded allocation in an "emergency" bypass path.

## Release idle entries and inspect the structure

`trim` is an explicit destructive policy operation: release every currently
unpinned resident. Unlike a failed insertion, the caller requested those removals.
Pinned residents remain. Cache destruction, in contrast, drops all of the cache's
handles; surviving reader pins can still keep their blocks alive.

Complete the source file with trim and a nonallocating structural check:

```cpp title="src/storage/block_cache.cpp"
void BlockCache::trim() noexcept {
    for (auto i = oldest_; i != none;) {
        const auto previous = slots_[i].previous;
        if (slots_[i].block.use_count() == 1) { evict(i); }
        i = previous;
    }
}

void BlockCache::check_structure() const {
    auto require = [](bool condition) {
        if (!condition) { throw std::logic_error("invalid cache structure"); }
    };
    std::size_t seen = 0;
    std::size_t charge = 0;
    auto previous = none;
    for (auto i = newest_; i != none; i = slots_[i].next) {
        require(i < slots_.size() && seen < slots_.size());
        const auto& slot = slots_[i];
        require(slot.block && slot.previous == previous && knows(slot.key.file));
        require(slot.key.bytes == slot.block->data.size());
        ++seen;
        charge += slot.block->charge;
        previous = i;
    }
    require(previous == oldest_ && seen == entries_ && charge == block_charge_);
    std::size_t occupied = 0;
    for (const auto& slot : slots_) {
        if (slot.block) { ++occupied; }
        else { require(slot.previous == none && slot.next == none); }
    }
    require(occupied == seen && resident_charge() <= byte_limit_);
}

}  // namespace strata::storage
```

The checker follows the list with a strict slot-count bound, validates back-links
and endpoint agreement, recounts resident charges, and compares occupied slots
with linked slots. It throws on a structural defect; it does not run a repair
algorithm or allocate another index to conceal the defect.

The overall miss cost includes loading and validating the frame, plus a bounded
number of slot/list scans. A cache hit avoids physical I/O and a fresh allocation
but still performs a linear slot search. Increasing $S$ without measuring lookup
cost would be a poor way to tune this design. Changing to a fixed open-addressed
lookup table later would require explicit collision/tombstone handling and a
charged capacity; `unordered_map` would not make those costs disappear.

Continue with [Part 3 — Reader integration and pressure tests](https://25d-228.github.io/cplusplus-learning/chapter-09-part-3.html).

