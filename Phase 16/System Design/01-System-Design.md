# System Design

> **Phase 16 — Advanced Topics → 16.8 System Design**
> Goal: Tie the whole roadmap together — a methodology for designing large-scale systems, the key building-block concepts, classic design problems, back-of-the-envelope estimation, and non-functional requirements (NFRs).

---

## 0. The Big Picture

**System design** is the capstone: composing everything in this roadmap (databases, caching, messaging, microservices, scaling, security, observability) into a coherent architecture that meets requirements at scale. It's also the format of senior **interviews** ("design a URL shortener / news feed / chat"). The skill is **reasoning about trade-offs** under constraints — there's rarely one right answer.

```
   Requirements ──► high-level design ──► deep-dive components ──► identify bottlenecks
        │                                                              │
        └────────── iterate, justify trade-offs (CAP, consistency, cost) ◄┘
```

> This synthesizes the entire roadmap: Phase 4/16.6 (data), 13 (perf/scale), 11 (messaging), 12 (microservices), 9 (observability), 15 (security), 14 (architecture). It's where you *combine* concepts, not learn new ones.

---

## 1. A System Design Methodology ⭐

A structured approach (great for interviews *and* real design):
| Step | Do |
|------|-----|
| **1. Clarify requirements** ⭐ | Functional (features) + non-functional (scale, latency, consistency — §4). **Ask questions**; don't assume |
| **2. Estimate scale** | Back-of-envelope: users, QPS, storage, bandwidth (§3) |
| **3. Define the API** | Key endpoints/contracts (REST/gRPC — Phase 7/12.2) |
| **4. Data model** | Entities, access patterns → choose DB(s) (Phase 4/16.6) |
| **5. High-level design** | Boxes & arrows: clients → LB/gateway → services → data/cache/queue |
| **6. Deep dive** | Detail the critical components/bottlenecks |
| **7. Identify & resolve bottlenecks** | Scaling, caching, sharding, async (Phase 13/11) |
| **8. Address NFRs** | Availability, consistency, security, observability (§4) |
> ⭐ **Start broad, then deep.** The biggest mistake is jumping to details before clarifying **requirements and scale**. State assumptions, drive the conversation, and **justify every choice with a trade-off** ("I'd use a cache here to cut read latency, accepting some staleness").

---

## 2. Key Building Blocks (the toolkit) ⭐

System design = assembling components you already know:
| Component | Role | Roadmap |
|-----------|------|---------|
| **Load balancer** | Distribute traffic | 13.3/12.4/16.7 |
| **API Gateway** | Edge: routing, auth, rate limit | 12.4 |
| **Stateless app servers** | Scale horizontally | 13.3 |
| **Database (SQL/NoSQL)** | Source of truth; replicate/shard | 4/16.6 |
| **Cache (Redis)** | Cut read latency/DB load | 5.7/4.8/13.1 |
| **CDN** | Edge-cache static content | 13.1/16.7 |
| **Message queue / Kafka** | Async, decouple, buffer, fan-out | 11 |
| **Search (Elasticsearch)** | Full-text search | 16.4 |
| **Object storage (S3)** | Files/blobs | 16.7 |
| **Observability** | Logs/metrics/traces | 9/12.7 |
```
   Client → CDN → LB → API Gateway → [stateless services] → Cache → DB (replicas/shards)
                                            │
                                            └──► Queue/Kafka → async workers → DB/search
```
> ⭐ Internalize this **reference architecture** — most large systems are variations of it. Know *when* to add each block: cache when reads dominate, queue when you need decoupling/buffering, replicas/shards when the DB is the bottleneck, CDN for static/global content.

---

## 3. Back-of-the-Envelope Estimation ⭐

Quick math to size the system and spot bottlenecks. Memorize the rough numbers.
| Quantity | Rough number |
|----------|--------------|
| Seconds/day | ~86,400 (~10^5) |
| 1 million writes/day | ~12 writes/sec average (×peak factor) |
| **Latency: memory** | ~100 ns |
| **Latency: SSD** | ~100 µs |
| **Latency: network round-trip (same DC)** | ~0.5 ms |
| **Latency: cross-region** | ~50–150 ms |
| **Latency: disk seek (HDD)** | ~10 ms |
```
   QPS:  Daily-Active-Users × actions/user/day ÷ 86,400  × peak-multiplier (e.g., 2–10×)
   Storage:  records/day × bytes/record × retention-days  (× replication factor)
   Bandwidth:  QPS × payload size
```
> ⭐ Estimation tells you **what kind of system you're building**: 100 QPS fits one box; 1M QPS needs sharding, caching, and async everywhere. The **latency numbers** ("memory ≫ SSD ≫ network ≫ disk; cross-region is expensive") justify caching (memory vs DB), CDNs (avoid cross-region), and avoiding chatty calls (12.2). Round aggressively — it's about **orders of magnitude**, not precision.

---

## 4. Non-Functional Requirements (NFRs) ⭐

The "-ilities" — often what actually shapes the design (more than the features):
| NFR | Concern | Roadmap |
|-----|---------|---------|
| **Scalability** | Handle growth | 13.3 |
| **Availability** | Uptime (e.g., 99.9% = ~43 min/month down) | replicas/multi-AZ (16.6/16.7) |
| **Latency / Performance** | Response time (p95/p99) | 13.1/13.2 |
| **Consistency** | Strong vs eventual (CAP) | 4.5/11.4/16.6 |
| **Reliability / Fault tolerance** | Survive failures | 12.3 (resilience) |
| **Durability** | Don't lose data | replication/backups (16.6) |
| **Security** | Protect data/access | Phase 15 |
| **Observability** | Operate/debug | 9/12.7 |
| **Maintainability / Cost** | Evolve / budget | 14/16.7 |
> ⭐ ⚠️ **NFRs drive the hard trade-offs.** The classic tension: **consistency vs availability** (CAP — 16.6): a banking ledger needs strong consistency (CP); a social feed prefers availability + eventual consistency (AP, 11.4). **Availability targets** ("how many 9s?") dictate redundancy/cost. Always ask which NFRs **matter most** for *this* system — you can't max them all.

---

## 5. Classic Design Problems (patterns) ⭐

Common interview/real problems and their key ideas:
| Problem | Key concepts |
|---------|--------------|
| **URL shortener** | Hashing/base62 ids, read-heavy → cache (13.1), KV store, 301 redirects (7.1) |
| **News feed / timeline** | Fan-out on write vs read, cache, CQRS read model (11.4), pagination (7.1) |
| **Chat / messaging** | WebSocket (16.3), message queue (11), presence, ordering, delivery semantics (11.1) |
| **Rate limiter** | Token bucket + Redis (7.1/12.4) |
| **Notification system** | Pub/sub fan-out (11), queues, idempotency (7.1) |
| **E-commerce checkout** | Saga + Outbox (11.4/12.6), idempotency, inventory consistency (4.5) |
| **Distributed ID generation** | Snowflake ids, UUIDs (avoid hot shards — 16.6) |
| **Search/autocomplete** | Elasticsearch / tries (16.4) |
| **Video/file platform** | Object storage + CDN + pre-signed URLs (16.7/7.1) |
> ⭐ Notice these are **recombinations of the same building blocks** (§2) and patterns from earlier phases. The "celebrity/hot-key problem" (fan-out, sharding), **read-heavy vs write-heavy**, and **consistency requirements** recur constantly. Learn the building blocks well and the problems become composition exercises.

---

## 6. System Design Principles & Trade-offs

- ⭐ **Every choice is a trade-off** — articulate what you gain and give up (latency vs consistency, cost vs availability, simplicity vs flexibility).
- **There is no perfect design** — only the best fit for the requirements/constraints.
- **Design for failure** (12.3) and **for scale you actually need** (don't over-engineer — 14.3 KISS/YAGNI; start with a modular monolith — 12.1).
- **Bottleneck thinking**: find and address the limiting resource (usually the DB — 13).
- **Iterate**: a good design evolves; start simple, scale when measured (13.2).
> ⭐ Senior engineering is **judgment under trade-offs**. The roadmap gave you the components and patterns; system design is using them wisely for the problem at hand — and **knowing what *not* to build**.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Designing before clarifying requirements/scale | Clarify functional + NFRs first (§1) |
| Skipping estimation | Back-of-envelope to size the system (§3) |
| Over-engineering for imaginary scale | Build for real needs; iterate (§6, 14.3) |
| Ignoring NFRs (only features) | NFRs drive the design (§4) |
| Claiming "strongly consistent AND highly available" under partition | Respect CAP trade-offs (§4, 16.6) |
| One-size DB for everything | Polyglot persistence per access pattern (§2, 16.6) |
| Not identifying the bottleneck | Bottleneck thinking (usually the DB) (§6, 13) |
| No failure/observability story | Design for failure + observability (§4, 12.3/9) |
| Not justifying choices | Articulate trade-offs for every decision (§6) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Synthesizes the whole roadmap**: data (Phase 4/16.6), perf/scale (13), messaging (11), microservices (12), security (15), observability (9), architecture (14).
- **APIs** (Phase 7), **caching** (5.7/13.1), **resilience** (12.3), **CAP/consistency** (16.6/11.4), **cloud** (16.7) are the design vocabulary.
- The reasoning skill behind building **Projects 5–7** end-to-end.
- The format of senior **system design interviews** — and the daily job of an architect.

---

## 9. Quick Self-Check Questions

1. What are the steps of a system design methodology, and why clarify requirements first?
2. What are the standard building blocks, and when do you add a cache / queue / replicas / CDN?
3. What latency numbers should you know, and how do they justify caching/CDNs?
4. How do you estimate QPS, storage, and bandwidth?
5. What are NFRs, and why do they drive the design more than features?
6. How does the consistency-vs-availability trade-off (CAP) shape different systems?
7. What does "99.9% availability" mean in downtime?
8. Pick a classic problem (feed/chat/URL shortener) and outline its key concepts.
9. Why is "every choice is a trade-off" the core mindset?
10. How does system design synthesize the rest of the roadmap?

---

## 10. Key Terms Glossary

- **System design:** composing components into a scalable architecture meeting requirements.
- **Functional vs non-functional requirements (NFRs):** features vs "-ilities" (scale/availability/latency/consistency).
- **Back-of-the-envelope estimation:** rough QPS/storage/bandwidth/latency math.
- **Building blocks:** LB, gateway, stateless services, DB, cache, CDN, queue, search, object storage, observability.
- **Availability (nines):** uptime target (99.9% ≈ 43 min/month down).
- **Consistency vs availability (CAP):** the core distributed trade-off (16.6).
- **Fan-out (on write/read):** push vs pull for feeds.
- **Bottleneck thinking:** find and scale the limiting resource.
- **Trade-off:** what you gain vs give up in a decision.
- **Polyglot persistence:** right database per workload.

---

*This is the note for **Section 16.8 — System Design**.*
*Previous section in roadmap: **16.7 Cloud / AWS**.*
*This completes **Phase 16 — Advanced Topics** — and the entire roadmap (Phases 0–16)! 🎉*
