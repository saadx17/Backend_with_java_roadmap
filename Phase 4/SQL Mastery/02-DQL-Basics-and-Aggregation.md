# SQL: DQL Basics & Aggregation

> **Phase 4 — Databases → 4.2 SQL Mastery**
> Goal: Master querying data with SELECT — WHERE, ORDER BY, LIMIT/OFFSET, DISTINCT, expressions (CASE, COALESCE), and aggregate functions with GROUP BY/HAVING.

---

## 0. The Big Picture

**DQL (Data Query Language)** is the `SELECT` statement — reading data. It's the most-used SQL and the deepest. This note covers single-table querying: filtering, sorting, limiting, expressions, and **aggregation** (summarizing groups of rows).

```sql
SELECT columns FROM table WHERE condition GROUP BY ... HAVING ... ORDER BY ... LIMIT ...;
```

> `SELECT` is where you spend most of your SQL time. Understanding its **logical execution order** (§7) is the key to writing and debugging queries correctly.

---

## 1. Basic SELECT

```sql
SELECT * FROM users;                      -- all columns, all rows (avoid * in production)
SELECT id, name, email FROM users;        -- specific columns (preferred)
SELECT name AS full_name FROM users;      -- alias a column
SELECT email FROM users LIMIT 10;
```
> ⚠️ **Avoid `SELECT *` in application code:** it fetches unneeded columns (slower, more network — Phase 0.2), breaks if columns change, and prevents covering-index optimizations (Phase 4.4). **List the columns you need.**

---

## 2. WHERE (Filtering Rows)

`WHERE` filters which rows are returned, using conditions:
```sql
SELECT * FROM users WHERE age >= 18;
SELECT * FROM users WHERE status = 'ACTIVE' AND age > 25;     -- AND
SELECT * FROM users WHERE status = 'ACTIVE' OR status = 'PENDING';  -- OR
SELECT * FROM orders WHERE total BETWEEN 50 AND 100;          -- range (inclusive)
SELECT * FROM users WHERE status IN ('ACTIVE', 'PENDING');    -- set membership
SELECT * FROM users WHERE name LIKE 'A%';                     -- pattern (starts with A)
SELECT * FROM users WHERE email LIKE '%@gmail.com';           -- ends with
SELECT * FROM users WHERE deleted_at IS NULL;                 -- NULL check (recall 4.1!)
SELECT * FROM users WHERE age NOT BETWEEN 13 AND 19;          -- negation
```
| Operator | Meaning |
|----------|---------|
| `= != <> < > <= >=` | Comparison (`<>` = not equal) |
| `AND` / `OR` / `NOT` | Logical combination |
| `BETWEEN a AND b` | Range (inclusive) |
| `IN (...)` | Match any in a list |
| `LIKE` / `ILIKE` | Pattern match (`%`=any chars, `_`=one char); ILIKE = case-insensitive (Postgres) |
| `IS NULL` / `IS NOT NULL` | **NULL check (never `= NULL`!)** |

### 2.1 The NULL trap (recall Phase 4.1)
> ⚠️ **Use `IS NULL`/`IS NOT NULL`, never `= NULL` or `!= NULL`.** Comparisons with NULL yield "unknown" (three-valued logic), so `WHERE col = NULL` returns **nothing**. Also: `WHERE status != 'ACTIVE'` **excludes NULL rows** (since `NULL != 'ACTIVE'` is unknown, not true) — a common bug. Use `WHERE status != 'ACTIVE' OR status IS NULL` if you want them.

---

## 3. ORDER BY (Sorting)

```sql
SELECT * FROM users ORDER BY name;                    -- ascending (default)
SELECT * FROM users ORDER BY age DESC;                -- descending
SELECT * FROM users ORDER BY status ASC, age DESC;    -- multiple keys (tie-breaker)
SELECT * FROM users ORDER BY created_at DESC NULLS LAST;  -- control NULL placement
```
- Sort by one or more columns; `ASC` (default) or `DESC`.
- Multiple columns = primary sort, then tie-breakers (recall Comparator chaining, Phase 1.6).
- `NULLS FIRST`/`NULLS LAST` controls where NULLs go (Postgres).
> Sorting on an **indexed** column is fast (the index is already ordered — Phase 4.4); sorting unindexed columns may require an expensive in-memory/disk sort.

---

## 4. LIMIT & OFFSET (Pagination)

```sql
SELECT * FROM users ORDER BY id LIMIT 10;             -- first 10 rows
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 20;   -- rows 21-30 (page 3, size 10)
```
- `LIMIT n` = return at most n rows.
- `OFFSET m` = skip the first m rows.
- **Pagination:** page `p` (1-based), size `s` → `LIMIT s OFFSET (p-1)*s`.
> ⚠️ **OFFSET pagination is slow for deep pages** — the DB must scan and discard all skipped rows (OFFSET 1,000,000 scans a million rows). For large datasets, use **keyset/cursor pagination** (`WHERE id > last_seen_id ORDER BY id LIMIT n`) — O(log n) via the index instead of O(offset). (Relevant for Spring Data `Pageable` — Phase 5.4.) Always pair LIMIT with `ORDER BY` — without it, row order is undefined.

---

## 5. DISTINCT (Removing Duplicates)

```sql
SELECT DISTINCT status FROM users;                    -- unique status values
SELECT DISTINCT status, country FROM users;           -- unique combinations
SELECT DISTINCT ON (user_id) * FROM orders            -- Postgres: first row per user_id
    ORDER BY user_id, created_at DESC;
```
- `DISTINCT` removes duplicate rows from the result.
- `DISTINCT ON (col)` (PostgreSQL) keeps the first row per group of `col` (often used for "latest per group").

---

## 6. Expressions: Aliases, CASE, COALESCE, CAST

### 6.1 Aliases
```sql
SELECT name AS user_name, age * 12 AS age_in_months FROM users;
SELECT u.name FROM users u;                            -- table alias (essential for joins, next note)
```

### 6.2 CASE WHEN (conditional logic)
SQL's if/else — produce different values based on conditions:
```sql
SELECT name,
    CASE
        WHEN age < 18 THEN 'Minor'
        WHEN age < 65 THEN 'Adult'
        ELSE 'Senior'
    END AS age_group
FROM users;
```

### 6.3 COALESCE & NULLIF (handling NULLs)
```sql
SELECT COALESCE(nickname, name, 'Anonymous') FROM users;  -- first non-NULL value
SELECT NULLIF(divisor, 0) FROM data;                       -- NULL if divisor=0 (avoid div-by-zero)
SELECT COALESCE(SUM(total), 0) FROM orders;                -- 0 instead of NULL when no rows
```
> `COALESCE` returns the first non-NULL argument — great for default values and avoiding NULL in results (recall Phase 4.1 NULL handling). `NULLIF(a, b)` returns NULL if a=b (useful to prevent division by zero).

### 6.4 CAST (type conversion)
```sql
SELECT CAST(price AS INTEGER) FROM products;          -- standard SQL
SELECT price::INTEGER FROM products;                   -- PostgreSQL shorthand
SELECT CAST('2026-01-01' AS DATE);
```

---

## 7. Aggregate Functions

**Aggregate functions** compute a single value over a **set of rows** — count, sum, average, etc.:
| Function | Returns |
|----------|---------|
| `COUNT(*)` | Number of rows |
| `COUNT(col)` | Number of **non-NULL** values in col |
| `COUNT(DISTINCT col)` | Number of distinct non-NULL values |
| `SUM(col)` | Total |
| `AVG(col)` | Average |
| `MIN(col)` / `MAX(col)` | Smallest / largest |
```sql
SELECT COUNT(*) FROM users;                           -- total users
SELECT COUNT(DISTINCT country) FROM users;            -- number of distinct countries
SELECT AVG(total), SUM(total), MAX(total) FROM orders;
SELECT COUNT(*) FROM users WHERE status = 'ACTIVE';   -- count with a filter
```
> ⚠️ `COUNT(*)` counts all rows; `COUNT(col)` **ignores NULLs** in that column — a subtle but important difference. Aggregates also ignore NULLs (e.g., `AVG` averages only non-NULL values).

---

## 8. GROUP BY (Aggregating Per Group)

`GROUP BY` splits rows into **groups** and applies aggregates **per group** — like SQL's version of `Collectors.groupingBy` (recall Phase 1.8!):
```sql
-- Count orders per user:
SELECT user_id, COUNT(*) AS order_count
FROM orders
GROUP BY user_id;

-- Total and average order value per status:
SELECT status, COUNT(*) AS cnt, SUM(total) AS revenue, AVG(total) AS avg_order
FROM orders
GROUP BY status;

-- Group by multiple columns:
SELECT country, status, COUNT(*)
FROM users
GROUP BY country, status;
```
### 8.1 The GROUP BY rule
> ⚠️ **Every column in the SELECT must either be in the GROUP BY or inside an aggregate function.** You can't select a non-grouped, non-aggregated column (which row's value would it pick?). E.g., `SELECT user_id, name, COUNT(*) ... GROUP BY user_id` is an error unless `name` is also grouped or aggregated.

---

## 9. HAVING (Filtering Groups)

`WHERE` filters **rows**; `HAVING` filters **groups** (after aggregation). Use `HAVING` to filter on aggregate results:
```sql
-- Users with more than 5 orders:
SELECT user_id, COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;                       -- filter GROUPS by the aggregate

-- High-revenue statuses:
SELECT status, SUM(total) AS revenue
FROM orders
WHERE created_at >= '2026-01-01'          -- WHERE filters rows BEFORE grouping
GROUP BY status
HAVING SUM(total) > 10000;                -- HAVING filters groups AFTER aggregation
```
| | `WHERE` | `HAVING` |
|---|---------|----------|
| Filters | Individual **rows** | **Groups** |
| Timing | **Before** GROUP BY | **After** GROUP BY |
| Can use aggregates? | ❌ No | ✅ Yes |
> Use `WHERE` to filter rows *before* grouping (more efficient — fewer rows to group); use `HAVING` only for conditions on aggregates. A common mistake is putting an aggregate condition in `WHERE` (which fails).

---

## 10. The Logical Execution Order of SELECT (Crucial!)

SQL is written in one order but **executed** in another. Understanding this explains many behaviors (e.g., why you can't use a SELECT alias in WHERE):
```
Written order:  SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT
Logical order:
  1. FROM        (pick the table/source)
  2. WHERE       (filter rows)
  3. GROUP BY    (form groups)
  4. HAVING      (filter groups)
  5. SELECT      (choose columns, compute expressions/aliases)
  6. DISTINCT
  7. ORDER BY    (sort)
  8. LIMIT/OFFSET (paginate)
```
> ⚠️ This order explains key gotchas: **you can't use a SELECT alias in WHERE** (WHERE runs before SELECT), but you **can** use it in ORDER BY (which runs after SELECT). And WHERE can't use aggregates (they're computed at GROUP BY/HAVING). Memorizing this order makes SQL behavior predictable.

---

## 11. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `SELECT *` in app code | List needed columns |
| `WHERE col = NULL` | Use `IS NULL` |
| `!=` excluding NULL rows unexpectedly | Add `OR col IS NULL` |
| LIMIT without ORDER BY | Always sort for deterministic results |
| Deep OFFSET pagination (slow) | Use keyset/cursor pagination |
| Aggregate condition in WHERE | Use HAVING |
| Non-grouped column in SELECT with GROUP BY | Group it or aggregate it |
| Using a SELECT alias in WHERE | Not allowed (execution order); repeat the expression |
| Forgetting `COUNT(col)` ignores NULLs | Use `COUNT(*)` for all rows |

---

## 12. Connection to Backend / Spring (Why This Matters Later)

- **Spring Data JPA** generates SELECTs from method names/`@Query` (Phase 5.4); understanding SQL helps you write efficient queries and read generated SQL.
- **Pagination** (`Pageable`, `Page<T>` — Phase 5.4) maps to LIMIT/OFFSET; know the deep-offset problem.
- **Reports/dashboards** (Projects 4, 5) use GROUP BY/aggregates — like `Collectors.groupingBy` (Phase 1.8) but in the DB (far more efficient than loading all rows into Java).
- **N+1 problem** (Phase 5.4): inefficient querying — knowing SQL helps you spot and fix it.
- **`SELECT *` avoidance** + projections (Phase 5.4) reduce data transfer.
- **EXPLAIN** (Phase 4.4) shows how these queries actually execute.

---

## 13. Quick Self-Check Questions

1. Why avoid `SELECT *` in application code?
2. How do you filter rows? Name the key WHERE operators.
3. Why must you use `IS NULL` instead of `= NULL`, and how can `!=` surprise you with NULLs?
4. How do you paginate, and why is deep OFFSET slow? What's the alternative?
5. What's the difference between `COUNT(*)` and `COUNT(col)`?
6. What does GROUP BY do, and what's the rule about SELECT columns with it?
7. What's the difference between WHERE and HAVING?
8. State the logical execution order of a SELECT and why it matters.

---

## 14. Key Terms Glossary

- **DQL / SELECT:** querying/reading data.
- **WHERE:** row filter (conditions, operators).
- **`IS NULL`:** the only correct NULL test.
- **ORDER BY:** sorting (ASC/DESC, multiple keys).
- **LIMIT / OFFSET:** result size / skip (pagination).
- **Keyset pagination:** WHERE-based pagination (fast for deep pages).
- **DISTINCT:** remove duplicate rows.
- **CASE / COALESCE / NULLIF / CAST:** conditional / first-non-null / null-if-equal / convert.
- **Aggregate function:** COUNT/SUM/AVG/MIN/MAX over rows.
- **GROUP BY:** aggregate per group.
- **HAVING:** filter groups (after aggregation).
- **Logical execution order:** FROM→WHERE→GROUP BY→HAVING→SELECT→ORDER BY→LIMIT.

---

*Previous topic: **DDL & DML**.*
*Next topic: **Joins, Subqueries & Set Operations**.*
