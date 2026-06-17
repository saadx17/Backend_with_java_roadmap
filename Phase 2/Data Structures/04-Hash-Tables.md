# Hash Tables

> **Phase 2 — Data Structures & Algorithms → 2.2 Data Structures**
> Goal: Master hash tables from the ground up — hash functions, collision resolution (chaining & open addressing), load factor & resizing — and implement a simplified HashMap.

---

## 0. The Big Picture

A **hash table** stores **key→value** pairs with **average O(1)** insertion, lookup, and deletion — the fastest general-purpose lookup structure. It works by converting a key into an array index via a **hash function**.

```
key  --hash function-->  index  -->  store value in array[index]
"name" --hash--> 3  ->  bucket[3] = "Alice"
get("name") -> hash -> 3 -> read bucket[3] -> "Alice"   (all O(1) average)
```

> This deepens **Phase 1.6** (HashMap internals) by building one from scratch. Hash tables are the backbone of caches, indexes, deduplication, and counting — used in virtually every backend.

---

## 1. The Core Idea: Mapping Keys to Indices

An array gives O(1) access *by index*. A hash table extends this to O(1) access *by key* by computing an index from the key:
```
index = hash(key) % capacity
```
If `hash` distributes keys evenly, each key maps to a distinct bucket → direct O(1) access.

---

## 2. Hash Functions

A **hash function** maps a key (of any type) to an integer (the **hash code**), which is then reduced to an array index.

### 2.1 Properties of a good hash function
| Property | Why |
|----------|-----|
| **Deterministic** | Same key → same hash, always (else you can't find it back) |
| **Uniform distribution** | Spreads keys evenly → fewer collisions |
| **Fast to compute** | It's called on every operation |
| **Avalanche** | Small key change → big hash change (reduces clustering) |

### 2.2 In Java: hashCode() + equals() (recall Phase 1.3/1.6)
Java keys provide `hashCode()` (the hash) and `equals()` (to confirm a match within a bucket):
```java
class Point {
    int x, y;
    @Override public int hashCode() { return Objects.hash(x, y); }   // hash
    @Override public boolean equals(Object o) { /* compare x,y */ }   // confirm
}
```
> ⚠️ **The hashCode/equals contract** (Phase 1.3) is mandatory: equal objects MUST have equal hash codes. Break it and keys get "lost" in the table. This is *why* you override both together.

### 2.3 Reducing a hash to an index
```java
int index = (key.hashCode() & 0x7FFFFFFF) % capacity;   // & makes it non-negative
// Java's HashMap uses: hash spreading + (capacity - 1) & hash  (capacity is power of 2)
```

---

## 3. Collisions (Unavoidable)

A **collision** occurs when two different keys hash to the **same index**. By the **pigeonhole principle**, collisions are inevitable (more possible keys than buckets). The two main resolution strategies:

---

## 4. Collision Resolution 1: Separate Chaining

Each bucket holds a **list** (chain) of entries that hashed there. On collision, append to that bucket's list.
```
bucket[3] -> ["name"→"Alice"] -> ["game"→"Chess"]   (both hashed to 3)
get("game"): hash->3, walk the chain, equals-match "game" -> "Chess"
```
- **Lookup:** hash to the bucket, then scan the chain with `equals` (O(k) for chain length k).
- Java's `HashMap` uses chaining, and **converts long chains (≥8) to balanced trees** → O(log k) (treeification, recall Phase 1.6).

### 4.1 Pros/cons
| Pros | Cons |
|------|------|
| Simple; handles high load gracefully | Extra memory (list nodes/pointers) |
| Easy deletion | Poor cache locality (chains scattered) |

---

## 5. Collision Resolution 2: Open Addressing

All entries live **in the array itself** — on collision, **probe** for the next empty slot. No separate lists.

### 5.1 Probing strategies
| Strategy | Next slot |
|----------|-----------|
| **Linear probing** | `(index + 1) % capacity`, `(index + 2) % capacity`, ... |
| **Quadratic probing** | `(index + 1²)`, `(index + 2²)`, ... (reduces clustering) |
| **Double hashing** | Use a second hash function for the step size |
```
Linear probing — insert "game" but bucket[3] is taken:
  bucket[3] full -> try bucket[4] -> empty -> place there
```

### 5.2 Pros/cons
| Pros | Cons |
|------|------|
| Better cache locality (all in one array) | **Clustering** (linear probing) degrades performance |
| No pointer overhead | Deletion is tricky (need tombstones) |
| | Must keep load factor low |

> **Chaining vs open addressing:** Java's `HashMap` uses **chaining** (with treeification). Open addressing (e.g., `IdentityHashMap`, and many high-performance maps) trades complexity for cache locality. Both are valid; chaining is more common in Java.

---

## 6. Load Factor & Resizing (recall Phase 1.6)

The **load factor** = `size / capacity` — how full the table is. As it rises, collisions increase and operations slow.

### 6.1 The threshold
```
load factor = number of entries / number of buckets
Java default: 0.75   -> resize when size > capacity × 0.75
```
- When the load factor exceeds the threshold, the table **resizes**: capacity **doubles** and **all entries are rehashed** into the new, larger array → O(n) (one-time, amortized O(1) per insert — Phase 2.1).
> **Why 0.75?** A balance: higher → less memory but more collisions; lower → fewer collisions but wasted space (Phase 1.6). Resizing keeps chains short and operations near O(1).

### 6.2 Rehashing
On resize, every key's index is recomputed (`hash % newCapacity`) and entries redistributed — because the modulo changes with capacity.

---

## 7. Implementing a Simplified HashMap (Chaining)

```java
public class SimpleHashMap<K, V> {
    private static class Entry<K, V> {
        K key; V value; Entry<K, V> next;
        Entry(K key, V value) { this.key = key; this.value = value; }
    }

    private Entry<K, V>[] buckets;
    private int size = 0;
    private static final double LOAD_FACTOR = 0.75;

    @SuppressWarnings("unchecked")
    public SimpleHashMap() { buckets = new Entry[16]; }

    private int indexFor(K key) {
        return (key.hashCode() & 0x7FFFFFFF) % buckets.length;   // non-negative index
    }

    public void put(K key, V value) {
        if ((double) size / buckets.length >= LOAD_FACTOR) resize();
        int i = indexFor(key);
        for (Entry<K, V> e = buckets[i]; e != null; e = e.next) {
            if (e.key.equals(key)) { e.value = value; return; }   // update existing
        }
        Entry<K, V> e = new Entry<>(key, value);                  // insert at head
        e.next = buckets[i];
        buckets[i] = e;
        size++;
    }

    public V get(K key) {
        int i = indexFor(key);
        for (Entry<K, V> e = buckets[i]; e != null; e = e.next) {
            if (e.key.equals(key)) return e.value;                // hash to bucket, equals to find
        }
        return null;
    }

    public void remove(K key) {
        int i = indexFor(key);
        Entry<K, V> prev = null;
        for (Entry<K, V> e = buckets[i]; e != null; prev = e, e = e.next) {
            if (e.key.equals(key)) {
                if (prev == null) buckets[i] = e.next;            // remove head
                else prev.next = e.next;                          // unlink
                size--;
                return;
            }
        }
    }

    @SuppressWarnings("unchecked")
    private void resize() {
        Entry<K, V>[] old = buckets;
        buckets = new Entry[old.length * 2];                      // double
        size = 0;
        for (Entry<K, V> head : old)                              // rehash all entries
            for (Entry<K, V> e = head; e != null; e = e.next)
                put(e.key, e.value);
    }
}
```
> This shows the whole mechanism: hash → bucket → chain walk with `equals` → resize at load factor. (Java's real HashMap adds hash spreading, treeification, and power-of-2 capacity — Phase 1.6.)

---

## 8. Complexity Summary

| Operation | Average | Worst case |
|-----------|---------|------------|
| Insert / get / remove | **O(1)** | O(n) (all collide) → O(log n) with treeification |
| Space | O(n) | O(n) |
> **O(1) is *average*, assuming a good hash function.** A bad hash (everything collides) degrades to O(n) — which is why hash quality and treeification matter (Phase 1.6). This is a **probabilistic** guarantee, not absolute.

---

## 9. Hash Tables in Java (recap Phase 1.6)

| Structure | Notes |
|-----------|-------|
| `HashMap` | Default; chaining + treeification; 1 null key |
| `LinkedHashMap` | Maintains insertion/access order (LRU) |
| `ConcurrentHashMap` | Thread-safe (Phase 1.10) |
| `HashSet` | A `Set` backed by a `HashMap` |
| `Hashtable` | Legacy, synchronized — avoid |

---

## 10. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Keys without proper `equals`/`hashCode` | Override both (or use records) — else keys are lost |
| Mutable keys whose hashCode changes after insert | Use immutable keys |
| Expecting guaranteed O(1) | It's *average*; bad hashing → O(n)/O(log n) |
| Forgetting collisions are inevitable | Use a resolution strategy |
| Not resizing (high load factor) | Long chains → slow; resize at ~0.75 |
| Negative index from `hashCode` | Mask with `& 0x7FFFFFFF` |
| Assuming iteration order | HashMap has none (use LinkedHashMap/TreeMap) |

---

## 11. Connection to Backend / Spring (Why This Matters Later)

- **Caching** (Phase 5.7): caches are hash tables; `ConcurrentHashMap` for thread-safe in-memory caches.
- **Database indexes** (Phase 4.4): hash indexes give O(1) equality lookups (vs B-tree for ranges — Phase 2.2 trees).
- **Deduplication & counting**: `HashSet`/`HashMap` + `merge`/`computeIfAbsent` (Phase 1.6).
- **Two-sum / frequency / grouping** patterns rely on hash tables (Phase 2.1 time-space trade-off, 1.8 groupingBy).
- **Redis** (Phase 4.8) is essentially a distributed hash table.
- **Bloom filters** (Phase 2.2) extend hashing for probabilistic membership.
- **Entity `equals`/`hashCode`** correctness is critical for JPA + collections (Phase 5.4).

---

## 12. Quick Self-Check Questions

1. How does a hash table achieve average O(1) lookup?
2. What makes a good hash function?
3. Why are collisions inevitable, and what are the two main resolution strategies?
4. How does separate chaining work? Open addressing?
5. What is the load factor, why 0.75, and what happens when it's exceeded?
6. What is rehashing and why is it needed on resize?
7. Why is hash-table O(1) "average" and not guaranteed?
8. Why must keys have correct, consistent `equals`/`hashCode`?

---

## 13. Key Terms Glossary

- **Hash table:** key→value store with average O(1) operations.
- **Hash function:** maps a key to an integer (hash code).
- **Bucket:** an array slot holding entries for a hash.
- **Collision:** two keys mapping to the same index.
- **Separate chaining:** buckets hold lists of colliding entries.
- **Open addressing:** probe for the next open slot (linear/quadratic/double hashing).
- **Clustering:** runs of filled slots degrading open addressing.
- **Load factor:** size / capacity (resize threshold).
- **Rehashing:** recomputing indices on resize.
- **Treeification:** converting long chains to trees (Java, Phase 1.6).
- **Tombstone:** deletion marker in open addressing.

---

*Previous topic: **Stacks & Queues**.*
*Next topic: **Trees (Binary Trees, BST, Balanced Trees)**.*
