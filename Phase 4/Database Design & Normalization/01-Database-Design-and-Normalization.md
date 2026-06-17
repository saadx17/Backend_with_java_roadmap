# Database Design & Normalization

> **Phase 4 — Databases → 4.3 Database Design & Normalization**
> Goal: Master how to design good relational schemas — ER modeling, the normal forms (1NF–BCNF), when to denormalize, junction tables, and naming/auditing conventions.

---

## 0. The Big Picture

**Database design** is deciding **how to structure your data into tables** — which tables exist, what columns they have, and how they relate. Good design (via **normalization**) eliminates redundancy and update anomalies; pragmatic design knows when to **denormalize** for performance.

```
Bad design: one giant table with repeated/redundant data -> anomalies, bugs
Good design: data split into related tables (normalized) -> consistent, flexible
```

> Design decisions are hard to change once you have production data, so getting the schema right early matters. This builds on Phase 4.1 (keys, constraints, relationships) and informs everything downstream (queries, JPA entities, performance).

---

## 1. ER Modeling (Entity-Relationship)

Before writing SQL, model your domain as **entities** and **relationships** — the conceptual blueprint.
| Concept | Meaning | Maps to |
|---------|---------|---------|
| **Entity** | A "thing" you store (User, Order, Product) | A **table** |
| **Attribute** | A property of an entity (name, price) | A **column** |
| **Relationship** | How entities relate (a User *places* Orders) | **Foreign keys** / junction tables |
| **Cardinality** | How many relate to how many | 1:1, 1:many, many:many |

### 1.1 Cardinality (recall Phase 4.1)
```
User  --places-->  Order        (one User has many Orders -> 1:many)
Order --contains--> Products     (many Orders contain many Products -> many:many)
User  --has-->     Profile       (one User has one Profile -> 1:1)
```
| Cardinality | Implementation |
|-------------|----------------|
| **One-to-Many** (most common) | FK on the "many" side (`orders.user_id`) |
| **One-to-One** | FK + UNIQUE, or shared PK |
| **Many-to-Many** | A **junction table** (§5) |

### 1.2 The design process
```
1. Identify ENTITIES (the nouns: User, Order, Product, Category).
2. Identify ATTRIBUTES (what each entity stores).
3. Identify RELATIONSHIPS and their cardinality.
4. Choose primary keys (surrogate ids — Phase 4.1).
5. Normalize (eliminate redundancy — §2).
6. Denormalize selectively if performance requires (§4).
```
> An **ER diagram** (boxes = entities, lines = relationships) visualizes this. Tools: dbdiagram.io, draw.io, or your IDE. Model first, then translate to `CREATE TABLE`.

---

## 2. Normalization — The Problem It Solves

**Normalization** is organizing data to **eliminate redundancy** and **prevent anomalies**. The core idea: **store each fact once, in one place.**

### 2.1 The anomalies of a bad (unnormalized) design
Consider one giant `orders` table that repeats customer info on every row:
```
orders (BAD — redundant):
+----+-----------+-------------+--------------+---------+
| id | cust_name | cust_email  | product      | price   |
+----+-----------+-------------+--------------+---------+
| 1  | Alice     | alice@x.com | Laptop       | 1000    |
| 2  | Alice     | alice@x.com | Mouse        | 25      |   <- Alice's info DUPLICATED
| 3  | Bob       | bob@x.com   | Laptop       | 1000    |
+----+-----------+-------------+--------------+---------+
```
| Anomaly | Problem |
|---------|---------|
| **Update anomaly** | Alice changes email → must update *every* row (miss one → inconsistent data) |
| **Insertion anomaly** | Can't add a customer until they place an order (no row to put them in) |
| **Deletion anomaly** | Delete Alice's only order → lose her info entirely |
| **Redundancy** | Same data stored many times → wasted space, inconsistency risk |
> Normalization fixes these by **splitting** data into separate tables (customers, products, orders) linked by foreign keys — each fact stored once.

---

## 3. The Normal Forms

Normalization proceeds through **normal forms** — each a stricter level of organization. You normalize *up* to a form. The main ones (1NF → 2NF → 3NF → BCNF):

### 3.1 First Normal Form (1NF) — atomic values, no repeating groups
> **1NF: each column holds a single (atomic) value; no repeating groups or multi-valued columns.**
```
NOT 1NF (multi-valued column):
| id | name  | phones              |
| 1  | Alice | "555-1234, 555-5678"|   <- multiple values in one column!

1NF (atomic — separate rows or a related table):
phones table:
| user_id | phone    |
| 1       | 555-1234 |
| 1       | 555-5678 |
```
- No comma-separated lists, no arrays-as-strings, no `phone1`/`phone2`/`phone3` columns.
- Each cell = one value; each row = one record with a key.

### 3.2 Second Normal Form (2NF) — no partial dependencies
> **2NF: be in 1NF + every non-key column depends on the *whole* primary key (only matters for composite PKs).**
```
NOT 2NF (composite PK: order_id + product_id; product_name depends only on product_id):
order_items:
| order_id | product_id | quantity | product_name |   <- product_name depends on product_id ONLY
                                                          (partial dependency!)

2NF (move product_name to a products table):
order_items: | order_id | product_id | quantity |
products:    | product_id | product_name | price |
```
- A **partial dependency** = a non-key column depends on only *part* of a composite key.
- Fix: move partially-dependent columns to their own table.

### 3.3 Third Normal Form (3NF) — no transitive dependencies
> **3NF: be in 2NF + non-key columns depend *only* on the key, not on other non-key columns (no transitive dependencies).**
```
NOT 3NF (department_name depends on department_id, not on the employee PK):
employees:
| emp_id | name  | department_id | department_name |   <- dept_name depends on dept_id (transitive!)

3NF (move department info to its own table):
employees:   | emp_id | name | department_id |
departments: | department_id | department_name |
```
- A **transitive dependency** = a non-key column depends on *another non-key column* (which depends on the key).
- Fix: move the dependent columns to a separate table keyed by the intermediate column.
> ⭐ **3NF is the practical target for most applications.** It eliminates most redundancy while keeping the schema usable. "Normalized" usually means 3NF.

### 3.4 Boyce-Codd Normal Form (BCNF) — a stricter 3NF
> **BCNF: a stricter 3NF — every determinant must be a candidate key.** Handles edge cases 3NF misses (multiple overlapping candidate keys). Rarely needed beyond 3NF in practice.

### 3.5 Higher forms (4NF, 5NF — awareness)
4NF (multi-valued dependencies) and 5NF (join dependencies) exist but are rarely relevant — **3NF/BCNF covers virtually all real designs.**

### 3.6 Quick summary
| Form | Rule (informal) |
|------|-----------------|
| **1NF** | Atomic values, no repeating groups |
| **2NF** | 1NF + no partial dependencies (whole-key dependence) |
| **3NF** | 2NF + no transitive dependencies (key-only dependence) |
| **BCNF** | 3NF + every determinant is a candidate key |
> Mnemonic: *"Every non-key column depends on **the key, the whole key, and nothing but the key**, so help me Codd."* (1NF = a value, 2NF = whole key, 3NF = nothing but the key.)

---

## 4. Denormalization (When to Break the Rules)

**Denormalization** deliberately introduces **redundancy** to improve **read performance** — the opposite of normalization. It's a trade-off, not a mistake.

### 4.1 The trade-off
| Normalized | Denormalized |
|------------|--------------|
| No redundancy, consistent | Redundant data (must keep in sync) |
| Slower reads (more joins) | **Faster reads** (fewer/no joins) |
| Faster, safer writes | Slower writes (update multiple copies) |
| Default for OLTP (transactional apps) | Common in OLAP/reporting/read-heavy systems |

### 4.2 When to denormalize
| Scenario | Why |
|----------|-----|
| **Read-heavy** workload where joins are too slow | Trade write complexity for read speed |
| **Reporting / analytics** (OLAP) | Pre-aggregated/wide tables avoid expensive joins |
| **Caching computed values** | Store a `comment_count` instead of `COUNT(*)` every read |
| **Avoiding expensive joins at scale** | Duplicate a frequently-needed column |
```sql
-- Denormalized: store a cached count to avoid COUNT(*) on every read
ALTER TABLE posts ADD COLUMN comment_count INTEGER DEFAULT 0;
-- (must keep it in sync when comments are added/removed — e.g., via triggers or app code)
```
> ⚠️ **Normalize first, denormalize only when measured performance demands it.** Premature denormalization adds redundancy and bug risk (sync issues) without proven benefit. The default is **3NF**; denormalize surgically with evidence (recall Phase 2.1 — measure before optimizing). Keeping denormalized data in sync (triggers, app logic, or batch jobs) is the cost.

### 4.3 OLTP vs OLAP (context)
| | **OLTP** (transactional) | **OLAP** (analytical) |
|---|--------------------------|------------------------|
| Workload | Many small reads/writes | Few large analytical queries |
| Design | **Normalized** (3NF) | Often **denormalized** (star/snowflake schemas) |
| Example | Your app's main database | Data warehouse / reporting DB |
> Your backend's primary database is typically **OLTP → normalized**. Reporting/analytics may use a separate **denormalized** warehouse (Phase 16).

---

## 5. Junction Tables (Many-to-Many)

A **many-to-many** relationship can't be a single FK — it needs a **junction table** (a.k.a. join/bridge/associative table) with FKs to both sides (recall Phase 4.1):
```
students <-> courses  (a student takes many courses; a course has many students)

student_courses (junction table):
+------------+-----------+-------------+
| student_id | course_id | enrolled_at |   <- composite PK (student_id, course_id)
+------------+-----------+-------------+
| 1          | 101       | 2026-01-15  |
| 1          | 102       | 2026-01-16  |
| 2          | 101       | 2026-01-17  |
+------------+-----------+-------------+
```
```sql
CREATE TABLE student_courses (
    student_id BIGINT NOT NULL REFERENCES students(id),
    course_id  BIGINT NOT NULL REFERENCES courses(id),
    enrolled_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (student_id, course_id)      -- composite PK prevents duplicate enrollments
);
```
- The junction table has **two FKs** (one to each table).
- Its **composite PK** (both FKs) prevents duplicate pairings.
- It can carry **extra attributes** about the relationship (`enrolled_at`, `grade`).
> ⚠️ A many-to-many *always* needs a junction table — you can't model it with a single column. This maps to JPA's `@ManyToMany` with a `@JoinTable` (Phase 5.4); if the junction has extra columns, model it as its own entity.

---

## 6. Naming Conventions

Consistent naming makes schemas readable and maintainable:
| Convention | Recommendation |
|------------|----------------|
| **Table names** | Lowercase, often **plural** (`users`, `orders`) — or singular; be consistent |
| **Column names** | `snake_case` (`created_at`, `user_id`) |
| **Primary key** | `id` |
| **Foreign keys** | `<referenced_table_singular>_id` (`user_id`, `product_id`) |
| **Junction tables** | Combine both names (`student_courses`, `order_items`) |
| **Booleans** | `is_active`, `has_paid` (clear true/false intent) |
| **Indexes** | `idx_<table>_<columns>` (`idx_users_email`) |
> ⚠️ **Avoid reserved words** (`user`, `order`, `group` are SQL keywords — `order` especially!). If unavoidable, quote them (`"order"`) — but better to pluralize (`orders`) or rename. Consistency matters more than the specific choice; pick conventions and stick to them.

---

## 7. Standard Columns: Audit, Soft Delete, Versioning (recall Phase 4.2.1)

Most well-designed tables include standard "housekeeping" columns:
```sql
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),   -- AUDIT: creation time
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),   -- AUDIT: last modification
    deleted_at  TIMESTAMPTZ,                           -- SOFT DELETE (NULL = active)
    version     INTEGER NOT NULL DEFAULT 0             -- VERSIONING: optimistic locking
);
```
| Column | Purpose | Spring mapping |
|--------|---------|----------------|
| `created_at` / `updated_at` | **Audit columns** — who/when (track changes) | `@CreatedDate` / `@LastModifiedDate` (Phase 5.4) |
| `deleted_at` / `is_deleted` | **Soft delete** — mark instead of removing (Phase 4.2.1) | `@SQLDelete` / `@Where` |
| `version` | **Versioning** — optimistic locking (detect concurrent edits, Phase 4.5) | `@Version` |
> These support **auditing** (who changed what, when), **recovery** (soft delete), and **concurrency control** (versioning — Phase 4.5). Spring Data JPA can auto-populate them (Phase 5.4 auditing).

---

## 8. Design Best-Practices Checklist
```
[ ] Model entities & relationships (ER) before coding
[ ] Use surrogate primary keys (id) — Phase 4.1
[ ] Normalize to 3NF (eliminate redundancy)
[ ] Use foreign keys for referential integrity (Phase 4.1)
[ ] Junction tables for many-to-many
[ ] DECIMAL for money, TIMESTAMPTZ for time (Phase 4.1)
[ ] Add constraints (NOT NULL, UNIQUE, CHECK) at the DB level (Phase 4.1)
[ ] Consistent naming (snake_case, plural tables, *_id FKs)
[ ] Audit columns (created_at/updated_at), soft delete, version where useful
[ ] Index foreign keys & frequently-queried columns (Phase 4.4)
[ ] Denormalize only when measured performance requires it
```

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| One giant table (no normalization) | Split into related tables (3NF) |
| Multi-valued columns (CSV in a cell) | Atomic values (1NF) — separate rows/table |
| Premature denormalization | Normalize first; denormalize with evidence |
| Many-to-many without a junction table | Always use a junction table |
| Reserved words as names (`order`, `user`) | Pluralize/rename |
| Inconsistent naming | Pick conventions and stick to them |
| No audit/versioning columns | Add `created_at`/`updated_at`/`version` |
| Forgetting to index FKs | Index them (joins/lookups — Phase 4.4) |
| Over-normalizing read-heavy reporting | Consider denormalization/OLAP |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **JPA entities (Phase 5.4)** map directly to this design: `@Entity`→table, relationships→`@OneToMany`/`@ManyToOne`/`@ManyToMany`, junction tables→`@JoinTable`.
- **Normalization** keeps your domain model clean; **denormalization** decisions affect query/caching strategy (Phase 5.7, 13).
- **Audit/version columns** → Spring Data auditing (`@CreatedDate`) and optimistic locking (`@Version`, Phase 4.5/5.4).
- **Indexing FKs** (Phase 4.4) makes the joins your relationships generate fast.
- **Migrations (Phase 8 Flyway)** version your schema's evolution.
- **DDD (Phase 12)**: entities/value objects/aggregates relate to ER modeling and table design.
- **Projects 4/5** require normalized schemas (tasks/projects, products/orders/categories).

---

## 11. Quick Self-Check Questions

1. What are entities, attributes, relationships, and cardinality in ER modeling?
2. What anomalies does normalization prevent (give the four)?
3. State the rules for 1NF, 2NF, and 3NF. Which is the practical target?
4. What's a partial vs a transitive dependency?
5. What is denormalization, when is it justified, and what's the cost?
6. What's the difference between OLTP and OLAP design?
7. How do you model a many-to-many relationship, and why?
8. Name good naming conventions and the reserved-word pitfall.
9. What standard housekeeping columns should tables have, and what do they support?

---

## 12. Key Terms Glossary

- **Database design:** structuring data into tables and relationships.
- **ER modeling:** entities, attributes, relationships, cardinality.
- **Normalization:** organizing data to eliminate redundancy/anomalies.
- **Update/insertion/deletion anomalies:** problems from redundant data.
- **1NF / 2NF / 3NF / BCNF:** progressive normal forms.
- **Partial / transitive dependency:** depends on part of the key / on a non-key column.
- **Denormalization:** adding redundancy for read performance.
- **OLTP / OLAP:** transactional (normalized) vs analytical (denormalized).
- **Junction (join/bridge) table:** implements many-to-many.
- **Composite primary key:** a PK spanning multiple columns.
- **Audit columns / soft delete / version:** housekeeping columns.
- **Naming conventions:** snake_case, plural tables, `*_id` FKs.

---

*This is the note for **Section 4.3 — Database Design & Normalization**.*
*Next section in roadmap: **4.4 Indexing**.*
