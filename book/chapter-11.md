# Chapter 11 — Flush installation and database reopening

## Part 1 — Authority, identities, and the directory boundary

A synchronized table is not yet part of a database. Its bytes might be complete,
but a new process still needs to know whether to use it, which log it replaces,
and which other files must exist. Conversely, a perfectly readable old log is not
permission to ignore a missing installed table. The database needs one explicit
source of authority, and a protocol that changes that authority without losing
its recovery dependencies.

This chapter builds a synchronous database around the existing skip list, table,
cache, and write-ahead log. It installs complete manifest snapshots, rotates the
active log, flushes sealed generations in order, owns a directory lock, and opens
only the files named by validated authority. Current-state point reads make the
integration observable. Immutable user snapshots and merged range reads remain
the next chapter; this implementation is still single-threaded.

> **Learning objectives**
>
> Separate file contents from persistent names; encode a bounded authoritative
> catalog; distinguish active-log tail repair from sealed-log corruption; prepare
> all serving owners before publication; retain recovery dependencies across failed
> maintenance; and diagnose the difference between an exception, abrupt process
> exit, and a physical persistence failure.

**Implementation allocation: 5.5 hours for all three parts.** Allow 1.25 hours
for manifest code, 1.75 for rotation/flush installation, 1.5 for the directory and
reopen path, and 1.0 for integration and fault checks. Reading is separate. These
are provisional implementation estimates, not a reason to skip an invariant.

Use the complete source state through
[Chapter 10](https://25d-228.github.io/cplusplus-learning/chapter-10.html), including
Chapter 9's charged constructors and Chapter 10's `truncate_to` methods. Keep the
fourteen existing CTest entries. This part creates `storage/manifest.*` and
`io/directory.*` and adds native descriptor adoption to `File`. Part 2 explicitly
extends `WalMemTable` and adds `storage/database.*`; Part 3 supplies all tests
and build additions. Repeated
filename blocks append in order. A block explicitly marked “replace” replaces
that named definition, not the whole translation unit.

```sh title="Terminal — source directories"
mkdir -p include/strata/io include/strata/storage src/io src/storage tests scripts
```

## A catalog is not a directory listing

Suppose the directory contains an old log, a half-written table, and a temporary
manifest. Sorting names and choosing the largest number does not answer whether
any of those files became authoritative. A larger identifier means “allocated
later,” not “successfully committed.” Reopening therefore reads exactly the file
named `MANIFEST`. A missing or corrupt authoritative manifest is an opening
failure. There is no “try the previous one” branch.

The permanent filename `LOCK` is a separate coordination object. The numbered
namespace contains `s-00000000000000000001.wal`, similarly named `.sst` files, and
`.tmp` manifest candidates. A temporary manifest is never an alternative recovery
root. Numbered files not referenced by the manifest are *orphans* for this open:
they may be unfinished outputs or retired dependencies. We count and preserve
them, rather than interpreting or deleting them. Physical reclamation will need
its own lifetime and high-water rules.

Creation and recovery are distinct operations. Recovery never creates an empty
database merely because `MANIFEST` disappeared. Creation refuses a directory
containing an existing manifest or any numbered engine file. An unsuccessful
first initialization can consequently leave a directory that requires deliberate
operator investigation, not automatic erasure and reuse. Unrelated files such as
`notes.txt` are preserved.

The caller supplies an existing, trusted local directory. Durability arguments
below assume that this directory and its parent-name binding are already stable.
Opening it is not an implementation of recursively creating and synchronizing
parent directories. The engine does not support network filesystems, hostile
concurrent file replacement, or manual edits while it is open.

## Give every installed source a sequence interval

A batch sequence is positive and strictly increasing. For a sealed log or a
flushed table, write its covered sequence interval as $(b,e]$: the base $b$ is
excluded, the end $e$ is included. The base is not a row to replay. A new active
log with base 2 begins with no rows and reports sequence 2; its first batch must
have sequence 3.

In this chapter each flushed generation produces **one table containing all its
versions**, including tombstones. The manifest contains at most sixteen tables
and four required logs, with the active log last. The other three log slots are
sealed generations. Table intervals form a contiguous prefix beginning after
zero. Sealed logs continue that prefix. The final active log has `end == 0` as an
open-end marker; its actual end is discovered by replay. Zero cannot be a sealed
end because a sealed generation must contain a batch.

Let $F$ denote the last flushed sequence. With $T$ table entries and $L$ log
entries, validation follows one scalar boundary $s$:

$$
s=0 \quad\longrightarrow\quad
\text{each table requires }b=s,\ e>b,\ \text{then sets }s=e.
$$

After the tables, require $s=F$. Apply the same rule to the first $L-1$ logs;
finally require the active log's base to equal $s$ and its end marker to be zero.
These are the loops in `validate_manifest`, not a separate proof left for the
reader to translate.

Here is the fixture's durable state progression. The temporary manifest names
consume identities too, even though rename removes those numbered names.

| Successful transition | Table coverage | Required logs | Next identity |
|---|---|---|---:|
| Create; then commit batches 1 and 2 | None | Log 1: base 0, active | 3 |
| Rotate | None | Log 1: `(0,2]`; log 3: base 2, active | 5 |
| Commit batch 3 | None | Same manifest; log 3 now contains batch 3 | 5 |
| Flush the oldest sealed log | Table 5: `(0,2]` | Log 3: base 2, active | 7 |

The final row advances $F$ from 0 to 2. It does not flush batch 3, and it does not
reset the active log's base to zero. The old log 1 can remain on disk without
remaining a recovery dependency. Its contents are irrelevant to normal reopen.

## Fix the manifest byte layout

This is a complete snapshot, not an append-only edit log. Every integer below is
unsigned little-endian. Counts describe the active prefixes of bounded arrays;
we never serialize C++ padding, unused array slots, or smart-pointer internals.

| Offset | Width | Field |
|---|---:|---|
| 0 | 8 | ASCII magic `STRAMAN1` |
| 8 | 4 | Format version, exactly 1 |
| 12 | 4 | Table count $T$, from 0 through 16 |
| 16 | 4 | Log count $L$, from 1 through 4 |
| 20 | 4 | Reserved, exactly zero |
| 24 | 8 | Manifest epoch, positive |
| 32 | 8 | Next file identity, strictly above every referenced identity |
| 40 | 8 | Last flushed sequence $F$ |
| 48 | $32T$ | Tables: identity, base, end, file length; four 64-bit words each |
| $48+32T$ | $24L$ | Logs: identity, base, end; three 64-bit words each |
| $48+32T+24L$ | 4 | CRC-32 over every preceding byte |

The total encoded length is

$$
N=48+32T+24L+4=52+32T+24L.
$$

An empty newly created database still needs one active log: $N=76$ bytes. Two
required logs use 100 bytes. One installed table and one active log use 108
bytes. At both caps, $N=52+512+96=660$ bytes. Counts are checked **before** their
products determine an allocation or a span. The database checks the actual file
length against this small limit before allocating its read buffer.

The epoch counts installed catalog transitions, not batches. Ordinary log writes
do not rewrite the manifest. The CRC detects accidental byte damage; semantic
validation detects a checksummed gap or duplicate identity. Neither authenticates
the file or proves that a storage device persisted it.

```cpp title="include/strata/storage/manifest.hpp"
#ifndef STRATA_STORAGE_MANIFEST_HPP
#define STRATA_STORAGE_MANIFEST_HPP

#include "strata/memory/budget.hpp"
#include "strata/storage/table.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <span>

namespace strata::storage {

struct TableRecord {
    std::uint64_t id = 0;
    std::uint64_t base = 0;
    std::uint64_t end = 0;
    std::uint64_t bytes = 0;
    bool operator==(const TableRecord&) const = default;
};
struct LogRecord {
    std::uint64_t id = 0;
    std::uint64_t base = 0;
    std::uint64_t end = 0;  // Zero only for the final, active log.
    bool operator==(const LogRecord&) const = default;
};
struct Manifest {
    static constexpr std::size_t max_tables = 16;
    static constexpr std::size_t max_logs = 4;
    static constexpr std::size_t header_bytes = 48;
    static constexpr std::size_t max_bytes =
        header_bytes + max_tables * 32 + max_logs * 24 + 4;
    std::uint64_t epoch = 0;
    std::uint64_t next_id = 1;
    std::uint64_t flushed = 0;
    std::size_t table_count = 0;
    std::size_t log_count = 0;
    std::array<TableRecord, max_tables> tables{};
    std::array<LogRecord, max_logs> logs{};
};
void validate_manifest(const Manifest& manifest);
memory::Vector<std::byte> encode_manifest(memory::Budget& budget,
                                         const Manifest& manifest);
Manifest decode_manifest(std::span<const std::byte> image);

}  // namespace strata::storage
#endif
```

`TableRecord` and `LogRecord` are value types with defaulted equality for checks.
The default member values produce an **uninitialized catalog**, not an acceptable
empty database: `epoch == 0` and `log_count == 0` must fail validation. Construction
will explicitly supply the initial active log. The arrays keep metadata bounded
without a dynamically growing catalog container.

`next_id` is a lower bound on future allocation. Gaps are harmless; reuse is not.
The validator also rejects a referenced identity equal to `next_id`. File kind
is not a second identity namespace: a log and table may not both claim the same
identity in one manifest.

Create `src/storage/manifest.cpp`. The small `require_manifest` helper maps
invalid persistent state to the already defined `FormatError`.

```cpp title="src/storage/manifest.cpp"
#include "strata/storage/manifest.hpp"
#include "strata/core/codec.hpp"

#include <algorithm>
#include <array>

namespace strata::storage {
namespace {
constexpr std::array<std::byte, 8> magic{
    std::byte{'S'}, std::byte{'T'}, std::byte{'R'}, std::byte{'A'},
    std::byte{'M'}, std::byte{'A'}, std::byte{'N'}, std::byte{'1'}};
void require_manifest(bool condition, const char* message) {
    if (!condition) { throw FormatError(message); }
}
}  // namespace
```

### Validate coverage without building another index

```cpp title="src/storage/manifest.cpp"
void validate_manifest(const Manifest& m) {
    require_manifest(m.epoch != 0 && m.next_id != 0, "manifest counters");
    require_manifest(m.table_count <= Manifest::max_tables && m.log_count >= 1 &&
                     m.log_count <= Manifest::max_logs, "manifest counts");
    std::array<std::uint64_t, Manifest::max_tables + Manifest::max_logs> ids{};
    std::size_t used = 0;
    const auto take_id = [&](std::uint64_t id) {
        require_manifest(id != 0 && id < m.next_id, "manifest file identity");
        for (std::size_t i = 0; i < used; ++i) {
            require_manifest(ids[i] != id, "duplicate manifest file identity");
        }
        ids[used++] = id;
    };
    std::uint64_t boundary = 0;
    for (std::size_t i = 0; i < m.table_count; ++i) {
        const auto& t = m.tables[i];
        take_id(t.id);
        require_manifest(t.base == boundary && t.end > t.base,
                         "table coverage is not a contiguous prefix");
        require_manifest(t.bytes >= table_format::header_bytes + table_format::footer_bytes &&
                         t.bytes <= table_format::max_file_bytes, "manifest table size");
        boundary = t.end;
    }
    require_manifest(boundary == m.flushed, "manifest flushed boundary");
    for (std::size_t i = 0; i < m.log_count; ++i) {
        const auto& log = m.logs[i];
        take_id(log.id);
        require_manifest(log.base == boundary, "log coverage gap or overlap");
        if (i + 1 == m.log_count) {
            require_manifest(log.end == 0, "active log must have open end");
        } else {
            require_manifest(log.end > log.base, "empty or reversed sealed log");
            boundary = log.end;
        }
    }
}
```

`take_id` captures the fixed `ids` array and its used length by reference. Calling
it checks one identity against earlier identities before appending it to that
stack array. At most twenty identities exist, so this bounded quadratic check
avoids an uncharged hash container and is not a database-size algorithm.

The table loop establishes the flushed prefix. The log loop deliberately does
not set `boundary` to zero for the final active log: zero is a marker, not the
next expected sequence. A missing sealed generation cannot be concealed by
letting a later base jump forward. Table file sizes are format-bounded here;
actual files and their contents are checked during opening.

### Encode a validated value, then decode a bounded image

The encoder allocates one charged workspace. Its `emit` lambda advances a byte
offset by eight after writing one word. Capturing `at` by reference is essential:
a fresh offset on each invocation would repeatedly overwrite the first field.
The vector remains alive while `out` borrows it and is not resized afterward.

```cpp title="src/storage/manifest.cpp"
memory::Vector<std::byte> encode_manifest(memory::Budget& budget, const Manifest& m) {
    validate_manifest(m);
    const auto size = Manifest::header_bytes + 32 * m.table_count + 24 * m.log_count + 4;
    memory::Vector<std::byte> image(size, memory::Allocator<std::byte>{budget});
    auto out = std::span<std::byte>{image};
    std::copy(magic.begin(), magic.end(), out.begin());
    put_le<4>(out.subspan(8, 4), 1);
    put_le<4>(out.subspan(12, 4), m.table_count);
    put_le<4>(out.subspan(16, 4), m.log_count);
    put_le<4>(out.subspan(20, 4), 0);
    put_le<8>(out.subspan(24, 8), m.epoch);
    put_le<8>(out.subspan(32, 8), m.next_id);
    put_le<8>(out.subspan(40, 8), m.flushed);
    std::size_t at = Manifest::header_bytes;
    const auto emit = [&](std::uint64_t value) {
        put_le<8>(out.subspan(at, 8), value);
        at += 8;
    };
    for (std::size_t i = 0; i < m.table_count; ++i) {
        const auto& t = m.tables[i];
        emit(t.id); emit(t.base); emit(t.end); emit(t.bytes);
    }
    for (std::size_t i = 0; i < m.log_count; ++i) {
        const auto& log = m.logs[i];
        emit(log.id); emit(log.base); emit(log.end);
    }
    put_le<4>(out.last(4), table_format::crc32(out.first(at)));
    return image;
}
```

For the 108-byte manifest after the first flush, the table words occupy bytes
48–79, the log words occupy 80–103, and the checksum occupies 104–107. Accordingly,
`at == 104` when the final `put_le<4>` runs. The zero-initialized vector and explicit
reserved-field write give a deterministic image, independent of structure layout.

The decoder first checks the fixed envelope, then the checksum, bounded counts,
and **exact** expected length. Only then can the `take` lambda advance through
variable-count records. A valid prefix followed by trailing garbage is not a
second supported format.

```cpp title="src/storage/manifest.cpp"
Manifest decode_manifest(std::span<const std::byte> image) {
    require_manifest(image.size() >= Manifest::header_bytes + 24 + 4 &&
                     image.size() <= Manifest::max_bytes, "manifest length");
    require_manifest(std::equal(magic.begin(), magic.end(), image.begin()), "manifest magic");
    require_manifest(get_le<4>(image.subspan(8, 4)) == 1 &&
                     get_le<4>(image.subspan(20, 4)) == 0, "manifest version/reserved");
    require_manifest(get_le<4>(image.last(4)) == table_format::crc32(image.first(image.size() - 4)),
                     "manifest checksum");
    Manifest m;
    m.table_count = static_cast<std::size_t>(get_le<4>(image.subspan(12, 4)));
    m.log_count = static_cast<std::size_t>(get_le<4>(image.subspan(16, 4)));
    require_manifest(m.table_count <= Manifest::max_tables && m.log_count >= 1 &&
                     m.log_count <= Manifest::max_logs, "manifest counts");
    require_manifest(image.size() == Manifest::header_bytes + 32 * m.table_count +
                     24 * m.log_count + 4, "manifest exact length");
    m.epoch = get_le<8>(image.subspan(24, 8));
    m.next_id = get_le<8>(image.subspan(32, 8));
    m.flushed = get_le<8>(image.subspan(40, 8));
    std::size_t at = Manifest::header_bytes;
    const auto take = [&]() {
        const auto value = get_le<8>(image.subspan(at, 8));
        at += 8;
        return value;
    };
    for (std::size_t i = 0; i < m.table_count; ++i) {
        auto& t = m.tables[i];
        t.id = take(); t.base = take(); t.end = take(); t.bytes = take();
    }
    for (std::size_t i = 0; i < m.log_count; ++i) {
        auto& log = m.logs[i];
        log.id = take(); log.base = take(); log.end = take();
    }
    validate_manifest(m);
    return m;
}

}  // namespace strata::storage
```

Returning a `Manifest` copies a bounded value. The encoded vector owns bytes;
the decoded value retains no span into them. The caller may release the workspace
immediately after decoding. The fixed arrays have unused slots, but only their
validated active prefixes influence opening and future serialization.

## Own the native directory and its lock

The existing `File` handles positioned data I/O through a `FileOps` backend.
Directory enumeration and advisory locking are native descriptor operations,
not simulated byte-file operations. Add a small native-only descriptor owner
rather than passing a fake `FileOps` handle to `flock` or `fdopendir`.

The OS [advisory-lock interface](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/flock.2.html)
provides `LOCK_EX | LOCK_NB`: acquire one exclusive lock or report contention
without waiting. It coordinates cooperating processes, not malicious writers.
The lock file is never unlinked during normal shutdown. Unlinking it while one
process still holds the old inode could let another process create a new inode
and acquire an unrelated lock under the same spelling.

The directory owner is declared before all database data-file owners. Reverse
member destruction therefore releases readers and logs first, then the lock and
directory descriptors. Construction failure uses those same member destructors;
there is no raw descriptor that waits for the failed complete object's destructor.

```cpp title="include/strata/io/directory.hpp"
#ifndef STRATA_IO_DIRECTORY_HPP
#define STRATA_IO_DIRECTORY_HPP

#include "strata/io/file.hpp"

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
```

`FileName` has 27 characters: two for `s-`, twenty decimal digits, four for the
extension, and one null terminator. This includes the complete unsigned 64-bit
domain without allocating a string. `Inventory` holds only counts and the largest
observed identity; it is not a collection of filenames.

`OpenedFile` transfers an ordinary `File` together with the length obtained from
that very opened descriptor. Checking a pathname and subsequently opening it
would otherwise separate the checked object from the used object. A trusted,
exclusively managed directory is still an explicit precondition.

```cpp title="include/strata/io/directory.hpp"
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
private:
    Inventory scan();
    Descriptor directory_;
    Descriptor lock_;  // Released after every Database child file has closed.
    Inventory inventory_;
};

}  // namespace strata::io
#endif
```

### Keep native error and ownership policy in one place

Create the implementation below. `openat` resolves a single filename relative to
the already opened directory, rather than rebuilding full paths throughout the
engine. `O_CLOEXEC`, `O_NOFOLLOW`, and `O_EXCL` respectively address exec leakage,
following a final symlink, and accidental reuse during creation. See the native
[open family](https://man7.org/linux/man-pages/man2/open.2.html) for these flags.
They are not a sandbox for an untrusted directory path; ancestor resolution and
hostile file replacement remain outside the contract.

```cpp title="src/io/directory.cpp"
#include "strata/io/directory.hpp"
#include "strata/core/codec.hpp"

#include <algorithm>
#include <cerrno>
#include <limits>
#include <memory>
#include <stdexcept>
#include <utility>

#include <dirent.h>
#include <fcntl.h>
#include <sys/file.h>
#include <sys/stat.h>
#include <unistd.h>

namespace strata::io {
namespace {
[[noreturn]] void native_error(const char* operation) {
    const int error = errno;
    throw IoError(error, operation, 0, 0);
}
void check_component(const char* name) {
    if (name == nullptr) { throw std::invalid_argument("null filename"); }
    const std::string_view text{name};
    if (text.empty() || text == "." || text == ".." || text.find('/') != text.npos) {
        throw std::invalid_argument("filename must be one directory component");
    }
}
void check_regular(const struct stat& info) {
    if (!S_ISREG(info.st_mode) || info.st_nlink != 1 || info.st_size < 0) {
        throw FormatError("engine file must be regular with one link");
    }
}
int open_relative(int directory, const char* name, int flags) {
    int result;
    do { result = ::openat(directory, name, flags | O_CLOEXEC | O_NOFOLLOW, 0600); }
    while (result < 0 && errno == EINTR);
    if (result < 0) { native_error("openat"); }
    return result;
}
}  // namespace
```

`native_error` copies `errno` before constructing an exception. `check_regular`
rejects directories, special files, negative lengths, and multiply linked engine
files. This also refuses a symlink discovered by `fstatat` without following it.
Foreign names are not opened or deleted by the scanner.

An interrupted open is retried. A close is attempted once: blindly retrying a
failed close can operate on a descriptor number reused by another operation.
These are the same ownership principles used by `File` in Chapter 6.

```cpp title="src/io/directory.cpp"
Descriptor::~Descriptor() noexcept {
    if (value_ >= 0) { (void)::close(value_); }
}
Descriptor::Descriptor(Descriptor&& other) noexcept : value_(other.release()) {}
Descriptor& Descriptor::operator=(Descriptor&& other) noexcept {
    if (this != &other) {
        if (value_ >= 0) { (void)::close(value_); }
        value_ = other.release();
    }
    return *this;
}
int Descriptor::release() noexcept { return std::exchange(value_, -1); }
```

The move constructor acquires `other.release()`; the source becomes `-1`.
Move assignment first releases its old descriptor and guards self-move. `release`
is a transfer operation, not a close. Its next use must immediately install the
transferred handle in another owner.

The bridge to `File` is intentionally small. In `include/strata/io/file.hpp`, add
this static declaration beside the existing `adopt` declaration:

```cpp title="include/strata/io/file.hpp — add native adoption"
    [[nodiscard]] static File adopt_native(int descriptor);
```

In `src/io/file.cpp`, add the following definition inside `strata::io`, after the
unnamed namespace containing `native_ops()` has closed. The existing `adopt`
validates nonnegative descriptors and installs ownership without allocating.
Only a real native descriptor may use this bridge; test backend handles continue
to use `adopt(the_backend, handle)`.

```cpp title="src/io/file.cpp — add native adoption"
File File::adopt_native(int descriptor) {
    return adopt(native_ops(), descriptor);
}
```

### Format and parse identities without wrapping

Append the filename generator. The loop writes decimal digits from right to
left. `FileName name{}` initializes the final null byte even though the extension
copy does not include a terminator.

```cpp title="src/io/directory.cpp"
FileName file_name(std::uint64_t id, FileKind kind) {
    if (id == 0) { throw std::invalid_argument("zero file identity"); }
    std::string_view suffix;
    switch (kind) {
    case FileKind::log: suffix = ".wal"; break;
    case FileKind::table: suffix = ".sst"; break;
    case FileKind::temporary: suffix = ".tmp"; break;
    default: throw std::invalid_argument("unknown file kind");
    }
    FileName name{};
    name[0] = 's'; name[1] = '-';
    for (std::size_t i = 22; i > 2;) {
        name[--i] = static_cast<char>('0' + id % 10);
        id /= 10;
    }
    std::copy(suffix.begin(), suffix.end(), name.begin() + 22);
    return name;
}
```

An identity of 12 becomes `s-00000000000000000012.wal`. The twenty digit slots are
always filled; the generator does not depend on locale or a platform's formatting
of integer types. It rejects zero rather than reserving an ambiguous all-zero
filename for the first log.

The parser ignores names outside `s-`, but rejects malformed names inside that
reserved namespace. For a current partial value $v$, digit $d$, and maximum
unsigned value $U$, require

$$
v\leq\left\lfloor\frac{U-d}{10}\right\rfloor
$$

before computing $10v+d$. This is Chapter 6's check-before-arithmetic rule at a
persistent identity boundary, not merely a printing detail.

```cpp title="src/io/directory.cpp"
std::optional<std::uint64_t> numbered_id(std::string_view name) {
    if (!name.starts_with("s-")) { return std::nullopt; }
    if (name.size() != 26 || (name.substr(22) != ".wal" &&
        name.substr(22) != ".sst" && name.substr(22) != ".tmp")) {
        throw FormatError("malformed name in reserved s- namespace");
    }
    std::uint64_t value = 0;
    for (std::size_t i = 2; i < 22; ++i) {
        if (name[i] < '0' || name[i] > '9') { throw FormatError("invalid file-id digit"); }
        const auto digit = static_cast<std::uint64_t>(name[i] - '0');
        if (value > (std::numeric_limits<std::uint64_t>::max() - digit) / 10) {
            throw FormatError("file identity overflow");
        }
        value = value * 10 + digit;
    }
    if (value == 0) { throw FormatError("zero numbered identity"); }
    return value;
}
```

`starts_with` is the C++20 counted-text operation on `string_view`. The view
borrows the current directory entry's name only until the next enumeration step.
No retained inventory field stores it. `substr(22)` is called only after the
length check's short-circuit expression establishes 26 characters.

### Acquire the lock before inspecting recovery state

```cpp title="src/io/directory.cpp"
Directory::Directory(const std::string& path) {
    if (path.empty() || path.find('\0') != std::string::npos) {
        throw std::invalid_argument("invalid database directory path");
    }
    int fd;
    do { fd = ::open(path.c_str(), O_RDONLY | O_DIRECTORY | O_CLOEXEC | O_NOFOLLOW); }
    while (fd < 0 && errno == EINTR);
    if (fd < 0) { native_error("open database directory"); }
    directory_ = Descriptor{fd};
    lock_ = Descriptor{open_relative(directory_.get(), "LOCK", O_RDWR | O_CREAT)};
    struct stat info{};
    if (::fstat(lock_.get(), &info) != 0) { native_error("stat LOCK"); }
    check_regular(info);
    int result;
    do { result = ::flock(lock_.get(), LOCK_EX | LOCK_NB); }
    while (result < 0 && errno == EINTR);
    if (result < 0) { native_error("lock database directory"); }
    inventory_ = scan();
}
```

The constructor rejects embedded nulls in the supplied path, opens the directory,
opens or creates the permanent `LOCK` file, validates it, and acquires the lock
before scanning. A competing process fails at `flock`; it does not parse a
manifest while another process changes it. A failed constructor closes every
already acquired descriptor through its fully constructed members.

There is no dependency on `std::filesystem::directory_iterator` or a vector of
paths. The scan opens a new description of `.` relative to the directory. A
simple `dup` would share directory-offset state; the independent open gives this
stream its own enumeration position.

`fdopendir` takes control of its descriptor **only on success**. Until then,
`scan_fd` owns it. After success, a `unique_ptr` with a local deleter closes the
stream, which closes its descriptor. The [directory-stream interface](https://man7.org/linux/man-pages/man3/fdopendir.3.html)
documents that ownership transfer.

```cpp title="src/io/directory.cpp"
Inventory Directory::scan() {
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

`readdir` returns null for both end-of-directory and error. Setting `errno` to zero
before each call distinguishes those cases. Closing the stream in a nonthrowing
deleter does not replace this explicit read-error check.

The cap of 4096 counts all non-dot entries, including foreign names. It bounds
enumeration work and prevents normal operation from silently accumulating
arbitrarily many preserved orphans. Exceeding it fails open; it does not trigger
unsafe automatic deletion. The C library's directory-stream bookkeeping is
outside the engine allocation ledger, as are OS descriptor tables; the scanner
itself retains no data-sized container.

### Create exclusively, synchronize explicitly, replace only authority

```cpp title="src/io/directory.cpp"
OpenedFile Directory::open(const char* name, OpenMode mode) {
    check_component(name);
    int flags;
    switch (mode) {
    case OpenMode::read_only: flags = O_RDONLY; break;
    case OpenMode::read_write: flags = O_RDWR; break;
    case OpenMode::create_new:
        if (inventory_.entries == max_entries) { throw std::length_error("directory entry cap"); }
        flags = O_RDWR | O_CREAT | O_EXCL;
        break;
    default: throw std::invalid_argument("unknown open mode");
    }
    Descriptor fd{open_relative(directory_.get(), name, flags)};
    if (mode == OpenMode::create_new) { ++inventory_.entries; }
    struct stat info{};
    if (::fstat(fd.get(), &info) != 0) { native_error("stat opened file"); }
    check_regular(info);
    return {File::adopt_native(fd.release()), static_cast<std::uint64_t>(info.st_size)};
}
```

`open` records a successful new name in its local entry count before any later
validation can throw. Once the descriptor passes `fstat`, `release` transfers it
into `File::adopt_native`. The returned owner now uses the established short-I/O,
interruption, synchronization, and close policies; the directory layer does not
implement a second set of transfer loops.

Append the two namespace operations. Only `MANIFEST` is replaced. A candidate
log or table is created at a fresh identity and is never overwritten in place.

```cpp title="src/io/directory.cpp"
void Directory::sync() {
    int result;
    do { result = ::fsync(directory_.get()); } while (result < 0 && errno == EINTR);
    if (result < 0) { native_error("sync database directory"); }
}
void Directory::replace_manifest(const char* temporary) {
    check_component(temporary);
    if (::renameat(directory_.get(), temporary, directory_.get(), "MANIFEST") != 0) {
        native_error("replace MANIFEST");
    }
    if (inventory_.manifest) { --inventory_.entries; }
    inventory_.manifest = true;
}

}  // namespace strata::io
```

A successful [rename](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/rename.2.html)
changes the destination binding atomically for ordinary pathname lookup. That is
not a claim about which binding survives power loss. The Linux
[fsync documentation](https://man7.org/linux/man-pages/man2/fsync.2.html) explicitly
separates file synchronization from synchronization of its containing directory.
Our protocol therefore synchronizes data and names at different steps, and never
treats an unsupported directory sync as success.

`replace_manifest` does not retry an ambiguous rename failure or remove the
source candidate afterward. The caller will stop using the database and reopen
its actual authority. When an existing manifest is replaced, two names become
one, so the tracked directory entry count falls by one. Initial installation
renames one candidate name to one authority name and does not change that count.

The directory code is infrastructure, not the persistence algorithm. In
[Part 2](https://25d-228.github.io/cplusplus-learning/chapter-11-part-2.html), the
ordering of these operations will make a new log usable and a sealed log
replaceable. A standalone call to `replace_manifest` has no such guarantee.
