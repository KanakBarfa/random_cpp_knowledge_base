# Inheritance Modes, Polymorphism & Virtual Layouts

Category: **Oop**

---

## 1. Inheritance Access Modes (`public`, `protected`, `private`)

In C++, the access specifier keyword placed before the base class name acts as an **access filter (ceiling)** for members inherited from that base class.

| Base Member Access | `public` Inheritance Mode | `protected` Inheritance Mode | `private` Inheritance Mode |
| :--- | :--- | :--- | :--- |
| **`public`** | Remains **`public`** in derived class | Becomes **`protected`** in derived class | Becomes **`private`** in derived class |
| **`protected`** | Remains **`protected`** in derived class | Remains **`protected`** in derived class | Becomes **`private`** in derived class |
| **`private`** | **Inaccessible** to derived class | **Inaccessible** to derived class | **Inaccessible** to derived class |

### Code Implementation & Verification

```cpp
class Base {
public:
    int pub_val = 1;
protected:
    int prot_val = 2;
private:
    int priv_val = 3;
};

// 1. Public Inheritance (IS-A Relationship)
class PublicDerived : public Base {
    void verify() {
        pub_val = 10;  // OK: Remains public
        prot_val = 20; // OK: Remains protected
        // priv_val = 30; // COMPILE ERROR: Base private members are inaccessible
    }
};

// 2. Protected Inheritance (Restricted Interface)
class ProtectedDerived : protected Base {
    void verify() {
        pub_val = 10;  // OK: Becomes protected inside ProtectedDerived
        prot_val = 20; // OK: Remains protected
    }
};

// 3. Private Inheritance (Implemented-In-Terms-Of)
class PrivateDerived : private Base {
    void verify() {
        pub_val = 10;  // OK: Becomes private inside PrivateDerived
        prot_val = 20; // OK: Becomes private inside PrivateDerived
    }
};

int main() {
    PublicDerived pub;
    pub.pub_val = 100; // OK: Accessible externally

    ProtectedDerived prot;
    // prot.pub_val = 100; // COMPILE ERROR: pub_val is protected externally

    PrivateDerived priv;
    // priv.pub_val = 100; // COMPILE ERROR: pub_val is private externally
}
```

---

## 2. Polymorphism & Virtual Function Contracts

### Virtual Destructors Critical Requirement
Any class declaring at least one `virtual` function **must declare a `virtual` destructor**. Deleting a derived instance via a base pointer when the base destructor is non-virtual causes **undefined behavior and memory leaks**:

```cpp
class Base {
public:
    virtual ~Base() = default; // Mandatory virtual destructor!
    virtual void speak() const = 0; // Pure virtual function -> Abstract Class
};

class Derived : public Base {
public:
    Derived() : data_(new int[100]) {}
    ~Derived() override { delete[] data_; } // Destructor invoked properly

    void speak() const override {
        // Implementation
    }
private:
    int* data_;
};

int main() {
    Base* ptr = new Derived();
    delete ptr; // Safely invokes ~Derived() followed by ~Base()
}
```

### Modern Keywords: `override` & `final`

- **`override`**: Directs compiler to verify that the method matches a base class virtual signature exactly (prevents subtle bugs caused by parameter type mismatches).
- **`final`**: Prevents further virtual overriding or class derivation:

```cpp
class FinalClass final : public Base { // Class cannot be inherited from
public:
    void speak() const override final; // Method cannot be overridden further
};
```

---

## 3. Multiple Inheritance & The Diamond Problem

Multiple inheritance allows a class to derive from multiple base classes. However, inheriting from two base classes that share a common grandparent creates the **Diamond Inheritance Problem**:

```
           [ GrandParent ]
             /         \
            /           \
       [ ParentA ]   [ ParentB ]
            \           /
             \         /
            [ Derived ]
```

Without special handling, `Derived` receives two duplicate subobject copies of `GrandParent`, creating symbol reference ambiguities and duplicate memory allocation.

### Resolution via Virtual Base Classes (`virtual public`)

Declaring base classes using `virtual public` ensures only a single shared instance of `GrandParent` exists inside `Derived`:

```cpp
#include <iostream>

struct GrandParent {
    int id = 42;
};

// Virtual base class inheritance
struct ParentA : virtual public GrandParent {};
struct ParentB : virtual public GrandParent {};

struct Derived : public ParentA, public ParentB {};

int main() {
    Derived d;
    std::cout << "GrandParent ID: " << d.id << "\n"; // Unambiguous single GrandParent instance!
}
```
