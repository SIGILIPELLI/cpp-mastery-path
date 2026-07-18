# 07 · Classes & Objects Basics

A class is a blueprint; an object is an instance created from that blueprint.

## Members and constructors

```cpp
#include <iostream>
#include <string>

class Person {
public:
    // fields ("member variables" / "data members")
    std::string name;
    int age;

    // constructor -- runs when you create a new Person
    Person(std::string name, int age) {
        this->name = name;   // "this->name" is the member, "name" is the parameter
        this->age = age;
    }

    void introduce() const {
        std::cout << "Hi, I'm " << name << ", age " << age << std::endl;
    }
};

int main() {
    Person alice("Alice", 30);
    Person bob("Bob", 25);

    alice.introduce();   // Hi, I'm Alice, age 30
    bob.introduce();     // Hi, I'm Bob, age 25

    std::cout << alice.name << std::endl;   // Alice -- direct member access
}
```

`this` is a pointer to the current instance — `this->name` disambiguates a
member from a parameter of the same name. The `const` after `introduce()`
promises the method won't modify the object's members; the compiler enforces
this.

## Constructor initializer lists

```cpp
class Point {
public:
    double x;
    double y;

    // initializer list -- preferred over assigning in the constructor body
    Point(double x, double y) : x(x), y(y) {
    }

    double distanceFromOrigin() const {
        return std::sqrt(x * x + y * y);
    }
};
```

The `: x(x), y(y)` syntax is a **member initializer list** — it initializes
members directly as the object is constructed, rather than default-constructing
them and then assigning inside the body. For simple members like `double` the
difference is minor, but for class-typed members (like a `std::string` or
another object) it avoids an unnecessary default construction, and it's
required for `const` members and references. Prefer initializer lists as your
default style.

## Multiple constructors (overloading)

```cpp
class Rectangle {
public:
    double width;
    double height;

    Rectangle(double width, double height) : width(width), height(height) {
    }

    // Overloaded constructor -- a square is a rectangle with equal sides
    Rectangle(double side) : Rectangle(side, side) {   // delegates to the other constructor
    }

    double area() const {
        return width * height;
    }
};

int main() {
    Rectangle r(4, 5);
    Rectangle square(3);
    std::cout << r.area() << std::endl;       // 20
    std::cout << square.area() << std::endl;  // 9
}
```

Just like free functions ([Module 4](04-functions-overloading.md)),
constructors can be overloaded — the compiler picks the right one based on
the arguments you pass to it.

## Encapsulation — private members with getters/setters

```cpp
#include <stdexcept>

class BankAccount {
private:
    double balance;   // private -- not accessible outside this class

public:
    BankAccount(double initialBalance) {
        if (initialBalance < 0) {
            throw std::invalid_argument("Initial balance cannot be negative");
        }
        balance = initialBalance;
    }

    double getBalance() const {
        return balance;
    }

    void deposit(double amount) {
        if (amount <= 0) {
            throw std::invalid_argument("Deposit must be positive");
        }
        balance += amount;
    }

    void withdraw(double amount) {
        if (amount > balance) {
            throw std::runtime_error("Insufficient funds");
        }
        balance -= amount;
    }
};

int main() {
    BankAccount account(100.0);
    account.deposit(50.0);
    account.withdraw(30.0);
    std::cout << account.getBalance() << std::endl;   // 120
    // account.balance = -500;   // won't compile -- balance is private
}
```

`public:` and `private:` are **access specifiers** — everything after one
applies until the next specifier (or the end of the class). Keeping data
`private` and exposing only controlled methods is called **encapsulation**;
it's the default, idiomatic way to design classes in C++.

## `struct` vs `class`

```cpp
struct Point3D {   // struct members are public by default
    double x, y, z;
};

class Point3DClass {   // class members are private by default
    double x, y, z;    // private here, unlike the struct above
};
```

`struct` and `class` are otherwise identical in C++ — the only difference is
the default access level. Convention: use `struct` for simple, passive data
bundles with no invariants to protect; use `class` when you have private
state, behavior, and invariants to enforce (like `BankAccount` above).

## Each object owns its own data

```cpp
class Circle {
public:
    double radius;

    Circle(double radius) : radius(radius) {
    }

    double area() const {
        return 3.14159 * radius * radius;
    }
};

int main() {
    Circle c1(2.0);
    Circle c2(5.0);
    std::cout << c1.area() << std::endl;   // 12.5664 -- each object has its own radius
    std::cout << c2.area() << std::endl;   // 78.5398
}
```

| Concept | Meaning |
|---------|---------|
| Class | Blueprint / type definition |
| Object (instance) | A concrete value of a class type |
| Member variable | A variable that lives on each instance |
| Constructor | Special member function that initializes a new instance |
| `this` | Pointer to the current instance |
| `private` / `public` | Access specifiers controlling encapsulation |

## Exercise

Write a `Book` class with private members `title`, `author`, and `pagesRead`
(starting at 0), a constructor taking `title` and `author`, a method
`readPages(int n)` that increases `pagesRead`, and a method
`getProgress(int totalPages) const` that returns the percentage read as a
`double`. Create two `Book` objects in `main`, read some pages on each, and
print their progress.
