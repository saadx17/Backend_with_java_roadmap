# Sealed Classes and Interfaces (Java 17)

> **Phase 1 — Java Language Mastery → 1.3 Object-Oriented Programming**
> Goal: Master sealed types — the `sealed`, `permits`, `non-sealed`, and `final` keywords; their use cases and design benefits; and why they pair perfectly with records, pattern matching, and exhaustive `switch`.

---

## 0. The Big Picture

A **sealed class or interface** restricts **which other classes/interfaces are allowed to extend or implement it**. Instead of a type being either fully open (anyone can subclass) or fully closed (`final`, nobody can), sealing gives you a **controlled middle ground**: *"these specific types — and only these — may extend me."*

```java
public sealed interface Shape permits Circle, Square, Triangle { }
```

This declares a **closed, known set** of subtypes. The author controls the hierarchy completely.

> **Why it matters:** Before sealed types, you couldn't have an inheritance hierarchy that was *both* extensible *and* closed. `final` allowed no subtypes; a normal class allowed unlimited, unknown subtypes. Sealing lets you say "exactly these," which unlocks **exhaustive `switch`**, safer domain modeling, and better API control.

Sealed types were finalized in **Java 17** (previewed in 15/16).

---

## 1. The Three-Way Choice for Types

Before sealed types, a type was one of two extremes. Now there are three:

| Modifier | Who can extend/implement it? |
|----------|------------------------------|
| *(normal class/interface)* | **Anyone** — unrestricted (open) |
| **`sealed`** | **Only the listed `permits` types** (controlled) |
| **`final`** | **No one** (closed) |

Sealing is the controlled middle ground between fully open and fully closed.

---

## 2. The `sealed` and `permits` Keywords

### 2.1 Basic syntax
```java
public sealed class Shape permits Circle, Square, Triangle {
    // common members
}
```
- `sealed` marks the class/interface as restricted.
- `permits` explicitly lists the **only** types allowed to directly extend/implement it.

### 2.2 The `permits` clause can be omitted in one case
If all permitted subtypes are declared in the **same source file**, you can drop `permits` — the compiler infers it:
```java
// All in Shape.java — permits is inferred:
public sealed interface Shape { }
final class Circle   implements Shape { }
final class Square   implements Shape { }
final class Triangle implements Shape { }
```

### 2.3 Sealed interfaces
Identical concept for interfaces (you saw this in the Interfaces note):
```java
public sealed interface Payment permits CreditCard, PayPal, BankTransfer { }
```

---

## 3. Rules for Permitted Subtypes (Critical)

Every type listed in `permits` **must itself declare one** of the following modifiers — this is mandatory and the compiler enforces it:

| Subtype modifier | Meaning |
|------------------|---------|
| **`final`** | The subtype is closed — no further subclassing. The hierarchy ends here. |
| **`sealed`** | The subtype continues the controlled hierarchy with its *own* `permits` list. |
| **`non-sealed`** | The subtype **reopens** the hierarchy — anyone can now extend *it* freely. |

```java
public sealed interface Shape permits Circle, Square, Polygon { }

public final class Circle implements Shape { }            // closed

public sealed interface Polygon extends Shape             // continues sealing
        permits Triangle, Rectangle { }

public final class Triangle  implements Polygon { }
public non-sealed class Rectangle implements Polygon { }  // reopened: anyone can extend Rectangle
```

> ⚠️ Forgetting to mark a permitted subtype with one of `final` / `sealed` / `non-sealed` is a **compile error**. This guarantees the "sealed" property can't be silently broken further down the chain.

### 3.1 Additional constraints
- Each permitted subtype must be a **direct** subtype and must **actually extend/implement** the sealed type.
- All permitted subtypes must be **accessible** to the sealed type at compile time, and (since Java 17) must be in the **same module** (or same package if in the unnamed module). You can't seal across arbitrary module boundaries.

---

## 4. The `non-sealed` Modifier

`non-sealed` is the escape hatch: a permitted subtype that wants to be **freely extensible again** uses `non-sealed`, breaking the seal *from that point downward*.

```java
public sealed class Animal permits Dog, Cat { }

public non-sealed class Dog extends Animal { }   // anyone can now subclass Dog
public final class Cat extends Animal { }        // Cat is closed

class Puppy extends Dog { }   // OK — Dog is non-sealed, so this is allowed
// class Kitten extends Cat {} // ERROR — Cat is final
```

Use `non-sealed` when part of your hierarchy needs to remain open for extension (e.g., a public extension point) while the rest stays controlled.

---

## 5. Use Cases & Design Benefits

### 5.1 Exhaustive `switch` (the headline benefit)
Because the compiler knows the **complete** set of subtypes, a `switch` covering all of them is provably **exhaustive** — **no `default` branch required**. If you later add a new permitted subtype, the compiler **flags every `switch` that no longer covers all cases** → you can't forget one.

```java
sealed interface Shape permits Circle, Square, Triangle {}
record Circle(double r)          implements Shape {}
record Square(double side)       implements Shape {}
record Triangle(double b, double h) implements Shape {}

double area(Shape s) {
    return switch (s) {                       // exhaustive — compiler verified
        case Circle c   -> Math.PI * c.r() * c.r();
        case Square sq  -> sq.side() * sq.side();
        case Triangle t -> 0.5 * t.b() * t.h();
        // no 'default' needed: all subtypes are covered
    };
}
```
> If you add `record Pentagon(...) implements Shape {}` to the `permits` list, this `switch` **won't compile** until you handle `Pentagon` — a powerful safety net. Without sealing, you'd need a `default` that silently swallows new cases.

### 5.2 Algebraic Data Types (closed domains)
Sealed types + records model **sum types** ("a value is exactly one of N known shapes") — a concept from functional programming. Perfect for closed domains:
```java
sealed interface Result<T> permits Success, Failure {}
record Success<T>(T value)        implements Result<T> {}
record Failure<T>(String error)   implements Result<T> {}
```
A `Result` is *either* a `Success` *or* a `Failure` — nothing else. Callers handle both via exhaustive `switch`.

### 5.3 Better domain modeling & documentation
- The hierarchy is **self-documenting**: the `permits` clause states the complete set of possibilities.
- Prevents **unexpected/unauthorized subclasses** that could violate invariants (a controlled form of the fragile-base-class defense from the Inheritance note).

### 5.4 API/library control
- Library authors can expose a type for **pattern matching/consumption** without allowing arbitrary external implementations — keeping the contract closed and evolvable.

### Benefits summary
| Benefit | Explanation |
|---------|-------------|
| **Exhaustiveness checking** | Compiler verifies `switch` covers all cases; flags missed ones |
| **Closed domain modeling** | Express "one of a fixed set" (algebraic data types) |
| **Safety** | No rogue subclasses breaking invariants |
| **Maintainability** | Adding a subtype surfaces every place that must handle it |
| **Clarity** | `permits` documents the full hierarchy |

---

## 6. Sealed Types + Records + Pattern Matching (the modern combo)

Sealed types shine when combined with **records** (immutable data carriers) and **pattern matching for switch** (Java 21):

```java
sealed interface Expr permits Num, Add, Mul {}
record Num(double value)        implements Expr {}
record Add(Expr left, Expr right) implements Expr {}
record Mul(Expr left, Expr right) implements Expr {}

double eval(Expr e) {
    return switch (e) {
        case Num n  -> n.value();
        case Add a  -> eval(a.left()) + eval(a.right());   // record deconstruction-friendly
        case Mul m  -> eval(m.left()) * eval(m.right());
    };
}
```
This trio (sealed + records + pattern matching) brings **functional-style, type-safe modeling** to Java. (Pattern matching for switch is covered in the switch/pattern-matching notes; records in their own 1.3 note.)

---

## 7. Sealed vs Final vs Abstract (when to use which)

| Goal | Use |
|------|-----|
| No subtypes at all | **`final`** class |
| Unrestricted extension (open API/framework) | normal (open) class/interface |
| A **known, fixed set** of subtypes | **`sealed`** with `permits` |
| Shared state/impl + open extension | **`abstract`** class (open) |
| Closed contract + exhaustive handling | **`sealed interface`** + records |

> Sealed is about **controlling the set of subtypes**; abstract/final are about **instantiation and extensibility**. They can combine — e.g., `public sealed abstract class Shape permits ...` (abstract for shared behavior, sealed for controlled subtypes).

---

## 8. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Permitted subtype not marked `final`/`sealed`/`non-sealed` | Add one of the three modifiers (compile error otherwise) |
| Listing a type in `permits` that doesn't actually extend it | Must be a real direct subtype |
| Permitted subtypes in a different module | Keep them in the same module (Java 17 rule) |
| Adding `default` to a sealed `switch` "just in case" | Omit it — let exhaustiveness checking protect you |
| Expecting `non-sealed` to keep things closed | `non-sealed` **reopens** extension downstream |
| Sealing when you actually want unlimited extension | Use a normal open type instead |
| Forgetting that adding a subtype breaks existing switches | That's intentional — handle the new case |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Domain modeling (Phase 12 DDD):** model closed sets of states/events/commands as sealed hierarchies (e.g., `OrderState permits Pending, Shipped, Delivered, Cancelled`) → exhaustive, type-safe state handling.
- **Result/error types:** `sealed interface ApiResult permits Ok, ValidationError, NotFound` makes error handling explicit and exhaustive instead of relying on exceptions or nullable returns.
- **Event-driven systems (Phase 11):** model a fixed catalog of domain events as a sealed hierarchy; consumers `switch` over them exhaustively.
- **Pattern matching pipelines:** sealed + records + switch replaces verbose visitor patterns and `instanceof` chains (Phase 14).
- **Safer refactoring:** when a new variant is added, the compiler points to every place that must be updated — invaluable in large codebases.

---

## 10. Quick Self-Check Questions

1. What does sealing a type accomplish that neither a normal class nor `final` can?
2. What are the three modifiers a permitted subtype must choose from, and what does each mean?
3. When can you omit the `permits` clause?
4. What does `non-sealed` do, and when would you use it?
5. Why do sealed types enable an exhaustive `switch` with no `default`?
6. What happens to existing `switch` statements when you add a new permitted subtype?
7. How do sealed types, records, and pattern matching combine, and what FP concept does this model?
8. What are the module/accessibility constraints on permitted subtypes?

---

## 11. Key Terms Glossary

- **Sealed type:** a class/interface that restricts which types may extend/implement it.
- **`sealed`:** keyword marking a type as restricted.
- **`permits`:** clause listing the allowed direct subtypes.
- **`final` (subtype):** closes the hierarchy at that subtype.
- **`sealed` (subtype):** continues controlled sealing with its own `permits`.
- **`non-sealed`:** reopens the hierarchy for unrestricted extension from that point.
- **Exhaustiveness checking:** compiler verification that a `switch` covers all subtypes.
- **Algebraic data type / sum type:** "a value is exactly one of a fixed set of variants."
- **Record:** immutable data carrier; pairs naturally with sealed types.
- **Pattern matching for switch:** matching on type (and deconstructing) in a `switch` (Java 21).
- **Closed hierarchy:** a type hierarchy with a known, complete set of subtypes.

---

*Previous topic: **Interfaces**.*
*Next topic in roadmap: **Records (Java 16+)**.*
