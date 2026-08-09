# 09 · Performance Profiling

C++ is chosen for speed, and that makes it uniquely easy to waste. A program
can be a hundred times slower than necessary while looking perfectly idiomatic,
because the expensive part — a copy, an allocation, a cache miss — is invisible
in the source. Profiling is the discipline of *measuring* where the time goes
instead of guessing, and it is a discipline precisely because the guess is
almost always wrong.

The rule that governs everything below: **measure, change one thing, measure
again.** An optimization you didn't verify is a refactor with extra risk.

## Timing with `<chrono>`

The portable, always-available tool is `std::chrono::steady_clock`. Use
`steady_clock`, not `system_clock` — `system_clock` tracks wall-clock time and
can jump backwards when NTP adjusts the machine's clock, which turns a duration
into a negative number.

A scope-based timer is the RAII idea from [Module 4](04-raii-deep-dive.md)
applied to measurement: construct it at the top of a block, and the destructor
reports how long the block took.

```cpp
#include <chrono>
#include <iostream>
#include <numeric>
#include <string>
#include <vector>

class Timer {
public:
    explicit Timer(std::string label)
        : label_(std::move(label)), start_(std::chrono::steady_clock::now()) {}
    ~Timer() {
        auto elapsed = std::chrono::steady_clock::now() - start_;
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count();
        std::cout << label_ << ": " << us << " us\n";
    }
private:
    std::string label_;
    std::chrono::steady_clock::time_point start_;
};

long long sumByValue(std::vector<int> v) {          // takes a COPY
    return std::accumulate(v.begin(), v.end(), 0LL);
}
long long sumByRef(const std::vector<int>& v) {     // no copy
    return std::accumulate(v.begin(), v.end(), 0LL);
}

int main() {
    std::vector<int> data(1'000'000);
    std::iota(data.begin(), data.end(), 1);

    long long a = 0, b = 0;
    {
        Timer t("sumByValue (copies 4 MB)");
        for (int i = 0; i < 20; ++i) a += sumByValue(data);
    }
    {
        Timer t("sumByRef   (no copy)   ");
        for (int i = 0; i < 20; ++i) b += sumByRef(data);
    }
    std::cout << "checksum " << (a == b ? "match" : "MISMATCH") << "\n";
}
```

```text
sumByValue (copies 4 MB): 3181 us
sumByRef   (no copy)   : 1014 us
checksum match
```

Both functions compute the same answer. The only difference is a missing `&`,
and it costs three times the runtime — 20 million elements of copying that the
source code never mentions. This is the single most common performance bug in
real C++ code, and it is why Level 3 spent a whole module on
[move semantics](02-move-semantics-rvalue-references.md).

## Always benchmark an optimized build

```bash
g++ -std=c++17 -O0 bench.cpp -o bench   # unoptimized: not representative
g++ -std=c++17 -O2 bench.cpp -o bench   # what you should measure
```

At `-O0` the compiler inlines nothing and keeps every temporary. Timings from a
debug build routinely rank two implementations in the *opposite* order from the
release build, so a "-O0 optimization" can be a release-build pessimization.
Profile what you ship.

## Comparing data structures, not opinions

"`unordered_map` is faster than `map`" is folklore until you measure it on your
key type and your access pattern. A small harness that times a lambda makes
comparisons cheap to write:

```cpp
#include <chrono>
#include <iostream>
#include <map>
#include <string>
#include <unordered_map>
#include <vector>

using Clock = std::chrono::steady_clock;

template <typename F>
long long timeUs(F&& f) {
    auto t0 = Clock::now();
    f();
    return std::chrono::duration_cast<std::chrono::microseconds>(
               Clock::now() - t0).count();
}

int main() {
    const int N = 200000;
    std::vector<std::string> keys;
    keys.reserve(N);
    for (int i = 0; i < N; ++i) keys.push_back("key" + std::to_string(i));

    std::map<std::string, int> ordered;
    std::unordered_map<std::string, int> hashed;

    std::cout << "insert  std::map           " << timeUs([&]{
        for (int i = 0; i < N; ++i) ordered[keys[i]] = i;
    }) << " us\n";
    std::cout << "insert  std::unordered_map " << timeUs([&]{
        for (int i = 0; i < N; ++i) hashed[keys[i]] = i;
    }) << " us\n";

    long long s1 = 0, s2 = 0;
    std::cout << "lookup  std::map           " << timeUs([&]{
        for (int i = 0; i < N; ++i) s1 += ordered[keys[i]];
    }) << " us\n";
    std::cout << "lookup  std::unordered_map " << timeUs([&]{
        for (int i = 0; i < N; ++i) s2 += hashed[keys[i]];
    }) << " us\n";

    std::cout << "sums equal: " << std::boolalpha << (s1 == s2) << "\n";
}
```

```text
insert  std::map           56024 us
insert  std::unordered_map 13470 us
lookup  std::map           23943 us
lookup  std::unordered_map 7890 us
sums equal: true
```

Roughly 3x on both operations, on this machine, for these keys. Note the two
caveats that the numbers force you to state: *on this machine* and *for these
keys*. `std::map` still wins when you need sorted iteration or `lower_bound`
range queries — a benchmark answers "which is faster here", never "which is
better".

## Finding the real hot spot

Micro-timing individual candidates only works once you know *what* to time. The
usual failure is optimizing a function that accounts for 2% of runtime. Two
approaches:

**Instrument coarsely first.** Drop a `Timer` around each major phase (parse,
compute, write). One phase will dominate. Recurse into that phase only.

**Use a sampling profiler.** On macOS, `xctrace` and Instruments' Time Profiler
template attach to a running process; on Linux, `perf` is the standard:

```bash
perf record -g ./myprogram          # sample the call stack ~1000x/sec
perf report                         # interactive, hottest functions first

valgrind --tool=callgrind ./myprogram   # exact instruction counts, ~50x slower
```

Compile with `-O2 -g` for profiling: `-O2` so the measurement reflects reality,
`-g` so the profiler can map addresses back to function and line names.

## What optimization actually looks like

The wins that matter are almost never clever arithmetic. They are *removing
work*: fewer allocations, fewer copies, better memory locality.

```cpp
std::string joinSlow(const std::vector<std::string>& parts) {
    std::string out;
    for (const auto& p : parts) out = out + p + ",";   // builds a NEW string each step
    return out;
}

std::string joinFast(const std::vector<std::string>& parts) {
    std::size_t total = 0;
    for (const auto& p : parts) total += p.size() + 1;
    std::string out;
    out.reserve(total);                                 // one allocation, up front
    for (const auto& p : parts) { out += p; out += ','; }
    return out;
}
```

```text
joinSlow 609 ms
joinFast 0 ms
identical: true, length 660000
```

Same output, same 660,000 characters. `joinSlow` is quadratic: `out + p` builds
a fresh temporary containing everything accumulated so far, so joining *n*
pieces copies roughly *n²/2* characters. `joinFast` reserves once and appends in
place. This is the shape of a genuine optimization — an algorithmic change, not
a micro-tweak — and it is why `reserve()` on `std::vector` and `std::string`
belongs in your reflexes whenever the final size is known or estimable.

## Cheat sheet

| Tool / technique | Use it for |
|------------------|-----------|
| `std::chrono::steady_clock` | Portable monotonic timing; never `system_clock` for durations |
| `duration_cast<microseconds>` | Convert a duration to a printable integer count |
| RAII `Timer` | Time a scope automatically, exception-safe |
| `-O2 -g` | The build to profile: real codegen, readable symbols |
| `perf record -g` / `perf report` | Linux sampling profiler — find the hot function |
| Instruments Time Profiler | macOS equivalent, GUI call-tree |
| `valgrind --tool=callgrind` | Exact instruction counts (slow, deterministic) |
| `valgrind --tool=massif` | Heap allocation profile over time |
| `-fsanitize=address` | Catch the memory bugs that "fast" rewrites introduce |
| `reserve()` | Kill repeated reallocation in `vector` / `string` growth |
| Pass by `const&` | Kill silent copies of containers at call boundaries |

## Traps

**Benchmarking a debug build.** `-O0` timings measure the compiler's laziness,
not your code. Always measure at the optimization level you ship.

**The compiler deletes your benchmark.** If a result is never used, `-O2` may
remove the entire computation and report an impossible 0 ns. Accumulate results
into a variable you print — every example above sums into a checksum and prints
it for exactly this reason.

**Timing one run of a fast function.** A microsecond-scale operation measured
once is dominated by clock overhead and scheduler noise. Loop it thousands of
times, divide, and run the whole benchmark a few times to see the spread.

**Optimizing before profiling.** Rewriting a function that consumes 2% of
runtime caps your total possible win at 2%, while adding bugs at full price.
Find the dominant cost first.

**`system_clock` for durations.** It can step backwards or jump forwards when
the system clock is corrected, producing negative or wildly inflated elapsed
times. Use `steady_clock`.

**Assuming asymptotics beat constants at every size.** For a handful of
elements, a linear scan over a contiguous `std::vector` usually beats a
`std::unordered_map` lookup — the vector fits in cache and the hash does not.
The crossover is real and only measurement locates it.

## Exercise

Take the `joinSlow`/`joinFast` pair and extend the comparison. Add a third
implementation, `joinStream`, that uses `std::ostringstream` and returns
`oss.str()`, and a fourth that appends to a `std::vector<char>` before
constructing the string. Time all four at 1,000 / 10,000 / 100,000 parts,
printing a small table of sizes versus milliseconds, and verify with an
`assert` that all four produce identical output. Then answer, from your own
numbers: at which size does `joinSlow`'s quadratic behaviour first become
obvious, and does the ranking of the other three change with size? Finally,
rebuild at `-O0` and confirm for yourself that the numbers — and possibly the
ranking — are different.
