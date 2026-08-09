# C++26 Standards & Language Enhancements

Category: **Standards**

---

## 1. Executive Summary

C++26 (ISO/IEC 14882:2026) is a major milestone release for modern system software engineering. It introduces native **Static Reflection** (`std::meta`), variadic **Pack Indexing**, standardized **Linear Algebra** (`<linalg>`), **Lock-Free Memory Reclamation** (`<hazard_pointer>` and `<rcu>`), native **Debugging Facilities** (`<debugging>`), deleted functions with diagnostic messages (`= delete("reason")`), and unnamed wildcard placeholder variables (`_`).

---

## 2. Static Reflection & Metaprogramming (`std::meta`)

Static reflection allows inspecting and code-generating class members, types, enums, and function signatures at compile time using standard C++ syntax, eliminating external code generators and macro hacks.

### Key Syntax Operators

1. **Reflection Operator (`^`)**: Generates a compile-time reflection object of type `std::meta::info`.
2. **Splice Operator (`[: :]`)**: Injects a reflection handle back into standard C++ source code as a type, expression, or entity.

```cpp
#include <meta>
#include <iostream>
#include <string_view>

struct UserAccount {
    int id;
    std::string name;
    double balance;
};

int main() {
    // 1. Reflect type into a meta info handle
    constexpr std::meta::info meta_type = ^UserAccount;

    // 2. Query type name at compile time
    constexpr std::string_view type_name = std::meta::name_of(meta_type);
    std::cout << "Reflected Type Name: " << type_name << "\n";

    // 3. Iterate over class members at compile time
    constexpr auto members = std::meta::members_of(meta_type);
    
    // 4. Splice type back into source code
    using ReflectedType = [: meta_type :];
    ReflectedType user{101, "Alice", 250.50};
    std::cout << "User ID via Spliced Type: " << user.id << "\n";
}
```

---

## 3. Variadic Pack Indexing (`T...[I]` & `args...[I]`)

Prior to C++26, accessing the $N$-th element of a variadic template parameter pack required recursive template inheritance or `std::get<I>(std::forward_as_tuple(args...))`. C++26 provides direct indexed access:

```cpp
#include <iostream>
#include <string>

// Direct Pack Indexing in C++26
template<typename... Types>
auto get_first_element(Types... args) {
    // Access argument at index 0 directly from pack
    return args...[0];
}

template<typename... Types>
using FirstType = Types...[0]; // Direct type pack indexing

int main() {
    FirstType<int, double, std::string> val = 42; // FirstType evaluates to int
    std::cout << "First arg: " << get_first_element(100, 3.14, "text") << "\n"; // Outputs 100
}
```

---

## 4. Standardized Linear Algebra Library (`<linalg>`)

C++26 provides a high-performance linear algebra interface built on top of standard Basic Linear Algebra Subprograms (BLAS) algorithms operating over `std::mdspan` multi-dimensional arrays:

```cpp
#include <mdspan>
#include <linalg>
#include <vector>
#include <iostream>

int main() {
    std::vector<double> A_data(9, 1.0);
    std::vector<double> x_data = {1.0, 2.0, 3.0};
    std::vector<double> y_data(3, 0.0);

    // Create 3x3 matrix view and vector views
    std::mdspan A(A_data.data(), 3, 3);
    std::mdspan x(x_data.data(), 3);
    std::mdspan y(y_data.data(), 3);

    // Perform Matrix-Vector Multiplication y = A * x using standard linalg
    std::linalg::matrix_vector_product(A, x, y);

    std::cout << "y[0] result: " << y[0] << "\n"; // Output: 6.0
}
```

---

## 5. Lock-Free Memory Reclamation (`<hazard_pointer>` & `<rcu>`)

C++26 adds concurrent memory management primitives essential for non-blocking lock-free data structures:

1. **Hazard Pointers (`<hazard_pointer>`)**: Protects memory nodes from being freed by concurrent threads while a reader thread is inspecting them.
2. **Read-Copy-Update (`<rcu>`)**: Provides lock-free read access with deferred memory cleanup during quiet periods (`std::rcu_synchronize()`).

```cpp
#include <hazard_pointer>
#include <atomic>

struct Node {
    int value;
    std::atomic<Node*> next;
};

void safe_reader_thread(std::atomic<Node*>& head) {
    // Acquire hazard pointer protection for lock-free read
    std::hazard_pointer hp = std::make_hazard_pointer();
    Node* ptr = hp.protect(head);
    
    if (ptr) {
        // Safely access node without fear of concurrent deallocation
        int val = ptr->value;
    }
}
```

---

## 6. Standard Debugging Facilities (`<debugging>`)

Provides cross-platform, standardized routines for interacting with native debuggers:

```cpp
#include <debugging>
#include <iostream>

void critical_routine() {
    if (std::is_debugger_present()) {
        std::cout << "Debugger detected; breaking...\n";
        std::breakpoint(); // Triggers hardware breakpoint
    }
}
```

---

## 7. Syntactic & Quality-of-Life Enhancements

### 1. Delete Functions with Reason (`= delete("message")`)
Allows library developers to specify custom compiler error messages when a deleted function is invoked:

```cpp
class ThreadManager {
public:
    // Custom error message displayed in compiler output if user attempts copy!
    ThreadManager(const ThreadManager&) = delete("Copying ThreadManager is disabled due to thread affinity; transfer ownership using std::move.");
    ThreadManager(ThreadManager&&) noexcept = default;
};
```

### 2. Unnamed Wildcard Placeholder Variables (`_`)
Unused variables can be declared as `_` without causing symbol collision compile errors:

```cpp
#include <mutex>
#include <tuple>

std::mutex mtx;

void process_data() {
    // Lock guard assigned to wildcard placeholder '_'
    std::lock_guard<std::mutex> _{mtx};

    // Ignore middle element using wildcard placeholder '_'
    auto [id, _, score] = std::make_tuple(101, "Ignored String", 99.5);
}
```
