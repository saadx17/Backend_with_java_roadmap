# Flyway (Versioned Database Migrations)

> **Phase 8 — Database Migration → 8.1 Flyway**
> Goal: Understand *why* database migrations exist and master **Flyway** — migration naming (`V1__...`), versioned vs repeatable migrations, Spring Boot integration, best practices (never modify applied scripts, backward-compatible changes, data migrations), the CLI commands (`migrate`/`clean`/`info`/`validate`/`repair`), and Java-based migrations.

---

## 0. The Big Picture

Your application code lives in **version control** (Git — Phase 0.4) and evolves through commits. But your **database schema** also evolves — new tables, columns, indexes (Phase 4.3/4.4). How do you keep every environment (your laptop, a teammate's, CI, staging, production) on the **same schema version**, reproducibly? Answer: **database migrations** — version-controlled, ordered SQL scripts that are applied automatically and tracked.

```
   Git history of CODE              Flyway history of SCHEMA
   ─────────────────               ────────────────────────
   commit a1b2  (feature)          V1__create_users.sql      ✅ applied
   commit c3d4  (bugfix)           V2__add_orders.sql        ✅ applied
   commit e5f6  (feature)          V3__add_email_index.sql   ✅ applied
                                   V4__add_status_column.sql ⏳ pending  ── runs on next startup
```

> **Flyway is "version control for your database."** It builds on SQL/DDL (Phase 4.2), schema design (Phase 4.3), and runs inside your Spring Boot app at startup (Phase 5.2). It's the disciplined alternative to Hibernate's `ddl-auto` (Phase 5.4 — never use `update`/`create` in production!).

### Why not just `spring.jpa.hibernate.ddl-auto`?
| `ddl-auto` (Hibernate) | Flyway migrations |
|------------------------|-------------------|
| Auto-derives schema from entities | Explicit, hand-written SQL |
| ⚠️ Unpredictable; can drop/alter data | Deterministic, reviewed in PRs |
| No history / no rollback awareness | Tracked, versioned, auditable |
| Fine for quick prototyping only | ✅ Production standard |
> ⭐ **Rule: use `ddl-auto: validate` (or `none`) + Flyway in real projects.** Let Flyway own the schema; let Hibernate only *validate* that entities match it. Never let Hibernate mutate a production schema.

---

## 1. What Problem Migrations Solve

| Problem | Without migrations | With Flyway |
|---------|-------------------|-------------|
| Schema drift across environments | "Works on my machine" | Same scripts → identical schema everywhere |
| Reproducing the DB from scratch | Manual SQL, ad-hoc | `migrate` rebuilds it deterministically |
| Team collaboration | Conflicting manual changes | Ordered, reviewed scripts in Git |
| Auditability | "Who added that column?" | Every change is a committed, dated file |
| CI/CD automation (Phase 10.3) | Manual DBA steps | Migrations run automatically on deploy |
| Rolling back / forward | Risky manual edits | Forward-only, controlled scripts |

> ⭐ Migrations make your schema **reproducible, reviewable, and automatable** — a hard requirement for CI/CD (Phase 10.3) and microservices where each service owns its DB (Phase 12.1).

---

## 2. How Flyway Works (the tracking table)

Flyway keeps a metadata table — **`flyway_schema_history`** — in your database recording every migration applied, in order, with a **checksum**.

```
flyway_schema_history
┌─────────────┬─────────┬──────────────────────┬──────────┬────────────┬─────────┐
│ installed_  │ version │ description          │ checksum │ installed_ │ success │
│ rank        │         │                      │          │ on         │         │
├─────────────┼─────────┼──────────────────────┼──────────┼────────────┼─────────┤
│ 1           │ 1       │ create users         │ -1289... │ 2026-06-01 │ true    │
│ 2           │ 2       │ add orders           │  9921... │ 2026-06-02 │ true    │
│ 3           │ 3       │ add email index      │  4471... │ 2026-06-10 │ true    │
└─────────────┴─────────┴──────────────────────┴──────────┴────────────┴─────────┘
```

The startup algorithm:
```
1. Scan the migrations location (default: classpath:db/migration) for scripts.
2. Read flyway_schema_history to see what's already applied.
3. Compute the set of PENDING migrations (version > current).
4. Apply them IN ORDER, each in a transaction (where the DB supports DDL transactions).
5. Record each one (version, checksum, success) in the history table.
6. VALIDATE: re-checksum applied scripts → if a file changed, FAIL (⚠️ see §6).
```
> ⭐ The **checksum** is how Flyway detects that an already-applied script was edited — a cardinal sin (§6). Each migration runs exactly **once** and is **immutable** afterward.

---

## 3. Migration Naming Convention

Flyway parses meaning from the **filename** — get this right or it won't run.

```
 V 2 __ add_orders_table .sql
 │ │  │  │                  │
 │ │  │  │                  └─ suffix (.sql)
 │ │  │  └─ description (underscores → spaces in history)
 │ │  └─ TWO underscores (separator) — required!
 │ └─ version (1, 2, 2.1, 20260616__... timestamp style, etc.)
 └─ prefix: V = Versioned, R = Repeatable, U = Undo
```

| Prefix | Type | Runs |
|--------|------|------|
| **`V`** | **Versioned** | Once, in version order (the workhorse) |
| **`R`** | **Repeatable** | Whenever its **checksum changes** (after all versioned, by description order) |
| **`U`** | **Undo** | Manual rollback of a versioned migration (⚠️ paid/Teams feature) |

Examples:
```
V1__create_users_table.sql
V2__add_orders_table.sql
V2.1__add_order_index.sql            # dotted minor version, runs after V2
V20260616120000__add_audit_columns.sql   # timestamp-based version (avoids merge collisions)
R__user_summary_view.sql             # repeatable: a view re-created when edited
```
> ⚠️ Common naming mistakes: **single** underscore (`V2_add.sql` ❌ — needs `__`), missing the `V`, or **duplicate version numbers** (two `V2__` files → Flyway errors). ⭐ On busy teams, **timestamp-based versions** (`V20260616...`) avoid two PRs both grabbing `V5__`.

---

## 4. Versioned vs Repeatable Migrations

### 4.1 Versioned (`V`) — schema changes
The normal case: each is applied **exactly once**, in order. DDL (create/alter/drop), reference-data inserts, one-off data migrations.
```sql
-- V4__add_status_to_orders.sql
ALTER TABLE orders ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'PENDING';
CREATE INDEX idx_orders_status ON orders(status);   -- (Phase 4.4 indexing)
```

### 4.2 Repeatable (`R`) — re-runnable objects
For objects you want to **redefine** when they change: **views, stored procedures, functions, triggers**. Flyway re-applies an `R__` script **whenever its checksum changes** (i.e., you edited it), always *after* all versioned migrations.
```sql
-- R__active_users_view.sql   (no version number)
CREATE OR REPLACE VIEW active_users AS
SELECT id, email FROM users WHERE deleted_at IS NULL;
```
| | Versioned `V` | Repeatable `R` |
|--|--------------|----------------|
| Version number | required | none |
| Runs | once | whenever checksum changes |
| Order | by version | after all `V`, by description |
| Best for | DDL, data migrations | views, procs, functions, triggers |
| Must be idempotent? | no | ✅ yes (use `CREATE OR REPLACE`) |

> ⭐ Repeatable migrations keep re-creatable objects (a view's definition) editable in **one file** in Git, instead of a chain of `ALTER`s. Make them **idempotent** (`CREATE OR REPLACE`, `DROP ... IF EXISTS`).

---

## 5. Spring Boot Integration

Flyway is **first-class in Spring Boot** — add the dependency and it runs **automatically on startup**, before JPA/Hibernate initializes.

```xml
<!-- Maven (Phase 3) -->
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
</dependency>
<!-- For PostgreSQL 15+/16 you also need the DB-specific module: -->
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

Place scripts in **`src/main/resources/db/migration/`** (the default classpath location). Configure in `application.yml` (Phase 5.2):
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false      # true only when adopting Flyway on an EXISTING db (§8)
    validate-on-migrate: true       # re-checksum applied scripts (default true)
    out-of-order: false             # reject migrations with a version lower than current
  jpa:
    hibernate:
      ddl-auto: validate            # ⭐ Flyway owns schema; Hibernate only validates (Phase 5.4)
```
- On startup, Spring Boot runs `flyway.migrate()` **before** the JPA `EntityManagerFactory` is created → Hibernate then `validate`s entities against the migrated schema.
- ⚠️ In a clustered deploy (many instances starting at once), Flyway takes a **lock** so only one instance migrates — but be deliberate (§7 talks about running migrations as a separate deploy step in K8s — Phase 10.2).

---

## 6. Best Practices ⭐ (the rules that matter)

| Rule | Why |
|------|-----|
| **Never modify an already-applied migration** | Flyway checksums it → editing breaks `validate` on every env that ran it (§2). Add a **new** migration instead. |
| **Never delete an applied migration** | History references it; new envs replay the full chain. |
| **One logical change per migration** | Easier review (Phase 0.4 PRs) & debugging. |
| **Make changes backward-compatible (expand/contract)** | Old app code must keep working during a rolling deploy (§6.1). |
| **Separate schema vs large data migrations** | Big `UPDATE`s can lock tables / time out — do them carefully/batched. |
| **Test migrations in CI** (e.g., Testcontainers — Phase 6.5) | Catch failures before production. |
| **Keep migrations forward-only in prod** | Rolling back schema is dangerous; fix forward with a new migration. |
| **Use the real DB (PostgreSQL), not H2, to test** | Dialect differences (Phase 4.6/6.6). |
| **Review migrations like code** | A bad `ALTER` can lock a table or lose data. |

### 6.1 Backward-compatible changes: the **Expand/Contract** pattern
A rolling deploy runs **old and new code simultaneously** for a while. A destructive schema change (rename/drop a column) breaks the old code. Do it in phases across releases:
```
Renaming `name` → `full_name` safely:
 Release 1 (EXPAND):   add full_name; app writes BOTH name and full_name (backfill old rows)
 Release 2 (MIGRATE):  app reads/writes full_name only; name now unused
 Release 3 (CONTRACT): drop the old name column   (a separate migration, after no code uses it)
```
> ⭐ This **expand → migrate → contract** dance is essential for **zero-downtime deploys** (Phase 10.2 rolling updates, Phase 12). Never rename/drop a column in the same release that stops using it.

### 6.2 Data migrations
- Reference/seed data: a versioned `V` script with `INSERT`s (idempotent where possible).
- Large transformations: batch them (`UPDATE ... WHERE id BETWEEN`), beware long locks (Phase 4.5 transactions/locking), consider doing them out-of-band rather than at app startup.

---

## 7. Flyway Commands (CLI / Maven / Gradle)

Beyond auto-migrate on startup, Flyway has commands (CLI, or `flyway-maven-plugin`/Gradle, or programmatic API).

| Command | What it does |
|---------|--------------|
| **`migrate`** | Apply all pending migrations (the default action) |
| **`info`** | Show status of each migration (applied / pending / failed) |
| **`validate`** | Verify applied migrations' checksums match the files (CI gate) |
| **`baseline`** | Mark an existing DB at a version → start using Flyway on a legacy DB (§8) |
| **`repair`** | Fix the history table: remove failed entries, **re-align checksums** after a *legitimate* change |
| **`clean`** | ⚠️☠️ **DROP everything** in the schema — wipes all objects |
| **`undo`** | Revert the last versioned migration (Teams/paid feature) |

```bash
flyway -url=jdbc:postgresql://localhost:5432/app -user=app -password=*** info
flyway migrate
flyway validate
```

> ⚠️☠️ **`flyway clean` deletes ALL database objects.** It is catastrophic in production. **Set `spring.flyway.clean-disabled=true` (the default since Flyway 9)** so it can never be triggered in prod. ⚠️ **`repair`** is a sharp tool — only use it to re-checksum after an *intentional* fix or to clear a *failed* migration entry; never to paper over an edited-applied-script mistake.

> ⭐ In CI/CD (Phase 10.3) and Kubernetes (Phase 10.2), a common pattern is running migrations as a **separate step/Job** *before* rolling out the new app pods — not relying on every replica auto-migrating at startup.

---

## 8. Adopting Flyway on an Existing Database (Baseline)

If you already have a populated production DB with no `flyway_schema_history`:
```yaml
spring:
  flyway:
    baseline-on-migrate: true     # create the history table and mark the baseline
    baseline-version: 1           # treat the current schema as version 1
```
- `baseline` tells Flyway "the schema is already at version N; only apply migrations **above** N."
- Capture the current schema as `V1__baseline.sql` for fresh environments, then continue with `V2__...`.

---

## 9. Java-Based Migrations

For migrations that **plain SQL can't express** (complex data transformation, calling app logic, conditional logic), write a Java migration implementing `BaseJavaMigration`.
```java
package db.migration;   // ⚠️ must be on the classpath under the migration package

public class V5__encrypt_existing_phone_numbers extends BaseJavaMigration {  // naming like SQL
    @Override
    public void migrate(Context context) throws Exception {
        try (var stmt = context.getConnection().createStatement();
             var rs = stmt.executeQuery("SELECT id, phone FROM users WHERE phone IS NOT NULL")) {
            while (rs.next()) {
                long id = rs.getLong("id");
                String encrypted = encrypt(rs.getString("phone"));   // app/crypto logic (Phase 15.3)
                try (var up = context.getConnection()
                        .prepareStatement("UPDATE users SET phone = ? WHERE id = ?")) {
                    up.setString(1, encrypted); up.setLong(2, id); up.executeUpdate();
                }
            }
        }
    }
}
```
| Java migration | Note |
|----------------|------|
| Class name | Same convention: `V5__description` / `R__description` |
| Extends | `BaseJavaMigration` (`migrate(Context)`) |
| Use for | Complex data transforms, encryption, calling services, conditional logic |
| Access | Raw JDBC `Connection` (Phase 4.7) via the `Context` |
| ⚠️ Manage your own transaction semantics | and it's checksummed like SQL |

> ⭐ Prefer **SQL migrations** for schema (clear, reviewable). Reach for **Java migrations** only when you genuinely need procedural logic. Keep heavy data work batched and idempotent.

---

## 10. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Editing an already-applied migration | Never — add a new one; checksum mismatch fails `validate` (§6) |
| Single underscore `V2_x.sql` | Needs double `__` after the version (§3) |
| Duplicate version numbers | Unique versions; use timestamps to avoid collisions (§3) |
| Letting `ddl-auto: update` manage prod schema | `validate`/`none` + Flyway owns schema (§0/§5) |
| Running `flyway clean` in prod | `clean-disabled=true` (default) (§7) |
| Rename/drop column in same release that stops using it | Expand/contract across releases (§6.1) |
| Testing migrations only on H2 | Test on real PostgreSQL (Testcontainers — Phase 6.5) |
| Huge data `UPDATE` at startup locks tables | Batch it / run out-of-band (§6.2, Phase 4.5) |
| Relying on every replica to auto-migrate | Run migrations as a pre-deploy Job (§7, Phase 10.2) |
| Using `repair` to hide an edited script | Only for failed entries / intentional re-checksum (§7) |
| Forgetting the DB-specific module (PG 15+) | Add `flyway-database-postgresql` (§5) |

---

## 11. Connection to Backend / Spring (Why This Matters Later)

- **Spring Boot auto-runs Flyway** before JPA init; pair with `ddl-auto: validate` (Phase 5.2/5.4).
- **Schema/index design** (Phase 4.3/4.4) is what your migrations create; **transactions/locking** (Phase 4.5) matter for data migrations.
- **JDBC** (Phase 4.7) underlies Java migrations.
- **Testing** (Phase 6.5/6.6) — run migrations against **Testcontainers** PostgreSQL in CI.
- **CI/CD** (Phase 10.3) & **Kubernetes** (Phase 10.2) — migrations as a deploy step/Job; expand/contract enables **zero-downtime rolling deploys**.
- **Microservices** (Phase 12.1) — each service owns & migrates its own database.
- **Security** (Phase 15.3) — data migrations may encrypt/mask PII.
- **Liquibase (8.2)** is the alternative tool — compared head-to-head there.
- Used in **Projects 4, 5, 7** (REST API & E-Commerce backends).

---

## 12. Quick Self-Check Questions

1. What problem do database migrations solve, and why not rely on Hibernate `ddl-auto`?
2. What is `flyway_schema_history`, and what role does the **checksum** play?
3. Decode the filename `V2.1__add_order_index.sql` — every part.
4. Versioned vs Repeatable migrations — when use each? Why must `R` be idempotent?
5. Where do scripts live in Spring Boot, and when does Flyway run relative to JPA?
6. Why must you never edit an applied migration? What do you do instead?
7. Explain the expand/contract pattern for renaming a column with zero downtime.
8. What does each command do: `migrate`, `info`, `validate`, `baseline`, `repair`, `clean`?
9. Why is `flyway clean` dangerous, and how is it disabled by default?
10. When would you write a Java-based migration instead of SQL?

---

## 13. Key Terms Glossary

- **Database migration:** a version-controlled, ordered, tracked schema/data change script.
- **Flyway:** migration tool that applies & tracks SQL/Java migrations.
- **`flyway_schema_history`:** the table recording applied migrations + checksums.
- **Versioned (`V`) migration:** runs once, in version order.
- **Repeatable (`R`) migration:** re-runs whenever its checksum changes (views/procs).
- **Checksum:** hash of a migration used to detect post-apply edits.
- **`migrate` / `info` / `validate` / `baseline` / `repair` / `clean`:** Flyway commands.
- **Baseline:** marking an existing DB at a starting version when adopting Flyway.
- **Expand/contract:** multi-release backward-compatible schema change pattern.
- **Data migration:** a migration that transforms existing rows (not just schema).
- **Java migration:** `BaseJavaMigration` for procedural migration logic.
- **`ddl-auto: validate`:** Hibernate verifies entities match the Flyway-owned schema.

---

*This is the note for **Section 8.1 — Flyway**.*
*Previous section in roadmap: **7.3 DTO Pattern and MapStruct**.*
*Next section in roadmap: **8.2 Liquibase**.*
