# Spring Transaction Management

> **Phase 5 — Spring Framework & Spring Boot → 5.5 Spring Transaction Management**
> Goal: Master `@Transactional` — propagation, isolation, readOnly/timeout/rollback rules, proxy-based mechanics (and their gotchas), and programmatic transactions.

---

## 0. The Big Picture

Spring's **`@Transactional`** brings declarative transactions (recall ACID, Phase 4.5) to your services — wrap a method in a transaction with one annotation, instead of manually managing `BEGIN`/`COMMIT`/`ROLLBACK` (recall JDBC transactions, Phase 4.7). It's the standard way to ensure data consistency.

```
@Transactional
public void transfer(...) {  /* multiple DB ops */  }
// Spring: open a transaction -> run the method -> commit (or rollback on exception)
```

> This ties together transactions (Phase 4.5), JPA (Phase 5.4 — defines the persistence-context boundary), and AOP (Phase 5.1 — `@Transactional` is implemented via proxies). The proxy mechanism causes the famous gotchas (§5).

---

## 1. Declarative Transactions with @Transactional

Annotate a method (or class) to make it transactional:
```java
@Service
public class AccountService {

    @Transactional                          // method runs in a transaction
    public void transfer(Long from, Long to, BigDecimal amount) {
        debit(from, amount);                // step 1
        credit(to, amount);                 // step 2
        // commit if the method returns normally; rollback if an exception is thrown
    }
}
```
> Spring automatically: opens a transaction before the method, **commits** if it returns normally, **rolls back** if it throws (per the rollback rules — §4). Without it, each DB statement auto-commits separately (recall Phase 4.7) → no atomicity. This replaces manual JDBC transaction handling (Phase 4.7).

---

## 2. The Rollback Rule (Critical Gotcha)

> ⚠️ **By default, `@Transactional` rolls back on unchecked exceptions (`RuntimeException` and `Error`) — but NOT on checked exceptions.** (Recall Phase 1.5 — this is *the* reason to prefer unchecked exceptions for business errors.)
```java
@Transactional
public void doWork() throws IOException {
    repository.save(entity);
    throw new IOException("oops");   // CHECKED -> does NOT roll back! The save COMMITS!
}

@Transactional
public void doWork2() {
    repository.save(entity);
    throw new IllegalStateException("oops");   // UNCHECKED -> rolls back (save undone)
}
```
### 2.1 Overriding the rule
```java
@Transactional(rollbackFor = IOException.class)      // also roll back on this checked exception
@Transactional(noRollbackFor = ValidationException.class)  // DON'T roll back on this
```
> ⚠️ This is one of the most surprising Spring behaviors: a **checked exception thrown from a `@Transactional` method does NOT roll back the transaction** by default — committing partial changes! Either use **unchecked exceptions** for business errors (recommended — Phase 1.5) or specify `rollbackFor`.

---

## 3. Propagation (How Transactions Nest)

**Propagation** controls what happens when a transactional method is called **from within another transaction** — does it join, suspend, or start a new one?
| Propagation | Behavior |
|-------------|----------|
| **`REQUIRED`** (default) | Join the existing transaction, or start one if none |
| **`REQUIRES_NEW`** | Always start a **new** transaction (suspend the current one) |
| **`SUPPORTS`** | Join if one exists, else run non-transactionally |
| **`MANDATORY`** | Must run in an existing transaction (else error) |
| **`NEVER`** | Must run with NO transaction (else error) |
| **`NESTED`** | A savepoint within the current transaction (partial rollback) |
```java
@Transactional                                       // REQUIRED (default) — joins the caller's tx
public void placeOrder() {
    saveOrder();
    auditLog();                                       // runs in the SAME transaction
}

@Transactional(propagation = Propagation.REQUIRES_NEW)  // independent transaction
public void auditLog() {
    // commits SEPARATELY — survives even if the outer transaction rolls back
}
```
> ⭐ **`REQUIRED` (default)** is right for most cases — related operations share one all-or-nothing transaction. **`REQUIRES_NEW`** is for operations that must commit independently (e.g., audit logging that should persist even if the main operation fails). Recall savepoints (Phase 4.5) ≈ `NESTED`.

---

## 4. Other @Transactional Attributes

```java
@Transactional(
    readOnly = true,                                 // read optimization (Phase 5.4)
    isolation = Isolation.READ_COMMITTED,            // isolation level (Phase 4.5)
    timeout = 30,                                    // seconds before forced rollback
    rollbackFor = Exception.class                    // rollback on all exceptions
)
```
| Attribute | Effect |
|-----------|--------|
| **`readOnly`** | Hint for read-only ops; Hibernate skips dirty-checking → faster (recall Phase 5.4) |
| **`isolation`** | Sets the DB isolation level (Phase 4.5 — READ_COMMITTED default, etc.) |
| **`timeout`** | Roll back if the transaction runs too long |
| **`rollbackFor`/`noRollbackFor`** | Customize rollback exceptions (§2) |
> **`readOnly = true`** is a key performance lever for read methods (recall Phase 5.4). **`isolation`** maps directly to the isolation levels from Phase 4.5 — use stricter levels (SERIALIZABLE) only when correctness demands it.

---

## 5. Proxy-Based Mechanics & Gotchas (Critical)

> **`@Transactional` is implemented via AOP proxies (recall Phase 5.1 AOP).** Spring wraps the bean in a proxy that opens/commits the transaction around the method. This causes two famous gotchas:

### 5.1 Public methods only
> ⚠️ `@Transactional` works **only on `public` methods** — the proxy can't intercept `private`/`protected`/package-private methods (recall Phase 5.1 — CGLIB can't advise non-public methods). A `@Transactional` private method silently does nothing.

### 5.2 The self-invocation problem (the #1 gotcha — recall Phase 5.1)
> ⚠️ **A `@Transactional` method called from *within the same class* bypasses the proxy → the transaction annotation is IGNORED.**
```java
@Service
public class OrderService {
    public void process() {
        createOrder();                       // ❌ internal call — bypasses the proxy!
    }
    @Transactional
    public void createOrder() {              // its @Transactional is NOT applied when called from process()
        // ...
    }
}
```
**Fixes (recall Phase 5.1):** move `createOrder()` to another bean, inject the bean into itself, or use `AopContext.currentProxy()`. This is the **most common "why isn't my transaction working?"** bug.

### 5.3 Only works on Spring beans
`@Transactional` only applies to **Spring-managed beans** (Phase 5.1) — not on objects you `new` yourself.

---

## 6. Programmatic Transactions (TransactionTemplate)

For fine-grained control (e.g., a transaction around only *part* of a method, or dynamic logic), use **`TransactionTemplate`** instead of the annotation:
```java
@Service
public class OrderService {
    private final TransactionTemplate txTemplate;
    public OrderService(PlatformTransactionManager tm) {
        this.txTemplate = new TransactionTemplate(tm);
    }
    public void process() {
        // ... non-transactional work ...
        txTemplate.execute(status -> {       // explicit transaction boundary
            saveOrder();
            updateInventory();
            return null;                     // (or status.setRollbackOnly() to force rollback)
        });
        // ... more non-transactional work ...
    }
}
```
> Use **programmatic transactions** when you need a transaction around a *specific block* (not the whole method) or conditional transaction logic. **Declarative (`@Transactional`) is preferred** for the common case (cleaner); programmatic is the escape hatch.

---

## 7. Transactions & the Persistence Context (recall Phase 5.4)

`@Transactional` defines the **boundary of the JPA persistence context** (Phase 5.4):
- The persistence context (first-level cache) lives for the transaction's duration.
- **Dirty checking** (auto-UPDATE — Phase 5.4) works *within* the transaction.
- **Lazy loading** works *within* the transaction; accessing lazy data after it ends → `LazyInitializationException` (Phase 5.4).
> This is why service methods that load entities, modify them, and rely on lazy loading **must be `@Transactional`** — the persistence context (and thus dirty checking and lazy loading) only exists within the transaction.

---

## 8. Distributed Transactions (JTA — Awareness)

A single `@Transactional` covers **one database**. Spanning **multiple resources** (two databases, or a DB + a message broker) needs **distributed transactions** (JTA / XA / 2-phase commit) — which are **complex, slow, and avoided** in modern microservices.
> ⚠️ ACID transactions **don't span microservices/services** (recall Phase 4.5). For multi-service consistency, use the **Saga pattern** (compensating transactions) and **eventual consistency** instead of distributed transactions (Phase 12.6). The **Outbox pattern** (Phase 12.6) safely combines a DB write + event publishing within one local transaction.

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Expecting rollback on checked exceptions | It doesn't by default — use unchecked / `rollbackFor` |
| Self-invocation (internal call) — tx ignored | Call via another bean / injected self (Phase 5.1) |
| `@Transactional` on a private method | Only works on public bean methods |
| `@Transactional` on a non-bean | Only applies to Spring beans |
| Missing `readOnly` on read methods | Add `@Transactional(readOnly=true)` (Phase 5.4) |
| Long transactions holding connections | Keep them short (pool/contention — Phase 4.5/4.7) |
| Lazy loading after the tx ends | `LazyInitializationException` — fetch within the tx (Phase 5.4) |
| Trying distributed transactions across services | Use Saga/eventual consistency (Phase 12.6) |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Built on AOP** (Phase 5.1) — proxies cause the self-invocation/public-only gotchas.
- **Built on transactions** (Phase 4.5) — propagation, isolation, rollback are DB concepts.
- **Defines the JPA persistence-context boundary** (Phase 5.4 — dirty checking, lazy loading).
- **Unchecked exceptions for business errors** (Phase 1.5) → reliable rollback.
- **`readOnly` + short transactions** are performance levers (Phase 13).
- **Saga/Outbox** replace distributed transactions in microservices (Phase 12.6).
- **`@TransactionalEventListener`** (Phase 5.1 events) fires after commit.

---

## 11. Quick Self-Check Questions

1. What does `@Transactional` do, and what does it replace?
2. What's the default rollback rule, and why prefer unchecked exceptions?
3. What are the propagation types? When use `REQUIRES_NEW`?
4. What do `readOnly`, `isolation`, and `timeout` do?
5. Why does `@Transactional` only work on public methods of Spring beans?
6. What is the self-invocation problem, and how do you fix it?
7. When use programmatic transactions (`TransactionTemplate`)?
8. How does `@Transactional` relate to the JPA persistence context?
9. Why can't a single `@Transactional` span microservices, and what's the alternative?

---

## 12. Key Terms Glossary

- **`@Transactional`:** declarative transaction management.
- **Rollback rule:** rollback on unchecked exceptions by default (`rollbackFor` to override).
- **Propagation (REQUIRED/REQUIRES_NEW/NESTED/...):** how transactions nest.
- **Isolation / readOnly / timeout:** DB isolation level / read optimization / time limit.
- **Proxy-based:** `@Transactional` works via AOP proxies (Phase 5.1).
- **Self-invocation problem:** internal calls bypass the proxy (tx ignored).
- **`TransactionTemplate`:** programmatic transaction control.
- **Persistence-context boundary:** the transaction scopes dirty checking & lazy loading.
- **JTA / distributed transaction:** spanning multiple resources (avoided).
- **Saga / Outbox:** microservice consistency patterns (Phase 12.6).

---

*This is the note for **Section 5.5 — Spring Transaction Management**.*
*Next section in roadmap: **5.6 Spring Security**.*
