# 04 · Systems Programming Patterns

C++ is used where the machine shows through: kernels, databases, game engines,
trading systems, embedded firmware. At that level the abstractions you've used
so far are still available, but you also have to reason about the *physical*
program — where bytes sit, how wide a cache line is, which pointer casts are
undefined behaviour, and how a resource that isn't memory gets cleaned up.

This module collects the patterns that recur in that kind of code.

## Memory layout and alignment

A struct is not the sum of its members. The compiler inserts padding so every
member sits at an address divisible by its alignment requirement.

```cpp
#include <cstddef>
#include <iostream>

struct Padded { char a; double b; char c; };   // 1 + 7 pad + 8 + 1 + 7 pad
struct Packed { double b; char a; char c; };   // 8 + 1 + 1 + 6 pad

int main() {
    std::cout << "sizeof(Padded) = " << sizeof(Padded)
              << ", sizeof(Packed) = " << sizeof(Packed) << "\n";
    std::cout << "offsetof b in Padded = " << offsetof(Padded, b)
              << ", in Packed = " << offsetof(Packed, b) << "\n";
    std::cout << "alignof(double) = " << alignof(double) << "\n";
}
```

```text
sizeof(Padded) = 24, sizeof(Packed) = 16
offsetof b in Padded = 8, in Packed = 0
alignof(double) = 8
```

Same three members, same types, **50% more memory** — purely from declaration
order. In an array of ten million of these that is 80 MB of nothing. The rule of
thumb: declare members in *descending order of alignment* (doubles and pointers
first, then ints, then shorts, then chars, then bools). It costs nothing and is
occasionally worth a lot.

`alignas(64)` forces a type or member onto a cache-line boundary — the standard
fix for **false sharing**, where two threads write to different variables that
happen to live in the same 64-byte line and end up ping-ponging that line
between cores.

## Data-oriented design: AoS vs SoA

The layout question that matters most is not padding but *what a cache line
brings with it*. A CPU never loads one float; it loads the 64-byte line
containing it.

```cpp
struct ParticleAoS { float x, y, z, vx, vy, vz, mass; char name[96]; };  // 124 bytes

struct ParticlesSoA {          // structure of arrays: one vector per field
    std::vector<float> x, y, z;
};
```

Summing only the `x` field of two million particles, ten times:

```text
sizeof(ParticleAoS) = 124 bytes
AoS sum of x: 45 ms
SoA sum of x: 19 ms
equal: true
```

Identical arithmetic, 2.4x apart. In the array-of-structs version, each 64-byte
cache line delivers *one* useful float and 60 bytes of name and velocity the
loop never touches. In the struct-of-arrays version every line is 16 useful
floats, and the hardware prefetcher and the vectorizer both work.

This is the whole idea behind data-oriented design: lay data out to match the
access pattern of the hot loop, not to match the conceptual object model. It is
also a trade — SoA makes "process one whole particle" slower and the code
clumsier — so apply it to the hot loop you actually
[profiled](09-performance-at-scale.md), not everywhere.

## Custom allocators: the arena

`new`/`delete` are general-purpose, thread-safe, and comparatively slow. When
you allocate thousands of short-lived objects with a shared lifetime — one
frame, one request, one parse — an **arena** (bump allocator) replaces the whole
problem with a pointer increment.

```cpp
#include <cstddef>
#include <memory>
#include <new>

class Arena {
public:
    explicit Arena(std::size_t bytes)
        : buffer_(new std::byte[bytes]), capacity_(bytes) {}

    void* allocate(std::size_t bytes, std::size_t align) {
        std::size_t current = reinterpret_cast<std::size_t>(buffer_.get()) + offset_;
        std::size_t aligned = (current + align - 1) & ~(align - 1);   // round up
        std::size_t padding = aligned - current;
        if (offset_ + padding + bytes > capacity_) throw std::bad_alloc{};
        offset_ += padding + bytes;
        return reinterpret_cast<void*>(aligned);
    }

    void reset() { offset_ = 0; }          // frees EVERYTHING in O(1)
    std::size_t used() const { return offset_; }

private:
    std::unique_ptr<std::byte[]> buffer_;
    std::size_t capacity_;
    std::size_t offset_ = 0;
};
```

```text
arena used 16 bytes for 1 char + 1 double
double is aligned: true
after reset, used = 0
```

The 16 bytes are 1 byte for the `char`, 7 bytes of alignment padding, and 8 for
the `double` — the allocator is doing the same rounding the compiler does for
struct members, and `(current + align - 1) & ~(align - 1)` is the standard
idiom for it (it only works because alignments are powers of two).

`reset()` is why arenas exist: deallocation is a single assignment, no matter
how many objects were allocated. The constraint is equally stark — you cannot
free one object, and any type with a non-trivial destructor must be destroyed
explicitly before the reset. C++17's `std::pmr::monotonic_buffer_resource` is
this exact structure, standardized, and `std::pmr::vector` etc. plug into it:

```cpp
std::byte stack[8192];
std::pmr::monotonic_buffer_resource pool{stack, sizeof stack};
std::pmr::vector<int> v{&pool};    // allocates from the stack buffer, never from the heap
```

## Type punning without undefined behaviour

Reading a `float`'s bits as a `uint32_t` by casting the pointer violates the
**strict aliasing rule** — the compiler is permitted to assume a `float*` and a
`uint32_t*` never refer to the same object, and `-O2` will act on that
assumption. The portable, zero-cost answer is `std::memcpy`:

```cpp
float f = 1.0f;
std::uint32_t bits;
std::memcpy(&bits, &f, sizeof bits);      // optimizes to a register move
std::cout << "1.0f bit pattern = 0x" << std::hex << bits << "\n";
```

```text
1.0f bit pattern = 0x3f800000
```

Every mainstream compiler recognizes this `memcpy` and emits no call at all.
C++20 adds `std::bit_cast<std::uint32_t>(f)`, which is the same thing as a
`constexpr` expression.

## Endianness

```cpp
std::uint32_t v = 0x01020304;
unsigned char b[4];
std::memcpy(b, &v, 4);
// b[0] == 0x04 on little-endian, 0x01 on big-endian
```

```text
first byte of 0x01020304 in memory = 0x4 (little endian)
```

This matters the moment bytes leave the process — a file format, a network
protocol, shared memory between architectures. The discipline is: **pick a
byte order for the wire, and convert explicitly at the boundary** with
`htonl`/`ntohl` or your own shift-and-mask serializer. Never `memcpy` a struct
straight onto a socket; you are exporting both your endianness and your padding.
C++20 gives you `std::endian::native` to check at compile time.

## RAII for non-memory resources

Smart pointers handle memory. File descriptors, sockets, mutex handles, GPU
buffers and `mmap` regions need the same treatment, and the pattern is
identical:

```cpp
class FileDescriptor {
public:
    explicit FileDescriptor(int fd) : fd_(fd) {}
    ~FileDescriptor() { if (fd_ >= 0) ::close(fd_); }

    FileDescriptor(const FileDescriptor&) = delete;
    FileDescriptor& operator=(const FileDescriptor&) = delete;

    FileDescriptor(FileDescriptor&& other) noexcept
        : fd_(std::exchange(other.fd_, -1)) {}          // steal, leave a safe -1
    FileDescriptor& operator=(FileDescriptor&& other) noexcept {
        if (this != &other) {
            if (fd_ >= 0) ::close(fd_);
            fd_ = std::exchange(other.fd_, -1);
        }
        return *this;
    }

    int get() const { return fd_; }
private:
    int fd_;
};
```

`std::exchange(other.fd_, -1)` is the move idiom in one expression: return the
old value, install the sentinel. Without it the moved-from object still holds
the descriptor and closes it twice — the resource equivalent of a double free.
See [Level 3 Module 4](../level-3/04-raii-deep-dive.md) for the full rule of
five treatment.

## PIMPL — compilation firewall

A member variable's type must be complete in the header, so every client
recompiles when a private detail changes. Hiding the state behind an opaque
pointer breaks that dependency:

```cpp
// widget.h -- no implementation headers needed here
class Widget {
public:
    Widget();
    ~Widget();                              // MUST be declared, defined in the .cpp
    Widget(Widget&&) noexcept;
    Widget& operator=(Widget&&) noexcept;
    void draw() const;
private:
    struct Impl;                            // declared, not defined
    std::unique_ptr<Impl> impl_;
};
```

The cost is one heap allocation and one indirection per object; the benefit is
that changing `Impl` recompiles one `.cpp` instead of the entire project. Use it
on stable public interfaces of large libraries, not on hot little value types.

## Bit manipulation (C++20 `<bit>`)

```cpp
#include <bit>
std::popcount(0b1011u);        // 3   -- set bits
std::countl_zero(1u);          // 31  -- leading zeros
std::has_single_bit(64u);      // true -- power of two?
std::bit_ceil(100u);           // 128 -- next power of two, for hash table sizing
std::rotl(0x12345678u, 8);     // rotate left
```

These compile to single CPU instructions (`popcnt`, `lzcnt`) where available and
to correct portable fallbacks where not — strictly better than the hand-rolled
bit tricks they replace.

## Cheat sheet

| Tool | Purpose |
|------|---------|
| `alignof(T)` / `alignas(N)` | Query / force alignment; `alignas(64)` kills false sharing |
| `offsetof(S, m)` | Byte offset of a member — check your padding |
| Descending-alignment member order | Free struct-size reduction |
| Struct-of-arrays | Make every cache line useful in the hot loop |
| `std::memcpy` / `std::bit_cast` | Type punning without breaking strict aliasing |
| `std::pmr::monotonic_buffer_resource` | Standard arena; pairs with `std::pmr::vector` |
| `operator new` / `delete` overloads | Per-class allocation strategy |
| `std::exchange(x, sentinel)` | The one-line move idiom for raw handles |
| PIMPL (`unique_ptr<Impl>`) | Break header dependencies, stabilize ABI |
| `std::popcount` / `bit_ceil` / `rotl` | Hardware bit operations, portably |
| `std::endian::native` | Compile-time byte-order check |
| `-fsanitize=address,undefined` | Catch the mistakes this module's techniques enable |

## Traps

**Strict aliasing violations survive testing and die in production.** Casting
`float*` to `int*` and dereferencing is UB; it works at `-O0`, works at `-O2`
until an inlining decision changes, and then silently computes garbage. Use
`memcpy` or `bit_cast`, always. `-fno-strict-aliasing` is a workaround, not a
fix.

**Misaligned access is UB even on forgiving hardware.** `reinterpret_cast`ing a
`char*` from the middle of a buffer to a `double*` happens to work on x86 and
faults on some ARM configurations — and UBSan flags it on all of them. Copy the
bytes out instead.

**An arena that never runs destructors leaks non-memory resources.** Allocating
a type holding a `std::string` or a file handle in an arena and then calling
`reset()` skips its destructor entirely. Restrict arenas to trivially
destructible types, or keep a list of destructors to run.

**`memcpy`ing a struct to disk or a socket exports your padding.** Padding bytes
are uninitialized, so the output is non-deterministic (bad for hashing and
checksums) and unreadable on a machine with different alignment rules. Serialize
field by field.

**PIMPL with a defaulted destructor in the header does not compile.** `~Widget()
= default;` in the header instantiates `~unique_ptr<Impl>` where `Impl` is
incomplete. Declare the destructor in the header, define it (`Widget::~Widget()
= default;`) in the `.cpp` after `Impl` is complete. Same for the move
operations.

**`alignas(64)` on a heap-allocated type needs C++17.** Before C++17,
`operator new` had no alignment-aware overload and over-aligned types were
silently under-aligned on the heap. Check your standard level if you rely on it.

## Exercise

Implement a fixed-size **pool allocator** for a single object type and prove it
beats `new`/`delete`.

1. `template <typename T, std::size_t N> class Pool` preallocates
   `alignas(T) std::byte storage_[N * sizeof(T)]` and maintains a free list
   *inside the unused blocks themselves* (store the next-free index in the
   block's first bytes — this is why pool allocators need no side table).
2. `T* acquire(Args&&...)` placement-news a `T` into a free block and returns
   it; `void release(T*)` calls `p->~T()` and pushes the block back on the free
   list. Throw `std::bad_alloc` when full.
3. Benchmark 1,000,000 acquire/release cycles of a 32-byte type against plain
   `new`/`delete`, using the [`Timer`](../level-3/09-performance-profiling.md)
   from Level 3. Report the ratio.
4. Verify the alignment invariant with an `assert` that every returned pointer
   satisfies `reinterpret_cast<uintptr_t>(p) % alignof(T) == 0`, and run the
   whole thing under `-fsanitize=address,undefined`. It must be clean — a
   custom allocator that trips ASan is a memory-corruption bug waiting for a
   deadline.

Then answer: what happens if a caller `release()`s the same pointer twice, and
what is the cheapest check that would catch it in a debug build?
