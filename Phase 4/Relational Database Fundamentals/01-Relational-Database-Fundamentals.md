# Relational Database Fundamentals

> **Phase 4 — Databases → 4.1 Relational Database Fundamentals**
> Goal: Understand the building blocks of relational databases — tables/rows/columns/schemas, primary & foreign keys, data types, and constraints — before learning SQL.

---

## 0. The Big Picture

A **relational database** organizes data into **tables** (relations) of **rows** and **columns**, with **relationships** between tables enforced by **keys** and **constraints**. It's the most common way backends store persistent, structured data.

```
A table = a spreadsheet-like grid:
  columns (fields, typed)  ->
  +----+----------+------------------+
  | id | name     | email            |   <- each ROW is one record
  +----+----------+------------------+
  | 1  | Alice    | alice@x.com      |
  | 2  | Bob      | bob@x.com        |
  +----+----------+------------------+
```

> This is the foundation for everything in Phase 4 (SQL, design, indexing, transactions, PostgreSQL, JDBC). Relational databases (PostgreSQL, MySQL) back the vast majority of backend systems — and recall *why* we need them: in-memory data (HashMaps, lists) vanishes on restart; databases provide **durable, queryable persistence** (recall Phase 0.1 RAM vs storage).

---

## 1. What "Relational" Means

The model is based on **relations** (tables) and the **relationships** between them. Data is split across multiple tables and linked by keys, rather than duplicated.
```
users table          orders table
+----+-------+       +----+---------+----------+
| id | name  |       | id | user_id | total    |
+----+-------+       +----+---------+----------+
| 1  | Alice |  <----| 10 |    1    | 99.50    |   (order 10 belongs to user 1)
| 2  | Bob   |       | 11 |    2    | 12.00    |
+----+-------+       +----+---------+----------+
                          ^ user_id links to users.id (a relationship)
```
> The relational model (Edgar Codd, 1970) + **SQL** (the query language, section 4.2) is the dominant paradigm. **RDBMS** = Relational Database Management System (PostgreSQL, MySQL, Oracle, SQL Server).

---

## 2. Core Structures: Table, Row, Column, Schema

| Term | Meaning | Spreadsheet analogy |
|------|---------|---------------------|
| **Table** (relation) | A collection of related data of one entity type | A sheet |
| **Row** (record/tuple) | One instance of the entity | A row |
| **Column** (field/attribute) | A named, typed property | A column |
| **Schema** | A namespace grouping tables (+ logical structure) | A workbook / folder of sheets |

### 2.1 Tables and rows
- A **table** represents one **entity type** (users, orders, products).
- Each **row** is one record (one user, one order).
- Each **column** has a **name** and a **data type** (§4) — all values in a column are the same type.

### 2.2 Schema (two meanings)
1. **Logical schema:** the overall *design* — which tables exist, their columns, types, and relationships (the "blueprint").
2. **Namespace schema:** in PostgreSQL, a `schema` is a named container for tables (like a folder), e.g., `public.users`, `auth.sessions`. Lets you organize/separate tables within one database.
> ⚠️ "Schema" is overloaded: it can mean the *structure/design* OR a *namespace*. Context tells you which. (PostgreSQL's default schema is `public` — Phase 4.6.)

---

## 3. Keys (How Rows Are Identified & Related)

Keys are columns that **uniquely identify** rows and **establish relationships** between tables — the heart of the relational model.

### 3.1 Primary Key (PK)
A **primary key** uniquely identifies each row in a table. It must be **unique** and **NOT NULL** — every row has exactly one, distinct PK.
```sql
CREATE TABLE users (
    id   BIGINT PRIMARY KEY,    -- the primary key
    name VARCHAR(100)
);
```
| Rule | Why |
|------|-----|
| Unique | No two rows share a PK value |
| Not null | Every row must be identifiable |
| Stable | Shouldn't change (other tables reference it) |
| One per table | Exactly one PK (can be composite — multiple columns) |

### 3.2 Natural vs Surrogate keys
| | **Natural key** | **Surrogate key** |
|---|-----------------|-------------------|
| Source | A real-world unique attribute (email, SSN, ISBN) | An artificial, system-generated value (an `id`) |
| Pros | Meaningful | Stable, never changes, no business meaning to leak |
| Cons | Can change (email changes!), may be large/sensitive | No inherent meaning |
> **Prefer surrogate keys** (a generated `id`) for primary keys in most cases — they're stable (a user's email might change, but their `id` never does) and decouple identity from business data. Natural keys can still be enforced as `UNIQUE` constraints (§5).

### 3.3 Auto-increment vs UUID (surrogate key options)
| | **Auto-increment** (sequential) | **UUID** (random) |
|---|-------------------------------|-------------------|
| Example | 1, 2, 3, ... (`SERIAL`/`BIGSERIAL`, `IDENTITY`) | `550e8400-e29b-41d4-...` |
| Pros | Small, ordered, fast indexes, human-readable | Globally unique, generatable client-side, no central counter |
| Cons | Reveals row count/order, needs the DB to assign it, hard across shards | Larger (16 bytes), random → worse index locality |
| Use when | Single DB, simple apps (default) | Distributed systems, sharding, security (don't expose counts) |
```sql
id BIGSERIAL PRIMARY KEY            -- PostgreSQL auto-increment (recall 4.6)
id UUID PRIMARY KEY DEFAULT gen_random_uuid()   -- UUID
```
> **Auto-increment** is simpler and faster for a single database; **UUIDs** shine in distributed systems (microservices, sharding — Phase 12/16) where you can't rely on one central counter, and avoid exposing how many records exist. (Recall JPA IDs are often `Long` to allow `null` before insert — Phase 1.2.)

### 3.4 Foreign Key (FK)
A **foreign key** is a column that references the **primary key of another table**, creating a **relationship** and enforcing **referential integrity**.
```sql
CREATE TABLE orders (
    id      BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    total   DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(id)   -- user_id MUST match a real users.id
);
```
> The FK guarantees `orders.user_id` always points to an **existing** `users.id` — you can't have an order for a non-existent user (**referential integrity**, §3.5).

### 3.5 Referential Integrity & cascade actions
The database **enforces** that FK values are valid:
- You **can't insert** an order with a `user_id` that doesn't exist.
- You **can't delete** a user who still has orders (by default) — unless you specify a cascade action:
| ON DELETE action | Effect when the referenced row is deleted |
|------------------|-------------------------------------------|
| `RESTRICT` / `NO ACTION` (default) | **Prevent** the delete (error) |
| `CASCADE` | Delete the dependent rows too |
| `SET NULL` | Set the FK to NULL in dependent rows |
| `SET DEFAULT` | Set the FK to its default value |
```sql
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
```
> ⚠️ `ON DELETE CASCADE` is powerful but dangerous — deleting one user could cascade-delete all their orders. Use deliberately. Referential integrity is a key reason to use a relational DB: the *database* enforces consistency, not just your application code.

### 3.6 Relationship types (cardinality)
| Relationship | Implementation |
|--------------|----------------|
| **One-to-Many** (most common) | FK on the "many" side (orders.user_id → users.id) |
| **One-to-One** | FK + UNIQUE on it (or shared PK) |
| **Many-to-Many** | A **junction/join table** with two FKs (e.g., `student_courses(student_id, course_id)`) |
> Many-to-many needs a third table because a single column can't hold multiple values relationally. (Detailed in Phase 4.3 design/normalization.)

---

## 4. Data Types

Each column has a **data type** that constrains what it can store. Common SQL types:
| Category | Types | Use |
|----------|-------|-----|
| **Integer** | `SMALLINT`, `INTEGER` (`INT`), `BIGINT` | Whole numbers (IDs, counts) |
| **Decimal/exact** | `DECIMAL(p,s)` / `NUMERIC` | **Money & exact values** |
| **Floating point** | `REAL`, `DOUBLE PRECISION` | Scientific/approximate (NOT money) |
| **Text** | `CHAR(n)`, `VARCHAR(n)`, `TEXT` | Strings |
| **Boolean** | `BOOLEAN` | true/false |
| **Date/time** | `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ` | Dates and times |
| **JSON** | `JSON`, `JSONB` (Postgres) | Semi-structured data (Phase 4.6) |
| **UUID** | `UUID` | Globally unique identifiers |
| **Binary** | `BYTEA` / `BLOB` | Binary data |

### 4.1 Key data-type decisions
- **Money → `DECIMAL`/`NUMERIC`, NEVER float** (recall Phase 1.2 — floats are imprecise; `0.1+0.2 ≠ 0.3`). `DECIMAL(10,2)` = up to 10 digits, 2 after the decimal.
- **Text:** `VARCHAR(n)` for bounded strings (with a length limit), `TEXT` for unbounded. (In PostgreSQL there's negligible performance difference — Phase 4.6.)
- **Timestamps:** prefer **`TIMESTAMPTZ`** (timestamp with time zone) to avoid timezone bugs — store everything in UTC.
- **`CHAR(n)`** is fixed-length (padded) — rarely needed; prefer `VARCHAR`/`TEXT`.
> ⚠️ Choosing the right type matters for correctness (money!), storage, and performance. The float-for-money mistake is a classic, costly bug.

### 4.2 NULL — the absence of a value
**NULL** represents **missing/unknown** data — it's *not* zero, empty string, or false.
- `NULL` means "no value here."
- Comparisons with NULL yield **unknown** (not true/false): `NULL = NULL` is **not** true! Use `IS NULL` / `IS NOT NULL`.
- This three-valued logic (true/false/unknown) is a frequent source of bugs in WHERE clauses (Phase 4.2).
> (Recall Phase 1.2/1.5 — NULL in databases maps to `null` in Java, hence wrapper types like `Integer`/`Long` in entities to represent nullable columns.)

---

## 5. Constraints (Rules the Database Enforces)

**Constraints** are rules that the database enforces on data — guaranteeing **data integrity** at the storage level (not just in application code).
| Constraint | Rule |
|------------|------|
| **`NOT NULL`** | The column must have a value |
| **`UNIQUE`** | No two rows can have the same value (e.g., email) |
| **`PRIMARY KEY`** | UNIQUE + NOT NULL (the row identifier) |
| **`FOREIGN KEY`** | Must reference an existing row (referential integrity) |
| **`CHECK`** | A custom condition must hold (e.g., `age >= 0`) |
| **`DEFAULT`** | A value used when none is provided |

```sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    email      VARCHAR(255) NOT NULL UNIQUE,        -- required & unique
    age        INTEGER CHECK (age >= 0),            -- must be non-negative
    status     VARCHAR(20) DEFAULT 'ACTIVE',        -- default value
    created_at TIMESTAMPTZ DEFAULT now()
);
```
> **Constraints enforce integrity at the database level** — even if a bug in your app (or another app) tries to insert bad data, the database rejects it. This is a major advantage: the DB is the **last line of defense** for data correctness (defense in depth — recall Phase 1.5 fail-fast, Phase 15 security). Don't rely on application validation alone.

### 5.1 Why DB-level constraints matter
| Without DB constraints | With DB constraints |
|------------------------|---------------------|
| Bad data can sneak in (app bugs, other clients, scripts) | The DB **guarantees** rules hold |
| Each app must re-implement validation | Enforced once, centrally |
| Data corruption possible | Data integrity protected |

---

## 6. Putting It Together: A Schema Example
```sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    email      VARCHAR(255) NOT NULL UNIQUE,
    name       VARCHAR(100) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE orders (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    total      DECIMAL(10,2) NOT NULL CHECK (total >= 0),
    status     VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMPTZ DEFAULT now(),
    FOREIGN KEY (user_id) REFERENCES users(id)   -- one user has many orders
);
```
This models: users with unique emails, and orders that must belong to a real user, with a non-negative total — all enforced by the database.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using `FLOAT`/`DOUBLE` for money | Use `DECIMAL`/`NUMERIC` |
| Natural keys (email) as the PK | Use a surrogate `id`; enforce email as `UNIQUE` |
| `NULL = NULL` expecting true | Use `IS NULL`; NULL comparisons are "unknown" |
| Relying only on app validation | Add DB constraints (last line of defense) |
| Storing timestamps without timezone | Use `TIMESTAMPTZ`, store UTC |
| Forgetting referential integrity | Use foreign keys |
| Unintended `ON DELETE CASCADE` | Use cascades deliberately |
| Many-to-many in one column | Use a junction table |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **SQL (Phase 4.2)** queries these tables; **schema design/normalization (Phase 4.3)** decides how to split data into tables.
- **JPA/Hibernate (Phase 5.4)** maps tables↔Java classes: `@Entity`→table, `@Id`→PK, `@Column`→column, `@OneToMany`/`@ManyToOne`→FK relationships, `@ManyToMany`→junction table.
- **JDBC (Phase 4.7)** is the low-level API to run SQL from Java.
- **Constraints + referential integrity** keep your data correct regardless of app bugs.
- **Surrogate keys (`Long`/`UUID`)** map to entity IDs (recall Phase 1.2 — `Long` allows null pre-insert).
- **`DECIMAL` for money** is mandatory in e-commerce/finance (Projects 5, 7).
- **Migrations (Phase 8 Flyway)** version these `CREATE TABLE`/`ALTER TABLE` statements.

---

## 9. Quick Self-Check Questions

1. What are tables, rows, columns, and schemas? (And the two meanings of "schema"?)
2. What makes something a primary key, and why prefer surrogate over natural keys?
3. Auto-increment vs UUID — pros, cons, and when to use each?
4. What does a foreign key do, and what is referential integrity?
5. What are the `ON DELETE` cascade options, and why is `CASCADE` risky?
6. How do you model one-to-many vs many-to-many relationships?
7. Why use `DECIMAL` instead of float for money?
8. What does NULL mean, and why is `NULL = NULL` not true?
9. Name the constraint types and why DB-level constraints matter.

---

## 10. Key Terms Glossary

- **Relational database / RDBMS:** data in related tables; managed by a system (PostgreSQL, etc.).
- **Table / row / column / schema:** relation / record / field / namespace-or-structure.
- **Primary key (PK):** unique, not-null row identifier.
- **Natural vs surrogate key:** real-world attribute vs generated id.
- **Auto-increment / UUID:** sequential vs random surrogate keys.
- **Foreign key (FK):** column referencing another table's PK.
- **Referential integrity:** FK values must reference existing rows.
- **Cascade (ON DELETE):** action when a referenced row is deleted.
- **Cardinality:** one-to-one / one-to-many / many-to-many.
- **Junction table:** links two tables for many-to-many.
- **Data type:** the kind of value a column stores.
- **NULL:** absence of a value (three-valued logic).
- **Constraint (NOT NULL/UNIQUE/CHECK/DEFAULT/PK/FK):** DB-enforced data rule.

---

*This is the note for **Section 4.1 — Relational Database Fundamentals**.*
*Next section in roadmap: **4.2 SQL Mastery**.*
