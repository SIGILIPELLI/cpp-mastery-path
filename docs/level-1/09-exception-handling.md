# 09 · Exception Handling Basics

Exceptions represent errors or unusual conditions detected at runtime. C++
lets you handle them explicitly instead of crashing the whole program.

## try / catch

```cpp
#include <iostream>
#include <stdexcept>

int main() {
    int a = 10;
    int b = 0;

    try {
        if (b == 0) {
            throw std::runtime_error("Cannot divide by zero");
        }
        std::cout << a / b << std::endl;
    } catch (const std::runtime_error& e) {
        std::cout << "Error: " << e.what() << std::endl;
    }

    std::cout << "Program continues normally." << std::endl;
}
// Error: Cannot divide by zero
// Program continues normally.
```

Unlike some languages, C++ does **not** throw an exception on integer
division by zero automatically — that's actually undefined behavior at the
hardware level. You must check and `throw` explicitly, as shown above.
`e.what()` returns a C-string description of the error.

## The standard exception hierarchy

```cpp
#include <stdexcept>

// std::exception                      -- the base class for all standard exceptions
//   +-- std::logic_error               -- errors detectable before running (bad arguments, etc.)
//   |     +-- std::invalid_argument
//   |     +-- std::out_of_range
//   |     +-- std::length_error
//   +-- std::runtime_error             -- errors only detectable at runtime
//         +-- std::overflow_error
//         +-- std::underflow_error
```

| Exception | Typically thrown when |
|-----------|------------------------|
| `std::invalid_argument` | A function received an argument that doesn't make sense |
| `std::out_of_range` | An index or key is outside the valid range (e.g. `vector::at`) |
| `std::runtime_error` | A general error only detectable while running |
| `std::logic_error` | A programming/logic mistake, in principle avoidable |

All of these derive from `std::exception`, so a single `catch (const
std::exception& e)` can catch any of them.

## Catching multiple exception types

```cpp
#include <vector>
#include <string>

void process(const std::string& input) {
    try {
        int value = std::stoi(input);
        std::cout << "Parsed: " << value << std::endl;
    } catch (const std::invalid_argument& e) {
        std::cout << "Not a number: " << input << std::endl;
    } catch (const std::out_of_range& e) {
        std::cout << "Number too large: " << input << std::endl;
    }
}

int main() {
    process("42");        // Parsed: 42
    process("oops");      // Not a number: oops
    process("99999999999999999999");  // Number too large: 99999999999999999999
}
```

Catch blocks are checked in order, top to bottom — put more specific
exception types before more general ones (a `catch (const std::exception&)`
placed first would swallow everything below it).

## Catching the base class as a safety net

```cpp
try {
    riskyOperation();
} catch (const std::exception& e) {
    // Catches std::runtime_error, std::invalid_argument, std::out_of_range,
    // and any other standard exception -- useful as a final fallback.
    std::cout << "Something went wrong: " << e.what() << std::endl;
}
```

## Throwing your own errors with standard exception types

```cpp
void validateAge(int age) {
    if (age < 0 || age > 150) {
        throw std::invalid_argument("Age out of range: " + std::to_string(age));
    }
}

int main() {
    try {
        validateAge(-5);
    } catch (const std::invalid_argument& e) {
        std::cout << "Validation failed: " << e.what() << std::endl;
    }
}
// Validation failed: Age out of range: -5
```

You don't need to write a custom exception class to throw a meaningful error
— reusing `std::invalid_argument`, `std::runtime_error`, etc. with a
descriptive message covers most everyday cases. Custom exception classes
(deriving from `std::exception`) are covered in Level 2.

## `catch(...)` — catching anything

```cpp
try {
    riskyOperation();
} catch (...) {
    std::cout << "An unknown error occurred." << std::endl;
}
```

`catch (...)` catches literally anything thrown, including types that aren't
`std::exception` at all. It's a last resort — you lose all information about
what actually went wrong, so prefer catching specific types whenever
possible.

## `finally`-equivalent: RAII, not a keyword

C++ has no `finally` keyword. Instead, cleanup that "always runs" is achieved
through **RAII** (Resource Acquisition Is Initialization) — object
destructors run automatically when a scope ends, whether it ended normally or
via an exception. You'll cover this pattern properly in Level 3, but it's
worth knowing the name now: it's *the* idiomatic C++ answer to "make sure
this cleanup always happens."

## Exercise

Write a function `int safeDivide(int a, int b)` that returns `a / b`, but
throws `std::invalid_argument` with the message `"Cannot divide by zero"` if
`b` is `0`. In `main`, call it inside a `try/catch` for both a valid and a
zero divisor, printing the result or the caught error message. Then write a
function `int parseAge(const std::string& input)` that calls `std::stoi`,
catches `std::invalid_argument`, and rethrows it with the clearer message
`"Invalid age: " + input`.
