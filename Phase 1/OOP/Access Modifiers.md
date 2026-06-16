**Access modifiers** control **who can see and use** a class member (field, method, constructor) or a type. They are Java's primary tool for **[[04-Encapsulation]]**, hiding internal details and exposing only a deliberate, controlled surface.

Java has **four** access levels, from most restrictive to most open:
```
private  <  default (package-private)  <  protected  <  public
 (most restrictive)                                   (most open)
```

> There is no `default` *keyword* - "default" / "package-private" means you write **no modifier at all**.

## 1. The Four Access Levels
Java provides **four types of access modifiers** to control the visibility and scope of classes, constructors, variables, and methods.

| Modifier | Keyword | Visible to... |
|----------|---------|---------------|
| **Private** | `private` | Only within the **same class** |
| **Default** (package-private) | *(none)* | Within the **same package** |
| **Protected** | `protected` | Same package **+ subclasses** (even in other packages) |
| **Public** | `public` | **Everywhere** |

### 1.1 The master visibility table
This single table answers almost every access question:

| Access from... | `private` | default | `protected` | `public` |
|----------------|:---------:|:-------:|:-----------:|:--------:|
| Same class | ✅ | ✅ | ✅ | ✅ |
| Same package (non-subclass) | ❌ | ✅ | ✅ | ✅ |
| Subclass in **same** package | ❌ | ✅ | ✅ | ✅ |
| Subclass in **different** package | ❌ | ❌ | ✅* | ✅ |
| Non-subclass in **different** package | ❌ | ❌ | ❌ | ✅ |

> \* `protected` access from a subclass in another package has a subtle restriction.

## 2. Each Modifier in Detail

### 2.1 `private` - class-only
The most restrictive. Accessible **only inside the declaring class**. The cornerstone of encapsulation.

```java title:bank_acc.java
public class BankAccount {
    private double balance;          // hidden from everyone outside

    public void deposit(double amt) { // controlled public access
        if (amt > 0) balance += amt;
    }
    public double getBalance() { return balance; }
}
```
- Fields should almost always be `private` - expose them only through methods (getters/setters) so you control validation and invariants.
- Even other instances *of the same class* can access each other's privates (it's class-level, not object-level):
  ```java
  public boolean richerThan(BankAccount other) {
      return this.balance > other.balance;  // OK: same class
  }
  ```

### 2.2 Default (package-private) - no keyword
When you write **no modifier**, the member is visible **within the same package only**.

```java title:helper.java
class Helper {           // package-private CLASS
    int value;           // package-private field
    void assist() {}     // package-private method
}
```
- Useful for classes/members meant to collaborate *within a module/package* but not be part of the public API.
- A great default for internal implementation classes you don't want leaking out.

### 2.3 `protected` - package + subclasses
Visible to:
1. The same class,
2. All classes in the **same package**, and
3. **Subclasses in any package** (this is the special power).

```java
public class Shape {
    protected String name;                 // subclasses can use directly
    protected void render() { /* ... */ }
}

// in a DIFFERENT package:
public class Circle extends Shape {
    void draw() {
        name = "circle";   // OK: inherited protected member
        render();          // OK
    }
}
```
- Designed for **[[05-Inheritance]]** - members a subclass legitimately needs, but the general public shouldn't touch.

### 2.4 `public` - everywhere
Visible to **all code** that can reference the class. This is your **public API contract**.

```java title:string_utils.java
public class StringUtils {
    public static boolean isEmpty(String s) {  // callable from anywhere
        return s == null || s.isEmpty();
    }
}
```
- Once public, changing or removing it can **break callers** → treat public members as commitments.

## 3. Access Levels Across Packages (the tricky parts)
Packages are Java's namespacing/organization mechanism. Access rules interact with packages in ways that trip people up.

### 3.1 Why packages matter
```
com.app.model      -> User.java, Order.java
com.app.service    -> UserService.java
com.app.internal   -> DbHelper.java   (package-private classes hide here)
```
- `default` and `protected` both grant **same-package** access - so package structure is itself an access-control tool.
- Code in a *different* package can only see `public` (and `protected` via inheritance) members.

### 3.2 The `protected` cross-package subtlety
A subclass in a **different** package can access a `protected` member **only through references of its own type (or a subtype)** - not through a reference of the parent type.

```java title:package.java
// package a;
public class Parent {
    protected int x;
}

// package b;
public class Child extends Parent {
    void demo(Child c, Parent p) {
        this.x = 1;   // OK
        c.x = 2;      // OK   -> reference is Child (own type)
        // p.x = 3;   // COMPILE ERROR -> p is Parent type, different package
    }
}
```
> Rationale: `protected` is for a subclass to manage **its own** inherited state, not to peek at arbitrary `Parent` instances across package boundaries.

### 3.3 Top-level type access
Only **two** modifiers are legal on **top-level** (non-nested) classes/interfaces:
- `public` - usable from any package.
- *(none)* default/package-private - usable only within its package.

```java
public class Api {}     // visible everywhere
class Internal {}       // visible only in this package
// private class Foo {} // ERROR: private/protected not allowed on top-level types
```
`private` and `protected` **are** allowed on **nested** (member) classes, though.

### 3.4 Interaction with the Java Module System (JPMS, awareness)
Since Java 9, **modules** (`module-info.java`) add another layer *on top of* access modifiers: a `public` type is only accessible to other modules if its package is **`exports`-ed**. So "public" no longer automatically means "visible to the whole world." (Full coverage in [[01-Java-Modules-JPMS]].)

## 4. Access Modifiers on Different Elements

| Element | private | default | protected | public |
|---------|:-------:|:-------:|:---------:|:------:|
| **Top-level class/interface** | ❌ | ✅ | ❌ | ✅ |
| **Nested class** | ✅ | ✅ | ✅ | ✅ |
| **Field** | ✅ | ✅ | ✅ | ✅ |
| **Method** | ✅ | ✅ | ✅ | ✅ |
| **Constructor** | ✅ | ✅ | ✅ | ✅ |

### 4.1 Private constructors (useful patterns)
```java
public class MathUtils {
    private MathUtils() {}      // prevent instantiation (utility class)
}

public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}      // force access via getInstance()
    public static Singleton getInstance() { return INSTANCE; }
}
```
- Private constructors enforce **utility classes**, **singletons**, and **static factory method** patterns (Phase 14 design patterns).

### 4.2 Overriding and access levels
When overriding a method, you **cannot reduce** its visibility (you may widen it):
```java
class A { protected void f() {} }
class B extends A {
    @Override public void f() {}      // OK: widened protected -> public
    // @Override private void f() {}  // ERROR: cannot narrow visibility
}
```
> Reason: narrowing would violate the **Liskov Substitution Principle** — a `B` must be usable everywhere an `A` is (Phase 14 SOLID).

## 5. Best Practices — Principle of Least Privilege

> **Principle of Least Privilege:** give every element the **narrowest** access that still lets it do its job. Start as restrictive as possible and widen only when there's a real need.

### 5.1 Concrete guidelines
| Guideline | Why |
|-----------|-----|
| Make **fields `private`** (almost always) | Protect invariants; control via methods |
| Expose state through **methods**, not public fields | Allows validation, lazy computation, future change |
| Keep classes/members **package-private** unless they're part of the API | Smaller public surface = easier to evolve |
| Use `protected` **only** for genuine inheritance hooks | Avoid leaking internals to subclasses needlessly |
| Make members **`public` deliberately** | Public = a contract you must maintain |
| Prefer **`private` constructors** for utility/singleton classes | Prevent misuse |
| Make immutable fields **`private final`** | Safety + thread-safety (Phase 1.10) |

### 5.2 Why least privilege matters
- **Encapsulation:** consumers depend only on the stable public API, not fragile internals.
- **Maintainability:** you can refactor private/package-private code freely without breaking callers.
- **Security:** less exposed surface = fewer ways to misuse or attack (Phase 15).
- **Reduced coupling:** narrower access prevents accidental tight dependencies.

### 5.3 The classic encapsulation pattern
```java
public class Temperature {
    private double celsius;                       // hidden state

    public double getCelsius() { return celsius; }

    public void setCelsius(double c) {
        if (c < -273.15)                          // validation lives here
            throw new IllegalArgumentException("Below absolute zero");
        this.celsius = c;
    }

    public double getFahrenheit() {               // derived, no stored field
        return celsius * 9 / 5 + 32;
    }
}
```
Because `celsius` is `private`, you could later change the internal representation (e.g., store Kelvin) **without breaking any caller**.

## 6. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Making fields `public` for convenience | Use `private` + getters/setters (or a record for pure data) |
| Using `protected` "just in case" | Use `private`/package-private until inheritance truly needs it |
| Trying `private`/`protected` on a top-level class | Only `public` or default allowed at top level |
| Narrowing visibility when overriding | Keep same or widen (LSP) |
| Assuming `public` = globally visible under modules | Must also `exports` the package (JPMS) |
| Overusing `public` and creating a huge API surface | Expose the minimum; everything public is a maintenance commitment |

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Encapsulation of entities/DTOs:** keep fields `private`, expose via accessors — JPA, Jackson, and validation all build on this.
- **Layered architecture (Phase 14):** package-private classes keep service/repository internals from leaking across layers; only interfaces/DTOs go `public`.
- **Spring beans:** Spring can inject into `private` fields via reflection, but **constructor injection** (public/package constructor) is preferred — access design shapes testability.
- **API design (Phase 7):** your `public` methods are the contract; minimizing them eases versioning and backward compatibility.
- **Security (Phase 15):** least privilege at the code level mirrors least privilege in auth/permissions — same mindset.
- **Modular monoliths / modules:** package-private + JPMS `exports` enforce real module boundaries.

## 8. Quick Self-Check Questions

1. List the four access levels from most to least restrictive. Which has no keyword?
2. From a class in a *different package*, what can you access if a member is `protected`? What about `default`?
3. Can two instances of the same class access each other's `private` fields? Why?
4. What's the special restriction on `protected` access from a subclass in another package?
5. Which access modifiers are legal on a **top-level** class?
6. When overriding, can you reduce a method's visibility? Why or why not?
7. State the principle of least privilege and give two reasons it matters.
8. Why should fields almost always be `private`? Give a concrete benefit.
9. How does the Java Module System change the meaning of `public`?

## 9. Key Terms Glossary

- **Access modifier:** keyword controlling visibility of a type or member.
- **`private`:** accessible only within the declaring class.
- **Default / package-private:** no keyword; accessible within the same package.
- **`protected`:** accessible in the same package and to subclasses (any package).
- **`public`:** accessible everywhere (subject to module `exports`).
- **Encapsulation:** hiding internal details behind a controlled interface.
- **Principle of least privilege:** grant the narrowest access necessary.
- **Package:** Java namespace/grouping that also defines an access boundary.
- **Top-level type:** a class/interface not nested inside another.
- **Liskov Substitution Principle:** subtypes must be usable wherever the supertype is (hence no narrowing on override).
- **JPMS / `exports`:** module system; a package must be exported for its `public` types to be visible across modules.
- **Static factory / singleton:** patterns often enforced via `private` constructors.