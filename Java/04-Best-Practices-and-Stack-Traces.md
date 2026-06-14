# Exception Handling Best Practices & Reading Stack Traces

> **Phase 1 — Java Language Mastery → 1.5 Exception Handling**
> Goal: Master exception-handling best practices (never swallow, fail fast, log-and-rethrow, specific exceptions) and how to read and interpret stack traces.

---

## 0. The Big Picture

Knowing the *mechanics* of exceptions isn't enough — using them **well** is what separates robust production code from fragile code. This note covers the discipline of good exception handling and the essential skill of **reading stack traces** to diagnose failures.

---

## 1. Best Practices

### 1.1 Never swallow exceptions
An **empty catch block** hides errors — the program continues in a broken state with no trace. This is one of the worst habits in Java.
```java
// TERRIBLE:
try {
    process();
} catch (Exception e) {
    // nothing — error vanishes silently
}

// BAD: print and continue (loses context in real systems)
catch (Exception e) { e.printStackTrace(); }

// GOOD: log with context, then handle or rethrow
catch (IOException e) {
    log.error("Failed to process file {}", path, e);
    throw new ProcessingException("Processing failed for " + path, e);
}
```
> If you truly must ignore an exception, **document why** with a comment — silence should be a deliberate, explained decision.

### 1.2 Never catch `Throwable` or `Error`
```java
// WRONG:
catch (Throwable t) { ... }   // also catches OutOfMemoryError, StackOverflowError
catch (Error e) { ... }       // these are unrecoverable
```
Catch the **narrowest** `Exception` type you can actually handle. Catching `Throwable`/`Error` masks fatal JVM problems.

### 1.3 Catch specific exceptions, not `Exception`
```java
// Vague:
catch (Exception e) { ... }

// Specific — handle each meaningfully:
catch (FileNotFoundException e) { useDefaultConfig(); }
catch (IOException e) { retry(); }
```
Specific catches let you respond appropriately and avoid accidentally catching bugs (NPEs, etc.).

### 1.4 Fail fast
Detect invalid states/inputs **as early as possible** and throw immediately, rather than letting bad data propagate:
```java
public void transfer(Account from, Account to, BigDecimal amount) {
    Objects.requireNonNull(from, "from");
    Objects.requireNonNull(to, "to");
    if (amount.signum() <= 0)
        throw new IllegalArgumentException("amount must be positive");
    // ... proceed knowing inputs are valid
}
```
Failing fast turns subtle, far-away bugs into obvious, immediate ones.

### 1.5 Log OR rethrow — not both (avoid duplicate logging)
```java
// BAD: logs the same error at every layer -> noisy duplicate stack traces
catch (IOException e) {
    log.error("error", e);
    throw new ServiceException(e);   // upper layer logs it AGAIN
}
```
> Either **handle and log** the exception at the layer that can deal with it, **or** rethrow (let a higher layer / global handler log it once). Logging at every level produces duplicate, confusing traces.

### 1.6 Wrap low-level exceptions (chaining) — preserve the cause
Translate leaky low-level exceptions into meaningful ones, **always passing the cause** (recall the chaining note):
```java
catch (SQLException e) {
    throw new DataAccessException("Could not save order " + id, e);  // keep the cause
}
```

### 1.7 Don't use exceptions for normal control flow
Exceptions are for **exceptional** conditions, not routine logic — they're expensive (stack trace capture) and obscure intent:
```java
// BAD: using exceptions as control flow
try {
    return map.get(key).toString();
} catch (NullPointerException e) {
    return "default";
}
// GOOD:
return map.getOrDefault(key, defaultValue).toString();
```

### 1.8 Clean up resources properly
Use **try-with-resources** (previous note), not manual finally, to guarantee cleanup and avoid leaks.

### 1.9 Provide context in messages
```java
// Useless:
throw new IllegalArgumentException("invalid");
// Useful:
throw new IllegalArgumentException("Invalid status '" + status + "' for order " + orderId);
```
Include the offending **values and identifiers** — your future self debugging production will thank you.

### 1.10 Don't log sensitive data
Never put passwords, tokens, full card numbers, or PII into exception messages/logs (they leak into log aggregators) — Phase 9, 15.

### 1.11 Best-practices checklist
```
[ ] No empty catch blocks (never swallow)
[ ] Catch specific exceptions, not Exception/Throwable
[ ] Fail fast on invalid input/state
[ ] Log OR rethrow, not both
[ ] Wrap-and-chain (preserve the cause)
[ ] Don't use exceptions for normal control flow
[ ] Use try-with-resources for cleanup
[ ] Include context (IDs, values) in messages
[ ] Never log secrets/PII
[ ] Prefer unchecked for domain errors
```

---

## 2. Reading and Interpreting Stack Traces

A **stack trace** is the snapshot of the call stack at the moment an exception was thrown — your primary debugging tool.

### 2.1 Anatomy of a stack trace
```
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "String.length()" because "name" is null
    at com.example.UserService.validate(UserService.java:42)
    at com.example.UserService.register(UserService.java:28)
    at com.example.Main.main(Main.java:15)
```
Reading it:
| Part | Meaning |
|------|---------|
| `java.lang.NullPointerException` | The **exception type** |
| `Cannot invoke ... because "name" is null` | The **message** (Java 14+ helpful NPE messages tell you *what* was null) |
| First `at ...` line | **Where it was thrown** (the most important line) |
| Each `at ...` below | The **call chain**, most recent first (top) → oldest (bottom) |
| `UserService.java:42` | File and **line number** |

### 2.2 How to read it (the method)
1. **Read the exception type and message first** — what went wrong.
2. **Look at the top `at` line** — where it was thrown (often the bug site).
3. **Find the first line from YOUR code** (ignore framework/JDK frames) — usually where to start investigating.
4. **Follow the chain down** to understand how you got there.

> The trace is **most-recent-first**: the top frame is where the exception originated; the bottom is the program entry point.

### 2.3 "Caused by:" — chained exceptions
When exceptions are chained (previous note), you'll see `Caused by:` sections — read them to find the **root cause**:
```
com.example.ServiceException: Failed to register user
    at com.example.UserService.register(UserService.java:30)
    ...
Caused by: java.sql.SQLException: connection refused
    at com.example.UserRepo.save(UserRepo.java:55)
    ...
```
> **The deepest `Caused by:` is usually the real root cause.** Here the actual problem is `connection refused`, wrapped in a `ServiceException`. Always read down to the last `Caused by`.

### 2.4 "Suppressed:" — from try-with-resources
Exceptions thrown by `close()` appear as `Suppressed:` entries (recall the try-with-resources note) — secondary failures during cleanup.

### 2.5 Common patterns to recognize
| Trace shows | Likely cause |
|-------------|--------------|
| `NullPointerException` | A null reference used (Java 14+ msg names the variable) |
| `ClassNotFoundException` / `NoClassDefFoundError` | Missing/misversioned dependency (Phase 1.11) |
| `StackOverflowError` (huge repeating trace) | Infinite recursion (Phase 0 stack note) |
| `OutOfMemoryError` | Heap/metaspace exhausted (Phase 1.11) |
| Same frame repeated | Recursion / retry loop |
| Long framework frames, one of your lines | Start at your line |

### 2.6 Capturing traces programmatically
```java
log.error("Operation failed", e);    // SLF4J logs the full trace (pass e as last arg)
String trace = ExceptionUtils.getStackTrace(e);  // (Apache Commons) as a String
```
> In real apps, **log the exception object** (don't just log `e.getMessage()` — you'd lose the trace). With SLF4J, pass the throwable as the **last argument**.

---

## 3. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Empty catch / swallowing | Always log or rethrow |
| Catching `Exception`/`Throwable` broadly | Catch specific types |
| Logging only `e.getMessage()` | Log the throwable to keep the stack trace |
| Logging the same exception at every layer | Log once (or rethrow) |
| Using exceptions for control flow | Use conditionals / `Optional` / `getOrDefault` |
| Throwing away the cause when wrapping | Chain it |
| Ignoring `Caused by:` | The deepest cause is usually the real bug |
| Vague messages | Include context (IDs, values) |
| Logging secrets/PII | Mask/exclude sensitive data |

---

## 4. Connection to Backend / Spring (Why This Matters Later)

- **Global exception handling** (`@RestControllerAdvice`) lets you log once and return clean error responses — embodying "log OR rethrow" (Phase 5.3).
- **Structured logging + MDC** (request/user IDs) makes traces traceable across a distributed system (Phase 9, 12).
- **Reading `Caused by:`** is *the* core skill for debugging production incidents (Phase 13, 16).
- **Fail fast + validation** maps to `400 Bad Request` and prevents bad data reaching the DB (Phase 5.3).
- **Never logging secrets/PII** is a security and compliance requirement (GDPR — Phase 15).
- **Don't-use-exceptions-for-control-flow** matters for performance in hot paths (stack trace capture is costly — Phase 13).

---

## 5. Quick Self-Check Questions

1. Why is an empty catch block dangerous, and what should you do instead?
2. Why should you never catch `Throwable` or `Error`?
3. What does "fail fast" mean and why is it valuable?
4. Why is "log OR rethrow" better than logging at every layer?
5. In a stack trace, where is the exception's origin — top or bottom?
6. What does `Caused by:` indicate, and which one is usually the root cause?
7. Why should you log the exception object, not just its message?
8. Give an example of misusing exceptions for control flow and its fix.

---

## 6. Key Terms Glossary

- **Swallowing:** catching an exception and doing nothing (anti-pattern).
- **Fail fast:** detecting and throwing on bad input/state immediately.
- **Log-or-rethrow:** handle and log once, or propagate — not both.
- **Wrap-and-chain:** translate to a meaningful exception, preserving the cause.
- **Stack trace:** snapshot of the call stack when an exception was thrown.
- **Frame:** one `at ...` line (a method call) in the trace.
- **`Caused by:`:** a chained underlying exception (root cause).
- **`Suppressed:`:** a secondary exception from resource `close()`.
- **Helpful NPE messages:** Java 14+ NPEs that name the null variable.
- **MDC:** Mapped Diagnostic Context — correlation data added to logs.

---

*Previous topic: **throw vs throws & Custom Exceptions**.*
*This completes **Section 1.5 — Exception Handling**.*
*Next section in roadmap: **1.6 Java Collections Framework**.*
