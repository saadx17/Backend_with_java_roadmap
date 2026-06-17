# JDBC (Java Database Connectivity)

> **Phase 4 — Databases → 4.7 JDBC**
> Goal: Master JDBC — the low-level Java API for databases: connections, `PreparedStatement` (and why), `ResultSet`, transaction management, batch operations, and resource handling. Plus connection pooling (HikariCP).

---

## 0. The Big Picture

**JDBC (Java Database Connectivity)** is the **standard Java API for talking to relational databases**. It's the low-level foundation: every higher-level tool (Spring JDBC, JPA/Hibernate — Phase 5.4) is built on top of JDBC. Understanding it shows you *what ORMs abstract away*.

```
Your Java code  ->  JDBC API  ->  JDBC Driver (e.g., postgresql.jar)  ->  Database
```

> Project 3 uses **pure JDBC (no ORM)** deliberately — so you understand the mechanics before letting frameworks hide them. This builds on all of Phase 4 (SQL, transactions, PostgreSQL) and Phase 1.5/1.9 (exceptions, resource management).

---

## 1. JDBC Architecture

JDBC is an **interface**; each database provides a **driver** implementing it:
```
java.sql.*  (the JDBC API — interfaces: Connection, Statement, ResultSet)
     ^ implemented by
JDBC Driver (vendor-specific JAR, e.g., org.postgresql:postgresql)
     ^ talks to
The database (over TCP, port 5432 — Phase 0.2/4.6)
```
- You program against `java.sql` **interfaces** (portable across databases — recall "program to interfaces," Phase 1.3).
- The **driver** (a dependency, `runtime` scope — recall Phase 3.1.2) implements them for a specific DB.
> Since JDBC 4+, drivers are **auto-loaded** (via the service loader) — you just add the driver dependency; no manual `Class.forName("org.postgresql.Driver")` needed.

---

## 2. Establishing a Connection

A **`Connection`** represents a session with the database:
```java
String url = "jdbc:postgresql://localhost:5432/mydb";   // JDBC URL
String user = "postgres", password = "secret";

try (Connection conn = DriverManager.getConnection(url, user, password)) {   // try-with-resources!
    // use the connection
}   // auto-closed (Phase 1.5)
```
### 2.1 The JDBC URL
```
jdbc:postgresql://host:port/database?param=value
       ^driver    ^host     ^port  ^db
```
> ⚠️ A `Connection` is an **expensive, scarce resource** (a network connection + DB-side process — Phase 4.6). **Always close it** (try-with-resources, Phase 1.5) — leaking connections exhausts the pool/`max_connections` and crashes the app. In real apps you get connections from a **pool** (§8), not `DriverManager` directly.

---

## 3. Statement vs PreparedStatement (Critical)

There are two ways to execute SQL. **Always use `PreparedStatement`.**

### 3.1 Statement (raw SQL — DANGEROUS)
```java
Statement stmt = conn.createStatement();
// ❌ NEVER build SQL with string concatenation from user input:
String sql = "SELECT * FROM users WHERE name = '" + userInput + "'";   // SQL INJECTION!
ResultSet rs = stmt.executeQuery(sql);
```
> ⚠️ **String-concatenated SQL = SQL injection vulnerability.** If `userInput` is `' OR '1'='1`, the query returns all rows; `'; DROP TABLE users; --` could destroy data. This is **OWASP A03: Injection** (Phase 15). **Never** build SQL by concatenating untrusted input.

### 3.2 PreparedStatement (parameterized — SAFE & faster)
A **`PreparedStatement`** uses **placeholders (`?`)** for values; the driver sends the SQL and the values **separately**, so user input can never be interpreted as SQL:
```java
String sql = "SELECT * FROM users WHERE name = ? AND age > ?";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setString(1, userInput);     // parameter 1 (1-based!) — safely bound
    ps.setInt(2, 18);               // parameter 2
    try (ResultSet rs = ps.executeQuery()) {
        // ...
    }
}
```
### 3.3 Why PreparedStatement (three reasons)
| Benefit | Why |
|---------|-----|
| **Prevents SQL injection** | Values are bound separately, never parsed as SQL |
| **Better performance** | The DB can parse/plan the SQL once and reuse it (precompiled) |
| **Type safety / correctness** | `setString`/`setInt` handle escaping, quoting, types, NULLs |
> ⚠️ **Always use `PreparedStatement` with `?` placeholders for any value — especially user input.** This is non-negotiable for security. Parameters are **1-based** (`setX(1, ...)` is the first `?`). (Spring's `JdbcTemplate`/JPA always parameterize under the hood.)

---

## 4. Executing SQL

| Method | For | Returns |
|--------|-----|---------|
| `executeQuery()` | SELECT | a `ResultSet` |
| `executeUpdate()` | INSERT/UPDATE/DELETE | int (rows affected) |
| `execute()` | any (incl. DDL) | boolean |
```java
// Query:
ResultSet rs = ps.executeQuery();
// Update:
int rowsAffected = ps.executeUpdate();
```

### 4.1 Getting generated keys (auto-increment IDs)
```java
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO users (name) VALUES (?)", Statement.RETURN_GENERATED_KEYS);
ps.setString(1, "Alice");
ps.executeUpdate();
try (ResultSet keys = ps.getGeneratedKeys()) {
    if (keys.next()) long id = keys.getLong(1);   // the generated PK (recall Phase 4.1)
}
```

---

## 5. ResultSet (Reading Query Results)

A **`ResultSet`** is a cursor over the rows returned by a query. You iterate with `next()` and read columns by name or index:
```java
try (ResultSet rs = ps.executeQuery()) {
    while (rs.next()) {                          // advance to the next row (false when done)
        long id = rs.getLong("id");             // by column name (preferred — clearer)
        String name = rs.getString("name");
        int age = rs.getInt("age");
        BigDecimal balance = rs.getBigDecimal("balance");   // DECIMAL -> BigDecimal (Phase 4.1)
        Timestamp created = rs.getTimestamp("created_at");
    }
}
```
### 5.1 Key points
- `rs.next()` advances the cursor and returns `false` when there are no more rows (loop condition — recall Phase 1.2 while loops).
- Read by **column name** (`getString("name")`) for clarity, or 1-based index.
- Use the **right getter** for the type: `getInt`, `getLong`, `getString`, `getBigDecimal` (money! — Phase 4.1), `getTimestamp`, `getBoolean`.
- ⚠️ **NULL handling:** `getInt` returns `0` for a SQL NULL — use `rs.wasNull()` to distinguish, or use object getters (recall Phase 1.2 — wrapper types for nullability).
> Mapping `ResultSet` rows to Java objects manually is tedious boilerplate — **this is exactly what ORMs (JPA/Hibernate) and `JdbcTemplate`'s `RowMapper` automate** (Phase 5.4).

---

## 6. Transaction Management (recall Phase 4.5)

By default, JDBC is in **auto-commit** mode (each statement commits immediately). For multi-statement transactions, turn it off and commit/rollback manually:
```java
try (Connection conn = dataSource.getConnection()) {
    conn.setAutoCommit(false);                  // start a transaction (disable auto-commit)
    try {
        // multiple statements...
        ps1.executeUpdate();
        ps2.executeUpdate();
        conn.commit();                          // all-or-nothing (Phase 4.5)
    } catch (SQLException e) {
        conn.rollback();                        // undo everything on error
        throw e;
    }
}
```
### 6.1 Savepoints
```java
Savepoint sp = conn.setSavepoint();
// ...
conn.rollback(sp);                              // partial rollback to the savepoint (Phase 4.5)
```
### 6.2 Isolation level
```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);   // (Phase 4.5)
```
> ⚠️ **Auto-commit is on by default** — each statement is its own transaction. For atomic multi-step operations, you **must** `setAutoCommit(false)` and `commit()`/`rollback()` explicitly. This manual ceremony is exactly what Spring's `@Transactional` automates (Phase 5.5).

---

## 7. Batch Operations (Performance)

Executing many statements one-by-one means a network round-trip each (slow — recall Phase 0.2 latency). **Batching** sends many at once:
```java
String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    for (User u : users) {
        ps.setString(1, u.getName());
        ps.setString(2, u.getEmail());
        ps.addBatch();                          // queue this statement
    }
    int[] results = ps.executeBatch();          // send all at once -> far fewer round-trips
}
```
> Batching is dramatically faster for bulk inserts/updates (Project 2/3) — one round-trip for many rows instead of one per row. (Postgres tip: add `?reWriteBatchedInserts=true` to the JDBC URL for even faster batched inserts.)

---

## 8. Resource Management (try-with-resources)

JDBC resources — `Connection`, `Statement`/`PreparedStatement`, `ResultSet` — are all `AutoCloseable` (Phase 1.5) and **must be closed**, in reverse order of opening. **Always use try-with-resources:**
```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    while (rs.next()) { /* ... */ }
}   // rs, ps, conn auto-closed in reverse order — even on exception (Phase 1.5)
```
> ⚠️ **Leaking JDBC resources is a top production bug** — unclosed connections exhaust the pool → the app hangs ("connection pool timeout"). try-with-resources (Phase 1.5) is mandatory. `SQLException` is a **checked exception** (Phase 1.5) — handle or propagate it.

---

## 9. Connection Pooling (HikariCP)

> Opening a database connection is **expensive** (TCP handshake + auth + DB-side process — Phase 0.2/4.6). Creating one per request is far too slow. A **connection pool** keeps a set of open connections and **reuses** them — exactly like a thread pool (Phase 1.10).

### 9.1 HikariCP — the standard
**HikariCP** is the fast, default connection pool (used by Spring Boot). You get connections from a `DataSource` (the pool), use them, and `close()` returns them to the pool (not actually closed):
```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
config.setUsername("postgres");
config.setPassword("secret");
config.setMaximumPoolSize(10);              // max connections in the pool
HikariDataSource dataSource = new HikariDataSource(config);

try (Connection conn = dataSource.getConnection()) {   // BORROW from the pool
    // use it
}   // close() RETURNS it to the pool (doesn't actually close the connection)
```
### 9.2 Key pool settings (Phase 13 tuning)
| Setting | Meaning |
|---------|---------|
| `maximumPoolSize` | Max connections (don't exceed DB `max_connections` / PgBouncer) |
| `minimumIdle` | Min idle connections kept ready |
| `connectionTimeout` | How long to wait for a connection before failing |
| `idleTimeout` / `maxLifetime` | When to retire idle/old connections |
> ⚠️ **Pool sizing matters:** too small → requests queue waiting for a connection; too large → overwhelms the DB (each connection = a DB process). A common formula is roughly `connections = cores × 2` (recall thread-pool sizing, Phase 1.10/13). Combine with **PgBouncer** at scale (Phase 4.6). Spring Boot uses HikariCP by default — you just configure it in `application.yml`.

---

## 10. JDBC vs Higher-Level Tools (the abstraction ladder)
| Layer | What it does | When |
|-------|-------------|------|
| **Raw JDBC** | Manual SQL, ResultSet mapping, connection handling | Learning, fine control, simple cases (Project 3) |
| **Spring JdbcTemplate** | JDBC minus boilerplate (handles resources, mapping via `RowMapper`) | SQL-centric apps |
| **JPA/Hibernate** | Object-relational mapping (entities ↔ tables) | Most Spring apps (Phase 5.4) |
> JDBC is the **foundation everything else uses**. Higher tools remove its boilerplate (resource handling, manual mapping) but you should understand JDBC to know what they do — and to drop down to it when needed.

---

## 11. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| String-concatenated SQL | Use `PreparedStatement` with `?` (SQL injection!) |
| Not closing connections/statements/result sets | try-with-resources (resource leak / pool exhaustion) |
| Forgetting `setAutoCommit(false)` for multi-step | Manage the transaction explicitly |
| Many single-row inserts | Use `addBatch()`/`executeBatch()` |
| `getInt` returning 0 for SQL NULL | Use `wasNull()` or object getters |
| Wrong getter type (e.g., float for money) | Use `getBigDecimal` (Phase 4.1) |
| Creating a connection per request | Use a connection pool (HikariCP) |
| Pool too small/large | Size it deliberately (Phase 13) |

---

## 12. Connection to Backend / Spring (Why This Matters Later)

- **Everything is built on JDBC:** Spring `JdbcTemplate`, JPA/Hibernate (Phase 5.4) all use it underneath — understanding it demystifies them.
- **`PreparedStatement` = SQL injection prevention** (Phase 15 OWASP A03) — JPA/Hibernate parameterize automatically.
- **HikariCP** is Spring Boot's default pool — you configure it in `application.yml` and tune it for throughput (Phase 13).
- **Transaction management** → Spring `@Transactional` (Phase 5.5) automates `setAutoCommit`/`commit`/`rollback`.
- **`RowMapper`** (Spring JDBC) and ORM mapping automate the `ResultSet` → object boilerplate.
- **Batch operations** for bulk processing (Projects 2/3).
- **Resource discipline** (try-with-resources, Phase 1.5) prevents the #1 JDBC bug.

---

## 13. Quick Self-Check Questions

1. What is JDBC, and how does the driver fit in?
2. Why must you use `PreparedStatement` instead of `Statement` with concatenation?
3. What three benefits does `PreparedStatement` provide?
4. How do you read rows from a `ResultSet`, and what's the NULL gotcha?
5. How do you manage a transaction in raw JDBC (auto-commit, commit, rollback)?
6. Why and how do you batch operations?
7. Why must JDBC resources be closed, and how?
8. What is connection pooling, why is it needed, and what is HikariCP?
9. How does JDBC relate to JdbcTemplate and JPA?

---

## 14. Key Terms Glossary

- **JDBC:** standard Java API for relational databases.
- **JDBC driver:** vendor implementation of the JDBC interfaces.
- **`Connection`:** a database session (expensive; must close).
- **`Statement` vs `PreparedStatement`:** raw SQL vs parameterized (`?`) — always use the latter.
- **SQL injection:** attack via unsanitized SQL (prevented by `PreparedStatement`).
- **`ResultSet`:** cursor over query results (iterate with `next()`).
- **`executeQuery` / `executeUpdate`:** SELECT / DML execution.
- **Auto-commit:** each statement commits immediately (default).
- **Batch (`addBatch`/`executeBatch`):** send many statements at once.
- **Connection pool / HikariCP:** reuse open connections (Spring Boot default).
- **`DataSource`:** the source of (pooled) connections.
- **`RowMapper`:** maps a ResultSet row to an object (Spring JDBC).

---

*This is the note for **Section 4.7 — JDBC**.*
*Next section in roadmap: **4.8 NoSQL Databases**.*
