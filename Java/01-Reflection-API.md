# Reflection API

> **Phase 1 — Java Language Mastery → 1.14 Reflection API**
> Goal: Master Java reflection — inspecting classes/fields/methods/annotations at runtime, creating instances and invoking methods dynamically, accessing private members, and the performance/security implications.

---

## 0. The Big Picture

**Reflection** lets a program **inspect and manipulate** its own structure — classes, fields, methods, constructors, annotations — **at runtime**, without knowing them at compile time. It's the mechanism behind the "magic" of Spring, Hibernate, Jackson, JUnit, and most Java frameworks.

```java
Class<?> clazz = Class.forName("com.example.User");   // get a class by name at runtime
Object instance = clazz.getDeclaredConstructor().newInstance();  // create it
Method m = clazz.getMethod("getName");                // find a method
Object name = m.invoke(instance);                     // call it dynamically
```

> Reflection is **how frameworks work generically** — they don't know your classes at compile time, so they discover and operate on them at runtime. Understanding it demystifies dependency injection, ORM mapping, serialization, and annotation processing (recall Phase 1.13).

---

## 1. Why Reflection Exists

Normally, code references classes/methods **statically** (known at compile time). But frameworks must work with **your** classes — which they've never seen. Reflection bridges this:
```
Spring at startup: "scan the classpath -> find classes annotated @Component ->
                    read their constructors -> create instances -> inject dependencies"
(All via reflection — Spring knows nothing about your classes at compile time.)
```
| Use case | Framework | What reflection does |
|----------|-----------|----------------------|
| Dependency injection | Spring | Find beans, invoke constructors, set fields |
| ORM mapping | Hibernate/JPA | Read `@Entity` fields, set/get values |
| Serialization | Jackson | Read/write object fields to/from JSON |
| Testing | JUnit | Find/invoke `@Test` methods |
| Mocking | Mockito | Create proxies, intercept calls |

---

## 2. The Class Object — Entry Point

Everything in reflection starts with a **`Class<?>` object** (the runtime representation of a type — recall it lives in metaspace, Phase 1.11). Three ways to get one:
```java
// 1. From an instance:
Class<?> c1 = myObject.getClass();

// 2. From a class literal (compile-time known):
Class<User> c2 = User.class;

// 3. By name (fully dynamic — may throw ClassNotFoundException, recall Phase 1.11):
Class<?> c3 = Class.forName("com.example.User");
```
> `getClass()` returns the **runtime** type (recall Phase 1.3 — `Object o = "hi"; o.getClass()` is `String`). `Class.forName` is the fully dynamic option (loads the class if needed).

---

## 3. Inspecting a Class

Once you have a `Class<?>`, you can examine its structure:
```java
Class<?> c = User.class;

c.getName();                  // fully-qualified: "com.example.User"
c.getSimpleName();            // "User"
c.getSuperclass();            // the parent Class
c.getInterfaces();            // implemented interfaces
c.getModifiers();             // int bitmask (use Modifier.isPublic(...), etc.)

c.getDeclaredFields();        // ALL fields declared in this class (incl. private)
c.getFields();                // only PUBLIC fields (incl. inherited)
c.getDeclaredMethods();       // ALL methods declared here (incl. private)
c.getMethods();               // PUBLIC methods (incl. inherited)
c.getDeclaredConstructors();  // all constructors
c.getAnnotations();           // annotations present at runtime (RUNTIME retention!)
```
> ⚠️ **`getXxx()` vs `getDeclaredXxx()`:** `getFields()`/`getMethods()` return only **public** members (including inherited); `getDeclaredFields()`/`getDeclaredMethods()` return **all** members **declared in this class** (including private) but **not** inherited ones. A frequent point of confusion.

---

## 4. Creating Instances Dynamically

```java
Class<?> c = Class.forName("com.example.User");

// No-arg constructor (the modern way; newInstance() is deprecated):
Object obj = c.getDeclaredConstructor().newInstance();

// Constructor with arguments:
Constructor<?> ctor = c.getDeclaredConstructor(String.class, int.class);
Object user = ctor.newInstance("Alice", 30);
```
> This is exactly how Spring instantiates beans and how JPA/Jackson create entity/DTO instances. It's also **why JPA entities need a no-arg constructor** (recall Phase 1.3 — Hibernate calls `getDeclaredConstructor().newInstance()`).

---

## 5. Invoking Methods Dynamically

```java
Class<?> c = obj.getClass();
Method method = c.getMethod("setName", String.class);   // find by name + param types
method.invoke(obj, "Bob");                               // call it on obj with args

Method getter = c.getMethod("getName");
Object result = getter.invoke(obj);                      // returns the result
```
- `getMethod(name, paramTypes...)` finds a public method; `getDeclaredMethod(...)` finds any (incl. private).
- `invoke(target, args...)` calls it; for static methods, pass `null` as the target.
- Exceptions thrown by the invoked method are wrapped in **`InvocationTargetException`** (unwrap with `getCause()` — recall Phase 1.5 chaining).

---

## 6. Accessing Private Members

Reflection can access (and modify) **private** fields and methods — bypassing normal access control (recall Phase 1.3 access modifiers):
```java
Field field = c.getDeclaredField("password");   // a private field
field.setAccessible(true);                       // bypass access checks
Object value = field.get(obj);                   // read it
field.set(obj, "newSecret");                     // write it (even if final-ish)

Method privateMethod = c.getDeclaredMethod("internalLogic");
privateMethod.setAccessible(true);
privateMethod.invoke(obj);
```
> `setAccessible(true)` defeats `private`/`protected` — powerful but dangerous. It's how Spring sets private `@Autowired` fields and how Hibernate populates private entity fields. ⚠️ Under **JPMS** (Phase 1.12), this requires the package to be `opens`ed, or you get an `InaccessibleObjectException`.

---

## 7. Reading Annotations via Reflection (ties to Phase 1.13)

Reflection is how frameworks **act on annotations** at runtime (which is why they must be `@Retention(RUNTIME)` — recall Phase 1.13):
```java
if (clazz.isAnnotationPresent(Entity.class)) {           // is it an @Entity?
    Entity entity = clazz.getAnnotation(Entity.class);
}
for (Field f : clazz.getDeclaredFields()) {
    if (f.isAnnotationPresent(Column.class)) {
        Column col = f.getAnnotation(Column.class);
        String columnName = col.name();                  // read annotation metadata
    }
}
```
> This is the complete picture of "annotations + reflection = framework magic": annotations declare intent (Phase 1.13), reflection reads and enacts it at runtime.

---

## 8. Dynamic Proxies (awareness)

Java's `java.lang.reflect.Proxy` can create **proxy objects** at runtime that implement interfaces and intercept method calls via an `InvocationHandler`:
```java
MyService proxy = (MyService) Proxy.newProxyInstance(
    loader, new Class[]{MyService.class},
    (proxyObj, method, args) -> {
        // intercept: e.g., start transaction, log, then call the real method
        return method.invoke(realTarget, args);
    });
```
> This underpins **Spring AOP** (`@Transactional`, `@Cacheable`), Mockito mocks, and more. Spring uses **JDK dynamic proxies** (for interfaces) or **CGLIB/ByteBuddy** (for classes — recall Phase 1.11 bytecode manipulation). (Full AOP coverage in Phase 5.1.)

---

## 9. Performance & Security Implications (Critical)

### 9.1 Performance
- Reflection is **slower** than direct calls: it involves lookups, security checks, and can't be optimized/inlined by the JIT as easily (recall Phase 1.11).
- For **occasional** use (startup, framework wiring) it's fine. In **hot loops**, avoid it — **cache** the reflected `Method`/`Field` objects if you must use it repeatedly.
- Modern alternatives reduce reflection cost: **`MethodHandle`/`VarHandle`** (faster), **compile-time codegen** (annotation processors — Phase 1.13: Lombok/MapStruct), and **Spring AOT** (Phase 16.5).

### 9.2 Security
- Reflection **breaks encapsulation** — `setAccessible(true)` accesses private state, bypassing the access controls you carefully designed (Phase 1.3).
- This is a **security concern**: untrusted code using reflection could read secrets or break invariants. A `SecurityManager` (now deprecated) historically restricted it; under **JPMS**, strong encapsulation + `opens` controls reflective access (Phase 1.12).
- **GraalVM native image** (Phase 16.5) struggles with reflection (it needs to know reflective targets ahead of time) → reflection config files / a push toward compile-time approaches.

### 9.3 Trade-offs summary
| Aspect | Reflection |
|--------|------------|
| Flexibility | Very high (work with unknown types) |
| Performance | Slower (cache reflected members; avoid in hot paths) |
| Type safety | Lost (errors surface at runtime, not compile time) |
| Encapsulation | Bypassable (`setAccessible`) — power + risk |
| Native image | Problematic (needs explicit config) |

> **Rule of thumb:** reflection is great for **frameworks and tooling** (run once at startup), not for everyday application logic. Prefer normal, type-safe code; let frameworks handle the reflection.

---

## 10. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using reflection in hot loops | Cache `Method`/`Field`; or avoid reflection there |
| Confusing `getMethods()` vs `getDeclaredMethods()` | public+inherited vs all-declared(-here) |
| Forgetting `setAccessible(true)` for privates | Set it (and handle JPMS `opens`) |
| Not unwrapping `InvocationTargetException` | Use `getCause()` for the real exception |
| Annotation not visible via reflection | It must be `@Retention(RUNTIME)` (Phase 1.13) |
| `newInstance()` (deprecated) | Use `getDeclaredConstructor().newInstance()` |
| Losing type safety silently | Reserve reflection for frameworks; prefer static code |
| Reflection breaking under native image | Provide reflection config / use AOT (Phase 16.5) |

---

## 11. Connection to Backend / Spring (Why This Matters Later)

- **Dependency Injection (Phase 5.1):** Spring uses reflection to find beans, invoke constructors, and set `@Autowired` fields.
- **JPA/Hibernate (Phase 5.4):** reflects on `@Entity` classes to read/write fields → why entities need a no-arg constructor and (under JPMS) `opens`.
- **Jackson (Phase 5.3):** reflects on fields/getters to serialize/deserialize JSON.
- **Spring AOP (Phase 5.1):** dynamic proxies wrap beans for `@Transactional`/`@Cacheable`.
- **Mockito (Phase 6):** creates mocks/proxies via reflection + bytecode manipulation.
- **Annotations + reflection (Phase 1.13):** the combined mechanism behind all of the above.
- **Performance/native (Phase 13, 16.5):** reflection cost motivates caching and AOT/compile-time alternatives.

---

## 12. Quick Self-Check Questions

1. What is reflection, and why do frameworks need it?
2. What are the three ways to obtain a `Class<?>` object?
3. What's the difference between `getMethods()` and `getDeclaredMethods()`?
4. How do you create an instance and invoke a method dynamically?
5. How do you access a private field, and what's the JPMS caveat?
6. Why must an annotation be `@Retention(RUNTIME)` to be read reflectively?
7. What are dynamic proxies used for in Spring?
8. What are the performance and security implications of reflection, and when should you use it?

---

## 13. Key Terms Glossary

- **Reflection:** runtime inspection/manipulation of code structure.
- **`Class<?>` object:** runtime representation of a type.
- **`Class.forName` / `.class` / `getClass()`:** ways to get a `Class`.
- **`Field` / `Method` / `Constructor`:** reflective members.
- **`getDeclared*` vs `get*`:** all-declared(-here) vs public(+inherited).
- **`newInstance()` (via constructor):** create an object reflectively.
- **`invoke()`:** call a method reflectively.
- **`setAccessible(true)`:** bypass access control to reach private members.
- **`InvocationTargetException`:** wraps an exception thrown by an invoked method.
- **Dynamic proxy / `InvocationHandler`:** runtime-generated interceptor objects.
- **`MethodHandle` / `VarHandle`:** faster reflective alternatives.
- **AOT / native image:** compile-time approach that limits reflection.

---

*This is the note for **Section 1.14 — Reflection API**.*
*This completes **Phase 1 — Java Language Mastery** (except 1.1 Dev Environment Setup).*
*Next: **1.1 Development Environment Setup**, or proceed to **Phase 2 — Data Structures & Algorithms**.*
