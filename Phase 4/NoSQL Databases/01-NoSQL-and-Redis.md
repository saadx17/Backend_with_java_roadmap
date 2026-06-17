# NoSQL Databases & Redis

> **Phase 4 — Databases → 4.8 NoSQL Databases**
> Goal: Understand why NoSQL exists, the CAP theorem, the NoSQL types, and — in depth — **Redis** (data structures, expiration, persistence, use cases, transactions, clustering, Spring integration). Plus MongoDB awareness.

---

## 0. The Big Picture

**NoSQL** ("Not Only SQL") databases relax the relational model (and often ACID) to gain **scalability, flexibility, or speed** for specific use cases. They complement — not replace — relational databases. **Redis** in particular (an in-memory key-value store) is something **you WILL use** as a backend engineer for caching, sessions, rate limiting, and more.

```
Relational (SQL):  structured, ACID, joins, strong consistency -> the default for core data
NoSQL:             flexible/scalable/fast for specific needs    -> caching, big scale, documents
```

> Most backends use a relational DB (PostgreSQL) for core data **plus** NoSQL stores (especially Redis) for caching and specialized needs. This note covers NoSQL concepts and goes deep on Redis (used in Projects 4, 5, 7).

---

## 1. Why NoSQL? (What It Trades)

| Driver | Relational limitation | NoSQL approach |
|--------|----------------------|----------------|
| **Scalability** | Hard to scale writes horizontally | Designed for horizontal scaling/sharding |
| **Flexibility** | Rigid schema (ALTER TABLE) | Schema-less / flexible documents |
| **Speed** | Disk + ACID overhead | In-memory / eventual consistency |
| **Specific shapes** | Tables don't fit graphs/docs/time-series | Purpose-built models |
> ⚠️ NoSQL isn't "better" than SQL — it's **different trade-offs**. You often **lose** joins, strong consistency, and ACID transactions in exchange for scale/flexibility/speed. Use the right tool: relational for structured transactional data, NoSQL for caching, huge scale, or non-relational shapes. **"Polyglot persistence"** = using multiple stores in one system.

---

## 2. The CAP Theorem

The **CAP theorem** states that a distributed data store can guarantee **at most two** of these three:
| Property | Meaning |
|----------|---------|
| **C**onsistency | Every read sees the latest write (all nodes agree) |
| **A**vailability | Every request gets a (non-error) response |
| **P**artition tolerance | The system works despite network failures between nodes |
```
In a distributed system, network PARTITIONS are unavoidable -> you must tolerate P.
So the real choice is: when a partition happens, favor C or A?
  CP: stay consistent, sacrifice availability (reject requests until consistent)
  AP: stay available, sacrifice consistency (serve possibly-stale data)
```
> Since partitions happen in any distributed system, **P is mandatory** — so the practical trade-off is **C vs A**. Examples: a strongly-consistent store (CP) may reject writes during a partition; an always-available store (AP) may serve stale data. (Single-node relational DBs sidestep CAP — it's about *distributed* systems. **PACELC** extends this: even without partitions, there's a latency-vs-consistency trade-off — Phase 16.8.)

---

## 3. Types of NoSQL Databases

| Type | Model | Examples | Use for |
|------|-------|----------|---------|
| **Key-value** | Simple key → value | **Redis**, DynamoDB, Memcached | Caching, sessions, fast lookups |
| **Document** | JSON-like documents | **MongoDB**, CouchDB | Flexible/nested data, catalogs |
| **Column-family** | Wide rows by column families | Cassandra, HBase | Huge write-heavy, time-series |
| **Graph** | Nodes + edges | Neo4j | Relationships (social, recommendations) |
> Each type fits a data shape: **key-value** for fast lookups/caching, **document** for flexible nested data, **column-family** for massive scale, **graph** for relationship-heavy data (recall graphs, Phase 2.2).

---

## 4. Redis (You WILL Use This)

**Redis** ("REmote DIctionary Server") is an **in-memory** key-value store — extremely fast (microsecond latency, since data is in RAM — recall Phase 0.1) with rich data structures. It's the most-used NoSQL store in backend systems.

### 4.1 Why Redis is fast
- **In-memory:** data lives in RAM (Phase 0.1 — RAM is ~100,000× faster than disk).
- **Single-threaded** core (no lock contention for commands — atomic by nature).
- **Simple, efficient** data structures.
> Redis trades durability (in-memory) for speed — but offers persistence options (§4.4) for safety. It's not just a cache: it's a versatile data structure server.

### 4.2 Redis Data Structures (the key feature)
Redis isn't just string→string; it stores **rich data structures** as values:
| Structure | Description | Use case |
|-----------|-------------|----------|
| **String** | Text/number/binary | Caching, counters (`INCR`), flags |
| **List** | Ordered list (linked list) | Queues, recent-items, timelines |
| **Set** | Unique unordered members | Tags, unique visitors, membership |
| **Sorted Set (ZSet)** | Set ordered by score | **Leaderboards**, rankings, priority queues |
| **Hash** | Field→value map (like a small object) | Storing objects (user profile fields) |
| **Stream** | Append-only log | Event streams, messaging |
```
SET user:1:name "Alice"           # string
INCR page:views                    # atomic counter (string)
LPUSH queue:tasks "task1"          # list (push)
SADD tags:post:1 "java" "spring"   # set (unique tags)
ZADD leaderboard 100 "alice"       # sorted set (score 100)
ZREVRANGE leaderboard 0 9          # top 10 by score!
HSET user:1 name "Alice" age 30    # hash (object-like)
```
> ⭐ **Sorted sets** make Redis perfect for **leaderboards** (ranked by score, O(log n) — recall heaps/balanced trees, Phase 2.2). **Hashes** efficiently store object fields. This rich structure support is why Redis is far more than a cache.

### 4.3 Key Expiration & Eviction
Redis can **expire** keys automatically (TTL) and **evict** keys when memory is full:
```
SET session:abc "data" EX 3600     # expire in 3600 seconds (1 hour)
TTL session:abc                     # check remaining time
EXPIRE key 60                       # set/update TTL
```
**Eviction policies** (when `maxmemory` is reached):
| Policy | Evicts |
|--------|--------|
| **`allkeys-lru`** | Least Recently Used key (recall LRU, Phase 2.2) |
| **`allkeys-lfu`** | Least Frequently Used |
| `volatile-lru` / `volatile-ttl` | LRU / soonest-to-expire among keys with a TTL |
| `noeviction` | Reject writes when full (default) |
> ⚠️ TTL + LRU/LFU eviction make Redis ideal as a **cache** — old/cold data is automatically removed (this is the LRU/LFU cache concept from Phase 2.2, implemented at scale). Always set TTLs on cache entries so stale data expires.

### 4.4 Persistence (RDB & AOF)
Although in-memory, Redis can persist to disk for durability:
| Option | How | Trade-off |
|--------|-----|-----------|
| **RDB** (snapshot) | Periodic point-in-time snapshots | Fast, compact; can lose recent data on crash |
| **AOF** (Append-Only File) | Logs every write operation | More durable; larger, slower replay |
| **Both** | Snapshot + log | Best durability (common in production) |
> If you use Redis only as a cache, persistence may not matter (you can rebuild from the source DB). If it's a primary store, enable AOF (+RDB). (Recall WAL/durability, Phase 4.5 — AOF is similar in spirit.)

### 4.5 Redis Use Cases (the important part)
| Use case | How Redis helps |
|----------|-----------------|
| **Caching** | Store DB query results / computed values; fast reads, TTL eviction (Phase 5.7) |
| **Sessions** | Store user sessions externally → stateless app servers (scaling, Phase 13) |
| **Rate limiting** | `INCR` a counter per user/IP with a TTL (sliding window — Phase 2.3/12.3) |
| **Leaderboards** | Sorted sets, ranked by score |
| **Pub/Sub** | Lightweight messaging (publish/subscribe channels) |
| **Distributed locks** | Coordinate across servers (Redisson — Phase 12.6) |
| **Counters / analytics** | Atomic `INCR` for views, likes |
> ⭐ **Caching and session storage are the two most common Redis uses** in backends. External sessions (Redis) make app servers **stateless** → horizontally scalable behind a load balancer (recall Phase 0.2 LB statelessness, Phase 13). Rate limiting via `INCR`+TTL is the classic Redis pattern.

### 4.6 Redis Transactions & Lua Scripting
- **Transactions (`MULTI`/`EXEC`):** queue commands and execute them atomically (no other command runs in between). Not full ACID — no rollback on logic errors, but isolation is guaranteed.
```
MULTI            # start
INCR counter
INCR counter
EXEC             # execute all atomically
```
- **Lua scripting (`EVAL`):** run a script atomically on the server — for complex atomic operations (e.g., a correct rate limiter, check-and-set) without round-trip races.
> Lua scripts are the robust way to do **atomic multi-step operations** in Redis (e.g., "increment if under limit") — avoiding race conditions between separate commands.

### 4.7 Redis Cluster & Sentinel (scaling/HA)
| Feature | Purpose |
|---------|---------|
| **Replication** | Replicas copy the primary (read scaling, backup) |
| **Sentinel** | Monitors + automatic **failover** (high availability) |
| **Cluster** | **Sharding** — data partitioned across multiple nodes (horizontal scale) |
> **Sentinel** provides HA (failover if the primary dies); **Cluster** provides horizontal scaling by sharding keys across nodes (recall partitioning/sharding, Phase 4.6/16.6).

### 4.8 Spring Data Redis
Spring integrates Redis cleanly:
```java
@Autowired RedisTemplate<String, Object> redisTemplate;   // low-level operations

// Or use Spring Cache abstraction (Phase 5.7) backed by Redis:
@Cacheable("products")
public Product getProduct(Long id) { ... }   // result cached in Redis automatically
```
> **Spring Cache + Redis** (Phase 5.7) lets you add caching with annotations (`@Cacheable`, `@CacheEvict`) — Redis as the cache store. `RedisTemplate` gives direct access for data structures (counters, sorted sets). (Projects 4/5/7 use Redis for caching.)

---

## 5. MongoDB (Awareness)

**MongoDB** is the most popular **document database** — stores flexible **JSON-like documents** (BSON) instead of rows.
```javascript
// A document (no fixed schema; nested data is natural):
{ "_id": 1, "name": "Alice", "address": { "city": "NYC" }, "tags": ["a", "b"] }
```
| Concept | MongoDB | Relational equivalent |
|---------|---------|----------------------|
| Database | database | database |
| Collection | collection | table |
| Document | document (BSON) | row |
| Field | field | column |
- **CRUD:** `insertOne`, `find`, `updateOne`, `deleteOne`.
- **Aggregation pipeline:** multi-stage data processing (`$match`, `$group`, `$sort` — like SQL GROUP BY chained).
- **Flexible schema:** documents in a collection can differ — good for evolving/varied data.
> Use MongoDB when data is **document-shaped, nested, and schema-flexible** (catalogs, content, user-generated data). But ⚠️ don't reach for it just to "avoid schemas" — relational + JSONB (Phase 4.6) often covers flexible-data needs while keeping ACID/joins. (Spring Data MongoDB provides repositories like Spring Data JPA.)

---

## 6. SQL vs NoSQL — Choosing

| Use **relational (SQL)** when... | Use **NoSQL** when... |
|----------------------------------|------------------------|
| Structured data with relationships | Flexible/varied schema |
| You need ACID transactions, joins | Massive horizontal scale needed |
| Strong consistency required | Eventual consistency is acceptable |
| Core business/transactional data | Caching (Redis), documents (Mongo), graphs |
> **The common pattern: PostgreSQL for core data + Redis for caching/sessions** (polyglot persistence). Most systems are relational-first, adding NoSQL where it fits. Don't replace your relational DB with NoSQL without a clear reason.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| "NoSQL is always better/faster" | It's different trade-offs; relational is often right |
| Using Redis without TTLs | Set TTLs so cache entries expire (avoid memory bloat) |
| Treating Redis transactions as full ACID | No rollback on logic errors; use Lua for atomic logic |
| Expecting joins in NoSQL | Most NoSQL has no joins — model data differently |
| MongoDB just to "avoid schema" | Relational + JSONB often suffices (Phase 4.6) |
| Stateful app servers (in-memory sessions) | Externalize sessions to Redis (statelessness) |
| Ignoring CAP trade-offs in distributed stores | Understand C-vs-A under partitions |
| Race conditions in multi-step Redis ops | Use Lua scripts / atomic commands |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Spring Cache + Redis (Phase 5.7):** `@Cacheable`/`@CacheEvict` backed by Redis — the main caching mechanism.
- **Spring Session + Redis (Phase 13):** external sessions → stateless, horizontally-scalable servers (recall LB statelessness, Phase 0.2).
- **Rate limiting (Phase 12.3):** Redis `INCR`+TTL / Lua for API gateways and resilience.
- **Distributed locks (Redisson — Phase 12.6):** coordinate across microservices.
- **Caching is a top performance lever (Phase 13):** trade memory for speed (Phase 2.1 time-space trade-off).
- **CAP theorem** is foundational for **system design** (Phase 16.8) and microservices consistency (Phase 12).
- **Testcontainers** spins up real Redis/Mongo in integration tests (Phase 6).
- **Projects 4, 5, 7** use Redis for caching frequently-accessed data.

---

## 9. Quick Self-Check Questions

1. Why does NoSQL exist, and what do you typically trade for its benefits?
2. State the CAP theorem. Why is the real choice C vs A?
3. Name the four NoSQL types and a use for each.
4. Why is Redis so fast, and what makes it more than a cache?
5. Name Redis data structures and a use for sorted sets and hashes.
6. How do TTL and eviction policies make Redis a good cache?
7. What are RDB and AOF persistence?
8. List five Redis use cases. Why externalize sessions to Redis?
9. How do Redis transactions/Lua scripting provide atomicity?
10. When would you choose MongoDB, and when is relational + JSONB enough?

---

## 10. Key Terms Glossary

- **NoSQL:** non-relational databases (key-value/document/column/graph).
- **Polyglot persistence:** using multiple data stores per system.
- **CAP theorem:** Consistency, Availability, Partition tolerance (pick 2; P is mandatory).
- **PACELC:** CAP + latency-vs-consistency trade-off without partitions.
- **Redis:** in-memory key-value data-structure store.
- **Redis data structures:** string, list, set, sorted set, hash, stream.
- **Sorted set (ZSet):** score-ordered set (leaderboards).
- **TTL / eviction (LRU/LFU):** key expiration / removal when full.
- **RDB / AOF:** snapshot / append-only-log persistence.
- **MULTI/EXEC / Lua (EVAL):** Redis transactions / atomic scripting.
- **Sentinel / Cluster:** HA failover / sharding.
- **MongoDB / document / collection / aggregation pipeline:** document DB concepts.
- **Spring Data Redis / `RedisTemplate`:** Spring's Redis integration.

---

*This is the note for **Section 4.8 — NoSQL Databases**.*
*This completes **Phase 4 — Databases**.*
*Next: **Phase 5 — Spring Framework & Spring Boot**.*
