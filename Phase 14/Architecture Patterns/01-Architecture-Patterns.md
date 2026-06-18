# Architecture Patterns

> **Phase 14 — Design Patterns & Architecture → 14.4 Architecture Patterns**
> Goal: Understand system-level structure — Layered architecture, Hexagonal (Ports & Adapters), Clean Architecture, DDD tactical design, Event-Driven, CQRS, and the Modular Monolith.

---

## 0. The Big Picture

Design patterns (14.1) organize classes; **architecture patterns** organize the **whole application** — its major boundaries, dependencies, and how business logic is isolated from infrastructure (DB, web, messaging). The recurring theme: **keep the domain (business rules) at the center, independent of frameworks and I/O**, so it stays testable and changeable.

```
   The big idea (Hexagonal/Clean):
        ┌──────────────────────────┐
        │   Frameworks / DB / Web   │  ← infrastructure (replaceable details)
        │   ┌──────────────────┐    │
        │   │   Application      │   │  ← use cases / orchestration
        │   │   ┌──────────┐     │   │
        │   │   │  Domain   │    │   │  ← business rules (no framework deps!)
        │   │   └──────────┘     │   │
        │   └──────────────────┘    │
        └──────────────────────────┘
   Dependencies point INWARD; the domain depends on NOTHING external (DIP — 14.2)
```

> Synthesizes DIP/SOLID (14.2), DDD (12.1), design patterns (14.1), and the distributed/event patterns (11.4/12.6). It governs how a Spring app (Phase 5) is structured and how microservices (Phase 12) are internally organized.

---

## 1. Layered (N-Tier) Architecture ⭐ (the classic default)

The traditional Spring structure — horizontal layers, each depending on the one below.
```
   ┌─────────────────┐
   │ Presentation     │  Controllers / REST (Phase 5.3/7)
   ├─────────────────┤
   │ Service / Business│ @Service, transactions (Phase 5.5)
   ├─────────────────┤
   │ Persistence/DAO  │  Repositories (Phase 5.4)
   ├─────────────────┤
   │ Database          │  (Phase 4)
   └─────────────────┘
```
| Pro | Con |
|-----|-----|
| Simple, familiar, fast to start | Domain depends on persistence (DB-centric) |
| Clear separation | Business logic can leak into services (anemic domain) |
| Great for CRUD apps | Harder to swap infrastructure; couples to framework |
> ⭐ Layered is **fine for most CRUD apps** and where to start. ⚠️ Its weakness: dependencies point **downward toward the database**, so the domain ends up depending on infrastructure — the inverse of what Hexagonal/Clean want. Tends toward **anemic domain models** (logic in services, dumb entities — contrast rich aggregates, 12.1).

---

## 2. Hexagonal Architecture (Ports & Adapters) ⭐

Alistair Cockburn's model: put the **domain/application at the center**; everything external (web, DB, messaging) connects through **ports** (interfaces) implemented by **adapters**. Dependencies point **inward** (DIP — 14.2).
```
        [REST adapter]   [CLI adapter]          (DRIVING / inbound adapters)
              │                │
              ▼                ▼
        ┌───────── Application core (use cases) ─────────┐
        │   Domain (entities, aggregates, rules — 12.1)   │
        └───────────────────────┬─────────────────────────┘
              ▲                  ▲
              │                  │
        [JPA adapter]      [Kafka adapter]      (DRIVEN / outbound adapters)
```
| Concept | Meaning |
|---------|---------|
| **Port** | An interface the core defines (e.g., `OrderRepository`, `PaymentGateway`) |
| **Driving (inbound) adapter** | Drives the app: REST controller, message listener, CLI |
| **Driven (outbound) adapter** | Driven by the app: JPA repo, Kafka publisher, HTTP client |
| **The core** | Domain + use cases — **no framework/DB imports** |
> ⭐ **The domain defines ports (interfaces); adapters implement them** (DIP). You can swap PostgreSQL for Mongo, REST for gRPC, by writing a new adapter — **the core never changes** and is **testable without any infrastructure** (mock the ports — Phase 6.3). This is DIP (14.2) applied at architecture scale. A DTO/mapper (7.3) at each adapter boundary is the ACL (12.1).

---

## 3. Clean Architecture

Robert C. Martin's concentric-circles refinement of Hexagonal/Onion — same core idea with named rings and **The Dependency Rule**.
```
   Entities (enterprise business rules)        ← innermost, most stable
   Use Cases (application business rules)
   Interface Adapters (controllers, presenters, gateways)
   Frameworks & Drivers (DB, web, Spring)      ← outermost, most volatile
```
> ⭐ **The Dependency Rule: source-code dependencies point only INWARD.** Inner circles know nothing about outer ones (the domain doesn't import Spring/JPA). Cross boundaries via interfaces (ports) + DTOs. Result: frameworks are **details** you can defer/replace; the business rules are the asset. ⚠️ It's more structure/ceremony — worth it for complex, long-lived domains; overkill for simple CRUD (KISS/YAGNI — 14.3).

| Hexagonal vs Clean vs Onion | Essentially the same principle |
|------------------------------|--------------------------------|
| All three: domain-centric, dependencies inward, infrastructure at the edge via interfaces |

---

## 4. DDD Tactical Design (architecture view)

(Recall tactical DDD — 12.1 §4.) At the architecture level, DDD shapes the **domain layer** of Hexagonal/Clean:
| Building block | Architectural role |
|----------------|--------------------|
| **Aggregate / Aggregate Root** | Consistency boundary = transaction boundary (one tx = one aggregate, 12.1) |
| **Entity / Value Object** | Rich domain model (logic in the domain, not anemic services) |
| **Domain Event** | Decouples within & across bounded contexts (→ Event-Driven §5) |
| **Repository** | A **port** (interface) for persistence; adapter implements it (Phase 5.4) |
| **Domain Service** | Domain logic spanning aggregates |
| **Bounded Context** | A module/service boundary (12.1) |
> ⭐ DDD gives you the **rich domain model** that Hexagonal/Clean protect at the center — avoiding the anemic domain that layered architecture tends to produce (§1). Repositories are ports; bounded contexts are module/service seams (12.1).

---

## 5. Event-Driven Architecture (recap) & CQRS

- **Event-Driven** (Phase 11.4 §1): components communicate via events — loose coupling, scalability, resilience. As an architecture style, the system's structure is organized around producing/consuming events (within a service via Spring events; across services via Kafka/RabbitMQ — Phase 11).
- **CQRS** (Phase 11.4 §2 / 12.6): separate read and write models as an architectural choice — often combined with event-driven and Hexagonal (commands/queries are use cases / ports).
> ⭐ These are detailed in Phase 11.4/12.6; here they're framed as **architecture styles** you can combine with Hexagonal/Clean (e.g., a hexagonal service that publishes domain events via an outbound adapter — Outbox, 11.4 §5).

---

## 6. Modular Monolith ⭐ (the pragmatic sweet spot)

(Recall 12.1 §1.) A **single deployable** with **strong internal module boundaries** (by bounded context) — modules communicate via well-defined interfaces/events, not by reaching into each other's internals or sharing tables.
```
   One deployable JAR:
   ┌── Orders module ──┐  ┌── Payments module ──┐  ┌── Inventory module ──┐
   │ own package/schema│  │ own package/schema   │  │ own package/schema   │
   │ public API only   │◄─┤ calls via interface  │  │ events between them   │
   └───────────────────┘  └──────────────────────┘  └──────────────────────┘
```
| Modular Monolith vs Microservices |
|-----------------------------------|
| ✅ Clean boundaries WITHOUT distributed-systems pain (no network, easy ACID, simple deploy/observability) |
| ✅ Can extract a module into a microservice later (Strangler Fig — 12.1 §6) if needed |
| ⚠️ Requires discipline to keep boundaries (tools like ArchUnit — Phase 6.7 — enforce them) |
> ⭐ **Start here, not with microservices** (12.1). A modular monolith gives you most of the architectural benefits (clear boundaries, team ownership) without the operational tax. Enforce module boundaries with **ArchUnit** (Phase 6.7). Extract to microservices only when scaling/org needs demand it.

---

## 7. Choosing an Architecture

| Situation | Architecture |
|-----------|--------------|
| Simple CRUD app | **Layered** (§1) — don't over-engineer (14.3) |
| Complex domain, long-lived | **Hexagonal/Clean + DDD** (§2/§3/§4) |
| Multiple cohesive domains, one team/deploy | **Modular Monolith** (§6) |
| Need independent scaling/teams (earned) | **Microservices** (Phase 12) — often each internally Hexagonal |
| High read/write divergence | **CQRS** (§5) |
| Loose coupling, async workflows | **Event-Driven** (§5) |
> ⭐ Architecture is about **trade-offs**, not fashion. Match it to domain complexity and organizational needs. Combine styles (a hexagonal modular monolith with event-driven modules is common and excellent). ⚠️ Resist cargo-culting microservices/Clean Architecture onto a simple app (KISS/YAGNI — 14.3).

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Domain depending on the database (layered trap) | Invert dependencies; ports/adapters (§2, DIP 14.2) |
| Anemic domain (logic in services) | Rich domain model / aggregates (§4, 12.1) |
| Clean Architecture on a simple CRUD app | Use Layered; KISS/YAGNI (§1/§7, 14.3) |
| Microservices-first | Modular monolith first; extract later (§6, 12.1) |
| Modules secretly sharing tables/internals | Enforce boundaries (ArchUnit — Phase 6.7) (§6) |
| Framework code leaking into domain | Keep core framework-free (§2/§3) |
| Treating architecture as one-size-fits-all | Choose per trade-offs (§7) |
| Skipping boundaries because "it's faster now" | Boundaries are cheap now, expensive to add later |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **DIP/SOLID** (14.2) is the engine of Hexagonal/Clean; **design patterns** (14.1) implement the pieces.
- **DDD** (12.1) defines the domain core & boundaries; **CQRS/Event-Driven/Outbox** (11.4/12.6) are combinable styles.
- Spring structures: controllers/services/repos (Phase 5.3/5.4/5.5) map to layers or adapters; **DTOs/mappers** (7.3) at boundaries.
- **ArchUnit** (Phase 6.7) enforces architecture rules in tests.
- **Modular monolith vs microservices** (12.1); **Strangler Fig** migration (12.1 §6).
- Underpins maintainability/testability (Phase 6) across all later work.
- Applied across **Projects 4–7** (each with an appropriate architecture).

---

## 10. Quick Self-Check Questions

1. What is the central idea shared by Hexagonal and Clean Architecture?
2. Describe Layered architecture and its main weakness.
3. What are ports and adapters (driving vs driven), and how does the core stay testable?
4. State Clean Architecture's Dependency Rule.
5. How are Hexagonal, Clean, and Onion related?
6. How does DDD tactical design shape the domain core, and what is the anemic-domain anti-pattern?
7. How do Event-Driven and CQRS combine with Hexagonal architecture?
8. What is a modular monolith, and why start there instead of microservices?
9. How do you enforce module/architecture boundaries?
10. How do you choose an architecture, and what's the risk of cargo-culting?

---

## 11. Key Terms Glossary

- **Architecture pattern:** system-level structure of boundaries & dependencies.
- **Layered (N-tier):** presentation/service/persistence/DB layers.
- **Hexagonal (Ports & Adapters):** domain core + ports (interfaces) + adapters.
- **Port / adapter (driving/driven):** interface / implementation (inbound/outbound).
- **Clean Architecture / Dependency Rule:** concentric rings; dependencies point inward.
- **Onion architecture:** equivalent domain-centric model.
- **DDD tactical (aggregate/entity/VO/domain event/repository):** rich domain modeling.
- **Anemic domain model:** logic in services, dumb data objects (anti-pattern).
- **Event-Driven / CQRS:** event-based / read-write-segregated styles.
- **Modular monolith:** one deployable with strong internal boundaries.
- **Strangler Fig:** incremental extraction to microservices.

---

*This is the note for **Section 14.4 — Architecture Patterns**.*
*Previous section in roadmap: **14.3 Clean Code**.*
*This completes **Phase 14 — Design Patterns & Architecture**.*
*Next: **Phase 15 — Security**.*
