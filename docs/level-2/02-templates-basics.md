# 02 · Templates Basics

Function overloading ([Level 1 Module 4](../level-1/04-functions-overloading.md))
lets you write `max(int, int)` and `max(double, double)` separately. That gets
tedious fast, and it silently omits every type you forgot. Templates let you
write the algorithm **once**, with the type as a parameter, and let the
compiler generate a specialised version for each type you actually use.

This is not runtime polymorphism. There is no vtable, no indirection, no cost.
The compiler stamps out real, fully-typed code at compile time — which is why
the entire Standard Template Library is built on templates.

## Function templates

```cpp
#include <iostream>
#include <string>

template <typename T>
T maximum(T a, T b) {
    return (a > b) ? a : b;
}

int main() {
    std::cout << maximum(3, 7) << std::endl;              // 7        -- T deduced as int
    std::cout << maximum(2.5, 1.5) << std::endl;          // 2.5      -- T deduced as double
    std::cout << maximum('a', 'z') << std::endl;          // z        -- T deduced as char
    std::cout << maximum(std::string("apple"),
                         std::string("pear")) << std::endl;  // pear  -- T deduced as std::string

    std::cout << maximum<double>(3, 7.5) << std::endl;    // 7.5 -- explicit T forces conversion
}
```

`template <typename T>` introduces a **type parameter**. When you call
`maximum(3, 7)`, the compiler performs **template argument deduction**: it sees
two `int`s, sets `T = int`, and instantiates a concrete `int maximum(int, int)`.

`typename` and `class` are interchangeable here — `template <class T>` means
exactly the same thing. Modern code prefers `typename`.

Deduction is strict: `maximum(3, 7.5)` fails to compile because `T` cannot be
both `int` and `double`. You either supply `T` explicitly (`maximum<double>`)
or use two type parameters.

## Multiple type parameters and deduced return types

```cpp
#include <iostream>

// Two independent parameters -- the return type is deduced from the expression
template <typename T, typename U>
auto add(T a, U b) {
    return a + b;   // C++14: return type deduced as decltype(a + b)
}

template <typename T>
void printAll(const T& container) {
    for (const auto& item : container) {
        std::cout << item << ' ';
    }
    std::cout << std::endl;
}

int main() {
    std::cout << add(1, 2.5) << std::endl;      // 3.5  -- T=int, U=double, returns double
    std::cout << add(1, 2) << std::endl;        // 3    -- returns int

    std::vector<int> nums{1, 2, 3};
    printAll(nums);                              // 1 2 3
}
// Output:
// 3.5
// 3
// 1 2 3
```

Notice `printAll` never names a container type. It only requires that whatever
you pass supports `begin()`/`end()` and that its elements are streamable. This
is **duck typing at compile time**: templates constrain by *usage*, not by a
declared interface. Pass something that doesn't fit and you get an error — at
the point of instantiation, deep inside the template body, which is why
template error messages are famously long.

## Class templates

```cpp
#include <iostream>
#include <stdexcept>
#include <vector>

template <typename T>
class Stack {
public:
    void push(const T& value) {
        items.push_back(value);
    }

    T pop() {
        if (items.empty()) {
            throw std::out_of_range("pop from empty stack");
        }
        T top = items.back();
        items.pop_back();
        return top;
    }

    const T& peek() const {
        if (items.empty()) {
            throw std::out_of_range("peek on empty stack");
        }
        return items.back();
    }

    bool empty() const { return items.empty(); }
    std::size_t size() const { return items.size(); }

private:
    std::vector<T> items;
};

int main() {
    Stack<int> numbers;
    numbers.push(10);
    numbers.push(20);
    std::cout << numbers.pop() << std::endl;    // 20
    std::cout << numbers.size() << std::endl;   // 1

    Stack<std::string> words;
    words.push("hello");
    std::cout << words.peek() << std::endl;     // hello
}
```

Class templates are **not** deduced from constructor arguments before C++17 —
you write `Stack<int>`, naming the type explicitly. C++17 added **class
template argument deduction (CTAD)**, which is why `std::vector v{1, 2, 3};`
compiles today and deduces `std::vector<int>`.

`Stack<int>` and `Stack<std::string>` are two completely unrelated types. They
share source code, not a base class — you cannot assign one to the other, and
you cannot store both in the same container without type erasure.

## Non-type template parameters

Templates can take compile-time *values*, not just types:

```cpp
#include <iostream>
#include <array>

template <typename T, std::size_t N>
class FixedBuffer {
public:
    T& operator[](std::size_t i) { return data[i]; }
    const T& operator[](std::size_t i) const { return data[i]; }
    constexpr std::size_t size() const { return N; }   // known at compile time

private:
    T data[N]{};   // real, stack-allocated array -- no heap, no pointer chasing
};

int main() {
    FixedBuffer<double, 4> buf;
    buf[0] = 1.5;
    buf[3] = 9.0;
    std::cout << buf.size() << " " << buf[0] << " " << buf[3] << std::endl;
    // 4 1.5 9
}
```

`N` is baked into the type: `FixedBuffer<double, 4>` and `FixedBuffer<double, 8>`
are different types with different sizes. This is exactly how `std::array<T, N>`
works, and it is why `std::array` has zero overhead compared to a raw array.

## Default template arguments

```cpp
#include <vector>
#include <functional>

// Compare defaults to std::less<T>, so most callers never mention it
template <typename T, typename Compare = std::less<T>>
T smallest(const std::vector<T>& values, Compare comp = Compare{}) {
    T best = values.at(0);
    for (const T& v : values) {
        if (comp(v, best)) best = v;
    }
    return best;
}

int main() {
    std::vector<int> v{5, 2, 9, 1};
    std::cout << smallest(v) << std::endl;                    // 1
    std::cout << smallest(v, std::greater<int>{}) << std::endl; // 9 -- "smallest" by reversed order
}
```

This is the STL's own design pattern: a sensible default that most users never
override, plus a hook for the ones who need it. `std::map`, `std::sort`, and
`std::priority_queue` all take a comparator this way.

## Template specialisation

Sometimes one type genuinely needs different logic. **Full specialisation**
provides a hand-written version for one specific type:

```cpp
#include <iostream>
#include <string>

template <typename T>
std::string describe(const T& value) {
    return "some value: " + std::to_string(value);
}

// Full specialisation for bool -- std::to_string(bool) would print 1/0
template <>
std::string describe<bool>(const bool& value) {
    return value ? "yes" : "no";
}

int main() {
    std::cout << describe(42) << std::endl;      // some value: 42
    std::cout << describe(true) << std::endl;    // yes
}
```

The compiler prefers the specialisation whenever the type matches exactly, and
falls back to the primary template otherwise.

## The trap: templates live in headers

```cpp
// stack.h  -- CORRECT: definition lives in the header
template <typename T>
class Stack {
    void push(const T& v) { items.push_back(v); }   // defined inline
    // ...
};
```

```cpp
// stack.h
template <typename T> class Stack { void push(const T& v); };

// stack.cpp  -- WRONG: this will not link
template <typename T> void Stack<T>::push(const T& v) { items.push_back(v); }
```

The second version compiles but fails at link time with "undefined reference to
`Stack<int>::push`". A template is not code — it's a recipe. The compiler can
only generate `Stack<int>::push` if it can see the *body* at the point where
`Stack<int>` is used. Since `stack.cpp` is compiled separately and never sees
`Stack<int>`, no code is ever generated.

**Put template definitions in the header.** (A `.tpp`/`.ipp` file `#include`d
at the bottom of the header is a common way to keep it tidy.) This is the
single most common beginner template error.

## Cheat sheet

| Feature | Syntax |
|---------|--------|
| Function template | `template <typename T> T f(T a);` |
| Class template | `template <typename T> class C { ... };` |
| Multiple parameters | `template <typename T, typename U>` |
| Non-type parameter | `template <typename T, std::size_t N>` |
| Default argument | `template <typename T, typename C = std::less<T>>` |
| Explicit instantiation at call site | `f<double>(x)` |
| Full specialisation | `template <> T f<int>(int a) { ... }` |
| Where definitions go | **Header file**, always |

## Exercise

Write a class template `Pair<A, B>` that stores two values of possibly
different types, with `first()` and `second()` accessors, a `swapped()` method
returning a `Pair<B, A>`, and a `print()` method. Then write a free function
template `template <typename A, typename B> Pair<A, B> makePair(A a, B b)` so
callers get type deduction without naming the types.

Next, write a function template `template <typename T> T sum(const std::vector<T>& v)`
that returns the total. Call it with a `std::vector<int>` and a
`std::vector<double>`. Finally, try calling it with a `std::vector<std::string>` —
read the error message carefully and note which line inside the template body
the compiler blames.
