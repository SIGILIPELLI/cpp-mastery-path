# 03 · Concurrency

Every program so far has run one instruction at a time. `std::thread` lets a
single process run multiple independent lines of execution at once — useful
for keeping a UI responsive while work happens in the background, or for
splitting a CPU-heavy computation across cores. The catch: as soon as two
threads touch the same memory, you need to actively prevent them from
tripping over each other. This module covers `std::thread`, `std::mutex`, and
the tools built around them. [Module 10](10-project-task-processor.md) turns
all of it into a working task processor.

## Starting and joining a thread

```cpp
#include <iostream>
#include <thread>

void greet(const std::string& name) {
    std::cout << "Hello from " << name << std::endl;
}

int main() {
    std::thread t(greet, "worker-1");   // starts running greet() concurrently, immediately

    std::cout << "Main thread continues..." << std::endl;

    t.join();    // block here until t finishes -- REQUIRED before t is destroyed
    std::cout << "Worker finished." << std::endl;
}
// Main thread continues...
// Hello from worker-1
// Worker finished.
```

The first two lines can print in **either order** — `main` and `t` are
racing. That's expected and fine here because they don't share any data. What
you must never skip is `join()` (wait for it to finish) or `detach()`
(explicitly let it run fully independently, forever, unsupervised). A
`std::thread` that's still joinable when its destructor runs calls
`std::terminate()` and kills the whole program — no exception, no
diagnostic, just an abrupt crash. This is the RAII lesson from
[Module 4](04-raii-deep-dive.md) applied to threads: every `std::thread`
needs exactly one of `join()`/`detach()` before it goes out of scope.

## The race condition

```cpp
#include <iostream>
#include <thread>

int counter = 0;

void incrementMany() {
    for (int i = 0; i < 100000; ++i) {
        counter++;    // NOT atomic: read, add 1, write back -- three separate steps
    }
}

int main() {
    std::thread t1(incrementMany);
    std::thread t2(incrementMany);
    t1.join();
    t2.join();
    std::cout << "counter = " << counter << std::endl;
}
// counter = 137482          <- expected 200000. The exact number varies every run.
```

`counter++` is not one CPU instruction — it's read, increment, write. If
thread A reads `counter` as 5, then thread B also reads it as 5 before A
writes back 6, both threads compute 6 and one increment is silently lost.
This is a **data race**: undefined behavior in the C++ standard, not just "a
wrong number" — the compiler is permitted to assume it never happens and
optimize accordingly, so the actual failure mode can be stranger than a
miscount.

## `std::mutex` — mutual exclusion

A mutex guarantees only one thread executes a given section of code at a
time.

```cpp
#include <iostream>
#include <thread>
#include <mutex>

int counter = 0;
std::mutex counterMutex;

void incrementMany() {
    for (int i = 0; i < 100000; ++i) {
        std::lock_guard<std::mutex> lock(counterMutex);   // locks on construction
        counter++;
    }   // lock released automatically here -- RAII, same idiom as smart pointers
}

int main() {
    std::thread t1(incrementMany);
    std::thread t2(incrementMany);
    t1.join();
    t2.join();
    std::cout << "counter = " << counter << std::endl;   // always 200000, every run
}
```

`std::lock_guard` is deliberately minimal: it locks in its constructor and
unlocks in its destructor, and that's it. Because it's RAII, the mutex is
released correctly even if an exception is thrown inside the locked section
— there is no `try/finally` needed, and no path where you forget to unlock.
**Never call `mutex.lock()`/`mutex.unlock()` by hand** in ordinary code; a
thrown exception between them leaves the mutex locked forever, deadlocking
every other thread that waits on it.

## `std::unique_lock` — when you need more control

`std::lock_guard` can't unlock early or be used with condition variables.
`std::unique_lock` is a more flexible RAII wrapper for those cases:

```cpp
#include <mutex>

std::mutex m;

void example() {
    std::unique_lock<std::mutex> lock(m);
    // ... critical section ...
    lock.unlock();          // manually release early, still exception-safe overall
    // ... non-critical work, other threads can now enter ...
    lock.lock();             // re-acquire if needed
}   // destructor unlocks again if still locked -- safe either way
```

Prefer `lock_guard` by default — it's slightly cheaper and communicates
"this lock's lifetime is exactly this scope." Reach for `unique_lock` only
when you need to unlock early or hand the lock to a `std::condition_variable`.

## Deadlock

```cpp
#include <mutex>
#include <thread>

std::mutex mA, mB;

void taskOne() {
    std::lock_guard<std::mutex> lockA(mA);
    std::lock_guard<std::mutex> lockB(mB);   // if taskTwo holds mB and wants mA now...
}

void taskTwo() {
    std::lock_guard<std::mutex> lockB(mB);
    std::lock_guard<std::mutex> lockA(mA);   // ...both threads wait forever. Deadlock.
}
```

Two threads, each holding one lock while waiting for the other's — neither
can proceed. **The fix: always acquire multiple mutexes in a fixed, global
order**, or lock them together atomically:

```cpp
#include <mutex>

void taskOne() {
    std::scoped_lock lock(mA, mB);   // C++17: locks both, deadlock-free, in any order
}

void taskTwo() {
    std::scoped_lock lock(mA, mB);   // same order argument -- std::scoped_lock sorts this out internally
}
```

`std::scoped_lock` (C++17) takes any number of mutexes and uses a
deadlock-avoidance algorithm internally, so callers don't need to agree on an
ordering by convention — prefer it whenever more than one mutex must be held
at once.

## `std::atomic` — lock-free for simple types

For a single counter, a full mutex is overkill. `std::atomic<T>` makes
individual operations indivisible in hardware, without an explicit lock.

```cpp
#include <iostream>
#include <thread>
#include <atomic>

std::atomic<int> counter{0};

void incrementMany() {
    for (int i = 0; i < 100000; ++i) {
        counter++;    // atomic increment -- a single indivisible hardware operation
    }
}

int main() {
    std::thread t1(incrementMany);
    std::thread t2(incrementMany);
    t1.join();
    t2.join();
    std::cout << "counter = " << counter << std::endl;   // always 200000
}
```

`std::atomic` only covers single operations on one variable (`++`, `+=`,
compare-and-swap). The moment correctness depends on *multiple* variables
staying consistent with each other, you need a mutex to protect the whole
group of them — atomics don't compose.

## `std::condition_variable` — waiting for a signal

A thread often needs to sleep until *another* thread says "there's work now,"
rather than repeatedly checking (a wasteful "busy loop").

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> jobs;
bool done = false;

void worker() {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        // Sleep until the predicate is true; avoids "spurious wakeup" bugs
        // and re-checks the condition automatically after each wake.
        cv.wait(lock, [] { return !jobs.empty() || done; });

        if (!jobs.empty()) {
            int job = jobs.front();
            jobs.pop();
            lock.unlock();
            std::cout << "processing job " << job << std::endl;
        } else if (done) {
            break;
        }
    }
}

int main() {
    std::thread w(worker);

    for (int i = 1; i <= 3; ++i) {
        { std::lock_guard<std::mutex> lock(mtx); jobs.push(i); }
        cv.notify_one();     // wake the worker: there's a job waiting
    }

    { std::lock_guard<std::mutex> lock(mtx); done = true; }
    cv.notify_one();
    w.join();
}
// processing job 1
// processing job 2
// processing job 3
```

`cv.wait(lock, predicate)` atomically releases the lock while sleeping (so
the producer can acquire it to push work) and re-acquires it before checking
the predicate again on wake. This exact pattern — mutex + condition_variable
+ shared queue — is the backbone of [Module 10's](10-project-task-processor.md)
thread pool.

## `std::async` and `std::future` — a result, not a thread

When you just want "run this and give me the return value later," `std::async`
is simpler than managing a `std::thread` and a shared variable by hand:

```cpp
#include <iostream>
#include <future>

int computeSquare(int x) { return x * x; }

int main() {
    std::future<int> result = std::async(std::launch::async, computeSquare, 12);

    std::cout << "doing other work..." << std::endl;

    std::cout << "square = " << result.get() << std::endl;   // blocks until ready
}
// doing other work...
// square = 144
```

`result.get()` blocks until the async task finishes and returns its value —
or rethrows an exception if the task threw one, propagating it across the
thread boundary automatically, which manual `std::thread` doesn't do for you.

## Cheat sheet

| Tool | Purpose |
|------|---------|
| `std::thread` | Start a concurrent function; must `join()` or `detach()` before destruction |
| `std::mutex` | Mutual exclusion — one thread in the critical section at a time |
| `std::lock_guard` | RAII lock, fixed scope, cheapest — the default choice |
| `std::unique_lock` | RAII lock, can unlock early or pair with a condition variable |
| `std::scoped_lock` | RAII lock over multiple mutexes at once, deadlock-free |
| `std::atomic<T>` | Lock-free indivisible ops on one variable |
| `std::condition_variable` | Sleep until another thread signals a condition, no busy-waiting |
| `std::async` / `std::future` | Run a function concurrently and collect its return value (or exception) later |

## Traps

**A data race is undefined behavior, not just "probably fine."** Even a
variable that "usually" reads correctly across threads without a mutex or
atomic is a bug that can manifest differently under a different compiler,
optimization level, or CPU — never rely on it happening to work.

**`join()` must run exactly once**, and calling it twice throws
`std::system_error`. If a thread might already have been joined, check
`t.joinable()` first.

**Forgetting to unlock manually-managed mutexes on an exception path** is why
`lock_guard`/`unique_lock` exist — never call `.lock()`/`.unlock()` directly
in code that can throw between them.

## Exercise

Build a thread-safe counter class `SafeCounter` wrapping an `int` and a
`std::mutex`, with `increment()` and `int value() const` methods (the latter
also needs to lock — reading while another thread writes is still a race).
Launch four threads, each calling `increment()` 50,000 times, `join()` them
all, and print the final value — confirm it's always exactly 200,000 across
several runs. Then replace the `int` + `mutex` with a single `std::atomic<int>`
and confirm the behaviour is identical but the code is shorter.
