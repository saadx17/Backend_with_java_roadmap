# PostgreSQL (Primary Database)

> **Phase 4 — Databases → 4.6 PostgreSQL**
> Goal: Get hands-on with PostgreSQL — installation/clients, its distinctive features (JSONB, arrays, full-text, etc.), stored procedures/triggers/views, partitioning, performance tuning, and backup/replication.

---

## 0. The Big Picture

**PostgreSQL** ("Postgres") is a powerful, open-source, standards-compliant relational database — the **most common choice for new backend projects**. It implements everything from Phases 4.1–4.5 (SQL, ACID, MVCC, indexing) and adds rich features (JSONB, arrays, full-text search, extensions) that blur the line between relational and document databases.

```
PostgreSQL = a robust, free, feature-rich RDBMS -> the default backend database
```

> This roadmap uses PostgreSQL as the **primary database** (Projects 3–7). It's reliable, has excellent SQL support, strong concurrency (MVCC — Phase 4.5), and a huge ecosystem. This note covers the practical, Postgres-specific knowledge on top of the general SQL/DB concepts you already learned.

---

## 1. Installation & Clients

### 1.1 Getting Postgres
| Method | Use |
|--------|-----|
| Native install (apt/brew/installer) | Local development |
| **Docker** (`docker run postgres`) | The easiest for dev (Phase 10) |
| Managed cloud (AWS RDS, Cloud SQL) | Production (Phase 16.6) |
```bash
docker run --name pg -e POSTGRES_PASSWORD=secret -p 5432:5432 -d postgres:16
# port 5432 = PostgreSQL default (recall Phase 0.2 ports)
```

### 1.2 psql — the command-line client
`psql` is the interactive terminal for Postgres:
```bash
psql -h localhost -U postgres -d mydb     # connect
```
| psql meta-command | Does |
|-------------------|------|
| `\l` | List databases |
| `\c dbname` | Connect to a database |
| `\dt` | List tables |
| `\d tablename` | Describe a table (columns, indexes, constraints) |
| `\di` | List indexes |
| `\du` | List users/roles |
| `\timing` | Toggle query timing |
| `\q` | Quit |

### 1.3 GUI clients
| Tool | Note |
|------|------|
| **pgAdmin** | The official GUI |
| **DBeaver** | Popular, multi-database |
| **DataGrip** / IntelliJ DB tools | JetBrains |
> For learning, **psql** + a GUI (pgAdmin/DBeaver) is a good combo. In a Spring app you'll connect via JDBC (Phase 4.7) / JPA (Phase 5.4).

---

## 2. PostgreSQL-Specific Features

This is where Postgres shines beyond standard SQL.

### 2.1 Auto-increment: SERIAL / IDENTITY (recall Phase 4.1)
```sql
id BIGSERIAL PRIMARY KEY                     -- classic auto-increment (uses a sequence)
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY   -- SQL-standard (preferred, modern)
```

### 2.2 UUID
```sql
id UUID PRIMARY KEY DEFAULT gen_random_uuid()   -- built-in random UUID (recall Phase 4.1)
```

### 2.3 JSONB (recall Phase 4.2.5)
Binary JSON — store and **query** semi-structured data, with GIN indexing:
```sql
CREATE TABLE events (id BIGSERIAL PRIMARY KEY, data JSONB);
SELECT data ->> 'type' FROM events WHERE data @> '{"type":"click"}';
CREATE INDEX idx_events_data ON events USING GIN(data);   -- index for @> queries (Phase 4.4)
```
> Prefer **`JSONB`** over `JSON` (binary, indexable, faster) — but don't overuse it for structured data (Phase 4.2.5).

### 2.4 Arrays as columns
Postgres can store arrays directly in a column:
```sql
CREATE TABLE posts (id BIGSERIAL PRIMARY KEY, tags TEXT[]);
INSERT INTO posts (tags) VALUES (ARRAY['java', 'spring']);
SELECT * FROM posts WHERE 'java' = ANY(tags);          -- query array membership
SELECT * FROM posts WHERE tags @> ARRAY['java'];        -- contains
```
> Arrays are convenient but a **junction table** (Phase 4.3) is often better for relational integrity/queryability — use arrays for simple, denormalized tag-like data.

### 2.5 Enums (CREATE TYPE)
Custom enumerated types:
```sql
CREATE TYPE order_status AS ENUM ('PENDING', 'SHIPPED', 'DELIVERED', 'CANCELLED');
CREATE TABLE orders (id BIGSERIAL PRIMARY KEY, status order_status NOT NULL);
```
> ⚠️ DB enums are rigid (adding a value requires `ALTER TYPE`). Many teams prefer a `VARCHAR` + CHECK constraint, or a lookup table, for flexibility. (Maps to Java enums via `@Enumerated(EnumType.STRING)` — recall Phase 1.3/5.4.)

### 2.6 Range types
Store ranges (e.g., date ranges) with built-in overlap operators:
```sql
CREATE TABLE bookings (room INT, during TSRANGE);   -- timestamp range
SELECT * FROM bookings WHERE during && '[2026-01-01, 2026-01-05)';  -- && = overlaps
```
Great for booking/scheduling systems (exclusion constraints prevent double-booking).

### 2.7 Full-text search (tsvector / tsquery)
Built-in text search — no separate search engine needed for moderate needs:
```sql
SELECT * FROM articles
WHERE to_tsvector('english', body) @@ to_tsquery('english', 'database & performance');
-- Index it for speed:
CREATE INDEX idx_articles_fts ON articles USING GIN(to_tsvector('english', body));
```
- `tsvector` = preprocessed searchable text; `tsquery` = the search query; `@@` = matches.
- Handles stemming, ranking, stop words.
> Postgres full-text search is great for moderate search needs (Project 4 "full-text search on descriptions"). For advanced search at scale, use **Elasticsearch** (Phase 16.4).

### 2.8 Schemas & extensions
- **Schemas** (namespaces — recall Phase 4.1): `CREATE SCHEMA auth; auth.users`. Default schema is `public`.
- **Extensions** add features: `CREATE EXTENSION pgcrypto;` (crypto), `postgis` (geospatial), `pg_stat_statements` (query stats — §6), `uuid-ossp`.

---

## 3. Stored Procedures & Functions (PL/pgSQL)

Postgres lets you write **functions/procedures** in `PL/pgSQL` (its procedural language) that run *inside* the database:
```sql
CREATE OR REPLACE FUNCTION get_user_order_count(uid BIGINT)
RETURNS INTEGER AS $$
BEGIN
    RETURN (SELECT COUNT(*) FROM orders WHERE user_id = uid);
END;
$$ LANGUAGE plpgsql;

SELECT get_user_order_count(5);
```
- **Functions** return a value; **procedures** (`CALL`) can manage transactions.
- Run logic close to the data (fast, no round-trips).
> ⚠️ **Use stored procedures sparingly in app development.** Business logic in the database is harder to test, version-control, and debug than Java code. Prefer keeping logic in the application (recall Phase 4.2.5 — DB vs app). Use them for performance-critical, data-intensive operations.

---

## 4. Triggers, Views & Materialized Views

### 4.1 Triggers
A **trigger** automatically runs a function in response to INSERT/UPDATE/DELETE:
```sql
CREATE TRIGGER set_updated_at
BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION update_timestamp();   -- auto-update updated_at (Phase 4.3)
```
> Triggers are powerful (auto-audit, maintaining denormalized counts — Phase 4.3) but can hide logic and cause surprises. Use deliberately; Spring auditing (`@LastModifiedDate`, Phase 5.4) often handles `updated_at` instead.

### 4.2 Views (recall Phase 4.2.1)
A saved query that acts as a virtual table:
```sql
CREATE VIEW active_users AS SELECT * FROM users WHERE deleted_at IS NULL;
```

### 4.3 Materialized views
Unlike a regular view, a **materialized view** *stores* the query result (faster reads, but can be stale — must refresh):
```sql
CREATE MATERIALIZED VIEW daily_sales AS
    SELECT DATE_TRUNC('day', created_at) AS day, SUM(total) FROM orders GROUP BY 1;
REFRESH MATERIALIZED VIEW daily_sales;          -- recompute (or CONCURRENTLY)
```
> **Materialized views** are great for expensive aggregations/reports (dashboards — Projects 4/5) that don't need real-time data — compute once, read many times (a denormalization/caching technique, Phase 4.3).

---

## 5. Partitioning

**Partitioning** splits a large table into smaller physical pieces (partitions) while presenting one logical table — improving performance and manageability for huge tables.
| Strategy | Split by |
|----------|----------|
| **Range** | Value ranges (e.g., by month: `created_at`) |
| **List** | Discrete values (e.g., by region) |
| **Hash** | A hash of the key (even distribution) |
```sql
CREATE TABLE orders (id BIGSERIAL, created_at TIMESTAMPTZ, total DECIMAL)
    PARTITION BY RANGE (created_at);
CREATE TABLE orders_2026_01 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```
> Benefits: queries that filter by the partition key only scan relevant partitions (**partition pruning**); old partitions can be dropped/archived cheaply. Used for **very large** time-series/log tables. (Distinct from **sharding** — splitting across servers, Phase 16.6.)

---

## 6. Performance Tuning

### 6.1 Configuration (postgresql.conf)
Key memory settings:
| Setting | Purpose |
|---------|---------|
| `shared_buffers` | DB cache (~25% of RAM) |
| `work_mem` | Memory per sort/hash operation |
| `effective_cache_size` | Hint to the planner about OS cache |
| `max_connections` | Connection limit (see PgBouncer below) |

### 6.2 pg_stat_statements (find slow queries)
This extension tracks query statistics — the go-to for finding what to optimize:
```sql
CREATE EXTENSION pg_stat_statements;
SELECT query, calls, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;
```
> `pg_stat_statements` shows your **slowest/most-frequent queries** → then use `EXPLAIN ANALYZE` (Phase 4.4) and add indexes. This is the core performance-tuning loop.

### 6.3 PgBouncer (connection pooling at the DB)
Postgres connections are relatively expensive (each is a process). **PgBouncer** is a lightweight connection pooler that sits in front of Postgres, multiplexing many client connections onto fewer DB connections.
> ⚠️ Postgres has a `max_connections` limit; opening thousands of connections exhausts it. **PgBouncer** (external pooler) + **HikariCP** (app-side pool — Phase 4.7) together manage connections efficiently. Critical at scale (Phase 13).

### 6.4 Other tuning levers (recall Phase 4.4)
- **Indexes** (the #1 lever — Phase 4.4).
- **`EXPLAIN ANALYZE`** to find seq scans/slow ops.
- **`VACUUM`/autovacuum** for MVCC bloat (Phase 4.5).
- **`ANALYZE`** to keep statistics fresh (good query plans).

---

## 7. Backup & Restore

| Tool | Use |
|------|-----|
| **`pg_dump`** | Logical backup of one database (SQL or custom format) |
| **`pg_dumpall`** | Backup all databases + global objects |
| **`pg_restore`** | Restore a custom-format dump |
| **PITR** (Point-In-Time Recovery) | Restore to a specific moment using WAL archives |
```bash
pg_dump -U postgres mydb > backup.sql           # backup
psql -U postgres mydb < backup.sql              # restore
pg_dump -Fc mydb > backup.dump && pg_restore -d mydb backup.dump   # custom format
```
> ⚠️ **Backups are only useful if you test restoring them.** A backup you've never restored is a hope, not a plan. Automate backups *and* periodically verify restores. PITR (via WAL — recall durability, Phase 4.5) enables recovery to any point in time.

---

## 8. Replication Basics

**Replication** copies data from a **primary** to one or more **replicas (standbys)** — for high availability and read scaling.
| Type | Behavior |
|------|----------|
| **Streaming replication** | Replicas continuously receive the primary's WAL stream |
| **Synchronous** | Primary waits for replica confirmation (no data loss, slower) |
| **Asynchronous** | Primary doesn't wait (faster, small lag, possible loss on failure) |
```
Primary (writes) --WAL stream--> Replica 1 (reads)
                              --> Replica 2 (reads)
```
> Use cases: **read replicas** offload read traffic from the primary (scaling reads — Phase 13/16.6); **failover** to a standby if the primary dies (high availability). Writes go to the primary; reads can be distributed to replicas. (Managed services like RDS handle this for you — Phase 16.6.)

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Too many direct connections | Use HikariCP + PgBouncer |
| Business logic in stored procedures | Keep it in the app (testable, versioned) |
| Overusing DB enums (rigid) | Consider VARCHAR + CHECK or lookup table |
| JSONB for structured data | Use real columns when you query the fields |
| Not running `pg_stat_statements`/EXPLAIN | Use them to find & fix slow queries |
| Backups never tested | Periodically verify restores |
| Ignoring autovacuum (MVCC bloat) | Let it run; monitor high-churn tables |
| Materialized view assumed real-time | It's stale until `REFRESH` |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Spring Data JPA / JDBC (Phase 4.7, 5.4)** connect to Postgres via the JDBC driver (port 5432).
- **JSONB, arrays, enums** map to JPA via Hibernate types (`@JdbcTypeCode`, `@Enumerated`).
- **Full-text search** powers search features (Project 4); Elasticsearch for scale (Phase 16.4).
- **HikariCP + PgBouncer** (Phase 4.7, 13) manage connections — critical for throughput.
- **`pg_stat_statements` + EXPLAIN + indexes** (Phase 4.4) is the performance-tuning loop (Phase 13).
- **Read replicas** for read scaling (Phase 13, 16.6); **Testcontainers** spins up real Postgres in tests (Phase 6).
- **Migrations (Phase 8 Flyway)** apply DDL to Postgres safely.
- **Docker Postgres** for local dev (Project 6 docker-compose).

---

## 11. Quick Self-Check Questions

1. What is psql, and name five useful meta-commands.
2. What's the difference between `JSON` and `JSONB`, and how do you index JSONB?
3. How do you store an auto-increment PK and a UUID PK in Postgres?
4. What is full-text search (`tsvector`/`tsquery`), and when use Elasticsearch instead?
5. When should you use stored procedures, and why use them sparingly?
6. What's the difference between a view and a materialized view?
7. What is partitioning, and what are the three strategies?
8. What does `pg_stat_statements` do, and what's the tuning loop?
9. Why use PgBouncer in addition to HikariCP?
10. What is streaming replication, and what are read replicas for?

---

## 12. Key Terms Glossary

- **PostgreSQL:** open-source, feature-rich RDBMS (default backend DB).
- **psql / pgAdmin:** CLI client / official GUI.
- **JSONB / arrays / enums / range types:** Postgres data-type features.
- **Full-text search (tsvector/tsquery):** built-in text search.
- **Schema / extension:** namespace / add-on feature.
- **PL/pgSQL:** Postgres procedural language for functions.
- **Trigger:** auto-run function on data change.
- **View / materialized view:** virtual query / stored, refreshable result.
- **Partitioning (range/list/hash):** splitting a big table physically.
- **`pg_stat_statements`:** query statistics extension.
- **PgBouncer:** external connection pooler.
- **`pg_dump`/`pg_restore`/PITR:** backup/restore/point-in-time recovery.
- **Streaming replication / read replica / failover:** HA & read scaling.

---

*This is the note for **Section 4.6 — PostgreSQL**.*
*Next section in roadmap: **4.7 JDBC**.*
