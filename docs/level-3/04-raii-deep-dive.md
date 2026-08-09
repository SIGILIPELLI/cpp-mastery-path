# 04 · RAII Deep Dive

[Smart pointers](../level-2/06-smart-pointers.md) and
[`std::lock_guard`](03-concurrency.md) are both instances of one idiom:
**Resource Acquisition Is Initialization**. Bind a resource's lifetime to an
object's lifetime — acquire it in the constructor, release it in the
destructor — and the language's own scope rules guarantee cleanup, even when
an exception unwinds the stack. This module generalizes RAII beyond memory
and mutexes to *any* resource, and covers the exception-safety guarantees it
enables.

## Why manual cleanup fails

```cpp
#include <cstdio>
#include <stdexcept>

void processFile(const char* path, bool shouldFail) {
    FILE* f = std::fopen(path, "r");
    if (!f) throw std::runtime_error("cannot open file");

    if (shouldFail) {
        throw std::runtime_error("something went wrong mid-processing");
        // fclose(f) below is NEVER reached -- the file handle leaks.
    }

    std::fclose(f);
}
```

Every early return, every exception thrown between acquisition and release,
is a leak waiting to happen. Wrapping every function in `try { ... } finally
{ cleanup(); }` (a keyword C++ doesn't even have) is exhausting and easy to
get wrong once a function has several resources or several exit points.

## A minimal RAII wrapper

```cpp
#include <cstdio>
#include <stdexcept>
#include <utility>

class FileHandle {
public:
    explicit FileHandle(const char* path, const char* mode) : file_(std::fopen(path, mode)) {
        if (!file_) throw std::runtime_error("failed to open file");
    }

    ~FileHandle() {
        if (file_) std::fclose(file_);   // ALWAYS runs -- normal return or exception
    }

    // Resources that aren't naturally shareable should be move-only:
    // deleting copy prevents two FileHandles from both fclose()-ing the same FILE*.
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    FileHandle(FileHandle&& other) noexcept : file_(other.file_) { other.file_ = nullptr; }
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            if (file_) std::fclose(file_);
            file_ = other.file_;
            other.file_ = nullptr;
        }
        return *this;
    }

    FILE* get() const { return file_; }

private:
    FILE* file_;
};

void processFile(const char* path, bool shouldFail) {
    FileHandle f(path, "r");            // acquired here

    if (shouldFail) {
        throw std::runtime_error("something went wrong mid-processing");
        // f's destructor STILL runs during stack unwinding -- file closed, guaranteed.
    }

    // ... use f.get() ...
}   // f's destructor runs here on the normal path
```

Notice the shape: this is the exact pattern `std::unique_ptr` uses for
memory, `std::lock_guard` uses for mutexes, and `std::ifstream` uses for
files internally. Once you've written one RAII wrapper, you recognize the
idiom everywhere in the standard library.

## Scope guards for one-off cleanup

Writing a whole class for a resource you only clean up in one place is
overkill. A small "run this lambda on scope exit" wrapper covers that case
generically:

```cpp
#include <iostream>
#include <functional>
#include <utility>

class ScopeGuard {
public:
    explicit ScopeGuard(std::function<void()> onExit) : onExit_(std::move(onExit)) {}
    ~ScopeGuard() { if (onExit_) onExit_(); }

    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;

    void dismiss() { onExit_ = nullptr; }   // cancel cleanup, e.g. after a successful commit

private:
    std::function<void()> onExit_;
};

void demo(bool commit) {
    std::cout << "acquiring temp resource" << std::endl;
    ScopeGuard cleanup([] { std::cout << "rolling back temp resource" << std::endl; });

    if (commit) {
        std::cout << "committed successfully" << std::endl;
        cleanup.dismiss();   // success path: skip the rollback
    }
}   // if not dismissed, cleanup's destructor prints the rollback message here

int main() {
    demo(true);
    std::cout << "---" << std::endl;
    demo(false);
}
// acquiring temp resource
// committed successfully
// ---
// acquiring temp resource
// rolling back temp resource
```

This "acquire, then guard a rollback that's cancelled on success" shape is
extremely common for things like temporary files, database transactions, or
partially-built objects — commit cancels the guard, any early exit (including
an exception) lets it fire.

## The three exception-safety guarantees

RAII is what makes these guarantees achievable in practice, not just
theory.

| Guarantee | Promise | Example |
|-----------|---------|---------|
| **Basic** | No leaks, no corruption; object is left in *some* valid state | `std::vector::push_back` leaves the vector valid even if the value's copy throws |
| **Strong** | Operation either fully succeeds or has *no visible effect* — like a transaction | `std::vector::push_back` when reallocation is needed: old buffer is untouched if a move/copy throws mid-way |
| **Nothrow** | The operation is guaranteed never to throw | `std::vector::pop_back`, `swap`, most destructors, moves marked `noexcept` |

Destructors in particular should almost never throw. If a destructor throws
while the stack is already unwinding from another exception, the program
calls `std::terminate()` immediately — there's no way to have two exceptions
in flight at once. This is why `~FileHandle()` above doesn't check
`fclose`'s return value with a `throw`; cleanup code swallows its own
errors or logs them, it doesn't propagate them.

## Rule of Zero, revisited

The best RAII code is code you don't have to write by hand at all:

```cpp
#include <memory>
#include <string>
#include <vector>

// No destructor, no copy/move operations written -- and none needed.
// unique_ptr, string, and vector each already manage their own resource
// correctly, so the compiler-generated destructor/copy/move for Report
// just calls each member's, in order. This is RAII composing for free.
class Report {
public:
    Report(std::string title) : title_(std::move(title)), data_(std::make_unique<std::vector<int>>()) {}
private:
    std::string title_;
    std::unique_ptr<std::vector<int>> data_;
};
```

Reach for a hand-written destructor (and therefore the full Rule of Five)
only when your class directly owns a *non-RAII* resource — a raw `FILE*`, an
OS handle, a socket descriptor. Everywhere else, compose existing RAII types
and let the compiler write the boilerplate.

## Cheat sheet

| Resource | RAII wrapper |
|----------|--------------|
| Heap memory | `std::unique_ptr` / `std::shared_ptr` |
| Mutex lock | `std::lock_guard` / `std::unique_lock` / `std::scoped_lock` |
| File handle | `std::fstream` family, or a custom wrapper around `FILE*` |
| One-off cleanup action | A small `ScopeGuard` with a cancellable lambda |
| A group of resources | Compose the above as data members (Rule of Zero) |

## Traps

**A `throw` inside a destructor is close to unforgivable.** If it fires
during unwinding from another exception, `std::terminate()` ends the program
with no further recovery possible.

**Freeing a resource twice** is as bad as never freeing it. Move-only RAII
types (like `FileHandle` above) must null out the source's handle in their
move constructor/assignment, or both the moved-from and moved-to objects will
try to release the same resource in their destructors.

**RAII protects against exceptions and early returns — it doesn't replace
validating results.** A destructor still shouldn't ignore an error signal
silently; log it, or provide an explicit `close()`/`commit()` method that can
report failure, with the destructor as the guaranteed fallback.

## Exercise

Write a `MutexUnlocker` RAII class that's the mirror image of `lock_guard`:
it takes an *already-locked* `std::unique_lock<std::mutex>&`, unlocks it in
its constructor, and re-locks it in its destructor — useful for temporarily
releasing a lock around a slow, non-thread-sensitive operation (like a
network call) inside an otherwise-locked function. Then extend the
`ScopeGuard` above into a tiny `TransactionGuard` used around three
sequential steps (e.g. printing "step 1/2/3 committed"); trigger a simulated
failure after step 2 and confirm the guard's rollback message fires while
steps 1–2's "commits" do not get undone individually — only the guard's own
single rollback action runs.
