# Chapter 20 — Concurrent readers, ingestion, and maintenance

## Part 2 — Build outside the lock; install against current authority

The output files from [Part 1](https://25d-228.github.io/cplusplus-learning/chapter-20.html)
are prepared but not authoritative. A writer can have advanced both the active
log and the manifest while they were built. Installation must prove that the
selected inputs still mean what the job assumed, then retain every unrelated
newer dependency.

This part completes `maintenance.cpp` and supplies the whole `Engine`. It has no
additional hour allocation. All source files and small declaration changes in
Part 1 are prerequisites.

## Revalidate the inputs, not just the clock

Let $E_c$ be the manifest epoch captured during preparation and $E_n$ the epoch
now. An equality test $E_n=E_c$ is sufficient to show the manifest has not changed,
but it is unnecessarily restrictive: a new active log does not invalidate a
compaction of existing tables. Conversely, accepting every changed epoch without
checking source identities would permit a stale output to replace a different
prefix.

For flush, require the exact same oldest sealed `LogRecord` and one free table
slot. For compaction, require the same base coverage, the same selected table
records in the same positions, and enough capacity for the replacement plus the
**current** unselected suffix. Comparing records checks IDs, interval endpoints,
and encoded file lengths; comparing counts alone is insufficient.

After validation, start from `database.next_manifest()`, which copies the latest
manifest and advances its epoch. Never begin with `captured_`. The owner arrays
are likewise prepared from current owners before the authority transition.

Append this complete function and the namespace closing brace to
`src/storage/maintenance.cpp`:

```cpp title="src/storage/maintenance.cpp"
bool MaintenanceJob::install(Database& database) {
    if (&database != database_ || phase_ != Phase::built) { throw std::logic_error("invalid maintenance install"); }
    database.require_healthy();
    const auto& current = database.manifest_;
    bool same = false;
    if (kind_ == MaintenanceKind::flush) {
        same = current.log_count > 1 && current.logs[0] == sealed_ &&
               current.table_count < Manifest::max_tables;
    } else {
        same = current.base_count == captured_.base_count && current.base_end == captured_.base_end &&
               current.table_count >= selected_ &&
               output_count_ <= Manifest::max_tables - (current.table_count - selected_);
        for (std::size_t i = 0; same && i < selected_; ++i) {
            same = current.tables[i] == captured_.tables[i];
        }
    }
    if (!same) { phase_ = Phase::stale; return false; }
    // Copy the LATEST manifest, never the captured one: rotations and new suffixes survive.
    auto candidate = database.next_manifest();
    auto next_tables = database.tables_;
    auto next_logs = database.logs_;
    rebased_ = current.epoch != captured_.epoch;
    if (kind_ == MaintenanceKind::flush) {
        candidate.tables[candidate.table_count] = records_[0];
        next_tables[candidate.table_count++] = readers_[0]; candidate.flushed = sealed_.end;
        for (std::size_t i = 1; i < candidate.log_count; ++i) {
            candidate.logs[i - 1] = candidate.logs[i]; next_logs[i - 1] = next_logs[i];
        }
        candidate.logs[--candidate.log_count] = {}; next_logs[candidate.log_count].reset();
    } else {
        candidate.tables = {}; next_tables = {};
        for (std::size_t i = 0; i < output_count_; ++i) {
            candidate.tables[i] = records_[i]; next_tables[i] = readers_[i];
        }
        for (std::size_t i = selected_; i < current.table_count; ++i) {
            candidate.tables[output_count_ + i - selected_] = current.tables[i];
            next_tables[output_count_ + i - selected_] = database.tables_[i];
        }
        candidate.base_count = output_count_; candidate.base_end = end_;
        candidate.table_count = output_count_ + current.table_count - selected_;
    }
    database.poisoned_ = true;  // From authority I/O onward a failure requires reopen.
    database.install(candidate);
    database.tables_.swap(next_tables); database.logs_.swap(next_logs);
    database.poisoned_ = false; phase_ = Phase::installed; stats_.installed = true;
    if (kind_ == MaintenanceKind::compact) { database.compaction_ = stats(); }
    return true;
}
}  // namespace strata::storage
```

Suppose a flush captured logs `[L1(0,1], L2(active after 1)]`. Before installation,
the writer commits sequence 2 and rotates, producing
`[L1(0,1], L2(1,2], L3(active after 2)]`. The flushed `L1` is still valid. The
candidate must retain `L2` and `L3`, not resurrect the captured active-log entry.
The first regression test in Part 3 checks exactly this case.

For compaction, consider tables `[T1, T2]` captured through sequence 2 and a
later table `T3` covering sequence 3. Replacement `[B1]` produces `[B1,T3]`, not
just `[B1]`. If compaction removes every row in its prefix, `base_count` becomes
zero but `base_end` retains the eliminated coverage. A future older source must
not be allowed to resurrect a deleted value.

A stale result returns `false` without changing authority and consumes the job.
The coordinator may prepare a new job from current state. It must not call
`build` again on the stale object or reuse the old file identities.

## The authority and serving-state boundary

`database.install` performs Chapter 11's dependency, manifest, and namespace
synchronization. From that call onward, failure poisons the database until
reopen. The next table/log arrays already exist. Swapping their owners after a
successful install allocates nothing.

The engine mutex spans installation and both swaps. A new capture cannot see
the new manifest with old owner arrays. An existing capture needs no such lock:
it owns its previous immutable source set. “Atomic publication” here means one
indivisible observation through the coordinator, not a bitwise atomic store of
an entire manifest or a new `std::atomic<shared_ptr<...>>` design.

Destruction of an old array may decrement shared ownership counts, but snapshots
still retain their own sources. Their final release is safe because Part 1
synchronized the ledger and registry. Physical removal comes later, after the
job itself has died.

## Define the coordinator's public contract

`Engine` implements `QueryInputs`, so the Chapter 19 executor can hold either a
frozen fixture or a live engine. `final` prevents further derivation of this
coordinator. An application may call `write`, capture through the executor, and
request maintenance concurrently. It must not destroy the engine while another
thread is executing an API call without owning its lifetime.

`stop` requests shutdown and wakes writers; `finish` is a **controller-only**
stop-and-join operation. Do not call `finish` concurrently, from the maintenance
thread, or from a hook. Calling it again serially is harmless except that a
stored error is rethrown again. Explicit shutdown exposes worker failure;
destruction joins but cannot throw it to an absent caller.

The header uses an incomplete nested `State` held by `unique_ptr`. The state
contains mutexes and is never moved. Define the destructor out of line after
`State` is complete, so destruction sees the actual owned type. This is simply
a way to keep synchronization details out of the public header, not a second
allocation layer for each query.

```cpp title="include/strata/runtime/engine.hpp"
#ifndef STRATA_RUNTIME_ENGINE_HPP
#define STRATA_RUNTIME_ENGINE_HPP
#include "strata/runtime/frozen_inputs.hpp"
#include "strata/storage/maintenance.hpp"

#include <cstddef>
#include <cstdint>
#include <exception>
#include <memory>
#include <span>
#include <string>

namespace strata::runtime {
struct EngineOptions {
    storage::DatabaseOptions storage{};
    storage::CompactionOptions compaction{};
    std::size_t storage_bytes = 64 * 1024 * 1024;
    std::size_t maintenance_bytes = 8 * 1024 * 1024;
    std::size_t maintenance_files = 33;
    std::size_t batches_per_log = 8;
    std::size_t compact_after = 4;
};
struct EngineStats {
    std::uint64_t sequence = 0, epoch = 0;
    std::size_t logs = 0, tables = 0, waiting_writers = 0;
    std::size_t flushes = 0, compactions = 0, rebased = 0, stale = 0, abandoned = 0;
    std::size_t storage_used = 0, maintenance_used = 0, maintenance_peak = 0;
    bool busy = false, stopped = false;
    storage::ReadRetention retained{};
    storage::CompactionStats last_job{};
    storage::ReclaimResult cleanup{};
    std::exception_ptr error;
};
class Engine final : public QueryInputs {
public:
    Engine(const std::string& directory, storage::DatabaseOpen mode,
           EngineOptions options = {}, storage::InstallHook maintenance_hook = {},
           storage::InstallHook installation_hook = {});
    ~Engine() noexcept override;
    Engine(const Engine&) = delete;
    Engine& operator=(const Engine&) = delete;
    [[nodiscard]] storage::WriteResult write(std::span<const storage::Mutation> changes);
    bool seal();  // Seal a nonempty active log, waiting only for a free log slot.
    void request_compaction();
    void request_collection();
    void await_idle();
    void await_writers(std::size_t count);  // Test rendezvous with a watchdog.
    void stop() noexcept;                  // Wakes blocked writers; no join.
    void finish();                        // Controller only: stop, join, rethrow error.
    EngineStats stats() const;
    storage::ReadView capture(std::shared_ptr<memory::Budget> budget) const override;
    std::size_t source_count() const override { return storage::read_detail::max_sources; }
private:
    struct State;
    std::unique_ptr<State> state_;
};
}  // namespace strata::runtime
#endif
```

Add the explicit stopped-writer reason. A known closed engine rejects before
appending the submitted batch; this differs from a write whose synchronization
outcome is uncertain. An existing stored engine error is rethrown instead of
being mislabeled as ordinary closure.

```diff title="include/strata/storage/wal.hpp — exact prerequisite changes"
--- include/strata/storage/wal.hpp
+++ include/strata/storage/wal.hpp
@@ -28,7 +28,7 @@
 enum class WriteOutcome { committed, rejected, uncertain };
 enum class WriteReason {
     none, invalid_batch, resource_exhausted, log_full, sequence_exhausted,
-    writer_poisoned, writer_sealed, io_failure, unexpected_failure
+    writer_poisoned, writer_sealed, engine_closed, io_failure, unexpected_failure
 };
 
 struct WriteResult {
```

## Build state before starting its thread

Start `src/runtime/engine.cpp`. Configuration validation rejects descriptor
allowances that cannot support the declared fixed upper bound. Memory capacity
is enforced by the allocators, rather than guessed from a platform-dependent
`sizeof` total. The low-capacity test will fail visibly when even a job cannot
fit.

`database` is constructed before the thread. Recovered active contents set the
batch threshold as already reached, so the next write seals them before growing
that generation again. Fresh engines count committed batches; the threshold
is neither a byte limit nor a timer. Arena and WAL limits still apply.

```cpp title="src/runtime/engine.cpp"
#include "strata/runtime/engine.hpp"
#include "strata/runtime/cancel.hpp"

#include <atomic>
#include <cerrno>
#include <chrono>
#include <condition_variable>
#include <mutex>
#include <stdexcept>
#include <system_error>
#include <thread>
#include <utility>

namespace strata::runtime {
namespace {
EngineOptions checked(EngineOptions options) {
    if (options.storage_bytes == 0 || options.maintenance_bytes == 0 ||
        options.batches_per_log == 0 || options.compact_after < 2 ||
        options.compact_after > storage::Manifest::max_tables ||
        options.compaction.max_output_files == 0 ||
        options.compaction.max_output_files > storage::Manifest::max_tables ||
        options.maintenance_files < storage::Manifest::max_tables + options.compaction.max_output_files + 1) {
        throw std::invalid_argument("invalid engine allowances");
    }
    return options;
}
}  // namespace
struct Engine::State {
    State(const std::string& directory, storage::DatabaseOpen mode, EngineOptions config,
          storage::InstallHook work_hook, storage::InstallHook install_hook)
        : options(checked(config)), storage_budget(std::make_shared<memory::Budget>(options.storage_bytes)),
          maintenance_budget(std::make_shared<memory::Budget>(options.maintenance_bytes)),
          database(storage_budget, directory, mode, options.storage, install_hook), hook(work_hook) {
        const auto& manifest = database.manifest();
        if (database.sequence() != manifest.logs[manifest.log_count - 1].base) {
            batches = options.batches_per_log;  // Recovered active contents: seal before growing again.
        }
    }
    bool compact_needed() const;
    bool pending() const;
    bool wait_slot(std::unique_lock<std::mutex>& lock);
    void fail(std::exception_ptr error) noexcept;  // Caller owns mutex.
    void collect();                              // Caller owns mutex; no job exists.
    void run() noexcept;
    void stop() noexcept;
    void join() noexcept { if (worker.joinable()) { worker.join(); } }
    void require_open() const {
        if (counts.error) { std::rethrow_exception(counts.error); }
        if (stopping.load(std::memory_order_relaxed)) { throw std::logic_error("engine stopped"); }
    }
    EngineOptions options;
    std::shared_ptr<memory::Budget> storage_budget, maintenance_budget;
    storage::Database database;
    storage::InstallHook hook;
    mutable std::mutex mutex;
    std::condition_variable changed, observed;
    std::atomic<bool> stopping{false};
    std::size_t batches = 0;
    bool compact_requested = false, collect_requested = false;
    EngineStats counts{};
    std::thread worker;  // Joined before any of the preceding fields are destroyed.
};
```

Declaration order gives destruction order in reverse. The thread is declared
last but is explicitly joined *before* state destruction starts. If thread
creation throws, the unique state owner destroys a fully constructed database
with no started worker. We do not detach and hope the borrowed `State*` remains
alive.

`counts`, `batches`, pending flags, and the database require `mutex`. The stop
flag is also read while building outside the lock, so it is atomic. Relaxed
loads suffice for that Boolean request: no manifest or error object is published
through it. Those objects are still observed under the mutex. The distinction
between relaxed atomics and cross-object synchronization is described in
[data races](https://eel.is/c++draft/intro.races).

## Admit work that can actually free a slot

The worker prefers a requested or pressure-triggered full-prefix compaction,
otherwise the oldest sealed log. Base-only repacking at a full table set is
allowed, as in Chapter 13. It may create the slot needed by a waiting flush.
If a completed repack still leaves all 16 slots occupied with sealed logs
waiting, fail instead of scheduling the same ineffective work forever.

```cpp title="src/runtime/engine.cpp"
bool Engine::State::compact_needed() const {
    const auto& m = database.manifest();
    return m.table_count != 0 && (compact_requested ||
        (m.table_count >= options.compact_after && m.table_count > m.base_count) ||
        (m.table_count == storage::Manifest::max_tables && m.log_count > 1));
}
bool Engine::State::pending() const {
    return compact_needed() || database.manifest().log_count > 1 || collect_requested;
}
```

A writer reaches backpressure when it must rotate and all four log slots are
occupied. It increments the diagnostic waiting count and calls `wait` with the
engine lock. Waiting atomically releases that mutex; returning reacquires it.
The loop rechecks both capacity and terminal state, including spurious wakeups.
This is the condition-variable contract in
[the standard library specification](https://eel.is/c++draft/thread.condition.condvar).

Holding the mutex while polling or sleeping would prevent the maintenance
installer from removing the oldest log. Notification is not the resource:
`manifest.log_count < max_logs` is the resource predicate.

```cpp title="src/runtime/engine.cpp"
bool Engine::State::wait_slot(std::unique_lock<std::mutex>& lock) {
    while (!counts.error && !stopping.load(std::memory_order_relaxed) &&
           database.manifest().log_count == storage::Manifest::max_logs) {
        ++counts.waiting_writers; observed.notify_all(); changed.notify_all();
        changed.wait(lock);  // Releases the same mutex the installer needs to free a slot.
        --counts.waiting_writers;
    }
    if (counts.error) { std::rethrow_exception(counts.error); }
    return !stopping.load(std::memory_order_relaxed);
}
void Engine::State::fail(std::exception_ptr error) noexcept {
    if (!counts.error) { counts.error = std::move(error); }
    stopping.store(true, std::memory_order_relaxed);
    changed.notify_all(); observed.notify_all();
}
void Engine::State::stop() noexcept {
    { std::lock_guard lock{mutex}; stopping.store(true, std::memory_order_relaxed); }
    changed.notify_all(); observed.notify_all();
}
void Engine::State::collect() {
    counts.cleanup = database.collect_obsolete();
    if (counts.cleanup.failed || counts.cleanup.sync_failed) {
        throw std::system_error(counts.cleanup.first_error ? counts.cleanup.first_error : EIO,
                                std::generic_category(), "maintenance cleanup incomplete");
    }
}
```

Failure stores the first exception and requests stop. Cleanup refusal is visible
as either failed deletion or failed synchronization; it is not silently declared
successful because the serving manifest is valid. Old snapshots can still read
their isolated sources after the engine stops accepting work.

There is no automatic busy retry of a failed I/O operation. A failed install may
already have replaced the manifest name, so reopening—not replaying the same
mutation on a poisoned object—decides authority.

## One worker, two critical sections, one private build

The next function is the worker state machine. Follow the ownership of `job`:
creation under the lock, build outside, validation/installation under the lock,
then destruction outside the lock. Collection starts only after that last owner
has gone. This ordering is what protects names during off-lock reads.

The shared job control block is charged to the maintenance budget. It has only
one controlling worker owner; shared ownership here is used for the existing
allocator facility, not to permit concurrent `build` and `stats` calls. Engine
statistics are copied only after building stops.

```cpp title="src/runtime/engine.cpp"
void Engine::State::run() noexcept {
    for (;;) {
        std::shared_ptr<storage::MaintenanceJob> job;
        try {
            {
                std::unique_lock lock{mutex};
                changed.wait(lock, [&] { return stopping.load(std::memory_order_relaxed) || pending(); });
                if (stopping.load(std::memory_order_relaxed)) { break; }
                counts.busy = true;
                if (compact_needed()) {
                    job = std::allocate_shared<storage::MaintenanceJob>(
                        memory::SharedAllocator<storage::MaintenanceJob>{maintenance_budget},
                        database, maintenance_budget, storage::MaintenanceKind::compact,
                        database.manifest().table_count, options.compaction);
                    compact_requested = false;
                } else if (database.manifest().log_count > 1) {
                    job = std::allocate_shared<storage::MaintenanceJob>(
                        memory::SharedAllocator<storage::MaintenanceJob>{maintenance_budget},
                        database, maintenance_budget, storage::MaintenanceKind::flush);
                } else {
                    collect_requested = false; collect(); counts.busy = false;
                    changed.notify_all(); continue;
                }
            }
            Cancellation token{stopping};
            { CancellationScope cancelable{&token}; job->build(hook); }
            bool installed = false;
            const bool compacted = job->kind() == storage::MaintenanceKind::compact;
            {
                std::lock_guard lock{mutex};
                if (!stopping.load(std::memory_order_relaxed)) {
                    // Authority installation is deliberately not cancelable.
                    CancellationScope noncancelable{nullptr};
                    installed = job->install(database);
                    counts.last_job = job->stats();
                    if (installed) {
                        if (job->kind() == storage::MaintenanceKind::flush) { ++counts.flushes; }
                        else { ++counts.compactions; }
                        if (job->rebased()) { ++counts.rebased; }
                    } else { ++counts.stale; }
                } else { ++counts.abandoned; }
            }
            job.reset();  // Release input owners and unused descriptors before collection.
            {
                std::lock_guard lock{mutex};
                if (installed && !stopping.load(std::memory_order_relaxed)) {
                    collect();
                    if (compacted && database.manifest().table_count == storage::Manifest::max_tables &&
                        database.manifest().log_count > 1) {
                        throw std::length_error("compaction left no table slot for flush");
                    }
                }
                counts.busy = false; changed.notify_all();
            }
        } catch (const Canceled&) {
            job.reset();
            std::lock_guard lock{mutex}; counts.busy = false; ++counts.abandoned;
            if (!stopping.load(std::memory_order_relaxed)) { fail(std::current_exception()); }
            changed.notify_all(); break;
        } catch (...) {
            const auto error = std::current_exception();
            if (job) {
                const auto report = job->stats(); job.reset();
                std::lock_guard lock{mutex}; counts.last_job = report;
            }
            std::lock_guard lock{mutex}; counts.busy = false; fail(error); break;
        }
    }
}
```

`CancellationScope` is thread-local. The build observes the engine stop token,
not an unrelated query's token. Installing a completed candidate has cancellation
masked: a cancellation exception must not split the durability sequence. If
stop wins the engine mutex before installation, the prepared job is abandoned.
If installation wins first, that valid installation may finish before shutdown
returns. Both outcomes preserve acknowledged writes.

On a build exception, the job dies before the engine stores the terminal error.
Temporary buffers and input readers disappear; incomplete files remain
nonauthoritative. On an installation exception, the database may be poisoned,
and the same visible error policy prevents further writes or captures. Do not
collect questionable artifacts through that old object.

`await_idle` is a controller observation, not a way to seize the worker. It
means no pending work and no active job at the observed point. Another writer
can create new work immediately afterward. Tests use controlled pauses when a
stronger ordering is required.

## Own the thread through every public exit

Constructing the state completes before its pointer is captured by the worker.
The destructor always stops and joins. A serial explicit `finish` additionally
rethrows the stored error after joining. It does not flush the final active log:
committed data is already represented by the WAL, and normal reopen replays it.

```cpp title="src/runtime/engine.cpp"
Engine::Engine(const std::string& directory, storage::DatabaseOpen mode, EngineOptions options,
               storage::InstallHook maintenance_hook, storage::InstallHook installation_hook)
    : state_(std::make_unique<State>(directory, mode, options, maintenance_hook, installation_hook)) {
    state_->worker = std::thread([state = state_.get()] { state->run(); });
}
Engine::~Engine() noexcept { state_->stop(); state_->join(); }
void Engine::stop() noexcept { state_->stop(); }
void Engine::finish() {
    state_->stop(); state_->join();
    const auto report = stats();
    if (report.error) { std::rethrow_exception(report.error); }
}
```

To drain already accepted queries normally, call `executor.finish()` before
`engine.finish()`. To cancel them, request the executor's cancellation and join
its workers, then finish the engine. Stopping the engine first can cause an
accepted but not yet captured query to fail capture; already captured views
remain usable. No contract promises an immediate stop of a blocked filesystem
call.

The application must also join its own writer threads. The engine owns the
maintenance thread, not every caller that happens to invoke `write`. The
backpressure test uses an explicit guard that requests engine stop before
joining its external writer.

## Keep write outcomes distinct

Writes are serialized through the engine mutex. When the committed-batch
threshold is reached, the next write waits for a log slot and rotates. A write
that is rejected by the underlying WAL does not increment the batch count.
A committed write increments it once. An uncertain outcome stops future engine
activity but returns that original uncertainty to its caller.

```cpp title="src/runtime/engine.cpp"
storage::WriteResult Engine::write(std::span<const storage::Mutation> changes) {
    auto& s = *state_; std::unique_lock lock{s.mutex};
    if (s.counts.error) { std::rethrow_exception(s.counts.error); }
    if (s.stopping.load(std::memory_order_relaxed)) {
        return {storage::WriteOutcome::rejected, storage::WriteReason::engine_closed, 0};
    }
    CancellationScope noncancelable{nullptr};
    if (s.batches == s.options.batches_per_log) {
        if (!s.wait_slot(lock)) {
            return {storage::WriteOutcome::rejected, storage::WriteReason::engine_closed, 0};
        }
        try { (void)s.database.rotate(); s.batches = 0; s.changed.notify_all(); }
        catch (...) { if (s.database.poisoned()) { s.fail(std::current_exception()); } throw; }
    }
    const auto answer = s.database.write(changes);
    if (answer.outcome == storage::WriteOutcome::committed) { ++s.batches; }
    if (answer.outcome == storage::WriteOutcome::uncertain) {
        s.fail(std::make_exception_ptr(std::runtime_error("uncertain write: reopen engine")));
    }
    return answer;
}

bool Engine::seal() {
    auto& s = *state_; std::unique_lock lock{s.mutex}; s.require_open();
    const auto& m = s.database.manifest();
    if (s.database.sequence() == m.logs[m.log_count - 1].base) { return false; }
    if (!s.wait_slot(lock)) { return false; }
    CancellationScope noncancelable{nullptr};
    try {
        const bool changed = s.database.rotate(); s.batches = 0; s.changed.notify_all(); return changed;
    } catch (...) { if (s.database.poisoned()) { s.fail(std::current_exception()); } throw; }
}
```

The submitted span is borrowed for the duration of `write`; it must remain
valid even while the caller waits for a log slot. Concurrent callers each keep
their own stable input buffers. The coordinator does not queue an unbounded copy
of their batches.

With one batch per log, the shutdown fixture holds three sealed logs and one
active log containing batch 4. A fifth writer cannot rotate. Stop changes the
predicate and wakes it; it returns `rejected/engine_closed` with sequence zero.
Batches 1–4 remain in the authoritative logs, even if their pending flush is
abandoned.

`seal` is a controller convenience for a nonempty active log. The worker does
not periodically flush a mutable memtable. Explicit sealing, a snapshot capture,
or the next threshold-crossing write produces immutable maintenance input.

## Capture under the mutex; construct readers outside it

Capture first produces and registers one immutable parent view under the engine
mutex. Snapshot pressure is rejected before rotation by the existing database
admission check. A full log set or retained-source limit is not bypassed by
allocating an extra generation. This implementation rejects a pressured capture
rather than sleeping inside capture; the executor will release its query permit
when that request fails.

After the parent exists, the mutex is released. Private file opens, metadata
reads, and cache allocation then happen outside it. Parent retention protects
every name during those opens. A caller's cancellation cannot interrupt the
rotation/authority mutation, but ordinary query checkpoints remain active during
private reader construction and execution.

```cpp title="src/runtime/engine.cpp"
storage::ReadView Engine::capture(std::shared_ptr<memory::Budget> budget) const {
    if (!budget) { throw std::invalid_argument("missing query budget"); }
    auto& s = *state_; storage::ReadView parent;
    {
        std::lock_guard lock{s.mutex}; s.require_open();
        CancellationScope noncancelable{nullptr};
        try {
            const auto logs = s.database.manifest().log_count;
            parent = s.database.snapshot();
            if (logs != s.database.manifest().log_count) { s.batches = 0; }
            s.changed.notify_all();
        } catch (...) { if (s.database.poisoned()) { s.fail(std::current_exception()); } throw; }
    }
    // Parent retention protects names through every read-only reopen. No engine lock
    // covers private reader allocation, table I/O, query execution, or parent release.
    return parent.isolate(std::move(budget), s.options.storage.cache_slots,
                          s.options.storage.cache_bytes, true);
}
```

There is a deliberate asymmetry: a waiting writer releases the engine lock and
can resume when maintenance frees a log slot; a capture under retention pressure
fails immediately. Neither operation waits while holding the lock needed to
relieve its pressure. A caller that chooses to retry must use an external
policy tied to released resources, not an unbounded busy loop in this API.

The remaining methods request maintenance, observe completion, expose a test
rendezvous, and copy statistics. `await_writers` has a watchdog so a broken test
fails rather than hanging the entire suite; production progress never depends
on its timeout.

```cpp title="src/runtime/engine.cpp"
void Engine::request_compaction() {
    auto& s = *state_; std::lock_guard lock{s.mutex}; s.require_open();
    if (s.database.manifest().table_count != 0) { s.compact_requested = true; }
    s.changed.notify_all();
}
void Engine::request_collection() {
    auto& s = *state_; std::lock_guard lock{s.mutex}; s.require_open();
    s.collect_requested = true; s.changed.notify_all();
}
void Engine::await_idle() {
    auto& s = *state_; std::unique_lock lock{s.mutex};
    s.changed.wait(lock, [&] {
        return s.counts.error || (!s.counts.busy &&
            (s.stopping.load(std::memory_order_relaxed) || !s.pending()));
    });
    if (s.counts.error) { std::rethrow_exception(s.counts.error); }
}
void Engine::await_writers(std::size_t count) {
    auto& s = *state_; std::unique_lock lock{s.mutex};
    if (!s.observed.wait_for(lock, std::chrono::seconds{5}, [&] { return s.counts.waiting_writers >= count; })) {
        throw std::runtime_error("writer rendezvous watchdog");
    }
}
EngineStats Engine::stats() const {
    auto& s = *state_; std::lock_guard lock{s.mutex}; auto result = s.counts;
    const auto& manifest = s.database.manifest();
    result.sequence = s.database.sequence(); result.epoch = manifest.epoch;
    result.logs = manifest.log_count; result.tables = manifest.table_count;
    result.stopped = s.stopping.load(std::memory_order_relaxed); result.retained = s.database.retention();
    result.storage_used = s.storage_budget->used(); result.maintenance_used = s.maintenance_budget->used();
    result.maintenance_peak = s.maintenance_budget->peak(); return result;
}
}  // namespace strata::runtime
```

## Check the lock order against actual call paths

An executor releases its scheduler mutex before invoking the provider. Engine
operations may acquire the registry or budget mutex briefly. The registry and
budget do not call back into the engine. Private cache operations acquire their
budget ledger only while accounting; no shared cache lock is added.

A typical final release follows this chain:

```text
query worker drops isolated view
  -> private readers/cache and query allocations release
  -> parent state drops its registered source references
  -> registry mutex updates retained counts
  -> source owners release arenas/metadata through the storage budget mutex
  -> final context, if any, closes the directory lock
```

The registry lock is not held while the owned sources are destroyed. The
allocator's ledger lock is not held while arbitrary query code executes. Do not
introduce a callback from either bookkeeping layer that acquires the engine
mutex; it would create the reverse edge of the lock order.

`EngineStats` coherently reports database sequence/epoch/counts while holding
the engine mutex. Its ledger observations are individually protected but not one
atomic global RSS sample; query releases can change charges between observations.
`last_job` is a completed-or-failed job report, not a race-prone live pointer into
an active builder.

The stopped engine and its maintenance thread can be gone while a captured
query still owns its parent context. That is intentional. Source files and the
directory lock remain alive until that last snapshot ends. The next part tests
both that retention and the final release on a different thread.

Continue with [Part 3 — Force publication, failure, and shutdown boundaries](https://25d-228.github.io/cplusplus-learning/chapter-20-part-3.html).

