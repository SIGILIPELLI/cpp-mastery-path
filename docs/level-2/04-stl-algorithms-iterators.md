# 04 · STL Algorithms & Iterators

The containers in [Module 3](03-stl-containers.md) store data. The algorithms
in `<algorithm>` operate on it — sorting, searching, transforming, filtering —
and they work on *any* container, because they never touch containers
directly. They talk to **iterators**.

That indirection is the whole design of the STL: N containers plus M
algorithms would normally require N×M implementations. With iterators as the
common language, you write N containers and M algorithms and everything
composes. Learning to reach for `<algorithm>` instead of hand-writing loops is
one of the clearest markers of an intermediate C++ programmer.

## Iterators: generalised pointers

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> v{10, 20, 30, 40};

    // Explicit iterator loop -- what a range-for compiles into
    for (std::vector<int>::iterator it = v.begin(); it != v.end(); ++it) {
        std::cout << *it << ' ';       // dereference like a pointer
    }
    std::cout << std::endl;

    for (auto it = v.begin(); it != v.end(); ++it) { /* 'auto' saves your sanity */ }

    // Reverse iteration
    for (auto it = v.rbegin(); it != v.rend(); ++it) {
        std::cout << *it << ' ';       // 40 30 20 10
    }
    std::cout << std::endl;

    std::cout << *(v.begin() + 2) << std::endl;   // 30 -- random access only
    std::cout << v.end() - v.begin() << std::endl; // 4
}
// Output:
// 10 20 30 40
// 40 30 20 10
// 30
// 4
```

`begin()` points at the first element; `end()` points **one past the last**.
That half-open range `[begin, end)` is why `it != end()` is the loop
condition, why `end() - begin()` is the size, and why an empty range is simply
`begin() == end()`. Dereferencing `end()` is undefined behaviour — it is a
position, not an element.

Use `cbegin()`/`cend()` when you want const iterators explicitly.

## Iterator categories

Not every iterator supports every operation. `std::list` iterators can't do
`it + 5`, which is why `std::sort` doesn't compile on a `std::list` (use
`list::sort()` instead).

| Category | Supports | Example container |
|----------|----------|-------------------|
| Input | `++`, `*` (read once) | `std::istream_iterator` |
| Output | `++`, `*` (write once) | `std::back_insert_iterator` |
| Forward | `++`, `*`, multi-pass | `std::forward_list` |
| Bidirectional | Forward + `--` | `std::list`, `std::map`, `std::set` |
| Random access | Bidirectional + `+n`, `-n`, `<`, `[]` | `std::vector`, `std::deque`, `std::array` |
| Contiguous (C++17) | Random access + guaranteed adjacent memory | `std::vector`, `std::array`, raw arrays |

Each category includes everything above it. An algorithm documents the weakest
category it needs — `std::find` needs only input iterators, `std::sort` needs
random access.

## Sorting

```cpp
#include <iostream>
#include <algorithm>
#include <vector>
#include <string>

struct Person {
    std::string name;
    int age;
};

int main() {
    std::vector<int> nums{5, 2, 9, 1, 7};

    std::sort(nums.begin(), nums.end());                    // ascending
    // 1 2 5 7 9

    std::sort(nums.begin(), nums.end(), std::greater<int>{}); // descending
    // 9 7 5 2 1

    // Custom comparator with a lambda
    std::vector<Person> people{{"Ada", 36}, {"Alan", 41}, {"Grace", 45}};
    std::sort(people.begin(), people.end(),
              [](const Person& a, const Person& b) { return a.age < b.age; });

    for (const auto& p : people) std::cout << p.name << ' ';
    std::cout << std::endl;   // Ada Alan Grace

    // Only need the top 3? Don't sort everything.
    std::vector<int> scores{4, 8, 1, 9, 3, 7, 2};
    std::partial_sort(scores.begin(), scores.begin() + 3, scores.end(),
                      std::greater<int>{});
    std::cout << scores[0] << ' ' << scores[1] << ' ' << scores[2] << std::endl;  // 9 8 7

    // stable_sort preserves the relative order of equal elements
    std::stable_sort(people.begin(), people.end(),
                     [](const Person& a, const Person& b) { return a.name < b.name; });
}
```

Your comparator must be a **strict weak ordering**: it returns `true` only if
`a` comes strictly before `b`. Writing `<=` instead of `<` breaks that
contract, and `std::sort` may then read past the end of the array — a real
crash, not a theoretical one.

## Searching

```cpp
#include <iostream>
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> v{4, 8, 15, 16, 23, 42};

    auto it = std::find(v.begin(), v.end(), 15);
    if (it != v.end()) {
        std::cout << "found at index " << (it - v.begin()) << std::endl;   // 2
    }

    // find_if takes a predicate
    auto even = std::find_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; });
    std::cout << *even << std::endl;   // 4

    // Whole-range predicates
    bool allPositive = std::all_of(v.begin(), v.end(), [](int x) { return x > 0; });
    bool anyBig     = std::any_of(v.begin(), v.end(), [](int x) { return x > 40; });
    bool noneZero   = std::none_of(v.begin(), v.end(), [](int x) { return x == 0; });
    std::cout << allPositive << anyBig << noneZero << std::endl;   // 111

    std::cout << std::count_if(v.begin(), v.end(),
                               [](int x) { return x > 10; }) << std::endl;  // 4

    // binary_search requires a SORTED range -- O(log n) instead of O(n)
    std::cout << std::binary_search(v.begin(), v.end(), 23) << std::endl;   // 1

    // lower_bound: first element NOT LESS than the value
    auto lb = std::lower_bound(v.begin(), v.end(), 16);
    std::cout << *lb << std::endl;   // 16

    auto [minIt, maxIt] = std::minmax_element(v.begin(), v.end());
    std::cout << *minIt << " " << *maxIt << std::endl;   // 4 42
}
```

`binary_search`, `lower_bound`, and `upper_bound` give you wrong answers, not
errors, on an unsorted range. There is no check — sortedness is a
precondition you must guarantee.

## Transforming and accumulating

```cpp
#include <iostream>
#include <algorithm>
#include <numeric>
#include <vector>
#include <string>

int main() {
    std::vector<int> nums{1, 2, 3, 4, 5};

    // transform: map each element through a function
    std::vector<int> squares(nums.size());
    std::transform(nums.begin(), nums.end(), squares.begin(),
                   [](int x) { return x * x; });
    for (int x : squares) std::cout << x << ' ';
    std::cout << std::endl;   // 1 4 9 16 25

    // Two-range transform
    std::vector<int> a{1, 2, 3}, b{10, 20, 30}, sums(3);
    std::transform(a.begin(), a.end(), b.begin(), sums.begin(),
                   [](int x, int y) { return x + y; });
    // sums == {11, 22, 33}

    // accumulate lives in <numeric>, not <algorithm>
    int total = std::accumulate(nums.begin(), nums.end(), 0);
    std::cout << total << std::endl;   // 15

    int product = std::accumulate(nums.begin(), nums.end(), 1,
                                  [](int acc, int x) { return acc * x; });
    std::cout << product << std::endl;   // 120

    // The initial value determines the accumulator TYPE -- a classic bug:
    std::vector<double> prices{1.5, 2.5, 3.0};
    double wrong = std::accumulate(prices.begin(), prices.end(), 0);    // int accumulator!
    double right = std::accumulate(prices.begin(), prices.end(), 0.0);  // double
    std::cout << wrong << " vs " << right << std::endl;   // 6 vs 7
}
```

That last one catches people constantly: passing `0` makes the accumulator an
`int`, so every partial sum is truncated. Pass `0.0`.

## The erase-remove idiom

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v{1, 2, 3, 2, 4, 2, 5};

    // std::remove does NOT remove anything -- it can't, algorithms don't know
    // about containers. It shuffles the survivors to the front and returns an
    // iterator to the new logical end.
    auto newEnd = std::remove(v.begin(), v.end(), 2);
    std::cout << v.size() << std::endl;   // still 7!

    v.erase(newEnd, v.end());             // the container actually shrinks here
    std::cout << v.size() << std::endl;   // 4
    for (int x : v) std::cout << x << ' ';
    std::cout << std::endl;               // 1 3 4 5

    // Usually written as one line -- the "erase-remove idiom"
    std::vector<int> w{1, 2, 3, 4, 5, 6};
    w.erase(std::remove_if(w.begin(), w.end(),
                           [](int x) { return x % 2 == 0; }),
            w.end());
    // w == {1, 3, 5}
}
// Output:
// 7
// 4
// 1 3 4 5
```

This surprises everyone once. An algorithm only ever sees iterators, so it
physically cannot change a container's size. C++20 adds free functions
`std::erase(v, 2)` and `std::erase_if(v, pred)` that do both steps; until you
can rely on C++20, the idiom above is the standard spelling.

## Lambdas and capture

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v{1, 5, 10, 15, 20};
    int threshold = 9;

    auto count = std::count_if(v.begin(), v.end(),
                               [threshold](int x) { return x > threshold; });  // capture by value
    std::cout << count << std::endl;   // 3

    int total = 0;
    std::for_each(v.begin(), v.end(), [&total](int x) { total += x; });  // capture by reference
    std::cout << total << std::endl;   // 51

    // [=] captures everything by value, [&] everything by reference.
    // DANGER: a [&] lambda that outlives the captured variables holds dangling
    // references. Only use [&] for lambdas consumed immediately, like these.
}
```

A lambda is just a compiler-generated struct with an `operator()`. Because its
type is known at compile time, the call is usually inlined completely — an
STL algorithm with a lambda typically compiles to the same machine code as a
hand-written loop.

## Algorithm cheat sheet

| Task | Algorithm | Header |
|------|-----------|--------|
| Sort | `sort`, `stable_sort`, `partial_sort` | `<algorithm>` |
| Find a value | `find`, `find_if`, `find_if_not` | `<algorithm>` |
| Fast find in sorted range | `binary_search`, `lower_bound`, `upper_bound` | `<algorithm>` |
| Count | `count`, `count_if` | `<algorithm>` |
| Test a whole range | `all_of`, `any_of`, `none_of` | `<algorithm>` |
| Map each element | `transform` | `<algorithm>` |
| Filter out elements | `remove_if` + `erase` | `<algorithm>` |
| Copy | `copy`, `copy_if`, `copy_n` | `<algorithm>` |
| Min / max | `min_element`, `max_element`, `minmax_element` | `<algorithm>` |
| Sum / fold | `accumulate`, `reduce` | `<numeric>` |
| Fill a range with a sequence | `iota` | `<numeric>` |
| Reverse / rotate / shuffle | `reverse`, `rotate`, `shuffle` | `<algorithm>` |
| Drop adjacent duplicates | `unique` + `erase` (sort first) | `<algorithm>` |

## Exercise

Start with `std::vector<std::string> words` holding a couple of dozen words,
some repeated and in mixed case.

1. Use `std::transform` to lowercase every word in place.
2. Use `std::sort` then `std::unique` + `erase` to reduce it to unique words.
3. Use `std::count_if` to report how many words are longer than 5 characters.
4. Use `std::partial_sort` with a custom comparator to find the 3 longest
   words without sorting the whole vector.
5. Use `std::accumulate` with a lambda to build a single comma-separated
   `std::string` of all the words.

Write every step with algorithms — no raw `for` loops. Then rewrite step 5 as a
hand-written loop and compare which version you find easier to read.
