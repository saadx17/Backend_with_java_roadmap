# Stream API — Creation & Operations

> **Phase 1 — Java Language Mastery → 1.8 Functional Programming**
> Goal: Master the Stream API — creating streams, intermediate operations (lazy), terminal operations, primitive streams, and how lazy evaluation/short-circuiting work.

---

## 0. The Big Picture

A **Stream** is a sequence of elements supporting **declarative, pipelined** operations. Instead of writing loops, you describe a **pipeline** of transformations (`filter` → `map` → `collect`).

```java
List<String> result = users.stream()           // source
    .filter(u -> u.isActive())                 // intermediate (lazy)
    .map(User::getName)                         // intermediate (lazy)
    .sorted()                                   // intermediate (lazy)
    .toList();                                  // terminal (triggers execution)
```

### 0.1 Key facts about streams
| Fact | Meaning |
|------|---------|
| **Not a data structure** | A stream doesn't store data — it processes data from a source |
| **Doesn't modify the source** | Streams are non-mutating; they produce new results |
| **Lazy** | Intermediate ops do nothing until a terminal op runs |
| **Single-use** | A stream can be consumed **once**; reusing throws `IllegalStateException` |
| **Pipelined** | Operations chain into a processing pipeline |

> Streams = **what** you want, not **how** to loop. They shine for filtering, transforming, and aggregating collections (recall Phase 1.6).

---

## 1. The Anatomy of a Stream Pipeline

```
SOURCE  ->  INTERMEDIATE ops (0+)  ->  TERMINAL op (exactly 1)
collection    filter, map, sorted...     collect, forEach, count...
            (lazy — return a Stream)    (eager — produce a result/side-effect)
```
- A pipeline has **one source**, **zero or more intermediate** ops, and **exactly one terminal** op.
- **Nothing runs until the terminal op is invoked** (laziness, §5).

---

## 2. Creating Streams

| Source | How |
|--------|-----|
| From a collection | `collection.stream()` / `parallelStream()` |
| From values | `Stream.of("a", "b", "c")` |
| From an array | `Arrays.stream(array)` |
| Empty | `Stream.empty()` |
| Infinite (iterate) | `Stream.iterate(0, n -> n + 2)` |
| Infinite (generate) | `Stream.generate(Math::random)` |
| Number ranges | `IntStream.range(0, 10)` / `rangeClosed(1, 10)` |
| File lines | `Files.lines(path)` |
| Reader lines | `bufferedReader.lines()` |

```java
List.of(1, 2, 3).stream();
Stream.of("a", "b", "c");
Arrays.stream(new int[]{1, 2, 3});             // IntStream
Stream.iterate(1, n -> n * 2).limit(5);        // 1,2,4,8,16 (must limit infinite streams!)
Stream.generate(() -> "x").limit(3);           // x,x,x
IntStream.rangeClosed(1, 5);                    // 1,2,3,4,5
```
> ⚠️ `Stream.iterate`/`generate` create **infinite** streams — you must `limit()` them (or use a short-circuiting op) or they'll run forever.

---

## 3. Intermediate Operations (Lazy — Return a Stream)

Intermediate ops **transform** a stream and return a **new stream**. They are **lazy** — they don't execute until a terminal op runs.

| Operation | Purpose |
|-----------|---------|
| `filter(predicate)` | Keep elements matching a condition |
| `map(function)` | Transform each element |
| `flatMap(function)` | Flatten nested streams into one |
| `distinct()` | Remove duplicates (uses `equals`) |
| `sorted()` / `sorted(comparator)` | Sort elements |
| `limit(n)` | Keep only the first n |
| `skip(n)` | Skip the first n |
| `peek(consumer)` | Inspect elements (debugging only) |
| `takeWhile(pred)` / `dropWhile(pred)` | Take/drop while condition holds (Java 9+) |
| `mapToInt/Long/Double` | Convert to a primitive stream |

### 3.1 filter & map (the workhorses)
```java
users.stream()
    .filter(u -> u.getAge() >= 18)    // keep adults
    .map(User::getName)                // User -> String (name)
    .forEach(System.out::println);
```

### 3.2 flatMap (flatten nested structures)
`flatMap` turns each element into a stream and **concatenates** them into one flat stream:
```java
List<List<Integer>> nested = List.of(List.of(1, 2), List.of(3, 4), List.of(5));
List<Integer> flat = nested.stream()
    .flatMap(List::stream)             // Stream<List<Integer>> -> Stream<Integer>
    .toList();                          // [1, 2, 3, 4, 5]

// Common: get all order items across all orders
orders.stream()
    .flatMap(order -> order.getItems().stream())
    .map(Item::getName)
    .toList();
```
> `map` = one-to-one transform; **`flatMap` = one-to-many then flatten**. Use flatMap when each element yields a *collection/stream* you want merged.

### 3.3 distinct, sorted, limit, skip
```java
Stream.of(3, 1, 2, 3, 1)
    .distinct()                        // 3, 1, 2
    .sorted()                          // 1, 2, 3
    .skip(1)                           // 2, 3
    .limit(1)                          // 2
    .toList();
```

### 3.4 peek (debugging only — not for side effects)
```java
list.stream()
    .peek(x -> System.out.println("before filter: " + x))   // inspect
    .filter(x -> x > 2)
    .peek(x -> System.out.println("after filter: " + x))
    .toList();
```
> ⚠️ `peek` is for **debugging** the pipeline, not for performing real side effects (its execution isn't guaranteed in all cases). Don't rely on it for logic.

### 3.5 takeWhile / dropWhile (Java 9+)
```java
Stream.of(1, 2, 3, 4, 1, 2)
    .takeWhile(n -> n < 4)             // 1, 2, 3 (stops at first failure)
    .toList();
Stream.of(1, 2, 3, 4, 1, 2)
    .dropWhile(n -> n < 4)             // 4, 1, 2 (drops the prefix)
    .toList();
```

---

## 4. Terminal Operations (Eager — Produce a Result)

Terminal ops **trigger execution** and produce a result or side effect. After a terminal op, the stream is **consumed** (can't be reused).

| Operation | Returns | Purpose |
|-----------|---------|---------|
| `forEach(consumer)` | void | Side effect per element |
| `forEachOrdered` | void | Like forEach but respects order (parallel) |
| `collect(collector)` | varies | Accumulate into a collection/result (next note) |
| `toList()` (Java 16+) | `List` | Collect to an unmodifiable list |
| `toArray()` | array | Collect to an array |
| `reduce(...)` | value/Optional | Combine elements into one value |
| `count()` | long | Number of elements |
| `min` / `max(comparator)` | `Optional` | Smallest / largest |
| `findFirst` / `findAny` | `Optional` | A matching element |
| `anyMatch` / `allMatch` / `noneMatch` | boolean | Predicate checks |

### 4.1 Collecting results
```java
List<String> names = users.stream().map(User::getName).toList();       // Java 16+
List<String> names2 = users.stream().map(User::getName)
                          .collect(Collectors.toList());                // older / mutable
String[] arr = stream.toArray(String[]::new);                          // method ref (Phase 1.8.2)
```

### 4.2 reduce (fold into one value)
`reduce` combines elements using an associative function:
```java
int sum = Stream.of(1, 2, 3, 4).reduce(0, Integer::sum);        // 10 (identity=0)
Optional<Integer> product = Stream.of(1, 2, 3).reduce((a, b) -> a * b);  // Optional[6]
String joined = Stream.of("a", "b", "c").reduce("", String::concat);    // "abc"
```
| Form | Returns |
|------|---------|
| `reduce(identity, accumulator)` | the value (identity if empty) |
| `reduce(accumulator)` | `Optional` (empty if no elements) |

### 4.3 Matching & finding (short-circuiting — see §5)
```java
boolean anyAdult  = users.stream().anyMatch(u -> u.getAge() >= 18);
boolean allActive = users.stream().allMatch(User::isActive);
boolean noneAdmin = users.stream().noneMatch(User::isAdmin);
Optional<User> first = users.stream().filter(User::isActive).findFirst();
```

### 4.4 count, min, max
```java
long n = users.stream().filter(User::isActive).count();
Optional<User> oldest = users.stream().max(Comparator.comparingInt(User::getAge));
```

---

## 5. Lazy Evaluation & Short-Circuiting (How Streams Really Work)

### 5.1 Laziness
Intermediate operations are **lazy** — they build up the pipeline but **don't run** until a terminal op is called. This enables efficiency: operations are **fused** and applied element-by-element.
```java
Stream.of(1, 2, 3, 4)
    .filter(n -> { System.out.println("filter " + n); return n % 2 == 0; })
    .map(n -> { System.out.println("map " + n); return n * 10; })
    .findFirst();
// Output shows filter/map interleaved, and STOPS early after finding the first match:
// filter 1, filter 2, map 2  -> findFirst returns 20. (3 and 4 never processed!)
```
> Each element flows through the **whole pipeline** before the next — not "filter all, then map all." This is more efficient and enables short-circuiting.

### 5.2 Short-circuiting operations
Some operations can **stop early** without processing every element:
| Short-circuit op | Stops when |
|------------------|------------|
| `findFirst` / `findAny` | A match is found |
| `anyMatch` | First true |
| `allMatch` / `noneMatch` | First counterexample |
| `limit(n)` | n elements collected |

This is why `limit()` makes **infinite streams** usable:
```java
Stream.iterate(1, n -> n + 1)     // infinite
    .filter(n -> n % 7 == 0)
    .limit(3)                      // short-circuits -> 7, 14, 21
    .toList();
```

### 5.3 Stateless vs stateful intermediate ops (awareness)
- **Stateless:** `filter`, `map` — process each element independently.
- **Stateful:** `sorted`, `distinct`, `limit` — may need to see other/all elements (e.g., `sorted` must buffer everything). Stateful ops can hurt performance on large/parallel streams.

---

## 6. Primitive Streams (IntStream, LongStream, DoubleStream)

To avoid boxing (recall Phase 1.2/1.7), use **primitive streams** for numeric work — they also add numeric convenience methods.
```java
IntStream.rangeClosed(1, 5).sum();                 // 15
IntStream.of(3, 1, 2).max();                       // OptionalInt[3]
IntStream.rangeClosed(1, 10).average();            // OptionalDouble
users.stream().mapToInt(User::getAge).average();   // object stream -> IntStream

// Conversions:
IntStream.range(0, 3).boxed().toList();            // IntStream -> Stream<Integer> (List)
stream.mapToInt(String::length).sum();             // Stream<String> -> IntStream -> sum
```
| Primitive stream | For |
|------------------|-----|
| `IntStream` | `int` |
| `LongStream` | `long` |
| `DoubleStream` | `double` |

Extras: `sum()`, `average()`, `min()`, `max()`, `summaryStatistics()` (count/sum/min/max/avg in one pass), `boxed()`.
```java
IntSummaryStatistics stats = users.stream().mapToInt(User::getAge).summaryStatistics();
stats.getAverage(); stats.getMax(); stats.getCount();
```

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Reusing a consumed stream | Streams are single-use — create a new one |
| Infinite stream without `limit` | Always bound `iterate`/`generate` |
| Using `peek` for real logic | It's debug-only; use `map`/`forEach` |
| Side effects / shared mutable state in lambdas | Keep stream ops pure |
| Boxing in numeric pipelines | Use `IntStream`/`mapToInt` |
| `forEach` to build a list (mutating an external list) | Use `collect`/`toList` |
| Expecting `filter`-all-then-`map`-all | It's element-by-element (lazy fusion) |
| Overusing streams for trivial loops | A simple `for` is sometimes clearer/faster |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Data transformation everywhere:** mapping entities → DTOs, filtering query results, aggregating reports (Phase 5, 7).
- **`Collectors.groupingBy`/`joining`** (next note) for reports & responses.
- **Spring Data** can return `Stream<T>` for large result sets (process lazily without loading all into memory — Phase 5.4).
- **`Optional`** is returned by `findFirst`/`min`/`max` and pairs with streams (next note).
- **Project 2** (file processor) uses streams + primitive streams for aggregation; parallel streams for speed (next note).
- **Performance:** laziness/short-circuiting avoid unnecessary work; primitive streams avoid boxing (Phase 13).

---

## 9. Quick Self-Check Questions

1. What is a stream, and how does it differ from a collection?
2. What are the three parts of a stream pipeline?
3. Name five ways to create a stream.
4. What's the difference between intermediate and terminal operations?
5. What does `flatMap` do that `map` doesn't?
6. Why is `peek` only for debugging?
7. Explain laziness and short-circuiting with an example.
8. How does `limit` make an infinite stream usable?
9. Why and when use primitive streams (`IntStream`)?
10. What happens if you reuse a stream after a terminal operation?

---

## 10. Key Terms Glossary

- **Stream:** a pipeline for processing a sequence of elements.
- **Source / intermediate / terminal:** where a stream starts / lazy transforms / the result trigger.
- **Lazy evaluation:** intermediate ops run only when a terminal op executes.
- **Short-circuiting:** stopping early (e.g., `findFirst`, `limit`, `anyMatch`).
- **`filter` / `map` / `flatMap`:** keep / transform / flatten.
- **`reduce`:** fold elements into a single value.
- **`collect` / `toList`:** accumulate into a result.
- **Stateless vs stateful op:** independent per element vs needing other elements.
- **Primitive stream (`IntStream`...):** boxing-free numeric streams.
- **`summaryStatistics`:** count/sum/min/max/avg in one pass.
- **Single-use:** a stream can be consumed only once.

---

*Previous topic: **Method References**.*
*Next topic: **Collectors & Parallel Streams**.*
