# Exception Hierarchy — Checked vs Unchecked

> **Phase 1 — Java Language Mastery → 1.5 Exception Handling**
> Goal: Master the Java exception class hierarchy and the crucial distinction between checked and unchecked exceptions — when each occurs and when to use them.

---

## 0. The Big Picture

An **exception** is an event that disrupts the normal flow of a program — an error condition represented as an **object**. Java's exception handling lets you **detect**, **propagate**, and **respond** to these conditions in a structured way (instead of crashing or returning error codes).

```java
try {
    int x = 10 / 0;          // throws ArithmeticException
} catch (ArithmeticException e) {
    System.out.println("Cannot divide by zero");
}
```

Everything throwable in Java is an object descending from **`Throwable`**.

---

## 1. The Throwable Hierarchy

```
                    Throwable
                   /         \
              Error           Exception
           (don't catch)     /         \
                   (checked exceptions)  RuntimeException
                   IOException, etc.    (unchecked exceptions)
                                         NullPointerException, etc.
```

| Class | Meaning | Catch it? |
|-------|---------|-----------|
| **`Throwable`** | Root of all errors/exceptions | ❌ Too broad |
| **`Error`** | Serious JVM-level problems (unrecoverable) | ❌ Never |
| **`Exception`** | Application-level problems (recoverable) | ✅ The checked branch |
| **`RuntimeException`** | A subclass of `Exception` — programming bugs | ✅ But usually fix the bug |

### 1.1 `Error` — don't touch
`Error` represents **serious, unrecoverable** problems the JVM throws — you generally **cannot and should not** handle them:
| Error | Cause |
|-------|-------|
| `OutOfMemoryError` | Heap/metaspace exhausted (Phase 1.11) |
| `StackOverflowError` | Too-deep recursion (Phase 0 stack note) |
| `NoClassDefFoundError` | A required class missing at runtime |
> Catching `Error` is almost always wrong — the program is in an unrecoverable state.

### 1.2 `Exception` — the part you handle
`Exception` and its subclasses represent conditions an application **can reasonably anticipate and recover from**. This branch splits into **checked** and **unchecked**.

---

## 2. The Key Split: Checked vs Unchecked

> **The single most important concept in this topic.**

| | **Checked** | **Unchecked (Runtime)** |
|---|-------------|-------------------------|
| Superclass | `Exception` (not RuntimeException) | `RuntimeException` |
| Compiler-enforced? | ✅ **Yes** — must handle or declare | ❌ No |
| Represents | Recoverable external conditions | Programming bugs |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |
| Typical response | Catch and recover, or propagate | Fix the code |

The deciding rule: **is it a subclass of `RuntimeException`?**
- Yes → **unchecked**.
- No (but is an `Exception`) → **checked**.

---

## 3. Checked Exceptions (Compile-Time)

A **checked exception** must be either **caught** (`try-catch`) or **declared** (`throws`) — the **compiler enforces** this. They represent **expected, recoverable** problems involving the outside world (files, network, DB).

```java
// Won't compile unless you handle or declare IOException:
public void readFile(String path) throws IOException {   // declare
    Files.readString(Path.of(path));                     // may throw IOException
}

// Or catch it:
public void readFileSafe(String path) {
    try {
        Files.readString(Path.of(path));
    } catch (IOException e) {
        log.error("Read failed", e);
    }
}
```

### 3.1 Common checked exceptions
| Exception | Thrown when... |
|-----------|----------------|
| `IOException` | I/O fails (file/network) |
| `FileNotFoundException` | File doesn't exist (subclass of IOException) |
| `SQLException` | Database access error |
| `ClassNotFoundException` | A class can't be loaded by name |
| `InterruptedException` | A blocked thread is interrupted |
| `ParseException` | Parsing fails |

### 3.2 When to use checked exceptions
- For **recoverable conditions** the **caller can reasonably be expected to handle** (e.g., retry, fall back, prompt user).
- ⚠️ **Modern critique:** checked exceptions are controversial. They force boilerplate, don't compose well with lambdas/streams, and many modern frameworks (incl. Spring) **prefer unchecked exceptions**. Use them sparingly — only when the caller can genuinely act on them.

---

## 4. Unchecked Exceptions (Runtime)

An **unchecked exception** (subclass of `RuntimeException`) is **not** checked by the compiler — you're not forced to catch or declare it. They usually indicate **programming errors / bugs** that should be **fixed**, not caught.

```java
String s = null;
s.length();   // NullPointerException — a bug; fix it, don't catch it everywhere
```

### 4.1 Common unchecked exceptions
| Exception | Thrown when... |
|-----------|----------------|
| `NullPointerException` (NPE) | Calling a method/field on `null` |
| `IllegalArgumentException` | A method gets an invalid argument |
| `IllegalStateException` | An object is in the wrong state for the call |
| `ArrayIndexOutOfBoundsException` | Invalid array index (Phase 1.2) |
| `IndexOutOfBoundsException` | Invalid list/string index |
| `ClassCastException` | Invalid reference cast (Phase 1.2) |
| `NumberFormatException` | Parsing a malformed number (subclass of IAE) |
| `UnsupportedOperationException` | Operation not supported (e.g., mutating an immutable list) |
| `ConcurrentModificationException` | Modifying a collection during iteration (Phase 1.6) |
| `ArithmeticException` | e.g., integer `/ 0` |

### 4.2 When to use unchecked exceptions
- For **programming errors** and **violated preconditions** (e.g., `IllegalArgumentException` for bad input, `IllegalStateException` for wrong state).
- For conditions the caller **cannot reasonably recover from**.
- This is the **modern default** for application/business exceptions (especially in Spring backends).

### 4.3 Validating preconditions (fail fast)
Throw unchecked exceptions early for invalid inputs:
```java
public void setAge(int age) {
    if (age < 0)
        throw new IllegalArgumentException("age must be >= 0, was " + age);
    this.age = age;
}

Objects.requireNonNull(name, "name must not be null");   // throws NPE with a message
```

---

## 5. Checked vs Unchecked — Decision Framework

```
Is the condition a programming bug / unrecoverable by the caller?
│
├─ Yes (bad arg, null, wrong state, logic error)  → UNCHECKED (RuntimeException)
│
└─ Can the caller realistically recover (retry, fallback, prompt)?
      ├─ Yes, and you want to force handling      → CHECKED (rare, use sparingly)
      └─ No / want clean code & lambdas            → UNCHECKED
```

| Use **unchecked** when... | Use **checked** when... |
|---------------------------|--------------------------|
| It's a bug (null, bad index) | It's an expected, recoverable I/O/external failure |
| Invalid arguments/state | The caller must be forced to deal with it |
| Business rule violations (common choice) | (Used sparingly in modern code) |

> **Modern guidance:** prefer **unchecked** exceptions for most application code. Reserve checked exceptions for the few cases where forcing the caller to handle a recoverable condition genuinely adds value.

---

## 6. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Catching `Throwable` or `Error` | Catch specific `Exception` subtypes only |
| Catching `Exception` broadly | Catch the narrowest type that applies |
| Catching unchecked exceptions to "handle" bugs | Fix the bug instead |
| Overusing checked exceptions | Prefer unchecked for non-recoverable conditions |
| Swallowing exceptions (empty catch) | Log and/or rethrow (next notes) |
| Throwing raw `Exception`/`RuntimeException` | Throw specific, meaningful types |
| Ignoring `InterruptedException` | Restore the interrupt flag (Phase 1.10) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Spring favors unchecked exceptions:** Spring wraps `SQLException` (checked) into `DataAccessException` (unchecked) so your code isn't littered with `try-catch` (Phase 5.4).
- **`@ControllerAdvice` / `@ExceptionHandler`** map exceptions to HTTP responses — custom unchecked exceptions are the idiomatic way to signal API errors (Phase 5.3).
- **`@Transactional` rollback** triggers on unchecked (RuntimeException) by default; checked exceptions don't roll back unless declared (Phase 5.5) — a critical, often-surprising detail.
- **Validation** throws `IllegalArgumentException`/`ConstraintViolationException` → mapped to `400 Bad Request`.
- **Reading exception hierarchies** is essential for debugging production stack traces (next-but-one note).

---

## 8. Quick Self-Check Questions

1. What is the root class of all exceptions, and what are its two main branches?
2. Why should you never catch `Error`? Give two examples of `Error`.
3. What's the precise rule that determines if an exception is checked or unchecked?
4. What does the compiler force you to do with a checked exception?
5. Give three checked and three unchecked exceptions.
6. When should you use a checked vs an unchecked exception?
7. Why do modern frameworks (and Spring) prefer unchecked exceptions?
8. How do you validate preconditions and fail fast?

---

## 9. Key Terms Glossary

- **Throwable:** root class of everything that can be thrown.
- **Error:** serious, unrecoverable JVM-level problem (don't catch).
- **Exception:** recoverable application-level problem.
- **Checked exception:** compiler-enforced; must be caught or declared.
- **Unchecked / RuntimeException:** not compiler-enforced; usually a bug.
- **`NullPointerException` (NPE):** using a null reference.
- **`IllegalArgumentException` / `IllegalStateException`:** bad argument / wrong state.
- **Fail fast:** detect and throw on invalid input as early as possible.
- **`Objects.requireNonNull`:** convenience null-check that throws NPE.

---

*This is the first note of **Section 1.5 — Exception Handling**.*
*Next topic: **try-catch-finally & try-with-resources**.*
