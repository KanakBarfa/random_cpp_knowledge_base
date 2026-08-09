# Smart Pointer Architecture, Sizing & Ownership Semantics

Category: **Memory**

---

## 1. Smart Pointer Taxonomy & Ownership Models

C++ standard smart pointers manage dynamic object lifecycles using Resource Acquisition Is Initialization (RAII):

- **`std::unique_ptr<T>`**: Represents **exclusive ownership**. Exactly one `unique_ptr` owns the underlying resource. Copying is disabled; ownership transfers strictly via move semantics.
- **`std::shared_ptr<T>`**: Represents **shared ownership**. Multiple `shared_ptr` instances reference the same object. The object is destroyed when the last owner releases reference.
- **`std::weak_ptr<T>`**: Represents **non-owning observation**. References an object managed by `shared_ptr` without incrementing its strong reference count. Prevents cyclic reference memory leaks.

---

## 2. Memory Layout & Sizing Deep Dive

### `std::unique_ptr<T, Deleter>` Memory Sizing

```cpp
#include <memory>
#include <iostream>

struct StatelessDeleter {
    void operator()(int* p) const { delete p; }
};

struct StatefulDeleter {
    int log_id;
    void operator()(int* p) const { delete p; }
};

int main() {
    // 1. Default unique_ptr (Stateless std::default_delete): 8 bytes
    std::cout << sizeof(std::unique_ptr<int>) << " bytes\n"; 

    // 2. Custom Stateless Deleter (Optimized via Empty Base Optimization): 8 bytes
    std::cout << sizeof(std::unique_ptr<int, StatelessDeleter>) << " bytes\n";

    // 3. Custom Stateful Deleter: Stores deleter instance (8 bytes pointer + 4/8 bytes state): 16 bytes
    std::cout << sizeof(std::unique_ptr<int, StatefulDeleter>) << " bytes\n";

    // 4. Function Pointer Deleter: Stores raw function pointer: 16 bytes
    std::cout << sizeof(std::unique_ptr<int, void(*)(int*)>) << " bytes\n";
}
```

### `std::shared_ptr<T>` Memory Layout & Control Block

On 64-bit architectures, `std::shared_ptr<T>` occupies exactly **16 bytes** (2 raw pointers):
1. **Raw Pointer to Managed Object** (`T* ptr`).
2. **Raw Pointer to Control Block** (`ControlBlock* cb`).

```
std::shared_ptr<T> (16 Bytes)
+-----------------------+-----------------------+
|  Ptr to Managed Object|  Ptr to Control Block |
|        (8 Bytes)      |        (8 Bytes)      |
+-----------------------+-----------------------+
            |                       |
            v                       v
     +--------------+    +---------------------------------+
     |  Object (T)  |    |         CONTROL BLOCK           |
     +--------------+    | - Strong Reference Count (atomic)|
                         | - Weak Reference Count   (atomic)|
                         | - Custom Deleter                |
                         | - Custom Allocator              |
                         | - Virtual Destructor / VTable   |
                         +---------------------------------+
```

### Control Block Memory Allocation: `make_shared` vs Explicit `new`

```cpp
// Method 1: Explicit raw new constructor
std::shared_ptr<Widget> sp1(new Widget());

// Method 2: std::make_shared factory function
std::shared_ptr<Widget> sp2 = std::make_shared<Widget>();
```

```
Method 1 (Explicit new): 2 SEPARATE ALLOCATIONS (Non-contiguous memory)
[Heap Allocation 1: Widget Object] <---- ptr
[Heap Allocation 2: Control Block] <---- cb

Method 2 (std::make_shared): 1 SINGLE ALLOCATION (Contiguous block)
+------------------------------------------------------+
| Control Block Header  |  Widget Object Data          |
+------------------------------------------------------+
^                       ^
cb                      ptr
```

#### Performance Comparison Matrix

| Aspect | `std::shared_ptr<T>(new T)` | `std::make_shared<T>()` |
| :--- | :--- | :--- |
| **Heap Allocations** | **2 distinct allocations** (Object + Control Block) | **1 single contiguous allocation** |
| **Cache Locality** | Poor (Object and Control block at random heap addresses) | Excellent (Control block and Object adjacent in RAM) |
| **Exception Safety** | Risk of leak if intermediate function arguments throw | 100% exception-safe |
| **Memory Deallocation** | Object memory freed immediately when strong count = 0 | Object memory delayed until **both** strong and weak counts = 0 |

---

## 3. `std::weak_ptr` & Breaking Reference Cycles

A cyclic reference occurs when objects hold `std::shared_ptr` instances pointing to each other, preventing reference counts from reaching zero:

```
[Node A (ref_count = 1)] ------------ shared_ptr -----------> [Node B (ref_count = 1)]
       ^                                                                |
       +------------------------- shared_ptr ---------------------------+
                             (MEMORY LEAK FOR EVER!)
```

### Resolving Cycles with `std::weak_ptr`
By converting one pointer in the cycle to `std::weak_ptr`, ownership becomes asymmetric:

```cpp
#include <iostream>
#include <memory>

struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev; // Break circular reference with weak_ptr!

    ~Node() { std::cout << "Node destroyed\n"; }
};

int main() {
    auto nodeA = std::make_shared<Node>();
    auto nodeB = std::make_shared<Node>();

    nodeA->next = nodeB;
    nodeB->prev = nodeA; // Non-owning observation; ref count of nodeA remains 1!
}
// Both nodeA and nodeB destruct successfully on scope exit!
```

### Safe Access via `.lock()`
`std::weak_ptr` cannot be dereferenced directly. Call `.lock()` to attempt promoting it to a `std::shared_ptr`:

```cpp
void inspect_observer(const std::weak_ptr<Node>& weak_node) {
    if (std::shared_ptr<Node> shared_node = weak_node.lock()) {
        // Object is active; safely use shared_node
        std::cout << "Node is alive\n";
    } else {
        // Object has been destroyed
        std::cout << "Node expired\n";
    }
}
```

---

## 4. Ownership Transfer & Idiomatic API Function Parameters

### Function Parameter Passing Guidelines

1. **Take Object by Reference/Pointer (`T&` / `T*`)**:
   Use when function only observes or operates on the resource without affecting ownership.
   ```cpp
   void render_mesh(const Mesh& mesh); // Preferred idiom
   ```

2. **Take `std::unique_ptr<T>` by Value**:
   Use when function explicitly **sinks (claims) exclusive ownership**.
   ```cpp
   void process_task(std::unique_ptr<Task> task) {
       // Function owns task and destroys it on scope exit
   }
   
   auto t = std::make_unique<Task>();
   process_task(std::move(t)); // Explicit move required!
   ```

3. **Take `std::shared_ptr<T>` by Value**:
   Use **only** when function intends to retain a shared owner copy (e.g., storing it in a class member or container).
   ```cpp
   class Session {
   public:
       Session(std::shared_ptr<User> user) : user_(std::move(user)) {}
   private:
       std::shared_ptr<User> user_;
   };
   ```

### Enabling `shared_from_this`
When an object managed by `shared_ptr` needs to pass a `shared_ptr` of itself to an API, inherit from `std::enable_shared_from_this<T>`:

```cpp
#include <memory>
#include <iostream>

class Worker : public std::enable_shared_from_this<Worker> {
public:
    void register_self() {
        // Creates a valid shared_ptr sharing ownership with existing control block
        std::shared_ptr<Worker> self_ptr = shared_from_this();
    }
};
```
