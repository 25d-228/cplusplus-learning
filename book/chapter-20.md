# Chapter 20 — Concurrent readers, ingestion, and maintenance

## Part 1 — Share lifetimes, not mutable readers

A query can keep a valid snapshot while the writer commits another batch. That
alone does not make the database concurrent. A disk reader updates counters and
cache recency while it reads. The final snapshot owner might release an arena on
a query worker. A compaction may finish after the writer has installed a new log.
Each of these operations touches a different kind of shared state.

This chapter connects the real writer, query executor, and maintenance path.
An old query will retain sequence 1. While a compaction of the prefix through
sequence 2 is paused, the writer will commit sequence 3 and rotate its log.
Installation must preserve that newer log set. A fresh query must see sequence
3, not an accidental combination of the two snapshots.

> **Learning objectives**
> Publish one coherent source set; distinguish immutable nodes from mutable
> reader bookkeeping; keep final-owner cleanup safe on another thread; build
> maintenance output without the writer mutex; revalidate selected inputs
> rather than rejecting every changed epoch; and stop a backpressured writer
> without losing acknowledged batches.

**Prerequisites:** the complete engine through
[Chapter 19](https://25d-228.github.io/cplusplus-learning/chapter-19.html), including
Chapter 18's synchronization exercises. In particular, keep Chapter 11's manifest
installation, Chapter 12's owning views, Chapter 13's prefix compaction and
retirement, and Chapter 19's query admission and cancellation contracts. The
standalone `Database` remains a synchronous API; the new `Engine` is its
concurrent coordinator, not an instruction to call old methods from arbitrary
threads.

**Implementation: 3.5 hours across all three parts.** Preserve the roadmap split:
publication/locking 1.25 hours; maintenance/backpressure 1.0; controlled overlap
and shutdown tests 1.25. Reading is separate. Record actual implementation time
rather than treating these estimates as minimum durations.

The new files are:

```text
include/strata/storage/maintenance.hpp
src/storage/maintenance.cpp
include/strata/runtime/engine.hpp
src/runtime/engine.cpp
tests/engine.cpp
scripts/check_engine_fixture.py
```

We also amend the budget, read-view, database-friend, WAL-reason, query-provider,
executor, and checkpoint declarations shown below, then append CMake targets.
New files are assembled by concatenating blocks with the same filename in order.
A block marked **complete file** replaces that prerequisite file; concatenate
its blocks first. Unified diffs modify only the named prerequisite: keep context,
remove `-` lines, and insert `+` lines. Do not paste diff prefixes into C++.

[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-20-part-2.html) supplies
installation and the coordinator. [Part 3](https://25d-228.github.io/cplusplus-learning/chapter-20-part-3.html)
supplies the complete checks and build commands. These are three parts of one
chapter, not three new implementation estimates.

## Choose an owner for every mutation

A shared ownership count answers when an object may die. It does not serialize
operations on that object. The standard's shared-pointer rules concern the
pointer/control machinery, not a lock around the pointee; see
[shared ownership](https://eel.is/c++draft/util.smartptr.shared). We therefore
retain immutable source objects but do not share mutable disk readers across
query workers.

| State | Ownership and synchronization in this chapter |
|---|---|
| `Database`, current WALs, manifest, namespace creation and retirement | One `Engine` mutex; one serialized writer |
| Frozen memtable nodes and encoded payloads | Immutable after publication; owning parent view keeps storage alive |
| Query disk readers, cache recency, pins, and read counters | Private to one query worker; no concurrent calls on the same reader/cache |
| Maintenance disk readers, cache, and builder buffers | Private to the maintenance job while it builds |
| Shared storage allocation ledger | Internal budget mutex, including final-owner deallocation |
| Snapshot-retention registry | Internal registry mutex; new snapshot admission also serialized by the engine |
| Maintenance results, failure, and writer-wait counts | Engine mutex; returned as value snapshots |
| Stop signal observed during unlocked work | Atomic Boolean; does not publish the manifest |

The original cache's pinned-bypass and admission algorithm is unchanged. There
is no duplicate-admission race *within a cache*, because one worker owns it.
Two queries may cache the same disk block twice in different budgets. That is a
real efficiency cost, not a race to resolve with an uncharged shared entry.
Parallelism is obtained by isolating mutable readers, not by locking a global
cache around a disk read.

The engine mutex still covers WAL synchronization, manifest installation, and
bounded namespace cleanup. These calls can be slow. It does **not** cover bulk
table scanning, compaction encoding, output verification, query sorting, or CSV
execution. This is not a lock-free writer or a claim of bounded commit latency.

## Repair the ledger before owners cross threads

Suppose a query is the last owner of an old arena. Its worker releases the arena
while the writer reserves space for a new batch. The old Chapter 9 counters
would be read and written concurrently even though neither thread touches the
same payload. Protect the ledger, not the payload.

Let $r$ be reserved bytes, $a$ be allocated bytes, and $L$ be the immutable
budget limit. A reservation for $n$ bytes succeeds only when

$$
0 \le r+a \le L,\qquad n \le L-(r+a).
$$

The subtraction form avoids overflowing an unchecked sum with $n$. Checking
and increasing $r$ must occur under one lock. For example, with $L=1000$,
$r=100$, and $a=700$, two simultaneous reservations of $150$ cannot both pass.
The first produces $(r,a)=(250,700)$; the second sees only 50 free bytes.

Apply the header changes. `mutable` permits a const observer to acquire its
bookkeeping mutex without treating the observation as a logical value change.
`limit_` never changes after construction, so `limit()` needs no ledger lock.

```diff title="include/strata/memory/budget.hpp — exact prerequisite changes"
--- include/strata/memory/budget.hpp
+++ include/strata/memory/budget.hpp
@@ -5,6 +5,7 @@
 #include <exception>
 #include <limits>
 #include <memory>
+#include <mutex>
 #include <new>
 #include <optional>
 #include <stdexcept>
@@ -51,18 +52,19 @@
     Budget& operator=(Budget&&) = delete;
     [[nodiscard]] Lease reserve(std::size_t bytes);
     std::size_t limit() const noexcept { return limit_; }
-    std::size_t reserved() const noexcept { return reserved_; }
-    std::size_t allocated() const noexcept { return allocated_; }
-    std::size_t used() const noexcept { return reserved_ + allocated_; }
-    std::size_t peak() const noexcept { return peak_; }
+    std::size_t reserved() const noexcept { std::lock_guard lock{mutex_}; return reserved_; }
+    std::size_t allocated() const noexcept { std::lock_guard lock{mutex_}; return allocated_; }
+    std::size_t used() const noexcept { std::lock_guard lock{mutex_}; return reserved_ + allocated_; }
+    std::size_t peak() const noexcept { std::lock_guard lock{mutex_}; return peak_; }
     void fail_allocation_on(std::size_t ordinal);
-    void clear_failure() noexcept { fail_after_.reset(); }
+    void clear_failure() noexcept { std::lock_guard lock{mutex_}; fail_after_.reset(); }
 
 private:
     template<class T> friend class Allocator;
     void before_allocate();
     void release_allocation(std::size_t bytes) noexcept;
-    std::size_t limit_;
+    mutable std::mutex mutex_;
+    const std::size_t limit_;
     std::size_t reserved_ = 0;
     std::size_t allocated_ = 0;
     std::size_t peak_ = 0;
```

Replace `src/memory/budget.cpp` with the following two blocks. Each lease still
has exactly one owner. Protecting the budget does not permit two threads to
move or reset the **same lease object** simultaneously.

A committed allocation transfers bytes from $r$ to $a$ without changing $r+a$.
The native allocation remains outside the ledger lock: its reservation already
prevents another thread from spending the same bytes. Allocation failure
unwinds the lease and returns the reservation.

```cpp title="src/memory/budget.cpp — complete file, first block"
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
        std::lock_guard lock{owner_->mutex_};
        if (bytes_ > owner_->reserved_) { std::terminate(); }
        owner_->reserved_ -= bytes_;
        owner_ = nullptr;
        bytes_ = 0;
    }
}

void Budget::Lease::commit_allocation() noexcept {
    if (!owner_) { std::terminate(); }
    std::lock_guard lock{owner_->mutex_};
    if (bytes_ > owner_->reserved_) { std::terminate(); }
    owner_->reserved_ -= bytes_;
    owner_->allocated_ += bytes_;
    owner_ = nullptr;
    bytes_ = 0;
}
```

`reserve` must not call the newly locking `used()` while already holding the
same nonrecursive mutex. It reads the protected fields directly. The observer
methods acquire their own lock only when called from outside that critical
section. Mutex acquisition after an earlier unlock provides the synchronization
we need; this is the rule in the
[mutex requirements](https://eel.is/c++draft/thread.mutex.requirements), not a
property of `std::size_t`.

The failure countdown is also serialized. In a concurrent run, “allocation 7”
means the seventh admitted call reaching that countdown, not a deterministic
choice of thread. Use single-threaded fault injection or controlled gates when
an exact failing operation matters.

```cpp title="src/memory/budget.cpp — complete file, second block"
Budget::~Budget() noexcept {
    if (used() != 0) { std::terminate(); }
}

Budget::Lease Budget::reserve(std::size_t bytes) {
    std::lock_guard lock{mutex_};
    if (bytes > limit_ - (reserved_ + allocated_)) { throw BudgetExceeded{}; }
    if (bytes == 0) { return {}; }
    reserved_ += bytes;
    peak_ = std::max(peak_, reserved_ + allocated_);
    return Lease{*this, bytes};
}

void Budget::release_allocation(std::size_t bytes) noexcept {
    std::lock_guard lock{mutex_};
    if (bytes > allocated_) { std::terminate(); }
    allocated_ -= bytes;
}

void Budget::fail_allocation_on(std::size_t ordinal) {
    std::lock_guard lock{mutex_};
    if (ordinal == 0) { throw std::invalid_argument("allocation ordinal is zero"); }
    fail_after_ = ordinal;
}

void Budget::before_allocate() {
    std::lock_guard lock{mutex_};
    if (fail_after_ && --*fail_after_ == 0) {
        fail_after_.reset();
        throw std::bad_alloc{};
    }
}

}  // namespace strata::memory
```

The cleanup methods retain their nonthrowing contracts. An exceptional failure
to acquire their internal mutex is an unrecoverable invariant/runtime failure,
not an allocation rejection to retry. Ordinary budget exhaustion still throws
before the requested allocation. Budget destruction requires all leases and
allocations gone and no concurrent callers; a mutex does not extend its own
lifetime.

## A final snapshot release is a registry mutation

Snapshot capture checks capacity and later activates a state. In between, it
may prepare a replacement writer. We do not hold the registry mutex over that
I/O. Instead, the engine mutex serializes **all new admissions**, while internal
registry locking allows concurrent releases. A release can only reduce pressure
between check and activation; it cannot consume an unreserved slot.

Add the forward declarations, friendships, registry mutex, and private isolation
entry point. Friendship grants access to these explicitly cooperating classes;
it is not synchronization and does not make the methods public.

```diff title="include/strata/storage/read_view.hpp — exact prerequisite changes"
--- include/strata/storage/read_view.hpp
+++ include/strata/storage/read_view.hpp
@@ -10,14 +10,16 @@
 #include <cstddef>
 #include <cstdint>
 #include <memory>
+#include <mutex>
 #include <optional>
 #include <span>
 #include <string>
 #include <utility>
 
-namespace strata::runtime { class FrozenInputs; }
+namespace strata::runtime { class FrozenInputs; class Engine; }
 
 namespace strata::storage {
+class MaintenanceJob;
 
 struct ReadLimits {
     std::size_t snapshots = 16;
@@ -50,7 +52,7 @@
     void check_admission(std::span<const Source> sources) const;
     void retain(std::span<const Source> sources) noexcept;
     void release(std::span<const Source> sources) noexcept;
-    ReadRetention retention() const noexcept { return counts_; }
+    ReadRetention retention() const noexcept { std::lock_guard lock{mutex_}; return counts_; }
     bool retains_file(std::uint64_t id, io::FileKind kind) const noexcept;
     std::shared_ptr<memory::Budget> budget;
     io::Directory directory;
@@ -63,6 +65,7 @@
         io::FileKind kind = io::FileKind::log;
     };
     std::size_t slot_for(const void* identity) const noexcept;
+    mutable std::mutex mutex_;
     ReadLimits limits_;
     ReadRetention counts_{};
     std::array<Slot, max_retained> slots_{};
@@ -133,9 +136,13 @@
     ReadCursor scan(Key lower, std::optional<Key> upper = std::nullopt) const;
 private:
     friend class Database;
+    friend class MaintenanceJob;
+    friend class strata::runtime::Engine;
     friend class strata::runtime::FrozenInputs;
     ReadView isolate_disk(std::shared_ptr<memory::Budget> budget,
                           std::size_t cache_slots, std::size_t cache_bytes) const;
+    ReadView isolate(std::shared_ptr<memory::Budget> budget, std::size_t cache_slots,
+                     std::size_t cache_bytes, bool allow_memory) const;
     explicit ReadView(std::shared_ptr<const read_detail::State> state) noexcept
         : state_(std::move(state)) {}
     void require_valid() const;
```

```diff title="src/storage/read_view.cpp — exact prerequisite changes"
--- src/storage/read_view.cpp
+++ src/storage/read_view.cpp
@@ -45,6 +45,7 @@
     return slots_.size();
 }
 void Context::check_admission(std::span<const Source> sources) const {
+    std::lock_guard lock{mutex_};
     if (sources.size() > max_sources) { throw std::length_error("too many view sources"); }
     if (counts_.snapshots == limits_.snapshots) {
         throw std::length_error("snapshot states exhausted");
@@ -69,6 +70,7 @@
 }
 
 void Context::retain(std::span<const Source> sources) noexcept {
+    std::lock_guard lock{mutex_};
     for (const auto& source : sources) {
         auto slot = slot_for(source.identity());
         if (slot == slots_.size()) {
@@ -84,6 +86,7 @@
     ++counts_.snapshots;
 }
 void Context::release(std::span<const Source> sources) noexcept {
+    std::lock_guard lock{mutex_};
     for (const auto& source : sources) {
         const auto slot = slot_for(source.identity());
         if (slot == slots_.size() || slots_[slot].references == 0) { std::terminate(); }
@@ -290,6 +293,7 @@
 
 
 bool read_detail::Context::retains_file(std::uint64_t id, io::FileKind kind) const noexcept {
+    std::lock_guard lock{mutex_};
     for (const auto& slot : slots_) {
         if (slot.references != 0 && slot.file_id == id && slot.kind == kind) { return true; }
     }
```

The registry's `slot_for` helper remains nonlocking and private: its callers
already hold that mutex. A collector acquires the engine mutex, checks the
registry briefly, then removes only an unreferenced name. A query releasing its
last parent takes only the registry and budget locks; it never tries to acquire
the engine mutex. This direction is important to avoiding a lock cycle.

A C++ lifetime ending can conflict with accesses to that storage; retaining the
last source owner is therefore part of the concurrency proof, not merely a
memory-leak concern. The relevant definition is
[data races and lifetime](https://eel.is/c++draft/intro.races).

## Share frozen nodes; reopen mutable readers

Chapter 19's immutable fixture required every source on disk. Live capture may
contain a recently sealed memtable. Its nodes, towers, and payloads are now
immutable: rotation installed a different active writer before the view escaped.
We can share that generation without copying all its rows.

Replace `ReadView::isolate_disk` with this delegating wrapper. The frozen fixture
retains its stricter disk-only contract; the live engine will opt into frozen
memory explicitly.

```cpp title="src/storage/read_view.cpp — replace ReadView::isolate_disk"
ReadView ReadView::isolate_disk(std::shared_ptr<memory::Budget> budget,
                               std::size_t cache_slots, std::size_t cache_bytes) const {
    return isolate(std::move(budget), cache_slots, cache_bytes, false);
}
```

Add the next function before the final namespace closing brace in the same
file. It first validates the complete parent, then allocates one private read
context. Disk sources reopen by stable file identity and get private cache IDs.
Memory sources copy only an owning source handle. Their mutable WAL I/O and
writer methods are never called through this read path.

```cpp title="src/storage/read_view.cpp — add isolated reader construction"
ReadView ReadView::isolate(std::shared_ptr<memory::Budget> budget, std::size_t cache_slots,
                           std::size_t cache_bytes, bool allow_memory) const {
    require_valid();
    if (!budget || state_->isolated) { throw std::invalid_argument("invalid isolated read parent"); }
    for (const auto& source : state_->sources()) {
        if (source.memory) {
            if (!allow_memory || !source.memory->table().frozen()) {
                throw std::logic_error("isolated memory source must be explicitly frozen");
            }
        } else if (!source.disk) { throw std::logic_error("missing isolated source"); }
    }
    auto local = std::allocate_shared<read_detail::IsolatedRead>(
        memory::SharedAllocator<read_detail::IsolatedRead>{budget},
        state_, budget, cache_slots, cache_bytes);
    std::array<read_detail::Source, read_detail::max_sources> sources{};
    for (std::size_t i = 0; i < state_->count; ++i) {
        const auto id = state_->held[i].file_id;
        if (state_->held[i].memory) {
            sources[i] = state_->held[i];  // Only immutable nodes, no shared reader bookkeeping.
            continue;
        }
        const auto name = io::file_name(id, io::FileKind::table);
        // Read-only open does not change Directory's inventory or shared counters.
        auto opened = state_->context->directory.open(name.data(), io::OpenMode::read_only);
        auto reader = std::allocate_shared<TableReader>(memory::SharedAllocator<TableReader>{budget},
            *budget, std::move(opened.file), opened.bytes, &local->cache, local->cache.new_file_id());
        sources[i].file_id = id;
        sources[i].disk = std::move(reader);
    }
    auto state = std::allocate_shared<read_detail::State>(memory::SharedAllocator<read_detail::State>{budget},
        std::move(local), state_->sequence, std::span{sources}.first(state_->count));
    return ReadView{std::move(state)};  // Not registered in the parent's mutable retention registry.
}
```

The parent state remains anchored inside `IsolatedRead`. That parent protects
file names during all read-only opens and protects memory until every cursor is
gone. A failed third reader construction destroys the first two private readers
and releases the parent normally. Nothing is registered twice in the storage
retention registry.

The shared `Directory` object has a deliberately narrow concurrent use here:
its read-only `open` branch uses the stable directory descriptor and does not
change its inventory. Creation, replacement, enumeration, and deletion remain
coordinated. This is a property of our Chapter 11 implementation, not permission
to call every `Directory` method concurrently.

A copied `ReadView` still shares its private reader state. Give one isolated
view to one query worker. Do not share that same view between independently
running queries; capture another isolated view for the other query instead.

## Admit for a stable upper bound

An executor must know its descriptor requirement *before* it captures a live
snapshot. Sampling “currently two source files” at submission is unsafe: the
writer can create additional generations before admission. Introduce a small
provider interface whose `source_count()` means a lifetime-stable **upper
bound**. The fixed fixture can return its exact immutable count. The live
engine will return `read_detail::max_sources`, which is 20.

`virtual` and `override` use the same interface mechanism as the file backend
in Chapter 6. The virtual destructor permits ownership through the base type.
The interface abstracts the capture boundary, not indexing, query algorithms,
or storage implementation.

```diff title="include/strata/runtime/frozen_inputs.hpp — exact prerequisite changes"
--- include/strata/runtime/frozen_inputs.hpp
+++ include/strata/runtime/frozen_inputs.hpp
@@ -8,14 +8,21 @@
 
 namespace strata::runtime {
 // Construct before starting workers. No writable Database handle escapes.
-class FrozenInputs {
+class QueryInputs {
+public:
+    virtual ~QueryInputs() = default;
+    virtual storage::ReadView capture(std::shared_ptr<memory::Budget> budget) const = 0;
+    // A lifetime-stable upper bound, not a mutable count sampled at submission.
+    virtual std::size_t source_count() const = 0;
+};
+class FrozenInputs : public QueryInputs {
 public:
     FrozenInputs(const std::string& path, storage::DatabaseOptions options = {},
                  std::size_t opening_bytes = 64 * 1024 * 1024);
     FrozenInputs(const FrozenInputs&) = delete;
     FrozenInputs& operator=(const FrozenInputs&) = delete;
-    storage::ReadView capture(std::shared_ptr<memory::Budget> budget) const;
-    std::size_t source_count() const { return view_.source_count(); }
+    storage::ReadView capture(std::shared_ptr<memory::Budget> budget) const override;
+    std::size_t source_count() const override { return view_.source_count(); }
     std::uint64_t sequence() const { return view_.sequence(); }
 private:
     storage::ReadView view_;
```

```diff title="include/strata/runtime/executor.hpp — exact prerequisite changes"
--- include/strata/runtime/executor.hpp
+++ include/strata/runtime/executor.hpp
@@ -24,7 +24,7 @@
 class Executor {
 public:
     static constexpr std::size_t max_workers = 8, max_queued = 64;
-    Executor(std::shared_ptr<const FrozenInputs> inputs, std::string temporary_parent,
+    Executor(std::shared_ptr<const QueryInputs> inputs, std::string temporary_parent,
              std::string output_parent, ExecutorOptions options = {},
              std::optional<std::size_t> fail_before_launch = std::nullopt,
              std::atomic<std::size_t>* live_threads = nullptr);
@@ -47,7 +47,7 @@
     bool fits(const Request& request) const;
     void join_all() noexcept;
     static void publish(Request& request, TaskResult result);
-    std::shared_ptr<const FrozenInputs> inputs_;
+    std::shared_ptr<const QueryInputs> inputs_;
     std::string temporary_parent_, output_parent_;
     ExecutorOptions options_;
     std::shared_ptr<task_detail::Wakeup> wakeup_ = std::make_shared<task_detail::Wakeup>();
```

The executor keeps the provider owner rather than an unowned reference. A new
`captured` checkpoint occurs only after the immutable view and its sequence
exist; it gives our tests a precise old-snapshot pause without sleeps. All
previous checkpoints and admission behavior remain in force.

```diff title="include/strata/runtime/cancel.hpp — exact prerequisite changes"
--- include/strata/runtime/cancel.hpp
+++ include/strata/runtime/cancel.hpp
@@ -8,7 +8,7 @@
 struct Canceled final : std::exception {
     const char* what() const noexcept override { return "query canceled"; }
 };
-enum class Boundary { admitted, batch, block, spill_pass, before_publish };
+enum class Boundary { admitted, captured, batch, block, spill_pass, before_publish };
 struct CheckpointHook {
     void (*call)(void*, Boundary) = nullptr;
     void* argument = nullptr;
```

```diff title="src/runtime/executor.cpp — exact prerequisite changes"
--- src/runtime/executor.cpp
+++ src/runtime/executor.cpp
@@ -7,7 +7,7 @@
 #include <utility>
 
 namespace strata::runtime {
-Executor::Executor(std::shared_ptr<const FrozenInputs> inputs, std::string temporary_parent,
+Executor::Executor(std::shared_ptr<const QueryInputs> inputs, std::string temporary_parent,
                    std::string output_parent, ExecutorOptions options,
                    std::optional<std::size_t> fail_before_launch,
                    std::atomic<std::size_t>* live_threads)
@@ -149,6 +149,7 @@
         try {
             auto view = inputs_->capture(budget);
             result.sequence = view.sequence();
+            checkpoint(Boundary::captured);
             events = query::make_sort_stats(budget); groups = query::make_sort_stats(budget);
             auto plan = query::make_query(std::move(view), request.options_, temporary_parent_,
                                          events, groups, request.query_hooks);
```

For the fixture, each sort declares three descriptors. Add the provider bound
and Chapter 19's three other query descriptors:

$$
F_q=20+3+3+3=29.
$$

The test admits two queries at once, so its executor allowance is $2F_q=58$.
These are software allowances, not 58 preopened kernel descriptors. Keeping the
bound conservative prevents later capture from exceeding what admission
promised. Existing `FrozenInputs` tests still use their smaller exact bound.

## Reserve a separate route for maintenance

Do not make the maintenance worker acquire a query permit while every query
permit belongs to work waiting for storage progress. Give storage, maintenance
workspaces, and admitted queries separate budgets. Let $S$ be the storage limit,
$M$ the maintenance limit, $P$ the query pool, and $C_s,C_m,C_i$ their current
charged bytes. Then our tracked dynamic allocations obey

$$
C_s\le S,\qquad C_m\le M,\qquad \sum_i C_i\le P,
\qquad C_s+C_m+\sum_i C_i\le S+M+P.
$$

The test uses $S=64$ MiB, $M=2$ MiB, and $P=4$ MiB: a 70 MiB aggregate charged
ceiling, not a claim that 70 MiB is resident. Thread stacks, allocator overhead,
fixed controller objects, library runtime memory, and the operating system's
file cache are not made into an RSS cap by this equation. Resident table-reader
metadata belongs to storage; temporary builders and private maintenance readers
belong to maintenance.

For at most 16 private input readers and $O$ precreated output files, reserve

$$
F_m=16+O+1
$$

maintenance descriptors. The final 1 covers authority installation. An output
writer closes before its replacement reader opens, so they do not add two slots
for the same output. With $O=4$, the test requires 21. Existing storage readers,
WALs, the directory lock, and query allowances are additional. Leave operating-
system headroom; an allowance cannot prevent an external descriptor shortage.

Separate pools prevent queries from consuming maintenance's allowance. They do
not guarantee progress with an undersized storage budget, permanent I/O failure,
or snapshots retained forever. Those conditions must fail visibly or reject
admission, not silently allocate beyond the configured bound.

## Separate preparation, build, and installation

A `MaintenanceJob` has three ownership phases. Construction runs under the
engine mutex: capture input identities, retain immutable sources, reserve output
IDs, and exclusively create a bounded set of output names. `build` runs without
that mutex. `install` reacquires it and validates that the selected inputs are
still the authoritative prefix or oldest sealed log.

Only one split job may exist in an engine. Collection is forbidden until that
job is destroyed, including after failure or stale rejection. This rule protects
its selected input names and precreated output names without pretending they
are already user snapshots or manifest dependencies.

Add the narrowly scoped database friendship:

```diff title="include/strata/storage/database.hpp — exact prerequisite changes"
--- include/strata/storage/database.hpp
+++ include/strata/storage/database.hpp
@@ -59,6 +59,7 @@
     const Manifest& manifest() const && = delete;
     std::size_t orphans_at_open() const noexcept { return orphans_at_open_; }
 private:
+    friend class MaintenanceJob;
     void require_healthy() const;
     std::uint64_t take_id();
     Manifest next_manifest() const;
```

Create the complete job header. `database_` is an identity check, not a pointer
that the unlocked build may dereference. `context_` and `workspace_` are real
owners. Fixed arrays bound output metadata even before any file is written.
`Phase` prevents a failed or consumed job from being rebuilt or installed twice.

```cpp title="include/strata/storage/maintenance.hpp"
#ifndef STRATA_STORAGE_MAINTENANCE_HPP
#define STRATA_STORAGE_MAINTENANCE_HPP
#include "strata/storage/database.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <memory>

namespace strata::storage {
enum class MaintenanceKind { flush, compact };
// Exactly one coordinator owns construction and installation. Only build runs unlocked.
class MaintenanceJob {
public:
    MaintenanceJob(Database& database, std::shared_ptr<memory::Budget> workspace,
                   MaintenanceKind kind, std::size_t selected = 0,
                   CompactionOptions options = {});
    MaintenanceJob(const MaintenanceJob&) = delete;
    MaintenanceJob& operator=(const MaintenanceJob&) = delete;
    void build(InstallHook hook = {});
    bool install(Database& database);  // False: stale inputs; authority was not changed.
    MaintenanceKind kind() const noexcept { return kind_; }
    CompactionStats stats() const noexcept;
    bool rebased() const noexcept { return rebased_; }
private:
    enum class Phase { prepared, building, built, installed, stale, failed };
    void finish_output(std::size_t index, TableInfo info, bool base);
    void build_flush(InstallHook hook);
    void build_compaction(InstallHook hook);
    Database* database_;  // Identity check only; never accessed by build.
    std::shared_ptr<read_detail::Context> context_;
    std::shared_ptr<memory::Budget> workspace_;
    MaintenanceKind kind_;
    Phase phase_ = Phase::prepared;
    Manifest captured_;
    LogRecord sealed_{};
    std::shared_ptr<const WalMemTable> memory_;
    ReadView input_;
    CompactionOptions options_;
    io::SyncMode sync_mode_;
    std::uint64_t end_ = 0;
    std::size_t selected_ = 0, output_limit_ = 0, output_count_ = 0;
    std::array<std::uint64_t, Manifest::max_tables> ids_{};
    std::array<io::File, Manifest::max_tables> files_{};
    std::array<TableRecord, Manifest::max_tables> records_{};
    std::array<std::shared_ptr<TableReader>, Manifest::max_tables> readers_{};
    CompactionStats stats_{};
    bool rebased_ = false;
};
}  // namespace strata::storage
#endif
```

Start the source file with preparation. Flush retains the **oldest sealed**
memtable. Compaction selects a complete table prefix, including every existing
base partition. The selection rule is the same tombstone-elision proof as
Chapter 13; we are changing when the work runs, not weakening its coverage.

```cpp title="src/storage/maintenance.cpp"
#include "strata/storage/maintenance.hpp"
#include "strata/runtime/cancel.hpp"

#include <algorithm>
#include <limits>
#include <optional>
#include <span>
#include <stdexcept>
#include <utility>

namespace strata::storage {
MaintenanceJob::MaintenanceJob(Database& database, std::shared_ptr<memory::Budget> workspace,
    MaintenanceKind kind, std::size_t selected, CompactionOptions options)
    : database_(&database), context_(database.context_), workspace_(std::move(workspace)),
      kind_(kind), captured_(database.manifest_), options_(options),
      sync_mode_(database.options_.sync_mode), selected_(selected) {
    database.require_healthy();
    if (!workspace_) { throw std::invalid_argument("missing maintenance workspace"); }
    (void)database.next_manifest();  // Epoch exhaustion before creating names.
    if (kind == MaintenanceKind::flush) {
        if (captured_.log_count < 2 || captured_.table_count == Manifest::max_tables) {
            throw std::length_error("no flushable log or no table slot");
        }
        sealed_ = captured_.logs[0]; memory_ = database.logs_[0]; end_ = sealed_.end;
        if (!memory_->table().frozen()) { throw std::logic_error("flush input is mutable"); }
        output_limit_ = 1; options_.output = database.options_.table;
    } else if (kind == MaintenanceKind::compact) {
        if (selected == 0 || selected < captured_.base_count || selected > captured_.table_count ||
            options.max_output_files == 0 || options.max_output_files > Manifest::max_tables ||
            options.max_output_bytes == 0) {
            throw std::invalid_argument("invalid maintenance prefix");
        }
        end_ = captured_.tables[selected - 1].end;
        output_limit_ = std::min(options.max_output_files,
                                Manifest::max_tables - (captured_.table_count - selected));
        std::array<read_detail::Source, read_detail::max_sources> sources{};
        for (std::size_t i = 0; i < selected; ++i) {
            sources[i].file_id = captured_.tables[i].id; sources[i].disk = database.tables_[i];
            io::IoBytes::add(stats_.input_entries, database.tables_[i]->entries(), stats_.counters_saturated);
            io::IoBytes::add(stats_.selected_file_bytes, captured_.tables[i].bytes, stats_.counters_saturated);
        }
        stats_.selected_files = selected;
        input_ = ReadView{std::allocate_shared<read_detail::State>(
            memory::SharedAllocator<read_detail::State>{workspace_}, context_, end_,
            std::span<const read_detail::Source>{sources.data(), selected})};
        // This private state is not a user snapshot. The coordinator forbids collection
        // until the job has been destroyed, including on a stale result.
    } else { throw std::invalid_argument("unknown maintenance kind"); }
    // Reserve identities and namespace entries while the coordinator owns Database.
    // Unused output names stay orphaned until the next explicit collection.
    for (std::size_t i = 0; i < output_limit_; ++i) {
        ids_[i] = database.take_id();
        const auto name = io::file_name(ids_[i], io::FileKind::table);
        files_[i] = context_->directory.open(name.data(), io::OpenMode::create_new).file;
    }
}
```

An output creation failure can leave earlier newly created names behind. Their
IDs have already advanced and must not be reused. Ordinary member cleanup closes
the descriptors; later collection or reopening handles the nonauthoritative
names. The existing manifest never references an incomplete output merely
because its pathname exists.

The private compaction `State` is intentionally not activated in user retention.
The coordinator's no-collection rule is therefore essential. A second arbitrary
collector thread would violate this protocol even though all shared pointers
were individually valid.

## Validate each output before it can become authoritative

The output transfer counter is computed from the job's file owners. A closed
`File` preserves its byte counters, so a partial output remains observable after
failure. Completed-file bytes and installed authority remain separate facts.

```cpp title="src/storage/maintenance.cpp"
CompactionStats MaintenanceJob::stats() const noexcept {
    auto result = stats_;
    // File preserves its transfer counters after close. Count partial output too.
    for (const auto& file : files_) {
        const auto traffic = file.io_bytes();
        io::IoBytes::add(result.output_write_bytes, traffic.written, result.counters_saturated);
        result.counters_saturated |= traffic.saturated;
    }
    return result;
}
```

An output is synchronized and closed, then reopened through a new reader with
**no shared cache**. Its resident metadata is charged to the storage budget;
its builder has already been destroyed. Full verification and a semantic scan
check the interval, row kind, key ordering, and base-partition boundaries before
that reader is placed in the prepared owner array.

```cpp title="src/storage/maintenance.cpp"
void MaintenanceJob::finish_output(std::size_t index, TableInfo info, bool base) {
    files_[index].sync(sync_mode_);
    files_[index].close_checked();
    const TableRecord record{ids_[index], base ? 0 : sealed_.base, end_, info.file_bytes};
    const auto name = io::file_name(record.id, io::FileKind::table);
    auto opened = context_->directory.open(name.data(), io::OpenMode::read_only);
    if (opened.bytes != record.bytes) { throw FormatError("maintenance output length"); }
    // Resident metadata uses the storage ledger. No shared cache is touched here.
    auto reader = std::allocate_shared<TableReader>(
        memory::SharedAllocator<TableReader>{context_->budget}, *context_->budget,
        std::move(opened.file), opened.bytes);
    reader->verify_all();
    auto cursor = reader->begin();
    std::optional<Key> previous;
    std::uint64_t smallest = std::numeric_limits<std::uint64_t>::max(), largest = 0;
    while (cursor.valid()) {
        const auto entry = cursor.entry();
        if (entry.key.sequence <= record.base || entry.key.sequence > record.end ||
            (base && (entry.kind != EntryKind::put ||
                      (previous && !KeyLess{}(*previous, entry.key.user))))) {
            throw FormatError("maintenance output semantics");
        }
        previous = entry.key.user;
        smallest = std::min(smallest, entry.key.sequence); largest = std::max(largest, entry.key.sequence);
        cursor.next();
    }
    if (!previous || (!base && (smallest != record.base + 1 || largest != record.end))) {
        throw FormatError("maintenance output interval endpoints");
    }
    if (base && index != 0 && !KeyLess{}(*readers_[index - 1]->last_key(), *reader->first_key())) {
        throw FormatError("maintenance base ranges overlap");
    }
    records_[index] = record; readers_[index] = std::move(reader); ++output_count_;
    ++stats_.output_files; stats_.completed_file_bytes += info.file_bytes;
}
```

For a flushed interval $(b,e]$, its minimum sequence must be $b+1$ and its maximum
must be $e$. For a compacted base through $e$, surviving rows may have older
sequences, but each must be a put with $0<s\le e$ and strictly increasing user
keys. An entirely deleted prefix produces no output reader; the manifest still
records its coverage boundary in Part 2.

`output_files` counts verified prepared files here. `output_write_bytes` includes
partial writes as well. `merge_read_bytes` below counts private input-reader
traffic, including their metadata reads; it does **not** count output verification
reads, directory metadata, WAL traffic, or physical device amplification. Do not
label it total engine I/O.

## Reuse the ordered scans and bounded builder

Flush has no sorting step. The frozen skip list already supplies the required
internal-key order. Its cancellation boundary occurs between rows; the file
layer also checks at block I/O boundaries. A hook may throw after a row to test
failure cleanup, but it must not reenter the engine.

```cpp title="src/storage/maintenance.cpp"
void MaintenanceJob::build_flush(InstallHook hook) {
    TableInfo info{};
    {
        TableBuilder builder{*workspace_, files_[0], options_.output};
        for (auto cursor = memory_->table().begin(); cursor.valid(); cursor.next()) {
            runtime::checkpoint(runtime::Boundary::batch);
            builder.append(cursor.entry()); ++stats_.output_entries; hook("maintenance-row");
        }
        info = builder.finish();
    }
    finish_output(0, info, false);
}
```

Compaction reopens private disk readers and uses the existing bounded heap merge.
One output builder exists at a time. `can_append` includes directory, Bloom, and
footer costs from Chapter 13; it is not a test of row bytes alone.

The local `ReadTraffic` object is a small scope guard: its destructor collects
counters on both success and exception. Its source span borrows from `local`,
which was constructed first and therefore outlives the guard. The cursor is
constructed last and dies first. No heap allocation is needed to record these
fixed-size observations.

```cpp title="src/storage/maintenance.cpp"
void MaintenanceJob::build_compaction(InstallHook hook) {
    auto local = input_.isolate_disk(workspace_, 1, 256);
    struct ReadTraffic {
        std::span<const read_detail::Source> sources;
        CompactionStats& report;
        ~ReadTraffic() noexcept {
            for (const auto& source : sources) {
                const auto traffic = source.disk->io_bytes();
                io::IoBytes::add(report.merge_read_bytes, traffic.read, report.counters_saturated);
                report.counters_saturated |= traffic.saturated;
            }
        }
    } meter{local.state_->sources(), stats_};
    auto cursor = local.scan({Relation::events, 0});
    while (cursor.valid()) {
        if (output_count_ == output_limit_) { throw std::length_error("maintenance output slots"); }
        auto output_options = options_.output;
        output_options.file_limit = std::min(output_options.file_limit,
            options_.max_output_bytes - stats_.completed_file_bytes);
        TableInfo info{};
        {
            TableBuilder builder{*workspace_, files_[output_count_], output_options};
            std::uint64_t rows = 0;
            while (cursor.valid() && builder.can_append(cursor.entry())) {
                runtime::checkpoint(runtime::Boundary::batch);
                builder.append(cursor.entry()); ++rows; ++stats_.output_entries;
                hook("maintenance-row"); cursor.next();
            }
            if (rows == 0) { throw std::length_error("one compacted row cannot fit"); }
            info = builder.finish();
        }
        finish_output(output_count_, info, true);
    }
}
```

If $D$ is the total output-byte allowance and $d$ the completed bytes so far,
the next file has limit $\min(L_f,D-d)$, where $L_f$ is its individual file cap.
Each completed file fits the remaining allowance, so the subtraction is valid.
If even one row cannot fit, fail; do not create an infinite series of empty runs.

Complete the build wrapper. It closes precreated but unused writers, leaving
their names explicitly nonauthoritative. Its state becomes `built` only after
the final test hook returns. Append the installation function in Part 2 before
closing this namespace; this source file is intentionally continued there.

```cpp title="src/storage/maintenance.cpp"
void MaintenanceJob::build(InstallHook hook) {
    if (phase_ != Phase::prepared) { throw std::logic_error("maintenance job cannot be rebuilt"); }
    phase_ = Phase::building;
    try {
        hook(kind_ == MaintenanceKind::flush ? "flush-build" : "compact-build");
        if (kind_ == MaintenanceKind::flush) { build_flush(hook); } else { build_compaction(hook); }
        // Close precreated-but-unused descriptors. Their names remain nonauthoritative.
        for (std::size_t i = output_count_; i < output_limit_; ++i) { files_[i].close_checked(); }
        hook("maintenance-built"); phase_ = Phase::built;
    } catch (...) { phase_ = Phase::failed; throw; }
}
```

At this boundary the output is valid, but the database still serves its original
source set. The writer may already have advanced. We have prepared a candidate,
not acquired permission to overwrite newer history.

Continue with [Part 2 — Build outside the lock; install against current authority](https://25d-228.github.io/cplusplus-learning/chapter-20-part-2.html).

