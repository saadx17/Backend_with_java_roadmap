# Stack Memory vs Heap Memory (Hardware Level)

> **Phase 0 — Computer Science Fundamentals → 0.1 How Computers Work**
> Goal: Understand what "stack" and "heap" actually *are* at the level of RAM, the CPU, and the OS — **before** we ever talk about how Java uses them.

---

## 0. The One-Sentence Summary

- **Stack** = a small, fast, *automatically managed* region of memory that grows and shrinks as functions are called and return. The CPU has a dedicated register (the **stack pointer**) for it.
- **Heap** = a large, flexible, *manually/runtime managed* region of memory used for data whose size or lifetime isn't known at compile time. Allocation is requested explicitly from the OS/runtime.

Both live in the same physical **RAM**. "Stack" and "heap" are *conventions and data-structure behaviors* imposed on regions of a process's memory — not separate pieces of hardware.

---

## 1. Where They Live: The Process Memory Layout

When the OS loads a program, it gives the process a **virtual address space**. That space is divided into segments:

```
  High Addresses
+---------------------------+
|         STACK             |  <- grows DOWNWARD (toward lower addresses)
|           |               |
|           v               |
|                           |
|   (unused gap / guard)    |
|                           |
|           ^               |
|           |               |
|         HEAP              |  <- grows UPWARD (toward higher addresses)
+---------------------------+
|   BSS (uninit. globals)   |  <- zero-initialized static data
+---------------------------+
|   DATA (init. globals)    |  <- initialized static/global variables
+---------------------------+
|   TEXT / CODE             |  <- the program's machine instructions (read-only)
+---------------------------+
  Low Addresses
```

**Key insight:** Stack and heap grow *toward each other*. The stack starts high and grows down; the heap starts low and grows up. The gap between them is free space they share. If they ever collide → **stack overflow** or **out-of-memory**.

| Segment | Contains | Lifetime |
|---------|----------|----------|
| **Text/Code** | Machine instructions | Whole program |
| **Data** | Initialized globals/statics | Whole program |
| **BSS** | Uninitialized globals/statics (zeroed) | Whole program |
| **Heap** | Dynamically allocated data | Until explicitly freed / GC'd |
| **Stack** | Function call frames, local variables | Until the function returns |

---

## 2. The Stack — Hardware Level

### 2.1 What it physically is
A contiguous block of RAM addresses managed as a **LIFO (Last-In, First-Out)** data structure. The CPU tracks the "top" of the stack using a dedicated register.

### 2.2 The CPU registers involved
- **Stack Pointer (SP)** — `RSP` on x86-64. Holds the address of the current top of the stack.
- **Base/Frame Pointer (BP)** — `RBP` on x86-64. Marks the base of the current function's stack frame, used to reference local variables and arguments reliably.

Because pushing/popping is just *adjusting a register* and writing to a nearby address, the stack is **extremely fast**. The top of the stack is almost always already in the **CPU cache** (great locality of reference).

### 2.3 Stack Frames (Activation Records)
Every time a function is called, a new **stack frame** is pushed. A frame typically contains:
- Function arguments
- The **return address** (where to resume after the function finishes)
- Saved previous frame pointer
- Local variables
- Temporary/scratch space

```
Call sequence: main() -> calculate() -> add()

   STACK (top = lowest address)
+------------------+  <- RSP (top, grows down)
|  add() frame     |
|   - locals       |
|   - return addr  |
+------------------+
|  calculate()     |
|   - locals       |
|   - return addr  |
+------------------+
|  main() frame    |
|   - locals       |
+------------------+
```

When `add()` returns, its frame is "popped" simply by moving the stack pointer back up. No cleanup work, no searching — instant.

### 2.4 Push / Pop at the instruction level
```asm
push rax      ; SP = SP - 8 ; store rax at [SP]
pop  rax      ; load [SP] into rax ; SP = SP + 8
```
A push is two micro-operations: decrement SP, write value. That's why allocation on the stack is essentially **free**.

### 2.5 Characteristics
| Property | Stack |
|----------|-------|
| Speed | Very fast (register-driven, cache-friendly) |
| Size | Small & fixed-ish (e.g., 1 MB default per thread on Linux, 512 KB–1 MB on JVM via `-Xss`) |
| Management | Automatic (compiler inserts push/pop) |
| Allocation order | Strict LIFO |
| Fragmentation | None (it's linear) |
| Thread scope | **Each thread has its OWN stack** |
| Failure mode | **Stack overflow** |

### 2.6 Stack Overflow (hardware reality)
The OS places a **guard page** (a non-accessible memory page) just past the stack's limit. If the stack grows into it (e.g., infinite recursion), the CPU triggers a **page fault**, the OS detects it, and the program crashes with a stack overflow error. This is a hardware/OS-level safety mechanism, not a software check.

---

## 3. The Heap — Hardware Level

### 3.1 What it physically is
A large pool of RAM addresses from which chunks of arbitrary size can be carved out **on demand at runtime**. Unlike the stack, there's no single "top" pointer — the runtime must *track* which regions are used and which are free.

### 3.2 How allocation actually works
A program doesn't talk to RAM chips directly. The chain looks like this:

```
Your code (e.g. new / malloc)
      |
      v
Memory allocator (e.g. JVM allocator, glibc malloc)
      |
      v
OS system calls (brk / sbrk to move the heap break, or mmap for big blocks)
      |
      v
Kernel virtual memory manager  ->  maps virtual pages to physical RAM frames
      |
      v
Physical RAM
```

- `brk`/`sbrk`: move the "program break" — the end of the heap segment — up or down.
- `mmap`: map a fresh region of memory (used for large allocations).

The allocator maintains internal bookkeeping (**free lists**, size classes, metadata headers) to know what's allocated and what's free.

### 3.3 Why it's slower than the stack
- Requires **searching** for a suitable free block (or splitting/coalescing blocks).
- May trigger a **system call** to the kernel when more memory is needed.
- Bookkeeping metadata overhead per allocation.
- Poorer **cache locality** — heap objects can be scattered across memory, causing more **cache misses**.

### 3.4 Fragmentation
Because chunks of different sizes are allocated and freed in arbitrary order, the heap suffers from:
- **External fragmentation** — free memory exists but is split into pieces too small to satisfy a request.
- **Internal fragmentation** — allocator rounds requests up to a size class, wasting the remainder.

The stack never has this problem because it's strictly linear.

### 3.5 Characteristics
| Property | Heap |
|----------|------|
| Speed | Slower (search + possible syscall + cache misses) |
| Size | Large (limited by RAM + swap / `-Xmx` in JVM) |
| Management | Manual (C/C++) or automatic via **Garbage Collector** (Java) |
| Allocation order | Arbitrary |
| Fragmentation | Yes (internal & external) |
| Thread scope | **Shared across all threads** of the process |
| Failure mode | **Out of memory** (OOM) |

---

## 4. Virtual Memory Connection

Both stack and heap addresses are **virtual addresses**. The CPU's **MMU (Memory Management Unit)** translates them to **physical RAM** addresses using **page tables**, in units called **pages** (typically 4 KB).

- Memory you "allocate" may not be backed by physical RAM until you actually touch it (**demand paging** / lazy allocation).
- Rarely used pages can be pushed to disk (**swap**), and faulted back in when accessed.

This is why two threads can each have a stack at "high addresses" without conflict — each sees its own virtual view. (Full detail covered later in the *Virtual memory and paging* topic.)

---

## 5. Stack vs Heap — Side-by-Side

| Aspect | **Stack** | **Heap** |
|--------|-----------|----------|
| Managed by | Compiler / CPU (SP register) | Allocator + (GC or programmer) |
| Speed | Very fast | Slower |
| Size | Small, limited | Large |
| Lifetime | Tied to function scope (auto) | Until freed / garbage collected |
| Access pattern | LIFO, contiguous | Random access |
| Cache behavior | Excellent locality | Often poor locality |
| Fragmentation | None | Yes |
| Thread sharing | Per-thread (private) | Shared |
| CPU support | Dedicated registers (RSP/RBP), push/pop | None — pure software bookkeeping |
| Typical failure | Stack overflow | Out of memory |
| Allocation cost | Adjust a register | Search free list, maybe syscall |

---

## 6. A Concrete Mental Model (Language-Agnostic)

```c
void example() {
    int x = 10;          // 'x' lives on the STACK (known size, known lifetime)
    int *p = malloc(40); // 'p' (the pointer) is on the STACK,
                         // but the 40 bytes it points to are on the HEAP
}                        // function returns: 'x' and 'p' vanish automatically,
                         // but the 40 heap bytes LEAK unless free(p) was called
```

This is the crucial distinction:
- **The pointer/reference** (a small fixed-size value) → typically on the **stack**.
- **The actual data it refers to** (variable size / shared / long-lived) → on the **heap**.

---

## 7. Why This Matters for a Backend (Java) Engineer

Even though this is "hardware level," it directly explains Java behavior you'll meet later:

- **Primitives & references** are stored on the **stack** (per thread); **objects** live on the **heap** (shared).
- **`StackOverflowError`** = deep/infinite recursion overran the thread's stack — exactly the guard-page mechanism above.
- **`OutOfMemoryError`** = the heap couldn't satisfy an allocation — the heap-exhaustion mechanism above.
- **Garbage Collection** exists *because* the heap requires lifetime management and suffers fragmentation.
- **Thread safety**: stacks are private (no sharing → naturally thread-safe locals); the heap is shared (→ needs synchronization).
- **Performance tuning** (`-Xss`, `-Xmx`, `-Xms`) directly sizes these regions.
- **Cache locality** explains why array-based structures often beat pointer-heavy ones in real workloads.

---

## 8. Quick Self-Check Questions

1. Why is stack allocation faster than heap allocation? *(Hint: register vs. search/syscall.)*
2. Which CPU register tracks the top of the stack?
3. Do the stack and heap live in different physical hardware? *(No — same RAM.)*
4. Why does each thread get its own stack but share the heap?
5. What hardware/OS mechanism actually detects a stack overflow?
6. What causes heap fragmentation, and why is the stack immune to it?
7. In `int *p = malloc(40);` — what's on the stack and what's on the heap?

---

## 9. Key Terms Glossary

- **Stack Pointer (SP/RSP):** register holding the current top-of-stack address.
- **Frame Pointer (BP/RBP):** register marking the base of the current stack frame.
- **Stack Frame / Activation Record:** per-call block holding args, locals, return address.
- **Return Address:** where execution resumes after a function returns.
- **Heap Allocator:** software (e.g., malloc, JVM) that hands out heap memory.
- **brk / sbrk / mmap:** syscalls that request memory from the kernel.
- **Free List:** allocator's record of available heap chunks.
- **Fragmentation:** wasted/unusable gaps in the heap.
- **Guard Page:** protected page used to detect stack overflow.
- **Page Fault:** CPU exception when accessing an unmapped/protected page.
- **Cache Locality:** how close in memory accessed data is (affects speed).

---

*Next topic in roadmap: **How programs are loaded into memory**.*
