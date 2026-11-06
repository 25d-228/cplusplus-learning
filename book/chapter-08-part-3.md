# Chapter 8 — Indexed immutable tables

## Part 3 — Verification and native checks

Continue from [Part 2](https://25d-228.github.io/cplusplus-learning/chapter-08-part-2.html).
The table header and implementation are complete. Now create the deterministic
test program and independent Python byte checker, then append the CMake targets.
Keep the earlier chapters' files unchanged. These checks belong to Chapter 8's
single **5.0-hour implementation allocation**, not an additional allocation.

The tests separate four questions: are bytes encoded as specified, does a seek
return the correct version, do corrupt inputs stay within bounded reads and
allocations, and does failure leave a file or cursor in its documented state?
A sanitizer can help with memory mistakes; it cannot answer all four questions.

## Build a deterministic file backend large enough for tables

Create `tests/table.cpp`. The helpers throw on failure in both Debug and Release.
`expect_error` catches only the named exception type; a different failure must
escape rather than accidentally satisfy the test.

```cpp title="tests/table.cpp"
#include "strata/storage/table.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <cstddef>
#include <cstdint>
#include <exception>
#include <filesystem>
#include <iostream>
#include <limits>
#include <optional>
#include <span>
#include <stdexcept>
#include <string>
#include <tuple>
#include <utility>
#include <vector>

namespace {
using namespace strata;
using namespace strata::storage;
namespace fmt = strata::storage::table_format;

void require(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}

template<class Exception, class Function>
void expect_error(const char* message, Function action) {
    try { action(); }
    catch (const Exception&) { return; }
    throw std::runtime_error(message);
}
```

The fixture owns a preallocated 512 KiB byte array through a vector. This test-only
allocation is not engine storage and is never used by the native implementation.
`install` copies an image only while no file is open. `image` deliberately copies
the used prefix so corruption tests can own and edit it independently.

```cpp title="tests/table.cpp"
class TableFileOps final : public io::FileOps {
public:
    std::vector<std::byte> bytes = std::vector<std::byte>(512 * 1024);
    std::size_t used = 0;
    std::size_t accepted = 0;
    std::size_t read_calls = 0;
    std::size_t close_calls = 0;
    std::size_t transfer_limit = 4096;
    std::optional<std::size_t> fail_after;
    std::optional<std::size_t> fail_read_call;
    bool interrupt = false;
    bool live = false;

    io::File open() {
        require(!live, "fixture already open");
        live = true;
        return io::File::adopt(*this, 8);
    }
    void install(std::span<const std::byte> image) {
        require(!live && image.size() <= bytes.size(), "invalid fixture install");
        std::copy(image.begin(), image.end(), bytes.begin());
        used = image.size();
    }
    std::vector<std::byte> image() const {
        return {bytes.begin(), bytes.begin() + static_cast<std::ptrdiff_t>(used)};
    }
```

The virtual operations below allocate nothing. They can therefore implement the
Chapter 6 `noexcept` backend interface without hiding allocation failures inside
an exception-to-error adapter. Every offset is compared with a size before it is
converted to `size_t` and used for indexing.

```cpp title="tests/table.cpp"
    io::Transfer read_at(int fd, std::span<std::byte> out,
                         std::uint64_t offset) noexcept override {
        ++read_calls;
        if (!live || fd != 8) { return {0, EBADF}; }
        if (fail_read_call && read_calls == *fail_read_call) { return {0, EIO}; }
        if (offset >= used) { return {0, 0}; }
        const auto at = static_cast<std::size_t>(offset);
        const auto count = std::min({out.size(), used - at, transfer_limit});
        std::copy_n(bytes.begin() + static_cast<std::ptrdiff_t>(at), count, out.begin());
        return {count, 0};
    }
    io::Transfer write_at(int fd, std::span<const std::byte> in,
                          std::uint64_t offset) noexcept override {
        if (!live || fd != 8) { return {0, EBADF}; }
        if (std::exchange(interrupt, false)) { return {0, EINTR}; }
        if (fail_after && accepted >= *fail_after) { return {0, EIO}; }
        if (offset >= bytes.size()) { return {0, ENOSPC}; }
        const auto at = static_cast<std::size_t>(offset);
        auto count = std::min({in.size(), bytes.size() - at, transfer_limit});
        if (fail_after) { count = std::min(count, *fail_after - accepted); }
        std::copy_n(in.begin(), count, bytes.begin() + static_cast<std::ptrdiff_t>(at));
        accepted += count;
        used = std::max(used, at + count);
        return {count, 0};
    }
    int sync(int fd, io::SyncMode mode) noexcept override {
        if (!live || fd != 8) { return EBADF; }
        return mode == io::SyncMode::file ? 0 : ENOTSUP;
    }
    int close(int fd) noexcept override {
        ++close_calls;
        if (!live || fd != 8) { return EBADF; }
        live = false;
        return 0;
    }
};
```

`accepted` counts successfully written bytes; `fail_after` makes a later call
return `EIO`. A successful partial call does not also return an error, matching
the earlier `Transfer` contract. `transfer_limit=3` forces the real `File` loops
to assemble frames across many calls. One injected `EINTR` checks retry without
counting bytes that were not transferred. The fixture must be declared before
its owning `File` or reader so it outlives their destructor calls.

## Construct the eight-version trace explicitly

```cpp title="tests/table.cpp"
void apply(MemTable& table, std::uint64_t sequence,
           std::span<const Mutation> changes) {
    auto pending = table.prepare(sequence, changes);
    pending.commit();
}

void populate(MemTable& table) {
    const std::array first{Mutation::put(Event{10, 100, 1, 10}),
        Mutation::put(Event{20, 100, 1, 20}), Mutation::put(Account{100, 1})};
    const std::array second{Mutation::put(Event{10, 100, 2, 22}),
        Mutation::put(Event{30, 100, 2, 30})};
    const std::array third{Mutation::erase({Relation::events, 10}),
        Mutation::put(Event{20, 100, 3, 23}), Mutation::put(Account{100, 7})};
    apply(table, 1, first);
    apply(table, 2, second);
    apply(table, 3, third);
    table.freeze();
}

TableOptions tiny_options() { return {64, 32, 32, 64 * 1024}; }

std::vector<std::byte> small_image() {
    MemTable table{64 * 1024};
    populate(table);
    TableFileOps ops;
    {
        auto file = ops.open();
        const auto info = write_table(file, table, tiny_options());
        require(info.entries == 8 && info.blocks == 8 && info.file_bytes == ops.used,
                "small fixture geometry");
    }
    require(!ops.live && ops.close_calls == 1, "writer file cleanup");
    return ops.image();
}
```

The three input batches create this sorted history:

| User key | Versions, newest first |
|---|---|
| events 10 | erase at 3; put amount 22 at 2; put amount 10 at 1 |
| events 20 | put amount 23 at 3; put amount 20 at 1 |
| events 30 | put amount 30 at 2 |
| accounts 100 | put region 7 at 3; put region 1 at 1 |

There are eight *versions*, four distinct user keys, and three latest live rows
because events 10 is deleted. The table builder stores all eight, including the
tombstone. At a 64-byte data payload limit each occupies a separate block. We
will also test a larger block size because one-row blocks can conceal a wrong
sparse-search implementation.

## Freeze the bit conventions, then measure false positives

```cpp title="tests/table.cpp"
void check_hashes() {
    const std::array<std::byte, 9> digits{std::byte{'1'}, std::byte{'2'}, std::byte{'3'},
        std::byte{'4'}, std::byte{'5'}, std::byte{'6'}, std::byte{'7'}, std::byte{'8'}, std::byte{'9'}};
    require(fmt::crc32({}) == 0 && fmt::crc32(digits) == 0xcbf43926U, "CRC known answer");
    require(fmt::crc32(std::span{digits}.subspan(4), fmt::crc32(std::span{digits}.first(4))) ==
            0xcbf43926U, "incremental CRC");
    std::array<std::byte, 8> bits{};
    fmt::bloom_add(bits, {Relation::events, 42});
    const std::array<std::byte, 8> expected{std::byte{0x20}, std::byte{0x80}, std::byte{0x40},
        std::byte{0}, std::byte{1}, std::byte{4}, std::byte{0}, std::byte{8}};
    require(bits == expected, "Bloom format known answer");
    std::array<std::byte, 7> invalid{};
    expect_error<std::invalid_argument>("accepted seven-byte filter", [&] {
        fmt::bloom_add(invalid, {Relation::events, 1});
    });
    std::array<std::byte, 128> filter{};
    for (std::uint64_t i = 0; i < 100; ++i) { fmt::bloom_add(filter, {Relation::events, i}); }
    for (std::uint64_t i = 0; i < 100; ++i) {
        require(fmt::bloom_may_contain(filter, {Relation::events, i}), "Bloom false negative");
    }
    std::size_t positives = 0;
    for (std::uint64_t i = 1000; i < 11000; ++i) {
        if (fmt::bloom_may_contain(filter, {Relation::events, i})) { ++positives; }
    }
    std::cout << "Bloom: " << positives << "/10000 absent probes were positive\n";
}
```

The checksum test has three independent expectations: empty input, a standard
fixed message, and incremental versus concatenated processing. The Bloom test
pins its byte-level format with events key 42, checks every one of 100 inserted
keys, and prints the result of 10,000 known-absent probes. The exact printed
positive count is not a statistical acceptance condition. A filter returning
true for an absent key is permitted; returning false for an inserted key is not.

For this deterministic implementation the observed sample is 97 positives out of
10,000, or 0.97%. That is an observation about this fixture, not a hardware speed
measurement, universal false-positive rate, or guarantee matching the idealized
0.76% calculation. Enlarging the filter can improve selectivity without changing
answers; clearing bits after inserting a key can break correctness.

## Test visibility at block boundaries and retain an owned answer

```cpp title="tests/table.cpp"
void check_boundaries(const std::vector<std::byte>& image) {
    TableFileOps ops;
    ops.install(image);
    std::optional<OwnedEntry> retained;
    {
        TableReader reader{ops.open(), image.size()};
        require(reader.block_reads() == 0, "open read data blocks");
        reader.verify_all();
        require(reader.block_reads() == 8, "stream did not read exactly eight blocks");
        for (const std::uint64_t visible : {3ULL, 2ULL, 1ULL}) {
            const auto before = reader.block_reads();
            auto found = reader.find({Relation::events, 10}, visible);
            require(found && found->key.sequence == visible, "cross-block visible version");
            require(reader.block_reads() == before + 1, "point lookup read multiple blocks");
            require(found->kind == (visible == 3 ? EntryKind::erase : EntryKind::put),
                    "tombstone was mistaken for absence");
        }
        require(!reader.find({Relation::events, 10}, 0), "found a version before creation");
        require(!reader.find({Relation::events, 0}), "before-first key exists");
        require(!reader.find({Relation::accounts, std::numeric_limits<std::uint64_t>::max()}),
                "after-last key exists");
        auto first = reader.lower_bound({{Relation::events, 0}, 0});
        require(first.valid() && first.entry().key.user.primary == 10, "before-first seek");
        auto gap = reader.lower_bound({{Relation::events, 11}, 99});
        require(gap.valid() && gap.entry().key.user.primary == 20, "between-key seek");
        auto end = reader.lower_bound({{Relation::accounts, 101}, 99});
        require(!end.valid(), "after-last seek");
        end.next();
        expect_error<std::out_of_range>("read an end cursor", [&] { (void)end.entry(); });
        std::uint64_t absent = 1000;
        while (reader.may_contain({Relation::events, absent}) && absent < 2000) { ++absent; }
        require(absent < 2000, "fixture produced no filter negative");
        const auto before = reader.block_reads();
        require(!reader.find({Relation::events, absent}) && reader.block_reads() == before,
                "negative filter performed a data-block read");
        retained = reader.find({Relation::events, 10}, 2);
        auto left = reader.begin();
        auto right = reader.begin();
        left.next();
        require(right.entry().key.sequence == 3 && left.entry().key.sequence == 2,
                "cursors unexpectedly shared a buffer");
        ops.fail_read_call = ops.read_calls + 1;
        expect_error<io::IoError>("missing next-block I/O failure", [&] { left.next(); });
        require(!left.valid(), "failed cursor retained an entry");
        left.next();
    }
    require(retained && decode_event(retained->view().value).amount_cents == 22,
            "point result did not own its bytes");
    require(!ops.live && ops.close_calls == 1, "reader file cleanup");
```

The three visibility checks for events 10 must each read exactly one data frame.
The first returns an erase, and the other two return different put versions.
Visibility zero finds no qualifying version; missing keys before, between, and
after the stored range exercise different search boundaries. The `lower_bound`
gap check bypasses the Bloom filter entirely, so a negative filter cannot conceal
a broken directory search.

The two simultaneous cursors prove that one cursor's advance does not overwrite
the other's block. The injected next-read failure makes the advancing cursor
terminal. The owning result `retained` survives both the cursor used by `find`
and the reader's entire scope; decoding it afterward would expose a returned
borrow into a destroyed block buffer.

Continue the same `check_boundaries` function with the empty-table path:

```cpp title="tests/table.cpp"
    TableFileOps empty_ops;
    TableInfo info{};
    {
        auto file = empty_ops.open();
        TableBuilder builder{file, tiny_options()};
        info = builder.finish();
        expect_error<std::logic_error>("finished twice", [&] { (void)builder.finish(); });
    }
    TableReader empty{empty_ops.open(), info.file_bytes};
    require(info.blocks == 0 && info.entries == 0 && info.file_bytes == 120,
            "empty table geometry");
    empty.verify_all();
    require(!empty.begin().valid() && !empty.find({Relation::events, 1}), "empty read");
    std::cout << "boundaries: old versions, tombstones, gaps, empty table, owned result OK\n";
}
```

The empty fixture finishes at 120 bytes with no data frames. Finishing twice is
an error; observing end and advancing end are normal operations. `verify_all`
works without special caller handling for an empty table.

## Compare with a structurally different reference

These test helpers copy complete entry values and express ordering with a tuple,
not with the production `InternalLess`:

```cpp title="tests/table.cpp"
OwnedEntry copy_entry(EntryView entry) {
    OwnedEntry result;
    result.key = entry.key;
    result.kind = entry.kind;
    result.length = entry.value.size();
    std::copy(entry.value.begin(), entry.value.end(), result.bytes.begin());
    return result;
}

auto order_key(const OwnedEntry& entry) {
    return std::tuple{static_cast<unsigned>(entry.key.user.relation), entry.key.user.primary,
        std::numeric_limits<std::uint64_t>::max() - entry.key.sequence};
}

bool same(EntryView a, EntryView b) {
    return a.key == b.key && a.kind == b.kind &&
           std::equal(a.value.begin(), a.value.end(), b.value.begin(), b.value.end());
}
```

For an unsigned sequence $s$, `UINT64_MAX - s` reverses its order without a signed
conversion. Tuple comparison then gives relation ascending, primary ascending,
and sequence descending. The full-range `std::equal` overload compares both
length and bytes; equal prefixes with different lengths do not count as equal rows.

The next fixture creates 200 batches and 600 versions. One event key family repeats,
one creates distinct keys, and the account keys repeat independently. Some updates
are tombstones. This makes the test sensitive to relation tags, long histories,
keys inside blocks, and the numeric 255/256 boundary.

```cpp title="tests/table.cpp"
void check_oracle() {
    MemTable table{1024 * 1024};
    std::vector<OwnedEntry> expected;
    for (std::uint64_t i = 0; i < 200; ++i) {
        const std::array changes{
            i % 7 == 0 ? Mutation::erase({Relation::events, i % 29})
                       : Mutation::put(Event{i % 29, i % 17, 1, i * 3}),
            Mutation::put(Event{100 + i, i % 17, 2, i * 5}),
            i % 11 == 0 ? Mutation::erase({Relation::accounts, i % 17})
                        : Mutation::put(Account{i % 17, static_cast<std::uint32_t>(i)})};
        for (const auto& change : changes) {
            expected.push_back(copy_entry({{change.key, i + 1}, change.kind,
                std::span{change.bytes}.first(change.length)}));
        }
        apply(table, i + 1, changes);
    }
    table.freeze();
    std::sort(expected.begin(), expected.end(), [](const OwnedEntry& a, const OwnedEntry& b) {
        return order_key(a) < order_key(b);
    });
    TableFileOps ops;
    TableInfo info{};
    {
        auto file = ops.open();
        info = write_table(file, table, {256, 256, 512, 512 * 1024});
    }
```

Continue `check_oracle` after writing the table. The point-reference path scans all
expected versions and chooses the largest sequence not exceeding visibility.
That is intentionally different from the production directory and lower-bound
algorithm, so they are less likely to share the same indexing defect.

```cpp title="tests/table.cpp"
    TableReader reader{ops.open(), info.file_bytes};
    auto cursor = reader.begin();
    for (const auto& row : expected) {
        require(cursor.valid() && same(cursor.entry(), row.view()), "full stream oracle mismatch");
        require(reader.may_contain(row.key.user), "stored key rejected by Bloom");
        cursor.next();
    }
    require(!cursor.valid() && expected.size() == info.entries, "stream count oracle");
    std::size_t queries = 0;
    const std::array<std::uint64_t, 6> snapshots{0, 1, 17, 60, 200,
        std::numeric_limits<std::uint64_t>::max()};
    for (const auto relation : {Relation::events, Relation::accounts}) {
        for (std::uint64_t primary = 0; primary < 320; ++primary) {
            const Key key{relation, primary};
            for (const auto visible : snapshots) {
                const OwnedEntry* best = nullptr;
                for (const auto& row : expected) {
                    if (row.key.user == key && row.key.sequence <= visible &&
                        (!best || best->key.sequence < row.key.sequence)) { best = &row; }
                }
                const auto actual = reader.find(key, visible);
                require(actual.has_value() == (best != nullptr), "point presence oracle");
                if (best) { require(same(actual->view(), best->view()), "point value oracle"); }
                ++queries;
            }
        }
    }
    reader.verify_all();
    std::cout << "buffers: metadata=" << reader.metadata_capacity()
              << " cursor=" << cursor.buffer_capacity() << " bytes\n";
    std::cout << "oracle: " << expected.size() << " versions, " << info.blocks << " blocks, "
              << queries << " point queries OK\n";
}
```

There are $2\times320\times6=3840$ point queries: two relations, 320 primary keys,
and six visibility values. The complete ordered stream must also match all 600
reference entries. A typical run here uses 122 blocks, a metadata vector capacity
of 6,148 bytes, and a cursor frame capacity of 268 bytes. Those capacities are
observations of the tested standard library, not mandated vector growth behavior.
The code does not time these runs or claim performance on the M4.

## Corrupt semantics even when the CRC still matches

A checksum-only test can be satisfied by a parser that does no semantic validation.
We need both damaged CRCs and deliberately repaired CRCs around invalid content.
The repair helper is test code, not a repair policy for a database file.

```cpp title="tests/table.cpp"
std::size_t metadata_offset(const std::vector<std::byte>& image) {
    return static_cast<std::size_t>(get_le<8>(std::span{image}.subspan(image.size() - 32, 8)));
}

void repair_crc(std::vector<std::byte>& image, std::size_t start, std::size_t length) {
    auto bytes = std::span{image}.subspan(start, length);
    put_le<4>(bytes.last(4), fmt::crc32(bytes.first(length - 4)));
}

void reject_image(const std::vector<std::byte>& image) {
    TableFileOps ops;
    ops.install(image);
    expect_error<FormatError>("accepted malformed table", [&] {
        TableReader reader{ops.open(), image.size()};
        reader.verify_all();
    });
    require(!ops.live && ops.close_calls == 1, "malformed input leaked file");
}
```

`reject_image` requires the reader to close its adopted file on every exception.
It calls `verify_all` because a corruption in an unvisited data block need not be
found by opening alone. The native reader path remains unchanged; only its backend
supplies bytes from memory.

```cpp title="tests/table.cpp"
void check_corruption(const std::vector<std::byte>& original) {
    const auto meta = metadata_offset(original);
    const auto footer = original.size() - fmt::footer_bytes;
    const auto meta_length = footer - meta;
    const auto first_frame = static_cast<std::size_t>(get_le<4>(std::span{original}.subspan(24, 4))) + 12;
    auto change = [&](std::size_t at, unsigned value, bool repair, std::size_t start, std::size_t len) {
        auto image = original;
        image.at(at) = static_cast<std::byte>(value);
        if (repair) { repair_crc(image, start, len); }
        reject_image(image);
    };
    change(8, 2, false, 0, 24);                  // Header checksum.
    change(8, 2, true, 0, 24);                   // Unsupported version with valid checksum.
    auto huge_block = original;
    put_le<4>(std::span{huge_block}.subspan(12, 4), 0xffffffffU);
    repair_crc(huge_block, 0, 24);
    reject_image(huge_block);
    change(footer, 0, false, footer, 40);         // Footer magic.
    change(meta + 8, 0xff, true, meta, meta_length); // Directory count disagrees with extent.
    change(meta + 12, 7, true, meta, meta_length);   // Non-power-of-two filter bytes.
    change(meta + 16, 7, true, meta, meta_length);   // Unsupported probe count.
    change(meta + 20 + 34, 25, true, meta, meta_length); // Gap before first data frame.
    change(meta + 20 + 9, 4, true, meta, meta_length);  // Directory endpoint disagrees with data.
    change(24 + 8 + 4 + 17, 3, true, 24, first_frame);  // Unknown entry kind.
    change(24 + 8 + 4 + 18, 31, true, 24, first_frame); // Oversize value length.
    change(24 + 8, 0, true, 24, first_frame);           // Empty stored block.
    change(24, 0xff, true, 24, first_frame);           // Frame length disagreement.
    change(24 + 4, 2, true, 24, first_frame);          // Wrong frame kind.
    change(24 + 8 + 4 + 1, 11, false, 24, first_frame); // Data checksum.
    change(24 + 8 + 4 + 1, 11, true, 24, first_frame);  // Wrong directory endpoint.
```

The lambda captures the small fixture by reference and creates a fresh owned copy
for each experiment. It changes one field, optionally recomputes the containing
CRC, and demands rejection. This keeps experiments independent: an early checksum
failure cannot mask a later malformed-length case left over from another test.
The large frame length is read only into the bounded buffer; a wrong tag cannot
be accepted merely because the checksum was recomputed.

Continue `check_corruption` with excessive metadata, a wrong total count, and a
logically invalid filter:

```cpp title="tests/table.cpp"
    auto huge_meta = original;
    put_le<4>(std::span{huge_meta}.subspan(footer + 16, 4), 0xffffffffU);
    repair_crc(huge_meta, footer, 40);
    reject_image(huge_meta);
    auto count = original;
    put_le<8>(std::span{count}.subspan(footer + 20, 8), 9);
    repair_crc(count, footer, 40);
    reject_image(count);
    auto missing = original;
    const auto filter_start = meta + 8 + 12 + 8 * fmt::directory_bytes;
    std::fill(missing.begin() + static_cast<std::ptrdiff_t>(filter_start),
              missing.begin() + static_cast<std::ptrdiff_t>(footer - 4), std::byte{0});
    repair_crc(missing, meta, meta_length);
    TableFileOps ops;
    ops.install(missing);
    {
        TableReader reader{ops.open(), missing.size()};
        require(!reader.find({Relation::events, 10}), "forged filter did not skip data");
        expect_error<FormatError>("full verification missed forged Bloom", [&] { reader.verify_all(); });
    }
```

The zeroed, recomputed filter shows the lazy-verification boundary explicitly:
`find` skips the data and returns absent, but `verify_all` detects a stored key
that the filter rejects. This is a deliberately forged logical inconsistency,
not a valid table that legitimately has a Bloom false negative.

Now set *extra* bits instead. An all-one filter is semantically safe, though
useless for pruning. Its positive answer must still be followed by a real search.
Then try every incomplete prefix of the valid 970-byte fixture.

```cpp title="tests/table.cpp"
    auto saturated = original;
    std::fill(saturated.begin() + static_cast<std::ptrdiff_t>(filter_start),
              saturated.begin() + static_cast<std::ptrdiff_t>(footer - 4), std::byte{0xff});
    repair_crc(saturated, meta, meta_length);
    TableFileOps positive;
    positive.install(saturated);
    {
        TableReader reader{positive.open(), saturated.size()};
        require(reader.may_contain({Relation::events, 11}), "saturated filter was negative");
        require(!reader.find({Relation::events, 11}), "filter positive became a stored row");
        reader.verify_all();  // Extra set bits are safe, just less selective.
    }
    for (std::size_t length = 0; length < original.size(); ++length) {
        reject_image({original.begin(), original.begin() + static_cast<std::ptrdiff_t>(length)});
    }
    auto extra = original;
    extra.push_back(std::byte{0});
    reject_image(extra);
    TableFileOps trailing;
    trailing.install(extra);
    expect_error<FormatError>("caller length concealed trailing bytes", [&] {
        TableReader reader{trailing.open(), original.size()};
    });
    std::cout << "corruption: checksums, repaired-checksum structural defects, "
              << original.size() << " truncated prefixes OK\n";
}
```

All 970 proper prefixes are incomplete tables and must be rejected. Appending
one trailing byte must also fail, even when the caller passes the original length.
Unlike a future log-recovery rule, this table reader does **not** ignore a torn
last block and return the preceding rows as a complete file. A standalone table
requires its complete footer and consistent extents.

## Failure does not mean that the output stayed empty

Start the write-failure tests:

```cpp title="tests/table.cpp"
void check_builder_failures(const std::vector<std::byte>& original) {
    MemTable table{64 * 1024};
    populate(table);
    TableFileOps short_ops;
    short_ops.transfer_limit = 3;
    short_ops.interrupt = true;
    {
        auto file = short_ops.open();
        (void)write_table(file, table, tiny_options());
    }
    require(short_ops.image() == original, "short transfers changed table bytes");
    {
        TableReader reader{short_ops.open(), original.size()};
        reader.verify_all();
    }
    TableFileOps failed;
    failed.transfer_limit = 3;
    failed.fail_after = 34;
    {
        auto file = failed.open();
        TableBuilder builder{file, tiny_options()};
        auto cursor = table.begin();
        builder.append(cursor.entry());
        cursor.next();
        expect_error<io::IoError>("missing partial-write failure", [&] { builder.append(cursor.entry()); });
        expect_error<std::logic_error>("failed builder resumed", [&] { (void)builder.finish(); });
        require(failed.used == 34, "partial effects were not observable");
    }
    reject_image(failed.image());
    TableFileOps footer_failure;
    footer_failure.fail_after = original.size() - 7;
    {
        auto file = footer_failure.open();
        expect_error<io::IoError>("missing footer failure", [&] {
            (void)write_table(file, table, tiny_options());
        });
    }
    reject_image(footer_failure.image());
```

With failure after 34 accepted bytes, the 24-byte table header is complete, as
is the first frame's eight-byte header, and two payload bytes have arrived.
That is an observable partial effect. The builder's failure state prevents
`finish` from reporting success; destruction frees memory but does not erase
those file bytes. The last-footer failure similarly produces no usable result.
Short transfers without an actual error must instead produce a byte-identical
970-byte file.

Continue with duplicate keys, resource caps, and existing-file protection:

```cpp title="tests/table.cpp"
    TableFileOps duplicate;
    {
        auto file = duplicate.open();
        TableBuilder builder{file, tiny_options()};
        const auto entry = table.begin().entry();
        builder.append(entry);
        expect_error<std::invalid_argument>("duplicate internal key accepted", [&] { builder.append(entry); });
        expect_error<std::logic_error>("invalid-input builder resumed", [&] { builder.append(entry); });
    }
    for (const auto options : {TableOptions{64, 1, 32, 65536}, TableOptions{64, 32, 8, 96}}) {
        TableFileOps capped;
        auto file = capped.open();
        expect_error<std::length_error>("missing table cap rejection", [&] {
            (void)write_table(file, table, options);
        });
    }
    TableFileOps nonempty;
    nonempty.install(original);
    {
        auto file = nonempty.open();
        expect_error<std::invalid_argument>("overwrote nonempty file", [&] {
            TableBuilder builder{file, tiny_options()};
        });
    }
    require(nonempty.image() == original, "rejected existing file changed");
    std::cout << "failures: short I/O, failed builders, footer failure, caps, cleanup OK\n";
}
```

The block-cap and file-cap configurations are valid but too small for this
nonempty fixture; they must throw `length_error`, not silently drop rows or grow
without limit. A nonempty file is rejected before its bytes change. The builder
is intended for newly created outputs, not an in-place replacement protocol.

## Exercise an actual file and clean up only what this call created

`native_fixture` uses `File::open(create_new)` rather than trusting the emptiness
probe alone. The `created` flag is set only after exclusive creation succeeds.
On failure, local file/reader owners unwind before the outer handler attempts
to remove the temporary path. A collision with an existing file must **not**
remove it.

The [nonthrowing `std::filesystem::remove` overload](https://eel.is/c++draft/fs.op.remove)
accepts an `error_code` output. We report a cleanup failure separately and rethrow
the original problem. This helper assumes its caller supplied an exclusively
owned temporary path; it is not a safe deletion protocol against another actor
replacing that pathname concurrently. The Python runner below supplies such a
private temporary directory.

```cpp title="tests/table.cpp"
void native_fixture(const std::string& path) {
    bool created = false;
    try {
        MemTable table{64 * 1024};
        populate(table);
        auto file = io::File::open(path, io::OpenMode::create_new);
        created = true;
        const auto info = write_table(file, table, tiny_options());
        file.sync(io::SyncMode::file);
        file.close_checked();
        TableReader reader{io::File::open(path, io::OpenMode::read_only), info.file_bytes};
        reader.verify_all();
        std::cout << "native table: bytes=" << info.file_bytes << " blocks=" << info.blocks
                  << " entries=" << info.entries << '\n';
    } catch (...) {
        if (created) {
            std::error_code error;
            std::filesystem::remove(path, error);
            if (error) { std::cerr << "fixture cleanup: " << error.message() << '\n'; }
        }
        throw;
    }
}
}  // namespace
```

The explicit file synchronization and checked close are observed operations, not
a claim about a power-cut experiment. A successful table is retained for the
Python decoder. A failure after creation removes only this fixture output when
cleanup succeeds. No manifest, rename sequence, directory synchronization, or
log installation is being performed.

Finish `tests/table.cpp` with its entry point:

```cpp title="tests/table.cpp"
int main(int argc, char** argv) {
    try {
        if (argc == 3 && std::string{argv[1]} == "--fixture") {
            native_fixture(argv[2]);
            return 0;
        }
        if (argc != 1) { throw std::invalid_argument("usage: strata-table-tests [--fixture PATH]"); }
        check_hashes();
        const auto image = small_image();
        check_boundaries(image);
        check_oracle();
        check_corruption(image);
        check_builder_failures(image);
        std::cout << "table checks passed\n";
        return 0;
    } catch (const std::exception& error) {
        std::cerr << "FAIL " << error.what() << '\n';
        return 1;
    }
}
```

The normal test sequence uses the deterministic backend. `--fixture PATH` selects
the real positional-I/O implementation. Both modes convert a standard exception
into a nonzero exit code and a visible diagnostic, including under `NDEBUG`.

## Inspect native bytes with a separate implementation

Create `scripts/check_table_fixture.py`. It uses Python only as an independent
fixture checker: the table construction, indexing, searching, checksums, and
filter implementation under test remain C++. Python's standard
[`zlib.crc32`](https://docs.python.org/3/library/zlib.html#zlib.crc32)
provides a separate CRC implementation; `struct` decodes explicit little-endian
fields without calling our C++ helpers. The collection type annotations require
Python 3.9 or newer; a compatible maintained Python 3 installation is sufficient.

```python title="scripts/check_table_fixture.py"
"""Check Chapter 8's small native table using independent Python byte parsing."""
import pathlib
import struct
import subprocess
import sys
import tempfile
import zlib


def need(condition: bool, message: str) -> None:
    if not condition:
        raise ValueError(message)


def mix(value: int) -> int:
    mask = (1 << 64) - 1
    value = ((value ^ (value >> 30)) * 0xBF58476D1CE4E5B9) & mask
    value = ((value ^ (value >> 27)) * 0x94D049BB133111EB) & mask
    return value ^ (value >> 31)


def make_filter(keys: set[tuple[int, int]], size: int) -> bytes:
    bits = bytearray(size)
    for relation, primary in keys:
        value = 14695981039346656037
        for byte in struct.pack("<BQ", relation, primary):
            value = ((value ^ byte) * 1099511628211) & ((1 << 64) - 1)
        start = mix(value)
        step = mix(value ^ 0x9E3779B97F4A7C15) | 1
        for probe in range(6):
            position = (start + probe * step) % (size * 8)
            bits[position // 8] |= 1 << (position % 8)
    return bytes(bits)
```

Python integers do not naturally wrap at 64 bits. Mask after each multiplication
in `mix` and the FNV loop. For positions we can use unbounded `start + probe * step`
and then modulo the power-of-two bit count; Part 1 showed why this yields the
same result as the C++ unsigned wrap. Omitting masks from the mixer itself would
be a different hash, not “higher precision.”

```python title="scripts/check_table_fixture.py"
def frame(data: bytes, offset: int, size: int, tag: int) -> bytes:
    need(size >= 12 and 0 <= offset <= len(data) - size, "frame extent")
    encoded = data[offset:offset + size]
    length, actual_tag = struct.unpack_from("<II", encoded)
    need(length == size - 12 and actual_tag == tag, "frame header")
    stored_crc, = struct.unpack_from("<I", encoded, size - 4)
    need(stored_crc == zlib.crc32(encoded[:-4]), "frame CRC")
    return encoded[8:-4]
```

This checker is intentionally for the fixed tiny fixture, not a second general
storage engine. It first requires exactly 970 bytes and the expected header and
metadata choices. It independently checks every extent and CRC before decoding
its one-entry data blocks.

```python title="scripts/check_table_fixture.py"
def verify(path: pathlib.Path) -> None:
    data = path.read_bytes()
    need(len(data) == 970, "small fixture must contain exactly 970 bytes")
    magic, version, block_limit, reserved, crc = struct.unpack_from("<8sIIII", data)
    need((magic, version, block_limit, reserved) == (b"STRATABL", 1, 64, 0), "header")
    need(crc == zlib.crc32(data[:20]), "header CRC")
    magic, meta_at, meta_size, count, total, crc = struct.unpack_from("<8sQIQQI", data, len(data) - 40)
    need(magic == b"STRATEND" and total == len(data) and count == 8, "footer")
    need(crc == zlib.crc32(data[-40:-4]), "footer CRC")
    need(meta_at + meta_size + 40 == len(data), "metadata extent")
    meta = frame(data, meta_at, meta_size, 2)
    blocks, filter_bytes, probes = struct.unpack_from("<III", meta)
    need((blocks, filter_bytes, probes) == (8, 32, 6), "metadata counts")
    need(len(meta) == 12 + 46 * blocks + filter_bytes, "metadata length")
    rows = []
    next_offset = 24
    for index in range(blocks):
        at = 12 + 46 * index
        first = struct.unpack_from("<BQQ", meta, at)
        last = struct.unpack_from("<BQQ", meta, at + 17)
        offset, size = struct.unpack_from("<QI", meta, at + 34)
        need(offset == next_offset and offset + size <= meta_at, "block continuity")
        payload = frame(data, offset, size, 1)
        entries, = struct.unpack_from("<I", payload)
        need(entries == 1, "tiny fixture block must contain one entry")
        relation, primary, sequence, kind, length = struct.unpack_from("<BQQBI", payload, 4)
        need(len(payload) == 4 + 22 + length, "entry length")
        need(first == last == (relation, primary, sequence), "directory endpoints")
        rows.append((relation, primary, sequence, kind, payload[26:]))
        next_offset = offset + size
    need(next_offset == meta_at, "unindexed bytes")
```

Continue the same `verify` function. Construct the expected row encodings directly
with `struct.pack` and compare the complete ordered history, not just a few
selected fields. Rebuild the filter from the set of expected user keys to check
its exact bytes as well.

```python title="scripts/check_table_fixture.py"
    event = lambda key, day, amount: struct.pack("<BBQQIQ", 1, 1, key, 100, day, amount)
    account = lambda region: struct.pack("<BBQI", 1, 2, 100, region)
    expected = [
        (1, 10, 3, 2, b""),
        (1, 10, 2, 1, event(10, 2, 22)),
        (1, 10, 1, 1, event(10, 1, 10)),
        (1, 20, 3, 1, event(20, 3, 23)),
        (1, 20, 1, 1, event(20, 1, 20)),
        (1, 30, 2, 1, event(30, 2, 30)),
        (2, 100, 3, 1, account(7)),
        (2, 100, 1, 1, account(1)),
    ]
    need(rows == expected, "entry order, versions, or values")
    need(meta[-filter_bytes:] == make_filter({(r[0], r[1]) for r in expected}, filter_bytes),
         "Bloom bytes disagree with independent encoding")
    print(f"oracle: {len(data)} bytes, {blocks} blocks, {len(rows)} versions; CRCs and Bloom bytes OK")
```

The script can check an existing fixture path or invoke the executable itself.
The latter mode gives CTest a private output directory, then attempts exclusive
creation again and confirms that the original bytes survived the rejection.

```python title="scripts/check_table_fixture.py"
def main() -> None:
    if len(sys.argv) == 2:
        verify(pathlib.Path(sys.argv[1]))
        return
    if len(sys.argv) == 3 and sys.argv[1] == "--program":
        executable = pathlib.Path(sys.argv[2]).resolve()
        with tempfile.TemporaryDirectory(prefix="strata-table-") as directory:
            path = pathlib.Path(directory) / "tiny.sst"
            subprocess.run([str(executable), "--fixture", str(path)], check=True)
            verify(path)
            before = path.read_bytes()
            duplicate = subprocess.run([str(executable), "--fixture", str(path)],
                                       capture_output=True, text=True, check=False)
            need(duplicate.returncode == 1 and path.read_bytes() == before,
                 "exclusive create did not preserve existing table")
            print("oracle: existing fixture preserved on create collision")
        return
    raise ValueError("usage: check_table_fixture.py FILE | --program EXECUTABLE")


if __name__ == "__main__":
    try:
        main()
    except (OSError, ValueError, struct.error, subprocess.CalledProcessError) as error:
        print(f"table fixture check failed: {error}", file=sys.stderr)
        sys.exit(1)
```

A codec bug can sometimes make a writer and its matching reader agree with each
other. Comparing their actual output with separately constructed bytes makes that
symmetry harder to hide. This is why the fixture checks its eight complete rows,
not only the file length or one returned amount.

## Build without dropping the earlier exercises

Append this block to the existing root `CMakeLists.txt`. `strata-core` and the
Python interpreter discovered in Chapter 6 already exist; do not redeclare them.

```cmake title="CMakeLists.txt"
target_sources(strata-core PRIVATE src/storage/table.cpp)
add_executable(strata-table-tests tests/table.cpp)
target_link_libraries(strata-table-tests PRIVATE strata-core)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(strata-table-tests PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion)
endif()
add_test(NAME table COMMAND strata-table-tests)
add_test(NAME table-native
    COMMAND "${Python3_EXECUTABLE}"
            "${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_table_fixture.py"
            --program $<TARGET_FILE:strata-table-tests>)
```

Use the compatible native compiler selected in Chapter 1. The commands below
assume the existing `build/debug` and `build/release` configurations; use fresh
directories rather than silently changing the compiler of an existing cache.

```sh title="Terminal — cumulative Debug checks"
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug --parallel
ctest --test-dir build/debug --output-on-failure
./build/debug/strata-table-tests
```

```sh title="Terminal — cumulative Release checks"
cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/release --parallel
ctest --test-dir build/release --output-on-failure
```

There are now **eleven** CTest entries. Representative normal output is:

```text title="Observed example output — table tests"
Bloom: 97/10000 absent probes were positive
boundaries: old versions, tombstones, gaps, empty table, owned result OK
buffers: metadata=6148 cursor=268 bytes
oracle: 600 versions, 122 blocks, 3840 point queries OK
corruption: checksums, repaired-checksum structural defects, 970 truncated prefixes OK
failures: short I/O, failed builders, footer failure, caps, cleanup OK
table checks passed
```

The geometry and query counts follow from the fixture. Vector capacities are
implementation observations, and the false-positive count is reported rather
than asserted. These runs were executed with Clang 17/libstdc++ on Linux x86-64.
They are not measurements or execution claims for native M4/macOS.

```sh title="Terminal — independent native-file check"
python3 scripts/check_table_fixture.py --program ./build/debug/strata-table-tests
```

```text title="Expected native fixture and oracle output"
native table: bytes=970 blocks=8 entries=8
oracle: 970 bytes, 8 blocks, 8 versions; CRCs and Bloom bytes OK
oracle: existing fixture preserved on create collision
```

This mode also passed in the Linux authoring environment. Native M4 execution,
Apple full-device synchronization, and physical power-loss behavior remain
unverified. The implementation uses no Linux-only I/O operation beyond the
already separated Chapter 6 backend; run these checks with your native toolchain.

## Diagnose a wrong search that stays inside every allocation

Temporarily change this expression in the cursor constructor:

```cpp title="Temporary diagnostic — wrong sparse key"
if (InternalLess{}(table_.directory(middle).first, *target)) {
```

Only replace `.last` with `.first` in that single binary-search condition. Do not
change the directory encoding or the endpoint validator. Rebuild the table target
and run it directly in both configurations:

```sh title="Terminal — the deliberate search defect must fail"
cmake --build build/debug --target strata-table-tests --parallel
./build/debug/strata-table-tests
cmake --build build/release --target strata-table-tests --parallel
./build/release/strata-table-tests
```

The small one-row-block boundary fixture passes because each block's first and
last keys are identical. The 256-byte-block oracle instead reports
`FAIL point presence oracle`. No invalid pointer is required for this bug: it
simply skips a block whose first key precedes the target but whose last key does
not. A memory sanitizer cannot prove the indexing contract.

**Restore `.last` before continuing.** Rebuild and rerun all eleven tests in both
configurations. The deliberate edit is not part of the finished implementation.
It was observed to fail in Debug, Release, and an instrumented build without a
sanitizer memory-error report; the corrected implementation was then restored.

## Check memory behavior separately from logical correctness

For a focused instrumented run, compile the complete dependencies, not only the
test translation unit:

```sh title="Terminal — focused address and undefined-behavior checks"
mkdir -p build/checks
clang++ -std=c++20 -O1 -g \
  -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
  -fsanitize=address,undefined -fno-omit-frame-pointer \
  -Iinclude \
  src/io/file.cpp src/memory/arena.cpp src/storage/memtable.cpp \
  src/storage/table.cpp tests/table.cpp \
  -o build/checks/table-sanitized
./build/checks/table-sanitized
python3 scripts/check_table_fixture.py --program ./build/checks/table-sanitized
```

Use the selected compatible `clang++`, not an unrelated architecture or toolchain
just to match a version number. The normal deterministic tests and native fixture
passed AddressSanitizer and UndefinedBehaviorSanitizer in the Linux environment;
Linux leak detection was enabled there. Supported sanitizer combinations and leak
detection differ by platform, so do not infer target support or target results
from that authoring run.

| Symptom | Likely cause and concrete check |
|---|---|
| A new table fails its header CRC | Check version/reserved bytes and CRC coverage `[0,20)`; do not include the CRC in itself. |
| Some old versions disappear only with larger blocks | Check the sparse search's **last** key and descending sequence comparison; run the 3,840-query oracle. |
| A filter rejects a tombstone-only key | Ensure erase entries call `bloom_add`, and hash the user key, not kind or sequence. |
| Reopening fails only after writing several blocks | Compare directory offset/complete frame length with actual emitted bytes; remember the 12-byte frame overhead. |
| `entry()` after an I/O error throws | The cursor is intentionally terminal. Do not keep using a pre-error borrowed row or silently retry it. |
| Correct results accompany growing memory | Count readers and live cursors and their vector capacities; one-block-per-cursor is not a global admission policy. |

## Two small variations that change understanding

First change the bulk fixture's block payload size from 256 to 512, leaving the
600-version oracle unchanged. Predict fewer directory entries, a larger cursor
buffer, and identical query answers. Run it and record the actual block count and
capacities. Do not change the tiny native fixture's options: its 970-byte oracle
intentionally fixes a format example. Restore the bulk size afterward so the
published sample output remains easy to compare.

Next change only the standalone Bloom experiment's 128-byte filter to 256 bytes.
All inserted keys must still pass; the absent-probe result is a new observation,
not a reason to hard-code a percentage. Explain why more versions of an already
inserted user key would not have the same effect as more distinct keys. Restore
the original size before the next chapter. These variations belong within the
chapter's testing allocation, not an extra benchmark project.

You now have independently readable immutable tables, a bounded streaming writer,
a sparse numeric index, a portable implemented checksum/filter, and explicit
cursor/result ownership. Nothing yet shares cached blocks, admits work against a
global byte budget, or installs these files in a recoverable database. **Chapter 9
adds memory budgets and a pinned block cache** on top of these finite buffers and
lifetimes; the table format and standalone checks remain the foundation.
