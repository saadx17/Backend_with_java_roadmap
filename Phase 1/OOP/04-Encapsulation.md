# Encapsulation

> **Phase 1 — Java Language Mastery → 1.3 Object-Oriented Programming**
> Goal: Master encapsulation — getters/setters, why exposing fields directly is dangerous, how to build truly immutable objects, defensive copying, and the Builder pattern for safe construction.

---

## 0. The Big Picture

**Encapsulation** is one of the four pillars of OOP. It means:
1. **Bundling** data (fields) and the methods that operate on it into a single unit (the class), and
2. **Hiding** the internal state, exposing only a controlled, deliberate interface.

> Think of a class as a **capsule**: the messy internals are sealed inside; the outside world interacts only through the buttons (methods) you choose to expose.

Encapsulation builds directly on **access modifiers** (previous note): you make fields `private` and control all interaction through methods. The payoff is **safety, flexibility, and maintainability** — you can change internals freely as long as the public contract stays stable.

---

## 1. Getters and Setters

The classic encapsulation mechanism: keep fields `private`, expose them through **accessor methods**.

```java
public class Person {
    private String name;     // hidden state
    private int age;

    // Getter (accessor)
    public String getName() { return name; }

    // Setter (mutator) — with validation
    public void setAge(int age) {
        if (age < 0)
            throw new IllegalArgumentException("Age cannot be negative");
        this.age = age;
    }
    public int getAge() { return age; }
}
```

### 1.1 Naming conventions (JavaBean convention)
| Field type | Getter | Setter |
|------------|--------|--------|
| Normal field `name` | `getName()` | `setName(...)` |
| `boolean` field `active` | `isActive()` | `setActive(...)` |

Frameworks (Jackson, JPA/Hibernate, Spring, validation) **rely on this convention** to discover and bind properties reflectively — so following it isn't just style, it's interoperability.

### 1.2 What getters/setters buy you
- **Validation** in setters (reject invalid state — see `setAge`).
- **Computed/derived values** in getters without a stored field:
  ```java
  public double getBmi() { return weight / (height * height); }
  ```
- **Read-only / write-only** properties (provide only a getter, or only a setter).
- **Future-proofing:** you can change the internal representation, add logging, lazy-loading, or caching **without changing callers**.

### 1.3 Getters/setters are NOT automatically "good"
> ⚠️ Blindly generating a getter **and** setter for every field defeats the purpose — that's just a public field with extra steps. Add a setter **only if mutation is genuinely needed**, and put real logic/validation where it belongs. Prefer immutability (§3) when you can.

---

## 2. Why Direct Field Access Is Bad

```java
// BAD: public fields
public class Point {
    public int x;
    public int y;
}
Point p = new Point();
p.x = -999;     // anyone can set ANY value, anytime — no control
```

Problems with exposing fields publicly:

| Problem | Explanation |
|---------|-------------|
| **No validation** | Callers can set invalid/inconsistent values (negative age, null where forbidden) |
| **Broken invariants** | The class can't guarantee its rules hold (e.g., `start <= end`) |
| **No flexibility to change** | The field name/type becomes part of the public API; renaming/refactoring breaks all callers |
| **No computed values** | Can't replace a field with a derived calculation later |
| **No thread safety** | Uncontrolled concurrent mutation → race conditions (Phase 1.10) |
| **Hard to debug** | State can change from anywhere; no single choke point to log/breakpoint |
| **Framework incompatibility** | JPA/Jackson/validation expect accessor methods |

> **Core idea:** a `private` field + accessor methods gives you a **single point of control**. With public fields, you have *zero* control and the field becomes a permanent, unchangeable contract.

---

## 3. Immutable Objects

An **immutable object** is one whose state **cannot change after construction**. Immutability is the strongest, safest form of encapsulation.

### 3.1 Why immutability is powerful
| Benefit | Why |
|---------|-----|
| **Thread-safe by default** | No mutation → no race conditions → no locks needed (Phase 1.10) |
| **Safe to share/cache** | Can be freely passed around and reused |
| **Predictable** | State never changes unexpectedly behind your back |
| **Valid by construction** | If the constructor enforces invariants, they hold forever |
| **Good map keys** | Hashcode never changes → safe in `HashMap`/`HashSet` |

### 3.2 The recipe for an immutable class
```java
public final class Money {                    // 1. final class: can't be subclassed
    private final String currency;            // 2. all fields private + final
    private final long amountCents;

    public Money(String currency, long amountCents) {  // 3. set all via constructor
        this.currency = currency;
        this.amountCents = amountCents;
    }

    public String getCurrency()  { return currency; }   // 4. getters only, NO setters
    public long getAmountCents() { return amountCents; }

    // 5. "mutators" RETURN A NEW OBJECT instead of changing this one
    public Money plus(long cents) {
        return new Money(currency, amountCents + cents);
    }
}
```

The five rules:
1. Make the **class `final`** (so subclasses can't add mutability / override behavior).
2. Make **all fields `private final`**.
3. **Set all fields in the constructor** — no setters.
4. Provide **getters only** (no mutators).
5. For "changes," **return a new instance** (like `String`, `LocalDate`, `BigDecimal`).
6. **(Critical) Defensively copy** any mutable fields on the way in *and* out — see §4.

### 3.3 Examples in the JDK
`String`, `Integer`/`Long` (and all wrappers), `LocalDate`/`LocalDateTime`/`Instant`, `BigDecimal`, `BigInteger`, `UUID` are all immutable. This is why `String.replace()` returns a *new* String rather than modifying the original.

### 3.4 Records (the modern shortcut — preview)
Java 16+ **records** generate an immutable class for you (private final fields, accessors, constructor, `equals`/`hashCode`/`toString`):
```java
public record Money(String currency, long amountCents) {}
```
(Records get their own dedicated note in 1.3 — but know they're the idiomatic way to write simple immutable data carriers.)

---

## 4. Defensive Copying

The leak that breaks encapsulation: even with `private final` fields and no setters, if a field is a **mutable object** (array, collection, `Date`, custom mutable type), exposing or storing the *reference* lets outsiders change your internal state. **Defensive copying** prevents this.

### 4.1 The problem
```java
public final class Schedule {
    private final List<String> events;
    public Schedule(List<String> events) {
        this.events = events;                  // ❌ stores the SAME list reference
    }
    public List<String> getEvents() {
        return events;                         // ❌ hands out the internal list
    }
}

List<String> input = new ArrayList<>(List.of("A"));
Schedule s = new Schedule(input);
input.add("HACKED");          // mutates Schedule's internal state from outside!
s.getEvents().add("ALSO HACKED");  // mutates it again via the getter!
```
Both the constructor and the getter "leaked" the internal mutable list. The object is **not** actually immutable.

### 4.2 The fix: copy on the way IN and on the way OUT
```java
public final class Schedule {
    private final List<String> events;

    public Schedule(List<String> events) {
        this.events = new ArrayList<>(events);     // ✅ copy IN (don't trust caller)
    }

    public List<String> getEvents() {
        return List.copyOf(events);                // ✅ copy OUT (unmodifiable copy)
        // or: return Collections.unmodifiableList(events);
    }
}
```
- **Copy IN:** so later changes to the caller's collection don't affect you.
- **Copy OUT:** so callers can't mutate your internal collection via the returned reference.

### 4.3 Shallow vs deep copy (important nuance)
A copy of a collection of **mutable elements** is still vulnerable — copying the *list* duplicates the references, but the *elements* are shared:
```java
// If elements are mutable (e.g., StringBuilder, custom mutable objects),
// new ArrayList<>(list) is a SHALLOW copy — elements are still shared.
// For full safety, deep-copy each element too.
```
| Copy type | What's duplicated |
|-----------|-------------------|
| **Shallow** | The container only; elements still shared |
| **Deep** | The container *and* all nested objects (fully independent) |

> Rule of thumb: if a field is mutable, defensively copy it. If its *elements* are also mutable and you need true independence, deep-copy. Immutable element types (e.g., `String`) make shallow copies sufficient.

### 4.4 When you can skip it
- The field type is **immutable** (`String`, `Integer`, `LocalDate`, another immutable class) → no copy needed.
- You store an **unmodifiable view** *and* control the source.

---

## 5. The Builder Pattern for Construction

When an object has **many fields** (especially many optional ones), telescoping constructors and giant parameter lists become unreadable and error-prone. The **Builder pattern** solves this with a fluent, readable construction API — and works beautifully with immutability.

### 5.1 The problem it solves
```java
// Telescoping constructors: hard to read, easy to mix up arguments
new User("Sam", "sam@x.com", 30, true, null, "admin", false);
//                                  ^ which boolean is which? what's null?
```

### 5.2 A Builder implementation
```java
public final class User {
    private final String name;        // required
    private final String email;       // required
    private final int age;            // optional
    private final String role;        // optional

    private User(Builder b) {         // private constructor: only Builder calls it
        this.name  = b.name;
        this.email = b.email;
        this.age   = b.age;
        this.role  = b.role;
    }

    public static Builder builder(String name, String email) {
        return new Builder(name, email);   // required fields up front
    }

    public static class Builder {
        private final String name;     // required
        private final String email;
        private int age = 0;           // optional, with defaults
        private String role = "USER";

        private Builder(String name, String email) {
            this.name = name;
            this.email = email;
        }
        public Builder age(int age)      { this.age = age; return this; }   // fluent
        public Builder role(String role) { this.role = role; return this; }

        public User build() {
            // central place to validate invariants before creating the object
            if (age < 0) throw new IllegalArgumentException("age < 0");
            return new User(this);
        }
    }
}
```

### 5.3 Usage — readable & safe
```java
User u = User.builder("Sam", "sam@x.com")   // required args explicit
             .age(30)                         // optional, named & fluent
             .role("ADMIN")
             .build();                        // validation happens here
```

### 5.4 Why the Builder is great
| Benefit | Why |
|---------|-----|
| **Readability** | Each value is labeled by its setter name |
| **Optional parameters** | Set only what you need; others get defaults |
| **Immutability-friendly** | Produces a fully-built immutable object in one `build()` |
| **Validation choke point** | Enforce invariants in `build()` |
| **No invalid intermediate state** | The real object isn't created until it's complete and valid |

### 5.5 In practice
- **Lombok** `@Builder` generates this boilerplate automatically (common in Spring projects).
- Many libraries expose builders (`HttpClient.newBuilder()`, `Stream.builder()`, Spring's `WebClient.builder()`).
- (The Builder pattern reappears in Phase 14 — Design Patterns.)

---

## 6. Putting It All Together — Encapsulation Checklist

```
[ ] Fields are private (never public, except constants: public static final)
[ ] Mutation only through methods that validate
[ ] No setter unless mutation is genuinely required
[ ] Prefer immutability (final class, final fields, no setters)
[ ] Defensively copy mutable fields IN (constructor) and OUT (getters)
[ ] Use a Builder (or record) for objects with many fields
[ ] Follow JavaBean naming (getX/isX/setX) for framework compatibility
[ ] Public API exposes behavior, not raw data
```

---

## 7. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Auto-generating getters **and** setters for every field | Add setters only when mutation is needed; prefer immutability |
| "Immutable" class that leaks a mutable collection/array | Defensive copy in constructor *and* getter |
| Shallow-copying a collection of mutable elements | Deep-copy when true independence is required |
| Public fields "for convenience" | Use private fields + accessors (or a record) |
| Forgetting `final` on class/fields in immutable types | Mark class `final`, fields `private final` |
| Validating in callers instead of in setter/`build()` | Centralize validation inside the class |
| Returning the internal array directly | Return a copy (`array.clone()`, `List.copyOf`) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **DTOs & entities (Phase 5):** encapsulation keeps domain objects valid; JPA/Jackson rely on JavaBean accessors. Immutable DTOs (records) are increasingly standard for API responses.
- **Thread safety (Phase 1.10):** immutable objects are inherently thread-safe — hugely valuable in concurrent backends; eliminates whole classes of bugs.
- **Defensive copying** prevents shared-mutable-state bugs when passing objects between layers/threads.
- **Builder pattern** is everywhere in Spring/HTTP clients and is the cleanest way to construct configuration and request objects.
- **Domain-Driven Design (Phase 12):** Value Objects are immutable; Entities encapsulate invariants — encapsulation is the foundation of good domain modeling.
- **Security (Phase 15):** hiding internal state and validating at boundaries is the code-level expression of defense in depth.

---

## 9. Quick Self-Check Questions

1. What two things does encapsulation combine, and what problem does it solve?
2. Give three concrete benefits of getters/setters over public fields.
3. Why is a public field worse than a private field with accessors?
4. List the rules for making a class truly immutable.
5. What is defensive copying, and why must you copy both *in* and *out*?
6. What's the difference between a shallow and a deep copy — when does each suffice?
7. When immutability isn't enough by itself, what mutable-field trap remains? How do you fix it?
8. What problems does the Builder pattern solve, and where does validation belong in it?
9. Why are immutable objects inherently thread-safe?

---

## 10. Key Terms Glossary

- **Encapsulation:** bundling data + behavior and hiding internal state behind a controlled interface.
- **Getter / accessor:** method that returns a field's value.
- **Setter / mutator:** method that changes a field's value (ideally with validation).
- **JavaBean convention:** `getX()`/`isX()`/`setX()` naming that frameworks rely on.
- **Invariant:** a rule about object state that must always hold.
- **Immutable object:** an object whose state can't change after construction.
- **Defensive copying:** copying mutable inputs/outputs to protect internal state.
- **Shallow copy:** duplicates the container, shares the elements.
- **Deep copy:** duplicates the container and all nested objects.
- **Builder pattern:** fluent, step-by-step construction producing a complete (often immutable) object.
- **Record:** Java 16+ concise immutable data carrier.
- **Value Object:** (DDD) small immutable object defined by its values.

---

*Previous topic: **Access Modifiers**.*
*Next topic in roadmap: **Inheritance (`extends`, overriding, `super`, IS-A, fragile base class, when NOT to use)**.*
