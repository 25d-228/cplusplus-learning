# Chapter 6 — Records, bytes, and a dependable file boundary

## Part 1 — Encodings and file ownership

A row can be correct in memory and meaningless on disk. A write can return
success after transferring only a prefix. An exception can release every file
handle without restoring the bytes that existed before the operation. These
are three different problems: representation, completion, and failure semantics.

We will give Strata explicit record encodings and a move-only file owner. The
owner will keep requesting the untransferred suffix, distinguish end-of-file
from an operating-system error, and report synchronization failures explicitly.
A deterministic backend will make short transfers and failure after four bytes
ordinary test cases instead of rare accidents.

Start from the restored workspace of
[Chapter 5](https://25d-228.github.io/cplusplus-learning/chapter-05.html), retaining
all six existing CTest entries. Chapters 2–5 supply spans, value types, RAII,
move operations, exceptions, optional values, and templates. No earlier exercise
`.cpp` is included in the engine. Every new helper needed here is defined below.

> **Learning objectives**
>
> Separate object layout from a byte format; encode unsigned fields without
> alignment assumptions; reject invalid lengths and arithmetic overflow; use a
> small virtual interface for deterministic I/O; transfer descriptor ownership;
> handle short and interrupted operations; distinguish cleanup, synchronization,
> and rollback; inspect a binary fixture with an independent decoder.

The allocation remains **3.0 implementation hours**: 1.0 for encoding, 1.25 for
file ownership and transfer loops, and 0.75 for fault checks and debugging.
Reading is separate. This is the first reusable engine layer, not a database,
write-ahead log, or claim of power-loss recovery.

This chapter has two linked parts. Part 1 supplies the engine interfaces and
implementation; [Part 2](https://25d-228.github.io/cplusplus-learning/chapter-06-part-2.html)
supplies the complete deterministic backend, tests, build additions, and byte
oracle. Together they form one chapter with the single 3.0-hour allocation above.
Read and implement both before advancing.

## Begin the engine without disturbing the exercises

Use the compatible native C++20 toolchain selected in Chapter 1. On the Apple
compiler path:

```sh title="Terminal — prepare the engine directories"
CXX="$(xcrun --find clang++)"
mkdir -p include/strata/core include/strata/io src/io tests scripts
```

Write these files in the learner workspace:

| File | Responsibility |
|---|---|
| `include/strata/core/record.hpp` | Numeric row and key value types |
| `include/strata/core/checked.hpp` | Size, total, and file-range arithmetic |
| `include/strata/core/codec.hpp` | Byte encoders, decoders, and format errors |
| `include/strata/io/file.hpp` | File owner, operation interface, I/O error context |
| `src/io/file.cpp` | Native adapter, ownership, and transfer loops |
| `tests/memory_file_ops.hpp` | Fixed-capacity deterministic I/O backend |
| `tests/core_io.cpp` | Checks and native fixture writer |
| `scripts/check_record_fixture.py` | Independent byte oracle and isolated native test |

Blocks with the same filename append **in reading order**. The CMake block
appends to the existing root file. Headers close their guards in their last
block; the source files close their namespaces where shown. Do not create
future storage, log, or query files yet.

The native layer requires regular seekable files, `pread`, `pwrite`, `open`,
`fstat`, `fsync`, `close`, and `O_CLOEXEC`. It checks for eight-bit bytes and a
signed file-offset type with at least 63 value bits. These are actual interface
requirements, not a required compiler banner or an assumed macOS release.
The native `full_device` synchronization path additionally requires the Apple
`F_FULLFSYNC` operation; unsupported builds return an explicit error.

## A row is a value; its encoding is another value

The two schemas remain the ones in the roadmap. Every integer field accepts
its entire unsigned range, including zero. A day is a numeric label; cents
are nonnegative integers. There is no floating-point amount or date parser.

`Relation` is a scoped enumeration with an explicitly chosen underlying type.
The values 1 and 2 are format decisions. `Key` groups relation and primary key;
`KeyLess` orders the relation first and the primary key second, without
subtracting unsigned integers. It therefore works near both ends of the range.

`bool operator==(const Event&) const = default;` asks C++20 to generate memberwise
equality. It compares **values**, not padding bytes. The trailing `const` means
the comparison does not mutate its left operand. See the language rules for
[enumerations](https://eel.is/c++draft/dcl.enum) and
[defaulted comparisons](https://eel.is/c++draft/class.compare.default).

```cpp title="include/strata/core/record.hpp"
#ifndef STRATA_CORE_RECORD_HPP
#define STRATA_CORE_RECORD_HPP

#include <cstdint>

namespace strata {

enum class Relation : std::uint8_t { events = 1, accounts = 2 };

struct Key {
    Relation relation;
    std::uint64_t primary;
    bool operator==(const Key&) const = default;
};

struct KeyLess {
    bool operator()(const Key& a, const Key& b) const noexcept {
        if (a.relation != b.relation) {
            return a.relation < b.relation;
        }
        return a.primary < b.primary;
    }
};

struct Event {
    std::uint64_t event_id;
    std::uint64_t account_id;
    std::uint32_t day;
    std::uint64_t amount_cents;
    bool operator==(const Event&) const = default;
};

struct Account {
    std::uint64_t account_id;
    std::uint32_t region_id;
    bool operator==(const Account&) const = default;
};

inline Key key_of(const Event& event) noexcept {
    return {Relation::events, event.event_id};
}

inline Key key_of(const Account& account) noexcept {
    return {Relation::accounts, account.account_id};
}

}  // namespace strata
#endif
```

The two `key_of` overloads are selected by their parameter type. An event with
identifier 10 and an account with identifier 10 have different keys. The
comparator places all event keys before account keys because the relation tags
are ordered that way. This is an engine policy, not an inference from C++ member
addresses. Later internal keys add sequence ordering separately.

A compiler may insert padding between members to satisfy alignment requirements.
For example, placing a 64-bit member after a 32-bit member can require unused
space. The actual `sizeof(Event)` is a property of the selected implementation,
not the number of bytes in our record. The C++ distinction between
[object and value representation](https://eel.is/c++draft/basic.types) explains
why copying an object's representation is not a portable persistent format.

We will not pack the structure or cast a disk byte pointer to `Event*`. Such a
cast does not validate size, establish a suitably aligned object, correct byte
order, or interpret a format version. Reading one byte at a time avoids all of
those typed-access assumptions. Decoding constructs an ordinary aligned C++
value after validating the bytes.

### Fix the byte layout before writing a loop

A record begins with version 1 and a relation tag, each one byte. All multi-byte
integers use **little-endian** order: the least significant byte comes first.
The common identifier is followed by that relation's fields. No padding bytes
are present.

| Event byte offsets | Width | Meaning |
|---|---:|---|
| 0 | 1 | Version, exactly 1 |
| 1 | 1 | Relation, exactly 1 |
| 2–9 | 8 | `event_id` |
| 10–17 | 8 | `account_id` |
| 18–21 | 4 | `day` |
| 22–29 | 8 | `amount_cents` |

| Account byte offsets | Width | Meaning |
|---|---:|---|
| 0 | 1 | Version, exactly 1 |
| 1 | 1 | Relation, exactly 2 |
| 2–9 | 8 | `account_id` |
| 10–13 | 4 | `region_id` |

Thus the sizes are $1+1+8+8+4+8=30$ and $1+1+8+4=14$ bytes. The decoder requires
exactly one such image: trailing bytes are not silently accepted as part of a
record. A higher-level table or log will provide its own framing, operation
kind, sequence information, and checksums. Those mechanisms are not hidden in
this codec. In particular, a bit change that produces another valid integer
cannot be detected by these row decoders alone.

Row images include the primary key. Storage metadata introduced later must
agree with that key; a deletion will not be represented by inventing an all-zero
row. The fixed two-record file at the end of this chapter is a fixture with known
boundaries, not a general parser for a concatenated database.

## Check arithmetic before allocating or touching a file

Suppose a size type has maximum value $M$, and $a$ and $b$ are nonnegative
representable sizes. Checking `a + b > M` is too late: unsigned addition may
already have wrapped. Rearrange the condition to $b \leq M-a$ before adding.
For a product of count $n$ and width $w>0$, require $n \leq \lfloor M/w \rfloor$.
When $w=0$, the product is zero and no division is needed.

Create the first part of the arithmetic header:

```cpp title="include/strata/core/checked.hpp"
#ifndef STRATA_CORE_CHECKED_HPP
#define STRATA_CORE_CHECKED_HPP

#include <cstddef>
#include <cstdint>
#include <limits>
#include <stdexcept>

namespace strata {

inline std::size_t checked_size_add(std::size_t a, std::size_t b) {
    if (b > std::numeric_limits<std::size_t>::max() - a) {
        throw std::length_error("size addition overflow");
    }
    return a + b;
}

inline std::size_t checked_size_mul(std::size_t count, std::size_t width) {
    if (width != 0 && count > std::numeric_limits<std::size_t>::max() / width) {
        throw std::length_error("size multiplication overflow");
    }
    return count * width;
}

inline std::uint64_t checked_u64_add(std::uint64_t a, std::uint64_t b) {
    if (b > std::numeric_limits<std::uint64_t>::max() - a) {
        throw std::overflow_error("unsigned total overflow");
    }
    return a + b;
}
```

`checked_size_mul(3, 30)` returns 90. With an illustrative maximum of 255, a count
of 9 at width 30 would be rejected because $9>\lfloor255/30\rfloor=8$. The actual
code uses the maximum of `size_t`, not that illustrative limit. The `&&` condition
short-circuits, so division by zero never occurs.

The sum helper for `uint64_t` serves numerical totals, not allocation lengths.
It uses `overflow_error` rather than treating a wrapped amount as a valid value.
The size helpers use `length_error`. All checks happen before the arithmetic
whose result they protect, and none needs a huge allocation to test its limit.

### An offset is not a pointer and not necessarily a size

The file API accepts unsigned 64-bit offsets but deliberately restricts the
**exclusive end** of each request to $L=2^{63}-1$. For starting offset $o$ and
byte count $n$, require $o\leq L$ and $n\leq L-o$ before computing $o+n$.
This fits a signed 64-bit native offset. The stricter exclusive-end convention
means the largest permitted nonempty byte index is $L-1$.

```cpp title="include/strata/core/checked.hpp"
inline constexpr std::uint64_t file_offset_limit =
    static_cast<std::uint64_t>(std::numeric_limits<std::int64_t>::max());
static_assert(sizeof(std::size_t) <= sizeof(std::uint64_t));

inline std::uint64_t checked_file_end(std::uint64_t offset, std::size_t count) {
    const auto length = static_cast<std::uint64_t>(count);
    if (offset > file_offset_limit || length > file_offset_limit - offset) {
        throw std::length_error("file range exceeds signed 64-bit limit");
    }
    return offset + length;
}

}  // namespace strata
#endif
```

For $o=L-3$ and $n=3$, the exclusive end is exactly $L$ and the request is
representable. At $o=L$, only an empty request is permitted. An offset of
`UINT64_MAX` is rejected even for zero bytes. The first `||` operand protects
`L - offset` from underflow.

The `size_t` width assertion makes conversion to `uint64_t` lossless on the
supported targets. `static_cast` expresses a conversion; it does **not** check
that arbitrary values fit. The surrounding bounds are the proof. Rejecting a
request at this boundary is a no-side-effect error. A later operating-system
failure has a different guarantee.

## Turn an unsigned integer into bytes

`std::byte` denotes a storage byte without making ordinary numeric arithmetic
implicit. Use `std::to_integer<T>` to obtain a number and `static_cast<std::byte>`
to store a checked byte-sized value. The relevant library operations are defined
in [byte operations](https://eel.is/c++draft/support.types.byteops); explicit
conversions are specified by [static casts](https://eel.is/c++draft/expr.static.cast).

A `FormatError` describes malformed external bytes. The public inheritance below
means it can be caught either by its own type or by `std::runtime_error` and
`std::exception`. Its constructor delegates message storage to its base class;
it does not retain the supplied `const char*` as our own borrowed member.

```cpp title="include/strata/core/codec.hpp"
#ifndef STRATA_CORE_CODEC_HPP
#define STRATA_CORE_CODEC_HPP

#include "strata/core/record.hpp"

#include <array>
#include <climits>
#include <cstddef>
#include <cstdint>
#include <limits>
#include <span>
#include <stdexcept>

namespace strata {

static_assert(CHAR_BIT == 8, "Strata's format uses 8-bit bytes");

class FormatError : public std::runtime_error {
public:
    explicit FormatError(const char* message) : std::runtime_error(message) {}
};
```

Let $W$ be the integer width in bytes and $x$ its value. Byte $i$, for
$0\leq i<W$, is

$$
b_i = \left\lfloor \frac{x}{2^{8i}} \right\rfloor \bmod 256.
$$

Decoding reverses this expansion:

$$
x=\sum_{i=0}^{W-1} b_i\,2^{8i}.
$$

For 500, the first byte is $500\bmod256=244$, hexadecimal `f4`. The next is
$\lfloor500/256\rfloor=1$. An eight-byte encoding is therefore
`f4 01 00 00 00 00 00 00`. For `0x0102030405060708`, it is
`08 07 06 05 04 03 02 01`.

Here `Width` is a **value template parameter**, unlike Chapter 5's element type
parameter. Only widths 4 and 8 satisfy the constraint. `if constexpr` chooses
its branch at instantiation: only the four-byte encoder needs to reject values
above `UINT32_MAX`. It is not an instruction to execute the whole function at
compile time. See [constexpr if](https://eel.is/c++draft/stmt.if).

```cpp title="include/strata/core/codec.hpp"
template<std::size_t Width>
requires (Width == 4 || Width == 8)
void put_le(std::span<std::byte> output, std::uint64_t value) {
    if (output.size() != Width) {
        throw std::invalid_argument("wrong integer output width");
    }
    if constexpr (Width == 4) {
        if (value > std::numeric_limits<std::uint32_t>::max()) {
            throw std::overflow_error("value does not fit four bytes");
        }
    }
    for (std::size_t i = 0; i < Width; ++i) {
        output[i] = static_cast<std::byte>((value >> (8 * i)) & 0xffU);
    }
}

template<std::size_t Width>
requires (Width == 4 || Width == 8)
std::uint64_t get_le(std::span<const std::byte> input) {
    if (input.size() != Width) {
        throw FormatError("wrong integer input width");
    }
    std::uint64_t value = 0;
    for (std::size_t i = 0; i < Width; ++i) {
        value |= std::to_integer<std::uint64_t>(input[i]) << (8 * i);
    }
    return value;
}
```

The encoder checks both destination length and value width **before** writing
any byte. The mask `0xffU` keeps the low eight bits after shifting. The decoder
converts each byte to `uint64_t` **before** the left shift: a large shift of a
narrow promoted integer would not be repaired by widening its result afterward.
For the largest width, shift amounts are 0, 8, ..., 56, never 64. This satisfies
the [shift-operator requirements](https://eel.is/c++draft/expr.shift).

`|=` combines disjoint groups of bits; addition would yield the same result here
because those groups do not overlap. The exact-span check precedes every index
access. No pointer is reinterpreted, no unaligned 64-bit load is needed, and the
host's byte order does not affect the output.

Do not compare little-endian byte strings to order numeric keys. The first byte
of 255 is `ff`, while the first byte of 256 is `00`. Lexicographic byte comparison
would reverse their numeric order. `KeyLess` operates on decoded values.

## Validate a complete image before decoding its fields

The helper below checks length before touching either header byte. It then
checks version and the expected relation. It does not cast an arbitrary input
tag into a trusted enum and continue. The event decoder cannot accept an
account tag simply because the rest of a padded input has enough bytes.

```cpp title="include/strata/core/codec.hpp"
inline constexpr std::size_t event_bytes = 30;
inline constexpr std::size_t account_bytes = 14;
inline constexpr std::byte record_version{1};

inline void require_record(std::span<const std::byte> input,
                           std::size_t size, Relation relation) {
    if (size < 2 || input.size() != size) {
        throw FormatError("record has wrong byte count");
    }
    if (input[0] != record_version) {
        throw FormatError("unsupported record version");
    }
    const auto tag = std::to_integer<std::uint8_t>(input[1]);
    if (tag != static_cast<std::uint8_t>(relation)) {
        throw FormatError("wrong or unknown relation tag");
    }
}
```

The `size < 2` guard also prevents accidental misuse of this helper with a
header shorter than its two inspected bytes. Its actual callers pass 30 and 14.
An exact input span may safely be subdivided at the offsets in the layout table.
The event encoder returns an owning array; the decoder returns an owning value.
Neither retains a view into its argument.

```cpp title="include/strata/core/codec.hpp"
inline std::array<std::byte, event_bytes> encode_event(const Event& event) {
    std::array<std::byte, event_bytes> result{};
    std::span<std::byte> out{result};
    out[0] = record_version;
    out[1] = static_cast<std::byte>(Relation::events);
    put_le<8>(out.subspan(2, 8), event.event_id);
    put_le<8>(out.subspan(10, 8), event.account_id);
    put_le<4>(out.subspan(18, 4), event.day);
    put_le<8>(out.subspan(22, 8), event.amount_cents);
    return result;
}

inline Event decode_event(std::span<const std::byte> input) {
    require_record(input, event_bytes, Relation::events);
    return {get_le<8>(input.subspan(2, 8)),
            get_le<8>(input.subspan(10, 8)),
            static_cast<std::uint32_t>(get_le<4>(input.subspan(18, 4))),
            get_le<8>(input.subspan(22, 8))};
}
```

The four-byte `get_le` result is at most `UINT32_MAX`, which justifies the cast
for `day`. The eight-byte fields need no narrowing. All fields are decoded into
the returned value; there is no partially overwritten caller-owned record if a
format check fails. Account encoding follows the same header and integer rules:

```cpp title="include/strata/core/codec.hpp"
inline std::array<std::byte, account_bytes> encode_account(const Account& account) {
    std::array<std::byte, account_bytes> result{};
    std::span<std::byte> out{result};
    out[0] = record_version;
    out[1] = static_cast<std::byte>(Relation::accounts);
    put_le<8>(out.subspan(2, 8), account.account_id);
    put_le<4>(out.subspan(10, 4), account.region_id);
    return result;
}

inline Account decode_account(std::span<const std::byte> input) {
    require_record(input, account_bytes, Relation::accounts);
    return {get_le<8>(input.subspan(2, 8)),
            static_cast<std::uint32_t>(get_le<4>(input.subspan(10, 4)))};
}

}  // namespace strata
#endif
```

Arrays, spans, and return values do different jobs here. The local array owns
exactly 30 or 14 bytes. The mutable span gives the integer helper access to a
selected range without transferring ownership. The returned array is a value,
not a span into a dead local. Decoding borrows only until the function returns.

## Give the transfer loop a narrow operation interface

A real file descriptor is an integer identifying an operating-system resource.
Copying that integer does not duplicate ownership safely. We need one owner for
the descriptor and a way to test what its operations do when they return short
counts or errors.

`FileOps` separates **one attempted operation** from **completion policy**.
The native implementation makes one system call. The test implementation uses
a small array. Both report a count and error code; `File` performs retries and
accumulates progress. Opening paths, creating directories, renaming, and removing
names are not part of this four-operation interface.

This introduces runtime polymorphism. A `virtual` member called through a base
reference or pointer dispatches to the implementation in the actual derived
object. `= 0` makes a member pure virtual: `FileOps` cannot be instantiated on its
own. `public FileOps` exposes the derived object as that interface; `override`
lets the compiler check the intended override; `final` later prevents further
derivation from the concrete adapter. These are the language rules for
[virtual functions](https://eel.is/c++draft/class.virtual) and
[abstract classes](https://eel.is/c++draft/class.abstract).

The base destructor is virtual so destruction through a suitable owning base
pointer would destroy the complete derived object. Our `File` will **borrow**
this interface, not delete it; virtual destruction does not extend its lifetime.

```cpp title="include/strata/io/file.hpp"
#ifndef STRATA_IO_FILE_HPP
#define STRATA_IO_FILE_HPP

#include <cstddef>
#include <cstdint>
#include <span>
#include <string>
#include <system_error>

namespace strata::io {

enum class SyncMode { file, full_device };
enum class OpenMode { read_only, read_write, create_new };

struct Transfer {
    std::size_t count;
    int error;
};

class FileOps {
public:
    virtual ~FileOps() = default;
    virtual Transfer read_at(int descriptor, std::span<std::byte> output,
                             std::uint64_t offset) noexcept = 0;
    virtual Transfer write_at(int descriptor, std::span<const std::byte> input,
                              std::uint64_t offset) noexcept = 0;
    virtual int sync(int descriptor, SyncMode mode) noexcept = 0;
    virtual int close(int descriptor) noexcept = 0;
};
```

The interface contract is small but essential:

| Result from one transfer | Meaning to `File` |
|---|---|
| `error == 0`, `count > 0` | That many bytes completed, no more than requested |
| `error == 0`, `count == 0` on read | End-of-file for this regular-file operation |
| `error == 0`, `count == 0` on nonempty write | No progress; treated as failure, not retried forever |
| Positive `error`, zero `count` | Failed attempt; `EINTR` is retried, other errors propagate |

The interface does not return both a partial count and an error in the same
result. A partial successful call followed by a failing call produces two
results. The native adapter captures `errno` only on failure, before another
operation can overwrite it. A successful call does not need to clear `errno`.
`sync` and `close` return zero on success or a positive error number, **not** the
native `-1` sentinel.

All backend operations are `noexcept`: they report errors as values and do not
allocate. This says nothing about whether a system call can block. Both backends
must respect the borrowed buffer bounds. The operations interface is an internal
contract, not a sandbox for arbitrary hostile subclasses.

### Carry failure context without confusing it with rollback

`IoError` adds the original requested offset and the number of bytes reported
successful by earlier iterations. `std::system_error` already owns an error code
and message; its [members](https://eel.is/c++draft/syserr.syserr.members) expose
`code()` and `what()`. The generic error category interprets the `errno` numbers
used by these backends. Exact message wording is platform-dependent.

```cpp title="include/strata/io/file.hpp"
class IoError : public std::system_error {
public:
    IoError(int error, const char* operation, std::uint64_t offset,
            std::size_t transferred)
        : std::system_error(error, std::generic_category(), operation),
          offset_(offset), transferred_(transferred) {}
    std::uint64_t offset() const noexcept { return offset_; }
    std::size_t transferred() const noexcept { return transferred_; }
private:
    std::uint64_t offset_;
    std::size_t transferred_;
};

inline constexpr std::size_t io_chunk_bytes = 1024 * 1024;
```

For a write starting at offset 5 that fails after four reported bytes, the error
has `offset() == 5` and `transferred() == 4`. That is observed progress, not a
promise that precisely four bytes reached persistent storage. The failing call
and storage stack may have further uncertain effects. A transaction layer must
not turn this number into an automatic whole-batch retry policy.

We cap each native request at one mebibyte. This bounds the count passed to an
individual system call; it does **not** allocate a buffer or bound the caller's
whole data set. The file layer borrows the caller's existing span.

### Declare one owner of the descriptor

The invariant is open: nonnegative descriptor and a valid backend pointer;
closed: descriptor `-1` and null backend. `adopt` transfers responsibility for an
already-valid handle with a matching backend. The caller must not close it or
adopt it again. The backend must outlive the file, including after moves. Only
the native factory and the controlled test fixture use this entry point here.

```cpp title="include/strata/io/file.hpp"
class File {
public:
    File() noexcept = default;
    ~File() noexcept;
    File(const File&) = delete;
    File& operator=(const File&) = delete;
    File(File&& other) noexcept;
    File& operator=(File&& other) noexcept;

    [[nodiscard]] static File open(const std::string& path, OpenMode mode);
    [[nodiscard]] static File adopt(FileOps& operations, int descriptor);
    bool is_open() const noexcept { return descriptor_ >= 0; }

    std::size_t read_up_to(std::uint64_t offset, std::span<std::byte> output);
    void read_exact(std::uint64_t offset, std::span<std::byte> output);
    void write_all(std::uint64_t offset, std::span<const std::byte> input);
    void sync(SyncMode mode);
    void close_checked();

private:
    File(FileOps& operations, int descriptor) noexcept
        : operations_(&operations), descriptor_(descriptor) {}
    void require_open() const;
    int close_once() noexcept;
    FileOps* operations_ = nullptr;
    int descriptor_ = -1;
};

}  // namespace strata::io
#endif
```

`static` factories belong to the class but need no existing instance. The private
constructor installs a verified handle without allocating. Default construction
produces an empty owner. Copying is deleted; moves are nonthrowing. Moving or
closing the owner concurrently with an operation is forbidden. Explicit offsets
remove dependence on a shared seek position, not the need for lifetime and
higher-level synchronization rules.

`read_up_to` fills the requested span unless it reaches end-of-file, returning
the completed count. `read_exact` requires the full count and otherwise throws
`FormatError`. `write_all` either completes the requested transfer or throws;
it does not undo a prefix. `sync` reports a requested synchronization result.
`close_checked` reports close failure and consumes this owner's responsibility.
The destructor performs only a best-effort, nonthrowing close, never a hidden
synchronization. Error-reporting code itself may allocate; only the explicitly
nonthrowing cleanup and backend methods promise not to propagate exceptions.

## Translate native calls, not their completion policy

Create `src/io/file.cpp`. The unnamed namespace gives the adapter and helper
internal linkage: other translation units use `File`, not `NativeOps`. The
leading `::` in `::pread` selects the global operating-system function rather
than a member with a similar name.

`pread` and `pwrite` accept an explicit offset and leave the shared file position
unchanged. Their return type, `ssize_t`, can represent either a nonnegative
count or `-1`. Convert to `size_t` **after** checking for a negative result.
See Apple's [read interface](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/pread.2.html)
and [write interface](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/pwrite.2.html).

```cpp title="src/io/file.cpp"
#include "strata/io/file.hpp"
#include "strata/core/checked.hpp"
#include "strata/core/codec.hpp"

#include <algorithm>
#include <cerrno>
#include <limits>
#include <stdexcept>
#include <utility>

#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

namespace strata::io {
namespace {

static_assert(std::numeric_limits<off_t>::is_signed);
static_assert(std::numeric_limits<off_t>::digits >= 63);
static_assert(std::numeric_limits<ssize_t>::is_signed);
static_assert(std::numeric_limits<ssize_t>::digits >= 21);

class NativeOps final : public FileOps {
public:
    Transfer read_at(int descriptor, std::span<std::byte> output,
                     std::uint64_t offset) noexcept override {
        const ssize_t result = ::pread(descriptor, output.data(), output.size(),
                                       static_cast<off_t>(offset));
        if (result < 0) {
            const int error = errno;
            return {0, error};
        }
        return {static_cast<std::size_t>(result), 0};
    }

    Transfer write_at(int descriptor, std::span<const std::byte> input,
                      std::uint64_t offset) noexcept override {
        const ssize_t result = ::pwrite(descriptor, input.data(), input.size(),
                                        static_cast<off_t>(offset));
        if (result < 0) {
            const int error = errno;
            return {0, error};
        }
        return {static_cast<std::size_t>(result), 0};
    }
```

`File` will validate the whole offset range and cap each request before invoking
these methods. Thus the `off_t` cast cannot narrow a valid offset, and a returned
count fits `ssize_t`. The 21-value-bit assertion is enough for a positive count
of 1,048,576. The native methods do not retry: policy belongs in exactly one
place, where the accumulated byte count is known.

### Synchronization is an explicit requested capability

For `SyncMode::file`, call `fsync`. For `full_device` on the Apple path, request
`F_FULLFSYNC`. If the latter operation is not available at build time or fails at
runtime, report that fact; do not silently substitute a weaker mode. Apple
explains the distinction between moving data to a drive cache and requesting
its flush in its [fsync manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
and [storage discussion](https://developer.apple.com/videos/play/wwdc2019/419/).
A successful request is still not our proof of physical power-loss behavior.

Append the rest of the adapter and the internal result validator. This closes
`NativeOps` and the unnamed namespace, but keeps `strata::io` open:

```cpp title="src/io/file.cpp"
    int sync(int descriptor, SyncMode mode) noexcept override {
        if (mode == SyncMode::file) {
            if (::fsync(descriptor) == 0) {
                return 0;
            }
            return errno;
        }
        if (mode != SyncMode::full_device) {
            return EINVAL;
        }
#if defined(__APPLE__) && defined(F_FULLFSYNC)
        if (::fcntl(descriptor, F_FULLFSYNC) == 0) {
            return 0;
        }
        return errno;
#else
        return ENOTSUP;
#endif
    }

    int close(int descriptor) noexcept override {
        if (::close(descriptor) == 0) {
            return 0;
        }
        return errno;
    }
};

FileOps& native_ops() noexcept {
    static NativeOps operations;
    return operations;
}

void require_transfer(Transfer result, std::size_t requested) {
    if (result.count > requested || result.error < 0 ||
        (result.error != 0 && result.count != 0)) {
        throw std::logic_error("invalid FileOps transfer result");
    }
}

}  // namespace
```

The `#if` selects a native implementation at compilation; it is not an attempt
to infer the learner's OS version. `ENOTSUP` explicitly distinguishes unsupported
full-device synchronization from success. The archived Apple
[`fcntl` documentation](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fcntl.2.html)
describes the request; its historical filesystem list is not a compatibility
matrix for a current machine. Check the selected SDK headers, local manual,
and actual call result.

`native_ops()` returns a reference to one function-local static adapter, rather
than creating a dangling temporary backend for each file. Files must not outlive
that adapter; ordinary function-scoped files satisfy this. We do not introduce
global file owners with ambiguous program-termination ordering.

`require_transfer` catches impossible backend responses before they corrupt the
loop's count, such as reporting more bytes than requested. This is a programming
error (`logic_error`), not malformed record data. It does not validate whether
a malicious backend actually respected the supplied memory range.

## Move the handle and close it exactly once at the owner boundary

Append adoption, destruction, move operations, and the two forms of close:

```cpp title="src/io/file.cpp"
File File::adopt(FileOps& operations, int descriptor) {
    if (descriptor < 0) {
        throw std::invalid_argument("cannot adopt a negative descriptor");
    }
    return File(operations, descriptor);
}

File::~File() noexcept {
    (void)close_once();
}

File::File(File&& other) noexcept
    : operations_(std::exchange(other.operations_, nullptr)),
      descriptor_(std::exchange(other.descriptor_, -1)) {}

File& File::operator=(File&& other) noexcept {
    if (this != &other) {
        (void)close_once();
        operations_ = std::exchange(other.operations_, nullptr);
        descriptor_ = std::exchange(other.descriptor_, -1);
    }
    return *this;
}

void File::require_open() const {
    if (!is_open()) {
        throw std::logic_error("file is closed");
    }
}

int File::close_once() noexcept {
    if (!is_open()) {
        return 0;
    }
    FileOps* operations = std::exchange(operations_, nullptr);
    const int descriptor = std::exchange(descriptor_, -1);
    return operations->close(descriptor);
}

void File::close_checked() {
    const int error = close_once();
    if (error != 0) {
        throw IoError(error, "close", 0, 0);
    }
}
```

The moves use the same exchange pattern as Chapter 3. Move construction empties
the source. Move assignment first attempts to close its old descriptor, then
transfers the pair; self-move leaves it unchanged. A valid backend moves with
its handle. A destination cannot retain its old backend with a new descriptor.

A move assignment is not a checked completion operation for its old file. When
that error matters, explicitly synchronize and `close_checked()` before assigning
another owner. Suppressing destructor errors is necessary for unwinding; it is
not permission to suppress errors at the success boundary of a durable operation.

`close_once` marks the owner closed **before** calling the backend. If
`close_checked` subsequently throws, the destructor cannot retry that raw integer.
Closing an already-closed owner is an idempotent no-op. A close error leaves this
wrapper closed even though persistence, and on some platforms descriptor state,
may be uncertain.

Do not apply the read/write `EINTR` retry loop mechanically to `close`. On Linux,
the descriptor can already have been released and reused before an error is
reported; retrying can close another resource. Its
[close manual](https://man7.org/linux/man-pages/man2/close.2.html) documents both
that hazard and differing historical platform semantics. Apple's archived
[close manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/close.2.html)
also reports `EINTR` and delayed write errors without establishing a universal
retry rule. This adapter uses one attempt and surfaces the error; it does not
promise portable leak-free recovery from an interrupted close on every POSIX
system. A new platform needs its own verified close policy, not a blind retry.

## Open a trusted regular path without truncating existing data

The ordinary path factory validates the C++ string before calling the C API.
An embedded NUL would otherwise cause `c_str()` to name a shorter path than the
caller intended. `create_new` uses `O_CREAT | O_EXCL`; an existing name is an
error, never a reason to truncate it. `read_write` opens an existing file.

The mode `0600` is an octal permission value: owner read/write, no group or other
access before the process's mask is applied. `O_CLOEXEC` requests that the handle
not survive a successful execution of another program. No `O_APPEND` flag is
used: offsets are deliberate. In particular, the Linux
[`pwrite` documentation](https://man7.org/linux/man-pages/man2/pread.2.html) warns
that append-opened descriptors do not honor the expected explicit-offset behavior.

```cpp title="src/io/file.cpp"
File File::open(const std::string& path, OpenMode mode) {
    if (path.empty() || path.find('\0') != std::string::npos) {
        throw std::invalid_argument("file path is empty or contains NUL");
    }
    int flags = O_CLOEXEC;
    switch (mode) {
    case OpenMode::read_only:  flags |= O_RDONLY; break;
    case OpenMode::read_write: flags |= O_RDWR; break;
    case OpenMode::create_new: flags |= O_RDWR | O_CREAT | O_EXCL; break;
    default: throw std::invalid_argument("unknown open mode");
    }
    int descriptor;
    do {
        descriptor = ::open(path.c_str(), flags, 0600);
    } while (descriptor < 0 && errno == EINTR);
    if (descriptor < 0) {
        const int error = errno;
        throw IoError(error, "open", 0, 0);
    }
    File result(native_ops(), descriptor);
    struct stat status {};
    int outcome;
    do {
        outcome = ::fstat(descriptor, &status);
    } while (outcome < 0 && errno == EINTR);
    if (outcome < 0) {
        const int error = errno;
        throw IoError(error, "fstat", 0, 0);
    }
    if (!S_ISREG(status.st_mode)) {
        throw std::invalid_argument("File requires a regular file");
    }
    return result;
}
```

After `open` succeeds, install the handle in `result` before running the fallible
`fstat` check. If metadata inspection or type validation throws, the local owner
attempts cleanup during unwinding. `S_ISREG` checks the descriptor's actual file
type, not a guessed filename suffix; see Apple's
[`fstat` interface](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/stat.2.html).

The factory is for trusted local engine paths, not a security sandbox for hostile
path components or devices. A type check after opening does not prevent a device
or FIFO open from blocking first. Nor does a failed post-creation check remove
the new directory entry. Owning the descriptor and owning a pathname cleanup
obligation are separate responsibilities.

The path factory and namespace operations are intentionally not covered by our
transfer fake. We test real creation and reopening independently. A later
installation protocol will define temporary names, rename ordering, directory
synchronization, and recovery; none is implied by this factory.

## Finish a read without mistaking a short count for end-of-file

Let $N$ be the requested span size and $d$ the count completed so far. Maintain
$0\leq d\leq N$. Each call receives a view beginning at `output[d]`, a count no
larger than $N-d$, and file offset $o+d$. For a positive result $r$, update
$d\leftarrow d+r$. The full-range check has already proved that these offsets
are representable.

A successful short result is progress, not end-of-file. Only a subsequent zero
result establishes that this attempt cannot read more from the regular file.

```cpp title="src/io/file.cpp"
std::size_t File::read_up_to(std::uint64_t offset, std::span<std::byte> output) {
    require_open();
    (void)checked_file_end(offset, output.size());
    std::size_t done = 0;
    while (done < output.size()) {
        const auto count = std::min(output.size() - done, io_chunk_bytes);
        const Transfer result = operations_->read_at(
            descriptor_, output.subspan(done, count),
            offset + static_cast<std::uint64_t>(done));
        require_transfer(result, count);
        if (result.error == EINTR) {
            continue;
        }
        if (result.error != 0) {
            throw IoError(result.error, "read_at", offset, done);
        }
        if (result.count == 0) {
            return done;
        }
        done += result.count;
    }
    return done;
}

void File::read_exact(std::uint64_t offset, std::span<std::byte> output) {
    if (read_up_to(offset, output) != output.size()) {
        throw FormatError("unexpected end of file");
    }
}
```

If the first attempt is interrupted before reporting progress, retry the same
suffix and offset. If two bytes succeeded, the next attempt starts two bytes
later. `EAGAIN`, `EIO`, and other errors are not converted into busy loops. This
is blocking regular-file I/O, not a readiness-driven socket interface.

For an empty span, an open file returns zero without making a backend call.
The offset is still checked. A closed owner is a usage error even for an empty
operation, making use-after-close detectable instead of conditionally invisible.

A request for six bytes from a file containing only four remaining bytes can
return four through `read_up_to`; `read_exact` instead throws. Both may have
changed the destination's prefix. After an I/O error, do not decode the entire
buffer as a valid record: the wrapper does not provide a strong no-change
guarantee for borrowed output. Prepare a private buffer when installed state
must survive a failed read. A read error's progress counts successful prior
calls, not an assurance about every memory effect of the failing native call.

## Finish a write without promising to undo it

The write loop has the same progress invariant. Its zero-count case differs:
there is no useful write-side end-of-file, so zero progress on a nonempty suffix
becomes `EIO`. Otherwise a faulty backend could keep the loop alive forever.
Append the write loop, explicit synchronization, and the namespace closure:

```cpp title="src/io/file.cpp"
void File::write_all(std::uint64_t offset, std::span<const std::byte> input) {
    require_open();
    (void)checked_file_end(offset, input.size());
    std::size_t done = 0;
    while (done < input.size()) {
        const auto count = std::min(input.size() - done, io_chunk_bytes);
        const Transfer result = operations_->write_at(
            descriptor_, input.subspan(done, count),
            offset + static_cast<std::uint64_t>(done));
        require_transfer(result, count);
        if (result.error == EINTR) {
            continue;
        }
        if (result.error != 0 || result.count == 0) {
            throw IoError(result.error != 0 ? result.error : EIO,
                          "write_at", offset, done);
        }
        done += result.count;
    }
}

void File::sync(SyncMode mode) {
    require_open();
    int error;
    do {
        error = operations_->sync(descriptor_, mode);
    } while (error == EINTR);
    if (error != 0) {
        throw IoError(error, "sync", 0, 0);
    }
}

}  // namespace strata::io
```

Consider six bytes `{10,20,30,40,50,60}` written at offset 5 by a backend that
accepts at most two bytes and fails after four:

| Call | Requested offset | Remaining bytes | Result | Completed count |
|---|---:|---:|---|---:|
| 1 | 5 | 6 | Accept 2 | 2 |
| 2 | 7 | 4 | Accept 2 | 4 |
| 3 | 9 | 2 | `EIO`, no reported progress | 4 |

The prefix at offsets 5–8 remains. Throwing and closing cannot undo it. Nor is
its presence in the test backend evidence of persistence. A future log writer
will stop accepting changes after an uncertain failure and use its recovery
protocol; this low-level owner does not invent transaction state.

For a five-byte transfer with one interruption followed by a two-byte cap,
the results are `EINTR`, 2, 2, 1: **four attempts**, not three. Both buffer
position and file offset must advance only with a positive reported count.

`sync` retries only interruption, without closing the handle. A non-interruption
failure propagates while the file remains open for cleanup or diagnosis.
Success of `write_all` means transfer completion; success of `sync` means the
requested primitive reported success; success of `close_checked` means release
reported success. None alone establishes atomic batch recovery or persistence
of a newly created name.

## Continue with the runnable checks

The engine headers and `src/io/file.cpp` are now complete. Keep the file owner's
backend-lifetime contract, exact record sizes, and failure-progress semantics
visible while testing them. [Part 2 — Deterministic faults and native checks](https://25d-228.github.io/cplusplus-learning/chapter-06-part-2.html)
provides every remaining test file, the CMake additions, expected output, and
independent byte oracle. The chapter's runnable checkpoint is at the end of that
part; no later chapter supplies missing code.
