# Chapter 8 — Indexed immutable tables

## Part 1 — A bounded format and a streaming builder

A frozen memtable contains an ordered history, but its nodes and pointers disappear
with the process. Writing those nodes to a file would preserve neither their
meaning nor their ownership. We need a different representation: explicitly encoded
entries, grouped into independently checked blocks, with enough index information
to find one block without reading all the others.

The result is an **immutable sorted table**, sometimes called an SSTable. Here,
“immutable” is a protocol: finish a new file once, then never alter its contents
while a reader uses it. The operating system does not enforce that protocol for us.
This chapter builds and reads standalone tables. It does not yet decide which
files belong to a database, install a flush durably, or recover a write-ahead log.

> **Learning objectives**
> Encode a versioned table without copying C++ object representations; implement
> CRC-32 and a deterministic Bloom filter; build bounded data blocks from ordered
> entries; seek using a sparse directory; preserve old versions and tombstones;
> distinguish safe parsing, checksum detection, and full structural verification.

**Prerequisites:** the completed code from
[Chapter 6](https://25d-228.github.io/cplusplus-learning/chapter-06.html), including
its second part, and both parts of
[Chapter 7](https://25d-228.github.io/cplusplus-learning/chapter-07.html).
Keep the five language-foundation programs and all nine existing CTest entries.
The code below extends the same `work/strata` source tree.

**Implementation allocation: 5.0 hours for all three parts together.** Allow 1.75 hours
for the format and builder, 2.0 for the directory/filter/reader, and 1.25 for
boundary tests, corruption experiments, and diagnosis. Reading is separate. The
small limits used in tests are deliberate: boundary behavior should not require
gigabytes of data or a long benchmark.

### The interface we already have

Chapter 6 supplies `Key {relation, primary}`, explicit 30-byte event and 14-byte
account codecs, `put_le<4/8>`, `get_le<4/8>`, and checked arithmetic.
`File::write_all(offset, bytes)` handles partial writes and interruptions;
`read_exact(offset, bytes)` either fills the range or throws. `FormatError`
represents malformed bytes or unexpected end of file; `IoError` represents an
operating-system/backend error. A thrown write can leave a prefix in the file.

Chapter 7 supplies `EntryView {key, kind, value}`. Its `InternalKey` contains a
user key and a positive sequence. `InternalLess` orders relation and primary key
ascending, then sequence **descending**. `EntryKind::erase` is a tombstone with
no value. `MemTable::begin()` returns a cursor with `valid()`, `entry()`, and
`next()`. Entry values borrow the memtable arena; the table must outlive traversal.
`freeze()` forbids new preparations but does not create a thread-safety mechanism.

The writer copies each supplied entry before returning from `append`. It never
retains the caller's span. The convenience `write_table` requires a frozen
memtable. Both that traversal and all table operations in this chapter execute
on one thread, without another actor changing their input or file.

```sh title="Terminal — add the table files"
mkdir -p include/strata/storage src/storage tests scripts
```

Create `include/strata/storage/table.hpp`, `src/storage/table.cpp`,
`tests/table.cpp`, and `scripts/check_table_fixture.py`. Append the final CMake
block to the existing `CMakeLists.txt`; do not replace the earlier targets.
Repeated filename-labelled listings append in order. Part 1 leaves the header
and source namespace open; Part 2 completes them. Part 3 supplies the complete
tests, independent byte checker, and build integration. Write all three parts
before building. No listing requires an unwritten later chapter.

## Specify bytes before writing a class

Every multibyte integer below uses the Chapter 6 **little-endian** encoding.
Offsets count bytes from the beginning of the file. Lengths are encoded counts,
not `sizeof` a C++ structure. Padding, alignment, pointers, and host endianness
never appear in the format.

```text title="Table version 1 — physical order"
24-byte header
zero or more data frames, contiguous and ordered
one metadata frame: directory plus Bloom filter
40-byte footer, at the end of the file
```

The header establishes the version and the data-block payload bound:

| Byte offset | Width | Meaning |
|---|---:|---|
| 0 | 8 | ASCII bytes `STRATABL` |
| 8 | 4 | Format version, exactly 1 |
| 12 | 4 | Maximum data payload bytes, from 64 through 65,536 |
| 16 | 4 | Reserved, must be zero |
| 20 | 4 | CRC-32 of header bytes `[0,20)` |

A frame is `payload_length:u32`, `tag:u32`, payload bytes, then `crc:u32`.
The tag is 1 for data and 2 for metadata. The CRC covers the **eight-byte frame
header and payload**, not its own trailing four bytes. A payload of $P$ bytes
therefore occupies $P+12$ file bytes. Protecting the length and tag as well as
the data detects ordinary damage to the framing information; we still check
those fields independently before trusting their meaning.

A data payload starts with a four-byte entry count. Each entry has this layout:

| Entry-relative offset | Width | Meaning |
|---|---:|---|
| 0 | 1 | Relation tag: events = 1, accounts = 2 |
| 1 | 8 | Numeric primary key |
| 9 | 8 | Positive sequence |
| 17 | 1 | Kind: put = 1, erase = 2 |
| 18 | 4 | Value byte count |
| 22 | Variable | Encoded row, or no bytes for a tombstone |

Thus a put event costs $22+30=52$ bytes, a put account costs $22+14=36$,
and a tombstone costs 22. A value keeps its own Chapter 6 version/relation/key
fields. This small duplication lets validation catch disagreement between an
index key and its encoded row. We do not strip those fields and invent a second
row codec here.

A **directory entry** is 46 bytes: first internal key (17), last internal key
(17), data-frame offset (8), and complete frame length (4). Both endpoints are
stored so a reader can validate separation between blocks and later compare
an actual block with its advertised range. This is sparse: one entry per block,
not one per row.

The metadata payload is `block_count:u32`, `filter_bytes:u32`,
`probe_count:u32`, followed by the 46-byte directory entries and the filter.
There is no alignment padding. For $D$ data blocks and $F$ filter bytes, its
payload length $M$ is

$$
M = 12 + 46D + F.
$$

The footer locates that bounded metadata without first scanning the data:

| Footer-relative offset | Width | Meaning |
|---|---:|---|
| 0 | 8 | ASCII bytes `STRATEND` |
| 8 | 8 | Metadata frame offset |
| 16 | 4 | Complete metadata frame length |
| 20 | 8 | Number of stored versions, including tombstones |
| 28 | 8 | Total file byte count, including this footer |
| 36 | 4 | CRC-32 of footer bytes `[0,36)` |

For an empty table with a 32-byte filter, $D=0$, $M=44$, and the file occupies
$24+(44+12)+40=120$ bytes. There are no empty data frames. The metadata and footer
still exist, so “empty table” is not confused with “unfinished empty file.”

These are Strata's version-1 choices, not an existing database's file format.
Explicit file headers, versions, and byte offsets are also visible in the
[SQLite file-format specification, section 1.3](https://www.sqlite.org/fileformat.html#the_database_header);
we do not reuse its page or record layout.

## Put a ceiling on every file-controlled count

Start the header. The standard headers are explicit even where a prerequisite
header happens to include one of them transitively.

```cpp title="include/strata/storage/table.hpp"
#ifndef STRATA_STORAGE_TABLE_HPP
#define STRATA_STORAGE_TABLE_HPP

#include "strata/io/file.hpp"
#include "strata/storage/memtable.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <limits>
#include <optional>
#include <span>
#include <vector>

namespace strata::storage {
```

The hard ceilings belong to this format implementation. A caller may choose
smaller writer limits, never larger ones. One table may occupy at most 64 MiB;
this is not a limit on a later database containing many tables. A filter has
8–65,536 bytes, a power of two, and always uses six probes. Metadata has a
256 KiB payload ceiling, and there are at most 4,096 directory entries.

```cpp title="include/strata/storage/table.hpp"
namespace table_format {
inline constexpr std::size_t header_bytes = 24;
inline constexpr std::size_t footer_bytes = 40;
inline constexpr std::size_t frame_overhead = 12;
inline constexpr std::size_t directory_bytes = 46;
inline constexpr std::size_t max_block_bytes = 64 * 1024;
inline constexpr std::size_t max_blocks = 4096;
inline constexpr std::size_t max_filter_bytes = 64 * 1024;
inline constexpr std::size_t max_metadata_bytes = 256 * 1024;
inline constexpr std::uint64_t max_file_bytes = 64 * 1024 * 1024;
inline constexpr std::uint32_t bloom_probes = 6;

std::uint32_t crc32(std::span<const std::byte> input,
                    std::uint32_t previous = 0) noexcept;
void bloom_add(std::span<std::byte> bits, Key key);
bool bloom_may_contain(std::span<const std::byte> bits, Key key);
}  // namespace table_format
```

For the largest accepted directory and filter, $M=12+46(4096)+65536=253964$,
which is below 262,144. The count checks make subsequent directory arithmetic
bounded; the reader must perform them **before** computing an untrusted layout.
The `crc32` argument `previous = 0` permits both whole-buffer and incremental
checksums. The Bloom functions operate on borrowed bit arrays rather than
allocating their own storage.

```cpp title="include/strata/storage/table.hpp"
struct TableOptions {
    std::size_t block_bytes = 4096;
    std::size_t block_limit = table_format::max_blocks;
    std::size_t filter_bytes = 8192;
    std::uint64_t file_limit = table_format::max_file_bytes;
};

struct TableInfo {
    std::uint64_t file_bytes;
    std::uint64_t entries;
    std::size_t blocks;
};
```

`TableInfo` is returned only after the footer write finishes. It records completed
logical structure, not synchronization, database installation, or power-loss
survival. A default-constructed `TableInfo{}` in a test is just zeroed test state;
it is not an open or valid table.

## Own buffers, borrow the output file

The builder owns two vectors: a fixed-size data payload buffer and a fixed-size
metadata workspace. It borrows the `File`. The file and its backend must outlive
the builder and remain exclusively available to it. The constructor requires a
new empty file; it never overwrites a completed table in place.

A builder is not movable: moving a partly written operation would complicate its
borrow and state without benefiting this synchronous interface. The destructor
is the ordinary member destructor; it frees buffers and **does not call
`finish`**. Successful finalization is an explicit, fallible operation.

```cpp title="include/strata/storage/table.hpp"
class TableBuilder {
public:
    explicit TableBuilder(io::File& file, TableOptions options = {});
    TableBuilder(const TableBuilder&) = delete;
    TableBuilder& operator=(const TableBuilder&) = delete;
    TableBuilder(TableBuilder&&) = delete;
    TableBuilder& operator=(TableBuilder&&) = delete;
    void append(EntryView entry);
    TableInfo finish();

private:
    enum class State { open, finished, failed };
    void require_open() const;
    void flush();
    std::span<std::byte> filter() noexcept;
    io::File& file_;
    TableOptions options_;
    std::vector<std::byte> block_;
    std::vector<std::byte> metadata_;
    std::size_t used_ = 4;
    std::uint32_t rows_ = 0;
    std::size_t blocks_ = 0;
    std::uint64_t offset_ = table_format::header_bytes;
    std::uint64_t entries_ = 0;
    std::optional<InternalKey> previous_;
    InternalKey first_{};
    State state_ = State::open;
};

TableInfo write_table(io::File& file, const MemTable& table,
                      TableOptions options = {});
```

`previous_` contains the last accepted internal key; it is absent before the first
entry. `first_` is meaningful only while the current block contains rows.
`used_` begins at four to reserve the block's entry-count prefix. The three
states make an important policy visible: any exception during `append` or
`finish` makes that builder unusable. Retrying a partly written frame at an
uncertain offset is not permitted.

The declaration uses only the language already developed: member initialization,
borrowed references, optional values, noncopyable owners, private helpers, and an
enumeration. We now implement every helper, starting with the byte primitives.

## A checksum is an implemented recurrence

Begin the source file:

```cpp title="src/storage/table.cpp"
#include "strata/storage/table.hpp"
#include "strata/core/checked.hpp"

#include <algorithm>
#include <array>
#include <cstring>
#include <stdexcept>
#include <utility>

namespace strata::storage {
namespace table_format {
```

We use the reflected CRC-32 convention with polynomial constant `0xedb88320`,
initial internal state `0xffffffff`, and final XOR `0xffffffff`. The recurrence
uses shifts and XOR, not a dependency or an instruction specific to one CPU.
The polynomial and conditioning agree with
[RFC 1952, Appendix 8](https://www.rfc-editor.org/rfc/rfc1952.html#section-8).
The implementation below processes bits directly rather than copying its
lookup-table implementation.

```cpp title="src/storage/table.cpp"
std::uint32_t crc32(std::span<const std::byte> input,
                    std::uint32_t previous) noexcept {
    std::uint32_t state = previous ^ 0xffffffffU;
    for (const auto byte : input) {
        state ^= std::to_integer<std::uint32_t>(byte);
        for (unsigned bit = 0; bit < 8; ++bit) {
            const bool low = (state & 1U) != 0;
            state >>= 1;
            if (low) { state ^= 0xedb88320U; }
        }
    }
    return state ^ 0xffffffffU;
}
```

For each byte, XOR that byte into the low eight bits, then perform eight reduction
steps. When the low bit is set, the shift is followed by XOR with the reflected
polynomial. For the ASCII byte `1`, `0xffffffff ^ 0x31 = 0xffffffce`.
The first reduction yields `0x7fffffe7`; the second shifts and XORs, yielding
`0xd2477cd3`. Continuing all eight steps, and then the remaining bytes, gives
`crc32("123456789") = 0xcbf43926`. The test uses that fixed known answer.

`previous` is an already finalized CRC. XORing it with `0xffffffff` recovers the
internal state before extending it. Therefore `crc32(payload, crc32(header))`
checks the concatenation without constructing a third buffer. Empty input with
previous zero returns zero.

A CRC is not authentication. Different byte strings can have the same CRC,
and someone who edits data can recompute its CRC. Checksums detect many accidental
changes; validators must still reject wrong lengths, tags, order, and row shapes.
No test in this chapter proves resistance to hostile file forgery.

## Hash the user key, not its current version

A Bloom filter starts with zero bits. Inserting a key sets several positions;
querying an absent bit proves that the key was not inserted into that correctly
constructed, unchanged filter. All positions being set means only “continue to
the table.” Versions and kinds do not belong in the hashed key: a lookup starts
with a user key and a visibility sequence, not a known stored sequence.

We feed exactly nine bytes to an FNV-1a accumulation: the relation byte followed
by the primary key's eight little-endian bytes. The multiplication and XOR order
and 64-bit constants follow [RFC 9923, sections 2 and 5](https://www.rfc-editor.org/rfc/rfc9923.html#section-2).
Two mixed results then produce a start and an odd step. `mix` uses the output
mixing steps of [Vigna's public-domain SplitMix implementation](https://prng.di.unimi.it/splitmix.c),
not its evolving generator state. These are fixed format choices, not random seeds.

```cpp title="src/storage/table.cpp"
namespace {
bool valid_filter_size(std::size_t size) noexcept {
    return size >= 8 && size <= max_filter_bytes && (size & (size - 1)) == 0;
}

std::uint64_t mix(std::uint64_t value) noexcept {
    value ^= value >> 30;
    value *= 0xbf58476d1ce4e5b9ULL;
    value ^= value >> 27;
    value *= 0x94d049bb133111ebULL;
    return value ^ (value >> 31);
}

std::array<std::uint64_t, 2> hashes(Key key) noexcept {
    std::uint64_t hash = 14695981039346656037ULL;
    hash ^= static_cast<std::uint8_t>(key.relation);
    hash *= 1099511628211ULL;
    for (unsigned i = 0; i < 8; ++i) {
        hash ^= (key.primary >> (8 * i)) & 0xffU;
        hash *= 1099511628211ULL;
    }
    return {mix(hash), mix(hash ^ 0x9e3779b97f4a7c15ULL) | 1ULL};
}
}  // namespace
```

The unnamed namespace keeps these helpers private to this translation unit.
`valid_filter_size` checks the range before evaluating the power-of-two test.
For a positive integer, `size & (size - 1)` clears its least significant set bit;
it becomes zero exactly when there was just one set bit.

The multiplications intentionally wrap modulo $2^{64}$; the types and suffixes
are unsigned. That is defined arithmetic, unlike overflowing signed arithmetic;
see the [C++ unsigned-integer rules](https://eel.is/c++draft/basic.fundamental).
Do not replace these variables with signed `long`. The mixing steps distribute
input-bit influence; they do not make the results cryptographically strong or
prove that our two results are independent random hashes.

Let $m=8F$ be the bit count, $a$ the first hash, $b$ the odd step, and $i$ a probe
number from 0 through 5. We use

$$
p_i = (a+ib)\bmod m.
$$

Because $m$ is a power of two, reducing with `& (m - 1)` is equivalent to the
modulo operation. Also, $m$ divides $2^{64}$, so the intermediate unsigned wrap
does not change the final position. An odd step is relatively prime to this
$m$ and walks all positions before repeating. This prevents the six probes from
collapsing onto a shorter even-step cycle; it does not prevent collisions between
*different* keys.

```cpp title="src/storage/table.cpp"
void bloom_add(std::span<std::byte> bits, Key key) {
    if (!valid_filter_size(bits.size())) {
        throw std::invalid_argument("invalid Bloom byte count");
    }
    auto hash = hashes(key);
    const auto mask = static_cast<std::uint64_t>(bits.size() * 8 - 1);
    for (std::uint32_t i = 0; i < bloom_probes; ++i) {
        const auto position = static_cast<std::size_t>(hash[0] & mask);
        bits[position / 8] |= static_cast<std::byte>(1U << (position % 8));
        hash[0] += hash[1];
    }
}
```

A position $p$ belongs to byte $\lfloor p/8\rfloor$ and bit $p\bmod 8$.
`|=` sets that bit without clearing any existing one. Inserting the same user key
again is harmless. Importantly, inserting a tombstone sets bits too: a table
containing a deletion must not be skipped when looking up that key.

```cpp title="src/storage/table.cpp"
bool bloom_may_contain(std::span<const std::byte> bits, Key key) {
    if (!valid_filter_size(bits.size())) {
        throw FormatError("invalid Bloom byte count");
    }
    auto hash = hashes(key);
    const auto mask = static_cast<std::uint64_t>(bits.size() * 8 - 1);
    for (std::uint32_t i = 0; i < bloom_probes; ++i) {
        const auto position = static_cast<std::size_t>(hash[0] & mask);
        const auto bit = static_cast<std::byte>(1U << (position % 8));
        if ((bits[position / 8] & bit) == std::byte{0}) { return false; }
        hash[0] += hash[1];
    }
    return true;
}
}  // namespace table_format
```

For events key 42 with eight filter bytes, the low six start bits give 15 and
the low six step bits give 27. The positions are 15, 42, 5, 32, 59, and 22.
The resulting bytes are `20 80 40 00 01 04 00 08` in hexadecimal. The first probe
sets byte 1, bit 7; the second sets byte 5, bit 2. The known-answer test freezes
this mapping so a later “harmless” hash change cannot silently break old tables.
Changing these constants, input encoding, or probe rules requires a format change.

### Estimate selectivity without turning it into a guarantee

Under an ideal independent uniform-probe model, one insertion probe misses a
chosen bit with probability $1-1/m$. After $kn$ probes for $n$ distinct keys and
$k$ positions per key, the probability of that bit remaining zero is

$$
q_0 = (1-1/m)^{kn} \approx e^{-kn/m}.
$$

Approximating the query positions as independent then gives a false-positive
probability $p_{\mathrm{fp}}\approx(1-e^{-kn/m})^k$. For $m=1024$, $n=100$,
$k=6$, it is about 0.76%. Repeated versions do not increase $n$. This calculation
is a sizing guide, not an exact prediction for our deterministic mixer or one
finite workload. [Kirsch and Mitzenmacher, sections 1–2](https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf)
explain double hashing and the standard approximation; their random-hash analysis
is not a proof of our chosen constants. The test checks every inserted key and
reports absent-key positives without asserting a random-looking exact count.

## Decode internal keys before comparing them

Continue the source with file-local constants and validation helpers:

```cpp title="src/storage/table.cpp"
namespace {
using namespace table_format;
constexpr std::array<std::byte, 8> header_magic{
    std::byte{'S'}, std::byte{'T'}, std::byte{'R'}, std::byte{'A'},
    std::byte{'T'}, std::byte{'A'}, std::byte{'B'}, std::byte{'L'}};
constexpr std::array<std::byte, 8> footer_magic{
    std::byte{'S'}, std::byte{'T'}, std::byte{'R'}, std::byte{'A'},
    std::byte{'T'}, std::byte{'E'}, std::byte{'N'}, std::byte{'D'}};
constexpr std::uint32_t data_tag = 1;
constexpr std::uint32_t metadata_tag = 2;

void require(bool condition, const char* message) {
    if (!condition) { throw FormatError(message); }
}

void valid_key(Key key) {
    require(key.relation == Relation::events || key.relation == Relation::accounts,
            "unknown table relation");
}
```

`using namespace table_format` is confined to this source file's unnamed namespace;
it does not leak names into a public header. The local `require` throws
`FormatError` and remains active in Release builds.

```cpp title="src/storage/table.cpp"
void put_key(std::span<std::byte> out, InternalKey key) {
    if (out.size() != 17) { throw std::logic_error("internal key width"); }
    out[0] = static_cast<std::byte>(key.user.relation);
    put_le<8>(out.subspan(1, 8), key.user.primary);
    put_le<8>(out.subspan(9, 8), key.sequence);
}

InternalKey get_key(std::span<const std::byte> in) {
    require(in.size() == 17, "internal key width");
    InternalKey key{{static_cast<Relation>(std::to_integer<std::uint8_t>(in[0])),
                     get_le<8>(in.subspan(1, 8))}, get_le<8>(in.subspan(9, 8))};
    valid_key(key.user);
    require(key.sequence != 0, "stored sequence is zero");
    return key;
}
```

Numeric little-endian bytes are **not** lexicographically sorted key bytes.
For example, primary 255 begins `ff 00`, while 256 begins `00 01`. Comparing
those byte strings would put 256 first. Decoding and calling `InternalLess`
restores the relation/primary/descending-sequence order. A stored sequence of zero
is rejected; a search at visibility zero will be allowed later, meaning “before
all stored versions.”

```cpp title="src/storage/table.cpp"
void validate_entry(EntryView entry) {
    valid_key(entry.key.user);
    require(entry.key.sequence != 0, "stored sequence is zero");
    if (entry.kind == EntryKind::erase) {
        require(entry.value.empty(), "tombstone has a payload");
    } else {
        require(entry.kind == EntryKind::put, "unknown entry kind");
        const Key key = entry.key.user.relation == Relation::events
            ? key_of(decode_event(entry.value)) : key_of(decode_account(entry.value));
        require(key == entry.key.user, "row key disagrees with entry key");
    }
}
```

The put path uses the exact Chapter 6 decoder for the indicated relation, including
its size and record-version checks. It then compares the decoded user key with
the entry key. Erase never calls a row decoder and must have no payload.
`validate_entry` also checks caller-supplied builder input, so invalid data is
rejected before becoming a valid table image.

The block decoder advances an index only after validating one complete entry:

```cpp title="src/storage/table.cpp"
EntryView read_entry(std::span<const std::byte> payload, std::size_t& at) {
    require(at <= payload.size() && payload.size() - at >= 22,
            "truncated entry header");
    const auto in = payload.subspan(at);
    const auto key = get_key(in.first(17));
    const auto kind = static_cast<EntryKind>(std::to_integer<std::uint8_t>(in[17]));
    const auto size = static_cast<std::size_t>(get_le<4>(in.subspan(18, 4)));
    require(size <= event_bytes && size <= in.size() - 22, "invalid entry length");
    EntryView entry{key, kind, in.subspan(22, size)};
    validate_entry(entry);
    at += 22 + size;
    return entry;
}
```

The condition checks `at <= payload.size()` before subtracting. Once 22 bytes
remain, reading the length field is safe; the length must then fit both the
largest row encoding and the actual suffix. Only after those checks do we create
the value subspan. `at += 22 + size` is bounded by the already checked payload.
A parsed `EntryView` owns its numeric fields but **borrows its value bytes**.
The reader must keep the block alive, which Part 2 makes explicit.

## Frame writes without concatenating buffers

`write_frame` checks the final extent before any write in that frame. It constructs
only an eight-byte header and four-byte CRC trailer. The caller's payload stays
in its existing buffer.

```cpp title="src/storage/table.cpp"
std::uint64_t write_frame(io::File& file, std::uint64_t offset,
                          std::uint32_t tag, std::span<const std::byte> payload,
                          std::uint64_t limit) {
    const auto size = checked_size_add(payload.size(), frame_overhead);
    const auto end = checked_file_end(offset, size);
    if (end > limit) { throw std::length_error("table file cap"); }
    std::array<std::byte, 8> header{};
    put_le<4>(std::span{header}.first(4), payload.size());
    put_le<4>(std::span{header}.subspan(4, 4), tag);
    std::array<std::byte, 4> trailer{};
    put_le<4>(trailer, crc32(payload, crc32(header)));
    file.write_all(offset, header);
    file.write_all(offset + 8, payload);
    file.write_all(end - 4, trailer);
    return end;
}
```

There are three `write_all` calls, not three assumed atomic writes. If the second
one fails after ten payload bytes, the header and those bytes can already exist.
The returned offset is available only after all three calls succeed. The
builder will treat any error as terminal instead of guessing how much to retry.
A later failure can leave earlier **complete** blocks too; those are not an
installed table just because their checksums are correct.

We also provide the bounded inverse now. Its `size` comes from an already
validated extent, never from an unchecked allocation request in the frame itself.

```cpp title="src/storage/table.cpp"
std::span<const std::byte> read_frame(io::File& file, std::uint64_t offset,
                                     std::size_t size, std::uint32_t tag,
                                     std::span<std::byte> buffer) {
    require(size >= frame_overhead && size <= buffer.size(), "frame size bound");
    auto frame = buffer.first(size);
    file.read_exact(offset, frame);
    require(get_le<4>(frame.first(4)) == size - frame_overhead,
            "frame length mismatch");
    require(get_le<4>(frame.subspan(4, 4)) == tag, "frame tag mismatch");
    require(get_le<4>(frame.last(4)) == crc32(frame.first(size - 4)),
            "frame checksum mismatch");
    return frame.subspan(8, size - frame_overhead);
}
```

The on-disk length must agree with that extent. A maliciously huge length field
does not trigger a huge allocation: `read_frame` reads only into the supplied
bounded buffer, then rejects the mismatch. The returned span excludes both frame
header and CRC. Its lifetime is the caller's buffer lifetime, not a new allocation.

## Check configuration before allocating workspaces

```cpp title="src/storage/table.cpp"
std::size_t metadata_size(std::size_t blocks, std::size_t filter_bytes) {
    return checked_size_add(checked_size_add(12,
        checked_size_mul(blocks, directory_bytes)), filter_bytes);
}

TableOptions validate_options(TableOptions options) {
    if (options.block_bytes < 64 || options.block_bytes > max_block_bytes ||
        options.block_limit == 0 || options.block_limit > max_blocks ||
        !table_format::valid_filter_size(options.filter_bytes) ||
        options.file_limit < header_bytes + footer_bytes + frame_overhead +
                                 metadata_size(0, options.filter_bytes) ||
        options.file_limit > max_file_bytes ||
        metadata_size(options.block_limit, options.filter_bytes) > max_metadata_bytes) {
        throw std::invalid_argument("invalid table options");
    }
    return options;
}
}  // namespace
```

The order of the `||` tests matters. Invalid block/filter counts short-circuit
before layout calculations. `metadata_size` still uses checked multiplication
and addition, rather than relying on an informal bound in every caller.
The minimum allowed file limit fits the empty table for the configured filter.
It need not fit any nonempty table: that distinction is useful for testing cap
exhaustion.

Start the builder after closing the unnamed namespace:

```cpp title="src/storage/table.cpp"
TableBuilder::TableBuilder(io::File& file, TableOptions options)
    : file_(file), options_(validate_options(options)),
      block_(options_.block_bytes),
      metadata_(metadata_size(options_.block_limit, options_.filter_bytes)) {
    std::array<std::byte, 1> probe{};
    if (file_.read_up_to(0, probe) != 0) {
        throw std::invalid_argument("builder requires a new empty file");
    }
    std::array<std::byte, header_bytes> header{};
    std::copy(header_magic.begin(), header_magic.end(), header.begin());
    put_le<4>(std::span{header}.subspan(8, 4), 1);
    put_le<4>(std::span{header}.subspan(12, 4), options_.block_bytes);
    put_le<4>(std::span{header}.last(4), crc32(std::span{header}.first(20)));
    file_.write_all(0, header);
}

void TableBuilder::require_open() const {
    if (state_ != State::open) { throw std::logic_error("table builder is not open"); }
}

std::span<std::byte> TableBuilder::filter() noexcept {
    return std::span{metadata_}.last(options_.filter_bytes);
}
```

Members initialize in declaration order. `options_` is validated before its
values size either vector. Both vectors are created once; appending records does
not grow them. The zero-initialized metadata tail is the Bloom filter workspace.
Only after allocation succeeds do we probe the file and write the header.
If construction throws, vector members are destroyed and the borrowed file is
still the caller's responsibility. The probe is not a lock: no other writer may
race this operation.

`require_open` guards both public operations. The filter accessor selects the
fixed tail of the maximum metadata workspace while building. Until finalization,
unused directory space lies between the live directory prefix and that tail.

## Flush a block only after its last entry is known

```cpp title="src/storage/table.cpp"
void TableBuilder::flush() {
    if (rows_ == 0) { return; }
    if (blocks_ == options_.block_limit) { throw std::length_error("table block cap"); }
    put_le<4>(std::span{block_}.first(4), rows_);
    const auto end = write_frame(file_, offset_, data_tag,
                                 std::span{block_}.first(used_), options_.file_limit);
    auto entry = std::span{metadata_}.subspan(12 + blocks_ * directory_bytes,
                                             directory_bytes);
    put_key(entry.first(17), first_);
    put_key(entry.subspan(17, 17), *previous_);
    put_le<8>(entry.subspan(34, 8), offset_);
    put_le<4>(entry.subspan(42, 4), used_ + frame_overhead);
    offset_ = end;
    ++blocks_;
    rows_ = 0;
    used_ = 4;
}
```

A successful flush writes the current payload and then records its endpoints
and extent in memory. Those directory writes use already validated fixed-width
fields and reserved space. `previous_` must exist because `rows_ != 0`.
The payload count is stored just before writing, so it matches the finalized
block. We reset only the logical prefix length and row count; stale buffer bytes
beyond `used_` are never serialized.

At a 64-byte payload limit, an event put needs 52 entry bytes plus the four-byte
count, so one event fits and two do not. A tombstone takes 22, but a tombstone and
an event put need $4+22+52=78$, so they also separate. The deliberately small
fixture therefore forces three versions of events key 10 into different blocks.
Nothing in the format requires all versions of one user key to share a block.

```cpp title="src/storage/table.cpp"
void TableBuilder::append(EntryView entry) {
    require_open();
    try {
        validate_entry(entry);
        if (previous_ && !InternalLess{}(*previous_, entry.key)) {
            throw std::invalid_argument("table input is not strictly increasing");
        }
        const auto length = 22 + entry.value.size();
        if (length > block_.size() - used_) { flush(); }
        if (blocks_ == options_.block_limit) { throw std::length_error("table block cap"); }
        if (rows_ == 0) { first_ = entry.key; }
        auto out = std::span{block_}.subspan(used_, length);
        put_key(out.first(17), entry.key);
        out[17] = static_cast<std::byte>(entry.kind);
        put_le<4>(out.subspan(18, 4), entry.value.size());
        std::copy(entry.value.begin(), entry.value.end(), out.begin() + 22);
        bloom_add(filter(), entry.key.user);
        used_ += length;
        ++rows_;
        ++entries_;  // Bounded by block_limit * (block_bytes - 4) / 22.
        previous_ = entry.key;
    } catch (...) {
        state_ = State::failed;
        throw;
    }
}
```

Order is checked against the last accepted key **before** a possible flush.
Duplicates of an entire internal key are invalid; two versions of the same user
key are valid only in descending sequence order. A new row that would overflow
the current payload first flushes that payload, then starts the next block.
The smallest payload limit, 64, can hold the largest one-entry payload, 56.
Therefore `subspan(used_, length)` is safe after this decision without a special
oversized-row path.

The caller may change or destroy its input after `append` returns: key and value
bytes have been copied. The filter receives the user key regardless of kind.
The version count cannot approach `uint64_t` overflow: even a maximal all-tombstone
table is bounded by $4096\lfloor(65536-4)/22\rfloor$ entries. Physical file limits
can reject the table earlier.

The catch-all marks the builder failed and rethrows the original exception.
This includes malformed input and an ordering error, even if those checks happened
before I/O. The deliberately simple policy prevents a caller from continuing a
stream whose ordering or physical state it no longer understands. It does not
claim to roll back bytes.

## Finish the directory before writing the footer

During construction, the filter sits after space for the maximum directory. The
completed file should contain only actual entries. Finalization moves the filter
down to follow the used directory prefix.

The source and destination ranges may overlap. `std::memmove` is specifically for
that situation, unlike an overlapping `memcpy`. Both ranges here contain live
`std::byte` elements, not C++ owner objects. This operation changes positions
inside a fixed vector; it does not change the vector's allocation or capacity.
The C++ [cstring synopsis](https://eel.is/c++draft/cstring.syn) gives the corresponding
standard-library byte-operation interface.

```cpp title="src/storage/table.cpp"
TableInfo TableBuilder::finish() {
    require_open();
    try {
        flush();
        const auto size = metadata_size(blocks_, options_.filter_bytes);
        const auto frame_bytes = size + frame_overhead;
        const auto total = checked_file_end(offset_, frame_bytes + footer_bytes);
        if (total > options_.file_limit) { throw std::length_error("table file cap"); }
        const auto filter_offset = 12 + blocks_ * directory_bytes;
        std::memmove(metadata_.data() + filter_offset, filter().data(), options_.filter_bytes);
        put_le<4>(std::span{metadata_}.first(4), blocks_);
        put_le<4>(std::span{metadata_}.subspan(4, 4), options_.filter_bytes);
        put_le<4>(std::span{metadata_}.subspan(8, 4), bloom_probes);
        const auto footer_at = write_frame(file_, offset_, metadata_tag,
            std::span{metadata_}.first(size), options_.file_limit);
        std::array<std::byte, footer_bytes> footer{};
        std::copy(footer_magic.begin(), footer_magic.end(), footer.begin());
        put_le<8>(std::span{footer}.subspan(8, 8), offset_);
        put_le<4>(std::span{footer}.subspan(16, 4), frame_bytes);
        put_le<8>(std::span{footer}.subspan(20, 8), entries_);
        put_le<8>(std::span{footer}.subspan(28, 8), total);
        put_le<4>(std::span{footer}.last(4), crc32(std::span{footer}.first(36)));
        file_.write_all(footer_at, footer);
        state_ = State::finished;
        return {total, entries_, blocks_};
    } catch (...) {
        state_ = State::failed;
        throw;
    }
}
```

The total-file cap includes metadata framing and the footer, not just data blocks.
`flush()` can have written a last complete block before that final cap check fails;
the failed builder still cannot be resumed. After the filter moves, no further
append is permitted even in principle. Success installs `State::finished`; any
exception installs `State::failed`.

For the eight-version fixture, the first tombstone frame is 38 bytes, five event
put frames are 68 bytes each, and two account put frames are 52 bytes each.
The data ends at $24+38+5(68)+2(52)=506$. With eight directory entries and 32
filter bytes, the metadata payload is $12+8(46)+32=412$ bytes. Its frame takes
424 bytes, followed by 40 footer bytes: the complete file is **970 bytes**.
These numbers are derived from the format, not from one compiler's structure sizes.

No synchronization is hidden in `finish`. The caller can explicitly request
`File::sync` and handle its error, as the native fixture does in Part 3. Even a
successful synchronization does not install a database manifest or establish a
power-loss protocol. Those are later mechanisms, not missing lines in this builder.

## Connect the frozen memtable without collecting its rows

```cpp title="src/storage/table.cpp"
TableInfo write_table(io::File& file, const MemTable& table, TableOptions options) {
    if (!table.frozen()) { throw std::logic_error("flush input must be frozen"); }
    TableBuilder builder{file, options};
    for (auto cursor = table.begin(); cursor.valid(); cursor.next()) {
        builder.append(cursor.entry());
    }
    return builder.finish();
}
```

This loop retains one input view at a time. It neither sorts again nor builds a
second vector of all entries. `append` enforces the stream's order independently
of trusting the memtable. Every historical version and tombstone is written;
this is not compaction or visible-row materialization.

An input exception unwinds the builder and frees its two workspaces. The borrowed
file remains open until its owner closes it; the partial file may remain on disk.
A path-owning caller must discard a failed temporary output, never advertise it
as a finished table. Part 3 supplies that cleanup for a native fixture, without
requiring a future manifest implementation.

### Count working storage honestly

Let $B$ be the configured data payload bytes, $D_{\max}$ the writer's directory
capacity, and $F$ the filter bytes. Its vector **requested element storage** is

$$
A_{\mathrm{builder}} = B + 12 + 46D_{\max} + F.
$$

With defaults, that is $4096+12+46(4096)+8192=200716$ bytes. Headers, trailers,
objects, stack frames, allocator bookkeeping, and the input memtable are additional.
A standard vector's capacity and allocator overhead are not proved equal to its
requested size by this equation. The important current guarantee is that allocation
requests do not grow with an unchecked file or an unbounded row stream. Chapter 9
will turn reservations and actual capacities into explicit budget charges.

Continue to [Part 2 — Seek, verify, and diagnose](https://25d-228.github.io/cplusplus-learning/chapter-08-part-2.html)
for the complete reader and its ownership rules, then
[Part 3 — Verification and native checks](https://25d-228.github.io/cplusplus-learning/chapter-08-part-3.html)
for the runnable tests. All three parts form one chapter and one five-hour
implementation allocation.
