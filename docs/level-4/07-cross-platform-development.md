# 07 · Cross-platform Development

"Portable C++" is not a property of the standard; it is a property of the code
you wrote and the CI matrix you run it on. The standard leaves a large number of
things implementation-defined on purpose — the size of `long`, whether `char` is
signed, how paths are spelled — and every one of those is a place where code
that works everywhere you tested it fails somewhere you didn't.

The strategy has three parts: **use the standard library instead of platform
APIs**, **isolate what remains behind a thin interface**, and **build on every
target in CI**. The last one is not optional. Portability that isn't compiled is
a hypothesis.

## What actually varies

```cpp
#include <bit>
#include <climits>
#include <iostream>

int main() {
    std::cout << "sizeof: int=" << sizeof(int) << " long=" << sizeof(long)
              << " long long=" << sizeof(long long) << " void*=" << sizeof(void*)
              << " size_t=" << sizeof(std::size_t) << "\n";
    std::cout << "CHAR_BIT=" << CHAR_BIT << ", char is "
              << (CHAR_MIN < 0 ? "signed" : "unsigned") << " by default\n";
    std::cout << "endian: " << (std::endian::native == std::endian::little
                                ? "little" : "big") << "\n";
}
```

```text
sizeof: int=4 long=8 long long=8 void*=8 size_t=8
CHAR_BIT=8, char is signed by default
endian: little
```

That's macOS on ARM64. On 64-bit **Windows**, `long` is **4 bytes**, not 8 —
the LLP64 model versus the LP64 model used by Linux and macOS. Code that assumes
`sizeof(long) == sizeof(void*)` is broken on Windows; code that assumes
`sizeof(long) == 4` is broken everywhere else.

`char` is signed on x86 Linux and macOS, and **unsigned on ARM Linux** and
several embedded targets. `char c = getchar(); if (c == EOF)` therefore behaves
differently by architecture — one of the oldest portability bugs there is.

The fixes are simple and mechanical:

```cpp
#include <cstdint>
std::int32_t  x;   // exactly 32 bits, everywhere
std::int64_t  y;   // exactly 64 bits, everywhere
std::uintptr_t p;  // big enough to hold a pointer
std::size_t   n;   // the type of sizeof and container sizes
```

Use `<cstdint>` types whenever the *width* is part of the meaning — file
formats, network protocols, hardware registers. Use `std::size_t` and
`std::ptrdiff_t` for sizes and offsets. Reserve plain `int` for small local
arithmetic where you genuinely don't care.

## Detecting platform and compiler

```cpp
const char* platform() {
#if defined(_WIN32)            // defined on 32- AND 64-bit Windows
    return "Windows";
#elif defined(__APPLE__)
    return "macOS/iOS";
#elif defined(__linux__)
    return "Linux";
#else
    return "unknown";
#endif
}

const char* compiler() {
#if defined(__clang__)         // MUST come first -- Clang also defines __GNUC__
    return "Clang";
#elif defined(_MSC_VER)
    return "MSVC";
#elif defined(__GNUC__)
    return "GCC";
#else
    return "unknown";
#endif
}
```

```text
platform: macOS/iOS, compiler: Clang
```

The ordering comment is the important part. Clang defines `__GNUC__` for
compatibility, and clang-cl defines `_MSC_VER`. Check the most specific
compiler first or your "GCC branch" will silently apply to Clang.

## Feature test macros — check for features, not versions

Version checks like `#if __GNUC__ >= 11` are guesses about which release
shipped which feature. The standard provides a direct answer:

```cpp
#include <version>          // C++20: all library feature-test macros

#ifdef __cpp_lib_ranges
    // std::ranges is available
#else
    // fall back to iterator pairs
#endif
```

```text
__cplusplus = 202002
__cpp_lib_ranges = 202110
```

The value is a date (`YYYYMM`) of the paper the implementation supports, so you
can require a specific revision: `#if __cpp_lib_ranges >= 202106`. Language
features have `__cpp_*` macros (e.g. `__cpp_concepts`) with no header needed.

## `std::filesystem` — the biggest portability win in the standard

Paths were the classic portability tax: separators, drive letters, case
sensitivity, UTF-16 filenames on Windows. `<filesystem>` (C++17) handles all of
it.

```cpp
#include <filesystem>
namespace fs = std::filesystem;

fs::path p = fs::path("data") / "logs" / "app.log";   // operator/ picks the separator
std::cout << "joined path: " << p << "\n";
std::cout << "  parent: " << p.parent_path() << ", stem: " << p.stem()
          << ", ext: " << p.extension() << "\n";
std::cout << "preferred separator: '" << char(fs::path::preferred_separator) << "'\n";
std::cout << "temp dir: " << fs::temp_directory_path() << "\n";
```

```text
joined path: "data/logs/app.log"
  parent: "data/logs", stem: "app", ext: ".log"
preferred separator: '/'
temp dir: "/var/folders/ly/vrxx_3s90hj1gj22sf7pklsw0000gn/T/"
```

On Windows the same code prints `"data\\logs\\app.log"` and `'\'`. Never build a
path with string concatenation and a hard-coded `'/'`; use `operator/`. And
never hard-code `/tmp` — `fs::temp_directory_path()` consults `TMPDIR`,
`%TEMP%`, or the platform default.

Other essentials: `fs::exists`, `fs::create_directories`, `fs::file_size`,
`fs::directory_iterator`, `fs::recursive_directory_iterator`, `fs::rename`,
`fs::remove_all`, `fs::absolute`, `fs::weakly_canonical`.

## Isolating what the standard doesn't cover

Dynamic library loading, memory mapping, process spawning, terminal control and
high-resolution sleeps have no standard API. The pattern is a **narrow
interface with per-platform implementations**, not `#ifdef`s scattered through
business logic:

```text
src/
    platform/platform.h          # one declaration per operation
    platform/platform_posix.cpp  # implementation, POSIX
    platform/platform_win32.cpp  # implementation, Windows
```

```cmake
if(WIN32)
    target_sources(app PRIVATE src/platform/platform_win32.cpp)
else()
    target_sources(app PRIVATE src/platform/platform_posix.cpp)
endif()
```

The rest of the program includes `platform.h` and contains no `#ifdef` at all.
That is the difference between code with two implementations and code with two
hundred conditional branches — the second kind cannot be reasoned about or
tested.

## CMake as the portability layer

```cmake
if(MSVC)
    target_compile_options(app PRIVATE /W4 /permissive- /utf-8 /EHsc)
    target_compile_definitions(app PRIVATE NOMINMAX WIN32_LEAN_AND_MEAN
                                           _CRT_SECURE_NO_WARNINGS)
else()
    target_compile_options(app PRIVATE -Wall -Wextra -Wpedantic)
endif()

find_package(Threads REQUIRED)
target_link_libraries(app PRIVATE Threads::Threads)   # -pthread, or nothing on Windows
```

`NOMINMAX` is not optional if you include `<windows.h>`: it defines `min` and
`max` as macros that break `std::min`, `std::numeric_limits<T>::max()`, and
anything else with those names. `WIN32_LEAN_AND_MEAN` cuts a large amount of
header. `/permissive-` turns on MSVC's conforming mode — without it, MSVC
accepts non-standard code that then fails on GCC and Clang.

`find_package(Threads)` plus `Threads::Threads` is the portable way to say
`-pthread`; hard-coding the flag breaks on MSVC.

## A CI matrix is the only real proof

```yaml
# .github/workflows/ci.yml
jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        build_type: [Debug, Release]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - run: cmake -S . -B build -DCMAKE_BUILD_TYPE=${{ matrix.build_type }}
      - run: cmake --build build --config ${{ matrix.build_type }}
      - run: ctest --test-dir build -C ${{ matrix.build_type }} --output-on-failure
```

Six configurations, three compilers, two standard-library implementations
(libstdc++, libc++, MSVC STL). `fail-fast: false` matters — you want to see
*all* the platform failures at once, not the first one.

## Text: encodings and line endings

Open files in **binary mode** (`std::ios::binary`) unless you specifically want
text translation. On Windows, text mode converts `\n` to `\r\n` on write and
back on read, which silently corrupts binary data and makes byte offsets and
file sizes disagree across platforms.

For filenames and user-visible text, prefer UTF-8 everywhere in your own code
and convert only at the Windows API boundary (`MultiByteToWideChar`, or just use
`fs::path`, which stores the native encoding and converts for you). Add
`/utf-8` to MSVC so it stops assuming the system codepage for source files.

## Cheat sheet

| Concern | Portable answer |
|---------|-----------------|
| Fixed-width integers | `<cstdint>`: `int32_t`, `uint64_t`, `uintptr_t` |
| Sizes and offsets | `std::size_t`, `std::ptrdiff_t` |
| `long` is 4 bytes on Windows | Never assume; use `int64_t` |
| `char` signedness varies | Cast to `unsigned char` before `<cctype>` functions |
| Byte order | `std::endian::native`; convert explicitly at I/O boundaries |
| Paths | `std::filesystem::path` and `operator/` |
| Temp / home directories | `fs::temp_directory_path()`, never `/tmp` |
| Feature availability | `<version>` + `__cpp_lib_*` macros |
| Compiler detection order | `__clang__` → `_MSC_VER` → `__GNUC__` |
| Threads | `<thread>` + `find_package(Threads)` / `Threads::Threads` |
| Windows macro pollution | `NOMINMAX`, `WIN32_LEAN_AND_MEAN` |
| MSVC conformance | `/permissive- /utf-8 /EHsc /W4` |
| Platform-specific code | Separate `.cpp` per platform, selected in CMake |
| Proof of portability | GitHub Actions matrix over 3 OSes × 2 build types |

## Traps

**`#ifdef __GNUC__` catching Clang.** Clang sets `__GNUC__`. Order your checks
from most specific to least.

**`std::endl` where `'\n'` was meant.** `std::endl` flushes the stream every
time — this is a performance bug, not a portability one, but it hides here
because people reach for it thinking it handles line endings. It does not; use
`'\n'`.

**Text-mode file I/O on Windows.** `\n` → `\r\n` translation corrupts binary
data and shifts every offset. Pass `std::ios::binary`.

**`std::tolower(c)` with a plain `char`.** If `char` is signed and the byte is
non-ASCII, the value is negative, and passing a negative value other than `EOF`
to `<cctype>` functions is undefined behaviour. Always
`std::tolower(static_cast<unsigned char>(c))`.

**Case-insensitive filesystems.** macOS (by default) and Windows treat
`Config.h` and `config.h` as the same file; Linux does not. A wrong-case
`#include` compiles on two of your three CI machines. This is one of the most
common cross-platform build failures, and only a Linux job catches it.

**Mixing standard-library implementations or build flags across binaries.**
libstdc++ and libc++ have different, incompatible ABIs for `std::string`. So do
Debug and Release MSVC runtimes. Every binary in a process must be built with
the same toolchain and runtime settings.

**`std::filesystem` needs a link flag on older toolchains.** GCC 8 and Clang 8
require `-lstdc++fs` / `-lc++fs`. CMake's `find_package(Filesystem)` module or a
version check handles it; a bare `#include <filesystem>` may configure fine and
fail at link.

**Assuming `hardware_concurrency()` is nonzero.** It is permitted to return `0`
when the count is unknown. Clamp it: `std::max(1u, std::thread::hardware_concurrency())`.

## Exercise

Make the [task processor](../level-3/10-project-task-processor.md) genuinely
portable and prove it.

1. Add a `platform/` layer with one function — `std::size_t availableMemoryMb()`
   — declared in `platform.h` and implemented twice: `sysconf(_SC_PHYS_PAGES)`
   on POSIX, `GlobalMemoryStatusEx` on Windows. Select the `.cpp` in CMake with
   `if(WIN32)`. No `#ifdef` may appear outside `platform/`.
2. Replace every hard-coded path in the demo with `std::filesystem::path`
   operations, and write its log to `fs::temp_directory_path() / "taskproc.log"`
   opened with `std::ios::binary`.
3. Use `std::max(1u, std::thread::hardware_concurrency())` for the worker count
   and print it.
4. Add a `<version>`-guarded fast path: if `__cpp_lib_jthread` is defined use
   `std::jthread`, otherwise fall back to `std::thread` plus an explicit join
   loop. Verify both branches compile by forcing the fallback with a temporary
   `#undef`-style test build.
5. Add a GitHub Actions matrix over `ubuntu-latest`, `macos-latest` and
   `windows-latest` × `Debug`/`Release`, with `fail-fast: false`, running
   `ctest`. Get all six green.

Step 5 will fail at least once for a reason you did not predict — that is the
lesson, and it is why the matrix exists.
