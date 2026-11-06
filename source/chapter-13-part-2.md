# Chapter 13 — Streaming compaction and safe reclamation

## Part 2 — Install new owners, then reclaim old names

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-13.html) established the
complete-prefix rule and output forecast. This part implements the two distinct
operations. `compact_prefix` creates and installs replacement tables. Only a later
`collect_obsolete` considers deletion. An exception during the first operation
must not turn its speculative outputs into authority; an error during the second
must not turn current input files into garbage.

All code below extends the completed Part 1 state. The allocation remains **4.0
implementation hours for the whole chapter**, not four hours per part. The
implementation and test hooks remain synchronous and nonreentrant. In particular,
a hook must not call back into the same database, capture views, or run collection.

## A source needs both object identity and persistent identity

Chapter 12's retention slots count distinct source **objects** using stable owner
addresses. That is sufficient for admission accounting, but a directory listing
contains persistent file IDs and suffixes, not `TableReader*` addresses. Record
both identities in the same bounded registry. Do not add a second owning registry
that accidentally keeps every retired source alive forever.

A table source records its `.sst` identity. A frozen memtable source records its
`.wal` identity, even though reads come from its arena. Preserving that WAL name
while its generation is retained gives one conservative policy for both source
kinds. Every source admitted by `Database::snapshot` must have a nonzero ID.

Apply these declaration changes to `read_view.hpp`. `file_id` is plain metadata;
the shared pointers remain the owning edges. `Slot::references` still counts
admitted states, not copies of a `ReadView` handle.

```diff title="include/strata/storage/read_view.hpp — exact declaration changes"
--- a/include/strata/storage/read_view.hpp
+++ b/include/strata/storage/read_view.hpp
@@ -32,6 +32,7 @@
 inline constexpr std::size_t max_retained = 128;
 
 struct Source {
+    std::uint64_t file_id = 0;
     std::shared_ptr<const WalMemTable> memory;
     std::shared_ptr<TableReader> disk;
     const void* identity() const noexcept {
@@ -48,11 +49,17 @@
     void retain(std::span<const Source> sources) noexcept;
     void release(std::span<const Source> sources) noexcept;
     ReadRetention retention() const noexcept { return counts_; }
+    bool retains_file(std::uint64_t id, io::FileKind kind) const noexcept;
     std::shared_ptr<memory::Budget> budget;
     io::Directory directory;
     BlockCache cache;
 private:
-    struct Slot { const void* identity = nullptr; std::size_t references = 0; };
+    struct Slot {
+        const void* identity = nullptr;
+        std::size_t references = 0;
+        std::uint64_t file_id = 0;
+        io::FileKind kind = io::FileKind::log;
+    };
     std::size_t slot_for(const void* identity) const noexcept;
     ReadLimits limits_;
     ReadRetention counts_{};
```

Inside `namespace read_detail` in `src/storage/read_view.cpp`, replace the complete
`Context::check_admission` and `Context::retain` definitions. The former checks the
new invariant before any rotation. The latter stores persistent identity only
when a previously unseen owner acquires a slot. Existing `release` clears the
whole slot with `{}` at its final reference, including the added fields.

```cpp title="src/storage/read_view.cpp — replace void Context::check_admission"
void Context::check_admission(std::span<const Source> sources) const {
    if (sources.size() > max_sources) { throw std::length_error("too many view sources"); }
    if (counts_.snapshots == limits_.snapshots) {
        throw std::length_error("snapshot states exhausted");
    }
    std::size_t added = 0;
    for (std::size_t i = 0; i < sources.size(); ++i) {
        const auto& source = sources[i];
        if (source.file_id == 0) { throw std::logic_error("source needs file identity"); }
        if (static_cast<bool>(source.memory) == static_cast<bool>(source.disk)) {
            throw std::logic_error("source must own exactly one generation");
        }
        for (std::size_t j = 0; j < i; ++j) {
            if (sources[j].identity() == source.identity()) {
                throw std::logic_error("duplicate source in view");
            }
        }
        if (slot_for(source.identity()) == slots_.size()) { ++added; }
    }
    if (added > limits_.retained_sources - counts_.sources) {
        throw std::length_error("retained source slots exhausted");
    }
}
```


```cpp title="src/storage/read_view.cpp — replace void Context::retain"
void Context::retain(std::span<const Source> sources) noexcept {
    for (const auto& source : sources) {
        auto slot = slot_for(source.identity());
        if (slot == slots_.size()) {
            slot = slot_for(nullptr);
            if (slot == slots_.size()) { std::terminate(); }
            slots_[slot].identity = source.identity();
            slots_[slot].file_id = source.file_id;
            slots_[slot].kind = source.memory ? io::FileKind::log : io::FileKind::table;
            ++counts_.sources;
        }
        ++slots_[slot].references;
    }
    ++counts_.snapshots;
}
```

After `namespace read_detail` has closed, but before the file's final
`namespace strata::storage` closing brace, add this qualified definition:

```cpp title="src/storage/read_view.cpp — add persistent retention lookup"
bool read_detail::Context::retains_file(std::uint64_t id, io::FileKind kind) const noexcept {
    for (const auto& slot : slots_) {
        if (slot.references != 0 && slot.file_id == id && slot.kind == kind) { return true; }
    }
    return false;
}
```

A cursor owns its state even after it reaches end-of-range. Exhaustion describes
iteration, not object destruction. Resetting the last handle destroys the state,
which unregisters its sources. Standard shared ownership destroys the managed
object only when its final owning handle goes away; see the C++ draft's
[`shared_ptr` destructor](https://eel.is/c++draft/util.smartptr.shared.dest).
Our registry records which persistent names that live state protects. It does not
infer safety from an arbitrary `use_count()` value or from whether a cursor is
currently positioned on a row.

## Enumerate first, remove second

Deleting while advancing a directory stream makes enumeration behavior harder to
reason about. Instead, collect one bounded list of numbered files under the
already-held directory lock, close the stream, and then process the list. Chapter
11 already caps the entire directory at 4,096 non-dot entries. Reserve at most
that many `NumberedFile` records with the shared budget before enumeration.

The vector contains metadata, not table contents. Its requested capacity charge
is $4096\,\mathrm{sizeof}(\texttt{NumberedFile})$ bytes on the current C++ ABI;
`sizeof` may include padding, so do not hard-code a purported portable byte count.
Allocation failure occurs before any deletion. The existing strict checks on
reserved entries still reject symlinks, nonregular files, and multiple hard links.
Foreign names are counted toward the directory bound but never returned for
collection.

Replace `include/strata/io/directory.hpp` in full:

```cpp title="include/strata/io/directory.hpp — complete file"
#ifndef STRATA_IO_DIRECTORY_HPP
#define STRATA_IO_DIRECTORY_HPP

#include "strata/io/file.hpp"
#include "strata/memory/budget.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <optional>
#include <string>
#include <string_view>

namespace strata::io {

// A native descriptor owner; never use it for a simulated FileOps handle.
class Descriptor {
public:
    explicit Descriptor(int value = -1) noexcept : value_(value) {}
    ~Descriptor() noexcept;
    Descriptor(const Descriptor&) = delete;
    Descriptor& operator=(const Descriptor&) = delete;
    Descriptor(Descriptor&& other) noexcept;
    Descriptor& operator=(Descriptor&& other) noexcept;
    int get() const noexcept { return value_; }
    int release() noexcept;
private:
    int value_;
};

enum class FileKind { log, table, temporary };
using FileName = std::array<char, 27>;
FileName file_name(std::uint64_t id, FileKind kind);
std::optional<std::uint64_t> numbered_id(std::string_view name);

struct Inventory {
    std::uint64_t largest_id = 0;
    std::size_t numbered_files = 0;
    std::size_t entries = 0;
    bool manifest = false;
};
struct OpenedFile {
    File file;
    std::uint64_t bytes;
};

struct NumberedFile {
    std::uint64_t id;
    FileKind kind;
    std::uint64_t bytes;
};
struct RemovedFile { bool removed = false; std::uint64_t bytes = 0; };

class Directory {
public:
    static constexpr std::size_t max_entries = 4096;
    explicit Directory(const std::string& path);
    Directory(const Directory&) = delete;
    Directory& operator=(const Directory&) = delete;
    Directory(Directory&&) = delete;
    Directory& operator=(Directory&&) = delete;
    Inventory inventory() const noexcept { return inventory_; }
    OpenedFile open(const char* name, OpenMode mode);
    void sync();
    void replace_manifest(const char* temporary);
    memory::Vector<NumberedFile> numbered_files(memory::Budget& budget);
    RemovedFile remove_numbered(std::uint64_t id, FileKind kind);
private:
    Inventory scan(memory::Vector<NumberedFile>* files = nullptr);
    Descriptor directory_;
    Descriptor lock_;  // Released after every Database child file has closed.
    Inventory inventory_;
};

}  // namespace strata::io
#endif
```

Replace `Directory::scan` in `src/io/directory.cpp` with this definition. Its default
null output pointer preserves the constructor's inventory-only scan. The owned
`DIR*` is destroyed on normal return and on exceptions; descriptor ownership
transfers to `fdopendir` only after successful creation.

```cpp title="src/io/directory.cpp — replace Inventory Directory::scan"
Inventory Directory::scan(memory::Vector<NumberedFile>* files) {
    Descriptor scan_fd{open_relative(directory_.get(), ".", O_RDONLY | O_DIRECTORY)};
    DIR* raw = ::fdopendir(scan_fd.get());
    if (raw == nullptr) { native_error("fdopendir"); }
    (void)scan_fd.release();  // fdopendir owns it only after successful return.
    struct CloseStream {
        void operator()(DIR* stream) const noexcept { (void)::closedir(stream); }
    };
    std::unique_ptr<DIR, CloseStream> stream{raw};
    Inventory result;
    for (;;) {
        errno = 0;
        const auto* entry = ::readdir(stream.get());
        if (entry == nullptr) {
            if (errno != 0) { native_error("readdir"); }
            break;
        }
        const std::string_view name{entry->d_name};
        if (name == "." || name == "..") { continue; }
        if (++result.entries > max_entries) { throw std::length_error("directory entry cap"); }
        const auto id = numbered_id(name);
        if (id || name == "MANIFEST" || name == "LOCK") {
            struct stat info{};
            if (::fstatat(directory_.get(), entry->d_name, &info, AT_SYMLINK_NOFOLLOW) != 0) {
                native_error("stat directory entry");
            }
            check_regular(info);
            if (id && files) {
                const auto suffix = name.substr(22);
                const auto kind = suffix == ".wal" ? FileKind::log
                    : suffix == ".sst" ? FileKind::table : FileKind::temporary;
                files->push_back({*id, kind, static_cast<std::uint64_t>(info.st_size)});
            }
        }
        if (id) {
            result.largest_id = std::max(result.largest_id, *id);
            ++result.numbered_files;
        }
        if (name == "MANIFEST") { result.manifest = true; }
    }
    return result;
}
```

Append these definitions inside `namespace strata::io`, before its final closing
brace. `remove_numbered` can construct only a reserved numbered filename. It
cannot express `MANIFEST`, `LOCK`, a parent path, or `notes.txt`. It rechecks the
entry before one unlink attempt and reports an absent entry as already gone.

```cpp title="src/io/directory.cpp — add bounded enumeration and unlink"
memory::Vector<NumberedFile> Directory::numbered_files(memory::Budget& budget) {
    memory::Vector<NumberedFile> files{memory::Allocator<NumberedFile>{budget}};
    files.reserve(max_entries);  // One bounded request, charged before enumeration.
    inventory_ = scan(&files);
    return files;
}

RemovedFile Directory::remove_numbered(std::uint64_t id, FileKind kind) {
    const auto name = file_name(id, kind);  // Cannot name LOCK, MANIFEST, or a foreign file.
    struct stat info{};
    if (::fstatat(directory_.get(), name.data(), &info, AT_SYMLINK_NOFOLLOW) != 0) {
        if (errno == ENOENT) { return {}; }
        native_error("stat before unlink");
    }
    check_regular(info);
    // One attempt: failure is observed by the caller, never blindly retried here.
    if (::unlinkat(directory_.get(), name.data(), 0) != 0) { native_error("unlink numbered file"); }
    if (inventory_.entries != 0) { --inventory_.entries; }
    if (inventory_.numbered_files != 0) { --inventory_.numbered_files; }
    return {true, static_cast<std::uint64_t>(info.st_size)};
}
```

The database must still decide eligibility; this low-level method is not that
policy. We assume a trusted local directory and cooperating users of the lock,
as in Chapter 11. A check followed by `unlinkat` is not a defense against an
adversary who can replace directory entries between those calls. No such hostile
concurrent mutation is admitted by this course's file boundary.

A successful unlink removes a name, not necessarily the last live storage
reference. Apple's archived [`unlink(2)` manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/unlink.2.html)
explains why an already-open file can remain usable after its name disappears.
Consequently, a test that only reads an old cursor can miss premature unlink.
We will also assert that retained source names remain present. `removed_file_bytes`
will mean the summed logical lengths of successfully unlinked files, not measured
filesystem blocks or immediately recovered physical space.

## Expose bounded operations and scoped measurements

Create this header. A call may produce at most `max_output_files` files and at most
`max_output_bytes` completed output bytes. The manifest's remaining slots impose
an additional cap. Each file also obeys its `TableOptions::file_limit`, block
count, and metadata limits. The total cap is a bound on **this operation's new
table outputs**, not an operating-system reservation or a database-wide disk cap.
Old inputs, retained generations, newer files, logs, and manifest temporaries can
coexist with it; actual disk-full errors remain possible.

```cpp title="include/strata/storage/compaction.hpp — complete file"
#ifndef STRATA_STORAGE_COMPACTION_HPP
#define STRATA_STORAGE_COMPACTION_HPP

#include "strata/storage/table.hpp"

#include <cstddef>
#include <cstdint>

namespace strata::storage {
struct CompactionOptions {
    TableOptions output{};
    std::size_t max_output_files = 16;
    std::uint64_t max_output_bytes = 64 * 1024 * 1024;
};
struct CompactionStats {
    std::size_t selected_files = 0;
    std::size_t output_files = 0;
    std::uint64_t input_entries = 0;
    std::uint64_t output_entries = 0;
    std::uint64_t selected_file_bytes = 0;
    std::uint64_t merge_read_bytes = 0;
    std::uint64_t output_write_bytes = 0;
    std::uint64_t completed_file_bytes = 0;
    bool counters_saturated = false;
    bool installed = false;
};
struct ReclaimResult {
    std::size_t live = 0;
    std::size_t retained = 0;
    std::size_t removed = 0;
    std::size_t failed = 0;
    std::uint64_t removed_file_bytes = 0;
    bool counters_saturated = false;
    bool sync_failed = false;
    int first_error = 0;
};
}  // namespace strata::storage
#endif
```

The distinction between the measurements is intentional:

| Field | Meaning |
|---|---|
| `input_entries` | Stored entries advertised by selected, validated input tables |
| `output_entries` | Rows successfully appended so far, including an uninstalled attempt |
| `selected_file_bytes` | Logical sizes of selected files, including their metadata |
| `merge_read_bytes` | Successful transfers on selected input readers during this operation |
| `output_write_bytes` | Successful transfers on new output descriptors, including partial failures |
| `completed_file_bytes` | Sizes of finished output tables, even if installation later fails |
| `installed` | The manifest transition and nonthrowing owner publication completed |

The read meter excludes earlier table opening, output verification, WAL work, and
manifest traffic. Cache hits may avoid input transfers. These are not total
engine I/O counters, device-sector counters, a throughput benchmark, or process
memory measurements. A saturated counter must not be treated as exact.

Replace `database.hpp` with the following complete header. The trailing `&` on
mutating methods retains the earlier lvalue-only API. The `base` argument to
`open_table` selects stricter key/value rules but permits compacted-away sequence
endpoints; ordinary flushes continue calling it with the default `false`.

```cpp title="include/strata/storage/database.hpp — complete file"
#ifndef STRATA_STORAGE_DATABASE_HPP
#define STRATA_STORAGE_DATABASE_HPP

#include "strata/io/directory.hpp"
#include "strata/storage/manifest.hpp"
#include "strata/storage/wal.hpp"
#include "strata/storage/read_view.hpp"
#include "strata/storage/compaction.hpp"

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
    ReadLimits reads{};
};
struct InstallHook {
    using Function = void (*)(void*, std::string_view);
    Function function = nullptr;
    void* context = nullptr;
    void operator()(std::string_view stage) const {
        if (function != nullptr) { function(context, stage); }
    }
};

class Database {
public:
    Database(std::shared_ptr<memory::Budget> budget, const std::string& directory,
             DatabaseOpen mode, DatabaseOptions options = {}, InstallHook hook = {});
    Database(const Database&) = delete;
    Database& operator=(const Database&) = delete;
    Database(Database&&) = delete;
    Database& operator=(Database&&) = delete;
    [[nodiscard]] WriteResult write(std::span<const Mutation> changes) &;
    std::optional<OwnedEntry> get(Key key) &;
    ReadView snapshot() &;
    void compact_prefix(std::size_t selected, CompactionOptions options = {}) &;
    CompactionStats last_compaction() const noexcept { return compaction_; }
    ReclaimResult collect_obsolete() &;
    ReadRetention retention() const noexcept { return context_->retention(); }
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
    std::shared_ptr<TableReader> open_table(TableRecord record, bool base = false);
    std::shared_ptr<read_detail::Context> context_;
    memory::Budget& budget_;
    DatabaseOptions options_;
    InstallHook hook_;
    Manifest manifest_{};
    std::array<std::shared_ptr<WalMemTable>, Manifest::max_logs> logs_{};
    std::array<std::shared_ptr<TableReader>, Manifest::max_tables> tables_{};
    std::uint64_t next_id_ = 1;
    std::size_t orphans_at_open_ = 0;
    bool poisoned_ = false;
    CompactionStats compaction_{};
};

}  // namespace strata::storage
#endif
```

## Reopen the meaning, not just the bytes

A valid checksum proves neither uniqueness nor coverage. Replace `Database::open_table`
in `src/storage/database.cpp`. It retains exact file-length checks, charged shared
ownership, and full-table verification. For a base partition it additionally
requires puts only, strictly increasing user keys, and at least one row. Every
row's sequence must still belong to the declared interval. A delta must still
witness both endpoints of its interval.

```cpp title="src/storage/database.cpp — replace std::shared_ptr<TableReader> Database::open_table"
std::shared_ptr<TableReader> Database::open_table(TableRecord record, bool base) {
    const auto name = io::file_name(record.id, io::FileKind::table);
    auto opened = context_->directory.open(name.data(), io::OpenMode::read_only);
    if (opened.bytes != record.bytes) { throw FormatError("table length differs from manifest"); }
    const auto cache_id = context_->cache.new_file_id();
    auto reader = std::allocate_shared<TableReader>(memory::SharedAllocator<TableReader>{context_->budget},
        budget_, std::move(opened.file), opened.bytes, &context_->cache, cache_id);
    reader->verify_all();
    std::uint64_t smallest = std::numeric_limits<std::uint64_t>::max();
    std::uint64_t largest = 0;
    std::optional<Key> previous;
    auto cursor = reader->begin();
    while (cursor.valid()) {
        const auto entry = cursor.entry();
        const auto sequence = entry.key.sequence;
        if (sequence <= record.base || sequence > record.end) {
            throw FormatError("table sequence outside manifest interval");
        }
        if (base && (entry.kind != EntryKind::put ||
            (previous && !KeyLess{}(*previous, entry.key.user)))) {
            throw FormatError("base must have one live value per key");
        }
        previous = entry.key.user;
        smallest = std::min(smallest, sequence);
        largest = std::max(largest, sequence);
        cursor.next();
    }
    if (record.end <= record.base || (base ? !previous.has_value()
        : smallest != record.base + 1 || largest != record.end)) {
        throw FormatError("table interval endpoints or base contents missing");
    }
    return reader;
}
```

`previous` is an optional **user key**, not an internal key. Two versions of the
same user key therefore fail the base check even though they would be valid in
an ordinary sorted delta. Empty bases are represented by zero files, so dereferencing
`first_key` and `last_key` for a successfully opened base partition is safe.

Replace `recover` as follows. Adjacent base partitions must satisfy
$\operatorname{last}(i-1)<\operatorname{first}(i)$ under `KeyLess`. This is both
nonoverlap and increasing manifest order. Reordering two otherwise valid base
records and recomputing the manifest checksum must still be rejected.

```cpp title="src/storage/database.cpp — replace void Database::recover"
void Database::recover() {
    auto authority = context_->directory.open("MANIFEST", io::OpenMode::read_write);
    if (authority.bytes < Manifest::header_bytes + 24 + 4 || authority.bytes > Manifest::max_bytes) {
        throw FormatError("MANIFEST outside size bounds");
    }
    memory::Vector<std::byte> image(static_cast<std::size_t>(authority.bytes),
                                     memory::Allocator<std::byte>{budget_});
    authority.file.read_exact(0, image);
    manifest_ = decode_manifest(image);
    next_id_ = std::max(next_id_, manifest_.next_id);
    for (std::size_t i = 0; i < manifest_.table_count; ++i) {
        tables_[i] = open_table(manifest_.tables[i], i < manifest_.base_count);
        if (i != 0 && i < manifest_.base_count &&
            !KeyLess{}(*tables_[i - 1]->last_key(), *tables_[i]->first_key())) {
            throw FormatError("base file key ranges overlap or are out of order");
        }
    }
    for (std::size_t i = 0; i < manifest_.log_count; ++i) {
        logs_[i] = open_log(manifest_.logs[i], i + 1 == manifest_.log_count
            ? WalOpen::recover : WalOpen::recover_sealed);
    }
    // Stabilize the observed authority before allowing writes depending on it.
    context_->directory.sync();
    authority.file.sync(options_.sync_mode);
    authority.file.close_checked();
}
```

Current point reads still search newer deltas before the base. Searching base
partitions in reverse is harmless because their user-key ranges are disjoint;
at most one contains a key. Snapshot scans already merge any bounded source set
by internal order. Neither mechanism needs an in-memory map of all compacted keys.

Replace `snapshot` to supply persistent IDs alongside each owner. All Chapter 12
admission, rotation, and activation ordering is retained. No file identity is
looked up by dereferencing a possibly retired pointer.

```cpp title="src/storage/database.cpp — replace ReadView Database::snapshot"
ReadView Database::snapshot() & {
    require_healthy();
    const auto active = manifest_.log_count - 1;
    const bool freeze = logs_[active]->table().size() != 0;
    if (freeze && manifest_.log_count == Manifest::max_logs) {
        throw std::length_error("sealed-log slots exhausted");
    }
    std::array<read_detail::Source, read_detail::max_sources> sources{};
    std::size_t count = 0;
    for (std::size_t i = 0; i < manifest_.table_count; ++i) {
        sources[count].file_id = manifest_.tables[i].id;
        sources[count++].disk = tables_[i];
    }
    for (std::size_t i = 0; i < manifest_.log_count; ++i) {
        if (i != active || freeze) {
            sources[count].file_id = manifest_.logs[i].id;
            sources[count++].memory = logs_[i];
        }
    }
    const auto captured = std::span<const read_detail::Source>{sources.data(), count};
    context_->check_admission(captured);  // No allocation, I/O, sealing, or ID consumption.
    auto state = std::allocate_shared<read_detail::State>(
        memory::SharedAllocator<read_detail::State>{context_->budget},
        context_, sequence(), captured);
    if (freeze) { (void)rotate(); }  // Prepare replacement writer before sealing/publication.
    state->activate();              // No allocation and no user callback after rotation.
    return ReadView{std::move(state)};
}
```

## Stream a replacement through the existing heap

Create `src/storage/compaction.cpp` from the following segments in order. The first
segment contains all includes and local helpers. The meter destructor bodies perform no new transfers or synchronization; they
only collect already-recorded counters. Releasing their owner members can still
perform the existing nonthrowing descriptor cleanup.
The read meter retains the selected reader owners until it has measured them,
including when the serving array has been swapped. The write meter is declared
after its file owner, so it reads that owner's final counters before the file
object is destroyed.

```cpp title="src/storage/compaction.cpp — complete file, segment 1"
#include "strata/storage/database.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <limits>
#include <stdexcept>
#include <utility>

namespace strata::storage {
namespace {
struct ReadMeter {
    std::array<std::shared_ptr<TableReader>, Manifest::max_tables> readers{};
    std::array<io::IoBytes, Manifest::max_tables> before{};
    std::size_t count = 0;
    CompactionStats& stats;
    explicit ReadMeter(CompactionStats& target) noexcept : stats(target) {}
    ~ReadMeter() noexcept {
        for (std::size_t i = 0; i < count; ++i) {
            const auto after = readers[i]->io_bytes();
            stats.counters_saturated |= before[i].saturated || after.saturated;
            io::IoBytes::add(stats.merge_read_bytes, after.read - before[i].read,
                             stats.counters_saturated);
        }
    }
};
struct WriteMeter {
    io::File& file;
    CompactionStats& stats;
    ~WriteMeter() noexcept {
        const auto bytes = file.io_bytes();
        stats.counters_saturated |= bytes.saturated;
        io::IoBytes::add(stats.output_write_bytes, bytes.written, stats.counters_saturated);
    }
};
bool live_file(const Manifest& manifest, const io::NumberedFile& file) noexcept {
    if (file.kind == io::FileKind::table) {
        for (std::size_t i = 0; i < manifest.table_count; ++i) {
            if (manifest.tables[i].id == file.id) { return true; }
        }
    } else if (file.kind == io::FileKind::log) {
        for (std::size_t i = 0; i < manifest.log_count; ++i) {
            if (manifest.logs[i].id == file.id) { return true; }
        }
    }
    return false;
}
void remember_error(ReclaimResult& result, int error) noexcept {
    if (result.first_error == 0) { result.first_error = error; }
}
}  // namespace
```

The eligibility predicate for cleanup is deliberately separate from the retention
query. A file can be live in the current manifest even when no user snapshot
exists. A temporary manifest candidate is never authoritative merely because it
has a high number or valid-looking bytes.

Begin the compaction method. The `selected` count indexes the leading table
records, not a sequence number. It must be nonzero and at least the existing base-file
count: all of that base, optionally plus newer deltas. A base-only rewrite does
not advance `base_end`; it can repack small partitions into larger files. When no
base exists, `base_count == 0` and any nonempty oldest prefix is eligible.

If $K$ tables are selected and $U$ newer tables remain, the maximum output count
is $\min(\texttt{max_output_files},16-U)$. This accounts for the unselected owners
that must remain in the manifest. `next_manifest` checks epoch exhaustion before
file effects. Statistics are reset after this preflight; a rejected invalid
selection leaves the previous operation's statistics unchanged.

```cpp title="src/storage/compaction.cpp — complete file, segment 2"
void Database::compact_prefix(std::size_t selected, CompactionOptions options) & {
    require_healthy();
    // Select the whole existing base, optionally extending it with complete newer deltas.
    if (selected == 0 || selected < manifest_.base_count || selected > manifest_.table_count ||
        options.max_output_files == 0 || options.max_output_files > Manifest::max_tables ||
        options.max_output_bytes == 0) {
        throw std::invalid_argument("invalid complete-prefix compaction");
    }
    const auto newer = manifest_.table_count - selected;
    const auto output_limit = std::min(options.max_output_files, Manifest::max_tables - newer);
    const auto end = manifest_.tables[selected - 1].end;
    auto candidate = next_manifest();
    compaction_ = {};
    compaction_.selected_files = selected;
    ReadMeter meter{compaction_};
    std::array<read_detail::Source, read_detail::max_sources> sources{};
    for (std::size_t i = 0; i < selected; ++i) {
        sources[i].file_id = manifest_.tables[i].id;
        sources[i].disk = tables_[i];
        meter.readers[i] = tables_[i];
        meter.before[i] = tables_[i]->io_bytes();
        ++meter.count;
        io::IoBytes::add(compaction_.input_entries, tables_[i]->entries(), compaction_.counters_saturated);
        io::IoBytes::add(compaction_.selected_file_bytes, manifest_.tables[i].bytes,
                         compaction_.counters_saturated);
    }
    // Private, synchronous workspace: it is never activated as an admitted user snapshot.
    auto state = std::allocate_shared<read_detail::State>(
        memory::SharedAllocator<read_detail::State>{context_->budget}, context_, end,
        std::span<const read_detail::Source>{sources.data(), selected});
    ReadView view{std::move(state)};
    auto cursor = view.scan({Relation::events, 0});
    std::array<std::shared_ptr<TableReader>, Manifest::max_tables> next_tables{};
    candidate.tables = {};
    candidate.table_count = 0;
    candidate.base_count = 0;
    candidate.base_end = end;
    poisoned_ = true;  // Preserve all artifacts after any subsequent failure; reopen decides.
```

This segment deliberately ends inside the method. Append the next segments
without adding a closing brace here. `Database` is already a friend of `ReadView`,
so it may construct a private view over the selected sources. This temporary
state is not activated as an admitted user snapshot: it cannot escape and no
collector can run concurrently or reentrantly. Its shared owners still keep all
inputs alive. The ordinary heap scan does the difficult work from Chapter 12,
including consuming every older version of each key and copying the output row
before invalidating a borrowed input span.

The new array of shared readers is fixed-size. `candidate.tables` is rebuilt,
while its logs and flushed boundary remain copied from the current manifest.
`base_end` becomes the selected prefix's end, not the latest log sequence. Only
after the input scan and workspace exist do we mark the database poisoned and
begin output effects. Earlier preparation failures need not poison it.

Continue with the output loop:

```cpp title="src/storage/compaction.cpp — complete file, segment 3"
    while (cursor.valid()) {
        if (candidate.base_count == output_limit) { throw std::length_error("compaction output slots"); }
        const auto remaining = options.max_output_bytes - compaction_.completed_file_bytes;
        auto table_options = options.output;
        table_options.file_limit = std::min(table_options.file_limit, remaining);
        const auto id = take_id();
        const auto name = io::file_name(id, io::FileKind::table);
        auto output = context_->directory.open(name.data(), io::OpenMode::create_new);
        WriteMeter output_meter{output.file, compaction_};
        TableInfo info{};
        {
            TableBuilder builder{budget_, output.file, table_options};
            std::uint64_t rows = 0;
            while (cursor.valid() && builder.can_append(cursor.entry())) {
                builder.append(cursor.entry());
                ++rows;
                ++compaction_.output_entries;
                hook_("compaction-row");
                cursor.next();
            }
            if (rows == 0) { throw std::length_error("one row cannot fit compaction output"); }
            info = builder.finish();
        }  // Output workspaces are gone before reader metadata is allocated.
        compaction_.completed_file_bytes += info.file_bytes;
        ++compaction_.output_files;
        output.file.sync(options_.sync_mode);
        output.file.close_checked();
        hook_("compaction-file-ready");
        const TableRecord record{id, 0, end, info.file_bytes};
        next_tables[candidate.base_count] = open_table(record, true);
        candidate.tables[candidate.base_count++] = record;
    }
```

The cursor advances **only after** a row is appended successfully. A negative
forecast leaves that row available for the next file. If the first row cannot
fit, `rows == 0` rejects the operation rather than creating empty files forever.
With 300-byte files, eight event results become four 286-byte files; two account
results share a 254-byte file. The five outputs total $4\cdot286+254=1398$ bytes.
The total output cap can make the last file's effective cap smaller than 300.

Builder buffers are destroyed before allocating the new reader's metadata. The
file is synchronized and checked-closed before it is opened for serving. The new
reader is fully validated before the manifest may name it. This conservative
verification rereads outputs; it is not included in `merge_read_bytes`, and it
costs work. The meter remains outside the builder scope so successful header and
block writes are counted even if a later allocation, append, sync, or open fails.

A failure after `poisoned_ = true` leaves all artifacts in place and disables
serving through this database facade. An earlier read view may still read its
unchanged sources. Reopen determines authority; a subsequent explicit collection
can remove uninstalled outputs. This intentionally broad failure policy avoids
pretending that a partially executed maintenance operation rolled back its I/O.

Finish the method:

```cpp title="src/storage/compaction.cpp — complete file, segment 4"
    for (std::size_t i = 0; i < newer; ++i) {
        candidate.tables[candidate.base_count + i] = manifest_.tables[selected + i];
        next_tables[candidate.base_count + i] = tables_[selected + i];
    }
    candidate.table_count = candidate.base_count + newer;
    hook_("compaction-prepared");
    install(candidate);  // The only authority transition; old files still exist.
    tables_.swap(next_tables);
    compaction_.installed = true;
    poisoned_ = false;
}
```

Only `install(candidate)` changes authority. Chapter 11's ordering still applies:
output contents are synchronized, dependency names are synchronized, a new
manifest is written and synchronized, the manifest name is replaced, and the
directory and authority are stabilized. No old input is removed on this path.
The remaining array swap, trivial statistics changes, and owner releases cannot
allocate. There is no post-install reader construction that could leave the
manifest committed while serving owners are missing.

The empty-output case skips the entire loop, installs positive `base_end` with
zero base files, and retains any newer tables and logs. A later delta after that
boundary is still contiguous. An old view that needs a deleted value continues
using its old files; the new base need not retain that value for it.

For $N$ stored input entries in $K$ selected sources, the existing heap uses
$O(N\log\max(2,K))$ comparison work, plus the initial source setup. Its dynamic
storage is bounded by the source count: one position/pin per source, the heap,
and one owned result row. Output adds one builder workspace at a time and bounded
metadata/owner storage for completed outputs. Retained old states, the cache, and
those new allocations all use the shared budget. A small budget can therefore
reject compaction even though no collection of all $N$ rows is allocated.
Fixed stack arrays and ordinary process/runtime overhead remain outside the
charged-allocation promise from Chapter 9.

## Fence identities before erasing their evidence

Suppose a failed operation left `s-00000000000000001000.tmp`, but the installed
manifest still says `next_id == 40`. Reopen sees ID 1000 and raises the in-memory
high-water mark. Simply deleting the orphan would erase the only persistent
evidence of that ID. Another reopen could then allocate 40 onward and eventually
reuse 1000. That contradicts our permanent identity discipline.

Before deleting anything, install a manifest whose `next_id` is greater than
every observed numbered identity. `install` itself consumes a temporary ID, then
writes the resulting next-ID value, so its own temporary is covered too. An
uncertain fence installation poisons the database and performs **no deletion**.
If the highest ID is the maximum unsigned value, fail explicitly: there is no
representable next ID.

Append the final segment of `compaction.cpp`:

```cpp title="src/storage/compaction.cpp — complete file, segment 5"
ReclaimResult Database::collect_obsolete() & {
    require_healthy();
    auto files = context_->directory.numbered_files(budget_);
    const auto largest = context_->directory.inventory().largest_id;
    if (largest == std::numeric_limits<std::uint64_t>::max()) {
        throw std::overflow_error("orphan file identity exhausts namespace");
    }
    next_id_ = std::max(next_id_, largest + 1);
    // Fence every observed identity before deleting the evidence that it existed.
    if (manifest_.next_id < next_id_) {
        auto candidate = next_manifest();
        poisoned_ = true;
        install(candidate);
        poisoned_ = false;
    }
    ReclaimResult result;
    context_->cache.trim();  // Remove idle byte copies; pins retain independent allocations.
    for (const auto& file : files) {
        if (live_file(manifest_, file)) { ++result.live; continue; }
        if (context_->retains_file(file.id, file.kind)) { ++result.retained; continue; }
        try {
            hook_("before-unlink");
            const auto removal = context_->directory.remove_numbered(file.id, file.kind);
            if (removal.removed) {
                ++result.removed;
                io::IoBytes::add(result.removed_file_bytes, removal.bytes, result.counters_saturated);
            }
        } catch (const std::system_error& error) {
            ++result.failed;
            remember_error(result, error.code().value());
        } catch (...) {
            ++result.failed;
            remember_error(result, EIO);  // Test hooks have no required errno representation.
        }
    }
    try {
        hook_("before-cleanup-sync");
        context_->directory.sync();
        if (options_.sync_mode == io::SyncMode::full_device) {
            auto authority = context_->directory.open("MANIFEST", io::OpenMode::read_write);
            authority.file.sync(options_.sync_mode);
            authority.file.close_checked();
        }
    } catch (const std::system_error& error) {
        result.sync_failed = true;
        remember_error(result, error.code().value());
    } catch (...) {
        result.sync_failed = true;
        remember_error(result, EIO);
    }
    return result;
}

}  // namespace strata::storage
```

Enumeration and any high-water manifest installation precede unlink attempts.
The current manifest is checked first, the retained-source registry second. Only
files failing both checks are candidates. Idle cached byte copies are trimmed,
but a cache pin alone is not a file owner: it retains its own immutable allocation,
not a descriptor or a right to reopen a removed file. Every database cursor that
can advance to another block retains its state and source owner, which the
registry protects.

The result distinguishes failure to remove a file from failure to synchronize
the namespace afterward. Failed individual deletions remain discoverable by a
future scan; the collector continues with other eligible files. The original
error code is retained when available. A test-hook exception without an errno
is represented by `EIO`. Ordinary cleanup failure does not poison a database
whose current dependencies were never eligible for deletion.

Directory synchronization runs even when zero files were removed. This permits a
retry to resolve a previous cleanup-sync failure after all names are already
absent. In full-device mode, the existing explicit device-flush request follows
the directory synchronization. Apple's archived
[`fsync(2)` discussion](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
distinguishes a host flush from `F_FULLFSYNC`; issuing that request is not a
measurement of hardware persistence. Do not infer power-loss correctness from
our process-exit tests. SQLite's [atomic-commit discussion, particularly sections
9.2 and 9.5](https://www.sqlite.org/atomiccommit.html) is useful context on flush
assumptions and recovery-file lifetime; this implementation does not use SQLite's
rollback-journal algorithm.

## Follow one retirement to its final release

A sequence-1 snapshot owns an arena/log source. Flush creates a table; that old
snapshot still owns the log. A second snapshot owns the table. A cursor from the
second snapshot owns its state. Compaction later installs five new base files.
The first log and old table are now absent from current authority, but both are
retained. Neither may be unlinked by our policy.

Reset the first snapshot: the old log becomes eligible. Reset the second snapshot
but keep its cursor: the old table remains ineligible. Exhaust the cursor: still
ineligible. Reset the cursor: its state's last owner disappears, its retention
slot is released, and the next collector may remove the table. This is a state
transition controlled by ownership, not by a timer or a guess that a query should
have finished.

The collector requires a live database facade. When only an escaped view remains,
it continues holding the directory lock and sources, but performs no background
cleanup. After it releases them, a later database reopen can enumerate leftovers
and collect them. No filesystem operations are hidden in a custom source deleter.

Part 3 exercises these transitions, checks a completely deleted prefix, and
interrupts installation and cleanup in separate native processes.
