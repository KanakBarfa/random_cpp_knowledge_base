# C++17 Standard Features & Modern Idioms

Category: **Standards**

---

## 1. Executive Summary

C++17 was a major feature release that modernized expressive syntax and system capabilities. Key advancements include structured bindings, selection statements with built-in initializers, class template argument deduction (CTAD), fold expressions, native cross-platform filesystem support (`std::filesystem`), and type-safe vocabulary types (`std::optional`, `std::variant`, `std::any`, `std::string_view`).

---

## 2. Core Language Enhancements

### 1. Structured Bindings
Allows unpacking elements of tuples, pairs, structs, or fixed arrays into distinct local variables in a single statement:

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<int, std::string> db = {{1, "Alpha"}, {2, "Beta"}};

    // Unpack map key-value pair directly in loop
    for (const auto& [id, name] : db) {
        std::cout << "ID: " << id << ", Name: " << name << "\n";
    }
}
```

### 2. Selection Statements with Initializers (`if`/`switch`)
Scopes temporary variables directly inside `if` or `switch` condition blocks, preventing variable leakage into enclosing scopes:

```cpp
#include <iostream>
#include <map>

void lookup(const std::map<int, std::string>& m, int key) {
    // 'it' is tightly scoped to if/else blocks
    if (auto it = m.find(key); it != m.end()) {
        std::cout << "Found: " << it->second << "\n";
    } else {
        std::cout << "Key " << key << " not found\n";
    }
}
```

### 3. Fold Expressions
Simplifies variadic template operations by evaluating binary operators over parameter packs:

```cpp
#include <iostream>

// Binary Left Fold Expression: ((args... + ...))
template<typename... Args>
auto sum(Args... args) {
    return (... + args);
}

int main() {
    std::cout << sum(1, 2, 3, 4, 5) << "\n"; // Evaluates to (((1 + 2) + 3) + 4) + 5 = 15
}
```

### 4. Inline Variables
Enables defining global constants or static class fields inside header files without violating the One Definition Rule (ODR):

```cpp
// Header: settings.h
#pragma once
#include <string_view>

inline constexpr std::string_view K_DEFAULT_HOST = "127.0.0.1";
inline int g_connection_count = 0;
```

---

## 3. Standard Library Modernization

### 1. Vocabulary Types (`std::optional`, `std::variant`, `std::any`, `std::string_view`)

```cpp
#include <optional>
#include <string_view>
#include <iostream>

// std::optional: Handles optional return values without pointers
std::optional<int> parse_int(std::string_view str) {
    try {
        return std::stoi(std::string(str));
    } catch (...) {
        return std::nullopt; // Represents absence of value
    }
}

int main() {
    if (auto result = parse_int("42"); result.has_value()) {
        std::cout << "Parsed: " << *result << "\n";
    }
}
```

### 2. Standard Filesystem Library (`std::filesystem`)
Cross-platform file and directory manipulation engine:

```cpp
#include <iostream>
#include <filesystem>

namespace fs = std::filesystem;

int main() {
    fs::path p = "/var/log/syslog";
    if (fs::exists(p)) {
        std::cout << "File size: " << fs::file_size(p) << " bytes\n";
    }
}
```

### 3. Parallel Execution Algorithms (`<execution>`)
Standard algorithm overloads executing across multi-threaded CPU cores:

```cpp
#include <vector>
#include <algorithm>
#include <execution>

int main() {
    std::vector<int> vec(1'000'000, 42);
    // Parallel execution policy sorts data concurrently across multi-core CPUs
    std::sort(std::execution::par, vec.begin(), vec.end());
}
```
