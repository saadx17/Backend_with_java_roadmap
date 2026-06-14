# String vs StringBuilder vs StringBuffer & Concatenation Performance

> **Phase 1 — Java Language Mastery → 1.4 Strings (Deep Dive)**
> Goal: Understand the three string-handling classes, when to use each, and why string concatenation in loops is a performance trap.

---

## 0. The Big Picture

Because `String` is **immutable**, building strings by repeated modification is wasteful — every change creates a new object. Java provides **mutable** alternatives for building strings efficiently:

| Class | Mutable? | Thread-safe? | Speed | Use when... |
|-------|----------|--------------|-------|-------------|
| **`String`** | ❌ Immutable | ✅ (immutable) | Fast for fixed text | Text that won't change |
| **`StringBuilder`** | ✅ Mutable | ❌ No | **Fastest** | Building strings in a single thread (the default choice) |
| **`StringBuffer`** | ✅ Mutable | ✅ Yes (synchronized) | Slower (locking) | Building strings shared across threads (rare) |

> **Rule of thumb:** use `String` for constant text, `StringBuilder` for building/modifying, and `StringBuffer` only in the rare case you truly need thread safety on the builder itself.

---

## 1. Why Not Just Use `String`?

Every "modifying" `String` operation allocates a **new** object (immutability, previous note):
```java
String s = "a";
s = s + "b";   // creates a new String "ab"  (the old "a" is now garbage)
s = s + "c";   // creates a new String "abc"
```
For a few operations this is fine. In a **loop**, it explodes (see §4).

---

## 2. StringBuilder (the workhorse)

A **mutable** sequence of characters. Modifications happen **in place** on an internal resizable `char[]`/`byte[]` — no new object per change.

```java
StringBuilder sb = new StringBuilder();
sb.append("Hello");
sb.append(", ").append("World");   // chainable (returns this)
sb.insert(0, ">> ");
sb.append('!');
String result = sb.toString();     // ">> Hello, World!"
```

### 2.1 Common methods
| Method | Effect |
|--------|--------|
| `append(x)` | Add to the end (overloaded for all types) |
| `insert(i, x)` | Insert at index |
| `delete(start, end)` / `deleteCharAt(i)` | Remove characters |
| `replace(start, end, str)` | Replace a range |
| `reverse()` | Reverse the contents |
| `setCharAt(i, c)` | Change one char |
| `charAt(i)` / `length()` | Read access |
| `toString()` | Produce the final `String` |
| `setLength(n)` / `capacity()` | Manage size |

### 2.2 Capacity (a performance detail)
- A `StringBuilder` has a backing array with a **capacity** (default 16 chars).
- When it fills, it **grows** (typically doubles) — which means copying.
- If you know the approximate final size, **pre-size it** to avoid regrowth:
  ```java
  StringBuilder sb = new StringBuilder(1024);  // pre-allocate capacity
  ```

### 2.3 Method chaining (fluent)
`append`, `insert`, etc. return `this`, enabling chaining (recall `this` uses from Classes & Objects note):
```java
String csv = new StringBuilder()
    .append("id").append(',')
    .append("name").append(',')
    .append("email")
    .toString();
```

---

## 3. StringBuffer (the synchronized version)

`StringBuffer` has the **same API** as `StringBuilder`, but its methods are **`synchronized`** (thread-safe). This makes it safe for multiple threads to modify the *same* buffer — at the cost of **locking overhead** on every call.

```java
StringBuffer sbf = new StringBuffer();
sbf.append("thread-safe");   // synchronized internally
```

### 3.1 StringBuilder vs StringBuffer
| Aspect | `StringBuilder` | `StringBuffer` |
|--------|-----------------|----------------|
| API | Identical | Identical |
| Thread safety | No | Yes (synchronized) |
| Performance | Faster | Slower (lock overhead) |
| Introduced | Java 5 | Java 1.0 |
| Use when | Single-threaded building (almost always) | Shared mutable buffer across threads (rare) |

> **In practice you almost always want `StringBuilder`.** Sharing a single mutable builder across threads is uncommon and usually a design smell — prefer building locally and combining results. `StringBuffer` is largely legacy. (Thread safety in depth: Phase 1.10.)

---

## 4. Concatenation Performance (the loop trap)

> **Never build strings with `+` inside a loop.**

### 4.1 The problem
```java
// BAD: O(n²) — each += creates a new String, copying all prior characters
String result = "";
for (int i = 0; i < 10_000; i++) {
    result += i + ",";       // allocates a new String every iteration
}
```
Each `+=` copies the entire growing string → for *n* iterations, total work is roughly **O(n²)** and generates *n* throwaway objects → GC pressure.

### 4.2 The fix
```java
// GOOD: O(n) — one mutable buffer, no per-iteration allocation
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10_000; i++) {
    sb.append(i).append(',');
}
String result = sb.toString();
```

### 4.3 What about simple `+` outside loops?
For a **fixed, small number** of concatenations, `+` is fine and readable:
```java
String msg = "User " + name + " logged in at " + time;   // perfectly fine
```
The compiler optimizes simple `+` expressions (historically into `StringBuilder`; since Java 9 via the `invokedynamic`-based **`makeConcatWithConstants`** / `StringConcatFactory`, which is often faster). **The trap is specifically `+` in a loop**, where the optimization can't help across iterations.

### 4.4 Other building options
| Approach | Best for |
|----------|----------|
| `StringBuilder` | Loops, conditional/incremental building |
| `String.join(delim, parts)` | Joining a known collection with a delimiter |
| `Collectors.joining(...)` (streams) | Joining stream results (Phase 1.8) |
| `String.format` / text blocks | Templated/structured text (next note) |

```java
String csv = String.join(",", "a", "b", "c");     // "a,b,c"
String csv2 = list.stream().collect(Collectors.joining(", "));
```

---

## 5. Quick Decision Guide

```
Need to build/modify a string?
│
├─ Fixed text, won't change                 → String
├─ A few concatenations (not in a loop)     → String + (compiler handles it)
├─ Building in a loop / incrementally       → StringBuilder
├─ Joining a collection with a delimiter    → String.join / Collectors.joining
└─ Shared mutable buffer across threads      → StringBuffer (rare)
```

---

## 6. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `+=` on String in a loop (O(n²)) | Use `StringBuilder` |
| Using `StringBuffer` by default | Use `StringBuilder` unless thread safety is truly needed |
| Not pre-sizing a large `StringBuilder` | Pass an initial capacity |
| Calling `toString()` repeatedly mid-build | Build fully, then `toString()` once |
| Joining with manual delimiter logic | Use `String.join` / `Collectors.joining` |
| Sharing one StringBuilder across threads | Build locally per thread, then combine |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Log message / response building** in hot paths: avoid `+` in loops; use `StringBuilder` or parameterized logging (`log.info("user {}", id)` — Phase 9).
- **CSV/report generation, query building** (Project 2, dynamic SQL) rely on efficient string building.
- **Performance:** string concatenation in loops is a classic cause of GC pressure and slow endpoints (Phase 13).
- **`Collectors.joining`** is the idiomatic way to build delimited output from streams (Phase 1.8).
- **Thread safety:** understanding why `StringBuilder` isn't thread-safe reinforces the broader concurrency model (Phase 1.10).

---

## 8. Quick Self-Check Questions

1. Why is repeated `String` concatenation in a loop inefficient (what's the complexity)?
2. What's the difference between `StringBuilder` and `StringBuffer`?
3. Which should you use by default, and why?
4. What is `StringBuilder` capacity, and why pre-size it?
5. Is simple `+` concatenation outside a loop a problem? Why or why not?
6. When would you use `String.join` or `Collectors.joining` instead of a builder?
7. Why is sharing a `StringBuilder` across threads usually a bad idea?

---

## 9. Key Terms Glossary

- **`String`:** immutable character sequence.
- **`StringBuilder`:** mutable, non-synchronized string builder (default choice).
- **`StringBuffer`:** mutable, synchronized (thread-safe) string builder.
- **`append`/`insert`/`delete`:** in-place mutation methods.
- **Capacity:** the backing array size of a builder (grows as needed).
- **Concatenation (`+`):** joining strings; compiler-optimized outside loops.
- **`StringConcatFactory` / `makeConcatWithConstants`:** Java 9+ runtime mechanism for `+`.
- **`String.join` / `Collectors.joining`:** delimiter-based joining utilities.

---

*Previous topic: **String Fundamentals**.*
*Next topic: **Common methods, formatting & text blocks**.*
