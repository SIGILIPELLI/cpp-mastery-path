# 08 · References & Pointers

References and pointers both let you refer to an existing variable without
copying it — but they work differently and are used in different situations.

## The address-of operator `&`

```cpp
#include <iostream>

int main() {
    int x = 42;
    std::cout << x << std::endl;    // 42 -- the value
    std::cout << &x << std::endl;   // 0x16b... -- the memory address where x lives
}
```

Every variable lives at some address in memory; `&x` gives you that address
rather than the value stored there.

## Pointers

```cpp
int x = 42;
int* ptr = &x;        // ptr holds the address of x

std::cout << *ptr << std::endl;    // 42 -- "*ptr" dereferences: "the value at this address"

*ptr = 100;            // modifies x through the pointer
std::cout << x << std::endl;       // 100 -- x itself changed
```

`int*` declares a pointer variable that stores an address of an `int`. `*ptr`
(the dereference operator) accesses the value stored at that address. Note
that `*` means two different things depending on context: in a declaration
(`int* ptr`) it says "this is a pointer type"; in an expression (`*ptr`) it
dereferences.

## `nullptr`

```cpp
int* ptr = nullptr;   // points to nothing -- the safe way to say "no address yet"

if (ptr == nullptr) {
    std::cout << "ptr is not pointing anywhere" << std::endl;
}

// std::cout << *ptr << std::endl;   // undefined behavior -- dereferencing null crashes
```

Always initialize pointers — either to a real address or to `nullptr` — and
check for `nullptr` before dereferencing a pointer that might not point
anywhere valid. Uninitialized pointers hold garbage addresses and are one of
the most common sources of crashes in C-style code.

## References

```cpp
int x = 42;
int& ref = x;    // ref is an ALIAS for x -- not a separate variable

ref = 100;
std::cout << x << std::endl;    // 100 -- changing ref changes x directly

x = 7;
std::cout << ref << std::endl;  // 7 -- they always refer to the same storage
```

A reference (`int&`) must be bound to a variable at the moment it's declared,
and it can never be rebound to refer to something else afterward — unlike a
pointer, which can be reassigned or set to `nullptr`. There's also no
dereference operator needed: you use `ref` exactly like you'd use `x`.

## Pointers vs. references at a glance

| | Pointer (`int*`) | Reference (`int&`) |
|---|---|---|
| Can be null | Yes (`nullptr`) | No — must always refer to something |
| Can be reassigned | Yes | No — bound once, forever |
| Needs dereferencing (`*`) | Yes | No |
| Typical use | Optional values, dynamic data structures, low-level APIs | Function parameters, avoiding copies |

## Pass by reference (revisited)

This is the most common everyday use of references — passing arguments to
functions without copying, and optionally letting the function modify the
caller's variable (first introduced in [Module 4](04-functions-overloading.md)):

```cpp
void doubleValue(int& n) {   // n is a reference to the caller's variable
    n *= 2;
}

void printInfo(const std::string& name) {   // const& avoids a copy, and forbids modification
    std::cout << "Name: " << name << std::endl;
}

int main() {
    int value = 21;
    doubleValue(value);
    std::cout << value << std::endl;   // 42

    printInfo("Ada");   // no copy of the string is made
}
```

`const T&` parameters are extremely common in idiomatic C++: they get the
efficiency of passing by reference (no copy) with the safety of passing by
value (the function can't modify your data).

## Pass by pointer

```cpp
void reset(int* n) {
    if (n != nullptr) {   // always check before dereferencing
        *n = 0;
    }
}

int main() {
    int value = 99;
    reset(&value);                 // pass the address explicitly with &
    std::cout << value << std::endl;   // 0

    reset(nullptr);                // safe -- the function checks first
}
```

Pass-by-pointer is chosen over pass-by-reference specifically when "no value"
is a meaningful possibility (you can pass `nullptr`), or in APIs that
originated in C. When the argument is always required, prefer a reference —
it can't accidentally be null.

## A quick rule of thumb

- Use a **reference** when the parameter is required and you either want to
  avoid a copy (`const T&`) or want to modify the caller's variable (`T&`).
- Use a **pointer** when the value might legitimately be absent (`nullptr`),
  or when you need to reseat it to point somewhere else later.

## Exercise

Write a function `void swapValues(int& a, int& b)` that swaps two integers
using references (no `std::swap`). Then write a function `bool findFirstNegative(const std::vector<int>& numbers, int* outIndex)`
that scans the vector for the first negative number: if found, stores its
index through `outIndex` and returns `true`; if `outIndex` is `nullptr` or no
negative number exists, returns `false` without dereferencing a null pointer.
