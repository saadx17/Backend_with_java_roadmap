# SOLID Principles

> **Phase 14 — Design Patterns & Architecture → 14.2 SOLID Principles**
> Goal: Master the five SOLID principles of object-oriented design — Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion — with Java/Spring examples and the problems each prevents.

---

## 0. The Big Picture

**SOLID** is five principles (Robert C. Martin) for writing OO code that is **maintainable, flexible, and testable** — code that absorbs change without breaking. They're the *why* behind many design patterns (14.1) and the foundation of clean architecture (14.4). Spring's whole design (DI, interfaces) is SOLID in action.

```
   S - Single Responsibility   : one reason to change
   O - Open/Closed             : open to extension, closed to modification
   L - Liskov Substitution     : subtypes must be usable as their base type
   I - Interface Segregation   : many small interfaces > one fat one
   D - Dependency Inversion     : depend on abstractions, not concretions
```

> SOLID underpins design patterns (14.1), Clean Code (14.3), and architecture (14.4 — hexagonal/clean depend on DIP). It's enabled by OOP (Phase 1.3), interfaces/generics (1.7), and realized by Spring DI (Phase 5.1). Good SOLID code is also easy to **test** (Phase 6 — mockable seams).

---

## 1. S — Single Responsibility Principle (SRP)

> **A class should have one, and only one, reason to change** — i.e., one responsibility / one actor it answers to.

```java
// ❌ Violates SRP — this class does persistence, email, AND PDF generation (3 reasons to change)
class OrderService {
    void placeOrder(Order o) { saveToDb(o); sendEmail(o); generateInvoicePdf(o); }
}

// ✅ Each responsibility in its own class
class OrderService { /* orchestrates */ private final OrderRepository repo;
    private final EmailService email; private final InvoiceService invoice; }
class OrderRepository { /* persistence (Phase 5.4) */ }
class EmailService { /* notifications */ }
class InvoiceService { /* PDF */ }
```
> ⭐ SRP keeps classes small, focused, and **testable** (mock one collaborator at a time — Phase 6.3). ⚠️ A "god class" / "manager that does everything" is the classic SRP violation (a Clean Code smell — 14.3). The "reason to change" lens: would the email team and the DB team both need to edit this class? Then split it.

---

## 2. O — Open/Closed Principle (OCP)

> **Software entities should be open for extension, but closed for modification** — add new behavior without editing existing, tested code.

```java
// ❌ Adding a payment type means EDITING this method (and re-testing it) every time
double fee(String type) {
    if (type.equals("CARD")) return ...; else if (type.equals("PAYPAL")) return ...;  // grows forever
}

// ✅ Open/Closed via Strategy (14.1) — add a new type by adding a class, not editing existing code
interface PaymentMethod { double fee(); }
class CardPayment implements PaymentMethod { public double fee() { ... } }
class PayPalPayment implements PaymentMethod { public double fee() { ... } }
// new method → new class implementing PaymentMethod; nothing existing changes
```
> ⭐ OCP is achieved through **abstraction + polymorphism** (Strategy/Template Method — 14.1). Spring makes it natural: inject a `List<PaymentMethod>` and the container supplies all implementations. ⚠️ Don't over-apply — add extension points where change is *likely* (YAGNI — 14.3), not everywhere.

---

## 3. L — Liskov Substitution Principle (LSP)

> **Subtypes must be substitutable for their base types** without breaking correctness — a subclass shouldn't violate the contract of its parent.

```java
// ❌ Classic violation: Square extends Rectangle but breaks setWidth/setHeight expectations
class Rectangle { void setWidth(int w); void setHeight(int h); }
class Square extends Rectangle { /* setting width also changes height → breaks callers' assumptions */ }

// ❌ Throwing on an inherited method that the base contract says works
class ReadOnlyList<T> extends ArrayList<T> { public boolean add(T t){ throw new UnsupportedOperationException(); } }
```
| LSP rule | Meaning |
|----------|---------|
| Don't strengthen preconditions | Subclass can't demand *more* than the base |
| Don't weaken postconditions | Subclass must deliver *at least* what the base promises |
| Don't throw new unexpected exceptions | Honor the base contract |
| Preserve invariants | Subclass keeps the base's guarantees |
> ⭐ LSP is about **behavioral contracts**, not just compiling. ⚠️ If a subclass needs to throw `UnsupportedOperationException` or check `instanceof` to special-case it, you're likely violating LSP — **prefer composition over inheritance** (Phase 1.3) or rethink the hierarchy. The Square/Rectangle problem is the canonical example.

---

## 4. I — Interface Segregation Principle (ISP)

> **Clients should not be forced to depend on methods they do not use** — prefer many small, focused interfaces over one fat one.

```java
// ❌ Fat interface forces implementers to stub methods they don't need
interface Worker { void work(); void eat(); void sleep(); }
class RobotWorker implements Worker { /* must implement eat()/sleep() pointlessly */ }

// ✅ Segregated, role-based interfaces
interface Workable { void work(); }
interface Eatable { void eat(); }
class RobotWorker implements Workable { public void work(){...} }   // only what it needs
```
> ⭐ ISP yields **role interfaces** — small, cohesive, easy to implement and mock (Phase 6.3). It complements SRP at the interface level. Java's `Runnable`, `Comparator`, `AutoCloseable` are tiny by design. ⚠️ A "fat" interface with 20 methods couples every client to all of them.

---

## 5. D — Dependency Inversion Principle (DIP) ⭐

> **High-level modules should not depend on low-level modules; both should depend on abstractions. Abstractions should not depend on details; details depend on abstractions.**

```java
// ❌ High-level service depends directly on a concrete low-level class (tight coupling)
class OrderService { private final PostgresOrderDao dao = new PostgresOrderDao(); }  // hard to test/swap

// ✅ Depend on an abstraction (interface); the concrete impl is injected (Spring DI — Phase 5.1)
class OrderService {
    private final OrderRepository repo;                 // abstraction (interface)
    OrderService(OrderRepository repo) { this.repo = repo; }  // injected — swap impls, mock in tests
}
interface OrderRepository { Order save(Order o); }
class JpaOrderRepository implements OrderRepository { ... }   // detail depends on the abstraction
```
> ⭐ **DIP is the soul of Spring** — the IoC container injects abstractions (Phase 5.1). It enables: swapping implementations, **testing with mocks** (Phase 6.3), and **hexagonal/clean architecture** (14.4 — the domain depends on *ports*, adapters implement them). The dependency arrow points **toward** abstractions, decoupling policy from detail. ⚠️ `new`-ing dependencies inside a class violates DIP (and makes testing hard).

---

## 6. SOLID Together & With Patterns

```
   DIP → inject abstractions (Spring DI)
   OCP → extend via new implementations of those abstractions (Strategy)
   ISP → keep the abstractions small/focused
   SRP → one responsibility per class behind each abstraction
   LSP → implementations honor the abstraction's contract
   = code that's testable (Phase 6), flexible, and architecture-ready (14.4)
```
| Principle | Enables pattern (14.1) |
|-----------|------------------------|
| OCP | Strategy, Template Method, Decorator |
| DIP | Dependency Injection, Repository, Ports & Adapters (14.4) |
| ISP | Role interfaces (Observer, Command) |
> ⭐ SOLID and patterns reinforce each other; patterns are often *how* you achieve SOLID. The payoff is **testability** and **changeability**. ⚠️ Like patterns, SOLID can be **over-applied** — endless tiny interfaces and indirection for a simple CRUD app is its own smell (balance with KISS/YAGNI — 14.3).

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| God class doing everything | SRP — split by responsibility (§1) |
| `if/switch` chains growing per new type | OCP via Strategy/polymorphism (§2) |
| Subclass throwing/`instanceof` special-casing | LSP — fix hierarchy or use composition (§3) |
| Fat interfaces forcing empty implementations | ISP — small role interfaces (§4) |
| `new`-ing concrete deps inside a class | DIP — inject abstractions (§5, Spring DI) |
| Over-abstracting a simple CRUD app | Balance SOLID with KISS/YAGNI (§6, 14.3) |
| Interfaces with a single impl "just in case" | Add abstraction when change is real (§2/§5) |
| Treating SOLID as rules, not guidelines | Apply judgment; they serve testability/change |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **DIP** is realized by **Spring DI/IoC** (Phase 5.1) — the central mechanism of the whole framework.
- SOLID makes code **testable** (Phase 6 — mockable seams via interfaces).
- Underpins **design patterns** (14.1) and **architecture** (14.4 — hexagonal/clean rely on DIP/ports).
- **Clean Code** (14.3) is the line-level complement; **SonarQube** (10.3) flags some violations.
- Built on **OOP/interfaces/abstraction** (Phase 1.3), **generics** (1.7).
- Applied across all **Projects** for maintainable design.

---

## 9. Quick Self-Check Questions

1. State each SOLID principle in one sentence.
2. What does "one reason to change" mean, and what's the god-class smell?
3. How do you make code open/closed, and which patterns help?
4. What is the Square/Rectangle problem, and what does LSP really govern?
5. What's wrong with fat interfaces, and what does ISP recommend?
6. Why is DIP "the soul of Spring," and how does DI realize it?
7. Why does `new`-ing dependencies inside a class violate DIP and hurt testing?
8. How do SOLID principles reinforce each other and enable patterns?
9. How does SOLID improve testability (Phase 6)?
10. How can SOLID be over-applied, and what balances it?

---

## 10. Key Terms Glossary

- **SOLID:** five OO design principles for maintainable code.
- **Single Responsibility (SRP):** one reason to change per class.
- **Open/Closed (OCP):** extend without modifying existing code.
- **Liskov Substitution (LSP):** subtypes substitutable for base types (honor contracts).
- **Interface Segregation (ISP):** small, focused (role) interfaces.
- **Dependency Inversion (DIP):** depend on abstractions; inject details.
- **Role interface:** a small interface for one capability.
- **Ports & adapters:** DIP applied architecturally (14.4).
- **KISS / YAGNI:** balancing principles against over-engineering (14.3).

---

*This is the note for **Section 14.2 — SOLID Principles**.*
*Previous section in roadmap: **14.1 Design Patterns**.*
*Next section in roadmap: **14.3 Clean Code**.*
