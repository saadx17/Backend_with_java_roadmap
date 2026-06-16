# Object Class Methods

> **Phase 1 — Java Language Mastery → 1.3 Object-Oriented Programming**
> Goal: Master the methods every Java object inherits from `java.lang.Object` — `toString()`, `equals()`, `hashCode()` (and their contracts), `clone()`, `finalize()`, `getClass()`, and `wait()`/`notify()`/`notifyAll()`.

---

## 0. The Big Picture

Every class in Java implicitly extends **`java.lang.Object`** (recall the Inheritance note). So **every object** inherits a set of fundamental methods. Knowing them — and overriding the right ones correctly — is essential for correct, idiomatic Java.

```java
public class Object {
    public String toString() { ... }
    public boolean equals(Object obj) { ... }
    public int hashCode() { ... }
    protected Object clone() throws CloneNotSupportedException { ... }
    protected void finalize() { ... }          // deprecated
    public final Class<?> getClass() { ... }
    public final void wait() throws InterruptedException { ... }
    public final void notify() { ... }
    public final void notifyAll() { ... }
}
```

| Method | Override it? | Purpose |
|--------|--------------|---------|
| `toString()` | ✅ Usually | Human-readable string representation |
| `equals()` | ✅ Often | Logical (value) equality |
| `hashCode()` | ✅ **Always with `equals()`** | Hash bucket placement |
| `clone()` | ⚠️ Rarely (avoid) | Copy an object |
| `finalize()` | ❌ Never (deprecated) | Pre-GC cleanup (don't use) |
| `getClass()` | ❌ Can't (final) | Runtime type info |
| `wait`/`notify`/`notifyAll` | ❌ Can't (final) | Thread coordination |

> The three you'll override most are **`toString()`, `equals()`, `hashCode()`** — and `equals`/`hashCode` **must always be overridden together**.

---

## 1. `toString()`

Returns a **string representation** of the object. The default is nearly useless — `ClassName@hexHashCode`:
```java
Person p = new Person("Sam", 30);
System.out.println(p);   // Person@1b6d3586   <- default: unhelpful
```

### 1.1 Override it for readability
```java
@Override
public String toString() {
    return "Person{name='" + name + "', age=" + age + "}";
}
// now: Person{name='Sam', age=30}
```

### 1.2 Guidelines
- Called automatically by string concatenation, `System.out.println`, loggers, and debuggers.
- Make it **concise, informative, and unambiguous** — great for **logging and debugging**.
- **Don't include sensitive data** (passwords, tokens, full PII) — they'll leak into logs (Phase 9, 15).
- **Records** generate a good `toString()` automatically (previous note). Lombok's `@ToString` does too.

---

## 2. `equals()` — Logical Equality

### 2.1 `==` vs `.equals()` (fundamental)
- **`==`** compares **references** (are these the *same object* in memory? — identity). For primitives, it compares values.
- **`.equals()`** compares **logical/value equality** (do these objects *mean the same thing?*) — **if overridden**.

```java
String a = new String("hi");
String b = new String("hi");
a == b;        // false -> different objects on the heap
a.equals(b);   // true  -> same content (String overrides equals)
```
The default `Object.equals()` is just `==` (identity). You override it to define meaningful equality.

### 2.2 The `equals()` contract (5 rules — must hold)
A correct `equals()` must be:
| Property | Meaning |
|----------|---------|
| **Reflexive** | `x.equals(x)` is always `true` |
| **Symmetric** | `x.equals(y)` ⇔ `y.equals(x)` |
| **Transitive** | if `x.equals(y)` and `y.equals(z)`, then `x.equals(z)` |
| **Consistent** | repeated calls return the same result (if objects unchanged) |
| **Non-null** | `x.equals(null)` is always `false` |

Violating these breaks collections (`HashSet`, `HashMap`, `List.contains`, etc.) in subtle, hard-to-debug ways.

### 2.3 A correct `equals()` implementation
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;                 // 1. same reference (fast path)
    if (o == null || getClass() != o.getClass()) return false;  // 2. type check
    Person person = (Person) o;                 // 3. safe cast
    return age == person.age                     // 4. compare significant fields
        && Objects.equals(name, person.name);    //    (null-safe for objects)
}
```
- Use `Objects.equals(a, b)` for null-safe field comparison.
- Compare **only the fields that define logical identity**.

### 2.4 `getClass()` vs `instanceof` in equals (subtle)
- `getClass() != o.getClass()` → strict: an object equals only objects of the *exact* same class (preserves symmetry with subclasses).
- `instanceof` → allows subclass instances to be equal, but can **break symmetry/transitivity** in inheritance hierarchies. `getClass()` is the safer default for most value types.

---

## 3. `hashCode()` — and Its Contract with `equals()`

`hashCode()` returns an `int` used to determine an object's **bucket** in hash-based collections (`HashMap`, `HashSet`, `HashTable`).

### 3.1 The golden rule
> **If you override `equals()`, you MUST override `hashCode()`** — and consistently.

### 3.2 The `hashCode()` contract
| Rule | Meaning |
|------|---------|
| **Consistency** | Same object → same hashCode across calls (if unchanged) |
| **equals ⇒ equal hashCodes** | If `a.equals(b)`, then `a.hashCode() == b.hashCode()` (**mandatory**) |
| **Unequal objects** | *May* have the same hashCode (collisions allowed), but distinct hashes improve performance |

> The reverse is **not** required: equal hashCodes do **not** imply `equals()` (collisions happen).

### 3.3 What breaks if you violate it
```java
// Class overrides equals() but NOT hashCode():
Set<Person> set = new HashSet<>();
set.add(new Person("Sam", 30));
set.contains(new Person("Sam", 30));   // FALSE! -> different hashCode -> wrong bucket
```
The object is "lost" in the map/set because lookup checks the bucket (via hashCode) *before* `equals()`. This is one of the most common real-world Java bugs.

### 3.4 How hash collections use both (recap from Collections concept)
```
put/get/contains:
  1. compute hashCode() -> pick the bucket
  2. within the bucket, use equals() to find the exact match
```
So **both** must agree: hashCode finds the bucket, equals finds the element.

### 3.5 A correct `hashCode()`
```java
@Override
public int hashCode() {
    return Objects.hash(name, age);   // use the SAME fields as equals()
}
```
- `Objects.hash(...)` is the simple, idiomatic choice.
- **Use the same fields** in `hashCode()` as in `equals()` — never more, never fewer.

### 3.6 Mutability warning
If you use mutable fields in `hashCode()` and then **mutate** an object that's a key in a `HashMap`/`HashSet`, its hashCode changes → it becomes **unreachable** in the collection. **Prefer immutable objects (or immutable key fields)** for map keys (ties to Encapsulation/Records notes).

> **Records** auto-generate correct, consistent `equals()` + `hashCode()` (value-based) — a major reason to use them for value objects/DTOs.

---

## 4. `clone()` — Shallow vs Deep Copy

`clone()` creates a copy of an object. It's notoriously **awkward and error-prone**, and generally **avoided** in modern Java.

### 4.1 How it (barely) works
- `Object.clone()` is `protected` and throws `CloneNotSupportedException` unless the class implements the **`Cloneable`** marker interface (recall the Interfaces note).
```java
public class Point implements Cloneable {
    int x, y;
    @Override
    public Point clone() {
        try {
            return (Point) super.clone();   // shallow copy
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

### 4.2 Shallow vs deep clone
- **Shallow clone (default):** copies field values directly. For reference fields, it copies the **reference** → original and clone **share** the nested objects.
- **Deep clone:** also clones the nested mutable objects → fully independent copy.
```java
@Override
public Person clone() {
    Person copy = (Person) super.clone();
    copy.addresses = new ArrayList<>(this.addresses);  // deep-copy mutable field
    return copy;
}
```
(Same shallow/deep distinction from the Encapsulation and Records notes.)

### 4.3 Why `clone()` is problematic (and what to use instead)
| Problem | Detail |
|---------|--------|
| Broken design | `Cloneable` is a marker with no `clone()` method; `clone()` lives on `Object` |
| Checked exception | Forces awkward try/catch |
| No constructor call | `clone()` bypasses constructors → can break invariants |
| Shallow by default | Easy to forget deep copying → shared mutable state bugs |

**Preferred alternatives (*Effective Java* Item 13):**
- **Copy constructor:** `new Person(other)` (Classes & Objects note).
- **Static copy factory:** `Person.copyOf(other)`.
- For immutable types (records, `String`), no copy is needed at all.

> **Bottom line:** avoid `clone()`/`Cloneable`. Use a copy constructor or factory.

---

## 5. `finalize()` — Deprecated, Don't Use

`finalize()` was historically called by the GC **before** reclaiming an object, intended for cleanup. **It is deprecated (since Java 9) and should never be used.**

### 5.1 Why it's bad
| Problem | Detail |
|---------|--------|
| **Unpredictable** | No guarantee it runs, or *when* (depends on GC) |
| **Slow** | Hurts GC performance |
| **Can resurrect objects** | An object can make itself reachable again in `finalize()` |
| **Exceptions ignored** | Exceptions in `finalize()` are swallowed |
| **Security/correctness risks** | Finalizer attacks, partially-constructed objects |

### 5.2 What to use instead
- **`try-with-resources`** + the **`AutoCloseable`** interface for deterministic cleanup of resources (files, sockets, connections):
```java
try (var conn = dataSource.getConnection()) {   // auto-closed, deterministically
    // use conn
}   // conn.close() called automatically, even on exception
```
- **`Cleaner`** (Java 9+) as a safety-net for native resources (advanced).

(Resource cleanup & try-with-resources are covered in Phase 1.5 Exceptions and 1.9 I/O.)

---

## 6. `getClass()` — Runtime Type Information

Returns the **runtime `Class` object** of the instance — the entry point to **reflection** (Phase 1.14).

```java
Object o = "hello";
System.out.println(o.getClass().getName());        // java.lang.String
System.out.println(o.getClass().getSimpleName());  // String
```
- `final` — **cannot be overridden**.
- Returns the **actual runtime type**, not the reference type (`o` is declared `Object` but returns `String`).
- Used in `equals()` (strict type checks), logging, frameworks, and reflection.

### 6.1 `getClass()` vs `.class` vs `instanceof`
| Expression | Gives | Resolved |
|------------|-------|----------|
| `obj.getClass()` | Runtime class of the object | Runtime |
| `String.class` | Class literal for a known type | Compile time |
| `obj instanceof String` | Boolean: is it (a subtype of) String? | Runtime |

---

## 7. `wait()`, `notify()`, `notifyAll()` — Thread Coordination

These `final` methods support **inter-thread communication** using an object's **intrinsic lock (monitor)**. They're the low-level basis of the *wait/notify* pattern.

```java
synchronized (lock) {          // must hold the monitor to call wait/notify
    while (!condition) {
        lock.wait();           // release the lock & wait until notified
    }
    // proceed when condition is true
}

synchronized (lock) {
    condition = true;
    lock.notifyAll();          // wake waiting threads
}
```

| Method | Effect |
|--------|--------|
| `wait()` | Releases the monitor and suspends the thread until notified/interrupted |
| `notify()` | Wakes **one** waiting thread on this monitor |
| `notifyAll()` | Wakes **all** waiting threads on this monitor (usually safer) |

### 7.1 Critical rules
- Must be called **inside a `synchronized` block** on the same object, or you get `IllegalMonitorStateException`.
- Always call `wait()` in a **loop checking the condition** (guards against *spurious wakeups* and lost notifications).
- Prefer `notifyAll()` over `notify()` to avoid stranded threads.

> ⚠️ These are **low-level primitives**. In modern code, prefer the higher-level `java.util.concurrent` tools — `BlockingQueue`, `CountDownLatch`, `Semaphore`, `Condition`, `CompletableFuture` — which are safer and clearer. (Full coverage in **Phase 1.10 — Concurrency**.)

---

## 8. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Overriding `equals()` but not `hashCode()` | Always override both, using the same fields |
| Using `==` to compare object content (e.g., Strings) | Use `.equals()` |
| Mutating a field used in `hashCode()` while it's a map key | Use immutable keys |
| Violating the equals contract (symmetry/transitivity) | Follow the 5 rules; use `getClass()` checks |
| Using `clone()`/`Cloneable` | Use a copy constructor or factory |
| Relying on `finalize()` for cleanup | Use try-with-resources / `AutoCloseable` |
| Calling `wait`/`notify` outside `synchronized` | Always hold the monitor; wait in a loop |
| Putting secrets in `toString()` | Exclude sensitive data |
| Using `notify()` when multiple conditions exist | Prefer `notifyAll()` |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **`equals()`/`hashCode()`** are critical for **JPA entities** (identity in collections, the `Set`-mapping caveat), caching keys, and deduplication — a frequent source of subtle bugs (Phase 5.4).
- **`toString()`** drives readable **logging** — but must exclude PII/secrets (Phase 9, 15).
- **Records** (and Lombok) auto-generate `equals`/`hashCode`/`toString` correctly — preferred for DTOs/value objects.
- **`try-with-resources` over `finalize()`** is mandatory for connections, streams, and sessions in backend code (HikariCP, JDBC — Phase 4/13).
- **`getClass()`/reflection** underpins Spring (DI, proxies, AOP), Jackson, and JPA (Phase 1.14, 5).
- **wait/notify → `java.util.concurrent`**: real backends use the high-level concurrency utilities, but understanding the monitor model explains how `synchronized` and locks work (Phase 1.10).

---

## 10. Quick Self-Check Questions

1. Why does every Java object have these methods, and which three do you most often override?
2. What's the difference between `==` and `.equals()`?
3. State the five rules of the `equals()` contract.
4. Why must `equals()` and `hashCode()` be overridden together? What breaks otherwise?
5. State the `hashCode()` contract. Can unequal objects share a hashCode?
6. How do hash-based collections use `hashCode()` and `equals()` together?
7. What's the difference between shallow and deep cloning? Why avoid `clone()`?
8. Why is `finalize()` deprecated, and what should you use instead?
9. What does `getClass()` return, and how does it differ from `.class` and `instanceof`?
10. What are the rules for calling `wait()`/`notify()`, and why wait in a loop?

---

## 11. Key Terms Glossary

- **`java.lang.Object`:** the root superclass of every Java class.
- **`toString()`:** human-readable representation of an object.
- **`==` vs `equals()`:** reference identity vs logical/value equality.
- **`equals()` contract:** reflexive, symmetric, transitive, consistent, non-null.
- **`hashCode()`:** int used to place objects in hash buckets.
- **`hashCode()`/`equals()` contract:** equal objects must have equal hash codes.
- **Hash collision:** different objects with the same hashCode (allowed).
- **`clone()` / `Cloneable`:** copy mechanism (avoid; prefer copy constructor/factory).
- **Shallow vs deep copy:** share nested objects vs duplicate them.
- **`finalize()`:** deprecated pre-GC cleanup hook (don't use).
- **`AutoCloseable` / try-with-resources:** deterministic resource cleanup.
- **`getClass()`:** runtime `Class` object; gateway to reflection.
- **Monitor / intrinsic lock:** per-object lock used by `synchronized`/`wait`/`notify`.
- **`wait` / `notify` / `notifyAll`:** low-level thread coordination primitives.
- **Spurious wakeup:** a thread waking without notification (why you wait in a loop).

---

*Previous topic: **Inner Classes**.*
*This completes **Section 1.3 — Object-Oriented Programming**.*
*Next section in roadmap: **1.4 Strings (Deep Dive)**.*
