# Chapter 13 — Streaming compaction and safe reclamation

## Part 1 — Replace a complete history prefix

A snapshot can keep a file alive after the database stops using it. That is a
correct lifetime property, but it does not by itself tell us when to delete the
file. Nor does merging several sorted files tell us which versions can safely
be omitted. This chapter connects those two decisions to the manifest's recovery
authority.

We implement a deliberately bounded policy: compact the **whole oldest table
prefix**, including any existing compacted base, into one or more size-capped
files. Newer tables and all logs remain outside the operation. This is an actual
streaming rewrite, not a map built from all records. The existing heap merge
selects live values; the builder decides exactly when the next row needs a new
file. After durable installation, an explicit collector removes only unreferenced
numbered files that no retained view needs.

> **Learning objectives**
> Prove when an obsolete version and a tombstone may disappear; encode a
> multi-file or empty base without inventing sequence gaps; forecast complete
> output size; preserve old readers across replacement; fence file identities
> before deletion; and distinguish transferred bytes, logical file sizes, and
> physically reclaimed space.

**Prerequisites:** the complete Chapter 12 implementation, including the Chapter
9 charged containers/cache, Chapter 10 `WalFileOps` test backend, Chapter 11
manifest installation and directory lock, and Chapter 12 owning views and heap
merge. Keep all existing tests. This chapter allocates **4.0 implementation
hours across all three parts**: 1.5 for selection/merge, 1.25 for
installation/reclamation, and 1.25 for interaction tests and counters. Reading is not
included. Reuse the earlier mechanisms rather than reimplementing the heap or
encodings. The engine remains single-threaded and nonreentrant.

[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-13-part-2.html)
implements installation and lifetime-aware cleanup.
[Part 3](https://25d-228.github.io/cplusplus-learning/chapter-13-part-3.html)
contains the complete test executable, independent native-file decoder, and build
instructions. Finish the changes in all three parts before building.

## Why a prefix, not an arbitrary selection?

For a user key $k$, let an entry's sequence be $s$. A put supplies a value; an erase
is a tombstone. Chapter 12 orders entries by user key ascending and sequence
descending, then consumes the whole key group. For a read boundary $e$, its answer
comes from the largest eligible sequence:

$$
s^*(k,e)=\max\{s : \text{an entry for }k\text{ exists and }s\leq e\}.
$$

An empty set means absent. A tombstone at $s^*(k,e)$ also means absent. This formula
says nothing about entries in a source we forgot to select.

Consider a put at sequence 1 in table A and an erase at sequence 2 in table B.
Compacting B alone and discarding its tombstone would leave A's put visible again.
Even a perfectly sorted, checksummed output would be semantically wrong. Instead,
select all installed table history through boundary $e=2$. There is now no
unselected older put to resurrect. The selected output can contain no row for
that key.

Our manifest makes the proof mechanical. Before the first compaction, tables
cover consecutive sequence intervals beginning at zero. After compaction, one
base represents all history in $(0,e]$, possibly using several key-partitioned
files. Later delta tables cover consecutive intervals starting at $e$. A legal
selection includes every base file and zero or more entire following delta tables;
without an existing base, select at least one table. Repacking just the base is
useful when larger output files can free table slots.
Every omitted table or log is newer than the selected end. Thus a tombstone that
wins within the selected prefix can be omitted along with all of its older puts.

An already-held sequence-1 view does **not** start using this sequence-2 base. It
continues owning table A. New views capture the current sources at a boundary at
least as new as the base. There is still no API that asks the current serving set
to reconstruct an arbitrary discarded historical sequence.

| Object | Before prefix replacement | After replacement |
|---|---|---|
| Current database | Old prefix, newer deltas, logs | New base, same newer deltas, same logs |
| Earlier view | Its captured source owners | Exactly the same owners |
| Uninstalled output | Not authoritative | Still not authoritative after a failed install |
| Collector | No deletion during replacement | Checks current authority and retained sources |

This policy is intentionally not a general leveled or tiered compaction picker.
It gives us a small, complete correctness argument while still requiring real
merge execution, bounded outputs, failure-safe installation, and delayed
reclamation. Rewriting a large oldest prefix repeatedly can be expensive. We will
measure that work rather than call the policy optimal.

## Give a compacted base an explicit format

Chapter 11's table record is `{id, base, end, bytes}`. Its interval describes
**coverage**, not just the minimum and maximum sequence physically left in the
file. For an ordinary flushed delta, both endpoints are witnessed by its rows.
After compaction, the newest live value for every key might come from sequence 1,
although the prefix also contains later deletions. Requiring an actual row at the
compacted end would either reject valid output or tempt us to invent a dummy row.

Introduce `base_end` and `base_count`. The first `base_count` table records are
ordered, disjoint user-key partitions of one base; each record has `base == 0` and
`end == base_end`. Their rows are puts with original sequences in that interval,
one row per user key. Remaining table records retain the old contiguous-delta
rules. A base whose last values were all deleted has a positive `base_end`, zero
base files, and no dummy table.

Keep the family magic `STRAMAN1`; the explicit version field selects the layout.
Version 1 remains byte-for-byte unchanged when `base_end == 0`. Version 2 uses the
former reserved field and appends one eight-byte header field:

| Offset | Width | Version 1 | Version 2 |
|---|---:|---|---|
| 0 | 8 | Family magic | Same family magic |
| 8 | 4 | Version 1 | Version 2 |
| 12 | 4 | Table count | Total table count, base files included |
| 16 | 4 | Log count | Log count |
| 20 | 4 | Zero | Base-file count |
| 24, 32, 40 | 8 each | Epoch, next ID, flushed boundary | Same meanings |
| 48 | 8 | First record begins here | Compacted base end |
| 56 | Variable | Already inside records | First record begins here |

All integers remain little-endian. There is still one trailing CRC-32 over all
preceding bytes. Let $T$ be total table records and $L$ log records. Each table
uses 32 bytes and each log 24. Including the checksum, exact lengths are

$$
N_1=52+32T+24L,\qquad N_2=60+32T+24L.
$$

At $T=16$ and $L=4$, $N_2=60+512+96=668$ bytes. An empty version-2 base with one
active log uses $60+24=84$ bytes. Counts are checked **before** they determine
subspans or loops. Bounds on directory key ranges require opening the files;
manifest checks alone cannot prove their contents.

Replace this header in full. The old constants, records, and codec declarations
remain available; `max_bytes` grows by eight, not by an unbounded variable header.

```cpp title="include/strata/storage/manifest.hpp — complete file"
#ifndef STRATA_STORAGE_MANIFEST_HPP
#define STRATA_STORAGE_MANIFEST_HPP

#include "strata/memory/budget.hpp"
#include "strata/storage/table.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <span>

namespace strata::storage {

struct TableRecord {
    std::uint64_t id = 0;
    std::uint64_t base = 0;
    std::uint64_t end = 0;
    std::uint64_t bytes = 0;
    bool operator==(const TableRecord&) const = default;
};
struct LogRecord {
    std::uint64_t id = 0;
    std::uint64_t base = 0;
    std::uint64_t end = 0;  // Zero only for the final, active log.
    bool operator==(const LogRecord&) const = default;
};
struct Manifest {
    static constexpr std::size_t max_tables = 16;
    static constexpr std::size_t max_logs = 4;
    static constexpr std::size_t header_bytes = 48;
    static constexpr std::size_t max_bytes =
        header_bytes + 8 + max_tables * 32 + max_logs * 24 + 4;
    std::uint64_t epoch = 0;
    std::uint64_t next_id = 1;
    std::uint64_t flushed = 0;
    std::uint64_t base_end = 0;
    std::size_t base_count = 0;
    std::size_t table_count = 0;
    std::size_t log_count = 0;
    std::array<TableRecord, max_tables> tables{};
    std::array<LogRecord, max_logs> logs{};
};
void validate_manifest(const Manifest& manifest);
memory::Vector<std::byte> encode_manifest(memory::Budget& budget,
                                         const Manifest& manifest);
Manifest decode_manifest(std::span<const std::byte> image);

}  // namespace strata::storage
#endif
```

Replace `src/storage/manifest.cpp` with the following four segments in order.
The first retains the family magic and the common rejection helper. A version
change is explicit data, not a guess made from file length.

```cpp title="src/storage/manifest.cpp — complete file, segment 1"
#include "strata/storage/manifest.hpp"
#include "strata/core/codec.hpp"

#include <algorithm>
#include <array>

namespace strata::storage {
namespace {
constexpr std::array<std::byte, 8> magic{
    std::byte{'S'}, std::byte{'T'}, std::byte{'R'}, std::byte{'A'},
    std::byte{'M'}, std::byte{'A'}, std::byte{'N'}, std::byte{'1'}};
void require_manifest(bool condition, const char* message) {
    if (!condition) { throw FormatError(message); }
}
}  // namespace
```

The validator advances `boundary` only for temporal delta intervals. It must
**not** advance once per base file: all those files describe the same temporal
prefix, partitioned by key. The shared `take_id` lambda still rejects duplicate
identities across tables and logs.

```cpp title="src/storage/manifest.cpp — complete file, segment 2"
void validate_manifest(const Manifest& m) {
    require_manifest(m.epoch != 0 && m.next_id != 0, "manifest counters");
    require_manifest(m.table_count <= Manifest::max_tables && m.log_count >= 1 &&
                     m.log_count <= Manifest::max_logs, "manifest counts");
    std::array<std::uint64_t, Manifest::max_tables + Manifest::max_logs> ids{};
    std::size_t used = 0;
    const auto take_id = [&](std::uint64_t id) {
        require_manifest(id != 0 && id < m.next_id, "manifest file identity");
        for (std::size_t i = 0; i < used; ++i) {
            require_manifest(ids[i] != id, "duplicate manifest file identity");
        }
        ids[used++] = id;
    };
    require_manifest(m.base_count <= m.table_count && m.base_end <= m.flushed &&
                     (m.base_end != 0 || m.base_count == 0), "manifest base coverage");
    std::uint64_t boundary = m.base_end;
    for (std::size_t i = 0; i < m.table_count; ++i) {
        const auto& t = m.tables[i];
        take_id(t.id);
        if (i < m.base_count) {
            require_manifest(t.base == 0 && t.end == m.base_end && t.end != 0,
                             "base file coverage mismatch");
        } else {
            require_manifest(t.base == boundary && t.end > t.base,
                             "table coverage is not a contiguous prefix");
            boundary = t.end;
        }
        require_manifest(t.bytes >= table_format::header_bytes + table_format::footer_bytes &&
                         t.bytes <= table_format::max_file_bytes, "manifest table size");
    }
    require_manifest(boundary == m.flushed, "manifest flushed boundary");
    for (std::size_t i = 0; i < m.log_count; ++i) {
        const auto& log = m.logs[i];
        take_id(log.id);
        require_manifest(log.base == boundary, "log coverage gap or overlap");
        if (i + 1 == m.log_count) {
            require_manifest(log.end == 0, "active log must have open end");
        } else {
            require_manifest(log.end > log.base, "empty or reversed sealed log");
            boundary = log.end;
        }
    }
}
```

For a base through sequence 3 split into five files, each of those first five
records says `(0,3]`. A following delta says `(3,4]`; `flushed` is 4. The active
log begins at 4 unless sealed logs extend the boundary. For an empty base through
2 followed by no delta, `boundary` begins and ends at 2 even though the table loop
has zero iterations. That is why absence of files is not absence of history.

The encoder allocates one charged, exactly sized byte vector. Its `emit` lambda
captures the current offset by reference, just as in Chapter 11. No table row or
pointer representation is copied into the manifest.

```cpp title="src/storage/manifest.cpp — complete file, segment 3"
memory::Vector<std::byte> encode_manifest(memory::Budget& budget, const Manifest& m) {
    validate_manifest(m);
    const auto header = Manifest::header_bytes + (m.base_end != 0 ? 8 : 0);
    const auto size = header + 32 * m.table_count + 24 * m.log_count + 4;
    memory::Vector<std::byte> image(size, memory::Allocator<std::byte>{budget});
    auto out = std::span<std::byte>{image};
    std::copy(magic.begin(), magic.end(), out.begin());
    put_le<4>(out.subspan(8, 4), m.base_end != 0 ? 2 : 1);
    put_le<4>(out.subspan(12, 4), m.table_count);
    put_le<4>(out.subspan(16, 4), m.log_count);
    put_le<4>(out.subspan(20, 4), m.base_count);
    put_le<8>(out.subspan(24, 8), m.epoch);
    put_le<8>(out.subspan(32, 8), m.next_id);
    put_le<8>(out.subspan(40, 8), m.flushed);
    if (m.base_end != 0) { put_le<8>(out.subspan(48, 8), m.base_end); }
    std::size_t at = header;
    const auto emit = [&](std::uint64_t value) {
        put_le<8>(out.subspan(at, 8), value);
        at += 8;
    };
    for (std::size_t i = 0; i < m.table_count; ++i) {
        const auto& t = m.tables[i];
        emit(t.id); emit(t.base); emit(t.end); emit(t.bytes);
    }
    for (std::size_t i = 0; i < m.log_count; ++i) {
        const auto& log = m.logs[i];
        emit(log.id); emit(log.base); emit(log.end);
    }
    put_le<4>(out.last(4), table_format::crc32(out.first(at)));
    return image;
}
```

The decoder first establishes the version-dependent minimum length, then verifies
the checksum and exact count-derived length. A checksummed version-2 image with
`base_end == 0` is rejected rather than becoming an ambiguous second spelling of
version 1. Semantic interval checks run after bounded decoding.

```cpp title="src/storage/manifest.cpp — complete file, segment 4"
Manifest decode_manifest(std::span<const std::byte> image) {
    require_manifest(image.size() >= Manifest::header_bytes + 24 + 4 &&
                     image.size() <= Manifest::max_bytes, "manifest length");
    require_manifest(std::equal(magic.begin(), magic.end(), image.begin()), "manifest magic");
    const auto version = get_le<4>(image.subspan(8, 4));
    require_manifest(version == 1 || version == 2, "manifest version");
    const auto header = Manifest::header_bytes + (version == 2 ? 8 : 0);
    require_manifest(image.size() >= header + 24 + 4, "manifest version length");
    require_manifest(version == 2 || get_le<4>(image.subspan(20, 4)) == 0,
                     "manifest reserved bits");
    require_manifest(get_le<4>(image.last(4)) == table_format::crc32(image.first(image.size() - 4)),
                     "manifest checksum");
    Manifest m;
    m.table_count = static_cast<std::size_t>(get_le<4>(image.subspan(12, 4)));
    m.log_count = static_cast<std::size_t>(get_le<4>(image.subspan(16, 4)));
    require_manifest(m.table_count <= Manifest::max_tables && m.log_count >= 1 &&
                     m.log_count <= Manifest::max_logs, "manifest counts");
    require_manifest(image.size() == header + 32 * m.table_count +
                     24 * m.log_count + 4, "manifest exact length");
    m.epoch = get_le<8>(image.subspan(24, 8));
    m.next_id = get_le<8>(image.subspan(32, 8));
    m.flushed = get_le<8>(image.subspan(40, 8));
    if (version == 2) {
        m.base_count = static_cast<std::size_t>(get_le<4>(image.subspan(20, 4)));
        m.base_end = get_le<8>(image.subspan(48, 8));
        require_manifest(m.base_end != 0, "version 2 requires a compacted boundary");
    }
    std::size_t at = header;
    const auto take = [&]() {
        const auto value = get_le<8>(image.subspan(at, 8));
        at += 8;
        return value;
    };
    for (std::size_t i = 0; i < m.table_count; ++i) {
        auto& t = m.tables[i];
        t.id = take(); t.base = take(); t.end = take(); t.bytes = take();
    }
    for (std::size_t i = 0; i < m.log_count; ++i) {
        auto& log = m.logs[i];
        log.id = take(); log.base = take(); log.end = take();
    }
    validate_manifest(m);
    return m;
}

}  // namespace strata::storage
```

This change is forward use of a new format by the updated reader. An old Chapter
11 reader is not expected to open version 2. Do not remove its version check to
make it do so. The updated reader still opens all valid version-1 directories.

## Stop before the next row exceeds the whole-file cap

Chapter 8's builder detects file-size failure, but discovering it in `finish()` is
too late for a streaming splitter. The row that caused the excess may already be
in the builder, and the input cursor may already have advanced. We need a
nonmutating question: **would appending this row and immediately finishing fit?**

Use these symbols for the existing builder fields. $u$ is `used_`, including the
four-byte count of the current data block. $b$ is the number of already flushed
data blocks, $o$ the offset immediately after them, $B$ the block-buffer capacity,
$f$ the Bloom-filter bytes, and $\ell=22+v$ the encoded row length for $v$ value
bytes. A data frame adds 12 bytes. Metadata with $n$ directory entries has payload
length $12+46n+f$, also wrapped in a 12-byte frame. The footer uses 40 bytes.

When $u+\ell\leq B$, finishing after this row needs $n=b+1$ blocks and

$$
S=o+(u+\ell+12)+12+(12+46n+f)+40.
$$

Otherwise the current block must flush, and the row starts a new block with its
own four-byte count. Now $n=b+2$ and

$$
S=o+(u+12)+(4+\ell+12)+12+(12+46n+f)+40.
$$

The 24-byte file header is already included in the initial $o$. This is a
forecast of actual framing, not a safety percentage. Event values use 30 bytes,
so $\ell=52$. With a 128-byte block and 32-byte filter, two event rows have
$u+\ell=108$ and one directory entry. Their whole file is

$$
24+(108+12)+12+(12+46+32)+40=286\text{ bytes}.
$$

A third event needs a second block and produces 400 bytes, not merely another
52 bytes. A cap of 286 therefore admits exactly two such rows; a cap of 285 does
not. The minimum block size is 64, so a validated maximum-sized row fits into a
fresh block: $4+52\leq64$. That fact prevents an empty-block special case in the
forecast.

In `table.hpp`, make the following exact declaration changes. In these small
unified diffs, lines beginning with `-` are removed, `+` lines are inserted, and
unprefixed context is retained; the markers are not C++ source. All algorithmic
functions appear in complete listings below.

```diff title="include/strata/storage/table.hpp — exact declaration changes"
--- a/include/strata/storage/table.hpp
+++ b/include/strata/storage/table.hpp
@@ -53,6 +53,7 @@
     TableBuilder& operator=(const TableBuilder&) = delete;
     TableBuilder(TableBuilder&&) = delete;
     TableBuilder& operator=(TableBuilder&&) = delete;
+    bool can_append(EntryView entry) const;
     void append(EntryView entry);
     TableInfo finish();
 
@@ -133,6 +134,9 @@
         std::numeric_limits<std::uint64_t>::max()) &;
     bool may_contain(Key key) const;
     void verify_all() &;
+    io::IoBytes io_bytes() const noexcept { return file_.io_bytes(); }
+    std::optional<Key> first_key() const;
+    std::optional<Key> last_key() const;
     std::uint64_t entries() const noexcept { return entries_; }
     std::size_t blocks() const noexcept { return blocks_; }
     std::uint64_t block_reads() const noexcept { return block_reads_; }
```

Add these functions inside `namespace strata::storage` in `table.cpp`, after its
existing definitions and before the final namespace-closing brace. They use the
existing `validate_entry`, `metadata_size`, and format constants from that same
translation unit; do not create duplicate helpers.

```cpp title="src/storage/table.cpp — add forecast and endpoints"
bool TableBuilder::can_append(EntryView entry) const {
    require_open();
    validate_entry(entry);
    if (previous_ && !InternalLess{}(*previous_, entry.key)) {
        throw std::invalid_argument("table input is not strictly increasing");
    }
    const auto length = 22 + entry.value.size();
    const bool split = length > block_.size() - used_;
    const auto final_blocks = blocks_ + (split ? 2 : 1);
    if (final_blocks > options_.block_limit) { return false; }
    // offset_ already includes every flushed frame, but not the current frame.
    const auto pending = split
        ? used_ + frame_overhead + 4 + length + frame_overhead
        : used_ + length + frame_overhead;
    const auto total = offset_ + pending + frame_overhead +
        metadata_size(final_blocks, options_.filter_bytes) + footer_bytes;
    return total <= options_.file_limit;
}

std::optional<Key> TableReader::first_key() const {
    return blocks_ == 0 ? std::nullopt : std::optional<Key>{directory(0).first.user};
}
std::optional<Key> TableReader::last_key() const {
    return blocks_ == 0 ? std::nullopt : std::optional<Key>{directory(blocks_ - 1).last.user};
}
```

`can_append` checks the builder state and the same strict internal ordering as
`append`. Invalid entries throw without modifying the builder. A valid entry that
would exceed either the block count or file cap returns `false`, leaving it ready
to finish. The caller retains the unconsumed row and offers it to the next file.
The arithmetic is bounded by the validated format limits: at most 4,096 blocks,
64 KiB block payloads, and 256 KiB metadata. It does not multiply attacker-provided
unbounded counts.

The endpoint getters expose authenticated directory keys, not proof that every
row is correct. Reopening a compacted base will first fully verify each table,
then require strictly increasing user keys inside it and strictly separated
ranges between files. Empty tables return `nullopt`; they are not admitted as
base partitions.

## Count work where successful transfers occur

File length and traffic answer different questions. Rewriting a 1,398-byte
output creates 1,398 logical bytes, but a failed operation may write a 24-byte
header and never finish any file. Reading a table may hit the block cache or may
reread a block. Metadata validation during reopen is not the same phase as input
merge traffic. Counters must have narrow names and precise boundaries.

Add per-`File` counters at Chapter 6's successful partial-transfer boundary. They
count bytes returned by `read_at` and `write_at`, including successful prefixes
before a later error. They do not count zero-progress errors, `EINTR`, or
synchronization requests, and they are not measurements of physical device I/O.
Overflow saturates at the largest unsigned 64-bit value and sets a flag, rather
than wrapping into a deceptively small number.

Replace `include/strata/io/file.hpp` in full:

```cpp title="include/strata/io/file.hpp — complete file"
#ifndef STRATA_IO_FILE_HPP
#define STRATA_IO_FILE_HPP

#include <cerrno>
#include <limits>
#include <cstddef>
#include <cstdint>
#include <span>
#include <string>
#include <system_error>

namespace strata::io {

enum class SyncMode { file, full_device };
enum class OpenMode { read_only, read_write, create_new };

struct Transfer {
    std::size_t count;
    int error;
};

class FileOps {
public:
    virtual ~FileOps() = default;
    virtual int truncate_to(int, std::uint64_t) noexcept { return ENOTSUP; }

    virtual Transfer read_at(int descriptor, std::span<std::byte> output,
                             std::uint64_t offset) noexcept = 0;
    virtual Transfer write_at(int descriptor, std::span<const std::byte> input,
                              std::uint64_t offset) noexcept = 0;
    virtual int sync(int descriptor, SyncMode mode) noexcept = 0;
    virtual int close(int descriptor) noexcept = 0;
};

class IoError : public std::system_error {
public:
    IoError(int error, const char* operation, std::uint64_t offset,
            std::size_t transferred)
        : std::system_error(error, std::generic_category(), operation),
          offset_(offset), transferred_(transferred) {}
    std::uint64_t offset() const noexcept { return offset_; }
    std::size_t transferred() const noexcept { return transferred_; }
private:
    std::uint64_t offset_;
    std::size_t transferred_;
};

inline constexpr std::size_t io_chunk_bytes = 1024 * 1024;

struct IoBytes {
    std::uint64_t read = 0;
    std::uint64_t written = 0;
    bool saturated = false;
    static void add(std::uint64_t& total, std::uint64_t count, bool& overflow) noexcept {
        const auto maximum = std::numeric_limits<std::uint64_t>::max();
        if (count > maximum - total) { total = maximum; overflow = true; }
        else { total += count; }
    }
};

class File {
public:
    File() noexcept = default;
    ~File() noexcept;
    File(const File&) = delete;
    File& operator=(const File&) = delete;
    File(File&& other) noexcept;
    File& operator=(File&& other) noexcept;

    [[nodiscard]] static File open(const std::string& path, OpenMode mode);
    [[nodiscard]] static File adopt(FileOps& operations, int descriptor);
    bool is_open() const noexcept { return descriptor_ >= 0; }
    IoBytes io_bytes() const noexcept { return bytes_; }

    std::size_t read_up_to(std::uint64_t offset, std::span<std::byte> output);
    void read_exact(std::uint64_t offset, std::span<std::byte> output);
    void write_all(std::uint64_t offset, std::span<const std::byte> input);
    void sync(SyncMode mode);
    void truncate_to(std::uint64_t length);
    [[nodiscard]] static File adopt_native(int descriptor);
    void close_checked();

private:
    File(FileOps& operations, int descriptor) noexcept
        : operations_(&operations), descriptor_(descriptor) {}
    void require_open() const;
    int close_once() noexcept;
    FileOps* operations_ = nullptr;
    int descriptor_ = -1;
    IoBytes bytes_{};
};

}  // namespace strata::io
#endif
```

The new member travels with the descriptor. A moved-from file has zero counters;
a closed file keeps its final counters for inspection. Move assignment discards
the destination's old per-file history when it takes a different descriptor.
`IoBytes::add` tests `count > maximum - total` before addition, so the overflow
check cannot itself overflow.

Replace the two move definitions in `src/io/file.cpp` with this contiguous block.
The existing destructor, close functions, native backend, and includes stay as in
Chapters 6 and 10.

```cpp title="src/io/file.cpp — replace both move definitions"
File::File(File&& other) noexcept
    : operations_(std::exchange(other.operations_, nullptr)),
      descriptor_(std::exchange(other.descriptor_, -1)),
      bytes_(std::exchange(other.bytes_, {})) {}

File& File::operator=(File&& other) noexcept {
    if (this != &other) {
        (void)close_once();
        operations_ = std::exchange(other.operations_, nullptr);
        descriptor_ = std::exchange(other.descriptor_, -1);
        bytes_ = std::exchange(other.bytes_, {});
    }
    return *this;
}
```

Replace `read_up_to` and `write_all` with the following complete functions. Notice
that accounting follows backend-contract validation and successful progress, but
precedes the next system call. With a nine-byte request, two successful two-byte
transfers followed by failure leave `written == 4`, not zero and not nine.

```cpp title="src/io/file.cpp — replace std::size_t File::read_up_to"
std::size_t File::read_up_to(std::uint64_t offset, std::span<std::byte> output) {
    require_open();
    (void)checked_file_end(offset, output.size());
    std::size_t done = 0;
    while (done < output.size()) {
        const auto count = std::min(output.size() - done, io_chunk_bytes);
        const Transfer result = operations_->read_at(
            descriptor_, output.subspan(done, count),
            offset + static_cast<std::uint64_t>(done));
        require_transfer(result, count);
        if (result.error == EINTR) {
            continue;
        }
        if (result.error != 0) {
            throw IoError(result.error, "read_at", offset, done);
        }
        if (result.count == 0) {
            return done;
        }
        IoBytes::add(bytes_.read, result.count, bytes_.saturated);
        done += result.count;
    }
    return done;
}
```


```cpp title="src/io/file.cpp — replace void File::write_all"
void File::write_all(std::uint64_t offset, std::span<const std::byte> input) {
    require_open();
    (void)checked_file_end(offset, input.size());
    std::size_t done = 0;
    while (done < input.size()) {
        const auto count = std::min(input.size() - done, io_chunk_bytes);
        const Transfer result = operations_->write_at(
            descriptor_, input.subspan(done, count),
            offset + static_cast<std::uint64_t>(done));
        require_transfer(result, count);
        if (result.error == EINTR) {
            continue;
        }
        if (result.error != 0 || result.count == 0) {
            throw IoError(result.error != 0 ? result.error : EIO,
                          "write_at", offset, done);
        }
        IoBytes::add(bytes_.written, result.count, bytes_.saturated);
        done += result.count;
    }
}
```

`read_exact` still delegates to `read_up_to`; do not count its bytes again. Neither
`sync` nor `truncate_to` transfers a caller buffer, so neither changes these byte
counters. A saturation flag makes a subsequent difference only a lower-quality
observation, which the operation-level meter will explicitly report.

## Boundary reached

The table layout has not changed. We added a nonmutating forecast and small
observability accessors. The manifest now distinguishes a temporal delta from a
key-partitioned base, including an empty one. Part 2 connects those definitions to
source ownership, streaming output, manifest installation, and explicit cleanup.
No deletion belongs in a `TableReader` destructor.
