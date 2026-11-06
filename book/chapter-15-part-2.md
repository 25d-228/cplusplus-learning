# Chapter 15 — External sorting with bounded fan-in

## Part 2 — Merge only what fits

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-15.html) defined the run
format, resource model, and ownership of temporary names. This part completes the
same implementation file and public header. All three parts share the **6.0-hour
implementation allocation**. No prior chapter's source is replaced.

The important separation is between **a sorted chunk**, **a completed run**, and
**a sorted query stream**. A chunk can be sorted without disk access. A run can be
complete while other runs still contain smaller rows. Only the merge heap decides
which available head is globally next.

## Write a run in fixed frames

The writer borrows one output frame from the core. It owns a run reservation and
an open file, in that declaration order. Constructing the run first ensures that a
subsequent create or header-write failure cannot leave an ownerless name. A
constructor's destructor body does not execute when that constructor fails, so
the header-writing catch explicitly closes the file before member unwinding
retires the run.

Append the first portion of `Writer`. The class continues in the next two segments.

```cpp title="src/query/external_sort.cpp — complete file, segment 12"
class Writer {
public:
    Writer(Space& space, std::uint64_t rows, std::size_t block_rows, std::span<std::byte> buffer)
        : space_(space), run_(space.reserve(rows)), file_(space.create(run_)),
          block_rows_(block_rows), buffer_(buffer) {
        try {
            std::array<std::byte, header_bytes> header{};
            std::copy(magic.begin(), magic.end(), header.begin());
            auto bytes = std::span{header};
            put_le<4>(bytes.subspan(8, 4), 1); put_le<4>(bytes.subspan(12, 4), row_bytes);
            put_le<4>(bytes.subspan(16, 4), block_rows); put_le<4>(bytes.subspan(20, 4), 1);
            put_le<8>(bytes.subspan(24, 8), rows);
            put_le<4>(bytes.last(4), crc32(bytes.first(header_bytes - 4)));
            space_.write(file_, run_, 0, bytes);
        } catch (...) { space_.close_noexcept(file_); throw; }
    }
    ~Writer() noexcept { space_.close_noexcept(file_); }
    Writer(const Writer&) = delete;
    Writer& operator=(const Writer&) = delete;
```

The header knows the total number of rows before writing: an initial run knows its
chunk count, and a merge sums its input descriptors. It therefore needs no final
seek back to patch a count. This also makes the length reservation exact from the
beginning.

`append` verifies nondecreasing order and enforces the promised count. It encodes
one row into the available frame slot before publishing the buffered count. The
previous row is an owning scalar copy; no view into a mutable frame is retained.

```cpp title="src/query/external_sort.cpp — complete file, segment 13"
    void append(Event event) {
        if (written_ == space_.slot(run_).rows || (previous_ && EventOrder{}(event, *previous_))) {
            throw std::logic_error("run writer count or order");
        }
        encode_row(event, buffer_.subspan(8 + buffered_ * row_bytes, row_bytes));
        previous_ = event; ++buffered_; ++written_;
        if (buffered_ == block_rows_) { flush(); }
    }
    Run finish() {
        if (written_ != space_.slot(run_).rows) { throw std::logic_error("incomplete run output"); }
        flush();
        std::array<std::byte, footer_bytes> footer{};
        std::copy(tail_magic.begin(), tail_magic.end(), footer.begin());
        auto bytes = std::span{footer};
        put_le<8>(bytes.subspan(8, 8), written_); put_le<4>(bytes.subspan(16, 4), blocks_);
        put_le<4>(bytes.last(4), crc32(bytes.first(footer_bytes - 4)));
        space_.write(file_, run_, offset_, bytes);
        require(offset_ + footer_bytes == space_.slot(run_).bytes, "run forecast mismatch");
        space_.close(file_);  // Temporary query output is not fsync-backed recovery state.
        space_.completed(run_);
        return std::move(run_);
    }
```

`finish` rejects missing rows before emitting a footer. It flushes a partial last
frame, checks the computed final offset against the reservation, and closes the
file explicitly. No synchronization barrier installs anything durably: a later
process must not recover a query by guessing that a run with a footer is authority.

Returning `std::move(run_)` transfers the name reservation without copying a file
or allocating. If the completed-run hook throws, the move never happens and the
writer retains responsibility for that name. Here is the flush routine and the
remaining members that close the class:

```cpp title="src/query/external_sort.cpp — complete file, segment 14"
private:
    void flush() {
        if (buffered_ == 0) { return; }
        const auto length = 12 + buffered_ * row_bytes;
        auto frame = buffer_.first(length);
        put_le<4>(frame.first(4), buffered_); put_le<4>(frame.subspan(4, 4), buffered_ * row_bytes);
        put_le<4>(frame.last(4), crc32(frame.first(length - 4)));
        space_.write(file_, run_, offset_, frame);
        offset_ += length; ++blocks_; buffered_ = 0;
    }
    Space& space_;
    Run run_;
    io::File file_;
    std::size_t block_rows_;
    std::span<std::byte> buffer_;
    std::uint64_t offset_ = header_bytes, written_ = 0;
    std::size_t buffered_ = 0;
    std::uint32_t blocks_ = 0;
    std::optional<Event> previous_;
};
```

With frame capacity two, a three-row run follows this trace: write a 36-byte
header; append rows 0 and 1 and write a 68-byte frame at offset 36; append row 2;
finish by writing its 40-byte frame at offset 104 and a 24-byte footer at offset
144. The result ends at 168. The injected disk failure in Part 3 occurs before
the write at 104, after a genuine 104-byte prefix already exists.

## Validate a whole input frame before exposing its first row

A reader receives a trusted run descriptor and the configured block capacity. Its
one buffer is allocated from that capacity, **not** from bytes just read from a
file. It opens the run only after the buffer allocation succeeds. Header and footer
checks compare all relevant fields with the descriptor, including total count,
order ID, checksums, and expected number of frames.

The `remaining_` count is initialized from the trusted descriptor only after those
checks. This makes `(remaining_ - 1)` in the footer check safe: a run cannot have
zero rows. Append the constructor and its explicit failure cleanup:

```cpp title="src/query/external_sort.cpp — complete file, segment 15"
class Reader {
public:
    Reader(std::shared_ptr<memory::Budget> budget, Space& space, const Run& run, std::size_t block_rows)
        : space_(space), run_(run), block_rows_(block_rows),
          buffer_(std::move(budget), 12 + row_bytes * block_rows), file_(space.open(run)) {
        try {
            std::array<std::byte, header_bytes> header{};
            std::array<std::byte, footer_bytes> footer{};
            space_.read(file_, run_, 0, header);
            space_.read(file_, run_, space_.slot(run_).bytes - footer_bytes, footer);
            const auto h = std::span<const std::byte>{header}, f = std::span<const std::byte>{footer};
            require(std::equal(magic.begin(), magic.end(), h.begin()) && get_le<4>(h.subspan(8, 4)) == 1 &&
                get_le<4>(h.subspan(12, 4)) == row_bytes && get_le<4>(h.subspan(16, 4)) == block_rows_ &&
                get_le<4>(h.subspan(20, 4)) == 1 && get_le<8>(h.subspan(24, 8)) == space_.slot(run_).rows &&
                get_le<4>(h.last(4)) == crc32(h.first(header_bytes - 4)), "run header");
            remaining_ = space_.slot(run_).rows;
            require(std::equal(tail_magic.begin(), tail_magic.end(), f.begin()) &&
                get_le<8>(f.subspan(8, 8)) == remaining_ &&
                get_le<4>(f.subspan(16, 4)) == 1 + (remaining_ - 1) / block_rows_ &&
                get_le<4>(f.last(4)) == crc32(f.first(footer_bytes - 4)), "run footer");
            refill();
        } catch (...) { space_.close_noexcept(file_); throw; }
    }
    ~Reader() noexcept { space_.close_noexcept(file_); }
    Reader(const Reader&) = delete;
    Reader& operator=(const Reader&) = delete;
```

`head` returns an `Event` by value. A caller can safely advance after copying it.
`next` decrements the total remaining count and either stays within the current
frame, loads the next frame, or verifies the terminal offset and closes the file.
The total count and the current frame position answer different questions; neither
one can replace the other at a partial final block.

```cpp title="src/query/external_sort.cpp — complete file, segment 16"
    bool valid() const noexcept { return remaining_ != 0; }
    Event head() {
        if (!valid()) { throw std::logic_error("exhausted run"); }
        return decode_row(buffer_.span().subspan(8 + position_ * row_bytes, row_bytes));
    }
    void next() {
        if (!valid()) { throw std::logic_error("advance exhausted run"); }
        --remaining_; ++position_;
        if (remaining_ == 0) {
            require(offset_ == space_.slot(run_).bytes - footer_bytes, "run trailing bytes");
            space_.close(file_);
        } else if (position_ == loaded_) { refill(); }
    }
    void close() { space_.close(file_); }
```

A refill knows its exact expected row count: the smaller of frame capacity and
remaining rows. It therefore knows the frame length before reading. A malformed
stored length cannot request a huge allocation or skip over unchecked bytes.
After checking count, payload length, and CRC, walk **every row** in the frame to
check nondecreasing order. Carry the last validated row across the frame boundary.
Only then expose position zero.

```cpp title="src/query/external_sort.cpp — complete file, segment 17"
private:
    void refill() {
        const auto count = static_cast<std::size_t>(std::min<std::uint64_t>(block_rows_, remaining_));
        const auto length = 12 + count * row_bytes;
        auto frame = buffer_.span().first(length);
        space_.read(file_, run_, offset_, frame);
        require(get_le<4>(frame.first(4)) == count && get_le<4>(frame.subspan(4, 4)) == count * row_bytes &&
                get_le<4>(frame.last(4)) == crc32(frame.first(length - 4)), "run frame");
        // Validate the entire frame, including its boundary with the preceding frame.
        for (std::size_t i = 0; i < count; ++i) {
            const auto event = decode_row(frame.subspan(8 + i * row_bytes, row_bytes));
            require(!previous_ || !EventOrder{}(event, *previous_), "run ordering");
            previous_ = event;
        }
        offset_ += length; loaded_ = count; position_ = 0;
    }
    Space& space_;
    const Run& run_;
    std::size_t block_rows_;
    Scratch<std::byte> buffer_;
    io::File file_;
    std::uint64_t remaining_ = 0, offset_ = header_bytes;
    std::size_t loaded_ = 0, position_ = 0;
    std::optional<Event> previous_;
};
```

Suppose one frame ends with account 8 and the next starts with account 3. Both
frames could have internally sorted payloads and valid checksums. `previous_`
catches the cross-frame violation. Conversely, two identical records may straddle
a frame boundary and remain valid because the comparison is strict, not `<=`.

The reader itself is an internal noncopyable object. If advancement throws, its
caller terminates the surrounding sort and releases the reader; there is no
public retry position that pretends a partly consumed frame was rolled back.

## Keep the heap and all pass descriptors in the charged core

The core stores two fixed arrays of run owners, one fixed array of optional
readers, and an index heap. An engaged `optional<Reader>` means a reader object
exists in that slot; `emplace` constructs it without relocating another reader,
and `reset` closes/releases it. The heap contains indices into those stable slots,
not copied readers, spans, or owning files.

`std::array::swap` later exchanges current and next run owners element by element.
Each run's nonthrowing move transfers its `Space` pointer and slot index. All
readers have been closed before that exchange, so no reader retains a reference
to a run object whose ownership is changing.

```cpp title="src/query/external_sort.cpp — complete file, segment 18"
class Core {
public:
    Core(std::shared_ptr<memory::Budget> budget, SortOptions options, std::string_view parent,
         std::shared_ptr<SortStats> stats, SpillHook hook)
        : budget_(std::move(budget)), options_(options), stats_(std::move(stats)),
          space_(options, parent, stats_, hook) {}
    ~Core() noexcept { cleanup(); }
    void initialize(std::size_t fixed_charge);
    void append(Event event);
    void finish();
    Pull next(EventBatch& output);
    void abort() noexcept { state_ = StreamState::failed; cleanup(); chunk_.reset(); frame_.reset(); stack_.reset(); }
    void retry_cleanup() noexcept { space_.retry(); if (sealed_ || state_ == StreamState::failed) { space_.remove_directory(); } }
    std::shared_ptr<memory::Budget> budget() const noexcept { return budget_; }
private:
    void spill_chunk();
    void merge_pass();
    void open_group(std::size_t first, std::size_t count);
    void close_group();
    void cleanup() noexcept;
    bool before(std::size_t a, std::size_t b);
    void push(std::size_t reader);
    Event take();
    void sift_down();
    void account_workspace(std::size_t input_frames);
    std::shared_ptr<memory::Budget> budget_;
    SortOptions options_;
    std::shared_ptr<SortStats> stats_;
    Space space_;
    memory::Budget::Lease stack_;
    std::optional<Scratch<Event>> chunk_;
    std::optional<Scratch<std::byte>> frame_;
    std::array<Run, max_runs> runs_{}, next_runs_{};
    std::array<std::optional<Reader>, max_fan_in> readers_{};
    std::array<std::size_t, max_fan_in> heap_{};
    std::size_t rows_ = 0, read_at_ = 0, run_count_ = 0, heap_size_ = 0;
    StreamState state_ = StreamState::ready;
    bool sealed_ = false;
};
```

The explicit destructor calls `cleanup` while all its members still exist. It
closes readers, retires names, and attempts directory cleanup. Normal member
unwinding then releases scratch arrays, the reservation, `Space`, and shared
handles. Already-reset run owners are harmless. If initialization throws after
only one scratch allocation, optional-member destruction releases that allocation.

## Admit the minimum useful merge before consuming input

Initialization applies Part 1's formulas with checked size arithmetic. The facade
has already validated positive row capacities and a descriptor allowance of at
least three. `fixed_charge` includes the actual allocator request for core/control
storage; it is not guessed from `sizeof(Core)` alone.

Only after proving that the local cap admits generation and a two-input merge does
initialization reserve $S$ and allocate the exact chunk and output frame. Root
budget admission may still refuse either request. No input row or temporary name
has been consumed at this point.

```cpp title="src/query/external_sort.cpp — complete file, segment 19"
void Core::initialize(std::size_t fixed_charge) {
    stats_->fixed_charge = fixed_charge;
    const auto frame = checked_size_add(12, checked_size_mul(row_bytes, options_.block_rows));
    const auto fixed = checked_size_add(fixed_charge, options_.stack_allowance);
    const auto generation = checked_size_add(checked_size_mul(sizeof(Event), options_.chunk_rows), frame);
    const auto minimum = checked_size_add(fixed, std::max(generation, checked_size_mul(3, frame)));
    if (minimum > options_.workspace_bytes) { throw std::length_error("sort workspace cannot make progress"); }
    stats_->fan_in = std::min({max_fan_in, options_.file_descriptors - 1,
                              (options_.workspace_bytes - fixed - frame) / frame});
    if (stats_->fan_in < 2) { throw std::length_error("sort needs two input buffers"); }
    stack_ = budget_->reserve(options_.stack_allowance);
    chunk_.emplace(budget_, options_.chunk_rows);
    frame_.emplace(budget_, frame);
    account_workspace(0);
}
void Core::account_workspace(std::size_t input_frames) {
    auto bytes = checked_size_add(stats_->fixed_charge, options_.stack_allowance);
    if (chunk_) { bytes = checked_size_add(bytes, chunk_->bytes()); }
    if (frame_) { bytes = checked_size_add(bytes, frame_->bytes()); }
    bytes = checked_size_add(bytes, checked_size_mul(input_frames, 12 + row_bytes * options_.block_rows));
    if (bytes > options_.workspace_bytes) { throw std::length_error("sort workspace allowance"); }
    stats_->peak_workspace = std::max(stats_->peak_workspace, bytes);
}
```

`account_workspace` checks phase-specific coexistence and records the modeled peak.
It runs before opening a group of readers. It includes the dynamic arrays whose
allocations follow, while the root ledger independently enforces every actual
request. The final streaming pass conservatively retains the output frame even
though it is no longer writing an intermediate file. Removing that small unused
buffer would be an optimization, not permission to increase earlier fan-in.

## Delay the first spill until one more row proves it necessary

If a chunk fills exactly and the upstream ends, the query can still use the
no-spill path. Do not spill merely because the chunk became full. On the **next
append**, spill the full previous chunk before copying the arriving event into
slot zero. The input batch owns that arriving event throughout the call.

For capacity three, appending rows A, B, and C creates no file. Receiving D sorts
and writes A–C, then stores D. If no D exists, `finish` sorts A–C in memory. Empty
filtered input batches never cause a flush or an end transition.

```cpp title="src/query/external_sort.cpp — complete file, segment 20"
void Core::spill_chunk() {
    if (run_count_ == options_.max_runs) { throw std::length_error("initial run descriptor cap"); }
    auto rows = chunk_->span().first(rows_);
    std::sort(rows.begin(), rows.end(), EventOrder{});
    Writer writer{space_, rows_, options_.block_rows, frame_->span()};
    for (const auto& event : rows) { writer.append(event); }
    runs_[run_count_++] = writer.finish();
    ++stats_->initial_runs; rows_ = 0;
}
void Core::append(Event event) {
    if (state_ != StreamState::ready || sealed_) { throw std::logic_error("sort not accepting input"); }
    try {
        const auto maximum = checked_size_mul(options_.chunk_rows, options_.max_runs);
        if (stats_->input_rows == maximum) { throw std::length_error("sort input cap"); }
        if (rows_ == options_.chunk_rows) { spill_chunk(); }
        chunk_->span()[rows_++] = event; ++stats_->input_rows;
    } catch (...) { abort(); throw; }
}
```

The input cap is checked before appending the row that would exceed it. The query
still becomes terminal because its upstream may already have delivered that row
inside an owned batch. Rejecting the row is not equivalent to restoring the whole
upstream position. Likewise, a failed spill preserves neither a reusable append
position nor a promise that previously generated runs remain available: abort
cleans this query's temporary work.

`std::sort` handles only this bounded, contiguous chunk. It is not an external
sorting library. In-memory comparison work is $O(n\log C)$ over $n$ input rows
with chunk capacity $C$, ignoring smaller final-chunk effects. External I/O,
buffering, pass scheduling, and all merging remain our implementation.

## A heap of current heads is enough

For every nonempty input run, the first unread row is the smallest remaining row
in that run. Therefore the smallest **among those heads** is the next global row.
No later row can be smaller without violating the run's validated order.

Heap index $i$ has children $2i+1$ and $2i+2$ and, for $i>0$, parent
$\lfloor(i-1)/2\rfloor$. Maintain that a parent does not compare greater than either
child. `before(a,b)` compares the owning values at reader slots `a` and `b`.
Identical records use the reader index as an internal deterministic tie decision;
this does not manufacture an input-stability guarantee.

```cpp title="src/query/external_sort.cpp — complete file, segment 21"
bool Core::before(std::size_t a, std::size_t b) {
    const auto x = readers_[a]->head(), y = readers_[b]->head();
    if (EventOrder{}(x, y)) { return true; }
    if (EventOrder{}(y, x)) { return false; }
    return a < b;  // Exact duplicates are indistinguishable, but heap decisions stay deterministic.
}
void Core::push(std::size_t reader) {
    std::size_t at = heap_size_++;
    while (at != 0) {
        const auto parent = (at - 1) / 2;
        if (!before(reader, heap_[parent])) { break; }
        heap_[at] = heap_[parent]; at = parent;
    }
    heap_[at] = reader;
}
```

`push` saves the new reader index and moves larger parent indices downward until
its insertion position is reached. It never reallocates and never swaps a reader.
At most the height of a heap with $k$ elements is traversed.

For example, three reader heads keyed `(account,event)` might be `(2,8)`, `(1,9)`,
and `(2,3)`. Inserting slots 0, 1, and 2 produces a root at slot 1. Removing `(1,9)`
may advance that reader to `(4,1)`. The root's key increased; repair downward to
choose `(2,3)` before `(2,8)`, not whichever run was opened first.

```cpp title="src/query/external_sort.cpp — complete file, segment 22"
void Core::sift_down() {
    if (heap_size_ == 0) { return; }
    const auto replacement = heap_[0];
    std::size_t at = 0;
    while (at * 2 + 1 < heap_size_) {
        auto child = at * 2 + 1;
        if (child + 1 < heap_size_ && before(heap_[child + 1], heap_[child])) { ++child; }
        if (!before(heap_[child], replacement)) { break; }
        heap_[at] = heap_[child]; at = child;
    }
    heap_[at] = replacement;
}
Event Core::take() {
    const auto index = heap_[0];
    const auto result = readers_[index]->head();  // Copy before advancing the reader's buffer.
    readers_[index]->next();
    if (!readers_[index]->valid()) { heap_[0] = heap_[--heap_size_]; }
    sift_down();
    return result;
}
```

`take` copies the winning row **before** advancing its reader. Advancing may reuse
the frame buffer or close the file. If that reader is exhausted, replace the root
index with the last heap index and shrink the heap; otherwise its new head remains
at the root until downward repair. One repair covers both cases.

A refill exception can occur after selecting a row but before `take` returns it.
The public pull catches that failure, clears the current output batch, and becomes
terminal. An earlier separately owned output batch remains valid, but it is only
a prefix of a failed query. Returning the selected row and silently skipping a
failed refill would be a correctness bug, not graceful degradation.

The open/close helpers construct the bounded reader group and build its heap:

```cpp title="src/query/external_sort.cpp — complete file, segment 23"
void Core::open_group(std::size_t first, std::size_t count) {
    account_workspace(count);
    heap_size_ = 0;
    for (std::size_t i = 0; i < count; ++i) {
        readers_[i].emplace(budget_, space_, runs_[first + i], options_.block_rows);
        push(i);
    }
}
void Core::close_group() {
    for (auto& reader : readers_) { if (reader) { reader->close(); reader.reset(); } }
    heap_size_ = 0;
}
```

Readers are inserted one by one. If allocating or opening a later reader fails,
the outer terminal cleanup resets every already-engaged reader. Each still-owned
run remains available until cleanup deliberately retires it. No descriptor or
frame buffer is acquired for a reader beyond the computed group size.

## Materialize one replacement before retiring its inputs

For a materialized group, sum its row counts, open at most $k$ readers, then create
one output run. The disk reservation still includes all inputs. Repeatedly take
the minimum head and append it to the output writer. The writer checks order and
count independently of the heap; a broken root repair often fails there before
the final oracle comparison.

After checked output close, close/reset input readers but keep their **names and
run owners**. Reopen and fully verify the replacement using one buffer. Only then
move the output into the next-pass array and retire the old names. This deliberately
adds a verification read of every materialized replacement. Its cost is counted,
not hidden as a free assertion.

```cpp title="src/query/external_sort.cpp — complete file, segment 24"
void Core::merge_pass() {
    std::size_t produced = 0;
    for (std::size_t first = 0; first < run_count_;) {
        const auto count = std::min(stats_->fan_in, run_count_ - first);
        if (count == 1) { next_runs_[produced++] = std::move(runs_[first++]); continue; }
        std::uint64_t rows = 0;
        for (std::size_t i = 0; i < count; ++i) { rows = checked_u64_add(rows, space_.slot(runs_[first + i]).rows); }
        open_group(first, count);
        Run output;
        {
            Writer writer{space_, rows, options_.block_rows, frame_->span()};
            while (heap_size_ != 0) { writer.append(take()); }
            output = writer.finish();
        }
        close_group();
        // Verify the replacement using one input buffer BEFORE retiring any input name.
        account_workspace(1);
        {
            Reader verify{budget_, space_, output, options_.block_rows};
            while (verify.valid()) { verify.next(); }
        }
        next_runs_[produced++] = std::move(output);
        for (std::size_t i = 0; i < count; ++i) { runs_[first + i].reset(); }
        ++stats_->merge_groups;
        first += count;
    }
    runs_.swap(next_runs_); run_count_ = produced; ++stats_->merge_passes;
}
```

A singleton group is carried by moving its owner; copying its bytes would provide
no reduction in run count. The row at a group boundary belongs to exactly one run
and is never consumed merely to determine that the next output is full.

Let $R_p$ be the number of runs before materialized pass $p$, and let $k$ be fan-in.
Grouping gives

$$
R_{p+1}=\left\lceil\frac{R_p}{k}\right\rceil.
$$

Because $k\ge2$, this strictly reduces any $R_p>k$. Stop materializing once
$R_p\le k$: those remaining runs can feed the final streaming merge directly.
For 25 rows, chunk capacity three, and fan-in two:

| Stage | Run row counts | Newly written files |
|---|---|---:|
| Initial generation | `3,3,3,3,3,3,3,3,1` | 9 |
| Materialized pass 1 | `6,6,6,6,1` | 4 |
| Materialized pass 2 | `12,12,1` | 2 |
| Materialized pass 3 | `24,1` | 1 |
| Final streaming merge | 25 rows delivered as consumer batches | 0 |

`merge_passes` counts the three completed materialized passes, not the final pull
merge. `merge_groups` counts the seven materialized multi-input groups; singletons
are not counted as rewritten groups. A failed partial pass can create completed
files without incrementing `merge_passes`. Those definitions matter when reading
failure statistics.

With $k$ inputs, each merged row performs at most $O(\log k)$ heap work. More passes
increase total reads and writes. Larger fan-in can reduce passes, but its simultaneous
buffers, open files, and replacement space must be admitted together.

## Seal once, then stream according to consumer capacity

`finish` distinguishes the in-memory path from the spill path. No-spill sorts only
the visible chunk and releases the unused output frame. Spill mode writes the
remaining nonempty chunk, destroys the chunk allocation, materializes necessary
passes, and opens the final group. The `sealed_` flag is set only after that
preparation succeeds.

```cpp title="src/query/external_sort.cpp — complete file, segment 25"
void Core::finish() {
    if (state_ != StreamState::ready || sealed_) { throw std::logic_error("sort finish state"); }
    try {
        if (run_count_ == 0) {
            auto rows = chunk_->span().first(rows_);
            std::sort(rows.begin(), rows.end(), EventOrder{});
            frame_.reset();
        } else {
            if (rows_ != 0) { spill_chunk(); }
            chunk_.reset();  // Merge fan-in may use the former chunk's allowance.
            while (run_count_ > stats_->fan_in) { merge_pass(); }
            open_group(0, run_count_);
        }
        sealed_ = true;
    } catch (...) { abort(); throw; }
}
```

At no point does the merge collect all output rows in an owning vector. The output
consumer can choose capacity one or a larger batch without changing run contents,
heap membership, or the sequence of values. The later join will rely on that
property when its own batch fills in the middle of a hot account's events.

Cleanup closes readers before retiring either pass's run owners. It may run after
a normal end, an error, explicit abort, or destructor abandonment. Run reset is
idempotent, and pending cleanup is retried without throwing.

```cpp title="src/query/external_sort.cpp — complete file, segment 26"
void Core::cleanup() noexcept {
    for (auto& reader : readers_) { reader.reset(); }
    heap_size_ = 0;
    for (auto& run : runs_) { run.reset(); }
    for (auto& run : next_runs_) { run.reset(); }
    run_count_ = 0;
    space_.retry(); space_.remove_directory();
}
Pull Core::next(EventBatch& output) {
    check_output(output, budget_, 1, state_);
    if (!sealed_) { throw std::logic_error("finish sort before reading"); }
    output.clear();
    if (state_ == StreamState::ended) { return Pull::end; }
    try {
        if (chunk_) {
            while (read_at_ < rows_ && output.size() < output.capacity()) { output.append(chunk_->span()[read_at_++]); }
        } else {
            while (heap_size_ != 0 && output.size() < output.capacity()) { output.append(take()); }
        }
        if (output.size() == 0) {
            state_ = StreamState::ended; cleanup(); chunk_.reset(); frame_.reset(); stack_.reset();
            return Pull::end;
        }
        stats_->output_rows += output.size();
        return Pull::batch;
    } catch (...) { output.clear(); abort(); throw; }
}
```

A last nonempty output is still `Pull::batch`. A subsequent call returns an empty,
absorbing `Pull::end` and releases temporary runs, buffers, and the stack allowance.
Abandoning before that call still triggers destructor cleanup. Fixed core/control
storage and the observation object remain charged while their owners remain alive.

`output_rows` advances only after a successful nonempty pull. Rows assembled in a
refill that later throws are cleared and do not count as delivered. The lower-level
read counters may nevertheless increase: useful output and transferred bytes are
different measurements.

The facade validates public options before allocation. It rejects a missing ledger,
a missing or previously used report, impossible row capacities, fewer than three
run descriptors, a zero disk allowance, or an undersized stack policy allowance.
The minimum stack allowance is a policy floor of 16 KiB, not a derived guarantee
about every standard-library build.

```cpp title="src/query/external_sort.cpp — complete file, segment 27"
}  // namespace sort_detail

SortBuffer::SortBuffer(std::shared_ptr<memory::Budget> budget, SortOptions options,
                       std::string_view parent, std::shared_ptr<SortStats> stats, SpillHook hook) {
    if (!budget || !stats || stats->used || options.chunk_rows == 0 || options.chunk_rows > (1U << 20) ||
        options.block_rows == 0 || options.block_rows > 4096 || options.max_runs < 2 ||
        options.max_runs > sort_detail::max_runs || options.file_descriptors < 3 ||
        options.stack_allowance < 16 * 1024 || options.disk_bytes == 0) {
        throw std::invalid_argument("invalid sort configuration or reused statistics");
    }
    stats->used = true;
    const auto before = budget->allocated();
    core_ = std::allocate_shared<sort_detail::Core>(memory::SharedAllocator<sort_detail::Core>{budget},
        budget, options, parent, stats, hook);
    // Synchronous construction performs no other allocation before this observation.
    const auto fixed_charge = budget->allocated() - before;
    core_->initialize(fixed_charge);
}
void SortBuffer::append(Event event) & { core_->append(event); }
void SortBuffer::finish() & { core_->finish(); }
Pull SortBuffer::next(EventBatch& output) & { return core_->next(output); }
void SortBuffer::abort() noexcept { if (core_) { core_->abort(); } }
void SortBuffer::retry_cleanup() noexcept { if (core_) { core_->retry_cleanup(); } }
std::shared_ptr<memory::Budget> SortBuffer::budget() const noexcept { return core_->budget(); }

}  // namespace strata::query
```

This closes `external_sort.cpp`. `allocate_shared` uses the owning allocator for
the actual core/control storage. Its constructor performs no other charged
allocation; the difference in `allocated()` therefore identifies $H$ under the
current synchronous, nonreentrant contract. `initialize` allocates dynamic buffers
after that observation. If it throws, constructor unwinding releases the partially
built core through the same allocator.

## Adapt a progress-producing source to a blocking operator

A sort is a **blocking operator**: before returning its first correctly ordered
row, it must finish consuming the input. After sealing, its output is streamed in
bounded batches. These two facts are compatible. Incremental output does not mean
we can emit a row before learning whether a smaller key appears later.

Append the remaining header segment. The source is owned by value, as in Chapter
14. The input batch shares its budget and must satisfy that source's required
capacity. The consumer output may use any positive capacity on the same ledger.

```cpp title="include/strata/query/external_sort.hpp — complete file, segment 2"
template<class Source> requires PullSource<Source, EventBatch>
class ExternalSort {
public:
    ExternalSort(Source source, std::size_t input_capacity, SortOptions options,
                 std::string_view temporary_parent, std::shared_ptr<SortStats> stats,
                 SpillHook hook = {})
        : source_(std::move(source)), input_(source_.budget(), input_capacity),
          sort_(source_.budget(), options, temporary_parent, std::move(stats), hook) {}
    ExternalSort(const ExternalSort&) = delete;
    ExternalSort& operator=(const ExternalSort&) = delete;
    ExternalSort(ExternalSort&&) = default;
    ExternalSort& operator=(ExternalSort&&) = default;
    std::shared_ptr<memory::Budget> budget() const noexcept { return sort_.budget(); }
    Pull next(EventBatch& output) & {
        check_output(output, budget(), 1, state_);  // No consumption on bad output.
        output.clear();
        if (state_ == StreamState::ended) { return Pull::end; }
        try {
            if (!built_) {
                for (;;) {
                    const auto result = source_.next(input_);
                    if (result == Pull::end) {
                        if (input_.size() != 0) { throw std::logic_error("nonempty sort end"); }
                        break;
                    }
                    // An empty progress batch is NOT end-of-stream.
                    for (std::size_t i = 0; i < input_.size(); ++i) { sort_.append(input_.row(i)); }
                }
                sort_.finish();
                built_ = true;
            }
            const auto result = sort_.next(output);
            if (result == Pull::end) { state_ = StreamState::ended; }
            return result;
        } catch (...) {
            output.clear(); state_ = StreamState::failed; sort_.abort(); throw;
        }
    }
    void retry_cleanup() noexcept { sort_.retry_cleanup(); }
private:
    Source source_;
    EventBatch input_;
    SortBuffer sort_;
    StreamState state_ = StreamState::ready;
    bool built_ = false;
};

}  // namespace strata::query
#endif
```

The loop ends only on `Pull::end`, which must have an empty batch. An empty
`Pull::batch` causes another upstream step; the source contract requires that step
to represent progress. There is no arbitrary "three empty batches means EOF"
shortcut. An invalid source that returns empty progress forever can still keep
this synchronous call busy; a concept checks the expression shapes, not semantic
termination.

`built_` becomes true only after `finish` succeeds. An exception anywhere during
input consumption, run generation, merge preparation, or output refill clears
current output, marks the adapter failed, and aborts the sort. Calling it again
cannot retry consumed input. Incorrect outer capacity or ledger is rejected
outside that execution catch and is therefore retryable without losing a sentinel
output row.

The defaulted adapter moves use the compiler's member-dependent exception
specification. We do not assert that every type satisfying `PullSource` has a
nonthrowing move. The provided source types do, and `SortBuffer` itself moves by
transferring shared ownership. A future throwing source must preserve its own
move contract; the concept alone does not provide one.

The source and input batch remain owned until adapter destruction, even after
sorting finishes. Thus a storage-backed plan can retain its snapshot throughout
output consumption. This is conservative, not free: its source owners and column
capacity remain in the common ledger and retention registry. Releasing the upstream
early would require an explicit optional-source lifetime change, not a dangling
borrow to save a few members.

## Count the I/O the implementation actually performs

For the nine-run example, initial output is 1,444 bytes. Materialized passes write
$4(264)=1056$, $2(468)=936$, and $876$ bytes. Total successful writes are

$$
W_{\mathrm{io}}=1444+1056+936+876=4312\text{ bytes}.
$$

Each created run is eventually read once as a merge input or a final input.
That accounts for another 4,312 bytes. Every materialized replacement is also
fully verified, adding $1056+936+876=2868$. Therefore successful read transfers are

$$
R_{\mathrm{io}}=4312+2868=7180\text{ bytes}.
$$

Headers and footers are read once per open, and each data frame once per traversal;
the verifier does not reread a frame for each row. The independent native fixture
checks these totals against the actual file lengths and values. It does not infer
SSD traffic or elapsed-time performance from them.

Continue with [Part 3 — Pressure, corruption, and the native run oracle](https://25d-228.github.io/cplusplus-learning/chapter-15-part-3.html).
