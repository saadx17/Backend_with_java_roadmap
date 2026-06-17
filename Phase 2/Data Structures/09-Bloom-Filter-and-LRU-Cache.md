# Bloom Filter & LRU Cache

> **Phase 2 — Data Structures & Algorithms → 2.2 Data Structures**
> Goal: Master two high-impact applied structures — the Bloom filter (probabilistic membership) and the LRU cache (eviction strategy) — both widely used in real backend systems.

---

## 0. The Big Picture

These two structures appear constantly in backend systems:
- **Bloom filter** — answers "is X *probably* in the set?" using tiny memory (probabilistic).
- **LRU cache** — a fixed-size cache that evicts the **least recently used** item when full.

Both are practical, interview-relevant, and combine concepts from earlier notes (hashing, hash maps, doubly linked lists).

---

## 1. Bloom Filter (Probabilistic Membership)

A **Bloom filter** is a **space-efficient probabilistic** data structure that tests whether an element is **possibly in a set** or **definitely not**. It can have **false positives** ("might be present" when it isn't) but **never false negatives** ("definitely not present" is always correct).

```
Query "is X in the set?"
  -> "Definitely NOT present"   (100% accurate — no false negatives)
  -> "Possibly present"          (might be a false positive)
```

### 1.1 How it works
- A **bit array** of size m (all 0s initially) + **k independent hash functions**.
- **Add(x):** hash x with all k functions → set those k bits to 1.
- **Contains(x):** hash x with all k functions → if **any** of those bits is 0 → **definitely not present**; if **all** are 1 → **possibly present** (could be coincidental overlap from other elements).
```
m=10 bits, k=3 hashes. Add "cat" -> bits {1,4,7}=1. Add "dog" -> bits {2,4,9}=1.
Query "cat": bits 1,4,7 all 1 -> "possibly present" ✓
Query "fox": hashes to {3,4,8}; bit 3 is 0 -> "definitely NOT present" ✓
Query "bird": hashes to {1,2,7}; all happen to be 1 (set by cat/dog) -> FALSE POSITIVE
```

### 1.2 Simplified implementation
```java
public class BloomFilter {
    private final BitSet bits;
    private final int size;
    private final int numHashes;

    public BloomFilter(int size, int numHashes) {
        this.size = size;
        this.numHashes = numHashes;
        this.bits = new BitSet(size);
    }
    public void add(String item) {
        for (int i = 0; i < numHashes; i++)
            bits.set(hash(item, i));            // set k bits
    }
    public boolean mightContain(String item) {
        for (int i = 0; i < numHashes; i++)
            if (!bits.get(hash(item, i))) return false;   // any 0 -> definitely NOT present
        return true;                                       // all 1 -> POSSIBLY present
    }
    private int hash(String item, int seed) {
        return Math.abs((item.hashCode() ^ (seed * 0x9E3779B1)) % size);   // k "different" hashes
    }
}
```

### 1.3 Key properties
| Property | Value |
|----------|-------|
| Add / query time | **O(k)** (constant — k hashes) |
| Space | **Tiny** (a bit array — far less than storing elements) |
| False positives | Possible (tunable via m and k) |
| False negatives | **Never** |
| Deletion | **Not supported** (clearing a bit could affect other elements) — use a Counting Bloom Filter |
> The false-positive rate depends on the bit-array size (m), number of hashes (k), and elements added (n). Bigger array / tuned k → fewer false positives. You **trade accuracy for huge space savings**.

### 1.4 Real-world uses (very common!)
| System | Use |
|--------|-----|
| **Databases (Cassandra, HBase, RocksDB)** | "Is this key *maybe* on disk?" → skip the expensive disk read if "definitely not" |
| **Caching/CDNs** | Avoid cache lookups for keys definitely not cached |
| **Web crawlers** | "Have I seen this URL?" (avoid re-crawling) |
| **Spam/malware filters** | "Is this URL/email on the blocklist?" |
| **Username availability** | Quick "definitely available" check before a DB query |
> The pattern: use a Bloom filter as a **cheap first filter** to avoid expensive operations (disk/DB/network) for things that are **definitely not present**. A "possibly present" answer falls through to the real (expensive) check. (Redis has Bloom filter support — Phase 4.8.)

---

## 2. LRU Cache (Least Recently Used)

An **LRU cache** is a fixed-capacity cache that, when full, **evicts the least recently used item** to make room. It keeps "hot" (recently accessed) data and discards "cold" data — the most common cache eviction policy.

```
Capacity 3. Access order matters:
  put(A), put(B), put(C)   -> [C, B, A]  (C most recent, A least)
  get(A)                    -> [A, C, B]  (A now most recent)
  put(D)                    -> evict B (least recently used) -> [D, A, C]
```

### 2.1 The requirement: O(1) get and put
A good LRU cache needs **O(1)** for both `get` and `put`, including eviction. This requires **two structures combined**:
| Need | Structure |
|------|-----------|
| O(1) key lookup | **HashMap** (key → node) |
| O(1) move-to-front & evict-from-back | **Doubly linked list** (recency order) |
> This is the classic interview answer: **HashMap + Doubly Linked List**. The map gives O(1) access to any node; the doubly linked list maintains recency order with O(1) move/remove (recall Phase 2.2 doubly linked lists — O(1) deletion of a known node).

### 2.2 How they work together
```
HashMap:  key -> node (O(1) lookup)
Doubly linked list:  MRU (head) <-> ... <-> LRU (tail)
  get(key):  map.get -> node; move node to head (most recently used)
  put(key):  if exists, update & move to head; else add to head;
             if over capacity, remove the tail (LRU) and its map entry
```

### 2.3 Implementation 1: the easy way — LinkedHashMap (recall Phase 1.6)
Java's `LinkedHashMap` in **access-order mode** with `removeEldestEntry` *is* an LRU cache:
```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    public LRUCache(int capacity) {
        super(16, 0.75f, true);          // true = ACCESS order (not insertion)
        this.capacity = capacity;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;        // evict the LRU entry when over capacity
    }
}
// usage:
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "a"); cache.get(1); cache.put(2, "b"); // ...
```

### 2.4 Implementation 2: from scratch (the interview version)
```java
public class LRUCache {
    private static class Node {
        int key, value;
        Node prev, next;
        Node(int k, int v) { key = k; value = v; }
    }
    private final Map<Integer, Node> map = new HashMap<>();
    private final Node head = new Node(0, 0), tail = new Node(0, 0);  // dummy sentinels
    private final int capacity;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail; tail.prev = head;     // empty list between sentinels
    }
    public int get(int key) {
        Node node = map.get(key);
        if (node == null) return -1;
        moveToFront(node);                      // mark as most recently used
        return node.value;
    }
    public void put(int key, int value) {
        Node node = map.get(key);
        if (node != null) {
            node.value = value;
            moveToFront(node);
        } else {
            if (map.size() == capacity) {       // evict LRU (the node before tail)
                Node lru = tail.prev;
                remove(lru);
                map.remove(lru.key);
            }
            Node n = new Node(key, value);
            map.put(key, n);
            addToFront(n);
        }
    }
    private void addToFront(Node n) {           // insert right after head
        n.next = head.next; n.prev = head;
        head.next.prev = n; head.next = n;
    }
    private void remove(Node n) {               // unlink (O(1), known node)
        n.prev.next = n.next; n.next.prev = n.prev;
    }
    private void moveToFront(Node n) { remove(n); addToFront(n); }
}
```
- **`get`/`put`: O(1)** — map lookup + O(1) list relink.
- **Sentinel** (dummy head/tail) nodes simplify edge cases (no null checks).

### 2.5 Other eviction policies (awareness)
| Policy | Evicts |
|--------|--------|
| **LRU** | Least Recently Used |
| **LFU** | Least Frequently Used (by access count) |
| **FIFO** | Oldest inserted |
| **TTL** | Expired entries (time-based) |
> Real cache libraries (**Caffeine**, **Redis** — Phase 5.7/4.8) offer LRU/LFU/TTL and size-based eviction. Caffeine actually uses an advanced policy (W-TinyLFU) that often beats plain LRU.

---

## 3. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Treating a Bloom "possibly present" as certain | It can be a false positive — verify with the real source |
| Expecting Bloom filters to give false negatives | They never do (only false positives) |
| Trying to delete from a standard Bloom filter | Not supported — use a Counting Bloom Filter |
| LRU cache without O(1) eviction | Combine HashMap + doubly linked list |
| Forgetting access-order in LinkedHashMap LRU | Pass `true` to the constructor |
| Unbounded cache (no eviction) | Memory leak (Phase 1.11) — always bound caches |
| Using LRU when LFU/TTL fits better | Match the policy to the access pattern |

---

## 4. Connection to Backend / Spring (Why This Matters Later)

- **Caching (Phase 5.7, 13):** LRU/LFU/TTL eviction is core to in-memory caches (Caffeine, Spring Cache) and Redis (Phase 4.8) — bounded caches prevent OOM (recall Phase 1.11 leaks).
- **Bloom filters:** Redis Bloom, database read optimization (skip disk for "definitely not present"), deduplication, rate-limiting pre-checks (Phase 4.8, 12).
- **LinkedHashMap LRU** is a quick in-process cache for hot data.
- **Cache invalidation & eviction strategies** are a major backend design topic (Phase 13).
- These structures embody the **time-space trade-off** (Phase 2.1): Bloom trades accuracy for space; caches trade memory for speed.
- **Interview classics:** "design an LRU cache" and "what's a Bloom filter" are extremely common (Phase 2.4, system design Phase 16.8).

---

## 5. Quick Self-Check Questions

1. What two answers can a Bloom filter give, and which is always accurate?
2. How does a Bloom filter add and query an element (bits + hashes)?
3. Why can't you delete from a standard Bloom filter?
4. Give two real-world uses of Bloom filters and the pattern they follow.
5. What does an LRU cache do when it's full?
6. Why does an O(1) LRU cache need both a HashMap and a doubly linked list?
7. How do you implement an LRU cache with `LinkedHashMap`?
8. Name other eviction policies and when each fits.

---

## 6. Key Terms Glossary

- **Bloom filter:** probabilistic set-membership structure (bit array + k hashes).
- **False positive / false negative:** "present" when absent / "absent" when present (Bloom never has the latter).
- **Counting Bloom filter:** variant supporting deletion (counters instead of bits).
- **LRU cache:** fixed-size cache evicting the least recently used item.
- **Eviction policy (LRU/LFU/FIFO/TTL):** which item to remove when full.
- **HashMap + doubly linked list:** the O(1) LRU implementation.
- **Sentinel node:** dummy head/tail simplifying linked-list edge cases.
- **`LinkedHashMap` access-order:** built-in LRU mechanism (Phase 1.6).
- **Caffeine / Redis:** production cache libraries (Phase 5.7/4.8).

---

*Previous topic: **Graph Algorithms & Union-Find**.*
*This completes **Section 2.2 — Data Structures**.*
*Next section in roadmap: **2.3 Algorithms**.*
