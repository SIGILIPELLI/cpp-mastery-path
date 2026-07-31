# 09 · Build Tools (CMake Basics)

Every project so far has compiled with one `g++` command. That stops scaling
fast: as soon as you have a dozen source files, a library dependency, and a
teammate on Windows, hand-written compiler invocations become unmaintainable.

CMake is the de facto standard build system generator for C++. It doesn't
compile anything itself — it reads a `CMakeLists.txt` describing *what* to
build and generates the actual build files for your platform's toolchain
(Makefiles, Ninja, Visual Studio projects, Xcode projects). You describe the
project once; every platform gets a native build.

## The smallest possible project

```text
hello/
    CMakeLists.txt
    main.cpp
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)

project(Hello VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)   # fail rather than silently falling back
set(CMAKE_CXX_EXTENSIONS OFF)         # use -std=c++17, not -std=gnu++17

add_executable(hello main.cpp)
```

```cpp
// main.cpp
#include <iostream>

int main() {
    std::cout << "Hello, CMake!" << std::endl;
}
```

Building it — always **out of source**, in a separate `build/` directory:

```bash
cd hello
cmake -S . -B build          # configure: read CMakeLists.txt, generate build files
cmake --build build          # build: invoke make/ninja/msbuild for you
./build/hello                # run
```

```text
-- The CXX compiler identification is GNU 13.2.0
-- Configuring done
-- Generating done
-- Build files have been written to: /home/you/hello/build
[ 50%] Building CXX object CMakeFiles/hello.dir/main.cpp.o
[100%] Linking CXX executable hello
Hello, CMake!
```

Out-of-source builds keep every generated artifact in `build/`, so your source
tree stays clean, `rm -rf build` is a complete reset, and you can keep separate
Debug and Release build directories side by side. Add `build/` to
`.gitignore` — never commit it.

## Multiple files and a library

```text
calculator/
    CMakeLists.txt
    include/
        calculator.h
    src/
        calculator.cpp
        main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.16)
project(Calculator VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# A library target -- reusable, testable, no main()
add_library(calc_lib
    src/calculator.cpp
)

# PUBLIC: anything linking calc_lib also gets this include directory
target_include_directories(calc_lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

# The executable
add_executable(calculator src/main.cpp)

# Linking also inherits calc_lib's PUBLIC include dirs -- no repetition needed
target_link_libraries(calculator PRIVATE calc_lib)
```

```cpp
// include/calculator.h
#ifndef CALCULATOR_H
#define CALCULATOR_H

namespace calc {
    double add(double a, double b);
    double divide(double a, double b);   // throws std::invalid_argument on b == 0
}

#endif
```

```cpp
// src/calculator.cpp
#include "calculator.h"
#include <stdexcept>

namespace calc {

double add(double a, double b) { return a + b; }

double divide(double a, double b) {
    if (b == 0.0) {
        throw std::invalid_argument("division by zero");
    }
    return a / b;
}

}
```

```cpp
// src/main.cpp
#include <iostream>
#include "calculator.h"

int main() {
    std::cout << calc::add(2, 3) << std::endl;        // 5
    std::cout << calc::divide(10, 4) << std::endl;    // 2.5
}
```

Splitting the logic into `calc_lib` isn't ceremony — it's what lets a test
executable link the same code without pulling in `main()`.

## PUBLIC, PRIVATE, INTERFACE

This is the concept that separates modern CMake from the copy-pasted variety,
and it's worth getting right.

| Keyword | Applies to the target itself | Propagates to things that link it |
|---------|------------------------------|-----------------------------------|
| `PRIVATE` | yes | no |
| `PUBLIC` | yes | yes |
| `INTERFACE` | no | yes |

The test: does a *consumer* of this target need to know about it?

```cmake
# calc_lib's own .cpp files use <zlib.h>, but calculator.h does not mention it.
# Consumers never see zlib -> PRIVATE.
target_link_libraries(calc_lib PRIVATE ZLIB::ZLIB)

# calculator.h #includes a header from include/, so consumers need that path too.
target_include_directories(calc_lib PUBLIC include)

# A header-only library has no sources of its own -> INTERFACE.
add_library(tiny_json INTERFACE)
target_include_directories(tiny_json INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/vendor)
```

Getting this right means dependencies flow automatically. Anything linking
`calc_lib` inherits its include paths without a single extra line, and doesn't
inherit implementation details it has no business seeing.

**Avoid the old-style commands.** `include_directories()`, `link_libraries()`,
and `add_definitions()` apply globally to every target in the directory,
including targets added later. Always prefer the `target_*` versions.

## Build types and compiler flags

```cmake
# Default to Release if the user didn't pick, so a bare `cmake -S . -B build`
# doesn't silently produce an unoptimised binary.
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

target_compile_options(calc_lib PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)

target_compile_definitions(calc_lib PRIVATE
    $<$<CONFIG:Debug>:CALC_DEBUG_LOGGING>
)
```

```bash
cmake -S . -B build-debug   -DCMAKE_BUILD_TYPE=Debug
cmake -S . -B build-release -DCMAKE_BUILD_TYPE=Release
cmake --build build-debug
```

| Build type | Typical GCC/Clang flags | Use for |
|------------|------------------------|---------|
| `Debug` | `-g -O0` | Debugging, breakpoints, readable stack traces |
| `Release` | `-O3 -DNDEBUG` | Shipping (`NDEBUG` disables `assert`) |
| `RelWithDebInfo` | `-O2 -g -DNDEBUG` | Profiling, production crash reports |
| `MinSizeRel` | `-Os -DNDEBUG` | Embedded, size-constrained targets |

The `$<...>` syntax is a **generator expression** — evaluated when the build
files are generated, not while `CMakeLists.txt` is being read. That's how one
configuration can carry per-compiler and per-build-type settings at once.

## Finding and fetching dependencies

```cmake
# 1. A library already installed on the system
find_package(Threads REQUIRED)
target_link_libraries(calculator PRIVATE Threads::Threads)

# 2. Download and build a dependency at configure time
include(FetchContent)
FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1          # always pin a tag, never a moving branch
)
FetchContent_MakeAvailable(fmt)

target_link_libraries(calculator PRIVATE fmt::fmt)
```

`FetchContent` is the simplest way to add a dependency with no external package
manager: CMake clones it and builds it as part of your project. Pin an exact
tag — tracking `main` means your build changes underneath you without warning.

`Threads::Threads` and `fmt::fmt` are **imported targets**. Linking one brings
along its include directories, compile definitions, and transitive
dependencies. That's why you'll see `::` in modern CMake everywhere: those are
targets carrying full usage requirements, not bare library names.

## Adding tests

```cmake
enable_testing()

add_executable(calc_tests tests/test_calculator.cpp)
target_link_libraries(calc_tests PRIVATE calc_lib)

add_test(NAME calculator_tests COMMAND calc_tests)
```

```bash
cmake --build build
ctest --test-dir build --output-on-failure
```

```text
Test project /home/you/calculator/build
    Start 1: calculator_tests
1/1 Test #1: calculator_tests .................   Passed    0.01 sec

100% tests passed, 0 tests failed out of 1
```

Any executable returning 0 on success counts as a passing test, so you can
start with a hand-rolled test file today and swap in Google Test later without
changing how you run the suite.

## Organising a larger project

```text
myproject/
    CMakeLists.txt          <- top level
    src/
        CMakeLists.txt
        main.cpp
    lib/
        CMakeLists.txt
        engine.cpp
        include/engine.h
    tests/
        CMakeLists.txt
        test_engine.cpp
```

```cmake
# top-level CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(MyProject VERSION 0.1 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_subdirectory(lib)
add_subdirectory(src)

# Only build tests when this is the top-level project, so consumers who
# FetchContent us don't have to build our test suite.
if(PROJECT_IS_TOP_LEVEL)
    enable_testing()
    add_subdirectory(tests)
endif()
```

Each `add_subdirectory` pulls in that folder's own `CMakeLists.txt`, keeping
each component's build rules next to its code.

## Command cheat sheet

| Task | Command |
|------|---------|
| Configure | `cmake -S . -B build` |
| Configure with a build type | `cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug` |
| Build everything | `cmake --build build` |
| Build in parallel | `cmake --build build -j 8` |
| Build one target | `cmake --build build --target calc_tests` |
| Run tests | `ctest --test-dir build --output-on-failure` |
| Clean | `cmake --build build --target clean` |
| Full reset | `rm -rf build` |
| Install | `cmake --install build --prefix /usr/local` |
| Use Ninja instead of Make | `cmake -S . -B build -G Ninja` |

| CMake command | Purpose |
|---------------|---------|
| `cmake_minimum_required()` | Set the policy/behaviour baseline (first line, always) |
| `project()` | Name, version, languages |
| `add_executable()` | Define a program target |
| `add_library()` | Define a static/shared/interface library target |
| `target_link_libraries()` | Link, and inherit usage requirements |
| `target_include_directories()` | Header search paths, with scope |
| `target_compile_options()` | Per-target compiler flags |
| `target_compile_definitions()` | Per-target `-D` macros |
| `find_package()` | Locate an installed dependency |
| `FetchContent_MakeAvailable()` | Download and build a dependency |
| `add_subdirectory()` | Include another directory's CMakeLists.txt |
| `add_test()` | Register a test with CTest |

## Exercise

Convert the Bank Account CLI from
[Level 1's project](../level-1/10-project-bank-account-cli.md) to CMake.

1. Restructure it as `include/account.h`, `src/account.cpp`, `src/main.cpp`.
2. Build `account.cpp` into a library target `bank_lib` with a `PUBLIC` include
   directory, and link it into a `bank` executable.
3. Enable `-Wall -Wextra` on GCC/Clang and `/W4` on MSVC using a generator
   expression.
4. Add `tests/test_account.cpp` — a plain `main()` that constructs an `Account`,
   asserts a withdrawal beyond the balance throws, and returns 0 on success.
   Register it with `add_test` and run it through `ctest`.
5. Create both `build-debug` and `build-release` directories from the same
   source and compare the binary sizes.

Then deliberately move `target_include_directories(bank_lib PUBLIC include)` to
`PRIVATE`, reconfigure, and read the resulting error. Understanding *why* it
breaks is the whole point of the PUBLIC/PRIVATE distinction.
