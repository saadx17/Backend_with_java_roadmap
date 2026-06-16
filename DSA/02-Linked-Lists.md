# Linked Lists

> **Phase 2 — Data Structures & Algorithms → 2.2 Data Structures**
> Goal: Master linked lists — singly, doubly, and circular — their structure, operations & complexity, classic algorithms (reversal, cycle detection), and when (rarely) to use them.

---

## 0. The Big Picture

A **linked list** stores elements in **nodes**, each holding data plus a **reference (pointer)** to the next node. Unlike arrays, nodes are **scattered in memory** and connected by pointers — so insertion/deletion is O(1) (at a known node), but random access is O(n).

```
Singly linked list:
  head -> [10|next] -> [20|next] -> [30|next] -> null
            node        node        node       (tail's next = null)
```

> Linked lists are a **classic teaching structure** and common in interviews (reversal, cycle detection), but in practice `ArrayList`/`ArrayDeque` almost always win due to cache locality (recall Phase 1.6). Understand them deeply for interviews; rarely choose them in real code.

---

## 1. Array vs Linked List (the fundamental trade-off)

| Aspect | Array | Linked List |
|--------|-------|-------------|
| Memory | Contiguous | Scattered (nodes + pointers) |
| Random access | **O(1)** | **O(n)** (must traverse) |
| Insert/delete at known position | O(n) (shift) | **O(1)** (relink) |
| Insert/delete at ends | O(1) end / O(n) start | **O(1)** both (with tail/doubly) |
| Memory overhead | Low | Higher (pointer per node) |
| Cache locality | **Excellent** | Poor (pointer chasing) |

> The key trade-off: arrays give **fast access, slow middle-modification**; linked lists give **slow access, fast modification at a known node**. But "fast modification" requires you to *already have* the node — finding it is O(n).

---

## 2. Singly Linked List

Each node points to the **next** node only.

### 2.1 The node
```java
class Node<T> {
    T data;
    Node<T> next;
    Node(T data) { this.data = data; }
}
```

### 2.2 A singly linked list implementation
```java
public class SinglyLinkedList<T> {
    private Node<T> head;
    private int size = 0;

    // Add to front — O(1)
    public void addFirst(T data) {
        Node<T> node = new Node<>(data);
        node.next = head;
        head = node;
        size++;
    }

    // Add to end — O(n) without a tail pointer (must traverse)
    public void addLast(T data) {
        Node<T> node = new Node<>(data);
        if (head == null) { head = node; }
        else {
            Node<T> cur = head;
            while (cur.next != null) cur = cur.next;   // traverse to tail — O(n)
            cur.next = node;
        }
        size++;
    }

    // Access by index — O(n)
    public T get(int index) {
        Node<T> cur = head;
        for (int i = 0; i < index; i++) cur = cur.next;
        return cur.data;
    }

    // Remove first — O(1)
    public T removeFirst() {
        T data = head.data;
        head = head.next;
        size--;
        return data;
    }
}
```

### 2.3 Complexity
| Operation | Singly LL |
|-----------|-----------|
| Access/search by index | O(n) |
| Insert/delete at **front** | O(1) |
| Insert/delete at **end** | O(n) (O(1) with a tail pointer) |
| Insert/delete at known node | O(1) (relink) |

---

## 3. Doubly Linked List

Each node points to **both** next **and** previous → enables O(1) deletion of a known node and backward traversal. This is what Java's `LinkedList` is (recall Phase 1.6).

### 3.1 The node
```java
class Node<T> {
    T data;
    Node<T> prev;
    Node<T> next;
    Node(T data) { this.data = data; }
}
```

### 3.2 Why doubly linked is more powerful
```
null <- [10] <-> [20] <-> [30] -> null
          prev/next links both directions
```
- **O(1) deletion of a known node** — you have its `prev`, so you can relink without traversing:
```java
// Remove a known node 'n' in O(1):
n.prev.next = n.next;
if (n.next != null) n.next.prev = n.prev;
```
- **Backward traversal** (from tail).
- Maintain a **tail** pointer → O(1) `addLast`/`removeLast`.
> Doubly linked lists power **LRU caches** (Phase 2.2 last note) and `LinkedHashMap` (Phase 1.6) — O(1) move-to-front/remove is exactly what an LRU needs.

### 3.3 Complexity
| Operation | Doubly LL (with head & tail) |
|-----------|------------------------------|
| Access/search | O(n) |
| Insert/delete at front or back | **O(1)** |
| Delete a known node | **O(1)** |
| Memory | Higher (two pointers/node) |

---

## 4. Circular Linked List

The **tail points back to the head** (instead of null), forming a cycle. Can be singly or doubly circular.
```
  [10] -> [20] -> [30] -+
   ^                    |
   +--------------------+   (tail.next = head)
```
**Use cases:** round-robin scheduling, circular buffers, repeating playlists — any "loop forever through items" scenario.
> ⚠️ Be careful with termination: there's no `null` to stop at — you must track when you return to the start, or you loop forever.

---

## 5. Classic Linked List Algorithms (Interview Essentials)

### 5.1 Reversing a singly linked list (O(n) time, O(1) space)
```java
Node<T> reverse(Node<T> head) {
    Node<T> prev = null;
    Node<T> cur = head;
    while (cur != null) {
        Node<T> next = cur.next;   // save next
        cur.next = prev;           // reverse the pointer
        prev = cur;                // advance prev
        cur = next;                // advance cur
    }
    return prev;                   // new head
}
```
> The classic "three-pointer" reversal — a must-know interview problem. (Recursively it's also possible but uses O(n) stack space — recall Phase 2.1.)

### 5.2 Cycle detection — Floyd's "tortoise and hare" (O(n) time, O(1) space)
```java
boolean hasCycle(Node<T> head) {
    Node<T> slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;          // moves 1 step
        fast = fast.next.next;     // moves 2 steps
        if (slow == fast) return true;   // they meet -> cycle!
    }
    return false;                  // fast reached null -> no cycle
}
```
> Two pointers at different speeds: if there's a cycle, the fast one **laps** the slow one and they meet. If not, fast reaches null. O(1) space — elegant. (Also finds the cycle's start with a second phase.)

### 5.3 Finding the middle (one pass, two pointers)
```java
Node<T> findMiddle(Node<T> head) {
    Node<T> slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;          // 1 step
        fast = fast.next.next;     // 2 steps -> when fast ends, slow is at the middle
    }
    return slow;
}
```

### 5.4 Other classics (awareness)
- Merge two sorted lists, detect intersection, remove Nth from end (two pointers), palindrome check.
- The **two-pointer (fast/slow)** technique (Phase 2.3) is the recurring theme.

---

## 6. Java's LinkedList (recap Phase 1.6)

`java.util.LinkedList` is a **doubly linked list** implementing both `List` and `Deque`:
```java
LinkedList<Integer> ll = new LinkedList<>();
ll.addFirst(1); ll.addLast(2);   // O(1) at both ends
ll.removeFirst();                 // O(1)
ll.get(5);                        // O(n) — no random access!
```
> ⚠️ **In practice, prefer `ArrayList` (for lists) and `ArrayDeque` (for queues/stacks)** — they're faster due to cache locality, even for many insertions (recall Phase 1.6). `LinkedList` is rarely the right choice in real Java code despite its theoretical O(1) inserts.

---

## 7. When to Use a Linked List

| Use a linked list when... | Avoid it when... |
|---------------------------|-------------------|
| You need O(1) insert/delete at a **known** node | You need random access (use array) |
| Implementing other structures (LRU, adjacency lists) | Performance/cache locality matters (use ArrayList) |
| Frequent additions/removals at both ends (use `ArrayDeque` instead, though) | Memory overhead is a concern |
| Learning/interviews | Most real-world list needs (ArrayList wins) |

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Expecting O(1) random access | Linked-list access is O(n) |
| Forgetting "O(1) insert" needs the node first | Finding it is O(n) |
| Choosing `LinkedList` for performance | `ArrayList`/`ArrayDeque` usually win (cache locality) |
| Null-pointer bugs when relinking | Carefully handle head/tail/empty cases |
| Infinite loop in a circular list | Track the start node |
| Losing the list during reversal | Save `next` before reversing the pointer |
| Recursive reversal on huge lists | O(n) stack → StackOverflow (Phase 2.1) |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **LRU caches** (Phase 2.2 last note, 5.7) use a doubly linked list + HashMap for O(1) get/put/evict.
- **`LinkedHashMap`** (Phase 1.6) maintains a doubly linked list through entries (insertion/access order).
- **Adjacency lists** for graphs use linked lists / lists per vertex (Phase 2.2 graphs).
- **Interview prep:** reversal and cycle detection are extremely common (Phase 2.4).
- **Two-pointer technique** (fast/slow) recurs across array and list problems (Phase 2.3).
- Reinforces *why* ArrayList beats LinkedList in real Java (cache, Phase 13).

---

## 10. Quick Self-Check Questions

1. What's the core trade-off between arrays and linked lists?
2. Why is linked-list random access O(n) but array access O(1)?
3. What does a doubly linked list add over a singly linked list?
4. Why is O(1) deletion only true for a *known* node?
5. Walk through reversing a singly linked list with three pointers.
6. Explain Floyd's cycle detection (tortoise and hare).
7. How do you find the middle of a list in one pass?
8. Why prefer `ArrayList`/`ArrayDeque` over `LinkedList` in practice?

---

## 11. Key Terms Glossary

- **Linked list:** nodes connected by pointers.
- **Node:** data + reference(s) to other node(s).
- **Singly / doubly / circular:** next-only / next+prev / tail→head.
- **Head / tail:** first / last node.
- **Pointer chasing:** following references (poor cache locality).
- **Tortoise and hare (Floyd's):** two-speed pointers for cycle detection.
- **Two-pointer (fast/slow):** technique for middle/cycle/Nth-from-end.
- **Adjacency list:** linked-list/list representation of graph edges.

---

*Previous topic: **Arrays & Dynamic Arrays**.*
*Next topic: **Stacks & Queues**.*
