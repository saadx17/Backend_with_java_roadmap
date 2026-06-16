# JVM Architecture Overview

> **Phase 1 — Java Language Mastery → 1.11 JVM Deep Dive**
> Goal: Understand the big picture of the JVM — JDK vs JRE vs JVM, the three subsystems (class loader, runtime data areas, execution engine), and how bytecode runs — before diving into each part.

---

## 0. The Big Picture

The **JVM (Java Virtual Machine)** is the runtime that executes Java bytecode. It's *why* "write once, run anywhere" works: you compile to platform-independent **bytecode**, and a platform-specific JVM runs it.

```
MyApp.java  --javac-->  MyApp.class (bytecode)  --JVM-->  runs on any OS/CPU
```

> Understanding the JVM separates senior engineers from juniors: it explains memory issues (`OutOfMemoryError`), performance (JIT, GC pauses), class-loading errors, and how to tune production systems. This builds on **Phase 0.1** (how programs load, stack/heap, virtual memory).

---

## 1. JDK vs JRE vs JVM (recap & precise definitions)

These three are often confused. From smallest to largest:
```
JDK (development kit)
 ├── JRE (runtime environment)
 │    ├── JVM (the virtual machine)
 │    └── core libraries (java.lang, java.util, ...)
 └── development tools (javac, jdb, jar, javap, jconsole, ...)
```
| Term | What it is | Contains |
|------|-----------|----------|
| **JVM** | The engine that **runs** bytecode | Class loader, memory areas, execution engine, GC |
| **JRE** | Everything needed to **run** Java apps | JVM + standard libraries |
| **JDK** | Everything needed to **develop** Java apps | JRE + compiler (`javac`) + tools |

> **You develop with the JDK; you run with (at least) the JRE; the JVM does the execution.** Since Java 11, there's no separate JRE download — you get a JDK (and can create a slim runtime with `jlink`).

---

## 2. The JVM Is a Native Program (recap from Phase 0.1)

A key insight (from the "how programs load" note): the **JVM itself is a native program** (an ELF/PE executable). When you run `java MyApp`:
```
1. The OS loads the JVM (a native program) into memory and starts it.
2. The JVM's CLASS LOADER loads your MyApp.class bytecode.
3. Bytecode is verified, then interpreted and/or JIT-compiled to native code.
4. Native code runs on the CPU.
```
So there are **two layers**: the OS runs the JVM; the JVM runs your classes. (Recall Phase 0.1.)

---

## 3. Bytecode — The Universal Instruction Set

`javac` compiles `.java` → **bytecode** (`.class`): platform-independent instructions for an imaginary "stack machine."
```java
int sum = a + b;
```
compiles to bytecode like:
```
iload_1      // push local var a onto the operand stack
iload_2      // push local var b
iadd         // pop two, add, push result
istore_3     // pop result into local var sum
```
- Bytecode is **not** machine code — the JVM interprets or JIT-compiles it to real CPU instructions (next note's execution engine).
- You can view bytecode with **`javap -c MyClass`** (bytecode note).
> Bytecode is the contract between the compiler and the JVM. The same `.class` runs on any JVM (Windows/Linux/Mac, x86/ARM) — portability.

---

## 4. The Three JVM Subsystems

The JVM has three major parts. The rest of section 1.11 is a deep dive into each:

```
+---------------------------------------------------------------+
|                            JVM                                |
|                                                               |
|  1. CLASS LOADER SUBSYSTEM                                    |
|     Loading -> Linking -> Initialization                      |
|              (gets bytecode into the JVM)                      |
|                          |                                    |
|                          v                                    |
|  2. RUNTIME DATA AREAS  (memory)                              |
|     Method Area (Metaspace) | Heap | Stacks | PC | Native    |
|                          |                                    |
|                          v                                    |
|  3. EXECUTION ENGINE                                          |
|     Interpreter + JIT Compiler + Garbage Collector            |
|              (runs the bytecode, manages memory)              |
+---------------------------------------------------------------+
```

### 4.1 Subsystem 1 — Class Loader Subsystem
**Loads, links, and initializes** classes into the JVM at runtime (lazily, on first use). Handles the bootstrap/platform/application loaders and the delegation model. (Next note.)
> This is the source of `ClassNotFoundException`, `NoClassDefFoundError`, and class-loading leaks.

### 4.2 Subsystem 2 — Runtime Data Areas (Memory)
Where the JVM stores everything at runtime:
| Area | Holds | Shared? |
|------|-------|---------|
| **Method Area / Metaspace** | Class metadata, static fields, constant pool | Shared |
| **Heap** | All objects (`new`), arrays | Shared (GC-managed) |
| **Stacks** | Per-thread frames, local variables | Per-thread |
| **PC Register** | Current instruction address | Per-thread |
| **Native Method Stack** | Native (JNI) call frames | Per-thread |
> This is where `OutOfMemoryError` (heap/metaspace) and `StackOverflowError` (stack) come from. (Recall Phase 0.1 stack vs heap.)

### 4.3 Subsystem 3 — Execution Engine
**Executes** the bytecode and **manages memory**:
- **Interpreter:** reads and executes bytecode instruction by instruction (fast startup, slower steady-state).
- **JIT Compiler:** compiles hot bytecode to optimized native code (slow start, fast steady-state).
- **Garbage Collector:** reclaims unreachable objects from the heap automatically.
> This is where JIT optimization (performance) and GC (pauses, tuning) live. (Next notes.)

---

## 5. How It All Flows Together

```
You write MyApp.java
   |  javac
   v
MyApp.class (bytecode)
   |  java MyApp  -> OS loads the JVM
   v
[ CLASS LOADER ]  loads/links/initializes MyApp + dependencies
   v
[ RUNTIME DATA AREAS ]  allocate heap, create stacks, store class metadata
   v
[ EXECUTION ENGINE ]
   - Interpreter runs bytecode immediately
   - JIT compiles "hot" methods to native code for speed
   - GC reclaims unreachable heap objects in the background
   v
Program runs on the CPU
```

---

## 6. JVM Implementations & Variants (awareness)

| Implementation | Note |
|----------------|------|
| **HotSpot** (OpenJDK/Oracle) | The standard, most widely used JVM |
| **GraalVM** | High-performance JVM + ahead-of-time **native image** compilation (Phase 16.5) |
| **OpenJ9** (Eclipse) | Lower memory footprint, IBM-originated |
> Most backends run **HotSpot** (via OpenJDK distributions like Temurin/Adoptium, Amazon Corretto). **GraalVM native images** (Phase 16.5) trade JIT for fast startup/low memory — relevant for serverless/containers.

---

## 7. Why This Matters for a Backend (Java) Engineer

- **Diagnosing memory issues:** knowing heap vs metaspace vs stack tells you which `OutOfMemoryError`/`StackOverflowError` you have and how to fix it (next notes).
- **Performance:** understanding JIT warm-up explains why apps are slow right after startup; GC understanding explains latency spikes (Phase 13).
- **Class-loading errors** (`ClassNotFoundException` vs `NoClassDefFoundError`) are common in deployments/dependency conflicts (next note).
- **Container tuning:** JVM memory settings in Docker/K8s (`-Xmx`, `MaxRAMPercentage`) require knowing the data areas (Phase 10).
- **Startup time:** GraalVM native images cut JIT/class-loading startup cost (Phase 16.5).

---

## 8. Quick Self-Check Questions

1. What is the JVM, and how does it enable "write once, run anywhere"?
2. Distinguish JDK, JRE, and JVM.
3. Why is the JVM described as "a native program that runs your bytecode"?
4. What is bytecode, and how is it different from machine code?
5. Name the three JVM subsystems and what each does.
6. List the runtime data areas and which are shared vs per-thread.
7. What are the three parts of the execution engine?
8. What's the difference between HotSpot and GraalVM?

---

## 9. Key Terms Glossary

- **JVM:** the virtual machine that executes Java bytecode.
- **JRE:** JVM + standard libraries (to run apps).
- **JDK:** JRE + dev tools/compiler (to build apps).
- **Bytecode:** platform-independent `.class` instructions.
- **Class loader subsystem:** loads/links/initializes classes.
- **Runtime data areas:** JVM memory (method area, heap, stacks, PC, native stack).
- **Execution engine:** interpreter + JIT + GC.
- **Interpreter / JIT:** instruction-by-instruction / compile-hot-code-to-native.
- **Garbage collector (GC):** reclaims unreachable heap objects.
- **HotSpot / GraalVM / OpenJ9:** JVM implementations.

---

*This is the first note of **Section 1.11 — JVM Deep Dive**.*
*Next topic: **Class Loader Subsystem**.*
