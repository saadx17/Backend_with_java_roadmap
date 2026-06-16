# Inheritance

> **Phase 1 — Java Language Mastery → 1.3 Object-Oriented Programming**
> Goal: Master inheritance — `extends`, method overriding rules, all uses of `super`, why constructors aren't inherited, the single-inheritance limitation, the IS-A relationship, the fragile base class problem, and (critically) **when NOT to use inheritance**.

---

## 0. The Big Picture

**Inheritance** lets one class (the **subclass**/child/derived) acquire the fields and methods of another (the **superclass**/parent/base), then extend or specialize them. It models an **IS-A** relationship and enables **code reuse** and **polymorphism**.

```java
class Animal {                    // superclass / parent / base
    void breathe() { System.out.println("breathing"); }
}

class Dog extends Animal {        // subclass / child / derived
    void bark() { System.out.println("woof"); }
}

Dog d = new Dog();
d.breathe();   // inherited from Animal
d.bark();      // defined in Dog
```

> ⚠️ Inheritance is powerful but **frequently overused**. The most important sections here are the **fragile base class problem** (§7) and **when NOT to use it** (§8) — favor **composition** by default.

---

## 1. The `extends` Keyword

`extends` establishes the inheritance link. A subclass inherits the **accessible** (non-private) members of its superclass.

```java
class Vehicle {
    protected String brand;
    protected int speed;
    void accelerate() { speed += 10; }
}

class Car extends Vehicle {       // Car IS-A Vehicle
    int numDoors;
    void honk() { System.out.println("beep"); }
}
```

### 1.1 What is and isn't inherited
| Member | Inherited? |
|--------|-----------|
| `public` / `protected` fields & methods | ✅ Yes |
| Default (package-private) members | ✅ Only if subclass is in the **same package** |
| `private` members | ❌ No (exist in the object but not directly accessible) |
| **Constructors** | ❌ **No** (see §4) |
| `static` members | ✅ Inherited but **hidden, not overridden** (see §2.5) |

### 1.2 Every class extends `Object`
If you don't write `extends`, your class implicitly extends `java.lang.Object` — the root of all Java classes. That's why every object has `toString()`, `equals()`, `hashCode()`, `getClass()`, etc. (their own dedicated note).

```java
class Foo {}            // == class Foo extends Object {}
```

---

## 2. Method Overriding

**Overriding** = a subclass provides its **own implementation** of a method already defined in the superclass, with the **same signature**. This is **runtime (dynamic) polymorphism** — the actual method called depends on the object's *runtime type*, not the reference type.

```java
class Animal {
    String sound() { return "..."; }
}
class Cat extends Animal {
    @Override
    String sound() { return "meow"; }   // overrides Animal.sound()
}

Animal a = new Cat();   // reference type Animal, runtime type Cat
System.out.println(a.sound());   // "meow" -> runtime type decides (dynamic dispatch)
```

### 2.1 The rules of overriding
For a method to legally override a parent method:
1. **Same name** and **same parameter list** (same signature). Different params = overloading, not overriding.
2. **Same or covariant return type** — return type may be a **subtype** of the parent's:
   ```java
   class Animal { Animal reproduce() { return new Animal(); } }
   class Dog extends Animal {
       @Override Dog reproduce() { return new Dog(); }   // covariant: OK
   }
   ```
3. **Cannot reduce visibility** — may keep the same or widen it (`protected` → `public` OK; `public` → `private` ERROR). (Ties to LSP — Phase 14.)
4. **Cannot throw broader checked exceptions** — may throw the same, narrower, or fewer checked exceptions (unchecked exceptions are unrestricted).
5. Only **inherited, overridable** methods can be overridden — **not** `static`, `final`, or `private` methods (see below).

### 2.2 The `@Override` annotation
Always use `@Override`. It's optional but tells the compiler "I intend to override" — if you typo the signature, the compiler **catches the mistake**:
```java
class Cat extends Animal {
    @Override
    String suond() { return "meow"; }   // COMPILE ERROR: doesn't override anything (typo!)
}
```
Without `@Override`, that typo would silently create a *new* method and the bug would be invisible.

### 2.3 What CAN'T be overridden
| Modifier | Why not |
|----------|---------|
| `final` methods | Explicitly sealed against overriding |
| `static` methods | Belong to the class; they're **hidden**, not overridden (§2.5) |
| `private` methods | Not visible to subclasses; a same-name method is just a new method |
| Constructors | Not inherited at all |

### 2.4 Overriding vs Overloading (don't confuse)
| | Overriding | Overloading |
|---|-----------|-------------|
| Signature | **Same** | **Different** params |
| Classes | Across parent/child | Same class (or inherited) |
| Bound at | **Runtime** (dynamic dispatch) | **Compile time** |
| Polymorphism | Dynamic | Static |
| `@Override` applies? | Yes | No |

### 2.5 Method hiding (static methods)
A subclass `static` method with the same signature **hides** the parent's. The version called depends on the **reference type** (compile time), not the object — the opposite of overriding:
```java
class Parent { static String who() { return "Parent"; } }
class Child extends Parent { static String who() { return "Child"; } }

Parent p = new Child();
System.out.println(p.who());        // "Parent" -> reference type! (hiding, not overriding)
System.out.println(Child.who());    // "Child"
```
> Field access is also resolved by **reference type**, not runtime type — fields are never polymorphic. Another reason to keep fields `private` and use methods.

---

## 3. The `super` Keyword (All Uses)

`super` refers to the **parent-class portion** of the current object. Three distinct uses:

| # | Use | Example |
|---|-----|---------|
| 1 | **Call the parent constructor** | `super(args);` (first line of constructor) |
| 2 | **Call the parent's (overridden) method** | `super.method();` |
| 3 | **Access a parent field** (when shadowed) | `super.field` |

```java
class Animal {
    String name = "animal";
    Animal(String name) { this.name = name; }
    void describe() { System.out.println("I am " + name); }
}

class Dog extends Animal {
    String name = "dog";                 // shadows Animal.name (avoid this in practice!)

    Dog() {
        super("Rex");                    // (1) call parent constructor — MUST be first
    }

    @Override
    void describe() {
        super.describe();                // (2) reuse parent behavior, then add to it
        System.out.println("name field = " + name + ", super.name = " + super.name); // (3)
    }
}
```

- `super(...)` **must be the first statement** in a constructor (and can't coexist with `this(...)`).
- `super.method()` is the standard way to **extend** rather than fully replace parent behavior.
- `super.field` is needed only when a subclass field **shadows** a parent field — which you should generally avoid.

---

## 4. Constructor Inheritance (or Lack Thereof)

> **Constructors are NOT inherited.** A subclass does not inherit its parent's constructors.

### 4.1 What actually happens
Every subclass constructor **must call a superclass constructor** (so the parent part of the object is initialized). If you don't call `super(...)` explicitly, the compiler inserts an implicit **`super()`** (the no-arg parent constructor) as the first statement.

```java
class Animal {
    Animal() { System.out.println("Animal()"); }
}
class Dog extends Animal {
    Dog() {
        // implicit super(); inserted here
        System.out.println("Dog()");
    }
}
new Dog();   // prints: Animal()  then  Dog()  -> parent constructed first
```

### 4.2 The classic compile error
If the parent has **no no-arg constructor** (because you declared a parameterized one and no default), the child **must** call `super(args)` explicitly — otherwise it won't compile:
```java
class Animal {
    Animal(String name) { }       // only a parameterized constructor exists
}
class Dog extends Animal {
    Dog() {                       // ERROR: implicit super() has no matching constructor
        // must write: super("Rex");
    }
}
```

### 4.3 Construction order (recap from Classes & Objects note)
Construction always runs **top-down**: `Object` → ... → parent → child. The parent is fully initialized before the child constructor body runs.
> This is why a **parent constructor must never call an overridable method** — the child's override would run before the child's fields are initialized (a classic bug).

---

## 5. Single Inheritance Limitation

> **Java supports single (class) inheritance only:** a class can `extends` **exactly one** direct superclass.

```java
class A {}
class B {}
// class C extends A, B {}   // ERROR: Java does NOT allow multiple class inheritance
```

### 5.1 Why? The Diamond Problem
With multiple class inheritance, if two parents define the same method, the child wouldn't know which to inherit (ambiguity). Java avoids this by allowing only one superclass.

### 5.2 How Java gets "multiple inheritance" — interfaces
A class can `extends` one class but `implements` **many interfaces** — gaining multiple *types* and (since Java 8) multiple sets of **default methods**:
```java
class Smartphone extends Device implements Camera, GPS, Phone { }
```
- Multiple inheritance of **type** and **behavior (default methods)**: ✅ via interfaces.
- Multiple inheritance of **state (fields)**: ❌ not allowed (interfaces can't hold instance state) — which is precisely what avoids the diamond ambiguity for state.
- If two interfaces provide conflicting default methods, the compiler forces you to **override and resolve** explicitly (`Camera.super.method()`).

(Interfaces get their own dedicated 1.3 note.)

---

## 6. The IS-A Relationship

Inheritance models **IS-A** (a subtype *is a kind of* the supertype). Use inheritance **only** when this is genuinely true.

```
Dog IS-A Animal        ✅   (a dog is a kind of animal)
Car IS-A Vehicle       ✅
Savings IS-A Account   ✅

Car IS-A Engine        ❌   (a car HAS-A engine — use composition!)
Stack IS-A ArrayList   ❌   (a stack is not a kind of list — exposes wrong operations)
```

### 6.1 IS-A vs HAS-A
| Relationship | Meaning | Mechanism |
|--------------|---------|-----------|
| **IS-A** | Subtype of | Inheritance (`extends` / `implements`) |
| **HAS-A** | Composed of / contains | Composition (a field referencing another object) |

```java
// HAS-A via composition (usually the better choice)
class Car {
    private Engine engine;   // Car HAS-A Engine
    private List<Wheel> wheels;
}
```

### 6.2 The Liskov Substitution Principle (preview)
A valid IS-A means: **anywhere the superclass is expected, the subclass must work correctly** without surprising behavior. If substituting the child breaks expectations, the inheritance is wrong (Phase 14 SOLID).

> Classic violation: `Square extends Rectangle`. A square *is-a* rectangle mathematically, but if `Rectangle` has independent `setWidth`/`setHeight`, a `Square` can't honor them without breaking the contract → LSP violation.

---

## 7. The Fragile Base Class Problem

> **Fragile base class problem:** changes to a superclass can **unexpectedly break subclasses**, even when the subclass code wasn't touched. Inheritance creates a tight, often hidden, coupling between parent and child.

### 7.1 The classic example (from *Effective Java*)
```java
// We want a HashSet that counts how many elements were ever added.
class CountingHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();          // looks correct...
        return super.addAll(c);
    }
    public int getAddCount() { return addCount; }
}

var s = new CountingHashSet<String>();
s.addAll(List.of("a", "b", "c"));
System.out.println(s.getAddCount());   // Expected 3... but prints 6!
```
**Why?** Internally, `HashSet.addAll()` calls `add()` for each element. So each element is counted **twice** (once in `addAll`, once in the `add` it triggers). The subclass broke because it depended on the **undocumented internal implementation** of the superclass. If a later JDK version changes whether `addAll` calls `add`, the behavior changes again — *fragile*.

### 7.2 Why it happens
- Subclasses depend on **implementation details** of the parent, not just its public contract.
- Self-use (a parent method calling its own overridable methods) is usually **undocumented**.
- A superclass author can change internals in a "safe" way that silently breaks subclasses.

### 7.3 The fix: composition + forwarding (the Decorator/Wrapper approach)
Instead of extending, **wrap** the object and forward calls — you depend only on the **public API**, not internals:
```java
class CountingSet<E> {
    private final Set<E> delegate;        // composition: HAS-A Set
    private int addCount = 0;

    CountingSet(Set<E> delegate) { this.delegate = delegate; }

    public boolean add(E e) { addCount++; return delegate.add(e); }
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return delegate.addAll(c);        // delegate's internal add() is NOT our add()
    }
    public int getAddCount() { return addCount; }
}
```
Now `addAll` counts correctly (3), and the wrapper is immune to the parent's internal changes.

---

## 8. When NOT to Use Inheritance

> **Default to composition. Reach for inheritance only when there's a true, stable IS-A relationship *and* the base class was designed for extension.**

### 8.1 Red flags — don't inherit when...
| Situation | Use instead |
|-----------|-------------|
| The relationship is **HAS-A**, not IS-A | Composition (a field) |
| You only want to **reuse code**, not model a subtype | Composition / delegation / utility methods |
| The superclass wasn't **designed and documented** for extension | Composition + forwarding (wrapper) |
| Substituting the subclass would **break expectations** (LSP) | Rethink the model |
| You'd inherit methods that **don't make sense** for the subtype | Composition (don't expose wrong API) |
| You need to **cross class hierarchies** | Interfaces |
| The behavior varies and should be **swappable at runtime** | **Strategy pattern** (composition) |

### 8.2 "Inheritance breaks encapsulation"
Inheritance is **white-box reuse**: the subclass sees and depends on the parent's internals. Composition is **black-box reuse**: you depend only on a public interface. Black-box is more robust → the industry guideline **"favor composition over inheritance"** (*Effective Java*, Item 18).

### 8.3 Design for inheritance or prohibit it
If a class **isn't** meant to be extended, make it **`final`** (or use sealed classes — Phase 1.3 sealed note) to prevent fragile subclassing. If it **is** meant to be extended, **document its self-use** (which methods call which overridable methods) so subclasses can extend safely.

### 8.4 Good uses of inheritance (when it IS right)
- Genuine, stable IS-A hierarchies (`IOException extends Exception`).
- Frameworks providing **template methods** / extension points designed for overriding.
- Implementing interfaces / abstract classes that define a clear contract.
- Sharing behavior across a small, controlled, well-documented hierarchy.

---

## 9. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Using inheritance for code reuse (not IS-A) | Use composition |
| Extending a class not designed for extension | Wrap it (composition + forwarding) |
| Forgetting `@Override` (silent typo bugs) | Always annotate overrides |
| Calling overridable methods from a constructor | Don't — child isn't initialized yet |
| Shadowing parent fields | Avoid; use `private` fields + accessors |
| Expecting `static` methods/fields to be polymorphic | They're resolved by reference type |
| Deep inheritance hierarchies (4+ levels) | Flatten; prefer composition/interfaces |
| Not making non-extensible classes `final` | Mark `final` (or `sealed`) |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Composition over inheritance** is the bedrock of Spring's **Dependency Injection** — you compose objects by injecting collaborators, not by extending them (Phase 5).
- **Interfaces + DI** (not class inheritance) drive testability and loose coupling (Phase 6 mocking, Phase 14 SOLID).
- **`@Override`** is everywhere when implementing Spring interfaces (`UserDetailsService`, `Filter`, etc.).
- **Strategy pattern** (composition) is the idiomatic way to make behavior pluggable in services.
- **Fragile base class** awareness prevents brittle abstract-class hierarchies in your domain model — DDD favors composition of value objects and aggregates (Phase 12).
- Frameworks use **template methods** (designed-for-extension inheritance) — e.g., `OncePerRequestFilter`, `AbstractAuthenticationProcessingFilter`.

---

## 11. Quick Self-Check Questions

1. What does `extends` do, and what is NOT inherited (name three)?
2. State the rules a method must satisfy to override another. Why use `@Override`?
3. What's the difference between overriding, overloading, and method hiding?
4. Are static methods and fields polymorphic? What determines which version is used?
5. List all three uses of `super`.
6. Why aren't constructors inherited, and what does the compiler insert if you omit `super(...)`?
7. Why does Java allow only single class inheritance, and how do you get "multiple inheritance"?
8. Explain IS-A vs HAS-A with an example of each.
9. Describe the fragile base class problem (and the `CountingHashSet` example). How does composition fix it?
10. Give four situations where you should NOT use inheritance.

---

## 12. Key Terms Glossary

- **Inheritance:** a subclass acquiring members of a superclass.
- **Superclass / subclass:** parent (base) / child (derived) class.
- **`extends`:** keyword establishing class inheritance.
- **Overriding:** subclass redefining an inherited method (same signature; runtime dispatch).
- **`@Override`:** annotation asserting an override; catches signature mistakes.
- **Covariant return type:** override returning a subtype of the parent's return type.
- **Method hiding:** subclass `static` method with same signature (resolved by reference type).
- **`super`:** reference to the parent-class portion (constructor/method/field).
- **Dynamic dispatch:** runtime selection of the overriding method based on object type.
- **IS-A / HAS-A:** subtype relationship (inheritance) vs containment (composition).
- **Composition:** building objects by holding references to other objects.
- **Single inheritance:** a class may extend only one class.
- **Diamond problem:** ambiguity from inheriting the same member via two paths.
- **Fragile base class problem:** superclass changes silently breaking subclasses.
- **Forwarding / wrapper:** delegating to a contained object (black-box reuse).
- **Liskov Substitution Principle:** subtypes must be substitutable for their supertype.
- **Template method:** a base method designed for subclasses to override at defined points.

---

*Previous topic: **Encapsulation**.*
*Next topic in roadmap: **Polymorphism (compile-time vs runtime, upcasting/downcasting, virtual method invocation, covariant returns)**.*
