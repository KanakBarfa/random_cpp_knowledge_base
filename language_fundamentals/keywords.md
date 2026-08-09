# Language Keywords & Conversion Mechanics

Category: **Language Fundamentals**

---

## 1. Comprehensive Keyword Technical Matrix

The table below summarizes essential C++ keywords across standard revisions, detailing their runtime/compile-time evaluation context, scope impact, and primary technical purpose:

| Keyword | Introduced | Processed By | Primary Technical Purpose |
| :--- | :--- | :--- | :--- |
| `inline` | C++98 / C++17 | Compiler / Linker | Allows definitions across multiple translation units without ODR violations; optimization hint for call-site expansion. |
| `constexpr` | C++11 | Compiler | Guarantees that expressions *can* be evaluated at compile time if inputs are constant expressions. |
| `consteval` | C++20 | Compiler | Defines an **immediate function** that *must* evaluate at compile time; runtime calls produce compile errors. |
| `constinit` | C++20 | Compiler | Guarantees static/thread-local variables are initialized at compile time, preventing Static Initialization Order Fiasco. |
| `explicit` | C++98 / C++20 | Compiler | Prevents implicit constructors or conversion operators; supports conditional `explicit(bool)` since C++20. |
| `mutable` | C++98 | Compiler | Allows modification of class data members within `const` member functions (for logical constness). |
| `noexcept` | C++11 | Compiler / Runtime | Declares exception non-throwing guarantee; enables container move optimizations when true. |
| `override` | C++11 | Compiler | Enforces compiler check that a virtual function overrides a base class virtual member function. |
| `final` | C++11 | Compiler | Prevents class inheritance or virtual function overriding in derived classes. |
| `nullptr` | C++11 | Compiler | Type-safe null pointer literal of type `std::nullptr_t`, eliminating integer zero (`0`/`NULL`) ambiguity. |
| `decltype` | C++11 | Compiler | Inspects the exact declared type or expression category at compile time without evaluating the expression. |
| `static` | C++98 | Compiler / Linker | Context-dependent: local lifetime extension, static class member sharing, or internal linkage. |
| `extern` | C++98 | Linker | Declares a symbol with external linkage whose definition exists in another translation unit. |
| `thread_local` | C++11 | Runtime / Linker | Specifies thread-duration storage; each thread maintains an isolated instance initialized on first access. |
| `delete("msg")` | C++26 | Compiler | Suppresses function generation with a custom compiler diagnostic message explaining the rationale. |
| `_` | C++26 | Compiler | Unnamed wildcard placeholder variable allowing unused declarations without name collision errors. |
| `^` / `[: :]` | C++26 | Compiler | Static Reflection operator (`^`) and Splicing operator (`[: :]`) for compile-time introspection. |

---

## 2. Linkage & Definition Control (`inline`, `static`, `extern`, `thread_local`)

### The `inline` Keyword: Linkage vs Optimization
Historically, `inline` was a compiler hint requesting inlining of function calls to eliminate stack-frame allocation overhead:

```cpp
inline int add(int a, int b) {
    return a + b;
}
```

In modern C++, compilers autonomously decide inlining based on heuristic performance models, ignoring `inline` hints for optimization purposes.

Its primary role in modern C++ is relaxing the **One Definition Rule (ODR)**:
- Functions or variables declared `inline` can be defined in header files included by multiple translation units (`.cpp` files).
- The linker merges identical inline definitions into a single binary symbol.
- **C++17 Inline Variables**: Global constants or static class members can be declared `inline` in header files without separate instantiation `.cpp` files:

```cpp
// Header: my_config.h
#pragma once
#include <string_view>

inline constexpr std::string_view K_APP_NAME = "EngineCore";
inline int g_execution_counter = 0; // Header-only global variable
```

### Static & External Linkage
- **`static` at Namespace Scope**: Restricts symbol visibility to the current translation unit (internal linkage).
- **`static` Inside Functions**: Preserves variable state across function invocations; initialized once thread-safely (since C++11).
- **`extern`**: Announces a symbol defined in another translation unit:

```cpp
// Shared Header
extern int g_global_system_state; // Declaration

// Shared Source File (.cpp)
int g_global_system_state = 42;   // Definition
```

### Thread-Local Storage (`thread_local`)
Specifies that a variable has thread storage duration. Each thread receives its own unique instance:

```cpp
#include <iostream>
#include <thread>

thread_local int t_thread_specific_id = 0;

void worker(int id) {
    t_thread_specific_id = id;
    std::cout << "Thread " << id << " sees ID: " << t_thread_specific_id << "\n";
}
```

---

## 3. Type Safety & Implicit Conversion Control (`explicit`)

### Implicit Conversion Vulnerabilities
By default, any single-argument constructor (or constructor with default values for remaining arguments) acts as an implicit conversion rule for the compiler:

```cpp
class Buffer {
public:
    Buffer(size_t capacity) : capacity_(capacity) {}
private:
    size_t capacity_;
};

void processBuffer(const Buffer& buf) {}

int main() {
    Buffer b = 1024;  // Implicitly converts 1024 (int) into Buffer(1024)
    processBuffer(2048); // Unexpectedly allocates a Buffer(2048) temporary!
}
```

This behavior hides resource allocations and allows unintended operations.

### Preventing Implicit Conversion with `explicit`
Applying `explicit` instructs the compiler to permit construction only during explicit initialization or direct casting:

```cpp
class Buffer {
public:
    explicit Buffer(size_t capacity) : capacity_(capacity) {}
};

int main() {
    // Buffer b = 1024;       // COMPILE ERROR: Conversion required
    Buffer b1(1024);          // OK: Direct initialization
    Buffer b2 = Buffer(1024); // OK: Explicit construction
    // processBuffer(2048);   // COMPILE ERROR: Cannot convert int to Buffer
    processBuffer(Buffer(2048)); // OK: Explicit temporary object
}
```

### C++11, C++20 & C++26 Extensions
1. **Explicit Conversion Operators (C++11)**:
   ```cpp
   class Handle {
   public:
       explicit operator bool() const { return valid_; }
   private:
       bool valid_ = false;
   };
   
   Handle h;
   if (h) {}            // OK: Contextual conversion to bool allowed
   // bool b = h;       // COMPILE ERROR: Implicit assignment prevented
   bool b = static_cast<bool>(h); // OK: Explicit conversion
   ```

2. **Conditional `explicit(bool)` (C++20)**:
   Enables library templates to selectively mark constructors `explicit` based on type traits:
   ```cpp
   template<typename T>
   struct Wrapper {
       template<typename U>
       explicit(!std::is_convertible_v<U, T>) Wrapper(U&& val) {}
   };
   ```

3. **Deleted Functions with Custom Rationale (`= delete("message")`) (C++26)**:
   ```cpp
   class LegacyStream {
   public:
       // C++26 allows specifying diagnostic explanation
       LegacyStream(const LegacyStream&) = delete("LegacyStream copying disabled; use move construction.");
   };
   ```

---

## 4. Exception Guarantees (`noexcept`)

`noexcept` serves two roles: a **specifier** indicating if a function can throw, and an **operator** evaluating compile-time throw possibilities.

```cpp
void safe_function() noexcept {
    // Guaranteed not to throw
}

void conditional_function() noexcept(noexcept(safe_function())) {
    safe_function();
}
```

### Container Move Optimization Criticality
Standard containers like `std::vector` resize by moving elements rather than copying them *only if the move constructor is marked `noexcept`*. 

If a move constructor lacks `noexcept`, `std::vector::resize()` falls back to copying elements to satisfy the **Strong Exception Guarantee**:

```cpp
class HeavyData {
public:
    // Required noexcept for vector move optimizations!
    HeavyData(HeavyData&& other) noexcept : data_(other.data_) {
        other.data_ = nullptr;
    }
private:
    int* data_;
};
```

---

## 5. Compile-Time Expression Inspection (`decltype`)

While `auto` deduces types by stripping top-level `const` and reference qualifications, `decltype` yields the exact declared type of an entity or expression:

```cpp
int x = 10;
const int& ref = x;

auto a = ref;           // Type is 'int' (const and reference stripped)
decltype(ref) b = x;    // Type is 'const int&' (exact type preserved)

// decltype((entity)) evaluates value category:
decltype((x)) lval_ref = x; // Type is 'int&'
```

---

## 6. Modern C++ Type Casting Operators

C-style casts `(TargetType)value` blindly attempt multiple conversion strategies, risking memory corruption and undefined behavior. C++ provides dedicated, type-safe casting operators:

```cpp
class Base { public: virtual ~Base() = default; };
class Derived : public Base { public: void speak() {} };
```

### 1. `static_cast`
Performs compile-time verified conversions between compatible types (e.g., numeric conversions, implicit pointer conversions down/up class hierarchies without virtual checks):

```cpp
double d = 3.14159;
int i = static_cast<int>(d); // Valid numeric cast

Base* b = new Derived();
Derived* d_ptr = static_cast<Derived*>(b); // Up/Downcast
```

### 2. `dynamic_cast`
Performs runtime polymorphic type verification using RTTI. Requires at least one `virtual` function in the base class:

```cpp
Base* b_ptr = new Base();
Derived* d_ptr = dynamic_cast<Derived*>(b_ptr); 

if (d_ptr != nullptr) {
    d_ptr->speak();
} else {
    // Cast failed safely; b_ptr is not a Derived instance
}
```

### 3. `const_cast`
Adds or casts away `const` or `volatile` qualifiers:

```cpp
void legacy_api(char* str) {}

const char* msg = "Hello";
legacy_api(const_cast<char*>(msg)); // Removes const qualifier
```

### 4. `reinterpret_cast`
Reinterprets bit patterns directly without changing binary representation:

```cpp
uintptr_t address = reinterpret_cast<uintptr_t>(b_ptr);
```

### 5. `std::bit_cast` (C++20)
Type-safe, `constexpr`-compatible binary reinterpretation:

```cpp
#include <bit>

float f = 1.0f;
uint32_t u = std::bit_cast<uint32_t>(f); // Reinterprets float bits as uint32_t
```
