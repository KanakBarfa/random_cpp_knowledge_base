# C++ Core Technical Reference & Architecture Knowledge Base

Welcome to the C++ Technical Reference Knowledge Base. This repository provides an authoritative, structured, and deep-dive engineering reference covering core C++ concepts, memory layouts, Object-Oriented Programming (OOP) mechanics, template metaprogramming, and modern language standard evolutions (C++14 through C++26).

---

## System Navigation Index

### 1. C++ Language Standards Evolution
*   [C++14 Overview & Features](standards/cpp14.md) — Generic lambdas, return type deduction, binary literals, digit separators, `std::make_unique`, and `[[deprecated]]`.
*   [C++17 Overview & Features](standards/cpp17.md) — Structured bindings, `if`/`switch` initializers, `std::filesystem`, vocabulary types (`std::optional`, `std::variant`, `std::any`), fold expressions, and parallel algorithms.
*   [C++20 Major Language Features](standards/cpp20.md) — Concepts & Constraints, Ranges & Views, Coroutines, C++ Modules, Spaceship operator (`<=>`), `std::format`, and `constexpr` vs `consteval` vs `constinit`.
*   [C++23 Standards & Improvements](standards/cpp23.md) — `std::print`/`std::println`, `std::expected`, monadic `std::optional`, Deducing `this`, multidimensional subscripting, and standard library modules (`import std;`).
*   [C++26 Standards & Language Enhancements](standards/cpp26.md) — Static Reflection (`std::meta`, `^`, `[: :]`), Pack Indexing (`T...[I]`), Linear Algebra (`<linalg>`), Lock-Free Memory Reclamation (`<hazard_pointer>`, `<rcu>`), Debugging Facilities (`<debugging>`), deleted functions with reason (`= delete("message")`), and wildcard placeholders (`_`).

### 2. Memory Architecture, Layout & Allocation
*   [Memory Allocators & High-Performance Engines](memory/allocators.md) — `new`/`delete` vs `malloc`/`free`, custom operator overloading, `std::allocator`, `std::pmr`, enterprise thread-caching allocators (TCMalloc, jemalloc), and C++26 lock-free memory reclamation (`<hazard_pointer>`, `<rcu>`).
*   [Smart Pointer Mechanics & Memory Footprint](memory/smart_pointers.md) — Sizing and internal architecture of `std::unique_ptr`, `std::shared_ptr`, and `std::weak_ptr`, control block structures, custom deleters, `make_shared` optimizations, and ownership transfer patterns.
*   [Class/Struct Memory Layout & Sizing](memory/layout_and_size.md) — Member alignment, struct padding, `#pragma pack`, `alignas`/`alignof`, empty class sizing (1-byte rule), Empty Base Optimization (EBO), C++20 `[[no_unique_address]]`, and `vptr`/`vtable` virtual dispatch overhead.

### 3. Object-Oriented Programming (OOP)
*   [Classes & Structs Mechanics](oop/classes_and_structs.md) — Technical differences (`struct` vs `class`), access specifiers, default inheritance rules, and member initializer list performance vs in-body assignment.
*   [Constructor Mechanics & Life Cycle](oop/constructors.md) — Default, Parameterized, Copy (Deep vs Shallow), Move (rvalue transfers), Delegating constructors, C++26 deleted functions with reason, Rule of Zero/Three/Five, and constructor exception safety unwinding.
*   [Inheritance Modes & Polymorphism](oop/inheritance.md) — Visibility specifiers (`public`, `protected`, `private`) in inheritance, virtual functions, abstract interfaces, `override`/`final`, and virtual base class layout in multiple inheritance.
*   [Const Correctness & Mutability](oop/const_correctness.md) — Const member functions, read-only guarantees, logical vs physical constness, the `mutable` keyword, and const overloading.

### 4. Templates & Metaprogramming
*   [Value Categories & Move Semantics](templates_and_metaprogramming/value_categories.md) — Formal taxonomy (lvalue, prvalue, xvalue, glvalue, rvalue), addressability rules, rvalue references (`T&&`), `std::move` cast mechanics, and perfect forwarding with `std::forward`.
*   [C++ Templates & Compile-Time Instantiation](templates_and_metaprogramming/templates.md) — Function, class, variable, and alias templates, template instantiation phases, full and partial template specialization, SFINAE (`std::enable_if`), C++20 Concepts, and C++26 Static Reflection (`std::meta`, `^`, `[: :]`) with Pack Indexing.
*   [Unions & Type-Safe Alternatives](templates_and_metaprogramming/unions.md) — Memory overlapping rules, `sizeof` compile-time evaluation, alignment padding, active member restrictions, and modern type-safe tagged unions (`std::variant` & `std::visit`).

### 5. Error & Exception Handling
*   [Custom Exceptions & Exception Safety](exceptions/custom_exceptions.md) — Standard exception hierarchy, designing custom exception classes, exception safety guarantees (Nothrow, Strong, Basic), RAII unwinding, and `noexcept` specifier performance implications.

### 6. Standard Containers & Specialized Data Structures
*   [std::bitset & Dynamic Bitset Architecture](containers/bitset.md) — Non-type template parameters, memory layout, stack vs heap allocation, complete API operational complexity, bitwise SIMD/SWAR optimizations, compiler intrinsics (`_Find_first`/`_Find_next`), and `std::vector<bool>` proxy reference pitfalls vs custom dynamic bitsets.

### 7. Language Fundamentals & Runtimes
*   [C Runtime Library (CRT) & System Execution](language_fundamentals/crt.md) — CRT architecture, startup wrapping (`main` execution environment), static object initialization/destruction order, CRT allocation vs C++ operators, and C-string / math CRT vs C++ standard library wrappers.
*   [Core Language Keywords & Type Inspection](language_fundamentals/keywords.md) — Comprehensive technical breakdown of `inline`, `constexpr`, `consteval`, `constinit`, `explicit`, `mutable`, `noexcept`, `override`, `final`, `nullptr`, `decltype`, `static`, `extern`, `thread_local`, standard casting operators, and C++26 additions (`_` placeholder, `= delete("reason")`, `^` reflection).

### 8. Tooling, Debugging & Profiling Architecture
*   [GDB, Valgrind & Callgrind Engineering Guide](tooling/debugging_and_profiling.md) — Interactive execution control, hardware watchpoints, post-mortem core dump analysis, multi-threaded debugging, Valgrind Memcheck shadow memory & leak taxonomy, Callgrind instruction counting & cache simulation, KCachegrind analysis, and diagnostic tool decision matrices.
