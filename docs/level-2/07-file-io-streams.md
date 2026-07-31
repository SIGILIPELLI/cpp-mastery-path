# 07 · File I/O & Streams

You've been using `std::cout` and `std::cin` since Level 1. Those are just two
particular streams. C++ models *every* character source and sink the same way
— files, the console, in-memory strings — so the `<<` and `>>` operators you
already know work identically on all of them.

That uniformity is the payoff. A function taking `std::ostream&` writes to the
console, a file, or a string buffer depending only on what you hand it, which
also makes it trivially testable.

## Writing a file

```cpp
#include <fstream>
#include <iostream>

int main() {
    std::ofstream out("scores.txt");     // opens for writing, TRUNCATES existing content

    if (!out) {                           // always check -- opening can fail
        std::cerr << "Could not open scores.txt for writing" << std::endl;
        return 1;
    }

    out << "Ada 95" << std::endl;
    out << "Alan 88" << std::endl;
    out << "Grace 97" << std::endl;

}   // <-- destructor closes the file automatically (RAII)
```

`std::ofstream` opens on construction and closes in its destructor. You do not
need `out.close()` unless you want to close early and check the result. If the
constructor fails, the stream is left in a failed state — `if (!out)` catches
it. Note the default: opening for writing **wipes the file**. Use
`std::ios::app` to append.

## Reading a file

There are three common ways, and picking the wrong one is the usual source of
bugs.

```cpp
#include <fstream>
#include <iostream>
#include <string>
#include <sstream>

int main() {
    // 1. Line by line -- the right default for text
    std::ifstream in("scores.txt");
    if (!in) {
        std::cerr << "Cannot open scores.txt" << std::endl;
        return 1;
    }

    std::string line;
    while (std::getline(in, line)) {          // stops at EOF or on error
        std::cout << "[" << line << "]" << std::endl;
    }

    // 2. Token by token -- whitespace-separated
    std::ifstream in2("scores.txt");
    std::string name;
    int score;
    while (in2 >> name >> score) {            // the whole expression is the condition
        std::cout << name << " scored " << score << std::endl;
    }

    // 3. Whole file into one string
    std::ifstream in3("scores.txt");
    std::stringstream buffer;
    buffer << in3.rdbuf();                    // slurp the entire stream buffer
    std::string contents = buffer.str();
    std::cout << contents.size() << " bytes" << std::endl;
}
// Output:
// [Ada 95]
// [Alan 88]
// [Grace 97]
// Ada scored 95
// Alan scored 88
// Grace scored 97
// 24 bytes
```

**The classic mistake:**

```cpp
while (!in.eof()) {          // WRONG
    std::getline(in, line);
    std::cout << line << std::endl;   // prints the last line twice
}
```

`eof()` reports whether the *previous* read hit end-of-file. On the final
iteration the read fails, `line` keeps its old value, and you process it a
second time. Always put the read itself in the condition:
`while (std::getline(in, line))`. That works because streams convert to `bool`
based on their error state.

## Stream state flags

```cpp
#include <sstream>
#include <iostream>

int main() {
    std::istringstream in("42 abc");
    int a = 0, b = 0;

    in >> a;
    std::cout << "a=" << a << " good=" << in.good() << std::endl;   // a=42 good=1

    in >> b;                                    // "abc" is not an int
    std::cout << "fail=" << in.fail() << std::endl;                  // fail=1

    // A failed stream stays failed -- all later reads are no-ops until you clear it
    in.clear();                                 // reset the flags
    std::string word;
    in >> word;
    std::cout << "word=" << word << std::endl;  // word=abc
}
```

| Flag | Meaning | Check with |
|------|---------|-----------|
| `goodbit` | Everything fine | `s.good()` |
| `eofbit` | End of input reached | `s.eof()` |
| `failbit` | Last operation failed (bad format, open failed) | `s.fail()` |
| `badbit` | Unrecoverable stream corruption | `s.bad()` |

`if (stream)` is true exactly when neither `failbit` nor `badbit` is set.
After a formatting failure the bad characters are **still in the buffer** — you
must `clear()` the flags and then discard them, or you'll loop forever:

```cpp
#include <limits>

std::cin.clear();
std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');   // skip the rest of the line
```

## Parsing structured lines

```cpp
#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <vector>

struct Record {
    std::string name;
    std::string city;
    int age;
};

std::vector<Record> loadCsv(const std::string& path) {
    std::vector<Record> records;
    std::ifstream in(path);
    if (!in) return records;

    std::string line;
    while (std::getline(in, line)) {
        if (line.empty()) continue;

        std::istringstream ss(line);       // treat the line as its own stream
        std::string name, city, ageText;

        // getline with a delimiter splits on commas
        if (!std::getline(ss, name, ',')) continue;
        if (!std::getline(ss, city, ',')) continue;
        if (!std::getline(ss, ageText))   continue;

        try {
            records.push_back({name, city, std::stoi(ageText)});
        } catch (const std::exception& e) {
            std::cerr << "Skipping bad row: " << line << std::endl;
        }
    }
    return records;
}
```

`std::istringstream` turns a string into a stream, letting you reuse the same
extraction machinery on data you already have in memory. `std::ostringstream`
does the reverse — it's the idiomatic way to build a formatted string:

```cpp
#include <sstream>
#include <iomanip>

std::ostringstream oss;
oss << "Total: " << std::fixed << std::setprecision(2) << 1234.5;
std::string label = oss.str();   // "Total: 1234.50"
```

## Formatting with manipulators

```cpp
#include <iostream>
#include <iomanip>

int main() {
    double pi = 3.14159265358979;

    std::cout << std::fixed << std::setprecision(3) << pi << std::endl;   // 3.142
    std::cout << std::scientific << pi << std::endl;                       // 3.142e+00
    std::cout << std::defaultfloat << std::setprecision(6);                // restore

    std::cout << std::setw(10) << std::right << "Name"
              << std::setw(8)  << "Score" << std::endl;
    std::cout << std::setw(10) << std::left << "Ada"
              << std::setw(8)  << 95 << std::endl;

    std::cout << std::setfill('0') << std::setw(5) << 42 << std::endl;    // 00042
    std::cout << std::setfill(' ');                                        // reset

    std::cout << std::hex << 255 << std::endl;      // ff
    std::cout << std::oct << 8 << std::endl;        // 10
    std::cout << std::dec << 255 << std::endl;      // 255

    std::cout << std::boolalpha << true << std::endl;   // true  (instead of 1)
}
```

Crucially, **all of these except `setw` are sticky**. Set `std::hex` and every
subsequent integer prints in hex until you set `std::dec`. `std::setw` applies
to the very next output item only, which is why it's repeated on each column
above. Forgetting stickiness is a frequent source of "why is my number
printing weirdly three functions later".

| Manipulator | Effect | Sticky? |
|-------------|--------|---------|
| `std::setw(n)` | Minimum field width | **no** — next item only |
| `std::setfill(c)` | Pad character | yes |
| `std::setprecision(n)` | Digits of precision | yes |
| `std::fixed` / `std::scientific` | Float notation | yes |
| `std::left` / `std::right` | Alignment within the field | yes |
| `std::hex` / `std::oct` / `std::dec` | Integer base | yes |
| `std::boolalpha` | `true`/`false` instead of `1`/`0` | yes |
| `std::endl` | Newline **plus flush** | n/a |

`std::endl` flushes the buffer every time. In a tight loop writing thousands of
lines, `'\n'` is meaningfully faster — flush only when you need the output to
appear immediately (progress messages, crash-adjacent logging).

## Open modes and binary I/O

| Mode | Meaning |
|------|---------|
| `std::ios::in` | Read (default for `ifstream`) |
| `std::ios::out` | Write, truncating (default for `ofstream`) |
| `std::ios::app` | Append — every write goes to the end |
| `std::ios::trunc` | Explicitly empty the file on open |
| `std::ios::ate` | Seek to end on open, but writes may go anywhere |
| `std::ios::binary` | No newline translation (essential on Windows) |

```cpp
#include <fstream>
#include <vector>

int main() {
    std::ofstream log("app.log", std::ios::app);      // append, don't wipe
    log << "started\n";

    // Binary: write raw bytes, no formatting
    std::vector<int> data{1, 2, 3, 4};
    std::ofstream bin("data.bin", std::ios::binary);
    bin.write(reinterpret_cast<const char*>(data.data()),
              data.size() * sizeof(int));
    bin.close();

    std::vector<int> loaded(4);
    std::ifstream binIn("data.bin", std::ios::binary);
    binIn.read(reinterpret_cast<char*>(loaded.data()),
               loaded.size() * sizeof(int));
    // loaded == {1, 2, 3, 4}
}
```

Binary dumps like this are fast but **not portable** — they encode your
platform's integer size and endianness. Fine for a local cache, wrong for a
file format other machines will read.

## Writing testable code with streams

```cpp
#include <iostream>
#include <sstream>
#include <vector>
#include <string>

// Takes an abstract ostream, so it doesn't care where the output goes
void printReport(std::ostream& os, const std::vector<std::string>& items) {
    for (std::size_t i = 0; i < items.size(); ++i) {
        os << (i + 1) << ". " << items[i] << "\n";
    }
}

int main() {
    std::vector<std::string> items{"apples", "bread"};

    printReport(std::cout, items);        // to the console

    std::ostringstream captured;
    printReport(captured, items);          // to a string, for a unit test
    if (captured.str() == "1. apples\n2. bread\n") {
        std::cout << "test passed" << std::endl;
    }
}
```

This is the practical reason to type parameters as `std::ostream&` rather than
hardcoding `std::cout`: the same function becomes verifiable without touching
the file system.

## Exercise

Write a small contact-book program.

1. Define `struct Contact { std::string name, email; int age; };`
2. `saveContacts(const std::string& path, const std::vector<Contact>&)` writes
   one comma-separated contact per line.
3. `loadContacts(const std::string& path)` reads them back with `std::getline`
   and `std::istringstream`, skipping malformed lines with a warning to
   `std::cerr` instead of crashing.
4. `printTable(std::ostream& os, const std::vector<Contact>&)` prints an
   aligned table using `std::setw` and `std::left`.

Verify a round trip: save, load, and confirm you got the same data back. Then
feed it a deliberately corrupted file (a row with a missing field, a row with
letters where the age should be) and confirm the program reports the bad rows
and keeps going — [Module 8](08-error-handling-exceptions.md) goes deeper on
that error-reporting decision.
