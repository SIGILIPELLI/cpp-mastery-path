# 02 · Advanced Concurrency

[Level 3 Module 3](../level-3/03-concurrency.md) covered threads, mutexes and
condition variables — enough to build the
[thread pool](../level-3/10-project-task-processor.md). This module covers the
layer above that: getting *results* back from concurrent work with futures,
sharing data without locks using atomics, letting many readers proceed in
parallel with `shared_mutex`, and the C++20 coordination primitives (`jthread`,
`latch`, `barrier`) that remove most of the remaining boilerplate.

The theme throughout: **prefer the highest-level tool that expresses your
intent.** Hand-rolled atomics are a last resort, not a badge of skill.

## Futures — concurrency that returns a value

`std::thread` runs a function; it cannot give you the return value.
`std::async` runs a function and hands back a `std::future<T>` you call `.get()`
on.

```cpp
#include <chrono>
#include <future>
#include <iostream>
#include <vector>

bool isPrime(int n) {
    if (n < 2) return false;
    for (int d = 2; (long long)d * d <= n; ++d)
        if (n % d == 0) return false;
    return true;
}

int countPrimes(int lo, int hi) {
    int n = 0;
    for (int i = lo; i < hi; ++i) if (isPrime(i)) ++n;
    return n;
}

int main() {
    const int LIMIT = 2'000'000;
    auto ms = [](auto a, auto b) {
        return std::chrono::duration_cast<std::chrono::milliseconds>(b - a).count();
    };

    auto t0 = std::chrono::steady_clock::now();
    int serial = countPrimes(0, LIMIT);
    auto t1 = std::chrono::steady_clock::now();

    const int chunks = 4;
    std::vector<std::future<int>> futures;
    for (int i = 0; i < chunks; ++i)
        futures.push_back(std::async(std::launch::async, countPrimes,
                                     LIMIT * i / chunks, LIMIT * (i + 1) / chunks));

    int parallel = 0;
    for (auto& f : futures) parallel += f.get();   // blocks until that chunk is done
    auto t2 = std::chrono::steady_clock::now();

    std::cout << "serial   " << serial   << " primes in " << ms(t0, t1) << " ms\n";
    std::cout << "parallel " << parallel << " primes in " << ms(t1, t2) << " ms\n";
    std::cout << "hardware_concurrency = " << std::thread::hardware_concurrency() << "\n";
}
```

```text
serial   148933 primes in 200 ms
parallel 148933 primes in 45 ms
hardware_concurrency = 8
```

Four and a half times faster with four chunks — better than 4x because the
chunks are uneven (small numbers are cheap to test, so the last chunk dominates
and the machine had spare cores to overlap it).

Two things are doing real work here. `std::launch::async` **forces** a new
thread; the default policy is `async|deferred`, and a library that picks
`deferred` runs your function lazily on the calling thread at `.get()` time —
zero parallelism, silently. Always pass the policy explicitly. And `.get()`
does more than return the value: if the task threw, `.get()` **rethrows that
exception in the calling thread**. That is the clean solution to the problem
the thread pool had to solve with a `try`/`catch` inside the worker.

`std::promise<T>` is the manual version: one thread holds the promise and calls
`set_value()`/`set_exception()`, another holds `promise.get_future()` and waits.
Use it when the producer isn't a single function call — a callback, an I/O
completion, a message arriving on a socket.

## Data races are not theoretical

```cpp
#include <iostream>
#include <thread>
#include <vector>

long counter = 0;   // shared, unsynchronized

int main() {
    const int THREADS = 8, PER = 200000;
    std::vector<std::thread> ts;
    for (int i = 0; i < THREADS; ++i)
        ts.emplace_back([&]{ for (int j = 0; j < PER; ++j) ++counter; });
    for (auto& t : ts) t.join();
    std::cout << "expected " << (long)THREADS * PER << "\n";
    std::cout << "got      " << counter << "\n";
}
```

```text
expected 1600000
got      407451
```

```text
expected 1600000
got      398916
```

Three runs, three different wrong answers, and roughly **three quarters of the
increments vanished**. `++counter` is load, add, store; two threads that load
the same value both store the same result, and one increment is lost. Note also
that this was built at `-O0` — at `-O2` the compiler may keep `counter` in a
register for the whole loop and print the *correct* answer, which is worse,
because the bug is still there and will reappear when the code changes.

## Atomics

`std::atomic<T>` makes read-modify-write indivisible:

```cpp
std::atomic<long> counted{0};
// in each thread:
counted.fetch_add(1, std::memory_order_relaxed);
```

Comparing the three approaches on 8 threads × 200,000 increments:

```text
expected 1600000
plain long   407451   (WRONG -- data race)
mutex        1600000  in 74 ms
atomic       1600000  in 46 ms
atomic<long> lock free: true
```

The atomic is right *and* faster, because on x86-64 and AArch64 an
`atomic<long>` compiles to a single lock-prefixed instruction rather than a
mutex acquire/release pair. `is_lock_free()` tells you whether that's true on
your target — `std::atomic<SomeBigStruct>` silently falls back to an internal
lock, at which point a real mutex is the honest choice.

Atomics only cover *one* variable. The moment an invariant spans two variables
("`size_` must match the number of entries in `data_`"), you need a mutex —
two atomic operations are not one atomic transaction.

## Memory ordering, briefly

Every atomic operation takes an optional `std::memory_order`. The three that
matter:

| Order | Guarantee | Use for |
|-------|-----------|---------|
| `seq_cst` (default) | One global total order all threads agree on | Everything, until profiling says otherwise |
| `acquire` / `release` | A release-store *happens-before* an acquire-load that reads it; everything written before the store is visible after the load | Publishing data behind a flag or pointer |
| `relaxed` | Atomicity only — no ordering with respect to other variables | Standalone counters and statistics |

The counter above uses `relaxed` legitimately: nobody reads it to decide
whether *other* memory is ready. The classic acquire/release pattern is
different:

```cpp
Data* ptr = nullptr;
std::atomic<bool> ready{false};

// producer
ptr = new Data{...};
ready.store(true, std::memory_order_release);   // publishes everything above

// consumer
while (!ready.load(std::memory_order_acquire)) { }
use(*ptr);                                       // guaranteed to see the writes
```

With `relaxed` on both sides this is broken: the CPU or compiler may make
`ready` visible before `ptr`'s contents, and the consumer reads garbage. If you
are not certain which order you need, use the default `seq_cst`. It is the
slowest and the only one that is never subtly wrong.

## `shared_mutex` — many readers, one writer

A plain mutex serializes readers against each other for no reason. When reads
vastly outnumber writes, `std::shared_mutex` (C++17) lets them overlap:

```cpp
#include <map>
#include <shared_mutex>
#include <string>

class Registry {
public:
    int get(const std::string& k) const {
        std::shared_lock<std::shared_mutex> lock(mutex_);   // MANY readers at once
        auto it = map_.find(k);
        return it == map_.end() ? -1 : it->second;
    }
    void set(const std::string& k, int v) {
        std::unique_lock<std::shared_mutex> lock(mutex_);   // ONE writer, exclusive
        map_[k] = v;
    }
private:
    mutable std::shared_mutex mutex_;
    std::map<std::string, int> map_;
};
```

`std::shared_lock` takes shared (read) ownership; `std::unique_lock` takes
exclusive (write) ownership. A `shared_mutex` is heavier than a plain
`std::mutex` — for short critical sections with a modest read:write ratio, the
plain mutex often wins. Measure it, as always.

## C++20: `jthread`, `latch`, `barrier`

```cpp
#include <atomic>
#include <barrier>
#include <latch>
#include <thread>
#include <vector>

int main() {
    Registry reg;
    reg.set("alpha", 1);

    std::latch ready(3);                      // one-shot countdown
    std::atomic<int> reads{0};
    {
        std::vector<std::jthread> readers;    // jthread joins in its destructor
        for (int i = 0; i < 3; ++i)
            readers.emplace_back([&]{
                ready.count_down();
                ready.wait();                 // nobody proceeds until all 3 arrive
                for (int j = 0; j < 1000; ++j)
                    if (reg.get("alpha") == 1) reads.fetch_add(1);
            });
    }   // every ~jthread joins here -- no manual loop
    std::cout << "successful concurrent reads: " << reads.load() << "\n";

    std::atomic<int> phase{0};
    std::barrier sync(3, [&]{ phase.fetch_add(1); });   // reusable, with a callback
    {
        std::vector<std::jthread> workers;
        for (int i = 0; i < 3; ++i)
            workers.emplace_back([&]{
                for (int round = 0; round < 3; ++round)
                    sync.arrive_and_wait();
            });
    }
    std::cout << "barrier phases completed: " << phase.load() << "\n";
}
```

```text
successful concurrent reads: 3000
barrier phases completed: 3
```

All 3,000 reads succeeded while three threads hammered the registry
concurrently — that's `shared_lock` doing its job.

`std::jthread` fixes `std::thread`'s worst design flaw: a `std::thread` that is
destroyed while still joinable calls `std::terminate()` and kills the process.
`jthread` joins automatically, and also carries a `std::stop_token` for
cooperative cancellation:

```cpp
std::jthread worker([](std::stop_token st) {
    while (!st.stop_requested()) { doOneUnitOfWork(); }
});
// worker.request_stop() is called automatically by ~jthread, before the join
```

`std::latch` counts down once and is then spent — use it for "wait for N
initializations". `std::barrier` resets after each phase and optionally runs a
completion function between phases — use it for iterative parallel algorithms
where every thread must finish round *k* before any starts round *k+1*.

## Cheat sheet

| Tool | Std | Use for |
|------|-----|---------|
| `std::async(std::launch::async, f, …)` | 11 | Run `f` on another thread, get a `future` back |
| `future<T>::get()` | 11 | Block for the result; **rethrows** the task's exception |
| `std::promise<T>` | 11 | Hand a value across threads when the producer isn't a function call |
| `std::packaged_task<T()>` | 11 | Wrap a callable so a pool can return futures |
| `std::atomic<T>` | 11 | Race-free single-variable read-modify-write |
| `fetch_add` / `compare_exchange_weak` | 11 | Atomic increment; CAS loop for lock-free algorithms |
| `memory_order_relaxed` | 11 | Standalone counters — atomicity, no ordering |
| `memory_order_acquire/release` | 11 | Publish data behind a flag or pointer |
| `std::shared_mutex` + `shared_lock` | 17 | Read-heavy shared state |
| `std::scoped_lock(m1, m2)` | 17 | Lock several mutexes deadlock-free |
| `std::jthread` | 20 | A thread that joins itself and supports `stop_token` |
| `std::latch` | 20 | One-shot "wait for N arrivals" |
| `std::barrier` | 20 | Reusable phase synchronization with a completion callback |
| `std::atomic_ref<T>` | 20 | Atomic access to an object you don't own the declaration of |

## Traps

**Default `std::async` policy may not run in parallel.** `async|deferred` lets
the implementation defer the call to `.get()` on the *calling* thread. Always
write `std::launch::async` when you mean parallelism.

**A discarded `std::async` future blocks.** The temporary future returned by
`std::async(...)` with no variable to bind to is destroyed at the end of the
statement, and its destructor *blocks until the task completes*. A loop of
`std::async(f, i);` statements runs entirely serially. Store the futures.

**`std::thread` destroyed while joinable terminates the process.** Not an
exception — `std::terminate()`. Use `jthread`, or make joining exception-safe
with an RAII wrapper.

**Atomic does not mean transactional.** `if (counter.load() < max) counter.fetch_add(1);`
is a race: another thread can push it past `max` between the two operations.
Use `compare_exchange_weak` in a loop, or a mutex.

**`relaxed` on a publication flag is a real bug on ARM.** x86 has strong
hardware ordering that hides the mistake; the same code on AArch64 or POWER
reads uninitialized memory. Test on the architecture you ship, or use `seq_cst`.

**Locking two mutexes in different orders deadlocks.** Thread A takes `m1` then
`m2`, thread B takes `m2` then `m1`, and both stop forever. Use
`std::scoped_lock lk(m1, m2)`, which uses a deadlock-avoidance algorithm, or fix
a global lock ordering and document it.

**Thread sanitizer, not eyeballs.** Build with
`-fsanitize=thread -g` and run your tests. TSan finds the races that pass a
thousand times and fail in production; the increments-vanishing example above is
reported instantly with both stack traces.

## Exercise

Build a `ConcurrentCounter` map — `std::unordered_map<std::string, long>` behind
a `std::shared_mutex` — with `increment(key)`, `get(key) const`, and
`snapshot() const` returning a copy of the whole map. Then:

1. Spawn 8 `std::jthread`s that each increment 5 random keys out of 20, one
   million times total, and verify with a `std::latch` that all threads start
   only after every one is constructed.
2. Assert that the sum of all values in `snapshot()` equals the total number of
   increments.
3. Add a second implementation backed by `std::unordered_map<std::string,
   std::atomic<long>>` **pre-populated with all 20 keys** (so the map itself
   never mutates and needs no lock), and time both.
4. Build both with `-fsanitize=thread` and confirm clean. Then deliberately
   remove the `shared_lock` from `get()` and confirm TSan reports the race — the
   point is to see the tool catch something your test run did not.

Which implementation wins, and what does it cost you? (Hint: the atomic map
cannot add a key at runtime, and `snapshot()` on it is not a consistent
point-in-time view.)
