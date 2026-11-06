# Chapter 15 — External sorting with bounded fan-in

## Part 3 — Pressure, corruption, and the native run oracle

This is the final part of the same **6.0-hour chapter**. Complete
[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-15.html) and
[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-15-part-2.html) first.
The remaining files exercise the implementation, not a simplified replacement.
Keep every earlier CTest entry.

A sorted output alone is not enough. We need to distinguish input exhaustion from
empty progress, account for input/output coexistence, observe failure after a
successful prefix, and establish that a retained output batch owns its values.
The fixture deliberately keeps its reference vectors outside the engine budget:
they are an oracle, not workspace silently available to the algorithm.

## Construct a source with observable empty progress

Create `tests/external_sort.cpp`. The test-only outer directory removes everything
it owns when the test ends. That broad fixture cleanup is **not** the engine's
cleanup policy: assertions inspect live names, counters, and directory emptiness
before the fixture disappears. The engine removes only its registered names.

The independent comparator constructs value tuples rather than calling
`EventOrder`. A shared ordering bug should not make the implementation and its
oracle agree by construction. Small reference vectors may use `std::sort`; the
code under test still generates and merges actual files.

```cpp title="tests/external_sort.cpp — complete file, segment 1"
#include "strata/query/external_sort.hpp"
#include "strata/query/operators.hpp"
#include "strata/storage/database.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <cstddef>
#include <cstdint>
#include <cstdlib>
#include <cstdio>
#include <exception>
#include <filesystem>
#include <fstream>
#include <iostream>
#include <limits>
#include <memory>
#include <span>
#include <stdexcept>
#include <string>
#include <string_view>
#include <system_error>
#include <tuple>
#include <type_traits>
#include <utility>
#include <vector>

namespace {
using namespace strata;
using namespace strata::query;
using namespace strata::storage;
namespace fs = std::filesystem;

void check(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}
template<class Error, class Function>
void expect(const Function& function) {
    try { function(); } catch (const Error&) { return; }
    throw std::runtime_error("expected exception missing");
}
class TemporaryDirectory {
public:
    TemporaryDirectory() {
        char pattern[] = "/tmp/strata-sort-test-XXXXXX";
        const auto path = ::mkdtemp(pattern);
        if (!path) { throw std::system_error(errno, std::generic_category(), "fixture directory"); }
        name_ = path;
    }
    ~TemporaryDirectory() noexcept { std::error_code ignored; fs::remove_all(name_, ignored); }
    const std::string& name() const noexcept { return name_; }
private:
    std::string name_;
};
SortOptions tiny() {
    SortOptions result;
    result.chunk_rows = 3; result.block_rows = 2;
    result.file_descriptors = 3; result.workspace_bytes = 128 * 1024;
    result.stack_allowance = 16 * 1024; result.disk_bytes = 64 * 1024;
    return result;
}
std::vector<Event> fixture(std::size_t count) {
    std::vector<Event> rows;
    for (std::size_t i = 0; i < count; ++i) {
        rows.push_back(Event{(i * 7) % 11, (i * 5) % 4,
                             static_cast<std::uint32_t>(i % 3), 100 + i});
    }
    return rows;
}
auto reference(std::vector<Event> rows) {
    // Independent value-tuple comparison: do not call the engine's EventOrder.
    std::sort(rows.begin(), rows.end(), [](Event a, Event b) {
        return std::tuple{a.account_id, a.event_id, a.day, a.amount_cents} <
               std::tuple{b.account_id, b.event_id, b.day, b.amount_cents};
    });
    return rows;
}
```

`Source` alternates an empty batch with a data batch. It may also emit empty
progress immediately before end. Once `ended_` is set, it never returns a batch
again. Its borrowed fixture span is safe because every test's `rows` owner
outlives the plan. This is a fixture-specific borrowing contract, not permission
for the query engine to retain arbitrary caller buffers.

`drain` accepts only nonempty output batches from this particular sealed sort.
That is stronger than the general upstream protocol: consuming all source input
has already happened before the first sorted batch is exposed. The second end
call checks absorption rather than assuming it.

```cpp title="tests/external_sort.cpp — complete file, segment 2"
class Source {
public:
    Source(std::shared_ptr<memory::Budget> budget, std::span<const Event> rows, bool empties)
        : budget_(std::move(budget)), rows_(rows), empties_(empties) {}
    std::shared_ptr<memory::Budget> budget() const noexcept { return budget_; }
    Pull next(EventBatch& output) {
        check_output(output, budget_, 1, StreamState::ready); output.clear();
        if (ended_) { return Pull::end; }
        if (empties_ && empty_turn_) { empty_turn_ = false; return Pull::batch; }
        empty_turn_ = true;
        if (at_ == rows_.size()) { ended_ = true; return Pull::end; }
        while (at_ < rows_.size() && output.size() < output.capacity()) { output.append(rows_[at_++]); }
        return Pull::batch;
    }
private:
    std::shared_ptr<memory::Budget> budget_;
    std::span<const Event> rows_;  // Test fixture owns this, outside engine accounting.
    bool empties_, empty_turn_ = true, ended_ = false;
    std::size_t at_ = 0;
};
static_assert(PullSource<ExternalSort<Source>, EventBatch>);
static_assert(!std::is_copy_constructible_v<SortBuffer>);
static_assert(std::is_nothrow_move_constructible_v<SortBuffer>);

template<class Plan>
std::vector<Event> drain(Plan& plan, EventBatch& output) {
    std::vector<Event> result;
    while (plan.next(output) == Pull::batch) {
        check(output.size() != 0, "a sealed sort should not return empty progress");
        for (std::size_t i = 0; i < output.size(); ++i) { result.push_back(output.row(i)); }
    }
    check(output.size() == 0 && plan.next(output) == Pull::end, "absorbing sort end");
    return result;
}
```

## Check boundaries before large inputs

With chunk capacity three, counts zero through three must create no run files.
Count four must spill. Counts six and seven distinguish an exact chunk boundary
from a partial final chunk. Count 384 fills exactly 128 initial-run slots without
requiring a 129th descriptor. Output capacities one and seven cut across all of
those boundaries independently.

For 25 rows, assert the exact nine initial runs, three materialized passes, seven
merge groups, and sixteen completed files. The read total includes verification
of intermediate outputs; it is not simply twice the write total. After drain,
all temporary names and descriptors must be gone even though the plan and report
are still alive. Their fixed object storage can remain charged until destruction.

```cpp title="tests/external_sort.cpp — complete file, segment 3"
void check_shape(std::size_t count, std::size_t output_capacity, bool print = false) {
    TemporaryDirectory parent;
    auto budget = std::make_shared<memory::Budget>(2 * 1024 * 1024);
    const auto rows = fixture(count);
    {
        auto stats = make_sort_stats(budget);
        {
            ExternalSort plan{Source{budget, rows, true}, 2, tiny(), parent.name(), stats};
            EventBatch output{budget, output_capacity};
            const auto actual = drain(plan, output);
            check(actual == reference(rows), "external sort differs from independent oracle");
            check(stats->input_rows == count && stats->output_rows == count, "sort counts");
            check(stats->live_files == 0 && !stats->directory_live && stats->disk_reserved == 0,
                  "drained sort leaked temporary storage");
            check(stats->open_run_files == 0 && stats->peak_open_run_files <= 3, "descriptor bound");
            check(stats->peak_workspace <= tiny().workspace_bytes, "workspace bound");
            if (count <= 3) { check(stats->files_created == 0 && stats->write_bytes == 0, "no-spill path"); }
            if (count == 25) {
                check(stats->initial_runs == 9 && stats->merge_passes == 3 && stats->merge_groups == 7,
                      "multi-pass plan");
                check(stats->files_created == 16 && stats->completed_files == 16, "run file count");
                check(stats->write_bytes == 4312 && stats->read_bytes == 7180, "actual transfer totals");
                if (print) {
                    std::cout << "rows=25 initial_runs=" << stats->initial_runs
                        << " merge_passes=" << stats->merge_passes << " fan_in=" << stats->fan_in
                        << " writes=" << stats->write_bytes << " reads=" << stats->read_bytes
                        << " peak_disk=" << stats->peak_disk_reserved
                        << " fixed_charge=" << stats->fixed_charge
                        << " peak_workspace=" << stats->peak_workspace << '\n';
                }
            }
        }
        check(stats->cleanup_errors == 0 && fs::is_empty(parent.name()), "ordinary cleanup");
    }
    check(budget->used() == 0, "shape ledger leak");
}
```

## Separate misuse from a failed execution

A call before `finish()` is misuse. A destination belonging to another ledger is
also rejected before consuming a row or clearing that destination. Neither needs
to poison an otherwise usable sort. A genuine execution failure after input has
been consumed is different: the source position cannot be reconstructed by simply
trying the same call again.

Moving the facade preserves the stable `Core` address. Move assignment first
releases the destination's previous owner; it does not overwrite raw pointers to
live files. The rejected options exercise initialization before disk effects.

```cpp title="tests/external_sort.cpp — complete file, segment 4"
void check_preflight_and_moves() {
    TemporaryDirectory parent;
    auto budget = std::make_shared<memory::Budget>(2 * 1024 * 1024);
    auto other = std::make_shared<memory::Budget>(1024);
    {
        auto stats = make_sort_stats(budget);
        SortBuffer first{budget, tiny(), parent.name(), stats};
        first.append(Event{2, 1, 0, 9}); first.append(Event{1, 1, 0, 7});
        expect<std::logic_error>([&] { EventBatch out{budget, 1}; first.next(out); });
        first.finish();
        SortBuffer moved{std::move(first)};
        EventBatch wrong{other, 1}; wrong.append(Event{99, 99, 99, 99});
        expect<std::invalid_argument>([&] { moved.next(wrong); });
        check(wrong.row(0).event_id == 99 && stats->output_rows == 0, "preflight consumed input");
        auto second_stats = make_sort_stats(budget);
        SortBuffer destination{budget, tiny(), parent.name(), second_stats};
        destination = std::move(moved);
        EventBatch output{budget, 1};
        check(drain(destination, output) == reference({Event{2,1,0,9}, Event{1,1,0,7}}), "move sort state");
    }
    check(budget->used() == 0 && other->used() == 0, "move ledger release");
    for (int option = 0; option < 5; ++option) {
        auto config = tiny();
        if (option == 0) { config.chunk_rows = 0; }
        if (option == 1) { config.block_rows = 0; }
        if (option == 2) { config.file_descriptors = 2; }
        if (option == 3) { config.max_runs = 129; }
        if (option == 4) { config.stack_allowance = 0; }
        auto stats = make_sort_stats(budget);
        expect<std::invalid_argument>([&] { SortBuffer sort{budget, config, parent.name(), stats}; });
        check(fs::is_empty(parent.name()), "bad config created disk work");
    }
    {
        auto config = tiny(); config.workspace_bytes = 1;
        auto stats = make_sort_stats(budget);
        expect<std::length_error>([&] { SortBuffer sort{budget, config, parent.name(), stats}; });
    }
    check(budget->used() == 0, "rejected workspace charge");
}
```

The workspace test measures the current implementation's fixed allocation, then
chooses an allowance that fits exactly two or four input frames. The descriptor
allowance is deliberately generous, so **bytes**, rather than descriptor count,
select the fan-in. For block capacity two, $F=12+28(2)=68$ bytes. At fan-in four,
the nine runs require one materialized pass: $9\to3$, followed by streaming output.
At fan-in two, they require three: $9\to5\to3\to2$.

One byte below $H+S+3F$ cannot support two inputs and one output. Rejection must
happen before a row is consumed. Exact duplicate records and maximum unsigned
fields check both equivalence and the codec's endpoint behavior; no signed
subtraction is used to determine their order.

```cpp title="tests/external_sort.cpp — complete file, segment 5"
void check_fan_in_and_duplicate_values() {
    TemporaryDirectory parent;
    auto budget = std::make_shared<memory::Budget>(2 * 1024 * 1024);
    std::size_t fixed = 0;
    {
        auto stats = make_sort_stats(budget);
        SortBuffer probe{budget, tiny(), parent.name(), stats};
        fixed = stats->fixed_charge;
    }
    for (const std::size_t fan_in : {2, 4}) {
        auto options = tiny(); options.file_descriptors = 33;
        options.workspace_bytes = fixed + options.stack_allowance + (fan_in + 1) * 68;
        auto stats = make_sort_stats(budget);
        const auto rows = fixture(25);
        ExternalSort plan{Source{budget, rows, true}, 2, options, parent.name(), stats};
        EventBatch output{budget, 3};
        check(drain(plan, output) == reference(rows), "byte-limited merge result");
        check(stats->fan_in == fan_in && stats->peak_workspace == options.workspace_bytes,
              "fan-in did not follow byte allowance");
        check(stats->merge_passes == (fan_in == 2 ? 3 : 1), "byte-limited pass count");
    }
    {
        auto options = tiny(); options.workspace_bytes = fixed + options.stack_allowance + 3 * 68 - 1;
        auto stats = make_sort_stats(budget);
        expect<std::length_error>([&] { SortBuffer sort{budget, options, parent.name(), stats}; });
    }
    {
        const auto maximum = std::numeric_limits<std::uint64_t>::max();
        std::vector<Event> rows(39, Event{7, 2, 4, 99});
        rows.push_back(Event{maximum, maximum, std::numeric_limits<std::uint32_t>::max(), maximum});
        rows.push_back(Event{0, 0, 0, 0});
        auto stats = make_sort_stats(budget);
        ExternalSort plan{Source{budget, rows, true}, 2, tiny(), parent.name(), stats};
        EventBatch output{budget, 3};
        check(drain(plan, output) == reference(rows), "exact duplicates or unsigned endpoints");
    }
    check(budget->used() == 0 && fs::is_empty(parent.name()), "fan-in ledger");
}
```

## Make file failures concrete

The hook is a synchronous diagnostic seam, not a scheduler or callback framework.
Its context is borrowed and must outlive the plan. The hook sees the complete
private-directory path and the current operation's file ID and offset. It may
throw before an operation or alter a just-completed file for the next reader.
It must not reenter the sorter.

The disk failure occurs before the operation at offset 104: the 36-byte header
and one 68-byte data frame already exist. This checks preservation of an actual
successful write prefix, not a fictitious rollback to an empty file. Chapter 6
already tests short system transfers within `File::write_all`; this seam tests
how the enclosing owner handles a later operation's failure.

The huge-header and wrong-order cases recompute their CRCs. They must therefore
be rejected by structural and semantic validation, not accidentally by an earlier
checksum mismatch. The copy mode is only for the independent small-file oracle;
these extra test copies are explicitly outside engine disk accounting.

```cpp title="tests/external_sort.cpp — complete file, segment 6"
std::string run_name(const char* directory, std::uint64_t id) {
    std::array<char, 32> name{};
    std::snprintf(name.data(), name.size(), "run-%020llu.bin", static_cast<unsigned long long>(id));
    return (fs::path{directory} / name.data()).string();
}
struct Fault {
    enum class Kind { disk, missing, checksum, huge_header, order, late, cleanup, copy };
    Kind kind;
    bool armed = true;
    std::string copies;
};
void alter_byte(const std::string& path, std::uint64_t offset) {
    auto file = io::File::open(path, io::OpenMode::read_write);
    std::array<std::byte, 1> byte{};
    file.read_exact(offset, byte); byte[0] ^= std::byte{1}; file.write_all(offset, byte);
    file.close_checked();
}
void hook(void* raw, SpillEvent event, const char* directory, std::uint64_t id, std::uint64_t offset) {
    auto& fault = *static_cast<Fault*>(raw);
    if (!fault.armed) { return; }
    if (fault.kind == Fault::Kind::copy && event == SpillEvent::completed_run) {
        fs::copy_file(run_name(directory, id), fs::path{fault.copies} / (std::to_string(id) + ".run"));
        return;
    }
    if (fault.kind == Fault::Kind::cleanup && event == SpillEvent::before_unlink) {
        throw std::system_error(EACCES, std::generic_category(), "injected unlink refusal");
    }
    if (id != 1) { return; }
    if (fault.kind == Fault::Kind::disk && event == SpillEvent::before_write && offset == 104) {
        fault.armed = false;
        throw io::IoError(ENOSPC, "injected later-frame disk failure", offset, 0);
    }
    if (fault.kind == Fault::Kind::late && event == SpillEvent::before_read && offset == 104) {
        alter_byte(run_name(directory, id), offset + 8); fault.armed = false; return;
    }
    if (event != SpillEvent::completed_run) { return; }
    if (fault.kind == Fault::Kind::missing) { fs::remove(run_name(directory, id)); fault.armed = false; }
    if (fault.kind == Fault::Kind::checksum) { alter_byte(run_name(directory, id), 44); fault.armed = false; }
    if (fault.kind == Fault::Kind::huge_header) {
        auto file = io::File::open(run_name(directory, id), io::OpenMode::read_write);
        std::array<std::byte, 36> header{}; file.read_exact(0, header);
        put_le<4>(std::span{header}.subspan(16, 4), 0xffffffffU);
        put_le<4>(std::span{header}.last(4), table_format::crc32(std::span{header}.first(32)));
        file.write_all(0, header); file.close_checked(); fault.armed = false;
    }
    if (fault.kind == Fault::Kind::order) {
        auto file = io::File::open(run_name(directory, id), io::OpenMode::read_write);
        std::array<std::byte, 68> frame{}; file.read_exact(36, frame);
        std::swap_ranges(frame.begin() + 8, frame.begin() + 36, frame.begin() + 36);
        put_le<4>(std::span{frame}.last(4), table_format::crc32(std::span{frame}.first(64)));
        file.write_all(36, frame); file.close_checked(); fault.armed = false;
    }
}
```

Each early failure must clear the current output and retire every owned run.
A missing run counts as confirmed absent during cleanup; the descriptor owner
must not treat a failed `openat` as an acquired file. In the injected disk failure,
`write_bytes` remains 104 and `completed_files` remains zero. Those are different
facts: partial bytes transferred are real even though no usable run exists.

The terminal-state assertion follows the first exception. It prevents a seemingly
successful retry that silently omits input already consumed before failure.

```cpp title="tests/external_sort.cpp — complete file, segment 7"
void check_faults() {
    const auto rows = fixture(10);
    for (const auto kind : {Fault::Kind::disk, Fault::Kind::missing, Fault::Kind::checksum,
                            Fault::Kind::huge_header, Fault::Kind::order}) {
        TemporaryDirectory parent;
        auto budget = std::make_shared<memory::Budget>(2 * 1024 * 1024);
        {
            auto stats = make_sort_stats(budget); Fault fault{kind, true, {}};
            ExternalSort plan{Source{budget, rows, false}, 2, tiny(), parent.name(), stats, SpillHook{hook, &fault}};
            EventBatch output{budget, 2};
            expect<std::exception>([&] { drain(plan, output); });
            check(output.size() == 0 && stats->live_files == 0 && stats->open_run_files == 0 &&
                  stats->disk_reserved == 0 && !stats->directory_live, "failure cleanup");
            expect<std::logic_error>([&] { plan.next(output); });
            if (kind == Fault::Kind::disk) {
                check(stats->write_bytes == 104 && stats->completed_files == 0, "successful disk prefix accounting");
            }
        }
        check(budget->used() == 0 && fs::is_empty(parent.name()), "fault lifetime release");
    }
}
```

## Fail after returning a valid batch

Use a five-row first run with two-row frames. Returning event zero advances that
reader to event one without reading its second frame yet. Arm the mutation only
after that first output. The next `take` copies event one, then advancing the
reader reaches offset 104 and discovers corruption. That call does **not** publish
its partial output: it clears it and terminalizes the stream. The separately
owned earlier batch still contains event zero.

Cleanup failure is not the same as a wrong query result. With unlink deliberately
refused, a correct result can finish while two files remain registered and charged.
The report retains the first error, live-name count, directory path, and disk
reservation. After removing the refusal, `retry_cleanup()` confirms deletion and
releases the charge. Destroying the sort also attempts bounded cleanup; it cannot
promise that the operating system will grant permission.

```cpp title="tests/external_sort.cpp — complete file, segment 8"
void check_late_and_cleanup() {
    TemporaryDirectory parent;
    auto budget = std::make_shared<memory::Budget>(2 * 1024 * 1024);
    {
        auto stats = make_sort_stats(budget); Fault fault{Fault::Kind::late, false, {}};
        auto options = tiny(); options.chunk_rows = 5;
        SortBuffer sort{budget, options, parent.name(), stats, SpillHook{hook, &fault}};
        for (std::uint64_t i = 0; i < 6; ++i) { sort.append(Event{i, 1, 0, i}); }
        sort.finish();
        EventBatch held{budget, 1}, output{budget, 1};
        check(sort.next(held) == Pull::batch && held.row(0).event_id == 0, "first output before corruption");
        fault.armed = true;
        expect<FormatError>([&] { sort.next(output); });
        check(output.size() == 0 && held.row(0).event_id == 0, "failed refill invalidated old owned output");
        expect<std::logic_error>([&] { sort.next(output); });
    }
    check(budget->used() == 0, "late failure ledger");
    {
        auto stats = make_sort_stats(budget); Fault fault{Fault::Kind::cleanup, true, {}};
        SortBuffer sort{budget, tiny(), parent.name(), stats, SpillHook{hook, &fault}};
        for (const auto row : fixture(4)) { sort.append(row); }
        sort.finish(); EventBatch output{budget, 2};
        check(drain(sort, output) == reference(fixture(4)), "cleanup error damaged correct result");
        check(stats->cleanup_errors > 0 && stats->first_cleanup_error == EACCES &&
              stats->live_files == 2 && stats->disk_reserved > 0 && stats->directory_live,
              "failed cleanup was hidden or uncharged");
        fault.armed = false; sort.retry_cleanup();
        check(stats->live_files == 0 && stats->disk_reserved == 0 && !stats->directory_live, "cleanup retry");
    }
    check(budget->used() == 0 && fs::is_empty(parent.name()), "cleanup ledger");
}
```

## Charge coexistence, not just completed input

A 1,500-byte disk allowance admits the initial 1,444 bytes but cannot admit the
first 264-byte replacement alongside them. Deleting an input early would hide
that shortage by sacrificing failure safety. Instead the operation fails and
owned temporary files are cleaned up.

Allocation injection walks charged acquisition ordinals until the first complete
success. This includes construction, column buffers, chunks, and merge-reader
frames. Every failed attempt is followed by zero outstanding run files and zero
remaining budget charge after its scope ends. The number of failure boundaries
is observed, not fixed as a portable ABI requirement.

```cpp title="tests/external_sort.cpp — complete file, segment 9"
void check_caps_and_allocations() {
    TemporaryDirectory parent;
    auto budget = std::make_shared<memory::Budget>(2 * 1024 * 1024);
    const auto rows = fixture(25);
    for (int mode = 0; mode < 2; ++mode) {
        auto options = tiny();
        if (mode == 0) { options.max_runs = 2; }
        else { options.disk_bytes = 1500; } // Initial 1444 fits; replacement coexistence does not.
        auto stats = make_sort_stats(budget);
        ExternalSort plan{Source{budget, rows, false}, 2, options, parent.name(), stats};
        EventBatch output{budget, 2};
        expect<std::length_error>([&] { drain(plan, output); });
        check(stats->live_files == 0 && stats->disk_reserved == 0, "cap cleanup");
    }
    check(budget->used() == 0, "cap ledger");
    std::size_t failures = 0;
    for (std::size_t ordinal = 1; ordinal < 128; ++ordinal) {
        bool success = false;
        {
            auto stats = make_sort_stats(budget);
            budget->fail_allocation_on(ordinal);
            try {
                ExternalSort plan{Source{budget, rows, true}, 2, tiny(), parent.name(), stats};
                EventBatch output{budget, 2};
                check(drain(plan, output) == reference(rows), "allocation retry result");
                success = true;
            } catch (const std::bad_alloc&) { ++failures; }
            budget->clear_failure();
            check(stats->live_files == 0 && stats->disk_reserved == 0 && stats->open_run_files == 0,
                  "allocation failure leaked run");
        }
        check(budget->used() == 0 && fs::is_empty(parent.name()), "allocation failure leaked charge");
        if (success) { break; }
        check(ordinal + 1 < 128, "allocation campaign did not reach success");
    }
    check(failures > 20, "too few allocation boundaries exercised");
    std::cout << "allocation_failures=" << failures << '\n';
}
```

## Connect the actual storage scan

The integrated plan uses an immutable database view, then
`EventScan -> EventFilter -> EventProject -> ExternalSort`. The first four events
have day zero; the remaining 25 have day three. With upstream capacity two, the
first two scan batches produce no matches. A sorter that mistakes either empty
batch for end would create no useful output and never exercise spilling.

No intermediate vector of all query rows is passed to the engine. Only the test
fixture and the independent expected result use vectors outside the ledger. The
plan's input and output batches, snapshot, and sort all share the same budget.

```cpp title="tests/external_sort.cpp — complete file, segment 10"
void check_pipeline() {
    TemporaryDirectory database_dir, parent;
    auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    using Plan = EventProject<EventFilter<EventScan>>;
    {
        DatabaseOptions options; options.arena_bytes = 256 * 1024;
        options.table = {128, 256, 32, 64 * 1024};
        Database database{budget, database_dir.name(), DatabaseOpen::create, options};
        std::vector<Mutation> mutations;
        std::vector<Event> expected;
        // First two scan batches (capacity two) have no matches.
        for (std::uint64_t i = 0; i < 29; ++i) {
            const Event row{i, (i * 5) % 7, static_cast<std::uint32_t>(i < 4 ? 0 : 3), i * 100};
            mutations.push_back(Mutation::put(row));
            if (row.day == 3) { expected.push_back(row); }
        }
        check(database.write(mutations).outcome == WriteOutcome::committed, "pipeline fixture");
        auto view = database.snapshot();
        auto stats = make_sort_stats(budget);
        Plan upstream{EventFilter{EventScan{view}, DayRange{3,4}, 2}, 2};
        ExternalSort plan{std::move(upstream), 2, tiny(), parent.name(), stats};
        EventBatch output{budget, 3};
        check(drain(plan, output) == reference(expected), "storage pipeline sort integration");
        check(stats->initial_runs == 9 && stats->merge_passes == 3, "pipeline did not spill");
    }
    check(budget->used() == 0 && fs::is_empty(parent.name()), "pipeline cleanup");
}
```

Pausing after one output leaves the final two input runs live. Destruction must
close readers before retiring those names. The next case consumes all remaining
root-budget headroom before final merge initialization. The sort-local plan may
fit, but the **shared** budget can still reject a reader. Releasing that pressure
must not revive a failed stream.

Finally, reset the caller's budget handle before finishing a spill. The sort,
output, report, and allocator controls still own the ledger. A weak observation
expires only after their final releases. This catches a lifetime error that a
numerically correct charge counter alone cannot detect.

```cpp title="tests/external_sort.cpp — complete file, segment 11"
void check_abandonment_and_shared_pressure() {
    TemporaryDirectory parent;
    auto budget = std::make_shared<memory::Budget>(2 * 1024 * 1024);
    {
        auto stats = make_sort_stats(budget);
        {
            SortBuffer sort{budget, tiny(), parent.name(), stats};
            for (const auto row : fixture(25)) { sort.append(row); }
            sort.finish(); EventBatch output{budget, 1};
            check(sort.next(output) == Pull::batch && stats->live_files == 2, "paused spilled output");
            // Destroy before draining: the run owners still have to retire their names.
        }
        check(stats->live_files == 0 && stats->open_run_files == 0 && !stats->directory_live,
              "abandoned sort leaked storage");
    }
    {
        auto stats = make_sort_stats(budget);
        SortBuffer sort{budget, tiny(), parent.name(), stats};
        for (const auto row : fixture(6)) { sort.append(row); }
        auto pressure = budget->reserve(budget->limit() - budget->used());
        expect<memory::BudgetExceeded>([&] { sort.finish(); });
        pressure.reset();
        EventBatch output{budget, 1};
        expect<std::logic_error>([&] { sort.next(output); });
        check(stats->live_files == 0 && stats->disk_reserved == 0, "shared pressure leaked spill");
    }
    check(budget->used() == 0 && fs::is_empty(parent.name()), "pressure ledger");
    std::weak_ptr<memory::Budget> weak = budget;
    {
        auto stats = make_sort_stats(budget);
        SortBuffer sort{budget, tiny(), parent.name(), stats};
        for (const auto row : fixture(4)) { sort.append(row); }
        EventBatch output{budget, 1}; budget.reset();
        sort.finish();
        check(drain(sort, output) == reference(fixture(4)) && !weak.expired(), "escaped budget lifetime");
    }
    check(weak.expired(), "final sort control retained budget");
}
```

## Keep completed native files for a separate decoder

The special fixture invocation copies each completed file to the caller's test
directory before ordinary engine cleanup. It checks the query result independently
and prints only deterministic transfer counters for the Python harness. Normal
invocation runs every test group, including exact no-spill boundaries and the
largest permitted initial-run count in this tiny configuration.

```cpp title="tests/external_sort.cpp — complete file, segment 12"
void native_fixture(const std::string& copies) {
    TemporaryDirectory parent;
    auto budget = std::make_shared<memory::Budget>(2 * 1024 * 1024);
    auto stats = make_sort_stats(budget);
    Fault copy{Fault::Kind::copy, true, copies};
    const auto rows = fixture(25);
    {
        ExternalSort plan{Source{budget, rows, true}, 2, tiny(), parent.name(), stats, SpillHook{hook, &copy}};
        EventBatch output{budget, 4};
        check(drain(plan, output) == reference(rows), "fixture sort");
    }
    check(stats->live_files == 0 && !stats->directory_live, "fixture cleanup");
    std::cout << "initial_runs=" << stats->initial_runs << " merge_passes=" << stats->merge_passes
              << " writes=" << stats->write_bytes << " reads=" << stats->read_bytes << '\n';
}
}  // namespace

int main(int argc, char** argv) {
    try {
        if (argc == 3 && std::string_view{argv[1]} == "fixture") { native_fixture(argv[2]); return 0; }
        if (argc != 1) { throw std::invalid_argument("usage: strata-sort-tests [fixture directory]"); }
        for (const std::size_t count : {0, 1, 2, 3, 4, 6, 7, 12, 25, 48, 101, 384}) {
            check_shape(count, 1); check_shape(count, 7);
        }
        check_shape(25, 4, true);
        check_preflight_and_moves(); check_fan_in_and_duplicate_values(); check_faults(); check_late_and_cleanup();
        check_caps_and_allocations(); check_pipeline(); check_abandonment_and_shared_pressure();
        std::cout << "external sort: all checks passed\n";
        return 0;
    } catch (const std::exception& error) {
        std::cerr << "external sort: " << error.what() << '\n'; return 1;
    }
}
```

Create `scripts/check_sort_fixture.py`. This script uses Python's `struct` and
`zlib`, not the C++ codec, to inspect actual files. Loading these tiny test images
into memory is an oracle convenience; the engine reader remains block-bounded.

The decoder checks the independently derived exact file length before walking
frames. It requires canonical full blocks except the last, checks every CRC,
then checks record order and the footer's counts. A checksum alone would not
prove that the run contains the intended rows.

```python title="scripts/check_sort_fixture.py — complete file, segment 1"
#!/usr/bin/env python3
"""Decode actual spill files independently of the C++ implementation."""
from pathlib import Path
import struct
import subprocess
import sys
import tempfile
import zlib


def need(condition: bool, message: str) -> None:
    if not condition:
        raise AssertionError(message)


def key(row: tuple[int, int, int, int]) -> tuple[int, int, int, int]:
    event, account, day, cents = row
    return account, event, day, cents


def decode(path: Path) -> list[tuple[int, int, int, int]]:
    image = path.read_bytes()  # Oracle files are deliberately tiny, outside engine accounting.
    need(len(image) >= 100 and image[:8] == b"STRRUN01", "header magic/length")
    version, width, block, order, count, checksum = struct.unpack_from("<IIIIQI", image, 8)
    need((version, width, order) == (1, 28, 1) and block == 2 and 0 < count <= 25, "header fields")
    need(zlib.crc32(image[:32]) == checksum, "header CRC")
    expected_blocks = (count + block - 1) // block
    need(len(image) == 60 + 28 * count + 12 * expected_blocks, "forecast length")
    rows = []
    at = 36
    while len(rows) < count:
        size, payload = struct.unpack_from("<II", image, at)
        need(size == min(block, count - len(rows)) and payload == 28 * size, "frame lengths")
        end = at + 8 + payload
        need(zlib.crc32(image[at:end]) == struct.unpack_from("<I", image, end)[0], "frame CRC")
        for offset in range(at + 8, end, 28):
            rows.append(struct.unpack_from("<QQIQ", image, offset))
        at = end + 4
    need(image[at:at + 8] == b"STREND01", "footer magic")
    tail_count, blocks, crc = struct.unpack_from("<QII", image, at + 8)
    need(tail_count == count and blocks == expected_blocks and at + 24 == len(image), "footer fields")
    need(zlib.crc32(image[at:at + 20]) == crc, "footer CRC")
    need(rows == sorted(rows, key=key), "run order")
    return rows
```

IDs 1 through 9 are initial runs. IDs 10 through 13 merge adjacent initial pairs;
run 9 is carried forward unchanged. IDs 14 and 15 merge those four replacements,
and ID 16 merges 14 and 15. Final output merges 16 with the carried run 9.

The harness verifies each relationship using decoded values, then adds actual
file lengths to check the 4,312-byte write counter. It does not merely compare a
second formula with the first formula. Completed-file copies remain test artifacts
and are removed by Python's temporary-directory owner.

```python title="scripts/check_sort_fixture.py — complete file, segment 2"
def main() -> None:
    if len(sys.argv) != 2:
        raise SystemExit("usage: check_sort_fixture.py path/to/strata-sort-tests")
    executable = str(Path(sys.argv[1]).resolve())
    with tempfile.TemporaryDirectory(prefix="strata-sort-oracle-") as temporary:
        root = Path(temporary)
        result = subprocess.run([executable, "fixture", str(root)], check=True,
                                capture_output=True, text=True)
        metrics = dict(part.split("=") for part in result.stdout.split())
        need(metrics == {"initial_runs": "9", "merge_passes": "3", "writes": "4312", "reads": "7180"},
             "observed transfer/pass counters")
        expected = [(i * 7 % 11, i * 5 % 4, i % 3, 100 + i) for i in range(25)]
        files = sorted(root.glob("*.run"), key=lambda p: int(p.stem))
        need([int(p.stem) for p in files] == list(range(1, 17)), "run IDs")
        decoded = {int(path.stem): decode(path) for path in files}
        for index in range(9):
            need(decoded[index + 1] == sorted(expected[index * 3:(index + 1) * 3], key=key),
                 "initial run rows")
        for output, inputs in {10: (1, 2), 11: (3, 4), 12: (5, 6), 13: (7, 8),
                               14: (10, 11), 15: (12, 13), 16: (14, 15)}.items():
            need(decoded[output] == sorted(decoded[inputs[0]] + decoded[inputs[1]], key=key),
                 "materialized merge rows")
        need(sorted(decoded[16] + decoded[9], key=key) == sorted(expected, key=key), "final merge inputs")
        need(sum(path.stat().st_size for path in files) == 4312, "actual completed file bytes")
    print("native spill oracle: 16 files, nine initial runs, three passes, all rows/CRCs verified")


if __name__ == "__main__":
    main()
```

## Build and run the cumulative checks

Append the following targets to the existing root `CMakeLists.txt`. Python was
already discovered by earlier chapters; do not add a second incompatible Python
configuration or remove prerequisite tests. No third-party sort, queue, or storage
library has been introduced.

```cmake title="CMakeLists.txt — complete file, segment 1"
target_sources(strata-core PRIVATE src/query/external_sort.cpp)
add_executable(strata-sort-tests tests/external_sort.cpp)
target_link_libraries(strata-sort-tests PRIVATE strata-core)
target_compile_options(strata-sort-tests PRIVATE -Wall -Wextra -Wpedantic)
add_test(NAME external-sort COMMAND strata-sort-tests)
add_test(NAME external-sort-native
    COMMAND ${Python3_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_sort_fixture.py
            $<TARGET_FILE:strata-sort-tests>)
```

From the learner source root, configure separate build directories. The compiler
must support the already-used C++20 features; no new tool version gate is added.
On the target Mac, inspect the selected native compiler and SDK as in Chapter 1.
Equivalent maintained toolchains are acceptable.

```sh
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_COMPILER=clang++
cmake --build build/debug -j 4
ctest --test-dir build/debug --output-on-failure
./build/debug/strata-sort-tests
python3 scripts/check_sort_fixture.py ./build/debug/strata-sort-tests

cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_COMPILER=clang++
cmake --build build/release -j 4
ctest --test-dir build/release --output-on-failure
```

The complete tree now has **22 CTest entries**. The normal sort program should
finish with `external sort: all checks passed`. Its 25-row trace must report nine
initial runs, three materialized passes, fan-in two, 4,312 write bytes, and 7,180
read bytes. The native oracle should report sixteen files and verified rows/CRCs.
The fixed object charge and allocation-failure count can differ with the compiler,
standard library, and ABI; the tests derive their needed allowance rather than
hard-code those values.

The Linux authoring run with Clang 17/libstdc++ observed:

```text
rows=25 initial_runs=9 merge_passes=3 fan_in=2 writes=4312 reads=7180 peak_disk=1912 fixed_charge=14984 peak_workspace=31572
allocation_failures=34
external sort: all checks passed
native spill oracle: 16 files, nine initial runs, three passes, all rows/CRCs verified
```

The 31,572-byte peak is the **sort-local model**, including the 16-KiB stack
allowance. It is not peak RSS or a measured stack high-water mark. Other operators,
output columns, the report, storage readers, and allocator implementation overhead
are not all represented by that one number. The root ledger still charges its
covered allocation requests across operators.

For the sanitizer build, use separate compiled objects and run the whole suite:

```sh
cmake -S . -B build/sanitize -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build/sanitize -j 4
ctest --test-dir build/sanitize --output-on-failure
```

All 22 tests passed in Debug, Release, and combined AddressSanitizer /
UndefinedBehaviorSanitizer configurations in the Linux authoring environment.
Native M4/macOS execution has not been verified. The temporary format intentionally
does not request durable synchronization: these checks establish neither query
restartability nor physical power-loss durability. An abrupt process termination
may leave a private temporary directory; this chapter does not implement a janitor.

## Break the heap, not the allocator

Temporarily remove only `sift_down();` from `Core::take`. Keep the copy-before-advance
and reader-lifetime code intact. Rebuild and run `strata-sort-tests` in all three
configurations. The authoring runs exited with status 1 and:

```text
external sort: external sort differs from independent oracle
```

There was no sanitizer memory-error report. The pointers and bounds can be legal
while the returned sequence violates ordering. For a two-reader example, heads
`(account 1, event 5)` and `(account 2, event 3)` initially choose reader zero.
After reader zero advances to `(account 9, event 7)`, failing to repair the heap
would choose account 9 before account 2. The invariant concerns **current** heads,
not the initial insertion order.

Restore the call and rebuild before continuing. The canonical listings contain
the correct implementation. Do not leave this diagnostic edit in a learner branch
that serves as the next chapter's prerequisite.

## Exercises and diagnosis

1. With 25 fixture rows and block capacity two, raise the descriptor allowance and
   choose a workspace that admits four inputs. Predict `9 -> 3` before running;
   compare pass count and transfers, not elapsed time from one tiny invocation.
   Then reduce the workspace by one byte below the two-input minimum and confirm
   rejection before any file is created. The complete test above is the executable
   acceptance criterion.
2. Keep unlink refused through destruction while retaining the report. Observe a
   nonzero cleanup-error count and the exact private directory path. Perform any
   manual removal only after the owner is gone and only inside that test directory.
   A zero memory charge is not evidence that filesystem cleanup succeeded. The
   ordinary test removes the refusal and retries so it leaves no deliberate debris.
3. Explain why replacing the stored row count or block capacity with a file-provided
   allocation size would undermine the workspace proof. Point to the checks against
   the in-memory run descriptor and configured block capacity, then repeat the
   checksummed huge-header case. The expected failure is validation, not an attempted
   enormous allocation.

An unexpected `sort workspace cannot make progress` means the configured local
allowance cannot hold fixed state plus either generation buffers or a two-way
merge. `BudgetExceeded` after a previously valid local plan instead suggests other
charged owners consume the shared allowance. A disk-cap failure after initial runs
succeed means replacement coexistence was not budgeted. A CRC error at offset 104
is not EOF, and a cleanup error is not permission to silently zero the disk ledger.
Inspect the first causal error before changing limits.

## References and the next boundary

The implementation uses the existing explicit codec, file boundary, and charged
allocator chapters. Relevant external contracts read for this chapter are the
C++ working draft's [ordering requirements](https://eel.is/c++draft/alg.sorting),
[`sort` complexity](https://eel.is/c++draft/sort),
[object construction](https://eel.is/c++draft/specialized.construct),
[destruction](https://eel.is/c++draft/specialized.destroy),
[member initialization order](https://eel.is/c++draft/class.base.init), and
[shared allocation](https://eel.is/c++draft/util.smartptr.shared.create).
The draft is a living reference, not a promise that all newer facilities are
available in C++20; this implementation uses only the named C++20 facilities.
Apple's archived [`mkdtemp` manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/mkdtemp.3.html)
and [`unlink` manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/unlink.2.html)
explain private-directory creation and the distinction between a name and an open
file. The [OpenBSD `open`/`openat` manual](https://man.openbsd.org/open.2) and
[`unlink`/`unlinkat` manual](https://man.openbsd.org/unlink.2) supply additional
concrete system-call contracts; they are not evidence of execution on macOS.

The exit criterion is a correctly ordered, owned event stream that stays within
its explicit workspace/run/descriptor limits, spills through multiple passes,
and makes cleanup failures visible. Keep all values needed by the next chapter:
account ID for matching, event ID for deterministic ordering, day, and cents.
Chapter 16 will implement the many-to-one merge join; Chapter 17 will add grouped
reduction. This chapter has not added those operators, parallel execution, or
cancellation checks.
