# Spring Cache Abstraction

> **Phase 5 — Spring Framework & Spring Boot → 5.7 Spring Cache Abstraction**
> Goal: Master Spring's caching — `@Cacheable`/`@CachePut`/`@CacheEvict`/`@Caching`, cache managers (Caffeine, Redis, etc.), cache configuration (TTL, eviction), and caching strategies.

---

## 0. The Big Picture

**Caching** stores the results of expensive operations (DB queries, computations, API calls) so repeated requests return the **cached result** instead of recomputing — dramatically improving performance. Spring's **Cache Abstraction** lets you add caching **declaratively** with annotations (via AOP — recall Phase 5.1), independent of the cache backend.

```
@Cacheable("products")
public Product getProduct(Long id) { ... }   // 1st call: runs the method & caches; 2nd call: returns cache
```

> Caching is a top performance lever (recall the time-space trade-off, Phase 2.1 — trade memory for speed; LRU caches, Phase 2.2; Redis, Phase 4.8). Spring makes it a one-annotation change. Build on AOP (Phase 5.1 — same proxy gotchas) and Redis (Phase 4.8).

---

## 1. Why Cache? (The Performance Win)

| Without cache | With cache |
|---------------|------------|
| Every request hits the DB / recomputes | Repeated requests served from memory |
| Slow, high DB load | Fast (microseconds), low DB load |
| Doesn't scale under load | Absorbs read traffic |
> Caching is ideal for **read-heavy, rarely-changing** data (product catalogs, config, user profiles) — the same data is requested repeatedly. It applies the time-space trade-off (Phase 2.1): use memory to avoid expensive recomputation/DB hits.

---

## 2. Enabling Caching & the Cache Annotations

Enable with `@EnableCaching`, then annotate methods (recall AOP — these work via proxies, Phase 5.1):
```java
@Configuration
@EnableCaching                          // turn on Spring's caching support
public class CacheConfig {}
```
| Annotation | Effect |
|------------|--------|
| **`@Cacheable`** | Cache the result; return cached on subsequent calls (skip the method) |
| **`@CachePut`** | Always run the method AND update the cache |
| **`@CacheEvict`** | Remove entries from the cache |
| **`@Caching`** | Combine multiple cache operations on one method |

### 2.1 @Cacheable (read-through caching)
```java
@Cacheable("products")                       // cache name = "products"
public Product getProduct(Long id) {
    return productRepository.findById(id)...; // only runs on a cache MISS
}
// 1st call(id=5): cache miss -> run method -> cache result under key 5
// 2nd call(id=5): cache HIT -> return cached value, method NOT executed
```
> The method body runs **only on a cache miss**; on a hit, the cached value is returned and the method is **skipped** entirely. The default cache **key** is the method arguments (`id` here).

### 2.2 @CachePut (update the cache)
```java
@CachePut(value = "products", key = "#product.id")   // always runs, updates the cache
public Product updateProduct(Product product) {
    return productRepository.save(product);
}
```
> Unlike `@Cacheable`, `@CachePut` **always executes** the method and **stores the result** — use it for updates so the cache stays consistent with the DB.

### 2.3 @CacheEvict (invalidate)
```java
@CacheEvict(value = "products", key = "#id")         // remove one entry
public void deleteProduct(Long id) { ... }

@CacheEvict(value = "products", allEntries = true)   // clear the entire cache
public void deleteAllProducts() { ... }
```
> Evict entries when the underlying data changes (delete/bulk update) so the cache doesn't serve stale data — this is **cache invalidation** (§5).

### 2.4 Cache keys (SpEL — recall Phase 5.1)
```java
@Cacheable(value = "users", key = "#email")                    // key = the email argument
@Cacheable(value = "users", key = "#user.id")                  // key = a field of the argument
@Cacheable(value = "results", condition = "#id > 10")          // only cache if condition holds
@Cacheable(value = "results", unless = "#result == null")      // don't cache null results
```
> Cache keys use **SpEL** (Phase 5.1). By default the key is the method args; customize with `key`. `condition` (cache only if true) and `unless` (don't cache if true — e.g., skip caching nulls) give fine control. ⚠️ Custom-object keys need correct `equals`/`hashCode` (recall Phase 1.3/2.2).

---

## 3. Cache Managers (The Backend)

Spring's caching is **backend-agnostic** — you pick a `CacheManager` implementation:
| CacheManager | Backed by | Use |
|--------------|-----------|-----|
| **`ConcurrentMapCacheManager`** | In-memory `ConcurrentHashMap` (Phase 1.6) | Default; dev/testing (not for production) |
| **Caffeine** | High-performance in-process cache | Single-instance apps (fast, local) |
| **Redis** | Redis (Phase 4.8) | **Distributed** caching (shared across instances) |
| **EhCache / Hazelcast** | Various | Other distributed/local options |

### 3.1 In-process (Caffeine) vs distributed (Redis)
| | **Caffeine (in-process)** | **Redis (distributed)** |
|---|---------------------------|--------------------------|
| Location | Inside each app instance (JVM heap) | A separate Redis server |
| Speed | Fastest (no network) | Fast (network hop) |
| Shared across instances? | ❌ No (each instance has its own) | ✅ Yes (one shared cache) |
| Survives app restart? | ❌ No | ✅ Yes |
| Use when | Single instance / per-instance data | Multiple instances / shared cache |
```yaml
# Redis cache config (Spring Boot):
spring:
  cache:
    type: redis
  data:
    redis:
      host: localhost
```
> ⚠️ **In a multi-instance deployment (behind a load balancer — Phase 0.2/13), use Redis** (a shared cache) — otherwise each instance has its **own** in-process cache, leading to **inconsistency** (instance A caches a value, instance B doesn't, or has a stale one). **Caffeine** is great for single instances or per-instance data. (Recall LRU/eviction, Phase 2.2; Redis, Phase 4.8.)

---

## 4. Cache Configuration (TTL & Eviction)

A cache needs limits so it doesn't grow forever (recall the unbounded-cache memory leak, Phase 1.11!):
| Setting | Purpose |
|---------|---------|
| **TTL (Time To Live)** | Entries expire after a duration (avoid stale data) |
| **Max size** | Cap the number of entries |
| **Eviction policy** | LRU/LFU — which to remove when full (recall Phase 2.2/4.8) |
```java
// Caffeine with TTL + max size:
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager mgr = new CaffeineCacheManager("products");
    mgr.setCaffeine(Caffeine.newBuilder()
        .maximumSize(1000)                          // max 1000 entries (LRU eviction)
        .expireAfterWrite(Duration.ofMinutes(10))); // TTL: 10 minutes
    return mgr;
}
```
> ⚠️ **Always set TTL and/or max size** — an unbounded cache is a **memory leak** (recall Phase 1.11 — the classic unbounded-static-map leak). TTL also bounds staleness (entries refresh after expiry). Eviction (LRU/LFU — Phase 2.2/4.8) removes cold entries when full. (Caffeine actually uses W-TinyLFU, often better than plain LRU.)

---

## 5. Cache Invalidation (The Hard Problem)

> "There are only two hard things in computer science: cache invalidation and naming things." Stale cache data is a top source of bugs.

The challenge: when the underlying data changes, the cache must be **updated or invalidated** — or you serve **stale** data.
| Strategy | How |
|----------|-----|
| **`@CacheEvict` on writes** | Evict the entry when data is updated/deleted (§2.3) |
| **`@CachePut` on updates** | Refresh the cache with the new value |
| **TTL** | Let entries expire automatically (bounds staleness) |
| **Event-driven invalidation** | Publish an event on change → evict (Phase 5.1 events / messaging Phase 11) |
> ⚠️ The hardest part of caching is **keeping it consistent with the source of truth**. Combine `@CacheEvict`/`@CachePut` on writes with a sensible TTL as a safety net. Accept some staleness for read-heavy data where it's tolerable; for data that must be fresh, don't cache (or use very short TTLs).

---

## 6. Caching Strategies (Patterns)

| Strategy | How it works |
|----------|--------------|
| **Cache-aside** (lazy) | App checks cache; on miss, loads from DB & populates cache (`@Cacheable` does this) |
| **Read-through** | The cache itself loads from the DB on a miss |
| **Write-through** | Writes go to cache AND DB synchronously |
| **Write-behind** (write-back) | Write to cache; flush to DB asynchronously (fast, risk of loss) |
> **`@Cacheable` implements cache-aside** (the most common strategy): check cache → on miss, run the method (load) → cache the result. Write-through/write-behind are for write-heavy caching scenarios.

---

## 7. The Proxy Gotcha (recall Phase 5.1/5.5)

> ⚠️ **Caching annotations work via AOP proxies (Phase 5.1) — so they have the SAME self-invocation problem as `@Transactional`.** A `@Cacheable` method called from *within the same class* bypasses the proxy → caching is **not applied**.
```java
public List<Product> getAll() {
    return getProduct(5);   // ❌ internal call — @Cacheable on getProduct is IGNORED!
}
@Cacheable("products")
public Product getProduct(Long id) { ... }
```
Fix: call via another bean / injected self (recall Phase 5.1/5.5). Also, caching only works on **public methods of Spring beans**.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Unbounded cache (no TTL/max size) | Always set limits (memory leak — Phase 1.11) |
| Stale data (no invalidation) | `@CacheEvict`/`@CachePut` on writes + TTL |
| In-process cache across instances | Use Redis (shared) for multi-instance (Phase 13) |
| Self-invocation (caching ignored) | Call via another bean (Phase 5.1) |
| Caching frequently-changing data | Don't cache it (or very short TTL) |
| Caching `null` results unintentionally | Use `unless = "#result == null"` |
| Custom key objects without equals/hashCode | Implement them (Phase 1.3) |
| Caching everything | Cache read-heavy, rarely-changing data only |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Built on AOP** (Phase 5.1) — same proxy/self-invocation gotchas as `@Transactional` (Phase 5.5).
- **Redis-backed cache** (Phase 4.8) for distributed/shared caching in multi-instance deployments (Phase 13).
- **Time-space trade-off** (Phase 2.1) + **LRU/LFU eviction** (Phase 2.2) in action.
- **Cache as a top performance lever** (Phase 13) — reduces DB load (Phase 4.4 indexing complements it).
- **Projects 4, 5, 7** use Redis caching for frequently-accessed data (catalogs, etc.).
- **HTTP caching** (ETags/Cache-Control — Phase 0.2/7) is a complementary layer.
- **Event-driven invalidation** ties to Spring events (Phase 5.1) / messaging (Phase 11).

---

## 10. Quick Self-Check Questions

1. Why cache, and what data is a good candidate?
2. What do `@Cacheable`, `@CachePut`, and `@CacheEvict` do?
3. When does a `@Cacheable` method body actually run?
4. How are cache keys determined, and how do you customize them?
5. When use Caffeine vs Redis as the cache backend?
6. Why must you set TTL/max size, and what's the eviction policy for?
7. What is cache invalidation, and why is it hard?
8. What is cache-aside, and which annotation implements it?
9. What proxy gotcha do caching annotations share with `@Transactional`?

---

## 11. Key Terms Glossary

- **Caching:** storing results to avoid recomputation/DB hits.
- **`@EnableCaching`:** enables Spring caching.
- **`@Cacheable` / `@CachePut` / `@CacheEvict` / `@Caching`:** cache / update / evict / combine.
- **Cache key:** identifies an entry (default = method args; SpEL customizable).
- **`CacheManager`:** the cache backend abstraction.
- **Caffeine / Redis:** in-process / distributed cache backends.
- **TTL / max size / eviction (LRU/LFU):** entry lifetime / capacity / removal policy.
- **Cache invalidation:** keeping the cache consistent with the source.
- **Cache-aside / read-through / write-through / write-behind:** caching strategies.
- **Self-invocation problem:** internal calls bypass the cache proxy.

---

*This is the note for **Section 5.7 — Spring Cache Abstraction**.*
*Next section in roadmap: **5.8 Spring Scheduling**.*
