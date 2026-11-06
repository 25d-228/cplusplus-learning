# Chapter 12 · Part 2 — Merge versions without borrowing dead bytes

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-12.html) gave each view
an immutable source set and a live context. This part turns those owners into
ordered results. Append all `src/storage/read_view.cpp` blocks below to that
part's file; its internal `read_detail` namespace is still open.

## One user key, several internal versions

The storage order from Chapter 7 is ascending user key, then descending sequence.
For user keys $k_a,k_b$ and sequences $s_a,s_b$, internal key A precedes B when:

$$
k_a < k_b \quad\text{or}\quad (k_a = k_b \text{ and } s_a > s_b).
$$

A user key includes both relation and primary key. Events precede accounts because
of their encoded relation ordering. A newer version of the same key comes first;
a numerically smaller sequence is not a better answer.

For a view with visible sequence $v$, the point answer comes from the greatest
stored sequence $s \leq v$ for that user key. If its kind is `erase`, the answer
is absent. Absence in one source is not global absence; a tombstone winning across
all sources is. Although proper capture gives us only generations at or below
$v$, the reads retain the explicit cutoff instead of relying on an undocumented
accident of fixture construction.

Consider two sources after the roadmap batch:

| Source A, older | Source B, newer |
|---|---|
| Event 101, sequence 1, put 250 | Event 101, sequence 2, put 900 |
| Event 102, sequence 1, put 700 | Event 102, sequence 2, erase |
| Event 103, sequence 1, put 50 | Account 10, sequence 2, put region 3 |

For event 101 the merge must choose B's 900 and consume A's 250 without emitting
it. For event 102 it must consume **both** the tombstone and the older 700, then
emit nothing. Merely skipping the tombstone record would resurrect A's value.

A flush can place A on disk without changing these rules. Its cursor might load
a new block while advancing past event 101; any span into the discarded frame
then expires. We therefore copy the winner before advancing its source.

## Store a position without making an immovable cursor movable

`MemTable::Cursor` is a small node pointer. `TableReader::Cursor` is intentionally
noncopyable and nonmovable; it retains a reference to its reader and a frame pin.
An `optional<TableReader::Cursor>` cannot simply `emplace(table.lower_bound(...))`:
that expression tries to construct a cursor from another cursor.

Instead give the optional a small containing type whose constructor directly
initializes its cursor member from the factory result:

```cpp title="src/storage/read_view.cpp"
struct Position {
    struct Disk {
        TableReader::Cursor cursor;
        Disk(TableReader& table, InternalKey target) : cursor(table.lower_bound(target)) {}
    };
    std::optional<MemTable::Cursor> memory;
    std::optional<Disk> disk;
    void start(const Source& source, InternalKey target) {
        if (source.memory) { memory.emplace(source.memory->table().lower_bound(target)); }
        else { disk.emplace(*source.disk, target); }
    }
    bool valid() const noexcept {
        return memory ? memory->valid() : disk && disk->cursor.valid();
    }
    EntryView entry() const {
        return memory ? memory->entry() : disk->cursor.entry();
    }
    void next() {
        if (memory) { memory->next(); }
        else { disk->cursor.next(); }
    }
    void reset() noexcept { disk.reset(); memory.reset(); }
};
```

`Disk(table, target)` initializes `cursor` from a same-type prvalue. The result is
constructed directly in that member; it does not require a move constructor.
This is the [same-type prvalue initialization rule](https://eel.is/c++draft/dcl.init.general),
not a request that the optimizer happen to remove a move. `disk.emplace(table,
target)` constructs `Disk` itself in optional storage, as specified by
[`optional::emplace`](https://eel.is/c++draft/optional.assign).

For a memory source, `memory.emplace(...)` copies the harmless node position.
For a disk source, `disk.emplace(...)` may allocate a charged frame and perform
I/O. If it throws, the optional remains empty. During an unfinished scan's cleanup,
already constructed positions release their pins automatically.

`Position` is an internal tagged choice implemented with two optionals. Exactly
one is engaged for an active source; `reset` makes both empty. Its `entry` and
`next` precondition is a valid initialized position. Only the heap loop calls
them, after inserting valid positions. The public cursor checks its own validity
before exposing a row.

The extra wrapper does not allocate an extra cursor object. `Disk`, the optional,
and every `Position` are inline members of one charged `Scan` allocation.

## A heap of source indices, not a second copy of the data

Append the complete scan declaration:

```cpp title="src/storage/read_view.cpp"
struct Scan {
    Scan(std::shared_ptr<const State> state, Key lower, std::optional<Key> upper);
    bool before(std::size_t a, std::size_t b) const;
    void push(std::size_t source);
    std::size_t pop();
    bool in_range(std::size_t source) const;
    void advance();
    void stop() noexcept;
    std::shared_ptr<const State> view;
    std::array<Position, max_sources> positions{};
    std::array<std::size_t, max_sources> heap{};
    std::size_t heap_size = 0;
    std::optional<Key> upper;
    std::optional<OwnedEntry> current;
};
```

The heap contains indices into `positions`, not entries, paths, or owning frame
copies. At most one index is present for each valid source. `heap_size` is the
active prefix of the fixed index array. Empty positions can retain no pin once
reset. The output `current` owns at most 30 payload bytes in its inline array.

`view` is declared before `positions`. Destruction therefore releases all pins
and reader borrows before releasing the source owners and context. The scan's
allocator also owns the budget through final deallocation. Moving `ReadCursor`
only moves its pointer to `Scan`; it does not relocate any position inside the
scan allocation.

For a zero-based binary heap, a child's parent and a parent's children are:

$$
p(i)=\left\lfloor\frac{i-1}{2}\right\rfloor\quad(i>0),\qquad
\ell(i)=2i+1,\qquad r(i)=2i+2.
$$

Only indices below `heap_size` exist. The current cap is 20, so these calculations
are far from `size_t` overflow. This bound is a reason for the arithmetic, not an
excuse to omit overflow reasoning in an unbounded heap.

The comparator and insertion routine are:

```cpp title="src/storage/read_view.cpp"
bool Scan::before(std::size_t a, std::size_t b) const {
    const auto x = positions[a].entry().key;
    const auto y = positions[b].entry().key;
    if (InternalLess{}(x, y)) { return true; }
    if (InternalLess{}(y, x)) { return false; }
    return a < b;
}
void Scan::push(std::size_t source) {
    if (heap_size == heap.size()) { throw std::logic_error("merge heap overflow"); }
    auto child = heap_size++;
    while (child != 0) {
        const auto parent = (child - 1) / 2;
        if (!before(source, heap[parent])) { break; }
        heap[child] = heap[parent];
        child = parent;
    }
    heap[child] = source;
}
```

`before(a,b)` compares the current internal keys of sources a and b. If they are
exactly equal, source index breaks the tie deterministically. Valid installed
generation intervals do not duplicate one internal key across sources; the tie
rule still gives the heap a complete ordering and agrees with point lookup's
first-source preference. It is not a repair policy for conflicting physical data.

Insertion keeps the new index in the local `source` variable. A larger parent is
moved down into the empty child slot until the correct position is reached. The
entry bytes and its underlying source cursor never move.

For three heads `A=(20,1)`, `B=(10,2)`, and `C=(10,1)`, omitting the common relation,
insert A: `[A]`. Insert B: it precedes A, giving `[B,A]`. Insert C: it follows B,
giving `[B,A,C]`. The root is the least internal key, which means the **newest**
version of the least user key.

Remove the minimum by moving the last index down the tree:

```cpp title="src/storage/read_view.cpp"
std::size_t Scan::pop() {
    const auto result = heap[0];
    const auto last = heap[--heap_size];
    if (heap_size == 0) { return result; }
    std::size_t parent = 0;
    while (2 * parent + 1 < heap_size) {
        auto child = 2 * parent + 1;
        if (child + 1 < heap_size && before(heap[child + 1], heap[child])) { ++child; }
        if (!before(heap[child], last)) { break; }
        heap[parent] = heap[child];
        parent = child;
    }
    heap[parent] = last;
    return result;
}
```

The caller guarantees a nonempty heap. Decrementing its size first leaves `last`
in a local variable even when it was the only element. With `[B,A,C]`, removing B
leaves C as the candidate root. A is its only child and follows C, so the result
is `[C,A]`. The next record is the older version of key 10, then key 20.

With two children, choose the earlier child before comparing it with `last`.
Choosing the left child unconditionally can violate the heap invariant even if
every source is individually sorted. `push` and `pop` touch only $O(\log K)$
indices for $K$ live source heads. They neither sort all records nor call an
opaque database merge routine.

## Half-open ranges without an overflowing sentinel

A range is lower-inclusive and upper-exclusive: $[a,b)$. Omitting the upper key
means through the end of the last relation. Equal endpoints are empty; reversed
endpoints are rejected. We never compute `largest_primary_key + 1`, so the maximum
64-bit account key remains readable.

Append bound checking and initialization:

```cpp title="src/storage/read_view.cpp"
bool Scan::in_range(std::size_t source) const {
    return positions[source].valid() &&
        (!upper || KeyLess{}(positions[source].entry().key.user, *upper));
}
Scan::Scan(std::shared_ptr<const State> state, Key lower, std::optional<Key> limit)
    : view(std::move(state)), upper(limit) {
    valid_key(lower);
    if (upper) {
        valid_key(*upper);
        if (KeyLess{}(*upper, lower)) { throw std::invalid_argument("reversed read range"); }
        if (*upper == lower) { return; }
    }
    const InternalKey target{lower, std::numeric_limits<std::uint64_t>::max()};
    for (std::size_t i = 0; i < view->count; ++i) {
        positions[i].start(view->held[i], target);
        if (in_range(i)) { push(i); }
        else { positions[i].reset(); }
    }
    advance();
}
```

`target` uses the largest sequence for the lower user key. Since sequences sort
descending, this seeks before every stored version of that key. The subsequent
winner check applies the visible sequence. Every source is positioned independently;
table positions use Chapter 8's last-internal-key sparse search, including when
one user key spans blocks.

An equal-endpoint range returns before initializing any source cursor. Other
ranges can read a boundary block to discover their first row. Advancing to learn
that the next key is out of range can also load one look-ahead block. This is not
an API promising zero I/O outside the returned user-key interval.

Only a valid position whose key precedes the upper endpoint enters the heap.
An excluded/exhausted position is reset immediately, releasing its pin. If a
later source's initialization throws, construction of the whole `Scan` fails and
C++ destroys all previously completed members. No partial public cursor escapes.

A new cursor is already positioned on its first output row when `scan` returns.
Initialization therefore calls `advance`, which may consume several obsolete
versions or tombstones before finding a row.

## Drain the entire key before exposing its answer

Append terminal cleanup and advancement:

```cpp title="src/storage/read_view.cpp"
void Scan::stop() noexcept {
    current.reset();
    heap_size = 0;
    for (auto& position : positions) { position.reset(); }
}
void Scan::advance() {
    current.reset();
    try {
        while (heap_size != 0) {
            const auto key = positions[heap[0]].entry().key.user;
            std::optional<OwnedEntry> winner;
            do {
                const auto source = pop();
                const auto entry = positions[source].entry();
                if (!winner && entry.key.sequence <= view->sequence) { winner = own(entry); }
                positions[source].next();
                if (in_range(source)) { push(source); }
                else { positions[source].reset(); }
            } while (heap_size != 0 && positions[heap[0]].entry().key.user == key);
            if (winner && winner->kind == EntryKind::put) {
                current = std::move(winner);
                return;
            }
        }
    } catch (...) {
        stop();
        throw;
    }
}
}  // namespace read_detail
```

Follow the inner `do` loop for event 101. B is popped first. Its 900-cent payload
is copied into `winner`; B then advances. If B crosses a block boundary, the
copy survives the old pin's release. Reinserting B updates the heap against
all other sources. A's sequence-1 version is then popped and consumed, but does
not replace the already chosen winner. Only when the heap's least key differs
does `current` receive the owning winner.

For event 102 the winner is an erase. The same loop consumes A's older put. The
outer loop then continues without exposing any event 102. The tombstone is not
returned as an empty row, and the older value is not revisited on the next call.

If the highest candidate has sequence greater than the visible boundary, it is
consumed without becoming a winner. The first later candidate at or below the
boundary wins. This follows from descending sequence order; no per-key vector or
sort is required.

The call to `next()` happens **after copying** the candidate. This is the key
ownership step. Saving only `EntryView winner` would save a span that can point
into a released block. Even when a cache happens to keep that block resident,
such a borrow would violate the API and fail under a charged bypass.

### Failure is terminal, not a resumable hole

A source can fail while the group is partly consumed: allocation may be denied,
I/O may fail, or a newly loaded block may be corrupt. Rewinding all cursors is not
implemented. Returning the copied candidate anyway would conceal that validation
failed before the group completed.

The catch block clears the output and heap, resets every position, and rethrows.
The public cursor becomes terminal. Further `next` calls are no-ops, and `entry`
throws. The immutable view remains valid; create a new cursor with an explicit
range to retry. Previously delivered rows are not rolled back. This is a
fail-stop streaming contract, not the strong guarantee for a whole externally
consumed query result.

Even a successful emission can involve reading ahead: the algorithm must drain
all old versions of the current key. A subsequent source failure therefore may
prevent returning a row whose candidate bytes were already copied. This is
intentional conservative behavior, and the fixture tests it at a frame boundary.

### Work and memory bounds

Let $N$ be the number of internal entries consumed, including hidden versions
and tombstones; $K$ the number of source positions; and $D$ the disk-source count.
Heap construction here uses repeated insertions, so total comparison work is:

$$
O\bigl(K\log(\max(2,K)) + N\log(\max(2,K))\bigr).
$$

For eight sources, a root-to-leaf path has at most three edges. Each consumed
entry performs one removal and at most one insertion; a group with five old
versions still incurs work for all five even though it emits one row. Complexity
must be expressed in internal entries, not only output rows.

The scan's arrays have fixed capacity 20 and are charged as part of its actual
control/object allocation. Each disk position pins at most one frame, sharing
it on a cache hit. An old pin is released before the table cursor requests its
next frame; the cache loader can still require temporary admission headroom as
in Chapter 9. A useful ownership upper bound is:

$$
M_{\text{scan}} \leq M_{\text{control+arrays}} +
\sum_{j=1}^{D} M_{\text{pinned frame},j} + M_{\text{load candidate}}.
$$

This expression is conservative because shared cached frames need not be counted
twice physically. It does not include the separately charged retained arenas,
reader metadata, or unrelated resident cache entries. All share the same budget,
which can reject a cursor even when its source count fits. No full relation is
collected in the merge implementation.

## Point reads choose across sources, not across call times

The internal namespace is now closed by the preceding block. Append these public
view definitions at `strata::storage` namespace scope:

```cpp title="src/storage/read_view.cpp"
void ReadView::require_valid() const {
    if (!state_) { throw std::logic_error("empty read view"); }
}
std::uint64_t ReadView::sequence() const { require_valid(); return state_->sequence; }
std::size_t ReadView::source_count() const { require_valid(); return state_->count; }
ReadRetention ReadView::retention() const { require_valid(); return state_->context->retention(); }
std::optional<OwnedEntry> ReadView::get(Key key) const {
    require_valid();
    valid_key(key);
    std::optional<OwnedEntry> winner;
    for (const auto& source : state_->sources()) {
        std::optional<OwnedEntry> candidate;
        if (source.memory) {
            if (const auto entry = source.memory->table().find(key, state_->sequence)) {
                candidate = own(*entry);
            }
        } else { candidate = source.disk->find(key, state_->sequence); }
        if (candidate && (!winner || candidate->key.sequence > winner->key.sequence)) {
            winner = std::move(candidate);
        }
    }
    if (winner && winner->kind == EntryKind::erase) { return std::nullopt; }
    return winner;
}
```

For each source, `find` supplies that source's latest version at or below the
view boundary. Comparing those candidates yields the globally latest version.
A tombstone is suppressed only **after** that global choice. Returning absence as
soon as any old source reports a tombstone would incorrectly hide a newer put.

The method examines at most the captured source count. It does not rely on the
array being sorted newest-first, making it useful after the catalog later learns
to install compacted file groups. It still depends on each source's ordering and
validation invariants. A failure in a required source propagates rather than
quietly returning a possibly incomplete answer.

Every candidate and the returned winner owns its payload. A point result can
therefore survive view reset. In contrast, a cursor entry is a short borrow into
that cursor's current owning row.

Finish the implementation file with the factory and cursor interface:

```cpp title="src/storage/read_view.cpp"
ReadCursor ReadView::scan(Key lower, std::optional<Key> upper) const {
    require_valid();
    return ReadCursor{std::allocate_shared<read_detail::Scan>(
        memory::SharedAllocator<read_detail::Scan>{state_->context->budget}, state_, lower, upper)};
}
bool ReadCursor::valid() const noexcept { return scan_ && scan_->current.has_value(); }
EntryView ReadCursor::entry() const & {
    if (!valid()) { throw std::logic_error("read cursor has no row"); }
    return scan_->current->view();
}
void ReadCursor::next() & {
    if (valid()) { scan_->advance(); }
}

}  // namespace strata::storage
```

The factory allocates one `Scan` through an owning allocator and passes it a
shared state owner. Allocation or initialization failure destroys that unfinished
scan and leaves the view unchanged. A successful cursor owns the state even when
created from `db.snapshot().scan(...)`.

The publicly move-only cursor need not expose its implementation layout. Its
shared-pointer member can move without relocating the nonmovable disk positions.
A default or moved-from cursor is simply invalid; `reset()` releases its scan,
which can release the last snapshot admission slot.

Before running the tests, predict these three cases. A range ends at account key
10: account 10 is excluded. A view contains a newer erase and an older put: neither
appears. An exhausted cursor is kept in a variable after its view is reset: the
state remains retained until that cursor is reset or destroyed. Part 3 checks
each boundary with the actual owners and files, rather than substituting an
in-memory map for the engine.

## References used for the language boundaries

The [initialization rules](https://eel.is/c++draft/dcl.init.general) explain the
nonmovable cursor member; [optional emplacement](https://eel.is/c++draft/optional.assign)
explains its in-place construction and exception state; and
[member destruction](https://eel.is/c++draft/class.dtor) determines why positions
must follow their view owner in declaration order. The linked working draft also
contains newer features; this implementation uses C++20 facilities, not every
facility shown on those pages. The heap and visibility derivations above are the
algorithms implemented here, not borrowed library behavior.

Continue with [Part 3](https://25d-228.github.io/cplusplus-learning/chapter-12-part-3.html)
for the complete fixture, prerequisite test migration, and build commands.
