# Chapter 16 — A streaming many-to-one merge join

## Part 3 — Skew, invalid streams, and snapshot checks

The engine code from
[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-16.html) and
[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-16-part-2.html)
now composes with the existing storage and sorter. This part supplies the
complete `tests/merge_join.cpp`, the CMake append, and runnable diagnosis. It
shares Chapter 16's 5.0-hour allocation; it is not a separate project.

The reference implementation in the tests deliberately uses a map lookup and a
separate tuple comparator. It may materialize small fixture data because it is a
test oracle, not the production join. The engine path must still consume the
account relation through `AccountScan`, with its configured bounded batch.

## Keep fixture ownership separate from engine ownership

Start the test file with imports, active checks, exception expectations, and an
isolated temporary directory. Fixture directory removal is restricted to the
unique directory this fixture created; it is not a database reclamation policy.
All objects using its path live inside its scope.

```cpp title="tests/merge_join.cpp — complete file, segment 1"
#include "strata/query/merge_join.hpp"
#include "strata/storage/database.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <cstdlib>
#include <exception>
#include <filesystem>
#include <iostream>
#include <limits>
#include <map>
#include <memory>
#include <span>
#include <stdexcept>
#include <string>
#include <system_error>
#include <tuple>
#include <type_traits>
#include <utility>
#include <vector>

namespace {
using namespace strata;
using namespace strata::query;
using namespace strata::storage;
namespace fs = std::filesystem;
void check(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}
template<class Error, class F>
void expect(F&& action) {
    try { action(); } catch (const Error&) { return; }
    throw std::runtime_error("expected exception missing");
}
class TemporaryDirectory {
public:
    TemporaryDirectory() {
        char pattern[] = "/tmp/strata-join-test-XXXXXX";
        const auto path = ::mkdtemp(pattern);
        if (!path) { throw std::system_error(errno, std::generic_category(), "fixture directory"); }
        path_ = path;
    }
    ~TemporaryDirectory() noexcept { std::error_code ignored; fs::remove_all(path_, ignored); }
    TemporaryDirectory(const TemporaryDirectory&) = delete;
    TemporaryDirectory& operator=(const TemporaryDirectory&) = delete;
    const std::string& name() const noexcept { return path_; }
private:
    std::string path_;
};
```

The following source can return a deliberately empty progress batch before each
real batch. A `SourceProbe` can count calls or inject a malformed end or exception.
The source borrows the fixture's row span; the enclosing test owns those arrays
or vectors until the plan is destroyed. Only this fixture adapter uses that
borrow. The database factory owns its read-view handles instead.

Type aliases instantiate one fixture source for each row/batch pair. The static
assertions check the expected interfaces and output move properties; they are
not substitutes for the ordering tests.

```cpp title="tests/merge_join.cpp — complete file, segment 2"
struct SourceProbe { std::size_t calls = 0; bool throw_next = false, nonempty_end = false; };
template<class Row, class Batch>
class Source {
public:
    Source(std::shared_ptr<memory::Budget> budget, std::span<const Row> rows,
           bool empties = false, SourceProbe* probe = nullptr)
        : budget_(std::move(budget)), rows_(rows), empties_(empties), probe_(probe) {}
    std::shared_ptr<memory::Budget> budget() const noexcept { return budget_; }
    Pull next(Batch& output) {
        check_output(output, budget_, 1, StreamState::ready); output.clear();
        if (probe_) {
            ++probe_->calls;
            if (probe_->throw_next) { throw std::runtime_error("injected upstream failure"); }
            if (probe_->nonempty_end) { output.append(Row{}); return Pull::end; }
        }
        if (ended_) { return Pull::end; }
        if (empties_ && empty_turn_) { empty_turn_ = false; return Pull::batch; }
        empty_turn_ = true;
        if (at_ == rows_.size()) { ended_ = true; return Pull::end; }
        while (at_ < rows_.size() && output.size() < output.capacity()) { output.append(rows_[at_++]); }
        return Pull::batch;
    }
private:
    std::shared_ptr<memory::Budget> budget_;
    std::span<const Row> rows_;  // The test fixture, not the engine, owns these rows.
    bool empties_, empty_turn_ = true, ended_ = false;
    SourceProbe* probe_;
    std::size_t at_ = 0;
};
using EventSource = Source<Event, EventBatch>;
using AccountSource = Source<Account, AccountBatch>;
using TestJoin = MergeJoin<EventSource, AccountSource>;
static_assert(PullSource<TestJoin, JoinedBatch>);
static_assert(PullSource<SnapshotJoin, JoinedBatch>);
static_assert(!std::is_copy_constructible_v<TestJoin>);
static_assert(std::is_nothrow_move_constructible_v<JoinedBatch>);
static_assert(std::is_nothrow_move_assignable_v<JoinedBatch>);
```

## Use an oracle with different failure modes

The oracle builds a dimension map, sorts facts by independent value tuples, and
looks up each account. It does not call `EventOrder`, either head loader, or the
join state machine. Its own duplicate-dimension check catches invalid test
fixtures rather than choosing one value arbitrarily.

`drain` is a **test collector**, not the production output API. It accepts empty
progress, gathers copied rows, bounds fixture calls to detect a stuck loop, and
checks absorbing end. A production sink should process each batch without
building an unbounded result vector; complete result publication comes later.

```cpp title="tests/merge_join.cpp — complete file, segment 3"
std::vector<JoinedRow> oracle(std::vector<Event> events, std::span<const Account> accounts) {
    std::map<std::uint64_t, std::uint32_t> dimension;
    for (const auto account : accounts) {
        check(dimension.emplace(account.account_id, account.region_id).second, "bad oracle dimension");
    }
    std::sort(events.begin(), events.end(), [](Event a, Event b) {
        return std::tuple{a.account_id, a.event_id, a.day, a.amount_cents} <
               std::tuple{b.account_id, b.event_id, b.day, b.amount_cents};
    });
    std::vector<JoinedRow> result;
    for (const auto event : events) {
        const auto found = dimension.find(event.account_id);
        if (found != dimension.end()) { result.push_back({event, found->second}); }
    }
    return result;
}
template<class Plan>
std::vector<JoinedRow> drain(Plan& plan, JoinedBatch& output) {
    std::vector<JoinedRow> rows;
    std::size_t calls = 0;
    for (;;) {
        check(++calls < 100000, "join stopped making progress");
        if (plan.next(output) == Pull::end) { break; }
        for (std::size_t i = 0; i < output.size(); ++i) { rows.push_back(output.row(i)); }
    }
    check(output.size() == 0 && plan.next(output) == Pull::end, "nonabsorbing join end");
    return rows;
}
```

## Test the output type before the join

A capacity-two joined batch has 64 column bytes. Five different failure ordinals
cover construction of its replacement's four event columns and region column.
Every failed replacement must preserve the old row, the exact borrowed region
address, its value, and the old charge. The remaining checks exercise whole-owner
moves, self-move, cross-ledger assignment, moved-from reuse, and overflow before
allocation.

```cpp title="tests/merge_join.cpp — complete file, segment 4"
void check_batches() {
    auto budget = std::make_shared<memory::Budget>(4096);
    auto other = std::make_shared<memory::Budget>(4096);
    {
        JoinedBatch batch{budget, 2}; const JoinedRow value{{1, 10, 3, 500}, 7};
        batch.append(value); check(budget->used() == 64 && batch.row(0) == value, "joined columns");
        const auto borrowed = batch.regions(); const auto* address = borrowed.data();
        for (std::size_t ordinal = 1; ordinal <= 5; ++ordinal) {
            budget->fail_allocation_on(ordinal);
            expect<std::bad_alloc>([&] { batch.replace_capacity(3); });
            budget->clear_failure();
            check(batch.row(0) == value && batch.regions().data() == address && borrowed[0] == 7 &&
                  budget->used() == 64, "failed joined replacement changed value or charge");
        }
        batch.append({{2, 10, 3, 100}, 7});
        expect<std::length_error>([&] { batch.append(value); });
        expect<std::out_of_range>([&] { (void)batch.row(2); });
        JoinedBatch moved{std::move(batch)};
        check(batch.size() == 0 && batch.capacity() == 0 && moved.size() == 2, "joined move");
        JoinedBatch destination{other, 3}; destination = std::move(moved);
        check(other->used() == 0 && destination.budget() == budget && destination.row(0) == value,
              "cross-ledger joined move");
        auto& alias = destination; destination = std::move(alias);
        check(destination.size() == 2, "joined self move");
        batch.replace_capacity(1); batch.append(value);
        destination.replace_capacity(3);
        check(destination.size() == 0 && destination.capacity_bytes() == 96, "joined replacement");
        expect<std::invalid_argument>([&] { JoinedBatch invalid{budget, 0}; });
        expect<std::invalid_argument>([] { JoinedBatch invalid{nullptr, 1}; });
        expect<std::length_error>([&] {
            JoinedBatch invalid{budget, std::numeric_limits<std::size_t>::max() / 32 + 1};
        });
    }
    check(budget->used() == 0 && other->used() == 0, "joined batch cleanup");
}
```

Moving the destination across ledgers is particularly useful here: returning
memory through the wrong allocator could otherwise be hidden by two budgets with
the same numeric limit. The assertions require the old destination ledger to
return to zero and the destination object to expose the source ledger. The
surviving output owns decoded columns, not any table block.

## Vary boundaries without changing the answer

The event fixture below has 31 rows, including a 27-row group for account 10.
Missing accounts 1 and 11 remove two rows. Accounts at zero and the maximum
64-bit ID test unsigned endpoints. Fourteen dimension rows are larger than every
account batch used by this test.

The Cartesian product of three event capacities, three account capacities, three
output capacities, three work limits, and two empty-progress modes contains
$3\cdot3\cdot3\cdot3\cdot2=162$ cases. Each compares every returned field against
the oracle. The test also checks all four empty/nonempty input combinations.

```cpp title="tests/merge_join.cpp — complete file, segment 5"
void check_shapes() {
    auto budget = std::make_shared<memory::Budget>(1024 * 1024);
    const auto maximum = std::numeric_limits<std::uint64_t>::max();
    std::vector<Event> events{{0, 0, 0, 0}, {1, 1, 3, 10}};
    for (std::uint64_t i = 2; i < 29; ++i) { events.push_back({i, 10, 3, i}); }
    events.push_back({29, 11, 4, 9}); events.push_back({30, maximum, 4, maximum});
    std::vector<Account> accounts{{0, 0}};
    for (std::uint64_t i = 2; i <= 25; i += 2) { accounts.push_back({i, static_cast<std::uint32_t>(i)}); }
    accounts.push_back({maximum, std::numeric_limits<std::uint32_t>::max()});
    const auto expected = oracle(events, accounts);
    std::size_t cases = 0;
    for (const std::size_t ecap : std::array<std::size_t, 3>{1, 2, 7}) {
        for (const std::size_t acap : std::array<std::size_t, 3>{1, 3, 5}) {
            for (const std::size_t outcap : std::array<std::size_t, 3>{1, 2, 8}) {
                for (const std::size_t work : std::array<std::size_t, 3>{1, 4, 100}) {
                    for (const bool empties : {false, true}) {
                        {
                            TestJoin plan{EventSource{budget, events, empties},
                                          AccountSource{budget, accounts, empties}, {ecap, acap, work}};
                            JoinedBatch output{budget, outcap};
                            check(drain(plan, output) == expected, "merge join differs from independent oracle");
                            const auto s = plan.stats();
                            check(s.events == events.size() && s.accounts == accounts.size() &&
                                  s.matched == expected.size() && s.unmatched == 2 &&
                                  s.comparisons <= events.size() + accounts.size(), "join counts");
                            check(budget->used() == output.capacity_bytes(), "end retained join inputs");
                        }
                        check(budget->used() == 0, "shape ledger"); ++cases;
                    }
                }
            }
        }
    }
    for (const bool empty_events : {false, true}) {
        for (const bool empty_accounts : {false, true}) {
            const std::span<const Event> e = empty_events ? std::span<const Event>{} : events;
            const std::span<const Account> a = empty_accounts ? std::span<const Account>{} : accounts;
            TestJoin plan{EventSource{budget, e, true}, AccountSource{budget, a, true}, {2, 2, 1}};
            JoinedBatch output{budget, 2};
            check(drain(plan, output) == ((empty_events || empty_accounts) ? std::vector<JoinedRow>{} : expected),
                  "empty-side result");
            check(plan.stats().events == e.size() && plan.stats().accounts == a.size(), "tail not validated");
        }
    }
    check(budget->used() == 0, "empty-side ledger");
    std::cout << "shape_cases=" << cases << " hot_account_rows=27 boundary_rows=" << expected.size() << '\n';
}
```

The test's `budget->used() == output.capacity_bytes()` after draining is an
ownership assertion. The fixture sources have no engine-owned dynamic data of
their own. Once `Inputs` is reset, only the consumer's column storage should
remain charged. End-of-stream alone would not have released those inputs if the
join retained both source objects forever.

## Separate retryable preflight from execution failure

A wrong output ledger must leave its sentinel row and both source call counts
unchanged. A paused move must preserve the current account head after event one
has already been returned. Replacing an already-owning destination join must
release its old input columns rather than leak them.

Six charged input-column acquisitions occur during simple join construction.
The final block reserves all remaining root allowance and arms the next
allocation to fail. The already constructed join still drains fixture sources:
its kernel does not allocate. The oracle/test vector allocation is outside this
engine ledger; this is not a claim that the whole process performs no allocation.

```cpp title="tests/merge_join.cpp — complete file, segment 6"
void check_preflight_and_moves() {
    auto budget = std::make_shared<memory::Budget>(4096);
    auto other = std::make_shared<memory::Budget>(4096);
    const std::array<Event, 3> events{{{1, 10, 1, 5}, {2, 10, 1, 6}, {3, 20, 2, 7}}};
    const std::array<Account, 2> accounts{{{10, 1}, {20, 2}}};
    {
        SourceProbe left, right;
        TestJoin plan{EventSource{budget, events, false, &left},
                      AccountSource{budget, accounts, false, &right}, {1, 1, 10}};
        JoinedBatch wrong{other, 1}; wrong.append({events[2], 99});
        expect<std::invalid_argument>([&] { plan.next(wrong); });
        check(left.calls == 0 && right.calls == 0 && wrong.row(0).region_id == 99, "preflight consumed input");
        JoinedBatch output{budget, 1};
        check(plan.next(output) == Pull::batch && output.row(0).event == events[0], "first row");
        TestJoin moved{std::move(plan)};
        TestJoin destination{EventSource{budget, {}}, AccountSource{budget, {}}, {2, 2, 10}};
        destination = std::move(moved);
        check(drain(destination, output) == std::vector<JoinedRow>{{events[1], 1}, {events[2], 2}},
              "paused join move lost head");
        expect<std::invalid_argument>([&] {
            TestJoin bad{EventSource{budget, events}, AccountSource{other, accounts}};
        });
        for (const JoinOptions options : {JoinOptions{0, 1, 1}, {1, 0, 1}, {1, 1, 0}}) {
            expect<std::invalid_argument>([&] {
                TestJoin bad{EventSource{budget, events}, AccountSource{budget, accounts}, options};
            });
        }
    }
    check(budget->used() == 0 && other->used() == 0, "preflight ledger");
    for (std::size_t ordinal = 1; ordinal <= 6; ++ordinal) {
        budget->fail_allocation_on(ordinal);
        expect<std::bad_alloc>([&] {
            TestJoin fail{EventSource{budget, events}, AccountSource{budget, accounts}, {2, 3, 10}};
        });
        budget->clear_failure(); check(budget->used() == 0, "failed join construction charge");
    }
    {
        TestJoin plan{EventSource{budget, events}, AccountSource{budget, accounts}, {1, 1, 10}};
        JoinedBatch output{budget, 1};
        auto pressure = budget->reserve(budget->limit() - budget->used());
        budget->fail_allocation_on(1);
        check(drain(plan, output).size() == 3, "join unexpectedly allocates while stepping");
        budget->clear_failure(); pressure.reset();
    }
    check(budget->used() == 0, "no-allocation join ledger");
}
```

## Reject broken input contracts, including unused tails

Run duplicate and descending dimension keys both inside and across batch
boundaries. Duplicate keys in the tail must fail even with no events. The same
reasoning applies to decreasing event keys after the account side has ended.
Exact duplicate events remain legal because they represent repeated input rows,
not a uniqueness violation in the dimension.

The final two loops distinguish malformed `end` from an ordinary upstream
exception. Both sources are tested as the failing side. Previously returned
output remains unchanged; the current refill is empty; removing the injected
fault cannot revive the terminal plan.

```cpp title="tests/merge_join.cpp — complete file, segment 7"
void check_invalid_streams() {
    auto budget = std::make_shared<memory::Budget>(4096);
    const std::vector<Event> good{{1, 10, 3, 1}, {2, 10, 3, 2}, {3, 20, 3, 3}};
    const std::vector<Account> unique{{10, 1}, {20, 2}, {30, 3}};
    auto reject = [&](std::span<const Event> events, std::span<const Account> accounts, std::size_t cap) {
        {
            TestJoin plan{EventSource{budget, events}, AccountSource{budget, accounts}, {cap, cap, 100}};
            JoinedBatch output{budget, 1};
            expect<std::logic_error>([&] { (void)drain(plan, output); });
            check(output.size() == 0 && plan.state() == StreamState::failed, "invalid stream not terminal");
            output.append({{99, 99, 9, 9}, 9});
            expect<std::logic_error>([&] { plan.next(output); });
            check(output.row(0).event.event_id == 99, "failed-state preflight changed output");
            check(budget->used() == output.capacity_bytes(), "failure retained join inputs");
        }
        check(budget->used() == 0, "invalid stream ledger");
    };
    for (const std::size_t cap : std::array<std::size_t, 2>{1, 3}) {
        const std::vector<Account> duplicate{{10, 1}, {10, 2}, {20, 2}};
        const std::vector<Account> trailing{{10, 1}, {20, 2}, {30, 3}, {30, 4}};
        const std::vector<Account> backwards{{10, 1}, {20, 2}, {19, 3}};
        reject(good, duplicate, cap); reject(good, trailing, cap); reject({}, trailing, cap);
        reject(good, backwards, cap);
        const std::vector<Event> bad_keys{{1, 10, 3, 1}, {2, 20, 3, 2}, {3, 10, 3, 3}};
        const std::vector<Event> bad_ties{{2, 10, 3, 1}, {1, 10, 3, 2}};
        reject(bad_keys, unique, cap); reject(bad_ties, unique, cap); reject(bad_keys, {}, cap);
    }
    {
        const std::vector<Event> repeated(3, good.front());
        TestJoin plan{EventSource{budget, repeated}, AccountSource{budget, unique}, {1, 1, 2}};
        JoinedBatch output{budget, 1}; check(drain(plan, output).size() == 3, "fact duplicates lost");
    }
    for (const bool left_bad : {false, true}) {
        SourceProbe left, right; (left_bad ? left : right).nonempty_end = true;
        TestJoin plan{EventSource{budget, good, false, &left},
                      AccountSource{budget, unique, false, &right}, {1, 1, 100}};
        JoinedBatch output{budget, 1};
        expect<std::logic_error>([&] { plan.next(output); });
        check(output.size() == 0 && plan.state() == StreamState::failed, "nonempty end accepted");
    }
    for (const bool fail_left : {false, true}) {
        SourceProbe left, right;
        const std::array<Event, 2> rows{{{1, 10, 3, 1}, {2, 20, 3, 2}}};
        TestJoin plan{EventSource{budget, rows, false, &left},
                      AccountSource{budget, unique, false, &right}, {1, 1, 100}};
        JoinedBatch prior{budget, 1};
        check(plan.next(prior) == Pull::batch && prior.row(0).region_id == 1, "fault prefix");
        (fail_left ? left : right).throw_next = true;
        JoinedBatch current{budget, 3}; current.append({{99, 99, 9, 9}, 99});
        expect<std::runtime_error>([&] { plan.next(current); });
        check(current.size() == 0 && prior.row(0).event.event_id == 1, "failed refill damaged owned prefix");
        (fail_left ? left : right).throw_next = false;
        expect<std::logic_error>([&] { plan.next(current); });
    }
    check(budget->used() == 0, "stream failure ledger");
}
```

## Make the seven-comparison trace executable

The first call returns events 100 and 101 after exactly one account-source call.
A second call returns event 102. Input counts are five and three, matched and
unmatched counts are three and two, and the comparison count is seven.

The second block deliberately supplies a duplicate account after one useful row
has already been staged during the failing refill. A separate earlier batch
retains event 100. The partial current batch disappears, even though the internal
match counter includes its discarded event 101.

```cpp title="tests/merge_join.cpp — complete file, segment 8"
void check_trace_and_partial_failure() {
    auto budget = std::make_shared<memory::Budget>(4096);
    const std::array<Event, 5> events{{{9, 1, 3, 5}, {100, 10, 3, 500}, {101, 10, 3, 250},
                                      {102, 20, 3, 700}, {103, 99, 3, 50}}};
    const std::array<Account, 3> accounts{{{10, 1}, {20, 2}, {30, 4}}};
    {
        SourceProbe left, right;
        TestJoin plan{EventSource{budget, events, false, &left},
                      AccountSource{budget, accounts, false, &right}, {1, 1, 100}};
        JoinedBatch output{budget, 2};
        check(budget->used() == 104, "28+12+64 column accounting");
        check(plan.next(output) == Pull::batch && output.size() == 2 &&
              output.row(0) == JoinedRow{events[1], 1} && output.row(1) == JoinedRow{events[2], 1},
              "first trace batch");
        check(right.calls == 1, "hot account was fetched again instead of reused");
        check(plan.next(output) == Pull::batch && output.size() == 1 &&
              output.row(0) == JoinedRow{events[3], 2}, "key transition trace");
        check(plan.next(output) == Pull::end, "trace completion");
        const auto s = plan.stats();
        check(s.events == 5 && s.accounts == 3 && s.matched == 3 && s.unmatched == 2, "trace counts");
        std::cout << "trace: events=" << s.events << " accounts=" << s.accounts << " matched=" << s.matched
                  << " unmatched=" << s.unmatched << " comparisons=" << s.comparisons << '\n';
    }
    {
        const std::array<Account, 2> duplicate{{{10, 1}, {10, 9}}};
        TestJoin plan{EventSource{budget, events}, AccountSource{budget, duplicate}, {1, 1, 100}};
        JoinedBatch prior{budget, 1};
        check(plan.next(prior) == Pull::batch && prior.row(0).event.event_id == 100, "partial-failure prefix");
        JoinedBatch current{budget, 4};
        expect<std::logic_error>([&] { plan.next(current); });
        check(current.size() == 0 && prior.row(0).event.event_id == 100 && plan.stats().matched == 2,
              "candidate row leaked or old output changed");
    }
    check(budget->used() == 0, "trace ledger");
}
```

This test distinguishes three facts that are often conflated: the operator may
have consumed a row, counted a match, and still not have returned that row in a
successful batch. A later result writer must use query completion, not a positive
match counter, to decide whether a result is authoritative.

## Reuse the existing native storage and sort configuration

The small sort uses three-row chunks, two-row run blocks, and three run-file
descriptors. Chapter 15's fan-in calculation therefore selects two input readers
plus one output writer. The database uses small table blocks so scans cross real
file-frame boundaries. A 64-byte total cache allowance for one slot leaves too
little room for these frames; reads exercise the charged bypass path rather
than relying on an indefinitely warm data cache.

All setup helpers are given here. The `seed_small` rows are the roadmap's example;
`commit` requires an acknowledged committed result rather than silently accepting
an uncertain write in a test fixture.

```cpp title="tests/merge_join.cpp — complete file, segment 9"
SortOptions tiny_sort() {
    SortOptions options;
    options.chunk_rows = 3; options.block_rows = 2; options.file_descriptors = 3;
    options.workspace_bytes = 128 * 1024; options.stack_allowance = 16 * 1024;
    options.disk_bytes = 64 * 1024;
    return options;
}
DatabaseOptions tiny_database() {
    DatabaseOptions options; options.arena_bytes = 256 * 1024;
    options.cache_slots = 1; options.cache_bytes = 64;
    options.table = {128, 256, 32, 64 * 1024};
    return options;
}
void commit(Database& db, std::span<const Mutation> changes) {
    check(db.write(changes).outcome == WriteOutcome::committed, "database fixture commit");
}
void seed_small(Database& db) {
    const std::array changes{
        Mutation::put(Account{10, 1}), Mutation::put(Account{20, 2}),
        Mutation::put(Event{100, 10, 3, 500}), Mutation::put(Event{101, 10, 3, 250}),
        Mutation::put(Event{102, 20, 3, 700}), Mutation::put(Event{103, 99, 3, 50}),
        Mutation::put(Event{104, 10, 4, 100})};
    commit(db, changes);
}
```

## Keep the old region with the old amount

The old day-three join has three rows. Pause after event 100, then atomically
change account 10 to region 3, change event 101 from 250 to 900 cents, and delete
event 102. Flush, compact, and collect while the old plan still exists.

The old plan must retain event 101 with 250 cents and region 1, plus event 102
with 700 cents and region 2. A new plan must return only events 100 and 101, both
in region 3, with 500 and 900 cents. No aggregation is required to observe whether
the two relations shared a snapshot.

```cpp title="tests/merge_join.cpp — complete file, segment 10"
void check_same_snapshot() {
    TemporaryDirectory path, parent;
    auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_database()}; seed_small(db);
        auto old_stats = make_sort_stats(budget);
        auto old = make_snapshot_join(db.snapshot(), DayRange{3, 4}, 2, {1, 1, 10},
                                       tiny_sort(), parent.name(), old_stats);
        JoinedBatch prefix{budget, 1};
        while (old.next(prefix) == Pull::batch && prefix.size() == 0) {}
        check(prefix.size() == 1 && prefix.row(0) == JoinedRow{{100, 10, 3, 500}, 1}, "old first row");
        check(db.retention().snapshots == 1, "two scans captured different read states");
        const std::array update{Mutation::put(Account{10, 3}), Mutation::put(Event{101, 10, 3, 900}),
                                Mutation::erase({Relation::events, 102})};
        commit(db, update); check(db.rotate(), "updated log rotation");
        while (db.flush_oldest()) {}
        CompactionOptions compact; compact.output = tiny_database().table;
        db.compact_prefix(db.manifest().table_count, compact);
        check(db.collect_obsolete().retained != 0, "paused join lost snapshot sources");
        JoinedBatch output{budget, 1};
        check(drain(old, output) == std::vector<JoinedRow>{{{101, 10, 3, 250}, 1}, {{102, 20, 3, 700}, 2}},
              "old join mixed new region, amount, or tombstone");
        check(prefix.row(0) == JoinedRow{{100, 10, 3, 500}, 1}, "paused output changed");
        check(db.retention().snapshots == 0, "ended join retained view");
        check(db.collect_obsolete().retained == 0, "old names still protected after end");
        auto new_stats = make_sort_stats(budget);
        auto current = make_snapshot_join(db.snapshot(), DayRange{3, 4}, 2, {2, 1, 4},
                                           tiny_sort(), parent.name(), new_stats);
        check(drain(current, output) == std::vector<JoinedRow>{{{100, 10, 3, 500}, 3}, {{101, 10, 3, 900}, 3}},
              "new join disagrees with atomic update");
        check(old_stats->initial_runs == 2 && old_stats->live_files == 0 && new_stats->live_files == 0,
              "small joined spill cleanup");
    }
    check(budget->used() == 0 && fs::is_empty(parent.name()), "snapshot join cleanup");
    std::cout << "snapshot rows: old=3 new=2; old region and amount stay together\n";
}
```

`retention().snapshots == 1` demonstrates that the two source handles share one
admitted state, rather than acquiring independent views. After the old query
ends, its inputs are gone even though the `old` facade and its statistics still
exist. Collection can then reclaim formerly retained names. The earlier output
batch is independent and remains readable throughout.

The fixture also proves the factory uses the real spill path: four qualifying
events exceed a three-row chunk, creating two initial runs. There is no map-based
shortcut for the native query.

## Stream a dimension larger than the buffers

The larger fixture stores 128 accounts and 100 events, then deletes account 20.
Only events with IDs 4 through 99 have day 3, leaving 96 facts. Of those, IDs 4
through 70 contribute 67 matches for account 10. Ten later facts use other
existing accounts. The remaining nineteen refer to deleted account 20 or missing
account 999. Thus $J=67+10=77$ and the visible dimension has 127 rows.

With chunk capacity three, initial run count is $\lceil96/3\rceil=32$. Two-way
materialized passes reduce that count as
$32\to16\to8\to4\to2$. The last two runs are merged directly into the join, so
there are four materialized passes, not five. While the query is paused, these
final run owners coexist with the account scan and output columns.

```cpp title="tests/merge_join.cpp — complete file, segment 11"
void check_spilled_dimension() {
    TemporaryDirectory path, parent;
    auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_database()};
        std::vector<Mutation> changes;
        std::vector<Account> dimension;
        std::vector<Event> facts;
        for (std::uint64_t id = 1; id <= 128; ++id) {
            const Account account{id, static_cast<std::uint32_t>(id % 7)};
            changes.push_back(Mutation::put(account));
            if (id != 20) { dimension.push_back(account); }
        }
        for (std::uint64_t id = 0; id < 100; ++id) {
            const auto key = id <= 70 ? 10 : (id % 3 == 0 ? 20 : (id % 3 == 1 ? 999 : id + 1));
            const Event event{id, key, static_cast<std::uint32_t>(id < 4 ? 0 : 3), 100 + id};
            changes.push_back(Mutation::put(event));
            if (event.day == 3) { facts.push_back(event); }
        }
        commit(db, changes); check(db.rotate() && db.flush_oldest(), "large fixture table");
        const std::array deleted{Mutation::erase({Relation::accounts, 20})}; commit(db, deleted);
        const auto expected = oracle(facts, dimension);
        auto stats = make_sort_stats(budget);
        auto plan = make_snapshot_join(db.snapshot(), DayRange{3, 4}, 2, {3, 2, 4},
                                        tiny_sort(), parent.name(), stats);
        JoinedBatch output{budget, 2}; std::vector<JoinedRow> rows;
        do { check(plan.next(output) == Pull::batch, "large join ended before first row"); }
        while (output.size() == 0);
        for (std::size_t i = 0; i < output.size(); ++i) { rows.push_back(output.row(i)); }
        check(stats->initial_runs == 32 && stats->merge_passes == 4 && stats->live_files == 2,
              "large join did not retain a multipass spilled input");
        const std::array update{Mutation::put(Account{10, 99})}; commit(db, update);
        check(db.rotate(), "large update rotation"); while (db.flush_oldest()) {}
        CompactionOptions compact; compact.output = tiny_database().table;
        db.compact_prefix(db.manifest().table_count, compact);
        check(db.collect_obsolete().retained != 0, "paused disk join lost retention");
        const auto tail = drain(plan, output); rows.insert(rows.end(), tail.begin(), tail.end());
        check(rows == expected, "spilled join differs from independent map oracle");
        check(plan.stats().events == 96 && plan.stats().accounts == 127 && plan.stats().matched == 77 &&
              plan.stats().unmatched == 19, "large join cardinality");
        check(db.retention().snapshots == 0 && stats->live_files == 0 && stats->disk_reserved == 0 &&
              stats->open_run_files == 0 && !stats->directory_live, "completed join retained resources");
        (void)db.collect_obsolete();
        std::cout << "spilled join: facts=" << plan.stats().events << " accounts=" << plan.stats().accounts
                  << " matched=" << plan.stats().matched << " initial_runs=" << stats->initial_runs
                  << " materialized_passes=" << stats->merge_passes << '\n';
    }
    check(budget->used() == 0 && fs::is_empty(parent.name()), "large join cleanup");
}
```

The account input holds two rows, not 127. Changing the hot account's region to
99 during the pause cannot affect the old query. Subsequent compaction and
collection also must not revoke its original sources. The independent map
oracle compares every returned event and region; counts alone would miss a
correct-cardinality join with wrong regions.

This is a native-file functional fixture on the machine running the tests. The
reported counts are not throughput measurements, process-memory measurements,
or a claim about how long the workload takes on an M4.

## Let a right-side failure destroy left-side spill owners

For 25 input events, the tiny sorter creates sixteen files over run generation
and materialized merging before its final output begins. Now make the account
source throw on its first pull. The join has already invoked the left source;
its failure cleanup must therefore destroy a fully built, partially consumed
sort, not merely an unstarted placeholder.

The next block creates common-budget pressure after plan construction. The
local sort workspace plan remains valid, but another reservation consumes the
root headroom. Failure is terminal even after releasing that reservation. The
last loop injects charged allocation failures at successive boundaries through
sort construction, run processing, join columns, and output creation, then
checks that the first successful plan matches the oracle.

```cpp title="tests/merge_join.cpp — complete file, segment 12"
void check_sort_failures_and_pressure() {
    TemporaryDirectory parent;
    auto budget = std::make_shared<memory::Budget>(2 * 1024 * 1024);
    std::vector<Event> events;
    for (std::uint64_t id = 0; id < 25; ++id) { events.push_back({id, id % 4 + 1, 3, id}); }
    const std::array<Account, 4> accounts{{{1, 1}, {2, 2}, {3, 3}, {4, 4}}};
    {
        auto stats = make_sort_stats(budget); SourceProbe fault; fault.throw_next = true;
        auto sorted = ExternalSort{EventSource{budget, events}, 2, tiny_sort(), parent.name(), stats};
        MergeJoin plan{std::move(sorted), AccountSource{budget, accounts, false, &fault}, JoinOptions{1, 1, 10}};
        JoinedBatch output{budget, 1};
        expect<std::runtime_error>([&] { plan.next(output); });
        check(stats->files_created == 16 && stats->live_files == 0 && stats->disk_reserved == 0 &&
              stats->open_run_files == 0 && !stats->directory_live, "right failure leaked left spill owners");
        fault.throw_next = false; expect<std::logic_error>([&] { plan.next(output); });
    }
    check(budget->used() == 0, "cross-source failure charge");
    {
        auto stats = make_sort_stats(budget);
        auto sorted = ExternalSort{EventSource{budget, events}, 2, tiny_sort(), parent.name(), stats};
        MergeJoin plan{std::move(sorted), AccountSource{budget, accounts}, JoinOptions{1, 1, 10}};
        JoinedBatch output{budget, 1};
        auto pressure = budget->reserve(budget->limit() - budget->used());
        expect<memory::BudgetExceeded>([&] { plan.next(output); });
        pressure.reset(); expect<std::logic_error>([&] { plan.next(output); });
        check(stats->live_files == 0 && !stats->directory_live, "shared pressure leaked directory");
    }
    check(budget->used() == 0, "pressure charge");
    std::size_t failures = 0;
    for (std::size_t ordinal = 1; ordinal < 128; ++ordinal) {
        bool success = false;
        {
            auto stats = make_sort_stats(budget);
            budget->fail_allocation_on(ordinal);
            try {
                auto sorted = ExternalSort{EventSource{budget, events, true}, 2, tiny_sort(), parent.name(), stats};
                MergeJoin plan{std::move(sorted), AccountSource{budget, accounts, true}, JoinOptions{2, 1, 4}};
                JoinedBatch output{budget, 2};
                check(drain(plan, output) == oracle(events, accounts), "allocation campaign result");
                success = true;
            } catch (const std::bad_alloc&) { ++failures; }
            budget->clear_failure();
            check(stats->live_files == 0 && stats->disk_reserved == 0 && stats->open_run_files == 0 &&
                  !stats->directory_live, "failed join allocation leaked sort resources");
        }
        check(budget->used() == 0 && fs::is_empty(parent.name()), "join allocation charge");
        if (success) { break; }
        check(ordinal + 1 < 128, "allocation campaign never succeeded");
    }
    std::cout << "joined_plan_allocation_failures=" << failures << '\n';
}
```

`SortStats` stays alive after the failing plan so file and descriptor cleanup is
observable. The allocation ordinal is a deterministic seam, not a portable count
of all library allocations in a process. This fixture observed 41 failing
ordinals before success in the Linux authoring environment; another compatible
standard-library layout can change that number without changing the contract.
Every tested scope still requires zero covered charge and no private temporary
names once its owners have been destroyed.

## Abandon output, then let ownership escape the database

A caller need not drain a stream. Destroying a paused join must release its
remaining final runs and snapshot handles. Conversely, a caller may destroy the
database while retaining a plan: the copied read-view handles keep its sources,
lock context, and ledger alive.

The final scope reopens the same fixture database, creates a plan and output,
then releases both the database and the caller's budget handle. The plan can
still return event 100. Destroying that plan removes its spill files but cannot
destroy the output columns. Only releasing the final output makes the weak
budget observation expire.

```cpp title="tests/merge_join.cpp — complete file, segment 13"
void check_abandonment_and_escape() {
    TemporaryDirectory path, parent;
    auto budget = std::make_shared<memory::Budget>(8 * 1024 * 1024);
    {
        Database db{budget, path.name(), DatabaseOpen::create, tiny_database()}; seed_small(db);
        auto stats = make_sort_stats(budget);
        {
            auto plan = make_snapshot_join(db.snapshot(), DayRange{3, 4}, 1, {1, 1, 10},
                                            tiny_sort(), parent.name(), stats);
            JoinedBatch output{budget, 1};
            while (plan.next(output) == Pull::batch && output.size() == 0) {}
            check(output.size() == 1 && stats->live_files == 2, "abandonment fixture");
        }
        check(db.retention().snapshots == 0 && stats->live_files == 0 && stats->open_run_files == 0 &&
              !stats->directory_live, "abandoned join retained sources");
    }
    check(budget->used() == 0 && fs::is_empty(parent.name()), "abandoned join charge");
    std::weak_ptr<memory::Budget> weak = budget;
    std::optional<SnapshotJoin> escaped;
    std::optional<JoinedBatch> held;
    {
        Database db{budget, path.name(), DatabaseOpen::recover, tiny_database()};
        auto stats = make_sort_stats(budget);
        escaped.emplace(make_snapshot_join(db.snapshot(), DayRange{3, 4}, 1, {1, 1, 10},
                                             tiny_sort(), parent.name(), stats));
        held.emplace(budget, 1); budget.reset();
    }
    check(!weak.expired(), "escaped join lost budget");
    while (escaped->next(*held) == Pull::batch && held->size() == 0) {}
    check(held->size() == 1 && held->row(0) == JoinedRow{{100, 10, 3, 500}, 1}, "escaped join value");
    escaped.reset();
    check(!weak.expired() && held->row(0).event.amount_cents == 500 && fs::is_empty(parent.name()),
          "owned output lost data or abandoned runs");
    held.reset(); check(weak.expired(), "last joined column retained budget");
    std::cout << "escaped join and output release the final budget owner\n";
}
```

The fixture directory objects outlive these escaped owners. Reversing that
scope order would intentionally remove files while a query still needed them;
RAII only works when the owning relationships represent the intended lifetime.
Finish the test file with its namespace closure and entry point:

```cpp title="tests/merge_join.cpp — complete file, segment 14"
}  // namespace

int main() {
    try {
        check_batches(); check_trace_and_partial_failure(); check_shapes();
        check_preflight_and_moves(); check_invalid_streams(); check_same_snapshot();
        check_spilled_dimension(); check_sort_failures_and_pressure(); check_abandonment_and_escape();
        std::cout << "merge join: all checks passed\n";
        return 0;
    } catch (const std::exception& error) {
        std::cerr << "merge join: " << error.what() << '\n'; return 1;
    }
}
```

## Add one target and keep every existing check

Append this block to the existing root `CMakeLists.txt`. Do not replace the
previous targets or move query implementation into a test-only library.

```cmake title="CMakeLists.txt — append join targets"
target_sources(strata-core PRIVATE src/query/merge_join.cpp)
add_executable(strata-join-tests tests/merge_join.cpp)
target_link_libraries(strata-join-tests PRIVATE strata-core)
target_compile_options(strata-join-tests PRIVATE -Wall -Wextra -Wpedantic -Wconversion -Wshadow)
add_test(NAME merge-join COMMAND strata-join-tests)
```

Run from the learner source root using a compatible C++20 compiler. These
commands extend the existing build workflow; no particular compiler release or
invented LTS channel is required.

```sh title="Terminal — cumulative Debug and Release checks"
cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build-debug -j 4
ctest --test-dir build-debug --output-on-failure
./build-debug/strata-join-tests

cmake -S . -B build-release -DCMAKE_BUILD_TYPE=Release
cmake --build build-release -j 4
ctest --test-dir build-release --output-on-failure
```

There are now **twenty-three CTest entries**. The following output was observed
with Clang 17 on Linux; the logical fixture counts are intended results on other
compatible targets too. The allocation-failure count is an observation, not a
required ABI constant.

```text title="Observed merge-join output on Linux"
trace: events=5 accounts=3 matched=3 unmatched=2 comparisons=7
shape_cases=162 hot_account_rows=27 boundary_rows=29
snapshot rows: old=3 new=2; old region and amount stay together
spilled join: facts=96 accounts=127 matched=77 initial_runs=32 materialized_passes=4
joined_plan_allocation_failures=41
escaped join and output release the final budget owner
merge join: all checks passed
```

A separate sanitizer configuration keeps memory diagnostics out of the normal
build flags:

```sh title="Terminal — address and undefined-behavior checks"
cmake -S . -B build-sanitize -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build-sanitize -j 4
ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 UBSAN_OPTIONS=halt_on_error=1 \
  ctest --test-dir build-sanitize --output-on-failure
```

Leak-checking availability depends on the platform sanitizer runtime. Use the
available native diagnostics rather than treating an unsupported leak option as
an application result. The full correct suite passed Debug, Release, and combined
AddressSanitizer/UndefinedBehaviorSanitizer configurations in the Linux authoring
container. Native M4/macOS execution remains unverified. No new process-crash,
physical power-loss, concurrency, or cancellation result is claimed here.

## Diagnose a logical defect that sanitizers cannot infer

Temporarily change only the equality branch in `MergeJoin::next`. Insert the
marked incorrect reset after clearing the event:

```cpp title="Temporary diagnostic edit — do not retain"
output.append({*event_, account_->region_id});
count(stats_.matched); event_.reset();
account_.reset();  // Deliberate bug: loses a match for the next same-account event.
if (output.size() == output.capacity()) { return Pull::batch; }
```

Rebuild the join target and run it in each configuration:

```sh title="Terminal — the deliberately wrong join must fail"
cmake --build build-debug --target strata-join-tests -j 4
./build-debug/strata-join-tests
cmake --build build-release --target strata-join-tests -j 4
./build-release/strata-join-tests
cmake --build build-sanitize --target strata-join-tests -j 4
./build-sanitize/strata-join-tests
```

All three diagnostic runs in the authoring environment exited with failure and
`merge join: first trace batch`, without a sanitizer memory-error report. The
code accesses valid objects but violates the advancement rule. Removing the
incorrect reset restores the three matched rows. Restore the correct branch before rerunning
the full passing suites; the retained implementation above is correct.

For another useful prediction, change the trace's work quota from 100 to one.
A new empty batch is expected when account 10 is discarded before loading
account 20. It is not missing data. The existing shape tests already verify that
these empty progress returns do not alter the concatenated result.

| Symptom | Likely cause | Useful check |
|---|---|---|
| Only the first event of a hot account survives | Both heads advanced on equality | Inspect `account_` across output return. |
| The last account duplicate is accepted | Join stopped as soon as events ended | Drain the unused dimension suffix in the empty-events test. |
| Correct row count, wrong regions after a write | Sources were captured from different views | Use the single-view factory; inspect old/new row values. |
| Empty output is reported as successful completion | Progress was confused with end | Run with work quota one and alternating empty fixture batches. |
| Failure leaves final spill files open | Only the failing source was released | Inspect caller-retained `SortStats` after `Inputs` destruction. |
| A join fails under shared pressure despite a valid local sort plan | Local limits were mistaken for root headroom | Sum all live columns, storage owners, and sort charges. |

The chapter's completed boundary is an owning, bounded many-to-one merge over a
spilled event stream and streamed accounts. The account value can survive
refills without retaining a borrow; end and failure release the upstream plan;
returned columns remain independently owned. Output is still in event-account
order. Chapter 17 will establish grouping order, implement checked aggregates,
and connect a complete synchronous query result.

## References used in this chapter

[PostgreSQL, Table Expressions, Joined Tables](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN)
provides the matching-pair definition of inner joins. Our unique-dimension
contract deliberately excludes many-to-many expansion and null semantics.

[PostgreSQL, `nodeMergejoin.c` introductory notes](https://doxygen.postgresql.org/nodeMergejoin_8c_source.html)
describes ordered synchronization and why a general repeated-key join needs
mark/restore behavior. This chapter implements its restricted state machine
independently; PostgreSQL is not an executor dependency.

[The C++ draft, requires expressions](https://eel.is/c++draft/expr.prim.req)
specifies the type/expression checks behind `PullSource`.
[The C++ draft, optional modifiers](https://eel.is/c++draft/optional.mod)
specifies destruction on `reset`. The draft is a living reference; only the
C++20 facilities used explicitly in these listings are required.

The prior mechanisms remain fully defined in
[Chapter 14](https://25d-228.github.io/cplusplus-learning/chapter-14.html)
and [Chapter 15](https://25d-228.github.io/cplusplus-learning/chapter-15.html).
No external reference substitutes for a required helper or test listing here.
