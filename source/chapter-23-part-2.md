# Chapter 23 — Profiling, optimization, and the final demonstration

## Part 2 — Remove work without changing the answer

Part 1 makes equivalent query shapes observable. This part adds two independent
controls: optional instrumentation to locate checksum work, and an optimized
checksum recurrence that preserves every format byte. It then supplies the
measurement harness. Keep the Part 1 files and all Chapter 22 prerequisites.

## Start with a hypothesis, not a preferred speedup

Every table frame and spill frame uses the same CRC-32 routine. With bounded
blocks, a query repeatedly validates existing frames, creates new ones, and
verifies intermediate replacements. The bitwise routine performs eight state
updates per input byte. This makes it a plausible CPU cost, not proof that it is
the dominant cost on every workload.

On a native Mac with Instruments available, use a `RelWithDebInfo` build, select
the **Time Profiler** template, choose `strata-measure` as the executable, and
supply the same database, temporary directory, output directory, variant, and
numeric arguments used below. Record several repetitions. Inspect the interval
covering query work, not only program launch. Use the call tree to distinguish
a function's own work from work attributed through its callees. A CPU profile
alone does not count exact invocations or explain elapsed time spent blocked.
Apple's [execution-frequency tutorial](https://developer.apple.com/tutorials/instruments/determining-execution-frequency)
explicitly separates those questions.

For command-line discovery, inspect the installed tools rather than assuming a
particular Xcode release or an Instruments installation:

```sh title="Terminal — optional native profiling tools"
xcrun --find xctrace
man xctrace
xcrun xctrace list templates
```

The [Apple command-line reference](https://developer.apple.com/documentation/xcode/xcode-command-line-tool-reference)
identifies `xctrace` as the trace recording/export tool and directs users to its
installed manual. The course's ordinary command-line build does not require
installing full Xcode solely to reproduce a screenshot. The scoped timer below
provides an additional reproducible, narrower measurement.

On Linux, an available GNU gprofng can sample the same optimized executable:

```sh title="Terminal — optional Linux sampling profile, after preparing measure-run"
gprofng collect app -p hi -o crc-baseline.er \
  build-baseline/strata-measure measure-run/database measure-run/temporary \
  measure-run/results profile.csv batch 256 512 2 7 100
gprofng display text -header -limit 20 -functions crc-baseline.er
```

The [GNU manual](https://sourceware.org/binutils/docs/gprofng.html) explains
sampling, inclusive/exclusive metrics, and these commands. Check the experiment
header before interpreting its function table. In the Linux check accompanying
this chapter, the collector warned that its interval timer changed to zero.
Its displayed percentages were **not accepted as reliable attribution**. A large
number in a profiler window does not override the profiler's own warning.

Use separate observations for separate questions: sampling for likely CPU paths;
scoped timings for a bounded region; allocation counters for ownership costs;
file-transfer counters for I/O volume; and waiting/rendezvous state for resource
admission or locks. A large elapsed-minus-CPU gap is a clue, not proof of disk
waiting: scheduling and unrelated contention can also create it.

## Make instrumentation optional and thread-local

Add these declarations to `table.hpp`. `CrcProfile` is a copyable observation,
not an owning resource. Reset and read it on the thread running the measured
synchronous query. No cross-thread aggregation is promised.

```diff title="include/strata/storage/table.hpp — exact prerequisite changes"
--- a/include/strata/storage/table.hpp
+++ b/include/strata/storage/table.hpp
@@ -27,6 +27,13 @@
 inline constexpr std::uint64_t max_file_bytes = 64 * 1024 * 1024;
 inline constexpr std::uint32_t bloom_probes = 6;
 
+// Optional same-thread instrumentation; not a concurrent aggregate.
+struct CrcProfile {
+    std::uint64_t calls = 0, bytes = 0, elapsed_ns = 0;
+    bool saturated = false;
+};
+CrcProfile crc_profile() noexcept;
+void reset_crc_profile() noexcept;
 std::uint32_t crc32(std::span<const std::byte> input,
                     std::uint32_t previous = 0) noexcept;
 void bloom_add(std::span<std::byte> bits, Key key);
```

The following exact `table.cpp` patch adds the profiling helper and the selectable
CRC implementation. Read its three mechanisms separately.

First, `thread_local` gives each calling thread its own counters. It does not
make a returned snapshot a live cross-thread view. `CrcTimer` records a
`steady_clock` point at entry and updates saturating counters at destruction.
No allocation or stream output occurs in the timed path. The saturation flag
turns an unusable observation into an explicit failure in the driver.

Second, `make_crc_table` is a `constexpr` function: the initialized array can be
computed during translation. Each entry applies the old eight-step recurrence
to one possible byte. The array occupies $256\times4=1024$ read-only data bytes;
it is not charged heap storage. `[[maybe_unused]]` avoids a warning when the
baseline build does not reference it. This attribute does not remove correctness
checks or change object lifetime.

Third, preprocessing selects one recurrence. `STRATA_CRC32_TABLE=0` retains the
original eight bit steps; `=1` uses one table lookup per byte.
`STRATA_PROFILE_CRC=0` removes the clock/counter path. Part 3's CMake options define
both macros as numeric 0 or 1. These controls are independent.

```diff title="src/storage/table.cpp — exact prerequisite changes"
--- a/src/storage/table.cpp
+++ b/src/storage/table.cpp
@@ -4,24 +4,83 @@
 #include <algorithm>
 #include <array>
 #include <cstring>
+#include <chrono>
+#include <limits>
 #include <stdexcept>
 #include <utility>
 
 namespace strata::storage {
 namespace table_format {
+namespace {
+#if STRATA_PROFILE_CRC
+thread_local CrcProfile current_profile;
+void profile_add(std::uint64_t& value, std::uint64_t delta) noexcept {
+    constexpr auto maximum = std::numeric_limits<std::uint64_t>::max();
+    if (delta > maximum - value) {
+        value = maximum; current_profile.saturated = true;
+    } else { value += delta; }
+}
+struct CrcTimer {
+    using Clock = std::chrono::steady_clock;
+    Clock::time_point start = Clock::now();
+    std::size_t bytes;
+    ~CrcTimer() noexcept {
+        const auto elapsed = std::chrono::duration_cast<std::chrono::nanoseconds>(Clock::now() - start).count();
+        profile_add(current_profile.calls, 1);
+        profile_add(current_profile.bytes, bytes);
+        profile_add(current_profile.elapsed_ns, static_cast<std::uint64_t>(elapsed));
+    }
+};
+#endif
+constexpr std::array<std::uint32_t, 256> make_crc_table() noexcept {
+    std::array<std::uint32_t, 256> result{};
+    for (std::size_t i = 0; i < result.size(); ++i) {
+        auto state = static_cast<std::uint32_t>(i);
+        for (unsigned bit = 0; bit < 8; ++bit) {
+            const bool low = (state & 1U) != 0;
+            state >>= 1;
+            if (low) { state ^= 0xedb88320U; }
+        }
+        result[i] = state;
+    }
+    return result;
+}
+[[maybe_unused]] constexpr auto crc_table = make_crc_table();
+}  // namespace
+CrcProfile crc_profile() noexcept {
+#if STRATA_PROFILE_CRC
+    return current_profile;
+#else
+    return {};
+#endif
+}
+void reset_crc_profile() noexcept {
+#if STRATA_PROFILE_CRC
+    current_profile = {};
+#endif
+}
 std::uint32_t crc32(std::span<const std::byte> input,
                     std::uint32_t previous) noexcept {
+    #if STRATA_PROFILE_CRC
+    const CrcTimer timer{CrcTimer::Clock::now(), input.size()};
+#endif
     std::uint32_t state = previous ^ 0xffffffffU;
     for (const auto byte : input) {
+#if STRATA_CRC32_TABLE
+        const auto index = (state ^ std::to_integer<std::uint32_t>(byte)) & 0xffU;
+        state = (state >> 8) ^ crc_table[index];
+#else
         state ^= std::to_integer<std::uint32_t>(byte);
         for (unsigned bit = 0; bit < 8; ++bit) {
             const bool low = (state & 1U) != 0;
             state >>= 1;
             if (low) { state ^= 0xedb88320U; }
         }
+#endif
     }
     return state ^ 0xffffffffU;
 }
+
 namespace {
 bool valid_filter_size(std::size_t size) noexcept {
     return size >= 8 && size <= max_filter_bytes && (size & (size - 1)) == 0;
```

A scope timer measures elapsed time **including any descheduling during the
scope**. Clock reads and counter updates have costs. Consequently its sum is not
an exact CPU-time partition, and an instrumented binary is not one of the
unprofiled timing-comparison builds. The harness rejects an instrumented record.
The code uses one start/end pair per CRC call, not per input byte.

## Derive the table recurrence

Let $s$ be the current unsigned 32-bit internal state, $b$ the next input byte,
and $P=\texttt{0xedb88320}$ the reflected polynomial already used by the file
format. One bit update is

$$
f(x)=(x\mathbin{\gg}1)\mathbin{\oplus}
\begin{cases}
P,&x\mathbin{\&}1=1,\\
0,&x\mathbin{\&}1=0.
\end{cases}
$$

Here $\gg$ is logical right shift, $\oplus$ is bitwise exclusive-or, and $\&$ is
bitwise and. The old byte step is $f^8(s\oplus b)$, where $f^8$ means applying
$f$ eight times. Define a 256-entry array by $T[i]=f^8(i)$ for byte values
$0\le i<256$.

Write $s\oplus b=(s\gg8)2^8+i$, with
$i=(s\oplus b)\mathbin{\&}255$. During those eight shifts, the original upper
24 bits move down eight positions. Their original low eight bits are zero, so
they cause no feedback on these first eight steps. The feedback from $i$ is
exactly $T[i]$. Exclusive-or is linear over these bit operations, giving

$$
s'=(s\gg8)\oplus T[(s\oplus b)\mathbin{\&}255].
$$

For an initially empty checksum, the internal state is `0xffffffff`. The first
ASCII byte of `"123456789"` is `0x31`. Its index is `0xce`, and the generated
entry is `T[0xce] = 0x7cdcefb7`. Thus

```text
state >> 8                 = 0x00ffffff
xor T[0xce]                = 0x7c231048
final xor for this one byte = 0x83dcefb7
```

The initial and final xor remain `0xffffffff`. A caller passing the previous
returned checksum therefore resumes the same internal state. Empty inputs and
split-buffer chaining are unchanged. CRC still detects certain accidental
changes; it is neither authentication nor a repair mechanism. Do not remove
checksums, persistence barriers, or required-file validation to obtain a faster
number.

## Define the experiment before running variants

Create `scripts/measure.py`. The workload is Chapter 21's documented periodic
recipe: by default 20,000 events, 257 accounts, ten days, eight regions, seed 23,
60 percent hot-account lanes, and 256 operations per input batch. Query days are
$[2,7)$. The generator also writes a delta, but the timed comparison loads only
the base. Final-demo scenarios exercise the delta separately.

With chunk capacity $C=512$, the first sort receives 10,000 selected rows in the
early plans and 20,000 in the late plan. Their initial run counts are

$$
R_{\mathrm{early}}=\lceil10000/512\rceil=20,\qquad
R_{\mathrm{late}}=\lceil20000/512\rceil=40.
$$

For fan-in four, the materialized passes reduce $20\to5\to2$ or $40\to10\to3$.
The remaining two or three runs stream into the next operator. Both sorts
therefore genuinely spill, even though this is a deliberately small reproducible
experiment. The account stream exceeds its configured batch capacity.

The run catalog still allows at most 128 initial runs. The harness checks the
unfiltered row count against $128C$ because the late plan sorts every event.
This bound is necessary, not sufficient: memory, disk, source metadata, and
compaction limits can reject a configuration earlier. The executable's maximum
chunk parameter does not promise that every allowed parameter fits its budget.

The first block imports the independent formula and defines the empirical
nearest-rank percentile. For $n$ sorted observations and integer percentile $p$,

$$
k=\lceil pn/100\rceil,\qquad Q_p=x_{k-1}.
$$

The array index is zero-based. With five observations, the empirical 95th
percentile is simply the maximum because $\lceil4.75\rceil=5$. It is not evidence
of a stable production tail. All summaries retain the sample count.

```python title="scripts/measure.py"
#!/usr/bin/env python3
"""Measure fresh-process queries on one prepared corpus; verify every answer."""
import argparse
import json
import math
from pathlib import Path
import platform
import random
import statistics
import subprocess
import sys
import time

from reference_query import compare, formula

HERE = Path(__file__).resolve().parent
VARIANTS = ("row", "batch", "fused", "late")


def nearest_rank(values, percent):
    if not values or not 1 <= percent <= 100:
        raise ValueError("nonempty samples and percentile 1..100 required")
    ordered = sorted(values)
    return ordered[(percent * len(ordered) + 99) // 100 - 1]
```

Capture actual build settings and a representative compilation command. The
experiment requires `CMAKE_EXPORT_COMPILE_COMMANDS=ON`; a directory name such as
`release` is not evidence of its flags. Equivalent compilers are allowed, but
two builds compared for one code change should otherwise use matching flags.

```python title="scripts/measure.py"
def build_description(directory):
    cache = directory / "CMakeCache.txt"
    commands = directory / "compile_commands.json"
    if not cache.is_file() or not commands.is_file():
        raise ValueError("build needs CMakeCache.txt and CMAKE_EXPORT_COMPILE_COMMANDS=ON")
    selected = [line for line in cache.read_text().splitlines()
                if line.startswith(("CMAKE_BUILD_TYPE:", "CMAKE_CXX_COMPILER:",
                                    "CMAKE_CXX_FLAGS", "STRATA_CRC32_TABLE:", "STRATA_PROFILE_CRC:"))]
    entries = json.loads(commands.read_text())
    command = next(item for item in entries if item["file"].endswith("src/storage/table.cpp"))
    compiler = next(line.split("=", 1)[1] for line in selected if line.startswith("CMAKE_CXX_COMPILER:"))
    banner = subprocess.run([compiler, "--version"], text=True, capture_output=True,
                            check=True, timeout=10).stdout.strip()
    return dict(directory=str(directory), compiler_banner=banner,
                cache=selected, representative_compile=command)
```

Run commands as argument arrays without a shell. Time the parent-observed process
interval separately from the query's internal interval. Keep stdout, stderr,
return code, and argument values when a command fails; never continue as though
a missing answer were a fast query. Python's
[`subprocess.run` contract](https://docs.python.org/3/library/subprocess.html#subprocess.run)
describes timeout handling and reaping of the immediate child. The timeout is a
diagnostic bound, not an asynchronous job service or a performance target.

```python title="scripts/measure.py"
def run(command, label, root, report, timeout):
    command = [str(x) for x in command]
    start = time.perf_counter_ns()
    completed = subprocess.run(command, text=True, capture_output=True, timeout=timeout)
    elapsed = (time.perf_counter_ns() - start) / 1e9
    (root / (label + ".stdout")).write_text(completed.stdout)
    (root / (label + ".stderr")).write_text(completed.stderr)
    report["commands"].append(dict(label=label, argv=command, wall_seconds=elapsed,
                                   returncode=completed.returncode))
    if completed.returncode:
        raise RuntimeError(f"{label} exited {completed.returncode}; retained stdout/stderr")
    return completed.stdout, elapsed
```

Parse exactly one JSON line. A profiling run with multiple repetitions is not
silently averaged here. Reject failed charge invariants and instrumentation-enabled
records before they enter a comparison. RSS conversion is an explicit parameter;
`null` normalized bytes with `rss_unit="raw"` means the unit was not established,
not zero memory.

```python title="scripts/measure.py"
def parse_record(text, rss_unit):
    lines = text.splitlines()
    if len(lines) != 1:
        raise ValueError("expected one measurement JSON record")
    row = json.loads(lines[0])
    for key in ("open_seconds", "query_seconds", "query_cpu_seconds"):
        if not isinstance(row.get(key), (int, float)) or not math.isfinite(row[key]) or row[key] < 0:
            raise ValueError(f"invalid {key}")
    if row["final_charged"] != 0 or row["peak_charged"] > row["query_cap"]:
        raise ValueError("charge invariant")
    if row["crc_profile_enabled"]:
        raise ValueError("timed comparisons require STRATA_PROFILE_CRC=OFF")
    row["rss_unit"] = rss_unit
    row["rss_max_bytes"] = (None if rss_unit == "raw" else
                             row["rss_max_raw"] * (1024 if rss_unit == "kib" else 1))
    return row
```

The experiment generates, loads/settles, explicitly compacts, and reopens in
separate commands. `load-and-settle` includes parsing, WAL synchronization,
concurrent maintenance, settlement, shutdown, and process overhead. Its throughput
is a named end-to-end ingestion measure, not the speed of one memtable loop.
The explicit compaction command prints that process's job traffic. Earlier
loading may have performed several jobs; its `last_job` counters are not their
sum and must not be advertised as total write amplification.

Warmup and measured rounds both execute **all** build/variant pairs. A seeded
shuffle changes their order per round rather than always giving the same variant
the warmest cache. Each child starts with a fresh private engine cache. The
operating-system cache is not cleared: call these repeated, OS-cache-uncontrolled
runs, not cold-device tests. Do not use privileged cache-dropping commands as a
hidden prerequisite.

```python title="scripts/measure.py"
def experiment(args, report):
    root = args.output.resolve()
    for name in ("temporary", "results"):
        (root / name).mkdir()
    builds = [("baseline", args.build.resolve())]
    if args.candidate:
        builds.append(("candidate", args.candidate.resolve()))
    report["builds"] = {label: build_description(path) for label, path in builds}
    executable = builds[0][1] / "strata"
    generated, generation_seconds = run(
        [sys.executable, HERE / "generate_data.py", root / "data", "--events", args.events,
         "--accounts", args.accounts, "--seed", args.seed, "--days", 10, "--regions", 8,
         "--hot-percent", 60, "--batch", 256], "generation", root, report, args.timeout)
    metadata = json.loads(generated)
    report["workload"] = metadata
    expected = formula(metadata, "base", args.begin, args.end)
    _, load_seconds = run([executable, "load", root / "database", root / "data/base.ops", "stream"],
                          "load-and-settle", root, report, args.timeout)
    run([executable, "compact", root / "database", "stream"], "explicit-compaction", root, report, args.timeout)
    run([executable, "inspect", root / "database", "stream"], "reopen-and-verify", root, report, args.timeout)
    report["load"] = dict(logical_bytes=metadata["base"]["logical_bytes"],
                           logical_bytes_per_second=metadata["base"]["logical_bytes"] / load_seconds,
                           includes="spawn, parse, WAL sync, concurrent maintenance, settle, shutdown")
    report["generation_seconds"] = generation_seconds
    report["samples"] = []
    pairs = [(label, path, variant) for label, path in builds for variant in VARIANTS]
    rng = random.Random(args.seed)
    for round_index in range(-args.warmups, args.repeats):
        order = pairs.copy()
        rng.shuffle(order)
        for label, path, variant in order:
            name = f"{label}-{variant}-{round_index}"
            text, wall = run([path / "strata-measure", root / "database", root / "temporary",
                             root / "results", name + ".csv", variant, args.rows, args.chunk,
                             args.begin, args.end, 1], name, root, report, args.timeout)
            row = parse_record(text, args.rss_unit)
            compare(root / "results" / (name + ".csv"), expected)
            if any((root / "temporary").iterdir()):
                raise ValueError("query left a spill name")
            if row["sequence"] != metadata["base"]["last_sequence"] or row["groups"] != len(expected):
                raise ValueError("measurement history or group count mismatch")
            if not row["event_runs"] or not row["group_runs"]:
                raise ValueError("measurement did not force both sorts to spill; enlarge input or reduce chunk")
            row.update(build=label, round=round_index, warmup=round_index < 0, process_seconds=wall)
            report["samples"].append(row)
            # A later failure retains all prior observations, without declaring completion.
            (root / "report.json").write_text(json.dumps(report, indent=2) + "\n")
```

Answer comparison and spill-directory inspection happen after the timed child
returns. Every sample must have the expected sequence and group count, no residual
spill names, and nonzero initial runs in both sorts. Reducing a filter until the
whole answer fits in memory would no longer satisfy this experiment's stated
purpose.

The report is saved incrementally with `complete=false`. A later failure retains
prior observations but produces no success claim. A report is ordinary measurement
data, not a publication journal. The summary below excludes every warmup.

```python title="scripts/measure.py"
    summary = []
    for label, _, variant in pairs:
        samples = [r for r in report["samples"] if r["build"] == label and r["variant"] == variant and not r["warmup"]]
        values = [r["query_seconds"] for r in samples]
        item = dict(build=label, variant=variant, n=len(values), minimum=min(values),
                    median=statistics.median(values), maximum=max(values),
                    sample_p95_nearest_rank=nearest_rank(values, 95),
                    tail_claim="none; these are empirical ranks, not production tail estimates")
        summary.append(item)
    report["summary"] = summary
    report["complete"] = True
```

Finish argument validation and the entry point. A fresh output directory is
required so a rerun cannot overwrite its prior evidence or a completed database.
`perf_counter_ns` avoids floating subtraction of large absolute timestamps; it
does not guarantee nanosecond physical resolution. The
[Python clock documentation](https://docs.python.org/3/library/time.html#time.perf_counter)
also supplies the queried implementation and resolution fields recorded here.

```python title="scripts/measure.py"
def main():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--build", type=Path, required=True)
    parser.add_argument("--candidate", type=Path)
    parser.add_argument("--output", type=Path, required=True)
    parser.add_argument("--events", type=int, default=20000)
    parser.add_argument("--accounts", type=int, default=257)
    parser.add_argument("--seed", type=int, default=23)
    parser.add_argument("--rows", type=int, default=256)
    parser.add_argument("--chunk", type=int, default=512)
    parser.add_argument("--begin", type=int, default=2)
    parser.add_argument("--end", type=int, default=7)
    parser.add_argument("--warmups", type=int, default=1)
    parser.add_argument("--repeats", type=int, default=5)
    parser.add_argument("--timeout", type=float, default=300)
    parser.add_argument("--rss-unit", choices=("raw", "bytes", "kib"), default="raw")
    args = parser.parse_args()
    if not (1 <= args.events <= 128 * args.chunk and 1 <= args.accounts <= 4096 and
            1 <= args.rows <= 4096 and 1 <= args.chunk <= 131072 and 0 <= args.seed < 1 << 64 and
            0 <= args.begin < args.end <= 10 and 0 <= args.warmups <= 10 and
            1 <= args.repeats <= 1000 and math.isfinite(args.timeout) and args.timeout > 0):
        parser.error("invalid workload/measurement bounds; late sort also needs <=128 initial runs")
    args.output.mkdir()  # Refuse an existing experiment; never delete it automatically.
    report = dict(complete=False, platform=platform.platform(), machine=platform.machine(),
                  python=sys.version, parameters={k: str(v) if isinstance(v, Path) else v
                                                 for k, v in vars(args).items()}, commands=[],
                  clock=time.get_clock_info("perf_counter").__dict__,
                  cache_policy="fresh private engine cache per query; OS cache not cleared; balanced shuffled order")
    try:
        experiment(args, report)
    except (OSError, ValueError, RuntimeError, KeyError, subprocess.SubprocessError) as error:
        report["failure"] = str(error)
        print(f"measurement failed: {error}; observations retained in {args.output}", file=sys.stderr)
        return 1
    finally:
        (args.output / "report.json").write_text(json.dumps(report, indent=2) + "\n")
    for item in report["summary"]:
        print(json.dumps(item, sort_keys=True))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

The report contains both process elapsed time and internal query elapsed time.
Subtracting the latter from the former does not isolate a single subsystem:
startup, opening, shutdown, JSON output, scheduling, and parent overhead remain.
Use the separately measured `open_seconds` to discuss reopening, while keeping
its verification work distinct from crash recovery correctness.
