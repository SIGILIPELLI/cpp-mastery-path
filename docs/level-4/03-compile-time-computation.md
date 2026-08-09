# 03 · Compile-time Computation

C++ has two execution environments. One is the CPU at runtime; the other is the
compiler, which can run a surprisingly large subset of the language while
building your program. Work moved into the second environment costs nothing at
runtime, cannot fail in production, and — this is the underrated part — turns a
whole class of bugs into build errors.

This module covers `constexpr`, `consteval`, `constinit`, `static_assert`,
`if constexpr`, fold expressions, and the template metaprogramming they mostly
replaced.

## `constexpr` — *may* run at compile time

A `constexpr` function is one the compiler is *allowed* to evaluate at build
time, when its arguments are themselves constants. The same function still works
perfectly well at runtime with runtime arguments.

```cpp
#include <array>
#include <iostream>
#include <string_view>

constexpr unsigned long long factorial(int n) {
    return n <= 1 ? 1ULL : n * factorial(n - 1);
}

// C++14 onwards: loops, locals and mutation are all fine in constexpr
constexpr std::array<int, 10> firstPrimes() {
    std::array<int, 10> out{};
    int found = 0, n = 2;
    while (found < 10) {
        bool prime = true;
        for (int d = 2; d * d <= n; ++d) if (n % d == 0) { prime = false; break; }
        if (prime) out[found++] = n;
        ++n;
    }
    return out;
}

constexpr std::size_t hashOf(std::string_view s) {   // FNV-1a
    std::size_t h = 1469598103934665603ULL;
    for (char c : s) { h ^= static_cast<unsigned char>(c); h *= 1099511628211ULL; }
    return h;
}

int main() {
    static_assert(factorial(20) == 2432902008176640000ULL);
    constexpr auto primes = firstPrimes();     // computed by the COMPILER
    static_assert(primes[9] == 29);
    static_assert(hashOf("GET") != hashOf("PUT"));

    std::cout << "20! = " << factorial(20) << "\n";
    std::cout << "primes:";
    for (int p : primes) std::cout << " " << p;
    std::cout << "\nhash(\"GET\") = " << hashOf("GET") << "\n";
    std::cout << "sizeof(primes) = " << sizeof(primes) << " bytes, in .rodata\n";
}
```

```text
20! = 2432902008176640000
primes: 2 3 5 7 11 13 17 19 23 29
hash("GET") = 15932647310772317037
sizeof(primes) = 40 bytes, in .rodata
```

`primes` is not computed when the program starts. The 40 bytes are baked into
the binary's read-only data by the compiler, exactly as if you had typed the ten
numbers out — but with the sieve as the single source of truth. That is the core
value proposition: **generated tables you cannot get wrong**.

`static_assert` is the verification tool for this world. It runs at compile time
and fails the build with your message. Every `static_assert` above is a test
that can never be skipped, never flake, and costs zero runtime.

## `consteval` — *must* run at compile time (C++20)

`constexpr` is permission; `consteval` is a requirement. A `consteval` function
that cannot be evaluated at compile time is a compile error, not a silent
fallback to runtime.

```cpp
consteval int square(int n) { return n * n; }

constexpr int s = square(12);   // fine: 144, computed at build time
```

Feed it a runtime value:

```cpp
int r; std::cin >> r; std::cout << square(r);
```

```text
error: call to consteval function 'square' is not a constant expression
note: read of non-const variable 'r' is not allowed in a constant expression
```

Use `consteval` when running at runtime would be a bug rather than merely slow —
a compile-time string validator, a units-checked literal, a lookup-table
generator that must never appear in the binary as executable code.

`constinit` (also C++20) is the third member of the family, and it solves a
different problem: it asserts that a static-storage variable is initialized at
compile time, eliminating the **static initialization order fiasco** without
forcing the variable to be `const`.

```cpp
constinit int g_counter = square(4);   // guaranteed zero dynamic init
```

## `if constexpr` — branches that don't exist

An ordinary `if` requires both branches to compile. `if constexpr` discards the
untaken branch *before* it is instantiated, which is what makes generic code
over unrelated types possible without overload tricks.

```cpp
#include <type_traits>

template <typename T>
constexpr const char* kindOf() {
    if constexpr (std::is_integral_v<T>)            return "integral";
    else if constexpr (std::is_floating_point_v<T>) return "floating";
    else if constexpr (std::is_pointer_v<T>)        return "pointer";
    else                                            return "other";
}
```

```text
int -> integral, double -> floating, char* -> pointer
```

This one construct replaced most of the tag-dispatch and `enable_if` machinery
from [Level 3 Module 1](../level-3/01-advanced-templates.md).

## Fold expressions (C++17)

Variadic templates used to need a recursive base case plus a recursive step for
every reduction. A fold expression is one line:

```cpp
template <typename... Ts>
constexpr auto sumAll(Ts... vs) { return (vs + ... + 0); }        // binary right fold

template <typename... Ts>
constexpr bool allPositive(Ts... vs) { return ((vs > 0) && ...); } // unary fold

static_assert(sumAll(1, 2, 3, 4) == 10);
static_assert(allPositive(1, 2, 3));
static_assert(!allPositive(1, -2, 3));
```

```text
sumAll(1,2,3,4) = 10
sumAll(1.5, 2.5) = 4
```

The `+ 0` is the identity element that makes the zero-argument case valid. Any
binary operator works — `,` is the trick for "do this to every argument":
`(std::cout << ... << vs);` prints them all.

## Compile-time dispatch: hashing strings into a `switch`

`switch` requires integral constant labels, so it cannot switch on a string —
unless the hash is `constexpr`:

```cpp
constexpr std::size_t operator""_fnv(const char* s, std::size_t n) {
    return hashOf(std::string_view(s, n));
}

const char* route(std::string_view method) {
    switch (hashOf(method)) {                  // hashed at RUNTIME (one pass)
        case "GET"_fnv:    return "read handler";     // hashed at COMPILE time
        case "POST"_fnv:   return "create handler";
        case "DELETE"_fnv: return "delete handler";
        default:           return "405";
    }
}
```

```text
read handler / create handler / 405
```

The case labels are integers by the time the compiler emits the jump table, so
this is a hash plus a jump instead of a chain of `strcmp` calls. It is also the
pattern's honest weakness: two colliding strings would silently route to the
same handler, and you would want a `static_assert` proving your specific label
set is collision-free.

## Template metaprogramming, and why it's shrinking

Before `constexpr` functions, compile-time arithmetic meant recursive class
templates with `static constexpr` members:

```cpp
template <int N> struct Fib { static constexpr long value = Fib<N-1>::value + Fib<N-2>::value; };
template <> struct Fib<0> { static constexpr long value = 0; };
template <> struct Fib<1> { static constexpr long value = 1; };

constexpr long fib(int n) {                      // the modern equivalent
    long a = 0, b = 1;
    for (int i = 0; i < n; ++i) { long t = a + b; a = b; b = t; }
    return a;
}

static_assert(Fib<40>::value == fib(40));
```

```text
Fib<40> = 102334155 == fib(40) = 102334155
```

Same answer. The `Fib<N>` version instantiates 41 distinct class templates,
bloats the compiler's memory, and produces incomprehensible errors; the
`constexpr` version is a loop anyone can read and is dramatically faster to
compile. Reach for template metaprogramming only for things that operate on
*types* — `std::conditional_t`, type lists, trait detection — because types are
the one thing `constexpr` functions still cannot compute.

## Cheat sheet

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

## Traps

**`constexpr` is a permission, not a promise.** `constexpr int x = f(a);`
forces compile-time evaluation because `x` is `constexpr`; a bare `f(a);` with a
runtime `a` just runs at runtime. If you need the guarantee, assign to a
`constexpr` variable, use it in a `static_assert`, or declare the function
`consteval`.

**Recursion depth and step limits are finite.** Compilers cap constant
evaluation (`-fconstexpr-depth`, `-fconstexpr-steps`, `-fconstexpr-ops-limit`).
A `constexpr` loop over ten million iterations will not fail at runtime — it
will fail the *build*, often with a confusing message. Keep compile-time work
proportionate.

**C++20 `constexpr` allocation cannot escape.** You may use `std::vector` inside
a constant-evaluated function, but every allocation must be freed before the
evaluation ends. Returning a `constexpr std::vector` to namespace scope does not
compile; return a `std::array` sized by a separate `constexpr` counting pass.

**Everything `constexpr` in a header is recompiled everywhere.** Heavy
compile-time computation is not free — it is paid by every translation unit that
includes it, on every build. Profile build times (`-ftime-trace` on Clang)
before scattering elaborate `constexpr` machinery across headers.

**`static_assert` inside an uninstantiated template never fires.** A
`static_assert(false)` in a template body is ill-formed in some compilers and
silently ignored in others until instantiation. Make the condition depend on the
template parameter — `static_assert(sizeof(T) == 0, "...")` — or use a
`requires` clause instead.

**`std::is_constant_evaluated()` is always true inside an `if constexpr`.** It
is a runtime-ish function whose value the compiler knows; putting it in
`if constexpr` makes the condition trivially true and silently deletes your
runtime path. Use a plain `if`.

## Exercise

Build a compile-time, collision-checked HTTP status table.

1. Write `constexpr std::string_view statusText(int code)` covering at least
   200, 201, 204, 400, 401, 403, 404, 409, 500, 503, returning `"unknown"`
   otherwise. Prove it with `static_assert(statusText(404) == "Not Found");`.
2. Write `constexpr std::array<int, N> knownCodes()` listing the codes, and a
   `constexpr bool allDistinct(...)` that verifies no duplicates — then
   `static_assert(allDistinct(knownCodes()));`.
3. Add a `consteval` factory `Status make(int code)` that fails the build for
   any code not in `knownCodes()`. Confirm that `make(499)` produces a compile
   error and `make(404)` does not.
4. Extend the `_fnv` routing example with a `constexpr` check that all your
   case-label strings hash distinctly, as a `static_assert`. Then deliberately
   add a string that collides (find one by brute force in a small runtime
   program) and watch the build fail.

Finally, run `g++ -ftime-report` (or `clang++ -ftime-trace`) on the result and
note how much build time your compile-time work actually costs. Everything in
this module is a trade, and this is the price side of it.
