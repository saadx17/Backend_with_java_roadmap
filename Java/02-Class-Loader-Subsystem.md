# Class Loader Subsystem

> **Phase 1 — Java Language Mastery → 1.11 JVM Deep Dive**
> Goal: Master how the JVM loads classes — the three phases (loading, linking, initialization), the built-in loaders, the delegation model, custom loaders, and `ClassNotFoundException` vs `NoClassDefFoundError`.

---

## 0. The Big Picture

The **class loader subsystem** is responsible for finding `.class` bytecode, bringing it into the JVM, verifying it, and preparing it for execution — **at runtime, lazily, on first use**. This explains class-loading errors (a common deployment headache) and how frameworks (Spring, app servers) isolate code.

```
First use of a class -> Class Loader: LOAD -> LINK -> INITIALIZE -> ready to use
```

> Classes are loaded **on demand** (when first referenced), not all at startup — analogous to demand paging (Phase 0.1).

---

## 1. The Three Phases of Class Loading

```
1. LOADING        find the .class bytes, create the Class object
2. LINKING        2a. Verification  2b. Preparation  2c. Resolution
3. INITIALIZATION run static initializers & static field assignments
```

### 1.1 Loading
The loader **finds** the `.class` file (from disk, JAR, network) and reads its bytecode into memory, creating a **`Class` object** (in the method area/metaspace) representing the type.

### 1.2 Linking (three sub-steps)
| Sub-step | What it does |
|----------|--------------|
| **Verification** | Checks the bytecode is valid & safe (correct format, no stack overflows, type-safe) — security & integrity |
| **Preparation** | Allocates memory for **static fields** and sets them to **default values** (`0`/`null`/`false`) — not their real values yet |
| **Resolution** | Replaces symbolic references (names) with direct references (memory addresses) — can be lazy |

### 1.3 Initialization
Runs the class's **static initializers** and assigns **static fields their real values** — recall the initialization order from Phase 1.3 (Classes & Objects):
```java
class Config {
    static int x = compute();          // assigned during INITIALIZATION (not preparation)
    static { System.out.println("init"); }  // static block runs here, ONCE
}
```
> **Initialization happens once per class, lazily** — only when the class is first **actively used** (instantiated, a static member accessed, etc.). This is why static blocks run "the first time you touch the class."

---

## 2. The Built-in Class Loaders

The JVM has a **hierarchy** of class loaders, each responsible for different classes:
```
Bootstrap Class Loader   (loads core JDK classes: java.lang.*, java.util.*)
        ^
Platform Class Loader    (loads platform/extension modules)   [Java 9+; was "Extension"]
        ^
Application Class Loader  (loads YOUR classes from the classpath)
        ^
(Custom class loaders)    (frameworks, app servers, plugins)
```
| Loader | Loads | Written in |
|--------|-------|-----------|
| **Bootstrap** | Core JDK (`java.base`: `String`, `Object`, collections) | Native (C/C++); appears as `null` |
| **Platform** (was Extension) | Platform modules (crypto, SQL, etc.) | Java |
| **Application** (System) | Your app classes from the **classpath/module path** | Java |

```java
String.class.getClassLoader();        // null (Bootstrap — core class)
MyClass.class.getClassLoader();       // Application class loader (your code)
```

---

## 3. The Delegation Model (Parent-First)

> **Key principle: a class loader first asks its PARENT to load a class, before trying itself.** This is **parent delegation**.

```
Request to load class X:
  Application loader -> "Parent (Platform), can you load X?"
     Platform loader -> "Parent (Bootstrap), can you load X?"
        Bootstrap -> tries; if not found, returns to Platform
     Platform -> tries; if not found, returns to Application
  Application -> tries to load X itself
```

### 3.1 Why delegation matters
| Benefit | Explanation |
|---------|-------------|
| **Security** | Core classes (`java.lang.String`) are always loaded by Bootstrap — you **can't** replace them with a malicious version |
| **No duplication** | A class is loaded once by the highest-capable loader |
| **Consistency** | Everyone shares the same core classes |
> Try to define your own `java.lang.String` — it'll be ignored; Bootstrap's version wins via delegation. This protects the JVM's integrity.

### 3.2 Class identity = name + loader
A class's identity is its **fully-qualified name + the loader that loaded it**. The **same** class loaded by **two different loaders** is treated as **two distinct types** → casting between them throws `ClassCastException`. (This is how app servers isolate apps, and a source of confusing "X cannot be cast to X" errors.)

---

## 4. Custom Class Loaders

You can write a custom `ClassLoader` to load classes from non-standard sources (network, encrypted files, databases) or to **isolate** classes:
```java
class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] bytes = loadBytecodeFrom(name);   // e.g., from a custom source
        return defineClass(name, bytes, 0, bytes.length);
    }
}
```
### 4.1 Where custom loaders are used
- **Application servers** (Tomcat) → each web app gets its own loader for **isolation** (so two apps can use different versions of a library).
- **Plugin systems / hot reload** → load/unload classes dynamically.
- **Spring Boot DevTools** → a restart class loader for fast reloads.
- **Frameworks** generating classes at runtime (proxies — Phase 1.14, Spring AOP).
> You rarely write custom loaders, but understanding them explains framework magic and classpath isolation.

---

## 5. ClassNotFoundException vs NoClassDefFoundError (Critical Distinction)

> One of the most confusing pairs in Java — and a frequent deployment issue.

| | **`ClassNotFoundException`** | **`NoClassDefFoundError`** |
|---|------------------------------|----------------------------|
| Type | Checked **Exception** | **Error** |
| When | A class is requested **by name at runtime** and not found | A class **was present at compile time** but is **missing/failed at runtime** |
| Trigger | `Class.forName("X")`, `loadClass("X")` reflection | Normal reference to a class that can't be loaded |
| Typical cause | Dynamic lookup of a name that isn't on the classpath | Missing JAR at runtime, or a class that **failed initialization earlier** |

### 5.1 Examples
```java
// ClassNotFoundException — explicit, dynamic lookup:
Class.forName("com.example.NotOnClasspath");   // -> ClassNotFoundException

// NoClassDefFoundError — the class existed at compile time but isn't loadable now:
// e.g., you compiled against a library JAR but forgot to include it at runtime,
// OR a class threw during its static initialization (so it can never be defined).
```

### 5.2 The subtle `NoClassDefFoundError` cause
A very confusing case: if a class's **static initializer throws** an exception, the class fails to initialize. The *first* attempt throws `ExceptionInInitializerError`; **subsequent** uses throw `NoClassDefFoundError` (the JVM marks the class as failed). So `NoClassDefFoundError` doesn't always mean "JAR missing" — it can mean "this class failed to initialize earlier."

### 5.3 How to debug them
| Symptom | Check |
|---------|-------|
| `ClassNotFoundException` | Is the class/JAR on the classpath? Typo in the name? Reflection target present? |
| `NoClassDefFoundError` | Missing runtime dependency? A `Caused by` earlier showing a static-init failure? Version conflict? |
> Read the **`Caused by:`** chain (recall Phase 1.5 stack traces) — for `NoClassDefFoundError`, the root cause is often a different earlier exception.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Thinking all classes load at startup | They load **lazily**, on first active use |
| Confusing the two phases of static fields | Preparation sets defaults; initialization sets real values |
| `ClassNotFoundException` vs `NoClassDefFoundError` confusion | Exception=dynamic-lookup miss; Error=present-at-compile-but-missing/failed-at-runtime |
| Trying to override core classes (`java.lang.*`) | Delegation makes Bootstrap win — you can't |
| "X cannot be cast to X" | Same class via different loaders = different types |
| Assuming `NoClassDefFoundError` = missing JAR | Could be a failed static initializer — read `Caused by` |
| Ignoring class-loading in app servers | Each app/loader isolates classes (version conflicts) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Deployment errors:** `ClassNotFoundException`/`NoClassDefFoundError` from missing or conflicting dependencies are common (Maven/Gradle, fat JARs — Phase 3, 10).
- **Spring Boot fat JARs** use a **custom class loader** to load nested JARs (Phase 5.2, 10).
- **Spring AOP / proxies** generate classes at runtime via class loaders (Phase 1.14, 5.1).
- **DevTools hot reload** uses a restart class loader (Phase 5.2).
- **Dependency conflicts** (two versions of a library) manifest as class-loading errors → understanding loaders helps resolve them (`mvn dependency:tree`, Phase 3).
- **Class-loading leaks** (redeploying apps without releasing old loaders) cause Metaspace OOM (later note, Phase 13).

---

## 8. Quick Self-Check Questions

1. What are the three phases of class loading, and what happens in each?
2. What's the difference between preparation and initialization for static fields?
3. When is a class initialized (its static block run)?
4. Name the three built-in class loaders and what each loads.
5. What is the parent-delegation model, and why does it matter (security)?
6. Why can the same class loaded by two loaders cause `ClassCastException`?
7. What's the precise difference between `ClassNotFoundException` and `NoClassDefFoundError`?
8. Give a non-obvious cause of `NoClassDefFoundError`.

---

## 9. Key Terms Glossary

- **Class loader subsystem:** loads/links/initializes classes at runtime.
- **Loading / linking / initialization:** find bytecode / verify+prepare+resolve / run static init.
- **Verification / preparation / resolution:** validity check / default-value allocation / symbolic→direct references.
- **Bootstrap / Platform / Application loaders:** core JDK / platform modules / your classpath.
- **Parent delegation:** ask the parent loader first.
- **Custom class loader:** user-defined loader (isolation, dynamic loading).
- **Class identity:** fully-qualified name + defining loader.
- **`ClassNotFoundException`:** dynamic lookup of an absent class (Exception).
- **`NoClassDefFoundError`:** compile-time-present class missing/failed at runtime (Error).
- **`ExceptionInInitializerError`:** thrown when a static initializer fails.

---

*Previous topic: **JVM Architecture Overview**.*
*Next topic: **Runtime Data Areas & Execution Engine (JIT)**.*
