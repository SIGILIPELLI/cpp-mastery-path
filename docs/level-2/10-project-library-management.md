# 10 · Project — Library Management System

This project pulls together everything from Level 2: OOP with inheritance,
STL containers, smart pointers, operator overloading, and file I/O with
streams. You'll build a small library system where books are polymorphic
(a `Book` base with an `EBook` subclass), owned by `std::unique_ptr` so
there's never a manual `delete`, and looked up through an `std::unordered_map`
keyed by ISBN.

## What you'll build

A command-line library system that:

- Adds physical books and e-books, both derived from a common `Book` base
- Looks up a book by ISBN in O(1) via a hash map
- Checks books out and back in, tracking whether a copy is available
- Lists the catalog, printing each book polymorphically (an e-book prints its
  file size; a physical book prints its shelf location)
- Saves and loads the catalog to a text file

## Project layout

```text
library/
    include/
        book.h
        library.h
    src/
        library.cpp
        main.cpp
```

`Book`, `EBook` and `PhysicalBook` are small enough to stay header-only —
every member function is defined inline in `book.h`, so there's no matching
`book.cpp`.

## include/book.h — the polymorphic hierarchy

```cpp
// book.h
#ifndef BOOK_H
#define BOOK_H

#include <string>
#include <iostream>

class Book {
public:
    Book(std::string isbn, std::string title, std::string author)
        : isbn_(std::move(isbn)), title_(std::move(title)), author_(std::move(author)),
          checkedOut_(false) {}

    // A base class destroyed through a base pointer needs a virtual
    // destructor -- without it, deleting an EBook* stored as Book*
    // (which is exactly what our unique_ptr<Book> does) would only run
    // ~Book() and skip ~EBook(), leaking anything EBook owns.
    virtual ~Book() = default;

    const std::string& isbn() const { return isbn_; }
    const std::string& title() const { return title_; }
    const std::string& author() const { return author_; }
    bool isCheckedOut() const { return checkedOut_; }

    void checkOut() { checkedOut_ = true; }
    void checkIn() { checkedOut_ = false; }

    // Every concrete Book knows how to describe itself; the catalog
    // never needs an if/else chain on book type to print correctly.
    virtual void describe(std::ostream& os) const {
        os << "[" << isbn_ << "] " << title_ << " by " << author_
           << (checkedOut_ ? " (checked out)" : " (available)");
    }

    // operator<< can't be virtual itself, so it forwards to describe(),
    // which can be. This is the standard pattern for polymorphic printing.
    friend std::ostream& operator<<(std::ostream& os, const Book& book) {
        book.describe(os);
        return os;
    }

private:
    std::string isbn_;
    std::string title_;
    std::string author_;
    bool checkedOut_;
};

class EBook : public Book {
public:
    EBook(std::string isbn, std::string title, std::string author, double fileSizeMb)
        : Book(std::move(isbn), std::move(title), std::move(author)),
          fileSizeMb_(fileSizeMb) {}

    void describe(std::ostream& os) const override {
        Book::describe(os);   // reuse the base description, then extend it
        os << " -- " << fileSizeMb_ << " MB ebook";
    }

private:
    double fileSizeMb_;
};

class PhysicalBook : public Book {
public:
    PhysicalBook(std::string isbn, std::string title, std::string author, std::string shelf)
        : Book(std::move(isbn), std::move(title), std::move(author)),
          shelf_(std::move(shelf)) {}

    void describe(std::ostream& os) const override {
        Book::describe(os);
        os << " -- shelf " << shelf_;
    }

private:
    std::string shelf_;
};

#endif
```

## include/library.h — the catalog

```cpp
// library.h
#ifndef LIBRARY_H
#define LIBRARY_H

#include <memory>
#include <string>
#include <unordered_map>
#include "book.h"

class Library {
public:
    // Takes ownership of the book. Returns false (and the unique_ptr keeps
    // the book, since it's passed by value and never moved-from) if the
    // ISBN is already in the catalog.
    bool addBook(std::unique_ptr<Book> book);

    // Returns nullptr if not found. The Library still owns the book --
    // callers get a view, never a transfer of ownership.
    Book* find(const std::string& isbn);

    bool checkOut(const std::string& isbn);
    bool checkIn(const std::string& isbn);

    void printCatalog(std::ostream& os) const;

    bool saveToFile(const std::string& path) const;

private:
    // unique_ptr<Book> in the map means the map owns every book, and
    // erasing an entry (or destroying the map) frees it automatically --
    // no destructor loop, no leak, no double free.
    std::unordered_map<std::string, std::unique_ptr<Book>> books_;
};

#endif
```

## src/library.cpp — implementation

```cpp
// library.cpp
#include <fstream>
#include "library.h"

bool Library::addBook(std::unique_ptr<Book> book) {
    const std::string& isbn = book->isbn();
    if (books_.count(isbn) > 0) {
        return false;   // duplicate ISBN -- 'book' still owns it, safely destroyed on return
    }
    books_[isbn] = std::move(book);   // transfer ownership into the map
    return true;
}

Book* Library::find(const std::string& isbn) {
    auto it = books_.find(isbn);
    return it == books_.end() ? nullptr : it->second.get();
}

bool Library::checkOut(const std::string& isbn) {
    Book* book = find(isbn);
    if (book == nullptr || book->isCheckedOut()) {
        return false;
    }
    book->checkOut();
    return true;
}

bool Library::checkIn(const std::string& isbn) {
    Book* book = find(isbn);
    if (book == nullptr || !book->isCheckedOut()) {
        return false;
    }
    book->checkIn();
    return true;
}

void Library::printCatalog(std::ostream& os) const {
    for (const auto& [isbn, book] : books_) {
        os << *book << "\n";   // operator<< dispatches to the right describe()
    }
}

bool Library::saveToFile(const std::string& path) const {
    std::ofstream out(path);
    if (!out) {
        return false;
    }
    for (const auto& [isbn, book] : books_) {
        out << *book << "\n";
    }
    return true;   // std::ofstream's destructor flushes and closes
}
```

## src/main.cpp — CLI

```cpp
// main.cpp
#include <iostream>
#include <memory>
#include <limits>
#include "library.h"

static void printMenu() {
    std::cout << "\n1. Add physical book  2. Add e-book  3. List catalog"
                 "\n4. Check out  5. Check in  6. Save & quit\n> ";
}

int main() {
    Library library;

    library.addBook(std::make_unique<PhysicalBook>(
        "978-0-13-468599-1", "Effective Modern C++", "Scott Meyers", "Shelf B3"));
    library.addBook(std::make_unique<EBook>(
        "978-1-4919-0399-5", "Programming Rust", "Jim Blandy", 8.4));

    int choice = 0;
    std::string isbn, title, author, extra;

    do {
        printMenu();
        std::cin >> choice;
        std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');

        switch (choice) {
            case 1: {
                std::cout << "ISBN: ";     std::getline(std::cin, isbn);
                std::cout << "Title: ";    std::getline(std::cin, title);
                std::cout << "Author: ";   std::getline(std::cin, author);
                std::cout << "Shelf: ";    std::getline(std::cin, extra);
                bool added = library.addBook(
                    std::make_unique<PhysicalBook>(isbn, title, author, extra));
                std::cout << (added ? "Added.\n" : "ISBN already exists.\n");
                break;
            }
            case 2: {
                std::cout << "ISBN: ";           std::getline(std::cin, isbn);
                std::cout << "Title: ";          std::getline(std::cin, title);
                std::cout << "Author: ";         std::getline(std::cin, author);
                std::cout << "File size (MB): "; std::getline(std::cin, extra);
                bool added = library.addBook(
                    std::make_unique<EBook>(isbn, title, author, std::stod(extra)));
                std::cout << (added ? "Added.\n" : "ISBN already exists.\n");
                break;
            }
            case 3:
                library.printCatalog(std::cout);
                break;
            case 4:
                std::cout << "ISBN: ";
                std::getline(std::cin, isbn);
                std::cout << (library.checkOut(isbn)
                                  ? "Checked out.\n"
                                  : "Not found or already checked out.\n");
                break;
            case 5:
                std::cout << "ISBN: ";
                std::getline(std::cin, isbn);
                std::cout << (library.checkIn(isbn)
                                  ? "Checked in.\n"
                                  : "Not found or not checked out.\n");
                break;
            case 6:
                library.saveToFile("catalog.txt");
                std::cout << "Saved. Goodbye!\n";
                break;
            default:
                std::cout << "Unknown option.\n";
        }
    } while (choice != 6);

    return 0;
    // library goes out of scope here -- every unique_ptr<Book> in its
    // unordered_map runs its destructor, and every EBook/PhysicalBook is
    // freed correctly through the virtual destructor. No delete anywhere
    // in this file.
}
```

## Compiling and running

```bash
g++ -std=c++17 -Wall -Wextra -Iinclude \
    -o library src/library.cpp src/main.cpp
./library
```

```text
1. Add physical book  2. Add e-book  3. List catalog
4. Check out  5. Check in  6. Save & quit
> 3
[978-1-4919-0399-5] Programming Rust by Jim Blandy (available) -- 8.4 MB ebook
[978-0-13-468599-1] Effective Modern C++ by Scott Meyers (available) -- shelf B3

1. Add physical book  2. Add e-book  3. List catalog
4. Check out  5. Check in  6. Save & quit
> 4
ISBN: 978-0-13-468599-1
Checked out.

1. Add physical book  2. Add e-book  3. List catalog
4. Check out  5. Check in  6. Save & quit
> 3
[978-1-4919-0399-5] Programming Rust by Jim Blandy (available) -- 8.4 MB ebook
[978-0-13-468599-1] Effective Modern C++ by Scott Meyers (checked out) -- shelf B3

1. Add physical book  2. Add e-book  3. List catalog
4. Check out  5. Check in  6. Save & quit
> 6
Saved. Goodbye!
```

(`std::unordered_map` iteration order is unspecified, so your own run may list
the two books in the opposite order — that's expected, not a bug.)

## Why `unique_ptr<Book>` and not `Book` by value

`std::unordered_map<std::string, Book>` would slice every `EBook` and
`PhysicalBook` down to a plain `Book` the moment it's inserted — the derived
fields and overridden `describe()` would be gone. Storing
`std::unique_ptr<Book>` instead keeps the *dynamic type* intact: the map holds
a pointer to whatever was actually constructed, `describe()` still dispatches
virtually, and the destructor still runs the derived class's cleanup. This is
the standard way to put polymorphic objects in an STL container.

## Stretch goals

- Add a `removeBook(isbn)` that erases the entry from the `unordered_map` —
  confirm with a debug build that the corresponding `Book` destructor actually
  runs (add a `std::cout` line to `~Book()` temporarily to see it).
- Add a third `Book` subclass, e.g. `AudioBook` with a `durationMinutes`
  field, and confirm `printCatalog` handles it with zero changes to
  `Library` — that's the payoff of the virtual `describe()` design.
- Replace the linear "does ISBN exist" check nowhere in this code (it's
  already O(1) via `unordered_map::count`) — instead, add a *second* index,
  `std::multimap<std::string, std::string> byAuthor`, mapping author name to
  ISBN, and support listing all books by a given author.
- Load `catalog.txt` back into a fresh `Library` on startup, so the catalog
  persists across runs the way [Module 7](07-file-io-streams.md) covers.

Completing this project means you're ready for **Level 3 · Advanced**.
