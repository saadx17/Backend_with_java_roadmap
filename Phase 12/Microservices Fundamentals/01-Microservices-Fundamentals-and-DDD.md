# Microservices Fundamentals & Domain-Driven Design

> **Phase 12 — Microservices → 12.1 Microservices Fundamentals**
> Goal: Understand monolith vs microservices vs modular monolith, the Domain-Driven Design (DDD) toolkit (bounded contexts, aggregates, entities vs value objects, domain events, domain services, anti-corruption layers, context mapping), service decomposition, the Strangler Fig migration, and the core microservices principles.

---

## 0. The Big Picture

A **microservices architecture** splits an application into **small, independently deployable services**, each owning a slice of the business and **its own database**, communicating over the network (REST — Phase 7, or messaging — Phase 11). The opposite is a **monolith** — one deployable. The hardest part isn't the tech; it's **drawing the boundaries** — which is exactly what **Domain-Driven Design (DDD)** helps you do.

```
   MONOLITH                          MICROSERVICES
   ┌──────────────────┐             ┌────────┐ ┌────────┐ ┌────────┐
   │ Orders           │             │ Orders │ │Payment │ │Inventory│
   │ Payments         │             │  +DB   │ │  +DB   │ │  +DB    │
   │ Inventory        │  ──split──► └────────┘ └────────┘ └────────┘
   │ (one DB, one jar)│             each: own deploy, own DB, own team, talks via API/events
   └──────────────────┘
```

> This is the architectural capstone — it ties together REST (Phase 7), messaging & async patterns (Phase 11), Docker/K8s (Phase 10), observability (Phase 9), and feeds the rest of Phase 12 (communication, resilience, gateway, config, distributed patterns, observability) and architecture patterns (Phase 14.4).

---

## 1. Monolith vs Microservices vs Modular Monolith

| | **Monolith** | **Modular Monolith** ⭐ | **Microservices** |
|--|--------------|-------------------------|-------------------|
| Deploy unit | One | One | Many (independent) |
| Modules | Often tangled | **Strong internal boundaries** (modules/packages) | Separate services |
| Database | One shared | One shared (per-module schemas) | **One per service** |
| Scaling | Whole app | Whole app | Per service |
| Team autonomy | Low | Medium | High |
| Operational complexity | **Low** | Low | **High** (network, deploy, observability) |
| Consistency | Easy (ACID — Phase 4.5) | Easy (ACID) | Hard (eventual — Phase 11.4) |
| Best for | Small apps, startups | **Most apps** — modular without distributed pain | Large orgs, independent scaling/teams |

> ⭐ **Start with a (modular) monolith.** Microservices add huge operational cost — network failures, distributed transactions (Phase 11.4 Saga), observability (12.7), deployment complexity (Phase 10). A **modular monolith** gives you clean boundaries *without* the distributed-systems tax, and you can extract services later (Strangler Fig §6) once boundaries are proven. ⚠️ "Microservices first" is a common, expensive mistake (the **distributed monolith** anti-pattern — §7).

---

## 2. Domain-Driven Design (DDD) — Why It Matters Here

DDD is a way to model complex software around the **business domain**, using a **Ubiquitous Language** (shared vocabulary between devs and domain experts). Its **strategic** patterns (bounded contexts, context mapping) tell you **where to draw service boundaries**; its **tactical** patterns (aggregates, entities, value objects, domain events) tell you how to model **inside** a service.

| DDD layer | Patterns | Answers |
|-----------|----------|---------|
| **Strategic** | Bounded Context, Ubiquitous Language, Context Mapping, ACL | *Where are the boundaries between services?* |
| **Tactical** | Entity, Value Object, Aggregate, Domain Event, Domain Service, Repository | *How do I model a service's domain?* |

> ⭐ **DDD bounded contexts are the #1 tool for finding service boundaries.** A microservice should usually map to one bounded context. (Tactical patterns also appear in Phase 14.4 architecture.)

---

## 3. Bounded Contexts & Context Mapping

A **Bounded Context** is an explicit boundary within which a model and its language are consistent. The *same word means different things* in different contexts → each gets its own model.
```
   "Customer" in SALES context   = lead, pipeline, discounts
   "Customer" in SUPPORT context = tickets, SLAs, history
   "Customer" in BILLING context = invoices, payment methods
   → 3 different models → 3 candidate services (not one shared "Customer" class!)
```
**Context Mapping** describes relationships between contexts:
| Relationship | Meaning |
|--------------|---------|
| **Partnership** | Two contexts succeed/fail together; coordinate |
| **Customer–Supplier** | Upstream provides; downstream depends |
| **Conformist** | Downstream just accepts the upstream model |
| **Anti-Corruption Layer (ACL)** ⭐ | Downstream **translates** the upstream model into its own (protects its domain) |
| **Shared Kernel** | A small shared model (use sparingly — creates coupling) |
| **Open Host Service / Published Language** | A well-defined public API/contract (REST/events — Phase 7) |
> ⭐ The **Anti-Corruption Layer (ACL)** is critical when integrating with legacy systems or external services: a translation layer so their model doesn't "leak" and corrupt yours (recall DTOs/mappers — Phase 7.3, which are a tactical form of this at the API edge).

---

## 4. Tactical DDD: Entities, Value Objects, Aggregates

| Building block | Definition | Example |
|----------------|------------|---------|
| **Entity** | Has a distinct **identity** that persists over time; mutable | `Order` (id=42), `Customer` |
| **Value Object (VO)** | Defined by its **attributes**, no identity; **immutable**; equality by value | `Money(59.90, USD)`, `Address`, `Email` |
| **Aggregate** ⭐ | A cluster of entities/VOs treated as **one consistency unit**, with one **Aggregate Root** as the entry point | `Order` (root) + `OrderLine`s (inside) |
| **Aggregate Root** | The only object outside code references; enforces invariants for the whole aggregate | `Order` |
| **Domain Event** | Something meaningful that happened in the domain | `OrderPlaced`, `PaymentReceived` |
| **Domain Service** | Domain logic that doesn't belong to one entity | `PricingService`, `TransferService` |
| **Repository** | Persists/retrieves aggregates (Phase 5.4 / 14.1) | `OrderRepository` |

```
   Aggregate "Order":
   ┌─ Order (ROOT, Entity) ──────────────────┐
   │   total: Money (VO)                       │   ← outside code touches ONLY the root
   │   shippingAddress: Address (VO)           │
   │   lines: List<OrderLine> (entities inside)│   ← modified only via the root's methods
   └───────────────────────────────────────────┘
   Invariant enforced by root: total == sum(lines)  (always valid)
```
> ⭐ **Aggregate rules:** (1) outside code references **only the root**; (2) the root enforces **invariants** for the whole aggregate; (3) **one transaction = one aggregate** (cross-aggregate changes go through events/eventual consistency — Phase 11.4); (4) reference other aggregates **by id**, not by object. **Entities vs VOs:** ⭐ prefer Value Objects (immutable, side-effect-free — recall records, Phase 1.3) wherever identity doesn't matter — `Money`/`Email`/`Address` as VOs is far safer than primitive `BigDecimal`/`String`.

### 4.1 Domain events
A domain event (`OrderPlaced`) records a fact. Within a service, Spring's `ApplicationEventPublisher` can dispatch them; across services they become **integration events** on Kafka/RabbitMQ (Phase 11) — the basis of event-driven architecture (11.4).

---

## 5. Service Decomposition Strategies

How to split a system into services:
| Strategy | Decompose by... |
|----------|-----------------|
| **By bounded context / subdomain** ⭐ | Business capability (DDD §3) — the primary, recommended approach |
| **By business capability** | What the business does (Ordering, Shipping, Billing) |
| **By use case / verb** | Sometimes (e.g., a Search service) |
| ❌ By technical layer | Don't split into "UI service, logic service, DB service" — that's a distributed monolith |
> ⭐ Each service should be **loosely coupled, highly cohesive**, own its data, and ideally be ownable by **one small team** ("two-pizza team", Conway's Law). ⚠️ Wrong boundaries (too fine-grained, or chatty/coupled) are the worst outcome — hence "boundaries first, via DDD."

---

## 6. The Strangler Fig Pattern (Monolith → Microservices)

You rarely build microservices from scratch — you **migrate** a monolith incrementally. The **Strangler Fig** pattern (Martin Fowler): gradually route functionality to new services while the monolith still runs, until the monolith is "strangled" away.
```
   [ Proxy / API Gateway (12.4) ]
        ├─ /orders   ──► NEW Order Service      ◄ extracted first
        └─ /*        ──► OLD Monolith           ◄ everything else, still running
   Over time: extract more slices → route them away → monolith shrinks → eventually gone
```
| Step | Action |
|------|--------|
| 1 | Put a proxy/gateway in front (12.4) |
| 2 | Pick a bounded context (§3); build it as a new service |
| 3 | Route its traffic to the new service; keep data in sync (events/CDC — Phase 11.2/11.4) |
| 4 | Repeat; decommission monolith parts as they're replaced |
> ⭐ **Incremental, low-risk** migration — no risky "big bang rewrite." Each step is reversible. Data migration uses events/Outbox/CDC (Phase 11.4). This is the standard real-world path.

---

## 7. Microservices Principles & Trade-offs

**Core principles:** single responsibility (one bounded context), **own your data** (no shared DB!), independently deployable, **decentralized** (own tech/data choices), **design for failure** (Phase 12.3), automate everything (CI/CD — Phase 10.3), observability built-in (12.7).

| Benefit | Cost / Risk |
|---------|-------------|
| Independent deploy & scale (Phase 13.3) | Network latency & failures (Phase 12.2/12.3) |
| Team autonomy | Distributed transactions → Saga (Phase 11.4) |
| Tech diversity | Operational complexity (12.7, Phase 10) |
| Fault isolation | Eventual consistency (Phase 11.4) |
| Targeted scaling | Testing/debugging harder; data duplication |

> ⚠️ **The Distributed Monolith** (worst of both worlds): services that are deployed separately but **tightly coupled** (synchronous chains, shared DB, must-deploy-together). Causes: wrong boundaries, shared database, chatty sync calls. **Avoid via:** correct DDD boundaries (§3), each service owning its data, async/events where possible (Phase 11), and resilience (12.3). ⭐ **Microservices are an organizational/scaling solution, not a default** — most teams should earn them by outgrowing a modular monolith.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| "Microservices first" by default | Start with a modular monolith; extract later (§1/§6) |
| Shared database across services | Each service owns its data (§7) |
| Splitting by technical layer | Split by bounded context/capability (§3/§5) |
| Distributed monolith (tight coupling) | Correct boundaries + async + resilience (§7) |
| Too fine-grained ("nanoservices") | Right-size to teams/cohesion (§5) |
| Cross-service ACID transactions | Saga + eventual consistency (Phase 11.4) |
| One transaction spanning aggregates | One tx = one aggregate; events for the rest (§4) |
| Anemic domain (logic in services, data in dumb entities) | Rich aggregates enforce invariants (§4) |
| Leaking external models into your domain | Anti-Corruption Layer (§3) |
| Big-bang rewrite | Strangler Fig (§6) |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Bounded contexts** → service boundaries; **aggregates/entities/VOs** modeled with JPA (Phase 5.4) & records (Phase 1.3); **repositories** (Phase 5.4/14.1).
- **Domain events** → Spring events internally, Kafka/RabbitMQ integration events across services (Phase 11).
- **Each service** = a Spring Boot app (Phase 5.2) in a container (Phase 10.1) on K8s (10.2).
- Feeds the rest of Phase 12: **communication** (12.2), **resilience** (12.3), **gateway** (12.4), **config** (12.5), **distributed patterns/Saga/Outbox** (12.6, Phase 11.4), **observability** (12.7).
- **Architecture patterns** (Phase 14.4 — hexagonal, DDD tactical, modular monolith) build on this; **SOLID** (14.2) at class level.
- **Strangler Fig** uses the **gateway** (12.4) + CDC/events (Phase 11).
- Capstone for **Project 7 (Microservices E-Commerce)**.

---

## 10. Quick Self-Check Questions

1. Compare monolith, modular monolith, and microservices — and why start with the monolith?
2. What is a bounded context, and why is it the key tool for service boundaries?
3. What does context mapping describe, and what is an Anti-Corruption Layer?
4. Entity vs Value Object — definition and an example of each. Why prefer VOs?
5. What is an aggregate and aggregate root, and what are the four aggregate rules?
6. Why is "one transaction = one aggregate," and how do cross-aggregate changes happen?
7. What are good vs bad service-decomposition strategies?
8. Explain the Strangler Fig pattern and why it beats a big-bang rewrite.
9. List core microservices principles and their main costs.
10. What is a distributed monolith, what causes it, and how do you avoid it?

---

## 11. Key Terms Glossary

- **Monolith / modular monolith / microservices:** one deploy / one deploy with strong boundaries / many independent services.
- **Domain-Driven Design (DDD):** modeling software around the business domain.
- **Ubiquitous Language:** shared dev/expert vocabulary.
- **Bounded Context:** boundary of a consistent model/language.
- **Context Mapping / Anti-Corruption Layer (ACL):** inter-context relationships / translation boundary.
- **Entity / Value Object:** identity-based mutable / attribute-based immutable.
- **Aggregate / Aggregate Root:** consistency cluster / its single entry point.
- **Domain Event / Domain Service / Repository:** fact / domain logic w/o a home entity / persistence abstraction.
- **Service decomposition:** splitting by bounded context/capability.
- **Strangler Fig:** incremental monolith→microservices migration.
- **Distributed monolith:** separately deployed but tightly coupled services (anti-pattern).
- **Conway's Law / two-pizza team:** architecture mirrors org / small autonomous team.

---

*This is the note for **Section 12.1 — Microservices Fundamentals**.*
*Previous section in roadmap: **11.4 Async Patterns**.*
*Next section in roadmap: **12.2 Inter-Service Communication**.*
