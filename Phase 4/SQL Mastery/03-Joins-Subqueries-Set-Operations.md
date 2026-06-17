# SQL: Joins, Subqueries & Set Operations

> **Phase 4 — Databases → 4.2 SQL Mastery**
> Goal: Master combining data across tables — all the JOIN types, subqueries (scalar, correlated, EXISTS), and set operations (UNION, INTERSECT, EXCEPT).

---

## 0. The Big Picture

Relational data is **split across tables** (recall Phase 4.1 — users, orders). **JOINs** recombine related rows; **subqueries** nest one query inside another; **set operations** combine query results vertically. These are how you answer real questions that span multiple tables.

```
users + orders -> JOIN -> "each order WITH its user's name"
```

> Joins are *the* defining feature of relational databases — and the most important SQL skill for real queries. Getting join types right (especially the INNER vs LEFT distinction) is essential.

---

## 1. Why Joins? (Recombining Normalized Data)

Data is normalized (split into tables, Phase 4.3) to avoid duplication. A **JOIN** matches rows from two tables based on a related column (usually a foreign key — Phase 4.1):
```sql
-- "Show each order with the buyer's name" — needs data from BOTH tables:
SELECT o.id, o.total, u.name
FROM orders o
JOIN users u ON o.user_id = u.id;    -- match orders.user_id to users.id
```
> Table **aliases** (`orders o`, `users u`) are essential in joins — they keep column references unambiguous (`o.id` vs `u.id`).

---

## 2. The JOIN Types

```
       users          orders
      (left)          (right)
   matching rows  +  matching rows   -> the join combines them
```
| Join | Returns |
|------|---------|
| **INNER JOIN** | Only rows with a **match in both** tables |
| **LEFT (OUTER) JOIN** | **All left** rows + matching right (NULLs where no match) |
| **RIGHT (OUTER) JOIN** | **All right** rows + matching left (NULLs where no match) |
| **FULL (OUTER) JOIN** | **All rows from both**, matched where possible (NULLs elsewhere) |
| **CROSS JOIN** | Every left row × every right row (Cartesian product) |
| **SELF JOIN** | A table joined to itself |

### 2.1 INNER JOIN (the most common)
Returns only rows that have a match in **both** tables:
```sql
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;   -- only users WHO HAVE orders
-- (INNER is the default; "JOIN" alone means INNER JOIN)
```
> Users with no orders are **excluded**; orders with no user can't exist (FK). Use INNER when you only want matched rows.

### 2.2 LEFT JOIN (keep all left rows)
Returns **all** rows from the left table, with matching right-table data (or NULLs if no match):
```sql
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;    -- ALL users, even those with no orders
-- users with no orders -> o.total is NULL
```
**Find rows with NO match** (a key LEFT JOIN pattern):
```sql
-- Users who have NEVER placed an order:
SELECT u.name
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;                         -- no matching order -> NULL
```
> ⚠️ **INNER vs LEFT is the #1 join decision.** INNER drops unmatched rows; LEFT keeps all left rows. "Show all users and their order count (including zero)" needs a LEFT JOIN — an INNER JOIN would silently drop users with no orders. The `WHERE right.id IS NULL` trick finds "left rows with no match."

### 2.3 RIGHT JOIN
The mirror of LEFT (all right rows + matching left). Rarely used — you can always rewrite a RIGHT JOIN as a LEFT JOIN by swapping table order (which is clearer).

### 2.4 FULL OUTER JOIN
All rows from both tables, matched where possible:
```sql
SELECT u.name, o.total
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;   -- everything from both sides
```

### 2.5 CROSS JOIN (Cartesian product)
Every combination of left × right rows — use with care (result size = rows_left × rows_right):
```sql
SELECT s.size, c.color FROM sizes s CROSS JOIN colors c;   -- all size/color combos
```
> ⚠️ A CROSS JOIN (or an accidental join with **no ON condition**) produces a Cartesian explosion — millions of rows from two modest tables. A common accidental-bug source.

### 2.6 SELF JOIN (table joined to itself)
Join a table to itself — e.g., employees and their managers (both in the same `employees` table):
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
JOIN employees m ON e.manager_id = m.id;    -- self join via aliases
```

### 2.7 Joining multiple tables
Chain joins to combine 3+ tables:
```sql
SELECT u.name, o.id AS order_id, p.name AS product
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id;
```

### 2.8 USING and NATURAL JOIN (shortcuts)
```sql
SELECT * FROM orders o JOIN users u USING (user_id);  -- when the column name is identical
-- NATURAL JOIN auto-joins on same-named columns -- AVOID (implicit/fragile)
```
> Prefer explicit `ON` conditions over `NATURAL JOIN` (which silently joins on all same-named columns — error-prone).

---

## 3. Subqueries (Nested Queries)

A **subquery** is a query nested inside another — used in `WHERE`, `FROM`, `SELECT`, or `HAVING`.

### 3.1 Scalar subquery (returns one value)
```sql
-- Orders above the average order total:
SELECT * FROM orders
WHERE total > (SELECT AVG(total) FROM orders);   -- subquery returns one value
```

### 3.2 Subquery in WHERE with IN (returns a list)
```sql
-- Users who have placed an order:
SELECT * FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders);
```

### 3.3 Subquery in FROM (derived table)
```sql
-- Use a subquery as a table:
SELECT user_id, order_count
FROM (SELECT user_id, COUNT(*) AS order_count FROM orders GROUP BY user_id) AS stats
WHERE order_count > 5;
```

### 3.4 Correlated subquery (references the outer query)
A **correlated subquery** runs once per outer row, referencing a column from the outer query:
```sql
-- Users whose latest order exceeds 100:
SELECT u.name FROM users u
WHERE (SELECT MAX(o.total) FROM orders o WHERE o.user_id = u.id) > 100;
--                                              ^^^^^^^^^ references the OUTER u.id
```
> ⚠️ **Correlated subqueries can be slow** — they re-execute for every outer row (potentially O(n²) — recall Phase 2.1). Often a JOIN or window function (next note) is faster. Use them when clearer, but watch performance.

### 3.5 EXISTS / NOT EXISTS (efficient existence checks)
`EXISTS` checks whether a subquery returns **any** rows — it short-circuits (stops at the first match), so it's often faster than `IN` for existence checks:
```sql
-- Users who have at least one order:
SELECT u.name FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- Users with NO orders:
SELECT u.name FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```
> `SELECT 1` is convention (the actual columns don't matter — only existence does). **`EXISTS` vs `IN`:** `EXISTS` short-circuits and handles NULLs cleanly; `NOT IN` with NULLs in the subquery is a classic bug (returns no rows). Prefer `NOT EXISTS` over `NOT IN` when NULLs are possible.

---

## 4. Set Operations (Combining Results Vertically)

While joins combine tables **horizontally** (side by side), **set operations** stack the results of two queries **vertically**. Both queries must have **compatible columns** (same number and types).
| Operation | Returns |
|-----------|---------|
| **`UNION`** | All rows from both queries, **duplicates removed** |
| **`UNION ALL`** | All rows from both, **duplicates kept** (faster) |
| **`INTERSECT`** | Only rows in **both** queries |
| **`EXCEPT`** (MINUS in Oracle) | Rows in the first query **but not** the second |
```sql
-- Combine two sources (dedup):
SELECT email FROM users UNION SELECT email FROM subscribers;

-- Keep duplicates (faster — no dedup step):
SELECT email FROM users UNION ALL SELECT email FROM subscribers;

-- Emails in both lists:
SELECT email FROM users INTERSECT SELECT email FROM subscribers;

-- Users who are NOT subscribers:
SELECT email FROM users EXCEPT SELECT email FROM subscribers;
```
> ⚠️ **`UNION` removes duplicates (extra sort/dedup work); `UNION ALL` doesn't.** If you know there are no duplicates (or don't care), use **`UNION ALL`** — it's faster. (Recall set algebra, Phase 1.6 — these are SQL's set operations.)

---

## 5. Join vs Subquery vs Set Operation (when to use which)
| Need | Use |
|------|-----|
| Combine columns from related tables | **JOIN** |
| Filter by a computed/aggregated value | **Subquery** (scalar/IN) |
| Existence check ("has any related row") | **EXISTS** |
| Stack results from two queries | **UNION / UNION ALL** |
| Find rows in one set but not another | **EXCEPT** / `LEFT JOIN ... IS NULL` / `NOT EXISTS` |
> Many problems can be solved multiple ways — often a **JOIN is faster than a correlated subquery**, and the query planner (Phase 4.4) may rewrite them. Write the clearest version, then optimize with EXPLAIN if needed.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using INNER JOIN when you need all left rows | Use LEFT JOIN |
| Forgetting the `ON` condition | Accidental CROSS JOIN (Cartesian explosion) |
| `NOT IN` with NULLs in the subquery | Use `NOT EXISTS` |
| Slow correlated subqueries | Rewrite as a JOIN or window function |
| `UNION` when duplicates are fine | Use `UNION ALL` (faster) |
| Ambiguous column names in joins | Use table aliases (`u.id`, `o.id`) |
| `NATURAL JOIN` surprises | Use explicit `ON` |
| Joining without an index on the join column | Slow joins (Phase 4.4 — index FKs) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **JPA relationships (Phase 5.4):** `@OneToMany`/`@ManyToOne` generate JOINs; `JOIN FETCH`/`@EntityGraph` control them to avoid the **N+1 problem** (a join issue).
- **The N+1 problem** is literally "running a query per row instead of one JOIN" — understanding joins is essential to spot/fix it.
- **Reports & dashboards** (Projects 4, 5) join multiple tables and aggregate.
- **`LEFT JOIN ... IS NULL`** and **`NOT EXISTS`** for "find missing" queries.
- **Index FK columns** (Phase 4.4) so joins are fast.
- **`EXPLAIN`** (Phase 4.4) reveals join strategies (nested loop, hash join, merge join).
- **Spring Data `@Query`** lets you write JPQL/native joins when method derivation isn't enough.

---

## 8. Quick Self-Check Questions

1. Why do relational databases need joins?
2. What's the difference between INNER and LEFT JOIN? When use each?
3. How do you find users who have *no* orders (two ways)?
4. What causes an accidental Cartesian product?
5. What's a correlated subquery, and why can it be slow?
6. When is `EXISTS` better than `IN`, and why prefer `NOT EXISTS` over `NOT IN`?
7. What's the difference between `UNION` and `UNION ALL`?
8. What does `INTERSECT` and `EXCEPT` do?

---

## 9. Key Terms Glossary

- **JOIN:** combine related rows from multiple tables.
- **INNER JOIN:** only matched rows in both.
- **LEFT/RIGHT/FULL OUTER JOIN:** keep all rows from left/right/both (NULLs for non-matches).
- **CROSS JOIN:** Cartesian product (all combinations).
- **SELF JOIN:** a table joined to itself.
- **Table alias:** short name for a table in a query.
- **Subquery:** a nested query (scalar / IN / derived table / correlated).
- **Correlated subquery:** references the outer query (runs per row).
- **EXISTS / NOT EXISTS:** efficient existence checks.
- **Set operations (UNION/UNION ALL/INTERSECT/EXCEPT):** combine query results vertically.

---

*Previous topic: **DQL Basics & Aggregation**.*
*Next topic: **Window Functions & Common Table Expressions (CTEs)**.*
