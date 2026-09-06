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

## How It Actually Works

A pointer is a variable whose value is a memory address — literally an
integer-sized (8 bytes on 64-bit systems) number that the CPU interprets as
"start reading/writing here." `&x` computes the address the compiler already
assigned `x` at compile time (its offset within the stack frame, or its
address in static/heap memory); `*p` means "go to the address stored in `p`
and read/write the bytes there." Dereferencing a pointer that holds garbage
or a freed address is undefined behavior precisely because the CPU will
happily read/write whatever is at that address — there's no safety net; it
might belong to another variable, might be unmapped memory (causing a
segmentation fault when the OS's memory manager notices), or might silently
"work" and corrupt something else instead.

A **reference** is not a separate runtime object at all in most
implementations — the compiler treats `int& r = x;` as another *name* for
the exact same memory location as `x`, and every use of `r` is compiled as
if you'd written `x` directly (or, when it can't be resolved to a direct
alias, as a pointer under the hood that the compiler dereferences
automatically). This is why references can't be null and can't be
reseated: the language enforces at compile time that a reference is bound
once, to one existing object, so there's no "dangling address with no
target" state to represent unless you deliberately create one by returning
a reference to something that has already been destroyed — at which point
the compiled code still tries to read that now-invalid memory location, no
different in mechanism from a dangling pointer.

`nullptr` is a pointer value guaranteed to compare unequal to every valid
object address; dereferencing it triggers a hardware-level fault on virtually
every platform because address `0` is deliberately left unmapped by the OS.

## Exercise

Write a function `void swapValues(int& a, int& b)` that swaps two integers
using references (no `std::swap`). Then write a function `bool findFirstNegative(const std::vector<int>& numbers, int* outIndex)`
that scans the vector for the first negative number: if found, stores its
index through `outIndex` and returns `true`; if `outIndex` is `nullptr` or no
negative number exists, returns `false` without dereferencing a null pointer.
