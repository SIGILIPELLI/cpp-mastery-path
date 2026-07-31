# 03 · STL Containers

You already met `std::vector` in [Level 1 Module 5](../level-1/05-arrays-vector-basics.md).
The Standard Library ships a whole family of containers, each with a different
performance profile. Choosing the right one is one of the highest-leverage
decisions in C++ — the difference between an O(n) lookup and an O(1) lookup on
a hot path is the difference between a program that scales and one that
doesn't.

This module covers the containers you'll reach for daily: `vector`, `deque`,
`list`, `map`/`set`, and their unordered hash-based cousins.

## Sequence containers: vector, deque, list

```cpp
#include <iostream>
#include <vector>
#include <deque>
#include <list>

int main() {
    // vector -- contiguous array, grows at the back
    std::vector<int> v{1, 2, 3};
    v.push_back(4);              // amortised O(1)
    v.insert(v.begin(), 0);      // O(n) -- everything shifts right
    std::cout << v[2] << std::endl;   // 2 -- O(1) random access

    // deque -- double-ended queue, fast push at BOTH ends
    std::deque<int> d{1, 2, 3};
    d.push_front(0);             // O(1) -- vector cannot do this cheaply
    d.push_back(4);              // O(1)
    std::cout << d[0] << std::endl;   // 0 -- still O(1) random access

    // list -- doubly linked list, fast insert/erase ANYWHERE
    std::list<int> l{1, 2, 3};
    auto it = l.begin();
    ++it;
    l.insert(it, 99);            // O(1) once you have the iterator
    // l[1];                     // compile error -- no random access
    for (int x : l) std::cout << x << ' ';
    std::cout << std::endl;
}
// Output:
// 2
// 0
// 1 99 2 3
```

The honest advice: **use `std::vector` unless you have a measured reason not
to.** Its contiguous memory is enormously cache-friendly, and on modern
hardware a linear scan over a vector routinely beats a "theoretically better"
linked list, because every `list` node is a separate allocation somewhere else
in memory. `std::list` earns its place when you hold long-lived iterators or
splice large ranges; `std::deque` when you genuinely push and pop at both ends.

## Vector capacity: the reallocation trap

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> v;
    std::cout << v.size() << " / " << v.capacity() << std::endl;   // 0 / 0

    for (int i = 0; i < 5; ++i) {
        v.push_back(i);
        std::cout << "size=" << v.size() << " capacity=" << v.capacity() << std::endl;
    }

    std::vector<int> w;
    w.reserve(1000);   // one allocation up front -- no reallocation for 1000 pushes
    std::cout << w.size() << " / " << w.capacity() << std::endl;   // 0 / 1000
}
// Output (capacity growth is implementation-defined; libstdc++ doubles):
// 0 / 0
// size=1 capacity=1
// size=2 capacity=2
// size=3 capacity=4
// size=4 capacity=4
// size=5 capacity=8
// 0 / 1000
```

`size()` is how many elements exist; `capacity()` is how many fit before the
vector must allocate a bigger block and move everything over. **When a vector
reallocates, every pointer, reference, and iterator into it becomes dangling.**

```cpp
std::vector<int> v{1, 2, 3};
int& ref = v[0];
v.push_back(4);       // may reallocate...
// std::cout << ref;  // ...and now 'ref' points at freed memory. Undefined behaviour.
```

If you know roughly how many elements you'll add, call `reserve()` first. It
avoids repeated copies *and* keeps references valid for that many pushes.

## Associative containers: map and set

```cpp
#include <iostream>
#include <map>
#include <set>
#include <string>

int main() {
    std::map<std::string, int> ages;
    ages["Ada"] = 36;
    ages["Grace"] = 45;
    ages.insert({"Alan", 41});

    // Iteration is ALWAYS in sorted key order
    for (const auto& [name, age] : ages) {        // C++17 structured bindings
        std::cout << name << " -> " << age << std::endl;
    }

    // Lookup
    if (ages.count("Ada")) {
        std::cout << "Ada is " << ages.at("Ada") << std::endl;
    }

    auto it = ages.find("Nobody");
    if (it == ages.end()) {
        std::cout << "Nobody not found" << std::endl;
    }

    std::set<int> unique{5, 1, 3, 1, 5};   // duplicates dropped
    for (int x : unique) std::cout << x << ' ';
    std::cout << std::endl;
}
// Output:
// Ada -> 36
// Alan -> 41
// Grace -> 45
// Ada is 36
// Nobody not found
// 1 3 5
```

`std::map` and `std::set` are balanced binary search trees (red-black trees in
practice). Everything is O(log n) and iteration comes out sorted for free.

**The `operator[]` trap on maps:**

```cpp
std::map<std::string, int> m;
std::cout << m.size() << std::endl;    // 0
if (m["missing"] == 0) { }             // inserts "missing" -> 0 as a side effect!
std::cout << m.size() << std::endl;    // 1  -- surprise
```

`operator[]` **default-constructs and inserts** a value if the key is absent.
That is convenient for counters (`counts[word]++` just works) and a bug
everywhere else. To read without inserting, use `at()` (throws
`std::out_of_range` on a missing key), `find()`, or `count()`. On a `const
map`, `operator[]` doesn't even compile — which is a good reason to pass maps
as `const&`.

## Unordered containers: hash tables

```cpp
#include <iostream>
#include <unordered_map>
#include <unordered_set>
#include <string>

int main() {
    std::unordered_map<std::string, int> counts;

    for (const std::string& word : {"apple", "pear", "apple", "fig", "apple"}) {
        counts[word]++;   // default-constructs to 0 on first sight, then increments
    }

    for (const auto& [word, n] : counts) {
        std::cout << word << ": " << n << std::endl;   // order is UNSPECIFIED
    }

    std::unordered_set<int> seen{1, 2, 3};
    std::cout << (seen.find(2) != seen.end()) << std::endl;   // 1
}
```

`unordered_map`/`unordered_set` give average O(1) lookup by hashing the key,
at the cost of losing sorted iteration. Rule of thumb:

- Need keys in sorted order, or range queries (`lower_bound`)? → `map`/`set`
- Just need fast lookup by key? → `unordered_map`/`unordered_set`

Worst case for a hash table is O(n) if every key collides, but the standard
hashers for built-in types and `std::string` are fine in practice.

## Container adaptors: stack, queue, priority_queue

```cpp
#include <iostream>
#include <stack>
#include <queue>

int main() {
    std::stack<int> s;              // LIFO -- built on std::deque by default
    s.push(1); s.push(2); s.push(3);
    std::cout << s.top() << std::endl;   // 3
    s.pop();
    std::cout << s.top() << std::endl;   // 2

    std::queue<std::string> q;      // FIFO
    q.push("first"); q.push("second");
    std::cout << q.front() << std::endl;  // first
    q.pop();

    std::priority_queue<int> pq;    // max-heap by default
    pq.push(3); pq.push(9); pq.push(5);
    std::cout << pq.top() << std::endl;   // 9

    // Min-heap: flip the comparator
    std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;
    minHeap.push(3); minHeap.push(9); minHeap.push(5);
    std::cout << minHeap.top() << std::endl;   // 3
}
```

These are **adaptors**: thin wrappers that restrict an underlying container to
a specific interface. Note `pop()` returns `void` — you call `top()` first,
then `pop()`. That split exists so a throwing copy constructor can't lose an
element mid-removal.

## Choosing a container

| Container | Lookup | Insert/erase at end | Insert/erase in middle | Ordered? | Contiguous? |
|-----------|--------|---------------------|------------------------|----------|-------------|
| `vector` | O(1) by index | O(1) amortised | O(n) | insertion order | yes |
| `deque` | O(1) by index | O(1) both ends | O(n) | insertion order | no |
| `list` | O(n) | O(1) | O(1) with iterator | insertion order | no |
| `map` / `set` | O(log n) by key | O(log n) | O(log n) | sorted by key | no |
| `unordered_map` / `unordered_set` | O(1) average | O(1) average | O(1) average | no | no |
| `array<T,N>` | O(1) by index | fixed size | fixed size | insertion order | yes |

## Iterator invalidation — the rules that bite

| Operation | What breaks |
|-----------|-------------|
| `vector` reallocation (`push_back` past capacity, `resize`, `reserve`) | **all** iterators, pointers, references |
| `vector::insert`/`erase` at position `p` | everything at or after `p` |
| `deque` insert/erase in the middle | all iterators; references too |
| `list` erase | only the iterator to the erased element |
| `map`/`set` erase | only the iterator to the erased element |
| `unordered_map` rehash (growth past load factor) | all **iterators**; references stay valid |

The safe erase idiom, since `erase` returns the next valid iterator:

```cpp
#include <vector>

std::vector<int> v{1, 2, 3, 4, 5, 6};

for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) {
        it = v.erase(it);   // erase returns the iterator to the NEXT element
    } else {
        ++it;               // only advance when we did NOT erase
    }
}
// v is now {1, 3, 5}
```

Incrementing after erasing (`v.erase(it); ++it;`) is undefined behaviour — a
bug that often "works" in testing and crashes in production.

## `emplace_back` vs `push_back`

```cpp
#include <vector>
#include <string>

struct Point { int x, y; Point(int x, int y) : x(x), y(y) {} };

std::vector<Point> pts;
pts.push_back(Point(1, 2));   // construct a temporary, then move it in
pts.emplace_back(1, 2);       // construct IN PLACE from the arguments -- no temporary
```

`emplace_back` forwards its arguments straight to the element's constructor,
skipping the temporary entirely. For cheap types the difference is negligible;
for types holding heap memory (`std::string`, other containers) it's real.

## Exercise

Write a word-frequency counter. Read all whitespace-separated words from a
`std::string` of text (use a `std::istringstream`), lowercase each one, and
count occurrences in a `std::unordered_map<std::string, int>`.

Then produce a report sorted by descending count (ties broken alphabetically):
copy the map's entries into a `std::vector<std::pair<std::string, int>>` and
sort it. Print the top 5.

Finally, do the same thing with `std::map` instead of `std::unordered_map` and
explain in a comment what changed about iteration order and why you still
needed the sorting step.
