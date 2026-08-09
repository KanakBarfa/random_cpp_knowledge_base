# Struct/Class Memory Layout, Alignment & Sizing

Category: **Memory**

---

## 1. Memory Alignment & Padding Rules

Hardware memory architectures access memory in multi-byte words (typically 4 or 8 bytes). To optimize CPU bus transfers and prevent hardware alignment faults, the compiler enforces **natural memory alignment**: a primitive data type of size $S$ must reside at a memory address divisible by $S$.

### Struct Padding & Layout Calculation

To satisfy alignment constraints, compilers insert uninitialized **padding bytes** between struct members and at the end of the structure:

```cpp
#include <iostream>

struct Unoptimized {
    char c;     // 1 byte
    // 3 padding bytes inserted here!
    int i;      // 4 bytes (aligned to 4-byte boundary)
    short s;    // 2 bytes
    // 2 padding bytes inserted at end so total size is multiple of max_align (4)
};

int main() {
    std::cout << "Unoptimized size: " << sizeof(Unoptimized) << " bytes\n"; // Output: 12 bytes
}
```

```
Unoptimized Memory Memory Layout (12 Bytes Total):
+---------+-------------------+-------------------+-------------------+
|  c (1B) |   Padding (3B)    |      i (4B)       |      s (2B) | Pad |
+---------+-------------------+-------------------+-------------------+
| byte 0  | bytes 1, 2, 3     | bytes 4, 5, 6, 7  | bytes 8, 9| 10,11 |
+---------+-------------------+-------------------+-------------------+
```

### Minimizing Padding via Member Reordering
Arranging struct members in descending order of size eliminates padding gaps:

```cpp
struct Optimized {
    int i;      // 4 bytes
    short s;    // 2 bytes
    char c;     // 1 byte
    // 1 padding byte inserted at end
};

int main() {
    std::cout << "Optimized size: " << sizeof(Optimized) << " bytes\n"; // Output: 8 bytes
}
```

### Alignment Specifiers (`alignof`, `alignas`, `#pragma pack`)

```cpp
#include <iostream>

// Enforce custom 16-byte alignment (useful for SIMD AVX registers)
struct alignas(16) AVXVector {
    float data[4];
};

// Packed structure (0 padding bytes; used for network protocols / binary hardware formats)
#pragma pack(push, 1)
struct PackedHeader {
    char type;
    int length;
};
#pragma pack(pop)

int main() {
    std::cout << "AVXVector alignment: " << alignof(AVXVector) << "\n"; // Output: 16
    std::cout << "PackedHeader size: "   << sizeof(PackedHeader) << "\n"; // Output: 5 bytes
}
```

---

## 2. Empty Class Sizing & Empty Base Optimization (EBO)

In C++, an empty class or struct containing no data members has a size of **1 byte**:

```cpp
struct EmptyClass {};
std::cout << sizeof(EmptyClass); // Output: 1 byte
```

### Why Size Is Not Zero: Object Identity Rule
The C++ ISO Standard mandates that every distinct object must have a unique memory address. If `sizeof(EmptyClass)` were 0, elements in an array `EmptyClass arr[10]` would share identical memory addresses (`&arr[0] == &arr[1]`), breaking pointer arithmetic and object identity guarantees.

### Empty Base Optimization (EBO)
When an empty class is used as a **base class** in an inheritance hierarchy, the compiler optimizes away its 1-byte placeholder overhead:

```cpp
#include <iostream>

struct EmptyBase {};

struct Derived : public EmptyBase {
    int value; // 4 bytes
};

int main() {
    // EBO reduces EmptyBase footprint inside Derived to 0 bytes!
    std::cout << "Size of Derived: " << sizeof(Derived) << " bytes\n"; // Output: 4 bytes (not 5 or 8!)
}
```

---

## 3. C++20 Modern Alternative: `[[no_unique_address]]`

Prior to C++20, library developers forced inheritance structures purely to leverage EBO for stateless allocators or custom deleters. C++20 introduced `[[no_unique_address]]` to apply EBO directly to class member variables:

```cpp
#include <iostream>

struct StatelessDeleter {};

// Pre-C++20: Forced inheritance for EBO
template<typename T>
class UniquePtrLegacy : private StatelessDeleter {
    T* ptr;
};

// Modern C++20: Clean member composition with [[no_unique_address]]
template<typename T>
class UniquePtrModern {
    T* ptr; // 8 bytes
    [[no_unique_address]] StatelessDeleter deleter; // 0 bytes footprint!
};

int main() {
    std::cout << "Modern UniquePtr size: " << sizeof(UniquePtrModern<int>) << " bytes\n"; // Output: 8 bytes
}
```

---

## 4. Polymorphic Class Overhead (`vptr` / `vtable`)

When a class declares or inherits at least one `virtual` function, the compiler inserts a hidden **Virtual Table Pointer (`vptr`)** into the object layout.

```cpp
#include <iostream>

struct NonPolymorphic {
    int data; // 4 bytes -> padded to 4
};

struct Polymorphic {
    int data; // 4 bytes
    virtual void process() {} // Adds 8-byte vptr on 64-bit architecture
};

int main() {
    std::cout << "NonPolymorphic size: " << sizeof(NonPolymorphic) << " bytes\n"; // Output: 4 bytes
    std::cout << "Polymorphic size:    " << sizeof(Polymorphic)    << " bytes\n"; // Output: 16 bytes (8B vptr + 4B data + 4B pad)
}
```

### Polymorphic Class Memory Layout Diagram

```
Polymorphic Object Instance Memory (16 Bytes Total):
+-------------------------------------------------------+
|  vptr (Virtual Pointer -> Points to Class VTable)     | (Bytes 0 - 7)
+-------------------------------------------------------+
|  data (int)                     | Alignment Pad (4B)  | (Bytes 8 - 15)
+-------------------------------------------------------+
                                  |
                                  v
                  Class VTable (Read-Only Static Memory)
                  +-----------------------------------+
                  |  &Polymorphic::process()          |
                  |  &Polymorphic::~Polymorphic()     |
                  +-----------------------------------+
```

### Multiple & Virtual Inheritance Layout Impact
- **Multiple Inheritance**: A derived class inheriting from $M$ polymorphic base classes contains $M$ distinct `vptr` pointers, increasing object size by $M \times 8$ bytes.
- **Virtual Base Inheritance (`virtual public Base`)**: Resolves the Diamond Inheritance Problem by inserting a hidden **Virtual Base Pointer (`vbase`)** to track dynamic base offsets, adding pointer overhead.
