# 01 · Setup & First Program

## Install a C++ compiler

C++ source files are compiled directly to native machine code — there's no
virtual machine in between. You need a compiler that implements at least
C++17. Pick one for your platform:

```bash
# macOS -- Xcode command line tools (provides clang++)
xcode-select --install

# Ubuntu/Debian -- GCC
sudo apt install build-essential

# Windows -- either install MSYS2/MinGW-w64 (g++), or Visual Studio
# Community Edition (includes MSVC's cl.exe and a full IDE)
```

Verify the install:

```bash
g++ --version
# g++ (Homebrew GCC ...) 13.x   -- or similar

clang++ --version
# Apple clang version 15.x      -- macOS default
```

Either `g++` or `clang++` works fine for everything in this course; examples
use `g++` but the commands are interchangeable.

## Compiling and running by hand

Create `hello.cpp`:

```cpp
// hello.cpp
#include <iostream>

int main() {
    std::cout << "Hello, world!" << std::endl;
    return 0;
}
```

Compile, then run:

```bash
g++ -std=c++17 -Wall -o hello hello.cpp
# produces an executable named "hello"

./hello
# Hello, world!
```

- `-std=c++17` tells the compiler which language standard to target. Always
  set this explicitly — the default varies by compiler and version.
- `-Wall` turns on common warnings. Treat warnings as bugs waiting to happen;
  get comfortable reading and fixing them from day one.
- `-o hello` names the output binary; without it you'd get `a.out`.

## Anatomy of the program

| Piece | Meaning |
|-------|---------|
| `#include <iostream>` | Pulls in the standard I/O library so `std::cout` is available. |
| `int main()` | The program's entry point — execution always starts here. |
| `std::cout << ...` | Streams text to standard output. `std::` means "from the standard library's namespace." |
| `std::endl` | Inserts a newline and flushes the output buffer. |
| `return 0;` | Tells the operating system the program exited successfully (non-zero means an error occurred). |
| `;` | Every statement ends with a semicolon. |
| `{ }` | Curly braces delimit blocks — function bodies, loop bodies, class bodies. |

## Header files and translation units

Every `#include` textually pastes in the contents of another file before
compilation. `<iostream>` is a *standard library header* — angle brackets
`< >` tell the compiler to look in its standard include paths. Later, when you
write your own headers, you'll use quotes instead: `#include "myheader.h"`.
Each `.cpp` file the compiler processes on its own is called a **translation
unit**; multi-file programs are covered starting in
[Module 10](10-project-bank-account-cli.md).

## Namespaces and `using namespace std;`

You'll often see `using namespace std;` at the top of beginner code so you can
write `cout` instead of `std::cout`. It saves typing, but it also pulls
*everything* from `std` into scope, which can cause silent name clashes in
larger programs. This course spells out `std::` explicitly — it's slightly
more typing but it's what production C++ code does, and it makes clear
exactly where every name comes from.

## Choosing an editor

Any of these work well for this course: **VS Code** with the "C/C++" and
"CMake Tools" extensions (free, lightweight, great for following along),
**CLion** (excellent C++-specific tooling, paid but free for students), or
your platform's default (Xcode on macOS, Visual Studio on Windows). VS Code is
the most common lightweight choice — pick one and move on, since the editor
matters far less than practice.

## Exercise

Write a program `greeter.cpp` with a `main` function that prints a greeting
for three different names, one per line, using `std::cout`. Compile it with
`g++ -std=c++17 -Wall -o greeter greeter.cpp` and run the resulting binary
from the command line.
