# Abstraction

> **Phase 1 — Java Language Mastery → 1.3 Object-Oriented Programming**
> Goal: Master abstraction — abstract classes (when and why), abstract methods, concrete methods inside abstract classes, and a clear decision framework for choosing **abstract class vs interface**.

---

## 0. The Big Picture

**Abstraction** is the OOP pillar of **hiding complexity by exposing only the essential, relevant features** of something, while suppressing the irrelevant details. You define **what** something does without (yet) specifying **how**.

> You drive a car using a steering wheel, pedals, and gear stick — you don't need to know how the engine combusts fuel. The controls are the **abstraction**; the engine internals are **hidden**.

In Java, abstraction is achieved with two tools:
1. **Abstract classes** (`abstract class`) — partial implementations meant to be extended.
2. **Interfaces** (`interface`) — pure contracts (covered in depth in their own note; compared here).

> **Abstraction vs Encapsulation:** Encapsulation **hides data** (the *how* of state). Abstraction **hides complexity/implementation** (the *what* vs *how* of behavior). They're complementary, not the same.

---

## 1. Abstract Methods

An **abstract method** is a method **declared without a body** — only its signature. It says "every concrete subclass *must* provide this behavior," but doesn't define it here.

```java
abstract class Shape {
    abstract double area();   // no body — just the contract (note the semicolon, no {})
}
```

### 1.1 Rules for abstract methods
- Declared with the `abstract` keyword and **no body** (ends with `;`).
- Can **only** exist inside an **abstract class** (or an interface).
- **Cannot** be `private`, `static`, or `final` (each would make overriding impossible — defeating the purpose).
- Any **concrete (non-abstract) subclass must override/implement** all inherited abstract methods, or it too must be declared `abstract`.

```java
class Circle extends Shape {
    private final double r;
    Circle(double r) { this.r = r; }
    @Override double area() { return Math.PI * r * r; }   // MUST implement
}
```

---

## 2. Abstract Classes (When and Why)

An **abstract class** is a class declared with the `abstract` keyword. It's a **partial blueprint** — it may contain abstract methods (no body) **and** concrete methods (with body), fields, constructors, and static members.

```java
abstract class Animal {
    protected String name;             // state (fields)

    Animal(String name) {              // constructor (called via subclass super())
        this.name = name;
    }

    abstract String sound();           // abstract: subclasses MUST define

    void sleep() {                     // concrete: shared by all subclasses
        System.out.println(name + " is sleeping");
    }
}
```

### 2.1 The defining property: cannot be instantiated
```java
// Animal a = new Animal("x");   // COMPILE ERROR: cannot instantiate abstract class
Animal a = new Dog("Rex");       // OK: instantiate a concrete subclass
```
It exists to be **extended**, not to be created directly. (It still has a constructor — invoked by subclasses via `super(...)` to initialize inherited state.)

### 2.2 When and why to use an abstract class
Use an abstract class when:
| Situation | Why an abstract class fits |
|-----------|----------------------------|
| Several classes share **common state + behavior** | Put shared fields and concrete methods in the base; subclasses inherit them |
| You want to provide a **partial implementation** | Implement the common parts; leave the varying parts abstract |
| You want to **enforce a template** of steps | Template Method pattern (§5) |
| The types form a genuine **IS-A** hierarchy | `Dog`/`Cat` IS-A `Animal` |
| You need **constructors** or **mutable instance fields** in the base | Interfaces can't hold instance state or constructors |

The core idea: **capture what's common once, force subclasses to fill in what differs.**

### 2.3 A class can be abstract even with no abstract methods
You can mark a class `abstract` purely to **prevent direct instantiation** (e.g., a base class that's meaningless on its own), even if all its methods are concrete:
```java
abstract class BaseService {   // abstract just to forbid `new BaseService()`
    void log(String m) { /* shared concrete logic */ }
}
```

---

## 3. Concrete Methods in Abstract Classes

This is a **key advantage** of abstract classes over (older) interfaces: an abstract class can mix **abstract methods (the contract)** with **concrete methods (shared implementation)**.

```java
abstract class PaymentProcessor {
    // CONCRETE: shared logic, written once, inherited by all subclasses
    final void process(double amount) {     // 'final' so subclasses can't break the flow
        validate(amount);
        boolean ok = charge(amount);        // abstract step — varies per processor
        if (ok) sendReceipt();
    }

    private void validate(double amount) {  // concrete helper
        if (amount <= 0) throw new IllegalArgumentException("amount <= 0");
    }
    private void sendReceipt() {            // concrete helper
        System.out.println("Receipt sent");
    }

    // ABSTRACT: each processor implements its own charging
    abstract boolean charge(double amount);
}

class StripeProcessor extends PaymentProcessor {
    @Override boolean charge(double amount) { /* Stripe API */ return true; }
}
class PayPalProcessor extends PaymentProcessor {
    @Override boolean charge(double amount) { /* PayPal API */ return true; }
}
```
The abstract class supplies the **invariant skeleton** (validate → charge → receipt) and concrete helpers; subclasses only provide the **variable part** (`charge`). This drastically reduces duplication.

---

## 4. Abstract Class vs Interface (Decision Framework)

This is one of the most important design decisions in OOP — and a perennial interview question.

### 4.1 Feature comparison
| Feature | **Abstract class** | **Interface** |
|---------|--------------------|---------------|
| Instantiable? | No | No |
| Methods with body | ✅ Yes (concrete) | ✅ Yes, but only `default` / `static` / `private` (Java 8/9+) |
| Abstract methods | ✅ Yes | ✅ Yes (implicitly `public abstract`) |
| Instance fields (state) | ✅ Yes | ❌ No (only `public static final` constants) |
| Constructors | ✅ Yes | ❌ No |
| Multiple inheritance | ❌ Extend **one** class | ✅ Implement **many** interfaces |
| Member access modifiers | Any (`private`/`protected`/`public`) | Members are `public` (since Java 9, also `private` helpers) |
| Relationship modeled | **IS-A** (tight, hierarchical) | **CAN-DO / capability** (looser) |
| When to add a method | Can add concrete method without breaking subclasses | Adding a method breaks implementers unless it's `default` |

### 4.2 The key distinctions
- **State:** Only abstract classes can hold **instance fields** and **constructors**. If you need shared mutable state, you need an abstract class.
- **Multiplicity:** A class extends **one** abstract class but implements **many** interfaces. Interfaces are the way to mix in multiple capabilities.
- **Evolution:** Since Java 8, interfaces can have **default methods**, so you *can* add behavior to an interface without breaking implementers — narrowing the historical gap. But interfaces still can't carry state.
- **Semantics:** Abstract class = "is a kind of" (strong IS-A). Interface = "is capable of / can do" (a role or capability), e.g., `Comparable`, `Serializable`, `Runnable`.

### 4.3 Decision framework — choose...
**Choose an ABSTRACT CLASS when:**
- Subclasses share **common state (fields)** and/or substantial **common implementation**.
- You need **constructors** to initialize that state.
- The types form a clear, single **IS-A hierarchy**.
- You want a **Template Method** skeleton with some steps fixed (`final`) and some abstract.

**Choose an INTERFACE when:**
- You're defining a **capability/role** that unrelated classes can adopt (e.g., `Comparable`, `Closeable`).
- You need **multiple inheritance** of type/behavior.
- You want **maximum decoupling** — "program to an interface" for DI, testing, and pluggability.
- You only need a **contract** (with optional default methods), not shared state.

**Modern default guidance:**
> **Prefer interfaces** for defining types and contracts (they're more flexible and enable multiple inheritance), and use **abstract classes** when you genuinely need shared state/implementation. A common idiom is an **interface + an abstract "skeletal" implementation** (`AbstractXxx`) that provides default behavior — best of both worlds (*Effective Java*, Item 20). Examples in the JDK: `List` (interface) + `AbstractList` (skeletal class).

```java
interface Collection<E> { /* contract */ }
abstract class AbstractCollection<E> implements Collection<E> {
    /* skeletal implementation of common methods */
}
```

### 4.4 Quick mental test
- "Do these things **share what they ARE**, including state?" → **abstract class**.
- "Do these things **share what they CAN DO**, possibly across unrelated hierarchies?" → **interface**.

---

## 5. Template Method Pattern (the natural home of abstract classes)

The payment example in §3 is the **Template Method pattern**: a `final` method in the abstract class defines the **invariant algorithm skeleton**, delegating the variable steps to abstract methods the subclasses implement.

```
process()  [final, in abstract class]   <- the fixed algorithm
   |-- validate()   [concrete]          <- shared
   |-- charge()     [abstract]          <- subclass-specific (the "hook")
   |-- sendReceipt()[concrete]          <- shared
```
- Guarantees the overall flow can't be broken (skeleton is `final`).
- Subclasses customize only the designated hook points.
- Used heavily in frameworks (e.g., Spring's `AbstractAuthenticationProcessingFilter`, `JdbcTemplate`). (Revisited in Phase 14 — Design Patterns.)

---

## 6. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Trying to instantiate an abstract class | Instantiate a concrete subclass |
| Forgetting to implement all abstract methods in a subclass | Implement them, or mark the subclass `abstract` too |
| Making an abstract method `private`/`static`/`final` | Not allowed — these prevent overriding |
| Using an abstract class where multiple inheritance is needed | Use interfaces (a class extends only one class) |
| Putting shared state in an interface | Interfaces can't have instance fields — use an abstract class |
| Defaulting to abstract classes out of habit | Prefer interfaces for contracts; abstract class only for shared state/impl |
| Deep abstract hierarchies | Keep shallow; favor composition (see Inheritance note) |
| Calling overridable (abstract) methods from a constructor | Subclass override runs before subclass fields are initialized |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Program to interfaces** is the backbone of Spring DI, testability, and loose coupling — interfaces define service/repository contracts; implementations are injected (Phase 5, 6, 14).
- **Skeletal implementations** (`AbstractXxx`) reduce boilerplate when many implementations share logic — a common framework pattern.
- **Template Method** appears throughout Spring (`JdbcTemplate`, `OncePerRequestFilter`, `AbstractRoutingDataSource`) — understanding abstraction makes these intuitive to extend.
- **Default methods** let library/interface authors evolve APIs without breaking implementers — relevant when designing your own SPIs.
- **Abstraction layering** (Controller → Service interface → Repository interface) is how clean architectures decouple concerns (Phase 14 hexagonal/clean architecture).

---

## 8. Quick Self-Check Questions

1. What is abstraction, and how does it differ from encapsulation?
2. What is an abstract method, and where can it be declared? Why can't it be `private`/`static`/`final`?
3. Why can't you instantiate an abstract class, and why does it still have a constructor?
4. Give a concrete reason to put a non-abstract (concrete) method in an abstract class.
5. List three things an abstract class can have that an interface cannot.
6. When should you choose an abstract class over an interface, and vice versa?
7. What is the "interface + skeletal implementation" idiom, and why is it useful?
8. Describe the Template Method pattern and why the skeleton method is often `final`.
9. Can an abstract class have zero abstract methods? Why might you do that?

---

## 9. Key Terms Glossary

- **Abstraction:** exposing essential features while hiding implementation complexity.
- **Abstract class:** a class that can't be instantiated; may mix abstract + concrete members.
- **Abstract method:** a method with a signature but no body; must be overridden by concrete subclasses.
- **Concrete method/class:** one with a full implementation / that can be instantiated.
- **Interface:** a pure contract type (with optional default/static/private methods).
- **Default method:** an interface method with a body (Java 8+).
- **Skeletal implementation:** an abstract class implementing an interface to provide common behavior (`AbstractXxx`).
- **Template Method pattern:** a fixed algorithm skeleton delegating variable steps to subclasses.
- **IS-A vs CAN-DO:** inheritance/abstract-class semantics vs interface/capability semantics.
- **Hook method:** an overridable step within a template method.

---

*Previous topic: **Polymorphism**.*
*Next topic in roadmap: **Interfaces (default/static/private methods, functional & marker interfaces, sealed interfaces)**.*
