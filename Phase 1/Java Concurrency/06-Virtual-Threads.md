# Virtual Threads (Project Loom, Java 21)

> **Phase 1 — Java Language Mastery → 1.10 Java Concurrency**
> Goal: Master virtual threads — platform vs virtual threads, how they enable massive concurrency, structured concurrency, pinning issues, and their impact on backend applications.

---

## 0. The Big Picture

**Virtual threads** (finalized in **Java 21**, Project Loom) are **lightweight threads managed by the JVM**, not the OS. They make it cheap to have **millions** of threads — letting you write **simple, blocking-style code** that scales like complex non-blocking/reactive code.

```java
// Looks blocking and simple — but scales to millions of concurrent tasks:
Thread.ofVirtual().start(() -> {
    var response = httpClient.send(request);   // "blocks" — but cheaply!
    process(response);
});
```

> This is one of the **biggest changes** to Java concurrency in years. It directly addresses the I/O-bound nature of backends (recall Phase 0.1: blocking vs non-blocking I/O, the C10K problem).

---

## 1. The Problem Virtual Threads Solve

### 1.1 Platform threads are expensive
A traditional Java thread (**platform thread**) maps **1:1 to an OS thread** (recall Phase 0.1). Each one:
- Costs ~**1 MB** of stack memory.
- Is **limited** — a machine can handle maybe a few thousand before resources/context-switching (Phase 0.1) become the bottleneck.

So the **thread-per-request** model (one thread per HTTP request) caps your concurrency at a few thousand requests — even though most threads are just **blocked waiting on I/O** (DB, network) doing nothing.

### 1.2 The old workarounds and their cost
| Approach | Problem |
|----------|---------|
| **Thread pools** (bounded) | Limited concurrency; requests queue when pool is full |
| **Reactive/async** (WebFlux, Phase 16) | Scales, but **complex** — callbacks, no blocking, hard to debug, "colored functions" |

Virtual threads give you the **scalability of async** with the **simplicity of blocking code**.

---

## 2. Platform Threads vs Virtual Threads

| | **Platform thread** | **Virtual thread** |
|---|---------------------|--------------------|
| Backed by | 1 OS thread (1:1) | Managed by the JVM (M:N) |
| Memory | ~1 MB stack | ~few KB (grows on demand) |
| Max count | Thousands | **Millions** |
| Creation cost | Expensive | Very cheap |
| Blocking I/O | Blocks the OS thread (wasteful) | **Unmounts** — frees the OS thread |
| Best for | CPU-bound work | **I/O-bound** work (most backends) |

### 2.1 How virtual threads work (mounting/unmounting)
Virtual threads run on a small pool of **carrier threads** (platform/OS threads). When a virtual thread **blocks on I/O**, the JVM **unmounts** it from its carrier and runs another virtual thread there — the OS thread is **never actually blocked**.
```
Many virtual threads  ->  multiplexed onto  ->  few carrier (OS) threads
When a virtual thread blocks on I/O:
   JVM parks it, frees the carrier thread to run another virtual thread
When the I/O completes:
   the virtual thread is remounted on a carrier and resumes
```
> Under the hood, the JDK rewrote blocking operations (sockets, files, `sleep`, locks) to **park the virtual thread** instead of the OS thread — using non-blocking I/O (epoll, Phase 0.1) internally. Your code looks blocking; the runtime makes it non-blocking. (Recall Phase 0.1's context-switching note: a virtual-thread switch is a cheap **user-mode** park, not an OS context switch.)

---

## 3. Creating Virtual Threads

```java
// Start one directly:
Thread.ofVirtual().start(() -> doWork());

// Named:
Thread vt = Thread.ofVirtual().name("vt-1").unstarted(() -> doWork());
vt.start();

// The recommended way — an executor that creates one virtual thread PER TASK:
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> {
            var data = fetchFromDatabase();   // blocking call — but cheap on a virtual thread
            return process(data);
        });
    }
}   // try-with-resources: executor closes & waits for all tasks
```
> **Key mindset shift:** with virtual threads you **don't pool them** — you create **one per task** (even millions). Pooling virtual threads is an anti-pattern (they're cheap; pooling defeats the purpose). Pools were for *expensive* platform threads.

---

## 4. Structured Concurrency (Java 21 preview)

**Structured concurrency** treats a group of related concurrent tasks as a **single unit of work** — if one fails or the scope is cancelled, the others are cancelled too. It brings structure (like try-with-resources) to concurrency.
```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Supplier<User>   user   = scope.fork(() -> fetchUser(id));      // subtask 1
    Supplier<Orders> orders = scope.fork(() -> fetchOrders(id));    // subtask 2

    scope.join();                  // wait for both
    scope.throwIfFailed();         // propagate the first failure (cancels the other)

    return new Result(user.get(), orders.get());   // both succeeded
}   // scope closes -> all subtasks guaranteed done/cancelled
```
- If one subtask fails, the others are **automatically cancelled** (no orphaned threads).
- Clear parent-child relationship → better error handling, cancellation, and observability.
> Structured concurrency + virtual threads = clean, reliable fan-out (call several services in parallel, all-or-nothing). It avoids leaked tasks and tangled error handling.

---

## 5. Pinning (the Main Gotcha)

A virtual thread is normally **unmounted** when it blocks. But in two cases it gets **"pinned"** — stuck on its carrier thread, unable to unmount, blocking the OS thread (defeating the benefit):
| Pinning cause | Why |
|---------------|-----|
| **`synchronized` block/method that blocks** | Holding a monitor while blocking pins the carrier |
| **Native code / JNI** that blocks | The runtime can't unmount across native frames |

### 5.1 Avoiding pinning
> ⚠️ If you block (I/O) **inside a `synchronized` block**, the virtual thread **pins** its carrier thread — under load this can exhaust carriers and stall everything. **Fix:** replace `synchronized` with **`ReentrantLock`** (previous note) around blocking sections — `ReentrantLock` doesn't pin.
```java
// Pins if I/O blocks inside:
synchronized (lock) { blockingIoCall(); }    // ❌ may pin the carrier

// Doesn't pin:
lock.lock();
try { blockingIoCall(); } finally { lock.unlock(); }   // ✅ ReentrantLock
```
> (Future Java versions aim to remove the `synchronized` pinning limitation, but for Java 21, prefer `ReentrantLock` around blocking code.)

---

## 6. Impact on Backend Applications

### 6.1 The thread-per-request model is viable again
With platform threads, thread-per-request capped at a few thousand concurrent requests. With virtual threads, you can have **a virtual thread per request** and handle **hundreds of thousands** — using **simple blocking code** (no reactive complexity).

### 6.2 Virtual threads vs reactive (WebFlux)
| | **Virtual threads** | **Reactive (WebFlux)** |
|---|---------------------|------------------------|
| Code style | Simple, blocking, sequential | Async, callbacks/operators |
| Learning curve | Low (normal Java) | High (Reactor, backpressure) |
| Debugging | Easy (normal stack traces) | Hard (broken stack traces) |
| Scales I/O-bound? | ✅ Yes | ✅ Yes |
| Backpressure | Manual | Built-in |
| When | Most I/O-bound services | Streaming, extreme throughput, backpressure needs |
> For most I/O-bound backends, **virtual threads let you keep simple blocking code and still scale** — reducing the need to go fully reactive. Reactive still wins for streaming and fine-grained backpressure (Phase 16).

### 6.3 Spring support
Spring Boot 3.2+ supports virtual threads via a single property:
```properties
spring.threads.virtual.enabled=true   # Tomcat handles each request on a virtual thread
```
> This makes a traditional Spring MVC app scale like a reactive one for I/O-bound workloads, with zero code changes.

### 6.4 When virtual threads DON'T help
- **CPU-bound** work: virtual threads don't add CPU; use a bounded pool sized to cores (previous note). Virtual threads help when threads **wait** (I/O), not when they **compute**.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| **Pooling** virtual threads | Don't — create one per task (they're cheap) |
| Blocking I/O inside `synchronized` (pinning) | Use `ReentrantLock` around blocking code |
| Using virtual threads for CPU-bound work | Use a platform-thread pool sized to cores |
| `ThreadLocal` overuse with millions of vthreads | Heavy ThreadLocals × millions = memory; clean up / prefer scoped values |
| Expecting them to speed up computation | They scale waiting, not computing |
| Thinking they replace reactive entirely | Reactive still wins for streaming/backpressure |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Thread-per-request at scale (Phase 5, 13):** `spring.threads.virtual.enabled=true` lets Spring MVC handle massive concurrency with simple code.
- **Simplifies I/O-bound services:** call DBs and other services with normal blocking code that scales (Phase 12).
- **Reduces the need for reactive (Phase 16)** in many cases — lower complexity for the same scalability.
- **Structured concurrency** is the clean way to fan out to multiple services and aggregate (Phase 12) with proper cancellation.
- **Pinning awareness** matters when integrating legacy `synchronized` code or native libraries.
- Ties together the whole section: virtual threads make the **blocking model from Phase 0.1** scale like the **non-blocking model** — best of both worlds.

---

## 9. Quick Self-Check Questions

1. What problem do virtual threads solve, and how do they relate to the C10K problem?
2. What's the difference between a platform thread and a virtual thread?
3. What happens when a virtual thread blocks on I/O (mounting/unmounting)?
4. Why should you NOT pool virtual threads?
5. What is structured concurrency, and what does it guarantee?
6. What is pinning, what causes it, and how do you avoid it?
7. Compare virtual threads vs reactive programming for an I/O-bound service.
8. When do virtual threads NOT help?
9. How do you enable virtual threads in Spring Boot?

---

## 10. Key Terms Glossary

- **Virtual thread:** a lightweight JVM-managed thread (Java 21).
- **Platform thread:** a traditional thread backed 1:1 by an OS thread.
- **Carrier thread:** the OS thread that runs (carries) virtual threads.
- **Mounting / unmounting:** attaching/detaching a virtual thread to/from a carrier.
- **Parking:** suspending a virtual thread cheaply (no OS context switch).
- **`Thread.ofVirtual` / `newVirtualThreadPerTaskExecutor`:** create virtual threads.
- **Structured concurrency:** treating concurrent subtasks as one unit (`StructuredTaskScope`).
- **Pinning:** a virtual thread stuck on its carrier (e.g., blocking in `synchronized`).
- **Thread-per-request:** one thread (now virtual) per HTTP request.
- **Project Loom:** the OpenJDK project that delivered virtual threads.

---

*Previous topic: **CompletableFuture, Synchronizers & ThreadLocal**.*
*This completes **Section 1.10 — Java Concurrency**.*
*Next section in roadmap: **1.11 JVM Deep Dive**.*
