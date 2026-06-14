# Collectors & Parallel Streams

> **Phase 1 — Java Language Mastery → 1.8 Functional Programming**
> Goal: Master `collect()` and the `Collectors` toolkit (grouping, partitioning, joining, downstream collectors, custom collectors) and understand parallel streams — when to use them and when not to.

---

## 0. The Big Picture

The terminal operation **`collect()`** accumulates stream elements into a result — a `List`, `Set`, `Map`, a `String`, a sum, or grouped/partitioned structures. The **`Collectors`** utility class provides ready-made recipes. This is where streams become powerful for **reporting and aggregation**.

```java
Map<Department, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));
```

---

## 1. Basic Collectors — to Collections

```java
import static java.util.stream.Collectors.*;

List<String> list = stream.collect(toList());              // a List (mutable, pre-Java 16)
List<String> ulist = stream.collect(toUnmodifiableList()); // immutable List
Set<String> set    = stream.collect(toSet());              // a Set (no order)
// Java 16+: stream.toList() is the concise immutable-list shortcut

// toCollection — choose the exact implementation:
TreeSet<String> sorted = stream.collect(toCollection(TreeSet::new));   // sorted set
LinkedList<String> ll  = stream.collect(toCollection(LinkedList::new));
```

---

## 2. Collecting to a Map — `toMap`

```java
Map<Long, User> byId = users.stream()
    .collect(toMap(User::getId, Function.identity()));   // key=id, value=the user

Map<String, Integer> nameToAge = users.stream()
    .collect(toMap(User::getName, User::getAge));
```
### 2.1 The duplicate-key trap
`toMap` **throws `IllegalStateException`** if two elements produce the **same key** — unless you supply a **merge function**:
```java
Map<String, Integer> counts = words.stream()
    .collect(toMap(w -> w, w -> 1, Integer::sum));   // merge: sum on key collision
// without the merge function, duplicate keys -> IllegalStateException
```
> ⚠️ A very common bug: `toMap` with non-unique keys crashes. Always provide a **merge function** when keys might collide.

---

## 3. Joining Strings — `joining`

```java
String csv = names.stream().collect(joining(", "));            // "Alice, Bob, Carol"
String wrapped = names.stream().collect(joining(", ", "[", "]")); // "[Alice, Bob, Carol]"
String plain = names.stream().collect(joining());              // "AliceBobCarol"
```
> `joining` is the idiomatic way to build delimited strings from streams (recall Phase 1.4 — better than manual `StringBuilder` loops).

---

## 4. Aggregation Collectors

```java
long count       = stream.collect(counting());                 // count
int totalAge     = users.stream().collect(summingInt(User::getAge));
double avgAge    = users.stream().collect(averagingInt(User::getAge));
Optional<User> oldest = users.stream().collect(maxBy(comparingInt(User::getAge)));
IntSummaryStatistics stats = users.stream().collect(summarizingInt(User::getAge));
// stats.getCount(), getSum(), getMin(), getMax(), getAverage()
```
> Note: for simple numeric aggregation, **primitive streams** (`mapToInt(...).sum()`, Phase 1.8.3) are often clearer. Collectors shine as **downstream** collectors inside grouping (§5).

---

## 5. Grouping — `groupingBy` (the powerhouse)

`groupingBy` partitions elements into a `Map` keyed by a classifier function — like SQL `GROUP BY`.

### 5.1 Basic grouping
```java
Map<Department, List<Employee>> byDept = employees.stream()
    .collect(groupingBy(Employee::getDepartment));
// key = department, value = list of employees in it
```

### 5.2 Grouping with a downstream collector
The 2-arg form applies another collector to each group — this is where it gets powerful:
```java
// Count employees per department:
Map<Department, Long> countByDept = employees.stream()
    .collect(groupingBy(Employee::getDepartment, counting()));

// Average salary per department:
Map<Department, Double> avgSalary = employees.stream()
    .collect(groupingBy(Employee::getDepartment, averagingDouble(Employee::getSalary)));

// Names per department (map values to names, collect to a list):
Map<Department, List<String>> namesByDept = employees.stream()
    .collect(groupingBy(Employee::getDepartment,
                        mapping(Employee::getName, toList())));

// Group into a specific Map type + downstream:
Map<Department, Set<Employee>> set = employees.stream()
    .collect(groupingBy(Employee::getDepartment, TreeMap::new, toSet()));
```
> **`groupingBy(classifier, downstream)`** is the single most useful collector for reports: "group by X, then do Y to each group" (count, average, sum, map, etc.).

---

## 6. Partitioning — `partitioningBy`

A special grouping into exactly **two** groups (true/false) by a predicate:
```java
Map<Boolean, List<Employee>> byActive = employees.stream()
    .collect(partitioningBy(Employee::isActive));
List<Employee> active   = byActive.get(true);
List<Employee> inactive = byActive.get(false);

// With downstream:
Map<Boolean, Long> countByActive = employees.stream()
    .collect(partitioningBy(Employee::isActive, counting()));
```
> `partitioningBy` always has **both** `true` and `false` keys (even if empty) — unlike `groupingBy` with a boolean, which omits empty groups.

---

## 7. Downstream Collectors (the building blocks)

Collectors used *inside* `groupingBy`/`partitioningBy` to process each group:
| Downstream collector | Effect |
|----------------------|--------|
| `counting()` | Count per group |
| `summingInt/Long/Double` | Sum per group |
| `averagingInt/Long/Double` | Average per group |
| `mapping(fn, downstream)` | Transform elements, then collect |
| `filtering(pred, downstream)` (Java 9+) | Filter within the group |
| `flatMapping(fn, downstream)` (Java 9+) | Flatten within the group |
| `reducing(...)` | Custom reduction per group |
| `collectingAndThen(coll, finisher)` | Post-process the result |
| `toList()` / `toSet()` | Collect group elements |

```java
// collectingAndThen — collect, then make immutable:
List<String> immutable = stream.collect(
    collectingAndThen(toList(), Collections::unmodifiableList));

// teeing (Java 12+) — combine TWO collectors:
record MinMax(int min, int max) {}
MinMax mm = numbers.stream().collect(
    teeing(minBy(naturalOrder()), maxBy(naturalOrder()),
           (min, max) -> new MinMax(min.get(), max.get())));
```

---

## 8. Custom Collectors (`Collector.of`)

For specialized accumulation, build a collector from its parts: a **supplier** (container), **accumulator** (add element), **combiner** (merge containers for parallel), and optional **finisher**:
```java
Collector<String, ?, String> customJoin = Collector.of(
    StringBuilder::new,                       // supplier: create container
    (sb, s) -> sb.append(s).append("|"),      // accumulator: add an element
    StringBuilder::append,                    // combiner: merge (for parallel)
    StringBuilder::toString                   // finisher: produce the result
);
```
> You rarely need custom collectors — the built-ins cover almost everything. But knowing the four parts (supplier/accumulator/combiner/finisher) demystifies how collectors work and why parallel streams need a **combiner**.

---

## 9. Parallel Streams

A **parallel stream** splits work across **multiple CPU cores** (recall Phase 0.2 multi-core, and `Spliterator` from Phase 1.6) using the common **ForkJoinPool**.
```java
long count = hugeList.parallelStream()
    .filter(this::isExpensive)
    .count();
// or: stream.parallel()  /  back to sequential: stream.sequential()
```

### 9.1 How it works
- The stream is **split** (via `Spliterator`) into chunks processed on different threads, then **combined**.
- Uses the shared **`ForkJoinPool.commonPool`** (size ≈ number of cores − 1, Phase 1.10).

### 9.2 When parallel streams HELP
| Condition | Why |
|-----------|-----|
| **Large** datasets | Enough work to outweigh splitting overhead |
| **CPU-bound** operations | Cores stay busy computing |
| **Stateless, independent** operations | No coordination needed |
| **Easily splittable source** | `ArrayList`, arrays, `IntStream.range` split well |

### 9.3 When parallel streams HURT (often!)
| Condition | Why it's bad |
|-----------|--------------|
| **Small** datasets | Splitting/merging overhead exceeds the gain |
| **I/O-bound** operations | Threads block; the common pool starves (use async instead — Phase 1.10) |
| **Shared mutable state** | Race conditions / wrong results / corruption |
| **Order-dependent** logic | Parallel doesn't guarantee processing order |
| **`LinkedList` / hard-to-split sources** | Poor splitting |
| **Stateful ops (`sorted`, `distinct`, `limit`)** | Costly to coordinate in parallel |

### 9.4 The cardinal rule
> ⚠️ **Parallel streams are NOT a free speedup.** They often make code *slower* or *incorrect*. Use them only for **large, CPU-bound, stateless** workloads on **splittable** sources — and **measure** (benchmark) before and after. **Never** mutate shared state in a parallel stream, and never use them for blocking I/O (they share the common pool with everything else).

```java
// DANGER: shared mutable state in a parallel stream -> race condition!
List<Integer> result = new ArrayList<>();
list.parallelStream().forEach(result::add);   // ArrayList is not thread-safe -> BUG
// Correct: use collect()
List<Integer> ok = list.parallelStream().collect(toList());   // thread-safe accumulation
```

---

## 10. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `toMap` with duplicate keys | Provide a merge function |
| Reaching for parallel streams to "go faster" | Only for large CPU-bound work; measure |
| Parallel stream with shared mutable state | Use `collect`; never mutate shared collections |
| Parallel streams for I/O | Use async/`CompletableFuture` (Phase 1.10) |
| Manual loops where a collector fits | Use `groupingBy`/`joining` etc. |
| Forgetting `groupingBy` downstream collectors | Use the 2-arg form (`counting`, `mapping`, ...) |
| Mutable `toList()` assumptions | `toList()` (Java 16) is unmodifiable |

---

## 11. Connection to Backend / Spring (Why This Matters Later)

- **Reports & analytics:** `groupingBy`/`summingInt`/`averagingDouble` build aggregations (sales by category, counts by status) — Phase 5, 7, Project 5 dashboards.
- **Entity → DTO mapping:** `map(...).toList()` / `toMap(...)` everywhere in service layers (Phase 7).
- **`joining`** for CSV/response building (Phase 1.4, Project 2).
- **`toMap(id, entity)`** to index query results by key for fast lookup.
- **Parallel streams** for CPU-bound batch processing (Project 2) — but prefer **`CompletableFuture`/executors** for I/O concurrency (Phase 1.10).
- Understanding the **combiner** clarifies why thread-safety matters in concurrent aggregation.

---

## 12. Quick Self-Check Questions

1. What does `collect()` do, and what does `Collectors` provide?
2. How do you collect a stream to a Map, and what's the duplicate-key trap?
3. How do you build a delimited string from a stream?
4. What does `groupingBy` do, and what is a downstream collector?
5. What's the difference between `groupingBy` (boolean) and `partitioningBy`?
6. Name three downstream collectors and what they do.
7. What are the four parts of a custom collector, and why is a combiner needed?
8. When do parallel streams help, and when do they hurt?
9. Why is mutating shared state in a parallel stream dangerous?

---

## 13. Key Terms Glossary

- **`collect()`:** terminal op accumulating elements into a result.
- **`Collectors`:** utility class of ready-made collectors.
- **`toList`/`toSet`/`toMap`/`toCollection`:** collect to collections.
- **Merge function:** resolves duplicate keys in `toMap`.
- **`joining`:** concatenate strings with a delimiter.
- **`groupingBy`:** partition into a Map by a classifier (like SQL GROUP BY).
- **`partitioningBy`:** split into true/false groups by a predicate.
- **Downstream collector:** a collector applied per group.
- **`mapping`/`filtering`/`collectingAndThen`/`teeing`:** composable collectors.
- **Custom collector:** supplier + accumulator + combiner + finisher.
- **Parallel stream:** splits work across cores via ForkJoinPool.
- **Combiner:** merges partial results in parallel processing.

---

*Previous topic: **Stream API — Creation & Operations**.*
*Next topic: **Optional**.*
