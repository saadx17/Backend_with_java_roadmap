# SQL: String, Date/Time & JSON Functions

> **Phase 4 — Databases → 4.2 SQL Mastery**
> Goal: Master the built-in functions for manipulating strings, dates/times, and JSON — the everyday tools for transforming data within queries (PostgreSQL focus).

---

## 0. The Big Picture

Beyond SELECT/JOIN/GROUP BY, SQL provides rich **built-in functions** to transform data **inside the query** — manipulating strings, computing dates, and querying JSON. Doing this in the database is often more efficient than pulling raw data into Java and transforming it there (less data transferred — recall Phase 0.2, and the `SELECT *` avoidance from Phase 4.2.2).

> Function syntax varies a bit by database; this note focuses on **PostgreSQL** (Phase 4.6) with notes on portability. These are the everyday data-shaping tools.

---

## 1. String Functions

| Function | Returns | Example |
|----------|---------|---------|
| `LENGTH(s)` / `CHAR_LENGTH(s)` | String length | `LENGTH('abc')` → 3 |
| `UPPER(s)` / `LOWER(s)` | Case conversion | `UPPER('abc')` → 'ABC' |
| `TRIM(s)` / `LTRIM` / `RTRIM` | Remove whitespace (or chars) | `TRIM('  x  ')` → 'x' |
| `SUBSTRING(s FROM a FOR b)` | Extract a portion | `SUBSTRING('hello' FROM 1 FOR 3)` → 'hel' |
| `CONCAT(a, b, ...)` or `\|\|` | Concatenate | `'a' \|\| 'b'` → 'ab' |
| `REPLACE(s, from, to)` | Replace substrings | `REPLACE('a-b', '-', '_')` → 'a_b' |
| `POSITION(sub IN s)` / `STRPOS` | Find index (1-based, 0 if absent) | `POSITION('l' IN 'hello')` → 3 |
| `LEFT(s, n)` / `RIGHT(s, n)` | First/last n chars | `LEFT('hello', 2)` → 'he' |
| `SPLIT_PART(s, delim, n)` | Split and take the n-th part | `SPLIT_PART('a,b,c', ',', 2)` → 'b' |
| `STRING_AGG(col, delim)` | Aggregate rows into one string | (see §1.2) |
```sql
SELECT UPPER(name), LENGTH(email), LEFT(name, 1) AS initial
FROM users;

-- Build a full name, handle NULLs:
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM users;
-- (CONCAT treats NULL as ''; the || operator yields NULL if any part is NULL)

-- Case-insensitive search (recall LIKE/ILIKE, Phase 4.2.2):
SELECT * FROM users WHERE LOWER(email) = LOWER('Alice@X.com');
SELECT * FROM users WHERE email ILIKE '%@gmail.com';   -- PostgreSQL case-insensitive
```

### 1.1 String indexing is 1-based (gotcha)
> ⚠️ Unlike Java (0-based — Phase 1.4), **SQL string positions are 1-based**. `SUBSTRING('hello', 1, 3)` = 'hel'. `POSITION` returns 1-based index, 0 if not found.

### 1.2 STRING_AGG (concatenate rows — like Collectors.joining)
```sql
-- Comma-separated list of product names per order:
SELECT order_id, STRING_AGG(product_name, ', ' ORDER BY product_name) AS products
FROM order_items
GROUP BY order_id;
```
> `STRING_AGG` is an **aggregate** that joins row values into one string — the SQL equivalent of `Collectors.joining` (Phase 1.8). (MySQL: `GROUP_CONCAT`.)

---

## 2. Date/Time Functions

Working with dates correctly is critical (recall `TIMESTAMPTZ`, Phase 4.1 — store UTC):
| Function | Returns |
|----------|---------|
| `NOW()` / `CURRENT_TIMESTAMP` | Current date+time (with timezone) |
| `CURRENT_DATE` / `CURRENT_TIME` | Today / current time |
| `AGE(a, b)` | Interval between two timestamps |
| `EXTRACT(field FROM ts)` | Get a part (year, month, day, hour, dow) |
| `DATE_TRUNC(unit, ts)` | Truncate to a unit (month, day, hour) |
| `ts + INTERVAL '1 day'` | Date arithmetic |
| `TO_CHAR(ts, format)` | Format a timestamp as text |
| `TO_DATE(s, format)` / `TO_TIMESTAMP` | Parse text into a date |
```sql
SELECT NOW();                                          -- current timestamp
SELECT created_at + INTERVAL '7 days' AS due_date FROM orders;   -- date math
SELECT EXTRACT(YEAR FROM created_at) AS year FROM orders;        -- extract a part
SELECT EXTRACT(DOW FROM created_at) AS day_of_week FROM orders;  -- 0=Sunday
SELECT AGE(NOW(), created_at) AS order_age FROM orders;          -- interval
SELECT TO_CHAR(created_at, 'YYYY-MM-DD') AS date_str FROM orders;-- format
```

### 2.1 DATE_TRUNC for grouping by period (very common)
```sql
-- Revenue per month (group time-series data by period):
SELECT DATE_TRUNC('month', created_at) AS month, SUM(total) AS revenue
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;
```
> `DATE_TRUNC` is essential for **time-series reports** (daily/monthly aggregates) — it "rounds down" a timestamp to a period so you can GROUP BY it. (Combine with window functions from the previous note for month-over-month growth.)

### 2.2 Filtering by date ranges
```sql
SELECT * FROM orders WHERE created_at >= '2026-01-01' AND created_at < '2026-02-01';  -- January
SELECT * FROM orders WHERE created_at >= NOW() - INTERVAL '30 days';  -- last 30 days
```
> ⚠️ For "in January," prefer `>= start AND < next-start` over `BETWEEN` with dates — `BETWEEN` is inclusive on both ends and can miss/duplicate boundary timestamps (e.g., the last millisecond). The half-open range `[start, nextStart)` is safer.

### 2.3 Timezone awareness (recall Phase 4.1)
- Store timestamps in **UTC** (`TIMESTAMPTZ`); convert for display.
- `AT TIME ZONE` converts: `created_at AT TIME ZONE 'America/New_York'`.
> Timezone bugs are a classic, painful class of bug — always be explicit about timezones (recall Phase 1.4 encoding discipline; same care applies to time).

---

## 3. JSON Functions (PostgreSQL JSONB)

PostgreSQL can store and **query** JSON in a column (`JSON` or, better, **`JSONB`** — binary, indexable). This blends relational and document storage (recall Phase 4.1 JSON type; NoSQL document model, Phase 4.8).
```sql
CREATE TABLE events (
    id   BIGSERIAL PRIMARY KEY,
    data JSONB                      -- stores JSON (prefer JSONB over JSON)
);
INSERT INTO events (data) VALUES ('{"type": "click", "user": {"id": 5}, "tags": ["a","b"]}');
```

### 3.1 JSON access operators
| Operator | Returns | Example |
|----------|---------|---------|
| `->` | JSON object/value | `data -> 'user'` → `{"id": 5}` (JSON) |
| `->>` | **Text** value | `data ->> 'type'` → `'click'` (text) |
| `#>` / `#>>` | Nested path (JSON / text) | `data #>> '{user,id}'` → `'5'` |
| `@>` | Contains | `data @> '{"type":"click"}'` → true |
| `?` | Key exists | `data ? 'type'` → true |
```sql
SELECT data ->> 'type' AS event_type FROM events;          -- extract as text
SELECT data #>> '{user,id}' AS user_id FROM events;        -- nested path
SELECT * FROM events WHERE data ->> 'type' = 'click';      -- filter on a JSON field
SELECT * FROM events WHERE data @> '{"type": "click"}';    -- containment (indexable!)
```
> ⚠️ Key distinction: **`->` returns JSON, `->>` returns text.** Use `->>` when comparing to a string. `JSONB` (not `JSON`) supports **GIN indexes** for fast `@>` containment queries (Phase 4.4).

### 3.2 When to use JSONB (and when not to)
| Use JSONB when... | Use normal columns when... |
|-------------------|----------------------------|
| Schema is flexible/varies per row (event payloads, settings) | Data is structured & queried/joined regularly |
| Semi-structured data (API responses, metadata) | You need strong typing, constraints, FKs |
| You don't query inside it much | You filter/sort/aggregate on the fields |
> JSONB is powerful but **don't overuse it** — if you're frequently querying inside JSON, the data probably wants to be **proper columns** (relational). JSONB is for genuinely semi-structured/variable data. (This is the relational-vs-document trade-off — Phase 4.8.)

---

## 4. Other Useful Functions

| Function | Use |
|----------|-----|
| `COALESCE` / `NULLIF` | NULL handling (recall Phase 4.2.2) |
| `ROUND(n, d)` / `CEIL` / `FLOOR` / `ABS` | Math |
| `GREATEST(a,b,...)` / `LEAST(...)` | Max/min across columns (not rows) |
| `gen_random_uuid()` | Generate a UUID (recall Phase 4.1) |
| `ARRAY_AGG(col)` | Aggregate rows into an array (Postgres) |
| `generate_series(a, b)` | Generate a sequence of numbers/dates |
```sql
SELECT ROUND(AVG(total), 2) FROM orders;               -- round to 2 decimals
SELECT GREATEST(price, min_price) FROM products;       -- max of two columns
SELECT ARRAY_AGG(tag) FROM post_tags GROUP BY post_id; -- rows -> array
```

---

## 5. Where to Transform: DB vs Application?

A recurring design question: do data shaping in **SQL** or in **Java**?
| Do it in SQL when... | Do it in Java when... |
|----------------------|------------------------|
| Aggregation/filtering reduces data transferred | Complex business logic |
| Set-based operations (the DB is optimized for them) | Logic that's hard to express in SQL |
| It avoids loading many rows into the app | Reusable across data sources |
> **Push filtering/aggregation to the database** (it's optimized for set operations and reduces network transfer — Phase 0.2). But keep complex *business* logic in the application (testable, maintainable). Don't load 1M rows into Java to count them — use `COUNT(*)`. Conversely, don't encode tangled business rules in SQL.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Assuming 0-based string indexing | SQL strings are 1-based |
| `\|\|` concatenation with NULL → NULL | Use `CONCAT` (treats NULL as '') or `COALESCE` |
| `BETWEEN` on timestamps (boundary issues) | Use `>= start AND < nextStart` |
| Storing local times / ignoring timezones | Store UTC (`TIMESTAMPTZ`), convert for display |
| `->` vs `->>` confusion | `->` = JSON, `->>` = text |
| Overusing JSONB for structured data | Use real columns when you query the fields |
| Transforming huge result sets in Java | Push filtering/aggregation to SQL |
| Function syntax differs across DBs | Check your DB's docs (PostgreSQL here) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Reports/dashboards** (Projects 4, 5): `DATE_TRUNC` + aggregates + window functions for time-series analytics.
- **Full-text search** (Phase 4.6): PostgreSQL `tsvector`/`tsquery` build on string functions.
- **JSONB columns** map to JPA via `@Type`/`@JdbcTypeCode` (Hibernate 6) — store flexible data alongside relational (Phase 5.4).
- **Push computation to the DB:** more efficient than loading rows into Java (recall Phase 4.2.2 `SELECT *` avoidance, Phase 0.2 network cost).
- **Native queries (`@Query(nativeQuery=true)`, Phase 5.4)** when you need DB-specific functions (window functions, JSONB ops) that JPQL doesn't support.
- **Timezone discipline** prevents a whole class of bugs (Phase 1.4 encoding parallel).

---

## 8. Quick Self-Check Questions

1. Name five common string functions. Is SQL string indexing 0- or 1-based?
2. What does `STRING_AGG` do, and what's its Java equivalent?
3. What does `DATE_TRUNC` do, and why is it key for time-series reports?
4. Why prefer `>= start AND < nextStart` over `BETWEEN` for date ranges?
5. Why store timestamps in UTC with `TIMESTAMPTZ`?
6. What's the difference between the `->` and `->>` JSON operators?
7. When should you use JSONB vs normal columns?
8. When should you transform data in SQL vs in Java?

---

## 9. Key Terms Glossary

- **String functions:** `LENGTH`, `UPPER`/`LOWER`, `SUBSTRING`, `CONCAT`/`||`, `REPLACE`, `SPLIT_PART`, `STRING_AGG`.
- **1-based indexing:** SQL string positions start at 1.
- **Date/time functions:** `NOW`, `EXTRACT`, `DATE_TRUNC`, `INTERVAL`, `TO_CHAR`/`TO_DATE`, `AGE`.
- **`DATE_TRUNC`:** round a timestamp down to a period (for grouping).
- **`TIMESTAMPTZ` / UTC:** timezone-aware storage.
- **JSONB:** binary, indexable JSON column type.
- **`->` / `->>`:** JSON access (returns JSON / text).
- **`@>`:** JSONB containment (GIN-indexable).
- **`STRING_AGG` / `ARRAY_AGG`:** aggregate rows into a string/array.
- **Set-based processing:** doing work in the DB rather than row-by-row in the app.

---

*Previous topic: **Window Functions & CTEs**.*
*This completes **Section 4.2 — SQL Mastery**.*
*Next section in roadmap: **4.3 Database Design & Normalization**.*
