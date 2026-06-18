# Scalability

> **Phase 13 — Performance & Optimization → 13.3 Scalability**
> Goal: Scale a backend to handle growth — vertical vs horizontal scaling, stateless design, distributed sessions (Spring Session + Redis), database scaling (read replicas, sharding), and load balancing.

---

## 0. The Big Picture

**Performance** (13.1) makes one instance faster; **scalability** is the ability to handle **more load by adding resources**. The key insight: you scale by adding **machines** (horizontal), which requires your app to be **stateless** so any instance can serve any request. The database usually becomes the hardest thing to scale.

```
   1 instance, 100 RPS  ──add instances──►  10 instances, ~1000 RPS  (if STATELESS + load-balanced)
                                              │
                                  but the shared DATABASE is now the bottleneck → scale the DB (§4)
```

> Builds on horizontal scaling via Kubernetes HPA (10.2), stateless design (12.1/12.5), Redis (4.8), DB design/indexing (4.3/4.4), and is validated by load testing (13.2). Complements performance tuning (13.1) — you usually do both.

---

## 1. Vertical vs Horizontal Scaling

| | **Vertical (scale up)** | **Horizontal (scale out)** ⭐ |
|--|------------------------|------------------------------|
| How | Bigger machine (more CPU/RAM) | More machines/instances |
| Limit | Hardware ceiling (finite) | ~Unlimited (add nodes) |
| Cost | Expensive at the top end | Commodity hardware, linear-ish |
| Downtime | Often needs restart | Add/remove instances live |
| Fault tolerance | ❌ Single point of failure | ✅ Redundancy built in |
| Requires | Nothing special | ⭐ **Stateless app** (§2) |
| Tooling | Resize the VM/Pod (VPA — 10.2) | Add Pods (HPA — 10.2), load balancer |

```
   VERTICAL:   [small]──►[BIG]         (one powerful box, single point of failure)
   HORIZONTAL: [box]──►[box][box][box] (many boxes, redundant, behind a load balancer)
```
> ⭐ **Horizontal scaling is the cloud-native default** — it's elastic, fault-tolerant, and cost-effective (commodity nodes, autoscaling — 10.2 HPA). Vertical scaling is simpler but hits a ceiling and is a SPOF. ⚠️ Horizontal scaling **only works if the app is stateless** (§2).

---

## 2. Stateless Design ⭐ (the enabler)

A **stateless** service keeps **no client-specific state in its own memory** between requests — every request carries (or fetches) what it needs. Then **any instance can serve any request**, so a load balancer can freely distribute traffic and you can add/remove instances at will.
```
   STATEFUL (bad for scaling):  user's session/cart stored in Instance A's memory
        → load balancer must pin the user to A ("sticky sessions") → A dies = state lost, uneven load
   STATELESS (good):  session/cart stored in Redis/DB; ANY instance can handle ANY request
        → free load balancing, add/remove instances, instance death = no data loss
```
| Keep OUT of instance memory | Put it in |
|-----------------------------|-----------|
| HTTP sessions | **Redis** (Spring Session — §3) |
| Cart/wizard state | Redis / DB / client (token) |
| Auth state | **Stateless JWT** (Phase 5.6) — no server session! |
| Config/secrets | External (Phase 12.5) |
| Files | Object storage (S3 — Phase 16.7) |
| Cache | Distributed (Redis — 13.1 §5) |
> ⭐ **Statelessness is the #1 prerequisite for horizontal scaling** (and K8s HPA — 10.2). This is why **JWT** (Phase 5.6) is preferred over server sessions in microservices — auth state travels in the token, not server memory. Recall REST's *stateless* constraint (Phase 7.1 §0).

---

## 3. Distributed Sessions (Spring Session + Redis)

When you *do* need server-side sessions (legacy/stateful flows) but want to scale horizontally, **externalize the session** to a shared store so all instances see it.
```
   Instance A ─┐
   Instance B ─┼──► [ Redis ] (shared session store)  ◄ any instance reads/writes the session
   Instance C ─┘
```
**Spring Session** transparently stores `HttpSession` in Redis (or JDBC/Hazelcast) instead of instance memory:
```xml
<dependency><groupId>org.springframework.session</groupId>
  <artifactId>spring-session-data-redis</artifactId></dependency>
```
```yaml
spring.session.store-type: redis   # HttpSession now lives in Redis → instances are stateless
```
> ⭐ Spring Session + Redis lets you keep the convenient `HttpSession` API **and** scale horizontally — no sticky sessions needed. ⚠️ Still, for new microservices, **prefer stateless JWT** (Phase 5.6) to avoid a session store entirely. Sticky sessions (load-balancer affinity) are a fragile workaround — avoid.

---

## 4. Database Scaling ⭐ (the hard part)

Stateless app instances scale easily; the **shared database** becomes the bottleneck. Strategies (increasing complexity):
| Strategy | How | Trade-off |
|----------|-----|-----------|
| **Vertical (bigger DB)** | More CPU/RAM/IOPS | Simple; hits a ceiling |
| **Connection pooling** | Reuse connections (HikariCP — 13.1 §3) | Necessary baseline |
| **Caching** | Offload reads to Redis (13.1 §5) | Reduces DB load hugely |
| **Read replicas** ⭐ | Replicate primary → read-only copies; route reads to replicas, writes to primary | Scales **reads**; ⚠️ **replication lag** = eventual consistency on reads |
| **Sharding (partitioning)** ⭐ | Split data across DBs by a shard key (e.g., `customerId % N`) | Scales **writes** + storage; ⚠️ complex (cross-shard queries/joins hard, rebalancing) |
| **CQRS read models** | Separate read store (Phase 11.4) | Scales reads, denormalized |
| **NewSQL / distributed DB** | CockroachDB, Spanner (Phase 16.6) | Horizontal SQL; operational complexity |

### 4.1 Read replicas
```
   writes ──► [ Primary DB ] ──replicate──► [ Replica 1 ] ◄── reads
                              └────────────► [ Replica 2 ] ◄── reads
   Most apps are read-heavy → routing reads to replicas relieves the primary
```
> ⭐ **Read replicas** are the first DB scaling step for read-heavy systems. ⚠️ **Replication lag**: a read replica may be slightly behind the primary → "read-your-own-writes" can show stale data (route critical reads to the primary, or accept eventual consistency — Phase 11.4).

### 4.2 Sharding
```
   shard key = customerId
   customers 0–999  → DB shard A    customers 1000–1999 → DB shard B  ...
```
> ⚠️ **Sharding is powerful but the last resort** — it makes cross-shard queries/joins/transactions hard, complicates the app, and rebalancing is painful. Choose the **shard key** carefully (even distribution, queries stay within a shard). Exhaust caching, replicas, and CQRS first. (Deeper in Phase 16.6.)

---

## 5. Load Balancing

A **load balancer (LB)** distributes incoming requests across instances — the front of horizontal scaling.
| Aspect | Note |
|--------|------|
| **Algorithms** | Round-robin, least-connections, weighted, IP-hash |
| **Layer 4 vs Layer 7** | TCP-level vs HTTP-aware (path/host routing — 12.4/10.2) |
| **Health checks** | Route only to healthy instances (readiness — 9.2/10.2) |
| **In Kubernetes** | **Service** load-balances across Pods (kube-proxy — 10.2); **Ingress/Gateway** at L7 (10.2/12.4) |
| ⚠️ Sticky sessions | Affinity to one instance — avoid; be stateless instead (§2) |
> ⭐ On Kubernetes you get LB "for free" via **Services** (10.2) + **HPA** (autoscaling). The LB only works well if instances are **interchangeable** (stateless §2) and **health-checked** (9.2).

---

## 6. Scalability Principles & Limits

- ⭐ **Find the bottleneck** — adding app instances does nothing if the **DB** (or a shared lock, or a downstream service) is the limit. Scale the actual constraint.
- **Async/decoupling** (Phase 11) absorbs spikes (load leveling) and decouples scaling per component.
- **Autoscaling** (HPA — 10.2) on metrics (9.2) — scale out under load, in when idle (cost).
- **Caching** (13.1) is often the cheapest scale win.
- ⚠️ **Amdahl/Universal Scalability Law:** scaling isn't infinite/linear — contention and coordination (locks, shared DB) cap it. Reduce shared state.
- **Validate with load tests** (13.2): does throughput actually rise as you add instances?
> ⭐ Scalability = **stateless app + horizontal scaling + scaled data tier + load balancing + async**, proven by load testing. The DB and shared state are almost always the real limits.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Stateful instances → can't scale out | Stateless design (§2); Redis sessions / JWT (§3) |
| Sticky sessions as a scaling strategy | Externalize state; be stateless (§2/§3) |
| Scaling app instances while DB is the bottleneck | Scale the DB (replicas/cache/CQRS/shard) (§4) |
| Ignoring replication lag | Route critical reads to primary; accept eventual consistency (§4.1) |
| Sharding too early | Cache/replicas/CQRS first; shard last (§4.2) |
| Bad shard key (hot shards) | Choose even-distribution key; keep queries in-shard (§4.2) |
| Vertical scaling forever (SPOF, ceiling) | Horizontal scaling + redundancy (§1) |
| LB routing to unhealthy instances | Health checks/readiness (§5, 9.2) |
| Assuming linear scaling | Mind contention/coordination limits (§6) |
| Not validating scaling | Load test (§6, 13.2) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Stateless design** ↔ REST statelessness (7.1), **JWT** (5.6), external config (12.5).
- **Horizontal scaling** via **K8s HPA** (10.2) on **Prometheus metrics** (9.2).
- **Spring Session + Redis** / distributed cache (4.8, 13.1).
- **DB scaling**: replicas/sharding (Phase 4, Phase 16.6 advanced), CQRS read models (11.4).
- **Load balancing** via K8s Services/Ingress/Gateway (10.2/12.4).
- **Async** (Phase 11) for load leveling; **resilience** (12.3) under spikes.
- **Validated by load testing** (13.2); complements **performance** (13.1).
- Central to **Project 7 (Microservices E-Commerce)** at scale.

---

## 9. Quick Self-Check Questions

1. Vertical vs horizontal scaling — trade-offs, and why is horizontal the cloud-native default?
2. What does "stateless" mean, and why is it the prerequisite for horizontal scaling?
3. What state must you move out of instance memory, and where?
4. How does Spring Session + Redis enable scaling, and why prefer JWT for new services?
5. Why does the database become the bottleneck, and what are the scaling options in order?
6. How do read replicas scale reads, and what is replication lag's consequence?
7. What is sharding, why is it a last resort, and how do you choose a shard key?
8. What does a load balancer do, and why must instances be stateless + health-checked?
9. Why must you scale the actual bottleneck, and why isn't scaling infinite/linear?
10. How do you validate that your system actually scales?

---

## 10. Key Terms Glossary

- **Scalability:** handling more load by adding resources.
- **Vertical (scale up) vs horizontal (scale out):** bigger machine vs more machines.
- **Stateless / stateful:** no per-client memory between requests vs holds state.
- **Sticky session (affinity):** pinning a user to one instance (avoid).
- **Spring Session:** externalize `HttpSession` to Redis/JDBC.
- **Read replica / replication lag:** read-only DB copy / staleness behind primary.
- **Sharding / shard key:** partition data across DBs / the partitioning attribute.
- **CQRS read model:** separate scalable read store (Phase 11.4).
- **Load balancer (L4/L7, round-robin/least-conn):** distributes requests.
- **HPA / autoscaling:** add/remove instances on metrics (10.2).
- **Universal Scalability Law:** contention/coordination cap scaling.

---

*This is the note for **Section 13.3 — Scalability**.*
*Previous section in roadmap: **13.2 Load Testing**.*
*This completes **Phase 13 — Performance & Optimization**.*
*Next: **Phase 14 — Design Patterns & Architecture**.*
