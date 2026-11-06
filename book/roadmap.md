# Strata: a C++ storage and query engine

Strata is an embedded, single-node engine for persistent numeric records and
bounded-memory analytical queries. You will build its in-memory index, disk
format, recovery protocol, read views, compaction, query operators, and concurrent
runtime. A small command-line program will exercise those mechanisms without
adding a server or a SQL parser.

The central problem is not merely saving rows. A successful write must have a
precise recovery meaning. A query must keep a consistent view while newer data
arrives. A sort must make progress when its input exceeds its allocation budget.
A canceled task must release its files, memory, and snapshot without destroying
resources another reader still uses. These requirements meet in the same engine.

> **Learning objectives**
>
> Explain and implement resource ownership, object lifetime, value and move
> semantics, exception guarantees, useful generic interfaces, explicit byte
> layouts, indexed storage, recovery ordering, external-memory algorithms, and
> synchronized publication. Diagnose failures and measure tradeoffs in the
> resulting system rather than treating these subjects as isolated vocabulary.

## What the engine does

### Two relations and one ordered key space

The workload uses two fixed schemas. The names below denote unsigned integer
fields with the stated widths; they do not prescribe a packed C++ object layout.

| Relation | Primary key | Value fields |
|---|---|---|
| Events | `event_id`, 64 bits | `account_id`, 64 bits; `day`, 32 bits; `amount_cents`, 64 bits |
| Accounts | `account_id`, 64 bits | `region_id`, 32 bits |

A storage key is the pair `(relation_id, primary_key)`. The day is an integer
label, not a date parser. Amounts are nonnegative integer cents; counts and sums
use checked 64-bit arithmetic. A result that would overflow is an error, not a
wrapped number. Records have explicit byte encodings; writing `sizeof(record)`
bytes from a C++ object is not the disk format.

The engine supports put, delete, point lookup, ordered range scanning, and
bounded atomic write batches that can change both relations. Each committed
batch receives one increasing sequence number. Repeated writes to the same
storage key within a single batch are rejected during validation; updates in
successive batches remain valid. A deletion is represented by a *tombstone*, a
record that suppresses older values.

A snapshot is a repeatable read view of both relations at one committed batch
boundary. This is not a general transaction system: there are no concurrent
writers, read-write transactions, historical timestamp queries, or conflict
resolution protocols. One process owns a database directory, with an operating
system lock preventing a second writer process from opening it.

### A query with a hand-checkable answer

The main query filters events by a day interval, joins them to existing accounts,
and groups by region and day. It returns an event count and a sum of cents for
each group, in `(region_id, day)` order. It is an inner, many-to-one join: an event
without a visible account is omitted; each account key has at most one visible
value in a snapshot.

Consider these accounts:

| Account | Region |
|---|---:|
| 10 | 1 |
| 20 | 2 |

And these events:

| Event | Account | Day | Cents |
|---|---:|---:|---:|
| 100 | 10 | 3 | 500 |
| 101 | 10 | 3 | 250 |
| 102 | 20 | 3 | 700 |
| 103 | 99 | 3 | 50 |
| 104 | 10 | 4 | 100 |

Filtering to day 3 produces these rows:

```text title="Expected query result"
region_id,day,event_count,total_cents
1,3,2,750
2,3,1,700
```

The missing account removes event 103, and the day filter removes event 104.
Now commit one batch that changes account 10 to region 3, changes event 101 to
900 cents, and deletes event 102. An older snapshot must still return the result
above. A new snapshot must return only `(3, 3, 2, 1400)`. No reader may combine
the new region with the old amount by observing a partial batch. This small
example becomes a reference case for storage, execution, and concurrent tests.

## Architecture and ownership

### Storage: ordered generations and sorted files

Strata uses a restricted log-structured merge design. A mutable *memtable* is an
arena-backed skip list ordered by storage key and descending sequence number.
You implement its search, insertion, allocation, and iteration. A write-ahead
log records complete batches before their in-memory publication. Sealed
memtables become immutable sorted tables on disk.

Sorted tables contain checksummed blocks, a sparse block directory, and a Bloom
filter for negative point lookups. A bounded cache owns reusable block buffers;
a pin keeps a block alive while a decoder borrows its bytes. File identity, block
identity, and the lifetime of those borrowed bytes are explicit. The learner
implements the cache policy, index search, checksums, and Bloom filter rather
than calling a storage library.

A small manifest names the installed table set, required log generations, and
recovery progress. Its publication protocol establishes when a new set of files
becomes authoritative. Flushes occur in generation order. The merge policy has
bounded newly flushed runs and one consolidated base run, which may span
size-limited files. Compaction merges the selected inputs into a new base using
bounded buffers. Rewriting a large base is a deliberate, measurable cost of this
limited policy; the book does not disguise it as a full industrial LSM hierarchy.

The [LevelDB implementation description](https://chromium.googlesource.com/external/leveldb/+/HEAD/doc/impl.md)
provides useful background on these file roles and on deletion markers. Strata
uses a narrower policy and develops its own installation and recovery rules;
the reference is neither a dependency nor a substitute for that protocol.

### Read views: freeze first, then share

Acquiring a snapshot seals the active memtable at a batch boundary and captures
its immutable sources together with the installed disk runs. Replacement
capacity is prepared before publication; an empty active table need not be
sealed again. Future writes use another memtable. Readers never traverse links
that a writer is modifying.

Each view owns the exact generations it captured. A later flush or compaction
does not silently replace sources inside that view. Thus an old view can keep an
arena or obsolete file alive even when a newer view no longer needs it. Bounds
on retained generations, metadata, and files make this cost visible. Snapshot
creation can report resource pressure; ingestion can wait or return a bounded
admission failure. Retention is not allowed to become an unbounded allocation.

Compaction chooses the latest version for each key in its selected sources. It
may discard a tombstone only when every possible older source for that key in
the new serving state has been covered. Older snapshots remain valid through
their original sources, not through retained versions in the replacement base.
There is no general multiversion transaction manager hidden behind this design.

### Execution: streaming operators with explicit barriers

The final operator pipeline is:

```text title="Query data flow"
event snapshot scan -> day filter -> narrow projection
    -> external sort by (account_id, event_id)
    -> merge join with the account snapshot scan in account_id order
    -> external sort by (region_id, day, event_id)
    -> streaming count/sum by (region_id, day)
    -> ordered output
```

Batches own their column buffers; views borrow from a named owner for a stated
interval. Sorting is a blocking boundary, so it generates temporary runs when
its allocation is exhausted. The learner implements run generation, buffered
readers, a merge heap, bounded fan-in, multiple merge passes, and temporary-file
cleanup. Only the sort of an already-budgeted in-memory chunk is delegated to
`std::sort`.

The account relation is scanned incrementally; the join does not assume that it
fits in memory. Repeated event keys are consumed one at a time against a single
matching account. The final sort makes equal aggregation keys adjacent, so the
aggregator needs one current group rather than an unbounded hash table. General
many-to-many joins and an optimizer are outside the scope.

### Runtime: admit work before it consumes resources

The first storage and query implementations execute synchronously. Only after
those paths work does the course add threads: one serialized writer, a dedicated
flush/compaction worker, and a small pool for independent read-only queries.
Whole-query jobs avoid nested tasks that wait for children in their own saturated
pool. Maintenance has separate progress resources rather than borrowing the last
query worker and its entire allocation.

Queued requests contain bounded descriptors, not full data sets or pre-acquired
snapshots. A query obtains its memory reservation and snapshot when admitted to
execution; the snapshot time is not the time its request entered the queue.
Cancellation, resource-pressure waits, exceptions, and shutdown all have explicit
state transitions. Blocking waits release the locks needed by the work that can
satisfy them. Thread construction failure and shutdown during a blocked operation
are part of the implementation, not exceptional afterthoughts.

## Boundaries that determine correctness

### Acknowledged, rejected, and uncertain writes

A batch is validated and its publication resources are prepared before entering
the irreversible part of the write path. Appending and synchronizing a complete
log record precedes publishing the batch. Publishing the prepared in-memory
state must not allocate or throw. A definite pre-commit rejection leaves the
visible state unchanged.

An I/O or synchronization failure after writing begins can instead produce an
*uncertain outcome*. The writer then stops accepting changes until recovery; a
caller must not assume that retrying the batch is harmless. Recovery preserves
acknowledged durable batches under the stated persistence assumptions, may
include a complete but unacknowledged final batch, and never exposes half a
batch. There is no automatic exactly-once retry API.

An incomplete final log frame is different from corruption in required committed
storage. The parser accepts only the specifically defined torn-tail cases; it
does not turn every checksum mismatch into permission to discard data. Required
missing files and malformed installed tables cause an explicit recovery failure.

Native process-stop tests and a deterministic persistence model serve different
purposes. Killing a process leaves operating-system caches alive; it does not
establish power-loss durability. Apple's archived
[`fsync` documentation](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
describes drive-cache concerns and `F_FULLFSYNC`. The file layer will inspect the
installed platform, check synchronization results, and state its file and
namespace persistence assumptions rather than silently downgrade them. SQLite's
[atomic-commit testing discussion](https://www.sqlite.org/atomiccommit.html#_testing_atomic_commit_behavior)
illustrates failure simulation, but its rollback-journal algorithm is not
Strata's recovery algorithm.

### Budgeted memory is not total process memory

The initial measurement configuration uses a **256 MiB engine-managed budget**,
with **32 MiB per admitted query**. These are experiment settings, not hardware
requirements or measured results. One mebibyte is $2^{20}$ bytes. Budget charges
include allocated capacities rather than only logical row sizes, plus bounded
metadata, pinned cache blocks, merge buffers, and retained arenas. A buffer shared
by several pins is charged once; a pin prevents that charge from being released.

For a merge with $k$ input runs, let $b_i$ be the byte capacity of one input
buffer, $b_o$ the output-buffer capacity, and $h(k)$ the charged heap and reader
metadata. Let $B_m$ be the bytes reserved for that merge after accounting for
other live operator buffers. Admission requires

$$
k b_i + b_o + h(k) \leq B_m.
$$

For example, with a 2 MiB merge reservation, eight 64 KiB input buffers, one
64 KiB output buffer, and 32 KiB of metadata, the charge is
$8 \times 64 + 64 + 32 = 608$ KiB, below 2048 KiB. Eight is the number of live
input readers, not the total number of runs ever created. More runs require
additional passes. The implementation also respects a file-descriptor limit;
memory alone does not determine fan-in.

Admission reserves capacity; allocation consumes it. Those counters must not
be added together as if they described two independent copies of the same bytes.
The course distinguishes reservations, live charged allocation, process resident
memory, and filesystem cache effects. Allocator bookkeeping, thread stacks,
runtime libraries, and the operating system make a 256 MiB engine budget
something other than a 256 MiB process-memory guarantee.

Run lists, table directories, queued tasks, and retained generations have explicit
caps or bounded representations. Cache misses have a reserved-buffer bypass path
when all cache entries are pinned; a full cache must not deadlock a scan. Disk
headroom is checked for old and new compaction files, pinned obsolete files,
logs, and overlapping spill passes. Reducing input size is acceptable only while
it still forces the spill and compaction paths being measured.

## Starting knowledge and native tools

The starting knowledge is basic C and Python: functions, loops, arrays, pointers,
and a simple command line. No prior knowledge of C++ ownership, database internals,
external-memory algorithms, or the memory model is assumed. The first five
chapters contain standalone runnable programs before the engine build begins.
Concurrency receives another language-focused introduction immediately before
threaded engine work.

Use native ARM64 macOS on the M4 Mac mini with 16 GB memory. Chapter 1 observes
the installed operating system and tools, compiles small feature probes, and
builds a multi-file program. It does not assume a particular macOS release or
require a virtual machine. A compatible Apple Clang/libc++ toolchain or maintained
LLVM alternative is suitable. The needed baseline is C++20 concepts and
`std::span`, together with the C++ standard library facilities actually used.
Apple documents [compiler, library, and deployment distinctions](https://developer.apple.com/xcode/cpp/);
upstream [libc++ feature status](https://libcxx.llvm.org/Status/Cxx20.html) is a
separate source, not an Apple compiler version map.

CMake and CTest manage builds and focused tests. CMake's
[language-standard meta-features](https://cmake.org/cmake/help/latest/manual/cmake-compile-features.7.html#requiring-language-standards)
select a language mode, but do not prove every required library facility is
available. Real feature probes and test programs decide compatibility. No C++23
`std::expected`, modules, `std::format`, coroutines, or `std::jthread` requirement
is introduced merely to use a newer feature.

LLDB, [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html), and a
separate [ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) build
are used where the installed toolchain supports them; undefined-behavior checks
are added to the applicable diagnostic build. Unsupported diagnostics are
reported, not mistaken for a passing run. Measurements use a separate optimized
build. Python supplies deterministic data generation, independent small-data
answers, and experiment drivers, never the engine's storage or query execution.

The standard library and operating system may supply containers for bounded
metadata, threads, synchronization, file primitives, and ordinary utilities. No
database, query, columnar-data, or task-runtime library supplies the required
index, recovery, cache, merge, executor, or scheduling mechanisms. The course
does not require an external test framework.

## The implementation workspace

The implementation lives on `work/strata`. The following is the intended source
layout as it grows; later files are created by their owning chapters, not required
by an earlier runnable example. Headers use the `strata` namespace. The book
names concrete files and complete prerequisite interfaces at each step.

```text title="Learner source tree"
CMakeLists.txt
include/strata/
  core/          errors, byte views, checked arithmetic, record types
  io/            file ownership and synchronization interface
  memory/        arena, budget, reservations
  storage/       memtable, table, cache, log, manifest, view, database
  query/         batch, scan, sort, merge join, aggregation
  runtime/       task, executor, cancellation, engine coordination
src/
  core/
  io/
  memory/
  storage/
  query/
  runtime/
apps/strata.cpp
exercises/
  chapter-01/    a multi-translation-unit program
  chapter-02/    values, containers, and borrowed views
  chapter-03/    a move-only resource owner
  chapter-04/    failure-safe state replacement
  chapter-05/    constrained generic code
  chapter-18/    synchronization and thread lifetime
tests/          unit, integration, fault, and concurrency tests
scripts/        deterministic fixtures, oracle, and measurements
```

Build trees, database directories, spill files, generated data, and measurement
output are local ignored artifacts,
not source. Chapter 1 supplies the first build targets; later chapters extend
them without requiring unfinished future targets.

## The chapter sequence and implementation budget

The plan allocates **86.0 hours of implementation**, plus **6.5 hours of additional
debugging and integration reserve**, for **92.5 hours total**. Reading, authoring,
site generation, and publication are excluded. Each chapter allocation already
includes its local tests and ordinary debugging; the reserve is for additional
cross-system problems, not a second charge for the same work. The estimates assume
a fast implementer following the complete worked mechanisms.

| Part | Chapters | Implementation hours |
|---|---|---:|
| Language and tools | 1–5 | 10.5 |
| Persistent storage and read views | 6–13 | 34.5 |
| Bounded query execution | 14–17 | 18.0 |
| Concurrency and runtime integration | 18–20 | 10.5 |
| Integrated workloads, failures, and measurements | 21–23 | 12.5 |
| Additional debugging and integration reserve | Across the course | 6.5 |
| **Total** | **23 chapters** | **92.5** |

The sequence is cumulative. A chapter requires all earlier completed code unless
its outline explicitly identifies a standalone exercise; the prerequisites below
name the particularly important dependencies. Each chapter ends in runnable code,
including chapters whose subsystem is not yet connected to the database. No
chapter needs a future implementation to make its own demonstration work.

| Chapter | Topic | Hours |
|---|---|---:|
| 1 | From C files to a native C++ program | 2.0 |
| 2 | Values, references, containers, and views | 2.0 |
| 3 | Lifetime, RAII, and move-only owners | 2.5 |
| 4 | Errors and failure-safe state changes | 2.0 |
| 5 | Templates, concepts, and ordering contracts | 2.0 |
| 6 | Records, bytes, and a dependable file boundary | 3.0 |
| 7 | An arena-backed skip-list memtable | 4.5 |
| 8 | Indexed immutable tables | 5.0 |
| 9 | Memory budgets and a pinned block cache | 3.5 |
| 10 | Atomic batches and write-ahead recovery | 4.5 |
| 11 | Flush installation and database reopening | 5.5 |
| 12 | Immutable snapshots and merged reads | 4.5 |
| 13 | Streaming compaction and safe reclamation | 4.0 |
| 14 | Owned batches and a streaming query pipeline | 3.5 |
| 15 | External sorting with bounded fan-in | 6.0 |
| 16 | A streaming many-to-one merge join | 5.0 |
| 17 | Grouping, checked aggregates, and complete queries | 3.5 |
| 18 | Synchronization and thread lifetime | 3.0 |
| 19 | Bounded execution and cooperative cancellation | 4.0 |
| 20 | Concurrent readers, ingestion, and maintenance | 3.5 |
| 21 | Reproducible workloads and independent answers | 4.5 |
| 22 | Recovery and resource-pressure experiments | 4.0 |
| 23 | Profiling, optimization, and the final demonstration | 4.0 |
| **Allocated implementation** | | **86.0** |

These are planning numbers rather than deadlines. Compare actual setup, coding,
and debugging time with the allocation after each chapter. Use actuals to revise
the remaining forecast. Reduce peripheral command variants, dataset size, or
optional measurement sweeps before removing the index, recovery, spill, or
concurrent-lifetime work. A larger explanation does not make a small coding task
consume more hours.

## Part I — Language and tools

### Chapter 1 — From C files to a native C++ program

**Prerequisites:** basic C functions, pointers, and shell commands. No engine code.
**Implementation: 2.0 hours.**

**A translation unit is not a program.** Separate declarations, definitions,
headers, and linkage in `exercises/chapter-01/main.cpp`, `stats.hpp`, and
`stats.cpp`. Introduce namespaces, initialization, a small aggregate, and the
C++ compilation model by extending a familiar C-style loop.

**Build for the machine you have.** Observe the installed macOS, architecture,
compiler, library, CMake, and debugger. Compile the required C++20 feature probes;
configure a native Debug build and explain include paths, warnings, and linkage.
The probes are complete short programs, not a requirement to understand concepts
before Chapter 5.

**A check that can fail.** Add the `foundation-01` executable and CTest entry.
Summarizing `{10, 20, 12}` must produce `count=3 sum=42`; empty input has count and
sum zero. Checks remain active in optimized builds rather than placing required
work inside `assert`.

**Follow an execution.** Set an LLDB breakpoint in the accumulation function,
inspect values, and diagnose a deliberately missing definition at link time.

**Outcome:** a working multi-file executable and repeatable build/test commands.
**Work split:** observation and probes 0.5; program 0.5; CMake/CTest 0.5;
debugger and diagnosis 0.5 hours.

### Chapter 2 — Values, references, containers, and views

**Prerequisites:** Chapter 1's build, functions, and initialization.
**Implementation: 2.0 hours.**

**Copy a value or refer to an object.** Compare value parameters, mutable
references, `const` references, and returned values. Introduce `auto`, range-for,
`std::array`, and `std::vector` with concrete element and capacity traces.

**Borrow bytes without owning them.** Use `std::span` and `std::string_view` in
complete programs under `exercises/chapter-02/`. Name each owner and show why
read-only access does not extend the owner's lifetime.

**Invalidation changes the contract.** Grow a vector, reacquire its views, and
separate size, capacity, storage, and iterator validity. The normal exercise
never dereferences a dangling view; predict invalidation before checking it with
a separate diagnostic example.

**Make interfaces say what changes.** Build `foundation-02`: changing the middle
value of `{3, 5, 8}` through a mutable span to 7 makes a read-only sum equal 18.
Copying the original array into a value parameter must not change its caller.

**Outcome:** small value/view programs and explicit borrowing rules ready for
block decoding and query batches. **Work split:** programs 1.0; interface
variations 0.5; invalidation diagnosis and tests 0.5 hours.

### Chapter 3 — Lifetime, RAII, and move-only owners

**Prerequisites:** Chapters 1–2, especially references and borrowed views.
**Implementation: 2.5 hours.**

**Storage and an object are different.** Introduce classes, constructors,
destructors, member initialization, access control, and the lifetime of members.
Trace scope exit and destruction order before introducing ownership transfer.
The [language lifetime rules](https://eel.is/c++draft/basic.life) are a reference
for this distinction, not prerequisite reading.

**Acquire and release together.** Build a small owning byte buffer in
`exercises/chapter-03/buffer_owner.cpp`. Contrast a value type, which can copy
its contents, with a move-only resource type; explain `std::unique_ptr`, deleted
copy operations, and the rule of zero before writing custom special members.

**A move is a change of responsibility.** Implement move construction, move
assignment, `std::move`, and a safe moved-from state. Borrowed spans are separate
from their owner. Account for an already-owned destination and self-move without
double release; explain why a nonthrowing move matters to containers.

**Observe cleanup.** `foundation-03` moves a three-byte buffer, verifies the
payload in its new owner and the documented empty source, then reports zero live
allocations after scope exit. Run the focused ownership checks with supported
memory diagnostics.

**Outcome:** a complete move-only owner and a resource-lifetime trace, not a
handwritten replacement for every standard smart pointer. **Work split:** owner
and lifetime exercises 1.0; move paths 1.0; checks and diagnosis 0.5 hours.

### Chapter 4 — Errors and failure-safe state changes

**Prerequisites:** Chapter 3's resource owner and destruction rules.
**Implementation: 2.0 hours.**

**Failures have different meanings.** Introduce exceptions, stack unwinding,
`std::optional` for absence, scoped status enums, and a small error vocabulary.
Distinguish invalid input, resource exhaustion, absent data, and failed I/O.
Destructors release resources without throwing; explicit completion reports
failures that callers need to see.

**Prepare, then commit.** Write a replacement operation for an owned record
buffer in `exercises/chapter-04/transactional_buffer.cpp`. Build and validate the
replacement before a nonthrowing swap. Contrast the no-change guarantee with a
weaker valid-but-changed state.

**Fail the second allocation.** Add a deterministic allocation-failure seam.
`foundation-04` starts with `{4, 5}`, fails a proposed replacement with `{8, 9,
10}`, and must preserve `{4, 5}` with no leaked allocation. A subsequent
successful replacement produces `{8, 9, 10}`.

**Memory rollback is not disk rollback.** Trace which guarantees survive an
exception and which cannot undo an external write. This prepares the distinction
between a rejected batch and an uncertain storage outcome.

**Outcome:** tested state replacement and a consistent error boundary.
**Work split:** error handling 0.5; replacement implementation 1.0; injected
failure and cleanup checks 0.5 hours.

### Chapter 5 — Templates, concepts, and ordering contracts

**Prerequisites:** Chapters 2–4, especially value semantics and error handling.
**Implementation: 2.0 hours.**

**Parameterize one real operation.** Introduce function templates, type
parameters, instantiation, lambdas, captures, and function objects using an
ordered merge over two small integer spans. Show why a template definition must
be available at its point of instantiation.

**Constrain the interface, test the meaning.** Add a C++20 concept for the
comparison/callable interface and explain a `requires` expression line by line.
The [concepts specification](https://eel.is/c++draft/concepts) distinguishes the
expressions a compiler can check from semantic requirements. A well-typed
comparator can still violate its ordering contract.

**Ownership crosses a generic boundary.** Return an owned result while accepting
borrowed input. Teach `typename`, overload selection, and only the forwarding
syntax actually needed; do not introduce a generic framework before a second
use exists.

**Compile and run both kinds of check.** In `exercises/chapter-05/`,
`foundation-05` merges `{1, 4}` and `{2, 4}` into `{1, 2, 4, 4}`. A separate
expected-failure compilation rejects a non-callable comparator. Runtime tests
cover equivalent keys and bad comparator behavior without calling a standard
algorithm with a knowingly invalid comparator.

**Outcome:** a complete constrained algorithm and precise generic-interface
vocabulary. **Work split:** syntax experiments 0.5; generic merge 0.75; ordering
tests 0.5; negative compilation 0.25 hours.

## Part II — Persistent storage and read views

### Chapter 6 — Records, bytes, and a dependable file boundary

**Prerequisites:** Chapters 1–5. This chapter starts the engine source tree.
**Implementation: 3.0 hours.**

**Specify the bytes.** Create `include/strata/core/` record and key types, checked
size arithmetic, and explicit integer encoders/decoders. Define relation tags,
endianness, supported record sizes, and overflow rejection with byte-by-byte
examples. Introduce `std::byte`, casts, alignment, and object representation
before using them.

**Own a file, not an integer.** Implement `io::File` in `include/strata/io/` and
`src/io/`, with move-only descriptor ownership, explicit completion operations,
and a nonthrowing destructor. Handle partial reads/writes, interrupted calls,
end-of-file, checked offsets, and native synchronization capability.

**Put failure behind a narrow seam.** Define the small file-operations interface
used by real files and deterministic test doubles. Teach runtime polymorphism
and virtual destruction here when introducing that interface. File and namespace
operations remain distinct; the seam is not a replacement filesystem project.

**Run the boundary.** Round-trip both schemas; reject truncated and oversized
records. Force two-byte transfers and a mid-write error, verify cleanup, and
inspect the resulting byte fixture independently.

**Outcome:** reusable codecs and tested native/test I/O paths, with no durability
claim from a successful ordinary write. **Work split:** encoding 1.0; file
ownership and transfer loops 1.25; fault checks and debugging 0.75 hours.

### Chapter 7 — An arena-backed skip-list memtable

**Prerequisites:** Chapters 3–6: ownership, exception guarantees, comparison,
record encoding, and checked arithmetic. **Implementation: 4.5 hours.**

**Allocate stable storage.** Implement aligned arena chunks under
`memory/arena.*`, with a byte cap, checked alignment, and prepared-allocation
rollback. Explain placement construction and explicit destruction before using
arena storage for nodes; do not use an out-of-bounds flexible array trick.

**Search at several heights.** Derive skip-list search and insertion using a
small tower trace. Implement nodes, bounded heights, deterministic test seeds,
and link updates under `storage/memtable.*`. Internal ordering is storage key
ascending, then sequence descending; comparisons do not subtract integers.

**Prepare a whole batch.** Validate duplicate keys, construct nodes and value
storage before committing links, and keep insertion publication nonthrowing.
Handle several prepared nodes whose ordering interacts. The memtable is still
single-threaded; immutable sharing comes later.

**Check structure, not just answers.** Compare randomized lookup and ordered
iteration with a test-only reference container. Exercise updates, tombstones,
height extremes, arena exhaustion, and allocation failure before publication.

**Outcome:** a learner-written ordered index that can be frozen and iterated,
plus a prepared-batch interface for the log chapter. **Work split:** arena 1.0;
index and preparation 2.5; structural tests and diagnosis 1.0 hours.

### Chapter 8 — Indexed immutable tables

**Prerequisites:** Chapters 6–7 and their codecs and ordered iterator.
**Implementation: 5.0 hours.**

**Build blocks with boundaries.** Implement `storage/table.*`: versioned file
headers, framed data blocks, checksums, and a footer locating bounded metadata.
Derive offsets and sizes from encoded bytes rather than C++ padding. Implement
the checksum and deterministic Bloom hashing in portable C++.

**Find a block without reading every block.** Build a sparse directory, binary
search it, and seek inside the selected block. Work through keys at block and
file boundaries, including repeated user keys with different sequences. A Bloom
filter may skip an absent key, never establish that a present key exists.

**Stream a table.** Connect the memtable iterator to the table builder and add a
reader that traverses blocks without loading the file. Enforce maximum file,
block, and metadata sizes before allocations or offset arithmetic.

**Break the format deliberately.** Check missing keys, empty tables, corrupted
checksums, malformed lengths, a damaged footer, and multi-block iteration. All
inserted keys pass the Bloom test; negative probes measure false positives
without asserting an exact random count.

**Outcome:** standalone indexed tables with validated point and sequential reads.
**Work split:** format/builder 1.75; directory/filter/reader 2.0; corruption and
boundary tests 1.25 hours.

### Chapter 9 — Memory budgets and a pinned block cache

**Prerequisites:** Chapters 3–5 and Chapter 8's immutable blocks.
**Implementation: 3.5 hours.**

**Charge storage before using it.** Implement `memory/budget.*` and move-only
leases, distinguishing reservation, allocation charge, and release. Cover arena
chunks, buffer capacities, and bounded container/control metadata. Any allocator
adapter used for metadata is taught here, including its allocation and ownership
contract; merely counting logical payload bytes is insufficient.

**Evict only what is unpinned.** Build `storage/block_cache.*` with an explicit
recency list, bounded lookup metadata, byte-based capacity, and pin handles.
Introduce shared ownership only where readers genuinely share one immutable
buffer; explain why shared lifetime does not imply synchronized mutation.

**A full cache is a normal condition.** Implement a charged, noncached read path
when pinned entries prevent admission. Stable file IDs prevent a stale cached
block from aliasing a newly created file. Transfer of a pin never charges the
same block twice.

**Observe the bound.** Use tiny capacities to force hits, misses, eviction,
bypass, and allocation rejection. Dropping the last pin releases the correct
charge; failed insertion does not lose the old cache entry.

**Outcome:** synchronous, budget-aware block access; threading is added in
Chapter 20. **Work split:** accounting 1.0; cache/pins 1.5; pressure tests 1.0 hours.

### Chapter 10 — Atomic batches and write-ahead recovery

**Prerequisites:** Chapters 4, 6, 7, and 9: prepared mutation, I/O, and budgets.
**Implementation: 4.5 hours.**

**Frame one commit.** Implement `storage/wal.*` and bounded batch encoding,
including lengths, sequence numbers, operation counts, and checksums. Distinguish
physical truncation from a logically complete batch. No partial replay becomes
visible.

**Cross the irreversible boundary.** Connect preparation, log append,
synchronization, and nonthrowing memtable publication. Define return states for
success, definite rejection, and uncertainty; uncertainty poisons the writer
until reopening. Allocation failure before append has a different outcome.

**Replay with a rule, not a guess.** Recover complete batches in sequence order.
Define exactly which incomplete final-frame cases may be ignored and reject
other corrupt required records. Do not resynchronize past arbitrary damage and
pretend the skipped data was uncommitted.

**Stop between transitions.** Exercise every byte truncation of a small final
batch, a complete unacknowledged batch, short I/O, and sync failure. Recovery
compares whole batches, including a batch touching both relations.

**Outcome:** a log-backed memtable harness that reopens without sorted-table or
manifest integration yet. **Work split:** framing 1.0; commit path 1.5; replay
1.0; fault checks 1.0 hours.

### Chapter 11 — Flush installation and database reopening

**Prerequisites:** Chapters 6–10 and their table, log, and budget interfaces.
**Implementation: 5.5 hours.**

**Name the authoritative state.** Implement `storage/manifest.*` with a versioned,
checksummed, bounded description of installed tables, required logs, sequence
progress, and file identities. Build and atomically replace a complete manifest
snapshot; teach the required data and namespace synchronization ordering. A
successful rename alone is not the persistence argument.

**Rotate without losing the recovery path.** Install a new active log before
acknowledging writes that depend on it. Seal and flush old memtables in generation
order, synchronize new table data/names, install the manifest change, and only
then retire superseded log dependencies. Trace crashes between these steps.

**Open the database deliberately.** Add `storage/database.*`, the directory lock,
validation of referenced files, ordered replay, and safe discovery of temporary
or orphan files. Never reuse an observed file identity after a crash or silently
fall back from a corrupt authoritative manifest to stale data.

**Run across process lifetimes.** A synchronous storage driver writes batches,
flushes, closes, and reopens. Test an empty directory, missing required table,
interrupted installation, unused temporary file, and competing directory open.

**Outcome:** a persistent single-threaded database, with synchronous maintenance
and runnable reopen tests. **Work split:** manifest 1.25; flush/rotation 1.75;
open/recovery 1.5; integration/fault checks 1.0 hours.

### Chapter 12 — Immutable snapshots and merged reads

**Prerequisites:** Chapters 7–11; especially immutable tables, pins, and file-set
publication. **Implementation: 4.5 hours.**

**Freeze at a batch boundary.** Add `storage/read_view.*`, reserve a replacement
memtable, and capture a complete immutable generation set before releasing the
old active state. Explain `shared_ptr<const T>` and the difference between
lifetime ownership and immutability. Bound both snapshots and retained sources.

**Choose the visible version.** Implement point lookup and a learner-written
cursor heap for ordered merging across frozen memtables and disk runs. Compare
sequence numbers, collapse duplicate keys, and suppress tombstones. Derive the
cursor's owner, pin lifetime, and advancement order with a duplicate-key trace.

**Keep a view unchanged.** Old views hold their original arenas/files even after
a synchronous flush publishes a newer serving set. Source replacement is not
performed inside an existing snapshot.

**Exercise the limits.** Apply the roadmap's cross-relation batch, check old and
new answers, and verify a range result across block boundaries. Exhaust retained
generation capacity, observe a no-change admission failure, release a view, and
retry successfully.

**Outcome:** consistent point/range reads and bounded snapshot ownership,
still without thread races to diagnose. **Work split:** view ownership 1.25;
merged lookup/scan 2.0; retention and visibility tests 1.25 hours.

### Chapter 13 — Streaming compaction and safe reclamation

**Prerequisites:** Chapters 8–12, including the merge cursor and manifest install.
**Implementation: 4.0 hours.**

**Select a complete older set.** Implement `storage/compaction.*` for bounded
new runs and one base run. Flush ordering establishes a selected prefix; frozen
or newly published newer sources remain outside that prefix. Define when the
selection covers all older versions relevant to dropping a tombstone.

**Merge into a replacement base.** Reuse the cursor mechanism, retain the newest
selected value per key, and split output into size-limited files with bounded
builders. Account for read buffers, output buffers, metadata, and disk headroom.

**Install before reclaiming.** Publish the new file set with the manifest
protocol. Newly acquired views use the new set; existing views do not. Remove an
obsolete file only after its last view/cache-related ownership permits it.
Failed deletions are observable and safely retried or rediscovered on reopening,
not thrown from a destructor.

**Measure the cost.** Force several flushes and a compaction with tiny limits;
verify no deleted key reappears. Keep an old view alive, inspect retained files,
then release it. Count bytes read and written, including failed output cleanup.

**Outcome:** a compacting persistent engine with observable retention and write
amplification. **Work split:** selection/merge 1.5; install/reclamation 1.25;
interaction tests and counters 1.25 hours.

## Part III — Bounded query execution

### Chapter 14 — Owned batches and a streaming query pipeline

**Prerequisites:** Chapters 2, 5, 9, and 12–13: views, generic interfaces,
reservations, and ordered snapshots. **Implementation: 3.5 hours.**

**Choose a batch layout.** Create `query/batch.*` with owned column buffers,
row counts, capacity accounting, and borrowed column spans. Compare arrays of
records with arrays of columns using actual field widths and address arithmetic.
Every borrowed view has a documented expiration point.

**Advance one operator.** Define the pull interface and separate end-of-stream
from an empty filtered batch. Implement `query/scan.*`, filtering, and projection;
introduce any new interface syntax before it appears in the pipeline.

**Make fewer bytes flow.** Retain only the fields later operators need and
compact selected rows into owned output. Include a one-row baseline for later
measurement rather than claiming that batching must win.

**Keep boundaries visible.** Run the tiny event fixture through scanning and
filtering with batch capacities one and three. Exercise empty input, all rows
filtered, exact buffer exhaustion, and a storage decoding error; no escaped view
may outlive the batch or cache pin it borrows from.

**Outcome:** a runnable, synchronous scan/filter/project executor.
**Work split:** batches/interfaces 1.0; operators 1.5; lifetime and boundary
checks 1.0 hours.

### Chapter 15 — External sorting with bounded fan-in

**Prerequisites:** Chapters 6, 9, 12, and 14: files, budgets, cursor heaps, and
batches. **Implementation: 6.0 hours.**

**Turn one budget into many runs.** Implement `query/external_sort.*`: reserve a
bounded chunk, sort it with `std::sort`, and spill an explicitly encoded run.
Include sort scratch/stack allowance and run-descriptor storage in the resource
model. Deterministic tie keys make the final ordering reproducible.

**Merge only what fits.** Derive buffer and descriptor bounds, implement buffered
run readers and the merge heap, and compute legal fan-in. If more runs exist,
perform multiple passes. Reject a reservation too small for the minimum useful
merge instead of entering a zero-progress loop.

**Own the temporary work.** A move-only run owner covers files through creation,
completed output, input retirement, failure, and later cancellation. Give spill
files their own validated format and bounded lengths. Reserve headroom for input
and output coexistence, not just the eventual final run.

**Force a second pass.** Tiny budgets produce more initial runs than fan-in.
Compare the streamed output against an independent in-memory oracle; test equal
keys, partial final blocks, missing/corrupt runs, disk-write failure, and cleanup.
A small query also exercises the no-spill path.

**Outcome:** reusable external sorting, with actual spill bytes and pass counts.
**Work split:** run generation 1.5; buffered multi-pass merge 2.25; ownership and
failure paths 1.0; integration/tests 1.25 hours.

### Chapter 16 — A streaming many-to-one merge join

**Prerequisites:** Chapters 12 and 14–15: consistent scans, batches, and sorting.
**Implementation: 5.0 hours.**

**Establish both input orders.** Sort projected events by account and event ID;
scan accounts in primary-key order from the same snapshot. Implement
`query/merge_join.*` without materializing the account relation.

**Advance the right cursor.** Derive the comparison cases for event key less
than, equal to, or greater than the account key. Repeated events reuse one
account row; unmatched events are omitted. An account stream that violates its
unique-visible-key contract is an error, not an invitation to emit duplicates.

**Pause without losing the match.** Resume across input and output batch
boundaries with explicit iterator state. Release a block pin only when no saved
view needs it; copy the small current account value when that simplifies the
lifetime. Share one reservation across overlapping operators rather than giving
each the full query budget.

**Test shape, skew, and absence.** Join with a hot account spanning many batches,
a dimension larger than a merge buffer, deleted/missing accounts, empty sides,
and a full output buffer at a key transition. Compare every output row with an
independent reference.

**Outcome:** a bounded-memory join that can consume a spilled fact stream and a
streamed dimension. **Work split:** state machine 1.5; streaming integration
1.5; budget/lifetime work 0.75; skew/boundary checks 1.25 hours.

### Chapter 17 — Grouping, checked aggregates, and complete queries

**Prerequisites:** Chapters 14–16 and the checked arithmetic from Chapter 6.
**Implementation: 3.5 hours.**

**Make groups adjacent.** Reuse external sorting for joined rows ordered by
region, day, and event ID. Explain why primary-key storage order is not grouping
order and why a second sort is required for this plan.

**Hold one group.** Implement `query/aggregate.*` with current key, count, and
sum. Emit on a key change and once at end-of-stream; empty input emits no group.
Check overflow before mutation, including at the final group.

**Connect one complete query.** Add a fixed-plan query API and a thin local
driver. Stage boundaries transfer or release reservations and temporary runs.
Results stream to an output sink instead of collecting every group in a vector.
The API reports completion separately from partial output on failure.

**Compare complete answers.** Reproduce both roadmap snapshot results, exercise
many groups and one giant group, force both sorts to spill, and test overflow.
A failed query must not leave an apparently successful final output artifact.

**Outcome:** the full analytical pipeline working synchronously against persistent
storage. **Work split:** aggregation 1.0; plan/output integration 1.25;
reference and failure checks 1.25 hours.

## Part IV — Concurrency and runtime integration

### Chapter 18 — Synchronization and thread lifetime

**Prerequisites:** Chapters 2–5. The exercise is standalone; the integrated engine
remains at Chapter 17. **Implementation: 3.0 hours.**

**Visibility needs a relationship.** Introduce `std::thread`, mutexes, lock guards,
unique locks, data races, and happens-before with two-thread traces. A shared
pointer protects lifetime, not the payload. An atomic flag is not a substitute
for protecting a multi-field transition. The
[mutex requirements](https://eel.is/c++draft/thread.mutex.requirements) explain
the relevant synchronization relation.

**Wait on state.** Build a small bounded ownership-transfer exercise in
`exercises/chapter-18/`. Teach predicate waits, spurious wakeups, and close versus
drain. The [condition-variable contract](https://eel.is/c++draft/thread.condition.condvar)
is connected to the predicate, not presented as a notification-counting service.

**Finish every thread.** Implement explicit joining and cleanup when construction
of a later worker fails. Capture worker exceptions instead of letting them escape
the thread function. Cancellation state, wakeup, and join are different actions.

**Run without sleeps as synchronization.** `foundation-18` transfers jobs valued
10, 20, and 12, drains to a sum of 42, and preserves a rejected late job in its
caller. Test close while producers and consumers are waiting, using controlled
rendezvous and a supported separate race-diagnostic build.

**Outcome:** runnable synchronization exercises before threaded engine mutation.
**Work split:** mutex/visibility programs 0.75; queue and lifecycle 1.25;
failure/wakeup checks 1.0 hours.

### Chapter 19 — Bounded execution and cooperative cancellation

**Prerequisites:** Chapters 9, 15, 17, and 18: reservations, temporary files,
complete queries, and synchronization. **Implementation: 4.0 hours.**

**A task is more than a callable.** Implement `runtime/task.*` and
`runtime/executor.*` with bounded request descriptors and explicit queued,
running, completed, failed, and canceled states. Teach `std::exception_ptr` and
completion ownership before transporting errors to callers.

**Admit an entire query.** Reserve a query's working allowance before acquiring
its snapshot and starting its operators. A queued request owns neither a full
input nor a snapshot. Jobs do not submit blocking descendants into the same
worker pool. Failure to admit has a defined retry/wait/cancel policy.

**Cancel at safe boundaries.** Implement a cancellation token and checks at
batch, block, and spill-pass boundaries. Wake resource waiters, unwind owned
runs and buffers, and report partial-output status without claiming that a
blocking filesystem operation is instantly interruptible.

**Close in an order that terminates.** Stop admission, resolve queued work,
request cancellation of running work, wake waiters, and join all workers before
destroying shared state. Test rejection ownership and partial worker startup.

**Outcome:** a bounded query executor running against an immutable/open-for-read
database fixture, before concurrent writes are enabled. **Work split:** task
states/pool 1.25; admission/cancellation 1.25; completion/shutdown/fault tests
1.5 hours.

### Chapter 20 — Concurrent readers, ingestion, and maintenance

**Prerequisites:** Chapters 11–13 and 18–19; all storage and query paths already
run synchronously. **Implementation: 3.5 hours.**

**Publish a whole serving state.** Add `runtime/engine.*` coordination around
writer commits, snapshot freezing, and manifest/file-set changes. State what
each mutex protects and when a captured maintenance plan must be revalidated.
Never overwrite newer state while installing older compaction output.

**Preserve maintenance progress.** Move ordered flush and compaction work to a
dedicated worker with its own minimum buffers and descriptors. Keep writer
backpressure and snapshot pressure waits outside the locks required to relieve
them. Long-lived snapshots may deliberately stall admission; they do not license
extra uncharged generations.

**Synchronize shared infrastructure.** Extend budgets, cache recency/pins, file
retirement, and counters to concurrent access without holding global locks over
bulk I/O. Avoid duplicate cache admission races and preserve the pinned-cache
bypass guarantee.

**Exercise actual overlap.** Use deterministic pause points to hold an old query,
commit the cross-relation update, publish a compaction, and cancel another query.
Verify old/new answers and then shut down while a writer is backpressured.
Maintenance failures reach the engine's visible error state.

**Outcome:** concurrent reads, ingestion, and maintenance with tested ownership
and lock order. **Work split:** publication/locking 1.25; maintenance/backpressure
1.0; controlled overlap and shutdown tests 1.25 hours.

## Part V — Integrated workloads, failures, and measurements

### Chapter 21 — Reproducible workloads and independent answers

**Prerequisites:** Chapters 1–20, especially the complete query and runtime APIs.
**Implementation: 4.5 hours.**

**Generate cases that exercise the engine.** Write `scripts/generate_data.py`
with a seed, controlled account skew, day selectivity, updates, tombstones, and
bounded batches. Keep small exact fixtures separate from larger streaming data.
Input size is defined in logical encoded bytes, not inferred from CSV text size.

**Compute answers independently.** Write `scripts/reference_query.py` using
straightforward Python state for small snapshots. Preserve commit boundaries,
unsigned ranges, join semantics, output order, and overflow rules. Use deterministic
checkable aggregates for larger data without requiring an unbounded Python copy
of the full database.

**Expose a thin driver.** Complete `apps/strata.cpp` with only the commands needed
to generate/import, mutate, query, compact, inspect counters, and reopen. Add an
in-process scenario runner for concurrent behavior; a directory lock means two
separate writer processes are not the concurrency demonstration.

**Make the run diagnosable.** Add fixed tiny-budget presets that force flush,
compaction, two merge passes, retention pressure, and cancellation. Preflight
storage space and resource limits. Report a failed stage with its seed and
parameters, not an unexplained final mismatch.

**Outcome:** repeatable correctness workloads and a single-machine end-to-end
scenario. **Work split:** generator/oracle 1.5; driver/scenario 1.25;
interaction checks 1.0; diagnosis/fixtures 0.75 hours.

### Chapter 22 — Recovery and resource-pressure experiments

**Prerequisites:** Chapters 6, 10–13, and 19–21, including failure seams and the
scenario driver. **Implementation: 4.0 hours.**

**Model persistence explicitly.** Extend the existing I/O double with separate
volatile and stable file bytes and namespace state. Select deterministic crash
images around append, sync, rename, manifest installation, and deletion; permit
specified unsynchronized writes to persist or disappear. The model's assumptions
are documented, not mistaken for all real filesystems.

**Test acknowledgement rather than convenience.** Track acknowledged and
uncertain batches externally to the crashed instance. After reopening, require
all acknowledged batches and atomic inclusion or exclusion of permitted uncertain
batches. A simulator that merely forgets every unsynchronized byte is not the
only failure image exercised.

**Use the native process boundary as well.** Add child-process stop points and
reopen tests. Cover damaged required files separately from valid truncated tails,
orphan output, injected no-space/sync failures, allocation pressure, and
cleanup after canceled spill. These native tests do not certify physical
power-loss behavior.

**Diagnose one failure chain.** Trace a failed installation from file ownership
to manifest state to replay decisions. Check that pressure errors terminate or
backpressure as specified without losing resources or acknowledged data.

**Outcome:** deterministic persistence, native process-stop, and pressure tests
with distinct meanings. **Work split:** persistence-model extension 1.25;
crash/reopen scenarios 1.25; resource interactions and diagnosis 1.5 hours.

### Chapter 23 — Profiling, optimization, and the final demonstration

**Prerequisites:** Chapters 1–22, with correctness, recovery, and concurrency
checks passing on the chosen environment. **Implementation: 4.0 hours.**

**Measure a specified workload.** Use `scripts/measure.py` to separate generation,
loading, querying, compaction, and recovery. Capture build flags, parameters,
seed, warmup policy, repeats, elapsed time, counters, charged peaks, and observed
resident memory. Explain the platform's memory units rather than assuming a
Linux timing format. Sanitized and optimized runs are different experiments.

**Find where work goes.** Profile a representative slow path with available
native tools and distinguish CPU work, allocation, file I/O, cache behavior, and
lock waiting. Define percentile calculations and sample counts; a handful of
queries is not a credible tail-latency distribution. Report warm/cold conditions
honestly without pretending the OS cache was cleared.

**Change one implementation decision.** Compare the one-row path with batching
and compare wider intermediate records with early projection/filtering on the
same data and result. Explain measured spill-volume changes and any regression.
Tune one demonstrated bottleneck, rerun correctness checks, and keep the simple
baseline available. No minimum speedup is required.

**Run the combined demonstration.** Exercise overlapping ingestion, old and new
queries, forced spill, compaction, cancellation, crash/reopen, and reclamation.
Present the actual answer checks and measurements with limits on what they show.

**Outcome:** the integrated engine, a reproducible demonstration, and a measured
optimization with an explained tradeoff. **Work split:** measurement harness
1.0; profiling/optimization 1.25; repeated comparisons 1.0; final integration
0.75 hours. The 6.5-hour course reserve remains available for additional debugging.

## The final demonstration

The demonstration uses the same engine for two scales. The tiny fixture has exact
human-readable answers and tiny limits that force otherwise rare transitions.
The larger fixture starts around **512 MiB–1 GiB of logical input**, with the
256 MiB managed budget and 32 MiB query allowance described above. Adjust sizes
to the available disk and observed runtime, while keeping the projected fact
stream larger than its sort allowance. A large file that filters down to one
small batch does not demonstrate external sorting. Include an account-stream
case larger than the join's working buffers to exercise dimension streaming.

Begin with a repeatable load and a snapshot query. Retain that view while a
bounded writer applies updates and deletions. Run new queries and background
compaction, force a second query to create spill files, and cancel it at a
controlled boundary. Check that its resources return while the older query's
resources remain valid. Release the old view and observe reclamation. Separately
inject a persistence/process failure, reopen, and compare whole recovered batches
and query answers with the permitted states from the failure scenario.

The final measurements include ingest throughput, query latency with sample
counts, block reads and cache misses, spill bytes and merge passes, compaction
bytes relative to logical input bytes, peak charged allocation, reservations,
and separately observed process memory. Report the actual tested configuration
and failures encountered. A slower optimized variant is a result to explain, not
a reason to change the workload until it looks faster.

This system does not include a SQL parser, arbitrary schemas, a network protocol,
replication, multiple concurrent writers, general transaction isolation,
lock-free indexing, arbitrary many-to-many joins, compression, or a cost-based
optimizer. Those exclusions keep the implemented storage, recovery, external
execution, and resource-lifetime mechanisms substantial within the stated budget.

## Technical references

These sources were consulted for the roadmap. The chapter explanations develop
the necessary mechanisms rather than assigning links in place of teaching.
The public C++ working draft evolves beyond C++20; only the facilities specified
in the chapter text are part of the course baseline. Apple archived manual pages
are background to verify against the installed platform, not evidence of its
exact SDK or filesystem behavior.

- **Language and native tool support:** [Apple C++ language support](https://developer.apple.com/xcode/cpp/),
  [libc++ C++20 status](https://libcxx.llvm.org/Status/Cxx20.html), and
  [CMake compile features — requiring language standards](https://cmake.org/cmake/help/latest/manual/cmake-compile-features.7.html#requiring-language-standards).
- **Lifetime, interfaces, and synchronization:** [object lifetime](https://eel.is/c++draft/basic.life),
  [concepts](https://eel.is/c++draft/concepts),
  [mutex requirements](https://eel.is/c++draft/thread.mutex.requirements), and
  [condition-variable wait](https://eel.is/c++draft/thread.condition.condvar).
- **Storage structure and failure reasoning:** [LevelDB implementation description](https://chromium.googlesource.com/external/leveldb/+/HEAD/doc/impl.md),
  [Apple archived `fsync` manual](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html), and
  [SQLite atomic commit, particularly sections 2 and 8](https://www.sqlite.org/atomiccommit.html).
- **Diagnostics:** [Clang AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html)
  and [Clang ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html).
