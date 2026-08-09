# Unions Architecture & Type-Safe Variant Alternatives

Category: **Templates And Metaprogramming**

---

## 1. C-Style `union` Sizing & Overlapping Memory Rules

A `union` is a low-level type construct where all data members share the **exact same starting memory address**.

```cpp
union DataPayload {
    char c;       // 1 byte
    int i;        // 4 bytes
    double d;     // 8 bytes
};
```

```
DataPayload Memory Layout (8 Bytes Total):
+-------------------------------------------------------+
|  char c (Byte 0)                                      |
+-------------------------------------------------------+
|  int i  (Bytes 0 - 3)                                 |
+-------------------------------------------------------+
|  double d (Bytes 0 - 7)                               |
+-------------------------------------------------------+
^
Starting Address (&u.c == &u.i == &u.d)
```

### `sizeof` Evaluation Rules

1. **Largest Member Rule**: The size of a union must be at least as large as its largest member field.
2. **Alignment Padding Rule**: The size is padded up to a multiple of the member with the largest alignment requirement (`alignof`).
3. **Compile-Time Constancy**: `sizeof(union)` is a compile-time constant expression. The footprint does **not** change at runtime based on which member is currently assigned or active.

```cpp
#include <iostream>

union Sample {
    char c;     // 1B (align 1)
    int i;      // 4B (align 4)
    double d;   // 8B (align 8)
};

int main() {
    Sample s;
    std::cout << "Union size: " << sizeof(Sample) << " bytes\n"; // Output: 8 bytes
    s.c = 'A';
    std::cout << "Size after char assignment: " << sizeof(s) << " bytes\n"; // STILL 8 bytes!
}
```

---

## 2. Incomplete vs Complete Types

Attempting to evaluate `sizeof` on an incomplete (forward-declared) union type before its definition brace closes generates a compile error:

```cpp
// 1. Forward Declaration (Incomplete Type)
union Buffer;

// int s = sizeof(Buffer); // COMPILE ERROR: Cannot compute size of incomplete type!

// 2. Definition (Type Completeness Reached)
union Buffer {
    int id;
    float value;
};

// 3. Usage (Complete Type)
int s = sizeof(Buffer); // OK: Evaluates to 4 bytes
```

---

## 3. C-Style `union` Safety Hazards & Undefined Behavior

Under standard C++ strict aliasing rules, writing to one union member (e.g., `d`) and reading from a different member (e.g., `i`) triggers **Undefined Behavior (UB)**:

```cpp
union Payload {
    int i;
    double d;
};

Payload p;
p.d = 3.14159;
// int val = p.i; // UNDEFINED BEHAVIOR: Reading inactive union member!
```

---

## 4. Modern Type-Safe Alternative: `std::variant` (C++17)

C++17 introduced `std::variant<Ts...>`, a type-safe discriminated union that tracks the active member type index at runtime.

### `std::variant` Memory Footprint Formula

$$\text{sizeof}(\text{std::variant}) = \text{sizeof}(\text{Largest Member}) + \text{sizeof}(\text{Discriminant Index}) + \text{Alignment Padding}$$

```cpp
#include <variant>
#include <string>
#include <iostream>

using ValueVariant = std::variant<int, double, std::string>;

int main() {
    ValueVariant v = 42; // Currently holds int

    // 1. Check active alternative
    if (std::holds_alternative<int>(v)) {
        std::cout << "Holds int: " << std::get<int>(v) << "\n";
    }

    // 2. Assign new type safely (destructs int, constructs std::string)
    v = std::string("Hello Modern C++");

    // 3. Safe pointer extraction with std::get_if
    if (auto str_ptr = std::get_if<std::string>(&v)) {
        std::cout << "Holds string: " << *str_ptr << "\n";
    }
}
```

### Pattern Matching via `std::visit` & Overloaded Lambdas
`std::visit` allows type-safe pattern matching across variant alternatives at compile time:

```cpp
#include <variant>
#include <iostream>
#include <string>

// Overload pattern matching helper pattern
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

int main() {
    std::variant<int, double, std::string> var = "Pattern Matching";

    std::visit(overloaded {
        [](int arg) { std::cout << "int: " << arg << "\n"; },
        [](double arg) { std::cout << "double: " << arg << "\n"; },
        [](const std::string& arg) { std::cout << "string: " << arg << "\n"; }
    }, var);
}
```
