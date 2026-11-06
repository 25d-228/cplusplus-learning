# Chapter 7 — An arena-backed skip-list memtable

## Part 1 — Stable storage and ordered search

Suppose the in-memory index contains event 10 and event 30. A batch adds event
20, adds event 25, and deletes event 10. Allocating and publishing each operation
in turn is unsafe: a later allocation failure could leave only half the batch
visible. Keeping every allocation alive is not sufficient either. Incorrect
link updates can make a fully allocated node unreachable.

We will implement two connected mechanisms. An arena supplies stable, aligned
storage under a byte cap and can abandon everything allocated during one
preparation. A skip list orders versions of records and publishes an entirely
prepared batch without allocating. The link algorithm, allocation policy,
rollback, and structural checks are ours; no standard ordered container sits
inside the engine.

Begin with the restored [Chapter 6](https://25d-228.github.io/cplusplus-learning/chapter-06.html)
workspace, including its [second part](https://25d-228.github.io/cplusplus-learning/chapter-06-part-2.html)
and all eight passing CTest entries. Keep Chapters 1–5's independent foundation
programs. We reuse `Key`, `KeyLess`, the `Event` and `Account` value types,
30/14-byte codecs, and checked size arithmetic. No preceding file is replaced.

> **Learning objectives**
>
> Allocate aligned storage without relocating existing nodes; separate storage
> from object lifetime; roll back a preparation without losing old values;
> derive skip-list search and insertion; order versions and deletion markers;
> publish interacting nodes without throwing; distinguish stable addresses from
> snapshot semantics; verify every level against a reference order.

The allocation remains **4.5 implementation hours**: arena 1.0, index and batch
preparation 2.5, and structural tests/debugging 1.0. Reading is separate. This
chapter has two complete parts with that **single** allocation. Part 1 develops
storage, record entries, and search. [Part 2](https://25d-228.github.io/cplusplus-learning/chapter-07-part-2.html)
finishes publication, diagnostics, tests, and the build. Implement both before
advancing; the source files continue across the two pages.

## The files and the ownership boundary

Create the directories, retaining the native C++20 compiler already selected:

```sh title="Terminal — extend the engine directories"
mkdir -p include/strata/memory include/strata/storage src/memory src/storage
```

| File | Responsibility |
|---|---|
| `include/strata/memory/arena.hpp` | Capped storage and one rollback transaction |
| `src/memory/arena.cpp` | Chunk acquisition, aligned bump allocation, rollback |
| `include/strata/storage/entry.hpp` | Internal keys, owned mutations, borrowed entries |
| `include/strata/storage/memtable.hpp` | Index, cursor, prepared-batch interfaces |
| `src/storage/memtable.cpp` | Search, preparation, link publication, invariants |
| `tests/memtable.cpp` | Lifetime, failure, version, and reference checks |

All filename-labelled blocks for a file append **in reading order**, first this
page and then Part 2. A later CMake block appends to the existing root file.
There is no new dependency or unfinished future target.

`MemTable` owns its arena. The arena owns chunks, and chunks provide storage for
nodes and their encoded values. A `Cursor` or `EntryView` borrows those objects;
it owns no chunk. A prepared-batch token temporarily controls an arena checkpoint
and the lifetimes of unpublished nodes. It must be destroyed or committed before
the table can be frozen, and must never outlive the table.

The entire layer is **single-threaded or externally exclusive**. Even an
otherwise read-only lookup must not overlap link publication in another thread.
A later snapshot layer will share frozen tables, not turn these ordinary pointer
writes into lock-free operations.

## An arena gives addresses a longer life than a vector

A vector can move its elements when capacity grows. That is unacceptable for a
node referenced by many other nodes. Instead, acquire independent chunks and
never move them. Each chunk holds 4096 payload bytes. A bump cursor advances
inside the newest chunk; a new chunk is acquired when the remaining suffix
cannot hold the next aligned request. We do not search old holes or individually
free committed nodes.

The payload is a genuine bounded array of `std::byte`, not a one-element array
followed by out-of-bounds indexing. `alignas(std::max_align_t)` makes its base
suitable for the fundamental alignments this arena supports. `alignof(T)` asks
for a type's alignment, whereas `sizeof(T)` asks for its complete object size.
An over-aligned type is rejected rather than silently misaligned. See the
[alignment rules](https://eel.is/c++draft/basic.align) and the special role of
[byte arrays that provide object storage](https://eel.is/c++draft/intro.object).

Start the arena header:

```cpp title="include/strata/memory/arena.hpp"
#ifndef STRATA_MEMORY_ARENA_HPP
#define STRATA_MEMORY_ARENA_HPP

#include <cstddef>
#include <optional>
#include <stdexcept>

namespace strata::memory {

class ArenaFull : public std::runtime_error {
public:
    ArenaFull() : std::runtime_error("arena byte cap reached") {}
};

class Arena {
public:
    static constexpr std::size_t chunk_payload = 4096;

private:
    struct Chunk {
        Chunk* previous = nullptr;
        std::size_t used = 0;
        alignas(std::max_align_t) std::byte bytes[chunk_payload];
    };
```

`ArenaFull` means the configured storage cap prevents another chunk. It differs
from `std::bad_alloc`, which can represent allocator failure or our test seam.
Both are exceptions; neither permits partial publication. This distinction
lets later admission code react to a full memtable without pretending the
system allocator failed.

`Chunk::previous` links **ownership history**, not search order. `used` is an
offset into its own byte array. The array is not initialized by `new Chunk`;
its bytes must be written or used to construct an object before reading them.
Pointer and offset members have explicit default initializers.

We charge `sizeof(Chunk)`, not merely 4096. This includes the link, offset,
member padding, payload, and any trailing padding. A partially filled chunk
costs the same as a full one. The allocator's hidden bookkeeping is not included.

### Give rollback one exclusive owner

The nested class `Arena::Transaction` may access private arena members, but it
still needs an actual `Arena&`; nesting does not manufacture an owning instance.
It records the current chunk, that chunk's cursor, and the total used bytes.
Only one transaction may be active. Deleting both copy and move operations
prevents multiple guards from believing they own the same rollback point.
The [nested-class access rules](https://eel.is/c++draft/class.access.nest)
explain the access permission, not a lifetime extension.

Continue the open `Arena` declaration:

```cpp title="include/strata/memory/arena.hpp"
public:
    class Transaction {
    public:
        explicit Transaction(Arena& arena);
        ~Transaction() noexcept;
        Transaction(const Transaction&) = delete;
        Transaction& operator=(const Transaction&) = delete;
        Transaction(Transaction&&) = delete;
        Transaction& operator=(Transaction&&) = delete;
        void* allocate(std::size_t bytes, std::size_t alignment);
        void commit() noexcept;

    private:
        Arena* arena_;
        Chunk* saved_head_;
        std::size_t saved_offset_;
        std::size_t saved_used_;
    };
```

On normal scope exit without `commit`, the transaction discards new chunks and
rewinds the original chunk. On commit, it stops owning that rollback obligation.
Objects constructed in the abandoned storage must be destroyed **before** the
transaction releases or rewinds it. This arena deliberately does not maintain a
hidden destructor registry; its caller owns object lifetimes.

Complete the header:

```cpp title="include/strata/memory/arena.hpp"
    explicit Arena(std::size_t byte_cap) noexcept : cap_(byte_cap) {}
    ~Arena() noexcept;
    Arena(const Arena&) = delete;
    Arena& operator=(const Arena&) = delete;
    Arena(Arena&&) = delete;
    Arena& operator=(Arena&&) = delete;

    static constexpr std::size_t chunk_charge() noexcept { return sizeof(Chunk); }
    std::size_t reserved_bytes() const noexcept { return reserved_; }
    std::size_t used_bytes() const noexcept { return used_; }
    bool transaction_active() const noexcept { return active_; }
    void fail_on_chunk(std::size_t ordinal);
    void clear_failure();

private:
    void* allocate(std::size_t bytes, std::size_t alignment);
    void drop_head() noexcept;
    Chunk* head_ = nullptr;
    std::size_t cap_;
    std::size_t reserved_ = 0;
    std::size_t used_ = 0;
    bool active_ = false;
    std::optional<std::size_t> fail_after_;
};

}  // namespace strata::memory
#endif
```

`reserved_bytes()` is the sum of complete live chunk charges. `used_bytes()`
is the sum of consumed payload prefixes, including alignment padding. The arena
object itself, the memtable's head pointers, stack scratch space, and operating
system memory are outside this local cap. Chapter 9 will compose subsystem
charges; this is not yet a whole-process or whole-engine budget.

`fail_on_chunk(k)` arms a one-based countdown for **new chunk acquisition**, not
every small allocation and not every allocation in the process. A request that
fits the existing chunk does not consume the countdown. Cap rejection precedes
the seam. A triggered failure disarms itself; unused countdown progress is
observational test state, not something rollback promises to restore.

## Rewinding storage without rewinding old objects

Let a checkpoint contain head $H$, its used offset $u$, and total consumed
payload $U$. A transaction can append bytes to $H$ and prepend new chunks. To
abandon it, delete chunks until the head is again $H$, restore that chunk's
cursor to $u$, and restore the aggregate to $U$. Older chunks were never changed.
A null $H$ means the arena was initially empty, so every acquired chunk is new.

Create the implementation:

```cpp title="src/memory/arena.cpp"
#include "strata/memory/arena.hpp"

#include <exception>
#include <new>

namespace strata::memory {

Arena::Transaction::Transaction(Arena& arena)
    : arena_(&arena), saved_head_(arena.head_),
      saved_offset_(arena.head_ ? arena.head_->used : 0),
      saved_used_(arena.used_) {
    if (arena.active_) {
        throw std::logic_error("nested arena transaction");
    }
    arena.active_ = true;
}

Arena::Transaction::~Transaction() noexcept {
    if (!arena_) {
        return;
    }
    while (arena_->head_ != saved_head_) {
        arena_->drop_head();
    }
    if (saved_head_) {
        saved_head_->used = saved_offset_;
    }
    arena_->used_ = saved_used_;
    arena_->active_ = false;
}
```

If a nested transaction is attempted, its constructor throws before setting the
active flag. The first guard remains responsible for its checkpoint. For a
successfully constructed guard, the destructor handles both ordinary abandonment
and stack unwinding. The comparison of head pointers is a comparison of chunk
identity, not an attempt to order unrelated addresses.

The checkpoint only rolls back **allocations**. It cannot undo a caller writing
through a pointer to old data. Our memtable will keep old values immutable and
avoid changing links until preparation is complete. That caller discipline is
what makes arena rollback useful for a strong state guarantee.

```cpp title="src/memory/arena.cpp"
void* Arena::Transaction::allocate(std::size_t bytes, std::size_t alignment) {
    if (!arena_) {
        throw std::logic_error("allocation after transaction commit");
    }
    return arena_->allocate(bytes, alignment);
}

void Arena::Transaction::commit() noexcept {
    if (arena_) {
        arena_->active_ = false;
        arena_ = nullptr;
    }
}

void Arena::drop_head() noexcept {
    Chunk* old = head_;
    head_ = old->previous;
    reserved_ -= sizeof(Chunk);
    delete old;
}

Arena::~Arena() noexcept {
    if (active_) {
        std::terminate();
    }
    while (head_) {
        drop_head();
    }
}
```

`drop_head` releases a complete `Chunk` with ordinary `delete`; it does not
`delete` arena-constructed nodes individually. Its caller knows a head exists.
The decrement is valid because each head was previously charged once. The
transaction restores the aggregate consumed-byte count after dropping all new
chunks; destruction of the whole arena no longer needs that counter.

Destroying an arena with an outstanding transaction is a lifetime contract
violation. `std::terminate()` makes that misuse fail immediately rather than
leaving a guard pointing into a destroyed owner. This is not an ordinary
resource-pressure outcome. A table is likewise nonmovable so an outstanding
token cannot quietly start referring to a different owner address.

Complete the small fault-control operations:

```cpp title="src/memory/arena.cpp"
void Arena::fail_on_chunk(std::size_t ordinal) {
    if (active_) {
        throw std::logic_error("cannot change failure seam during transaction");
    }
    if (ordinal == 0) {
        throw std::invalid_argument("chunk failure ordinal must be positive");
    }
    fail_after_ = ordinal;
}

void Arena::clear_failure() {
    if (active_) {
        throw std::logic_error("cannot change failure seam during transaction");
    }
    fail_after_.reset();
}
```

Changing the seam while a transaction is active is rejected. It would otherwise
make an in-progress operation's test policy depend on unrelated activity. The
same guard makes the seam unusable as a hidden callback during commit.

## Derive alignment before doing pointer arithmetic

Let $C=4096$ be the payload capacity, $u$ the current used offset, $a$ the requested
alignment, and $n$ the requested nonzero byte count. The base is aligned to every
supported $a$. The padding needed to reach the next aligned offset is

$$
p = (a-(u\bmod a))\bmod a.
$$

This also returns zero when $u$ is already aligned. Require $a>0$ before the
modulo. For $u=5$, $a=8$, and $n=16$, padding is 3, the returned offset is 8,
and the new cursor is $5+3+16=24$. Bytes 5 through 7 are padding, not part of the
new object. With $u=4090$, $a=8$, and $n=8$, padding is 6 but the request cannot
fit: no bytes remain after alignment.

Avoid computing an unchecked `u + p + n`. First require

$$
p\leq C-u,\qquad n\leq C-u-p.
$$

Only then advance the cursor. The loop invariant is $0\leq u\leq C$. The first
test protects the second subtraction. A positive power of two has exactly one
set bit, so `a & (a - 1)` is zero; the zero-alignment test must precede that
expression. We support only such alignments no greater than `max_align_t`.

Begin the allocation function:

```cpp title="src/memory/arena.cpp"
void* Arena::allocate(std::size_t bytes, std::size_t alignment) {
    if (alignment == 0 || (alignment & (alignment - 1)) != 0 ||
        alignment > alignof(std::max_align_t)) {
        throw std::invalid_argument("unsupported arena alignment");
    }
    if (bytes == 0 || bytes > chunk_payload) {
        throw std::length_error("arena request must fit one nonempty chunk");
    }
    auto try_chunk = [this, bytes, alignment]() -> void* {
        if (!head_) {
            return nullptr;
        }
        const auto padding = (alignment - head_->used % alignment) % alignment;
        const auto available = chunk_payload - head_->used;
        if (padding > available || bytes > available - padding) {
            return nullptr;
        }
        auto* result = head_->bytes + head_->used + padding;
        head_->used += padding + bytes;
        used_ += padding + bytes;
        return result;
    };
```

`try_chunk` is a local lambda. `[this, bytes, alignment]` borrows this arena
through `this` and copies the two request sizes. The explicit `-> void*` fixes
the return type because one branch returns `nullptr` and another returns a byte
pointer. It is used only during this call. It neither escapes nor allocates.

A `void*` denotes an address without a pointed-to object type. Returning it
separates storage allocation from construction. No `Node` has been constructed
merely because this function advanced the cursor. The successful arithmetic is
bounded by one payload. Aggregate used bytes cannot overflow either: consumed
payload never exceeds the sum of charged live chunks, which fits the cap.

Finish the function and namespace:

```cpp title="src/memory/arena.cpp"
    if (void* result = try_chunk()) {
        return result;
    }
    if (sizeof(Chunk) > cap_ - reserved_) {
        throw ArenaFull{};
    }
    if (fail_after_ && --*fail_after_ == 0) {
        fail_after_.reset();
        throw std::bad_alloc{};
    }
    auto* fresh = new Chunk;
    fresh->previous = head_;
    head_ = fresh;
    reserved_ += sizeof(Chunk);
    return try_chunk();
}

}  // namespace strata::memory
```

Let $B$ be the configured cap, $R$ current reserved bytes, and $K=\operatorname{sizeof}(\text{Chunk})$. The maintained invariant $R\leq B$ allows the admission check
$K\leq B-R$ without overflowing `R + K`. `new Chunk` can throw, but until it
returns, no ownership pointer or charge has changed. After acquisition, linking
the chunk and updating the charge cannot fail.

The second call to `try_chunk` must succeed: the new chunk has offset zero, a
suitably aligned base, and the request was already checked against 4096. The
last chunk's unusable tail remains charged. This is intentional fragmentation,
not lost accounting. Our arena favors predictable lifetime and rollback over
packing every free byte.

The condition `if (void* result = try_chunk())` declares a local pointer and tests
whether it is nonnull. Its scope is that `if` statement. It is not a cast from
integer truth to an address.

## Construct an object; do not merely cast to one

The language requires enough suitably aligned storage and completed
initialization for an object's lifetime to begin. A cast alone does not run a
constructor. Placement construction places an object in storage already owned
elsewhere. A placement `new` expression uses an address rather than acquiring
its own allocation; C++20's `std::construct_at` expresses that operation through
`<memory>` and returns a pointer to the constructed object. The relevant sources
are [object lifetime](https://eel.is/c++draft/basic.life),
[placement new](https://eel.is/c++draft/expr.new), and
[`construct_at`](https://eel.is/c++draft/specialized.construct).

For our node the concrete operation will be:

```cpp title="Construction pattern — explained here, implemented below"
auto* node = std::construct_at(static_cast<Node*>(
    transaction_.allocate(sizeof(Node), alignof(Node))));
```

The allocator returns suitably sized storage. `static_cast<Node*>` gives that
address the type needed by `construct_at`; construction, not the cast, creates
the initialized node. We retain the pointer returned by construction rather
than relying on a stale pointer from an earlier occupant of the same storage.
The empty argument list value-initializes this concrete node, including null
links. It does not initialize a disk format.

`std::destroy_at(node)` invokes the destructor without freeing the containing
chunk. This pairs with placement construction, whereas `delete node` would
attempt to release storage not independently acquired by `new Node`. Destroy
nodes before releasing their chunks. The library specifies this distinction in
[`destroy_at`](https://eel.is/c++draft/specialized.destroy). Part 2 includes a
nontrivial counted object so this lifetime step is observable, rather than
relying only on nodes whose destructor currently does no work.

## Represent versions, not overwrites

A storage key is the pair `(relation, primary)`. Its order is already defined by
Chapter 6's `KeyLess`. An **internal key** adds a positive batch sequence. For
equal storage keys, a larger sequence sorts **earlier**, placing newest versions
first. Different storage keys still sort ascending. No subtraction participates
in the engine comparator, so the full unsigned range is usable.

For one key, versions at sequences 9, 7, and 4 appear as `9, 7, 4`. A lookup
visible through sequence 8 starts at internal key `(key,8)` and finds sequence 7.
A lookup through zero finds none, because committed sequences are positive.
Sequence gaps are valid; wrapping or reusing a sequence in one table is not.

An erase is a stored **tombstone**, not the absence of a node. If sequence 7 is
an erase, a lookup through 8 returns that erase. It must not fall through to the
put at sequence 4. A later merged reader needs the distinction to suppress an
older value in another source.

Create the entry header:

```cpp title="include/strata/storage/entry.hpp"
#ifndef STRATA_STORAGE_ENTRY_HPP
#define STRATA_STORAGE_ENTRY_HPP

#include "strata/core/codec.hpp"
#include "strata/core/record.hpp"

#include <algorithm>
#include <array>
#include <cstddef>
#include <cstdint>
#include <span>
#include <stdexcept>

namespace strata::storage {

enum class EntryKind : std::uint8_t { put = 1, erase = 2 };

struct InternalKey {
    Key user;
    std::uint64_t sequence;
    bool operator==(const InternalKey&) const = default;
};

struct InternalLess {
    bool operator()(const InternalKey& a, const InternalKey& b) const noexcept {
        if (KeyLess{}(a.user, b.user)) { return true; }
        if (KeyLess{}(b.user, a.user)) { return false; }
        return a.sequence > b.sequence;
    }
};

struct EntryView {
    InternalKey key;
    EntryKind kind;
    std::span<const std::byte> value;
};
```

`EntryView` owns a copy of the key and kind, but only a span into the value bytes.
Copying this view does not keep a memtable alive. A put carries exactly its
relation's encoded row; an erase carries no row bytes. Sequence and operation
kind are storage metadata, not silently added to Chapter 6's codec.

### Requests own their input image

We could accept a span into a temporary encoded array, but that would make
lifetime errors easy. A `Mutation` instead owns a fixed 30-byte array, enough for
either schema, plus a logical length. Account images use the first 14 bytes.
Factories are `static` member functions: call `Mutation::put(event)` without an
existing mutation object. Overload selection distinguishes an event from an
account by argument type.

```cpp title="include/strata/storage/entry.hpp"
struct Mutation {
    Key key{};
    EntryKind kind = EntryKind::erase;
    std::array<std::byte, event_bytes> bytes{};
    std::size_t length = 0;

    static Mutation put(const Event& event) {
        return {key_of(event), EntryKind::put, encode_event(event), event_bytes};
    }
    static Mutation put(const Account& account) {
        Mutation result;
        result.key = key_of(account);
        result.kind = EntryKind::put;
        const auto image = encode_account(account);
        std::copy(image.begin(), image.end(), result.bytes.begin());
        result.length = account_bytes;
        return result;
    }
    static Mutation erase(Key key) noexcept {
        Mutation result;
        result.key = key;
        return result;
    }
};
```

These factories make the usual path correct, but their public fields are not a
trust boundary. A caller can still fabricate an inconsistent key or length.
Validate the entire request before allocating nodes. A repeated storage key
within a batch is rejected later, even when one operation is a put and another
an erase. That policy avoids an implicit "first wins" or "last wins" rule.

```cpp title="include/strata/storage/entry.hpp"
inline void validate_mutation(const Mutation& mutation) {
    if (mutation.key.relation != Relation::events &&
        mutation.key.relation != Relation::accounts) {
        throw std::invalid_argument("unknown mutation relation");
    }
    if (mutation.kind == EntryKind::erase) {
        if (mutation.length != 0) {
            throw std::invalid_argument("tombstone has a value");
        }
        return;
    }
    if (mutation.kind != EntryKind::put || mutation.length > mutation.bytes.size()) {
        throw std::invalid_argument("invalid mutation kind or size");
    }
    const auto image = std::span<const std::byte>{mutation.bytes}.first(mutation.length);
    const auto decoded_key = mutation.key.relation == Relation::events
        ? key_of(decode_event(image)) : key_of(decode_account(image));
    if (decoded_key != mutation.key) {
        throw std::invalid_argument("mutation key disagrees with encoded row");
    }
}

}  // namespace strata::storage
#endif
```

The length guard precedes `.first(length)`. An erase with nonzero length is
rejected; a put must decode with the expected relation, version, and exact byte
count. The decoded row's key must equal the metadata key. Otherwise a point
lookup could find a value that claims to belong to a different row.

`FormatError` remains the codec's malformed-image error. Invalid operation
metadata uses `invalid_argument`. Either error happens before the index changes.
The unused suffix of an account's owning array is not copied into the memtable.

## A skip list is several ordered lists sharing nodes

Level zero contains every version. A node with height $h$ participates in levels
$0$ through $h-1$. At each level it points to the next node participating there;
its height is not the number of following rows to skip. Higher levels bypass
some lower-level work without creating copies of the records.

This is the probabilistic search-and-splice structure introduced by
[William Pugh, *Skip Lists: A Probabilistic Alternative to Balanced Trees*,
first two pages](https://15721.courses.cs.cmu.edu/spring2017/papers/07-oltpindexes1/pugh-skiplists-cacm1990.pdf).
Our versioned ordering, arena rollback, and batch publication policy are
additional design decisions. We do not implement physical deletion of individual
nodes; an erase adds another version, and a whole generation is reclaimed later.

Consider this illustrative shape, ignoring sequences because all keys differ:

```text title="A four-node search shape"
height:       1       3       1       2
level 2: head ------>20-----------------> end
level 1: head ------>20------------->40-> end
level 0: head ->10-->20-->30--------->40-> end
```

To seek 25, begin at the highest level. Advance to 20, but not beyond the end.
At level 1, 40 would overshoot, so stay at 20 and descend. At level 0, 30 would
also overshoot. The next node at level zero is therefore the lower bound, 30.
Seeking exactly 20 must stop **before** 20, so the horizontal comparison is
strictly less than, not less than or equal to.

### Bounded towers without pretending a one-element array is larger

We choose a maximum height of 16 and give every node a real `std::array` of 16
link pointers. Only the first `height` slots participate; the remaining slots
stay null. This spends extra memory for a simple, portable object layout. It is
not the variable-length trailing-array technique seen in some implementations.

The node also has a separate pending-chain pointer. Before publication that
chain owns the inventory of constructed nodes. It must not reuse `next[0]`,
whose meaning changes during publication. Node payloads live in separately
allocated regions of the same arena, so tombstones need no payload allocation.

Start the memtable header:

```cpp title="include/strata/storage/memtable.hpp"
#ifndef STRATA_STORAGE_MEMTABLE_HPP
#define STRATA_STORAGE_MEMTABLE_HPP

#include "strata/memory/arena.hpp"
#include "strata/storage/entry.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <limits>
#include <optional>
#include <span>

namespace strata::storage {

class MemTable {
public:
    static constexpr std::size_t max_height = 16;
    static constexpr std::size_t max_batch = 256;

private:
    struct Node {
        InternalKey key{};
        EntryKind kind = EntryKind::erase;
        std::span<const std::byte> value{};
        std::size_t height = 1;
        std::array<Node*, max_height> next{};
        Node* pending_next = nullptr;
    };
```

The fields in an arena node are ordinary C++ objects; `InternalKey`, spans, and
link arrays may contain implementation padding. None is written directly to
storage. The static payload limit and complete node size are checked in the
constructor implementation below.

We keep one head pointer for each level instead of inventing a sentinel key
smaller than every possible unsigned key. A null predecessor means "use the
head pointer at this level," not an actual node with an invalid relation tag.

### A cursor borrows; a preparation owns a pending change

The two nested interfaces have different responsibilities. A cursor holds one
borrowed node pointer and can advance at level zero. A prepared token holds the
rollback guard and unpublished-node chain. Their constructors are private so
only `MemTable` can produce a valid one. The explicit `friend class MemTable`
permits the enclosing class to call those private constructors; outer classes
do not automatically gain access to nested classes' private members. See
[friend declarations](https://eel.is/c++draft/class.friend).

```cpp title="include/strata/storage/memtable.hpp"
public:
    class Cursor {
    public:
        bool valid() const noexcept { return node_ != nullptr; }
        EntryView entry() const;
        void next() noexcept;
    private:
        friend class MemTable;
        explicit Cursor(const Node* node) noexcept : node_(node) {}
        const Node* node_;
    };

    class Prepared {
    public:
        ~Prepared() noexcept;
        Prepared(const Prepared&) = delete;
        Prepared& operator=(const Prepared&) = delete;
        Prepared(Prepared&&) = delete;
        Prepared& operator=(Prepared&&) = delete;
        void commit() noexcept;
    private:
        friend class MemTable;
        Prepared(MemTable& table, std::uint64_t sequence,
                 std::span<const Mutation> changes);
        void destroy_pending() noexcept;
        MemTable& table_;
        memory::Arena::Transaction transaction_;
        Node* first_ = nullptr;
        Node* last_ = nullptr;
        std::size_t new_size_ = 0;
        std::uint64_t sequence_;
        std::uint32_t next_seed_;
    };
```

`table_` is a reference, so the token does not own or extend its table's lifetime.
`transaction_` is declared before the pending pointers. The token's destructor
body will destroy unpublished nodes first; automatic member destruction then
runs the transaction's rollback. The copied generator state belongs to the
candidate until commit.

The token is deliberately neither copyable nor movable. That is compatible
with returning a freshly constructed token by value, as Part 2 explains. It
also avoids a move-assignment case that might unexpectedly abandon an unrelated
prepared batch.

Continue with the public operations:

```cpp title="include/strata/storage/memtable.hpp"
    explicit MemTable(std::size_t byte_cap, std::uint32_t seed = 1,
                      std::size_t fixed_height = 0);
    ~MemTable() noexcept;
    MemTable(const MemTable&) = delete;
    MemTable& operator=(const MemTable&) = delete;
    MemTable(MemTable&&) = delete;
    MemTable& operator=(MemTable&&) = delete;

    Prepared prepare(std::uint64_t sequence, std::span<const Mutation> changes) &;
    Cursor begin() const & noexcept;
    Cursor begin() const && = delete;
    Cursor lower_bound(InternalKey key) const & noexcept;
    Cursor lower_bound(InternalKey key) const && = delete;
    std::optional<EntryView> find(Key key, std::uint64_t visible =
        std::numeric_limits<std::uint64_t>::max()) const &;
    std::optional<EntryView> find(Key, std::uint64_t =
        std::numeric_limits<std::uint64_t>::max()) const && = delete;
    void freeze();
    bool frozen() const noexcept { return frozen_; }
    std::size_t size() const noexcept { return size_; }
    std::uint64_t last_sequence() const noexcept { return last_sequence_; }
    std::size_t reserved_bytes() const noexcept { return arena_.reserved_bytes(); }
    std::size_t used_bytes() const noexcept { return arena_.used_bytes(); }
    static constexpr std::size_t node_bytes() noexcept { return sizeof(Node); }
    void fail_on_chunk(std::size_t ordinal) { arena_.fail_on_chunk(ordinal); }
    void clear_failure() { arena_.clear_failure(); }
    void check_structure() const;
    std::array<std::size_t, max_height> height_counts() const noexcept;
```

A default `visible` argument means the largest representable sequence. Explicit
values allow boundary tests and future merged reads. The ref-qualified `const &`
accessors borrow from an lvalue table; their `const &&` counterparts are deleted
to reject obvious borrowing from a temporary. This is a useful restriction,
not a lifetime proof for every possible caller.

`size()` counts **versions**, including tombstones, not distinct live keys.
`freeze()` only seals this table against later preparation. It does not flush,
create a durable snapshot, synchronize threads, or transfer ownership.

Finish the private state:

```cpp title="include/strata/storage/memtable.hpp"
private:
    Node*& link(Node* previous, std::size_t level) noexcept;
    const Node* link(const Node* previous, std::size_t level) const noexcept;
    void publish(Node* node) noexcept;
    std::size_t choose_height(std::uint32_t& state) const noexcept;
    memory::Arena arena_;
    std::array<Node*, max_height> heads_{};
    std::size_t size_ = 0;
    std::uint64_t last_sequence_ = 0;
    std::uint32_t seed_;
    std::size_t fixed_height_;
    bool frozen_ = false;
};

}  // namespace strata::storage
#endif
```

Declare the arena before the index state so it is destroyed last. The table's
destructor body will traverse level zero while all node storage is still alive.
No head array or cursor owns a node separately. `fixed_height` is a deterministic
test configuration: zero chooses generated heights; 1 through 16 force one
height for every node. It is not an adaptive performance setting.

## Begin with construction and whole-generation destruction

The type traits with suffix `_v` are compile-time Boolean queries from
`<type_traits>`. `is_nothrow_default_constructible_v<Node>` checks that empty node
construction cannot throw, and `is_trivially_destructible_v<Node>` checks that
node destruction is trivial. See the [type-property specifications](https://eel.is/c++draft/meta.unary.prop).
These assertions guard this concrete representation; they do not say that
arbitrary placement constructors cannot fail.

Create the source file:

```cpp title="src/storage/memtable.cpp"
#include "strata/storage/memtable.hpp"
#include "strata/core/checked.hpp"

#include <algorithm>
#include <exception>
#include <memory>
#include <stdexcept>
#include <type_traits>

namespace strata::storage {

MemTable::MemTable(std::size_t byte_cap, std::uint32_t seed,
                   std::size_t fixed_height)
    : arena_(byte_cap), seed_(seed), fixed_height_(fixed_height) {
    static_assert(alignof(Node) <= alignof(std::max_align_t));
    static_assert(sizeof(Node) <= memory::Arena::chunk_payload);
    static_assert(std::is_nothrow_default_constructible_v<Node>);
    static_assert(std::is_trivially_destructible_v<Node>);
    if (seed == 0 || fixed_height > max_height) {
        throw std::invalid_argument("invalid height generator configuration");
    }
}

MemTable::~MemTable() noexcept {
    if (arena_.transaction_active()) {
        std::terminate();
    }
    Node* node = heads_[0];
    while (node) {
        Node* next = node->next[0];
        std::destroy_at(node);
        node = next;
    }
}
```

Nonthrowing node construction makes the interval between obtaining node storage
and registering its lifetime in the pending chain safe. Later payload allocation
can still fail and must be covered by that chain's cleanup.

Destruction saves `next[0]` **before** ending the current node's lifetime. Reading
that member afterward would use a dead object. The arena then releases its own
chunks automatically after the table destructor body finishes. Values are byte
regions owned by those chunks; there is no separate heap payload to delete.

The zero generator seed is rejected because the chosen shift/XOR recurrence
would remain zero forever. The fixed-height upper bound is validated once rather
than checked at every link access.

## Search by preserving the predecessor invariant

The mutable helper returns `Node*&`: a **reference to a pointer slot**, not just
a copied pointer. Assigning through it updates either a node's actual outgoing
link or the actual head slot. The const helper returns a borrowed pointer value
and cannot expose a mutable link. These overloads share the sentinel-free head
policy.

```cpp title="src/storage/memtable.cpp"
MemTable::Node*& MemTable::link(Node* previous, std::size_t level) noexcept {
    return previous ? previous->next[level] : heads_[level];
}

const MemTable::Node* MemTable::link(const Node* previous,
                                   std::size_t level) const noexcept {
    return previous ? previous->next[level] : heads_[level];
}

MemTable::Cursor MemTable::begin() const & noexcept {
    return Cursor{heads_[0]};
}

MemTable::Cursor MemTable::lower_bound(InternalKey key) const & noexcept {
    const Node* previous = nullptr;
    for (std::size_t level = max_height; level != 0;) {
        --level;
        const Node* next = link(previous, level);
        while (next && InternalLess{}(next->key, key)) {
            previous = next;
            next = link(previous, level);
        }
    }
    return Cursor{link(previous, 0)};
}
```

At the start of each horizontal step, `previous` is null or has an internal key
strictly before the target. Moving to `next` preserves that invariant. When the
next key is not before the target, descending one level keeps the same valid
predecessor and examines finer links. A node reached at a higher level also
participates at all lower levels, so its lower links exist.

The outer loop starts at 16 and decrements **inside** the loop before use. It
visits levels 15 through 0 without testing an unsigned variable against zero
with `>=`, which would never become false after wraparound. Once level zero is
finished, its next link is the first key not before the target, or null at end.

`begin` returns the first level-zero node. It is raw internal iteration, not a
query that collapses old versions. Both cursor operations and lookup are small
enough to keep the borrowing behavior explicit:

```cpp title="src/storage/memtable.cpp"
EntryView MemTable::Cursor::entry() const {
    if (!node_) {
        throw std::out_of_range("entry at end of memtable");
    }
    return {node_->key, node_->kind, node_->value};
}

void MemTable::Cursor::next() noexcept {
    if (node_) {
        node_ = node_->next[0];
    }
}

std::optional<EntryView> MemTable::find(Key key, std::uint64_t visible) const & {
    auto cursor = lower_bound({key, visible});
    if (!cursor.valid()) {
        return std::nullopt;
    }
    auto entry = cursor.entry();
    if (entry.key.user != key) {
        return std::nullopt;
    }
    return entry;
}
```

`entry()` returns a new view object, not a node reference. At end it throws
`out_of_range`; `next()` at end is an idempotent no-op. The lookup checks the
storage key after lower bound because the next node can belong to another key.
An equal key with an erase kind is returned as a **present tombstone**. A missing
optional and a tombstone are intentionally different results.

A previously returned value span remains valid across later insertions because
old arena storage does not move or get overwritten. A cursor resumed after
insertion can nevertheless see changed forward links. Therefore iteration must
not span a commit when a stable result is required. Freeze first, and keep the
whole table alive for the entire read. Stable addresses are not by themselves
snapshot semantics.

## Heights: a useful model, not a worst-case promise

With ideal independent promotion probability $1/2$, a node reaches height at
least $h$ with probability $2^{-(h-1)}$. For maximum height $H$, its expected
participating height is

$$
\mathbb{E}[h]=\sum_{j=1}^{H}2^{-(j-1)}=2(1-2^{-H}).
$$

For $H=16$, that is approximately 1.99997 participating levels. With $n$ nodes,
level $i$ has expected membership $n/2^i$. Sixteen nodes would have expected
memberships 16, 8, 4, 2, 1 in the first five levels; an actual fixture need not
match those numbers.

These calculations describe an idealized distribution, not memory allocation:
our node **always stores 16 pointers**, even when its height is one. Nor is a
fixed height cap an asymptotic logarithmic guarantee for arbitrarily large $n$.
The top level still has roughly $n/2^{15}$ nodes in this model, and an all-height-one
list has linear search. Chunk caps bound practical table sizes; generated heights
provide a useful expected search shape, not an adversarial performance guarantee.

Use a reproducible 32-bit shift/XOR generator. A generated zero bit promotes;
stop at the first one bit or at the height cap:

```cpp title="src/storage/memtable.cpp"
std::size_t MemTable::choose_height(std::uint32_t& state) const noexcept {
    if (fixed_height_ != 0) {
        return fixed_height_;
    }
    state ^= state << 13;
    state ^= state >> 17;
    state ^= state << 5;
    auto bits = state;
    std::size_t height = 1;
    while (height < max_height && (bits & 1U) == 0) {
        ++height;
        bits >>= 1;
    }
    return height;
}
```

The three shifts update a local 32-bit unsigned state. Every shift count is
smaller than 32. Copying that state into a preparation makes a failed attempt
leave the table's future height sequence unchanged. This simple generator is
not cryptographic, and this implementation does not claim its bits satisfy the
independence assumptions of the probability model. The test seed makes failures
repeatable; forcing heights 1 and 16 exercises both structural extremes without
waiting for a rare generated tower.

The source namespace intentionally remains open here. Continue in
[Part 2 — Prepare, publish, and verify](https://25d-228.github.io/cplusplus-learning/chapter-07-part-2.html)
to finish the source, add the complete tests, and run the nine-test cumulative
suite. Nothing in Part 1 is a durable database yet.

## References for this part

The links above identify the particular language and algorithm facts used here.
The public working draft evolves beyond C++20; only the facilities explicitly
used by these listings belong to our C++20 baseline. No newer draft feature is
required. Pugh's paper supplies the skip-list background, not our memory,
versioning, or batch-failure policy. Its figures are not copied into the book.
