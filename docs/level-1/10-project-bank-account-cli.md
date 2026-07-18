# 10 · Project — Bank Account CLI

A small end-to-end project combining everything from Level 1: classes, arrays
(`std::vector`), strings, references, and exception handling.

## What you'll build

A command-line banking system that:

- Creates accounts with a name and starting balance
- Deposits and withdraws money (with validation)
- Lists all accounts
- Throws and catches exceptions for invalid operations (e.g. overdrawing)

## Project layout

```text
bank/
    account.h
    account.cpp
    main.cpp
```

## account.h — class declaration

```cpp
// account.h
#ifndef ACCOUNT_H
#define ACCOUNT_H

#include <string>

class Account {
public:
    Account(const std::string& owner, double initialBalance);

    void deposit(double amount);
    void withdraw(double amount);   // throws std::runtime_error on insufficient funds

    const std::string& getOwner() const;
    double getBalance() const;

private:
    std::string owner;
    double balance;
};

#endif
```

## account.cpp — implementation

```cpp
// account.cpp
#include "account.h"
#include <stdexcept>

Account::Account(const std::string& owner, double initialBalance)
    : owner(owner), balance(initialBalance) {
    if (initialBalance < 0) {
        throw std::invalid_argument("Initial balance cannot be negative");
    }
}

void Account::deposit(double amount) {
    if (amount <= 0) {
        throw std::invalid_argument("Deposit amount must be positive");
    }
    balance += amount;
}

void Account::withdraw(double amount) {
    if (amount <= 0) {
        throw std::invalid_argument("Withdrawal amount must be positive");
    }
    if (amount > balance) {
        throw std::runtime_error("Insufficient funds");
    }
    balance -= amount;
}

const std::string& Account::getOwner() const {
    return owner;
}

double Account::getBalance() const {
    return balance;
}
```

## main.cpp — CLI logic

```cpp
// main.cpp
#include <iostream>
#include <vector>
#include "account.h"

int findAccount(const std::vector<Account>& accounts, const std::string& owner) {
    for (size_t i = 0; i < accounts.size(); i++) {
        if (accounts[i].getOwner() == owner) {
            return static_cast<int>(i);
        }
    }
    return -1;
}

int main() {
    std::vector<Account> accounts;
    int choice = 0;

    do {
        std::cout << "\n1. Open account  2. Deposit  3. Withdraw  4. List  5. Quit\n> ";
        std::cin >> choice;

        try {
            if (choice == 1) {
                std::string owner;
                double balance;
                std::cout << "Owner name: ";
                std::cin >> owner;
                std::cout << "Initial balance: ";
                std::cin >> balance;
                accounts.emplace_back(owner, balance);
                std::cout << "Account opened for " << owner << std::endl;

            } else if (choice == 2 || choice == 3) {
                std::string owner;
                double amount;
                std::cout << "Owner name: ";
                std::cin >> owner;
                std::cout << "Amount: ";
                std::cin >> amount;

                int idx = findAccount(accounts, owner);
                if (idx < 0) {
                    std::cout << "No such account." << std::endl;
                    continue;
                }

                if (choice == 2) {
                    accounts[idx].deposit(amount);
                } else {
                    accounts[idx].withdraw(amount);
                }
                std::cout << "New balance: " << accounts[idx].getBalance() << std::endl;

            } else if (choice == 4) {
                for (const auto& acc : accounts) {
                    std::cout << acc.getOwner() << ": $" << acc.getBalance() << std::endl;
                }
            }
        } catch (const std::exception& e) {
            std::cout << "Error: " << e.what() << std::endl;
        }

    } while (choice != 5);

    std::cout << "Goodbye!" << std::endl;
    return 0;
}
```

## Compiling and running

```bash
g++ -std=c++17 -Wall -o bank main.cpp account.cpp
./bank
```

```text
1. Open account  2. Deposit  3. Withdraw  4. List  5. Quit
> 1
Owner name: Ada
Initial balance: 100
Account opened for Ada

1. Open account  2. Deposit  3. Withdraw  4. List  5. Quit
> 3
Owner name: Ada
Amount: 500
Error: Insufficient funds
```

## Stretch goals

- Add a `transfer` operation between two accounts (deposit + withdraw as one
  logical unit — think about what should happen if the withdraw fails).
- Add a transaction log (a `std::vector<std::string>` of history entries) per
  account.
- Replace the raw `std::vector<Account>` search with a `std::map<std::string,
  Account>` keyed by owner name for O(log n) lookups.

Completing this project means you're ready for **Level 2 · Intermediate**.
