# Distributed Patterns (Saga, Outbox, CQRS, Event Sourcing, Locking, Idempotency)

> **Phase 12 — Microservices → 12.6 Distributed Patterns**
> Goal: Consolidate the key distributed-systems patterns for microservices — Saga, the Outbox pattern with Debezium, CQRS, Event Sourcing, distributed locking with Redisson, and idempotency.

---

## 0. The Big Picture

This section gathers the patterns that solve the hard problems of microservices: **maintaining consistency** across services that each own a database (12.1), **reliably publishing events**, **scaling reads**, and **coordinating** concurrent work — all without a global transaction. Most were introduced as **async patterns in Phase 11.4**; here we frame them explicitly as the microservices toolkit and add **distributed locking** and consolidate **idempotency**.

```
   Cross-service consistency ──► Saga (+ compensation)
   Reliable event publishing ──► Outbox (+ Debezium CDC)
   Scale/segregate reads ─────► CQRS (+ Event Sourcing for history)
   Coordinate concurrency ────► Distributed locking (Redisson)
   Survive duplicate delivery ► Idempotency (keys / Inbox)
```

> Deep mechanics live in **Phase 11.4** (Saga/Outbox/Inbox/CQRS/Event Sourcing) and **Phase 7.1** (idempotency keys); this note ties them to microservices and adds distributed locking. Builds on Kafka/RabbitMQ (11.2/11.3), transactions (4.5), Redis (4.8), and resilience (12.3).

---

## 1. Saga (distributed transaction)

A business transaction across services = a sequence of **local transactions**, each emitting an event that triggers the next; failures trigger **compensating transactions** (semantic undo). No 2PC. (Full detail — **Phase 11.4 §4**.)
```
   Order → Payment → Inventory → Shipping
   step fails → compensate completed steps in reverse (refund, cancel) → consistent
```
| Style | Note |
|-------|------|
| **Orchestration** | Central coordinator drives steps (visible, controllable) — Camunda/Temporal/state machine |
| **Choreography** | Services react to events, no central brain (loose, can be "event spaghetti") |
> ⭐ Recall (11.4): every step + compensation must be **idempotent** (§5); choose orchestration for complex flows, choreography for simple ones. This is the answer to "how do I do a transaction across services?"

---

## 2. Outbox Pattern + Debezium ⭐

Solves the **dual-write problem**: updating the DB and publishing an event can't be atomic across two systems. Write the event into an **outbox table in the same DB transaction**, then a relay publishes it. (Full detail — **Phase 11.4 §5**.)
```
   ┌ one local tx (Phase 4.5) ┐
   │ INSERT order             │
   │ INSERT outbox(event)     │ ──committed together──► Debezium CDC (11.2 §8) ──► Kafka ──► consumers
   └──────────────────────────┘
```
- **Debezium** (Kafka Connect CDC — 11.2 §8) tails the DB transaction log and turns committed outbox rows into events → no polling, no lost events.
- Guarantees **event published iff DB committed**; at-least-once → consumers idempotent (§5).
> ⭐ Outbox + Debezium is *the* reliable way to publish events from a microservice. Pair with the **Inbox** pattern (11.4 §6) on the consumer side for end-to-end **effectively-once**.

---

## 3. CQRS & Event Sourcing (in microservices)

- **CQRS** (11.4 §2): separate write & read models. In microservices, a read model can be a **denormalized view** maintained from other services' events → answers cross-service queries **without runtime fan-out** (recall API composition vs read model — 12.2 §6).
- **Event Sourcing** (11.4 §3): store events as the source of truth; rebuild state/projections by replay. Gives audit trail + the ability to build new read models retroactively.
```
   Write side (commands) ──events──► [ Kafka ] ──► Read side projection (denormalized, fast)
   Query side reads the projection → no calling 5 services at query time
```
> ⭐ Use **CQRS read models** to defeat the "chatty aggregation" problem (12.2). Use **Event Sourcing** only where history/auditability justifies its complexity (11.4 §3). They're independent but synergistic.

---

## 4. Distributed Locking (Redisson) ⭐

Within one JVM you use `synchronized`/`Lock` (Phase 1.10). Across **multiple instances** (10.2) of a service, in-process locks don't work — you need a **distributed lock** so only one instance does a critical action at a time (e.g., run a scheduled job once, prevent concurrent processing of the same entity).
```
   Instance A ──acquire lock "order:42"──► [ Redis ]  ◄── Instance B (blocked / skips)
        do the critical section
   ──release──► (or auto-expire via TTL)
```
**Redisson** is a Redis Java client offering a distributed `RLock` (Redis-backed):
```java
RLock lock = redisson.getLock("order:" + orderId);
if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {   // wait up to 5s, hold up to 30s (auto-expire)
    try { processOnce(orderId); }              // only one instance enters
    finally { lock.unlock(); }
}
```
| Concern | Guidance |
|---------|----------|
| **TTL / lease** | Lock auto-expires so a crashed holder doesn't deadlock forever |
| **Correctness** | Naive Redis locks have edge cases; Redisson implements safer algorithms (incl. Redlock-style for multi-node) |
| **Alternatives** | ZooKeeper/etcd locks; DB advisory locks (Postgres — Phase 4) |
| ⚠️ Caveat | Distributed locks are **not perfectly safe** under clock skew/GC pauses (Phase 1.11) — prefer **idempotency** (§5) where possible instead of relying on a lock |
> ⭐ Common uses: ensure a **@Scheduled** job (Phase 5.8) runs on **one** instance only, prevent double-processing, leader election. ⚠️ The famous critique (Kleppmann): locks can't guarantee mutual exclusion under all failures — design idempotently so a rare double-execution is harmless (§5).

---

## 5. Idempotency (consolidated)

Because everything here is **at-least-once** (11.1) and may **retry** (12.3), the universal safety net is **idempotency** — doing an operation twice has the same effect as once. (Mechanics — **Phase 7.1 §1**.)
| Technique | Where |
|-----------|-------|
| **Idempotency keys** | Client-supplied unique id; dedup store (Redis — 4.8) (Phase 7.1) |
| **Inbox pattern** | Record processed message ids; skip duplicates (11.4 §6) |
| **Natural idempotency** | Design ops to be repeatable (`SET status=PAID`, upserts) |
| **Optimistic concurrency** | `@Version` / ETags (Phase 4.5 / 7.1 §3) |
> ⭐ **Idempotency is the thread connecting Saga, Outbox, retries, and locking** — it lets you safely use at-least-once delivery, retries, and even tolerate a distributed-lock failure. Make every consumer/step idempotent.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Cross-service ACID / 2PC | Saga + compensations (§1, 11.4) |
| Dual-write (DB then broker) divergence | Outbox + Debezium (§2, 11.4) |
| Query-time fan-out to many services | CQRS read model (§3, 12.2) |
| Event Sourcing everywhere | Use only where history/audit justifies it (§3) |
| In-process lock across instances | Distributed lock (Redisson) (§4) |
| Trusting a distributed lock for correctness | Add idempotency; locks aren't perfectly safe (§4/§5) |
| Lock with no TTL → deadlock on crash | Lease/TTL auto-expiry (§4) |
| Non-idempotent consumers/steps | Idempotency keys / Inbox (§5, 7.1/11.4) |
| Forgetting tracing across the saga | Propagate trace context (12.7) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- Consolidates **Phase 11.4** (Saga/Outbox/Inbox/CQRS/Event Sourcing) + **Phase 7.1** (idempotency) for microservices.
- **Outbox/CDC** via **Kafka/Debezium** (11.2); **read models** via Kafka Streams (11.2).
- **Distributed locking** uses **Redis/Redisson** (Phase 4.8) and guards **@Scheduled** jobs (Phase 5.8); contrasts JVM concurrency (Phase 1.10).
- **Transactions** (Phase 4.5/5.5) make Outbox/Inbox locally atomic.
- **Resilience** (12.3) retries → idempotency; **observability** (12.7) traces sagas.
- Several are **design patterns** (Phase 14.1) and **architecture patterns** (14.4).
- Core of **Project 7 (Microservices E-Commerce)** (place-order saga, outbox, CQRS, idempotency).

---

## 8. Quick Self-Check Questions

1. How does a Saga replace a cross-service transaction, and what role do compensations play?
2. What is the dual-write problem, and how do Outbox + Debezium solve it reliably?
3. How does a CQRS read model avoid query-time fan-out across services?
4. When is Event Sourcing worth its complexity?
5. Why don't in-process locks work across service instances?
6. How does a Redisson distributed lock work, and why is a TTL essential?
7. Why are distributed locks "not perfectly safe," and what's the recommended mitigation?
8. List four idempotency techniques and where each applies.
9. Why is idempotency the thread connecting all these patterns?
10. Which patterns here are detailed in Phase 11.4 vs added here?

---

## 9. Key Terms Glossary

- **Saga / compensation:** distributed transaction via local steps / semantic undo.
- **Orchestration vs choreography:** central coordinator vs event reactions.
- **Outbox pattern / Debezium / CDC:** atomic event write + log-based relay.
- **Inbox pattern:** consumer-side dedup for idempotency.
- **CQRS / read model / projection:** segregated reads / denormalized view / event-built view.
- **Event Sourcing:** events as source of truth; rebuild by replay.
- **Distributed lock / Redisson / RLock:** cross-instance mutual exclusion via Redis.
- **Lease / TTL:** auto-expiry to avoid deadlock on crash.
- **Idempotency / idempotency key / optimistic concurrency:** repeat-safe operations.
- **Effectively-once:** at-least-once + idempotency.

---

*This is the note for **Section 12.6 — Distributed Patterns**.*
*Previous section in roadmap: **12.5 Configuration Management**.*
*Next section in roadmap: **12.7 Observability in Microservices**.*
