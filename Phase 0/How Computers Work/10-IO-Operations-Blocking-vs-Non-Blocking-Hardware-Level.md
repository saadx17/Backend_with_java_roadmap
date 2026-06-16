# I/O Operations (Blocking vs Non-Blocking at Hardware Level)

> **Phase 0 — Computer Science Fundamentals → 0.1 How Computers Work**
> Goal: Understand how data physically moves between the CPU and the outside world (disk, network, keyboard), and what "blocking" vs "non-blocking" really means at the hardware/OS level — the foundation for high-performance backend I/O.

---

## 0. The One-Sentence Summary

**I/O (Input/Output)** is any data transfer between the CPU/RAM and an external device (disk, network card, keyboard). Because devices are **vastly slower** than the CPU, the key question is: *while waiting for the device, does the thread sit idle (**blocking**) or do something else (**non-blocking**)?* This single choice determines how many concurrent connections a server can handle.

---

## 1. Why I/O Is the Central Problem

The CPU is astonishingly fast compared to I/O devices. The classic **latency numbers** make this concrete:

| Operation | Approx. latency | Scaled to "1 second = 1 CPU cycle" |
|-----------|-----------------|------------------------------------|
| L1 cache reference | ~1 ns | ~1 second |
| Main memory (RAM) access | ~100 ns | ~minutes |
| SSD random read | ~16 µs | ~hours |
| Network round trip (same datacenter) | ~0.5 ms | ~days |
| Disk (HDD) seek | ~10 ms | ~weeks |
| Network round trip (cross-continent) | ~150 ms | ~months/years |

> **Takeaway:** while waiting for a network packet or disk read, the CPU could have executed **millions** of instructions. The entire art of I/O is *not wasting that time.*

This is why backends are usually **I/O-bound**, not CPU-bound — they spend most of their time waiting on databases, network calls, and disks.

---

## 2. How I/O Physically Works (Hardware Level)

The CPU doesn't talk to disks/NICs directly. There's a chain of hardware:

```
CPU  <->  Device Controller  <->  Physical Device
                |
            uses RAM via DMA, signals CPU via interrupts
```

Key hardware mechanisms:

### 2.1 Device Controllers & Registers
Each device (disk, NIC) has a **controller** with registers. The CPU issues commands by writing to these registers ("read sector X into buffer Y").

### 2.2 Three ways the CPU handles I/O transfer

| Method | How it works | Cost |
|--------|--------------|------|
| **Programmed I/O (polling)** | CPU repeatedly checks ("is it done yet?") and copies data byte-by-byte | Wastes CPU cycles busy-waiting |
| **Interrupt-driven I/O** | CPU issues command, does other work; device sends an **interrupt** when done | Better, but CPU still copies the data |
| **DMA (Direct Memory Access)** | A **DMA controller** moves data directly between device and RAM **without the CPU**; interrupts the CPU only when the whole transfer is complete | Best — frees the CPU during transfer |

### 2.3 Interrupts
An **interrupt** is a hardware signal that tells the CPU "an event happened" (e.g., disk read finished, packet arrived). The CPU pauses its current work, runs an **interrupt handler** (ISR), then resumes. Interrupts are how devices asynchronously notify the CPU — the hardware basis for *non-blocking* I/O.

### 2.4 DMA in detail
```
1. CPU tells DMA controller: "move 4KB from disk to RAM address 0x...".
2. CPU goes off and does other work (or the thread is switched out).
3. DMA controller transfers the data directly into RAM.
4. When done, DMA raises an INTERRUPT.
5. CPU's interrupt handler marks the I/O as complete.
```
DMA is *why* a CPU core can stay productive while gigabytes move through it.

---

## 3. The User Space / Kernel Space Boundary

Application code (user space) cannot touch hardware directly — it must ask the **OS kernel** via a **system call** (e.g., `read()`, `write()`, `recv()`, `send()`).

```
Your app (user space)
     |  read()  -> SYSTEM CALL (mode switch to kernel)
     v
Kernel (privileged)
     |  talks to device controller / DMA
     v
Hardware device
```

Data typically lands in a **kernel buffer** first (via DMA), then is **copied to your application's buffer**. This boundary crossing (mode switch + copy) is part of I/O cost, and it's exactly where "blocking vs non-blocking" is decided.

---

## 4. Blocking I/O

In **blocking I/O**, when a thread makes an I/O call, the OS **puts the thread to sleep** (blocked state) until the data is ready. The thread does nothing — it doesn't even consume CPU — but it is **stuck**, holding its stack and resources.

```
Thread: read(socket)
   |
   |--- OS: no data yet -> BLOCK this thread (context-switch it out)
   |        ... thread sleeps, CPU runs other threads ...
   |--- data arrives (interrupt) -> OS wakes the thread
   v
read() returns with the data -> thread continues
```

Characteristics:
- **Simple to program** — code reads top to bottom, sequentially.
- **One thread per connection** — each blocked thread is tied up waiting.
- Scaling to many connections needs **many threads** → high memory (each stack) + lots of **context switching** (recall previous note).

> This is the classic "thread-per-request" server model. It's intuitive but doesn't scale to tens of thousands of concurrent connections (the **C10K problem**).

---

## 5. Non-Blocking I/O

In **non-blocking I/O**, an I/O call returns **immediately** — if data isn't ready, it returns a "would block / try again" status instead of sleeping the thread. The thread stays free to do other work.

```
Thread: read(socket)  -> returns immediately
   |        "no data yet" (EWOULDBLOCK)
   |--- thread does other useful work / checks other sockets
   |--- later: read(socket) again -> "here's the data"
   v
```

The problem: naively, the thread would have to **poll** ("ready yet? ready yet?") — wasteful. The real power comes from combining non-blocking sockets with an **I/O multiplexer** (next section).

---

## 6. I/O Multiplexing — One Thread, Many Connections

**I/O multiplexing** lets a **single thread monitor thousands of connections** and act only on the ones that are ready. The OS provides "readiness notification" mechanisms:

| Mechanism | OS | Scalability |
|-----------|-----|-------------|
| **`select()`** | All | Poor (scans all FDs each call, limit ~1024) |
| **`poll()`** | All | Slightly better, still O(n) scan |
| **`epoll`** | Linux | Excellent — O(1) notifications, scales to 100k+ |
| **`kqueue`** | BSD/macOS | Excellent (like epoll) |
| **IOCP** | Windows | Excellent (completion-based) |

```
Single thread + epoll:
   epoll_wait()  -> "sockets 3, 17, 42 are ready to read"
        |
        +-- handle socket 3
        +-- handle socket 17
        +-- handle socket 42
   loop back to epoll_wait()
```

> This is the **event loop** model — one (or a few) threads handling enormous numbers of connections. It's how Nginx, Node.js, Netty, and reactive Java servers achieve massive concurrency with few threads → minimal context switching and memory.

---

## 7. Readiness vs Completion (two models of non-blocking)

| Model | Question answered | Examples |
|-------|-------------------|----------|
| **Readiness-based** | "Is this socket *ready* so I can do the I/O now?" | epoll, kqueue, Java NIO `Selector` |
| **Completion-based** | "Tell me when the I/O is *already done* (data in my buffer)." | Windows IOCP, Linux `io_uring`, Java NIO.2 `AsynchronousChannel` |

- **Readiness:** the OS says "go ahead, you can read now"; *you* do the read.
- **Completion (true async):** you submit the operation, the OS (via DMA) completes it and hands you the finished result.

---

## 8. Synchronous vs Asynchronous vs Blocking vs Non-Blocking

These two axes are often confused. They are **different things**:

- **Blocking / Non-blocking** = does the *call* wait for the resource to be ready?
- **Synchronous / Asynchronous** = does *your code* wait for the *result*, or get notified later (callback/future)?

| Combination | Meaning |
|-------------|---------|
| **Sync + Blocking** | Call waits, thread sleeps until done (classic `InputStream.read()`) |
| **Sync + Non-blocking** | Call returns immediately; you poll/check readiness yourself (NIO `Selector`) |
| **Async + (Non-blocking)** | You submit the op and get notified on completion (callbacks/futures, NIO.2, `io_uring`) |

> Rule of thumb: *Blocking* is about the **thread**; *Asynchronous* is about the **result delivery**.

---

## 9. How This Maps to Java I/O

This hardware/OS foundation directly explains Java's three I/O families (detailed in **Phase 1.9 — Java I/O**):

| Java API | Model | Underlying mechanism |
|----------|-------|----------------------|
| **java.io** (`InputStream`/`OutputStream`, classic) | Blocking, stream-based | One thread per connection, blocking syscalls |
| **java.nio** (`Channel` + `Selector`, Java 1.4+) | Non-blocking, readiness | `select`/`poll`/**epoll**/kqueue |
| **java.nio.2 / NIO async** (`AsynchronousChannel`) | Asynchronous, completion | OS completion ports / async I/O |

Higher-level frameworks build on these:
- **Netty** → built on Java NIO (epoll) → powers Spring WebFlux, gRPC, many DB drivers.
- **Spring MVC** (Phase 5) → traditional **blocking**, thread-per-request (simple, great with thread pools / virtual threads).
- **Spring WebFlux** (Phase 16) → **non-blocking**, event-loop, reactive — for high-concurrency, I/O-bound workloads.

### Virtual Threads (Java 21) — the game changer
Virtual threads let you **write simple blocking-style code** that *scales like non-blocking*:
- When a virtual thread does a "blocking" I/O call, the JVM **unmounts** it from its carrier (OS) thread and runs something else — the OS thread is **never actually blocked**.
- Under the hood, the JDK uses non-blocking I/O (epoll) and **parks** the virtual thread until data is ready.
- Result: thread-per-request simplicity **+** event-loop scalability, without the callback complexity of reactive code.

```
Blocking code on a virtual thread:
  String data = socket.read();   // looks blocking...
  // ...but JVM parks the virtual thread & frees the OS thread under the hood
```

---

## 10. Why This Matters for a Backend (Java) Engineer

- Backends are **I/O-bound** — mastering I/O models is the difference between a server handling 200 vs 200,000 connections.
- Explains the **C10K problem** and why event loops / NIO / virtual threads exist.
- Informs the choice: **Spring MVC (blocking)** vs **WebFlux (reactive)** vs **MVC + virtual threads**.
- Explains why **DMA + interrupts** mean a few CPU cores can saturate a network card.
- Grounds **thread-pool sizing**: I/O-bound work tolerates many threads (they block); CPU-bound work doesn't.
- Foundation for understanding **connection pools**, async DB drivers (R2DBC), and reactive streams (Phase 16).

---

## 11. Quick Self-Check Questions

1. Why is I/O the central performance concern for backends? (Think latency numbers.)
2. What is **DMA**, and why does it matter for keeping the CPU productive?
3. What role do **interrupts** play in non-blocking I/O?
4. In blocking I/O, what happens to the thread while it waits?
5. What problem does **I/O multiplexing** (epoll/select) solve?
6. Distinguish **readiness-based** from **completion-based** non-blocking I/O.
7. How do **blocking/non-blocking** and **synchronous/asynchronous** differ?
8. How do **virtual threads** give blocking-style code non-blocking scalability?

---

## 12. Key Terms Glossary

- **I/O:** data transfer between CPU/RAM and external devices.
- **I/O-bound / CPU-bound:** workload limited mostly by waiting on I/O vs by computation.
- **Device controller:** hardware that manages a device and exposes registers to the CPU.
- **Polling / Programmed I/O:** CPU repeatedly checks device status (wasteful).
- **Interrupt:** hardware signal notifying the CPU an event occurred.
- **DMA (Direct Memory Access):** hardware moving data device↔RAM without the CPU.
- **System call:** user-space request to the kernel (e.g., `read`, `write`).
- **User space / kernel space:** unprivileged app memory vs privileged OS memory.
- **Blocking I/O:** the calling thread sleeps until the I/O completes.
- **Non-blocking I/O:** the call returns immediately if not ready.
- **I/O multiplexing:** one thread monitoring many connections (`select`/`poll`/`epoll`).
- **epoll / kqueue / IOCP:** scalable OS readiness/completion notification APIs.
- **Event loop:** single-threaded loop dispatching ready I/O events.
- **Readiness vs completion:** "you can do I/O now" vs "I/O is already done."
- **Synchronous / Asynchronous:** result awaited inline vs delivered via callback/future.
- **C10K problem:** the historical challenge of handling 10,000+ concurrent connections.
- **Virtual thread:** JVM-managed thread that parks on blocking I/O (Java 21).

---

*Previous topic: **Context switching**.*
*This completes **Section 0.1 — How Computers Work**.*
*Next section in roadmap: **0.2 Networking Fundamentals → OSI Model**.*
