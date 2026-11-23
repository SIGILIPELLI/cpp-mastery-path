---
title: "C++ Interview Questions and Answers (178 Questions)"
description: "178 C++ interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the C++ Mastery Path."
---

# C++ Interview Questions and Answers

**178 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free C++ Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** What does `g++ -std=c++20 hello.cpp -o hello` do?

**Answer:** Compiles hello.cpp as C++20 into an executable named hello

C++ compiles straight to native code with no VM.

**Q2.** What does `#include <iostream>` provide?

**Answer:** Declarations for std::cout and std::cin

It's a preprocessor include.

**Q3.** What does `return 0;` in main signal?

**Answer:** Successful exit status to the operating system

Non-zero indicates failure.

**Q4.** Why is C++ 'statically typed'?

**Answer:** Every variable's type is fixed at compile time

Type errors appear at compile time.

### [02 · Variables, Types &amp; Operators](level-1/02-variables-types-operators.md)

**Q5.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ int a=7,b=2; cout<<a/b<<' '<<a%b<<' '<<double(a)/b; }
```

**Answer:** 3 1 3.5

Integer division truncates. Verified by compiling and running.

**Q6.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ auto x=5; auto y=2.0; auto z=x*y; cout<<z<<' '<<sizeof(z); }
```

**Answer:** 10 8

int * double is double (8 bytes). Verified by compiling and running.

**Q7.** What does `const int n = 5;` guarantee?

**Answer:** n can't be modified after initialization

Prefer const by default.

**Q8.** Why prefer brace initialization `int x{3.5};`?

**Answer:** It rejects narrowing conversions at compile time

Narrowing becomes an error.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ int s=0; for(int i=1;i<=10;i++){ if(i%2==0) continue; if(i>7) break; s+=i; } cout<<s; }
```

**Answer:** 16

Odd numbers 1+3+5+7 = 16; 9 breaks. Verified by compiling and running.

**Q10.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ vector<int> v{1,2,3}; int t=0; for(auto x: v) t+=x*x; cout<<t; }
```

**Answer:** 14

Range-for over a vector. Verified by compiling and running.

**Q11.** What happens if a switch case lacks break?

**Answer:** Execution falls through to the next case

Use [[fallthrough]] to mark intentional cases.

**Q12.** Which loop runs at least once?

**Answer:** do { } while (cond);

The condition is checked after the body.

### [04 · Functions &amp; Overloading](level-1/04-functions-overloading.md)

**Q13.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
void f(int){cout<<"int";}
void f(double){cout<<"double";}
void f(const string&){cout<<"string";}
int main(){ f(1); f(1.0); f(string("a")); }
```

**Answer:** intdoublestring

The overload with the best-matching parameter type is chosen. Verified by compiling and running.

**Q14.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
void inc(int& x){ x++; }
int main(){ int a=1; inc(a); inc(a); cout<<a; }
```

**Answer:** 3

A reference parameter aliases the caller's variable. Verified by compiling and running.

**Q15.** Can you overload functions by return type alone?

**Answer:** No: parameter lists must differ

The return type isn't part of overload resolution.

**Q16.** What does a default argument `void f(int x = 5)` allow?

**Answer:** Calling f() with the default used for x

Defaults go on the right-most parameters.

### [05 · Arrays &amp; std::vector Basics](level-1/05-arrays-vector-basics.md)

**Q17.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ vector<int> v; for(int i=0;i<5;i++) v.push_back(i*i); cout<<v.size()<<' '<<v.back()<<' '<<v[2]; }
```

**Answer:** 5 16 4

push_back appends; indices start at 0. Verified by compiling and running.

**Q18.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ vector<int> v{1,2,3}; try{ v.at(5); }catch(const out_of_range&){ cout<<"caught"; } }
```

**Answer:** caught

at() checks bounds; operator[] does not. Verified by compiling and running.

**Q19.** What happens when a vector exceeds its capacity?

**Answer:** It allocates a bigger buffer, moves elements and invalidates iterators/pointers

Reserve to avoid reallocations.

**Q20.** Why prefer std::vector over a raw C array?

**Answer:** It tracks its size, manages memory and can grow

RAII manages the buffer.

### [06 · Strings (std::string)](level-1/06-strings.md)

**Q21.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ string s="hello"; s+=" world"; cout<<s.size()<<' '<<s.substr(6)<<' '<<s.find('o'); }
```

**Answer:** 11 world 4

find returns the first index. Verified by compiling and running.

**Q22.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ string a="abc",b="abd"; cout<<(a<b)<<(a=="abc"); }
```

**Answer:** 11

std::string compares lexicographically with ==, &lt;. Verified by compiling and running.

**Q23.** What does `std::string::npos` mean?

**Answer:** find found nothing

Compare find's result to npos.

**Q24.** Why is std::string safer than char arrays?

**Answer:** It manages memory and length, avoiding manual overflow bugs

No manual terminator handling.

### [07 · Classes &amp; Objects Basics](level-1/07-classes-objects.md)

**Q25.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
class C{ int n; public: C(int x):n(x){ cout<<"ctor"<<n; } ~C(){ cout<<"dtor"<<n; } };
int main(){ C a(1); { C b(2); } cout<<"end"; }
```

**Answer:** ctor1ctor2dtor2enddtor1

Objects are destroyed at the end of their scope, in reverse order. Verified by compiling and running.

**Q26.** What does `private` mean?

**Answer:** Members are accessible only inside the class (and friends)

Encapsulation hides representation.

**Q27.** What does `const` after a member function signature mean?

**Answer:** The function doesn't modify the object

const objects can call only const members.

**Q28.** What does the member initializer list do?

**Answer:** Initializes members before the constructor body runs

Required for const and reference members.

### [08 · References &amp; Pointers](level-1/08-references-pointers.md)

**Q29.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ int a=1; int& r=a; int* p=&a; r=5; *p+=1; cout<<a<<r<<*p; }
```

**Answer:** 666

r and p both refer to a. Verified by compiling and running.

**Q30.** How does a reference differ from a pointer?

**Answer:** It must be bound at initialization, can't be reseated and can't be null

Use pointers for optional or reseatable.

**Q31.** What does dereferencing nullptr do?

**Answer:** Undefined behavior

Check before use.

**Q32.** Why pass big objects by const reference?

**Answer:** It avoids copying while promising not to modify

A common default parameter style.

### [09 · Exception Handling Basics](level-1/09-exception-handling.md)

**Q33.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
#include <stdexcept>
int main(){ try{ throw runtime_error("boom"); }catch(const exception& e){ cout<<e.what(); } }
```

**Answer:** boom

Catch by const reference. Verified by compiling and running.

**Q34.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
#include <stdexcept>
struct R{ ~R(){ cout<<"cleanup "; } };
int main(){ try{ R r; throw 1; }catch(int){ cout<<"caught"; } }
```

**Answer:** cleanup caught

Stack unwinding destroys locals before the handler runs. Verified by compiling and running.

**Q35.** Why catch by const reference?

**Answer:** It avoids slicing and copying

Catch polymorphic exceptions by reference.

**Q36.** What does an uncaught exception do?

**Answer:** Calls std::terminate, ending the program

Handle exceptions where you can recover.

### [10 · Project — Bank Account CLI](level-1/10-project-bank-account-cli.md)

**Q37.** Why validate a withdrawal against the balance?

**Answer:** To keep the invariant that a balance never goes negative

Reject or throw on invalid operations.

**Q38.** Why store money in integer cents rather than double?

**Answer:** Floating-point can't represent many decimals exactly

Avoid rounding drift.

**Q39.** Why wrap the account's data as private?

**Answer:** So all changes go through methods that enforce the rules

Encapsulation.

**Q40.** How should invalid user input be handled?

**Answer:** Check the stream state and re-prompt instead of assuming success

cin can fail.

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

**Q41.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
struct B{ virtual void f(){cout<<"B";} virtual ~B()=default; };
struct D:B{ void f() override {cout<<"D";} };
int main(){ unique_ptr<B> p=make_unique<D>(); p->f(); }
```

**Answer:** D

A virtual call dispatches on the dynamic type. Verified by compiling and running.

**Q42.** What does this print (no virtual)?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
struct B{ void f(){cout<<"B";} };
struct D:B{ void f(){cout<<"D";} };
int main(){ D d; B* p=&d; p->f(); }
```

**Answer:** B

Without virtual, the static type decides. Verified by compiling and running.

**Q43.** Why give a polymorphic base class a virtual destructor?

**Answer:** Deleting a derived object through a base pointer would otherwise be undefined behavior

Make base destructors virtual.

**Q44.** What does `= 0` after a virtual function mean?

**Answer:** It's pure virtual, making the class abstract

Derived classes must implement it.

### [02 · Templates Basics](level-2/02-templates-basics.md)

**Q45.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
template<typename T> T mx(T a,T b){ return a>b?a:b; }
int main(){ cout<<mx(3,7)<<' '<<mx(2.5,1.5)<<' '<<mx<string>("a","b"); }
```

**Answer:** 7 2.5 b

The compiler generates a version per type. Verified by compiling and running.

**Q46.** When are templates instantiated?

**Answer:** At compile time for each type used

Errors can appear in instantiated code.

**Q47.** What does a class template look like?

**Answer:** template&lt;typename T&gt; class Stack { ... };

Use Stack&lt;int&gt; to instantiate.

**Q48.** Where are template definitions usually placed?

**Answer:** In headers, since the compiler needs the full definition to instantiate

Otherwise you get link errors.

### [03 · STL Containers](level-2/03-stl-containers.md)

**Q49.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ map<string,int> m; m["b"]=2; m["a"]=1; m["b"]+=5; for(auto&[k,v]:m) cout<<k<<v<<' '; }
```

**Answer:** a1 b7

std::map keeps keys sorted. Verified by compiling and running.

**Q50.** Which container gives average O(1) lookup by key?

**Answer:** std::unordered_map

map is O(log n) but ordered.

**Q51.** When is std::vector the right default?

**Answer:** When you want contiguous storage and fast iteration

It's the default sequence container.

**Q52.** What does map's operator[] do for a missing key?

**Answer:** Inserts a default-constructed value

Use find or at to avoid insertion.

### [04 · STL Algorithms &amp; Iterators](level-2/04-stl-algorithms-iterators.md)

**Q53.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
#include <algorithm>
#include <numeric>
int main(){ vector<int> v{5,2,9,1}; sort(v.begin(),v.end()); cout<<v[0]<<v[3]<<' '<<accumulate(v.begin(),v.end(),0)<<' '<<count_if(v.begin(),v.end(),[](int x){return x>2;}); }
```

**Answer:** 19 17 2

sort ascending; sum 17; two elements &gt; 2. Verified by compiling and running.

**Q54.** What do begin() and end() denote?

**Answer:** A half-open range: end points one past the last element

Never dereference end().

**Q55.** What does erase-remove idiom do?

**Answer:** remove shifts kept elements forward, then erase trims the tail

remove doesn't shrink the container.

**Q56.** Why prefer algorithms over hand-written loops?

**Answer:** They express intent and are well tested

Use &lt;algorithm&gt;.

### [05 · Operator Overloading](level-2/05-operator-overloading.md)

**Q57.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
struct V{ int x,y; V operator+(const V&o)const{ return {x+o.x,y+o.y}; } bool operator==(const V&o)const{ return x==o.x&&y==o.y; } };
int main(){ V a{1,2},b{3,4}; V c=a+b; cout<<c.x<<c.y<<(c==V{4,6}); }
```

**Answer:** 461

Operators are ordinary functions with special names. Verified by compiling and running.

**Q58.** Which operator is usually overloaded as a non-member for streams?

**Answer:** operator&lt;&lt; for std::ostream

The stream is the left operand.

**Q59.** What rule applies when overloading operators?

**Answer:** Keep their conventional meaning so code stays intuitive

Don't surprise users.

**Q60.** Why mark operator== const?

**Answer:** So it works on const objects

Comparison shouldn't modify.

### [06 · Smart Pointers](level-2/06-smart-pointers.md)

**Q61.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
int main(){ auto a=make_shared<int>(5); auto b=a; cout<<a.use_count(); b.reset(); cout<<a.use_count(); }
```

**Answer:** 21

shared_ptr counts owners. Verified by compiling and running.

**Q62.** What does unique_ptr express?

**Answer:** Exclusive ownership: movable but not copyable

Default smart pointer.

**Q63.** What problem does weak_ptr solve?

**Answer:** Breaking shared_ptr reference cycles

It doesn't own.

**Q64.** Why prefer make_unique over `new`?

**Answer:** It's exception-safe and avoids raw owning pointers

No manual delete.

### [07 · File I/O &amp; Streams](level-2/07-file-io-streams.md)

**Q65.** Which class reads from files?

**Answer:** std::ifstream

ofstream writes.

**Q66.** What should you check after opening a file?

**Answer:** Whether it opened successfully (e.g. if (!file))

Failed opens are silent otherwise.

**Q67.** What closes a std::fstream automatically?

**Answer:** Its destructor at the end of scope (RAII)

No explicit close is needed.

**Q68.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
using namespace std;
#include <sstream>
int main(){ istringstream in("3 4 5"); int a,b,c; in>>a>>b>>c; cout<<a+b+c; }
```

**Answer:** 12

String streams parse like files. Verified by compiling and running.

### [08 · Error Handling with Exceptions](level-2/08-error-handling-exceptions.md)

**Q69.** Which is a good use of exceptions?

**Answer:** Reporting failures a caller can't ignore or handle locally

Don't use them for ordinary control flow.

**Q70.** What does `noexcept` promise?

**Answer:** The function won't throw; if it does, std::terminate runs

Move operations should be noexcept.

**Q71.** What is RAII's role in exception safety?

**Answer:** Destructors release resources during stack unwinding

No leaks even when exceptions fly.

**Q72.** Why define custom exception types?

**Answer:** So callers can catch specific failures

Derive from std::exception.

### [09 · Build Tools (CMake Basics)](level-2/09-build-tools-cmake.md)

**Q73.** What does `target_link_libraries(app PRIVATE calc_lib)` do?

**Answer:** Links the library into app and uses its PUBLIC properties

CMake targets carry usage requirements.

**Q74.** What does PUBLIC mean for an include directory?

**Answer:** Anything linking the library also gets that include directory

PRIVATE is internal only.

**Q75.** Why build out of source (a separate build dir)?

**Answer:** It keeps generated files out of the source tree

cmake -S . -B build.

**Q76.** What generates the build system?

**Answer:** CMake generates Makefiles or Ninja files

A meta-build system.

### [10 · Project — Library Management System](level-2/10-project-library-management.md)

**Q77.** Why model Book and Member as separate classes?

**Answer:** Each has its own data and responsibilities (single responsibility)

It keeps code cohesive.

**Q78.** How should lending prevent double-lending?

**Answer:** Check the book's state and reject or throw on a second loan

Enforce invariants.

**Q79.** Which container suits lookup of books by ISBN?

**Answer:** An unordered_map keyed by ISBN

Average O(1) lookup.

**Q80.** Why use smart pointers for owned records?

**Answer:** To manage lifetimes without manual delete

Prefer unique_ptr.

## Level 3 · Advanced

### [01 · Advanced Templates](level-3/01-advanced-templates.md)

**Q81.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
template<typename T> struct IsInt{ static constexpr bool v=false; };
template<> struct IsInt<int>{ static constexpr bool v=true; };
int main(){ cout<<IsInt<int>::v<<IsInt<double>::v; }
```

**Answer:** 10

A specialization overrides the primary template for one type. Verified by compiling and running.

**Q82.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
template<typename... A> int count(A... a){ return sizeof...(a); }
int main(){ cout<<count()<<count(1)<<count(1,'a',2.0); }
```

**Answer:** 013

A variadic pack's size is known at compile time. Verified by compiling and running.

**Q83.** What does `if constexpr` do?

**Answer:** Discards the untaken branch at compile time inside a template

Branches may then use type-specific code.

**Q84.** What do concepts (C++20) provide?

**Answer:** Named constraints on template parameters with clearer errors

requires clauses restrict types.

### [02 · Move Semantics &amp; Rvalue References](level-3/02-move-semantics-rvalue-references.md)

**Q85.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
struct S{ S(){cout<<"c";} S(const S&){cout<<"copy";} S(S&&)noexcept{cout<<"move";} };
int main(){ S a; S b=move(a); S c=b; }
```

**Answer:** cmovecopy

std::move casts to an rvalue so the move constructor is chosen. Verified by compiling and running.

**Q86.** What does moving a std::vector do?

**Answer:** Transfers its buffer pointer instead of copying elements

The source is left valid but unspecified.

**Q87.** Why mark move constructors noexcept?

**Answer:** vector uses move on reallocation only if it can't throw

Otherwise it copies for safety.

**Q88.** What is the rule of five?

**Answer:** If you define a destructor, copy ctor/assign or move ctor/assign, consider defining all five

Or follow the rule of zero.

### [03 · Concurrency](level-3/03-concurrency.md)

**Q89.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
int main(){ int x=0; mutex m; auto f=[&]{ for(int i=0;i<1000;i++){ lock_guard<mutex> g(m); x++; } }; thread a(f),b(f); a.join(); b.join(); cout<<x; }
```

**Answer:** 2000

A mutex makes the increment safe. Verified by compiling and running.

**Q90.** What is a data race?

**Answer:** Unsynchronized concurrent access with at least one write: undefined behavior

Use mutexes or atomics.

**Q91.** Why prefer std::lock_guard over manual lock/unlock?

**Answer:** It unlocks automatically even if an exception is thrown (RAII)

Avoids leaks of the lock.

**Q92.** What does a std::thread destructor do if still joinable?

**Answer:** Calls std::terminate; join or detach first

std::jthread joins automatically.

### [04 · RAII Deep Dive](level-3/04-raii-deep-dive.md)

**Q93.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
struct G{ string n; G(string s):n(s){cout<<"+"<<n;} ~G(){cout<<"-"<<n;} };
int main(){ G a("a"); G b("b"); }
```

**Answer:** +a+b-b-a

Destruction runs in reverse construction order. Verified by compiling and running.

**Q94.** What is RAII?

**Answer:** Resource acquisition is initialization: tie a resource's lifetime to an object's

Locks, files and memory use it.

**Q95.** Why is RAII exception-safe?

**Answer:** Destructors run during stack unwinding, releasing resources

No manual cleanup paths.

**Q96.** What should a RAII class do about copying?

**Answer:** Delete or define copy semantics so ownership stays clear

Often move-only.

### [05 · Design Patterns in C++](level-3/05-design-patterns.md)

**Q97.** What does the Observer pattern provide?

**Answer:** Subjects notify registered observers of changes without tight coupling

Event systems use it.

**Q98.** What does the Strategy pattern do?

**Answer:** Swaps interchangeable algorithms behind a common interface

std::function can implement it.

**Q99.** What risk does a Singleton add?

**Answer:** Global state that makes testing and dependencies harder

Prefer injection.

**Q100.** What does the Factory pattern centralize?

**Answer:** Object creation, returning a base interface

Callers don't name concrete types.

### [06 · Networking Basics](level-3/06-networking-basics.md)

**Q101.** In a TCP client the order is...

**Answer:** socket, connect, send/recv, close

Servers use bind, listen and accept.

**Q102.** Why must you handle partial sends/reads?

**Answer:** TCP is a byte stream, and calls may transfer fewer bytes than requested

Loop until all bytes move.

**Q103.** What does htons do?

**Answer:** Converts a 16-bit value to network byte order

Network order is big-endian.

**Q104.** What's blocking I/O's downside for servers?

**Answer:** One slow client can stall a thread; use threads or event loops

Scale with epoll/kqueue or async.

### [07 · Working with SQLite from C++](level-3/07-sqlite.md)

**Q105.** Why use prepared statements with bound parameters?

**Answer:** They prevent SQL injection and reuse the parsed query

Never concatenate user input into SQL.

**Q106.** What does a transaction give you?

**Answer:** Atomic all-or-nothing changes

BEGIN ... COMMIT or ROLLBACK.

**Q107.** Why wrap sqlite3 handles in RAII?

**Answer:** To close/finalize them on every exit path

No leaks on exceptions.

**Q108.** What should you check after sqlite3_step?

**Answer:** Its return code (SQLITE_ROW, SQLITE_DONE or an error)

Handle errors.

### [08 · Testing with Google Test](level-3/08-testing-googletest.md)

**Q109.** What's the difference between EXPECT_EQ and ASSERT_EQ?

**Answer:** EXPECT continues after a failure; ASSERT aborts the current test

Use ASSERT when later code needs the check.

**Q110.** What does a test fixture do?

**Answer:** Shares setup/teardown across tests

TEST_F uses it.

**Q111.** What makes a good unit test?

**Answer:** Fast, isolated and deterministic

Flaky tests erode trust.

**Q112.** What do mocks replace?

**Answer:** Collaborators so you can test a unit in isolation

GoogleMock expects calls.

### [09 · Performance Profiling](level-3/09-performance-profiling.md)

**Q113.** What's the first rule of optimization?

**Answer:** Measure with a profiler and optimize the hot spot

Benchmark in release builds.

**Q114.** Why does cache locality matter?

**Answer:** Contiguous data (like vector) is faster to iterate than pointer-chasing

Prefer vector.

**Q115.** What does -O2 do?

**Answer:** Enables optimizations in the compiler

Profile optimized builds.

**Q116.** What's a pitfall of microbenchmarks?

**Answer:** The optimizer may eliminate the work; prevent that with sinks or benchmark libraries

Check what's measured.

### [10 · Project — Multi-threaded Task Processor](level-3/10-project-task-processor.md)

**Q117.** What synchronizes producers and consumers of a task queue?

**Answer:** A mutex plus a condition variable

The consumer waits for work.

**Q118.** Why should a worker wait in a loop on the condition variable?

**Answer:** Spurious wakeups mean the predicate must be rechecked

Use wait with a predicate.

**Q119.** How do you shut a thread pool down cleanly?

**Answer:** Signal stop, notify all workers and join them

Avoid dangling threads.

**Q120.** What should tasks never do?

**Answer:** Leak exceptions out of the worker thread, which would call std::terminate

Catch inside the worker.

## Level 4 · Master

### [01 · Modern C++ Features (17/20)](level-4/01-modern-cpp-features.md)

**Q121.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
#include <optional>
#include <variant>
int main(){ optional<int> o; cout<<o.value_or(7); variant<int,string> v=string("hi"); cout<<get<string>(v); auto [a,b]=pair{1,2}; cout<<a+b; }
```

**Answer:** 7hi3

optional, variant and structured bindings are C++17. Verified by compiling and running.

**Q122.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
#include <string_view>
int main(){ string_view sv="hello world"; cout<<sv.substr(0,5)<<sv.size(); }
```

**Answer:** hello11

string_view is a non-owning view. Verified by compiling and running.

**Q123.** What is the danger of string_view?

**Answer:** It can dangle if the underlying string is destroyed

Mind lifetimes.

**Q124.** What do ranges (C++20) add?

**Answer:** Composable lazy views over sequences

std::views::filter and transform.

### [02 · Advanced Concurrency](level-4/02-advanced-concurrency.md)

**Q125.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
#include <future>
int main(){ auto f=async(launch::async,[]{ return 6*7; }); cout<<f.get(); }
```

**Answer:** 42

future::get waits for the result. Verified by compiling and running.

**Q126.** What does std::atomic provide?

**Answer:** Lock-free (where available) indivisible operations on a variable

fetch_add avoids data races.

**Q127.** What is a deadlock?

**Answer:** Threads waiting on each other's locks forever; lock in a consistent order or use std::scoped_lock

scoped_lock locks several mutexes safely.

**Q128.** What does memory_order_relaxed guarantee?

**Answer:** Atomicity only, not ordering with other memory operations

Use stronger orders when ordering matters.

### [03 · Compile-time Computation](level-4/03-compile-time-computation.md)

**Q129.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
constexpr int fact(int n){ return n<=1?1:n*fact(n-1); }
int main(){ constexpr int f=fact(5); static_assert(f==120); cout<<f; }
```

**Answer:** 120

constexpr functions can run at compile time. Verified by compiling and running.

**Q130.** What does static_assert do?

**Answer:** Checks a condition at compile time and fails the build if false

Great for size and trait checks.

**Q131.** What does consteval require?

**Answer:** The function be evaluated at compile time

Immediate functions.

**Q132.** Why use constexpr tables?

**Answer:** They're computed at compile time with zero runtime cost

Moves work to build.

### [04 · Systems Programming Patterns](level-4/04-systems-programming-patterns.md)

**Q133.** What does mmap give you?

**Answer:** File contents mapped into memory for pointer-style access

Great for large read-mostly files.

**Q134.** Why use RAII wrappers for file descriptors?

**Answer:** To close them on every path

Avoid leaks.

**Q135.** What does epoll/kqueue enable?

**Answer:** Handling many sockets with one thread via event notification

Scalable servers.

**Q136.** What's a zero-copy goal?

**Answer:** Avoid copying buffers between layers

sendfile and views.

### [05 · Security Best Practices](level-4/05-security-best-practices.md)

**Q137.** Which is safer than strcpy?

**Answer:** std::string or bounded copies with sizes checked

Avoid raw buffers.

**Q138.** What compiler options help catch bugs?

**Answer:** -Wall -Wextra and sanitizers (-fsanitize=address,undefined)

Use in CI.

**Q139.** Why validate all external input?

**Answer:** Untrusted sizes and formats cause overflows and injection

Never trust the network or files.

**Q140.** What does `-fstack-protector` do?

**Answer:** Adds canaries that detect stack smashing

Hardening, not a fix.

### [06 · Large-Scale Build Systems](level-4/06-large-scale-build-systems.md)

**Q141.** Why use precompiled headers or unity builds?

**Answer:** To reduce compile time for large projects

Trade-offs exist.

**Q142.** What does ccache do?

**Answer:** Caches compiler outputs keyed by inputs to skip rebuilds

Speeds incremental and CI builds.

**Q143.** Why prefer target-based CMake?

**Answer:** Usage requirements propagate cleanly through dependencies

Avoid global include_directories.

**Q144.** What does Ninja improve over Make?

**Answer:** Faster incremental builds and parallelism

CMake can generate Ninja.

### [07 · Cross-platform Development](level-4/07-cross-platform-development.md)

**Q145.** Why avoid platform-specific headers in shared code?

**Answer:** They break builds elsewhere; isolate them behind an interface

Use #ifdef sparingly.

**Q146.** What differs between platforms that bites binary file formats?

**Answer:** Endianness and type sizes

Use fixed-width types.

**Q147.** What does &lt;filesystem&gt; provide?

**Answer:** A portable API for paths and directories

C++17.

**Q148.** How do you verify cross-platform support?

**Answer:** Build and test on each target in CI

Matrix builds.

### [08 · Testing at Scale &amp; CI](level-4/08-testing-at-scale-ci.md)

**Q149.** What should block a merge in CI?

**Answer:** Failing build, tests and sanitizer/lint checks

Make the gate automatic.

**Q150.** Why run tests under sanitizers?

**Answer:** They find memory errors and UB that normal runs miss

ASan, UBSan, TSan.

**Q151.** What is test flakiness?

**Answer:** Tests that pass and fail without code changes

Fix or quarantine.

**Q152.** What does code coverage tell you?

**Answer:** Which code ran during tests, not whether behavior was checked

Strong asserts matter.

### [09 · Performance at Scale](level-4/09-performance-at-scale.md)

**Q153.** What's usually the biggest win for data-heavy code?

**Answer:** Better algorithms and data layout (contiguous, cache-friendly)

Measure first.

**Q154.** What is false sharing?

**Answer:** Threads writing different variables on the same cache line, hurting performance

Pad or separate hot data.

**Q155.** What does reserve() on a vector avoid?

**Answer:** Repeated reallocations while growing

Know sizes up front.

**Q156.** When is parallelism worthwhile?

**Answer:** When work is big enough to outweigh synchronization overhead

Amdahl's law limits gains.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What does the capstone integrate?

**Answer:** Templates, RAII, concurrency, testing, CMake and profiling in one project

It ties the course together.

**Q158.** What's a sound structure for a C++ project?

**Answer:** A library target, an executable and tests, built with CMake

Separate concerns.

**Q159.** How should errors flow in the design?

**Answer:** Exceptions or error types with RAII cleanup, documented per API

Consistency.

**Q160.** How do you validate the finished project?

**Answer:** Tests, sanitizers, profiling and CI on clean builds

Layered checks.

## Mixed questions across all levels

**Q161.** What does `g++ -std=c++20 main.cpp -o app` produce?

**Answer:** A native executable named app, compiled directly from C++ source

C++ has no VM.

**Q162.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
int main(){ int a=7,b=2; cout<<a/b<<' '<<double(a)/b; }
```

**Answer:** 3 3.5

Integer division truncates. Verified by compiling and running.

**Q163.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
void f(int){cout<<'i';} void f(double){cout<<'d';}
int main(){ f(1); f(1.5); f('a'); }
```

**Answer:** idi

char promotes to int, which is the better match. Verified by compiling and running.

**Q164.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
struct B{ virtual void f(){cout<<'B';} virtual ~B()=default; };
struct D:B{ void f() override {cout<<'D';} };
int main(){ B* p=new D; p->f(); delete p; }
```

**Answer:** D

Virtual dispatch uses the dynamic type. Verified by compiling and running.

**Q165.** A base class has no virtual destructor and you `delete` a Derived through a Base*. What happens?

**Answer:** Undefined behavior: the derived destructor may not run

Give polymorphic bases a virtual destructor.

**Q166.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
int main(){ vector<int> v{1,2,3}; for(int i=0;i<3;i++) v.push_back(v[i]*2); cout<<v.size()<<v.back(); }
```

**Answer:** 66

The vector grows to 6 elements and the last is 3*2 = 6. Verified by compiling and running.

**Q167.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
struct G{ string n; G(string s):n(s){cout<<'+'<<n;} ~G(){cout<<'-'<<n;} };
int main(){ G a("a"); { G b("b"); } G c("c"); }
```

**Answer:** +a+b-b+c-c-a

Scope exit destroys b; c and a are destroyed in reverse order. Verified by compiling and running.

**Q168.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
int main(){ auto p=make_shared<int>(1); auto q=p; weak_ptr<int> w=p; p.reset(); cout<<q.use_count()<<w.expired(); q.reset(); cout<<w.expired(); }
```

**Answer:** 101

One owner remains, then none: the weak_ptr expires. Verified by compiling and running.

**Q169.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
struct S{ S(){cout<<'c';} S(const S&){cout<<'k';} S(S&&)noexcept{cout<<'m';} };
S make(){ S s; return s; }
int main(){ S a=make(); S b=move(a); S c=b; }
```

**Answer:** cmk

Returning a local is elided; move and copy are chosen by value category. Verified by compiling and running.

**Q170.** A std::vector&lt;T&gt; reallocates. Which T operation matters for performance?

**Answer:** A noexcept move constructor, so elements are moved rather than copied

vector uses move_if_noexcept.

**Q171.** Which is the correct way to protect a shared counter incremented by many threads?

**Answer:** Use std::mutex with lock_guard, or std::atomic

Otherwise it's a data race (undefined behavior).

**Q172.** What does this print?

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <map>
#include <thread>
#include <mutex>
#include <atomic>
using namespace std;
int main(){ atomic<int> n{0}; vector<thread> t; for(int i=0;i<4;i++) t.emplace_back([&]{ for(int j=0;j<1000;j++) n++; }); for(auto&x:t) x.join(); cout<<n; }
```

**Answer:** 4000

atomic increments can't lose updates. Verified by compiling and running.

**Q173.** A std::thread goes out of scope while still joinable. What happens?

**Answer:** std::terminate is called; join or detach it first (or use std::jthread)

Always manage thread lifetimes.

**Q174.** Why wait on a condition variable with a predicate?

**Answer:** Spurious wakeups and missed notifications mean the condition must be rechecked

cv.wait(lock, pred).

**Q175.** A function template is defined only in a .cpp file but used elsewhere. What happens?

**Answer:** Typically a link error: the compiler needs the definition to instantiate it

Put template definitions in headers.

**Q176.** What does `if constexpr` do inside a template?

**Answer:** Discards the untaken branch at compile time

Branches can use type-specific code.

**Q177.** Which tooling combination best catches memory and UB bugs in tests?

**Answer:** Compile with -fsanitize=address,undefined and run the test suite

Use them in CI.

**Q178.** How do you target-link a library so dependents also get its include path in CMake?

**Answer:** target_include_directories(lib PUBLIC include) and target_link_libraries(app PRIVATE lib)

Usage requirements propagate.

## Practical coding questions

### A move-only RAII handle

Complete the Handle class: print `open <name>` on construction, delete copying, implement a move constructor and move assignment so ownership transfers, and print `close <name>` exactly once for each resource still owned when destroyed or overwritten. The provided main must print the expected output.

Starter code:

```
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

Expected output:

```text
open a
open c
close a
end
close c
```

??? success "Show a solution"

    ```
    #include <iostream>
    #include <utility>
    #include <string>
    using namespace std;

    class Handle {
        string name_;
        bool owns_;
    public:
        explicit Handle(string n) : name_(move(n)), owns_(true) { cout << "open " << name_ << "\n"; }
        Handle(const Handle&) = delete;
        Handle& operator=(const Handle&) = delete;
        Handle(Handle&& o) noexcept : name_(move(o.name_)), owns_(o.owns_) { o.owns_ = false; }
        Handle& operator=(Handle&& o) noexcept {
            if (this != &o) {
                release();
                name_ = move(o.name_);
                owns_ = o.owns_;
                o.owns_ = false;
            }
            return *this;
        }
        ~Handle() { release(); }
    private:
        void release() { if (owns_) { cout << "close " << name_ << "\n"; owns_ = false; } }
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

    Moving transfers ownership by clearing the source's owns_ flag, so each resource is closed exactly once. Move assignment releases the current resource first. Verified by compiling and running the solution.

### A thread-safe counter

Make Counter safe for four threads each adding 1 twenty-five thousand times. Protect the shared value with a mutex (use RAII locking), keep get() const-correct, and print the total.

Starter code:

```
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

Expected output:

```text
100000
```

??? success "Show a solution"

    ```
    #include <iostream>
    #include <thread>
    #include <mutex>
    #include <vector>
    using namespace std;

    class Counter {
        mutable mutex m_;
        long value_ = 0;
    public:
        void add(long n) {
            lock_guard<mutex> g(m_);
            value_ += n;
        }
        long get() const {
            lock_guard<mutex> g(m_);
            return value_;
        }
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

    Without a lock the increments race and the total is unpredictable. lock_guard releases the mutex on every path, and the mutex is mutable so a const get() can lock it. Verified by compiling and running the solution.

---

More free practice: the [C++ Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
