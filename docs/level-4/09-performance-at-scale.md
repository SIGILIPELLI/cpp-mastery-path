# 09 · Performance at Scale

[Level 3 Module 9](../level-3/09-performance-profiling.md) covered how to
measure. This module covers what the measurements usually turn out to be about.

At master level the performance question stops being "which algorithm" and
becomes "how does this code interact with the machine". A modern CPU executes
several instructions per cycle and stalls for hundreds of cycles on a main-memory
access. That ratio — roughly 200:1 — means **most optimization at scale is
memory optimization**, and the biggest wins come from changing where data lives
and how it is walked, not from clever arithmetic.

## The number that explains most of it

| Access | Latency | In "instructions you could have run" |
|--------|---------|--------------------------------------|
| Register | 0 cycles | — |
| L1 cache | ~4 cycles | ~10 |
| L2 cache | ~12 cycles | ~40 |
| L3 cache | ~40 cycles | ~150 |
| Main memory | ~200+ cycles | ~800 |
| SSD read | ~100,000 cycles | ~400,000 |

A cache line is 64 bytes on x86-64 and 64-128 on ARM. The CPU never fetches one
value; it fetches the line. Every performance property below follows from those
two facts.

## Stride: the same data, walked two ways

```cpp
const int N = 4096;
std::vector<float> m(std::size_t(N) * N, 1.0f);   // 64 MB, row-major

// stride 1: consecutive addresses
for (int r = 0; r < N; ++r) for (int c = 0; c < N; ++c) a += m[r*N + c];

// stride N: 16 KB apart -- a new cache line every single access
for (int c = 0; c < N; ++c) for (int r = 0; r < N; ++r) b += m[r*N + c];
```

```text
row-major traversal (stride 1)   : 32928 us
column-major traversal (stride N): 77836 us
same total: true
```

Identical additions, identical result, **2.4x** apart. Swapping two loop lines is
the entire fix. On a matrix that doesn't fit in L3 at all, the gap widens to 5-10x.

The general form: **the innermost loop should walk the fastest-varying index of
your layout.** For genuinely large matrices, go further and process in tiles that
fit in L2, so each loaded block is reused before eviction.

## Pointer chasing

```cpp
std::vector<int> vec(4'000'000);
std::list<int>   lst(vec.begin(), vec.end());
for (int x : vec) s1 += x;
for (int x : lst) s2 += x;
```

```text
std::vector traversal: 492 us
std::list   traversal: 15048 us
```

**30x.** Both are O(n) with the same number of additions. The vector is one
contiguous run the prefetcher can stream ahead of; the list is four million
independent heap nodes, each requiring the previous load to *complete* before the
next address is even known. That dependency chain is unhidable.

This is why `std::vector` is the correct default container and `std::list` almost
never is. Even for middle insertions, a vector's `memmove` usually beats a list's
allocation and cache misses until the element count is very large. Measure the
crossover on your data; it is much further out than intuition suggests.

## False sharing

Two threads writing to *different* variables that share a cache line force that
line to bounce between cores. No data race, no incorrect result — just the
coherence protocol destroying your scaling.

```cpp
struct Shared { std::atomic<long> v{0}; };                 // 8 bytes: 8 per line
struct Padded { alignas(128) std::atomic<long> v{0}; };    // one per line

// 8 threads, each hammering ITS OWN counter, 20M increments
std::vector<Counter> counters(threads);
counters[i].v.fetch_add(1, std::memory_order_relaxed);
```

```text
sizeof(Shared)=8  sizeof(Padded)=128
false sharing (packed): 1618 ms
padded to cache line  : 60 ms
```

**27x from one `alignas`.** Each thread touches only its own counter; they are
logically independent. But eight 8-byte counters land in one 64-byte line, so
every increment on any core invalidates that line on all seven others.

C++17 names the constant for you: `std::hardware_destructive_interference_size`.
The rule of thumb — pad per-thread mutable state to a cache line, and prefer
thread-local accumulation with a single merge at the end over shared counters
entirely.

## Branch misprediction

A modern CPU speculates 15-20 instructions ahead. A mispredicted branch flushes
that pipeline: 15-20 cycles wasted. In a tight loop over unpredictable data, that
dominates.

The mitigations, in order of preference:

1. **Sort or partition the data** so branches become predictable.
2. **Make the branch branchless**: `sum += x * (x >= threshold)` or
   `std::max(a, b)` compile to conditional moves.
3. **Hoist the branch out of the loop** — two specialized loops beat one loop
   with an invariant `if` inside it.
4. `[[likely]]` / `[[unlikely]]` attributes (C++20) as a last resort, only where
   profiling proved the distribution.

Be warned that this is the area where benchmarks lie most readily: at `-O2` the
vectorizer frequently converts the branchy loop into branchless SIMD by itself,
and the classic "sorted array is faster" demonstration then shows no difference
at all. Always check the generated assembly (`-S -masm=intel`, or Compiler
Explorer) before concluding you outsmarted the compiler.

## Allocation

`new` is a synchronized, general-purpose function that may take hundreds of
nanoseconds. At scale, allocation count is a first-class metric.

- **`reserve()` before a known-size fill.** One allocation instead of *log n*
  reallocations plus *n* moves.
- **Reuse buffers across iterations** instead of constructing a fresh
  `std::vector` per loop pass — hoist it out and `clear()` it (which keeps
  capacity).
- **Small-buffer optimization**: `std::string` stores ~15-22 characters inline
  with no allocation at all. Staying under that threshold is free.
- **Arenas / `std::pmr`** for request-scoped or frame-scoped objects, as covered
  in [Module 4](04-systems-programming-patterns.md).
- **`emplace_back` over `push_back`** when constructing in place, to skip a move.
- **`shrink_to_fit` is a request**, not a command, and it reallocates. Rarely
  worth it.

Track allocations directly — Valgrind's massif, `heaptrack`, or simply
overloading global `operator new` in a debug build to count calls. A 30% speedup
from removing allocations in a hot path is routine.

## Parallelism, in increasing order of effort

**Parallel STL (C++17)** is the cheapest parallelism in the language:

```cpp
#include <execution>
std::sort(std::execution::par_unseq, v.begin(), v.end());
std::transform(std::execution::par, in.begin(), in.end(), out.begin(), f);
double total = std::reduce(std::execution::par, v.begin(), v.end(), 0.0);
```

`seq` / `par` / `par_unseq` / `unseq`. Note that `std::reduce` replaces
`std::accumulate` here — `accumulate` is specified as strictly sequential
left-fold, so it cannot be parallelized, and `reduce` requires your operation to
be associative and commutative (which is why floating-point `reduce` may give
slightly different results run to run). Support varies: libstdc++ needs Intel
TBB linked, libc++ has been slower to ship it.

**Vectorization (SIMD)** is mostly the compiler's job. Help it by using
contiguous data, avoiding loop-carried dependencies and aliasing, and building
with `-O3 -march=native` where you control the deployment target. Check what
happened with `-Rpass=loop-vectorize` (Clang) or
`-fopt-info-vec-missed` (GCC) instead of guessing.

**Threads** for coarse-grained work, via the pool from
[Level 3 Module 10](../level-3/10-project-task-processor.md). Amdahl's law is the
governing constraint: if 10% of the work is serial, infinite cores give at most
10x. Measure the serial fraction before adding threads.

## Whole-program compiler optimizations

| Flag | Effect | Typical gain |
|------|--------|--------------|
| `-O2` | The sane default | baseline |
| `-O3` | More aggressive inlining/unrolling | 0-10%, sometimes negative |
| `-march=native` | Use this CPU's instruction set | 5-20% on numeric code |
| `-flto` | Link-time optimization: inline across TUs | 5-15% |
| PGO (`-fprofile-generate` → run → `-fprofile-use`) | Lay out code by real branch frequencies | 10-20% |
| BOLT / propeller | Post-link binary layout | 5-15% on large binaries |

PGO is the most reliably underused of these. The workflow is three commands, and
it gives the optimizer the one thing static analysis cannot have — knowledge of
which branches actually run. `-march=native` is the dangerous one: the binary
will `SIGILL` on any older CPU, so use explicit levels (`-march=x86-64-v2`) for
distributed builds.

## Lock contention

An uncontended mutex is ~20 ns. A contended one is a syscall, a context switch,
and a scheduler round trip. Fixes, in order:

1. **Shrink the critical section** — do the computation outside the lock, hold it
   only for the update.
2. **Shard** — 64 mutexes indexed by `hash(key) % 64` instead of one.
3. **`std::shared_mutex`** when reads dominate ([Module 2](02-advanced-concurrency.md)).
4. **Thread-local accumulation** with a periodic merge — no lock in the hot path.
5. **Lock-free structures** only when 1-4 are exhausted; they are extremely hard
   to get right and often slower under low contention.

## Cheat sheet

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Slow loop over a big array | Wrong traversal order | Match the loop order to memory layout |
| Slow container traversal | Pointer chasing | `vector` instead of `list`/`map` |
| Parallel code doesn't scale | False sharing | `alignas(64)`, per-thread state |
| Parallel code scales to a point | Amdahl / lock contention | Shard locks, shrink critical sections |
| Time in `malloc` | Allocation churn | `reserve`, reuse buffers, arenas / `pmr` |
| High branch-miss rate | Unpredictable data | Sort, partition, or go branchless |
| No SIMD in the hot loop | Aliasing / dependencies | Contiguous data, `-Rpass=loop-vectorize` |
| Fast micro, slow macro | Optimized the wrong 2% | Profile first, always |

| Tool | Use |
|------|-----|
| `perf stat -e cache-misses,branch-misses` | Hardware counters — confirm the *cause* |
| `perf record -g` / Instruments | Where the time goes |
| `heaptrack` / `massif` | Where the allocations go |
| Google Benchmark + `DoNotOptimize` | Regression-checked microbenchmarks |
| Compiler Explorer / `-S` | What the compiler actually emitted |
| `-flto`, PGO, `-march=` | Free-ish whole-program wins |

## Traps

**Optimizing without a profile.** Still the number one mistake, and it gets worse
at scale because large systems have less intuitive hot spots, not more.

**Microbenchmarks that don't reflect production.** A benchmark with data hot in
L1 measures something your production code, whose data is cold, will never
experience. Size the working set realistically.

**`-O3` assumed better than `-O2`.** More inlining means a bigger binary, more
instruction-cache pressure, and occasionally slower code. Measure both.

**`-march=native` in a shipped artifact.** Illegal-instruction crashes on any
machine older than your build host.

**Parallelizing before removing the serial waste.** Four threads doing
unnecessary allocations is still slower than one thread doing none. Fix the
single-threaded profile first — it also reduces the work you then have to
coordinate.

**`std::accumulate` with `std::execution::par`.** It compiles and silently stays
sequential; the parallel spelling is `std::reduce`.

**Assuming the vectorizer is on.** Aliasing (two pointers the compiler cannot
prove are distinct), loop-carried dependencies, and `volatile` all disable it.
Ask the compiler with `-fopt-info-vec-missed` rather than assuming.

**Padding everything to a cache line.** False-sharing padding costs memory and
cache capacity. Apply it to per-thread mutable state only, not to every struct.

## Exercise

Build a parallel word-frequency counter and drive it from 1x to as close to
*N*x as you can, documenting every step with numbers.

1. **Baseline**: read a large text file (generate ~500 MB if you don't have
   one), split on whitespace, count into a `std::map<std::string, int>`. Time it.
2. **Container**: switch to `std::unordered_map` with `reserve()`. Report the
   ratio and explain which of the two changes mattered more.
3. **Allocation**: replace `std::string` keys built per word with
   `std::string_view` into the file buffer (read the whole file once). Count
   allocations before and after by overloading global `operator new`.
4. **Parallel**: split the buffer into *T* chunks at whitespace boundaries, count
   each in its **own** map on its own thread, then merge. Plot speedup for
   T = 1, 2, 4, 8, 16 and identify where it stops scaling.
5. **False sharing**: deliberately make all threads share one padded-vs-unpadded
   `std::atomic<long>` total-word counter, and measure both. Confirm you can
   reproduce a gap like the 27x above.
6. **Compiler**: rebuild the best version with `-flto`, then with PGO
   (`-fprofile-generate`, run on a sample, `-fprofile-use`). Report both deltas.
7. **Confirm the cause, not just the effect**: run
   `perf stat -e cache-misses,branch-misses,instructions` on the baseline and the
   final version and show that the counters moved in the direction your
   explanation predicts.

Step 7 is what separates performance engineering from guessing that happened to
work.
