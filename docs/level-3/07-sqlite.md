# 07 · Working with SQLite from C++

SQLite is a full SQL database engine that lives in a single library and
reads/writes a single file — no server process, no network connection,
just a C API you link against directly. It ships preinstalled on macOS and
most Linux distributions, which makes it the natural first database to learn
from C++: everything in this module runs against `sqlite3.h` and
`-lsqlite3`, no separate server to start. The API is C, not C++, which means
manual resource management — a perfect place to apply the RAII lessons from
[Module 4](04-raii-deep-dive.md).

## Opening a database and running a statement

```cpp
#include <iostream>
#include <sqlite3.h>

int main() {
    sqlite3* db = nullptr;
    int rc = sqlite3_open("example.db", &db);   // creates the file if it doesn't exist
    if (rc != SQLITE_OK) {
        std::cerr << "cannot open database: " << sqlite3_errmsg(db) << std::endl;
        return 1;
    }
    std::cout << "database opened" << std::endl;

    const char* createTable =
        "CREATE TABLE IF NOT EXISTS employees ("
        "  id INTEGER PRIMARY KEY,"
        "  name TEXT NOT NULL,"
        "  department TEXT"
        ");";

    char* errMsg = nullptr;
    rc = sqlite3_exec(db, createTable, nullptr, nullptr, &errMsg);
    if (rc != SQLITE_OK) {
        std::cerr << "exec failed: " << errMsg << std::endl;
        sqlite3_free(errMsg);
    } else {
        std::cout << "table ready" << std::endl;
    }

    sqlite3_close(db);
}
// database opened
// table ready
```

`sqlite3_exec` is the blunt tool: it runs a whole SQL string and (optionally)
calls a callback per result row. It's fine for statements with no
result-set-you-care-about, like `CREATE TABLE`. For anything involving
untrusted or variable data, it's the wrong tool — see the next section.

## Prepared statements — the safe way to pass values

Building SQL by string-concatenating values in is both a correctness hazard
(quoting names, dates, and special characters correctly by hand is
error-prone) and a **SQL injection** vulnerability the moment any of that
data comes from outside the program. Prepared statements compile the SQL
once with `?` placeholders, then bind values in separately:

```cpp
#include <iostream>
#include <sqlite3.h>

void insertEmployee(sqlite3* db, const std::string& name, const std::string& department) {
    const char* sql = "INSERT INTO employees (name, department) VALUES (?, ?);";
    sqlite3_stmt* stmt = nullptr;

    if (sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr) != SQLITE_OK) {
        std::cerr << "prepare failed: " << sqlite3_errmsg(db) << std::endl;
        return;
    }

    // SQLITE_TRANSIENT tells SQLite to copy the string now, since it may
    // go out of scope before the statement executes.
    sqlite3_bind_text(stmt, 1, name.c_str(), -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 2, department.c_str(), -1, SQLITE_TRANSIENT);

    if (sqlite3_step(stmt) != SQLITE_DONE) {
        std::cerr << "insert failed: " << sqlite3_errmsg(db) << std::endl;
    } else {
        std::cout << "inserted " << name << std::endl;
    }

    sqlite3_finalize(stmt);   // always release the compiled statement
}

int main() {
    sqlite3* db;
    sqlite3_open("example.db", &db);
    sqlite3_exec(db, "CREATE TABLE IF NOT EXISTS employees (id INTEGER PRIMARY KEY, name TEXT, department TEXT);", nullptr, nullptr, nullptr);
    sqlite3_exec(db, "DELETE FROM employees;", nullptr, nullptr, nullptr);   // start clean for the demo

    insertEmployee(db, "Grace Hopper", "Engineering");
    insertEmployee(db, "Ada Lovelace", "Research");

    sqlite3_close(db);
}
// inserted Grace Hopper
// inserted Ada Lovelace
```

Even names containing a `'` (like `O'Brien`) are handled correctly through a
bound parameter — the driver, not string concatenation, is responsible for
escaping. This is worth internalizing as a rule, not just a suggestion:
**never** build a SQL string by concatenating in a value that isn't a
hardcoded constant.

## Querying rows back out

`sqlite3_step` doubles as both "run this" (for statements with no rows) and
"give me the next row" (for a `SELECT`) — it returns `SQLITE_ROW` once per
row, then `SQLITE_DONE` when there are no more:

```cpp
#include <iostream>
#include <sqlite3.h>

void printAllEmployees(sqlite3* db) {
    const char* sql = "SELECT id, name, department FROM employees ORDER BY id;";
    sqlite3_stmt* stmt = nullptr;
    sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr);

    while (sqlite3_step(stmt) == SQLITE_ROW) {
        int id = sqlite3_column_int(stmt, 0);
        const unsigned char* name = sqlite3_column_text(stmt, 1);
        const unsigned char* dept = sqlite3_column_text(stmt, 2);
        std::cout << id << ": " << name << " (" << dept << ")" << std::endl;
    }

    sqlite3_finalize(stmt);
}

int main() {
    sqlite3* db;
    sqlite3_open("example.db", &db);
    printAllEmployees(db);
    sqlite3_close(db);
}
// 1: Grace Hopper (Engineering)
// 2: Ada Lovelace (Research)
```

Column values are pulled out with a `sqlite3_column_*` function matching the
type you expect (`_int`, `_double`, `_text`, `_blob`) and a zero-based column
index — there's no compile-time type checking here, since SQLite doesn't
know your C++ types; getting the index or type wrong is a runtime bug, not a
compiler error.

## An RAII wrapper for the connection

Wrapping `sqlite3*` the same way [Module 4](04-raii-deep-dive.md) wrapped
`FILE*` removes an entire class of "forgot to close it" and "closed it
twice" bugs:

```cpp
#include <iostream>
#include <sqlite3.h>
#include <stdexcept>
#include <string>

class Database {
public:
    explicit Database(const std::string& path) {
        if (sqlite3_open(path.c_str(), &db_) != SQLITE_OK) {
            std::string msg = sqlite3_errmsg(db_);
            sqlite3_close(db_);
            throw std::runtime_error("cannot open database: " + msg);
        }
    }

    ~Database() { sqlite3_close(db_); }

    Database(const Database&) = delete;
    Database& operator=(const Database&) = delete;

    sqlite3* handle() const { return db_; }

private:
    sqlite3* db_ = nullptr;
};

int main() {
    Database db("example.db");
    std::cout << "connection open, will close automatically" << std::endl;
}   // db's destructor calls sqlite3_close() here -- even if an exception had been thrown above
// connection open, will close automatically
```

## Cheat sheet

| Function | Purpose |
|----------|---------|
| `sqlite3_open` / `sqlite3_close` | Open/close a database file (creates it if missing) |
| `sqlite3_exec` | Run a SQL string with no bound parameters; optional row callback |
| `sqlite3_prepare_v2` | Compile SQL with `?` placeholders into a reusable statement |
| `sqlite3_bind_*` | Bind a value to a `?` placeholder by 1-based index |
| `sqlite3_step` | Execute, or advance to the next result row (`SQLITE_ROW` / `SQLITE_DONE`) |
| `sqlite3_column_*` | Read a column's value from the current row by 0-based index |
| `sqlite3_finalize` | Release a prepared statement (always required) |
| `sqlite3_errmsg` | Human-readable text for the last error on a connection |

## Traps

**String-concatenating values into SQL is a SQL injection bug**, not just a
style nit — a name containing `'; DROP TABLE employees; --` will do exactly
what it says if it lands in the SQL text directly instead of through a bound
parameter.

**Every `sqlite3_prepare_v2` needs a matching `sqlite3_finalize`**, on every
return path, including early returns on error — otherwise the statement (and
any lock it implies) is never released. This is the same resource-per-scope
problem as `fopen`/`fclose`, and the same fix applies: wrap it in an RAII
type instead of finalizing by hand at every exit point.

**`sqlite3_column_text` returns a pointer valid only until the next
`sqlite3_step` or `sqlite3_finalize` call on that statement.** Copy it into a
`std::string` immediately if it needs to outlive the current row.

## Exercise

Wrap a prepared statement in an RAII class `Statement` (constructor calls
`sqlite3_prepare_v2`, destructor calls `sqlite3_finalize`, move-only like the
`FileHandle` from Module 4) so `insertEmployee` and `printAllEmployees` no
longer need a manual `sqlite3_finalize` call. Then add a
`findByDepartment(sqlite3* db, const std::string& department)` function that
binds one parameter, collects matching rows into a
`std::vector<std::pair<int, std::string>>` of `(id, name)`, and returns it —
confirm it returns exactly the rows you expect after inserting a few more
employees across different departments.
