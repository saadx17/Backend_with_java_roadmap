# Queue & Deque Implementations

> **Phase 1 — Java Language Mastery → 1.6 Java Collections Framework**
> Goal: Master `Queue` and `Deque` — `ArrayDeque`, `PriorityQueue`, `LinkedList` as a queue, and the concurrent `BlockingQueue`/`ConcurrentLinkedQueue` — their behavior, methods, and use cases.

---

## 0. The Big Picture

A **`Queue<E>`** holds elements for **processing in a particular order** — classically **FIFO** (First-In-First-Out). A **`Deque<E>`** ("double-ended queue") allows adding/removing from **both ends**, so it can act as a queue *or* a stack.

```
Queue (FIFO):   enqueue -> [ a  b  c ] -> dequeue   (add at back, remove from front)
Stack (LIFO):                push/pop at the same end
Deque:          add/remove at BOTH ends
```

> Queues model **work to be processed in order** — task queues, BFS traversal, producer-consumer pipelines, scheduling.

---

## 1. The Queue Interface & Its Two Method Families

`Queue` offers two styles of methods — one **throws** on failure, one **returns a special value**. Knowing both prevents bugs:

| Operation | Throws exception | Returns special value |
|-----------|------------------|------------------------|
| **Insert** | `add(e)` | `offer(e)` → `false` if full |
| **Remove** | `remove()` | `poll()` → `null` if empty |
| **Examine** (peek) | `element()` | `peek()` → `null` if empty |

```java
Queue<String> q = new LinkedList<>();
q.offer("a");           // add to back (returns false instead of throwing if bounded/full)
q.offer("b");
q.peek();               // "a"  (look at front, don't remove)
q.poll();               // "a"  (remove and return front); null if empty
```
> **Prefer `offer`/`poll`/`peek`** (the non-throwing trio) for normal flow — they return `null`/`false` instead of throwing on empty/full, which is usually what you want.

---

## 2. ArrayDeque (the preferred queue AND stack)

`ArrayDeque` is a **resizable-array** double-ended queue. It's the **recommended** implementation for both queues and stacks.

### 2.1 As a queue (FIFO)
```java
Deque<String> queue = new ArrayDeque<>();
queue.offer("a");        // offerLast — add to back
queue.offer("b");
queue.poll();            // pollFirst — remove from front -> "a"
```

### 2.2 As a stack (LIFO) — replaces the legacy `Stack`
```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);           // addFirst — push onto top
stack.push(2);
stack.peek();            // 2  (top)
stack.pop();             // 2  (remove top)
```

### 2.3 Why ArrayDeque
| Aspect | ArrayDeque |
|--------|-----------|
| Add/remove at both ends | **O(1)** |
| Memory | Compact (array-backed, cache-friendly) |
| Nulls | ❌ Not allowed (null is the "empty" sentinel) |
| Thread-safe? | No (single-threaded use) |

> **Use `ArrayDeque` instead of `Stack` (legacy) and instead of `LinkedList` for queues/stacks** — it's faster (better cache locality) and not legacy. (*Effective Java* recommends it.)

---

## 3. PriorityQueue (a heap)

`PriorityQueue` orders elements by **priority**, not insertion order. The **head is always the smallest** (by natural order or a `Comparator`). Internally it's a **binary heap**.

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();   // min-heap (natural order)
pq.offer(30); pq.offer(10); pq.offer(20);
pq.poll();   // 10  (smallest first)
pq.poll();   // 20
pq.poll();   // 30

// Max-heap (reverse order):
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());

// Custom priority (e.g., tasks by priority field):
PriorityQueue<Task> tasks = new PriorityQueue<>(Comparator.comparingInt(Task::priority));
```

### 3.1 Characteristics
| Operation | Complexity |
|-----------|-----------|
| `offer` (insert) | O(log n) |
| `poll` (remove min) | O(log n) |
| `peek` (view min) | O(1) |

- **Not sorted overall** — only the **head** is guaranteed smallest; iteration order is *not* sorted.
- No nulls; elements must be `Comparable` or have a `Comparator`.
> **Use a `PriorityQueue`** when you repeatedly need the smallest/largest/highest-priority element (scheduling, Dijkstra's algorithm — Phase 2, top-K problems).

---

## 4. LinkedList as a Queue/Deque

`LinkedList` implements `Deque`, so it can serve as a queue (recall List note). However:
- It works, but `ArrayDeque` is **faster** (cache locality) for queue/stack use.
- `LinkedList` allows **null** elements (ArrayDeque doesn't) — occasionally a reason to choose it, but rarely.
> **Prefer `ArrayDeque`** unless you specifically need List behavior + Deque, or null elements.

---

## 5. Concurrent Queues (thread-safe)

For multi-threaded scenarios (especially the **producer-consumer pattern**), the JCF provides thread-safe queues in `java.util.concurrent` (full detail in Phase 1.10).

### 5.1 BlockingQueue
A **`BlockingQueue`** blocks the thread when it tries to take from an empty queue (waits for an element) or put into a full queue (waits for space) — the backbone of **producer-consumer**:
```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);  // bounded capacity 100

// Producer thread:
queue.put(task);     // blocks if the queue is full

// Consumer thread:
Task t = queue.take();  // blocks if the queue is empty
```
| Implementation | Notes |
|----------------|-------|
| `ArrayBlockingQueue` | Bounded, array-backed |
| `LinkedBlockingQueue` | Optionally bounded, linked nodes |
| `PriorityBlockingQueue` | Blocking + priority ordering |
| `DelayQueue` | Elements available only after a delay |
| `SynchronousQueue` | No capacity — hands off directly producer→consumer |

> `BlockingQueue` is exactly what **thread pools** (`ThreadPoolExecutor`) use internally to hold pending tasks (Phase 1.10) — and what you'd use in **Project 2** (multithreaded file processor).

### 5.2 ConcurrentLinkedQueue
A **non-blocking**, thread-safe, lock-free FIFO queue (uses CAS — compare-and-swap). Good for high-concurrency scenarios where you don't want blocking.
```java
Queue<Event> events = new ConcurrentLinkedQueue<>();   // lock-free, thread-safe
```

---

## 6. Queue/Deque Implementations Compared

| Implementation | Type | Ordering | Thread-safe | Use when |
|----------------|------|----------|-------------|----------|
| **ArrayDeque** | Deque | FIFO or LIFO | No | Default queue/stack |
| **PriorityQueue** | Queue | By priority (heap) | No | Always need min/max/priority element |
| **LinkedList** | Deque | FIFO/LIFO | No | Need List + Deque, or nulls |
| **BlockingQueue** (impls) | Queue | FIFO/priority | **Yes** | Producer-consumer, thread pools |
| **ConcurrentLinkedQueue** | Queue | FIFO | **Yes** (lock-free) | High-concurrency non-blocking |

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using `Stack` (legacy) | Use `ArrayDeque` as a stack |
| Using `LinkedList` for queues by default | Prefer `ArrayDeque` (faster) |
| Putting `null` in `ArrayDeque`/`PriorityQueue` | Not allowed (null is a sentinel) |
| Expecting `PriorityQueue` iteration to be sorted | Only the **head** is ordered; iteration isn't sorted |
| Mixing throwing vs non-throwing methods unintentionally | Prefer `offer`/`poll`/`peek` |
| Using a non-thread-safe queue across threads | Use `BlockingQueue`/`ConcurrentLinkedQueue` |
| Unbounded queue causing memory blowup | Use a **bounded** BlockingQueue for backpressure |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Thread pools & async (Phase 1.10):** `ThreadPoolExecutor` uses a `BlockingQueue` to hold pending tasks; bounded queues provide **backpressure** (recall Phase 0.1 I/O).
- **Producer-consumer (Project 2):** `BlockingQueue` is the canonical pattern for pipeline processing.
- **Message processing (Phase 11):** in-memory queues buffer work before/after Kafka/RabbitMQ consumers.
- **`PriorityQueue`** for scheduling, rate limiting, and graph algorithms (Phase 2, 12).
- **`ArrayDeque`** as an efficient stack for parsing, BFS/DFS, undo stacks (Phase 2).
- **`ConcurrentLinkedQueue`** for lock-free event buffers in high-throughput services (Phase 13).

---

## 9. Quick Self-Check Questions

1. What's the difference between a Queue (FIFO) and a Deque?
2. What are the two method families of `Queue`, and which should you prefer?
3. Why use `ArrayDeque` instead of `Stack` and `LinkedList`?
4. How does a `PriorityQueue` order elements, and is its iteration sorted?
5. What's the complexity of `PriorityQueue` insert/remove/peek?
6. What does a `BlockingQueue` do when empty/full, and what pattern does it enable?
7. Where is `BlockingQueue` used internally in Java?
8. What's the difference between `BlockingQueue` and `ConcurrentLinkedQueue`?

---

## 10. Key Terms Glossary

- **Queue:** ordered collection for processing (typically FIFO).
- **Deque:** double-ended queue (add/remove at both ends).
- **FIFO / LIFO:** first-in-first-out / last-in-first-out.
- **`offer`/`poll`/`peek`:** non-throwing insert/remove/examine.
- **`ArrayDeque`:** array-backed deque; preferred queue & stack.
- **`PriorityQueue`:** heap-ordered queue (head = highest priority).
- **Binary heap:** the tree structure backing a priority queue.
- **`BlockingQueue`:** thread-safe queue that blocks on empty/full.
- **Producer-consumer:** pattern of producers adding and consumers taking work.
- **Backpressure:** bounding a queue to slow producers when consumers lag.
- **`ConcurrentLinkedQueue`:** lock-free thread-safe FIFO queue.

---

*Previous topic: **Set Implementations**.*
*Next topic: **Map Implementations (incl. HashMap internals)**.*
