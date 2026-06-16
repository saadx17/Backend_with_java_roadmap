# Java Modules (JPMS — Java Platform Module System)

> **Phase 1 — Java Language Mastery → 1.12 Java Modules (Java 9+)**
> Goal: Understand the Java Platform Module System — `module-info.java` (exports, requires, opens), module path vs classpath, and automatic/unnamed modules.

---

## 0. The Big Picture

The **Java Platform Module System (JPMS)**, introduced in **Java 9** ("Project Jigsaw"), adds a layer of structure **above packages**: a **module** is a named, self-describing group of packages with **explicit dependencies** and **explicit exports**. It enforces **strong encapsulation** at the module level.

```
Class -> Package -> MODULE   (modules are the new top-level grouping)
```

> JPMS lets you say: "this module **requires** these other modules, and **exports** only these packages — everything else is hidden, even if public." It's reliable configuration + strong encapsulation. The JDK itself was modularized with it (`java.base`, `java.sql`, etc.).

> ⚠️ **Reality check:** most **application** and **Spring Boot** projects still use the **classpath**, not modules. JPMS is widely used *inside the JDK* and some libraries, but app adoption is limited. Know it conceptually; you'll rarely write `module-info.java` in typical backend work.

---

## 1. Why Modules Exist (the problems they solve)

Before Java 9, the **classpath** had real weaknesses:
| Problem (classpath era) | What JPMS adds |
|-------------------------|----------------|
| **No real encapsulation** — `public` = visible to everyone | Strong encapsulation: only **exported** packages are accessible |
| **"JAR hell"** — missing/duplicate JARs found only at runtime | Reliable configuration: missing modules detected at **startup** |
| **Monolithic JDK** — always shipped the whole thing | Modular JDK → small custom runtimes via `jlink` |
| No explicit dependency declaration | Modules declare `requires` explicitly |

---

## 2. The module-info.java Descriptor

A module is defined by a **`module-info.java`** file at the root of its source, declaring its name, dependencies, and exports:
```java
module com.example.myapp {            // module name (usually reverse-DNS)
    requires java.sql;                // depends on the java.sql module
    requires com.example.utils;       // depends on another module

    exports com.example.myapp.api;    // makes this package accessible to others
    exports com.example.myapp.model;
    // packages NOT exported are hidden, even if their classes are public!
}
```

### 2.1 The key directives
| Directive | Meaning |
|-----------|---------|
| **`requires X`** | This module depends on module X (X's exported packages become available) |
| **`requires transitive X`** | Also re-export X to *your* consumers (they get X without requiring it themselves) |
| **`requires static X`** | Compile-time only (optional at runtime) |
| **`exports P`** | Make package P accessible to all other modules |
| **`exports P to M`** | Qualified export — only module M can access P |
| **`opens P`** | Allow **reflective** access to P at runtime (for frameworks) |
| **`opens P to M`** | Qualified reflective access |
| **`uses S` / `provides S with Impl`** | Service-loader (SPI) declarations |

### 2.2 exports vs opens (important distinction)
- **`exports`** → allows **compile-time + normal** access to a package's public types.
- **`opens`** → allows **reflective** access (including to private members via `setAccessible`) — needed by frameworks like Spring, Hibernate, Jackson that use reflection (Phase 1.14).
```java
exports com.example.api;        // others can call its public API normally
opens   com.example.model;       // frameworks can reflect on it (e.g., JPA/Jackson)
```
> Strong encapsulation **blocks reflection** by default — so frameworks that reflect into your classes need the package `opens`ed (or the whole module `open`). This is a common JPMS friction point with Spring/Hibernate.

---

## 3. Strong Encapsulation — The Core Benefit

> **In a module, `public` no longer means "visible to everyone."** A public class in a **non-exported** package is **invisible** outside the module.

```
module com.example.app {
    exports com.example.app.api;       // api package -> accessible
    // com.example.app.internal is NOT exported -> HIDDEN
}
```
- `com.example.app.api.Service` → usable by other modules.
- `com.example.app.internal.Helper` (even if `public`) → **inaccessible** outside the module.

This finally enforces the "internal vs public API" boundary that packages alone couldn't (recall access modifiers, Phase 1.3 — JPMS adds the module layer on top: a type is accessible only if it's `public` **and** in an **exported** package **and** the consumer **requires** your module).

---

## 4. Module Path vs Classpath

| | **Classpath** (`-cp`) | **Module path** (`--module-path` / `-p`) |
|---|----------------------|------------------------------------------|
| Era | Original (pre-Java 9) | Java 9+ |
| Encapsulation | None (`public` = open) | Strong (exports only) |
| Dependencies | Implicit, runtime-discovered | Explicit (`requires`), startup-validated |
| Failure mode | `NoClassDefFoundError` at runtime | Missing module error at startup |
| JARs become | Plain JARs | Modules (named/automatic) |

```bash
# Classpath (traditional):
java -cp app.jar:libs/* com.example.Main

# Module path:
java --module-path mods --module com.example.app/com.example.Main
```
> Both can coexist. The **classpath still works** and is what most apps use. The module path enables JPMS features but requires modularized dependencies (many libraries still aren't fully modular).

---

## 5. Module Types: Named, Automatic, Unnamed

JPMS has to bridge the old (classpath) and new (modules) worlds, so there are three kinds of modules:

### 5.1 Named modules
A "real" module with an explicit `module-info.java` (like §2). Full encapsulation and explicit dependencies.

### 5.2 Automatic modules
A **plain JAR (no `module-info.java`) placed on the module path** is treated as an **automatic module**:
- Its name is derived from the JAR filename (or `Automatic-Module-Name` in its manifest).
- It **exports all** its packages and **requires all** other modules → a bridge for not-yet-modularized libraries.
> Automatic modules let you use legacy JARs on the module path while migrating. They're a transitional mechanism.

### 5.3 The unnamed module
**Everything on the classpath** belongs to the single **unnamed module**:
- It can read **all** modules and exports **all** its packages.
- Named modules **cannot** depend on the unnamed module (you can't `requires` the classpath).
> This is why pure classpath apps "just work" — they live in the unnamed module with no JPMS restrictions. It's the backward-compatibility escape hatch.

| Module type | Has module-info? | Where | Encapsulation |
|-------------|------------------|-------|---------------|
| **Named** | Yes | Module path | Strong (exports only) |
| **Automatic** | No (plain JAR) | Module path | Exports everything (bridge) |
| **Unnamed** | No | **Classpath** | None (legacy behavior) |

---

## 6. Tooling: jlink & jdeps

- **`jlink`:** create a **custom, minimal runtime image** containing only the modules your app needs → small, self-contained distributions (great for containers — smaller images, Phase 10).
- **`jdeps`:** analyze a project's dependencies and module relationships (helps migration).
```bash
jlink --add-modules com.example.app --output myruntime   # custom slim JRE
jdeps --module-path mods app.jar                         # dependency analysis
```
> `jlink` is one of JPMS's most practical wins: ship a ~40MB runtime instead of a full ~200MB JDK.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Thinking `public` = accessible across modules | Must also be in an **exported** package + consumer `requires` it |
| Frameworks failing with reflection errors | `opens` the package (or use an `open module`) |
| Trying to `requires` the classpath/unnamed module | Not allowed; named modules can't depend on it |
| Assuming all apps must be modularized | Classpath still works; most apps use it |
| Forgetting tons of libraries aren't modular | Automatic modules / classpath bridge the gap |
| Confusing `exports` and `opens` | exports = normal access; opens = reflective access |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Reality:** most **Spring Boot apps run on the classpath** (unnamed module) — you'll rarely write `module-info.java`. But understanding JPMS explains some reflection/`opens` errors and why the JDK is modular.
- **`jlink` slim runtimes** reduce Docker image size (Phase 10) — a practical benefit.
- **Reflection (Phase 1.14):** Spring/Hibernate/Jackson reflect into your classes; under JPMS that requires `opens` — a common migration hurdle.
- **Modular monoliths (Phase 14):** the *concept* of strong module boundaries with explicit internal APIs is valuable even without JPMS (often enforced via packages + ArchUnit, Phase 6.7).
- **GraalVM native image (Phase 16.5)** interacts with module/reflection configuration.

---

## 9. Quick Self-Check Questions

1. What does a module add on top of packages?
2. What two problems with the classpath does JPMS solve?
3. What do `requires`, `exports`, and `opens` do? How do exports and opens differ?
4. Under JPMS, does `public` guarantee cross-module access? What else is required?
5. What's the difference between the classpath and the module path?
6. Explain named, automatic, and unnamed modules.
7. Why might a framework using reflection fail under JPMS, and how do you fix it?
8. What are `jlink` and `jdeps` for?

---

## 10. Key Terms Glossary

- **JPMS / Java Module System:** module layer above packages (Java 9+).
- **Module:** a named group of packages with explicit dependencies & exports.
- **`module-info.java`:** the module descriptor.
- **`requires` / `requires transitive` / `requires static`:** dependency declarations.
- **`exports` / `exports ... to`:** make a package accessible (qualified).
- **`opens` / `open module`:** allow reflective access.
- **Strong encapsulation:** public types in non-exported packages are hidden.
- **Module path vs classpath:** JPMS-aware vs legacy.
- **Named / automatic / unnamed module:** explicit module / bridged JAR / classpath.
- **`jlink` / `jdeps`:** custom runtime images / dependency analysis.

---

*This is the note for **Section 1.12 — Java Modules (JPMS)**.*
*Next section in roadmap: **1.13 Annotations**.*
