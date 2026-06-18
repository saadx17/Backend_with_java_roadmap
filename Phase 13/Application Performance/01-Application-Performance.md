# Application Performance

> **Phase 13 — Performance & Optimization → 13.1 Application Performance**
> Goal: Profile and tune a Java backend — profiling tools (Async Profiler, JFR, VisualVM) and flame graphs, JVM tuning, connection-pool tuning (HikariCP), database performance, caching strategies, HTTP optimization, and async processing.

---

## 0. The Big Picture

Performance work is **measure → find the bottleneck → fix → re-measure**. You don't guess; you **profile**. The biggest wins in a typical Java backend are usually **database** (slow queries, N+1) and **I/O** (remote calls), not raw CPU — so optimize where the data points, not where intuition says.

```
   Profile (find the real bottleneck)  ──►  Fix the biggest one  ──►  Re-measure
        │                                                              │
        └──────────── repeat (Amdahl's law: optimize the dominant cost) ◄┘
```

> Builds on the JVM (Phase 1.11 — heap/GC), JDBC/indexing (Phase 4.4/4.7), JPA performance (Phase 5.4), caching (Phase 5.7), async (Phase 5.3/5.8), and observability (Phase 9.2 — metrics tell you *where* to look). Feeds load testing (13.2) and scalability (13.3).

> ⭐ **"Premature optimization is the root of all evil" (Knuth)** — but *deliberate, measured* optimization is essential. ⚠️ Don't micro-optimize hot-feeling code; **profile first**.

---

## 1. Profiling Tools & Flame Graphs

| Tool | What | Note |
|------|------|------|
| **JFR (Java Flight Recorder)** ⭐ | Built-in, low-overhead event recorder | Production-safe (~1% overhead); records CPU, allocation, GC, locks, I/O |
| **Async Profiler** ⭐ | Low-overhead sampling CPU/alloc profiler | Best **flame graphs**; no safepoint bias; native + Java stacks |
| **VisualVM** | GUI profiler/monitor | Heap dumps, threads, sampling — good for dev |
| **JConsole / Mission Control (JMC)** | JMX monitoring / analyze JFR recordings | JMC reads JFR files |
| **Heap dump + MAT** | Memory leak analysis | Find what's retaining memory (Phase 1.11) |

### 1.1 Flame graphs ⭐
A **flame graph** visualizes where CPU time (or allocation) is spent: width = time spent, stack = call hierarchy. The **widest bars are your hottest paths**.
```
   [============ handleRequest 100% ============]
   [== parseJson 15% ==][===== queryDb 70% =====][render 15%]
                         [== N+1 selectUser 60% ==]   ◄── the bottleneck is obvious
```
> ⭐ Generate flame graphs with **Async Profiler** (or from JFR). Read top-down for the call tree, look for the **widest frames** — that's where to optimize. Wide DB/serialization frames are the usual suspects.

---

## 2. JVM Tuning

(Recall heap/GC internals — Phase 1.11.) Tune **after** profiling shows GC/memory is the issue.
| Area | Lever |
|------|-------|
| **Heap size** | `-Xmx`/`-Xms`, or in containers **`-XX:MaxRAMPercentage`** (10.1 §7.1) |
| **GC choice** | **G1** (default, balanced), **ZGC**/**Shenandoah** (low pause, large heaps), Parallel (throughput/batch) |
| **GC tuning** | Pause-time goals (`-XX:MaxGCPauseMillis`), region sizes — usually leave defaults |
| **Diagnostics** | GC logs (`-Xlog:gc*`), monitor pause times & frequency (9.2) |
| **Startup** | AppCDS, tiered compilation; GraalVM native for fast startup (Phase 16.5) |
> ⭐ Modern GCs are good — **don't over-tune**. Most "GC problems" are really **allocation problems** (creating too much garbage) → fix the code (fewer objects, reuse, streaming) rather than fiddling GC flags. In containers, sizing the heap to the limit (10.1) prevents OOMKills. Watch GC pause time / throughput as metrics (9.2).

---

## 3. Connection Pool Tuning (HikariCP) ⭐

Opening a DB connection is expensive; a **connection pool** reuses them. Spring Boot's default is **HikariCP** (fast, reliable). Pool sizing is one of the most impactful and **misunderstood** tunings.
```yaml
spring.datasource.hikari:
  maximum-pool-size: 10          # ⭐ usually SMALLER than you think
  minimum-idle: 10
  connection-timeout: 30000      # ms to wait for a connection before failing
  max-lifetime: 1800000          # recycle connections (< DB/infra timeout)
  idle-timeout: 600000
```
| Setting | Meaning |
|---------|---------|
| `maximum-pool-size` | Max concurrent DB connections (the key knob) |
| `connection-timeout` | Wait limit for a free connection (→ fail fast) |
| `max-lifetime` | Retire old connections (avoid stale/firewalled conns) |
| `minimum-idle` | Idle connections kept ready |
> ⭐ ⚠️ **Bigger is NOT better.** A pool larger than the DB can handle causes **contention** at the database (more connections fighting for CPU/locks → slower). A common formula: `connections ≈ (core_count × 2) + effective_spindle_count` — often **10–20**, not 100s. The pool size should match what the **DB** can do, not your thread count. Size with load tests (13.2). Pool metrics via Micrometer (9.2).

---

## 4. Database Performance ⭐ (the usual #1 bottleneck)

| Problem | Fix |
|---------|-----|
| **Missing indexes** | Add indexes on filtered/joined/sorted columns (Phase 4.4); read `EXPLAIN` plans |
| **N+1 queries** ⭐ | Fetch joins / `@EntityGraph` / batch fetching (Phase 5.4) |
| **Over-fetching** | Select only needed columns/projections (DTO projections — Phase 5.4/7.3) |
| **Deep offset pagination** | Cursor/keyset pagination (Phase 7.1 §5.2) |
| **Chatty transactions** | Batch writes (`saveAll`, JDBC batch — Phase 4.7) |
| **Lock contention** | Right isolation level / optimistic locking (Phase 4.5) |
| **Slow queries** | `EXPLAIN ANALYZE`, query rewrite, denormalize/CQRS read models (Phase 11.4) |
> ⭐ **The N+1 query problem** (Phase 5.4) is the most common JPA performance killer: loading a list then lazily loading each element's relation = 1 + N queries. Fix with **join fetch / `@EntityGraph` / batch size**. Always profile actual SQL (enable SQL logging — 9.1, or use a profiler). The DB is where most backend latency lives.

---

## 5. Caching Strategies ⭐

(Recall Spring Cache — Phase 5.7; Redis — Phase 4.8.) Caching avoids repeating expensive work (DB queries, computations, remote calls).
| Strategy | How it works |
|----------|--------------|
| **Cache-aside** (lazy) ⭐ | App checks cache; miss → load from DB → put in cache. Most common (`@Cacheable`) |
| **Read-through / write-through** | Cache library loads/writes DB transparently |
| **Write-behind** | Write to cache, async flush to DB (fast writes, risk on crash) |
| **Local (Caffeine)** | In-JVM, fastest, per-instance (⚠️ not shared across instances) |
| **Distributed (Redis)** | Shared across instances — needed for microservices (Phase 4.8/12) |
```java
@Cacheable("products")            // cache-aside via Spring Cache (Phase 5.7)
public Product get(Long id) { return repo.findById(id).orElseThrow(); }
@CacheEvict(value="products", key="#p.id")
public void update(Product p) { ... }
```
| Caching concern | Note |
|-----------------|------|
| **TTL / eviction** | Expire stale data; size limits (LRU/LFU) |
| **Invalidation** ⭐ | "There are only two hard things..." — evict on writes (`@CacheEvict`) |
| **Cache stampede** | Many misses at once hammer the DB → use locking/`sync=true`/staggered TTL |
| **Hit ratio** | Monitor (9.2) — a low hit ratio means the cache isn't helping |
> ⭐ **Cache-aside + Redis** (distributed) is the microservices default. ⚠️ The hard part is **invalidation** (stale data) — cache things that are read-heavy and tolerate slight staleness; evict on change. Watch the **hit ratio** (9.2) to confirm value.

---

## 6. HTTP & Async Optimization

| Technique | Benefit |
|-----------|---------|
| **Response compression** (gzip/br) | Smaller payloads (`server.compression.enabled=true`) |
| **HTTP/2** | Multiplexing, fewer connections |
| **ETags / conditional requests** | Skip transfer when unchanged (Phase 7.1 §3) — 304 |
| **Pagination / projections** | Don't return huge payloads (Phase 7.1) |
| **Connection reuse / keep-alive** | Avoid handshake cost on outbound calls (12.2) |
| **Parallelize independent calls** | `CompletableFuture`/reactive (12.2 §6) |
| **`@Async` / offload** | Do slow work off the request thread (Phase 5.3/5.8); 202+polling (Phase 7.1 §6) |
| **Reactive (WebFlux)** | Handle many concurrent I/O-bound requests with few threads (Phase 16.1) |
| **Virtual threads (Java 21)** | Cheap blocking concurrency (Phase 1.10) — high I/O throughput without reactive complexity |
> ⭐ For **I/O-bound** workloads (most backends), the win is **concurrency** (don't block threads waiting): async/parallel calls, reactive, or **virtual threads** (Java 21 — Phase 1.10/1.11). For **latency**, cut payload size (compression, projections) and round trips (caching, batching).

---

## 7. A Performance Methodology

```
1. Define SLOs/targets (p95 latency, throughput) — measurable (9.2/13.2)
2. Load test to reproduce (13.2) — realistic traffic
3. Profile under load (JFR/Async Profiler) — find the bottleneck (§1)
4. Fix the BIGGEST contributor (DB? GC? lock? I/O?)
5. Re-measure — did it move the metric?
6. Repeat until targets met; watch in prod (9.2)
```
> ⭐ Optimize the **dominant cost** (Amdahl's law). One fixed N+1 query often beats a week of micro-tuning. Always tie work to a **metric** (p95 — 9.2/13.2), not a feeling.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Optimizing without profiling | Measure first (§1/§7) |
| Oversized connection pool | Right-size (often 10–20); match DB capacity (§3) |
| N+1 queries | Fetch joins / `@EntityGraph` (§4, 5.4) |
| Over-tuning GC instead of reducing allocation | Fix allocation in code (§2) |
| Local cache assumed shared across instances | Use Redis for distributed (§5) |
| Caching without invalidation/TTL | Evict on write; set TTL (§5) |
| Cache stampede on cold cache | sync loading / staggered TTL (§5) |
| Blocking threads on I/O | Async/reactive/virtual threads (§6) |
| Returning huge unpaginated payloads | Paginate/project/compress (§4/§6) |
| Tuning to a feeling, not a metric | Tie to p95/throughput (§7, 13.2) |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **JVM/GC/heap** (Phase 1.11), **virtual threads** (1.10); **container heap** sizing (10.1).
- **DB**: indexing (4.4), JDBC batching (4.7), JPA N+1/projections (5.4), pagination (7.1).
- **HikariCP** is Spring's default pool (5.2); metrics via Micrometer (9.2).
- **Caching** (Spring Cache 5.7, Redis 4.8); **async** (5.3/5.8), **reactive** (16.1).
- **Observability** (9.2) provides the metrics; **load testing** (13.2) reproduces load; **scalability** (13.3) is the horizontal complement.
- **APM** (9.3) auto-finds slow methods/N+1.
- Applied in **Projects 5/7** (perf tuning under load).

---

## 10. Quick Self-Check Questions

1. What's the performance methodology, and why profile before optimizing?
2. Compare JFR, Async Profiler, and VisualVM. What does a flame graph show?
3. When tune the JVM/GC, and why is allocation often the real problem?
4. Why is a bigger connection pool often *worse*, and how do you size HikariCP?
5. What is the N+1 query problem and how do you fix it?
6. Name three database performance fixes beyond indexing.
7. Compare caching strategies; why is cache-aside + Redis the microservices default?
8. What is cache invalidation/stampede and how do you handle each?
9. How do you optimize I/O-bound workloads (concurrency options)?
10. Why optimize the dominant cost (Amdahl), tied to a metric?

---

## 11. Key Terms Glossary

- **Profiling / flame graph:** measuring where time/allocation goes / visualizing hot paths.
- **JFR / Async Profiler / VisualVM / JMC / MAT:** profiling & analysis tools.
- **GC (G1/ZGC/Shenandoah/Parallel):** garbage collectors.
- **Connection pool / HikariCP:** reusable DB connections / Spring's default pool.
- **N+1 query:** 1 + N queries from lazy loading a collection's relations.
- **Projection:** selecting only needed fields.
- **Caching (cache-aside / write-through / write-behind):** strategies to avoid repeated work.
- **Local (Caffeine) vs distributed (Redis) cache:** per-instance vs shared.
- **Cache invalidation / stampede / hit ratio:** eviction / thundering herd / effectiveness.
- **Virtual threads:** cheap JVM threads for blocking I/O concurrency (Java 21).
- **Amdahl's law:** speedup limited by the dominant cost.

---

*This is the note for **Section 13.1 — Application Performance**.*
*Previous section in roadmap: **12.7 Observability in Microservices**.*
*Next section in roadmap: **13.2 Load Testing**.*
