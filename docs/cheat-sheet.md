---
title: "C++ Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free C++ cheat sheet: 23 quick-reference tables from every level of the C++ Mastery Path, each linked to its full lesson."
---

# C++ Cheat Sheet

Quick-reference tables from **23 lessons** of the free C++ Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

| Concept | Syntax | What it buys you |
|---------|--------|------------------|
| Public inheritance | `class D : public B` | "D is-a B"; D usable wherever B is |
| Virtual function | `virtual void f();` | Runtime dispatch on the real object type |
| Override | `void f() override;` | Compiler-checked override |
| Pure virtual | `virtual void f() = 0;` | Abstract class / interface |
| Virtual destructor | `virtual ~B() = default;` | Safe `delete` through a base pointer |
| `final` | `class D final` / `void f() final` | Blocks further derivation; enables devirtualisation |
| `protected` | `protected: int x;` | Visible to derived classes, not to the outside |

### [02 · Templates Basics](level-2/02-templates-basics.md)

| Feature | Syntax |
|---------|--------|
| Function template | `template <typename T> T f(T a);` |
| Class template | `template <typename T> class C { ... };` |
| Multiple parameters | `template <typename T, typename U>` |
| Non-type parameter | `template <typename T, std::size_t N>` |
| Default argument | `template <typename T, typename C = std::less<T>>` |
| Explicit instantiation at call site | `f<double>(x)` |
| Full specialisation | `template <> T f<int>(int a) { ... }` |
| Where definitions go | **Header file**, always |

### [04 · STL Algorithms &amp; Iterators](level-2/04-stl-algorithms-iterators.md)

| Task | Algorithm | Header |
|------|-----------|--------|
| Sort | `sort`, `stable_sort`, `partial_sort` | `<algorithm>` |
| Find a value | `find`, `find_if`, `find_if_not` | `<algorithm>` |
| Fast find in sorted range | `binary_search`, `lower_bound`, `upper_bound` | `<algorithm>` |
| Count | `count`, `count_if` | `<algorithm>` |
| Test a whole range | `all_of`, `any_of`, `none_of` | `<algorithm>` |
| Map each element | `transform` | `<algorithm>` |
| Filter out elements | `remove_if` + `erase` | `<algorithm>` |
| Copy | `copy`, `copy_if`, `copy_n` | `<algorithm>` |
| Min / max | `min_element`, `max_element`, `minmax_element` | `<algorithm>` |
| Sum / fold | `accumulate`, `reduce` | `<numeric>` |
| Fill a range with a sequence | `iota` | `<numeric>` |
| Reverse / rotate / shuffle | `reverse`, `rotate`, `shuffle` | `<algorithm>` |
| Drop adjacent duplicates | `unique` + `erase` (sort first) | `<algorithm>` |

### [05 · Operator Overloading](level-2/05-operator-overloading.md)

| Operator | Typical signature | Returns |
|----------|-------------------|---------|
| `a + b` | `T operator+(const T&, const T&)` (free) | new value |
| `a += b` | `T& operator+=(const T&)` (member) | `*this` |
| `-a` | `T operator-() const` (member) | new value |
| `a == b` | `bool operator==(const T&, const T&)` (free) | `bool` |
| `a < b` | `bool operator<(const T&, const T&)` (free) | `bool` |
| `os << a` | `std::ostream& operator<<(std::ostream&, const T&)` (free) | the stream |
| `a[i]` | `V& operator[](level-2/std::size_t)` + `const` version (member) | reference |
| `a(x, y)` | `R operator()(X, Y)` (member) | anything |
| `++a` | `T& operator++()` (member) | `*this` |
| `a++` | `T operator++(int)` (member, dummy `int` param) | copy of the old value |

### [09 · Build Tools (CMake Basics)](level-2/09-build-tools-cmake.md)

| Task | Command |
|------|---------|
| Configure | `cmake -S . -B build` |
| Configure with a build type | `cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug` |
| Build everything | `cmake --build build` |
| Build in parallel | `cmake --build build -j 8` |
| Build one target | `cmake --build build --target calc_tests` |
| Run tests | `ctest --test-dir build --output-on-failure` |
| Clean | `cmake --build build --target clean` |
| Full reset | `rm -rf build` |
| Install | `cmake --install build --prefix /usr/local` |
| Use Ninja instead of Make | `cmake -S . -B build -G Ninja` |

| CMake command | Purpose |
|---------------|---------|
| `cmake_minimum_required()` | Set the policy/behaviour baseline (first line, always) |
| `project()` | Name, version, languages |
| `add_executable()` | Define a program target |
| `add_library()` | Define a static/shared/interface library target |
| `target_link_libraries()` | Link, and inherit usage requirements |
| `target_include_directories()` | Header search paths, with scope |
| `target_compile_options()` | Per-target compiler flags |
| `target_compile_definitions()` | Per-target `-D` macros |
| `find_package()` | Locate an installed dependency |
| `FetchContent_MakeAvailable()` | Download and build a dependency |
| `add_subdirectory()` | Include another directory's CMakeLists.txt |
| `add_test()` | Register a test with CTest |

## Level 3 · Advanced

### [01 · Advanced Templates](level-3/01-advanced-templates.md)

| Technique | Use when |
|-----------|----------|
| Full specialization (`template<> struct X<int>`) | One exact type needs different behaviour |
| Partial specialization (`template<typename T> struct X<T*>`) | A *shape* of types (pointers, pairs, containers) needs different behaviour |
| Variadic templates + recursion | Pre-C++17 codebases, or when each step needs distinct logic |
| Fold expressions `(args op ...)` | C++17+, a single operator applied across a whole pack |
| `std::enable_if` / SFINAE | Choosing between whole overloads based on a trait |
| `if constexpr` | Branching inside one function body based on a trait (prefer this over SFINAE when possible) |
| Non-type template parameter (`template<int N>`) | Baking a compile-time constant (size, flag) into the type itself |

### [02 · Move Semantics &amp; Rvalue References](level-3/02-move-semantics-rvalue-references.md)

| Syntax | Meaning |
|--------|---------|
| `T&` | lvalue reference — binds to named objects |
| `const T&` | binds to anything (lvalue or rvalue), read-only |
| `T&&` (fixed `T`) | rvalue reference — binds only to temporaries/`std::move`d values |
| `T&&` (`T` is a deduced template param) | forwarding reference — binds to either |
| `std::move(x)` | cast `x` to an rvalue reference; moves nothing itself |
| `std::forward<T>(x)` | inside a template, forward `x` preserving its original value category |

### [03 · Concurrency](level-3/03-concurrency.md)

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

### [04 · RAII Deep Dive](level-3/04-raii-deep-dive.md)

| Resource | RAII wrapper |
|----------|--------------|
| Heap memory | `std::unique_ptr` / `std::shared_ptr` |
| Mutex lock | `std::lock_guard` / `std::unique_lock` / `std::scoped_lock` |
| File handle | `std::fstream` family, or a custom wrapper around `FILE*` |
| One-off cleanup action | A small `ScopeGuard` with a cancellable lambda |
| A group of resources | Compose the above as data members (Rule of Zero) |

### [05 · Design Patterns in C++](level-3/05-design-patterns.md)

| Pattern | Problem it solves | Modern C++ idiom |
|---------|-------------------|------------------|
| Singleton | Exactly one instance, globally reachable | Function-local `static` (Meyer's Singleton) |
| Factory Method | Decouple construction from usage | Function returning `std::unique_ptr<Base>` |
| Observer | Broadcast events to unknown listeners | `std::vector<std::shared_ptr<Observer>>` + virtual callback |

### [06 · Networking Basics](level-3/06-networking-basics.md)

| Call | Role | Notes |
|------|------|-------|
| `socket()` | Create an endpoint | `AF_INET` + `SOCK_STREAM` = IPv4 TCP |
| `bind()` | Server: attach to a local port | Needed before `listen()` |
| `listen()` | Server: start accepting connections | Second arg is the pending-connection backlog |
| `accept()` | Server: block until a client connects | Returns a *new* fd for that client |
| `connect()` | Client: initiate a connection | Blocks until the handshake completes or fails |
| `send()`/`write()` | Send bytes | May send fewer bytes than requested — check the return value |
| `recv()`/`read()` | Receive bytes | Returns 0 on a graceful close, negative on error |
| `htons`/`htonl`, `ntohs`/`ntohl` | Byte-order conversion | Required for any multi-byte value on the wire |
| `close()` | Release the socket | Wrap in an RAII type, as shown above |

### [07 · Working with SQLite from C++](level-3/07-sqlite.md)

| Function | Purpose |
|----------|---------|
| `sqlite3_open` / `sqlite3_close` | Open/close a database file (creates it if missing) |
| `sqlite3_exec` | Run a SQL string with no bound parameters; optional row callback |
| `sqlite3_prepare_v2` | Compile SQL with `?` placeholders into a reusable statement |
| `sqlite3_bind_*` | Bind a value to a `?` placeholder by 1-based index |
| `sqlite3_step` | Execute, or advance to the next result row (`SQLITE_ROW` / `SQLITE_DONE`) |
| `sqlite3_column_*` | Read a column's value from the current row by 0-based index |
| `sqlite3_finalize` | Release a prepared statement (always required) |
| `sqlite3_errmsg` | Human-readable text for the last error on a connection |

### [08 · Testing with Google Test](level-3/08-testing-googletest.md)

| Macro / flag | Purpose |
|--------------|---------|
| `TEST(Suite, Name)` | Define a standalone test case |
| `TEST_F(Fixture, Name)` | Define a test using a fixture's members |
| `TEST_P` + `INSTANTIATE_TEST_SUITE_P` | Run one test body over many parameter values |
| `EXPECT_EQ` / `NE` / `LT` / `GT` | Compare two values; report both on failure, continue |
| `ASSERT_EQ` (and friends) | Same comparison, but abort the current test on failure |
| `EXPECT_TRUE` / `EXPECT_FALSE` | Assert a boolean condition |
| `EXPECT_STREQ` / `STRNE` | Compare C strings by **content**, not pointer |
| `EXPECT_NEAR(a, b, tol)` | Compare floating point within a tolerance |
| `EXPECT_THROW` / `NO_THROW` / `ANY_THROW` | Assert exception behaviour |
| `SetUp()` / `TearDown()` | Fixture hooks run before/after **each** test |
| `--gtest_filter=` | Run only matching tests |
| `--gtest_repeat=` / `--gtest_shuffle` | Rerun / randomize order to expose flaky tests |

### [09 · Performance Profiling](level-3/09-performance-profiling.md)

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

## Level 4 · Master

### [01 · Modern C++ Features (17/20)](level-4/01-modern-cpp-features.md)

| Feature | Std | Replaces |
|---------|-----|----------|
| `auto [a, b] = pair` | 17 | `.first` / `.second`, `std::get<N>` |
| `if (init; cond)` | 17 | A variable leaking past the `if` it belongs to |
| `std::optional<T>` | 17 | Sentinel values, out-param + `bool`, nullable pointers |
| `std::variant` + `std::visit` | 17 | `union` + manual tag, `void*`, base-class hierarchies for plain data |
| `if constexpr` | 17 | Tag dispatch, SFINAE overload pairs |
| `std::string_view` | 17 | `const std::string&` params that force a copy from a literal |
| `std::filesystem` | 17 | Platform-specific path and directory APIs |
| `[[nodiscard]]` | 17 | Comments asking callers not to ignore a result |
| Concepts / `requires` | 20 | SFINAE, `enable_if`, unreadable template errors |
| Ranges & views | 20 | `begin()/end()` pairs, eager intermediate containers |
| `operator<=>` | 20 | Six hand-written, mutually-inconsistent comparisons |
| `std::span<T>` | 20 | `(pointer, length)` parameter pairs |
| Designated initializers | 20 | Comment-annotated positional aggregate init |
| `constinit` / `consteval` | 20 | Runtime init of globals; "hopefully constexpr" functions |

### [02 · Advanced Concurrency](level-4/02-advanced-concurrency.md)

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

### [03 · Compile-time Computation](level-4/03-compile-time-computation.md)

| Construct | Std | Meaning |
|-----------|-----|---------|
| `constexpr` function | 11/14 | *May* be evaluated at compile time; still callable at runtime |
| `constexpr` variable | 11 | Value fixed at compile time; usable as an array size or `case` label |
| `consteval` function | 20 | **Must** be evaluated at compile time — runtime call is an error |
| `constinit` variable | 20 | Static-storage variable with guaranteed compile-time init (not const) |
| `static_assert(cond, "msg")` | 11/17 | Compile-time test; message optional since C++17 |
| `if constexpr (cond)` | 17 | Discard the untaken branch before instantiation |
| `(args + ... + init)` | 17 | Fold a parameter pack with any binary operator |
| `std::is_integral_v<T>` etc. | 11/17 | Type traits — the `_v` suffix gives the value directly |
| `std::conditional_t<B,T,F>` | 11/14 | Pick one of two **types** at compile time |
| `std::array` in `constexpr` | 17 | Compile-time computable fixed container |
| `std::vector`/`std::string` in `constexpr` | 20 | Allowed, but the allocation must not escape the constant evaluation |
| `std::is_constant_evaluated()` | 20 | Branch on "am I running at compile time right now?" |

### [04 · Systems Programming Patterns](level-4/04-systems-programming-patterns.md)

| Tool | Purpose |
|------|---------|
| `alignof(T)` / `alignas(N)` | Query / force alignment; `alignas(64)` kills false sharing |
| `offsetof(S, m)` | Byte offset of a member — check your padding |
| Descending-alignment member order | Free struct-size reduction |
| Struct-of-arrays | Make every cache line useful in the hot loop |
| `std::memcpy` / `std::bit_cast` | Type punning without breaking strict aliasing |
| `std::pmr::monotonic_buffer_resource` | Standard arena; pairs with `std::pmr::vector` |
| `operator new` / `delete` overloads | Per-class allocation strategy |
| `std::exchange(x, sentinel)` | The one-line move idiom for raw handles |
| PIMPL (`unique_ptr<Impl>`) | Break header dependencies, stabilize ABI |
| `std::popcount` / `bit_ceil` / `rotl` | Hardware bit operations, portably |
| `std::endian::native` | Compile-time byte-order check |
| `-fsanitize=address,undefined` | Catch the mistakes this module's techniques enable |

### [05 · Security Best Practices](level-4/05-security-best-practices.md)

| Practice | Tool / construct |
|----------|------------------|
| Find memory errors | `-fsanitize=address` (+ `,undefined`) in CI |
| Find data races | `-fsanitize=thread` |
| Find uninitialized reads | `-fsanitize=memory` (Clang) or Valgrind memcheck |
| Checked indexing | `.at()`, `_GLIBCXX_ASSERTIONS`, libc++ hardening |
| Pointer + length as one type | `std::span<T>`, `std::string_view` |
| Parse without silent failure | `std::from_chars`, `std::optional` return |
| Safe formatting | `std::format` (C++20), never `sprintf` |
| Subprocess without a shell | `posix_spawn` / `execve` with an argv array |
| Constant-time secret compare | `sodium_memcmp` or an XOR-accumulate loop |
| Cryptographic randomness | `getrandom` / `arc4random_buf` / `BCryptGenRandom` |
| Static analysis | `clang-tidy` (`cert-*`, `bugprone-*`, `cppcoreguidelines-*`) |
| Fuzzing | `-fsanitize=fuzzer` (libFuzzer) on every parser you own |
| Dependency CVEs | `cargo-audit`-equivalent: OSV-Scanner, Conan/vcpkg advisories |

### [06 · Large-Scale Build Systems](level-4/06-large-scale-build-systems.md)

| Command | Purpose |
|---------|---------|
| `target_include_directories(t PUBLIC …)` | Include paths that propagate to consumers |
| `target_link_libraries(t PRIVATE dep::dep)` | Link + inherit dep's public requirements |
| `target_compile_features(t PUBLIC cxx_std_20)` | Require a standard, propagate the requirement |
| `add_library(ns::name ALIAS name)` | Same target name in-tree and after install |
| `$<BUILD_INTERFACE:…>` / `$<INSTALL_INTERFACE:…>` | Different paths before and after install |
| `FetchContent_Declare/MakeAvailable` | Build a pinned dependency from source |
| `install(TARGETS … EXPORT …)` + `install(EXPORT …)` | Make `find_package` work downstream |
| `enable_testing()` + `gtest_discover_tests` | Register each test case with CTest |
| `ctest -j8 --output-on-failure` | Run the suite in parallel, show failures |
| `cmake --preset dev` | Reproducible, named configurations |
| `CMAKE_EXPORT_COMPILE_COMMANDS` | `compile_commands.json` for clangd / clang-tidy |
| `cmake --build build --target help` | List available targets |

### [07 · Cross-platform Development](level-4/07-cross-platform-development.md)

| Concern | Portable answer |
|---------|-----------------|
| Fixed-width integers | `<cstdint>`: `int32_t`, `uint64_t`, `uintptr_t` |
| Sizes and offsets | `std::size_t`, `std::ptrdiff_t` |
| `long` is 4 bytes on Windows | Never assume; use `int64_t` |
| `char` signedness varies | Cast to `unsigned char` before `<cctype>` functions |
| Byte order | `std::endian::native`; convert explicitly at I/O boundaries |
| Paths | `std::filesystem::path` and `operator/` |
| Temp / home directories | `fs::temp_directory_path()`, never `/tmp` |
| Feature availability | `<version>` + `__cpp_lib_*` macros |
| Compiler detection order | `__clang__` → `_MSC_VER` → `__GNUC__` |
| Threads | `<thread>` + `find_package(Threads)` / `Threads::Threads` |
| Windows macro pollution | `NOMINMAX`, `WIN32_LEAN_AND_MEAN` |
| MSVC conformance | `/permissive- /utf-8 /EHsc /W4` |
| Platform-specific code | Separate `.cpp` per platform, selected in CMake |
| Proof of portability | GitHub Actions matrix over 3 OSes × 2 build types |

### [08 · Testing at Scale &amp; CI](level-4/08-testing-at-scale-ci.md)

| Tool | Purpose |
|------|---------|
| `MOCK_METHOD(ret, name, (args), (const, override))` | Declare a mock method |
| `EXPECT_CALL(m, f(args)).WillOnce(Return(v))` | Expect one call, stub the result |
| `.Times(0)` / `.Times(AtLeast(n))` | Assert a call never / repeatedly happens |
| `NiceMock<T>` / `StrictMock<T>` | Silence uninteresting calls / make them failures |
| `TYPED_TEST_SUITE` | Run one test body over many types |
| `TEST_P` + `INSTANTIATE_TEST_SUITE_P` | Run one test body over many values |
| `-fprofile-instr-generate -fcoverage-mapping` | Clang coverage instrumentation |
| `llvm-cov report` / `show` | Line **and branch** coverage |
| `gtest_discover_tests(t PROPERTIES LABELS …)` | Register cases with CTest, labelled |
| `ctest -j16 -L unit --output-on-failure` | Parallel, filtered, verbose-on-failure |
| `ctest --repeat until-fail:50` | Flake hunting |
| `benchmark::DoNotOptimize` | Stop the optimizer deleting a benchmark |
| `-fsanitize=fuzzer,address` | libFuzzer + ASan on a parser |
| `clang-tidy` / `clang-format --dry-run --Werror` | Static analysis and style, in CI |

### [09 · Performance at Scale](level-4/09-performance-at-scale.md)

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

---

Go deeper with the full [C++ Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
