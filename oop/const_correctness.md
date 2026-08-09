# Const Correctness, Logical Mutability & Overloading

Category: **Oop**

---

## 1. Const Correctness & Read-Only Safety

Const correctness is a foundational design paradigm in C++ that enforces compile-time immutability contracts. Marking a member function `const` guarantees that invoking the function will not mutate the observable state of the object.

```cpp
class BankAccount {
public:
    explicit BankAccount(double balance) : balance_(balance) {}

    // Const Member Function: Read-only accessor contract
    [[nodiscard]] double get_balance() const {
        // balance_ += 10.0; // COMPILE ERROR: Cannot modify member variables in const function!
        return balance_;
    }

    // Non-Const Member Function: Mutator function
    void deposit(double amount) {
        balance_ += amount;
    }

private:
    double balance_;
};
```

### Compiler Pointer Decoration & Safety Enforcement
Inside a `const` member function of class `T`, the implicit `this` pointer is decorated as:
$$\text{\tt const T* const this}$$

This ensures:
1. `const` references (`const BankAccount&`) can **only** call `const` member functions.
2. Accidental state modification generates an immediate compile-time error.

```cpp
void audit_account(const BankAccount& account) {
    std::cout << "Balance: " << account.get_balance() << "\n"; // OK: Const function call
    // account.deposit(100.0); // COMPILE ERROR: Cannot call non-const function on const reference!
}
```

---

## 2. Physical vs Logical Constness & the `mutable` Keyword

C++ distinguishes between two definitions of constness:

- **Physical (Bitwise) Constness**: Every single byte/bit of object memory remains completely unchanged.
- **Logical Constness**: The object's user-facing state appears unchanged, but internal non-observable state (e.g., caches, mutexes) can be updated.

### The `mutable` Specifier
The `mutable` keyword allows specific class data members to be modified even inside `const` member functions.

```cpp
#include <iostream>
#include <mutex>
#include <string>

class ThreadSafeCache {
public:
    std::string fetch_data() const {
        // 1. Thread Synchronization via mutable mutex
        std::lock_guard<std::mutex> lock(mutex_); 

        // 2. Performance Metrics via mutable counter
        ++access_count_; 

        // 3. Lazy Cache Initialization via mutable cache variables
        if (!cache_valid_) {
            cached_result_ = "Computed Heavy Data";
            cache_valid_ = true;
        }

        return cached_result_;
    }

    [[nodiscard]] size_t get_access_count() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return access_count_;
    }

private:
    mutable std::mutex mutex_;            // Mutex must be lockable in const methods
    mutable size_t access_count_ = 0;     // Metric counter
    mutable std::string cached_result_;   // Cache buffer
    mutable bool cache_valid_ = false;    // Cache status
};
```

---

## 3. Overloading Member Functions on Constness

Classes providing subscript or element accessor operations often double-implement methods: a `const` overload returning a `const` reference for read-only access, and a non-const overload returning a mutable reference for writing:

```cpp
#include <iostream>
#include <vector>

class CustomContainer {
public:
    CustomContainer(size_t size) : data_(size, 0) {}

    // 1. Non-Const Overload: Allows modifying container elements
    int& operator[](size_t index) {
        return data_[index];
    }

    // 2. Const Overload: Read-only access for const object references
    const int& operator[](size_t index) const {
        return data_[index];
    }

private:
    std::vector<int> data_;
};

int main() {
    CustomContainer mutable_c(5);
    mutable_c[0] = 42; // Calls non-const operator[]

    const CustomContainer const_c(5);
    int val = const_c[0]; // Calls const operator[]
    // const_c[0] = 99;   // COMPILE ERROR: Cannot assign through const reference!
}
```

### `std::as_const` Utility (C++17)
The `<utility>` header provides `std::as_const(x)`, which converts a reference `T&` into `const T&`, forcing the invocation of `const` overloaded functions:

```cpp
#include <utility>

CustomContainer container(5);
// Force invocation of const operator[] on mutable container
const int& read_only_ref = std::as_const(container)[0];
```
