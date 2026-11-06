# Chapter 10 — Atomic batches and write-ahead recovery

An update changes event 100 from 500 cents to 700 cents and moves account 10 from
region 1 to region 2. Both changes belong to the same batch. The skip list can
prepare both nodes before linking either, but an arena disappears when the
process exits. A table file is not the answer yet: creating and installing a
replacement table for every small batch would be a different write path.

A **write-ahead log**, or WAL, records the whole batch before the in-memory index
publishes it. Reopening reads complete batches back into a new memtable. The hard
part is deciding what a failed call means. A failure before append can reject the
batch definitely. A failure after append starts can leave a complete recoverable
batch even though its caller never received success.

> **Learning objectives**
> Encode bounded, checksummed batches; distinguish definite rejection from an
> uncertain external effect; publish prepared nodes without allocating after
> synchronization; recover only complete batches; and repair a narrowly defined
> incomplete tail without disguising corruption as a successful reopen.

This chapter has three parts and one **4.5-hour implementation allocation**:
framing 1.0; the commit path 1.5; replay 1.0; fault checks 1.0. Reading is separate.
[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-10-part-2.html)
connects these codecs to file ownership and publication.
[Part 3](https://25d-228.github.io/cplusplus-learning/chapter-10-part-3.html)
supplies all tests, native checks, and build additions. All three parts form one
runnable increment; no later chapter is required.

## The prerequisite state and the new boundary

Use the complete code through
[Chapter 9](https://25d-228.github.io/cplusplus-learning/chapter-09.html), including
its arena, memtable, and table-reader migrations. In particular,
`MemTable(budget, arena_cap)` uses the common charged allocator; do not restore
the earlier uncharged constructor. The foundation executables remain unchanged.

`Mutation` owns its inline, at-most-30-byte row image. Its key contains a relation
tag and an unsigned primary key; `validate_mutation` checks the tag, operation,
exact row codec, and agreement between the key and encoded row. A tombstone has
no row bytes. `MemTable::prepare(sequence, changes)` returns a nonmovable prepared
token: it validates the batch, allocates nodes, copies row bytes, and retains a
rollback checkpoint. Its `commit()` is `noexcept`; abandonment destroys pending
nodes and restores the checkpoint. `find` returns a borrowed `EntryView`, not a
lifetime-owning snapshot.

`io::File` owns its descriptor, borrows a `FileOps` backend, retries interrupted
transfers, and advances offsets across short transfers. `write_all` can throw
after modifying a prefix. `sync` reports failures and does not silently downgrade
full-device synchronization. `memory::Vector<T>` charges allocator requests to
its explicit budget, and Chapter 8's `table_format::crc32` supports incremental
checksumming. We reuse that checksum rather than create another subtly different
implementation.

Create these five files. The filename-labelled listings in reading order create
their complete contents. Repeated blocks for one file append; Part 2 also names
explicit additions to the existing I/O code. Part 3 appends CMake targets.

```sh title="Terminal — WAL source and checks"
mkdir -p include/strata/storage src/storage tests scripts
```

The new files are `include/strata/storage/wal.hpp`, `src/storage/wal.cpp`,
`tests/wal_file_ops.hpp`, `tests/wal.cpp`, and `scripts/check_wal_fixture.py`.
There is no new dependency. `strata-core` already contains the checksum, record,
arena, and file implementations.

The result is a **single, exclusively owned log and memtable**, not a database
catalog. There is no directory lock, manifest, flush installation, multi-process
access, or concurrent reader publication yet. Only one thread uses this object
and its budget/backend at a time. Callbacks must not reenter it. Here, “atomic
batch” means indivisible logical visibility at operation boundaries and whole-batch
replay; it does not mean an atomic CPU instruction or one atomic filesystem write.

## Three answers instead of one Boolean

Let the visible sequence initially be 1. Preparing sequence 2 does not change the
visible sequence. Its frame can then be partly written, wholly written, synchronized,
published, and finally reported to the caller. These are distinct events:

| Last completed step | In-memory visible sequence | What reopening may find |
|---|---:|---|
| Validation or allocation failed before append | 1 | No new bytes from this attempt |
| Some frame bytes appended, then an error | 1 | A permitted incomplete tail, or possibly a complete batch under a broader I/O failure |
| Complete frame appended; synchronization reports failure | 1 | The complete sequence-2 batch may be present |
| Synchronization returned; process stopped before publication or response | 1, or no live process | Complete sequence 2 under the stated persistence assumptions |
| Publication and return completed | 2 | Sequence 2 must be retained under those assumptions |

A negative Boolean would collapse definite rejection and uncertainty. Returning
“failed, safe to retry” after a synchronization error would invent knowledge that
the process does not have. A repeated put might appear harmless in this fixture,
but it is still another sequence; retries of larger application actions are not
automatically exactly once.

A write result therefore contains an outcome, a reason, an optional-in-meaning
sequence number, and an integer I/O error. Sequence zero means no candidate was
accepted for append. For an uncertain result, the number identifies the candidate,
**not** a durable commit certificate. The writer remains poisoned until destroyed
and reopened; clearing a Boolean without replay is not a recovery procedure.

SQLite's [WAL overview, section 2](https://www.sqlite.org/wal.html#how_wal_works)
describes the general separation between appending changes and later checkpointing.
Strata's record format and replay policy below are its own: this is not SQLite's
page WAL, commit-record encoding, or recovery algorithm.

## Write a format, not an object representation

No pointer, `std::span`, `Mutation` object, enum padding, or node tower is copied
as a C++ object representation. Every multibyte integer below is unsigned and
little-endian. Lengths count **encoded bytes**, not `sizeof` a C++ structure.

The file starts with a fixed 32-byte header:

| Offset | Bytes | Meaning |
|---:|---:|---|
| 0 | 8 | ASCII `STRWAL01` |
| 8 | 4 | Format version, 1 |
| 12 | 4 | Header byte count, 32 |
| 16 | 8 | Base sequence before this log's first batch |
| 24 | 4 | Reserved, all zero |
| 28 | 4 | CRC-32 of bytes 0 through 27 |

Use base sequence zero for this standalone harness. The reader requires an
**expected** base and compares it with the encoded header. A later log generation
can begin after an installed sequence, but a nonzero base does not manufacture
older rows. The boundary tests use a large base solely to exercise sequence
arithmetic. This header is not a globally unique file identity or an authoritative
catalog; those are different responsibilities.

After the header, each batch has a 24-byte header, a payload, and a four-byte
trailer checksum:

| Relative offset | Bytes | Meaning |
|---:|---:|---|
| 0 | 4 | ASCII `BAT1` |
| 4 | 4 | Payload byte count |
| 8 | 8 | Batch sequence |
| 16 | 4 | Operation count |
| 20 | 4 | CRC-32 of batch-header bytes 0 through 19 |
| 24 | Payload length | Exactly the encoded operations |
| 24 + payload length | 4 | CRC-32 of the entire 24-byte header followed by the payload |

The small header checksum protects the length before it determines allocation
or a body read. The trailer checks the complete frame. A checksum is error
detection, not authentication, proof of persistence, or evidence that a caller
received success. CRC collisions remain possible. There is no separate commit
record: a complete, valid frame is the unit replay accepts.

Every operation has a 14-byte envelope: relation at offset 0 (one byte), kind at
1 (one byte), primary key at 2 (eight bytes), value length at 10 (four bytes), then
the row image. Put is kind 1; erase is kind 2. A put carries the existing 30-byte
event or 14-byte account encoding. Erase carries zero value bytes. Repeated keys
**within** one batch are rejected; changes to the same key in later batches are
new versions. Operation order is retained in the log, although memtable preparation
sorts its pending nodes for publication.

Let $n$ be the operation count and $v_i$ the value byte count of operation $i$.
The payload byte count $P$ and total frame byte count $F$ are

$$
P = \sum_{i=0}^{n-1}(14 + v_i), \qquad F = 24 + P + 4.
$$

Here $1 \le n \le 256$ and $v_i$ is 0, 14, or 30. Therefore

$$
14n \le P \le 44n \le 11264, \qquad F \le 11292.
$$

For the event/account update, $P=(14+30)+(14+14)=72$ bytes and $F=100$ bytes.
The first batch occupies file offsets 32 through 131, making its exclusive end
132. The second ends at 232. A later event deletion plus account put has payload
$14+(14+14)=42$ bytes and frame length 70, so the third end is 302. These values
will be checked by both C++ and a separately written Python decoder.

The physical log is capped at 64 MiB, where one MiB is $2^{20}$ bytes. This is an
implementation limit for this format/harness, not a measured throughput target.
There is also a common memory budget and a per-memtable arena cap; an admissible
log size does not guarantee sufficient replay memory.

```cpp title="include/strata/storage/wal.hpp"
#ifndef STRATA_STORAGE_WAL_HPP
#define STRATA_STORAGE_WAL_HPP

#include "strata/io/file.hpp"
#include "strata/memory/budget.hpp"
#include "strata/storage/memtable.hpp"

#include <cstddef>
#include <cstdint>
#include <span>

namespace strata::storage {

namespace wal_format {
inline constexpr std::size_t file_header_bytes = 32;
inline constexpr std::size_t batch_header_bytes = 24;
inline constexpr std::size_t operation_header_bytes = 14;
inline constexpr std::size_t checksum_bytes = 4;
inline constexpr std::size_t max_payload_bytes =
    MemTable::max_batch * (operation_header_bytes + event_bytes);
inline constexpr std::uint64_t max_log_bytes = 64 * 1024 * 1024;

memory::Vector<std::byte> encode_batch(memory::Budget& budget,
    std::uint64_t sequence, std::span<const Mutation> changes);
}  // namespace wal_format
```

These constants derive the maximum payload from the existing batch cap and codec
size. They are not copied from an ABI measurement. `encode_batch` returns an owning
charged vector; it retains no span into the caller's mutations. Its returned
storage must not outlive the budget supplied to its allocator.

Add the result types. A scoped enum requires qualification such as
`WriteOutcome::uncertain`; it cannot silently act as a Boolean success flag.

```cpp title="include/strata/storage/wal.hpp"
enum class WalOpen { create, recover };
enum class WriteOutcome { committed, rejected, uncertain };
enum class WriteReason {
    none, invalid_batch, resource_exhausted, log_full, sequence_exhausted,
    writer_poisoned, io_failure, unexpected_failure
};

struct WriteResult {
    WriteOutcome outcome;
    WriteReason reason;
    std::uint64_t sequence;  // Zero for rejection; candidate for uncertainty.
    int io_error = 0;
};
```

`WriteReason::writer_poisoned` describes the **newly rejected request**. It does
not resolve the outcome of the earlier uncertain request. `io_error` is the
backend error number, zero when no such number is available. Keeping the result
small and value-owned avoids constructing diagnostic strings after a successful
synchronization. Unexpected exceptions during append are classified conservatively;
Part 2 explains why the catch-all is restricted to that boundary.

Declare the owner next. `WalOpen::create` requires an empty writable file;
`WalOpen::recover` requires an existing writable WAL of the expected format/base.
Both opening modes take ownership of the supplied `File`. The backend and budget
must outlive the owner. Construction failure closes the moved-in descriptor.

```cpp title="include/strata/storage/wal.hpp"
class WalMemTable {
public:
    WalMemTable(memory::Budget& budget, io::File file, std::size_t arena_cap,
                WalOpen open_mode, io::SyncMode sync_mode,
                std::uint64_t base_sequence = 0);
    WalMemTable(const WalMemTable&) = delete;
    WalMemTable& operator=(const WalMemTable&) = delete;
    WalMemTable(WalMemTable&&) = delete;
    WalMemTable& operator=(WalMemTable&&) = delete;

    [[nodiscard]] WriteResult write(std::span<const Mutation> changes) &;
    const MemTable& table() const & noexcept { return table_; }
    const MemTable& table() const && = delete;
    std::uint64_t sequence() const noexcept { return sequence_; }
    std::uint64_t log_end() const noexcept { return end_; }
    bool poisoned() const noexcept { return poisoned_; }
    bool repaired_tail() const noexcept { return repaired_tail_; }

private:
    void initialize(std::uint64_t base);
    void replay(std::uint64_t expected_base);
    memory::Budget& budget_;
    io::File file_;
    MemTable table_;
    io::SyncMode sync_mode_;
    std::uint64_t sequence_;
    std::uint64_t end_ = wal_format::file_header_bytes;
    bool poisoned_ = false;
    bool repaired_tail_ = false;
};

}  // namespace strata::storage
#endif
```

The declaration order matters: `table_` is destroyed before `file_`, and both
before the external budget may disappear. The reference member does not own the
budget. No custom destructor is needed: member owners provide cleanup. The
existing `File` destructor is nonthrowing and is not a replacement for explicit
synchronization or a way to report a delayed close failure.

The class cannot be copied or moved, because its prepared tokens and borrowed
views depend on stable owner identity. The lvalue-qualified `table()` permits a
borrow only from an lvalue owner, just like earlier view accessors. This does not
extend its lifetime. A `const MemTable&` blocks normal mutation through that API;
it does not make concurrent access safe.

`sequence()` is the published log sequence, possibly the base before any new
rows exist. `log_end()` is the end of the last **published** frame. After an
uncertain append, the physical file can be longer. `repaired_tail()` reports
whether opening removed a nonempty permitted tail; it is not a continuously
updated “file damaged” flag. `[[nodiscard]]` makes an ignored write result a
useful compiler diagnostic, not a runtime control-flow guarantee.

## Validate the request before allocating its encoding

Start `src/storage/wal.cpp`. The internal helpers live in an unnamed namespace,
so they belong to this translation unit rather than the public library API.
The include of `table.hpp` reuses only the implemented CRC-32 routine; no table
builder, reader, or cache participates in logging.

```cpp title="src/storage/wal.cpp"
#include "strata/storage/wal.hpp"
#include "strata/core/codec.hpp"
#include "strata/storage/table.hpp"

#include <algorithm>
#include <array>
#include <limits>
#include <new>
#include <stdexcept>
#include <utility>

namespace strata::storage {
namespace {
using namespace wal_format;
using table_format::crc32;
constexpr std::array<std::byte, 8> file_magic{
    std::byte{'S'}, std::byte{'T'}, std::byte{'R'}, std::byte{'W'},
    std::byte{'A'}, std::byte{'L'}, std::byte{'0'}, std::byte{'1'}};
constexpr std::array<std::byte, 4> batch_magic{
    std::byte{'B'}, std::byte{'A'}, std::byte{'T'}, std::byte{'1'}};

void require_wal(bool condition, const char* message) {
    if (!condition) { throw FormatError(message); }
}
```

`require_wal` classifies invalid persisted bytes as `FormatError`. It does not
catch an I/O error and relabel it as end-of-file. That distinction is essential:
a read returning zero at EOF can justify a tail decision; a failed read cannot.

The payload-size function checks every mutation and rejects duplicate storage
keys before allocating the frame. It uses a bounded nested loop, deliberately
simpler than introducing another temporary sort buffer.

```cpp title="src/storage/wal.cpp"
std::size_t payload_size(std::span<const Mutation> changes) {
    if (changes.empty() || changes.size() > MemTable::max_batch) {
        throw std::length_error("WAL batch must contain 1..256 operations");
    }
    std::size_t bytes = 0;
    for (std::size_t i = 0; i < changes.size(); ++i) {
        validate_mutation(changes[i]);
        for (std::size_t j = 0; j < i; ++j) {
            if (changes[j].key == changes[i].key) {
                throw std::invalid_argument("duplicate WAL batch key");
            }
        }
        bytes += operation_header_bytes + changes[i].length;
    }
    return bytes;  // At most 256 * (14 + 30) = 11264.
}
```

For $n$ operations the duplicate check makes $n(n-1)/2$ key comparisons, at most
32640 for the cap of 256. This is a stated small-batch cost, not an unbounded
quadratic ingest algorithm. The memtable will validate defensively again; the
codec can also be exercised independently.

`validate_mutation` runs before adding each length. Once it succeeds, the value
length is bounded by the inline array, so the sum cannot exceed 11264. This proof
is why an ordinary addition is safe here. It would stop being valid if the batch
cap or payload representation became unbounded. `FormatError` from an invalid
row codec and `invalid_argument` from an invalid operation both mean a bad caller
batch on encoding; the writer will translate them before append.

The next helper encodes the file header. Value initialization makes the reserved
bytes zero. A reader regenerates the one expected header and compares all 32 bytes,
checking the version, length, base, reserved bytes, and CRC together.

```cpp title="src/storage/wal.cpp"
std::array<std::byte, file_header_bytes> file_header(std::uint64_t base) {
    std::array<std::byte, file_header_bytes> image{};
    auto bytes = std::span{image};
    std::copy(file_magic.begin(), file_magic.end(), image.begin());
    put_le<4>(bytes.subspan(8, 4), 1);
    put_le<4>(bytes.subspan(12, 4), file_header_bytes);
    put_le<8>(bytes.subspan(16, 8), base);
    // Bytes 24..27 are reserved zero bytes.
    put_le<4>(bytes.last(4), crc32(bytes.first(28)));
    return image;
}
```

Comparing with the expected header is intentionally stricter than accepting
unknown reserved fields or silently upgrading formats. A short or damaged file
header is fatal: it is not an incomplete final **batch**. A failed initial create
can leave an unusable file, and this harness will not overwrite it on reopen.

## Decide what a partial header can tell us

A frame can end before all 24 header bytes exist. We cannot verify the complete
header checksum then. We can still reject contradictions already visible in the
prefix: wrong magic, an out-of-range complete length, a wrong complete sequence,
or an impossible complete count. Only fields whose full bytes are present are
interpreted. Half of an eight-byte integer is not an integer padded with zeroes.

`check_prefix` receives between one and 24 bytes. Its caller has already rejected
sequence exhaustion and passes the exact next sequence. It does not itself decide
to ignore data; it validates the prefix before replay makes the EOF decision.

```cpp title="src/storage/wal.cpp"
void check_prefix(std::span<const std::byte> header, std::uint64_t next) {
    const auto magic_count = std::min(header.size(), batch_magic.size());
    require_wal(std::equal(header.begin(), header.begin() +
        static_cast<std::ptrdiff_t>(magic_count), batch_magic.begin()),
        "bad WAL batch magic");
    if (header.size() >= 8) {
        const auto bytes = get_le<4>(header.subspan(4, 4));
        require_wal(bytes >= operation_header_bytes && bytes <= max_payload_bytes,
                    "WAL payload bound");
    }
    if (header.size() >= 16) {
        require_wal(get_le<8>(header.subspan(8, 8)) == next,
                    "noncontiguous WAL sequence");
    }
    if (header.size() >= 20) {
        const auto count = get_le<4>(header.subspan(16, 4));
        const auto bytes = get_le<4>(header.subspan(4, 4));
        require_wal(count >= 1 && count <= MemTable::max_batch,
                    "WAL operation count bound");
        require_wal(bytes >= count * operation_header_bytes &&
                    bytes <= count * (operation_header_bytes + event_bytes),
                    "WAL payload/count mismatch");
    }
    if (header.size() == batch_header_bytes) {
        require_wal(get_le<4>(header.last(4)) == crc32(header.first(20)),
                    "WAL batch header checksum");
    }
}
```

The first comparison consumes only the available magic bytes. At lengths 8, 16,
and 20 the corresponding fields are complete. At length 24 the checksum must
also match. The count/length inequalities are only outer bounds; a payload with
the right aggregate length can still contain malformed individual records.

Consider a 17-byte final header prefix for sequence 2. It contains all magic,
length, and sequence bytes, but only the first count byte. A correct prefix passes
these checks; replay can remove it because EOF occurs before the header is
complete. Changing its complete sequence to 3 makes opening fail instead. A
single byte `B` after the last complete frame is an indistinguishable valid prefix
of `BAT1`; a single byte `X` is not. No checksum scheme can reconstruct an absent
acknowledgement history from that one byte. Part 2 states the failure model that
makes ignoring such a permitted prefix meaningful.

## Decode a whole payload before linking any row

Only a complete frame with a valid trailer checksum reaches `decode_payload`.
The count has already been limited to 1 through 256, and the body to 11264 bytes.
The decoder owns its mutation workspace through the common allocator; it does
not return views into a frame that will disappear at the next iteration.

```cpp title="src/storage/wal.cpp"
memory::Vector<Mutation> decode_payload(memory::Budget& budget,
    std::span<const std::byte> bytes, std::size_t count) {
    memory::Vector<Mutation> changes(count, memory::Allocator<Mutation>{budget});
    std::size_t at = 0;
    for (auto& change : changes) {
        require_wal(operation_header_bytes <= bytes.size() - at,
                    "short WAL operation header");
        const auto header = bytes.subspan(at, operation_header_bytes);
        change.key.relation = static_cast<Relation>(std::to_integer<std::uint8_t>(header[0]));
        change.kind = static_cast<EntryKind>(std::to_integer<std::uint8_t>(header[1]));
        change.key.primary = get_le<8>(header.subspan(2, 8));
        change.length = static_cast<std::size_t>(get_le<4>(header.subspan(10, 4)));
        at += operation_header_bytes;
        require_wal(change.length <= change.bytes.size() && change.length <= bytes.size() - at,
                    "WAL value length bound");
        std::copy_n(bytes.begin() + static_cast<std::ptrdiff_t>(at),
                    change.length, change.bytes.begin());
        at += change.length;
    }
    require_wal(at == bytes.size(), "trailing WAL payload bytes");
    try {
        (void)payload_size(changes);
    } catch (const std::invalid_argument&) {
        throw FormatError("invalid WAL mutation or duplicate key");
    }
    return changes;
}
}  // namespace
```

At the start of each iteration, `at <= bytes.size()`. Checking the 14-byte
header against `bytes.size() - at` therefore cannot underflow. After consuming
that header, the same subtraction bounds the value copy. Length is checked
against both the remaining payload and the destination's inline capacity before
`copy_n` sees it. The final equality rejects trailing payload bytes, including
an operation count smaller than the actual encoded list.

After structural parsing, `payload_size(changes)` reuses the schema/key/duplicate
checks. An `invalid_argument` now describes invalid **persisted data**, so it
becomes `FormatError`. A codec's existing `FormatError` propagates directly.
Allocation failure is not translated into corruption; reopening with more memory
may succeed. The bounded count prevents `payload_size`'s count exception here.

No node has been linked at any of these steps. A malformed second operation must
not leave the first operation committed. The closing brace in this block ends
the unnamed namespace, not `strata::storage`.

## Emit the frame in one charged workspace

Add the encoder at namespace scope. It allocates exactly one vector for the
complete batch image, fills fields and operation bytes, then fills the trailer.
The allocator charges its actual requested capacity; “one vector” is not an
assumption that the process uses no other memory.

```cpp title="src/storage/wal.cpp"
memory::Vector<std::byte> wal_format::encode_batch(memory::Budget& budget,
    std::uint64_t sequence, std::span<const Mutation> changes) {
    if (sequence == 0) { throw std::invalid_argument("zero WAL sequence"); }
    const auto payload = payload_size(changes);
    memory::Vector<std::byte> image(batch_header_bytes + payload + checksum_bytes,
                                    memory::Allocator<std::byte>{budget});
    auto bytes = std::span{image};
    std::copy(batch_magic.begin(), batch_magic.end(), image.begin());
    put_le<4>(bytes.subspan(4, 4), payload);
    put_le<8>(bytes.subspan(8, 8), sequence);
    put_le<4>(bytes.subspan(16, 4), changes.size());
    put_le<4>(bytes.subspan(20, 4), crc32(bytes.first(20)));
    std::size_t at = batch_header_bytes;
    for (const auto& change : changes) {
        bytes[at] = static_cast<std::byte>(change.key.relation);
        bytes[at + 1] = static_cast<std::byte>(change.kind);
        put_le<8>(bytes.subspan(at + 2, 8), change.key.primary);
        put_le<4>(bytes.subspan(at + 10, 4), change.length);
        at += operation_header_bytes;
        std::copy_n(change.bytes.begin(), change.length,
                    image.begin() + static_cast<std::ptrdiff_t>(at));
        at += change.length;
    }
    put_le<4>(bytes.last(4), crc32(bytes.first(at)));
    return image;
}
```

The loop's cursor `at` starts at 24. For the first event put it advances by 14
and then 30, reaching 68. The account put advances by 14 and 14, reaching 96.
The frame has 100 bytes; its last four receive the CRC of bytes 0 through 95.
The header CRC is already inside those bytes, so the trailer also covers it.

The encoder rejects sequence zero, but it does not impose a predecessor on a
standalone image. The writer supplies `sequence_ + 1`; replay verifies that exact
relationship. This separates a pure framing function from the state machine.

If returned-vector copy elision is not performed for the named local `image`,
its move transfers charged storage rather than copying all bytes. This is the
ordinary return of a movable vector, unlike the nonmovable prepared-token return
already developed in Chapter 7. The input span remains borrowed only during this
call. It must not be mutated concurrently or changed between encoding and
memtable preparation in the writer.

Before continuing, calculate the erase-plus-account frame without running it:
its payload is 42 bytes, its trailer starts at relative offset 66, and its total
length is 70. A value length of 1 on the erase is rejected even if the aggregate
payload length still falls inside $14n \le P \le 44n$. Aggregate bounds never
replace semantic validation.

`src/storage/wal.cpp` intentionally continues in Part 2; its outer namespace is
still open. Proceed to
[the commit and recovery state machine](https://25d-228.github.io/cplusplus-learning/chapter-10-part-2.html).
