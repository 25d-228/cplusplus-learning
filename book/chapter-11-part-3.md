# Chapter 11 — Flush installation and database reopening

## Part 3 — Failure cases across process lifetimes

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-11.html) made the
manifest and directory boundary explicit.
[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-11-part-2.html) built
the installation protocol and serving owners. Now test the difference between
bytes that exist, a catalog that names them, and an object that may safely serve
requests. This is the final part of the same **5.5-hour chapter**, not a second
implementation allocation.

The C++ executable has two jobs. With no arguments it tests the bounded manifest
codec and the strict sealed-log boundary against an in-memory file backend. With
a command and directory it performs a small native database operation. The
Python driver launches that executable repeatedly, decodes the manifest with
independent `struct` and `zlib` operations, damages disposable copies, and starts
a genuinely separate lock contender. It does not implement the engine.

The tests must not erase a real database. The Python driver creates and owns its
entire temporary directory tree. Manual commands below create a new directory
and leave it available for inspection. Never point a corruption exercise at your
only copy of useful data.

## Check format semantics independently of checksums

Create `tests/database.cpp`. It reuses the complete `WalFileOps` test backend from
Chapter 10's `tests/wal_file_ops.hpp`; do not substitute the native backend for
these deterministic sealed-tail checks. No other test helper is missing.

```cpp title="tests/database.cpp"
#include "strata/storage/database.hpp"
#include "wal_file_ops.hpp"

#include <array>
#include <cstdlib>
#include <exception>
#include <iostream>
#include <limits>
#include <stdexcept>
#include <string>
#include <string_view>

namespace {
using namespace strata;
using namespace strata::storage;
void check(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}
template<class Function>
void format_failure(const Function& function) {
    try { function(); }
    catch (const FormatError&) { return; }
    throw std::runtime_error("expected format rejection");
}
```

`format_failure` accepts only the expected `FormatError`. A `bad_alloc`, a native
I/O error, or failure to throw is not a passing corruption check. That distinction
prevents a test from “passing” merely because something unrelated went wrong.

The smallest manifest has zero tables and one active log. Its length is
$48+24+4=76$ bytes. The full bounded manifest has sixteen table records and four
log records: $48+16\cdot32+4\cdot24+4=660$ bytes. Each single-bit mutation below is
restored before trying the next byte position. The test is **660 mutations**, not
all possible corruptions and not all eight bit positions per byte.

```cpp title="tests/database.cpp"
void check_manifest() {
    memory::Budget budget{1024 * 1024};
    Manifest m;
    m.epoch = 7; m.next_id = 2; m.log_count = 1; m.logs[0] = {1, 0, 0};
    {
        auto image = encode_manifest(budget, m);
        check(image.size() == 76, "empty-database manifest length");
        const auto decoded = decode_manifest(image);
        check(decoded.epoch == 7 && decoded.logs[0] == m.logs[0], "manifest round trip");
        for (std::size_t n = 0; n < image.size(); ++n) {
            format_failure([&] { (void)decode_manifest(std::span<const std::byte>{image}.first(n)); });
        }
    }
    m.table_count = Manifest::max_tables;
    m.log_count = Manifest::max_logs;
    m.flushed = m.table_count;
    m.next_id = m.table_count + m.log_count + 1;
    for (std::size_t i = 0; i < m.table_count; ++i) {
        m.tables[i] = {i + 1, i, i + 1, 4096};
    }
    for (std::size_t i = 0; i < m.log_count; ++i) {
        m.logs[i] = {m.table_count + i + 1, m.table_count + i,
                     i + 1 == m.log_count ? 0 : m.table_count + i + 1};
    }
    {
        auto image = encode_manifest(budget, m);
        check(image.size() == 660, "maximum manifest length");
        check(decode_manifest(image).tables[15] == m.tables[15], "last table round trip");
        for (std::size_t i = 0; i < image.size(); ++i) {
            image[i] ^= std::byte{1};
            format_failure([&] { (void)decode_manifest(image); });
            image[i] ^= std::byte{1};
        }
        // A correct checksum does not authorize inconsistent coverage.
        put_le<8>(std::span<std::byte>{image}.subspan(56, 8), 1);
        put_le<4>(std::span<std::byte>{image}.last(4),
                  table_format::crc32(std::span<const std::byte>{image}.first(image.size() - 4)));
        format_failure([&] { (void)decode_manifest(image); });
    }
    const auto bad = [&](auto change) {
        auto candidate = m;
        change(candidate);
        format_failure([&] { validate_manifest(candidate); });
    };
    bad([](Manifest& value) { value.logs[0].id = value.tables[0].id; });
    bad([](Manifest& value) { value.logs[0].base += 1; });
    bad([](Manifest& value) { value.logs[3].end = 22; });
    bad([](Manifest& value) { value.flushed = 15; });
    bad([](Manifest& value) { value.next_id = 20; });
    bad([](Manifest& value) { value.table_count = 17; });
    bad([](Manifest& value) { value.epoch = 0; });
    check(budget.used() == 0, "manifest charge released");
    const auto name = io::file_name(std::numeric_limits<std::uint64_t>::max(), io::FileKind::table);
    check(io::numbered_id(name.data()) == std::numeric_limits<std::uint64_t>::max(), "full id domain");
    check(!io::numbered_id("notes.txt"), "foreign name");
    format_failure([] { (void)io::numbered_id("s-18446744073709551616.tmp"); });
    format_failure([] { (void)io::numbered_id("s-00000000000000000000.wal"); });
}
```

The mutation at offset 56 changes the first table's `base`. Recomputing the CRC
makes its checksum correct while its history coverage is wrong: a first interval
starting at 1 omits the required boundary at 0. The decoder must reject that
image for its semantics. Tests also reject duplicate file identities, a gap
between sources, a nonzero active end, the wrong flushed boundary, exhausted
high-water admission, an excessive table count, and epoch zero.

The filename exercise reaches `UINT64_MAX` without overflowing decimal parsing.
A larger decimal value must fail rather than wrap into an existing identity.
`notes.txt` is foreign to the reserved namespace; zero and malformed reserved
identities are errors, not filenames to silently skip.

## A sealed prefix cannot be repaired as an active tail

The next test writes two complete account batches, then freezes the writer.
It records the boundary after the first batch so the test does not depend on a
hardcoded account-frame size. Reopening as sealed must recover exactly sequence
2 and retain a frozen memtable. Removing a complete last batch is just as invalid
as removing one byte from that batch.

```cpp title="tests/database.cpp"
void check_sealed_replay() {
    memory::Budget budget{1024 * 1024};
    WalFileOps ops;
    const std::array first{Mutation::put(Account{10, 1})};
    const std::array second{Mutation::put(Account{10, 2})};
    std::size_t boundary = 0;
    {
        WalMemTable writer{budget, ops.open(), 64 * 1024, WalOpen::create, io::SyncMode::file};
        check(writer.write(first).outcome == WriteOutcome::committed, "first WAL batch");
        boundary = ops.length;
        check(writer.write(second).outcome == WriteOutcome::committed, "second WAL batch");
        writer.seal();
        check(writer.write(first).reason == WriteReason::writer_sealed, "sealed owner rejects write");
    }
    const auto full = ops.image;
    const auto length = ops.length;
    {
        WalMemTable reader{budget, ops.open(), 64 * 1024, WalOpen::recover_sealed,
                           io::SyncMode::file, 0, 2};
        check(reader.table().frozen() && reader.sequence() == 2, "sealed recovery");
    }
    for (const auto cut : {boundary, length - 1}) {
        ops.set_image(std::span<const std::byte>{full}.first(cut));
        const auto truncates = ops.truncate_calls;
        format_failure([&] {
            WalMemTable reader{budget, ops.open(), 64 * 1024, WalOpen::recover_sealed,
                               io::SyncMode::file, 0, 2};
        });
        check(ops.length == cut && ops.truncate_calls == truncates, "sealed damage not repaired");
        check(!ops.live && budget.used() == 0, "failed sealed constructor cleanup");
    }
    ops.set_image(std::span<const std::byte>{full}.first(length - 1));
    {
        WalMemTable active{budget, ops.open(), 64 * 1024, WalOpen::recover, io::SyncMode::file};
        check(active.sequence() == 1 && active.repaired_tail(), "active tail still repairable");
        check(ops.length == boundary, "active tail truncated to verified frame");
    }
    check(budget.used() == 0, "WAL charges released");
}
```

There are two independent assertions after a failed sealed constructor. The
storage image remains unchanged, including the truncate-call count; the failed
owner also closes its descriptor and releases every charged allocation. Merely
throwing eventually would be insufficient if replay had already shortened a
sealed dependency.

The final active-log open uses the same partial bytes but a different contract.
It is allowed to discard the incomplete final frame and recover sequence 1.
Treating these two modes identically would silently discard manifest-authorized
history. The distinction is a recovery invariant, not a different checksum
algorithm.

## Make interruption observable without changing the algorithm

The installation callback receives a borrowed `Cut` through the `void*` context
introduced in Part 2. The cast is valid because this driver always passes the
address of that exact type and keeps it alive longer than the database. No
callback starts another database operation. `armed` remains false while opening
so an injection intended for rotation does not accidentally fire during initial
creation or recovery.

```cpp title="tests/database.cpp"
struct Injected : std::runtime_error {
    Injected() : std::runtime_error("installation checkpoint") {}
};
struct Cut {
    std::string stage;
    bool armed = false;
    bool abrupt = false;
    bool arm_allocation_after_commit = false;
    bool armed_after_commit = false;
    memory::Budget* budget = nullptr;
};
void checkpoint(void* pointer, std::string_view stage) {
    auto& cut = *static_cast<Cut*>(pointer);
    if (!cut.armed) { return; }
    if (stage == "manifest-closed" && cut.arm_allocation_after_commit) {
        cut.budget->fail_allocation_on(1);
        cut.armed_after_commit = true;
    }
    if (stage == cut.stage) {
        std::cout << "cut=" << stage << '\n' << std::flush;
        if (cut.abrupt) { std::_Exit(73); }
        throw Injected{};
    }
}
```

The exception path throws after a completed boundary and runs normal C++
cleanup. The abrupt path flushes the diagnostic stream and calls `std::_Exit(73)`:
no stack unwinding, database destructor, or normal stream cleanup is relied on.
These are two different process behaviors. Neither removes the running kernel's
page cache, reorders device writes, or models a power interruption.

At `manifest-closed`, a separate mode arms the next charged allocation instead
of interrupting. After the operation returns, a one-byte probe must still trigger
that failure. This checks that publication after durable installation did not
perform another charged allocation. It does not intercept allocations outside
our `Budget` allocator or prove the absence of every possible library allocation.

The data fixture is deliberately small but crosses both relations in each batch.
Sequence $s$ writes account 10's region as $s$ and event 100's amount as
$400+100s$, except that sequence 3 writes an event tombstone. Holding the keys
constant forces newer generations to override older sources.

```cpp title="tests/database.cpp"
void append_fixture(Database& database) {
    const auto sequence = database.sequence() + 1;
    check(sequence < 1000, "fixture sequence range");
    const auto event = sequence == 3 ? Mutation::erase({Relation::events, 100})
        : Mutation::put(Event{100, 10, 3, 400 + 100 * sequence});
    const std::array batch{event, Mutation::put(Account{10, static_cast<std::uint32_t>(sequence)})};
    check(database.write(batch).outcome == WriteOutcome::committed, "fixture write committed");
}
void inspect(Database& database) {
    const auto sequence = database.sequence();
    const auto account = database.get({Relation::accounts, 10});
    const auto event = database.get({Relation::events, 100});
    check(account && decode_account(account->view().value).region_id == sequence, "fixture account");
    check((sequence == 3 && !event) || (sequence != 3 && event &&
          decode_event(event->view().value).amount_cents == 400 + 100 * sequence), "fixture event");
    std::cout << "sequence=" << sequence << " tables=" << database.manifest().table_count
              << " logs=" << database.manifest().log_count << " flushed=" << database.manifest().flushed
              << " epoch=" << database.manifest().epoch << " orphans=" << database.orphans_at_open()
              << " event=" << (event ? "present" : "absent") << '\n';
}
```

The driver checks decoded values, not only the manifest counters. After a
tombstone has moved into an installed table, the older event must remain absent.
A catalog that looks plausible while `get` resurrects that event fails here.
`orphans_at_open()` is the inventory observed when this process opened; it does
not update after each operation. The next process may therefore print one more
orphan even though no additional mutation occurred between the two outputs.

## Keep the native command boundary small

The driver uses an existing directory and an explicit create/recover choice.
`--allocation N` accepts an ordinal from 1 through 128. `std::stoull` can throw;
the final `main` reports conversion and range errors through the same exception
boundary. Numeric parsing is test infrastructure, not an engine format parser.

```cpp title="tests/database.cpp"
int native_driver(int argc, char** argv) {
    if (argc < 3) { throw std::invalid_argument("expected command and existing directory"); }
    const std::string command{argv[1]};
    DatabaseOptions options;
    memory::Budget budget{16 * 1024 * 1024};
    Cut cut;
    cut.budget = &budget;
    std::size_t fail_allocation = 0;
    for (int i = 3; i < argc; ++i) {
        const std::string_view option{argv[i]};
        if (option == "--full") { options.sync_mode = io::SyncMode::full_device; }
        else if (option == "--arm-after-commit") { cut.arm_allocation_after_commit = true; }
        else if ((option == "--fault" || option == "--exit") && i + 1 < argc) {
            cut.stage = argv[++i]; cut.abrupt = option == "--exit";
        } else if (option == "--allocation" && i + 1 < argc) {
            const std::string text{argv[++i]};
            std::size_t consumed = 0;
            const auto value = std::stoull(text, &consumed);
            if (consumed != text.size() || value == 0 || value > 128) {
                throw std::invalid_argument("allocation ordinal must be 1..128");
            }
            fail_allocation = static_cast<std::size_t>(value);
        } else { throw std::invalid_argument("unknown driver option"); }
    }
```

`Cut` and `Budget` outlive `Database`. Failure injection is enabled only after
construction, and the old epoch/sequence are copied before the operation. A
post-install exception may leave a newer manifest on disk, yet this poisoned
object must not pretend its old serving set is current. Diagnostics may inspect
the old state; public reads and writes may not continue.

```cpp title="tests/database.cpp"
    int status = 0;
    {
        Database database{budget, argv[2], command == "create" ? DatabaseOpen::create : DatabaseOpen::recover,
                          options, InstallHook{checkpoint, &cut}};
        const auto old_epoch = database.manifest().epoch;
        const auto old_sequence = database.sequence();
        cut.armed = true;
        if (fail_allocation != 0) { budget.fail_allocation_on(fail_allocation); }
        try {
            if (command == "create") { append_fixture(database); append_fixture(database); }
            else if (command == "append") { append_fixture(database); }
            else if (command == "rotate") { (void)database.rotate(); }
            else if (command == "flush") { (void)database.flush_oldest(); }
            else if (command == "hold") {
                std::cout << "locked\n" << std::flush;
                std::string line;
                std::getline(std::cin, line);
            } else if (command != "inspect") { throw std::invalid_argument("unknown command"); }
            if (cut.armed_after_commit) {
                bool failed = false;
                try { memory::Vector<std::byte> probe(1, memory::Allocator<std::byte>{budget}); }
                catch (const std::bad_alloc&) { failed = true; }
                check(failed, "post-install publication must not allocate");
            }
            budget.clear_failure();
            inspect(database);
        } catch (const Injected&) {
            check(database.poisoned(), "checkpoint failure poisons database");
            check(database.manifest().epoch == old_epoch && database.sequence() == old_sequence,
                  "failed install did not publish serving state");
            const std::array batch{Mutation::put(Account{20, 9})};
            check(database.write(batch).reason == WriteReason::writer_poisoned, "poisoned writer rejects");
            bool refused_read = false;
            try { (void)database.get({Relation::accounts, 10}); }
            catch (const std::logic_error&) { refused_read = true; }
            check(refused_read, "poisoned read refused");
            std::cout << "poisoned=1 rejected=1\n";
            status = 3;
        } catch (const std::bad_alloc&) {
            budget.clear_failure();
            check(database.poisoned(), "installation allocation failure stops writer");
            std::cout << "allocation-failed poisoned=1\n";
            status = 4;
        } catch (const std::length_error& error) {
            check(!database.poisoned(), "slot rejection must precede irreversible region");
            std::cout << "admission=" << error.what() << '\n';
            status = 5;
        }
    }
    check(budget.used() == 0, "database destruction releases all charged storage");
    return status;
}
```

The exception cases have distinct exit codes so the Python driver cannot mistake
an expected installation failure for a successful command. Code 3 means a
checkpoint exception, 4 an injected charged-allocation failure, and 5 a bounded
slot-admission rejection. Ordinary construction or format errors reach `main`
and produce code 1. The abrupt path uses 73 and cannot check in-process cleanup.

`length_error` is expected only for the deliberate pre-effect generation caps
in these fixtures. It must leave the object healthy. By contrast, the chosen
allocation injections occur inside the conservative uncertain maintenance region
and must poison it. The outer scope checks zero charged bytes after destruction
on every normally unwound path.

```cpp title="tests/database.cpp"
}  // namespace

int main(int argc, char** argv) {
    try {
        if (argc == 1) {
            check_manifest();
            check_sealed_replay();
            std::cout << "manifest: 660 corruption positions; sealed replay: strict boundary; charges: zero\n";
            return 0;
        }
        return native_driver(argc, argv);
    } catch (const std::exception& error) {
        std::cerr << "database: " << error.what() << '\n';
        return 1;
    }
}
```

## Decode native authority with a separate implementation

Create `scripts/check_database_fixture.py`. The decoder below does not import
C++ layouts or call back into the executable for metadata. The little-endian
format strings derive directly from Part 1's byte table. In `"<8sIIIIQQQ"`,
`8s` reads eight bytes, each `I` four bytes, and each `Q` eight bytes, with no native
padding. `zlib.crc32` provides an independent check on our CRC implementation.

This is a small-fixture decoder. Its `read_bytes()` calls are not the engine's
bounded streaming path, and it does not replace the C++ table validator. The
native executable separately verifies table framing, row ordering, checksums,
and sequence coverage while opening.

```python title="scripts/check_database_fixture.py"
#!/usr/bin/env python3
"""Check native database installation in disposable directories, never user data."""
from __future__ import annotations

import argparse
from pathlib import Path
import shutil
import struct
import subprocess
import tempfile
import zlib


def require(condition: bool, message: str) -> None:
    if not condition:
        raise RuntimeError(message)


def decode_manifest(directory: Path) -> dict:
    data = (directory / "MANIFEST").read_bytes()
    require(76 <= len(data) <= 660, "independent manifest size")
    magic, version, nt, nl, reserved, epoch, next_id, flushed = struct.unpack_from("<8sIIIIQQQ", data)
    require(magic == b"STRAMAN1" and version == 1 and reserved == 0, "manifest header")
    require(nt <= 16 and 1 <= nl <= 4 and epoch > 0, "manifest counts/epoch")
    require(len(data) == 48 + nt * 32 + nl * 24 + 4, "manifest exact length")
    require(struct.unpack_from("<I", data, len(data) - 4)[0] == zlib.crc32(data[:-4]), "manifest CRC")
    tables = [struct.unpack_from("<QQQQ", data, 48 + i * 32) for i in range(nt)]
    logs = [struct.unpack_from("<QQQ", data, 48 + nt * 32 + i * 24) for i in range(nl)]
    identities = [t[0] for t in tables] + [log[0] for log in logs]
    require(len(set(identities)) == len(identities), "manifest duplicate IDs")
    require(all(0 < identity < next_id for identity in identities), "manifest high-water bound")
    boundary = 0
    for identity, base, end, size in tables:
        require(base == boundary and end > base and 64 <= size <= 64 * 1024 * 1024, "table interval")
        require((directory / f"s-{identity:020}.sst").stat().st_size == size, "referenced table length")
        boundary = end
    require(boundary == flushed, "flushed prefix")
    for index, (identity, base, end) in enumerate(logs):
        require(base == boundary, "log boundary")
        wal = (directory / f"s-{identity:020}.wal").read_bytes()
        require(len(wal) >= 32 and wal[:8] == b"STRWAL01", "referenced WAL header")
        require(struct.unpack_from("<Q", wal, 16)[0] == base, "WAL header base")
        if index + 1 == nl:
            require(end == 0, "active open end")
        else:
            require(end > base, "sealed positive interval")
            boundary = end
    return {"epoch": epoch, "next": next_id, "flushed": flushed,
            "tables": tables, "logs": logs, "bytes": data}
```

For every table, the decoder checks the actual referenced file length. For every
log, it checks the native header's base against the manifest. The active log has
an open end of zero; sealed intervals advance the boundary. Missing references
are errors rather than permission to adopt another file from the directory.

The next blocks continue the same file and the same `main` function. Preserve
indentation across blocks. `run` launches a fresh process with a 60-second test
timeout and requires both its exit status and an identifying output substring.
A timeout is a failing test, not a reason to retry an operation against the same
possibly changed database.

```python title="scripts/check_database_fixture.py"
def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--program", required=True, type=Path)
    parser.add_argument("--full", action="store_true", help="require full-device synchronization")
    args = parser.parse_args()
    program = args.program.resolve()

    def command(action: str, directory: Path, *extra: str) -> list[str]:
        return [str(program), action, str(directory), *extra, *(["--full"] if args.full else [])]

    def run(action: str, directory: Path, *extra: str, code: int = 0, text: str = "") -> subprocess.CompletedProcess:
        result = subprocess.run(command(action, directory, *extra), capture_output=True, text=True, timeout=60)
        output = result.stdout + result.stderr
        require(result.returncode == code, f"{action}: exit={result.returncode}, expected={code}: {output}")
        require(text in output, f"{action}: missing {text!r}: {output}")
        return result

    with tempfile.TemporaryDirectory(prefix="strata-database-") as temporary:
        root = Path(temporary)
        seed = root / "seed"
        seed.mkdir()
        run("inspect", seed, code=1, text="authoritative MANIFEST missing")
        (seed / "notes.txt").write_text("unrelated file\n")
        run("create", seed, text="sequence=2 tables=0 logs=1")
        initial = decode_manifest(seed)
        require(initial["next"] == 3 and initial["logs"] == [(1, 0, 0)], "initial identities")
        old = initial["bytes"]
        run("create", seed, code=1, text="create requires no existing engine files")
        require((seed / "MANIFEST").read_bytes() == old, "exclusive database create")

        serial = 0

        def clone(source: Path) -> Path:
            nonlocal serial
            serial += 1
            destination = root / f"case-{serial}"
            shutil.copytree(source, destination)
            return destination
```

The original fixture is sequence 2 in log 1, with a one-log manifest whose next
unused identity is 3: temporary manifest 2 consumed an identity even though its
name became `MANIFEST`. `clone` copies only closed fixture trees; no running
writer owns a copied directory. Each subsequent case receives its own path and
lock inode.

The successful lifecycle rotates first, commits sequence 3 into the new active
log, flushes the older generation, then rotates and flushes the tombstone's
generation. Reopening must still return absence for event 100. An empty active
memtable makes rotation a true no-op, including unchanged manifest bytes.

```python title="scripts/check_database_fixture.py"
        rotated = clone(seed)
        run("rotate", rotated, "--arm-after-commit", text="logs=2")
        r = decode_manifest(rotated)
        require(r["logs"] == [(1, 0, 2), (3, 2, 0)] and r["next"] == 5, "rotation coverage")
        same = r["bytes"]
        run("rotate", rotated)  # Empty active table does not create another generation.
        require((rotated / "MANIFEST").read_bytes() == same, "empty rotate no-op")
        run("append", rotated, text="sequence=3 tables=0 logs=2")
        flushed = clone(rotated)
        run("flush", flushed, "--arm-after-commit", text="flushed=2")
        f = decode_manifest(flushed)
        require(len(f["tables"]) == 1 and f["logs"] == [(3, 2, 0)], "flush removes oldest dependency")
        require((flushed / "s-00000000000000000001.wal").exists(), "retired WAL not prematurely removed")
        run("rotate", flushed)
        run("flush", flushed, text="sequence=3 tables=2 logs=1 flushed=3")
        run("inspect", flushed, text="event=absent")
        require((flushed / "notes.txt").read_text() == "unrelated file\n", "foreign file preserved")
        print("native lifecycle: ordered flushes, reopen, and disk tombstone suppression passed")
```

Now interrupt at every named install boundary. There are ten common stages and
one operation-specific stage: `log-ready` for rotation or `table-ready` for a
flush. Two operations, eleven stages, and two interruption modes give

$$
2\cdot11\cdot2=44
$$

cases: 22 exception cases and 22 abrupt process exits. The live kernel has not
crashed. Therefore a completed rename is immediately observable by the next
process, even at a checkpoint before directory synchronization. The test expects
the old epoch before rename and the new epoch after it. **That observation must
not be reused as a power-loss recovery rule.**

```python title="scripts/check_database_fixture.py"
        common = ["dependencies-synced", "manifest-created", "manifest-written", "manifest-synced",
                  "before-rename", "manifest-renamed", "before-directory-sync", "directory-synced",
                  "manifest-stabilized", "manifest-closed"]
        cases = 0
        for action, source, first in [("rotate", seed, "log-ready"), ("flush", rotated, "table-ready")]:
            before = decode_manifest(source)
            for option, exit_code in [("--fault", 3), ("--exit", 73)]:
                for stage in [first, *common]:
                    directory = clone(source)
                    run(action, directory, option, stage, code=exit_code, text=f"cut={stage}")
                    run("inspect", directory)
                    after = decode_manifest(directory)
                    renamed = stage in common[5:]
                    require(after["epoch"] == before["epoch"] + int(renamed), f"authority after {stage}")
                    if not renamed:
                        require(after["bytes"] == before["bytes"], f"old authority retained at {stage}")
                    # Reopened maintenance can finish whichever transition survived.
                    if action == "rotate" and len(after["logs"]) == 1:
                        run("rotate", directory)
                    while len(decode_manifest(directory)["logs"]) > 1:
                        run("flush", directory)
                    run("inspect", directory)
                    cases += 1
        print(f"installation checkpoints: {cases} exception/abrupt-exit cases passed")
```

Each reopened database completes whichever maintenance work remains, then checks
its values again. This catches a protocol that reopens once but cannot safely
continue after an interrupted rotation or flush.

The allocation sweep starts from a fresh copy for each ordinal. It requires a
poisoned operation failure, unchanged manifest bytes, and a successful reopen.
The first ordinal beyond the operation's actual allocation count succeeds and
ends the sweep. Counts are observations of this implementation and standard
library, not a portable API contract. No failure is allowed to silently evade
the reopen check.

```python title="scripts/check_database_fixture.py"
        for action, source in [("rotate", seed), ("flush", rotated)]:
            failed = 0
            for ordinal in range(1, 65):
                directory = clone(source)
                result = subprocess.run(command(action, directory, "--allocation", str(ordinal)),
                                        capture_output=True, text=True, timeout=60)
                require(result.returncode in (0, 4), f"unexpected allocation failure: {result.stdout}{result.stderr}")
                run("inspect", directory)
                if result.returncode == 0:
                    break
                failed += 1
                require((directory / "MANIFEST").read_bytes() == (source / "MANIFEST").read_bytes(),
                        "allocation failure changed authority")
            else:
                raise RuntimeError("allocation sweep never reached success")
            require(failed > 0, "allocation seam was exercised")
            print(f"{action} allocation failure points: {failed}")
```

## Reject attractive but unauthorized recovery paths

A valid orphan manifest candidate does not repair a corrupt authoritative
`MANIFEST`. Likewise, an old WAL retained after a flush does not replace a
missing or corrupt table that the current manifest names. The checks below keep
those tempting fallback inputs in the directory and still require failure.

The two native sealed-log cuts remove either the entire last batch or its last
byte. This fixture has two 100-byte cross-relation frames after a 32-byte header,
so its length is $32+100+100=232$ bytes. Cuts at 132 and 231 must leave the damaged
file untouched. A four-byte incomplete active-frame header, by contrast, is a
permitted active tail and is removed before serving.

```python title="scripts/check_database_fixture.py"
        # Missing/corrupt authority is not a request to recreate or use an orphan.
        directory = clone(flushed)
        (directory / "s-00000000000000000900.tmp").write_bytes((directory / "MANIFEST").read_bytes())
        image = bytearray((directory / "MANIFEST").read_bytes())
        image[-1] ^= 1
        (directory / "MANIFEST").write_bytes(image)
        run("inspect", directory, code=1, text="manifest checksum")
        require((directory / "MANIFEST").read_bytes() == image, "corrupt authority unchanged")
        directory = clone(seed)
        (directory / "MANIFEST").unlink()
        run("inspect", directory, code=1, text="authoritative MANIFEST missing")
        run("create", directory, code=1, text="create requires no existing engine files")

        directory = clone(flushed)
        manifest = decode_manifest(directory)
        table_path = directory / f"s-{manifest['tables'][0][0]:020}.sst"
        table_path.unlink()
        run("inspect", directory, code=1, text="openat")
        directory = clone(flushed)
        manifest = decode_manifest(directory)
        table_path = directory / f"s-{manifest['tables'][0][0]:020}.sst"
        image = bytearray(table_path.read_bytes())
        image[40] ^= 1
        table_path.write_bytes(image)
        run("inspect", directory, code=1, text="checksum")

        # A sealed WAL cannot lose a whole final batch, nor repair a partial one.
        for cut in (132, 231):  # This two-batch WAL is 32 + 100 + 100 bytes.
            directory = clone(rotated)
            log_path = directory / "s-00000000000000000001.wal"
            before = log_path.read_bytes()[:cut]
            log_path.write_bytes(before)
            run("inspect", directory, code=1, text="sealed WAL does not match manifest end")
            require(log_path.read_bytes() == before, "sealed damage was not truncated")
        directory = clone(rotated)
        (directory / "s-00000000000000000001.wal").unlink()
        run("inspect", directory, code=1, text="openat")
        directory = clone(seed)
        log_path = directory / "s-00000000000000000001.wal"
        before = log_path.read_bytes()
        log_path.write_bytes(before + b"BAT1")
        run("inspect", directory)
        require(log_path.read_bytes() == before, "active incomplete tail repaired")
        print("reopen rejection: corrupt authority, missing/corrupt table, and sealed WAL damage passed")
```

The next cases check namespace rules and bounded admission. Observed orphan IDs
900 and 901 push the next log to 902, the temporary manifest to 903, and the
persisted next-ID value to 904. Their file contents are deliberately useless;
only their names affect high-water admission. Neither becomes recovery data.

A symlink in the reserved namespace is refused. The generation tests then fill
four log slots and sixteen table slots, respectively. These are format/resource
bounds, not a request to overwrite the oldest live dependency. Flushing the
oldest sealed log frees a log slot; this chapter has no compaction operation to
free a table slot, so table-cap rejection remains the correct result.

```python title="scripts/check_database_fixture.py"
        directory = clone(seed)
        orphan_wal = directory / "s-00000000000000000900.wal"
        orphan_wal.write_bytes(b"not a valid WAL; not a recovery dependency")
        orphan_temp = directory / "s-00000000000000000901.tmp"
        orphan_temp.write_bytes(b"partial manifest")
        run("inspect", directory, text="orphans=2")
        run("rotate", directory)
        manifest = decode_manifest(directory)
        require(manifest["logs"][-1][0] == 902 and manifest["next"] == 904, "observed IDs never reused")
        require(orphan_wal.exists() and orphan_temp.exists(), "orphans preserved")
        directory = clone(seed)
        log_path = directory / "s-00000000000000000001.wal"
        saved = directory / "saved-wal"
        log_path.rename(saved)
        log_path.symlink_to(saved.name)
        run("inspect", directory, code=1, text="engine file must be regular with one link")

        # Bound retained generations; reject a fifth log without changing authority.
        directory = clone(seed)
        for i in range(3):
            run("rotate", directory)
            run("append", directory)
        before = (directory / "MANIFEST").read_bytes()
        run("rotate", directory, code=5, text="sealed-log slots exhausted")
        require((directory / "MANIFEST").read_bytes() == before, "log-slot admission unchanged")
        run("flush", directory)
        run("rotate", directory)
        run("inspect", directory)

        directory = clone(seed)
        for i in range(16):
            run("rotate", directory)
            run("flush", directory)
            run("append", directory)
        run("rotate", directory)
        before = (directory / "MANIFEST").read_bytes()
        run("flush", directory, code=5, text="table slots exhausted")
        require((directory / "MANIFEST").read_bytes() == before, "table-slot admission unchanged")
        run("inspect", directory)
        print("identities and bounds: orphan high-water, symlink refusal, log/table caps passed")
```

The lock check is not two `Directory` objects in one process. A child announces
`locked` only after it has opened the database and retained its lock. A separate
child must fail to open. The holder then receives a line on standard input,
returns normally, and releases the descriptor; the following open succeeds.
`LOCK` remains the same inode throughout. Removing and recreating it would allow
cooperating processes to lock different objects under the same pathname.

```python title="scripts/check_database_fixture.py"
        lock_inode = (seed / "LOCK").stat().st_ino
        holder = subprocess.Popen(command("hold", seed), stdin=subprocess.PIPE, stdout=subprocess.PIPE,
                                  stderr=subprocess.PIPE, text=True)
        try:
            require(holder.stdout is not None, "holder stdout")
            require(holder.stdout.readline().strip() == "locked", "holder readiness")
            run("inspect", seed, code=1, text="lock database directory")
            output, errors = holder.communicate("release\n", timeout=60)
            require(holder.returncode == 0, f"holder exit: {output}{errors}")
        finally:
            if holder.poll() is None:
                holder.kill()
                holder.communicate(timeout=60)
        require((seed / "LOCK").stat().st_ino == lock_inode, "LOCK inode preserved")
        run("inspect", seed)
        print("directory lock: second process refused, later reopen passed")


if __name__ == "__main__":
    main()
```

## Build and run all prerequisite checks

Append the following to the existing root `CMakeLists.txt`. Chapter 8 already
found the Python interpreter, and Chapters 6–10 already configured `strata-core`,
its public headers, and the C++20 requirement. Keep those declarations and all
fourteen earlier tests. Do not create a second library containing duplicate
copies of `file.cpp`, `wal.cpp`, or their symbols.

```cmake title="CMakeLists.txt"
target_sources(strata-core PRIVATE
    src/io/directory.cpp src/storage/manifest.cpp src/storage/database.cpp)
add_executable(strata-database-tests tests/database.cpp)
target_link_libraries(strata-database-tests PRIVATE strata-core)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(strata-database-tests PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion)
endif()
add_test(NAME database COMMAND strata-database-tests)
add_test(NAME database-native
    COMMAND "${Python3_EXECUTABLE}"
            "${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_database_fixture.py"
            --program $<TARGET_FILE:strata-database-tests>)
```

From the learner repository root, configure and run Debug and Release. A
compatible maintained Clang or GCC with the earlier feature checks is sufficient;
these commands do not require an invented LTS release or a specific macOS
version. Use the same compiler family consistently within each build directory.

```sh title="Terminal — complete normal builds"
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug -j 2
ctest --test-dir build/debug --output-on-failure

cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/release -j 2
ctest --test-dir build/release --output-on-failure
```

There are now sixteen CTest entries. Direct execution without arguments should
print:

```text title="Expected — deterministic database checks"
manifest: 660 corruption positions; sealed replay: strict boundary; charges: zero
```

Run the native fixture separately to see its diagnostic summary:

```sh title="Terminal — native process checks"
python3 scripts/check_database_fixture.py --program build/debug/strata-database-tests
```

```text title="Observed — ordinary synchronization on Linux"
native lifecycle: ordered flushes, reopen, and disk tombstone suppression passed
installation checkpoints: 44 exception/abrupt-exit cases passed
rotate allocation failure points: 2
flush allocation failure points: 9
reopen rejection: corrupt authority, missing/corrupt table, and sealed WAL damage passed
identities and bounds: orphan high-water, symlink refusal, log/table caps passed
directory lock: second process refused, later reopen passed
```

The allocation counts above were observed with Clang 17 and libstdc++ on Linux
x86-64. Another standard library can allocate differently. Preserve the sweep's
invariants instead of replacing it with hardcoded counts of two and nine.

On the native M4 machine, additionally request the full-device mode explicitly:

```sh title="Terminal — request the Apple full-device path"
python3 scripts/check_database_fixture.py --program build/debug/strata-database-tests --full
```

This is a separate required-to-succeed request for that test run, not permission
to silently fall back when `F_FULLFSYNC` or directory synchronization fails. A
successful run establishes that the requested calls returned successfully for
these files and this filesystem. It is not an experiment in cutting device
power. The full-device run and native M4/macOS behavior have **not** been verified
in the Linux authoring environment.

For a manual trace, create a new temporary directory. Every command below is a
separate process:

```sh title="Terminal — inspect a small database"
database_dir="$(mktemp -d "${TMPDIR:-/tmp}/strata-chapter11.XXXXXX")"
printf 'Database directory: %s\n' "$database_dir"
build/debug/strata-database-tests create "$database_dir"
build/debug/strata-database-tests rotate "$database_dir"
build/debug/strata-database-tests append "$database_dir"
build/debug/strata-database-tests flush "$database_dir"
build/debug/strata-database-tests rotate "$database_dir"
build/debug/strata-database-tests flush "$database_dir"
build/debug/strata-database-tests inspect "$database_dir"
```

```text title="Expected — generations and current visibility"
sequence=2 tables=0 logs=1 flushed=0 epoch=1 orphans=0 event=present
sequence=2 tables=0 logs=2 flushed=0 epoch=2 orphans=0 event=present
sequence=3 tables=0 logs=2 flushed=0 epoch=2 orphans=0 event=absent
sequence=3 tables=1 logs=1 flushed=2 epoch=3 orphans=0 event=absent
sequence=3 tables=1 logs=2 flushed=2 epoch=4 orphans=1 event=absent
sequence=3 tables=2 logs=1 flushed=3 epoch=5 orphans=1 event=absent
sequence=3 tables=2 logs=1 flushed=3 epoch=5 orphans=2 event=absent
```

The final two orphans are retired WAL files, not missing data. The manifest has
two installed tables and one empty active log at base 3. You can inspect the
numbered files and preserve this directory; the example never deletes it.

Use a separate sanitizer build rather than mixing instrumented and
uninstrumented object files. The flags below cover both undefined behavior and
address errors. Platform-supported leak detection is useful, but do not force a
Linux-only sanitizer setting on macOS.

```sh title="Terminal — sanitizer build"
cmake -S . -B build/sanitized -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build/sanitized -j 2
ctest --test-dir build/sanitized --output-on-failure
```

The exact cumulative listings passed all sixteen tests in Debug, Release, and
AddressSanitizer/UndefinedBehaviorSanitizer configurations on Linux with Clang
17. Linux leak detection was enabled for the sanitizer checks. No native
M4/macOS execution, physical power-loss experiment, network-filesystem guarantee,
process RSS cap, or performance result is claimed by these checks.

## Diagnose a logical recovery defect

Temporarily remove the following guard from `WalMemTable::replay` in
`src/storage/wal.cpp` and put `(void)required_end;` in its place. The cast only
suppresses an unused-parameter warning; it does not perform a check.

```cpp title="Diagnostic — guard to remove temporarily"
    if (required_end != 0) {
        require_wal(!repaired_tail_ && sequence_ == required_end,
                    "sealed WAL does not match manifest end");
    }
```

Rebuild `strata-database-tests` in Debug, Release, and the sanitizer configuration,
then run it with no arguments. For the complete-boundary truncation, the broken
reader accepts sequence 1 even though the manifest required sequence 2. The
expected test result is exit status 1 and `database: expected format rejection`.
The test is reporting a **missing** format rejection, not confirming one.

That deliberate defect produced the same logical failure in all three tested
configurations, with no sanitizer report. The memory operations can be legal
while the recovery decision is wrong. Restore the complete guard before
continuing, rebuild all configurations, and rerun the entire CTest suite. Never
leave the diagnostic edit in a source tree used to open a persistent database.

## Troubleshooting by violated boundary

| Observation | Check first | Do not “repair” it by |
|---|---|---|
| Second process cannot acquire `LOCK` | Whether the first process still owns its descriptor | Unlinking or replacing `LOCK` |
| `authoritative MANIFEST missing` in recover mode | The exact directory and explicit create/recover intent | Creating an empty database over numbered files |
| Manifest CRC or interval validation fails | The authoritative bytes and their provenance | Selecting the largest `.tmp` filename |
| Sealed WAL end mismatch | Manifest `(base,end]`, exact complete-frame boundary, and file length | Applying active-tail truncation to the sealed log |
| Installed table is missing or damaged | The specific manifest dependency and copied fixture | Falling back to a retired WAL |
| Failure after rename poisons the old object | Reopen using the one authoritative manifest | Retrying writes through the old serving owners |
| Flush exhausts the shared budget | Old/new reader overlap, builder workspaces, and candidate image | Releasing a live recovery source before publication |
| Log or table slots are exhausted | Whether ordered flush or later compaction can free the intended slot | Overwriting a live file or inflating format counts |
| Synchronization request fails | Filesystem/device support and the exact failing operation | Downgrading the requested guarantee silently |

A database directory containing foreign files is not automatically invalid;
`notes.txt` is preserved. A file in the reserved `s-` namespace, however, must
obey the naming and regular-file rules, including the one-link requirement.
The directory cap counts foreign entries too so enumeration remains bounded.
This is a cooperating-process, trusted-local-directory model, not a hostile
multi-user sandbox.

## What is now implemented

A caller can create a database in an existing directory, commit atomic batches,
rotate an active log, flush sealed generations in order, close the process, and
reopen from a bounded checksummed manifest. All serving owners are prepared
before successful publication. Uncertain maintenance stops the old owner;
reopening decides authority. Unreferenced files are recognized without reuse or
adoption, and retained rather than reclaimed prematurely.

Current point reads resolve the latest value or tombstone across those owners.
They are not immutable multi-call snapshots, and there is no merged range cursor
or concurrent access yet. Chapter 12 will give those serving generations an
explicit snapshot lifetime and merge order. That work must preserve the sealed
end boundary, durable identity high-water, and separation between removing a
source from authority and reclaiming its file.

## References used in this chapter

The format, interval policy, and installation state machine are implemented here;
they are not claims that any one system call supplies a complete database
transaction. The following primary references supply the underlying contracts.

- [Apple `flock(2)`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/flock.2.html),
  [Apple `open(2)`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/open.2.html),
  and [Apple `rename(2)`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/rename.2.html):
  advisory locks, creation/open flags, and pathname replacement. These are archived
  manual pages; the native feature checks and actual filesystem behavior still
  matter on the target machine.
- [Apple `fsync(2)`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
  and [Apple `fcntl(2)`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fcntl.2.html):
  synchronization requests and the distinct `F_FULLFSYNC` request. The discussion
  here keeps device behavior and successful-call assumptions explicit.
- [Linux `fsync(2)`](https://man7.org/linux/man-pages/man2/fsync.2.html),
  [`open(2)` / `openat`](https://man7.org/linux/man-pages/man2/open.2.html), and
  [`opendir(3)` / `fdopendir`](https://man7.org/linux/man-pages/man3/fdopendir.3.html):
  directory synchronization, descriptor-relative opens, and directory-stream
  ownership used in the authoring-platform implementation.
- [C++20 draft N4861, shared-pointer creation](https://timsong-cpp.github.io/cppwp/n4861/util.smartptr.shared.create),
  [shared-pointer assignment](https://timsong-cpp.github.io/cppwp/n4861/util.smartptr.shared.assign),
  and [construction/destruction during exception handling](https://timsong-cpp.github.io/cppwp/n4861/except.ctor):
  allocator-aware owner construction, nonthrowing owner transfer, and cleanup
  when a database constructor does not complete.
