# C++20 Standard Features & Paradigm Shift

Category: **Standards**

---

## 1. Executive Summary

C++20 is widely regarded as a monumental architectural release on par with C++11. It introduced four transformative paradigm pillars: **Concepts & Constraints**, **Ranges & Views**, **Coroutines**, and **C++ Modules**, alongside the Three-Way Comparison Operator (`<=>`), `std::format`, and immediate functions (`consteval`).

---

## 2. The Four Pillars of C++20

### 1. Concepts & Constraints
Concepts replace cryptic multi-page template compiler errors with readable compile-time type verification contracts:

```cpp
#include <iostream>
#include <concepts>

// Define a custom concept constraining type requirements
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

// Constrain template function using numeric concept
template<Numeric T>
T multiply(T a, T b) {
    return a * b;
}

int main() {
    multiply(10, 20);     // OK: int satisfies Numeric concept
    multiply(3.14, 2.0);  // OK: double satisfies Numeric concept
    // multiply("A", "B"); // COMPILE ERROR: std::string fails Numeric concept!
}
```

### 2. Ranges & Views (`std::ranges`)
Ranges allow functional pipeline composition over data sequences with **lazy evaluation**:

```cpp
#include <iostream>
#include <vector>
#include <ranges>

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Lazy processing pipeline: Filter even numbers, then square them
    auto pipeline = numbers 
        | std::views::filter([](int n) { return n % 2 == 0; })
        | std::views::transform([](int n) { return n * n; });

    // Operations evaluate on-the-fly during iteration!
    for (int val : pipeline) {
        std::cout << val << " "; // Output: 4 16 36 64 100
    }
}
```

### 3. Coroutines (`co_yield`, `co_await`, `co_return`)
Coroutines are functions whose execution can be suspended and resumed, enabling asynchronous I/O and lazy generators:

```cpp
#include <iostream>
#include <coroutine>

// Coroutines introduce keywords: co_yield (generator), co_await (async wait), co_return (finish)
// Requires a promise_type handler structure
```

### 4. C++ Modules (`export module`, `import`)
Modules replace `#include` header preprocessor inclusions, solving macro isolation, header ordering bugs, and accelerating build speeds by up to 10x:

```cpp
// Module Interface Unit: math_utils.ixx
export module math_utils;

export int add(int a, int b) {
    return a + b;
}

// Consumer File: main.cpp
import math_utils;

int main() {
    int result = add(5, 10);
}
```

---

## 3. Spaceship Operator (`<=>`) & Modern Comparison

The Three-Way Comparison Operator calculates ordering relation (`<`, `==`, `>`) in a single function call. Specifying `= default` auto-generates all 6 relational operators:

```cpp
#include <iostream>
#include <compare>

struct Point {
    int x;
    int y;

    // Compiler automatically generates ==, !=, <, <=, >, >=
    auto operator<=>(const Point&) const = default;
};

int main() {
    Point p1{1, 2};
    Point p2{1, 3};

    if (p1 < p2) {
        std::cout << "p1 is smaller\n";
    }
}
```

---

## 4. Compile-Time Function Execution (`constexpr` vs `consteval` vs `constinit`)

| Keyword | Evaluation Guarantee | Fallback to Runtime? | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **`constexpr`** | Evaluates at compile time if inputs are constant; otherwise executes at runtime. | **YES** | Flexible math & utility functions usable at build time or runtime. |
| **`consteval`** | **Immediate Function**: MUST evaluate at compile time. Runtime calls trigger compile errors. | **NO** | Compile-time validation (e.g., regex checks, format string verification). |
| **`constinit`** | Guarantees static/thread-local variable is initialized at compile time. | N/A | Prevents Static Initialization Order Fiasco for global variables. |

```cpp
consteval int strict_compile_time(int n) {
    return n * 2;
}

int main() {
    constexpr int a = strict_compile_time(5); // OK: Evaluates at compile time
    int runtime_var = 10;
    // int b = strict_compile_time(runtime_var); // COMPILE ERROR: runtime_var is not constant!
}
```

---

## 5. Type-Safe Formatting (`std::format`)

Provides Python-style type-safe string formatting without `iostream` verbosity or `printf` security bugs:

```cpp
#include <format>
#include <iostream>
#include <string>

int main() {
    std::string msg = std::format("User ID: {}, Score: {:.2f}", 101, 98.456);
    std::cout << msg << "\n"; // Output: User ID: 101, Score: 98.46
}
```
