# How Programs Are Loaded Into Memory

> **Phase 0 — Computer Science Fundamentals → 0.1 How Computers Work**
> Goal: Understand the full journey from a file on disk to a running process in RAM — what the OS, the **loader**, and the CPU actually do to "run" a program.

---

## 0. The One-Sentence Summary

A program on disk is just a **passive file** (machine code + data + metadata). To *run* it, the OS creates a **process**, the **loader** copies/maps the file's pieces into a virtual address space, sets up the stack/heap, wires up shared libraries, and finally jumps the CPU to the program's entry point.

> **Program** = a file on disk (static).
> **Process** = a running instance of that program in memory (active).

---

## 1. From Source Code to an Executable (the build pipeline)

Before anything can be loaded, source code becomes an executable file. (For Java this differs — see §8.)

```
  source.c
     |  (1) Preprocessor   -> expands #include, #define
     v
  expanded source
     |  (2) Compiler       -> translates to assembly
     v
  assembly (.s)
     |  (3) Assembler      -> machine code object file
     v
  object file (.o)   <-- relocatable, has unresolved symbols
     |  (4) Linker         -> combines objects + libraries, resolves symbols
     v
  executable (e.g. a.out / ELF / PE)   <-- ready to load
```

| Stage | Tool | Output |
|-------|------|--------|
| Preprocess | Preprocessor | Pure source |
| Compile | Compiler | Assembly |
| Assemble | Assembler | Object file (`.o`) — relocatable |
| Link | Linker | Executable / shared library |

**Static vs dynamic linking** (important — affects loading):
- **Static linking:** library code is copied *into* the executable at build time. Bigger file, self-contained, faster startup.
- **Dynamic linking:** the executable only *references* shared libraries (`.so` on Linux, `.dll` on Windows, `.dylib` on macOS). They are loaded and linked **at load/run time**. Smaller file, shared in RAM across processes, but needs the dynamic linker.

---

## 2. The Executable File Format

An executable isn't raw machine code — it's a structured file the OS knows how to read. Common formats:

| OS | Format |
|----|--------|
| Linux / Unix | **ELF** (Executable and Linkable Format) |
| Windows | **PE** (Portable Executable, `.exe`/`.dll`) |
| macOS | **Mach-O** |

An ELF file is organized into **sections** (for linking) and **segments** (for loading). Key parts the loader cares about:

```
+----------------------------+
|  ELF Header                |  magic number, architecture, entry point address
+----------------------------+
|  Program Headers           |  tells the loader WHICH segments to map & HOW
+----------------------------+
|  .text   (code)            |  the machine instructions      -> read + execute
|  .rodata (read-only data)  |  constants, string literals    -> read only
|  .data   (init. globals)   |  initialized global/static vars -> read + write
|  .bss    (uninit. globals) |  zero-initialized vars (takes NO file space)  
+----------------------------+
|  Section Headers           |  metadata used by linker/debuggers
+----------------------------+
```

> **`.bss` trick:** Uninitialized globals don't need to be stored in the file — the loader just records "reserve N zeroed bytes." This keeps executables small.

---

## 3. The Loading Sequence (Step by Step)

When you run a program (e.g., type `./app` or the shell calls `execve`), here's what happens:

```
1. User asks to run program  (shell -> fork() + execve())
2. Kernel verifies the file  (valid ELF? permissions? architecture match?)
3. Kernel creates a NEW virtual address space for the process
4. Loader maps file segments into that address space:
       .text  -> code segment   (read/execute)
       .data  -> data segment   (read/write)
       .bss   -> zeroed region   (read/write)
5. Set up the run-time memory: create the STACK, prepare the HEAP region
6. If dynamically linked -> invoke the DYNAMIC LINKER/LOADER
       - load required shared libraries (.so)
       - resolve and bind symbols (functions/variables)
7. Set up arguments: argc, argv, environment variables on the stack
8. Set the CPU's instruction pointer to the ENTRY POINT
9. CPU begins executing -> program is now a running PROCESS
```

### 3.1 `fork()` + `execve()` (the Unix model)
- **`fork()`** creates a new process by *duplicating* the calling process.
- **`execve()`** then *replaces* that process's memory image with the new program. This is the moment "loading" really happens.
- The shell uses this pair: fork a child, then exec the program in it.

---

## 4. Demand Paging — The Loader is Lazy

A crucial modern detail: **the whole program is usually NOT copied into RAM up front.**

Instead, the loader uses **memory-mapping (`mmap`)** to associate file regions with virtual addresses, *without* reading them yet. Pages are loaded **on demand**:

```
CPU tries to execute/access an address in a not-yet-loaded page
            |
            v
   PAGE FAULT (CPU exception -> kernel)
            |
            v
   Kernel reads the needed 4 KB page from the executable file on disk into RAM
            |
            v
   Updates page table; resumes the instruction (transparent to the program)
```

Benefits:
- **Fast startup** — only what's actually used gets loaded.
- **Memory efficiency** — unused code/data never occupies RAM.
- **Sharing** — read-only code pages (e.g., the C library) can be shared by many processes pointing to the same physical frames.

---

## 5. Resulting Process Memory Layout

After loading, the process's virtual address space looks like the familiar map (this connects directly to the previous *Stack vs Heap* note):

```
  High Addresses
+---------------------------+
|         STACK             |  set up by loader; grows DOWN; gets argc/argv/env
|           v               |
|        (free gap)         |
|           ^               |
|         HEAP              |  starts empty; grows UP via brk/mmap at runtime
+---------------------------+
|  BSS    (zeroed globals)  |  reserved & zeroed by loader
+---------------------------+
|  DATA   (init. globals)   |  copied from file
+---------------------------+
|  RODATA (constants)       |  read-only
+---------------------------+
|  TEXT   (code)            |  mapped from file, read+execute
+---------------------------+
  Low Addresses
+ shared libraries mapped somewhere in between (dynamic linking)
```

---

## 6. The Dynamic Linker / Loader (for dynamically linked programs)

If the program uses shared libraries, the kernel hands control to a special program first — the **dynamic linker** (`ld.so` / `ld-linux.so` on Linux):

1. Reads the list of required shared libraries from the executable.
2. Locates them on disk (search paths, `LD_LIBRARY_PATH`, cache).
3. Maps each library into the address space.
4. **Resolves symbols** — connects function calls to their real addresses.
5. Performs **relocations** — patches addresses now that final locations are known.

**Lazy binding (PLT/GOT):** Function addresses are often resolved only on **first call**, using the **Procedure Linkage Table (PLT)** and **Global Offset Table (GOT)**, to speed up startup. (Awareness-level detail.)

---

## 7. ASLR — A Security Note

Modern OSes use **ASLR (Address Space Layout Randomization)**: the stack, heap, and library load addresses are **randomized** each run. This prevents attackers from reliably predicting where code/data lives (defends against buffer-overflow exploits). It's why the same program shows different addresses in memory on each launch.

---

## 8. How Java Programs Are Loaded (the important difference)

Java does **not** produce a native ELF/PE executable from your code. The flow is different — and matters for every Java backend engineer:

```
  MyApp.java
     |  javac  (compile)
     v
  MyApp.class  -> contains BYTECODE (platform-independent), not machine code
     |
     v
  You run:  java MyApp
     |
     v
  The OS loads the JVM itself (the JVM is a native ELF/PE program!)
     |
     v
  The JVM's CLASS LOADER loads .class files into the JVM at runtime
     |
     v
  Bytecode is verified, then interpreted and/or JIT-compiled to native machine code
     |
     v
  Native code runs on the CPU
```

Key points:
- **Two layers of loading:** (1) the OS loads the **JVM** (a normal native program, loaded exactly as described in §3); (2) the JVM's **Class Loader** then loads your `.class` files into the JVM's own managed memory.
- Your classes are loaded **lazily**, on first use — analogous to demand paging, but at the class level.
- This is why "how programs are loaded" has a *native* layer (the JVM) and a *managed* layer (your bytecode).
- Deeper detail (Bootstrap/Platform/Application class loaders, delegation model, verification) is covered later in **Phase 1.11 — JVM Deep Dive → Class Loader Subsystem**.

---

## 9. Quick Comparison: Native vs Java Loading

| Step | Native (C/C++) | Java |
|------|----------------|------|
| Compiled to | Machine code (ELF/PE) | Bytecode (`.class`) |
| What the OS loads | Your executable directly | The **JVM** executable |
| Who loads your code | OS loader | JVM **Class Loader** |
| When code is loaded | Demand paging (per page) | Lazily (per class, on first use) |
| Final execution | CPU runs native code directly | JVM interprets / JIT-compiles to native |
| Portability | Per-platform binary | Same `.class` runs on any JVM |

---

## 10. Why This Matters for a Backend (Java) Engineer

- Explains **JVM startup cost** — the OS loads the JVM, then classes load on demand. (Why native images / GraalVM exist to cut startup time — Phase 16.5.)
- **`ClassNotFoundException` / `NoClassDefFoundError`** come directly from the class-loading step.
- Understanding **static vs dynamic linking** and shared libraries helps when debugging native dependencies (e.g., JNI, native libs in Docker images).
- **Demand paging** explains why a freshly started app can be slow on first requests (code/pages/classes warming up).
- Connects to **containerization** (Phase 10): a Docker image must contain the JVM + your classes + any native `.so` files for loading to succeed.

---

## 11. Quick Self-Check Questions

1. What is the difference between a *program* and a *process*?
2. What does the **linker** do, and how does static linking differ from dynamic linking?
3. What is stored in `.text`, `.data`, and `.bss`? Why does `.bss` take no space in the file?
4. What is **demand paging**, and what triggers a page to actually load?
5. What is the role of `fork()` vs `execve()`?
6. In Java, what two layers of "loading" happen, and who performs each?
7. What does **ASLR** protect against?
8. Why does a Java app start slower than an equivalent native binary?

---

## 12. Key Terms Glossary

- **Loader:** OS component that maps an executable into memory and prepares it to run.
- **ELF / PE / Mach-O:** executable file formats (Linux / Windows / macOS).
- **Linker:** combines object files + libraries, resolves symbols into an executable.
- **Static linking:** library code copied into the executable at build time.
- **Dynamic linking:** libraries referenced and loaded at run time (`.so`/`.dll`).
- **Dynamic linker (`ld.so`):** loads & links shared libraries at run time.
- **Segment / Section:** loadable region / link-time metadata region of an executable.
- **`.text` / `.data` / `.bss`:** code / initialized globals / uninitialized globals.
- **Entry point:** address where execution begins.
- **`fork()` / `execve()`:** create a process / replace its memory image with a program.
- **Demand paging:** loading pages from disk only when accessed.
- **Page fault:** CPU exception when accessing a not-yet-loaded page.
- **ASLR:** randomizing memory layout for security.
- **Class Loader:** JVM component that loads `.class` bytecode at runtime.
- **Bytecode:** platform-independent instructions in a `.class` file.

---

*Previous topic: **Stack memory vs Heap memory (hardware level)**.*
*Next topic in roadmap: **Process vs Thread at the OS level**.*
