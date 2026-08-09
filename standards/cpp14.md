# C++14 Standard Features & Quality-of-Life Evolution

Category: **Standards**

---

## 1. Executive Summary

C++14 represents a targeted refinement release designed to polish features introduced in C++11. It eliminated major library omissions (such as `std::make_unique`), relaxed compile-time `constexpr` evaluation rules, and expanded lambda capabilities with generic parameter deduction and capture expressions.

---

## 2. Core Language Enhancements

### 1. Generic Lambdas & Init-Captures
C++14 introduced `auto` parameter deduction in lambdas (effectively generating an anonymous closure with a template `operator()`) and generalized capture expressions:

```cpp
#include <iostream>
#include <memory>
#include <string>

int main() {
    // Generic Lambda: Operates on arbitrary types via auto deduction
    auto generic_add = [](auto a, auto b) { return a + b; };
    std::cout << generic_add(5, 10) << "\n";             // int addition
    std::cout << generic_add(std::string("A"), "B") << "\n"; // string concatenation

    // Lambda Init-Capture: Move resources directly into closure scope
    auto ptr = std::make_unique<int>(42);
    auto seeker = [captured_ptr = std::move(ptr)]() {
        std::cout << "Captured value: " << *captured_ptr << "\n";
    };
    seeker();
}
```

### 2. Function Return Type Deduction
Functions can deduce their return types using `auto`. If multiple return paths exist, all return expressions must evaluate to the exact same type:

```cpp
#include <vector>

// Compiler deduces return type as std::vector<int>
auto create_indices(int count) {
    std::vector<int> vec;
    for (int i = 0; i < count; ++i) vec.push_back(i);
    return vec;
}
```

### 3. Binary Literals & Digit Separators
Enhances code readability for bitmasks and large numeric constants:

```cpp
int mask = 0b1101'0000'0011'0110; // Binary literal with single-quote separator
long long distance_meters = 149'597'870'700; // Earth-Sun distance
```

### 4. `constexpr` Function Relaxations
C++11 restricted `constexpr` functions to a single `return` statement. C++14 relaxed these constraints to permit local variables, loops, conditional branching, and local mutations:

```cpp
constexpr long long compute_factorial(int n) {
    if (n < 0) return 0;
    long long result = 1;
    for (int i = 1; i <= n; ++i) { // Loops permitted in C++14 constexpr!
        result *= i;
    }
    return result;
}

static_assert(compute_factorial(5) == 120, "Compile-time evaluation verified");
```

---

## 3. Standard Library Additions

### `std::make_unique` Factory Function
C++11 provided `std::make_shared` but omitted `std::make_unique`. C++14 resolved this gap, providing a clean, exception-safe way to allocate unique smart pointers:

```cpp
#include <memory>

struct Sensor {};

void process(std::unique_ptr<Sensor> s, int priority) {}

int main() {
    // Pre-C++14 (Unsafe: Potential memory leak if get_priority() throws after new Sensor)
    // process(std::unique_ptr<Sensor>(new Sensor()), get_priority());

    // C++14 Exception-safe idiom
    process(std::make_unique<Sensor>(), 42);
}
```

### Deprecation Attribute (`[[deprecated]]`)
Standardized attribute alerting compiler users to outdated APIs:

```cpp
[[deprecated("Use process_v2() instead to prevent buffer truncation.")]]
void process_legacy() {}
```
