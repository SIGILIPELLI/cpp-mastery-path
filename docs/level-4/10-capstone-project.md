# 10 · Capstone Project

This is the last module of the path, and it is a real service rather than an
exercise: **kvcache**, a networked, thread-safe, LRU key-value cache with TTL
expiry, a text wire protocol, a unit test suite, a CMake package, and a CI
matrix. A miniature memcached.

Every level shows up. Classes and RAII from Level 1; STL containers, smart
ownership and CMake from Level 2; concurrency, sockets and Google Test from
Level 3; `shared_mutex`, `std::optional`, sanitizers, CTest and GitHub Actions
from Level 4. The architectural point of the design is one idea worth carrying
into your own work: **separate the logic from the I/O**, so the interesting
parts can be tested without a network.

## What you'll build

```text
$ kvcached 9099 1024 60          # port, capacity, TTL seconds

SET user:1 Ada Lovelace   ->  OK
GET user:1                ->  VALUE Ada Lovelace
GET nobody                ->  NOT_FOUND
DEL user:1                ->  DELETED
STATS                     ->  STATS hits=1 misses=1 evictions=0 expirations=0 size=0
```

- O(1) `get`/`put`/`erase` via a hash map into an intrusive LRU list
- Per-entry TTL; expired entries are evicted lazily on access
- Capacity-bounded with least-recently-used eviction
- Thread-safe with `std::shared_mutex`
- Line-oriented protocol over TCP, hardened against malformed input
- Hit/miss/eviction/expiration counters exposed via `STATS`

## Project layout

```text
kvcache/
    CMakeLists.txt
    include/kvcache/
        lru_cache.h      # the cache -- header-only, no I/O
        protocol.h       # command types + parser declaration
        server.h         # command application + TCP server
    src/
        protocol.cpp
        server.cpp
        main.cpp
    tests/
        test_lru_cache.cpp
        test_protocol.cpp
    .github/workflows/ci.yml
```

## include/kvcache/lru_cache.h

The data structure is the classic pairing: a `std::list` holding entries in
recency order, plus an `unordered_map` from key to **iterator into that list**.
`std::list` iterators are never invalidated by insertion or by erasing other
elements, which is exactly the property that makes this work — and one of the
few places where `std::list` is genuinely the right container.

```cpp
#pragma once

#include <chrono>
#include <list>
#include <optional>
#include <shared_mutex>
#include <string>
#include <unordered_map>

namespace kvcache {

struct Stats {
    std::size_t hits = 0, misses = 0, evictions = 0, expirations = 0, size = 0;
};

class LruCache {
public:
    using Clock = std::chrono::steady_clock;

    explicit LruCache(std::size_t capacity,
                      Clock::duration ttl = std::chrono::seconds(60))
        : capacity_(capacity == 0 ? 1 : capacity), ttl_(ttl) {}

    void put(const std::string& key, std::string value) {
        std::unique_lock lock(mutex_);
        auto it = index_.find(key);
        if (it != index_.end()) {                       // update in place
            it->second->value = std::move(value);
            it->second->expiresAt = Clock::now() + ttl_;
            order_.splice(order_.begin(), order_, it->second);   // O(1) move to front
            return;
        }
        order_.push_front(Entry{key, std::move(value), Clock::now() + ttl_});
        index_[key] = order_.begin();
        if (index_.size() > capacity_) evictOldest();
    }

    std::optional<std::string> get(const std::string& key) {
        std::unique_lock lock(mutex_);                  // NOT shared: get() mutates order
        auto it = index_.find(key);
        if (it == index_.end()) { ++stats_.misses; return std::nullopt; }

        if (Clock::now() >= it->second->expiresAt) {    // lazy expiry
            order_.erase(it->second);
            index_.erase(it);
            ++stats_.expirations; ++stats_.misses;
            return std::nullopt;
        }
        order_.splice(order_.begin(), order_, it->second);
        ++stats_.hits;
        return it->second->value;
    }

    bool erase(const std::string& key) {
        std::unique_lock lock(mutex_);
        auto it = index_.find(key);
        if (it == index_.end()) return false;
        order_.erase(it->second);
        index_.erase(it);
        return true;
    }

    Stats stats() const {
        std::shared_lock lock(mutex_);                  // genuinely read-only
        Stats s = stats_;
        s.size = index_.size();
        return s;
    }

    std::size_t size() const {
        std::shared_lock lock(mutex_);
        return index_.size();
    }

private:
    struct Entry {
        std::string key;                                // needed to erase from index_
        std::string value;
        Clock::time_point expiresAt;
    };

    void evictOldest() {                                // caller holds the lock
        const Entry& oldest = order_.back();
        index_.erase(oldest.key);
        order_.pop_back();
        ++stats_.evictions;
    }

    mutable std::shared_mutex mutex_;
    std::size_t capacity_;
    Clock::duration ttl_;
    std::list<Entry> order_;
    std::unordered_map<std::string, std::list<Entry>::iterator> index_;
    Stats stats_;
};

}  // namespace kvcache
```

Three decisions worth defending:

**`get()` takes a `unique_lock`, not a `shared_lock`.** It looks like a read,
but it splices the entry to the front and bumps a counter — it writes. Using
`shared_lock` here would be a data race that passes every functional test and
fails under TSan. Only `stats()` and `size()` are true readers.

**`Entry` stores its own key.** It looks redundant against the map key, but
`evictOldest()` needs to erase from `index_` and only has the list node.

**Expiry is lazy.** No background thread scans for expired entries; they are
detected on access. This keeps the design simple and means an untouched expired
entry occupies memory until it is either accessed or evicted by capacity
pressure — a real trade-off you should be able to state out loud.

## include/kvcache/protocol.h and src/protocol.cpp

The parser is pure: `std::string_view` in, `std::optional<Command>` out. No
sockets, no cache, no globals — so it is trivially unit-testable and trivially
fuzzable.

```cpp
namespace kvcache {

enum class Verb { Get, Set, Del, Stats };

struct Command {
    Verb verb;
    std::string key;
    std::string value;
};

// GET <key> | SET <key> <value...> | DEL <key> | STATS
std::optional<Command> parseCommand(std::string_view line);

}
```

```cpp
std::optional<Command> parseCommand(std::string_view line) {
    line = trim(line);                        // strips spaces, \r and \n
    if (line.empty()) return std::nullopt;

    const std::size_t sp1 = line.find(' ');
    const std::string verb = upper(line.substr(0, sp1));

    if (verb == "STATS") {
        if (sp1 != std::string_view::npos) return std::nullopt;   // no arguments allowed
        return Command{Verb::Stats, "", ""};
    }
    if (sp1 == std::string_view::npos) return std::nullopt;       // every other verb needs a key

    std::string_view rest = trim(line.substr(sp1 + 1));
    if (rest.empty()) return std::nullopt;

    const std::size_t sp2 = rest.find(' ');
    std::string key(rest.substr(0, sp2));
    if (key.empty() || key.size() > 250) return std::nullopt;     // bound it

    if (verb == "GET") return sp2 == std::string_view::npos
        ? std::optional{Command{Verb::Get, std::move(key), ""}} : std::nullopt;
    if (verb == "DEL") return sp2 == std::string_view::npos
        ? std::optional{Command{Verb::Del, std::move(key), ""}} : std::nullopt;
    if (verb == "SET") {
        if (sp2 == std::string_view::npos) return std::nullopt;
        std::string value(trim(rest.substr(sp2 + 1)));
        if (value.empty()) return std::nullopt;
        return Command{Verb::Set, std::move(key), std::move(value)};
    }
    return std::nullopt;                                          // unknown verb
}
```

Every failure path returns `std::nullopt` rather than throwing or returning a
partially-filled struct. That's the [security](05-security-best-practices.md)
discipline: validate completely at the boundary, and hand the rest of the
program a type that can only hold a valid command. The 250-byte key limit is
there because "the client controls the length" is how memory exhaustion starts.

## include/kvcache/server.h — logic first, sockets second

```cpp
// Pure: applies one parsed command. No I/O -- this is what tests call.
std::string applyCommand(LruCache& cache, const Command& cmd);

// Parse + apply, including the malformed case.
std::string handleLine(LruCache& cache, std::string_view line);

class Server {
public:
    Server(LruCache& cache, unsigned short port, unsigned workers);
    ~Server();                                  // calls stop()
    Server(const Server&) = delete;             // owns a socket: not copyable
    Server& operator=(const Server&) = delete;

    void run();                                 // blocking accept loop
    void stop();
    unsigned short port() const { return port_; }

private:
    void handleClient(int clientFd);
    LruCache& cache_;
    unsigned short port_;
    unsigned workers_;
    int listenFd_ = -1;
    std::atomic<bool> running_{false};
    std::vector<std::thread> threads_;
};
```

`handleLine` is the seam. Everything a user would call the service to do is
reachable through a function taking a `string_view` and returning a `string`, so
the entire request/response behaviour is testable with no socket, no port
allocation, and no timing.

## src/server.cpp — the sockets

```cpp
std::string applyCommand(LruCache& cache, const Command& cmd) {
    switch (cmd.verb) {
        case Verb::Set:
            cache.put(cmd.key, cmd.value);
            return "OK\r\n";
        case Verb::Get: {
            auto v = cache.get(cmd.key);
            return v ? "VALUE " + *v + "\r\n" : "NOT_FOUND\r\n";
        }
        case Verb::Del:
            return cache.erase(cmd.key) ? "DELETED\r\n" : "NOT_FOUND\r\n";
        case Verb::Stats: {
            Stats s = cache.stats();
            std::ostringstream os;
            os << "STATS hits=" << s.hits << " misses=" << s.misses
               << " evictions=" << s.evictions << " expirations=" << s.expirations
               << " size=" << s.size << "\r\n";
            return os.str();
        }
    }
    return "ERROR unknown verb\r\n";
}

std::string handleLine(LruCache& cache, std::string_view line) {
    auto cmd = parseCommand(line);
    if (!cmd) return "ERROR bad command\r\n";
    return applyCommand(cache, *cmd);
}
```

```cpp
void Server::run() {
    listenFd_ = ::socket(AF_INET, SOCK_STREAM, 0);
    if (listenFd_ < 0) throw std::runtime_error("socket() failed");

    int yes = 1;
    ::setsockopt(listenFd_, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);
    //  ^ without this, restarting the server fails with EADDRINUSE for ~60s
    //    while the old socket sits in TIME_WAIT

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);   // loopback only -- not 0.0.0.0
    addr.sin_port = htons(port_);

    if (::bind(listenFd_, reinterpret_cast<sockaddr*>(&addr), sizeof addr) < 0)
        throw std::runtime_error("bind() failed on port " + std::to_string(port_));
    if (::listen(listenFd_, 64) < 0)
        throw std::runtime_error("listen() failed");

    socklen_t len = sizeof addr;
    ::getsockname(listenFd_, reinterpret_cast<sockaddr*>(&addr), &len);
    port_ = ntohs(addr.sin_port);      // resolves port 0 -> the OS-assigned port

    running_ = true;
    while (running_) {
        int client = ::accept(listenFd_, nullptr, nullptr);
        if (client < 0) break;
        threads_.emplace_back(&Server::handleClient, this, client);
        if (threads_.size() >= workers_) {
            for (auto& t : threads_) if (t.joinable()) t.join();
            threads_.clear();
        }
    }
}

void Server::handleClient(int clientFd) {
    std::string buffer;
    char chunk[4096];
    ssize_t n;
    while ((n = ::recv(clientFd, chunk, sizeof chunk, 0)) > 0) {
        buffer.append(chunk, static_cast<std::size_t>(n));
        std::size_t pos;
        while ((pos = buffer.find('\n')) != std::string::npos) {
            std::string line = buffer.substr(0, pos);
            buffer.erase(0, pos + 1);
            std::string reply = handleLine(cache_, line);
            ::send(clientFd, reply.data(), reply.size(), 0);
        }
        if (buffer.size() > 64 * 1024) break;   // refuse an unbounded line
    }
    ::close(clientFd);
}
```

The buffering loop is the part people get wrong. **TCP is a byte stream, not a
message stream**: one `recv` may return half a command, or three commands, or
two and a half. The only correct approach is to append to a buffer and consume
*complete* framed units — here, everything up to a `\n`. The 64 KB cap is what
stops a client that never sends a newline from growing your memory without
bound.

Binding to `INADDR_LOOPBACK` rather than `INADDR_ANY` is deliberate: a cache
with no authentication must not be reachable from the network. This is the
actual cause of a long history of exposed-memcached incidents.

## Building it

```cmake
cmake_minimum_required(VERSION 3.20)
project(kvcache VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

find_package(Threads REQUIRED)

add_library(kvcache_lib src/protocol.cpp src/server.cpp)
add_library(kvcache::lib ALIAS kvcache_lib)
target_include_directories(kvcache_lib PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>)
target_link_libraries(kvcache_lib PUBLIC Threads::Threads)
target_compile_options(kvcache_lib PRIVATE -Wall -Wextra -Wpedantic)

add_executable(kvcached src/main.cpp)
target_link_libraries(kvcached PRIVATE kvcache::lib)

option(KVCACHE_BUILD_TESTS "Build tests" ON)
if(KVCACHE_BUILD_TESTS)
    enable_testing()
    find_package(GTest REQUIRED)
    add_executable(kvcache_tests tests/test_lru_cache.cpp tests/test_protocol.cpp)
    target_link_libraries(kvcache_tests PRIVATE kvcache::lib GTest::gtest GTest::gtest_main)
    include(GoogleTest)
    gtest_discover_tests(kvcache_tests PROPERTIES LABELS "unit" TIMEOUT 30)
endif()
```

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build
```

```text
[ 12%] Building CXX object CMakeFiles/kvcache_lib.dir/src/protocol.cpp.o
[ 25%] Building CXX object CMakeFiles/kvcache_lib.dir/src/server.cpp.o
[ 37%] Linking CXX static library libkvcache_lib.a
[ 50%] Building CXX object CMakeFiles/kvcached.dir/src/main.cpp.o
[ 62%] Linking CXX executable kvcached
[ 75%] Building CXX object CMakeFiles/kvcache_tests.dir/tests/test_lru_cache.cpp.o
[ 87%] Building CXX object CMakeFiles/kvcache_tests.dir/tests/test_protocol.cpp.o
[100%] Linking CXX executable kvcache_tests
```

## The test suite

```cpp
TEST(LruCacheTest, EvictsLeastRecentlyUsed) {
    LruCache cache(3);
    cache.put("a", "1"); cache.put("b", "2"); cache.put("c", "3");
    (void)cache.get("a");            // "a" is now most recent; "b" is oldest
    cache.put("d", "4");             // over capacity -> evict "b"

    EXPECT_TRUE(cache.get("a").has_value());
    EXPECT_FALSE(cache.get("b").has_value());     // the eviction victim
    EXPECT_EQ(cache.stats().evictions, 1u);
    EXPECT_EQ(cache.size(), 3u);
}

TEST(LruCacheTest, ExpiredEntryIsAMiss) {
    LruCache cache(4, std::chrono::milliseconds(20));
    cache.put("a", "1");
    std::this_thread::sleep_for(std::chrono::milliseconds(40));
    EXPECT_FALSE(cache.get("a").has_value());
    EXPECT_EQ(cache.stats().expirations, 1u);
    EXPECT_EQ(cache.size(), 0u);
}

TEST(LruCacheTest, ConcurrentAccessIsSafe) {
    LruCache cache(256);
    std::vector<std::thread> ts;
    for (int t = 0; t < 8; ++t)
        ts.emplace_back([&, t] {
            for (int i = 0; i < 5000; ++i) {
                std::string k = "k" + std::to_string((t * 7 + i) % 300);
                cache.put(k, std::to_string(i));
                (void)cache.get(k);
            }
        });
    for (auto& t : ts) t.join();
    EXPECT_LE(cache.size(), 256u);        // capacity invariant held under load
    EXPECT_GT(cache.stats().hits, 0u);
}
```

Malformed input gets a parameterized sweep, so each bad string is its own
reported case:

```cpp
class MalformedTest : public ::testing::TestWithParam<const char*> {};
TEST_P(MalformedTest, RejectsGarbage) {
    EXPECT_FALSE(parseCommand(GetParam()).has_value());
}
INSTANTIATE_TEST_SUITE_P(BadInputs, MalformedTest, ::testing::Values(
    "", "   ", "GET", "GET a b", "SET", "SET onlykey", "DEL", "STATS extra",
    "BOGUS x", "\r\n"));

TEST(HandleLineTest, RoundTripsThroughCache) {
    LruCache cache(8);
    EXPECT_EQ(handleLine(cache, "SET name ada"), "OK\r\n");
    EXPECT_EQ(handleLine(cache, "GET name"),     "VALUE ada\r\n");
    EXPECT_EQ(handleLine(cache, "GET missing"),  "NOT_FOUND\r\n");
    EXPECT_EQ(handleLine(cache, "DEL name"),     "DELETED\r\n");
    EXPECT_EQ(handleLine(cache, "nonsense"),     "ERROR bad command\r\n");
}
```

```bash
cd build && ctest -j8 --output-on-failure
```

```text
...
20/21 Test  #7: LruCacheTest.ConcurrentAccessIsSafe ....................   Passed    0.03 sec
21/21 Test  #5: LruCacheTest.ExpiredEntryIsAMiss .......................   Passed    0.05 sec

100% tests passed out of 21

Label Time Summary:
unit    =   0.19 sec*proc (21 tests)

Total Test time (real) =   0.05 sec
```

Twenty-one cases in 50 ms, and the entire request/response surface is covered
without opening a socket. That is the dividend of the logic/I/O split.

## Running it

Start the daemon with a deliberately tiny capacity of 3 so eviction is visible:

```bash
./build/kvcached 9099 3 60
```

```text
kvcache listening on 127.0.0.1:9099 with 8 workers
```

Then drive it with `nc`:

```bash
printf 'SET user:1 Ada Lovelace\nGET user:1\nGET nobody\nSET user:2 Grace Hopper\nSET user:3 Alan Turing\nSET user:4 Ken Thompson\nGET user:1\nDEL user:3\nDEL user:3\nSTATS\nbogus command here\n' | nc 127.0.0.1 9099
```

```text
OK
VALUE Ada Lovelace
NOT_FOUND
OK
OK
OK
NOT_FOUND
DELETED
NOT_FOUND
STATS hits=1 misses=2 evictions=1 expirations=0 size=2
ERROR bad command
```

Trace the seventh line, because it is the whole design working. `GET user:1`
succeeded at the top and fails here. Capacity is 3; after `SET user:2`,
`user:3`, `user:4`, the cache held `{4, 3, 2}` and `user:1` — least recently
used despite its earlier read — was evicted. `STATS` confirms it:
`evictions=1`, and `size=2` because `DEL user:3` then removed another. Eleven
commands arrived in a single TCP segment and were framed correctly into eleven
responses.

## Continuous integration

```yaml
# .github/workflows/ci.yml
name: ci
on: [push, pull_request]

jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        include:
          - { os: ubuntu-latest, name: asan,    flags: "-fsanitize=address,undefined -g -O1" }
          - { os: ubuntu-latest, name: tsan,    flags: "-fsanitize=thread -g -O1" }
          - { os: ubuntu-latest, name: release, flags: "-O2" }
          - { os: macos-latest,  name: release, flags: "-O2" }
    runs-on: ${{ matrix.os }}
    name: ${{ matrix.os }} / ${{ matrix.name }}
    steps:
      - uses: actions/checkout@v4
      - name: Install GoogleTest
        run: |
          if [ "$RUNNER_OS" = "Linux" ]; then sudo apt-get update && sudo apt-get install -y libgtest-dev cmake
          else brew install googletest; fi
      - run: cmake -S . -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_CXX_FLAGS="${{ matrix.flags }}"
      - run: cmake --build build --parallel
      - run: ctest --test-dir build -j4 --output-on-failure
      - name: Flake check
        run: ctest --test-dir build -L unit --repeat until-fail:20
```

The TSan job is not decoration. `ConcurrentAccessIsSafe` passing once proves
very little; passing twenty times under ThreadSanitizer is what justifies the
claim in the class's name. If you change `get()` to take a `shared_lock` —
which looks correct and passes every functional test — this is the job that
turns red.

## Stretch goals

- **Replace thread-per-connection with the thread pool** from
  [Level 3 Module 10](../level-3/10-project-task-processor.md), then go further
  and make the accept loop event-driven with `epoll`/`kqueue`. Measure
  connections-per-second before and after with a load generator; thread-per-
  connection collapses somewhere around a few thousand concurrent clients and
  you should be able to show exactly where.
- **Shard the cache.** One `shared_mutex` is a single contention point. Split
  into 16 independent `LruCache` instances indexed by `std::hash<std::string>{}(key) % 16`
  and measure throughput at 1, 4, 8 and 16 threads — this is the lock-sharding
  technique from [Module 9](09-performance-at-scale.md) applied to real code.
- **Fuzz the parser.** Write an `LLVMFuzzerTestOneInput` around `parseCommand`,
  build with `-fsanitize=fuzzer,address`, and run it for ten minutes. Commit
  every crashing input as a regression test. Then fuzz `handleLine` so the cache
  itself is in the loop.
- **Add persistence.** Append every mutating command to a log file, and replay
  it at startup. Then add a periodic compaction that rewrites the log from the
  current cache contents. Use `std::filesystem` and binary-mode streams so it
  works on Windows too ([Module 7](07-cross-platform-development.md)).
- **Add `INCR`, `TTL <key>`, and `KEYS <prefix>`.** Each one forces a design
  decision: `INCR` needs atomic read-modify-write inside the lock, `TTL` needs
  remaining-lifetime arithmetic, and `KEYS` needs you to decide what a scan does
  to LRU order (it should not touch it — write the test that proves it doesn't).
- **Package and install it.** Add `install()` rules with an export set, then
  build a separate consumer project that does `find_package(kvcache REQUIRED)`
  and links `kvcache::lib` ([Module 6](06-large-scale-build-systems.md)).
- **Benchmark it properly.** Add a Google Benchmark target measuring
  `put`/`get` throughput at various hit ratios and capacities, with
  `DoNotOptimize`. Record baselines and add a CI job that fails on a regression
  beyond 10%.
- **Harden it.** Add a maximum value size, a per-connection command rate limit,
  and an optional `AUTH <token>` handshake using the constant-time comparison
  from [Module 5](05-security-best-practices.md). Then justify, in a paragraph,
  why binding to loopback is still the most important security control in the
  whole program.

## You've finished the path

Four levels, forty modules, from `std::cout << "Hello"` to a concurrent network
service with a sanitizer matrix. The C++ that matters from here is not another
language feature — it is judgement: knowing which of these tools a problem
actually calls for, and being willing to measure instead of assume.

Build something with it.
