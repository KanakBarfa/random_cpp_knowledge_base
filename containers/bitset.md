# std::bitset Architecture & Dynamic Bitset Design

Category: **Containers**

---

## 1. Technical Foundations of `std::bitset<N>`

`std::bitset<N>` is a fixed-size sequence of $N$ bits stored compactly. Unlike dynamic containers, $N$ is a **non-type template parameter** that must be evaluated at compile time.

```cpp
#include <bitset>
#include <iostream>

// N must be a compile-time constant expression
constexpr size_t BIT_COUNT = 128;
std::bitset<BIT_COUNT> flags;
```

### Architectural & Design Rationale

1. **Stack Allocation & Zero Heap Overhead**:
   Because $N$ is fixed at compile time, `sizeof(std::bitset<N>)` is deterministic. The storage array is allocated entirely on the stack (or within static memory), avoiding dynamic heap memory allocation (`malloc`/`new`) and pointer indirection.

2. **Word-Level Data Packing**:
   Internally, `std::bitset` groups bits into machine words (typically `unsigned long` or `uint64_t` depending on platform architecture). 
   - A `std::bitset<64>` occupies 8 bytes (1 word).
   - A `std::bitset<65>` occupies 16 bytes (2 words due to alignment padding).

3. **Compiler Optimization & Instruction Unrolling**:
   With a constant size $N$, the compiler generates hardware-level single-instruction bitwise operations (e.g., 64-bit `AND`, `OR`, `XOR`, `POPCNT`), unrolling loops completely at build time.

---

## 2. API Reference & Operational Complexity

In the table below:
- $N$: Total number of bits (`Size`).
- $W$: Target platform word size in bits (typically 64 on modern x86_64 / ARM64 architectures).

| Category | Operation / Method | Description | Time Complexity | Exception Behavior |
| :--- | :--- | :--- | :--- | :--- |
| **Element Access** | `b[i]` | Access bit at index `i` (returns proxy reference `reference`) | $O(1)$ | No bounds check |
| | `b.test(i)` | Access bit at index `i` with bounds verification | $O(1)$ | Throws `std::out_of_range` if `i >= N` |
| **Bit Manipulation** | `b.set(i, val)` | Sets bit `i` to `val` (default `true`) | $O(1)$ | Throws `std::out_of_range` if `i >= N` |
| | `b.set()` | Sets **all** bits to 1 | $O(N/W)$ | `noexcept` |
| | `b.reset(i)` | Resets bit `i` to 0 | $O(1)$ | Throws `std::out_of_range` if `i >= N` |
| | `b.reset()` | Resets **all** bits to 0 | $O(N/W)$ | `noexcept` |
| | `b.flip(i)` | Toggles bit `i` | $O(1)$ | Throws `std::out_of_range` if `i >= N` |
| | `b.flip()` | Inverts **all** bits | $O(N/W)$ | `noexcept` |
| **Bitwise Operators** | `b1 & b2`, `b1 \| b2`, `b1 ^ b2` | Bitwise AND, OR, XOR operations | $O(N/W)$ | `noexcept` |
| | `~b` | Bitwise NOT (inversion) | $O(N/W)$ | `noexcept` |
| | `b << k`, `b >> k` | Shift bits left or right by `k` positions | $O(N/W)$ | `noexcept` |
| **Query Operations** | `b.count()` | Returns count of set bits (1s) via hardware `popcount` | $O(N/W)$ | `noexcept` |
| | `b.any()`, `b.none()`, `b.all()` | Check if any, no, or all bits are set | $O(N/W)$ | `noexcept` |
| | `b.size()` | Returns total bit capacity $N$ | $O(1)$ | `noexcept` |
| **Conversions** | `b.to_string()` | Constructs `std::string` representation | $O(N)$ | May throw `std::bad_alloc` |
| | `b.to_ulong()` | Converts to `unsigned long` | $O(N/W)$ | Throws `std::overflow_error` if $N > 32/64$ |
| | `b.to_ullong()` | Converts to `unsigned long long` | $O(N/W)$ | Throws `std::overflow_error` if $N > 64$ |

---

## 3. Performance & Compiler Intrinsic Optimizations

### Compiler Bit Search Extensions
While not part of the ISO C++ Standard, major compilers (GCC and Clang) provide optimized internal routines for scanning set bits without full loops:

```cpp
#include <bitset>
#include <iostream>

void iterate_set_bits(const std::bitset<1024>& b) {
    // Non-standard GCC/Clang intrinsic extensions
    for (size_t i = b._Find_first(); i < b.size(); i = b._Find_next(i)) {
        std::cout << "Set bit found at index: " << i << "\n";
    }
}
```

---

## 4. Dynamic Bitsets: `std::bitset` vs `std::vector<bool>` vs Custom Design

When the bit sequence size is only known at **runtime**, `std::bitset<N>` cannot be used.

### The Pitfalls of `std::vector<bool>`
Although `std::vector<bool>` dynamically sizes bit arrays, it is widely considered a flawed STL specialization:
1. **Non-Conforming Container**: It does not return true `bool&` references. Instead, it returns temporary proxy objects (`std::vector<bool>::reference`), breaking template code expecting `T&`.
2. **Thread-Safety Hazard**: Concurrent writes to different bits located within the *same* 64-bit word cause data races, as CPUs read/write full word chunks.

### High-Performance Production Dynamic Bitset Implementation
Below is a C++20 compliant, lock-free read, production-grade dynamic bitset implementation utilizing `std::vector<uint64_t>`:

```cpp
#include <iostream>
#include <vector>
#include <cstdint>
#include <stdexcept>
#include <algorithm>

class DynamicBitset {
public:
    explicit DynamicBitset(size_t bit_count) 
        : bit_count_(bit_count), 
          chunks_((bit_count + 63) / 64, 0) {}

    void set(size_t index) {
        if (index >= bit_count_) throw std::out_of_range("Index out of bounds");
        chunks_[index >> 6] |= (1ULL << (index & 63));
    }

    void reset(size_t index) {
        if (index >= bit_count_) throw std::out_of_range("Index out of bounds");
        chunks_[index >> 6] &= ~(1ULL << (index & 63));
    }

    [[nodiscard]] bool test(size_t index) const {
        if (index >= bit_count_) throw std::out_of_range("Index out of bounds");
        return (chunks_[index >> 6] & (1ULL << (index & 63))) != 0;
    }

    // Bitwise OR in O(bit_count / 64)
    DynamicBitset& operator|=(const DynamicBitset& rhs) {
        if (bit_count_ != rhs.bit_count_) {
            throw std::invalid_argument("Bitset dimensions must match");
        }
        for (size_t i = 0; i < chunks_.size(); ++i) {
            chunks_[i] |= rhs.chunks_[i];
        }
        return *this;
    }

    [[nodiscard]] size_t size() const noexcept { return bit_count_; }

    [[nodiscard]] size_t count() const noexcept {
        size_t total = 0;
        for (uint64_t chunk : chunks_) {
            total += std::popcount(chunk); // C++20 hardware bit count
        }
        return total;
    }

private:
    size_t bit_count_;
    std::vector<uint64_t> chunks_;
};

int main() {
    DynamicBitset db1(1000);
    DynamicBitset db2(1000);

    db1.set(42);
    db1.set(512);
    db2.set(512);

    db1 |= db2;

    std::cout << "Set bits count: " << db1.count() << "\n"; // Output: 2
    std::cout << "Bit 42 set: " << std::boolalpha << db1.test(42) << "\n";
}
```
