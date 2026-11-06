# Chapter 16 — A streaming many-to-one merge join

## Part 2 — Resume without borrowing stale rows

Continue the two files from
[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-16.html).
The state machine operates on owning head values. A source can refill or clear
its batch without changing a previously copied head, and an output pause need
not preserve a borrow into the storage cursor's current frame.

This part completes the engine code. It does not change Chapter 15's run format,
heap, or cleanup policy. The full chapter still shares one 5.0-hour implementation
allocation.

## Translate batch progress into head progress

Each head loader returns one of three local results: a copied row is ready,
upstream made progress but yielded no row, or the source ended. These results are
not interchangeable. If a filter rejects every event in a scan batch, the join
must not conclude that the relation is empty.

The next definition uses a **trailing return type**: `auto function() -> Head`.
Here `auto` does not request return-type deduction; `-> Head` explicitly names the
return type after the qualified member name has established class scope. It lets
us name the private nested enum `Head` directly. The repeated template and
`requires` lines must agree with the class declaration.

Append this complete member definition to the header:

```cpp title="include/strata/query/merge_join.hpp — complete file, segment 4"
template<class Events, class Accounts>
    requires PullSource<Events, EventBatch> && PullSource<Accounts, AccountBatch>
auto MergeJoin<Events, Accounts>::event_head() -> Head {
    if (event_) { return Head::row; }
    if (event_end_) { return Head::end; }
    auto& in = *inputs_;
    if (in.event_at == in.events.size()) {
        const auto result = in.left.next(in.events);
        in.event_at = 0;
        if (result == Pull::end) {
            if (in.events.size() != 0) { throw std::logic_error("nonempty event end"); }
            event_end_ = true; return Head::end;
        }
        if (result != Pull::batch) { throw std::logic_error("invalid event pull result"); }
        if (in.events.size() == 0) { return Head::progress; }
    }
    const auto candidate = in.events.row(in.event_at);
    if (previous_event_ && EventOrder{}(candidate, *previous_event_)) {
        throw std::logic_error("join events are not sorted");
    }
    previous_event_ = candidate; event_ = candidate;
    ++in.event_at; count(stats_.events);
    return Head::row;
}
```

The first two tests are cheap and important. An existing head is reused without
reloading its row or incrementing its count. Once `event_end_` is true, the
upstream source is never called again.

When the batch index equals its size, its old rows have all been loaded. One call
to `left.next` replaces that batch. A valid `Pull::end` must be empty. An empty
`Pull::batch` returns `Head::progress` immediately rather than entering an
unbounded empty-batch skip loop. The caller can receive an empty progress result
and decide when to pull again. A source that repeatedly reports progress without
actually progressing violates the semantic source contract; the concept cannot
prove otherwise.

A nonempty batch yields a **copy** through `row(event_at)`. Test
`EventOrder(candidate, previous)` rather than `!EventOrder(previous, candidate)`:
the former rejects a decrease while allowing equal fact rows. This is the
nondecreasing order compatible with Chapter 15. The cursor index advances only
after that check succeeds. A corrupt ordering raises an exception before that
candidate can participate in a join.

The input batch index may already point to the next row while `event_` still
holds the current copied row. Those are different responsibilities: the index
tracks loading; clearing the head tracks logical consumption. Confusing them is
a common cause of skipped rows at output boundaries.

## Enforce uniqueness across account refills

The account loader has the same progress protocol but a stricter comparison.
Account ID must increase, not merely fail to decrease. Keep the last loaded
account in `previous_account_`, even when a batch is cleared. Otherwise a
capacity-one input would make every duplicate look like the first row of a new
batch.

```cpp title="include/strata/query/merge_join.hpp — complete file, segment 5"
template<class Events, class Accounts>
    requires PullSource<Events, EventBatch> && PullSource<Accounts, AccountBatch>
auto MergeJoin<Events, Accounts>::account_head() -> Head {
    if (account_) { return Head::row; }
    if (account_end_) { return Head::end; }
    auto& in = *inputs_;
    if (in.account_at == in.accounts.size()) {
        const auto result = in.right.next(in.accounts);
        in.account_at = 0;
        if (result == Pull::end) {
            if (in.accounts.size() != 0) { throw std::logic_error("nonempty account end"); }
            account_end_ = true; return Head::end;
        }
        if (result != Pull::batch) { throw std::logic_error("invalid account pull result"); }
        if (in.accounts.size() == 0) { return Head::progress; }
    }
    const auto candidate = in.accounts.row(in.account_at);
    if (previous_account_ && candidate.account_id <= previous_account_->account_id) {
        throw std::logic_error("join accounts must have increasing unique keys");
    }
    previous_account_ = candidate; account_ = candidate;
    ++in.account_at; count(stats_.accounts);
    return Head::row;
}
```

Suppose one batch ends with `(10,region 1)` and the next starts with
`(10,region 9)`. The saved previous key is still 10, so `candidate.account_id <=
previous_account_->account_id` rejects the second row. The comparison never
subtracts unsigned IDs, so key zero and the maximum 64-bit key are ordinary
endpoints rather than overflow hazards.

A current account may survive many calls to `next`. It is not a view of
`in.accounts.row(...)`; that function returns an `Account` value and the optional
stores another value copy. Releasing a table block pin during a later source
refill cannot invalidate the account head. We deliberately spend a few scalar
bytes to simplify the ownership argument instead of holding a pin merely to
remember an ID and a region.

Validation is incremental. A useful row in an earlier output batch can precede
a duplicate discovered later. Until the stream completes successfully, those
earlier rows form only a prefix of a possibly failed query. The next chapter
will use that distinction when publishing a complete result.

## Finish only after both tails have been checked

An inner join could stop producing rows as soon as either input ends. Our
operator makes a stronger completion statement: both ordered-source contracts
have been checked through their ends. Therefore, after events end it drains and
validates remaining accounts; after accounts end it drains and validates events
while counting them unmatched.

This policy costs reads for an unused suffix. It is deliberate. Accounts
`(10,20,30,30)` must not be reported as valid merely because the last event used
account 20. Even an empty event relation does not hide a duplicate in the account
stream. A future early-stop operator would need a different contract, such as
"the consumed prefix was valid," rather than silently weakening this one.

Each loop iteration below performs at most one comparison, one tail discard, or
a completion/progress decision. The configured quota prevents an arbitrarily
long unmatched suffix from being consumed entirely by the join's own loop in a
single call. It does **not** bound time spent inside an upstream call. In
particular, `ExternalSort` builds all its runs on its first pull. No latency or
cooperative cancellation guarantee is implied.

```cpp title="include/strata/query/merge_join.hpp — complete file, segment 6"
template<class Events, class Accounts>
    requires PullSource<Events, EventBatch> && PullSource<Accounts, AccountBatch>
Pull MergeJoin<Events, Accounts>::next(JoinedBatch& output) & {
    check_output(output, budget_, 1, state_);  // Retryable; no clear or input advance yet.
    output.clear();
    if (state_ == StreamState::ended) { return Pull::end; }
    try {
        for (std::size_t step = 0; step < options_.work_rows; ++step) {
            const auto left = event_head();
            if (left == Head::progress) { break; }
            const auto right = account_head();
            if (right == Head::progress) { break; }
            if (left == Head::end && right == Head::end) {
                state_ = StreamState::ended; release_inputs();
                return output.size() == 0 ? Pull::end : Pull::batch;
            }
            if (left == Head::end) {
                account_.reset();  // Validate the unused account suffix too.
                continue;
            }
            if (right == Head::end) {
                count(stats_.unmatched); event_.reset();
                continue;
            }
            count(stats_.comparisons);
            if (event_->account_id < account_->account_id) {
                count(stats_.unmatched); event_.reset();
            } else if (event_->account_id > account_->account_id) {
                account_.reset();
            } else {
                output.append({*event_, account_->region_id});
                count(stats_.matched); event_.reset();
                // Keep account_: the next event may have this same account key.
                if (output.size() == output.capacity()) { return Pull::batch; }
            }
        }
        if (output.size() == 0) { count(stats_.progress_returns); }
        return Pull::batch;
    } catch (...) {
        output.clear(); state_ = StreamState::failed; release_inputs(); throw;
    }
}
```

The output preflight is outside `try`. A wrong ledger or unusable capacity is a
caller error before execution: the old output and input position remain intact,
and the caller may supply a valid batch. A failed stream is also rejected there,
without clearing a caller's sentinel rows. After entering refill, an exception
means the stream may have consumed input. The catch block clears partial output,
marks the stream failed, destroys its input owners, and rethrows. It does not
pretend that retrying the same state could reconstruct consumed rows.

On equality, the appended `JoinedRow` contains copied event and region values.
Clearing `event_` is the logical consumption step. **Do not clear `account_`
there.** A full output returns immediately with that account still available for
the next pull. At an event-key increase, the greater-than branch eventually
clears it. All comparison paths make progress without rescanning a group.

`Pull::batch` may be empty when the step quota is reached, the source reports
empty progress, or the join has so far discarded only unmatched rows. Such a
return is not end. Conversely, discovering both ends can leave a final nonempty
output. The code marks the stream ended and releases its inputs but returns
`batch` for those rows; the following valid call clears output and returns `end`.
Once ended, it never reconstructs `Inputs`.

There is no dynamic allocation in the merge kernel after its batch construction.
`row`, optional scalar assignment/reset, comparison, and append use existing
storage. An upstream scan, sort, or file read can still allocate or throw. The
failure tests separate those two statements instead of treating a no-allocation
join loop as a no-failure query.

## Follow a failure across both sources

Consider a capacity-one earlier output containing event 100. On the next call,
the join stages event 101 into a different output batch. It then advances to an
event for account 20 and discovers that the next account key is another 10. The
account loader throws. The current batch is cleared, so staged event 101 is not
returned as part of a successful refill. The earlier batch still owns event 100
and remains readable.

The internal `matched` counter is now two: one returned row and one staged row
that was discarded. That is why it must not be called "delivered rows." At a
successful, unsaturated end, no staged rows have been discarded and the useful
identity is

$$
\texttt{events}=\texttt{matched}+\texttt{unmatched}.
$$

On failure, `Inputs` destruction also matters when the account side throws before
returning its first row. The left external sort may already have created several
runs and opened the final merge inputs. Releasing only the failing account
cursor would leak the rest of the query's work. Resetting the single input owner
destroys **both** source chains in their normal RAII order.

Destruction attempts cleanup; it cannot force a failed filesystem unlink to
succeed. Keep the caller-owned `SortStats` alive to inspect Chapter 15's cleanup
error, live-file count, disk reservation, and private directory path. Once this
join has destroyed its sorted source, it no longer exposes that source's
`retry_cleanup` method. There is no automatic crash-leftover janitor or durable
query restart in this chapter. Successful test cleanup is not a blanket
filesystem guarantee.

## Construct both scans from one read view

A generic join can check shared-budget identity and stream order. It cannot infer
that two independently supplied sources describe the same snapshot. Provide one
factory for the real database plan that receives a single captured `ReadView`.
Append the aliases and declaration, then close the header:

```cpp title="include/strata/query/merge_join.hpp — complete file, segment 7"
using ProjectedEvents = EventProject<EventFilter<EventScan>>;
using SnapshotJoin = MergeJoin<ExternalSort<ProjectedEvents>, AccountScan>;
SnapshotJoin make_snapshot_join(storage::ReadView view, DayRange days,
    std::size_t upstream_rows, JoinOptions join_options, SortOptions sort_options,
    std::string_view temporary_parent, std::shared_ptr<SortStats> sort_stats,
    SpillHook hook = {});

}  // namespace strata::query
#endif
```

`ProjectedEvents` names the existing scan/filter/project composition.
`SnapshotJoin` names the concrete join specialization; neither alias creates an
object or hides another executor. The factory takes ownership of a read-view
handle, not a `Database&`, so it has no reason to capture a new snapshot midway
through construction. Its definition closes the implementation file:

```cpp title="src/query/merge_join.cpp — complete file, segment 4"
SnapshotJoin make_snapshot_join(storage::ReadView view, DayRange days,
    std::size_t upstream_rows, JoinOptions join_options, SortOptions sort_options,
    std::string_view temporary_parent, std::shared_ptr<SortStats> sort_stats, SpillHook hook) {
    auto projected = EventProject{EventFilter{EventScan{view}, days, upstream_rows}, upstream_rows};
    auto sorted = ExternalSort{std::move(projected), upstream_rows, sort_options,
                               temporary_parent, std::move(sort_stats), hook};
    return SnapshotJoin{std::move(sorted), AccountScan{std::move(view)}, join_options};
}
}  // namespace strata::query
```

`EventScan{view}` copies a handle to the already captured immutable state. The
last line moves the factory's remaining handle into `AccountScan`. Both scans
therefore retain one state with one committed sequence boundary. They can have
separate cursors and buffers without having separate logical snapshots.

A concrete caller uses `db.snapshot()` **once**, passes the resulting view to
this function, creates an output batch with the same budget, and pulls until
`Pull::end`. Part 3 shows that complete call sequence. It also verifies that an
old plan keeps the old region and old event amount together after one atomic
batch changes both. Two calls to `db.snapshot()` separated by a write would not
provide that property, even though both views were individually valid.

The upstream capacity is the same at scan/filter/project/sort-input boundaries,
as required by Chapter 14's operator scratch contracts. Join input capacities
can differ: the sort accepts any positive output capacity, and `AccountScan`
fills whatever positive account batch it receives. The consumer can use output
capacity one without reducing the sort's chunk size or materializing the
account relation.

## Account for simultaneous columns, not one operator at a time

Let $E$ be the join's event input capacity, $A_b$ its account input capacity, and
$O$ the consumer output capacity, all measured in rows. Their column allocation
sum is

$$
M_{\mathrm{join}}=28E+12A_b+32O\quad\text{bytes}.
$$

The coefficients are the actual fields: an event has three 64-bit columns and
one 32-bit column; an account has one of each width; a joined row adds one more
32-bit column. With $E=1$, $A_b=1$, and $O=2$, the sum is $28+12+64=104$ bytes.
The small fixture checks that exact ledger charge before execution.

Now include the existing pipeline. Let $U$ be its common upstream batch capacity.
The filter retains a `ScanBatch` costing $36U$, the projection retains another
`ScanBatch` costing $36U$, and `ExternalSort` retains its input `EventBatch`
costing $28U$. These owners coexist with the join buffers:

$$
M_{\mathrm{columns}}=100U+28E+12A_b+32O.
$$

For $U=2$, $E=3$, $A_b=2$, and $O=2$, this is
$200+84+24+64=372$ bytes. It does **not** include the sort's charged core, chunk
and merge frames, its reserved stack allowance, the shared report, storage
cursor controls, table metadata, cached blocks, or retained arenas. Those live
allocations/reservations also use the same root `Budget`. The sort's local
workspace limit is not another independent allowance to add for free.

There are also fixed-size C++ control objects, including `MergeJoin` itself and
its optional copied heads, that may live on the caller's stack. Their bounded
`sizeof` is not charged as a heap allocation here. Stack, library internals,
allocator overhead, and operating-system page cache remain reasons that this
ledger is not a process-RSS cap. Chapter 15's explicit sort-stack allowance is a
policy reservation, not a measurement of every stack frame in the query.

The dimension can have 127 or 127,000 rows without enlarging a two-row account
batch. The retained storage generations and their metadata still have their own
limits. "The join does not materialize the dimension" is not the same as
"the entire database has constant memory use." At end, the input columns and
sources are destroyed while the consumer output remains charged and readable.

## The output has join order, not group order

Emission only removes unmatched events and adds a region; it never reorders
surviving events. Therefore output remains a subsequence of the event order.
Two adjacent account IDs can map to unrelated regions, and one region can appear
in many separated account groups. That does not make the join incorrect. It
means a later grouping stage cannot assume equal `(region_id, day)` keys are
adjacent. Chapter 17 will establish that different order before aggregating.

Before continuing, predict the trace with output capacity one. There should be
three nonempty output batches, with event IDs 100, 101, and 102. Account 10 must
still be fetched only once before the first two matches. Then set `work_rows`
to one: additional empty progress returns are expected, but the concatenated
rows and final counts must remain identical. The test program in
[Part 3](https://25d-228.github.io/cplusplus-learning/chapter-16-part-3.html)
checks these distinctions using an independent algorithm.
