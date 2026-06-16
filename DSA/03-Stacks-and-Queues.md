# Stacks & Queues

> **Phase 2 — Data Structures & Algorithms → 2.2 Data Structures**
> Goal: Master stacks (LIFO) and queues (FIFO) — including circular queues, priority queues, and deques — their implementations, operations, and the problems they solve.

---

## 0. The Big Picture

**Stacks** and **queues** are **restricted-access** linear structures: you can only add/remove from specific ends. This restriction is exactly what makes them useful for modeling order-dependent processing.

```
STACK (LIFO — Last In, First Out):     QUEUE (FIFO — First In, First Out):
   push ->  [c]  <- pop (top)            enqueue -> [a][b][c] -> dequeue
            [b]                                       (rear)    (front)
            [a]
  (like a stack of plates)              (like a line of people)
```

> These build directly on **Phase 1.6** (ArrayDeque, PriorityQueue) — here we cover the concepts, implementations, and algorithmic uses.

---

## 1. Stacks (LIFO)

A **stack** allows access only at the **top**: the **last** element added is the **first** removed.

### 1.1 Core operations (all O(1))
| Operation | Effect |
|-----------|--------|
| `push(x)` | Add to the top |
| `pop()` | Remove and return the top |
| `peek()` | Look at the top without removing |
| `isEmpty()` | Is it empty? |

### 1.2 Array-based stack implementation
```java
public class ArrayStack<T> {
    private Object[] data = new Object[10];
    private int top = -1;             // index of the top element (-1 = empty)

    public void push(T x) {
        if (top == data.length - 1) resize();
        data[++top] = x;             // O(1) amortized
    }
    @SuppressWarnings("unchecked")
    public T pop() {
        if (isEmpty()) throw new RuntimeException("empty");
        T x = (T) data[top];
        data[top--] = null;          // avoid memory leak (Phase 1.11)
        return x;                    // O(1)
    }
    @SuppressWarnings("unchecked")
    public T peek() { return (T) data[top]; }
    public boolean isEmpty() { return top == -1; }
    private void resize() {
        data = java.util.Arrays.copyOf(data, data.length * 2);
    }
}
```

### 1.3 Stacks in Java — use ArrayDeque (recall Phase 1.6)
```java
Deque<Integer> stack = new ArrayDeque<>();   // PREFERRED (not legacy Stack)
stack.push(1); stack.push(2);
stack.peek();   // 2
stack.pop();    // 2
```
> ⚠️ **Don't use the legacy `java.util.Stack`** (synchronized, extends Vector — Phase 1.6). Use `ArrayDeque` as a stack.

### 1.4 Where stacks are used
| Use case | Why a stack |
|----------|-------------|
| **Function call stack** | LIFO call/return (recall Phase 0.1/1.2) |
| **Undo/redo** | Last action undone first |
| **Expression evaluation** | Parsing parentheses, postfix evaluation |
| **Backtracking** (DFS) | Remember where to return (Phase 2.3) |
| **Browser back button** | Last page first |
| **Balanced parentheses** | Push opens, pop on close |

### 1.5 Classic stack problem: balanced parentheses
```java
boolean isBalanced(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = Map.of(')', '(', ']', '[', '}', '{');
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '[' || c == '{') stack.push(c);
        else if (pairs.containsKey(c)) {
            if (stack.isEmpty() || stack.pop() != pairs.get(c)) return false;
        }
    }
    return stack.isEmpty();        // all opens matched
}
```

---

## 2. Queues (FIFO)

A **queue** allows adding at the **rear** and removing from the **front**: the **first** element added is the **first** removed.

### 2.1 Core operations (all O(1))
| Operation | Effect |
|-----------|--------|
| `enqueue(x)` / `offer(x)` | Add to the rear |
| `dequeue()` / `poll()` | Remove and return the front |
| `peek()` | Look at the front |
| `isEmpty()` | Is it empty? |

### 2.2 Why a naive array queue is inefficient
If you `dequeue` by removing index 0 of an array, you must **shift all elements** → O(n). The fix is a **circular queue** (§3) or a linked structure.

### 2.3 Queues in Java — use ArrayDeque (recall Phase 1.6)
```java
Queue<Integer> queue = new ArrayDeque<>();   // PREFERRED
queue.offer(1); queue.offer(2);
queue.peek();   // 1 (front)
queue.poll();   // 1
```
> Prefer the non-throwing `offer`/`poll`/`peek` trio (return null/false) over `add`/`remove`/`element` (throw) — recall Phase 1.6.

### 2.4 Where queues are used
| Use case | Why a queue |
|----------|-------------|
| **BFS traversal** | Process level by level (Phase 2.2 graphs/trees) |
| **Task scheduling / job queues** | First come, first served |
| **Producer-consumer** | BlockingQueue buffers work (Phase 1.6/1.10) |
| **Request handling** | Process requests in order |
| **Print/print spooler** | FIFO order |

---

## 3. Circular Queue (Ring Buffer)

A **circular queue** uses a fixed array with **front** and **rear** indices that **wrap around** (using modulo), reusing freed slots — O(1) enqueue/dequeue with no shifting.
```
Capacity 5, indices wrap with % 5:
  [_][b][c][d][_]      front=1, rear=4
  enqueue 'e' -> rear = (4+1)%5 = 0 -> wraps to index 0
```

### 3.1 Implementation
```java
public class CircularQueue {
    private int[] data;
    private int front = 0, size = 0, capacity;

    public CircularQueue(int capacity) {
        this.capacity = capacity;
        data = new int[capacity];
    }
    public boolean enqueue(int x) {
        if (size == capacity) return false;          // full
        int rear = (front + size) % capacity;        // wrap with modulo
        data[rear] = x;
        size++;
        return true;
    }
    public int dequeue() {
        if (size == 0) throw new RuntimeException("empty");
        int x = data[front];
        front = (front + 1) % capacity;              // advance front, wrapping
        size--;
        return x;
    }
}
```
> **Ring buffers** are everywhere in systems: bounded queues, logging buffers, streaming, audio/video buffers. The modulo wrap-around reuses memory without shifting. (`ArrayDeque` is internally a growable circular buffer.)

---

## 4. Deque (Double-Ended Queue)

A **deque** ("deck") allows add/remove at **both** ends → it can act as a **stack OR a queue** (recall Phase 1.6). `ArrayDeque` is the go-to.
```java
Deque<Integer> deque = new ArrayDeque<>();
deque.addFirst(1); deque.addLast(2);
deque.removeFirst(); deque.removeLast();   // all O(1)
```
| As a... | Methods |
|---------|---------|
| Stack (LIFO) | `push`/`pop`/`peek` (front) |
| Queue (FIFO) | `offer`/`poll`/`peek` (rear in, front out) |
> Deques also power the **sliding-window-maximum** algorithm (a monotonic deque) — a classic Phase 2.3 pattern.

---

## 5. Priority Queue (Heap-Based)

A **priority queue** dequeues by **priority**, not insertion order — the highest-priority (or lowest, by default min) element comes out first. Internally a **binary heap** (covered in the Heaps note, Phase 2.2).
```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();          // smallest first
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
minHeap.offer(30); minHeap.offer(10); minHeap.offer(20);
minHeap.poll();   // 10 (smallest)
```
| Operation | Complexity |
|-----------|-----------|
| `offer` (insert) | O(log n) |
| `poll` (remove min/max) | O(log n) |
| `peek` | O(1) |
> Use when you repeatedly need the min/max/highest-priority element: **scheduling, Dijkstra's shortest path** (Phase 2.3), **top-K** problems, **merge K sorted lists**. (Full heap mechanics in the Heaps note.)

---

## 6. Concurrent Variants (recap Phase 1.6/1.10)

For multi-threaded use:
- **`BlockingQueue`** (`ArrayBlockingQueue`, `LinkedBlockingQueue`) — producer-consumer; blocks on empty/full (Phase 1.10, Project 2).
- **`ConcurrentLinkedQueue`** — lock-free thread-safe FIFO.
- **`PriorityBlockingQueue`** — thread-safe priority queue.

---

## 7. Summary Comparison

| Structure | Access pattern | Add/remove | Backed by | Java class |
|-----------|----------------|-----------|-----------|------------|
| **Stack** | LIFO | O(1) at top | array/linked | `ArrayDeque` |
| **Queue** | FIFO | O(1) ends | array(circular)/linked | `ArrayDeque` |
| **Circular queue** | FIFO, fixed cap | O(1), wraps | array | (custom / ArrayDeque) |
| **Deque** | both ends | O(1) both | array | `ArrayDeque` |
| **Priority queue** | by priority | O(log n) | binary heap | `PriorityQueue` |

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using legacy `Stack` class | Use `ArrayDeque` |
| Naive array queue (dequeue shifts) → O(n) | Use a circular queue / `ArrayDeque` |
| Expecting `PriorityQueue` iteration to be sorted | Only the head is ordered (Phase 1.6) |
| Putting null in `ArrayDeque`/`PriorityQueue` | Not allowed |
| Forgetting to null out popped slots | Memory leak (lingering references, Phase 1.11) |
| Mixing throwing vs non-throwing methods | Prefer `offer`/`poll`/`peek` |
| Circular queue full/empty ambiguity | Track `size` explicitly |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **BFS** (queue) and **DFS** (stack) traversals power graph/tree algorithms (Phase 2.2/2.3).
- **Producer-consumer** with `BlockingQueue` (Phase 1.10, Project 2) and thread-pool work queues (Phase 1.10).
- **Priority queues** for scheduling, rate limiting, Dijkstra (Phase 2.3, 12).
- **Ring buffers** for logging, streaming, and bounded buffers (Phase 9, 11).
- **Message queues** (Kafka/RabbitMQ, Phase 11) are distributed FIFO/priority queues — same concepts at scale.
- **Undo/redo, expression parsing, backtracking** use stacks.

---

## 10. Quick Self-Check Questions

1. What's the difference between LIFO and FIFO?
2. What are a stack's core operations, and what's their complexity?
3. Why is a naive array-based queue inefficient, and how does a circular queue fix it?
4. How does modulo enable wrap-around in a circular queue?
5. What is a deque, and how can it act as both a stack and a queue?
6. How does a priority queue differ from a regular queue, and what's it backed by?
7. Which Java classes should you use for stacks, queues, and deques?
8. Give two real-world uses each for stacks and queues.

---

## 11. Key Terms Glossary

- **Stack (LIFO):** add/remove at the top only.
- **Queue (FIFO):** add at rear, remove at front.
- **push/pop/peek:** stack add/remove/inspect.
- **enqueue/dequeue (offer/poll):** queue add/remove.
- **Circular queue / ring buffer:** fixed array with wrap-around indices.
- **Deque:** double-ended queue (both ends).
- **Priority queue:** dequeue by priority (heap-based).
- **Binary heap:** the structure backing a priority queue (Phase 2.2 Heaps).
- **BlockingQueue / ConcurrentLinkedQueue:** thread-safe queues.

---

*Previous topic: **Linked Lists**.*
*Next topic: **Hash Tables**.*
