# 06 · Strings (std::string)

`std::string` is C++'s standard, safe, resizable string type — prefer it over
raw C-style `char*` strings for almost everything in application code.

## Creating and printing strings

```cpp
#include <iostream>
#include <string>

int main() {
    std::string name = "Ada Lovelace";
    std::cout << name << std::endl;          // Ada Lovelace
    std::cout << name.length() << std::endl; // 12 -- length() and size() are equivalent
}
```

`#include <string>` is required to use `std::string` — `<iostream>` alone
does not pull it in.

## Concatenation

```cpp
std::string first = "Grace";
std::string last = "Hopper";

std::string full = first + " " + last;        // + concatenates strings
std::cout << full << std::endl;                // Grace Hopper

full += "!";                                    // += appends in place
std::cout << full << std::endl;                 // Grace Hopper!
```

## Converting between strings and numbers

```cpp
int age = 30;
std::string ageStr = std::to_string(age);       // int -> std::string
std::cout << "Age: " + ageStr << std::endl;      // Age: 30

std::string input = "42";
int parsed = std::stoi(input);                   // std::string -> int
double parsedD = std::stod("3.14");              // std::string -> double

std::cout << parsed + 8 << std::endl;            // 50
```

`std::stoi` throws `std::invalid_argument` if the string doesn't start with a
valid number — see [Module 9](09-exception-handling.md) for handling that
safely.

## Comparing strings

```cpp
std::string a = "apple";
std::string b = "banana";

std::cout << (a == b) << std::endl;   // 0 (false)
std::cout << (a < b) << std::endl;    // 1 (true) -- lexicographic ("dictionary") order
std::cout << (a != b) << std::endl;   // 1 (true)
```

`std::string` supports `==`, `!=`, `<`, `>`, etc. directly — no special
method needed, unlike some languages.

## Accessing and iterating characters

```cpp
std::string word = "hello";

std::cout << word[0] << std::endl;        // h
std::cout << word.at(1) << std::endl;     // e -- bounds-checked, like vector

for (char c : word) {
    std::cout << c << "-";
}
std::cout << std::endl;
// h-e-l-l-o-
```

A `std::string` behaves a lot like a `std::vector<char>` — the same
`operator[]` vs `.at()` tradeoff from [Module 5](05-arrays-vector-basics.md)
applies here.

## Substrings and searching

```cpp
std::string sentence = "The quick brown fox";

std::string sub = sentence.substr(4, 5);       // starts at index 4, length 5
std::cout << sub << std::endl;                  // quick

size_t pos = sentence.find("brown");
if (pos != std::string::npos) {                // npos means "not found"
    std::cout << "Found at index " << pos << std::endl;   // Found at index 10
}

size_t missing = sentence.find("zebra");
std::cout << (missing == std::string::npos) << std::endl;  // 1 (true) -- not found
```

`std::string::npos` is a special constant meaning "no position" — always
compare against it rather than assuming `-1`, since `find` returns an
unsigned `size_t`.

## Useful transformations

```cpp
#include <algorithm>

std::string text = "Hello World";

std::string upper = text;
std::transform(upper.begin(), upper.end(), upper.begin(), ::toupper);
std::cout << upper << std::endl;   // HELLO WORLD

std::string padded = "  trim me  ";
size_t start = padded.find_first_not_of(' ');
size_t end = padded.find_last_not_of(' ');
std::string trimmed = padded.substr(start, end - start + 1);
std::cout << "[" << trimmed << "]" << std::endl;   // [trim me]
```

`<algorithm>` provides generic operations like `std::transform`, which are
covered more thoroughly with the STL in Level 2 — the pattern above (apply a
function to every character) is a common one worth recognizing early.

## Building strings piece by piece with stringstream

```cpp
#include <sstream>

std::ostringstream oss;
oss << "Total: " << 42 << " items, $" << 19.99;
std::string result = oss.str();
std::cout << result << std::endl;   // Total: 42 items, $19.99
```

`std::ostringstream` is handy when you need to build up a formatted string
from mixed types (numbers, strings) without a lot of manual `std::to_string`
and `+` calls.

## How It Actually Works

`std::string` is not a primitive — it's a class that manages a heap-allocated
buffer of characters, much like `std::vector<char>` internally, plus a
null terminator it maintains automatically so `.c_str()` can hand raw C APIs
a valid C-string. Most implementations also apply **Small String
Optimization (SSO)**: strings shorter than roughly 15-22 characters
(implementation-dependent) are stored directly inside the `std::string`
object's own stack/member memory, with no heap allocation at all. Only once
a string grows past that threshold does it allocate on the heap — which is
why short strings are essentially free to copy and construct, while long
ones incur a real `new[]` call.

Concatenating with `+` on `std::string` allocates a brand-new buffer sized
to hold both operands and copies both into it — repeated concatenation in a
loop (`result += s;` many times) can trigger the same reallocate-and-copy
growth pattern as `std::vector`, which is why `+=`/`append` on the same
string object is cheaper than chaining `+` to build new temporaries
repeatedly.

Raw C-style `char*` strings, by contrast, are just a pointer to the first
byte of a sequence that keeps going until a `'\0'` byte is found — there's no
length stored anywhere, so `strlen` has to scan byte-by-byte until it hits
that terminator. This is the root cause of classic C string bugs: read or
write past the terminator (or forget it entirely) and every string function
either walks off into unrelated memory or corrupts it. `std::string` sidesteps
this by tracking its length explicitly as a member field, so `.size()` is an
O(1) lookup, not a scan.

## Exercise

Write a function `std::string reverseWords(const std::string& sentence)` that
takes a sentence and returns it with the order of words reversed (e.g. `"The
quick fox"` becomes `"fox quick The"`). You can split on spaces manually
using `find` and `substr` in a loop, storing each word in a
`std::vector<std::string>` before reassembling it in reverse.
