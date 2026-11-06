# Chapter 19 — Bounded execution and cooperative cancellation

## Part 3 — Force the waiting and publication boundaries

The scheduler's contract is not "the test eventually prints two answers." We
need to observe a worker waiting for resources, cancel a request behind the
blocked queue head, show that the caller still owns a rejected request, and
cancel on both sides of the result-publication gate.

This completes the same **4.0-hour chapter** begun in
[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-19.html) and continued
in [Part 2](https://25d-228.github.io/cplusplus-learning/chapter-19-part-2.html).
Use the exact prerequisite amendments before building. All test helpers below
are included; no unseen test framework or scheduling service is required.

## A small answer with genuinely spilling execution

The fixture has accounts 10 and 20 in regions 1 and 2. Event index $i$ runs from
0 through 59. Its account is 20 when $i$ is divisible by 3, otherwise 10; its day
is 3 and its cents value is $i+1$. Region 2 therefore receives the 20 values
$1,4,7,\ldots,58$:

$$
\text{region 2 total}=20(1+58)/2=590.
$$

All 60 events total $60\cdot61/2=1830$ cents. Region 1 has the remaining 40 events
and $1830-590=1240$ cents. The expected CSV is thus independently calculable,
not extracted from a previous C++ query run.

Each sort admits only three input rows per initial run. Sixty rows require
$\lceil60/3\rceil=20$ initial runs. With merge fan-in two, materialized passes
reduce that count through $20\to10\to5\to3\to2$. The final two-way merge streams
into its consumer, so the expected materialized-pass count is four for **each**
sort. Small row and block buffers force the real spill/cleanup paths while
keeping the fixture inexpensive.

Create `tests/executor.cpp`. Begin with imports, assertions, the temporary owner,
and ordinary file helpers:

```cpp title="tests/executor.cpp"
#include "strata/runtime/executor.hpp"

#include <array>
#include <cerrno>
#include <chrono>
#include <condition_variable>
#include <cstdlib>
#include <filesystem>
#include <fstream>
#include <iostream>
#include <iterator>
#include <mutex>
#include <stdexcept>
#include <string>
#include <system_error>
#include <vector>

namespace {
using namespace strata;
using namespace strata::storage;
using namespace strata::query;
using namespace strata::runtime;
namespace fs = std::filesystem;
constexpr std::size_t allowance = 2 * 1024 * 1024;
const std::string expected_csv = "region_id,day,event_count,total_cents\n1,3,40,1240\n2,3,20,590\n";
void check(bool value, const char* message) { if (!value) { throw std::runtime_error(message); } }
template<class Error, class F> void expect(F action) {
    try { action(); } catch (const Error&) { return; }
    throw std::runtime_error("missing expected exception");
}
class TemporaryDirectory {
public:
    TemporaryDirectory() {
        char pattern[] = "/tmp/strata-executor-XXXXXX";
        const auto p = ::mkdtemp(pattern);
        if (!p) { throw std::system_error(errno, std::generic_category(), "test directory"); }
        path_ = p;
    }
    ~TemporaryDirectory() noexcept { std::error_code ignored; fs::remove_all(path_, ignored); }
    const std::string& name() const { return path_; }
private: std::string path_;
};
std::string read(const fs::path& path) {
    std::ifstream input{path};
    check(static_cast<bool>(input), "missing result file");
    return {std::istreambuf_iterator<char>{input}, {}};
}
void old_result(const fs::path& path) { std::ofstream{path} << "previous completed result\n"; }
```

`TemporaryDirectory` owns only the test root. Its destructor uses the
error-code overload of removal so a diagnostic exception cannot cause another
exception during unwinding. The runtime's own cleanup is asserted *before* this
fallback removes the fixture. It must not hide a missing runtime unlink.

Configure tiny operator buffers and create a disk-only database before any
worker exists:

```cpp title="tests/executor.cpp"
DatabaseOptions database_options() {
    DatabaseOptions options; options.arena_bytes = 256 * 1024;
    options.cache_slots = 2; options.cache_bytes = 256;
    options.table = {128, 256, 32, 64 * 1024}; return options;
}
QueryOptions query_options() {
    QueryOptions options; options.days = {3, 4}; options.upstream_rows = 2;
    options.join = {2, 2, 4}; options.joined_rows = 2;
    options.aggregate = {2, 4}; options.output_rows = 1;
    SortOptions sort; sort.chunk_rows = 3; sort.block_rows = 2;
    sort.file_descriptors = 3; sort.workspace_bytes = 128 * 1024;
    sort.stack_allowance = 16 * 1024; sort.disk_bytes = 128 * 1024;
    options.event_sort = sort; options.group_sort = sort; return options;
}
std::shared_ptr<const FrozenInputs> fixture(const fs::path& root) {
    fs::create_directories(root / "db"); fs::create_directories(root / "spill");
    fs::create_directories(root / "out");
    auto budget = std::make_shared<memory::Budget>(16 * 1024 * 1024);
    {
        Database db{budget, (root / "db").string(), DatabaseOpen::create, database_options()};
        std::vector<Mutation> rows{Mutation::put(Account{10, 1}), Mutation::put(Account{20, 2})};
        for (std::uint64_t i = 0; i < 60; ++i) {
            rows.push_back(Mutation::put(Event{100 + i, i % 3 == 0 ? 20U : 10U, 3, i + 1}));
        }
        check(db.write(rows).outcome == WriteOutcome::committed, "fixture commit");
        check(db.rotate() && db.flush_oldest(), "disk-only fixture preparation");
    }
    check(budget->used() == 0, "fixture setup leaked charges");
    return std::make_shared<FrozenInputs>((root / "db").string(), database_options());
}
```

`rotate` and `flush_oldest` complete outside the executor. The ordinary setup
budget must return to zero after the local database dies. Only then does
`FrozenInputs` recover and retain the immutable sources. Reopening here also
exercises the existing manifest authority, table decoding, and directory lock
instead of substituting a vector of fake query rows.

The result helper checks normal teardown before returning the copied report.
The separate deletion-refusal test will intentionally bypass its clean-spill
expectation. A fifteen-second completion wait is a diagnostic watchdog, not the
means of coordinating threads.

```cpp title="tests/executor.cpp"
TaskResult result(const TaskHandle& handle) {
    const auto answer = handle.wait_for(std::chrono::seconds{15});
    check(answer.has_value(), "task completion watchdog");
    check(answer->final_bytes == 0, "query allowance returned before all owners died");
    check(answer->events.live_files == 0 && answer->groups.live_files == 0 &&
          !answer->events.directory_live && !answer->groups.directory_live &&
          answer->result_cleanup_error == 0, "query cleanup incomplete");
    return *answer;
}
std::unique_ptr<Request> request(const std::string& name, std::size_t bytes = allowance) {
    return std::make_unique<Request>(query_options(), bytes, name);
}
```

Use a gate to hold workers immediately after admission. The controller waits
until the exact number of arrivals has been observed under a mutex. No sleep is
used to guess that a worker has started. Hook and gate lifetimes extend until
the executor has joined; the timeout path reports a broken test rather than
leaving a permanently blocked worker.

```cpp title="tests/executor.cpp"
class Gate {
public:
    static void hook(void* self, Boundary point) {
        if (point != Boundary::admitted) { return; }
        auto& gate = *static_cast<Gate*>(self);
        std::unique_lock lock{gate.mutex_}; ++gate.arrived_; gate.changed_.notify_all();
        if (!gate.changed_.wait_for(lock, std::chrono::seconds{10}, [&] { return gate.open_; })) {
            throw std::runtime_error("worker gate watchdog");
        }
    }
    void await(std::size_t count) {
        std::unique_lock lock{mutex_};
        if (!changed_.wait_for(lock, std::chrono::seconds{5}, [&] { return arrived_ >= count; })) {
            throw std::runtime_error("arrival watchdog");
        }
    }
    void release() { { std::lock_guard lock{mutex_}; open_ = true; } changed_.notify_all(); }
private:
    std::mutex mutex_; std::condition_variable changed_; std::size_t arrived_ = 0; bool open_ = false;
};
```

The gate callback runs on a worker and borrows `Gate`. Its counters and flag use
one mutex. `release` publishes the open flag before notification. Although two
arrivals are required, the scheduler does not depend on this gate at all; it is
only a way to observe a stable intermediate state.

## Prove overlapping admission and ownership-preserving rejection

Append the two-query check:

```cpp title="tests/executor.cpp"
void check_overlap(const fs::path& root, std::shared_ptr<const FrozenInputs> inputs) {
    Gate gate;
    std::atomic<std::size_t> live{0};
    Executor executor{inputs, (root / "spill").string(), (root / "out").string(),
                      {2, 4, 2 * allowance, 20}, std::nullopt, &live};
    auto a = request("a.csv"), b = request("b.csv");
    a->checkpoints = {Gate::hook, &gate}; b->checkpoints = {Gate::hook, &gate};
    auto ah = a->handle(), bh = b->handle();
    check(executor.submit(a) == Submit::accepted && !a, "first acceptance ownership");
    check(executor.submit(b) == Submit::accepted && !b, "second acceptance ownership");
    gate.await(2);
    const auto busy = executor.stats();
    check(busy.running == 2 && busy.used_bytes == 2 * allowance && busy.used_files == 20,
          "queries did not overlap with complete reservations");
    auto duplicate = request("a.csv"); auto* address = duplicate.get();
    check(executor.submit(duplicate) == Submit::output_busy && duplicate.get() == address,
          "conflicting output accepted or ownership lost");
    gate.release(); executor.finish();
    const auto ar = result(ah), br = result(bh);
    check(ar.state == TaskState::completed && br.state == TaskState::completed &&
          ar.staged_groups == 2 && br.staged_groups == 2 && ar.sequence == 1 && br.sequence == 1,
          "overlapping query completion");
    check(read(root / "out/a.csv") == expected_csv && read(root / "out/b.csv") == expected_csv,
          "concurrent answers differ");
    check(ar.events.initial_runs == 20 && ar.groups.initial_runs == 20 &&
          ar.events.merge_passes == 4 && ar.groups.merge_passes == 4, "spilling paths not exercised");
    check(executor.stats().used_bytes == 0 && executor.stats().used_files == 0 && live == 0,
          "joined executor retained resources");
    auto late = request("late.csv");
    check(executor.submit(late) == Submit::closed && late, "closed submit lost ownership");
    std::cout << "overlap: 2 queries, 20/20 runs, 4/4 passes, 2 groups each\n";
}
```

At the gate, both requests are running and their allowances sum to 4 MiB and 20
file descriptors. A duplicate `a.csv` descriptor remains at its original address
after `output_busy`. Once released, both queries must complete with sequence 1,
two groups, and the exact CSV. Each must exercise both spilling sorts. `finish`
then returns only after the diagnostic live-thread count is zero.

The CSV names differ, so the two workers do not race to replace one destination.
The input names are shared but immutable; the readers/caches and output owners
are distinct. The final rejected submission also proves that closing does not
silently consume late work.

## Cancel behind a head that cannot fit

Append the independent memory-pressure and descriptor-pressure check:

```cpp title="tests/executor.cpp"
void check_wait_and_reject(const fs::path& root, std::shared_ptr<const FrozenInputs> inputs, bool files) {
    Gate gate;
    Executor executor{inputs, (root / "spill").string(), (root / "out").string(),
        {2, 2, files ? 2 * allowance : allowance, files ? 10U : 20U}};
    auto impossible = request("impossible.csv", 3 * allowance); auto* original = impossible.get();
    check(executor.submit(impossible) == Submit::impossible && impossible.get() == original,
          "impossible admission took ownership");
    expect<std::logic_error>([&] { (void)impossible->handle().wait(); });
    auto a = request("held.csv"); a->checkpoints = {Gate::hook, &gate}; auto ah = a->handle();
    check(executor.submit(a) == Submit::accepted, "held submit"); gate.await(1);
    auto b = request("blocked.csv"), c = request("behind.csv"); auto bh = b->handle(), ch = c->handle();
    check(executor.submit(b) == Submit::accepted && executor.submit(c) == Submit::accepted, "queued admission");
    executor.await_waiters(1);
    check(bh.state() == TaskState::queued && ch.state() == TaskState::queued && executor.stats().running == 1,
          "resource waiter acquired a snapshot or started");
    auto excess = request("full.csv"); auto* kept = excess.get();
    check(executor.submit(excess) == Submit::full && excess.get() == kept, "full rejection ownership");
    ch.cancel(); check(result(ch).state == TaskState::canceled, "canceled tail remained behind blocked head");
    bh.cancel(); check(result(bh).state == TaskState::canceled, "resource cancellation failed to wake worker");
    check(!fs::exists(root / "out/blocked.csv") && !fs::exists(root / "out/behind.csv"), "queued task opened output");
    gate.release(); executor.finish(); check(result(ah).state == TaskState::completed, "held task did not drain");
}
```

Call this twice. With `files == false`, memory permits one running query; with
`files == true`, descriptor capacity permits one. In both cases a second worker
is observed waiting and B/C remain `queued`. Cancel the tail C first. If the
scheduler checks only the head's resource predicate, C's wait times out even
though it needs no resources to become canceled. Cancel B next, then release A.

The test also covers impossible and full submissions while preserving the exact
caller-owned pointer. Checking that no canceled queued destination exists helps
catch an implementation that captures a snapshot or starts a sink too early.
The invariant still comes from the code path: queued cancellation never calls
`execute`.

## Stop at a boundary, then stop too late

Two borrowed hook contexts let the worker itself request cancellation at a
precise point. `CancelAt` uses the ordinary checkpoint path; `LateCancel` runs
inside the sink after the final checkpoint has already passed.

```cpp title="tests/executor.cpp"
struct CancelAt {
    TaskHandle handle;
    Boundary target;
    std::size_t remaining = 1;
    static void hook(void* self, Boundary point) {
        auto& state = *static_cast<CancelAt*>(self);
        if (point == state.target && state.remaining != 0 && --state.remaining == 0) { state.handle.cancel(); }
    }
};
struct LateCancel {
    TaskHandle handle;
    static void hook(void* self, ResultEvent point) {
        if (point == ResultEvent::before_publish) { static_cast<LateCancel*>(self)->handle.cancel(); }
    }
};
void check_cancellation(const fs::path& root, std::shared_ptr<const FrozenInputs> inputs) {
    Executor executor{inputs, (root / "spill").string(), (root / "out").string(), {2, 4, 2 * allowance, 20}};
    for (auto boundary : {Boundary::admitted, Boundary::batch, Boundary::block,
                          Boundary::spill_pass, Boundary::before_publish}) {
        old_result(root / "out/cancel.csv");
        auto work = request("cancel.csv"); auto handle = work->handle();
        CancelAt cancel{handle, boundary, boundary == Boundary::block ? 30U : 1U};
        work->checkpoints = {CancelAt::hook, &cancel};
        check(executor.submit(work) == Submit::accepted, "cancel request submit");
        const auto answer = result(handle);
        check(answer.state == TaskState::canceled && !answer.error, "cancellation classified as failure or success");
        check(read(root / "out/cancel.csv") == "previous completed result\n" && fs::is_empty(root / "spill"),
              "cancellation published partial output or stranded runs");
        if (boundary == Boundary::spill_pass) { check(answer.events.files_created >= 20, "canceled before runs existed"); }
        if (boundary == Boundary::before_publish) { check(answer.staged_groups == 2, "no staged output at final gate"); }
    }
    auto work = request("late-cancel.csv"); auto handle = work->handle(); LateCancel late{handle};
    work->result_hook = {LateCancel::hook, &late};
    check(executor.submit(work) == Submit::accepted, "late cancel submit");
    check(result(handle).state == TaskState::completed && read(root / "out/late-cancel.csv") == expected_csv,
          "late cancellation overrode successful publication");
    executor.finish(); std::cout << "cancellation: admission, batch, block, pass, publication gate; late commit wins\n";
}
```

For the block case, the thirtieth block checkpoint avoids limiting the test to
the first metadata read. The spill-pass case requires at least twenty created
runs, so cancellation really owns temporary files to unwind. At the final gate,
both groups have been staged but the old destination must remain unchanged.

The late hook sets the same stop flag immediately before rename, inside the
masked finish scope. The result must be `completed` and the new file must be
complete. This deterministic interleaving exposes a logical race in outcome
classification even when every individual memory access is properly synchronized.

A cancellation that races with a different exception has no universal precedence
in this design. The path actually observed decides: a `Canceled` exception gives
canceled, a different exception gives failed, and successful publication gives
completed. We do not relabel outcomes afterward by sampling the flag.

## Failure is local to one query

Append a sink error and the first twenty-four charged-allocation failures:

```cpp title="tests/executor.cpp"
void fail_write(void*, ResultEvent event) {
    if (event == ResultEvent::before_write) { throw std::runtime_error("injected sink write failure"); }
}
void check_errors(const fs::path& root, std::shared_ptr<const FrozenInputs> inputs) {
    Executor executor{inputs, (root / "spill").string(), (root / "out").string(), {2, 4, allowance, 20}};
    old_result(root / "out/fail.csv");
    auto work = request("fail.csv"); auto handle = work->handle(); work->result_hook = {fail_write, nullptr};
    check(executor.submit(work) == Submit::accepted, "failure submit");
    const auto failed = result(handle);
    check(failed.state == TaskState::failed && failed.error, "missing transported exception");
    expect<std::runtime_error>([&] { std::rethrow_exception(failed.error); });
    check(read(root / "out/fail.csv") == "previous completed result\n", "failure replaced completed result");
    std::size_t failures = 0;
    for (std::size_t ordinal = 1; ordinal <= 24; ++ordinal) {
        auto candidate = request("allocation.csv"); candidate->fail_allocation_on = ordinal;
        auto h = candidate->handle(); check(executor.submit(candidate) == Submit::accepted, "allocation submit");
        const auto r = result(h);
        if (r.state == TaskState::failed) { ++failures; expect<std::bad_alloc>([&] { std::rethrow_exception(r.error); }); }
        else { check(r.state == TaskState::completed, "unexpected allocation result"); }
    }
    check(failures == 24 && fs::is_empty(root / "spill"), "early allocation campaign or cleanup");
    auto tiny = request("tiny.csv", 512); auto tiny_handle = tiny->handle();
    check(executor.submit(tiny) == Submit::accepted, "small capped query admission");
    const auto rejected = result(tiny_handle);
    check(rejected.state == TaskState::failed, "insufficient workspace did not fail locally");
    expect<std::bad_alloc>([&] { std::rethrow_exception(rejected.error); });
    auto retry = request("retry.csv"); auto retry_handle = retry->handle();
    check(executor.submit(retry) == Submit::accepted, "post-failure retry submit");
    check(result(retry_handle).state == TaskState::completed, "failed task poisoned executor or held allowance");
    executor.finish(); std::cout << "failure: 24 allocation positions, budget cap, sink error; next query succeeds\n";
}
```

The captured sink exception is rethrown on the controller and checked by type.
The old completed CSV survives. The allocation loop is a targeted early-failure
campaign, not a claim that every allocation site in every plan was exhaustively
failed. All twenty-four selected ordinals must produce `bad_alloc`, return their
working ownership, and leave no run directory.

A 512-byte query fits the scheduler's pool claim but cannot construct the private
read context and plan. It is accepted, then fails locally. The later normal query
must succeed. If it remains queued, inspect reservation return; if it allocates
through a dead ledger, inspect escaped read/statistics ownership. Neither problem
is fixed by catching and discarding the original exception.

## A cleanup report must not pretend the disk is empty

Append the deliberate unlink refusal:

```cpp title="tests/executor.cpp"
void refuse_unlink(void*, SpillEvent event, const char*, std::uint64_t, std::uint64_t) {
    if (event == SpillEvent::before_unlink) { throw std::runtime_error("injected removal refusal"); }
}
void check_cleanup_report(const fs::path& root, std::shared_ptr<const FrozenInputs> inputs) {
    Executor executor{inputs, (root / "spill").string(), (root / "out").string(), {1, 2, allowance, 20}};
    auto work = request("cleanup.csv"); auto handle = work->handle();
    work->query_hooks.events = {refuse_unlink, nullptr};
    CancelAt cancel{handle, Boundary::spill_pass, 1}; work->checkpoints = {CancelAt::hook, &cancel};
    check(executor.submit(work) == Submit::accepted, "cleanup report submit");
    const auto report = handle.wait(); executor.finish();
    check(report.state == TaskState::canceled && report.final_bytes == 0 &&
          report.events.cleanup_errors != 0 && report.events.live_files >= 20 &&
          report.events.disk_reserved != 0 && report.events.directory_live &&
          executor.stats().used_bytes == 0 && executor.stats().used_files == 0,
          "unresolved cleanup was hidden or confused with memory/descriptor release");
    check(fs::is_directory(report.events.directory.data()), "missing observable spill orphan");
    // Fixture cleanup, after terminal completion: production code would explicitly retry this known directory.
    fs::remove_all(report.events.directory.data());
    check(fs::is_empty(root / "spill"), "fixture orphan cleanup");
    auto unaccepted = request("pre-canceled.csv"); auto h = unaccepted->handle(); h.cancel();
    // The completed executor rejects ownership even for an already-canceled descriptor.
    check(executor.submit(unaccepted) == Submit::closed && unaccepted && h.state() == TaskState::created,
          "rejected request acquired terminal status");
    Executor second{inputs, (root / "spill").string(), (root / "out").string(), {1, 2, allowance, 20}};
    check(second.submit(unaccepted) == Submit::accepted, "pre-canceled acceptance");
    check(result(h).state == TaskState::canceled && !fs::exists(root / "out/pre-canceled.csv"),
          "pre-canceled descriptor ran");
    second.finish();
    expect<std::invalid_argument>([] { (void)request("Alias.csv"); });
    std::cout << "cleanup refusal: orphan path and retained disk charge remain observable\n";
}
```

Cancellation at the first spill pass now encounters an unlink hook that always
fails. The result has zero charged working bytes and the executor has returned
its memory/descriptor allowances, yet `live_files`, `disk_reserved`, cleanup
errors, and the directory path remain observable. Those facts are compatible:
closing a file descriptor does not remove its name or reclaim its disk contents.

Only this test explicitly removes its known fixture directory after terminal
completion. Production cleanup should operate on the reported, controlled spill
namespace and surface failure; it must not recursively delete arbitrary user
paths. No automatic retry service is implemented here. A result-sink cleanup
error is separately reported by `result_cleanup_error`.

The remainder of the check proves that cancellation before submission persists
across a rejected `closed` return and a later accepted submission. Rejection
must not invent a terminal owner. Uppercase names are rejected to preserve the
executor's bounded lowercase output namespace.

## Unwind the controller and partially started constructors

Append shutdown tests and the complete `main`:

```cpp title="tests/executor.cpp"
void check_shutdown(const fs::path& root, std::shared_ptr<const FrozenInputs> inputs) {
    std::atomic<std::size_t> live{0};
    for (std::size_t fail = 0; fail < 3; ++fail) {
        expect<std::runtime_error>([&] {
            Executor executor{inputs, (root / "spill").string(), (root / "out").string(),
                              {3, 4, allowance, 20}, fail, &live};
        });
        check(live == 0, "partially started workers escaped construction failure");
    }
    std::vector<TaskHandle> handles;
    try {
        Executor executor{inputs, (root / "spill").string(), (root / "out").string(),
                          {2, 16, 2 * allowance, 20}, std::nullopt, &live};
        for (std::size_t i = 0; i < 10; ++i) {
            auto work = request("shutdown-" + std::to_string(i) + ".csv");
            handles.push_back(work->handle()); check(executor.submit(work) == Submit::accepted, "shutdown submit");
        }
        throw std::runtime_error("caller unwinds");
    } catch (const std::runtime_error& error) { check(std::string_view{error.what()} == "caller unwinds", "unexpected shutdown error"); }
    for (const auto& handle : handles) {
        const auto r = result(handle);
        check(r.state == TaskState::completed || r.state == TaskState::canceled, "shutdown failed to resolve a task");
    }
    check(live == 0 && fs::is_empty(root / "spill"), "shutdown did not join or clean spill owners");
}
}  // namespace
int main(int argc, char** argv) {
    try {
        if (argc == 3 && std::string_view{argv[1]} == "--fixture") {
            const fs::path root{argv[2]}; auto inputs = fixture(root); check_overlap(root, inputs); return 0;
        }
        if (argc != 1) { throw std::invalid_argument("usage: strata-executor-tests [--fixture ROOT]"); }
        TemporaryDirectory directory; const fs::path root{directory.name()}; auto inputs = fixture(root);
        check_overlap(root, inputs); check_wait_and_reject(root, inputs, false); check_wait_and_reject(root, inputs, true);
        check_cancellation(root, inputs); check_errors(root, inputs); check_cleanup_report(root, inputs); check_shutdown(root, inputs);
        std::cout << "bounded execution, cancellation, isolation, and joined shutdown: all checks passed\n";
        return 0;
    } catch (const std::exception& error) { std::cerr << error.what() << '\n'; return 1; }
}
```

The first loop fails before launching worker 0, 1, or 2. Every already-started
thread must be joined before the constructor exception reaches the caller.
The second case accepts ten requests, then deliberately throws from the
controller's scope. Some requests may finish before cancellation; others may
be canceled. Both outcomes are legal. An unresolved handle, escaped worker,
failed query, or remaining spill directory is not.

`TaskHandle` values remain outside the executor's scope. Their completion data
are still readable because they own only their controls, not a raw executor
pointer. Cancellation after executor teardown is safe as well: the weak wakeup
may be empty, while the task's completed result remains unchanged.

The executable's `--fixture ROOT` mode runs only the successful overlap check
and leaves its two completed CSV files for an independent process to examine.
It does not leave spill runs or a running thread.

## Check actual files without the C++ query algorithms

Create `scripts/check_executor_fixture.py`:

```python title="scripts/check_executor_fixture.py"
#!/usr/bin/env python3
"""Check real concurrent CSV answers without using the C++ query implementation."""
import csv
import subprocess
import sys
import tempfile
from collections import defaultdict
from pathlib import Path


def main() -> None:
    if len(sys.argv) != 2:
        raise SystemExit("usage: check_executor_fixture.py EXECUTABLE")
    executable = str(Path(sys.argv[1]).resolve())
    expected = defaultdict(lambda: [0, 0])
    for index in range(60):
        region = 2 if index % 3 == 0 else 1
        totals = expected[region, 3]
        totals[0] += 1
        totals[1] += index + 1
    rows = [(region, day, *totals) for (region, day), totals in sorted(expected.items())]
    with tempfile.TemporaryDirectory(prefix="strata-executor-oracle-") as directory:
        root = Path(directory)
        subprocess.run([executable, "--fixture", str(root)], check=True, timeout=60)
        for name in ("a.csv", "b.csv"):
            with (root / "out" / name).open(newline="") as source:
                reader = csv.reader(source)
                assert next(reader) == ["region_id", "day", "event_count", "total_cents"]
                actual = [tuple(map(int, row)) for row in reader]
            assert actual == rows, (name, actual, rows)
        assert not list((root / "spill").iterdir()), "temporary runs survived completion"
        assert sorted(path.name for path in (root / "out").iterdir()) == ["a.csv", "b.csv"]
    print("independent concurrent CSV oracle: 60 events, 2 groups, 2 identical completed files")


if __name__ == "__main__":
    main()
```

The Python oracle calculates counts and sums directly from the sixty generated
indices. It neither invokes a C++ aggregation helper nor trusts the C++ expected
CSV string. It compares both real files, including the header and row order,
then checks that the spill parent is empty and no temporary output survived.
The surrounding temporary directory makes repeated runs independent.

## Build the integrated program, not only the scheduler

Chapter 18 already supplies `find_package(Threads REQUIRED)`, and the engine
build already locates the Python interpreter. Append the following to the root
`CMakeLists.txt`. `PUBLIC Threads::Threads` propagates the runtime's threading
link requirement to executables using the core library.

```cmake title="CMakeLists.txt"
# Chapter 19: bounded query execution against an immutable fixture.
target_sources(strata-core PRIVATE src/runtime/task.cpp src/runtime/frozen_inputs.cpp src/runtime/executor.cpp)
target_link_libraries(strata-core PUBLIC Threads::Threads)
add_executable(strata-executor-tests tests/executor.cpp)
target_link_libraries(strata-executor-tests PRIVATE strata-core)
add_test(NAME bounded-executor COMMAND strata-executor-tests)
set_tests_properties(bounded-executor PROPERTIES TIMEOUT 90)
add_test(NAME bounded-executor-native
    COMMAND ${Python3_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_executor_fixture.py
            $<TARGET_FILE:strata-executor-tests>)
set_tests_properties(bounded-executor-native PROPERTIES TIMEOUT 90)
```

Run from the learner repository root. Reuse compatible native compiler versions;
C++20 plus the actual prerequisite feature checks remain the requirements.
Do not copy Linux compiler paths into a macOS build.

```sh title="Terminal — Debug and Release"
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug --parallel 4
ctest --test-dir build/debug --output-on-failure

cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/release --parallel 4
ctest --test-dir build/release --output-on-failure

./build/debug/strata-executor-tests
python3 scripts/check_executor_fixture.py ./build/debug/strata-executor-tests
```

The normal runtime executable prints these five diagnostic lines when its checks
pass; timings and thread scheduling order are not part of the expected output:

```text
overlap: 2 queries, 20/20 runs, 4/4 passes, 2 groups each
cancellation: admission, batch, block, pass, publication gate; late commit wins
failure: 24 allocation positions, budget cap, sink error; next query succeeds
cleanup refusal: orphan path and retained disk charge remain observable
bounded execution, cancellation, isolation, and joined shutdown: all checks passed
```

The independent oracle additionally prints `independent concurrent CSV oracle: 60 events, 2 groups, 2 identical
completed files` after the subprocess's overlap line.

Run memory/undefined-behavior instrumentation separately from race detection:

```sh title="Terminal — address and undefined behavior instrumentation"
cmake -S . -B build/asan -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build/asan --parallel 4
ctest --test-dir build/asan --output-on-failure
```

Use a separate directory and a compiler/runtime that actually supports
ThreadSanitizer on the host:

```sh title="Terminal — race instrumentation"
cmake -S . -B build/tsan -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=thread -fno-omit-frame-pointer -O1" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=thread"
cmake --build build/tsan --target strata-executor-tests --parallel 4
ctest --test-dir build/tsan -R '^bounded-executor(-native)?$' --output-on-failure
```

The official [ThreadSanitizer documentation](https://clang.llvm.org/docs/ThreadSanitizer.html)
explains compile/link instrumentation and host support. It detects races in
executed paths, not every possible schedule or logical error. A toolchain that
cannot launch its sanitizer runtime has not completed this check; record the
limitation rather than substituting an ordinary run. Sanitizer overhead is not
query-engine resource usage, and race instrumentation is not a performance run.

## Break the outcome rule without introducing a memory race

In `Executor::execute`, temporarily replace the single successful assignment
following `consume_results` with this intentionally wrong code:

```cpp title="Diagnostic only — wrong post-publication classification"
result.state = request.control_->stop.load(std::memory_order_relaxed)
    ? TaskState::canceled : TaskState::completed;
```

Rebuild `strata-executor-tests` and run it. The late-cancellation fixture must
fail with `late cancellation overrode successful publication`. The stop read is
atomic and all owners can still be released correctly; nevertheless the reported
outcome contradicts the already replaced file. Sanitizers cannot replace this
state-machine assertion. Restore `result.state = TaskState::completed;`, rebuild,
and rerun the normal checks before proceeding.

As another reasoning exercise, move snapshot capture above scheduler admission
on paper and count the retained owners when the queue fills. Even if only two
workers execute, queued snapshots could pin old generations and allocate caches
before any query allowance exists. The requirement is not merely a worker count;
it is the precise point where ownership is admitted.

## Diagnosis follows the boundary that failed

| Symptom | First boundary to inspect |
|---|---|
| Canceled tail times out behind a large head | Cancellation scan before resource admission; cancel mutex handshake |
| Idle workers repeatedly wake without requests | Test-observation notifications incorrectly using the worker condition variable |
| `final_bytes` is nonzero | Escaped stats/read view/cache owners or a charged allocation that lacks cleanup |
| Two queries trigger a race in cache or budget | A shared parent reader/context was served instead of a private isolated state |
| Published file exists but handle says canceled | Stop flag sampled after the publication gate |
| Query fails and the next task never starts | Allowances not returned on the failed path |
| Memory is returned but spill directory remains | Explicit cleanup error/path report; do not confuse close with unlink |
| Destructor blocks indefinitely | A blocked native call, unreleased test gate, or a forbidden descendant wait |

A watchdog failure deserves a trace of states and owners, not a larger sleep.
Start with `queued`, `running`, `waiting`, and the two reservation totals. Check
which mutex protects each observation. Then inspect the query's last boundary
and cleanup report. The executor does not offer forced termination of a running
thread, and destroying its memory while it is blocked would not be a repair.

## Executed checks and limits

The complete cumulative learner program passed **29 CTest entries** in Debug,
Release, and AddressSanitizer/UndefinedBehaviorSanitizer builds with Clang 17 on
Linux. A separate GCC 14.2 ThreadSanitizer build passed the focused runtime and
native CSV checks. The deliberately wrong post-publication classification failed
the intended logical assertion in all four configurations without a sanitizer
memory/race report; the correct assignment was restored before the final runs.

The checks include independent memory/descriptor waits, queued-tail cancellation,
five running cancellation boundaries, successful publication after a late stop,
twenty-four allocation positions, insufficient local workspace, sink failure,
observable cleanup refusal, pre-submission cancellation, partial thread startup,
and controller unwinding. They exercise real two-sort plans and native table,
spill, and CSV files. They are not an exhaustive schedule exploration.

**Native M4/macOS execution remains unverified.** The immutable fixture is not
concurrent ingestion. Cooperative cancellation is not interruption of a blocked
filesystem call. Charged allocations are not RSS, descriptor allowances are not
kernel reservations, per-sort disk caps are not a global disk quota, and completed
CSV publication is not power-loss durability.

The next chapter adds coordinated live reads, one writer, and background
maintenance. Keep this chapter's boundaries: admit before capturing, do not share
mutable read bookkeeping accidentally, return owners before terminal completion,
and join before destroying shared state.

