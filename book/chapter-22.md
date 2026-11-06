# Chapter 22 — Recovery and resource-pressure experiments

## Part 1 — Separate bytes, names, and acknowledgements

A process can terminate without losing the kernel's cached file contents. A
synchronization call can report an error even though the bytes survive. A newly
installed manifest can be readable while one of the names it references is
missing from the persisted directory. These are different experiments. A test
that treats every failure as “discard everything since the last successful
call” cannot distinguish them.

We will put the existing database behind a small, explicit persistence model,
then run a separate native child-process campaign. The model calls the real
`Database`, WAL decoder, table reader, installer, and collector. It does not
reimplement their decisions. A parent-side acknowledgement ledger determines
which recovered histories are allowed. Finally, native spilling queries will
encounter quota pressure, injected no-space errors, cancellation, and refused
cleanup without replacing an earlier completed result.

> **Learning objectives**
> State a crash model's assumptions; separate file contents from namespace
> bindings; preserve acknowledged batch history rather than merely row count;
> distinguish permitted tail repair from corruption; freeze an injected crash
> image despite host-language unwinding; coordinate a real process kill without
> sleeps; and follow pressure failures through ownership and cleanup.

**Prerequisites:** the complete implementation through
[Chapter 21](https://25d-228.github.io/cplusplus-learning/chapter-21.html), especially
Chapter 6's `FileOps`, Chapters 10–13's recovery and manifest rules, and Chapters
19–21's executor, spill hooks, and workload presets. This chapter adds six test
or script files, six narrowly scoped prerequisite-file edits, and a CMake append.
The production default remains native I/O. The threaded engine does not use the
synchronous test backend.

**Implementation: 4.0 hours across all three parts.** Allocate 1.25 hours to the
persistence model and boundary, 1.25 to the crash campaigns, and 1.5 to pressure,
diagnosis, and integration checks. Reading is separate. Reuse existing codecs,
fixtures, and resource machinery; do not build another storage engine inside the
test. Record actual implementation time.

Create these files. Concatenate successive blocks for a new filename in reading
order; the six `diff` blocks are exact edits to prerequisite files, not new files.

```text
tests/persistence_model.hpp
tests/persistence_model.cpp
tests/recovery_fixture.hpp
tests/recovery.cpp
tests/recovery_native.cpp
scripts/check_recovery.py
```

[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-22-part-2.html)
constructs allowed histories and crash images. [Part 3](https://25d-228.github.io/cplusplus-learning/chapter-22-part-3.html)
implements native process control, resource pressure, and complete run commands.

## A recovered state is not a subset of old keys

Let $B_i$ denote the complete mutation batch with sequence $i$. Let $D_i$ be the
logical database after applying $B_1$ through $B_i$ in order. An update replaces
an old value; a tombstone removes a key. Consequently, preserving acknowledged
history does **not** mean retaining every key that ever appeared in an
acknowledged batch. It means recovering the correct state of that history.

Let $A$ be the highest sequence acknowledged outside the instance that will
crash. Let $R$ be the recovered sequence. These tests have at most one attempted
but unacknowledged batch, $B_{A+1}$. With no such attempt, require $R=A$. With
that attempt, allow

$$
R\in\{A,A+1\},\qquad \text{recovered rows}=D_R.
$$

The second condition matters as much as the first. Reporting sequence 3 with
half of batch 3's updates is a failure. Returning sequence 2 with resurrected
batch-1 values is also a failure. Conversely, recovering all of an unacknowledged
batch can be legitimate: its synchronization might have completed before the
caller learned the result. A definitely rejected batch never enters this set.

The fixture's second batch changes account 10's region to 4, replaces event 1's
amount with 11, deletes event 2, and inserts event 3. Its third batch removes
account 20 and event 3, replaces event 1 with amount 13, and inserts account 30
and event 4. Thus sequence 2 and sequence 3 both contain four live rows. A row
count cannot distinguish them.

The ledger will live outside the simulated database object, or in the Python
parent outside the killed child. It is not reconstructed from `MANIFEST` or the
WAL. The native parent is still on the same powered machine: this is an external
**process-lifetime** observer, not a durable distributed client or a production
retry/deduplication protocol.

## Specify the model before implementing it

For inode $i$, keep volatile bytes and length $V_i$, plus synchronized bytes and
length $S_i$. A successful modeled file sync performs $S_i\leftarrow V_i$.
Separately, let $N_v$ map current names to inodes, and $N_s$ map synchronized names
to inodes. A successful directory sync performs $N_s\leftarrow N_v$. Syncing an
inode does not create its directory binding; syncing names does not invent
missing file contents.

A crash image chooses bytes and names independently. We exercise synchronized
contents, current contents, correct append prefixes, and a deterministic mix
that keeps current bytes for even-numbered inodes. Namespace choices include
synchronized names, current names, and a third image that retains a completed
manifest rename without unrelated unsynchronized creates or deletions. That
last choice exposes a dependency-ordering bug that an all-old/all-new namespace
model can miss.

This is a **bounded test model**, not an APFS implementation. It has one
preexisting root with a durable `LOCK`, one exclusive owner at a time, 64 inode
slots, 64 handles, and at most 16 KiB per file. A successful sync is an ideal
persistence barrier for the selected object. File and full-device sync modes
share that abstraction here; the model cannot validate the physical meaning of
either native operation. Existing synchronized bytes are not damaged by a later
append. Workloads append to WALs, build new immutable files, or shorten an
incomplete WAL tail; they do not overwrite an acknowledged prefix in place.

The model assumes an intact old or new manifest binding, not a half-decoded
rename. Its survival profiles are selected examples, **not all subsets of all
writes**, all sector layouts, concurrent filesystem histories, device failures,
or malicious modifications. One additional test creates a complete frame with
wrong bytes and requires fail-closed behavior. It does not promise automatic
salvage of arbitrary corruption. These limits are part of the result, not
reasons to omit the experiment.

SQLite's [atomic-commit discussion, Section 8](https://www.sqlite.org/atomiccommit.html#_testing_atomic_commit_behavior)
provides a useful primary example of testing recovery through a simulated file
interface. Its rollback-journal algorithm and crash model are not the algorithm
implemented here. We use the testing principle, while stating this model's
narrower assumptions explicitly.

## Keep simulated handles out of native descriptor owners

`File` already dispatches through `FileOps`, but `Directory` currently performs
native namespace operations directly. Add a small optional directory backend.
It returns ordinary `OpenedFile` objects, whose `File` borrows the same backend's
`FileOps` subobject. `Directory` retains a shared owner of the backend until all
its dependent files have been released through the existing context graph.

The abstract base has a virtual destructor because a shared owner can hold it
through the base type. `acquire`/`release` describe exclusive context ownership;
`release` must not throw. An unsuccessful `acquire` must leave no acquisition to
release. `scan` may allocate through its caller-provided charged vector. Native
`Descriptor` must never receive one of the model's small integer handle indices:
calling the operating system's `close` on such an integer could close an
unrelated real descriptor.

```diff title="include/strata/io/directory.hpp — exact prerequisite changes"
--- a/include/strata/io/directory.hpp
+++ b/include/strata/io/directory.hpp
@@ -8,6 +8,7 @@
 #include <cstddef>
 #include <cstdint>
 #include <optional>
+#include <memory>
 #include <string>
 #include <string_view>
 
@@ -51,10 +52,26 @@
 };
 struct RemovedFile { bool removed = false; std::uint64_t bytes = 0; };
 
+// An optional bounded diagnostic backend. Returned Files borrow this owner.
+// The native path remains the default; a backend must outlive every File it supplies.
+class DirectoryBackend {
+public:
+    virtual ~DirectoryBackend() = default;
+    virtual void acquire() = 0;
+    virtual void release() noexcept = 0;
+    virtual Inventory scan(memory::Vector<NumberedFile>* files) = 0;
+    virtual OpenedFile open(const char* name, OpenMode mode) = 0;
+    virtual void sync_directory() = 0;
+    virtual void replace_manifest(const char* temporary) = 0;
+    virtual RemovedFile remove_numbered(std::uint64_t id, FileKind kind) = 0;
+};
+
 class Directory {
 public:
     static constexpr std::size_t max_entries = 4096;
-    explicit Directory(const std::string& path);
+    explicit Directory(const std::string& path,
+                       std::shared_ptr<DirectoryBackend> backend = {});
+    ~Directory() noexcept;
     Directory(const Directory&) = delete;
     Directory& operator=(const Directory&) = delete;
     Directory(Directory&&) = delete;
@@ -67,6 +84,7 @@
     RemovedFile remove_numbered(std::uint64_t id, FileKind kind);
 private:
     Inventory scan(memory::Vector<NumberedFile>* files = nullptr);
+    std::shared_ptr<DirectoryBackend> backend_;
     Descriptor directory_;
     Descriptor lock_;  // Released after every Database child file has closed.
     Inventory inventory_;
```

The default argument is an empty `shared_ptr`, so every existing constructor call
still chooses the native path. A simulated constructor acquires first, then
scans. If scanning throws, the constructor itself releases the acquisition:
the destructor of a not-fully-constructed `Directory` will not run. In a completed
object, the destructor releases the acquisition while the backend owner still
exists. The native `directory_` and `lock_` members retain their original RAII
cleanup when no backend was supplied.

Keep component-name validation, exclusive-create inventory accounting, and
numbered-file restrictions in `Directory`. Delegation is only the operation
boundary, not permission to bypass those contracts.

```diff title="src/io/directory.cpp — exact prerequisite changes"
--- a/src/io/directory.cpp
+++ b/src/io/directory.cpp
@@ -92,9 +92,16 @@
     return value;
 }
 
-Directory::Directory(const std::string& path) {
+Directory::Directory(const std::string& path, std::shared_ptr<DirectoryBackend> backend)
+    : backend_(std::move(backend)) {
     if (path.empty() || path.find('\0') != std::string::npos) {
         throw std::invalid_argument("invalid database directory path");
+    }
+    if (backend_) {
+        backend_->acquire();
+        try { inventory_ = scan(); }
+        catch (...) { backend_->release(); throw; }
+        return;
     }
     int fd;
     do { fd = ::open(path.c_str(), O_RDONLY | O_DIRECTORY | O_CLOEXEC | O_NOFOLLOW); }
@@ -112,7 +119,12 @@
     inventory_ = scan();
 }
 
+Directory::~Directory() noexcept {
+    if (backend_) { backend_->release(); }
+}
+
 Inventory Directory::scan(memory::Vector<NumberedFile>* files) {
+    if (backend_) { return backend_->scan(files); }
     Descriptor scan_fd{open_relative(directory_.get(), ".", O_RDONLY | O_DIRECTORY)};
     DIR* raw = ::fdopendir(scan_fd.get());
     if (raw == nullptr) { native_error("fdopendir"); }
@@ -167,6 +179,11 @@
         break;
     default: throw std::invalid_argument("unknown open mode");
     }
+    if (backend_) {
+        auto opened = backend_->open(name, mode);
+        if (mode == OpenMode::create_new) { ++inventory_.entries; }
+        return opened;
+    }
     Descriptor fd{open_relative(directory_.get(), name, flags)};
     if (mode == OpenMode::create_new) { ++inventory_.entries; }
     struct stat info{};
@@ -176,13 +193,15 @@
 }
 
 void Directory::sync() {
+    if (backend_) { backend_->sync_directory(); return; }
     int result;
     do { result = ::fsync(directory_.get()); } while (result < 0 && errno == EINTR);
     if (result < 0) { native_error("sync database directory"); }
 }
 void Directory::replace_manifest(const char* temporary) {
     check_component(temporary);
-    if (::renameat(directory_.get(), temporary, directory_.get(), "MANIFEST") != 0) {
+    if (backend_) { backend_->replace_manifest(temporary); }
+    else if (::renameat(directory_.get(), temporary, directory_.get(), "MANIFEST") != 0) {
         native_error("replace MANIFEST");
     }
     if (inventory_.manifest) { --inventory_.entries; }
@@ -197,6 +216,14 @@
 }
 
 RemovedFile Directory::remove_numbered(std::uint64_t id, FileKind kind) {
+    if (backend_) {
+        const auto result = backend_->remove_numbered(id, kind);
+        if (result.removed) {
+            if (inventory_.entries != 0) { --inventory_.entries; }
+            if (inventory_.numbered_files != 0) { --inventory_.numbered_files; }
+        }
+        return result;
+    }
     const auto name = file_name(id, kind);  // Cannot name LOCK, MANIFEST, or a foreign file.
     struct stat info{};
     if (::fstatat(directory_.get(), name.data(), &info, AT_SYMLINK_NOFOLLOW) != 0) {
```

Pass the optional owner through `DatabaseOptions` into `read_detail::Context`.
The context already outlives retained sources and their files. The explicit
owner also prevents a test from returning a `File` whose virtual dispatch target
has already been destroyed. These changes do not make `Database` concurrent,
and the diagnostic backend below must not be supplied to concurrent workers.

```diff title="include/strata/storage/database.hpp — exact prerequisite changes"
--- a/include/strata/storage/database.hpp
+++ b/include/strata/storage/database.hpp
@@ -26,6 +26,7 @@
     TableOptions table{};
     io::SyncMode sync_mode = io::SyncMode::file;
     ReadLimits reads{};
+    std::shared_ptr<io::DirectoryBackend> directory_backend{};
 };
 struct InstallHook {
     using Function = void (*)(void*, std::string_view);
```

```diff title="include/strata/storage/read_view.hpp — exact prerequisite changes"
--- a/include/strata/storage/read_view.hpp
+++ b/include/strata/storage/read_view.hpp
@@ -48,7 +48,8 @@
 class Context {
 public:
     Context(std::shared_ptr<memory::Budget> owner, const std::string& directory_name,
-            std::size_t cache_slots, std::size_t cache_bytes, ReadLimits limits);
+            std::size_t cache_slots, std::size_t cache_bytes, ReadLimits limits,
+            std::shared_ptr<io::DirectoryBackend> backend = {});
     void check_admission(std::span<const Source> sources) const;
     void retain(std::span<const Source> sources) noexcept;
     void release(std::span<const Source> sources) noexcept;
```

```diff title="src/storage/read_view.cpp — exact prerequisite changes"
--- a/src/storage/read_view.cpp
+++ b/src/storage/read_view.cpp
@@ -30,9 +30,10 @@
 
 namespace read_detail {
 Context::Context(std::shared_ptr<memory::Budget> owner, const std::string& directory_name,
-                 std::size_t cache_slots, std::size_t cache_bytes, ReadLimits limits)
+                 std::size_t cache_slots, std::size_t cache_bytes, ReadLimits limits,
+                 std::shared_ptr<io::DirectoryBackend> backend)
     : budget(checked_owner(std::move(owner))),
-      directory(directory_name), cache(*budget, cache_slots, cache_bytes), limits_(limits) {
+      directory(directory_name, std::move(backend)), cache(*budget, cache_slots, cache_bytes), limits_(limits) {
     if (limits.snapshots == 0 || limits.snapshots > max_snapshots ||
         limits.retained_sources == 0 || limits.retained_sources > max_retained) {
         throw std::invalid_argument("invalid snapshot retention limits");
```

```diff title="src/storage/database.cpp — exact prerequisite changes"
--- a/src/storage/database.cpp
+++ b/src/storage/database.cpp
@@ -24,7 +24,7 @@
                    DatabaseOpen mode, DatabaseOptions options, InstallHook hook)
     : context_(std::allocate_shared<read_detail::Context>(
           memory::SharedAllocator<read_detail::Context>{budget}, budget, directory,
-          options.cache_slots, options.cache_bytes, options.reads)),
+          options.cache_slots, options.cache_bytes, options.reads, options.directory_backend)),
       budget_(*context_->budget), options_(options), hook_(hook) {
     if (options.arena_bytes < memory::Arena::chunk_charge() ||
         (options.sync_mode != io::SyncMode::file && options.sync_mode != io::SyncMode::full_device)) {
```

No installer ordering or recovery rule changes in these edits. A test backend
is valuable precisely because it reaches the same implementation. Replacing
`install` with a separate simulated installer would allow the test to prove the
wrong program correct.

## Represent identity separately from reachability

A name holds a small inode index; a handle holds the same index plus access mode.
Removing a name does not invalidate existing handles. Conversely, keeping file
bytes without a name does not make those bytes discoverable during reopening.
Slots are not recycled within one short campaign. This deliberately avoids an
inode-reuse simulation while leaving production file-ID fencing under test.

`PersistenceModel` implements **two independent interfaces** using public
inheritance. `DirectoryBackend` supplies namespace operations; `FileOps` supplies
positioned byte operations. There is no common virtual base and no diamond.
`shared_ptr<PersistenceModel>` converts to `shared_ptr<DirectoryBackend>` while
retaining the original object's control ownership; `File::adopt(*this, index)`
selects the `FileOps` base by its parameter type.

```cpp title="tests/persistence_model.hpp — complete file, policy and diagnostic surface"
#ifndef STRATA_TEST_PERSISTENCE_MODEL_HPP
#define STRATA_TEST_PERSISTENCE_MODEL_HPP
#include "strata/io/directory.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <memory>
#include <optional>
#include <span>
#include <string_view>

namespace recovery_test {
enum class Effect { create, write, file_sync, directory_sync, rename, truncate, remove, acknowledgement };
enum class Bytes { stable, current, prefix, alternating, torn };
struct Image { Bytes bytes; bool current_names; std::size_t prefix = 5; bool rename_only = false; };
struct Trace { Effect effect{}; std::size_t inode = 0; std::uint64_t offset = 0; std::size_t count = 0; };

// Synchronous, single-directory test backend. It is NOT a concurrent filesystem.
class PersistenceModel final : public strata::io::DirectoryBackend, public strata::io::FileOps {
public:
    static constexpr std::size_t max_nodes = 64, file_cap = 16384, max_handles = 64;
    PersistenceModel();
    std::shared_ptr<PersistenceModel> clone() const;
    void arm(std::size_t cut = 0) noexcept;
    void checkpoint();
    void restart(Image image);
    bool crashed() const noexcept { return crashed_; }
    std::size_t steps() const noexcept { return steps_; }
    Trace last() const noexcept { return last_; }
    std::size_t handles() const noexcept;
    bool locked() const noexcept { return locked_; }
    std::size_t node_for(std::string_view name) const;
    std::span<std::byte> volatile_bytes(std::string_view name);
    void append_garbage(std::string_view name, std::size_t bytes);
    void erase_name(std::string_view name);

    std::optional<std::size_t> write_allowance;
    bool fail_file_sync = false, failed_sync_persists = false;
    bool fail_directory_sync = false, failed_directory_sync_persists = false;
    bool fail_unlink = false;
```

`Image` is a value describing one restart. `current_names=false` selects the
synchronized namespace; `rename_only=true` overrides that choice with the shadow
rename image. `prefix` is a byte count, not a percentage. `write_allowance` is
optional: absence means no injected byte limit, while zero requests immediate
no-space failure.

The overridden file calls are `noexcept` because Chapter 6's byte boundary
reports numeric errors. Namespace methods may throw `system_error`; they already
feed the database's maintenance failure path. Keep the return conventions
different rather than letting an exception escape a `noexcept` override.

```cpp title="tests/persistence_model.hpp — complete file, operation overrides"
    void acquire() override;
    void release() noexcept override;
    strata::io::Inventory scan(strata::memory::Vector<strata::io::NumberedFile>* files) override;
    strata::io::OpenedFile open(const char* name, strata::io::OpenMode mode) override;
    void sync_directory() override;
    void replace_manifest(const char* temporary) override;
    strata::io::RemovedFile remove_numbered(std::uint64_t id, strata::io::FileKind kind) override;
    strata::io::Transfer read_at(int fd, std::span<std::byte> out, std::uint64_t offset) noexcept override;
    strata::io::Transfer write_at(int fd, std::span<const std::byte> in, std::uint64_t offset) noexcept override;
    int sync(int fd, strata::io::SyncMode mode) noexcept override;
    int truncate_to(int fd, std::uint64_t length) noexcept override;
    int close(int fd) noexcept override;
```

The two fixed byte arrays in every node are the test machine's simulated disk,
not engine workspaces. Their payload alone is

$$
64\times 2\times 16384=2097152\text{ bytes}.
$$

Names, handle records, lengths, and object/control overhead are additional.
This harness allocation is deliberately outside `memory::Budget`; each tested
database has a separate 8 MiB budget. A zero engine charge proves that its
tracked owners released, not that the entire process uses zero memory.

```cpp title="tests/persistence_model.hpp — complete file, bounded state"
private:
    struct Contents { std::array<std::byte, file_cap> data{}; std::size_t length = 0; };
    struct Node { Contents live, stable; };
    struct Name { std::array<char, 27> text{}; std::size_t inode = 0; bool used = false; };
    using Names = std::array<Name, max_nodes>;
    struct Handle { std::size_t inode = 0; bool used = false, writable = false; };
    static std::size_t find(const Names& names, std::string_view name) noexcept;
    static void name_slot(Name& slot, std::string_view name, std::size_t inode);
    Handle* handle(int fd) noexcept;
    void healthy() const;
    bool effect(Effect event, std::size_t node = 0, std::uint64_t offset = 0,
                std::size_t count = 0) noexcept;
    std::array<Node, max_nodes> nodes_{};
    Names live_names_{}, stable_names_{}, rename_names_{};
    std::array<Handle, max_handles> handles_{};
    std::size_t nodes_used_ = 1, steps_ = 0, cut_ = 0;
    bool locked_ = false, crashed_ = false;
    Trace last_{};
};
}  // namespace recovery_test
#endif
```

## Freeze effects, not C++ cleanup

The model begins with one stable `LOCK` binding. It permits cloning only after
all owners and handles are gone. A clone is a full value copy of this bounded
machine, not a copy of a live database connection. Short-circuit name checks
avoid treating unused slots as strings.

```cpp title="tests/persistence_model.cpp — complete file, names, ownership, and seed state"
#include "persistence_model.hpp"

#include <algorithm>
#include <cerrno>
#include <exception>
#include <stdexcept>
#include <system_error>

namespace recovery_test {
using namespace strata;
namespace {
[[noreturn]] void fail(int code, const char* what) {
    throw std::system_error(code, std::generic_category(), what);
}
}
std::size_t PersistenceModel::find(const Names& names, std::string_view name) noexcept {
    for (std::size_t i = 0; i < names.size(); ++i) {
        if (names[i].used && std::string_view{names[i].text.data()} == name) { return i; }
    }
    return names.size();
}
void PersistenceModel::name_slot(Name& slot, std::string_view name, std::size_t inode) {
    if (name.empty() || name.size() >= slot.text.size() || name.find('/') != name.npos ||
        name.find('\0') != name.npos) { throw std::invalid_argument("model name"); }
    slot = {}; std::copy(name.begin(), name.end(), slot.text.begin());
    slot.used = true; slot.inode = inode;
}
PersistenceModel::PersistenceModel() {
    name_slot(live_names_[0], "LOCK", 0); stable_names_ = live_names_; rename_names_ = stable_names_;
}
std::shared_ptr<PersistenceModel> PersistenceModel::clone() const {
    if (locked_ || handles()) { throw std::logic_error("clone needs closed owners"); }
    return std::make_shared<PersistenceModel>(*this);
}
std::size_t PersistenceModel::handles() const noexcept {
    return static_cast<std::size_t>(std::count_if(handles_.begin(), handles_.end(),
                                                [](const auto& h) { return h.used; }));
}
void PersistenceModel::healthy() const {
    if (crashed_) { fail(EIO, "model power cut"); }
    if (!locked_) { throw std::logic_error("model directory not acquired"); }
}
void PersistenceModel::acquire() {
    if (crashed_) { fail(EIO, "restart model before acquire"); }
    if (locked_) { fail(EWOULDBLOCK, "model lock held"); }
    locked_ = true;
}
void PersistenceModel::release() noexcept {
    if (!locked_ || handles() != 0) { std::terminate(); }
    locked_ = false;
}
```

`arm(k)` counts persistent effects from a known point. After the $k$th effect,
`crashed_` becomes true. Later reads and writes return `EIO`; namespace operations
throw. Closing handles and releasing host-side owners remain legal and cannot
change the frozen byte/name image. Thus the real database can unwind in the test
process without accidentally synchronizing or deleting its way out of the crash.

This is a simulation technique, not a claim that destructors run on a killed
process. The native campaign in Part 3 uses actual termination without normal
C++ unwinding. Acknowledgement is also a counted boundary: the external ledger
is updated before calling `checkpoint`, so cutting there must preserve the newly
acknowledged batch.

```cpp title="tests/persistence_model.cpp — complete file, effect counting and freezing"
void PersistenceModel::arm(std::size_t cut) noexcept {
    steps_ = 0; cut_ = cut; last_ = {};
}
bool PersistenceModel::effect(Effect event, std::size_t node, std::uint64_t offset,
                              std::size_t count) noexcept {
    last_ = {event, node, offset, count};
    ++steps_;
    if (cut_ != 0 && steps_ == cut_) { crashed_ = true; }
    return crashed_;
}
void PersistenceModel::checkpoint() {
    healthy();
    if (effect(Effect::acknowledgement)) { fail(EIO, "cut after external acknowledgement"); }
}
```

Restart chooses a namespace first, then bytes. For a correct append prefix, let
$s$ be the synchronized file length, $v$ the volatile length, and $p$ the selected
extra-byte count. When $v\ge s$, the chosen length is

$$
\ell=s+\min(p,v-s).
$$

The prefix reuses the actual written bytes, not zeros. For a shortening operation
($v<s$), this profile chooses the old synchronized contents; the current profile
chooses the shortened contents. The alternating profile varies by inode, not by
write call, and therefore does not claim exhaustive reorder coverage. The torn
profile is a separate corruption experiment: it retains current bytes and flips
one newly appended byte when such a byte exists.

```cpp title="tests/persistence_model.cpp — complete file, select the next boot image"
void PersistenceModel::restart(Image image) {
    if (locked_ || handles()) { throw std::logic_error("restart while owners survive"); }
    if (image.rename_only) { live_names_ = rename_names_; }
    else if (!image.current_names) { live_names_ = stable_names_; }
    stable_names_ = live_names_; rename_names_ = stable_names_;
    for (std::size_t i = 0; i < nodes_used_; ++i) {
        auto& n = nodes_[i];
        const bool keep = image.bytes == Bytes::current || image.bytes == Bytes::torn ||
                          (image.bytes == Bytes::alternating && i % 2 == 0);
        if (image.bytes == Bytes::prefix && n.live.length >= n.stable.length) {
            n.live.length = n.stable.length +
                std::min(image.prefix, n.live.length - n.stable.length);
        } else if (!keep) { n.live = n.stable; }
        if (image.bytes == Bytes::torn && n.live.length > n.stable.length) {
            n.live.data[n.live.length - 1] ^= std::byte{1};
        }
        n.stable = n.live;  // The chosen crash image is the next boot's stable baseline.
    }
    crashed_ = false; arm(); write_allowance.reset();
    fail_file_sync = failed_sync_persists = false;
    fail_directory_sync = failed_directory_sync_persists = fail_unlink = false;
}
```

Once selected, the image becomes the next boot's stable baseline. Fault controls
reset, while inode identities and chosen bytes remain. `restart` rejects live
owners; it is not a method for changing persistence underneath running queries.
The following helpers permit deliberate damage only with no live handles. The
name `append_garbage` describes zero extension, **not** a valid WAL tail; Part 2
will overwrite it with bytes from a real encoded batch for the repair scenario.

```cpp title="tests/persistence_model.cpp — complete file, closed-fixture damage helpers"
std::size_t PersistenceModel::node_for(std::string_view name) const {
    const auto index = find(live_names_, name);
    if (index == live_names_.size()) { fail(ENOENT, "model name missing"); }
    return live_names_[index].inode;
}
std::span<std::byte> PersistenceModel::volatile_bytes(std::string_view name) {
    if (locked_ || handles()) { throw std::logic_error("damage needs closed owners"); }
    auto& c = nodes_[node_for(name)].live; return {c.data.data(), c.length};
}
void PersistenceModel::append_garbage(std::string_view name, std::size_t bytes) {
    (void)volatile_bytes(name);
    auto& c = nodes_[node_for(name)].live;
    if (bytes > file_cap - c.length) { throw std::length_error("model damage size"); }
    std::fill_n(c.data.begin() + static_cast<std::ptrdiff_t>(c.length), bytes, std::byte{0});
    c.length += bytes;
}
void PersistenceModel::erase_name(std::string_view name) {
    (void)volatile_bytes(name); live_names_[find(live_names_, name)] = {};
}
```

## Open, sync, rename, and unlink are different effects

Scanning reports the same numbered-name grammar as the native directory. It does
not inspect contents to decide whether an unreferenced file is authoritative.
`open(create_new)` first allocates an inode and binding, records the creation,
and only then installs a handle. A cut immediately after creation can therefore
leave a name without returning a `File` to the caller. No descriptor is leaked
because the handle has not yet been published.

```cpp title="tests/persistence_model.cpp — complete file, inventory and file creation"
io::Inventory PersistenceModel::scan(memory::Vector<io::NumberedFile>* files) {
    healthy(); io::Inventory result;
    for (const auto& item : live_names_) {
        if (!item.used) { continue; }
        ++result.entries;
        const std::string_view name{item.text.data()};
        if (name == "MANIFEST") { result.manifest = true; }
        if (const auto id = io::numbered_id(name)) {
            result.largest_id = std::max(result.largest_id, *id); ++result.numbered_files;
            const auto suffix = name.substr(22);
            const auto kind = suffix == ".wal" ? io::FileKind::log :
                suffix == ".sst" ? io::FileKind::table : io::FileKind::temporary;
            if (files) { files->push_back({*id, kind, nodes_[item.inode].live.length}); }
        }
    }
    return result;
}
io::OpenedFile PersistenceModel::open(const char* name, io::OpenMode mode) {
    healthy();
    auto slot = find(live_names_, name);
    std::size_t descriptor = 0;
    while (descriptor < handles_.size() && handles_[descriptor].used) { ++descriptor; }
    if (descriptor == handles_.size()) { fail(EMFILE, "model handles exhausted"); }
    if (mode == io::OpenMode::create_new) {
        if (slot != live_names_.size()) { fail(EEXIST, "model exclusive create"); }
        slot = 0; while (slot < live_names_.size() && live_names_[slot].used) { ++slot; }
        if (slot == live_names_.size() || nodes_used_ == max_nodes) { fail(ENOSPC, "model inode cap"); }
        name_slot(live_names_[slot], name, nodes_used_++);
        if (effect(Effect::create, live_names_[slot].inode)) { fail(EIO, "cut after create"); }
    } else if (slot == live_names_.size()) { fail(ENOENT, "model open"); }
    const auto node = live_names_[slot].inode;
    handles_[descriptor] = {node, true, mode != io::OpenMode::read_only};
    return {io::File::adopt(*this, static_cast<int>(descriptor)), nodes_[node].live.length};
}
```

A directory sync error may occur with or without persistence. The flag controls
that uncertainty independently of its return value. The shadow namespace starts
at the last successful name barrier. On a manifest rename it receives the new
`MANIFEST` binding, but not unrelated current names. This yields a concrete
counterexample image: a new manifest survives while a newly created log's name
does not. The correctly ordered installer prevents that combination by syncing
the dependency names before the rename can happen.

```cpp title="tests/persistence_model.cpp — complete file, name persistence and atomic replacement"
void PersistenceModel::sync_directory() {
    healthy();
    if (!fail_directory_sync || failed_directory_sync_persists) {
        stable_names_ = live_names_; rename_names_ = stable_names_;
    }
    if (effect(Effect::directory_sync) || fail_directory_sync) { fail(EIO, "model directory sync"); }
}
void PersistenceModel::replace_manifest(const char* temporary) {
    healthy();
    const auto from = find(live_names_, temporary);
    if (from == live_names_.size()) { fail(ENOENT, "model rename source"); }
    const auto to = find(live_names_, "MANIFEST");
    if (to != live_names_.size()) { live_names_[to] = {}; }
    const auto node = live_names_[from].inode;
    name_slot(live_names_[from], "MANIFEST", node);
    // An unsynchronized rename can persist without unrelated creates or deletes.
    const auto shadow_from = find(rename_names_, temporary);
    if (shadow_from != rename_names_.size()) { rename_names_[shadow_from] = {}; }
    auto shadow_to = find(rename_names_, "MANIFEST");
    if (shadow_to == rename_names_.size()) {
        shadow_to = 0;
        while (shadow_to < rename_names_.size() && rename_names_[shadow_to].used) { ++shadow_to; }
        if (shadow_to == rename_names_.size()) { throw std::logic_error("model shadow namespace full"); }
    }
    name_slot(rename_names_[shadow_to], "MANIFEST", node);
    if (effect(Effect::rename, node)) { fail(EIO, "cut after rename"); }
}
```

Removal changes only reachability. The returned size belongs to the removed
binding's inode, and open handles continue to point there. `EACCES` is injected
before removal; a crash cut is injected after removal. Those cases must not be
reported identically. The collector can confirm a refusal, but a cut after
unlink leaves the surviving namespace undecided until restart chooses an image.

```cpp title="tests/persistence_model.cpp — complete file, unlink and handle validation"
io::RemovedFile PersistenceModel::remove_numbered(std::uint64_t id, io::FileKind kind) {
    healthy();
    if (fail_unlink) { fail(EACCES, "model deletion denied"); }
    const auto name = io::file_name(id, kind);
    const auto slot = find(live_names_, name.data());
    if (slot == live_names_.size()) { return {}; }
    const auto node = live_names_[slot].inode;
    const auto length = nodes_[node].live.length;
    live_names_[slot] = {};  // Open handles continue to name the inode.
    if (effect(Effect::remove, node)) { fail(EIO, "cut after unlink"); }
    return {true, length};
}
PersistenceModel::Handle* PersistenceModel::handle(int fd) noexcept {
    if (fd < 0 || static_cast<std::size_t>(fd) >= handles_.size() || !handles_[static_cast<std::size_t>(fd)].used) {
        return nullptr;
    }
    return &handles_[static_cast<std::size_t>(fd)];
}
```

## Short transfers expose the existing retry loops

Reads and writes transfer at most seven bytes. This is a deliberate test limit,
not a sector size or a native performance choice. `File::write_all` must advance
both its buffer and its offset after each successful short write. If an allowance
of 19 bytes remains, transfers of 7, 7, and 5 succeed; the next attempt returns
`ENOSPC`.

At a crash cut the model may copy bytes and then return zero progress with an
error. That reproduces the uncertainty at the caller boundary: observing an
error is not evidence that no side effect happened. After `crashed_` latches,
subsequent attempts cannot modify the frozen image.

```cpp title="tests/persistence_model.cpp — complete file, bounded read and write"
io::Transfer PersistenceModel::read_at(int fd, std::span<std::byte> out,
                                       std::uint64_t offset) noexcept {
    if (crashed_) { return {0, EIO}; }
    const auto* h = handle(fd); if (!h) { return {0, EBADF}; }
    const auto& c = nodes_[h->inode].live;
    if (offset >= c.length) { return {0, 0}; }
    const auto at = static_cast<std::size_t>(offset);
    const auto count = std::min({out.size(), c.length - at, std::size_t{7}});
    std::copy_n(c.data.begin() + static_cast<std::ptrdiff_t>(at), count, out.begin());
    return {count, 0};
}
io::Transfer PersistenceModel::write_at(int fd, std::span<const std::byte> in,
                                        std::uint64_t offset) noexcept {
    if (crashed_) { return {0, EIO}; }
    const auto* h = handle(fd); if (!h || !h->writable) { return {0, EBADF}; }
    if (offset >= file_cap || (write_allowance && *write_allowance == 0)) { return {0, ENOSPC}; }
    auto& c = nodes_[h->inode].live; const auto at = static_cast<std::size_t>(offset);
    auto count = std::min({in.size(), file_cap - at, std::size_t{7}});
    if (write_allowance) { count = std::min(count, *write_allowance); *write_allowance -= count; }
    std::copy_n(in.begin(), count, c.data.begin() + static_cast<std::ptrdiff_t>(at));
    c.length = std::max(c.length, at + count);
    if (effect(Effect::write, h->inode, offset, count)) { return {0, EIO}; }
    return {count, 0};
}
```

A successful file sync copies contents **and length**, not names. An injected
failed sync either performs that copy or leaves the synchronized version alone.
Truncation changes only the volatile length until an appropriate sync. This is
why replay must sync a repaired active tail before publishing its recovered
owner. Close merely releases a handle; neither byte durability nor namespace
persistence is smuggled into destruction.

```cpp title="tests/persistence_model.cpp — complete file, sync, truncation, and close"
int PersistenceModel::sync(int fd, io::SyncMode mode) noexcept {
    if (crashed_) { return EIO; }
    const auto* h = handle(fd); if (!h) { return EBADF; }
    if (mode != io::SyncMode::file && mode != io::SyncMode::full_device) { return EINVAL; }
    auto& n = nodes_[h->inode];
    if (!fail_file_sync || failed_sync_persists) { n.stable = n.live; }
    if (effect(Effect::file_sync, h->inode) || fail_file_sync) { return EIO; }
    return 0;
}
int PersistenceModel::truncate_to(int fd, std::uint64_t length) noexcept {
    if (crashed_) { return EIO; }
    const auto* h = handle(fd); if (!h || !h->writable) { return EBADF; }
    auto& c = nodes_[h->inode].live;
    if (length > c.length) { return EINVAL; }
    c.length = static_cast<std::size_t>(length);
    return effect(Effect::truncate, h->inode, length) ? EIO : 0;
}
int PersistenceModel::close(int fd) noexcept {
    auto* h = handle(fd); if (!h) { return EBADF; }
    *h = {}; return 0;  // Close neither syncs bytes nor changes the namespace.
}
}  // namespace recovery_test
```

The backend is now complete. It has enough state to ask whether a manifest's
bytes, its binding, and its dependencies survive independently. The next part
uses explicit batch states to decide which combinations must reopen successfully
and which damaged images must be rejected.

