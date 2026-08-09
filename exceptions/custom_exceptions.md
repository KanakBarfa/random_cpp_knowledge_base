# Custom Exceptions, Hierarchy & Exception Safety

Category: **Error & Exception Handling**

---

## 1. ISO Standard Exception Hierarchy

C++ provides a standard exception hierarchy defined across `<exception>`, `<stdexcept>`, and `<system_error>`.

```
                        std::exception
                             /   \
                            /     \
               std::logic_error  std::runtime_error
                   /     \           /     \
                  /       \         /       \
  std::invalid_argument  std::out_of_range  std::overflow_error
```

### Standard Exception Categories

- **`std::logic_error`**: Represents errors preventable through proper program logic and parameter validation before call execution (e.g., `std::out_of_range`, `std::invalid_argument`).
- **`std::runtime_error`**: Represents operational errors detected only during program execution (e.g., `std::overflow_error`, I/O failures, resource allocation issues).

---

## 2. Designing Custom Exception Classes

When building custom exception types, inherit from `std::runtime_error` or `std::exception` and override `what() const noexcept`:

```cpp
#include <exception>
#include <string>
#include <iostream>

// Production Custom Exception Class
class DatabaseException : public std::exception {
public:
    DatabaseException(int error_code, std::string message)
        : error_code_(error_code), 
          full_message_("DB Error [" + std::to_string(error_code) + "]: " + message) {}

    // Override what() with non-throwing guarantee
    [[nodiscard]] const char* what() const noexcept override {
        return full_message_.c_str();
    }

    [[nodiscard]] int get_error_code() const noexcept {
        return error_code_;
    }

private:
    int error_code_;
    std::string full_message_;
};

void connect_to_database() {
    throw DatabaseException(1004, "Connection timed out to host 10.0.0.1");
}

int main() {
    try {
        connect_to_database();
    } catch (const DatabaseException& ex) {
        std::cerr << ex.what() << "\n";
        std::cerr << "Diagnostic Code: " << ex.get_error_code() << "\n";
    } catch (const std::exception& ex) {
        std::cerr << "Generic STL Error: " << ex.what() << "\n";
    }
}
```

---

## 3. Exception Safety Guarantees

Functions in robust C++ codebases provide one of four strict exception safety guarantees:

```
[Nothrow Guarantee] > [Strong Guarantee] > [Basic Guarantee] > [No Guarantee (UB)]
```

### 1. Nothrow (No-Fail) Guarantee
The function is guaranteed **never** to throw an exception under any circumstance. Specified via `noexcept`:
```cpp
void cleanup() noexcept;
```

### 2. Strong Exception Guarantee (Commit-or-Rollback)
If the function fails and throws an exception, the application state rolls back completely to its state prior to function invocation. No data is lost or corrupted:

```cpp
#include <vector>
#include <iostream>

class TransactionalVector {
public:
    void append_batch(const std::vector<int>& items) {
        std::vector<int> temp = data_; // 1. Copy current state
        for (int item : items) {
            temp.push_back(item);     // 2. Perform operations on copy
        }
        data_ = std::move(temp);      // 3. Commit state via non-throwing move!
    }
private:
    std::vector<int> data_;
};
```

### 3. Basic Exception Guarantee
If an exception throws, no resources or memory are leaked, and all objects remain in valid (though possibly altered) states.

### 4. No Guarantee
An exception leaves objects in corrupt states or causes resource leaks. This is unacceptable in enterprise software.

---

## 4. Exception Performance Mechanics (`noexcept`)

Modern C++ compilers use **Zero-Cost Exception Handling** (Itanium ABI unwind tables). On happy-path execution where no exception is thrown, `try`-blocks incur **zero CPU instruction overhead**. 

### `noexcept` Termination Behavior
If a function marked `noexcept` attempts to emit an unhandled exception, stack unwinding is immediately aborted and `std::terminate()` is invoked:

```cpp
void strict_no_throw() noexcept {
    throw std::runtime_error("Fatal Breach"); // Immediately calls std::terminate()!
}
```
