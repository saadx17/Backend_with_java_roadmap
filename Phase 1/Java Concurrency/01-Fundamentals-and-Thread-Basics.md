# Concurrency Fundamentals & Thread Basics

> **Phase 1 — Java Language Mastery → 1.10 Java Concurrency**
> Goal: Understand concurrency vs parallelism, processes vs threads, and the basics of creating and managing threads (lifecycle, key methods, daemon threads, uncaught exceptions).

---

## 0. The Big Picture

**Concurrency** is doing multiple things in overlapping time periods. For backend engineers it's essential: a server handles **many requests at once**, processes data in parallel, and runs background tasks — all via threads.

```
One request handler thread per request (classic model):
  [Thread 1] -> handle request A
  [Thread 2] -> handle request B   } all running "at the same time"
  [Thread 3] -> handle request C
```

> This section is large and critical. It builds directly on **Phase 0.1** (process vs thread, context switching, shared heap) — re-skim that note: the shared heap is *why* concurrency is hard.

---

## 1. Concurrency vs Parallelism

| | **Concurrency** | **Parallelism** |
|---|-----------------|-----------------|
| Definition | Multiple tasks **in progress** over the same period (interleaved) | Multiple tasks **executing simultaneously** |
| Hardware | Possible on **1 core** (time-slicing) | Requires **multiple cores** |
| About | **Structure** (dealing with many things) | **Execution** (doing many things at once) |

```
Concurrency (1 core):  T1-- T2-- T1-- T2--   (interleaved via context switching)
Parallelism (2 cores): Core0: T1======
                       Core1: T2======       (truly simultaneous)
```
> (Recall Phase 0.1.) Concurrency is the broader concept; parallelism is concurrency *plus* simultaneous execution on multiple cores. Threads enable both.

---

## 2. Processes vs Threads (recap from Phase 0.1)

| | **Process** | **Thread** |
|---|-------------|------------|
| Memory | Own isolated address space | **Shares** the process's heap |
| Creation cost | Heavy | Light |
| Communication | IPC (complex) | Shared memory (direct, but needs sync) |
| Crash impact | Isolated | Can affect the whole process |

**The crucial fact for this whole section:**
> Threads in a JVM **share the heap** (objects, static fields) but each has its **own stack** (local variables). Shared mutable state → **race conditions** → why we need synchronization. Local variables are naturally thread-safe.

---

## 3. Creating Threads

### 3.1 Three ways

**1. Implement `Runnable` (preferred — favors composition):**
```java
Runnable task = () -> System.out.println("running on " + Thread.currentThread().getName());
Thread t = new Thread(task);
t.start();          // start() runs run() on a NEW thread
```

**2. Extend `Thread` (avoid — ties you to inheritance):**
```java
class MyThread extends Thread {
    public void run() { System.out.println("running"); }
}
new MyThread().start();
```

**3. `Callable` + executor (returns a result — see executors note):**
```java
Callable<Integer> task = () -> 42;   // returns a value, can throw checked exceptions
```

> **Prefer `Runnable`/`Callable` + an executor** over extending `Thread` (composition over inheritance — Phase 1.3). In modern code you almost never create raw `Thread`s directly; you submit tasks to a **thread pool** (executors note).

### 3.2 `start()` vs `run()` (a classic gotcha)
```java
t.start();   // creates a NEW thread, then calls run() on it  ✅
t.run();     // just calls run() on the CURRENT thread — NO new thread!  ❌
```
> **Always call `start()`**, not `run()`. Calling `run()` directly executes the code on the current thread (no concurrency) — a very common beginner mistake.

---

## 4. Thread Lifecycle (States)

A Java thread moves through these states (`Thread.State` enum):
```
NEW  --start()-->  RUNNABLE  <-->  RUNNING (on a core)
                      |  ^
         (lock/wait)  |  | (lock acquired / notified / timeout)
                      v  |
   BLOCKED / WAITING / TIMED_WAITING
                      |
                  TERMINATED (run() finished)
```
| State | Meaning |
|-------|---------|
| **NEW** | Created but not started |
| **RUNNABLE** | Ready to run / running (the scheduler decides; recall Phase 0.1) |
| **BLOCKED** | Waiting to acquire a monitor lock (`synchronized`) |
| **WAITING** | Waiting indefinitely (`wait()`, `join()`, `park()`) |
| **TIMED_WAITING** | Waiting with a timeout (`sleep(n)`, `wait(n)`) |
| **TERMINATED** | Finished (run() returned or threw) |

> (Compare to Phase 0.1's OS-level process/thread states — same idea, JVM-level.)

---

## 5. Key Thread Methods

| Method | Effect |
|--------|--------|
| `start()` | Begin execution on a new thread |
| `Thread.sleep(ms)` | Pause the **current** thread (TIMED_WAITING); keeps locks! |
| `join()` | Wait for **another** thread to finish |
| `interrupt()` | Request the thread to stop (sets its interrupt flag) |
| `Thread.currentThread()` | Reference to the running thread |
| `isAlive()` | Is the thread still running? |
| `setName()` / `getName()` | Name (useful in logs/debugging) |

### 5.1 join — wait for completion
```java
Thread worker = new Thread(task);
worker.start();
worker.join();        // current thread blocks until 'worker' finishes
System.out.println("worker done");
```

### 5.2 sleep — pause (and a caution)
```java
Thread.sleep(1000);   // pause ~1 second
```
> ⚠️ `sleep()` **does not release locks** it holds — sleeping inside a `synchronized` block keeps everyone else blocked. `wait()` (next note) *does* release the lock. Don't confuse them.

### 5.3 Interruption (cooperative cancellation)
Java threads can't be forcibly killed safely. Instead, `interrupt()` sets a flag; well-behaved code **checks** it and stops cooperatively:
```java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {   // check the flag
        doWork();
    }
});
worker.start();
worker.interrupt();   // request stop (sets the flag)
```
Blocking methods (`sleep`, `wait`, `join`) throw **`InterruptedException`** when interrupted:
```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // RESTORE the flag (best practice!)
    return;                               // then stop gracefully
}
```
> ⚠️ **Never swallow `InterruptedException`** (recall Phase 1.5). Either propagate it or **restore the interrupt flag** (`Thread.currentThread().interrupt()`) so callers know cancellation was requested. Swallowing it breaks cancellation.

---

## 6. Daemon Threads

A **daemon thread** is a background thread that **doesn't prevent the JVM from exiting**. When all *non-daemon* (user) threads finish, the JVM shuts down — killing any remaining daemon threads.
```java
Thread t = new Thread(backgroundTask);
t.setDaemon(true);    // must set BEFORE start()
t.start();
```
| | User (non-daemon) thread | Daemon thread |
|---|--------------------------|----------------|
| Keeps JVM alive? | ✅ Yes | ❌ No |
| Use for | Main work | Background services (GC, monitoring, cleanup) |

> Examples: the **garbage collector** runs on daemon threads. Use daemon threads for background tasks that should die with the app — but **not** for work that must complete (it could be killed mid-task).

---

## 7. UncaughtExceptionHandler

If a thread throws an exception that isn't caught, the thread **terminates** and (by default) prints the stack trace. You can install a handler:
```java
Thread t = new Thread(task);
t.setUncaughtExceptionHandler((thread, ex) ->
    log.error("Thread {} died", thread.getName(), ex));
t.start();

// Global default for all threads:
Thread.setDefaultUncaughtExceptionHandler((thread, ex) -> log.error("Uncaught", ex));
```
> An uncaught exception kills only **that** thread, not the JVM (unless it's the only/main thread). In thread pools, uncaught exceptions can silently kill pool threads — handle them (executors note).

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Calling `run()` instead of `start()` | Use `start()` to get a new thread |
| Confusing concurrency with parallelism | Concurrency = structure; parallelism = simultaneous execution |
| Forgetting threads share the heap | Shared mutable state needs synchronization (next note) |
| `sleep()` expecting it to release locks | It doesn't; `wait()` does |
| Swallowing `InterruptedException` | Restore the flag or propagate |
| Trying to forcibly kill a thread (`stop()`) | Deprecated/unsafe; use cooperative interruption |
| Creating raw threads everywhere | Use executors/thread pools (next notes) |
| Daemon thread for must-finish work | It may be killed on JVM exit |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Every web request** runs on a thread (Tomcat thread pool) — concurrency is the default reality of a backend.
- **Shared state** (singletons, caches, static fields) must be thread-safe (next notes) — a top source of production bugs.
- **`@Async`, `@Scheduled`** (Phase 5.8) run on background threads.
- **Graceful shutdown** (Phase 0.3 SIGTERM, Phase 10) relies on interruption/cancellation to drain in-flight work.
- **Thread pools** (executors note) back the entire request-handling model — sizing them matters (Phase 13).
- **Virtual threads** (last note) change the economics of thread-per-request.

---

## 10. Quick Self-Check Questions

1. What's the difference between concurrency and parallelism?
2. Why is shared mutable state the central challenge of concurrency?
3. What are the three ways to create a thread, and which is preferred?
4. What's the difference between `start()` and `run()`?
5. List the thread lifecycle states.
6. What's the difference between `sleep()` and `wait()` regarding locks?
7. How does cooperative interruption work, and how should you handle `InterruptedException`?
8. What is a daemon thread, and when would you use one?
9. What happens when a thread throws an uncaught exception?

---

## 11. Key Terms Glossary

- **Concurrency / parallelism:** overlapping tasks / simultaneous execution.
- **Thread:** an independent path of execution sharing the process heap.
- **`Runnable` / `Callable`:** task (no result) / task returning a value.
- **`start()` vs `run()`:** new thread vs current thread.
- **Thread lifecycle:** NEW → RUNNABLE → (BLOCKED/WAITING/TIMED_WAITING) → TERMINATED.
- **`join()`:** wait for a thread to finish.
- **`sleep()`:** pause the current thread (keeps locks).
- **Interruption / `InterruptedException`:** cooperative cancellation mechanism.
- **Daemon thread:** background thread that doesn't keep the JVM alive.
- **`UncaughtExceptionHandler`:** handles exceptions that escape a thread.

---

*This is the first note of **Section 1.10 — Java Concurrency**.*
*Next topic: **Synchronization, volatile & the Java Memory Model**.*
