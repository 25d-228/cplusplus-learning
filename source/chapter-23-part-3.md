# Chapter 23 — Profiling, optimization, and the final demonstration

## Part 3 — Verify, repeat, and explain the result

The optimized recurrence is only useful if it preserves existing data. The
query variants are only comparable if they return the same complete answer.
This part supplies both checks, runs the final demonstration, and describes the
limits of the observed measurements. Keep Parts 1–2 and Chapter 22 source.

## Test the recurrence independently of its table

Create `tests/measurement.cpp`. The reference CRC retains a bitwise expression
and does not consult the generated lookup array. Check every prefix of a
1,024-byte pattern under three initial checksums, every possible one-byte input,
and the known `"123456789"` vector. Split each prefix at a third of its length to
verify chaining: this catches an incorrect initial/final xor even when a single
buffer happens to pass.

```cpp title="tests/measurement.cpp"
#include "strata/query/selection.hpp"
#include "strata/storage/table.hpp"

#include <array>
#include <iostream>
#include <string_view>

namespace {
using namespace strata;
using namespace strata::query;
void check(bool value, const char* message) {
    if (!value) { throw std::runtime_error(message); }
}
std::uint32_t reference(std::span<const std::byte> bytes, std::uint32_t previous) {
    auto state = ~previous;
    for (const auto byte : bytes) {
        state ^= std::to_integer<std::uint32_t>(byte);
        for (unsigned j = 0; j < 8; ++j) {
            state = (state >> 1) ^ ((state & 1U) ? 0xedb88320U : 0U);
        }
    }
    return ~state;
}
void crc_checks() {
    std::array<std::byte, 1024> bytes{};
    for (std::size_t i = 0; i < bytes.size(); ++i) { bytes[i] = std::byte((i * 37 + i / 7) & 255U); }
    const std::span<const std::byte> all{bytes};
    for (const auto seed : {0U, 0xffffffffU, 0x12345678U}) {
        for (std::size_t n = 0; n <= bytes.size(); ++n) {
            const auto prefix = all.first(n);
            check(storage::table_format::crc32(prefix, seed) == reference(prefix, seed), "CRC prefix differs");
            const auto split = n / 3;
            check(storage::table_format::crc32(prefix.subspan(split),
                  storage::table_format::crc32(prefix.first(split), seed)) == reference(prefix, seed),
                  "CRC chained prefix differs");
        }
    }
    for (unsigned value = 0; value < 256; ++value) {
        const std::array one{std::byte(value)};
        check(storage::table_format::crc32(one) == reference(one, 0), "CRC single byte differs");
    }
    constexpr std::string_view text = "123456789";
    check(storage::table_format::crc32(std::as_bytes(std::span{text.data(), text.size()})) == 0xcbf43926U,
          "CRC known vector differs");
}
```

The tiny source below makes empty progress observable before a selected row.
Its separate failing mode appends a partial input and then throws. This checks
state transitions, not merely cleanup at process exit.

```cpp title="tests/measurement.cpp"
struct WideSource {
    std::shared_ptr<memory::Budget> owner;
    std::size_t step = 0;
    bool fail = false;
    std::shared_ptr<memory::Budget> budget() const { return owner; }
    Pull next(ScanBatch& output) {
        output.clear();
        if (fail) { output.append({{99, 1, 4, 1}, 3}); throw std::runtime_error("source failure"); }
        if (step++ == 0) { output.append({{1, 1, 0, 7}, 3}); return Pull::batch; }
        if (step == 2) { return Pull::batch; }  // Empty progress, not completion.
        if (step == 3) { output.append({{2, 1, 4, 11}, 3}); return Pull::batch; }
        return Pull::end;
    }
};
```

A too-small output must be rejected before consuming the source. After that
rejection, the first legal call still sees the first row. A failed refill clears
output and makes later calls fail rather than retrying the consumed prefix.
The final budget check includes the destruction of both good and failed plans.

```cpp title="tests/measurement.cpp"
void selection_checks() {
    auto budget = std::make_shared<memory::Budget>(1024 * 1024);
    {
        FilterProject selected{WideSource{budget}, DayRange{4, 5}, 2};
        EventBatch output{budget, 2}, short_output{budget, 1};
        bool rejected = false;
        try { (void)selected.next(short_output); } catch (const std::invalid_argument&) { rejected = true; }
        check(rejected, "bad output must reject before source advances");
        check(selected.next(output) == Pull::batch && output.size() == 0, "filtered empty is not end");
        check(selected.next(output) == Pull::batch && output.size() == 0, "upstream empty is not end");
        check(selected.next(output) == Pull::batch && output.size() == 1 && output.row(0).amount_cents == 11,
              "selected row lost");
        check(selected.next(output) == Pull::end && output.size() == 0, "selection completion");
        FilterProject failed{WideSource{budget, 0, true}, DayRange{0, 10}, 2};
        bool threw = false;
        try { (void)failed.next(output); } catch (const std::runtime_error&) { threw = true; }
        check(threw && output.size() == 0, "failure clears output");
        threw = false;
        try { (void)failed.next(output); } catch (const std::logic_error&) { threw = true; }
        check(threw, "failed selection must be terminal");
    }
    check(budget->used() == 0, "selection charge survived owners");
}
}  // namespace
int main() {
    try {
        crc_checks(); selection_checks();
        std::cout << "measurement checks passed: 3075 prefixes, chaining, 256 bytes, selection boundaries\n";
    } catch (const std::exception& e) { std::cerr << e.what() << '\n'; return 1; }
}
```

## Compare outputs and work, not just checksums

Create `scripts/check_measurement.py`. It prepares 123 events and 17 accounts
and runs all four plans against a selective interval, an empty interval, and the
entire unsigned day domain. The analytical formula and bounded mutation replay
must agree before either is used to check CSV. This is 12 complete query
executions, not a hash of output produced by the same query implementation.

```python title="scripts/check_measurement.py"
#!/usr/bin/env python3
"""Independent answer checks for all four measured query shapes."""
import json
from pathlib import Path
import subprocess
import sys
import tempfile

from generate_data import generate
from reference_query import compare, formula, replay
from measure import nearest_rank, parse_record


def run(argv, ok=True):
    p = subprocess.run([str(x) for x in argv], capture_output=True, text=True, timeout=90)
    if (p.returncode == 0) != ok:
        raise RuntimeError(f"unexpected status {p.returncode}: {argv}\n{p.stdout}\n{p.stderr}")
    return p.stdout
```

```python title="scripts/check_measurement.py"
def main():
    executable, measured = map(lambda x: Path(x).resolve(), sys.argv[1:])
    if nearest_rank([9, 1, 5, 3, 7], 95) != 9:
        raise AssertionError("nearest-rank percentile")
    with tempfile.TemporaryDirectory(prefix="strata-measure-check-") as directory:
        root = Path(directory)
        p = dict(seed=23, events=123, accounts=17, days=10, regions=8,
                 hot_percent=60, batch=32, update_every=7, delete_every=13)
        meta = generate(root / "data", p)
        run([executable, "load", root / "db", root / "data/base.ops", "stream"])
        run([executable, "compact", root / "db", "stream"])
        (root / "temporary").mkdir(); (root / "results").mkdir()
        observed = {}
        for lower, upper in ((2, 7), (0, 0), (0, 1 << 32)):
            expected = formula(meta, "base", lower, upper)
            independent = replay([root / "data/base.ops"], meta["base"]["last_sequence"], lower, upper, 1000)
            if expected != independent:
                raise AssertionError("replay/formula disagreement")
            for variant in ("row", "batch", "fused", "late"):
                text = run([measured, root / "db", root / "temporary", root / "results",
                            "answer.csv", variant, 3, 8, lower, upper, 1])
                row = parse_record(text, "raw")
                compare(root / "results/answer.csv", expected)
                if list((root / "temporary").iterdir()):
                    raise AssertionError("spill cleanup")
                if (lower, upper) == (2, 7):
                    observed[variant] = row
```

The early variants must have identical sort traffic and run counts. Moving a
predicate later must increase first-sort spill writes in the selective fixture.
Those assertions distinguish physical work from elapsed-time noise. A rejected
configuration must leave an already completed answer unchanged.

```python title="scripts/check_measurement.py"
        for field in ("event_read", "event_write", "group_read", "group_write", "event_runs", "group_runs"):
            if len({observed[v][field] for v in ("row", "batch", "fused")}) != 1:
                raise AssertionError(f"batching/fusion changed physical sort work: {field}")
        if not observed["late"]["event_write"] > observed["batch"]["event_write"]:
            raise AssertionError("late predicate did not increase spill writes")
        # Bad configuration must not replace an already completed answer.
        prior = (root / "results/answer.csv").read_bytes()
        run([measured, root / "db", root / "temporary", root / "results", "answer.csv",
             "fused", 0, 8, 2, 7, 1], ok=False)
        if (root / "results/answer.csv").read_bytes() != prior:
            raise AssertionError("rejected measurement replaced completed answer")
        print("verified 12 query variants/bounds, identical early-sort traffic, greater late spill, no residual files")
        print(json.dumps({v: {k: row[k] for k in ("event_rows", "event_write", "event_runs", "groups")}
                          for v, row in observed.items()}, sort_keys=True))


if __name__ == "__main__":
    main()
```

Append these targets and options to the existing root `CMakeLists.txt`. The
checksum table is on by default, while timing instrumentation is off. The
compile definitions are public so the linked measurement executable reports the
same choices as its core library. There is no download, database dependency, or
benchmark framework.

```cmake title="CMakeLists.txt — append Chapter 23 targets"
# Chapter 23: separately selectable format-preserving checksum implementation.
option(STRATA_CRC32_TABLE "Use a 256-entry CRC table instead of eight bit steps" ON)
option(STRATA_PROFILE_CRC "Time CRC calls on the current thread; not a benchmark build" OFF)
target_compile_definitions(strata-core PUBLIC
    STRATA_CRC32_TABLE=$<BOOL:${STRATA_CRC32_TABLE}>
    STRATA_PROFILE_CRC=$<BOOL:${STRATA_PROFILE_CRC}>)
add_executable(strata-measure apps/measure_query.cpp)
target_link_libraries(strata-measure PRIVATE strata-workload)
target_compile_options(strata-measure PRIVATE -Wall -Wextra -Wpedantic -Wconversion -Wshadow)
add_executable(strata-measurement-tests tests/measurement.cpp)
target_link_libraries(strata-measurement-tests PRIVATE strata-core)
target_compile_options(strata-measurement-tests PRIVATE -Wall -Wextra -Wpedantic -Wconversion -Wshadow)
add_test(NAME measurement-unit COMMAND strata-measurement-tests)
add_test(NAME measurement-native COMMAND ${Python3_EXECUTABLE}
    ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_measurement.py $<TARGET_FILE:strata> $<TARGET_FILE:strata-measure>)
set_tests_properties(measurement-unit measurement-native PROPERTIES TIMEOUT 180)
```

## Build matching baseline and candidate binaries

Use the native compiler selected in Chapter 1. The commands below intentionally
do not require one compiler version or fabricate an LTS designation. Both builds
use the same build type; only the checksum option differs. Keep profiling and
sanitizer flags out of these performance builds.

```sh title="Terminal — optimized baseline and candidate"
cmake -S . -B build-baseline -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DSTRATA_CRC32_TABLE=OFF -DSTRATA_PROFILE_CRC=OFF
cmake --build build-baseline --parallel 4
ctest --test-dir build-baseline --output-on-failure
cmake -S . -B build-candidate -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DSTRATA_CRC32_TABLE=ON -DSTRATA_PROFILE_CRC=OFF
cmake --build build-candidate --parallel 4
ctest --test-dir build-candidate --output-on-failure
```

The new unit test prints:

```text
measurement checks passed: 3075 prefixes, chaining, 256 bytes, selection boundaries
```

A separate memory-sanitized build remains a correctness experiment. Its allocator,
layout, and instrumentation costs cannot establish optimized throughput.

```sh title="Terminal — focused memory-sanitized checks"
cmake -S . -B build-memory -DCMAKE_BUILD_TYPE=Debug \
  -DSTRATA_CRC32_TABLE=ON -DSTRATA_PROFILE_CRC=OFF \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build-memory --parallel 4
ctest --test-dir build-memory --output-on-failure
```

Now run the actual comparison. `raw` is portable as an unconverted observation;
use `kib` on Linux after confirming its documented units, or `bytes` when the
installed Mac manual specifies bytes. Record the chosen interpretation.

```sh title="Terminal — fresh measurement directory"
python3 scripts/measure.py --build build-baseline --candidate build-candidate \
  --output measure-run --events 20000 --accounts 257 --seed 23 \
  --rows 256 --chunk 512 --warmups 1 --repeats 5 --rss-unit raw
```

Avoid other builds or deliberate stress processes while collecting comparison
samples. The harness runs all samples sequentially; it does not reserve exclusive
hardware or make a shared machine quiet. Keep rejected or noisy trials separate
rather than selecting the best repetitions after seeing their times.

A separate instrumented build can locate CRC work without contaminating that
comparison. The reset/read operations concern the current synchronous query
thread only. Thirty repetitions make small regions easier to inspect while
opening the master input just once.

```sh title="Terminal — optional scoped CRC profile"
cmake -S . -B build-profile -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DSTRATA_CRC32_TABLE=OFF -DSTRATA_PROFILE_CRC=ON
cmake --build build-profile --target strata-measure --parallel 4
build-profile/strata-measure measure-run/database measure-run/temporary \
  measure-run/results scoped-profile.csv batch 256 512 2 7 30
```

Its `crc_calls`, `crc_bytes`, and `crc_seconds` expose execution count, processed
volume, and elapsed scope time. A real parser rejects a profile counter that
saturated. The main harness refuses these records even if the result is correct.

## Observe admission at a controlled pause

At the live scenario's existing two gates, both requests are admitted and cannot
complete until their gates are released. Add these observations to
`apps/scenario.cpp`. This changes no scheduling or storage decision. The fixed
source upper bound and both sort allowances require 29 descriptor slots per task;
two tasks hold 58. A reservation is an admission promise, not a claim that 58
native descriptors are open or 4 MiB have been allocated.

```diff title="apps/scenario.cpp — exact prerequisite changes"
--- a/apps/scenario.cpp
+++ b/apps/scenario.cpp
@@ -81,6 +81,11 @@
             previous.open(root / "out/cancel.csv"); previous << "previous completed result\n"; previous.close();
         }
         const auto cancel = submit(executor, p, "cancel.csv", &cancel_gate); cancel_gate.await();
+        const auto held = executor.stats();
+        require(held.running == 2 && held.used_bytes == 2 * p.query_bytes && held.used_files == 58,
+                "paused tasks did not hold complete reservations");
+        std::cout << "reservations held_bytes=" << held.used_bytes << " held_files=" << held.used_files
+                  << " running=" << held.running << '\n';
         const auto epoch = engine->stats().epoch;
         bool pressure = false;
         try { (void)engine->capture(std::make_shared<memory::Budget>(p.query_bytes)); }
@@ -100,6 +105,11 @@
         }
         const auto fresh = submit(executor, p, "new.csv");
         old_gate.release(); executor.finish();
+        const auto released = executor.stats();
+        require(released.used_bytes == 0 && released.used_files == 0 && released.running == 0,
+                "joined executor retained reservations");
+        std::cout << "reservations final_bytes=" << released.used_bytes << " final_files=" << released.used_files
+                  << " peak_bytes=" << released.peak_bytes << " peak_running=" << released.peak_running << '\n';
         const auto before = wait_result(old), after = wait_result(fresh);
         require_completed(before); require_completed(after);
         require(before.sequence == original && after.sequence == latest, "mixed snapshot boundary");
```

The joined executor must return both admission counters to zero. Its recorded
peak is separate from query-local charged peaks and resident memory. Calling
`stats()` at an arbitrary moment would not establish the same boundary; the
already-implemented gates make the observation repeatable.

## Run the combined demonstration

Create `scripts/final_demo.py`. It orchestrates the existing mechanisms; it does
not introduce another recovery algorithm or hide missing synchronization inside
a shell sleep. First, the controlled live scenario holds the old sequence while
writes and maintenance advance the engine. It rejects excess retention, cancels
a spilling query, then releases owners and collects obsolete files.

```python title="scripts/final_demo.py"
#!/usr/bin/env python3
"""Run the controlled live scenario, independent answers, recovery, and pressure."""
import argparse
import json
from pathlib import Path
import subprocess
import sys
import time

from reference_query import compare, formula, replay

HERE = Path(__file__).resolve().parent


def main():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--build", type=Path, required=True)
    parser.add_argument("--output", type=Path, required=True)
    args = parser.parse_args()
    build, root = args.build.resolve(), args.output.resolve()
    root.mkdir()  # Never erase an existing demonstration.
    report = dict(complete=False, stages=[])

    def run(name, command):
        start = time.perf_counter()
        result = subprocess.run([str(x) for x in command], text=True, capture_output=True, timeout=300)
        (root / (name + ".stdout")).write_text(result.stdout)
        (root / (name + ".stderr")).write_text(result.stderr)
        report["stages"].append(dict(name=name, seconds=time.perf_counter() - start,
                                     returncode=result.returncode, argv=[str(x) for x in command]))
        if result.returncode:
            raise RuntimeError(f"{name} failed; inspect its retained logs")
        return result.stdout
```

Independently check the old and new files. The reopened query runs in a separate
process against the same live-scenario database, after the original engine has
finished. Cancellation must preserve the previous complete output verbatim.

```python title="scripts/final_demo.py"
    try:
        metadata = json.loads(run("generate", [sys.executable, HERE / "generate_data.py", root / "data"]))
        executable = build / "strata"
        run("live", [executable, "scenario", root / "live", root / "data/base.ops",
                     root / "data/delta.ops", 0, 8])
        for phase, name in (("base", "old.csv"), ("delta", "new.csv")):
            expected = formula(metadata, phase, 0, 8)
            inputs = [root / "data/base.ops"]
            if phase == "delta":
                inputs.append(root / "data/delta.ops")
            if expected != replay(inputs, metadata[phase]["last_sequence"], 0, 8, 1000):
                raise ValueError("formula/replay mismatch")
            compare(root / "live/out" / name, expected)
        run("reopened-query", [executable, "query", root / "live/db", root / "live/spill",
                               root / "live/out", "reopened.csv", 0, 8, "tiny"])
        compare(root / "live/out/reopened.csv", formula(metadata, "delta", 0, 8))
        if (root / "live/out/cancel.csv").read_text() != "previous completed result\n":
            raise ValueError("cancellation changed previous answer")
        if list((root / "live/spill").iterdir()):
            raise ValueError("live demonstration spill residue")
```

The persistence model, parent-controlled SIGKILL campaign, and resource-pressure
campaign are separate experiments with separate assumptions. Run all of them,
then rerun the variant boundaries. A failed stage keeps its diagnostic files and
leaves `complete=false`. No automatic deletion or blind retry is performed.

```python title="scripts/final_demo.py"
        run("persistence-model", [build / "strata-recovery-tests"])
        for mode in ("native", "pressure"):
            run("recovery-" + mode, [sys.executable, HERE / "check_recovery.py",
                                    build / "strata-recovery-native", mode])
        run("measurement-boundaries", [sys.executable, HERE / "check_measurement.py",
                                       executable, build / "strata-measure"])
        report["complete"] = True
        print("verified old/new/reopened answers, canceled output, live reclamation, model, SIGKILL, pressure, and variants")
    except (OSError, ValueError, RuntimeError, subprocess.SubprocessError) as error:
        report["failure"] = str(error)
        print(f"final demonstration failed: {error}", file=sys.stderr)
        return 1
    finally:
        (root / "result.json").write_text(json.dumps(report, indent=2) + "\n")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

```sh title="Terminal — combined checked demonstration"
python3 scripts/final_demo.py --build build-candidate --output final-demo
```

A successful run prints:

```text
verified old/new/reopened answers, canceled output, live reclamation, model, SIGKILL, pressure, and variants
```

The default live workload has a base at sequence 5 and a delta at sequence 6.
It forces both query sorts to spill, retains the old answer across ingestion and
compaction, preserves canceled output, and checks the reopened fresh answer.
The recovery stage explores Chapter 22's 2,754 selected crash images, its native
26 stop-point kills, and pressure failures. This script does not turn those
bounded campaigns into exhaustive interleaving or physical power-loss tests.

## Observations from an executed run

The following are observations from **Clang 17, Linux x86_64**, using matching
`RelWithDebInfo` builds (`-O2 -g -DNDEBUG`), the 20,000-event fixture, seed 23,
batch capacity 256, chunk capacity 512, block capacity 64, and fan-in four.
Each pair had one excluded warmup and five measured fresh-process samples.
OS caches were not cleared. No deliberate build or profiling job ran during
this final comparison. These are not M4 measurements or production tail estimates.

| Plan | Bitwise CRC median (ms) | Table CRC median (ms) | Candidate range (ms), n=5 |
|---|---:|---:|---:|
| `row` | 94.79 | 38.21 | 35.41–43.73 |
| `batch` | 95.99 | 35.79 | 33.03–42.39 |
| `fused` | 93.66 | 35.57 | 31.43–40.38 |
| `late` | 129.07 | 55.12 | 49.63–60.66 |

The bitwise batched median was slightly **slower** than its one-row median in
this run. That is retained rather than recast as a batching success. The candidate
batch and fused timings are close and their ranges overlap; five repetitions do
not establish a stable timing advantage for fusion. The batch checksum comparison
was about 2.68 times faster by the ratio of these sample medians. That number
applies to this workload/build/environment, not to CRC implementations generally.

The separate instrumented baseline executed **2,832 CRC calls over 5,571,838
bytes per query**. Across 30 in-process repetitions, median accumulated CRC scope
time was 72.69 ms and median query elapsed time was 99.35 ms. These are elapsed
instrumented regions, not reliable CPU percentages or uninstrumented latency.
Together with the format-preserving toggle, they support CRC as a consequential
cost in this fixture. The warning-bearing sampling profile was not used for
quantitative attribution.

| Work or charge | `row` | `batch` | `fused` | `late` |
|---|---:|---:|---:|---:|
| First-sort input rows | 10,000 | 10,000 | 10,000 | 20,000 |
| First-sort initial runs | 20 | 20 | 20 | 40 |
| First-sort written bytes | 796,240 | 796,240 | 796,240 | 1,694,448 |
| First-sort read bytes | 1,309,396 | 1,309,396 | 1,309,396 | 2,822,740 |
| Second-sort written bytes | 909,008 | 909,008 | 909,008 | 909,008 |
| Peak query charge, bytes | 500,834 | 558,974 | 549,758 | 556,926 |

All plans returned **40 groups from the same 10,000 joined events at sequence 80**.
The second sort had 20 initial runs; both sorts made two materialized merge
passes. Fusion removed exactly 9,216 bytes from the observed charged peak here.
It did not change spill traffic. Late filtering increased first-sort writes from
796,240 to 1,694,448 bytes. The number of passes remained two, but the first sort
processed twice as many rows and generated more frames/runs. A pass count alone
would conceal that increase.

Every plan recorded 20,194 table bytes during isolated capture, then 1,053,396
table bytes and 259 block loads during query reads. Its private cache recorded
259 misses and one hit. These are engine counters, not physical disk traffic
or processor cache statistics. They are consistent across the controlled plans.

The workload contained **603,598 logical input bytes**. Generation took 1.323 s,
load-and-settle 0.310 s, explicit base repacking 0.107 s, and reopening/verification
0.039 s as parent-observed command intervals. The named ingestion rate was about
**1.94 million logical bytes/s**, including the costs listed in the report. These
single phase observations are not repeated throughput estimates. The explicit
compaction process reported one job: 1,073,590 bytes read and the same number
written. Its output/logical-input ratio is approximately 1.78; it is a ratio for
that **one explicit base repack**, not cumulative lifetime write amplification.

Candidate process high-water observations ranged from 98,787,328 to 99,180,544
bytes after Linux KiB conversion, despite query charged peaks below 0.6 MiB.
That difference is not an unexplained ledger leak: the two measurements cover
different objects and intervals. In particular, Linux resource observations
survive `exec`, and process launch/opening history can influence this high-water
value ([getrusage notes](https://man7.org/linux/man-pages/man2/getrusage.2.html)).
It is **not query-only resident memory**. Phase-specific resident-memory sampling
with the native profiler is needed to attribute that value; do not subtract it
from the query ledger or claim a 32 MiB process cap.

In the controlled live demonstration, the paused old and cancelable queries hold
**4,194,304 reserved bytes and 58 descriptor slots** together. After joined
shutdown both admission counts return to zero. The old sequence-5 query produces
16 groups; sequence 6 produces 24. Each completed query makes five materialized
passes in each sort. Cancellation preserves its previous completed output,
reopening preserves sequence 6, and the final collection removes two obsolete
files. Reserved allowance, charged allocations, observed resident memory, and
retained disk files are four distinct quantities.

Both optimized builds passed all **39 CTest entries**. The candidate also passed
all 39 in a Debug AddressSanitizer/UndefinedBehaviorSanitizer build. The canonical
listings reconstruct **124 learner source/build files**. The independent variant
and final-demo checks passed. The empty-means-end diagnostic below failed the
intended assertion in optimized and memory-sanitized builds without a sanitizer
memory report; the correct implementation remains in these listings.

**Limits:** native M4/macOS execution, Apple full-device synchronization, hardware
power-loss durability, larger-scale throughput, and representative tail latency
remain unverified. No speedup is required when repeating the experiment.


## Make one deliberate defect visible

Temporarily change the fused operator's successful return from `Pull::batch` to
`output.size() ? Pull::batch : Pull::end`. Rebuild only `strata-measurement-tests`
and run it. The expected failure is `filtered empty is not end`, before memory
safety tools need to report anything. Restore the correct return and rerun the
unit and native checks. This is a semantic defect in a valid-memory program.

The checksum optimization has a different failure mode: an incorrect recurrence
can make both writers and readers agree with each other while disagreeing with
the prior format. Retain the independent bitwise oracle, known vector, split
checks, and the existing Python/native file decoders. A self-consistent roundtrip
alone is not compatibility evidence.

## Diagnose discrepancies before tuning another parameter

| Observation | Check next |
|---|---|
| Different sequence or CSV between variants | Stop timing; inspect capture and predicate placement. |
| Early-plan spill bytes differ | Verify equal chunk/block settings and projected fields; batching alone should not change this fixture's runs. |
| A profile shows nearly no samples or warns about its timer | Treat percentages as unusable; use another collector or the scoped experiment. |
| Charged peak falls but resident maximum does not | Check measurement lifetime, opening peak, allocator retention, and RSS units. |
| CPU time is much smaller than elapsed | Inspect scheduling, filesystem delays, and wait states; do not infer one cause from the difference alone. |
| `maximum initial runs` or workspace rejection | Check the late plan's unfiltered count, $\lceil n/C\rceil$, and simultaneous memory before increasing a limit. |
| Result is correct but cleanup checks fail | Preserve the diagnostic; a published answer does not imply reclaimed temporary disk. |
| A failure run appears much faster | Exclude it from success latency; compare only completed, independently verified answers. |

A useful variation is to change the predicate to select almost every day while
keeping the data, chunk, and batch settings fixed. Predict the first-sort run
counts before executing. As selectivity approaches one, the early predicate's
spill-volume advantage should disappear. The fused path can still remove an
intermediate copy. If timings regress, retain the observation and explain the
additional operator or cache effects instead of changing the answer definition.

## What is now complete

Strata connects explicit record encodings, lifetime-safe ownership, a prepared
skip-list batch, WAL-backed publication, immutable indexed tables, pinned cache
entries, manifest authority, snapshot retention, streaming compaction, bounded
external sorts, a many-to-one merge join, checked aggregation, cooperative
cancellation, and a supervised live runtime. The demonstrations exercise these
as one system rather than isolated feature names.

The scope remains deliberately single-node, fixed-schema, and single-writer.
There is no general SQL parser, arbitrary many-to-many join, distributed commit,
lock-free index, or unbounded disk-backed run catalog. The measured fixture is
smaller than the roadmap's exploratory 512 MiB–1 GiB scale; it does not establish
that larger scale. Increasing data requires checking table, catalog, disk,
retention, and admission bounds while preserving spill and dimension-streaming
conditions. The implemented difficult mechanisms are not replaced with a
wrapper to avoid those constraints.

On the M4, rerun correctness first, then record the actual compiler, flags, local
memory units, workload, and measurements. Do not transfer the Linux times or a
CRC speedup ratio to that hardware. A useful result is an independently correct
answer accompanied by a defensible explanation of its work, costs, and failure
boundaries—not a required performance number.
