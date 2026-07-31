# 05 · Operator Overloading

`std::string` supports `+`. `std::vector` supports `[]`. `std::cout` supports
`<<`. None of these are built-in language features for those types — they are
ordinary functions with unusual names, written in the Standard Library using a
mechanism you can use too.

Operator overloading lets your own types read like built-in ones. Used well, a
`Vector2 c = a + b;` is dramatically clearer than `Vector2 c = a.plus(b);`.
Used badly — overloading `+` to mean "send over the network" — it makes code
unreadable. The guiding rule is simple: **only overload an operator when its
conventional meaning is obvious for your type.**

## A worked example: a 2D vector

```cpp
#include <iostream>
#include <cmath>

class Vector2 {
public:
    Vector2(double x = 0.0, double y = 0.0) : x(x), y(y) {}

    // Member operator: the LEFT operand is 'this'
    Vector2 operator+(const Vector2& rhs) const {
        return Vector2(x + rhs.x, y + rhs.y);
    }

    Vector2 operator-(const Vector2& rhs) const {
        return Vector2(x - rhs.x, y - rhs.y);
    }

    // Scalar multiply:  v * 3
    Vector2 operator*(double scalar) const {
        return Vector2(x * scalar, y * scalar);
    }

    // Unary minus -- one operand, no parameters
    Vector2 operator-() const {
        return Vector2(-x, -y);
    }

    // Compound assignment: modifies *this and returns a reference to it
    Vector2& operator+=(const Vector2& rhs) {
        x += rhs.x;
        y += rhs.y;
        return *this;
    }

    double length() const { return std::sqrt(x * x + y * y); }

    double getX() const { return x; }
    double getY() const { return y; }

private:
    double x, y;
};

int main() {
    Vector2 a(1.0, 2.0);
    Vector2 b(3.0, 4.0);

    Vector2 sum = a + b;              // Vector2(4, 6)
    Vector2 diff = b - a;             // Vector2(2, 2)
    Vector2 scaled = a * 3.0;         // Vector2(3, 6)
    Vector2 neg = -a;                 // Vector2(-1, -2)

    a += b;                           // a is now Vector2(4, 6)

    std::cout << sum.getX() << ", " << sum.getY() << std::endl;   // 4, 6
    std::cout << b.length() << std::endl;                          // 5
}
```

Three conventions in that code are worth calling out:

- Binary arithmetic operators are `const` and return a **new object by value** —
  `a + b` must not modify `a`.
- Compound assignment (`+=`) returns `Vector2&`, a reference to the modified
  object, so that `(a += b) += c` works like it does for built-in types.
- Take the right-hand operand as `const&` to avoid a copy.

The idiomatic way to keep them consistent is to implement `+=` first and define
`+` in terms of it:

```cpp
// Free function, outside the class
Vector2 operator+(Vector2 lhs, const Vector2& rhs) {   // lhs BY VALUE -- it's our copy
    lhs += rhs;      // reuse the member operator
    return lhs;
}
```

## Member vs. free function

```cpp
// Member: 'v * 3.0' works, but '3.0 * v' does NOT.
// The left operand must be a Vector2 for a member operator to apply.

// Free function fixes the reversed case:
Vector2 operator*(double scalar, const Vector2& v) {
    return v * scalar;   // delegate to the member version
}

int main() {
    Vector2 v(1.0, 2.0);
    Vector2 a = v * 3.0;   // member operator
    Vector2 b = 3.0 * v;   // free operator -- would not compile without it
}
```

This asymmetry is the main reason to prefer free functions for symmetric
binary operators. A free function treats both operands equally and allows
implicit conversion on *either* side.

| Operator | Implement as |
|----------|--------------|
| `=`, `[]`, `()`, `->` | **must** be a member |
| `+=`, `-=`, `*=`, `++`, `--` | member (they modify the left operand) |
| `+`, `-`, `*`, `/`, `==`, `<` | free function (symmetric operands) |
| `<<`, `>>` (streams) | free function (left operand is the stream) |

## Stream operators

```cpp
#include <iostream>
#include <sstream>
#include <string>

class Vector2 {
public:
    Vector2(double x = 0, double y = 0) : x(x), y(y) {}
    double x, y;
};

// Must be a free function: the left operand is std::ostream, not Vector2.
// Return the stream by reference so calls chain: cout << a << b << '\n';
std::ostream& operator<<(std::ostream& os, const Vector2& v) {
    os << "(" << v.x << ", " << v.y << ")";
    return os;
}

// Input operator: takes a NON-const reference to fill in
std::istream& operator>>(std::istream& is, Vector2& v) {
    is >> v.x >> v.y;
    return is;
}

int main() {
    Vector2 a(1.5, 2.5);
    std::cout << "a = " << a << std::endl;      // a = (1.5, 2.5)

    std::istringstream input("7 8");
    Vector2 b;
    input >> b;
    std::cout << "b = " << b << std::endl;      // b = (7, 8)
}
// Output:
// a = (1.5, 2.5)
// b = (7, 8)
```

Two non-negotiables: return `std::ostream&` (not `void`, or chaining breaks)
and never write `std::endl` inside `operator<<` — leave line breaks to the
caller.

If your class has private members, the stream operator needs access. Either
add public getters, or declare it a `friend`:

```cpp
class Vector2 {
    double x, y;
    friend std::ostream& operator<<(std::ostream&, const Vector2&);   // grants access
};
```

## Comparison operators

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

class Version {
public:
    Version(int major, int minor) : major(major), minor(minor) {}

    friend bool operator==(const Version& a, const Version& b) {
        return a.major == b.major && a.minor == b.minor;
    }
    friend bool operator!=(const Version& a, const Version& b) {
        return !(a == b);           // define in terms of == -- never duplicate logic
    }
    friend bool operator<(const Version& a, const Version& b) {
        if (a.major != b.major) return a.major < b.major;
        return a.minor < b.minor;
    }
    friend bool operator>(const Version& a, const Version& b)  { return b < a; }
    friend bool operator<=(const Version& a, const Version& b) { return !(b < a); }
    friend bool operator>=(const Version& a, const Version& b) { return !(a < b); }

    friend std::ostream& operator<<(std::ostream& os, const Version& v) {
        return os << v.major << "." << v.minor;
    }

private:
    int major, minor;
};

int main() {
    std::vector<Version> versions{{2, 1}, {1, 9}, {2, 0}};
    std::sort(versions.begin(), versions.end());   // uses operator<

    for (const auto& v : versions) std::cout << v << ' ';
    std::cout << std::endl;   // 1.9 2.0 2.1
}
```

Defining `operator<` is what makes your type usable as a `std::map` key, a
`std::set` element, or a `std::sort` target with no extra comparator. Write
`<` and `==` honestly, then derive the other four from them — that guarantees
they stay mutually consistent.

C++20 collapses all six into one **three-way comparison** ("spaceship"):

```cpp
#include <compare>

class Version {
    int major, minor;
public:
    // Generates ==, !=, <, <=, >, >= automatically, comparing members in order
    auto operator<=>(const Version&) const = default;
    bool operator==(const Version&) const = default;
};
```

## Subscript and function-call operators

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>

class Grid {
public:
    Grid(int rows, int cols) : rows(rows), cols(cols), data(rows * cols, 0) {}

    // Non-const version returns a MUTABLE reference: grid(1, 2) = 5;
    int& operator()(int r, int c) {
        if (r < 0 || r >= rows || c < 0 || c >= cols) {
            throw std::out_of_range("Grid index out of range");
        }
        return data[r * cols + c];
    }

    // const version returns a read-only reference -- needed for const Grid objects
    const int& operator()(int r, int c) const {
        return data[r * cols + c];
    }

private:
    int rows, cols;
    std::vector<int> data;
};

// A "functor" -- an object that behaves like a function
class MultiplyBy {
public:
    MultiplyBy(int factor) : factor(factor) {}
    int operator()(int x) const { return x * factor; }   // callable state
private:
    int factor;
};

int main() {
    Grid g(3, 3);
    g(1, 1) = 7;                       // operator() returning int& makes this assignable
    std::cout << g(1, 1) << std::endl; // 7

    MultiplyBy triple(3);
    std::cout << triple(5) << std::endl;   // 15
}
```

`operator()` can take any number of arguments, which is why it's the natural
choice for a 2D index — `operator[]` accepts only one argument before C++23.
An object with `operator()` is a **functor**, and it's exactly what a lambda
compiles into.

Note the const/non-const pair. Without the `const` overload, you couldn't read
from a `const Grid&` at all. Providing both is standard practice for any
accessor that returns a reference.

## Operators you should *not* overload

- `&&`, `||` — overloading them destroys short-circuit evaluation. Both
  operands get evaluated, always. Almost never worth it.
- `,` (comma) — same problem, plus nobody expects it.
- `&` (address-of) — breaks generic code that legitimately takes addresses.
- `->` — legitimate for smart pointers and iterators, confusing anywhere else.

And you simply cannot overload `.`, `.*`, `::`, `?:`, or `sizeof`.

## Cheat sheet

| Operator | Typical signature | Returns |
|----------|-------------------|---------|
| `a + b` | `T operator+(const T&, const T&)` (free) | new value |
| `a += b` | `T& operator+=(const T&)` (member) | `*this` |
| `-a` | `T operator-() const` (member) | new value |
| `a == b` | `bool operator==(const T&, const T&)` (free) | `bool` |
| `a < b` | `bool operator<(const T&, const T&)` (free) | `bool` |
| `os << a` | `std::ostream& operator<<(std::ostream&, const T&)` (free) | the stream |
| `a[i]` | `V& operator[](std::size_t)` + `const` version (member) | reference |
| `a(x, y)` | `R operator()(X, Y)` (member) | anything |
| `++a` | `T& operator++()` (member) | `*this` |
| `a++` | `T operator++(int)` (member, dummy `int` param) | copy of the old value |

## Exercise

Write a `Money` class storing an amount in integer cents (never use `double`
for currency — rounding error accumulates). Give it:

- A constructor taking dollars and cents
- `operator+`, `operator-`, and `operator+=` as appropriate free/member functions
- `operator*(int quantity)` and the reversed `operator*(int, const Money&)`
- All six comparison operators, derived from `==` and `<`
- `operator<<` printing `$12.50` format (watch the zero-padding on cents)

Then put several `Money` values in a `std::vector`, `std::sort` them, and use
`std::accumulate` with `Money{0,0}` as the initial value to total them —
verifying that your operators integrate cleanly with
[the STL algorithms from Module 4](04-stl-algorithms-iterators.md).
