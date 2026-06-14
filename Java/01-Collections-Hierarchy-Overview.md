# Java Collections Framework — Hierarchy Overview

> **Phase 1 — Java Language Mastery → 1.6 Java Collections Framework**
> Goal: Understand the structure of the Collections Framework — the core interfaces, how they relate, and the mental map you'll use to choose the right collection.

---

## 0. The Big Picture

The **Java Collections Framework (JCF)** is a unified architecture for storing and manipulating groups of objects. Instead of rolling your own data structures, you use battle-tested, generic implementations of lists, sets, queues, and maps.

```
A Collection = a container of objects (elements).
The framework = interfaces (the contracts) + implementations (the concrete classes) + algorithms (utility methods).
```

> Mastering collections is foundational: you'll use `List`, `Map`, and `Set` in virtually every method you write. Choosing the right one affects correctness *and* performance.

---

## 1. Why a Framework?

| Without a framework | With the JCF |
|---------------------|--------------|
| Everyone writes their own list/map | Standard, reusable implementations |
| Inconsistent APIs | Unified interfaces (`add`, `remove`, `iterator`) |
| Bugs in hand-rolled structures | Battle-tested, optimized code |
| Hard to swap implementations | Program to interfaces → swap freely |

The JCF embodies **"program to an interface"** (recall Phase 1.3): you declare variables as `List`/`Map`/`Set` and can swap the concrete implementation (`ArrayList` → `LinkedList`) without changing other code.

```java
List<String> names = new ArrayList<>();   // interface type on the left
// later: change to new LinkedList<>() — the rest of the code is unaffected
```

---

## 2. The Two Hierarchies

The framework has **two separate root hierarchies**:
1. **`Collection`** — for groups of individual elements (List, Set, Queue).
2. **`Map`** — for key-value pairs (a *separate* hierarchy — a Map is **not** a Collection).

```
              Iterable
                 |
             Collection
        ________|________________
       |          |             |
      List       Set          Queue
       |          |             |
  ArrayList   HashSet      LinkedList
  LinkedList  LinkedHashSet ArrayDeque
  Vector      TreeSet      PriorityQueue
              EnumSet

   Map   (SEPARATE hierarchy — not a Collection!)
    |
  HashMap, LinkedHashMap, TreeMap, Hashtable,
  ConcurrentHashMap, EnumMap, WeakHashMap, ...
```

> ⚠️ **Common gotcha:** `Map` does **not** extend `Collection`. It's its own hierarchy because it stores *pairs*, not single elements. (But you can get `Collection`-like views: `map.keySet()`, `map.values()`, `map.entrySet()`.)

---

## 3. The Core Interfaces

### 3.1 `Iterable<T>` — the root
The top of the collection hierarchy. Anything `Iterable` can be used in a **for-each loop** (recall Phase 1.2 loops) and provides an `iterator()`.
```java
for (String s : anyIterable) { ... }   // works because of Iterable
```

### 3.2 `Collection<E>` — the base contract
Defines the common operations all collections share:
| Method | Purpose |
|--------|---------|
| `add(e)` / `remove(e)` | Add / remove an element |
| `contains(e)` | Membership test |
| `size()` / `isEmpty()` | Count / emptiness |
| `clear()` | Remove all |
| `iterator()` | Get an iterator |
| `stream()` | Get a stream (Phase 1.8) |

### 3.3 The three `Collection` sub-interfaces

| Interface | Models | Key traits | Duplicates? | Ordered? |
|-----------|--------|-----------|-------------|----------|
| **`List<E>`** | An ordered sequence | Index-based access | ✅ Yes | ✅ Insertion order |
| **`Set<E>`** | A collection of unique elements | No duplicates | ❌ No | Depends on impl |
| **`Queue<E>`** | Elements processed in an order | FIFO/priority/deque | ✅ Yes | By queue policy |

### 3.4 `Map<K,V>` — the separate hierarchy
Stores **key → value** pairs with **unique keys**:
| Method | Purpose |
|--------|---------|
| `put(k, v)` / `get(k)` | Store / retrieve by key |
| `remove(k)` | Remove by key |
| `containsKey(k)` / `containsValue(v)` | Lookups |
| `keySet()` / `values()` / `entrySet()` | Collection views |
| `getOrDefault` / `putIfAbsent` / `computeIfAbsent` / `merge` | Convenience methods |

---

## 4. Interfaces vs Implementations

A crucial distinction:
- **Interfaces** (`List`, `Set`, `Map`, `Queue`) define **what** you can do (the contract).
- **Implementations** (`ArrayList`, `HashSet`, `HashMap`, etc.) define **how** it's done (the data structure + performance).

```java
List<Integer> a = new ArrayList<>();    // dynamic array implementation
List<Integer> b = new LinkedList<>();   // doubly-linked list implementation
// Both honor the List contract; performance characteristics differ.
```

> **Best practice:** declare the variable as the **interface** type and only choose the concrete class at construction. This keeps code flexible (recall polymorphism, Phase 1.3).

---

## 5. Generics in Collections (recap/preview)

Collections are **generic** (`List<String>`, `Map<String, Integer>`) — they're parameterized by element type for **compile-time type safety** (full detail in Phase 1.7):
```java
List<String> names = new ArrayList<>();
names.add("Sam");
// names.add(42);          // COMPILE ERROR — type safety
String s = names.get(0);   // no cast needed
```
- Before generics (Java 5), collections held `Object` and required casting (error-prone).
- Collections store **objects only**, not primitives → primitives are **autoboxed** (`int` → `Integer`, recall Phase 1.2). For performance-critical primitive work, use arrays or primitive streams (Phase 1.8).

---

## 6. The `Collections` and `Arrays` Utility Classes (preview)

Two helper classes provide static algorithms (covered fully in the last note of this section):
- **`Collections`** — `sort`, `reverse`, `shuffle`, `max`, `min`, `unmodifiableList`, `synchronizedList`, etc.
- **`Arrays`** — `sort`, `asList`, `stream`, `copyOf` (recall Phase 1.2 arrays note).

```java
Collections.sort(list);
List<String> immutable = List.of("a", "b");   // immutable factory (Java 9+)
```

---

## 7. The Map of Implementations (your cheat sheet)

A high-level overview (each detailed in its own note):
| Interface | Implementations | One-line summary |
|-----------|-----------------|------------------|
| **List** | `ArrayList`, `LinkedList`, `Vector`, `CopyOnWriteArrayList` | Ordered, indexed, duplicates |
| **Set** | `HashSet`, `LinkedHashSet`, `TreeSet`, `EnumSet` | Unique elements |
| **Queue/Deque** | `ArrayDeque`, `PriorityQueue`, `LinkedList`, `*BlockingQueue` | Ordered processing |
| **Map** | `HashMap`, `LinkedHashMap`, `TreeMap`, `ConcurrentHashMap`, `EnumMap` | Key-value pairs |

---

## 8. Common Pitfalls / Misconceptions

| Misconception | Reality |
|---------------|---------|
| "Map is a Collection" | It's a **separate** hierarchy |
| Declaring `ArrayList<String> x = ...` everywhere | Declare as `List<String>` (program to interface) |
| "Collections can hold primitives" | They hold objects; primitives are autoboxed |
| Choosing a collection at random | Pick based on access pattern & performance (last note) |
| Confusing interfaces with implementations | Interface = contract; class = data structure |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Collections are everywhere:** method params/returns, DTOs, query results (`List<User>`), config (`Map<String,String>`), caches.
- **Spring Data** repositories return `List`/`Page`/`Set` of entities (Phase 5.4).
- **Choosing the right collection** affects API performance directly (Phase 13).
- **Concurrent collections** (`ConcurrentHashMap`) are essential in multi-threaded backends (Phase 1.10).
- **Streams** (Phase 1.8) operate on collections for filtering/transforming data — the modern data-processing style.

---

## 10. Quick Self-Check Questions

1. What are the two root hierarchies of the JCF, and why is Map separate?
2. What does `Iterable` enable?
3. What distinguishes List, Set, and Queue?
4. What's the difference between an interface and an implementation here? Why declare the interface type?
5. Why can't collections hold primitives directly, and what happens instead?
6. What are the three "view" methods of a Map?
7. Name one implementation each of List, Set, Queue, and Map.

---

## 11. Key Terms Glossary

- **Collections Framework (JCF):** unified architecture for groups of objects.
- **`Iterable`:** root interface enabling for-each iteration.
- **`Collection`:** base interface for List/Set/Queue.
- **`List` / `Set` / `Queue`:** ordered+indexed / unique / ordered-processing collections.
- **`Map`:** key-value store (separate hierarchy).
- **Interface vs implementation:** the contract vs the concrete data structure.
- **Generics:** type parameters for compile-time safety (`List<String>`).
- **Autoboxing:** automatic primitive↔wrapper conversion in collections.
- **View:** a `Collection`-style window into a Map (`keySet`, `values`, `entrySet`).
- **`Collections` / `Arrays`:** utility classes with static algorithms.

---

*This is the first note of **Section 1.6 — Java Collections Framework**.*
*Next topic: **List Implementations**.*
