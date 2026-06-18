# Advanced Database (Replication, Sharding, NewSQL, Time-Series, DBaaS)

> **Phase 16 — Advanced Topics → 16.6 Advanced Database**
> Goal: Understand database scaling and specialization at depth — replication, sharding/partitioning, the CAP theorem, NewSQL & distributed SQL, time-series databases, and managed databases (DBaaS).

---

## 0. The Big Picture

The database is usually the **hardest thing to scale** (recall 13.3) and the most varied. Beyond the relational fundamentals (Phase 4) and basic scaling (13.3), production systems use **replication** for availability/read scaling, **sharding** for write/storage scaling, **specialized databases** (time-series, NewSQL) for specific workloads, and **managed services (DBaaS)** to offload operations. Underpinning the trade-offs is the **CAP theorem**.

```
   Scale reads ───────► Replication (primary + read replicas)
   Scale writes/storage ► Sharding (partition by key)
   Horizontal SQL ──────► NewSQL / Distributed SQL (Spanner, CockroachDB)
   Time-stamped data ───► Time-series DB (TimescaleDB, InfluxDB)
   Offload ops ─────────► DBaaS (RDS, Aurora, Cloud SQL)
```

> Extends DB fundamentals (Phase 4), indexing (4.4), transactions/consistency (4.5), and scalability (13.3). CAP underlies eventual consistency (11.4) and NoSQL choices (4.8). DBaaS relates to cloud (16.7).

---

## 1. Replication ⭐

Maintaining **copies** of data across servers for **availability** (failover) and **read scaling** (recall read replicas — 13.3 §4.1).
| Mode | How | Trade-off |
|------|-----|-----------|
| **Primary–Replica (leader–follower)** ⭐ | Writes to primary → replicated to read-only replicas | Scales reads; ⚠️ replication lag (stale reads) |
| **Synchronous replication** | Primary waits for replica ack before commit | Strong consistency; higher write latency |
| **Asynchronous replication** | Primary commits, replicates after | Low latency; risk of data loss on failover |
| **Multi-primary (multi-master)** | Multiple writable nodes | Write availability; ⚠️ conflict resolution complexity |
| **Failover** | Promote a replica when primary dies | Need automation (managed by DBaaS — §6) |
> ⭐ **Primary–replica with read replicas is the standard first scaling step** for read-heavy systems. ⚠️ **Replication lag** means replicas can be slightly behind → route read-your-own-writes to the primary or accept eventual consistency (13.3/11.4). Sync replication = safer but slower; async = faster but can lose recent writes on failover.

---

## 2. Sharding / Partitioning ⭐

Splitting data **horizontally** across multiple databases/nodes to scale **writes and storage** beyond one machine (recall 13.3 §4.2).
| Concept | Meaning |
|---------|---------|
| **Shard key** ⭐ | The attribute that decides which shard a row lives in (e.g., `customerId`) |
| **Range sharding** | By value ranges (ids 0–999, 1000–1999) — ⚠️ hotspots if uneven |
| **Hash sharding** | `hash(key) % N` — even distribution, but range queries scatter |
| **Directory/lookup** | A map of key → shard (flexible, extra hop) |
| **Vertical partitioning** | Split columns/tables by feature (vs horizontal rows) |
```
   shard key = hash(customerId) % 4 → distributes customers across 4 shards evenly
```
| Challenge | Note |
|-----------|------|
| Cross-shard queries/joins | ⚠️ Hard/expensive — must gather from many shards |
| Cross-shard transactions | ⚠️ Lose single-DB ACID → distributed tx/Saga (11.4) |
| Rebalancing | Adding shards = moving data (painful) |
| Hotspots | A bad shard key concentrates load (celebrity problem) |
> ⚠️ ⭐ **Sharding is the last-resort scaling lever** — it sacrifices easy joins, cross-shard transactions, and operational simplicity. Choose a shard key that **distributes evenly** and keeps common queries **within one shard**. Exhaust caching (13.1), read replicas (§1), and CQRS (11.4) first. (NewSQL — §4 — hides much of this.)

---

## 3. CAP Theorem ⭐

In a distributed data store, during a **network partition (P)** you can guarantee at most **two** of: **Consistency, Availability, Partition tolerance.** Since partitions *will* happen, the real choice under partition is **C vs A**.
```
   C - Consistency:  every read sees the latest write (or an error)
   A - Availability: every request gets a (non-error) response
   P - Partition tolerance: works despite network splits  ← non-negotiable in distributed systems
   → During a partition: choose CONSISTENCY (refuse/err) OR AVAILABILITY (serve possibly-stale)
```
| System leans | Behavior under partition | Examples |
|--------------|--------------------------|----------|
| **CP** | Stay consistent, sacrifice availability (some nodes refuse) | Traditional RDBMS clusters, MongoDB (default), HBase, ZooKeeper |
| **AP** | Stay available, allow stale/eventual consistency | Cassandra, DynamoDB, Riak |
> ⭐ CAP is about behavior **during a partition** (normally you get both C and A). **PACELC** extends it: Else (no partition) you trade **Latency vs Consistency**. This is *why* eventual consistency (11.4) exists — distributed systems often choose A + low latency over strict C. Pick a database whose CAP stance matches your needs (a banking ledger wants CP; a shopping cart can be AP).

---

## 4. NewSQL & Distributed SQL ⭐

The historical dilemma: **RDBMS** (ACID, SQL, joins — but hard to scale horizontally) vs **NoSQL** (scalable — but gives up ACID/SQL, 4.8). **NewSQL / Distributed SQL** aims for **both**: horizontal scalability **and** ACID transactions with SQL.
| System | Note |
|--------|------|
| **Google Spanner** | Globally-distributed, strongly consistent SQL (uses synchronized clocks/TrueTime) |
| **CockroachDB** ⭐ | Open-source, Postgres-compatible, distributed SQL, survives node/region failures |
| **YugabyteDB / TiDB** | Distributed SQL, Postgres/MySQL-compatible |
| **Vitess** | Sharded MySQL (powers large-scale MySQL) |
> ⭐ **NewSQL gives you horizontal scaling without manual sharding (§2)** — the database distributes/rebalances data and provides distributed ACID transactions, while you write normal SQL. The cost is operational complexity and sometimes higher latency for distributed transactions. A compelling option when you outgrow a single RDBMS but want to keep SQL/ACID.

---

## 5. Specialized Databases (Time-Series & others)

| DB type | Optimized for | Examples |
|---------|---------------|----------|
| **Time-series (TSDB)** ⭐ | Timestamped data (metrics, IoT, events) — fast time-range writes/queries, downsampling, retention | **TimescaleDB** (Postgres ext.), **InfluxDB**, Prometheus (9.2) |
| **Document** | JSON documents (4.8) | MongoDB |
| **Key-value** | Simple fast lookups, cache (4.8) | Redis, DynamoDB |
| **Wide-column** | Huge write throughput, AP (§3) | Cassandra |
| **Graph** | Relationships/traversals | Neo4j |
| **Search** | Full-text search (16.4) | Elasticsearch |
| **Vector** | Similarity search (AI/embeddings) | pgvector, Pinecone |
> ⭐ **Time-series databases** are purpose-built for append-heavy, time-stamped data (metrics — 9.2, IoT, financial ticks): they handle high write rates, time-bucketed aggregations, downsampling, and automatic retention far better than a generic RDBMS. **Polyglot persistence** = using the right database per workload (each microservice/bounded context — 12.1 — can choose its own).

---

## 6. Managed Databases (DBaaS)

**Database-as-a-Service** offloads operations (provisioning, patching, backups, replication, failover, scaling) to a cloud provider.
| DBaaS | Note |
|-------|------|
| **AWS RDS** | Managed Postgres/MySQL/etc. (16.7) |
| **AWS Aurora** ⭐ | Cloud-native MySQL/Postgres-compatible; auto-scaling storage, fast replicas |
| **Google Cloud SQL / Spanner** | Managed SQL / distributed SQL |
| **Azure SQL / Cosmos DB** | Managed SQL / multi-model |
| **MongoDB Atlas, Redis Cloud** | Managed NoSQL/cache |
> ⭐ **DBaaS handles the hard operational parts** — automated backups (Phase 8 migrations still yours), point-in-time recovery, replication/failover, encryption-at-rest (15.3), and scaling — so you focus on the data model. ⚠️ Trade-offs: cost, some loss of control/tuning, and vendor lock-in. The default for most teams now (you rarely self-manage production databases).

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Sharding too early | Cache/replicas/CQRS first; shard last (§2, 13.3) |
| Bad shard key → hotspots | Even-distribution key; keep queries in-shard (§2) |
| Reading replicas expecting fresh data | Replication lag → route critical reads to primary (§1) |
| Thinking CAP lets you "have all three" | Under partition, choose C or A (§3) |
| NoSQL chosen for "scale" without need | RDBMS/NewSQL may fit better (§4, 4.8) |
| Generic RDBMS for high-rate metrics | Use a time-series DB (§5) |
| One DB for everything | Polyglot persistence per workload (§5) |
| Self-managing prod DBs needlessly | Use DBaaS (§6) |
| Forgetting GDPR/erasure across replicas/shards | Plan deletion everywhere (15.3) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- Extends DB fundamentals (Phase 4), indexing (4.4), transactions/consistency (4.5), NoSQL (4.8), scalability (13.3).
- **CAP/eventual consistency** underlies async patterns (11.4) and microservices data ownership (12.1).
- **Time-series** ↔ metrics/Prometheus (9.2); **search** ↔ Elasticsearch (16.4).
- **DBaaS** ↔ cloud (16.7); encryption-at-rest (15.3); migrations still via Flyway/Liquibase (Phase 8).
- **Polyglot persistence** ↔ per-service DBs (12.1); sharding/replicas tie to scaling (13.3).
- Relevant to scaling **Project 7** at large data volumes.

---

## 9. Quick Self-Check Questions

1. What does replication achieve, and what is replication lag's consequence?
2. Sync vs async vs multi-primary replication — trade-offs?
3. What is sharding, how do range/hash sharding differ, and what makes a good shard key?
4. Why is sharding a last resort, and what does it sacrifice?
5. State the CAP theorem; what's the real choice during a partition? What does PACELC add?
6. Give an example of a CP vs an AP system.
7. What is NewSQL/Distributed SQL, and what problem does it solve?
8. What are time-series databases optimized for, and when use one?
9. What is polyglot persistence?
10. What does DBaaS offload, and what are its trade-offs?

---

## 10. Key Terms Glossary

- **Replication (primary–replica, sync/async, multi-primary):** copies for HA/read scaling.
- **Replication lag / failover:** staleness on replicas / promoting a replica.
- **Sharding / shard key / range vs hash:** horizontal partitioning of data.
- **CAP theorem (CP/AP) / PACELC:** consistency-availability-partition trade-offs.
- **NewSQL / Distributed SQL:** horizontally-scalable ACID SQL (Spanner, CockroachDB).
- **Time-series database (TSDB):** optimized for timestamped data.
- **Polyglot persistence:** right database per workload.
- **DBaaS / RDS / Aurora:** managed database services.
- **Vector / graph / wide-column / document DB:** specialized stores.

---

*This is the note for **Section 16.6 — Advanced Database**.*
*Previous section in roadmap: **16.5 GraalVM / Native Image**.*
*Next section in roadmap: **16.7 Cloud / AWS**.*
