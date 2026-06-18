# Liquibase (Database-Agnostic Migrations & Rollback)

> **Phase 8 — Database Migration → 8.2 Liquibase**
> Goal: Understand **Liquibase** — the changelog (XML/YAML/JSON/SQL), changesets and their identity, the rollback capability, database-agnostic change types, Spring Boot integration, and a clear-eyed **Flyway vs Liquibase** comparison.

---

## 0. The Big Picture

**Liquibase** is the other major Java database-migration tool (alongside Flyway — 8.1). It solves the **same core problem** — version-control your schema, apply changes in order, track them, reproduce any environment (recall 8.1 §1). The key philosophical difference: Liquibase centers on a **changelog** of **changesets** that can be written in a **database-agnostic** format (XML/YAML/JSON), from which Liquibase generates the correct SQL per database — and it has **first-class rollback** support.

```
        changelog (master)                       DATABASECHANGELOG (tracking table)
        ──────────────────                       ─────────────────────────────────
        ├─ changeset alice:1  create users   ──► id=1  author=alice  md5=...  ✅
        ├─ changeset bob:2    add orders      ──► id=2  author=bob    md5=...  ✅
        └─ changeset alice:3  add email index ──► id=3  author=alice  md5=...  ⏳ pending
                  │
   abstract change types  ──►  Liquibase generates dialect-specific SQL  ──►  PostgreSQL / MySQL / Oracle...
```

> Like Flyway, it builds on SQL/DDL (Phase 4.2), schema design (Phase 4.3/4.4), runs in Spring Boot at startup (Phase 5.2), and pairs with `ddl-auto: validate` (Phase 5.4). Everything you learned about *why* migrations matter in 8.1 applies here.

---

## 1. Core Concepts: Changelog & Changeset

| Concept | Meaning |
|---------|---------|
| **Changelog** | The ordered list of all changes. A **master changelog** usually `include`s smaller per-feature changelog files. |
| **Changeset** | A single, atomic unit of change, uniquely identified by **`id` + `author` + file path**. Applied once. |
| **Change type** | An abstract operation (`createTable`, `addColumn`, `createIndex`, `addForeignKeyConstraint`...) that Liquibase translates to dialect-specific SQL. |
| **`DATABASECHANGELOG`** | Tracking table (Liquibase's equivalent of `flyway_schema_history`) — records each applied changeset + an MD5 checksum. |
| **`DATABASECHANGELOGLOCK`** | A lock table so only one process migrates at a time (clustered startups — Phase 10.2). |

> ⭐ A changeset's **identity is `id` + `author` + filename**, not a numeric version. This is more flexible than Flyway's `V<n>` (no version-number collisions on busy teams), but requires discipline (unique ids). Each changeset runs **once**; like Flyway, **never edit an applied changeset** — the MD5 checksum will mismatch and fail validation (recall 8.1 §6).

---

## 2. Changelog Formats (XML / YAML / JSON / SQL)

Liquibase supports four formats for the **same** capabilities. Pick **one** and stay consistent.

### 2.1 XML (the classic, most feature-complete)
```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog" ...>

  <changeSet id="1" author="alice">
    <createTable tableName="users">
      <column name="id" type="bigint" autoIncrement="true">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <column name="email" type="varchar(255)">
        <constraints nullable="false" unique="true"/>
      </column>
    </createTable>
    <rollback>                              <!-- explicit rollback for this changeset -->
      <dropTable tableName="users"/>
    </rollback>
  </changeSet>

</databaseChangeLog>
```

### 2.2 YAML (popular in Spring projects)
```yaml
databaseChangeLog:
  - changeSet:
      id: 2
      author: bob
      changes:
        - addColumn:
            tableName: orders
            columns:
              - column: { name: status, type: varchar(20), defaultValue: PENDING }
      rollback:
        - dropColumn: { tableName: orders, columnName: status }
```

### 2.3 JSON — same structure as YAML, JSON syntax.

### 2.4 Formatted SQL (closest to Flyway — for SQL purists)
```sql
--liquibase formatted sql

--changeset alice:3
CREATE INDEX idx_users_email ON users(email);
--rollback DROP INDEX idx_users_email;
```

| Format | Pros | Cons |
|--------|------|------|
| **XML** | Most complete; great tooling/validation (XSD) | Verbose |
| **YAML/JSON** | Concise; fits Spring config style | Slightly less tooling |
| **Formatted SQL** | Plain SQL, familiar (like Flyway); full control | Loses database-agnosticism / auto-rollback |

> ⭐ The XML/YAML/JSON formats use **abstract change types** → Liquibase generates the right SQL for whichever database you run on (DB-agnostic). The **SQL format** gives up that portability (and most auto-rollback) but is the most direct.

### 2.5 Master changelog (organizing files)
```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - include: { file: db/changelog/changes/001-create-users.yaml }
  - include: { file: db/changelog/changes/002-add-orders.yaml }
  - includeAll: { path: db/changelog/changes/ }   # auto-include a whole folder (sorted)
```

---

## 3. Changesets in Depth

```yaml
- changeSet:
    id: 5
    author: alice
    context: "prod"                 # only run in the "prod" context (env-specific)
    labels: "v2.0"                  # label for selective runs
    comment: "Add audit columns"
    preConditions:                  # run only if conditions hold (else skip/warn/fail)
      - onFail: MARK_RAN
      - not: [ columnExists: { tableName: orders, columnName: created_at } ]
    changes:
      - addColumn:
          tableName: orders
          columns:
            - column: { name: created_at, type: timestamp }
    rollback:
      - dropColumn: { tableName: orders, columnName: created_at }
```

| Attribute | Purpose |
|-----------|---------|
| `id` + `author` | Unique identity (with file path) |
| `context` | Run only in certain environments (`dev`/`test`/`prod`) — like Spring profiles (Phase 5.1) |
| `labels` | Tag changesets for selective deploys (`--labels=v2.0`) |
| `preConditions` | Guard execution (e.g., only add a column if it doesn't exist) |
| `runOnChange` | Re-run when the changeset changes (like Flyway's Repeatable — 8.1 §4.2) |
| `runAlways` | Run every update (rare) |
| `comment` | Documentation |

> ⭐ **`context`/`labels`** are a Liquibase strength: ship one changelog but run different subsets per environment. `preConditions` make changesets defensive/idempotent. `runOnChange` covers the "repeatable view/proc" use case Flyway handles with `R__`.

---

## 4. Rollback ⭐ (Liquibase's signature feature)

Unlike Flyway (where rollback/Undo is a paid feature — 8.1 §7), **Liquibase has built-in rollback** in all editions.

**How rollback is defined:**
1. **Automatic** — for many change types, Liquibase **infers** the rollback (e.g., the rollback of `createTable` is `dropTable`).
2. **Explicit** — you provide a `<rollback>` block (required when it can't be inferred, e.g., `dropColumn`, raw `sql`, data changes).

**Rollback commands:**
```bash
liquibase rollbackCount 1                 # undo the last 1 changeset
liquibase rollbackToDate 2026-06-01       # roll back to a point in time
liquibase rollback v1.0                    # roll back to a tagged state
liquibase tag v2.0                         # create a named tag (a restore point)
liquibase updateSQL                        # PREVIEW the SQL without running it (dry run)
liquibase rollbackSQL v1.0                 # preview rollback SQL
```

| Concept | Note |
|---------|------|
| **Tag** | A named restore point (`liquibase tag v2.0`) you can roll back to |
| **rollbackCount / rollbackToDate / rollback <tag>** | Different ways to choose how far back |
| **Auto vs explicit rollback** | Provide explicit `<rollback>` for destructive/data changes |
| **`updateSQL` / `rollbackSQL`** | Dry-run preview of generated SQL (great for review/DBAs) |

> ⚠️ **Rollback is powerful but not magic.** Rolling back a `dropColumn` cannot recover the **data** that was dropped — only the structure (and only if you defined it). In production, the safer norm is still **fix-forward** (a new changeset), exactly as in Flyway (8.1 §6). Treat rollback as a controlled escape hatch, not a routine. ⭐ The **`updateSQL` dry-run** (review the SQL before it touches prod) is genuinely valuable.

---

## 5. Spring Boot Integration

Liquibase is auto-configured by Spring Boot — add the dependency and point it at the master changelog; it runs on startup before JPA init (just like Flyway).

```xml
<dependency>
  <groupId>org.liquibase</groupId>
  <artifactId>liquibase-core</artifactId>
</dependency>
```
```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
    contexts: prod                  # active contexts (env-specific changesets)
    default-schema: public
  jpa:
    hibernate:
      ddl-auto: validate            # ⭐ Liquibase owns schema; Hibernate validates (Phase 5.4)
```
> ⚠️ **Don't enable both Flyway and Liquibase** in the same app — pick one. Spring Boot will run whichever is on the classpath/configured. Default changelog path is `db/changelog/db.changelog-master.yaml`. Like Flyway, it acquires a **lock** (`DATABASECHANGELOGLOCK`) so clustered startups don't migrate concurrently (consider a pre-deploy Job in K8s — Phase 10.2, recall 8.1 §7).

---

## 6. Other Notable Capabilities

| Feature | What it does |
|---------|--------------|
| **`generateChangeLog`** | Reverse-engineer a changelog from an **existing** database (adoption) |
| **`diff` / `diffChangeLog`** | Compare two databases and generate the changeset to sync them ⭐ |
| **`changelogSync`** | Mark changesets as applied without running (baseline an existing DB) |
| **`history` / `status`** | Show applied / pending changesets (like Flyway `info`) |
| **`validate`** | Verify the changelog is well-formed & checksums match |
| **`dropAll`** | ⚠️☠️ Drop all objects (Liquibase's `clean` equivalent — dangerous in prod) |

> ⭐ **`diff`/`diffChangeLog`** is a distinctive Liquibase superpower: point it at two DBs (or a DB vs your entities) and it **generates** the migration to reconcile them. Useful for adoption and catching drift — but **always review the generated changeset** before trusting it.

---

## 7. Flyway vs Liquibase ⭐ (choosing)

| Dimension | **Flyway** (8.1) | **Liquibase** (this note) |
|-----------|------------------|---------------------------|
| Philosophy | SQL-first, simple, convention | Changelog/changeset, abstracted, feature-rich |
| Migration format | **SQL** (+ Java) | **XML/YAML/JSON/SQL** |
| Database-agnostic changes | ❌ you write dialect SQL | ✅ abstract change types → per-DB SQL |
| **Rollback** | Undo is a **paid** feature | ✅ built-in (auto + explicit) |
| Migration identity | Version number `V<n>__` | `id` + `author` + file |
| Tracking table | `flyway_schema_history` | `DATABASECHANGELOG` |
| Repeatable objects | `R__` migrations | `runOnChange` |
| Conditional/env logic | minimal | `context`, `labels`, `preConditions` |
| Diff/generate from DB | ❌ (no) | ✅ `diff` / `generateChangeLog` |
| Learning curve | **Lower** (just SQL) | Higher (concepts + format) |
| "Feel" | Lightweight & explicit | Comprehensive & flexible |

**When to pick which:**
- **Flyway** ⭐ if: you're a SQL-comfortable team on **one database** (e.g., PostgreSQL), want **simplicity** and plain reviewable SQL. (Most common default in modern Spring projects.)
- **Liquibase** if: you need **database portability**, **built-in rollback**, **conditional/env-specific** changesets, or **diff/auto-generate** from an existing schema — and accept more complexity.

> ⭐ **Both are excellent and battle-tested.** The decision is largely team preference. The cardinal rules are identical (8.1 §6): **never edit applied migrations, keep them backward-compatible (expand/contract), test on the real DB, prefer fix-forward in prod, run as a controlled deploy step.** The *tool* matters far less than the *discipline*.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Editing an applied changeset | Never — add a new one; MD5 mismatch fails validation (§1) |
| Duplicate `id`+`author` in a file | Keep changeset identity unique |
| Enabling both Flyway and Liquibase | Use exactly one (§5) |
| Assuming rollback restores dropped **data** | It restores structure only; prefer fix-forward (§4) |
| Mixing changelog formats randomly | Pick one (XML/YAML/JSON/SQL) and be consistent (§2) |
| Trusting `diffChangeLog` output blindly | Always review generated changesets (§6) |
| `ddl-auto: update` alongside Liquibase | `validate`/`none` — Liquibase owns the schema (§5) |
| Running `dropAll` in prod | Catastrophic — guard it (§6) |
| One giant changelog file | Split via `include`/`includeAll` master changelog (§2.5) |
| Testing only on H2 | Test on the real DB (Testcontainers — Phase 6.5) |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Same role as Flyway (8.1):** owns the schema; pair with `ddl-auto: validate` (Phase 5.2/5.4).
- **Creates the schema/indexes** designed in Phase 4.3/4.4; data changes touch transactions/locking (Phase 4.5).
- **`context`/`labels`** parallel Spring **profiles** (Phase 5.1) for env-specific behavior.
- **Testing** (Phase 6.5/6.6) — run changelogs against **Testcontainers** PostgreSQL in CI.
- **CI/CD & Kubernetes** (Phase 10.2/10.3) — migrations as a pre-deploy Job; expand/contract → zero-downtime (8.1 §6.1).
- **Microservices** (Phase 12.1) — each service migrates its own DB.
- **JDBC** (Phase 4.7) underlies the connection; **security** (Phase 15.3) for PII data changes.
- Choose Flyway **or** Liquibase in **Projects 4, 5, 7**.

---

## 10. Quick Self-Check Questions

1. What are a *changelog* and a *changeset*, and what gives a changeset its identity?
2. What does Liquibase track, and what role does the MD5 checksum play?
3. Name the four changelog formats; what does the SQL format give up?
4. What are abstract change types, and how do they enable database-agnostic migrations?
5. How does Liquibase rollback work (automatic vs explicit), and what are its commands?
6. Why is rollback "not magic" — what can't it recover, and what's the prod norm instead?
7. What do `context`, `labels`, and `preConditions` do on a changeset?
8. How do you integrate Liquibase with Spring Boot, and why not enable Flyway too?
9. What do `diff`/`generateChangeLog` do, and what's the caveat?
10. Give three deciding factors for choosing Flyway vs Liquibase.

---

## 11. Key Terms Glossary

- **Liquibase:** database-migration tool centered on changelogs/changesets with rollback.
- **Changelog / master changelog:** ordered list of changes / root file that includes others.
- **Changeset:** atomic change identified by `id` + `author` + file; runs once.
- **Change type:** abstract operation (`createTable`, `addColumn`...) → dialect SQL.
- **`DATABASECHANGELOG` / `DATABASECHANGELOGLOCK`:** tracking table / lock table.
- **Rollback (auto/explicit) / tag:** undo a change / a named restore point.
- **`context` / `labels` / `preConditions`:** env targeting / selective runs / guards.
- **`runOnChange`:** re-run a changeset when edited (repeatable objects).
- **`diff` / `generateChangeLog`:** compare DBs / reverse-engineer a changelog.
- **`updateSQL` / `rollbackSQL`:** dry-run preview of generated SQL.
- **`ddl-auto: validate`:** Hibernate validates entities against the Liquibase-owned schema.

---

*This is the note for **Section 8.2 — Liquibase**.*
*Previous section in roadmap: **8.1 Flyway**.*
*This completes **Phase 8 — Database Migration**.*
*Next: **Phase 9 — Logging & Monitoring**.*
