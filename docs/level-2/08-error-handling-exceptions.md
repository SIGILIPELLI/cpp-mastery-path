# 08 · Error Handling with Exceptions

[Level 1 Module 9](../level-1/09-exception-handling.md) covered `try`/`catch`
and the standard exception types. That's enough for small programs. Once a
codebase grows, you need more: exceptions that carry structured data, cleanup
that's guaranteed even mid-throw, and a considered answer to "what state is my
object in after a failure?"

This module covers custom exception classes, RAII-based cleanup, and the
exception safety guarantees that library authors design around.

## Custom exception classes

Standard exceptions carry a string. Often you want the caller to react
programmatically — retry on a timeout, prompt the user on a validation failure
— and parsing `what()` to decide is fragile.

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

// Derive from std::runtime_error to inherit the what() machinery for free
class ValidationError : public std::runtime_error {
public:
    ValidationError(const std::string& field, const std::string& reason)
        : std::runtime_error("Validation failed for '" + field + "': " + reason),
          field(field), reason(reason) {}

    const std::string& getField()  const { return field; }
    const std::string& getReason() const { return reason; }

private:
    std::string field;
    std::string reason;
};

void setAge(int age) {
    if (age < 0)   throw ValidationError("age", "must not be negative");
    if (age > 150) throw ValidationError("age", "unrealistically large");
}

int main() {
    try {
        setAge(-3);
    } catch (const ValidationError& e) {
        std::cout << e.what() << std::endl;                     // full message
        std::cout << "field: " << e.getField() << std::endl;    // structured access
    }
}
// Output:
// Validation failed for 'age': must not be negative
// field: age
```

Deriving from `std::runtime_error` (or `std::logic_error`) is the pragmatic
choice: you pass the message to the base constructor and inherit a correct,
`noexcept`, allocation-safe `what()`. Deriving straight from `std::exception`
means implementing `what()` yourself, and if you store a `std::string` member
you must be careful — copying the exception during propagation could throw.

## Exception hierarchies

Group related errors under a common base so callers can catch at whatever
granularity they need:

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

class DatabaseError : public std::runtime_error {
public:
    explicit DatabaseError(const std::string& msg) : std::runtime_error(msg) {}
};

class ConnectionError : public DatabaseError {
public:
    ConnectionError(const std::string& host, int port)
        : DatabaseError("Cannot connect to " + host + ":" + std::to_string(port)),
          host(host), port(port) {}
    const std::string& getHost() const { return host; }
    bool isRetryable() const { return true; }
private:
    std::string host;
    int port;
};

class QueryError : public DatabaseError {
public:
    QueryError(const std::string& sql, const std::string& msg)
        : DatabaseError("Query failed: " + msg), sql(sql) {}
    const std::string& getSql() const { return sql; }
    bool isRetryable() const { return false; }
private:
    std::string sql;
};

void connect() { throw ConnectionError("db.local", 5432); }

int main() {
    try {
        connect();
    } catch (const ConnectionError& e) {           // most specific first
        std::cout << "Retryable: " << e.what() << std::endl;
    } catch (const QueryError& e) {
        std::cout << "Fix the SQL: " << e.getSql() << std::endl;
    } catch (const DatabaseError& e) {             // any other DB error
        std::cout << "DB error: " << e.what() << std::endl;
    } catch (const std::exception& e) {            // last-resort net
        std::cout << "Unexpected: " << e.what() << std::endl;
    }
}
// Output:
// Retryable: Cannot connect to db.local:5432
```

Catch order matters — handlers are tried top to bottom with no
best-match analysis. Put `catch (const std::exception&)` last, or it swallows
everything below it.

**Always catch by `const&`.** Catching by value slices a derived exception down
to the handler's type (the same slicing problem as
[Module 1](01-oop-deep-dive.md)) and copies it needlessly.

## Rethrowing and adding context

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

void loadConfig(const std::string& path) {
    throw std::runtime_error("file not found");
}

void startup() {
    try {
        loadConfig("app.conf");
    } catch (const std::exception& e) {
        // Wrap with context, preserving the original message
        throw std::runtime_error(std::string("Startup failed: ") + e.what());
    }
}

void logAndPropagate() {
    try {
        startup();
    } catch (...) {
        std::cerr << "logging..." << std::endl;
        throw;          // bare 'throw' RETHROWS the current exception unchanged
    }
}

int main() {
    try {
        logAndPropagate();
    } catch (const std::exception& e) {
        std::cout << e.what() << std::endl;
    }
}
// Output:
// logging...
// Startup failed: file not found
```

A bare `throw;` inside a catch block rethrows the *original* exception object,
preserving its dynamic type. Writing `throw e;` instead would slice it down to
the static type of `e` — a subtle and very common bug.

C++11's `std::nested_exception` (via `std::throw_with_nested`) lets you chain
causes without losing the original object, similar to Java's exception cause
chain.

## RAII: cleanup that cannot be skipped

C++ has no `finally`. It doesn't need one, because destructors of local objects
run during **stack unwinding** — the process of exiting scopes as an exception
propagates.

```cpp
#include <iostream>
#include <fstream>
#include <memory>
#include <stdexcept>

class ScopedTimer {
public:
    ScopedTimer(std::string label) : label(label) {
        std::cout << "[start] " << label << std::endl;
    }
    ~ScopedTimer() {                        // runs on normal exit AND on throw
        std::cout << "[end] " << label << std::endl;
    }
private:
    std::string label;
};

void risky(bool fail) {
    ScopedTimer timer("risky");
    auto buffer = std::make_unique<int[]>(1000);   // freed automatically
    std::ofstream out("temp.txt");                  // closed automatically

    if (fail) throw std::runtime_error("something broke");

    std::cout << "finished normally" << std::endl;
}

int main() {
    try {
        risky(true);
    } catch (const std::exception& e) {
        std::cout << "caught: " << e.what() << std::endl;
    }
}
// Output:
// [start] risky
// [end] risky          <-- destructor ran during unwinding
// caught: something broke
```

Compare that to a manual version with `new`/`delete` and `fclose`: every early
exit needs its own cleanup path, and a `throw` skips all of them. This is why
[smart pointers](06-smart-pointers.md) matter so much for correctness, not
just convenience.

## Never let a destructor throw

```cpp
class Dangerous {
public:
    ~Dangerous() {
        throw std::runtime_error("boom");   // DO NOT DO THIS
    }
};
```

If an exception is already propagating and a destructor throws a second one,
the runtime calls `std::terminate` and your program dies immediately — no
handler runs. Since C++11 destructors are implicitly `noexcept`, so this is
almost always instant termination.

If a destructor performs work that can fail (flushing a buffer, committing a
transaction), catch and log inside it, and additionally expose an explicit
`close()`/`commit()` that callers can invoke to see the error:

```cpp
class Writer {
public:
    void close() {                     // explicit: may throw, caller can handle it
        if (!flushed) { flush(); flushed = true; }
    }
    ~Writer() {
        try { close(); }
        catch (...) { /* log, but never let it escape */ }
    }
private:
    bool flushed = false;
    void flush();
};
```

## Exception safety guarantees

When a function throws, what happened to your data? Library authors classify
this in four levels, and it's worth stating explicitly which one your functions
provide.

| Guarantee | Promise |
|-----------|---------|
| **No-throw** (`noexcept`) | Never throws. Destructors, swaps, and moves should be here. |
| **Strong** | If it throws, state is exactly as before — the operation is all-or-nothing. |
| **Basic** | If it throws, everything is still valid and destructible, but possibly modified. |
| **None** | Anything may be corrupted or leaked. Avoid. |

The **copy-and-swap** idiom is the standard route to the strong guarantee: do
all the risky work on a copy, then swap it in with a `noexcept` operation.

```cpp
#include <vector>
#include <string>
#include <utility>

class Inventory {
public:
    // Basic guarantee only: if the second push throws, the first already landed
    void addPairUnsafe(const std::string& a, const std::string& b) {
        items.push_back(a);
        items.push_back(b);     // may throw -- 'a' is left behind
    }

    // Strong guarantee: build a complete new state, then commit atomically
    void addPair(const std::string& a, const std::string& b) {
        std::vector<std::string> copy = items;   // may throw -- harmless, nothing changed
        copy.push_back(a);
        copy.push_back(b);                        // may throw -- still harmless
        items.swap(copy);                         // noexcept -- the commit point
    }

private:
    std::vector<std::string> items;
};
```

`std::vector::swap` just exchanges internal pointers, so it cannot throw. Every
statement that could throw happens before the commit, so a failure leaves
`items` untouched.

## `noexcept`

```cpp
#include <utility>

class Buffer {
public:
    // Promising noexcept lets std::vector MOVE these during reallocation
    // instead of copying them -- a large performance difference.
    Buffer(Buffer&& other) noexcept
        : data(other.data), size(other.size) {
        other.data = nullptr;
        other.size = 0;
    }

    void swap(Buffer& other) noexcept {
        std::swap(data, other.data);
        std::swap(size, other.size);
    }

private:
    char* data = nullptr;
    std::size_t size = 0;
};
```

`noexcept` is a promise, not a check. If a `noexcept` function throws anyway,
`std::terminate` runs — there is no catching it. Mark a function `noexcept`
only when you are certain, and use it especially on move constructors and
`swap`, because the Standard Library inspects that promise to choose faster
code paths.

## When *not* to use exceptions

Exceptions are for **exceptional** conditions, not routine control flow.
"User typed a letter instead of a number" in an input loop is expected, and a
return value models it better:

```cpp
#include <optional>
#include <string>

// Failure is normal and expected here -- return, don't throw
std::optional<int> tryParseInt(const std::string& text) {
    try {
        std::size_t pos = 0;
        int value = std::stoi(text, &pos);
        if (pos != text.size()) return std::nullopt;   // trailing junk
        return value;
    } catch (const std::exception&) {
        return std::nullopt;
    }
}

int main() {
    if (auto n = tryParseInt("42")) {
        std::cout << *n << std::endl;      // 42
    }
    if (!tryParseInt("abc")) {
        std::cout << "not a number" << std::endl;
    }
}
```

Rough guide: **throw** when the caller probably cannot continue (a config file
is missing, an invariant is violated, a constructor cannot build a valid
object). **Return a value** when failure is a normal outcome the caller will
routinely handle. Constructors are the strongest case for throwing — they have
no return value, so an exception is the only way to refuse to create an invalid
object.

## Exercise

Build an exception hierarchy for a configuration loader.

1. `ConfigError : std::runtime_error` as the base.
2. `MissingKeyError` storing the key name; `TypeMismatchError` storing the key,
   the expected type, and the actual text.
3. A `Config` class wrapping `std::map<std::string, std::string>` with
   `int getInt(const std::string& key) const` that throws `MissingKeyError` if
   the key is absent and `TypeMismatchError` if the value doesn't parse.
4. A `std::optional<int> tryGetInt(...)` alternative that returns `std::nullopt`
   instead of throwing. Write a short comment explaining which you'd expose as
   the primary API and why.
5. Add `void applyAll(const std::map<...>& updates)` that provides the **strong
   guarantee** using copy-and-swap — validate every update against a copy, then
   commit only if all of them pass. Prove it with a test where the third update
   is invalid and the config is verifiably unchanged afterwards.
