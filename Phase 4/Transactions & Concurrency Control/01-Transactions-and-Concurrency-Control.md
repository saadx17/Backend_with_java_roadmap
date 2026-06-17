# Transactions & Concurrency Control

> **Phase 4 — Databases → 4.5 Transactions and Concurrency Control**
> Goal: Master transactions — ACID properties, isolation levels, the concurrency problems they prevent, locking, MVCC, and optimistic vs pessimistic locking.

---

## 0. The Big Picture

A **transaction** is a group of database operations treated as a **single, all-or-nothing unit** — either *all* succeed (commit) or *all* fail (rollback). **Concurrency control** governs how the database keeps data correct when **many transactions run at once**.

```
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- step 1
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;   -- step 2
COMMIT;   -- both happen, or (on ROLLBACK) neither does -> money never vanishes
```

> Transactions are the foundation of **data correctness** in concurrent systems. Without them, a crash between steps, or two transactions interfering, corrupts your data. This underpins Spring's `@Transactional` (Phase 5.5) and connects to Java concurrency (Phase 1.10 — same hazards, different layer).

---

## 1. What Is a Transaction?

A transaction bundles multiple statements so they execute as one logical operation:
```sql
BEGIN;                                    -- start
  -- one or more statements
COMMIT;                                   -- make all changes permanent
-- or ROLLBACK; -- undo everything since BEGIN
```
- **Commit:** all changes become permanent and visible.
- **Rollback:** all changes are undone (as if nothing happened).
- **Savepoint:** a marker to partially roll back to (`SAVEPOINT sp; ... ROLLBACK TO sp;`).
> The classic example is a **bank transfer**: debit one account, credit another. If only the debit succeeds (crash in between), money disappears. A transaction guarantees **both or neither** (recall the DML transaction preview, Phase 4.2.1).

---

## 2. ACID Properties

Transactions provide four guarantees, remembered as **ACID**:
| Property | Guarantee | Meaning |
|----------|-----------|---------|
| **Atomicity** | All-or-nothing | Every statement succeeds, or all are rolled back |
| **Consistency** | Valid states only | The DB moves from one valid state to another (constraints hold — Phase 4.1) |
| **Isolation** | Concurrent = correct | Concurrent transactions don't corrupt each other's data (§4) |
| **Durability** | Committed = permanent | Once committed, data survives crashes/power loss (written to disk — recall Phase 0.1) |

### 2.1 Each property in detail
- **Atomicity:** the transfer either fully completes or fully reverts — no partial state. Implemented via a **write-ahead log (WAL)** that can undo/redo.
- **Consistency:** all constraints (PK, FK, CHECK, UNIQUE — Phase 4.1) and rules hold before and after. The transaction can't leave the DB in an invalid state.
- **Isolation:** even with many transactions running simultaneously, each behaves *as if* it ran alone (to a degree controlled by the isolation level — §3).
- **Durability:** after `COMMIT`, the data is safely persisted — a crash won't lose it (the WAL is flushed to durable storage; recall RAM is volatile, storage is not — Phase 0.1).
> ACID is *the* reason relational databases are trusted for critical data (money, orders, inventory). NoSQL systems often relax some of these for scale (CAP theorem — Phase 4.8).

---

## 3. Concurrency Problems (What Isolation Prevents)

When transactions run concurrently *without* proper isolation, several problems arise (these mirror Java's race conditions — Phase 1.10 — but at the database level):
| Problem | What happens |
|---------|--------------|
| **Dirty read** | T1 reads data T2 wrote but **hasn't committed** (and T2 might roll back!) |
| **Non-repeatable read** | T1 reads a row twice and gets **different values** (T2 committed an update in between) |
| **Phantom read** | T1 runs the same query twice and gets **different rows** (T2 inserted/deleted matching rows) |
| **Lost update** | Two transactions read-modify-write the same row; one overwrites the other's change |
| **Write skew** | Two transactions read overlapping data, make decisions, and write — together violating an invariant |

### 3.1 Examples
```
DIRTY READ:
  T2: UPDATE balance = 0 (not committed)
  T1: reads balance = 0   <- but T2 rolls back! T1 saw data that never existed.

NON-REPEATABLE READ:
  T1: reads balance = 100
  T2: UPDATE balance = 50; COMMIT
  T1: reads balance = 50  <- same query, different value within one transaction.

PHANTOM READ:
  T1: SELECT COUNT(*) WHERE status='ACTIVE' -> 5
  T2: INSERT a new ACTIVE row; COMMIT
  T1: SELECT COUNT(*) WHERE status='ACTIVE' -> 6  <- a "phantom" row appeared.

LOST UPDATE:
  T1: read count=10        T2: read count=10
  T1: write count=11       T2: write count=11   <- T2 overwrites T1; should be 12!
```
> ⚠️ **Lost update** is the database version of the `count++` race condition (recall Phase 1.10) — and the reason for locking (§5/§6). Different isolation levels prevent different subsets of these problems.

---

## 4. Isolation Levels

The **isolation level** controls how much transactions are protected from each other — trading **correctness** for **performance** (stricter = safer but slower/more contention).
| Level | Dirty read | Non-repeatable read | Phantom read |
|-------|:----------:|:-------------------:|:------------:|
| **READ UNCOMMITTED** | ❌ possible | ❌ possible | ❌ possible |
| **READ COMMITTED** | ✅ prevented | ❌ possible | ❌ possible |
| **REPEATABLE READ** | ✅ prevented | ✅ prevented | ❌ possible* |
| **SERIALIZABLE** | ✅ prevented | ✅ prevented | ✅ prevented |
(\* PostgreSQL's REPEATABLE READ also prevents phantoms via MVCC — stricter than the SQL standard.)

### 4.1 The levels explained
- **READ UNCOMMITTED:** weakest; allows dirty reads. Rarely used.
- **READ COMMITTED:** you only see committed data. **The default in PostgreSQL** (and most DBs) — a good balance for most apps.
- **REPEATABLE READ:** a transaction sees a consistent snapshot — re-reads return the same data. Prevents non-repeatable reads.
- **SERIALIZABLE:** strongest; transactions behave *as if* run one at a time (sequentially). Prevents all anomalies (including write skew) but has the most contention/overhead.
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;   -- set per transaction
BEGIN; ... COMMIT;
```
> **Trade-off:** higher isolation = fewer anomalies but more locking/conflicts (and potential serialization failures you must retry). **READ COMMITTED** is the pragmatic default; use **SERIALIZABLE** for critical invariants (financial, inventory) where correctness trumps throughput. (Spring's `@Transactional(isolation=...)` sets this — Phase 5.5.)

---

## 5. Locking

Databases use **locks** to control concurrent access — preventing conflicting operations on the same data (the DB-level analog of `synchronized` — Phase 1.10).
| Lock type | Effect |
|-----------|--------|
| **Shared (read) lock** | Multiple transactions can read; blocks writers |
| **Exclusive (write) lock** | One transaction writes; blocks all others |
| **Row-level lock** | Locks specific rows (fine-grained, high concurrency) |
| **Table-level lock** | Locks the whole table (coarse, low concurrency) |
| **Advisory lock** | Application-defined locks (PostgreSQL `pg_advisory_lock`) for custom coordination |

### 5.1 Explicit locking: SELECT FOR UPDATE / FOR SHARE
You can explicitly lock rows you intend to modify, to prevent concurrent changes (used for **pessimistic locking** — §7):
```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;   -- lock this row; others wait
-- now safely read-modify-write without a lost update
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;                                             -- lock released
```
- `FOR UPDATE`: exclusive lock — others can't read-for-update or write this row until you commit.
- `FOR SHARE`: shared lock — others can read but not modify.
> `SELECT ... FOR UPDATE` prevents lost updates by locking the row while you work with it (others block). It's the mechanism behind pessimistic locking (§7).

### 5.2 Deadlocks
Like in Java (Phase 1.10), database transactions can **deadlock** — each holding a lock the other needs:
```
T1: locks row A, wants row B
T2: locks row B, wants row A   -> both wait forever -> DEADLOCK
```
> Databases **detect deadlocks** automatically and **abort one transaction** (it gets a deadlock error and must retry). **Prevention** (same as Java, Phase 1.10): acquire locks in a **consistent order**, keep transactions short, minimize lock scope.

---

## 6. MVCC (Multi-Version Concurrency Control)

**MVCC** is how PostgreSQL (and many modern DBs) achieves isolation **without readers blocking writers** — the key to high concurrency.
```
Instead of locking rows for reads, MVCC keeps MULTIPLE VERSIONS of each row:
  - A writer creates a NEW version of the row (the old one stays).
  - Each transaction sees a consistent SNAPSHOT (the versions valid when it started).
  - Readers never block writers; writers never block readers.
```

### 6.1 How MVCC works
- Each row has hidden version metadata (transaction IDs marking when it became visible/invalid).
- A transaction sees only the row versions that were **committed before it started** (its snapshot).
- An `UPDATE` doesn't overwrite — it writes a **new version** and marks the old one dead.
- **Readers and writers don't block each other** → far better concurrency than pure locking.

### 6.2 The cost: dead tuples & VACUUM
> ⚠️ MVCC's old row versions ("**dead tuples**") accumulate and must be cleaned up — that's what **`VACUUM`/autovacuum** does (recall index bloat, Phase 4.4). This is also why high-update tables bloat. MVCC trades storage/cleanup overhead for excellent read/write concurrency.
> **MVCC is a major reason PostgreSQL handles concurrent workloads well** — reads get consistent snapshots without blocking writes.

---

## 7. Optimistic vs Pessimistic Locking

Two strategies for handling concurrent updates to the same data — a key application-level design choice:
| | **Pessimistic locking** | **Optimistic locking** |
|---|------------------------|------------------------|
| Assumption | Conflicts are **likely** | Conflicts are **rare** |
| How | **Lock** the row up front (`FOR UPDATE`) | **No lock**; check a `version` at write time |
| Conflict handling | Others wait | Detect conflict → reject & retry |
| Cost | Lock contention, possible deadlocks | Wasted work on retry (when conflicts happen) |
| Use when | High contention, critical correctness | Low contention, high concurrency (web apps) |

### 7.1 Pessimistic locking
```sql
BEGIN;
SELECT * FROM products WHERE id = 5 FOR UPDATE;   -- lock now; others block
UPDATE products SET stock = stock - 1 WHERE id = 5;
COMMIT;
```
Use when conflicts are likely (e.g., flash-sale inventory) and you must prevent them outright.

### 7.2 Optimistic locking (version column — recall Phase 4.3)
No lock during the read. At write time, check that the `version` hasn't changed since you read it; if it has, someone else updated it → fail and retry:
```sql
-- Read: version = 3
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 5 AND version = 3;     -- only succeeds if version is STILL 3
-- If 0 rows updated -> someone else changed it -> conflict -> retry
```
> ⚠️ **Optimistic locking is the common default for web apps** — most concurrent requests touch different rows, so locking everything is wasteful. The `version` column (Phase 4.3) detects the rare collision. This maps directly to **JPA's `@Version`** annotation, which throws `OptimisticLockException` on conflict (Phase 5.4). Pessimistic locking (`@Lock(PESSIMISTIC_WRITE)`) is for genuinely high-contention cases (e.g., e-commerce inventory — Project 5).

---

## 8. Transactions and Spring (preview — Phase 5.5)

In application code, you rarely write `BEGIN/COMMIT` manually — Spring's **`@Transactional`** manages it declaratively:
```java
@Transactional
public void transfer(Long from, Long to, BigDecimal amount) {
    debit(from, amount);
    credit(to, amount);
    // Spring commits if the method returns normally; rolls back on a RuntimeException
}
```
> ⚠️ Key Spring behavior (recall Phase 1.5): `@Transactional` **rolls back on unchecked (RuntimeException)** by default, **not** on checked exceptions (unless declared). This is *the* reason to prefer unchecked exceptions for business errors. (Full coverage — propagation, isolation, readOnly — in Phase 5.5.)

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Multi-step changes without a transaction | Wrap in BEGIN/COMMIT (or `@Transactional`) |
| Always using SERIALIZABLE "to be safe" | Use READ COMMITTED default; SERIALIZABLE only when needed |
| Lost updates from read-modify-write | Optimistic (version) or pessimistic (`FOR UPDATE`) locking |
| Long transactions holding locks | Keep transactions short (contention, MVCC bloat) |
| Inconsistent lock ordering → deadlocks | Acquire locks in a consistent order |
| Ignoring MVCC bloat on high-update tables | Let autovacuum run; monitor |
| Expecting `@Transactional` to roll back on checked exceptions | It doesn't by default — use unchecked / `rollbackFor` |
| Pessimistic locking everywhere | Default to optimistic for web apps |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **`@Transactional` (Phase 5.5):** declarative transactions — propagation, isolation, readOnly, rollback rules; the unchecked-exception rollback gotcha.
- **`@Version` optimistic locking (Phase 5.4):** the `version` column (Phase 4.3) in action; `OptimisticLockException`.
- **Pessimistic locking (`@Lock`)** for high-contention scenarios (inventory — Project 5).
- **Concurrency parallels (Phase 1.10):** dirty reads/lost updates ≈ race conditions; deadlocks; locking ≈ `synchronized`. Same problems, different layer.
- **Connection pooling (Phase 4.7, 13):** transactions hold a connection — long transactions starve the pool.
- **Distributed transactions / Saga (Phase 12.6):** ACID doesn't span microservices → eventual consistency patterns.
- **WAL & durability** connect to backups/replication (Phase 4.6).

---

## 11. Quick Self-Check Questions

1. What is a transaction, and what are commit/rollback/savepoint?
2. State the four ACID properties and what each guarantees.
3. Define dirty read, non-repeatable read, phantom read, and lost update.
4. What are the four isolation levels, and which anomalies does each prevent? Which is the default?
5. What's the difference between shared and exclusive locks? What does `SELECT FOR UPDATE` do?
6. What is a deadlock, and how is it handled/prevented?
7. How does MVCC provide isolation without blocking, and what's its cost?
8. Optimistic vs pessimistic locking — how does each work and when use it?
9. How does `@Transactional` relate to all this, and what's the rollback gotcha?

---

## 12. Key Terms Glossary

- **Transaction:** an all-or-nothing group of operations.
- **Commit / rollback / savepoint:** persist / undo / partial-undo marker.
- **ACID:** Atomicity, Consistency, Isolation, Durability.
- **WAL (write-ahead log):** durability/atomicity mechanism.
- **Dirty / non-repeatable / phantom read; lost update; write skew:** concurrency anomalies.
- **Isolation level (READ UNCOMMITTED/COMMITTED, REPEATABLE READ, SERIALIZABLE):** anomaly protection vs performance.
- **Lock (shared/exclusive, row/table/advisory):** concurrency control mechanism.
- **`SELECT FOR UPDATE` / `FOR SHARE`:** explicit row locking.
- **Deadlock:** mutual lock-waiting (DB detects & aborts one).
- **MVCC:** multi-version concurrency control (readers don't block writers).
- **Dead tuple / VACUUM:** old row version / cleanup.
- **Optimistic vs pessimistic locking:** version-check vs lock-upfront.

---

*This is the note for **Section 4.5 — Transactions and Concurrency Control**.*
*Next section in roadmap: **4.6 PostgreSQL**.*
