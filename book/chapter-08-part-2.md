# Chapter 8 — Indexed immutable tables

## Part 2 — Seek, verify, and diagnose

Continue from [Part 1](https://25d-228.github.io/cplusplus-learning/chapter-08.html).
The file format, CRC/Bloom operations, entry parser, framed I/O, and builder are
already written. Complete `table.hpp` and `table.cpp` below. Part 3 then supplies
the complete tests, native byte-check script, and build integration. All three
parts share **5.0 implementation hours**; they are not separate projects or
extra chapter allocations.

A reader must not turn a small on-disk integer into an arbitrary allocation.
It must also avoid returning a span into a local buffer that is about to die.
We solve those two problems together: one owned metadata buffer per reader,
one owned block buffer per cursor, and a small owning result for a point lookup.

## Keep a point result after its cursor disappears

Continue the header, still inside `namespace strata::storage`:

```cpp title="include/strata/storage/table.hpp"
struct OwnedEntry {
    InternalKey key{};
    EntryKind kind = EntryKind::erase;
    std::array<std::byte, event_bytes> bytes{};
    std::size_t length = 0;
    EntryView view() const & noexcept { return {key, kind, {bytes.data(), length}}; }
    EntryView view() const && = delete;
};
```

`OwnedEntry` is a value type. Its maximum 30-byte payload is stored directly in an
array, so returning an optional result requires no payload allocation. `length`
selects the live encoded prefix. This object is not serialized with `sizeof`;
its possible padding and unused array suffix are irrelevant to the file format.
`view()` borrows this owner, and the deleted rvalue overload discourages borrowing
from a temporary. Copy or move the **owner**, not a saved view into its array,
when retaining a result beyond a scope.

The reader's nested cursor holds a reference to the reader, a vector of frame
bytes, and an optional current `EntryView`. That optional owns the *view object*,
not the bytes it describes.

```cpp title="include/strata/storage/table.hpp"
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
        std::size_t buffer_capacity() const noexcept { return buffer_.capacity(); }
        EntryView entry() const;
        void next();
    private:
        friend class TableReader;
        Cursor(TableReader& table, std::optional<InternalKey> target);
        void load(std::size_t block);
        TableReader& table_;
        std::vector<std::byte> buffer_;
        std::span<const std::byte> payload_{};
        std::size_t block_ = 0;
        std::size_t at_ = 0;
        std::uint32_t remaining_ = 0;
        std::optional<EntryView> current_;
    };
```

The private `Directory` is a decoded in-memory convenience. We do not create an
array of these structures or use its `sizeof` as an on-disk width. Each access
will decode exactly 46 bytes from the retained metadata. `buffer_capacity` is a
small observation hook for comparing requested storage with actual vector capacity.

A cursor cannot be copied or moved. Returning `Cursor{...}` directly constructs
it in the caller under the same prvalue initialization rule used for Chapter 7's
prepared token. The reader cannot move either, so a cursor's reader reference
has one stable owner. These restrictions do not extend lifetimes: the caller
must destroy cursors before their reader.

```cpp title="include/strata/storage/table.hpp"
    TableReader(io::File file, std::uint64_t file_bytes);
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
    std::span<const std::byte> filter() const noexcept;
    io::File file_;
    std::vector<std::byte> metadata_;
    std::size_t block_bytes_ = 0;
    std::size_t blocks_ = 0;
    std::uint64_t entries_ = 0;
    std::uint64_t block_reads_ = 0;
};

}  // namespace strata::storage
#endif
```

The reader takes `File` **by value**, then moves it into its member. Its destructor
closes that file through Chapter 6's RAII owner. If construction fails, the member
is still destroyed and closes it. A caller-owned `FileOps` backend must outlive
this reader; moving a `File` does not extend the backend's lifetime.

`begin` and `lower_bound` are lvalue-only because their cursors borrow the reader.
`find` returns an owning optional; `nullopt` means no qualifying version in this
table. A present result whose kind is `erase` is a deletion, not absence. Later
multi-table lookup must not skip that tombstone and resurrect a put from an older
file. The reader methods are deliberately nonconcurrent; even `block_reads_` is
an ordinary, unsynchronized diagnostic counter.

## Establish bounds using fixed-size reads

Continue `src/storage/table.cpp`. The reader receives a file byte count. For a
freshly built file it can use `TableInfo::file_bytes`; a later opening layer can
obtain the count from file metadata. We treat the supplied count as untrusted:
check its ceiling, confirm the footer's count, and probe one byte beyond it.
The underlying file must remain unchanged during opening and reading. A size
observation plus positional reads does not eliminate a race with another writer.

```cpp title="src/storage/table.cpp"
TableReader::TableReader(io::File file, std::uint64_t file_bytes)
    : file_(std::move(file)) {
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
```

The minimum file size is 96 bytes: a header, footer, and metadata frame with
zero directory entries and the minimum eight-byte filter. Before subtracting
40 to locate the footer, we know the file is at least that large. Both initial
reads use fixed stack arrays. Their CRCs and magic bytes are checked before
using the footer's variable extents.

The key extent test is expressed by subtraction. Let $L$ be the supplied file
size, $O$ the metadata offset, and $S$ its complete frame size. After proving
$24\le O\le L-40$, we require

$$
S = L-40-O.
$$

This simultaneously rejects gaps, overlap with the footer, and a metadata extent
outside the file without computing an unchecked $O+S$. The ceiling on $S$ is
checked before the allocation in the next listing. A declared size smaller than
the actual file is rejected by the one-byte end probe; a larger size normally
fails an exact read or the footer checks. We do not scan past an invalid footer
looking for something plausible.

The constructor continues; do not add a closing brace between these two blocks:

```cpp title="src/storage/table.cpp"
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

`metadata_` retains the **whole frame**, including its eight-byte prefix and
four-byte CRC. The returned payload span merely selects its inside. Counts are
validated before `metadata_size` and before any directory slicing. Empty blocks
are not legal, so zero blocks and zero total entries must agree.

The directory loop checks more than sorted endpoints. Each block must begin
exactly at the previous block's end, fit the data-block bound, and finish before
metadata. The final end must equal the metadata start. Thus no extra data region
is silently ignored, and two directory entries cannot refer to the same frame.
The check on total entries is a coarse upper bound, derived from the minimum
22-byte entry. Its exact agreement with the data is checked by `verify_all`.

Opening validates only the header, footer, and metadata. It performs **zero data
block reads**. This is necessary for useful random access, but it is not a claim
that every data block has already passed its checksum or semantic checks.

## Decode a sparse entry without another index allocation

```cpp title="src/storage/table.cpp"
TableReader::Directory TableReader::directory(std::size_t index) const {
    if (index >= blocks_) { throw std::out_of_range("table directory index"); }
    const auto in = std::span{metadata_}.subspan(8 + 12 + index * directory_bytes,
                                                directory_bytes);
    return {get_key(in.first(17)), get_key(in.subspan(17, 17)),
            get_le<8>(in.subspan(34, 8)),
            static_cast<std::size_t>(get_le<4>(in.subspan(42, 4)))};
}

std::span<const std::byte> TableReader::filter() const noexcept {
    const auto start = 8 + 12 + blocks_ * directory_bytes;
    return std::span{metadata_}.subspan(start, metadata_.size() - 4 - start);
}

bool TableReader::may_contain(Key key) const {
    valid_key(key);
    return blocks_ != 0 && bloom_may_contain(filter(), key);
}
```

The directory begins at metadata-frame offset $8+12=20$. Entry $i$ begins at
$20+46i$. The filter follows the last directory entry and ends before the frame's
four-byte CRC. All of these slices rely on the constructor having validated the
exact metadata length and count.

`may_contain` validates the requested relation. It never hashes an internal
sequence. For an empty table it returns false regardless of the stored filter
bits. For a nonempty table it consults the checked metadata filter; its boolean
is not a row result.

## Search last keys, then seek inside one block

The directory is ordered by disjoint internal-key ranges. For a target $T$, select
the first directory entry whose **last** key is not before $T$. Blocks with last
keys before $T$ cannot contain the lower bound. If every block ends before $T$,
the result is the end cursor without allocating a data buffer.

This is the partition used by lower-bound binary search; the
[C++ binary-search specification](https://eel.is/c++draft/alg.binary.search)
explains why a partitioned range is the necessary condition, not just a vague
claim that “the keys look sorted.” Here we write the loop explicitly over encoded
metadata rather than constructing a separate container of keys.

```cpp title="src/storage/table.cpp"
TableReader::Cursor TableReader::begin() & { return Cursor{*this, std::nullopt}; }
TableReader::Cursor TableReader::lower_bound(InternalKey target) & {
    valid_key(target.user);  // A search sequence of zero is legal.
    return Cursor{*this, target};
}

TableReader::Cursor::Cursor(TableReader& table, std::optional<InternalKey> target)
    : table_(table) {
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
    buffer_.resize(table_.block_bytes_ + frame_overhead);
    load(low);
    while (target && current_ && InternalLess{}(current_->key, *target)) { next(); }
}
```

`low` and `high` delimit a half-open interval of candidate directory entries.
`middle = low + (high - low) / 2` avoids adding the two endpoints directly.
On the true branch, the whole middle block is too early, so its index can be
excluded with `low = middle + 1`. Otherwise the answer is at or before middle,
so `high = middle`. Both steps shrink the interval.

Consider this *internal-key* order for one relation:

| Block | Versions in order |
|---|---|
| 0 | key 10 at sequences 9, 7 |
| 1 | key 10 at sequences 5, 3 |
| 2 | key 11 at sequence 8 |

For key 10 visible at sequence 6, the target is `(10,6)`. With `low=0, high=3`,
`middle=1`: last `(10,3)` is not before the target, so `high=1`. Now middle is 0;
last `(10,7)` **is** before the target because larger sequences sort first, so
`low=1`. We load block 1 and return sequence 5. At visibility 8 we must instead
load block 0 and return sequence 7. Binary-searching for the first *first key*
not before `(10,8)` would skip block 0 and return the wrong version.

After selecting a block, the cursor scans its encoded entries until it reaches
the lower bound. This is a bounded linear within-block seek, not a second binary
index. It avoids a per-record offset array. The directory search is logarithmic
in block count; entry validation and seeking cost linear work in the selected
block's payload. There is no claim that a single disk read means a single
comparison or a constant-time point lookup.

For a valid table, the chosen last key guarantees that the lower bound lies
inside the chosen block, even when the same user key has versions in many blocks.
A result for a *different* user key means the requested key has no visible version
here. A target sequence of zero intentionally falls after every stored version
of that user key.

## Validate a whole block before exposing its first row

```cpp title="src/storage/table.cpp"
void TableReader::Cursor::load(std::size_t block) {
    current_.reset();
    const auto d = table_.directory(block);
    payload_ = read_frame(table_.file_, d.offset, d.frame_bytes, data_tag, buffer_);
    ++table_.block_reads_;
    require(payload_.size() >= 4, "missing block count");
    const auto count = static_cast<std::uint32_t>(get_le<4>(payload_.first(4)));
    require(count != 0 && count <= (payload_.size() - 4) / 22, "block count bound");
    std::size_t at = 4;
    std::optional<InternalKey> previous;
    for (std::uint32_t i = 0; i < count; ++i) {
        const auto e = read_entry(payload_, at);
        require(!previous || InternalLess{}(*previous, e.key), "unordered block entries");
        require(i != 0 || e.key == d.first, "block first key mismatch");
        require(i + 1 != count || e.key == d.last, "block last key mismatch");
        require(bloom_may_contain(table_.filter(), e.key.user), "Bloom false negative");
        previous = e.key;
    }
    require(at == payload_.size(), "trailing block bytes");
    block_ = block;
    at_ = 4;
    current_ = read_entry(payload_, at_);
    remaining_ = count - 1;
}
```

`load` first clears the current entry, then reads into the cursor's existing
buffer. It validates the CRC, record count, every row encoding, strict internal
order, both directory endpoints, and complete payload consumption. It also checks
that each stored user key passes the metadata Bloom filter. Only then does it
publish the first `EntryView` in `current_`.

The parsing pass uses one local previous key; it does not retain a vector of
rows. A second decode obtains the first view. Later `next` calls decode each
following row again. This trades modest repeated bounded parsing for simple
ownership and no per-record auxiliary allocation. Optimizing it requires a
measurement and a new memory accounting argument, not silently introducing an
unbounded cache.

`block_reads_` increments after a frame passes its checksum. It counts loaded
frames, not system calls, bytes, physical disk operations, or time: Chapter 6
may need many short reads to fill one frame. A failed checksum does not increment
it; a later semantic error can. It is a diagnostic for these tests, not a durable
or cross-thread performance counter.

```cpp title="src/storage/table.cpp"
EntryView TableReader::Cursor::entry() const {
    if (!current_) { throw std::out_of_range("table cursor at end"); }
    return *current_;
}

void TableReader::Cursor::next() {
    if (!current_) { return; }
    current_.reset();
    // Failure leaves a terminal cursor, never the previous visible entry.
    if (remaining_ != 0) {
        current_ = read_entry(payload_, at_);
        --remaining_;
    } else if (block_ + 1 < table_.blocks_) {
        load(block_ + 1);
    }
}
```

Treat every call to `next`, successful or not, as invalidating previously borrowed
entry values. Advancing within one block may happen to leave the old bytes intact,
but crossing a block reuses that allocation. Keeping the same address is not
keeping the same value. Copy a needed row or use the owning point-result API.

An exception during advancement leaves `current_` empty. The cursor becomes
terminal; `entry()` throws and `next()` is a no-op. It does not retry uncertain
I/O or return a stale row as though advancement succeeded. Create a new cursor
only after deciding how to handle the underlying error.

Two cursors have independent buffers and positions. Advancing one does not
invalidate a view from the other, provided the reader and immutable file remain
alive. This is a lifetime property under single-threaded calls, not permission
to race their shared `File` backend or diagnostic state.

## A filter positive is permission to search, not proof of a row

```cpp title="src/storage/table.cpp"
std::optional<OwnedEntry> TableReader::find(Key key, std::uint64_t visible) & {
    if (!may_contain(key)) { return std::nullopt; }
    auto cursor = lower_bound({key, visible});
    if (!cursor.valid() || cursor.entry().key.user != key) { return std::nullopt; }
    const auto entry = cursor.entry();
    OwnedEntry result;
    result.key = entry.key;
    result.kind = entry.kind;
    result.length = entry.value.size();
    std::copy(entry.value.begin(), entry.value.end(), result.bytes.begin());
    return result;
}

void TableReader::verify_all() & {
    std::uint64_t count = 0;
    for (auto cursor = begin(); cursor.valid(); cursor.next()) { ++count; }
    require(count == entries_, "table total entry count mismatch");
}

}  // namespace strata::storage
```

`find` can return immediately on a filter negative. On a positive it performs the
real lower-bound lookup and checks the user key. It copies at most 30 bytes into
`OwnedEntry` *before* the local cursor dies. A tombstone returns an owning result
with length zero; absence returns `nullopt`.

`verify_all` visits every frame and compares the observed version count with the
footer. It bypasses the early-return Bloom path by using `begin`. Use it when
checking a freshly produced table or investigating corruption. An ordinary lookup
is not an implicit full-file scrub.

### A checked filter is still not an authentication boundary

A random changed filter byte is normally detected by its metadata CRC. However,
a file with deliberately cleared filter bits **and a recomputed CRC** can make a
point lookup return absent without reading the relevant data. Full verification
will read that data and detect the filter false negative. Similarly, a negative
lookup does not inspect an unrelated damaged data block. We test this distinction
rather than claiming that opening a table proves every negative answer against
arbitrarily forged metadata.

The format assumes an honest producer and immutable files, checks accidental
corruption, and bounds parsing of malformed input. Authentication or proof against
a malicious party who can rewrite all bytes is outside its contract. Even a
complete semantic scan cannot discover rows that an attacker consistently removed
from both data and metadata.

### Working storage is bounded per reader, not yet per engine

For $D$ actual blocks, $F$ filter bytes, block payload bound $B$, and $C$ active
cursors that have loaded a block, the requested dynamic element storage is

$$
A_{\mathrm{read}} = (24+46D+F) + C(B+12).
$$

The first term is the entire metadata frame. The second is the set of independent
cursor buffers. Empty/end cursors can allocate no block storage. Numeric state,
vector objects, allocator overhead, and OS file caching are additional. The tests
print actual vector capacities rather than calling this formula process memory.
Arbitrarily creating cursors or readers is still arbitrarily much memory. Chapter 9
will add admission and accounting; this chapter establishes finite requests and
explicit lifetimes for it to control.

Continue to [Part 3 — Verification and native checks](https://25d-228.github.io/cplusplus-learning/chapter-08-part-3.html)
for the complete test program, independent native-byte checker, and build
integration. These are part of the same chapter, not later prerequisites.
