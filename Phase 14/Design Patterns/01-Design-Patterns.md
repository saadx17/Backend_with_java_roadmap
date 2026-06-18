# Design Patterns (GoF + Enterprise/Modern)

> **Phase 14 — Design Patterns & Architecture → 14.1 Design Patterns**
> Goal: Know the reusable design patterns — the GoF Creational, Structural, and Behavioral patterns — plus the enterprise/modern backend patterns (Repository, Unit of Work, Specification, Circuit Breaker, Saga, Outbox, CQRS, Event Sourcing) and where Spring already uses them.

---

## 0. The Big Picture

A **design pattern** is a named, reusable solution to a recurring design problem — a shared vocabulary among engineers. The classic catalog is the **Gang of Four (GoF)** 23 patterns (1994), grouped into Creational, Structural, Behavioral. On top of those, backend engineering has **enterprise patterns** (Repository, Unit of Work...) and **distributed patterns** (Circuit Breaker, Saga, Outbox, CQRS — already met in Phases 11.4/12).

```
   GoF:  Creational (how to create objects)
         Structural (how to compose objects)
         Behavioral (how objects interact)
   Enterprise/Modern: Repository, Unit of Work, Specification, Circuit Breaker, Saga, Outbox, CQRS, Event Sourcing
```

> ⭐ **You already use most of these via Spring** — recognizing them deepens understanding. Patterns connect to OOP (Phase 1.3), SOLID (14.2), Clean Code (14.3), architecture (14.4), and the distributed patterns of Phases 11.4/12.6. ⚠️ Patterns are tools, not goals — don't force them (over-engineering is a real smell, 14.3).

---

## 1. Creational Patterns (object creation)

| Pattern | Intent | Spring / Java example |
|---------|--------|------------------------|
| **Singleton** | One shared instance | ⭐ Spring beans are singletons by default (Phase 5.1) — but managed by the container, not the GoF static way |
| **Factory Method** | Subclass decides which object to create | `Calendar.getInstance()`, `BeanFactory` |
| **Abstract Factory** | Families of related objects | `DocumentBuilderFactory`, connection factories |
| **Builder** ⭐ | Step-by-step construction of complex objects | `StringBuilder`, Lombok `@Builder`, `UriComponentsBuilder`, records + builders (Phase 1.3) |
| **Prototype** | Clone an existing object | Spring `@Scope("prototype")` beans |
```java
Order order = Order.builder()        // Builder — readable, immutable construction (Phase 1.3)
    .customerId(42).addItem(item).total(Money.of("59.90")).build();
```
> ⭐ **Builder** is the most useful day-to-day (complex immutable objects, test data builders — Phase 6.5). **Singleton** is everywhere via Spring's container (better than the static-field GoF version — testable, managed).

---

## 2. Structural Patterns (object composition)

| Pattern | Intent | Example |
|---------|--------|---------|
| **Adapter** | Make incompatible interfaces work together | Wrapping a third-party API; an **ACL** (12.1); `Arrays.asList` |
| **Decorator** ⭐ | Add behavior by wrapping | `BufferedReader` wraps `Reader` (Phase 1.9); Resilience4j wraps calls (12.3) |
| **Proxy** ⭐ | A stand-in controlling access | ⭐ Spring **AOP**/`@Transactional`/`@Cacheable` use dynamic proxies (Phase 5.1/5.5/5.7) |
| **Facade** | Simplified interface to a subsystem | A service facade over complex logic; SLF4J/Micrometer facades (9.1/9.2) |
| **Composite** | Tree of objects treated uniformly | UI trees, file systems |
| **Bridge** | Decouple abstraction from implementation | SLF4J (interface) + Logback (impl) (9.1) |
| **Flyweight** | Share fine-grained objects | `Integer.valueOf` cache, String pool (Phase 1.4) |
> ⭐ **Proxy** is foundational in Spring — `@Transactional`, `@Cacheable`, `@Async`, security all work via proxies (which is why **self-invocation bypasses them** — recall Phase 5.5!). **Decorator** = the wrapping you do with I/O streams and resilience.

---

## 3. Behavioral Patterns (object interaction)

| Pattern | Intent | Example |
|---------|--------|---------|
| **Strategy** ⭐ | Interchangeable algorithms behind an interface | `Comparator` (Phase 1.6); pluggable validators; payment strategies |
| **Observer** ⭐ | Notify dependents of changes | Spring `ApplicationEvent`/listeners; domain events (12.1); reactive (16.1) |
| **Template Method** | Skeleton with overridable steps | `JdbcTemplate`/`RestTemplate` (Phase 4.7/5.9); `AbstractList` |
| **Command** | Encapsulate a request as an object | `Runnable`/`Callable` (Phase 1.10); messaging commands (11.4) |
| **Iterator** | Sequential access | `Iterator`/`Iterable` (Phase 1.6) |
| **State** | Behavior changes with internal state | Order status state machine; circuit breaker states (12.3) |
| **Chain of Responsibility** | Pass a request along handlers | Servlet **filters**/interceptors (Phase 5.3); gateway filters (12.4) |
| **Mediator** | Centralize complex interactions | Saga orchestrator (11.4/12.6) |
| **Visitor / Memento / Interpreter** | Operations on structures / snapshots / grammars | AST visitors, etc. |
> ⭐ **Strategy** and **Observer** are the workhorses. Strategy = "swap the algorithm" (the basis of `Comparator`, pluggable behavior, and DI of interfaces). Observer = events (Spring events, domain events 12.1, reactive streams 16.1).

---

## 4. Enterprise Patterns (data & service layer)

| Pattern | Intent | In Spring |
|---------|--------|-----------|
| **Repository** ⭐ | Abstraction over data access (collection-like) | Spring Data `JpaRepository` (Phase 5.4) |
| **Unit of Work** | Track changes, commit/rollback as one | JPA **persistence context** + `@Transactional` (Phase 5.4/5.5) |
| **Specification** ⭐ | Composable, reusable query criteria | Spring Data `Specification` / Criteria API (Phase 5.4); filtering (7.1) |
| **DTO / Data Mapper** | Separate transfer/persistence models | DTOs + MapStruct (Phase 7.3) |
| **Service Layer** | Application logic / transaction boundary | `@Service` (Phase 5.1/5.5) |
```java
// Specification — compose query criteria (Phase 5.4); powers dynamic filtering (Phase 7.1)
Specification<Order> spec = statusIs(PENDING).and(totalAbove(BigDecimal.TEN));
List<Order> results = orderRepo.findAll(spec);
```
> ⭐ **Repository** + **Unit of Work** are how Spring Data + JPA already work (you've been using them since Phase 5.4). **Specification** is the clean way to build dynamic/composable queries — the proper answer to filtering (7.1 §2) without query-string-to-SQL hacks.

---

## 5. Modern / Distributed Patterns (recap)

These were detailed earlier — listed here as part of the catalog:
| Pattern | Where detailed |
|---------|----------------|
| **Circuit Breaker** | Resilience (Phase 12.3) — fail fast on a broken dependency |
| **Saga** | Phase 11.4 §4 / 12.6 — distributed transaction via local steps + compensation |
| **Outbox** | Phase 11.4 §5 / 12.6 — reliable event publishing (dual-write fix) |
| **CQRS** | Phase 11.4 §2 / 12.6 — separate read/write models |
| **Event Sourcing** | Phase 11.4 §3 / 12.6 — events as source of truth |
| **Idempotent Receiver / Inbox** | Phase 7.1 / 11.4 §6 — safe duplicate handling |
> ⭐ These are the **distributed-systems patterns** — the GoF for microservices. They solve consistency/reliability/scale across services (Phase 12), not single-process object design.

---

## 6. Using Patterns Well

| Guideline | Why |
|-----------|-----|
| Learn to **recognize** patterns | Shared vocabulary; understand frameworks (Spring) |
| Apply to a **real problem**, not preemptively | Avoid over-engineering (14.3) |
| Prefer the **simplest** thing that works | YAGNI (14.3) |
| Many patterns = **DI of an interface** | Strategy/Observer/Repository all lean on interfaces (SOLID — 14.2) |
| Favor **composition over inheritance** | Decorator/Strategy over deep hierarchies (Phase 1.3) |
> ⚠️ **Pattern overuse is a smell.** A `AbstractSingletonProxyFactoryBean`-style cathedral for a simple need is worse than plain code. Patterns earn their keep when they reduce real complexity or communicate intent.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Forcing patterns where unneeded | Apply to real problems only (§6, 14.3 YAGNI) |
| GoF Singleton via static state | Use Spring-managed singleton beans (§1) |
| Self-invocation bypassing proxies | Understand Proxy pattern in Spring AOP (§2, Phase 5.5) |
| Deep inheritance instead of Strategy/Decorator | Composition over inheritance (§6, 1.3) |
| Reinventing Repository/Unit of Work | Use Spring Data + `@Transactional` (§4) |
| String-built dynamic queries | Specification pattern (§4, 7.1) |
| Treating distributed patterns as object patterns | They solve cross-service problems (§5, Phase 12) |
| Pattern cargo-culting (names without intent) | Know the *problem* each solves |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- Spring **is** a patterns showcase: Singleton (beans), Proxy (AOP/`@Transactional`/`@Cacheable`), Template Method (`*Template`), Factory (`BeanFactory`), Observer (events), Strategy (DI of interfaces), Repository/Unit of Work (Data/JPA).
- **SOLID** (14.2) underpins good pattern use; **Clean Code** (14.3) warns against overuse; **architecture** (14.4) composes them at the macro level.
- **Specification** ↔ filtering (7.1) & JPA (5.4); **DTO/Mapper** ↔ 7.3; **Circuit Breaker/Saga/Outbox/CQRS** ↔ Phases 12.3/11.4/12.6.
- Builds on **OOP** (Phase 1.3), **generics/functional** (1.7/1.8), **collections** (1.6).
- Applied across all **Projects** (clean, pattern-aware design).

---

## 9. Quick Self-Check Questions

1. What are the three GoF categories, and what does each address?
2. Where does Spring use Singleton and Proxy, and why does Proxy explain self-invocation issues?
3. When would you use Builder, and what real classes embody it?
4. Explain Strategy and Observer with Java/Spring examples.
5. What is the Repository pattern, and how does Spring Data implement it?
6. What is Unit of Work, and how does JPA + `@Transactional` realize it?
7. What is the Specification pattern, and what problem does it solve cleanly?
8. Which "modern" patterns are really distributed-systems patterns, and where are they detailed?
9. Why is composition often better than inheritance for these patterns?
10. When is using a pattern a mistake?

---

## 10. Key Terms Glossary

- **Design pattern / GoF:** reusable solution to a recurring problem / the classic 23.
- **Creational / Structural / Behavioral:** creation / composition / interaction patterns.
- **Singleton / Factory / Builder / Prototype:** creational patterns.
- **Adapter / Decorator / Proxy / Facade / Bridge / Composite / Flyweight:** structural.
- **Strategy / Observer / Template Method / Command / State / Chain of Responsibility / Mediator:** behavioral.
- **Repository / Unit of Work / Specification / Service Layer:** enterprise data patterns.
- **Circuit Breaker / Saga / Outbox / CQRS / Event Sourcing / Inbox:** distributed patterns.
- **Composition over inheritance / YAGNI:** design guidelines.

---

*This is the note for **Section 14.1 — Design Patterns**.*
*Previous section in roadmap: **13.3 Scalability**.*
*Next section in roadmap: **14.2 SOLID Principles**.*
