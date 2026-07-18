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

## Exercise

Write a program that declares a rectangle's `width` and `height` as `double`,
computes and prints its area and perimeter, then declares an `int` number of
`items` and a `double` `pricePerItem`, computing the total cost. Use `const`
for any value that shouldn't change, and use `static_cast` to print the total
cost rounded down to a whole number of dollars.
