# Chapter 17 — Grouping, checked aggregates, and complete queries

## Part 2 — Keep one group; publish one complete result

Continue from [Part 1](https://25d-228.github.io/cplusplus-learning/chapter-17.html).
Its joined sorter preserves account, amount, day, event ID, and region while
placing each region/day group contiguously. Now the aggregate can be smaller
than either sort: one unfinished group, one previous row for order validation,
and a bounded input batch. The output batch owns every completed group.

## Count and sum are one state change

For group $g$, let $I_g$ be the occurrences of joined rows with key $g$. Define
its count $n_g$ and total cents $s_g$ by

$$
n_g=|I_g|,\qquad s_g=\sum_{i\in I_g} v_i.
$$

The empty stream has no groups. It does **not** emit `(0,0,0,0)`. A group with
zero total is legitimate when it contains one or more zero-cent events. Both
outputs use `uint64_t`; the keys use `uint32_t`. Their maximum total is
$M=2^{64}-1=18446744073709551615$.

Before incorporating amount $v$, require $n_g<M$ and $v\le M-s_g$. Only then can
we install $(n_g+1,s_g+v)$. Reusing Chapter 6's checked addition twice computes
the two candidate values before either assignment. For example, count 2 and
sum $M$ can accept another zero-cent event, but not a one-cent event. On the
latter failure, count must remain 2, not 3.

Unsigned wraparound is defined modulo a power of two by the
[C++ fundamental type rules](https://eel.is/c++draft/basic.fundamental), but it is
not an acceptable financial total. An unsigned-overflow bug may produce no
undefined-behavior sanitizer report. This chapter deliberately rejects overflow
instead of saturating, silently wrapping, or switching to floating point.

## Own the output columns

`GroupRow` is an owning scalar value. `GroupBatch` contains four independently
charged arrays: region and day at four bytes each, count and cents at eight
bytes each. Capacity $O$ therefore costs $24O$ column bytes, irrespective of
`sizeof(GroupRow)` or its padding. `rows_` identifies initialized visible entries,
not allocated capacity. The member functions follow the existing batch contracts.

```cpp title="include/strata/query/aggregate.hpp — complete file, segment 1"
#ifndef STRATA_QUERY_AGGREGATE_HPP
#define STRATA_QUERY_AGGREGATE_HPP

#include "strata/query/joined_sort.hpp"

#include <cstddef>
#include <cstdint>
#include <memory>
#include <optional>
#include <stdexcept>
#include <utility>

namespace strata::query {
struct GroupRow {
    std::uint32_t region_id = 0, day = 0;
    std::uint64_t event_count = 0, total_cents = 0;
    bool operator==(const GroupRow&) const = default;
};
// Both totals commit together; either failed addition leaves the whole row unchanged.
void add_event(GroupRow& group, std::uint64_t cents);

class GroupBatch {
public:
    static constexpr std::size_t row_bytes = 24;
    GroupBatch(std::shared_ptr<memory::Budget> budget, std::size_t capacity);
    GroupBatch(const GroupBatch&) = delete;
    GroupBatch& operator=(const GroupBatch&) = delete;
    GroupBatch(GroupBatch&& other) noexcept;
    GroupBatch& operator=(GroupBatch&& other) noexcept;
    void swap(GroupBatch& other) noexcept;
    void replace_capacity(std::size_t capacity);
    void clear() noexcept { rows_ = 0; }
    void append(GroupRow row);
    GroupRow row(std::size_t index) const;
    std::size_t size() const noexcept { return rows_; }
    std::size_t capacity() const noexcept { return capacity_; }
    std::size_t capacity_bytes() const noexcept { return capacity_ * row_bytes; }
    std::shared_ptr<memory::Budget> budget() const noexcept { return regions_.budget(); }
private:
    std::size_t capacity_, rows_ = 0;
    Column<std::uint32_t> regions_, days_;
    Column<std::uint64_t> counts_, totals_;
};
struct AggregateOptions {
    std::size_t input_rows = 256;
    std::size_t work_rows = 4096;
};
AggregateOptions checked_aggregate_options(AggregateOptions options);
```

The implementation validates capacity before allocating. Construction failure
unwinds already constructed columns. Move construction exchanges scalar lengths
with zero and transfers columns with their matching budget owners. Move assignment
uses a temporary and nonthrowing swap so the old allocations are released under
their original owners. Self-move is a no-op. As for prior batches, a moved-from
batch is reusable only after assigning valid storage; it is not a detached
budget factory.

```cpp title="src/query/aggregate.cpp — complete file, segment 1"
#include "strata/query/aggregate.hpp"

namespace strata::query {
namespace {
std::size_t group_capacity(std::size_t capacity) {
    if (capacity == 0) { throw std::invalid_argument("zero group batch capacity"); }
    (void)checked_size_mul(capacity, GroupBatch::row_bytes);
    return capacity;
}
}  // namespace
void add_event(GroupRow& group, std::uint64_t cents) {
    const auto count = checked_u64_add(group.event_count, 1);
    const auto total = checked_u64_add(group.total_cents, cents);
    group.event_count = count;
    group.total_cents = total;
}
GroupBatch::GroupBatch(std::shared_ptr<memory::Budget> budget, std::size_t capacity)
    : capacity_(group_capacity(capacity)), regions_(budget, capacity_), days_(budget, capacity_),
      counts_(budget, capacity_), totals_(std::move(budget), capacity_) {}
GroupBatch::GroupBatch(GroupBatch&& other) noexcept
    : capacity_(std::exchange(other.capacity_, 0)), rows_(std::exchange(other.rows_, 0)),
      regions_(std::move(other.regions_)), days_(std::move(other.days_)),
      counts_(std::move(other.counts_)), totals_(std::move(other.totals_)) {}
GroupBatch& GroupBatch::operator=(GroupBatch&& other) noexcept {
    if (this != &other) { GroupBatch candidate{std::move(other)}; swap(candidate); }
    return *this;
}
void GroupBatch::swap(GroupBatch& other) noexcept {
    using std::swap;
    swap(capacity_, other.capacity_); swap(rows_, other.rows_);
    regions_.swap(other.regions_); days_.swap(other.days_);
    counts_.swap(other.counts_); totals_.swap(other.totals_);
}
```

A replacement creates all four new arrays before swapping. A failed replacement
therefore preserves the old rows and charge. A successful replacement intentionally
clears the logical contents. `append` writes all four scalar fields before
publishing the increased row count. This publication is a synchronous invariant,
not an atomic operation for multiple threads.

```cpp title="src/query/aggregate.cpp — complete file, segment 2"
void GroupBatch::replace_capacity(std::size_t capacity) {
    GroupBatch candidate{budget(), capacity}; swap(candidate);
}
void GroupBatch::append(GroupRow row) {
    if (rows_ == capacity_) { throw std::length_error("group batch full"); }
    if (row.event_count == 0) { throw std::invalid_argument("empty group row"); }
    regions_.storage()[rows_] = row.region_id; days_.storage()[rows_] = row.day;
    counts_.storage()[rows_] = row.event_count; totals_.storage()[rows_] = row.total_cents;
    ++rows_;
}
GroupRow GroupBatch::row(std::size_t index) const {
    if (index >= rows_) { throw std::out_of_range("group batch row"); }
    return {regions_.storage()[index], days_.storage()[index],
            counts_.storage()[index], totals_.storage()[index]};
}
AggregateOptions checked_aggregate_options(AggregateOptions options) {
    if (options.input_rows == 0 || options.work_rows == 0) {
        throw std::invalid_argument("zero aggregate capacity or work limit");
    }
    (void)checked_size_mul(options.input_rows, JoinedBatch::row_bytes);
    return options;
}
}  // namespace strata::query
```

## Make boundaries explicit

The aggregation inputs own the upstream source and a joined batch. `current_`
exists only after seeing a first row. `previous_` validates the complete joined
ordering across refills. Neither optional grows with the number of rows or groups.
The copied previous row is a validation aid, not a second accumulator.

A positive work quota limits rows consumed **inside this loop** per call. It can
return an empty progress batch in the middle of a large group. It cannot limit
the time an upstream sort takes to produce its next batch. The entire pipeline
is still synchronous and noncancellable.

```cpp title="include/strata/query/aggregate.hpp — complete file, segment 2"
template<class Source> requires PullSource<Source, JoinedBatch>
class GroupAggregate {
    struct Inputs {
        Source source;
        JoinedBatch batch;
        std::size_t at = 0;
        Inputs(Source value, std::size_t capacity)
            : source(std::move(value)), batch(source.budget(), capacity) {}
    };
public:
    GroupAggregate(Source source, AggregateOptions options = {})
        : budget_(source.budget()), options_(checked_aggregate_options(options)),
          inputs_(std::in_place, std::move(source), options_.input_rows) {}
    GroupAggregate(const GroupAggregate&) = delete;
    GroupAggregate& operator=(const GroupAggregate&) = delete;
    GroupAggregate(GroupAggregate&&) = default;
    GroupAggregate& operator=(GroupAggregate&&) = default;
    std::shared_ptr<memory::Budget> budget() const noexcept { return budget_; }
    Pull next(GroupBatch& output) &;
private:
    std::shared_ptr<memory::Budget> budget_;
    AggregateOptions options_;
    std::optional<Inputs> inputs_;
    std::optional<JoinedRow> previous_;
    std::optional<GroupRow> current_;
    StreamState state_ = StreamState::ready;
};
```

Consider sorted group keys `A,A,B,B,C` and output capacity one. On reading the
first `B`, emit completed `A`, initialize `B`, and incorporate that same boundary
row before returning. The next call resumes with one `B` already counted. If
instead the cursor advanced without retaining that row, the first event of every
later group would disappear.

The converse error is failing to consume the boundary row: it would be included
twice. The source index advances exactly once, after `add_event` succeeds. On
failure the whole stream is terminal, so no recovery path needs to guess whether
a partly processed row should be replayed.

At EOF there is no next key to trigger a transition. Explicitly emit `current_`
once, release the upstream owner, and mark ended. If that emission returns a
nonempty batch, the following call returns empty EOF. An empty upstream **batch**
must not flush `current_`; the same group can continue afterward.

```cpp title="include/strata/query/aggregate.hpp — complete file, segment 3"
template<class Source> requires PullSource<Source, JoinedBatch>
Pull GroupAggregate<Source>::next(GroupBatch& output) & {
    check_output(output, budget_, 1, state_);
    output.clear();
    if (state_ == StreamState::ended) { return Pull::end; }
    try {
        for (std::size_t work = 0; work < options_.work_rows; ++work) {
            auto& input = *inputs_;
            if (input.at == input.batch.size()) {
                const auto result = input.source.next(input.batch);
                input.at = 0;
                if (result == Pull::end) {
                    if (input.batch.size() != 0) { throw std::logic_error("nonempty aggregate end"); }
                    if (current_) { output.append(*current_); current_.reset(); }
                    inputs_.reset(); previous_.reset(); state_ = StreamState::ended;
                    return output.size() == 0 ? Pull::end : Pull::batch;
                }
                if (input.batch.size() == 0) { return Pull::batch; }
            }
            const auto value = input.batch.row(input.at);  // Own before any refill.
            if (previous_ && JoinedOrder{}(value, *previous_)) {
                throw FormatError("aggregate input ordering");
            }
            if (current_ && (current_->region_id != value.region_id || current_->day != value.event.day)) {
                output.append(*current_); current_.reset();
            }
            if (!current_) { current_ = GroupRow{value.region_id, value.event.day, 0, 0}; }
            add_event(*current_, value.event.amount_cents);
            previous_ = value; ++input.at;
            // The boundary row already belongs to the NEXT group when this is full.
            if (output.size() == output.capacity()) { return Pull::batch; }
        }
        return Pull::batch;  // A giant group may make progress without emitting yet.
    } catch (...) {
        output.clear(); state_ = StreamState::failed;
        inputs_.reset(); previous_.reset(); current_.reset(); throw;
    }
}
}  // namespace strata::query
#endif
```

The `input` reference is borrowed from the optional. The EOF branch destroys that
optional, then immediately returns without using `input` again. Similarly, the
catch path never reads it after cleanup. `value` is copied before any source
refill. These details make early destruction compatible with ordinary references.

An order error, refill exception, or overflow clears **this call's** output,
releases upstream sources, and sets terminal failure. An earlier, separate owning
batch remains readable. Thus a caller can observe a valid prefix before learning
that the query failed. A sink must not confuse that prefix with a complete answer.
The preflight `check_output` sits outside the `try`: a foreign-budget output or
already-failed stream rejects before altering the caller's sentinel batch.

## What remains live at each phase?

Let $U$ be the upstream scan/filter/project batch capacity, $E$ the join's event
input capacity, $A$ its account input capacity, $J$ the second sort's joined-input
capacity, $G$ the aggregate input capacity, and $O$ its output capacity. During
second-sort generation the owning column arrays can total

$$
100U+28E+12A+32J+32G+24O.
$$

The $100U$ is Chapter 16's filter input (36), projection input (36), and first-sort
input (28) per row. It excludes both sort cores, row chunks, frame buffers,
metadata, snapshots, and the database. Add those actual charged allocations;
do not assume a “100-byte row” is a universal pipeline footprint.

After second-sort generation, the join/scans/first-sort columns disappear. The
aggregate needs $32G+24O$ column bytes plus the second sort's workspace and its
constant scalar state. Its own storage is $O(G+O)$, independent of the size of
one group and the total number of groups. At valid EOF it releases the second
sort too. Output batches, caller-held sort statistics, and any caller-held
snapshot can still keep charges alive. Standard exception storage, test oracles,
allocator metadata, and kernel caches are not all part of this ledger.

## The sink contract separates acceptance from completion

`begin` establishes a fresh result attempt. `append` accepts a complete group,
not the query. `finish` publishes successful completion only after valid EOF.
`abort` is idempotent and nonthrowing. A generic consumer may be unable to retract
accepted rows; it must still identify them as a failed query prefix.

A crucial implementer contract is that **`finish` must not throw after committing
publication**. Otherwise a caller could report failure after the completed
artifact was already installed. Our CSV sink performs all throwing work before
rename succeeds, then only assigns scalar state. Hooks may throw before the
named boundary, never after publication.

The sink below uses a trusted, stable private output directory and one validated
basename. It is not a sandbox for hostile directory replacement, a shared-directory
security service, or a cross-filesystem transaction. The filename limits are
explicit fixed capacities, not guessed filesystem limits. The OS may reject a
name even below our 512-byte path storage bound.

```cpp title="include/strata/query/result.hpp — complete file, segment 1"
#ifndef STRATA_QUERY_RESULT_HPP
#define STRATA_QUERY_RESULT_HPP

#include "strata/query/aggregate.hpp"
#include "strata/io/file.hpp"

#include <array>
#include <cstdint>
#include <string_view>

namespace strata::query {
class ResultSink {
public:
    virtual ~ResultSink() = default;
    virtual void begin() = 0;
    virtual void append(GroupRow row) = 0;
    virtual void finish() = 0;
    virtual void abort() noexcept = 0;
};
enum class ResultEvent { before_write, before_close, before_publish, before_unlink };
struct ResultHook {
    void (*call)(void*, ResultEvent) = nullptr;
    void* argument = nullptr;
};
class CsvResult final : public ResultSink {
public:
    CsvResult(std::string_view parent, std::string_view name, ResultHook hook = {});
    ~CsvResult() noexcept override { abort(); }
    CsvResult(const CsvResult&) = delete;
    CsvResult& operator=(const CsvResult&) = delete;
    CsvResult(CsvResult&&) = delete;
    CsvResult& operator=(CsvResult&&) = delete;
    void begin() override;
    void append(GroupRow row) override;
    void finish() override;
    void abort() noexcept override;
    void retry_cleanup() noexcept;
    bool published() const noexcept { return state_ == State::published; }
    bool temporary_live() const noexcept { return temporary_live_; }
    const char* temporary_path() const noexcept { return temporary_.data(); }
    int cleanup_error() const noexcept { return cleanup_error_; }
private:
    enum class State { fresh, open, published, aborted };
    void require_open() const;
    void write_text(std::string_view text);
    void hook(ResultEvent event) { if (hook_.call) { hook_.call(hook_.argument, event); } }
    void record_cleanup_error(int error) noexcept { if (cleanup_error_ == 0) { cleanup_error_ = error; } }
    std::array<char, 512> final_{}, temporary_{};
    io::File file_;
    ResultHook hook_;
    std::uint64_t offset_ = 0;
    State state_ = State::fresh;
    bool temporary_live_ = false;
    int cleanup_error_ = 0;
};
```

`consume_results` owns its plan by value. On return or unwinding, that ownership
releases the pipeline even when the caller has not manually walked every source.
Its `staged_groups` counter advances only after a sink accepts a row. It does not
count bytes, and a sink write can fail after transferring only part of a line.
All accepted rows remain provisional until `finish` succeeds.

```cpp title="include/strata/query/result.hpp — complete file, segment 2"
template<class Plan> requires PullSource<Plan, GroupBatch>
void consume_results(Plan plan, std::size_t output_rows, ResultSink& sink,
                     std::uint64_t& staged_groups) {
    try {
        GroupBatch output{plan.budget(), output_rows};
        sink.begin();
        for (;;) {
            const auto result = plan.next(output);
            if (result == Pull::end) {
                if (output.size() != 0) { throw std::logic_error("nonempty result end"); }
                break;
            }
            for (std::size_t i = 0; i < output.size(); ++i) {
                const auto next_count = checked_u64_add(staged_groups, 1);
                sink.append(output.row(i));
                staged_groups = next_count;
            }
        }
        sink.finish();  // The only publication boundary, after successful EOF.
    } catch (...) { sink.abort(); throw; }
}
}  // namespace strata::query
#endif
```

## Write under a temporary name, not the final one

The sink constructs two paths in its fixed arrays: the requested final basename
and a same-parent `.strata-result-XXXXXX` template. A caller must not choose that
temporary prefix as its result basename. No filesystem operation occurs in the
constructor. The already completed destination, if any, remains untouched while
the new query runs.

`mkstemp` creates and opens a unique file in one operation and modifies its writable
template. Apple's archived [mkstemp manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/mkstemp.3.html)
describes the returned descriptor and creation mode 0600. This avoids a separate
existence-check/open sequence. Use a private output parent as well; safe temporary
creation does not secure an attacker-controlled path prefix. The archive is an
API reference, not a claim about the installed macOS version. Inspect `man 3
mkstemp` locally when checking that platform.

```cpp title="src/query/result.cpp — complete file, segment 1"
#include "strata/query/result.hpp"
#include "strata/io/directory.hpp"

#include <algorithm>
#include <cerrno>
#include <cstdio>
#include <cstdlib>
#include <fcntl.h>
#include <span>
#include <stdexcept>
#include <system_error>
#include <unistd.h>

namespace strata::query {
CsvResult::CsvResult(std::string_view parent, std::string_view name, ResultHook hook)
    : hook_(hook) {
    constexpr std::string_view temporary_name = "/.strata-result-XXXXXX";
    if (parent.empty() || parent.find('\0') != std::string_view::npos || name.empty() ||
        name == "." || name == ".." || name.find('/') != std::string_view::npos ||
        name.find('\0') != std::string_view::npos || name.starts_with(".strata-result-") ||
        parent.size() >= final_.size() || name.size() >= final_.size() - parent.size() - 1 ||
        parent.size() >= temporary_.size() - temporary_name.size()) {
        throw std::invalid_argument("result parent or basename");
    }
    std::copy(parent.begin(), parent.end(), final_.begin());
    final_[parent.size()] = '/';
    std::copy(name.begin(), name.end(), final_.begin() + parent.size() + 1);
    std::copy(parent.begin(), parent.end(), temporary_.begin());
    std::copy(temporary_name.begin(), temporary_name.end(), temporary_.begin() + parent.size());
}
void CsvResult::require_open() const {
    if (state_ != State::open) { throw std::logic_error("result sink not open"); }
}
void CsvResult::write_text(std::string_view text) {
    require_open();
    const auto end = checked_file_end(offset_, text.size());
    hook(ResultEvent::before_write);
    file_.write_all(offset_, std::as_bytes(std::span{text.data(), text.size()}));
    offset_ = end;
}
```

Ownership of the name begins immediately after successful creation. A temporary
`Descriptor` guards the raw handle while close-on-exec is set; `File::adopt_native`
then becomes the sole handle owner. If header writing fails, abort attempts both
close and unlink. The existing partial-I/O loop writes every requested byte or
throws; an exception does not imply that nothing reached the temporary file.

```cpp title="src/query/result.cpp — complete file, segment 2"
void CsvResult::begin() {
    if (state_ != State::fresh) { throw std::logic_error("result sink is single-use"); }
    try {
        const int fd = ::mkstemp(temporary_.data());
        if (fd < 0) { throw std::system_error(errno, std::generic_category(), "create result temporary"); }
        temporary_live_ = true;
        io::Descriptor guard{fd};
        if (::fcntl(fd, F_SETFD, FD_CLOEXEC) != 0) {
            throw std::system_error(errno, std::generic_category(), "result close-on-exec");
        }
        file_ = io::File::adopt_native(guard.release());
        state_ = State::open;
        write_text("region_id,day,event_count,total_cents\n");
    } catch (...) { abort(); throw; }
}
```

Numeric CSV needs no quoting for these unsigned fields. The maximum row contains
two 10-digit and two 20-digit values, three commas, and one newline: 64 characters.
A 96-byte stack array leaves space for the terminator. The explicit
`unsigned long long` conversions match `%llu`; assuming `uint64_t` always aliases
that exact type would make a variadic format call nonportable.

```cpp title="src/query/result.cpp — complete file, segment 3"
void CsvResult::append(GroupRow row) {
    require_open();
    try {
        if (row.event_count == 0) { throw std::invalid_argument("empty CSV group"); }
        std::array<char, 96> line{};
        const int n = std::snprintf(line.data(), line.size(), "%llu,%llu,%llu,%llu\n",
            static_cast<unsigned long long>(row.region_id), static_cast<unsigned long long>(row.day),
            static_cast<unsigned long long>(row.event_count), static_cast<unsigned long long>(row.total_cents));
        if (n < 0 || static_cast<std::size_t>(n) >= line.size()) { throw std::length_error("CSV line width"); }
        write_text({line.data(), static_cast<std::size_t>(n)});
    } catch (...) { abort(); throw; }
}
```

`finish` closes before renaming. Both names are in the same parent, avoiding a
cross-filesystem rename request. A successful rename replaces an existing regular
result name without writing a partial new prefix into that old file; see Apple's
archived [rename manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/rename.2.html)
for the interface and filesystem constraint. The final name is the completion
boundary for this local synchronous tool.

This sink **does not synchronize file contents or the parent directory**. It is
not a durable database installation or a power-loss-safe result checkpoint.
Ordinary failures before publication preserve an existing final result in our
fault checks. A process exit can leave a temporary name. We neither adopt such a
name on restart nor claim it contains a complete answer. Keep this boundary
separate from Chapter 11's synchronized manifest protocol.

```cpp title="src/query/result.cpp — complete file, segment 4"
void CsvResult::finish() {
    require_open();
    try {
        hook(ResultEvent::before_close); file_.close_checked();
        hook(ResultEvent::before_publish);
        if (::rename(temporary_.data(), final_.data()) != 0) {
            throw std::system_error(errno, std::generic_category(), "publish result");
        }
        // No throwing operation follows successful rename.
        temporary_live_ = false; state_ = State::published;
    } catch (...) { abort(); throw; }
}
```

An aborted sink closes its descriptor and then attempts name removal. The cleanup functions record errors instead of throwing during unwinding.

```cpp title="src/query/result.cpp — complete file, segment 5"
void CsvResult::retry_cleanup() noexcept {
    if (!temporary_live_ || state_ != State::aborted) { return; }
    try {
        hook(ResultEvent::before_unlink);
        if (::unlink(temporary_.data()) != 0 && errno != ENOENT) {
            throw std::system_error(errno, std::generic_category(), "unlink result temporary");
        }
        temporary_live_ = false;
    } catch (const std::system_error& error) { record_cleanup_error(error.code().value()); }
    catch (...) { record_cleanup_error(EIO); }
}
void CsvResult::abort() noexcept {
    if (state_ == State::published) { return; }
    state_ = State::aborted;
    try { if (file_.is_open()) { file_.close_checked(); } }
    catch (const std::system_error& error) { record_cleanup_error(error.code().value()); }
    catch (...) { record_cleanup_error(EIO); }
    retry_cleanup();
}
}  // namespace strata::query
```

An unlink refusal retains `temporary_live_` and the path, records the first cleanup
error, and permits explicit retry after the cause is removed. Error history stays
visible after a later successful retry. Destruction tries cleanup again but cannot
promise absence when the OS continues to refuse deletion. A published file is
never unlinked by abort or destruction.

## Return an owning error, not a borrowed exception reference

The complete-query API returns a status plus an optional error handle. C++'s
`std::exception_ptr` owns a reference to an exception object; a default instance is
empty. Inside `catch (...)`, `std::current_exception()` captures the handled
exception, and `std::rethrow_exception` later throws through a nonempty handle.
The [exception propagation reference](https://eel.is/c++draft/propagation) specifies
the object's retained lifetime and the null-handle precondition. We need only
these long-established facilities, not newer features in that evolving draft.

This is already useful without threads: the driver can inspect counters before
printing the original diagnostic. `current_exception()` is nonthrowing, but its
implementation can substitute a memory-allocation failure exception when capturing
requires unavailable storage. Do not present exception storage as a charged,
allocation-free byte string or retain a reference to a caught stack variable.

```cpp title="include/strata/query/complete_query.hpp — complete file, segment 1"
#ifndef STRATA_QUERY_COMPLETE_QUERY_HPP
#define STRATA_QUERY_COMPLETE_QUERY_HPP

#include "strata/query/result.hpp"

#include <exception>
#include <memory>
#include <string_view>

namespace strata::query {
struct QueryOptions {
    DayRange days{0, 0};
    std::size_t upstream_rows = 256;
    JoinOptions join{};
    SortOptions event_sort{}, group_sort{};
    std::size_t joined_rows = 256;
    AggregateOptions aggregate{};
    std::size_t output_rows = 128;
};
struct QueryHooks { SpillHook events{}, groups{}; };
using CompleteQuery = GroupAggregate<JoinedExternalSort<SnapshotJoin>>;
CompleteQuery make_query(storage::ReadView view, const QueryOptions& options,
    std::string_view temporary_parent, std::shared_ptr<SortStats> events,
    std::shared_ptr<SortStats> groups, QueryHooks hooks = {});

enum class QueryCompletion { failed, complete };
struct QueryReport {
    QueryCompletion completion = QueryCompletion::failed;
    std::uint64_t staged_groups = 0;
    std::exception_ptr error;
    std::shared_ptr<SortStats> event_sort, group_sort;
};
QueryReport execute_query(storage::ReadView view, const QueryOptions& options,
    std::string_view temporary_parent, ResultSink& sink, QueryHooks hooks = {}) noexcept;
}  // namespace strata::query
#endif
```

`make_query` receives one captured view; the join factory copies that one ownership
boundary into both scans. The first sort orders facts for joining. The second
sort receives joined rows, not a fresh database scan. Finally, `GroupAggregate`
consumes regional order. No source reopens a later snapshot between stages.

`execute_query` creates separate statistics owners, owns the resulting plan,
consumes it, and marks complete only after sink completion. Reports can keep those
statistics and the root ledger alive after files and input data are released.
Failure to create the report's second statistics object is handled just like a
later allocation failure. A caller's `db.snapshot()` executes **before** entering
this function; exceptions while acquiring that argument belong to the caller's
outer boundary, not the returned report.

```cpp title="src/query/complete_query.cpp — complete file, segment 1"
#include "strata/query/complete_query.hpp"

namespace strata::query {
CompleteQuery make_query(storage::ReadView view, const QueryOptions& options,
    std::string_view temporary_parent, std::shared_ptr<SortStats> events,
    std::shared_ptr<SortStats> groups, QueryHooks hooks) {
    if (options.output_rows == 0) { throw std::invalid_argument("zero query output capacity"); }
    (void)checked_size_mul(options.output_rows, GroupBatch::row_bytes);
    auto join = make_snapshot_join(std::move(view), options.days, options.upstream_rows,
        options.join, options.event_sort, temporary_parent, std::move(events), hooks.events);
    auto sorted = JoinedExternalSort{std::move(join), options.joined_rows, options.group_sort,
        temporary_parent, std::move(groups), hooks.groups};
    return CompleteQuery{std::move(sorted), options.aggregate};
}
QueryReport execute_query(storage::ReadView view, const QueryOptions& options,
    std::string_view temporary_parent, ResultSink& sink, QueryHooks hooks) noexcept {
    QueryReport report;
    try {
        const auto budget = view.budget_owner();
        report.event_sort = make_sort_stats(budget);
        report.group_sort = make_sort_stats(budget);
        auto plan = make_query(std::move(view), options, temporary_parent,
                               report.event_sort, report.group_sort, hooks);
        consume_results(std::move(plan), options.output_rows, sink, report.staged_groups);
        report.completion = QueryCompletion::complete;
    } catch (...) {
        report.error = std::current_exception();
        sink.abort();
    }
    return report;
}
}  // namespace strata::query
```

## A thin command-line driver

The driver accepts database, temporary parent, output parent, output basename,
and two day bounds. It does not parse SQL. Day values are unsigned 32-bit integers,
but the half-open upper bound may equal $2^{32}$ to include the last day.
`std::from_chars` returns both an error code and the first unconsumed position;
checking only the error would accept a prefix such as `2x`. The
[integer conversion specification](https://eel.is/c++draft/charconv.from.chars)
describes these two results. We require every character to be consumed and then
check the day domain and interval order.

The optional `tiny` preset forces three-row runs, two-row frames, two-way merge,
and one-group output for a small fixture. It is a mechanism test, not a production
performance preset. Both sorts retain the 128-run cap, so this preset rejects
more than 384 input rows per sort. Default chunk size 4096 allows at most 524,288
rows per sort with that same metadata cap. Increase explicit capacities only
when the shared budget and machine support them; there is no silent unlimited mode.

The driver opens the database for recovery and captures a snapshot using the
existing exclusive directory lock. It is **not a read-only attachment** to another
live process: snapshot capture can seal/rotate the active log. This is a local
single-process synchronous tool until the later runtime integration chapters.

```cpp title="tools/query.cpp — complete file, segment 1"
#include "strata/query/complete_query.hpp"
#include "strata/storage/database.hpp"

#include <charconv>
#include <cstdint>
#include <exception>
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string_view>
#include <system_error>

namespace {
std::uint64_t day_bound(std::string_view text) {
    std::uint64_t value = 0;
    const auto [end, error] = std::from_chars(text.data(), text.data() + text.size(), value);
    if (error != std::errc{} || end != text.data() + text.size() || value > (std::uint64_t{1} << 32)) {
        throw std::invalid_argument("day bound must be an integer in [0, 4294967296]");
    }
    return value;
}
}
int main(int argc, char** argv) {
    if (argc != 7 && argc != 8) {
        std::cerr << "usage: strata-query DATABASE TEMP_PARENT OUTPUT_PARENT OUTPUT_NAME DAY_BEGIN DAY_END [tiny]\n";
        return 2;
    }
    try {
        using namespace strata;
        using namespace strata::query;
        QueryOptions options; options.days = DayRange{day_bound(argv[5]), day_bound(argv[6])};
        if (argc == 8) {
            if (std::string_view{argv[7]} != "tiny") { throw std::invalid_argument("unknown query preset"); }
            options.upstream_rows = 2; options.join = {2, 2, 4};
            options.joined_rows = 2; options.aggregate = {2, 4}; options.output_rows = 1;
            for (auto* sort : {&options.event_sort, &options.group_sort}) {
                sort->chunk_rows = 3; sort->block_rows = 2; sort->file_descriptors = 3;
                sort->workspace_bytes = 128 * 1024; sort->stack_allowance = 16 * 1024;
            }
        }
        auto budget = std::make_shared<memory::Budget>(64 * 1024 * 1024);
        storage::Database db{budget, argv[1], storage::DatabaseOpen::recover};
        CsvResult sink{argv[3], argv[4]};
        const auto report = execute_query(db.snapshot(), options, argv[2], sink);
        if (report.completion != QueryCompletion::complete) {
            std::cerr << "query failed; staged_groups=" << report.staged_groups
                      << "; result was not published\n";
            if (sink.temporary_live()) { std::cerr << "temporary remains: " << sink.temporary_path() << '\n'; }
            if (report.error) { std::rethrow_exception(report.error); }
            return 1;
        }
        std::cout << "complete groups=" << report.staged_groups
                  << " event_runs=" << report.event_sort->initial_runs
                  << " group_runs=" << report.group_sort->initial_runs
                  << " peak_charged_bytes=" << budget->peak() << '\n';
        if (report.event_sort->cleanup_errors || report.group_sort->cleanup_errors || sink.cleanup_error()) {
            std::cerr << "result complete; temporary cleanup reported errors\n";
        }
        return 0;
    } catch (const std::exception& error) {
        std::cerr << error.what() << '\n'; return 1;
    }
}
```

The 64 MiB root is a covered-allocation limit, not a claim that process memory is
64 MiB. The reported peak includes database/query charges observed over this run;
it is neither isolated operator memory nor RSS. Both sort statistics separately
show whether spilling occurred. A complete result may still be accompanied by a
cleanup warning: an unused temporary file's deletion failure does not retroactively
change successfully computed values, but it must not disappear from diagnostics.

**Checkpoint.** Trace `(1,3,100,500)`, `(1,3,101,250)`, `(2,3,102,700)` through an
output capacity of one. The first return emits `(1,3,2,750)` and retains the last
row in the current group. EOF emits `(2,3,1,700)`, and the following call returns
empty EOF. Next replace the last group by amounts $M$ and 1: the earlier owning
batch is valid, the current output clears, the sink aborts, and no new complete
result may replace the old one. Part 3 executes these distinctions.

