# Heaps, Tries & Specialized Trees (Segment, Fenwick, B-Tree)

> **Phase 2 — Data Structures & Algorithms → 2.2 Data Structures**
> Goal: Master heaps (priority queues), tries (autocomplete), and the specialized trees you should know conceptually — Segment/Fenwick trees and B-Trees/B+ Trees (database indexes).

---

## 0. The Big Picture

Beyond BSTs (previous note), several specialized tree structures solve specific problems extremely well:
- **Heaps** → fast min/max access (priority queues).
- **Tries** → prefix-based string search (autocomplete).
- **Segment/Fenwick trees** → fast range queries with updates.
- **B-Trees/B+ Trees** → disk-based indexes (databases).

> Heaps and tries you should be able to implement; segment/Fenwick are competitive-programming tools; B-trees are conceptual (databases — Phase 4.4). All build on tree fundamentals from the previous note.

---

## 1. Heaps (Binary Heap)

A **binary heap** is a **complete binary tree** (all levels full except the last, filled left-to-right) satisfying the **heap property**. It gives O(1) access to the min (or max) and O(log n) insert/remove — the structure behind **priority queues** (Phase 1.6/2.2).

### 1.1 The heap property
| Heap type | Property |
|-----------|----------|
| **Min-heap** | Each parent ≤ its children → **root is the minimum** |
| **Max-heap** | Each parent ≥ its children → **root is the maximum** |
```
Min-heap:        10           Max-heap:       50
                /  \                          /  \
              20    15                       30    40
              /\    /                        /\    /
            30 25  18                       10 20 35
(root = smallest)                          (root = largest)
```
> ⚠️ A heap is **NOT sorted** (unlike a BST). It only guarantees the root is min/max and the parent-child relationship — siblings have no order. Inorder traversal of a heap is *not* sorted.

### 1.2 Array representation (the clever part)
A complete binary tree is stored compactly in an **array** — no node objects/pointers needed:
```
For a node at index i (0-based):
  left child  = 2i + 1
  right child = 2i + 2
  parent      = (i - 1) / 2

Heap [10, 20, 15, 30, 25, 18]  represents:
        10 (i=0)
       /      \
   20 (i=1)   15 (i=2)
   /   \       /
 30(3) 25(4) 18(5)
```
> The array layout makes heaps **cache-friendly** (Phase 0.2) and memory-efficient — a key reason they're fast.

### 1.3 Core operations
**Insert (sift up / bubble up):** add at the end, then swap up while smaller than parent (min-heap) → O(log n).
**Extract-min (sift down / bubble down):** remove root, move last element to root, swap down with the smaller child → O(log n).
```java
public class MinHeap {
    private int[] heap = new int[16];
    private int size = 0;

    public void insert(int val) {
        if (size == heap.length) heap = java.util.Arrays.copyOf(heap, size * 2);
        heap[size] = val;
        siftUp(size++);                       // O(log n)
    }
    public int extractMin() {
        int min = heap[0];
        heap[0] = heap[--size];               // move last to root
        siftDown(0);                          // O(log n)
        return min;
    }
    private void siftUp(int i) {
        while (i > 0 && heap[i] < heap[(i-1)/2]) {   // smaller than parent?
            swap(i, (i-1)/2);
            i = (i-1)/2;
        }
    }
    private void siftDown(int i) {
        while (2*i + 1 < size) {
            int child = 2*i + 1;                            // left child
            if (child+1 < size && heap[child+1] < heap[child]) child++;  // smaller child
            if (heap[i] <= heap[child]) break;
            swap(i, child);
            i = child;
        }
    }
    private void swap(int a, int b) { int t = heap[a]; heap[a] = heap[b]; heap[b] = t; }
}
```

### 1.4 Complexity & heapify
| Operation | Complexity |
|-----------|-----------|
| Peek (min/max) | **O(1)** |
| Insert | O(log n) |
| Extract min/max | O(log n) |
| **Build heap (heapify)** from n elements | **O(n)** (not O(n log n)!) |
| Search arbitrary element | O(n) |
> **Heapify** building a heap from an array is O(n) (a clever bottom-up sift-down), not O(n log n) — a common surprise. **Heap sort** uses a heap to sort in O(n log n) (Phase 2.3).

### 1.5 In Java & uses
```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();              // Phase 1.6
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
```
**Uses:** priority queues, **Dijkstra's shortest path** (Phase 2.3), **top-K** problems, **merge K sorted lists**, **median of a stream** (two heaps), scheduling.

---

## 2. Tries (Prefix Trees)

A **trie** (pronounced "try") is a tree for storing **strings** where each node represents a **character**, and paths from the root spell out words. It excels at **prefix-based** operations.
```
Insert "cat", "car", "card", "dog":
         root
        /    \
       c      d
       |      |
       a      o
      / \     |
     t   r    g*
     *   |\
         d *
         |
         *      (* = end of a word)
```

### 2.1 Implementation
```java
public class Trie {
    private static class Node {
        Map<Character, Node> children = new HashMap<>();   // or Node[26] for a-z
        boolean isEndOfWord = false;
    }
    private final Node root = new Node();

    public void insert(String word) {                      // O(L), L = word length
        Node cur = root;
        for (char c : word.toCharArray())
            cur = cur.children.computeIfAbsent(c, k -> new Node());
        cur.isEndOfWord = true;
    }
    public boolean search(String word) {                   // O(L)
        Node node = find(word);
        return node != null && node.isEndOfWord;
    }
    public boolean startsWith(String prefix) {             // O(L) — the trie's superpower
        return find(prefix) != null;
    }
    private Node find(String s) {
        Node cur = root;
        for (char c : s.toCharArray()) {
            cur = cur.children.get(c);
            if (cur == null) return null;
        }
        return cur;
    }
}
```

### 2.2 Complexity & uses
| Operation | Complexity |
|-----------|-----------|
| Insert / search / startsWith | **O(L)** (L = string length — *independent of how many words!*) |
| Space | Can be high (a node per character) |
> A trie's killer feature: lookup time depends on the **word length**, not the **number of words** — so prefix search stays fast in a huge dictionary. **Uses:** autocomplete, spell-checkers, IP routing, prefix matching, word games. (Trade-off: high memory.)

---

## 3. Segment Tree (Range Queries with Updates)

A **segment tree** answers **range queries** (sum, min, max over `[l, r]`) **and** supports **point/range updates**, both in **O(log n)**.
```
Array [1, 3, 5, 7] -> segment tree where each node stores a range's aggregate:
              [0-3] sum=16
             /          \
        [0-1] sum=4    [2-3] sum=12
        /     \         /     \
     [0]=1  [1]=3    [2]=5   [3]=7
```
| Operation | Complexity |
|-----------|-----------|
| Build | O(n) |
| Range query (sum/min/max) | O(log n) |
| Update | O(log n) |
> **vs prefix sums** (Phase 2.2 arrays): prefix sums give O(1) range queries but O(n) updates. Segment trees give O(log n) for **both** — use them when the array **changes** between queries. Mostly a competitive-programming/analytics tool.

---

## 4. Fenwick Tree (Binary Indexed Tree / BIT)

A **Fenwick tree** is a simpler, more compact structure for **prefix sums with updates** — O(log n) for both, using clever bit manipulation (recall Phase 0.1/1.2 bitwise).
| | Segment tree | Fenwick tree |
|---|--------------|--------------|
| Queries | sum/min/max/etc. | Mainly prefix sums |
| Code complexity | More | Less (very compact) |
| Memory | ~2-4n | n |
> Use a **Fenwick tree** for cumulative-sum-with-updates (e.g., running totals, frequency counts). It's the go-to when you only need prefix sums and want minimal code. (Awareness-level; competitive programming.)

---

## 5. B-Tree & B+ Tree (Database Indexes — Conceptual)

**B-Trees** and **B+ Trees** are **self-balancing trees optimized for disk** (not RAM). Unlike binary trees, each node holds **many keys** and has **many children** (high "fan-out") — minimizing the number of slow disk reads.

### 5.1 Why high fan-out?
```
Disk reads are slow (Phase 0.1). A binary tree of 1M items is ~20 levels deep -> ~20 disk reads.
A B-tree with 100 keys/node is only ~3 levels deep -> ~3 disk reads.
```
- Each node = one **disk page** (~4-16 KB). Reading a node loads many keys at once.
- Fewer levels → **fewer disk I/Os** → much faster for disk-resident data.

### 5.2 B-Tree vs B+ Tree
| | B-Tree | B+ Tree |
|---|--------|---------|
| Data location | In internal nodes + leaves | **Only in leaves** |
| Leaf links | No | **Leaves linked** (fast range scans) |
| Range queries | Slower | **Fast** (traverse linked leaves) |
| Used by | Some DBs/filesystems | **Most relational DB indexes** (MySQL InnoDB, PostgreSQL) |
> ⚠️ **This is why database indexes make queries fast** (Phase 4.4): a B+ tree index turns an O(n) full-table scan into an O(log n) lookup, and its linked leaves make **range queries** (`WHERE x BETWEEN a AND b`, `ORDER BY`) efficient. You won't implement one, but understanding it explains indexing.

---

## 6. Summary: Which Specialized Structure?

| Need | Structure |
|------|-----------|
| Fast min/max, priority | **Heap** (PriorityQueue) |
| Prefix/autocomplete on strings | **Trie** |
| Range queries + updates (in memory) | **Segment tree** |
| Prefix sums + updates (compact) | **Fenwick tree** |
| Disk-based index, range scans | **B+ Tree** (databases) |
| Sorted in-memory map + ranges | **Red-Black tree** (TreeMap, prev note) |

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Thinking a heap is sorted | It only guarantees root = min/max |
| Expecting O(log n) build for a heap | Heapify is O(n) |
| Using a heap to search for an arbitrary element | That's O(n); heaps are for min/max |
| Underestimating trie memory | Tries trade space for fast prefix ops |
| Prefix sums when the array changes | Use a segment/Fenwick tree (O(log n) updates) |
| Thinking DB indexes are binary trees | They're B+ trees (high fan-out, disk-optimized) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Priority queues / heaps:** Dijkstra (Phase 2.3), top-K analytics, scheduling, rate limiting (Phase 12).
- **Tries:** autocomplete/search-suggest features (Phase 16.4 Elasticsearch does this at scale).
- **B+ Trees:** the foundation of **database indexing** — *the* most impactful backend performance lever (Phase 4.4, 13). EXPLAIN plans, index design, and query tuning all rest on this.
- **Segment/Fenwick:** analytics/reporting with frequent updates (rarely hand-coded; awareness).
- **Heaps in PriorityQueue:** thread scheduling, delay queues (Phase 1.6/1.10).

---

## 9. Quick Self-Check Questions

1. What's the heap property, and is a heap sorted?
2. How is a binary heap stored in an array (child/parent index formulas)?
3. Why is building a heap O(n) but inserting one element O(log n)?
4. What makes a trie's lookup independent of the number of words?
5. When use a segment tree vs prefix sums?
6. How does a Fenwick tree differ from a segment tree?
7. Why do B-trees have high fan-out, and why does that matter for databases?
8. What's the difference between a B-tree and a B+ tree, and which do DB indexes use?

---

## 10. Key Terms Glossary

- **Binary heap:** complete binary tree with the heap property (array-backed).
- **Min-heap / max-heap:** root is smallest / largest.
- **Sift up/down (heapify):** restore the heap property; building is O(n).
- **Priority queue:** heap-backed queue ordered by priority.
- **Trie (prefix tree):** character-node tree for string prefix operations.
- **Segment tree:** range queries + updates in O(log n).
- **Fenwick tree (BIT):** compact prefix-sum-with-updates structure.
- **B-Tree / B+ Tree:** high-fan-out, disk-optimized balanced trees (DB indexes).
- **Fan-out:** number of children per node (high → shallow tree → fewer disk reads).

---

*Previous topic: **Trees: Binary, BST & Balanced**.*
*Next topic: **Graphs (Representation & Traversals)**.*
