# Constructor Architecture, Lifecycle & Exception Safety

Category: **Oop**

---

## 1. The Rule of Zero / Three / Five

C++ class lifecycles are governed by Special Member Functions. Modern C++ architecture prescribes management rules depending on whether a class directly owns low-level memory resources:

```
                  Does the class manage a raw resource handle?
                                       |
                   +-------------------+-------------------+
                   |                                       |
                   NO                                     YES
                   v                                       v
          [RULE OF ZERO]                            [RULE OF FIVE]
   Rely on RAII types (smart pointers,       Implement custom:
   std::vector, std::string). Allow          1. Destructor
   compiler to auto-generate all 5           2. Copy Constructor
   special functions.                        3. Copy Assignment Operator
                                             4. Move Constructor
                                             5. Move Assignment Operator
```

---

## 2. Comprehensive Constructor Types & Mechanics

### 1. Default Constructor
Constructs an object without arguments. If any user-defined constructor is declared, the compiler **suppresses** automatic default constructor generation:

```cpp
class Widget {
public:
    Widget() = default; // Explicitly request default constructor compiler generation
    explicit Widget(int id) : id_(id) {}
private:
    int id_ = 0;
};
```

### 2. Copy Constructor (Deep vs Shallow Copy)
Creates a new object by copying an existing object instance of the same class.

- **Shallow Copy (Compiler Default)**: Copies member fields bit-for-bit. If a member is a raw pointer, both copies point to the same memory, causing **double-free crashes** on destruction.
- **Deep Copy (Required for Raw Resources)**: Allocates independent heap memory and duplicates underlying data:

```cpp
#include <iostream>
#include <cstring>

class Buffer {
public:
    explicit Buffer(size_t size) : size_(size), data_(new char[size]) {}

    // Deep Copy Constructor
    Buffer(const Buffer& other) : size_(other.size_), data_(new char[other.size_]) {
        std::memcpy(data_, other.data_, size_);
    }

    ~Buffer() { delete[] data_; }

private:
    size_t size_;
    char* data_;
};
```

### 3. Move Constructor (Resource Stealing) (C++11)
Transfers ownership of dynamic resources from a temporary rvalue instance to a new instance without deep copying. **Must be marked `noexcept`**:

```cpp
// Move Constructor
Buffer(Buffer&& other) noexcept 
    : size_(other.size_), data_(other.data_) {
    // Invalidate source object pointers so its destructor won't free stolen memory!
    other.size_ = 0;
    other.data_ = nullptr;
}
```

### 4. Delegating Constructors (C++11)
Allows constructors within the same class to invoke sibling constructors in their initializer lists, eliminating duplicate setup logic:

```cpp
class Server {
public:
    // Master Constructor
    Server(std::string host, int port) 
        : host_(std::move(host)), port_(port) {}

    // Delegating Constructors
    explicit Server(int port) : Server("127.0.0.1", port) {}
    Server() : Server("127.0.0.1", 8080) {}

private:
    std::string host_;
    int port_;
};
```

---

## 3. Constructor Exception Safety & Stack Unwinding

A fundamental invariant of C++ object lifecycles states:

> **If a constructor throws an exception, the object is considered to have NEVER been successfully created. Consequently, the class's own destructor will NOT be called.**

```
Constructor Invoked
       │
[Member A Constructed] ──► Success
       │
[Member B Construction] ──► THROWS EXCEPTION!
       │
  (UNWINDING)
       ├──► Member A Destructor INVOKED automatically!
       ├──► Member B Destructor SKIPPED (was never constructed)
       └──► Class Destructor (~MyClass) SKIPPED!
```

### 1. Memory Leak Pitfall: The Raw Pointer Failure

```cpp
#include <iostream>
#include <stdexcept>

class MemoryLeakDanger {
public:
    MemoryLeakDanger() {
        resA_ = new int[100]; // Raw Allocation 1: Succeeds
        
        throw std::runtime_error("Construction failed halfway!"); // Exception thrown!

        resB_ = new int[100]; // Never reached
    }

    ~MemoryLeakDanger() {
        // NEVER EXECUTED because constructor failed!
        delete[] resA_; // MEMORY LEAKED FOREVER!
        delete[] resB_;
    }

private:
    int* resA_;
    int* resB_;
};
```

### 2. Exception-Safe Solution: RAII & Smart Pointers

Using smart pointers (`std::unique_ptr`) guarantees exception safety. During stack unwinding, any fully-constructed member variables have their destructors called automatically, even though the parent class destructor is skipped:

```cpp
#include <iostream>
#include <memory>
#include <stdexcept>

class ExceptionSafeRAII {
public:
    ExceptionSafeRAII() 
        : resA_(std::make_unique<int[]>(100)) { // Member resA_ fully constructed
        
        throw std::runtime_error("Construction failed halfway!");
        
        // resB_ is never initialized, but resA_'s destructor runs automatically 
        // during stack unwinding, cleaning up memory safely!
    }

    ~ExceptionSafeRAII() = default;

private:
    std::unique_ptr<int[]> resA_;
    std::unique_ptr<int[]> resB_;
};
```
