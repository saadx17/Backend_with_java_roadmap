# Set Implementations

> **Phase 1 — Java Language Mastery → 1.6 Java Collections Framework**
> Goal: Master the `Set` implementations — `HashSet`, `LinkedHashSet`, `TreeSet`, `EnumSet`, and immutable sets — their internals, ordering, and when to use each.

---

## 0. The Big Picture

A **`Set<E>`** is a collection of **unique elements** — **no duplicates**. Adding an element that already exists has no effect. Sets are ideal for membership testing, deduplication, and uniqueness constraints.

```java
Set<String> tags = new HashSet<>();
tags.add("java");
tags.add("spring");
tags.add("java");        // ignored — already present
tags.size();             // 2
tags.contains("java");   // true (fast!)
```

| Set trait | Value |
|-----------|-------|
| Duplicates? | ❌ Never |
| Indexed access? | ❌ No (`get(i)` doesn't exist) |
| Ordering? | Depends on implementation |

> **Uniqueness is determined by `equals()` and `hashCode()`** (for hash-based sets) — recall Phase 1.3 Object methods. This is *the* most important thing to understand about sets.

---

## 1. HashSet (the default choice)

`HashSet` stores elements using a **hash table** — internally backed by a **`HashMap`** (elements are the keys). It offers the fastest operations but **no ordering**.

### 1.1 How it works
- Each element's **`hashCode()`** decides its bucket; **`equals()`** confirms uniqueness within the bucket (recall the HashMap mechanism — detailed in the Map note).
- **No defined iteration order** — elements come out in an unpredictable (hash-based) order.

### 1.2 Performance
| Operation | Complexity |
|-----------|-----------|
| `add` / `remove` / `contains` | **O(1)** average |

### 1.3 The equals/hashCode requirement (critical)
For a `HashSet` to detect duplicates correctly, elements **must implement `equals()` and `hashCode()` consistently** (Phase 1.3):
```java
Set<Point> points = new HashSet<>();
points.add(new Point(1, 2));
points.add(new Point(1, 2));   // treated as DUPLICATE only if Point overrides equals/hashCode!
// Without proper overrides -> 2 elements (uses identity) -> BUG
```
> ⚠️ If your custom class doesn't override `equals`/`hashCode`, a `HashSet` uses object identity → "equal" objects are treated as distinct. **Use records or proper overrides** for elements stored in sets.

> **Use `HashSet` by default** when you need uniqueness and don't care about order.

---

## 2. LinkedHashSet (insertion-ordered)

`LinkedHashSet` extends `HashSet` but also maintains a **doubly-linked list** through the elements, preserving **insertion order** during iteration.

```java
Set<String> set = new LinkedHashSet<>();
set.add("c"); set.add("a"); set.add("b");
// iteration order: c, a, b  (insertion order preserved)
```
- Same **O(1)** operations as `HashSet`, with a small memory/overhead cost for the linked list.
- **Use when** you need uniqueness **and** predictable (insertion) iteration order — e.g., deduplicating while keeping the original sequence.

---

## 3. TreeSet (sorted)

`TreeSet` keeps elements **sorted** — backed by a **Red-Black tree** (a self-balancing binary search tree). Iteration yields elements in **sorted order**.

### 3.1 How it works
- Elements are ordered by their **natural ordering** (`Comparable`) or a supplied **`Comparator`** (next note).
- Operations are **O(log n)** (tree height).

### 3.2 Performance & extras
| Operation | Complexity |
|-----------|-----------|
| `add` / `remove` / `contains` | **O(log n)** |

`TreeSet` implements `NavigableSet`, adding powerful ordered queries:
```java
TreeSet<Integer> set = new TreeSet<>(List.of(10, 20, 30, 40));
set.first();          // 10  (smallest)
set.last();           // 40  (largest)
set.floor(25);        // 20  (largest <= 25)
set.ceiling(25);      // 30  (smallest >= 25)
set.headSet(30);      // {10, 20}  (less than 30)
set.tailSet(30);      // {30, 40}  (>= 30)
set.descendingSet();  // {40, 30, 20, 10}
```

### 3.3 Requirements
- Elements must be **`Comparable`**, or you must provide a **`Comparator`** — otherwise `add` throws `ClassCastException`.
- **No nulls** (can't compare null).
> **Use `TreeSet`** when you need elements kept sorted or need range/navigation queries.

---

## 4. EnumSet (specialized & extremely efficient)

`EnumSet` is a high-performance `Set` implementation **specifically for enum types** (recall Phase 1.3 Enums note). Internally it's a **bit vector** (each enum constant = one bit).

```java
enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

EnumSet<Day> weekend  = EnumSet.of(Day.SAT, Day.SUN);
EnumSet<Day> weekdays = EnumSet.complementOf(weekend);
EnumSet<Day> all      = EnumSet.allOf(Day.class);
EnumSet<Day> none     = EnumSet.noneOf(Day.class);
EnumSet<Day> range    = EnumSet.range(Day.MON, Day.FRI);
```
- **Extremely fast and compact** (bitwise operations) — far better than `HashSet` for enums.
- Iterates in **enum declaration order**.
> **Always prefer `EnumSet` over `HashSet` for enum elements** (and over `int` bit flags) — *Effective Java* Item 36.

---

## 5. Immutable Sets

```java
Set<String> fixed = Set.of("a", "b", "c");      // immutable (Java 9+)
Set<String> copy  = Set.copyOf(existingSet);     // immutable copy
Set<String> view  = Collections.unmodifiableSet(backing);  // read-only view
```
- `Set.of(...)` is immutable, **rejects null**, and throws on **duplicate arguments** at creation.
- Immutable sets are **thread-safe** and great for constants / defensive returns (recall Phase 1.3).

---

## 6. Set Implementations Compared

| Implementation | Ordering | Backing structure | add/remove/contains | Nulls | Use when |
|----------------|----------|-------------------|---------------------|-------|----------|
| **HashSet** | None | Hash table | O(1) avg | 1 null | Default; fast uniqueness |
| **LinkedHashSet** | Insertion order | Hash table + linked list | O(1) avg | 1 null | Uniqueness + predictable order |
| **TreeSet** | **Sorted** | Red-Black tree | O(log n) | No nulls | Sorted / range queries |
| **EnumSet** | Enum order | Bit vector | O(1) | No nulls | Enum elements (always) |

---

## 7. Common Set Operations (set algebra)

```java
Set<Integer> a = new HashSet<>(List.of(1, 2, 3, 4));
Set<Integer> b = new HashSet<>(List.of(3, 4, 5, 6));

a.addAll(b);       // UNION       -> {1,2,3,4,5,6}
a.retainAll(b);    // INTERSECTION-> keep only common {3,4}
a.removeAll(b);    // DIFFERENCE  -> remove b's elements
a.containsAll(b);  // subset check
```
> Sets give you **set algebra** (union/intersection/difference) directly via bulk operations — handy for permissions, tags, deduplication.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Custom elements without `equals`/`hashCode` | Override them (or use records) — else duplicates aren't detected |
| Expecting `HashSet` to keep order | It has **no order**; use `LinkedHashSet`/`TreeSet` |
| Putting non-`Comparable` elements in `TreeSet` | Provide a `Comparator` or implement `Comparable` |
| Nulls in `TreeSet`/`EnumSet` | Not allowed |
| Using `HashSet` for enums | Use `EnumSet` (far faster) |
| Mutating elements used in a `HashSet` | If the element's hashCode changes, it's "lost" (use immutable elements) |
| Modifying `Set.of(...)` | It's immutable |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Deduplication & membership:** roles/permissions sets, unique IDs, tag sets (`Set<Role>`).
- **JPA `@ManyToMany`/`@OneToMany`** often map to `Set` — requires correct `equals`/`hashCode` on entities (a notorious JPA pitfall, Phase 5.4).
- **`EnumSet`** for feature flags / permission sets (Phase 1.3, 5).
- **`LinkedHashSet`** to dedupe while preserving order (e.g., ordered unique results).
- **Immutable sets** for constants and safe returns (Phase 1.3, 7).
- **Set algebra** for authorization checks (does user's roles intersect required roles?).

---

## 10. Quick Self-Check Questions

1. What defines a `Set`, and how is uniqueness determined for hash-based sets?
2. How is `HashSet` implemented, and what's its operation complexity?
3. Why must set elements implement `equals`/`hashCode` correctly?
4. How does `LinkedHashSet` differ from `HashSet`?
5. How is `TreeSet` implemented, what ordering does it give, and what extra queries?
6. Why prefer `EnumSet` for enum elements?
7. What's the difference between `Set.of`, `Set.copyOf`, and `unmodifiableSet`?
8. How do you compute union, intersection, and difference of two sets?

---

## 11. Key Terms Glossary

- **`Set`:** collection of unique elements (no duplicates).
- **`HashSet`:** hash-table set; O(1), unordered (default).
- **`LinkedHashSet`:** hash set preserving insertion order.
- **`TreeSet`:** sorted set (Red-Black tree); O(log n); `NavigableSet`.
- **`EnumSet`:** bit-vector set for enums (very fast).
- **Natural ordering / `Comparator`:** how `TreeSet` orders elements.
- **`NavigableSet`:** ordered queries (`floor`, `ceiling`, `headSet`, etc.).
- **Set algebra:** union (`addAll`), intersection (`retainAll`), difference (`removeAll`).
- **Immutable set:** unmodifiable set (`Set.of`, `Set.copyOf`).

---

*Previous topic: **List Implementations**.*
*Next topic: **Queue & Deque Implementations**.*
