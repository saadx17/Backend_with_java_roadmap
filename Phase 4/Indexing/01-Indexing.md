# Indexing (Critical for Performance)

> **Phase 4 — Databases → 4.4 Indexing**
> Goal: Master database indexing — index types, single/composite/covering/partial indexes, EXPLAIN/EXPLAIN ANALYZE, index scan vs sequential scan, selectivity, and index maintenance.

---

## 0. The Big Picture

An **index** is a separate data structure that lets the database **find rows quickly** without scanning the entire table — turning an O(n) full-table scan into an O(log n) lookup (recall B-trees, Phase 2.2). **Indexing is the single most impactful database performance lever.**

```
Without an index: WHERE email = 'x' -> scan EVERY row (O(n), slow on big tables)
With an index:    WHERE email = 'x' -> jump straight to it (O(log n), fast)
```

> Think of an index like the **index at the back of a book**: instead of reading every page to find a topic, you look it up in the index and jump to the page. This note is where you learn to make queries fast — and the EXPLAIN tool to verify it.

---

## 1. How Indexes Work (B-Tree Recap)

Most database indexes are **B+ trees** (recall Phase 2.2) — balanced, high-fan-out trees optimized for disk:
```
The index stores (indexed_value -> row_location), kept SORTED in a B+ tree.
Lookup WHERE email = 'x':
  traverse the tree (O(log n)) -> find the value -> follow the pointer to the row
```
- The index is a **separate structure** from the table, kept **sorted** by the indexed column(s).
- Lookups, range scans, and ordered reads become **O(log n)** instead of O(n).
- Because B+ tree leaves are linked (Phase 2.2), **range queries** (`BETWEEN`, `>`, `ORDER BY`) are also fast.
> The primary key automatically gets an index. Other columns need indexes created explicitly (recall `CREATE INDEX`, Phase 4.2.1).

---

## 2. Index Types

PostgreSQL (and most RDBMSs) support several index types for different needs:
| Type | Best for | Notes |
|------|----------|-------|
| **B-Tree** (default) | Equality & **range** queries (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`) | The default; covers ~95% of needs |
| **Hash** | Equality only (`=`) | Faster for pure equality, but no ranges; rarely needed |
| **GIN** (Generalized Inverted Index) | **Multi-value** columns: JSONB, arrays, full-text search | "Which rows contain this element?" |
| **GiST** | Geometric/spatial data, ranges, nearest-neighbor | PostGIS, range types |
| **BRIN** (Block Range) | **Huge tables** with naturally-ordered data (e.g., timestamps) | Tiny, low-overhead; for append-only/time-series |
| **Full-text** (GIN on tsvector) | Text search | `tsvector`/`tsquery` (Phase 4.6) |
```sql
CREATE INDEX idx_users_email ON users(email);              -- B-tree (default)
CREATE INDEX idx_events_data ON events USING GIN(data);    -- GIN for JSONB (Phase 4.2.5)
CREATE INDEX idx_logs_time ON logs USING BRIN(created_at); -- BRIN for time-series
```
> **Default to B-Tree** unless you have a specific reason (JSONB/array → GIN, huge time-series → BRIN, geo → GiST). B-Tree handles equality *and* ranges *and* sorting.

---

## 3. Single-Column vs Composite Indexes

### 3.1 Single-column index
```sql
CREATE INDEX idx_users_email ON users(email);   -- speeds WHERE email = ...
```

### 3.2 Composite (multi-column) index — column order matters!
A **composite index** covers multiple columns. The **order of columns is critical** — it follows the **"leftmost prefix" rule**:
```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status, created_at);
```
This index can be used for queries filtering on:
- `user_id` ✅ (leftmost)
- `user_id AND status` ✅ (leftmost prefix)
- `user_id AND status AND created_at` ✅ (full)
- **`status` alone** ❌ (skips the leftmost column — index NOT usable)
- **`status AND created_at`** ❌ (no `user_id` prefix)
```
Composite index (user_id, status, created_at) is like a phone book sorted by
(last_name, first_name): you can find "all Smiths" or "Smith, John", but you
CANNOT efficiently find "all Johns" (the first name) without the last name.
```
> ⚠️ **Column order in a composite index is one of the most important and misunderstood indexing concepts.** Put the column you filter on **most selectively / most often** first, and equality columns before range columns. A composite index can serve queries using its **leftmost prefix** — but not queries that skip the first column.

### 3.3 Rule of thumb for composite index order
```
1. Equality columns first (WHERE col = ...)
2. Then range columns (WHERE col > ..., BETWEEN)
3. Then columns used for ORDER BY
```
Example: for `WHERE user_id = ? AND created_at > ? ORDER BY created_at`, index `(user_id, created_at)`.

---

## 4. Specialized Indexes

### 4.1 Unique index
Enforces uniqueness **and** speeds lookups (a `UNIQUE` constraint creates one — recall Phase 4.1):
```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);   -- no two rows with the same email
```

### 4.2 Partial index (index a subset of rows)
Index only the rows matching a condition — smaller and faster when you only query a subset:
```sql
-- Only index ACTIVE users (if that's what you query):
CREATE INDEX idx_active_users ON users(email) WHERE status = 'ACTIVE';
-- Only index non-deleted rows (soft delete — Phase 4.2/4.3):
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE deleted_at IS NULL;
```
> Partial indexes are great with **soft deletes** — index only `WHERE deleted_at IS NULL` so the index stays small and only covers live rows.

### 4.3 Covering index (index-only scan)
An index that contains **all columns a query needs** — so the DB answers the query **from the index alone**, never touching the table (an "index-only scan"):
```sql
-- Query: SELECT email FROM users WHERE status = 'ACTIVE';
CREATE INDEX idx_covering ON users(status) INCLUDE (email);   -- INCLUDE adds email to the index
-- Now the query reads only the index — no table lookup needed (faster!)
```
> A **covering index** avoids the extra "fetch the row from the table" step. PostgreSQL's `INCLUDE` adds non-key columns to the index for this purpose. (Recall: this is why `SELECT *` defeats covering indexes — Phase 4.2.2.)

### 4.4 Expression (functional) index
Index the result of an **expression**, so queries using that expression can use the index:
```sql
-- Index lowercased email for case-insensitive lookups:
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- Now this can use the index:
SELECT * FROM users WHERE LOWER(email) = LOWER('Alice@X.com');
```
> ⚠️ Applying a function to an indexed column in a `WHERE` clause (`WHERE LOWER(email) = ...`) **disables a normal index** on that column — unless you create an **expression index** on `LOWER(email)`. This is a common "why isn't my index used?" cause.

---

## 5. EXPLAIN & EXPLAIN ANALYZE (Reading Query Plans)

> **EXPLAIN is your #1 tool for understanding and fixing query performance.** It shows the **query plan** — how the database will execute a query, including whether it uses an index.
```sql
EXPLAIN SELECT * FROM users WHERE email = 'alice@x.com';
-- Shows the PLAN (estimated cost) without running the query

EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@x.com';
-- Actually RUNS the query and shows REAL timings + row counts
```

### 5.1 Reading a plan
```
EXPLAIN ANALYZE output (simplified):
  Index Scan using idx_users_email on users   (cost=0.29..8.30 rows=1) (actual time=0.05..0.06 rows=1)
    Index Cond: (email = 'alice@x.com')
                ^^^^^^^^^^ GOOD: using the index!

vs (no index):
  Seq Scan on users   (cost=0.00..1500.00 rows=1) (actual time=12.5..45.0 rows=1)
    Filter: (email = 'alice@x.com')
    Rows Removed by Filter: 99999
                ^^^^^^^^^^ BAD: scanned the whole table (Seq Scan)!
```
| What to look for | Meaning |
|------------------|---------|
| **Index Scan** | ✅ Using an index (fast) |
| **Index Only Scan** | ✅✅ Covering index (no table fetch) |
| **Seq Scan** (sequential scan) | ⚠️ Reading the whole table (slow on big tables) |
| **Rows Removed by Filter** | Wasted work scanning rows that didn't match |
| **cost** / **actual time** | Estimated cost / real time (compare!) |
| **rows** (estimated vs actual) | If far off → stale statistics (run `ANALYZE`) |
> Use `EXPLAIN ANALYZE` to **verify** an index is actually being used and to find slow operations (seq scans on big tables, expensive sorts, nested-loop joins on large data). This is the core skill of query optimization.

---

## 6. Index Scan vs Sequential Scan (and when seq scan is correct!)

| | **Index Scan** | **Sequential (full table) Scan** |
|---|----------------|----------------------------------|
| How | Traverse the index, fetch matching rows | Read every row in the table |
| Speed | Fast for **few** matching rows | Fast when reading **most/all** rows |
| Use when | Selective query (few matches) | Returning a large fraction of the table |

> ⚠️ **A sequential scan isn't always bad!** If a query returns **most** of the table (low selectivity), the planner may *correctly* choose a seq scan — using the index would mean many random I/Os (jumping around), which is slower than reading the table sequentially. The optimizer decides based on **selectivity** (§7). Don't assume "index = always faster."

---

## 7. Selectivity (Why Some Indexes Don't Help)

**Selectivity** = how well an index narrows the results. High selectivity (few rows per value) → index is useful. Low selectivity (many rows per value) → index may be useless.
```
High selectivity: WHERE email = 'x'  -> matches ~1 row -> index GREAT
Low selectivity:  WHERE gender = 'M' -> matches ~50% of rows -> index USELESS (seq scan better)
Low selectivity:  WHERE is_active = true -> if 95% are active -> index unhelpful
```
> ⚠️ **Don't index low-selectivity columns** (booleans, status with few values, gender) for equality — the index won't help and just costs write overhead. Indexes pay off on **high-selectivity** columns (unique-ish values: email, user_id, id) where they eliminate most rows. (A partial index, §4.2, can help for skewed distributions.)

---

## 8. The Cost of Indexes (Why Not Index Everything?)

> ⚠️ **Indexes are not free.** They speed up reads but **slow down writes** and **consume storage**. Every `INSERT`/`UPDATE`/`DELETE` must also update every affected index.
| Cost | Detail |
|------|--------|
| **Write overhead** | Each insert/update/delete must maintain all relevant indexes |
| **Storage** | Indexes can be large (sometimes bigger than the table) |
| **Maintenance** | Indexes need periodic upkeep (bloat — §9) |
> The trade-off: indexes make **reads faster** but **writes slower** (recall the read/write trade-off from denormalization, Phase 4.3). **Index what you query, not everything.** A write-heavy table with many unused indexes pays a constant tax. Periodically drop unused indexes.

### 8.1 What to index (guidelines)
| Index... | Don't index... |
|----------|----------------|
| Primary keys (automatic) | Low-selectivity columns (booleans) |
| Foreign keys (joins! Phase 4.3) | Columns you never filter/sort on |
| Columns in frequent `WHERE`/`JOIN`/`ORDER BY` | Small tables (seq scan is fine) |
| High-selectivity columns (email, user_id) | Columns that change very frequently (write cost) |
> **Always index foreign keys** — joins and "find children of X" queries depend on them (and Postgres doesn't auto-index FKs, unlike PKs). This is a top cause of slow joins.

---

## 9. Index Maintenance & Bloat

Over time, indexes accumulate **bloat** — dead space from updates/deletes (recall MVCC creates row versions — Phase 4.5). Bloated indexes are larger and slower.
| Maintenance | Command (PostgreSQL) |
|-------------|----------------------|
| Update statistics (helps the planner) | `ANALYZE table;` |
| Reclaim dead space | `VACUUM table;` / autovacuum |
| Rebuild a bloated index | `REINDEX INDEX idx_name;` (or `CONCURRENTLY`) |
| Find unused indexes | Query `pg_stat_user_indexes` |
> PostgreSQL's **autovacuum** handles most maintenance automatically. But monitor for bloat on high-churn tables, keep statistics fresh (stale stats → bad plans → seq scans), and drop indexes that `pg_stat_user_indexes` shows are never used.

---

## 10. The Indexing Workflow (Putting It Together)
```
1. Identify slow queries (logs, pg_stat_statements, APM — Phase 9).
2. Run EXPLAIN ANALYZE -> look for Seq Scans on big tables, expensive sorts.
3. Add an index on the WHERE/JOIN/ORDER BY columns (right order for composites).
4. Re-run EXPLAIN ANALYZE -> confirm an Index Scan + faster time.
5. Avoid over-indexing (write cost); drop unused indexes.
6. Keep statistics fresh (ANALYZE); monitor bloat.
```

---

## 11. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Indexing everything | Index what you query; writes pay the cost |
| Wrong composite index column order | Equality first, then range/sort; leftmost-prefix rule |
| Indexing low-selectivity columns | Index high-selectivity columns (or use partial index) |
| Function on a column disabling the index | Use an expression index (`LOWER(email)`) |
| Not indexing foreign keys | Always index FKs (joins) |
| Assuming Seq Scan is always bad | It's correct for low-selectivity / small tables |
| `SELECT *` defeating covering indexes | Select only needed columns |
| Not using EXPLAIN to verify | Always check the plan |
| Ignoring stale statistics | Run `ANALYZE`; let autovacuum work |

---

## 12. Connection to Backend / Spring (Why This Matters Later)

- **The #1 performance lever (Phase 13):** missing/wrong indexes are the most common cause of slow endpoints. EXPLAIN ANALYZE is a daily debugging tool.
- **JPA/Hibernate (Phase 5.4):** generated queries still need proper indexes; the **N+1 problem** is worsened without FK indexes. You add indexes via migrations or `@Index` in `@Table`.
- **Foreign-key indexing** makes the joins from your relationships (Phase 4.3) fast.
- **Covering indexes + projections** (Phase 5.4) reduce data fetched (recall avoid `SELECT *`, Phase 4.2.2).
- **`pg_stat_statements`** (Phase 4.6) finds slow queries to index.
- **Connection pooling + indexing** together determine DB throughput (Phase 13, Project 3 HikariCP).
- **B+ trees** (Phase 2.2) are the theory behind all of this.

---

## 13. Quick Self-Check Questions

1. What is an index, and how does it turn O(n) into O(log n)?
2. What's the default index type, and when use GIN/BRIN/GiST instead?
3. In a composite index `(a, b, c)`, which queries can use it (leftmost-prefix rule)?
4. How do you order columns in a composite index?
5. What is a covering index / index-only scan, and why is it fast?
6. What is an expression index, and what problem does it solve?
7. What does EXPLAIN ANALYZE show, and what do "Index Scan" vs "Seq Scan" mean?
8. Why isn't a sequential scan always bad? What is selectivity?
9. Why shouldn't you index everything?
10. What maintenance do indexes need?

---

## 14. Key Terms Glossary

- **Index:** a structure for fast row lookup (usually a B+ tree).
- **B-Tree / Hash / GIN / GiST / BRIN:** index types for different data/queries.
- **Composite index:** index on multiple columns (leftmost-prefix rule).
- **Leftmost prefix:** which column combos a composite index can serve.
- **Unique / partial / covering / expression index:** uniqueness / subset / all-needed-columns / computed.
- **Index-only scan:** answering a query from the index alone.
- **EXPLAIN / EXPLAIN ANALYZE:** show the query plan / run it with real timings.
- **Index Scan vs Sequential (Seq) Scan:** index lookup vs full-table read.
- **Selectivity:** how well an index narrows results.
- **Index bloat:** dead space accumulating in an index.
- **`ANALYZE` / `VACUUM` / `REINDEX`:** statistics / dead-space cleanup / rebuild.

---

*This is the note for **Section 4.4 — Indexing**.*
*Next section in roadmap: **4.5 Transactions and Concurrency Control**.*
