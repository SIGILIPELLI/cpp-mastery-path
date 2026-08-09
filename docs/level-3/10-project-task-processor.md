# 10 · Project — Multi-threaded Task Processor

This project is the capstone of Level 3. You'll build a **thread pool**: a
fixed set of worker threads that pull work items off a shared queue and run
them, so a program can process a batch of independent jobs in parallel without
spawning a thread per job.

Everything Level 3 covered shows up here. The queue is guarded by a
[mutex and condition variable](03-concurrency.md); the pool owns its threads
and joins them in its destructor, which is [RAII](04-raii-deep-dive.md) applied
to threads; the queue is a [template](01-advanced-templates.md) over the element
type; tasks are moved into the queue rather than copied
([move semantics](02-move-semantics-rvalue-references.md)); and the whole thing
is worth [profiling](09-performance-profiling.md) at the end.

## What you'll build

A `ThreadPool` that:

- Starts *N* worker threads on construction and joins them all on destruction
- Accepts any `void()` callable via `submit()` — usually a lambda capturing its
  own data
- Blocks idle workers on a condition variable instead of spinning
- Shuts down cleanly: `close()` wakes every worker, each drains remaining work,
  then exits
- Counts completed and failed tasks with `std::atomic`, and contains an
  exception thrown by a task so one bad job cannot kill a worker

## Project layout

```text
taskproc/
    include/
        task_queue.h     # thread-safe queue template (header-only)
        thread_pool.h    # pool interface
    src/
        thread_pool.cpp  # pool implementation
        main.cpp         # driver: submit a batch of document jobs
```

## include/task_queue.h — the thread-safe queue

```cpp
// task_queue.h
#ifndef TASK_QUEUE_H
#define TASK_QUEUE_H

#include <condition_variable>
#include <mutex>
#include <optional>
#include <queue>

template <typename T>
class TaskQueue {
public:
    void push(T value) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            if (closed_) return;              // refuse work after shutdown starts
            queue_.push(std::move(value));
        }                                     // unlock BEFORE notifying: the woken
        cv_.notify_one();                     // thread would otherwise block again
    }

    // Returns std::nullopt only when the queue is closed AND drained.
    std::optional<T> pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        cv_.wait(lock, [this] { return !queue_.empty() || closed_; });
        if (queue_.empty()) return std::nullopt;
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    void close() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            closed_ = true;
        }
        cv_.notify_all();                     // wake EVERY waiting worker, not one
    }

    std::size_t size() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return queue_.size();
    }

private:
    mutable std::mutex mutex_;                // mutable: lockable from const size()
    std::condition_variable cv_;
    std::queue<T> queue_;
    bool closed_ = false;
};

#endif
```

Three details carry the correctness of this class:

`cv_.wait(lock, predicate)` is the predicate form, and it is not optional
politeness. A condition variable is allowed to wake a thread with no
notification at all (a *spurious wakeup*); the predicate form re-checks the
condition in a loop and goes back to sleep if it isn't satisfied.

The predicate is `!queue_.empty() || closed_`, not just `!queue_.empty()`.
Without the `|| closed_` term, a worker that goes to sleep on an empty queue
after `close()` has already run would never be woken again, and the destructor's
`join()` would hang forever.

`mutex_` is `mutable` so `size()` can stay `const` while still locking. Locking
is not a logical mutation of the queue.

## include/thread_pool.h — the pool interface

```cpp
// thread_pool.h
#ifndef THREAD_POOL_H
#define THREAD_POOL_H

#include "task_queue.h"

#include <atomic>
#include <functional>
#include <thread>
#include <vector>

using Task = std::function<void()>;

class ThreadPool {
public:
    explicit ThreadPool(unsigned workers);
    ~ThreadPool();

    // A pool owns OS threads. Copying it makes no sense, so delete the copy
    // operations rather than let the compiler generate something broken.
    ThreadPool(const ThreadPool&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

    void submit(Task task);
    void shutdown();

    unsigned long completed() const { return completed_.load(); }
    unsigned long failed() const { return failed_.load(); }

private:
    void workerLoop(unsigned id);

    TaskQueue<Task> queue_;
    std::vector<std::thread> threads_;
    std::atomic<unsigned long> completed_{0};
    std::atomic<unsigned long> failed_{0};
    bool stopped_ = false;
};

#endif
```

`std::function<void()>` type-erases the callable, so `submit()` accepts a
lambda, a function pointer, or a functor with no template machinery at the call
site. The counters are `std::atomic` because several workers increment them
concurrently — a plain `unsigned long` there is a data race and therefore
undefined behaviour, even though `++` "looks" atomic.

## src/thread_pool.cpp — the implementation

```cpp
// thread_pool.cpp
#include "thread_pool.h"

#include <iostream>
#include <mutex>

namespace {
std::mutex g_coutMutex;   // std::cerr from many threads interleaves without this
}

ThreadPool::ThreadPool(unsigned workers) {
    threads_.reserve(workers);
    for (unsigned i = 0; i < workers; ++i)
        threads_.emplace_back(&ThreadPool::workerLoop, this, i);
}

ThreadPool::~ThreadPool() { shutdown(); }

void ThreadPool::submit(Task task) { queue_.push(std::move(task)); }

void ThreadPool::shutdown() {
    if (stopped_) return;        // idempotent: safe to call explicitly AND from ~ThreadPool
    stopped_ = true;
    queue_.close();
    for (auto& t : threads_)
        if (t.joinable()) t.join();
}

void ThreadPool::workerLoop(unsigned id) {
    while (true) {
        auto task = queue_.pop();
        if (!task) break;                  // nullopt == closed and drained
        try {
            (*task)();
            completed_.fetch_add(1);
        } catch (const std::exception& e) {
            // An exception escaping a thread's entry function calls
            // std::terminate() and kills the whole process. Catch it here.
            failed_.fetch_add(1);
            std::lock_guard<std::mutex> lock(g_coutMutex);
            std::cerr << "[worker " << id << "] task threw: " << e.what() << "\n";
        }
    }
}
```

The `try`/`catch` around `(*task)()` is the single most important line in the
file. An exception that propagates out of the function a `std::thread` was
started with does **not** unwind into the parent — it calls `std::terminate()`
and aborts the process. Every worker loop in production code needs this guard.

## src/main.cpp — the driver

```cpp
// main.cpp
#include "thread_pool.h"

#include <atomic>
#include <cctype>
#include <chrono>
#include <iostream>
#include <mutex>
#include <stdexcept>
#include <string>
#include <thread>
#include <vector>

std::mutex outMutex;

void log(const std::string& msg) {
    std::lock_guard<std::mutex> lock(outMutex);
    std::cout << msg << "\n";
}

// A deliberately slow "job" so parallelism is visible in the timings.
std::size_t countLetters(const std::string& text) {
    std::this_thread::sleep_for(std::chrono::milliseconds(20));
    std::size_t n = 0;
    for (char c : text)
        if (std::isalpha(static_cast<unsigned char>(c))) ++n;
    return n;
}

int main() {
    const unsigned workers = 4;
    std::vector<std::string> documents = {
        "the quick brown fox", "jumps over the lazy dog",
        "c++ concurrency in action", "raii owns the resource",
        "move semantics avoid copies", "templates are compile time",
        "sqlite stores rows", "google test asserts", "<<BAD>>",
        "profile before you optimize"
    };

    std::atomic<std::size_t> totalLetters{0};
    auto start = std::chrono::steady_clock::now();

    {
        ThreadPool pool(workers);
        for (std::size_t i = 0; i < documents.size(); ++i) {
            std::string doc = documents[i];         // capture BY VALUE below --
            pool.submit([i, doc, &totalLetters] {   // the task outlives this loop
                if (doc == "<<BAD>>")
                    throw std::runtime_error("document " + std::to_string(i) +
                                             " is corrupt");
                std::size_t n = countLetters(doc);
                totalLetters.fetch_add(n);
                log("doc " + std::to_string(i) + " -> " + std::to_string(n) +
                    " letters");
            });
        }
        log("all " + std::to_string(documents.size()) + " tasks submitted");
    }   // pool destructor -> shutdown() -> close() -> join every worker

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
                  std::chrono::steady_clock::now() - start).count();

    std::cout << "\ntotal letters: " << totalLetters.load() << "\n";
    std::cout << "elapsed: " << ms << " ms with " << workers << " workers\n";
    std::cout << "(serial would be about " << documents.size() * 20 << " ms)\n";
}
```

The lambda captures `doc` **by value**. Capturing `documents[i]` by reference
would be a dangling reference the moment the loop variable moves on, and the
bug would only appear under timing that lets the worker run late — the worst
kind of concurrency bug to debug.

## Building it

```bash
g++ -std=c++17 -O2 -Wall -Wextra -pthread -Iinclude \
    -o taskproc src/thread_pool.cpp src/main.cpp
```

`-pthread` is required at both compile and link time on Linux; on macOS it is
harmless. A CMake version, following [Level 2 Module 9](../level-2/09-build-tools-cmake.md):

```cmake
cmake_minimum_required(VERSION 3.16)
project(taskproc CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Threads REQUIRED)
add_executable(taskproc src/thread_pool.cpp src/main.cpp)
target_include_directories(taskproc PRIVATE include)
target_link_libraries(taskproc PRIVATE Threads::Threads)
```

## Running it

```text
all 10 tasks submitted
doc 0 -> 16 letters
doc 3 -> 19 letters
doc 2 -> 20 letters
doc 1 -> 19 letters
doc 4 -> 24 letters
doc 7 -> 17 letters
[worker 0] task threw: document 8 is corrupt
doc 6 -> 16 letters
doc 5 -> 23 letters
doc 9 -> 24 letters

total letters: 178
elapsed: 68 ms with 4 workers
(serial would be about 200 ms)
```

Read the output carefully, because two things in it are the whole point.

**The line order is not the submission order** — `doc 3` finishes before
`doc 1`, and which worker gets which document is decided by the OS scheduler.
Your run will interleave differently. `total letters: 178` is nevertheless
identical every time, because the accumulation goes through a `std::atomic`.
Non-deterministic *scheduling* with deterministic *results* is exactly what a
correctly synchronized program looks like.

**The corrupt document did not stop anything.** Worker 0 caught the exception,
counted a failure, and immediately went back for more work. Docs 5, 6 and 9
were processed afterwards.

And 68 ms against a serial 200 ms is the parallel speedup — under 4x because
ten 20 ms jobs across four workers takes three rounds, not two and a half.

## Verify it with tests

The queue is the piece worth testing directly, using
[Google Test](08-testing-googletest.md):

```cpp
TEST(TaskQueueTest, PopReturnsNulloptAfterCloseAndDrain) {
    TaskQueue<int> q;
    q.push(1);
    q.close();
    EXPECT_EQ(q.pop().value(), 1);   // drains what was already queued
    EXPECT_FALSE(q.pop().has_value());
}

TEST(ThreadPoolTest, RunsEveryTaskExactlyOnce) {
    std::atomic<int> counter{0};
    { ThreadPool pool(4); for (int i = 0; i < 1000; ++i) pool.submit([&]{ ++counter; }); }
    EXPECT_EQ(counter.load(), 1000);   // destructor joined -> all work is done
}
```

Run these under `--gtest_repeat=100`. Concurrency bugs pass once and fail on the
fortieth run; a single green test proves very little.

## Stretch goals

- Make `submit()` return a `std::future<T>` so a caller can retrieve a task's
  *result*, not just fire and forget. Wrap the callable in a
  `std::packaged_task<T()>` and return its `get_future()` — this turns the pool
  into something close to a real executor.
- Add a `waitIdle()` that blocks until the queue is empty **and** no worker is
  mid-task. You'll need a second condition variable and an active-task counter;
  getting the "and no worker is mid-task" half right is the interesting part.
- Bound the queue: block `push()` once more than *K* tasks are pending, so a
  fast producer can't grow the queue without limit. This needs a second
  condition variable signalling "space available" and turns the class into a
  classic bounded producer-consumer buffer.
- Give each worker its own deque and implement **work stealing**: an idle worker
  takes from the back of a busy worker's deque. Measure with the techniques from
  [Module 9](09-performance-profiling.md) whether it actually helps for your
  task sizes — for 20 ms tasks it will not, and knowing why is the lesson.
- Replace the `sleep_for` job with real work — parse the SQLite rows from
  [Module 7](07-sqlite.md) in parallel, or fetch several URLs with the sockets
  from [Module 6](06-networking-basics.md) — and profile whether you are
  CPU-bound or I/O-bound.

Completing this project means you're ready for **Level 4 · Master**.
