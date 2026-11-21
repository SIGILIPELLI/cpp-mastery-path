---
title: "Final Quiz"
description: "Test what you have learned across all four levels of C++ Mastery Path with a 18-question final quiz."
---

# Final Quiz

18 questions drawn from all four levels of this course. You need **70% (13 out of 18)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does `g++ -std=c++20 main.cpp -o app` produce?",
    "options": [
      "An interpreted script",
      "A bytecode file",
      "A library only",
      "A native executable named app, compiled directly from C++ source"
    ],
    "correctIndex": 3,
    "explanation": "C++ has no VM."
  },
  {
    "question": "What does this print?\n```cpp\n#include <iostream>\n#include <vector>\n#include <string>\n#include <memory>\n#include <map>\n#include <thread>\n#include <mutex>\n#include <atomic>\nusing namespace std;\nint main(){ int a=7,b=2; cout<<a/b<<' '<<double(a)/b; }\n```\n",
    "options": [
      "3.5 3.5",
      "3 3",
      "3 3.5",
      "4 3.5"
    ],
    "correctIndex": 2,
    "explanation": "Integer division truncates. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```cpp\n#include <iostream>\n#include <vector>\n#include <string>\n#include <memory>\n#include <map>\n#include <thread>\n#include <mutex>\n#include <atomic>\nusing namespace std;\nvoid f(int){cout<<'i';} void f(double){cout<<'d';}\nint main(){ f(1); f(1.5); f('a'); }\n```\n",
    "options": [
      "ddd",
      "idi",
      "Compile error",
      "iii"
    ],
    "correctIndex": 1,
    "explanation": "char promotes to int, which is the better match. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```cpp\n#include <iostream>\n#include <vector>\n#include <string>\n#include <memory>\n#include <map>\n#include <thread>\n#include <mutex>\n#include <atomic>\nusing namespace std;\nstruct B{ virtual void f(){cout<<'B';} virtual ~B()=default; };\nstruct D:B{ void f() override {cout<<'D';} };\nint main(){ B* p=new D; p->f(); delete p; }\n```\n",
    "options": [
      "BD",
      "Compile error",
      "D",
      "B"
    ],
    "correctIndex": 2,
    "explanation": "Virtual dispatch uses the dynamic type. Verified by compiling and running."
  },
  {
    "question": "A base class has no virtual destructor and you `delete` a Derived through a Base*. What happens?",
    "options": [
      "It works",
      "Undefined behavior: the derived destructor may not run",
      "Memory leak only",
      "Compile error"
    ],
    "correctIndex": 1,
    "explanation": "Give polymorphic bases a virtual destructor."
  },
  {
    "question": "What does this print?\n```cpp\n#include <iostream>\n#include <vector>\n#include <string>\n#include <memory>\n#include <map>\n#include <thread>\n#include <mutex>\n#include <atomic>\nusing namespace std;\nint main(){ vector<int> v{1,2,3}; for(int i=0;i<3;i++) v.push_back(v[i]*2); cout<<v.size()<<v.back(); }\n```\n",
    "options": [
      "66",
      "64",
      "Crash",
      "36"
    ],
    "correctIndex": 0,
    "explanation": "The vector grows to 6 elements and the last is 3*2 = 6. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```cpp\n#include <iostream>\n#include <vector>\n#include <string>\n#include <memory>\n#include <map>\n#include <thread>\n#include <mutex>\n#include <atomic>\nusing namespace std;\nstruct G{ string n; G(string s):n(s){cout<<'+'<<n;} ~G(){cout<<'-'<<n;} };\nint main(){ G a(\"a\"); { G b(\"b\"); } G c(\"c\"); }\n```\n",
    "options": [
      "+a+b-b-a+c-c",
      "+a+b+c-a-b-c",
      "+a-a+b-b+c-c",
      "+a+b-b+c-c-a"
    ],
    "correctIndex": 3,
    "explanation": "Scope exit destroys b; c and a are destroyed in reverse order. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```cpp\n#include <iostream>\n#include <vector>\n#include <string>\n#include <memory>\n#include <map>\n#include <thread>\n#include <mutex>\n#include <atomic>\nusing namespace std;\nint main(){ auto p=make_shared<int>(1); auto q=p; weak_ptr<int> w=p; p.reset(); cout<<q.use_count()<<w.expired(); q.reset(); cout<<w.expired(); }\n```\n",
    "options": [
      "201",
      "111",
      "100",
      "101"
    ],
    "correctIndex": 3,
    "explanation": "One owner remains, then none: the weak_ptr expires. Verified by compiling and running."
  },
  {
    "question": "What does this print?\n```cpp\n#include <iostream>\n#include <vector>\n#include <string>\n#include <memory>\n#include <map>\n#include <thread>\n#include <mutex>\n#include <atomic>\nusing namespace std;\nstruct S{ S(){cout<<'c';} S(const S&){cout<<'k';} S(S&&)noexcept{cout<<'m';} };\nS make(){ S s; return s; }\nint main(){ S a=make(); S b=move(a); S c=b; }\n```\n",
    "options": [
      "ckkk",
      "cmm",
      "cckm",
      "cmk"
    ],
    "correctIndex": 3,
    "explanation": "Returning a local is elided; move and copy are chosen by value category. Verified by compiling and running."
  },
  {
    "question": "A std::vector<T> reallocates. Which T operation matters for performance?",
    "options": [
      "operator==",
      "The default constructor",
      "The destructor",
      "A noexcept move constructor, so elements are moved rather than copied"
    ],
    "correctIndex": 3,
    "explanation": "vector uses move_if_noexcept."
  },
  {
    "question": "Which is the correct way to protect a shared counter incremented by many threads?",
    "options": [
      "Use sleep",
      "Use std::mutex with lock_guard, or std::atomic",
      "Nothing, ints are safe",
      "Use volatile"
    ],
    "correctIndex": 1,
    "explanation": "Otherwise it's a data race (undefined behavior)."
  },
  {
    "question": "What does this print?\n```cpp\n#include <iostream>\n#include <vector>\n#include <string>\n#include <memory>\n#include <map>\n#include <thread>\n#include <mutex>\n#include <atomic>\nusing namespace std;\nint main(){ atomic<int> n{0}; vector<thread> t; for(int i=0;i<4;i++) t.emplace_back([&]{ for(int j=0;j<1000;j++) n++; }); for(auto&x:t) x.join(); cout<<n; }\n```\n",
    "options": [
      "4000",
      "Undefined",
      "0",
      "Less than 4000"
    ],
    "correctIndex": 0,
    "explanation": "atomic increments can't lose updates. Verified by compiling and running."
  },
  {
    "question": "A std::thread goes out of scope while still joinable. What happens?",
    "options": [
      "std::terminate is called; join or detach it first (or use std::jthread)",
      "It joins automatically",
      "Nothing",
      "It's detached"
    ],
    "correctIndex": 0,
    "explanation": "Always manage thread lifetimes."
  },
  {
    "question": "Why wait on a condition variable with a predicate?",
    "options": [
      "Style",
      "Required by syntax",
      "Spurious wakeups and missed notifications mean the condition must be rechecked",
      "Speed"
    ],
    "correctIndex": 2,
    "explanation": "cv.wait(lock, pred)."
  },
  {
    "question": "A function template is defined only in a .cpp file but used elsewhere. What happens?",
    "options": [
      "A warning",
      "It's slower",
      "Typically a link error: the compiler needs the definition to instantiate it",
      "It works"
    ],
    "correctIndex": 2,
    "explanation": "Put template definitions in headers."
  },
  {
    "question": "What does `if constexpr` do inside a template?",
    "options": [
      "Defers to runtime",
      "Discards the untaken branch at compile time",
      "Checks types at runtime",
      "Runs a loop"
    ],
    "correctIndex": 1,
    "explanation": "Branches can use type-specific code."
  },
  {
    "question": "Which tooling combination best catches memory and UB bugs in tests?",
    "options": [
      "Compile with -fsanitize=address,undefined and run the test suite",
      "-w",
      "-s",
      "-O3"
    ],
    "correctIndex": 0,
    "explanation": "Use them in CI."
  },
  {
    "question": "How do you target-link a library so dependents also get its include path in CMake?",
    "options": [
      "Use #include absolute paths",
      "Copy headers",
      "target_include_directories(lib PUBLIC include) and target_link_libraries(app PRIVATE lib)",
      "include_directories globally"
    ],
    "correctIndex": 2,
    "explanation": "Usage requirements propagate."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A move-only RAII handle

Complete the Handle class: print `open <name>` on construction, delete copying, implement a move constructor and move assignment so ownership transfers, and print `close <name>` exactly once for each resource still owned when destroyed or overwritten. The provided main must print the expected output.

**Starter code**

```cpp
#include <iostream>
#include <utility>
#include <string>
using namespace std;

class Handle {
    string name_;
public:
    // TODO: constructor printing "open <name>", move-only semantics
    // (delete copy, define move ctor and move assignment), and a destructor
    // that prints "close <name>" exactly once for each resource that is still owned.
    explicit Handle(string n) : name_(move(n)) {}
};

Handle make(const string& n) { return Handle(n); }

int main() {
    Handle a("a");
    Handle b = move(a);
    {
        Handle c = make("c");
        b = move(c);
    }
    cout << "end\n";
}
```

**Expected output**

```text
open a
open c
close a
end
close c
```

<details><summary>Show solution and explanation</summary>

<pre><code>#include &lt;iostream&gt;
#include &lt;utility&gt;
#include &lt;string&gt;
using namespace std;

class Handle {
    string name_;
    bool owns_;
public:
    explicit Handle(string n) : name_(move(n)), owns_(true) { cout &lt;&lt; &quot;open &quot; &lt;&lt; name_ &lt;&lt; &quot;\n&quot;; }
    Handle(const Handle&amp;) = delete;
    Handle&amp; operator=(const Handle&amp;) = delete;
    Handle(Handle&amp;&amp; o) noexcept : name_(move(o.name_)), owns_(o.owns_) { o.owns_ = false; }
    Handle&amp; operator=(Handle&amp;&amp; o) noexcept {
        if (this != &amp;o) {
            release();
            name_ = move(o.name_);
            owns_ = o.owns_;
            o.owns_ = false;
        }
        return *this;
    }
    ~Handle() { release(); }
private:
    void release() { if (owns_) { cout &lt;&lt; &quot;close &quot; &lt;&lt; name_ &lt;&lt; &quot;\n&quot;; owns_ = false; } }
};

Handle make(const string&amp; n) { return Handle(n); }

int main() {
    Handle a(&quot;a&quot;);
    Handle b = move(a);
    {
        Handle c = make(&quot;c&quot;);
        b = move(c);
    }
    cout &lt;&lt; &quot;end\n&quot;;
}
</code></pre>

<p>Moving transfers ownership by clearing the source&#x27;s owns_ flag, so each resource is closed exactly once. Move assignment releases the current resource first. Verified by compiling and running the solution.</p>
</details>

### A thread-safe counter

Make Counter safe for four threads each adding 1 twenty-five thousand times. Protect the shared value with a mutex (use RAII locking), keep get() const-correct, and print the total.

**Starter code**

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>
using namespace std;

class Counter {
    long value_ = 0;
public:
    // TODO: make add(long) and get() safe to call from several threads at once
    void add(long n) { value_ += n; }
    long get() const { return value_; }
};

int main() {
    Counter c;
    vector<thread> ts;
    for (int t = 0; t < 4; t++)
        ts.emplace_back([&c] { for (int i = 0; i < 25000; i++) c.add(1); });
    for (auto& t : ts) t.join();
    cout << c.get() << "\n";
}
```

**Expected output**

```text
100000
```

<details><summary>Show solution and explanation</summary>

<pre><code>#include &lt;iostream&gt;
#include &lt;thread&gt;
#include &lt;mutex&gt;
#include &lt;vector&gt;
using namespace std;

class Counter {
    mutable mutex m_;
    long value_ = 0;
public:
    void add(long n) {
        lock_guard&lt;mutex&gt; g(m_);
        value_ += n;
    }
    long get() const {
        lock_guard&lt;mutex&gt; g(m_);
        return value_;
    }
};

int main() {
    Counter c;
    vector&lt;thread&gt; ts;
    for (int t = 0; t &lt; 4; t++)
        ts.emplace_back([&amp;c] { for (int i = 0; i &lt; 25000; i++) c.add(1); });
    for (auto&amp; t : ts) t.join();
    cout &lt;&lt; c.get() &lt;&lt; &quot;\n&quot;;
}
</code></pre>

<p>Without a lock the increments race and the total is unpredictable. lock_guard releases the mutex on every path, and the mutex is mutable so a const get() can lock it. Verified by compiling and running the solution.</p>
</details>
