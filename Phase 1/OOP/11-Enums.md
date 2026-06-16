# Enums

> **Phase 1 — Java Language Mastery → 1.3 Object-Oriented Programming**
> Goal: Master enums — from basic constant sets to enums with fields/constructors/methods, enums with abstract methods, `EnumSet`/`EnumMap`, enums in switch, and the enum singleton idiom.

---

## 0. The Big Picture

An **enum** (enumeration) is a special class representing a **fixed, finite set of named constants**. Each constant is a single, predefined instance of the enum type — created once, type-safe, and impossible to instantiate more of.

```java
public enum Day { MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY }
```

> **Why enums over constants?** Before enums, people used `public static final int MONDAY = 1;` — which is **type-unsafe** (any `int` could be passed), gives no namespace, and prints meaningless numbers. Enums fix all of this: a method taking a `Day` can *only* receive a real `Day`.

Enums are far more powerful than constant lists — they're **full-fledged classes** that can have fields, constructors, methods, and even per-constant behavior.

---

## 1. Basic Enums

### 1.1 Declaration and use
```java
public enum Direction { NORTH, SOUTH, EAST, WEST }

Direction d = Direction.NORTH;        // type-safe
if (d == Direction.NORTH) { ... }     // compare with == (safe — single instances)
```
- Each constant (`NORTH`, etc.) is a `public static final` instance of `Direction`.
- Compare enums with **`==`** (not `.equals()`) — they're singletons, so `==` is safe, null-safe, and clear (though `.equals()` also works).

### 1.2 Built-in methods (every enum gets these for free)
| Method | Returns |
|--------|---------|
| `values()` | An array of all constants (in declaration order) |
| `valueOf(String)` | The constant matching the name (throws `IllegalArgumentException` if none) |
| `name()` | The exact declared name as a `String` |
| `ordinal()` | The zero-based position in declaration order |
| `compareTo()` | Compares by `ordinal()` (enums implement `Comparable`) |

```java
for (Direction dir : Direction.values())      // NORTH SOUTH EAST WEST
    System.out.println(dir.name() + " = " + dir.ordinal());

Direction n = Direction.valueOf("NORTH");     // -> Direction.NORTH
```

> ⚠️ **Never persist or rely on `ordinal()`** for business logic or database storage — reordering or inserting constants silently changes ordinals and corrupts data. Store the `name()` (string) instead, or an explicit field (§2). This is critical for JPA (`@Enumerated(EnumType.STRING)` — see §8).

### 1.3 Enums are real classes
- Implicitly extend `java.lang.Enum` (so they **can't extend another class**), but **can implement interfaces**.
- Are implicitly `final` (can't be subclassed) — except the special anonymous-subclass case for constant-specific bodies (§4).
- Get `equals()`, `hashCode()`, `toString()` (returns `name()` by default) from `Enum`.

---

## 2. Enums with Fields, Constructors, and Methods

Enums can carry **data** and **behavior**. Each constant can be constructed with arguments, and you can add fields, a (private) constructor, and methods.

```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),      // each constant passes constructor args
    EARTH  (5.976e+24, 6.37814e6),
    JUPITER(1.9e+27,   7.1492e7);      // <- semicolon required after the last constant
                                       //    when there are more members below

    private final double mass;          // fields (should be final -> immutable)
    private final double radius;

    Planet(double mass, double radius) { // constructor is implicitly private
        this.mass = mass;
        this.radius = radius;
    }

    public double mass()   { return mass; }
    public double radius() { return radius; }

    public double surfaceGravity() {     // behavior
        return 6.67300E-11 * mass / (radius * radius);
    }
}

double g = Planet.EARTH.surfaceGravity();
```

### 2.1 Key rules
- **Constants come first**, then a **semicolon**, then fields/constructor/methods.
- The **constructor is implicitly `private`** (you can't `new` an enum) — it's called once per constant at class load.
- Fields should be **`final`** to keep enum constants immutable (they're effectively singletons shared everywhere).

### 2.2 A common idiom: explicit code/value field
Instead of relying on `ordinal()`, attach a stable code:
```java
public enum Status {
    ACTIVE("A"), INACTIVE("I"), DELETED("D");

    private final String code;
    Status(String code) { this.code = code; }
    public String code() { return code; }

    public static Status fromCode(String code) {   // reverse lookup
        for (Status s : values())
            if (s.code.equals(code)) return s;
        throw new IllegalArgumentException("Unknown code: " + code);
    }
}
```

---

## 3. Enums Implementing Interfaces

Enums can implement interfaces, letting each enum participate in polymorphism:
```java
interface Describable { String describe(); }

public enum Coin implements Describable {
    PENNY(1), NICKEL(5), DIME(10), QUARTER(25);
    private final int cents;
    Coin(int cents) { this.cents = cents; }
    @Override public String describe() { return name() + " = " + cents + "¢"; }
}
```

---

## 4. Enums with Abstract Methods (Constant-Specific Behavior)

The most powerful enum feature: each constant can provide its **own implementation** of a method. You declare an **abstract method** on the enum, and **each constant must override it** in its own body. This makes the enum a clean alternative to `switch`-based behavior.

```java
public enum Operation {
    PLUS("+")  { public int apply(int a, int b) { return a + b; } },
    MINUS("-") { public int apply(int a, int b) { return a - b; } },
    TIMES("*") { public int apply(int a, int b) { return a * b; } },
    DIVIDE("/"){ public int apply(int a, int b) { return a / b; } };

    private final String symbol;
    Operation(String symbol) { this.symbol = symbol; }

    public abstract int apply(int a, int b);    // each constant implements this
    public String symbol() { return symbol; }
}

int r = Operation.PLUS.apply(2, 3);             // 5
for (Operation op : Operation.values())
    System.out.println(2 + " " + op.symbol() + " 3 = " + op.apply(2, 3));
```

### 4.1 How it works
- Each constant with a `{ ... }` body is actually an **anonymous subclass** of the enum.
- This is the **"constant-specific method implementation"** pattern — behavior lives *with* each constant, instead of a giant `switch` that must be updated whenever you add a constant.
- Adding a new constant **forces** you to implement the abstract method → compiler-enforced completeness (similar safety to sealed types' exhaustiveness).

> This is the idiomatic *Effective Java* approach (Item 34) — prefer constant-specific methods over `switch` on enums for varying behavior.

---

## 5. Enums in Switch Statements

Enums integrate cleanly with `switch` — you reference constants **without** the enum-type prefix:

```java
String activity = switch (day) {            // enhanced switch (Java 14+)
    case SATURDAY, SUNDAY -> "Rest";        // no Day. prefix needed
    case MONDAY           -> "Gym";
    default               -> "Work";
};
```
Traditional form:
```java
switch (day) {
    case MONDAY: System.out.println("Gym"); break;
    case SATURDAY:
    case SUNDAY: System.out.println("Rest"); break;
    default: System.out.println("Work");
}
```

### 5.1 Exhaustiveness
With the **enhanced switch expression**, if you cover **all** enum constants, you don't need a `default` (the compiler verifies exhaustiveness). If a new constant is added later and you used a switch *expression* without `default`, it won't compile until handled — a useful safety net (similar to sealed types).

> For behavior that varies per constant, prefer **constant-specific methods** (§4) over switch; for simple branching at the call site, switch is fine. (switch/pattern-matching details in their own notes.)

---

## 6. EnumSet and EnumMap (Specialized, High-Performance Collections)

The JDK provides two collections **optimized specifically for enum keys** — far faster and more memory-efficient than `HashSet`/`HashMap` for enums.

### 6.1 `EnumSet` — a set of enum constants
Internally implemented as a **bit vector** (each constant = one bit) → extremely fast and compact.
```java
EnumSet<Day> weekend  = EnumSet.of(Day.SATURDAY, Day.SUNDAY);
EnumSet<Day> weekdays = EnumSet.complementOf(weekend);   // all the others
EnumSet<Day> all      = EnumSet.allOf(Day.class);
EnumSet<Day> none     = EnumSet.noneOf(Day.class);
EnumSet<Day> range    = EnumSet.range(Day.MONDAY, Day.FRIDAY);

if (weekend.contains(Day.SATURDAY)) { ... }   // O(1), bitwise
```
> Use `EnumSet` instead of `int` bit flags (`1 | 2 | 4`) — it's type-safe, readable, and just as fast (*Effective Java* Item 36).

### 6.2 `EnumMap` — a map keyed by an enum
Internally backed by an **array indexed by `ordinal()`** → very fast, ordered by constant declaration, no hashing.
```java
EnumMap<Day, String> plans = new EnumMap<>(Day.class);
plans.put(Day.MONDAY, "Gym");
plans.put(Day.FRIDAY, "Movie");
String m = plans.get(Day.MONDAY);             // O(1), array access
```
> Prefer `EnumMap` over `HashMap` whenever the keys are enum constants — better performance, predictable iteration order, less memory.

| Collection | Backed by | Use when |
|------------|-----------|----------|
| `EnumSet` | Bit vector | A set of enum values |
| `EnumMap` | Array indexed by ordinal | A map with enum keys |

---

## 7. Singleton via Enum (the Best Singleton)

A single-constant enum is the **most robust way to implement the Singleton pattern** in Java (*Effective Java* Item 3).

```java
public enum DatabaseConnection {
    INSTANCE;                                  // the single instance

    private final String url = "jdbc:...";
    public void query(String sql) { /* ... */ }
}

// usage:
DatabaseConnection.INSTANCE.query("SELECT 1");
```

### 7.1 Why enum singletons are superior
| Benefit | Explanation |
|---------|-------------|
| **Thread-safe instantiation** | The JVM guarantees enum constants are created once, safely, at class load |
| **Serialization-safe** | Enums handle serialization correctly — no risk of creating a second instance on deserialization (a real flaw of classic singletons) |
| **Reflection-safe** | The JVM prevents reflective instantiation of enums (can't be broken via reflection) |
| **Concise** | No double-checked locking, no `volatile`, no boilerplate |

> Classic singleton implementations (lazy holder, double-checked locking) can be broken by reflection or serialization. The enum singleton is immune to both — it's the recommended approach when a singleton is genuinely needed. (In Spring, you usually use a singleton-scoped bean instead — see §9.)

---

## 8. Enums and Persistence / Serialization (must-know)

> **Store enums by `name()` (string), never by `ordinal()`.**

- `ordinal()` is **fragile**: inserting or reordering constants shifts every later ordinal, silently corrupting stored data.
- In **JPA/Hibernate**, always use `@Enumerated(EnumType.STRING)` (the default is `ORDINAL` — avoid it!):
  ```java
  @Enumerated(EnumType.STRING)
  private Status status;            // stored as "ACTIVE", resilient to reordering
  ```
- In JSON (Jackson), enums serialize to their `name()` by default — stable across versions.

---

## 9. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Persisting/relying on `ordinal()` | Use `name()` or an explicit code field; `@Enumerated(EnumType.STRING)` |
| Mutable fields in enum constants | Make fields `final` — enum constants are shared singletons |
| `valueOf()` with an unknown name | It throws — validate input or catch `IllegalArgumentException` |
| Using `int` bit flags instead of `EnumSet` | Use `EnumSet` (type-safe, fast) |
| Using `HashMap` with enum keys | Use `EnumMap` (faster, ordered) |
| Big `switch` on enums for varying behavior | Use constant-specific (abstract) methods |
| Trying to extend an enum | Enums are `final`; implement interfaces instead |
| Comparing enums with `.equals()` only | `==` is safe, null-safe, and clearer for enums |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Domain status/state modeling:** order status, user roles, payment states — enums make these type-safe and self-documenting (`OrderStatus.SHIPPED`).
- **JPA entities (Phase 5.4):** `@Enumerated(EnumType.STRING)` is a standard, must-know mapping to store enums safely.
- **State machines & constant-specific methods:** model workflow transitions with enums that carry behavior (Phase 5/12).
- **Strategy via enums:** enums with abstract methods implement the Strategy pattern compactly (Phase 14).
- **Configuration & flags:** `EnumSet` for feature flags / permission sets; `EnumMap` for per-enum config — efficient and clear.
- **Singletons:** enum singleton is the canonical thread-safe singleton (though Spring beans usually fill this role).
- **Validation/serialization:** enums map cleanly to API request/response values via Jackson.

---

## 11. Quick Self-Check Questions

1. Why are enums better than `public static final int` constants?
2. What methods does every enum get for free, and which one should you never persist?
3. How do you add data and behavior to an enum? Why is the constructor private?
4. What are constant-specific (abstract) methods, and why are they better than a `switch`?
5. Can an enum extend a class? Implement an interface? Be subclassed?
6. When and why would you use `EnumSet` and `EnumMap` instead of `HashSet`/`HashMap`?
7. Why is the enum singleton the most robust singleton implementation?
8. Why must you use `@Enumerated(EnumType.STRING)` in JPA?
9. Should you compare enums with `==` or `.equals()`? Why?

---

## 12. Key Terms Glossary

- **Enum:** a type representing a fixed set of named constant instances.
- **Constant:** a single predefined instance of the enum (`public static final`).
- **`values()`:** array of all constants in declaration order.
- **`valueOf(String)`:** look up a constant by name.
- **`name()`:** the declared name of a constant (stable; safe to persist).
- **`ordinal()`:** zero-based declaration position (fragile; don't persist).
- **Constant-specific method:** per-constant override of an enum abstract method (anonymous subclass).
- **`EnumSet`:** bit-vector-backed set optimized for enum elements.
- **`EnumMap`:** array-backed map optimized for enum keys.
- **Enum singleton:** a single-constant enum used as a robust Singleton.
- **`@Enumerated(EnumType.STRING)`:** JPA mapping to store an enum by name.
- **`java.lang.Enum`:** the implicit superclass of all enums.

---

*Previous topic: **Records (Java 16+)**.*
*Next topic in roadmap: **Inner Classes (member, static nested, local, anonymous)**.*
