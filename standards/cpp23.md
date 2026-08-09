# C++23 Standards & Improvements

Category: **Standards**

---

## 1. Executive Summary

C++23 serves as a major library expansion and syntax refinement standard following C++20. Highlighting features include `std::print`/`std::println`, monadic error handling with `std::expected`, explicit object parameters ("Deducing `this`"), multidimensional subscripting, monadic `std::optional` extensions, and full standard library modules (`import std;`).

---

## 2. Modern Console Output (`std::print` & `std::println`)

C++23 introduced the `<print>` header, bypassing slow, verbose `iostream` streams and providing ultra-fast, unicode-aware, type-safe console formatting directly to standard output:

```cpp
#include <print>
#include <string>

int main() {
    std::string user = "Administrator";
    int active_connections = 42;

    // std::println automatically appends a newline character
    std::println("System Status: User '{}' has {} active sessions", user, active_connections);
}
```

---

## 3. Modern Error Handling: `std::expected<T, E>`

`std::expected<T, E>` represents a vocabulary type that contains either an expected return value of type `T` or an error object of type `E`. It eliminates exception overhead while enforcing explicit error checking:

```cpp
#include <expected>
#include <string>
#include <print>

enum class MathError { DivisionByZero, NegativeLogarithm };

std::expected<double, MathError> safe_divide(double num, double denom) {
    if (denom == 0.0) {
        return std::unexpected(MathError::DivisionByZero);
    }
    return num / denom;
}

int main() {
    auto result = safe_divide(10.0, 0.0);

    if (result) {
        std::println("Division result: {}", *result);
    } else {
        if (result.error() == MathError::DivisionByZero) {
            std::println("Error: Cannot divide by zero!");
        }
    }
}
```

---

## 4. Language Syntax Innovations

### 1. Explicit Object Parameter ("Deducing `this`")
Allows member functions to accept the `this` instance explicitly as a parameter, eliminating duplicate `const` and `rvalue` function overloads and simplifying recursive lambdas:

```cpp
#include <print>

struct FibonacciCalculator {
    // Explicit 'this' parameter enables direct recursive lambda invocation
    auto calculate(this auto self, int n) -> int {
        if (n <= 1) return n;
        return self.calculate(n - 1) + self.calculate(n - 2);
    }
};

int main() {
    FibonacciCalculator calc;
    std::println("Fib(10) = {}", calc.calculate(10));
}
```

### 2. Multidimensional Subscript Operator (`operator[]`)
Enables classes to accept multiple arguments inside square bracket subscript expressions:

```cpp
#include <vector>
#include <print>

class Matrix2D {
public:
    Matrix2D(size_t rows, size_t cols) 
        : cols_(cols), data_(rows * cols, 0) {}

    // C++23 Multidimensional Subscripting
    int& operator[](size_t r, size_t c) {
        return data_[r * cols_ + c];
    }

private:
    size_t cols_;
    std::vector<int> data_;
};

int main() {
    Matrix2D mat(3, 3);
    mat[1, 2] = 42; // Syntax enabled in C++23!
    std::println("Matrix value at (1,2): {}", mat[1, 2]);
}
```

---

## 5. Standard Library Enhancements

### 1. Monadic Operations for `std::optional` & `std::expected`
Chains operations cleanly via `.and_then()`, `.transform()`, and `.or_else()`:

```cpp
#include <optional>
#include <string>
#include <print>

std::optional<int> parse_str(std::string s) { return 42; }
std::optional<int> double_val(int v)        { return v * 2; }

int main() {
    std::optional<std::string> input = "42";

    // Monadic pipeline execution
    auto final_result = input
        .and_then(parse_str)
        .and_then(double_val);

    if (final_result) {
        std::println("Pipeline result: {}", *final_result);
    }
}
```

### 2. Extended Ranges Adaptors & `std::ranges::to`
C++23 introduced `std::views::zip`, `std::views::chunk`, `std::views::slide`, and `std::ranges::to` for converting range views back into concrete STL containers:

```cpp
#include <vector>
#include <ranges>
#include <print>

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6};

    // Convert pipeline view directly into a std::vector using std::ranges::to
    auto squared_vec = numbers 
        | std::views::transform([](int n) { return n * n; })
        | std::ranges::to<std::vector>();
}
```

### 3. Full Standard Library Modules (`import std;`)
C++23 standardizes importing the entire ISO Standard Library as a single module unit:

```cpp
import std; // Imports all standard containers, algorithms, and I/O types in one line!

int main() {
    std::println("Hello C++23 Standard Library Module!");
}
```
