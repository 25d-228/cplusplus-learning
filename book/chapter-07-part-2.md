# Chapter 7 — An arena-backed skip-list memtable

## Part 2 — Prepare, publish, and verify

Continue from [Part 1](https://25d-228.github.io/cplusplus-learning/chapter-07.html).
The arena, entry types, memtable declaration, search, and height generator are
already written. We now finish `src/storage/memtable.cpp`, construct the complete
test executable, and append its build target. Both parts share the chapter's
**4.5 implementation hours**. Keep the eight earlier tests.

A prepared batch is not a second visible index. It is an exclusive owner of
unpublished objects and a rollback point. Its lifetime creates a useful boundary:
all validation and allocation happen before publication; no ordinary recoverable
failure remains once publication begins.

| State | Searchable entries | Arena storage | Height-generator state |
|---|---|---|---|
| Before preparation | Committed versions | Committed chunks and prefixes | Committed seed |
| Prepared token alive | Still only committed versions | Old storage plus candidate nodes/values | Committed seed; token holds a private candidate |
| Token abandoned or construction throws | Same committed versions | Original chunks and prefixes restored | Original seed |
| Token committed | Entire batch added | Candidate storage retained | Candidate seed installed |

This is a failure guarantee within **exclusive execution**. It is not a
transaction-isolation implementation, an atomic CPU instruction, or a persistence
protocol. A caller must not read from another thread during either preparation
or publication.

## Validate and sort before constructing nodes

Each batch has one positive sequence number, greater than the table's previous
committed sequence. It contains 1–256 mutations with distinct storage keys.
Multiple versions of a key across batches are allowed; two changes to the same
key inside one batch are rejected. This avoids an undeclared “first or last
operation wins” rule.

The fixed array `order` stores indices into the input, not copies of its encoded
values. Sort only its active prefix. The comparator from Chapter 6 is a
nonthrowing strict order over the validated numeric keys; it does not call user
code. Sorting and duplicate detection still belong to preparation, before we
publish anything.

The factory returns `Prepared{...}` directly even though `Prepared` is neither
copyable nor movable. A same-type class prvalue initializes its destination
directly under the C++17-and-later rules; C++20 supports this. In
`auto prepared = table.prepare(...)`, there is no intermediate token that needs
a move constructor. Do not rewrite the factory as a named local followed by
`return local;`: named-return optimization is a different rule and cannot be
assumed for this immovable type. See
[class initialization](https://eel.is/c++draft/dcl.init#general).

Continue the source namespace left open in Part 1:

```cpp title="src/storage/memtable.cpp"
MemTable::Prepared MemTable::prepare(std::uint64_t sequence,
                                     std::span<const Mutation> changes) & {
    if (frozen_) {
        throw std::logic_error("cannot prepare a frozen memtable");
    }
    return Prepared{*this, sequence, changes};
}

MemTable::Prepared::Prepared(MemTable& table, std::uint64_t sequence,
                             std::span<const Mutation> changes)
    : table_(table), transaction_(table.arena_), sequence_(sequence),
      next_seed_(table.seed_) {
    if (changes.empty() || changes.size() > max_batch) {
        throw std::length_error("batch must contain 1..256 changes");
    }
    if (sequence == 0 || sequence <= table.last_sequence_) {
        throw std::invalid_argument("batch sequence must increase");
    }
    new_size_ = checked_size_add(table.size_, changes.size());
    std::array<std::size_t, max_batch> order{};
    for (std::size_t i = 0; i < changes.size(); ++i) {
        validate_mutation(changes[i]);
        order[i] = i;
    }
    const auto active = std::span<std::size_t>{order}.first(changes.size());
    std::sort(active.begin(), active.end(), [changes](std::size_t a, std::size_t b) {
        return KeyLess{}(changes[a].key, changes[b].key);
    });
    for (std::size_t i = 1; i < active.size(); ++i) {
        if (changes[order[i - 1]].key == changes[order[i]].key) {
            throw std::invalid_argument("duplicate key within batch");
        }
    }
```

Member initialization precedes the constructor body. Thus `transaction_` already
owns the arena checkpoint when a size, sequence, format, or duplicate check
throws. Its destructor restores the transaction flag even though no node was
allocated. A second outstanding preparation fails while constructing that
transaction member; it does not reset the first token's guard.

`checked_size_add` computes the future version count now. Committing must not
perform an overflow check that could throw after linking some nodes. Input
validation also precedes `span::first(change.length)` in the entry validator,
so an invalid length cannot create an out-of-bounds view.

Capturing `changes` by value in the sorting lambda copies a span, not its rows.
The span remains valid throughout this synchronous constructor call. The next
step copies every accepted payload into the arena; the completed token does not
retain the span or refer back to the mutation array.

### Register each lifetime before the next failure point

Node construction is nonthrowing for this exact `Node` type, as checked in Part 1.
The allocation immediately before it can throw. After construction, put the node
on a **pending chain** before allocating its value. If that value allocation
fails, the new node must already be reachable by cleanup.

`pending_next` is not one of the skip-list links. While a token owns a node, all
of its searchable `next` slots are still null. The pending chain has its own
head and tail and cannot be reached by `begin`, `lower_bound`, or `find`.

Finish the constructor:

```cpp title="src/storage/memtable.cpp"
    try {
        for (const auto index : active) {
            const auto& change = changes[index];
            auto* node = std::construct_at(static_cast<Node*>(
                transaction_.allocate(sizeof(Node), alignof(Node))));
            if (last_) { last_->pending_next = node; }
            else { first_ = node; }
            last_ = node;
            node->key = {change.key, sequence};
            node->kind = change.kind;
            node->height = table.choose_height(next_seed_);
            if (change.length != 0) {
                auto* bytes = static_cast<std::byte*>(
                    transaction_.allocate(change.length, alignof(std::byte)));
                for (std::size_t i = 0; i < change.length; ++i) {
                    bytes[i] = change.bytes[i];
                }
                node->value = {bytes, change.length};
            }
        }
    } catch (...) {
        destroy_pending();
        throw;
    }
}
```

A put copies 30 event bytes or 14 account bytes. An erase has no value allocation
and retains the default empty span. Mutation factories construct the usual well-formed values,
but preparation still checks the public mutation fields: a caller can edit those
fields directly.

Notice the order inside the loop: allocate storage, begin node lifetime,
register cleanup, set scalar metadata and height, allocate value storage, then
copy bytes. The potentially failing operations are before publication and are
covered by the pending chain plus arena checkpoint. Scalar assignment, span
assignment, and the byte-copy loop do not allocate.

The catch-all handler performs cleanup and immediately rethrows the original
exception. It does not convert all failures into “not found.” When a constructor
body throws, the class's own destructor is not called; fully constructed members
are destroyed. We therefore need explicit pending-node destruction in the catch,
followed by the transaction member's ordinary rollback during unwinding. The
[constructor-unwinding rules](https://eel.is/c++draft/except.ctor) are the reason
for this division, not an optional defensive convention.

```cpp title="src/storage/memtable.cpp"
void MemTable::Prepared::destroy_pending() noexcept {
    while (first_) {
        Node* next = first_->pending_next;
        std::destroy_at(first_);
        first_ = next;
    }
    last_ = nullptr;
}

MemTable::Prepared::~Prepared() noexcept {
    destroy_pending();
}
```

Saving the next pointer must happen **before** `destroy_at`. Accessing a destroyed
object to find the next node would violate the lifetime boundary. Node destruction
currently releases no separate heap allocation: its payload is a borrowed span
into the same arena. We still end its lifetime explicitly and keep the trivial,
nonthrowing destruction restriction visible in the type assertions.

On ordinary token abandonment, the destructor body first destroys pending nodes.
Only afterward are members destroyed, including `transaction_`, which rewinds or
releases their storage. On constructor failure, the catch does the first part
and member unwinding does the second. On successful commit, the pending chain is
empty and the transaction is disarmed, so neither cleanup path destroys the
published entries.

### Trace a failed preparation

Assume an existing chunk has a committed prefix of $u_0$ bytes and the arena has
reserved $R_0$ bytes. A preparation may consume the rest of that chunk, acquire
another chunk, and then fail while acquiring a second new chunk. The following
sequence is independent of the platform's node size:

1. Destroy every node on the pending chain, including one whose value allocation
   failed. The previously committed nodes are not on that chain.
2. Release each chunk newer than the saved chunk. Restore its cursor to $u_0$.
3. Restore the saved aggregate used count and clear the active transaction flag.

Reserved storage returns to $R_0$ because every newly acquired chunk is released.
Bytes beyond the restored cursor need not be cleared: they are outside the live
allocation prefix and will be overwritten or constructed before reuse. Committed
links and the committed height-generator seed were never changed.

The failure countdown is deliberately diagnostic state, not committed database
state. A triggered injection disarms itself. Retrying without rearming can succeed;
we do not promise to roll back a log counter, diagnostic counter, or allocator's
hidden bookkeeping.

## Publish against the current list, not stale predecessor paths

Preparation knows the nodes' keys, heights, and payloads. It does **not** save a
predecessor path for each node. Those paths could become stale as earlier nodes
in the same batch are linked.

At level zero, consider this initial chain and two prepared keys:

```text title="Interacting insertions at one level"
initial:        head -> 10 -> 30
prepared:               20, 25

publish 20:     head -> 10 -> 20 -> 30
publish 25:     head -> 10 -> 20 -> 25 -> 30
```

Before either publication, both keys have predecessor 10. Reusing that predecessor
for 25 after publishing 20 can either overwrite the new path or create the wrong
order, depending on how the outgoing link is assigned. Sorting the batch alone
does not repair a stale predecessor array.

Instead, search the **current** index afresh for each node. At every level, record
the last node strictly less than the new internal key. `nullptr` means that the
incoming link is the table's head at that level. `link(previous, level)` returns
a reference to that precise pointer slot.

```cpp title="src/storage/memtable.cpp"
void MemTable::publish(Node* node) noexcept {
    std::array<Node*, max_height> previous{};
    Node* at = nullptr;
    for (std::size_t level = max_height; level != 0;) {
        --level;
        Node* next = link(at, level);
        while (next && InternalLess{}(next->key, node->key)) {
            at = next;
            next = link(at, level);
        }
        previous[level] = at;
    }
    for (std::size_t level = 0; level < node->height; ++level) {
        Node*& incoming = link(previous[level], level);
        node->next[level] = incoming;
        incoming = node;
    }
}
```

For a participating level, first set the new node's outgoing link to the old
successor, then redirect the incoming link to the new node. Other links are
unchanged. If the level did not previously contain a node, its head simply
becomes this node. A node of height $h$ participates in indices $0$ through $h-1$;
higher slots stay null.

There is no recoverable failure in this function: the predecessor array is fixed
stack storage, comparison is the built-in numeric policy, and the rest is pointer
assignment. It neither allocates nor invokes callbacks. The earlier uniqueness
checks and strictly increasing batch sequence ensure there is no identical
internal key already in the table. The function therefore does not need to
replace or destroy an existing node.

Correctness here is not a hardware atomicity claim. During these pointer writes,
a concurrent reader would have an unsynchronized access and could observe an
incomplete structure. External exclusion is part of the API contract, not an
optimization that callers may omit.

## Commit the entire prepared chain

After each node is linked, remove its pending-chain pointer. Install the already
checked count, sequence, and candidate generator seed only after the chain has
been exhausted. Finally tell the arena transaction to retain the allocations.

```cpp title="src/storage/memtable.cpp"
void MemTable::Prepared::commit() noexcept {
    if (!first_) {
        return;
    }
    while (first_) {
        Node* node = first_;
        first_ = node->pending_next;
        table_.publish(node);
        node->pending_next = nullptr;
    }
    last_ = nullptr;
    table_.size_ = new_size_;
    table_.last_sequence_ = sequence_;
    table_.seed_ = next_seed_;
    transaction_.commit();
}
```

The first call commits; later calls see an empty chain and do nothing. There is
no “empty prepared batch” ambiguity because empty input was rejected before
construction succeeded. An already committed token may remain in scope without
blocking a later preparation: its arena guard is disarmed.

An exception-safe operation is not automatically crash-safe. A later write-ahead
log will have to record the **same accepted batch** and complete its required
synchronization before this commit. In particular, it must not prepare copied
bytes and then log a caller buffer that has been changed in the meantime. Here
we establish the in-memory boundary only: no log record, disk table, or restart
recovery exists yet.

The `noexcept` declaration expresses an obligation. If a future edit adds a
throwing operation to `commit`, termination would replace rollback; it would not
magically preserve the contract. Keep all allocation, checked arithmetic, fallible
I/O, and extensible callbacks outside this path.

### Freeze without pretending to reclaim versions

A frozen memtable accepts no further preparation. Freezing while a token exists
is rejected so that “frozen” cannot later acquire hidden new entries. Repeated
freeze calls are harmless.

The height histogram reports the number of nodes with each exact height: element
zero counts height one, not all members of level zero. It is a diagnostic walk,
not a cached concurrent metric.

```cpp title="src/storage/memtable.cpp"
void MemTable::freeze() {
    if (arena_.transaction_active()) {
        throw std::logic_error("cannot freeze during preparation");
    }
    frozen_ = true;
}

std::array<std::size_t, MemTable::max_height>
MemTable::height_counts() const noexcept {
    std::array<std::size_t, max_height> result{};
    for (const Node* node = heads_[0]; node; node = node->next[0]) {
        ++result[node->height - 1];
    }
    return result;
}
```

All versions, including tombstones, remain until the table is destroyed. There
is no per-key deletion of old nodes and no recycling of committed arena space.
When a cap prevents preparation, a later engine layer will rotate or flush the
table; this layer neither evicts silently nor exceeds its cap. `freeze` alone
provides neither shared ownership nor synchronization for readers in other
threads.

## Verify the structure, not only successful lookups

A corrupt upper level might leave small level-zero scans looking correct. We
need a stronger check than “the last inserted key can be found.”

First traverse level zero. Require strictly increasing internal keys, legal
heights and sequences, no pending links on visible nodes, and no links above a
node's height. The published count bounds traversal: reaching an additional node
when `seen == size_` detects an overlong chain, including a cycle, before another
iteration. At the end, the count must match exactly.

```cpp title="src/storage/memtable.cpp"
void MemTable::check_structure() const {
    std::size_t seen = 0;
    const Node* previous = nullptr;
    for (const Node* node = heads_[0]; node; node = node->next[0]) {
        if (seen == size_ || node->height == 0 || node->height > max_height ||
            node->key.sequence == 0 || node->key.sequence > last_sequence_ ||
            (previous && !InternalLess{}(previous->key, node->key)) ||
            node->pending_next != nullptr) {
            throw std::logic_error("invalid level-zero chain");
        }
        ++seen;
        for (std::size_t level = node->height; level < max_height; ++level) {
            if (node->next[level]) {
                throw std::logic_error("link above node height");
            }
        }
        previous = node;
    }
    if (seen != size_) {
        throw std::logic_error("memtable size disagrees with chain");
    }
```

Now fix an upper-level index $i$. Its complete chain must equal the level-zero
subsequence whose nodes have height greater than $i$. Merely checking that an
upper-level node also appears somewhere below would miss absent eligible nodes.
Compare pointers to the required subsequence in order, then reject extra tail
nodes:

```cpp title="src/storage/memtable.cpp"
    for (std::size_t level = 1; level < max_height; ++level) {
        const Node* actual = heads_[level];
        for (const Node* node = heads_[0]; node; node = node->next[0]) {
            if (node->height > level) {
                if (actual != node) {
                    throw std::logic_error("upper level is not the required subsequence");
                }
                actual = actual->next[level];
            }
        }
        if (actual) {
            throw std::logic_error("extra upper-level node");
        }
    }
}

}  // namespace strata::storage
```

With $n$ versions and a height cap $H$, the diagnostic performs at most a constant
multiple of $nH$ steps. Here $H=16$. It is intended for tests and deliberate
inspection, not every production read. It assumes pointers refer to live nodes
owned by this table; it cannot safely validate arbitrary corrupted addresses.
Sanitizers and the semantic checker answer different questions.

`height_counts` and this checker use ordinary pointers. Call them under the same
exclusive-access rule as the rest of the mutable table. Do not mistake a const
member function for a thread-safety guarantee.

## Build a test program with active checks

Create `tests/memtable.cpp`. The complete program below tests the arena separately
before composing it with the memtable. It also contains a standard-map oracle,
but that container is **test-only**: no engine operation delegates its ordering
or storage to it.

`require` throws instead of using `assert`, so Release builds retain every check.
`expect_error<Exception>` catches only the specified exception family; an unrelated
exception escapes and fails the executable. The callable is deduced through the
function template, as in Chapter 5. `apply` also calls `commit` twice to test its
idempotent completed state.

```cpp title="tests/memtable.cpp"
#include "strata/core/checked.hpp"
#include "strata/memory/arena.hpp"
#include "strata/storage/memtable.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <exception>
#include <iostream>
#include <limits>
#include <map>
#include <memory>
#include <new>
#include <span>
#include <stdexcept>
#include <string>
#include <tuple>
#include <type_traits>
#include <utility>
#include <vector>

namespace {
using namespace strata;
using namespace strata::storage;
using memory::Arena;

void require(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}

template<class Exception, class Function>
void expect_error(const char* message, Function action) {
    try { action(); }
    catch (const Exception&) { return; }
    throw std::runtime_error(message);
}

std::size_t budget(std::size_t chunks) {
    return checked_size_mul(chunks, Arena::chunk_charge());
}

void apply(MemTable& table, std::uint64_t sequence,
           std::span<const Mutation> changes) {
    auto prepared = table.prepare(sequence, changes);
    prepared.commit();
    prepared.commit();
    table.check_structure();
}
```

### An object can own a lifetime without owning its storage

`Tracked` increments a live-object counter during construction and decrements it
at destruction. Its bytes belong to the arena, so an ordinary `delete` would be
wrong. `DestroyOnly` is a custom `unique_ptr` deleter: it ends the object's lifetime
with `destroy_at` but does not free the chunk containing it. The unique pointer
owns **the destruction obligation**, while the arena owns the allocation.

The second template parameter of `std::unique_ptr<T, D>` is the deleter type. On
destruction of a non-null owner, it calls that deleter. This is the same unique
ownership mechanism from Chapter 3 with a different cleanup operation; see the
[unique-pointer destructor contract](https://eel.is/c++draft/unique.ptr.single.dtor).

Start the fixture by committing five bytes and retaining a pointer to its first
byte. That odd-sized prefix makes the next aligned allocation exercise padding:

```cpp title="tests/memtable.cpp"
struct Tracked {
    int& live;
    std::uint64_t value;
    Tracked(int& count, std::uint64_t initial) noexcept : live(count), value(initial) {
        ++live;
    }
    ~Tracked() noexcept { --live; }
};

struct DestroyOnly {
    void operator()(Tracked* object) const noexcept { std::destroy_at(object); }
};

void check_arena() {
    Arena arena{budget(3)};
    std::byte* original = nullptr;
    {
        Arena::Transaction transaction{arena};
        original = static_cast<std::byte*>(transaction.allocate(5, 1));
        original[0] = std::byte{77};
        transaction.commit();
        expect_error<std::logic_error>("allocated through completed transaction", [&] {
            transaction.allocate(1, 1);
        });
    }
```

Declare the lifetime guard **after** the arena transaction. Reverse local
destruction order then destroys the tracked object before the transaction
rewinds its storage. This is an ownership relationship expressed through scope,
not through a comment that cleanup “should happen eventually.”

The test casts an address to `std::uintptr_t` only to inspect its alignment on our
native target. `reinterpret_cast` here exposes the address representation; it
neither begins an object lifetime nor makes an arbitrary pointer safe. The
allocator itself uses byte-array offsets, not integer-to-pointer fabrication.

Continue the function:

```cpp title="tests/memtable.cpp"
    const auto before_reserved = arena.reserved_bytes();
    const auto before_used = arena.used_bytes();
    int live = 0;
    {
        Arena::Transaction transaction{arena};
        auto* raw = transaction.allocate(sizeof(Tracked), alignof(Tracked));
        require(reinterpret_cast<std::uintptr_t>(raw) % alignof(Tracked) == 0,
                "misaligned object storage");
        std::unique_ptr<Tracked, DestroyOnly> object{
            std::construct_at(static_cast<Tracked*>(raw), live, 42)};
        require(live == 1 && object->value == 42, "placement lifetime did not start");
        transaction.allocate(Arena::chunk_payload, 1);
        require(arena.reserved_bytes() == budget(2), "second chunk missing");
        expect_error<std::logic_error>("nested transaction accepted", [&] {
            Arena::Transaction nested{arena};
        });
    }
    require(live == 0 && arena.reserved_bytes() == before_reserved &&
            arena.used_bytes() == before_used && original[0] == std::byte{77},
            "arena rollback lost old bytes or retained new storage");
    {
        Arena::Transaction transaction{arena};
        expect_error<std::invalid_argument>("bad alignment accepted", [&] {
            transaction.allocate(8, 3);
        });
        expect_error<std::invalid_argument>("zero alignment accepted", [&] {
            transaction.allocate(8, 0);
        });
        expect_error<std::invalid_argument>("extended alignment accepted", [&] {
            transaction.allocate(8, 2 * alignof(std::max_align_t));
        });
        expect_error<std::length_error>("zero size accepted", [&] {
            transaction.allocate(0, 1);
        });
        expect_error<std::length_error>("oversized request accepted", [&] {
            transaction.allocate(Arena::chunk_payload + 1, 1);
        });
    }
```

The transaction's full-payload request cannot fit after the existing prefix and
the tracked object, so it must acquire a second chunk. On leaving the scope,
`live` returns to zero, both accounting values return to their previous values,
and the committed sentinel still equals 77.

Validation failures use the same transaction without consuming bytes. The
remaining checks fill exactly one payload, reject growth beyond its charge,
and reject any allocation from a zero-cap arena:

```cpp title="tests/memtable.cpp"
    Arena tiny{budget(1)};
    {
        Arena::Transaction transaction{tiny};
        transaction.allocate(Arena::chunk_payload, 1);
        transaction.commit();
    }
    expect_error<memory::ArenaFull>("byte cap ignored", [&] {
        Arena::Transaction transaction{tiny};
        transaction.allocate(1, 1);
    });
    require(tiny.reserved_bytes() == budget(1) && tiny.used_bytes() == 4096,
            "failed cap check changed accounting");
    Arena empty{0};
    expect_error<memory::ArenaFull>("zero cap allocated", [&] {
        Arena::Transaction transaction{empty};
        transaction.allocate(1, 1);
    });
    std::cout << "arena: aligned lifetime, rollback, validation, byte cap checked\n";
}
```

A zero-byte allocation is rejected by design, rather than returning a special
pointer with unclear lifetime. Empty tombstones do not call the allocator for
value storage. Requests larger than one payload are also rejected; this arena
serves small nodes and the fixed-size row images, not arbitrary large blobs.

### Follow versions, borrowed values, and the preparation boundary

Start with three deliberately unsorted mutations: event 30, event 10, and account
10. Their storage keys differ even when their numeric primary keys match. The
next batch inserts events 25 and 20 and writes a tombstone for event 10.

While the token is alive, neither the size nor a lookup of event 20 changes. A
nested preparation and freeze both fail. Then overwrite the caller's mutation
for event 25 **after** preparation. Committing must still install the copied put,
not the caller's new erase:

```cpp title="tests/memtable.cpp"
void check_versions() {
    MemTable table{budget(8), 9};
    require(!table.begin().valid(), "new table is not empty");
    auto end = table.begin();
    end.next();
    expect_error<std::out_of_range>("end cursor yielded entry", [&] { end.entry(); });
    std::array first{Mutation::put(Event{30, 3, 1, 300}),
                     Mutation::put(Event{10, 1, 1, 100}),
                     Mutation::put(Account{10, 7})};
    apply(table, 1, first);
    const auto old = table.find({Relation::events, 10});
    require(old && decode_event(old->value).amount_cents == 100, "old put missing");
    const auto before = table.size();
    std::array second{Mutation::put(Event{25, 2, 1, 250}),
                      Mutation::erase({Relation::events, 10}),
                      Mutation::put(Event{20, 2, 1, 200})};
    {
        auto prepared = table.prepare(2, second);
        require(table.size() == before && !table.find({Relation::events, 20}),
                "prepared nodes became visible");
        table.check_structure();
        expect_error<std::logic_error>("freeze accepted outstanding preparation", [&] {
            table.freeze();
        });
        expect_error<std::logic_error>("second preparation accepted", [&] {
            auto nested = table.prepare(2, second);
        });
        second[0] = Mutation::erase({Relation::events, 25});
        prepared.commit();
        prepared.commit();
    }
```

The next block checks the complete order, not just the newly inserted keys. For
event 10, sequence 2 comes before sequence 1. An unrestricted lookup returns the
tombstone; a lookup at visible sequence 1 returns the old put with its original
payload address. At visible sequence 0, no version qualifies.

```cpp title="tests/memtable.cpp"
    table.check_structure();
    const auto deleted = table.find({Relation::events, 10});
    require(deleted && deleted->kind == EntryKind::erase && deleted->value.empty(),
            "tombstone is not distinct from absence");
    const auto historical = table.find({Relation::events, 10}, 1);
    require(historical && historical->value.data() == old->value.data() &&
            decode_event(old->value).amount_cents == 100, "older version invalidated");
    require(!table.find({Relation::events, 10}, 0) &&
            !table.find({Relation::events, 99}), "absence lookup failed");
    const auto prepared_copy = table.find({Relation::events, 25});
    require(prepared_copy && prepared_copy->kind == EntryKind::put &&
            decode_event(prepared_copy->value).amount_cents == 250,
            "prepared batch retained caller storage");
    const std::array<InternalKey, 6> expected{{
        {{Relation::events, 10}, 2}, {{Relation::events, 10}, 1},
        {{Relation::events, 20}, 2}, {{Relation::events, 25}, 2},
        {{Relation::events, 30}, 1}, {{Relation::accounts, 10}, 1}}};
    auto cursor = table.begin();
    for (const auto& key : expected) {
        require(cursor.valid() && cursor.entry().key == key, "wrong version order");
        cursor.next();
    }
    require(!cursor.valid(), "extra node in iteration");
    auto gap = table.lower_bound({{Relation::events, 21}, 2});
    require(gap.valid() && gap.entry().key.user.primary == 25, "gap seek failed");
    table.freeze();
    table.freeze();
    expect_error<std::logic_error>("frozen table accepted write", [&] {
        auto rejected = table.prepare(3, second);
    });
    require(table.frozen(), "freeze flag lost");
    std::cout << "versions: 10@2 erase, 10@1 put, 20@2, 25@2, 30@1, account10@1\n";
}
```

Returning a tombstone is different from returning `nullopt`. A future lookup
across multiple tables must stop when it finds a visible tombstone, rather than
continue into an older table and resurrect a deleted value. This chapter does
not implement that multi-table lookup, but it provides the information needed
for the correct decision.

The borrowed old value survives insertion because its allocation is not moved
or freed. It does not keep the table alive. Destroying the table still invalidates
it, and retaining a cursor while committing does not create a consistent scan
snapshot. Reacquire a cursor after a commit unless an external protocol guarantees
that its iteration semantics are acceptable.

### Reject malformed requests before allocating

The public `Mutation` representation makes explicit validation essential. The
next function checks duplicate keys, empty or oversized batches, a zero sequence,
a disagreement between metadata and encoded key, an excessive value length, and
an invalid row-format version.

```cpp title="tests/memtable.cpp"
void check_validation() {
    MemTable table{budget(8)};
    std::array good{Mutation::put(Event{0, 0, 0, 0})};
    std::array duplicate{good[0], Mutation::erase({Relation::events, 0})};
    expect_error<std::invalid_argument>("duplicate key accepted", [&] {
        auto rejected = table.prepare(1, duplicate);
    });
    expect_error<std::length_error>("empty batch accepted", [&] {
        auto rejected = table.prepare(1, {});
    });
    std::array<Mutation, MemTable::max_batch + 1> oversized{};
    expect_error<std::length_error>("oversized batch accepted", [&] {
        auto rejected = table.prepare(1, oversized);
    });
    expect_error<std::invalid_argument>("sequence zero accepted", [&] {
        auto rejected = table.prepare(0, good);
    });
    auto wrong = good;
    wrong[0].key.primary = 99;
    expect_error<std::invalid_argument>("key/image mismatch accepted", [&] {
        auto rejected = table.prepare(1, wrong);
    });
    wrong = good;
    wrong[0].length = event_bytes + 1;
    expect_error<std::invalid_argument>("out-of-range length accepted", [&] {
        auto rejected = table.prepare(1, wrong);
    });
    wrong = good;
    wrong[0].bytes[0] = std::byte{2};
    expect_error<FormatError>("bad row version accepted", [&] {
        auto rejected = table.prepare(1, wrong);
    });
```

Continue with malformed tombstones, unknown enumeration values, repeated
sequences, and maximum unsigned fields:

```cpp title="tests/memtable.cpp"
    wrong = good;
    wrong[0].kind = EntryKind::erase;
    expect_error<std::invalid_argument>("tombstone value accepted", [&] {
        auto rejected = table.prepare(1, wrong);
    });
    wrong = good;
    wrong[0].kind = static_cast<EntryKind>(99);
    expect_error<std::invalid_argument>("unknown kind accepted", [&] {
        auto rejected = table.prepare(1, wrong);
    });
    wrong = good;
    wrong[0].key.relation = static_cast<Relation>(99);
    expect_error<std::invalid_argument>("unknown relation accepted", [&] {
        auto rejected = table.prepare(1, wrong);
    });
    require(table.size() == 0 && table.reserved_bytes() == 0,
            "early validation allocated or published");
    apply(table, 1, good);
    expect_error<std::invalid_argument>("reused sequence accepted", [&] {
        auto rejected = table.prepare(1, good);
    });
    const auto maximum = std::numeric_limits<std::uint64_t>::max();
    std::array high{Mutation::put(Event{maximum, maximum, 0, maximum})};
    apply(table, maximum, high);
    const auto found = table.find({Relation::events, maximum});
    require(found && found->key.sequence == maximum &&
            decode_event(found->value).amount_cents == maximum, "maximum key failed");
    expect_error<std::invalid_argument>("sequence wrap accepted", [&] {
        auto rejected = table.prepare(0, good);
    });
    std::cout << "validation: duplicates, malformed values, limits, sequence edges checked\n";
}
```

`UINT64_MAX` is a valid key and sequence value. Accepting the maximum sequence
does not permit the next sequence to wrap to zero; the engine will eventually
need a terminal sequence-exhaustion policy. This table enforces strict increase
without computing `last_sequence + 1`, so the validation itself cannot wrap.

No test fabricates an enormous span to exercise overflow. Scalar limits and the
bounded batch array provide valid objects throughout. Earlier codec tests already
cover exact row lengths and all truncated prefixes; these tests add the mutation
boundary rather than duplicating every decoder case.

### Fail after some preparation work has already succeeded

A failed first allocation is useful but insufficient. It says little about
cleanup after a partially built candidate. Keep one committed event, save its
payload address and arena charges, then prepare 64 new events. The batch is large
enough to need multiple chunks for this fixed node layout.

Inject failure at the first and second **new chunk** acquisitions. Before either
failure, allocations may already have succeeded in the tail of the existing
chunk. Before the second failure, an entire new chunk has also been acquired.
Both attempts must restore the old entries and exact accounting:

```cpp title="tests/memtable.cpp"
void check_rollback() {
    MemTable table{budget(32), 123};
    MemTable reference{budget(32), 123};
    std::array initial{Mutation::put(Event{1, 1, 1, 11})};
    apply(table, 1, initial);
    apply(reference, 1, initial);
    const auto old = table.find({Relation::events, 1});
    const auto reserved = table.reserved_bytes();
    const auto used = table.used_bytes();
    std::array<Mutation, 64> batch{};
    for (std::size_t i = 0; i < batch.size(); ++i) {
        batch[i] = Mutation::put(Event{100 + i, 1, 1, i});
    }
    for (const auto ordinal : {std::size_t{1}, std::size_t{2}}) {
        table.fail_on_chunk(ordinal);
        expect_error<std::bad_alloc>("injected chunk allocation did not fail", [&] {
            auto rejected = table.prepare(2, batch);
        });
        table.check_structure();
        require(table.size() == 1 && table.last_sequence() == 1 &&
                table.reserved_bytes() == reserved && table.used_bytes() == used &&
                table.find({Relation::events, 1})->value.data() == old->value.data() &&
                decode_event(old->value).amount_cents == 11,
                "failed preparation changed committed state");
    }
```

Also abandon a completely prepared batch without calling `commit`. Then retry
successfully and compare its height histogram and used-byte count with a clean
table that never experienced the failed attempts. Finally exercise ordinary cap
exhaustion, which is a different error from the injected allocator failure:

```cpp title="tests/memtable.cpp"
    {
        auto abandoned = table.prepare(2, batch);
        require(!table.find({Relation::events, 100}), "abandoned batch visible");
    }
    require(table.reserved_bytes() == reserved && table.used_bytes() == used,
            "abandoned batch kept its arena storage");
    apply(table, 2, batch);
    apply(reference, 2, batch);
    require(table.height_counts() == reference.height_counts() &&
            table.used_bytes() == reference.used_bytes(), "rollback changed retry shape");
    MemTable capped{budget(1)};
    apply(capped, 1, initial);
    const auto capped_used = capped.used_bytes();
    expect_error<memory::ArenaFull>("memtable cap did not reject batch", [&] {
        auto rejected = capped.prepare(2, batch);
    });
    require(capped.size() == 1 && capped.used_bytes() == capped_used &&
            capped.reserved_bytes() == budget(1), "cap failure was not rolled back");
    capped.check_structure();
    std::cout << "preparation: first/second new-chunk failure and abandonment roll back\n";
}
```

The histogram and charge comparison check that retries do not consume the
committed height sequence or retain padding from an abandoned attempt. A matching
histogram alone would not prove every node received the same height; the stronger
reason is visible in the implementation: preparation changes only `next_seed_`,
and only commit assigns it to `table_.seed_`.

These failures are deterministic test seams. They do not claim to reproduce all
system allocator failures or operating-system out-of-memory behavior. Their role
is to execute real cleanup paths at controlled points, including after partial
success.

## Compare against an independently expressed order

The oracle uses `std::map` with a tuple key. A tuple stores a fixed heterogeneous
sequence of values; `std::get<0>(t)` and `std::get<1>(t)` select its first two
components at compile time. Tuple comparisons are lexicographic. A map maintains
one value for each ordered key and provides ordered iteration and `lower_bound`.
See [tuple element access](https://eel.is/c++draft/tuple.elem),
[tuple comparisons](https://eel.is/c++draft/tuple.rel), and the
[map interface](https://eel.is/c++draft/map.overview).

Do not give the map our `InternalLess`. Sharing that comparator could let the
implementation and oracle agree on the same mistake. Instead, represent an
internal key as

$$
(\text{relation},\ \text{primary},\ M-\text{sequence}),
\qquad M=2^{64}-1.
$$

Each component uses ordinary ascending comparison. For unsigned sequences
$s_1>s_2$, we have $M-s_1<M-s_2$, with no underflow because both sequences lie
between zero and $M$. Thus newer versions sort first without reusing the
memtable's comparator code.

The map owns a copy of every expected byte sequence. `Reference::value_type`
names its element type, a pair containing a const key and its associated value.
Comparing it with an `EntryView` tests metadata, value length, and each byte:

```cpp title="tests/memtable.cpp"
using RefKey = std::tuple<std::uint8_t, std::uint64_t, std::uint64_t>;
struct RefValue {
    EntryKind kind;
    std::vector<std::byte> bytes;
};
using Reference = std::map<RefKey, RefValue>;

RefKey reference_key(Key key, std::uint64_t sequence) {
    return {static_cast<std::uint8_t>(key.relation), key.primary,
            std::numeric_limits<std::uint64_t>::max() - sequence};
}

void compare_entry(EntryView actual, const Reference::value_type& expected) {
    require(reference_key(actual.key.user, actual.key.sequence) == expected.first &&
            actual.kind == expected.second.kind &&
            actual.value.size() == expected.second.bytes.size(), "oracle metadata mismatch");
    for (std::size_t i = 0; i < actual.value.size(); ++i) {
        require(actual.value[i] == expected.second.bytes[i], "oracle payload mismatch");
    }
}
```

### A reproducible mixed workload

Use 300 batches with three distinct storage keys each. There are 64 possible keys:
events 0–31 and accounts 0–31. Offsets 0, 13, and 26 modulo 64 make the three keys
within a batch distinct. Every fifth operation by the stated arithmetic pattern
is an erase; other operations alternate between the two row types according to
the key's relation.

The unsigned 32-bit recurrence deliberately wraps modulo $2^{32}$ on the target.
It is a deterministic fixture generator, not a claim about statistical quality.
The skip-list's height seed is separate, so record selection does not depend on
how many height bits the index consumes.

`map::emplace` constructs an entry from its key and value. The vector in `RefValue`
uses the iterator-range constructor to copy the encoded bytes; it does not borrow
the batch array. Continue the test file:

```cpp title="tests/memtable.cpp"
void check_randomized() {
    MemTable table{budget(128), 0x12345678U};
    Reference reference;
    std::uint32_t random = 7;
    for (std::uint64_t sequence = 1; sequence <= 300; ++sequence) {
        random = random * 1664525U + 1013904223U;
        std::array<Mutation, 3> batch{};
        for (std::size_t j = 0; j < batch.size(); ++j) {
            const auto slot = (static_cast<std::size_t>(random % 64U) + 13 * j) % 64;
            const auto relation = slot < 32 ? Relation::events : Relation::accounts;
            const auto primary = static_cast<std::uint64_t>(slot % 32);
            if ((sequence + j) % 5 == 0) {
                batch[j] = Mutation::erase({relation, primary});
            } else if (relation == Relation::events) {
                batch[j] = Mutation::put(Event{primary, primary, 3, sequence});
            } else {
                batch[j] = Mutation::put(Account{primary, static_cast<std::uint32_t>(sequence)});
            }
        }
        apply(table, sequence, batch);
        for (const auto& change : batch) {
            const auto image = std::span<const std::byte>{change.bytes}.first(change.length);
            reference.emplace(reference_key(change.key, sequence),
                              RefValue{change.kind, {image.begin(), image.end()}});
        }
```

After **every** batch, compare the entire ordered iteration and every key at
three visible-sequence bounds: zero, halfway to the current sequence, and the
current sequence. A map lower bound is only a candidate: its relation and primary
must still match, or the requested key is absent.

```cpp title="tests/memtable.cpp"
        auto cursor = table.begin();
        for (const auto& expected : reference) {
            require(cursor.valid(), "oracle iteration ended early");
            compare_entry(cursor.entry(), expected);
            cursor.next();
        }
        require(!cursor.valid() && table.size() == reference.size(), "oracle size mismatch");
        for (std::uint64_t slot = 0; slot < 64; ++slot) {
            const Key key{slot < 32 ? Relation::events : Relation::accounts, slot % 32};
            const std::array<std::uint64_t, 3> views{0, sequence / 2, sequence};
            for (const auto visible : views) {
                const auto expected = reference.lower_bound(reference_key(key, visible));
                const bool present = expected != reference.end() &&
                    std::get<0>(expected->first) == static_cast<std::uint8_t>(key.relation) &&
                    std::get<1>(expected->first) == key.primary;
                const auto actual = table.find(key, visible);
                require(actual.has_value() == present, "oracle lookup presence mismatch");
                if (present) { compare_entry(*actual, *expected); }
            }
        }
        require(table.used_bytes() <= table.reserved_bytes() &&
                table.reserved_bytes() <= budget(128), "arena charge exceeded cap");
    }
    const auto heights = table.height_counts();
    require(heights[0] != 0 && heights[1] != 0, "seed exercised no mixed heights");
    std::cout << "oracle: 300 batches, 900 versions, 57600 lookups agree\n";
    std::cout << "layout: node=" << MemTable::node_bytes()
              << " chunk-charge=" << Arena::chunk_charge()
              << " reserved=" << table.reserved_bytes()
              << " used=" << table.used_bytes() << '\n';
}
```

The final table contains $300\times3=900$ versions, not necessarily 900 live user
rows. The point checks total $300\times64\times3=57{,}600$. Iteration comparisons
and structural checks are additional work. Reusing a row's encoded bytes in the
oracle does not independently test the codec; Chapter 6's Python decoder covers
that separate boundary.

The printed sizes make physical cost visible. An example from a Linux x86-64
Clang/libstdc++ execution was:

```text title="Example layout observation — not a required native result"
layout: node=192 chunk-charge=4112 reserved=197376 used=190002
```

For those measurements, 48 chunks cost $48\times4112=197{,}376$ charged bytes.
The difference between reserved and used is $7{,}374$ bytes. It contains chunk
metadata/padding and unconsumed payload suffixes. Used bytes already include
node-internal padding and inter-allocation alignment gaps, so “used” is not
synonymous with application payload.

The fixed tower alone stores 16 pointers per node. It pays for transparent,
ordinary object layout even when a node participates in only one or two levels.
A future compact layout would need a separate, correct lifetime and allocation
design; indexing past a one-element member is not that design. The native target
may report different sizes. Compare invariants and printed observations, not a
hard-coded ABI assumption. This fixture is a correctness and accounting check,
not a throughput benchmark or a 16 GB whole-process measurement.

### Force both height extremes

Generated mixed heights exercise typical paths, but they need not produce the
maximum tower in a small run. Explicitly run the same insertion/lookup checks
with all heights equal to one and then all equal to 16. The latter makes every
level contain every node; the former reduces the index to one linked list.

```cpp title="tests/memtable.cpp"
void check_heights() {
    for (const auto height : {std::size_t{1}, MemTable::max_height}) {
        MemTable table{budget(4), 1, height};
        std::array<Mutation, 12> batch{};
        for (std::size_t i = 0; i < batch.size(); ++i) {
            batch[i] = Mutation::put(Event{batch.size() - i, 0, 0, i});
        }
        apply(table, 1, batch);
        require(table.height_counts()[height - 1] == batch.size(), "forced height ignored");
        for (std::uint64_t key = 1; key <= batch.size(); ++key) {
            require(table.find({Relation::events, key}).has_value(), "height-edge lookup failed");
        }
    }
    expect_error<std::invalid_argument>("zero seed accepted", [] { MemTable table{1, 0}; });
    expect_error<std::invalid_argument>("excessive height accepted", [] {
        MemTable table{1, 1, MemTable::max_height + 1};
    });
    std::cout << "towers: height 1 and height 16 have exact upper-level membership\n";
}
}  // namespace
```

Before `main`, understand its two compile-time checks. The `_v` type-trait helpers
from `<type_traits>` expose Boolean properties. The first rejects accidental
movability of the prepared token. The second uses `noexcept(expression)` to ask
whether calling `commit` can propagate an exception according to its declaration.
`std::declval<T>()` supplies an expression with type `T` in this **unevaluated**
context; no token is constructed or function executed. It must not be called as
ordinary runtime code. See [type properties](https://eel.is/c++draft/meta.unary.prop)
and [`declval`](https://eel.is/c++draft/declval).

```cpp title="tests/memtable.cpp"
int main() {
    static_assert(!std::is_move_constructible_v<MemTable::Prepared>);
    static_assert(noexcept(std::declval<MemTable::Prepared&>().commit()));
    try {
        check_arena();
        check_versions();
        check_validation();
        check_rollback();
        check_randomized();
        check_heights();
        std::cout << "memtable checks passed\n";
        return 0;
    } catch (const std::exception& error) {
        std::cerr << "FAIL " << error.what() << '\n';
        return 1;
    }
}
```

The static assertion checks the declared contract, not the function body's
semantic safety. The failure tests and the audit of operations inside commit are
still necessary. Adding `noexcept` to code with hidden allocation would not be a
substitute for preparing that allocation earlier.

## Connect the engine sources and run all tests

Append this block to the existing root `CMakeLists.txt`. Do not replace the eight
previous test registrations or create a second `strata-core` target.
[`target_sources`](https://cmake.org/cmake/help/latest/command/target_sources.html)
adds the two implementation files to the existing library. The new test links
that same library and inherits its public include directory and C++20 requirement.

```cmake title="CMakeLists.txt"
target_sources(strata-core PRIVATE
    src/memory/arena.cpp
    src/storage/memtable.cpp)
add_executable(strata-memtable-tests tests/memtable.cpp)
target_link_libraries(strata-memtable-tests PRIVATE strata-core)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(strata-memtable-tests PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion)
endif()
add_test(NAME memtable COMMAND strata-memtable-tests)
```

From the learner workspace root, retain the native compiler selection from
Chapter 6. On macOS, `xcrun --find clang++` selects the installed developer-tool
compiler; an explicitly chosen compatible compiler may be supplied through `CXX`.
The chapter requires the C++20 lifetime facilities shown here, not a particular
marketing version or an invented compiler LTS designation. Keep a build
directory's compiler consistent; use a fresh directory when changing compilers.

```sh title="Terminal — Debug and Release checks"
export CXX="${CXX:-$(xcrun --find clang++)}"
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_COMPILER="$CXX"
cmake --build build/debug -j 4
ctest --test-dir build/debug --output-on-failure
./build/debug/strata-memtable-tests

cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_COMPILER="$CXX"
cmake --build build/release -j 4
ctest --test-dir build/release --output-on-failure
```

On another compatible authoring/test system, set `CXX` to that system's compiler
first; `xcrun` is a macOS tool. These commands should produce **nine** passing
CTest entries. The new `memtable` entry has this output, with the four layout
numbers replaced by the measurements from your executable:

```text title="Expected checks — the layout line is target-dependent"
arena: aligned lifetime, rollback, validation, byte cap checked
versions: 10@2 erase, 10@1 put, 20@2, 25@2, 30@1, account10@1
validation: duplicates, malformed values, limits, sequence edges checked
preparation: first/second new-chunk failure and abandonment roll back
oracle: 300 batches, 900 versions, 57600 lookups agree
layout: node=<bytes> chunk-charge=<bytes> reserved=<bytes> used=<bytes>
towers: height 1 and height 16 have exact upper-level membership
memtable checks passed
```

The angle-bracket fields are explanatory output placeholders, not source to paste
into the program. Everything needed to print their actual values is in the test
listing. Preserve the ordinary foundation and native file-I/O checks: a new index
must not silently replace the earlier coverage.

### Run the lifetime and bounds instrumentation

Build a focused instrumented executable from the same two new library sources
and the complete test file. The record codec and checked helpers are header-only;
this executable does not perform file I/O and therefore does not need to link
`src/io/file.cpp` separately.

```sh title="Terminal — focused AddressSanitizer and UndefinedBehaviorSanitizer run"
mkdir -p build/sanitize
"$CXX" -std=c++20 -g -O1 -fno-omit-frame-pointer \
  -fsanitize=address,undefined \
  -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
  -I include src/memory/arena.cpp src/storage/memtable.cpp \
  tests/memtable.cpp -o build/sanitize/strata-memtable-tests
./build/sanitize/strata-memtable-tests
```

Expect the normal checks and no sanitizer diagnostic. A supported native Clang
runtime is required. Leak-detection availability and defaults differ by platform;
normal correctness must not depend on enabling a particular leak option. Consult
[AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) and
[UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)
for the actual tool behavior.

## Introduce one legal-memory, wrong-structure defect

Temporarily edit **only the publication loop** in `MemTable::publish`: change its
upper bound from `node->height` to `1`. Do not change height generation, the
predecessor search, or the structural checker. This publishes every node at
level zero but omits required upper-level links.

Rebuild both ordinary configurations and run only the new test:

```sh title="Terminal — the deliberately defective runs must fail"
cmake --build build/debug -j 4
ctest --test-dir build/debug -R '^memtable$' --output-on-failure
cmake --build build/release -j 4
ctest --test-dir build/release -R '^memtable$' --output-on-failure
```

Expect a nonzero test result containing
`FAIL upper level is not the required subsequence`. The first arena check can
still succeed. The same focused sanitizer build can report the logical failure
without a memory-error diagnostic: the pointers accessed are legal, but the
required index topology is wrong.

Restore `node->height`, rebuild, and rerun the full nine-test suite in both
configurations. The passing restored implementation, not the defective variation,
is the prerequisite for the next chapter. This exercise explains why neither
sanitizer silence nor a few successful point lookups proves an index correct.

## Diagnose disagreements at the boundary that owns them

| Symptom | Inspect first | Reason |
|---|---|---|
| Newest lookup returns an older put | `InternalLess`, then the visible-sequence lower bound | User keys ascend, but sequences descend; a tombstone is still an entry. |
| A batch loses one of two nearby keys | Predecessor search inside each `publish` call | Earlier nodes in the same batch change later predecessor paths. |
| Failure restores entries but keeps extra charges | Pending cleanup followed by the arena checkpoint | Unpublished nodes must die before their chunks are released or their cursor is rewound. |
| An old view fails after an otherwise valid insertion | Chunk stability and the lifetime of the table | Insertions preserve old storage; table destruction does not. |
| Lookups seem plausible but the checker fails | Exact upper-level subsequences and unused slots | A legal address does not establish correct level membership. |
| Release passes a deliberately broken invariant | Whether checks use `require`, not disabled `assert` | Test obligations must survive optimization and build-mode changes. |

For a small debugger trace, break in `MemTable::publish` while running
`strata-memtable-tests`, inspect `node->key`, and follow `previous[0]`,
`node->next[0]`, and `heads_[0]` through the two pointer assignments. Use the
LLDB setup from Chapter 1 and compare the event 20/25 insertion against the
worked chain above. Debugger rendering of private types and optimized locals
varies; the deterministic tests remain the executable check.

## What is now available

The engine now has a capped, aligned, rollback-capable allocator and a real
ordered in-memory index. It stores multiple versions, preserves tombstones,
performs lower-bound and visible-sequence lookups, iterates internal order, and
retains stable payload addresses until generation destruction. A prepared token
owns all candidate work; committing it performs only nonthrowing link and scalar
updates under exclusive access.

The tests cover object lifetime and charge restoration, malformed input, borrowed
payload stability, interacting inserts, maximum sequences, deterministic chunk
failures, abandoned preparations, cap exhaustion, forced tower shapes, and an
independently expressed reference order. None of this supplies durable commits,
snapshot ownership across generations, concurrent mutation, or compaction yet.

Chapter 8 will turn this ordered entry stream into an explicitly encoded sorted
table with a sparse index and Bloom filter, and implement the corresponding
checked reader. It will consume the interfaces just built rather than substitute
an opaque storage library. Keep this chapter's restored source and all nine
passing tests as its starting point.
