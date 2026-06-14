# Inner Classes (Nested Classes)

> **Phase 1 — Java Language Mastery → 1.3 Object-Oriented Programming**
> Goal: Master Java's nested class types — member inner class, static nested class, local inner class, and anonymous inner class — and know exactly when to use which.

---

## 0. The Big Picture

A **nested class** is a class declared **inside another class**. Java uses nesting to **logically group classes that are only used in one place**, increase **encapsulation**, and create more **readable, maintainable** code.

```
class Outer {
    class Inner {}              // (1) Member inner class (non-static)
    static class Nested {}      // (2) Static nested class
    void method() {
        class Local {}          // (3) Local inner class (inside a method)
        Runnable r = new Runnable() {   // (4) Anonymous inner class
            public void run() {}
        };
    }
}
```

### 0.1 The four types at a glance
| Type | Static? | Where declared | Holds outer instance ref? |
|------|---------|----------------|---------------------------|
| **Member inner class** | No | Inside class body | ✅ Yes |
| **Static nested class** | Yes | Inside class body | ❌ No |
| **Local inner class** | No | Inside a method/block | ✅ Yes |
| **Anonymous inner class** | No | Inline expression | ✅ Yes |

> **Terminology note:** Strictly, "**inner class**" = a *non-static* nested class (it has an enclosing instance). "**Static nested class**" is technically not an "inner" class. People loosely say "inner classes" for all of them — this note covers all four.

---

## 1. Member Inner Class (Non-Static Nested)

A non-static class declared directly inside another class's body. It is **associated with an instance** of the outer class and can access **all** of the outer's members — **including `private` ones**.

```java
public class Bank {
    private String name = "MyBank";

    public class Account {                 // member inner class
        private double balance;
        public void show() {
            System.out.println(balance + " at " + name);  // accesses outer's private field!
        }
    }
}
```

### 1.1 Creating one (needs an outer instance)
Because it's tied to an outer instance, you need an `Outer` object first:
```java
Bank bank = new Bank();
Bank.Account acc = bank.new Account();    // unusual syntax: outerInstance.new Inner()
```

### 1.2 Key properties
- Has an **implicit reference to the enclosing instance** → can use `Outer.this` to refer to it:
  ```java
  public class Outer {
      int x = 10;
      class Inner {
          int x = 20;
          void print() {
              System.out.println(this.x);        // 20 (inner)
              System.out.println(Outer.this.x);  // 10 (outer)
          }
      }
  }
  ```
- Can access all outer members (including `private`).
- **Cannot** declare `static` members (except `static final` constants) — because it's tied to an instance. (Relaxed in Java 16+.)

### 1.3 ⚠️ The memory leak risk
Because a member inner class holds a hidden reference to its outer instance, **the outer object can't be garbage-collected while the inner object is alive**. If an inner instance outlives its outer (e.g., stored in a long-lived collection or passed to a listener), it **keeps the entire outer object in memory** → a subtle memory leak. **If the inner class doesn't need the outer instance, make it `static`** (see §2).

---

## 2. Static Nested Class

A nested class marked `static`. It is **NOT tied to an outer instance** — it behaves like a regular top-level class that just happens to live inside another for namespacing/grouping.

```java
public class Outer {
    private static int counter = 0;
    private int instanceField;

    public static class Builder {          // static nested class
        public Outer build() { return new Outer(); }
        // can access Outer's STATIC members (counter), but NOT instanceField
    }
}

Outer.Builder b = new Outer.Builder();     // no outer instance needed
```

### 2.1 Key properties
- **No implicit outer reference** → no memory-leak risk, more efficient.
- Can access only the outer's **static** members directly (not instance members).
- Created **without** an outer instance: `new Outer.Builder()`.
- This is the **most common and generally preferred** form of nested class.

### 2.2 Member inner vs static nested
| Aspect | Member inner (non-static) | Static nested |
|--------|---------------------------|---------------|
| Outer instance needed | ✅ Yes (`outer.new Inner()`) | ❌ No (`new Outer.Nested()`) |
| Holds outer reference | ✅ Yes (leak risk) | ❌ No |
| Access outer instance members | ✅ Yes | ❌ Only static |
| Memory overhead | Extra hidden reference | None |
| Typical use | Tightly bound to outer instance | Helper/builder grouped with outer |

> **Rule of thumb (*Effective Java* Item 24):** if a nested class doesn't need access to the enclosing **instance**, **make it `static`.** A non-static member class that doesn't use the outer instance just wastes memory and risks leaks.

### 2.3 Common uses
- **Builder pattern** (recall the Encapsulation note — `User.Builder`).
- Helper/data-holder classes scoped to the outer class (e.g., `Map.Entry` is a nested type).

---

## 3. Local Inner Class

A class declared **inside a method, constructor, or block**. Its scope is limited to that block — it's invisible outside.

```java
public class Calculator {
    public Runnable createTask(int base) {
        int multiplier = 10;               // local variable

        class Task implements Runnable {   // local inner class
            @Override public void run() {
                System.out.println(base * multiplier);  // captures base & multiplier
            }
        }
        return new Task();
    }
}
```

### 3.1 Key properties
- **Scoped to the enclosing block** — can't be used elsewhere, not even by other methods.
- Can access the enclosing class's members **and** local variables of the method — but only **effectively final** ones (see §3.2).
- Rarely used directly; **lambdas/anonymous classes usually replace them**. Useful when you need a *named* class used multiple times within one method.

### 3.2 Variable capture & "effectively final"
A local (or anonymous) inner class can use local variables only if they are **`final` or effectively final** (assigned once, never reassigned):
```java
int x = 5;            // effectively final (never reassigned) -> OK to capture
// x = 6;             // if you uncomment this, capture would FAIL to compile
Runnable r = () -> System.out.println(x);
```
**Why?** The inner class instance may outlive the method's stack frame. Java **captures a copy** of the local variable's value (it can't safely reference a stack slot that's gone). Requiring effective-finality avoids confusing situations where the copy and the original could diverge. (This same rule governs lambdas — Phase 1.8.)

---

## 4. Anonymous Inner Class

A class with **no name**, declared and instantiated **in a single expression**. Used to create a **one-off** implementation of an interface or subclass of a class on the spot.

```java
Runnable r = new Runnable() {              // anonymous class implementing Runnable
    @Override public void run() {
        System.out.println("running");
    }
};

button.addActionListener(new ActionListener() {   // classic event-listener use
    @Override public void actionPerformed(ActionEvent e) {
        System.out.println("clicked");
    }
});
```

### 4.1 Key properties
- **No name**, **no constructor** (you can't define one), used **exactly once**.
- Can extend a class **or** implement **one** interface (not both, not multiple interfaces).
- Captures effectively-final local variables (like local classes).
- The syntax `new Type() { ... }` means "create an anonymous subclass/implementation of `Type` and instantiate it."
- Compiled to a class named like `Outer$1.class`.

### 4.2 Anonymous class vs Lambda (very important)
Since Java 8, **lambdas replace anonymous classes for functional interfaces** (single abstract method). Lambdas are shorter and clearer:
```java
// Anonymous inner class (verbose):
Runnable a = new Runnable() {
    @Override public void run() { System.out.println("hi"); }
};

// Lambda (preferred for functional interfaces):
Runnable b = () -> System.out.println("hi");
```

| Aspect | Anonymous inner class | Lambda |
|--------|-----------------------|--------|
| Works for | Any interface/abstract class | **Functional interfaces only** (1 abstract method) |
| Multiple methods | ✅ Can implement several | ❌ Only the single abstract method |
| `this` refers to | The **anonymous class instance** | The **enclosing instance** |
| Can have fields/state | ✅ Yes | ❌ No |
| Verbosity | High | Low |
| Creates a new class file | Yes (`Outer$1`) | No (uses `invokedynamic`) |

> **Use a lambda** when implementing a functional interface. **Use an anonymous class** when you need multiple methods, state/fields, or to subclass a (non-functional) class. (Lambdas in depth: Phase 1.8.)

### 4.3 The `this` gotcha
Inside an **anonymous class**, `this` refers to the *anonymous instance*, not the outer object — use `Outer.this` for the outer. Inside a **lambda**, `this` refers to the *enclosing* instance. This difference occasionally causes bugs.

---

## 5. Choosing the Right Nested Class (Decision Guide)

```
Need a class nested in another?
│
├─ Does it need the outer INSTANCE's members?
│     ├─ No  → STATIC nested class   (preferred default; no leak risk)
│     └─ Yes → non-static MEMBER inner class
│
├─ Only used inside ONE method?
│     ├─ Reused/named within the method → LOCAL inner class
│     └─ One-off implementation inline:
│           ├─ Functional interface (1 method) → LAMBDA (modern choice)
│           └─ Needs multiple methods / state / subclass a class → ANONYMOUS class
```

| Use case | Best choice |
|----------|-------------|
| Helper/builder grouped with outer | **Static nested class** |
| Class genuinely needs the outer instance | **Member inner class** |
| Short-lived class used once in a method | **Local class** (rare) |
| One-off functional interface impl | **Lambda** |
| One-off impl needing state/multiple methods | **Anonymous class** |

---

## 6. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Non-static inner class causing memory leaks | Make it `static` if it doesn't need the outer instance |
| Forgetting `outer.new Inner()` syntax | Member inner classes need an outer instance |
| Trying to reassign a captured local variable | Captured locals must be effectively final |
| Using anonymous class for a functional interface | Use a lambda instead |
| Confusing `this` in anonymous class vs lambda | Anonymous `this` = itself; lambda `this` = enclosing |
| Putting `static` members in a non-static inner class | Not allowed pre-Java 16 (use static nested class) |
| Overusing deeply nested classes | Extract to top-level classes for readability |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Builder pattern** (static nested class) is everywhere in idiomatic Java and Spring config/HTTP clients (Encapsulation note).
- **Lambdas** (the modern replacement for anonymous classes) power the **Stream API**, `Comparator`s, callbacks, `CompletableFuture`, and reactive code (Phase 1.8, 16).
- **Anonymous classes** still appear for callbacks needing multiple methods or state, and in tests (e.g., custom `Answer` in Mockito).
- **Memory-leak awareness:** non-static inner classes (and inner-class-based listeners/callbacks) are a classic source of leaks in long-running servers — directly relevant to Phase 13 performance and Phase 1.11 leak diagnosis.
- **`ThreadLocal` / callback patterns:** understanding capture and the outer-reference behavior helps avoid subtle concurrency and lifecycle bugs (Phase 1.10).

---

## 8. Quick Self-Check Questions

1. What are the four kinds of nested classes, and which one is not truly an "inner" class?
2. How do you instantiate a member inner class, and why is the syntax unusual?
3. What memory-leak risk do non-static inner classes pose, and how do you avoid it?
4. When should a nested class be `static` (state the rule of thumb)?
5. What can a static nested class access from its outer class, and what can't it?
6. What does "effectively final" mean, and why must captured locals satisfy it?
7. When should you use a lambda vs an anonymous inner class?
8. What does `this` refer to inside an anonymous class vs a lambda?
9. Give a real-world use for a static nested class and for an anonymous class.

---

## 9. Key Terms Glossary

- **Nested class:** a class declared inside another class.
- **Member inner class:** non-static nested class tied to an outer instance.
- **Static nested class:** nested class with no outer-instance reference.
- **Local inner class:** class declared inside a method/block.
- **Anonymous inner class:** unnamed class defined and instantiated in one expression.
- **`Outer.this`:** reference to the enclosing instance from within an inner class.
- **`outer.new Inner()`:** syntax to create a member inner class.
- **Effectively final:** a variable assigned once and never reassigned (capturable).
- **Variable capture:** an inner class/lambda using enclosing local variables (by copy).
- **Lambda:** concise implementation of a functional interface (replaces many anonymous classes).
- **`invokedynamic`:** the JVM mechanism backing lambdas (no extra class file).

---

*Previous topic: **Enums**.*
*Next topic in roadmap: **Object Class Methods (`toString`, `equals`, `hashCode`, `clone`, `finalize`, `getClass`, `wait`/`notify`)**.*
