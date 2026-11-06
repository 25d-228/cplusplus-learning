# Chapter 18 — Synchronization and thread lifetime

## Part 1 — Visibility and bounded ownership transfer

An owning batch can remain alive while two threads corrupt its contents. A queue
can protect every pointer with a mutex and still lose accepted work on shutdown.
A worker can finish its calculation and leave a joinable thread handle whose
destruction terminates the program. These are three different failures: unsafe
access, incorrect state transitions, and incomplete lifetime management.

We will make those differences observable before introducing threads into Strata.
The final exercise transfers jobs valued `10`, `20`, and `12`, drains to a sum of
42, and leaves a rejected late job valued `99` with its caller. It also stops
blocked producers and consumers, transports worker exceptions, and cleans up when
a later worker cannot be started. The queue is a teaching component, not the
engine's query scheduler.

> **Learning objectives**
> Trace the synchronization that makes a value visible; protect a multi-field
> invariant; distinguish notification from permission; transfer unique ownership
> through a bounded ring; separate draining from cancellation; and join every
> started thread before destroying anything it can access.

**Prerequisites:** the runnable programs and CMake setup from
[Chapters 1–5](https://25d-228.github.io/cplusplus-learning/chapter-01.html),
especially borrowed references, RAII, moves, exceptions, and lambda captures.
The engine remains exactly at
[Chapter 17](https://25d-228.github.io/cplusplus-learning/chapter-17.html).
Nothing in this chapter makes its budgets, cache, or database safe for concurrent
calls. The exercises include no engine headers and modify no engine source.

**Implementation: 3.0 hours across both parts.** Allocate approximately 0.75 hours
to visibility and mutex programs, 1.25 to queue and thread ownership, and 1.0 to
failure/wakeup checks and diagnosis. Reading is separate. These are working
estimates, not minimum time requirements.

Create `exercises/chapter-18/` in the learner workspace. The two parts supply
these complete files:

```text
exercises/chapter-18/
  visibility.cpp       # A separate runnable visibility program.
  job_queue.hpp
  job_queue.cpp
  workers.hpp
  workers.cpp
  main.cpp             # Ownership, shutdown, and failure exercises.
  racy_counter.cpp     # Isolated, intentionally incorrect diagnostic.
CMakeLists.txt          # Append the target definitions in Part 2.
```

For each named file, concatenate its filename-labelled C++ blocks in reading
order, with a blank line between blocks. Every function and header is supplied.
The diagnostic is excluded from the normal build and test suite. Part 1 develops
the visible-state and queue contracts; [Part 2](https://25d-228.github.io/cplusplus-learning/chapter-18-part-2.html)
completes the worker owner, tests, and build commands.

## A thread is not a delayed function call

Constructing a `std::thread` starts a separate execution of its callable. The
constructing thread continues independently; it does not wait for that callable
to return. A lambda such as `[&] { result = input + 22; }` stores references to the
surrounding variables it uses. Copying the callable does not extend those
variables' lifetimes. We must keep them alive until the worker is finished.

The first program has one writer of `result` and reads it only after joining.
Create the beginning of `visibility.cpp`:

```cpp title="exercises/chapter-18/visibility.cpp"
#include <atomic>
#include <cstdint>
#include <exception>
#include <iostream>
#include <memory>
#include <mutex>
#include <stdexcept>
#include <thread>
#include <version>

#if !defined(__cpp_lib_atomic_wait) || __cpp_lib_atomic_wait < 201907L
#error "This exercise needs the C++20 standard-library atomic wait/notify feature."
#endif

namespace {
void require(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}

void check_start_and_join() {
    int input = 20;
    int result = 0;
    std::thread worker([&] { result = input + 22; });
    worker.join();
    require(result == 42, "joined result differs");
}
```

The unnamed namespace keeps these helpers local to this translation unit.
`require` throws on a failed check even in a Release build. The feature check
asks for the C++20 standard-library **atomic wait/notify** facility used below;
a compiler's acceptance of `-std=c++20` alone is not that library test. The
[`<version>` feature-test interface](https://eel.is/c++draft/version.syn)
lets us check the actual library rather than a compiler banner. An appropriate
native Apple or LLVM toolchain is sufficient; no particular macOS release is
required here.

The worker captures `input` and `result` by reference. Its expression cannot
throw in this fixture. `worker.join()` waits for completion; after a successful
join the handle is no longer joinable. We do not read `result` while the worker
might be writing it. Thread construction and joining provide synchronization,
rather than a guess about how quickly the worker runs. See
[thread construction](https://eel.is/c++draft/thread.thread.constr) and
[join](https://eel.is/c++draft/thread.thread.member).

Write $A \prec B$ for an operation sequenced before another in one thread, and
$A \rightarrow_{\mathrm{sync}} B$ for a synchronization edge between threads.
Combining those edges gives a happens-before relationship. Here the useful path
is:

$$
\text{write result}
\prec \text{worker completion}
\rightarrow_{\mathrm{sync}} \text{join returns}
\prec \text{read result}.
$$

The labels name operations, not elapsed times. Reading after a long sleep would
not establish that path. A join performed **after** an unsynchronized conflicting
read would not repair the earlier read either.

A plain increment is a read followed by computation and a write. Imagining two
threads both reading 0 and both writing 1 explains a possible lost-update
symptom, but is not a specification for an incorrect C++ program. Potentially
concurrent conflicting accesses, at least one non-atomic, without the required
happens-before ordering constitute a data race and produce undefined behavior.
The language's [data-race rule](https://eel.is/c++draft/intro.races) is stronger
than “the answer may be slightly inaccurate.” `volatile` does not supply the
missing cross-thread relationship.

## Protect the relationship between fields

Our next object records a count and a sum. Every accepted contribution is two,
so the representation invariant is

$$
S = 2C,
$$

where $C$ is `count` and $S$ is `sum`. Each of two threads contributes 5,000 times;
the final values must be $C=10{,}000$ and $S=20{,}000$. These fixture bounds fit
comfortably in `uint64_t`. We want readers to see the pair before or after a
contribution, never halfway through it.

Before the next listing, introduce a small exception-transport mechanism.
`std::exception_ptr` is a nullable owner referring to an exception object.
Inside `catch (...)`, `std::current_exception()` captures the current exception.
After joining, `std::rethrow_exception()` raises it in the controlling thread.
A worker's exception does not automatically propagate through `join`. An uncaught
exception escaping the thread callable invokes termination. The
[exception propagation interface](https://eel.is/c++draft/propagation) and the
[thread constructor contract](https://eel.is/c++draft/thread.thread.constr)
describe these separate mechanisms.

Append the complete ledger and its check:

```cpp title="exercises/chapter-18/visibility.cpp"
struct Ledger {
    std::mutex mutex;
    std::uint64_t count = 0;
    std::uint64_t sum = 0;
};

void add_twos(Ledger& ledger) {
    for (unsigned i = 0; i < 5000; ++i) {
        std::lock_guard lock(ledger.mutex);
        ++ledger.count;
        ledger.sum += 2;
        require(ledger.sum == 2 * ledger.count, "multi-field invariant broken");
    }
}

void check_mutex() {
    auto ledger = std::make_shared<Ledger>();
    std::exception_ptr worker_error;
    std::thread worker([ledger, &worker_error] {
        try { add_twos(*ledger); }
        catch (...) { worker_error = std::current_exception(); }
    });
    try {
        add_twos(*ledger);
    } catch (...) {
        worker.join();
        throw;
    }
    worker.join();
    if (worker_error) { std::rethrow_exception(worker_error); }
    require(ledger->count == 10000 && ledger->sum == 20000,
            "protected ledger differs");
}
```

`std::mutex` gives exclusive access. `std::lock_guard lock(ledger.mutex)` acquires
it immediately and releases it when the guard's scope ends, including exception
unwinding. Class template argument deduction infers `lock_guard<std::mutex>`.
The guard is named: a temporary guard destroyed at the end of its declaration
would not protect the later statements.

All modifications and the in-loop invariant check use **the same mutex**. For
example, thread A changes `(7,14)` to `(8,16)` while holding it. Thread B cannot
enter its guarded region and inspect `(8,14)`. A prior unlock synchronizes with a
subsequent successful lock of that mutex; this supplies visibility as well as
exclusion. See the [mutex requirements](https://eel.is/c++draft/thread.mutex.requirements).

The worker captures `ledger` **by value**, creating another `shared_ptr` handle,
and `worker_error` **by reference**. Each handle keeps the same `Ledger` alive.
The reference count does not lock the pointed-to fields. The mutex, not the
shared ownership, makes the updates safe. Similarly, concurrent mutation of the
same ordinary `shared_ptr` handle would require its own synchronization; separate
handles sharing a control block are a different case. The
[shared-pointer specification](https://eel.is/c++draft/util.smartptr.shared)
distinguishes handle operations from access to the pointed-to object.

Only the worker writes `worker_error`; the caller reads it after joining. No
extra mutex is needed for that access pattern. If the caller's own loop throws,
its catch joins before rethrowing. The worker catches its own error and exits, so
the caller never waits for an exception to magically cross the thread boundary.
In the normal path both loops have ended before the final unguarded ledger read.
If both loops failed, this simple fixture reports the caller's exception first.

Changing both fields to atomics would remove races on the individual fields,
but two separate atomic stores still would not be one pair update. Even with
strong ordering, a reader could observe one new field and one old field. For
this representation a single mutex is the direct solution. Lock-free algorithms
are not a prerequisite for correct concurrent software.

## A one-shot flag can publish immutable data

An atomic is useful when its precise contract matches the problem. The next
fixture has one writer, one reader, and one transition from not-ready to ready.
After publication, the writer never changes the payload again.

Append the final portion of `visibility.cpp`:

```cpp title="exercises/chapter-18/visibility.cpp"
struct Published {
    std::uint64_t count = 0;
    std::uint64_t sum = 0;
};

void check_one_shot_publication() {
    Published payload;
    std::atomic<bool> ready{false};
    std::thread writer([&] {
        payload = {3, 42};
        ready.store(true, std::memory_order_release);
        ready.notify_one();
    });
    ready.wait(false, std::memory_order_acquire);
    const Published observed = payload; // Before join: acquire is essential.
    writer.join();
    require(observed.count == 3 && observed.sum == 42,
            "published pair differs");
}
} // namespace

int main() {
    try {
        check_start_and_join();
        check_mutex();
        check_one_shot_publication();
        std::cout << "joined=42, ledger=(10000,20000), published=(3,42)\n";
        return 0;
    } catch (const std::exception& error) {
        std::cerr << "foundation-18-visibility: " << error.what() << '\n';
        return 1;
    }
}
```

`std::atomic<bool>` makes operations on `ready` atomic. The explicit release
store publishes preceding writes; an acquire observation of that stored value
orders subsequent reads. The
[release/acquire rule](https://eel.is/c++draft/atomics.order) supplies the edge:

$$
\text{write payload}
\prec \text{store true, release}
\rightarrow_{\mathrm{sync}} \text{observe true, acquire}
\prec \text{copy payload}.
$$

`ready.wait(false, acquire)` checks the atomic value and waits while it still
compares equal to `false`. Its return follows observation of a different value;
our only other value is the single published `true`. `notify_one` wakes a waiter
to check; it is not publication by itself. If the store happens first, the
wait observes the changed value without needing a remembered notification. See
[atomic wait and notification operations](https://eel.is/c++draft/atomics.types.operations).

Notice that `observed` is copied **before** joining the writer. The acquire is
therefore doing real work. Joining first would make this a poor demonstration
because join would independently order the payload read. The check that can
throw still occurs after join, keeping the reference captures alive safely.

Replacing acquire/release with relaxed operations would leave the non-atomic
payload without this publication relationship. Resetting the flag and reusing
the payload would introduce a different protocol: the writer could overwrite
fields while a previous reader uses them. This example is deliberately not a
reusable mailbox or an engine-generation publication algorithm.

This small program can be compiled before finishing the rest of the chapter:

```sh
mkdir -p build/chapter-18
xcrun clang++ -std=c++20 -Wall -Wextra -Wpedantic -pthread \
  exercises/chapter-18/visibility.cpp -o build/chapter-18/visibility
build/chapter-18/visibility
```

Expected output:

```text
joined=42, ledger=(10000,20000), published=(3,42)
```

Use the equivalent native compiler command for an already configured LLVM
toolchain. A missing atomic-wait feature is a standard-library/toolchain issue,
not a reason to remove synchronization or substitute sleeps.

## Define ownership before defining the ring

We now transfer `unique_ptr<Job>` objects through a bounded queue. A successful
push empties the caller's handle. A rejected push leaves that exact handle and
job intact. Pop transfers ownership to one consumer. Empty jobs are forbidden,
so an empty pop result can mean that this queue will deliver no further work.

The caller must exclusively own its handle during `push`; the queue's mutex
cannot protect some other thread concurrently resetting that same caller handle.
The job's bytes are initialized before submission and become consumer-owned after
pop. No participant keeps a mutable alias and continues writing after transfer.

The queue has three states:

| State | New pushes | Existing queued jobs | Empty pop |
|---|---|---|---|
| `open` | Wait for a free slot, then transfer | Available to consumers | Wait for state or work |
| `draining` | Reject without moving | Still available | Return empty after drain |
| `canceled` | Reject without moving | Discard outside the mutex | Return empty |

Close means stop admitting and **drain**. Cancel means stop admitting and discard
jobs not yet removed. Neither action joins a thread, and neither can revoke a
job that a worker already owns. Calling close after cancel cannot reopen or
restore the queue. Repeated close or cancel is permitted.

Create the beginning of `job_queue.hpp`:

```cpp title="exercises/chapter-18/job_queue.hpp"
#ifndef STRATA_EXERCISES_CHAPTER_18_JOB_QUEUE_HPP
#define STRATA_EXERCISES_CHAPTER_18_JOB_QUEUE_HPP

#include <atomic>
#include <condition_variable>
#include <cstddef>
#include <cstdint>
#include <memory>
#include <mutex>
#include <vector>

namespace foundation18 {

// Exercise-only lifetime counter, not a memory budget.
struct Job {
    explicit Job(std::uint64_t value, bool fail = false) noexcept;
    ~Job();
    Job(const Job&) = delete;
    Job& operator=(const Job&) = delete;
    std::uint64_t value;
    bool fail;
    static std::size_t live() noexcept;
private:
    inline static std::atomic<std::size_t> live_{0};
};

enum class QueueState { open, draining, canceled };

struct QueueProbe {
    QueueState state;
    std::size_t queued;
    std::size_t producers;
    std::size_t consumers;
    std::uint64_t predicate_checks;
};
```

The job's `fail` field is a deterministic processing failure used in Part 2.
`Job` itself is not copied or moved; its `unique_ptr` is moved. `inline static`
gives the exercise one shared counter definition across translation units.
Atomic relaxed increments/decrements count live jobs without claiming to publish
their contents. We check final zero after all owner scopes and joins have ended.
This is not the engine's allocation accounting.

`QueueProbe` is a copy of observable queue state. `producers` and `consumers`
count operations registered in a predicate wait; `predicate_checks` lets a test
establish that a false condition was checked again after an extra notification.
These are exercise diagnostics, not a scheduling interface.

Complete the header:

```cpp title="exercises/chapter-18/job_queue.hpp"
class JobQueue {
public:
    explicit JobQueue(std::size_t capacity);
    JobQueue(const JobQueue&) = delete;
    JobQueue& operator=(const JobQueue&) = delete;

    // Rejection never consumes job. The caller exclusively owns its handle.
    bool push(std::unique_ptr<Job>& job);
    std::unique_ptr<Job> pop();
    void close();
    void cancel();

    // Observability for deterministic exercises, not a production queue API.
    QueueProbe probe();
    void await_waiters(std::size_t producers, std::size_t consumers,
                       std::uint64_t minimum_checks = 0);
    void notify_without_change();

private:
    static std::size_t checked_capacity(std::size_t capacity);
    std::size_t advance(std::size_t index) const noexcept;
    void wait_for_space(std::unique_lock<std::mutex>& lock);
    void wait_for_item(std::unique_lock<std::mutex>& lock);

    const std::size_t capacity_;
    std::mutex mutex_;
    std::condition_variable space_;
    std::condition_variable item_;
    std::condition_variable observed_;
    std::vector<std::unique_ptr<Job>> slots_;
    std::size_t read_ = 0;
    std::size_t write_ = 0;
    std::size_t count_ = 0;
    QueueState state_ = QueueState::open;
    std::size_t producers_ = 0;
    std::size_t consumers_ = 0;
    std::uint64_t checks_ = 0;
};

} // namespace foundation18
#endif
```

`std::unique_lock<std::mutex>` is an owning lock wrapper that also permits
explicit unlocking and relocking. A condition-variable wait needs that ability:
it must let another thread acquire the mutex and change the condition. A
`lock_guard` deliberately has no such interface and suits shorter, non-waiting
critical sections.

The mutex protects **all** ring slots, indices, count, state, and probe counters.
The fixed capacity never changes. The condition variables do not separately own
the state. `space_` serves producers, `item_` serves consumers, and `observed_`
serves the test's state rendezvous. No queue operation recursively takes the
mutex; helpers receive an already-held lock.

For capacity $C$, queued count $N$, next read slot $R$, and next write slot $W$,
the live ring obeys

$$
0 \le N \le C,\qquad 0 \le R < C,\qquad 0 \le W < C.
$$

The $N$ slots reached from $R$ by wrapped increments own jobs; the others are
empty. Equal read/write indices alone cannot distinguish empty from full, so we
keep the count. In the canceled state the representation is intentionally
simpler: the slot vector is empty, $N=0$, and every operation checks cancellation
before touching a slot.

With capacity two, follow the actual indices:

| Operation | Slots afterward | Read | Write | Count |
|---|---|---:|---:|---:|
| Start | empty, empty | 0 | 0 | 0 |
| Push 10 | 10, empty | 0 | 1 | 1 |
| Push 20 | 10, 20 | 0 | 0 | 2 |
| Pop | empty, 20 | 1 | 0 | 1 |
| Push 12 | 12, 20 | 1 | 1 | 2 |
| Close | 12, 20 | 1 | 1 | 2 |
| Pop | 12, empty | 0 | 1 | 1 |
| Pop | empty, empty | 1 | 1 | 0 |

The pop order is 10, 20, 12, even though the final physical slots held 12 before
20. Closing changes permission to submit, not the order of accepted jobs.

## Allocate before concurrent transfer begins

Create the start of `job_queue.cpp`:

```cpp title="exercises/chapter-18/job_queue.cpp"
#include "job_queue.hpp"

#include <chrono>
#include <stdexcept>
#include <utility>

namespace foundation18 {

Job::Job(std::uint64_t input, bool should_fail) noexcept
    : value(input), fail(should_fail) {
    live_.fetch_add(1, std::memory_order_relaxed);
}
Job::~Job() { live_.fetch_sub(1, std::memory_order_relaxed); }
std::size_t Job::live() noexcept {
    return live_.load(std::memory_order_relaxed);
}

std::size_t JobQueue::checked_capacity(std::size_t capacity) {
    if (capacity == 0) {
        throw std::invalid_argument("queue capacity must be positive");
    }
    return capacity;
}
JobQueue::JobQueue(std::size_t capacity)
    : capacity_(checked_capacity(capacity)), slots_(capacity_) {}

std::size_t JobQueue::advance(std::size_t index) const noexcept {
    return index + 1 == capacity_ ? 0 : index + 1;
}
```

The capacity check runs before constructing the vector of empty handles.
Allocation failure occurs before the queue can be shared. There is no vector
growth during transfer. `advance` increments only a valid index below capacity;
the conditional wraps the last index to zero without needing remainder or an
unbounded monotonically growing position.

`fetch_add(1, relaxed)` and `fetch_sub(1, relaxed)` are indivisible updates of the
counter; their returned previous values are unused. They do not authorize access
to a job or prove it is safe to destroy a queue. After joining all participants,
the fixture may read the final count; until then any reported count is only a
momentary observation. Atomicity of the counter is not synchronization of all
other memory.

## A notification is a reason to inspect state

A full producer needs the condition “there is space **or** the queue no longer
accepts work.” A consumer needs “there is an item **or** the state will not
produce another one.” Waiting solely for space or an item would strand threads
when shutdown changes only the queue state.

Append the producer wait helper:

```cpp title="exercises/chapter-18/job_queue.cpp"
void JobQueue::wait_for_space(std::unique_lock<std::mutex>& lock) {
    ++producers_;
    observed_.notify_all();
    try {
        space_.wait(lock, [this] {
            ++checks_;
            observed_.notify_all();
            return state_ != QueueState::open || count_ < capacity_;
        });
    } catch (...) {
        --producers_;
        observed_.notify_all();
        throw;
    }
    --producers_;
    observed_.notify_all();
}
```

The predicate form of `wait` repeats the condition check under the mutex. The
underlying wait releases the mutex and blocks as one operation, then reacquires
before continuing. This avoids a window in which a producer checks “full,” misses
a notification, and only afterward begins to sleep. The same mutex must protect
the condition's changes. A wake may be spurious, or another producer may take the
slot first; notification therefore cannot be treated as a permit. See the
[condition-variable contract](https://eel.is/c++draft/thread.condition.condvar).

Registration happens while holding `mutex_`. An observer acquiring this mutex
and seeing a producer registered knows the operation reached the predicate-wait
protocol. It does **not** know whether an operating-system thread is physically
asleep at that exact instant. That distinction is harmless: the queue cannot
complete this full-state push until space or state changes. The catch restores
diagnostic bookkeeping before propagating a catchable error; a failure to
reacquire the mutex is a library termination case, not a recovery promise.

Append the analogous consumer helper:

```cpp title="exercises/chapter-18/job_queue.cpp"
void JobQueue::wait_for_item(std::unique_lock<std::mutex>& lock) {
    ++consumers_;
    observed_.notify_all();
    try {
        item_.wait(lock, [this] {
            ++checks_;
            observed_.notify_all();
            return state_ != QueueState::open || count_ != 0;
        });
    } catch (...) {
        --consumers_;
        observed_.notify_all();
        throw;
    }
    --consumers_;
    observed_.notify_all();
}
```

Here the meaningful resource is a queued job, not a free slot. In both helpers
`observed_.notify_all()` lets the controller inspect already-protected counters.
Notifying while holding the mutex is allowed; an awakened observer acquires it
before reading. The normal queue notifications below occur after unlocking,
avoiding an immediate attempt to acquire a still-held lock. Neither placement
replaces the predicate or the state transition.

## Move only at the acceptance point

Append the complete push:

```cpp title="exercises/chapter-18/job_queue.cpp"
bool JobQueue::push(std::unique_ptr<Job>& job) {
    if (!job) {
        throw std::invalid_argument("cannot enqueue an empty job");
    }
    std::unique_lock lock(mutex_);
    if (state_ == QueueState::open && count_ == capacity_) {
        wait_for_space(lock);
    }
    if (state_ != QueueState::open) {
        return false;
    }
    slots_[write_] = std::move(job);
    write_ = advance(write_);
    ++count_;
    lock.unlock();
    item_.notify_one();
    return true;
}
```

The `if` outside the helper is a fast-path test. It is not an unsafe one-time
replacement for the wait loop: the helper itself performs a predicate wait.
After waiting, we examine state again while holding the same lock. If close won
the race, the function returns without touching `job`. If admission won, pointer
move, index update, and count update all occur before unlocking. No observer can
see the intermediate ring representation.

Consider close racing a push into a nonfull queue. Whichever takes the mutex and
performs its transition first determines the result. A push accepted just before
close remains accepted; close does not retrospectively reject it. The successful
ring update is the operation's *linearization point*: we can reason about it as
one event in an order consistent with these protected transitions.

Append the complete pop:

```cpp title="exercises/chapter-18/job_queue.cpp"
std::unique_ptr<Job> JobQueue::pop() {
    std::unique_lock lock(mutex_);
    if (state_ == QueueState::open && count_ == 0) {
        wait_for_item(lock);
    }
    if (state_ == QueueState::canceled || count_ == 0) {
        return {};
    }
    auto result = std::move(slots_[read_]);
    read_ = advance(read_);
    --count_;
    lock.unlock();
    space_.notify_one();
    return result;
}
```

An open queue with zero count has already waited. The return-empty branch is
therefore cancellation or a drained closed queue, not temporary lack of work.
The local `result` becomes the sole owner before the slot is reused. Destruction
of a consumer-owned job happens in the consumer, not inside this critical section.

The condition deliberately does **not** say “return empty whenever the queue is
not open.” Such code would protect memory correctly but discard the promised
drain semantics. Part 2 changes exactly that condition as a diagnostic mutation.
A race detector cannot determine that the wrong shutdown policy was implemented.

## Stop admission, then decide what remains owned

Append close:

```cpp title="exercises/chapter-18/job_queue.cpp"
void JobQueue::close() {
    {
        std::lock_guard lock(mutex_);
        if (state_ == QueueState::open) {
            state_ = QueueState::draining;
        }
    }
    space_.notify_all();
    item_.notify_all();
}
```

We wake both classes of waiter. A full producer must discover rejection; an
empty consumer must discover completion. Every waiting thread may need to exit,
so `notify_one` is not enough for shutdown. A worker still holding a job can
finish it and then observe the draining state on its next pop.

Append cancellation:

```cpp title="exercises/chapter-18/job_queue.cpp"
void JobQueue::cancel() {
    std::vector<std::unique_ptr<Job>> abandoned;
    {
        std::lock_guard lock(mutex_);
        state_ = QueueState::canceled;
        abandoned.swap(slots_);
        count_ = 0;
        read_ = 0;
        write_ = 0;
    }
    space_.notify_all();
    item_.notify_all();
    // abandoned and its jobs are destroyed here, outside the queue mutex.
}
```

Swapping with a local, initially empty vector transfers ownership of all queued
jobs without allocating new slots. The empty member vector is valid because
canceled operations never index it. After unlocking, we wake waiters and destroy
the abandoned jobs outside the mutex. A more complex job destructor could
release other resources; keeping it out of the lock avoids turning cleanup into
an unplanned nested critical section. This exercise's destructor only updates
an atomic counter.

If a worker popped job 10 immediately before cancellation, that job is no longer
in `slots_`. The abandoned vector cannot contain it. The worker may finish job 10
even after cancel returns. Cancellation is not rollback, forced termination, or
proof that all users have stopped. The eventual owner must join them before
destroying the queue. Concurrent cancellation calls are safe: after the first
swap, the member vector is empty and later swaps have no queued owners to take.

## Bound the backlog without pretending to bound a process

Let $C$ be queue capacity, $W$ the number of workers, and $P$ the number of
producers. Suppose each worker owns at most one removed job and each producer
allocates at most one pending job before blocking in `push`. During ordinary
processing, the participating jobs satisfy

$$
J \le C + W + P.
$$

The three terms are distinct owners: queued, running, and waiting to submit.
For the Part 2 stress fixture, $C=3$, $W=3$, and $P=3$, giving at most nine such
jobs despite 600 total transfers. A bound of three would forget both workers
and blocked producers. Cancellation temporarily moves queued ownership to the
canceling caller for destruction; it does not create another copy of a job.

This bound says nothing about jobs retained independently by callers, thread
stacks, allocator metadata, vector capacity overhead, exceptions, or process
resident memory. It is not Strata's charged query-memory admission policy.
Blocking admission can also deadlock when the producer holds a resource needed
by a consumer. The queue itself cannot diagnose that larger dependency cycle.
We keep its critical sections short and hold no application mutex across push.

## Observe waiting without using time as evidence

Complete `job_queue.cpp`:

```cpp title="exercises/chapter-18/job_queue.cpp"
QueueProbe JobQueue::probe() {
    std::lock_guard lock(mutex_);
    return {state_, count_, producers_, consumers_, checks_};
}

void JobQueue::await_waiters(std::size_t producers, std::size_t consumers,
                            std::uint64_t minimum_checks) {
    std::unique_lock lock(mutex_);
    const bool reached = observed_.wait_for(lock, std::chrono::seconds(5), [&] {
        return producers_ >= producers && consumers_ >= consumers &&
               checks_ >= minimum_checks;
    });
    if (!reached) {
        throw std::runtime_error("queue rendezvous watchdog expired");
    }
}

void JobQueue::notify_without_change() {
    space_.notify_all();
    item_.notify_all();
}

} // namespace foundation18
```

`await_waiters` uses a protected predicate to establish a rendezvous. Its
five-second timeout is only a watchdog reporting a broken test or an unusually
stalled environment; waiting five seconds is never evidence that the desired
state occurred. The predicate must actually be true. Part 2 also gives the whole
test process a CTest timeout so a missing shutdown wakeup does not hang forever.

`notify_without_change` sends an extra notification without changing the queue.
Tests wait for a later predicate evaluation and confirm that a full producer or
empty consumer remains registered. This models tolerating a wakeup without
permission. It does not claim to force a particular operating-system spurious
wakeup or prove which notification caused a check.

The observation counters belong to this exercise. They add a condition variable
and notifications that a production queue need not carry. They are used only at
the fixture's modest operation counts, not as an unbounded telemetry API. Sharing
the queue mutex prevents the tests from adding a race while observing one.

## What must survive a shutdown request

The queue now knows how to change state and wake callers. It still cannot own
their threads. Destroying it while a blocked push, pop, or notification uses its
mutex and condition variables would violate its lifetime contract. A destructor
that merely called close would not solve that problem: the waiters would be
awake, not necessarily finished.

[Part 2 — Join before destroying shared state](https://25d-228.github.io/cplusplus-learning/chapter-18-part-2.html)
provides the thread owner and verifies that references, exception slots, queue
storage, and mutexes survive through the final join.

### References for this part

The links above identify the mechanisms used here, not missing implementation.
The C++ working draft is a live document; this chapter uses only the C++20 subset
exercised by the code, not newer declarations visible on those pages. Relevant
sections are data races, thread construction and joining, mutex requirements,
condition-variable waiting, exception propagation, shared-pointer operations,
and atomic ordering/waiting. Technical pages were consulted on November 6, 2026.
