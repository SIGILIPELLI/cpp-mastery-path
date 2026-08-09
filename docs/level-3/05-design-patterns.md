# 05 · Design Patterns in C++

Design patterns are named solutions to recurring design problems — a shared
vocabulary, not a library you import. What makes them worth a dedicated
module here is that C++ has *idiomatic* ways to implement the classics that
differ from the textbook Java/C# versions: smart pointers replace manual
memory management, and RAII (from [Module 4](04-raii-deep-dive.md)) removes
whole categories of cleanup bugs the original pattern descriptions had to
worry about. This module covers three of the most common patterns —
Singleton, Factory, and Observer — the modern C++ way.

## Singleton — exactly one instance, safely

The intent is simple: exactly one instance of a class exists, reachable from
anywhere. The classic danger is a hand-rolled version that's either not
thread-safe (two threads both see "not created yet" and both construct it) or
leaks the instance. C++11 solved this in the language itself:

```cpp
#include <iostream>

class Logger {
public:
    static Logger& instance() {
        static Logger instance;   // constructed on first call, guaranteed thread-safe by the standard
        return instance;
    }

    void log(const std::string& message) { std::cout << "[LOG] " << message << std::endl; }

    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;

private:
    Logger() { std::cout << "Logger constructed" << std::endl; }
};

int main() {
    std::cout << "before first log call" << std::endl;
    Logger::instance().log("starting up");
    Logger::instance().log("still the same instance");
}
// before first log call
// Logger constructed
// [LOG] starting up
// [LOG] still the same instance
```

This is called a **Meyer's Singleton**. Since C++11, the standard guarantees
that initialization of a function-local `static` is thread-safe: if two
threads call `instance()` for the first time simultaneously, the runtime
inserts locking so the constructor runs exactly once and every caller waits
for it, with no extra code from you. It's also lazy (nothing is built until
first use) and leak-free at a predictable time (destroyed in reverse order of
construction at program exit, like any static).

Prefer this over a Singleton reachable through a raw global pointer or a
manually-locked "check a bool, then construct" pattern — both are strictly
worse versions of what `static` already gives you for free.

## Factory Method — decouple "what to build" from "how to use it"

A factory hides the concrete type behind an interface, so calling code
depends only on a base class, not on which derived class gets constructed:

```cpp
#include <iostream>
#include <memory>
#include <string>

class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual std::string name() const = 0;
};

class Circle : public Shape {
public:
    explicit Circle(double radius) : radius_(radius) {}
    double area() const override { return 3.14159 * radius_ * radius_; }
    std::string name() const override { return "Circle"; }
private:
    double radius_;
};

class Square : public Shape {
public:
    explicit Square(double side) : side_(side) {}
    double area() const override { return side_ * side_; }
    std::string name() const override { return "Square"; }
private:
    double side_;
};

// The factory function: callers ask for a kind of shape by name and get
// back an owning pointer to the interface -- they never name Circle or
// Square directly.
std::unique_ptr<Shape> makeShape(const std::string& kind, double size) {
    if (kind == "circle") return std::make_unique<Circle>(size);
    if (kind == "square") return std::make_unique<Square>(size);
    throw std::invalid_argument("unknown shape kind: " + kind);
}

int main() {
    for (const auto& kind : {"circle", "square"}) {
        std::unique_ptr<Shape> shape = makeShape(kind, 2.0);
        std::cout << shape->name() << " area = " << shape->area() << std::endl;
    }
}
// Circle area = 12.5664
// Square area = 4
```

Returning `std::unique_ptr<Shape>` instead of a raw `Shape*` is what makes
this modern-C++-idiomatic: ownership of the new object transfers to the
caller unambiguously, and there's no way to forget to `delete` it — the
factory's job is choosing *which* type to build, not managing its lifetime,
and `unique_ptr` handles the lifetime automatically. Adding a new shape means
adding one `if` branch (or, for a large family, a `std::unordered_map<std::string,
std::function<...>>` registry) — call sites never change.

## Observer — notify a set of listeners without knowing who they are

The Observer pattern lets a "subject" broadcast events to any number of
"observers" that registered interest, without the subject holding
type-specific knowledge of them:

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

class Observer {
public:
    virtual ~Observer() = default;
    virtual void onTemperatureChanged(double celsius) = 0;
};

class TemperatureSensor {
public:
    void addObserver(std::shared_ptr<Observer> obs) { observers_.push_back(std::move(obs)); }

    void setTemperature(double celsius) {
        current_ = celsius;
        for (const auto& obs : observers_) obs->onTemperatureChanged(current_);
    }

private:
    double current_ = 0.0;
    std::vector<std::shared_ptr<Observer>> observers_;
};

class Display : public Observer {
public:
    explicit Display(std::string label) : label_(std::move(label)) {}
    void onTemperatureChanged(double celsius) override {
        std::cout << label_ << " now shows " << celsius << "C" << std::endl;
    }
private:
    std::string label_;
};

int main() {
    TemperatureSensor sensor;
    sensor.addObserver(std::make_shared<Display>("Kitchen display"));
    sensor.addObserver(std::make_shared<Display>("Phone app"));

    sensor.setTemperature(21.5);
    sensor.setTemperature(23.0);
}
// Kitchen display now shows 21.5C
// Phone app now shows 21.5C
// Kitchen display now shows 23C
// Phone app now shows 23C
```

Storing `std::shared_ptr<Observer>` rather than raw pointers means the
subject can share ownership of observers with whoever else holds them,
without either side worrying about who deletes what. If observers instead
need to outlive the subject's interest in them *without* being kept alive
purely by that registration (a common cause of surprising lifetime
extension), store `std::weak_ptr<Observer>` instead and `.lock()` it before
calling — skipping any observer whose `lock()` returns null because it's
already been destroyed elsewhere.

## Cheat sheet

| Pattern | Problem it solves | Modern C++ idiom |
|---------|-------------------|------------------|
| Singleton | Exactly one instance, globally reachable | Function-local `static` (Meyer's Singleton) |
| Factory Method | Decouple construction from usage | Function returning `std::unique_ptr<Base>` |
| Observer | Broadcast events to unknown listeners | `std::vector<std::shared_ptr<Observer>>` + virtual callback |

## Traps

**A Singleton is a global with extra ceremony.** It's the right tool for a
handful of truly-one-of-these things (a logger, a hardware register map) —
reaching for it as a default way to avoid passing dependencies around makes
code harder to test, since every user of the Singleton is now implicitly
coupled to it.

**A factory that returns a raw pointer** pushes the "who deletes this?"
question back onto every call site. Returning `std::unique_ptr` (or
`std::shared_ptr` when shared ownership is genuinely needed) answers the
question once, in the factory itself.

**An Observer holding `shared_ptr`s to observers that also hold a `shared_ptr`
back to the subject** is a reference cycle — neither side's reference count
ever reaches zero, and both leak. If a back-reference is needed, make it a
`weak_ptr`, exactly as with the cycles discussed in
[Level 2's smart pointers module](../level-2/06-smart-pointers.md).

## Exercise

Extend the shape factory with a `Triangle` (base and height) and register it
in `makeShape`. Then give `Shape` a pure virtual `describe()` that returns a
formatted string, and write a free function `totalArea` that takes a
`const std::vector<std::unique_ptr<Shape>>&` and sums their areas using a
range-based `for` loop. Finally, add a second observer type, `Logger`, that
appends every temperature reading to a `std::vector<double>` instead of
printing, and confirm both a `Display` and a `Logger` can watch the same
`TemperatureSensor` at once.
