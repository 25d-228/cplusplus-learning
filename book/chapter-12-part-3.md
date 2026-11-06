# Chapter 12 — Immutable snapshots and merged reads

## Part 3 — Retention, failure, and ownership checks

Parts [1](https://25d-228.github.io/cplusplus-learning/chapter-12.html) and
[2](https://25d-228.github.io/cplusplus-learning/chapter-12-part-2.html) supplied the
owners and merge. Now test their claims at the boundaries where a superficially
correct implementation usually fails: a flush replaces the database's source but
not a view's source; a tombstone must hide an older value; a failed `next()` must
not leave a plausible partial position; and the final cursor may be destroyed
long after the database and the caller's budget handle.

This is the final part of the chapter's **4.5-hour implementation allocation**.
Keep all earlier tests. The new fixture adds one CTest entry; the complete suite
has seventeen. None of the new tests requires a server or starts a reader thread.

## Update the existing database fixture without weakening its assertions

`Database` now requires a `std::shared_ptr<memory::Budget>` rather than a borrowed
budget. Its internal context needs real ownership; fabricating a shared pointer
with a no-op deleter would leave a dangling object when the original scope ends.
Standalone arenas, WAL owners, and readers still have their existing borrowed
budget contracts. Do not change those constructors.

In each of the three budget-owning scopes of `tests/database.cpp`, change the
stack budget to an owning handle plus a reference. Pass the handle only to
`Database`; keep the reference for every existing charge/fault assertion. The
following one-time edit performs that exact migration and refuses an unexpected
input. Run it from the learner source root, not the book package:

```bash title="one-time migration of tests/database.cpp"
python3 - <<'PYCODE'
from pathlib import Path
import re

path = Path("tests/database.cpp")
text = path.read_text(encoding="utf-8")
if "budget_owner" in text:
    raise SystemExit("Database fixture already migrated; do not apply twice")
if "#include <memory>" not in text:
    text = text.replace("#include <limits>", "#include <limits>\n#include <memory>", 1)
text, count = re.subn(
    r"memory::Budget budget\{([^{}]+)\};",
    r"auto budget_owner = std::make_shared<memory::Budget>(\1);\n"
    r"    auto& budget = *budget_owner;", text)
if count != 3:
    raise SystemExit(f"Expected three budget declarations, found {count}; no file written")
text, calls = re.subn(r"(Database\s+\w+\{)budget,", r"\1budget_owner,", text)
if calls == 0 or "memory::Budget budget{" in text:
    raise SystemExit("Unexpected fixture layout; no file written")
path.write_text(text, encoding="utf-8")
print(f"Migrated {count} budget owners and {calls} Database constructions")
PYCODE
```

A typical migrated scope reads `Database db{budget_owner, ...}` followed by
unchanged checks such as `budget.used() == 0`. The reference does not own anything;
the handle and, while necessary, the context do. The root budget's small bootstrap
allocation remains outside its own accounting, as explained in Part 1. This does
not waive charging of reader, source, state, or scan allocations.

## Give the test its own temporary directory

Create `tests/read_view.cpp`. Its filename-labelled blocks below append in reading
order. The test's directory helper creates a fresh private name and removes its
fixture at scope exit. It uses the nonthrowing `std::filesystem::remove_all` overload
that reports through `std::error_code`: a cleanup failure must not throw from a
destructor while another exception is active. This is **test cleanup**, not the
engine's future file-reclamation policy. The directory object is declared before
the engine handles, so those handles normally release first.

The library uses bounded storage; this test may use an ordinary map and vector as
an independent oracle. Those test-only containers are not included in engine
memory accounting. The tiny cache intentionally cannot admit a complete event
frame, exercising Chapter 9's charged bypass instead of silently making every
read a cache hit.

```cpp title="tests/read_view.cpp"
#include "strata/storage/database.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <cstddef>
#include <cstdint>
#include <cstdlib>
#include <exception>
#include <filesystem>
#include <iostream>
#include <limits>
#include <map>
#include <memory>
#include <optional>
#include <span>
#include <stdexcept>
#include <string>
#include <string_view>
#include <system_error>
#include <utility>
#include <vector>

namespace {
using namespace strata;
using namespace strata::storage;

void check(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}
template<class Error, class Function>
void expect(const Function& function) {
    try { function(); }
    catch (const Error&) { return; }
    throw std::runtime_error("expected exception was not thrown");
}
class TemporaryDirectory {
public:
    TemporaryDirectory() {
        char pattern[] = "/tmp/strata-read-XXXXXX";
        const auto created = ::mkdtemp(pattern);
        if (!created) { throw std::system_error(errno, std::generic_category(), "mkdtemp"); }
        name_ = created;
    }
    ~TemporaryDirectory() noexcept {
        std::error_code ignored;
        std::filesystem::remove_all(name_, ignored);
    }
    TemporaryDirectory(const TemporaryDirectory&) = delete;
    TemporaryDirectory& operator=(const TemporaryDirectory&) = delete;
    const std::string& name() const noexcept { return name_; }
    std::size_t files() const {
        std::size_t count = 0;
        for (const auto& entry : std::filesystem::directory_iterator(name_)) {
            (void)entry;
            ++count;
        }
        return count;
    }
private:
    std::string name_;
};
DatabaseOptions tiny_options() {
    DatabaseOptions options;
    options.arena_bytes = 256 * 1024;
    options.cache_slots = 1;
    options.cache_bytes = 128;  // Full event frames bypass; still charged globally.
    options.table.block_bytes = 128;
    options.table.block_limit = 256;
    options.table.filter_bytes = 32;
    return options;
}
void commit(Database& db, std::span<const Mutation> mutations) {
    check(db.write(mutations).outcome == WriteOutcome::committed, "fixture batch committed");
}
std::uint64_t cents(const ReadView& view, std::uint64_t id) {
    const auto entry = view.get({Relation::events, id});
    check(entry.has_value(), "expected visible event");
    return decode_event(entry->view().value).amount_cents;
}
std::uint32_t region(const ReadView& view, std::uint64_t id) {
    const auto entry = view.get({Relation::accounts, id});
    check(entry.has_value(), "expected visible account");
    return decode_account(entry->view().value).region_id;
}
```

`expect<Error>` catches only the expected category. An unrelated exception
propagates to `main` and fails the executable. This matters when a test is intended
to prove a retention-limit rejection: an allocation failure must not accidentally
count as success.

The next helper answers the roadmap's small query: for day 3, match each event to
its account and group the event count and amount by region. It performs a point
lookup for each event using the **same view** that supplies the scan. Using
`Database::get` instead would let later account updates contaminate an old event
scan. The four-region fixed array and at-most-2000-cent fixture are deliberate
bounds, not a general join or an unchecked production aggregation API. The
external join and aggregation engine comes later.

```cpp title="tests/read_view.cpp"
using Groups = std::array<std::pair<std::uint64_t, std::uint64_t>, 4>;
Groups fixture_answer(const ReadView& view) {
    Groups groups{};
    // A tiny test oracle, not the later join/execution engine.
    auto cursor = view.scan({Relation::events, 0}, Key{Relation::accounts, 0});
    for (; cursor.valid(); cursor.next()) {
        const auto event = decode_event(cursor.entry().value);
        if (event.day != 3) { continue; }
        const auto account = view.get({Relation::accounts, event.account_id});
        if (!account) { continue; }
        const auto r = decode_account(account->view().value).region_id;
        check(r < groups.size(), "fixture region bound");
        ++groups[r].first;
        groups[r].second += event.amount_cents;  // Fixture total is at most 2000.
    }
    return groups;
}
```

There are five initial events. Event 103 has no matching account and event 104
belongs to day 4. Before the batch, region 1 contains events 100 and 101 for
$500+250=750$ cents; region 2 contains event 102 for 700 cents. After the batch,
region 3 contains events 100 and 101 for $500+900=1400$ cents; event 102 is absent.

The complete test captures both boundaries, flushes both memtables, and checks all
three source arrangements: an old retained arena, a newer view of two arenas, and
a newly captured pair of tables. The next listing starts the function; the block
following it completes the same function.

```cpp title="tests/read_view.cpp"
void check_repeatable() {
    TemporaryDirectory path;
    auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_options()};
        const std::array initial{
            Mutation::put(Account{10, 1}), Mutation::put(Account{20, 2}),
            Mutation::put(Event{100, 10, 3, 500}), Mutation::put(Event{101, 10, 3, 250}),
            Mutation::put(Event{102, 20, 3, 700}), Mutation::put(Event{103, 99, 3, 50}),
            Mutation::put(Event{104, 10, 4, 100})};
        commit(db, initial);
        auto old = db.snapshot();
        const std::array change{Mutation::put(Account{10, 3}),
            Mutation::put(Event{101, 10, 3, 900}), Mutation::erase({Relation::events, 102})};
        commit(db, change);
        auto newer = db.snapshot();
        check(old.sequence() == 1 && newer.sequence() == 2, "complete batch boundaries");
        check(region(old, 10) == 1 && cents(old, 101) == 250, "old cross-relation values");
        check(region(newer, 10) == 3 && cents(newer, 101) == 900, "new cross-relation values");
        check(old.get({Relation::events, 102}).has_value() && !newer.get({Relation::events, 102}),
              "new tombstone must suppress older value");
        check(db.flush_oldest() && db.flush_oldest(), "flush both captured arenas");
        auto disk = db.snapshot();  // Active is empty: no extra rotation.
        const auto epoch = db.manifest().epoch;
        auto again = db.snapshot();
        check(db.manifest().epoch == epoch && again.source_count() == 2, "empty active not rotated");
        Groups before{}; before[1] = {2, 750}; before[2] = {1, 700};
        Groups after{}; after[3] = {2, 1400};
        check(fixture_answer(old) == before, "old fixture answer survives flush");
        check(fixture_answer(newer) == after, "new fixture answer");
        check(fixture_answer(disk) == after, "disk merge answer");
        check(old.source_count() == 1 && newer.source_count() == 2, "old source arrays unchanged");
```

The second empty-active capture must not rotate again. Its epoch check is not
merely a performance assertion: unnecessarily sealing an empty generation would
consume bounded log slots and manufacture maintenance for a read boundary that
already exists.

The range $[101,104)$ must yield 101 and 103 only. The erase hides 102, and the
exclusive bound excludes 104. An empty interval succeeds with no rows; a reversed
interval is an input error. Invalid relation tags are rejected instead of reaching
an ordering function that was never specified for them.

```cpp title="tests/read_view.cpp"
auto range = disk.scan({Relation::events, 101}, Key{Relation::events, 104});
        for (const auto id : {101ULL, 103ULL}) {
            check(range.valid() && range.entry().key.user.primary == id, "half-open range with erase");
            range.next();
        }
        check(!range.valid(), "range upper bound excluded");
        expect<std::logic_error>([&] { (void)range.entry(); });
        auto empty = disk.scan({Relation::events, 104}, Key{Relation::events, 104});
        check(!empty.valid(), "equal endpoints are empty");
        expect<std::invalid_argument>([&] {
            (void)disk.scan({Relation::events, 105}, Key{Relation::events, 104});
        });
        expect<std::invalid_argument>([&] { (void)disk.get({static_cast<Relation>(9), 0}); });
    }
    check(budget->used() == 0, "repeatable-read charges released");
    std::cout << "snapshot fixture: old=(1,3,2,750),(2,3,1,700); new=(3,3,2,1400)\n";
}
```

## Compare multiple retained boundaries against a separate model

A merge tested only with disjoint keys cannot expose incorrect version priority.
Build an independent current-state map whose update rule is simpler than the
engine: a put replaces one entry and an erase removes it. Copy that map whenever
a view is captured. The model stores the sequence as well as the payload, so a
result with the right bytes but the wrong internal version still fails.

`std::map::lower_bound` is used only by this oracle. The engine's range implementation
remains the hand-written source heap from Part 2. `std::ranges::equal`, introduced
in the earlier tests, compares the counted byte ranges rather than their addresses.

```cpp title="tests/read_view.cpp"
using Model = std::map<Key, OwnedEntry, KeyLess>;
void apply(Model& model, std::span<const Mutation> batch, std::uint64_t sequence) {
    for (const auto& mutation : batch) {
        if (mutation.kind == EntryKind::erase) { model.erase(mutation.key); }
        else {
            OwnedEntry row;
            row.key = {mutation.key, sequence}; row.kind = mutation.kind;
            row.bytes = mutation.bytes; row.length = mutation.length;
            model[mutation.key] = row;
        }
    }
}
bool same(EntryView actual, const OwnedEntry& expected) {
    return actual.key == expected.key && actual.kind == expected.kind &&
        std::ranges::equal(actual.value, expected.view().value);
}
void compare_range(const ReadView& view, const Model& expected,
                   Key lower, std::optional<Key> upper) {
    auto cursor = view.scan(lower, upper);
    auto reference = expected.lower_bound(lower);
    while (reference != expected.end() && (!upper || KeyLess{}(reference->first, *upper))) {
        check(cursor.valid() && same(cursor.entry(), reference->second), "merged range oracle");
        cursor.next(); ++reference;
    }
    check(!cursor.valid(), "merged range has no extra rows");
}
```

Each generation has three batches. Each batch touches 32 event keys and 32 account
keys once, giving $8\times3\times64=1536$ stored versions. Multiplication by five
permutes the 32 key positions because five and 32 have no common factor other than
one; the round offset changes insertion order without introducing duplicate keys
inside a batch. The modulo-seven erase schedule mixes newer tombstones with older
puts. None of these choices depends on random-generator implementation details.

Capture before each flush. A saved view therefore combines prior disk tables with
that generation's retained arena. Later flushes change the live database, not the
saved source arrays. This creates actual mixed-source merges rather than eight
copies of a single final table.

```cpp title="tests/read_view.cpp"
void check_many_sources() {
    TemporaryDirectory path;
    auto budget = std::make_shared<memory::Budget>(16 * 1024 * 1024);
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_options()};
        Model current;
        std::vector<std::pair<ReadView, Model>> saved;
        for (std::uint64_t generation = 0; generation < 8; ++generation) {
            for (std::uint64_t round = 0; round < 3; ++round) {
                std::array<Mutation, 64> batch{};
                for (std::size_t n = 0; n < batch.size(); ++n) {
                    const auto id = (static_cast<std::uint64_t>(n / 2) * 5 + round * 7) % 32;
                    const auto relation = n % 2 == 0 ? Relation::events : Relation::accounts;
                    const Key key{relation, id};
                    const auto value = 100 * generation + 10 * round + id;
                    batch[n] = (generation + round + id) % 7 == 0 ? Mutation::erase(key)
                        : relation == Relation::events ? Mutation::put(Event{id, id, 3, value})
                        : Mutation::put(Account{id, static_cast<std::uint32_t>(value)});
                }
                commit(db, batch);
                apply(current, batch, db.sequence());
            }
            saved.emplace_back(db.snapshot(), current);
            check(db.flush_oldest(), "generation flushed after capture");
        }
```

For each of eight boundaries, compare 64 point lookups and six ranges. The ranges
include a cross-relation boundary, a missing starting key, and an absent upper
bound. Their totals are $8\times64=512$ point checks and $8\times6=48$ ranges.
A range assertion verifies both contents and termination; returning all expected
rows followed by an extra obsolete version must fail too.

```cpp title="tests/read_view.cpp"
for (const auto& [view, expected] : saved) {
            for (const auto relation : {Relation::events, Relation::accounts}) {
                for (std::uint64_t id = 0; id < 32; ++id) {
                    const auto actual = view.get({relation, id});
                    const auto wanted = expected.find({relation, id});
                    check(actual.has_value() == (wanted != expected.end()), "point presence oracle");
                    if (actual) { check(same(actual->view(), wanted->second), "point value oracle"); }
                }
            }
            compare_range(view, expected, {Relation::events, 0}, std::nullopt);
            compare_range(view, expected, {Relation::events, 3}, Key{Relation::events, 13});
            compare_range(view, expected, {Relation::events, 31}, Key{Relation::accounts, 5});
            compare_range(view, expected, {Relation::accounts, 7}, Key{Relation::accounts, 22});
            compare_range(view, expected, {Relation::accounts, 31}, std::nullopt);
            compare_range(view, expected, {Relation::accounts, 32}, std::nullopt);
        }
    }
    check(budget->used() == 0, "multi-source charges released");
    std::cout << "merge oracle: 1536 versions; 8 retained views; 512 point checks; 48 ranges\n";
}
```

## Distinguish handle count from retained-source count

Set the snapshot-state limit to one. Copying its view is allowed because both
handles share one state. Creating a cursor from it also shares that state. After
both view handles are reset, the cursor still prevents another state from being
admitted—even after it has reached the end. Exhaustion releases its current row
and source positions, but does not silently reset the public cursor owner.

Before a rejected capture, record the manifest epoch, current sequence, file count,
and charged bytes. Admission must fail before a new WAL, sealing, or state allocation
changes any of them. Resetting the cursor then releases the final state, allowing
the identical capture to succeed. A second scope isolates the distinct-source
limit: retained arena $M_1$, its replacement table $T_1$, and new arena $M_2$
require three distinct source slots, even though the new view itself needs only two.

```cpp title="tests/read_view.cpp"
void check_retention() {
    TemporaryDirectory path;
    auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    auto options = tiny_options(); options.reads = {1, 2};
    {
        Database db{budget, path.name(), DatabaseOpen::create, options};
        const std::array first{Mutation::put(Account{10, 1})};
        commit(db, first);
        auto old = db.snapshot();
        auto copied = old;
        check(db.retention().snapshots == 1 && db.retention().sources == 1, "copies share admission");
        check(db.flush_oldest(), "retained arena replaced only in database");
        const std::array next{Mutation::put(Account{10, 2})}; commit(db, next);
        auto cursor = old.scan({Relation::accounts, 0});
        old.reset(); copied.reset();
        const auto epoch = db.manifest().epoch;
        const auto files = path.files();
        const auto charged = budget->used();
        expect<std::length_error>([&] { (void)db.snapshot(); });
        check(!db.poisoned() && db.manifest().epoch == epoch && db.sequence() == 2 &&
              path.files() == files && budget->used() == charged, "admission failure changed state");
        cursor.next();
        check(!cursor.valid() && db.retention().snapshots == 1, "exhausted cursor still owns view");
        expect<std::length_error>([&] { (void)db.snapshot(); });
        cursor.reset();
        check(db.retention().snapshots == 0 && db.retention().sources == 0, "last owner releases slots");
        auto accepted = db.snapshot();
        check(accepted.source_count() == 2 && region(accepted, 10) == 2, "retry sees unchanged writer");
        expect<std::length_error>([&] { (void)db.snapshot(); }); // Snapshot-state cap, no new sources.
    }
    check(budget->used() == 0, "retention charges released");
    // Isolate the distinct-source cap from the snapshot-state cap.
    TemporaryDirectory other;
    options.reads = {4, 2};
    {
        Database db{budget, other.name(), DatabaseOpen::create, options};
        const std::array one{Mutation::put(Account{1, 1})}; commit(db, one);
        auto old = db.snapshot(); (void)db.flush_oldest(); commit(db, one);
        const auto epoch = db.manifest().epoch;
        expect<std::length_error>([&] { (void)db.snapshot(); });
        check(!db.poisoned() && db.manifest().epoch == epoch && db.retention().sources == 1,
              "distinct-source rejection precedes sealing");
        old.reset();
        auto accepted = db.snapshot();
        check(accepted.source_count() == 2, "source-slot retry after release");
    }
    check(budget->used() == 0, "source-slot charges released");
    std::cout << "retention: copies share quota; cursors retain quota; rejection then release/retry\n";
}
```

A practical diagnosis follows directly. When capture reports exhausted retention,
inspect outstanding cursors as well as view variables; then inspect whether a
flushed generation is still retained as an arena. Increasing only the cache size
cannot resolve either ownership limit. The test's `budget->used() == 0` is checked
after every engine owner in the scope has been destroyed, not while a legitimate
view still owns storage.

## Separate rejection from an uncertain installation

The hook has the same borrowed, synchronous, nonreentrant contract as Chapter 11.
It can either throw immediately after manifest rename, or arm the next charged
allocation at `manifest-closed`. Neither action emulates a physical power failure.
The latter action tests a precise property: returning a newly installed snapshot
must not allocate after that checkpoint.

```cpp title="tests/read_view.cpp"
struct Cut {
    bool fail_install = false;
    bool arm_after_install = false;
    memory::Budget* budget = nullptr;
};
struct Injected : std::runtime_error { Injected() : std::runtime_error("snapshot install cut") {} };
void checkpoint(void* pointer, std::string_view stage) {
    auto& cut = *static_cast<Cut*>(pointer);
    if (cut.fail_install && stage == "manifest-renamed") { throw Injected{}; }
    if (cut.arm_after_install && stage == "manifest-closed") { cut.budget->fail_allocation_on(1); }
}
```

Sweep allocation failures from the first charged allocation until a capture
completes. Construct a fresh directory each time, so one failed installation cannot
change the next ordinal's prerequisites. The state allocation precedes rotation:
its failure leaves the old database healthy, the directory unchanged, and no
retention registered. A later failure inside rotation follows Chapter 11's stricter
rule: stop the old database object and recover from authoritative files.

The ordinal is a test seam, not an allocator ABI. The fixture requires at least two
failing preparation points and eventual completion; it does not insist that every
standard library produce exactly the Linux run's allocation count.

```cpp title="tests/read_view.cpp"
void check_capture_failures() {
    std::size_t failures = 0;
    bool completed = false;
    for (std::size_t ordinal = 1; ordinal <= 16 && !completed; ++ordinal) {
        TemporaryDirectory path;
        auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
        {
            Database db{budget, path.name(), DatabaseOpen::create, tiny_options()};
            const std::array batch{Mutation::put(Account{1, 1})}; commit(db, batch);
            const auto epoch = db.manifest().epoch;
            const auto files = path.files();
            const auto charged = budget->used();
            budget->fail_allocation_on(ordinal);
            try {
                auto view = db.snapshot();
                check(region(view, 1) == 1 && db.retention().snapshots == 1, "successful capture");
                completed = true;
            } catch (const std::bad_alloc&) {
                ++failures;
                check(db.retention().snapshots == 0 && db.retention().sources == 0,
                      "failed capture leaked retention");
                if (ordinal == 1) {
                    check(!db.poisoned() && db.manifest().epoch == epoch && path.files() == files &&
                          budget->used() == charged, "pre-rotation allocation changed state");
                } else { check(db.poisoned(), "effectful rotation failure must stop writer"); }
            }
            budget->clear_failure();
        }
        check(budget->used() == 0, "capture-failure charges released");
    }
    check(completed && failures >= 2, "capture failure sweep reached success");
```

Continue the same function. First arm an allocation failure after installation.
The capture must return successfully; a deliberately separate charged vector
allocation must then consume the still-armed failure. If snapshot publication
allocated, the capture itself would throw instead.

Next retain that old view and interrupt a later capture after manifest rename.
The candidate state was never admitted. The old view still answers its immutable
sequence after the failed database is destroyed. Only after releasing that view
can the test reopen the directory: the shared context intentionally retains the
exclusive lock. Reopen follows the installed manifest, not the failed facade's
old in-memory diagnostics.

```cpp title="tests/read_view.cpp"
TemporaryDirectory path;
    auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    Cut cut; cut.budget = budget.get();
    ReadView old;
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_options(), {checkpoint, &cut}};
        const std::array first{Mutation::put(Account{1, 1})}; commit(db, first);
        cut.arm_after_install = true;
        old = db.snapshot();
        expect<std::bad_alloc>([&] {
            memory::Vector<std::byte> probe(1, memory::Allocator<std::byte>{*budget});
        });
        budget->clear_failure(); cut.arm_after_install = false;
        const std::array next{Mutation::put(Account{1, 2})}; commit(db, next);
        cut.fail_install = true;
        expect<Injected>([&] { (void)db.snapshot(); });
        check(db.poisoned() && region(old, 1) == 1, "old view survives uncertain maintenance");
        check(db.retention().snapshots == 1, "failed candidate not admitted");
        expect<std::logic_error>([&] { (void)db.snapshot(); });
    }
    check(region(old, 1) == 1, "view reads after writer destruction");
    old.reset();
    {
        Database reopened{budget, path.name(), DatabaseOpen::recover, tiny_options()};
        auto current = reopened.snapshot();
        check(region(current, 1) == 2, "reopen uses installed authority");
    }
    check(budget->used() == 0, "post-failure reopen releases charges");
    std::cout << "capture: " << failures << " allocation failures; post-install publication allocates nothing\n";
}
```

## Make a failed scan terminal, not partially resumable

The next test first rejects allocation of a scan object, then reserves all remaining
budget to exercise the capacity error separately. Neither failure creates another
snapshot state. A cursor owns an existing state; it does not register a new one.

The table payload budget is 128 bytes. With the existing record layout, an event
row occupies 52 bytes, so two rows fit and a third does not. Scan construction
consumes row 0's version group and leaves row 1 at the source head. The next call
consumes row 1, then attempts to load the next block. Injecting a failure at that
load exercises failure **after** an input cursor has advanced, rather than only
failure before any work begins.

The candidate row for that unsuccessful call is not published. `valid()` becomes
false, `entry()` rejects, and another `next()` cannot resume midway through the
group. The immutable view is still usable; a freshly constructed cursor can retry
from an explicit lower bound. Already delivered rows are not rolled back.

```cpp title="tests/read_view.cpp"
void check_scan_failure() {
    TemporaryDirectory path;
    auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_options()};
        std::array<Mutation, 40> rows{};
        for (std::size_t i = 0; i < rows.size(); ++i) {
            rows[i] = Mutation::put(Event{static_cast<std::uint64_t>(i), 1, 3, 50});
        }
        commit(db, rows); (void)db.rotate(); (void)db.flush_oldest();
        auto view = db.snapshot();
        const auto admitted = db.retention().snapshots;
        budget->fail_allocation_on(1);
        expect<std::bad_alloc>([&] { (void)view.scan({Relation::events, 0}); });
        budget->clear_failure();
        check(db.retention().snapshots == admitted, "scan-construction failure leaked view");
        {
            auto deny = budget->reserve(budget->limit() - budget->used());
            expect<memory::BudgetExceeded>([&] { (void)view.scan({Relation::events, 0}); });
        }
        auto cursor = view.scan({Relation::events, 0});
        check(cursor.entry().key.user.primary == 0, "initial scan row");
        budget->fail_allocation_on(1);
        expect<std::bad_alloc>([&] { cursor.next(); });
        budget->clear_failure();
        check(!cursor.valid(), "advance failure must make cursor terminal");
        expect<std::logic_error>([&] { (void)cursor.entry(); });
        cursor.next(); check(!cursor.valid(), "terminal cursor does not resume");
        check(cents(view, 1) == 50, "view remains usable after cursor failure");
        cursor = view.scan({Relation::events, 0});
        std::size_t count = 0;
        for (; cursor.valid(); cursor.next()) { ++count; }
        check(count == rows.size(), "new cursor retries from its explicit start");
    }
    check(budget->used() == 0, "scan-failure charges released");
    std::cout << "scan failure: allocation/cap rejection; terminal failed cursor; explicit restart\n";
}
```

This is why a successful prefix followed by an exception is not a complete query
answer. A future materializing operator must discard or mark its partial output
on failure. A streaming consumer must report the failure to its caller. Silently
turning every read error into end-of-range would make corruption or memory pressure
look like a valid smaller result.

## Let the last cursor outlive every obvious owner

`std::weak_ptr` observes shared ownership without keeping the object alive. Its
`expired()` query becomes true when no shared owner of the object remains. It does
not prove that the weak control block itself has been deallocated, nor is it a
synchronization mechanism for access to `Budget`. Here it lets the test detect a
lifetime cycle or a missing owner without extending the budget's life itself.
See the [weak-pointer observer rules](https://eel.is/c++draft/util.smartptr.weak.obs).

Declare the escaped view and cursor outside the database scope. Inside, reset the
caller's last explicit budget handle **before** destroying the database. Afterwards,
the view must still perform a new lookup and construct a new table cursor. Merely
reading previously copied bytes would not test the cache/budget/file dependency
chain. Then release the view but retain the cursor, check that a competing open
cannot take the directory lock, and advance that cursor once more.

The second key is `UINT64_MAX`. An absent upper bound must include it without
computing an overflowing `key + 1` sentinel. Releasing the final cursor must destroy
the context and budget without a cycle; reopening with a different budget must
then succeed.

```cpp title="tests/read_view.cpp"
void check_escaped_owners() {
    TemporaryDirectory path;
    std::weak_ptr<memory::Budget> observer;
    ReadView escaped;
    ReadCursor cursor;
    {
        auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
        observer = budget;
        Database db{budget, path.name(), DatabaseOpen::create, tiny_options()};
        const std::array batch{Mutation::put(Account{0, 4}),
            Mutation::put(Account{std::numeric_limits<std::uint64_t>::max(), 5})};
        commit(db, batch); (void)db.rotate(); (void)db.flush_oldest();
        escaped = db.snapshot();
        cursor = escaped.scan({Relation::accounts, 0});
        budget.reset();  // No external root owner remains.
    }
    check(!observer.expired() && region(escaped, 0) == 4, "view owns budget/cache/files after DB");
    auto last = escaped.scan({Relation::accounts, std::numeric_limits<std::uint64_t>::max()});
    check(last.valid() && decode_account(last.entry().value).region_id == 5, "maximum key has no +1 sentinel");
    last.reset();
    escaped.reset();
    auto other = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    expect<io::IoError>([&] { Database refused{other, path.name(), DatabaseOpen::recover, tiny_options()}; });
    check(other->used() == 0, "lock-refused context cleans up");
    check(cursor.valid() && decode_account(cursor.entry().value).region_id == 4, "cursor owns view");
    cursor.next();
    check(cursor.valid() && decode_account(cursor.entry().value).region_id == 5, "cursor advances after DB");
    cursor.reset();
    check(observer.expired(), "last cursor releases budget and context without cycle");
    {
        Database reopened{other, path.name(), DatabaseOpen::recover, tiny_options()};
        auto empty = ReadView{};
        expect<std::logic_error>([&] { (void)empty.sequence(); });
        auto view = reopened.snapshot();
        auto moved = std::move(view);
        check(!view.valid() && region(moved, 0) == 4, "moved-from view is empty");
    }
    check(other->used() == 0, "escaped-owner charges released");
    std::cout << "ownership: views and cursors outlive Database; lock and budget release at last owner\n";
}
```

The lock assertion is intentionally stronger than merely preserving open table
file descriptors. Retaining the context prevents another cooperating writer from
reopening the database and changing its namespace while these old readers are
alive. This conservative policy is acceptable for this single-writer local engine.
A design that allows reopening alongside detached views would need a different
cross-instance lifetime and reclamation protocol; it is not obtained by dropping
the lock member early.

Finish the file with its exception boundary. No assertion depends on `assert`, so
Release builds run the same logical checks.

```cpp title="tests/read_view.cpp"
}  // namespace
int main() {
    try {
        check_repeatable();
        check_many_sources();
        check_retention();
        check_capture_failures();
        check_scan_failure();
        check_escaped_owners();
        std::cout << "read views: all checks passed; retained charges returned to zero\n";
        return 0;
    } catch (const std::exception& error) {
        std::cerr << "read-view: " << error.what() << '\n';
        return 1;
    }
}
```

## Add the source and run the complete suite

Append these lines to the learner's root `CMakeLists.txt`. The library already
contains the modified `database.cpp`; only the new source needs adding. Existing
test targets, warning options, and Python native-file checks stay in place.

```cmake title="CMakeLists.txt"
target_sources(strata-core PRIVATE src/storage/read_view.cpp)
add_executable(strata-read-view-tests tests/read_view.cpp)
target_link_libraries(strata-read-view-tests PRIVATE strata-core)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(strata-read-view-tests PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion)
endif()
add_test(NAME read-view COMMAND strata-read-view-tests)
```

From the learner root, rebuild both normal configurations. Reuse compatible
CMake/C++20 tools from Chapter 1; the paths below do not require one particular
compiler release. On the target Mac, use the native toolchain and inspect its
actual architecture as Chapter 1 describes rather than assuming the shell is
native just because the machine has an M4 processor.

```bash title="build and test"
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug --parallel 4
ctest --test-dir build/debug --output-on-failure
./build/debug/strata-read-view-tests

cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/release --parallel 4
ctest --test-dir build/release --output-on-failure
```

Run the sanitizer configuration separately. Do not combine these flags with a
previous incompatible sanitizer build. Leak checking here was exercised on Linux;
use the sanitizer facilities actually available in the native Apple toolchain
rather than treating a platform's unavailable leak option as an engine failure.

```bash title="address and undefined-behavior checks"
cmake -S . -B build/sanitize -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build/sanitize --parallel 4
ctest --test-dir build/sanitize --output-on-failure
```

The new executable produced this output in the Linux authoring check:

```text title="observed read-view output"
snapshot fixture: old=(1,3,2,750),(2,3,1,700); new=(3,3,2,1400)
merge oracle: 1536 versions; 8 retained views; 512 point checks; 48 ranges
retention: copies share quota; cursors retain quota; rejection then release/retry
capture: 3 allocation failures; post-install publication allocates nothing
scan failure: allocation/cap rejection; terminal failed cursor; explicit restart
ownership: views and cursors outlive Database; lock and budget release at last owner
read views: all checks passed; retained charges returned to zero
```

All **seventeen cumulative CTest entries passed in Debug, Release, and combined
AddressSanitizer/UndefinedBehaviorSanitizer builds with Clang 17 on Linux**. The
three allocation failures in the output are an observation of that build, not a
portable constant. The logical assertions, retained-source bounds, and eventual
zero charged allocation remain the portable requirements.

Native M4/macOS execution has not been performed here. These new checks exercise
ordinary native files and controlled exceptions, not physical power loss or
concurrent access. The earlier database tests still run, including their process
exit and lock scenarios; this chapter does not upgrade those experiments into a
device-durability claim. Charged allocation is not process resident memory.

## A wrong comparator can remain perfectly memory-safe

Temporarily replace `Scan::before` with the following diagnostic version. It adds
one early condition before the two correct internal-key comparisons; identical
internal keys still use the original source-index tie breaker.

```cpp
bool Scan::before(std::size_t a, std::size_t b) const {
    const auto x = positions[a].entry().key;
    const auto y = positions[b].entry().key;
    if (x.user == y.user && x.sequence != y.sequence) {
        return x.sequence < y.sequence;  // Deliberately wrong: oldest first.
    }
    if (InternalLess{}(x, y)) { return true; }
    if (InternalLess{}(y, x)) { return false; }
    return a < b;
}
```

This still orders user keys correctly. It still forms a valid heap. Every pointer
can remain within its allocation. But the first eligible row in a group is now the
oldest one, so newer values and tombstones lose. Rebuild and run only the
`read-view` test in Debug, Release, and the sanitizer configuration. The observed
failure was `read-view: new fixture answer` in all three, with no sanitizer memory
error. The semantic test catches what memory instrumentation cannot.

Restore the original `Scan::before` from Part 2 and rerun **all** seventeen tests before
continuing. The canonical listings contain the correct comparator, not the defect.
Do not invert the complete comparison: reversing user-key order would test a
different failure and obscure the version-priority lesson.

## Completion boundary

The finished chapter has a real immutable serving set, not a mutable database
pointer plus a sequence label. Capture either rejects before rotation, completes
publication without allocation after installation, or follows the existing
uncertain-owner stop/reopen rule. Views and cursors retain the resources they
actually use, with limits on admitted states, distinct retained sources, and
charged allocations. Range reads own one output row and bounded source positions;
they do not collect the database into a map.

Before advancing, explain why a cursor at end-of-range still occupies a retention
slot; why two handles to one view do not consume two state slots; why a flushed
arena and its table count as two retained sources; and why copying a candidate
before `source.next()` is necessary even though the final public row owns bytes.
Use the existing tests to answer each question by observing ownership or output,
not by adding elapsed hours to the estimate.

No file is deleted by this chapter. Snapshot lifetime supplies one prerequisite
for safe reclamation, not a complete reclamation policy. The next chapter builds
streaming compaction and connects obsolete-file cleanup to installed authority and
remaining readers. It must not use this public, tombstone-suppressing cursor for a
partial-source compaction that still needs tombstones to hide unmerged older data.

## References used for the ownership checks

The C++ working draft's [shared-pointer construction rules](https://eel.is/c++draft/util.smartptr.shared.const),
[shared allocation rules](https://eel.is/c++draft/util.smartptr.shared.create), and
[shared-pointer destruction rules](https://eel.is/c++draft/util.smartptr.shared.dest)
explain which owners survive each scope. The
[weak-pointer observers](https://eel.is/c++draft/util.smartptr.weak.obs) explain
what `expired()` actually tests. These references were read for the lifetime
boundaries; the file-installation protocol remains the explicit implementation
and assumptions from Chapter 11. Accessed 2026-11-06.
