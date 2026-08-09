# 08 · Testing with Google Test

Up to now, "does it work?" has been answered by running the program and
reading the output by eye. That stops scaling the moment a codebase has more
than a handful of behaviours — you can't manually re-check every one of them
after every change. A unit test framework turns each expectation into code
that runs automatically and reports pass/fail. **Google Test** (gtest) is the
de-facto standard for C++: it's a library you link against, a few macros, and
a test runner binary.

## Installing and building

On macOS, `brew install googletest` installs the headers and static
libraries; on Debian/Ubuntu it's `apt install libgtest-dev`. A test binary is
compiled like any other program, plus the include path, the library path, and
two libraries — `gtest` (the framework) and `gtest_main` (a ready-made
`main()` that runs every test it finds, so you don't write one):

```bash
g++ -std=c++17 test_math.cpp \
    -I/opt/homebrew/opt/googletest/include \
    -L/opt/homebrew/opt/googletest/lib \
    -lgtest -lgtest_main -pthread \
    -o test_math
```

`-pthread` is required because gtest uses threads internally, even if your
code doesn't.

## Your first test

```cpp
#include <gtest/gtest.h>

int add(int a, int b) { return a + b; }

TEST(AddTest, HandlesPositiveNumbers) {
    EXPECT_EQ(add(2, 3), 5);
}

TEST(AddTest, HandlesNegativeNumbers) {
    EXPECT_EQ(add(-2, -3), -5);
    EXPECT_EQ(add(-2, 3), 1);
}
```

`TEST(SuiteName, TestName)` defines one test. The first argument groups
related tests into a **suite**; the second names the individual case. There is
no registration list to maintain — the macro registers the test with the
framework at static-initialization time, and `gtest_main` runs everything
registered. Running the binary:

```text
Running main() from .../googletest/src/gtest_main.cc
[==========] Running 2 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 2 tests from AddTest
[ RUN      ] AddTest.HandlesPositiveNumbers
[       OK ] AddTest.HandlesPositiveNumbers (0 ms)
[ RUN      ] AddTest.HandlesNegativeNumbers
[       OK ] AddTest.HandlesNegativeNumbers (0 ms)
[----------] 2 tests from AddTest (0 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 2 tests.
```

## What a failure looks like

Change `add` to return `a - b` and rerun:

```text
[ RUN      ] AddTest.HandlesPositiveNumbers
test_math.cpp:6: Failure
Expected equality of these values:
  add(2, 3)
    Which is: -1
  5

[  FAILED  ] AddTest.HandlesPositiveNumbers (0 ms)
...
[  PASSED  ] 0 tests.
[  FAILED  ] 1 test, listed below:
[  FAILED  ] AddTest.HandlesPositiveNumbers

 1 FAILED TEST
```

Note what the report contains: the file and line, the *expression* that
failed, and the actual value it produced. That's why `EXPECT_EQ(a, b)` is
better than `EXPECT_TRUE(a == b)` — the latter can only print "false", while
the former prints both operands. Also note the process **exit code is 1** on
failure and 0 on success, which is what makes a test binary usable from a
`make` target or a CI pipeline.

## `EXPECT_*` vs `ASSERT_*`

Every assertion comes in two flavours. `EXPECT_EQ` records a failure and
**keeps going**; `ASSERT_EQ` records the failure and **returns from the test
function immediately**.

```cpp
TEST(VectorTest, SecondElementIsTwo) {
    std::vector<int> v = makeVector();

    ASSERT_EQ(v.size(), 2u);   // if this fails, stop -- v[1] below would be UB
    EXPECT_EQ(v[1], 2);        // only reached when the size check passed
}
```

Use `ASSERT_*` when continuing would crash or produce meaningless cascading
failures (null pointers, out-of-range indexing); use `EXPECT_*` everywhere
else, so one run reports every problem instead of only the first.

## Fixtures — shared setup with `TEST_F`

When several tests need the same starting state, put it in a fixture class
derived from `::testing::Test`:

```cpp
#include <gtest/gtest.h>
#include <stdexcept>
#include <string>
#include <vector>

class Inventory {
public:
    void add(const std::string& item, int qty) {
        if (qty <= 0) throw std::invalid_argument("quantity must be positive");
        items_.push_back({item, qty});
    }
    int total() const {
        int sum = 0;
        for (const auto& p : items_) sum += p.second;
        return sum;
    }
    std::size_t size() const { return items_.size(); }
private:
    std::vector<std::pair<std::string, int>> items_;
};

class InventoryTest : public ::testing::Test {
protected:
    void SetUp() override {          // runs before EACH test in this fixture
        inv.add("bolts", 10);
        inv.add("nuts", 5);
    }
    Inventory inv;                    // members are accessible in every TEST_F body
};

TEST_F(InventoryTest, StartsWithTwoItems)  { EXPECT_EQ(inv.size(), 2u); }
TEST_F(InventoryTest, TotalsQuantities)    { EXPECT_EQ(inv.total(), 15); }

TEST_F(InventoryTest, AddingAffectsOnlyThisTest) {
    inv.add("washers", 100);
    EXPECT_EQ(inv.total(), 115);
}

TEST_F(InventoryTest, RejectsNonPositiveQuantity) {
    EXPECT_THROW(inv.add("screws", 0), std::invalid_argument);
    EXPECT_NO_THROW(inv.add("screws", 1));
}
// [       OK ] InventoryTest.StartsWithTwoItems (0 ms)
// [       OK ] InventoryTest.TotalsQuantities (0 ms)
// [       OK ] InventoryTest.AddingAffectsOnlyThisTest (0 ms)
// [       OK ] InventoryTest.RejectsNonPositiveQuantity (0 ms)
// [  PASSED  ] 4 tests.
```

The critical detail: gtest constructs a **brand new fixture object for every
single test**, calls `SetUp()`, runs the body, then calls `TearDown()` and
destroys it. `AddingAffectsOnlyThisTest` pushes the total to 115, yet
`TotalsQuantities` still sees 15 — tests cannot contaminate each other
through the fixture, and they can run in any order.

`EXPECT_THROW(statement, ExceptionType)` passes only if that exact type (or a
derived type) is thrown; `EXPECT_NO_THROW` passes only if nothing is.

## Parameterized tests

Running the same test body over many inputs doesn't need a copy-paste loop:

```cpp
#include <gtest/gtest.h>

bool isEven(int n) { return n % 2 == 0; }

class EvenTest : public ::testing::TestWithParam<int> {};

TEST_P(EvenTest, DetectsEvenNumbers) {
    EXPECT_TRUE(isEven(GetParam()));
}

INSTANTIATE_TEST_SUITE_P(SmallEvens, EvenTest, ::testing::Values(0, 2, 4, 100));
// [       OK ] SmallEvens/EvenTest.DetectsEvenNumbers/0 (0 ms)
// [       OK ] SmallEvens/EvenTest.DetectsEvenNumbers/1 (0 ms)
// [       OK ] SmallEvens/EvenTest.DetectsEvenNumbers/2 (0 ms)
// [       OK ] SmallEvens/EvenTest.DetectsEvenNumbers/3 (0 ms)
// [  PASSED  ] 4 tests.
```

Each value becomes a **separate reported test**, so a failure on input 100
names the case rather than hiding inside a loop that stopped at the first
bad value.

## Running a subset

A test binary accepts flags. `--gtest_filter` takes a glob over
`Suite.Test` names:

```bash
./test_inventory --gtest_filter='InventoryTest.Total*'
# Note: Google Test filter = InventoryTest.Total*
# [       OK ] InventoryTest.TotalsQuantities (0 ms)
# [  PASSED  ] 1 test.
```

Other useful ones: `--gtest_list_tests` (print names without running),
`--gtest_repeat=100` (rerun repeatedly — invaluable for flushing out the
intermittent concurrency bugs from [Module 3](03-concurrency.md)), and
`--gtest_shuffle` (randomize order to catch tests that secretly depend on
each other).

## Cheat sheet

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

## Traps

**`EXPECT_EQ` on two `char*` compares pointers, not text.** `EXPECT_EQ(name,
"Ada")` on a `const char*` almost always fails even when the characters
match, because two separate string literals need not share an address. Use
`EXPECT_STREQ` for C strings, or compare `std::string` values (where
`operator==` is a content comparison and `EXPECT_EQ` is correct).

**`EXPECT_EQ` on `double` compares exact bit patterns.** `0.1 + 0.2` is not
`0.3` in binary floating point. Use `EXPECT_DOUBLE_EQ` (a few-ULP tolerance)
or `EXPECT_NEAR(a, b, 1e-9)` for an explicit one.

**`ASSERT_*` only returns from the function it appears in.** If you factor
assertions into a helper that returns `void`, an `ASSERT_*` failure aborts the
*helper*, and the calling test carries on as if nothing happened. Either keep
assertions in the test body, or use `EXPECT_*` in helpers.

**Tests that share global state are order-dependent and will eventually
bite.** The per-test fixture reconstruction protects fixture members, not
globals or `static` locals. Run with `--gtest_shuffle` occasionally to find
tests that only pass in one particular order.

## Exercise

Write a `Stack` class (fixed capacity, `push`, `pop`, `top`, `empty`,
`size`; `pop`/`top` on an empty stack throws `std::out_of_range`, `push` on a
full one throws `std::overflow_error`). Then write a gtest suite for it using
a `TEST_F` fixture that starts with an empty stack of capacity 3, covering:
LIFO ordering, `size()` after a mix of pushes and pops, both exception cases
with `EXPECT_THROW`, and an `ASSERT_FALSE(s.empty())` guard before a `top()`
check. Confirm all tests pass, then deliberately break `pop()` (forget to
shrink the size) and read the failure report to see exactly which cases catch
it — a good test suite should fail in more than one place.
