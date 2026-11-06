# Chapter 23 — Profiling, optimization, and the final demonstration

## Part 1 — Define the operation before timing it

A faster query is not useful if it observes a different snapshot, silently skips
validation, or reports success before its output is complete. A lower charged
peak is not automatically a lower resident-memory peak. Finally, doing less
work and doing the same work faster are different improvements. This chapter
makes those distinctions executable.

We retain the complete Chapter 22 engine. Four equivalent query plans expose
batch size, intermediate copying, and predicate placement. A separately
selectable checksum implementation changes no stored bytes. A measurement
program runs the real two-sort join/aggregation pipeline, records its work, and
verifies every result using Chapter 21's independent answer. The final driver
then exercises the live and recovery scenarios together, without confusing a
controlled schedule with a throughput benchmark.

> **Learning objectives**
> Define timer and ownership boundaries; distinguish charged allocation from
> resident memory; compare semantically equivalent plans; derive a byte-at-a-time
> CRC update; interpret sampling and scoped instrumentation; retain a baseline;
> and report measured improvements, regressions, and unverified behavior.

**Prerequisites:** all source through
[Chapter 22](https://25d-228.github.io/cplusplus-learning/chapter-22.html), including
the Chapter 21 generator, analytical and replay references, `strata` driver,
Chapter 20 live engine, and Chapter 17 completion-aware result sink. The engine
should pass its existing correctness checks before measuring it.

**Implementation: 4.0 hours across all three parts.** Allocate 1.0 to the harness,
1.25 to profiling and the selectable optimization, 1.0 to repeated comparisons,
and 0.75 to final integration. The course allocation remains 86 hours plus the
6.5-hour debugging/integration reserve, excluding reading. Record actual time;
no additional implementation hours are inferred from the length of this chapter.

Create these six files, concatenating same-filename blocks in reading order:

```text
include/strata/query/selection.hpp
apps/measure_query.cpp
tests/measurement.cpp
scripts/measure.py
scripts/check_measurement.py
scripts/final_demo.py
```

Apply the exact prerequisite changes to `read_view.hpp/.cpp` in this part,
`table.hpp/.cpp` in Part 2, and append Part 3's CMake block. There is no replacement
storage engine or new concurrency protocol. The new measurement executable is
synchronous; the final demonstration uses the existing concurrent runtime.

[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-23-part-2.html)
implements instrumentation, CRC selection, and the experiment harness.
[Part 3](https://25d-228.github.io/cplusplus-learning/chapter-23-part-3.html)
provides tests, complete commands, observed results, and the final demonstration.

## One answer, four controlled plans

All variants use the same captured sequence, day interval, account relation,
join rule, grouping order, checked arithmetic, and CSV completion protocol.
Only the named decision changes:

| Variant | Event-side work before the join | Batch capacity |
|---|---|---:|
| `row` | Scan → filter → project → external sort | 1 |
| `batch` | The identical operators and predicate | Configured, initially 256 |
| `fused` | Scan → combined filter/project → external sort | Same configured capacity |
| `late` | Scan → project → external sort → filter | Same configured capacity |

The second joined-row sort and aggregate remain the same algorithms. The `row`
comparison sets capacities throughout the pipeline to one; it does **not** shrink
the sort chunk or spill block to one. Otherwise changing run count and system-call
granularity would be mistaken for the effect of operator batching.

The late predicate selects by event day and does not depend on account values.
Removing rows after sorting preserves the order of the rows that remain. It is
therefore legal for this fixed query, although more expensive when many rows are
rejected. This is not a rule that arbitrary predicates can cross arbitrary joins.
The native check compares every variant against both independent references.

A versioned scan row has 28 event-field bytes plus an eight-byte sequence. Its
column storage is 36 bytes per slot, independently of padding in a C++ aggregate.
For capacity $b$, the original filter input, project input, and sort input need

$$
M_{\mathrm{separate}} = 36b+36b+28b=100b.
$$

The fused operator keeps one versioned input and appends projected values directly
into the sort's input batch:

$$
M_{\mathrm{fused}}=36b+28b=64b,\qquad
\Delta M=36b.
$$

At $b=256$, this removes 9,216 simultaneous column bytes and one copied
versioned intermediate. Other operators, sort chunks, readers, cache entries,
and control objects still consume memory. The global peak need not fall by
exactly 9,216 because its maximum may occur in another phase. Both paths still
write the same **28-byte event encoding** to spill files. The native test requires
identical early-plan sort traffic; fusion is not credited with fictional on-disk
record compression.

## Fuse copying without weakening the pull protocol

The following first block creates `selection.hpp`. `Source` is constrained using
Chapter 5's concepts and Chapter 14's `PullSource`: it must supply versioned
`ScanBatch` values. Member declaration order matters: `source_` is constructed
before `input_`, so the latter can obtain the moved-in source's budget.
`check_output` runs before any source consumption. Its rejection leaves the
input position unchanged.

```cpp title="include/strata/query/selection.hpp"
#ifndef STRATA_QUERY_SELECTION_HPP
#define STRATA_QUERY_SELECTION_HPP
#include "strata/query/operators.hpp"

namespace strata::query {
// Same predicate and projected value as EventProject<EventFilter<Source>>.
// The version remains in the scan buffer; no second wide batch is copied.
template<class Source> requires PullSource<Source, ScanBatch>
class FilterProject {
public:
    FilterProject(Source source, DayRange days, std::size_t rows)
        : source_(std::move(source)), input_(source_.budget(), rows), days_(days) {}
    std::shared_ptr<memory::Budget> budget() const noexcept { return input_.budget(); }
    Pull next(EventBatch& output) & {
        check_output(output, budget(), input_.capacity(), state_);
        output.clear();
        if (state_ == StreamState::ended) { return Pull::end; }
        try {
            const auto result = source_.next(input_);
            if (result == Pull::end) {
                if (input_.size()) { throw std::logic_error("nonempty selection end"); }
                state_ = StreamState::ended; return Pull::end;
            }
            if (result != Pull::batch) { throw std::logic_error("invalid selection pull"); }
            const auto days = input_.events().columns().days;
            for (std::size_t i = 0; i < input_.size(); ++i) {
                if (days_.contains(days[i])) { output.append(input_.events().row(i)); }
            }
            return Pull::batch;
        } catch (...) {
            output.clear(); input_.clear(); state_ = StreamState::failed; throw;
        }
    }
private:
    Source source_;
    ScanBatch input_;
    DayRange days_;
    StreamState state_ = StreamState::ready;
};
```

`input_.events().row(i)` produces the event value, not a borrowed reference into
an output that will be refilled later. The sequence remains in the scan buffer;
the query does not need to copy it after the storage layer has resolved versions.
An empty selection returns `Pull::batch`. Only an upstream `Pull::end` ends this
operator. On a refill exception, both partial outputs are cleared and the stream
becomes terminal. Reusing its already-consumed source would not constitute retry.

For the late-filter experiment, the source supplies `EventBatch` instead. Append
the following complete operator and header closing lines. It has the same failure
and empty-progress behavior, but its input has already passed through the first
sort. Neither operator is a new asynchronous task.

```cpp title="include/strata/query/selection.hpp"
// A deliberately late predicate for the controlled sort-before-filter baseline.
// Removing rows from an ordered stream preserves the order of those remaining.
template<class Source> requires PullSource<Source, EventBatch>
class EventWhere {
public:
    EventWhere(Source source, DayRange days, std::size_t rows)
        : source_(std::move(source)), input_(source_.budget(), rows), days_(days) {}
    std::shared_ptr<memory::Budget> budget() const noexcept { return input_.budget(); }
    Pull next(EventBatch& output) & {
        check_output(output, budget(), input_.capacity(), state_);
        output.clear();
        if (state_ == StreamState::ended) { return Pull::end; }
        try {
            const auto result = source_.next(input_);
            if (result == Pull::end) {
                if (input_.size()) { throw std::logic_error("nonempty late-filter end"); }
                state_ = StreamState::ended; return Pull::end;
            }
            if (result != Pull::batch) { throw std::logic_error("invalid late-filter pull"); }
            for (std::size_t i = 0; i < input_.size(); ++i) {
                const auto event = input_.row(i);
                if (days_.contains(event.day)) { output.append(event); }
            }
            return Pull::batch;
        } catch (...) {
            output.clear(); input_.clear(); state_ = StreamState::failed; throw;
        }
    }
private:
    Source source_;
    EventBatch input_;
    DayRange days_;
    StreamState state_ = StreamState::ready;
};
}  // namespace strata::query
#endif
```

The classes own noncopyable batch members, so they cannot accidentally copy an
entire plan. Their implicit moves use the already-defined batch/source moves.
There is no custom destructor or worker to join here. Upstream owners remain
part of the plan and are released when that plan is destroyed.

## Observe private readers, not a shared mutable cache

Chapter 20 gives each query isolated readers and a private cache. Add a small
measurement snapshot to `ReadView`. It is deliberately **not** an API for reading
counters concurrently with another thread mutating those same readers.

```diff title="include/strata/storage/read_view.hpp — exact prerequisite changes"
--- a/include/strata/storage/read_view.hpp
+++ b/include/strata/storage/read_view.hpp
@@ -122,6 +122,10 @@
     std::shared_ptr<read_detail::Scan> scan_;
 };
 
+struct ReadMetrics {
+    std::uint64_t table_bytes = 0, block_reads = 0;
+    BlockCache::Statistics cache{};
+};
 class ReadView {
 public:
     std::shared_ptr<memory::Budget> budget_owner() const;
@@ -132,6 +136,8 @@
     std::uint64_t sequence() const;
     std::size_t source_count() const;
     ReadRetention retention() const;
+    // Sample only an isolated view on its owning thread, with no concurrent reads.
+    ReadMetrics metrics() const;
     std::optional<OwnedEntry> get(Key key) const;
     ReadCursor scan(Key lower, std::optional<Key> upper = std::nullopt) const;
 private:
```

```diff title="src/storage/read_view.cpp — exact prerequisite changes"
--- a/src/storage/read_view.cpp
+++ b/src/storage/read_view.cpp
@@ -1,4 +1,5 @@
 #include "strata/storage/read_view.hpp"
+#include "strata/core/checked.hpp"
 
 #include <algorithm>
 #include <exception>
@@ -336,4 +337,17 @@
     return ReadView{std::move(state)};  // Not registered in the parent's mutable retention registry.
 }
 
+ReadMetrics ReadView::metrics() const {
+    require_valid();
+    if (!state_->isolated) { throw std::logic_error("metrics require an isolated read owner"); }
+    ReadMetrics result;
+    result.cache = state_->isolated->cache.statistics();
+    for (const auto& source : state_->sources()) {
+        if (source.disk) {
+            result.table_bytes = checked_u64_add(result.table_bytes, source.disk->io_bytes().read);
+            result.block_reads = checked_u64_add(result.block_reads, source.disk->block_reads());
+        }
+    }
+    return result;
+}
 }  // namespace strata::storage
```

The added include supplies checked counter addition. `metrics()` rejects a normal
nonisolated view rather than suggesting that arbitrary parent-cache observations
are safe. It sums file-transfer and block-load counters across the current
isolated source set, then copies that isolated cache's statistics. `table_bytes`
counts bytes delivered through the file boundary, not physical device reads.
`block_reads` counts block loads, not hardware cache misses.

We retain an extra measurement view until the plan finishes so those counters
remain readable. This also retains its readers/cache longer than the minimal
production query might. Every compared variant pays that observation cost.
Do not silently compare these peaks with a run that omitted the observer.

Capture itself can read headers and metadata. Record a `before` snapshot after
capture, then `after` after consumption. Their difference isolates subsequent
query reads. The full query timer still includes capture and teardown. These are
two different boundaries, intentionally named rather than reconciled by hiding
one of the costs.

## Measure elapsed time, CPU time, and resident memory separately

Create `apps/measure_query.cpp`. `steady_clock` measures a monotonic interval;
`duration<double>` expresses its result in seconds rather than assuming the clock's
native tick unit. The [clock contract](https://eel.is/c++draft/time.clock.steady)
requires nondecreasing values unaffected by clock adjustment. It does not promise
nanosecond measurement accuracy merely because a duration can represent nanoseconds.

`getrusage` supplies cumulative process CPU time and a resident-memory high-water
mark. CPU time is a difference of user-plus-system time; maximum resident memory
is **not** a quantity to subtract between two samples. Its peak can come from
earlier opening work. The program outputs its raw value and leaves unit selection
to the harness.

```cpp title="apps/measure_query.cpp"
#include "strata/workload/driver.hpp"
#include "strata/query/selection.hpp"

#include <cerrno>
#include <chrono>
#include <iostream>
#include <string>
#include <system_error>
#include <sys/resource.h>

namespace {
using namespace strata;
using namespace strata::query;
using Clock = std::chrono::steady_clock;
double seconds(Clock::time_point start, Clock::time_point end) {
    return std::chrono::duration<double>(end - start).count();
}
struct Usage {
    rusage raw{};
    Usage() {
        if (::getrusage(RUSAGE_SELF, &raw) != 0) {
            throw std::system_error(errno, std::generic_category(), "getrusage");
        }
    }
    double cpu() const {
        return static_cast<double>(raw.ru_utime.tv_sec) + static_cast<double>(raw.ru_stime.tv_sec) +
            (static_cast<double>(raw.ru_utime.tv_usec) + static_cast<double>(raw.ru_stime.tv_usec)) / 1e6;
    }
};
```

The join/second-sort/aggregate suffix is shared literally, not reimplemented for
each benchmark. Function parameters take owning plans by value and then move
them into the next operator. `consume_results` remains responsible for publishing
only complete output. The group counter is a staged result count, not a claim
that partially written CSV has been committed.

```cpp title="apps/measure_query.cpp"
template<class Events> requires PullSource<Events, EventBatch>
void finish_query(Events events, storage::ReadView view, const QueryOptions& options,
                  const char* temporary, const std::shared_ptr<SortStats>& groups,
                  ResultSink& sink, std::uint64_t& count) {
    auto join = MergeJoin{std::move(events), AccountScan{std::move(view)}, options.join};
    auto sorted = JoinedExternalSort{std::move(join), options.joined_rows,
                                    options.group_sort, temporary, groups};
    auto plan = GroupAggregate{std::move(sorted), options.aggregate};
    consume_results(std::move(plan), options.output_rows, sink, count);
}
```

The dispatch below changes only the event-side composition. Class-template
argument deduction selects the concrete source types from constructor arguments;
this is the same composition technique used in Chapters 16–17. A copy of the
`ReadView` shares its immutable owner, not the contents of the database.

```cpp title="apps/measure_query.cpp"
void run_variant(std::string_view variant, storage::ReadView view, const QueryOptions& q,
                 const char* temporary, const std::shared_ptr<SortStats>& events,
                 const std::shared_ptr<SortStats>& groups, ResultSink& sink,
                 std::uint64_t& count) {
    const auto n = q.upstream_rows;
    if (variant == "late") {
        auto source = EventProject{EventScan{view}, n};
        auto sorted = ExternalSort{std::move(source), n, q.event_sort, temporary, events};
        auto selected = EventWhere{std::move(sorted), q.days, n};
        finish_query(std::move(selected), std::move(view), q, temporary, groups, sink, count);
    } else if (variant == "fused") {
        auto source = FilterProject{EventScan{view}, q.days, n};
        auto sorted = ExternalSort{std::move(source), n, q.event_sort, temporary, events};
        finish_query(std::move(sorted), std::move(view), q, temporary, groups, sink, count);
    } else {
        auto source = EventProject{EventFilter{EventScan{view}, q.days, n}, n};
        auto sorted = ExternalSort{std::move(source), n, q.event_sort, temporary, events};
        finish_query(std::move(sorted), std::move(view), q, temporary, groups, sink, count);
    }
}
```

The statistics handles do not own spill files. Inspecting them after the plan's
destruction reveals whether its cleanup completed. Reject saturated counters:
a wrapped or saturated byte total cannot support a quantitative comparison.

```cpp title="apps/measure_query.cpp"
void clean(const SortStats& s) {
    if (s.live_files || s.open_run_files || s.directory_live || s.cleanup_errors || s.saturated) {
        throw std::runtime_error("sort cleanup or counter failure");
    }
}
}  // namespace
```

Validate parameters before opening inputs or touching a result. Set all pipeline
capacities together, but keep merge work limits, block rows, and descriptor
allowances fixed. Five sort descriptors allow at most four input readers plus
one output writer, subject also to the memory calculation from Chapter 15.
The 128-run catalog bound remains in force.

```cpp title="apps/measure_query.cpp"
int main(int argc, char** argv) {
    try {
        if (argc != 11) {
            throw std::invalid_argument("usage: strata-measure DATABASE TEMP OUTPUT NAME row|batch|fused|late ROWS CHUNK DAY_BEGIN DAY_END REPEATS");
        }
        const std::string_view variant{argv[5]};
        if (variant != "row" && variant != "batch" && variant != "fused" && variant != "late") {
            throw std::invalid_argument("unknown measurement variant");
        }
        const auto requested = workload::unsigned_number(argv[6]);
        const auto chunk = workload::unsigned_number(argv[7]);
        const auto repeats = workload::unsigned_number(argv[10]);
        if (!requested || requested > 4096 || !chunk || chunk > 131072 || !repeats || repeats > 1000) {
            throw std::invalid_argument("measurement row/chunk/repeat bound");
        }
        const auto n = static_cast<std::size_t>(variant == "row" ? 1 : requested);
        auto preset = workload::preset("stream", {workload::unsigned_number(argv[8]),
                                                   workload::unsigned_number(argv[9])});
        auto q = preset.query;
        q.upstream_rows = n; q.join = {n, n, 1024}; q.joined_rows = n;
        q.aggregate = {n, 1024}; q.output_rows = n;
        for (auto* sort : {&q.event_sort, &q.group_sort}) {
            sort->chunk_rows = static_cast<std::size_t>(chunk);
            sort->block_rows = 64; sort->file_descriptors = 5;
        }
```

`FrozenInputs` reopens and verifies the compacted database once, with its default
64 MiB opening allowance. Each query gets a separate 32 MiB budget and fresh
private cache. These are admission limits, not preallocated arrays or a combined
process-memory cap.

The query timer starts before isolated capture and ends after owners and the
result sink leave scope. It therefore includes metadata opening, query work,
CSV close/publication, and cleanup. The database's earlier reopen interval is
reported separately. An optional same-thread CRC timer is reset here; Part 2
implements it. Normal measurement builds disable that instrumentation.

```cpp title="apps/measure_query.cpp"
        const auto opening_start = Clock::now();
        runtime::FrozenInputs inputs{argv[1], preset.engine.storage};
        const auto opening_end = Clock::now();
        for (std::uint64_t iteration = 0; iteration < repeats; ++iteration) {
            auto budget = std::make_shared<memory::Budget>(32 * 1024 * 1024);
            SortStats event_stats{}, group_stats{};
            storage::ReadMetrics before{}, after{};
            std::uint64_t count = 0;
            storage::table_format::reset_crc_profile();
            const Usage usage_before;
            const auto start = Clock::now();
            {
                auto view = inputs.capture(budget);  // New private cache on every repetition.
                before = view.metrics();
                auto events = make_sort_stats(budget), groups = make_sort_stats(budget);
                CsvResult sink{argv[3], argv[4]};
                run_variant(variant, view, q, argv[2], events, groups, sink, count);
                after = view.metrics();  // The observation handle intentionally retains readers.
                event_stats = *events; group_stats = *groups;
                clean(event_stats); clean(group_stats);
                if (!sink.published() || sink.cleanup_error()) { throw std::runtime_error("result cleanup"); }
            }
            const auto end = Clock::now();
            const Usage usage_after;
            const auto crc = storage::table_format::crc_profile();
            if (crc.saturated) { throw std::overflow_error("CRC profile saturated"); }
            if (budget->used()) { throw std::logic_error("measurement owners survived query"); }
```

`inputs` remains alive across optional repetitions for profiling. The main harness
uses one repetition per child, so each timed sample opens a new process. Repeated
in-process profile runs instead amortize that process's opening interval; do not
mix those samples into the fresh-process comparison.

The complete output record follows. It contains counters and parameters, not
paths interpolated into unescaped JSON. All numeric sizes are bytes unless their
field explicitly says rows, runs, passes, seconds, or raw RSS.

```cpp title="apps/measure_query.cpp"
            std::cout << "{\"variant\":\"" << variant << "\",\"iteration\":" << iteration
              << ",\"crc_profile_enabled\":" << STRATA_PROFILE_CRC
              << ",\"crc_calls\":" << crc.calls << ",\"crc_bytes\":" << crc.bytes
              << ",\"crc_seconds\":" << static_cast<double>(crc.elapsed_ns) / 1e9
              << ",\"crc_table\":" << STRATA_CRC32_TABLE << ",\"rows\":" << n
              << ",\"chunk_rows\":" << chunk << ",\"sequence\":" << inputs.sequence()
              << ",\"open_seconds\":" << seconds(opening_start, opening_end)
              << ",\"query_seconds\":" << seconds(start, end)
              << ",\"query_cpu_seconds\":" << usage_after.cpu() - usage_before.cpu()
              << ",\"rss_max_raw\":" << usage_after.raw.ru_maxrss
              << ",\"query_cap\":33554432,\"peak_charged\":" << budget->peak()
              << ",\"final_charged\":" << budget->used() << ",\"groups\":" << count
              << ",\"event_rows\":" << event_stats.input_rows
              << ",\"joined_rows\":" << group_stats.input_rows
              << ",\"event_runs\":" << event_stats.initial_runs << ",\"group_runs\":" << group_stats.initial_runs
              << ",\"event_passes\":" << event_stats.merge_passes << ",\"group_passes\":" << group_stats.merge_passes
              << ",\"event_read\":" << event_stats.read_bytes << ",\"group_read\":" << group_stats.read_bytes
              << ",\"event_write\":" << event_stats.write_bytes << ",\"group_write\":" << group_stats.write_bytes
              << ",\"capture_table_bytes\":" << before.table_bytes
              << ",\"query_table_bytes\":" << after.table_bytes - before.table_bytes
              << ",\"query_blocks\":" << after.block_reads - before.block_reads
              << ",\"cache_hits\":" << after.cache.hits - before.cache.hits
              << ",\"cache_misses\":" << after.cache.misses - before.cache.misses
              << ",\"cache_bypasses\":" << after.cache.bypasses - before.cache.bypasses
              << ",\"event_peak_disk\":" << event_stats.peak_disk_reserved
              << ",\"group_peak_disk\":" << group_stats.peak_disk_reserved << "}\n";
        }
    } catch (const std::exception& error) {
        std::cerr << "measurement: " << error.what() << '\n'; return 1;
    }
}
```

`event_read` includes reads used to verify intermediate replacements, not merely
one final scan. `event_peak_disk` and `group_peak_disk` are separate per-sort
high-water marks; adding them does not prove that both peaks occurred together.
A result with zero final charged bytes must still pass the spill-name and cleanup
checks. Conversely, zero live query owners says nothing about the master
`FrozenInputs`, stack, stream, allocator, or operating-system memory.

On Linux, [getrusage's manual](https://man7.org/linux/man-pages/man2/getrusage.2.html)
defines `ru_maxrss` in KiB, so conversion to bytes multiplies by 1024. On the Mac,
inspect the **installed** `man 2 getrusage` before choosing the conversion. The
harness accepts `raw`, `bytes`, or `kib` and defaults to `raw`; an unconfirmed unit
must not become a convincing but wrong memory figure. An archived manual for a
different SDK is not evidence of the currently installed interface. The native
M4 execution and local unit observation remain tasks to run on that machine.
