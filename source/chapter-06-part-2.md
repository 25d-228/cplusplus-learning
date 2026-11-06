# Chapter 6 — Records, bytes, and a dependable file boundary

## Part 2 — Deterministic faults and native checks

This is the continuation of [Part 1](https://25d-228.github.io/cplusplus-learning/chapter-06.html),
not another chapter or another time allocation. Retain the five engine files
written there: `core/record.hpp`, `core/checked.hpp`, `core/codec.hpp`,
`io/file.hpp` under `include/strata/`, and `src/io/file.cpp`. Earlier foundation
sources and all six earlier CTest entries remain unchanged.

Part 1 defined 30-byte event and 14-byte account images, checked unsigned size and
file-range arithmetic, and the move-only `strata::io::File`. Its borrowed
`FileOps` backend performs one attempt; `File` finishes short transfers and
retries interruption. `IoError` records the original offset and known completed
count. EOF can yield a short `read_up_to`, while `read_exact` rejects truncation.
The nonthrowing destructor attempts close once; explicit sync/close report
failures. No operation promises to roll back bytes already written.

Here we give those contracts controlled counterexamples, complete the native
fixture writer, compare bytes with an independent decoder, and add two tests.
Every repeated filename block appends in order, as in Part 1. The full chapter's
3.0-hour estimate already includes these checks; the combined implementation is
the prerequisite for Chapter 7.

## Make short transfers and faults reproducible

The test backend is deliberately one 64-byte file, not a replacement filesystem.
It owns fixed storage, a logical length, and a single live handle numbered 7.
Counters and fault settings are single-threaded test state. Reopening this fixture
keeps its bytes and counters; create a fresh backend for a fresh test.

A backend is declared before the `File` borrowing it, so reverse destruction
order closes the file before destroying its operations object. `open_fixture`
marks that slot live and transfers its handle through `adopt`; it performs no
native open. There is no allocation, lock, or global failure injector in its
virtual methods.

```cpp title="tests/memory_file_ops.hpp"
#ifndef STRATA_TEST_MEMORY_FILE_OPS_HPP
#define STRATA_TEST_MEMORY_FILE_OPS_HPP

#include "strata/io/file.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <cstddef>
#include <cstdint>
#include <optional>
#include <span>
#include <stdexcept>
#include <utility>

class MemoryFileOps final : public strata::io::FileOps {
public:
    std::array<std::byte, 64> bytes{};
    std::size_t used = 0;
    std::size_t accepted = 0;
    std::size_t write_calls = 0;
    std::size_t read_calls = 0;
    std::size_t sync_calls = 0;
    std::size_t close_calls = 0;
    bool live = false;
    bool interrupt_write = false;
    bool interrupt_read = false;
    bool interrupt_sync = false;
    bool zero_write = false;
    std::optional<std::size_t> fail_after;
    std::optional<std::size_t> fail_read_call;
    int sync_error = 0;
    int close_error = 0;

    strata::io::File open_fixture() {
        if (live) {
            throw std::logic_error("fixture already open");
        }
        live = true;
        return strata::io::File::adopt(*this, 7);
    }
```

`fail_after` is a limit on cumulative **accepted bytes**, not attempted bytes
or the absolute file offset. Setting it to 4 leaves the first four successful
bytes and fails the next request. `interrupt_write` is a one-shot flag:
`std::exchange(flag, false)` returns its previous value while clearing it.
The initializer-list form of `std::min` takes the smallest of the three given
`size_t` values; it is not allocating a vector.

```cpp title="tests/memory_file_ops.hpp"
strata::io::Transfer write_at(int descriptor,
                                  std::span<const std::byte> input,
                                  std::uint64_t offset) noexcept override {
        ++write_calls;
        if (!live || descriptor != 7) { return {0, EBADF}; }
        if (std::exchange(interrupt_write, false)) { return {0, EINTR}; }
        if (zero_write) { return {0, 0}; }
        if (fail_after && accepted >= *fail_after) { return {0, EIO}; }
        if (offset >= bytes.size()) { return {0, ENOSPC}; }
        const auto at = static_cast<std::size_t>(offset);
        auto count = std::min({input.size(), std::size_t{2}, bytes.size() - at});
        if (fail_after) { count = std::min(count, *fail_after - accepted); }
        for (std::size_t i = 0; i < count; ++i) { bytes[at + i] = input[i]; }
        accepted += count;
        used = std::max(used, at + count);
        return {count, 0};
    }

    strata::io::Transfer read_at(int descriptor, std::span<std::byte> output,
                                 std::uint64_t offset) noexcept override {
        ++read_calls;
        if (!live || descriptor != 7) { return {0, EBADF}; }
        if (std::exchange(interrupt_read, false)) { return {0, EINTR}; }
        if (fail_read_call && read_calls == *fail_read_call) { return {0, EIO}; }
        if (offset >= used) { return {0, 0}; }
        const auto at = static_cast<std::size_t>(offset);
        const auto count = std::min({output.size(), std::size_t{2}, used - at});
        for (std::size_t i = 0; i < count; ++i) { output[i] = bytes[at + i]; }
        return {count, 0};
    }
```

The offset is checked before narrowing to `size_t` or subtracting it from a
capacity. Reads return zero at or beyond `used`. An injected failed read does
not modify its destination; that is this fake's defined behavior, not a claim
that every native error preserves the unread suffix. `fail_read_call` counts
attempts, including interrupted calls, and is compared with the incremented
counter.

The fixed array begins zero-filled, making gap bytes predictable in our
nonzero-offset examples. It is not an experiment in physical sparse-file
allocation. Its `close` below consumes the logical slot even when it reports an
injected error. This models the dangerous case in which a retry would act on a
handle no longer owned by the caller.

```cpp title="tests/memory_file_ops.hpp"
int sync(int descriptor, strata::io::SyncMode mode) noexcept override {
        ++sync_calls;
        if (!live || descriptor != 7) { return EBADF; }
        if (std::exchange(interrupt_sync, false)) { return EINTR; }
        if (mode != strata::io::SyncMode::file) { return ENOTSUP; }
        return sync_error;
    }

    int close(int descriptor) noexcept override {
        ++close_calls;
        if (!live || descriptor != 7) { return EBADF; }
        live = false;
        return close_error;
    }
};

#endif
```

The fake always rejects `full_device`; that verifies error propagation without
pretending to simulate a storage controller. It does not track dirty versus
persistent images, directory entries, process crashes, or device-cache ordering.
Those require different experiments.

## Check the contracts with small concrete inputs

The test program uses throwing checks so they remain active in Release builds.
`expect_throw<Error>` runs a callable, accepts only the named exception family,
and fails if the call returns normally. Other exception types propagate to
`main`. `expect_io` checks the numeric error and progress fields, not a
platform-specific `what()` string. These helpers are small templates using the
callable and capture syntax from Chapter 5.

An unnamed namespace keeps these helpers local to the executable. The `using`
declarations shorten names in that namespace only; the engine headers do not
inject them into every including translation unit.

```cpp title="tests/core_io.cpp"
#include "memory_file_ops.hpp"
#include "strata/core/checked.hpp"
#include "strata/core/codec.hpp"
#include "strata/io/file.hpp"

#include <array>
#include <cerrno>
#include <cstddef>
#include <cstdint>
#include <exception>
#include <iostream>
#include <limits>
#include <span>
#include <stdexcept>
#include <string>
#include <string_view>
#include <utility>

namespace {
using strata::io::File;
using strata::io::IoError;
using strata::io::OpenMode;
using strata::io::SyncMode;

void require(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}

template<class Error, class Action>
void expect_throw(const Action& action, const char* message) {
    try { action(); }
    catch (const Error&) { return; }
    throw std::runtime_error(message);
}

template<class Action>
void expect_io(const Action& action, int code, std::uint64_t offset,
               std::size_t transferred) {
    try { action(); }
    catch (const IoError& error) {
        require(error.code().value() == code && error.offset() == offset &&
                error.transferred() == transferred, "wrong I/O failure context");
        return;
    }
    throw std::runtime_error("expected IoError");
}
```

The codec fixture matches the first event/account example in the roadmap.
For both schemas, try **every** shorter prefix, including zero and one byte, so
a header access cannot hide ahead of a length check. Also test excess length,
wrong version, unknown and wrong relation tags, full-width integers, zero
values, and rejected narrowing that leaves its destination unchanged.

```cpp title="tests/core_io.cpp"
void check_codecs() {
    using namespace strata;
    const Event event{100, 10, 3, 500};
    const Account account{10, 1};
    const auto event_image = encode_event(event);
    const auto account_image = encode_account(account);
    require(decode_event(event_image) == event, "event round-trip");
    require(decode_account(account_image) == account, "account round-trip");
    require(event_image[22] == std::byte{0xf4} &&
            event_image[23] == std::byte{0x01}, "500 byte order");
    require(key_of(event) == Key{Relation::events, 100}, "event key");
    require(KeyLess{}(key_of(event), key_of(account)), "relation ordering");
    require(KeyLess{}({Relation::events, 255}, {Relation::events, 256}),
            "numeric rather than serialized key ordering");
    const auto max64 = std::numeric_limits<std::uint64_t>::max();
    const auto max32 = std::numeric_limits<std::uint32_t>::max();
    const Event largest{max64, max64, max32, max64};
    require(decode_event(encode_event(largest)) == largest, "full-width fields");
    require(decode_account(encode_account(Account{0, 0})) == Account{0, 0},
            "zero is a value");

    std::array<std::byte, 8> number{};
    put_le<8>(number, 0x0102030405060708ULL);
    require(number[0] == std::byte{8} && number[7] == std::byte{1}, "eight bytes");
    require(get_le<8>(number) == 0x0102030405060708ULL, "eight-byte decode");
    const std::span<const std::byte> view{event_image};
    for (std::size_t length = 0; length < view.size(); ++length) {
        expect_throw<FormatError>([&] { (void)decode_event(view.first(length)); },
                                  "truncated event accepted");
    }
    const std::span<const std::byte> account_view{account_image};
    for (std::size_t length = 0; length < account_view.size(); ++length) {
        expect_throw<FormatError>([&] { (void)decode_account(account_view.first(length)); },
                                  "truncated account accepted");
    }
    const std::array<std::byte, event_bytes + 1> too_long{};
    expect_throw<FormatError>([&] { (void)decode_event(too_long); }, "extra byte");
    const std::array<std::byte, account_bytes + 1> long_account{};
    expect_throw<FormatError>([&] { (void)decode_account(long_account); }, "extra account byte");
    auto bad = event_image;
    bad[0] = std::byte{2};
    expect_throw<FormatError>([&] { (void)decode_event(bad); }, "bad version");
    bad = event_image;
    bad[1] = std::byte{99};
    expect_throw<FormatError>([&] { (void)decode_event(bad); }, "unknown relation");
    bad[1] = std::byte{2};
    expect_throw<FormatError>([&] { (void)decode_event(bad); }, "wrong relation");
    const auto before = number;
    expect_throw<std::overflow_error>([&] {
        put_le<4>(std::span<std::byte>{number}.first(4), std::uint64_t{1} << 32);
    }, "narrowing accepted");
    require(number == before, "failed integer encode changed output");
    expect_throw<FormatError>([&] { (void)get_le<4>(number); }, "integer width");
    std::cout << "codec: event=30 account=14; malformed records rejected\n";
}
```

The temporary from `encode_event(largest)` lives until the end of that full
expression; `decode_event` consumes its borrowed bytes during the call and
retains no view. This use is safe, unlike returning a span into a local array.
The test distinguishes numerical key order from serialized-byte order without
using an invalid comparator.

Next exercise arithmetic with scalar maxima, not fabricated enormous spans:

```cpp title="tests/core_io.cpp"
void check_arithmetic() {
    using namespace strata;
    const auto limit = std::numeric_limits<std::size_t>::max();
    require(checked_size_add(limit, 0) == limit, "limit plus zero");
    require(checked_size_mul(limit, 0) == 0, "zero width");
    require(checked_size_mul(3, event_bytes) == 90, "three events");
    expect_throw<std::length_error>([&] { (void)checked_size_add(limit, 1); }, "add");
    expect_throw<std::length_error>([&] { (void)checked_size_mul(limit, 2); }, "mul");
    expect_throw<std::overflow_error>([] {
        (void)checked_u64_add(std::numeric_limits<std::uint64_t>::max(), 1);
    }, "total");
    require(checked_file_end(file_offset_limit - 3, 3) == file_offset_limit,
            "largest permitted end");
    require(checked_file_end(file_offset_limit, 0) == file_offset_limit,
            "empty range at limit");
    expect_throw<std::length_error>([] { (void)checked_file_end(file_offset_limit, 1); },
                                   "offset boundary");
    expect_throw<std::length_error>([] { (void)checked_file_end(UINT64_MAX, 0); },
                                   "offset before subtraction");
    std::cout << "arithmetic: overflow rejected before I/O\n";
}
```

A three-event reservation is 90 bytes. The boundary calls prove that addition,
multiplication, totals, and file offsets take their intended error paths.
The implementation does not need to allocate `SIZE_MAX` bytes to reject a count.

The first I/O check writes five bytes beginning at offset 3. It forces one
interruption in each transfer direction and in synchronization. Closed-state,
empty-request, and out-of-range behavior are checked with the same owner:

```cpp title="tests/core_io.cpp"
void check_short_transfers() {
    MemoryFileOps operations;
    operations.interrupt_write = true;
    operations.interrupt_read = true;
    operations.interrupt_sync = true;
    {
        auto file = operations.open_fixture();
        const std::array<std::byte, 5> input{
            std::byte{1}, std::byte{2}, std::byte{3}, std::byte{4}, std::byte{5}};
        std::array<std::byte, 5> output{};
        file.write_all(3, input);
        file.read_exact(3, output);
        require(output == input, "short transfers lost bytes");
        require(operations.write_calls == 4 && operations.read_calls == 4,
                "interruption plus 2+2+1");
        require(operations.used == 8 && operations.bytes[0] == std::byte{0},
                "nonzero offset");
        file.write_all(strata::file_offset_limit, {});
        require(file.read_up_to(strata::file_offset_limit, {}) == 0, "empty read");
        expect_throw<std::length_error>([&] {
            file.write_all(strata::file_offset_limit, input);
        }, "out-of-range write");
        require(operations.write_calls == 4 && operations.read_calls == 4,
                "empty/range check reached backend");
        file.sync(SyncMode::file);
        require(operations.sync_calls == 2, "sync interruption");
        file.close_checked();
        file.close_checked();
        require(!file.is_open(), "close state");
        expect_throw<std::logic_error>([&] { file.write_all(0, {}); }, "closed use");
    }
    require(!operations.live && operations.close_calls == 1, "close once");
    std::cout << "short I/O: EINTR + 2+2+1; explicit close once\n";
}
```

The exact four-call counts would catch forgetting to advance the offset, treating
a short transfer as completion, or counting `EINTR` as progress. The empty requests
and the rejected range must not increase either transfer counter. Calling
`close_checked` twice and then destroying the owner must still yield one backend
close call.

Now observe a failure with an externally visible prefix. The sentinel byte `ee`
marks output positions the fake never filled. A short read at EOF and a read
error after two bytes must have different results:

```cpp title="tests/core_io.cpp"
void check_partial_failure() {
    MemoryFileOps operations;
    operations.fail_after = 4;
    const std::array<std::byte, 6> input{
        std::byte{10}, std::byte{20}, std::byte{30},
        std::byte{40}, std::byte{50}, std::byte{60}};
    {
        auto file = operations.open_fixture();
        expect_io([&] { file.write_all(5, input); }, EIO, 5, 4);
        require(operations.used == 9 && operations.accepted == 4 &&
                operations.write_calls == 3, "wrong failed-write progress");
        for (std::size_t i = 0; i < 4; ++i) {
            require(operations.bytes[5 + i] == input[i], "prefix disappeared");
        }
        std::array<std::byte, 6> output{};
        output.fill(std::byte{0xee});
        require(file.read_up_to(5, output) == 4, "short EOF count");
        require(output[4] == std::byte{0xee} && output[5] == std::byte{0xee},
                "EOF changed unread suffix");
        expect_throw<strata::FormatError>([&] { file.read_exact(5, output); },
                                         "truncated exact read accepted");
        operations.fail_read_call = operations.read_calls + 2;
        output.fill(std::byte{0xee});
        expect_io([&] { (void)file.read_up_to(5, output); }, EIO, 5, 2);
        require(output[0] == input[0] && output[1] == input[1] &&
                output[2] == std::byte{0xee}, "read failure progress");
    }
    require(!operations.live && operations.close_calls == 1, "failure cleanup");
    std::cout << "write failure: 4 bytes remain; EOF and read errors stay distinct\n";
}
```

The file's logical end is 9: five preceding bytes plus four accepted bytes.
The first failure is caught **inside** the file's scope to inspect that state;
the final scope exit closes the handle. This is not resuming a database write
after an uncertain commit. It is inspecting a deterministic low-level fixture.

The next check moves a file into an already-owning destination, tests self-move,
rejects zero write progress, surfaces synchronization errors, and models an
interrupted close whose handle has already been consumed:

```cpp title="tests/core_io.cpp"
void check_completion_and_moves() {
    MemoryFileOps first;
    MemoryFileOps second;
    {
        auto source = first.open_fixture();
        auto moved = std::move(source);
        require(!source.is_open() && moved.is_open(), "move construction");
        auto destination = second.open_fixture();
        destination = std::move(moved);
        require(!moved.is_open() && destination.is_open(), "move assignment");
        require(second.close_calls == 1 && !second.live, "old destination close");
        auto& alias = destination;
        destination = std::move(alias);
        require(destination.is_open() && first.close_calls == 0, "self move");
        first.zero_write = true;
        const std::array<std::byte, 1> one{std::byte{1}};
        expect_io([&] { destination.write_all(0, one); }, EIO, 0, 0);
        first.sync_error = EIO;
        expect_io([&] { destination.sync(SyncMode::file); }, EIO, 0, 0);
        require(destination.is_open(), "failed sync discarded handle");
        expect_io([&] { destination.sync(SyncMode::full_device); }, ENOTSUP, 0, 0);
        first.close_error = EINTR;
        expect_io([&] { destination.close_checked(); }, EINTR, 0, 0);
        require(!destination.is_open() && !first.live, "failed close state");
        destination.close_checked();
    }
    require(first.close_calls == 1 && second.close_calls == 1, "close retried");
    MemoryFileOps unwinding;
    expect_throw<std::runtime_error>([&] {
        auto file = unwinding.open_fixture();
        throw std::runtime_error("leave scope");
    }, "unwinding");
    require(!unwinding.live && unwinding.close_calls == 1, "unwind close");
    std::cout << "owners: moves, zero progress, sync/close errors, unwinding checked\n";
}
```

The moved owner must keep the first backend, while assignment closes the second
backend's old handle exactly once. `destination` is already closed when the
injected `EINTR` escapes `close_checked`; its subsequent close and destructor must
not repeat the attempt. The final deliberately thrown exception verifies cleanup
during ordinary stack unwinding.

Passing these checks proves the wrapper's exercised state transitions under the
fake's rules. It does not prove native close behavior under all signals or prove
that every acquisition is released under every possible OS failure.

## Write and reopen a real 44-byte fixture

The native path creates a **new** file, writes the event at offset 0 and account
at offset 30, synchronizes in the requested mode, explicitly closes, and opens a
new read-only owner. It decodes both records and checks for EOF at offset 44.
No binary data is written through a C++ object pointer.

```cpp title="tests/core_io.cpp"
void write_fixture(const std::string& path, SyncMode mode) {
    const strata::Event event{100, 10, 3, 500};
    const strata::Account account{10, 1};
    const auto e = strata::encode_event(event);
    const auto a = strata::encode_account(account);
    auto writer = File::open(path, OpenMode::create_new);
    writer.write_all(0, e);
    writer.write_all(strata::event_bytes, a);
    writer.sync(mode);
    writer.close_checked();

    auto reader = File::open(path, OpenMode::read_only);
    std::array<std::byte, strata::event_bytes> event_copy{};
    std::array<std::byte, strata::account_bytes> account_copy{};
    reader.read_exact(0, event_copy);
    reader.read_exact(strata::event_bytes, account_copy);
    require(strata::decode_event(event_copy) == event, "native event");
    require(strata::decode_account(account_copy) == account, "native account");
    std::array<std::byte, 1> extra{};
    require(reader.read_up_to(strata::event_bytes + strata::account_bytes, extra) == 0,
            "unexpected trailing native bytes");
    reader.close_checked();
    std::cout << "native: 44 bytes written, synchronized, closed, reopened\n";
}

}  // namespace
```

If synchronization fails, unwinding still attempts to close the writer; the
program returns failure and may leave a partially or fully written pathname.
It does not claim that failure left the filesystem unchanged. On success it
leaves the fixture for independent inspection. No directory synchronization or
atomic publication protocol is implemented here.

The entry point runs deterministic checks with no arguments, or writes the
native fixture with `--fixture PATH`. A final `full-device` argument requests
the stronger native operation explicitly. A request that is unsupported fails;
it is not silently retried in the ordinary mode.

```cpp title="tests/core_io.cpp"
int main(int argc, char** argv) {
    try {
        if ((argc == 3 || argc == 4) && std::string_view{argv[1]} == "--fixture") {
            const auto mode = argc == 4 ? SyncMode::full_device : SyncMode::file;
            if (argc == 4 && std::string_view{argv[3]} != "full-device") {
                throw std::invalid_argument("expected full-device");
            }
            write_fixture(argv[2], mode);
            return 0;
        }
        if (argc != 1) {
            throw std::invalid_argument("usage: strata-core-tests [--fixture PATH [full-device]]");
        }
        check_codecs();
        check_arithmetic();
        check_short_transfers();
        check_partial_failure();
        check_completion_and_moves();
        std::cout << "core I/O checks passed\n";
        return 0;
    } catch (const IoError& error) {
        std::cerr << "I/O failure: " << error.what() << "; offset=" << error.offset()
                  << "; transferred=" << error.transferred() << '\n';
    } catch (const std::exception& error) {
        std::cerr << "FAIL " << error.what() << '\n';
    }
    return 1;
}
```

The `IoError` handler comes first because it is also a `std::exception`. It reports
its extra context before the general handler. Both return failure. The normal
no-argument checks do not need a filesystem path and do not open a native file.

### Use a different implementation to check the bytes

Round-tripping through our own encoder and decoder could preserve the same
mistake in both. Python's `struct` module supplies an independent encoding.
The prefix `<` selects little-endian standard widths with **no native alignment
padding**; `B` is one byte, `Q` eight, and `I` four. The event format is therefore
`<BBQQIQ`, and the account format is `<BBQI`. See the official
[struct format documentation](https://docs.python.org/3/library/struct.html).

The script below either checks an existing fixture or creates an isolated
temporary directory, invokes the C++ writer, checks its bytes, and deliberately
tries to create the same name again. That second run must fail without changing
the file. Python removes its temporary directory after the child processes
finish; it does not implement the engine's I/O loop.

```python title="scripts/check_record_fixture.py"
"""Independently check Chapter 6 bytes; optionally run its native fixture writer."""
import pathlib
import struct
import subprocess
import sys
import tempfile


def verify(path: pathlib.Path) -> None:
    data = path.read_bytes()
    expected = (
        struct.pack("<BBQQIQ", 1, 1, 100, 10, 3, 500)
        + struct.pack("<BBQI", 1, 2, 10, 1)
    )
    if data != expected:
        raise ValueError(f"wrong record bytes: expected {expected.hex()}, got {data.hex()}")
    event = struct.unpack_from("<BBQQIQ", data, 0)
    account = struct.unpack_from("<BBQI", data, 30)
    print(f"oracle: bytes={len(data)} event={event} account={account}")


def main() -> None:
    if len(sys.argv) == 2:
        verify(pathlib.Path(sys.argv[1]))
        return
    if len(sys.argv) == 3 and sys.argv[1] == "--program":
        executable = pathlib.Path(sys.argv[2]).resolve()
        with tempfile.TemporaryDirectory(prefix="strata-core-") as directory:
            path = pathlib.Path(directory) / "records.bin"
            subprocess.run([str(executable), "--fixture", str(path)], check=True)
            verify(path)
            before = path.read_bytes()
            collision = subprocess.run(
                [str(executable), "--fixture", str(path)],
                capture_output=True, text=True, check=False,
            )
            if collision.returncode != 1 or path.read_bytes() != before:
                raise ValueError("create_new did not preserve an existing fixture")
            print("oracle: create_new rejected collision without changing the file")
        return
    raise ValueError("usage: check_record_fixture.py FILE | --program EXECUTABLE")


if __name__ == "__main__":
    try:
        main()
    except (OSError, ValueError, subprocess.CalledProcessError) as error:
        print(f"fixture check failed: {error}", file=sys.stderr)
        sys.exit(1)
```

`check=True` turns an unexpected child failure into a Python exception. The
intentional collision uses `check=False`, then tests its status and verifies
unchanged bytes. Exact operating-system error text is not part of this oracle.
Temporary-directory cleanup belongs to the test harness; it is not a claim that
`File` owns or automatically removes names.

## Add the library and two tests to the existing build

Append the following to the root `CMakeLists.txt`, leaving all earlier targets
unchanged. Header-only codecs acquire their include path from `strata-core`;
`file.cpp` is compiled once into that static library. `strata-core-tests` links
the library rather than including the implementation source.

```cmake title="CMakeLists.txt"
add_library(strata-core STATIC src/io/file.cpp)
target_include_directories(strata-core PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/include")
target_compile_features(strata-core PUBLIC cxx_std_20)

add_executable(strata-core-tests tests/core_io.cpp)
target_link_libraries(strata-core-tests PRIVATE strata-core)

foreach(target IN ITEMS strata-core strata-core-tests)
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
        target_compile_options(${target} PRIVATE
            -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion)
    endif()
endforeach()

find_package(Python3 COMPONENTS Interpreter REQUIRED)
add_test(NAME core-io COMMAND strata-core-tests)
add_test(NAME core-io-native
    COMMAND "${Python3_EXECUTABLE}"
            "${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_record_fixture.py"
            --program $<TARGET_FILE:strata-core-tests>)
```

`FindPython3` requests only the interpreter, not Python development libraries.
`$<TARGET_FILE:strata-core-tests>` is a CMake generator expression that supplies
the executable's actual path to the Python command. It avoids hard-coding a
Debug-specific path. These mechanisms are documented by
[FindPython3](https://cmake.org/cmake/help/latest/module/FindPython3.html) and
[add_test](https://cmake.org/cmake/help/latest/command/add_test.html).
No change to the earlier CMake version declaration is needed.

```sh title="Terminal — build and run the cumulative suite"
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug
ctest --test-dir build/debug --output-on-failure

cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/release
ctest --test-dir build/release --output-on-failure
```

Keep the compiler already configured in those build trees. A new build tree
can select the same native compiler with `-DCMAKE_CXX_COMPILER="$CXX"`, as in
Chapter 1. If CMake cannot locate Python, point `-DPython3_EXECUTABLE` at the
compatible interpreter you actually use; do not install development libraries
that this script does not require.

There are now **eight** CTest entries: the earlier six, `core-io`, and
`core-io-native`. The native test makes its own temporary directory, so repeated
runs do not depend on deleting a stale fixture by hand. A passing test process
exits zero. To see the deterministic trace directly:

```sh title="Terminal — inspect the deterministic trace"
./build/debug/strata-core-tests
```

```text title="Expected deterministic output"
codec: event=30 account=14; malformed records rejected
arithmetic: overflow rejected before I/O
short I/O: EINTR + 2+2+1; explicit close once
write failure: 4 bytes remain; EOF and read errors stay distinct
owners: moves, zero progress, sync/close errors, unwinding checked
core I/O checks passed
```

```sh title="Terminal — independent native round-trip"
python3 scripts/check_record_fixture.py --program ./build/debug/strata-core-tests
```

```text title="Expected native/oracle output"
native: 44 bytes written, synchronized, closed, reopened
oracle: bytes=44 event=(1, 1, 100, 10, 3, 500) account=(1, 2, 10, 1)
oracle: create_new rejected collision without changing the file
```

These are expected outputs for the supplied code, not measurements of the
learner's M4. In particular, a successful native round-trip exercises ordinary
execution through a live operating system; it does not crash that system or
remove drive power.

### Keep one fixture for a byte-level inspection

Choose a pathname that does not already exist. The following commands leave it
in the local build directory:

```sh title="Terminal — inspect the encoded bytes"
./build/debug/strata-core-tests --fixture build/debug/chapter-06-records.bin
python3 scripts/check_record_fixture.py build/debug/chapter-06-records.bin
od -An -v -tx1 build/debug/chapter-06-records.bin
```

Ignoring line wrapping, the event and account images are:

```text title="Expected byte sequence"
event (30 bytes):
01 01 64 00 00 00 00 00 00 00 0a 00 00 00 00 00
00 00 03 00 00 00 f4 01 00 00 00 00 00 00
account (14 bytes):
01 02 0a 00 00 00 00 00 00 00 01 00 00 00
```

Locate the four `day` bytes at offsets 18–21 and the `f4 01` amount at offsets
22–23. Counting from the start of the whole file, the account tag is at offset
31, not 1. All offsets in each schema table are relative to that record's start.
A second fixture creation at the same name is supposed to fail. Inspect that
file or use a new filename; do not change the API to truncate merely to make a
repeat command succeed.

On the native Mac, probe the explicit stronger mode with another new pathname:

```sh title="Terminal — request native full-device synchronization"
./build/debug/strata-core-tests --fixture build/debug/chapter-06-full.bin full-device
```

Success gives the same byte fixture and reports that the requested native call
completed. Failure is also informative: record its code and check local
`man 2 fsync`, `man 2 fcntl`, the selected SDK, and the filesystem/device in use.
Do not rewrite an unsupported or failed request as a successful durability test.
No specific macOS version, filesystem, or drive-cache behavior is assumed.

## A logical defect can survive memory diagnostics

Temporarily change only the offset argument in `File::write_all` from
`offset + static_cast<std::uint64_t>(done)` to `offset`. Leave the read loop
unchanged. With two-byte writes, every call now overwrites the same position.
The buffers remain in bounds, yet the file content is wrong.

Rebuild and run the deterministic check in both configurations. It must return
failure, either at the exact-read boundary or the content check. A memory
sanitizer may report nothing because the defect is in file positioning, not
memory access. Restore the original advancing offset before proceeding and
rerun all eight tests. This temporary defect is not prerequisite code.

For supported native compiler/runtime combinations, run the restored program
under both AddressSanitizer and UndefinedBehaviorSanitizer:

```sh title="Terminal — diagnose the restored file boundary"
mkdir -p build/diagnostics
"$CXX" -std=c++20 -O1 -g -fno-omit-frame-pointer \
  -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
  -fsanitize=address,undefined -Iinclude \
  src/io/file.cpp tests/core_io.cpp \
  -o build/diagnostics/strata-core-sanitized
./build/diagnostics/strata-core-sanitized
python3 scripts/check_record_fixture.py --program ./build/diagnostics/strata-core-sanitized
```

The expected output is unchanged and there should be no sanitizer diagnostic.
The deliberately caught `IoError` objects are test outcomes, not memory errors.
See Clang's [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html)
and [UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)
documentation for platform support and limitations. Instrumentation does not
prove persistence, correct byte order, or an operating system's close semantics.

## Diagnose the layer that actually failed

| Observation | Likely boundary | Focused action |
|---|---|---|
| Decoder rejects an otherwise plausible record | Exact size, version, or relation | Inspect the header and count before changing field offsets |
| Round-trip passes but oracle fails | Shared encoder/decoder mistake | Compare the fixed byte table and little-endian arithmetic |
| Read returns fewer bytes without throwing | `read_up_to` reached EOF | Use the returned count; use `read_exact` where truncation is invalid |
| A short write loses or repeats bytes | Buffer position or file offset | Trace `done` and both suffix arguments on every successful call |
| Write loops forever | Zero progress or incorrect retry classification | Treat zero nonempty progress as failure; retry only interruption |
| Failure test has a correct exception but no prefix | Accidental simulated rollback | Verify the fake modifies bytes before returning successful counts |
| File closes twice after a move or close error | Coupled owner state | Transfer/reset both backend and descriptor before release reporting |
| Fixture cannot be created on a repeat command | Exclusive creation worked | Inspect the existing file or choose a new name |
| `full-device` fails | Unsupported request or real I/O failure | Preserve the distinction; inspect the native error and capability |
| Native file opens but does not round-trip after a crash | A different experiment | This chapter has no log or namespace recovery protocol |

The important failure-state distinction is now concrete. Arithmetic and format
validation can reject before mutation. A read can partly fill a private buffer.
A write can leave bytes behind. A failed synchronization does not say that no
data reached storage. Cleanup releases resources as far as its primitive
permits; it does not reverse external effects.

## What carries forward

Keep all eight new source/script files, the `strata-core` library target, and
both new tests. The arithmetic helpers check sizes and ranges before use. The
version-1 row codecs own their returned values and retain no borrowed buffers.
The file owner has explicit move, close, synchronization, and partial-progress
rules; test backends remain borrowed and must outlive it. The temporary wrong
offset edit must be restored.

Next, the index will use `KeyLess`, checked allocation sizes, and stable resource
ownership to build an arena-backed skip list. That chapter introduces aligned
storage and explicit node construction before using them. The present layer
remains independently runnable; no future index or log is needed to test it.

## Technical references

The linked C++ working draft includes newer facilities, but the chapter uses
C++20-compatible features. Apple archive pages describe the native interfaces;
the installed SDK and runtime result determine the local capability.

- C++ language/library: [object representation](https://eel.is/c++draft/basic.types),
  [byte operations](https://eel.is/c++draft/support.types.byteops),
  [shifts](https://eel.is/c++draft/expr.shift),
  [constexpr if](https://eel.is/c++draft/stmt.if),
  [virtual dispatch](https://eel.is/c++draft/class.virtual),
  [abstract classes](https://eel.is/c++draft/class.abstract), and
  [destruction](https://eel.is/c++draft/class.dtor).
- Apple native interfaces: [open](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/open.2.html),
  [pread](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/pread.2.html),
  [pwrite](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/pwrite.2.html),
  [fstat](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/stat.2.html),
  [fsync](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html),
  [fcntl](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fcntl.2.html), and
  [close](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/close.2.html).
- Platform distinctions: Apple, [Optimizing Storage in Your App](https://developer.apple.com/videos/play/wwdc2019/419/),
  synchronization discussion at approximately 10:54–13:38; Linux man-pages,
  [close caveats](https://man7.org/linux/man-pages/man2/close.2.html) and
  [pread/pwrite, including the append-mode caveat](https://man7.org/linux/man-pages/man2/pread.2.html).
- Independent checks and tooling: Python [struct](https://docs.python.org/3/library/struct.html),
  CMake [FindPython3](https://cmake.org/cmake/help/latest/module/FindPython3.html)
  and [add_test](https://cmake.org/cmake/help/latest/command/add_test.html),
  Clang [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html)
  and [UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html).

