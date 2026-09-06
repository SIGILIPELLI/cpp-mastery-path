# 03 · Control Flow

Control flow statements decide which code runs, and how many times.

## if / else if / else

```cpp
#include <iostream>

int main() {
    int score = 72;

    if (score >= 90) {
        std::cout << "Grade: A" << std::endl;
    } else if (score >= 80) {
        std::cout << "Grade: B" << std::endl;
    } else if (score >= 70) {
        std::cout << "Grade: C" << std::endl;
    } else {
        std::cout << "Grade: F" << std::endl;
    }
    // Grade: C
}
```

Conditions are evaluated top to bottom; the first branch whose condition is
`true` runs, and the rest are skipped.

## switch

```cpp
int day = 3;

switch (day) {
    case 1:
        std::cout << "Monday" << std::endl;
        break;
    case 2:
        std::cout << "Tuesday" << std::endl;
        break;
    case 3:
        std::cout << "Wednesday" << std::endl;
        break;
    default:
        std::cout << "Some other day" << std::endl;
        break;
}
// Wednesday
```

`break` stops the switch from "falling through" into the next case. Omitting
it is legal C++ and occasionally used intentionally (to group cases), but
forgetting it by accident is a classic bug — `-Wall` (via `-Wimplicit-fallthrough`
in newer compilers) will flag suspicious cases.

## while loops

```cpp
int count = 0;

while (count < 5) {
    std::cout << count << " ";
    count++;
}
std::cout << std::endl;
// 0 1 2 3 4
```

`while` checks the condition *before* each iteration, so the body might run
zero times.

## do-while loops

```cpp
int attempts = 0;

do {
    std::cout << "Attempt " << attempts << std::endl;
    attempts++;
} while (attempts < 3);
// Attempt 0
// Attempt 1
// Attempt 2
```

`do-while` checks the condition *after* the body, so it always runs at least
once — useful for things like "prompt the user, then re-prompt while input is
invalid."

## for loops

```cpp
for (int i = 0; i < 5; i++) {
    std::cout << i << " ";
}
std::cout << std::endl;
// 0 1 2 3 4
```

The three parts — initialization, condition, update — run once, then check,
then repeat: init → check → body → update → check → body → ... until the
condition is false.

## Range-based for loops

```cpp
#include <vector>

std::vector<int> numbers = {10, 20, 30, 40};

for (int n : numbers) {
    std::cout << n << " ";
}
std::cout << std::endl;
// 10 20 30 40

for (const auto& n : numbers) {   // avoids copying each element
    std::cout << n * 2 << " ";
}
std::cout << std::endl;
// 20 40 60 80
```

Range-based `for` (introduced in C++11) iterates directly over a container's
elements without manual indexing. `std::vector` is covered in depth in
[Module 5](05-arrays-vector-basics.md); `const auto&` avoids copying each
element, which matters for larger types like `std::string`.

## break and continue

```cpp
for (int i = 0; i < 10; i++) {
    if (i == 5) {
        break;      // exit the loop entirely
    }
    std::cout << i << " ";
}
std::cout << std::endl;
// 0 1 2 3 4

for (int i = 0; i < 10; i++) {
    if (i % 2 != 0) {
        continue;   // skip the rest of this iteration
    }
    std::cout << i << " ";
}
std::cout << std::endl;
// 0 2 4 6 8
```

## How It Actually Works

At the machine level there's no such thing as `if`, `for`, or `while` —
every control-flow statement the compiler sees becomes **comparison
instructions plus conditional jumps**. `if (x > 0) { a(); } else { b(); }`
lowers to roughly: compare `x` to `0`, jump to the `else` block's address if
the comparison is false, otherwise fall through to `a()`'s code, then an
unconditional jump past the `else` block. The CPU has no concept of nested
braces — it only has a linear instruction stream and jumps that hop around
in it.

A `for (int i = 0; i < n; i++)` loop desugars into the same
initialize-check-jump-increment pattern as a `while` loop with a jump back to
the top; the compiler treats them as the same construct. Modern compilers
also try **loop unrolling** — duplicating the loop body several times to cut
down on the number of conditional jumps executed, since a mispredicted
branch (the CPU guessed wrong about whether a jump would be taken) stalls
the pipeline for several cycles. This is why tight, predictable loops (like
counting up to a fixed bound) are cheap, while loops with unpredictable
branching inside them run measurably slower than the instruction count alone
would suggest — the CPU's branch predictor cache learns patterns, and code
that behaves consistently gets consistently correct guesses.

`switch` on an integral type compiles differently again: with enough
contiguous cases, the compiler can emit a **jump table** — an array of
addresses indexed directly by the switch value, letting it jump straight to
the matching case in one lookup instead of a chain of comparisons.

## Exercise

Write a program that loops from 1 to 50 and, for each number: prints
"FizzBuzz" if it's divisible by both 3 and 5, "Fizz" if only by 3, "Buzz" if
only by 5, and the number itself otherwise. Then write a separate loop using
`std::vector<int>` that sums only the even numbers in the vector `{3, 8, 12,
7, 19, 24}` and prints the total.
