# Chapter 10 · Part 2 — Commit, uncertainty, and strict replay

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-10.html) defined the
file/frame format, public owner, encoder, and bounded payload decoder. Continue
from those exact files and the Chapter 9 prerequisite state. This part supplies
the complete write/recovery path and the small I/O extension it requires.

## The persistence assumptions are part of the algorithm

The log owner has exclusive access to one existing file incarnation. Nobody else
appends, truncates, replaces, or mutates it while it is open. There is one writer,
one synchronous budget domain, and no reentrant call. The immutable table cache
must not cache this mutable log.

For the following recovery argument, successful synchronization with the selected
mode must retain the synchronized prefix and its length, and later appends must
not damage that prefix. An interrupted unsynchronized append may leave no bytes,
a correct prefix of its frame, or the complete valid frame. Recovery also accepts
complete valid unacknowledged batches. A full-sized damaged frame is reported as
corruption, not silently treated as an uncommitted batch.

These are explicit assumptions, not conclusions of a unit test. Real failures can
leave non-prefix garbage, damage an earlier sector, lose directory entries, or
violate a claimed flush guarantee. The strict parser rejects many such images,
but it does not reconstruct missing data. In particular, removal of an entire
acknowledged suffix at a frame boundary can look exactly like a shorter valid
log. There is no durable acknowledgement ledger here that could distinguish them.
A valid prefix is not proof that all once-acknowledged bytes survived.

Apple's archived [fsync manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
distinguishes host-to-device flushing from `F_FULLFSYNC` drive-cache requests.
Inspect `man 2 fsync` and `man 2 fcntl` on the actual Mac. The caller must choose
`SyncMode::file` or `SyncMode::full_device` explicitly; an unsupported full-device
request fails rather than falling back. The native checks later use file sync by
default and provide a separate `--full` option. Neither setting plus a successful
ordinary process run is a physical power-loss experiment.

Nor does file synchronization install the filename or a database manifest. This
chapter assumes that the intended file is available at reopen. Initial creation,
namespace persistence, generation selection, and retirement ordering need the
next chapter's installation protocol. SQLite's
[hardware assumptions and atomic-commit testing discussion](https://www.sqlite.org/atomiccommit.html)
is useful for separating a persistence model from hardware behavior; its
rollback-journal protocol is not being copied into this WAL.

## Remove a tail rather than append after it

Suppose the good end is 132 and a failed append leaves 17 bytes, making physical
length 149. Merely remembering 132 in memory is not enough. A shorter later frame
could leave stale bytes beyond its end, and appending at 149 would put a new frame
after an incomplete one. Recovery must cut the file back to the verified end
before it starts a new append history.

Extend the existing file boundary without changing its old methods. In
`include/strata/io/file.hpp`, add `#include <cerrno>`. Inside `FileOps`, immediately
after its `close` declaration, add the following method:

```cpp title="include/strata/io/file.hpp — add FileOps method"
virtual int truncate_to(int, std::uint64_t) noexcept { return ENOTSUP; }
```

A default unsupported response keeps earlier deterministic backends honest:
backends that have no truncate behavior do not pretend to perform it, and their
existing read/write tests still compile. A new backend must override this method
before it can repair a tail. Unnamed parameters are legal because the default
implementation does not use them. The return convention remains zero or a
positive error number, not an exception.

Inside `File`'s public section, immediately after `close_checked`, add:

```cpp title="include/strata/io/file.hpp — add File declaration"
void truncate_to(std::uint64_t length);
```

In `src/io/file.cpp`, add this override inside `NativeOps`, before `close`:

```cpp title="src/io/file.cpp — add NativeOps method"
int truncate_to(int descriptor, std::uint64_t length) noexcept override {
        if (::ftruncate(descriptor, static_cast<off_t>(length)) == 0) { return 0; }
        return errno;
    }
```

The existing `<unistd.h>` and `off_t` checks already provide the native declaration
and offset type. The wrapper below checks the length before this cast. The
[archived Apple truncate manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/truncate.2.html)
describes shrinking a writable regular file and reporting failures. This wrapper
passes a target length; the recovery code uses it only to shrink. Truncation is
an externally visible operation, not a memory rollback.

Add this complete `File` definition before the closing `strata::io` namespace
brace. Do not place it inside `NativeOps`:

```cpp title="src/io/file.cpp — add File method"
void File::truncate_to(std::uint64_t length) {
    require_open();
    (void)checked_file_end(length, 0);
    int error;
    do { error = operations_->truncate_to(descriptor_, length); } while (error == EINTR);
    if (error != 0) { throw IoError(error, "truncate_to", length, 0); }
}
```

`checked_file_end(length, 0)` enforces the same nonnegative signed-offset domain
as earlier transfers. Retrying an interrupted request for the same target length
is appropriate under exclusive ownership; it is not the blind descriptor-close
retry rejected in Chapter 6. A non-interruption error propagates. The caller must
not infer that the external file stayed unchanged merely because an error was
reported. A subsequent synchronization is still required.

These are all changes to prerequisite engine files in this chapter. Do not alter
the earlier transfer loops, constructor signatures, allocator, or memtable code.

## Construct privately, then expose one usable owner

Append the constructor and initialization function to `src/storage/wal.cpp`,
which Part 1 left inside `namespace strata::storage`:

```cpp title="src/storage/wal.cpp"
WalMemTable::WalMemTable(memory::Budget& budget, io::File file,
    std::size_t arena_cap, WalOpen open_mode, io::SyncMode sync_mode,
    std::uint64_t base_sequence)
    : budget_(budget), file_(std::move(file)), table_(budget, arena_cap),
      sync_mode_(sync_mode), sequence_(base_sequence) {
    if (!file_.is_open() || (sync_mode != io::SyncMode::file &&
        sync_mode != io::SyncMode::full_device)) {
        throw std::invalid_argument("invalid WAL file or sync mode");
    }
    switch (open_mode) {
    case WalOpen::create: initialize(base_sequence); break;
    case WalOpen::recover: replay(base_sequence); break;
    default: throw std::invalid_argument("invalid WAL open mode");
    }
}

void WalMemTable::initialize(std::uint64_t base) {
    std::array<std::byte, 1> probe{};
    if (file_.read_up_to(0, probe) != 0) {
        throw std::invalid_argument("new WAL must be empty");
    }
    const auto header = file_header(base);
    file_.write_all(0, header);
    file_.sync(sync_mode_);
}
```

The file is moved into its member before the constructor body runs. The memtable
starts empty under the supplied budget and cap. Invalid modes fail before log
I/O. The creation path probes for existing bytes, writes the complete format header, then
synchronizes it. A failed create leaves no usable owner and may leave a partial
external file. `OpenMode::create_new` in the caller prevents overwriting a file
that already exists; this is not an orphan-file discovery policy.

Recovery happens inside construction, before a caller can borrow `table()`. It
may replay several batches internally and then fail on another. That is safe
because the caller never receives the half-recovered object. C++ destroys fully
constructed member subobjects when construction throws; it does not call the
unfinished complete object's destructor. Here the memtable destroys its nodes
and arena chunks, and the file member closes its descriptor. See
[the C++20 constructor/unwinding rules](https://timsong-cpp.github.io/cppwp/n4861/except.ctor).
The external budget/backend owners remain the caller's responsibility.

The operation returns no movable database handle: the caller constructs the
owner directly in its final scope. This avoids requiring move support from the
memtable. For this harness, insufficient replay memory is an opening failure;
there is no hidden partial database available for reads or writes afterward.

## Prepare everything that could allocate before append

The write path first rejects a poisoned writer or an exhausted sequence domain.
Otherwise it builds the frame, checks the log byte cap, and prepares all memtable
nodes. Only then does it enter the region where an error may have an uncertain
external effect. Append this function in full:

```cpp title="src/storage/wal.cpp"
WriteResult WalMemTable::write(std::span<const Mutation> changes) & {
    if (poisoned_) {
        return {WriteOutcome::rejected, WriteReason::writer_poisoned, 0};
    }
    if (sequence_ == std::numeric_limits<std::uint64_t>::max()) {
        return {WriteOutcome::rejected, WriteReason::sequence_exhausted, 0};
    }
    const auto next = sequence_ + 1;
    try {
        auto image = wal_format::encode_batch(budget_, next, changes);
        if (image.size() > wal_format::max_log_bytes - end_) {
            return {WriteOutcome::rejected, WriteReason::log_full, 0};
        }
        auto prepared = table_.prepare(next, changes);
        // From this point onward an exception cannot prove that no bytes persisted.
        poisoned_ = true;
        try {
            file_.write_all(end_, image);
            file_.sync(sync_mode_);
        } catch (const io::IoError& error) {
            return {WriteOutcome::uncertain, WriteReason::io_failure, next,
                    error.code().value()};
        } catch (...) {
            return {WriteOutcome::uncertain, WriteReason::unexpected_failure, next};
        }
        prepared.commit();
        end_ += image.size();
        sequence_ = next;
        poisoned_ = false;
        return {WriteOutcome::committed, WriteReason::none, next};
    } catch (const FormatError&) {
        return {WriteOutcome::rejected, WriteReason::invalid_batch, 0};
    } catch (const std::invalid_argument&) {
        return {WriteOutcome::rejected, WriteReason::invalid_batch, 0};
    } catch (const std::length_error&) {
        return {WriteOutcome::rejected, WriteReason::invalid_batch, 0};
    } catch (const memory::ArenaFull&) {
        return {WriteOutcome::rejected, WriteReason::resource_exhausted, 0};
    } catch (const std::bad_alloc&) {
        return {WriteOutcome::rejected, WriteReason::resource_exhausted, 0};
    }
}
```

Read the nested `try` blocks as two different error domains, not two arbitrary
styles of exception handling. The outer region has not performed append while
encoding and preparation can throw. Known bad input and resource failures become
definite rejection. An unexpected programming error there is not silently
converted into a resource shortage; it can propagate before append.

The inner region begins with `poisoned_ = true`, before `write_all`. Even an error
on the first backend call is classified conservatively as uncertain. This layer
cannot generally prove what an external device did, and the caller should not
interpret `IoError::transferred() == 0` as a universal persistence guarantee.
There is no attempt to rewind the log in the failing writer.

An `IoError` preserves its integer code in the result. `catch (...)` means “catch
any remaining C++ exception,” not “catch signals, process termination, or machine
failure.” It is used only around the irreversible I/O boundary and returns
`unexpected_failure` **without clearing poison**. For example, constructing an
I/O diagnostic could itself encounter memory exhaustion after bytes were written.
That must not reach the outer `bad_alloc` handler as a definite rejection.

On either uncertain return, the local `prepared` token is destroyed and rolls
back unpublished nodes; then the image vector releases its allocation. Returning
from inside a `try` still destroys these automatic objects. The old table remains
visible, and its old payload views remain valid, but the physical log may have a
new suffix. Diagnostic counters and high-water memory are not rolled back.

After a successful synchronization, `prepared.commit()` performs only the
nonthrowing link publication already implemented in Chapter 7. End/sequence
updates and constructing the small result cannot allocate. The local owners'
cleanup is nonthrowing. This is why a late allocation error cannot turn a reported
success into a missing in-memory batch. Marking an arbitrary allocating function
`noexcept` would not solve that problem: an escaping exception would terminate,
as specified by [the C++20 termination rules](https://timsong-cpp.github.io/cppwp/n4861/except.terminate).
We rely on the actual commit algorithm, not just the keyword.

The writer does not export a prepared token or allow another mutation between
encoding and preparation. The caller's input must remain unchanged through the
synchronous call. The table is exposed only as const, so a normal caller cannot
freeze it or install an unrelated sequence through this interface.

### Trace a partial external effect

Start with the two first-batch rows, sequence 1, and published end 132. For batch
2, the encoder owns a 100-byte image and preparation owns two unlinked nodes.
The test backend accepts 17 bytes in several short transfers, then returns an
I/O error. The physical end is $132+17=149$; the published end stays 132.

The return is `{uncertain, io_failure, 2, EIO}`. Pending node lifetimes end,
prepared arena usage is restored, and the frame workspace is released. The old
event still contains 500, and the account still contains region 1. Trying another
write on this owner returns `writer_poisoned` without another backend write.

With a synchronization error instead, all 100 frame bytes can exist at offsets
132 through 231, but publication still has not happened. Reopening can legitimately
show 700 and region 2. These two outcomes are not contradictory; the original
return was uncertain, not an assertion that the operation never existed.

### Count temporary memory as well as retained nodes

Let $C_f$ be the allocated frame vector's byte capacity and $A$ the bytes of all
currently acquired arena chunks. The writer's dynamic charge during preparation
includes $A+C_f$, plus any other live engine consumers sharing the budget.
Pending payload suballocations are not charged a second time: they live inside
those charged chunks. The caller's inline mutation array is outside this engine
allocation ledger, as are stack objects, exception/path/runtime storage, allocator
internals, and the operating-system file cache.

In replay, let $C_b$ be the body-buffer byte capacity and $C_m$ the mutation
vector capacity in elements. The simultaneous dynamic charge includes

$$
A + C_b + C_m \operatorname{sizeof}(\mathrm{Mutation}).
$$

Both vectors use charged allocators, including capacity rather than just logical
contents. Every replay iteration releases its temporary workspaces. Committed
nodes remain charged, because every accepted version is still in this memtable.
A small budget can therefore stop replay even though one frame fits. That is an
honest resource failure, not permission to skip older committed batches. Flush
and generation installation will solve that growth problem later.

## Give every end-of-file case a rule

Recovery proceeds from the verified file header and the last complete frame
boundary. It never searches forward for the next `BAT1` after damage. Doing so
could skip a required batch or mistake payload bytes for framing.

| Observation at the next frame boundary | Action |
|---|---|
| Zero header bytes, clean EOF | End the complete prefix |
| 1–23 header bytes, with every available complete field valid | Ignore this incomplete final prefix; repair back to the boundary |
| 1–23 header bytes contradicting magic or a complete field | Fail opening |
| Full 24-byte header with invalid checksum, fields, sequence, or bounds | Fail opening, even at the end |
| Valid full header, but EOF before all payload and trailer bytes | Ignore only that incomplete final frame; repair back to the boundary |
| Full frame with bad trailer CRC or invalid operations | Fail opening, even for the final frame |
| An I/O error or allocation error while doing any of this | Fail opening; do not relabel it as EOF |

For an incomplete body, existing payload bytes need not decode: the batch is
never exposed. This is a truncation policy, not a general corruption repair
algorithm. With the full body present, bad content is fatal. Correct outer CRCs
do not excuse invalid inner row schemas, duplicate keys, sequence gaps, or
trailing bytes.

The “incomplete final frame” is identified by actual EOF from `read_up_to`, not
by an exception from `read_exact` on arbitrary file positions. Header fields are
bounded before allocation; no file-controlled length can request an unbounded
vector. The complete file size is also capped through the running end offset.

## Replay into the unexposed table, then repair and synchronize

Append the complete replay function and the outer namespace closing brace:

```cpp title="src/storage/wal.cpp"
void WalMemTable::replay(std::uint64_t expected_base) {
    using namespace wal_format;
    std::array<std::byte, file_header_bytes> actual{};
    file_.read_exact(0, actual);
    require_wal(actual == file_header(expected_base), "WAL header/base mismatch");
    for (;;) {
        std::array<std::byte, batch_header_bytes> header{};
        const auto have = file_.read_up_to(end_, header);
        if (have == 0) { break; }
        require_wal(have <= max_log_bytes - end_, "WAL exceeds file cap");
        require_wal(sequence_ != std::numeric_limits<std::uint64_t>::max(),
                    "WAL sequence overflow");
        const auto next = sequence_ + 1;
        const auto prefix = std::span<const std::byte>{header}.first(have);
        check_prefix(prefix, next);
        if (have != header.size()) { repaired_tail_ = true; break; }
        const auto payload = static_cast<std::size_t>(get_le<4>(prefix.subspan(4, 4)));
        const auto count = static_cast<std::size_t>(get_le<4>(prefix.subspan(16, 4)));
        const auto frame_bytes = batch_header_bytes + payload + checksum_bytes;
        require_wal(frame_bytes <= max_log_bytes - end_, "WAL frame exceeds file cap");
        memory::Vector<std::byte> body(payload + checksum_bytes,
                                       memory::Allocator<std::byte>{budget_});
        if (file_.read_up_to(end_ + batch_header_bytes, body) != body.size()) {
            repaired_tail_ = true;
            break;
        }
        const auto data = std::span<const std::byte>{body}.first(payload);
        require_wal(get_le<4>(std::span<const std::byte>{body}.last(4)) ==
                    crc32(data, crc32(header)), "WAL batch checksum");
        auto changes = decode_payload(budget_, data, count);
        auto prepared = table_.prepare(next, changes);
        prepared.commit();
        sequence_ = next;
        end_ += frame_bytes;
    }
    // No partially recovered object has escaped the constructor.
    if (repaired_tail_) { file_.truncate_to(end_); }
    // Also synchronize a complete but previously unacknowledged last batch.
    file_.sync(sync_mode_);
}

}  // namespace strata::storage
```

At the top of each iteration, `end_` is the exclusive end of complete accepted
frames, no greater than the file cap. A zero-byte read ends normally. Nonempty
bytes after the maximum sequence are rejected before `sequence_ + 1` can wrap.
`check_prefix` examines only the bytes actually returned; the remaining
zero-initialized array bytes never stand in for unread bytes.

For a complete header, `frame_bytes <= max_log_bytes - end_` is checked before
addition. The length and count have already been checked by `check_prefix`.
The body allocation is at most 11268 bytes: the 11264-byte payload plus its
four-byte trailer. Reading a shorter body reaches EOF, so the pending frame is
not decoded or partially replayed.

For a complete body, the incremental checksum starts with `crc32(header)` and
continues over `data`. Chapter 8's `previous` parameter resumes the CRC state;
this equals one CRC over the concatenated header and payload. It excludes the
trailer itself. Only after that comparison does decoding allocate mutation
objects and validate the row list. Only after decoding does `prepare` allocate
nodes. Thus **frame validation, semantic validation, and in-memory preparation**
all precede that batch's link publication.

The local `changes` vector remains alive during preparation. The prepared nodes
copy its row bytes, so destroying that vector after commit cannot leave nodes
borrowing freed memory. Subsequent iterations can replace the body workspace
without invalidating earlier committed payloads. A later corrupt batch can make
the whole constructor fail; member cleanup then removes those earlier replayed
nodes before any object was returned.

The final two operations matter as much as parsing. Only after the entire
accepted prefix has been validated and replayed do we truncate a permitted tail.
An earlier corruption or memory failure therefore does not trigger tail removal.
Then we synchronize **even if no tail was removed**. A complete final batch may
have come from an uncertain prior write, and opening must stabilize the accepted
history before exposing it as the basis for new acknowledgements.

Truncation success followed by synchronization failure can leave a physically
shortened file while construction fails. The member owners still release their
resources; the external effect is not rolled back. Reopening must parse the
actual file again. Likewise, a backend can report a truncate error after an
external effect; the caller receives an opening error, not a claim that the old
length is intact. The deterministic backend will test a specific no-effect
truncate-error image and a shorten-then-sync-error image, not all real devices.

## Read the recovery traces in both directions

For the 149-byte partial-append image, replay accepts the 32-byte file header and
100-byte first batch. Its next read returns 17 bytes. Magic, length 72, and
sequence 2 are complete and valid; the count is incomplete. It leaves the last
complete end at 132, truncates there, synchronizes, and returns a sequence-1 owner.
A subsequent batch-2 write begins at 132, not 149, and ends at 232.

For the 232-byte image left by a failed synchronization, the second frame is
complete and valid. Replay accepts both operations, advances its sequence to 2,
finds clean EOF, synchronizes, and returns. No tail was repaired. The old writer
and the reopened owner cannot coexist in this harness: the old owner must first
release its descriptor and borrowed views.

For the same 232-byte image with one wrong final checksum byte, opening throws
`FormatError`. It must not quietly return sequence 1. For a corrupted first frame
followed by a valid second frame, opening also fails; there is no resynchronizing
search. Finally, an exact truncation to 132 produces a structurally clean
sequence-1 log. Without external evidence it cannot reveal whether a sequence-2
acknowledgement once existed. That is precisely why the prefix-preservation
assumption must be stated, not hidden in a checksum test.

At this boundary, the owner is either usable after complete opening, poisoned
after an uncertain write, or never constructed. There is no “partly recovered,
probably usable” state. Continue with the complete
[pressure, corruption, and native checks](https://25d-228.github.io/cplusplus-learning/chapter-10-part-3.html).
