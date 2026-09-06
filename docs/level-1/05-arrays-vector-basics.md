# 05 · Arrays & std::vector Basics

C++ gives you two main ways to store a sequence of values: fixed-size
C-style arrays, and the dynamically-sized `std::vector`.

## C-style arrays

```cpp
#include <iostream>

int main() {
    int scores[5] = {90, 85, 77, 92, 88};

    std::cout << scores[0] << std::endl;   // 90 -- indexing starts at 0
    std::cout << scores[4] << std::endl;   // 88 -- last valid index is size - 1

    scores[2] = 100;                       // arrays are mutable
    std::cout << scores[2] << std::endl;   // 100

    int size = sizeof(scores) / sizeof(scores[0]);
    std::cout << "size: " << size << std::endl;   // 5
}
```

C-style arrays have a **fixed size decided at compile time** and don't know
their own length — you have to track it yourself (or compute it with the
`sizeof` trick above, which only works for arrays, not pointers). There's also
**no bounds checking**: `scores[10]` compiles and may silently corrupt memory
or crash. For these reasons, modern C++ code reaches for `std::vector` almost
everywhere a raw array might have been used in C.

## std::vector — a dynamic, resizable array

```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int> numbers;        // starts empty
    numbers.push_back(10);           // append an element
    numbers.push_back(20);
    numbers.push_back(30);

    std::cout << numbers.size() << std::endl;   // 3
    std::cout << numbers[0] << std::endl;        // 10
    std::cout << numbers.at(1) << std::endl;     // 20

    numbers[1] = 25;                             // modify in place
    std::cout << numbers[1] << std::endl;        // 25
}
```

`std::vector<int>` declares a vector holding `int` elements — the type in
angle brackets is the *element type*. `push_back` grows the vector as needed,
so you never have to decide a size up front.

## `operator[]` vs `.at()`

```cpp
std::vector<int> v = {1, 2, 3};

std::cout << v[10] << std::endl;    // undefined behavior -- no bounds check, may crash or return garbage
// std::cout << v.at(10) << std::endl;  // throws std::out_of_range -- safe, checked
```

`operator[]` is fast but unchecked; `.at()` is slightly slower but throws a
catchable exception on an invalid index. Prefer `.at()` when indices come
from outside input, and `[]` in tight loops where you've already validated
the range. Exceptions are covered in [Module 9](09-exception-handling.md).

## Initializing a vector with values

```cpp
std::vector<std::string> names = {"Alice", "Bob", "Carol"};
std::vector<double> prices(5, 0.0);   // 5 elements, each initialized to 0.0

std::cout << names.size() << std::endl;   // 3
std::cout << prices.size() << std::endl;  // 5
```

## Iterating over a vector

```cpp
std::vector<int> temps = {68, 72, 75, 70, 66};

// Index-based
for (size_t i = 0; i < temps.size(); i++) {
    std::cout << temps[i] << " ";
}
std::cout << std::endl;

// Range-based (preferred when you don't need the index)
for (int t : temps) {
    std::cout << t << " ";
}
std::cout << std::endl;
// 68 72 75 70 66   (both loops)
```

`size()` returns a `size_t` (an unsigned integer type), so the loop counter
`i` should be `size_t` too — comparing a signed `int` against an unsigned
`size_t` can trigger compiler warnings and subtle bugs with negative values.

## Common vector operations

```cpp
std::vector<int> v = {5, 3, 8, 1};

v.push_back(9);           // {5, 3, 8, 1, 9}
v.pop_back();              // {5, 3, 8, 1} -- removes the last element
std::cout << v.empty() << std::endl;   // 0 (false) -- not empty
std::cout << v.front() << std::endl;   // 5 -- first element
std::cout << v.back() << std::endl;    // 1 -- last element
v.clear();                 // removes all elements
std::cout << v.empty() << std::endl;   // 1 (true)
```

## A 2D grid with a vector of vectors

```cpp
std::vector<std::vector<int>> grid = {
    {1, 2, 3},
    {4, 5, 6}
};

std::cout << grid[1][2] << std::endl;   // 6 -- row 1, column 2

for (const auto& row : grid) {
    for (int cell : row) {
        std::cout << cell << " ";
    }
    std::cout << std::endl;
}
// 1 2 3
// 4 5 6
```

## How It Actually Works

A C-style array `int arr[5];` declared inside a function is just **five
contiguous `int`-sized slots carved out of the current stack frame** — no
allocator involved, no bookkeeping stored anywhere near it. `arr[2]` doesn't
"look up" an element; it computes an address — `arr`'s base address plus
`2 * sizeof(int)` — and reads/writes memory there directly. That's why
arrays don't know their own length at runtime: the compiler only tracks the
size while compiling (`sizeof(arr)` works because the compiler still has the
type), but once the array decays to a raw pointer (e.g. when passed to a
function), that information is gone, and out-of-bounds access silently reads
or corrupts whatever memory happens to sit past the array — undefined
behavior with no bounds check to catch it.

`std::vector<int>` is a small, fixed-size object (typically three pointers:
begin, end, and end-of-capacity) that sits wherever you declare it, but it
owns a **separate block of heap memory** it allocates with `new[]` under the
hood for the actual elements. `push_back` checks if there's spare capacity;
if not, it allocates a *new*, larger block (commonly doubling the previous
capacity), move-or-copy-constructs every existing element into it, and frees
the old block — an operation that is `O(n)` on the rare occasions it happens,
but averages out to **amortized O(1)** per push because doubling means it
happens exponentially less often as the vector grows. This is also why
inserting a lot of elements is faster when you `reserve()` capacity
up-front: it eliminates the repeated reallocate-and-move cycles. And it's
why a pointer or iterator into a vector can be silently invalidated by a
`push_back` — the whole backing block may have moved to a new address.

## Exercise

Write a program that builds a `std::vector<int>` of the squares of the
numbers 1 through 10 (using `push_back` in a loop), then prints the vector's
`size()`, its sum (using a range-based `for` loop), and its largest element
(track a running maximum as you iterate). Use `.at()` when reading the first
and last elements.
