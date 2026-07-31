# 06 · Smart Pointers

[Level 1 Module 8](../level-1/08-references-pointers.md) introduced raw
pointers and `new`/`delete`. Manual memory management is where most C++ bugs
come from: forget a `delete` and you leak; `delete` twice and you corrupt the
heap; return early or throw between `new` and `delete` and you leak again,
silently.

Smart pointers solve this by making ownership a property of the *type*. A
smart pointer is a small object that owns a heap allocation and frees it in
its destructor. Because destructors run automatically when a scope ends —
including when an exception unwinds the stack — the cleanup simply cannot be
skipped. This is **RAII** (Resource Acquisition Is Initialization), and it is
the single most important idiom in C++.

Modern guidance: **you should almost never write `new` or `delete` in
application code.**

## The problem, concretely

```cpp
#include <stdexcept>

void leaky(bool fail) {
    int* data = new int[1000];

    if (fail) {
        throw std::runtime_error("boom");   // LEAK: delete[] never runs
    }

    delete[] data;   // only reached on the happy path
}
```

Adding a `try/catch` around every allocation is exhausting and easy to get
wrong. Adding an early `return` later reintroduces the bug. The fix is to stop
managing lifetime by hand.

## `std::unique_ptr` — exclusive ownership

```cpp
#include <iostream>
#include <memory>
#include <string>

class Resource {
public:
    Resource(std::string name) : name(name) {
        std::cout << "Acquired " << name << std::endl;
    }
    ~Resource() {
        std::cout << "Released " << name << std::endl;
    }
    void use() const { std::cout << "Using " << name << std::endl; }
private:
    std::string name;
};

int main() {
    {
        std::unique_ptr<Resource> r = std::make_unique<Resource>("file.txt");
        r->use();               // -> works exactly like a raw pointer
        (*r).use();             // so does *
        if (r) std::cout << "r is non-null" << std::endl;
    }   // <-- destructor runs HERE, automatically

    std::cout << "after scope" << std::endl;
}
// Output:
// Acquired file.txt
// Using file.txt
// r is non-null
// Released file.txt
// after scope
```

`std::make_unique<T>(args...)` allocates a `T`, forwards `args` to its
constructor, and wraps it. Prefer it over `std::unique_ptr<T>(new T(...))`:
it's shorter, mentions the type once, and is exception-safe in argument lists.

A `unique_ptr` is **move-only** — copying it is a compile error, which is
precisely the point:

```cpp
#include <memory>
#include <utility>

std::unique_ptr<Resource> a = std::make_unique<Resource>("A");
// std::unique_ptr<Resource> b = a;              // compile error: copy deleted
std::unique_ptr<Resource> b = std::move(a);      // OK: ownership transfers
// 'a' is now null; 'b' owns the resource

if (!a) { /* true -- a was emptied by the move */ }
```

Because ownership can never be duplicated, "who deletes this?" always has
exactly one answer. A `unique_ptr` is the same size as a raw pointer and, with
optimisation on, generates the same machine code. It is free.

Passing it around:

```cpp
void observe(const Resource& r);              // doesn't own -- takes a reference
void observePtr(const Resource* r);           // doesn't own, may be null
void consume(std::unique_ptr<Resource> r);    // TAKES ownership -- caller must move
std::unique_ptr<Resource> produce();          // GIVES ownership to the caller
```

The signature documents the ownership contract. A function that merely reads
an object should take `const T&`, never `const std::unique_ptr<T>&` — that
would needlessly restrict callers to one storage strategy.

## Returning and storing unique_ptr

```cpp
#include <memory>
#include <vector>
#include <string>

class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    Circle(double r) : r(r) {}
    double area() const override { return 3.14159265 * r * r; }
private:
    double r;
};

std::unique_ptr<Shape> makeShape(const std::string& kind) {
    if (kind == "circle") return std::make_unique<Circle>(2.0);
    return nullptr;
}

int main() {
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(makeShape("circle"));
    shapes.push_back(std::make_unique<Circle>(1.0));

    for (const auto& s : shapes) {
        if (s) std::cout << s->area() << std::endl;
    }
}   // every Shape is destroyed here, through the virtual destructor
```

This is the standard way to hold a polymorphic collection — and it's why
[Module 1's](01-oop-deep-dive.md) virtual destructor rule matters: without
`virtual ~Shape()`, `unique_ptr<Shape>` would destroy only the base part.

## `std::shared_ptr` — shared ownership

```cpp
#include <iostream>
#include <memory>

int main() {
    std::shared_ptr<Resource> a = std::make_shared<Resource>("shared.db");
    std::cout << a.use_count() << std::endl;   // 1

    {
        std::shared_ptr<Resource> b = a;        // copy is ALLOWED; refcount++
        std::cout << a.use_count() << std::endl;   // 2
        b->use();
    }   // b dies, refcount--

    std::cout << a.use_count() << std::endl;   // 1
}   // refcount hits 0 -> Resource destroyed here
// Output:
// Acquired shared.db
// 1
// 2
// Using shared.db
// 1
// Released shared.db
```

`shared_ptr` keeps a **reference count** in a separate heap block (the control
block). Each copy increments it; each destruction decrements it; the object is
destroyed when it hits zero. `std::make_shared` allocates the object and the
control block together in one allocation, which is both faster and more
cache-friendly than `std::shared_ptr<T>(new T)`.

That bookkeeping is not free. The refcount updates are **atomic**, so they
work across threads but cost more than a plain increment. A `shared_ptr` is
also twice the size of a raw pointer. **Reach for `unique_ptr` first** and
switch to `shared_ptr` only when ownership genuinely is shared and you can't
say which owner outlives the others.

## The cycle trap and `std::weak_ptr`

```cpp
#include <iostream>
#include <memory>

struct Node {
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> prev;      // BUG: creates a cycle
    ~Node() { std::cout << "Node destroyed" << std::endl; }
};

int main() {
    auto a = std::make_shared<Node>();
    auto b = std::make_shared<Node>();
    a->next = b;
    b->prev = a;      // a and b now each hold a reference to the other
}
// Output: (nothing) -- neither refcount ever reaches zero. Both nodes LEAK.
```

Reference counting cannot break cycles. The fix is `std::weak_ptr`: a
non-owning observer that does not affect the count.

```cpp
#include <iostream>
#include <memory>

struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;        // weak: observes without owning
    ~Node() { std::cout << "Node destroyed" << std::endl; }
};

int main() {
    auto a = std::make_shared<Node>();
    auto b = std::make_shared<Node>();
    a->next = b;
    b->prev = a;

    // To use a weak_ptr you must lock() it into a temporary shared_ptr,
    // which atomically checks whether the object is still alive.
    if (std::shared_ptr<Node> parent = b->prev.lock()) {
        std::cout << "parent still alive" << std::endl;
    } else {
        std::cout << "parent is gone" << std::endl;
    }
}
// Output:
// parent still alive
// Node destroyed
// Node destroyed
```

The general pattern: **owners hold `shared_ptr` downward, observers hold
`weak_ptr` upward.** A tree node owns its children and weakly references its
parent; a cache holds `weak_ptr`s so it never keeps dead entries alive.

## Traps worth memorising

**Never build two smart pointers from the same raw pointer.**

```cpp
Resource* raw = new Resource("oops");
std::shared_ptr<Resource> a(raw);
std::shared_ptr<Resource> b(raw);   // separate control block -- DOUBLE DELETE
```

Both think they're the sole owner group. Use `std::shared_ptr<Resource> b = a;`
instead. The wider rule: hand a raw pointer to a smart pointer exactly once,
right where it's created — which is what `make_unique`/`make_shared` enforce.

**`get()` returns a non-owning raw pointer. Never `delete` it.**

```cpp
auto p = std::make_unique<Resource>("x");
Resource* raw = p.get();     // fine for passing to a legacy C API
// delete raw;               // catastrophic -- unique_ptr will delete it again
```

**Don't let a raw pointer outlive its owner.**

```cpp
Resource* dangling = nullptr;
{
    auto p = std::make_unique<Resource>("temp");
    dangling = p.get();
}   // p destroyed, memory freed
// dangling->use();   // undefined behaviour -- classic dangling pointer
```

**Arrays need the array form**, though `std::vector` is almost always better:

```cpp
std::unique_ptr<int[]> arr = std::make_unique<int[]>(100);   // calls delete[]
arr[0] = 42;
std::vector<int> better(100);                                 // prefer this
```

## Choosing

| Use case | Type |
|----------|------|
| One owner, clear lifetime (the default) | `std::unique_ptr<T>` |
| Genuinely shared ownership, unknown last user | `std::shared_ptr<T>` |
| Observe something owned elsewhere, may be dead | `std::weak_ptr<T>` |
| Observe something guaranteed alive, never null | `T&` |
| Observe something guaranteed alive, may be null | `const T*` (raw, non-owning) |
| A fixed-size buffer of values | `std::vector<T>` / `std::array<T,N>` |

| Operation | `unique_ptr` | `shared_ptr` |
|-----------|--------------|--------------|
| Copyable | no (move only) | yes |
| Size | same as raw pointer | 2 pointers |
| Overhead | zero | atomic refcount + control block |
| Create with | `std::make_unique<T>(...)` | `std::make_shared<T>(...)` |
| Release ownership | `.release()` | (not applicable) |
| Replace contents | `.reset(newPtr)` | `.reset(newPtr)` |
| Check emptiness | `if (p)` | `if (p)` |

## Exercise

Model a small file system tree. A `Directory` owns a
`std::vector<std::shared_ptr<Entry>>` of children, and every `Entry` holds a
`std::weak_ptr<Directory>` back to its parent. `Entry` is an abstract base with
`virtual std::string path() const` — implemented by walking `parent.lock()`
upward until it returns null — and a virtual destructor.

Print a destructor message from each class, build a two-level tree in `main`,
print each entry's full path, and confirm from the output that every object is
destroyed exactly once when the tree goes out of scope.

Then change the parent link from `weak_ptr` to `shared_ptr`, rerun, and watch
the destructor messages disappear. That silence is a memory leak — the exact
thing `weak_ptr` exists to prevent.
