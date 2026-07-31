# 01 · OOP Deep Dive

[Level 1 Module 7](../level-1/07-classes-objects.md) covered a single class in
isolation. Real programs need classes that *relate* to each other: a `Circle`
and a `Square` that are both `Shape`s, a `FileLogger` that can stand in
anywhere a `Logger` is expected. That is what inheritance and polymorphism
give you — one interface, many implementations, with the decision about which
implementation runs deferred until runtime.

## Inheritance basics

```cpp
#include <iostream>
#include <string>

class Animal {
public:
    Animal(std::string name) : name(name) {}

    void eat() const {
        std::cout << name << " is eating." << std::endl;
    }

protected:
    std::string name;   // accessible in this class AND in derived classes
};

// Dog "is-a" Animal
class Dog : public Animal {
public:
    Dog(std::string name) : Animal(name) {}   // must initialise the base part

    void bark() const {
        std::cout << name << " says Woof!" << std::endl;   // 'name' is protected, so visible here
    }
};

int main() {
    Dog rex("Rex");
    rex.eat();    // inherited from Animal
    rex.bark();   // Dog's own method
}
// Output:
// Rex is eating.
// Rex says Woof!
```

Three access specifiers now matter, not two:

| Specifier | Inside the class | In derived classes | Outside |
|-----------|------------------|--------------------|---------|
| `public` | yes | yes | yes |
| `protected` | yes | yes | no |
| `private` | yes | **no** | no |

The `: public Animal` part is the **inheritance mode**. `public` inheritance
means "Dog is-a Animal" and is what you want almost every time. `private`
inheritance means "Dog is implemented in terms of Animal" and is rare — prefer
composition (holding an `Animal` member) for that.

A derived constructor must construct the base subobject first. If you don't
name it in the initializer list, the compiler calls the base's default
constructor — and fails to compile if there isn't one, which is exactly what
happens above if you omit `: Animal(name)`.

## Virtual functions and polymorphism

Without `virtual`, C++ picks which function to call based on the *static* type
of the expression — the type the compiler sees, not the object actually there.

```cpp
#include <iostream>

class Shape {
public:
    virtual double area() const { return 0.0; }   // virtual => runtime dispatch
    void describe() const {                       // NOT virtual
        std::cout << "A generic shape" << std::endl;
    }
    virtual ~Shape() = default;                   // see "the two big traps" below
};

class Circle : public Shape {
public:
    Circle(double r) : r(r) {}
    double area() const override { return 3.14159265 * r * r; }
    void describe() const { std::cout << "A circle" << std::endl; }
private:
    double r;
};

int main() {
    Circle c(2.0);
    Shape& s = c;   // a Shape reference bound to a Circle object

    std::cout << s.area() << std::endl;   // 12.5664 -- virtual: calls Circle::area
    s.describe();                         // "A generic shape" -- NOT virtual: calls Shape's
}
```

That second call is the classic trap. `describe` is resolved at compile time
from the declared type `Shape&`, so you get the base version even though the
object really is a `Circle`. Mark anything you intend to override `virtual`.

The `override` keyword is not decoration — it makes the compiler verify that
you really are overriding something. Misspell the name, get the `const` wrong,
or change a parameter type, and you silently create a brand-new function
instead of an override. With `override`, that becomes a compile error. Use it
on every override you write.

## Abstract classes and pure virtual functions

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <string>

class Shape {
public:
    virtual double area() const = 0;          // pure virtual -- no implementation here
    virtual std::string name() const = 0;
    virtual ~Shape() = default;

    // A non-virtual helper implemented in terms of the virtual interface
    void report() const {
        std::cout << name() << " has area " << area() << std::endl;
    }
};

class Rectangle : public Shape {
public:
    Rectangle(double w, double h) : w(w), h(h) {}
    double area() const override { return w * h; }
    std::string name() const override { return "Rectangle"; }
private:
    double w, h;
};

class Circle : public Shape {
public:
    Circle(double r) : r(r) {}
    double area() const override { return 3.14159265 * r * r; }
    std::string name() const override { return "Circle"; }
private:
    double r;
};

int main() {
    // Shape s;   // compile error: cannot instantiate an abstract class

    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Rectangle>(3.0, 4.0));
    shapes.push_back(std::make_unique<Circle>(1.0));

    double total = 0.0;
    for (const auto& shape : shapes) {
        shape->report();          // runtime dispatch to the right override
        total += shape->area();
    }
    std::cout << "Total area: " << total << std::endl;
}
// Output:
// Rectangle has area 12
// Circle has area 3.14159
// Total area: 15.1416
```

`= 0` makes a function **pure virtual**, which makes the class **abstract**:
you cannot create an object of it, only of a derived class that implements
every pure virtual. This is how C++ expresses an interface.

Note the container type: `std::vector<std::unique_ptr<Shape>>`, not
`std::vector<Shape>`. You cannot store polymorphic objects by value in a
container of the base type — see the next section for why. Smart pointers get
their own treatment in [Module 6](06-smart-pointers.md).

## The two big traps: slicing and missing virtual destructors

```cpp
#include <iostream>

class Base {
public:
    virtual void speak() const { std::cout << "Base" << std::endl; }
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    void speak() const override { std::cout << "Derived" << std::endl; }
    int extraData = 42;
};

void byValue(Base b)            { b.speak(); }   // SLICES the argument
void byReference(const Base& b) { b.speak(); }   // no slicing

int main() {
    Derived d;
    byValue(d);        // "Base"    -- the Derived part was chopped off
    byReference(d);    // "Derived" -- correct
}
```

**Object slicing**: copying a `Derived` into a `Base`-typed variable copies
only the `Base` subobject. `extraData` is gone and virtual dispatch reverts to
the base. The rule that follows: **pass polymorphic types by reference or
pointer, never by value.** ([Module 8 of Level 1](../level-1/08-references-pointers.md)
covers the reference syntax itself.)

```cpp
class BadBase {
public:
    virtual void work() {}
    ~BadBase() { std::cout << "~BadBase" << std::endl; }   // NOT virtual -- bug
};

class BadDerived : public BadBase {
public:
    ~BadDerived() { std::cout << "~BadDerived" << std::endl; }
};

int main() {
    BadBase* p = new BadDerived();
    delete p;   // undefined behaviour: ~BadDerived never runs, its members leak
}
// Output (typical):
// ~BadBase
```

If you ever delete a derived object through a base pointer, the base
destructor **must** be virtual. Rule of thumb: any class with a virtual
function should also have a virtual destructor (or a `protected` non-virtual
one, if you never intend polymorphic deletion). `virtual ~Shape() = default;`
costs one line and prevents a whole family of leaks.

## `final`, and how dispatch actually works

```cpp
class Widget {
public:
    virtual void draw() const;
    virtual ~Widget() = default;
};

class Button final : public Widget {   // nobody may derive from Button
public:
    void draw() const override;
};
```

Under the hood, each polymorphic class gets a **vtable** — a table of function
pointers — and each object stores a hidden pointer to it. A virtual call costs
one extra pointer indirection: read the vtable pointer, index into it, call.
That is cheap, but it usually blocks inlining. `final` lets the compiler prove
no further overrides exist and *devirtualise* the call back into a direct one.

## Calling the base implementation

Overriding does not have to mean replacing:

```cpp
#include <iostream>
#include <string>

class Employee {
public:
    Employee(std::string name, double base) : name(name), base(base) {}
    virtual double pay() const { return base; }
    virtual ~Employee() = default;
protected:
    std::string name;
    double base;
};

class Manager : public Employee {
public:
    Manager(std::string name, double base, double bonus)
        : Employee(name, base), bonus(bonus) {}

    double pay() const override {
        return Employee::pay() + bonus;   // explicit qualification calls the base version
    }
private:
    double bonus;
};

int main() {
    Manager m("Ada", 5000, 1500);
    Employee& e = m;
    std::cout << e.pay() << std::endl;   // 6500
}
```

`Employee::pay()` with the explicit class qualifier bypasses virtual dispatch
and calls that specific implementation — the standard way to extend rather
than replace behaviour.

## Cheat sheet

| Concept | Syntax | What it buys you |
|---------|--------|------------------|
| Public inheritance | `class D : public B` | "D is-a B"; D usable wherever B is |
| Virtual function | `virtual void f();` | Runtime dispatch on the real object type |
| Override | `void f() override;` | Compiler-checked override |
| Pure virtual | `virtual void f() = 0;` | Abstract class / interface |
| Virtual destructor | `virtual ~B() = default;` | Safe `delete` through a base pointer |
| `final` | `class D final` / `void f() final` | Blocks further derivation; enables devirtualisation |
| `protected` | `protected: int x;` | Visible to derived classes, not to the outside |

## Exercise

Build a small shape hierarchy. Define an abstract class `Shape` with pure
virtual `double area() const`, `double perimeter() const`, and
`std::string name() const`, plus a virtual destructor. Derive `Rectangle`,
`Circle`, and `Triangle` (use Heron's formula for the triangle's area). Store
several of them in a `std::vector<std::unique_ptr<Shape>>`, print a table of
name / area / perimeter, and report which shape has the largest area.

Then break it deliberately: remove `virtual` from `~Shape`, add a
`std::vector<double>` member to one derived class, and run the program with
`g++ -std=c++17 -fsanitize=address shapes.cpp -o shapes`. Observe the reported
leak, put `virtual` back, and confirm it disappears.
