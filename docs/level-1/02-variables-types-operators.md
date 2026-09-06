# 02 · Variables, Types & Operators

C++ is statically typed: every variable has a fixed type, decided at compile
time, that never changes.

## Fundamental types

```cpp
#include <iostream>

int main() {
    int age = 30;                 // whole numbers, typically 32-bit
    double price = 19.99;         // double-precision floating point
    float ratio = 0.5f;           // single-precision (note the "f" suffix)
    char grade = 'A';             // a single character, in single quotes
    bool isActive = true;         // true / false
    long population = 8'000'000;  // digit separators (') improve readability

    std::cout << age << " " << price << " " << grade << " "
              << isActive << std::endl;
    // 30 19.99 A 1   -- bool prints as 1/0 by default
}
```

| Type | Typical size | Holds |
|------|-------------|-------|
| `int` | 4 bytes | Whole numbers |
| `double` | 8 bytes | Decimal numbers (default choice for floating point) |
| `float` | 4 bytes | Decimal numbers, less precision, less memory |
| `char` | 1 byte | A single character |
| `bool` | 1 byte | `true` or `false` |
| `long` / `long long` | 4 / 8 bytes | Larger whole numbers |

Sizes are platform/compiler dependent guarantees, not fixed constants — use
`sizeof(int)` if you need the exact size on your system.

## `const` and `auto`

```cpp
const double PI = 3.14159;   // cannot be reassigned after initialization
// PI = 3.0;                 // compile error: assignment of read-only variable

auto count = 10;             // compiler infers "int" from the initializer
auto name = std::string("Ada");  // infers "std::string"
```

Prefer `const` for values that never change — it documents intent and lets
the compiler catch accidental reassignment. `auto` is useful when the type is
obvious from context or verbose to spell out; don't overuse it where an
explicit type would be clearer to a reader.

## Arithmetic operators

```cpp
int a = 17, b = 5;

std::cout << a + b << std::endl;   // 22
std::cout << a - b << std::endl;   // 12
std::cout << a * b << std::endl;   // 85
std::cout << a / b << std::endl;   // 3  -- integer division truncates!
std::cout << a % b << std::endl;   // 2  -- remainder ("modulo")

double x = 17.0 / 5.0;
std::cout << x << std::endl;       // 3.4 -- floating-point division
```

Integer division truncating toward zero is one of the most common early bugs
— if you need a fractional result, make sure at least one operand is a
floating-point type.

## Comparison and logical operators

```cpp
int a = 5, b = 10;

std::cout << (a == b) << std::endl;  // 0 (false) -- equality
std::cout << (a != b) << std::endl;  // 1 (true)  -- inequality
std::cout << (a < b)  << std::endl;  // 1
std::cout << (a >= b) << std::endl;  // 0

bool loggedIn = true, isAdmin = false;
std::cout << (loggedIn && isAdmin) << std::endl;  // 0 -- AND
std::cout << (loggedIn || isAdmin) << std::endl;  // 1 -- OR
std::cout << (!isAdmin) << std::endl;             // 1 -- NOT
```

`=` is assignment; `==` is comparison. Mixing them up (`if (a = b)` instead of
`if (a == b)`) compiles but silently does the wrong thing — `-Wall` will warn
you about this, which is one more reason to always enable it.

## Compound assignment and increment operators

```cpp
int score = 10;
score += 5;   // score = score + 5  -> 15
score -= 3;   // -> 12
score *= 2;   // -> 24
score /= 4;   // -> 6

int i = 0;
i++;          // post-increment: i becomes 1
++i;          // pre-increment: i becomes 2
```

For loop counters and simple cases, `i++` and `++i` behave the same; the
difference (whether the *old* or *new* value is used as the expression's
result) matters once you use the operator inline, e.g. `arr[i++]`.

## Type conversion and casting

```cpp
int wholeNumber = 7;
double asDouble = wholeNumber;         // implicit widening: 7 -> 7.0

double price = 19.99;
int truncated = static_cast<int>(price);  // explicit narrowing: 19.99 -> 19

std::cout << truncated << std::endl;   // 19
```

`static_cast<T>(value)` is the safe, explicit way to convert between related
types in C++ — prefer it over the old C-style `(int)price` cast, which is
harder to search for and easier to misuse.

## How It Actually Works

Every fundamental type maps to a fixed number of bytes the compiler reserves
either in a CPU register or on the stack — there's no hidden object header
the way there is for, say, a Python `int`. On a typical 64-bit platform:
`bool` is 1 byte, `int` is 4 bytes, `double` is 8 bytes, `char` is 1 byte.
`sizeof(x)` asks the compiler for that number directly, computed entirely at
compile time — it costs nothing at runtime.

Declaring `int x = 5;` inside a function doesn't call any allocator: the
compiler has already decided, while generating machine code for that
function, how many bytes of stack space the function needs in total, and `x`
is just a fixed offset into that reserved block (e.g. "4 bytes starting at
`rbp - 12`" in x86-64 terms). Assigning to `x` is a single `mov` instruction.

Type conversions are where the mechanism matters most. `int i = 3.9;` doesn't
round — the compiler emits a truncating float-to-int conversion instruction,
so the fractional part is discarded, giving `3`. Mixing `int` and `double` in
an expression triggers **implicit promotion**: the `int` is widened to
`double` *before* the operation, so `7 / 2` is integer division (`3`,
remainder discarded at the machine level) while `7 / 2.0` promotes `7` to
`7.0` first and does floating-point division. Integer overflow on signed
types is undefined behavior — the bit pattern wraps according to two's
complement in practice on virtually every real compiler, but the standard
doesn't guarantee it, which is why sanitizers flag it even when the output
"looks right."

## Exercise

Write a program that declares a rectangle's `width` and `height` as `double`,
computes and prints its area and perimeter, then declares an `int` number of
`items` and a `double` `pricePerItem`, computing the total cost. Use `const`
for any value that shouldn't change, and use `static_cast` to print the total
cost rounded down to a whole number of dollars.
