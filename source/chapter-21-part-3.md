# Chapter 21 — Reproducible workloads and independent answers

## Part 3 — Force old, new, canceled, and reopened answers

Sequential commands can validate reopening, but they cannot demonstrate a query
retaining its old sources while ingestion and maintenance change current
authority. Two writer processes are not the answer: the directory lock excludes
that arrangement. The scenario below uses one `Engine`, two query workers, and
the dedicated maintenance worker in one process.

**Prerequisite code:** Chapter 20 and Parts
[1](https://25d-228.github.io/cplusplus-learning/chapter-21.html) and
[2](https://25d-228.github.io/cplusplus-learning/chapter-21-part-2.html) of this
chapter. Keep the `tiny` configuration and the default 120-event input for the
controlled scenario. Smaller or differently filtered input may not reach a
spill-pass checkpoint; the scenario then fails rather than claiming to have
exercised cancellation during spilling.

## Give the controller an observable boundary

There are two gates. The old query pauses after capture, keeping its original
snapshot alive. The canceled query pauses at a spill-pass boundary, after it has
created runs. A gate owns its mutex, condition variable, and two booleans.
`arrived_` announces that the worker reached the boundary; `open_` permits it to
continue. Both predicates are observed under the same mutex. Notify calls alone
are not the state.

The callback's `void*` context borrows a `Gate`. We keep both gate objects outside
the executor's scope so the borrowed pointer outlives every worker. The first
matching checkpoint waits; after release, future matching checkpoints pass
immediately because `open_` remains true. Thirty-second waits are watchdogs for
an invalid fixture or stuck implementation, not the mechanism establishing
ordering.

```cpp title="apps/scenario.cpp"
#include "strata/workload/driver.hpp"

#include <chrono>
#include <condition_variable>
#include <fstream>
#include <iostream>
#include <mutex>
#include <stdexcept>

namespace strata::workload {
namespace {
using namespace runtime;
void require(bool good, const char* message) {
    if (!good) { throw std::runtime_error(message); }
}
class Gate {
public:
    explicit Gate(Boundary boundary) : boundary_(boundary) {}
    static void hook(void* context, Boundary boundary) {
        auto& gate = *static_cast<Gate*>(context);
        if (boundary != gate.boundary_) { return; }
        std::unique_lock lock{gate.mutex_};
        gate.arrived_ = true; gate.changed_.notify_all();
        if (!gate.changed_.wait_for(lock, std::chrono::seconds{30}, [&] { return gate.open_; })) {
            throw std::runtime_error("scenario release watchdog");
        }
    }
    void await() {
        std::unique_lock lock{mutex_};
        if (!changed_.wait_for(lock, std::chrono::seconds{30}, [&] { return arrived_; })) {
            throw std::runtime_error("scenario arrival watchdog; check selected rows/preset");
        }
    }
    void release() noexcept {
        { std::lock_guard lock{mutex_}; open_ = true; }
        changed_.notify_all();
    }
private:
    Boundary boundary_;
    std::mutex mutex_; std::condition_variable changed_;
    bool arrived_ = false, open_ = false;
};
```

An exception in the controller must not strand a worker behind a gate while
the executor destructor waits to join it. `Release` is declared **after** the
executor, so reverse destruction opens both gates before destroying the
executor. This is a small application of the lifetime discipline from Chapters
3 and 18. The gates themselves are declared earlier and remain alive during
both operations.

`submit` does not keep a second request owner. It takes the handle, asks the
executor to accept the request, and returns the handle. On rejection the local
unique owner destroys the unsubmitted request before the helper throws.

```cpp title="apps/scenario.cpp"
struct Release {
    Gate& old; Gate& cancel;
    ~Release() { old.release(); cancel.release(); }
};
TaskHandle submit(Executor& executor, const Preset& p, const char* name, Gate* gate = nullptr) {
    auto request = std::make_unique<Request>(p.query, p.query_bytes, name);
    if (gate) { request->checkpoints = {Gate::hook, gate}; }
    auto handle = request->handle();
    require(executor.submit(request) == Submit::accepted, "scenario request rejected");
    return handle;
}
}  // namespace
```

The scenario uses a new root containing `db`, `spill`, and `out`. It checks that
base and delta seeds agree and sets the snapshot-state limit to two. After base
import and maintenance, the old query reaches `captured`. Only then does the
controller apply the delta and request compaction. With the defaults, these
boundaries are sequences 5 and 6.

The old query consumes one whole-query reservation while paused. The canceled
query uses the other; neither leaves room for another query reservation. The
maintenance worker uses its separate allowance. This is why resource ownership
rather than just worker count matters to the scenario.

```cpp title="apps/scenario.cpp"
void scenario(const std::filesystem::path& root, const std::string& base,
              const std::string& delta, query::DayRange days) {
    namespace fs = std::filesystem;
    auto p = preset("tiny", days); p.engine.storage.reads.snapshots = 2;
    const auto parent = root.has_parent_path() ? root.parent_path() : fs::path{"."};
    preflight(parent, p.space_floor);
    require(fs::create_directory(root), "scenario root must be new");
    for (const auto* child : {"db", "spill", "out"}) { fs::create_directory(root / child); }
    Input initial{base}, changes{delta};
    require(initial.seed() == changes.seed(), "scenario seed mismatch");
    std::uint64_t latest = 0;
    Gate old_gate{Boundary::captured}, cancel_gate{Boundary::spill_pass};
    {
        auto engine = std::make_shared<Engine>((root / "db").string(), storage::DatabaseOpen::create, p.engine);
        apply(*engine, initial); settled(*engine, false);
        const auto original = engine->stats().sequence;
        Executor executor{engine, (root / "spill").string(), (root / "out").string(),
                          {2, 4, 2 * p.query_bytes, 58}};
        Release release{old_gate, cancel_gate};  // Opens gates BEFORE executor destruction on every exit.
        const auto old = submit(executor, p, "old.csv", &old_gate); old_gate.await();
        apply(*engine, changes); settled(*engine, true);
        latest = engine->stats().sequence;
        require(latest > original && engine->stats().compactions != 0, "scenario needs a nonempty delta and compaction");
        {
            std::ofstream previous;
            previous.exceptions(std::ios::badbit | std::ios::failbit);
            previous.open(root / "out/cancel.csv"); previous << "previous completed result\n"; previous.close();
        }
```

The sentinel file represents a previous completed result. Its write and close
must themselves be checked; otherwise a later cancellation could appear to
preserve a file that was never successfully created.

When the canceled query is paused in a spill pass, two snapshot states are
retained. A direct third capture must fail with the retention-limit error,
without installing a different manifest epoch. This isolates snapshot retention
from the executor queue's own admission. Release and cancellation are separate:
requesting stop does not magically release this test gate, so the controller
must open it as well.

```cpp title="apps/scenario.cpp"
        const auto cancel = submit(executor, p, "cancel.csv", &cancel_gate); cancel_gate.await();
        const auto epoch = engine->stats().epoch;
        bool pressure = false;
        try { (void)engine->capture(std::make_shared<memory::Budget>(p.query_bytes)); }
        catch (const std::length_error&) { pressure = true; }
        require(pressure && engine->stats().epoch == epoch, "retention pressure changed authority");
        cancel.cancel(); cancel_gate.release();
        const auto canceled = wait_result(cancel);
        require(canceled.state == TaskState::canceled && canceled.events.initial_runs > 1 &&
                canceled.final_bytes == 0 && canceled.events.live_files == 0 &&
                canceled.groups.live_files == 0 && !canceled.events.directory_live &&
                !canceled.groups.directory_live && canceled.result_cleanup_error == 0,
                "canceled query retained owners or did not spill");
        {
            std::ifstream previous{root / "out/cancel.csv"}; std::string line;
            require(static_cast<bool>(std::getline(previous, line)) && line == "previous completed result" &&
                    previous.peek() == std::char_traits<char>::eof(), "cancellation replaced previous output");
        }
```

After cancellation frees its reservation, submit the fresh query. Releasing the
old query allows both successful executions to finish. Both answers are checked
for their sequence boundaries, both sorts must have at least two materialized
passes, and terminal cleanup must have released the owners. The controller
then collects obsolete names, stops the engine, and destroys its owners before
opening a new `Database` object on the same directory.

```cpp title="apps/scenario.cpp"
        const auto fresh = submit(executor, p, "new.csv");
        old_gate.release(); executor.finish();
        const auto before = wait_result(old), after = wait_result(fresh);
        require_completed(before); require_completed(after);
        require(before.sequence == original && after.sequence == latest, "mixed snapshot boundary");
        require(before.events.merge_passes >= 2 && before.groups.merge_passes >= 2 &&
                after.events.merge_passes >= 2 && after.groups.merge_passes >= 2,
                "scenario input did not force two materialized passes in both sorts");
        print_result("old", before); print_result("new", after); print_result("canceled", canceled);
        engine->request_collection(); engine->await_idle(); print_engine(engine->stats());
        require(engine->stats().retained.snapshots == 0 && fs::is_empty(root / "spill"), "scenario cleanup incomplete");
        engine->finish();
    }
    auto budget = std::make_shared<memory::Budget>(p.engine.storage_bytes);
    {
        storage::Database reopened{budget, (root / "db").string(), storage::DatabaseOpen::recover, p.engine.storage};
        require(reopened.sequence() == latest, "reopen lost the acknowledged boundary");
    }
    require(budget->used() == 0, "reopen owner leak");
    std::cout << "scenario seed=" << initial.seed() << " reopened_sequence=" << latest
              << " retention_rejected=1 canceled_output_preserved=1\n";
}
}  // namespace strata::workload
```

The scenario checks sequence and resource invariants in C++. The Python test
below independently computes the entire old and new answers and performs a
further query after reopen. That division matters: an internally consistent
but wrong C++ answer must still fail the test.

| Point in the default run | Retained states | Required observation |
|---|---:|---|
| Old query paused after capture | 1 | Its boundary is sequence 5 |
| Writer applies delta and compacts | 1 | Current boundary becomes 6; old state survives |
| Canceled query pauses in spilling | 2 | Third capture rejects without changing epoch |
| Canceled task becomes terminal | 1 | Previous output survives; spill owners are gone |
| Old and fresh queries finish | 0 | Old answer is sequence 5; fresh answer is 6 |
| Engine owners die and database reopens | 0 | Recovered sequence is 6 |

This fixture establishes ownership overlap, not a throughput comparison. It
uses no sleeps, no second writer process, and no claim that every possible
thread schedule was explored.

## A handwritten oracle before a generated one

Create the test script. Three literal inputs supply answers simple enough to
calculate without the generator. The first batch has two matches. The second
changes account 10's region and event 1's amount while deleting event 2. The
third deletes account 10 and inserts another referencing event: the inner join
must now be empty. A zero region is not a tombstone.

The subprocess helper captures both output streams and includes the command,
arguments, and status when a stage fails. Its timeout is a harness watchdog.
The child that tests a low descriptor limit changes only its own inherited
limits, not the controller's. This use of `preexec_fn` is confined to the
single-threaded Python harness; it is not a pattern for the threaded engine.

```python title="scripts/check_workload.py"
#!/usr/bin/env python3
"""Small correctness checks, not performance or power-loss experiments."""
import copy
import csv
import json
from pathlib import Path
import re
import resource
import subprocess
import sys
import tempfile

import generate_data as generator
import reference_query as reference

PARAMETERS = dict(seed=7, events=120, accounts=17, days=8, regions=4,
                  hot_percent=70, batch=32, update_every=7, delete_every=13)
BASE = "STRATA1 9 0\nB 1 4\nA 10 1\nA 20 2\nE 1 10 3 7\nE 2 20 3 9\nC 1\nEND 1 4 88\n"
DELTA = "STRATA1 9 1\nB 2 3\nA 10 4\nE 1 10 3 11\nX E 2\nC 2\nEND 2 3 53\n"
ERASE = "STRATA1 9 2\nB 3 2\nX A 10\nE 3 10 3 5\nC 3\nEND 3 2 39\n"


def check(condition, message):
    if not condition:
        raise RuntimeError(message)


def reject(action):
    try:
        action()
    except (ValueError, OverflowError, UnicodeError, csv.Error):
        return
    raise RuntimeError("expected reference rejection")


def command(exe, *args, ok=True, preexec=None):
    result = subprocess.run([str(exe), *map(str, args)], text=True, capture_output=True,
                            timeout=120, preexec_fn=preexec)
    check((result.returncode == 0) == ok,
          f"stage={args[0]} args={args[1:]} exit={result.returncode}\n{result.stdout}\n{result.stderr}")
    return result.stdout + result.stderr
```

The reference test then covers 18 seed/size combinations: three seeds,
including the maximum unsigned 64-bit value, and six sizes spanning empty,
partial-period, exact-period, and multiple-period cases. Forced-hot settings are
0, 70, and 100; batch sizes are 1, 32, and 256. Each case compares replay with
the independently computed formula for both phases. Repeated generation checks
actual bytes, not merely equal row counts.

Seven malformed answer files also check the CSV comparator itself, including
wrong width, missing/extra rows, noncanonical numbers, a changed value, and an
unterminated quoted field. The trillion-event check changes only metadata and evaluates the formula. It
checks the multiplicity derivation against one full period without writing a
trillion-event input or building a trillion-entry dictionary.

```python title="scripts/check_workload.py"
def reference_checks(root):
    base, delta, erase = [root / name for name in ("base.ops", "delta.ops", "erase.ops")]
    for path, text in ((base, BASE), (delta, DELTA), (erase, ERASE)):
        path.write_text(text, encoding="ascii")
    check(reference.replay([base, delta, erase], 0, 3, 4) == [], "sequence zero")
    check(reference.replay([base, delta, erase], 1, 3, 4) == [(1, 3, 1, 7), (2, 3, 1, 9)], "manual old answer")
    check(reference.replay([base, delta, erase], 2, 3, 4) == [(4, 3, 1, 11)], "manual updated answer")
    check(reference.replay([base, delta, erase], 3, 3, 4) == [], "deleted dimension must remove matches")
    reject(lambda: reference.replay([base], 2, 3, 4))
    reject(lambda: reference.replay([base], 1, 3, 4, max_keys=1))
    reject(lambda: reference.add_group({(1, 3): (1, reference.U64)}, 1, 3, 1, 1))
    cases = 0
    for seed_index, seed in enumerate((0, 7, reference.U64)):
        for size in (0, 1, 999, 1000, 1001, 2007):
            p = dict(PARAMETERS, seed=seed, events=size, accounts=23,
                     hot_percent=(0, 70, 100)[seed_index], batch=(1, 32, 256)[seed_index])
            path = root / f"case-{cases}"
            meta = generator.generate(path, p)
            check(meta["base"]["logical_bytes"] == 30 * size + 14 * 23, "logical bytes")
            for phase in ("base", "delta"):
                inputs = [path / "base.ops"] + ([path / "delta.ops"] if phase == "delta" else [])
                expected = reference.replay(inputs, meta[phase]["last_sequence"], 1, 5)
                check(expected == reference.formula(meta, phase, 1, 5), f"formula seed={seed} N={size} phase={phase}")
            cases += 1
    a, b = root / "repeat-a", root / "repeat-b"
    meta = generator.generate(a, PARAMETERS)
    generator.generate(b, PARAMETERS)
    check(all((a / name).read_bytes() == (b / name).read_bytes()
              for name in ("base.ops", "delta.ops", "meta.json")), "generator not reproducible")
    huge = copy.deepcopy(meta); huge["parameters"]["events"] = 10**12
    unit = copy.deepcopy(meta); unit["parameters"]["events"] = 1000
    for phase in ("base", "delta"):
        check(reference.formula(huge, phase, 0, 8) ==
              [(r, d, c * 10**9, s * 10**9) for r, d, c, s in reference.formula(unit, phase, 0, 8)],
              "constant-period answer disagrees with whole-cycle multiplicity")
    header = "region_id,day,event_count,total_cents\n"
    malformed = ["wrong\n", header, header + "1,3,1,7,0\n",
                 header + "1,3,1,7\n1,3,1,7\n", header + "01,3,1,7\n",
                 header + "1,3,1,8\n", header + '\"unterminated\n']
    actual = root / "bad-answer.csv"
    for text in malformed:
        actual.write_text(text, encoding="ascii")
        reject(lambda: reference.compare(actual, [(1, 3, 1, 7)]))
    print(f"reference: {cases} seed/size cases, both phases; manual snapshots; trillion-event formula; seven bad CSVs")
```

The next test makes input rejection a shared external contract without sharing
the parser. It supplies 22 invalid forms to both C++ and Python. A truncated
terminator, an oversized count, a duplicate key, an out-of-range unsigned field,
and a malformed `END` must not become a plausible different workload.

The preflight tests request more free space than the filesystem reports and
launch a child with a soft descriptor limit of 128. Python's
[resource documentation](https://docs.python.org/3/library/resource.html#resource.setrlimit)
describes the soft/hard tuple and its Unix availability. The harness lowers
both limits in the child, then exits; it never tries to raise the parent limit.

```python title="scripts/check_workload.py"
def parser_checks(exe, root):
    valid = root / "valid.ops"; valid.write_text(BASE)
    command(exe, "validate", valid)
    bad_texts = [BASE.replace("STRATA1", "STRATA2"), BASE.replace("B 1 4", "B 1 0"),
                 BASE.replace("B 1 4", "B 1 257"), BASE.replace("B 1 4", "B 2 4"),
                 BASE.replace("C 1", "C 2"), BASE.replace("END 1 4 88", "END 1 4 87"),
                 BASE.replace("A 10 1", "A 10 -1"), BASE.replace("A 10 1", "A 10 +1"),
                 BASE.replace("A 10 1", "A 10 01"), BASE.replace("A 10 1", "A 10 4294967296"),
                 BASE.replace("E 1 10 3 7", "E 1 10 4294967296 7"),
                 BASE.replace("E 1 10 3 7", f"E 1 10 3 {1 << 64}"),
                 BASE.replace("A 20 2", "A 10 2"), BASE.replace("E 1", "Q 1"),
                 BASE.replace("A 10 1", "A  10 1"), BASE.replace("A 10 1", "A\t10 1"),
                 BASE.replace("A 10 1", "A " + "1" * 256 + " 1"), BASE.replace("\n", "\r\n"),
                 BASE[:-1], BASE.split("END")[0], BASE + "extra\n", BASE.replace("A 10 1", " A 10 1")]
    path = root / "bad.ops"
    for index, text in enumerate(bad_texts):
        path.write_bytes(text.encode("ascii"))
        command(exe, "validate", path, ok=False)
        reject(lambda: list(reference.batches(path)))
    command(exe, "preflight", root, reference.U64, ok=False)
    low = lambda: resource.setrlimit(resource.RLIMIT_NOFILE, (128, 128))
    command(exe, "preflight", root, 0, ok=False, preexec=low)
    print(f"parser: {len(bad_texts)} rejected forms; disk/descriptor preflight failures")


def query(exe, root, db, name, begin=3, end=4, preset="tiny", ok=True):
    for child in ("out", "spill"):
        (root / child).mkdir(exist_ok=True)
    return command(exe, "query", db, root / "spill", root / "out", name, begin, end, preset, ok=ok)
```

The native test first exercises the handwritten history, a repeated-apply
rejection, and a malformed second batch. Notice that it reopens the failed
import and queries the committed prefix. Merely seeing a nonzero importer exit
would not establish what survived.

```python title="scripts/check_workload.py"
def native_checks(exe, root):
    parser_checks(exe, root)
    paths = []
    for name, text in (("base", BASE), ("delta", DELTA), ("erase", ERASE)):
        path = root / (name + ".ops"); path.write_text(text); paths.append(path)
    db = root / "hand-db"
    command(exe, "load", db, paths[0], "tiny")
    query(exe, root, db, "hand.csv")
    reference.compare(root / "out/hand.csv", [(1, 3, 1, 7), (2, 3, 1, 9)])
    command(exe, "apply", db, paths[1], "tiny")
    query(exe, root, db, "hand.csv")
    reference.compare(root / "out/hand.csv", [(4, 3, 1, 11)])
    command(exe, "apply", db, paths[2], "tiny")
    command(exe, "compact", db, "tiny")
    query(exe, root, db, "hand.csv")
    reference.compare(root / "out/hand.csv", [])
    refused = command(exe, "apply", db, paths[2], "tiny", ok=False)
    check("base-sequence mismatch" in refused and "last_acknowledged=3" in refused, "repeat diagnosis")
    late = root / "late.ops"
    late.write_text(BASE.split("END")[0] + "B 2 1\nE 1 10 3 99\nC 99\n")
    failure = command(exe, "load", root / "partial-db", late, "tiny", ok=False)
    check("seed=9" in failure and "last_acknowledged=1" in failure, "partial import diagnosis")
    check("inspect sequence=1 " in command(exe, "inspect", root / "partial-db", "tiny"), "partial batch committed")
    query(exe, root, root / "partial-db", "prefix.csv")
    reference.compare(root / "out/prefix.csv", [(1, 3, 1, 7), (2, 3, 1, 9)])
```

The next two databases separate valid maximum integers from invalid arithmetic.
One event has the maximum ID, account ID, amount, day, and region. A half-open
interval ending at $2^{32}$ must include that day. Another database has two
matching events with cents $2^{64}-1$ and 1. Import is valid; their aggregate is
not. Both the engine and Python must reject the query, and the previous completed
CSV must remain unchanged.

```python title="scripts/check_workload.py"
    upper = root / "upper.ops"
    upper.write_text(f"STRATA1 0 0\nB 1 2\nA {reference.U64} 4294967295\n"
                     f"E {reference.U64} {reference.U64} 4294967295 {reference.U64}\nC 1\nEND 1 2 44\n")
    command(exe, "load", root / "upper-db", upper, "tiny")
    query(exe, root, root / "upper-db", "upper.csv", (1 << 32) - 1, 1 << 32)
    reference.compare(root / "out/upper.csv", [((1 << 32) - 1, (1 << 32) - 1, 1, reference.U64)])
    overflow = root / "overflow.ops"
    overflow.write_text(f"STRATA1 0 0\nB 1 3\nA 1 1\nE 1 1 3 {reference.U64}\nE 2 1 3 1\nC 1\nEND 1 3 74\n")
    command(exe, "load", root / "overflow-db", overflow, "tiny")
    saved = root / "out/overflow.csv"; saved.write_text("previous\n")
    query(exe, root, root / "overflow-db", saved.name, ok=False)
    check(saved.read_text() == "previous\n", "overflow published an incomplete answer")
    reject(lambda: reference.replay([overflow], 1, 3, 4))
```

The generated native fixture then checks both complete phases through the
actual driver and compares them with both oracle modes. It requires multiple
materialized passes in both sorts and confirms an empty spill parent after
completion. These assertions would catch a preset that accidentally made every
query in-memory.

```python title="scripts/check_workload.py"
    data = root / "generated"; meta = generator.generate(data, PARAMETERS)
    db = root / "generated-db"; command(exe, "load", db, data / "base.ops", "tiny")
    for phase in ("base", "delta"):
        if phase == "delta":
            command(exe, "apply", db, data / "delta.ops", "tiny")
            command(exe, "compact", db, "tiny")
        report = query(exe, root, db, phase + ".csv", 1, 5)
        paths = [data / "base.ops"] + ([data / "delta.ops"] if phase == "delta" else [])
        expected = reference.replay(paths, meta[phase]["last_sequence"], 1, 5)
        reference.compare(root / "out" / (phase + ".csv"), expected)
        check(expected == reference.formula(meta, phase, 1, 5), "generated formula/native disagreement")
        check(all(int(re.search(key + r"=(\d+)", report)[1]) >= 2
                  for key in ("event_passes", "group_passes")), "tiny preset did not force passes")
    check(not list((root / "spill").iterdir()), "spill residue")
    print("native: exact updates/deletes, partial-import prefix, u64 limits, overflow preservation, two-sort oracles")
```

Finally, run the in-process scenario and compare every emitted group. The
additional reopened query is a new process invoking the regular driver, after
the scenario has released the directory lock. The canceled output is checked
byte-for-byte against its sentinel. None of these checks derives expected totals
from C++ counters.

```python title="scripts/check_workload.py"
def scenario_checks(exe, root):
    data = root / "inputs"; meta = generator.generate(data, PARAMETERS)
    run = root / "scenario"
    report = command(exe, "scenario", run, data / "base.ops", data / "delta.ops", 1, 5)
    print(report, end="")
    for name, phase in (("old", "base"), ("new", "delta")):
        inputs = [data / "base.ops"] + ([data / "delta.ops"] if phase == "delta" else [])
        reference.compare(run / "out" / (name + ".csv"),
                          reference.replay(inputs, meta[phase]["last_sequence"], 1, 5))
    query(exe, run, run / "db", "reopened.csv", 1, 5)
    reference.compare(run / "out/reopened.csv", reference.formula(meta, "delta", 1, 5))
    check((run / "out/cancel.csv").read_text() == "previous completed result\n", "cancel replaced result")
    check("retention_rejected=1" in report and not list((run / "spill").iterdir()), "pressure/cleanup not exercised")
    print("scenario oracle: old/new/reopened answers verified independently")


def main():
    if len(sys.argv) != 3 or sys.argv[2] not in ("reference", "native", "scenario"):
        raise SystemExit("usage: check_workload.py STRATA reference|native|scenario")
    executable = Path(sys.argv[1]).resolve()
    with tempfile.TemporaryDirectory(prefix="strata-workload-") as folder:
        root = Path(folder)
        if sys.argv[2] == "reference":
            reference_checks(root)
        elif sys.argv[2] == "native":
            native_checks(executable, root)
        else:
            scenario_checks(executable, root)


if __name__ == "__main__":
    main()
```

## Connect the driver to the existing build

Append this block to the learner root `CMakeLists.txt`. `strata-workload` links
the existing core rather than copying it. `PUBLIC` propagates the core's include
path and thread dependency to the driver. The tests use the already configured
Python interpreter. No new package manager or third-party runtime is needed.

```cmake title="CMakeLists.txt — append"
# Chapter 21: input adapter and thin workload driver; core mechanisms stay unchanged.
add_library(strata-workload STATIC src/workload/input.cpp src/workload/driver.cpp)
target_link_libraries(strata-workload PUBLIC strata-core)
target_compile_options(strata-workload PRIVATE -Wall -Wextra -Wpedantic -Wconversion -Wshadow)
add_executable(strata apps/strata.cpp apps/scenario.cpp)
target_link_libraries(strata PRIVATE strata-workload)
target_compile_options(strata PRIVATE -Wall -Wextra -Wpedantic -Wconversion -Wshadow)
add_test(NAME workload-reference COMMAND ${Python3_EXECUTABLE}
    ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_workload.py $<TARGET_FILE:strata> reference)
add_test(NAME workload-native COMMAND ${Python3_EXECUTABLE}
    ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_workload.py $<TARGET_FILE:strata> native)
add_test(NAME workload-scenario COMMAND ${Python3_EXECUTABLE}
    ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_workload.py $<TARGET_FILE:strata> scenario)
set_tests_properties(workload-reference workload-native workload-scenario PROPERTIES TIMEOUT 180)
```

```sh title="Terminal — build and check the complete source state"
cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build-debug -j 4
ctest --test-dir build-debug --output-on-failure
cmake -S . -B build-release -DCMAKE_BUILD_TYPE=Release
cmake --build build-release -j 4
ctest --test-dir build-release --output-on-failure
```

The cumulative engine suite now has 34 entries. The three added names are
`workload-reference`, `workload-native`, and `workload-scenario`. A focused rerun
is `ctest --test-dir build-debug -R '^workload-' --output-on-failure`; retain the
full run before declaring the chapter integrated.

Use a separate sanitizer directory so instrumented and uninstrumented objects
are not mixed. The compiler must support the selected sanitizer for the target;
there is no required compiler-version label beyond the features actually used.

```sh title="Terminal — memory and undefined-behavior diagnostics"
cmake -S . -B build-asan -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build-asan -j 4
ctest --test-dir build-asan --output-on-failure
```

ThreadSanitizer is a separate build, not an additional flag on the address-
sanitized build. Use Chapter 18's supported compiler/runtime procedure and run
`workload-scenario`. Do not treat an unsupported runtime or startup failure as
a successful race check. Also keep the logical oracle: absence of a race report
says nothing about whether an account tombstone changed the answer correctly.

## Run the controlled scenario directly

```sh title="Terminal — old, canceled, fresh, then reopened"
run=$(mktemp -d "${TMPDIR:-/tmp}/strata-scenario.XXXXXX")
python3 scripts/generate_data.py "$run/input"
build-debug/strata scenario "$run/demo" \
  "$run/input/base.ops" "$run/input/delta.ops" 1 5
python3 scripts/reference_query.py replay \
  "$run/input/base.ops" "$run/input/delta.ops" \
  --sequence 5 --begin 1 --end 5 --actual "$run/demo/out/old.csv"
python3 scripts/reference_query.py replay \
  "$run/input/base.ops" "$run/input/delta.ops" \
  --sequence 6 --begin 1 --end 5 --actual "$run/demo/out/new.csv"
```

The meaningful deterministic fields are:

```text
old:      sequence=5 groups=8  event_input=60 join_rows=60
          event_runs=20 group_runs=20 event_passes=4 group_passes=4
new:      sequence=6 groups=12 event_input=55 join_rows=55
          event_runs=19 group_runs=19 event_passes=4 group_passes=4
canceled: sequence=6; nonzero initial runs; final charged bytes=0
scenario: reopened_sequence=6 retention_rejected=1 canceled_output_preserved=1
```

These are selected fields, not an exact copy of the complete console line.
Maintenance counts, epochs, memory peaks, available disk, and paths can differ.
The old and fresh query results must both release their charged storage; the
completed CSVs may remain because they are the intended output.

## Scale past the workspace, not past the implemented catalog

The larger check uses 200000 events and 257 accounts, with 256 operations per
batch and the stream preset. Its base logical input is

$$
200000(30)+257(14)=6003598\text{ bytes}.
$$

This exceeds each sort's 2 MiB workspace. Selecting all eight days yields
$\lceil200000/2048\rceil=98$ initial event runs, below the 128-run cap.
With fan-in eight, the two materialized passes are
$98\rightarrow13\rightarrow2$. The group sort also spills. This demonstrates
bounded processing of data larger than a workspace; it is not a claim that the
input exceeds total machine RAM or that the query allocation peak equals RSS.

```sh title="Terminal — larger generated input with analytical answers"
run=$(mktemp -d "${TMPDIR:-/tmp}/strata-stream.XXXXXX")
python3 scripts/generate_data.py "$run/input" \
  --events 200000 --accounts 257 --batch 256
mkdir "$run/spill" "$run/out"
build-release/strata load "$run/db" "$run/input/base.ops" stream
build-release/strata query "$run/db" "$run/spill" "$run/out" base.csv 0 8 stream
python3 scripts/reference_query.py formula "$run/input/meta.json" \
  --phase base --begin 0 --end 8 --actual "$run/out/base.csv"
build-release/strata apply "$run/db" "$run/input/delta.ops" stream
build-release/strata compact "$run/db" stream
build-release/strata query "$run/db" "$run/spill" "$run/out" delta.csv 0 8 stream
python3 scripts/reference_query.py formula "$run/input/meta.json" \
  --phase delta --begin 0 --end 8 --actual "$run/out/delta.csv"
build-release/strata inspect "$run/db" stream
```

The base ends at sequence 783. Its query consumes 200000 events, joins 200000,
and emits 32 groups. The delta has 41870 operations and 931465 logical bytes,
ending at sequence 947. Its query consumes 184600 surviving events, joins
180200 after account deletions, and still emits 32 groups. The latter sorts
start with 91 event runs and 88 joined runs; both make two materialized passes.
A constant number of groups does not imply unchanged values—the oracle compares
every count and cents total.

In the Linux Release check, the base sorts wrote 16863048 and 19263048 spill
bytes; the updated sorts wrote 15564648 and 17355948. These are each sort's
reported write traffic, including materialized runs, not database disk size.
The corresponding query ledger peaks were 784870 and 767390 bytes in that
build, with zero final charge. Those allocation-size observations are not
portable thresholds or performance claims. No timing or M4 memory measurement
is inferred from them.

## Make a wrong answer fail without relying on a sanitizer

Temporarily change the tombstone branch in `Input::operation`: for an account
tombstone, return `Mutation::put(Account{id, 0})` instead of `Mutation::erase`.
Keep event tombstones unchanged. Build and run only `workload-native`.
The handwritten third phase should be empty, but the defect retains account 10
in region zero and emits an extra answer row. The Python comparator must fail.
This is a semantic defect with valid memory accesses; a sanitizer need not
report it. Restore the original branch and rerun the full suites.

The normal Chapter 21 source and the cumulative 34-test suite were checked in
Linux Debug, Release, and AddressSanitizer/UndefinedBehaviorSanitizer builds
with Clang 17. The temporary account-delete defect failed the expected logical
comparison in all three configurations and was restored before the final
passing runs. The focused `workload-scenario` test also passed a separate GCC
14.2 ThreadSanitizer build; this was not a full ThreadSanitizer suite. The larger
200000-event run was checked in Release against both
phase formulas. These are correctness and resource-owner checks, not native
macOS execution or physical power-loss tests.

## Diagnose the stage, not just the last error line

| Symptom | Likely boundary to inspect | Useful next check |
|---|---|---|
| `base-sequence mismatch` | A file was repeated, applied to the wrong history, or a prior import stopped after commits | Inspect recovered sequence and reported `last_acknowledged`; do not automatically replay |
| Import fails after a valid first batch | The file, not the batch, was assumed atomic | Query the reopened committed prefix; inspect the rejected batch's terminator |
| Scenario arrival watchdog | Selected input never spills, or a worker failed before the requested boundary | Run the corresponding ordinary query and inspect the named task error and run counts |
| Extra group with region zero | A deleted account became a value, or a tombstone was ignored | Run the handwritten third phase before inspecting generated totals |
| Formula agrees at 1000 but not 1001 | Tail multiplicity is wrong | Lane 0 occurs twice; every other lane once |
| Run-descriptor limit | Row count exceeds `chunk_rows * max_runs` | Keep the test within the documented preset, or explicitly redesign the catalog; do not just increase the input |
| Completed CSV plus cleanup error | Publication and temporary-file deletion have different outcomes | Preserve the completed answer and inspect remaining spill owners/names |

As a useful variation, set `--hot-percent 100 --update-every 0 --delete-every 0`
and use the full day range. Event payloads then all target account 1, but account
updates/deletions still follow their separate rule. Predict the number of join
matches before running; then compare both replay and formula. Another variation
is batch size 1: the final values stay the same, while snapshot sequence numbers
and maintenance frequency change. Read metadata rather than hard-coding 5 and 6.

The completed chapter leaves a reproducible input format, two independent
answer paths, a bounded C++ adapter, and a thin driver over the real engine.
Chapter 22 will use these interfaces for recovery and resource-pressure
experiments. Normal command completion and a clean reopen here do not establish
the outcomes of a crash or simulated persistence failure.
