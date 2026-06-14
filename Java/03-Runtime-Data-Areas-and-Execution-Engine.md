# Runtime Data Areas & Execution Engine (JIT)

> **Phase 1 — Java Language Mastery → 1.11 JVM Deep Dive**
> Goal: Master where the JVM stores everything at runtime (heap, stacks, metaspace, PC, native stack) and how the execution engine runs bytecode — interpreter, JIT compiler, and JIT optimizations.

---

## 0. The Big Picture

The **runtime data areas** are the JVM's memory regions; the **execution engine** runs the bytecode in them. Together they explain memory errors (`OutOfMemoryError`, `StackOverflowError`) and performance (interpreter warm-up, JIT speedup). This builds on **Phase 0.1** (stack vs heap, virtual memory).

```
RUNTIME DATA AREAS (memory)          EXECUTION ENGINE (runs bytecode)
  Heap | Stacks | Metaspace | PC      Interpreter -> JIT Compiler -> (GC)
```

---

## 1. The Runtime Data Areas

```
+------------------------- JVM Memory -------------------------+
|  SHARED across all threads:                                  |
|    +-------------------+   +----------------------------+    |
|    |       HEAP        |   |  Method Area (Metaspace)   |    |
|    | (all objects,     |   | (class metadata, statics,  |    |
|    |  GC-managed)      |   |  runtime constant pool)    |    |
|    +-------------------+   +----------------------------+    |
|                                                             |
|  PER-THREAD (one set per thread):                           |
|    +-----------+  +-------------+  +---------------------+   |
|    | JVM Stack |  | PC Register |  | Native Method Stack |   |
|    +-----------+  +-------------+  +---------------------+   |
+-------------------------------------------------------------+
```

| Area | Holds | Shared? | Error on exhaustion |
|------|-------|---------|---------------------|
| **Heap** | All objects (`new`), arrays | Shared | `OutOfMemoryError: Java heap space` |
| **Method Area / Metaspace** | Class metadata, static fields, constant pool | Shared | `OutOfMemoryError: Metaspace` |
| **JVM Stack** | Frames: local vars, operand stack, return addr | Per-thread | `StackOverflowError` |
| **PC Register** | Address of the current instruction | Per-thread | — |
| **Native Method Stack** | Native (JNI) call frames | Per-thread | `StackOverflowError` (native) |

---

## 2. The Heap (Objects Live Here)

The **heap** is the shared region where **all objects and arrays** are allocated (`new`) and where the **garbage collector** operates (next note). It's the biggest, most-tuned region.

### 2.1 Generational structure
HotSpot divides the heap into **generations** based on the **weak generational hypothesis** (most objects die young — next note):
```
HEAP
├── Young Generation
│    ├── Eden          (new objects allocated here)
│    ├── Survivor S0
│    └── Survivor S1
└── Old (Tenured) Generation   (long-lived objects promoted here)
```
- New objects → **Eden**. Survivors of GC → **Survivor** spaces → eventually **promoted** to **Old gen**.
- This split lets the GC collect the young gen frequently and cheaply (Minor GC) — most garbage is there (next note).

### 2.2 TLAB (Thread-Local Allocation Buffer)
Each thread gets a small private chunk of Eden — a **TLAB** — so it can allocate objects **without synchronization** (recall Phase 1.3 `new`, Phase 1.10 concurrency). Fast, lock-free allocation for the common case.

### 2.3 Sizing
`-Xms` (initial) / `-Xmx` (max) set the heap size (options note). When the heap can't satisfy an allocation even after GC → **`OutOfMemoryError: Java heap space`**.

---

## 3. Method Area / Metaspace (Class Metadata)

The **method area** stores **per-class** data: the class structure, **static fields**, method bytecode, and the **runtime constant pool**.

### 3.1 Metaspace (Java 8+)
- Pre-Java 8 this was **PermGen** (a fixed-size part of the heap) — a frequent source of `OutOfMemoryError: PermGen`.
- Java 8 replaced it with **Metaspace**, which lives in **native memory** (off-heap) and **grows dynamically** by default.
- Still can OOM: `OutOfMemoryError: Metaspace` — usually from **class-loading leaks** (loading too many classes / not releasing class loaders — recall previous note; common with dynamic proxies, repeated redeploys).
> Cap it if needed: `-XX:MaxMetaspaceSize` (options note).

### 3.2 String pool
The **string intern pool** (recall Phase 1.4) lives in the **heap** (since Java 7), not the method area.

---

## 4. JVM Stacks (Per-Thread Execution)

Each thread has its **own stack** (recall Phase 0.1 — this is why local variables are thread-safe). The stack holds **frames** — one per method call:
```
Thread stack (grows with each call, shrinks on return):
  +-----------------------+  <- top (current method)
  | frame: process()      |   - local variables
  +-----------------------+   - operand stack (for bytecode computation)
  | frame: calculate()    |   - return address
  +-----------------------+
  | frame: main()         |
  +-----------------------+
```
### 4.1 Stack frames
Each frame contains:
- **Local variable array** (method params + locals — primitives and references).
- **Operand stack** (working space for bytecode operations, e.g., `iadd`).
- **Frame data** (return address, reference to the constant pool).

### 4.2 StackOverflowError
Deep/infinite recursion piles up frames until the stack is exhausted → **`StackOverflowError`** (recall Phase 0.1 guard page, Phase 1.2 recursion):
```java
int bad(int n) { return bad(n + 1); }   // no base case -> StackOverflowError
```
Stack size is set with `-Xss` (options note). Bigger stacks allow deeper recursion but cost memory per thread.

---

## 5. PC Register & Native Method Stack

- **PC (Program Counter) Register:** per-thread; holds the address of the **current bytecode instruction** (recall Phase 0.1 instruction cycle / the CPU's program counter — the JVM has its own per-thread).
- **Native Method Stack:** per-thread; holds frames for **native (non-Java) methods** called via **JNI** (e.g., OS calls, native libraries).

---

## 6. The Execution Engine

The execution engine **runs the bytecode**. It has two ways to execute, used together (**tiered compilation**):

### 6.1 The Interpreter
Reads and executes bytecode **one instruction at a time** (its own fetch-decode-execute loop — recall Phase 0.1):
- **Pro:** starts **immediately** (no compilation delay).
- **Con:** slower per-instruction than native code.

### 6.2 The JIT (Just-In-Time) Compiler
Identifies **"hot" code** (frequently executed methods/loops) and **compiles it to optimized native machine code** at runtime:
- **Pro:** near-native speed for hot paths.
- **Con:** compilation takes time/CPU; only worth it for hot code.

### 6.3 The hybrid: tiered compilation
HotSpot uses **both** — start interpreting (fast startup), then JIT-compile hot code (fast steady-state):
```
Method first runs        -> INTERPRETED (immediate)
Runs many times (hot)    -> JIT compiles it -> NATIVE CODE (fast)
```
The JIT has two compilers:
| Compiler | Role |
|----------|------|
| **C1 (client)** | Fast compilation, lighter optimization (quick wins) |
| **C2 (server)** | Slower compilation, aggressive optimization (peak performance) |
- **Tiered compilation** uses C1 first, then promotes very-hot code to C2.
> ⚠️ This explains **JVM warm-up:** an app is **slow right after startup** (interpreting) and gets **faster** as the JIT kicks in. Critical for benchmarking (use JMH — Phase 13) and for understanding cold-start latency (containers/serverless).

---

## 7. JIT Optimizations

Because the JIT compiles at runtime, it can use **runtime profiling** to optimize aggressively — often beyond what a static (ahead-of-time) compiler could:
| Optimization | What it does |
|--------------|--------------|
| **Method inlining** | Replace a method call with its body (eliminates call overhead) — the most impactful |
| **Escape analysis** | If an object never "escapes" a method, allocate it on the stack (or eliminate it) — avoids heap allocation/GC |
| **Dead code elimination** | Remove code whose results are never used |
| **Loop optimizations** | Unrolling, hoisting invariant code out of loops |
| **Devirtualization** | Turn a virtual call into a direct/inlined call when only one implementation is seen (recall Phase 1.3 polymorphism) |
| **Branch prediction hints / speculation** | Optimize the common path based on observed behavior |

> **Escape analysis** is why "creating short-lived objects" is often cheap — the JIT may avoid heap allocation entirely. This is also why micro-benchmarking by hand is unreliable (the JIT optimizes away "useless" code) → use **JMH** (Phase 13).

### 7.1 AOT & GraalVM (awareness)
- **AOT (Ahead-Of-Time) compilation** compiles bytecode to native code **before** running (no warm-up) — e.g., **GraalVM native image** (Phase 16.5).
- Trade-off: **fast startup + low memory** (great for serverless/containers) **vs** lower peak throughput (no runtime profiling) and limited reflection/dynamic features.
- **GraalVM** also has a high-performance JIT compiler usable in the regular JVM.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Confusing which OOM you have | Heap space / Metaspace / GC overhead are different causes |
| Thinking PermGen still exists | It's **Metaspace** (native memory) since Java 8 |
| Benchmarking right after startup | The JVM warms up (JIT) — use JMH, warm up first |
| Hand-rolled micro-benchmarks | The JIT eliminates dead code — results are misleading |
| Deep recursion blowing the stack | Convert to iteration / increase `-Xss` (Phase 1.2) |
| Metaspace OOM ignored | Often a class-loading leak (proxies, redeploys) |
| Assuming objects always go on the heap | Escape analysis may stack-allocate/eliminate them |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Memory tuning (Phase 10, 13):** `-Xms`/`-Xmx` (heap), `-XX:MaxMetaspaceSize`, `-Xss` (stack) — sized per the data areas; in containers use `-XX:MaxRAMPercentage`.
- **JVM warm-up** explains slow first requests after deploy → warm-up strategies, readiness probes (Phase 9, 10).
- **`OutOfMemoryError` diagnosis (Phase 13):** heap dumps (next note) for heap-space; class-loader analysis for Metaspace.
- **GraalVM native image (Phase 16.5):** eliminates warm-up for fast container/serverless startup.
- **Escape analysis** justifies favoring small, short-lived objects and immutability (Phase 1.3) — the JIT handles them efficiently.
- **GC** (next note) operates on the heap generations described here.

---

## 10. Quick Self-Check Questions

1. List the runtime data areas; which are shared vs per-thread?
2. What lives on the heap, and how is it divided (generations)?
3. What is a TLAB and why does it help?
4. What replaced PermGen in Java 8, and where does it live?
5. What's in a stack frame, and what causes `StackOverflowError`?
6. What's the difference between the interpreter and the JIT? Why use both?
7. What are C1 and C2, and what is tiered compilation?
8. Explain JVM warm-up and why it matters for benchmarking.
9. Name three JIT optimizations; what does escape analysis enable?

---

## 11. Key Terms Glossary

- **Heap:** shared region for all objects (GC-managed).
- **Young/Old generation, Eden, Survivor:** heap subdivisions (generational GC).
- **TLAB:** per-thread allocation buffer (lock-free allocation).
- **Method Area / Metaspace:** class metadata & statics (native memory since Java 8).
- **PermGen:** the pre-Java-8 method area (fixed size).
- **JVM stack / frame:** per-thread call stack / per-method frame.
- **Operand stack / local variable array:** bytecode working space / method locals.
- **PC register:** current-instruction pointer (per thread).
- **Native method stack:** frames for JNI/native calls.
- **Interpreter / JIT:** instruction-by-instruction / compile-hot-to-native.
- **C1 / C2 / tiered compilation:** fast / aggressive compilers used together.
- **JIT optimizations:** inlining, escape analysis, dead-code elimination, devirtualization.
- **AOT / GraalVM native image:** compile-before-run (fast startup, no warm-up).

---

*Previous topic: **Class Loader Subsystem**.*
*Next topic: **Garbage Collection Deep Dive**.*
