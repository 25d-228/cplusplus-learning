# Chapter 11 · Part 2 — Rotation, publication, and strict reopening

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-11.html) supplied the
bounded manifest and the native directory boundary. This part connects them to
the actual table and WAL owners. Continue the same learner source tree; these
parts share Chapter 11's **5.5 implementation hours**.

The central rule is simple to state but easy to violate: **a new source must be
fully prepared and persistently named before a manifest depends on it; an old
source must remain available until authority no longer depends on it.** Preparing
an in-memory object and making a name persistent are different operations.

## A sealed log has no uncertain tail to forgive

Chapter 10 permits an incomplete final batch in an active log because an append
may have failed before acknowledgment. Rotation establishes a stronger fact: the
old generation ended at a known committed sequence, and no later writes may use
it. If its manifest says `(0,2]`, then replaying only batch 1 is corruption, even
when the file ends cleanly at that batch boundary.

Add `WalOpen::recover_sealed`, an optional expected sealed end, and a `seal`
operation. A default zero expected end preserves the earlier standalone WAL call
sites. Zero is invalid in sealed-recovery mode. Replace the **entire**
`include/strata/storage/wal.hpp` with the following two consecutive blocks. The
format constants and batch encoder signature remain unchanged.

```cpp title="include/strata/storage/wal.hpp — replace entire header"
#ifndef STRATA_STORAGE_WAL_HPP
#define STRATA_STORAGE_WAL_HPP

#include "strata/io/file.hpp"
#include "strata/memory/budget.hpp"
#include "strata/storage/memtable.hpp"

#include <cstddef>
#include <cstdint>
#include <span>

namespace strata::storage {

namespace wal_format {
inline constexpr std::size_t file_header_bytes = 32;
inline constexpr std::size_t batch_header_bytes = 24;
inline constexpr std::size_t operation_header_bytes = 14;
inline constexpr std::size_t checksum_bytes = 4;
inline constexpr std::size_t max_payload_bytes =
    MemTable::max_batch * (operation_header_bytes + event_bytes);
inline constexpr std::uint64_t max_log_bytes = 64 * 1024 * 1024;

memory::Vector<std::byte> encode_batch(memory::Budget& budget,
    std::uint64_t sequence, std::span<const Mutation> changes);
}  // namespace wal_format

enum class WalOpen { create, recover, recover_sealed };
enum class WriteOutcome { committed, rejected, uncertain };
enum class WriteReason {
    none, invalid_batch, resource_exhausted, log_full, sequence_exhausted,
    writer_poisoned, writer_sealed, io_failure, unexpected_failure
};

struct WriteResult {
    WriteOutcome outcome;
    WriteReason reason;
    std::uint64_t sequence;  // Zero for rejection; candidate for uncertainty.
    int io_error = 0;
};
```

`writer_sealed` is a definite rejection of a new request, not an uncertain outcome
for the old committed batches. A frozen memtable is no longer an insertion
target. The new result makes that state explicit rather than allowing the
internal `prepare` call to throw an unclassified logic exception.

```cpp title="include/strata/storage/wal.hpp — replace entire header"
class WalMemTable {
public:
    WalMemTable(memory::Budget& budget, io::File file, std::size_t arena_cap,
                WalOpen open_mode, io::SyncMode sync_mode,
                std::uint64_t base_sequence = 0, std::uint64_t sealed_end = 0);
    WalMemTable(const WalMemTable&) = delete;
    WalMemTable& operator=(const WalMemTable&) = delete;
    WalMemTable(WalMemTable&&) = delete;
    WalMemTable& operator=(WalMemTable&&) = delete;

    [[nodiscard]] WriteResult write(std::span<const Mutation> changes) &;
    void seal() &;
    const MemTable& table() const & noexcept { return table_; }
    const MemTable& table() const && = delete;
    std::uint64_t sequence() const noexcept { return sequence_; }
    std::uint64_t log_end() const noexcept { return end_; }
    bool poisoned() const noexcept { return poisoned_; }
    bool repaired_tail() const noexcept { return repaired_tail_; }

private:
    void initialize(std::uint64_t base);
    void replay(std::uint64_t expected_base, std::uint64_t required_end = 0);
    memory::Budget& budget_;
    io::File file_;
    MemTable table_;
    io::SyncMode sync_mode_;
    std::uint64_t sequence_;
    std::uint64_t end_ = wal_format::file_header_bytes;
    bool poisoned_ = false;
    bool repaired_tail_ = false;
};

}  // namespace strata::storage
#endif
```

In `src/storage/wal.cpp`, replace the constructor with this definition. Keep all
existing format helpers and `initialize`. The constructor validates the
relationship between open mode, base, and expected end before invoking replay.

```cpp title="src/storage/wal.cpp — replace constructor"
WalMemTable::WalMemTable(memory::Budget& budget, io::File file,
    std::size_t arena_cap, WalOpen open_mode, io::SyncMode sync_mode,
    std::uint64_t base_sequence, std::uint64_t sealed_end)
    : budget_(budget), file_(std::move(file)), table_(budget, arena_cap),
      sync_mode_(sync_mode), sequence_(base_sequence) {
    if (!file_.is_open() || (sync_mode != io::SyncMode::file &&
        sync_mode != io::SyncMode::full_device)) {
        throw std::invalid_argument("invalid WAL file or sync mode");
    }
    if ((open_mode == WalOpen::recover_sealed && sealed_end <= base_sequence) ||
        (open_mode != WalOpen::recover_sealed && sealed_end != 0)) {
        throw std::invalid_argument("invalid sealed WAL boundary");
    }
    switch (open_mode) {
    case WalOpen::create: initialize(base_sequence); break;
    case WalOpen::recover: replay(base_sequence); break;
    case WalOpen::recover_sealed: replay(base_sequence, sealed_end); seal(); break;
    default: throw std::invalid_argument("invalid WAL open mode");
    }
}
```

Add the following function in `strata::storage`. `MemTable::freeze` already
refuses an outstanding preparation and marks the table immutable. This wrapper
also refuses to seal an uncertain writer: a log containing an unacknowledged
complete batch must first resolve its state by reopening, not by declaring the
old visible sequence to be its final end.

```cpp title="src/storage/wal.cpp — add seal"
void WalMemTable::seal() & {
    if (poisoned_) { throw std::logic_error("cannot seal an uncertain WAL"); }
    table_.freeze();
}
```

Replace `WalMemTable::write` with the complete function below. Its only new branch
is the frozen-table rejection before encoding or I/O. The prepare–append–sync–
publish protocol and its existing exception classification are retained.

```cpp title="src/storage/wal.cpp — replace write"
WriteResult WalMemTable::write(std::span<const Mutation> changes) & {
    if (poisoned_) {
        return {WriteOutcome::rejected, WriteReason::writer_poisoned, 0};
    }
    if (table_.frozen()) {
        return {WriteOutcome::rejected, WriteReason::writer_sealed, 0};
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

Replace `replay` with the following complete definition. The new check is after
all complete batches have been replayed into the still-unexposed owner, but
**before** truncation. This ordering matters: failing a sealed-log open must not
first “repair” the evidence that caused failure.

```cpp title="src/storage/wal.cpp — replace replay"
void WalMemTable::replay(std::uint64_t expected_base, std::uint64_t required_end) {
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
    if (required_end != 0) {
        require_wal(!repaired_tail_ && sequence_ == required_end,
                    "sealed WAL does not match manifest end");
    }
    if (repaired_tail_) { file_.truncate_to(end_); }
    // Also synchronize a complete but previously unacknowledged last batch.
    file_.sync(sync_mode_);
}
```

For a required end of 2, a valid first batch followed by EOF leaves
`sequence_ == 1` and fails the new check. A complete first batch plus a partial
second batch sets `repaired_tail_` and also fails, without calling `truncate_to`.
A file containing a complete unexpected third batch reaches sequence 3 and fails.
Only exact replay through 2 succeeds and is then frozen by the constructor.

For active-log recovery, `required_end == 0`: the existing permitted-tail policy
still truncates an incomplete last frame and synchronizes the recovered file.
The distinction is supplied by authoritative metadata, not guessed from the
filename or the physical length. Complete malformed frames remain errors in
both modes.

These are all prerequisite changes in this chapter: native descriptor adoption
in Part 1, the replacement WAL header, and these four WAL definitions. Preserve
all other existing files and their charged allocation contracts.

## State the persistence assumptions before the install code

The native path requests file synchronization through Chapter 6's `File::sync`,
uses same-directory rename for authority replacement, and requests directory
synchronization through Part 1's `Directory::sync`. The protocol's recovery
argument is conditional on a storage stack that honors the requested ordering:
previously stabilized bytes and names survive, a later failed write does not
modify unrelated stabilized files, and a crash during authority replacement
leaves the old or new complete binding rather than a fabricated mixed manifest.
Corrupt or missing authority is detected and refused, not silently reconstructed.

Those assumptions are stronger than the mere existence of a function named
`rename`. The chapter does not infer them from a filesystem marketing label or
from a passing process-exit test. A root directory lost from its parent, a device
that lies about flushes, or a filesystem that does not support directory sync is
outside the successful durability argument.

Apple's [fsync description](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
and [F_FULLFSYNC interface](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fcntl.2.html)
distinguish ordinary file synchronization from requesting a drive-cache flush.
Our `full_device` mode is never silently downgraded. After the final directory
sync, it requests another full-device synchronization through the still-open
manifest file. This asks for a device flush after the namespace writes have been
issued; it is not an experimentally established power-loss guarantee. Inspect
the installed SDK manuals and actual filesystem behavior on the target machine;
the archived manuals are not a declaration of its macOS version or supported
filesystem list.

Let $D(X)$ mean successful requested synchronization of file contents $X$;
$N$ mean successful directory synchronization; $R$ mean replacing the authority
name; and $P$ mean publishing the prepared serving owners in memory. The install
order is

$$
D(\text{new dependencies})\ \to\ N\ \to\
D(\text{candidate manifest})\ \to\ R\ \to\ N\ \to\
D(\text{manifest, final barrier})\ \to\ P.
$$

The arrows describe ordering in our program under the stated persistence model,
not a claim that the operating system implements a multi-file transaction for us.
The old dependencies remain present throughout. The final file synchronization
is redundant in some ordinary-file cases but keeps the stronger requested mode's
post-directory barrier explicit.

## Prepare owners, not merely filenames

A `Database` owns a bounded array of WAL owners and a bounded array of table
readers. The heap owners are allocated through `std::allocate_shared` with the
existing charged allocator. Moving a pointer between array slots does not move
its memtable or invalidate that memtable's internal addresses.

This is not yet a snapshot API. The shared pointers stay private, and all access
is synchronous. Their use here separates stable object identity from array-slot
identity and allows fallible construction before catalog publication. The budget
must outlive the database and every charged owner; the database cannot be moved
or copied. Its `get` result owns its row bytes and can outlive the database.

Create the header in two blocks:

```cpp title="include/strata/storage/database.hpp"
#ifndef STRATA_STORAGE_DATABASE_HPP
#define STRATA_STORAGE_DATABASE_HPP

#include "strata/io/directory.hpp"
#include "strata/storage/manifest.hpp"
#include "strata/storage/wal.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <memory>
#include <optional>
#include <span>
#include <string>
#include <string_view>

namespace strata::storage {

enum class DatabaseOpen { create, recover };
struct DatabaseOptions {
    std::size_t arena_bytes = 1024 * 1024;
    std::size_t cache_slots = 64;
    std::size_t cache_bytes = 512 * 1024;
    TableOptions table{};
    io::SyncMode sync_mode = io::SyncMode::file;
};
struct InstallHook {
    using Function = void (*)(void*, std::string_view);
    Function function = nullptr;
    void* context = nullptr;
    void operator()(std::string_view stage) const {
        if (function != nullptr) { function(context, stage); }
    }
};
```

`InstallHook::Function` is a pointer to a free function taking a context pointer
and a borrowed stage name. `void (*)(...)` declares the pointer's calling
signature; it is not a pointer to a C++ member function. The test context owns any
state. Its callback casts the pointer back to the correct type. This avoids an
allocating callback wrapper at a boundary where allocation timing is under test.

A null function makes the hook a no-op. A supplied hook must outlive the database,
must use the matching context type, and must not reenter database methods. Stage
names are string literals, but the callback API still promises only a synchronous
borrow. Part 3 supplies the complete callback; no hidden injection framework is
required.

`DatabaseOptions` bounds an individual arena separately from the shared budget.
The cache has both a slot and a byte bound. Table options remain the existing
builder options; an impossible output configuration can fail maintenance, not
magically enlarge its file or allocation caps.

```cpp title="include/strata/storage/database.hpp"
class Database {
public:
    Database(memory::Budget& budget, const std::string& directory,
             DatabaseOpen mode, DatabaseOptions options = {}, InstallHook hook = {});
    Database(const Database&) = delete;
    Database& operator=(const Database&) = delete;
    Database(Database&&) = delete;
    Database& operator=(Database&&) = delete;
    [[nodiscard]] WriteResult write(std::span<const Mutation> changes) &;
    std::optional<OwnedEntry> get(Key key) &;
    bool rotate() &;
    bool flush_oldest() &;
    std::uint64_t sequence() const noexcept;
    bool poisoned() const noexcept { return poisoned_; }
    const Manifest& manifest() const & noexcept { return manifest_; }
    const Manifest& manifest() const && = delete;
    std::size_t orphans_at_open() const noexcept { return orphans_at_open_; }
private:
    void require_healthy() const;
    std::uint64_t take_id();
    Manifest next_manifest() const;
    void install(Manifest candidate);
    void create();
    void recover();
    std::shared_ptr<WalMemTable> open_log(LogRecord record, WalOpen mode);
    std::shared_ptr<TableReader> open_table(TableRecord record);
    memory::Budget& budget_;
    io::Directory directory_;
    DatabaseOptions options_;
    InstallHook hook_;
    BlockCache cache_;
    Manifest manifest_{};
    std::array<std::shared_ptr<WalMemTable>, Manifest::max_logs> logs_{};
    std::array<std::shared_ptr<TableReader>, Manifest::max_tables> tables_{};
    std::uint64_t next_id_ = 1;
    std::size_t orphans_at_open_ = 0;
    bool poisoned_ = false;
};

}  // namespace strata::storage
#endif
```

Notice the declaration order: `directory_`, then cache, then manifest and data
owners. On destruction, table/WAL owners disappear before cache and directory.
The lock remains held while those child descriptors close. `budget_` is only a
reference; the caller is responsible for its lifetime.

`manifest()` exposes diagnostic metadata, not an editable catalog. `sequence()`
reports the last locally published batch. After poisoning, these diagnostic
values may describe the pre-install serving state even when a new manifest name
is already visible on disk. Data reads through `get` refuse a poisoned owner.
The only permitted next lifecycle step is destruction and authoritative reopen.

### Copy a result before its source can go away

Create `src/storage/database.cpp`. The `copy_entry` helper is the complete bridge
from an arena-borrowed row to the small owning result already introduced by the
table reader. It neither stores a span nor allocates another record payload.

```cpp title="src/storage/database.cpp"
#include "strata/storage/database.hpp"
#include "strata/core/codec.hpp"

#include <algorithm>
#include <limits>
#include <stdexcept>
#include <type_traits>
#include <utility>

namespace strata::storage {
namespace {
OwnedEntry copy_entry(EntryView entry) {
    OwnedEntry result;
    result.key = entry.key;
    result.kind = entry.kind;
    result.length = entry.value.size();
    if (result.length > result.bytes.size()) { throw FormatError("oversized owned row"); }
    std::copy(entry.value.begin(), entry.value.end(), result.bytes.begin());
    return result;
}
}  // namespace
```

`EntryView::value` may point into a memtable arena. Copying `entry.key` and
`entry.kind` is not enough: the payload itself must be copied before the source
can be retired. `OwnedEntry` contains room for the largest supported row, and
`length` identifies its active byte prefix. The capacity check makes an invalid
internal view an error rather than an unchecked copy.

### Refuse ambiguous creation; choose the next identity monotonically

```cpp title="src/storage/database.cpp"
Database::Database(memory::Budget& budget, const std::string& directory,
                   DatabaseOpen mode, DatabaseOptions options, InstallHook hook)
    : budget_(budget), directory_(directory), options_(options), hook_(hook),
      cache_(budget, options.cache_slots, options.cache_bytes) {
    if (options.arena_bytes < memory::Arena::chunk_charge() ||
        (options.sync_mode != io::SyncMode::file && options.sync_mode != io::SyncMode::full_device)) {
        throw std::invalid_argument("invalid database options");
    }
    const auto inventory = directory_.inventory();
    if (inventory.largest_id == std::numeric_limits<std::uint64_t>::max()) {
        throw std::overflow_error("observed file identity exhausts namespace");
    }
    next_id_ = inventory.largest_id + 1;
    switch (mode) {
    case DatabaseOpen::create:
        if (inventory.manifest || inventory.numbered_files != 0) {
            throw FormatError("create requires no existing engine files");
        }
        create();
        break;
    case DatabaseOpen::recover:
        if (!inventory.manifest) { throw FormatError("authoritative MANIFEST missing"); }
        recover();
        orphans_at_open_ = inventory.numbered_files - manifest_.table_count - manifest_.log_count;
        break;
    default: throw std::invalid_argument("invalid database open mode");
    }
}
```

The directory constructor has already acquired the lock and scanned names before
this body runs. Cache allocation failure unwinds that directory owner. Recovery
sets the file-id lower bound from both sources of information:

$$
\text{next identity}=\max\bigl(\text{manifest.next\_id},\
1+\text{largest observed numbered identity}\bigr).
$$

The increment is checked before use; observing the maximum unsigned identity
fails opening instead of wrapping to zero. A corrupt orphan is still an observed
identity: its contents are not needed to avoid name reuse. If an interrupted
rotation created log 3 while authority still says `next_id == 3`, scanning that
file forces the next allocation above 3.

The current implementation preserves all orphans. Deleting an unreferenced
high-id file before persisting a corresponding high-water mark would break this
argument on the next reopen. Do not add an eager cleanup loop. Later reclamation
must first stabilize authority that remembers every observed identity it removes.

`orphans_at_open` is a count from this opening, not a continuously updated live
file count. Missing referenced files make recovery throw before that count is
reported. The manifest's unique referenced IDs and successful required opens
justify subtracting its reference counts from the scanned numbered-file count.

```cpp title="src/storage/database.cpp"
void Database::require_healthy() const {
    if (poisoned_) { throw std::logic_error("database requires reopen"); }
}
std::uint64_t Database::take_id() {
    if (next_id_ == std::numeric_limits<std::uint64_t>::max()) {
        throw std::overflow_error("file identities exhausted");
    }
    return next_id_++;
}
Manifest Database::next_manifest() const {
    if (manifest_.epoch == std::numeric_limits<std::uint64_t>::max()) {
        throw std::overflow_error("manifest epochs exhausted");
    }
    auto result = manifest_;
    ++result.epoch;
    return result;
}
```

Neither identity nor epoch arithmetic is allowed to wrap. `take_id` consumes an
identity in memory even if later preparation fails; gaps are acceptable. A
successful manifest remembers the advanced counter. After an unsuccessful
installation, retained numbered outputs preserve the additional evidence for
the next process. No correctness property requires consecutive filenames.

### Recover a log using the base supplied by authority

```cpp title="src/storage/database.cpp"
std::shared_ptr<WalMemTable> Database::open_log(LogRecord record, WalOpen mode) {
    const auto name = io::file_name(record.id, io::FileKind::log);
    auto opened = directory_.open(name.data(), mode == WalOpen::create
        ? io::OpenMode::create_new : io::OpenMode::read_write);
    if (opened.bytes > wal_format::max_log_bytes) { throw FormatError("WAL file cap"); }
    return std::allocate_shared<WalMemTable>(memory::Allocator<WalMemTable>{budget_},
        budget_, std::move(opened.file), options_.arena_bytes, mode, options_.sync_mode,
        record.base, mode == WalOpen::recover_sealed ? record.end : 0);
}
```

A new log uses exclusive creation. An existing log must already exist and be
writable for permitted active-tail repair and synchronization. The actual opened
length is bounded before replay. The constructor checks the file header against
`record.base`; a valid WAL copied from a different interval is not accepted.

The [allocate_shared interface](https://timsong-cpp.github.io/cppwp/n4861/util.smartptr.shared.create)
constructs the owner directly in allocated storage and uses the supplied allocator
for its allocation bookkeeping. That keeps the object and control storage within
the budget. Constructor failure destroys already constructed subobjects and
returns the allocation. It does **not** undo native file creation or bytes the
constructor wrote before failing.

### Verify table contents before they become serving state

```cpp title="src/storage/database.cpp"
std::shared_ptr<TableReader> Database::open_table(TableRecord record) {
    const auto name = io::file_name(record.id, io::FileKind::table);
    auto opened = directory_.open(name.data(), io::OpenMode::read_only);
    if (opened.bytes != record.bytes) { throw FormatError("table length differs from manifest"); }
    // Durable IDs name files. Cache IDs name bindings inside this cache instance.
    const auto cache_id = cache_.new_file_id();
    auto reader = std::allocate_shared<TableReader>(memory::Allocator<TableReader>{budget_},
        budget_, std::move(opened.file), opened.bytes, &cache_, cache_id);
    reader->verify_all();
    std::uint64_t smallest = std::numeric_limits<std::uint64_t>::max();
    std::uint64_t largest = 0;
    auto cursor = reader->begin();
    while (cursor.valid()) {
        const auto sequence = cursor.entry().key.sequence;
        if (sequence <= record.base || sequence > record.end) {
            throw FormatError("table sequence outside manifest interval");
        }
        smallest = std::min(smallest, sequence);
        largest = std::max(largest, sequence);
        cursor.next();
    }
    if (record.end <= record.base || smallest != record.base + 1 || largest != record.end) {
        throw FormatError("table interval endpoints missing");
    }
    return reader;
}
```

There are two identity domains. The manifest's `record.id` is a persistent file
incarnation within this database directory. `cache_.new_file_id()` creates a
binding within this cache instance. Passing a durable identity directly to a
cache that never issued it would violate Chapter 9's API. Each reader gets one
fresh local binding for one immutable file and keeps it for its lifetime. A new
database has a new cache; it does not reuse previous process entries.

`verify_all` rereads data independently of the cache. The following cursor pass
checks the relationship between otherwise valid table bytes and the manifest's
sequence interval. Every version must lie in $(b,e]$, and both the first and last
batch sequences must occur. Tombstones count as entries. The current flush keeps
all versions, so the first batch cannot legitimately disappear. This does not
claim authentication or detection of every cleverly rewritten valid file.

The two passes cost extra reads. Their buffers remain bounded; no full-table
vector is built. Combining both validations in one uncached cursor is a possible
later interface improvement, not a reason to skip either check here. A constructor
or validation failure leaves no new reader published.

## Install authority, then publish with nonthrowing assignments

The installer receives a complete candidate value. It first reserves a unique
temporary filename, includes the resulting high-water mark in the candidate,
and encodes it in charged storage. All dependency files have already been fully
written and synchronized by the caller. The directory sync before candidate
creation establishes their names under the stated persistence model.

```cpp title="src/storage/database.cpp"
void Database::install(Manifest candidate) {
    static_assert(std::is_nothrow_copy_assignable_v<Manifest>);
    const auto temporary = io::file_name(take_id(), io::FileKind::temporary);
    candidate.next_id = next_id_;
    auto image = encode_manifest(budget_, candidate);
    directory_.sync();
    hook_("dependencies-synced");
    auto opened = directory_.open(temporary.data(), io::OpenMode::create_new);
    hook_("manifest-created");
    opened.file.write_all(0, image);
    hook_("manifest-written");
    opened.file.sync(options_.sync_mode);
    hook_("manifest-synced");
    hook_("before-rename");
    directory_.replace_manifest(temporary.data());
    hook_("manifest-renamed");
    hook_("before-directory-sync");
    directory_.sync();
    hook_("directory-synced");
    // In full-device mode, issue a device flush after directory fsync has completed.
    opened.file.sync(options_.sync_mode);
    hook_("manifest-stabilized");
    opened.file.close_checked();
    hook_("manifest-closed");
    manifest_ = candidate;  // The remaining caller-side owner swaps cannot allocate.
}
```

The still-open descriptor continues to refer to the candidate file after rename;
it does not reopen the spelling of `MANIFEST`. After the final directory and file
synchronizations, explicit close reports any close error before in-memory
publication. A close failure cannot prove that the install did not take effect,
so maintenance remains stopped and reopening decides authority.

The test hook runs only before the final assignment. Even a hook exception at
`manifest-closed` leaves the previous in-memory catalog unmodified. After the
assignment, callers perform only bounded nonthrowing smart-pointer moves and
counter updates. The C++20 [shared-pointer assignments](https://timsong-cpp.github.io/cppwp/n4861/util.smartptr.shared.assign)
are nonthrowing; no file open, reader construction, logging string, or vector
resize is allowed after that boundary.

A test hook is not a simulated storage device. Throwing at `before-directory-sync`
means the synchronization was not attempted. Exiting after `manifest-renamed`
means the live kernel has performed the rename but not necessarily made it
power-loss durable. The tests must label these facts accurately.

| Interruption | Authoritative recovery choices under the stated model | Still required |
|---|---|---|
| New output incomplete or its name not stabilized | Old manifest | All old required files |
| Dependency names stabilized, candidate absent or partial | Old manifest | All old required files |
| Candidate synchronized, rename not performed | Old manifest | All old required files |
| Rename performed, final barriers unfinished | Old or new complete manifest | Both dependency sets remain available |
| Final barriers and publication completed | New manifest | Its referenced files |

The implementation does not delete files in any row. That simplifies the
transition proof while leaving disk reclamation explicit unfinished maintenance,
not an invisible promise of bounded disk use.

### Initialize a new database through the same installer

```cpp title="src/storage/database.cpp"
void Database::create() {
    Manifest initial;
    initial.epoch = 1;
    initial.log_count = 1;
    initial.logs[0] = {take_id(), 0, 0};
    auto active = open_log(initial.logs[0], WalOpen::create);
    hook_("log-ready");
    install(initial);
    logs_[0] = std::move(active);
}
```

The initial active owner is prepared before installation. It writes and
synchronizes a base-zero WAL header; the installer synchronizes its directory
name before authorizing it. After authority is installed, assigning the prepared
pointer cannot fail. Only then does the constructor return a usable database.

A failed first creation may leave a WAL or temporary file without `MANIFEST`.
The object does not escape, and all descriptors and charged owners unwind.
Subsequent automatic creation refuses those files. The absence of an acknowledged
database is not permission to erase a path that may contain work someone needs
to inspect.

## Reopen only the named sources, in their declared roles

```cpp title="src/storage/database.cpp"
void Database::recover() {
    auto authority = directory_.open("MANIFEST", io::OpenMode::read_write);
    if (authority.bytes < Manifest::header_bytes + 24 + 4 || authority.bytes > Manifest::max_bytes) {
        throw FormatError("MANIFEST outside size bounds");
    }
    memory::Vector<std::byte> image(static_cast<std::size_t>(authority.bytes),
                                     memory::Allocator<std::byte>{budget_});
    authority.file.read_exact(0, image);
    manifest_ = decode_manifest(image);
    next_id_ = std::max(next_id_, manifest_.next_id);
    for (std::size_t i = 0; i < manifest_.table_count; ++i) {
        tables_[i] = open_table(manifest_.tables[i]);
    }
    for (std::size_t i = 0; i < manifest_.log_count; ++i) {
        logs_[i] = open_log(manifest_.logs[i], i + 1 == manifest_.log_count
            ? WalOpen::recover : WalOpen::recover_sealed);
    }
    // Stabilize the observed authority before allowing writes depending on it.
    directory_.sync();
    authority.file.sync(options_.sync_mode);
    authority.file.close_checked();
}
```

The `MANIFEST` descriptor is opened for read/write because recovery must request
synchronization; the code never overwrites its bytes. Its actual length is
bounded before allocating the image. Decoding establishes coverage before any
source is accepted.

Tables are opened and fully checked first. Required sealed logs then replay with
their exact ends, followed by the active log with the permitted-tail policy.
Every owner remains inside an unfinished database until all these steps and the
final authority synchronization succeed. If the last source fails, earlier
owners are destroyed; no partially recovered database is returned. This follows
the language's [constructor cleanup rules](https://timsong-cpp.github.io/cppwp/n4861/except.ctor),
not a catch block that tries to manually guess which members were initialized.

A manifest observed after an interrupted rename may be newer than the previous
caller's acknowledged state. Stabilizing that observed authority before returning
prevents later acknowledged writes from depending on an uncompleted installation
barrier. An active WAL may already have been shortened when a later sync fails;
opening still fails, and the next attempt must validate the actual files again.

There is intentionally no fallback to an orphan log when an installed table is
missing. Such fallback could omit data, reintroduce a deleted row, or invent an
older catalog. Report the missing/corrupt dependency and preserve the evidence.

## Current reads must respect newer tombstones

```cpp title="src/storage/database.cpp"
std::uint64_t Database::sequence() const noexcept {
    return logs_[manifest_.log_count - 1]->sequence();
}
WriteResult Database::write(std::span<const Mutation> changes) & {
    if (poisoned_) { return {WriteOutcome::rejected, WriteReason::writer_poisoned, 0}; }
    auto result = logs_[manifest_.log_count - 1]->write(changes);
    if (result.outcome == WriteOutcome::uncertain) { poisoned_ = true; }
    return result;
}
```

`write` retains Chapter 10's three outcomes. Definite validation/resource rejection
leaves the owner usable; an uncertain WAL result poisons the whole database.
A maintenance exception after entering its effectful region also leaves the
poison flag set. This deliberately conservative policy avoids using a stale
active-log choice after a potentially installed manifest.

The point reader searches required logs newest first, then installed tables
newest first. Contiguous nonoverlapping generation intervals justify choosing
the first matching source. Within that source the existing memtable/table lookup
chooses its newest visible version.

```cpp title="src/storage/database.cpp"
std::optional<OwnedEntry> Database::get(Key key) & {
    require_healthy();
    if (key.relation != Relation::events && key.relation != Relation::accounts) {
        throw std::invalid_argument("unknown key relation");
    }
    for (std::size_t i = manifest_.log_count; i > 0; --i) {
        if (const auto entry = logs_[i - 1]->table().find(key)) {
            if (entry->kind == EntryKind::erase) { return std::nullopt; }
            return copy_entry(*entry);
        }
    }
    for (std::size_t i = manifest_.table_count; i > 0; --i) {
        if (auto entry = tables_[i - 1]->find(key)) {
            if (entry->kind == EntryKind::erase) { return std::nullopt; }
            return entry;
        }
    }
    return std::nullopt;
}
```

A tombstone returns absence immediately. Continuing to an older source would
resurrect the previous value. In the fixture, table 5 eventually contains event
100 at sequence 2, while a later table contains its sequence-3 tombstone. Reading
all disk tables in reverse generation order must return absence, not the older
700-cent event.

The result is an owning value and the read is current-state only. This is not a
repeatable snapshot across multiple calls: a later write may change the answer.
Do not retain an internal cursor or invoke these methods concurrently. The next
chapter will capture immutable owner sets and implement merged scans.

## Rotation changes the recovery dependency before accepting new writes

Rotation is a no-op when the active memtable is empty. Otherwise it requires a
spare log slot, records the old active end, and prepares a fresh active owner with
that base. Slot and epoch rejection happen before entering the effectful region.

```cpp title="src/storage/database.cpp"
bool Database::rotate() & {
    require_healthy();
    const auto count = manifest_.log_count;
    if (logs_[count - 1]->table().size() == 0) { return false; }
    if (count == Manifest::max_logs) { throw std::length_error("sealed-log slots exhausted"); }
    auto candidate = next_manifest();
    candidate.logs[count - 1].end = sequence();
    candidate.logs[count] = {take_id(), sequence(), 0};
    ++candidate.log_count;
    poisoned_ = true;  // All later failures require reopening, even before rename.
    auto active = open_log(candidate.logs[count], WalOpen::create);
    hook_("log-ready");
    logs_[count - 1]->seal();
    install(candidate);
    logs_[count] = std::move(active);
    poisoned_ = false;
    return true;
}
```

`poisoned_` becomes true before any output file is created. If later allocation,
open, sync, sealing, manifest encoding, or installation throws, the flag remains
true. The caller cannot append to an active log whose identity may now disagree
with disk authority. Even failures we might individually prove harmless use this
single conservative maintenance policy; recovery restores a coherent state.

The old owner is frozen before installation. A failed installation may therefore
leave it frozen in memory while the old disk manifest still calls it active.
That is acceptable only because the failed owner refuses further work. Reopening
constructs a fresh mutable owner when the old manifest remains authoritative.

After successful installation, the prepared active pointer is moved into its
slot and the poison flag is cleared. No new batch can be acknowledged on this
log before that point. `rotate` itself acknowledges a metadata transition, not
a new data batch; `sequence()` is unchanged.

The four-log cap is not a loss policy. When it is reached, `rotate` throws a
no-change admission error before poisoning. Flush the oldest generation to free
a slot, then retry. The active writer can also reject a new batch because its
arena or WAL is full; the caller must inspect that definite result and perform
maintenance rather than silently treating it as a commit.

## Flush exactly the oldest sealed generation

The source at `logs_[0]` is the only generation this operation may install.
Later sealed logs remain required until the prefix before them is installed.
The active log is never flushed directly; rotate it first. The output file keeps
all internal versions, not merely the latest value of each key.

```cpp title="src/storage/database.cpp"
bool Database::flush_oldest() & {
    require_healthy();
    if (manifest_.log_count == 1) { return false; }
    if (manifest_.table_count == Manifest::max_tables) { throw std::length_error("table slots exhausted"); }
    auto candidate = next_manifest();
    const auto sealed = manifest_.logs[0];
    const auto table_id = take_id();
    const auto name = io::file_name(table_id, io::FileKind::table);
    poisoned_ = true;
    auto output = directory_.open(name.data(), io::OpenMode::create_new);
    const auto info = write_table(budget_, output.file, logs_[0]->table(), options_.table);
    output.file.sync(options_.sync_mode);
    output.file.close_checked();
    hook_("table-ready");
    const TableRecord record{table_id, sealed.base, sealed.end, info.file_bytes};
    auto reader = open_table(record);  // Complete all fallible serving-state preparation.
    candidate.tables[candidate.table_count++] = record;
    candidate.flushed = sealed.end;
    for (std::size_t i = 1; i < candidate.log_count; ++i) { candidate.logs[i - 1] = candidate.logs[i]; }
    candidate.logs[--candidate.log_count] = {};
    install(candidate);
    tables_[manifest_.table_count - 1] = std::move(reader);
    for (std::size_t i = 0; i < manifest_.log_count; ++i) { logs_[i] = std::move(logs_[i + 1]); }
    logs_[manifest_.log_count].reset();
    poisoned_ = false;
    return true;
}

}  // namespace strata::storage
```

The output table is synchronized and explicitly closed before it is reopened as
a reader. Its format, file length, interval membership, and serving metadata are
fully prepared before the manifest changes. `write_table` streams from the
already frozen skip list; there is no additional full-table sort or collection.

The candidate appends the table, advances `flushed`, and removes the first log
record. Only after installation does the actual owner array shift left. For two
logs, moving slot 1 into slot 0 releases the old sealed owner and transfers the
active owner; the vacated slot is empty. No bytes inside either memtable move.
For more logs, the increasing-index loop never overwrites an owner it still
needs to move: each right-hand slot is consumed once.

The old WAL filename is intentionally retained. It is no longer required by the
new manifest, but disk deletion is separate from releasing its in-memory owner.
Future snapshots may also retain owners after a newer serving set stops using
them. Retiring a dependency and reclaiming a file must not be collapsed into one
unconditional destructor action.

There is a temporary memory cost. If $A$ denotes retained arena charges, $C$ cache
and reader charges, $W$ builder workspaces, and $M$ the candidate manifest image,
the flush must admit the overlap

$$
A+C+W+M\ \leq\ B,
$$

where $B$ is the shared allocation budget. Reader construction and verification
can also coexist with the old source; the actual ledger charges their allocations,
including shared ownership bookkeeping. Root-object inline arrays, bounded stack
objects, library error strings, OS state, and allocator overhead are not an RSS
cap. Releasing a source early to make an otherwise impossible allocation succeed
would destroy the failure guarantee.

At sixteen installed tables, further flushes reject before beginning I/O. This
chapter deliberately stops rather than overwriting an old run. Compaction will
change the serving catalog and reclaim obsolete sources; the current format
policy is one complete table per flushed generation, not a finished multi-file
base-run representation.

The synchronous database is now complete enough to create, write, rotate, flush,
close, and reopen. [Part 3](https://25d-228.github.io/cplusplus-learning/chapter-11-part-3.html)
checks those operations against independent manifest decoding, deliberate
corruption, resource pressure, a second process, and interruption at each install
boundary.
