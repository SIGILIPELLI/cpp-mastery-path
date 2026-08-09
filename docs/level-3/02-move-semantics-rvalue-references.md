# 02 · Move Semantics & Rvalue References

Every constructor and assignment you've written so far has implicitly
**copied**. Copying a `std::vector<std::string>` with a million entries means
allocating a new buffer and duplicating every string. Often, though, the
source object is a temporary that's about to be destroyed anyway — copying it
is pure waste. Move semantics let the compiler recognize that case and
*steal* the source's internals instead of duplicating them.

## lvalues and rvalues, briefly

Every expression in C++ is either an **lvalue** (has a name, a persistent
address you could take `&` of) or an **rvalue** (a temporary, about to
disappear).

```cpp
int x = 5;        // x is an lvalue -- it has a name and an address
int y = x + 1;    // (x + 1) is an rvalue -- a temporary, gone after this line

std::string makeGreeting() { return "hi"; }

std::string a = makeGreeting();   // the return value is an rvalue -- a temporary
std::string b = a;                // 'a' is an lvalue -- has a name, still needed after
```

This distinction is what move semantics is built on: an rvalue is a value
nobody else can reference again, so it's always safe to cannibalize it.

## Rvalue references — `T&&`

A regular reference `T&` can only bind to an lvalue. `T&&` is a new kind of
reference that binds *only* to rvalues, and it's how a function says "I
promise to only touch things nobody else needs anymore."

```cpp
#include <iostream>

void inspect(const std::string& s) { std::cout << "lvalue ref: " << s << std::endl; }
void inspect(std::string&& s)      { std::cout << "rvalue ref: " << s << std::endl; }

int main() {
    std::string name = "Ada";
    inspect(name);               // name is an lvalue -> calls the T& overload
    inspect("Ada");              // calls T&& -- a string literal converted to a temporary
    inspect(std::string("Bob")); // calls T&& -- an explicit temporary
}
// lvalue ref: Ada
// rvalue ref: Ada
// rvalue ref: Bob
```

Overload resolution picks the `&&` version whenever the argument is a
temporary, and the `&` version whenever it's a named object. This is exactly
how the standard library picks a move constructor over a copy constructor.

## Move constructor and move assignment

A class that owns a resource (heap memory, a file handle, a socket) can
define a **move constructor** and **move assignment operator** that transfer
ownership instead of duplicating it.

```cpp
#include <iostream>
#include <cstring>

class Buffer {
public:
    explicit Buffer(std::size_t size) : size_(size), data_(new int[size]) {
        std::cout << "Allocated " << size_ << " ints" << std::endl;
    }

    // Copy constructor -- duplicates the heap buffer. Expensive.
    Buffer(const Buffer& other) : size_(other.size_), data_(new int[other.size_]) {
        std::memcpy(data_, other.data_, size_ * sizeof(int));
        std::cout << "Copied " << size_ << " ints" << std::endl;
    }

    // Move constructor -- steals the pointer, leaves 'other' empty. Cheap: O(1).
    Buffer(Buffer&& other) noexcept : size_(other.size_), data_(other.data_) {
        other.size_ = 0;
        other.data_ = nullptr;         // 'other' no longer owns anything
        std::cout << "Moved buffer" << std::endl;
    }

    // Move assignment -- release our own resource, then steal the source's.
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = other.data_;
            other.size_ = 0;
            other.data_ = nullptr;
        }
        return *this;
    }

    ~Buffer() { delete[] data_; }

    std::size_t size() const { return size_; }

private:
    std::size_t size_;
    int* data_;
};

Buffer makeBuffer() {
    Buffer b(1000);
    return b;    // moved (or elided entirely) into the caller -- not copied
}

int main() {
    Buffer a(10);
    Buffer b = std::move(a);    // explicit move: 'a' is emptied, 'b' now owns the memory
    std::cout << "a.size()=" << a.size() << " b.size()=" << b.size() << std::endl;

    Buffer c = makeBuffer();    // typically constructed directly in c's storage (RVO)
}
// Allocated 10 ints
// Moved buffer
// a.size()=0 b.size()=1000
// Allocated 1000 ints
```

The move constructor is marked `noexcept` deliberately: `std::vector` checks
this at compile time, and if a type's move constructor might throw, `vector`
falls back to the slower copy constructor when it needs to relocate elements
during a resize — because a throw mid-move would leave the vector in a
half-moved, unrecoverable state. Always mark move operations `noexcept` when
they genuinely cannot throw.

## `std::move` doesn't move anything

This is the single most common misunderstanding. `std::move` is a **cast** —
it does zero work at runtime. It just tells the compiler "treat this named
lvalue as an rvalue," which makes the `&&` overloads eligible.

```cpp
#include <string>
#include <utility>

std::string s = "hello";
std::string t = std::move(s);   // std::move(s) is just static_cast<std::string&&>(s)
// The actual "move" happens inside std::string's move constructor,
// which runs because the cast made it a viable candidate.
```

**After a move, the source is in a "valid but unspecified" state.** For
standard library types this typically means empty, but the exact state isn't
guaranteed beyond "safe to destroy or reassign." Never read from a moved-from
object before reassigning it:

```cpp
std::string s = "hello";
std::string t = std::move(s);
std::cout << s.size() << std::endl;   // legal, but don't rely on a specific value
s = "reset";                          // fine -- reassignment always works
```

## The Rule of Five (and Zero)

If a class needs a custom destructor, it almost certainly needs to think
about all five special member functions together — the **Rule of Five**
(the old Rule of Three, extended for move):

| Function | Purpose |
|----------|---------|
| Destructor | Release the owned resource |
| Copy constructor | Duplicate the resource |
| Copy assignment | Release ours, duplicate theirs |
| Move constructor | Steal the resource |
| Move assignment | Release ours, steal theirs |

Declaring *any one* of these suppresses the compiler's implicitly generated
move constructor/assignment (though copy operations are still generated in
some cases, deprecated since C++11 — don't rely on it). Better yet: follow
the **Rule of Zero**. Store resources in `std::vector`, `std::string`,
`std::unique_ptr`, etc. instead of raw pointers, and let the compiler
generate all five members for free, correctly, because each member already
knows how to move and copy itself.

```cpp
// Rule of Zero: no destructor, no copy/move ops written by hand --
// the compiler generates all five correctly because every member
// (string, vector, unique_ptr) already manages its own resource.
class Report {
public:
    Report(std::string title) : title_(std::move(title)) {}
private:
    std::string title_;
    std::vector<int> scores_;
};
```

## Perfect forwarding — `std::forward` and forwarding references

A function template parameter written `T&&` (not a fixed type's `&&`) is a
**forwarding reference**: it can bind to either an lvalue or an rvalue,
deducing `T` differently in each case. `std::forward<T>` then passes it along
*preserving* whichever it originally was.

```cpp
#include <iostream>
#include <utility>

void handle(const std::string&) { std::cout << "handle(lvalue)" << std::endl; }
void handle(std::string&&)      { std::cout << "handle(rvalue)" << std::endl; }

// T&& here is a forwarding reference, not a plain rvalue reference,
// because T is a template parameter deduced at the call site.
template <typename T>
void relay(T&& arg) {
    handle(std::forward<T>(arg));   // forwards as whatever category it was received as
}

int main() {
    std::string name = "Ada";
    relay(name);              // handle(lvalue) -- name is an lvalue
    relay(std::string("x"));   // handle(rvalue) -- a temporary
}
// handle(lvalue)
// handle(rvalue)
```

Without `std::forward`, `arg` inside `relay` would always be treated as an
lvalue (it has a name!), so the rvalue overload of `handle` would never be
selected — the "moveability" of the original argument would be silently
lost. This is exactly how `std::make_unique`, `std::vector::emplace_back`,
and other factory functions forward constructor arguments without an extra
copy.

## Cheat sheet

| Syntax | Meaning |
|--------|---------|
| `T&` | lvalue reference — binds to named objects |
| `const T&` | binds to anything (lvalue or rvalue), read-only |
| `T&&` (fixed `T`) | rvalue reference — binds only to temporaries/`std::move`d values |
| `T&&` (`T` is a deduced template param) | forwarding reference — binds to either |
| `std::move(x)` | cast `x` to an rvalue reference; moves nothing itself |
| `std::forward<T>(x)` | inside a template, forward `x` preserving its original value category |

## Traps

**Returning `std::move(localVar)` from a function usually hurts.** The
compiler's Return Value Optimization already avoids the copy/move entirely
for a local variable returned by value; wrapping it in `std::move` can
*disable* that optimization and force an actual move instead of zero
operations.

```cpp
Buffer makeBuffer() {
    Buffer b(1000);
    return b;              // best: RVO, likely zero copies/moves at all
    // return std::move(b);   // worse: forces a move, defeats RVO
}
```

**A moved-from object is still a live object.** It will still be destroyed,
and it can still be assigned to — but calling ordinary methods on it before
reassigning is asking for surprises, even if it technically compiles.

## Exercise

Give the `Buffer` class above a `Buffer(const Buffer&) = delete;` (make it
move-only, like `std::unique_ptr`), keeping the move constructor and move
assignment. Confirm `Buffer b2 = b1;` now fails to compile with a clear "use
of deleted function" error, while `Buffer b2 = std::move(b1);` still works.
Then write a `template <typename T> void logAndForward(T&& value)` that
prints whether it received an lvalue or rvalue (hint: `std::is_lvalue_reference<T>`
inside the template — `T` deduces to `U&` for lvalues and `U` for rvalues) and
forwards `value` into a `std::vector<Buffer>::push_back`.
