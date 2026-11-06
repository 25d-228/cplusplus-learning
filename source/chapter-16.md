# Chapter 16 — A streaming many-to-one merge join

## Part 1 — One account value, many events

A sorted event stream is not yet a regional result. It contains account IDs, not
regions. An event whose account was deleted must disappear from an inner join;
an event whose account changed region must use the region from the **same read
view** as the event. Looking up each event in a newly captured account snapshot
would violate that requirement even when every individual lookup was correct.

We will merge Chapter 15's event stream with an account scan from one captured
`ReadView`. The account relation can be much larger than its input buffer. A hot
account can match thousands of events without growing a group vector: retain one
copied account row and reuse it until the event key moves forward.

> **Learning objectives**
> Derive the three ordered comparison cases; implement a resumable many-to-one
> merge; distinguish copied row state from borrowed batch storage; enforce input
> ordering and dimension uniqueness; preserve empty progress; account for
> overlapping pipeline buffers; and connect terminal failure to upstream spill
> and snapshot cleanup.

**Prerequisites:** the complete code through all three parts of
[Chapter 15](https://25d-228.github.io/cplusplus-learning/chapter-15.html), including
[Chapter 14's](https://25d-228.github.io/cplusplus-learning/chapter-14.html) owning
batches and pull protocol and
[Chapter 12's](https://25d-228.github.io/cplusplus-learning/chapter-12.html) owning
read views. Keep the twenty-two existing CTest entries. We reuse `EventOrder`,
`ExternalSort`, `EventScan`, `AccountScan`, `Column`, `check_output`, and the shared
budget; we do not replace their implementations.

**Implementation allocation: 5.0 hours for all three parts together.** Allocate
1.5 hours to the state machine, 1.5 to streaming integration, 0.75 to budget and
lifetime behavior, and 1.25 to boundaries, skew, and failure checks. Reading is
separate. This chapter remains synchronous. Grouping, an additional sort order,
query-result publication, and cancellation are not introduced here.

Create `include/strata/query/merge_join.hpp`, `src/query/merge_join.cpp`, and
`tests/merge_join.cpp`. Numbered segments for each filename concatenate in reading
order across these three parts. The header deliberately stays open until Part 2;
the implementation namespace closes there too. Part 3 supplies the full test
program and a small CMake append. No prior source file changes except that append.

[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-16-part-2.html)
implements head loading, advancement, failure, and the same-view factory.
[Part 3](https://25d-228.github.io/cplusplus-learning/chapter-16-part-3.html)
provides the runnable checks.

## State the multiplicity before writing the loop

An **inner join** emits a row for each matching pair, not one row for each distinct
key. The PostgreSQL manual's
[inner-join definition](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)
states that matching-pair interpretation. Our narrower relation contract says
that the visible accounts stream has exactly one row per present account ID.
Consequently, an input event produces either one joined row or none.

Let $N$ be the number of events after filtering, let $k_i$ be event $i$'s account
ID, and let $K$ be the set of visible account IDs in the captured view. Define
$\mathbf{1}[P]$ as one when proposition $P$ is true and zero otherwise. The number
of output rows is

$$
J=\sum_{i=0}^{N-1}\mathbf{1}[k_i\in K],\qquad 0\leq J\leq N.
$$

For event account IDs `(10,10,20,99)` and visible accounts `(10,20)`, the four
indicator values are `(1,1,1,0)`, so $J=3$. The two events for account 10 both
survive. This is not deduplication or aggregation. Exact duplicate fact rows also
retain their multiplicity in the reusable operator, although an ordinary visible
event scan produces one row per event primary key.

The accounts side is different. Seeing account 10 twice violates the operator's
unique-visible-key contract, even when both copies have the same region. We
reject the stream instead of multiplying output rows. A general many-to-many
merge join must handle the cross-product of equal-key groups; PostgreSQL's
[merge-join implementation notes](https://doxygen.postgresql.org/nodeMergejoin_8c_source.html)
explain why repeated keys on both sides can require restoring an earlier input
position. The implementation here needs no such replay because the dimension
key is unique. It has no null values, outer-join padding, arbitrary predicates,
or implicit foreign-key enforcement. Missing accounts simply omit events.

## Use order to prove which row can be discarded

The left input is ordered lexicographically by
`(account_id, event_id, day, amount_cents)`, exactly as Chapter 15's `EventOrder`
specifies. The first component is sufficient for joining; the remaining fields
make output order deterministic. The right input is strictly increasing by
`account_id`. Both use unsigned comparisons, not subtraction that might wrap.

Write $e$ for the current event account ID and $a$ for the current account ID.
Only three comparisons are possible:

| Comparison | What the ordering proves | State change |
|---|---|---|
| $e<a$ | All remaining accounts have keys at least $a$; this event cannot match. | Discard the event and count it as unmatched. |
| $e>a$ | All remaining events have keys at least $e$; this account cannot match another event. | Discard the account. Keep the event. |
| $e=a$ | The unique account supplies this event's region. | Emit a copied joined row; discard the event, **keep the account**. |

The last action is the essential many-to-one rule. Advancing both sides after a
match would lose the second event for account 10. Advancing neither side would
emit the same pair forever. Clearing only the event head makes one unit of
progress while preserving a possible next match.

There is also a useful loop invariant. Every discarded event has either been
emitted exactly once or proved unmatched. Every discarded account is too small
for the remaining events, or the event input has ended. A current head is an
owned value that has been loaded but not yet discarded. Rows before the input
batch index have already been copied into a head at most once; they are not
loaded again after an output pause.

Let $A$ be the number of visible account rows. Each comparison discards at least
one previously undiscarded input row, so the number of comparisons is bounded by
$N+A$. Tail validation also visits each row at most once. Excluding upstream
sorting, decoding, and empty-progress calls, the merge work is $O(N+A)$ and its
current-row state is $O(1)$. The bound depends on ordered inputs, not on uniform
key frequencies. A single account matching almost every event does not require
memory proportional to that group.

## Walk across an output boundary

Use event notation `(event_id, account_id, day, cents)`. The event input is
`(9,1,3,5)`, `(100,10,3,500)`, `(101,10,3,250)`, `(102,20,3,700)`, and
`(103,99,3,50)`. The accounts are `(10,1)`, `(20,2)`, and `(30,4)`, where the
second field is a region. Give each input buffer one row and the output two rows.

| Comparison number | Current event/account keys | Action | Saved account afterward |
|---:|---|---|---|
| 1 | 1 / 10 | Discard event 9. | 10 → region 1 |
| 2 | 10 / 10 | Emit event 100 in region 1. | 10 → region 1 |
| 3 | 10 / 10 | Emit event 101; output is full. Return. | 10 → region 1 |
| 4 | 20 / 10 | On the next call, discard account 10. | None |
| 5 | 20 / 20 | Emit event 102 in region 2. | 20 → region 2 |
| 6 | 99 / 20 | Discard account 20. | None |
| 7 | 99 / 30 | Discard account 30. | None |

After comparison 7, the account source reports end. Event 103 is unmatched;
there is no eighth key comparison. When both inputs end, the second output batch
contains event 102. The next call returns empty `Pull::end`.

The first output call asked the account source for only one batch, despite
matching two events and refilling the event buffer three times. That observation
will be an assertion, not just a diagram. It catches a state-machine defect that
can remain entirely within valid memory addresses.

## Make the joined output own all five fields

A joined result keeps the complete `Event` and adds `region_id`. Event ID remains
useful as a deterministic tie-breaker; account ID remains useful for diagnosis;
day, cents, and region are needed by the next query stages. The batch stores
columns rather than an array of padded `JoinedRow` objects.

Compose an `EventBatch` with one `Column<uint32_t>`. There is only one visible row
count, inside `events_`. `size()` delegates to it; a second independently updated
count would create a new consistency invariant for no benefit. Begin the header:

```cpp title="include/strata/query/merge_join.hpp — complete file, segment 1"
#ifndef STRATA_QUERY_MERGE_JOIN_HPP
#define STRATA_QUERY_MERGE_JOIN_HPP

#include "strata/query/external_sort.hpp"
#include "strata/query/operators.hpp"

#include <cstddef>
#include <cstdint>
#include <limits>
#include <memory>
#include <optional>
#include <span>
#include <stdexcept>
#include <string_view>
#include <utility>

namespace strata::query {

struct JoinedRow {
    Event event;
    std::uint32_t region_id;
    bool operator==(const JoinedRow&) const = default;
};
class JoinedBatch {
public:
    static constexpr std::size_t row_bytes = EventBatch::row_bytes + sizeof(std::uint32_t);
    JoinedBatch(std::shared_ptr<memory::Budget> budget, std::size_t capacity);
    JoinedBatch(const JoinedBatch&) = delete;
    JoinedBatch& operator=(const JoinedBatch&) = delete;
    JoinedBatch(JoinedBatch&&) noexcept = default;
    JoinedBatch& operator=(JoinedBatch&&) noexcept = default;
    void swap(JoinedBatch& other) noexcept;
    void replace_capacity(std::size_t capacity);
    void clear() noexcept { events_.clear(); }
    void append(JoinedRow row);
    JoinedRow row(std::size_t index) const;
    const EventBatch& events() const & noexcept { return events_; }
    const EventBatch& events() const && = delete;
    std::span<const std::uint32_t> regions() const & noexcept {
        return regions_.storage().first(size());
    }
    std::span<const std::uint32_t> regions() const && = delete;
    std::size_t size() const noexcept { return events_.size(); }
    std::size_t capacity() const noexcept { return events_.capacity(); }
    std::size_t capacity_bytes() const noexcept { return capacity() * row_bytes; }
    std::shared_ptr<memory::Budget> budget() const noexcept { return events_.budget(); }
private:
    EventBatch events_;
    Column<std::uint32_t> regions_;
};
```

The defaulted move operations are appropriate **because** both members already
implement the required ownership behavior. Moving `EventBatch` empties its source
count and capacities; moving `Column` transfers its allocation with the allocator
that owns the corresponding budget. There is no additional scalar count here
that a memberwise move would accidentally leave nonzero. This is different from
default-moving a type with a raw owning pointer or an independent count.

The `events()` and `regions()` accessors expose read-only borrows, not ownership.
As in Chapter 14, treat them as expired at successful mutation, refill, move,
swap, capacity replacement, or destruction. `row(i)` returns a copied value.
The rvalue-deleted accessors prevent directly borrowing from a temporary batch;
they cannot prove every caller obeys the remaining lifetime contract.

A joined column row requires three 64-bit fields and two 32-bit fields:

$$
w_J=3\cdot8+2\cdot4=32\ \text{bytes}.
$$

This is the sum of column element widths, not `sizeof(JoinedRow)`. Its `Event`
subobject can have padding. A capacity-two output therefore requests 64 element
bytes through five allocations, regardless of the aggregate row object's padding.

## Prepare replacement storage before touching the old batch

Start `src/query/merge_join.cpp` with a checked capacity helper, construction,
swap, and replacement. The namespace remains open for the following segments:

```cpp title="src/query/merge_join.cpp — complete file, segment 1"
#include "strata/query/merge_join.hpp"

namespace strata::query {
namespace {
std::size_t joined_capacity(std::size_t capacity) {
    if (capacity == 0) { throw std::invalid_argument("zero joined batch capacity"); }
    (void)checked_size_mul(capacity, JoinedBatch::row_bytes);
    return capacity;
}
}  // namespace

JoinedBatch::JoinedBatch(std::shared_ptr<memory::Budget> budget, std::size_t capacity)
    : events_(budget, joined_capacity(capacity)), regions_(std::move(budget), capacity) {}
void JoinedBatch::swap(JoinedBatch& other) noexcept {
    events_.swap(other.events_); regions_.swap(other.regions_);
}
void JoinedBatch::replace_capacity(std::size_t capacity) {
    JoinedBatch candidate{budget(), capacity}; swap(candidate);
}
```

`joined_capacity` checks the **whole** column footprint before the first
allocation. Relying only on each column's multiplication would not establish
that their sum fits `size_t`. The helper returns the original capacity; it is
called while initializing `events_`, before the region column is constructed.
An exception from that later column destroys the already constructed event
columns automatically.

`replace_capacity` does not reserve more space while retaining rows. It installs
an empty batch of the requested capacity after a complete candidate exists.
Failure leaves the original rows, borrowed addresses, and charges intact. For a
capacity-two old batch and a capacity-three candidate, old and new storage can
coexist at a peak of $64+96=160$ column bytes. Releasing old storage first would
reduce that transient requirement but abandon the strong replacement guarantee.

Now append and read rows:

```cpp title="src/query/merge_join.cpp — complete file, segment 2"
void JoinedBatch::append(JoinedRow row) {
    if (size() == capacity()) { throw std::length_error("joined batch full"); }
    regions_.storage()[size()] = row.region_id;
    events_.append(row.event);  // One shared visible count, published last.
}
JoinedRow JoinedBatch::row(std::size_t index) const {
    const auto event = events_.row(index);  // Bounds check precedes region access.
    return {event, regions_.storage()[index]};
}
```

The full-batch check precedes either write. `regions_[size()]` is initialized
first; `events_.append` writes the event scalars and increments the shared visible
count last. Its capacity precondition is already established, and these unsigned
assignments do not throw. A reader cannot observe an extra event row lacking a
region. This argument would need revisiting for throwing element types; our
existing `ColumnNumber` deliberately excludes them.

For `row(index)`, the event lookup checks bounds before the region access. A
moved-from batch has zero capacity and size; appending directly is rejected, but
`replace_capacity` can make it usable again because its moved-from column
allocators retain valid budget owners. No raw pointer is exposed for ownership
transfer.

## Name three different capacities

`event_rows` and `account_rows` configure the two input batches. The caller's
`JoinedBatch` determines output capacity independently. `work_rows` is a bound
on iterations of the join state machine per pull; it is not a memory allowance
or an end-to-end latency promise.

The statistics below are copied observations. Saturation prevents counters from
wrapping and falsely appearing small. It does not stop a valid query. In
particular, `matched` counts rows staged internally; a later error can discard
those rows from the current output batch. It is not an output-commit counter.
Continue the header:

```cpp title="include/strata/query/merge_join.hpp — complete file, segment 2"
struct JoinOptions {
    std::size_t event_rows = 256;
    std::size_t account_rows = 256;
    std::size_t work_rows = 1024;  // At most this many join transitions per next().
};
struct JoinStats {
    std::uint64_t events = 0, accounts = 0, comparisons = 0;
    std::uint64_t matched = 0, unmatched = 0, progress_returns = 0;
    bool saturated = false;
};
namespace join_detail {
JoinOptions checked_options(JoinOptions options);
std::shared_ptr<memory::Budget> common_budget(
    std::shared_ptr<memory::Budget> events, const std::shared_ptr<memory::Budget>& accounts);
inline void increment(std::uint64_t& value, bool& saturated) noexcept {
    if (value == std::numeric_limits<std::uint64_t>::max()) { saturated = true; }
    else { ++value; }
}
}  // namespace join_detail
```

The post-increment in `increment` executes only below the maximum value. At the
maximum, the value stays there and `saturated` becomes true. Interpret equalities
between counters only when that flag is false.

The two non-inline helpers belong in the implementation file. `common_budget`
compares the ledger objects, not two equal numeric limits. Giving each side a
separate 8 MiB ledger would permit their sum to exceed an intended shared 8 MiB
allowance. It would also make downstream allocation ownership harder to follow.

```cpp title="src/query/merge_join.cpp — complete file, segment 3"
namespace join_detail {
JoinOptions checked_options(JoinOptions options) {
    if (options.event_rows == 0 || options.account_rows == 0 || options.work_rows == 0) {
        throw std::invalid_argument("zero join capacity or work limit");
    }
    (void)checked_size_add(checked_size_mul(options.event_rows, EventBatch::row_bytes),
                           checked_size_mul(options.account_rows, AccountBatch::row_bytes));
    return options;
}
std::shared_ptr<memory::Budget> common_budget(std::shared_ptr<memory::Budget> events,
    const std::shared_ptr<memory::Budget>& accounts) {
    if (!events || events.get() != accounts.get()) {
        throw std::invalid_argument("join sources need one shared budget");
    }
    return events;
}
}  // namespace join_detail
```

These checks establish construction preconditions, not a guarantee that every
future upstream request will fit. Other live readers and snapshots share the
same ledger. A locally valid sort or join can still encounter `BudgetExceeded`.
The default capacities are choices, not minimum hardware requirements.

Notice a limitation of the budget check: two independent snapshots from the
same database naturally share one ledger. Equality of budget pointers does not
prove that two sources describe the same committed boundary. The factory in
Part 2 will establish that property by construction.

## Keep input lifetime in one optional owner

The complete class declaration follows. `Inputs` is a nested class holding both
upstream sources, both reusable batches, and their next-unloaded indices. The
outer `std::optional<Inputs>` holds that object **inline**. `std::in_place` tells
the optional to construct its contained `Inputs` directly from the following
arguments; it does not ask for another heap allocation.

Why use an optional for an object that always exists after successful
construction? Completion or failure should destroy the entire upstream ownership
chain immediately, while leaving the join object available to answer an end or
failed-state check. According to the
[`optional::reset` contract](https://eel.is/c++draft/optional.mod), resetting an
engaged optional destroys its contained object and disengages it.

The `requires` clause checks the two source interfaces through Chapter 14's
`PullSource` concept. It checks callable expressions and return types, not that
rows are sorted, keys are unique, or an empty batch represents real progress;
that distinction follows the standard's
[requires-expression rules](https://eel.is/c++draft/expr.prim.req).

```cpp title="include/strata/query/merge_join.hpp — complete file, segment 3"
template<class Events, class Accounts>
    requires PullSource<Events, EventBatch> && PullSource<Accounts, AccountBatch>
class MergeJoin {
    struct Inputs {
        Events left;
        Accounts right;
        EventBatch events;
        AccountBatch accounts;
        std::size_t event_at = 0, account_at = 0;
        Inputs(Events e, Accounts a, std::shared_ptr<memory::Budget> budget, JoinOptions options)
            : left(std::move(e)), right(std::move(a)), events(budget, options.event_rows),
              accounts(std::move(budget), options.account_rows) {}
    };
    enum class Head { row, progress, end };
public:
    MergeJoin(Events events, Accounts accounts, JoinOptions options = {})
        : budget_(join_detail::common_budget(events.budget(), accounts.budget())),
          options_(join_detail::checked_options(options)),
          inputs_(std::in_place, std::move(events), std::move(accounts), budget_, options_) {}
    MergeJoin(const MergeJoin&) = delete;
    MergeJoin& operator=(const MergeJoin&) = delete;
    MergeJoin(MergeJoin&&) = default;
    MergeJoin& operator=(MergeJoin&&) = default;
    std::shared_ptr<memory::Budget> budget() const noexcept { return budget_; }
    JoinStats stats() const noexcept { return stats_; }
    StreamState state() const noexcept { return state_; }
    Pull next(JoinedBatch& output) &;
private:
    Head event_head();
    Head account_head();
    void release_inputs() noexcept {
        event_.reset(); account_.reset();
        previous_event_.reset(); previous_account_.reset();
        inputs_.reset();  // Destroys upstream plans, cursors, and spill owners.
    }
    void count(std::uint64_t& value) noexcept { join_detail::increment(value, stats_.saturated); }
    std::shared_ptr<memory::Budget> budget_;
    JoinOptions options_;
    std::optional<Inputs> inputs_;
    std::optional<Event> event_, previous_event_;
    std::optional<Account> account_, previous_account_;
    bool event_end_ = false, account_end_ = false;
    StreamState state_ = StreamState::ready;
    JoinStats stats_{};
};
```

Member declaration order is intentional. `budget_` and validated `options_`
exist before `inputs_` construction. A constructor exception destroys completed
members and parameters, including any sources already moved into the operation.
The arguments are ownership transfers by value: rejection does not promise to
restore the caller's moved-from source object.

`event_` and `account_` own scalar copies of the current heads. Their `previous_`
partners retain the last loaded rows for ordering checks even when a head has
been discarded. No pointer refers back into `Inputs` or into an upstream batch.
That lets a paused join move without retaining a pointer to its former address.
Use only destruction or reassignment on a moved-from join. The stronger
reusable-empty promise of the batch types does not extend to the whole pipeline.

`release_inputs` first clears visible and validation heads, then destroys the
source chain. A sorted source's destructor releases its final merge readers and
spill owners; account/event scans release cursor and read-view handles. The
separate `budget_` member keeps output checks usable after that teardown. Every
source must obey the existing nonthrowing-cleanup contract.

At this point the types and ownership graph are complete, but the header still
needs its three member-function definitions and factory declaration. Continue
with [Part 2](https://25d-228.github.io/cplusplus-learning/chapter-16-part-2.html)
before compiling.
