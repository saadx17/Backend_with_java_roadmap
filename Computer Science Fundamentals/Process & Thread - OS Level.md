A **process** is a running program with its **own private memory space** and resources, isolated from other processes.
**Process = container of resources + at least one thread.**

A **thread** is a unit of execution that lives **inside** a process and **shares that process's memory** with sibling threads, but has its **own stack and registers**.
**Thread = the thing that actually runs on a CPU core.**

A process always has **at least one thread** (the main thread). A process with multiple threads is **multithreaded**.

## 1. What is a Process?
When the OS loads a program (see previous note), it creates a **process**: an isolated execution environment with everything the program needs to run.

A process owns:
- Its **own virtual address space** (Text, Data, BSS, Heap, Stack — see Stack vs Heap note)
- Open **file descriptors** (files, sockets, pipes)
- **Environment variables** and command-line args
- A unique **Process ID (PID)**
- Security context (user/group, permissions)
- Memory mappings, signal handlers, etc.

**Key property: Isolation.** One process cannot directly read/write another process's memory. The OS + MMU (hardware) enforce this. If one process crashes, others are unaffected.
```
+-------------------- PROCESS A --------------------+
|  PID, file descriptors, env vars, security        |
|  +---------------------------------------------+  |
|  |  VIRTUAL ADDRESS SPACE (private)            |  |
|  |  Stack | Heap | BSS | Data | Text/Code      |  |
|  +---------------------------------------------+  |
|  Main Thread (registers + its own stack)          |
+---------------------------------------------------+
        (completely isolated from Process B)
```

## 2. What is a Thread?
A thread is a **lightweight unit of execution within a process**. Multiple threads in the same process run "the same program" but at different points in the code, concurrently.

Each thread has its **own**:
- **Stack** (its own call frames & local variables)
- **Registers** (including its own **program counter / instruction pointer** and **stack pointer**)
- Thread ID
- Thread-local storage (e.g., Java's `ThreadLocal`)

All threads in a process **share**:
- **Heap** (all dynamically allocated objects)
- **Code/Text segment** (the instructions)
- **Global/static data** (Data + BSS)
- **Open file descriptors** and other process resources

```
+----------------------- PROCESS -----------------------+
|  SHARED:  Heap | Globals (Data/BSS) | Code | Files     |
|                                                        |
|  Thread 1            Thread 2            Thread 3       |
|  +-----------+       +-----------+       +-----------+  |
|  | own stack |       | own stack |       | own stack |  |
|  | registers |       | registers |       | registers |  |
|  | PC, SP    |       | PC, SP    |       | PC, SP    |  |
|  +-----------+       +-----------+       +-----------+  |
+--------------------------------------------------------+
```

> This is **the single most important diagram** for understanding concurrency: threads share the heap and globals (→ need synchronization), but each has a private stack (→ locals are naturally thread-safe).

## 3. Process Vs. Thread

| Aspect | **Process** | **Thread** |
|--------|-------------|------------|
| Memory space | Own private address space | Shares process's address space |
| Heap | Private to the process | **Shared** among threads |
| Stack | One per process (main thread) | **Each thread has its own** |
| Code/globals | Private to process | **Shared** among threads |
| Isolation | Strong (OS + MMU enforced) | None (within a process) |
| Creation cost | Heavy (new address space) | Light (reuse address space) |
| Context switch cost | Higher (swap address space, flush TLB) | Lower (same address space) |
| Communication | IPC (pipes, sockets, shared mem) — complex | Shared memory — direct but needs sync |
| Crash impact | Isolated — one crash ≠ others | One thread crash can take down whole process |
| Created by | `fork()` / `CreateProcess` | `pthread_create` / `clone` / `CreateThread` |

## 4. Why Threads are "Lightweight"?
Creating a process means building a whole new address space, page tables, file descriptor table, etc. Creating a thread mostly means allocating a **new stack** and a **register set** — the expensive address space is **reused**.

| Operation | Process | Thread |
|-----------|---------|--------|
| Allocate address space | Yes (expensive) | No (shared) |
| Allocate stack | Yes | Yes |
| Copy/set up page tables | Yes | No |
| Set up file descriptors | Yes | Shared |

This is why servers use **thread pools** rather than spawning a process per request.

## 5. Context Switching (Process vs Thread)

A single CPU core runs only one thread at a time. The OS **scheduler** rapidly switches between them to create the illusion of parallelism. A **context switch** = saving the current execution state and loading another's.

- **Thread → thread (same process):** save/restore registers + stack pointer. The address space stays the same → cheaper.
- **Process → process:** all of the above **plus** switching the virtual address space, which usually means **flushing the TLB** (Translation Lookaside Buffer — the cache of virtual→physical address translations). This is more expensive.

> (Context switching gets its own dedicated note later in the roadmap, here we only contrast process vs thread cost.)

## 6. Concurrency vs Parallelism

- **Concurrency:** multiple tasks *in progress* over the same period, interleaved on one or more cores (structure).
- **Parallelism:** multiple tasks *literally executing at the same instant* on multiple cores (execution).

Threads enable **both**: concurrency on a single core (via context switching) and true parallelism across multiple cores.

```
Single core (concurrency):  T1--  T2--  T1--  T2--   (interleaved)
Multi  core (parallelism):  Core0: T1======
                            Core1: T2======          (simultaneous)
```

## 7. The Cost of Sharing: Why Concurrency is Hard
Because threads **share the heap and globals**, two threads touching the same data can collide:

```
Shared variable:  counter = 0   (lives on the HEAP / globals)

Thread A: read counter (0) -> add 1 -> write (1)
Thread B: read counter (0) -> add 1 -> write (1)   // ran in between!
Result: counter = 1   (should be 2!)  -> RACE CONDITION
```

This is why we need **synchronization** (locks, atomics, etc.). The private stack avoids this for local variables, but shared objects require coordination. (Deep coverage in **Phase 1.10 — Java Concurrency**.)

| Problem | Cause |
|---------|-------|
| **Race condition** | Unsynchronized access to shared data |
| **Deadlock** | Threads waiting on each other's locks forever |
| **Starvation** | A thread never gets CPU/lock time |
| **Visibility issues** | One thread's write not seen by another (caching/reordering) |

## 8. Inter-Process Communication (IPC)
Since processes are isolated, they can't just share variables. They communicate via OS-provided mechanisms:

| IPC Mechanism | Description |
|---------------|-------------|
| **Pipes / Named pipes** | Byte stream between processes |
| **Sockets** | Network or local communication (TCP/UDP/Unix sockets) |
| **Shared memory** | A region mapped into multiple processes (fastest) |
| **Message queues** | OS-managed message passing |
| **Signals** | Simple async notifications (e.g., `SIGTERM`) |
| **Files** | Coordinate via reading/writing files |

Threads, by contrast, just share variables on the heap — no IPC needed (but they pay the price in synchronization complexity).

## 9. How This Maps to Java
Java exposes OS threads directly (until virtual threads):

- A running JVM **is one process** (one PID).
- Each `Thread` you create is typically backed by a real **OS thread** (a "platform thread").
- All threads in the JVM **share the heap** → that's why object access needs synchronization.
- Each thread gets its **own JVM stack** (sized with `-Xss`) → local variables and method frames are per-thread.
- `ThreadLocal` = thread-private storage built on this model.

**Virtual Threads (Java 21, Project Loom):** Lightweight threads managed by the **JVM**, not the OS. Many virtual threads are multiplexed onto a few OS (platform) threads. This makes it cheap to have *millions* of threads — ideal for I/O-heavy backends. (Covered in **Phase 1.10 — Virtual Threads**.)

```
Platform thread:  Java Thread  <->  1 OS thread        (heavy, limited)
Virtual thread:   many Java Threads -> few OS threads  (light, scalable)
```

## 10. Why This Matters for a Backend (Java) Engineer

- A web server handles many requests via **threads** (thread pool), not processes — because threads are cheap and share caches/connections.
- Understanding **shared heap** explains _why_ you need `synchronized`, `volatile`, `ConcurrentHashMap`, atomics, etc.
- Understanding **isolation** explains why crashing one request's thread can risk the whole JVM, and why some systems run multiple JVM **processes** for fault isolation.
- **Context switch cost** informs thread pool sizing — too many threads = thrashing.
- **Virtual threads** change the economics of concurrency for I/O-bound services (huge for modern backends).
- IPC concepts underpin **microservices** communication (sockets, message queues — Phases 11–12).

## 11. Quick Self-Check Questions

1. What does a process own that a thread does not?
2. What do threads in the same process **share**, and what do they keep **private**?
3. Why is creating a thread cheaper than creating a process?
4. Why is a process→process context switch more expensive than a thread→thread switch?
5. Why does sharing the heap make concurrency hard? Give a race condition example.
6. What is the difference between concurrency and parallelism?
7. How do isolated processes communicate (name 3 IPC mechanisms)?
8. In Java, what's the difference between a platform thread and a virtual thread?

## 12. Key Terms

- **Process:** running program with its own isolated address space + resources.
- **Thread:** unit of execution within a process; shares process memory.
- **PID:** unique Process ID.
- **Address space:** the private virtual memory of a process.
- **Main thread:** the first thread that runs when a process starts.
- **Thread-local storage:** memory private to a single thread.
- **Context switch:** saving one execution state and loading another.
- **TLB:** Translation Lookaside Buffer — cache of address translations, flushed on process switch.
- **Concurrency:** multiple tasks in progress (interleaved).
- **Parallelism:** multiple tasks executing simultaneously on multiple cores.
- **Race condition:** incorrect result from unsynchronized shared access.
- **Deadlock:** mutual waiting that never resolves.
- **IPC:** Inter-Process Communication (pipes, sockets, shared memory, etc.).
- **Platform thread:** Java thread backed 1:1 by an OS thread.
- **Virtual thread:** lightweight JVM-managed thread (Java 21+).