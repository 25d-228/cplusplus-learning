# Chapter 17 — Grouping, checked aggregates, and complete queries

## Part 1 — Preserve the joined row through a second sort

A join can be correct without placing equal groups next to one another. Chapter 16
emits events in account order. Accounts 10 and 30 can belong to region 1 while
account 20 belongs to region 2. Their joined regions are then `1, 2, 1`. Flushing
a regional total whenever the region changes would emit region 1 twice.

This chapter completes a fixed query: select events in a half-open day interval,
join each to its visible account in the same snapshot, then count events and sum
integer cents by region and day. We sort the **whole joined row** into group order,
keep one unfinished group, and stream completed groups into an output sink. A
successful row is not a successful query: only valid end-of-stream followed by
successful sink publication produces a complete result.

> **Learning objectives**
> Reuse an external-memory algorithm through a small compile-time schema; preserve
> every joined field across spill; derive checked count/sum updates; implement
> key-change and final-group emission; distinguish progress, completion, and
> failure; and publish a completed result without exposing a partial replacement.

**Prerequisites:** all code through the three parts of
[Chapter 16](https://25d-228.github.io/cplusplus-learning/chapter-16.html), including
[Chapter 15's](https://25d-228.github.io/cplusplus-learning/chapter-15.html) complete
sort, spill ownership, counters, and native-file checks. Keep all 23 existing
CTest entries. Chapters 5, 6, 9, 12, and 14 supply templates, checked arithmetic,
charged allocation, snapshot ownership, and the pull protocol.

**Implementation allocation: 3.5 hours for all three parts together.** Allocate
1.0 hour to checked aggregation, 1.25 to joined-sort/query/output integration, and
1.25 to tests, debugging, and independent answers. The integration reuses your existing spill and merge mechanisms. Reading is
separate from the implementation allocation.

Create the files named by the listings. **Replace only
`src/query/external_sort.cpp` in full** with the segments in this part. Keep its
existing public header, tests, and event-sort interface. All other query files
below are new; Part 3 appends CMake targets. Repeated filename segments concatenate
in reading order. They are complete files, not patches with omitted helpers.

[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-17-part-2.html)
implements grouping, the completed-query API, and the file sink.
[Part 3](https://25d-228.github.io/cplusplus-learning/chapter-17-part-3.html)
provides all tests, the independent decoder, commands, and failure exercises.
No concurrency, SQL parser, query optimizer, cancellation, or durable query restart
is introduced here.

## Derive the new ordering

For joined row $x$, let $r(x)$ be its region, $d(x)$ its day, $e(x)$ its event ID,
$a(x)$ its account ID, and $v(x)$ its nonnegative amount in cents. A group key is
$g(x)=(r(x),d(x))$. Sort by the lexicographic tuple

$$
K(x)=(r(x),d(x),e(x),a(x),v(x)).
$$

The first two components make each group contiguous. Event ID orders rows within
a group; account and amount are deterministic tie breakers for otherwise identical
IDs in an arbitrary test stream. Visible database event IDs are unique, but the
sort itself does not deduplicate or discard equal rows. Each occurrence must still
contribute to the aggregate.

Suppose account-order input contains `(region, day, event, cents)` values
`(1,3,100,500)`, `(2,3,102,700)`, `(1,3,103,250)`. The second sort produces
`(1,3,100,500)`, `(1,3,103,250)`, `(2,3,102,700)`. Now one region/day accumulator
suffices: the transition to region 2 proves no later region-1/day-3 row remains.
Do not sort only by day or only by region; either omits part of the group key.

Tuple comparison uses ordinary ordered unsigned fields rather than subtraction.
It preserves the strict-weak-order requirements developed in Chapter 5; the
[C++ ordering requirements](https://eel.is/c++draft/alg.sorting.general) explain
why a comparison used by `std::sort` must have that property. The comparator is
nonthrowing and carries no mutable state.

## A compile-time schema, not a second copy of the algorithm

A **schema policy** here is a small type that supplies nested `Row`, `Batch`, and
`Order` types, two constants, and row codecs. It does not allocate, own files, or
schedule work. `TypedCore<EventSchema>` and `TypedCore<JoinedSchema>` instantiate
the same run generation, reader, heap, and merge-pass code for different row types.
There is no virtual call per comparison or decoded row.

In a template, `typename Schema::Row` says that the dependent name is a type.
Without that information, a dependent qualified name can be parsed as something
other than a type. See the draft's
[template name-resolution rules](https://eel.is/c++draft/temp.res). An alias such
as `using Row = typename Schema::Row;` then allows local declarations like
`std::optional<Row>`. A function definition outside the class must still write
`typename Schema::Row TypedCore<Schema>::take()` because that return type appears
before the member's scope has been established.

These policies are private to one implementation file. We do not expose an
unbounded plugin interface or claim a concept proves that the codecs preserve
fields. Tests must establish byte order, row widths, ordering, and round trips.
The existing `PullSource<Source, JoinedBatch>` concept remains the public adapter's
syntactic contract; it cannot prove correct EOF behavior or ownership either.

Start the new public header with the row comparator and a move-only facade. A
forward-declared `JoinedCore` keeps implementation details out of the header;
shared ownership inside the facade is for lifetime/allocation management, not
permission to copy the facade or use it concurrently.

```cpp title="include/strata/query/joined_sort.hpp — complete file, segment 1"
#ifndef STRATA_QUERY_JOINED_SORT_HPP
#define STRATA_QUERY_JOINED_SORT_HPP

#include "strata/query/merge_join.hpp"

#include <memory>
#include <optional>
#include <string_view>
#include <utility>

namespace strata::query {

struct JoinedOrder {
    bool operator()(const JoinedRow& a, const JoinedRow& b) const noexcept;
};
namespace sort_detail { class JoinedCore; }
class JoinedSortBuffer {
public:
    JoinedSortBuffer(std::shared_ptr<memory::Budget> budget, SortOptions options,
                     std::string_view parent, std::shared_ptr<SortStats> stats,
                     SpillHook hook = {});
    JoinedSortBuffer(const JoinedSortBuffer&) = delete;
    JoinedSortBuffer& operator=(const JoinedSortBuffer&) = delete;
    JoinedSortBuffer(JoinedSortBuffer&&) noexcept = default;
    JoinedSortBuffer& operator=(JoinedSortBuffer&&) noexcept = default;
    void append(JoinedRow row) &;
    void finish() &;
    Pull next(JoinedBatch& output) &;
    void abort() noexcept;
    void retry_cleanup() noexcept;
    std::shared_ptr<memory::Budget> budget() const noexcept;
private:
    std::shared_ptr<sort_detail::JoinedCore> core_;
};
```

The replacement implementation starts with both comparators and common format
constants. Keeping `EventOrder` unchanged is important: the merge join still
requires `(account_id,event_id,...)` order, not regional order.

```cpp title="src/query/external_sort.cpp — complete file, segment 1"
#include "strata/query/joined_sort.hpp"
#include "strata/core/codec.hpp"
#include "strata/io/directory.hpp"
#include "strata/storage/table.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <fcntl.h>
#include <limits>
#include <memory>
#include <optional>
#include <span>
#include <stdexcept>
#include <system_error>
#include <tuple>
#include <type_traits>
#include <unistd.h>
#include <sys/stat.h>
#include <utility>

namespace strata::query {

bool EventOrder::operator()(const Event& a, const Event& b) const noexcept {
    return std::tie(a.account_id, a.event_id, a.day, a.amount_cents) <
           std::tie(b.account_id, b.event_id, b.day, b.amount_cents);
}
bool JoinedOrder::operator()(const JoinedRow& a, const JoinedRow& b) const noexcept {
    return std::tie(a.region_id, a.event.day, a.event.event_id, a.event.account_id, a.event.amount_cents) <
           std::tie(b.region_id, b.event.day, b.event.event_id, b.event.account_id, b.event.amount_cents);
}
std::shared_ptr<SortStats> make_sort_stats(std::shared_ptr<memory::Budget> budget) {
    return std::allocate_shared<SortStats>(memory::SharedAllocator<SortStats>{std::move(budget)});
}

namespace sort_detail {
using storage::table_format::crc32;
constexpr std::size_t max_runs = 128, max_fan_in = 32;
constexpr std::size_t header_bytes = 36, footer_bytes = 24;
constexpr std::array<std::byte, 8> magic{
    std::byte{'S'}, std::byte{'T'}, std::byte{'R'}, std::byte{'R'},
    std::byte{'U'}, std::byte{'N'}, std::byte{'0'}, std::byte{'1'}};
constexpr std::array<std::byte, 8> tail_magic{
    std::byte{'S'}, std::byte{'T'}, std::byte{'R'}, std::byte{'E'},
    std::byte{'N'}, std::byte{'D'}, std::byte{'0'}, std::byte{'1'}};

void require(bool condition, const char* message) {
    if (!condition) { throw FormatError(message); }
}
std::uint64_t run_bytes(std::uint64_t rows, std::size_t block_rows, std::size_t row_bytes) {
    if ((row_bytes != 28 && row_bytes != 32) || rows == 0 || rows > (std::uint64_t{1} << 27) || block_rows == 0 || block_rows > 4096) {
        throw std::length_error("run row limits");
    }
    const auto blocks = 1 + (rows - 1) / block_rows;
    // The validated caps make these products fit u64 and signed file offsets.
    return header_bytes + rows * row_bytes + blocks * 12 + footer_bytes;
}
```

## Distinguish row layout from row representation

Event runs retain width 28 and order ID 1. Joined runs use width 32 and order ID 2.
Both use the same 36-byte header, 24-byte footer, and checksummed frames. Header
version stays 1 because that format already contains row width and ordering ID.
The existing reader accepts only its own pair; version 1 never meant “every
possible row has the same interpretation.”

| Joined-row byte offset | Width | Meaning |
|---|---:|---|
| 0 | 8 | Event ID |
| 8 | 8 | Account ID |
| 16 | 4 | Day |
| 20 | 8 | Amount in cents |
| 28 | 4 | Region ID |

The event prefix reuses its existing explicit codec. The final four bytes cannot
be reconstructed from an event after the account snapshot is released, so they
must be written. Neither `sizeof(JoinedRow)` nor native struct padding defines
this representation. The in-memory chunk, in contrast, allocates `sizeof(Row)`
per element; its padding is real allocated storage and must be charged.

```cpp title="src/query/external_sort.cpp — complete file, segment 2"
struct EventSchema {
    using Row = Event;
    using Batch = EventBatch;
    using Order = EventOrder;
    static constexpr std::size_t row_bytes = 28;
    static constexpr std::uint32_t order_id = 1;
    static Row decode(std::span<const std::byte> row) {
        require(row.size() == row_bytes, "run row width");
        return {get_le<8>(row.first(8)), get_le<8>(row.subspan(8, 8)),
                static_cast<std::uint32_t>(get_le<4>(row.subspan(16, 4))),
                get_le<8>(row.subspan(20, 8))};
    }
    static void encode(Row event, std::span<std::byte> row) {
        require(row.size() == row_bytes, "run encode width");
        put_le<8>(row.first(8), event.event_id);
        put_le<8>(row.subspan(8, 8), event.account_id);
        put_le<4>(row.subspan(16, 4), event.day);
        put_le<8>(row.subspan(20, 8), event.amount_cents);
    }
};
struct JoinedSchema {
    using Row = JoinedRow;
    using Batch = JoinedBatch;
    using Order = JoinedOrder;
    static constexpr std::size_t row_bytes = 32;
    static constexpr std::uint32_t order_id = 2;
    static Row decode(std::span<const std::byte> row) {
        require(row.size() == row_bytes, "joined run row width");
        return {EventSchema::decode(row.first(28)),
                static_cast<std::uint32_t>(get_le<4>(row.last(4)))};
    }
    static void encode(Row value, std::span<std::byte> row) {
        require(row.size() == row_bytes, "joined run encode width");
        EventSchema::encode(value.event, row.first(28));
        put_le<4>(row.last(4), value.region_id);
    }
};
```

The header checksum covers the ordering ID as well as the width. A valid checksum
alone is insufficient: Part 3 changes the joined order ID to 1 and recomputes the
checksum. The joined reader must still reject that header. Checksums detect byte
changes; schema validation prevents a well-checksummed misinterpretation.

For $n>0$ rows, encoded width $w$, and $b>0$ rows per frame, let $L(n,w,b)$ be the
complete run length. Each frame adds eight prefix bytes and four CRC bytes:

$$
L(n,w,b)=36+nw+12\left\lceil\frac{n}{b}\right\rceil+24.
$$

For three joined rows with two rows per frame, the file is
$36+3(32)+2(12)+24=180$ bytes. The same event run is 168 bytes. A single joined-row
run is 104 bytes. Passing the trusted schema width to `Space` makes disk
reservation include the region bytes instead of silently undercharging them.

## Keep the existing ownership boundary

The next segments retain Chapter 15's mechanisms. `Scratch<T>` charges actual
array storage and frees it through the owning allocator. A `Run` is a move-only
claim on a `Space` slot, not an owning path string. Each slot remembers expected
rows/bytes and whether a name was created, completed, or retired. Capacity remains
fixed: at most 128 initial runs and 32 input readers. Those are explicit scale
limits, not permission to buffer all input.

```cpp title="src/query/external_sort.cpp — complete file, segment 3"
template<class T>
class Scratch {
public:
    Scratch(std::shared_ptr<memory::Budget> budget, std::size_t count)
        : allocator_(std::move(budget)), count_(count) {
        static_assert(std::is_nothrow_default_constructible_v<T>);
        static_assert(std::is_nothrow_destructible_v<T>);
        data_ = allocator_.allocate(count_);
        for (std::size_t i = 0; i < count_; ++i) { std::construct_at(data_ + i); }
    }
    ~Scratch() noexcept {
        if (data_) { std::destroy_n(data_, count_); allocator_.deallocate(data_, count_); }
    }
    Scratch(const Scratch&) = delete;
    Scratch& operator=(const Scratch&) = delete;
    std::span<T> span() noexcept { return {data_, count_}; }
    std::size_t bytes() const noexcept { return count_ * sizeof(T); }
private:
    memory::SharedAllocator<T> allocator_;
    T* data_ = nullptr;
    std::size_t count_;
};

class Space;
class Run {
public:
    Run() noexcept = default;
    Run(Space& space, std::size_t slot) noexcept : space_(&space), slot_(slot) {}
    ~Run() noexcept { reset(); }
    Run(const Run&) = delete;
    Run& operator=(const Run&) = delete;
    Run(Run&& other) noexcept
        : space_(std::exchange(other.space_, nullptr)), slot_(other.slot_) {}
    Run& operator=(Run&& other) noexcept {
        if (this != &other) { reset(); space_ = std::exchange(other.space_, nullptr); slot_ = other.slot_; }
        return *this;
    }
    void reset() noexcept;
    std::size_t slot() const noexcept { return slot_; }
private:
    Space* space_ = nullptr;
    std::size_t slot_ = 0;
};
```

`Space` now receives `row_width` once from the policy. It is not selected by an
untrusted file. The destructor retries retired names and attempts directory
removal after the run owners have been destroyed. Persistent unlink failure is
still recorded in caller-held statistics; destruction cannot force success.

```cpp title="src/query/external_sort.cpp — complete file, segment 4"
class Space {
public:
    struct Slot {
        std::uint64_t id = 0, rows = 0, bytes = 0;
        bool created = false, complete = false, retired = false;
    };
    Space(SortOptions options, std::string_view parent, std::shared_ptr<SortStats> stats,
          SpillHook hook, std::size_t row_width)
        : options_(options), stats_(std::move(stats)), hook_(hook), row_width_(row_width) {
        constexpr std::string_view suffix = "/strata-sort-XXXXXX";
        if (parent.empty() || parent.find('\0') != std::string_view::npos ||
            parent.size() >= pattern_.size() - suffix.size()) {
            throw std::invalid_argument("temporary parent path length");
        }
        std::copy(parent.begin(), parent.end(), pattern_.begin());
        std::copy(suffix.begin(), suffix.end(), pattern_.begin() + parent.size());
    }
    ~Space() noexcept { retry(); remove_directory(); }
    Space(const Space&) = delete;
    Space& operator=(const Space&) = delete;
    Run reserve(std::uint64_t rows);
    io::File create(const Run& run);
    io::File open(const Run& run);
    void close(io::File& file);
    void close_noexcept(io::File& file) noexcept;
    void retire(std::size_t slot) noexcept;
    void retry() noexcept;
    const Slot& slot(const Run& run) const { return slots_.at(run.slot()); }
    void completed(const Run& run);
    void write(io::File& file, const Run& run, std::uint64_t offset, std::span<const std::byte> data);
    void read(io::File& file, const Run& run, std::uint64_t offset, std::span<std::byte> data);
    void remove_directory() noexcept;
private:
    friend class Run;
    void ensure_directory();
    void remove_slot(std::size_t index) noexcept;
    void cleanup_error(int code) noexcept;
    void hook(SpillEvent event, std::uint64_t id, std::uint64_t offset) {
        if (hook_.call) { hook_.call(hook_.argument, event, stats_->directory.data(), id, offset); }
    }
    std::array<char, 32> name(std::uint64_t id) const;
    void opened() noexcept;
    SortOptions options_;
    std::shared_ptr<SortStats> stats_;
    SpillHook hook_;
    std::size_t row_width_;
    std::array<char, 512> pattern_{};
    io::Descriptor directory_;
    std::array<Slot, max_runs + 1> slots_{};
    std::uint64_t next_id_ = 1;
};
```

An input name remains protected until its `Run` is reset. Directory creation is
lazy, so an in-memory sort never needs a spill directory. The private parent must
exist and be trusted; the engine creates a unique child and uses the directory
handle for run operations. A slot and disk allowance are acquired before file
creation, preserving the partial-create cleanup contract from Chapter 15.

```cpp title="src/query/external_sort.cpp — complete file, segment 5"
void Run::reset() noexcept {
    if (space_) { space_->retire(slot_); space_ = nullptr; }
}
std::array<char, 32> Space::name(std::uint64_t id) const {
    std::array<char, 32> result{};
    const auto n = std::snprintf(result.data(), result.size(), "run-%020llu.bin",
                               static_cast<unsigned long long>(id));
    if (n < 0 || static_cast<std::size_t>(n) >= result.size()) {
        throw std::logic_error("run filename width");
    }
    return result;
}
void Space::cleanup_error(int code) noexcept {
    if (stats_->cleanup_errors != std::numeric_limits<std::size_t>::max()) { ++stats_->cleanup_errors; }
    if (stats_->first_cleanup_error == 0) { stats_->first_cleanup_error = code; }
}

void Space::ensure_directory() {
    if (directory_.get() >= 0) { return; }
    if (!::mkdtemp(pattern_.data())) { throw std::system_error(errno, std::generic_category(), "mkdtemp sort"); }
    stats_->directory = pattern_;
    stats_->directory_live = true;
    const int descriptor = ::open(pattern_.data(), O_RDONLY | O_DIRECTORY | O_CLOEXEC | O_NOFOLLOW);
    if (descriptor < 0) { throw std::system_error(errno, std::generic_category(), "open sort directory"); }
    directory_ = io::Descriptor{descriptor};
}
Run Space::reserve(std::uint64_t rows) {
    const auto bytes = run_bytes(rows, options_.block_rows, row_width_);
    if (bytes > options_.disk_bytes - stats_->disk_reserved) {
        throw std::length_error("spill coexistence disk allowance exhausted");
    }
    std::size_t slot_index = 0;
    while (slot_index < slots_.size() && slots_[slot_index].id != 0) { ++slot_index; }
    if (slot_index == slots_.size()) { throw std::length_error("spill name slots exhausted"); }
    if (next_id_ == std::numeric_limits<std::uint64_t>::max()) { throw std::overflow_error("run ID exhausted"); }
    slots_[slot_index] = Slot{next_id_++, rows, bytes, false, false, false};
    stats_->disk_reserved += bytes;
    stats_->peak_disk_reserved = std::max(stats_->peak_disk_reserved, stats_->disk_reserved);
    return Run{*this, slot_index};
}
```

Opening and closing a run also update the descriptor ledger. Exclusive creation
must not overwrite a prior run; size verification on open rejects a shortened or
extended file before interpreting its frames. Explicit close can throw; destructor
cleanup cannot. These are the same distinctions used by table and WAL owners.

```cpp title="src/query/external_sort.cpp — complete file, segment 6"
void Space::opened() noexcept {
    ++stats_->open_run_files;
    stats_->peak_open_run_files = std::max(stats_->peak_open_run_files, stats_->open_run_files);
}
io::File Space::create(const Run& run) {
    ensure_directory();
    if (stats_->open_run_files == options_.file_descriptors) { throw std::length_error("sort descriptor allowance"); }
    auto& entry = slots_.at(run.slot());
    const auto filename = name(entry.id);
    const int descriptor = ::openat(directory_.get(), filename.data(),
        O_RDWR | O_CREAT | O_EXCL | O_CLOEXEC | O_NOFOLLOW, 0600);
    if (descriptor < 0) { throw std::system_error(errno, std::generic_category(), "create run"); }
    entry.created = true;
    ++stats_->files_created; ++stats_->live_files; opened();
    return io::File::adopt_native(descriptor);
}

io::File Space::open(const Run& run) {
    const auto& entry = slot(run);
    require(entry.complete && entry.created && !entry.retired, "opening incomplete run");
    if (stats_->open_run_files == options_.file_descriptors) { throw std::length_error("sort descriptor allowance"); }
    const auto filename = name(entry.id);
    io::Descriptor descriptor{::openat(directory_.get(), filename.data(), O_RDONLY | O_CLOEXEC | O_NOFOLLOW)};
    if (descriptor.get() < 0) { throw std::system_error(errno, std::generic_category(), "open run"); }
    opened();  // Count the native handle even if validation subsequently fails.
    try {
        struct stat info{};
        if (::fstat(descriptor.get(), &info) != 0) { throw std::system_error(errno, std::generic_category(), "stat run"); }
        require(S_ISREG(info.st_mode) && info.st_nlink == 1 && info.st_size >= 0 &&
                static_cast<std::uint64_t>(info.st_size) == entry.bytes, "run file type or length");
    } catch (...) { --stats_->open_run_files; throw; }
    return io::File::adopt_native(descriptor.release());
}
void Space::close(io::File& file) {
    if (file.is_open()) { --stats_->open_run_files; file.close_checked(); }
}
void Space::close_noexcept(io::File& file) noexcept {
    try { close(file); } catch (const std::system_error& error) { cleanup_error(error.code().value()); }
    catch (...) { cleanup_error(EIO); }
}
```

Retiring means “this owner no longer needs the run,” not “the file is definitely
gone.” Only confirmed absence releases its disk charge. An output replacement
and all of its inputs can therefore coexist during a merge, and denied cleanup
can make a later disk reservation fail. Do not unconditionally decrement disk
usage in a destructor.

```cpp title="src/query/external_sort.cpp — complete file, segment 7"
void Space::remove_slot(std::size_t index) noexcept {
    auto& entry = slots_[index];
    if (entry.id == 0 || !entry.retired) { return; }
    try {
        if (entry.created) {
            hook(SpillEvent::before_unlink, entry.id, 0);
            const auto filename = name(entry.id);
            if (::unlinkat(directory_.get(), filename.data(), 0) != 0 && errno != ENOENT) {
                throw std::system_error(errno, std::generic_category(), "unlink run");
            }
            --stats_->live_files; ++stats_->files_removed;
        }
        stats_->disk_reserved -= entry.bytes;
        entry = {};
    } catch (const std::system_error& error) { cleanup_error(error.code().value()); }
    catch (...) { cleanup_error(EIO); }
}
void Space::retire(std::size_t slot_index) noexcept {
    slots_[slot_index].retired = true;
    remove_slot(slot_index);
}
void Space::retry() noexcept {
    for (std::size_t i = 0; i < slots_.size(); ++i) { remove_slot(i); }
}
void Space::remove_directory() noexcept {
    if (!stats_->directory_live || stats_->live_files != 0) { return; }
    if (::rmdir(stats_->directory.data()) != 0 && errno != ENOENT) { cleanup_error(errno); return; }
    stats_->directory_live = false;
    directory_ = io::Descriptor{};
}
```

The read/write counters measure bytes successfully transferred through the file
boundary, including partial effects before an exception. Completed-file counters
are separate. A verification reread costs real read bytes and is counted. Hooks
are borrowed synchronous test callbacks; their state must outlive the operation,
and they must not reenter the sorter.

```cpp title="src/query/external_sort.cpp — complete file, segment 8"
void Space::completed(const Run& run) {
    slots_[run.slot()].complete = true;
    ++stats_->completed_files;
    hook(SpillEvent::completed_run, slot(run).id, slot(run).bytes);
}
void Space::write(io::File& file, const Run& run, std::uint64_t offset, std::span<const std::byte> data) {
    hook(SpillEvent::before_write, slot(run).id, offset);
    const auto before = file.io_bytes().written;
    try { file.write_all(offset, data); }
    catch (...) {
        io::IoBytes::add(stats_->write_bytes, file.io_bytes().written - before, stats_->saturated); throw;
    }
    io::IoBytes::add(stats_->write_bytes, file.io_bytes().written - before, stats_->saturated);
}
void Space::read(io::File& file, const Run& run, std::uint64_t offset, std::span<std::byte> data) {
    hook(SpillEvent::before_read, slot(run).id, offset);
    const auto before = file.io_bytes().read;
    try { file.read_exact(offset, data); }
    catch (...) {
        io::IoBytes::add(stats_->read_bytes, file.io_bytes().read - before, stats_->saturated); throw;
    }
    io::IoBytes::add(stats_->read_bytes, file.io_bytes().read - before, stats_->saturated);
}
```

## Parameterize the frame writer

`Writer<Schema>` changes the row type, comparator, width, order ID, and codecs;
it does not change the protocol. The constructor reserves the complete file,
opens it, and writes the header. `append` validates nondecreasing order and the
promised count before encoding a row. This catches an algorithmic merge defect
before publishing a completed replacement.

```cpp title="src/query/external_sort.cpp — complete file, segment 9"
template<class Schema>
class Writer {
    using Row = typename Schema::Row;
    using Order = typename Schema::Order;
    static constexpr auto row_bytes = Schema::row_bytes;
public:
    Writer(Space& space, std::uint64_t rows, std::size_t block_rows, std::span<std::byte> buffer)
        : space_(space), run_(space.reserve(rows)), file_(space.create(run_)),
          block_rows_(block_rows), buffer_(buffer) {
        try {
            std::array<std::byte, header_bytes> header{};
            std::copy(magic.begin(), magic.end(), header.begin());
            auto bytes = std::span{header};
            put_le<4>(bytes.subspan(8, 4), 1); put_le<4>(bytes.subspan(12, 4), row_bytes);
            put_le<4>(bytes.subspan(16, 4), block_rows); put_le<4>(bytes.subspan(20, 4), Schema::order_id);
            put_le<8>(bytes.subspan(24, 8), rows);
            put_le<4>(bytes.last(4), crc32(bytes.first(header_bytes - 4)));
            space_.write(file_, run_, 0, bytes);
        } catch (...) { space_.close_noexcept(file_); throw; }
    }
    ~Writer() noexcept { space_.close_noexcept(file_); }
    Writer(const Writer&) = delete;
    Writer& operator=(const Writer&) = delete;

    void append(Row event) {
        if (written_ == space_.slot(run_).rows || (previous_ && Order{}(event, *previous_))) {
            throw std::logic_error("run writer count or order");
        }
        Schema::encode(event, buffer_.subspan(8 + buffered_ * row_bytes, row_bytes));
        previous_ = event; ++buffered_; ++written_;
        if (buffered_ == block_rows_) { flush(); }
    }
```

A full frame flushes; a final partial frame flushes in `finish`. Finishing checks
the row count, writes and checks the forecast footer position, closes the file,
and only then records completion. Query temporaries are not recovery authority:
there is deliberately no `fsync` here. `Run` retains responsibility for a partial
file if any step throws.

```cpp title="src/query/external_sort.cpp — complete file, segment 10"
    Run finish() {
        if (written_ != space_.slot(run_).rows) { throw std::logic_error("incomplete run output"); }
        flush();
        std::array<std::byte, footer_bytes> footer{};
        std::copy(tail_magic.begin(), tail_magic.end(), footer.begin());
        auto bytes = std::span{footer};
        put_le<8>(bytes.subspan(8, 8), written_); put_le<4>(bytes.subspan(16, 4), blocks_);
        put_le<4>(bytes.last(4), crc32(bytes.first(footer_bytes - 4)));
        space_.write(file_, run_, offset_, bytes);
        require(offset_ + footer_bytes == space_.slot(run_).bytes, "run forecast mismatch");
        space_.close(file_);  // Temporary query output is not fsync-backed recovery state.
        space_.completed(run_);
        return std::move(run_);
    }

private:
    void flush() {
        if (buffered_ == 0) { return; }
        const auto length = 12 + buffered_ * row_bytes;
        auto frame = buffer_.first(length);
        put_le<4>(frame.first(4), buffered_); put_le<4>(frame.subspan(4, 4), buffered_ * row_bytes);
        put_le<4>(frame.last(4), crc32(frame.first(length - 4)));
        space_.write(file_, run_, offset_, frame);
        offset_ += length; ++blocks_; buffered_ = 0;
    }
    Space& space_;
    Run run_;
    io::File file_;
    std::size_t block_rows_;
    std::span<std::byte> buffer_;
    std::uint64_t offset_ = header_bytes, written_ = 0;
    std::size_t buffered_ = 0;
    std::uint32_t blocks_ = 0;
    std::optional<Row> previous_;
};
```

## Validate before exposing typed rows

The reader compares both header width and order ID with its compile-time policy.
It also checks count, block size, footer, and checksums. A row decoder cannot repair
a wrong header or an out-of-order run. Its single charged frame buffer limits
resident encoded data regardless of run length.

```cpp title="src/query/external_sort.cpp — complete file, segment 11"
template<class Schema>
class Reader {
    using Row = typename Schema::Row;
    using Order = typename Schema::Order;
    static constexpr auto row_bytes = Schema::row_bytes;
public:
    Reader(std::shared_ptr<memory::Budget> budget, Space& space, const Run& run, std::size_t block_rows)
        : space_(space), run_(run), block_rows_(block_rows),
          buffer_(std::move(budget), 12 + row_bytes * block_rows), file_(space.open(run)) {
        try {
            std::array<std::byte, header_bytes> header{};
            std::array<std::byte, footer_bytes> footer{};
            space_.read(file_, run_, 0, header);
            space_.read(file_, run_, space_.slot(run_).bytes - footer_bytes, footer);
            const auto h = std::span<const std::byte>{header}, f = std::span<const std::byte>{footer};
            require(std::equal(magic.begin(), magic.end(), h.begin()) && get_le<4>(h.subspan(8, 4)) == 1 &&
                get_le<4>(h.subspan(12, 4)) == row_bytes && get_le<4>(h.subspan(16, 4)) == block_rows_ &&
                get_le<4>(h.subspan(20, 4)) == Schema::order_id && get_le<8>(h.subspan(24, 8)) == space_.slot(run_).rows &&
                get_le<4>(h.last(4)) == crc32(h.first(header_bytes - 4)), "run header");
            remaining_ = space_.slot(run_).rows;
            require(std::equal(tail_magic.begin(), tail_magic.end(), f.begin()) &&
                get_le<8>(f.subspan(8, 8)) == remaining_ &&
                get_le<4>(f.subspan(16, 4)) == 1 + (remaining_ - 1) / block_rows_ &&
                get_le<4>(f.last(4)) == crc32(f.first(footer_bytes - 4)), "run footer");
            refill();
        } catch (...) { space_.close_noexcept(file_); throw; }
    }
    ~Reader() noexcept { space_.close_noexcept(file_); }
    Reader(const Reader&) = delete;
    Reader& operator=(const Reader&) = delete;
```

`head` returns a value, so advancing the reader cannot invalidate the caller's
copy. `next` enforces exact end coverage. Each refill verifies frame size/CRC and
row order before handing the first row to the heap; `previous_` spans frame
boundaries. The final `remaining_` transition also detects trailing bytes.

```cpp title="src/query/external_sort.cpp — complete file, segment 12"
    bool valid() const noexcept { return remaining_ != 0; }
    Row head() {
        if (!valid()) { throw std::logic_error("exhausted run"); }
        return Schema::decode(buffer_.span().subspan(8 + position_ * row_bytes, row_bytes));
    }
    void next() {
        if (!valid()) { throw std::logic_error("advance exhausted run"); }
        --remaining_; ++position_;
        if (remaining_ == 0) {
            require(offset_ == space_.slot(run_).bytes - footer_bytes, "run trailing bytes");
            space_.close(file_);
        } else if (position_ == loaded_) { refill(); }
    }
    void close() { space_.close(file_); }

private:
    void refill() {
        const auto count = static_cast<std::size_t>(std::min<std::uint64_t>(block_rows_, remaining_));
        const auto length = 12 + count * row_bytes;
        auto frame = buffer_.span().first(length);
        space_.read(file_, run_, offset_, frame);
        require(get_le<4>(frame.first(4)) == count && get_le<4>(frame.subspan(4, 4)) == count * row_bytes &&
                get_le<4>(frame.last(4)) == crc32(frame.first(length - 4)), "run frame");
        // Validate the entire frame, including its boundary with the preceding frame.
        for (std::size_t i = 0; i < count; ++i) {
            const auto event = Schema::decode(frame.subspan(8 + i * row_bytes, row_bytes));
            require(!previous_ || !Order{}(event, *previous_), "run ordering");
            previous_ = event;
        }
        offset_ += length; loaded_ = count; position_ = 0;
    }
    Space& space_;
    const Run& run_;
    std::size_t block_rows_;
    Scratch<std::byte> buffer_;
    io::File file_;
    std::uint64_t remaining_ = 0, offset_ = header_bytes;
    std::size_t loaded_ = 0, position_ = 0;
    std::optional<Row> previous_;
};
```

## One core, two instantiations

The core owns generation storage, one output frame, bounded run descriptors,
readers, and heap indices. `Space` is declared before the `Run` arrays: reverse
member destruction keeps it alive while runs retire themselves. `cleanup` also
closes readers before retiring their names. `stack_` is an allowance reservation,
not a dynamically allocated replacement for the C++ call stack.

```cpp title="src/query/external_sort.cpp — complete file, segment 13"
template<class Schema>
class TypedCore {
    using Row = typename Schema::Row;
    using Batch = typename Schema::Batch;
    using Order = typename Schema::Order;
    static constexpr auto row_bytes = Schema::row_bytes;
public:
    TypedCore(std::shared_ptr<memory::Budget> budget, SortOptions options, std::string_view parent,
         std::shared_ptr<SortStats> stats, SpillHook hook)
        : budget_(std::move(budget)), options_(options), stats_(std::move(stats)),
          space_(options, parent, stats_, hook, row_bytes) {}
    ~TypedCore() noexcept { cleanup(); }
    void initialize(std::size_t fixed_charge);
    void append(Row event);
    void finish();
    Pull next(Batch& output);
    void abort() noexcept { state_ = StreamState::failed; cleanup(); chunk_.reset(); frame_.reset(); stack_.reset(); }
    void retry_cleanup() noexcept { space_.retry(); if (sealed_ || state_ == StreamState::failed) { space_.remove_directory(); } }
    std::shared_ptr<memory::Budget> budget() const noexcept { return budget_; }
private:
    void spill_chunk();
    void merge_pass();
    void open_group(std::size_t first, std::size_t count);
    void close_group();
    void cleanup() noexcept;
    bool before(std::size_t a, std::size_t b);
    void push(std::size_t reader);
    Row take();
    void sift_down();
    void account_workspace(std::size_t input_frames);
    std::shared_ptr<memory::Budget> budget_;
    SortOptions options_;
    std::shared_ptr<SortStats> stats_;
    Space space_;
    memory::Budget::Lease stack_;
    std::optional<Scratch<Row>> chunk_;
    std::optional<Scratch<std::byte>> frame_;
    std::array<Run, max_runs> runs_{}, next_runs_{};
    std::array<std::optional<Reader<Schema>>, max_fan_in> readers_{};
    std::array<std::size_t, max_fan_in> heap_{};
    std::size_t rows_ = 0, read_at_ = 0, run_count_ = 0, heap_size_ = 0;
    StreamState state_ = StreamState::ready;
    bool sealed_ = false;
};
```

Let $W$ be this sort's workspace allowance, $Q$ its measured fixed control charge
plus stack allowance, $F=12+wb$ one frame, $C$ its chunk row capacity, and $D$ its
run-file-descriptor allowance. Generation needs $Q+C\operatorname{sizeof}(Row)+F$.
Merging needs $Q+(f+1)F$ for $f$ input frames and one output frame. Therefore

$$
f=\min\left(32,D-1,\left\lfloor\frac{W-Q-F}{F}\right\rfloor\right),\qquad f\ge2.
$$

The initialization check requires enough space for both generation and a
three-frame merge. Checked arithmetic precedes subtraction. The root ledger
independently enforces simultaneous allocations across both sorts and storage;
one sort's configured allowance is not a reservation that bypasses that ledger.

```cpp title="src/query/external_sort.cpp — complete file, segment 14"
template<class Schema>
void TypedCore<Schema>::initialize(std::size_t fixed_charge) {
    stats_->fixed_charge = fixed_charge;
    const auto frame = checked_size_add(12, checked_size_mul(row_bytes, options_.block_rows));
    const auto fixed = checked_size_add(fixed_charge, options_.stack_allowance);
    const auto generation = checked_size_add(checked_size_mul(sizeof(Row), options_.chunk_rows), frame);
    const auto minimum = checked_size_add(fixed, std::max(generation, checked_size_mul(3, frame)));
    if (minimum > options_.workspace_bytes) { throw std::length_error("sort workspace cannot make progress"); }
    stats_->fan_in = std::min({max_fan_in, options_.file_descriptors - 1,
                              (options_.workspace_bytes - fixed - frame) / frame});
    if (stats_->fan_in < 2) { throw std::length_error("sort needs two input buffers"); }
    stack_ = budget_->reserve(options_.stack_allowance);
    chunk_.emplace(budget_, options_.chunk_rows);
    frame_.emplace(budget_, frame);
    account_workspace(0);
}
template<class Schema>
void TypedCore<Schema>::account_workspace(std::size_t input_frames) {
    auto bytes = checked_size_add(stats_->fixed_charge, options_.stack_allowance);
    if (chunk_) { bytes = checked_size_add(bytes, chunk_->bytes()); }
    if (frame_) { bytes = checked_size_add(bytes, frame_->bytes()); }
    bytes = checked_size_add(bytes, checked_size_mul(input_frames, 12 + row_bytes * options_.block_rows));
    if (bytes > options_.workspace_bytes) { throw std::length_error("sort workspace allowance"); }
    stats_->peak_workspace = std::max(stats_->peak_workspace, bytes);
}
```

Generation sorts only the filled chunk and writes it directly. Input beyond
`chunk_rows * max_runs` is rejected rather than growing metadata without a bound.
An exactly full first chunk stays in memory until another row arrives: small
queries need neither a file nor a merge. The cap applies independently to both
sorts; an inner join cannot increase event cardinality here.

```cpp title="src/query/external_sort.cpp — complete file, segment 15"
template<class Schema>
void TypedCore<Schema>::spill_chunk() {
    if (run_count_ == options_.max_runs) { throw std::length_error("initial run descriptor cap"); }
    auto rows = chunk_->span().first(rows_);
    std::sort(rows.begin(), rows.end(), Order{});
    Writer<Schema> writer{space_, rows_, options_.block_rows, frame_->span()};
    for (const auto& event : rows) { writer.append(event); }
    runs_[run_count_++] = writer.finish();
    ++stats_->initial_runs; rows_ = 0;
}
template<class Schema>
void TypedCore<Schema>::append(Row event) {
    if (state_ != StreamState::ready || sealed_) { throw std::logic_error("sort not accepting input"); }
    try {
        const auto maximum = checked_size_mul(options_.chunk_rows, options_.max_runs);
        if (stats_->input_rows == maximum) { throw std::length_error("sort input cap"); }
        if (rows_ == options_.chunk_rows) { spill_chunk(); }
        chunk_->span()[rows_++] = event; ++stats_->input_rows;
    } catch (...) { abort(); throw; }
}
```

## Preserve the heap algorithm

The heap stores reader indices, not copied runs. For equal complete rows, reader
index breaks ties deterministically. `push` moves a new head up the parent chain.
After consuming the smallest head, `sift_down` repairs the root against its smaller
child. Replacing `EventOrder` with `Order` in **all** of these operations matters:
a regional chunk sort combined with an account-order heap is not a regional sort.

```cpp title="src/query/external_sort.cpp — complete file, segment 16"
template<class Schema>
bool TypedCore<Schema>::before(std::size_t a, std::size_t b) {
    const auto x = readers_[a]->head(), y = readers_[b]->head();
    if (Order{}(x, y)) { return true; }
    if (Order{}(y, x)) { return false; }
    return a < b;  // Exact duplicates are indistinguishable, but heap decisions stay deterministic.
}
template<class Schema>
void TypedCore<Schema>::push(std::size_t reader) {
    std::size_t at = heap_size_++;
    while (at != 0) {
        const auto parent = (at - 1) / 2;
        if (!before(reader, heap_[parent])) { break; }
        heap_[at] = heap_[parent]; at = parent;
    }
    heap_[at] = reader;
}

template<class Schema>
void TypedCore<Schema>::sift_down() {
    if (heap_size_ == 0) { return; }
    const auto replacement = heap_[0];
    std::size_t at = 0;
    while (at * 2 + 1 < heap_size_) {
        auto child = at * 2 + 1;
        if (child + 1 < heap_size_ && before(heap_[child + 1], heap_[child])) { ++child; }
        if (!before(heap_[child], replacement)) { break; }
        heap_[at] = heap_[child]; at = child;
    }
    heap_[at] = replacement;
}
```

`take` copies a row before advancing its reader. An exhausted reader is removed
from the heap, and a changed nonexhausted head is repaired by the same operation.
`open_group` checks the simultaneous frame allowance before constructing readers.
A failure partway through construction is handled by the core's cleanup path.

```cpp title="src/query/external_sort.cpp — complete file, segment 17"
template<class Schema>
typename Schema::Row TypedCore<Schema>::take() {
    const auto index = heap_[0];
    const auto result = readers_[index]->head();  // Copy before advancing the reader's buffer.
    readers_[index]->next();
    if (!readers_[index]->valid()) { heap_[0] = heap_[--heap_size_]; }
    sift_down();
    return result;
}

template<class Schema>
void TypedCore<Schema>::open_group(std::size_t first, std::size_t count) {
    account_workspace(count);
    heap_size_ = 0;
    for (std::size_t i = 0; i < count; ++i) {
        readers_[i].emplace(budget_, space_, runs_[first + i], options_.block_rows);
        push(i);
    }
}
template<class Schema>
void TypedCore<Schema>::close_group() {
    for (auto& reader : readers_) { if (reader) { reader->close(); reader.reset(); } }
    heap_size_ = 0;
}
```

A merge pass reads at most the chosen fan-in, writes one output, closes readers,
then rereads the entire replacement before retiring inputs. A leftover singleton
moves its existing run owner instead of copying its bytes. The fixed
`next_runs_` array owns completed replacements if a later group fails.

```cpp title="src/query/external_sort.cpp — complete file, segment 18"
template<class Schema>
void TypedCore<Schema>::merge_pass() {
    std::size_t produced = 0;
    for (std::size_t first = 0; first < run_count_;) {
        const auto count = std::min(stats_->fan_in, run_count_ - first);
        if (count == 1) { next_runs_[produced++] = std::move(runs_[first++]); continue; }
        std::uint64_t rows = 0;
        for (std::size_t i = 0; i < count; ++i) { rows = checked_u64_add(rows, space_.slot(runs_[first + i]).rows); }
        open_group(first, count);
        Run output;
        {
            Writer<Schema> writer{space_, rows, options_.block_rows, frame_->span()};
            while (heap_size_ != 0) { writer.append(take()); }
            output = writer.finish();
        }
        close_group();
        // Verify the replacement using one input buffer BEFORE retiring any input name.
        account_workspace(1);
        {
            Reader<Schema> verify{budget_, space_, output, options_.block_rows};
            while (verify.valid()) { verify.next(); }
        }
        next_runs_[produced++] = std::move(output);
        for (std::size_t i = 0; i < count; ++i) { runs_[first + i].reset(); }
        ++stats_->merge_groups;
        first += count;
    }
    runs_.swap(next_runs_); run_count_ = produced; ++stats_->merge_passes;
}
```

After generation, `finish` releases the row chunk before opening merge buffers.
Materialized passes stop as soon as the remaining runs fit one final merge; that
last merge streams directly into consumer batches. EOF releases run owners and
charged buffers. Any failure clears the current output and makes the sort
terminal; retrying a partially consumed stream would be incorrect.

```cpp title="src/query/external_sort.cpp — complete file, segment 19"
template<class Schema>
void TypedCore<Schema>::finish() {
    if (state_ != StreamState::ready || sealed_) { throw std::logic_error("sort finish state"); }
    try {
        if (run_count_ == 0) {
            auto rows = chunk_->span().first(rows_);
            std::sort(rows.begin(), rows.end(), Order{});
            frame_.reset();
        } else {
            if (rows_ != 0) { spill_chunk(); }
            chunk_.reset();  // Merge fan-in may use the former chunk's allowance.
            while (run_count_ > stats_->fan_in) { merge_pass(); }
            open_group(0, run_count_);
        }
        sealed_ = true;
    } catch (...) { abort(); throw; }
}

template<class Schema>
void TypedCore<Schema>::cleanup() noexcept {
    for (auto& reader : readers_) { reader.reset(); }
    heap_size_ = 0;
    for (auto& run : runs_) { run.reset(); }
    for (auto& run : next_runs_) { run.reset(); }
    run_count_ = 0;
    space_.retry(); space_.remove_directory();
}
template<class Schema>
Pull TypedCore<Schema>::next(Batch& output) {
    check_output(output, budget_, 1, state_);
    if (!sealed_) { throw std::logic_error("finish sort before reading"); }
    output.clear();
    if (state_ == StreamState::ended) { return Pull::end; }
    try {
        if (chunk_) {
            while (read_at_ < rows_ && output.size() < output.capacity()) { output.append(chunk_->span()[read_at_++]); }
        } else {
            while (heap_size_ != 0 && output.size() < output.capacity()) { output.append(take()); }
        }
        if (output.size() == 0) {
            state_ = StreamState::ended; cleanup(); chunk_.reset(); frame_.reset(); stack_.reset();
            return Pull::end;
        }
        stats_->output_rows += output.size();
        return Pull::batch;
    } catch (...) { output.clear(); abort(); throw; }
}
```

For the 25-row joined fixture, use chunk capacity 3 and fan-in 2. Nine initial
runs reduce as `9 → 5 → 3 → 2`. Only the three intermediate passes materialize;
the final two-way merge does not create a seventeenth file. Eight full initial
runs and one singleton write $8(180)+104=1544$ bytes. The three passes write
1152, 1032, and 972 bytes, respectively: **4700 write bytes in total**. Input
reads plus replacement verification and final delivery total **7856 read bytes**.
Part 3 checks those mechanics against actual native files, not timing estimates.

The next two concrete classes inherit the template constructors with
`using Base::Base`. That syntax makes the corresponding base constructors
available for constructing the derived type. It does not add virtual dispatch.
We allocate and destroy each concrete type through its actual owning control
block, never through a raw nonvirtual base pointer.

```cpp title="src/query/external_sort.cpp — complete file, segment 20"
class Core final : public TypedCore<EventSchema> {
public:
    using TypedCore<EventSchema>::TypedCore;
};
class JoinedCore final : public TypedCore<JoinedSchema> {
public:
    using TypedCore<JoinedSchema>::TypedCore;
};

}  // namespace sort_detail

SortBuffer::SortBuffer(std::shared_ptr<memory::Budget> budget, SortOptions options,
                       std::string_view parent, std::shared_ptr<SortStats> stats, SpillHook hook) {
    if (!budget || !stats || stats->used || options.chunk_rows == 0 || options.chunk_rows > (1U << 20) ||
        options.block_rows == 0 || options.block_rows > 4096 || options.max_runs < 2 ||
        options.max_runs > sort_detail::max_runs || options.file_descriptors < 3 ||
        options.stack_allowance < 16 * 1024 || options.disk_bytes == 0) {
        throw std::invalid_argument("invalid sort configuration or reused statistics");
    }
    stats->used = true;
    const auto before = budget->allocated();
    core_ = std::allocate_shared<sort_detail::Core>(memory::SharedAllocator<sort_detail::Core>{budget},
        budget, options, parent, stats, hook);
    // Synchronous construction performs no other allocation before this observation.
    const auto fixed_charge = budget->allocated() - before;
    core_->initialize(fixed_charge);
}
void SortBuffer::append(Event event) & { core_->append(event); }
void SortBuffer::finish() & { core_->finish(); }
Pull SortBuffer::next(EventBatch& output) & { return core_->next(output); }
void SortBuffer::abort() noexcept { if (core_) { core_->abort(); } }
void SortBuffer::retry_cleanup() noexcept { if (core_) { core_->retry_cleanup(); } }
std::shared_ptr<memory::Budget> SortBuffer::budget() const noexcept { return core_->budget(); }
```

The new facade performs the same validation and charged allocation as the old
one. The fixed allocation is measured separately for each concrete instantiation;
`optional<JoinedRow>` and `optional<Event>` need not occupy equal space. The
measurement assumes synchronous construction with no interleaved ledger writers,
as did Chapter 15. Chapter 18 must introduce synchronization before concurrency
can rely on a revised accounting protocol.

```cpp title="src/query/external_sort.cpp — complete file, segment 21"

JoinedSortBuffer::JoinedSortBuffer(std::shared_ptr<memory::Budget> budget, SortOptions options,
                       std::string_view parent, std::shared_ptr<SortStats> stats, SpillHook hook) {
    if (!budget || !stats || stats->used || options.chunk_rows == 0 || options.chunk_rows > (1U << 20) ||
        options.block_rows == 0 || options.block_rows > 4096 || options.max_runs < 2 ||
        options.max_runs > sort_detail::max_runs || options.file_descriptors < 3 ||
        options.stack_allowance < 16 * 1024 || options.disk_bytes == 0) {
        throw std::invalid_argument("invalid sort configuration or reused statistics");
    }
    stats->used = true;
    const auto before = budget->allocated();
    core_ = std::allocate_shared<sort_detail::JoinedCore>(memory::SharedAllocator<sort_detail::JoinedCore>{budget},
        budget, options, parent, stats, hook);
    // Synchronous construction performs no other allocation before this observation.
    const auto fixed_charge = budget->allocated() - before;
    core_->initialize(fixed_charge);
}
void JoinedSortBuffer::append(JoinedRow event) & { core_->append(event); }
void JoinedSortBuffer::finish() & { core_->finish(); }
Pull JoinedSortBuffer::next(JoinedBatch& output) & { return core_->next(output); }
void JoinedSortBuffer::abort() noexcept { if (core_) { core_->abort(); } }
void JoinedSortBuffer::retry_cleanup() noexcept { if (core_) { core_->retry_cleanup(); } }
std::shared_ptr<memory::Budget> JoinedSortBuffer::budget() const noexcept { return core_->budget(); }

}  // namespace strata::query
```

## Stop retaining the upstream plan after generation

Complete the new header with `JoinedExternalSort`. Its optional `Inputs` owns the
entire upstream join and one joined batch. The first pull builds the second
sort; it does not promise a short wall-clock response. Empty upstream batches
mean progress, not EOF. Nonempty EOF is a protocol error.

Once every joined value has been copied into the second sort's chunk or runs,
`inputs_.reset()` destroys the first sort, join, scans, snapshot, and their
columns **before** second-sort merge readers are opened. The result remains
snapshot-consistent because joined rows already contain the account's region.
No later database lookup is required. The second sort's control object and
statistics can still outlive its data files.

```cpp title="include/strata/query/joined_sort.hpp — complete file, segment 2"
template<class Source> requires PullSource<Source, JoinedBatch>
class JoinedExternalSort {
    struct Inputs {
        Source source;
        JoinedBatch batch;
        Inputs(Source value, std::size_t capacity)
            : source(std::move(value)), batch(source.budget(), capacity) {}
    };
public:
    JoinedExternalSort(Source source, std::size_t input_rows, SortOptions options,
                       std::string_view parent, std::shared_ptr<SortStats> stats,
                       SpillHook hook = {})
        : inputs_(std::in_place, std::move(source), input_rows),
          sort_(inputs_->source.budget(), options, parent, std::move(stats), hook) {}
    JoinedExternalSort(const JoinedExternalSort&) = delete;
    JoinedExternalSort& operator=(const JoinedExternalSort&) = delete;
    JoinedExternalSort(JoinedExternalSort&&) = default;
    JoinedExternalSort& operator=(JoinedExternalSort&&) = default;
    std::shared_ptr<memory::Budget> budget() const noexcept { return sort_.budget(); }
    Pull next(JoinedBatch& output) & {
        check_output(output, budget(), 1, state_);
        output.clear();
        if (state_ == StreamState::ended) { return Pull::end; }
        try {
            if (inputs_) {
                for (;;) {
                    const auto result = inputs_->source.next(inputs_->batch);
                    if (result == Pull::end) {
                        if (inputs_->batch.size() != 0) { throw std::logic_error("nonempty joined sort end"); }
                        break;
                    }
                    for (std::size_t i = 0; i < inputs_->batch.size(); ++i) {
                        sort_.append(inputs_->batch.row(i));
                    }
                }
                // Release the first sort, join, scans, snapshot, and input columns
                // before the second sort allocates its merge-reader buffers.
                inputs_.reset();
                sort_.finish();
            }
            const auto result = sort_.next(output);
            if (result == Pull::end) { state_ = StreamState::ended; }
            return result;
        } catch (...) {
            output.clear(); state_ = StreamState::failed;
            inputs_.reset(); sort_.abort(); throw;
        }
    }
    void retry_cleanup() noexcept { sort_.retry_cleanup(); }
private:
    std::optional<Inputs> inputs_;
    JoinedSortBuffer sort_;
    StreamState state_ = StreamState::ready;
};
}  // namespace strata::query
#endif
```

**Checkpoint.** Build the existing targets after this refactoring. All 23 earlier
tests must still pass, including event-run byte decoding. On a three-row joined
in-memory input, inspect that all five fields survive and that the region/day
prefix determines order. Do not temporarily reinterpret joined rows as events:
that discards precisely the dimension value whose snapshot lifetime we just
released.

The two sorts have separate disk/workspace/descriptor allowances. During second
run generation, first-sort runs can still be live. The shared root budget limits
covered simultaneous memory, but there is not yet one combined query-wide disk
or descriptor broker. Choose allowances that fit the machine and remember the
sink, database files, and standard runtime consume additional descriptors. The
recorded allocation ledger is not an operating-system resident-memory cap.

