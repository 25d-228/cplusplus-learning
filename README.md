# C++ Systems Engineering: Strata

Build an embedded storage-and-query engine in C++20, from resource ownership to
crash recovery, external-memory execution, and concurrent maintenance.

Strata stores two fixed numeric relations and executes filtered join/aggregation
queries. You implement the arena-backed skip list, indexed sorted tables, pinned
block cache, write-ahead log, atomic batches, immutable snapshots, compaction,
external sorting, merge join, and bounded query runtime. Database and executor
libraries do not provide these mechanisms.

## Read the book

<!-- course-links:start -->
- [Roadmap — Strata: a C++ storage and query engine](https://25d-228.github.io/cplusplus-learning/index.html)
- [Chapter 1 — From C files to a native C++ program](https://25d-228.github.io/cplusplus-learning/chapter-01.html)
- [Chapter 2 — Values, references, containers, and views](https://25d-228.github.io/cplusplus-learning/chapter-02.html)
- [Chapter 3 — Lifetime, RAII, and move-only owners](https://25d-228.github.io/cplusplus-learning/chapter-03.html)
- [Chapter 4 — Errors and failure-safe state changes](https://25d-228.github.io/cplusplus-learning/chapter-04.html)
- [Chapter 5 — Templates, concepts, and ordering contracts](https://25d-228.github.io/cplusplus-learning/chapter-05.html)
- [Chapter 6 · Part 1 — Records, bytes, and a dependable file boundary](https://25d-228.github.io/cplusplus-learning/chapter-06.html)
- [Chapter 6 · Part 2 — Deterministic faults and native checks](https://25d-228.github.io/cplusplus-learning/chapter-06-part-2.html)
- [Chapter 7 · Part 1 — An arena-backed skip-list memtable](https://25d-228.github.io/cplusplus-learning/chapter-07.html)
- [Chapter 7 · Part 2 — Prepare, publish, and verify](https://25d-228.github.io/cplusplus-learning/chapter-07-part-2.html)
- [Chapter 8 · Part 1 — Indexed immutable tables](https://25d-228.github.io/cplusplus-learning/chapter-08.html)
- [Chapter 8 · Part 2 — Seek, verify, and diagnose](https://25d-228.github.io/cplusplus-learning/chapter-08-part-2.html)
- [Chapter 8 · Part 3 — Verification and native checks](https://25d-228.github.io/cplusplus-learning/chapter-08-part-3.html)
- [Chapter 9 · Part 1 — Memory budgets and a pinned block cache](https://25d-228.github.io/cplusplus-learning/chapter-09.html)
- [Chapter 9 · Part 2 — Recency, pins, and admission](https://25d-228.github.io/cplusplus-learning/chapter-09-part-2.html)
- [Chapter 9 · Part 3 — Reader integration and pressure tests](https://25d-228.github.io/cplusplus-learning/chapter-09-part-3.html)
- [Chapter 10 · Part 1 — Atomic batches and write-ahead recovery](https://25d-228.github.io/cplusplus-learning/chapter-10.html)
- [Chapter 10 · Part 2 — Commit, uncertainty, and strict replay](https://25d-228.github.io/cplusplus-learning/chapter-10-part-2.html)
- [Chapter 10 · Part 3 — Failure images, pressure, and native checks](https://25d-228.github.io/cplusplus-learning/chapter-10-part-3.html)
- [Chapter 11 · Part 1 — Flush installation and database reopening](https://25d-228.github.io/cplusplus-learning/chapter-11.html)
- [Chapter 11 · Part 2 — Install, rotate, flush, and reopen](https://25d-228.github.io/cplusplus-learning/chapter-11-part-2.html)
- [Chapter 11 · Part 3 — Failure cases across process lifetimes](https://25d-228.github.io/cplusplus-learning/chapter-11-part-3.html)
- [Chapter 12 · Part 1 — Immutable snapshots and merged reads](https://25d-228.github.io/cplusplus-learning/chapter-12.html)
- [Chapter 12 · Part 2 — Merge versions without borrowing dead bytes](https://25d-228.github.io/cplusplus-learning/chapter-12-part-2.html)
- [Chapter 12 · Part 3 — Retention, failure, and ownership checks](https://25d-228.github.io/cplusplus-learning/chapter-12-part-3.html)
- [Chapter 13 · Part 1 — Streaming compaction and safe reclamation](https://25d-228.github.io/cplusplus-learning/chapter-13.html)
- [Chapter 13 · Part 2 — Install new owners, then reclaim old names](https://25d-228.github.io/cplusplus-learning/chapter-13-part-2.html)
- [Chapter 13 · Part 3 — Retention, bounds, and restart checks](https://25d-228.github.io/cplusplus-learning/chapter-13-part-3.html)
- [Chapter 14 · Part 1 — Owned batches and a streaming query pipeline](https://25d-228.github.io/cplusplus-learning/chapter-14.html)
- [Chapter 14 · Part 2 — Pull without confusing emptiness and completion](https://25d-228.github.io/cplusplus-learning/chapter-14-part-2.html)
- [Chapter 14 · Part 3 — Boundary, retention, and refill checks](https://25d-228.github.io/cplusplus-learning/chapter-14-part-3.html)
- [Chapter 15 · Part 1 — External sorting with bounded fan-in](https://25d-228.github.io/cplusplus-learning/chapter-15.html)
- [Chapter 15 · Part 2 — Merge only what fits](https://25d-228.github.io/cplusplus-learning/chapter-15-part-2.html)
- [Chapter 15 · Part 3 — Pressure, corruption, and the native run oracle](https://25d-228.github.io/cplusplus-learning/chapter-15-part-3.html)
- [Chapter 16 · Part 1 — A streaming many-to-one merge join](https://25d-228.github.io/cplusplus-learning/chapter-16.html)
- [Chapter 16 · Part 2 — Resume without borrowing stale rows](https://25d-228.github.io/cplusplus-learning/chapter-16-part-2.html)
- [Chapter 16 · Part 3 — Skew, invalid streams, and snapshot checks](https://25d-228.github.io/cplusplus-learning/chapter-16-part-3.html)
- [Chapter 17 · Part 1 — Grouping, checked aggregates, and complete queries](https://25d-228.github.io/cplusplus-learning/chapter-17.html)
- [Chapter 17 · Part 2 — Keep one group; publish one complete result](https://25d-228.github.io/cplusplus-learning/chapter-17-part-2.html)
- [Chapter 17 · Part 3 — Boundary, failure, and two-sort checks](https://25d-228.github.io/cplusplus-learning/chapter-17-part-3.html)
- [Chapter 18 · Part 1 — Synchronization and thread lifetime](https://25d-228.github.io/cplusplus-learning/chapter-18.html)
- [Chapter 18 · Part 2 — Join before destroying shared state](https://25d-228.github.io/cplusplus-learning/chapter-18-part-2.html)
- [Chapter 19 · Part 1 — Bounded execution and cooperative cancellation](https://25d-228.github.io/cplusplus-learning/chapter-19.html)
- [Chapter 19 · Part 2 — A terminal result owns no running query](https://25d-228.github.io/cplusplus-learning/chapter-19-part-2.html)
- [Chapter 19 · Part 3 — Force the waiting and publication boundaries](https://25d-228.github.io/cplusplus-learning/chapter-19-part-3.html)
- [Chapter 20 · Part 1 — Concurrent readers, ingestion, and maintenance](https://25d-228.github.io/cplusplus-learning/chapter-20.html)
- [Chapter 20 · Part 2 — Build outside the lock; install against current authority](https://25d-228.github.io/cplusplus-learning/chapter-20-part-2.html)
- [Chapter 20 · Part 3 — Force publication, failure, and shutdown boundaries](https://25d-228.github.io/cplusplus-learning/chapter-20-part-3.html)
- [Chapter 21 · Part 1 — Reproducible workloads and independent answers](https://25d-228.github.io/cplusplus-learning/chapter-21.html)
- [Chapter 21 · Part 2 — Parse bounded input; drive the existing engine](https://25d-228.github.io/cplusplus-learning/chapter-21-part-2.html)
- [Chapter 21 · Part 3 — Force old, new, canceled, and reopened answers](https://25d-228.github.io/cplusplus-learning/chapter-21-part-3.html)
- [Chapter 22 · Part 1 — Recovery and resource-pressure experiments](https://25d-228.github.io/cplusplus-learning/chapter-22.html)
- [Chapter 22 · Part 2 — Recover an allowed history, not a convenient answer](https://25d-228.github.io/cplusplus-learning/chapter-22-part-2.html)
- [Chapter 22 · Part 3 — Kill the process; retain the observer](https://25d-228.github.io/cplusplus-learning/chapter-22-part-3.html)
- [Chapter 23 · Part 1 — Profiling, optimization, and the final demonstration](https://25d-228.github.io/cplusplus-learning/chapter-23.html)
- [Chapter 23 · Part 2 — Remove work without changing the answer](https://25d-228.github.io/cplusplus-learning/chapter-23-part-2.html)
- [Chapter 23 · Part 3 — Verify, repeat, and explain the result](https://25d-228.github.io/cplusplus-learning/chapter-23-part-3.html)
<!-- course-links:end -->

The complete 23-chapter book and roadmap provide prerequisites, section
outlines, runnable outcomes, and estimates. Five language-foundation chapters
precede the engine; a synchronization chapter precedes threaded integration.
Planned implementation is **86 hours plus 6.5 hours of debugging/integration
reserve**. Reading is excluded.

| Sequence | Focus |
|---|---|
| Chapters 1–5 | Native builds, values/views, lifetime, RAII/moves, errors, templates/concepts |
| Chapters 6–13 | Byte layouts, indexing/cache, logging/recovery, snapshots, compaction |
| Chapters 14–17 | Batches, external sorting, streaming join and aggregation |
| Chapters 18–20 | Synchronization, bounded execution, cancellation, concurrent maintenance |
| Chapters 21–23 | Reproducible workloads, failure experiments, profiling, final demonstration |

## Workspace

Use a compatible native Apple Clang/libc++ or LLVM toolchain on ARM64 macOS.
Chapter 1 checks the actual required features rather than requiring a particular
compiler banner or macOS release. CMake/CTest, native debugging tools, and Python
fixtures support the work; Python does not implement the engine.

`main` is this concise entry point. `gh-pages` holds the canonical book and
website. Keep learner implementation on the persistent **`work/strata`** branch;
do not merge the website branch into it. See [AGENTS.md](AGENTS.md) for workspace
and delegated-assistance rules.

The final demonstration combines ingestion, repeatable snapshot queries, forced
spill, compaction, cancellation, crash/reopen checks, and measured memory/I/O
behavior on a 16 GB M4 Mac mini. The scope excludes SQL parsing, networking,
distribution, and multiple writers.
