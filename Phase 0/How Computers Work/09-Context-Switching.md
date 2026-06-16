# Context Switching

> **Phase 0 — Computer Science Fundamentals → 0.1 How Computers Work**
> Goal: Understand how the OS makes one CPU core appear to run many tasks at once — by saving and restoring execution state — and why this has a real performance cost.

---

## 0. The One-Sentence Summary

A **context switch** is the act of the OS **saving the state of the currently running task** (process or thread) and **loading the saved state of another**, so a single CPU core can be shared among many tasks. It creates the *illusion* of simultaneous execution but is **pure overhead** — no useful work happens during the switch itself.

---

## 1. The Core Problem It Solves

A CPU core runs **exactly one thread at a time**. But systems have hundreds or thousands of threads/processes "running." The OS solves this by **time-sharing**: give each task a tiny slice of CPU time, then rapidly switch to the next.

```
One core, many tasks (over time):
  | T1 | T2 | T3 | T1 | T2 | T3 | ...   (switches happen between slices)
   ^^^^      ^^^^
   time slice  context switch in between (overhead)
```

Switching fast enough (thousands of times per second) makes it *look* like everything runs at once → this is **concurrency** (recall: concurrency vs parallelism from the Process vs Thread note).

---

## 2. What is "Context"?

The **context** is everything the CPU needs to resume a task exactly where it left off — its complete execution state:

| Saved state | Why it's needed |
|-------------|-----------------|
| **Program Counter (PC / instruction pointer)** | Which instruction to run next |
| **CPU registers** (general purpose) | In-progress computation values |
| **Stack Pointer (SP)** | Top of this task's stack |
| **Status / flag registers** | Condition flags, CPU mode |
| **Memory mapping info** (for processes) | Which page table / address space to use |

This saved state is stored in an OS data structure:
- **PCB (Process Control Block)** — per process
- **TCB (Thread Control Block)** — per thread

---

## 3. The Context Switch Sequence

```
Task A is running on the core
        |
   Trigger occurs (timer interrupt, blocking call, higher-priority task, etc.)
        |
        v
1. CPU traps into the OS kernel (switch to kernel mode)
2. SAVE Task A's context  -> into A's PCB/TCB
3. Scheduler picks the next task (Task B)
4. LOAD Task B's context  <- from B's PCB/TCB
   (for a process switch: also switch address space + flush/tag TLB)
5. Return to user mode, resume Task B at its saved PC
        |
        v
Task B is now running on the core
```

Steps 2 and 4 are the actual "switch." Steps 1 and 5 involve a **mode switch** (user ↔ kernel).

> **Mode switch ≠ context switch.** A mode switch (user→kernel, e.g., a system call) changes privilege level but stays in the *same* task. A context switch changes *which task* runs. A context switch usually involves mode switches, but not vice versa.

---

## 4. What Triggers a Context Switch?

| Trigger | Type | Example |
|---------|------|---------|
| **Timer interrupt (time slice expires)** | Involuntary (preemption) | OS scheduler forces a switch after a quantum |
| **Blocking I/O / system call** | Voluntary | Thread waits for disk/network → yields the CPU |
| **Waiting on a lock / sleep / wait()** | Voluntary | Thread blocks until a resource is free |
| **Higher-priority task becomes ready** | Involuntary (preemption) | Scheduler preempts the current task |
| **Interrupt from hardware** | Involuntary | Device signals completion |
| **Task finishes / exits** | Voluntary | Nothing left to run → switch to another |

- **Preemptive scheduling:** OS can forcibly interrupt a running task (modern OSes).
- **Cooperative scheduling:** tasks must voluntarily yield (older/embedded systems).

---

## 5. Process Switch vs Thread Switch (the cost difference)

Not all context switches cost the same — this is a key insight from the **Process vs Thread** and **Virtual Memory** notes:

| | Thread → Thread (same process) | Process → Process |
|---|---|---|
| Save/restore registers + PC + SP | Yes | Yes |
| Switch **address space** (page tables) | **No** (shared) | **Yes** |
| **Flush / reload TLB** | Mostly no | **Yes** (expensive) |
| Cold CPU caches afterward | Some | More (different working set) |
| Relative cost | **Cheaper** | **More expensive** |

> A **process switch** must change the virtual address space, which typically forces a **TLB flush** (the cache of virtual→physical translations). After the switch, translations and CPU caches are "cold," causing extra misses. A **thread switch within the same process** keeps the address space, so it skips this — that's why threads are cheaper to switch between.

---

## 6. The Hidden Costs (why context switching hurts performance)

The visible cost is saving/loading registers, but the **indirect costs are usually larger**:

| Cost | Explanation |
|------|-------------|
| **Direct cost** | Cycles spent saving/restoring registers, running the scheduler |
| **TLB flush** (process switch) | Subsequent memory accesses suffer TLB misses → page-table walks |
| **Cache pollution** | New task evicts the old task's data from CPU caches (L1/L2/L3) → cache misses when the old task resumes |
| **Pipeline flush** | CPU instruction pipeline must be cleared and refilled |
| **Scheduler overhead** | Time spent deciding what to run next |

> **No useful application work happens during a context switch** — it's pure overhead. The goal is to switch *enough* for responsiveness but not so much that overhead dominates.

---

## 7. The Danger of Too Many Threads

If you create far more runnable threads than CPU cores, the OS spends excessive time switching between them rather than doing work:

```
Few threads  : | work | work | work |        (high throughput)
Too many     : |sw| w |sw| w |sw| w |sw|      (overhead eats into work)
               ^^ context switches everywhere
```

This is **CPU thrashing** (analogous to memory thrashing from the paging note). Symptoms: high CPU usage with low actual throughput, rising context-switch counts.

**Practical takeaway:** for **CPU-bound** work, the optimal number of threads ≈ number of cores. More threads only helps when threads spend time **blocked** (e.g., I/O-bound work), since blocked threads aren't competing for the CPU.

---

## 8. Context Switching and Java

- Each Java **platform thread** maps to an OS thread, so every Java thread switch is an **OS context switch** with all the costs above.
- **Thread pool sizing** (e.g., `ThreadPoolExecutor`, Phase 1.10 / Phase 13) is fundamentally about *minimizing wasteful context switches*:
  - CPU-bound pool size ≈ `number of cores`.
  - I/O-bound pools can be larger (threads block, freeing the CPU).
- **Lock contention** causes extra switches: threads blocked on a lock get switched out and back in repeatedly → a hidden performance killer.
- **Virtual threads (Java 21, Project Loom)** dramatically reduce this cost for I/O-bound workloads:
  - When a virtual thread blocks (e.g., on I/O), the JVM **parks** it and runs another virtual thread on the same OS (carrier) thread — a cheap **user-mode switch**, not a full OS context switch.
  - This is why you can have *millions* of virtual threads without the context-switch storm that millions of platform threads would cause.

```
Platform threads:  block -> OS context switch (expensive, kernel involved)
Virtual threads :  block -> JVM parks/unmounts (cheap, user-mode)
```

---

## 9. Why This Matters for a Backend (Java) Engineer

- Explains **why thread pools exist** and how to size them (Phase 13 performance tuning).
- Explains why **too many threads slows a server down** instead of speeding it up.
- Explains the real cost of **lock contention** and blocking — not just waiting, but extra switching.
- Justifies the design of **virtual threads** and reactive/non-blocking I/O for high-concurrency services.
- Helps you read OS metrics: a spiking **context-switch rate** (`vmstat`, `pidstat`) is a red flag for thread thrashing or excessive contention.

---

## 10. Quick Self-Check Questions

1. What exactly is saved and restored during a context switch?
2. Where is a task's context stored (which OS structures)?
3. Name three things that can trigger a context switch.
4. What's the difference between a **mode switch** and a **context switch**?
5. Why is a process→process switch more expensive than a thread→thread switch?
6. Besides saving/restoring registers, what are the *hidden* costs of a context switch?
7. Why does creating far more threads than cores often *reduce* throughput?
8. How do virtual threads reduce context-switching cost for I/O-bound work?

---

## 11. Key Terms Glossary

- **Context switch:** saving one task's execution state and loading another's.
- **Context:** the full CPU state needed to resume a task (PC, registers, SP, flags, mapping).
- **PCB / TCB:** Process / Thread Control Block — where context is stored.
- **Time slice / quantum:** the CPU time a task gets before being preempted.
- **Preemption:** OS forcibly interrupting a running task.
- **Preemptive vs cooperative scheduling:** OS-forced vs voluntary yielding.
- **Mode switch:** changing privilege level (user ↔ kernel) within the same task.
- **TLB flush:** invalidating cached address translations (on process switch).
- **Cache pollution:** a new task evicting another's data from CPU caches.
- **CPU thrashing:** excessive switching where overhead dominates useful work.
- **Carrier thread:** an OS thread that runs (carries) virtual threads in Java.
- **Parking:** suspending a (virtual) thread cheaply without an OS context switch.

---

*Previous topic: **Virtual memory and paging**.*
*Next topic in roadmap: **I/O operations (blocking vs non-blocking at hardware level)**.*
