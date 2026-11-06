# Chapter 15 — External sorting with bounded fan-in

## Part 1 — Turn one workspace into owned runs

The storage cursor returns events in event-ID order. The next query stage needs
all events for the same account together. A vector containing the entire query
would make that reordering easy, but would also let input size determine memory
use. Sorting each batch independently is not enough: account 2 in the last batch
must precede account 9 in the first batch.

We will sort bounded chunks, write each sorted chunk as a temporary **run**, and
merge only as many runs as the configured buffers and file-descriptor allowance
permit. More runs require additional passes. The final pass produces batches
directly; it does not write a final file merely to read it back immediately.

> **Learning objectives**
> Derive a real merge fan-in from simultaneously live resources; give every
> temporary name an owner before writing bytes; encode and validate bounded run
> frames; implement the merge heap rather than delegate the external algorithm;
> preserve input files until replacement verification finishes; distinguish
> execution failure from cleanup failure; preserve empty-batch progress and
> terminal-error semantics when composing with the query pipeline.

**Prerequisites:** the complete code through all three parts of
[Chapter 14](https://25d-228.github.io/cplusplus-learning/chapter-14.html).
Keep the twenty existing CTest entries. Chapter 6 supplies explicit codecs and
complete file transfers, Chapter 9 supplies the allocation ledger, Chapter 12
supplies owning allocators and the earlier heap reasoning, and Chapter 14 supplies
`EventBatch`, `Pull`, `StreamState`, and `check_output`. No prerequisite file is
replaced in this chapter; only `CMakeLists.txt` is extended.

**Implementation allocation: 6.0 hours for all three parts together.** The work
split remains 1.5 hours for run generation, 2.25 for buffered multi-pass merging,
1.0 for ownership and failure paths, and 1.25 for integration, tests, and diagnosis.
Reading is separate. This is still synchronous: no worker threads, cooperative
cancellation, join, or aggregation is introduced here.

Create `include/strata/query/external_sort.hpp`, `src/query/external_sort.cpp`,
`tests/external_sort.cpp`, and `scripts/check_sort_fixture.py`. Concatenate the
numbered segments for each filename in reading order across all three parts.
A namespace or class may deliberately continue into the next segment. The
complete files compile together after Part 3; do not compile an unfinished part.

[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-15-part-2.html) completes
the run reader, writer, merge state machine, and pull adapter.
[Part 3](https://25d-228.github.io/cplusplus-learning/chapter-15-part-3.html) supplies
the complete tests, independent decoder, and build extension.

## Specify the order before allocating anything

The primary comparison is `(account_id, event_id)`. An actual visible event scan
has one row per event ID, so this resolves ties in the intended query. The reusable
sort also accepts repeated input values. To make differing payloads deterministic,
compare day and cents after those first two fields. Exactly equal records remain
interchangeable; we do not promise original-input stability for duplicates.

For example, write a row as `(event, account, day, cents)`. The rows
`(90,2,3,100)`, `(10,1,4,200)`, and `(5,2,1,300)` become
`(10,1,4,200)`, `(5,2,1,300)`, `(90,2,3,100)`. Event 90 is not first merely because
its storage position was encountered first. No field is projected away.

The comparison uses unsigned fields and lexicographic comparison, not subtraction
that could wrap at an endpoint. It supplies the strict weak ordering required by
[standard sorting algorithms](https://eel.is/c++draft/alg.sorting.general).
Chapter 5's distinction remains important: a well-formed callable alone does not
prove this semantic property.

## Expose limits, measurements, and a move-only facade

The public options name **different** limits. `workspace_bytes` caps this sort's
modeled workspace; the shared `Budget` also charges upstream, downstream, and
storage allocations. `disk_bytes` caps simultaneous expected run lengths, not
physical filesystem blocks. `file_descriptors` is an allowance for open run files;
the temporary-directory descriptor and all other application descriptors are extra.

`max_runs` bounds initial-run metadata. The implementation has 128 run slots and
32 merge-reader slots, all accounted as fixed state. It rejects input above
`chunk_rows * max_runs`; it does not quietly allocate a vector of arbitrary run
descriptors. Defaults allow at most 524,288 input rows, subject to the other limits.
The hard chunk cap is 1,048,576 rows, and block capacity is 1 through 4,096 rows.
These are explicit scale limits, not a claim to sort an unbounded stream with a
finite namespace. An on-disk run catalog would be a different extension.

`SortStats` is a caller-retainable observation object. Create it with
`make_sort_stats`, and treat its fields as read-only while the sorter exists. A
report is single-use: reusing or resetting live counters would invalidate admission
arithmetic. Its shared ownership makes destructor cleanup errors observable after
the facade has disappeared. Statistics storage is charged to the same root budget,
but is outside the sort-local workspace calculation.

Start the header with the imports, options, report, hook, and facade. The template
adapter is appended in Part 2 before the final namespace and header-guard closure.

```cpp title="include/strata/query/external_sort.hpp — complete file, segment 1"
#ifndef STRATA_QUERY_EXTERNAL_SORT_HPP
#define STRATA_QUERY_EXTERNAL_SORT_HPP

#include "strata/query/scan.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <memory>
#include <string_view>
#include <utility>

namespace strata::query {

struct EventOrder {
    bool operator()(const Event& a, const Event& b) const noexcept;
};
struct SortOptions {
    std::size_t chunk_rows = 4096;
    std::size_t block_rows = 256;
    std::size_t max_runs = 128;
    std::size_t file_descriptors = 17;  // Run descriptors only; directory is extra.
    std::size_t workspace_bytes = 2 * 1024 * 1024;
    std::size_t stack_allowance = 64 * 1024;
    std::uint64_t disk_bytes = 256 * 1024 * 1024;
};
struct SortStats {
    std::uint64_t input_rows = 0, output_rows = 0;
    std::uint64_t initial_runs = 0, merge_passes = 0, merge_groups = 0;
    std::uint64_t files_created = 0, files_removed = 0, completed_files = 0;
    std::uint64_t read_bytes = 0, write_bytes = 0;
    std::uint64_t disk_reserved = 0, peak_disk_reserved = 0;
    std::size_t live_files = 0, open_run_files = 0, peak_open_run_files = 0;
    std::size_t fixed_charge = 0, peak_workspace = 0, fan_in = 0;
    std::size_t cleanup_errors = 0;
    int first_cleanup_error = 0;
    bool directory_live = false, used = false, saturated = false;
    std::array<char, 512> directory{};
};
std::shared_ptr<SortStats> make_sort_stats(std::shared_ptr<memory::Budget> budget);

enum class SpillEvent { before_write, before_read, completed_run, before_unlink };
struct SpillHook {
    void (*call)(void*, SpillEvent, const char*, std::uint64_t, std::uint64_t) = nullptr;
    void* argument = nullptr;
};
namespace sort_detail { class Core; }

class SortBuffer {
public:
    SortBuffer(std::shared_ptr<memory::Budget> budget, SortOptions options,
               std::string_view temporary_parent, std::shared_ptr<SortStats> stats,
               SpillHook hook = {});
    SortBuffer(const SortBuffer&) = delete;
    SortBuffer& operator=(const SortBuffer&) = delete;
    SortBuffer(SortBuffer&&) noexcept = default;
    SortBuffer& operator=(SortBuffer&&) noexcept = default;
    void append(Event event) &;
    void finish() &;
    Pull next(EventBatch& output) &;
    void abort() noexcept;
    void retry_cleanup() noexcept;
    std::shared_ptr<memory::Budget> budget() const noexcept;
private:
    std::shared_ptr<sort_detail::Core> core_;
};
```

`sort_detail::Core` is a **forward declaration**: it introduces a class name without
its layout. The facade can store a `shared_ptr` to that incomplete class. The
factory in Part 2 constructs the complete object with the owning allocator; clients
never construct or copy it directly. Moving `SortBuffer` transfers a handle rather
than relocating internal objects that borrow each other's addresses. Use the
move destination; only destroy or reassign the source.

The hook is a plain function pointer plus a borrowed argument pointer. It is not a
scheduler or a callback ownership system. A non-null hook argument must outlive
the sorter, including its destructor. Hooks run synchronously and must not reenter
this sorter, mutate its statistics, or capture owning cycles. The tests use them
to inject errors or change a just-completed fixture file. Normal callers pass `{}`.

`append` is the run-generation boundary. `finish` seals the input and prepares
sorted output. `next` requires a sealed input. A wrong output ledger or zero
capacity is rejected before clearing or consuming anything. Failures during actual
sorting or reading make the sorter terminal; `abort` releases temporary work but
does not turn it back into an accepting sorter.

Now begin the implementation file. `std::tie` creates tuples of references used
only during this comparison call; no reference is retained. The statistics factory
uses Chapter 12's owning allocator so its control storage can safely outlive the
caller's original budget handle. The allocator-aware shared-creation mechanism is
specified in [shared-pointer creation](https://eel.is/c++draft/util.smartptr.shared.create).

```cpp title="src/query/external_sort.cpp — complete file, segment 1"
#include "strata/query/external_sort.hpp"
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
std::shared_ptr<SortStats> make_sort_stats(std::shared_ptr<memory::Budget> budget) {
    return std::allocate_shared<SortStats>(memory::SharedAllocator<SortStats>{std::move(budget)});
}
```

## Encode values, not C++ object padding

Runs have their own format. They are not immutable database tables and never
appear in a manifest. Their only row schema is the projected event's four numeric
fields. A row occupies 28 encoded bytes: event ID at byte 0, account ID at byte 8,
day at byte 16, and cents at byte 20. Multi-byte values use the existing explicit
little-endian helpers. `sizeof(Event)` is used for memory accounting, never for
the persistent row width.

| Region | Byte layout |
|---|---|
| Header, 36 bytes | Eight-byte magic; version, row width, block-row capacity, and order ID as four `u32` values; total rows as `u64`; CRC-32 as `u32` |
| Data frame | Row count `u32`; payload bytes `u32`; that many 28-byte rows; CRC-32 `u32` |
| Footer, 24 bytes | Eight-byte tail magic; total rows `u64`; frame count `u32`; CRC-32 `u32` |

Each checksum covers its region except the final checksum field. All full frames
contain the configured number of rows; only the last may be partial. The order ID
is 1 for the comparison just defined. A reader checks that the header agrees with
its trusted configuration and run descriptor; it never allocates according to an
unchecked file-controlled capacity.

Let $n>0$ be a run's row count and $b>0$ its configured rows per frame. The number
of frames is

$$
q(n,b) = 1 + \left\lfloor\frac{n-1}{b}\right\rfloor.
$$

This computes a ceiling without adding $b-1$ to $n$. Each row contributes 28 bytes,
each frame adds 12, and the header plus footer contributes 60. Therefore the exact
finished file length is

$$
L(n,b) = 60 + 28n + 12q(n,b).
$$

For three rows and two rows per frame, the frames occupy 68 and 40 bytes. The
whole file is $36+68+40+24=168$ bytes. A one-row run is 100 bytes; a twelve-row run
with the same frame capacity is $60+336+72=468$ bytes. Those numbers later drive
both disk admission and the independent decoder.

The hard input caps imply at most $2^{27}$ rows in any run. Products in this
length calculation fit both unsigned 64-bit arithmetic and the signed file-offset
limit, and its frame count fits the footer's 32-bit field. Empty input produces
no run at all. Append the constants and complete row codecs:

```cpp title="src/query/external_sort.cpp — complete file, segment 2"
namespace sort_detail {
using storage::table_format::crc32;
constexpr std::size_t max_runs = 128, max_fan_in = 32;
constexpr std::size_t header_bytes = 36, footer_bytes = 24, row_bytes = 28;
constexpr std::array<std::byte, 8> magic{
    std::byte{'S'}, std::byte{'T'}, std::byte{'R'}, std::byte{'R'},
    std::byte{'U'}, std::byte{'N'}, std::byte{'0'}, std::byte{'1'}};
constexpr std::array<std::byte, 8> tail_magic{
    std::byte{'S'}, std::byte{'T'}, std::byte{'R'}, std::byte{'E'},
    std::byte{'N'}, std::byte{'D'}, std::byte{'0'}, std::byte{'1'}};

void require(bool condition, const char* message) {
    if (!condition) { throw FormatError(message); }
}
std::uint64_t run_bytes(std::uint64_t rows, std::size_t block_rows) {
    if (rows == 0 || rows > (std::uint64_t{1} << 27) || block_rows == 0 || block_rows > 4096) {
        throw std::length_error("run row limits");
    }
    const auto blocks = 1 + (rows - 1) / block_rows;
    // The validated caps make these products fit u64 and signed file offsets.
    return header_bytes + rows * row_bytes + blocks * 12 + footer_bytes;
}
Event decode_row(std::span<const std::byte> row) {
    require(row.size() == row_bytes, "run row width");
    return {get_le<8>(row.first(8)), get_le<8>(row.subspan(8, 8)),
            static_cast<std::uint32_t>(get_le<4>(row.subspan(16, 4))),
            get_le<8>(row.subspan(20, 8))};
}
void encode_row(Event event, std::span<std::byte> row) {
    put_le<8>(row.first(8), event.event_id);
    put_le<8>(row.subspan(8, 8), event.account_id);
    put_le<4>(row.subspan(16, 4), event.day);
    put_le<8>(row.subspan(20, 8), event.amount_cents);
}
```

CRC-32 is the already implemented accidental-corruption check from Chapter 8, not
a cryptographic authentication scheme. Its presence does not justify ignoring
row counts, exact file lengths, order, or semantic field widths. A sortedness error
with a recomputed checksum must still be rejected.

## Exact workspace storage and explicit object lifetime

A vector's capacity can exceed a request, and `shrink_to_fit` is non-binding; see
[vector capacity](https://eel.is/c++draft/vector.capacity). For this chapter's fixed
workspaces, an exact typed allocation is simpler to account for. `Scratch<T>` owns
exactly `count` elements through `SharedAllocator<T>`. We use it only for `Event`
and `std::byte`, whose default construction and destruction do not throw.

This is not a general exception-safe replacement for `std::vector`. The static
assertions deliberately reject types whose construction would require tracking a
partially initialized prefix. `construct_at` begins each element's lifetime;
`destroy_n` ends those lifetimes before deallocation. These are distinct operations,
as specified by [construction](https://eel.is/c++draft/specialized.construct) and
[destruction](https://eel.is/c++draft/specialized.destroy).

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
```

`allocate` checks `count * sizeof(T)` in the allocator before obtaining storage.
Consequently the same multiplication in `bytes()` is valid for a successfully
constructed owner. Every element is value-initialized once. Only a separate visible
count determines how many initialized `Event` slots currently contain input rows.

The type is noncopyable and immovable. Later `optional<Scratch<T>>` members use
`emplace` to construct it in its final location and `reset` to destroy it. Those
operations do not require moving the contained object. Because the enclosing core
is stable behind its facade, a movable scratch buffer is unnecessary.

## Model the peak, not just the final run

Define the following quantities, all in bytes except the row and descriptor counts:

* $H$ is the actual charged core/control allocation, including fixed run slots,
  pending-name slots, reader objects, the heap array, and fixed path storage.
* $S$ is an explicit stack/library-scratch allowance; $C$ is chunk-row capacity;
  $w=\mathrm{sizeof}(\texttt{Event})$ is the in-memory row stride.
* $F=12+28b$ is one full encoded-frame buffer; $W$ is the sort-local workspace cap;
  $D$ is the run-file descriptor allowance; $k$ is merge fan-in.

Generation owns a chunk and an output frame. Merging releases the chunk and owns
$k$ input frames plus that output frame. The two phases do not need their largest
buffers simultaneously. Their required modeled peak is

$$
M(k)=H+S+\max(Cw+F,(k+1)F).
$$

At least two inputs and one output are necessary to reduce the run count. Reject
before consuming input unless

$$
W\ge H+S+\max(Cw+F,3F).
$$

After reserving fixed state, the allowance, and one output buffer, legal fan-in is

$$
k=\min\!\left(32,\ D-1,\left\lfloor\frac{W-H-S-F}{F}\right\rfloor\right).
$$

The subtraction is performed only after the minimum-progress check. The resulting
$k$ must be at least two; otherwise a merge loop could repeatedly produce as many
runs as it consumed. Descriptor limits and byte limits are independent: abundant
memory does not justify opening too many files.

For the tested Linux ABI, $H=14984$, $S=16384$, $C=3$, $w=32$, and $b=2$, giving
$F=68$. Generation uses 31,532 modeled bytes; a two-input merge uses 31,572. A
workspace cap of 31,571 is one byte too small for the minimum merge, even though
three input records alone are tiny. These are observed ABI-specific values, not
macOS layout requirements. The test reads $H$ from a probe before checking exact
byte thresholds on the current build.

`H` is measured around synchronous core construction **before** dynamic buffers
are initialized. The same technique is not safe attribution amid unrelated
concurrent allocations; the later concurrency integration must replace that
assumption. Both actual arrays and the stack reservation use the common ledger;
the workspace formula is a separate admission calculation, not a second charge
for the same arrays.

The $S$ reservation is a policy allowance, not a portable proof about
`std::sort`'s internal memory. The standard specifies $O(n\log n)$ comparisons
but not a byte-exact auxiliary-space bound; see [sort](https://eel.is/c++draft/sort).
No execution-policy overload is used. The ledger does not intercept library or
operating-system allocations, measure stack pages, or cap process resident memory.
A tighter physical-memory contract would require controlling or separately
instrumenting that implementation rather than renaming the allowance a guarantee.

The upstream filter and projection also remain alive. With upstream capacity $u$
and consumer capacity $v$, their scan batches, the sort's input batch, and the
consumer allocate $36u+36u+28u+28v=100u+28v$ column bytes. At $u=2,v=3$, that is
284 bytes **in addition to** this workspace, snapshot readers, cache frames, and
statistics. All share one root ledger. A locally feasible sort can still fail
because another owner legitimately holds the remaining global headroom.

## Separate a run's name from its open file handle

A run may spend most of its life closed. Its move-only owner therefore retains a
slot in `Space`, not a permanently open descriptor. A reader or writer owns the
shorter-lived `File` handle. `Run::reset` retires the name; `Space` either removes
it or keeps a bounded pending-cleanup record.

This is the ownership order: the stable core owns `Space`, then run owners, then
readers. Readers borrow their run owner, and run owners borrow `Space`. Cleanup
must close readers before retiring names. Moving the facade does not move any of
these borrowed objects. C++ initializes members in declaration order, not the
visual order chosen for constructor arguments; see
[member initialization](https://eel.is/c++draft/class.base.init).

```cpp title="src/query/external_sort.cpp — complete file, segment 4"
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

`std::exchange` empties the source's pointer while obtaining its previous value.
The slot index alone is not ownership: a null `space_` means that this handle has
nothing to retire. Assignment retires an old destination before taking the new
slot. Self-assignment leaves ownership unchanged.

The namespace has 129 pending slots: at most 128 input names plus one replacement
being prepared. A multi-pass merge retires each completed group's inputs before
starting its next group. Failed cleanup may occupy slots longer; new admission
then fails rather than overflowing the registry. The two arrays of 128 run owners
in the core represent current and next passes, not 256 independently admitted
initial inputs.

Here is the complete `Space` declaration. Native descriptors use Chapter 11's
small `Descriptor` owner; data transfers use Chapter 6's `File` boundary.

```cpp title="src/query/external_sort.cpp — complete file, segment 5"
class Space {
public:
    struct Slot {
        std::uint64_t id = 0, rows = 0, bytes = 0;
        bool created = false, complete = false, retired = false;
    };
    Space(SortOptions options, std::string_view parent, std::shared_ptr<SortStats> stats,
          SpillHook hook) : options_(options), stats_(std::move(stats)), hook_(hook) {
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
    std::array<char, 512> pattern_{};
    io::Descriptor directory_;
    std::array<Slot, max_runs + 1> slots_{};
    std::uint64_t next_id_ = 1;
};
```

The copied parent path uses fixed storage and rejects embedded NULs or excessive
length. No path allocation grows with the number of runs. Names use a monotonic ID
inside this one private temporary directory; they are unrelated to database file
IDs. `Slot::created` distinguishes owning a reserved length from owning an actual
filesystem name. An exclusive-create failure must never authorize deleting a
preexisting name.

Append reset, bounded name formatting, and nonthrowing error observation:

```cpp title="src/query/external_sort.cpp — complete file, segment 6"
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
```

A destructor cannot sensibly throw while another exception is unwinding. The
report records the first cleanup error code and a saturating count of attempts
that failed. Multiple retries may contribute multiple errors for one file; that
counter is not a count of distinct leaked files. `live_files` and `disk_reserved`
show what has not yet been confirmed absent.

## Reserve coexistence before creating a file

The directory is created lazily: the no-spill path never creates one. `mkdtemp`
requires writable template storage and creates a private directory; Apple's
[archived manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/mkdtemp.3.html)
describes its mode and mutation of the template. The code passes the writable
fixed array, not a string literal. Use a trusted parent directory. This is not
protection against a hostile process renaming ancestors during the operation.

`reserve` charges the **whole expected output length before creation**, while all
input lengths remain charged. This is a logical allowance, not an operating-system
space reservation or a promise that the filesystem will accept writes. Real or
injected `ENOSPC` still terminates the query.

```cpp title="src/query/external_sort.cpp — complete file, segment 7"
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
    const auto bytes = run_bytes(rows, options_.block_rows);
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

For nine initial runs containing 25 rows at chunk capacity three and frame capacity
two, the initial files total $8(168)+100=1444$ bytes. During the third materialized
pass, a twelve-row run and another twelve-row run coexist with a one-row carried
run: $468+468+100=1036$. Preparing their 24-row replacement requires another
$L(24,2)=876$, so the peak is $1036+876=1912$ bytes. The eventual two final inputs
occupy only $876+100=976$ bytes. Sizing disk allowance for either 976 or 1444 alone
would accept an impossible transition.

A failed partial write still retains the entire expected-length charge until its
name is removed. This is conservative and intentional. Returning that charge
merely because fewer bytes were successfully written would allow repeated failed
outputs to accumulate outside the model.

## Open relative to the owned directory, then validate

Run creation uses `O_CREAT | O_EXCL`; opening existing input uses no creation flag.
`O_NOFOLLOW` refuses a final-component symbolic link and `O_CLOEXEC` keeps these
handles from ordinary exec inheritance. `openat` resolves the short run name
relative to the retained directory descriptor. These meanings are documented in
[the open/openat interface](https://man.openbsd.org/open.2); no OpenBSD-only flag is
used. Native macOS uses the corresponding interfaces, not a Linux-only syscall.

```cpp title="src/query/external_sort.cpp — complete file, segment 8"
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
```

The successful creation marks the slot before returning its `File`. If later
header construction or writing fails, the already established run owner can retire
the name. A failed `openat` leaves `created` false, so cleanup releases only the
reservation. `opened` counts an acquired run handle, not a directory handle.

Opening input requires a completed, nonretired run. It checks regular-file type,
a single hard link, and the exact forecast length before exposing data. The
native handle is counted even if `fstat` or a validation check then fails.

```cpp title="src/query/external_sort.cpp — complete file, segment 9"
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

`close_checked` consumes the descriptor even on an error, as established in Chapter
6; decrement the count once before calling it. The destructor path catches and
reports an error instead of retrying a possibly reused descriptor number. There
is no polling loop for descriptor exhaustion: the caller's allowance is checked,
and a lower operating-system limit may still cause a normal open error.

## Retirement is not silent recursive deletion

A retired slot remains in `Space` until its owned name has been removed or is
already absent. A failed removal preserves both the slot and its disk allowance.
`retry` only visits retired entries, never live inputs. It does not list or delete
unknown files. An unexpected foreign entry may prevent removing the directory;
that failure is reported rather than answered with recursive deletion.

```cpp title="src/query/external_sort.cpp — complete file, segment 10"
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

`files_removed` counts owned names whose absence was confirmed, including `ENOENT`
when a fault hook already removed one. It is not a measurement of disk blocks
freed. Unlinking a name while a file is still open does not immediately release
all associated storage, as Apple's
[unlink manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/unlink.2.html)
explains. Our ordering closes all input handles before name retirement, and
[unlinkat](https://man.openbsd.org/unlink.2) keeps that retirement relative to the
owned directory.

Normal execution may succeed even if cleanup fails. Keep the report, inspect the
live counters, and call `retry_cleanup` while the facade still exists. Destruction
makes another bounded attempt and leaves the path in the report when cleanup
cannot finish. After the sorter is destroyed, that report is diagnostic information,
not a surviving retry service. Do not delete another live query's directory.

Process termination skips these destructors. Old private directories may therefore
remain after a crash; this chapter neither adopts them nor implements an automatic
startup scavenger. These are disposable query files, not recovery authority.

## Meter successful transfers, including failed attempts' prefixes

Each complete transfer goes through `File`, which already handles partial I/O and
interruption. The wrapper differences its byte counter on both success and failure.
A call that throws after successfully writing a prefix contributes that prefix.
The before-operation fault hook runs first; a hook failure itself transferred no
bytes in that operation. Earlier successful operations remain counted.

```cpp title="src/query/external_sort.cpp — complete file, segment 11"
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

A completed file means the writer finished its framing and checked close. The
`completed_run` hook executes afterward. It can still throw, leaving the local run
owner responsible for cleanup. Intermediate replacement verification in Part 2
is a further step before old inputs can retire.

These counters are bytes transferred by the file API. They include headers,
footers, framing, and explicit verification rereads. They do not measure physical
device traffic, page-cache misses, throughput, or durable writes. The run writer
will intentionally omit `fsync`: none of these temporary files is a committed
database generation.

Continue with [Part 2 — Merge only what fits](https://25d-228.github.io/cplusplus-learning/chapter-15-part-2.html).
