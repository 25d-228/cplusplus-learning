# Chapter 21 — Reproducible workloads and independent answers

## Part 1 — A workload is a sequence of committed batches

A query result is not correct merely because the engine can read the file it
just wrote. The writer, reader, join, and expected-result code might all make
the same mistake. An update can also change an answer without changing the
number of input rows. We need a workload whose mutation history is explicit,
a small reference implementation that does not call the engine, and a larger
answer that does not require a Python dictionary containing the entire database.

This chapter supplies those pieces and a thin command-line program. The storage
and query mechanisms remain the Chapter 20 implementations. The driver will
import complete batches, run the fixed query, compact, inspect, and reopen.
An in-process scenario will hold an old query while the writer applies a delta,
reject another snapshot at the retention limit, cancel a spilling query, and
verify both the old and new answers independently.

> **Learning objectives**
> Specify reproducible input rather than an incidental random stream; compare
> committed snapshots rather than input prefixes; distinguish logical record
> bytes from text and physical I/O; parse bounded numeric input without losing
> unsigned precision; derive an analytical answer; and make a concurrent
> demonstration establish its ordering with predicates rather than sleeps.

**Prerequisites:** the complete engine through
[Chapter 20](https://25d-228.github.io/cplusplus-learning/chapter-20.html).
Keep its `Engine`, `Executor`, `Request`, private query readers, shared budget
ledger, and immutable capture interfaces. Chapter 17 supplies the fixed query
and completion-aware CSV sink; Chapter 19 supplies task outcomes and cooperative
cancellation. No prerequisite source file is replaced in this chapter.

**Implementation: 4.5 hours across all three parts.** Allocate 1.5 hours to the
generator/reference, 1.25 to the driver/scenario, 1.0 to controlled interactions,
and 0.75 to fixture and diagnosis checks. Reading is separate. Record actual
time; a longer explanation is not a reason to spend longer typing the adapter.

Create these files, concatenating blocks with the same filename in reading
order, then append the CMake block in Part 3:

```text
scripts/generate_data.py
scripts/reference_query.py
scripts/check_workload.py
include/strata/workload/input.hpp
include/strata/workload/driver.hpp
src/workload/input.cpp
src/workload/driver.cpp
apps/strata.cpp
apps/scenario.cpp
```

[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-21-part-2.html)
implements the C++ adapter. [Part 3](https://25d-228.github.io/cplusplus-learning/chapter-21-part-3.html)
implements the overlap scenario, checks, and complete run commands.

## Put the commit boundary in the input

The input is a deliberately narrow ASCII format, **not general CSV**. Each line
has one space between fields and ends with LF. Integers are unsigned decimal
without a sign or leading zero, except the single digit `0`. Content is at most
255 bytes per line. An event ID and account ID fit in 64 bits; day and region fit
in 32 bits; cents fit in 64 bits. Amounts are integers, not binary floating-point
currency.

```text
STRATA1 9 0
B 1 4
A 10 1
A 20 2
E 1 10 3 7
E 2 20 3 9
C 1
END 1 4 88
```

The header contains the format version, seed, and required **base sequence**.
`B 1 4` begins sequence 1 with four operations. `A` is an account put:
`account_id region_id`. `E` is an event put:
`event_id account_id day cents`. `X A id` and `X E id` are tombstones.
`C 1` terminates that batch. `END` records the last sequence, operation count,
and logical byte total for the file.

A batch contains 1 through 256 distinct relation/key pairs. Account 10 and event
10 are different keys. Two event-10 operations in the same batch are rejected;
we do not silently decide which one wins. A file can contain many batches, but
only one batch is buffered at a time. Sequences are contiguous and absolute,
not renumbered by each importing process.

The parser returns a batch **only after** consuming its matching `C` line.
Then `Engine::write` supplies the actual WAL-backed commit. The text `C` is
neither a durable record nor proof that the database has accepted anything.
If the second batch is malformed, the first acknowledged batch remains
committed. If `END` is missing after otherwise valid batches, import fails but
those batches also remain. This is batch atomicity, not whole-file atomicity.
The program reports the last acknowledged sequence and refuses automatic retry.
A sequence guard also does not prove database identity: two unrelated databases
can have the same sequence. These fixtures use new directories and a known
history, not a production deduplication protocol.

For the sample, a query over days $[3,4)$ joins both events and returns
`(region, day, count, cents)` values `(1,3,1,7)` and `(2,3,1,9)`.
A second batch can move account 10 to region 4, change event 1 to 11 cents, and
delete event 2. Sequence 2 then returns only `(4,3,1,11)`. A sequence-1 query must
still return the earlier two rows. Counting input lines cannot establish this.

## Count bytes with an explicit denominator

Let $E$ be the number of event puts, $A$ the number of account puts, and $X$ the
number of tombstones **submitted in this file**, including updates. Define the
workload's logical encoded-byte measure as

$$
L = 30E + 14A + 9X.
$$

The put widths are Chapter 6's encodings. The 9-byte tombstone measure is one
relation byte plus an eight-byte primary key. It is a workload accounting
convention, **not the complete WAL tombstone representation**. Sequence numbers,
frame checksums, manifests, block directories, repeated versions, and physical
writes are excluded. Therefore $L$ is neither live-data size nor disk usage.
The generator separately reports the actual text-file size, including headers
and decimal digits. Later measurements can divide physical I/O by $L$, but must
name that denominator.

The sample has two accounts and two events, so $L=2(14)+2(30)=88$.
With the default 17 accounts and 120 events, the base has 137 operations and
$L=17(14)+120(30)=3838$. At 32 operations per batch, its last sequence is
$\lceil137/32\rceil=5$. The default delta has 30 operations and 621 logical bytes,
so it commits as sequence 6. Changing the batch size changes these sequence
numbers even when the final logical values are unchanged.

## Generate bounded batches, not a giant list

Start the generator. `pending` owns at most `batch` small tuples. The operation
iterator yields one tuple at a time; no list of all events is built. The nested
`flush` function uses `nonlocal` to update counters belonging to `write_phase`,
not new local variables. File mode `x` rejects an existing name. The output
context closes the file on every exit, but a failed generation may leave an
incomplete file; it is not a transactional file-creation facility.

```python title="scripts/generate_data.py"
#!/usr/bin/env python3
"""Emit a versioned, bounded-batch workload. No engine or oracle imports."""
import argparse
import json
from pathlib import Path

U64 = (1 << 64) - 1
PERIOD = 1000


def write_phase(path, seed, base, operations, batch):
    sequence, count, logical = base, 0, 0
    pending = []
    with path.open("x", encoding="ascii", newline="\n") as out:
        out.write(f"STRATA1 {seed} {base}\n")

        def flush():
            nonlocal sequence, count, logical
            if not pending:
                return
            if sequence == U64:
                raise ValueError("sequence overflow")
            sequence += 1
            out.write(f"B {sequence} {len(pending)}\n")
            for fields in pending:
                out.write(" ".join(map(str, fields)) + "\n")
                logical += {"E": 30, "A": 14, "X": 9}[fields[0]]
            out.write(f"C {sequence}\n")
            count += len(pending)
            pending.clear()

        for operation in operations:
            pending.append(operation)
            if len(pending) == batch:
                flush()
        flush()
        out.write(f"END {sequence} {count} {logical}\n")
    return dict(base_sequence=base, last_sequence=sequence,
                operations=count, logical_bytes=logical,
                text_bytes=path.stat().st_size)
```

`logical` increases once per serialized operation; it does not count `B`, `C`,
or `END` text. `pending.clear()` occurs only after the matching terminator has
been written. The generator is not issuing database writes, so this ordering
specifies a complete input file rather than claiming disk durability.

## Use a documented pattern for skew and updates

A seed alone does not specify every high-level random algorithm forever.
Python's [reproducibility notes](https://docs.python.org/3/library/random.html#notes-on-reproducibility)
distinguish the stable `random()` sequence guarantee from algorithms that may
change. Here we avoid that dependency: the seed is an integer parameter in an
explicit recipe, not a claim that the data is random.

Let $N$ be event count, $a$ account count, $d$ the number of days, $g$ the number
of regions, $h$ the seed, and $s$ the forced-hot percentage. Event identity $i$
runs from 1 through $N$. Define its **lane**

$$
r=(i-1)\bmod1000.
$$

The account is 1 when $r\bmod100<s$; otherwise it is
$1+(17r+h)\bmod a$. Day is $(r+h)\bmod d$, and cents are
$1+(37r+h)\bmod1000$. Account $j$ initially belongs to region
$1+(j+h)\bmod g$. A fallback account can also be 1, so the true hot-account
fraction can exceed the forced fraction. A short final period need not have
the same distribution as a complete period.

The delta deletes every configured deletion lane first. A surviving update
lane moves to the next account, wrapping at $a$, and adds 1000 cents. Deletion
wins when both rules select a lane. Separately, account IDs divisible by 11 are
deleted; remaining IDs divisible by 5 move to the next region. Zero disables
an event update/deletion interval. These choices create dimension changes,
missing join partners, hot keys, and repeated versions with calculable answers.

```python title="scripts/generate_data.py"
def event(p, identity, changed=False):
    lane = (identity - 1) % PERIOD
    account = (1 if lane % 100 < p["hot_percent"] else
               1 + (17 * lane + p["seed"]) % p["accounts"])
    cents = 1 + (37 * lane + p["seed"]) % 1000
    if changed:
        account = 1 + account % p["accounts"]
        cents += 1000
    return ("E", identity, account, (lane + p["seed"]) % p["days"], cents)


def base_rows(p):
    for account in range(1, p["accounts"] + 1):
        yield ("A", account, 1 + (account + p["seed"]) % p["regions"])
    for identity in range(1, p["events"] + 1):
        yield event(p, identity)


def delta_rows(p):
    for account in range(1, p["accounts"] + 1):
        if account % 11 == 0:
            yield ("X", "A", account)
        elif account % 5 == 0:
            region = 1 + (account + p["seed"]) % p["regions"]
            yield ("A", account, 1 + region % p["regions"])
    for identity in range(1, p["events"] + 1):
        lane = (identity - 1) % PERIOD
        if p["delete_every"] and lane % p["delete_every"] == 0:
            yield ("X", "E", identity)
        elif p["update_every"] and lane % p["update_every"] == 0:
            yield event(p, identity, changed=True)
```

For the defaults, event 22 has lane 21: account 1, day
$(21+7)\bmod8=4$, and $1+(37\cdot21+7)\bmod1000=785$ cents.
Lane 21 is divisible by 7 but not 13, so the delta changes it to account 2 and
1785 cents. Account 1 maps to region 1; account 2 maps to region 2. Its
contribution moves between groups. Event 27 has lane 26 and is deleted instead.

Recipe arithmetic uses mathematical integers before taking a remainder; it does
not add an implicit 64-bit wraparound. The pattern repeats every 1000 identities.
It is a controlled stress fixture,
not a model of real customer traffic or a statistical performance benchmark.
In particular, update spacing restarts at each period. Preserve the recipe
name when reproducing an answer; changing this rule requires a new recipe.

The complete entry point validates bounds, writes the two phases, and writes
metadata last. A directory with partial data but no completed `meta.json`
should not be treated as a completed generation. The script does not remove
such a directory automatically.

```python title="scripts/generate_data.py"
def generate(directory, p):
    if not (0 <= p["seed"] <= U64 and 0 <= p["events"] <= 10**12 and
            1 <= p["accounts"] <= 4096 and 1 <= p["days"] <= 64 and
            1 <= p["regions"] <= 32 and 0 <= p["hot_percent"] <= 100 and
            1 <= p["batch"] <= 256 and 0 <= p["update_every"] <= PERIOD and
            0 <= p["delete_every"] <= PERIOD):
        raise ValueError("parameters outside workload-v1 limits")
    directory.mkdir(parents=True, exist_ok=False)
    base = write_phase(directory / "base.ops", p["seed"], 0, base_rows(p), p["batch"])
    delta = write_phase(directory / "delta.ops", p["seed"], base["last_sequence"],
                        delta_rows(p), p["batch"])
    meta = dict(recipe="strata-periodic-v1", parameters=p, base=base, delta=delta)
    # Written last: a partial generation has no completed metadata record.
    with (directory / "meta.json").open("x", encoding="utf-8") as out:
        json.dump(meta, out, indent=2, sort_keys=True)
        out.write("\n")
    return meta


def main():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("directory", type=Path)
    for name, default in (("seed", 7), ("events", 120), ("accounts", 17),
                          ("days", 8), ("regions", 4), ("hot-percent", 70),
                          ("batch", 32), ("update-every", 7), ("delete-every", 13)):
        parser.add_argument("--" + name, type=int, default=default)
    args = vars(parser.parse_args())
    directory = args.pop("directory")
    try:
        print(json.dumps(generate(directory, args), sort_keys=True))
    except (OSError, ValueError) as error:
        parser.exit(1, f"generate seed={args['seed']} directory={directory}: {error}\n")


if __name__ == "__main__":
    main()
```

The generator's $10^{12}$ event limit is an arithmetic/input-generation bound,
not a promise that this engine can import or query that many rows. The engine's
run-descriptor, table, and memory limits remain in force. Part 2 derives the
much smaller limits of the supplied execution presets.

## Parse independently before evaluating a snapshot

The reference program does not import `generate_data`, C++ encodings, or engine
query results. Its small mode stores current values in Python dictionaries and
applies **whole validated batches** up to a requested sequence. The complete
file is still validated, including later batches and `END`, even when evaluating
an older snapshot. Thus malformed later text is not silently accepted as a
valid workload description.

Begin with integer parsing and the header of the batch iterator. `readline(257)`
places a bound on text admitted for one line. Accepted lines occupy at most
256 bytes including LF. ASCII decoding and the numeric grammar prevent locale
or Unicode digits from changing the format. The first yielded item describes
the file's seed and base sequence; subsequent yields are completed batches.

```python title="scripts/reference_query.py"
#!/usr/bin/env python3
"""Independent dictionary replay and constant-period analytical answers."""
import argparse
import csv
import json
from pathlib import Path
import sys

U64 = (1 << 64) - 1
HEADER = ["region_id", "day", "event_count", "total_cents"]


def unsigned(text, maximum=U64):
    if (not text or not text.isascii() or not text.isdecimal() or
            (len(text) > 1 and text[0] == "0")):
        raise ValueError(f"noncanonical unsigned integer: {text!r}")
    value = int(text)
    if value > maximum:
        raise ValueError("integer out of range")
    return value


def batches(path):
    with path.open("rb") as stream:
        line_number = 0

        def line():
            nonlocal line_number
            raw = stream.readline(257)
            line_number += 1
            if not raw.endswith(b"\n") or len(raw) > 256:
                raise ValueError(f"{path}:{line_number}: missing/long/unterminated line")
            text = raw[:-1].decode("ascii")
            tokens = text.split(" ")
            if any(not token for token in tokens):
                raise ValueError("empty token")
            return tokens

        head = line()
        if len(head) != 3 or head[0] != "STRATA1":
            raise ValueError("workload header")
        seed, sequence = map(unsigned, head[1:])
        yield ("header", seed, sequence)
```

The rest of the iterator recognizes puts and tombstones. Its duplicate-key set
is bounded by 256. A put's `value` is a tuple or region integer; `None` means a
tombstone, so region zero is a valid value and is not confused with deletion.
`logical` is parser working state. If parsing raises, discard this iterator;
its counters are not a recovery checkpoint.

```python title="scripts/reference_query.py"
        count = logical = 0
        while True:
            words = line()
            if words[0] == "END":
                if len(words) != 4 or list(map(unsigned, words[1:])) != [sequence, count, logical]:
                    raise ValueError("workload totals")
                if stream.read(1):
                    raise ValueError("bytes after END")
                return
            if len(words) != 3 or words[0] != "B" or sequence == U64:
                raise ValueError("batch header")
            next_sequence, size = map(unsigned, words[1:])
            if next_sequence != sequence + 1 or not 1 <= size <= 256:
                raise ValueError("batch sequence/count")
            changes, keys = [], set()
            for _ in range(size):
                row = line()
                if row[0] == "E" and len(row) == 5:
                    identity, account, day, cents = map(unsigned, row[1:])
                    if day >= 1 << 32:
                        raise ValueError("event day")
                    key, value, width = ("E", identity), (account, day, cents), 30
                elif row[0] == "A" and len(row) == 3:
                    identity, region = map(unsigned, row[1:])
                    if region >= 1 << 32:
                        raise ValueError("account region")
                    key, value, width = ("A", identity), region, 14
                elif row[0] == "X" and len(row) == 3 and row[1] in ("E", "A"):
                    key, value, width = (row[1], unsigned(row[2])), None, 9
                else:
                    raise ValueError("operation fields")
                if key in keys:
                    raise ValueError("duplicate key in batch")
                keys.add(key)
                changes.append((key, value))
                logical += width
            if line() != ["C", str(next_sequence)]:
                raise ValueError("batch terminator")
            sequence = next_sequence
            count += size
            if max(count, logical) > U64:
                raise ValueError("input totals overflow")
            yield (sequence, changes)
```

Notice where `yield (sequence, changes)` occurs: after the entire batch and
matching terminator have been checked. The caller cannot accidentally observe
half a batch. A Python generator resumes after its last `yield`; it does not
restart the function. The file context remains owned by that iterator until
it finishes or closes.

## Preserve unsigned and aggregate semantics

Python can represent values beyond 64 bits. The engine cannot. An independent
oracle must therefore check the engine's domain rather than quietly accepting
an answer that C++ must reject. Let $U=2^{64}-1$. For a group with current count
$c$ and total $t$, and a contribution of $\Delta c,\Delta t$, require

$$
c+\Delta c\le U,\qquad t+\Delta t\le U.
$$

Only then store both new values. All contributions are nonnegative, so a final
overflow cannot be canceled by later rows; enumeration order does not change
the pass/fail decision. There is no floating-point tolerance. Sort the final
`(region, day)` keys lexicographically and compare every integer exactly.

```python title="scripts/reference_query.py"
def add_group(groups, region, day, count, cents):
    previous_count, previous_cents = groups.get((region, day), (0, 0))
    new_count, new_cents = previous_count + count, previous_cents + cents
    if max(new_count, new_cents) > U64:
        raise OverflowError("aggregate exceeds uint64")
    groups[(region, day)] = (new_count, new_cents)


def answer(events, accounts, begin, end):
    groups = {}
    for account, day, cents in events.values():
        if begin <= day < end and account in accounts:
            add_group(groups, accounts[account], day, 1, cents)
    return [(region, day, *totals) for (region, day), totals in sorted(groups.items())]


def replay(paths, target, begin, end, max_keys=100000):
    events, accounts = {}, {}
    sequence, seed = 0, None
    for path in paths:
        records = iter(batches(path))
        _, file_seed, base = next(records)
        if base != sequence or (seed is not None and seed != file_seed):
            raise ValueError("input files do not form one ordered workload")
        seed = file_seed
        for sequence, changes in records:
            if sequence > target:
                continue
            # Each batch is fully parsed and validated before any state changes.
            for (relation, identity), value in changes:
                state = events if relation == "E" else accounts
                if value is None:
                    state.pop(identity, None)
                else:
                    if identity not in state and len(events) + len(accounts) == max_keys:
                        raise ValueError("small oracle key cap; use formula for generated large data")
                    state[identity] = value
    if sequence < target:
        raise ValueError("requested sequence is not present")
    return answer(events, accounts, begin, end)
```

`replay` checks that each file starts exactly where the preceding one ended
and that their seeds agree. It deliberately starts from sequence zero: a delta
alone is insufficient to reconstruct its base. The bounded dictionary mode
rejects insertion beyond `max_keys` rather than swapping away the machine.
The cap counts live account and event keys; temporary batch objects are
separate. A deleted account removes inner-join matches even when its events
remain in the event dictionary.

The comparison target is the requested committed sequence, not the sequence
of the last physical record read. `sequence` continues advancing during input
validation after `target`, while the dictionaries stop changing. An absent
requested sequence is an error. This separation is what permits checking a
held old view against a fully supplied new history.

## Derive the large answer without materializing events

For the periodic recipe, all payload fields and mutation decisions depend on
lane, not on the number of the period. Event IDs remain distinct, but the final
query does not group by ID. Write

$$
N=1000q+t,\qquad 0\le t<1000.
$$

Here $q$ is the number of complete periods and $t$ the remaining events. Lane
$r$ occurs

$$
m_r=q+\mathbf{1}[r<t]
$$

times; the bracket is 1 when its condition is true and 0 otherwise. For
$N=2007$, lanes 0 through 6 occur three times and all other lanes twice.
The sum is $7(3)+993(2)=2007$. This is the loop's `multiplicity`.

After applying deletion, update, day filtering, and account existence for a
lane, it contributes $m_r$ events and $m_r v_r$ cents to its region/day group,
where $v_r$ is the surviving event amount. The code enumerates exactly 1000
lanes and at most $32\cdot64=2048$ output groups. It never enumerates $N$ events
or imports the generator's event function.

```python title="scripts/reference_query.py"
def formula(metadata, phase, begin, end):
    if metadata["recipe"] != "strata-periodic-v1":
        raise ValueError("unknown analytical recipe")
    p = metadata["parameters"]
    n, a, seed = p["events"], p["accounts"], p["seed"]
    if not (0 <= n <= 10**12 and 1 <= a <= 4096 and 0 <= seed <= U64 and
            1 <= p["days"] <= 64 and 1 <= p["regions"] <= 32 and
            0 <= p["hot_percent"] <= 100 and 0 <= p["update_every"] <= 1000 and
            0 <= p["delete_every"] <= 1000):
        raise ValueError("recipe parameter range")
    groups = {}
    cycles, tail = divmod(n, 1000)
    # Enumerate lanes, not events; no event dictionary or generated input import.
    for lane in range(1000):
        multiplicity = cycles + int(lane < tail)
        if not multiplicity:
            continue
        if phase == "delta" and p["delete_every"] and lane % p["delete_every"] == 0:
            continue
        identity = 1 if lane % 100 < p["hot_percent"] else (17 * lane + seed) % a + 1
        amount = (37 * lane + seed) % 1000 + 1
        if phase == "delta" and p["update_every"] and lane % p["update_every"] == 0:
            identity = identity % a + 1
            amount += 1000
        day = (lane + seed) % p["days"]
        if not begin <= day < end or (phase == "delta" and identity % 11 == 0):
            continue
        region = (identity + seed) % p["regions"] + 1
        if phase == "delta" and identity % 5 == 0:
            region = region % p["regions"] + 1
        add_group(groups, region, day, multiplicity, multiplicity * amount)
    return [(region, day, *totals) for (region, day), totals in sorted(groups.items())]
```

This analytical mode supports the **complete base or complete delta phase**.
It does not describe an arbitrary intermediate batch: a mid-file snapshot can
contain only some identities from a lane. Use dictionary replay for those small
snapshots. It also assumes that the files follow `strata-periodic-v1`; changing
an operation by hand does not change the metadata formula. We therefore
cross-check generator, replay, and formula on small cases and retain handwritten
fixtures whose answers do not depend on either recipe implementation.

For $N=10^{12}$, $q=10^9$ and $t=0$. Every group answer is the corresponding
1000-event answer multiplied by $10^9$, subject to the same unsigned checks.
The test computes this formula only. It does not generate, import, or claim to
have processed a trillion events in the engine.

## Compare ordered results without converting integers to floats

The CSV reader returns strings by default. Its documentation also specifies
opening file objects with `newline=""`; see
[Python's CSV reader](https://docs.python.org/3/library/csv.html#csv.reader).
Keep that behavior and call `unsigned` yourself. Do not use an option that
converts numeric fields to floating point: values near $2^{64}$ cannot be
validated that way.

Comparison is a streaming walk over the actual file. It rejects the wrong
header, missing rows, wrong widths, changed values, and extra rows. The expected
rows are sorted, so a duplicate or out-of-order actual group is also rejected.
For an empty answer, a header with no data rows is correct.

```python title="scripts/reference_query.py"
def compare(path, expected):
    with path.open(newline="", encoding="ascii") as source:
        rows = csv.reader(source, strict=True)
        if next(rows, None) != HEADER:
            raise ValueError("answer CSV header")
        for index, wanted in enumerate(expected):
            row = next(rows, None)
            if row is None or len(row) != 4:
                raise ValueError(f"answer row {index}: missing/wrong width")
            actual = tuple(unsigned(x) for x in row)
            if actual != wanted:
                raise ValueError(f"answer row {index}: expected {wanted}, got {actual}")
        if next(rows, None) is not None:
            raise ValueError("extra answer row")
```

The command-line entry point validates the day interval and chooses replay or
formula explicitly. Supplying `--actual` compares a completed output; omitting
it prints the expected CSV. Errors name the selected mode and input paths.
The reference does not create an expected file next to the database or consult
any table, cache, or manifest bytes.

```python title="scripts/reference_query.py"
def main():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("mode", choices=("replay", "formula"))
    parser.add_argument("inputs", nargs="+", type=Path)
    parser.add_argument("--sequence", type=int)
    parser.add_argument("--phase", choices=("base", "delta"), default="delta")
    parser.add_argument("--begin", type=int, default=0)
    parser.add_argument("--end", type=int, default=1 << 32)
    parser.add_argument("--max-keys", type=int, default=100000)
    parser.add_argument("--actual", type=Path)
    args = parser.parse_args()
    try:
        if not (0 <= args.begin <= args.end <= 1 << 32 and args.max_keys > 0):
            raise ValueError("query bounds/key cap")
        if args.mode == "replay":
            if args.sequence is None or not 0 <= args.sequence <= U64:
                raise ValueError("replay needs an unsigned --sequence")
            expected = replay(args.inputs, args.sequence, args.begin, args.end, args.max_keys)
        else:
            if len(args.inputs) != 1:
                raise ValueError("formula needs one meta.json")
            expected = formula(json.loads(args.inputs[0].read_text()), args.phase, args.begin, args.end)
        if args.actual:
            compare(args.actual, expected)
            print(f"verified groups={len(expected)} mode={args.mode}")
        else:
            writer = csv.writer(sys.stdout, lineterminator="\n")
            writer.writerow(HEADER)
            writer.writerows(expected)
    except (OSError, ValueError, OverflowError, KeyError, UnicodeError, csv.Error) as error:
        parser.exit(1, f"reference {args.mode} inputs={args.inputs}: {error}\n")


if __name__ == "__main__":
    main()
```

## First runnable checks

These commands need only Python and the three completed files above, not the
C++ executable. Run them from the learner source root. The chosen directory
must not already exist.

```sh title="Terminal — generate and compare two independent small answers"
python3 scripts/generate_data.py /tmp/strata-workload-small
python3 scripts/reference_query.py replay \
  /tmp/strata-workload-small/base.ops /tmp/strata-workload-small/delta.ops \
  --sequence 5 --begin 1 --end 5 > /tmp/strata-old-reference.csv
python3 scripts/reference_query.py formula /tmp/strata-workload-small/meta.json \
  --phase base --begin 1 --end 5 --actual /tmp/strata-old-reference.csv
```

The last command prints `verified groups=8 mode=formula`. The base metadata
reports sequence 5, 137 operations, and 3838 logical bytes. Evaluating sequence
6 or `--phase delta` yields twelve groups, not eight. Changing only the seed
or batch size requires reading the resulting metadata instead of reusing these
sequence constants.

The next part turns the same format into a bounded C++ source of real mutations.
The two implementations should agree about valid inputs, but they must not share
parser code that would make the same error invisible.
