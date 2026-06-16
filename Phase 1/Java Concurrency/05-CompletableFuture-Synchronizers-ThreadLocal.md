# CompletableFuture, Synchronizers & ThreadLocal

> **Phase 1 — Java Language Mastery → 1.10 Java Concurrency**
> Goal: Master asynchronous composition with `CompletableFuture`, coordination with synchronizers (latches, barriers, semaphores), and per-thread state with `ThreadLocal`.

---

## 0. The Big Picture

`Future` (previous note) lets you get a result later — but `get()` **blocks**, and you can't easily chain or combine futures. **`CompletableFuture`** fixes this: it enables **non-blocking, composable** async pipelines. This note also covers **synchronizers** (tools to coordinate threads) and **`ThreadLocal`** (per-thread storage).

```java
CompletableFuture
    .supplyAsync(() -> fetchUser(id))          // run async
    .thenApply(User::getName)                   // transform when done (non-blocking)
    .thenAccept(System.out::println)            // consume the result
    .exceptionally(ex -> { log.error("", ex); return null; });  // handle errors
```

---

## 1. CompletableFuture — Async Composition

A `CompletableFuture<T>` represents an async computation you can **chain, combine, and handle errors on** — without blocking (recall functional programming, Phase 1.8: it's all lambdas/method references).

### 1.1 Creating async tasks
```java
// Run a task that returns a value (on the common ForkJoinPool, or a given executor):
CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> compute());
CompletableFuture<Integer> cf2 = CompletableFuture.supplyAsync(() -> compute(), myExecutor);

// Run a task with no result:
CompletableFuture<Void> cf3 = CompletableFuture.runAsync(() -> doWork());

// An already-completed future:
CompletableFuture<String> done = CompletableFuture.completedFuture("ready");
```
> ⚠️ By default, `*Async` methods run on the **shared ForkJoinPool.commonPool**. For **blocking I/O**, pass your **own executor** so you don't starve the common pool (recall Phase 0.1/1.8 — never block the shared pool).

### 1.2 Transforming results (the chaining methods)
| Method | Takes | Returns | Use |
|--------|-------|---------|-----|
| `thenApply(fn)` | `Function` | `CF<R>` | transform the result |
| `thenAccept(consumer)` | `Consumer` | `CF<Void>` | consume the result |
| `thenRun(runnable)` | `Runnable` | `CF<Void>` | run after, ignoring the result |
| `thenCompose(fn)` | `Function -> CF` | `CF<R>` | **flatMap** — chain a dependent future |
| `thenCombine(other, fn)` | another CF + BiFunction | `CF<R>` | combine two independent futures |

```java
CompletableFuture.supplyAsync(() -> fetchUser(id))      // CF<User>
    .thenApply(User::getEmail)                           // CF<String> (transform)
    .thenAccept(email -> sendNotification(email));       // CF<Void> (consume)
```

### 1.3 thenApply vs thenCompose (map vs flatMap — recall Phase 1.8)
```java
// thenApply: function returns a plain value -> nested if it returned a CF
cf.thenApply(user -> user.getName());                    // CF<String>

// thenCompose: function returns ANOTHER CompletableFuture -> flattened (avoids CF<CF<T>>)
cf.thenCompose(user -> fetchOrdersAsync(user.getId()));  // CF<List<Order>>, not CF<CF<...>>
```
> Use **`thenCompose`** when the next step is *itself* async (returns a `CompletableFuture`) — same map-vs-flatMap distinction as streams/Optional (Phase 1.8).

### 1.4 Combining multiple futures
```java
CompletableFuture<Integer> a = CompletableFuture.supplyAsync(() -> 10);
CompletableFuture<Integer> b = CompletableFuture.supplyAsync(() -> 20);

a.thenCombine(b, (x, y) -> x + y);          // combine two -> 30

// Wait for ALL / ANY:
CompletableFuture.allOf(a, b, c).join();    // wait for all to complete
CompletableFuture.anyOf(a, b, c).join();    // wait for the first to complete
```
> `allOf` is the idiomatic way to **fan out** many async calls and wait for all (e.g., call 5 services in parallel, then aggregate).

### 1.5 Error handling
| Method | Behavior |
|--------|----------|
| `exceptionally(fn)` | Recover from an exception with a fallback value |
| `handle(biFn)` | Handle both result and exception (always runs) |
| `whenComplete(biConsumer)` | Side-effect on completion (result or exception) |

```java
CompletableFuture.supplyAsync(() -> riskyCall())
    .exceptionally(ex -> "fallback")               // on error -> fallback value
    .handle((result, ex) -> ex != null ? "error" : result);  // both cases
```

### 1.6 Timeouts (Java 9+)
```java
cf.orTimeout(2, TimeUnit.SECONDS);                          // fail with TimeoutException
cf.completeOnTimeout("default", 2, TimeUnit.SECONDS);       // fall back to a value
```

### 1.7 Getting the result
```java
cf.get();      // blocks; throws checked ExecutionException/InterruptedException
cf.join();     // blocks; throws unchecked CompletionException (nicer in pipelines)
```
> Prefer building the **whole pipeline** (then... / exceptionally) and only block (`join`) at the very end, if at all — that's the point of `CompletableFuture` (non-blocking composition).

---

## 2. Synchronizers (Coordinating Threads)

`java.util.concurrent` provides higher-level coordination tools (better than raw `wait`/`notify`).

### 2.1 CountDownLatch (wait for N events — one-time)
A latch starts at a count; threads `countDown()` it; waiters block in `await()` until it reaches 0. **One-shot** (can't reset).
```java
CountDownLatch latch = new CountDownLatch(3);   // wait for 3 tasks
for (int i = 0; i < 3; i++) {
    executor.submit(() -> { doWork(); latch.countDown(); });  // each signals done
}
latch.await();   // main thread blocks until all 3 have counted down
System.out.println("all done");
```
> Use for "wait until N things finish" (e.g., wait for all workers to start/complete, or coordinate a test).

### 2.2 CyclicBarrier (wait for N threads to rendezvous — reusable)
All threads `await()` at a barrier; once N have arrived, they're **all released together**. **Reusable** (resets automatically).
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("all arrived"));
// each thread: barrier.await();  -> blocks until 3 threads reach this point, then all proceed
```
| | CountDownLatch | CyclicBarrier |
|---|---------------|----------------|
| Reusable? | No (one-shot) | Yes (cyclic) |
| Waits for | N count-downs | N threads to rendezvous |
| Use | "wait for tasks to finish" | "threads meet at a checkpoint, repeatedly" |

### 2.3 Semaphore (limit concurrent access)
A semaphore has N **permits**; `acquire()` takes one (blocks if none), `release()` returns one. Limits how many threads access a resource concurrently.
```java
Semaphore semaphore = new Semaphore(5);   // allow at most 5 concurrent
semaphore.acquire();
try {
    accessLimitedResource();    // at most 5 threads here at once
} finally {
    semaphore.release();
}
```
> Use for **rate limiting / resource pools** (e.g., limit concurrent DB connections, throttle external API calls). A `Semaphore(1)` acts like a mutex.

### 2.4 Phaser & Exchanger (awareness)
- **`Phaser`:** a flexible, reusable barrier supporting multiple phases and dynamic party registration.
- **`Exchanger`:** lets two threads swap objects at a synchronization point.

| Synchronizer | One line |
|--------------|----------|
| `CountDownLatch` | Wait for N events (one-time) |
| `CyclicBarrier` | N threads meet at a point (reusable) |
| `Semaphore` | Limit concurrent access to N |
| `Phaser` | Flexible multi-phase barrier |
| `Exchanger` | Two threads swap data |

---

## 3. ThreadLocal (Per-Thread State)

`ThreadLocal<T>` gives each thread its **own independent copy** of a variable — no sharing, so no synchronization needed.
```java
private static final ThreadLocal<SimpleDateFormat> dateFormat =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

String formatted = dateFormat.get().format(new Date());   // each thread has its own formatter
```
### 3.1 How it works & why
- Each thread that calls `get()` sees its **own** value (stored in the thread itself).
- Common uses: **non-thread-safe objects** reused per thread (`SimpleDateFormat`, which is *not* thread-safe!), and **context propagation** (user ID, request ID, transaction, security context).

### 3.2 InheritableThreadLocal
A variant where **child threads inherit** the parent thread's value at creation time:
```java
ThreadLocal<String> context = new InheritableThreadLocal<>();
```

### 3.3 ⚠️ The memory leak danger
> **`ThreadLocal` + thread pools = potential memory leak.** Pool threads are **reused** and live a long time. If you set a `ThreadLocal` and don't `remove()` it, the value lingers on that pooled thread indefinitely — a leak, and possibly **leaking one request's data into another request** (a serious bug/security issue).
```java
try {
    threadLocal.set(value);
    // ... use it ...
} finally {
    threadLocal.remove();   // ALWAYS clean up in pooled environments!
}
```
> In Spring, frameworks manage this for you (e.g., `RequestContextHolder`, security context) and clean up per request — but if you use `ThreadLocal` directly, **always `remove()` in a finally**.

---

## 4. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Blocking on `future.get()` instead of composing | Chain with `thenApply`/`thenCompose` |
| `CompletableFuture` blocking I/O on common pool | Pass a dedicated executor |
| `thenApply` where step is async (CF<CF<T>>) | Use `thenCompose` |
| Forgetting error handling on a CF | Add `exceptionally`/`handle` |
| `CountDownLatch` when you need reuse | Use `CyclicBarrier` |
| `ThreadLocal` without `remove()` in a pool | Always `remove()` in finally (leak/data-bleed) |
| Sharing `SimpleDateFormat` across threads | Use `ThreadLocal` or `DateTimeFormatter` (immutable) |
| Swallowing `CompletionException`/`ExecutionException` | Unwrap the cause and handle it |

---

## 5. Connection to Backend / Spring (Why This Matters Later)

- **`CompletableFuture`** powers **`@Async`** return types and parallel service calls (call N microservices, `allOf`, aggregate) — Phase 5, 12.
- **`WebClient`** (Phase 5.9) returns reactive types; `CompletableFuture` is the imperative async equivalent.
- **`Semaphore`** for rate limiting / bounding concurrency to external systems (Phase 12.3 Resilience4j builds on these ideas).
- **`CountDownLatch`** in tests and startup coordination.
- **`ThreadLocal`** underpins **request/security/transaction context** in Spring (`RequestContextHolder`, `SecurityContextHolder`, `TransactionSynchronizationManager`) and **MDC** logging (Phase 9) — and the leak caveat is critical with thread pools.
- **Resilience patterns** (timeouts, fallbacks) map to `orTimeout`/`exceptionally` (Phase 12.3).

---

## 6. Quick Self-Check Questions

1. What does `CompletableFuture` add over `Future`?
2. What's the difference between `thenApply` and `thenCompose`?
3. How do you run multiple futures in parallel and wait for all of them?
4. How do you handle errors and timeouts in a `CompletableFuture`?
5. Why should blocking I/O futures use a custom executor?
6. Compare `CountDownLatch` and `CyclicBarrier`.
7. What is a `Semaphore` used for?
8. What is `ThreadLocal`, and why is it useful for `SimpleDateFormat`?
9. Why must you call `ThreadLocal.remove()` in a thread pool?

---

## 7. Key Terms Glossary

- **`CompletableFuture`:** composable, non-blocking async result.
- **`supplyAsync` / `runAsync`:** start async tasks (with/without result).
- **`thenApply` / `thenCompose` / `thenCombine`:** transform / flatMap / combine.
- **`allOf` / `anyOf`:** wait for all / first of several futures.
- **`exceptionally` / `handle` / `whenComplete`:** error handling.
- **`orTimeout` / `completeOnTimeout`:** timeouts.
- **`join` vs `get`:** unchecked vs checked blocking retrieval.
- **Synchronizer:** a thread-coordination tool.
- **`CountDownLatch` / `CyclicBarrier` / `Semaphore`:** wait-for-N / rendezvous / permit-limited access.
- **`ThreadLocal`:** per-thread variable copy.
- **`InheritableThreadLocal`:** child threads inherit the value.

---

*Previous topic: **Executor Framework, Future & Callable**.*
*Next topic: **Virtual Threads (Project Loom, Java 21)**.*
