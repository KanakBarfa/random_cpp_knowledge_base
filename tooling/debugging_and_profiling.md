# C++ Debugging & Profiling Architecture: GDB, Valgrind & Callgrind

Category: **Tooling & Diagnostics**

---

## 1. Compilation & Symbol Architecture

Proper debugging and profiling require preparing binary artifacts with appropriate compiler metadata and frame information.

```
+-------------------------------------------------------------------------+
|                  Compilation Flags for Diagnostic Workflows             |
+-------------------------------------------------------------------------+
|  Debug Mode:     g++ -Og -g3 -fno-omit-frame-pointer main.cpp -o app   |
|  Profile Mode:   g++ -O2 -g  -fno-omit-frame-pointer main.cpp -o app   |
+-------------------------------------------------------------------------+
```

### Compiler Flag Breakdown

| Flag | Purpose | Impact on Debugging & Profiling |
| :--- | :--- | :--- |
| `-g` / `-g3` | Emits DWARF debug symbols. `-g3` additionally includes macro definitions (`#define`), allowing macro expansion evaluation in GDB. | Essential for mapping machine addresses to source lines and variable names. |
| `-O0` | Disables optimizations completely. | Variables map 1:1 to memory/stack slots; lifetimes are precise, but code runs significantly slower. |
| `-Og` | Optimization level specifically designed for the edit-compile-debug cycle. | Enables optimizations that do not degrade debugging experience (preserves call stacks and variable locations). |
| `-O2 -g` | High optimization with debug symbols. | Required for realistic profiling (e.g., with Callgrind or `perf`). Inlined functions and vectorized loops may complicate line-by-line stepping. |
| `-fno-omit-frame-pointer` | Instructs compiler to preserve the frame pointer register (`%rbp` on x86-64) instead of reusing it as a general-purpose register. | **Critical for accurate stack unwinding** in GDB backtraces, profilers, and sanitizers. Minimal performance penalty (~1%). |

### Detached Debug Symbols
For production environments where binary size must be minimized but post-mortem debugging is required:

```bash
# 1. Compile with full debug symbols
g++ -O2 -g -fno-omit-frame-pointer -o my_app main.cpp

# 2. Extract debug information to a separate symbol file
objcopy --only-keep-debug my_app my_app.debug

# 3. Strip debug symbols from the production binary
strip --strip-debug --strip-unneeded my_app

# 4. Link binary to debug symbols
objcopy --add-gnu-debuglink=my_app.debug my_app

# In GDB: symbols are automatically loaded, or manually via:
# (gdb) symbol-file my_app.debug
```

---

## 2. GDB (GNU Debugger) Deep Dive

GDB interacts directly with the Linux `ptrace` system call to control target processes, inspect registers, modify memory, and intercept signals.

```
                  GDB Process Controller
                            │
               ptrace(PTRACE_ATTACH / TRACEME)
                            │
                            ▼
              Target C++ Executable Process
       +─────────────────────────────────────────+
       | Registers: $rip, $rsp, $rbp, $rax...     |
       | Memory: Stack frames, Heap, Text (.text)|
       | Breakpoints: Injected int3 (0xCC opcode)|
       | Hardware Watchpoints: Debug Registers   |
       +─────────────────────────────────────────+
```

### 1. Launching & Attaching
```bash
# Start an executable with arguments
gdb --args ./my_app --port 8080 --threads 4

# Attach to an already running process by PID
gdb -p 12345

# Inspect a post-mortem crash core dump
gdb ./my_app core.dump
```

### 2. Breakpoints, Watchpoints & Catchpoints

#### Breakpoint Controls
```gdb
(gdb) break main                       # Break at function entry
(gdb) break src/parser.cpp:142         # Break at file and line
(gdb) break MyClass::processData       # Break at C++ class member function
(gdb) rbreak ^parse_.*                 # Break on all functions matching regex

# Conditional Breakpoints (only stop when condition evaluates to true)
(gdb) break parser.cpp:150 if count > 1000 && ptr != 0

# Temporary Breakpoints (automatically deletes itself after hit once)
(gdb) tbreak init_subsystem

# Managing Breakpoints
(gdb) info breakpoints                # List all breakpoints with IDs and hit counts
(gdb) disable 2                       # Temporarily disable breakpoint 2
(gdb) enable 2                        # Re-enable breakpoint 2
(gdb) delete 2                        # Permanently delete breakpoint 2
(gdb) ignore 1 500                    # Ignore breakpoint 1 for the next 500 hits
```

#### Hardware Watchpoints (Memory Modification Traps)
Hardware watchpoints leverage CPU debug registers (`DR0`–`DR3` on x86) to halt execution immediately when a memory address changes without slowing down execution:
```gdb
(gdb) watch variable_name             # Breaks when variable is WRITTEN to
(gdb) rwatch variable_name            # Breaks when variable is READ from
(gdb) awatch variable_name            # Breaks on any READ or WRITE access
(gdb) watch *(int*)0x7fffffffe120     # Watch arbitrary memory address
```

#### Catchpoints (C++ Exceptions & System Calls)
```gdb
(gdb) catch throw                     # Intercept every C++ exception when thrown
(gdb) catch catch                     # Intercept where an exception is caught
(gdb) catch syscall write             # Intercept specific Linux syscall
(gdb) catch signal SIGSEGV            # Intercept segmentation faults
```

### 3. Execution Control & Navigation
```gdb
(gdb) run (or r)                      # Start program execution from beginning
(gdb) continue (or c)                 # Continue execution until next breakpoint/signal
(gdb) next (or n)                     # Step OVER next line (does not enter functions)
(gdb) step (or s)                     # Step INTO function on current line
(gdb) finish                          # Execute until current function finishes & returns
(gdb) until 45                        # Execute until line 45 is reached (escapes loops)
(gdb) advance process_packet          # Run forward until target function is entered
```

### 4. Stack Frame Inspection & Post-Mortem Debugging
When an application crashes (or hits a breakpoint), inspect the call stack:

```gdb
(gdb) bt                              # Print standard backtrace (callstack)
(gdb) bt full                         # Print backtrace with local variables in all frames
(gdb) frame 2 (or f 2)                # Switch context to stack frame 2
(gdb) up                              # Move up one caller frame in the stack
(gdb) down                            # Move down one callee frame in the stack
(gdb) info frame                      # Detailed frame metadata (registers, stack addresses)
(gdb) info args                       # Function arguments passed to current frame
(gdb) info locals                     # All local variables declared in current frame
```

### 5. Data & Memory Inspection

#### Printing Variables & Standard Library Containers
```gdb
(gdb) print var                       # Print value of variable
(gdb) print/x var                     # Print in Hexadecimal
(gdb) print/t var                     # Print in Binary
(gdb) print *ptr                      # Dereference pointer
(gdb) print ptr->member               # Access struct/class field

# Arrays and Dynamic Slices
(gdb) print *array@10                 # Print 10 contiguous elements from array pointer

# C++ Standard Library Pretty-Printing
(gdb) set print pretty on             # Format nested structs and containers cleanly
(gdb) print my_vector                 # Prints std::vector elements via Python pretty-printer
(gdb) print my_map                    # Prints key-value pairs of std::map
```

#### Low-Level Memory Examination (`x` command)
Format syntax: `x/[count][format][size] <address>`
* Formats: `x` (hex), `d` (decimal), `u` (unsigned), `s` (string), `i` (instruction)
* Units: `b` (byte), `h` (halfword/2 bytes), `w` (word/4 bytes), `g` (giant/8 bytes)

```gdb
(gdb) x/16xb $rsp                     # Inspect 16 bytes in hex at stack pointer
(gdb) x/4xg &my_struct                # Inspect 4 64-bit words at struct address
(gdb) x/5i $rip                       # Disassemble next 5 machine instructions at instruction pointer
(gdb) x/s string_ptr                  # Examine memory as a null-terminated C string
```

### 6. Assembly & Text User Interface (TUI) Mode
```gdb
(gdb) disassemble /m MyClass::compute # Interleave source code lines with assembly
(gdb) info registers                  # Display all CPU general-purpose registers
(gdb) print $rax                      # Display register value
(gdb) layout src                      # Switch to split TUI mode showing source code
(gdb) layout asm                      # Switch to split TUI mode showing assembly instructions
(gdb) layout split                    # Display both source code and assembly side-by-side
(gdb) tui disable                     # Exit TUI mode back to standard CLI
```

### 7. Multi-Threaded Debugging
```gdb
(gdb) info threads                    # List all running threads with thread IDs and LWP PIDs
(gdb) thread 3                        # Switch context to thread 3
(gdb) thread apply all bt             # Execute backtrace across EVERY thread simultaneously
(gdb) set scheduler-locking on        # Freeze all other threads while stepping current thread
(gdb) set scheduler-locking off       # Allow all threads to resume normally
```

### 8. Reverse Debugging (Process Execution Recording)
GDB can record execution history and step backwards in time:
```gdb
(gdb) record                          # Begin recording machine state transitions
(gdb) reverse-step (or rs)            # Step backward into previous line
(gdb) reverse-next (or rn)            # Step backward over previous line
(gdb) reverse-continue (or rc)        # Run backward until previous breakpoint or watchpoint
(gdb) record stop                     # Terminate recording session
```

---

## 3. Valgrind Memcheck: Memory Error & Leak Analysis

Valgrind is a dynamic binary instrumentation framework. The default and most widely used tool is **Memcheck**, which tracks memory allocation and validity at the byte/bit level via **Shadow Memory**.

```
+─────────────────────────────────────────────────────────────────────────+
|                 Valgrind Memcheck Shadow Memory Architecture            |
+─────────────────────────────────────────────────────────────────────────+
|  Real Memory Byte:        [ 0x42 ] (8 bits of application data)         |
|                                                                         |
|  Shadow Memory Tracking:                                                |
|  1. Addressability Bits (A-Bits): Is this memory address valid to access|
|     (Allocated via malloc/new, or active stack frame?)                  |
|  2. Validity Bits (V-Bits): Are the data bits initialized?              |
|     (Have valid values been written to each bit?)                       |
+─────────────────────────────────────────────────────────────────────────+
```

### The Standard Memcheck Command
Run Memcheck with full leak resolution and origin tracking enabled:

```bash
valgrind \
  --tool=memcheck \
  --leak-check=full \
  --show-leak-kinds=all \
  --track-origins=yes \
  --verbose \
  --log-file=valgrind_report.log \
  ./my_app arg1 arg2
```

### Key Error Diagnostics & Root Causes

#### 1. Invalid Read / Write (Buffer Overflows & Use-After-Free)
Occurs when an address is not allocated or falls outside allocated bounds (A-bit check fails):

```
==12345== Invalid read of size 4
==12345==    at 0x401142: process(int*) (main.cpp:12)
==12345==    by 0x4011AB: main (main.cpp:25)
==12345==  Address 0x5204068 is 0 bytes after a block of size 40 alloc'd
==12345==    at 0x4C31B25: malloc (vg_replace_malloc.c:309)
==12345==    by 0x40118A: main (main.cpp:22)
```
* **Root Cause**: Heap or stack buffer overflow, or dereferencing freed memory.
* **Fix**: Validate buffer sizes; replace raw allocations with `std::vector` or `std::array`.

#### 2. Conditional Jump or Move Depends on Uninitialised Value
Occurs when uninitialized memory is read and used in control flow branching:

```
==12345== Conditional jump or move depends on uninitialised value(s)
==12345==    at 0x401234: evaluate(bool) (logic.cpp:45)
==12345==  Uninitialised value was created by a stack allocation
==12345==    at 0x401210: compute() (logic.cpp:30)
```
* **Note**: `--track-origins=yes` is crucial here: it points directly to the exact stack line or heap allocation where the uninitialized memory was created.

#### 3. Mismatched Free / Delete
Occurs when the allocation and deallocation APIs do not match:
* `malloc()` freed with `delete` or `delete[]`
* `new` freed with `free()` or `delete[]`
* `new[]` freed with `delete` (causes memory corruption or missed destructor calls)

#### 4. Memory Leak Taxonomy

```
+─────────────────────────────────────────────────────────────────────────+
|                  Valgrind Memory Leak Categorization                    |
+─────────────────────────────────────────────────────────────────────────+
| [Definitely Lost]  No pointers point to the block. Total leak.          |
| [Indirectly Lost]  Referenced only by blocks that are Definitely Lost.  |
| [Possibly Lost]    Pointers point to the interior of the block.         |
| [Still Reachable]  Pointers still exist at program termination.         |
+─────────────────────────────────────────────────────────────────────────+
```

| Leak Category | Technical Definition | Recommended Action |
| :--- | :--- | :--- |
| **Definitely Lost** | No pointer to the start of the block exists anywhere in memory. | **Critical Bug**. Must be resolved; memory cannot be recovered without a restart. |
| **Indirectly Lost** | The block is referenced only by another block that is itself definitely lost (e.g., node of a lost linked list). | Fix the root pointer causing the "Definitely Lost" parent leak; the child leak resolves automatically. |
| **Possibly Lost** | Pointers point to interior bytes of the allocated block, but not the start header. | Often caused by custom allocators or pointer tricks; inspect to ensure it is not an accidental leak. |
| **Still Reachable** | A valid pointer still points to the block at program termination. | Common for static singletons, caches, and global resources freed on OS process exit. Generally non-fatal, but good practice to clean up via RAII. |

### Valgrind Suppression Files
To ignore expected false positives or third-party library allocations:
```bash
# Generate suppression entries automatically
valgrind --leak-check=full --gen-suppressions=all ./my_app

# Run using a custom suppression file
valgrind --suppressions=my_rules.supp ./my_app
```

---

## 4. Callgrind & KCachegrind: Deterministic Profiling

**Callgrind** is a Valgrind profiling skin that collects call-graph execution data and simulates CPU cache behavior.

### Why Callgrind?
* **Deterministic Simulation**: Unlike statistical sampling profilers (which interrupt the CPU every $N$ milliseconds), Callgrind instruments every basic block. Results are **100% reproducible**.
* **Zero Sampling Bias**: Functions that execute quickly but frequently are tracked with exact instruction-level precision.
* **Full Call Graph**: Maps exactly which caller invoked which callee, and how many times.

```
                  Callgrind Runtime Pipeline
                              │
               Translates Machine Instructions
                              │
                              ▼
    +─────────────────────────────────────────────────────+
    | 1. Counts Instruction Execution (Ir)                |
    | 2. Records Function Calls & Returns (Call Graph)    |
    | 3. Simulates L1 / LL Instruction & Data Caches      |
    +─────────────────────────────────────────────────────+
                              │
                              ▼
                 Writes 'callgrind.out.<pid>'
                              │
               +──────────────┴──────────────+
               │                             │
               ▼                             ▼
     callgrind_annotate (CLI)        KCachegrind / QCachegrind (GUI)
```

### 1. Generating Profiles with Callgrind

Compile with `-O2 -g -fno-omit-frame-pointer`:

```bash
# Full execution profile with simulated cache behavior
valgrind \
  --tool=callgrind \
  --dump-instr=yes \
  --collect-jumps=yes \
  --simulate-cache=yes \
  ./my_app
```

This generates a file named `callgrind.out.<pid>` in the current directory.

### 2. Selective Profiling via Client Requests
Profiling an entire application startup and shutdown distorts metrics. Use selective profiling:

```bash
# Start Callgrind with instrumentation disabled
valgrind --tool=callgrind --instr-atstart=no ./my_app
```

In your C++ code, bracket the hot code with Callgrind client macros:

```cpp
#include <valgrind/callgrind.h>

void execute_benchmark() {
    // Enable Callgrind instrumentation for this section only
    CALLGRIND_START_INSTRUMENTATION;

    process_million_events();

    // Disable instrumentation
    CALLGRIND_STOP_INSTRUMENTATION;
    
    // Dump profile data immediately to disk
    CALLGRIND_DUMP_STATS;
}
```

### 3. Command-Line Analysis: `callgrind_annotate`
Analyze the profile directly in the terminal:

```bash
# Display top functions ordered by Instruction Read (Ir) count
callgrind_annotate --auto=yes callgrind.out.12345 | head -n 30
```

To annotate a specific source file line by line:
```bash
callgrind_annotate --auto=yes callgrind.out.12345 src/hot_algorithm.cpp
```

Sample output:
```
--------------------------------------------------------------------------------
Ir          Line  Source Code
--------------------------------------------------------------------------------
.             10  void multiply_matrix(double* A, double* B, double* C, int N) {
40,000         11      for (int i = 0; i < N; ++i) {
40,000,000     12          for (int j = 0; j < N; ++j) {
80,000,000     13              for (int k = 0; k < N; ++k) {
1,200,000,000  14                  C[i*N + j] += A[i*N + k] * B[k*N + j];
.              15              }
.              16          }
.              17      }
.              18  }
```

### 4. Graphical Visualization: KCachegrind / QCachegrind
Open the output file in KCachegrind:

```bash
kcachegrind callgrind.out.12345
```

#### Core Metrics in KCachegrind
* **Inclusive Cost (`Incl.`)**: Time/instructions spent within the function **plus all functions called by it**.
  * Use to understand high-level architectural cost and subsystem bottlenecks.
* **Exclusive Cost (`Self`)**: Time/instructions spent **strictly within the body of the function itself**, excluding any callees.
  * Use to identify hot leaf routines, tight loops, and algorithmic computation choke points.
* **Call Graph View**: Renders an interactive visual flowchart showing function call paths, call counts, and relative percentages of execution cost.
* **Cache Miss Metrics**:
  * `I1mr`: L1 Instruction Cache Misses
  * `D1mr`: L1 Data Cache Misses
  * `LLmr`: Last-Level (L3) Cache Misses (shows where memory stalls occur)

---

## 5. Companion Valgrind Diagnostic Tools

Beyond Memcheck and Callgrind, the Valgrind suite provides specialized tools for profiling memory footprints and detecting concurrency defects:

### 1. Massif: Heap Memory Profiler
Massif samples heap allocations over time to identify high-watermark memory consumption and memory leaks:

```bash
# Run Massif heap profiler
valgrind --tool=massif --pages-as-heap=yes ./my_app

# Visualize heap allocation graph in CLI
ms_print massif.out.12345
```

Outputs an ASCII graph showing peak memory usage over time with call stacks responsible for the highest allocations.

### 2. Helgrind: Thread Race & Deadlock Detector
Helgrind analyzes multi-threaded POSIX/C++ programs to detect concurrency bugs:

```bash
# Run Helgrind concurrency analyzer
valgrind --tool=helgrind ./my_app
```

Detects:
* Data races (concurrent access to memory without synchronization).
* Inconsistent lock acquisition orders (potential deadlocks).
* Destruction of active mutexes.

---

## 6. Diagnostic Tool Decision Matrix

Choosing the right tool depends on whether you are debugging correctness, testing memory safety, or optimizing performance:

| Tool | Primary Purpose | Runtime Overhead | Recompilation Required? | Best Used For |
| :--- | :--- | :--- | :--- | :--- |
| **GDB** | Interactive execution control, stepping, state inspection | Negligible (until paused) | Yes (`-g` for line/symbol info) | Crashing bugs, inspecting logic bugs, interactive debugging, core dumps. |
| **Valgrind Memcheck** | Memory leak detection, invalid reads/writes, uninitialized data | **10x – 30x slowdown** | No (works on raw machine code, but `-g` recommended) | Finding elusive memory corruption and leak chains without special compiler instrumentation. |
| **AddressSanitizer (ASan)** | Memory safety (OOB, UAF, leak detection) | **2x slowdown** | **Yes** (`-fsanitize=address`) | High-speed CI testing, fuzzing, and catching buffer overflows quickly. |
| **Callgrind + KCachegrind** | Deterministic call graph and cache profiling | **20x – 50x slowdown** | No (works on raw binaries, but `-g` recommended) | Exact instruction-count profiling, algorithm complexity analysis, call trees. |
| **Linux `perf`** | Hardware performance counters, CPU sampling | **$\le 1\% - 2\%$ slowdown** | No (`-fno-omit-frame-pointer` recommended) | **Production profiling**, hot CPU flamegraphs, hardware cache/branch misses. |

---

## 7. Unified Command Quick Reference

### GDB Quick Sheet
```gdb
# Run and Break
r [args]                       # Run program with arguments
b file.cpp:42                  # Set breakpoint
b Func if x > 10               # Set conditional breakpoint
watch my_var                   # Set hardware watchpoint on write
catch throw                    # Trap on any C++ exception throw

# Stepping and Stack
c                              # Continue execution
n                              # Next line (step over)
s                              # Step into function
fin                            # Finish current function and return
bt                             # Print call stack
bt full                        # Print call stack with local variables
f <n>                          # Switch to stack frame n
info locals                    # Print local variables

# Data and Registers
p var                          # Print variable value
p/x var                        # Print in hexadecimal
p *arr@size                    # Print array slice
x/16xb 0xaddr                  # Examine 16 bytes in hex at address
info registers                 # Print CPU registers
disas /m func                  # Disassemble function with source lines

# Threads
info threads                   # List all threads
thread <id>                    # Switch to specific thread
thread apply all bt            # Backtrace all threads
```

### Valgrind & Profiling Quick Sheet
```bash
# Memory Leak & Error Check
valgrind --leak-check=full --show-leak-kinds=all --track-origins=yes ./app

# Memory Profiling (Heap high watermark)
valgrind --tool=massif ./app && ms_print massif.out.*

# Call Graph & Instruction Profiling
valgrind --tool=callgrind --simulate-cache=yes ./app
callgrind_annotate --auto=yes callgrind.out.<pid>
kcachegrind callgrind.out.<pid> &

# Concurrency & Data Race Detection
valgrind --tool=helgrind ./app
```
