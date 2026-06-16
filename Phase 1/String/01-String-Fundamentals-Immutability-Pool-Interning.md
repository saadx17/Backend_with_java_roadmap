# String Fundamentals — Immutability, String Pool, Interning & Comparison

> **Phase 1 — Java Language Mastery → 1.4 Strings (Deep Dive)**
> Goal: Master what makes `String` special — immutability (why & how), the string pool/intern pool, interning, and the all-important `==` vs `.equals()` distinction.

---

## 0. The Big Picture

`String` is the most-used class in Java — and it behaves unlike ordinary objects in three deeply connected ways:
1. **Immutable** — a `String`'s contents can never change after creation.
2. **Pooled/interned** — string *literals* are cached and shared in a special memory region.
3. **Special syntax** — created with literals (`"hi"`) and concatenated with `+`.

Understanding these explains performance, memory behavior, security, and the classic `==` vs `.equals()` bug.

```java
String s = "hello";   // a literal — goes in the string pool
String t = new String("hello");  // explicitly a new heap object (NOT pooled)
```

---

## 1. String Immutability

### 1.1 What it means
Once a `String` object is created, its character data **cannot be modified**. Every operation that "changes" a string actually **returns a brand-new String**, leaving the original untouched.

```java
String s = "hello";
s.toUpperCase();              // returns "HELLO" but DOESN'T change s
System.out.println(s);       // "hello"  -> unchanged!

s = s.toUpperCase();         // to "keep" the result, reassign
System.out.println(s);       // "HELLO"
```
Methods like `concat`, `replace`, `substring`, `trim`, `toUpperCase` all **return new strings**.

### 1.2 How immutability is implemented
- Internally, `String` stores its data in a **`private final byte[]`** (a `char[]` before Java 9 — see the encoding note for Compact Strings).
- The array is `final` and never exposed or mutated; no method changes it.
- `String` is a **`final` class** → can't be subclassed to add mutating behavior.

### 1.3 Why strings are immutable (the benefits)
| Benefit | Explanation |
|---------|-------------|
| **Safe sharing / pooling** | Immutability is what makes the string pool possible (many references can share one object safely) |
| **Thread safety** | Immutable objects are inherently thread-safe — no synchronization needed (Phase 1.10) |
| **Hashcode caching** | `String` caches its `hashCode()` (computed once) → fast `HashMap` keys |
| **Security** | Strings used for file paths, URLs, DB connections, class names can't be changed after validation (defends against TOCTOU attacks) |
| **Reliable as map keys** | A key's hashcode never changes (recall the hashCode mutability warning from the Object methods note) |

> **Trade-off:** immutability means string-building via repeated concatenation creates lots of throwaway objects → use `StringBuilder` (next note).

---

## 2. The String Pool (Intern Pool)

The **string pool** (a.k.a. **string constant pool** / **intern pool**) is a special area where the JVM stores **unique string literals**. When you use a literal, the JVM reuses the pooled instance instead of creating duplicates.

```java
String a = "hello";
String b = "hello";
System.out.println(a == b);   // true!  -> both reference the SAME pooled object
```
Because `"hello"` is a literal, both `a` and `b` point to the **one** pooled `"hello"`. This saves memory (one copy of each distinct literal).

### 2.1 Where the pool lives
- Since **Java 7**, the string pool lives in the **heap** (moved from PermGen). This allows it to be garbage-collected and to grow.
- (Before Java 7 it lived in PermGen, which had a fixed size — a source of OOM errors.)

### 2.2 Literal vs `new String(...)`
```java
String s1 = "hello";              // pooled
String s2 = "hello";              // same pooled object  -> s1 == s2 is true
String s3 = new String("hello");  // NEW heap object, NOT pooled -> s1 == s3 is FALSE
```
> **`new String("hello")` always creates a fresh object on the heap**, bypassing the pool. It's almost always unnecessary and wasteful — prefer literals. (`s1.equals(s3)` is still `true` — same content.)

### 2.3 Compile-time constant folding
The compiler concatenates **constant** literals at compile time, and the result is pooled:
```java
String a = "hel" + "lo";   // folded to "hello" at compile time -> pooled
String b = "hello";
System.out.println(a == b); // true

String pre = "hel";
String c = pre + "lo";      // pre is a variable -> computed at RUNTIME -> NOT pooled
System.out.println(a == c); // false  (unless interned — see §3)
```

---

## 3. String Interning

**Interning** = putting a string into the pool (or getting the pooled reference if it already exists), via `String.intern()`.

```java
String c = new String("hello");   // heap object, not pooled
String d = c.intern();            // returns the POOLED "hello"
String e = "hello";
System.out.println(d == e);       // true  -> both are the pooled instance
System.out.println(c == e);       // false -> c is still the separate heap object
```

### 3.1 What `intern()` does
- If the pool already contains a string equal (by `.equals()`) to this one → returns the **pooled** reference.
- Otherwise → adds this string to the pool and returns it.

### 3.2 When (and whether) to use interning
- **Rarely needed manually.** Literals are auto-interned.
- Can save memory if you have **massive numbers of duplicate runtime-built strings** (e.g., parsing data with many repeated values).
- ⚠️ Overusing `intern()` can **hurt** — it adds pool-management overhead and the pool competes for heap. Modern advice: usually let the JVM handle it; consider a dedicated cache (`Map`) instead of `intern()` for deduplication.

---

## 4. String Comparison: `==` vs `.equals()` (Critical)

> The single most important practical takeaway about strings.

| Operator | Compares | Use for |
|----------|----------|---------|
| **`==`** | **References** (same object in memory?) | Almost never for strings |
| **`.equals()`** | **Content** (same characters?) | **Always** for string content |

```java
String a = "hello";
String b = "hello";
String c = new String("hello");

a == b;          // true  (both pooled — coincidence of pooling!)
a == c;          // false (c is a separate heap object)
a.equals(c);     // true  (same content) — THIS is what you want
```

### 4.1 Why `==` is a bug waiting to happen
`a == b` being `true` is an **accident of pooling**, not a reliable rule. As soon as a string comes from user input, a database, a file, or runtime concatenation, `==` will return `false` even for identical content:
```java
String input = readUserInput();   // "yes"
if (input == "yes") { ... }       // BUG: almost always false!
if (input.equals("yes")) { ... }  // correct
```

### 4.2 Best practices
- **Always use `.equals()`** (or `.equalsIgnoreCase()`) to compare string content.
- **Null-safe trick:** put the literal first, or use `Objects.equals`:
  ```java
  "yes".equals(input);            // null-safe: never NPEs even if input is null
  Objects.equals(a, b);           // null-safe both ways
  input.equalsIgnoreCase("YES");  // case-insensitive
  ```
- For **ordering**, use `compareTo()` (lexicographic) — returns negative/zero/positive.

```java
"apple".compareTo("banana");   // negative ("apple" < "banana")
"a".compareToIgnoreCase("A");  // 0
```

---

## 5. Memory Picture (tying it together)

```java
String a = "hello";               // pooled
String b = "hello";               // same pooled object
String c = new String("hello");   // separate heap object
```
```
   STACK            STRING POOL (heap)        HEAP
  a ---------+----> [ "hello" ] <----+
  b ---------+                       |
                                     |
  c --------------> [ "hello" ] (separate object, content equal)
```
- `a` and `b` → the one pooled `"hello"` (`a == b` true).
- `c` → its own object (`a == c` false, `a.equals(c)` true).

---

## 6. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Comparing strings with `==` | Use `.equals()` / `.equalsIgnoreCase()` |
| `new String("literal")` unnecessarily | Use the literal directly |
| Expecting a method to mutate the string | Strings are immutable — reassign the result |
| `input.equals("x")` NPE when input is null | Use `"x".equals(input)` or `Objects.equals` |
| Overusing `intern()` | Usually unnecessary; consider a `Map` cache |
| Building strings with `+` in a loop | Use `StringBuilder` (next note) |
| Putting secrets in `String` (lingers in pool/memory) | Use `char[]` for passwords; clear after use |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **`==` vs `.equals()`** is a constant correctness issue with request params, headers, DB values, enum-like codes.
- **Immutability + thread safety:** strings can be shared freely across request-handling threads without locks (Phase 1.10).
- **Hashcode caching** makes `String` excellent as `HashMap`/cache keys (config, Redis keys — Phase 4/5).
- **Security:** validated paths/URLs/SQL can't be tampered with after checks; **passwords should use `char[]`**, not `String`, to avoid lingering in memory/pool (Phase 15).
- **Memory:** huge numbers of duplicate strings (e.g., parsing) can bloat the heap — deduplication or `-XX:+UseStringDeduplication` (G1 GC) helps (Phase 1.11).

---

## 8. Quick Self-Check Questions

1. What does it mean that `String` is immutable, and how is it enforced?
2. Give three benefits of string immutability.
3. What is the string pool, and where does it live since Java 7?
4. Why does `"hello" == "hello"` return `true` but `"hello" == new String("hello")` return `false`?
5. What does `String.intern()` do, and when might you use it?
6. Why should you always compare strings with `.equals()` instead of `==`?
7. How do you compare strings null-safely and case-insensitively?
8. Why should passwords use `char[]` instead of `String`?

---

## 9. Key Terms Glossary

- **Immutable:** state can't change after creation (returns new objects instead).
- **String pool / intern pool:** heap region caching unique string literals.
- **String literal:** a string written directly in source (`"hi"`) — auto-pooled.
- **Interning:** placing/looking up a string in the pool (`intern()`).
- **Constant folding:** compile-time concatenation of literal constants.
- **`==` vs `.equals()`:** reference identity vs content equality.
- **`compareTo()`:** lexicographic ordering comparison.
- **String deduplication:** GC feature collapsing duplicate string contents.
- **Compact Strings:** Java 9+ byte[]-backed storage (see encoding note).

---

*This is the first note of **Section 1.4 — Strings (Deep Dive)**.*
*Next topic: **String vs StringBuilder vs StringBuffer & concatenation performance**.*
