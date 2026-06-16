# List Implementations

> **Phase 1 — Java Language Mastery → 1.6 Java Collections Framework**
> Goal: Master the `List` implementations — `ArrayList`, `LinkedList`, `Vector`, `Stack`, `CopyOnWriteArrayList`, and immutable lists — their internals, performance, and when to use each.

---

## 0. The Big Picture

A **`List<E>`** is an **ordered** collection that allows **duplicates** and provides **index-based access** (`get(i)`, `add(i, e)`). It's the most-used collection type.

```java
List<String> names = new ArrayList<>();
names.add("Alice");          // index 0
names.add("Bob");            // index 1
names.add("Alice");          // duplicates allowed
names.get(0);                // "Alice"
names.size();                // 3
```

| List trait | Value |
|------------|-------|
| Ordered? | ✅ Insertion order preserved |
| Duplicates? | ✅ Allowed |
| Indexed access? | ✅ Yes (`get(i)`) |
| Null elements? | ✅ Allowed (most impls) |

---

## 1. ArrayList (the default choice)

`ArrayList` is backed by a **dynamic (resizable) array**. It's the go-to List for most use cases.

### 1.1 How it works
- Stores elements in an internal `Object[]`.
- **Random access is O(1)** — direct index into the array (cache-friendly, recall Phase 0.2 cache locality).
- When full, it **grows** by allocating a larger array (typically ~1.5×) and copying — **amortized O(1)** for `add` at the end.

### 1.2 Performance
| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `get(i)` / `set(i)` | **O(1)** | Direct array index |
| `add(e)` (at end) | **Amortized O(1)** | Occasional resize/copy |
| `add(i, e)` (middle) | O(n) | Shifts elements right |
| `remove(i)` (middle) | O(n) | Shifts elements left |
| `contains(e)` / `indexOf` | O(n) | Linear scan |

### 1.3 Capacity tuning
Like `StringBuilder` (Phase 1.4), pre-sizing avoids repeated resizing:
```java
List<Integer> list = new ArrayList<>(10_000);  // initial capacity hint
```
> **Use `ArrayList` by default.** It's best for: indexed access, iteration, and appending — which covers ~90% of use cases.

---

## 2. LinkedList (doubly-linked list)

`LinkedList` is a **doubly-linked list** — each element (node) holds references to the previous and next nodes. It also implements `Deque` (so it can act as a queue/stack).

### 2.1 Performance
| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `get(i)` | **O(n)** | Must walk the chain — no random access! |
| `add`/`remove` at ends (first/last) | **O(1)** | Just relink pointers |
| `add`/`remove` in middle (with iterator) | O(1) at the node | But O(n) to *find* the node |
| `contains` | O(n) | Linear scan |

### 2.2 ArrayList vs LinkedList (the classic comparison)
| Aspect | ArrayList | LinkedList |
|--------|-----------|------------|
| Backing structure | Dynamic array | Doubly-linked nodes |
| Random access `get(i)` | **O(1)** | O(n) |
| Add/remove at ends | Amortized O(1) (end) | O(1) (both ends) |
| Add/remove in middle | O(n) (shift) | O(1) at node (O(n) to reach) |
| Memory | Compact (just data) | Higher (2 pointers per node) |
| Cache locality | **Excellent** | Poor (scattered nodes) |

> ⚠️ **In practice, `ArrayList` almost always wins** — even for many insertions/deletions — because its **cache locality** (contiguous memory) outweighs `LinkedList`'s theoretical advantages. `LinkedList` is rarely the right choice; prefer `ArrayDeque` for queue/stack needs (next note). This is a common interview "gotcha": the textbook Big-O favors LinkedList for middle inserts, but real hardware favors ArrayList.

---

## 3. Vector & Stack (legacy — avoid)

### 3.1 Vector
`Vector` is an old (Java 1.0) `ArrayList` equivalent whose methods are **`synchronized`** (thread-safe). It's **legacy**:
- Every method locks → slow even single-threaded.
- For thread safety, prefer `Collections.synchronizedList()` or `CopyOnWriteArrayList` (§5), or better, design to avoid shared mutable lists (Phase 1.10).

### 3.2 Stack
`Stack` extends `Vector` (LIFO). Also **legacy** — its inheritance from Vector exposes inappropriate methods and it's synchronized.
> **Use `ArrayDeque` as a stack instead** (next note): `push`/`pop`/`peek`, faster, not legacy. (`Deque<Integer> stack = new ArrayDeque<>();`)

| Legacy class | Use instead |
|--------------|-------------|
| `Vector` | `ArrayList` (+ synchronization if truly needed) |
| `Stack` | `ArrayDeque` |

---

## 4. CopyOnWriteArrayList (concurrent)

A **thread-safe** List for **read-heavy, write-rare** concurrent scenarios. On every **write** (add/set/remove), it **copies the entire underlying array**.

### 4.1 Characteristics
- **Reads are lock-free and fast** (no synchronization on read).
- **Writes are expensive** (full array copy each time).
- Iterators see a **snapshot** → never throw `ConcurrentModificationException` (next note's topic).

```java
List<Listener> listeners = new CopyOnWriteArrayList<>();  // many reads, rare writes
```
> Ideal for things like **event listener lists** or config read by many threads but updated rarely. Bad for write-heavy workloads. (Concurrency depth: Phase 1.10.)

---

## 5. Immutable Lists

Java provides ways to create **unmodifiable** lists — attempts to modify them throw `UnsupportedOperationException`.

### 5.1 `List.of()` (Java 9+) — truly immutable
```java
List<String> fixed = List.of("a", "b", "c");   // immutable
// fixed.add("d");   // UnsupportedOperationException
// List.of(...) also rejects null elements
```

### 5.2 `List.copyOf()` — immutable copy of an existing collection
```java
List<String> snapshot = List.copyOf(mutableList);   // independent immutable copy
```

### 5.3 `Collections.unmodifiableList()` — a read-only *view*
```java
List<String> view = Collections.unmodifiableList(backingList);
// view can't be modified directly, but changes to backingList ARE visible (it's a view!)
```

| Approach | Immutable? | Independent copy? |
|----------|-----------|-------------------|
| `List.of(...)` | ✅ Truly | ✅ (new list) |
| `List.copyOf(c)` | ✅ Truly | ✅ (copy) |
| `Collections.unmodifiableList(c)` | Read-only **view** | ❌ (reflects backing changes) |

> Immutable lists are **thread-safe** and great for constants, defensive returns, and DTO fields (recall encapsulation/defensive copying, Phase 1.3). Returning an immutable/empty list (never `null`) is a best practice.

---

## 6. Common List Operations

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.add("d");                  // append
list.add(1, "x");               // insert at index 1
list.set(0, "z");               // replace
list.remove("x");               // remove by value
list.remove(0);                 // remove by index (careful: int = index!)
list.contains("b");             // membership
list.indexOf("c");              // first index, or -1
list.subList(1, 3);             // view of a range [1,3)
list.removeIf(s -> s.isEmpty()); // conditional removal (safe during iteration!)
Collections.sort(list);          // sort (or list.sort(Comparator))
```

### 6.1 The `remove` overload trap
```java
List<Integer> nums = new ArrayList<>(List.of(10, 20, 30));
nums.remove(1);                  // removes by INDEX -> removes 20 (element at index 1)
nums.remove(Integer.valueOf(10)); // removes by VALUE -> removes the element 10
```
> For `List<Integer>`, `remove(int)` means **index**, `remove(Object)` means **value**. A classic bug — be explicit with `Integer.valueOf(...)` to remove by value.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using `LinkedList` for "many inserts" | `ArrayList` usually wins (cache locality) |
| Using `Vector`/`Stack` | Use `ArrayList`/`ArrayDeque` |
| `remove(int)` vs `remove(Object)` confusion | Use `Integer.valueOf(x)` to remove by value |
| Modifying a list during for-each | Use `Iterator.remove()` / `removeIf()` (next note) |
| Returning `null` instead of empty list | Return `List.of()` / `Collections.emptyList()` |
| Modifying a `List.of(...)` | It's immutable → use a mutable list |
| Not pre-sizing a large ArrayList | Pass initial capacity |
| Treating `unmodifiableList` as a copy | It's a view of the backing list |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **`ArrayList` / `List`** is the default for query results, DTO collections, request/response bodies (Phase 5).
- **Immutable lists / defensive copies** protect entity and DTO state (Phase 1.3, 5).
- **Return empty lists, never null** — a core API-design rule that prevents NPEs (Phase 7).
- **`CopyOnWriteArrayList`** for thread-safe listener/config lists (Phase 1.10).
- **Performance:** picking ArrayList over LinkedList and pre-sizing matter in hot paths (Phase 13).

---

## 9. Quick Self-Check Questions

1. What three traits define a `List`?
2. How is `ArrayList` implemented, and what's the complexity of `get(i)` and `add` at the end?
3. How is `LinkedList` implemented, and where is it O(1) vs O(n)?
4. Why does `ArrayList` usually beat `LinkedList` in practice despite Big-O?
5. Why are `Vector` and `Stack` discouraged, and what replaces them?
6. When is `CopyOnWriteArrayList` appropriate?
7. What's the difference between `List.of`, `List.copyOf`, and `Collections.unmodifiableList`?
8. Explain the `remove(int)` vs `remove(Object)` trap for `List<Integer>`.

---

## 10. Key Terms Glossary

- **`List`:** ordered, indexed collection allowing duplicates.
- **`ArrayList`:** dynamic-array List (default choice; O(1) random access).
- **`LinkedList`:** doubly-linked list; O(1) ends, O(n) random access; also a Deque.
- **Amortized O(1):** average constant time despite occasional resizes.
- **`Vector` / `Stack`:** legacy synchronized List/LIFO (avoid).
- **`CopyOnWriteArrayList`:** thread-safe, read-optimized List (copies on write).
- **Immutable list:** unmodifiable list (`List.of`, `List.copyOf`).
- **Unmodifiable view:** read-only window over a mutable backing list.
- **Cache locality:** contiguous memory making array access fast.
- **`removeIf`:** safe conditional removal.

---

*Previous topic: **Collections Hierarchy Overview**.*
*Next topic: **Set Implementations**.*
