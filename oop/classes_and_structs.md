# Class & Struct Architecture & Member Initialization

Category: **Oop**

---

## 1. Technical Differences: `struct` vs `class`

In C++, `struct` and `class` keywords are functionally identical with only two specifier default differences:

1. **Default Member Access**:
   - `struct` defaults member access to **`public`**.
   - `class` defaults member access to **`private`**.

2. **Default Inheritance Visibility**:
   - `struct` defaults base class inheritance to **`public`**.
   - `class` defaults base class inheritance to **`private`**.

```cpp
class Base {};

// 1. Struct: Members & inheritance are public by default
struct DerivedStruct : Base {
    int x; // public
};

// 2. Class: Members & inheritance are private by default
class DerivedClass : Base { // Private inheritance from Base
    int x; // private
};
```

### Architectural Usage Conventions

| Paradigm | Target Use Case | Access Strategy | Primary Purpose |
| :--- | :--- | :--- | :--- |
| **`struct`** | **Plain Old Data (POD) / Aggregates** | Public data members | Bundling related passive data variables without strict internal invariants (e.g., `Point2D`, `RGBAColor`, `ConfigParams`). |
| **`class`** | **Encapsulated Domain Entities** | Private data members with public accessors | Enforcing internal invariants, managing dynamic resource lifecycles, and exposing abstraction interfaces. |

---

## 2. Member Initializer Lists vs In-Body Assignment

Understanding object construction mechanics is vital for writing high-performance C++ code. Object creation follows a two-phase execution pipeline:

```
[Phase 1: Initialization Phase] ---------> [Phase 2: Body Execution Phase]
Constructs members via Initializer List     Executes code inside Constructor {}
(Direct Construction in-place)              (Assignment via operator=)
```

### Direct Initialization vs In-Body Assignment Performance

#### Scenario A: Inefficient In-Body Assignment

```cpp
#include <string>
#include <iostream>

class Person {
public:
    Person(const std::string& name) {
        // IN-BODY ASSIGNMENT
        name_ = name; 
    }
private:
    std::string name_;
};
```

**Execution Sequence**:
1. **Phase 1 (Implicit Default Construction)**: Before entering `{}` constructor body, `name_` is initialized using `std::string()` default constructor (allocating an empty string buffer).
2. **Phase 2 (Assignment Operator)**: Inside `{}` body, `name_.operator=(name)` overwrites the existing empty string with a copy of `name`.
3. **Result**: Double initialization overhead (default construction + copy assignment).

#### Scenario B: Efficient Member Initializer List

```cpp
class Person {
public:
    // MEMBER INITIALIZER LIST
    Person(const std::string& name) : name_(name) {
        // Body left empty or used for validation
    }
private:
    std::string name_;
};
```

**Execution Sequence**:
1. **Phase 1 (Direct Construction)**: `name_` is constructed directly via `std::string(const std::string&)` copy constructor in-place.
2. **Phase 2 (Body Execution)**: Empty. Zero redundant operations performed.

---

## 3. Mandatory Use Cases for Initializer Lists

Member initializer lists are strictly required (compilation fails otherwise) in four fundamental scenarios:

```cpp
class Dependency {
public:
    Dependency(int id) {} // No default constructor available!
};

class MandatoryExample {
public:
    MandatoryExample(int& ref, int val, int dep_id) 
        : ref_member_(ref),       // 1. Reference members MUST be bound upon initialization
          const_member_(val),     // 2. Const members CANNOT be assigned to after creation
          dep_member_(dep_id)     // 3. Members lacking default constructors MUST be initialized explicitly
    {}

private:
    int& ref_member_;
    const int const_member_;
    Dependency dep_member_;
};
```

### Declaration Order Execution Rule
Data members are **always initialized in the order they are declared inside the class definition**, NOT the order listed in the member initializer list!

```cpp
class OrderExample {
public:
    // BAD PRACTICE: Initializer list implies 'b' is initialized first, but 'a' is initialized first!
    OrderExample(int val) : b(val), a(b * 2) {} // CAUSES UNDEFINED BEHAVIOR: 'b' is uninitialized when computing 'a'!

private:
    int a; // Declared 1st -> Initialized 1st
    int b; // Declared 2nd -> Initialized 2nd
};
```

---

## 4. In-Class Member Initializers (C++11 Default Initializers)

C++11 enables default member initializers directly inside class definitions:

```cpp
class Configuration {
private:
    int port_ = 8080;               // In-class default initializer
    std::string host_ = "localhost"; // In-class default initializer

public:
    Configuration() = default; // Uses in-class defaults: port_=8080, host_="localhost"

    // Constructor initializer list overrides in-class default!
    Configuration(int custom_port) : port_(custom_port) {} // port_=custom_port, host_="localhost"
};
```
