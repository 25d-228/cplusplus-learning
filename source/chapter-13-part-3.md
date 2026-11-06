# Chapter 13 — Streaming compaction and safe reclamation

## Part 3 — Observe retained names, bounded outputs, and restart behavior

The completed [Part 1](https://25d-228.github.io/cplusplus-learning/chapter-13.html)
and [Part 2](https://25d-228.github.io/cplusplus-learning/chapter-13-part-2.html)
are the prerequisites for these checks. All three parts form one chapter with one
**4.0-hour implementation allocation**. The tests add no database library or
execution framework. Their `std::map` is an independent expected-result model,
not the engine's compaction implementation.

The central fixture has five committed batches. The first three are selected;
the fourth table and fifth batch remain newer than the compacted boundary.

| Batch | Changes |
|---|---|
| 1 | Events 0–7, amounts `10 + key`, account 10/day 3; accounts 10→region 1 and 20→region 2 |
| 2 | Delete event 0; event 1→901 cents; account 10→region 3 |
| 3 | Event 2→702 cents; event 8→808 cents/account 20; account 20→region 4 |
| 4 | Reinsert event 0→1,000 cents; delete event 3; account 10→region 5 |
| 5 | Event 9→909 cents/account 20; delete event 4 |

The first three tables contain $10+3+3=16$ stored versions. At boundary 3 there
are eight live events, keys 1–8, and two accounts: ten results. Event 0's selected
tombstone must not resurrect its sequence-1 value. But batch 4's newer reinsertion
must still be visible to a current reader. An old boundary-1 view must continue
seeing the original ten rows. These are three different correct answers, not
three views of a mutable global map.

## Build one explicit test program

Create `tests/compaction.cpp` by concatenating the following segments in order.
The first opens an unnamed namespace and defines the small assertion helpers.
`check` remains active in Release, unlike a disabled `assert`. `expect` catches
only the named exception family; an unrelated exception fails the test.

```cpp title="tests/compaction.cpp — complete file, segment 1"
#include "strata/storage/database.hpp"
#include "wal_file_ops.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <cstdlib>
#include <exception>
#include <filesystem>
#include <fstream>
#include <iostream>
#include <limits>
#include <map>
#include <memory>
#include <span>
#include <stdexcept>
#include <string>
#include <string_view>
#include <system_error>
#include <vector>

namespace {
using namespace strata;
using namespace strata::storage;
void check(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}
template<class Error, class Function>
void expect(const Function& function) {
    try { function(); } catch (const Error&) { return; }
    throw std::runtime_error("expected exception missing");
}
```

A temporary directory owns the whole test namespace and removes it only after
database/view owners in the test scope have been destroyed. This fixture cleanup
is not the production reclamation policy. Tiny 128-byte blocks and two cache
slots force interesting transitions with a small amount of data; output tables
are capped at 300 bytes.

```cpp title="tests/compaction.cpp — complete file, segment 2"
class TemporaryDirectory {
public:
    TemporaryDirectory() {
        char pattern[] = "/tmp/strata-compact-XXXXXX";
        const auto path = ::mkdtemp(pattern);
        if (!path) { throw std::system_error(errno, std::generic_category(), "mkdtemp"); }
        name_ = path;
    }
    ~TemporaryDirectory() noexcept {
        std::error_code ignored;
        std::filesystem::remove_all(name_, ignored);
    }
    const std::string& name() const noexcept { return name_; }
private:
    std::string name_;
};
DatabaseOptions tiny_options() {
    DatabaseOptions options;
    options.arena_bytes = 256 * 1024;
    options.cache_slots = 2;
    options.cache_bytes = 1024;
    options.table = {128, 256, 32, 64 * 1024};
    return options;
}
CompactionOptions tiny_compaction() {
    return {{128, 16, 32, 300}, 16, 16 * 300};
}
```

The independent model applies each committed mutation in batch order. The
structured binding in `matches` names the map entry's key and value without
copying them, as in the earlier language chapters. Point reads, range order, row
bytes, original sequence numbers, and absence are all compared. Merely comparing
row counts would miss a wrong surviving version.

```cpp title="tests/compaction.cpp — complete file, segment 3"
using Model = std::map<Key, OwnedEntry, KeyLess>;
void apply(Model& model, std::uint64_t sequence, std::span<const Mutation> mutations) {
    for (const auto& mutation : mutations) {
        if (mutation.kind == EntryKind::erase) { model.erase(mutation.key); }
        else {
            model[mutation.key] = {{mutation.key, sequence}, mutation.kind,
                                  mutation.bytes, mutation.length};
        }
    }
}
bool same(EntryView a, EntryView b) {
    return a.key == b.key && a.kind == b.kind &&
        std::equal(a.value.begin(), a.value.end(), b.value.begin(), b.value.end());
}
void matches(const ReadView& view, const Model& model) {
    auto cursor = view.scan({Relation::events, 0});
    for (const auto& [key, expected] : model) {
        const auto found = view.get(key);
        check(found && same(found->view(), expected.view()), "point matches independent model");
        check(cursor.valid() && same(cursor.entry(), expected.view()), "scan matches independent model");
        cursor.next();
    }
    check(!cursor.valid(), "no extra live row");
    for (std::uint64_t key = 0; key < 12; ++key) {
        const Key user{Relation::events, key};
        check(view.get(user).has_value() == model.contains(user), "absence matches independent model");
    }
}
```

These are the complete fixture builders. `commit(..., false)` leaves the new
batch active; normal calls rotate and flush it. `seed` creates consecutive flushed
generations, not one file containing precomputed query answers. The file-name
helper uses the production naming rule only to inspect existence; it does not
choose collection eligibility.

```cpp title="tests/compaction.cpp — complete file, segment 4"
std::vector<Mutation> batch(unsigned generation) {
    if (generation == 1) {
        std::vector<Mutation> result;
        for (std::uint64_t k = 0; k < 8; ++k) {
            result.push_back(Mutation::put(Event{k, 10, 3, 10 + k}));
        }
        result.push_back(Mutation::put(Account{10, 1}));
        result.push_back(Mutation::put(Account{20, 2}));
        return result;
    }
    if (generation == 2) {
        return {Mutation::erase({Relation::events, 0}),
                Mutation::put(Event{1, 10, 3, 901}), Mutation::put(Account{10, 3})};
    }
    if (generation == 3) {
        return {Mutation::put(Event{2, 10, 3, 702}), Mutation::put(Event{8, 20, 3, 808}),
                Mutation::put(Account{20, 4})};
    }
    if (generation == 4) {
        return {Mutation::put(Event{0, 10, 3, 1000}), Mutation::erase({Relation::events, 3}),
                Mutation::put(Account{10, 5})};
    }
    return {Mutation::put(Event{9, 20, 3, 909}), Mutation::erase({Relation::events, 4})};
}
void commit(Database& db, Model& model, unsigned generation, bool flush = true) {
    const auto mutations = batch(generation);
    const auto result = db.write(mutations);
    check(result.outcome == WriteOutcome::committed, "fixture commit");
    apply(model, result.sequence, mutations);
    if (flush) { check(db.rotate() && db.flush_oldest(), "fixture flush"); }
}
void seed(Database& db, Model& model, unsigned count = 4) {
    for (unsigned i = 1; i <= count; ++i) { commit(db, model, i); }
}
std::filesystem::path name(const std::string& directory, std::uint64_t id, io::FileKind kind) {
    return std::filesystem::path{directory} / io::file_name(id, kind).data();
}
```

## Check an exact cap and a partial transfer

The first test uses a 286-byte cap, where two event rows fit exactly. It offers a
third row without advancing an input cursor, observes rejection, and finishes the
two-row file. It then uses Chapter 10's `WalFileOps` backend to allow four bytes
of a nine-byte write before failure. The name of that test backend does not limit
it to WAL framing; it implements the same `FileOps` boundary used here.

```cpp title="tests/compaction.cpp — complete file, segment 5"
void check_forecast_and_counters() {
    memory::Budget budget{1024 * 1024};
    WalFileOps disk;
    {
        auto file = disk.open();
        TableBuilder builder{budget, file, {128, 8, 32, 286}};
        auto a = Mutation::put(Event{1, 10, 3, 11});
        auto b = Mutation::put(Event{2, 10, 3, 12});
        auto c = Mutation::put(Event{3, 10, 3, 13});
        const auto row = [](const Mutation& m) {
            return EntryView{{m.key, 1}, m.kind, {m.bytes.data(), m.length}};
        };
        check(builder.can_append(row(a)), "first row fits"); builder.append(row(a));
        check(builder.can_append(row(b)), "exact last byte fits"); builder.append(row(b));
        check(!builder.can_append(row(c)), "third row needs another frame and index");
        const auto info = builder.finish();
        check(info.file_bytes == 286 && file.io_bytes().written == 286, "exact forecast and write count");
    }
    disk.length = 0;
    disk.write_allowance = 4;
    disk.interrupt_write = true;
    {
        auto file = disk.open();
        const std::array<std::byte, 9> bytes{};
        expect<io::IoError>([&] { file.write_all(0, bytes); });
        check(file.io_bytes().written == 4, "partial transfers counted before failure");
        auto moved = std::move(file);
        check(file.io_bytes().written == 0 && moved.io_bytes().written == 4, "counter follows file move");
        std::array<std::byte, 9> out{};
        disk.interrupt_read = true;
        check(moved.read_up_to(0, out) == 4 && moved.io_bytes().read == 4, "EOF and interruption count");
    }
    std::uint64_t count = std::numeric_limits<std::uint64_t>::max() - 1;
    bool saturated = false;
    io::IoBytes::add(count, 3, saturated);
    check(saturated && count == std::numeric_limits<std::uint64_t>::max(), "counter saturates");
    check(budget.used() == 0, "forecast releases charge");
}
```

The interruption flags exercise retry without double counting. Moving the file
moves its accounting history. The last addition tests saturation near the
unsigned limit without attempting an enormous allocation or transfer.

## Check the complete version-2 envelope

This synthetic manifest fills all 16 table and four log slots. The test checks
each of 668 single-bit corruption positions and each shorter prefix. It then
tries structurally invalid but otherwise ordinary manifest objects. These checks
validate the codec; they do not claim that fictitious table IDs have real files.
Real-file key-range validation is exercised later.

```cpp title="tests/compaction.cpp — complete file, segment 6"
void check_manifest_v2() {
    memory::Budget budget{1024 * 1024};
    Manifest m;
    m.epoch = 2; m.next_id = 21; m.flushed = 10; m.base_end = 10;
    m.base_count = m.table_count = 16; m.log_count = 4;
    for (std::size_t i = 0; i < 16; ++i) { m.tables[i] = {i + 1, 0, 10, 234}; }
    m.logs[0] = {17, 10, 11}; m.logs[1] = {18, 11, 12};
    m.logs[2] = {19, 12, 13}; m.logs[3] = {20, 13, 0};
    {
        auto image = encode_manifest(budget, m);
        check(image.size() == 668 && decode_manifest(image).base_count == 16, "v2 maximum and grouping");
        for (std::size_t i = 0; i < image.size(); ++i) {
            image[i] ^= std::byte{1};
            expect<FormatError>([&] { (void)decode_manifest(image); });
            image[i] ^= std::byte{1};
            expect<FormatError>([&] { (void)decode_manifest(std::span<const std::byte>{image}.first(i)); });
        }
        auto bad = m; bad.base_count = 17;
        expect<FormatError>([&] { validate_manifest(bad); });
        bad = m; bad.tables[1].end = 9;
        expect<FormatError>([&] { validate_manifest(bad); });
        bad = m; bad.base_end = 11;
        expect<FormatError>([&] { validate_manifest(bad); });
    }
    m.base_count = m.table_count = 0; m.log_count = 1; m.logs[0] = {17, 10, 0};
    auto image = encode_manifest(budget, m);
    const auto decoded = decode_manifest(image);
    check(image.size() == 84 && decoded.base_end == 10 && decoded.base_count == 0,
          "empty base still covers its sequence prefix");
}
```

The zero-file example preserves coverage through sequence 10 and keeps one
active log beginning there. The 84-byte image is valid without a table that
pretends to contain a value at sequence 10.

## Retain the old WAL, then only the old cursor

This test exercises the whole lifetime trace. It captures an arena-backed view,
flushes, captures a table-backed view and cursor, and then creates a view through
batch 3. Compaction must not rewrite any of those states. Their owning handles
are released in a deliberate order so that eligibility changes are observable.

```cpp title="tests/compaction.cpp — complete file, segment 7"
void check_retention_and_prefix() {
    TemporaryDirectory path;
    auto budget = std::make_shared<memory::Budget>(16 * 1024 * 1024);
    Model expected;
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_options()};
        commit(db, expected, 1, false);
        const auto first = expected;
        auto memory_view = db.snapshot();
        const auto first_log = db.manifest().logs[0].id;
        check(db.flush_oldest(), "flush captured arena");
        auto disk_view = db.snapshot();
        const auto first_table = db.manifest().tables[0].id;
        auto escaped_cursor = disk_view.scan({Relation::events, 0});
        commit(db, expected, 2); commit(db, expected, 3);
        const auto at_three = expected;
        auto middle = db.snapshot();
        commit(db, expected, 4); commit(db, expected, 5, false);
        const auto newer_table = db.manifest().tables[3];
        const auto active = db.manifest().logs[0].id;
        db.compact_prefix(3, tiny_compaction());
        const auto stats = db.last_compaction();
        check(stats.installed && stats.input_entries == 16 && stats.output_entries == 10,
              "merge removes overwritten versions and selected tombstone");
        check(stats.output_files == 5 && stats.completed_file_bytes == stats.output_write_bytes,
              "five bounded completed outputs");
        check(stats.merge_read_bytes != 0 && !stats.counters_saturated, "actual input transfers measured");
        check(db.manifest().base_end == 3 && db.manifest().base_count == 5 &&
              db.manifest().tables[5] == newer_table && db.manifest().logs[0].id == active,
              "newer table and active log excluded from prefix");
        for (std::size_t i = 0; i < db.manifest().base_count; ++i) {
            check(db.manifest().tables[i].bytes <= 300, "each output respects whole-file cap");
        }
        matches(memory_view, first); matches(disk_view, first); matches(middle, at_three);
        matches(db.snapshot(), expected);
        std::ofstream(path.name() + "/notes.txt") << "foreign file\n";
        auto cleanup = db.collect_obsolete();
        check(cleanup.retained == 4 && cleanup.failed == 0 && !cleanup.sync_failed,
              "old WAL and three old tables retained");
        check(std::filesystem::exists(name(path.name(), first_log, io::FileKind::log)), "retained WAL name");
        memory_view.reset(); middle.reset(); disk_view.reset();
        cleanup = db.collect_obsolete();
        check(cleanup.retained == 1, "cursor alone retains old table source");
        check(std::filesystem::exists(name(path.name(), first_table, io::FileKind::table)), "cursor file exists");
        std::size_t rows = 0;
        while (escaped_cursor.valid()) { ++rows; escaped_cursor.next(); }
        check(rows == first.size(), "old cursor advances after unrelated reclamation");
        check(db.collect_obsolete().retained == 1, "exhausted cursor still owns its view");
        escaped_cursor.reset();
        cleanup = db.collect_obsolete();
        check(cleanup.retained == 0 && cleanup.removed == 1, "final cursor release enables deletion");
        check(std::filesystem::exists(path.name() + "/notes.txt"), "foreign file preserved");
        const auto epoch = db.manifest().epoch;
        expect<std::invalid_argument>([&] { db.compact_prefix(1, tiny_compaction()); });
        check(db.manifest().epoch == epoch && !db.poisoned(), "cannot select part of the base");
        // Consume the newer delta together with the whole old base; do not include sealed log 5.
        db.compact_prefix(db.manifest().table_count, tiny_compaction());
        matches(db.snapshot(), expected);
        check(db.manifest().base_end == 4, "second compaction extends base boundary");
        (void)db.collect_obsolete();
        std::cout << "compaction: input_entries=" << stats.input_entries
                  << " output_entries=" << stats.output_entries << " files=" << stats.output_files
                  << " read_bytes=" << stats.merge_read_bytes << " written_bytes=" << stats.output_write_bytes << '\n';
    }
    check(budget->used() == 0, "all compaction and read charges returned");
    {
        Database reopened{budget, path.name(), DatabaseOpen::recover, tiny_options()};
        matches(reopened.snapshot(), expected);
    }
    check(budget->used() == 0, "reopened owners released");
}
```

The first collection reports four retained obsolete sources: the first log and
three old tables. After releasing the views, only the cursor's first table
remains. Even after that cursor is exhausted, the filename must exist. Its final
reset permits exactly that last retained file to be removed. A foreign text file
survives every collection.

The later call compacts the whole base plus batch 4, but not batch 5's newer log.
This tests repeated base extension, not only the first version-1-to-version-2
transition. Charge returns to zero after all database and read owners leave
scope, then a fresh database reconstructs the same current result.

## Repack a full base instead of trapping the writer

A capacity bound is incomplete when maintenance cannot make progress at the
bound. Thirty-two event rows split into sixteen two-row files fill every table
slot. A newer sealed log cannot flush. Repacking the **same complete base** with
a larger per-file cap produces one file at the same sequence boundary; the
pending flush can then proceed. This is why a base-only selection is allowed.
It is not permission to select an arbitrary subset of the base.

```cpp title="tests/compaction.cpp — complete file, segment 8"
void check_base_repacking() {
    TemporaryDirectory path;
    auto budget = std::make_shared<memory::Budget>(16 * 1024 * 1024);
    Model expected;
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_options()};
        std::vector<Mutation> rows;
        for (std::uint64_t key = 0; key < 32; ++key) {
            rows.push_back(Mutation::put(Event{key, 10, 3, 100 + key}));
        }
        const auto result = db.write(rows);
        check(result.outcome == WriteOutcome::committed, "repack fixture write");
        apply(expected, result.sequence, rows);
        check(db.rotate() && db.flush_oldest(), "repack fixture flush");
        db.compact_prefix(1, tiny_compaction());
        check(db.manifest().base_count == 16 && db.manifest().table_count == 16,
              "small partitions fill every table slot");
        auto old = db.snapshot();
        const std::array later{Mutation::put(Event{99, 10, 3, 999})};
        const auto write = db.write(later);
        check(write.outcome == WriteOutcome::committed && db.rotate(), "newer sealed log");
        apply(expected, write.sequence, later);
        expect<std::length_error>([&] { (void)db.flush_oldest(); });
        check(!db.poisoned(), "full table slots reject flush before effects");
        auto options = tiny_compaction();
        options.output.file_limit = options.max_output_bytes = 64 * 1024;
        options.output.block_limit = 256;
        options.max_output_files = 1;
        db.compact_prefix(db.manifest().base_count, options);
        check(db.manifest().base_count == 1 && db.manifest().base_end == 1,
              "base-only repack frees slots without advancing boundary");
        check(old.get({Relation::events, 0}) && !old.get({Relation::events, 99}),
              "older view does not acquire newer log");
        check(db.flush_oldest(), "freed slot permits pending flush");
        matches(db.snapshot(), expected);
        check(db.collect_obsolete().retained == 16, "old base partitions remain retained");
        old.reset();
        check(db.collect_obsolete().retained == 0, "old base released");
    }
    check(budget->used() == 0, "repack owners released");
    Database reopened{budget, path.name(), DatabaseOpen::recover, tiny_options()};
    matches(reopened.snapshot(), expected);
}
```

The old view keeps all sixteen original partitions alive while the current
manifest uses the repacked file and a later delta. Repacking reduces serving
slots, not necessarily immediate disk usage: retained snapshots still have a
cost. If format, output, or memory caps genuinely cannot accommodate the live
base, report capacity failure; no automatic guarantee of unbounded ingestion is
being made.

## Let all selected values disappear

The next test creates a put followed by its erase. The correct output is zero
files with a nonzero covered boundary, not an empty table and not an unchanged
zero boundary. An older view must still see the put. After reopening, a new batch
can extend that empty base and compact normally.

```cpp title="tests/compaction.cpp — complete file, segment 9"
void check_empty_base() {
    TemporaryDirectory path;
    auto budget = std::make_shared<memory::Budget>(4 * 1024 * 1024);
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_options()};
        const std::array put{Mutation::put(Event{1, 10, 3, 12})};
        const std::array erase{Mutation::erase({Relation::events, 1})};
        check(db.write(put).outcome == WriteOutcome::committed, "empty-base put");
        (void)db.rotate(); (void)db.flush_oldest();
        auto old = db.snapshot();
        check(db.write(erase).outcome == WriteOutcome::committed, "empty-base erase");
        (void)db.rotate(); (void)db.flush_oldest();
        db.compact_prefix(2, tiny_compaction());
        check(db.manifest().base_end == 2 && db.manifest().base_count == 0 &&
              db.manifest().table_count == 0 && db.last_compaction().output_write_bytes == 0,
              "all-deleted prefix installs zero files, not an empty file");
        check(!db.get({Relation::events, 1}) && old.get({Relation::events, 1}), "no deleted-key resurrection");
        (void)db.collect_obsolete(); old.reset(); (void)db.collect_obsolete();
    }
    Database reopened{budget, path.name(), DatabaseOpen::recover, tiny_options()};
    check(!reopened.get({Relation::events, 1}), "empty base survives reopen");
    Model expected;
    commit(reopened, expected, 3);
    reopened.compact_prefix(1, tiny_compaction());
    matches(reopened.snapshot(), expected);
}
```

## Inject failure at observable boundaries

The hook's erased `void*` points to a `Cut` object whose scope outlives the database.
Only `checkpoint` converts it back to its actual type. A cut is armed after setup,
fires once, and either throws a controlled error or calls `std::_Exit(73)`.
`_Exit` deliberately skips stack unwinding; it is not a simulation of loss of
kernel caches or device power. The other hook mode arms the next charged
allocation only after `manifest-closed`, allowing us to test the nonallocating
publication tail.

```cpp title="tests/compaction.cpp — complete file, segment 10"
struct Cut {
    std::string stage;
    bool crash = false;
    bool armed = false;
    memory::Budget* allocation_after_install = nullptr;
};
void checkpoint(void* pointer, std::string_view stage) {
    auto& cut = *static_cast<Cut*>(pointer);
    if (cut.allocation_after_install && stage == "manifest-closed") {
        cut.allocation_after_install->fail_allocation_on(1);
        cut.allocation_after_install = nullptr;
    }
    if (!cut.armed || cut.stage != stage) { return; }
    cut.armed = false;
    if (cut.crash) { std::_Exit(73); }
    throw std::system_error(EIO, std::generic_category(), "injected maintenance failure");
}
```

The cleanup test first injects a pre-unlink error and then a cleanup-sync error.
Both must be reported without poisoning an otherwise valid serving set. It then
adds an unreferenced high-ID file to expose the identity-fence requirement. A
failed fence must leave that file present and require reopen. Successful cleanup
must preserve the high-water value after the orphan name has disappeared.

```cpp title="tests/compaction.cpp — complete file, segment 11"
void check_cleanup_failures_and_highwater() {
    TemporaryDirectory path;
    auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    Model model;
    Cut cut;
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_options(), {checkpoint, &cut}};
        seed(db, model, 3); db.compact_prefix(3, tiny_compaction());
        cut.stage = "before-unlink"; cut.armed = true;
        auto result = db.collect_obsolete();
        check(result.failed == 1 && result.first_error == EIO && !db.poisoned(), "unlink error is visible, not fatal");
        cut.stage = "before-cleanup-sync"; cut.armed = true;
        result = db.collect_obsolete();
        check(result.removed == 1 && result.sync_failed && !db.poisoned(), "cleanup sync uncertainty is visible");
        result = db.collect_obsolete();
        check(result.removed == 0 && !result.sync_failed, "empty cleanup retries synchronization");
        matches(db.snapshot(), model);
    }
    const auto high_orphan = name(path.name(), 1000, io::FileKind::temporary);
    std::ofstream(high_orphan) << "unfinished";
    {
        Database db{budget, path.name(), DatabaseOpen::recover, tiny_options(), {checkpoint, &cut}};
        cut.stage = "before-rename"; cut.armed = true;
        expect<std::system_error>([&] { (void)db.collect_obsolete(); });
        check(db.poisoned() && std::filesystem::exists(high_orphan), "failed identity fence deletes nothing");
    }
    {
        Database db{budget, path.name(), DatabaseOpen::recover, tiny_options()};
        const auto result = db.collect_obsolete();
        check(result.removed >= 1 && db.manifest().next_id > 1000, "identity fenced before orphan removal");
        check(!std::filesystem::exists(high_orphan), "orphan removed after fence");
    }
    {
        Database db{budget, path.name(), DatabaseOpen::recover, tiny_options()};
        check(db.manifest().next_id > 1000, "identity survives removal and reopen");
        matches(db.snapshot(), model);
    }
}
```

Injecting before `unlink` checks policy/error reporting, not an actual filesystem
permission or media failure. The native path still executes real enumeration,
normal unlink, and synchronization. Do not relabel the injected exception as an
observed disk failure.

The allocation sweep rebuilds the same fixture for each failure ordinal. It
covers charged state/control storage, input pins, output buffers, metadata, and
new reader owners without assuming one implementation's exact allocation count.
After any failure, an old view must remain usable. A fresh reopen must see a whole
valid serving state, and collection must handle any leftover output.

```cpp title="tests/compaction.cpp — complete file, segment 12"
void check_failures() {
    std::size_t failures = 0;
    bool succeeded = false;
    for (std::size_t ordinal = 1; ordinal <= 256 && !succeeded; ++ordinal) {
        TemporaryDirectory path;
        auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
        Model model;
        {
            Database db{budget, path.name(), DatabaseOpen::create, tiny_options()};
            seed(db, model, 3);
            auto old = db.snapshot();
            budget->fail_allocation_on(ordinal);
            try { db.compact_prefix(3, tiny_compaction()); succeeded = true; }
            catch (const std::bad_alloc&) { ++failures; }
            budget->clear_failure();
            matches(old, model);
            if (!db.poisoned()) { matches(db.snapshot(), model); }
        }
        check(budget->used() == 0, "failed candidate released all charged owners");
        {
            Database db{budget, path.name(), DatabaseOpen::recover, tiny_options()};
            matches(db.snapshot(), model);
            (void)db.collect_obsolete();
        }
    }
    check(succeeded && failures > 5, "allocation sweep reached buffers and owner controls");
    for (const auto mode : {0, 1, 2}) {
        TemporaryDirectory path;
        auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
        Model model;
        {
            Database db{budget, path.name(), DatabaseOpen::create, tiny_options()};
            seed(db, model, 3);
            auto options = tiny_compaction();
            if (mode == 0) { options.max_output_files = 1; }
            if (mode == 1) { options.max_output_bytes = 300; }
            if (mode == 2) { options.output.file_limit = 233; }
            expect<std::exception>([&] { db.compact_prefix(3, options); });
            check(db.poisoned() && !db.last_compaction().installed, "bounded output failure requires reopen");
            check(db.last_compaction().output_write_bytes != 0, "partial output bytes remain visible");
        }
        Database db{budget, path.name(), DatabaseOpen::recover, tiny_options()};
        matches(db.snapshot(), model);
        check(db.collect_obsolete().removed_file_bytes > 0, "failed output cleanup measured");
    }
    {
        TemporaryDirectory path;
        auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
        Model model; Cut cut;
        Database db{budget, path.name(), DatabaseOpen::create, tiny_options(), {checkpoint, &cut}};
        seed(db, model, 3);
        cut.allocation_after_install = budget.get();
        db.compact_prefix(3, tiny_compaction());
        // Publication must not have consumed the next allocation failure.
        expect<std::bad_alloc>([&] { (void)memory::Allocator<std::byte>{*budget}.allocate(1); });
        budget->clear_failure(); matches(db.snapshot(), model);
        auto lease = budget->reserve(budget->limit() - budget->used());
        const auto epoch = db.manifest().epoch;
        expect<memory::BudgetExceeded>([&] { (void)db.collect_obsolete(); });
        check(db.manifest().epoch == epoch && !db.poisoned(), "cleanup admission changes no authority");
    }
    std::cout << "allocation failure points=" << failures << "\n";
}
```

The three capacity failures differ: output-file slots, aggregate output bytes,
and a per-file cap too small even for one event row. They leave partial artifacts
and an uninstalled result. The final budget lease consumes all remaining charged
capacity, so directory-list admission fails before any new manifest epoch.

Finish the file with its native-process interface and `main`. The native fixture
uses batches 1–4; the in-process test above additionally exercises active batch 5.
The `inspect` action compares all current rows against the independent model and
creates no new mutations.

```cpp title="tests/compaction.cpp — complete file, segment 13"
int native(int argc, char** argv) {
    if (argc < 3) { throw std::invalid_argument("native action needs directory"); }
    const std::string action{argv[1]}, directory{argv[2]};
    auto budget = std::make_shared<memory::Budget>(16 * 1024 * 1024);
    Cut cut;
    if (argc >= 4) { cut.stage = argv[3]; }
    if (argc >= 5) { cut.crash = std::string_view{argv[4]} == "crash"; }
    Database db{budget, directory, action == "seed" ? DatabaseOpen::create : DatabaseOpen::recover,
                tiny_options(), {checkpoint, &cut}};
    Model model;
    if (action == "seed") { seed(db, model); }
    else if (action == "compact") {
        cut.armed = !cut.stage.empty(); db.compact_prefix(3, tiny_compaction());
    } else if (action == "collect") {
        cut.armed = !cut.stage.empty();
        const auto result = db.collect_obsolete();
        std::cout << "removed=" << result.removed << " bytes=" << result.removed_file_bytes
                  << " failed=" << result.failed << " sync_failed=" << result.sync_failed << '\n';
    } else if (action == "inspect") {
        for (unsigned i = 1; i <= 4; ++i) { const auto b = batch(i); apply(model, i, b); }
        matches(db.snapshot(), model);
    } else { throw std::invalid_argument("unknown native action"); }
    std::cout << "sequence=" << db.sequence() << " base=" << db.manifest().base_end
              << " base_files=" << db.manifest().base_count << '\n';
    return 0;
}
}  // namespace
int main(int argc, char** argv) {
    try {
        if (argc != 1) { return native(argc, argv); }
        check_forecast_and_counters(); check_manifest_v2(); check_retention_and_prefix();
        check_base_repacking(); check_empty_base(); check_cleanup_failures_and_highwater(); check_failures();
        std::cout << "compaction and reclamation checks passed\n";
        return 0;
    } catch (const std::exception& error) {
        std::cerr << "compaction check failed: " << error.what() << '\n';
        return 1;
    }
}
```

## Decode the files independently

Create `scripts/check_compaction_fixture.py` from the following segments. It imports
only `frame`, `make_filter`, and `need` from Chapter 8's complete
`scripts/check_table_fixture.py`, which must remain beside it. Those helpers parse
a framed checksum, construct expected Bloom bytes, and raise a test failure.
They do not invoke the C++ reader. Python is an independent fixture oracle, not
a dependency of the engine or its bounded-memory path.

```python title="scripts/check_compaction_fixture.py — complete file, segment 1"
#!/usr/bin/env python3
"""Decode compacted files and interrupt native maintenance in disposable directories."""
from __future__ import annotations

import argparse
from pathlib import Path
import shutil
import struct
import subprocess
import tempfile
import zlib

from check_table_fixture import frame, make_filter, need
```

The table decoder walks every data block, checks directory endpoints and exact
consumption, and reconstructs Bloom bytes from decoded user keys. It uses ordinary
Python lists because these disposable fixtures are small; this is not a claim
that the verification script is a bounded production reader.

```python title="scripts/check_compaction_fixture.py — complete file, segment 2"
def table(path: Path) -> list[tuple]:
    data = path.read_bytes()
    magic, version, block_cap, reserved, crc = struct.unpack_from("<8sIIII", data)
    need(magic == b"STRATABL" and version == 1 and reserved == 0, "table header")
    need(64 <= block_cap <= 65536 and crc == zlib.crc32(data[:20]), "table header bounds/CRC")
    magic, meta_at, meta_size, count, total, crc = struct.unpack_from("<8sQIQQI", data, len(data) - 40)
    need(magic == b"STRATEND" and total == len(data), "table footer")
    need(crc == zlib.crc32(data[-40:-4]) and meta_at + meta_size + 40 == total, "footer CRC/extent")
    meta = frame(data, meta_at, meta_size, 2)
    blocks, filter_bytes, probes = struct.unpack_from("<III", meta)
    need(blocks <= 4096 and probes == 6 and len(meta) == 12 + 46 * blocks + filter_bytes,
         "bounded table directory")
    rows: list[tuple] = []
    next_offset = 24
    for index in range(blocks):
        at = 12 + 46 * index
        first, last = struct.unpack_from("<BQQ", meta, at), struct.unpack_from("<BQQ", meta, at + 17)
        offset, size = struct.unpack_from("<QI", meta, at + 34)
        need(offset == next_offset and size <= block_cap + 12, "block extent")
        payload = frame(data, offset, size, 1)
        entries, = struct.unpack_from("<I", payload)
        position = 4
        block_rows = []
        for _ in range(entries):
            relation, primary, sequence, kind, length = struct.unpack_from("<BQQBI", payload, position)
            position += 22
            need(length <= 30 and position + length <= len(payload), "row length")
            block_rows.append((relation, primary, sequence, kind, payload[position:position + length]))
            position += length
        need(position == len(payload) and block_rows, "block consumption")
        need(block_rows[0][:3] == first and block_rows[-1][:3] == last, "directory endpoints")
        rows.extend(block_rows)
        next_offset += size
    need(next_offset == meta_at and count == len(rows), "whole-table accounting")
    need(rows == sorted(rows, key=lambda row: (row[0], row[1], -row[2])), "internal order")
    need(meta[-filter_bytes:] == make_filter({row[:2] for row in rows}, filter_bytes), "Bloom bytes")
    return rows
```

The manifest decoder independently handles both header lengths. For base files
it checks the shared temporal boundary, 300-byte fixture cap, absence of
tombstones, and globally increasing unique user keys. For deltas and logs it
retains the older contiguous-interval rules. File existence and size are checked
against the authority, not guessed from the highest numbered filename.

```python title="scripts/check_compaction_fixture.py — complete file, segment 3"
def manifest(directory: Path) -> dict:
    data = (directory / "MANIFEST").read_bytes()
    need(76 <= len(data) <= 668, "manifest size")
    magic, version, nt, nl, base_count, epoch, next_id, flushed = struct.unpack_from("<8sIIIIQQQ", data)
    need(magic == b"STRAMAN1" and version in (1, 2), "manifest family/version")
    header = 56 if version == 2 else 48
    base_end = struct.unpack_from("<Q", data, 48)[0] if version == 2 else 0
    need(nt <= 16 and 1 <= nl <= 4 and base_count <= nt and epoch > 0, "manifest counts")
    need((version == 1 and base_count == 0) or (version == 2 and 0 < base_end <= flushed), "base form")
    need(len(data) == header + nt * 32 + nl * 24 + 4, "manifest exact length")
    need(struct.unpack_from("<I", data, len(data) - 4)[0] == zlib.crc32(data[:-4]), "manifest CRC")
    tables = [struct.unpack_from("<QQQQ", data, header + 32 * i) for i in range(nt)]
    logs = [struct.unpack_from("<QQQ", data, header + 32 * nt + 24 * i) for i in range(nl)]
    ids = [record[0] for record in tables + logs]
    need(len(set(ids)) == len(ids) and all(0 < value < next_id for value in ids), "manifest identities")
    boundary = base_end
    base_rows = []
    for i, (identity, start, end, size) in enumerate(tables):
        path = directory / f"s-{identity:020}.sst"
        need(path.stat().st_size == size, "manifest file size")
        rows = table(path)
        need(rows and all(start < row[2] <= end for row in rows), "table interval membership")
        if i < base_count:
            need(start == 0 and end == base_end and size <= 300, "base interval/file cap")
            need(all(row[3] == 1 for row in rows), "base contains no tombstone")
            base_rows.extend(rows)
        else:
            need(start == boundary and end > start, "delta prefix")
            need(min(row[2] for row in rows) == start + 1 and max(row[2] for row in rows) == end,
                 "delta coverage endpoints")
            boundary = end
    need(boundary == flushed, "flushed coverage")
    keys = [row[:2] for row in base_rows]
    need(keys == sorted(set(keys)), "base files have disjoint increasing user ranges")
    for i, (identity, start, end) in enumerate(logs):
        need(start == boundary, "log coverage")
        need((directory / f"s-{identity:020}.wal").is_file(), "required log exists")
        if i + 1 == nl:
            need(end == 0, "active marker")
        else:
            need(end > start, "sealed interval"); boundary = end
    return {"bytes": data, "base": base_end, "base_count": base_count,
            "next": next_id, "tables": tables, "logs": logs, "base_rows": base_rows}
```

Expected base rows are calculated directly from batches 1–3. The two accounts and
eight events preserve their actual winning sequence numbers. Their value bytes
are independently encoded using Python `struct`, including format and relation
bytes, so a self-consistent bug in the C++ encoder and decoder cannot alone make
the oracle pass.

```python title="scripts/check_compaction_fixture.py — complete file, segment 4"
def expected_base() -> list[tuple]:
    rows = []
    for key in range(1, 9):
        sequence, account, amount = 1, 10, 10 + key
        if key == 1: sequence, amount = 2, 901
        if key == 2: sequence, amount = 3, 702
        if key == 8: sequence, account, amount = 3, 20, 808
        rows.append((1, key, sequence, 1, struct.pack("<BBQQIQ", 1, 1, key, account, 3, amount)))
    rows.extend([(2, 10, 2, 1, struct.pack("<BBQI", 1, 2, 10, 3)),
                 (2, 20, 3, 1, struct.pack("<BBQI", 1, 2, 20, 4))])
    return rows
```

Begin `main`. Each case gets a copy of a closed seed directory, so neither live
file descriptors nor a copied locked process are involved. The normal case
verifies all five base files, unchanged newer dependencies, and that collection
leaves only currently authoritative numbered files.

```python title="scripts/check_compaction_fixture.py — complete file, segment 5"
def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--program", type=Path, required=True)
    args = parser.parse_args()
    executable = args.program.resolve()

    def run(action: str, directory: Path, *extra: str, code: int = 0, text: str = "") -> str:
        result = subprocess.run([str(executable), action, str(directory), *extra],
                                capture_output=True, text=True, timeout=60)
        output = result.stdout + result.stderr
        need(result.returncode == code, f"{action} exit={result.returncode}, expected={code}: {output}")
        need(text in output, f"{action} missing {text!r}: {output}")
        return output

    with tempfile.TemporaryDirectory(prefix="strata-compaction-") as temporary:
        root = Path(temporary)
        seed = root / "seed"; seed.mkdir()
        run("seed", seed)
        before = manifest(seed)
        need(before["base"] == 0 and len(before["tables"]) == 4, "legacy seed")
        serial = 0

        def clone(source: Path) -> Path:
            nonlocal serial
            serial += 1
            destination = root / f"case-{serial}"
            shutil.copytree(source, destination)
            return destination

        normal = clone(seed)
        run("compact", normal)
        after = manifest(normal)
        need(after["base_rows"] == expected_base() and after["base_count"] == 5, "independent base contents")
        need(after["tables"][-1] == before["tables"][-1] and after["logs"] == before["logs"], "newer files unchanged")
        run("collect", normal); run("inspect", normal)
        current = manifest(normal)
        need(len(list(normal.glob("s-*"))) == len(current["tables"]) + len(current["logs"]), "only live numbered files remain")
```

Continue the same function with the interruption campaign, then the module entry
point. There are 13 compaction cut points, each tested once by an exception and
once by abrupt process exit: 26 interruptions. The process-visible manifest
changes at rename; every fresh inspector must nevertheless return the complete
same logical current database. Two more process exits interrupt cleanup. A
checksummed swap of base records and a missing required file must fail reopening.

```python title="scripts/check_compaction_fixture.py — complete file, segment 6"
        stages = ["compaction-row", "compaction-file-ready", "compaction-prepared",
                  "dependencies-synced", "manifest-created", "manifest-written", "manifest-synced",
                  "before-rename", "manifest-renamed", "before-directory-sync", "directory-synced",
                  "manifest-stabilized", "manifest-closed"]
        installed_at = stages.index("manifest-renamed")
        for crash in (False, True):
            for index, stage in enumerate(stages):
                directory = clone(seed)
                run("compact", directory, stage, "crash" if crash else "throw", code=73 if crash else 1)
                observed = manifest(directory)
                need(observed["base"] == (3 if index >= installed_at else 0), "visible rename boundary")
                run("inspect", directory); run("collect", directory); run("inspect", directory)

        for stage in ("before-unlink", "before-cleanup-sync"):
            directory = clone(normal)
            (directory / "s-00000000000000001000.tmp").write_bytes(b"failed output")
            run("collect", directory, stage, "crash", code=73)
            need(manifest(directory)["next"] > 1000, "persisted high-water precedes cleanup")
            run("collect", directory); run("inspect", directory)
            need(not (directory / "s-00000000000000001000.tmp").exists(), "cleanup retry")
            need(manifest(directory)["next"] > 1000, "removed high identity not reused")

        # Valid checksums do not authorize overlapping key partitions.
        damaged = clone(normal)
        image = bytearray((damaged / "MANIFEST").read_bytes())
        first, second = image[56:88], image[88:120]
        image[56:88], image[88:120] = second, first
        struct.pack_into("<I", image, len(image) - 4, zlib.crc32(image[:-4]))
        (damaged / "MANIFEST").write_bytes(image)
        run("inspect", damaged, code=1, text="base file key ranges overlap")
        missing = clone(normal)
        (missing / f"s-{after['tables'][0][0]:020}.sst").unlink()
        run("inspect", missing, code=1, text="openat")
        print("oracle: five base files, ten live rows; checksums, Bloom bytes and boundaries verified")
        print("oracle: 26 compaction interruptions and 2 cleanup process exits; reopen results preserved")


if __name__ == "__main__":
    main()
```

The campaign observes real process termination and reopening on the same running
operating system. It does not erase volatile kernel state, reorder persisted
writes, simulate every possible short sector write, or test physical power loss.
Before rename the old manifest remains authoritative; afterward the new one is
visible to these fresh processes. A failed operation may have committed its
maintenance transition, which is why the old facade stops rather than claiming
rollback.

## Build and run from the learner root

Append this block to the existing `CMakeLists.txt`; do not replace earlier targets
or add a second copy of the `read_view.cpp` source. The Python interpreter was
already found by the previous native-fixture tests.

```cmake title="CMakeLists.txt — append compaction targets"
target_sources(strata-core PRIVATE src/storage/compaction.cpp)
add_executable(strata-compaction-tests tests/compaction.cpp)
target_link_libraries(strata-compaction-tests PRIVATE strata-core)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(strata-compaction-tests PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion)
endif()
add_test(NAME compaction COMMAND strata-compaction-tests)
add_test(NAME compaction-native
    COMMAND "${Python3_EXECUTABLE}"
            "${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_compaction_fixture.py"
            --program $<TARGET_FILE:strata-compaction-tests>)
```

Use the compatible native C++20/CMake tools established in Chapter 1. No new
compiler-version gate or macOS-version assumption is added. The target is still
native ARM64 on the M4 Mac mini; inspect the actual architecture and toolchain
instead of relying on the machine model alone.

```bash title="build and check all chapters"
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug --parallel 4
ctest --test-dir build/debug --output-on-failure
./build/debug/strata-compaction-tests
python3 scripts/check_compaction_fixture.py --program ./build/debug/strata-compaction-tests

cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/release --parallel 4
ctest --test-dir build/release --output-on-failure
```

Run address/undefined-behavior instrumentation in a separate build. Use the
sanitizer facilities available in the native toolchain. Linux leak-detection
settings are not mandatory flags for Apple's toolchain.

```bash title="sanitizer configuration"
cmake -S . -B build/sanitize -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build/sanitize --parallel 4
ctest --test-dir build/sanitize --output-on-failure
```

The observed Clang 17/Linux fixture output was:

```text title="observed compaction output"
compaction: input_entries=16 output_entries=10 files=5 read_bytes=746 written_bytes=1398
allocation failure points=59
compaction and reclamation checks passed
oracle: five base files, ten live rows; checksums, Bloom bytes and boundaries verified
oracle: 26 compaction interruptions and 2 cleanup process exits; reopen results preserved
```

The allocation count is an observation, not a portable requirement. Input read
bytes depend on the deliberately small cache and its prior access history;
1,398 output bytes follow the fixed framing and split policy for this fixture.
The success assertions, bounds, old-view results, and eventual returned charges
are the portable checks.

All **nineteen cumulative CTest entries passed in Debug, Release, and combined
AddressSanitizer/UndefinedBehaviorSanitizer builds with Clang 17 on Linux**. Native
M4/macOS execution and Apple full-device synchronization remain unverified. The
process-interruption campaign does not establish physical power-loss durability.
No concurrent-use, process-memory ceiling, or throughput result is claimed.

## A lifetime-policy defect that sanitizers do not detect

Temporarily change only the retained-file branch in `Database::collect_obsolete`:

```cpp title="temporary diagnostic edit; do not retain"
if (false && context_->retains_file(file.id, file.kind)) {
    ++result.retained;
    continue;
}
```

This deliberately removes every obsolete name regardless of retained readers.
The old reader may still have an open descriptor, so a successful read is not
proof that the policy was followed. Rebuild and run only the `compaction` test
in each configuration:

```bash title="diagnose the retention defect"
cmake --build build/debug --parallel 4
ctest --test-dir build/debug -R '^compaction$' --output-on-failure
cmake --build build/release --parallel 4
ctest --test-dir build/release -R '^compaction$' --output-on-failure
cmake --build build/sanitize --parallel 4
ctest --test-dir build/sanitize -R '^compaction$' --output-on-failure
```

The observed failure in all three configurations was
`compaction check failed: old WAL and three old tables retained`, without a
sanitizer memory report. Restore the branch from Part 2 and rerun all nineteen
tests. The canonical implementation contains the correct branch; this diagnostic
is not a new API or a second implementation to keep.

## Read the measurements without changing their meaning

For the selected three fixture files, their logical lengths total 1,594 bytes.
The completed replacements total 1,398 bytes. Define $I$ as selected-file bytes
and $W$ as successful new-output write bytes. The local rewrite ratio is

$$
R=\frac{W}{I}=\frac{1398}{1594}\approx0.877.
$$

This does not mean compaction had negative cost. It performed 1,398 additional
write bytes while old and new files coexisted. The smaller final representation
removed six stored versions but paid for five separate headers, filters,
directories, and footers. The measured 746 input-transfer bytes are smaller than
$I$ because the phase excludes opening/verification and some blocks were already
cached. Do not substitute $W/746$ and call it a disk write-amplification factor.

Whole-engine write amplification would need an explicit denominator of accepted
logical input bytes and a numerator including WAL, flush, compaction, and chosen
metadata traffic. These operation-local counters do not yet measure all of that.
Repeated base-only repacking demonstrates extra writes without new logical data;
its cost is visible even when the row count and base boundary do not change.
After a failed attempt, `output_write_bytes` can be positive while `installed`
is false. Later cleanup reports logical lengths successfully unlinked. Do not
subtract those lengths from write traffic: deletion does not undo previous writes.

## Diagnose the boundary, not just the exception name

| Symptom | Likely boundary to inspect |
|---|---|
| Deleted key reappears | An arbitrary partial source set was fed to the tombstone-suppressing cursor |
| Second output loses its first row | Cursor advanced after a negative `can_append` result |
| A 286-byte cap rejects two event rows | Metadata/frame/footer accounting was omitted or counted twice |
| Empty compaction cannot reopen | Base coverage was derived from an actual output row instead of `base_end` |
| Reads work but retained names vanish | Registry protection was skipped; open descriptors mask early unlink |
| High orphan ID is reused after restart | Its name was removed before the manifest high-water fence |
| Reopen rejects swapped checksummed base records | Correctly detected nonincreasing or overlapping user-key partitions |
| Repacking cannot allocate despite small logical rows | Old pins/states, output workspaces, and owner metadata coexist under the shared budget |

For a small variation, change the 286-byte forecast test to cap 285. Predict the
second admission result, then update that isolated test to finish after its first
row and check the 234-byte file. Restore the original fixture afterward. The
one-byte cap change exposes a real framing boundary, not a timing difference.

## Completion boundary and next mechanism

The database now rewrites a complete older prefix into bounded immutable files,
installs their authority before changing serving owners, and leaves retained
snapshots on their original generations. It can represent an empty base, extend
one, or repack it without advancing visibility. Collection protects live and
retained files, persists identity high-water before deletion, reports partial
cleanup failure, and remains retryable after process exit.

The next chapter builds owned column batches and a pull-based scan/filter/project
pipeline on these snapshots. It must preserve the distinction between an empty
filtered batch and end-of-stream and must charge batch capacity. It does not yet
add external sorting or concurrency.

## References used for the boundaries

The C++ working draft's [`shared_ptr` destructor rules](https://eel.is/c++draft/util.smartptr.shared.dest)
were used for final-owner lifetime. Apple's archived
[`unlink(2)`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/unlink.2.html)
and [`fsync(2)`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
manuals were read for name removal versus open-file lifetime and host versus
device-flush requests. These are archival references, not evidence of native
execution on the reader's installed macOS. SQLite's
[atomic-commit discussion, sections 9.2 and 9.5](https://www.sqlite.org/atomiccommit.html)
provides additional context on persistence assumptions and recovery-file
lifetimes; its rollback-journal protocol is not copied here. Accessed 2026-11-06.
