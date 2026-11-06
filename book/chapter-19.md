# Chapter 19 — Bounded execution and cooperative cancellation

## Part 1 — Admit ownership before doing work

Two individually bounded queries can exceed their shared allowance when they
start together. A canceled query can retain its spill files while reporting
that its worker is idle. A third query can wait forever if cancellation changes
an atomic flag without waking the condition variable on which admission sleeps.
A worker pool must resolve these problems, not merely call a function on another
thread.

We will run the complete two-sort query from Chapter 17 on a prepared immutable
database. Two workers will overlap, with private read-side caches and budgets,
while a scheduler reserves each query's entire working allowance and descriptor
allowance. Queued descriptors will own neither a snapshot nor a materialized
input. Every accepted descriptor will acquire one terminal outcome, even when
shutdown, allocation failure, or cancellation interrupts the calculation.

> **Learning objectives**
> Separate accepted work from admitted work; distinguish immutable data from
> mutable reader bookkeeping; reserve resources before constructing operators;
> wake a canceled resource waiter without a lost notification; transport a worker
> exception without transporting its query owners; and define cancellation on
> both sides of the result-publication boundary.

**Prerequisites:** the complete engine through
[Chapter 17](https://25d-228.github.io/cplusplus-learning/chapter-17.html), including
its fixed query plan and completion-aware CSV sink, and the synchronization
exercises in [Chapter 18](https://25d-228.github.io/cplusplus-learning/chapter-18.html).
Keep the Chapter 9 allocation ledger, Chapter 12 read-view ownership, and Chapter
15 spill cleanup contracts available. No earlier engine class has silently
become safe for concurrent mutation.

**Implementation: 4.0 hours across all three parts.** Keep the roadmap split:
1.25 hours for task states and the pool, 1.25 for admission and cancellation, and
1.5 for completion, shutdown, and fault checks. Reading is separate. Implement
and test in small increments; the estimate is not a required minimum.

Create `include/strata/runtime/` and `src/runtime/`. The three parts provide:

```text
include/strata/runtime/cancel.hpp
include/strata/runtime/frozen_inputs.hpp
include/strata/runtime/task.hpp
include/strata/runtime/executor.hpp
src/runtime/frozen_inputs.cpp
src/runtime/task.cpp
src/runtime/executor.cpp
tests/executor.cpp
scripts/check_executor_fixture.py
```

We also amend `storage/read_view.hpp`, `storage/read_view.cpp`, `io/file.cpp`,
`query/scan.cpp`, `query/external_sort.cpp`, `query/result.hpp`, and append CMake
targets. Every amendment appears below. For new files, concatenate blocks with
the same filename in reading order, adding a newline between blocks. Unified
diffs describe edits to the named prerequisite file: retain context lines,
remove `-` lines, and insert `+` lines; the prefixes are not C++ syntax.

[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-19-part-2.html) implements
the task and scheduler. [Part 3](https://25d-228.github.io/cplusplus-learning/chapter-19-part-3.html)
supplies the complete deterministic and native-file checks. This chapter does
not yet permit concurrent ingestion or maintenance.

## One request, three different bounds

An accepted request occupies a queue slot. An admitted request additionally
owns permission to use working memory and file descriptors. An actually running
operator has allocated some portion of that permission. Those are three
separate events.

Let $Q$ be queue capacity, $W$ worker count, and $A_i$ query $i$'s declared working
allowance in bytes. Let $C_i$ be that query's current charged allocation, and $P$
the scheduler's pool allowance. Admission maintains

$$
0 \le C_i \le A_i,\qquad \sum_{i\text{ admitted}} A_i \le P.
$$

The first inequality is enforced by a private Chapter 9 `Budget`. The second is
enforced under the scheduler mutex, *before* that budget or a read view is
created. Reserving $A_i$ is conservative: unused permission is not lent to a
second query that might later need it simultaneously. We deliberately avoid
incremental allocation waits inside operators. Otherwise two half-built sorts
could each hold resources needed for the other's next allocation.

With two 2 MiB requests and a 4 MiB pool, both can run. With a 2 MiB pool, one
runs and the other remains a small queued descriptor. A 6 MiB request can never
fit that pool; submission returns `impossible` without taking its owner. A
request whose allowance fits the pool can still fail if that allowance is too
small for the requested plan. That is a local query failure, not permission to
exceed the cap.

The descriptor rule is similarly explicit. Let $T$ be the frozen view's table
source count, and $F_e,F_g$ the two sorts' configured descriptor allowances.
Reserve

$$
F_i=T+F_e+F_g+3.
$$

The three extra descriptors cover two spill-directory handles and one temporary
CSV file. Both sorts can coexist, so use their sum rather than their maximum.
For the one-table fixture with three descriptors per sort, $F_i=1+3+3+3=10$.
A pool allowance of 20 admits two such queries; an allowance of 10 admits only
one even when memory is plentiful. Open failures from the operating system
still become ordinary query failures: these are internal allowances, not a
reservation granted by the kernel.

The bounded queue and fixed workers limit executor-owned unfinished descriptors
to $Q+W$. This does **not** bound arbitrarily many caller-created rejected
requests or retained completion handles. Thread stacks, standard-library control
blocks, exception storage, allocator overhead, result bytes, and the fixed
opening context also have separate costs. The fixture's opening budget is
64 MiB by default; it is not charged repeatedly to the query pool. The allocation
ledger is not resident process memory. Spill disk caps remain per sort, not a
new global disk quota.

## Immutable data is not an immutable reader

The Chapter 12 `ReadView` retains its files and protects its historical answer.
It does not make cache recency links, pin counts, allocation totals, or the
retention registry atomic. Even a method named `get` can update those structures.
Sharing its existing `TableReader` objects across workers would therefore be
incorrect.

The reference-count operations of separate `shared_ptr` objects do not make the
pointed-to object safe for arbitrary concurrent use. The language library's
[shared ownership rules](https://eel.is/c++draft/util.smartptr.shared) describe
ownership machinery, not a mutex for our cache. We must choose an access policy
for each pointed-to object.

Here the policy is **thread confinement with a frozen parent**:

```text
controller: FrozenInputs -> master ReadView -> immutable file identities
                                      |       directory lock and baseline owners
                    shared lifetime --+
                                      |
worker A: isolated State -> own TableReaders -> own cache -> own Budget
worker B: isolated State -> own TableReaders -> own cache -> own Budget
```

The parent is captured before workers start and is never served directly to a
query. Each worker reopens the same immutable files, allocating its own metadata
and buffers. It does not copy all rows. The original state anchors filenames and
the directory lock; the private state does not mutate the parent's retention
registry. A query's borrowed batches never cross into another worker.

This costs duplicate metadata and cache contents. It also avoids putting a giant
mutex around query execution: the two sorts, joins, and aggregations can really
overlap. The next chapter can build a live capture/maintenance protocol on an
explicit boundary instead of pretending that an ordinary `Database` is already
thread-safe.

## Retain the parent without allocating through it

In `include/strata/storage/read_view.hpp`, add a forward declaration for the
runtime provider, an isolated read owner, and the alternative state constructor.
The `friend` declaration exposes the new capture operation only to that provider,
not to arbitrary code holding a normal `ReadView`.

```diff title="include/strata/storage/read_view.hpp"
--- a/include/strata/storage/read_view.hpp
+++ b/include/strata/storage/read_view.hpp
@@ -14,6 +14,8 @@
 #include <span>
 #include <string>
 #include <utility>
+
+namespace strata::runtime { class FrozenInputs; }
 
 namespace strata::storage {
 
@@ -66,15 +68,31 @@
     std::array<Slot, max_retained> slots_{};
 };
 
+struct State;
+struct IsolatedRead {
+    IsolatedRead(std::shared_ptr<const State> parent, std::shared_ptr<memory::Budget> owner,
+                 std::size_t slots, std::size_t bytes)
+        : anchor(std::move(parent)), budget(std::move(owner)), cache(*budget, slots, bytes) {}
+    std::shared_ptr<const State> anchor;
+    std::shared_ptr<memory::Budget> budget;
+    BlockCache cache;
+};
+
 struct State {
     State(std::shared_ptr<Context> owner, std::uint64_t visible,
           std::span<const Source> sources);
+    State(std::shared_ptr<IsolatedRead> owner, std::uint64_t visible,
+          std::span<const Source> sources);
+    std::shared_ptr<memory::Budget> budget_owner() const noexcept {
+        return isolated ? isolated->budget : context->budget;
+    }
     ~State() noexcept;
     State(const State&) = delete;
     State& operator=(const State&) = delete;
     void activate() noexcept;
     std::span<const Source> sources() const noexcept { return {held.data(), count}; }
     std::shared_ptr<Context> context;
+    std::shared_ptr<IsolatedRead> isolated;
     std::uint64_t sequence;
     std::array<Source, max_sources> held{};
     std::size_t count;
@@ -115,6 +133,9 @@
     ReadCursor scan(Key lower, std::optional<Key> upper = std::nullopt) const;
 private:
     friend class Database;
+    friend class strata::runtime::FrozenInputs;
+    ReadView isolate_disk(std::shared_ptr<memory::Budget> budget,
+                          std::size_t cache_slots, std::size_t cache_bytes) const;
     explicit ReadView(std::shared_ptr<const read_detail::State> state) noexcept
         : state_(std::move(state)) {}
     void require_valid() const;
```

`IsolatedRead` has three members in lifetime order. Its `anchor` retains the
original state; `budget` owns the private ledger; `cache` borrows that ledger.
Reverse destruction releases the cache before the budget. The `State::held`
source array is declared after `isolated`, so its readers disappear before the
cache they use. A cursor's owning `State` keeps the entire chain alive.

There are now two legal state shapes. An ordinary state has `context`, and its
`active` flag records registration in that context. An isolated state has
`isolated`, no `context`, and stays inactive. Its destructor must not call
`context->release`. The existing conditional destructor already expresses that
rule. `budget_owner()` is the one selection point used by cursor allocation.
An isolated state's `retention()` reports its own one state and source count,
not a concurrent reading of the parent's registry.

The following complete amendment to `src/storage/read_view.cpp` adds the second
constructor, switches those selection points, and defines disk isolation.

```diff title="src/storage/read_view.cpp"
--- a/src/storage/read_view.cpp
+++ b/src/storage/read_view.cpp
@@ -99,6 +99,12 @@
              std::span<const Source> sources)
     : context(std::move(owner)), sequence(visible), count(sources.size()) {
     if (count > held.size()) { throw std::length_error("view source bound"); }
+    std::copy(sources.begin(), sources.end(), held.begin());
+}
+State::State(std::shared_ptr<IsolatedRead> owner, std::uint64_t visible,
+             std::span<const Source> sources)
+    : isolated(std::move(owner)), sequence(visible), count(sources.size()) {
+    if (count > held.size()) { throw std::length_error("isolated source bound"); }
     std::copy(sources.begin(), sources.end(), held.begin());
 }
 State::~State() noexcept {
@@ -252,7 +258,10 @@
 }
 std::uint64_t ReadView::sequence() const { require_valid(); return state_->sequence; }
 std::size_t ReadView::source_count() const { require_valid(); return state_->count; }
-ReadRetention ReadView::retention() const { require_valid(); return state_->context->retention(); }
+ReadRetention ReadView::retention() const {
+    require_valid();
+    return state_->isolated ? ReadRetention{1, state_->count} : state_->context->retention();
+}
 std::optional<OwnedEntry> ReadView::get(Key key) const {
     require_valid();
     valid_key(key);
@@ -275,7 +284,7 @@
 ReadCursor ReadView::scan(Key lower, std::optional<Key> upper) const {
     require_valid();
     return ReadCursor{std::allocate_shared<read_detail::Scan>(
-        memory::SharedAllocator<read_detail::Scan>{state_->context->budget}, state_, lower, upper)};
+        memory::SharedAllocator<read_detail::Scan>{state_->budget_owner()}, state_, lower, upper)};
 }
 bool ReadCursor::valid() const noexcept { return scan_ && scan_->current.has_value(); }
 EntryView ReadCursor::entry() const & {
@@ -288,7 +297,34 @@
 
 std::shared_ptr<memory::Budget> ReadView::budget_owner() const {
     require_valid();
-    return state_->context->budget;
+    return state_->budget_owner();
+}
+
+
+ReadView ReadView::isolate_disk(std::shared_ptr<memory::Budget> budget,
+                               std::size_t cache_slots, std::size_t cache_bytes) const {
+    require_valid();
+    if (!budget || state_->isolated) { throw std::invalid_argument("invalid isolated read parent"); }
+    for (const auto& source : state_->sources()) {
+        if (source.memory || !source.disk) { throw std::logic_error("fixture must be fully flushed"); }
+    }
+    auto local = std::allocate_shared<read_detail::IsolatedRead>(
+        memory::SharedAllocator<read_detail::IsolatedRead>{budget},
+        state_, budget, cache_slots, cache_bytes);
+    std::array<read_detail::Source, read_detail::max_sources> sources{};
+    for (std::size_t i = 0; i < state_->count; ++i) {
+        const auto id = state_->held[i].file_id;
+        const auto name = io::file_name(id, io::FileKind::table);
+        // Read-only open does not change Directory's inventory or shared counters.
+        auto opened = state_->context->directory.open(name.data(), io::OpenMode::read_only);
+        auto reader = std::allocate_shared<TableReader>(memory::SharedAllocator<TableReader>{budget},
+            *budget, std::move(opened.file), opened.bytes, &local->cache, local->cache.new_file_id());
+        sources[i].file_id = id;
+        sources[i].disk = std::move(reader);
+    }
+    auto state = std::allocate_shared<read_detail::State>(memory::SharedAllocator<read_detail::State>{budget},
+        std::move(local), state_->sequence, std::span{sources}.first(state_->count));
+    return ReadView{std::move(state)};  // Not registered in the parent's mutable retention registry.
 }
 
 }  // namespace strata::storage
```

In `isolate_disk`, the first loop rejects any memtable source before allocating
private state. The second loop opens each retained table using the parent's
directory descriptor. We use only `Directory::open(..., read_only)`: this branch
reads the descriptor, performs the relative open and file-size inspection, and
does not update the directory inventory. No other directory operation runs
concurrently in this chapter. This is an argument about the actual branch's
accesses, not a claim that all non-const directory methods are thread-safe.

A private cache gives each new reader a private cache identity. Reusing the
parent reader merely because the bytes are immutable would reintroduce shared
mutable counters and cache state. If the third reader construction fails, the
local `sources` array destroys the first two readers before `local` destroys its
cache. The returned state's control allocation uses `SharedAllocator`, retaining
the private budget through the last control-storage deallocation as in Chapter
12. Nothing allocates through the parent's unsynchronized ledger.

The master remains owned by `FrozenInputs` throughout all worker activity. Its
retention-registry release and baseline deallocations occur only after joining
workers and releasing that provider. A normal mutable database handle must not
escape beside it. The cooperation contract still excludes external processes
that ignore the directory lock and overwrite table files.

## Open once, then expose only isolated captures

Create this provider header:

```cpp title="include/strata/runtime/frozen_inputs.hpp"
#ifndef STRATA_RUNTIME_FROZEN_INPUTS_HPP
#define STRATA_RUNTIME_FROZEN_INPUTS_HPP
#include "strata/storage/database.hpp"

#include <cstddef>
#include <memory>
#include <string>

namespace strata::runtime {
// Construct before starting workers. No writable Database handle escapes.
class FrozenInputs {
public:
    FrozenInputs(const std::string& path, storage::DatabaseOptions options = {},
                 std::size_t opening_bytes = 64 * 1024 * 1024);
    FrozenInputs(const FrozenInputs&) = delete;
    FrozenInputs& operator=(const FrozenInputs&) = delete;
    storage::ReadView capture(std::shared_ptr<memory::Budget> budget) const;
    std::size_t source_count() const { return view_.source_count(); }
    std::uint64_t sequence() const { return view_.sequence(); }
private:
    storage::ReadView view_;
    std::size_t cache_slots_, cache_bytes_;
};
}  // namespace strata::runtime
#endif
```

The constructor opens the prepared database while the controller is still alone.
It requires exactly one active WAL whose base equals the recovered visible
sequence. Thus no unflushed mutation needs to be copied into a query-local
memtable. Earlier chapters' rotation and flush operations prepare that state.
An empty database is also legal.

The setup open uses ordinary database recovery. It can synchronize or repair an
allowed active-log tail; it is not a promise that startup performs only read-only
system calls. After construction, the local `Database` is gone and the provider
exposes only sequence, source count, and isolated reads. Do not call maintenance
in this constructor to hide an unprepared fixture.

```cpp title="src/runtime/frozen_inputs.cpp"
#include "strata/runtime/frozen_inputs.hpp"

#include <stdexcept>
#include <utility>

namespace strata::runtime {
FrozenInputs::FrozenInputs(const std::string& path, storage::DatabaseOptions options,
                           std::size_t opening_bytes)
    : cache_slots_(options.cache_slots), cache_bytes_(options.cache_bytes) {
    auto budget = std::make_shared<memory::Budget>(opening_bytes);
    storage::Database database{budget, path, storage::DatabaseOpen::recover, options};
    // Do not maintain or rotate here: the controller must prepare a disk-only fixture.
    const auto& manifest = database.manifest();
    if (manifest.log_count != 1 || database.sequence() != manifest.logs[0].base) {
        throw std::invalid_argument("prepare and flush the immutable fixture before opening");
    }
    view_ = database.snapshot();
}
storage::ReadView FrozenInputs::capture(std::shared_ptr<memory::Budget> budget) const {
    return view_.isolate_disk(std::move(budget), cache_slots_, cache_bytes_);
}
}  // namespace strata::runtime
```

The private `capture` result can outlive the local call because it owns both its
private ledger and its parent anchor. Within this executor it is created, used,
and destroyed on one worker. Callers receive scalar completion statistics and a
published CSV name, not an escaped plan or allocator-backed batch. That narrower
interface is what makes the confinement argument tractable.

## Cancellation is a request, not an asynchronous destructor

We need a token reachable from deeply nested block I/O without adding a token
parameter to every existing codec and operator. Use a scoped thread-local pointer
to a borrowed token. `thread_local` denotes a distinct object for each thread;
our inline header variable names the current thread's slot across translation
units. An `inline` variable permits this header definition in multiple translation
units without creating a different program variable in each; it is not a request
to optimize a function call. See [thread storage duration](https://eel.is/c++draft/basic.stc.thread)
and the [inline specifier](https://eel.is/c++draft/dcl.inline).
The pointer's lifetime is not the token's lifetime: an RAII scope restores the
previous pointer before the stack token dies.

This design is for ordinary synchronous calls on a worker. A coroutine that
resumes on another thread, a child thread, or a nested blocking pool submission
would not inherit this contract. We implement none of those. A null scope means
"temporarily do not observe cancellation," which will matter at result commit.

Create the complete token header:

```cpp title="include/strata/runtime/cancel.hpp"
#ifndef STRATA_RUNTIME_CANCEL_HPP
#define STRATA_RUNTIME_CANCEL_HPP
#include <atomic>
#include <exception>
#include <utility>

namespace strata::runtime {
struct Canceled final : std::exception {
    const char* what() const noexcept override { return "query canceled"; }
};
enum class Boundary { admitted, batch, block, spill_pass, before_publish };
struct CheckpointHook {
    void (*call)(void*, Boundary) = nullptr;
    void* argument = nullptr;
};
struct Cancellation {
    const std::atomic<bool>& requested;
    CheckpointHook hook{};
};
inline thread_local const Cancellation* current_cancellation = nullptr;
class CancellationScope {
public:
    explicit CancellationScope(const Cancellation* value) noexcept
        : previous_(std::exchange(current_cancellation, value)) {}
    ~CancellationScope() noexcept { current_cancellation = previous_; }
    CancellationScope(const CancellationScope&) = delete;
    CancellationScope& operator=(const CancellationScope&) = delete;
private:
    const Cancellation* previous_;
};
inline void checkpoint(Boundary boundary) {
    const auto* token = current_cancellation;
    if (!token) { return; }
    if (token->requested.load(std::memory_order_relaxed)) { throw Canceled{}; }
    if (token->hook.call) { token->hook.call(token->hook.argument, boundary); }
    if (token->requested.load(std::memory_order_relaxed)) { throw Canceled{}; }
}
}  // namespace strata::runtime
#endif
```

`Cancellation::requested` borrows an atomic flag in the task's completion owner.
The worker retains that owner for the whole call. `CheckpointHook` is a borrowed
function pointer/context pair used for controlled tests; production use leaves
it empty. The callback must not destroy its request, retain borrowed query rows,
or recursively wait for work on this pool.

The first load catches an already requested stop. The hook can request a stop
at an exact boundary; the second load observes it before the operation continues.
`Canceled` is an ordinary exception, so the existing owned plans, runs, buffers,
and file handles unwind normally. No other thread destroys those objects.

The flag carries only the stop decision. It does not publish a result, a mutable
message, or a pointer. Therefore relaxed atomic loads/stores are sufficient;
result visibility will come from a separate mutex. The distinction between atomic
access and payload ordering is specified in
[atomic ordering](https://eel.is/c++draft/atomics.order). Adding a non-atomic
"reason" field beside the flag would require its own synchronization policy.

`CancellationScope` uses `std::exchange`: remember the old slot, install the new
pointer, then restore the old slot on scope exit. With no token installed, every
checkpoint is a cheap no-op and the synchronous earlier tests keep their
semantics. Never put a throwing checkpoint into a destructor or cleanup routine.
A stop must initiate cleanup, not repeatedly interrupt it.

## Place checks where the work is bounded

Add the token include and one check to each transfer loop in `src/io/file.cpp`:

```diff title="src/io/file.cpp"
--- a/src/io/file.cpp
+++ b/src/io/file.cpp
@@ -1,4 +1,5 @@
 #include "strata/io/file.hpp"
+#include "strata/runtime/cancel.hpp"
 #include "strata/core/checked.hpp"
 #include "strata/core/codec.hpp"
 
@@ -180,6 +181,7 @@
     (void)checked_file_end(offset, output.size());
     std::size_t done = 0;
     while (done < output.size()) {
+        runtime::checkpoint(runtime::Boundary::block);
         const auto count = std::min(output.size() - done, io_chunk_bytes);
         const Transfer result = operations_->read_at(
             descriptor_, output.subspan(done, count),
@@ -211,6 +213,7 @@
     (void)checked_file_end(offset, input.size());
     std::size_t done = 0;
     while (done < input.size()) {
+        runtime::checkpoint(runtime::Boundary::block);
         const auto count = std::min(input.size() - done, io_chunk_bytes);
         const Transfer result = operations_->write_at(
             descriptor_, input.subspan(done, count),
```

The check precedes each existing bounded transfer. If an earlier transfer already
wrote bytes, cancellation does not undo them. For query spill/output files those
partial effects belong to temporary owners and are cleaned up by their existing
failure paths. This does not turn arbitrary database writes into cancellable
transactions: the executor runs queries only.

A blocked native read or write cannot be interrupted by changing this C++ flag.
Response time includes the current call, scheduling, and work before the next
check. A chunk sort is also allowed to finish its bounded in-memory chunk before
observing a later boundary. There is no hard wall-clock cancellation deadline.

Add scan checks *inside* the exception-handling region so cancellation leaves
those streams terminal and releases their source ownership:

```diff title="src/query/scan.cpp"
--- a/src/query/scan.cpp
+++ b/src/query/scan.cpp
@@ -1,6 +1,7 @@
 #include "strata/query/scan.hpp"
 
 #include "strata/core/codec.hpp"
+#include "strata/runtime/cancel.hpp"
 
 namespace strata::query {
 
@@ -28,6 +29,7 @@
     output.clear();
     if (state_ == StreamState::ended) { return Pull::end; }
     try {
+        runtime::checkpoint(runtime::Boundary::batch);
         if (!started_) {
             cursor_ = view_.scan({Relation::events, 0}, Key{Relation::accounts, 0});
             started_ = true;
@@ -50,6 +52,7 @@
     output.clear();
     if (state_ == StreamState::ended) { return Pull::end; }
     try {
+        runtime::checkpoint(runtime::Boundary::batch);
         if (!started_) {
             cursor_ = view_.scan({Relation::accounts, 0});
             started_ = true;
```

Add a check before each materialized spill pass. The existing block loops and
source scans supply finer boundaries within long sorts; the pass check also
provides a precise test point after initial runs exist.

```diff title="src/query/external_sort.cpp"
--- a/src/query/external_sort.cpp
+++ b/src/query/external_sort.cpp
@@ -1,3 +1,4 @@
+#include "strata/runtime/cancel.hpp"
 #include "strata/query/joined_sort.hpp"
 #include "strata/core/codec.hpp"
 #include "strata/io/directory.hpp"
@@ -622,6 +623,7 @@
 
 template<class Schema>
 void TypedCore<Schema>::merge_pass() {
+    runtime::checkpoint(runtime::Boundary::spill_pass);
     std::size_t produced = 0;
     for (std::size_t first = 0; first < run_count_;) {
         const auto count = std::min(stats_->fan_in, run_count_ - first);
```

These are intentionally small amendments. Keep the existing partial-transfer
logic, terminal stream handling, run ownership, and replacement verification.
Cancellation composes with those mechanisms; it does not replace them with an
unexplained global abort.

## Completion has a point after which cancellation loses

Chapter 17's sink accumulates a temporary CSV and publishes only after valid
end-of-stream. We must decide what happens when cancellation races with its final
rename. The policy is:

| Observation point | Task outcome | Previously completed destination |
|---|---|---|
| Stop observed before the publication gate | Canceled | Preserved |
| Gate passed; `finish()` throws | Failed | Governed by the sink's failure contract |
| Gate passed; `finish()` successfully publishes | Completed | Replaced with the complete new result |

A cancellation request can arrive after the gate but before rename. It is too
late to demand cancellation from this operation. We mask cooperative checks
through `finish()` and report its actual outcome. Do not inspect the flag again
after success and retroactively label the published file canceled.

Amend `include/strata/query/result.hpp` as follows:

```diff title="include/strata/query/result.hpp"
--- a/include/strata/query/result.hpp
+++ b/include/strata/query/result.hpp
@@ -3,6 +3,7 @@
 
 #include "strata/query/aggregate.hpp"
 #include "strata/io/file.hpp"
+#include "strata/runtime/cancel.hpp"
 
 #include <array>
 #include <cstdint>
@@ -61,6 +62,7 @@
         GroupBatch output{plan.budget(), output_rows};
         sink.begin();
         for (;;) {
+            runtime::checkpoint(runtime::Boundary::batch);
             const auto result = plan.next(output);
             if (result == Pull::end) {
                 if (output.size() != 0) { throw std::logic_error("nonempty result end"); }
@@ -72,7 +74,9 @@
                 staged_groups = next_count;
             }
         }
-        sink.finish();  // The only publication boundary, after successful EOF.
+        runtime::checkpoint(runtime::Boundary::before_publish);
+        runtime::CancellationScope noncancelable{nullptr};
+        sink.finish();  // Completion wins a cancellation arriving after this gate.
     } catch (...) { sink.abort(); throw; }
 }
 }  // namespace strata::query
```

The output batch and plan still die through ordinary scopes. The noncancelable
scope dies before the surrounding catch or its caller runs. A genuine close or
rename failure remains an exception; masking a stop does not swallow I/O errors.
`CsvResult::finish` already performs no throwing operation after successful
rename. Other `ResultSink` implementations used with this policy must likewise
make successful return mean completed publication and must not throw after an
irreversible success.

`staged_groups` counts rows appended to the temporary sink. It is useful on a
canceled or failed query but does not mean those rows were externally published.
A query can stage both groups and then be canceled at `before_publish`, leaving
the earlier completed CSV untouched. These result semantics do not add crash
recovery or power-loss durability to temporary query output.

At this point we have safe synchronous building blocks for a worker: a private
read context, bounded allocations, cancellation points, and one publication rule.
[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-19-part-2.html) supplies
the ownership and synchronization that decide when those blocks may run.

