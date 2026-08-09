# 08 · Testing at Scale & CI

[Level 3 Module 8](../level-3/08-testing-googletest.md) covered writing a test.
This module covers running thousands of them, on every commit, on every
platform, fast enough that people don't route around the process.

The framing that keeps a large suite healthy is the **test pyramid**: many fast
unit tests, fewer integration tests, very few end-to-end tests. Inverted
pyramids — a handful of unit tests under a mountain of slow end-to-end runs —
produce suites that take 40 minutes, fail intermittently, and get skipped.

| Layer | Count | Runtime each | Scope |
|-------|-------|--------------|-------|
| Unit | thousands | < 1 ms | One class, dependencies mocked |
| Integration | hundreds | < 1 s | Several components, real DB/filesystem |
| End-to-end | tens | seconds | The whole binary, real I/O |

## Mocking with GoogleMock

A unit test needs the unit *isolated*. If your class calls the system clock and
a database, the test is neither fast nor deterministic. Depend on an interface,
and let the test supply a fake.

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>

class Clock {
public:
    virtual ~Clock() = default;
    virtual long nowSeconds() const = 0;
};

class Storage {
public:
    virtual ~Storage() = default;
    virtual bool save(const std::string& key, const std::string& value) = 0;
    virtual std::string load(const std::string& key) const = 0;
};

class SessionManager {                      // the unit under test
public:
    SessionManager(Clock& clock, Storage& storage, long ttl)
        : clock_(clock), storage_(storage), ttl_(ttl) {}

    bool create(const std::string& id) {
        return storage_.save(id, std::to_string(clock_.nowSeconds()));
    }
    bool isValid(const std::string& id) const {
        std::string created = storage_.load(id);
        if (created.empty()) return false;
        return clock_.nowSeconds() - std::stol(created) < ttl_;
    }
private:
    Clock& clock_;
    Storage& storage_;
    long ttl_;
};

class MockClock : public Clock {
public:
    MOCK_METHOD(long, nowSeconds, (), (const, override));
};
class MockStorage : public Storage {
public:
    MOCK_METHOD(bool, save, (const std::string&, const std::string&), (override));
    MOCK_METHOD(std::string, load, (const std::string&), (const, override));
};

using ::testing::_;
using ::testing::Return;

TEST(SessionManagerTest, CreateStoresCurrentTime) {
    MockClock clock; MockStorage storage;
    EXPECT_CALL(clock, nowSeconds()).WillOnce(Return(1000));
    EXPECT_CALL(storage, save("abc", "1000")).WillOnce(Return(true));

    SessionManager mgr(clock, storage, 300);
    EXPECT_TRUE(mgr.create("abc"));
}

TEST(SessionManagerTest, ExpiredSessionIsInvalid) {
    MockClock clock; MockStorage storage;
    EXPECT_CALL(storage, load("abc")).WillOnce(Return("1000"));
    EXPECT_CALL(clock, nowSeconds()).WillOnce(Return(1400));   // 400s later, ttl 300

    SessionManager mgr(clock, storage, 300);
    EXPECT_FALSE(mgr.isValid("abc"));
}

TEST(SessionManagerTest, UnknownSessionIsInvalid) {
    MockClock clock; MockStorage storage;
    EXPECT_CALL(storage, load(_)).WillOnce(Return(""));
    EXPECT_CALL(clock, nowSeconds()).Times(0);   // must NOT be consulted at all
    SessionManager mgr(clock, storage, 300);
    EXPECT_FALSE(mgr.isValid("nope"));
}
```

```text
[ RUN      ] SessionManagerTest.CreateStoresCurrentTime
[       OK ] SessionManagerTest.CreateStoresCurrentTime (0 ms)
[ RUN      ] SessionManagerTest.ExpiredSessionIsInvalid
[       OK ] SessionManagerTest.ExpiredSessionIsInvalid (0 ms)
[ RUN      ] SessionManagerTest.UnknownSessionIsInvalid
[       OK ] SessionManagerTest.UnknownSessionIsInvalid (0 ms)
[  PASSED  ] 3 tests.
```

Zero milliseconds, no clock, no database, and the "session expired 400 seconds
later" case is tested without waiting 400 seconds. `Times(0)` is the
underappreciated one — it asserts a *negative*: that the code short-circuits
before consulting the clock.

Break the expectation deliberately and the report is exact:

```text
Unexpected mock function call - returning default value.
    Function call: save(@0x16f8d2610 "abc", @0x16f8d2730 "1000")
          Returns: false
Google Mock tried the following 1 expectation, but it didn't match:

gm.cpp:58: EXPECT_CALL(storage, save("abc", "9999"))...
  Expected arg #1: is equal to "9999"
           Actual: "1000"
         Expected: to be called once
           Actual: never called - unsatisfied and active
```

Argument-by-argument, with the actual value. Matchers go far beyond equality:
`Eq`, `Ne`, `Gt`, `HasSubstr`, `StartsWith`, `ElementsAre`,
`UnorderedElementsAre`, `Pointee`, `Field(&S::x, Gt(3))`, `AllOf`, `Not`.

## Typed and parameterized tests

Testing one behaviour across many types, without copy-paste:

```cpp
template <typename T> class ContainerTest : public ::testing::Test {
protected:
    T container;
};
using ContainerTypes = ::testing::Types<std::vector<int>, std::deque<int>, std::list<int>>;
TYPED_TEST_SUITE(ContainerTest, ContainerTypes);

TYPED_TEST(ContainerTest, StartsEmptyThenGrows) {
    EXPECT_TRUE(this->container.empty());
    this->container.push_back(1);
    EXPECT_EQ(this->container.size(), 1u);
}
```

Note `this->container`: in a template base class, unqualified names aren't
looked up in the dependent base, so `container` alone does not compile. That
error confuses everyone exactly once.

## Coverage — and why the number lies

```bash
clang++ -O0 -fprofile-instr-generate -fcoverage-mapping tests.cpp lib.cpp -lgtest -lgtest_main -o t
LLVM_PROFILE_FILE=cov.profraw ./t
llvm-profdata merge -sparse cov.profraw -o cov.profdata
llvm-cov report ./t -instr-profile=cov.profdata lib.cpp
```

For a four-branch `classify()` function tested with only `3` and `4`:

```text
Filename    Regions  Missed  Cover   Lines  Missed  Cover   Branches  Missed  Cover
cov.cpp          10       2  80.00%      6       0 100.00%         6       2  66.67%
```

**100% line coverage, 66.67% branch coverage.** Every line executed; the
negative and zero cases never did — because each `if` fits on one line, and
executing the line does not mean taking both edges.

```text
    2|      2|int classify(int n) {
    3|      2|    if (n < 0) return -1;
    4|      2|    if (n == 0) return 0;
    5|      2|    if (n % 2 == 0) return 2;
    6|      1|    return 1;
```

That is the standing warning about coverage: it is a *floor*, useful for finding
completely untested code, and it says nothing about assertion quality. A test
that calls every function and asserts nothing scores 100%. Use it to find gaps;
never use it as a target with a ratchet. (GCC's equivalent is
`--coverage` plus `gcovr` or `lcov`.)

## Running the suite at scale with CTest

```cmake
include(GoogleTest)
gtest_discover_tests(unit_tests PROPERTIES LABELS "unit" TIMEOUT 10)
gtest_discover_tests(integration_tests PROPERTIES LABELS "integration" TIMEOUT 120)
```

```bash
ctest -j16 --output-on-failure        # parallel across test cases
ctest -L unit                         # only fast tests, for the pre-commit hook
ctest --rerun-failed --output-on-failure
ctest --repeat until-fail:50 -L unit  # hunt flakes
```

Labels plus timeouts are what let a 10,000-test suite stay usable: developers
run `-L unit` in seconds, CI runs everything in parallel, and a hung test fails
in 10 seconds instead of blocking the machine.

## Benchmarking as a test

Correctness regressions get caught; performance regressions ship. Google
Benchmark makes speed a checked property:

```cpp
#include <benchmark/benchmark.h>

static void BM_VectorPushBack(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> v;
        v.reserve(state.range(0));
        for (int i = 0; i < state.range(0); ++i) v.push_back(i);
        benchmark::DoNotOptimize(v.data());   // stop -O2 deleting the work
    }
    state.SetItemsProcessed(state.iterations() * state.range(0));
}
BENCHMARK(BM_VectorPushBack)->Range(8, 8 << 10);
```

`benchmark::DoNotOptimize` is the fix for the "impossible 0 ns" problem from
[Level 3 Module 9](../level-3/09-performance-profiling.md). Store results with
`--benchmark_out=` and compare across commits with `compare.py`; alert on a
regression beyond your noise floor (typically 5-10% on CI hardware).

## Fuzzing

Every parser that touches untrusted input deserves a fuzzer. It is about fifteen
lines:

```cpp
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    parseRecord(std::span<const uint8_t>(data, size));   // must not crash
    return 0;
}
```

```bash
clang++ -g -O1 -fsanitize=fuzzer,address,undefined fuzz_parser.cpp parser.cpp -o fuzz_parser
./fuzz_parser corpus/ -max_total_time=300
```

Commit the corpus, and commit every crashing input the fuzzer finds as a
regression test. A five-minute fuzz job per CI run finds inputs no human writes.

## A CI pipeline that earns its keep

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        include:
          - { os: ubuntu-latest,  preset: asan }
          - { os: ubuntu-latest,  preset: tsan }
          - { os: ubuntu-latest,  preset: coverage }
          - { os: macos-latest,   preset: release }
          - { os: windows-latest, preset: release }
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - run: cmake --preset ${{ matrix.preset }}
      - run: cmake --build --preset ${{ matrix.preset }}
      - run: ctest --preset ${{ matrix.preset }} --output-on-failure
```

The presets come from [Module 6](06-large-scale-build-systems.md), so the exact
same commands work locally. The essential jobs, in priority order:

1. **A sanitizer job** (ASan+UBSan). Highest bug-per-minute ratio of anything here.
2. **A second compiler.** GCC and Clang diagnose different things; MSVC diagnoses
   a third set.
3. **`-Werror`.** A warning nobody fixes is a warning nobody reads.
4. **`clang-tidy` and `clang-format --dry-run --Werror`** on the diff.
5. **TSan**, if you have threads.
6. **A short fuzz run** on each parser.

## Managing flakiness

A flaky test is worse than no test: it trains the team to re-run red builds.
Treat a flake as a **P1 bug in the test or the code**, not a nuisance.

- Find them: `ctest --repeat until-fail:100`, `--gtest_shuffle`,
  `--gtest_repeat=100`.
- The usual causes: real data races (run TSan), a dependency on wall-clock time
  (inject a `Clock`, as above), shared global state between tests, reliance on
  filesystem or map iteration order, and fixed ports or paths that collide under
  `ctest -j`.
- Quarantine with a label (`ctest -LE flaky`) only with a linked issue and a
  deadline. Permanent quarantine is deletion with extra steps.

## Cheat sheet

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

## Traps

**Mocking a concrete class you own.** If it isn't virtual, GoogleMock can't
override it. Either extract an interface or use a template parameter for the
dependency (compile-time polymorphism, zero runtime cost) — do not add
`virtual` purely for tests without accepting that it changes your design.

**`NiceMock` everywhere hides real bugs.** It silences warnings about calls you
didn't expect — including the call that shouldn't have happened. Default to a
plain mock; reach for `NiceMock` only when the noise is genuinely irrelevant.

**Testing the mock instead of the code.** `EXPECT_CALL(m, f()).WillOnce(Return(5));
EXPECT_EQ(m.f(), 5);` asserts that GoogleMock works. Assert on the *unit's*
observable behaviour.

**Chasing a coverage percentage.** Teams that mandate 90% get tests written to
touch lines. The example above scores 100% on lines while missing half the
branches. Measure branch coverage, and review assertions in code review.

**Coverage builds are `-O0`.** Never draw performance conclusions from them, and
keep coverage as a separate CI job from your benchmark job.

**`ASSERT_*` inside a helper function.** It returns from the *helper*; the test
continues. Same trap as in Level 3, and it gets worse as suites grow and
assertions get factored out. Use `EXPECT_*` in helpers, or wrap calls in
`ASSERT_NO_FATAL_FAILURE(...)`.

**Tests that share a fixed port, temp path, or database.** They pass serially
and fail under `ctest -j16`. Derive per-test unique names, or serialize them
with `RESOURCE_LOCK`.

**Sanitizer jobs that don't actually run the tests.** Building with
`-fsanitize=address` finds nothing on its own — ASan is a *runtime* tool. The
CI step that matters is `ctest`, not `cmake --build`.

## Exercise

Build a fully instrumented test pipeline for a small `RateLimiter` class —
`bool allow(const std::string& user)`, token-bucket, capacity *N*, refilling *R*
tokens per second, depending on an injected `Clock` interface.

1. Write unit tests with a `MockClock`: burst up to capacity, the (*N*+1)-th
   request denied, allowed again after simulated refill time, and per-user
   independence. Use `Times(AtLeast(1))` on the clock and at least one
   `Field`/`AllOf` matcher.
2. Add a `TYPED_TEST_SUITE` running the same behavioural contract against two
   storage backends (`std::map` and `std::unordered_map`).
3. Add a `TEST_P` sweep over capacities {1, 5, 100} and refill rates {1, 10}.
4. Measure coverage with `llvm-cov`. Report **line and branch** coverage
   separately, find one uncovered branch, and add the test that covers it.
5. Add a Google Benchmark for `allow()` under 10,000 distinct users, using
   `DoNotOptimize`. Record the baseline number.
6. Wire a CI matrix with `asan`, `tsan` and `coverage` presets, label the tests
   `unit`, and confirm `ctest -j8 -L unit --repeat until-fail:20` is green
   twenty times running.

Step 6 is the acceptance criterion. A suite that passes once is a
demonstration; a suite that passes twenty times under TSan is a test suite.
