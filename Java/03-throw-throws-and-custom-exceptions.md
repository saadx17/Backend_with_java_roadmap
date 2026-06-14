# throw vs throws & Custom Exceptions

> **Phase 1 — Java Language Mastery → 1.5 Exception Handling**
> Goal: Master the `throw` and `throws` keywords (and how they differ) and how to design custom exceptions — when to create them, checked vs unchecked, error codes/context, and exception chaining.

---

## 0. The Big Picture

- **`throw`** — an **action**: actually *throws* an exception object at runtime.
- **`throws`** — a **declaration**: states in a method signature that it *might* throw certain (checked) exceptions.

```java
public void withdraw(double amt) throws InsufficientFundsException {  // throws: declaration
    if (amt > balance)
        throw new InsufficientFundsException("Not enough funds");     // throw: action
}
```

Custom exceptions let you model your domain's error conditions with meaningful types, messages, and context.

---

## 1. `throw` — Throwing an Exception

`throw` raises an exception, immediately interrupting normal flow and propagating up the call stack until caught (or terminating the thread).

```java
throw new IllegalArgumentException("age must be >= 0");
```

### 1.1 Rules
- You throw an **instance** (an object), not a class: `throw new X()`, not `throw X`.
- The object must be a `Throwable` (or subclass).
- Code after a `throw` is **unreachable** (compile error if you put statements after it in the same block).
- Throwing **unchecked** exceptions needs no declaration; throwing **checked** ones requires a `throws` clause (or a surrounding try-catch).

```java
public int parsePositive(String s) {
    int n = Integer.parseInt(s);            // may throw NumberFormatException (unchecked)
    if (n < 0)
        throw new IllegalArgumentException("must be positive: " + n);
    return n;
}
```

---

## 2. `throws` — Declaring Exceptions

`throws` in a method signature declares which **checked** exceptions the method may propagate, shifting the handling responsibility to the **caller**.

```java
public String readConfig(String path) throws IOException {   // caller must handle
    return Files.readString(Path.of(path));
}
```
- Only **checked** exceptions *need* to be declared (unchecked can be declared for documentation but it's optional).
- Multiple exceptions: `throws IOException, SQLException`.
- The caller must then **catch** them or **re-declare** `throws`.

### 2.1 `throw` vs `throws` at a glance
| | `throw` | `throws` |
|---|---------|----------|
| What it is | Statement (action) | Method-signature clause (declaration) |
| Where | Inside a method body | In the method signature |
| Used with | An exception **instance** | Exception **class names** |
| Purpose | Actually raise an exception | Declare possible checked exceptions |
| Count | Throws one at a time | Can list several |

### 2.2 Overriding and `throws` (recap from Inheritance note)
An overriding method **cannot declare broader checked exceptions** than the method it overrides (it may declare the same, narrower, or none). Unchecked exceptions are unrestricted.

---

## 3. Exception Propagation

When an exception is thrown and not caught in the current method, it **propagates up the call stack** until a matching `catch` is found — or the thread terminates (printing the stack trace).

```
main() calls a() calls b() calls c()  -- c() throws
   c() throws -> no catch -> propagates to b()
   b() no catch -> propagates to a()
   a() has try-catch -> handled here
```
> Each method either handles the exception or passes it up. Uncaught exceptions reaching the top of a thread are handled by the thread's `UncaughtExceptionHandler` (default: print stack trace and terminate the thread — Phase 1.10).

---

## 4. Custom Exceptions

You create custom exception classes to represent **domain-specific** error conditions with clear, meaningful types.

### 4.1 Custom unchecked exception (the common case)
```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
    public ResourceNotFoundException(String message, Throwable cause) {  // chaining
        super(message, cause);
    }
}

// usage:
throw new ResourceNotFoundException("User " + id + " not found");
```

### 4.2 Custom checked exception
```java
public class InsufficientFundsException extends Exception {   // extends Exception = checked
    public InsufficientFundsException(String message) {
        super(message);
    }
}
```

### 4.3 The four standard constructors
Mirror `Throwable`'s constructors so your exception supports messages and chaining:
```java
public class OrderException extends RuntimeException {
    public OrderException() { super(); }
    public OrderException(String message) { super(message); }
    public OrderException(String message, Throwable cause) { super(message, cause); }
    public OrderException(Throwable cause) { super(cause); }
}
```

---

## 5. When to Create a Custom Exception

| Create one when... | Use a built-in when... |
|--------------------|------------------------|
| The condition is **domain-specific** (`PaymentDeclinedException`) | A standard exception fits (`IllegalArgumentException`) |
| You want callers to **catch it specifically** | No special handling needed |
| You need to attach **extra context** (error code, IDs) | A message suffices |
| It improves **readability/intent** of the API | It would just duplicate a built-in |

> Don't create a custom exception for every error — reuse `IllegalArgumentException`, `IllegalStateException`, etc. where they fit. Create custom types when they add **meaning** or **handling distinction**.

### 5.1 Checked vs unchecked for custom exceptions
- **Default to unchecked** (`extends RuntimeException`) for business/domain errors — cleaner, works with lambdas/streams, and is the Spring convention.
- Use **checked** only when the caller can realistically recover and you want to *force* them to handle it.

---

## 6. Adding Error Codes and Context

Custom exceptions can carry structured data beyond a message — invaluable for APIs, logging, and i18n.

```java
public class ApiException extends RuntimeException {
    private final String errorCode;
    private final int httpStatus;

    public ApiException(String errorCode, int httpStatus, String message) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }
    public String getErrorCode() { return errorCode; }
    public int getHttpStatus()   { return httpStatus; }
}

// usage:
throw new ApiException("USER_NOT_FOUND", 404, "User " + id + " does not exist");
```
This lets a global handler map the exception → a structured HTTP error response (Phase 5.3, 7) with a stable error code clients can react to.

---

## 7. Exception Chaining (the `cause`)

**Exception chaining** wraps a low-level exception inside a higher-level one, preserving the original as the **cause** — so you keep the full diagnostic trail while exposing a meaningful abstraction.

```java
try {
    jdbcTemplate.query(...);
} catch (SQLException e) {
    throw new DataAccessException("Failed to load user " + id, e);  // wrap: e is the cause
}
```

### 7.1 Why chain?
- **Abstraction:** callers deal with a meaningful exception (`DataAccessException`) instead of a leaky low-level one (`SQLException`).
- **Diagnostics:** the original cause is preserved — the stack trace shows `Caused by: ...` (next note).
- **Don't lose information:** never catch an exception and throw a new one *without* passing the cause.

```java
// BAD: loses the original cause
catch (SQLException e) {
    throw new DataAccessException("DB error");   // original e is LOST
}
// GOOD: preserve it
catch (SQLException e) {
    throw new DataAccessException("DB error", e);
}
```

### 7.2 Accessing the cause
```java
catch (DataAccessException e) {
    Throwable cause = e.getCause();   // the original SQLException
}
```
You can also chain manually with `initCause()`, but the constructor form is preferred.

---

## 8. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Confusing `throw` (action) and `throws` (declaration) | `throw` an instance; `throws` declares types |
| Throwing a new exception without the cause | Pass the original as `cause` (chaining) |
| Creating a custom exception for every case | Reuse built-ins where they fit |
| Making domain exceptions checked by default | Prefer unchecked (RuntimeException) |
| Missing the `(String, Throwable)` constructor | Always include it for chaining |
| Declaring broader checked exceptions when overriding | Same/narrower/none only |
| Putting code after `throw` | It's unreachable — remove it |
| Generic messages with no context | Include IDs/values/error codes |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Custom unchecked exceptions** + **`@RestControllerAdvice`/`@ExceptionHandler`** is the standard Spring pattern for clean API error handling (Phase 5.3).
- **Error codes + HTTP status** in exceptions map directly to **ProblemDetail (RFC 7807)** responses (Phase 5.3, 7).
- **Exception chaining** is how Spring wraps `SQLException` → `DataAccessException` while preserving the cause (Phase 5.4).
- **Unchecked + `@Transactional`:** RuntimeExceptions trigger rollback by default; checked ones don't (Phase 5.5) — design domain exceptions accordingly.
- **`throws` discipline** keeps service interfaces clean; over-declaring checked exceptions leaks implementation details.

---

## 10. Quick Self-Check Questions

1. What's the difference between `throw` and `throws`?
2. Do you throw a class or an instance? Can code follow a `throw`?
3. Which exceptions must be declared with `throws`?
4. How does exception propagation work up the call stack?
5. When should you create a custom exception vs reuse a built-in?
6. Should custom domain exceptions usually be checked or unchecked? Why?
7. What is exception chaining, and why must you preserve the cause?
8. How would you add an error code and HTTP status to a custom exception?

---

## 11. Key Terms Glossary

- **`throw`:** statement that raises an exception instance.
- **`throws`:** signature clause declaring possible checked exceptions.
- **Propagation:** an uncaught exception traveling up the call stack.
- **Custom exception:** a user-defined exception class.
- **Checked/unchecked custom exception:** extends `Exception` / `RuntimeException`.
- **Exception chaining:** wrapping an exception, preserving the original as `cause`.
- **`cause` / `getCause()`:** the wrapped original exception.
- **Error code:** a stable identifier attached to an exception for clients/logs.
- **`UncaughtExceptionHandler`:** handles exceptions that escape a thread.

---

*Previous topic: **try-catch-finally & try-with-resources**.*
*Next topic: **Best Practices & Reading Stack Traces**.*
