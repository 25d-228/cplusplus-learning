# Chapter 19 — Bounded execution and cooperative cancellation

## Part 2 — A terminal result owns no running query

[Part 1](https://25d-228.github.io/cplusplus-learning/chapter-19.html) made a query
safe to run on one worker: its read-side state is private, its allocation cap is
real, and cancellation unwinds its existing owners. Now we decide which worker
may start it, how the caller observes completion, and how shutdown resolves every
accepted request.

This is the second part of the same **4.0-hour chapter**, not another allocation.
The code operates against `FrozenInputs`; it must not capture a mutable database
inside an arbitrary callable. No job may submit a blocking descendant to this
same pool. A pool with all workers waiting for their own queued children would
be deadlocked even if every mutex were correctly used.

## A task handle is an owner of the answer, not the work

Use these state transitions:

```text
created --accepted--> queued --allowances reserved--> running
                         |                              |
                         | canceled before admission    +--> completed
                         +------------------------------+--> canceled
                                                        +--> failed
```

A rejected submission stays `created` and remains with its caller. An accepted
request is uniquely owned by the queue or by one worker. A separate shared
control object survives through any copied handles. That control object contains
the stop flag, state, completion statistics, and an exception reference. It does
not contain a read view, output batch, or private query budget.

`std::exception_ptr` retains the caught exception beyond the catch block.
`std::current_exception()` captures it while handling the error; the caller can
later use `std::rethrow_exception` after checking the pointer is non-null. An
exception is not magically thrown on the caller's thread. The ownership and
capture behavior is specified in
[exception propagation](https://eel.is/c++draft/propagation). Capture can itself
substitute an allocation-related exception, so do not promise perfect error-text
preservation under global memory exhaustion. Query and diagnostic exceptions
must not smuggle borrowed rows or private allocator-backed owners to the caller.

The completion control needs to wake a scheduler that may already have been
destroyed by the time a handle is used. Store a `weak_ptr` to a small `Wakeup`
object, not a raw pointer to `Executor`. Calling `lock()` yields a temporary
strong owner or an empty pointer atomically; it does not extend the executor's
lifetime. This operation is defined in
[weak ownership observers](https://eel.is/c++draft/util.smartptr.weak.obs).
The weak-pointer *member itself* is accessed under the task mutex while submission
binds it and cancellation reads it.

Begin `include/strata/runtime/task.hpp` with the result and synchronization owners:

```cpp title="include/strata/runtime/task.hpp"
#ifndef STRATA_RUNTIME_TASK_HPP
#define STRATA_RUNTIME_TASK_HPP
#include "strata/query/complete_query.hpp"
#include "strata/runtime/cancel.hpp"

#include <array>
#include <atomic>
#include <cstddef>
#include <cstdint>
#include <chrono>
#include <condition_variable>
#include <exception>
#include <memory>
#include <mutex>
#include <optional>
#include <string_view>
#include <utility>

namespace strata::runtime {
enum class TaskState { created, queued, running, completed, failed, canceled };
inline bool terminal(TaskState state) noexcept {
    return state == TaskState::completed || state == TaskState::failed || state == TaskState::canceled;
}
struct TaskResult {
    TaskState state = TaskState::created;
    std::uint64_t sequence = 0, staged_groups = 0;
    std::size_t peak_bytes = 0, final_bytes = 0;
    query::SortStats events{}, groups{};  // Values, never escaped allocator-backed owners.
    int result_cleanup_error = 0;
    std::exception_ptr error;
};
namespace task_detail {
struct Wakeup { std::mutex mutex; std::condition_variable changed, observed; };
struct Control {
    std::atomic<bool> stop{false};
    mutable std::mutex mutex;
    std::condition_variable done;
    TaskResult result;
    std::weak_ptr<Wakeup> wakeup;
};
}  // namespace task_detail
```

`SortStats` is copied by value after the plans die. Its fixed arrays preserve
spill-directory names and cleanup reports without preserving a query allocator.
`staged_groups` is not an acknowledgment of result publication. `final_bytes`
will be sampled after every query-owned charged allocation has been released;
the normal invariant is zero.

There are two condition variables in `Wakeup`. `changed` wakes workers after new
work, cancellation, capacity release, or closure. `observed` wakes only a
controller-side test waiting to see blocked workers. Combining those channels
would let idle workers notify one another merely for entering the wait state,
creating a wake/recheck cycle with no useful work.

Finish the header with the caller handle and fixed descriptor:

```cpp title="include/strata/runtime/task.hpp"
class Executor;
class Request;
class TaskHandle {
public:
    TaskHandle() noexcept = default;
    void cancel() const;
    TaskState state() const;
    TaskResult wait() const;
    std::optional<TaskResult> wait_for(std::chrono::milliseconds timeout) const;
private:
    friend class Request;
    explicit TaskHandle(std::shared_ptr<task_detail::Control> control) : control_(std::move(control)) {}
    void require_valid() const;
    std::shared_ptr<task_detail::Control> control_;
};
class Request {
public:
    Request(query::QueryOptions options, std::size_t working_bytes, std::string_view output_name);
    Request(const Request&) = delete;
    Request& operator=(const Request&) = delete;
    TaskHandle handle() const { return TaskHandle{control_}; }
    // Optional borrowed diagnostics; configure before submit and keep contexts alive until completion.
    CheckpointHook checkpoints{};
    query::QueryHooks query_hooks{};
    query::ResultHook result_hook{};
    std::optional<std::size_t> fail_allocation_on;
private:
    friend class Executor;
    query::QueryOptions options_;
    std::size_t working_bytes_;
    std::array<char, 128> output_{};
    std::shared_ptr<task_detail::Control> control_;
};
}  // namespace strata::runtime
#endif
```

The descriptor has a bounded basename and a fixed-size query configuration.
`handle()` copies only the control owner. `submit` will accept a reference to a
`unique_ptr<Request>` and move it only on success, preserving Chapter 18's
rejection-ownership contract.

The public hooks are diagnostic borrows, not a general task API. Configure them
before submission and keep their contexts alive until terminal completion. Do
not modify a submitted descriptor through a saved raw pointer. A checkpoint hook
that pauses at a rendezvous must eventually be released by its test controller;
the executor cannot cooperatively interrupt arbitrary user callback code.

## Cancel a resource waiter without losing the wakeup

The scheduler will examine stop flags while holding its mutex and then wait on
`changed`. Storing an atomic flag and calling `notify_all()` alone is insufficient:
the notification could occur after the predicate check but before the worker
actually enters its wait.

Use a short mutex handshake. Cancellation stores the flag, obtains a strong
`Wakeup` owner under the task mutex, releases that mutex, then locks and unlocks
the scheduler mutex before notifying. Consider a worker that has already read
"not canceled" while holding that mutex. Cancellation cannot finish the
handshake until the worker releases the mutex as part of entering its wait.
The following notification then wakes it. Conversely, if the handshake completes
first, the worker's next predicate check sees the stop request.

The condition-variable wait atomically releases its associated mutex and blocks,
then reacquires the mutex before returning; it may wake spuriously. Those rules
come from [condition variables](https://eel.is/c++draft/thread.condition.condvar).
Our complete predicate loop, not the notification, decides permission.

Create the beginning of `src/runtime/task.cpp`:

```cpp title="src/runtime/task.cpp"
#include "strata/runtime/task.hpp"
#include <algorithm>
#include <stdexcept>

namespace strata::runtime {
void TaskHandle::require_valid() const {
    if (!control_) { throw std::logic_error("empty task handle"); }
}
void TaskHandle::cancel() const {
    require_valid();
    std::shared_ptr<task_detail::Wakeup> wakeup;
    {
        std::lock_guard lock{control_->mutex};
        control_->stop.store(true, std::memory_order_relaxed);
        wakeup = control_->wakeup.lock();
    }
    if (wakeup) {
        // Pair with the scheduler's predicate-and-wait interval; avoid a lost wakeup.
        { std::lock_guard lock{wakeup->mutex}; }
        wakeup->changed.notify_all();
    }
}
```

Submission and worker dispatch sometimes hold the scheduler mutex and then take
a task mutex. Cancellation deliberately releases the task mutex *before* taking
the scheduler mutex. Holding both in the opposite order would create a deadlock
cycle. The otherwise empty lock scope is therefore meaningful synchronization,
not redundant locking.

Pre-submission cancellation is also legal. Its stop flag stays set. If submission
wins first, it binds the wakeup under the same task mutex; if cancellation wins
first, submission's own notification wakes workers to observe the already-set
flag. Neither interleaving leaves accepted canceled work asleep indefinitely.

State inspection and completion waits use only the task mutex:

```cpp title="src/runtime/task.cpp"
TaskState TaskHandle::state() const {
    require_valid();
    std::lock_guard lock{control_->mutex};
    return control_->result.state;
}
TaskResult TaskHandle::wait() const {
    require_valid();
    std::unique_lock lock{control_->mutex};
    if (control_->result.state == TaskState::created) { throw std::logic_error("task was not accepted"); }
    control_->done.wait(lock, [this] { return terminal(control_->result.state); });
    return control_->result;
}
std::optional<TaskResult> TaskHandle::wait_for(std::chrono::milliseconds timeout) const {
    require_valid();
    std::unique_lock lock{control_->mutex};
    if (control_->result.state == TaskState::created) { throw std::logic_error("task was not accepted"); }
    if (!control_->done.wait_for(lock, timeout, [this] { return terminal(control_->result.state); })) {
        return std::nullopt;
    }
    return control_->result;
}
```

Waiting on a never-accepted descriptor throws instead of waiting forever for an
owner that does not exist. `wait_for` returns an empty optional on timeout; it
does not cancel the task and does not establish a query deadline. The complete
result is copied while holding the mutex, after a terminal-state predicate has
succeeded. Do not expose references into `Control::result` after unlocking.

The constructor validates a result basename, then copies it into a zero-filled
array. Leave one byte for the terminating zero. Restrict names to lowercase
ASCII letters, digits, dot, dash, and underscore, with a letter/digit first.
This avoids treating simple case variants as distinct concurrent destinations
on a case-insensitive filesystem. It is not a sandbox against hostile directory
changes. One executor must exclusively control its output namespace, with no
other publisher replacing those names.

```cpp title="src/runtime/task.cpp"
Request::Request(query::QueryOptions options, std::size_t working_bytes, std::string_view output_name)
    : options_(options), working_bytes_(working_bytes), control_(std::make_shared<task_detail::Control>()) {
    if (working_bytes == 0 || output_name.empty() || output_name.size() >= output_.size() ||
        output_name == "." || output_name == ".." || output_name.find('/') != std::string_view::npos ||
        output_name.find('\0') != std::string_view::npos) {
        throw std::invalid_argument("invalid bounded query request");
    }
    const auto letter_or_digit = [](char c) { return (c >= 'a' && c <= 'z') || (c >= '0' && c <= '9'); };
    if (!letter_or_digit(output_name.front()) || !std::all_of(output_name.begin(), output_name.end(),
        [&](char c) { return letter_or_digit(c) || c == '-' || c == '_' || c == '.'; })) {
        throw std::invalid_argument("output names use lowercase ASCII letters, digits, dot, dash, underscore");
    }
    std::copy(output_name.begin(), output_name.end(), output_.begin());
}
}  // namespace strata::runtime
```

The executor prevents matching names only while requests are queued or running.
Later intentional reuse replaces the completed file. A retained handle is an
immutable record of that task's outcome, not a permanent snapshot of a pathname
that future tasks may replace.

## Keep scheduler storage fixed and query storage private

Create `include/strata/runtime/executor.hpp`:

```cpp title="include/strata/runtime/executor.hpp"
#ifndef STRATA_RUNTIME_EXECUTOR_HPP
#define STRATA_RUNTIME_EXECUTOR_HPP
#include "strata/runtime/task.hpp"
#include "strata/runtime/frozen_inputs.hpp"

#include <array>
#include <atomic>
#include <cstddef>
#include <memory>
#include <optional>
#include <string>
#include <thread>

namespace strata::runtime {
struct ExecutorOptions {
    std::size_t workers = 2, queued = 8;
    std::size_t working_bytes = 32 * 1024 * 1024, file_descriptors = 128;
};
struct ExecutorStats {
    std::size_t queued = 0, running = 0, waiting = 0;
    std::size_t used_bytes = 0, used_files = 0, peak_bytes = 0, peak_running = 0;
};
enum class Submit { accepted, full, closed, impossible, output_busy };
class Executor {
public:
    static constexpr std::size_t max_workers = 8, max_queued = 64;
    Executor(std::shared_ptr<const FrozenInputs> inputs, std::string temporary_parent,
             std::string output_parent, ExecutorOptions options = {},
             std::optional<std::size_t> fail_before_launch = std::nullopt,
             std::atomic<std::size_t>* live_threads = nullptr);
    ~Executor() noexcept;
    Executor(const Executor&) = delete;
    Executor& operator=(const Executor&) = delete;
    [[nodiscard]] Submit submit(std::unique_ptr<Request>& request);
    void close();                    // Stop admission; drain accepted requests.
    void cancel_all();               // Stop admission; request cancellation of all work.
    void finish();                   // Controller only: close and join, then return.
    ExecutorStats stats() const;
    void await_waiters(std::size_t count);  // Bounded diagnostic rendezvous, not scheduling.
private:
    struct Dispatch { std::unique_ptr<Request> request; bool admitted = false; std::size_t files = 0; };
    Dispatch take(std::size_t worker);
    std::unique_ptr<Request> remove(std::size_t at) noexcept;
    void worker(std::size_t index) noexcept;
    TaskResult execute(Request& request) noexcept;
    std::size_t files_for(const Request& request) const;
    bool fits(const Request& request) const;
    void join_all() noexcept;
    static void publish(Request& request, TaskResult result);
    std::shared_ptr<const FrozenInputs> inputs_;
    std::string temporary_parent_, output_parent_;
    ExecutorOptions options_;
    std::shared_ptr<task_detail::Wakeup> wakeup_ = std::make_shared<task_detail::Wakeup>();
    std::array<std::unique_ptr<Request>, max_queued> queue_{};
    std::array<Request*, max_workers> running_{};
    ExecutorStats counts_{};
    bool accepting_ = true;
    std::atomic<std::size_t>* live_threads_;
    std::array<std::thread, max_workers> threads_{};
};
}  // namespace strata::runtime
#endif
```

`queue_` has 64 physical slots and a configured logical capacity no larger than
that. Removing a middle canceled descriptor shifts at most 63 unique pointers;
this bounded linear operation makes the cancellation rule explicit. We do not
need a general lock-free queue.

`running_` contains non-owning pointers protected by the scheduler mutex. The
corresponding worker's `Dispatch` owns each request. The pointer is installed
before execution and cleared before request destruction. Shutdown uses it only
to set the atomic stop flag. It never follows it into a query's private data.

Every member is constructed before any worker starts. `threads_` is declared
last, but declaration order alone would not join it: the destructor body must
request termination and join while all earlier members still exist. Only the
controller calls `finish()` or destroys the executor. Do not race two joins,
call `finish()` from a worker, or destroy the executor while another thread is
calling its public methods. Task handles have a separate safe lifetime.

## Start a partial set of workers, or clean up all of it

Begin `src/runtime/executor.cpp`:

```cpp title="src/runtime/executor.cpp"
#include "strata/runtime/executor.hpp"
#include "strata/core/checked.hpp"

#include <algorithm>
#include <chrono>
#include <stdexcept>
#include <utility>

namespace strata::runtime {
Executor::Executor(std::shared_ptr<const FrozenInputs> inputs, std::string temporary_parent,
                   std::string output_parent, ExecutorOptions options,
                   std::optional<std::size_t> fail_before_launch,
                   std::atomic<std::size_t>* live_threads)
    : inputs_(std::move(inputs)), temporary_parent_(std::move(temporary_parent)),
      output_parent_(std::move(output_parent)), options_(options), live_threads_(live_threads) {
    if (!inputs_ || options.workers == 0 || options.workers > max_workers ||
        options.queued == 0 || options.queued > max_queued || options.working_bytes == 0 ||
        options.file_descriptors == 0 || temporary_parent_.empty() || output_parent_.empty() ||
        temporary_parent_.size() > 350 || output_parent_.size() > 350 ||
        temporary_parent_.find('\0') != std::string::npos || output_parent_.find('\0') != std::string::npos) {
        throw std::invalid_argument("invalid executor limits or paths");
    }
    try {
        for (std::size_t i = 0; i < options_.workers; ++i) {
            if (fail_before_launch == i) { throw std::runtime_error("injected thread launch failure"); }
            threads_[i] = std::thread([this, i] { worker(i); });
        }
    } catch (...) { cancel_all(); join_all(); throw; }
}
Executor::~Executor() noexcept { cancel_all(); join_all(); }
```

A failed constructor does not run `~Executor`. The catch is therefore essential:
it closes admission, requests cancellation, wakes any already waiting workers,
and joins each successfully created thread before member destruction. The
optional injected ordinal exercises exactly that ownership path without needing
the operating system to run out of threads.

`[this, i]` captures a borrowed executor pointer and a copied worker index. The
object cannot move or disappear while that callable executes. Destruction uses
cancel-and-join rather than detach. A successful join waits for thread completion
and synchronizes with it; the thread-handle object itself still needs controlled
access. See [thread members](https://eel.is/c++draft/thread.thread.member).
A synchronization infrastructure failure in this `noexcept` shutdown path is
fatal; silently detaching would leave dangling access and is not a recovery.

The next functions implement whole-query permission and ownership transfer:

```cpp title="src/runtime/executor.cpp"
std::size_t Executor::files_for(const Request& request) const {
    // Both sorts can coexist. Include table files, two spill directories, and result file.
    auto files = checked_size_add(inputs_->source_count(), 3);
    files = checked_size_add(files, request.options_.event_sort.file_descriptors);
    return checked_size_add(files, request.options_.group_sort.file_descriptors);
}
bool Executor::fits(const Request& request) const {
    return request.working_bytes_ <= options_.working_bytes - counts_.used_bytes &&
        files_for(request) <= options_.file_descriptors - counts_.used_files;
}
Submit Executor::submit(std::unique_ptr<Request>& request) {
    if (!request) { throw std::invalid_argument("empty request"); }
    const auto files = files_for(*request);  // May reject overflow without taking ownership.
    std::lock_guard lock{wakeup_->mutex};
    if (!accepting_) { return Submit::closed; }
    if (request->working_bytes_ > options_.working_bytes || files > options_.file_descriptors) {
        return Submit::impossible;
    }
    const std::string_view name{request->output_.data()};
    for (std::size_t i = 0; i < counts_.queued; ++i) {
        if (name == queue_[i]->output_.data()) { return Submit::output_busy; }
    }
    for (auto* active : running_) {
        if (active && name == active->output_.data()) { return Submit::output_busy; }
    }
    if (counts_.queued == options_.queued) { return Submit::full; }
    {
        std::lock_guard task_lock{request->control_->mutex};
        if (request->control_->result.state != TaskState::created) { throw std::logic_error("reused task"); }
        request->control_->wakeup = wakeup_;
        request->control_->result.state = TaskState::queued;
    }
    queue_[counts_.queued++] = std::move(request);
    wakeup_->changed.notify_all();
    return Submit::accepted;
}
```

`fits` subtracts current reservations from the configured caps. Subtraction is
safe because all updates preserve `used <= cap` under the same mutex. Checked
addition in `files_for` rejects unrepresentable descriptor totals before taking
ownership. The request's configuration is immutable after acceptance, so its
validated claim cannot grow while queued.

Submission is nonblocking. `full` is transient caller backpressure: retain the
descriptor, perform other work, then retry or discard it. `impossible` requires a
smaller claim or a larger pool. `output_busy` requires waiting for the conflicting
operation or choosing a different name. `closed` will not become open again.
No return value silently destroys the caller's request. Invalid null/reused
requests throw as programming errors.

Accepted work waits internally for resources. Admission is FIFO among
noncanceled requests. A large head can delay a smaller follower even if the
latter fits; that deliberate head-of-line policy avoids a hidden best-fit
scheduler and has no operating-system fairness guarantee. Cancellation of any
queued descriptor is processed without requiring its resource claim to fit.

## Dispatch only after reserving every allowance

Append the queue removal and worker acquisition functions:

```cpp title="src/runtime/executor.cpp"
std::unique_ptr<Request> Executor::remove(std::size_t at) noexcept {
    auto result = std::move(queue_[at]);
    for (std::size_t i = at + 1; i < counts_.queued; ++i) { queue_[i - 1] = std::move(queue_[i]); }
    queue_[--counts_.queued].reset();
    return result;
}
Executor::Dispatch Executor::take(std::size_t worker_index) {
    std::unique_lock lock{wakeup_->mutex};
    for (;;) {
        // Resolve any canceled descriptor, including behind a resource-blocked FIFO head.
        for (std::size_t i = 0; i < counts_.queued; ++i) {
            if (queue_[i]->control_->stop.load(std::memory_order_relaxed)) {
                return {remove(i), false, 0};
            }
        }
        if (counts_.queued != 0 && fits(*queue_[0])) {
            auto request = remove(0);
            const auto files = files_for(*request);
            counts_.used_bytes += request->working_bytes_; counts_.used_files += files;
            ++counts_.running;
            counts_.peak_running = std::max(counts_.peak_running, counts_.running);
            counts_.peak_bytes = std::max(counts_.peak_bytes, counts_.used_bytes);
            running_[worker_index] = request.get();
            {
                std::lock_guard task_lock{request->control_->mutex};
                request->control_->result.state = TaskState::running;
            }
            return {std::move(request), true, files};
        }
        if (!accepting_ && counts_.queued == 0) { return {}; }
        ++counts_.waiting;
        wakeup_->observed.notify_all();
        wakeup_->changed.wait(lock);  // Every wake returns to the complete predicate loop.
        --counts_.waiting;
    }
}
```

Walk a pressure example with two workers, a 2 MiB pool, and 2 MiB requests A, B,
and C. A is running. B and C are queued; the spare worker cannot admit B and
waits. Cancel C. The wakeup returns the worker to the top, where the scan removes
C before checking B's resource claim. C becomes canceled without ever opening a
table, result file, or spill directory. Cancel B next and it also resolves. A can
then finish normally. Repeating this test with plentiful memory but a 10-file
allowance exercises descriptor pressure independently.

An admitted dispatch updates reservations, running count, peaks, the raw
running pointer, and task state under the scheduler mutex. It moves the request
out before releasing that lock. A canceled dispatch has no allowance and leaves
its state queued until terminal publication; another caller can observe that
brief intermediate state, but never a fabricated running state.

The wait is deliberately inside an explicit loop because there are several
predicates and actions. Closure ends an idle worker only when the queue is empty.
If accepted work remains, `close` drains it; a running worker's later resource
release wakes any resource-blocked colleagues. No operator executes under this
mutex.

## Destroy query owners before publishing completion

Append result publication and the worker loop:

```cpp title="src/runtime/executor.cpp"
void Executor::publish(Request& request, TaskResult result) {
    {
        std::lock_guard lock{request.control_->mutex};
        if (!terminal(result.state) || terminal(request.control_->result.state)) { std::terminate(); }
        request.control_->result = std::move(result);
    }
    request.control_->done.notify_all();
}
void Executor::worker(std::size_t index) noexcept {
    if (live_threads_) { live_threads_->fetch_add(1, std::memory_order_relaxed); }
    for (;;) {
        auto dispatch = take(index);
        if (!dispatch.request) { break; }
        TaskResult result;
        if (dispatch.admitted) {
            result = execute(*dispatch.request);
            {
                std::lock_guard lock{wakeup_->mutex};
                running_[index] = nullptr;
                counts_.used_bytes -= dispatch.request->working_bytes_;
                counts_.used_files -= dispatch.files;
                --counts_.running;
            }
            wakeup_->changed.notify_all();
        } else { result.state = TaskState::canceled; }
        // All query owners have been destroyed, and allowances returned, before terminal status.
        publish(*dispatch.request, std::move(result));
    }
    if (live_threads_) { live_threads_->fetch_sub(1, std::memory_order_relaxed); }
}
```

`execute` returns only after its local plans, views, sinks, and statistics owners
are gone. The worker then returns reservations and clears the running pointer,
and only afterward makes the terminal result visible. A caller whose `wait()`
returns can rely on query memory/descriptor teardown being complete. Later work
may already have consumed the returned allowance; a global used counter need
not be zero while unrelated tasks run.

The queue does not destroy an arbitrary plan under the scheduler mutex because
queued descriptors contain no plan. A worker's descriptor itself dies outside
the lock at the end of the loop iteration. `publish` takes only the task mutex,
checks that a terminal result is not being overwritten, and notifies waiters.
The one worker owner supplies exactly one publication; the mutex supplies
visibility, not a second ownership arbitration scheme.

The worker function is `noexcept`. Ordinary query exceptions must be caught by
`execute`; an uncaught scheduler/invariant failure is not reported as a successful
query. The `live_threads` pointer is optional test instrumentation and must
outlive the executor. It is not used as a substitute for joining.

## One call scope owns the query and its cancellation token

Append `execute`:

```cpp title="src/runtime/executor.cpp"
TaskResult Executor::execute(Request& request) noexcept {
    TaskResult result;
    result.state = TaskState::failed;
    Cancellation cancellation{request.control_->stop, request.checkpoints};
    CancellationScope scope{&cancellation};
    std::shared_ptr<memory::Budget> budget;
    std::shared_ptr<query::SortStats> events, groups;
    try {
        checkpoint(Boundary::admitted);  // Reservation exists; no snapshot or workspace yet.
        budget = std::make_shared<memory::Budget>(request.working_bytes_);
        if (request.fail_allocation_on) { budget->fail_allocation_on(*request.fail_allocation_on); }
        query::CsvResult sink{output_parent_, request.output_.data(), request.result_hook};
        try {
            auto view = inputs_->capture(budget);
            result.sequence = view.sequence();
            events = query::make_sort_stats(budget); groups = query::make_sort_stats(budget);
            auto plan = query::make_query(std::move(view), request.options_, temporary_parent_,
                                         events, groups, request.query_hooks);
            query::consume_results(std::move(plan), request.options_.output_rows, sink, result.staged_groups);
            result.state = TaskState::completed;
        } catch (const Canceled&) { result.state = TaskState::canceled; }
          catch (...) { result.error = std::current_exception(); }
        sink.abort();  // Does not remove a published destination.
        result.result_cleanup_error = sink.cleanup_error();
    } catch (const Canceled&) { result.state = TaskState::canceled; }
      catch (...) { result.error = std::current_exception(); }
    if (events) { result.events = *events; }
    if (groups) { result.groups = *groups; }
    events.reset(); groups.reset();
    if (budget) { result.peak_bytes = budget->peak(); result.final_bytes = budget->used(); }
    return result;
}
```

Read this function from the scopes inward. The outer cancellation scope borrows
the still-owned request's stop flag. The `admitted` checkpoint happens after the
scheduler's reservation but before any snapshot or workspace. Only then do we
create a private `Budget`, result sink, isolated view, two stats owners, and the
complete query plan.

The inner `try` owns `view` and `plan`; both disappear before its catch executes.
`consume_results` takes the plan by value, owns its output batch, and aborts the
sink on failure. The sink is outside that inner scope so we can record its cleanup
status after unwinding. `abort` is safe after successful publication and does not
remove the completed destination. The outer catch also covers failures before
the inner scope exists, such as budget or sink construction.

`Canceled` is classified separately and carries no worker error. Other exceptions
leave the default `failed` state and populate `error`. Successful consumption sets
`completed` without rereading the stop flag. This preserves Part 1's publication
rule when cancellation arrives during the masked finish region.

After the sink has left scope, copy the two fixed-size statistics values, release
their charged owners, and sample the ledger. The invariant is

$$
C_i\text{ at terminal publication}=0.
$$

The `Budget` object's own small standard control allocation is outside its ledger;
the ledger covers allocations requested *through* it, including read metadata,
cache storage, and operator buffers. Exception objects and caller-held handles
are also outside that ledger. Do not generalize this check to "the process uses
zero memory."

If spill unlink is refused, file descriptors and working allocations still die,
but disk bytes can remain. The copied statistics retain the directory, live-file
count, reserved disk bytes, and cleanup errors. The runtime does not release
those physical bytes by decrementing a memory counter. Nor does it automatically
retry arbitrary directories after the task's owner is gone. A controller must
surface the report and explicitly resolve known orphan paths; repeated cleanup
failures can accumulate disk use beyond a sequence of per-query caps.

## Close, cancel, wake, join

Finish `src/runtime/executor.cpp`:

```cpp title="src/runtime/executor.cpp"
void Executor::close() {
    { std::lock_guard lock{wakeup_->mutex}; accepting_ = false; }
    wakeup_->changed.notify_all();
}
void Executor::cancel_all() {
    {
        std::lock_guard lock{wakeup_->mutex};
        accepting_ = false;
        for (std::size_t i = 0; i < counts_.queued; ++i) {
            queue_[i]->control_->stop.store(true, std::memory_order_relaxed);
        }
        for (auto* request : running_) {
            if (request) { request->control_->stop.store(true, std::memory_order_relaxed); }
        }
    }
    wakeup_->changed.notify_all();
}
void Executor::join_all() noexcept {
    for (auto& thread : threads_) { if (thread.joinable()) { thread.join(); } }
}
void Executor::finish() { close(); join_all(); }
ExecutorStats Executor::stats() const {
    std::lock_guard lock{wakeup_->mutex}; return counts_;
}
void Executor::await_waiters(std::size_t count) {
    std::unique_lock lock{wakeup_->mutex};
    if (!wakeup_->observed.wait_for(lock, std::chrono::seconds{5},
            [&] { return counts_.waiting >= count; })) {
        throw std::runtime_error("executor waiter watchdog expired");
    }
}
}  // namespace strata::runtime
```

`close()` rejects new submissions but keeps accepted work. `finish()` closes and
joins, so it drains by default. `cancel_all()` rejects new submissions, marks
queued and running requests, and wakes all resource waiters. The destructor uses
that cancel path before joining. A running request that already passed the
publication gate may still complete successfully; shutdown does not falsify its
result.

`cancel_all` touches only fixed scheduler state and atomic flags while holding
the mutex. It never waits for a query's private budget lock, runs a callback, or
performs filesystem I/O there. Waiting for workers happens in `join_all` with no
scheduler lock held. Otherwise the controller could block on workers that need
that very lock to exit.

The diagnostic `await_waiters` waits for an observable count, not for an arbitrary
sleep duration. Its five-second limit is a test watchdog, not a scheduling
assumption or performance requirement. Production code does not need to call it.

## Read the resource protocol as one trace

For a successful query, the sequence is:

```text
caller owns Request; caller may retain TaskHandle
  -> submit transfers Request into fixed queue
  -> worker reserves memory cap and descriptor allowance
  -> private budget and snapshot/readers are constructed
  -> both sorts spill; join and aggregation produce staged groups
  -> final cancellation check passes
  -> sink publishes complete result
  -> plans, views, buffers, sink and stats owners are destroyed
  -> worker returns resource allowances
  -> terminal TaskResult becomes visible to all waiting handles
```

A queued cancellation skips every step from private-budget construction through
resource return because it acquired nothing. A running failure takes the unwind
path through those owners and still returns its reservation. One failed query
does not poison the executor; the next admissible request can run.

The controller must retain borrowed diagnostic contexts until the corresponding
terminal result. It must stop external submissions before destroying the
executor. It must not retain a mutable `Database` beside this immutable fixture.
These are concrete lifetime and access requirements, not things a worker-count
limit can enforce automatically.

[Part 3](https://25d-228.github.io/cplusplus-learning/chapter-19-part-3.html) makes
these traces observable with rendezvous, cancellation at precise boundaries,
independent CSV answers, failure injection, and a deliberately wrong but race-free
completion rule.

