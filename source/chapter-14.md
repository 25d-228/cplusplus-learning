# Chapter 14 — Owned batches and a streaming query pipeline

## Part 1 — Make the batch own every column

A storage cursor can produce a correct row and still be unsafe as a query result.
Its `EntryView` borrows bytes whose meaning ends at the next cursor advance. A
filter that saves that span and then asks storage for another row has saved a
borrow, not the first row. A later operator needs a different boundary: an owning
batch whose buffers remain valid independently of the storage position.

There is also a control-flow problem. The first three events can all fail a day
predicate while a later event passes. A batch with zero selected rows means that
one input step produced no output. It does **not** mean that storage has ended.
This chapter implements both boundaries explicitly.

> **Learning objectives**
> Build charged typed column storage; distinguish allocated capacity from visible
> rows; transfer complete batch ownership; define a pull protocol with separate
> batch and end results; decode a snapshot into owned fields before advancing;
> compose scan, filter, and projection without borrowing an operator that has
> gone away; make a failed refill terminal rather than silently losing rows.

**Prerequisites:** the complete source through all three parts of
[Chapter 13](https://25d-228.github.io/cplusplus-learning/chapter-13.html), including
Chapter 12's owning `ReadView` and `ReadCursor`. Keep all nineteen existing CTest
entries. Chapters 2–5 supply spans, move-only owners, failure-safe replacement,
and constraints; Chapters 9 and 12 supply `Budget` and `SharedAllocator`.
The existing storage algorithms and earlier chapter bodies are unchanged.

**Implementation allocation: 3.5 hours for all three parts together.** Use 1.0
hour for batches and interfaces, 1.5 for scan/filter/project integration, and 1.0
for lifetime, pressure, boundary tests, and diagnosis. Reading is separate.
This remains synchronous. External sorting, joins, aggregation, and threads are
not part of this chapter.

Part 1 creates `include/strata/query/batch.hpp` and `src/query/batch.cpp`.
[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-14-part-2.html) adds the
operators and one additive `ReadView` accessor.
[Part 3](https://25d-228.github.io/cplusplus-learning/chapter-14-part-3.html) supplies
the complete runnable test and CMake extension. Concatenate the numbered segments
for each new filename in reading order; they are complete files, not optional
fragments. Do not compile halfway through the three parts.

```sh title="Terminal — query source directories"
mkdir -p include/strata/query src/query tests
```

## Count the fields before choosing a layout

An event has three 64-bit fields and one 32-bit field: event ID, account ID,
amount in cents, and day. On the engine's eight-bit-byte targets, its numeric
field widths total

$$
w_e = 8 + 8 + 4 + 8 = 28\text{ bytes}.
$$

Here $w_e$ is the sum of field widths, **not** `sizeof(Event)`. An array of ordinary
`Event` objects uses the compiler's object stride, including any padding. The
[object-representation rules](https://timsong-cpp.github.io/cppwp/n4861/basic.types)
distinguish object storage from a field's value representation. The persistent
30-byte encoding from Chapter 6 is a third layout: it adds version and relation
bytes but does not include C++ padding.

Let $B$ be an imagined byte address of an array's first event, $S$ its
`sizeof(Event)`, and $o_a$ the `offsetof(Event, amount_cents)` value. The amount
field of row index $i$ is at

$$
A_{\mathrm{row}}(i) = B + iS + o_a.
$$

Let $P_a$ instead denote the first byte address of a separate amount column.
A contiguous array of 64-bit amounts uses

$$
A_{\mathrm{column}}(i) = P_a + 8i.
$$

In the Linux authoring build, `sizeof(Event)` was 32 and the amount offset was
24. Three row-array amounts therefore lie at offsets 24, 56, and 88; the three
column amounts lie at offsets 0, 8, and 16 within their own allocation. The test
prints these object-layout observations on your compiler. Do not require those
particular padding values on another implementation, and do not subtract pointers
to unrelated column allocations. Array pointer arithmetic stays within its
[own array](https://timsong-cpp.github.io/cppwp/n4861/basic.compound).

We use three concrete batch shapes:

| Batch | Columns | Allocated field bytes for capacity three |
|---|---|---:|
| `ScanBatch` | Event's four fields, plus the visible storage sequence | 108 |
| `EventBatch` | Event ID, account ID, day, cents | 84 |
| `AccountBatch` | Account ID, region ID | 36 |

The scan sequence exposes which committed version supplied a row. It is useful
when checking repeatable reads but not needed for the later join or aggregation.
Projection will discard it. **None of the four event fields can disappear from
the planned query:** account ID is the join key; event ID provides deterministic
sort tie-breaking; day and cents are needed for grouping and totals. Do not
manufacture a smaller result by silently discarding required information.

A fused scan could omit the sequence immediately. We keep a visible scan/filter/
projection boundary here to learn its ownership and cost. This is not a claim
that three copying stages are an optimized plan.

## Allocate one typed column, not a packed byte arena

A column is an array of exactly one scalar type. Its allocator obtains aligned
storage for that type; we never reinterpret an arbitrary byte buffer as an array
of 64-bit integers. `ColumnNumber` restricts the template to the two unsigned
integer types used by these schemas. The `||` combines two type-equality checks,
not two runtime branches.

Start `batch.hpp` with the imports, constraint, and column owner:

```cpp title="include/strata/query/batch.hpp — complete file, segment 1"
#ifndef STRATA_QUERY_BATCH_HPP
#define STRATA_QUERY_BATCH_HPP

#include "strata/core/checked.hpp"
#include "strata/core/record.hpp"
#include "strata/memory/shared_allocator.hpp"

#include <concepts>
#include <cstddef>
#include <cstdint>
#include <memory>
#include <span>
#include <stdexcept>
#include <utility>

namespace strata::query {

template<class T>
concept ColumnNumber = std::same_as<T, std::uint32_t> || std::same_as<T, std::uint64_t>;

template<ColumnNumber T>
class Column {
public:
    Column(std::shared_ptr<memory::Budget> budget, std::size_t capacity)
        : allocator_(std::move(budget)), capacity_(capacity) {
        data_ = allocator_.allocate(capacity_);
        // Only the two unsigned scalar types are permitted: construction cannot throw.
        for (std::size_t i = 0; i < capacity_; ++i) { std::construct_at(data_ + i, T{0}); }
    }
    ~Column() noexcept {
        if (data_) {
            std::destroy_n(data_, capacity_);
            allocator_.deallocate(data_, capacity_);
        }
    }
    Column(const Column&) = delete;
    Column& operator=(const Column&) = delete;
    Column(Column&& other) noexcept
        : allocator_(other.allocator_), data_(std::exchange(other.data_, nullptr)),
          capacity_(std::exchange(other.capacity_, 0)) {}
    Column& operator=(Column&& other) noexcept {
        if (this != &other) { Column candidate{std::move(other)}; swap(candidate); }
        return *this;
    }
    void swap(Column& other) noexcept {
        using std::swap;
        swap(allocator_, other.allocator_);
        swap(data_, other.data_);
        swap(capacity_, other.capacity_);
    }
    std::span<T> storage() & noexcept { return {data_, capacity_}; }
    std::span<const T> storage() const & noexcept { return {data_, capacity_}; }
    std::span<T> storage() && = delete;
    std::span<const T> storage() const && = delete;
    std::shared_ptr<memory::Budget> budget() const noexcept { return allocator_.owner(); }
private:
    memory::SharedAllocator<T> allocator_;
    T* data_ = nullptr;
    std::size_t capacity_;
};
```

`allocator_.allocate(capacity_)` uses Chapter 12's owning allocator. It retains
the budget, reserves and charges the requested bytes through Chapter 9, and
checks multiplication before acquisition. The default allocator's
[allocation contract](https://timsong-cpp.github.io/cppwp/n4861/allocator.members)
provides array storage; we explicitly begin each element's lifetime with
[`construct_at`](https://timsong-cpp.github.io/cppwp/n4861/specialized.construct).
For these two scalar types, construction from zero cannot throw. Broadening the
constraint to arbitrary classes would invalidate that reasoning and require
partial-construction cleanup before linking a finished owner.

All capacity elements are initialized, including slots not yet logically visible
as rows. This pays an initialization cost once during construction, rather than
making every append negotiate object lifetime. A future measurement may justify
a different initialization strategy; it must preserve the lifetime argument.
The destructor performs the matching
[`destroy_n`](https://timsong-cpp.github.io/cppwp/n4861/specialized.destroy) and
deallocation while the owning allocator still exists. Its budget cannot vanish
between returning storage and returning the charge.

The move constructor **copies the allocator handle** and transfers the pointer
and capacity. Copying a shared pointer creates another ownership edge, not another
budget or another array. The source retains a usable allocator but no storage.
Move assignment builds a nonallocating temporary, swaps complete ownership
triples, and lets that temporary release the old destination. Allocator, pointer,
and count always travel together, including between two different budgets.

`storage() &` returns a mutable span only from a mutable lvalue column;
`storage() const &` exposes read-only elements. The rvalue overloads are deleted
because borrowing directly from a temporary owner is a likely lifetime error.
The batch classes below keep their columns private. Ordinary query clients only
receive read-only spans of the visible prefix, not mutable access to spare slots.
A [span is non-owning](https://timsong-cpp.github.io/cppwp/n4861/span.overview): copying
it does not retain the column or its budget.

## Publish one row count for four columns

Let $C$ be a batch's row capacity and $r$ its visible row count. Maintain

$$
0 \le r \le C.
$$

Each event column has $C$ live scalar objects, but only indices in $[0,r)$ are
query rows. There is one row count, not a separately mutable count per column.
The following interface returns all four borrowed spans together, each with
length $r$. `row(i)` instead returns an owning scalar `Event` value.

```cpp title="include/strata/query/batch.hpp — complete file, segment 2"
struct EventColumns {
    std::span<const std::uint64_t> event_ids;
    std::span<const std::uint64_t> account_ids;
    std::span<const std::uint32_t> days;
    std::span<const std::uint64_t> amounts;
};

class EventBatch {
public:
    static constexpr std::size_t row_bytes = 3 * sizeof(std::uint64_t) + sizeof(std::uint32_t);
    EventBatch(std::shared_ptr<memory::Budget> budget, std::size_t capacity);
    EventBatch(const EventBatch&) = delete;
    EventBatch& operator=(const EventBatch&) = delete;
    EventBatch(EventBatch&& other) noexcept;
    EventBatch& operator=(EventBatch&& other) noexcept;
    void swap(EventBatch& other) noexcept;
    void replace_capacity(std::size_t capacity);
    void clear() noexcept { rows_ = 0; }
    void append(Event event);
    Event row(std::size_t index) const;
    EventColumns columns() const & noexcept;
    EventColumns columns() const && = delete;
    std::size_t size() const noexcept { return rows_; }
    std::size_t capacity() const noexcept { return capacity_; }
    std::size_t capacity_bytes() const noexcept { return capacity_ * row_bytes; }
    std::shared_ptr<memory::Budget> budget() const noexcept { return ids_.budget(); }
private:
    std::size_t capacity_;
    std::size_t rows_ = 0;
    Column<std::uint64_t> ids_;
    Column<std::uint64_t> accounts_;
    Column<std::uint32_t> days_;
    Column<std::uint64_t> amounts_;
};
```

`capacity_bytes()` measures acquired column payload, not `r * row_bytes`.
`clear()` changes logical visibility without returning any allocation. The
capacity remains available for the next pull, and the ledger must keep charging
it. The batch object and its allocator handles normally live inside a caller's
stack object or enclosing operator; their fixed object storage is not a hidden
heap allocation made by this class.

Start `batch.cpp`. The common capacity check runs before the first column is
constructed. It rejects zero capacity, which would otherwise allow a scanner to
make no progress forever, and guards the **whole batch's** byte arithmetic before
any partial acquisition.

```cpp title="src/query/batch.cpp — complete file, segment 1"
#include "strata/query/batch.hpp"

namespace strata::query {
namespace {
std::size_t checked_capacity(std::size_t capacity, std::size_t row_bytes) {
    if (capacity == 0) { throw std::invalid_argument("zero batch capacity"); }
    (void)checked_size_mul(capacity, row_bytes);
    return capacity;
}
}  // namespace

EventBatch::EventBatch(std::shared_ptr<memory::Budget> budget, std::size_t capacity)
    : capacity_(checked_capacity(capacity, row_bytes)), ids_(budget, capacity_),
      accounts_(budget, capacity_), days_(budget, capacity_),
      amounts_(std::move(budget), capacity_) {}
EventBatch::EventBatch(EventBatch&& other) noexcept
    : capacity_(std::exchange(other.capacity_, 0)), rows_(std::exchange(other.rows_, 0)),
      ids_(std::move(other.ids_)), accounts_(std::move(other.accounts_)),
      days_(std::move(other.days_)), amounts_(std::move(other.amounts_)) {}
EventBatch& EventBatch::operator=(EventBatch&& other) noexcept {
    if (this != &other) { EventBatch candidate{std::move(other)}; swap(candidate); }
    return *this;
}
void EventBatch::swap(EventBatch& other) noexcept {
    using std::swap;
    swap(capacity_, other.capacity_); swap(rows_, other.rows_);
    ids_.swap(other.ids_); accounts_.swap(other.accounts_);
    days_.swap(other.days_); amounts_.swap(other.amounts_);
}
void EventBatch::replace_capacity(std::size_t capacity) {
    EventBatch candidate{budget(), capacity};
    swap(candidate);
}
```

Construction follows member declaration order. If the third column acquisition
fails, the second and first columns are destroyed; the unfinished `EventBatch`
destructor is not required for that cleanup. Both the original exception and the
ledger's final count therefore remain meaningful.

The move constructor transfers every column and exchanges the source's row count
and capacity with zero. Moving only the vectors or pointers while leaving a
nonzero source row count would make the source advertise nonexistent elements.
The explicit self-assignment check prevents destroying the source when it is also
the destination.

`replace_capacity` is deliberately **not** `vector::reserve`. On success it installs
a new, empty batch with exactly the requested capacity; it does not preserve rows.
On failure the old batch, its row count, and its existing borrows remain unchanged.
The new candidate must coexist with the old allocations until the swap.

For a change from capacity three to seven, the old event columns occupy 84 bytes
and the candidate occupies 196. The transition requires

$$
M_{\mathrm{replace}} = 3w_e + 7w_e = 280\text{ bytes},
$$

apart from other already charged engine storage. A ledger with space for only the
final 196 bytes can still reject the transition. That is the price of preserving
the old value while preparation can fail.

Now append the data access and publication functions:

```cpp title="src/query/batch.cpp — complete file, segment 2"
void EventBatch::append(Event event) {
    if (rows_ == capacity_) { throw std::length_error("event batch full"); }
    ids_.storage()[rows_] = event.event_id;
    accounts_.storage()[rows_] = event.account_id;
    days_.storage()[rows_] = event.day;
    amounts_.storage()[rows_] = event.amount_cents;
    ++rows_;
}
Event EventBatch::row(std::size_t index) const {
    if (index >= rows_) { throw std::out_of_range("event batch row"); }
    return {ids_.storage()[index], accounts_.storage()[index],
            days_.storage()[index], amounts_.storage()[index]};
}
EventColumns EventBatch::columns() const & noexcept {
    return {ids_.storage().first(rows_), accounts_.storage().first(rows_),
            days_.storage().first(rows_), amounts_.storage().first(rows_)};
}
```

`append` receives its row by value. It checks capacity before any store, writes
all four scalar fields, and increments `rows_` last. After the check, those stores
cannot allocate or throw. The batch therefore cannot advertise a partially written
row. This is an in-memory, single-threaded publication rule, not atomic hardware
publication to another thread.

`row` checks the index before touching any column. `columns` builds four spans
using the same prefix count. The client must not read the spare initialized zeros
as if they were additional events.

Our conservative borrow contract is: **reacquire column spans after any successful
mutation, clear, move, swap, refill, or replacement; never use them after the owner
is destroyed.** A successful append may leave old addresses intact, but an earlier
span still has its earlier length. A failed capacity replacement or rejected
preflight does not invalidate the old view because nothing changed. We do not
rely on sanitizers to enforce these logical expiration points: readable memory
can still represent the wrong batch generation.

## Keep storage provenance until projection

`ScannedEvent` is an owning value containing one `Event` and its sequence.
`ScanBatch` embeds an `EventBatch` plus one sequence column. Its visible count is
always the event batch's count; there is no independent sequence-row count to
fall out of step.

```cpp title="include/strata/query/batch.hpp — complete file, segment 3"
struct ScannedEvent {
    Event event;
    std::uint64_t sequence;
    bool operator==(const ScannedEvent&) const = default;
};
class ScanBatch {
public:
    static constexpr std::size_t row_bytes = EventBatch::row_bytes + sizeof(std::uint64_t);
    ScanBatch(std::shared_ptr<memory::Budget> budget, std::size_t capacity);
    ScanBatch(const ScanBatch&) = delete;
    ScanBatch& operator=(const ScanBatch&) = delete;
    ScanBatch(ScanBatch&&) noexcept = default;
    ScanBatch& operator=(ScanBatch&&) noexcept = default;
    void swap(ScanBatch& other) noexcept;
    void replace_capacity(std::size_t capacity);
    void clear() noexcept { events_.clear(); }
    void append(ScannedEvent row);
    ScannedEvent row(std::size_t index) const;
    const EventBatch& events() const & noexcept { return events_; }
    const EventBatch& events() const && = delete;
    std::span<const std::uint64_t> sequences() const & noexcept {
        return sequences_.storage().first(size());
    }
    std::span<const std::uint64_t> sequences() const && = delete;
    std::size_t size() const noexcept { return events_.size(); }
    std::size_t capacity() const noexcept { return events_.capacity(); }
    std::size_t capacity_bytes() const noexcept { return capacity() * row_bytes; }
    std::shared_ptr<memory::Budget> budget() const noexcept { return events_.budget(); }
private:
    EventBatch events_;
    Column<std::uint64_t> sequences_;
};
```

The defaulted move operations are safe here because both member owners already
implement the required empty-source behavior and nonthrowing ownership transfer.
There is no extra scalar count left behind in `ScanBatch`. Its `events()` result
is a borrowed reference to the member, not a copy of the event columns. The
rvalue overload is again prohibited.

Append these complete definitions to `batch.cpp`:

```cpp title="src/query/batch.cpp — complete file, segment 3"
ScanBatch::ScanBatch(std::shared_ptr<memory::Budget> budget, std::size_t capacity)
    : events_(budget, checked_capacity(capacity, row_bytes)),
      sequences_(std::move(budget), capacity) {}
void ScanBatch::swap(ScanBatch& other) noexcept {
    events_.swap(other.events_); sequences_.swap(other.sequences_);
}
void ScanBatch::replace_capacity(std::size_t capacity) {
    ScanBatch candidate{budget(), capacity};
    swap(candidate);
}
void ScanBatch::append(ScannedEvent row) {
    if (size() == capacity()) { throw std::length_error("scan batch full"); }
    if (row.sequence == 0) { throw std::invalid_argument("zero visible sequence"); }
    sequences_.storage()[size()] = row.sequence;
    events_.append(row.event);  // Publishes the shared row count last.
}
ScannedEvent ScanBatch::row(std::size_t index) const {
    const auto event = events_.row(index);  // Bounds check before the sequence access.
    return {event, sequences_.storage()[index]};
}
```

The outer constructor validates `capacity * 36` before constructing the four event
columns. Validating only `capacity * 28` would not establish that the complete
five-column accounting expression is representable.

An append first proves there is room and that the visible sequence is nonzero.
It then stores the sequence and lets `events_.append` publish the shared count.
That second call cannot run out of room: no operation in between changes the
count or capacity. Since the event payload is already an owning value, no cursor
advance or borrowed input can invalidate it during these stores.

A capacity-three scan batch with two visible rows still charges 108 bytes. Moving
it yields a destination with the same two rows and a source with size and capacity
zero. The source may be given new storage with `replace_capacity`; it cannot be
used as a scan output while its capacity remains zero.

## Give the dimension relation its own shape

Accounts need only a 64-bit account ID and a 32-bit region ID. Do not store them
in event columns with unused fields or narrow their 64-bit keys. Finish the header:

```cpp title="include/strata/query/batch.hpp — complete file, segment 4"
struct AccountColumns {
    std::span<const std::uint64_t> account_ids;
    std::span<const std::uint32_t> regions;
};
class AccountBatch {
public:
    static constexpr std::size_t row_bytes = sizeof(std::uint64_t) + sizeof(std::uint32_t);
    AccountBatch(std::shared_ptr<memory::Budget> budget, std::size_t capacity);
    AccountBatch(const AccountBatch&) = delete;
    AccountBatch& operator=(const AccountBatch&) = delete;
    AccountBatch(AccountBatch&& other) noexcept;
    AccountBatch& operator=(AccountBatch&& other) noexcept;
    void swap(AccountBatch& other) noexcept;
    void replace_capacity(std::size_t capacity);
    void clear() noexcept { rows_ = 0; }
    void append(Account account);
    Account row(std::size_t index) const;
    AccountColumns columns() const & noexcept;
    AccountColumns columns() const && = delete;
    std::size_t size() const noexcept { return rows_; }
    std::size_t capacity() const noexcept { return capacity_; }
    std::size_t capacity_bytes() const noexcept { return capacity_ * row_bytes; }
    std::shared_ptr<memory::Budget> budget() const noexcept { return ids_.budget(); }
private:
    std::size_t capacity_;
    std::size_t rows_ = 0;
    Column<std::uint64_t> ids_;
    Column<std::uint32_t> regions_;
};

}  // namespace strata::query
#endif
```

The account owner repeats the same small, concrete invariant with two columns.
A general tuple/variant schema framework would add machinery without improving
this fixed-schema execution boundary. The shared `Column` implementation already
centralizes typed allocation, accounting, cleanup, and move behavior.

Finish `batch.cpp`:

```cpp title="src/query/batch.cpp — complete file, segment 4"
AccountBatch::AccountBatch(std::shared_ptr<memory::Budget> budget, std::size_t capacity)
    : capacity_(checked_capacity(capacity, row_bytes)), ids_(budget, capacity_),
      regions_(std::move(budget), capacity_) {}
AccountBatch::AccountBatch(AccountBatch&& other) noexcept
    : capacity_(std::exchange(other.capacity_, 0)), rows_(std::exchange(other.rows_, 0)),
      ids_(std::move(other.ids_)), regions_(std::move(other.regions_)) {}
AccountBatch& AccountBatch::operator=(AccountBatch&& other) noexcept {
    if (this != &other) { AccountBatch candidate{std::move(other)}; swap(candidate); }
    return *this;
}
void AccountBatch::swap(AccountBatch& other) noexcept {
    using std::swap;
    swap(capacity_, other.capacity_); swap(rows_, other.rows_);
    ids_.swap(other.ids_); regions_.swap(other.regions_);
}
void AccountBatch::replace_capacity(std::size_t capacity) {
    AccountBatch candidate{budget(), capacity};
    swap(candidate);
}
void AccountBatch::append(Account account) {
    if (rows_ == capacity_) { throw std::length_error("account batch full"); }
    ids_.storage()[rows_] = account.account_id;
    regions_.storage()[rows_] = account.region_id;
    ++rows_;
}
Account AccountBatch::row(std::size_t index) const {
    if (index >= rows_) { throw std::out_of_range("account batch row"); }
    return {ids_.storage()[index], regions_.storage()[index]};
}
AccountColumns AccountBatch::columns() const & noexcept {
    return {ids_.storage().first(rows_), regions_.storage().first(rows_)};
}

}  // namespace strata::query
```

The account append publishes its count only after both fields are stored, and
account capacity replacement follows the same prepare/swap rule. There are no
per-row allocations in any batch append.

The exact column allocations for capacity $C$ are

$$
M_{\mathrm{event}} = 28C,\qquad
M_{\mathrm{scan}} = 36C,\qquad
M_{\mathrm{account}} = 12C.
$$

These are request sizes charged by our allocator, not process-resident-memory
measurements. Allocator bookkeeping, stack storage, error-message internals,
file-system cache, and the root budget's bootstrap control remain outside this
ledger, as in Chapter 9. Cursor/source allocations remain charged separately.
Keeping several batches alive must sum their allocations; “each batch fits” is
not an aggregate memory bound.

We now have owning output containers, but no producer yet. Continue with
[Part 2 — Pull without confusing emptiness and completion](https://25d-228.github.io/cplusplus-learning/chapter-14-part-2.html).
