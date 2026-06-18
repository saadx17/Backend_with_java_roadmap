# Spring Data JPA: Entities, Relationships & Persistence Context

> **Phase 5 — Spring Framework & Spring Boot → 5.4 Spring Data JPA**
> Goal: Master JPA entity mapping, relationship mapping (and the N+1 problem), the entity lifecycle, and the persistence context.

---

## 0. The Big Picture

**JPA (Jakarta Persistence API)** is a **specification** for object-relational mapping (ORM); **Hibernate** is the most common **implementation**. **Spring Data JPA** sits on top, removing boilerplate. ORM maps **Java objects ↔ database tables** (recall Phase 4.1/4.3 — entities map to tables) so you work with objects instead of writing SQL/JDBC (Phase 4.7) by hand.

```
Java object (User entity)  <-- JPA/Hibernate ORM -->  database row (users table)
You call repository.save(user);  Hibernate generates the INSERT/UPDATE.
```

> JPA is the standard data layer for Spring apps. It builds on everything in Phase 4 (tables, keys, relationships, transactions, JDBC) and Phase 5.1 (beans/DI). This note covers entities, relationships, and the persistence model; the next covers repositories & queries.

---

## 1. JPA, Hibernate & Spring Data JPA (the layers)
```
Spring Data JPA  -> removes repository boilerplate (you declare interfaces)
      |  built on
JPA (specification: @Entity, EntityManager, JPQL)
      |  implemented by
Hibernate (the ORM engine that generates SQL)
      |  uses
JDBC (Phase 4.7) -> the database
```
> "Program to the spec" (recall Phase 1.3): you code against **JPA** annotations/interfaces; **Hibernate** is the swappable implementation; **Spring Data JPA** generates your repository implementations. Spring Boot auto-configures all of this with `starter-data-jpa` (Phase 5.2).

---

## 2. Entity Mapping

An **entity** is a Java class mapped to a database table:
```java
@Entity                                    // marks this as a JPA entity (-> a table)
@Table(name = "users")                     // table name (recall naming, Phase 4.3)
public class User {
    @Id                                    // the primary key (Phase 4.1)
    @GeneratedValue(strategy = GenerationType.IDENTITY)   // auto-increment (Phase 4.1)
    private Long id;                        // Long -> nullable before insert (recall Phase 1.2)

    @Column(nullable = false, unique = true, length = 255)   // column constraints (Phase 4.1)
    private String email;

    @Column(name = "full_name")            // map to a differently-named column
    private String name;

    @Enumerated(EnumType.STRING)           // store enum as its name, NOT ordinal! (recall Phase 1.3)
    private Status status;

    private LocalDateTime createdAt;        // Java 8 time types are supported (Phase 1.4)

    @Transient                             // NOT persisted (like Phase 1.9 transient)
    private String tempValue;

    protected User() {}                    // JPA REQUIRES a no-arg constructor (recall Phase 1.3!)
    // ... getters/setters ...
}
```
| Annotation | Maps |
|------------|------|
| `@Entity` / `@Table` | Class → table |
| `@Id` / `@GeneratedValue` | Primary key / generation strategy |
| `@Column` | Field → column (name, nullable, unique, length) |
| `@Enumerated(EnumType.STRING)` | Enum → string column |
| `@Transient` | Excluded from persistence |
| `@Lob`, `@Embedded`, `@MappedSuperclass` | Large object / embedded value / shared base |

### 2.1 Critical entity rules
> ⚠️ Three must-know rules:
> 1. **Entities need a no-arg constructor** — Hibernate instantiates them reflectively (recall Phase 1.3/1.14). This is why **records can't be JPA entities** (no no-arg constructor — recall Phase 1.3) — use records for **DTOs** instead (Phase 7).
> 2. **`@Enumerated(EnumType.STRING)`, NEVER ORDINAL** — ordinal is fragile; reordering enum constants corrupts data (recall the enum persistence warning, Phase 1.3/4.1).
> 3. **Use `Long` (wrapper) for the ID** — it's `null` before the entity is saved (recall Phase 1.2 — wrappers represent "not set yet").

### 2.2 ID generation strategies (recall Phase 4.1)
| Strategy | Behavior |
|----------|----------|
| `IDENTITY` | DB auto-increment column (Postgres `SERIAL`) |
| `SEQUENCE` | A DB sequence (better for batch inserts; Postgres default) |
| `UUID` | A UUID PK (distributed systems — Phase 4.1) |

### 2.3 Inheritance & embeddables
- **`@MappedSuperclass`** — share common fields (e.g., a `BaseEntity` with `id`, `createdAt`) across entities.
- **`@Embeddable`/`@Embedded`** — a value object embedded as columns (e.g., an `Address` embedded in `User`) — recall value objects (Phase 1.3/12).
- **`@Inheritance`** strategies (`SINGLE_TABLE`, `JOINED`, `TABLE_PER_CLASS`) map class hierarchies to tables.

---

## 3. Relationship Mapping

JPA maps the relationships from your schema design (Phase 4.3 — 1:1, 1:many, many:many) to object references:
| Annotation | Relationship | Maps |
|------------|--------------|------|
| **`@OneToMany`** | One has many | A collection on the "one" side |
| **`@ManyToOne`** | Many belong to one | A reference + FK on the "many" side |
| **`@OneToOne`** | One to one | A reference (FK + unique) |
| **`@ManyToMany`** | Many to many | Via a junction table (`@JoinTable` — Phase 4.3) |
```java
@Entity
public class Order {
    @Id @GeneratedValue private Long id;

    @ManyToOne(fetch = FetchType.LAZY)        // many orders -> one user (FK on this side)
    @JoinColumn(name = "user_id")             // the FK column (Phase 4.1)
    private User user;
}

@Entity
public class User {
    @Id @GeneratedValue private Long id;

    @OneToMany(mappedBy = "user",             // "user" = the field in Order owning the FK
               cascade = CascadeType.ALL,     // cascade ops to children (Phase 4.1)
               orphanRemoval = true,          // delete children removed from the collection
               fetch = FetchType.LAZY)        // load orders only when accessed (default for collections)
    private List<Order> orders = new ArrayList<>();
}
```
| Attribute | Meaning |
|-----------|---------|
| **`mappedBy`** | Marks the **inverse** side (the other side owns the FK) |
| **`cascade`** | Propagate operations (persist/remove) to related entities (Phase 4.1) |
| **`orphanRemoval`** | Delete a child when removed from the parent's collection |
| **`fetch`** | `LAZY` (load on access) vs `EAGER` (load immediately) |
| **`@JoinColumn`** | The FK column |

### 3.1 Always use FetchType.LAZY for collections
> ⚠️ **Use `FetchType.LAZY` for `@OneToMany`/`@ManyToMany`** (it's the default for them). `EAGER` loads related data on every query even when you don't need it → wasteful, and a major cause of the N+1 problem (§4) and performance issues. `@ManyToOne`/`@OneToOne` default to `EAGER` — consider setting them `LAZY` too.

### 3.2 The N+1 Problem (Critical!)
> ⚠️ **The N+1 problem is the #1 JPA performance pitfall.** Loading N parent entities, then accessing a lazy collection on each, triggers **1 query for the parents + N queries for the children** = N+1 queries (recall Phase 2.1 — hidden cost in a loop; Phase 4.2 — should be one JOIN).
```java
List<User> users = userRepository.findAll();         // 1 query (SELECT * FROM users)
for (User u : users) {
    u.getOrders().size();                            // N queries! one per user (lazy load)
}                                                     // -> 1 + N queries instead of 1 JOIN
```
**Fixes:**
| Fix | How |
|-----|-----|
| **`JOIN FETCH`** | `@Query("SELECT u FROM User u JOIN FETCH u.orders")` — one query with a join |
| **`@EntityGraph`** | Declaratively fetch relationships in one query (next note) |
| **Batch fetching** | `@BatchSize` / `hibernate.default_batch_fetch_size` — fetch children in batches |
| **DTO projections** | Select only what you need (next note) |
> The N+1 problem is silent (the code works, just slowly) — detect it by **logging SQL** (`spring.jpa.show-sql=true` / `hibernate.SQL` logging) and watching for repeated queries. This is one of the most important things to understand about JPA.

### 3.3 LazyInitializationException
> ⚠️ Accessing a LAZY relationship **outside** an active persistence context (after the transaction/session closed — e.g., in a controller after the service method returned) throws **`LazyInitializationException`**. Fix: fetch the data within the transaction (`JOIN FETCH`/`@EntityGraph`), or map to a DTO inside the transactional service. **Don't** "fix" it with `EAGER` or `open-session-in-view` (anti-patterns).

---

## 4. Entity Lifecycle (Persistence States)

A JPA entity moves through **four states**, managed by the **persistence context** (Hibernate's first-level cache — §5):
```
        new User()           persist()/save()
NEW (transient) ------------> MANAGED (persistent) --------> REMOVED
   not tracked                 tracked & synced              marked for deletion
                                    |  detach() / tx ends
                                    v
                                DETACHED (no longer tracked)
```
| State | Meaning |
|-------|---------|
| **Transient (new)** | A `new` object, not yet associated with the DB |
| **Managed (persistent)** | Tracked by the persistence context; changes are auto-synced (dirty checking — §5) |
| **Detached** | Was managed, but the context closed (changes no longer tracked) |
| **Removed** | Marked for deletion on flush/commit |
> The key insight: a **managed** entity's changes are **automatically persisted** (you don't call `save()` for updates — see dirty checking §5). A **detached** entity (returned from a closed transaction) is not tracked — changing it does nothing until you re-attach (`merge`).

---

## 5. The Persistence Context (First-Level Cache)

The **persistence context** (the JPA `EntityManager` / Hibernate `Session`) is a **first-level cache** that tracks all **managed** entities within a transaction. It provides three powerful behaviors:

### 5.1 First-level cache
Within one transaction, fetching the same entity twice returns the **same instance** (one DB query, cached):
```java
User u1 = repo.findById(1L).get();
User u2 = repo.findById(1L).get();   // SAME object — no second DB query (cached in the context)
// u1 == u2  -> true
```

### 5.2 Dirty checking (automatic updates!)
> ⭐ **Hibernate automatically detects changes to managed entities and writes UPDATEs on flush/commit — you do NOT call `save()` for updates.**
```java
@Transactional
public void updateEmail(Long id, String email) {
    User user = repo.findById(id).get();   // managed entity
    user.setEmail(email);                   // just change it...
    // NO save() needed! Hibernate detects the change (dirty checking)
}                                           // ...and issues UPDATE on commit
```
> This surprises newcomers: **changing a managed entity inside a transaction auto-persists** the change. `save()` is mainly for *new* entities (and re-attaching detached ones via `merge`).

### 5.3 Flushing
- **Flush** = synchronizing the persistence context to the database (executing pending SQL). Happens automatically before queries and at commit.
- Changes are **batched** and flushed efficiently, not one statement at a time.

### 5.4 Second-level cache (awareness)
The **second-level cache** is shared *across* transactions/sessions (e.g., Ehcache/Hazelcast) — caches entities between requests. Optional; configure carefully. (Application caching is usually done with Redis/Spring Cache instead — Phase 5.7.)

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using a record as a JPA entity | Entities need a no-arg constructor; use records for DTOs |
| `@Enumerated(ORDINAL)` (the default!) | Always `EnumType.STRING` |
| `EAGER` fetching collections | Use `LAZY` (default) + `JOIN FETCH` when needed |
| N+1 query problem | `JOIN FETCH` / `@EntityGraph` / batch / DTOs |
| `LazyInitializationException` | Fetch within the transaction / map to DTO |
| Calling `save()` to update a managed entity | Dirty checking auto-updates it |
| Exposing entities in the API | Use DTOs (Phase 7) |
| Primitive `long` for ID | Use `Long` (nullable before insert) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Maps directly to your schema (Phase 4.1/4.3):** `@Entity`→table, `@ManyToOne`→FK, `@ManyToMany`→junction table.
- **N+1 + indexing FKs (Phase 4.4)** are the top JPA performance concerns (Phase 13).
- **`@Transactional` (Phase 5.5)** defines the persistence-context boundary (dirty checking, lazy loading work *within* it).
- **DTOs + MapStruct (Phase 7)** decouple entities from the API (and avoid lazy/serialization issues).
- **Records for DTOs** (Phase 1.3), classes for entities (the no-arg-constructor distinction).
- **Optimistic locking (`@Version` — next note, Phase 4.5)** for concurrent updates.
- **Testcontainers (Phase 6)** test against real PostgreSQL.

---

## 8. Quick Self-Check Questions

1. What's the relationship between JPA, Hibernate, and Spring Data JPA?
2. What are the three critical entity rules (constructor, enum, ID type)?
3. Why can't a record be a JPA entity?
4. How do you map the four relationship types? What does `mappedBy` mean?
5. What is the N+1 problem, and how do you fix it?
6. What causes `LazyInitializationException`?
7. What are the four entity lifecycle states?
8. What is dirty checking, and why don't you call `save()` to update a managed entity?
9. What is the persistence context / first-level cache?

---

## 9. Key Terms Glossary

- **JPA / Hibernate / Spring Data JPA:** spec / implementation / boilerplate-remover.
- **Entity (`@Entity`):** a Java class mapped to a table.
- **`@Id`/`@GeneratedValue`/`@Column`/`@Enumerated`:** mapping annotations.
- **Relationship annotations (`@OneToMany`/`@ManyToOne`/`@ManyToMany`):** map associations.
- **`mappedBy` / `cascade` / `orphanRemoval` / fetch:** relationship attributes.
- **FetchType LAZY/EAGER:** load-on-access vs load-immediately.
- **N+1 problem:** 1 + N queries from lazy loading in a loop.
- **`LazyInitializationException`:** accessing lazy data outside the context.
- **Entity lifecycle (transient/managed/detached/removed):** persistence states.
- **Persistence context / first-level cache:** the `EntityManager`'s tracked entities.
- **Dirty checking:** auto-UPDATE of changed managed entities.
- **Flush:** syncing the context to the DB.

---

*This is the first note of **Section 5.4 — Spring Data JPA**.*
*Next topic: **Repositories, Queries, Pagination, Projections & Performance**.*
