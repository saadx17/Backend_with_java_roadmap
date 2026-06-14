# Map Implementations & HashMap Internals

> **Phase 1 — Java Language Mastery → 1.6 Java Collections Framework**
> Goal: Master the `Map` implementations and deeply understand HashMap internals (load factor, rehashing, treeification, collision resolution, and the equals/hashCode contract) — a top interview and performance topic.

---

## 0. The Big Picture

A **`Map<K,V>`** stores **key → value** pairs with **unique keys**. It's not a `Collection` (separate hierarchy, recall the overview note). Maps are everywhere: caches, lookups, counting, grouping, configuration.

```java
Map<String, Integer> ages = new HashMap<>();
ages.put("Alice", 30);
ages.put("Bob", 25);
ages.get("Alice");        // 30
ages.containsKey("Bob");  // true
ages.put("Alice", 31);    // overwrites — keys are unique
```

> **HashMap internals** are one of the most-asked Java interview topics *and* directly affect real performance — so this note goes deep.

---

## 1. Essential Map Operations

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);                     // insert/overwrite
map.get("a");                        // 1 (or null if absent)
map.getOrDefault("x", 0);            // 0 if absent (no null)
map.putIfAbsent("a", 99);            // only if key missing
map.remove("a");                     // remove by key
map.containsKey("a"); map.containsValue(1);
map.size(); map.isEmpty();

// Functional updates (very useful):
map.computeIfAbsent("list", k -> new ArrayList<>()).add(x);  // lazy init + use
map.merge("count", 1, Integer::sum);   // increment a counter (insert 1 or add 1)
map.compute("a", (k, v) -> v == null ? 1 : v + 1);

// Iteration (use entrySet for key+value):
for (Map.Entry<String, Integer> e : map.entrySet()) {
    e.getKey(); e.getValue();
}
map.forEach((k, v) -> System.out.println(k + "=" + v));
```
> `computeIfAbsent` and `merge` are idiomatic for **grouping** and **counting** — learn them; they replace verbose null-check patterns.

---

## 2. HashMap (the default) — Internals Deep Dive

`HashMap` stores entries in an **array of buckets**, placing each key by its hash. Average **O(1)** for get/put. Let's open the hood.

### 2.1 The structure
```
HashMap = array of buckets (Node[] table)
Each bucket holds entries whose keys hash to that index.
Within a bucket: a linked list (or, when large, a balanced tree).

  index:  0    1    2    3    4    5  ...
        [   ][A→B][   ][C ][   ][D→E→F]   <- bucket 5 has a collision chain
```

### 2.2 How `put(key, value)` works
```
1. Compute key.hashCode().
2. "Spread" the hash (HashMap mixes high/low bits to reduce collisions).
3. bucket index = hash & (capacity - 1)   (fast modulo; capacity is a power of 2)
4. Go to that bucket:
     - empty?            -> place the new entry
     - has entries?      -> walk them; if a key .equals() matches -> OVERWRITE value
                            else -> append the new entry (collision)
```

### 2.3 How `get(key)` works
```
1. Compute the bucket index from key's hash (same as put).
2. Walk that bucket; use equals() to find the matching key.
3. Return its value (or null if not found).
```

> **This is why `hashCode()` and `equals()` must be correct and consistent** (recall Phase 1.3 Object methods):
> - **`hashCode()`** picks the **bucket** (fast lookup).
> - **`equals()`** finds the **exact key** within the bucket.
> Break either and your keys get "lost" — `map.get(key)` returns null even though you put it in!

### 2.4 Initial capacity & load factor
| Parameter | Default | Meaning |
|-----------|---------|---------|
| **Initial capacity** | 16 | Number of buckets to start |
| **Load factor** | **0.75** | How full it gets before resizing |
| **Threshold** | capacity × load factor (16 × 0.75 = **12**) | Resize trigger |

When `size > threshold`, the map **resizes**.

### 2.5 Rehashing (resizing)
```
When entries exceed threshold (e.g., 12 of 16):
  1. Capacity DOUBLES (16 -> 32).
  2. ALL entries are REDISTRIBUTED into the new, larger bucket array (rehashed).
  3. Threshold updates (32 × 0.75 = 24).
```
- Resizing is **O(n)** (one-time cost spread over many inserts → amortized O(1)).
- **Pre-size** to avoid repeated resizing if you know the size:
  ```java
  new HashMap<>(expectedSize / 0.75 + 1);   // avoid resizes
  ```
> The **0.75 load factor** balances space vs collisions: higher = less memory but more collisions; lower = more memory but faster lookups. The default is a well-tuned compromise.

### 2.6 Collision resolution & treeification
When multiple keys land in the same bucket (a **collision**), HashMap chains them:
- **Default: a linked list** within the bucket → lookup within the bucket is O(k).
- **Treeification (Java 8+):** if a single bucket's chain exceeds **8 entries** (TREEIFY_THRESHOLD) **and** capacity ≥ 64, that bucket converts to a **balanced red-black tree** → bucket lookup becomes **O(log k)** instead of O(k).
- If the bucket shrinks below **6** (UNTREEIFY_THRESHOLD), it converts back to a list.

```
Normal bucket:    list  -> O(k) worst case within bucket
Big bucket (≥8):  tree  -> O(log k)   (defends against hash-collision attacks / bad hashCodes)
```
> Treeification was added in Java 8 to prevent worst-case O(n) lookups (e.g., DoS via crafted colliding keys, or terrible `hashCode()` implementations).

### 2.7 HashMap performance summary
| Operation | Average | Worst case (bad hashing) |
|-----------|---------|--------------------------|
| `get` / `put` / `containsKey` | **O(1)** | O(log n) with treeification (was O(n) pre-Java 8) |

### 2.8 Other HashMap facts
- Allows **one null key** and multiple null values.
- **Not thread-safe** — concurrent modification can corrupt it or cause infinite loops (pre-Java 8); use `ConcurrentHashMap` for concurrency (§6).
- **No ordering** guarantee for iteration.

---

## 3. LinkedHashMap (insertion/access order)

Extends `HashMap` and maintains a **linked list** through entries, preserving **insertion order** (or **access order**) on iteration.
```java
Map<String, Integer> m = new LinkedHashMap<>();  // iterates in insertion order
```
### 3.1 Access-order mode → easy LRU cache
With access-order mode and an overridable `removeEldestEntry`, `LinkedHashMap` makes a simple **LRU cache**:
```java
Map<K, V> lru = new LinkedHashMap<>(16, 0.75f, true) {  // true = access order
    protected boolean removeEldestEntry(Map.Entry<K,V> e) {
        return size() > MAX_ENTRIES;   // evict least-recently-used when over capacity
    }
};
```
> This is the classic **LRU cache** implementation (recall Phase 2 LRU Cache) — and a common interview question.

---

## 4. TreeMap (sorted by key)

Backed by a **Red-Black tree**; keeps **keys sorted** (natural order or `Comparator`). Operations are **O(log n)**.
```java
TreeMap<String, Integer> tm = new TreeMap<>();   // sorted by key
tm.firstKey(); tm.lastKey();
tm.floorKey("m");    // largest key <= "m"
tm.ceilingKey("m");  // smallest key >= "m"
tm.headMap("m");     // keys < "m"
tm.subMap("a", "m"); // range view
```
- Implements `NavigableMap` (range/navigation queries, like `TreeSet`).
- Keys must be `Comparable` or have a `Comparator`; **no null keys**.
> **Use `TreeMap`** when you need keys kept sorted or range queries.

---

## 5. Hashtable (legacy — avoid)

`Hashtable` is the old (Java 1.0), **synchronized** `HashMap` equivalent. Avoid it:
- Every method is synchronized (slow), and it's legacy.
- Doesn't allow null keys/values.
> **For thread safety, use `ConcurrentHashMap`** (far better), not `Hashtable`.

---

## 6. ConcurrentHashMap (thread-safe, high-performance)

A **thread-safe** map designed for concurrency, far superior to `Hashtable`:
- Uses **fine-grained locking / CAS** on individual bins (not one global lock) → high concurrent throughput.
- Reads are generally **lock-free**; writes lock only the relevant bucket.
- Does **not allow null** keys or values (ambiguity in concurrent context).
- Provides atomic operations: `compute`, `merge`, `computeIfAbsent`, `putIfAbsent`.
```java
ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();
counts.merge("hits", 1, Integer::sum);   // thread-safe atomic increment
```
> **Use `ConcurrentHashMap`** whenever a map is shared across threads (caches, counters, registries) — Phase 1.10. It's one of the most important concurrent collections.

---

## 7. Specialized Maps

| Map | Key feature | Use when |
|-----|-------------|----------|
| **EnumMap** | Array indexed by enum ordinal — very fast & compact | Keys are enum constants (always prefer over HashMap) |
| **WeakHashMap** | Keys held by **weak references** → entries GC'd when key unreferenced elsewhere | Caches/metadata that shouldn't prevent GC |
| **IdentityHashMap** | Uses `==` (reference equality), not `equals()` | Rare; identity-based keying |

```java
EnumMap<Day, String> plan = new EnumMap<>(Day.class);  // fast enum-keyed map
```
> **EnumMap** (recall Phase 1.3 Enums) is the go-to for enum keys. **WeakHashMap** avoids memory leaks in caches (keys are reclaimed when no longer used elsewhere — ties to Phase 1.11 references).

---

## 8. Immutable Maps

```java
Map<String, Integer> m = Map.of("a", 1, "b", 2);        // immutable (up to 10 pairs)
Map<String, Integer> big = Map.ofEntries(                // for many pairs
    Map.entry("a", 1), Map.entry("b", 2));
Map<String, Integer> copy = Map.copyOf(existing);        // immutable copy
Map<String, Integer> view = Collections.unmodifiableMap(backing);  // read-only view
```
- `Map.of(...)` is immutable, rejects null, and throws on duplicate keys.

---

## 9. Map Implementations Compared

| Implementation | Ordering | Backing | get/put | Nulls | Thread-safe | Use when |
|----------------|----------|---------|---------|-------|-------------|----------|
| **HashMap** | None | Hash table | O(1) avg | 1 null key | No | Default |
| **LinkedHashMap** | Insertion/access | Hash + linked list | O(1) avg | 1 null key | No | Ordered iteration / LRU |
| **TreeMap** | Sorted by key | Red-Black tree | O(log n) | No null key | No | Sorted / range queries |
| **Hashtable** | None | Hash table | O(1) | No nulls | Yes (legacy) | Don't use |
| **ConcurrentHashMap** | None | Hash + CAS/locks | O(1) avg | No nulls | **Yes** | Concurrent access |
| **EnumMap** | Enum order | Array | O(1) | No null key | No | Enum keys |
| **WeakHashMap** | None | Hash + weak keys | O(1) | 1 null key | No | GC-sensitive caches |

---

## 10. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Keys without proper `equals`/`hashCode` | Override them (or use records) — else `get` returns null |
| Mutating a key after inserting (hashCode changes) | Use immutable keys |
| Using `HashMap` across threads | Use `ConcurrentHashMap` |
| Using `Hashtable` for thread safety | Use `ConcurrentHashMap` |
| `null` keys/values in `ConcurrentHashMap`/`TreeMap` | Not allowed |
| Not pre-sizing a large HashMap | Pass initial capacity to avoid rehashing |
| Verbose null-check counting/grouping | Use `merge` / `computeIfAbsent` |
| Using `HashMap` for enum keys | Use `EnumMap` |
| Expecting HashMap iteration order | It's unordered (use LinkedHashMap/TreeMap) |

---

## 11. Connection to Backend / Spring (Why This Matters Later)

- **Caching:** in-memory caches are maps; `ConcurrentHashMap` for thread-safe caches; `LinkedHashMap` for LRU (Phase 5.7, 13).
- **HashMap internals** explain why **entity `equals`/`hashCode`** matters when entities are map keys / in sets (Phase 5.4 JPA pitfalls).
- **`ConcurrentHashMap`** is a staple of multi-threaded backends — counters, registries, rate limiters (Phase 1.10, 12).
- **`computeIfAbsent`/`merge`** are idiomatic for grouping/counting (often replaced by `Collectors.groupingBy` in streams — Phase 1.8).
- **`EnumMap`** for per-enum config; **immutable maps** for constants (Phase 1.3).
- **Performance tuning** (load factor, pre-sizing) matters in hot paths (Phase 13).

---

## 12. Quick Self-Check Questions

1. Why is `Map` not part of the `Collection` hierarchy?
2. Walk through how `HashMap.put` and `get` work internally.
3. What roles do `hashCode()` and `equals()` play in a HashMap?
4. What are the default initial capacity and load factor, and what's the resize threshold?
5. What is rehashing, and when does it happen?
6. What is treeification, when does it trigger, and why was it added?
7. How does `LinkedHashMap` enable an LRU cache?
8. Why use `ConcurrentHashMap` instead of `Hashtable` or a synchronized HashMap?
9. When would you use `EnumMap`, `TreeMap`, and `WeakHashMap`?

---

## 13. Key Terms Glossary

- **`Map`:** key→value store with unique keys.
- **Bucket / bin:** a slot in the hash table's array.
- **Hash / spread:** key's hashCode, mixed to choose a bucket.
- **Load factor (0.75):** fullness threshold before resizing.
- **Rehashing / resize:** doubling capacity and redistributing entries.
- **Collision:** multiple keys mapping to the same bucket.
- **Treeification:** converting a long bucket chain to a red-black tree (≥8 entries).
- **`computeIfAbsent` / `merge`:** functional update helpers.
- **`LinkedHashMap`:** insertion/access-ordered map (LRU-capable).
- **`TreeMap` / `NavigableMap`:** sorted map with range queries.
- **`ConcurrentHashMap`:** thread-safe high-performance map.
- **`EnumMap` / `WeakHashMap` / `IdentityHashMap`:** enum-keyed / weak-key / identity-keyed maps.
- **Immutable map:** unmodifiable map (`Map.of`, `Map.copyOf`).

---

*Previous topic: **Queue & Deque Implementations**.*
*Next topic: **Comparable/Comparator, Iterators & Choosing the Right Collection**.*
