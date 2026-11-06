# Chapter 22 — Recovery and resource-pressure experiments

## Part 2 — Recover an allowed history, not a convenient answer

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-22.html) supplies the
optional directory boundary and synchronous persistence model. Keep those edits
and both model files. This part adds the fixtures and the complete model test
executable. All parts share the chapter's 4.0-hour implementation allocation.

## Make the answer distinguish all-or-nothing behavior

Use three short batches rather than random traffic. The first establishes both
relations. The second updates, deletes, and inserts. The third changes both
relations again. An atomicity failure therefore has several independent ways
to become visible: wrong cents, an old account region, a resurrected event,
a missing account, or a value carrying the wrong sequence.

```cpp title="tests/recovery_fixture.hpp — complete file, mutations and assertions"
#ifndef STRATA_TEST_RECOVERY_FIXTURE_HPP
#define STRATA_TEST_RECOVERY_FIXTURE_HPP
#include "strata/storage/database.hpp"

#include <array>
#include <cstdint>
#include <optional>
#include <span>
#include <stdexcept>
#include <string_view>

namespace recovery_test {
using namespace strata;
using namespace strata::storage;
inline void require(bool ok, const char* what) { if (!ok) { throw std::runtime_error(what); } }
inline auto first_batch() {
    return std::array{Mutation::put(Account{10, 1}), Mutation::put(Account{20, 2}),
        Mutation::put(Event{1, 10, 3, 7}), Mutation::put(Event{2, 20, 3, 9})};
}
inline auto second_batch() {
    return std::array{Mutation::put(Account{10, 4}), Mutation::put(Event{1, 10, 3, 11}),
        Mutation::erase({Relation::events, 2}), Mutation::put(Event{3, 20, 4, 5})};
}
inline auto third_batch() {
    return std::array{Mutation::erase({Relation::accounts, 20}), Mutation::put(Event{1, 10, 3, 13}),
        Mutation::erase({Relation::events, 3}), Mutation::put(Account{30, 6}),
        Mutation::put(Event{4, 30, 4, 17})};
}
```

These mutations reuse the real `Mutation::put` encodings; they do not bypass the
WAL. Expected values below are handwritten from their meanings. The table
limits keep files small enough for the model's 16 KiB inode capacity. The arena,
cache, and output caps remain real engine limits, not test-only replacements.

```cpp title="tests/recovery_fixture.hpp — complete file, bounded options and row checks"
inline DatabaseOptions options() {
    DatabaseOptions value; value.arena_bytes = 256 * 1024;
    value.cache_slots = 2; value.cache_bytes = 256;
    value.table = {128, 64, 32, 8192}; return value;
}
inline CompactionOptions compact_options() {
    CompactionOptions value; value.output = options().table;
    value.max_output_files = 4; value.max_output_bytes = 32768; return value;
}
inline void committed(Database& db, std::span<const Mutation> changes, std::uint64_t sequence) {
    const auto r = db.write(changes);
    require(r.outcome == WriteOutcome::committed && r.sequence == sequence, "fixture commit");
}
inline void account(Database& db, std::uint64_t id, std::optional<std::uint32_t> region,
                    std::uint64_t sequence = 0) {
    const auto value = db.get({Relation::accounts, id});
    require(value.has_value() == region.has_value(), "account presence/atomicity");
    if (value) {
        require(decode_account(value->view().value) == Account{id, *region} &&
                value->key.sequence == sequence, "account value/version");
    }
}
inline void event(Database& db, std::uint64_t id, std::optional<Event> expected,
                  std::uint64_t sequence = 0) {
    const auto value = db.get({Relation::events, id});
    require(value.has_value() == expected.has_value(), "event presence/atomicity");
    if (value) {
        require(decode_event(value->view().value) == *expected &&
                value->key.sequence == sequence, "event value/version");
    }
}
```

After sequence 2, event 1 has value `(10,3,11)` at version 2, event 3 has
`(20,4,5)` at version 2, account 10 has region 4 at version 2, and account 20
still has region 2 at version 1. After sequence 3, event 1 has cents 13 at
version 3, event 4 and account 30 exist at version 3, while event 3 and account 20
are absent. Account 10 remains at version 2. Event 2 remains absent in both
states.

`Ledger::permits` admits only the acknowledged state, plus the single explicitly
attempted successor when it has not been acknowledged. It does not allow an
arbitrary lower sequence merely because recovery managed to parse it.

```cpp title="tests/recovery_fixture.hpp — complete file, exact histories and external ledger"
// Explicit expected rows, not a replay through the engine's own mutation algorithm.
inline void check_rows(Database& db, std::uint64_t sequence) {
    require(db.sequence() == sequence && (sequence == 2 || sequence == 3), "recovered sequence");
    account(db, 10, 4, 2); account(db, 20, sequence == 2 ? std::optional<std::uint32_t>{2} : std::nullopt, 1);
    account(db, 30, sequence == 3 ? std::optional<std::uint32_t>{6} : std::nullopt, 3);
    event(db, 1, Event{1, 10, 3, sequence == 2 ? 11U : 13U}, sequence);
    event(db, 2, std::nullopt);
    event(db, 3, sequence == 2 ? std::optional<Event>{{3, 20, 4, 5}} : std::nullopt, 2);
    event(db, 4, sequence == 3 ? std::optional<Event>{{4, 30, 4, 17}} : std::nullopt, 3);
    event(db, 99, std::nullopt); account(db, 99, std::nullopt);
}
struct Ledger {
    std::uint64_t acknowledged = 2;
    bool third_attempted = false;
    bool permits(std::uint64_t recovered) const noexcept {
        return recovered == acknowledged ||
            (acknowledged == 2 && third_attempted && recovered == 3);
    }
};
}  // namespace recovery_test
#endif
```

## Seed the shape of storage, not just its rows

The same logical sequence 2 can reside in an active WAL, a sealed WAL, two delta
tables, or a compacted base. Different representations stress different failure
chains. `seed` creates each representation through real database operations and
checks that all charged owners and handles are gone before returning the model.

`authority` decodes manifest bytes only to locate fixture files for controlled
damage. It does not determine the oracle's allowed sequence or expected rows.
`on` passes the shared model owner into the newly added boundary; the string
`"model"` is a nonempty diagnostic path and does not name a native directory.

```cpp title="tests/recovery.cpp — complete file, test setup and locations"
#include "persistence_model.hpp"
#include "recovery_fixture.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <iostream>
#include <memory>
#include <string>
#include <system_error>

namespace {
using namespace recovery_test;
constexpr std::size_t budget_bytes = 8 * 1024 * 1024;
std::shared_ptr<memory::Budget> budget() { return std::make_shared<memory::Budget>(budget_bytes); }
DatabaseOptions on(const std::shared_ptr<PersistenceModel>& fs) {
    auto value = options(); value.directory_backend = fs; return value;
}
Manifest authority(const std::shared_ptr<PersistenceModel>& fs) {
    return decode_manifest(fs->volatile_bytes("MANIFEST"));
}
std::string log_name(const Manifest& m, std::size_t index) {
    return io::file_name(m.logs[index].id, io::FileKind::log).data();
}
```

For the repair seed, append exactly the first seven bytes of a valid encoded
batch 3 and make that incomplete tail persistent. Chapter 10's 32-byte WAL header
plus the first two frames has length

$$
32+172+158=362\text{ bytes}.
$$

The damaged tail makes it 369 bytes. Recovery must retain sequence 2, truncate
back to 362, and sync the repaired file. It must also survive a second cut during
that repair. Seven zero bytes are a different experiment: once enough header
bytes exist to reject the magic, the decoder reports corruption. Calling every
short suffix “a valid torn append” would weaken the recovery contract.

```cpp title="tests/recovery.cpp — complete file, seed real storage representations"
std::shared_ptr<PersistenceModel> seed(std::string_view scenario) {
    auto fs = std::make_shared<PersistenceModel>(); auto b = budget();
    {
        Database db{b, "model", DatabaseOpen::create, on(fs)};
        committed(db, first_batch(), 1);
        if (scenario == "compact" || scenario == "collect") {
            require(db.rotate() && db.flush_oldest(), "first delta");
        }
        committed(db, second_batch(), 2);
        if (scenario == "flush" || scenario == "compact" || scenario == "collect") {
            require(db.rotate(), "seed rotation");
        }
        if (scenario == "compact" || scenario == "collect") {
            require(db.flush_oldest(), "second delta");
        }
        if (scenario == "collect") { db.compact_prefix(2, compact_options()); }
    }
    require(b->used() == 0 && !fs->locked() && fs->handles() == 0, "seed leaked owners");
    if (scenario == "repair") {
        const auto m = authority(fs); const auto name = log_name(m, m.log_count - 1);
        const auto frame = wal_format::encode_batch(*b, 3, third_batch());
        fs->append_garbage(name, 7);
        std::copy_n(frame.begin(), 7, fs->volatile_bytes(name).end() - 7);
        fs->restart({Bytes::current, true});  // The incomplete tail is already persistent.
    }
    return fs;
}
```

## Keep the ledger outside the crashed owner

The ordinary append sets `third_attempted` before entering `write`. Only a
committed result advances `acknowledged`; the following checkpoint represents
a cut after that observation. If the model cut causes an uncertain result, the
still-live but poisoned database must reject another write. We do not retry the
same batch automatically.

An exception from construction, maintenance, or the acknowledgement checkpoint
passes through the same outer cleanup check. The catch runs after the local
`Database` has unwound, so it can require zero charged bytes, zero handles, and
an unlocked model. That checks host-side exception safety separately from the
frozen disk image.

```cpp title="tests/recovery.cpp — complete file, one operation under a chosen cut"
void run(const std::shared_ptr<PersistenceModel>& fs, std::string_view scenario, Ledger& ledger) {
    auto b = budget();
    try {
        Database db{b, "model", DatabaseOpen::recover, on(fs)};
        if (scenario == "append") {
            ledger.third_attempted = true;
            const auto result = db.write(third_batch());
            if (result.outcome == WriteOutcome::committed) {
                ledger.acknowledged = result.sequence; fs->checkpoint();
            } else {
                require(fs->crashed() && result.outcome == WriteOutcome::uncertain, "unexpected append outcome");
                require(db.write(third_batch()).outcome == WriteOutcome::rejected, "uncertain writer stayed writable");
            }
        } else if (scenario == "rotate") { require(db.rotate(), "rotate did no work"); }
        else if (scenario == "flush") { require(db.flush_oldest(), "flush did no work"); }
        else if (scenario == "compact") { db.compact_prefix(2, compact_options()); }
        else if (scenario == "collect") {
            const auto r = db.collect_obsolete();
            require(fs->crashed() || (r.failed == 0 && !r.sync_failed && r.removed > 0), "collection failed");
        } else if (scenario != "repair") { throw std::invalid_argument("scenario"); }
    } catch (...) {
        require(b->used() == 0 && fs->handles() == 0 && !fs->locked(), "failed operation leaked");
        throw;
    }
    require(b->used() == 0 && fs->handles() == 0 && !fs->locked(), "operation leaked");
}
```

Recovery is also allowed to fail while constructing owners, so it needs the same
cleanup check. On successful open, compare the external ledger first, then the
exact logical values and versions. Returning a sequence number alone would
leave most atomicity defects undetected.

```cpp title="tests/recovery.cpp — complete file, reopen and validate the complete state"
std::uint64_t recover(const std::shared_ptr<PersistenceModel>& fs, const Ledger& ledger) {
    auto b = budget(); std::uint64_t sequence = 0;
    try {
        Database db{b, "model", DatabaseOpen::recover, on(fs)};
        sequence = db.sequence();
        require(ledger.permits(sequence), "acknowledged history lost or unattempted batch appeared");
        check_rows(db, sequence);
    } catch (...) {
        require(b->used() == 0 && fs->handles() == 0 && !fs->locked(), "failed recovery leaked");
        throw;
    }
    require(b->used() == 0 && fs->handles() == 0 && !fs->locked(), "recovery leaked");
    return sequence;
}
```

## Enumerate cuts in a reproducible trace

An uncut probe records the number of persistent effects in one operation. For
each effect index, clone the same closed seed, arm that cut, and execute again.
A non-crash `system_error` is not accepted as a successful injection. Require
that the requested cut was reached and that no later persistent effect occurred.
Then choose one of nine survival profiles and reopen twice.

The nine profiles are `(stable,current)` combinations for bytes and names, a
five-byte append prefix with current names, two alternating-inode profiles, and
two rename-only profiles. They are intentionally enumerated, not randomly chosen.
A failure reports scenario, cut index, image index, and effect category. This is
sufficient to reproduce one failing chain without storing a large execution log.

```cpp title="tests/recovery.cpp — complete file, all selected boundaries and images"
void crash_campaign() {
    constexpr std::array images{Image{Bytes::stable, false}, Image{Bytes::current, false},
        Image{Bytes::stable, true}, Image{Bytes::current, true}, Image{Bytes::prefix, true},
        Image{Bytes::alternating, false}, Image{Bytes::alternating, true},
        Image{Bytes::stable, true, 5, true}, Image{Bytes::current, true, 5, true}};
    std::size_t cases = 0; std::array<std::size_t, 8> effects{};
    for (const auto scenario : {"append", "rotate", "flush", "compact", "collect", "repair"}) {
        auto initial = seed(scenario); auto probe = initial->clone(); Ledger ordinary;
        probe->arm(); run(probe, scenario, ordinary); const auto steps = probe->steps();
        require(steps != 0, "empty crash campaign");
        for (std::size_t cut = 1; cut <= steps; ++cut) {
            for (std::size_t image = 0; image < images.size(); ++image) {
                auto fs = initial->clone(); Ledger ledger; fs->arm(cut);
                try { run(fs, scenario, ledger); }
                catch (const std::system_error&) { if (!fs->crashed()) { throw; } }
                require(fs->crashed() && fs->steps() == cut, "cut was not reached or storage changed afterward");
                const auto boundary = fs->last(); ++effects[static_cast<std::size_t>(boundary.effect)];
                fs->restart(images[image]);
                try {
                    const auto answer = recover(fs, ledger);
                    require(recover(fs, ledger) == answer, "second reopen changed recovered history");
                } catch (const std::exception& error) {
                    throw std::runtime_error(std::string{scenario} + " cut=" + std::to_string(cut) +
                        " image=" + std::to_string(image) + " effect=" +
                        std::to_string(static_cast<int>(boundary.effect)) + ": " + error.what());
                }
                ++cases;
            }
        }
        std::cout << scenario << ": boundaries=" << steps << " images=" << steps * images.size() << '\n';
    }
    for (auto count : effects) { require(count != 0, "unexercised persistent effect"); }
    std::cout << "persistence: " << cases << " crash images; every effect covered; two reopens each\n";
}
```

For the shown implementation, effect counts are 30 for append, 31 for rotation,
139 for flush, 94 for compaction, 8 for collection, and 4 for tail repair.
Therefore the selected campaign contains

$$
9(30+31+139+94+8+4)=2754\text{ crash images}.
$$

Each is reopened twice. A changed implementation may alter the effect count;
the test discovers it from its probe rather than hard-coding these totals.
The `effects` array only checks that all eight listed effect categories occur
somewhere in these traces. It is not coverage of every possible crash schedule.

The second reopen matters. The first may repair a tail or stabilize a manifest.
Those changes must themselves yield the same logical answer when opened again.
A recovery routine that succeeds once but leaves a differently interpreted image
has not completed its job.

## A failed sync can still have persisted the batch

Batch 3 is 172 bytes. After injecting a failed sync, enumerate every correct
append prefix from zero through 172 bytes. For prefix $p<172$, the active tail
is incomplete and the recovered state must be $D_2$. For $p=172$, the full valid
frame survives and recovery must produce $D_3$, although no success was returned
to the writer. The number of prefix cases is $172+1=173$.

Separately, corrupt the complete final frame. Its length is no longer evidence
that its payload is valid. Require `FormatError`, not a successful rollback that
silently erases an arbitrary complete frame. Finally, make a failing file-sync
call either copy to stable storage or leave stable storage unchanged. Both calls
return the same error, yet the next boot can legitimately recover different
complete histories.

```cpp title="tests/recovery.cpp — complete file, uncertainty, all append prefixes, and corruption"
void failed_sync_and_prefixes() {
    auto fs = seed("append"); const auto m = authority(fs);
    const auto name = log_name(m, m.log_count - 1);
    const auto old_length = fs->volatile_bytes(name).size(); auto b = budget();
    {
        Database db{b, "model", DatabaseOpen::recover, on(fs)};
        fs->fail_file_sync = true;
        const auto r = db.write(third_batch());
        require(r.outcome == WriteOutcome::uncertain && r.io_error == EIO && db.sequence() == 2,
                "sync error must not publish");
    }
    require(b->used() == 0, "failed-sync owners");
    const auto added = fs->volatile_bytes(name).size() - old_length;
    require(added == 172, "fixture batch width changed");
    const Ledger uncertain{2, true};
    for (std::size_t prefix = 0; prefix <= added; ++prefix) {
        auto image = fs->clone(); image->restart({Bytes::prefix, true, prefix});
        require(recover(image, uncertain) == (prefix == added ? 3U : 2U), "prefix atomicity");
    }
    auto torn = fs->clone(); torn->restart({Bytes::torn, true});
    bool rejected = false;
    try { (void)recover(torn, uncertain); } catch (const FormatError&) { rejected = true; }
    require(rejected, "complete corrupt final frame was silently discarded");
    for (bool persists : {false, true}) {
        auto image = seed("append"); auto owner = budget();
        {
            Database db{owner, "model", DatabaseOpen::recover, on(image)};
            image->fail_file_sync = true; image->failed_sync_persists = persists;
            require(db.write(third_batch()).outcome == WriteOutcome::uncertain, "failure reported success");
        }
        image->restart({Bytes::stable, false});
        require(recover(image, uncertain) == (persists ? 3U : 2U), "failed sync incorrectly implies no persistence");
    }
    std::cout << "uncertainty: 173 append prefixes; failed sync persists or disappears; torn complete frame rejected\n";
}
```

This distinction separates **availability under the stated valid-prefix model**
from **detection of damaged data**. A complete corrupt required frame deliberately
prevents automatic opening. A repair tool would need a separate policy and
explicit data-loss decisions; widening this chapter's normal replay to skip it
would conceal rather than solve the failure.

## Preparation pressure is not write uncertainty

Exhaust the charged budget before constructing a new batch: the write is
definitely rejected, the old state remains readable, and the database is not
poisoned. In contrast, an error once the append boundary is entered is classified
as uncertain. The generic byte boundary does not promise that an error implies
zero side effects, even when this particular injected case used allowance zero.

The 19-byte allowance forces multiple successful short writes before `ENOSPC`.
Recovery of the current image must still exclude the incomplete batch. Allocation
failure during reopening is different again: construction fails, all partial
owners disappear, and retry without the allocation fault can recover the same
acknowledged state.

```cpp title="tests/recovery.cpp — complete file, no-space, allocation sweep, and reservation pressure"
void pressure_and_required_files() {
    for (const auto allowance : {std::size_t{0}, std::size_t{19}}) {
        auto fs = seed("append"); auto b = budget();
        {
            Database db{b, "model", DatabaseOpen::recover, on(fs)};
            fs->write_allowance = allowance;
            const auto r = db.write(third_batch());
            require(r.outcome == WriteOutcome::uncertain && r.io_error == ENOSPC && db.poisoned(), "no-space classification");
        }
        fs->restart({Bytes::current, true}); require(recover(fs, {2, true}) == 2, "partial no-space batch exposed");
    }
    std::size_t failed_allocations = 0;
    for (std::size_t ordinal = 1; ordinal <= 64; ++ordinal) {
        auto fs = seed("compact"); auto b = budget(); b->fail_allocation_on(ordinal); bool failed = false;
        try { Database db{b, "model", DatabaseOpen::recover, on(fs)}; check_rows(db, 2); }
        catch (const std::bad_alloc&) { failed = true; }
        require(b->used() == 0 && fs->handles() == 0 && !fs->locked(), "reopen allocation cleanup");
        b->clear_failure(); require(recover(fs, {}) == 2, "allocation error changed acknowledged rows");
        if (!failed) { break; }
        ++failed_allocations;
    }
    require(failed_allocations >= 4 && failed_allocations < 64, "allocation sweep did not reach success");
    auto fs = seed("append"); auto b = budget();
    {
        Database db{b, "model", DatabaseOpen::recover, on(fs)};
        auto held = b->reserve(b->limit() - b->used());
        const auto r = db.write(third_batch());
        require(r.outcome == WriteOutcome::rejected && !db.poisoned() && db.sequence() == 2,
                "preparation pressure not definite rejection");
    }
    require(b->used() == 0 && recover(fs, {}) == 2, "admission reservation leaked");
```

The allocation sweep advances the failure ordinal until construction succeeds;
its numeric count can depend on implementation details. It covers 34 failing
allocation positions in the tested implementation. The initial fixture uses two
delta tables so reopening allocates real metadata, readers, and recovered state.
The final clean retry is as important as the thrown `bad_alloc`.

Required-file damage is not an invitation to search for any older file that
happens to parse. A manifest has one authority. Corrupt its magic, corrupt a
referenced table, remove a required log, or shorten a sealed log by an entire
complete batch. Each must fail closed. A sealed file's expected final sequence
comes from the manifest; “the shorter file ends on a valid frame boundary” is
not sufficient to accept it.

```cpp title="tests/recovery.cpp — complete file, required-file failures"
    for (const auto target : {"manifest", "table", "sealed", "missing"}) {
        auto damaged = seed(target == std::string_view{"table"} ? "compact" : "flush");
        const auto m = authority(damaged);
        if (target == std::string_view{"manifest"}) { damaged->volatile_bytes("MANIFEST")[0] ^= std::byte{1}; }
        else if (target == std::string_view{"table"}) {
            damaged->volatile_bytes(io::file_name(m.tables[0].id, io::FileKind::table).data())[0] ^= std::byte{1};
        } else if (target == std::string_view{"sealed"}) {
            io::Directory directory{"model", damaged};
            auto opened = directory.open(log_name(m, 0).c_str(), io::OpenMode::read_write);
            opened.file.truncate_to(wal_format::file_header_bytes + 172); // remove all of batch 2
        } else { damaged->erase_name(log_name(m, 0)); }
        bool rejected = false;
        try { (void)recover(damaged, {}); } catch (const FormatError&) { rejected = true; }
        catch (const std::system_error& error) { rejected = error.code().value() == ENOENT; }
        require(rejected, "required file damage ignored");
    }
    std::cout << "pressure: ENOSPC at 0/19 bytes; " << failed_allocations
              << " reopen allocation failures; reserved budget rejected; required files fail closed\n";
}
```

## Reclamation can succeed logically while cleanup remains uncertain

First deny every unlink. The collector must report a refusal rather than
claiming removed bytes. Then permit unlink but make the following directory
sync fail. Select both possible persistence outcomes: the deleted names return
on restart, or they remain absent. In either case the installed database state
must still be $D_2$, because those names were obsolete before deletion.

A surviving orphan is safe to ignore as authority, but its identity is still
important. Create invalid temporary file 900, synchronize its contents and name,
and reopen. The database counts the orphan without replaying it. Collection must
persist a `next_id` fence greater than 900 **before** deleting the only directory
evidence of that identity. After another restart, rotation must not reuse 900.
This tests the protocol from Chapter 13 rather than replacing it with a counter
inside the test.

```cpp title="tests/recovery.cpp — complete file, cleanup outcomes, identity fencing, and main"
void orphan_and_cleanup() {
    for (bool persists : {false, true}) {
        auto fs = seed("collect"); auto b = budget();
        {
            Database db{b, "model", DatabaseOpen::recover, on(fs)};
            fs->fail_unlink = true;
            const auto denied = db.collect_obsolete();
            require(denied.failed != 0 && denied.first_error == EACCES, "deletion refusal hidden");
            fs->fail_unlink = false; fs->fail_directory_sync = true;
            fs->failed_directory_sync_persists = persists;
            const auto uncertain = db.collect_obsolete();
            require(uncertain.removed != 0 && uncertain.sync_failed, "cleanup namespace uncertainty hidden");
        }
        fs->restart({Bytes::stable, false}); require(recover(fs, {}) == 2, "cleanup lost authority");
        {
            Database db{b, "model", DatabaseOpen::recover, on(fs)};
            require((db.orphans_at_open() == 0) == persists, "unsynced deletion survival not modeled");
            const auto retry = db.collect_obsolete(); require(!retry.sync_failed && retry.failed == 0, "cleanup retry");
        }
        require(b->used() == 0, "cleanup owner leaked");
    }
    auto fs = seed("append");
    {
        io::Directory directory{"model", fs};
        auto orphan = directory.open(io::file_name(900, io::FileKind::temporary).data(), io::OpenMode::create_new);
        const std::array bytes{std::byte{0xff}}; orphan.file.write_all(0, bytes);
        orphan.file.sync(io::SyncMode::file); directory.sync();
    }
    auto b = budget();
    {
        Database db{b, "model", DatabaseOpen::recover, on(fs)};
        require(db.orphans_at_open() == 1, "orphan not counted"); check_rows(db, 2);
        require(db.collect_obsolete().removed == 1 && db.manifest().next_id > 900, "identity fence missing");
    }
    fs->restart({Bytes::stable, false});
    {
        Database db{b, "model", DatabaseOpen::recover, on(fs)};
        require(db.rotate() && db.manifest().logs[db.manifest().log_count - 1].id > 900,
                "deleted orphan identity was reused");
    }
    require(b->used() == 0, "orphan cleanup leaked");
    std::cout << "namespace: failed cleanup observable; deleted names may return; orphan 900 never becomes authority or reused\n";
}
}  // namespace
int main() {
    try {
        crash_campaign(); failed_sync_and_prefixes(); pressure_and_required_files(); orphan_and_cleanup();
        std::cout << "recovery model: all checks passed\n";
    } catch (const std::exception& error) {
        std::cerr << "recovery model: " << error.what() << '\n'; return 1;
    }
}
```

## Trace the failure chain across three owners

Consider rotating after sequence 2. The old manifest names the old active WAL.
Rotation creates and synchronizes a new empty WAL, and prepares a candidate
manifest that seals the old WAL and names the new active one. At this point:

| Object | Contents | Namespace or authority |
|---|---|---|
| Old WAL | Acknowledged batches 1 and 2 | Required by the old manifest |
| New WAL | Valid synchronized header, base sequence 2 | New name may still be volatile |
| Candidate manifest | Old sealed log plus new active log | Not yet installed as `MANIFEST` |

The dependency directory sync makes the new WAL name stable. Only afterward
may the installer create, sync, and rename the candidate manifest. A crash after
the rename may keep old or new authority; either authority has all dependencies
needed to recover the acknowledged state.

Now omit just that first directory sync. The candidate's bytes can be stable,
and its rename can survive, while the new WAL's binding disappears. Recovery
reads a perfectly checksummed manifest but receives `ENOENT` opening its new
active log. There is no memory error or data race. The defect is a missing
persistence dependency. The rename-only profile is what makes this chain visible.

Part 3 turns this trace into a controlled diagnostic edit and contrasts it with
a real process kill, where already-created names ordinarily remain in the live
kernel filesystem state.

