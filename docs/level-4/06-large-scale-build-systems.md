# 06 · Large-Scale Build Systems

[Level 2 Module 9](../level-2/09-build-tools-cmake.md) introduced CMake as a way
to stop typing `g++` by hand. At scale the build system becomes something else:
the thing that decides whether a change takes 4 seconds or 40 minutes to
validate, whether your library is usable by anyone else, and whether "works on
my machine" is a joke or a support burden.

The single idea that organizes modern CMake is **targets carry their own
requirements**. A target knows its include directories, its compile features,
its definitions and its dependencies; anything that links it inherits the public
ones automatically. Global `include_directories()` and
`set(CMAKE_CXX_FLAGS ...)` are the old way and they do not compose.

## A properly structured project

```text
mathlib/
    CMakeLists.txt
    include/mathlib/stats.h     # the public API -- note the namespaced subdir
    src/stats.cpp
    app/main.cpp
    tests/test_stats.cpp
```

`include/mathlib/stats.h` rather than `include/stats.h` is deliberate: consumers
write `#include "mathlib/stats.h"`, which cannot collide with another library's
`stats.h`.

```cmake
cmake_minimum_required(VERSION 3.20)
project(mathlib VERSION 1.2.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)          # -std=c++20, not -std=gnu++20
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)  # compile_commands.json for clangd/clang-tidy

add_library(mathlib src/stats.cpp)
add_library(mathlib::mathlib ALIAS mathlib)   # same name in-tree and installed

target_include_directories(mathlib
    PUBLIC  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<INSTALL_INTERFACE:include>
    PRIVATE src)

target_compile_features(mathlib PUBLIC cxx_std_20)
target_compile_options(mathlib PRIVATE -Wall -Wextra -Wpedantic)

add_executable(statsapp app/main.cpp)
target_link_libraries(statsapp PRIVATE mathlib::mathlib)

option(MATHLIB_BUILD_TESTS "Build the test suite" ON)
if(MATHLIB_BUILD_TESTS)
    enable_testing()
    find_package(GTest REQUIRED)
    add_executable(test_stats tests/test_stats.cpp)
    target_link_libraries(test_stats PRIVATE mathlib::mathlib GTest::gtest GTest::gtest_main)
    include(GoogleTest)
    gtest_discover_tests(test_stats)   # registers each TEST() with CTest individually
endif()
```

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

```text
-- Found GTest: /opt/homebrew/lib/cmake/GTest/GTestConfig.cmake (found version "1.17.0")
-- Configuring done (0.6s)
-- Generating done (0.0s)
[ 16%] Building CXX object CMakeFiles/mathlib.dir/src/stats.cpp.o
[ 33%] Linking CXX static library libmathlib.a
[ 50%] Building CXX object CMakeFiles/statsapp.dir/app/main.cpp.o
[ 66%] Linking CXX executable statsapp
[ 83%] Building CXX object CMakeFiles/test_stats.dir/tests/test_stats.cpp.o
[100%] Linking CXX executable test_stats
```

```bash
cd build && ctest --output-on-failure
```

```text
    Start 1: StatsTest.MeanOfKnownSet
1/3 Test #1: StatsTest.MeanOfKnownSet .........   Passed    0.00 sec
    Start 2: StatsTest.StddevOfKnownSet
2/3 Test #2: StatsTest.StddevOfKnownSet .......   Passed    0.00 sec
    Start 3: StatsTest.EmptyThrows
3/3 Test #3: StatsTest.EmptyThrows ............   Passed    0.00 sec

100% tests passed out of 3
```

`gtest_discover_tests` runs the test binary at build time with
`--gtest_list_tests` and registers every case as its own CTest entry, so
`ctest -j8` parallelizes across test *cases* and a failure names the case
directly.

## PUBLIC, PRIVATE, INTERFACE

This is the concept that makes large builds tractable, and it is about
**propagation**, not access control.

| Keyword | Used when building the target | Propagated to consumers |
|---------|:-----------------------------:|:-----------------------:|
| `PRIVATE` | yes | no |
| `PUBLIC` | yes | yes |
| `INTERFACE` | no | yes |

If `stats.h` includes `<Eigen/Dense>`, Eigen is `PUBLIC` — anyone including your
header needs it too. If only `stats.cpp` includes it, it is `PRIVATE`, and your
consumers never see it. Getting this wrong in the permissive direction
(everything `PUBLIC`) is how a 20-target project ends up with every target
depending on every header in the repo, and a one-line change rebuilding
everything.

## Getting dependencies

Three mainstream answers, in increasing order of control:

**FetchContent** — CMake downloads and builds the dependency as part of your
build. Zero setup for the user, at the cost of rebuilding it:

```cmake
include(FetchContent)
FetchContent_Declare(googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.17.0)          # always pin a TAG, never a branch
FetchContent_MakeAvailable(googletest)
```

**vcpkg / Conan** — a real package manager with binary caching:

```bash
vcpkg install fmt spdlog gtest
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
```

**`find_package`** against a system or pre-built install — fastest, but you own
provisioning it.

Whichever you choose, the consumption side is identical:
`target_link_libraries(app PRIVATE GTest::gtest)`. That is the payoff of
imported targets.

## Making your library installable

```cmake
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

install(TARGETS mathlib EXPORT mathlibTargets
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(EXPORT mathlibTargets
        FILE mathlibTargets.cmake
        NAMESPACE mathlib::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/mathlib)

write_basic_package_version_file(mathlibConfigVersion.cmake
        VERSION ${PROJECT_VERSION} COMPATIBILITY SameMajorVersion)
```

Now a downstream project writes `find_package(mathlib 1.2 REQUIRED)` and
`target_link_libraries(theirs PRIVATE mathlib::mathlib)`, and the
`$<INSTALL_INTERFACE:include>` from earlier is what makes the include path
correct in the installed layout. The `BUILD_INTERFACE`/`INSTALL_INTERFACE`
generator-expression pair exists precisely because those two paths differ.

## Presets — reproducible configurations

`CMakePresets.json` replaces the wiki page of `cmake -D...` incantations that
every large project accumulates:

```json
{
  "version": 3,
  "configurePresets": [
    { "name": "dev", "generator": "Ninja", "binaryDir": "build/dev",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug",
                          "CMAKE_EXPORT_COMPILE_COMMANDS": "ON" } },
    { "name": "asan", "inherits": "dev", "binaryDir": "build/asan",
      "cacheVariables": { "CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -g" } },
    { "name": "release", "inherits": "dev", "binaryDir": "build/release",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "RelWithDebInfo" } }
  ]
}
```

```bash
cmake --preset asan && cmake --build --preset asan && ctest --preset asan
```

CI and developers now run byte-identical configurations, which is the only way
"it passes locally" stays meaningful.

## Making builds fast

| Technique | Typical effect | Cost |
|-----------|---------------|------|
| **Ninja generator** (`-G Ninja`) | 10-30% over Make; far better incremental | none |
| **ccache / sccache** | Near-instant rebuild of unchanged TUs across branches | disk |
| **Precompiled headers** (`target_precompile_headers`) | Large win on STL-heavy code | can hide missing includes |
| **Unity builds** (`CMAKE_UNITY_BUILD`) | 2-4x on full builds | worse incremental; ODR surprises |
| **Forward declarations & PIMPL** | Cuts the rebuild *fan-out* | design effort |
| **`-fvisibility=hidden`** | Smaller, faster-linking shared libraries | must export explicitly |
| **`mold` / `lld` linker** | Link time from tens of seconds to ~1 | none |
| **`include-what-you-use`** | Removes transitive header bloat at the source | churn |

The most valuable of these is not a flag. **The dependency graph of your headers
is your build time.** A widely-included header that pulls in `<regex>` costs
every translation unit in the project; the fix is forward declaration and
[PIMPL](04-systems-programming-patterns.md), not a faster machine.

Measure before optimizing here too — Clang's `-ftime-trace` emits a Chrome
trace per TU showing exactly which header and which template instantiation ate
the seconds.

## C++20 modules, briefly

```cpp
// stats.cppm
export module mathlib.stats;
import <vector>;
export namespace mathlib {
    double mean(const std::vector<double>& v);
}
```

```cmake
target_sources(mathlib PUBLIC FILE_SET CXX_MODULES FILES src/stats.cppm)
```

Modules replace textual inclusion with a compiled interface, which eliminates
both header-parsing cost and macro leakage. Support arrived in CMake 3.28 with
Ninja 1.11+ and recent Clang/GCC/MSVC, and it works — but tooling around it
(static analyzers, older CI images, third-party libraries that still ship
headers) is still catching up. Know what they are; adopt them when your whole
toolchain is ready, not before.

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `target_include_directories(t PUBLIC …)` | Include paths that propagate to consumers |
| `target_link_libraries(t PRIVATE dep::dep)` | Link + inherit dep's public requirements |
| `target_compile_features(t PUBLIC cxx_std_20)` | Require a standard, propagate the requirement |
| `add_library(ns::name ALIAS name)` | Same target name in-tree and after install |
| `$<BUILD_INTERFACE:…>` / `$<INSTALL_INTERFACE:…>` | Different paths before and after install |
| `FetchContent_Declare/MakeAvailable` | Build a pinned dependency from source |
| `install(TARGETS … EXPORT …)` + `install(EXPORT …)` | Make `find_package` work downstream |
| `enable_testing()` + `gtest_discover_tests` | Register each test case with CTest |
| `ctest -j8 --output-on-failure` | Run the suite in parallel, show failures |
| `cmake --preset dev` | Reproducible, named configurations |
| `CMAKE_EXPORT_COMPILE_COMMANDS` | `compile_commands.json` for clangd / clang-tidy |
| `cmake --build build --target help` | List available targets |

## Traps

**`include_directories()` / `link_libraries()` without a target.** These are
directory-scoped globals that leak into every target defined afterwards,
including ones added later by a subdirectory. Always use the `target_` form.

**No `CMAKE_BUILD_TYPE` with a single-config generator means no optimization
flags at all** — not `-O0`, not `-O2`, just nothing. Benchmarks from such a
build are meaningless. Set it explicitly or via a preset.

**`file(GLOB …)` for source lists.** CMake globs at *configure* time, so a
newly added file is silently not built until someone reconfigures — and the
error appears as a link failure on someone else's machine. List sources
explicitly. `CONFIGURE_DEPENDS` mitigates it and still isn't reliable across
generators.

**Pinning a dependency to a branch.** `GIT_TAG main` makes your build
non-reproducible and your CI failures arrive from someone else's commit. Pin a
tag or a commit hash.

**Everything `PUBLIC`.** Every consumer then depends on every private header,
your rebuild fan-out becomes the whole repo, and an internal dependency becomes
part of your ABI. Default to `PRIVATE` and promote deliberately.

**Precompiled headers hiding missing includes.** A `.cpp` that compiles only
because the PCH happened to include `<string>` breaks the moment the PCH changes
or another build system compiles it. Run `include-what-you-use` or a
no-PCH configuration in CI.

**Unity builds and ODR.** Concatenating translation units merges their anonymous
namespaces and `static` symbols; two files with a `static int counter;` now
share one. Unity builds must be validated by a non-unity CI job.

## Exercise

Convert the [Level 3 task processor](../level-3/10-project-task-processor.md)
into a properly packaged CMake project.

1. Split it into a `taskpool` library target (the queue and pool) plus a
   `taskdemo` executable, with the headers under `include/taskpool/`. Make the
   include directory `PUBLIC` and `Threads::Threads` a `PUBLIC` link dependency
   (it must propagate — consumers need `-pthread` too).
2. Add a `tests/` target using GoogleTest via `FetchContent` pinned to
   `v1.17.0`, register it with `gtest_discover_tests`, and confirm
   `ctest --output-on-failure` shows each `TEST` separately.
3. Add `install()` rules with an export set, install to a local prefix
   (`cmake --install build --prefix /tmp/stage`), then write a *separate*
   throwaway project that does `find_package(taskpool REQUIRED)` and links it.
   It must build with no manual include paths — that is the actual test of
   whether your `BUILD_INTERFACE`/`INSTALL_INTERFACE` split is right.
4. Add a `CMakePresets.json` with `dev`, `asan` (`-fsanitize=address,undefined`)
   and `tsan` (`-fsanitize=thread`) presets. Run the test suite under all three.
   TSan on a thread pool is not a formality — expect it to find something if
   your queue is not exactly right.
5. Time a full build with Make and with Ninja, then time an incremental build
   after touching one `.cpp` and after touching one `.h`. Report the four
   numbers and explain the difference between the last two.
