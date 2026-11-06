# Chapter 20 — Concurrent readers, ingestion, and maintenance

## Part 3 — Force publication, failure, and shutdown boundaries

We will not infer overlap from a fast wall-clock result. A gate will hold the
maintenance worker outside the writer mutex, and the writer must commit and
rotate before that gate is released. Another gate holds an old query after
capture. A third holds a query at a real spill pass so cancellation must clean
actual files without publishing partial output.

Complete [Part 1](https://25d-228.github.io/cplusplus-learning/chapter-20.html) and
[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-20-part-2.html) first.
All blocks named `tests/engine.cpp` below concatenate into one complete program.
No mock query replaces the Chapter 17 two-sort join/aggregation pipeline.

## Construct small but structurally demanding data

Create the test file with its imports, ordinary throwing checks, temporary
namespace owner, and text reader. `TemporaryDirectory` is a test owner only;
production cleanup failures remain visible through the engine and task reports.
The test removes its private root during cleanup and never targets a caller's
existing directory unless the explicit fixture mode is selected.

```cpp title="tests/engine.cpp"
#include "strata/runtime/engine.hpp"
#include "strata/runtime/executor.hpp"
#include "strata/core/codec.hpp"

#include <array>
#include <atomic>
#include <cerrno>
#include <chrono>
#include <condition_variable>
#include <cstdlib>
#include <cstdint>
#include <exception>
#include <filesystem>
#include <fstream>
#include <iostream>
#include <iterator>
#include <memory>
#include <mutex>
#include <stdexcept>
#include <string>
#include <string_view>
#include <system_error>
#include <thread>
#include <vector>

namespace {
using namespace strata;
using namespace strata::storage;
using namespace strata::query;
using namespace strata::runtime;
namespace fs = std::filesystem;
constexpr std::size_t query_bytes = 2 * 1024 * 1024;
void check(bool value, const char* message) { if (!value) { throw std::runtime_error(message); } }
template<class Error, class F> void expect(F action) {
    try { action(); } catch (const Error&) { return; }
    throw std::runtime_error("missing expected exception");
}
class TemporaryDirectory {
public:
    TemporaryDirectory() {
        char pattern[] = "/tmp/strata-engine-XXXXXX";
        const auto p = ::mkdtemp(pattern);
        if (!p) { throw std::system_error(errno, std::generic_category(), "test directory"); }
        path_ = p;
    }
    ~TemporaryDirectory() noexcept { std::error_code ignored; fs::remove_all(path_, ignored); }
    const std::string& name() const { return path_; }
private: std::string path_;
};
std::string read(const fs::path& path) {
    std::ifstream file{path}; check(static_cast<bool>(file), "missing test file");
    return {std::istreambuf_iterator<char>{file}, {}};
}
```

Use small blocks and three-row sort chunks so the input must spill. The two
accounts initially map to regions 1 and 2. Event IDs are $100+i$ for
$0\le i<60$, amounts are $i+1$, and every third event belongs to account 20.
Thus all values and expected answers remain small enough to inspect manually.

```cpp title="tests/engine.cpp"
EngineOptions options() {
    EngineOptions o; o.storage.arena_bytes = 256 * 1024;
    o.storage.cache_slots = 2; o.storage.cache_bytes = 256;
    o.storage.table = {128, 256, 32, 64 * 1024};
    o.compaction.output = o.storage.table; o.compaction.max_output_files = 4;
    o.compaction.max_output_bytes = 256 * 1024;
    o.maintenance_files = Manifest::max_tables + 4 + 1;
    o.maintenance_bytes = 2 * 1024 * 1024;
    o.compact_after = 16; return o;
}
QueryOptions query_options() {
    QueryOptions q; q.days = {3, 4}; q.upstream_rows = 2; q.join = {2, 2, 4};
    q.joined_rows = 2; q.aggregate = {2, 4}; q.output_rows = 1;
    SortOptions sort; sort.chunk_rows = 3; sort.block_rows = 2; sort.file_descriptors = 3;
    sort.workspace_bytes = 128 * 1024; sort.stack_allowance = 16 * 1024; sort.disk_bytes = 128 * 1024;
    q.event_sort = sort; q.group_sort = sort; return q;
}
std::vector<Mutation> original() {
    std::vector<Mutation> rows{Mutation::put(Account{10, 1}), Mutation::put(Account{20, 2})};
    for (std::uint64_t i = 0; i < 60; ++i) {
        rows.push_back(Mutation::put(Event{100 + i, i % 3 == 0 ? 20U : 10U, 3, i + 1}));
    }
    return rows;
}
void committed(WriteResult r, std::uint64_t sequence) {
    check(r.outcome == WriteOutcome::committed && r.sequence == sequence, "unexpected write outcome");
}
std::uint64_t cents(const ReadView& view, std::uint64_t id) {
    auto row = view.get({Relation::events, id}); check(row.has_value(), "missing event");
    return decode_event(row->view().value).amount_cents;
}
TaskResult result(const TaskHandle& handle, bool cleaned = true) {
    auto r = handle.wait_for(std::chrono::seconds{15}); check(r.has_value(), "query watchdog");
    check(r->final_bytes == 0, "query retained charged owners after terminal result");
    if (cleaned) {
        check(r->events.live_files == 0 && r->groups.live_files == 0 &&
              !r->events.directory_live && !r->groups.directory_live && r->result_cleanup_error == 0,
              "query cleanup incomplete");
    }
    return *r;
}
```

`TaskHandle::wait_for` is a test watchdog, not a claim about production latency.
A terminal task must report zero charged query bytes and no unresolved spill or
result cleanup. We still read the output files independently; an empty resource
ledger does not prove a correct query answer.

For region 2, the amounts are $1,4,7,\ldots,58$: twenty values. Their total is

$$
20(1+58)/2=590.
$$

All sixty amounts total $60\cdot61/2=1830$, leaving $1830-590=1240$ for the forty
region-1 events. At sequence 2, account 10 moves to region 3, event 101 changes
from 2 cents to 900, and event 102's 3 cents are deleted. Its new count is 39
and total is $1240-2+900-3=2135$. At sequence 3, account 20 moves to region 4
and a new 100-cent event adds one row: count 21 and total $590+100=690$.

The query must select one account/event snapshot together. Reading the new
accounts with old events would produce plausible numbers under the wrong
region IDs, so comparing only the grand total would miss the error.

## Establish the pause, then perform the competing operation

The `Gate` owns its mutex and condition variable. A callback announces arrival
while holding the gate mutex, then waits on a predicate that releases it. The
controller observes that arrival before performing the competing action.
The timeouts report broken rendezvous; no sleep is used to guess whether the
other thread has reached a boundary.

`ReleaseGates` is created after the engine/executor owners. During unwinding it
releases their borrowed callbacks before those owners try to join workers. A
test assertion must not strand the very worker that a destructor is joining.

```cpp title="tests/engine.cpp"
class Gate {
public:
    explicit Gate(std::string_view stage = {}, Boundary boundary = Boundary::captured)
        : stage_(stage), boundary_(boundary) {}
    static void maintenance(void* pointer, std::string_view stage) {
        auto& gate = *static_cast<Gate*>(pointer); if (stage == gate.stage_) { gate.arrive(); }
    }
    static void query(void* pointer, Boundary boundary) {
        auto& gate = *static_cast<Gate*>(pointer); if (boundary == gate.boundary_) { gate.arrive(); }
    }
    void arrive() {
        std::unique_lock lock{mutex_}; arrived_ = true; changed_.notify_all();
        if (!changed_.wait_for(lock, std::chrono::seconds{12}, [&] { return open_; })) {
            throw std::runtime_error("pause watchdog");
        }
    }
    void await() {
        std::unique_lock lock{mutex_};
        if (!changed_.wait_for(lock, std::chrono::seconds{5}, [&] { return arrived_; })) {
            throw std::runtime_error("arrival watchdog");
        }
    }
    void release() noexcept { { std::lock_guard lock{mutex_}; open_ = true; } changed_.notify_all(); }
private:
    std::string_view stage_; Boundary boundary_; std::mutex mutex_; std::condition_variable changed_;
    bool arrived_ = false, open_ = false;
};
struct ReleaseGates {
    Gate& a; Gate& b; Gate& c;
    ~ReleaseGates() { a.release(); b.release(); c.release(); }
};
```

Hooks borrow their context pointers. Keep the gate or failure object alive
until every worker using it has joined. Maintenance build hooks run without
the engine mutex; installation hooks run inside the authority critical section.
Neither is a general-purpose reentrant extension point. In particular, do not
call `engine.stats()` from an installation hook that already owns that mutex.

## Run the complete overlap fixture

The executor has three worker threads but permits only two simultaneous full
query reservations: $2\times2$ MiB and $2\times29$ descriptors. The old query
and cancelable query occupy that entire pool while both are paused. Maintenance
must still install its prepared compaction and flush the later log.

The sequence is explicit: flush sequence 1, hold its query, commit/flush sequence
2, pause compaction, commit/rotate sequence 3, and hold another query at a spill
pass. Only then release maintenance. Check its result before releasing either
query. This tests progress independent of the query pool, not just different
threads running at unrelated times.

```cpp title="tests/engine.cpp"
void check_overlap(const fs::path& root) {
    fs::create_directories(root / "db"); fs::create_directories(root / "spill");
    fs::create_directories(root / "out");
    Gate maintenance{"compact-build"}, old_query, canceled_query{{}, Boundary::spill_pass};
    {
        auto engine = std::make_shared<Engine>((root / "db").string(), DatabaseOpen::create,
                                             options(), InstallHook{Gate::maintenance, &maintenance});
        committed(engine->write(original()), 1); check(engine->seal(), "initial seal"); engine->await_idle();
        check(engine->stats().tables == 1, "initial flush");
        Executor executor{engine, (root / "spill").string(), (root / "out").string(),
                          {3, 4, 2 * query_bytes, 2 * (read_detail::max_sources + 9)}};
        ReleaseGates release{maintenance, old_query, canceled_query};
        auto old = std::make_unique<Request>(query_options(), query_bytes, "old.csv");
        old->checkpoints = {Gate::query, &old_query}; auto old_handle = old->handle();
        check(executor.submit(old) == Submit::accepted, "old query submit"); old_query.await();
        const std::array update{Mutation::put(Account{10, 3}), Mutation::put(Event{101, 10, 3, 900}),
                                Mutation::erase({Relation::events, 102})};
        committed(engine->write(update), 2); check(engine->seal(), "updated seal"); engine->await_idle();
        engine->request_compaction(); maintenance.await();
        // The maintenance scan is paused OUTSIDE the writer mutex. This commit and
        // rotation must both finish before the pause is released.
        const std::array later{Mutation::put(Account{20, 4}), Mutation::put(Event{160, 20, 3, 100})};
        committed(engine->write(later), 3); check(engine->seal(), "overlapping rotation");
        check(engine->stats().busy && engine->stats().sequence == 3, "writer did not overlap compaction");
        std::ofstream{root / "out/cancel.csv"} << "previous completed result\n";
        auto cancel = std::make_unique<Request>(query_options(), query_bytes, "cancel.csv");
        cancel->checkpoints = {Gate::query, &canceled_query}; auto cancel_handle = cancel->handle();
        check(executor.submit(cancel) == Submit::accepted, "cancel query submit"); canceled_query.await();
        // Both paused requests hold the entire query pool. Maintenance has its own allowance.
        maintenance.release(); engine->await_idle();
        const auto installed = engine->stats();
        check(installed.compactions == 1 && installed.flushes == 3 && installed.rebased >= 1 &&
              installed.sequence == 3 && installed.logs == 1 && installed.cleanup.retained > 0,
              "installation lost new state or old retention");
        cancel_handle.cancel(); canceled_query.release();
        const auto canceled = result(cancel_handle);
        check(canceled.state == TaskState::canceled && canceled.events.initial_runs > 1,
              "live spilling query cancellation");
        check(read(root / "out/cancel.csv") == "previous completed result\n", "canceled query published");
        auto fresh = std::make_unique<Request>(query_options(), query_bytes, "new.csv");
        auto fresh_handle = fresh->handle(); check(executor.submit(fresh) == Submit::accepted, "fresh submit");
        old_query.release(); executor.finish();
        const auto a = result(old_handle), b = result(fresh_handle);
        check(a.state == TaskState::completed && b.state == TaskState::completed &&
              a.sequence == 1 && b.sequence == 3 && a.staged_groups == 2 && b.staged_groups == 2,
              "old/new query completion boundaries");
        check(read(root / "out/old.csv") ==
              "region_id,day,event_count,total_cents\n1,3,40,1240\n2,3,20,590\n", "old answer changed");
        check(read(root / "out/new.csv") ==
              "region_id,day,event_count,total_cents\n3,3,39,2135\n4,3,21,690\n", "new answer mixed snapshots");
        check(a.events.initial_runs == 20 && a.groups.initial_runs == 20 &&
              b.events.initial_runs == 20 && b.groups.initial_runs == 20 &&
              a.events.merge_passes == 4 && a.groups.merge_passes == 4 &&
              b.events.merge_passes == 4 && b.groups.merge_passes == 4, "live queries did not spill");
        engine->request_collection(); engine->await_idle();
        check(engine->stats().retained.snapshots == 0 && engine->stats().cleanup.retained == 0 &&
              engine->stats().maintenance_used == 0, "retention or workspace did not release");
        engine->finish();
        check(engine->write(later).reason == WriteReason::engine_closed, "stopped writer accepted data");
    }
    auto budget = std::make_shared<memory::Budget>(16 * 1024 * 1024);
    {
        Database db{budget, (root / "db").string(), DatabaseOpen::recover, options().storage};
        check(db.sequence() == 3 && db.orphans_at_open() == 0, "live install not recoverable or cleanup incomplete");
        auto view = db.snapshot(); check(cents(view, 101) == 900 && cents(view, 160) == 100 &&
              !view.get({Relation::events, 102}), "reopened values disagree");
    }
    check(budget->used() == 0 && fs::is_empty(root / "spill"), "fixture retained resources");
    std::cout << "live overlap: sequence 1 -> 3, old/new answers, 1 compaction, 3 flushes, canceled spill\n";
}
```

Both successful queries process sixty events, but their answers differ. Each
sort creates twenty initial runs. With fan-in two, materialized counts are
$20\to10\to5\to3\to2$: four passes, followed by the final streaming merge.
The canceled query has also created more than one run before cancellation, so
its cleanup is not an empty-input shortcut.

While the old query remains paused, collection must report retained obsolete
sources. After terminal results release the views, explicit collection must
leave zero retained snapshot states and zero maintenance workspace bytes.
Reopening then checks sequence 3 and the actual updates and tombstone, not just
the CSV files.

The CSV sink preserves `cancel.csv`'s previous completed contents. Cancellation
is not translated into a new empty result. The test also verifies that neither
successful query leaves spill files behind.

## Test revalidation without relying on a lucky schedule

Use the split job directly under a single controlling thread to exercise
three precise changes between build and install: a newer log set, a replaced
selected prefix, and a newer unselected table suffix. There is still at most
one split job alive; no collector runs while it exists. The synchronous
compaction used to make the stale case is deliberate and fully completed
before the stale job tries to install.

```cpp title="tests/engine.cpp"
void check_stale_and_rebase() {
    TemporaryDirectory root;
    auto budget = std::make_shared<memory::Budget>(16 * 1024 * 1024);
    auto workspace = std::make_shared<memory::Budget>(2 * 1024 * 1024);
    {
        Database db{budget, root.name(), DatabaseOpen::create, options().storage};
        const std::array a{Mutation::put(Event{1, 10, 3, 10})};
        const std::array b{Mutation::put(Event{1, 10, 3, 20})};
        committed(db.write(a), 1); check(db.rotate(), "split initial rotate");
        {
            MaintenanceJob flush{db, workspace, MaintenanceKind::flush}; flush.build();
            committed(db.write(b), 2); check(db.rotate(), "split newer rotation");
            check(flush.install(db) && flush.rebased() && db.sequence() == 2 && db.manifest().log_count == 2,
                  "flush replaced newer log set");
        }
        check(db.flush_oldest(), "flush second delta");
        {
            MaintenanceJob stale{db, workspace, MaintenanceKind::compact, 2, options().compaction};
            stale.build(); db.compact_prefix(2, options().compaction);
            const auto before = db.manifest().epoch;
            check(!stale.install(db) && db.manifest().epoch == before, "stale maintenance was installed");
            expect<std::logic_error>([&] { (void)stale.install(db); });
        }
        {
            const auto selected = db.manifest().table_count;
            MaintenanceJob prefix{db, workspace, MaintenanceKind::compact, selected, options().compaction};
            prefix.build();
            committed(db.write(std::array{Mutation::put(Event{2, 10, 3, 30})}), 3);
            check(db.rotate() && db.flush_oldest(), "newer suffix flush");
            check(prefix.install(db) && prefix.rebased() && db.manifest().table_count == 2,
                  "compaction discarded a newer table suffix");
        }
        auto view = db.snapshot(); check(cents(view, 1) == 20 && cents(view, 2) == 30, "rebased results");
        view.reset();
        committed(db.write(std::array{Mutation::erase({Relation::events, 1}),
                                     Mutation::erase({Relation::events, 2})}), 4);
        check(db.rotate() && db.flush_oldest(), "delete generation");
        {
            MaintenanceJob empty{db, workspace, MaintenanceKind::compact, db.manifest().table_count, options().compaction};
            empty.build(); check(empty.install(db), "empty compaction install");
            check(empty.stats().output_files == 0 && db.manifest().base_end == 4 && db.manifest().base_count == 0,
                  "empty prefix lost its coverage");
        }
        check(db.collect_obsolete().failed == 0, "split test cleanup");
    }
    check(budget->used() == 0 && workspace->used() == 0, "split plan owners escaped");
    std::cout << "split install: preserve newer logs and tables, reject stale prefix, keep empty coverage\n";
}
```

The first and third jobs must rebase successfully despite an epoch change.
The middle job must reject because its selected records are no longer current.
Its second installation attempt throws: stale is a terminal phase, not permission
to try the same candidate repeatedly. Finally, deleting both keys checks a zero-
file base with nonzero coverage through sequence 4.

## Stop while a writer cannot make progress

Pause the first flush, then create three sealed generations and one active
generation under a one-batch threshold. The fifth writer enters the actual
condition-variable wait. `await_writers(1)` confirms that state before stop is
requested.

The local `JoinWriter` guard requests engine stop before joining the external
writer. This order remains valid if an assertion or watchdog throws. Without
that request, the guard could wait forever for a slot that the paused
maintenance worker is not allowed to free.

```cpp title="tests/engine.cpp"
void check_shutdown_waiter() {
    TemporaryDirectory root; Gate flush{"flush-build"}, unused1, unused2;
    {
        auto config = options(); config.batches_per_log = 1;
        auto engine = std::make_shared<Engine>(root.name(), DatabaseOpen::create, config,
                                              InstallHook{Gate::maintenance, &flush});
        ReleaseGates release{flush, unused1, unused2};
        committed(engine->write(std::array{Mutation::put(Event{1, 10, 3, 1})}), 1);
        check(engine->seal(), "shutdown seal1"); flush.await();
        for (std::uint64_t i = 2; i <= 4; ++i) {
            committed(engine->write(std::array{Mutation::put(Event{i, 10, 3, i})}), i);
            if (i != 4) { check(engine->seal(), "shutdown fill sealed slots"); }
        }
        WriteResult answer{}; std::exception_ptr error;
        std::thread writer([&] {
            try { answer = engine->write(std::array{Mutation::put(Event{5, 10, 3, 5})}); }
            catch (...) { error = std::current_exception(); }
        });
        // The guard stops the owner before joining, even when the rendezvous assertion throws.
        struct JoinWriter {
            Engine& engine; std::thread& thread;
            ~JoinWriter() { engine.stop(); if (thread.joinable()) { thread.join(); } }
        } join{*engine, writer};
        engine->await_writers(1); check(engine->stats().logs == 4, "writer was not backpressured");
        engine->stop(); writer.join(); check(!error, "shutdown writer threw");
        check(answer.outcome == WriteOutcome::rejected && answer.reason == WriteReason::engine_closed,
              "shutdown did not wake the blocked writer with a definite rejection");
        flush.release(); engine->finish();
        check(engine->stats().abandoned == 1 && engine->stats().maintenance_used == 0,
              "stopped maintenance retained workspace");
    }
    auto budget = std::make_shared<memory::Budget>(16 * 1024 * 1024);
    Database db{budget, root.name(), DatabaseOpen::recover, options().storage};
    check(db.sequence() == 4 && db.orphans_at_open() != 0, "shutdown recovery authority");
    for (std::uint64_t i = 1; i <= 4; ++i) { check(db.get({Relation::events, i}).has_value(), "acknowledged write missing"); }
    check(!db.get({Relation::events, 5}), "rejected write was replayed");
    std::cout << "shutdown: writer at four-log cap wakes; four acknowledged batches reopen\n";
}
```

Stop abandons the not-yet-installed flush. Its files may exist as orphans; they
are not adopted on reopen. The authoritative WALs still recover all four
acknowledged batches, and event 5 must be absent. Shutdown is therefore a tested
state transition, not a blanket request to discard every buffered object.

## Make worker failure visible without revoking old readers

One failure hook throws during output construction. Another throws after
manifest rename, where authority may already have changed. Arm the second only
after the rotation needed by capture, otherwise the test would fail the wrong
operation.

The already captured view contains frozen memory. Both failure cases must leave
it readable, stop new writes, make `await_idle` and explicit `finish` report the
stored exception, and reopen to the one acknowledged sequence.

```cpp title="tests/engine.cpp"
struct FailWork {
    static void hook(void*, std::string_view stage) {
        if (stage == "maintenance-row") { throw std::runtime_error("injected maintenance failure"); }
    }
};
struct FailInstall {
    std::atomic<bool> armed{false};
    static void hook(void* pointer, std::string_view stage) {
        auto& self = *static_cast<FailInstall*>(pointer);
        if (stage == "manifest-renamed" && self.armed.load(std::memory_order_relaxed)) {
            throw std::runtime_error("injected installation uncertainty");
        }
    }
};
void check_visible_failure(bool installing) {
    TemporaryDirectory root; Gate built{"maintenance-built"}, unused1, unused2; FailInstall install;
    auto query_budget = std::make_shared<memory::Budget>(query_bytes);
    {
        auto engine = std::make_shared<Engine>(root.name(), DatabaseOpen::create, options(),
            installing ? InstallHook{Gate::maintenance, &built} : InstallHook{FailWork::hook, nullptr},
            InstallHook{FailInstall::hook, &install});
        ReleaseGates release{built, unused1, unused2};
        committed(engine->write(std::array{Mutation::put(Event{1, 10, 3, 42})}), 1);
        auto view = engine->capture(query_budget);  // Captures frozen memory, not a row copy.
        if (installing) { built.await(); install.armed.store(true, std::memory_order_relaxed); built.release(); }
        expect<std::runtime_error>([&] { engine->await_idle(); });
        const auto report = engine->stats();
        check(report.error && report.stopped && !report.busy && report.maintenance_used == 0,
              "maintenance error did not reach visible state");
        check(cents(view, 1) == 42, "maintenance failure revoked earlier snapshot");
        expect<std::runtime_error>([&] { (void)engine->write(std::array{Mutation::put(Event{2, 10, 3, 9})}); });
        expect<std::runtime_error>([&] { engine->finish(); });
        view.reset();
    }
    check(query_budget->used() == 0, "failed engine snapshot leaked workspace");
    auto budget = std::make_shared<memory::Budget>(16 * 1024 * 1024);
    Database db{budget, root.name(), DatabaseOpen::recover, options().storage};
    check(db.sequence() == 1 && db.get({Relation::events, 1}) && !db.get({Relation::events, 2}),
          "failure reopen lost acknowledgement or added rejected write");
}
```

A construction failure is not an uncertain commit, while a rename-stage failure
requires reopening authority. The engine deliberately stops after either kind
of maintenance failure rather than silently continuing with no worker. An
existing view is independent of that decision because its state has already
been captured and its source ownership is complete.

## Release the last owner on a different thread

A snapshot cap of one rejects the second capture before changing the manifest
epoch. Then destroy the engine and the caller's query-budget handle while keeping
one isolated view. The directory lock must remain held; an attempted reopen
fails until that view is released.

The final release is performed by a new thread. This exercises the registry,
arena/table releases, shared allocator control storage, and directory lock
through their actual transitive owners—not merely a standalone atomic counter.

```cpp title="tests/engine.cpp"
void check_retention_and_lifetime() {
    TemporaryDirectory root;
    auto local = std::make_shared<memory::Budget>(query_bytes);
    std::weak_ptr<memory::Budget> weak = local; ReadView escaped;
    {
        auto config = options(); config.storage.reads.snapshots = 1;
        auto engine = std::make_shared<Engine>(root.name(), DatabaseOpen::create, config);
        committed(engine->write(std::array{Mutation::put(Event{1, 10, 3, 11})}), 1);
        escaped = engine->capture(local); engine->await_idle();
        const auto before = engine->stats();
        expect<std::length_error>([&] { (void)engine->capture(local); });
        check(engine->stats().epoch == before.epoch, "retention rejection changed authority");
        engine->finish();
    }
    local.reset(); check(!weak.expired() && cents(escaped, 1) == 11, "escaped snapshot lost transitive owners");
    auto opening = std::make_shared<memory::Budget>(16 * 1024 * 1024);
    expect<std::system_error>([&] { Database blocked{opening, root.name(), DatabaseOpen::recover, options().storage}; });
    std::thread release([view = std::move(escaped)]() mutable { view.reset(); }); release.join();
    check(weak.expired(), "final control storage retained private budget");
    Database db{opening, root.name(), DatabaseOpen::recover, options().storage};
    check(db.sequence() == 1, "escaped owner did not release directory lock");
    std::cout << "retention: reject before rotation; final snapshot and allocator release on another thread\n";
}
```

The weak budget handle is only an observation. It does not keep the private
query budget alive. Its expiration after the release thread joins establishes
that the last shared-control allocation no longer owns that budget either.
The join orders the subsequent check after the release; the test never reads
ordinary shared state while that release is still running.

## Stress the ledger and reject an impossible allowance

Six threads each perform 500 independent reservation/allocation/release cycles.
At most six pairs of a 200-byte lease and a $64\cdot8=512$-byte vector coexist:
$6(200+512)=4272$ bytes, within an 8192-byte limit. A smaller observed peak is
expected when their lifetimes do not fully overlap; do not assert a particular
scheduler interleaving.

The thread-start loop also joins a partially started set if a later creation
throws. This is the same lifetime rule taught in Chapter 18, applied to test
code rather than ignored there.

```cpp title="tests/engine.cpp"
void check_shared_ledger() {
    auto budget = std::make_shared<memory::Budget>(8192);
    std::array<std::thread, 6> threads;
    std::atomic<bool> bad{false};
    // Only this fixture's independent containers are concurrent; each lease still has one owner.
    try {
        for (auto& thread : threads) {
            thread = std::thread([&] {
                try {
                    for (std::size_t i = 0; i < 500; ++i) {
                        auto reserved = budget->reserve(200);
                        memory::Vector<std::uint64_t> values(64, 7, memory::Allocator<std::uint64_t>{*budget});
                        if (values[63] != 7 || budget->used() > budget->limit()) { bad = true; }
                    }
                } catch (...) { bad = true; }
            });
        }
    } catch (...) {
        for (auto& thread : threads) { if (thread.joinable()) { thread.join(); } }
        throw;
    }
    for (auto& thread : threads) { thread.join(); }
    check(!bad && budget->used() == 0 && budget->peak() <= budget->limit(), "concurrent ledger invariant");
    std::cout << "ledger: 3000 concurrent reserve/allocate/release cycles, zero final charge\n";
}
```

A 64-byte maintenance budget cannot fit the job and its control storage. The
worker must fail visibly, release its charge, and stop the engine; it must not
borrow from query memory. A descriptor configuration of one is rejected at
configuration validation, before trying to open the already locked database.
The normal entry point runs every check. Fixture mode leaves its outputs in an
explicitly supplied disposable root for the Python oracle.

```cpp title="tests/engine.cpp"
void check_private_allowance_failure() {
    TemporaryDirectory root;
    auto config = options(); config.maintenance_bytes = 64;
    auto engine = std::make_shared<Engine>(root.name(), DatabaseOpen::create, config);
    committed(engine->write(std::array{Mutation::put(Event{1, 10, 3, 42})}), 1);
    check(engine->seal(), "low maintenance allowance seal");
    expect<std::bad_alloc>([&] { engine->await_idle(); });
    check(engine->stats().error && engine->stats().maintenance_used == 0, "maintenance allowance failure hidden");
    expect<std::bad_alloc>([&] { engine->finish(); });
    config = options(); config.maintenance_files = 1;
    expect<std::invalid_argument>([&] { Engine invalid{root.name(), DatabaseOpen::recover, config}; });
}
}  // namespace
int main(int argc, char** argv) {
    try {
        if (argc == 3 && std::string_view{argv[1]} == "--fixture") { check_overlap(argv[2]); return 0; }
        if (argc != 1) { throw std::invalid_argument("usage: strata-engine-tests [--fixture DIRECTORY]"); }
        check_shared_ledger(); check_stale_and_rebase();
        { TemporaryDirectory root; check_overlap(root.name()); }
        check_shutdown_waiter(); check_visible_failure(false); check_visible_failure(true);
        check_retention_and_lifetime(); check_private_allowance_failure();
        std::cout << "engine checks passed\n"; return 0;
    } catch (const std::exception& error) { std::cerr << "engine: " << error.what() << '\n'; return 1; }
}
```

## Recompute the answer without the C++ pipeline

Create this Python script. It uses ordinary dictionaries for a deliberately
small fixture, applies the same two atomic state changes, and groups by region
and day without calling the C++ join, sorter, or aggregate. It compares every
CSV field and checks preservation of the canceled destination and absence of
spill files.

This is a fixture oracle, not the larger workload generator introduced in the
next chapter. Its size is fixed and small, so keeping both expected answers in
Python does not undermine the C++ pipeline's bounded-memory design.

```python title="scripts/check_engine_fixture.py"
#!/usr/bin/env python3
"""Calculate old/new answers independently of C++ snapshots, sorting, and joins."""
import csv
import subprocess
import sys
import tempfile
from collections import defaultdict
from pathlib import Path


def answer(accounts: dict[int, int], events: dict[int, tuple[int, int, int]]) -> list[tuple[int, ...]]:
    totals: dict[tuple[int, int], list[int]] = defaultdict(lambda: [0, 0])
    for account, day, cents in events.values():
        if 3 <= day < 4 and account in accounts:
            group = totals[accounts[account], day]
            group[0] += 1
            group[1] += cents
    return [(region, day, *values) for (region, day), values in sorted(totals.items())]


def main() -> None:
    if len(sys.argv) != 2:
        raise SystemExit("usage: check_engine_fixture.py EXECUTABLE")
    executable = str(Path(sys.argv[1]).resolve())
    accounts = {10: 1, 20: 2}
    events = {100 + i: (20 if i % 3 == 0 else 10, 3, i + 1) for i in range(60)}
    old = answer(accounts, events)
    accounts[10] = 3
    events[101] = (10, 3, 900)
    del events[102]
    accounts[20] = 4
    events[160] = (20, 3, 100)
    new = answer(accounts, events)
    with tempfile.TemporaryDirectory(prefix="strata-live-oracle-") as directory:
        root = Path(directory)
        subprocess.run([executable, "--fixture", directory], check=True, timeout=60)
        for name, expected in (("old.csv", old), ("new.csv", new)):
            with (root / "out" / name).open(newline="") as source:
                reader = csv.reader(source)
                assert next(reader) == ["region_id", "day", "event_count", "total_cents"]
                actual = [tuple(map(int, row)) for row in reader]
            assert actual == expected, (name, actual, expected)
        assert (root / "out/cancel.csv").read_text() == "previous completed result\n"
        assert not list((root / "spill").iterdir()), "temporary runs survived terminal results"
        assert sorted(p.name for p in (root / "out").iterdir()) == ["cancel.csv", "new.csv", "old.csv"]
    print("independent live CSV oracle: sequences 1/3, two distinct answers, canceled output preserved")


if __name__ == "__main__":
    main()
```

## Connect the targets

Append these lines to the existing root `CMakeLists.txt`. `Threads::Threads`,
`Python3_EXECUTABLE`, the C++20 requirement, and the core warning flags already
come from the named prerequisites. No new dependency implements storage or
execution for us.

```cmake title="CMakeLists.txt — append Chapter 20 targets"
# Chapter 20: live storage coordination and off-lock maintenance.
target_sources(strata-core PRIVATE src/storage/maintenance.cpp src/runtime/engine.cpp)
add_executable(strata-engine-tests tests/engine.cpp)
target_link_libraries(strata-engine-tests PRIVATE strata-core)
target_compile_options(strata-engine-tests PRIVATE -Wall -Wextra -Wpedantic -Wconversion -Wshadow)
add_test(NAME live-engine COMMAND strata-engine-tests)
set_tests_properties(live-engine PROPERTIES TIMEOUT 90)
add_test(NAME live-engine-native
    COMMAND ${Python3_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_engine_fixture.py
            $<TARGET_FILE:strata-engine-tests>)
set_tests_properties(live-engine-native PROPERTIES TIMEOUT 90)
```

Run from the learner source root, using the native compiler whose feature probe
passed Chapter 1:

```sh
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug -j4
ctest --test-dir build/debug --output-on-failure

cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/release -j4
ctest --test-dir build/release --output-on-failure
```

The cumulative suite now contains **31 tests**, including `live-engine` and
`live-engine-native`. A successful direct `strata-engine-tests` invocation prints
these lines; intervening checks also exercise the two maintenance failures and
insufficient-allowance rejection:

```text
ledger: 3000 concurrent reserve/allocate/release cycles, zero final charge
split install: preserve newer logs and tables, reject stale prefix, keep empty coverage
live overlap: sequence 1 -> 3, old/new answers, 1 compaction, 3 flushes, canceled spill
shutdown: writer at four-log cap wakes; four acknowledged batches reopen
retention: reject before rotation; final snapshot and allocator release on another thread
engine checks passed
```

The Python test ends with:

```text
independent live CSV oracle: sequences 1/3, two distinct answers, canceled output preserved
```

Use separate build trees for memory/undefined-behavior checking and race
checking. Do not combine AddressSanitizer and ThreadSanitizer in one binary.
The compiler/runtime must support the selected sanitizer on the local machine.

```sh
cmake -S . -B build/asan -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build/asan -j4
ctest --test-dir build/asan --output-on-failure

cmake -S . -B build/tsan -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=thread -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=thread"
cmake --build build/tsan --target strata-engine-tests strata-executor-tests -j4
ctest --test-dir build/tsan -R 'live-engine|bounded-executor' --output-on-failure
```

If a sanitizer runtime cannot start, record that limitation separately from a
passing program. The ordinary checks must still run. Native Apple Clang/libc++
and M4 execution need their own results; a Linux run does not substitute for
that platform check. The code requires the already probed C++20 facilities,
not a newly mandated compiler banner or invented operating-system version.

## Diagnose a race-free installation mistake

Temporarily change the stale test inside `MaintenanceJob::install` from

```cpp title="Diagnostic — original revalidation condition"
if (!same) { phase_ = Phase::stale; return false; }
```

to

```cpp title="Diagnostic — temporary overstrict epoch condition"
if (!same || current.epoch != captured_.epoch) {
    phase_ = Phase::stale;
    return false;
}
```

Rebuild and run the direct test. It must fail with
`engine: flush replaced newer log set`. The selected sealed log has not changed;
only an unrelated newer generation was added. The mutant refuses valid work
and breaks the installation contract without a memory error or data race.

Restore the original condition before continuing. Then inspect the distinct
stale-prefix test: removing input-identity checks would be the opposite bug,
accepting a candidate whose proof no longer applies. Do not “fix” either test
by dropping the assertion or by copying the captured manifest back over the
current one.

## Read failures as ownership and progress evidence

| Symptom | First boundary to inspect |
|---|---|
| Writer cannot commit while `compact-build` is paused | Engine mutex accidentally spans `job->build`; gate is not outside the critical section |
| Fifth writer never wakes after stop | Stop does not notify `changed`, or the wait predicate ignores stopping/error |
| New CSV uses old regions or old amounts | Account/event scans did not share one captured sequence, or installation discarded newer dependencies |
| ThreadSanitizer reports a cache/counter race | A private reader/view was shared between workers; do not merely make its owner count atomic |
| Final query charge or storage retention remains nonzero | Terminal task retained a plan/view, or final allocator/control storage still owns its budget |
| Snapshot limit rejection changes epoch | Admission was moved after rotation, violating the no-side-effects pressure check |
| Maintenance loops forever with 16 tables | A base-only repack cannot free a slot; report capacity failure instead of retrying unchanged work |
| Reopen fails after a maintenance error | Distinguish valid orphan files from a missing/corrupt authoritative dependency; preserve Chapter 11's strict policy |

`collect_obsolete` remains bounded by the directory inventory limit, but its
metadata operations run under the engine mutex. The file system can block that
mutex for a substantial time. Measurements in later chapters should distinguish
this from bulk compaction, which runs unlocked; a single “maintenance time”
number would hide the tradeoff.

## Verified boundary and remaining limits

The cumulative 31-test suite passes Debug, Release, and
AddressSanitizer/UndefinedBehaviorSanitizer builds with Clang 17 on Linux. The
four focused executor/live-engine tests pass a separate GCC 14.2 ThreadSanitizer
build. The overstrict-epoch mutation fails its logical assertion in all four
configurations without a sanitizer report, and the correct implementation is
restored before the final passing runs. The Linux race-check build used
`-fno-pie`/`-no-pie` for that runtime; these are not prescribed native macOS flags.

These tests establish the demonstrated code paths, not freedom from every
possible interleaving. Native M4/macOS execution and Apple full-device
synchronization remain unverified. Ordinary reopen after joined shutdown is not
a power-loss experiment. Budget counters are not RSS; descriptor allowances
are not kernel reservations; long-lived snapshots deliberately retain resources.

The engine now coordinates real concurrent queries, ingestion, and maintenance
with explicit lifetime, admission, failure, and publication boundaries. It still
has one serialized writer, one maintenance worker, fixed table/log limits, and
per-query reader/cache isolation. Do not replace those explicit bounds with an
unbounded queue of frozen generations to hide pressure.

## References used for this chapter

The implementation above is self-contained with its named prerequisites. These
primary references clarify the C++ rules behind its synchronization choices:

- [C++ working draft: mutex requirements](https://eel.is/c++draft/thread.mutex.requirements),
  particularly lock/unlock synchronization and noncopyable mutex lifetime.
- [C++ working draft: shared ownership pointers](https://eel.is/c++draft/util.smartptr.shared),
  particularly final-owner destruction and the distinction between pointer
  machinery and accesses to the pointed-to object.
- [C++ working draft: condition variables](https://eel.is/c++draft/thread.condition.condvar),
  particularly atomic unlock-and-wait, re-locking, predicate waits, and destruction.
- [C++ working draft: data races](https://eel.is/c++draft/intro.races),
  particularly conflicting lifetime operations, happens-before, and relaxed atomics.

These draft pages can include facilities beyond C++20; this chapter uses only
its already established C++20 baseline. It does not require newer signatures
shown elsewhere on those pages.

**Next:** Chapter 21 — Reproducible workloads and independent answers. It will
turn the small exact fixtures into controlled end-to-end workloads without
changing what a commit, snapshot, cancellation, or completed answer means.

