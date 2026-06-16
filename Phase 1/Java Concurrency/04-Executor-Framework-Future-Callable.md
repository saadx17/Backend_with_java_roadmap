# Executor Framework, Future & Callable

> **Phase 1 — Java Language Mastery → 1.10 Java Concurrency**
> Goal: Master the modern way to run concurrent tasks — the Executor framework, thread pools (ThreadPoolExecutor), Callable/Future, scheduling, and the Fork/Join framework.

---

## 0. The Big Picture

Creating raw `Thread`s for every task is wasteful (thread creation is expensive — recall Phase 0.1) and uncontrolled (unbounded threads → resource exhaustion). The **Executor framework** decouples **task submission** from **thread management**: you submit tasks, a **thread pool** runs them on a managed, reusable set of threads.

```
You: submit tasks  ->  [ Thread Pool: reuses N worker threads ]  ->  results
```

> **This is how every backend handles concurrency.** Tomcat, `@Async`, scheduled jobs, parallel processing — all use executors/pools under the hood.

---

## 1. ExecutorService — Submitting Tasks

`ExecutorService` is the core interface: submit `Runnable`/`Callable` tasks, get them executed by a pool.
```java
ExecutorService executor = Executors.newFixedThreadPool(4);   // pool of 4 threads

executor.execute(() -> System.out.println("fire and forget"));   // Runnable, no result
Future<Integer> future = executor.submit(() -> compute());       // Callable, returns a Future

executor.shutdown();   // stop accepting new tasks; finish running ones
```

| Method | Purpose |
|--------|---------|
| `execute(Runnable)` | Run a task (no result) |
| `submit(Callable/Runnable)` | Run and get a `Future` |
| `invokeAll(tasks)` | Run all, wait for all, return `List<Future>` |
| `invokeAny(tasks)` | Run all, return the first successful result |
| `shutdown()` / `shutdownNow()` | Graceful / forceful shutdown |
| `awaitTermination(timeout)` | Block until tasks finish or timeout |

---

## 2. Callable & Future

### 2.1 Callable — a task that returns a value
Unlike `Runnable` (returns void, can't throw checked exceptions), **`Callable<V>`** returns a value and can throw checked exceptions:
```java
Callable<Integer> task = () -> {
    Thread.sleep(100);
    return 42;                  // returns a result
};
```

### 2.2 Future — a handle to a pending result
`submit()` returns a **`Future<V>`** — a placeholder for a result that will be available later:
```java
Future<Integer> future = executor.submit(task);
// ... do other work while it runs ...
Integer result = future.get();          // BLOCKS until the result is ready
Integer r2 = future.get(2, TimeUnit.SECONDS);  // with timeout -> TimeoutException
boolean done = future.isDone();
future.cancel(true);                    // attempt to cancel (interrupt)
```
| `Future` method | Effect |
|-----------------|--------|
| `get()` | Block until done, return result (or throw `ExecutionException`) |
| `get(timeout)` | Block up to timeout |
| `isDone()` / `isCancelled()` | Status checks |
| `cancel(mayInterrupt)` | Attempt cancellation |

> ⚠️ **`future.get()` blocks** — calling it immediately after `submit()` defeats the purpose (you've just made it synchronous). Submit several tasks, *then* collect results. (`CompletableFuture`, next note, fixes the blocking limitation with composition.)

---

## 3. The Executors Factory & Pool Types

`Executors` provides factory methods for common pool types:
| Factory method | Pool behavior |
|----------------|---------------|
| `newFixedThreadPool(n)` | Fixed n threads; tasks queue if all busy |
| `newCachedThreadPool()` | Grows/shrinks on demand; reuses idle threads (unbounded!) |
| `newSingleThreadExecutor()` | One thread; tasks run sequentially |
| `newScheduledThreadPool(n)` | For delayed/periodic tasks (§5) |
| `newVirtualThreadPerTaskExecutor()` | One virtual thread per task (Java 21 — last note) |

> ⚠️ **`newCachedThreadPool` is unbounded** — under load it can spawn thousands of threads and exhaust resources. For servers, prefer a **bounded** pool (fixed, or a custom `ThreadPoolExecutor`) so the queue provides backpressure.

---

## 4. ThreadPoolExecutor (Custom Pools — Production)

The factory methods are convenient but hide important parameters. For production, configure a **`ThreadPoolExecutor`** explicitly:
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                              // corePoolSize  — threads kept alive
    8,                              // maximumPoolSize — max threads under load
    60, TimeUnit.SECONDS,           // keepAliveTime — idle non-core threads die after this
    new ArrayBlockingQueue<>(100),  // workQueue — bounded! (backpressure)
    new ThreadFactoryBuilder().setNameFormat("worker-%d").build(),  // named threads
    new ThreadPoolExecutor.CallerRunsPolicy()   // rejection policy
);
```

### 4.1 The parameters
| Parameter | Meaning |
|-----------|---------|
| **corePoolSize** | Threads kept alive even when idle |
| **maximumPoolSize** | Max threads when the queue is full |
| **keepAliveTime** | How long extra (non-core) idle threads survive |
| **workQueue** | Holds pending tasks (the `BlockingQueue` — Phase 1.6) |
| **threadFactory** | Creates threads (name them for debugging!) |
| **rejectedExecutionHandler** | What to do when the pool + queue are full |

### 4.2 How tasks flow through a pool
```
Task submitted:
  1. core threads available?      -> run it on a core thread
  2. else, queue has space?       -> queue it
  3. else, below maxPoolSize?     -> create a new thread to run it
  4. else (queue full + max)      -> REJECT (apply rejection policy)
```

### 4.3 Rejection policies (when full)
| Policy | Behavior |
|--------|----------|
| `AbortPolicy` (default) | Throw `RejectedExecutionException` |
| `CallerRunsPolicy` | Run the task on the **calling** thread (natural backpressure) |
| `DiscardPolicy` | Silently drop the task |
| `DiscardOldestPolicy` | Drop the oldest queued task, then retry |

### 4.4 Sizing the pool (recall Phase 0.1 context switching)
| Workload | Pool size guideline |
|----------|---------------------|
| **CPU-bound** | ≈ number of cores (`Runtime.getRuntime().availableProcessors()`) |
| **I/O-bound** | Larger (threads block waiting) — e.g., cores × (1 + wait/compute ratio) |
> Too few threads underutilize cores; too many cause context-switch thrashing (Phase 0.1) and memory pressure. **Bounded queues** prevent unbounded memory growth and apply backpressure (recall Phase 1.6 BlockingQueue).

---

## 5. ScheduledExecutorService (Delayed & Periodic Tasks)

For running tasks after a delay or repeatedly:
```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

scheduler.schedule(task, 5, TimeUnit.SECONDS);                      // once, after 5s
scheduler.scheduleAtFixedRate(task, 0, 1, TimeUnit.MINUTES);        // every minute (fixed rate)
scheduler.scheduleWithFixedDelay(task, 0, 1, TimeUnit.MINUTES);     // 1 min AFTER each finishes
```
| Method | Timing |
|--------|--------|
| `schedule` | Run once after a delay |
| `scheduleAtFixedRate` | Start every period (regardless of task duration) |
| `scheduleWithFixedDelay` | Wait a fixed gap *between* runs |
> Replaces the old `Timer`/`TimerTask` (which used a single thread and broke on exceptions). Spring's `@Scheduled` (Phase 5.8) builds on this.

---

## 6. ExecutorService Lifecycle (Shutdown Properly)

> ⚠️ **An executor's threads keep the JVM alive** (they're non-daemon by default). You **must shut it down** or the app won't exit.

```java
executor.shutdown();                          // stop accepting new tasks; finish queued ones
if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {  // wait for completion
    executor.shutdownNow();                   // force: interrupt running tasks
}
```
| Method | Effect |
|--------|--------|
| `shutdown()` | Graceful: no new tasks; existing ones finish |
| `shutdownNow()` | Forceful: interrupt running tasks, return queued ones |
| `awaitTermination(t)` | Block until done or timeout |

> The standard pattern: `shutdown()` → `awaitTermination()` → `shutdownNow()` if it times out. This enables **graceful shutdown** (recall Phase 0.3 SIGTERM, Phase 10).

---

## 7. Fork/Join Framework (Divide & Conquer Parallelism)

The **Fork/Join framework** (`ForkJoinPool`) is for **recursively splittable** CPU-bound tasks — split a big task into subtasks, run them in parallel, combine results. It uses **work-stealing**.
```java
class SumTask extends RecursiveTask<Long> {     // RecursiveTask returns a value
    private final long[] arr; private final int lo, hi;
    SumTask(long[] arr, int lo, int hi) { this.arr = arr; this.lo = lo; this.hi = hi; }

    protected Long compute() {
        if (hi - lo <= 1000) {                  // base case: compute directly
            long sum = 0;
            for (int i = lo; i < hi; i++) sum += arr[i];
            return sum;
        }
        int mid = (lo + hi) / 2;
        SumTask left  = new SumTask(arr, lo, mid);
        SumTask right = new SumTask(arr, mid, hi);
        left.fork();                            // run left asynchronously
        long rightResult = right.compute();     // compute right directly
        return left.join() + rightResult;       // combine
    }
}
long total = new ForkJoinPool().invoke(new SumTask(array, 0, array.length));
```
| Class | Use |
|-------|-----|
| `RecursiveTask<V>` | Subtask that returns a result |
| `RecursiveAction` | Subtask with no result |
| `ForkJoinPool` | The pool (has a shared `commonPool`) |

### 7.1 Work-stealing
Each worker thread has its own task deque; **idle threads "steal" tasks** from busy threads' deques → excellent CPU utilization.
> **Parallel streams** (Phase 1.8) run on the Fork/Join **common pool**. The same caveats apply: only for **CPU-bound, splittable** work; never for blocking I/O (it starves the shared pool).

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Creating raw threads per task | Use an executor/thread pool |
| `newCachedThreadPool` under heavy load | Use a bounded pool (backpressure) |
| Calling `future.get()` right after `submit()` | Submit all, then collect — or use `CompletableFuture` |
| Forgetting to shut down the executor | `shutdown()` + `awaitTermination()` (JVM won't exit otherwise) |
| Unbounded work queue | Use a bounded queue + rejection policy |
| Wrong pool size | CPU-bound ≈ cores; I/O-bound larger |
| Blocking I/O on the Fork/Join common pool | Use a dedicated pool / async |
| Swallowing exceptions in pool tasks | `Future.get()` throws `ExecutionException`; handle/log it |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Tomcat/Jetty** use thread pools to handle requests — pool sizing affects throughput (Phase 13).
- **`@Async`** (Phase 5) runs methods on a Spring-managed `ThreadPoolTaskExecutor` (configure core/max/queue!).
- **`@Scheduled`** (Phase 5.8) uses a scheduled executor; ShedLock coordinates across instances.
- **Project 2** (file processor): thread pool + `BlockingQueue` for producer-consumer.
- **Graceful shutdown** (Phase 10): Spring shuts down executors to drain in-flight work on SIGTERM.
- **`CompletableFuture`** (next note) builds on executors for non-blocking composition.
- **Connection pools** (HikariCP, Phase 4) apply the same pooling principle to DB connections.

---

## 10. Quick Self-Check Questions

1. Why use a thread pool instead of creating threads per task?
2. What's the difference between `Runnable` and `Callable`? Between `execute` and `submit`?
3. What does `Future.get()` do, and what's the trap with calling it too early?
4. Why is `newCachedThreadPool` risky for servers?
5. Explain the `ThreadPoolExecutor` parameters and how a task flows through the pool.
6. Name the rejection policies and when `CallerRunsPolicy` is useful.
7. How do you size a pool for CPU-bound vs I/O-bound work?
8. Why must you shut down an executor, and what's the graceful pattern?
9. What is the Fork/Join framework and work-stealing for?

---

## 11. Key Terms Glossary

- **Executor framework:** decouples task submission from thread management.
- **`ExecutorService`:** submits and manages tasks.
- **Thread pool:** a reusable set of worker threads.
- **`Runnable` / `Callable`:** task without / with a result.
- **`Future`:** a handle to a pending result (`get` blocks).
- **`Executors`:** factory for common pool types.
- **`ThreadPoolExecutor`:** configurable pool (core/max/queue/rejection).
- **Rejection policy:** behavior when the pool+queue are full.
- **`ScheduledExecutorService`:** delayed/periodic task execution.
- **`shutdown` / `shutdownNow` / `awaitTermination`:** lifecycle control.
- **Fork/Join / `ForkJoinPool`:** divide-and-conquer parallelism.
- **Work-stealing:** idle threads taking tasks from busy ones.

---

*Previous topic: **Locks, Atomics & Concurrent Collections**.*
*Next topic: **CompletableFuture, Synchronizers & ThreadLocal**.*
