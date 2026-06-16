# Polymorphism

> **Phase 1 — Java Language Mastery → 1.3 Object-Oriented Programming**
> Goal: Master polymorphism — compile-time (overloading) vs runtime (overriding + dynamic dispatch), upcasting/downcasting, how virtual method invocation works under the hood, and covariant return types.

---

## 0. The Big Picture

**Polymorphism** (Greek: "many forms") lets a **single interface/reference type represent many underlying types**, so the *same call* can behave differently depending on the actual object. It's the pillar of OOP that makes code **extensible** and **decoupled**.

```java
Animal a = new Dog();   // an Animal reference holding a Dog
a.sound();              // behaves as a Dog -> "woof"
a = new Cat();          // same reference, now a Cat
a.sound();              // behaves as a Cat -> "meow"
```

Two kinds in Java:
| Type | Mechanism | Bound at | Also called |
|------|-----------|----------|-------------|
| **Compile-time polymorphism** | Method **overloading** | Compile time | Static / early binding |
| **Runtime polymorphism** | Method **overriding** + dynamic dispatch | Runtime | Dynamic / late binding |

> The single most important idea: with overriding, **the object's runtime type — not the reference type — decides which method runs.**

---

## 1. Compile-Time Polymorphism (Method Overloading)

**Overloading** = multiple methods with the **same name** but **different parameter lists** in the same class. The compiler decides which to call **at compile time**, based on the **static (declared) types** of the arguments. (Full mechanics in the Methods note — recap here.)

```java
class Printer {
    void print(int x)    { System.out.println("int: " + x); }
    void print(double x) { System.out.println("double: " + x); }
    void print(String x) { System.out.println("String: " + x); }
}

Printer p = new Printer();
p.print(10);      // "int: 10"     -> chosen at compile time
p.print(3.14);    // "double: 3.14"
p.print("hi");    // "String: hi"
```

### 1.1 Key properties
- Resolved by the **compiler** using argument types (number, types, order).
- Return type and parameter *names* don't participate.
- Resolution preference: **exact match → widening → autoboxing → varargs**.
- This is "static binding" because the target method is fixed in the bytecode at compile time.

> Overloading is "polymorphism" in the loose sense (one name, many forms), but it has **nothing to do with inheritance** — it's purely compile-time name resolution.

---

## 2. Runtime Polymorphism (Method Overriding + Dynamic Dispatch)

**Overriding** = a subclass provides its own implementation of an inherited method (same signature). The JVM selects the method **at runtime** based on the **actual object type** — this selection is **dynamic dispatch** (a.k.a. dynamic method dispatch / late binding).

```java
class Animal { String sound() { return "..."; } }
class Dog extends Animal { @Override String sound() { return "woof"; } }
class Cat extends Animal { @Override String sound() { return "meow"; } }

List<Animal> zoo = List.of(new Dog(), new Cat(), new Animal());
for (Animal a : zoo) {
    System.out.println(a.sound());   // woof / meow / ...  -> runtime type decides
}
```

### 2.1 Why this is so powerful
You can write code against the **superclass/interface type** and it automatically works with **any subtype** — including ones written *after* your code:
```java
void describe(Animal a) {       // works for Dog, Cat, and any future Animal
    System.out.println(a.sound());
}
```
This is the **Open/Closed Principle** (Phase 14): open for extension (new `Animal` subtypes), closed for modification (`describe` never changes).

### 2.2 Reference type vs runtime type (the crux)
```java
Animal a = new Dog();
```
- **Reference type** = `Animal` (the declared type). Controls **what you can call** (compile-time: only `Animal`'s members are accessible).
- **Runtime type** = `Dog` (the actual object). Controls **which overridden method executes** (runtime: `Dog.sound()`).

```java
Animal a = new Dog();
a.sound();   // OK -> "woof" (Dog's override runs)
// a.bark(); // COMPILE ERROR -> bark() not in Animal's type, even though object is a Dog
```

### 2.3 What is and isn't polymorphic
| Member | Polymorphic (runtime-resolved)? | Resolved by |
|--------|:-------------------------------:|-------------|
| **Instance methods (overridden)** | ✅ Yes | Runtime type |
| **Fields** | ❌ No | Reference type (compile time) |
| **`static` methods** | ❌ No (hidden, not overridden) | Reference type |
| **`private` / `final` methods** | ❌ No | Statically bound |

```java
class Parent { int x = 1; static String who(){ return "P"; } }
class Child extends Parent { int x = 2; static String who(){ return "C"; } }

Parent p = new Child();
System.out.println(p.x);        // 1  -> FIELDS use reference type (not polymorphic)
System.out.println(p.who());    // P  -> static methods use reference type (hiding)
```
> This is yet another reason to keep fields `private` and access state through (polymorphic) methods.

---

## 3. Upcasting and Downcasting

Casting changes the **reference type** you use to view an object — the object itself never changes.

### 3.1 Upcasting (subtype → supertype) — implicit & always safe
```java
Dog d = new Dog();
Animal a = d;              // upcast: implicit, no cast needed, always safe
Animal a2 = new Dog();     // common form
```
- Treats a `Dog` as an `Animal`. Always valid because a `Dog` **IS-A** `Animal`.
- You **lose access** to `Dog`-specific members through the `Animal` reference, but overridden methods still dispatch to `Dog`'s versions.
- This is what enables writing general code against base types.

### 3.2 Downcasting (supertype → subtype) — explicit & risky
```java
Animal a = new Dog();
Dog d = (Dog) a;          // downcast: explicit, you assert "this really is a Dog"
d.bark();                 // now Dog-specific members are accessible
```
- Needed to access subtype-specific members.
- **Risky:** if the object isn't actually that subtype, you get a **`ClassCastException`** at runtime:
  ```java
  Animal a = new Cat();
  Dog d = (Dog) a;        // compiles, but throws ClassCastException at runtime!
  ```

### 3.3 Guard downcasts with `instanceof`
```java
Animal a = getAnimal();
if (a instanceof Dog) {
    Dog d = (Dog) a;
    d.bark();
}
```
**Pattern matching for `instanceof`** (Java 16+) makes this cleaner — test and bind in one step:
```java
if (a instanceof Dog d) {   // checks AND casts to 'd' if true
    d.bark();
}
```
(Pattern matching is expanded in the switch/pattern-matching notes.)

### 3.4 Summary
| | Upcast | Downcast |
|---|--------|----------|
| Direction | Child → Parent | Parent → Child |
| Syntax | Implicit (`Animal a = dog;`) | Explicit (`(Dog) a`) |
| Safety | Always safe | Can throw `ClassCastException` |
| Guard | Not needed | Use `instanceof` first |

---

## 4. Virtual Method Invocation (How Dynamic Dispatch Works)

In Java, **instance methods are "virtual" by default** — calls are resolved by the **runtime type** via dynamic dispatch. (In C++ you must mark methods `virtual`; in Java it's automatic unless `final`/`static`/`private`.)

### 4.1 The mechanism: vtables
Conceptually, the JVM uses a **virtual method table (vtable)** per class — an array of pointers to the actual method implementations. Each object's header has a **klass pointer** (recall the Classes & Objects note's object header) leading to its class's vtable.

```
Animal a = new Dog();
a.sound();

1. Look at the object 'a' points to -> it's a Dog (via klass pointer in header).
2. Find Dog's vtable.
3. Look up the slot for sound().
4. Dog overrides sound(), so that slot points to Dog.sound().
5. Call Dog.sound() -> "woof".
```
If `Dog` did **not** override `sound()`, its vtable slot would point to the inherited `Animal.sound()`.

### 4.2 Bytecode (awareness)
Java compiles these calls to specific invoke instructions:
| Instruction | Used for |
|-------------|----------|
| `invokevirtual` | Normal (overridable) instance method calls → **dynamic dispatch** |
| `invokeinterface` | Calls through an interface reference |
| `invokestatic` | `static` methods (no dispatch) |
| `invokespecial` | Constructors, `private` methods, `super.method()` (no dispatch) |
| `invokedynamic` | Lambdas, dynamic languages |

> `super.method()` uses `invokespecial` precisely **to bypass dynamic dispatch** and call the parent's version directly — that's why `super` always reaches the parent and never re-dispatches to the child.

### 4.3 Performance note
Dynamic dispatch has a tiny cost (an indirection). The **JIT compiler** (Phase 1.11) often eliminates it via **devirtualization/inlining** when it can prove only one implementation is used (monomorphic call sites) — so in practice virtual calls are usually very fast.

---

## 5. Covariant Return Types

A **covariant return type** lets an overriding method return a **subtype** of the type returned by the overridden parent method (allowed since Java 5).

```java
class Animal {
    Animal reproduce() { return new Animal(); }
}
class Dog extends Animal {
    @Override
    Dog reproduce() { return new Dog(); }   // returns Dog (a subtype of Animal): OK
}
```

### 5.1 Why it's useful
Callers working with a `Dog` get a `Dog` back without casting:
```java
Dog d = new Dog();
Dog puppy = d.reproduce();   // no cast needed thanks to covariant return
```
Without covariance, the override would have to return `Animal`, forcing callers to downcast.

### 5.2 Rules
- The return type must be a **subtype** of the parent method's return type (and the method must otherwise be a valid override — same name & parameters).
- Works only for **reference types** (not primitives).
- A classic real example: `Object.clone()` returns `Object`, but an override can return the actual class type:
  ```java
  @Override
  public Dog clone() { return (Dog) super.clone(); }   // covariant clone
  ```
- The builder pattern and fluent APIs often use covariant returns so subclass builders return the subclass type.

---

## 6. Putting It All Together

```java
interface Shape { double area(); }              // polymorphic contract

class Circle implements Shape {
    private final double r;
    Circle(double r) { this.r = r; }
    @Override public double area() { return Math.PI * r * r; }
}
class Square implements Shape {
    private final double s;
    Square(double s) { this.s = s; }
    @Override public double area() { return s * s; }
}

// Code written ONCE against Shape works for ALL current & future shapes:
double totalArea(List<Shape> shapes) {
    double sum = 0;
    for (Shape sh : shapes) sum += sh.area();   // dynamic dispatch per element
    return sum;
}
```
Add a new `Triangle implements Shape` later — `totalArea` needs **zero changes**. That's the practical payoff of runtime polymorphism.

---

## 7. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Expecting fields to be polymorphic | Fields use reference type; use methods for polymorphic behavior |
| Expecting `static` methods to override | They **hide**; resolved by reference type |
| Unchecked downcast → `ClassCastException` | Guard with `instanceof` (ideally pattern matching) |
| Calling subtype methods through a base reference | Not visible at compile time; downcast or redesign |
| Forgetting `@Override` (silent non-override) | Always annotate |
| Calling overridable methods in a constructor | Subclass override runs before subclass fields init |
| Overusing downcasting / `instanceof` chains | Often a design smell — use polymorphism or the Visitor/Strategy pattern |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Program to interfaces, not implementations:** Spring DI injects an interface type; the concrete bean is the runtime type → polymorphism is the engine of loose coupling (Phase 5, 14).
- **Strategy pattern:** swap behavior by injecting different implementations of the same interface — pure runtime polymorphism.
- **Collections of a base type** (`List<PaymentProcessor>`) let you iterate and dispatch dynamically — common in plugin/handler architectures.
- **Open/Closed Principle:** add new implementations without touching existing code (Phase 14 SOLID).
- **Testing/mocking (Phase 6):** mocks are subtypes substituted via upcasting — only possible because of polymorphism.
- **Jackson/JPA polymorphic mapping** (`@JsonTypeInfo`, inheritance strategies) rely on these type relationships.

---

## 9. Quick Self-Check Questions

1. What are the two kinds of polymorphism in Java, and when is each resolved?
2. With `Animal a = new Dog();`, which type controls *what you can call* and which controls *which method runs*?
3. Are fields and static methods polymorphic? What determines which version is used?
4. What's the difference between upcasting and downcasting? Which can fail and how?
5. How does `instanceof` (and pattern matching) make downcasting safe?
6. Explain dynamic dispatch using the vtable model.
7. Why does `super.method()` not re-dispatch to the subclass? (Hint: which invoke instruction.)
8. What is a covariant return type and why is it useful? Give an example.
9. How does runtime polymorphism enable the Open/Closed Principle?

---

## 10. Key Terms Glossary

- **Polymorphism:** one interface/reference representing many actual types.
- **Compile-time polymorphism:** overloading; resolved by the compiler.
- **Runtime polymorphism:** overriding; resolved at runtime by object type.
- **Dynamic dispatch / late binding:** runtime selection of the overriding method.
- **Static binding / early binding:** compile-time method selection.
- **Reference type:** the declared type of a variable (controls accessible members).
- **Runtime type:** the actual class of the object (controls overridden method choice).
- **Upcasting:** treating a subtype as a supertype (implicit, safe).
- **Downcasting:** treating a supertype reference as a subtype (explicit, risky).
- **`ClassCastException`:** thrown by an invalid downcast.
- **`instanceof` (+ pattern matching):** runtime type test (and bind).
- **Virtual method:** an instance method resolved by dynamic dispatch (Java default).
- **vtable:** per-class table of method pointers used for dispatch.
- **invokevirtual / invokespecial / invokeinterface / invokestatic / invokedynamic:** JVM call instructions.
- **Covariant return type:** override returning a subtype of the parent's return type.

---

*Previous topic: **Inheritance**.*
*Next topic in roadmap: **Abstraction (abstract classes & methods, abstract class vs interface)**.*
