# Chapter 21 — Reproducible workloads and independent answers

## Part 2 — Parse bounded input; drive the existing engine

The input boundary should be smaller than the engine it drives. We need no
SQL parser, general CSV importer, or network service. We need strict unsigned
fields, a bounded batch buffer, and a clear answer to this question: after a
failure, which batches might already be committed?

**Prerequisite code:** Chapter 20's complete source state plus the generator
and reference from [Part 1](https://25d-228.github.io/cplusplus-learning/chapter-21.html).
All files below are new. `Engine::write` takes a borrowed mutation span and
finishes its preparation and commit before returning. `Executor` accepts an
owning request and returns a handle whose terminal result contains no running
query owner. We reuse these contracts rather than introducing a second writer
or bypassing cancellation in the driver.

## One buffer owns every input borrow

`Input` owns the file stream, a 255-byte line array, six token views, and a fixed
array of 256 `Mutation` values. `tokens()` returns views into the line array.
The next call overwrites them, so the parser converts every field into an owning
numeric mutation before reading another line. `next()` returns a span into the
mutation array; the importer must finish the write before asking for another
batch.

The stream's buffering, path string, diagnostic strings, and fixed adapter
arrays are host-side costs, not engine-budget charges. Their bound is useful,
but it does not turn the ledger into a process-memory cap. In particular, the
standard stream may allocate its own implementation-specific buffer.

```cpp title="include/strata/workload/input.hpp"
#ifndef STRATA_WORKLOAD_INPUT_HPP
#define STRATA_WORKLOAD_INPUT_HPP
#include "strata/storage/memtable.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <fstream>
#include <span>
#include <string>
#include <string_view>

namespace strata::workload {
std::uint64_t unsigned_number(std::string_view text);
class Input {
public:
    explicit Input(const std::string& path);
    // Borrows this object's fixed batch until the next next() or destruction.
    std::span<const storage::Mutation> next();
    std::uint64_t seed() const noexcept { return seed_; }
    std::uint64_t sequence() const noexcept { return sequence_; }
    std::uint64_t operations() const noexcept { return operations_; }
    std::uint64_t logical_bytes() const noexcept { return logical_; }
    std::size_t line_number() const noexcept { return line_; }
private:
    std::span<const std::string_view> tokens();
    [[noreturn]] void bad(std::string_view reason) const;
    storage::Mutation operation(std::span<const std::string_view> words);
    std::string path_;
    std::ifstream stream_;
    std::array<char, 255> line_bytes_{};
    std::array<std::string_view, 6> words_{};
    std::array<storage::Mutation, storage::MemTable::max_batch> batch_{};
    std::uint64_t seed_ = 0, sequence_ = 0, operations_ = 0, logical_ = 0;
    std::size_t line_ = 0;
    bool ended_ = false;
};
}  // namespace strata::workload
#endif
```

`unsigned_number` uses integer `std::from_chars`. Its result has a pointer to
the first unconsumed character and an error code. Structured binding names those
two members `end` and `error`. Both must be checked: parsing the numeric prefix
of `12tail` is not a successful field conversion. The
[integer conversion contract](https://eel.is/c++draft/charconv.from.chars)
specifies out-of-range and invalid-input results and does not throw; our wrapper
turns rejection into an exception. No null terminator is needed for a view.
The leading-zero rule is additional format policy.

```cpp title="src/workload/input.cpp"
#include "strata/workload/input.hpp"
#include "strata/core/checked.hpp"

#include <charconv>
#include <limits>
#include <stdexcept>
#include <system_error>

namespace strata::workload {
std::uint64_t unsigned_number(std::string_view text) {
    std::uint64_t value = 0;
    if (text.empty() || (text.size() > 1 && text.front() == '0')) {
        throw std::invalid_argument("noncanonical unsigned integer");
    }
    const auto [end, error] = std::from_chars(text.data(), text.data() + text.size(), value);
    if (error != std::errc{} || end != text.data() + text.size()) {
        throw std::invalid_argument("invalid/out-of-range unsigned integer");
    }
    return value;
}
[[noreturn]] void Input::bad(std::string_view reason) const {
    throw std::runtime_error(path_ + ":" + std::to_string(line_) + ": " + std::string{reason});
}
```

Keep `stream_.get()` in its integer return type. Narrowing it to `char` before
checking EOF could lose the distinction between a character and the sentinel.
Check the content bound before writing the next byte, and reject controls,
non-ASCII bytes, tabs, and carriage returns. Thus CRLF is intentionally not an
alternate serialization of this format.

Tokenization returns at least one nonempty field or throws. Six views are enough
for the five-field event form and a bounded rejection of excess fields. No token
can point into a temporary `std::string`: every view points into `line_bytes_`.

```cpp title="src/workload/input.cpp"
std::span<const std::string_view> Input::tokens() {
    ++line_;
    std::size_t size = 0;
    for (;;) {
        const auto next = stream_.get();
        if (next == std::char_traits<char>::eof()) {
            bad(stream_.bad() ? "input I/O failure" : "missing or unterminated line");
        }
        if (next == '\n') { break; }
        if (size == line_bytes_.size()) { bad("line exceeds 255 content bytes"); }
        if (next < 32 || next > 126) { bad("only printable ASCII and LF are permitted"); }
        line_bytes_[size++] = static_cast<char>(next);
    }
    std::string_view line{line_bytes_.data(), size};
    std::size_t count = 0, begin = 0;
    for (;;) {
        const auto space = line.find(' ', begin);
        const auto end = space == std::string_view::npos ? line.size() : space;
        if (end == begin || count == words_.size()) { bad("empty or excessive fields"); }
        words_[count++] = line.substr(begin, end - begin);
        if (space == std::string_view::npos) { break; }
        begin = space + 1;
    }
    return {words_.data(), count};
}
Input::Input(const std::string& path) : path_(path), stream_(path, std::ios::binary) {
    if (!stream_) { bad("cannot open input"); }
    const auto words = tokens();
    if (words.size() != 3 || words[0] != "STRATA1") { bad("workload version/header"); }
    seed_ = unsigned_number(words[1]); sequence_ = unsigned_number(words[2]);
}
```

The constructor consumes the header once. An exception closes its already
constructed stream member automatically. During operation decoding, the small
`narrow` lambda checks the range **before** converting to 32 bits. A cast alone
would not reject an oversized day or region.

Puts and tombstones are different mutation kinds. In particular, an account
tombstone must not become `Account{id, 0}`: region zero is valid, and that
mistake would retain join matches under a new region instead of deleting them.

```cpp title="src/workload/input.cpp"
storage::Mutation Input::operation(std::span<const std::string_view> words) {
    const auto narrow = [](std::string_view text) {
        const auto value = unsigned_number(text);
        if (value > std::numeric_limits<std::uint32_t>::max()) {
            throw std::out_of_range("32-bit field");
        }
        return static_cast<std::uint32_t>(value);
    };
    if (words[0] == "E" && words.size() == 5) {
        logical_ = checked_u64_add(logical_, 30);
        return storage::Mutation::put(Event{unsigned_number(words[1]), unsigned_number(words[2]),
                                            narrow(words[3]), unsigned_number(words[4])});
    }
    if (words[0] == "A" && words.size() == 3) {
        logical_ = checked_u64_add(logical_, 14);
        return storage::Mutation::put(Account{unsigned_number(words[1]), narrow(words[2])});
    }
    if (words[0] == "X" && words.size() == 3 && (words[1] == "A" || words[1] == "E")) {
        logical_ = checked_u64_add(logical_, 9);
        const auto relation = words[1] == "A" ? Relation::accounts : Relation::events;
        return storage::Mutation::erase({relation, unsigned_number(words[2])});
    }
    bad("unknown operation or field count");
}
```

`next()` saves the batch sequence and count as integer values before reading
operations. Saving token views instead would leave them referring to the next
line. The duplicate scan is quadratic in batch size, but the size is at most
256: at most $256\cdot255/2=32640$ key comparisons. This avoids an uncharged
hash table and keeps the boundary deterministic. It is a deliberate small-batch
tradeoff, not a proposed algorithm for an unbounded import batch.

```cpp title="src/workload/input.cpp"
std::span<const storage::Mutation> Input::next() {
    if (ended_) { return {}; }
    const auto words = tokens();
    if (words[0] == "END") {
        if (words.size() != 4 || unsigned_number(words[1]) != sequence_ ||
            unsigned_number(words[2]) != operations_ || unsigned_number(words[3]) != logical_) {
            bad("END totals differ");
        }
        if (stream_.get() != std::char_traits<char>::eof() || stream_.bad()) {
            bad("bytes or I/O error after END");
        }
        ended_ = true; return {};
    }
    if (words.size() != 3 || words[0] != "B") { bad("expected batch header"); }
    const auto sequence = unsigned_number(words[1]), count = unsigned_number(words[2]);
    if (sequence != checked_u64_add(sequence_, 1) || count == 0 || count > batch_.size()) {
        bad("batch sequence/count");
    }
    for (std::size_t i = 0; i < count; ++i) {
        batch_[i] = operation(tokens());
        for (std::size_t j = 0; j < i; ++j) {
            if (batch_[j].key == batch_[i].key) { bad("duplicate key in batch"); }
        }
    }
    const auto commit = tokens();
    if (commit.size() != 2 || commit[0] != "C" || unsigned_number(commit[1]) != sequence) {
        bad("missing matching batch terminator");
    }
    operations_ = checked_u64_add(operations_, count);
    sequence_ = sequence;
    return {batch_.data(), static_cast<std::size_t>(count)};
}
}  // namespace strata::workload
```

The `END` case checks totals and then requires EOF. The successful empty span
therefore means a validated end of input, not an empty batch. After a parsing
exception, discard the `Input` object; its running counters may include part of
the rejected batch. The engine has not seen that batch because `next()` has not
returned it. No parser-level rollback is needed to protect the database.

For a useful failure trace, place a valid sequence-1 batch before
`B 2 1`, a valid event, and `C 99`. Sequence 1 can commit. Sequence 2 never reaches
`Engine::write`; the adapter reports its line number and the importer reports
`last_acknowledged=1`. Reopening must return sequence 1's answer. A missing final
`END` is different: all preceding terminated batches might already be committed.

## Presets should force mechanisms without hiding limits

A preset is an explicit configuration, not a promise to accept arbitrarily large
input. The `tiny` preset keeps batch capacity, spill chunk size, fan-in, cache,
and table blocks small. The `stream` preset raises workspace granularity but
preserves the existing maximum of 128 initial run descriptors per sort.

Let $C$ be sort chunk rows, $R$ maximum initial runs, and $n$ selected input rows
for that sort. Run generation needs $\lceil n/C\rceil$ descriptors. Its current
bound is therefore

$$
\lceil n/C\rceil\le R,\qquad n\le CR.
$$

For `tiny`, $C=3$ and $R=128$, allowing at most 384 selected rows. For `stream`,
$C=2048$, giving 262144. These are **per-sort row limits**, not just byte limits.
Input exceeding them is rejected; do not describe this implementation as an
unbounded external sort. A broader disk-backed run catalog is outside this
chapter. The test workload is chosen to exercise multiple passes within the
implemented bound.

With two input readers and one output writer, the tiny merge allowance is three
file descriptors, so fan-in is two when memory also fits. Twenty initial runs
then shrink as

$$
20\rightarrow10\rightarrow5\rightarrow3\rightarrow2.
$$

Those are four materialized passes. The final two-way merge streams to the next
operator rather than writing one more run. The stream preset allows eight input
readers and one output writer. Ninety-eight initial runs become 13 and then 2:
two materialized passes, followed by the streaming final merge.

The adapter declarations gather presets, import, result checking, and scenario
entry points without adding another core abstraction.

```cpp title="include/strata/workload/driver.hpp"
#ifndef STRATA_WORKLOAD_DRIVER_HPP
#define STRATA_WORKLOAD_DRIVER_HPP
#include "strata/workload/input.hpp"
#include "strata/runtime/engine.hpp"
#include "strata/runtime/executor.hpp"

#include <filesystem>
#include <string_view>

namespace strata::workload {
struct Preset {
    runtime::EngineOptions engine;
    query::QueryOptions query;
    std::size_t query_bytes = 16 * 1024 * 1024;
    std::uint64_t space_floor = 768 * 1024 * 1024;
};
Preset preset(std::string_view name, query::DayRange days = {0, std::uint64_t{1} << 32});
void preflight(const std::filesystem::path& directory, std::uint64_t space_floor);
void apply(runtime::Engine& engine, Input& input);
void settled(runtime::Engine& engine, bool compact);
void print_engine(const runtime::EngineStats& stats);
void print_result(std::string_view label, const runtime::TaskResult& result);
runtime::TaskResult wait_result(const runtime::TaskHandle& handle);
void require_completed(const runtime::TaskResult& result);
void scenario(const std::filesystem::path& root, const std::string& base,
              const std::string& delta, query::DayRange days);
}  // namespace strata::workload
#endif
```

The preset implementation follows. The stream case retains Chapter 20's default
storage and maintenance options. Tiny maintenance has its own 4 MiB allowance,
separate from each 2 MiB query reservation. That separation allows maintenance to
finish while a held query consumes its entire reserved slot.

```cpp title="src/workload/driver.cpp"
#include "strata/workload/driver.hpp"

#include <cerrno>
#include <chrono>
#include <iostream>
#include <limits>
#include <stdexcept>
#include <system_error>
#include <sys/resource.h>

namespace strata::workload {
Preset preset(std::string_view name, query::DayRange days) {
    Preset p; p.query.days = days;
    // Both presets remain below the 128 initial-run descriptor cap.
    for (auto* sort : {&p.query.event_sort, &p.query.group_sort}) {
        sort->chunk_rows = 2048; sort->block_rows = 128; sort->file_descriptors = 9;
    }
    if (name == "stream") { return p; }
    if (name != "tiny") { throw std::invalid_argument("preset must be tiny or stream"); }
    p.engine.storage.arena_bytes = 256 * 1024;
    p.engine.storage.cache_slots = 2; p.engine.storage.cache_bytes = 256;
    p.engine.storage.table = {128, 1024, 64, 1024 * 1024};
    p.engine.storage_bytes = 32 * 1024 * 1024;
    p.engine.batches_per_log = 1; p.engine.compact_after = 4;
    p.engine.compaction.output = p.engine.storage.table;
    p.engine.compaction.max_output_files = 4;
    p.engine.compaction.max_output_bytes = 4 * 1024 * 1024;
    p.engine.maintenance_bytes = 4 * 1024 * 1024; p.engine.maintenance_files = 21;
    p.query.upstream_rows = 2; p.query.join = {2, 2, 4};
    p.query.joined_rows = 2; p.query.aggregate = {2, 4}; p.query.output_rows = 1;
    for (auto* sort : {&p.query.event_sort, &p.query.group_sort}) {
        sort->chunk_rows = 3; sort->block_rows = 2; sort->file_descriptors = 3;
        sort->workspace_bytes = 128 * 1024; sort->stack_allowance = 16 * 1024;
        sort->disk_bytes = 256 * 1024;
    }
    p.query_bytes = 2 * 1024 * 1024; p.space_floor = 64 * 1024 * 1024;
    return p;
}
```

Important limits are visible rather than inferred from the machine's 16 GB:

| Setting | Tiny | Stream |
|---|---:|---:|
| Rows per sort chunk | 3 | 2048 |
| Rows per spill block | 2 | 128 |
| Sort descriptor allowance | 3 | 9 |
| Maximum initial runs | 128 | 128 |
| Per-query reservation | 2 MiB | 16 MiB |
| Required currently available disk | 64 MiB | 768 MiB |

The tiny 256 KiB per-sort spill allowance covers the intended small scenario,
including input/output coexistence during replacement. Stream keeps the existing
256 MiB per-sort disk allowance. Neither preset's free-space check reserves that
space. Table metadata bounds, log slots, retained snapshots, and other engine
limits still apply independently of these numbers.

## Preflight observes; admission owns

`filesystem::space(...).available` reports currently available space; an unknown
value is represented by `uintmax_t(-1)`. See the
[filesystem space contract](https://eel.is/c++draft/fs.op.space).
The driver rejects unknown or insufficient availability, but another process can
consume disk immediately afterward. Runtime write errors still matter.

For descriptors, read the soft `RLIMIT_NOFILE` value with `getrlimit`. Apple's
[archived manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/getrlimit.2.html)
describes the call, `rlim_cur`, and the open-file limit; the installed
`man 2 getrlimit` is the place to inspect the target system's details. This
adapter requires at least 256 as explicit headroom, not as an asserted macOS
default. It does not change either limit. Query admission still applies its
separate conservative descriptor allowance.

```cpp title="src/workload/driver.cpp"
void preflight(const std::filesystem::path& directory, std::uint64_t space_floor) {
    if (!std::filesystem::is_directory(directory)) { throw std::invalid_argument("preflight needs an existing directory"); }
    const auto available = std::filesystem::space(directory).available;
    if (available == std::numeric_limits<std::uintmax_t>::max() || available < space_floor) {
        throw std::runtime_error("free-space preflight failed");
    }
    rlimit limit{};
    if (::getrlimit(RLIMIT_NOFILE, &limit) != 0) {
        throw std::system_error(errno, std::generic_category(), "getrlimit");
    }
    if (limit.rlim_cur != RLIM_INFINITY && limit.rlim_cur < 256) {
        throw std::runtime_error("descriptor preflight needs a soft limit of at least 256");
    }
    std::cout << "preflight directory=" << directory << " available_bytes=" << available
              << " required_available=" << space_floor << " descriptor_floor=256\n";
}
```

A preflight failure is a configuration diagnosis, not evidence of a core
algorithm failure. Inspect `ulimit -n` and the selected directories; choose a
permitted limit or a different filesystem explicitly. Do not raise a process
limit blindly or interpret an admission estimate as a kernel reservation.

## A successful batch advances one acknowledged boundary

The importer first compares the database's current sequence with the file's base.
It then alternates `next()` and `write()`, so no future batch overwrites the
borrowed mutations while a write is using them. A committed result must have
the sequence declared by the input.

A definitely rejected batch and an uncertain write both stop import. For an
uncertain result, reopen before deciding what is present: the log might contain
a complete batch whose caller did not receive success. Do not skip ahead based
only on the last acknowledged value. The driver intentionally supplies no
automatic retry, deduplication, or partial-file resumption command.

```cpp title="src/workload/driver.cpp"
void apply(runtime::Engine& engine, Input& input) {
    auto acknowledged = engine.stats().sequence;
    std::cout << "import seed=" << input.seed() << " expected_base=" << input.sequence()
              << " actual_base=" << acknowledged << '\n';
    try {
        if (acknowledged != input.sequence()) { throw std::runtime_error("base-sequence mismatch; no automatic replay"); }
        for (;;) {
            const auto changes = input.next();
            if (changes.empty()) { break; }
            const auto result = engine.write(changes);
            if (result.outcome != storage::WriteOutcome::committed) {
                throw std::runtime_error(std::string{result.outcome == storage::WriteOutcome::uncertain
                    ? "uncertain write; reopen before deciding anything" : "rejected batch"} +
                    "; candidate=" + std::to_string(result.sequence) +
                    "; reason_code=" + std::to_string(static_cast<int>(result.reason)));
            }
            acknowledged = result.sequence;
            if (acknowledged != input.sequence()) { throw std::logic_error("commit sequence differs from input"); }
        }
    } catch (const std::exception& error) {
        throw std::runtime_error("import seed=" + std::to_string(input.seed()) +
            " line=" + std::to_string(input.line_number()) +
            " last_acknowledged=" + std::to_string(acknowledged) + ": " + error.what());
    }
    std::cout << "import complete seed=" << input.seed() << " sequence=" << acknowledged
              << " operations=" << input.operations() << " logical_bytes=" << input.logical_bytes() << '\n';
}
```

The exception wrapper records the seed, input line, and last acknowledged
sequence. It does not claim that all later batches are absent after an uncertain
write. The seed helps reproduce the fixture; the sequence identifies the
boundary whose success the caller actually observed.

`settled` seals the active data and waits for maintenance. An explicit compaction
request then runs through the existing worker. Collection is also explicit.
These are ordinary graceful operations, not fault injection or a physical
power-loss test. Counters returned by `stats()` are copied under the engine's
coordinator lock.

```cpp title="src/workload/driver.cpp"
void settled(runtime::Engine& engine, bool compact) {
    (void)engine.seal(); engine.await_idle();
    if (compact) { engine.request_compaction(); engine.await_idle(); }
    engine.request_collection(); engine.await_idle();
}
void print_engine(const runtime::EngineStats& s) {
    std::cout << "engine sequence=" << s.sequence << " epoch=" << s.epoch
              << " logs=" << s.logs << " tables=" << s.tables << " flushes=" << s.flushes
              << " compactions=" << s.compactions << " retained_states=" << s.retained.snapshots
              << " storage_used=" << s.storage_used << " maintenance_peak=" << s.maintenance_peak
              << " last_job_read=" << s.last_job.merge_read_bytes
              << " last_job_write=" << s.last_job.output_write_bytes
              << " last_collection_removed=" << s.cleanup.removed << '\n';
}
void print_result(std::string_view label, const runtime::TaskResult& r) {
    std::cout << label << " sequence=" << r.sequence << " groups=" << r.staged_groups
              << " event_input=" << r.events.input_rows << " join_rows=" << r.groups.input_rows
              << " event_runs=" << r.events.initial_runs << " group_runs=" << r.groups.initial_runs
              << " event_passes=" << r.events.merge_passes << " group_passes=" << r.groups.merge_passes
              << " peak_charged=" << r.peak_bytes << " final_charged=" << r.final_bytes
              << " event_write=" << r.events.write_bytes << " group_write=" << r.groups.write_bytes << '\n';
}
```

The printed labels intentionally distinguish current-process counters from
persistent state. `sequence` comes from the database history. `flushes` and
`compactions` count work in this engine object's lifetime; reopening does not
reconstruct historical totals. `last_job_read` and `last_job_write` describe
one maintenance job, not all compaction traffic. Query `peak_charged` is the
ledger peak, not resident memory; `final_charged=0` means those charged owners
were released. Streams, paths, thread stacks, allocator bookkeeping, and the
operating system remain outside that statement.

A terminal task can report completion while also reporting a cleanup problem.
`require_completed` checks both. If publication succeeded but removing a spill
file failed, the answer may be valid while the command exits with a cleanup
error. Do not replace that diagnostic with an unconditional success message.

```cpp title="src/workload/driver.cpp"
runtime::TaskResult wait_result(const runtime::TaskHandle& handle) {
    // A diagnostic timeout, not a performance target or a scheduling primitive.
    const auto result = handle.wait_for(std::chrono::minutes{5});
    if (!result) { throw std::runtime_error("query watchdog expired"); }
    return *result;
}
void require_completed(const runtime::TaskResult& r) {
    if (r.state != runtime::TaskState::completed) {
        if (r.error) { std::rethrow_exception(r.error); }
        throw std::runtime_error("query did not complete");
    }
    if (r.final_bytes || r.events.live_files || r.groups.live_files ||
        r.events.directory_live || r.groups.directory_live || r.events.cleanup_errors ||
        r.groups.cleanup_errors || r.result_cleanup_error) {
        throw std::runtime_error("answer published, but resource cleanup is incomplete");
    }
}
}  // namespace strata::workload
```

The five-minute wait is a watchdog, not synchronization or a performance target.
On timeout the caller unwinds through the executor owner, which requests stop
and joins. Cooperative cancellation cannot instantly interrupt a blocked file
operation. The scenario's ordering will use explicit gates, not elapsed time.

## Dispatch commands without reimplementing the engine

The first half of `main` validates input or imports it. `load` requires a new
database directory; `apply` recovers an existing one and checks the input base.
The input header is opened before creating a new database, avoiding an empty
database for an immediately invalid header. A later failure deliberately leaves
the directory available for inspection. Never remove a possibly committed
prefix merely because the command returned an error.

```cpp title="apps/strata.cpp"
#include "strata/workload/driver.hpp"

#include <exception>
#include <filesystem>
#include <iostream>
#include <stdexcept>
#include <string_view>

int main(int argc, char** argv) {
    using namespace strata;
    using namespace strata::workload;
    namespace fs = std::filesystem;
    try {
        const std::string_view command = argc > 1 ? argv[1] : "";
        if (command == "validate" && argc == 3) {
            Input input{argv[2]}; while (!input.next().empty()) {}
            std::cout << "validated seed=" << input.seed() << " sequence=" << input.sequence()
                      << " operations=" << input.operations() << " logical_bytes=" << input.logical_bytes() << '\n';
            return 0;
        }
        if (command == "preflight" && argc == 4) { preflight(argv[2], unsigned_number(argv[3])); return 0; }
        if (command == "scenario" && argc == 7) {
            scenario(argv[2], argv[3], argv[4], {unsigned_number(argv[5]), unsigned_number(argv[6])}); return 0;
        }
        if ((command == "load" || command == "apply") && argc == 5) {
            const auto p = preset(argv[4]); Input input{argv[3]}; const fs::path directory{argv[2]};
            const auto parent = directory.has_parent_path() ? directory.parent_path() : fs::path{"."};
            preflight(command == "load" ? parent : directory, p.space_floor);
            if (command == "load" && !fs::create_directory(directory)) {
                throw std::runtime_error("load requires a new database directory");
            }
            runtime::Engine engine{directory.string(), command == "load" ? storage::DatabaseOpen::create :
                storage::DatabaseOpen::recover, p.engine};
            apply(engine, input); settled(engine, false); print_engine(engine.stats()); engine.finish(); return 0;
        }
```

`inspect` is an ordinary recovery open followed by manifest observations. It is
**not** a forensic no-write mode: recovery can repair a permitted active-log
tail and synchronize. `compact` uses `Engine` so it follows the same maintenance
path as the concurrent scenario. Neither command bypasses the directory lock.

```cpp title="apps/strata.cpp"
        if ((command == "inspect" || command == "compact") && argc == 4) {
            const auto p = preset(argv[3]); preflight(argv[2], p.space_floor);
            if (command == "compact") {
                runtime::Engine engine{argv[2], storage::DatabaseOpen::recover, p.engine};
                settled(engine, true); print_engine(engine.stats()); engine.finish();
            } else {
                auto budget = std::make_shared<memory::Budget>(p.engine.storage_bytes);
                storage::Database database{budget, argv[2], storage::DatabaseOpen::recover, p.engine.storage};
                const auto& m = database.manifest();
                std::cout << "inspect sequence=" << database.sequence() << " epoch=" << m.epoch
                          << " logs=" << m.log_count << " tables=" << m.table_count
                          << " orphans=" << database.orphans_at_open() << " charged=" << budget->used() << '\n';
            }
            return 0;
        }
```

`query` uses one worker and one queue slot. It copies the handle before
submission transfers request ownership. All paths, output naming, query
admission, cancellation semantics, CSV staging, and final publication remain
the existing implementations. A rejected submit cannot be disguised as an empty
answer. `finish()` joins before we inspect completion and stop the engine.
The command's day upper bound may be $2^{32}$ even though a stored day is at
most $2^{32}-1$; Chapter 17's `DayRange` uses the wider representation needed
for an exclusive endpoint.

```cpp title="apps/strata.cpp"
        if (command == "query" && argc == 9) {
            const auto p = preset(argv[8], {unsigned_number(argv[6]), unsigned_number(argv[7])});
            for (const auto index : {2, 3, 4}) { preflight(argv[index], p.space_floor); }
            auto engine = std::make_shared<runtime::Engine>(argv[2], storage::DatabaseOpen::recover, p.engine);
            runtime::Executor executor{engine, argv[3], argv[4], {1, 1, p.query_bytes, 128}};
            auto request = std::make_unique<runtime::Request>(p.query, p.query_bytes, argv[5]);
            const auto handle = request->handle();
            if (executor.submit(request) != runtime::Submit::accepted) { throw std::runtime_error("query admission rejected"); }
            const auto result = wait_result(handle); executor.finish();
            print_result("query", result); require_completed(result); engine->finish(); return 0;
        }
        std::cerr << "usage:\n"
          "  strata validate INPUT\n  strata preflight DIRECTORY REQUIRED_AVAILABLE_BYTES\n"
          "  strata load|apply DATABASE INPUT tiny|stream\n"
          "  strata inspect|compact DATABASE tiny|stream\n"
          "  strata query DATABASE TEMP_PARENT OUTPUT_PARENT NAME DAY_BEGIN DAY_END tiny|stream\n"
          "  strata scenario NEW_ROOT BASE_INPUT DELTA_INPUT DAY_BEGIN DAY_END\n";
        return 2;
    } catch (const std::exception& error) {
        std::cerr << "strata command=" << (argc > 1 ? argv[1] : "missing") << ": " << error.what() << '\n';
        return 1;
    }
}
```

## Inspect a complete small history

After appending Part 3's CMake targets and building, these commands run from the
learner source root. `mktemp` gives the run a fresh parent directory; `load`
creates its `db` child. Do not start an additional query process against the same
open writer. The database directory lock intentionally prevents that model.

```sh title="Terminal — import, query, update, compact, and reopen"
run=$(mktemp -d "${TMPDIR:-/tmp}/strata-workload.XXXXXX")
python3 scripts/generate_data.py "$run/input"
mkdir "$run/spill" "$run/out"
build-debug/strata validate "$run/input/base.ops"
build-debug/strata load "$run/db" "$run/input/base.ops" tiny
build-debug/strata query "$run/db" "$run/spill" "$run/out" base.csv 1 5 tiny
python3 scripts/reference_query.py replay "$run/input/base.ops" \
  --sequence 5 --begin 1 --end 5 --actual "$run/out/base.csv"
build-debug/strata apply "$run/db" "$run/input/delta.ops" tiny
build-debug/strata compact "$run/db" tiny
build-debug/strata inspect "$run/db" tiny
build-debug/strata query "$run/db" "$run/spill" "$run/out" delta.csv 1 5 tiny
python3 scripts/reference_query.py replay \
  "$run/input/base.ops" "$run/input/delta.ops" \
  --sequence 6 --begin 1 --end 5 --actual "$run/out/delta.csv"
```

The base query has 60 filtered events, 60 joined rows, and eight groups. The
updated query has 55 filtered and joined rows and twelve groups. Both sorts
make four materialized merge passes in both queries. The latter CSV includes
`2,4,1,1785`, the moved event traced in Part 1. Base `region 1, day 4` instead
contains twelve events totaling 5516 cents. The complete reference compares all
rows; these examples are not a substitute for it.

Running `apply` again on the same delta must fail the base-sequence check and
report `last_acknowledged=6`. It must not replay the updates as new sequences.
A simple absence of a crash is not the expected check.

The final part keeps one process open while these operations overlap, then
verifies that the answers still name coherent committed histories.
