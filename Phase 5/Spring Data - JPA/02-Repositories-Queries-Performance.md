# Spring Data JPA: Repositories, Queries, Pagination & Performance

> **Phase 5 — Spring Framework & Spring Boot → 5.4 Spring Data JPA**
> Goal: Master Spring Data repositories — query methods, `@Query` (JPQL/native), pagination & sorting, Specifications, projections, entity graphs, auditing, locking, and performance tuning.

---

## 0. The Big Picture

**Spring Data JPA repositories** eliminate data-access boilerplate: you declare an **interface**, and Spring **generates the implementation** at runtime (via reflection/proxies — recall Phase 1.14). You get CRUD, query methods, pagination, and more — for free.

```
You declare:  interface UserRepository extends JpaRepository<User, Long> {}
Spring generates: save, findById, findAll, delete, count, ... — no implementation written!
```

> This is Spring Data's magic — and it's all built on the JPA entities/persistence context from the previous note. This note covers querying and performance.

---

## 1. Repository Interfaces

You extend a Spring Data interface; Spring provides the implementation as a bean (Phase 5.1):
```java
@Repository                                        // (optional — Spring Data detects it anyway)
public interface UserRepository extends JpaRepository<User, Long> {
    //                                              entity type ^^^^  ^^^^ ID type (recall generics, Phase 1.7)
}
```
| Interface | Provides |
|-----------|----------|
| `Repository<T, ID>` | Marker (base) |
| `CrudRepository<T, ID>` | `save`, `findById`, `findAll`, `delete`, `count` |
| `PagingAndSortingRepository<T, ID>` | + pagination & sorting |
| **`JpaRepository<T, ID>`** | + JPA extras (`flush`, `saveAll`, batch) — **the common choice** |
```java
userRepository.save(user);                    // INSERT or UPDATE
userRepository.findById(42L);                 // returns Optional<User> (recall Phase 1.8!)
userRepository.findAll();
userRepository.deleteById(42L);
userRepository.count();
```
> ⭐ `JpaRepository<User, Long>` is a **generic interface** (Phase 1.7) you parameterize with your entity and ID type. `findById` returns **`Optional<T>`** (recall Phase 1.8) → the idiomatic `findById(id).orElseThrow(() -> new NotFoundException())`.

---

## 2. Query Methods (Derived from Method Names)

Spring Data **generates queries from method names** — you write the method signature, it parses the name into a query (no implementation needed):
```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);                      // WHERE email = ?
    List<User> findByStatus(Status status);                        // WHERE status = ?
    List<User> findByStatusAndAgeGreaterThan(Status s, int age);   // WHERE status=? AND age>?
    List<User> findByNameContainingIgnoreCase(String name);        // WHERE name ILIKE %?%
    List<User> findByCreatedAtBetween(LocalDate from, LocalDate to);// WHERE createdAt BETWEEN
    long countByStatus(Status status);                             // SELECT COUNT(*)
    boolean existsByEmail(String email);                           // SELECT EXISTS
    void deleteByStatus(Status status);
    List<User> findTop10ByOrderByCreatedAtDesc();                  // ORDER BY + LIMIT 10
}
```
| Keyword | SQL (recall Phase 4.2) |
|---------|------------------------|
| `findBy` / `countBy` / `existsBy` / `deleteBy` | SELECT / COUNT / EXISTS / DELETE |
| `And` / `Or` | combine conditions |
| `GreaterThan` / `LessThan` / `Between` | range |
| `Like` / `Containing` / `StartingWith` | pattern (LIKE) |
| `IgnoreCase` | case-insensitive |
| `OrderBy...Desc` | sorting |
| `Top10` / `First` | limit |
> Spring parses the method name into a JPQL query at startup. Great for simple-to-moderate queries; for complex ones, use `@Query` (§3). ⚠️ Long method names get unreadable — switch to `@Query` when they do.

---

## 3. @Query (JPQL & Native SQL)

For queries too complex for method names, write them explicitly:
```java
public interface UserRepository extends JpaRepository<User, Long> {

    // JPQL — queries ENTITIES (not tables); uses class/field names:
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmailJpql(@Param("email") String email);

    // JPQL with JOIN FETCH — solves N+1 (recall previous note):
    @Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> findByIdWithOrders(@Param("id") Long id);

    // Native SQL — raw database SQL (for DB-specific features, e.g., window functions — Phase 4.2):
    @Query(value = "SELECT * FROM users WHERE created_at > :date", nativeQuery = true)
    List<User> findRecentNative(@Param("date") LocalDate date);

    // Modifying query (UPDATE/DELETE) — needs @Modifying + @Transactional:
    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.lastLogin < :date")
    int deactivateInactive(@Param("status") Status status, @Param("date") LocalDate date);
}
```
| | **JPQL** | **Native SQL** |
|---|----------|----------------|
| Operates on | **Entities/fields** (`User u`, `u.email`) | **Tables/columns** (raw SQL) |
| Portable | Across databases | DB-specific |
| Use for | Most custom queries | DB features JPQL lacks (window functions, JSONB — Phase 4.2/4.6) |
> **JPQL** queries the *object model* (entity & field names), not tables — Hibernate translates it to SQL. Use **native** queries for DB-specific SQL (window functions, CTEs, JSONB ops — Phase 4.2). `@Modifying` (+ `@Transactional`) marks UPDATE/DELETE queries.

---

## 4. Pagination & Sorting

For list endpoints, **never load everything** — paginate (recall LIMIT/OFFSET, Phase 4.2):
```java
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByStatus(Status status, Pageable pageable);   // paged result
    Slice<User> findByActive(boolean active, Pageable pageable); // slice (no total count)
}

// In the service/controller:
Pageable pageable = PageRequest.of(0, 20, Sort.by("createdAt").descending());  // page 0, size 20
Page<User> page = userRepository.findByStatus(Status.ACTIVE, pageable);
page.getContent();        // the rows
page.getTotalElements();  // total count (extra COUNT query)
page.getTotalPages();
```
| Type | Includes total count? | Use |
|------|----------------------|-----|
| **`Page<T>`** | ✅ Yes (runs a COUNT query) | When the UI needs total/last page |
| **`Slice<T>`** | ❌ No (just "has next?") | Infinite scroll (cheaper — no COUNT) |
> ⚠️ `Page<T>` runs an **extra COUNT query** to compute total pages — `Slice<T>` skips it (cheaper) when you only need "is there a next page?" Recall the deep-OFFSET problem (Phase 4.2) — for large datasets, keyset pagination beats `Pageable`'s OFFSET. Spring Boot can bind `Pageable` directly from request params (`?page=0&size=20&sort=createdAt,desc`).

---

## 5. Specifications & QueryDSL (Dynamic Queries)

For **dynamic** queries (filters that vary at runtime — e.g., an advanced search with optional criteria), method names/`@Query` are too rigid. Use **Specifications** (type-safe, composable):
```java
public interface UserRepository extends JpaRepository<User, Long>,
                                        JpaSpecificationExecutor<User> {}   // add this

// Build composable criteria:
Specification<User> spec = Specification.where(null);
if (status != null) spec = spec.and((root, q, cb) -> cb.equal(root.get("status"), status));
if (minAge != null) spec = spec.and((root, q, cb) -> cb.greaterThan(root.get("age"), minAge));
List<User> results = userRepository.findAll(spec);   // combines only the applied filters
```
> **Specifications** build queries dynamically from criteria (great for filterable list endpoints — Project 4 "filtering"). **QueryDSL** is an alternative offering more type-safe, fluent queries (generates a metamodel). Both solve "the query depends on which filters the user provided."

---

## 6. Projections (Fetch Only What You Need)

Loading full entities when you need a few fields is wasteful (recall avoid `SELECT *`, Phase 4.2). **Projections** select a subset:
```java
// Interface-based projection (Spring implements it):
public interface UserSummary {
    Long getId();
    String getName();      // only id and name are fetched
}
public interface UserRepository extends JpaRepository<User, Long> {
    List<UserSummary> findByStatus(Status status);   // returns projections, not full entities
}

// DTO/class-based projection via JPQL constructor expression:
@Query("SELECT new com.example.UserDto(u.id, u.name) FROM User u")
List<UserDto> findAllSummaries();
```
> Projections fetch **only the needed columns** → less data transferred (Phase 0.2), can use covering indexes (Phase 4.4), and avoid lazy-loading issues. **Records make great projection DTOs** (Phase 1.3). Use projections for read-heavy list/report endpoints.

---

## 7. Entity Graphs (Declarative Fetch — N+1 Fix)

**`@EntityGraph`** declaratively specifies which relationships to fetch in **one query** (an alternative to `JOIN FETCH` — solves N+1, recall previous note):
```java
public interface UserRepository extends JpaRepository<User, Long> {
    @EntityGraph(attributePaths = {"orders"})        // fetch orders in the same query
    List<User> findAll();
}
```
> `@EntityGraph` is a clean, reusable way to control eager fetching per query — fetching the listed relationships in one query instead of N+1.

---

## 8. Auditing (Auto-Populate created/updated)

Spring Data auto-fills **audit columns** (recall Phase 4.2/4.3 — `created_at`, `updated_at`):
```java
@Configuration
@EnableJpaAuditing                                   // enable auditing
public class JpaConfig {}

@Entity
@EntityListeners(AuditingEntityListener.class)
public class User {
    @CreatedDate  private LocalDateTime createdAt;    // auto-set on insert
    @LastModifiedDate private LocalDateTime updatedAt; // auto-set on update
    @CreatedBy    private String createdBy;            // auto-set (with an AuditorAware bean)
}
```
> This automates the audit columns from your schema design (Phase 4.3) — no manual timestamp setting. `@CreatedBy`/`@LastModifiedBy` capture the current user (via an `AuditorAware` bean reading the security context — Phase 5.6).

---

## 9. Optimistic & Pessimistic Locking (recall Phase 4.5)

### 9.1 Optimistic locking (`@Version` — the common choice)
```java
@Entity
public class Product {
    @Version                                          // optimistic locking version column (Phase 4.3/4.5)
    private Integer version;
}
```
> Hibernate adds `WHERE version = ?` to updates and increments it; a concurrent update fails with **`OptimisticLockException`** (recall Phase 4.5 — version column detects the conflict; retry on failure). The default for web apps.

### 9.2 Pessimistic locking (for high contention)
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)                 // SELECT ... FOR UPDATE (Phase 4.5)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Product findByIdForUpdate(@Param("id") Long id);
```
> Pessimistic locking (`SELECT ... FOR UPDATE`, Phase 4.5) for genuinely high-contention rows (e.g., inventory — Project 5). Locks the row so others wait.

---

## 10. Performance Tuning (Critical)

JPA performance issues are common — the key levers:
| Lever | How |
|-------|-----|
| **Fix N+1** | `JOIN FETCH` / `@EntityGraph` / batch fetching (previous note) |
| **Read-only queries** | `@Transactional(readOnly = true)` — Hibernate skips dirty-checking → faster |
| **Pagination** | Never `findAll()` on big tables; page it |
| **Projections/DTOs** | Fetch only needed fields (Phase 4.2) |
| **Batch inserts/updates** | `hibernate.jdbc.batch_size` (recall JDBC batching, Phase 4.7) |
| **Index FK & queried columns** | Phase 4.4 (the DB side) |
| **Log & review SQL** | `spring.jpa.show-sql` / Hibernate SQL logging to catch N+1 and bad queries |
```java
@Transactional(readOnly = true)      // optimization for read-only operations (also Phase 5.5)
public List<UserDto> getActiveUsers() { ... }
```
> ⚠️ **`@Transactional(readOnly = true)`** on read methods lets Hibernate skip dirty-checking and flushing → meaningful performance gain for reads. Combine with **fixing N+1, pagination, projections, and indexing (Phase 4.4)** — these are the top JPA/DB performance levers (Phase 13). Always **log the generated SQL** during development to catch problems early.

---

## 11. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `findAll()` on a huge table | Paginate (`Pageable`) |
| N+1 queries (silent) | `JOIN FETCH` / `@EntityGraph`; log SQL |
| Unreadable derived method names | Use `@Query` |
| Returning full entities for read endpoints | Use projections/DTOs |
| Forgetting `@Modifying`/`@Transactional` on UPDATE queries | Add both |
| `Page` everywhere (extra COUNT cost) | Use `Slice` when total isn't needed |
| No `readOnly` on read transactions | Add `@Transactional(readOnly=true)` |
| Ignoring optimistic locking on concurrent updates | Add `@Version` |
| Not indexing queried/FK columns | Index them (Phase 4.4) |

---

## 12. Connection to Backend / Spring (Why This Matters Later)

- **Repositories are beans** (Phase 5.1) — Spring generates them via proxies/reflection (Phase 1.14).
- **`findById` returns `Optional`** (Phase 1.8) → `orElseThrow` → mapped to `404` by `@RestControllerAdvice` (Phase 5.3).
- **N+1, indexing, read-only, projections** are the core performance topics (Phase 13).
- **Specifications + Pageable** power filterable, paginated list endpoints (Projects 4, 5).
- **`@Version` optimistic locking** (Phase 4.5) for concurrent updates (Project 4).
- **DTOs/projections + MapStruct** (Phase 7) decouple entities from the API.
- **Auditing** uses the security context (Phase 5.6) for `@CreatedBy`.
- **Native queries** for window-function reports/dashboards (Phase 4.2, Projects 4/5).
- **Testcontainers** (Phase 6) test repositories against real PostgreSQL.

---

## 13. Quick Self-Check Questions

1. How do you create a repository, and what do you get for free?
2. How do query methods work, and when should you switch to `@Query`?
3. What's the difference between JPQL and native queries?
4. How do you paginate, and what's the difference between `Page` and `Slice`?
5. When and how do you use Specifications?
6. What are projections, and why use them?
7. How does `@EntityGraph` help with N+1?
8. How do `@Version` (optimistic) and `@Lock` (pessimistic) locking work?
9. What are the top JPA performance levers (name four)?

---

## 14. Key Terms Glossary

- **`JpaRepository<T, ID>`:** generic repository interface (CRUD + more).
- **Query method:** a query derived from a method name.
- **`@Query` (JPQL / native):** explicit entity-based / raw SQL query.
- **`@Modifying`:** marks an UPDATE/DELETE query.
- **`Pageable` / `Page` / `Slice`:** pagination request / result with count / result without count.
- **Specification / QueryDSL:** dynamic, composable queries.
- **Projection:** fetching a subset of fields (interface or DTO based).
- **`@EntityGraph`:** declarative fetch plan (N+1 fix).
- **Auditing (`@CreatedDate`/`@LastModifiedDate`):** auto-populated audit fields.
- **`@Version` / `@Lock`:** optimistic / pessimistic locking.
- **`@Transactional(readOnly=true)`:** read-optimized transaction.

---

*Previous topic: **Entities, Relationships & Persistence Context**.*
*This completes **Section 5.4 — Spring Data JPA**.*
*Next section in roadmap: **5.5 Spring Transaction Management**.*
