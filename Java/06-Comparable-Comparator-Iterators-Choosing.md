# Comparable, Comparator, Iterators & Choosing the Right Collection

> **Phase 1 — Java Language Mastery → 1.6 Java Collections Framework**
> Goal: Master ordering (`Comparable` vs `Comparator`), iteration (`Iterator`, `ListIterator`, fail-fast vs fail-safe, `Spliterator`), the `Collections`/`Arrays` utilities, and a decision framework for choosing the right collection.

---

## 0. The Big Picture

This note ties the framework together: **how to order** elements (sorting), **how to traverse** collections safely, the **utility classes**, and finally **how to choose** the right collection for any situation.

---

## 1. Comparable vs Comparator (Ordering)

To sort objects, Java needs to know how to compare them. Two mechanisms:

| | **`Comparable<T>`** | **`Comparator<T>`** |
|---|---------------------|---------------------|
| Defines | The **natural** ordering | A **custom/external** ordering |
| Method | `int compareTo(T o)` | `int compare(T a, T b)` |
| Where | Implemented **inside** the class | A **separate** object/lambda |
| How many | One per class | As many as you want |
| Package | `java.lang` | `java.util` |

Both return: **negative** (a < b), **zero** (a == b), **positive** (a > b).

### 1.1 Comparable — natural ordering
The class implements `Comparable` to define its *default* sort order:
```java
class Person implements Comparable<Person> {
    String name; int age;
    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age);   // natural order = by age
    }
}
Collections.sort(people);   // uses compareTo — no comparator needed
```
> Use `Integer.compare`, `Double.compare`, etc. — **never** `a - b` for ints (can overflow!).

### 1.2 Comparator — custom orderings
Define ordering *outside* the class — flexible, multiple orderings:
```java
people.sort(Comparator.comparing(Person::getName));            // by name
people.sort(Comparator.comparingInt(Person::getAge));          // by age
people.sort(Comparator.comparing(Person::getName).reversed()); // descending
```

### 1.3 Comparator chaining & helpers (very useful)
```java
people.sort(
    Comparator.comparing(Person::getLastName)
              .thenComparing(Person::getFirstName)   // tie-breaker
              .thenComparingInt(Person::getAge)
);

Comparator.naturalOrder();         // natural order
Comparator.reverseOrder();         // reverse natural
Comparator.comparing(Person::getName, Comparator.nullsFirst(Comparator.naturalOrder())); // null-safe
```
| Method | Purpose |
|--------|---------|
| `comparing(keyFn)` | Compare by an extracted key |
| `comparingInt/Long/Double` | Primitive-specialized (avoids boxing) |
| `thenComparing(...)` | Secondary sort (tie-breaker) |
| `reversed()` | Reverse the order |
| `nullsFirst` / `nullsLast` | Null-safe ordering |

> `Comparator.comparing(...).thenComparing(...)` (built with lambdas/method references — Phase 1.8) is the modern, readable way to sort by multiple fields. `Comparator` is itself a **functional interface** (Phase 1.3).

---

## 2. Iterators

### 2.1 Iterator — the basic traversal
An **`Iterator`** traverses a collection one element at a time. Every `Iterable` provides one (recall the overview note). The for-each loop uses it under the hood.
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.isEmpty()) it.remove();   // SAFE removal during iteration!
}
```
| Method | Purpose |
|--------|---------|
| `hasNext()` | Is there another element? |
| `next()` | Return the next element (advances) |
| `remove()` | Remove the last-returned element (safe) |

> `Iterator.remove()` is the **safe** way to remove during iteration — modifying the collection directly in a for-each throws `ConcurrentModificationException` (§2.3). `removeIf(predicate)` is the modern shortcut.

### 2.2 ListIterator — bidirectional, for Lists
`ListIterator` (for `List`s only) adds backward traversal, index access, and in-place `set`/`add`:
```java
ListIterator<String> lit = list.listIterator();
while (lit.hasNext()) {
    String s = lit.next();
    lit.set(s.toUpperCase());   // replace in place
}
while (lit.hasPrevious()) {     // traverse backwards
    System.out.println(lit.previous());
}
```

### 2.3 Fail-fast vs Fail-safe iterators (important)
| | **Fail-fast** | **Fail-safe** |
|---|---------------|---------------|
| Behavior | Throws `ConcurrentModificationException` if the collection is modified during iteration | Iterates over a **snapshot/copy**; no exception |
| Detects changes via | A `modCount` counter | Working on a copy |
| Examples | `ArrayList`, `HashMap`, `HashSet` iterators | `CopyOnWriteArrayList`, `ConcurrentHashMap` iterators |
| Reflects concurrent changes? | (Would throw) | No (sees the snapshot) |

```java
// Fail-fast -> exception:
for (String s : arrayList) {
    arrayList.remove(s);      // ConcurrentModificationException!
}
// Safe options:
arrayList.removeIf(s -> condition);          // modern, safe
Iterator<String> it = arrayList.iterator();  // or explicit iterator.remove()
```
> **`ConcurrentModificationException`** (recall Phase 1.2 loops) means a collection was structurally modified during iteration. It's a **fail-fast safety mechanism**, not necessarily about threads — it happens single-threaded too. Fix with `Iterator.remove()`, `removeIf`, or a concurrent collection.

### 2.4 Spliterator (parallel iteration)
A **`Spliterator`** ("splittable iterator") can **partition** a collection for **parallel** processing — it's the engine behind **parallel streams** (Phase 1.8). You rarely use it directly, but it's why `collection.parallelStream()` can split work across cores (recall Phase 0.2 multi-core).

---

## 3. The Collections Utility Class

`java.util.Collections` provides static algorithms operating on collections:
```java
Collections.sort(list);                       // sort (natural order)
Collections.sort(list, comparator);           // sort with comparator
Collections.reverse(list);
Collections.shuffle(list);
Collections.max(list); Collections.min(list);
Collections.frequency(list, x);               // count occurrences
Collections.binarySearch(sortedList, key);    // O(log n) (must be sorted)
Collections.emptyList(); Collections.singletonList(x);
Collections.unmodifiableList(list);           // read-only view
Collections.synchronizedList(list);           // thread-safe wrapper
Collections.nCopies(3, "x");                  // [x, x, x]
```

### 3.1 Arrays utility (recap, Phase 1.2)
```java
Arrays.sort(array);
Arrays.asList(1, 2, 3);          // fixed-size List view of an array
Arrays.stream(array);            // to a stream (Phase 1.8)
Arrays.fill(array, 0);
```

---

## 4. Choosing the Right Collection (Decision Framework)

This is the practical payoff of the whole section.

### 4.1 Top-level decision
```
Storing key-value pairs?      -> Map
Storing single elements?
   Need uniqueness?           -> Set
   Need order/index/dupes?    -> List
   Processing in an order?    -> Queue/Deque
```

### 4.2 Within each, pick the implementation
```
LIST:
  Default / indexed access / iterate     -> ArrayList
  Need thread-safe, read-heavy           -> CopyOnWriteArrayList
  (rarely) need List + Deque / nulls     -> LinkedList

SET:
  Default / fast uniqueness              -> HashSet
  Need insertion order                   -> LinkedHashSet
  Need sorted / range queries            -> TreeSet
  Enum elements                          -> EnumSet

QUEUE/DEQUE:
  Default queue or stack                 -> ArrayDeque
  Need priority/min/max                  -> PriorityQueue
  Producer-consumer / thread-safe        -> BlockingQueue
  Lock-free concurrent FIFO              -> ConcurrentLinkedQueue

MAP:
  Default                                -> HashMap
  Insertion order / LRU                  -> LinkedHashMap
  Sorted keys / range queries            -> TreeMap
  Concurrent access                      -> ConcurrentHashMap
  Enum keys                              -> EnumMap
  GC-sensitive cache                     -> WeakHashMap
```

### 4.3 Quick decision questions
| Question | Leads to |
|----------|----------|
| Key-value? | Map family |
| Duplicates allowed? | List (yes) vs Set (no) |
| Need index access? | ArrayList |
| Need it sorted? | TreeSet / TreeMap |
| Need insertion order? | LinkedHashSet / LinkedHashMap |
| FIFO/LIFO/priority processing? | ArrayDeque / PriorityQueue |
| Shared across threads? | Concurrent* / CopyOnWrite* |
| Enum keys/elements? | EnumMap / EnumSet |

### 4.4 Default choices to memorize
> When unsure: **`ArrayList`** (list), **`HashMap`** (map), **`HashSet`** (set), **`ArrayDeque`** (queue/stack). For concurrency: **`ConcurrentHashMap`**. Swap to a more specialized type only when you need ordering, sorting, or concurrency.

---

## 5. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `a - b` in a comparator (overflow) | Use `Integer.compare(a, b)` |
| Modifying a collection in a for-each | Use `Iterator.remove()` / `removeIf` |
| Treating `ConcurrentModificationException` as thread-only | It happens single-threaded too |
| Sorting non-`Comparable` objects without a Comparator | Provide a `Comparator` |
| Forgetting `thenComparing` for tie-breakers | Chain comparators |
| Using `Collections.synchronizedX` and iterating without external sync | Synchronize the iteration block (or use concurrent collections) |
| Choosing collections randomly | Use the decision framework |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- **Sorting query results / DTOs** by multiple fields uses `Comparator.comparing().thenComparing()` (Phase 5, 7).
- **`removeIf`/iterators** avoid `ConcurrentModificationException` bugs in services (Phase 1.10).
- **Choosing the right collection** directly impacts endpoint performance (Phase 13).
- **`Spliterator`/parallel streams** for CPU-bound data processing across cores (Phase 1.8, 13, Project 2).
- **Concurrent collections** are mandatory for shared state in multi-threaded backends (Phase 1.10).
- **Sorting/`Comparator`** underlies pagination ordering, leaderboards, and ranking (Phase 4, 5).

---

## 7. Quick Self-Check Questions

1. What's the difference between `Comparable` and `Comparator`? When use each?
2. Why use `Integer.compare(a,b)` instead of `a - b`?
3. How do you sort by multiple fields with tie-breakers?
4. How do you safely remove elements during iteration?
5. What is `ConcurrentModificationException`, and is it only about threads?
6. What's the difference between fail-fast and fail-safe iterators? Give examples.
7. What does `ListIterator` add over `Iterator`?
8. What is a `Spliterator` for?
9. Walk through choosing a collection for: (a) a thread-safe cache, (b) sorted unique numbers, (c) a task queue, (d) FIFO with duplicates and index access.

---

## 8. Key Terms Glossary

- **`Comparable` / `compareTo`:** natural ordering defined in the class.
- **`Comparator` / `compare`:** external/custom ordering.
- **`comparing` / `thenComparing` / `reversed`:** comparator builders.
- **`Iterator`:** single-element traversal with safe `remove()`.
- **`ListIterator`:** bidirectional iterator for Lists (set/add).
- **Fail-fast:** iterator that throws on concurrent modification.
- **Fail-safe:** iterator over a snapshot (no exception).
- **`ConcurrentModificationException`:** thrown on structural modification during iteration.
- **`modCount`:** modification counter enabling fail-fast detection.
- **`Spliterator`:** splittable iterator for parallel processing.
- **`Collections` / `Arrays`:** utility classes of static algorithms.
- **Decision framework:** rules for choosing the right collection.

---

*Previous topic: **Map Implementations & HashMap Internals**.*
*This completes **Section 1.6 — Java Collections Framework**.*
*Next section in roadmap: **1.7 Generics**.*
