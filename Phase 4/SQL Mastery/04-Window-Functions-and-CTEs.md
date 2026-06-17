# SQL: Window Functions & CTEs

> **Phase 4 — Databases → 4.2 SQL Mastery**
> Goal: Master two advanced, high-value SQL features — window functions (ranking, running totals, lag/lead) and Common Table Expressions (CTEs, including recursive).

---

## 0. The Big Picture

These are the SQL features that separate intermediate from advanced:
- **Window functions** perform calculations **across a set of rows related to the current row** — *without collapsing them into groups* (unlike GROUP BY). Great for rankings, running totals, comparisons to other rows.
- **CTEs (WITH clause)** name a subquery so you can reference it like a temporary table — making complex queries readable, and enabling **recursion**.

```
GROUP BY:        collapses rows into one per group (aggregate)
Window function: keeps every row, but adds a calculation across related rows
```

> Window functions are essential for **reports, analytics, and dashboards** (Projects 4, 5). They're heavily tested and incredibly powerful — learn them well.

---

## 1. Window Functions — The Core Idea

A **window function** computes a value over a "window" of rows **related to the current row**, but — crucially — **keeps all the individual rows** (unlike GROUP BY which collapses them).
```sql
-- GROUP BY collapses: one row per department
SELECT department, AVG(salary) FROM employees GROUP BY department;

-- Window function KEEPS every row, adds the dept average alongside each:
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg   -- the "window"
FROM employees;
-- Result: every employee row, PLUS their department's average -> can compare each to it
```
> The magic: `OVER (...)` defines a **window** of related rows. The function (AVG, RANK, etc.) is computed over that window, but each row is preserved. This lets you compare each row to its group, rank within a group, compute running totals — things GROUP BY can't do.

---

## 2. The OVER Clause (Defining the Window)

`OVER (...)` defines which rows the function sees, via three optional parts:
```sql
function() OVER (
    PARTITION BY column      -- split rows into groups (like GROUP BY, but keeps rows)
    ORDER BY column          -- order within each partition (needed for ranking/running totals)
    ROWS/RANGE frame         -- limit the window to a subset of the partition (frame clause)
)
```
| Clause | Purpose |
|--------|---------|
| **`PARTITION BY`** | Divide rows into groups; the function resets per partition |
| **`ORDER BY`** | Order rows within the partition (defines "previous"/"running") |
| **Frame** (`ROWS BETWEEN ...`) | Limit to a sliding range of rows (e.g., current + preceding) |
- No `PARTITION BY` → the whole result is one window.
- `ORDER BY` inside `OVER` is different from the query's final `ORDER BY` — it orders rows *within the window* for the calculation.

---

## 3. Ranking Functions

Assign a rank/number to each row within its partition:
| Function | Behavior on ties |
|----------|------------------|
| **`ROW_NUMBER()`** | Unique sequential number (1,2,3,4 — ties get different numbers) |
| **`RANK()`** | Same rank for ties, then **gaps** (1,2,2,4) |
| **`DENSE_RANK()`** | Same rank for ties, **no gaps** (1,2,2,3) |
| **`NTILE(n)`** | Divide rows into n roughly-equal buckets (quartiles, etc.) |
```sql
SELECT name, department, salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank
FROM employees;
```
### 3.1 The classic use: "top N per group"
```sql
-- Top 3 highest-paid employees PER department:
SELECT * FROM (
    SELECT name, department, salary,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
) ranked
WHERE rn <= 3;
```
> ⚠️ **"Top N per group" is a window-function classic** — you can't easily do it with GROUP BY. `ROW_NUMBER()` + a partition + filtering on the rank is the standard pattern. Know the ROW_NUMBER vs RANK vs DENSE_RANK distinction (ties).

---

## 4. Aggregate Functions as Window Functions (Running Totals)

Any aggregate (`SUM`, `AVG`, `COUNT`, etc.) can be a window function — enabling **running totals** and moving averages:
```sql
-- Running total of order amounts over time:
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date) AS running_total   -- cumulative sum
FROM orders;

-- Each row's share of its department's total salary:
SELECT name, salary,
       salary * 100.0 / SUM(salary) OVER (PARTITION BY department) AS pct_of_dept
FROM employees;

-- 7-day moving average (frame clause):
SELECT order_date, amount,
       AVG(amount) OVER (ORDER BY order_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg
FROM orders;
```
> When an aggregate has `ORDER BY` in its `OVER`, it becomes **cumulative** (running) by default. The **frame clause** (`ROWS BETWEEN ...`) controls exactly which rows are included — enabling moving averages/sums (recall sliding window, Phase 2.3 — same concept in SQL!).

---

## 5. LAG, LEAD, FIRST_VALUE, LAST_VALUE (Comparing to Other Rows)

These access values from **other rows** relative to the current one — without a self-join:
| Function | Returns |
|----------|---------|
| **`LAG(col, n)`** | The value n rows **before** the current (default n=1) |
| **`LEAD(col, n)`** | The value n rows **after** the current |
| **`FIRST_VALUE(col)`** | The first value in the window |
| **`LAST_VALUE(col)`** | The last value in the window |
```sql
-- Compare each month's sales to the previous month:
SELECT month, sales,
       LAG(sales) OVER (ORDER BY month) AS prev_month,
       sales - LAG(sales) OVER (ORDER BY month) AS change      -- month-over-month delta
FROM monthly_sales;

-- Days between consecutive orders per user:
SELECT user_id, order_date,
       order_date - LAG(order_date) OVER (PARTITION BY user_id ORDER BY order_date) AS days_since_last
FROM orders;
```
> `LAG`/`LEAD` are perfect for **time-series comparisons** (month-over-month growth, gaps between events) — replacing awkward self-joins (recall Phase 4.2.3) with clean, efficient window functions.

---

## 6. Common Table Expressions (CTEs)

A **CTE** (`WITH` clause) defines a **named temporary result set** you can reference within the query — making complex queries readable by breaking them into named steps.
```sql
WITH high_value_orders AS (              -- define a named subquery
    SELECT user_id, SUM(total) AS spent
    FROM orders
    GROUP BY user_id
    HAVING SUM(total) > 1000
)
SELECT u.name, h.spent                    -- reference it like a table
FROM users u
JOIN high_value_orders h ON u.id = h.user_id;
```

### 6.1 Multiple CTEs (chained steps)
```sql
WITH
  active AS (SELECT * FROM users WHERE status = 'ACTIVE'),
  order_counts AS (SELECT user_id, COUNT(*) AS cnt FROM orders GROUP BY user_id)
SELECT a.name, COALESCE(oc.cnt, 0) AS orders
FROM active a
LEFT JOIN order_counts oc ON a.id = oc.user_id;
```
> ⚠️ CTEs vs subqueries: a CTE is often **more readable** than a deeply nested subquery (FROM-subquery, Phase 4.2.3) — you read top-to-bottom in named steps. Functionally similar; choose CTEs for clarity in complex queries. (Performance is usually comparable; modern planners inline them.)

---

## 7. Recursive CTEs (Hierarchies & Graphs)

A **recursive CTE** references itself — perfect for **hierarchical/tree data** (org charts, category trees, bill-of-materials) and graph traversal (recall trees/graphs, Phase 2.2!):
```sql
WITH RECURSIVE subordinates AS (
    -- Anchor (base case): start with the top manager
    SELECT id, name, manager_id, 1 AS level
    FROM employees WHERE id = 1

    UNION ALL

    -- Recursive part: find employees reporting to the previous level
    SELECT e.id, e.name, e.manager_id, s.level + 1
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id    -- joins to the CTE itself
)
SELECT * FROM subordinates;                        -- the whole org tree under employee 1
```
- **Anchor member:** the starting rows (base case — recall recursion, Phase 2.3).
- **Recursive member:** references the CTE, adding the next level, until no new rows.
- `UNION ALL` combines each level.
> Recursive CTEs traverse trees/graphs **inside the database** — org hierarchies, category trees, "all descendants/ancestors," shortest paths. This is BFS/DFS (Phase 2.2) expressed in SQL. ⚠️ Always ensure termination (the recursion must eventually produce no new rows) or guard with a depth limit.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using GROUP BY for "top N per group" | Use `ROW_NUMBER()` window function |
| Confusing ROW_NUMBER / RANK / DENSE_RANK | Know their tie behavior |
| Forgetting `ORDER BY` in `OVER` for running totals | Required for cumulative behavior |
| `LAST_VALUE` returning the current row | Default frame is "up to current row" — set the frame |
| Deeply nested subqueries (unreadable) | Use CTEs for clarity |
| Recursive CTE without a termination | Ensure it stops (or add a depth limit) |
| Using a self-join for prev/next row | Use `LAG`/`LEAD` |
| Expecting window functions in WHERE | They run after WHERE — wrap in a subquery/CTE to filter on them |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Reports & dashboards** (Projects 4, 5): rankings (leaderboards), running totals (revenue over time), month-over-month growth — all window functions.
- **Sales/analytics queries** (Project 5 admin dashboard "using SQL window functions") — explicitly in the roadmap.
- **Pagination on ranked results** (top-N per group) for feeds/listings.
- **Spring Data `@Query`** (native SQL) when you need window functions/CTEs (JPQL doesn't fully support them).
- **Hierarchical data** (categories, org charts, comment threads) via recursive CTEs — often more efficient than loading the tree into Java and traversing it.
- **Time-series metrics** (Phase 9) — moving averages, deltas via window functions.

---

## 10. Quick Self-Check Questions

1. How does a window function differ from GROUP BY?
2. What do the three parts of the `OVER` clause do?
3. What's the difference between ROW_NUMBER, RANK, and DENSE_RANK?
4. How do you get the "top N per group"?
5. How do you compute a running total? A moving average?
6. What do LAG and LEAD do, and what do they replace?
7. What is a CTE, and why use one over a nested subquery?
8. What is a recursive CTE used for, and what are its two parts?

---

## 11. Key Terms Glossary

- **Window function:** computes over related rows without collapsing them.
- **`OVER`:** defines the window (PARTITION BY / ORDER BY / frame).
- **`PARTITION BY`:** groups rows for the window (rows preserved).
- **Frame clause (`ROWS BETWEEN`):** limits the window to a sliding range.
- **ROW_NUMBER / RANK / DENSE_RANK / NTILE:** ranking functions.
- **Running total / moving average:** cumulative/windowed aggregates.
- **LAG / LEAD:** value from a previous/next row.
- **FIRST_VALUE / LAST_VALUE:** edge values of the window.
- **CTE (`WITH`):** a named, reusable subquery.
- **Recursive CTE:** a self-referencing CTE for hierarchies/graphs.
- **Anchor / recursive member:** base case / recursion of a recursive CTE.

---

*Previous topic: **Joins, Subqueries & Set Operations**.*
*Next topic: **String, Date/Time & JSON Functions**.*
