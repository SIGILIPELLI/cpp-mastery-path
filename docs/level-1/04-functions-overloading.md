# 04 · Functions & Overloading

Functions package up reusable logic behind a name, parameters, and a return
type.

## Declaring and calling functions

```cpp
#include <iostream>

int add(int a, int b) {
    return a + b;
}

int main() {
    int result = add(3, 4);
    std::cout << result << std::endl;   // 7
}
```

`int add(int a, int b)` declares a function named `add` that takes two `int`
parameters and returns an `int`. The body executes when the function is
*called*, not when it's defined.

## Void functions (no return value)

```cpp
void printBanner(const std::string& title) {
    std::cout << "=== " << title << " ===" << std::endl;
}

int main() {
    printBanner("Report");
    // === Report ===
}
```

`void` means the function performs an action but doesn't hand a value back to
the caller. A bare `return;` (with no value) can still be used to exit early.

## Parameters: pass by value vs. pass by reference

```cpp
void increment(int x) {        // pass by value -- x is a COPY
    x++;
}

void incrementRef(int& x) {    // pass by reference -- x IS the original
    x++;
}

int main() {
    int a = 10;
    increment(a);
    std::cout << a << std::endl;      // 10 -- unchanged, the function modified its own copy

    incrementRef(a);
    std::cout << a << std::endl;      // 11 -- changed, since x refers to a directly
}
```

By default C++ passes arguments **by value** — the function receives a copy.
Adding `&` to the parameter type makes it a **reference** parameter, which
lets the function modify the caller's variable directly. References are
covered in depth in [Module 8](08-references-pointers.md).

## Passing larger objects efficiently with `const&`

```cpp
#include <string>

// Passing std::string by value would copy the whole string on every call.
// const& avoids the copy while still preventing the function from modifying it.
void greet(const std::string& name) {
    std::cout << "Hello, " << name << "!" << std::endl;
}
```

A good default rule: pass small, cheap-to-copy types (`int`, `double`,
`char`, `bool`) by value, and pass larger types (`std::string`,
`std::vector`, your own classes) by `const&` unless the function needs to
modify the caller's copy.

## Default arguments

```cpp
double calculatePrice(double base, double taxRate = 0.08) {
    return base + (base * taxRate);
}

int main() {
    std::cout << calculatePrice(100.0) << std::endl;        // 108 -- uses default 0.08
    std::cout << calculatePrice(100.0, 0.05) << std::endl;  // 105 -- overrides the default
}
```

Default arguments must be trailing (you can't have a defaulted parameter
followed by a non-defaulted one).

## Function overloading

C++ lets you define multiple functions with the **same name** as long as
their parameter lists differ in type or count — the compiler picks the right
one based on the arguments at the call site.

```cpp
int multiply(int a, int b) {
    return a * b;
}

double multiply(double a, double b) {
    return a * b;
}

int multiply(int a, int b, int c) {
    return a * b * c;
}

int main() {
    std::cout << multiply(3, 4) << std::endl;         // 12       -- calls int version
    std::cout << multiply(2.5, 4.0) << std::endl;      // 10       -- calls double version
    std::cout << multiply(2, 3, 4) << std::endl;       // 24       -- calls 3-arg version
}
```

Overload resolution is based purely on the parameter list — the return type
alone cannot distinguish two overloads.

## Function declarations vs. definitions

So far each function has been defined where it's used. In larger programs
you typically **declare** a function's signature (often in a header file) and
**define** its body separately:

```cpp
// Declaration (a "prototype") -- tells the compiler this function exists
int square(int n);

int main() {
    std::cout << square(5) << std::endl;   // 25 -- can call before the definition appears
}

// Definition -- the actual implementation, can appear later in the file
int square(int n) {
    return n * n;
}
```

This becomes essential once code is split across multiple `.cpp` and `.h`
files, which you'll do starting in [Module 10](10-project-bank-account-cli.md).

## Exercise

Write overloaded functions named `area`: one taking a single `double` (for a
square's side), one taking two `double`s (for a rectangle's width and
height), and one taking a `double` radius that computes a circle's area using
`3.14159`. Call all three from `main` and print the results.
