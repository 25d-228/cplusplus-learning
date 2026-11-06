# Chapter 18 — Synchronization and thread lifetime

## Part 2 — Join before destroying shared state

A stop flag, a wakeup, and a join answer different questions. The state decides
whether another operation is allowed. The wakeup gives a blocked thread a chance
to inspect that state. The join establishes that the thread has finished and
will no longer use its references. Omitting any one can leave a different bug.

This part completes the files begun in
[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-18.html).
It shares the chapter's **3.0-hour allocation**. Keep all engine code at
Chapter 17 unchanged. The only cumulative build change is an appended group of
standalone targets.

## A thread owner needs a policy for incomplete construction

`SumWorkers` owns its queue and every worker handle. It is deliberately not a
general callable executor: each job either contributes a checked integer value
or raises a deterministic error. That narrow operation keeps attention on
lifetime and failure rather than task type erasure.

Multiple callers may submit using independent job handles. `request_cancel`
may overlap submission and worker processing. The controlling thread alone
calls `finish` and destroys the owner; those lifecycle operations must not run
concurrently with each other, and destruction requires external submitters to
have stopped using the object. Workers never call `finish`. A live object is not
made safe to destroy by holding a pointer to it in another thread.

Create the complete `workers.hpp`:

```cpp title="exercises/chapter-18/workers.hpp"
#ifndef STRATA_EXERCISES_CHAPTER_18_WORKERS_HPP
#define STRATA_EXERCISES_CHAPTER_18_WORKERS_HPP

#include "job_queue.hpp"

#include <exception>
#include <limits>
#include <thread>

namespace foundation18 {

struct ThreadCounts {
    std::atomic<std::size_t> started{0};
    std::atomic<std::size_t> exited{0};
    std::atomic<std::size_t> joined{0};
};

struct Totals {
    std::uint64_t jobs = 0;
    std::uint64_t sum = 0;
};

class SumWorkers {
public:
    // counts, when supplied, must outlive this owner and its constructor.
    SumWorkers(std::size_t capacity, std::size_t worker_count,
               ThreadCounts* counts = nullptr,
               std::size_t fail_before = std::numeric_limits<std::size_t>::max());
    ~SumWorkers();
    SumWorkers(const SumWorkers&) = delete;
    SumWorkers& operator=(const SumWorkers&) = delete;
    SumWorkers(SumWorkers&&) = delete;
    SumWorkers& operator=(SumWorkers&&) = delete;

    bool submit(std::unique_ptr<Job>& job);
    void request_cancel();
    Totals finish();
    Totals totals();

private:
    void work() noexcept;
    void join_all() noexcept;

    JobQueue queue_;
    std::mutex result_mutex_;
    Totals totals_;
    std::exception_ptr error_;
    ThreadCounts* counts_;
    std::vector<std::thread> threads_;
};

} // namespace foundation18
#endif
```

The mutex protects the totals and first recorded worker exception together.
The thread vector is controlled only by the owner thread. `ThreadCounts` is
optional external diagnostic storage: if supplied, it must outlive construction,
all worker execution, and destruction. Tests declare it before the owner.
Atomic counters do not extend the lifetime of their containing object.

`fail_before` is a zero-based launch index. Passing 2 simulates failure just
before starting the third worker. The default maximum `size_t` disables the
injection. No attempt is made to exhaust real system thread resources.

The class is nonmovable because worker lambdas capture its `this` pointer. Moving
its fields would not retarget those pointers, and mutexes themselves are not
movable value objects. Shared ownership is not needed when one controlling
scope already owns the entire lifetime.

Start `workers.cpp` with construction and cleanup:

```cpp title="exercises/chapter-18/workers.cpp"
#include "workers.hpp"

#include <stdexcept>
#include <system_error>

namespace foundation18 {

SumWorkers::SumWorkers(std::size_t capacity, std::size_t worker_count,
                       ThreadCounts* counts, std::size_t fail_before)
    : queue_(capacity), counts_(counts) {
    if (worker_count == 0 || worker_count > 8) {
        throw std::invalid_argument("exercise worker count must be 1..8");
    }
    threads_.reserve(worker_count); // No vector growth after the first start.
    try {
        for (std::size_t i = 0; i < worker_count; ++i) {
            if (i == fail_before) {
                throw std::system_error(
                    std::make_error_code(std::errc::resource_unavailable_try_again),
                    "injected thread start failure");
            }
            threads_.emplace_back([this] { work(); });
        }
    } catch (...) {
        queue_.cancel();
        join_all();
        throw;
    }
}

SumWorkers::~SumWorkers() {
    queue_.cancel();
    join_all();
}
```

All members are initialized before the constructor body starts launching threads.
An early worker may run while later workers are being constructed, but uses only
the already-initialized queue, result state, and diagnostic pointer. It does not
access the thread vector. The vector reserves handle storage before any thread
starts, so later launch attempts do not trigger vector growth. `emplace_back`
constructs a `std::thread` directly in that reserved slot from the lambda.

If the third launch throws, the `SumWorkers` object was **not successfully
constructed**. Its destructor will not run. Relying on that destructor for cleanup
would leave two joinable member handles to be destroyed during unwinding. The
constructor's catch must cancel the queue, join the existing workers, and only
then rethrow the original start error. A real thread-constructor exception reaches
the same catch. The simulated `system_error` uses the standard resource-unavailable
error category rather than claiming an actual operating-system failure. See the
[thread creation failure contract](https://eel.is/c++draft/thread.thread.constr).

With failure before launch index 2, the sequence is:

| Controller | Existing workers |
|---|---|
| Reserve three handle slots | None yet |
| Start worker 0, then worker 1 | May reach empty-queue wait |
| Inject failure before worker 2 | Still own references to this object |
| Cancel queue and notify | Observe canceled state, return from pop |
| Join workers 0 and 1 | Complete; no later access to members |
| Rethrow start error | Member destruction is now safe |

Neither worker is guaranteed to have reached its wait before the failure. If
cancellation wins first, their first pop sees canceled and returns immediately.
Both schedules are correct. Tests inspect started/exited/joined counts **after**
constructor unwinding, without depending on sleep durations.

The normal destructor uses cancellation, not implicit draining. Call `finish`
explicitly for a completed sum and an observable worker error. During caller
exception unwinding, destructor cleanup discards queued jobs, lets already-owned
jobs finish, and joins. It cannot throw a worker error while another exception
is already unwinding.

## Never join while holding a mutex a worker needs

Append submission, observation, and joining:

```cpp title="exercises/chapter-18/workers.cpp"
bool SumWorkers::submit(std::unique_ptr<Job>& job) {
    return queue_.push(job);
}

void SumWorkers::request_cancel() { queue_.cancel(); }

Totals SumWorkers::totals() {
    std::lock_guard lock(result_mutex_);
    return totals_;
}

void SumWorkers::join_all() noexcept {
    for (auto& thread : threads_) {
        if (thread.joinable()) {
            thread.join();
            if (counts_) {
                counts_->joined.fetch_add(1, std::memory_order_relaxed);
            }
        }
    }
}
```

`submit` preserves the queue's ownership contract. `totals` returns a coherent
copy under the result mutex; callers do not borrow mutable internal fields.
`request_cancel` changes queue policy and wakes waiters, but does not join.
The controller must still finish or destroy the owner.

`join_all` holds neither queue nor result mutex. Joining while holding a mutex
needed by a worker's final accounting would produce a cycle: the controller
waits for the worker, while the worker waits for the controller's lock. The
controller never calls this routine from a worker, which would attempt to join
itself. A joined handle is skipped on subsequent calls.

`join_all` is `noexcept` for cleanup. Under the stated ownership contract every
join is on a valid, distinct, controller-owned joinable handle. Should a join
still throw a system error, this implementation fails fast through termination;
it does **not** detach and then destroy storage still in use. Failure of the
underlying synchronization primitives during destructor cleanup is likewise not
modeled as a recoverable job error. Normal job exceptions and injected launch
failures are handled; arbitrary synchronization-system failure is a different
boundary. See the possible errors in
[`thread::join`](https://eel.is/c++draft/thread.thread.member).

## Deliver an exception only after cleanup can no longer race

Append `finish`:

```cpp title="exercises/chapter-18/workers.cpp"
Totals SumWorkers::finish() {
    queue_.close();
    join_all();
    std::exception_ptr error;
    Totals result;
    {
        std::lock_guard lock(result_mutex_);
        error = error_;
        result = totals_;
    }
    if (error) {
        std::rethrow_exception(error);
    }
    return result;
}
```

Closing first stops admission while preserving accepted work. Joining then waits
for all workers. Only afterward do we copy the outcome and rethrow any captured
error. The rethrow happens outside the result mutex, so the caller's error handler
cannot run while that lock remains held. Repeated finish returns the same totals
or rethrows the same first recorded error; it does not start another execution.

The return value is the sum of **completed contributions**. After explicit
cancellation it may be partial. This exercise has no per-job completion records
or canceled-result discriminant, so a caller that requested cancellation must
not interpret the returned pair as proof that every accepted job ran. The next
runtime chapter supplies explicit task states; adding arbitrary callables here
would not supply those semantics by itself.

Complete the worker loop and implementation file:

```cpp title="exercises/chapter-18/workers.cpp"
void SumWorkers::work() noexcept {
    if (counts_) {
        counts_->started.fetch_add(1, std::memory_order_relaxed);
    }
    try {
        while (auto job = queue_.pop()) {
            if (job->fail) {
                throw std::runtime_error("injected job failure");
            }
            std::lock_guard lock(result_mutex_);
            const auto limit = std::numeric_limits<std::uint64_t>::max();
            if (totals_.jobs == limit || job->value > limit - totals_.sum) {
                throw std::overflow_error("worker totals overflow");
            }
            ++totals_.jobs;
            totals_.sum += job->value;
        }
    } catch (...) {
        {
            std::lock_guard lock(result_mutex_);
            if (!error_) {
                error_ = std::current_exception();
            }
        }
        queue_.cancel(); // Do not take queue_ while holding result_mutex_.
    }
    if (counts_) {
        counts_->exited.fetch_add(1, std::memory_order_relaxed);
    }
}

} // namespace foundation18
```

`while (auto job = queue_.pop())` declares an owning handle whose Boolean
conversion determines whether the body runs. Each successful pop gives this
worker exclusive job ownership. The body-local lock guard is destroyed before
the iteration's job owner, so releasing the job happens outside the result lock.
If processing throws, normal unwinding releases both before entering the catch.

Let $n$ be the existing count, $s$ the existing sum, $v$ the job value, and $M$ the
maximum `uint64_t`. An admissible contribution requires

$$
n < M,\qquad v \le M-s.
$$

Only after both checks do we publish the new pair $(n+1,s+v)$. The subtraction is
safe because $s$ is already representable. For $s=M$ and $v=1$, the check fails
before either field changes. This is Chapter 4's failure-safe state transition
inside one mutex, not a durable transaction or rollback of earlier jobs.

The catch stores only the first error to acquire `result_mutex_`. With several
workers, that is not necessarily the earliest submitted failing job. Recording
it precedes queue cancellation, and the result lock is released before acquiring
the queue lock. No path nests these two mutexes. Other workers may already own
jobs; they may finish their contributions before their next pop notices canceled
state. Partial totals are therefore not generally deterministic after concurrent
failure. The single-worker failure fixture below fixes the relevant ordering so
that its expected partial sum is exactly 10.

A bare `std::thread` destructor invokes termination if its handle is still
joinable, even when the underlying function has already returned. Joinable is a
handle state, not a synonym for “currently executing.” Explicit joining is not
optional housekeeping. See
[thread destruction](https://eel.is/c++draft/thread.thread.destr).
C++20 `std::jthread` supplies stop-request-and-join behavior during destruction,
but a stop request does not itself change this queue's predicate or notify its
ordinary condition variables. Replacing the handle type alone would not fix a
join-before-wakeup deadlock. Its behavior is specified under
[jthread construction and destruction](https://eel.is/c++draft/thread.jthread.cons).

## Make test failures clean up threads too

Tests need the same lifetime discipline as application code. A `require` that
throws while an unguarded test thread waits in pop could terminate the program
or hang cleanup, obscuring the original assertion. The small harness below
cancels its associated queue and joins during destruction.

Begin `main.cpp`:

```cpp title="exercises/chapter-18/main.cpp"
#include "workers.hpp"

#include <array>
#include <iostream>
#include <stdexcept>
#include <system_error>
#include <utility>

using namespace foundation18;

namespace {
void require(bool condition, const char* message) {
    if (!condition) { throw std::runtime_error(message); }
}

template<class Error, class Function>
void expect_error(Function function, const char* message) {
    bool caught = false;
    try { function(); }
    catch (const Error&) { caught = true; }
    require(caught, message);
}
```

`expect_error<Error>` checks one specified exception family; an unrelated error
continues outward instead of counting as success. It takes its callable by value,
a pattern from Chapter 5. `require` remains active with `NDEBUG` defined.

Append the test-only thread guard:

```cpp title="exercises/chapter-18/main.cpp"
// Test harness: all referenced locals must be declared BEFORE this guard.
class QueueThreads {
public:
    explicit QueueThreads(JobQueue& queue) : queue_(queue) {}
    ~QueueThreads() {
        queue_.cancel();
        join_all();
    }
    QueueThreads(const QueueThreads&) = delete;
    QueueThreads& operator=(const QueueThreads&) = delete;

    template<class Function>
    std::size_t start(Function function) {
        if (used_ == threads_.size()) {
            throw std::length_error("test thread capacity exceeded");
        }
        const auto index = used_;
        threads_[index] = std::thread(
            [this, index, task = std::move(function)]() mutable {
                try { task(); }
                catch (...) {
                    errors_[index] = std::current_exception();
                    queue_.cancel();
                }
            });
        ++used_;
        return index;
    }

    void join_one(std::size_t index) {
        if (index >= used_) { throw std::out_of_range("test thread index"); }
        if (threads_[index].joinable()) { threads_[index].join(); }
    }

    void finish() {
        join_all();
        for (std::size_t i = 0; i < used_; ++i) {
            if (errors_[i]) { std::rethrow_exception(errors_[i]); }
        }
    }
private:
    void join_all() noexcept {
        for (auto& thread : threads_) {
            if (thread.joinable()) { thread.join(); }
        }
    }
    JobQueue& queue_;
    std::array<std::exception_ptr, 6> errors_{};
    std::array<std::thread, 6> threads_{};
    std::size_t used_ = 0;
};
```

The fixed six-slot arrays are enough for three producers and three consumers.
Default-constructed thread handles are not joinable. Each worker writes only its
own `errors_[index]`; the controller inspects those slots after joining. Distinct
ordinary array elements are distinct storage, unlike packed `vector<bool>` proxy
bits. Only the controller changes `used_`.

The lambda's `task = std::move(function)` is an **initialization capture**: the
closure owns the passed callable. `mutable` permits invoking a callable with a
non-const call operator. Its references, if any, are still borrowed. Capturing a
closure by value does not turn references inside that closure into owning values.

Declare every object borrowed by these closures **before** `QueueThreads`.
Reverse destruction order then runs the guard, wakes the threads, and joins before
those objects disappear. If starting a later harness thread fails, the already
constructed guard supplies the same cleanup. The worker catch records its own
error and cancels the queue so other threads do not wait forever for work that
will never arrive.

`finish` intentionally does not close the queue: tests must first arrange that
their operations can finish. A producer-only test frees a slot; a consumer-only
test supplies an item or closes; the mixed fixture joins its producers and then
closes. The destructor is the fallback on assertion failure, not a substitute
for testing each normal protocol.

## Distinguish correct synchronization from correct queue semantics

Append the single-thread ring and shutdown check:

```cpp title="exercises/chapter-18/main.cpp"
void check_fifo_and_contracts() {
    expect_error<std::invalid_argument>([] { JobQueue invalid(0); },
                                       "zero capacity accepted");
    JobQueue queue(2);
    std::unique_ptr<Job> empty;
    expect_error<std::invalid_argument>([&] { queue.push(empty); },
                                       "empty job accepted");
    for (std::uint64_t base = 0; base < 6; base += 2) {
        auto first = std::make_unique<Job>(base + 1);
        auto second = std::make_unique<Job>(base + 2);
        require(queue.push(first) && queue.push(second), "open queue rejected");
        require(!first && !second, "accepted job stayed in caller");
        auto a = queue.pop();
        auto b = queue.pop();
        require(a && b && a->value == base + 1 && b->value == base + 2,
                "ring order or wrap differs");
    }
    auto last = std::make_unique<Job>(42);
    require(queue.push(last), "last open push rejected");
    queue.close();
    queue.close();
    auto late = std::make_unique<Job>(99);
    auto* identity = late.get();
    require(!queue.push(late) && late.get() == identity, "late job consumed");
    auto drained = queue.pop();
    require(drained && drained->value == 42, "close discarded accepted job");
    require(!queue.pop(), "drained queue returned an item");
}
```

This checks exact FIFO order through repeated wraparound, not just a sum.
`identity` records the address owned before a rejected submission; equality
proves that rejection did not consume or replace that job. Calling close twice
must not discard the already-accepted value 42.

For a logical mutation, change pop's return-empty condition from
`state_ == QueueState::canceled || count_ == 0` to
`state_ != QueueState::open || count_ == 0`. All accesses remain protected,
but the test must report `close discarded accepted job`. This is a protocol error,
not a data race. Restore the original line before continuing.

## Establish blocked waiters, then change state

Append the empty-consumer scenario:

```cpp title="exercises/chapter-18/main.cpp"
void check_waiting_consumers(bool cancel) {
    JobQueue queue(1);
    std::array<std::unique_ptr<Job>, 3> results;
    QueueThreads threads(queue);
    for (std::size_t i = 0; i < results.size(); ++i) {
        threads.start([&, i] { results[i] = queue.pop(); });
    }
    queue.await_waiters(0, 3);
    const auto before = queue.probe();
    queue.notify_without_change();
    queue.await_waiters(0, 3, before.predicate_checks + 1);
    require(queue.probe().queued == 0, "notification invented an item");
    if (cancel) { queue.cancel(); } else { queue.close(); }
    threads.finish();
    for (const auto& result : results) {
        require(!result, "closed empty queue produced a job");
    }
}
```

The controller waits until all three pop operations are registered on an empty
queue. It sends an extra notification and waits for another predicate check
without adding work. Output handles are read only after joining. Close and
cancel are both tested; each must wake all three consumers to return empty.
A five-second sleep followed by inspecting `results` would be both an unreliable
scheduling assumption and an unsynchronized access.

Append the full-producer scenario:

```cpp title="exercises/chapter-18/main.cpp"
void check_waiting_producers(bool cancel) {
    JobQueue queue(1);
    auto queued = std::make_unique<Job>(10);
    require(queue.push(queued), "initial push rejected");
    std::array<std::unique_ptr<Job>, 2> pending;
    pending[0] = std::make_unique<Job>(20);
    pending[1] = std::make_unique<Job>(12);
    const std::array<Job*, 2> identities{pending[0].get(), pending[1].get()};
    std::array<bool, 2> accepted{true, true}; // Not vector<bool>.
    QueueThreads threads(queue);
    for (std::size_t i = 0; i < pending.size(); ++i) {
        threads.start([&, i] { accepted[i] = queue.push(pending[i]); });
    }
    queue.await_waiters(2, 0);
    const auto before = queue.probe();
    queue.notify_without_change();
    queue.await_waiters(2, 0, before.predicate_checks + 1);
    require(queue.probe().queued == 1, "notification made room");
    if (cancel) { queue.cancel(); } else { queue.close(); }
    threads.finish();
    for (std::size_t i = 0; i < pending.size(); ++i) {
        require(!accepted[i] && pending[i].get() == identities[i],
                "rejected producer lost ownership");
    }
    if (cancel) {
        queue.close(); // Closing cannot reverse cancellation.
        queue.cancel();
        require(!queue.pop(), "cancel retained queued work");
        require(queue.probe().state == QueueState::canceled, "cancel reversed");
    } else {
        auto drained = queue.pop();
        require(drained && drained->value == 10, "close did not drain");
        require(!queue.pop(), "drain did not finish");
    }
}
```

Here one queued job fills the single slot, while two producers each own a pending
job. The controller never reads or resets those handles until the producers are
joined. It can inspect the ring through the locked `probe` meanwhile. Both close
and cancel reject waiters without moving their handles. Close still delivers
the initial 10; cancel does not. Close after cancel must not reverse the state.

The pair of tests distinguishes waking “at least one” from waking **every**
shutdown waiter. They also exercise nonproductive notifications. A passing run
establishes these controlled executions, not fairness for every scheduler.

Append the ordinary resource-change wakeup checks:

```cpp title="exercises/chapter-18/main.cpp"
void check_transfer_wakeups() {
    {
        JobQueue queue(1);
        auto first = std::make_unique<Job>(10);
        auto second = std::make_unique<Job>(20);
        bool accepted = false;
        require(queue.push(first), "initial push rejected");
        QueueThreads threads(queue);
        threads.start([&] { accepted = queue.push(second); });
        queue.await_waiters(1, 0);
        auto result = queue.pop();
        threads.finish(); // Removing the first job must wake this producer.
        require(result && result->value == 10 && accepted && !second,
                "space wakeup did not transfer ownership");
        queue.close();
        auto next = queue.pop();
        require(next && next->value == 20 && !queue.pop(), "second job missing");
    }
    {
        JobQueue queue(1);
        std::unique_ptr<Job> result;
        auto job = std::make_unique<Job>(12);
        QueueThreads threads(queue);
        threads.start([&] { result = queue.pop(); });
        queue.await_waiters(0, 1);
        require(queue.push(job), "waiting consumer push rejected");
        threads.finish(); // Inserting the job must wake this consumer.
        queue.close();
        require(result && result->value == 12 && !job, "item wakeup lost job");
    }
}
```

The first removes 10 from a full queue and waits for the blocked producer to
transfer 20. The second supplies 12 to an already-waiting consumer. Joining in
these two cases is legitimate because the controller has already performed the
resource change allowing the worker to finish. Shutdown alone is not the only
path that must notify correctly.

## Check each job, not just the aggregate total

Append the multiple-producer/multiple-consumer fixture:

```cpp title="exercises/chapter-18/main.cpp"
void check_multiple_producers_and_consumers() {
    constexpr std::size_t per_producer = 200;
    constexpr std::size_t producer_count = 3;
    constexpr std::size_t consumer_count = 3;
    constexpr std::size_t total = producer_count * per_producer;
    JobQueue queue(3);
    std::array<std::array<unsigned, total + 1>, consumer_count> seen{};
    std::array<std::size_t, producer_count> producer_threads{};
    QueueThreads threads(queue);
    for (std::size_t c = 0; c < consumer_count; ++c) {
        threads.start([&, c] {
            while (auto job = queue.pop()) {
                require(job->value >= 1 && job->value <= total, "invalid job ID");
                ++seen[c][static_cast<std::size_t>(job->value)];
            }
        });
    }
    for (std::size_t p = 0; p < producer_count; ++p) {
        producer_threads[p] = threads.start([&, p] {
            for (std::size_t i = 1; i <= per_producer; ++i) {
                auto job = std::make_unique<Job>(p * per_producer + i);
                require(queue.push(job) && !job, "producer rejected unexpectedly");
            }
        });
    }
    for (auto index : producer_threads) { threads.join_one(index); }
    queue.close();
    threads.finish();
    std::uint64_t sum = 0;
    for (std::size_t id = 1; id <= total; ++id) {
        unsigned occurrences = 0;
        for (const auto& consumer : seen) { occurrences += consumer[id]; }
        require(occurrences == 1, "job was lost or processed more than once");
        sum += id;
    }
    require(sum == 180300, "600-job fixture sum differs");
}
```

Three producers generate disjoint ID intervals `1..200`, `201..400`, and
`401..600`. Each of three consumers owns its own histogram row. No two consumers
write the same row, and the controller reads all rows only after joining. The
arrays exist before any thread starts; they are test-oracle storage, not a
production result container.

For each ID $i$, let $h_{c,i}$ be the count recorded by consumer $c$. The actual
coverage check is

$$
\sum_{c=0}^{2} h_{c,i}=1\quad\text{for every }i\in\{1,\ldots,600\}.
$$

Only after checking every ID do we compute the fixture sum

$$
\sum_{i=1}^{600}i=\frac{600\cdot601}{2}=180{,}300.
$$

A sum alone could conceal one missing ID and a compensating duplicate. We do not
expect a particular consumer to receive a particular ID, nor a deterministic
interleaving between producers. FIFO orders accepted ring transitions, not the
wall-clock time when independent producers began constructing their jobs.

The controller first joins producers, which can finish because consumers are
already running. It then closes the queue and joins consumers. Joining consumers
first on an open, empty queue would never finish: no operation would tell them
that production had ended.

## Run the promised result and retain the rejected owner

Append the sum fixture:

```cpp title="exercises/chapter-18/main.cpp"
void check_sum_and_late_job() {
    ThreadCounts counts;
    {
        SumWorkers workers(2, 2, &counts);
        auto a = std::make_unique<Job>(10);
        auto b = std::make_unique<Job>(20);
        auto c = std::make_unique<Job>(12);
        require(workers.submit(a) && workers.submit(b) && workers.submit(c),
                "normal submission rejected");
        require(!a && !b && !c, "accepted ownership not transferred");
        const auto totals = workers.finish();
        require(totals.jobs == 3 && totals.sum == 42, "drained totals differ");
        require(workers.finish().sum == 42, "repeat finish differs");
        auto late = std::make_unique<Job>(99);
        require(!workers.submit(late) && late && late->value == 99,
                "late submission consumed its job");
        std::cout << "sum=42, late job retained=99\n";
    }
    require(counts.started == 2 && counts.exited == 2 && counts.joined == 2,
            "normal owner did not join exactly twice");
}
```

The three successful submissions empty the caller's handles. Closing and joining
through `finish` yields three contributions totaling 42; the exact split across
the two workers is irrelevant. The later 99 remains owned by the caller. Reading
an atomic counter in comparisons such as `counts.started == 2` invokes its atomic
value conversion; these checks happen after joins, not as a polling protocol.
The counter is diagnostic, not the reason the worker lifetimes are safe.

Append processing failure, overflow, and explicit cancellation checks:

```cpp title="exercises/chapter-18/main.cpp"
void check_errors_and_cancel() {
    {
        SumWorkers workers(3, 1);
        auto a = std::make_unique<Job>(10);
        auto failure = std::make_unique<Job>(20, true);
        auto trailing = std::make_unique<Job>(99);
        require(workers.submit(a) && workers.submit(failure), "failure setup");
        const bool accepted = workers.submit(trailing);
        require(accepted == !trailing, "trailing ownership disagrees");
        expect_error<std::runtime_error>([&] { workers.finish(); },
                                         "worker exception disappeared");
        const auto partial = workers.totals();
        require(partial.jobs == 1 && partial.sum == 10, "error rolled back or ran tail");
        expect_error<std::runtime_error>([&] { workers.finish(); },
                                         "repeat finish lost the error");
    }
    {
        SumWorkers workers(2, 1);
        auto maximum = std::make_unique<Job>(std::numeric_limits<std::uint64_t>::max());
        auto extra = std::make_unique<Job>(1);
        require(workers.submit(maximum) && workers.submit(extra), "overflow setup");
        expect_error<std::overflow_error>([&] { workers.finish(); },
                                          "worker sum wrapped");
        const auto partial = workers.totals();
        require(partial.jobs == 1 &&
                partial.sum == std::numeric_limits<std::uint64_t>::max(),
                "failed addition changed totals");
    }
    {
        ThreadCounts counts;
        SumWorkers workers(1, 3, &counts);
        workers.request_cancel();
        workers.request_cancel();
        require(workers.finish().jobs == 0, "empty cancel made work");
        auto rejected = std::make_unique<Job>(7);
        require(!workers.submit(rejected) && rejected, "cancel consumed late job");
        require(counts.exited == 3 && counts.joined == 3, "cancel did not join");
    }
}
```

With one worker, FIFO fixes the order: 10 contributes first, the failing 20 raises
an exception, and 99 cannot contribute. The 99 may have been admitted before
cancellation or may still belong to the caller; both schedules are allowed.
`accepted == !trailing` checks the appropriate ownership in either case. The
partial total remains `(1,10)`, and `finish` rethrows the recorded failure even
on a second call.

The overflow fixture completes the maximum representable value, then rejects
adding 1 before changing either total. The cancellation fixture has no jobs, so
it makes no timing assumption about whether a worker was already waiting when
cancellation occurred. Waiting-state shutdown itself was covered separately with
explicit rendezvous.

Append construction and unwinding checks:

```cpp title="exercises/chapter-18/main.cpp"
void check_lifetime_and_startup() {
    for (std::size_t fail_before = 0; fail_before < 3; ++fail_before) {
        ThreadCounts counts;
        expect_error<std::system_error>([&] {
            SumWorkers failed(1, 3, &counts, fail_before);
        }, "startup injection did not fail");
        require(counts.started == fail_before && counts.exited == fail_before &&
                counts.joined == fail_before, "partial startup leaked a thread");
    }
    ThreadCounts counts;
    struct CallerFailure {};
    expect_error<CallerFailure>([&] {
        SumWorkers workers(1, 3, &counts);
        throw CallerFailure{};
    }, "caller failure missing");
    require(counts.started == 3 && counts.exited == 3 && counts.joined == 3,
            "unwinding did not join workers");
    expect_error<std::invalid_argument>([] { SumWorkers invalid(1, 0); },
                                       "zero workers accepted");
}
} // namespace

int main() {
    try {
        check_fifo_and_contracts();
        check_waiting_consumers(false);
        check_waiting_consumers(true);
        check_waiting_producers(false);
        check_waiting_producers(true);
        check_transfer_wakeups();
        check_multiple_producers_and_consumers();
        check_sum_and_late_job();
        check_errors_and_cancel();
        check_lifetime_and_startup();
        require(Job::live() == 0, "jobs survived joined cleanup");
        std::cout << "600 jobs exactly once; waiters, errors, startup, cleanup passed\n"
                  << "live jobs=0\n";
        return 0;
    } catch (const std::exception& error) {
        std::cerr << "foundation-18: " << error.what() << '\n';
        return 1;
    }
}
```

Failure before launch indices 0, 1, and 2 verifies cleanup of zero, one, and two
started workers. The observer outlives the failed construction. Throwing a
`CallerFailure` after starting three workers verifies destructor cleanup during
ordinary stack unwinding. At the end, every owner scope has ended and every
started thread has been joined, so the final live-job count must be zero.

The expected output of `foundation-18` is:

```text
sum=42, late job retained=99
600 jobs exactly once; waiters, errors, startup, cleanup passed
live jobs=0
```

No worker prints concurrently. The controlling thread reports only after the
relevant checks, avoiding nondeterministic interleaved logging as a supposed
proof of correctness.

## Append native build targets, not a new engine configuration

Append the following block to the existing root `CMakeLists.txt`. Do not replace
the earlier targets. `Threads::Threads` carries the platform's required compile
and link settings; its flags may be empty where support is already provided by
the system libraries. `THREADS_PREFER_PTHREAD_FLAG` requests the compiler option
when appropriate rather than hard-coding a particular thread library. See
[CMake's FindThreads module](https://cmake.org/cmake/help/latest/module/FindThreads.html).

```cmake title="CMakeLists.txt"
# Chapter 18: standalone synchronization exercises, not a threaded engine.
set(THREADS_PREFER_PTHREAD_FLAG TRUE)
find_package(Threads REQUIRED)

add_executable(foundation-18-visibility
    exercises/chapter-18/visibility.cpp)
target_compile_features(foundation-18-visibility PRIVATE cxx_std_20)
target_compile_options(foundation-18-visibility PRIVATE -Wall -Wextra -Wpedantic)
target_link_libraries(foundation-18-visibility PRIVATE Threads::Threads)
add_test(NAME foundation-18-visibility COMMAND foundation-18-visibility)

add_executable(foundation-18
    exercises/chapter-18/job_queue.cpp
    exercises/chapter-18/workers.cpp
    exercises/chapter-18/main.cpp)
target_compile_features(foundation-18 PRIVATE cxx_std_20)
target_compile_options(foundation-18 PRIVATE -Wall -Wextra -Wpedantic)
target_link_libraries(foundation-18 PRIVATE Threads::Threads)
add_test(NAME foundation-18 COMMAND foundation-18)
set_tests_properties(foundation-18 foundation-18-visibility PROPERTIES TIMEOUT 30)

# Intentionally wrong: never part of the default build or passing CTest suite.
add_executable(foundation-18-racy EXCLUDE_FROM_ALL
    exercises/chapter-18/racy_counter.cpp)
target_compile_features(foundation-18-racy PRIVATE cxx_std_20)
target_compile_options(foundation-18-racy PRIVATE -Wall -Wextra -Wpedantic)
target_link_libraries(foundation-18-racy PRIVATE Threads::Threads)
```

`EXCLUDE_FROM_ALL` keeps the intentionally wrong executable out of the default
build. There is deliberately no `add_test` for it. A correct normal CTest run
must never depend on executing undefined behavior. The 30-second timeouts are
watchdogs, not synchronization; a stuck join is a failed test, not a reason to
increase an arbitrary sleep.

From the learner workspace, configure and run the two normal targets:

```sh
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_COMPILER="$(xcrun --find clang++)"
cmake --build build/debug --target foundation-18-visibility foundation-18 -j 4
ctest --test-dir build/debug --output-on-failure \
  -R '^foundation-18(-visibility)?$'

cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_COMPILER="$(xcrun --find clang++)"
cmake --build build/release --target foundation-18-visibility foundation-18 -j 4
ctest --test-dir build/release --output-on-failure \
  -R '^foundation-18(-visibility)?$'
```

Use the configured compiler path for an equivalent native LLVM installation.
When reusing a build directory, keep its compiler selection consistent; choose a
new directory when changing toolchains. The regular expressions select exactly
the two new passing tests. The earlier foundation and engine tests remain
available through the unchanged cumulative configuration.

A memory/undefined-behavior diagnostic build is separate:

```sh
cmake -S . -B build/asan-18 -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_COMPILER="$(xcrun --find clang++)" \
  -DCMAKE_CXX_FLAGS='-fsanitize=address,undefined -fno-omit-frame-pointer' \
  -DCMAKE_EXE_LINKER_FLAGS='-fsanitize=address,undefined'
cmake --build build/asan-18 --target foundation-18-visibility foundation-18 -j 4
ctest --test-dir build/asan-18 --output-on-failure \
  -R '^foundation-18(-visibility)?$'
```

These instruments can catch memory misuse and selected undefined operations.
They are not a data-race checker and do not establish that a queue drains the
right jobs. Keep logical assertions in every configuration.

## Isolate the data race instead of normalizing it

Create `racy_counter.cpp` exactly as this separate diagnostic:

```cpp title="exercises/chapter-18/racy_counter.cpp"
// Intentionally undefined behavior. Build/run only as a race-detector exercise.
#include <array>
#include <atomic>
#include <iostream>
#include <memory>
#include <thread>

struct Counter { unsigned value = 0; };

int main() {
    auto shared = std::make_shared<Counter>();
    std::atomic<unsigned> arrived{0};
    std::atomic<bool> go{false};
    std::array<std::thread, 2> threads;
    try {
        for (auto& thread : threads) {
            thread = std::thread([shared, &arrived, &go] {
                arrived.fetch_add(1, std::memory_order_release);
                arrived.notify_one();
                go.wait(false, std::memory_order_acquire);
                for (unsigned i = 0; i < 100000; ++i) {
                    ++shared->value; // Deliberate unsynchronized conflicting writes.
                }
            });
        }
    } catch (...) {
        go.store(true, std::memory_order_release);
        go.notify_all();
        for (auto& thread : threads) {
            if (thread.joinable()) { thread.join(); }
        }
        throw;
    }
    auto count = arrived.load(std::memory_order_acquire);
    while (count != 2) {
        arrived.wait(count, std::memory_order_acquire);
        count = arrived.load(std::memory_order_acquire);
    }
    go.store(true, std::memory_order_release);
    go.notify_all();
    for (auto& thread : threads) { thread.join(); }
    std::cout << "diagnostic only; no valid expected count: " << shared->value << '\n';
}
```

Both closures own a copy of the `shared_ptr`, so the counter's lifetime is not
the bug. The arrival counter and start gate are atomic; they arrange for both
workers to reach a controlled starting phase. The accesses to `Counter::value`
after that gate are plain, conflicting increments with no ordering between
workers. Joining both afterward does not repair those earlier accesses.

The atomic `arrived.wait(count, acquire)` waits for a change from its last
observed value; the loop then loads the current count again. This prevents
notification counting from becoming a second bug in the diagnostic. A partial
thread-start failure releases the start gate and joins any existing worker
before rethrowing. That cleanup does not make the normal two-worker increment
legal: the program intentionally has undefined behavior and belongs nowhere in
the passing suite.

Build a **separate** race-diagnostic configuration:

```sh
cmake -S . -B build/tsan-18 -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_COMPILER="$(xcrun --find clang++)" \
  -DCMAKE_CXX_FLAGS='-O1 -g -fsanitize=thread' \
  -DCMAKE_EXE_LINKER_FLAGS='-fsanitize=thread'
cmake --build build/tsan-18 \
  --target foundation-18-visibility foundation-18 foundation-18-racy -j 4
ctest --test-dir build/tsan-18 --output-on-failure \
  -R '^foundation-18(-visibility)?$'

status=0
TSAN_OPTIONS=halt_on_error=1 build/tsan-18/foundation-18-racy \
  > build/tsan-18/racy.log 2>&1 || status=$?
cat build/tsan-18/racy.log
printf 'diagnostic exit=%s\n' "$status"
```

The normal tests should pass without a race report. The deliberately wrong
program should report conflicting accesses at `++shared->value`, with worker
creation stacks. A nonzero exit by itself is not enough: a runtime load error or
unsupported environment is not a detected race. Read the diagnostic. Its address,
thread identifiers, stack formatting, and exit code are not portable expectations.
Even printing 200,000 would not validate this program; undefined behavior has no
required numerical result.

Clang documents ThreadSanitizer support for Darwin ARM64 and compilation/linking
with `-fsanitize=thread`; `-g` retains source locations, and modest optimization
is appropriate for its diagnostic use. Consult the
[ThreadSanitizer documentation](https://clang.llvm.org/docs/ThreadSanitizer.html)
for the installed runtime's limitations. Use a working compatible native toolchain,
not an invented required version. Do not combine ThreadSanitizer with
AddressSanitizer in the same executable. GCC's
[instrumentation options](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html)
also document that incompatibility; GCC is an additional testing option where
available, not a macOS installation requirement.

A sanitizer substantially changes execution and resource behavior. Its passing
run is evidence about exercised accesses, not a proof of race freedom, scheduling
fairness, absence of deadlock, or production throughput. The controlled
rendezvous tests and explicit state reasoning remain necessary.

## Repair the scalar without pretending to repair a multi-field protocol

As a small variation, copy `racy_counter.cpp` to a temporary file outside the
normal target. Replace the declaration and increment with:

```cpp
struct Counter { std::atomic<unsigned> value{0}; };
```

```cpp
shared->value.fetch_add(1, std::memory_order_relaxed);
```

Replace the final output statement with the following statements:

```cpp
const auto result = shared->value.load(std::memory_order_relaxed);
std::cout << "atomic count=" << result << '\n';
return result == 200000 ? 0 : 1;
```

Now the shared operation is one scalar read-modify-write. No other payload is
published through the counter, so relaxed ordering is sufficient for this
purpose. The joins precede the final read. Rebuild the temporary version with
ThreadSanitizer: the expected result is `atomic count=200000` without the original
race report. Do not apply this edit to the queue's count and assume the slots,
indices, and state now form an atomic transition; they still need their common
mutex. Preserve the intentionally wrong canonical diagnostic for later comparison.

## Make a race-free protocol mistake on purpose

In a saved working copy of `job_queue.cpp`, make the one-line pop change described
earlier: return empty for **any** non-open state. Rebuild `foundation-18` in Debug,
Release, and the supported sanitizer configurations, then run it directly. The
expected logical failure is:

```text
foundation-18: close discarded accepted job
```

Its exit is nonzero even in Release. A clean sanitizer result would not contradict
this assertion: the program discarded accepted work under the mutex. Restore the
original canceled-or-empty condition, rebuild, and require both normal tests to
pass again. Do not leave the mutation in the learner workspace for the next chapter.

## Diagnose the missing relationship

| Symptom | Inspect first | Concrete correction |
|---|---|---|
| Shutdown hangs with several consumers | Close/cancel state and both waiter classes | Set state under the mutex, notify all, then join |
| Close loses accepted jobs | Pop's return-empty branch | Drain until count is zero unless canceled |
| A rejected push empties the caller | Location of the pointer move | Move only after deciding to accept under the mutex |
| Exception during startup terminates | Constructor catch and joinable members | Cancel and join existing workers before rethrowing |
| A job exception terminates immediately | The thread callable boundary | Capture the exception, clean up, rethrow in the controller |
| Pair totals sometimes disagree | Whether all fields use the same lock | Protect and inspect the whole invariant together |
| Race despite shared ownership | Accesses to the pointed-to payload | Add appropriate synchronization; reference counting is not that lock |
| Rendezvous times out | Predicate counters and actual thread startup | Diagnose the blocked operation; do not treat a longer sleep as a fix |

A missing notification may leave the controller blocked in `join`, while a worker
waits in pop or push. A debugger's all-thread backtraces can reveal that wait
relationship; the waiting stack alone is not evidence that the mutex was forgotten.
The relevant question is which operation can make the predicate true, and whether
that operation can still run. Assertions made before releasing a required test
resource can themselves create a hang, which is why test cleanup is also owned.

The five-second rendezvous watchdog can expire while a debugger has paused
threads. That is a test-environment interruption, not a successful synchronization
result. For interactive stepping, adjust only the diagnostic watchdog in a scratch
copy; never remove the production predicate or rely on scheduling delays.

## What has and has not been established

The new normal programs were executed with Clang 17 on Linux in Debug, Release,
and AddressSanitizer/UndefinedBehaviorSanitizer configurations. A separate GCC
14.2 ThreadSanitizer build ran the normal programs without a race report and
reported the deliberately shared-counter race. These are execution observations,
not required compiler versions. Native M4/macOS execution remains unverified.

The queue fixture transfers 600 distinct jobs exactly once, deliberately wakes
waiters without supplying permission, closes and cancels with producers and
consumers registered, retains rejected ownership, propagates job failures,
preserves totals on checked overflow, joins partially started workers, and reaches
zero live jobs. None of those observations proves arbitrary scheduler fairness.
The exercise also does not promise prompt interruption of a blocking filesystem
operation or a bound on whole-process memory.

Most importantly, the engine is still synchronous. An immutable database fixture
can still use a mutable cache, accounting ledger, retirement registry, or counters
on its read path. Before multiple query workers share those objects, each mutable
access needs synchronization or genuine isolation. A `shared_ptr<ReadView>` alone
would not supply it. The next chapter adds bounded execution and cooperative
cancellation to complete queries; it must preserve these lifetime rules while
introducing explicit task completion and admission semantics.

### References for this part

In addition to Part 1's C++ references, the implementations above rely on the
specified thread creation/join/destruction behavior and exception transport.
CMake's `FindThreads` documentation explains platform-specific build support;
Clang's ThreadSanitizer documentation and GCC's instrumentation documentation
explain the diagnostic configurations. The linked pages were consulted on
November 6, 2026. Their current tool-version labels are not course requirements.
