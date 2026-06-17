# SQL: DDL & DML

> **Phase 4 — Databases → 4.2 SQL Mastery**
> Goal: Master the SQL statements that define structure (DDL: CREATE/ALTER/DROP, indexes, views) and modify data (DML: INSERT/UPDATE/DELETE/UPSERT).

---

## 0. The Big Picture

**SQL (Structured Query Language)** is the language for relational databases. It splits into categories by purpose:
| Category | Stands for | Statements | Purpose |
|----------|-----------|------------|---------|
| **DDL** | Data **Definition** | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` | Define/change **structure** (tables, indexes) |
| **DML** | Data **Manipulation** | `INSERT`, `UPDATE`, `DELETE` | Change **data** (rows) |
| **DQL** | Data **Query** | `SELECT` | Read data (next note) |
| **DCL/TCL** | Control/Transaction | `GRANT`, `COMMIT`, `ROLLBACK` | Permissions / transactions (Phase 4.5) |

> This note covers **DDL** (defining tables/indexes/views) and **DML** (inserting/updating/deleting rows). Querying (`SELECT`) is the next notes. This builds directly on Phase 4.1 (tables, keys, constraints, data types).

---

## 1. DDL — Defining Structure

### 1.1 CREATE TABLE
Defines a table with columns, types, and constraints (recall Phase 4.1):
```sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,                 -- auto-increment PK
    email      VARCHAR(255) NOT NULL UNIQUE,          -- required & unique
    name       VARCHAR(100) NOT NULL,
    age        INTEGER CHECK (age >= 0),              -- CHECK constraint
    status     VARCHAR(20) NOT NULL DEFAULT 'ACTIVE', -- DEFAULT value
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    id      BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),     -- inline foreign key
    total   DECIMAL(10,2) NOT NULL CHECK (total >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- `IF NOT EXISTS` avoids an error if the table already exists: `CREATE TABLE IF NOT EXISTS ...`.
- Constraints can be **inline** (on the column) or **table-level** (named, at the end) — named constraints are easier to manage:
```sql
CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
```

### 1.2 ALTER TABLE
Modify an existing table's structure:
```sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);            -- add a column
ALTER TABLE users DROP COLUMN phone;                       -- remove a column
ALTER TABLE users ALTER COLUMN name TYPE VARCHAR(200);     -- change a type
ALTER TABLE users RENAME COLUMN name TO full_name;         -- rename
ALTER TABLE users ADD CONSTRAINT chk_age CHECK (age < 150);-- add a constraint
ALTER TABLE users DROP CONSTRAINT chk_age;                 -- drop a constraint
```
> ⚠️ `ALTER TABLE` on large tables can **lock the table** and be slow (rewriting data). In production, schema changes need care — that's exactly what **migrations (Phase 8 Flyway)** manage safely, and why **zero-downtime** changes must be backward-compatible.

### 1.3 DROP vs TRUNCATE vs DELETE (crucial distinction)
| Statement | What it does | Reversible? | Speed | Category |
|-----------|--------------|-------------|-------|----------|
| **`DROP TABLE`** | Deletes the **entire table** (structure + data) | No | Fast | DDL |
| **`TRUNCATE`** | Deletes **all rows**, keeps the table structure | No (usually) | **Very fast** | DDL |
| **`DELETE`** | Deletes **rows** (optionally with WHERE) | Yes (in a transaction) | Slower | DML |
```sql
DROP TABLE orders;              -- table gone entirely
TRUNCATE TABLE orders;          -- all rows gone, table remains (fast, resets sequences)
DELETE FROM orders;             -- all rows gone (slower, logged, transactional)
DELETE FROM orders WHERE id = 5;-- delete specific rows
```
> ⚠️ **Key difference:** `DELETE` is DML (transactional, can be rolled back, fires triggers, logs each row); `TRUNCATE`/`DROP` are DDL (fast, minimal logging, can't easily roll back). Use `DELETE ... WHERE` for selective removal; `TRUNCATE` to quickly empty a table.

### 1.4 CREATE INDEX
An **index** speeds up lookups (recall B-trees, Phase 2.2 / indexing, Phase 4.4):
```sql
CREATE INDEX idx_users_email ON users(email);             -- speeds WHERE email = ...
CREATE UNIQUE INDEX idx_users_email_uniq ON users(email); -- also enforces uniqueness
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);  -- composite
DROP INDEX idx_users_email;
```
> Indexes make reads faster but writes slightly slower (the index must be updated too). Full treatment in Phase 4.4 — for now: an index is a separate structure that lets the DB find rows without scanning the whole table.

### 1.5 CREATE VIEW
A **view** is a saved query that acts like a virtual table — it doesn't store data, it re-runs the query each time:
```sql
CREATE VIEW active_users AS
    SELECT id, name, email FROM users WHERE status = 'ACTIVE';

SELECT * FROM active_users;     -- query the view like a table
```
- Simplifies complex queries, encapsulates logic, can restrict column access (security).
- A **materialized view** (Phase 4.6) *does* store the result (refreshed on demand) — faster reads, but can be stale.

---

## 2. DML — Modifying Data

### 2.1 INSERT
Add new rows:
```sql
-- Single row:
INSERT INTO users (email, name, age)
VALUES ('alice@x.com', 'Alice', 30);

-- Multiple rows (one statement — more efficient):
INSERT INTO users (email, name, age) VALUES
    ('bob@x.com', 'Bob', 25),
    ('carol@x.com', 'Carol', 35);

-- Insert from a query (copy data):
INSERT INTO archived_users (id, name)
    SELECT id, name FROM users WHERE status = 'INACTIVE';

-- Return the generated id (PostgreSQL):
INSERT INTO users (email, name) VALUES ('dan@x.com', 'Dan')
    RETURNING id;
```
- List the columns explicitly (don't rely on column order).
- **Batch inserts** (multiple rows per statement) are far faster than many single-row inserts (fewer round-trips — recall Phase 0.2 network latency; relevant for JDBC batching, Phase 4.7).
- `RETURNING` (PostgreSQL) gives back generated values (like the auto-increment `id`) — useful instead of a separate query.

### 2.2 UPDATE
Modify existing rows. **Always use a `WHERE` clause** unless you really mean to update every row:
```sql
UPDATE users SET status = 'INACTIVE' WHERE id = 5;
UPDATE users SET age = age + 1, updated_at = now() WHERE id = 5;   -- multiple columns
UPDATE orders SET total = total * 1.1 WHERE created_at < '2026-01-01';
```
> ⚠️ **`UPDATE` without `WHERE` updates EVERY row.** This is a classic, catastrophic mistake. Always double-check your `WHERE`. (In production, run a `SELECT` with the same `WHERE` first to verify what you'll affect.)

### 2.3 DELETE
Remove rows. **Same `WHERE` warning as UPDATE:**
```sql
DELETE FROM orders WHERE status = 'CANCELLED';
DELETE FROM users WHERE id = 5;
DELETE FROM orders;             -- ⚠️ deletes ALL rows (no WHERE)
```

### 2.4 Soft delete (a common pattern)
Instead of physically deleting, mark rows as deleted with a flag/timestamp — preserving history and enabling "undo":
```sql
-- Soft delete: don't actually remove the row
UPDATE users SET deleted_at = now() WHERE id = 5;
-- Then queries filter out "deleted" rows:
SELECT * FROM users WHERE deleted_at IS NULL;
```
> **Soft deletes** (a `deleted_at` or `is_deleted` column) are common in real systems — they keep an audit trail and allow recovery. The trade-off: every query must filter them out, and the table grows. (Hibernate supports `@SQLDelete`/`@Where` for this — Phase 5.4.)

---

## 3. UPSERT (Insert or Update)

**UPSERT** = insert a row, but if it already exists (conflict on a unique key), update it instead. PostgreSQL uses `INSERT ... ON CONFLICT`:
```sql
INSERT INTO users (email, name, age)
VALUES ('alice@x.com', 'Alice', 31)
ON CONFLICT (email)                       -- if email already exists...
DO UPDATE SET name = EXCLUDED.name,        -- ...update instead
              age = EXCLUDED.age;

-- "Do nothing" variant (insert only if absent):
INSERT INTO users (email, name) VALUES ('alice@x.com', 'Alice')
ON CONFLICT (email) DO NOTHING;
```
- `EXCLUDED` refers to the row that *would have been* inserted.
- `ON CONFLICT DO NOTHING` = "insert if not present, otherwise skip."
> UPSERT is essential for **idempotent** operations (recall Phase 0.2 HTTP idempotency, Phase 7) — e.g., "create or update this record" without a race between a SELECT-then-INSERT. (MySQL uses `INSERT ... ON DUPLICATE KEY UPDATE`; the SQL standard has `MERGE`.)

---

## 4. Transactions (preview — full detail Phase 4.5)

DML statements can be grouped into a **transaction** — all succeed or all fail together (atomicity):
```sql
BEGIN;                                          -- start a transaction
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                                          -- make changes permanent
-- or ROLLBACK; to undo everything if something went wrong
```
> ⚠️ Without a transaction, if the second UPDATE fails after the first succeeds, money disappears! Transactions guarantee **all-or-nothing**. (Full ACID coverage in Phase 4.5; Spring's `@Transactional` in Phase 5.5.)

---

## 5. Audit Columns & Conventions (recap Phase 4.1/4.3 preview)

Real tables usually include **audit columns** and follow naming conventions:
```sql
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),    -- audit: when created
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),    -- audit: last modified
    deleted_at  TIMESTAMPTZ,                            -- soft delete (NULL = active)
    version     INTEGER NOT NULL DEFAULT 0              -- optimistic locking (Phase 4.5/5.4)
);
```
| Convention | Why |
|------------|-----|
| `created_at`/`updated_at` | Audit trail |
| `deleted_at` | Soft delete |
| `version` | Optimistic locking (Phase 4.5) |
| `snake_case` names, plural tables | Common SQL convention |
> Spring Data JPA auto-populates `created_at`/`updated_at` with `@CreatedDate`/`@LastModifiedDate` (Phase 5.4 auditing), and `version` with `@Version` (optimistic locking).

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `UPDATE`/`DELETE` without `WHERE` | Always include `WHERE`; SELECT first to verify |
| Confusing DELETE / TRUNCATE / DROP | DELETE=rows (transactional), TRUNCATE=all rows (fast), DROP=table |
| Many single-row INSERTs | Batch them (one statement / JDBC batching, Phase 4.7) |
| Not listing INSERT columns | Always list columns explicitly |
| `ALTER TABLE` on big tables in prod | Use migrations + backward-compatible changes (Phase 8) |
| Hard-deleting data you might need | Consider soft delete (`deleted_at`) |
| SELECT-then-INSERT race | Use UPSERT (`ON CONFLICT`) |
| Forgetting transactions for multi-step changes | Wrap in `BEGIN`/`COMMIT` |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Migrations (Phase 8 Flyway):** version-controlled `CREATE`/`ALTER` DDL — never run ad-hoc DDL in production.
- **JPA/Hibernate (Phase 5.4):** generates INSERT/UPDATE/DELETE under the hood; `save()` is an upsert-like operation.
- **JDBC (Phase 4.7):** you'll write DML with `PreparedStatement` + batching.
- **Audit columns + soft delete** map to Spring Data auditing (`@CreatedDate`) and `@SQLDelete`.
- **`version` column** → optimistic locking (`@Version`, Phase 4.5/5.4).
- **UPSERT** for idempotent operations (Phase 7) and avoiding race conditions.
- **Transactions** (`@Transactional`, Phase 5.5) wrap DML for atomicity.

---

## 8. Quick Self-Check Questions

1. What do DDL and DML stand for, and which statements belong to each?
2. How do you create a table with constraints? What's `IF NOT EXISTS`?
3. What's the difference between DROP, TRUNCATE, and DELETE?
4. Why are batch INSERTs better than many single-row inserts?
5. What's the danger of UPDATE/DELETE without WHERE, and how do you guard against it?
6. What is a soft delete, and why use it?
7. What is UPSERT, and why is it useful for idempotency?
8. What are audit columns, and how does Spring populate them?

---

## 9. Key Terms Glossary

- **SQL:** the relational query/manipulation language.
- **DDL / DML / DQL / DCL/TCL:** definition / manipulation / query / control statements.
- **CREATE / ALTER / DROP / TRUNCATE:** define/modify/delete table; empty table.
- **INSERT / UPDATE / DELETE:** add / modify / remove rows.
- **UPSERT (`ON CONFLICT`):** insert-or-update.
- **`RETURNING`:** return generated values from a DML statement (PostgreSQL).
- **Index / view:** lookup-speeding structure / saved virtual query.
- **Soft delete:** marking rows deleted instead of removing them.
- **Audit columns:** `created_at`/`updated_at`/`deleted_at`/`version`.
- **Transaction (BEGIN/COMMIT/ROLLBACK):** all-or-nothing group of changes.

---

*This is the first note of **Section 4.2 — SQL Mastery**.*
*Next topic: **DQL Basics & Aggregation (SELECT, WHERE, GROUP BY)**.*
