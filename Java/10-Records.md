# Records (Java 16+)

> **Phase 1 — Java Language Mastery → 1.3 Object-Oriented Programming**
> Goal: Master records — the compact syntax for immutable data carriers; canonical and compact constructors; adding custom methods; and a clear decision framework for records vs classes.

---

## 0. The Big Picture

A **record** is a special kind of class designed to be a **transparent, immutable carrier of data**. With one line, the compiler generates the boilerplate you'd otherwise write by hand: private final fields, a constructor, accessors, `equals()`, `hashCode()`, and `toString()`.

```java
public record Point(int x, int y) {}
```

That single line is roughly equivalent to ~30 lines of a traditional immutable class (see §2). Records were finalized in **Java 16** (previewed in 14/15).

> **The core idea:** when a class's purpose is simply to *hold data* (DTOs, value objects, query results, tuples), a record removes the ceremony and makes the intent obvious: "this is just data, and it's immutable."

Records build directly on the **immutability** and **encapsulation** concepts from earlier notes — they're the idiomatic, modern way to write a simple immutable type.

---

## 1. Compact Syntax for Data Carriers

### 1.1 Declaration
```java
public record Person(String name, int age) {}
```
The parameters in the header `(String name, int age)` are called the **record components**. From them, the compiler auto-generates everything below.

### 1.2 What the compiler generates automatically
| Generated member | Details |
|------------------|---------|
| **`private final` field** per component | `name`, `age` — immutable |
| **Canonical constructor** | `Person(String name, int age)` setting all fields |
| **Accessor per component** | `name()`, `age()` — **not** `getName()` (no JavaBean prefix!) |
| **`equals()`** | Value-based: equal if all components are equal |
| **`hashCode()`** | Consistent with `equals()`, based on all components |
| **`toString()`** | `Person[name=Sam, age=30]` |

```java
Person p = new Person("Sam", 30);
p.name();              // "Sam"   <- accessor, NOT getName()
p.age();               // 30
System.out.println(p); // Person[name=Sam, age=30]

Person p2 = new Person("Sam", 30);
p.equals(p2);          // true  -> value-based equality, for free
```

> ⚠️ **Accessor naming:** record accessors are `componentName()`, not `getComponentName()`. So it's `p.name()`, not `p.getName()`. (Some frameworks bridge this; see §6.)

---

## 2. Record vs the Equivalent Hand-Written Class

A record replaces all of this:
```java
// The traditional equivalent of  record Point(int x, int y) {}
public final class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int x() { return x; }
    public int y() { return y; }
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point p)) return false;
        return x == p.x && y == p.y;
    }
    @Override public int hashCode() { return Objects.hash(x, y); }
    @Override public String toString() { return "Point[x=" + x + ", y=" + y + "]"; }
}
```
≈ 30 lines → **1 line**. Less code = fewer bugs, clearer intent.

---

## 3. Records Are Immutable (by design)

- All fields are **`private final`** — no setters are generated, and you can't add them to change components.
- A record is implicitly **`final`** — it **cannot be extended** (no inheritance of records).
- A record **cannot extend another class** (it implicitly extends `java.lang.Record`), but it **can implement interfaces**.

```java
public record Money(String currency, long cents) implements Comparable<Money> {
    @Override public int compareTo(Money o) { return Long.compare(cents, o.cents); }
}
```

> Because they're immutable, records are **inherently thread-safe** and safe to use as **map keys** / set elements (their hashCode never changes) — the same benefits covered in the Encapsulation note.

### 3.1 The mutable-component caveat (defensive copying still matters!)
A record's *reference* fields are final, but if a component is a **mutable object** (e.g., a `List`, array, `Date`), the contents can still be changed from outside — the record is only **shallowly** immutable. For true immutability, defensively copy in a **compact constructor** (§4.2) and/or in an accessor:
```java
public record Team(String name, List<String> members) {
    public Team {                                  // compact constructor
        members = List.copyOf(members);            // defensive copy IN (unmodifiable)
    }
}
```
(Same shallow-vs-deep-copy lesson from the Encapsulation note — records don't exempt you from it.)

---

## 4. Constructors in Records

### 4.1 Canonical constructor
The **canonical constructor** matches the record header (all components). It's generated automatically, but you can **declare it explicitly** to add validation or transformation:
```java
public record Range(int start, int end) {
    public Range(int start, int end) {             // explicit canonical constructor
        if (start > end)
            throw new IllegalArgumentException("start > end");
        this.start = start;                        // must assign all fields
        this.end = end;
    }
}
```

### 4.2 Compact constructor (the idiomatic form)
A **compact constructor** is a shorthand: no parameter list, and you **don't assign the fields** — the compiler does the assignment **after** your code runs. Use it for **validation** and **normalization/defensive copying**:
```java
public record Range(int start, int end) {
    public Range {                                 // compact: no parens, no field assignment
        if (start > end)
            throw new IllegalArgumentException("start > end");
        // implicit: this.start = start; this.end = end;  (added by compiler)
    }
}
```
You can also **reassign the parameters** to normalize values before they're stored:
```java
public record User(String name, String email) {
    public User {
        name  = name.strip();                      // normalize (compiler assigns the result)
        email = email.toLowerCase();
    }
}
```

> **Compact constructor = the preferred place for validation and defensive copies.** It's cleaner than the full canonical constructor for most cases.

### 4.3 Additional (non-canonical) constructors
You can add extra constructors, but each **must delegate to the canonical constructor** via `this(...)`:
```java
public record Point(int x, int y) {
    public Point(int x) {        // extra constructor
        this(x, 0);              // MUST call the canonical constructor
    }
    public Point() {
        this(0, 0);
    }
}
```

---

## 5. Custom Methods and Members in Records

Records aren't limited to data — you can add behavior, as long as you don't break immutability.

### 5.1 Instance methods (derived values, behavior)
```java
public record Rectangle(double width, double height) {
    public double area() { return width * height; }        // computed method
    public boolean isSquare() { return width == height; }
}
```

### 5.2 Static fields, static methods, and factory methods
```java
public record Money(String currency, long cents) {
    public static final Money ZERO = new Money("USD", 0);   // static field OK

    public static Money ofDollars(double dollars) {          // static factory
        return new Money("USD", Math.round(dollars * 100));
    }
}
```

### 5.3 Overriding generated methods
You can override the auto-generated `toString()`, `equals()`, or an accessor if you need custom behavior:
```java
public record Temperature(double celsius) {
    @Override public String toString() { return celsius + "°C"; }
}
```

### 5.4 What records CANNOT do
| Not allowed | Reason |
|-------------|--------|
| **Instance fields** beyond the components | State must come only from components (transparency) |
| **`extends`** another class | Records already extend `java.lang.Record` |
| Being **extended** (subclassed) | Records are implicitly `final` |
| **Setters / mutating components** | Records are immutable |
| Non-`static` initializer blocks | Initialization goes through constructors |

> Static fields and methods are fine; only **extra instance state** is forbidden.

---

## 6. Records vs Classes — Decision Framework

### 6.1 Use a RECORD when...
- The type's main job is to **hold data** (DTOs, value objects, query/projection results, event payloads, tuples, composite map keys).
- You want **immutability** and **value-based equality**.
- The state is fully captured by a **fixed set of components**.
- You want to eliminate boilerplate.

### 6.2 Use a regular CLASS when...
- You need **mutable** state (fields that change after construction).
- You need to **extend another class** or be **extended** (inheritance).
- You need **encapsulation that hides** internal representation (records are *transparent* — components are exposed via accessors).
- The object has **identity-based** semantics (two distinct objects shouldn't be "equal" just because their fields match).
- You need significant **behavior** with internal state beyond the data.

### 6.3 Comparison table
| Aspect | **Record** | **Regular class** |
|--------|-----------|-------------------|
| Mutability | Immutable | Your choice |
| Boilerplate | Auto-generated | Manual |
| Equality | Value-based (auto) | Identity by default (override for value) |
| Inheritance | Can't extend/be extended | Full inheritance |
| Encapsulation | Transparent (components exposed) | Can hide internals |
| Accessors | `component()` | Anything (e.g., `getX()`) |
| Best for | Data carriers / value objects | Rich behavior, mutable or identity types |

### 6.4 Records + sealed interfaces (the powerful combo)
Records pair naturally with **sealed interfaces** (previous note) to model closed domains / algebraic data types, enabling exhaustive pattern-matching `switch`:
```java
sealed interface Shape permits Circle, Square {}
record Circle(double r)    implements Shape {}
record Square(double side) implements Shape {}
```
(Pattern matching expands on this in the switch/pattern-matching notes.)

---

## 7. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Expecting `getName()` accessors | Records use `name()` (no `get` prefix) |
| Assuming a record with a `List`/array is fully immutable | Defensively copy in the compact constructor |
| Trying to add mutable instance fields | Not allowed — model state via components only |
| Trying to extend a record or subclass it | Records are `final` and can't extend classes; use interfaces |
| Forgetting to delegate extra constructors to canonical | Non-canonical constructors must call `this(...)` |
| Using records for entities with identity/lifecycle | Use a class (e.g., JPA entities — see §8) |
| Assigning fields in a compact constructor | Don't — the compiler assigns; just validate/normalize |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **DTOs (Phase 5, 7):** records are now the idiomatic choice for request/response DTOs — immutable, concise, value-based. Jackson (2.12+) and Spring serialize/deserialize records well.
- **Value Objects (Phase 12 DDD):** records are a perfect fit for immutable value objects (`Money`, `Address`, `Coordinates`).
- **Query projections (Phase 5.4):** Spring Data JPA supports **record-based DTO projections** for read queries.
- **`@ConfigurationProperties` (Phase 5.2):** records can bind configuration (immutable config).
- **JPA entities:** ❌ **records are NOT suitable as JPA `@Entity` types** — Hibernate needs a no-arg constructor, mutable fields, and proxying. Use classes for entities, records for DTOs.
- **Sealed + records + pattern matching:** model results, events, and domain states as type-safe closed hierarchies (Phases 11, 12, 14).
- **Concurrency (Phase 1.10):** immutable records are inherently thread-safe — safe to share across threads without locks.

---

## 9. Quick Self-Check Questions

1. What does a single-line record declaration generate for you?
2. How are record accessors named, and how does that differ from JavaBean getters?
3. Why are records immutable and `final`? Can a record extend a class or implement interfaces?
4. What's the difference between the canonical and compact constructor? Where should validation go?
5. In a compact constructor, do you assign the fields yourself? Why or why not?
6. How do you make a record with a `List` component truly immutable?
7. What can a record NOT have or do (list three)?
8. When should you use a record vs a regular class?
9. Why are records unsuitable as JPA entities but ideal as DTOs?

---

## 10. Key Terms Glossary

- **Record:** a transparent, immutable data-carrier class with auto-generated members.
- **Record component:** a parameter in the record header; becomes a `private final` field + accessor.
- **Canonical constructor:** the constructor whose parameters match all components.
- **Compact constructor:** shorthand canonical constructor (no params, no field assignment) for validation/normalization.
- **Accessor:** auto-generated `component()` method (no `get` prefix).
- **Value-based equality:** equality determined by component values, not identity.
- **Transparent carrier:** a type whose state is fully exposed through its components.
- **`java.lang.Record`:** the implicit superclass of all records.
- **Defensive copy:** copying mutable components to preserve immutability.
- **Value Object:** (DDD) an immutable object defined by its values — a natural record use.
- **DTO:** Data Transfer Object — a common record use case in APIs.

---

*Previous topic: **Sealed Classes and Interfaces (Java 17)**.*
*Next topic in roadmap: **Enums (basic, with fields/methods, abstract methods, EnumSet/EnumMap, singleton via enum)**.*
