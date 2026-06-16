# Synchronization, volatile & the Java Memory Model

> **Phase 1 — Java Language Mastery → 1.10 Java Concurrency**
> Goal: Master the core of thread safety — race conditions, `synchronized`, monitors, `volatile`, the Java Memory Model (happens-before), and the hazards: deadlock, livelock, starvation.

---

## 0. The Big Picture

Because threads **share the heap** (Phase 0.1, previous note), two threads touching the same data can corrupt it. **Synchronization** coordinates access so shared mutable state stays correct. This note covers the two core problems — **atomicity** (race conditions, solved by locks) and **visibility** (stale reads, solved by `volatile`/locks) — and the Java Memory Model that governs both.

> Thread safety has **two** independent concerns: (1) **atomicity** — operations that must happen as an indivisible unit; (2) **visibility** — one thread seeing another's writes. You need to handle both.

---

## 1. Race Conditions & Critical Sections

A **race condition** occurs when the correctness of a result depends on the **timing/interleaving** of threads accessing shared data.

### 1.1 The classic example: `count++` is not atomic
```java
class Counter {
    private int count = 0;
    public void increment() { count++; }   // NOT atomic!
}
```
`count++` is actually **three** operations: **read** count → **add 1** → **write** back. Two threads can interleave:
```
count = 0
Thread A: read count (0)
Thread B: read count (0)        <- A hasn't written yet
Thread A: 0 + 1 -> write 1
Thread B: 0 + 1 -> write 1      <- overwrites A's work!
Result: count = 1  (should be 2!)  -> LOST UPDATE
```
> A **critical section** is the code that accesses shared mutable state and must not be executed by multiple threads simultaneously. The fix: make it **mutually exclusive** with synchronization.

---

## 2. The `synchronized` Keyword

`synchronized` enforces **mutual exclusion**: only **one thread at a time** can execute a synchronized block/method on a given **monitor (lock)**.

### 2.1 Synchronized methods
```java
class Counter {
    private int count = 0;
    public synchronized void increment() { count++; }   // locks on 'this'
    public synchronized int get() { return count; }
}
```
- An instance synchronized method locks on **`this`**.
- A static synchronized method locks on the **`Class` object**.

### 2.2 Synchronized blocks (finer-grained — preferred)
```java
class Counter {
    private int count = 0;
    private final Object lock = new Object();   // dedicated lock object

    public void increment() {
        synchronized (lock) {       // only this block is locked
            count++;
        }
    }
}
```
> **Prefer synchronized blocks with a dedicated `private final Object lock`** over synchronizing on `this` — it gives finer control and avoids external code locking on your object. Keep critical sections **as small as possible** (lock contention hurts performance — Phase 0.1 context switching).

### 2.3 Monitors & intrinsic locks
Every Java object has an **intrinsic lock (monitor)** (recall the object header, Phase 1.3). `synchronized` acquires it on entry and releases it on exit (even on exception). This is what `wait()`/`notify()` use (Phase 1.3 Object methods).

### 2.4 Reentrancy
Java's intrinsic locks are **reentrant** — a thread that holds a lock can re-acquire it (e.g., a synchronized method calling another synchronized method on the same object) without deadlocking itself.
```java
synchronized void a() { b(); }      // calling b() while holding the lock is fine
synchronized void b() { ... }       // reentrant — same thread re-acquires
```

---

## 3. The `volatile` Keyword (Visibility)

`synchronized` solves **atomicity**; **`volatile`** solves **visibility**. Without it, one thread's write to a field may **not be seen** by another thread (each CPU core can cache values, and the compiler/JIT can reorder/optimize — recall Phase 0.1 cache, Phase 0.2 cores).

### 3.1 The visibility problem
```java
class Worker {
    private boolean running = true;       // NOT volatile
    public void stop() { running = false; }
    public void run() {
        while (running) { doWork(); }     // may loop FOREVER — never sees running=false!
    }
}
```
Another thread calling `stop()` sets `running = false`, but the running thread may have cached `running = true` and never see the update.

### 3.2 The fix
```java
private volatile boolean running = true;   // writes are immediately visible to all threads
```
`volatile` guarantees:
- **Visibility:** a write is immediately visible to other threads (no stale caches).
- **No reordering** across the volatile access (a memory barrier).

### 3.3 What volatile does NOT do
> ⚠️ **`volatile` does NOT provide atomicity for compound operations.** `volatile int count; count++;` is **still a race condition** (read-modify-write isn't atomic — §1). `volatile` only guarantees visibility of individual reads/writes, not compound atomicity.

| Need | Use |
|------|-----|
| Visibility of a simple flag/reference | **`volatile`** |
| Atomic compound update (`count++`) | **`synchronized`** / `AtomicInteger` (next note) |
| Both | `synchronized` (provides both) |

---

## 4. The Java Memory Model (JMM) & Happens-Before

The **Java Memory Model** defines the rules for **when one thread's writes become visible to another** — and what reorderings the compiler/CPU may perform.

### 4.1 Instruction reordering
For performance, the compiler, JIT, and CPU may **reorder** instructions (as long as single-threaded behavior is preserved). This is invisible single-threaded but can break multi-threaded code without proper synchronization.

### 4.2 The happens-before relationship
**Happens-before** is the JMM's guarantee: if action A *happens-before* action B, then A's effects are **visible** to B. Key happens-before rules:
| Happens-before guarantee |
|--------------------------|
| **Program order:** within one thread, statements happen-before later ones |
| **Monitor lock:** unlocking a monitor happens-before any later locking of the same monitor |
| **volatile:** a write to a volatile happens-before every later read of it |
| **Thread start:** `thread.start()` happens-before the thread's actions |
| **Thread join:** a thread's actions happen-before another's return from `join()` |
| **Transitivity:** if A→B and B→C, then A→C |

> The practical takeaway: **`synchronized` and `volatile` establish happens-before relationships** that guarantee visibility and ordering. Without them, you have **no guarantee** another thread sees your writes — even if your code "looks" correct.

### 4.3 final fields & safe publication
`final` fields (Phase 1.2) have a special JMM guarantee: once a constructor completes, other threads see the correctly-initialized final fields. This is why **immutable objects** (Phase 1.3) are inherently thread-safe — no mutation means no visibility/atomicity issues at all.
> **Immutability is the simplest path to thread safety.** Prefer immutable objects; they need no synchronization.

---

## 5. Concurrency Hazards: Deadlock, Livelock, Starvation

### 5.1 Deadlock
Two (or more) threads each hold a lock the other needs → **both wait forever**.
```java
// Thread 1: synchronized(A) { synchronized(B) { ... } }
// Thread 2: synchronized(B) { synchronized(A) { ... } }
// T1 holds A waits for B; T2 holds B waits for A -> DEADLOCK
```
**Prevention:**
| Strategy | How |
|----------|-----|
| **Lock ordering** | Always acquire locks in the **same global order** |
| **Lock timeout** | Use `tryLock(timeout)` (next note) instead of blocking forever |
| **Minimize lock scope** | Hold fewer locks, for less time |
| **Avoid nested locks** | Don't acquire a second lock while holding the first if avoidable |

### 5.2 Livelock
Threads aren't blocked but keep **responding to each other** and making no progress (e.g., two people stepping aside repeatedly in a hallway). They're active but stuck.

### 5.3 Starvation
A thread **never gets** the CPU or a lock because others monopolize it (e.g., low-priority threads starved by high-priority ones, or unfair locks). Fairness policies (next note) mitigate this.

| Hazard | What's happening |
|--------|------------------|
| **Deadlock** | Everyone blocked, waiting on each other forever |
| **Livelock** | Active but making no progress |
| **Starvation** | A thread perpetually denied resources |

---

## 6. wait() / notify() / notifyAll() (recap from Phase 1.3)

The low-level thread-coordination primitives (must hold the monitor; wait in a loop):
```java
synchronized (lock) {
    while (!condition) lock.wait();   // releases lock & waits; loop guards spurious wakeups
    // proceed
}
synchronized (lock) {
    condition = true;
    lock.notifyAll();                 // wake waiting threads
}
```
> These are **low-level**. In modern code, prefer higher-level tools: `BlockingQueue`, `CountDownLatch`, `Semaphore`, `Condition`, `CompletableFuture` (later notes) — they're safer and clearer.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Thinking `count++` is atomic | It's read-modify-write — use `synchronized`/`Atomic*` |
| Using `volatile` for compound atomicity | `volatile` is visibility only |
| Synchronizing on `this` or a mutable/`String` lock | Use a `private final Object lock` |
| Holding locks too long / doing I/O in a lock | Keep critical sections tiny |
| Assuming writes are visible without sync | Establish happens-before (`volatile`/`synchronized`) |
| Inconsistent lock ordering | Acquire locks in a fixed global order (deadlock prevention) |
| Using `notify()` when several conditions exist | Prefer `notifyAll()`; always `wait()` in a loop |
| Mutable shared state everywhere | Prefer immutability / confinement |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Singleton beans are shared across request threads** — any mutable field needs thread safety (or, better, be stateless/immutable). A top Spring concurrency bug.
- **`volatile`/`synchronized`** appear in caches, lazy initialization, and flags.
- **Immutability** (records, Phase 1.3) is the safest design for shared DTOs/value objects.
- **Deadlocks** happen in DB transactions and lock-ordering issues (Phase 4.5) — same prevention principles.
- **Visibility bugs** are subtle and intermittent — understanding the JMM helps you reason about correctness.
- Higher-level tools (next notes) usually replace raw `synchronized`/`wait` in real code.

---

## 9. Quick Self-Check Questions

1. Why is `count++` not atomic, and what's a race condition?
2. What is a critical section, and how does `synchronized` protect it?
3. What does an instance vs static synchronized method lock on?
4. Why prefer a dedicated lock object over synchronizing on `this`?
5. What problem does `volatile` solve, and what does it NOT solve?
6. What is the happens-before relationship, and name three things that establish it?
7. Why are immutable objects inherently thread-safe?
8. Define deadlock, livelock, and starvation. How do you prevent deadlock?

---

## 10. Key Terms Glossary

- **Race condition:** result depends on thread interleaving.
- **Critical section:** code accessing shared state that needs exclusivity.
- **Atomicity / visibility:** indivisible operation / seeing others' writes.
- **`synchronized`:** mutual-exclusion via an object's monitor lock.
- **Monitor / intrinsic lock:** the per-object lock used by `synchronized`.
- **Reentrant:** a thread can re-acquire a lock it already holds.
- **`volatile`:** guarantees visibility & ordering (not atomicity) of a field.
- **Java Memory Model (JMM):** rules for visibility and reordering.
- **Happens-before:** guarantee that one action's effects are visible to another.
- **Instruction reordering:** compiler/CPU reordering for performance.
- **Deadlock / livelock / starvation:** mutual blocking / active no-progress / perpetual denial.
- **Safe publication:** correctly making an object visible to other threads.

---

*Previous topic: **Concurrency Fundamentals & Thread Basics**.*
*Next topic: **Locks, Atomics & Concurrent Collections**.*
