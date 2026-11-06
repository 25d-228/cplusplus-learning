# Chapter 22 — Recovery and resource-pressure experiments

## Part 3 — Kill the process; retain the observer

Keep [Part 1](https://25d-228.github.io/cplusplus-learning/chapter-22.html)'s
boundary/model and [Part 2](https://25d-228.github.io/cplusplus-learning/chapter-22-part-2.html)'s
fixtures and campaigns. This part adds a native executable, its independent
Python controller, the build entries, pressure checks, and a deliberately broken
barrier experiment. It completes the same 4.0-hour chapter.

## Stop at an observed boundary, not after a guessed delay

The child opens its own database, prints acknowledgements only after committed
writes, and stops at an exact named hook. `STOP` is flushed before it waits on
standard input. The parent keeps that input pipe open but sends nothing. Once
it reads the expected marker, it kills the child and waits for its termination.
No sleep is used to guess whether rotation or compaction has reached the target.

`stop` is `[[noreturn]]`: either the parent terminates the process or unexpected
input/EOF triggers `_Exit(94)`. That fallback must not be confused with the
requested signal. The Python controller checks the actual termination code.
A watchdog detects a missing rendezvous; it is not an ordering mechanism.

```cpp title="tests/recovery_native.cpp — complete file, native stop protocol and hook"
#include "recovery_fixture.hpp"
#include "strata/runtime/executor.hpp"
#include "strata/workload/driver.hpp"

#include <cerrno>
#include <chrono>
#include <cstdint>
#include <exception>
#include <cstdlib>
#include <filesystem>
#include <fstream>
#include <iostream>
#include <iterator>
#include <memory>
#include <string>
#include <string_view>
#include <system_error>
#include <vector>

namespace {
using namespace recovery_test;
using namespace strata::runtime;
namespace fs = std::filesystem;
[[noreturn]] void stop(std::string_view name) {
    std::cout << "STOP " << name << std::endl;
    char unused = 0;
    std::cin.get(unused);  // Parent keeps this pipe open, sends nothing, then kills us.
    std::_Exit(94);        // EOF or unexpected input must not look like the requested kill.
}
struct StopAt {
    std::string_view target;
    bool armed = false;
    static void hook(void* self, std::string_view stage) {
        const auto& value = *static_cast<StopAt*>(self);
        if (value.armed && value.target == stage) { stop(stage); }
    }
};
```

The fixture acknowledges sequences 1 and 2 before arming maintenance hooks.
Append has two explicit stop points: after the batch has committed but before
printing acknowledgement 3, and after acknowledgement 3. The first must allow
both histories according to the external ledger; this particular native path
will normally recover the complete third batch. The second requires it.

For compaction, flush the first and second batches separately so there are two
delta tables to replace. For collection, install the compacted base before
arming the deletion hook. Otherwise a test called “cleanup” might actually be
stopping during the setup flush.

```cpp title="tests/recovery_native.cpp — complete file, native acknowledged history and stop placement"
void writer(const fs::path& root, std::string_view action, std::string_view stage) {
    require(fs::create_directory(root), "writer needs a new directory");
    auto b = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    StopAt boundary{stage};
    Database db{b, root.string(), DatabaseOpen::create, options(), {StopAt::hook, &boundary}};
    committed(db, first_batch(), 1); std::cout << "ACK 1" << std::endl;
    if (action == "compact" || action == "collect") {
        require(db.rotate() && db.flush_oldest(), "native first delta");
    }
    committed(db, second_batch(), 2); std::cout << "ACK 2" << std::endl;
    if (action == "append") {
        std::cout << "BEGIN 3" << std::endl;
        committed(db, third_batch(), 3);
        if (stage == "before-ack") { stop(stage); }
        std::cout << "ACK 3" << std::endl;
        if (stage == "after-ack") { stop(stage); }
    } else if (action == "rotate") {
        boundary.armed = true; require(db.rotate(), "native rotation");
    } else if (action == "flush") {
        require(db.rotate(), "native sealing"); boundary.armed = true;
        require(db.flush_oldest(), "native flush");
    } else if (action == "compact" || action == "collect") {
        require(db.rotate() && db.flush_oldest(), "native second delta");
        boundary.armed = action == "compact";
        db.compact_prefix(2, compact_options());
        if (action == "collect") { boundary.armed = true; (void)db.collect_obsolete(); }
    } else { throw std::invalid_argument("native action"); }
    throw std::runtime_error("requested stop point was not reached");
}
```

Dumping opens a new `Database`, prints its sequence, checks every fixture-domain
key from 0 through 30 in both relations, and prints authoritative file references.
Python compares exact field tuples rather than trusting a C++ “passed” message.
The `L` and `T` lines locate required files for separate offline damage tests;
they are not the oracle for expected logical values.

Like the normal driver, this command performs normal recovery and may sync or
repair a permitted tail. It is **not** a forensic read-only inspection command.
The corruption campaign works on closed copies so this behavior is explicit.

```cpp title="tests/recovery_native.cpp — complete file, dump and output reader"
void dump(const fs::path& root) {
    auto b = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    {
        Database db{b, root.string(), DatabaseOpen::recover, options()};
        std::cout << "S " << db.sequence() << '\n';
        for (const auto relation : {Relation::events, Relation::accounts}) {
            for (std::uint64_t id = 0; id <= 30; ++id) {
                const auto row = db.get({relation, id}); if (!row) { continue; }
                if (relation == Relation::events) {
                    const auto e = decode_event(row->view().value);
                    std::cout << "E " << e.event_id << ' ' << e.account_id << ' ' << e.day << ' '
                              << e.amount_cents << ' ' << row->key.sequence << '\n';
                } else {
                    const auto a = decode_account(row->view().value);
                    std::cout << "A " << a.account_id << ' ' << a.region_id << ' ' << row->key.sequence << '\n';
                }
            }
        }
        const auto& manifest = db.manifest();
        for (std::size_t i = 0; i < manifest.log_count; ++i) {
            const auto l = manifest.logs[i]; std::cout << "L " << l.id << ' ' << l.base << ' ' << l.end << '\n';
        }
        for (std::size_t i = 0; i < manifest.table_count; ++i) {
            const auto t = manifest.tables[i]; std::cout << "T " << t.id << ' ' << t.base << ' ' << t.end << ' ' << t.bytes << '\n';
        }
    }
    require(b->used() == 0, "native dump retained owners");
}
std::string read(const fs::path& name) {
    std::ifstream in{name}; require(static_cast<bool>(in), "missing native output");
    return {std::istreambuf_iterator<char>{in}, {}};
}
```

## Pressure the real spill cleanup path

Reuse Chapter 19's executor and Chapter 21's tiny query preset. The input database
is frozen for these focused query-pressure tests; Chapter 20/21's separate live
scenario remains the test of concurrent ingestion. A spill hook throws `ENOSPC`
on the fourth write attempt, after earlier writes have had effects. Another
hook can refuse every unlink with `EACCES`. These are injected errors at a
specified boundary, not an experiment that fills the user's filesystem.

The cancellation callback owns a `TaskHandle` and requests stop at a spill-pass
checkpoint. Both callback state objects outlive the executor. Their fields are
only mutated by that query's worker; the controller inspects results after
completion and joined shutdown.

```cpp title="tests/recovery_native.cpp — complete file, query faults and cooperative cancellation"
struct CancelSpill {
    TaskHandle handle;
    static void hook(void* self, Boundary boundary) {
        if (boundary == Boundary::spill_pass) { static_cast<CancelSpill*>(self)->handle.cancel(); }
    }
};
struct Fault {
    bool no_space = false, deny_cleanup = false;
    std::size_t writes = 0;
    static void hook(void* self, query::SpillEvent event, const char*, std::uint64_t, std::uint64_t) {
        auto& f = *static_cast<Fault*>(self);
        if (event == query::SpillEvent::before_write && ++f.writes == 4 && f.no_space) {
            throw std::system_error(ENOSPC, std::generic_category(), "injected spill no-space");
        }
        if (event == query::SpillEvent::before_unlink && f.deny_cleanup) {
            throw std::system_error(EACCES, std::generic_category(), "injected spill deletion refusal");
        }
    }
};
```

Create 60 events with amounts 1 through 60 on day 3. Event index $i$ targets
account 20 when $i\bmod 3=0$, otherwise account 10. Account regions are 2 and 1.
The independent expected groups are therefore:

$$
\begin{aligned}
\text{region 2 count}&=20,\\
\text{region 2 cents}&=\sum_{k=0}^{19}(3k+1)
=3\frac{19\cdot20}{2}+20=590,\\
\text{region 1 count}&=60-20=40,\\
\text{region 1 cents}&=\frac{60\cdot61}{2}-590=1240.
\end{aligned}
$$

The tiny run capacity of three forces 20 initial runs in each sort for a
successful query. Each mode starts with an old completed-output marker. Disk
quota 200 permits some work but rejects subsequent admission; no-space fails
inside spill writing; cancellation stops at a pass boundary; cleanup refusal
leaves residual files reported; and the final normal query proves successful
work can proceed after the pressure cases.

```cpp title="tests/recovery_native.cpp — complete file, input construction and terminal resource checks"
void pressure(const fs::path& root) {
    require(fs::create_directory(root), "pressure needs new root");
    fs::create_directory(root / "db"); fs::create_directory(root / "spill"); fs::create_directory(root / "out");
    auto b = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    {
        Database db{b, (root / "db").string(), DatabaseOpen::create, options()};
        std::vector<Mutation> rows{Mutation::put(Account{10, 1}), Mutation::put(Account{20, 2})};
        for (std::uint64_t i = 0; i < 60; ++i) {
            rows.push_back(Mutation::put(Event{100 + i, i % 3 == 0 ? 20U : 10U, 3, i + 1}));
        }
        committed(db, rows, 1); require(db.rotate() && db.flush_oldest(), "pressure disk fixture");
    }
    require(b->used() == 0, "pressure setup leaked");
    auto inputs = std::make_shared<FrozenInputs>((root / "db").string(), options());
    constexpr auto allowance = std::size_t{2 * 1024 * 1024};
    const std::string old = "previous completed result\n";
    for (const std::string mode : {"quota", "no-space", "cancel", "cancel-denied", "complete"}) {
        const auto name = mode + ".csv"; { std::ofstream out{root / "out" / name}; out << old; }
        auto q = workload::preset("tiny", {3, 4}).query;
        if (mode == "quota") { q.event_sort.disk_bytes = 200; }
        Fault fault{mode == "no-space", mode == "cancel-denied"}; CancelSpill cancel;
        Executor executor{inputs, (root / "spill").string(), (root / "out").string(), {1, 2, allowance, 16}};
        auto request = std::make_unique<Request>(q, allowance, name); const auto handle = request->handle();
        request->query_hooks.events = {Fault::hook, &fault};
        if (mode.starts_with("cancel")) { cancel.handle = handle; request->checkpoints = {CancelSpill::hook, &cancel}; }
        require(executor.submit(request) == Submit::accepted && !request, "pressure submit");
        const auto result = handle.wait_for(std::chrono::seconds{20});
        require(result.has_value(), "pressure completion watchdog"); executor.finish();
        const auto& r = *result;
        require(r.sequence == 1 && r.final_bytes == 0 && executor.stats().used_bytes == 0 &&
                executor.stats().used_files == 0 && executor.stats().running == 0, "terminal task retained live owners");
```

A terminal result must retain no live charged query workspace, resource-admission
bytes, descriptors, or running task slot. That does **not** imply that every disk
name disappeared. In the cleanup-denied case, copied statistics retain a positive
residual-file count and its first cleanup error after live query owners are gone.
The failure path must also keep the original exception: quota exhaustion is a
`length_error`, while the injected write failure retains `ENOSPC`.

```cpp title="tests/recovery_native.cpp — complete file, outcomes and preserved completed output"
        if (mode == "complete") {
            workload::require_completed(r);
            require(r.staged_groups == 2 && r.events.initial_runs == 20 && r.groups.initial_runs == 20,
                    "post-pressure answer did not spill/complete");
        } else {
            require(read(root / "out" / name) == old, "pressure replaced completed answer");
            require(r.state == (mode.starts_with("cancel") ? TaskState::canceled : TaskState::failed), "pressure outcome");
            require(r.events.initial_runs != 0 || (mode == "no-space" && r.events.write_bytes != 0), "failure missed spill work");
            if (mode == "quota") {
                bool right = false; try { if (r.error) { std::rethrow_exception(r.error); } }
                catch (const std::length_error&) { right = true; }
                require(right, "disk admission lost its pressure error");
            }
            if (mode == "no-space") {
                bool right = false; try { if (r.error) { std::rethrow_exception(r.error); } }
                catch (const std::system_error& e) { right = e.code().value() == ENOSPC; }
                require(right, "spill ENOSPC lost original cause");
            }
        }
```

Only the test controller removes intentionally stranded spill files, and only
after `finish` has joined every worker. This is confined to the new fixture's
spill directory, never its database directory. The result is checked **before**
that cleanup; otherwise the test could erase evidence and falsely attribute the
work to the engine. Then drop the frozen-input owner, reopen the database, and
verify all 60 acknowledged events survived every query failure.

```cpp title="tests/recovery_native.cpp — complete file, observable residue, reopen, and command dispatch"
        if (mode == "cancel-denied") {
            require(r.events.live_files != 0 && r.events.cleanup_errors != 0 &&
                    r.events.first_cleanup_error == EACCES && !fs::is_empty(root / "spill"), "cleanup refusal was hidden");
            // Test controller cleanup only, after every task owner has died. No live engine files.
            for (const auto& entry : fs::directory_iterator(root / "spill")) { fs::remove_all(entry.path()); }
        } else {
            require(r.events.live_files == 0 && r.groups.live_files == 0 &&
                    !r.events.directory_live && !r.groups.directory_live && fs::is_empty(root / "spill"), "spill residue");
        }
        std::cout << "pressure " << mode << " state=" << static_cast<int>(r.state)
                  << " event_runs=" << r.events.initial_runs << " final_charge=" << r.final_bytes
                  << " residual_files=" << r.events.live_files << '\n';
    }
    inputs.reset();
    {
        Database db{b, (root / "db").string(), DatabaseOpen::recover, options()};
        require(db.sequence() == 1, "query pressure lost acknowledged database");
        for (std::uint64_t i = 0; i < 60; ++i) {
            event(db, 100 + i, Event{100 + i, i % 3 == 0 ? 20U : 10U, 3, i + 1}, 1);
        }
    }
    require(b->used() == 0, "pressure reopening leaked");
    std::cout << "pressure reopened_sequence=1 all_60_events_verified\n";
}
}  // namespace
int main(int argc, char** argv) {
    try {
        if (argc == 5 && std::string_view{argv[1]} == "stop") { writer(argv[2], argv[3], argv[4]); }
        else if (argc == 3 && std::string_view{argv[1]} == "dump") { dump(argv[2]); }
        else if (argc == 3 && std::string_view{argv[1]} == "pressure") { pressure(argv[2]); }
        else { throw std::invalid_argument("usage: stop NEW_DIR ACTION STAGE | dump DIR | pressure NEW_DIR"); }
    } catch (const std::exception& error) {
        std::cerr << "recovery native: " << error.what() << '\n'; return 1;
    }
}
```

## Give the parent an independent ledger

The Python controller uses only the standard library. Commands are argument
lists, not shell strings. A nonzero exit from a damaged-database open must be the
native program's caught-error path, not any arbitrary crash or missing executable.
Run the script without Python's `-O` option: its `assert` checks must remain active.

The [Python subprocess reference](https://docs.python.org/3/library/subprocess.html#popen-objects)
documents POSIX `kill`, negative signal return codes, pipe handling, and reaping.
The controller below drains standard output while waiting and sends standard
error to a temporary file so an unread error pipe cannot block the child.

```python title="scripts/check_recovery.py — complete file, imports and ordinary subprocess runs"
#!/usr/bin/env python3
"""Independent parent-side acknowledgement and native-file recovery checks."""
import argparse
import csv
import os
from pathlib import Path
import selectors
import shutil
import signal
import subprocess
import tempfile
import time


def run(program: Path, *args: object, ok: bool = True) -> str:
    result = subprocess.run([str(program), *map(str, args)], text=True,
                            capture_output=True, timeout=30)
    if result.returncode != (0 if ok else 1):
        raise AssertionError(f"unexpected status {result.returncode}: {args}\n"
                             f"{result.stdout}\n{result.stderr}")
    if not ok:
        assert result.stderr.startswith("recovery native:"), result.stderr
    return result.stdout if ok else result.stderr
```

`DefaultSelector` waits for pipe readiness; `os.read` then supplies arbitrary byte
chunks. A chunk may contain half a line or several lines, so `pending` retains the
unconsumed suffix. The parser advances `acknowledged` only for sequential `ACK`
lines and accepts `BEGIN 3` only after observing acknowledgement 2. The
[selector interface](https://docs.python.org/3/library/selectors.html#selectors.BaseSelector.select)
supports this readiness-plus-timeout protocol on Unix pipes.

Cleanup covers success, malformed output, premature exit, and watchdog failure.
An exception does not leave the child running. `communicate` finishes pipe
handling and reaps it after the signal; the parent verifies `-SIGKILL`, rather
than accepting `_Exit(94)` or normal return as equivalent.

```python title="scripts/check_recovery.py — complete file, stop-point rendezvous and mandatory reaping"
def killed_at(program: Path, root: Path, action: str, stage: str) -> tuple[int, bool]:
    acknowledged, attempted = 0, False
    # stderr is a file, not an undrained pipe that could block the child.
    with tempfile.TemporaryFile() as errors:
        child = subprocess.Popen([str(program), "stop", str(root), action, stage],
                                 stdin=subprocess.PIPE, stdout=subprocess.PIPE,
                                 stderr=errors)
        assert child.stdout is not None and child.stdin is not None
        pending = b""
        stopped = False
        deadline = time.monotonic() + 30
        try:
            with selectors.DefaultSelector() as ready:
                ready.register(child.stdout, selectors.EVENT_READ)
                while not stopped:
                    remaining = deadline - time.monotonic()
                    if remaining <= 0 or not ready.select(remaining):
                        raise AssertionError("child stop-point watchdog expired")
                    chunk = os.read(child.stdout.fileno(), 4096)
                    if not chunk:
                        raise AssertionError("child exited before requested stop")
                    pending += chunk
                    while b"\n" in pending:
                        line, pending = pending.split(b"\n", 1)
                        fields = line.decode("ascii").split()
                        if fields == ["BEGIN", "3"]:
                            assert acknowledged == 2 and not attempted
                            attempted = True
                        elif len(fields) == 2 and fields[0] == "ACK":
                            sequence = int(fields[1])
                            assert sequence == acknowledged + 1
                            assert sequence <= 2 or attempted
                            acknowledged = sequence
                        elif fields == ["STOP", stage]:
                            stopped = True
                            break
                        else:
                            raise AssertionError(f"unexpected child line: {fields}")
            child.kill()  # SIGKILL on the native POSIX targets; not a graceful exit.
            child.communicate(timeout=10)
            assert child.returncode == -signal.SIGKILL
            assert acknowledged in (2, 3)
            return acknowledged, attempted
        except BaseException:
            if child.poll() is None:
                child.kill()
            child.communicate(timeout=10)
            errors.seek(0)
            detail = errors.read().decode(errors="replace")
            if detail:
                print(detail)
            raise
        finally:
            child.stdout.close()
            child.stdin.close()
```

The expected rows are repeated here intentionally, as a small independently
written oracle. This code neither imports the C++ fixture nor decodes its WAL to
learn what the answer ought to be. The sequence selects one of two explicit
states, and the external acknowledgement/attempt pair limits which selection is
legal. Table and log descriptions are allowed extra diagnostic lines, not rows.

```python title="scripts/check_recovery.py — complete file, exact independent answers"
def expected(sequence: int) -> list[tuple[str, ...]]:
    # This oracle is deliberately independent of the C++ fixture/decoder.
    if sequence == 2:
        rows = ["E 1 10 3 11 2", "E 3 20 4 5 2", "A 10 4 2", "A 20 2 1"]
    elif sequence == 3:
        rows = ["E 1 10 3 13 3", "E 4 30 4 17 3", "A 10 4 2", "A 30 6 3"]
    else:
        raise AssertionError(f"unexpected recovered sequence {sequence}")
    return [tuple(row.split()) for row in rows]


def checked_dump(program: Path, root: Path, acknowledged: int, attempted: bool) -> list[list[str]]:
    fields = [line.split() for line in run(program, "dump", root).splitlines()]
    assert fields and fields[0][0] == "S"
    sequence = int(fields[0][1])
    allowed = {acknowledged}
    if acknowledged == 2 and attempted:
        allowed.add(3)
    assert sequence in allowed, (acknowledged, attempted, sequence)
    assert [tuple(row) for row in fields if row[0] in ("E", "A")] == expected(sequence)
    assert all(row[0] in ("S", "E", "A", "L", "T") for row in fields)
    return fields
```

## Run separate native crash and damage campaigns

The native campaign has 26 stop points: two acknowledgement boundaries, seven
rotation boundaries, seven flush boundaries, eight compaction boundaries, and
two collection boundaries. Each has a fresh child-created directory. Both
reopens must produce identical logical and manifest descriptions. A missing
requested hook causes a failure, not a skipped test.

```python title="scripts/check_recovery.py — complete file, native stop-point enumeration"
def native(program: Path, root: Path) -> None:
    common = ["dependencies-synced", "manifest-written", "manifest-synced",
              "manifest-renamed", "directory-synced", "manifest-stabilized"]
    stages = {
        "append": ["before-ack", "after-ack"],
        "rotate": ["log-ready", *common],
        "flush": ["table-ready", *common],
        "compact": ["compaction-file-ready", "compaction-prepared", *common],
        "collect": ["before-unlink", "before-cleanup-sync"],
    }
    saved: dict[str, Path] = {}
    count = 0
    for action, points in stages.items():
        for stage in points:
            path = root / f"{action}-{stage}"
            ack, attempted = killed_at(program, path, action, stage)
            first = checked_dump(program, path, ack, attempted)
            assert checked_dump(program, path, ack, attempted) == first
            saved[action] = path
            count += 1
    print(f"native: {count} controlled SIGKILL cases; external acknowledgements; two reopens each")
```

Copy only closed, already-recovered fixtures for corruption checks. Remove a
referenced table, flip a table byte, damage the manifest checksum, or remove the
entire second batch from a sealed WAL. The sealed WAL originally has 362 bytes;
truncating to $32+172=204$ leaves a valid first batch but violates its required
sealed end. The test checks that the failed open did not silently repair this
file or rewrite the deliberately damaged table/manifest.

An invalid temporary file 900 is different: it is unreferenced, so reopening
must keep the logical answer and leave the file alone. Collection and its
identity fence were tested in Part 2. Normal recovery must not adopt the orphan
merely because its identifier is high.

```python title="scripts/check_recovery.py — complete file, offline required-file damage and orphan check"
    def copy(source: str, name: str) -> Path:
        return Path(shutil.copytree(saved[source], root / name))

    # All damage is to closed fixture copies, never a live database.
    table_root = copy("flush", "missing-table")
    description = checked_dump(program, table_root, 2, False)
    table_id = next(int(row[1]) for row in description if row[0] == "T")
    table_name = f"s-{table_id:020d}.sst"
    (table_root / table_name).unlink()
    run(program, "dump", table_root, ok=False)
    corrupt = copy("flush", "corrupt-table")
    table = corrupt / table_name
    contents = bytearray(table.read_bytes())
    contents[50] ^= 1
    table.write_bytes(contents)
    run(program, "dump", corrupt, ok=False)
    assert table.read_bytes() == contents
    manifest_root = copy("flush", "corrupt-manifest")
    manifest = manifest_root / "MANIFEST"
    contents = bytearray(manifest.read_bytes()); contents[-1] ^= 1
    manifest.write_bytes(contents)
    run(program, "dump", manifest_root, ok=False)
    assert manifest.read_bytes() == contents
    sealed_root = copy("rotate", "short-sealed")
    description = checked_dump(program, sealed_root, 2, False)
    sealed_id = next(int(row[1]) for row in description if row[0] == "L" and int(row[3]) != 0)
    sealed = sealed_root / f"s-{sealed_id:020d}.wal"
    assert sealed.stat().st_size == 362
    with sealed.open("r+b") as out:
        out.truncate(204)  # Whole batch 2 removed, not merely a short final frame.
    run(program, "dump", sealed_root, ok=False)
    assert sealed.stat().st_size == 204
    orphan_root = copy("flush", "unreferenced-junk")
    orphan = orphan_root / "s-00000000000000000900.tmp"
    orphan.write_bytes(b"not a recoverable generation\n")
    checked_dump(program, orphan_root, 2, False)
    assert orphan.read_bytes() == b"not a recoverable generation\n"
    print("native: required-file loss, corrupt table/manifest, and short sealed log fail closed; orphan ignored")
```

## Verify the completed answer after pressure

Python checks the successful CSV independently against the arithmetic above,
checks all four old-result markers, and inspects final directory contents.
There must be no hidden partial-result files. The spill directory is empty only
after the controller's explicitly documented cleanup of the refusal case.
This final check does not erase or reinterpret the earlier residual-file report.

```python title="scripts/check_recovery.py — complete file, pressure oracle and script entry point"
def pressure(program: Path, root: Path) -> None:
    print(run(program, "pressure", root), end="")
    with (root / "out" / "complete.csv").open(newline="") as source:
        rows = list(csv.reader(source))
    assert rows == [["region_id", "day", "event_count", "total_cents"],
                    ["1", "3", "40", "1240"], ["2", "3", "20", "590"]], rows
    for name in ("quota", "no-space", "cancel", "cancel-denied"):
        assert (root / "out" / f"{name}.csv").read_text() == "previous completed result\n"
    assert not any((root / "spill").iterdir())
    assert sorted(p.name for p in (root / "out").iterdir()) == sorted(
        f"{name}.csv" for name in ("quota", "no-space", "cancel", "cancel-denied", "complete"))
    print("pressure: independent grouped answer, old outputs, and final directory contents verified")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("program", type=Path)
    parser.add_argument("mode", choices=("native", "pressure"))
    args = parser.parse_args()
    program = args.program.resolve(strict=True)
    with tempfile.TemporaryDirectory(prefix="strata-recovery-") as scratch:
        root = Path(scratch)
        if args.mode == "native":
            native(program, root)
        else:
            pressure(program, root / "pressure")


if __name__ == "__main__":
    main()
```

## Build and run the complete experiments

Append this block to the existing `CMakeLists.txt`. It reuses the already-found
Python interpreter, `strata-core`, `strata-workload`, and their thread linkage.
It adds no new runtime dependency. Keep the Chapter 21 tests and earlier entries.

```cmake title="CMakeLists.txt — append Chapter 22 tests"

# Chapter 22: modeled persistence is separate from native process termination.
add_executable(strata-recovery-tests tests/persistence_model.cpp tests/recovery.cpp)
target_link_libraries(strata-recovery-tests PRIVATE strata-core)
target_compile_options(strata-recovery-tests PRIVATE -Wall -Wextra -Wpedantic -Wconversion -Wshadow)
add_test(NAME recovery-model COMMAND strata-recovery-tests)
set_tests_properties(recovery-model PROPERTIES TIMEOUT 90)
add_executable(strata-recovery-native tests/recovery_native.cpp)
target_link_libraries(strata-recovery-native PRIVATE strata-workload)
target_compile_options(strata-recovery-native PRIVATE -Wall -Wextra -Wpedantic -Wconversion -Wshadow)
add_test(NAME recovery-native COMMAND ${Python3_EXECUTABLE}
  ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_recovery.py $<TARGET_FILE:strata-recovery-native> native)
add_test(NAME recovery-pressure COMMAND ${Python3_EXECUTABLE}
  ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_recovery.py $<TARGET_FILE:strata-recovery-native> pressure)
set_tests_properties(recovery-native recovery-pressure PROPERTIES TIMEOUT 120)
```

Use the same compatible C++20 compiler, CMake, and Python setup as the earlier
chapters. On the M4 Mac mini, build natively with your installed compiler; do not
assume a particular macOS release or require a branded compiler version. The
native controller requires POSIX process signals and Unix pipe selection.

```sh
cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build-debug --parallel 4
ctest --test-dir build-debug --output-on-failure
ctest --test-dir build-debug -R '^recovery-' --output-on-failure -V

cmake -S . -B build-release -DCMAKE_BUILD_TYPE=Release
cmake --build build-release --parallel 4
ctest --test-dir build-release --output-on-failure
```

For direct diagnosis, the commands behind the three new tests are:

```sh
./build-debug/strata-recovery-tests
python3 scripts/check_recovery.py ./build-debug/strata-recovery-native native
python3 scripts/check_recovery.py ./build-debug/strata-recovery-native pressure
```

The model should report 2,754 selected crash images, 173 append prefixes, and
zero surviving tracked owners at each checked boundary. Native output should
report 26 controlled SIGKILL cases and the required-file checks. The pressure
fixture reports failed quota/no-space tasks, canceled cancellation tasks, and
one completed task. Numeric task-state values are diagnostic enum values;
assertions compare the named states. The refusal case reports 20 residual event
runs, while the ordinary canceled case reports none. Both preserve the previous
completed output.

Keep memory and race sanitizers in separate build directories. A normal sanitizer
run must use the correct implementation, not the deliberate defect below.

```sh
cmake -S . -B build-asan -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS='-fsanitize=address,undefined -fno-omit-frame-pointer' \
  -DCMAKE_EXE_LINKER_FLAGS='-fsanitize=address,undefined'
cmake --build build-asan --parallel 4
ASAN_OPTIONS=detect_leaks=1 UBSAN_OPTIONS=halt_on_error=1 \
  ctest --test-dir build-asan --output-on-failure

cmake -S . -B build-tsan -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS='-fsanitize=thread -fno-omit-frame-pointer' \
  -DCMAKE_EXE_LINKER_FLAGS='-fsanitize=thread'
cmake --build build-tsan --target strata-recovery-native --parallel 4
TSAN_OPTIONS=halt_on_error=1 \
  ctest --test-dir build-tsan -R '^recovery-pressure$' --output-on-failure
```

ThreadSanitizer support depends on the compiler/runtime/host combination. Diagnose
a startup or mapping failure as a tool-environment failure, not a race report or
a passing test. Chapter 18 explains that distinction. The synchronous model is
not a thread-interleaving test. The focused pressure test exercises worker and
controller ownership; it does not enumerate every concurrent schedule.

## Remove one barrier and predict the first failure

Make one temporary diagnostic edit inside `Database::install` in
`src/storage/database.cpp`: remove only the first
`context_->directory.sync()` after `encode_manifest`. Leave the later directory
sync after rename in place. The deliberately incorrect fragment is:

```cpp title="Diagnostic only — missing dependency-name barrier"
auto image = encode_manifest(budget_, candidate);
// Deliberate defect: dependency names have not been synchronized here.
hook_("dependencies-synced");
auto opened = context_->directory.open(temporary.data(), io::OpenMode::create_new);
```

The hook's label is not a persistence guarantee. Rebuild and run `recovery-model`.
The selected failing trace is rotation, cut 28, image 7, effect 4 (`rename`): the
new manifest's synchronized bytes and renamed binding survive, but the new
active log's binding does not. Opening that required log raises `ENOENT`.
The database must not invent an empty replacement or choose a different manifest
merely to make the test pass.

In the verified Linux run, this defect failed the model in Debug, Release, and
AddressSanitizer/UndefinedBehaviorSanitizer builds without a sanitizer memory
error. The same defective installer passed all 26 native SIGKILL cases. That
contrast is the point: killing a user process does not ask the live kernel to
roll back arbitrary unsynchronized names. Passing a native crash campaign alone
is not evidence that the missing persistence barrier is harmless.

Restore the first directory sync, rebuild, and rerun the complete suite. Do not
retain the diagnostic fragment as an optimization. The correct implementation
used for the final checks contains both directory barriers.

## Interpret failures before retrying

| Observation | Required interpretation and next check |
|---|---|
| Recovered sequence is below the external acknowledgement | Lost acknowledged history; inspect the selected image and missing dependency, not just frame validity. |
| Correct sequence, wrong values or tombstone visibility | Atomicity or version-resolution failure; compare the exact expected state. |
| Valid incomplete active tail is repaired | Permitted replay behavior; verify synchronized repair and the second reopen. |
| Complete corrupt frame or shortened sealed WAL rejects | Fail-closed behavior, not permission to discard acknowledged data. |
| Budget admission rejects before append | Definite rejection; resources release and the old database remains usable. |
| Append/sync returns an I/O error | Uncertain outcome; stop using that writer and reopen before deciding how to reconcile the attempted batch. |
| Canceled query reports residual files and a cleanup error | Cancellation completed, cleanup did not; retain that evidence and remove only verified temporary fixture data after owners die. |
| Watchdog expires before `STOP` | The rendezvous was not established; inspect child errors and hook placement rather than increasing a sleep. |

Do not delete a WAL, table, `MANIFEST`, or `LOCK` merely because it appears old.
This chapter's controlled damage helpers operate only on disposable closed
fixtures. SQLite's [corruption guide, Sections 1.2–1.4 and 3](https://www.sqlite.org/howtocorrupt.html)
illustrates why recovery dependencies and synchronization assumptions need to be
considered together. Strata's own manifest and retention graph remain the
specific authority for its files.

## What these checks establish—and what they do not

The verified authoring runs used Clang 17 on Linux: all 37 cumulative CTest entries
passed Debug, Release, and AddressSanitizer/UndefinedBehaviorSanitizer builds.
A separate GCC 14.2 ThreadSanitizer build passed the focused `recovery-pressure`
test. The independent parent compared native recovered states and completed CSV
answers. A process killed with SIGKILL cannot run a sanitizer's normal exit-time
cleanup report; host-side cleanup is checked separately by the model's owner
assertions and ordinary pressure completions.

These are actual Linux checks, **not native M4/macOS results**. The native tests
use the existing default file-sync mode. Apple's archived
[`fsync(2)` manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
distinguishes host flushing from the stronger `F_FULLFSYNC` request. Consult the
installed system's manuals and the Chapter 6 implementation on the target Mac;
this model treats both modes as ideal file barriers and cannot validate their
hardware behavior. No physical power-cut experiment, device-cache certification,
or universally safe filesystem assumption follows from these tests.

The experiments also do not add an automatic corruption-repair utility, a
production exactly-once client protocol, or unlimited resource capacity. They
establish reproducible allowed-history, fail-closed, and cleanup behavior within
the stated model and fixtures. The next chapter measures the existing mechanisms,
changes one demonstrated performance decision, and assembles the final
correctness-and-resource demonstration.

