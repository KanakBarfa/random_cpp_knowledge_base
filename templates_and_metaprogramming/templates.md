# Templates, Instantiation & Metaprogramming Foundations

Category: **Templates And Metaprogramming**

---

## 1. Template Taxonomy & Syntax

C++ templates provide compile-time generic abstractions, allowing functions, classes, variables, and type aliases to operate across arbitrary data types without runtime overhead.

```cpp
#include <iostream>
#include <string>
#include <unordered_map>

// 1. Function Template
template<typename T>
T max_val(T a, T b) {
    return (a > b) ? a : b;
}

// 2. Class Template
template<typename K, typename V>
class KeyValuePair {
public:
    KeyValuePair(K key, V val) : key_(key), val_(val) {}
private:
    K key_;
    V val_;
};

// 3. Variable Template (C++14)
template<typename T>
constexpr T PI = T(3.1415926535897932385L);

// 4. Alias Template (using)
template<typename T>
using StringMap = std::unordered_map<std::string, T>;

int main() {
    std::cout << max_val(10, 20) << "\n";         // Instantiates max_val<int>
    std::cout << PI<double> << "\n";               // Instantiates PI<double>
    StringMap<int> age_db;                        // Instantiates std::unordered_map<std::string, int>
}
```

---

## 2. Template Instantiation & Compile-Time Overhead

Templates are pure blueprints; the compiler emits zero binary machine code for a template until it is instantiated with concrete type parameters.

```
Source Code (template<typename T> void process(T val))
                          │
       +------------------+------------------+
       | (Encounter process(42))            | (Encounter process(3.14))
       v                                     v
[Instantiates process<int>]           [Instantiates process<double>]
Emits machine code for int            Emits machine code for double
```

### Code Bloat Mitigation via `extern template` (C++11)
When a template is instantiated across 50 translation units (`.cpp` files), the compiler generates duplicate code in every `.o` file, relying on the linker to deduplicate them.

C++11 introduced `extern template` to suppress instantiation in specific TUs:

```cpp
// Header: matrix.h
template<typename T> class Matrix { /* ... */ };

// Source: matrix.cpp (Explicit Instantiation Definition)
template class Matrix<float>; // Compiler generates machine code once here!

// Other Source Files: (Explicit Instantiation Declaration)
extern template class Matrix<float>; // Prevents code generation in this TU!
```

---

## 3. Template Specialization

### Full vs Partial Specialization

- **Full Specialization**: Replaces the generic blueprint entirely for a specific, explicit type target.
- **Partial Specialization**: Customizes template implementation for a subset of types (e.g., all pointer types `T*`, or containers of `std::vector<T>`).

```cpp
#include <iostream>

// Primary Class Template
template<typename T>
struct Formatter {
    static void format(T val) {
        std::cout << "Generic format: " << val << "\n";
    }
};

// 1. Full Specialization (Targeting bool)
template<>
struct Formatter<bool> {
    static void format(bool val) {
        std::cout << "Bool format: " << (val ? "TRUE" : "FALSE") << "\n";
    }
};

// 2. Partial Specialization (Targeting all pointer types T*)
template<typename T>
struct Formatter<T*> {
    static void format(T* ptr) {
        std::cout << "Pointer format address: " << static_cast<const void*>(ptr) << "\n";
    }
};

int main() {
    int x = 42;
    Formatter<int>::format(x);     // Uses Primary Template
    Formatter<bool>::format(true); // Uses Full Specialization
    Formatter<int*>::format(&x);   // Uses Partial Specialization
}
```

---

## 4. Compile-Time Metaprogramming & Introspection (SFINAE vs Concepts)

### SFINAE (*Substitution Failure Is Not An Error*) (C++11)
SFINAE states that if an invalid type substitution occurs during template overload candidate evaluation, the compiler discards the candidate without raising a compile error:

```cpp
#include <iostream>
#include <type_traits>

// Enable only for integral types via SFINAE
template<typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
numeric_operation(T val) {
    std::cout << "Integral operation: " << val << "\n";
}

// Enable only for floating point types via SFINAE
template<typename T>
typename std::enable_if<std::is_floating_point<T>::value, void>::type
numeric_operation(T val) {
    std::cout << "Floating point operation: " << val << "\n";
}
```

### Modern C++20 Replacement: Concepts & Constraints
C++20 replaces verbose SFINAE template boilerplate with readable, compile-time verified **Concepts**:

```cpp
#include <iostream>
#include <concepts>

// Modern C++20 Concept constraint replaces std::enable_if!
template<std::integral T>
void modern_numeric_op(T val) {
    std::cout << "Modern integral op: " << val << "\n";
}

template<std::floating_point T>
void modern_numeric_op(T val) {
    std::cout << "Modern floating op: " << val << "\n";
}
```
