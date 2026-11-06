# Chapter 10 · Part 3 — Failure images, pressure, and native checks

The code from [Part 1](https://25d-228.github.io/cplusplus-learning/chapter-10.html)
and [Part 2](https://25d-228.github.io/cplusplus-learning/chapter-10-part-2.html)
now has a concrete claim to test: no partially replayed batch becomes visible,
and no allocation is needed between successful synchronization and publication.
A round trip alone cannot test either claim. We need controlled partial effects,
a completed but unacknowledged frame, and failures that occur after useful
preparation work has already happened.

This final part completes the same 4.5-hour chapter. It supplies every test helper,
the native-file driver, an independent decoder, and CMake additions. No separate
engine solution or future database layer is needed to run them.

## Model a byte image, not an imaginary durable disk

Create `tests/wal_file_ops.hpp`. Its fixed 64 KiB array is **test fixture storage**,
not an engine allocation and not a simulated hardware cache. The backend keeps
bytes written even when a later synchronization reports an error. That gives one
permitted complete-unacknowledged image; it does not model which unsynchronized
bytes a real power failure would preserve. The truncation loop below installs
other byte images explicitly.

The double permits one live handle, transfers at most three bytes per call, and
can interrupt individual operations. `write_allowance` counts successful bytes
remaining before an injected write error; zero means fail the next write. A
positive `fail_read_on` counts backend calls, not records. These are test controls,
not production interfaces.

```cpp title="tests/wal_file_ops.hpp"
#ifndef STRATA_TEST_WAL_FILE_OPS_HPP
#define STRATA_TEST_WAL_FILE_OPS_HPP

#include "strata/storage/wal.hpp"

#include <algorithm>
#include <array>
#include <cerrno>
#include <cstddef>
#include <cstdint>
#include <optional>
#include <span>
#include <stdexcept>
#include <utility>

class WalFileOps final : public strata::io::FileOps {
public:
    std::array<std::byte, 64 * 1024> image{};
    std::size_t length = 0;
    std::size_t write_calls = 0, sync_calls = 0, truncate_calls = 0;
    bool live = false;
    bool interrupt_read = false, interrupt_write = false;
    bool interrupt_sync = false, interrupt_truncate = false;
    bool fail_sync = false, fail_truncate = false;
    std::optional<std::size_t> write_allowance;
    std::optional<std::size_t> fail_read_on;
    const strata::storage::WalMemTable* observed = nullptr;
    std::uint64_t expected_sequence = 0;
    std::size_t expected_rows = 0;
    bool early_publication = false;
    strata::memory::Budget* reject_allocation_after_sync = nullptr;

    strata::io::File open() {
        if (live) { throw std::logic_error("test file already open"); }
        live = true;
        return strata::io::File::adopt(*this, 17);
    }
    void set_image(std::span<const std::byte> bytes) {
        if (live || bytes.size() > image.size()) {
            throw std::invalid_argument("invalid test image installation");
        }
        std::copy(bytes.begin(), bytes.end(), image.begin());
        length = bytes.size();
    }
```

`set_image` requires the prior handle to have closed. The tests never silently
replace the bytes underneath an active writer. The `observed` pointer is a
short-lived diagnostic borrow: at each write/sync boundary the double checks the
writer still exposes the preceding sequence and node count. `close` clears that
pointer before the owner can disappear. No fixture callback mutates the writer.

Append the two transfer functions. Both obey Chapter 6's `Transfer` convention:
a nonzero error is paired with zero successful bytes **for that backend call**.
Earlier successful calls have already changed the image.

```cpp title="tests/wal_file_ops.hpp"
strata::io::Transfer read_at(int fd, std::span<std::byte> out,
                                 std::uint64_t offset) noexcept override {
        if (!live || fd != 17) { return {0, EBADF}; }
        if (std::exchange(interrupt_read, false)) { return {0, EINTR}; }
        if (fail_read_on && --*fail_read_on == 0) {
            fail_read_on.reset(); return {0, EIO};
        }
        if (offset >= length) { return {0, 0}; }
        const auto at = static_cast<std::size_t>(offset);
        const auto n = std::min({out.size(), length - at, std::size_t{3}});
        std::copy_n(image.begin() + static_cast<std::ptrdiff_t>(at), n, out.begin());
        return {n, 0};
    }
    strata::io::Transfer write_at(int fd, std::span<const std::byte> in,
                                  std::uint64_t offset) noexcept override {
        ++write_calls;
        observe();
        if (!live || fd != 17) { return {0, EBADF}; }
        if (std::exchange(interrupt_write, false)) { return {0, EINTR}; }
        if (write_allowance && *write_allowance == 0) { return {0, EIO}; }
        if (offset >= image.size()) { return {0, ENOSPC}; }
        const auto at = static_cast<std::size_t>(offset);
        auto n = std::min({in.size(), image.size() - at, std::size_t{3}});
        if (write_allowance) { n = std::min(n, *write_allowance); *write_allowance -= n; }
        std::copy_n(in.begin(), n, image.begin() + static_cast<std::ptrdiff_t>(at));
        length = std::max(length, at + n);
        return {n, 0};
    }
```

For a 17-byte allowance, the writer accepts 3, 3, 3, 3, 3, and 2 bytes, then
returns `EIO` on the next call. `File::write_all` sees those successes separately
and advances its offset each time. The double therefore checks the real transfer
loop, rather than replacing `write_all` with an unrealistic all-or-nothing mock.

The offsets are converted to `size_t` only after being compared with the bounded
fixture size. A zero read at or beyond `length` is EOF. An injected `EIO` is not
EOF and must escape recovery.

Append the remaining methods and close the class/header:

```cpp title="tests/wal_file_ops.hpp"
int sync(int fd, strata::io::SyncMode) noexcept override {
        ++sync_calls;
        observe();
        if (!live || fd != 17) { return EBADF; }
        if (std::exchange(interrupt_sync, false)) { return EINTR; }
        if (fail_sync) { return EIO; }
        if (reject_allocation_after_sync) {
            reject_allocation_after_sync->fail_allocation_on(1);
            reject_allocation_after_sync = nullptr;
        }
        return 0;
    }
    int truncate_to(int fd, std::uint64_t size) noexcept override {
        ++truncate_calls;
        if (!live || fd != 17) { return EBADF; }
        if (std::exchange(interrupt_truncate, false)) { return EINTR; }
        if (fail_truncate) { return EIO; }
        if (size > length) { return EINVAL; }  // This double models shrinking only.
        length = static_cast<std::size_t>(size);
        return 0;
    }
    int close(int fd) noexcept override {
        if (!live || fd != 17) { return EBADF; }
        live = false;
        observed = nullptr;
        return 0;
    }
private:
    void observe() noexcept {
        if (observed && (observed->sequence() != expected_sequence ||
                         observed->table().size() != expected_rows)) {
            early_publication = true;
        }
    }
};
#endif
```

A successful `sync` can arm the budget to reject its next allocation. The normal
commit must finish without consuming that failure. The test then deliberately
allocates one byte and requires that allocation to fail. This is stronger than
checking only that the final byte charge is unchanged: an allocate-then-free
mistake after sync could leave the same charge while still being unsafe.

`fail_allocation_on(1)` receives a valid positive ordinal and does not itself
allocate. The test backend's methods are `noexcept`; do not add a throwing test
assertion inside them. `observe` instead records a Boolean that the calling test
checks afterward. The backend accepts either synchronization mode as a logical
operation; it supplies no hardware guarantee for either one.

The truncate double supports only shrinking. Its injected truncate error leaves
the image unchanged; its sync error can occur after a successful shrink. These
are two explicit failure cases, not a general promise about native errors.

## Make the cross-relation fixture readable

Create `tests/wal.cpp`. The tests use active checks, not `assert`, so Release
builds exercise the same expectations. Begin with imports, local constants, and
the two small assertion helpers:

```cpp title="tests/wal.cpp"
#include "wal_file_ops.hpp"
#include "strata/core/codec.hpp"
#include "strata/storage/table.hpp"

#include <algorithm>
#include <array>
#include <cstdlib>
#include <exception>
#include <iostream>
#include <limits>
#include <span>
#include <stdexcept>
#include <string>
#include <utility>

namespace {
using namespace strata;
using namespace strata::storage;
constexpr std::size_t arena_cap = 1024 * 1024;
constexpr std::size_t budget_cap = 2 * 1024 * 1024;
constexpr auto sync_mode = io::SyncMode::file;

void check(bool ok, const char* message) {
    if (!ok) { throw std::runtime_error(message); }
}
template<class Error, class Function>
void expect_error(const Function& function, const char* message) {
    try { function(); }
    catch (const Error&) { return; }
    throw std::runtime_error(message);
}
```

The fixture ledger is deliberately larger than the tiny data. Resource-pressure
tests will constrain it explicitly rather than depend on incidental allocator
behavior. `expect_error<Error>` catches only the named error family. An unrelated
exception fails the test through `main` instead of being counted as success.

Add the record builders, value checks, and initial-image generator:

```cpp title="tests/wal.cpp"
std::array<Mutation, 2> batch(std::uint32_t region) {
    return {Mutation::put(Event{100, 10, 3, region == 1 ? 500U : 700U}),
            Mutation::put(Account{10, region})};
}
std::array<Mutation, 2> deletion() {
    return {Mutation::erase({Relation::events, 100}), Mutation::put(Account{10, 3})};
}
void expect_committed(WriteResult result, std::uint64_t sequence) {
    check(result.outcome == WriteOutcome::committed && result.reason == WriteReason::none &&
          result.sequence == sequence, "expected committed batch");
}
void check_values(const WalMemTable& wal, std::uint64_t visible,
                  std::uint32_t region, bool deleted = false) {
    const auto e = wal.table().find({Relation::events, 100}, visible);
    const auto a = wal.table().find({Relation::accounts, 10}, visible);
    check(e && a && a->kind == EntryKind::put, "missing cross-relation batch member");
    check(decode_account(a->value) == Account{10, region}, "account value mismatch");
    if (deleted) { check(e->kind == EntryKind::erase && e->value.empty(), "missing tombstone"); }
    else {
        check(e->kind == EntryKind::put && decode_event(e->value) ==
              Event{100, 10, 3, region == 1 ? 500U : 700U}, "event value mismatch");
    }
}
void seed(WalFileOps& disk) {
    memory::Budget budget{budget_cap};
    WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::create, sync_mode};
    expect_committed(wal.write(batch(1)), 1);
    expect_committed(wal.write(batch(2)), 2);
    check(disk.length == 232, "fixture length changed");
}
```

The conditional expression in `batch` selects 500 for the initial event and 700
for its update. `check_values` checks **both** relations at the same visible
sequence. Checking only the event would let an implementation silently omit the
account. At sequence 3 the event entry must be a tombstone; absence of a version
would be a different result for a versioned index.

`seed` creates two complete frames and closes its owner before returning. Its
232-byte image is then suitable for independent opening tests. A test fixture
can use fixed expected lengths because it describes this exact encoded data,
not a generic platform-dependent object layout.

## Observe publication order and allocation timing

Append the first test. It samples the old view before the update and keeps that
view only while the same owner is alive. Its observer checks both sequence and
node count, so moving link publication earlier while delaying the sequence scalar
would still be detected.

```cpp title="tests/wal.cpp"
void check_commit_order() {
    WalFileOps disk;
    memory::Budget budget{budget_cap};
    {
        WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::create, sync_mode};
        disk.observed = &wal;
        disk.interrupt_write = disk.interrupt_sync = true;
        expect_committed(wal.write(batch(1)), 1);
        check(!disk.early_publication, "published before synchronization");
        const auto old = wal.table().find({Relation::events, 100}, 1)->value;
        disk.expected_sequence = 1; disk.expected_rows = 2;
        disk.reject_allocation_after_sync = &budget;
        expect_committed(wal.write(batch(2)), 2);
        check(!disk.early_publication, "published before synchronization");
        expect_error<std::bad_alloc>([&] {
            auto* p = memory::Allocator<std::byte>{budget}.allocate(1);
            memory::Allocator<std::byte>{budget}.deallocate(p, 1);
        }, "post-sync allocation seam was consumed during publication");
        check(decode_event(old).amount_cents == 500, "committed old bytes changed");
        check_values(wal, 1, 1); check_values(wal, 2, 2);
        disk.expected_sequence = 2; disk.expected_rows = 4;
        expect_committed(wal.write(deletion()), 3);
        check_values(wal, 3, 3, true);
        wal.table().check_structure();
        check(wal.log_end() == 302 && !wal.poisoned(), "wrong committed end");
    }
    check(budget.used() == 0 && !disk.live, "commit cleanup leaked");
    {
        disk.interrupt_read = true;
        WalMemTable recovered{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
        check(recovered.sequence() == 3 && recovered.table().size() == 6, "replay lost versions");
        check_values(recovered, 1, 1); check_values(recovered, 2, 2);
        check_values(recovered, 3, 3, true);
    }
    check(budget.used() == 0, "replay cleanup leaked");
}
```

The first commit retries interrupted write and sync calls. Before the second,
the observer's expected state becomes sequence 1 with two stored versions.
Synchronization arms the failure seam. A successful second commit followed by a
failed one-byte test allocation establishes that publication did not allocate
through the engine budget. This does not instrument every C++ runtime allocation;
the source-level commit path supplies the corresponding no-allocation argument.

The old view continues to decode 500 because these committed arena bytes were
not overwritten by the update. This is sequential version access, not a complete
snapshot-lifetime API. After the third batch, the table has six stored versions,
although only the account remains live in the latest logical state. Reopening
checks all three visible sequence choices again.

## Reject before append, including late preparation failure

The next test first rejects malformed/empty batches, reserves the whole available
budget, and checks a zero-cap arena. None may write another log byte or poison
the otherwise usable writer. It then sweeps allocation-failure ordinals for a
48-operation batch until one whole write succeeds.

```cpp title="tests/wal.cpp"
void check_rejections_and_pressure() {
    WalFileOps disk;
    memory::Budget budget{budget_cap};
    {
        WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::create, sync_mode};
        const auto writes = disk.write_calls;
        auto invalid = batch(1);
        invalid[1] = invalid[0];
        check(wal.write(invalid).reason == WriteReason::invalid_batch, "duplicate accepted");
        invalid = batch(1); invalid[0].key.primary = 99;
        check(wal.write(invalid).reason == WriteReason::invalid_batch, "key mismatch accepted");
        invalid = batch(1); invalid[0].length = event_bytes + 1;
        check(wal.write(invalid).reason == WriteReason::invalid_batch, "oversize value accepted");
        check(wal.write({}).reason == WriteReason::invalid_batch, "empty batch accepted");
        auto hold = budget.reserve(budget.limit());
        check(wal.write(batch(1)).reason == WriteReason::resource_exhausted, "budget bypass");
        hold.reset();
        check(disk.write_calls == writes && disk.length == 32 && wal.sequence() == 0 &&
              !wal.poisoned(), "rejection performed append");
        expect_committed(wal.write(batch(1)), 1);
    }
    {
        WalFileOps tiny;
        WalMemTable wal{budget, tiny.open(), 0, WalOpen::create, sync_mode};
        check(wal.write(batch(1)).reason == WriteReason::resource_exhausted &&
              !wal.poisoned() && tiny.length == 32, "arena rejection changed log");
    }
    std::array<Mutation, 48> many{};
    for (std::size_t i = 0; i < many.size(); ++i) {
        many[i] = Mutation::put(Event{static_cast<std::uint64_t>(i), 10, 3, 1});
    }
    bool succeeded = false;
    std::size_t rejected = 0;
    for (std::size_t ordinal = 1; ordinal <= 32; ++ordinal) {
        WalFileOps fresh;
        {
            WalMemTable wal{budget, fresh.open(), arena_cap, WalOpen::create, sync_mode};
            budget.fail_allocation_on(ordinal);
            const auto result = wal.write(many);
            budget.clear_failure();
            if (result.outcome == WriteOutcome::committed) { succeeded = true; }
            else {
                ++rejected;
                check(result.outcome == WriteOutcome::rejected &&
                      result.reason == WriteReason::resource_exhausted && !wal.poisoned() &&
                      wal.table().size() == 0 && fresh.length == 32 && budget.used() == 0,
                      "pre-append allocation rejection changed state");
            }
        }
        check(budget.used() == 0, "allocation sweep leaked");
        if (succeeded) { break; }
    }
    check(succeeded && rejected >= 2, "allocation sweep missed workspace or arena");
    std::cout << "pre-append allocation failures checked=" << rejected << '\n';
}
```

The 48-operation batch needs more than one arena chunk. A later ordinal therefore
fails after earlier chunk acquisitions, not merely before any work began. The
token must destroy pending nodes and return every newly acquired chunk's charge.
The empty table's charge returns to zero, and the file stays at its 32-byte header.

The sweep's upper bound prevents an accidentally nonterminating test. Reaching
a successful ordinal means every earlier allocator request was individually
rejected once in a fresh writer. The observed count was four in the tested
Clang/libstdc++ build; the assertion intentionally requires at least the frame
and arena boundaries, not a particular ABI's node/chunk packing.

A definite rejection does not consume sequence 1. The subsequent valid small
batch commits at sequence 1 without reopening. That is different from poison.

## Keep an uncertain writer stopped

Append the partial-write and synchronization-error checks. The three write
allowances exercise no reported successful bytes, a partial header, and a frame
missing its final byte. The complete-tail sweep comes afterward.

```cpp title="tests/wal.cpp"
void check_uncertainty() {
    for (const auto allowed : {std::size_t{0}, std::size_t{17}, std::size_t{99}}) {
        WalFileOps disk;
        memory::Budget budget{budget_cap};
        {
            WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::create, sync_mode};
            expect_committed(wal.write(batch(1)), 1);
            const auto charge = budget.used();
            const auto old = wal.table().find({Relation::events, 100})->value;
            const auto used = wal.table().used_bytes();
            disk.write_allowance = allowed;
            const auto result = wal.write(batch(2));
            check(result.outcome == WriteOutcome::uncertain && result.sequence == 2 &&
                  result.io_error == EIO && wal.poisoned() && wal.sequence() == 1 &&
                  wal.table().size() == 2 && wal.table().used_bytes() == used &&
                  budget.used() == charge, "uncertain append published or leaked");
            check(decode_event(old).amount_cents == 500, "failed batch invalidated old view");
            const auto calls = disk.write_calls;
            check(wal.write(batch(2)).reason == WriteReason::writer_poisoned &&
                  disk.write_calls == calls, "poisoned writer appended again");
            check(disk.length == 132 + allowed, "partial effect not preserved");
        }
        disk.write_allowance.reset();
        {
            WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
            check(wal.sequence() == 1 && disk.length == 132, "partial batch replayed");
            expect_committed(wal.write(batch(2)), 2);
        }
        check(budget.used() == 0, "uncertain cleanup leaked");
    }
    WalFileOps disk;
    memory::Budget budget{budget_cap};
    {
        WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::create, sync_mode};
        expect_committed(wal.write(batch(1)), 1);
        disk.fail_sync = true;
        const auto result = wal.write(batch(2));
        check(result.outcome == WriteOutcome::uncertain && wal.sequence() == 1 &&
              wal.table().size() == 2 && disk.length == 232, "sync failure semantics");
        check_values(wal, 1, 1);
    }
    expect_error<io::IoError>([&] {
        WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
    }, "recovery must synchronize even a complete log");
    check(!disk.live && budget.used() == 0 && disk.length == 232, "failed open escaped");
    disk.fail_sync = false;
    {
        WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
        check(wal.sequence() == 2 && !wal.repaired_tail(), "complete unacknowledged batch lost");
        check_values(wal, 2, 2);
    }
}
```

The rollback check compares arena used bytes as well as the shared budget charge.
A failed preparation could consume room inside an already charged chunk without
changing that chunk's charge; checking only the ledger would miss that leak of
arena capacity. The prior borrowed row also remains readable.

After an uncertain return, another request is rejected without any new backend
write calls. Only after the old owner is destroyed does recovery trim a permitted
tail and resume. With a synchronization failure the file contains a complete
second frame, while the live writer still exposes only the first batch. Keeping
the sync failure enabled makes opening itself fail. Disabling it allows opening
to accept and synchronize both batches.

This is a complete-but-unacknowledged test image, not a process killed at an
instruction boundary. It exercises the parser's permitted answer without
pretending to model physical power loss or lost interprocess messages.

## Cut the final batch at every byte boundary

A 100-byte final batch has 101 retained-prefix lengths, from zero through 100.
The first 100 are shorter than the frame; zero leaves no tail bytes at all.
For every incomplete prefix, recovery must expose exactly the preceding two rows,
not one or both rows from the new batch. The complete prefix exposes both new
versions. Then write again and reopen again, so stale suffixes cannot hide behind
a one-time successful read.

```cpp title="tests/wal.cpp"
void check_tail_prefixes() {
    WalFileOps original;
    seed(original);
    for (std::size_t kept = 0; kept <= 100; ++kept) {
        WalFileOps disk;
        disk.set_image(std::span{original.image}.first(132 + kept));
        memory::Budget budget{budget_cap};
        {
            WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
            const auto sequence = kept == 100 ? 2U : 1U;
            check(wal.sequence() == sequence && wal.table().size() == 2 * sequence,
                  "tail exposed half a batch");
            check_values(wal, sequence, sequence);
            check(disk.length == (kept == 100 ? 232U : 132U), "tail not physically removed");
            check(wal.repaired_tail() == (kept > 0 && kept < 100), "wrong repair flag");
            if (kept != 100) { expect_committed(wal.write(batch(2)), 2); }
        }
        {
            WalMemTable again{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
            check(again.sequence() == 2 && !again.repaired_tail(), "resume left stale suffix");
        }
        check(budget.used() == 0, "prefix replay leaked");
    }
    std::cout << "final-batch prefixes checked=101 (100 incomplete, 1 complete)\n";
}
```

At `kept == 0`, EOF is exactly at 132 and `repaired_tail()` is false. At lengths
1 through 23 the header is incomplete; 24 through 99 leave an incomplete body or
trailer. These nonempty permitted tails are physically removed. At 100 the final
frame is complete, no repair occurs, and sequence 2 is accepted.

This exhausts byte cuts of **this** small valid final frame. It is not exhaustive
over payload contents, device failures, or all file sizes. The maximum-batch test
below separately exercises the format's operation and frame-size bounds.

## Corruption is not permission to discard a frame

Add helpers that require failed opening to release all charged resources and
leave the physical image unchanged. `rewrite_checksums` is a test-only tool for
making structurally checksummed but semantically invalid records; otherwise a
CRC failure would prevent us from reaching inner validation.

```cpp title="tests/wal.cpp"
void expect_bad_image(WalFileOps& disk) {
    const auto before = disk.image;
    const auto length = disk.length;
    memory::Budget budget{budget_cap};
    expect_error<FormatError>([&] {
        WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
    }, "corrupt WAL accepted");
    check(!disk.live && budget.used() == 0 && disk.image == before && disk.length == length &&
          disk.truncate_calls == 0, "corruption handling modified file or leaked");
}
void rewrite_checksums(WalFileOps& disk, std::size_t start) {
    auto header = std::span{disk.image}.subspan(start, wal_format::batch_header_bytes);
    put_le<4>(header.last(4), table_format::crc32(header.first(20)));
    const auto payload = static_cast<std::size_t>(get_le<4>(header.subspan(4, 4)));
    const auto frame = std::span{disk.image}.subspan(start, 24 + payload + 4);
    put_le<4>(frame.last(4), table_format::crc32(frame.first(frame.size() - 4)));
}
```

The following test flips one chosen bit in each of the fixture's 232 bytes. It
therefore covers both the complete final batch and earlier required data. It
also tries every short file-header prefix; those are all fatal rather than
accepted batch tails.

The structured mutations then repair their checksums after introducing a sequence
gap, zero count, unknown operation, or key/row disagreement. The last case builds
two account operations and changes the second key in **both** its envelope and
its row image, so only the duplicate-key rule can reject it after codec checks.

```cpp title="tests/wal.cpp"
void check_corruption() {
    WalFileOps original;
    seed(original);
    for (std::size_t at = 0; at < original.length; ++at) {
        WalFileOps disk;
        disk.set_image(std::span{original.image}.first(original.length));
        disk.image[at] ^= std::byte{1};
        expect_bad_image(disk);
    }
    for (std::size_t n = 0; n < 32; ++n) {
        WalFileOps disk;
        disk.set_image(std::span{original.image}.first(n));
        expect_bad_image(disk);
    }
    for (int variant = 0; variant < 4; ++variant) {
        WalFileOps disk;
        disk.set_image(std::span{original.image}.first(original.length));
        auto bytes = std::span{disk.image};
        if (variant == 0) { put_le<8>(bytes.subspan(140, 8), 3); } // Sequence gap.
        if (variant == 1) { put_le<4>(bytes.subspan(148, 4), 0); } // Empty batch.
        if (variant == 2) { bytes[157] = std::byte{9}; }           // Unknown operation.
        if (variant == 3) { put_le<8>(bytes.subspan(158, 8), 99); }// Key/row mismatch.
        rewrite_checksums(disk, 132);
        expect_bad_image(disk);
    }
    {
        WalFileOps disk;
        disk.set_image(std::span{original.image}.first(156)); // Full final header, no body.
        auto header = std::span{disk.image}.subspan(132, 24);
        put_le<4>(header.subspan(4, 4), wal_format::max_payload_bytes + 1);
        put_le<4>(header.last(4), table_format::crc32(header.first(20)));
        expect_bad_image(disk); // An invalid length is not a permitted torn body.
    }
    {
        WalFileOps disk;
        disk.set_image(std::span{original.image}.first(135));
        disk.image[132] = std::byte{'X'};
        expect_bad_image(disk); // Short garbage is not automatically a valid tail.
    }
    {
        memory::Budget budget{budget_cap};
        const std::array changes{Mutation::put(Account{10, 1}), Mutation::put(Account{11, 2})};
        auto frame = wal_format::encode_batch(budget, 2, changes);
        // Second operation starts at 24 + 14 + 14 = 52.
        put_le<8>(std::span{frame}.subspan(54, 8), 10);
        put_le<8>(std::span{frame}.subspan(68, 8), 10);
        WalFileOps disk;
        disk.set_image(std::span{original.image}.first(132));
        std::copy(frame.begin(), frame.end(), disk.image.begin() + 132);
        disk.length += frame.size();
        rewrite_checksums(disk, 132);
        expect_bad_image(disk);
    }
    std::cout << "single-bit corruptions checked=232; short file headers checked=32\n";
}
```

For the duplicate fixture, each account operation is 28 bytes. Its second
operation starts at relative offset `24 + 28 = 52`; its envelope key is at 54
and its encoded row key at `52 + 14 + 2 = 68`. Both must change to construct a
true duplicate rather than an easier envelope/row mismatch. Those offsets derive
from the explicit format, never from a C++ member address.

The length-bound case contains a complete final header and no body, but its
repaired header checksum describes a payload one byte above the cap. It must
fail before treating the absent body as a tolerated tail. The short `X` prefix
likewise contradicts the magic and is rejected rather than removed. Complete
checksums are never an excuse to ignore semantic constraints.

## Opening can fail after useful replay work

A partially recovered table must not escape when reading, repairing, synchronizing,
or allocating later state fails. The next test first exercises read error,
truncate error, and truncate-success/sync-error cases. It then sweeps recovery
allocations, including failures after earlier batches have already been replayed
inside the unfinished constructor.

```cpp title="tests/wal.cpp"
void check_failed_recovery() {
    WalFileOps original;
    seed(original);
    for (int variant = 0; variant < 3; ++variant) {
        WalFileOps disk;
        disk.set_image(std::span{original.image}.first(149));
        if (variant == 0) { disk.fail_read_on = 15; }
        if (variant == 1) { disk.fail_truncate = true; }
        if (variant == 2) { disk.fail_sync = true; }
        memory::Budget budget{budget_cap};
        expect_error<io::IoError>([&] {
            WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
        }, "recovery I/O error hidden as a tail");
        check(!disk.live && budget.used() == 0, "failed recovery exposed resources");
        check(disk.length == (variant == 2 ? 132U : 149U), "wrong repair failure effect");
        disk.fail_read_on.reset(); disk.fail_truncate = disk.fail_sync = false;
        disk.interrupt_truncate = true;
        WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
        check(wal.sequence() == 1, "repair retry failed");
    }
    bool succeeded = false;
    for (std::size_t ordinal = 1; ordinal <= 32; ++ordinal) {
        WalFileOps disk;
        disk.set_image(std::span{original.image}.first(original.length));
        memory::Budget budget{budget_cap};
        budget.fail_allocation_on(ordinal);
        try {
            WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
            check(wal.sequence() == 2, "resource retry replay mismatch");
            succeeded = true;
        } catch (const std::bad_alloc&) {}
        budget.clear_failure();
        check(budget.used() == 0 && !disk.live && disk.length == original.length &&
              disk.image == original.image, "replay allocation failure altered source");
        if (succeeded) { break; }
    }
    check(succeeded, "recovery allocation sweep never succeeded");
}
```

For the first two failure variants the deterministic image stays 149 bytes.
For the third it has already shrunk to 132 before the injected sync error. Every
failure closes the handle and returns all charged allocations. A later opening
uses the actual remaining image, not a cached assumption about the failed call.
The enabled truncate interruption is retried when repair is still needed.

In the allocation sweep the file is complete, so there is no reason to modify it.
Failure may destroy a table containing replayed first-batch nodes, but their bytes
still exist in the log and are replayed on the next attempt. A cleanup success
is not permission to return those partial nodes to an application.

Add the format and sequence-boundary checks:

```cpp title="tests/wal.cpp"
void check_bounds() {
    memory::Budget budget{budget_cap};
    WalFileOps disk;
    std::array<Mutation, MemTable::max_batch + 1> changes{};
    for (std::size_t i = 0; i < changes.size(); ++i) {
        changes[i] = Mutation::put(Event{static_cast<std::uint64_t>(i), 10, 3, 1});
    }
    {
        WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::create, sync_mode};
        check(wal.write(changes).reason == WriteReason::invalid_batch, "257-operation batch accepted");
        expect_committed(wal.write(std::span{changes}.first(MemTable::max_batch)), 1);
        check(wal.log_end() == 32 + 24 + 11264 + 4, "maximum frame bound mismatch");
    }
    {
        WalMemTable wal{budget, disk.open(), arena_cap, WalOpen::recover, sync_mode};
        check(wal.table().size() == 256, "maximum batch lost entries");
    }
    WalFileOps high;
    const auto maximum = std::numeric_limits<std::uint64_t>::max();
    {
        WalMemTable wal{budget, high.open(), arena_cap, WalOpen::create, sync_mode, maximum - 1};
        expect_committed(wal.write(batch(1)), maximum);
        check(wal.write(batch(2)).reason == WriteReason::sequence_exhausted && !wal.poisoned(),
              "sequence overflow accepted");
    }
    {
        WalMemTable wal{budget, high.open(), arena_cap, WalOpen::recover, sync_mode, maximum - 1};
        check(wal.sequence() == maximum, "maximum sequence replay mismatch");
    }
    expect_bad_image(high); // Default expected base zero must not match another epoch.
}
```

The 257-operation request is rejected, while 256 maximum-size event puts produce
11264 payload bytes and an 11292-byte frame. The final test starts a standalone
log at base `UINT64_MAX - 1`, writes exactly one maximum-numbered sequence, and
rejects the next request before addition can wrap. Reopening with the correct
expected base succeeds; reopening that same file with the default zero base
fails. This is arithmetic and log-generation validation, not a database with
all earlier sequence history.

## Exercise real files across ordinary process exits

The same test executable has a thin native mode. Add it and `main` to finish
`tests/wal.cpp`. `--write` uses exclusive creation. `--recover-append` expects a
permitted tail after two complete batches and appends the deletion batch after
repair. `--inspect` opens again and checks the resulting three sequences.

```cpp title="tests/wal.cpp"
void native(const std::string& command, const std::string& path, io::SyncMode mode) {
    memory::Budget budget{budget_cap};
    if (command == "--write") {
        WalMemTable wal{budget, io::File::open(path, io::OpenMode::create_new),
                        arena_cap, WalOpen::create, mode};
        expect_committed(wal.write(batch(1)), 1);
        expect_committed(wal.write(batch(2)), 2);
        std::cout << "native WAL sequence=2 bytes=" << wal.log_end() << '\n';
    } else if (command == "--recover-append" || command == "--inspect") {
        WalMemTable wal{budget, io::File::open(path, io::OpenMode::read_write),
                        arena_cap, WalOpen::recover, mode};
        if (command == "--recover-append") {
            check(wal.sequence() == 2 && wal.repaired_tail(), "expected repaired native tail");
            check_values(wal, 2, 2);
            expect_committed(wal.write(deletion()), 3);
        }
        check(wal.sequence() == 3, "native recovered sequence mismatch");
        check_values(wal, 1, 1); check_values(wal, 2, 2); check_values(wal, 3, 3, true);
        std::cout << "native WAL sequence=3 bytes=" << wal.log_end() << '\n';
    } else { throw std::invalid_argument("unknown native command"); }
}
}  // namespace

int main(int argc, char** argv) {
    try {
        if (argc == 3 || (argc == 4 && std::string{argv[3]} == "--full")) {
            native(argv[1], argv[2], argc == 4 ? io::SyncMode::full_device : sync_mode);
            return EXIT_SUCCESS;
        }
        if (argc != 1) { throw std::invalid_argument("use no arguments, or COMMAND PATH [--full]"); }
        check_commit_order(); check_rejections_and_pressure(); check_uncertainty();
        check_tail_prefixes(); check_corruption(); check_failed_recovery(); check_bounds();
        std::cout << "WAL checks passed; all test-owned budgets returned to zero\n";
        return EXIT_SUCCESS;
    } catch (const std::exception& error) {
        std::cerr << "WAL check failed: " << error.what() << '\n';
        return EXIT_FAILURE;
    }
}
```

No-argument execution runs the deterministic checks. Supplying `--full` as the
fourth argument selects the existing full-device synchronization branch; otherwise
the explicit choice is file synchronization. A failure remains a failing exit,
not a silently downgraded success. All command modes own their file and memtable
inside a budget scope; ordinary scope cleanup closes them before process exit.

The Python check supplies independent byte interpretation and invokes these modes
in separate processes. It is not the storage or recovery implementation. Create
`scripts/check_wal_fixture.py` with the following imports and decoder:

```python title="scripts/check_wal_fixture.py"
#!/usr/bin/env python3
"""Decode a real WAL independently and check repair across ordinary process exits."""
import argparse
from pathlib import Path
import struct
import subprocess
import tempfile
import zlib


def require(condition: bool, message: str) -> None:
    if not condition:
        raise ValueError(message)


def decode(data: bytes) -> list[tuple[int, list[tuple]]]:
    require(len(data) >= 32, "short file header")
    require(data[:8] == b"STRWAL01", "file magic")
    version, size, base, reserved, checksum = struct.unpack_from("<IIQII", data, 8)
    require((version, size, base, reserved) == (1, 32, 0, 0), "file header fields")
    require(zlib.crc32(data[:28]) == checksum, "file header checksum")
    at, expected = 32, 1
    batches = []
    while at != len(data):
        require(len(data) - at >= 24, "short batch header")
        magic, length, sequence, count, header_crc = struct.unpack_from("<4sIQII", data, at)
        require(magic == b"BAT1" and sequence == expected, "batch identity")
        require(1 <= count <= 256 and 14 * count <= length <= 44 * count, "batch bounds")
        require(zlib.crc32(data[at:at + 20]) == header_crc, "batch header checksum")
        end = at + 24 + length
        require(end + 4 <= len(data), "short complete frame")
        require(zlib.crc32(data[at:end]) == struct.unpack_from("<I", data, end)[0],
                "batch checksum")
        pos, operations, keys = at + 24, [], set()
        for _ in range(count):
            require(end - pos >= 14, "short operation header")
            relation, kind, primary, value_length = struct.unpack_from("<BBQI", data, pos)
            pos += 14
            require(relation in (1, 2) and kind in (1, 2), "operation tags")
            require(value_length <= end - pos, "operation bound")
            key = relation, primary
            require(key not in keys, "duplicate key")
            keys.add(key)
            value = data[pos:pos + value_length]
            pos += value_length
            if kind == 2:
                require(value_length == 0, "tombstone payload")
                row = None
            else:
                require(value_length == (30 if relation == 1 else 14), "row length")
                row = struct.unpack("<BBQQIQ" if relation == 1 else "<BBQI", value)
                require(row[:3] == (1, relation, primary), "row identity")
                row = row[2:]
            operations.append((relation, kind, primary, row))
        require(pos == end, "trailing payload")
        batches.append((sequence, operations))
        expected += 1
        at = end + 4
    return batches
```

[`struct`'s leading `<`](https://docs.python.org/3/library/struct.html#byte-order-size-and-alignment) requests explicit little-endian standard-width fields
without native alignment. The formats correspond directly to the tables in
Part 1. Python's [`zlib.crc32`](https://docs.python.org/3/library/zlib.html#zlib.crc32) is an independent implementation from our bitwise C++
checksum. The decoder deliberately requires complete frames; it does **not**
reimplement the engine's permissive-tail rule and then use that same rule as an
oracle for itself.

After reading each batch, the reference checker applies its operations to a small
Python dictionary. This is a test-only expected-state calculation, not the
learner's storage implementation or an unbounded query substitute. Add it now:

```python title="scripts/check_wal_fixture.py"
def verify(batches: list[tuple[int, list[tuple]]], final_sequence: int) -> None:
    require(len(batches) == final_sequence, "batch count")
    state = {}
    for sequence, operations in batches:
        for relation, kind, primary, row in operations:
            if kind == 2:
                state.pop((relation, primary), None)
            else:
                state[(relation, primary)] = row
        require(state[(2, 10)] == (10, sequence), "account snapshot")
        if sequence == 3:
            require((1, 100) not in state, "event deletion")
        else:
            require(state[(1, 100)] == (100, 10, 3, 500 if sequence == 1 else 700),
                    "event snapshot")
```

Finally add the driver. It verifies the initial 232-byte file and refusal to
create over it, appends a 13-byte third-header prefix, runs native recovery, and
requires a 302-byte result with its original 232-byte complete prefix unchanged.
The corruption case flips the final checksum and requires the native diagnostic
rather than accepting any unrelated failing exit.

```python title="scripts/check_wal_fixture.py"
def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--program", required=True, type=Path)
    parser.add_argument("--full", action="store_true", help="require native full-device sync")
    args = parser.parse_args()
    program = args.program.resolve()
    with tempfile.TemporaryDirectory(prefix="strata-wal-") as directory:
        path = Path(directory) / "fixture.wal"

        def run(command: str, target: Path, success: bool = True) -> subprocess.CompletedProcess:
            result = subprocess.run([str(program), command, str(target)] +
                                    (["--full"] if args.full else []),
                                    capture_output=True, text=True, timeout=30)
            require((result.returncode == 0) == success,
                    f"unexpected exit {result.returncode}: {result.stdout}{result.stderr}")
            if success:
                print(result.stdout, end="")
            return result

        run("--write", path)
        original = path.read_bytes()
        require(len(original) == 232, "native initial length")
        verify(decode(original), 2)
        run("--write", path, False)
        require(path.read_bytes() == original, "exclusive create altered existing WAL")

        # Append a 13-byte prefix of a correctly sequenced third batch header.
        tail = bytearray(original[132:156])
        struct.pack_into("<Q", tail, 8, 3)
        struct.pack_into("<I", tail, 20, zlib.crc32(tail[:20]))
        with path.open("ab") as file:
            file.write(tail[:13])
        run("--recover-append", path)
        repaired = path.read_bytes()
        require(len(repaired) == 302 and repaired[:232] == original, "native repair boundary")
        verify(decode(repaired), 3)
        run("--inspect", path)

        broken = Path(directory) / "corrupt.wal"
        damage = bytearray(repaired)
        damage[-1] ^= 1
        broken.write_bytes(damage)
        failure = run("--inspect", broken, False)
        require("WAL batch checksum" in failure.stderr, "wrong corruption diagnostic")
        require(broken.read_bytes() == damage, "corrupt complete frame was silently removed")
    print("independent WAL decoder passed; native repair preserved complete prefix")


if __name__ == "__main__":
    main()
```

The temporary directory isolates every run. Only tiny fixtures are read into
Python memory. The success checks remain active under Python optimization because
`require` uses an explicit conditional rather than `assert`. A command timeout
is a test-driver bound, not an engine performance assertion.

The tail is written by a normal Python append and the engine is reopened after
normal process exits. This validates native offset/truncate/sync plumbing and
file-format compatibility. It is not a crash-kill experiment, a namespace
persistence test, or a physical power-loss test. Those distinctions should remain
visible when interpreting a green test run.

## Build the cumulative program

Append the following to the root `CMakeLists.txt`. Keep every earlier target and
test. Chapter 6 already discovered the Python interpreter; do not duplicate that
configuration or require a new build system.

```cmake title="CMakeLists.txt"
target_sources(strata-core PRIVATE src/storage/wal.cpp)
add_executable(strata-wal-tests tests/wal.cpp)
target_link_libraries(strata-wal-tests PRIVATE strata-core)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(strata-wal-tests PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion)
endif()
add_test(NAME wal COMMAND strata-wal-tests)
add_test(NAME wal-native
    COMMAND "${Python3_EXECUTABLE}"
            "${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_wal_fixture.py"
            --program $<TARGET_FILE:strata-wal-tests>)
```

```sh title="Terminal — Debug and Release"
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug --parallel
ctest --test-dir build/debug --output-on-failure
cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/release --parallel
ctest --test-dir build/release --output-on-failure
```

Use the native toolchain selection from Chapter 1. No new compiler banner,
operating-system version, or LTS label is required. The cumulative suite now has
**14 CTest entries**. The normal deterministic WAL executable printed the
following in the tested Clang 17/libstdc++ Linux environment; the allocation-count
line can vary with compatible implementations:

```text title="Observed deterministic WAL output"
pre-append allocation failures checked=4
final-batch prefixes checked=101 (100 incomplete, 1 complete)
single-bit corruptions checked=232; short file headers checked=32
WAL checks passed; all test-owned budgets returned to zero
```

```sh title="Terminal — independent native file check"
python3 scripts/check_wal_fixture.py --program build/debug/strata-wal-tests
```

```text title="Observed native-file and independent-decoder output"
native WAL sequence=2 bytes=232
native WAL sequence=3 bytes=302
native WAL sequence=3 bytes=302
independent WAL decoder passed; native repair preserved complete prefix
```

On the target Mac, a separate full-device run requests that mode explicitly:

```sh title="Terminal — explicit full-device request"
python3 scripts/check_wal_fixture.py --program build/debug/strata-wal-tests --full
```

Inspect and report a failure rather than removing `--full` and calling it the
same experiment. The native M4/macOS path and Apple's full-device branch were
not executed in the authoring environment. The default native-file checks did
execute on Linux; a passed file-sync run establishes only what those checks test.

The diagnostic build from earlier chapters also applies. Use a separate build
tree so its flags do not contaminate optimized measurements:

```sh title="Terminal — instrumented build"
cmake -S . -B build/sanitize -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-O1 -g -fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
cmake --build build/sanitize --parallel
ctest --test-dir build/sanitize --output-on-failure
```

The exact cumulative source passed all 14 tests in Debug, Release, and
AddressSanitizer/UndefinedBehaviorSanitizer builds with Clang 17 on Linux.
Linux leak detection was enabled in the instrumented authoring check. Use the
installed toolchain's supported diagnostics on the Mac, and distinguish an
unavailable diagnostic from a passing run. Clang's
[AddressSanitizer usage documentation](https://clang.llvm.org/docs/AddressSanitizer.html#usage)
explains the instrumentation/link flags. Sanitizers do not certify write ordering.

## Break the ordering without breaking memory safety

Make one temporary edit in `WalMemTable::write`: move the **existing**
`prepared.commit();` line from after the inner I/O `try` to immediately before
`file_.write_all(end_, image);` inside that inner `try`. Do not move the replay
commit, and do not leave a second writer commit call behind. Rebuild and run only
the WAL test in both Debug and Release:

```sh title="Terminal — the deliberate order defect must fail"
cmake --build build/debug --target strata-wal-tests
ctest --test-dir build/debug -R '^wal$' --output-on-failure
cmake --build build/release --target strata-wal-tests
ctest --test-dir build/release -R '^wal$' --output-on-failure
```

```text title="Expected diagnostic — observed in all three build configurations"
WAL check failed: published before synchronization
```

The observer sees new nodes during append even though the sequence scalar has not
yet advanced. No out-of-bounds access is needed for this defect. The temporary
version produced the same logical failure in the instrumented build without an
AddressSanitizer or UndefinedBehaviorSanitizer report. Restoring commit to its
post-sync location made the tests pass again. **Restore it and rerun the complete
suite before continuing.**

A second useful variation needs no engine edit: change the test-only partial
write allowance from 17 to 50. Predict the effect first. The header is now
complete and valid, but the body/trailer is incomplete. The writer still returns
uncertainty and preserves its old memtable; recovery still truncates to 132.
Then change only `fail_sync` instead of limiting writes: all 100 bytes remain,
and the reopened sequence becomes 2. Restore the published fixture afterward.

## Diagnose the boundary that failed

| Symptom | Inspect first |
|---|---|
| A rejected batch changed file length | A write happened before encoding/preparation completed, or an I/O exception reached a pre-append rejection handler |
| Account/event states disagree after replay | Operations were published during parsing rather than after full-frame and whole-batch validation |
| An old view changes after an uncertain write | Preparation overwrote committed bytes, or rollback used the wrong arena checkpoint |
| Reopen silently loses a corrupt complete final batch | A checksum/schema error was converted to a tolerated EOF case |
| Reopen after repair fails on stale bytes | The physical tail was not truncated, or the next append used physical length instead of verified end |
| Replay fails with a budget error | Count retained arena chunks plus both temporary vector capacities; do not bypass the ledger or skip records |
| The first read error becomes a successful short log | An I/O exception was mistaken for EOF |
| `--full` fails while the normal run passes | The two runs requested different synchronization modes; inspect platform support and the actual error |

The chapter ends with a log-backed memtable that encodes whole batches, exposes
explicit uncertain outcomes, repairs only permitted incomplete tails, and refuses
to publish an incompletely opened state. It has not installed any sorted table
or chosen between log generations. The next chapter adds flush installation,
authoritative manifest state, directory ownership, and database reopening; do
not bolt an untracked table file onto this log and assume its bytes are now safe
to discard.

## References used in this chapter

The format and state machines above are developed in the text and complete
listings. These sources supply the particular language and platform background,
not missing implementation steps.

- [SQLite WAL, section 2](https://www.sqlite.org/wal.html#how_wal_works) for the general append/checkpoint distinction;
  [atomic commit, sections 2 and 8](https://www.sqlite.org/atomiccommit.html) for explicit persistence assumptions and failure-testing distinctions. SQLite's algorithms/formats differ from Strata's.
- [Apple archived fsync manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
  and [truncate manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/truncate.2.html)
  for the native operations. Check the installed manuals rather than inferring the current SDK or filesystem from archived documentation.
- The C++20 draft's [constructor cleanup](https://timsong-cpp.github.io/cppwp/n4861/except.ctor),
  [termination rules](https://timsong-cpp.github.io/cppwp/n4861/except.terminate), and
  [return copy/move elision](https://timsong-cpp.github.io/cppwp/n4861/class.copy.elision)
  for owner cleanup and the named-vector return. The nonmovable prepared-token return remains the distinct Chapter 7 case.
- [Clang AddressSanitizer usage](https://clang.llvm.org/docs/AddressSanitizer.html#usage)
  for the instrumented build; a passing sanitizer run is not a persistence-order proof.
