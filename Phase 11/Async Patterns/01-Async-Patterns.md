# Async / Event-Driven Patterns

> **Phase 11 — Messaging → 11.4 Async Patterns**
> Goal: Master the architectural patterns built on messaging — event-driven architecture, CQRS, Event Sourcing, the Saga pattern (orchestration vs choreography), the Transactional Outbox & Inbox patterns, and eventual consistency.

---

## 0. The Big Picture

Messaging (11.1) + a broker (Kafka 11.2 / RabbitMQ 11.3) unlock **architectural patterns** for building decoupled, scalable, resilient distributed systems. These patterns solve the central hard problem of microservices: **keeping data consistent across services that each own their own database** — without distributed transactions.

```
   The core tension:
   Service A (DB A) ──must coordinate a change with──► Service B (DB B)
   ❌ No shared transaction across services/DBs (2PC is fragile/slow)
   ✅ Use EVENTS + these patterns → EVENTUAL consistency (reliably)
```

> These patterns build on messaging fundamentals (11.1 — at-least-once, idempotency, DLQ), Kafka/RabbitMQ (11.2/11.3), transactions (Phase 4.5), and idempotency (Phase 7.1). They are core to **microservices** (Phase 12 — esp. 12.6 distributed patterns) and **architecture** (Phase 14.4). Several appear in **Design Patterns** (Phase 14.1).

---

## 1. Event-Driven Architecture (EDA)

Services communicate by **producing and reacting to events** ("something happened") rather than calling each other directly. The producer doesn't know or care who consumes.

```
   Order Service ──emits──► [ OrderPlaced event ] ──► Inventory Service (reserve stock)
                                                  ├─► Email Service     (send confirmation)
                                                  └─► Analytics Service (record metric)
   Order Service doesn't call any of them — it just publishes the fact.
```
| Event style | Meaning |
|-------------|---------|
| **Event notification** | "X happened" (thin) — consumers fetch details if needed |
| **Event-carried state transfer** | Event carries the data → consumers don't call back (less coupling, more duplication) |
| **Command** | "Do X" (directed at a specific handler) — vs an event which is a broadcast fact |
> ⭐ EDA gives **loose coupling** (add a new consumer without touching the producer), **scalability**, and **resilience** (a down consumer doesn't break the producer — broker buffers). ⚠️ Costs: eventual consistency (§6), harder debugging (tracing — 9.2/12.7), and you must handle duplicates/ordering (11.1).

---

## 2. CQRS (Command Query Responsibility Segregation)

**Separate the write model from the read model.** Commands (writes) and queries (reads) use different models — often different stores — optimized independently.
```
   COMMANDS (writes) ──► Write model (normalized, validated) ──► DB / event log
                                          │ (events/sync)
                                          ▼
   QUERIES (reads)  ◄── Read model(s) (denormalized, fast, purpose-built — e.g., Elasticsearch 16.4)
```
| | Without CQRS | With CQRS |
|--|--------------|-----------|
| Model | One model for read+write | Separate write & read models |
| Reads | Constrained by the write schema | Denormalized, fast, tailored per query |
| Complexity | Lower | Higher (two models, sync between them) |
| Scaling | Together | Reads & writes scale independently (Phase 13.3) |
> ⭐ Use CQRS when read and write needs **diverge sharply** (complex reporting, high read load, many query shapes). The read model is updated **asynchronously** from write-side events (→ eventual consistency §6). ⚠️ Don't apply CQRS everywhere — it adds real complexity; many services are fine with one model. Often paired with Event Sourcing (§3) but **independent** of it.

---

## 3. Event Sourcing

Instead of storing **current state**, store the **full sequence of events** that led to it. Current state is **derived by replaying events**.
```
   Traditional:   UPDATE account SET balance = 90   (you lose the history; only "now")
   Event-sourced: append [Deposited 100][Withdrew 10]  → balance = replay = 90  (full history kept)
```
| Concept | Meaning |
|---------|---------|
| **Event store** | Append-only log of domain events (Kafka compacted topic §5 of 11.2, or a dedicated store) |
| **Replay** | Rebuild state by re-applying events from the start |
| **Snapshot** | Periodic saved state to avoid replaying from zero (perf) |
| **Projection** | Build a read model (CQRS §2) from the event stream |
| Benefits | Full audit trail, time-travel, rebuild any read model, temporal queries |
| ⚠️ Costs | Complexity, schema evolution of events, eventual consistency, no easy `UPDATE`/`DELETE` |
> ⭐ Event Sourcing gives a **perfect audit log** and the ability to **rebuild any projection** (great with CQRS §2). ⚠️ It's a **big commitment** — event versioning, replay performance (snapshots), and developer learning curve are real. Use it where history/auditability is a first-class requirement (finance, ledgers); don't default to it.

---

## 4. Saga Pattern ⭐ (distributed transactions without 2PC)

A business transaction spanning **multiple services** can't use one ACID transaction (each service owns its DB). A **Saga** is a sequence of **local transactions**, each publishing an event that triggers the next; if a step fails, **compensating transactions** undo the prior steps.
```
   Place Order Saga:
   1. Order: create order (PENDING)        ─ompensate► cancel order
   2. Payment: charge card                 ─compensate► refund
   3. Inventory: reserve stock             ─compensate► release stock
   4. Shipping: schedule delivery
   If step 3 fails → run compensations for 2 and 1 (refund + cancel) → consistent end state
```
> ⭐ Sagas trade atomicity for **eventual consistency via compensation** — there's no rollback across services, so you **semantically undo** completed steps. Every step (and compensation) must be **idempotent** (11.1/Phase 7.1) since events can be redelivered.

### 4.1 Orchestration vs Choreography
| | **Orchestration** | **Choreography** |
|--|-------------------|------------------|
| Control | A central **orchestrator** tells each service what to do next | No central brain — each service **reacts to events** and emits the next |
| Flow | Commands from coordinator | Event chain (each step listens & publishes) |
| Visibility | ⭐ Explicit, easy to follow/monitor | Implicit (must trace events — 9.2/12.7) |
| Coupling | Coupled to the orchestrator | Loosely coupled |
| Risk | Orchestrator is a complex hub | "Event spaghetti" — hard to see the whole flow |
| Tools | Camunda, Temporal, a state machine | Plain pub/sub on Kafka/RabbitMQ |
```
ORCHESTRATION:  [Orchestrator] → Order → Payment → Inventory  (central conductor)
CHOREOGRAPHY:   OrderPlaced → (Payment reacts) PaymentDone → (Inventory reacts) ...  (no conductor)
```
> ⭐ **Orchestration** for complex, evolving flows where you need visibility/control. **Choreography** for simple, loosely-coupled flows. Many systems mix them. (Saga is a key **distributed pattern** revisited in Phase 12.6 and Phase 14.1.)

---

## 5. Transactional Outbox Pattern ⭐ (the dual-write problem)

**The problem:** a service must update its **database** *and* **publish an event** atomically. If you do two separate writes (DB commit, then send to Kafka), a crash between them leaves them **inconsistent** (DB updated but event lost, or vice-versa). You can't put a DB and a broker in one transaction.

**The solution:** write the event to an **outbox table in the same DB transaction** as the business change. A separate process publishes outbox rows to the broker.
```
   ┌─ ONE local DB transaction (atomic — Phase 4.5) ─┐
   │  INSERT order ...                                │
   │  INSERT INTO outbox (event) VALUES (OrderPlaced) │
   └──────────────────────────────────────────────────┘
                          │  (committed together)
        Relay (poller OR Debezium CDC — 11.2 §8) reads outbox ──► publishes to Kafka ──► mark sent
```
| Relay mechanism | Note |
|-----------------|------|
| **Polling publisher** | A scheduled job reads unsent outbox rows and publishes them |
| **CDC (Debezium)** ⭐ | Tails the DB log; turns committed outbox inserts into events — no polling (11.2 §8) |
> ⭐ The Outbox pattern guarantees **the event is published iff the DB change committed** (no dual-write divergence). It produces **at-least-once** delivery → consumers must be **idempotent**. This is *the* standard way to reliably publish events from a service (revisited Phase 12.6).

---

## 6. Inbox Pattern & Idempotent Consumers

The mirror of Outbox on the **consuming** side. Since delivery is at-least-once (11.1 §4), a consumer may receive the **same message twice**. The **Inbox pattern**: record each processed message id in an **inbox table** (in the same transaction as the side effects); if the id is already there, **skip** (dedupe).
```
   ┌─ ONE transaction ─┐
   │ if messageId in inbox → SKIP (already processed)  │  ◄ dedupe
   │ else: apply side effects + INSERT messageId into inbox │
   └────────────────────┘
```
> ⭐ **Inbox = durable idempotency** for consumers (recall idempotency keys — Phase 7.1 §1; Redis dedup — Phase 4.8). Together, **Outbox (reliable publish) + Inbox (dedupe consume) = end-to-end "effectively-once"** on top of at-least-once infrastructure. This is the practical answer to "exactly-once" (11.1 §4).

---

## 7. Eventual Consistency

In a distributed, event-driven system you give up **immediate (strong) consistency** across services in exchange for availability & decoupling. Data becomes consistent **eventually** — after events propagate.
| | Strong consistency | Eventual consistency |
|--|--------------------|----------------------|
| When | Single DB / ACID transaction (Phase 4.5) | Across services via events |
| Read after write | Always sees latest | May briefly see stale data |
| Trade-off | Coupling, lower availability | Availability + decoupling, temporary staleness |
> ⭐ Eventual consistency is a **deliberate design choice** (CAP/PACELC — Phase 4/16.6). Design the **UX & API** for it: async responses (202 + polling — Phase 7.1 §6), "your order is being processed", optimistic UI, and idempotent retries. ⚠️ Don't pretend it's instantaneous — surface the in-progress state. Sagas (§4), CQRS (§2), and Outbox/Inbox (§5/§6) all live in this eventually-consistent world.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Dual-write (DB then broker) → divergence | **Outbox** pattern (§5) |
| Non-idempotent consumers | **Inbox** / dedupe (§6, 11.1, Phase 7.1) |
| Distributed 2PC across services | **Saga** + compensations (§4) |
| Forgetting compensating transactions | Define a compensation per saga step (§4) |
| Choreography "event spaghetti" | Orchestration for complex flows; trace events (§4.1, 9.2/12.7) |
| CQRS/Event Sourcing everywhere | Apply only where complexity pays off (§2/§3) |
| Expecting strong consistency cross-service | Embrace & design for eventual consistency (§7) |
| Event Sourcing without snapshots | Snapshot to bound replay cost (§3) |
| Breaking event schemas | Schema Registry / versioning (11.2 §9, Phase 7.1) |
| No DLQ for failed events | Add DLQ + retries (11.1 §6) |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- Built on messaging (11.1) + **Kafka** (11.2 — Streams for CQRS read models / Event Sourcing log; Debezium for Outbox) / **RabbitMQ** (11.3 — task/command routing for sagas).
- **Outbox/Inbox/Saga/CQRS/Event Sourcing** are the **distributed patterns** of **microservices** (Phase 12.6) and **architecture patterns** (Phase 14.4); also in **design patterns** (Phase 14.1).
- **Transactions** (Phase 4.5/5.5) make Outbox/Inbox atomic locally; **idempotency** (Phase 7.1), **Redis** dedup (Phase 4.8).
- **Distributed locking** (Redisson — Phase 12.6) sometimes guards saga steps.
- **Observability** (9.2/12.7) — trace event chains; **DLQ** for failures (11.1).
- **Eventual consistency** ↔ CAP/replication (Phase 16.6), async APIs (Phase 7.1 §6).
- Central to **Project 7 (Microservices E-Commerce)** (place-order saga, outbox, CQRS).

---

## 10. Quick Self-Check Questions

1. What core microservices problem do these patterns solve, and why not use a distributed transaction?
2. What is event-driven architecture, and what's the difference between an event and a command?
3. What is CQRS, when is it worth it, and how is the read model kept updated?
4. What is Event Sourcing, what are snapshots/projections for, and what are the costs?
5. What is a Saga, and how do compensating transactions replace rollback?
6. Orchestration vs choreography — trade-offs of each?
7. Explain the dual-write problem and how the Outbox pattern solves it. What relays the events?
8. What is the Inbox pattern, and how do Outbox + Inbox give "effectively-once"?
9. What is eventual consistency, and how do you design the API/UX for it?
10. Why must every step in these patterns be idempotent?

---

## 11. Key Terms Glossary

- **Event-driven architecture (EDA):** services react to events, not direct calls.
- **Event vs command:** "X happened" (broadcast) vs "do X" (directed).
- **Event-carried state transfer:** events include data so consumers don't call back.
- **CQRS:** separate write model and read model(s).
- **Event Sourcing / event store / replay / snapshot / projection:** store events as truth, derive state.
- **Saga:** sequence of local transactions with compensations.
- **Compensating transaction:** semantic undo of a completed step.
- **Orchestration vs choreography:** central coordinator vs event-reaction chain.
- **Outbox pattern:** write event to a DB table in the same tx; relay publishes it.
- **Dual-write problem:** inconsistency from writing to DB and broker separately.
- **Inbox pattern:** dedupe processed message ids for idempotent consumption.
- **Effectively-once:** at-least-once + idempotency (Outbox + Inbox).
- **Eventual consistency:** cross-service data converges over time.

---

*This is the note for **Section 11.4 — Async Patterns**.*
*Previous section in roadmap: **11.3 RabbitMQ**.*
*This completes **Phase 11 — Messaging**.*
*Next: **Phase 12 — Microservices**.*
