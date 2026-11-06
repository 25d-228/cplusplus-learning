# Chapter 12 — Immutable snapshots and merged reads

## Part 1 — Own a complete read boundary

An atomic write batch prevents half of a batch from becoming visible. It does
not make two separate current-state reads repeatable. Read account 10, commit a
batch, then read event 101: Chapter 11's `Database::get` can legally combine the
old account with the new event. Each call is individually correct; their combined
answer belongs to no single committed state.

A read view changes that contract. It captures one sequence and the exact owners
needed to answer all reads at that sequence. Later writes use a different active
memtable. A later flush can change the database's serving set without replacing
anything inside an existing view.

> **Learning objectives**
> Build ownership that survives database destruction; distinguish shared ownership
> from frozen contents; admit a bounded immutable source set before rotation;
> implement ordered point and range reads with a hand-written heap; and make
> failed cursor advancement terminal rather than silently skipping data.

**Prerequisites:** the completed Chapters 7–11, including Chapter 9's charged
constructors and pinned table reader, and Chapter 11's sealed-log checks and
manifest installation. Keep the language-foundation programs and all existing
tests. This chapter has **4.5 implementation hours across its three parts**:
1.25 for ownership/capture, 2.0 for merged lookup/scan, and 1.25 for retention and
visibility tests. Reading is not included. Nothing in this chapter starts threads.

[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-12-part-2.html)
implements the merge. [Part 3](https://25d-228.github.io/cplusplus-learning/chapter-12-part-3.html)
supplies the runnable checks and build changes. Finish all three parts before
building the new executable. They form one chapter, not three separate projects.

## What the view promises

Use the roadmap's batch. Account 10 changes from region 1 to region 3, event 101
changes from 250 to 900 cents, and event 102 is deleted. The changes share sequence 2.

| Read boundary | Account 10 | Event 101 | Event 102 |
|---|---:|---:|---|
| View captured at sequence 1 | Region 1 | 250 cents | Present |
| View captured at sequence 2 | Region 3 | 900 cents | Absent |
| Sequence-1 view after both generations are flushed | Region 1 | 250 cents | Present |

A view returns owning point results and creates independent range cursors. A cursor
keeps the view alive even after its original handle is reset. Both may outlive the
`Database` facade. They cannot be used concurrently yet: the cache, counters,
budget, source readers, and capture protocol remain synchronous and nonreentrant.
Reference counting is not a replacement for the synchronization chapter.

There is no arbitrary `snapshot_at(old_sequence)` API and no read-write
transaction. Capturing freezes the currently published boundary; retaining that
handle preserves it. The distinction between committed visibility and a stable
read snapshot also appears in SQLite's [isolation explanation](https://www.sqlite.org/isolation.html).
Here the implementation is our own immutable-generation protocol, not SQLite's
page/WAL mechanism.

## Draw every ownership edge, including the ledger

Chapter 11's table reader owns its descriptor and metadata, but borrows its cache
and budget. Keeping only a `shared_ptr<TableReader>` after destroying the database
would therefore leave two dangling dependencies. Keeping an arena's node address
is even less sufficient: an address does not retain its allocation.

The new lifetime graph is:

```text title="Owning edges; borrowed edges are described below"
Database ---------> Context ---------> Budget
   |                   |                 ^
   |                   +-> Directory     |
   |                   |   and LOCK      |
   |                   +-> BlockCache    |
   +-> current WAL/table owners          |
                                         |
ReadView ---------> const State ----------+
                         |                |
                         +-> Context      |
                         +-> sealed WAL/table owners

ReadCursor ------> Scan ------> const State
                     +-> per-source positions and frame pins
                     +-> one inline owning output row

SharedAllocator copies in owner control storage ------> Budget
```

The last line matters during destruction. `State` can be the last owner of its
context. Destroying its members may destroy that context; only afterward does the
shared-pointer implementation finish releasing storage for the state itself. An
allocator that merely borrowed `Budget*` would not, by itself, keep that final
deallocation safe. We give the allocator its own shared budget owner.

There is no cycle: neither `Budget` nor `Context` owns a view or a scan. The retention
registry stores identities and counts, not owning pointers back to states.
Table-reader borrows remain valid because the state retains the context, and a
scan destroys positions and pins before dropping its state.

The directory lock also belongs to the context. Destroying the database while a
view exists does **not** release the lock. This intentionally prevents another
cooperating process from reopening and modifying files while escaped readers are
still using them. Release the final view/cursor before reopening that directory.
This conservative lifecycle is part of the interface, not a surprising leak.

## Let the allocator retain its budget

`std::shared_ptr<T>` copies an ownership handle; it does not copy `T`. A
`shared_ptr<const T>` denies mutation through that access path, but cannot stop a
separate mutable alias from changing the same object. We will combine const
state ownership with actual sealing. See the standard draft's
[shared-pointer constructors](https://eel.is/c++draft/util.smartptr.shared.const)
for ownership-sharing conversions; do not manufacture a second independent
shared pointer from an existing owner's raw pointer.

Create `include/strata/memory/shared_allocator.hpp`. The new allocator delegates
allocation sizes, failure injection, and ledger updates to Chapter 9's allocator.
It changes only **who retains the ledger**. The complete file is:

```cpp title="include/strata/memory/shared_allocator.hpp"
#ifndef STRATA_MEMORY_SHARED_ALLOCATOR_HPP
#define STRATA_MEMORY_SHARED_ALLOCATOR_HPP

#include "strata/memory/budget.hpp"

#include <cstddef>
#include <memory>
#include <stdexcept>
#include <type_traits>
#include <utility>

namespace strata::memory {

template<class T>
class SharedAllocator {
public:
    using value_type = T;
    using propagate_on_container_copy_assignment = std::true_type;
    using propagate_on_container_move_assignment = std::true_type;
    using propagate_on_container_swap = std::true_type;
    using is_always_equal = std::false_type;

    explicit SharedAllocator(std::shared_ptr<Budget> owner) : owner_(std::move(owner)) {
        if (!owner_) { throw std::invalid_argument("missing budget owner"); }
    }
    template<class U>
    SharedAllocator(const SharedAllocator<U>& other) noexcept : owner_(other.owner()) {}
    const std::shared_ptr<Budget>& owner() const noexcept { return owner_; }
    [[nodiscard]] T* allocate(std::size_t count) {
        return Allocator<T>{*owner_}.allocate(count);
    }
    void deallocate(T* pointer, std::size_t count) noexcept {
        Allocator<T>{*owner_}.deallocate(pointer, count);
    }
    template<class U>
    bool operator==(const SharedAllocator<U>& other) const noexcept {
        return owner_.get() == other.owner().get();
    }
private:
    std::shared_ptr<Budget> owner_;
};

}  // namespace strata::memory
#endif
```

The converting constructor permits rebinding from `SharedAllocator<T>` to
`SharedAllocator<U>` while retaining the same ledger. `allocate_shared` may
allocate an implementation-defined control/object type rather than exactly `T`;
its [creation requirements](https://eel.is/c++draft/util.smartptr.shared.create)
use a copied, rebound allocator. Consequently the ledger records the real request,
not a guessed `sizeof(T)` plus an invented control-block constant.

Copying the allocator copies a shared pointer without allocating another budget.
`deallocate` still has a live `owner_` when it returns storage and subtracts the
charge. Only afterward can releasing the allocator's own copy destroy the budget.
The null-owner check prevents constructing a context with an unusable ledger.

The caller creates the small root ledger with `std::make_shared<Budget>(limit)`.
That bootstrap object/control storage is outside its own accounting. Context,
view, scan, source-owner controls, metadata, cache frames, and arena chunks are
charged. Call-stack space, filesystem caches, allocator overhead outside recorded
requests, and fixture containers are still outside the ledger. It remains an
engine allocation budget, not a process-resident-memory limit.

Lower-level `MemTable`, `TableReader`, and `BlockPin` APIs retain their Chapter 9
borrowed-budget contracts. Only the database-level lifetime boundary is changing;
we are not adding a hidden global budget or duplicating the accounting algorithm.

## A source set has a fixed shape

Create `include/strata/storage/read_view.hpp`. Append its following blocks in order.
The source cap is the manifest's existing table-plus-log cap, not an unbounded
vector of every file ever created. A source contains exactly one owning pointer:
a sealed WAL/memtable owner or an immutable table reader.

```cpp title="include/strata/storage/read_view.hpp"
#ifndef STRATA_STORAGE_READ_VIEW_HPP
#define STRATA_STORAGE_READ_VIEW_HPP

#include "strata/io/directory.hpp"
#include "strata/memory/shared_allocator.hpp"
#include "strata/storage/manifest.hpp"
#include "strata/storage/wal.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <memory>
#include <optional>
#include <span>
#include <string>
#include <utility>

namespace strata::storage {

struct ReadLimits {
    std::size_t snapshots = 16;
    std::size_t retained_sources = 64;
};
struct ReadRetention {
    std::size_t snapshots = 0;
    std::size_t sources = 0;
};

namespace read_detail {
inline constexpr std::size_t max_sources = Manifest::max_tables + Manifest::max_logs;
inline constexpr std::size_t max_snapshots = 32;
inline constexpr std::size_t max_retained = 128;

struct Source {
    std::shared_ptr<const WalMemTable> memory;
    std::shared_ptr<TableReader> disk;
    const void* identity() const noexcept {
        return memory ? static_cast<const void*>(memory.get())
                      : static_cast<const void*>(disk.get());
    }
};
```

`Source::identity()` is an in-process complete-object identity used only while
that object is retained. It is neither the durable file ID nor the cache binding
ID. Once its final registry reference disappears, that address may be reused.
We never carry the raw identity across reopening or use it to locate a file.

The memtable path is `shared_ptr<const WalMemTable>`: read code cannot call its
writer methods. The disk path remains `shared_ptr<TableReader>` because reading
updates block-read counters and cache state. Its underlying table bytes are
immutable even though its bookkeeping is not. Constness is not transitive through
a pointer member; the sealed-owner invariant is what protects stored rows.

Append the context and state declarations:

```cpp title="include/strata/storage/read_view.hpp"
class Context {
public:
    Context(std::shared_ptr<memory::Budget> owner, const std::string& directory_name,
            std::size_t cache_slots, std::size_t cache_bytes, ReadLimits limits);
    void check_admission(std::span<const Source> sources) const;
    void retain(std::span<const Source> sources) noexcept;
    void release(std::span<const Source> sources) noexcept;
    ReadRetention retention() const noexcept { return counts_; }
    std::shared_ptr<memory::Budget> budget;
    io::Directory directory;
    BlockCache cache;
private:
    struct Slot { const void* identity = nullptr; std::size_t references = 0; };
    std::size_t slot_for(const void* identity) const noexcept;
    ReadLimits limits_;
    ReadRetention counts_{};
    std::array<Slot, max_retained> slots_{};
};

struct State {
    State(std::shared_ptr<Context> owner, std::uint64_t visible,
          std::span<const Source> sources);
    ~State() noexcept;
    State(const State&) = delete;
    State& operator=(const State&) = delete;
    void activate() noexcept;
    std::span<const Source> sources() const noexcept { return {held.data(), count}; }
    std::shared_ptr<Context> context;
    std::uint64_t sequence;
    std::array<Source, max_sources> held{};
    std::size_t count;
    bool active = false;
};
struct Scan;
}  // namespace read_detail
```

`Context` owns its budget before constructing the directory and cache. The
member declaration order makes cache destruction precede directory/lock release,
which in turn precedes releasing the budget member. `State` declares the context
before its held sources, so source destruction happens first. These are concrete
uses of [reverse member destruction order](https://eel.is/c++draft/class.dtor).
The owning allocator separately protects the later control-storage deallocation.

A state is prepared privately with `active == false`. It is not admitted to the
registry until the freeze/install step succeeds. Destruction of an unactivated
candidate releases its source copies without decrementing counters that were
never incremented. An activated state unregisters once, when its final shared
owner disappears—not once for every copied `ReadView` handle.

The array has 20 slots with the current manifest caps. Its unused shared pointers
are empty. The actual `count` bounds every traversal. No state stores a borrowed
span into the caller's source array; the constructor copies the owning handles.

## Separate a repeatable view from a mutable cursor

Finish the header with the public handles:

```cpp title="include/strata/storage/read_view.hpp"
class ReadCursor {
public:
    ReadCursor() noexcept = default;
    ReadCursor(const ReadCursor&) = delete;
    ReadCursor& operator=(const ReadCursor&) = delete;
    ReadCursor(ReadCursor&&) noexcept = default;
    ReadCursor& operator=(ReadCursor&&) noexcept = default;
    bool valid() const noexcept;
    EntryView entry() const &;
    EntryView entry() const && = delete;
    void next() &;
    void reset() noexcept { scan_.reset(); }
private:
    friend class ReadView;
    explicit ReadCursor(std::shared_ptr<read_detail::Scan> scan) noexcept
        : scan_(std::move(scan)) {}
    std::shared_ptr<read_detail::Scan> scan_;
};

class ReadView {
public:
    ReadView() noexcept = default;
    bool valid() const noexcept { return static_cast<bool>(state_); }
    void reset() noexcept { state_.reset(); }
    std::uint64_t sequence() const;
    std::size_t source_count() const;
    ReadRetention retention() const;
    std::optional<OwnedEntry> get(Key key) const;
    ReadCursor scan(Key lower, std::optional<Key> upper = std::nullopt) const;
private:
    friend class Database;
    explicit ReadView(std::shared_ptr<const read_detail::State> state) noexcept
        : state_(std::move(state)) {}
    void require_valid() const;
    std::shared_ptr<const read_detail::State> state_;
};

}  // namespace strata::storage
#endif
```

A default or moved-from view is empty. Copying a view intentionally shares one
state and one admission slot. `reset()` releases that handle; another copy or
cursor can keep the state alive. `get` returns a small owning `OwnedEntry`, so it
needs no rvalue restriction. `scan` also owns what it needs, allowing a temporary
view to create a longer-lived cursor.

A cursor has one mutable position and is move-only. Its internal `shared_ptr` is
used for the allocator-backed factory and lifetime cleanup; the public API does
not let clients copy the traversal position. The forward declaration of `Scan`
keeps the heap/position layout in the implementation file. Shared-pointer handles
can be declared with an incomplete pointee type; the allocation is performed
later where that type is complete.

`entry()` returns a borrow into the cursor's owning row. The reference-qualified
accessor rejects a direct borrow from a temporary cursor. A previously returned
span must still not outlive the cursor or a call that advances/resets/replaces it.
Qualifier restrictions are useful, not a complete lifetime proof.

## Count the union, not the number of pointer copies

Let $V$ be the number of admitted state objects, $S_i$ the set of source identities
held by state $i$, and $R$ the number of distinct retained source identities.
The limits apply to:

$$
V \leq V_{\max}, \qquad R = \left|\bigcup_{i=1}^{V} S_i\right| \leq R_{\max}.
$$

Copies of a view do not add another $S_i$. Capturing a fresh view over the same
sources adds one state but no new source identity. A cursor retains its state's
slot even after reaching end-of-range; `reset` or destruction ends that retention.

Suppose view A holds memtable M1. The database flushes M1 into table T1, then
writes M2. A new view would hold T1 and M2, while A still owns M1. With a retained
source cap of two, the union would contain three identities, so capture must fail.
After A's final owner disappears, capturing T1 and M2 fits. Replacing A's source
with T1 behind its back would hide retention rather than implement it.

The registry has at most 128 slots. Checking admission scans this bounded array;
it does not allocate a map. Its cost is bounded by the small configured source
caps, and it is not advertised as constant-time hashing.

Create `src/storage/read_view.cpp`. Start with complete headers and helpers:

```cpp title="src/storage/read_view.cpp"
#include "strata/storage/read_view.hpp"

#include <algorithm>
#include <exception>
#include <limits>
#include <stdexcept>
#include <utility>

namespace strata::storage {
namespace {
std::shared_ptr<memory::Budget> checked_owner(std::shared_ptr<memory::Budget> owner) {
    if (!owner) { throw std::invalid_argument("missing budget owner"); }
    return owner;
}
void valid_key(Key key) {
    if (key.relation != Relation::events && key.relation != Relation::accounts) {
        throw std::invalid_argument("unknown read relation");
    }
}
OwnedEntry own(EntryView entry) {
    OwnedEntry result;
    result.key = entry.key;
    result.kind = entry.kind;
    result.length = entry.value.size();
    if (result.length > result.bytes.size()) { throw FormatError("oversized read row"); }
    std::copy(entry.value.begin(), entry.value.end(), result.bytes.begin());
    return result;
}
}  // namespace
```

`checked_owner` returns an owning handle after checking it, rather than returning
a reference to a nullable pointer's target. `own` copies at most the 30-byte event
payload into an inline result. It is also the lifetime boundary used later when
a cursor is about to advance and invalidate its source span.

Open the internal namespace and implement admission:

```cpp title="src/storage/read_view.cpp"
namespace read_detail {
Context::Context(std::shared_ptr<memory::Budget> owner, const std::string& directory_name,
                 std::size_t cache_slots, std::size_t cache_bytes, ReadLimits limits)
    : budget(checked_owner(std::move(owner))),
      directory(directory_name), cache(*budget, cache_slots, cache_bytes), limits_(limits) {
    if (limits.snapshots == 0 || limits.snapshots > max_snapshots ||
        limits.retained_sources == 0 || limits.retained_sources > max_retained) {
        throw std::invalid_argument("invalid snapshot retention limits");
    }
}
std::size_t Context::slot_for(const void* identity) const noexcept {
    for (std::size_t i = 0; i < slots_.size(); ++i) {
        if (slots_[i].identity == identity) { return i; }
    }
    return slots_.size();
}
void Context::check_admission(std::span<const Source> sources) const {
    if (sources.size() > max_sources) { throw std::length_error("too many view sources"); }
    if (counts_.snapshots == limits_.snapshots) {
        throw std::length_error("snapshot states exhausted");
    }
    std::size_t added = 0;
    for (std::size_t i = 0; i < sources.size(); ++i) {
        const auto& source = sources[i];
        if (static_cast<bool>(source.memory) == static_cast<bool>(source.disk)) {
            throw std::logic_error("source must own exactly one generation");
        }
        for (std::size_t j = 0; j < i; ++j) {
            if (sources[j].identity() == source.identity()) {
                throw std::logic_error("duplicate source in view");
            }
        }
        if (slot_for(source.identity()) == slots_.size()) { ++added; }
    }
    if (added > limits_.retained_sources - counts_.sources) {
        throw std::length_error("retained source slots exhausted");
    }
}
```

Each source must select exactly one alternative. A null/null source or a
memory/disk pair is a programming error; duplicate source identities inside one
state are also rejected. That makes the registration loop's reference counts
unambiguous. `added` counts only identities absent from the existing union.

The subtraction test avoids unchecked addition. Existing invariants guarantee
`counts_.sources <= limits_.retained_sources`, so comparing `added` with the
remaining slots cannot underflow. Capacity rejection performs no allocation,
file operation, source freeze, identifier consumption, or serving publication.

A raw pointer comparison here tests identity only; it does not dereference a
stale address. The state still owns every source while the registry contains its
identity. There is no weak-pointer polling or `use_count()`-based synchronization.

Append registration, release, and state construction:

```cpp title="src/storage/read_view.cpp"
void Context::retain(std::span<const Source> sources) noexcept {
    for (const auto& source : sources) {
        auto slot = slot_for(source.identity());
        if (slot == slots_.size()) {
            slot = slot_for(nullptr);
            if (slot == slots_.size()) { std::terminate(); }
            slots_[slot].identity = source.identity();
            ++counts_.sources;
        }
        ++slots_[slot].references;
    }
    ++counts_.snapshots;
}
void Context::release(std::span<const Source> sources) noexcept {
    for (const auto& source : sources) {
        const auto slot = slot_for(source.identity());
        if (slot == slots_.size() || slots_[slot].references == 0) { std::terminate(); }
        if (--slots_[slot].references == 0) {
            slots_[slot] = {};
            --counts_.sources;
        }
    }
    if (counts_.snapshots == 0) { std::terminate(); }
    --counts_.snapshots;
}
State::State(std::shared_ptr<Context> owner, std::uint64_t visible,
             std::span<const Source> sources)
    : context(std::move(owner)), sequence(visible), count(sources.size()) {
    if (count > held.size()) { throw std::length_error("view source bound"); }
    std::copy(sources.begin(), sources.end(), held.begin());
}
State::~State() noexcept {
    if (active) { context->release(sources()); }
}
void State::activate() noexcept {
    if (active) { std::terminate(); }
    for (const auto& source : sources()) {
        if (source.memory && !source.memory->table().frozen()) { std::terminate(); }
    }
    context->retain(sources());
    active = true;
}
```

`retain` runs only after a successful preflight and with no intervening reentrant
operation. Therefore a free registry slot is guaranteed. Its termination guards
mark internal invariant failures, not ordinary admission errors; ordinary pressure
was already reported by `check_admission`. The maximum reference count for a
source is the maximum admitted state count, 32, not the number of handle copies.

`activate` additionally verifies that every captured memtable is actually frozen.
A conversion to `shared_ptr<const WalMemTable>` did not freeze it. This guard catches
a capture implementation that forgets to rotate before publication.

A state's destructor unregisters while it still owns its sources and context.
The member destructors then release those owners. Because no callback or thread
can observe this intermediate cleanup, counters and ownership change as one
synchronous operation from the caller's perspective. The cache may retain its own
blocks after a source dies; those blocks remain charged under the context.

Leave `namespace read_detail` open here: Part 2 appends `Position`, `Scan`, and
all remaining definitions to this same file. Do not add a closing namespace yet.

## Move the directory and cache into the shared context

Replace `include/strata/storage/database.hpp` in its entirety with the following
blocks. Existing write, rotation, flush, manifest, and diagnostic contracts remain.
The constructor now takes `shared_ptr<Budget>` rather than `Budget&`. There is no
borrowed overload that would pretend a stack ledger has acquired shared ownership.

```cpp title="include/strata/storage/database.hpp — replace entire header"
#ifndef STRATA_STORAGE_DATABASE_HPP
#define STRATA_STORAGE_DATABASE_HPP

#include "strata/io/directory.hpp"
#include "strata/storage/manifest.hpp"
#include "strata/storage/wal.hpp"
#include "strata/storage/read_view.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <memory>
#include <optional>
#include <span>
#include <string>
#include <string_view>

namespace strata::storage {

enum class DatabaseOpen { create, recover };
struct DatabaseOptions {
    std::size_t arena_bytes = 1024 * 1024;
    std::size_t cache_slots = 64;
    std::size_t cache_bytes = 512 * 1024;
    TableOptions table{};
    io::SyncMode sync_mode = io::SyncMode::file;
    ReadLimits reads{};
};
struct InstallHook {
    using Function = void (*)(void*, std::string_view);
    Function function = nullptr;
    void* context = nullptr;
    void operator()(std::string_view stage) const {
        if (function != nullptr) { function(context, stage); }
    }
};
```

`ReadLimits` is appended to the existing options. Defaults admit 16 distinct
states and 64 retained source identities. These are independent of the four-log
and sixteen-table serving caps. A request can fail any of these bounds; a source
slot is not a byte reservation and cannot override the shared memory budget.

The database now owns `context_` instead of owning its directory/cache directly:

```cpp title="include/strata/storage/database.hpp — replace entire header"
class Database {
public:
    Database(std::shared_ptr<memory::Budget> budget, const std::string& directory,
             DatabaseOpen mode, DatabaseOptions options = {}, InstallHook hook = {});
    Database(const Database&) = delete;
    Database& operator=(const Database&) = delete;
    Database(Database&&) = delete;
    Database& operator=(Database&&) = delete;
    [[nodiscard]] WriteResult write(std::span<const Mutation> changes) &;
    std::optional<OwnedEntry> get(Key key) &;
    ReadView snapshot() &;
    ReadRetention retention() const noexcept { return context_->retention(); }
    bool rotate() &;
    bool flush_oldest() &;
    std::uint64_t sequence() const noexcept;
    bool poisoned() const noexcept { return poisoned_; }
    const Manifest& manifest() const & noexcept { return manifest_; }
    const Manifest& manifest() const && = delete;
    std::size_t orphans_at_open() const noexcept { return orphans_at_open_; }
private:
    void require_healthy() const;
    std::uint64_t take_id();
    Manifest next_manifest() const;
    void install(Manifest candidate);
    void create();
    void recover();
    std::shared_ptr<WalMemTable> open_log(LogRecord record, WalOpen mode);
    std::shared_ptr<TableReader> open_table(TableRecord record);
    std::shared_ptr<read_detail::Context> context_;
    memory::Budget& budget_;
    DatabaseOptions options_;
    InstallHook hook_;
    Manifest manifest_{};
    std::array<std::shared_ptr<WalMemTable>, Manifest::max_logs> logs_{};
    std::array<std::shared_ptr<TableReader>, Manifest::max_tables> tables_{};
    std::uint64_t next_id_ = 1;
    std::size_t orphans_at_open_ = 0;
    bool poisoned_ = false;
};

}  // namespace strata::storage
#endif
```

The `budget_` reference is convenient for unchanged lower-level calls. It now
borrows from `context_->budget`, and `context_` is declared first. The database
cannot outlive its own context owner, including during member cleanup.

In `src/storage/database.cpp`, replace the **complete constructor** with:

```cpp title="src/storage/database.cpp — replace constructor"
Database::Database(std::shared_ptr<memory::Budget> budget, const std::string& directory,
                   DatabaseOpen mode, DatabaseOptions options, InstallHook hook)
    : context_(std::allocate_shared<read_detail::Context>(
          memory::SharedAllocator<read_detail::Context>{budget}, budget, directory,
          options.cache_slots, options.cache_bytes, options.reads)),
      budget_(*context_->budget), options_(options), hook_(hook) {
    if (options.arena_bytes < memory::Arena::chunk_charge() ||
        (options.sync_mode != io::SyncMode::file && options.sync_mode != io::SyncMode::full_device)) {
        throw std::invalid_argument("invalid database options");
    }
    const auto inventory = context_->directory.inventory();
    if (inventory.largest_id == std::numeric_limits<std::uint64_t>::max()) {
        throw std::overflow_error("observed file identity exhausts namespace");
    }
    next_id_ = inventory.largest_id + 1;
    switch (mode) {
    case DatabaseOpen::create:
        if (inventory.manifest || inventory.numbered_files != 0) {
            throw FormatError("create requires no existing engine files");
        }
        create();
        break;
    case DatabaseOpen::recover:
        if (!inventory.manifest) { throw FormatError("authoritative MANIFEST missing"); }
        recover();
        orphans_at_open_ = inventory.numbered_files - manifest_.table_count - manifest_.log_count;
        break;
    default: throw std::invalid_argument("invalid database open mode");
    }
}
```

Both the allocator and constructor arguments receive copies of the same budget
handle. Do not pass `std::move(budget)` to one argument while another argument
still needs to copy it: argument evaluation order would make that brittle.
Inventory, create/recover policy, namespace high-water, and orphan accounting
are unchanged. Only their owning context moved.

Replace the complete `open_log` definition with:

```cpp title="src/storage/database.cpp — replace open_log"
std::shared_ptr<WalMemTable> Database::open_log(LogRecord record, WalOpen mode) {
    const auto name = io::file_name(record.id, io::FileKind::log);
    auto opened = context_->directory.open(name.data(), mode == WalOpen::create
        ? io::OpenMode::create_new : io::OpenMode::read_write);
    if (opened.bytes > wal_format::max_log_bytes) { throw FormatError("WAL file cap"); }
    return std::allocate_shared<WalMemTable>(memory::SharedAllocator<WalMemTable>{context_->budget},
        budget_, std::move(opened.file), options_.arena_bytes, mode, options_.sync_mode,
        record.base, mode == WalOpen::recover_sealed ? record.end : 0);
}
```

The WAL object's control storage retains the budget; its internal memtable still
borrows the same ledger. The file is still opened in its manifest-prescribed role.
Sealed replay still requires the exact end from Chapter 11—snapshot lifetime does
not relax persistence validation.

Replace the complete `open_table` definition with:

```cpp title="src/storage/database.cpp — replace open_table"
std::shared_ptr<TableReader> Database::open_table(TableRecord record) {
    const auto name = io::file_name(record.id, io::FileKind::table);
    auto opened = context_->directory.open(name.data(), io::OpenMode::read_only);
    if (opened.bytes != record.bytes) { throw FormatError("table length differs from manifest"); }
    // Durable IDs name files. Cache IDs name bindings inside this cache instance.
    const auto cache_id = context_->cache.new_file_id();
    auto reader = std::allocate_shared<TableReader>(memory::SharedAllocator<TableReader>{context_->budget},
        budget_, std::move(opened.file), opened.bytes, &context_->cache, cache_id);
    reader->verify_all();
    std::uint64_t smallest = std::numeric_limits<std::uint64_t>::max();
    std::uint64_t largest = 0;
    auto cursor = reader->begin();
    while (cursor.valid()) {
        const auto sequence = cursor.entry().key.sequence;
        if (sequence <= record.base || sequence > record.end) {
            throw FormatError("table sequence outside manifest interval");
        }
        smallest = std::min(smallest, sequence);
        largest = std::max(largest, sequence);
        cursor.next();
    }
    if (record.end <= record.base || smallest != record.base + 1 || largest != record.end) {
        throw FormatError("table interval endpoints missing");
    }
    return reader;
}
```

The reader borrows the context's cache, while its containing read state retains
that context. File IDs and cache IDs remain different domains. Full verification,
interval endpoints, and sequence membership are unchanged; this is not permission
to adopt a convenient orphan table.

In all **remaining definitions** in `src/storage/database.cpp`, replace each
`directory_.` member access with `context_->directory.`. This updates `install`,
`create`, `recover`, and flush output creation without changing their ordering.
The three replacement functions above already use the new spelling. This exact
mechanical edit is safe to repeat:

```sh title="Terminal — redirect the remaining directory accesses"
python3 - <<'PYEDIT'
from pathlib import Path
path = Path("src/storage/database.cpp")
source = path.read_text()
source = source.replace("directory_.", "context_->directory.")
path.write_text(source)
PYEDIT
```

Do not change `install`'s synchronization barriers, `rotate`'s poison handling,
or `flush_oldest`'s ordered source retirement. All other definitions stay as in
Chapter 11. The old cache field had references only in the constructor and
`open_table`, both fully replaced above.

## Capture before the effectful rotation, publish after it

Add the following complete method immediately before the closing
`strata::storage` namespace in `src/storage/database.cpp`:

```cpp title="src/storage/database.cpp — add snapshot"
ReadView Database::snapshot() & {
    require_healthy();
    const auto active = manifest_.log_count - 1;
    const bool freeze = logs_[active]->table().size() != 0;
    if (freeze && manifest_.log_count == Manifest::max_logs) {
        throw std::length_error("sealed-log slots exhausted");
    }
    std::array<read_detail::Source, read_detail::max_sources> sources{};
    std::size_t count = 0;
    for (std::size_t i = 0; i < manifest_.table_count; ++i) {
        sources[count++].disk = tables_[i];
    }
    for (std::size_t i = 0; i < manifest_.log_count; ++i) {
        if (i != active || freeze) { sources[count++].memory = logs_[i]; }
    }
    const auto captured = std::span<const read_detail::Source>{sources.data(), count};
    context_->check_admission(captured);  // No allocation, I/O, sealing, or ID consumption.
    auto state = std::allocate_shared<read_detail::State>(
        memory::SharedAllocator<read_detail::State>{context_->budget},
        context_, sequence(), captured);
    if (freeze) { (void)rotate(); }  // Prepare replacement writer before sealing/publication.
    state->activate();              // No allocation and no user callback after rotation.
    return ReadView{std::move(state)};
}
```

First inspect the active table. An empty active table contributes no records and
need not be rotated or captured. Every sealed predecessor is retained. If the
active table is nonempty, the pending state includes that owner but remains
private until rotation has made it immutable.

The fixed array copies owning handles on the stack. It does not copy row payloads.
Preflight counts the union before allocating a state; the state itself is then
allocated and populated **before** rotation. Chapter 11's `rotate` prepares the
replacement WAL/memtable owner before sealing the old one, synchronizes the
required files, installs authority, and publishes the replacement by nonthrowing
owner moves. The data sequence does not change during this maintenance.

The replacement owner is prepared, not a promise of unlimited future write
capacity. Its empty arena allocates chunks lazily; a subsequent batch can still
be definitely rejected for lack of charged memory. This distinction keeps
"prepared replacement" separate from an invented guarantee about future loads.

After `rotate` succeeds, `activate` and the conversion/move into `ReadView` need
no allocation, I/O, or user callback. Thus there is no late state-allocation failure
after a successfully installed freeze boundary. Part 3 arms the next allocation
at `manifest-closed` to check that property directly.

| Failure location | Returned view | Database behavior |
|---|---|---|
| Retention cap or no spare log slot | None | No change; remains usable |
| State allocation before rotation | None | No change; remains usable |
| Rotation after entering its effectful region | None | Poisoned; destroy and reopen |
| Successful rotation and activation | Complete immutable view | New active writer is usable |

An existing view remains readable when a later database operation is poisoned:
it owns an earlier sealed source set. It is not allowed to consult that database's
possibly stale current owner arrays. If a view still holds the directory lock,
release it before reopening. An error does not implicitly revoke another caller's
read handle.

This capture operation may perform durable log rotation and therefore is not an
O(1), allocation-free snapshot promise. Repeated captures without intervening
writes avoid rotation but still allocate distinct states and consume view slots.
Batching read requests or reusing a view can amortize that explicit cost later.

The full on-disk formats are unchanged. No obsolete file is deleted here, no
concurrent reader/writer safety is claimed, and no new power-loss guarantee is
inferred from ownership. Continue with the actual merge in
[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-12-part-2.html).
