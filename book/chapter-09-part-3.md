# Chapter 9 — Memory budgets and a pinned block cache

## Part 3 — Reader integration and pressure tests

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-09.html) supplies the
common ledger; [Part 2](https://25d-228.github.io/cplusplus-learning/chapter-09-part-2.html)
supplies pins and eviction. Now connect them at Chapter 8's actual frame-loading
boundary, rather than introducing a second reader that ignores the format checks.
This is the final part of Chapter 9 and shares its 3.5-hour allocation.

A successful cache miss must mean more than "some bytes were read." Before those
bytes can be shared, validate the frame length/tag/checksum, entry counts and
bounds, key order, directory endpoints, record encodings, and Bloom membership.
The cache stores a complete **validated frame**, including framing bytes. Hits
can then avoid both disk access and a repeated full validation pass, provided all
readers use the correct immutable file identity.

## Replace a reusable buffer with an owning pin

Keep `OwnedEntry` unchanged. In `include/strata/storage/table.hpp`, replace the
**entire TableReader class definition**, from `class TableReader {` through its
closing `};`, with the following definition. Keep the enclosing namespace and
header guard. Part 1 already added the `block_cache.hpp` include and changed the
builder declarations; no other header definition needs replacement.

```cpp title="include/strata/storage/table.hpp — replace TableReader"
class TableReader {
    struct Directory {
        InternalKey first;
        InternalKey last;
        std::uint64_t offset;
        std::size_t frame_bytes;
    };
public:
    class Cursor {
    public:
        Cursor(const Cursor&) = delete;
        Cursor& operator=(const Cursor&) = delete;
        Cursor(Cursor&&) = delete;
        Cursor& operator=(Cursor&&) = delete;
        bool valid() const noexcept { return current_.has_value(); }
        std::size_t buffer_capacity() const noexcept { return pin_.capacity(); }
        EntryView entry() const;
        void next();
    private:
        friend class TableReader;
        Cursor(TableReader& table, std::optional<InternalKey> target, bool cached = true);
        void load(std::size_t block);
        TableReader& table_;
        bool cached_;
        BlockPin pin_;
        std::span<const std::byte> payload_{};
        std::size_t block_ = 0;
        std::size_t at_ = 0;
        std::uint32_t remaining_ = 0;
        std::optional<EntryView> current_;
    };

    TableReader(memory::Budget& budget, io::File file, std::uint64_t file_bytes,
                BlockCache* cache = nullptr, FileId file_id = 0);
    TableReader(const TableReader&) = delete;
    TableReader& operator=(const TableReader&) = delete;
    TableReader(TableReader&&) = delete;
    TableReader& operator=(TableReader&&) = delete;
    Cursor begin() &;
    Cursor begin() && = delete;
    Cursor lower_bound(InternalKey target) &;
    Cursor lower_bound(InternalKey) && = delete;
    std::optional<OwnedEntry> find(Key key, std::uint64_t visible =
        std::numeric_limits<std::uint64_t>::max()) &;
    bool may_contain(Key key) const;
    void verify_all() &;
    std::uint64_t entries() const noexcept { return entries_; }
    std::size_t blocks() const noexcept { return blocks_; }
    std::uint64_t block_reads() const noexcept { return block_reads_; }
    std::size_t metadata_capacity() const noexcept { return metadata_.capacity(); }

private:
    Directory directory(std::size_t index) const;
    BlockPin load_block(Directory directory, bool cached);
    void validate_block(Directory directory, std::span<const std::byte> frame) const;
    std::span<const std::byte> filter() const noexcept;
    memory::Budget& budget_;
    BlockCache* cache_;
    FileId file_id_;
    io::File file_;
    memory::Vector<std::byte> metadata_;
    std::size_t block_bytes_ = 0;
    std::size_t blocks_ = 0;
    std::uint64_t entries_ = 0;
    std::uint64_t block_reads_ = 0;
};
```

The reader borrows a budget and, optionally, a cache. Both must outlive it. The
reader still owns its file and its metadata vector. Two readers sharing data
blocks retain **two charged metadata allocations**; this chapter does not pretend
to share metadata merely because their file IDs match.

Each cursor owns `BlockPin pin_` instead of a reusable vector. `buffer_capacity()`
now reports the capacity of its currently pinned frame, or zero when it has no
pin. The frame is usually smaller than the configured maximum block size. This
changes the observation method's precise meaning; it does not change the file
format or `entry()`'s borrowing contract.

`cached_` controls whether a cursor consults the cache. Public `begin` and
`lower_bound` construct normal cached cursors. `verify_all` will construct an
uncached cursor, because verifying stored bytes and trusting an earlier cached
copy are different operations.

## Charge metadata while constructing the reader

Replace the complete `TableReader` constructor in `src/storage/table.cpp`.
The format checks are the Chapter 8 checks; the new initialization and binding
check at the start make allocation ownership explicit:

```cpp title="src/storage/table.cpp — replace reader constructor"
TableReader::TableReader(memory::Budget& budget, io::File file, std::uint64_t file_bytes,
                         BlockCache* cache, FileId file_id)
    : budget_(budget), cache_(cache), file_id_(file_id), file_(std::move(file)),
      metadata_(memory::Allocator<std::byte>{budget}) {
    if ((cache_ && (&cache_->budget() != &budget_ || !cache_->knows(file_id_))) ||
        (!cache_ && file_id_ != 0)) {
        throw std::invalid_argument("invalid table cache binding");
    }
    require(file_bytes >= header_bytes + footer_bytes + frame_overhead + 20 &&
            file_bytes <= max_file_bytes, "table file size bound");
    std::array<std::byte, header_bytes> header{};
    std::array<std::byte, footer_bytes> footer{};
    file_.read_exact(0, header);
    file_.read_exact(file_bytes - footer_bytes, footer);
    require(std::equal(header_magic.begin(), header_magic.end(), header.begin()),
            "table header magic");
    require(get_le<4>(std::span{header}.last(4)) == crc32(std::span{header}.first(20)),
            "table header checksum");
    require(get_le<4>(std::span{header}.subspan(8, 4)) == 1, "unsupported table version");
    require(get_le<4>(std::span{header}.subspan(16, 4)) == 0, "reserved header bits");
    block_bytes_ = static_cast<std::size_t>(get_le<4>(std::span{header}.subspan(12, 4)));
    require(block_bytes_ >= 64 && block_bytes_ <= max_block_bytes, "data block cap");
    require(std::equal(footer_magic.begin(), footer_magic.end(), footer.begin()),
            "table footer magic");
    require(get_le<4>(std::span{footer}.last(4)) == crc32(std::span{footer}.first(36)),
            "table footer checksum");
    require(get_le<8>(std::span{footer}.subspan(28, 8)) == file_bytes, "table size mismatch");
    std::array<std::byte, 1> probe{};
    require(file_.read_up_to(file_bytes, probe) == 0, "trailing table bytes");
    const auto meta_at = get_le<8>(std::span{footer}.subspan(8, 8));
    const auto meta_size = static_cast<std::size_t>(get_le<4>(std::span{footer}.subspan(16, 4)));
    entries_ = get_le<8>(std::span{footer}.subspan(20, 8));
    require(meta_at >= header_bytes && meta_at <= file_bytes - footer_bytes,
            "metadata offset bound");
    require(meta_size >= frame_overhead + 20 && meta_size <= max_metadata_bytes + frame_overhead &&
            meta_size == file_bytes - footer_bytes - meta_at, "metadata size bound");

    metadata_.resize(meta_size);
    const auto payload = read_frame(file_, meta_at, meta_size, metadata_tag, metadata_);
    blocks_ = static_cast<std::size_t>(get_le<4>(payload.first(4)));
    const auto filter_bytes = static_cast<std::size_t>(get_le<4>(payload.subspan(4, 4)));
    require(blocks_ <= max_blocks && table_format::valid_filter_size(filter_bytes),
            "metadata count bound");
    require(get_le<4>(payload.subspan(8, 4)) == bloom_probes, "unsupported Bloom scheme");
    require(metadata_size(blocks_, filter_bytes) == payload.size(), "metadata layout mismatch");
    require((blocks_ == 0) == (entries_ == 0), "empty-table count mismatch");
    require(entries_ >= blocks_ &&
            entries_ <= blocks_ * ((block_bytes_ - 4) / 22), "entry count bound");
    std::uint64_t end = header_bytes;
    std::optional<InternalKey> previous;
    for (std::size_t i = 0; i < blocks_; ++i) {
        const auto d = directory(i);
        require(!InternalLess{}(d.last, d.first), "reversed block endpoints");
        require(!previous || InternalLess{}(*previous, d.first), "overlapping block order");
        require(d.offset == end && d.offset <= meta_at, "noncontiguous data blocks");
        require(d.frame_bytes >= frame_overhead + 4 + 22 &&
                d.frame_bytes <= frame_overhead + block_bytes_ &&
                d.frame_bytes <= meta_at - d.offset, "data frame bound");
        end += d.frame_bytes;
        previous = d.last;
    }
    require(end == meta_at, "unindexed data region");
}
```

When a cache is supplied, it must use this exact budget object and recognize the
file ID. Without a cache, require ID zero: there is no cache namespace in which
another number would have meaning. Both invalid bindings and metadata allocation
failures unwind the moved-in file owner, so an unsuccessful construction does
not leak a descriptor.

Notice the order before `metadata_.resize(meta_size)`: fixed header/footer reads,
file-bound checks, then the allocation. The byte ledger does not replace format
validation. A corrupt claimed size must still fail its format bounds rather than
consume the entire available budget.

The metadata reader still calls the existing `read_frame` helper. All its uses
accept spans, so changing a vector's allocator does not require copying its
contents into another vector. The old `directory`, `filter`, `may_contain`,
`begin`, and `lower_bound` definitions remain unchanged.

## Validate before a cache entry exists

Remove the old `Cursor::load` definition and replace it with the three complete
functions below. First add this reader-level validator. It receives the frame
range already bounded by a checked directory entry:

```cpp title="src/storage/table.cpp — add validate_block"
void TableReader::validate_block(Directory d, std::span<const std::byte> frame) const {
    require(frame.size() == d.frame_bytes, "cached frame size mismatch");
    require(get_le<4>(frame.first(4)) == frame.size() - frame_overhead,
            "frame length mismatch");
    require(get_le<4>(frame.subspan(4, 4)) == data_tag, "frame tag mismatch");
    require(get_le<4>(frame.last(4)) == crc32(frame.first(frame.size() - 4)),
            "frame checksum mismatch");
    const auto payload = frame.subspan(8, frame.size() - frame_overhead);
    require(payload.size() >= 4, "missing block count");
    const auto count = static_cast<std::uint32_t>(get_le<4>(payload.first(4)));
    require(count != 0 && count <= (payload.size() - 4) / 22, "block count bound");
    std::size_t at = 4;
    std::optional<InternalKey> previous;
    for (std::uint32_t i = 0; i < count; ++i) {
        const auto e = read_entry(payload, at);
        require(!previous || InternalLess{}(*previous, e.key), "unordered block entries");
        require(i != 0 || e.key == d.first, "block first key mismatch");
        require(i + 1 != count || e.key == d.last, "block last key mismatch");
        require(bloom_may_contain(filter(), e.key.user), "Bloom false negative");
        previous = e.key;
    }
    require(at == payload.size(), "trailing block bytes");
}
```

This moves the old semantic loop out of the cursor without weakening it. Its
first range comes from an already validated directory: frame length is at least
a frame header, count, one entry header, and checksum. The size-equality check
therefore precedes safe fixed-field reads. Every decoded row is validated before
**any** row in that frame can be exposed.

Now add the loader that connects the file owner to the cache:

```cpp title="src/storage/table.cpp — add load_block"
BlockPin TableReader::load_block(Directory d, bool cached) {
    const auto read = [this, d](std::span<std::byte> out) {
        file_.read_exact(d.offset, out);
        validate_block(d, out);
        ++block_reads_;  // A completely validated physical data-frame read.
    };
    if (cached && cache_) { return cache_->get({file_id_, d.offset, d.frame_bytes}, read); }
    return BlockPin::load(budget_, d.frame_bytes, read);
}
```

The lambda captures `this`, a pointer to the current reader, and a **copy** of
`d`, the directory descriptor. It does not copy the reader or retain a borrowed
local descriptor. Its reference arguments exist only during the synchronous
call to `get` or `BlockPin::load`.

A failed partial read or failed validator throws before `get` calls admission.
The incomplete candidate is destroyed and old cache entries survive. The
`block_reads_` counter now counts physical data-frame reads that passed the
**entire** validator, rather than Chapter 8's earlier increment after CRC alone.
Hits do not increment it; failed reads/semantic checks do not increment it.
The fixture backend separately counts all low-level read attempts, including
short transfers and failed calls.

A cache hit does not rerun this lambda. It reuses bytes previously validated for
the same immutable file incarnation, offset, and size. Passing one file ID for
two different files would break this reasoning even if their checksums were
individually valid. An identity contract is not replaced by a checksum.

Finally add the new `Cursor::load`:

```cpp title="src/storage/table.cpp — replace cursor load"
void TableReader::Cursor::load(std::size_t block) {
    current_.reset();
    payload_ = {};
    remaining_ = 0;
    pin_.reset();  // This cursor no longer prevents eviction of the old block.
    const auto d = table_.directory(block);
    pin_ = table_.load_block(d, cached_);
    payload_ = pin_.bytes().subspan(8, d.frame_bytes - frame_overhead);
    const auto count = static_cast<std::uint32_t>(get_le<4>(payload_.first(4)));
    block_ = block;
    at_ = 4;
    current_ = read_entry(payload_, at_);
    remaining_ = count - 1;
}
```

Clear the old visible entry and borrowed payload before dropping the old pin.
That invalidates the cursor's previous row and stops this cursor from preventing
its old block's eviction. Then acquire the new validated pin. If acquisition
throws, `current_` stays empty and the cursor is terminal; there is no half-visible
new row and no promise that the previous row can be retried.

The old block may still be resident or pinned by another cursor. Releasing this
cursor's handle does not authorize destroying somebody else's allocation.
After success, `payload_` borrows a subspan of the newly owned pin, and the first
row is decoded from bytes already validated by the loader. The count cannot be
zero because the validator rejected that case.

## Finish the cursor state changes

Replace the complete cursor constructor. It still searches directory **last**
keys, retaining Chapter 8's version-spanning correctness, but it no longer
allocates a maximum-size private buffer before `load`:

```cpp title="src/storage/table.cpp — replace cursor constructor"
TableReader::Cursor::Cursor(TableReader& table, std::optional<InternalKey> target,
                            bool cached)
    : table_(table), cached_(cached) {
    std::size_t low = 0;
    std::size_t high = table_.blocks_;
    if (target) {
        while (low < high) {
            const auto middle = low + (high - low) / 2;
            if (InternalLess{}(table_.directory(middle).last, *target)) {
                low = middle + 1;
            } else { high = middle; }
        }
    }
    if (low == table_.blocks_) { return; }
    load(low);
    while (target && current_ && InternalLess{}(current_->key, *target)) { next(); }
}
```

Replace `Cursor::next` as well. Normal end-of-stream now releases its pin rather
than retaining the final frame until cursor destruction. The conservative rule
remains: treat every call to `next` as invalidating the previous `EntryView`.
Do not retain its span just because some other owner happens to keep the same
allocation alive.

```cpp title="src/storage/table.cpp — replace next"
void TableReader::Cursor::next() {
    if (!current_) { return; }
    current_.reset();
    // Failure leaves a terminal cursor, never the previous visible entry.
    if (remaining_ != 0) {
        current_ = read_entry(payload_, at_);
        --remaining_;
    } else if (block_ + 1 < table_.blocks_) {
        load(block_ + 1);
    } else {
        payload_ = {};
        pin_.reset();
    }
}
```

Keep `Cursor::entry` and `TableReader::find` unchanged. A successful point lookup
still copies its at-most-30-byte payload into `OwnedEntry`. In particular, a
bypass pin can disappear on return from `find` while its returned row remains
valid: the result owns different inline storage.

Replace `verify_all` with this complete definition:

```cpp title="src/storage/table.cpp — replace verify_all"
void TableReader::verify_all() & {
    std::uint64_t count = 0;
    for (Cursor cursor{*this, std::nullopt, false}; cursor.valid(); cursor.next()) { ++count; }
    require(count == entries_, "table total entry count mismatch");
}
```

This cursor does not call cache lookup or admission. Every block is read and
validated anew using a charged temporary pin. A warm cache can continue serving
its previously validated bytes after an injected disk fault; `verify_all` must
still discover that fault. Verification needs headroom for its own block even
when the cache is full, because it does not evict entries behind the caller's back.

There is a performance tradeoff: without a cache, each new frame now allocates an
owning block rather than reusing one cursor vector. The uniform lifetime model is
explicit and testable; it is not a claim that this version minimizes allocations
for long scans. Cache hits do remove repeated frame allocations and I/O. Later
measurements can separate those cases before optimizing exclusive buffer reuse.

## Migrate the existing fixtures, not their assertions

All arena, memtable, builder, and reader constructors now require a budget first.
For example, `MemTable table{cap}` becomes `MemTable table{budget, cap}`, and
`write_table(file, table, options)` becomes
`write_table(budget, file, table, options)`. The application should own its budget
in its enclosing engine context, not in a hidden singleton.

The Chapter 7 and Chapter 8 test files use many short local fixture functions.
Give **each test executable** one explicit fixture ledger in its anonymous
namespace. The following mechanical edit updates exactly those known source
files and preserves all earlier assertions. Run it from the learner source root.
The marker makes a second run a no-op, not a second insertion of arguments.

```sh title="Terminal — adapt prerequisite test call sites"
python3 - <<'PYEDIT'
from pathlib import Path
import re

marker = "strata::memory::Budget fixture_budget{64 * 1024 * 1024};"
for name in ("tests/memtable.cpp", "tests/table.cpp"):
    path = Path(name)
    source = path.read_text()
    if marker in source:
        print(name, "already migrated")
        continue
    if "namespace {\n" not in source:
        raise SystemExit(f"Missing expected anonymous namespace: {name}")
    source = source.replace("namespace {\n", "namespace {\n" + marker + "\n", 1)
    source, count = re.subn(
        r"\b(Arena|MemTable|TableReader|TableBuilder)(\s+\w+\s*\{)",
        r"\1\2fixture_budget, ", source)
    if count == 0:
        raise SystemExit(f"No expected construction sites found: {name}")
    source = source.replace("write_table(file,", "write_table(fixture_budget, file,")
    path.write_text(source)
    print(name, "migrated")
PYEDIT
```

This script is a convenience for the **stated prerequisite files**, not a general
C++ refactoring tool. Review the resulting constructor arguments if you have
changed those fixtures. The 64 MiB limit does not allocate 64 MiB; it admits
actual requests as each test runs. The existing helper named `budget(chunks)` in
`tests/memtable.cpp` still computes a local arena cap; `fixture_budget` is a
different object and does not replace that helper.

## Use a bounded file fixture with multiple open readers

Create `tests/cache_file_ops.hpp`. This backend has 8192 fixed bytes and at most
15 simultaneous fixture descriptors. It returns at most seven bytes per transfer,
so the real `File::read_exact`/`write_all` loops remain involved. Its storage is
fixture memory outside the engine ledger; it is not a second storage engine.

```cpp title="tests/cache_file_ops.hpp"
#ifndef STRATA_TEST_CACHE_FILE_OPS_HPP
#define STRATA_TEST_CACHE_FILE_OPS_HPP

#include "strata/io/file.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <cstddef>
#include <cstdint>
#include <span>
#include <stdexcept>

class CacheFileOps final : public strata::io::FileOps {
public:
    std::array<std::byte, 8192> bytes{};
    std::size_t used = 0;
    std::size_t reads = 0;
    bool fail_reads = false;

    strata::io::File open() {
        for (std::size_t i = 1; i < live_.size(); ++i) {
            if (!live_[i]) {
                live_[i] = true;
                return strata::io::File::adopt(*this, static_cast<int>(i));
            }
        }
        throw std::runtime_error("fixture descriptor limit");
    }
    std::size_t live_files() const noexcept {
        return static_cast<std::size_t>(std::count(live_.begin(), live_.end(), true));
    }
    strata::io::Transfer read_at(int fd, std::span<std::byte> out,
                                std::uint64_t offset) noexcept override {
        ++reads;
        if (!valid(fd)) { return {0, EBADF}; }
        if (fail_reads) { return {0, EIO}; }
        if (offset >= used) { return {0, 0}; }
        const auto at = static_cast<std::size_t>(offset);
        const auto count = std::min({out.size(), used - at, std::size_t{7}});
        std::copy_n(bytes.data() + at, count, out.data());
        return {count, 0};
    }
    strata::io::Transfer write_at(int fd, std::span<const std::byte> in,
                                 std::uint64_t offset) noexcept override {
        if (!valid(fd)) { return {0, EBADF}; }
        if (offset >= bytes.size()) { return {0, ENOSPC}; }
        const auto at = static_cast<std::size_t>(offset);
        const auto count = std::min({in.size(), bytes.size() - at, std::size_t{7}});
        std::copy_n(in.data(), count, bytes.data() + at);
        used = std::max(used, at + count);
        return {count, 0};
    }
    int sync(int fd, strata::io::SyncMode mode) noexcept override {
        if (!valid(fd)) { return EBADF; }
        return mode == strata::io::SyncMode::file ? 0 : ENOTSUP;
    }
    int close(int fd) noexcept override {
        if (!valid(fd)) { return EBADF; }
        live_[static_cast<std::size_t>(fd)] = false;
        return 0;
    }
private:
    bool valid(int fd) const noexcept {
        return fd > 0 && static_cast<std::size_t>(fd) < live_.size() &&
               live_[static_cast<std::size_t>(fd)];
    }
    std::array<bool, 16> live_{};
};

#endif
```

`open` allocates no heap storage: it claims a Boolean descriptor slot.
Two readers can therefore own independent file handles to the same immutable
fixture bytes. `fail_reads` injects an I/O failure without making a cached hit
fail. `live_files` checks that failed reader construction closes its moved-in file.

## Test the ledger without depending on a library layout

Create `tests/block_cache.cpp`. The first helpers throw on a failed condition in
both Debug and Release. The charge probes briefly construct real objects to
measure this implementation's requests; they return numbers, not retained
allocations. Budget objects are declared before their users so they are destroyed
last.

```cpp title="tests/block_cache.cpp"
#include "strata/storage/table.hpp"
#include "cache_file_ops.hpp"

#include <algorithm>
#include <array>
#include <cstddef>
#include <cstdint>
#include <exception>
#include <iostream>
#include <limits>
#include <memory>
#include <span>
#include <stdexcept>
#include <type_traits>
#include <utility>

namespace {
using namespace strata;
using namespace strata::storage;
using memory::Budget;
using memory::BudgetExceeded;

void require(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}

template<class Exception, class Action>
void expect_error(const char* message, const Action& action) {
    try { action(); }
    catch (const Exception&) { return; }
    throw std::runtime_error(message);
}

void zero(std::span<std::byte> out) { std::fill(out.begin(), out.end(), std::byte{0}); }

std::size_t block_charge(std::size_t bytes) {
    Budget budget{1024 * 1024};
    auto pin = BlockPin::load(budget, bytes, zero);
    require(pin.charge() == budget.allocated(), "block overhead escaped accounting");
    require(pin.charge() > pin.capacity(), "shared block has no control charge");
    return pin.charge();
}

std::size_t metadata_charge(std::size_t slots) {
    Budget budget{1024 * 1024};
    BlockCache cache{budget, slots, 1024 * 1024};
    require(cache.metadata_charge() == budget.allocated(), "slot allocation uncharged");
    return cache.metadata_charge();
}
```

`block_charge(64)` includes more than 64 requested bytes because the owning block
and its shared-control storage also exist. `metadata_charge(2)` measures the
actual fixed slot vector. These are local layout measurements, not constants to
write into an on-disk format.

Test empty reservations, subtraction-based overflow protection, moves, assignment,
and self-move. A maximum-size **reservation** is only a ledger operation; this
test does not try to allocate `SIZE_MAX` physical bytes:

```cpp title="tests/block_cache.cpp"
void check_budget() {
    Budget budget{16};
    require(budget.reserve(0).bytes() == 0, "zero reservation");
    {
        auto first = budget.reserve(12);
        require(budget.reserved() == 12 && budget.allocated() == 0, "reservation state");
        expect_error<BudgetExceeded>("oversized reservation accepted", [&] {
            auto rejected = budget.reserve(5);
        });
        auto moved = std::move(first);
        require(first.bytes() == 0 && moved.bytes() == 12 && budget.used() == 12,
                "lease move charged twice");
        auto second = budget.reserve(4);
        moved = std::move(second);
        require(budget.used() == 4 && second.bytes() == 0, "lease assignment leaked");
        auto& same = moved;
        moved = std::move(same);
        require(moved.bytes() == 4, "lease self move lost ownership");
    }
    require(budget.used() == 0 && budget.peak() == 16, "lease release/peak");
    Budget huge{std::numeric_limits<std::size_t>::max()};
    auto all = huge.reserve(huge.limit());
    expect_error<BudgetExceeded>("ledger arithmetic wrapped", [&] { (void)huge.reserve(1); });
    expect_error<std::bad_array_new_length>("allocation multiplication wrapped", [&] {
        (void)memory::Allocator<std::uint64_t>{budget}.allocate(
            std::numeric_limits<std::size_t>::max());
    });
    expect_error<std::invalid_argument>("zero failure ordinal accepted", [&] {
        budget.fail_allocation_on(0);
    });
}
```

After reserving 12 then 4, move-assigning the 4-byte lease over the 12-byte lease
must return the original 12 and leave exactly 4 reserved. The high-water mark
remains 16. Checking only zero usage at final scope exit would miss an incorrect
intermediate ownership transfer.

The allocator test checks capacity, failure cleanup, over-alignment, and
propagation across two distinct budgets:

```cpp title="tests/block_cache.cpp"
void check_allocator() {
    Budget budget{4096};
    {
        memory::Vector<std::uint64_t> values{memory::Allocator<std::uint64_t>{budget}};
        values.reserve(10);
        values.push_back(7);
        values.push_back(9);
        const auto charged = values.capacity() * sizeof(std::uint64_t);
        require(budget.allocated() == charged && budget.reserved() == 0,
                "vector counted size instead of capacity");
        auto* address = values.data();
        budget.fail_allocation_on(1);
        expect_error<std::bad_alloc>("growth failure missing", [&] {
            values.reserve(values.capacity() + 1);
        });
        require(values.data() == address && values[1] == 9 && budget.used() == charged,
                "failed growth changed value or leaked charge");
        values.clear();
        require(budget.used() == charged, "clear incorrectly released vector capacity");
    }
    require(budget.used() == 0, "vector destruction leaked");
    struct alignas(64) Aligned { std::uint64_t value = 4; };
    memory::Allocator<Aligned> allocator{budget};
    auto* pointer = allocator.allocate(1);
    std::construct_at(pointer);
    require(reinterpret_cast<std::uintptr_t>(pointer) % alignof(Aligned) == 0,
            "allocator lost over-alignment");
    require(budget.used() == sizeof(Aligned), "aligned allocation charge");
    std::destroy_at(pointer);
    allocator.deallocate(pointer, 1);
    allocator.deallocate(allocator.allocate(0), 0);
    require(budget.used() == 0, "direct allocator release");

    Budget other{4096};
    {
        memory::Vector<std::uint64_t> left(3, memory::Allocator<std::uint64_t>{budget});
        memory::Vector<std::uint64_t> right(5, memory::Allocator<std::uint64_t>{other});
        const auto old_left = budget.used();
        const auto old_right = other.used();
        left.swap(right);
        require(&left.get_allocator().budget() == &other &&
                &right.get_allocator().budget() == &budget, "swap lost budget identity");
        require(budget.used() == old_left && other.used() == old_right,
                "swap transferred physical allocations between ledgers");
        left = std::move(right);
        require(other.used() == 0 && budget.used() == old_left,
                "move assignment failed to release its destination");
    }
    require(budget.used() == 0 && other.used() == 0, "allocator propagation leaked");
}
```

Swapping two vectors swaps the allocation owners and their allocator identities,
not the physical allocations' ledger charges. The original budgets still account
for their original allocations. Move-assigning over a destination must free that
destination using its old allocator before it takes ownership of the source's
allocation and allocator. The test makes those lifetime effects observable.

## Trace pins, bypass, and the final owner

The next test uses a two-block resident limit. A and B are pinned; a miss for C
must return a charged bypass without displacing either. Copying and moving C's
pin must not allocate another block. Releasing only one copy must not return the
charge early.

```cpp title="tests/block_cache.cpp"
void check_pins() {
    Budget budget{64 * 1024};
    const auto one = block_charge(64);
    const auto meta = metadata_charge(2);
    {
        BlockCache cache{budget, 2, meta + 2 * one};
        const auto file = cache.new_file_id();
        const BlockKey a{file, 0, 64}, b{file, 64, 64}, c{file, 128, 64};
        std::size_t loads = 0;
        const auto read = [&](std::span<std::byte> out) { ++loads; zero(out); };
        auto pa = cache.get(a, read);
        auto pb = cache.get(b, read);
        const auto baseline = budget.used();
        auto hit = cache.get(a, read);
        require(loads == 2 && hit.bytes().data() == pa.bytes().data(), "cache did not share hit");
        require(budget.used() == baseline, "hit charged again");
        hit.reset();
        auto pc = cache.get(c, read);
        require(!cache.contains(c) && cache.contains(a) && cache.contains(b),
                "pinned admission displaced a resident");
        require(budget.used() == baseline + one, "bypass not charged");
        auto copied = pc;
        auto moved = std::move(pc);
        require(!pc && copied.bytes().data() == moved.bytes().data() &&
                budget.used() == baseline + one, "pin copy/move allocated payload");
        moved.reset();
        require(budget.used() == baseline + one, "non-last bypass pin freed charge");
        copied.reset();
        require(budget.used() == baseline, "last bypass pin leaked charge");
        pb.reset();
        require(budget.used() == baseline, "cached last-pin release freed retained block");
        auto admitted = cache.get(c, read);
        require(cache.contains(a) && !cache.contains(b) && cache.contains(c),
                "wrong unpinned victim");
        require(pa.bytes()[0] == std::byte{0} && budget.used() == baseline,
                "eviction invalidated pinned bytes or leaked");
        cache.check_structure();
        pa.reset();
        admitted.reset();
        cache.trim();
        require(cache.entries() == 0 && budget.used() == meta, "trim failed");
        require(cache.statistics().hits == 1 && cache.statistics().misses == 4 &&
                cache.statistics().bypasses == 1 && cache.statistics().evictions == 3,
                "cache trace counters");
    }
    require(budget.used() == 0, "cache metadata leaked");
    BlockPin survivor;
    {
        BlockCache cache{budget, 1, metadata_charge(1) + one};
        survivor = cache.get({cache.new_file_id(), 0, 64}, zero);
    }
    require(budget.used() == one && survivor.bytes().size() == 64,
            "pin could not outlive cache owner");
    survivor.reset();
    require(budget.used() == 0, "surviving pin leaked");
}
```

After B's reader pin disappears, B is still charged but evictable. A second miss
for C now admits C by removing B. The first bypass was never installed, so the
loader really runs again. Trimming after all reader pins disappear leaves only
the slot metadata. Cache destruction releases that too.

The final scope in this function tests a different direction of lifetime: the
cache disappears before its returned pin. Only the frame's own charge survives;
its bytes remain accessible until that last pin is reset.

## Make byte pressure differ from slot pressure

A cache with four slots can still hold only two small blocks under its byte limit.
The test below exercises recency, a failed plan that must evict nothing, a
successful two-victim admission, and an oversized block that must bypass:

```cpp title="tests/block_cache.cpp"
void check_recency_and_bytes() {
    Budget budget{64 * 1024};
    const auto one = block_charge(64);
    const auto meta = metadata_charge(4);
    BlockCache cache{budget, 4, meta + 2 * one};
    const auto file = cache.new_file_id();
    const BlockKey a{file, 0, 64}, b{file, 64, 64}, c{file, 128, 64};
    cache.get(a, zero).reset();
    cache.get(b, zero).reset();
    cache.get(a, zero).reset();
    cache.get(c, zero).reset();
    require(cache.contains(a) && !cache.contains(b) && cache.contains(c),
            "hit did not refresh recency");
    // Empty slots remain; byte capacity, not slot count, caused that eviction.
    require(cache.entries() == 2, "byte limit ignored");
    auto pinned = cache.get(a, zero);
    const auto before = cache.statistics().evictions;
    auto large = cache.get({file, 256, 128}, zero);
    require(cache.contains(c) && cache.statistics().evictions == before,
            "failed admission partially evicted old entries");
    large.reset();
    pinned.reset();
    auto admitted = cache.get({file, 256, 128}, zero);
    require(cache.entries() == 1 && !cache.contains(a) && !cache.contains(c) &&
            cache.statistics().evictions == before + 2, "multi-victim byte admission");
    cache.check_structure();
    const auto oversized = cache.get({file, 512, 4096}, zero);
    require(cache.entries() == 1 && oversized.bytes().size() == 4096,
            "oversized block should bypass");
}
```

In the partial-plan case A is pinned and C is not. The larger candidate cannot
fit by releasing C alone, so C must remain resident. Checking only that A survived
would miss a policy bug that unnecessarily discarded C on a failed admission.

## Fail before eviction, including the second allocation

The next function first reserves every remaining admitted byte. A hit still
works because it shares existing storage; a cold miss throws `BudgetExceeded`
before touching the old entry. Then a loader partially fills its private candidate
and throws. That candidate must not appear in the cache.

Finally sweep the allocator failure ordinal from one upward until construction
succeeds. This visits the control-storage and payload requests without asserting
that every library combines shared allocations in the same way. The finite bound
is a diagnostic guard for this small constructor; exhausting it fails the test
rather than silently declaring success.

```cpp title="tests/block_cache.cpp"
void check_failure_and_identity() {
    Budget budget{64 * 1024};
    const auto one = block_charge(64);
    BlockCache cache{budget, 1, metadata_charge(1) + one};
    const auto old_file = cache.new_file_id();
    const BlockKey old{old_file, 0, 64};
    const BlockKey fresh{cache.new_file_id(), 0, 64};
    cache.get(old, zero).reset();
    const auto baseline = budget.used();
    {
        auto hold = budget.reserve(budget.limit() - budget.used());
        auto hit = cache.get(old, zero);
        require(hit.bytes().size() == 64, "hit needs allocation headroom");
        expect_error<BudgetExceeded>("cold miss bypassed global budget", [&] {
            (void)cache.get(fresh, zero);
        });
        require(cache.contains(old), "budget rejection evicted old block");
    }
    expect_error<std::runtime_error>("loader failure missing", [&] {
        (void)cache.get(fresh, [](std::span<std::byte> out) {
            out[0] = std::byte{99};
            throw std::runtime_error("validation failed after a partial fill");
        });
    });
    require(cache.contains(old) && budget.used() == baseline, "loader failure lost state");
    std::size_t failures = 0;
    bool completed = false;
    for (std::size_t ordinal = 1; ordinal <= 16; ++ordinal) {
        budget.fail_allocation_on(ordinal);
        try {
            auto pin = cache.get(fresh, [](std::span<std::byte> out) {
                std::fill(out.begin(), out.end(), std::byte{42});
            });
            require(pin.bytes()[0] == std::byte{42} && !cache.contains(old),
                    "new file ID reused old bytes");
            completed = true;
        } catch (const std::bad_alloc&) {
            ++failures;
            require(cache.contains(old) && !cache.contains(fresh) &&
                    budget.used() == baseline && budget.reserved() == 0,
                    "failed allocation lost resident or leaked charge");
        }
        budget.clear_failure();
        if (completed) { break; }
    }
    require(completed && failures >= 2, "did not exercise control and payload failures");
    expect_error<std::invalid_argument>("unissued file ID accepted", [&] {
        (void)cache.get({0, 0, 64}, zero);
    });
    expect_error<std::invalid_argument>("zero block accepted", [&] {
        (void)cache.get({fresh.file, 0, 0}, zero);
    });
    cache.check_structure();
    std::cout << "allocation failure points: " << failures << '\n';
}
```

The old and fresh keys intentionally have the same offset and size but different
issued file IDs. Their fill bytes differ. The fresh result must contain 42, not
the old zeros. This exposes an implementation that accidentally drops file
identity from the lookup key.

After every injected allocation failure, check **both** old membership and exact
current usage. A global zero-at-exit check alone could overlook lost cache entries
or a temporary reservation left attached to a live scope.

## Enforce one budget across different subsystems

The next test makes the shared budget exactly one arena chunk. The first memtable
uses it; the second memtable cannot acquire a chunk despite its larger local cap.
A table builder cannot allocate workspaces against the same exhausted ledger
and must not write a header when construction fails.

```cpp title="tests/block_cache.cpp"
void check_shared_budget() {
    const auto chunk = memory::Arena::chunk_charge();
    Budget budget{chunk};
    {
        MemTable first{budget, 2 * chunk};
        const std::array changes{Mutation::put(Event{10, 1, 2, 3})};
        { auto pending = first.prepare(1, changes); pending.commit(); }
        require(budget.allocated() == chunk && first.reserved_bytes() == chunk,
                "arena omitted chunk overhead");
        MemTable second{budget, 2 * chunk};
        expect_error<BudgetExceeded>("two arenas exceeded shared bound", [&] {
            auto pending = second.prepare(1, changes);
        });
        require(second.size() == 0 && second.reserved_bytes() == 0 && first.size() == 1,
                "arena budget failure changed committed state");
        CacheFileOps ops;
        auto file = ops.open();
        expect_error<BudgetExceeded>("builder escaped shared bound", [&] {
            TableBuilder builder{budget, file, {64, 8, 32, 8192}};
        });
        require(ops.used == 0, "failed workspace allocation wrote a header");
    }
    require(budget.used() == 0, "arena destruction leaked");
    Budget tiny{1};
    expect_error<BudgetExceeded>("cache metadata uncharged", [&] {
        BlockCache cache{tiny, 1, 8192};
    });
    require(tiny.used() == 0, "metadata failure leaked");
    expect_error<std::invalid_argument>("cache accepted zero slots", [&] {
        BlockCache cache{budget, 0, 8192};
    });
    expect_error<std::invalid_argument>("cache ignored metadata byte limit", [&] {
        BlockCache cache{budget, 1, 0};
    });
}
```

This is the integration that a cache-only payload counter would miss. Cache
metadata also has to fit the common budget; a one-byte budget cannot create even
one slot. A zero-slot cache and a cache whose byte limit excludes its own metadata
are separate configuration errors, not allocation failures.

## Read actual encoded tables through shared pins

Build three one-row event blocks using the real table builder. For each block,
the encoded width is

$$
8 + 4 + 22 + 30 + 4 = 68\text{ bytes}.
$$

The terms are frame header, entry count, entry header, event value, and CRC,
respectively. Because two such rows would exceed the 64-byte payload cap, each
row occupies a separate frame. The builder's metadata and workspace allocations
must be gone before the reader/cache test starts.

```cpp title="tests/block_cache.cpp"
TableInfo make_fixture(CacheFileOps& ops, Budget& budget) {
    auto file = ops.open();
    TableBuilder builder{budget, file, {64, 8, 32, 8192}};
    for (const auto key : {std::uint64_t{10}, std::uint64_t{20}, std::uint64_t{30}}) {
        const auto mutation = Mutation::put(Event{key, 7, 1, 100 + key});
        builder.append({{mutation.key, 1}, mutation.kind, std::span<const std::byte>{mutation.bytes}.first(mutation.length)});
    }
    return builder.finish();
}
```

The boundary test opens two readers under the same immutable file ID. Their first
cursors must point into the same pinned frame, but their directory/filter metadata
remains separately charged. Holding blocks 10 and 20 while looking up 30 forces
a point-query bypass; the returned `OwnedEntry` survives after the bypass charge
is released.

The test then distinguishes a hit from verification, injects media damage after
warming the cache, checks a cold corrupt load never enters the cache, and forces
a cursor advance to fail under a fully reserved budget. The intentional file-byte
change is fault injection, **not** an allowed mutation of an immutable table.

```cpp title="tests/block_cache.cpp"
void check_table_boundary() {
    Budget budget{128 * 1024};
    CacheFileOps ops;
    const auto info = make_fixture(ops, budget);
    require(budget.used() == 0 && info.blocks == 3, "fixture builder lifetime");
    // One event: frame header8 + count4 + entry header22 + value30 + CRC4 = 68.
    const auto one = block_charge(68);
    const auto meta = metadata_charge(2);
    {
        BlockCache cache{budget, 2, meta + 2 * one};
        const auto file = cache.new_file_id();
        TableReader first{budget, ops.open(), info.file_bytes, &cache, file};
        TableReader second{budget, ops.open(), info.file_bytes, &cache, file};
        require(budget.used() == meta + first.metadata_capacity() + second.metadata_capacity(),
                "reader metadata not charged separately");
        auto left = first.begin();
        auto same = second.begin();
        require(left.entry().value.data() == same.entry().value.data() &&
                first.block_reads() == 1 && second.block_reads() == 0,
                "readers did not share a validated immutable frame");
        auto right = second.lower_bound({{Relation::events, 20}, 1});
        const auto baseline = budget.used();
        const auto result = first.find({Relation::events, 30});
        require(result && result->key.user.primary == 30 &&
                cache.statistics().bypasses == 1 && budget.used() == baseline,
                "point-result copy did not release its bypass pin");
        require(decode_event(left.entry().value).event_id == 10 &&
                decode_event(right.entry().value).event_id == 20, "pinned row changed");
        const auto reads = ops.reads;
        ops.fail_reads = true;
        require(first.find({Relation::events, 10}).has_value() && ops.reads == reads,
                "cache hit performed I/O");
        expect_error<io::IoError>("verify_all used cached data", [&] { first.verify_all(); });
        ops.fail_reads = false;
        const auto before_verify = first.block_reads();
        first.verify_all();
        require(first.block_reads() == before_verify + 3 && budget.used() == baseline,
                "verification failed to reread every block or leaked its bypass");

        // Inject media damage after warming the cache; this is not a legal table update.
        const auto damaged_at = table_format::header_bytes + 68 - 1;
        ops.bytes[damaged_at] ^= std::byte{1};
        require(first.find({Relation::events, 10}).has_value(), "warm hit unexpectedly reread");
        expect_error<FormatError>("verification missed media damage", [&] { first.verify_all(); });
        const auto fresh_id = cache.new_file_id();
        {
            TableReader cold{budget, ops.open(), info.file_bytes, &cache, fresh_id};
            const auto before = budget.used();
            expect_error<FormatError>("bad frame entered cache", [&] { (void)cold.begin(); });
            require(budget.used() == before && cache.entries() == 2,
                    "bad frame changed cache membership or budget");
        }
        ops.bytes[damaged_at] ^= std::byte{1};
        {
            auto hold = budget.reserve(budget.limit() - budget.used());
            left.next();  // Block20 is already resident: no allocation needed.
            require(left.entry().key.user.primary == 20, "hit under exhausted budget");
            expect_error<BudgetExceeded>("advance ignored exhausted budget", [&] { left.next(); });
            require(!left.valid(), "failed cursor advance kept a visible row");
            left.next();
            expect_error<std::out_of_range>("terminal cursor returned row", [&] { (void)left.entry(); });
            const auto live = ops.live_files();
            expect_error<BudgetExceeded>("new reader metadata not charged", [&] {
                TableReader rejected{budget, ops.open(), info.file_bytes};
            });
            require(ops.live_files() == live, "failed reader construction leaked file");
        }
        Budget other{4096};
        expect_error<std::invalid_argument>("reader/cache budget mismatch accepted", [&] {
            TableReader wrong{other, ops.open(), info.file_bytes, &cache, file};
        });
        require(other.used() == 0 && budget.reserved() == 0, "binding failure leaked");
        cache.check_structure();
    }
    require(budget.used() == 0 && ops.live_files() == 0, "table/cache scope leaked");
    std::cout << "table frames: " << info.blocks << "; all charges released\n";
}
```

`left.next()` can first succeed under exhausted headroom because block 20 is
already resident. Its next advance needs absent block 30 and therefore fails.
The cursor must become terminal, while another cursor's pinned frame remains
valid. The failed construction of a new reader must release its file despite
failing before it can obtain metadata storage.

Close the anonymous namespace and add the complete program entry point:

```cpp title="tests/block_cache.cpp"
}  // namespace

int main() {
    static_assert(!std::is_copy_constructible_v<Budget::Lease>);
    static_assert(std::is_nothrow_move_constructible_v<Budget::Lease>);
    static_assert(std::is_nothrow_copy_constructible_v<BlockPin>);
    static_assert(std::is_nothrow_move_constructible_v<BlockPin>);
    try {
        check_budget();
        check_allocator();
        check_pins();
        check_recency_and_bytes();
        check_failure_and_identity();
        check_shared_budget();
        check_table_boundary();
        std::cout << "budget/cache checks passed\n";
    } catch (const std::exception& error) {
        std::cerr << "budget/cache: " << error.what() << '\n';
        return 1;
    }
}
```

## Build and run the cumulative program

Append this block to the existing root `CMakeLists.txt`. The allocator is templated
in its header; only the non-template budget and cache definitions are added to
the existing engine library. Keep all earlier sources and test registrations.

```cmake title="CMakeLists.txt"
target_sources(strata-core PRIVATE src/memory/budget.cpp src/storage/block_cache.cpp)
add_executable(strata-cache-tests tests/block_cache.cpp)
target_link_libraries(strata-cache-tests PRIVATE strata-core)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(strata-cache-tests PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion)
endif()
add_test(NAME block-cache COMMAND strata-cache-tests)
```

```sh title="Terminal — Debug and Release checks"
cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build-debug --parallel
ctest --test-dir build-debug --output-on-failure
./build-debug/strata-cache-tests

cmake -S . -B build-release -DCMAKE_BUILD_TYPE=Release
cmake --build build-release --parallel
ctest --test-dir build-release --output-on-failure
```

There are now twelve CTest entries: the previous eleven plus `block-cache`.
The normal program's output on the tested Clang 17/libstdc++ Linux implementation
was:

```text title="Observed normal output — allocation count is implementation-dependent"
allocation failure points: 2
table frames: 3; all charges released
budget/cache checks passed
```

The count of two reflects that implementation's requests for shared object/control
storage and frame storage. The test's ownership and release assertions do not
hard-code their byte layouts. Run the same program on native Apple ARM64; a
passing Linux run does not establish target macOS execution or a target ABI size.
No toolchain upgrade is required merely to match an author's compiler banner:
the source uses the course's C++20 features and allocator interfaces.

Use the existing sanitizer workflow, or configure a separate build as follows
with a compatible Clang toolchain:

```sh title="Terminal — instrumented checks"
cmake -S . -B build-sanitized -DCMAKE_BUILD_TYPE=Debug   -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer"   -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build-sanitized --parallel
ctest --test-dir build-sanitized --output-on-failure
```

These tests include fake I/O for deterministic pressure/fault cases and the earlier
native-file fixtures for codec/table integration. Neither is a power-loss durability
experiment. Sanitizers can expose memory and undefined-behavior defects; the
explicit ledger and cache assertions check policies sanitizers do not know.

## Deliberately break recency, not lifetime

Inside `BlockCache::get`, temporarily replace `touch(hit);` with a comment so a
hit no longer updates recency. Do not change the loader, pin, or eviction code.
Rebuild and run only the new test in Debug and Release:

```sh title="Terminal — temporary logical defect"
cmake --build build-debug --parallel
ctest --test-dir build-debug -R '^block-cache$' --output-on-failure
cmake --build build-release --parallel
ctest --test-dir build-release -R '^block-cache$' --output-on-failure
```

Expected failure: `budget/cache: hit did not refresh recency`, with nonzero process
status. A, B, hit-A, C should retain A and C. Without the touch, the implementation
evicts A instead. Every allocation can remain valid and every charge can still
balance, so a sanitizer report is not required for the defect to be real.

Restore `touch(hit);`, rebuild, and rerun the full twelve-test suite before
continuing. The source listings in this chapter contain the correct function.
Do not retain the diagnostic edit in the next chapter's prerequisite state.

A second prediction needs no code defect: while a resident block is pinned, call
`trim`. The entry remains. Reset the pin and call `trim` again; the block charge
is returned, leaving metadata until cache destruction. Explain the difference
using owners, not the vague phrase "the cache freed the pointer."

## Diagnose the boundary that failed

| Symptom | First boundary to inspect |
|---|---|
| Old tests reject constructor arguments | A required first budget argument or the matching declaration/definition migration was missed. |
| Usage falls after `vector.clear()` | The counter follows logical size instead of allocator-owned capacity. |
| Last bypass-pin release leaves bytes charged | A shared owner still exists, or deallocation did not return its original ledger charge. |
| A failed miss removes an old entry | Eviction happened before candidate construction/validation or before the feasibility pass completed. |
| A hit reads the file | The loader is being invoked before lookup, or readers received different file IDs. |
| Replacing a file returns old rows | File identity was reused for a different immutable incarnation, or omitted from the key. |
| `verify_all` misses damage after a warm hit | Verification used the cache rather than the uncached cursor path. |
| Termination at budget destruction | A lease/allocation outlived the ledger, or a release path is missing. |

Do not respond to every `BudgetExceeded` by increasing a cap. Inspect retained
metadata, resident blocks, live pins, and temporary candidate headroom first.
A cache can satisfy its own resident limit while the shared budget is exhausted
by another subsystem. A bypass can be correct while repeated bypass reads are an
undesirable workload pattern. Those are distinguishable observations, not one
undifferentiated "out of memory" symptom.

## What the boundary now guarantees

The implemented ledger admits requested dynamic storage before acquisition and
returns charges on matching release. The cache has bounded charged lookup/recency
metadata, byte-based residency, immutable owning pins, stable file-incarnation
keys, nonthrowing admission after successful preparation, and charged bypass.
Failed candidate construction, I/O, or validation does not discard older entries.
The reader shares a validated frame without weakening its parser, while full
verification rereads stored bytes.

The limits remain explicit: single-threaded/exclusive use; no process-RSS cap;
no accounting for hidden allocator overhead; no unbounded input or metadata
container disguised by a payload counter; no automatic durable table installation;
and no claim that a pin synchronizes concurrent cache mutation. A shared budget
is a common admission domain, not a collection of mutexes or a filesystem log.

The next chapter connects prepared memtable changes to a framed write-ahead log.
These allocation boundaries matter there: rejection before any log append is
not the same state as an uncertain write after an irreversible external effect.

## Technical references

The explanations use the C++20 working draft N4861's
[allocator requirements](https://timsong-cpp.github.io/cppwp/n4861/allocator.requirements),
[allocator member functions](https://timsong-cpp.github.io/cppwp/n4861/allocator.members),
[allocator-trait member types](https://timsong-cpp.github.io/cppwp/n4861/allocator.traits.types),
and [container allocation/propagation rules](https://timsong-cpp.github.io/cppwp/n4861/container.requirements.general).
They define the storage and ownership contracts our adapter must satisfy, not the
budgeting policy implemented here.

For shared ownership, see [allocate_shared creation](https://timsong-cpp.github.io/cppwp/n4861/util.smartptr.shared.create),
[shared-pointer destruction](https://timsong-cpp.github.io/cppwp/n4861/util.smartptr.shared.dest),
and [use_count observers and synchronization limits](https://timsong-cpp.github.io/cppwp/n4861/util.smartptr.shared.obs).
For storage lifetime, see [construct_at](https://timsong-cpp.github.io/cppwp/n4861/specialized.construct),
[destroy_at](https://timsong-cpp.github.io/cppwp/n4861/specialized.destroy), and
[vector capacity](https://timsong-cpp.github.io/cppwp/n4861/vector.capacity).
The table encoding, checksums, and record-codec references remain in Chapter 8;
this chapter changes ownership and admission, not their byte format.

