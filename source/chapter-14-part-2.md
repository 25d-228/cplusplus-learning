# Chapter 14 — Owned batches and a streaming query pipeline

## Part 2 — Pull without confusing emptiness and completion

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-14.html) supplied the
owned columns. This part connects them to one retained snapshot and composes a
filter and projection. All code still belongs to the chapter's single **3.5-hour
implementation allocation**. Keep the storage engine and its earlier tests.

## Define a step, not an ambiguous row count

A consumer asks for one step with `next(output)`. The output owner belongs to the
consumer. The producer can replace its logical contents but cannot retain an
unannounced borrow into them. The result says whether the producer completed an
input step or reached permanent end:

| Result or event | Output after the call | May another valid call produce rows? |
|---|---|---|
| `Pull::batch` | Zero through capacity rows | Yes |
| `Pull::end` | Zero rows | No |
| Output capacity/budget rejected before work | Previous output unchanged | Yes, with a valid output |
| Exception during execution/refill | Zero rows; stream becomes failed | No; reconstruct the query |
| Call on an already failed stream | Rejected without work | No |

End is absorbing for **valid** calls. An ended producer still rejects an output
with the wrong ledger or inadequate capacity. A failed producer never turns an
error into a successful end result.

This differs from a row iterator, where no row can naturally mean end. A filter
must be able to say “I consumed three rows, selected none, and have not reached
end.” A consumer must examine `Pull`, not treat `output.size() == 0` as its stop
condition.

Start `include/strata/query/scan.hpp` with the protocol and its reusable preflight:

```cpp title="include/strata/query/scan.hpp — complete file, segment 1"
#ifndef STRATA_QUERY_SCAN_HPP
#define STRATA_QUERY_SCAN_HPP

#include "strata/query/batch.hpp"
#include "strata/storage/read_view.hpp"

#include <concepts>
#include <cstddef>
#include <memory>
#include <stdexcept>
#include <utility>

namespace strata::query {

enum class Pull { batch, end };
enum class StreamState { ready, ended, failed };

template<class Source, class Batch>
concept PullSource = requires(Source& source, Batch& output) {
    { source.next(output) } -> std::same_as<Pull>;
    { source.budget() } -> std::same_as<std::shared_ptr<memory::Budget>>;
};

template<class Batch>
void check_output(const Batch& output, const std::shared_ptr<memory::Budget>& budget,
                  std::size_t minimum, StreamState state) {
    if (state == StreamState::failed) { throw std::logic_error("query stream failed"); }
    if (output.capacity() < minimum || output.budget().get() != budget.get()) {
        throw std::invalid_argument("query output capacity or budget mismatch");
    }
}
```

`PullSource<Source, Batch>` applies Chapter 5's expression constraints to an
actual execution interface. The expressions in a
[`requires` expression](https://timsong-cpp.github.io/cppwp/n4861/expr.prim.req)
are checked without executing a query. The `-> std::same_as<Pull>` part requires
the exact result type of `next`; it does not call a conversion to a loosely
related boolean. `budget()` must return an owning handle by value.

The compiler cannot prove this protocol's semantic obligations. A type might
return `Pull::batch` forever without consuming input, retain a dangling reference,
or claim end with nonempty output. Our concrete sources make progress, own their
source dependencies, and honor end/failure behavior. The generic wrappers also
reject a nonempty end batch. They do not pretend that a syntactic concept is a
proof of lifetime or eventual termination.

`check_output` runs **before** clearing output or asking upstream for work. A
wrongly sized buffer therefore cannot cause a consumed row to disappear. It also
requires the exact same budget object as the source. A new uncharged ledger with
the same numeric limit is not an acceptable substitute.

## Retain the snapshot's actual ledger

A `ReadView` already owns a context, which owns the common budget. Add an accessor
rather than accept a second unrelated budget argument for a scan. This is the only
change to existing library interfaces in this chapter.

In `include/strata/storage/read_view.hpp`, inside the public section of `ReadView`,
add this declaration immediately after `ReadRetention retention() const;`:

```cpp title="include/strata/storage/read_view.hpp — add one declaration"
    std::shared_ptr<memory::Budget> budget_owner() const;
```

In `src/storage/read_view.cpp`, add this whole function before the final closing
brace of `namespace strata::storage`. Existing includes and definitions suffice;
do not replace any previous method:

```cpp title="src/storage/read_view.cpp — add budget_owner"
std::shared_ptr<memory::Budget> ReadView::budget_owner() const {
    require_valid();
    return state_->context->budget;
}
```

The returned shared pointer retains the existing ledger; it does not allocate a
new budget or a second budget control block. An empty/moved-from view is rejected
by `require_valid`. Query batches can now keep the ledger alive even after every
source owner has gone. They do **not** need to retain table files merely to retain
already decoded scalar rows.

## Separate storage borrowing from decoded values

A scan owns its `ReadView`, a budget handle, and eventually a `ReadCursor`. The
cursor is created lazily on the first pull, so constructing the plan does not yet
perform a storage scan. The retained view still fixes the query's read boundary.
Continue the header with both concrete source types:

```cpp title="include/strata/query/scan.hpp — complete file, segment 2"
ScannedEvent decode_visible_event(storage::EntryView entry);
Account decode_visible_account(storage::EntryView entry);

class EventScan {
public:
    explicit EventScan(storage::ReadView view)
        : view_(std::move(view)), budget_(view_.budget_owner()) {}
    EventScan(const EventScan&) = delete;
    EventScan& operator=(const EventScan&) = delete;
    EventScan(EventScan&&) noexcept = default;
    EventScan& operator=(EventScan&&) noexcept = default;
    Pull next(ScanBatch& output) &;
    std::shared_ptr<memory::Budget> budget() const noexcept { return budget_; }
private:
    storage::ReadView view_;
    std::shared_ptr<memory::Budget> budget_;
    storage::ReadCursor cursor_;
    StreamState state_ = StreamState::ready;
    bool started_ = false;
};

class AccountScan {
public:
    explicit AccountScan(storage::ReadView view)
        : view_(std::move(view)), budget_(view_.budget_owner()) {}
    AccountScan(const AccountScan&) = delete;
    AccountScan& operator=(const AccountScan&) = delete;
    AccountScan(AccountScan&&) noexcept = default;
    AccountScan& operator=(AccountScan&&) noexcept = default;
    Pull next(AccountBatch& output) &;
    std::shared_ptr<memory::Budget> budget() const noexcept { return budget_; }
private:
    storage::ReadView view_;
    std::shared_ptr<memory::Budget> budget_;
    storage::ReadCursor cursor_;
    StreamState state_ = StreamState::ready;
    bool started_ = false;
};

}  // namespace strata::query
#endif
```

The constructor accepts the view by value, then moves that handle into the source.
Passing an lvalue view copies its shared ownership; passing a temporary transfers
it. Neither captures a new database snapshot. The view member is initialized
before `budget_`, so the latter can safely obtain the same context's ledger.

`next(...) &` can be called only on an lvalue operator. A query is a state machine,
not a one-shot expression whose producer disappears immediately after a returned
borrow. The producer itself is movable but not copyable: copying a live cursor
would raise unanswered questions about whether consumption is shared or duplicated.
After moving an operator, use the destination; destroy or reassign the moved-from
operator rather than pulling it. Batch moves have the stronger reusable-empty
contract from Part 1.

Create `src/query/scan.cpp`. These two helpers turn a borrowed storage entry into
an owning value before any source advance:

```cpp title="src/query/scan.cpp — complete file, segment 1"
#include "strata/query/scan.hpp"

#include "strata/core/codec.hpp"

namespace strata::query {

ScannedEvent decode_visible_event(storage::EntryView entry) {
    if (entry.kind != storage::EntryKind::put || entry.key.sequence == 0 ||
        entry.key.user.relation != Relation::events) {
        throw FormatError("event scan requires a visible event put");
    }
    const auto event = decode_event(entry.value);
    if (key_of(event) != entry.key.user) { throw FormatError("event scan key mismatch"); }
    return {event, entry.key.sequence};
}
Account decode_visible_account(storage::EntryView entry) {
    if (entry.kind != storage::EntryKind::put || entry.key.sequence == 0 ||
        entry.key.user.relation != Relation::accounts) {
        throw FormatError("account scan requires a visible account put");
    }
    const auto account = decode_account(entry.value);
    if (key_of(account) != entry.key.user) { throw FormatError("account scan key mismatch"); }
    return account;
}
```

The byte codecs from Chapter 6 check exact length, version, and relation tag.
These wrappers additionally check that a visible row is a nonzero-sequence put
from the intended relation, and that its decoded primary key agrees with the
storage key. A tombstone is not an event containing zero cents. Storage's merged
cursor normally suppresses tombstones already, but an execution boundary should
not reinterpret a violated input contract as valid business data.

A returned `ScannedEvent` contains only integer values. Its lifetime does not
depend on `entry.value`, the cursor's owning `OwnedEntry`, or any cache pin that
was needed to assemble that entry. That is the precise point where a storage
borrow stops propagating into the query pipeline.

## Refill from one half-open relation range

Append the event scan:

```cpp title="src/query/scan.cpp — complete file, segment 2"
Pull EventScan::next(ScanBatch& output) & {
    check_output(output, budget_, 1, state_);  // Reject misuse before consuming anything.
    output.clear();
    if (state_ == StreamState::ended) { return Pull::end; }
    try {
        if (!started_) {
            cursor_ = view_.scan({Relation::events, 0}, Key{Relation::accounts, 0});
            started_ = true;
        }
        while (output.size() < output.capacity() && cursor_.valid()) {
            const auto row = decode_visible_event(cursor_.entry());
            output.append(row);  // All fields now belong to output, not to the cursor.
            cursor_.next();
        }
        if (!cursor_.valid()) { cursor_.reset(); state_ = StreamState::ended; }
        return output.size() == 0 ? Pull::end : Pull::batch;
    } catch (...) {
        output.clear(); cursor_.reset(); state_ = StreamState::failed;
        throw;
    }
}
```

The lower bound `{events, 0}` and exclusive upper bound `{accounts, 0}` cover all
event keys, including the largest possible 64-bit event ID, without computing
“largest ID plus one.” The source never performs mutable database point reads for
individual rows. Every row comes from the retained `ReadView`'s merged cursor.

Inside the loop, the order is deliberate:

```text title="One scan iteration"
borrow current entry -> decode owning scalars -> copy into output columns
                    -> advance cursor and expire the storage borrow
```

An output allocation failure cannot occur in `append`: capacity was acquired in
Part 1. Storage advancement can still allocate a new frame or fail while decoding
or reading. The merged cursor may need to look ahead through a key group to decide
the next visible row. If that fails after two rows have been copied into this
refill, the catch block clears the **entire current refill** and makes the source
terminal. It does not publish a plausible two-row success.

Previously returned batches owned by other objects remain valid. They are only a
prefix of a failed query, not a successfully completed query result. We do not
rewind the storage cursor and retry this same producer: some input was consumed.
A new query can restart from its retained snapshot. A complete result sink will
need its own success/failure boundary later in the course.

When the cursor runs out, the source releases its mutable cursor but retains the
view until the source object is released. A final nonempty refill still returns
`Pull::batch`, even if its size equals capacity exactly. The following valid call
returns `Pull::end` with empty output. An initially empty relation returns end on
its first call. Thus neither a short final batch nor an exactly full final batch
requires an extra fake row.

Finish the source file with the account scan:

```cpp title="src/query/scan.cpp — complete file, segment 3"
Pull AccountScan::next(AccountBatch& output) & {
    check_output(output, budget_, 1, state_);
    output.clear();
    if (state_ == StreamState::ended) { return Pull::end; }
    try {
        if (!started_) {
            cursor_ = view_.scan({Relation::accounts, 0});
            started_ = true;
        }
        while (output.size() < output.capacity() && cursor_.valid()) {
            const auto row = decode_visible_account(cursor_.entry());
            output.append(row);
            cursor_.next();
        }
        if (!cursor_.valid()) { cursor_.reset(); state_ = StreamState::ended; }
        return output.size() == 0 ? Pull::end : Pull::batch;
    } catch (...) {
        output.clear(); cursor_.reset(); state_ = StreamState::failed;
        throw;
    }
}

}  // namespace strata::query
```

Account keys are the final relation in the fixed schema, so the range starts at
`{accounts, 0}` and has no upper bound. The decoder still checks the relation.
Accounts arrive in primary-key order with at most one visible value per account,
a property supplied by the same version/tombstone merge as the event scan.

The two scanners deliberately have parallel, explicit loops. There is no hidden
record dispatch framework. The later merge join can consume `AccountBatch`
without assuming the entire account relation fits in memory.

## Make a predicate with representable endpoints

Create `include/strata/query/operators.hpp`. Days are 32-bit unsigned labels, but
a half-open interval containing the maximum day needs an upper endpoint one past
that maximum. Store endpoints in 64 bits:

$$
0 \le d_{\mathrm{low}} \le d_{\mathrm{high}} \le 2^{32},
\qquad d_{\mathrm{low}} \le d < d_{\mathrm{high}}.
$$

Here $d$ is an event's day. Equal endpoints select no rows. The final possible day,
$2^{32}-1$, is selected by the interval $[2^{32}-1,2^{32})$ without wrapping a
32-bit addition. Start the header:

```cpp title="include/strata/query/operators.hpp — complete file, segment 1"
#ifndef STRATA_QUERY_OPERATORS_HPP
#define STRATA_QUERY_OPERATORS_HPP

#include "strata/query/scan.hpp"

#include <cstddef>
#include <cstdint>
#include <memory>
#include <stdexcept>
#include <utility>

namespace strata::query {

class DayRange {
public:
    DayRange(std::uint64_t lower, std::uint64_t upper) : lower_(lower), upper_(upper) {
        if (lower > upper || upper > (std::uint64_t{1} << 32)) {
            throw std::invalid_argument("day interval outside [0, 2^32]");
        }
    }
    bool contains(std::uint32_t day) const noexcept { return lower_ <= day && day < upper_; }
private:
    std::uint64_t lower_;
    std::uint64_t upper_;
};
```

A `DayRange` is a small validated value. Its per-row `contains` operation cannot
throw or mutate source state. Arbitrary user callbacks, expression parsing, and
nullable columns are outside this fixed-plan chapter.

## Compact selected rows without reallocating

`EventFilter` owns an upstream source **by value** and an input `ScanBatch`.
Its caller supplies a separate output batch large enough for the worst case:
every input row might pass. No references to the caller's operator object or
output spans are saved.

```cpp title="include/strata/query/operators.hpp — complete file, segment 2"

```

The template's member definition must remain visible in the header so a caller
can instantiate it for its chosen source type, as in Chapter 5. Member declaration
order constructs `source_` before asking it for the budget used by `input_`.
If that input allocation fails, construction unwinds the already-owned source.
It cannot leave a half-built filter retaining an unowned cursor.

Let $n$ be the input row count and let $p_i$ be one when input row $i$ passes the
day predicate and zero otherwise. The selected count is

$$
k = \sum_{i=0}^{n-1} p_i, \qquad 0 \le k \le n \le C.
$$

The output position of a selected input row $i$ is the number of selected rows
before it:

$$
j(i) = \sum_{h=0}^{i-1} p_h.
$$

`output.size()` is that running count. `append` writes the row at this position
and increments it. For days `[2, 3, 2, 3]` and the interval `[3, 4)`, input rows
1 and 3 land at output rows 0 and 1, preserving their relative order. No selection
vector or output growth allocation is needed.

We borrow the input's day column once for the loop. Nothing in the loop refills,
resizes, or moves `input_`, so that borrow remains valid. Each selected row is
copied as an owning `ScannedEvent` into output. The next upstream pull may overwrite
input safely; output owns different arrays.

The **unconditional** `return Pull::batch` after filtering is essential. With
input days `[3, 3, 3]` and a `[4, 5)` filter, the selected count is zero but one
input batch was consumed. The next batch can contain day 4. The filter takes one
upstream batch step per pull rather than hiding an arbitrarily long loop that
searches for a nonempty result. This does not promise bounded wall-clock latency:
I/O and storage's version resolution still have their own costs.

## Project away provenance, not required query fields

The projection owns another scan-shaped input buffer and emits an `EventBatch`.
All four event fields survive. The sequence column ends here.

```cpp title="include/strata/query/operators.hpp — complete file, segment 3"
template<class Source> requires PullSource<Source, ScanBatch>
class EventFilter {
public:
    EventFilter(Source source, DayRange days, std::size_t capacity)
        : source_(std::move(source)), input_(source_.budget(), capacity), days_(days) {}
    EventFilter(const EventFilter&) = delete;
    EventFilter& operator=(const EventFilter&) = delete;
    EventFilter(EventFilter&&) = default;
    EventFilter& operator=(EventFilter&&) = default;
    std::shared_ptr<memory::Budget> budget() const noexcept { return input_.budget(); }
    Pull next(ScanBatch& output) & {
        check_output(output, budget(), input_.capacity(), state_);
        output.clear();
        if (state_ == StreamState::ended) { return Pull::end; }
        try {
            if (source_.next(input_) == Pull::end) {
                if (input_.size() != 0) { throw std::logic_error("nonempty end batch"); }
                state_ = StreamState::ended;
                return Pull::end;
            }
            const auto days = input_.events().columns().days;
            for (std::size_t i = 0; i < input_.size(); ++i) {
                if (days_.contains(days[i])) { output.append(input_.row(i)); }
            }
            return Pull::batch;  // Zero selected rows are still a completed input step.
        } catch (...) {
            output.clear(); input_.clear(); state_ = StreamState::failed;
            throw;
        }
    }
private:
    Source source_;
    ScanBatch input_;
    DayRange days_;
    StreamState state_ = StreamState::ready;
};

template<class Source> requires PullSource<Source, ScanBatch>
class EventProject {
public:
    EventProject(Source source, std::size_t capacity)
        : source_(std::move(source)), input_(source_.budget(), capacity) {}
    EventProject(const EventProject&) = delete;
    EventProject& operator=(const EventProject&) = delete;
    EventProject(EventProject&&) = default;
    EventProject& operator=(EventProject&&) = default;
    std::shared_ptr<memory::Budget> budget() const noexcept { return input_.budget(); }
    Pull next(EventBatch& output) & {
        check_output(output, budget(), input_.capacity(), state_);
        output.clear();
        if (state_ == StreamState::ended) { return Pull::end; }
        try {
            if (source_.next(input_) == Pull::end) {
                if (input_.size() != 0) { throw std::logic_error("nonempty end batch"); }
                state_ = StreamState::ended;
                return Pull::end;
            }
            for (std::size_t i = 0; i < input_.size(); ++i) {
                output.append(input_.events().row(i));
            }
            return Pull::batch;
        } catch (...) {
            output.clear(); input_.clear(); state_ = StreamState::failed;
            throw;
        }
    }
private:
    Source source_;
    ScanBatch input_;
    StreamState state_ = StreamState::ready;
};

}  // namespace strata::query
#endif
```

This stage also propagates a successful zero-row step. It does not convert it to
end. Both wrappers catch failures after upstream work begins, clear their current
output, and become terminal. A failure may leave the upstream source at a later
position; retrying the wrapper would not reconstruct the lost logical batch.

Each wrapper requires output capacity at least its configured input capacity.
Use compatible capacities when composing sources: the projection's input must
satisfy its filter's output requirement. A direct caller's preflight mistake is
rejected before work; an incorrectly composed upstream buffer detected *inside*
a pull is an execution exception and terminalizes the outer wrapper. The supplied
plan uses one capacity throughout, avoiding that configuration ambiguity.

The concrete composition, implemented as a complete helper in Part 3, is:

```cpp title="Composition shape — the runnable helper appears in Part 3"
using Plan = EventProject<EventFilter<EventScan>>;
```

The outer object contains its filter, which contains its scan, which owns the
read view. Moving a plan transfers that ownership chain. There is no heap-allocated
operator graph, `std::function`, or framework implementing these loops. A template
constraint describes the required operations; concrete types and their members
provide the actual ownership.

## Account for simultaneous buffers

For one capacity $C$, the filter owns a scan input of $36C$ bytes, the projection
owns another scan input of $36C$, and the consumer owns projected output of
$28C$. These allocations coexist:

$$
M_{\mathrm{columns}} = 36C + 36C + 28C = 100C.
$$

At capacity three, this is 300 charged bytes before constructing the lazy storage
cursor. A separately active account output adds $12C_a$ bytes, where $C_a$ is its
own capacity. Source readers, retained snapshots, cache frames, cursor state,
stack objects, and excluded runtime overhead are not erased by that formula.
The common ledger admits charged allocations from all of them together; it is
not yet a separate whole-query reservation or a process-memory cap.

Projection reduces **selected-row payload** from 36 to 28 bytes and prevents
storage sequence values from flowing into later operators. It does not immediately
free the two upstream buffers, nor does an empty output free its capacity. A more
fused plan could avoid intermediate copies; a larger batch can also waste more
capacity when few rows pass. Those are reasons to measure, not reasons to assert
that a columnar batch must be faster.

## Trace the tiny relation before running it

Use the roadmap's five events, ordered by event ID:

| ID | Account | Day | Cents |
|---:|---:|---:|---:|
| 100 | 10 | 3 | 500 |
| 101 | 10 | 3 | 250 |
| 102 | 20 | 3 | 700 |
| 103 | 99 | 3 | 50 |
| 104 | 10 | 4 | 100 |

For capacity three and day interval `[4, 5)`, trace these pulls:

| Pull | Scan input IDs | Filter count | Project result |
|---|---|---:|---|
| 1 | 100, 101, 102 | 0 | `batch`, zero rows |
| 2 | 103, 104 | 1 | `batch`, event 104 |
| 3 | None | 0 | `end`, zero rows |

For `[3, 4)`, the two successful batch sizes are three and one. Event 103 remains
present: account 99 is missing, but this chapter has not implemented the inner
join. Dropping that row now would conceal work that belongs to the join.

Capacity one is the same executor used as a one-row baseline. It produces five
batch steps, the last empty under the day-3 predicate, followed by end. The
capacity-three execution produces two batch steps followed by end. These are
observed logical call counts, not a timing benchmark or a prediction about M4
speedup. Both must match independent expected rows before any timing is useful.

Continue with [Part 3 — Boundary, retention, and refill checks](https://25d-228.github.io/cplusplus-learning/chapter-14-part-3.html).
