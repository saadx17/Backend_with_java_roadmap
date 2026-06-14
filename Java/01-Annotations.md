# Annotations

> **Phase 1 — Java Language Mastery → 1.13 Annotations**
> Goal: Master Java annotations — built-in annotations, meta-annotations, retention policies, custom annotations, and annotation processors. Annotations power Spring, JPA, validation, and most modern Java frameworks.

---

## 0. The Big Picture

An **annotation** is **metadata** attached to code (classes, methods, fields, parameters) that doesn't change behavior by itself but can be **read** by the compiler, tools, or frameworks (at compile time or runtime via reflection — Phase 1.14) to drive behavior.

```java
@Override                       // metadata: "this method overrides a parent's"
public String toString() { ... }

@Entity @Table(name = "users")  // metadata: "map this class to the users table"
public class User { ... }
```

> Annotations are **everywhere** in modern Java/Spring (`@RestController`, `@Autowired`, `@Transactional`, `@Entity`, `@Test`). Understanding how they work demystifies all framework "magic." This note is foundational for nearly every later phase.

---

## 1. What Annotations Are (and Aren't)

- An annotation is **declarative metadata** — it *describes* code; it doesn't *do* anything on its own.
- Something must **process** the annotation to give it meaning: the **compiler** (`@Override`), an **annotation processor** (compile-time codegen, §6), or a **framework via reflection** at runtime (Spring scanning for `@Component`).
```
@Component on a class -> Spring SCANS for it at startup -> creates & manages a bean
(the annotation alone does nothing; Spring's processing gives it meaning)
```

---

## 2. Built-in Annotations

Java provides several standard annotations in `java.lang`:
| Annotation | Purpose |
|------------|---------|
| **`@Override`** | Asserts a method overrides a supertype method (compiler checks it — recall Phase 1.3) |
| **`@Deprecated`** | Marks an API as obsolete; the compiler warns on use (`@Deprecated(since="2.0", forRemoval=true)`) |
| **`@SuppressWarnings`** | Tell the compiler to suppress specific warnings (`@SuppressWarnings("unchecked")` — recall Phase 1.7) |
| **`@FunctionalInterface`** | Enforce that an interface has exactly one abstract method (recall Phase 1.3/1.8) |
| **`@SafeVarargs`** | Assert a generic varargs method is heap-pollution-safe (recall Phase 1.7) |

```java
@Override public boolean equals(Object o) { ... }   // catches signature typos
@Deprecated public void oldMethod() { ... }          // warns callers
@SuppressWarnings("unchecked") List<String> l = (List<String>) raw;
```

---

## 3. Meta-Annotations (Annotations on Annotations)

**Meta-annotations** configure how custom annotations behave. They're in `java.lang.annotation`:
| Meta-annotation | Controls |
|-----------------|----------|
| **`@Retention`** | **How long** the annotation is kept (SOURCE / CLASS / RUNTIME) — §4 |
| **`@Target`** | **Where** it can be applied (TYPE, METHOD, FIELD, PARAMETER, ...) |
| **`@Documented`** | Include it in generated Javadoc |
| **`@Inherited`** | Subclasses inherit the annotation from their superclass |
| **`@Repeatable`** | Allow the annotation to be applied multiple times to the same element |

```java
@Retention(RetentionPolicy.RUNTIME)     // available at runtime via reflection
@Target(ElementType.METHOD)             // only on methods
@Documented
public @interface Loggable { }
```

### 3.1 @Target — where an annotation may go
```java
@Target({ElementType.TYPE, ElementType.METHOD})   // on classes AND methods
```
| `ElementType` | Applies to |
|---------------|-----------|
| `TYPE` | Classes, interfaces, enums |
| `METHOD` | Methods |
| `FIELD` | Fields |
| `PARAMETER` | Method parameters |
| `CONSTRUCTOR`, `LOCAL_VARIABLE`, `ANNOTATION_TYPE`, `PACKAGE`, `TYPE_PARAMETER`, `TYPE_USE` | (others) |

---

## 4. Retention Policies (Critical Concept)

**`@Retention`** decides **how long** an annotation survives — this determines *who can read it*:
| Policy | Kept until | Readable by | Example |
|--------|-----------|-------------|---------|
| **`SOURCE`** | Discarded after compilation | Compiler/tools only | `@Override`, `@SuppressWarnings` |
| **`CLASS`** | In the `.class` file, **not** loaded at runtime | Bytecode tools | (default) |
| **`RUNTIME`** | Available at **runtime** via reflection | Frameworks (Spring, JUnit) | `@Test`, `@Component`, `@Entity` |

```java
@Retention(RetentionPolicy.RUNTIME)   // MUST be RUNTIME for reflection-based frameworks
public @interface MyAnnotation { }
```
> ⚠️ **For a framework to read your annotation at runtime (via reflection), it MUST be `@Retention(RUNTIME)`.** This is the #1 gotcha when writing custom annotations for Spring-style processing — `SOURCE`/`CLASS` annotations are invisible at runtime. (`SOURCE` is for compile-time-only things like `@Override`.)

---

## 5. Custom Annotations

You define an annotation with `@interface`. Its "methods" become **elements** (attributes):
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RateLimit {
    int requests() default 100;            // element with a default
    int perSeconds() default 60;
    String key();                          // required element (no default)
}
```
Usage:
```java
@RateLimit(requests = 10, perSeconds = 1, key = "login")
public void login() { ... }

@RateLimit(key = "search")   // uses defaults for requests/perSeconds
public void search() { ... }
```

### 5.1 Element rules
- Elements look like methods but define **attributes**; types allowed: primitives, `String`, `Class`, enums, other annotations, and arrays of these.
- `default` provides a default value (then the element is optional).
- A single element named **`value`** can be set without naming it: `@MyAnn("x")` ≡ `@MyAnn(value="x")`.
- An annotation with no elements is a **marker annotation** (like a marker interface — recall Phase 1.3).

### 5.2 Reading a custom annotation (reflection — Phase 1.14)
The annotation does nothing until code reads it. At runtime:
```java
Method method = obj.getClass().getMethod("login");
if (method.isAnnotationPresent(RateLimit.class)) {
    RateLimit rl = method.getAnnotation(RateLimit.class);
    int limit = rl.requests();           // read the metadata
    // ... enforce the rate limit ...
}
```
> This read-via-reflection pattern is **exactly how Spring works**: it scans classes/methods for annotations and acts on them (creating beans, wrapping in proxies for `@Transactional`, etc.). (Full reflection coverage in Phase 1.14.)

---

## 6. Annotation Processors (Compile-Time)

An **annotation processor** runs at **compile time** to read annotations and **generate code**, validate, or produce metadata — *before* the program runs. This avoids runtime reflection overhead.
| Tool/use | What it does |
|----------|--------------|
| **Lombok** | Generates getters/setters/builders/`equals`/`hashCode` from `@Data`, `@Builder`, etc. |
| **MapStruct** | Generates DTO↔entity mappers from `@Mapper` (Phase 7.3) |
| **Hibernate JPA metamodel** | Generates type-safe query classes |
| **Dagger** | Compile-time dependency injection |
| Your own | Implement `javax.annotation.processing.Processor` |

```java
// Lombok — @Data generates getters/setters/toString/equals/hashCode at compile time:
@Data
public class User { private String name; private int age; }
```
> **Compile-time processing** (annotation processors / code generation) is faster at runtime than **reflection** (no runtime cost) and catches errors at compile time. Modern frameworks increasingly prefer it (MapStruct, Dagger, Spring's AOT — Phase 16.5).

### 6.1 Compile-time vs runtime annotation processing
| | Compile-time (processors) | Runtime (reflection) |
|---|---------------------------|----------------------|
| When | During `javac` | While the app runs |
| Retention needed | SOURCE/CLASS | **RUNTIME** |
| Cost at runtime | None (code generated) | Reflection overhead |
| Examples | Lombok, MapStruct, Dagger | Spring DI, JUnit, Jackson |

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Custom annotation invisible to a framework | Use `@Retention(RUNTIME)` |
| Expecting an annotation to "do" something alone | It's metadata — something must process it |
| Wrong `@Target` (e.g., method annotation on a field) | Match `@Target` to where you apply it |
| Forgetting defaults make elements optional | Add `default` for optional elements |
| Confusing SOURCE/CLASS/RUNTIME retention | RUNTIME for reflection; SOURCE for compile-only |
| Overusing reflection when codegen fits | Prefer annotation processors (Lombok/MapStruct) for performance |
| Marker annotation vs marker interface confusion | Annotation = metadata; interface = a type (Phase 1.3) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Spring is built on annotations:** `@Component`/`@Service`/`@Repository`/`@Controller`, `@Autowired`, `@Configuration`/`@Bean`, `@Transactional`, `@RequestMapping` — all RUNTIME-retained, read via reflection (Phase 5).
- **JPA/Hibernate:** `@Entity`, `@Table`, `@Id`, `@Column`, `@OneToMany` map objects to tables (Phase 5.4).
- **Validation (JSR 380):** `@NotNull`, `@Size`, `@Email`, plus **custom validators** via custom annotations + `ConstraintValidator` (Phase 5.3).
- **Testing:** `@Test`, `@BeforeEach`, `@Mock` (Phase 6).
- **Custom annotations** for cross-cutting concerns (rate limiting, auditing, `@Loggable`) processed via Spring AOP (Phase 5.1).
- **Lombok/MapStruct** (compile-time processors) cut boilerplate in DTOs/mappers (Phase 7).
- **Spring AOT/native (Phase 16.5)** shifts annotation processing to compile time for faster startup.

---

## 9. Quick Self-Check Questions

1. What is an annotation, and what does it do on its own?
2. Name four built-in annotations and their purposes.
3. What are meta-annotations? What do `@Retention` and `@Target` control?
4. What are the three retention policies, and which is required for runtime/reflection-based frameworks?
5. How do you define a custom annotation with a required and an optional element?
6. What is a marker annotation?
7. How does a framework read a custom annotation at runtime?
8. What is an annotation processor, and how does compile-time processing differ from runtime reflection?

---

## 10. Key Terms Glossary

- **Annotation:** declarative metadata attached to code.
- **`@interface`:** keyword to define an annotation.
- **Element:** an annotation attribute (looks like a method).
- **Marker annotation:** an annotation with no elements.
- **Built-in annotations:** `@Override`, `@Deprecated`, `@SuppressWarnings`, etc.
- **Meta-annotation:** an annotation that configures other annotations.
- **`@Retention` (SOURCE/CLASS/RUNTIME):** how long an annotation is kept.
- **`@Target`:** where an annotation may be applied.
- **`@Inherited` / `@Repeatable` / `@Documented`:** inheritance / repetition / Javadoc.
- **Annotation processor:** compile-time tool that reads annotations (codegen/validation).
- **Reflection-based processing:** reading annotations at runtime (Spring, JUnit).

---

*This is the note for **Section 1.13 — Annotations**.*
*Next section in roadmap: **1.14 Reflection API**.*
