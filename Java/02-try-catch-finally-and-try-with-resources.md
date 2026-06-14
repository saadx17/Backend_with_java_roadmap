# try-catch-finally & try-with-resources

> **Phase 1 — Java Language Mastery → 1.5 Exception Handling**
> Goal: Master the mechanics of handling exceptions — multiple catch, multi-catch, finally guarantees, the return-in-finally trap, and try-with-resources (AutoCloseable, suppressed exceptions).

---

## 0. The Big Picture

The `try-catch-finally` construct is how you **handle** exceptions:
- **`try`** — the code that might throw.
- **`catch`** — handles a specific exception type.
- **`finally`** — always runs (cleanup), whether or not an exception occurred.

```java
try {
    riskyOperation();
} catch (IOException e) {
    handle(e);
} finally {
    cleanup();             // always runs
}
```

For resources (files, connections, streams), **try-with-resources** is the modern, safer replacement for manual `finally` cleanup.

---

## 1. Basic try-catch

```java
try {
    int[] arr = new int[3];
    arr[5] = 10;                              // throws ArrayIndexOutOfBoundsException
} catch (ArrayIndexOutOfBoundsException e) {
    System.out.println("Bad index: " + e.getMessage());
}
```
- If the `try` block throws, execution jumps to the matching `catch`.
- If no exception occurs, `catch` blocks are skipped.
- If no `catch` matches, the exception **propagates** up the call stack.

---

## 2. Multiple Catch Blocks

Handle different exception types differently. Order matters: **most specific first**.

```java
try {
    process(input);
} catch (FileNotFoundException e) {       // most specific first
    log.error("File missing", e);
} catch (IOException e) {                  // broader (superclass of FileNotFound)
    log.error("I/O error", e);
} catch (Exception e) {                    // most general last
    log.error("Unexpected", e);
}
```
> ⚠️ A broader type **before** a narrower subtype is a **compile error** (the narrower catch would be unreachable):
```java
// catch (IOException e) {...}
// catch (FileNotFoundException e) {...}  // COMPILE ERROR: already caught above
```

---

## 3. Multi-Catch (Java 7+)

Catch **multiple unrelated** exception types in **one** block with `|` — avoids duplicated handling code:
```java
try {
    process();
} catch (IOException | SQLException e) {     // handle both the same way
    log.error("Operation failed", e);
}
```
### 3.1 Rules
- The types must **not** be in a subclass/superclass relationship with each other (would be redundant).
- The caught variable `e` is **implicitly final** — you can't reassign it.
- Its compile-time type is the **common supertype** of the listed exceptions.

---

## 4. The `finally` Block

`finally` runs **no matter what** — after the try (and any catch), whether the block completed normally, threw, or even returned. It's for **cleanup** (closing resources, releasing locks).

```java
Connection conn = null;
try {
    conn = getConnection();
    conn.executeUpdate(sql);
} catch (SQLException e) {
    log.error("DB error", e);
} finally {
    if (conn != null) conn.close();   // always runs — release the resource
}
```

### 4.1 Execution guarantees
`finally` runs in essentially all cases:
| Scenario | Does `finally` run? |
|----------|---------------------|
| `try` completes normally | ✅ Yes |
| Exception caught | ✅ Yes (after catch) |
| Exception not caught (propagates) | ✅ Yes (then propagates) |
| `return` inside try or catch | ✅ Yes (before actually returning) |
| `break`/`continue` exits the block | ✅ Yes |
| **`System.exit()`** | ❌ No (JVM terminates) |
| JVM crash / power loss / killed thread | ❌ No |

### 4.2 ⚠️ The return-in-finally trap (DON'T do this)
A `return` (or `throw`) inside `finally` **overrides** any return/exception from the try/catch — silently swallowing results and exceptions:
```java
int bad() {
    try {
        return 1;
    } finally {
        return 2;     // BUG: overrides the try's return -> method returns 2!
    }
}

int worse() {
    try {
        throw new RuntimeException("real error");
    } finally {
        return 0;     // BUG: swallows the exception entirely!
    }
}
```
> **Never `return` or `throw` from a `finally` block.** It hides real results and exceptions. Use `finally` only for cleanup.

### 4.3 `finally` modifying the return value (subtle)
```java
int tricky() {
    int x = 1;
    try {
        return x;        // the value 1 is "captured" here
    } finally {
        x = 99;          // does NOT change the already-captured return value
    }
}                        // returns 1, not 99
```
The return value is evaluated *before* `finally` runs (for primitives) — another reason to keep `finally` to pure cleanup.

---

## 5. try-with-resources (Java 7+) — the modern way

Manual `finally` cleanup is verbose and error-prone (forgetting to close, NPEs, nested try-catch in finally). **try-with-resources** automatically closes resources for you.

### 5.1 Syntax
```java
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    return br.readLine();
}   // br.close() called AUTOMATICALLY here — even if an exception is thrown
```
- Resources declared in the `()` are **automatically closed** at the end of the block, in **reverse** order of declaration.
- No explicit `finally` needed.

### 5.2 Multiple resources
```java
try (var in = new FileInputStream("in.txt");
     var out = new FileOutputStream("out.txt")) {   // both auto-closed (out first, then in)
    in.transferTo(out);
}
```

### 5.3 The `AutoCloseable` interface
Any resource usable in try-with-resources must implement **`AutoCloseable`** (or `Closeable`), which declares a single `close()` method:
```java
public class MyResource implements AutoCloseable {
    public MyResource() { System.out.println("opened"); }
    public void use()   { System.out.println("using"); }
    @Override public void close() { System.out.println("closed"); }
}

try (MyResource r = new MyResource()) {
    r.use();
}   // prints: opened, using, closed
```
Most JDK resources (streams, readers, `Connection`, `Statement`, `ResultSet`, `Scanner`) already implement it.

### 5.4 Enhanced try-with-resources (Java 9+)
You can use an **already-declared effectively-final** variable directly (no need to re-declare in the parentheses):
```java
BufferedReader br = new BufferedReader(...);    // effectively final
try (br) {                                       // Java 9+: use existing variable
    br.readLine();
}
```

---

## 6. Suppressed Exceptions

A subtle but important detail: what if the `try` body throws **and** `close()` *also* throws?

- With **manual** try-finally, the exception from `close()` in `finally` would **replace** (hide) the original — losing the real error.
- With **try-with-resources**, the original exception is preserved, and the `close()` exception is attached as a **suppressed exception**.

```java
try (MyResource r = new MyResource()) {
    throw new RuntimeException("primary");     // this is the main exception
}   // if close() also throws, that's SUPPRESSED, not lost

// retrieve suppressed exceptions:
catch (Exception e) {
    for (Throwable s : e.getSuppressed()) {    // the close() exception(s)
        log.warn("Suppressed: ", s);
    }
}
```
> This is a key advantage of try-with-resources: **you never silently lose the original exception** when cleanup also fails.

---

## 7. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Broader catch before narrower (unreachable) | Order most-specific first |
| `return`/`throw` in `finally` | Never — use finally only for cleanup |
| Manual `close()` in `finally` (verbose, leaks) | Use try-with-resources |
| Forgetting to close resources | try-with-resources auto-closes |
| Losing the original exception when close() throws | try-with-resources keeps it (suppressed) |
| Empty catch block (swallowing) | Log and/or rethrow (next note) |
| Null check forgotten in manual finally | try-with-resources handles null gracefully |
| Catching exceptions you can't handle | Let them propagate |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **try-with-resources is mandatory** for JDBC (`Connection`/`Statement`/`ResultSet`), file I/O, and any `AutoCloseable` — prevents connection/resource leaks that exhaust pools (Phase 4, 13).
- **Spring `JdbcTemplate`/JPA** manage resource closing for you (built on these patterns) — but understanding them explains the abstraction.
- **`finally`/cleanup discipline** prevents leaks in long-running servers (Phase 1.11).
- **Suppressed exceptions** appear in real stack traces (next note) — knowing to read `Suppressed:` sections aids debugging.
- **`@Transactional`** wraps method calls with commit/rollback logic conceptually similar to try-finally (Phase 5.5).

---

## 9. Quick Self-Check Questions

1. In what order should multiple catch blocks be arranged, and why?
2. What is multi-catch, and what's a restriction on the types it can combine?
3. When does `finally` run, and name two cases where it does NOT.
4. Why must you never `return` or `throw` from a `finally` block?
5. What does try-with-resources do, and in what order are resources closed?
6. What interface must a resource implement for try-with-resources?
7. What is a suppressed exception, and why is it important?
8. What's the advantage of try-with-resources over manual try-finally?

---

## 10. Key Terms Glossary

- **`try`/`catch`/`finally`:** handle exceptions / handle a type / always-run cleanup.
- **Multiple catch:** several catch blocks, most specific first.
- **Multi-catch (`|`):** one catch handling several unrelated types.
- **`finally` guarantees:** runs except on `System.exit`, JVM crash, killed thread.
- **Return-in-finally trap:** a `finally` return/throw overriding the real result.
- **try-with-resources:** auto-closes resources declared in `()`.
- **`AutoCloseable` / `Closeable`:** interface providing `close()`.
- **Suppressed exception:** a secondary exception (e.g., from `close()`) attached to the primary.
- **Resource leak:** failing to release a resource (connection, file handle).

---

*Previous topic: **Exception Hierarchy — Checked vs Unchecked**.*
*Next topic: **throw vs throws & Custom Exceptions**.*
