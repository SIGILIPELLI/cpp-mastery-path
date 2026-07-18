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

## Exercise

Write a function `std::string reverseWords(const std::string& sentence)` that
takes a sentence and returns it with the order of words reversed (e.g. `"The
quick fox"` becomes `"fox quick The"`). You can split on spaces manually
using `find` and `substr` in a loop, storing each word in a
`std::vector<std::string>` before reassembling it in reverse.
