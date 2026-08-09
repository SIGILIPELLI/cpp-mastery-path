# 01 · Advanced Templates

[Level 2's templates module](../level-2/02-templates-basics.md) covered
function and class templates: write the logic once, let the compiler stamp
out a version for each type you use. This module goes further into what
templates actually *are* — a compile-time code generator you can steer with
specialization, variadic packs, and conditional compilation. Get comfortable
here and things like `std::vector`, `std::function`, and `std::enable_if`
stop looking like magic.

## Full specialization — a different body for one exact type

A primary template gives the general case; a full specialization overrides it
for one specific type.

```cpp
#include <iostream>
#include <string>

template <typename T>
struct TypeName {
    static std::string get() { return "unknown"; }
};

// Full specialization: this exact type gets its own body entirely.
template <>
struct TypeName<int> {
    static std::string get() { return "int"; }
};

template <>
struct TypeName<double> {
    static std::string get() { return "double"; }
};

int main() {
    std::cout << TypeName<int>::get() << std::endl;      // int
    std::cout << TypeName<double>::get() << std::endl;    // double
    std::cout << TypeName<char>::get() << std::endl;      // unknown -- falls back to the primary template
}
```

Note `TypeName<>` here is a *class* template used purely to carry information
at compile time — there's no object, no constructor call, just
`TypeName<int>::get()`. This "type -> value" mapping is exactly how
`std::is_integral<T>` and friends work under the hood.

## Partial specialization — a pattern of types, not one exact type

Full specialization matches one concrete type. **Partial** specialization
matches a *shape* of type, like "any pointer" or "any `std::vector<T>`".
Only class templates can be partially specialized (function templates
cannot — overloading covers that case for functions).

```cpp
#include <iostream>
#include <string>

template <typename T>
struct TypeName {
    static std::string get() { return "T"; }
};

// Partial specialization: matches T* for ANY T, not one specific pointer type.
template <typename T>
struct TypeName<T*> {
    static std::string get() { return TypeName<T>::get() + "*"; }
};

template <>
struct TypeName<int> { static std::string get() { return "int"; } };

int main() {
    std::cout << TypeName<int>::get() << std::endl;     // int
    std::cout << TypeName<int*>::get() << std::endl;    // int*
    std::cout << TypeName<int**>::get() << std::endl;   // int**
}
```

`TypeName<int**>` matches `TypeName<T*>` with `T = int*`, which itself
recurses into `TypeName<T*>` with `T = int`, which hits the full
specialization. This recursive-partial-specialization trick is the classic
way to peel a compile-time type apart layer by layer.

## Variadic templates — functions over any number of arguments

Before C++11 there was no way to write one template that accepted 2, 3, or 7
arguments of possibly different types. A **parameter pack** does exactly that.

```cpp
#include <iostream>

// Base case: exactly one argument left, stop recursing.
template <typename T>
T sum(T value) {
    return value;
}

// Recursive case: peel off `first`, recurse on the rest of the pack.
template <typename T, typename... Rest>
T sum(T first, Rest... rest) {
    return first + sum(rest...);
}

int main() {
    std::cout << sum(1, 2, 3, 4) << std::endl;         // 10
    std::cout << sum(1.5, 2.25) << std::endl;          // 3.75
}
```

Each call to `sum` with a different argument count and type mix instantiates
a *different* overload set — the compiler generates exactly the recursion
depth it needs and nothing more, and modern optimizers inline the whole
chain away.

## Fold expressions (C++17) — the same idea, no recursion

Writing a base case and a recursive case for every variadic operation gets
old fast. C++17 fold expressions collapse a parameter pack with an operator
directly.

```cpp
#include <iostream>

template <typename... Args>
auto sum(Args... args) {
    return (args + ...);      // unary right fold: args[0] + (args[1] + (args[2] + ...))
}

template <typename... Args>
void printAll(Args... args) {
    ((std::cout << args << " "), ...);   // fold over the comma operator
    std::cout << std::endl;
}

int main() {
    std::cout << sum(1, 2, 3, 4) << std::endl;   // 10
    printAll(1, "two", 3.0, 'x');                 // 1 two 3 x
}
```

`(args + ...)` requires at least one argument for `+` — there's no identity
element the compiler can fall back to for zero arguments. If you need a
zero-argument case, use a binary fold with a seed value: `(0 + ... + args)`.

## SFINAE and `std::enable_if` — picking an overload by trait

**SFINAE** ("Substitution Failure Is Not An Error") means: if substituting a
type into a template produces an invalid expression, the compiler silently
removes that overload from consideration instead of erroring — *as long as
another viable overload exists*. `std::enable_if` weaponizes this to turn
type traits into overload switches.

```cpp
#include <iostream>
#include <type_traits>

// Only participates in overload resolution when T is an integral type.
template <typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
half(T value) {
    return value / 2;                 // integer division
}

// Only participates when T is a floating-point type.
template <typename T>
typename std::enable_if<std::is_floating_point<T>::value, T>::type
half(T value) {
    return value / 2.0;
}

int main() {
    std::cout << half(7) << std::endl;      // 3   -- integral overload
    std::cout << half(7.0) << std::endl;    // 3.5 -- floating-point overload
}
```

When `T` is `int`, `std::is_floating_point<T>::value` is `false`, so
`enable_if<false, T>::type` doesn't exist — that overload is quietly dropped,
not a compile error. Exactly one overload survives for any given `T`.

## `if constexpr` (C++17) — the modern, readable alternative

For branching on a compile-time condition *inside* one function body,
`if constexpr` is almost always clearer than SFINAE overloads:

```cpp
#include <iostream>
#include <type_traits>

template <typename T>
auto describe(const T& value) {
    if constexpr (std::is_integral_v<T>) {
        std::cout << value << " is an integer" << std::endl;
    } else if constexpr (std::is_floating_point_v<T>) {
        std::cout << value << " is a float" << std::endl;
    } else {
        std::cout << "some other type" << std::endl;
    }
}

int main() {
    describe(42);        // 42 is an integer
    describe(3.14);       // 3.14 is a float
    describe("hi");        // some other type
}
```

The crucial difference from an ordinary `if`: the branches **not taken are
discarded before code generation**, not just skipped at runtime. That means
`describe` can safely contain code that would fail to compile for other
types, as long as it's guarded by an `if constexpr` branch that's false for
that instantiation. A runtime `if` would still try to compile every branch
for every `T` and fail.

## Non-type template parameters

Templates aren't limited to types — a compile-time constant (an integer,
`bool`, or pointer/reference with static storage) can be a template
parameter too. `std::array<T, N>` is built exactly this way.

```cpp
#include <iostream>
#include <cstddef>

template <typename T, std::size_t N>
class FixedBuffer {
public:
    constexpr std::size_t size() const { return N; }
    T& operator[](std::size_t i) { return data_[i]; }
    const T& operator[](std::size_t i) const { return data_[i]; }
private:
    T data_[N];   // N is known at compile time -- no heap allocation needed
};

int main() {
    FixedBuffer<int, 4> buf;
    for (std::size_t i = 0; i < buf.size(); ++i) buf[i] = static_cast<int>(i * i);
    for (std::size_t i = 0; i < buf.size(); ++i) std::cout << buf[i] << " ";
    std::cout << std::endl;   // 0 1 4 9
}
```

`FixedBuffer<int, 4>` and `FixedBuffer<int, 8>` are two entirely different,
unrelated types generated from the same source — each with its own fixed-size
array baked in at compile time, no pointer indirection required.

## Cheat sheet

| Technique | Use when |
|-----------|----------|
| Full specialization (`template<> struct X<int>`) | One exact type needs different behaviour |
| Partial specialization (`template<typename T> struct X<T*>`) | A *shape* of types (pointers, pairs, containers) needs different behaviour |
| Variadic templates + recursion | Pre-C++17 codebases, or when each step needs distinct logic |
| Fold expressions `(args op ...)` | C++17+, a single operator applied across a whole pack |
| `std::enable_if` / SFINAE | Choosing between whole overloads based on a trait |
| `if constexpr` | Branching inside one function body based on a trait (prefer this over SFINAE when possible) |
| Non-type template parameter (`template<int N>`) | Baking a compile-time constant (size, flag) into the type itself |

## Traps worth knowing

**Errors surface far from the mistake.** A bad instantiation reports where
the template body fails, often three call layers deep, wrapped in the full
expanded type name. Read template errors from the bottom (the actual
substitution failure) upward, not top to bottom.

**Templates are only compiled when instantiated.** A member function of a
class template that's never called can contain a genuine bug and the
compiler will never see it. Test every code path you rely on, not just the
ones you happen to exercise in `main`.

**Code bloat is real.** Every distinct set of template arguments produces its
own compiled function or class — `sum<int>` and `sum<double>` are separate
machine code, not one generic routine. This is usually a fair trade for the
type safety and speed (no virtual dispatch), but it can inflate binary size
if you instantiate the same template over dozens of unrelated types.

## Exercise

Write a class template `Box<T>` holding one value of type `T`, with a
`describe()` method that prints the value. Then:

1. Add a partial specialization `Box<T*>` that prints the pointed-to value
   (or `"null"` if the pointer is null) instead of the address.
2. Write a variadic function template `makeBoxes(Args... args)` that returns
   a `std::tuple` of `Box<Args>...`, one box per argument, using a fold
   expression or `std::make_tuple`.
3. Add a free function `template <typename T> void printIfNumeric(const T& v)`
   that uses `if constexpr` with `std::is_arithmetic_v<T>` to print the value
   when `T` is numeric and print `"(not numeric)"` otherwise — call it with an
   `int`, a `double`, and a `std::string` to confirm all three paths compile.
