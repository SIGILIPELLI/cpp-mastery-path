# 05 · Security Best Practices

Memory-safety defects in C and C++ have been roughly **70% of the severe
vulnerabilities** reported by Microsoft, Google Chrome and the Android team over
the past decade. That statistic is not an argument against C++; it is an
argument that the language hands you a category of bug that other languages take
away, and that you have to take it away yourself — deliberately, with tools and
habits.

Everything in this module is defensive. None of it is exotic. The programs below
are deliberately broken so you can see what the tooling says when it catches
them.

## The core rule: prefer the type that cannot be wrong

Most C++ security bugs are a raw pointer or a raw length that got out of sync
with reality. The first line of defence is not a check — it's a type that makes
the check unnecessary.

| Don't | Do | Why |
|-------|----|-----|
| `char buf[64]` + `strcpy` | `std::string` | Growth is the container's problem |
| `T* data, size_t len` params | `std::span<T>` (C++20) | Length travels with the pointer |
| `new` / `delete` | `unique_ptr` / `make_shared` | No leak, no double free, no forgotten path |
| `v[i]` on untrusted `i` | `v.at(i)` | Throws instead of corrupting memory |
| `const char*` params | `std::string_view` | Length known; no scan for a NUL that may not exist |
| `sprintf` | `std::format` (C++20) / `ostringstream` | No format-string mismatch |
| `atoi` | `std::from_chars` | Reports failure instead of returning 0 |

## Out-of-bounds access, and the tool that finds it

```cpp
#include <iostream>
#include <vector>
int main() {
    std::vector<int> v{1, 2, 3};
    std::cout << v[5] << "\n";        // out of bounds -- operator[] does NOT check
}
```

Built normally, this prints whatever byte pattern happened to be there, or
crashes, or works fine for a year. Built with AddressSanitizer:

```bash
g++ -std=c++20 -O1 -g -fsanitize=address prog.cpp -o prog
```

```text
ERROR: AddressSanitizer: heap-buffer-overflow on address 0x6020000000e4
READ of size 4 at 0x6020000000e4 thread T0
    #0 0x000102f78b00 in main se1.cpp:6

0x6020000000e4 is located 8 bytes after 12-byte region [0x6020000000d0,0x6020000000dc)
allocated by thread T0 here:
    #0 in operator new(unsigned long)
    #1 0x000102f78940 in main se1.cpp:4
```

Exact line of the bad read, exact line of the allocation, and how far past the
end it went. ASan costs roughly 2x runtime and 3x memory — completely acceptable
for a test suite, and it should be running on one of your CI configurations
from day one.

`v.at(5)` would have thrown `std::out_of_range` instead. Use `at()` whenever the
index came from outside the function; use `[]` only when a nearby invariant
already proves the index is in range.

## Use-after-free

```cpp
auto p = std::make_unique<std::string>("secret");
std::string* raw = p.get();
p.reset();                     // freed here
std::cout << *raw << "\n";     // use-after-free
```

```text
ERROR: AddressSanitizer: heap-use-after-free on address 0x603000001d07
READ of size 1 at 0x603000001d07 thread T0
    #0 in main se3.cpp:9
0x603000001d07 is located 23 bytes inside of 24-byte region
freed by thread T0 here: ...
```

Note that smart pointers did not prevent this — the moment you extract a raw
pointer with `.get()` and store it, you are back to manual lifetime tracking.
The rule is: **raw pointers and references may be used, never stored.** Pass
them down the stack as non-owning parameters; do not put them in a member, a
container, or a lambda that outlives the scope.

Use-after-free is the single most exploited class of memory bug in browsers,
because a freed heap block is often reallocated for attacker-controlled data.

## Integer overflow

Signed overflow is undefined behaviour; unsigned overflow wraps silently. Both
break size calculations, which is exactly where they turn into buffer
overflows.

```cpp
std::size_t count = std::numeric_limits<std::size_t>::max() / 4 + 1;
std::size_t elem = 8;
std::cout << "count*8 = " << (count * elem) << "\n";
```

```text
count       = 4611686018427387904
count*8     = 0   <-- wrapped
sizeIsSafe  = false
```

`malloc(count * elem)` here allocates **zero bytes** and every subsequent write
lands somewhere else. The check is division, done before the multiply:

```cpp
bool sizeIsSafe(std::size_t count, std::size_t elemSize) {
    return elemSize == 0 ||
           count <= std::numeric_limits<std::size_t>::max() / elemSize;
}
```

UBSan catches the signed case at runtime:

```bash
g++ -std=c++20 -O0 -g -fsanitize=undefined prog.cpp -o prog
```

```text
runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
```

## Signed/unsigned comparison

```cpp
int n = -1;
unsigned int m = 1;
std::cout << (n < m);
```

```text
(-1 < 1u) evaluates to false   <-- -1 converted to a huge unsigned
```

`-1` is converted to `4294967295` before the comparison. Every bounds check of
the form `if (index < buffer.size())` where `index` is a signed value that could
go negative is broken this way. `-Wall -Wextra` warns
(`-Wsign-compare`) — treat that warning as an error. C++20's
`std::cmp_less(n, m)` does the mathematically correct comparison.

## Validate at the boundary

Untrusted input is anything from a file, a socket, an environment variable, a
command line, or another process. Validate it **once**, at the point it enters
your program, and convert it into a type that carries the guarantee.

```cpp
#include <charconv>
#include <optional>
#include <string_view>

std::optional<int> parsePort(std::string_view s) {
    int value{};
    auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), value);
    if (ec != std::errc{} || ptr != s.data() + s.size()) return std::nullopt;
    if (value < 1 || value > 65535) return std::nullopt;     // domain check too
    return value;
}
```

`std::from_chars` reports failure through `ec`, doesn't allocate, doesn't throw,
and doesn't consult the locale. `atoi("abc")` returns `0`, which is
indistinguishable from a valid `0`. The `ptr != end` check rejects `"80abc"`,
which most parsers accept as `80`.

## Never build a shell command from input

```cpp
std::system(("ls " + userPath).c_str());   // command injection: userPath = "x; rm -rf ~"
```

There is no escaping scheme for this that is worth trusting. Use `posix_spawn`
or `fork`/`execve` with an argument **array**, which never goes through a shell
parser. The same principle governs SQL — parameterized statements, never string
concatenation, as covered in
[Level 3 Module 7](../level-3/07-sqlite.md).

## Secrets and cryptography

Three rules that cover most of what application code gets wrong:

**Never roll your own crypto.** Use libsodium, OpenSSL, or your platform's
library. This includes "just hashing a password" — use Argon2 or bcrypt through
a real library, never SHA-256 of a salt-and-password string.

**Compare secrets in constant time.** `==` on a `std::string` returns as soon as
it finds a differing byte, and the timing difference is measurable across a
network. Use `sodium_memcmp` or an explicit accumulate-the-XOR loop:

```cpp
bool constantTimeEqual(std::string_view a, std::string_view b) {
    if (a.size() != b.size()) return false;
    unsigned char diff = 0;
    for (std::size_t i = 0; i < a.size(); ++i)
        diff |= static_cast<unsigned char>(a[i] ^ b[i]);   // no early exit
    return diff == 0;
}
```

**Use a cryptographic random source for anything security-relevant.**
`std::rand()` and `std::mt19937` are predictable from a handful of outputs.
`std::random_device` *may* be cryptographic but is not guaranteed to be — on
some toolchains it has historically returned a constant sequence. For tokens,
keys, salts and nonces, call the OS: `getrandom()` on Linux,
`arc4random_buf()` on BSD/macOS, `BCryptGenRandom` on Windows.

## Build with the hardening flags on

```bash
g++ -std=c++20 -O2 \
    -Wall -Wextra -Wpedantic -Wconversion -Wshadow -Werror \
    -D_GLIBCXX_ASSERTIONS \
    -D_FORTIFY_SOURCE=2 \
    -fstack-protector-strong \
    -fPIE -pie -Wl,-z,relro,-z,now \
    app.cpp -o app
```

`_GLIBCXX_ASSERTIONS` turns `v[i]` into a checked access in libstdc++ (libc++
uses `-D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_EXTENSIVE`).
`_FORTIFY_SOURCE` adds compile- and run-time bounds checks to `memcpy` and
friends. `-fstack-protector-strong` puts a canary before the return address.
`relro,now` makes the GOT read-only after loading. Each of these costs a
fraction of a percent and removes an exploitation technique.

## Cheat sheet

| Practice | Tool / construct |
|----------|------------------|
| Find memory errors | `-fsanitize=address` (+ `,undefined`) in CI |
| Find data races | `-fsanitize=thread` |
| Find uninitialized reads | `-fsanitize=memory` (Clang) or Valgrind memcheck |
| Checked indexing | `.at()`, `_GLIBCXX_ASSERTIONS`, libc++ hardening |
| Pointer + length as one type | `std::span<T>`, `std::string_view` |
| Parse without silent failure | `std::from_chars`, `std::optional` return |
| Safe formatting | `std::format` (C++20), never `sprintf` |
| Subprocess without a shell | `posix_spawn` / `execve` with an argv array |
| Constant-time secret compare | `sodium_memcmp` or an XOR-accumulate loop |
| Cryptographic randomness | `getrandom` / `arc4random_buf` / `BCryptGenRandom` |
| Static analysis | `clang-tidy` (`cert-*`, `bugprone-*`, `cppcoreguidelines-*`) |
| Fuzzing | `-fsanitize=fuzzer` (libFuzzer) on every parser you own |
| Dependency CVEs | `cargo-audit`-equivalent: OSV-Scanner, Conan/vcpkg advisories |

## Traps

**Sanitizers only find what your tests execute.** ASan is not a static prover;
an unreachable code path is an unchecked code path. Combine sanitizers with
fuzzing (`-fsanitize=fuzzer,address`) on every function that parses untrusted
bytes — that combination finds the bugs your test corpus never imagined.

**Never ship a sanitizer build.** ASan disables ASLR-related hardening and adds
a large, predictable shadow-memory mapping. It is a development tool.

**`-fsanitize=address` and `-fsanitize=thread` are mutually exclusive.** Build
and run two separate configurations.

**`std::string_view` and `std::span` do not own.** They are precisely the
"pointer plus length" you were trying to escape, with the length managed for
you. A view outliving its buffer is a use-after-free with a modern-looking type.

**Clearing a secret with `memset` may be optimized away.** The compiler sees a
write to memory that is never read again and deletes it — this is a real CVE
class. Use `explicit_bzero`, `SecureZeroMemory`, or
`std::fill` through a `volatile` pointer.

**Exception safety is a security property.** A function that allocates, throws,
and leaks gives an attacker a memory-exhaustion primitive. The RAII discipline
from [Level 3 Module 4](../level-3/04-raii-deep-dive.md) is what makes leak-free
unwinding automatic.

**`std::random_device` is not guaranteed random.** The standard permits a
deterministic implementation. Do not use it for keys.

## Exercise

Take a deliberately vulnerable parser and harden it, measuring the difference at
every step.

1. Write a "record parser" that reads length-prefixed records from a
   `std::vector<unsigned char>` buffer: a 4-byte little-endian length, then that
   many bytes of payload. Write it *badly* first — `memcpy` the length, then
   `memcpy` the payload into a fixed `char buf[256]`, with no checks.
2. Build it with `-fsanitize=address,undefined` and feed it a crafted buffer
   whose declared length is 100000. Record exactly what ASan reports.
3. Now harden it: reject a length that exceeds the remaining buffer, reject a
   length that exceeds your maximum record size, use `std::span` for the buffer
   and `std::string` for the payload, and return `std::optional<Record>`.
   Confirm the same input is now cleanly rejected.
4. Add a length of `0xFFFFFFFF` and confirm your `offset + length` arithmetic
   does not wrap — write the check as `length > buffer.size() - offset`, never
   `offset + length > buffer.size()`, and explain in a comment why.
5. Wrap it in a libFuzzer harness
   (`extern "C" int LLVMFuzzerTestOneInput(const uint8_t*, size_t)`), build with
   `-fsanitize=fuzzer,address`, and run it for five minutes. Report whether it
   found anything you missed.

Step 5 is the point of the exercise. Hand-written tests confirm what you already
thought of; the fuzzer is what finds the case you didn't.
