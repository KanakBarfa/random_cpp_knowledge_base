# Value Categories, Move Semantics & Perfect Forwarding

Category: **Templates And Metaprogramming**

---

## 1. ISO C++ Value Category Taxonomy

Every C++ expression is characterized by two independent properties:
1. **Has Identity**: The program holds a memory address pointing to the object (can be evaluated via address-of operator `&`).
2. **Can Be Moved From**: The object can be bound to an rvalue reference, allowing its resources to be stolen because its lifetime is expiring.

```
                      Expressions
                     /           \
                    /             \
            glvalue                 rvalue
         (has identity)       (can be moved from)
           /        \         /        \
          /          \       /          \
      lvalue          xvalue          prvalue
(has identity,     (has identity,  (no identity,
 can't be moved)   can be moved)   can be moved)
```

### Definitions & Expression Classification

| Value Category | Identity? | Movable? | Expression Examples |
| :--- | :--- | :--- | :--- |
| **`lvalue`** | **YES** | NO | Named variables (`int x`), reference return functions (`T& get()`), array subscripting (`arr[i]`), string literals (`"Hello"`). |
| **`prvalue`** (*pure rvalue*) | NO | **YES** | Numeric literals (`42`, `3.14`), temporary objects (`std::string("data")`), value return functions (`int add()`). |
| **`xvalue`** (*eXpiring value*) | **YES** | **YES** | Result of `std::move(var)`, functions returning rvalue references (`T&&`). |

---

## 2. Reference Binding Rules

| Reference Type | Binds to `lvalue`? | Binds to `prvalue`? | Binds to `xvalue`? |
| :--- | :--- | :--- | :--- |
| **Lvalue Reference (`T&`)** | **YES** | NO | NO |
| **Const Lvalue Reference (`const T&`)** | **YES** | **YES** (extends temporary lifetime) | **YES** |
| **Rvalue Reference (`T&&`)** | NO | **YES** | **YES** |

```cpp
int x = 10;

int& ref1 = x;                 // OK: Lvalue reference binds to lvalue
// int& ref2 = 42;             // COMPILE ERROR: Cannot bind non-const lvalue ref to prvalue
const int& ref3 = 42;          // OK: Const lvalue reference extends prvalue lifetime!

int&& rref1 = 42;              // OK: Rvalue reference binds to prvalue
int&& rref2 = std::move(x);    // OK: Rvalue reference binds to xvalue
// int&& rref3 = x;            // COMPILE ERROR: Cannot bind rvalue ref to lvalue
```

---

## 3. Move Semantics & `std::move` Mechanics

### What `std::move` Actually Does
`std::move` does **not** move any bytes or execute runtime machine code instructions. It is purely a compile-time static cast converting an expression into an rvalue reference (`xvalue`):

```cpp
template<typename T>
constexpr std::remove_reference_t<T>&& move(T&& arg) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(arg);
}
```

### Move Constructor & Move Assignment Implementation

```cpp
#include <iostream>
#include <utility>
#include <cstring>

class DynamicBuffer {
public:
    explicit DynamicBuffer(size_t size) 
        : size_(size), data_(new char[size]) {}

    ~DynamicBuffer() { delete[] data_; }

    // 1. Copy Constructor (Expensive Deep Copy)
    DynamicBuffer(const DynamicBuffer& other) 
        : size_(other.size_), data_(new char[other.size_]) {
        std::memcpy(data_, other.data_, size_);
    }

    // 2. Move Constructor (Cheap Pointer Transfer)
    DynamicBuffer(DynamicBuffer&& other) noexcept 
        : size_(other.size_), data_(other.data_) {
        // Leave moved-from object in a valid but empty state
        other.size_ = 0;
        other.data_ = nullptr;
    }

    // 3. Move Assignment Operator
    DynamicBuffer& operator=(DynamicBuffer&& other) noexcept {
        if (this != &other) {
            delete[] data_; // Release existing resource

            size_ = other.size_;
            data_ = other.data_;

            other.size_ = 0;
            other.data_ = nullptr;
        }
        return *this;
    }

private:
    size_t size_;
    char* data_;
};
```

---

## 4. Forwarding References (`T&&`) & Perfect Forwarding

### Universal / Forwarding Reference Context
A `T&&` parameter inside a template function with type deduction is a **Forwarding Reference** (not an rvalue reference):

```cpp
template<typename T>
void wrapper(T&& arg); // T&& is a Forwarding Reference!
```

### Reference Collapsing Rules
When a forwarding reference deduces a argument type, reference collapsing rules apply:

$$\text{\tt \& + \& } \longrightarrow \text{\tt \&}$$
$$\text{\tt \& + \&\& } \longrightarrow \text{\tt \&}$$
$$\text{\tt \&\& + \& } \longrightarrow \text{\tt \&}$$
$$\text{\tt \&\& + \&\& } \longrightarrow \text{\tt \&\&}$$

### Perfect Forwarding with `std::forward`
`std::forward<T>(arg)` preserves the exact value category (lvalue or rvalue) of the original passed argument:

```cpp
#include <iostream>
#include <utility>

void process(const int& x) { std::cout << "Processed as Lvalue\n"; }
void process(int&& x)      { std::cout << "Processed as Rvalue\n"; }

template<typename T>
void forwarder(T&& arg) {
    // std::forward restores original value category during argument passing
    process(std::forward<T>(arg));
}

int main() {
    int val = 100;

    forwarder(val);        // Output: Processed as Lvalue
    forwarder(42);         // Output: Processed as Rvalue
    forwarder(std::move(val)); // Output: Processed as Rvalue
}
```
