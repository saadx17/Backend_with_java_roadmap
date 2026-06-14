# Locks, Atomics & Concurrent Collections

> **Phase 1 — Java Language Mastery → 1.10 Java Concurrency**
> Goal: Master the `java.util.concurrent` toolkit beyond `synchronized` — explicit `Lock`s, atomic classes (CAS), and concurrent collections.

---

## 0. The Big Picture

`synchronized` (previous note) is simple but limited. The **`java.util.concurrent`** package (`j.u.c.`, by Doug Lea) provides higher-level, more flexible, and often faster tools: explicit **locks**, lock-free **atomics**, and thread-safe **collections**.

```
synchronized      -> simple, built-in, but inflexible
explicit Locks    -> tryLock, fairness, multiple conditions, interruptible
Atomic classes    -> lock-free single-variable updates (CAS)
Concurrent colls  -> thread-safe maps/lists/queues, high throughput
```

> **Prefer these over raw `synchronized`/`wait`** in real code — they're safer, more capable, and well-tuned.

---

## 1. Explicit Locks (`Lock` interface)

`ReentrantLock` and friends provide everything `synchronized` does, **plus** more control.

### 1.1 ReentrantLock
```java
private final Lock lock = new ReentrantLock();

public void doWork() {
    lock.lock();                 // acquire
    try {
        // critical section
    } finally {
        lock.unlock();           // ALWAYS unlock in finally!
    }
}
```
> ⚠️ Unlike `synchronized` (auto-released), you **must `unlock()` in a `finally`** — forgetting it leaks the lock and deadlocks everyone.

### 1.2 What explicit locks add over synchronized
| Feature | `synchronized` | `ReentrantLock` |
|---------|----------------|-----------------|
| Auto-release | ✅ Yes | ❌ Manual `unlock()` |
| **`tryLock()`** (non-blocking) | ❌ | ✅ (avoid deadlock) |
| **`tryLock(timeout)`** | ❌ | ✅ |
| **Interruptible** (`lockInterruptibly`) | ❌ | ✅ |
| **Fairness** policy | ❌ | ✅ (`new ReentrantLock(true)`) |
| Multiple **conditions** | one (wait/notify) | ✅ many (`newCondition()`) |

```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {    // give up after 1s -> deadlock avoidance
    try { ... } finally { lock.unlock(); }
} else {
    // couldn't get the lock — do something else
}
```
- **Fairness:** `new ReentrantLock(true)` grants the lock to the longest-waiting thread (prevents starvation, but slower). Default is unfair (faster).

### 1.3 ReadWriteLock (many readers, one writer)
When reads vastly outnumber writes, a `ReadWriteLock` allows **concurrent reads** but **exclusive writes** — better throughput than a single lock:
```java
private final ReadWriteLock rw = new ReentrantReadWriteLock();

public V read(K key) {
    rw.readLock().lock();        // multiple readers can hold this simultaneously
    try { return map.get(key); } finally { rw.readLock().unlock(); }
}
public void write(K key, V val) {
    rw.writeLock().lock();       // exclusive — blocks all readers and writers
    try { map.put(key, val); } finally { rw.writeLock().unlock(); }
}
```

### 1.4 StampedLock (optimistic reads — advanced)
`StampedLock` (Java 8+) adds **optimistic reading** — read without locking, then validate; retry if a write occurred. Faster for read-heavy workloads but trickier (not reentrant).
```java
long stamp = sl.tryOptimisticRead();   // no lock — just a stamp
int value = this.value;
if (!sl.validate(stamp)) {             // a write happened -> fall back to a real read lock
    stamp = sl.readLock();
    try { value = this.value; } finally { sl.unlockRead(stamp); }
}
```

### 1.5 Condition (await/signal)
`Condition` is the `Lock` equivalent of `wait`/`notify` — and you can have **multiple** conditions per lock (e.g., "not full" and "not empty" for a bounded buffer):
```java
private final Lock lock = new ReentrantLock();
private final Condition notEmpty = lock.newCondition();
private final Condition notFull  = lock.newCondition();
// producer: notFull.await() / notEmpty.signal();  consumer: notEmpty.await() / notFull.signal();
```

---

## 2. Atomic Classes (Lock-Free, CAS-Based)

For updating a **single variable** atomically, atomic classes are **faster than locks** — they use a hardware **compare-and-swap (CAS)** operation, avoiding locking entirely.

### 2.1 The atomic classes
| Class | For |
|-------|-----|
| `AtomicInteger` / `AtomicLong` | atomic int/long counters |
| `AtomicBoolean` | atomic flag |
| `AtomicReference<T>` | atomic object reference |
| `AtomicIntegerArray`, etc. | atomic array elements |

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();      // atomic ++ (the race-free version of count++!)
counter.addAndGet(5);
counter.compareAndSet(10, 20);  // set to 20 only if currently 10
int v = counter.get();
```
> `AtomicInteger.incrementAndGet()` solves the `count++` race (previous note) **without a lock** — faster and simpler than `synchronized` for single counters.

### 2.2 Compare-And-Swap (CAS)
CAS is the heart of lock-free programming:
```
CAS(memory, expected, newValue):
   if memory == expected:  memory = newValue; return true     (success)
   else:                   return false                        (someone else changed it)
```
Atomics loop on CAS until they succeed ("retry on conflict"). It's a single hardware instruction (recall Phase 0.1 ALU) → no OS-level locking, no context switches for the common case.
> **Lock-free ≠ wait-free:** under heavy contention, CAS retries can spin. But for low-to-moderate contention, atomics are excellent.

### 2.3 LongAdder / LongAccumulator (high-contention counters)
Under **very high contention**, `AtomicLong` CAS retries become a bottleneck. **`LongAdder`** spreads the count across multiple internal cells (reducing contention) and sums them on read:
```java
LongAdder requests = new LongAdder();
requests.increment();        // very fast under high contention
long total = requests.sum(); // combine cells when you need the total
```
> Use **`LongAdder`** for hot counters (metrics, request counts) under high concurrency; use `AtomicLong` when you frequently need the exact current value.

---

## 3. Concurrent Collections (recap + depth from Phase 1.6)

Regular collections (`HashMap`, `ArrayList`) are **not thread-safe** — concurrent use corrupts them. `java.util.concurrent` provides thread-safe alternatives that are **far better than** `Collections.synchronizedX` (which locks the whole collection).

### 3.1 ConcurrentHashMap (the workhorse)
```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.merge("count", 1, Integer::sum);              // atomic increment
map.computeIfAbsent("list", k -> new ArrayList<>());
```
- **Fine-grained locking / CAS** on individual bins → high concurrent throughput (no global lock).
- Reads are generally lock-free; writes lock only the relevant bin.
- **No null** keys/values (ambiguous in concurrent context).
- Provides **atomic** operations: `compute`, `merge`, `putIfAbsent`, `computeIfAbsent`.
> ⚠️ Individual operations are atomic, but **compound** sequences (`if (!map.containsKey(k)) map.put(k, v)`) are **not** — use the atomic combos (`putIfAbsent`, `computeIfAbsent`, `merge`) instead.

### 3.2 CopyOnWriteArrayList (recap, Phase 1.6)
Thread-safe list for **read-heavy, write-rare** scenarios — copies the array on each write; reads are lock-free. Ideal for listener lists. Iterators see a snapshot (no `ConcurrentModificationException`).

### 3.3 Concurrent queues (recap, Phase 1.6)
| Collection | Use |
|------------|-----|
| `BlockingQueue` (`ArrayBlockingQueue`, `LinkedBlockingQueue`) | Producer-consumer; blocks on empty/full |
| `ConcurrentLinkedQueue` | Lock-free FIFO |
| `PriorityBlockingQueue` | Thread-safe priority queue |
| `ConcurrentSkipListMap` / `Set` | Concurrent **sorted** map/set |

### 3.4 synchronized wrappers vs concurrent collections
| | `Collections.synchronizedMap` | `ConcurrentHashMap` |
|---|------------------------------|---------------------|
| Locking | One global lock (whole map) | Fine-grained (per bin) |
| Throughput under contention | Low | **High** |
| Iteration | Must externally synchronize | Weakly consistent, safe |
> **Always prefer the concurrent collections** over `synchronized*` wrappers and legacy `Hashtable`/`Vector`.

---

## 4. Choosing the Right Synchronization Tool

```
Updating a single number/flag/reference?  -> Atomic* (AtomicInteger, AtomicReference)
Hot counter under high contention?         -> LongAdder
Need a thread-safe map?                     -> ConcurrentHashMap
Read-mostly list?                           -> CopyOnWriteArrayList
Producer-consumer?                          -> BlockingQueue
Simple mutual exclusion?                    -> synchronized block
Need tryLock/timeout/fairness/conditions?   -> ReentrantLock
Many readers, few writers?                  -> ReadWriteLock / StampedLock
Best of all: no shared mutable state        -> immutability / confinement
```

---

## 5. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Forgetting `unlock()` (or not in `finally`) | Always `unlock()` in `finally` |
| Using `volatile`/locks for a simple counter | Use `AtomicInteger`/`LongAdder` |
| `synchronizedMap` for high concurrency | Use `ConcurrentHashMap` |
| Compound ops on ConcurrentHashMap not atomic | Use `computeIfAbsent`/`merge`/`putIfAbsent` |
| Null keys/values in ConcurrentHashMap | Not allowed |
| CopyOnWriteArrayList for write-heavy use | It copies on every write — read-mostly only |
| Assuming atomics scale infinitely | Under heavy contention, use `LongAdder` |
| Over-locking (one big lock) | Use finer-grained locks / concurrent collections |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- **`ConcurrentHashMap`** is the default for thread-safe caches, registries, and rate limiters in services (Phase 5.7, 12).
- **`AtomicLong`/`LongAdder`** for metrics/counters (Micrometer, Phase 9).
- **`ReentrantLock`/`ReadWriteLock`** for guarding shared resources when `synchronized` isn't enough.
- **`BlockingQueue`** backs thread pools and producer-consumer pipelines (next note, Project 2).
- **Distributed locks** (Redisson, Phase 12.6) extend these concepts across services.
- **Immutability** (Phase 1.3) avoids the need for any of this — the best default.

---

## 7. Quick Self-Check Questions

1. What does `ReentrantLock` offer over `synchronized`? Why must you `unlock()` in `finally`?
2. When would you use a `ReadWriteLock`?
3. What is CAS, and how do atomic classes use it to avoid locks?
4. How does `AtomicInteger.incrementAndGet()` fix the `count++` race?
5. When is `LongAdder` better than `AtomicLong`?
6. Why is `ConcurrentHashMap` better than `Collections.synchronizedMap`?
7. Why are compound operations on `ConcurrentHashMap` not automatically atomic, and what's the fix?
8. Walk through choosing a tool for: a hot counter, a thread-safe cache, a read-mostly list, producer-consumer.

---

## 8. Key Terms Glossary

- **`java.util.concurrent` (j.u.c.):** the concurrency utility package.
- **`Lock` / `ReentrantLock`:** explicit lock with tryLock, fairness, conditions.
- **`tryLock` / fairness / `Condition`:** non-blocking acquire / anti-starvation / await-signal.
- **`ReadWriteLock`:** concurrent reads, exclusive writes.
- **`StampedLock`:** optimistic read locking (advanced).
- **Atomic classes:** lock-free single-variable updates.
- **CAS (compare-and-swap):** hardware atomic update primitive.
- **`LongAdder`:** high-contention counter (striped cells).
- **`ConcurrentHashMap`:** high-throughput thread-safe map.
- **`CopyOnWriteArrayList`:** read-optimized thread-safe list.
- **`BlockingQueue` / `ConcurrentLinkedQueue`:** thread-safe queues.
- **Lock-free:** progress without mutual-exclusion locks (via CAS).

---

*Previous topic: **Synchronization, volatile & the JMM**.*
*Next topic: **Executor Framework, Future & Callable**.*
