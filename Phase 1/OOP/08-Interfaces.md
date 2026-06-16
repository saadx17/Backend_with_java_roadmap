# Interfaces

> **Phase 1 — Java Language Mastery → 1.3 Object-Oriented Programming**
> Goal: Master interfaces — definition & implementation, multiple implementation, default/static/private methods, functional interfaces, marker interfaces, interface segregation, and sealed interfaces.

---

## 0. The Big Picture

An **interface** is a **pure contract**: it declares **what** a type can do (a set of method signatures) without dictating **how**. Any class that `implements` the interface promises to provide those behaviors. Interfaces define **capability/role** ("CAN-DO") rather than identity ("IS-A").

```java
interface Drawable {
    void draw();              // contract: implementers must provide draw()
}

class Circle implements Drawable {
    @Override public void draw() { System.out.println("drawing circle"); }
}
```

Interfaces are the foundation of **loose coupling** in Java — "program to an interface, not an implementation." They power dependency injection, testing/mocking, and pluggable designs.

> Recap of the abstract-class-vs-interface decision is in the Abstraction note; this note dives into interfaces themselves.

---

## 1. Interface Definition and Implementation

### 1.1 Defining an interface
```java
interface Vehicle {
    int MAX_SPEED = 200;          // implicitly public static final (a CONSTANT)
    void start();                 // implicitly public abstract
    void stop();
}
```

**Implicit modifiers** (you usually omit them):
| Member | Implicitly... |
|--------|---------------|
| Fields | `public static final` (constants — no instance state allowed) |
| Abstract methods | `public abstract` |
| Default/static methods | `public` (unless `private`, Java 9+) |

> ⚠️ Interfaces **cannot have instance fields** — only constants. They hold no per-object state. This is the key structural difference from abstract classes.

### 1.2 Implementing an interface
```java
class Car implements Vehicle {
    @Override public void start() { System.out.println("vroom"); }
    @Override public void stop()  { System.out.println("stopped"); }
}
```
- A concrete class **must implement all** abstract methods (or be declared `abstract`).
- Implementing methods must be `public` (can't reduce the interface's implied `public` visibility).

### 1.3 Using interfaces polymorphically
```java
Vehicle v = new Car();   // reference type = interface, runtime type = Car
v.start();               // dynamic dispatch -> Car.start()
```
You write code against the **interface type**, and it works with any implementation — current or future (runtime polymorphism, previous note).

### 1.4 Interfaces extending interfaces
An interface can `extends` **one or more** other interfaces (note: `extends`, not `implements`):
```java
interface A { void a(); }
interface B { void b(); }
interface C extends A, B {   // C inherits both contracts
    void c();
}
// A class implementing C must provide a(), b(), and c().
```

---

## 2. Multiple Interface Implementation

A class can implement **many** interfaces — this is how Java achieves **multiple inheritance of type** (recall the single-class-inheritance limitation from the Inheritance note).

```java
interface Camera { void takePhoto(); }
interface GPS    { void navigate(); }
interface Phone  { void call(); }

class Smartphone implements Camera, GPS, Phone {
    @Override public void takePhoto() { /* ... */ }
    @Override public void navigate()  { /* ... */ }
    @Override public void call()      { /* ... */ }
}
```
- A `Smartphone` **IS-A** `Camera`, `GPS`, **and** `Phone` simultaneously — it can be used wherever any of those types is expected.
- This avoids the diamond problem for **state** because interfaces carry no instance fields.

### 2.1 The diamond problem with default methods
If two implemented interfaces provide **conflicting default methods** (same signature), the compiler **forces you to resolve it** by overriding and choosing explicitly:
```java
interface X { default String hi() { return "X"; } }
interface Y { default String hi() { return "Y"; } }

class Z implements X, Y {
    @Override public String hi() {
        return X.super.hi();      // explicitly pick X's version (or Y's, or your own)
    }
}
```
> `InterfaceName.super.method()` is the syntax to call a specific interface's default method.

---

## 3. Default Methods (Java 8+)

A **default method** is an interface method **with a body**, marked `default`. Implementing classes inherit it automatically and may override it.

```java
interface Greeter {
    String name();
    default String greet() {                 // has a body
        return "Hello, " + name();
    }
}
class English implements Greeter {
    @Override public String name() { return "Sam"; }
    // greet() inherited for free; can override if needed
}
```

### 3.1 Why default methods exist
**Interface evolution.** Before Java 8, adding a method to an interface broke **every** implementer. Default methods let you add new behavior to an interface **without breaking existing implementations** — the JDK used this to add `stream()`, `forEach()`, etc. to existing collection interfaces.

```java
// Added to the Collection interface in Java 8 — old code kept compiling:
interface Collection<E> {
    default Stream<E> stream() { /* default impl */ ... }
}
```

### 3.2 Default method guidelines
- Use them for **convenience/optional behavior** or **backward-compatible API evolution**.
- Don't use them to fake stateful behavior (interfaces still have no instance fields).
- They reduce — but don't eliminate — the need for abstract "skeletal" classes.

---

## 4. Static Methods in Interfaces (Java 8+)

Interfaces can have `static` methods with bodies — utility/factory methods that belong to the interface itself (called on the interface name, **not** inherited by implementers).

```java
interface Validator {
    boolean isValid(String s);

    static Validator notEmpty() {            // static factory method
        return s -> s != null && !s.isBlank();
    }
}

Validator v = Validator.notEmpty();          // called on the interface
```
- Called via `InterfaceName.method()`, never via an instance or implementing class.
- Common for **factory methods** and **helpers** related to the interface (e.g., `Comparator.naturalOrder()`, `Comparator.comparing(...)`).

---

## 5. Private Methods in Interfaces (Java 9+)

Since Java 9, interfaces can have **`private`** (and `private static`) methods. These are **helper methods** to share code between **default/static methods** without exposing it in the public API.

```java
interface Logger {
    default void info(String msg)  { log("INFO",  msg); }
    default void error(String msg) { log("ERROR", msg); }

    private void log(String level, String msg) {     // private helper, not part of contract
        System.out.println("[" + level + "] " + msg);
    }
}
```
- Reduce duplication among default methods.
- Not inherited, not part of the public contract.

### 5.1 Evolution of interface capabilities (summary)
| Java version | Interfaces gained |
|--------------|-------------------|
| ≤ 7 | Only constants + abstract methods |
| 8 | `default` and `static` methods (with bodies) |
| 9 | `private` and `private static` methods |
| 17 | `sealed` interfaces (§8) |

---

## 6. Functional Interfaces

A **functional interface** has **exactly one abstract method** (SAM — Single Abstract Method). It can be implemented concisely with a **lambda expression** or **method reference**.

```java
@FunctionalInterface
interface Calculator {
    int apply(int a, int b);    // the single abstract method
}

Calculator add = (a, b) -> a + b;          // lambda implements the interface
Calculator mul = (a, b) -> a * b;
System.out.println(add.apply(2, 3));       // 5
```

### 6.1 The `@FunctionalInterface` annotation
- Optional but recommended: the compiler **enforces** that the interface has exactly one abstract method (catches accidental additions).
- Default, static, and private methods **don't count** toward the single-abstract-method rule (only abstract methods do).

### 6.2 Built-in functional interfaces
Java provides a rich set in `java.util.function` — the basis of the Stream API and functional programming (full coverage in **Phase 1.8**):
| Interface | Abstract method | Meaning |
|-----------|-----------------|---------|
| `Function<T,R>` | `R apply(T)` | transform T → R |
| `Predicate<T>` | `boolean test(T)` | test a condition |
| `Consumer<T>` | `void accept(T)` | consume, no return |
| `Supplier<T>` | `T get()` | supply a value |
| `Runnable` | `void run()` | a runnable action |
| `Comparator<T>` | `int compare(T,T)` | ordering |

> Functional interfaces are *why* lambdas exist in Java — a lambda is just a concise implementation of a functional interface's single method. (Deep dive in Phase 1.8.)

---

## 7. Marker Interfaces

A **marker interface** has **no methods at all** — it exists purely to **tag/mark** a class with metadata that code or the JVM checks (usually via `instanceof`).

```java
public interface Serializable {}    // no methods — just a marker
```

### 7.1 Classic JDK examples
| Marker interface | Meaning when implemented |
|------------------|--------------------------|
| `Serializable` | Instances can be serialized (Phase 1.9) |
| `Cloneable` | `Object.clone()` is permitted (without it, `clone()` throws) |
| `RandomAccess` | A `List` supports fast (O(1)) index access (e.g., `ArrayList`) |

### 7.2 How they're used
```java
if (obj instanceof Serializable) {
    // safe to serialize
}
```
The presence of the type *is* the information.

### 7.3 Modern alternative: annotations
Today, **annotations** (e.g., `@Entity`, `@Deprecated`) often serve the tagging role markers used to. But marker interfaces still have one advantage: they create a **type** that the compiler/`instanceof` can use directly, enabling type-safe APIs. (Annotations covered in Phase 1.13.)

---

## 8. Sealed Interfaces (Java 17+)

A **sealed interface** restricts **which classes/interfaces may implement (or extend) it** — you explicitly list the permitted subtypes with `permits`. This gives you a **closed, known set** of implementations.

```java
public sealed interface Shape permits Circle, Square, Triangle {
    double area();
}

public final class Circle   implements Shape { public double area(){ return 0; } }
public final class Square   implements Shape { public double area(){ return 0; } }
public final class Triangle implements Shape { public double area(){ return 0; } }
```

### 8.1 Rules for permitted subtypes
Each permitted subtype must be declared one of:
- **`final`** — no further extension, or
- **`sealed`** — continues the controlled hierarchy with its own `permits`, or
- **`non-sealed`** — reopens it for unrestricted extension.

### 8.2 Why sealed interfaces are powerful
- **Exhaustive `switch`:** the compiler knows the *complete* set of subtypes, so a `switch` over them can be checked for exhaustiveness — no `default` branch needed when all cases are covered:
  ```java
  double describe(Shape s) {
      return switch (s) {              // exhaustive: compiler knows all 3 types
          case Circle c   -> c.area();
          case Square sq  -> sq.area();
          case Triangle t -> t.area();
      };
  }
  ```
- Models **algebraic data types / closed domains** safely (great with records + pattern matching).
- Documents intent and prevents unexpected implementations.

(Sealed classes/interfaces have a dedicated 1.3 note for deeper coverage; here we focus on the interface angle.)

---

## 9. Interface Segregation (the "I" in SOLID)

> **Interface Segregation Principle (ISP):** clients should **not be forced to depend on methods they don't use**. Prefer **many small, focused interfaces** over one large "fat" interface.

### 9.1 The problem (fat interface)
```java
interface Worker {
    void work();
    void eat();      // a robot worker doesn't eat!
    void sleep();    // ...or sleep
}
class Robot implements Worker {
    public void work()  { /* ok */ }
    public void eat()   { throw new UnsupportedOperationException(); } // forced, wrong
    public void sleep() { throw new UnsupportedOperationException(); }
}
```
Forcing `Robot` to implement `eat()`/`sleep()` is a design smell — those methods are irrelevant to it.

### 9.2 The fix (segregated interfaces)
```java
interface Workable { void work(); }
interface Eatable  { void eat(); }
interface Sleepable{ void sleep(); }

class Human implements Workable, Eatable, Sleepable { /* all three */ }
class Robot implements Workable { /* only what it needs */ }
```
Each class implements **only** the capabilities it actually has. (Full ISP treatment in Phase 14 — SOLID.)

---

## 10. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Trying to add instance fields to an interface | Interfaces hold only `public static final` constants; use an abstract class for state |
| Reducing visibility when implementing a method | Implementations must be `public` |
| Forgetting to resolve conflicting default methods | Override and pick via `Interface.super.method()` |
| Adding a second abstract method to a `@FunctionalInterface` | Keep exactly one; defaults/statics don't count |
| Calling a static interface method on an instance | Call it on the interface name |
| Fat interfaces forcing irrelevant methods | Segregate into small, role-based interfaces (ISP) |
| Using `default` methods to simulate state | Not possible/appropriate — no instance fields |
| Implementing a sealed interface from a non-permitted class | Only `permits`-listed (or appropriately sealed) types may implement |

---

## 11. Connection to Backend / Spring (Why This Matters Later)

- **"Program to interfaces"** is the core of Spring DI: services/repositories are defined as interfaces; Spring injects implementations → loose coupling and easy swapping (Phase 5).
- **Testing/mocking (Phase 6):** mocks implement the interface and are substituted in — only possible because dependencies are interface-typed.
- **Spring Data repositories** *are* interfaces — you declare the contract and Spring generates the implementation at runtime.
- **Functional interfaces + lambdas** power Stream pipelines, `Comparator`s, callbacks, and reactive code (Phase 1.8, 16).
- **Default methods** let framework authors evolve SPIs without breaking your implementations.
- **Sealed interfaces + records + pattern matching** model closed domains (e.g., result/event types) safely — increasingly idiomatic in modern Java backends.
- **Interface segregation** keeps service contracts focused and APIs clean (Phase 7, 14).

---

## 12. Quick Self-Check Questions

1. What implicit modifiers do interface fields and abstract methods have? Can interfaces hold instance state?
2. How does a class achieve "multiple inheritance" via interfaces, and how is the diamond problem handled for default methods?
3. Why were default methods added in Java 8? Give a real JDK example.
4. What are static and private interface methods used for?
5. What defines a functional interface, and what does `@FunctionalInterface` enforce?
6. What is a marker interface? Name two from the JDK and the modern alternative.
7. What does a sealed interface restrict, and what must permitted subtypes be declared?
8. Why do sealed interfaces enable exhaustive `switch` without a `default`?
9. State the Interface Segregation Principle and give an example of violating and fixing it.

---

## 13. Key Terms Glossary

- **Interface:** a contract declaring method signatures (capability/role) a type provides.
- **`implements` / `extends` (interfaces):** a class implements interfaces; an interface extends other interfaces.
- **Constant:** an interface field — implicitly `public static final`.
- **Default method:** an interface method with a body (`default`); inheritable, overridable (Java 8+).
- **Static method (interface):** body method called on the interface itself (Java 8+).
- **Private method (interface):** helper for default/static methods (Java 9+).
- **Functional interface:** exactly one abstract method (SAM); implementable by a lambda.
- **`@FunctionalInterface`:** annotation enforcing the single-abstract-method rule.
- **Lambda / method reference:** concise implementations of a functional interface.
- **Marker interface:** an empty interface used as a type tag (e.g., `Serializable`).
- **Sealed interface:** restricts permitted implementers via `permits` (Java 17+).
- **`permits` / `final` / `sealed` / `non-sealed`:** controls of a sealed hierarchy.
- **Interface Segregation Principle (ISP):** prefer small, focused interfaces.
- **`Interface.super.method()`:** syntax to call a specific interface's default method.

---

*Previous topic: **Abstraction**.*
*Next topic in roadmap: **Sealed Classes and Interfaces (Java 17) — deeper dive**.*
