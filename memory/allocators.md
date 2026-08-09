# Memory Allocators & High-Performance Engines

Category: **Memory**

---

## 1. Fundamentals: `new`/`delete` vs `malloc`/`free`

C++ provides two memory allocation subsystems: the low-level C Runtime Memory API (`malloc`/`free`) and C++ object operators (`new`/`delete`).

```
+-------------------------------------------------------------------+
|               C++ Expression: MyClass* obj = new MyClass();       |
+-------------------------------------------------------------------+
                                  |
         +------------------------+------------------------+
         |                                                 |
         v                                                 v
[Step 1: Raw Allocation]                       [Step 2: Initialization]
Calls operator new(sizeof(MyClass))            Executes MyClass::MyClass()
Allocates raw heap bytes                       Initializes fields & invariants
         |                                                 |
         +------------------------+------------------------+
                                  |
                                  v
                Returns fully constructed MyClass* pointer
```

### Technical Comparison Matrix

| Feature | C++ `new` / `delete` | C `malloc()` / `free()` |
| :--- | :--- | :--- |
| **Entity Category** | C++ Language Operator | C Runtime Library Function |
| **Object Lifecycle** | **Calls constructors & destructors** automatically | Unaware of objects; allocates raw, uninitialized bytes |
| **Type Safety** | Type-safe; returns `T*` matching requested type | Non-type-safe; returns `void*` requiring manual cast |
| **Failure Mode** | Throws `std::bad_alloc` exception (or returns `nullptr` with `std::nothrow`) | Returns `NULL` pointer on allocation failure |
| **Customization** | Can be overloaded globally or per class | Cannot be overloaded |
| **Size Calculation** | Handled automatically by compiler (`sizeof(T)`) | Must be calculated manually (`num * sizeof(T)`) |

### Placement `new` & Explicit Destruction
When memory is pre-allocated (e.g., custom arena buffers or container capacity), placement `new` constructs an object in-place without requesting additional memory:

```cpp
#include <iostream>
#include <new>

struct Resource {
    int id;
    Resource(int i) : id(i) { std::cout << "Resource " << id << " constructed\n"; }
    ~Resource() { std::cout << "Resource " << id << " destroyed\n"; }
};

int main() {
    // 1. Allocate raw byte memory buffer
    alignas(Resource) char buffer[sizeof(Resource)];

    // 2. Placement new: Construct object in pre-allocated buffer
    Resource* res_ptr = new (buffer) Resource(42);

    // 3. MUST explicitly call destructor for placement new!
    res_ptr->~Resource();
    
    // Memory buffer stack memory cleaned up automatically on scope exit
}
```

---

## 2. Standard C++ Allocator Abstraction & Polymorphic Memory (`std::pmr`)

Standard library containers decoupling data structures from raw memory management via allocator types.

### `std::allocator_traits` Interface Layer
Since C++11, containers interact with memory through `std::allocator_traits<Alloc>` rather than invoking allocators directly:

```cpp
#include <memory>
#include <vector>

template<typename T>
class CustomAllocator {
public:
    using value_type = T;

    CustomAllocator() noexcept = default;

    T* allocate(size_t n) {
        if (auto p = static_cast<T*>(std::malloc(n * sizeof(T)))) {
            return p;
        }
        throw std::bad_alloc();
    }

    void deallocate(T* p, size_t n) noexcept {
        std::free(p);
    }
};
```

### Polymorphic Memory Resources (`std::pmr`) (C++17)
Standard containers using standard allocators embed the allocator type in their template signature (`std::vector<int, CustomAlloc>`), preventing type interop. C++17 introduced `std::pmr` to allow runtime memory resource substitution:

```cpp
#include <vector>
#include <array>
#include <memory_resource>
#include <iostream>

int main() {
    // 200 KB stack buffer allocation
    std::array<std::byte, 200000> stack_buffer;

    // Monotonic buffer resource: zero-heap allocation engine
    std::pmr::monotonic_buffer_resource mem_pool(
        stack_buffer.data(), stack_buffer.size()
    );

    // Vector using runtime memory resource
    std::pmr::vector<int> numbers(&mem_pool);

    for (int i = 0; i < 1000; ++i) {
        numbers.push_back(i); // Allocates directly from stack_buffer!
    }

    std::cout << "Vector size: " << numbers.size() << "\n";
}
```

---

## 3. High-Performance Enterprise Allocators (TCMalloc / jemalloc)

Standard system allocators (`glibc malloc`) rely on global heap locks, causing severe thread contention in highly concurrent applications. High-performance enterprise allocators like Google's **TCMalloc** (Thread-Caching Malloc) resolve lock bottlenecking using multi-tier memory caches.

```
+-----------------------------------------------------------------------+
|                              THREAD 1                                 |
|  Thread-Local Cache (Small allocations <= 256 KB) [LOCK-FREE ACCESS]  |
+-----------------------------------------------------------------------+
                                  | (Cache Miss / Batch Release)
                                  v
+-----------------------------------------------------------------------+
|                            CENTRAL CACHE                              |
|           Per-Size-Class Central Spans [MUTEX LOCKED BATCHES]        |
+-----------------------------------------------------------------------+
                                  | (Page Fetch)
                                  v
+-----------------------------------------------------------------------+
|                             PAGE HEAP                                 |
|          Manages 8 KB Page Allocations & Large Memory Blocks          |
+-----------------------------------------------------------------------+
```

### TCMalloc Tier Architecture

1. **Thread-Local Caches (L1)**:
   - Each thread maintains a thread-local, lock-free memory cache partitioned into size classes (e.g., 8 bytes, 16 bytes, 32 bytes... up to 256 KB).
   - Over 95% of application `malloc`/`free` calls are serviced instantly from this thread-local cache without locking or kernel context switches.

2. **Central Caches (L2)**:
   - Shared central spans servicing thread-local cache misses.
   - Locks are acquired only when thread caches request a batch of objects or return unused memory blocks.

3. **Page Heap (L3)**:
   - Manages physical memory pages (typically 8 KB pages).
   - Allocations greater than 256 KB bypass thread caches and are allocated directly from the Page Heap.

### Architectural Performance Benefits
- **Lock Contention Elimination**: Threads allocate memory concurrently without blocking each other.
- **Cache-Line Alignment**: Memory allocations align to CPU cache lines (64 bytes), eliminating false sharing across cores.
- **Fragmentation Reduction**: Fixed size-class binning prevents heap fragmentation over long uptime durations.
