# 01 · Modern C++ Features (17/20)

C++17 and C++20 changed what idiomatic C++ looks like more than any release
since C++11. Code that used to need an out-parameter, a `bool` success flag, a
hand-rolled tagged union, or four lines of iterator boilerplate now says what it
means in one line. This module covers the features that show up constantly in
modern codebases, with the emphasis on *why each one exists* — every one of them
replaces a specific, familiar pain.

Everything here compiles with `-std=c++20` on GCC 11+, Clang 14+, or MSVC
19.30+. Where a feature is C++17 it is marked, because plenty of production code
is still pinned there.

## Structured bindings (C++17)

Unpacking a pair, tuple, or plain struct used to mean `.first`, `.second`, or
`std::get<0>`. Now it means naming the parts:

```cpp
#include <iostream>
#include <map>
#include <string>
#include <tuple>

struct Point { int x; int y; };

std::tuple<std::string, int, double> lookup() { return {"widget", 42, 9.99}; }

int main() {
    Point p{3, 4};
    auto [x, y] = p;                       // works on any public-fields struct
    std::cout << "point: " << x << "," << y << "\n";

    auto [name, qty, price] = lookup();    // and on tuples
    std::cout << "row: " << name << " x" << qty << " @ " << price << "\n";

    std::map<std::string, int> stock{{"bolts", 10}, {"nuts", 5}};
    for (const auto& [key, count] : stock) // and on map elements -- the big win
        std::cout << "  " << key << " = " << count << "\n";
}
```

```text
point: 3,4
row: widget x42 @ 9.99
  bolts = 10
  nuts = 5
```

The map loop is where this pays off daily. `for (const auto& kv : stock)` forced
readers to remember that `kv.second` is the count; `[key, count]` states it.

## `if`/`switch` with initializer (C++17)

```cpp
if (auto it = stock.find("bolts"); it != stock.end())
    std::cout << it->second << " bolts\n";
// `it` does not exist here -- its scope ends with the if/else
```

The initializer runs, then the condition is evaluated, and the variable is
scoped to the `if` *and its `else`*. This is the same lifetime discipline RAII
gives you for resources, applied to lookup handles.

## `std::optional` — "a value, or nothing" (C++17)

The old ways to say "this might fail" were all bad: a sentinel value that a
valid input might legitimately produce, an out-parameter plus a `bool`, or a
pointer that the caller might forget to check.

```cpp
#include <optional>
#include <string_view>

std::optional<int> parsePort(std::string_view s) {
    int value = 0;
    for (char c : s) {
        if (c < '0' || c > '9') return std::nullopt;
        value = value * 10 + (c - '0');
    }
    return s.empty() ? std::nullopt : std::optional<int>{value};
}

int main() {
    if (auto port = parsePort("8080"); port)
        std::cout << "port " << *port << "\n";
    std::cout << "bad port -> " << parsePort("80x0").value_or(-1) << "\n";
}
```

```text
port 8080
bad port -> -1
```

`*port` and `port->` access the value, `value_or(fallback)` handles the empty
case inline, and `value()` throws `std::bad_optional_access` instead of being
undefined. The type itself now documents the contract, so a caller cannot
silently ignore the failure case the way they can with a returned `-1`.

## `std::variant` — a type-safe union (C++17)

```cpp
#include <variant>

using Field = std::variant<int, double, std::string>;

std::string describe(const Field& f) {
    return std::visit([](const auto& v) -> std::string {
        using T = std::decay_t<decltype(v)>;
        if constexpr (std::is_same_v<T, int>)         return "int " + std::to_string(v);
        else if constexpr (std::is_same_v<T, double>) return "double " + std::to_string(v);
        else                                          return "string \"" + v + "\"";
    }, f);
}

int main() {
    for (const Field& f : std::vector<Field>{7, 2.5, std::string("hi")})
        std::cout << describe(f) << "\n";
}
```

```text
int 7
double 2.500000
string "hi"
```

A `union` cannot hold a `std::string` safely — it has a non-trivial destructor,
and the union has no idea which member is live. `std::variant` tracks the active
alternative, runs the right destructor, and `std::visit` is checked at compile
time: add a fourth alternative to `Field` and the `if constexpr` chain fails to
compile until you handle it. That is exhaustiveness checking, and it is the
whole reason to prefer `variant` over a hand-rolled tag-plus-union.

`if constexpr` (also C++17) is what makes the generic lambda work: the branches
that don't apply to `T` are **discarded before instantiation**, so
`"string \"" + v` never has to be valid for `int`.

## Concepts (C++20)

Template errors used to be the worst diagnostics in programming — a page of
instantiation backtrace pointing at a line inside `<algorithm>`. Concepts move
the check to the call site and name the requirement:

```cpp
#include <concepts>
#include <numeric>
#include <vector>

template <typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template <Numeric T>
T mean(const std::vector<T>& v) {
    return std::accumulate(v.begin(), v.end(), T{}) / static_cast<T>(v.size());
}

int main() {
    std::cout << "mean int:    " << mean(std::vector<int>{1,2,3,4}) << "\n";
    std::cout << "mean double: " << mean(std::vector<double>{1.5, 2.5, 3.0}) << "\n";
}
```

```text
mean int:    2
mean double: 2.33333
```

(`mean` of `{1,2,3,4}` is 2, not 2.5, because `T` is `int` and the division
truncates — a good reminder that a generic function inherits the semantics of
its type parameter.)

Now call it with the wrong type, `mean(std::vector<std::string>{"a","b"})`:

```text
error: no matching function for call to 'mean'
note: candidate template ignored: constraints not satisfied [with T = std::string]
note: because 'std::string' does not satisfy 'Numeric'
note: because 'std::string' does not satisfy 'integral'
note: because 'is_integral_v<std::string>' evaluated to false
```

Five lines, in your own vocabulary, pointing at your own call. Compare that to
the SFINAE-era alternative from [Level 3 Module 1](../level-3/01-advanced-templates.md).

## Ranges (C++20)

The iterator-pair interface leaks: `std::sort(v.begin(), v.end())` names the
container twice and lets you mismatch two containers' iterators. Ranges take the
container itself, and views compose lazily.

```cpp
#include <algorithm>
#include <ranges>

struct Employee { std::string name; int age; double salary; };

int main() {
    std::vector<Employee> staff{
        {"Ada", 36, 120000}, {"Grace", 45, 150000},
        {"Linus", 28, 95000}, {"Bjarne", 52, 180000}, {"Ken", 31, 110000}};

    auto seniorNames = staff
        | std::views::filter([](const Employee& e) { return e.age >= 35; })
        | std::views::transform([](const Employee& e) { return e.name; });

    std::cout << "seniors:";
    for (const auto& n : seniorNames) std::cout << " " << n;
    std::cout << "\n";

    std::vector<int> nums{5, 3, 9, 1, 7, 3};
    std::ranges::sort(nums);                      // no .begin()/.end()
    std::cout << "found 7:    " << std::boolalpha
              << std::ranges::binary_search(nums, 7) << "\n";

    for (int n : nums | std::views::drop(2) | std::views::take(3))
        std::cout << "window " << n << "\n";
}
```

```text
seniors: Ada Grace Bjarne
found 7:    true
window 3
window 5
window 7
```

`seniorNames` allocates nothing and computes nothing at the point of definition.
The filter and transform run one element at a time, only as the `for` loop pulls
them. That laziness is why chaining views does not create an intermediate vector
per stage the way a chain of `std::copy_if`/`std::transform` calls would.

## The spaceship operator (C++20)

Writing `==`, `!=`, `<`, `<=`, `>`, `>=` by hand for one class is six functions
that must agree with each other. `operator<=>` generates all of them:

```cpp
struct Version {
    int major, minor, patch;
    auto operator<=>(const Version&) const = default;   // all six, lexicographic
    bool operator==(const Version&) const = default;
};

static_assert(Version{1,2,3} < Version{1,10,0});
static_assert(Version{2,0,0} >= Version{1,99,99});
```

The defaulted version compares members top to bottom, which is exactly what you
want for a version number, a date, or a sort key. Write it manually only when
the ordering is not member-wise.

## Cheat sheet

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

## Traps

**`std::optional<T&>` does not exist.** There is no optional reference in the
standard. Use `T*` (nullable by nature) or
`std::optional<std::reference_wrapper<T>>` — the former is almost always
clearer.

**A dangling `string_view`.** `std::string_view sv = getName();` where `getName`
returns `std::string` by value binds to a temporary that dies at the end of the
statement. `string_view` never owns; keep the owner alive for at least as long
as the view, and never return one that refers to a local.

**Views are invalidated like iterators.** A `filter_view` over a `std::vector`
holds onto the vector. `push_back` may reallocate, and every element the view
would have produced is then dangling. Materialize with `std::ranges::to`
(C++23) or a manual copy if the underlying container will change.

**`std::visit` requires every alternative be handled.** With a generic lambda
and `if constexpr`, a missing `else` branch is a compile error only if the
return types disagree — annotate the lambda's return type (`-> std::string`
above) so a forgotten branch fails loudly instead of deducing something
surprising.

**Structured bindings are not variables you can capture pre-C++20.** In C++17,
a lambda could not capture `x` from `auto [x, y] = ...`. C++20 fixed it, but
compilers vary; if a capture mysteriously fails, that's why.

**Concepts constrain, they do not overload-rank by default.** Two functions with
unrelated concepts are ambiguous for a type satisfying both. Subsumption only
orders constraints that are literally composed from the same atomic constraints,
so build concepts by refining others (`Numeric` → `std::integral`) rather than
writing two independent `requires` clauses.

## Exercise

Build a small config-value type using the features above. Define
`using Value = std::variant<bool, long, double, std::string>;` and a
`std::map<std::string, Value> config`. Then write:

1. `std::optional<Value> get(const std::map<std::string, Value>&, std::string_view key)` —
   returns `std::nullopt` for a missing key, and is called with an
   `if`-with-initializer at the call site.
2. `std::string render(const Value&)` using `std::visit` and `if constexpr`,
   printing `true`/`false` for the bool rather than `1`/`0`.
3. A concept `Stringable` satisfied by any type `T` for which
   `std::to_string(T{})` is valid (use a `requires` expression), and a
   constrained `template <Stringable T> Value make(T)`.
4. A ranges pipeline that prints the keys of every entry holding a `std::string`,
   sorted — `std::views::filter` on
   `std::holds_alternative<std::string>(kv.second)`, then
   `std::views::transform` to the key.

Confirm that calling `make(std::vector<int>{})` produces a short constraint
error naming `Stringable`, not a template backtrace.
