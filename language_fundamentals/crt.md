# C Runtime Library (CRT) & System Execution Architecture

Category: **Language Fundamentals**

---

## 1. CRT Architecture vs C++ Standard Library

The **C Runtime Library (CRT)** is the underlying low-level system library that provides startup code, execution context, memory management, and POSIX/C standard library calls required by compiled executables.

```
+-------------------------------------------------------------+
|                     C++ Application Code                    |
+-------------------------------------------------------------+
|   C++ Standard Library (STL)   |   Direct C Runtime Calls   |
| (libstdc++ / libc++ / MSVC)   | (glibc / ucrt / musl CRT)   |
+-------------------------------+-----------------------------+
|                  C Runtime Library (CRT)                   |
|       [Startup, Memory, I/O Buffers, Process Control]       |
+-------------------------------------------------------------+
|                 Operating System Kernel API                 |
|             (POSIX syscalls / Win32 API calls)              |
+-------------------------------------------------------------+
```

### Key Differences Matrix

| Feature | C Runtime Library (CRT) | C++ Standard Library |
| :--- | :--- | :--- |
| **Core Header Included** | `<stdio.h>`, `<stdlib.h>`, `<string.h>` | `<iostream>`, `<vector>`, `<memory>`, `<string>` |
| **Memory Operators** | `malloc()`, `free()`, `realloc()`, `calloc()` | `operator new`, `operator delete`, `allocator<T>` |
| **Initialization** | OS entry point (`_start`), CRT initializers | Static constructor calls (`.init_array`) |
| **Object Awareness** | Unaware (raw bytes & POD handling) | Constructor/Destructor lifecycle aware |
| **Error Handling** | `errno` global integer, `NULL` return codes | Exceptions (`throw`/`catch`), `std::expected` |
| **Type Safety** | Low (`void*`, format specifier mismatches) | High (Templates, strong typing, type traits) |

---

## 2. CRT Startup, Process Lifecycle & Termination Flow

When an operating system executes a C++ binary, control does **not** jump directly to `main()`. Instead, the OS loader calls the CRT entry point:

```
[OS Kernel Execve] 
       │
       ▼
[CRT Entry Point (_start / mainCRTStartup)]
       │
       ▼
[CRT Environment Setup (argc, argv, envp, heap init)]
       │
       ▼
[Global & Static C++ Constructors (.init_array / .ctors)]
       │
       ▼
[User Code: main()]
       │
       ▼
[Global & Static C++ Destructors (.fini_array / atexit)]
       │
       ▼
[CRT Process Cleanup & Exit Syscall]
```

### Execution Termination Semantics

1. **`exit(int code)`**:
   - Flushes and closes all CRT open streams (`FILE*`).
   - Calls functions registered with `atexit()` in reverse order of registration.
   - Executes global and static object destructors.
   - Returns status code to the OS kernel.

2. **`quick_exit(int code)` (C11 / C++11)**:
   - Does **not** execute global/static destructors.
   - Does **not** flush standard CRT streams.
   - Calls functions registered via `at_quick_exit()`.

3. **`_Exit(int code)` / `_exit(int code)`**:
   - Immediately terminates process execution via OS kernel system call (`sys_exit`).
   - Bypasses all CRT cleanup, destructors, and stream flushing routines.

---

## 3. CRT Raw Memory Allocation vs C++ Object Construction

CRT memory routines allocate raw heap memory blocks without initializing object state:

```cpp
#include <cstdlib>
#include <iostream>

struct Sample {
    Sample() { std::cout << "Constructor invoked\n"; }
    ~Sample() { std::cout << "Destructor invoked\n"; }
};

int main() {
    // 1. CRT Allocation (No constructors called)
    Sample* s1 = static_cast<Sample*>(std::malloc(sizeof(Sample)));
    std::free(s1); // No destructors called!

    // 2. C++ Allocation (Constructors & destructors executed)
    Sample* s2 = new Sample();
    delete s2;
}
```

### Memory Function Operational Rules
- **`malloc(size_t size)`**: Allocates contiguous block of uninitialized bytes. Returns `nullptr` on failure.
- **`calloc(size_t num, size_t size)`**: Allocates memory for array of elements and zeroes all bits.
- **`realloc(void* ptr, size_t new_size)`**: Resizes existing allocation block, copying contents to new memory if reallocation requires relocation.
- **`free(void* ptr)`**: Deallocates block previously allocated by `malloc`/`calloc`/`realloc`. Passing `nullptr` is safe (no-op).

---

## 4. CRT Stream I/O vs C++ Iostreams & Synchronization

The C CRT uses file pointers (`FILE*`) and buffered stream handlers (`stdout`, `stdin`, `stderr`). C++ Iostreams (`std::cout`, `std::cin`) wrap CRT buffer streams by default to allow mixing C and C++ I/O.

### Synchronization Overhead (`sync_with_stdio`)
Because `std::cout` synchronizes its internal buffers with CRT `stdout` after every I/O operation, performance in I/O-heavy applications drops significantly. 

Disabling synchronization unbinds C++ streams from the CRT buffers for maximum throughput:

```cpp
#include <iostream>

int main() {
    // Disable synchronization between C++ iostreams and C CRT stdio streams
    std::ios_base::sync_with_stdio(false);
    
    // Untie cin from cout (cin won't automatically flush cout before reading)
    std::cin.tie(nullptr);

    std::cout << "High-performance un-buffered I/O\n";
}
```

---

## 5. Low-Level Memory & String Operations (`<cstring>`)

CRT provides fast block memory primitives executed via CPU vector instructions:

```cpp
#include <cstring>
#include <vector>

void low_level_copy(void* dest, const void* src, size_t bytes) {
    // Requires non-overlapping memory regions
    std::memcpy(dest, src, bytes);
}

void safe_overlapping_copy(void* dest, const void* src, size_t bytes) {
    // Safely handles overlapping source and destination blocks
    std::memmove(dest, src, bytes);
}
```

### Safety Rule
In modern C++, prefer type-safe wrappers like `std::copy`, `std::string_view`, and `std::span` over direct CRT `<cstring>` functions to prevent buffer overflow vulnerabilities.
