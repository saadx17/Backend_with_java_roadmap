# Optional&lt;T&gt;

> **Phase 1 — Java Language Mastery → 1.8 Functional Programming**
> Goal: Master `Optional<T>` — creating, extracting, transforming, and checking it — and learn when to use it (and the anti-patterns to avoid).

---

## 0. The Big Picture

**`Optional<T>`** is a container that may or may not hold a value. It exists to make **"absence of a value" explicit** in the type system — a deliberate alternative to returning `null` and the dreaded `NullPointerException` (recall Phase 1.5).

```java
Optional<User> user = userRepository.findById(42);   // might find one, might not
user.ifPresent(u -> System.out.println(u.getName())); // do something only if present
String name = user.map(User::getName).orElse("Unknown");  // safe extraction with default
```

> `Optional` says, in the type signature, **"this might be empty — handle it."** It doesn't *eliminate* nulls, but it forces callers to consider absence, reducing NPEs.

---

## 1. The Problem Optional Solves

Returning `null` is implicit and dangerous — callers forget to check:
```java
User user = findUser(id);        // returns null if not found
user.getName();                  // NullPointerException if null! (and the type didn't warn you)
```
With `Optional`, the **type itself** signals possible absence, and the API guides you to handle it:
```java
Optional<User> user = findUser(id);
user.map(User::getName).orElse("Unknown");   // can't accidentally dereference
```

---

## 2. Creating an Optional

| Method | Use |
|--------|-----|
| `Optional.of(value)` | Wrap a **non-null** value (throws NPE if null!) |
| `Optional.ofNullable(value)` | Wrap a value that **might be null** → empty if null |
| `Optional.empty()` | An explicitly empty Optional |

```java
Optional<String> a = Optional.of("hello");          // value present
Optional<String> b = Optional.ofNullable(maybeNull); // present or empty
Optional<String> c = Optional.empty();               // empty
// Optional.of(null);   // NullPointerException! — use ofNullable for nullable values
```
> Use `of` only when you're certain the value is non-null; use `ofNullable` when it might be null.

---

## 3. Checking Presence

```java
Optional<User> opt = findUser(id);

opt.isPresent();    // true if a value is present
opt.isEmpty();      // true if empty (Java 11+)

// Conditional actions (preferred over isPresent + get):
opt.ifPresent(u -> System.out.println(u.getName()));   // run only if present
opt.ifPresentOrElse(                                    // Java 9+
    u -> System.out.println("Found: " + u.getName()),
    () -> System.out.println("Not found")               // else branch
);
```

---

## 4. Extracting the Value

| Method | If present | If empty |
|--------|-----------|----------|
| `get()` | returns value | **throws `NoSuchElementException`** ⚠️ |
| `orElse(default)` | returns value | returns the default |
| `orElseGet(supplier)` | returns value | returns `supplier.get()` (lazy) |
| `orElseThrow()` | returns value | throws `NoSuchElementException` (Java 10+) |
| `orElseThrow(supplier)` | returns value | throws your custom exception |

```java
String name = opt.map(User::getName).orElse("Unknown");           // default value
User u = opt.orElseThrow(() -> new UserNotFoundException(id));     // custom exception
User u2 = opt.orElseGet(() -> createDefaultUser());                // lazy default
```

### 4.1 `orElse` vs `orElseGet` (subtle but important)
```java
opt.orElse(expensiveDefault());      // expensiveDefault() ALWAYS runs (even if present!)
opt.orElseGet(() -> expensiveDefault());  // runs ONLY if empty (lazy)
```
> ⚠️ **`orElse` always evaluates its argument**, even when the Optional has a value. If the default is expensive (DB call, object creation), use **`orElseGet`** (a `Supplier`, recall Phase 1.8.1) so it's only computed when needed.

### 4.2 Avoid `get()`
> ⚠️ **`get()` is an anti-pattern** — it throws if empty, defeating Optional's purpose (it's just a disguised null check). Prefer `orElse`/`orElseGet`/`orElseThrow`/`map`/`ifPresent`. Modern Java even discourages `get()`.

---

## 5. Transforming an Optional (the functional part)

`Optional` has stream-like methods so you can chain transformations safely:
| Method | Purpose |
|--------|---------|
| `map(fn)` | Transform the value if present (returns `Optional<R>`) |
| `flatMap(fn)` | Like map, but for functions returning `Optional` (avoids nesting) |
| `filter(predicate)` | Keep the value only if it matches (else empty) |

```java
// map — transform if present:
Optional<String> name = findUser(id).map(User::getName);   // Optional<User> -> Optional<String>

// Chain multiple transforms:
String city = findUser(id)
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("Unknown");

// filter — conditional:
Optional<User> adult = findUser(id).filter(u -> u.getAge() >= 18);
```

### 5.1 `map` vs `flatMap`
When the mapping function **itself returns an `Optional`**, `map` would give you `Optional<Optional<X>>` (nested). **`flatMap`** flattens it:
```java
// getAddress() returns Optional<Address>:
Optional<Optional<Address>> nested = findUser(id).map(User::getAddressOptional);  // ugly!
Optional<Address> flat = findUser(id).flatMap(User::getAddressOptional);          // flattened
```
> Same idea as `Stream.flatMap` (Phase 1.8.3): use `flatMap` when the function returns an `Optional`/stream you want unwrapped.

---

## 6. Putting It Together (idiomatic chains)

```java
// Find user -> get email -> validate -> default; all null-safe:
String email = userRepository.findById(id)
    .map(User::getEmail)
    .filter(e -> e.contains("@"))
    .orElse("no-email@example.com");

// Find or throw a domain exception (common in service layers):
User user = userRepository.findById(id)
    .orElseThrow(() -> new ResourceNotFoundException("User " + id));

// Optional + Stream: collect present values
List<String> names = userIds.stream()
    .map(this::findUser)            // Stream<Optional<User>>
    .flatMap(Optional::stream)      // Java 9+: drop empties -> Stream<User>
    .map(User::getName)
    .toList();
```

---

## 7. When to Use Optional (and When NOT to)

### 7.1 ✅ Use Optional for...
- **Return types** of methods that may legitimately return "nothing" (`findById`, lookups, searches).
- Making "no result" **explicit** so callers handle it.

### 7.2 ❌ Do NOT use Optional for...
| Anti-pattern | Why it's wrong |
|--------------|----------------|
| **Fields** | Adds overhead, not serializable-friendly; use null + validation |
| **Method parameters** | Forces callers to wrap; use overloads or nullable params |
| **Collections** | Return an **empty collection**, not `Optional<List>` |
| Calling `.get()` without checking | Defeats the purpose (NPE-equivalent) |
| `Optional.ofNullable(x).orElse(y)` instead of a simple null check | Over-engineering |
| Wrapping then immediately `isPresent()/get()` | Just use the value/null directly |

```java
// BAD: Optional as a field
class User { private Optional<String> name; }   // don't — use null + getter logic
// BAD: Optional parameter
void process(Optional<String> name) {}          // don't — overload or accept nullable
// BAD: Optional of a collection
Optional<List<User>> findUsers();               // return empty List instead
```

> **Rule of thumb (from the JDK designers):** `Optional` is intended primarily as a **return type** to signal "a result may be absent." Don't use it for fields, parameters, or collections.

---

## 8. Optional and Primitives

To avoid boxing, there are primitive variants (returned by primitive streams, Phase 1.8.3):
- `OptionalInt`, `OptionalLong`, `OptionalDouble`
```java
OptionalInt max = IntStream.of(3, 1, 2).max();
int result = max.orElse(0);
```

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Calling `get()` without checking | Use `orElse`/`orElseThrow`/`map` |
| `Optional.of(nullableValue)` | Use `Optional.ofNullable` |
| `orElse(expensive())` always runs | Use `orElseGet(() -> expensive())` |
| `Optional` as a field/parameter | Use null + validation / overloads |
| `Optional<List<T>>` | Return an empty list |
| `isPresent()` + `get()` everywhere | Use `ifPresent`/`map`/`orElse` (declarative) |
| Thinking Optional eliminates all nulls | It signals absence; you still avoid nulls elsewhere |
| Nesting `Optional<Optional<T>>` | Use `flatMap` |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Spring Data repositories** return `Optional<T>` from `findById` — the canonical use: `findById(id).orElseThrow(() -> new NotFoundException())` (Phase 5.4).
- **Service layer:** map missing entities to domain exceptions → mapped to `404` by `@ExceptionHandler` (Phase 5.3).
- **Null-safety** reduces NPEs in request handling and DTO mapping (Phase 7).
- **Streams + Optional:** `flatMap(Optional::stream)` to filter present values (Phase 1.8.3).
- **`orElseGet` for lazy defaults** (e.g., config fallbacks, expensive lookups) — Phase 5.2.
- Reactive equivalent: `Mono<T>` (0 or 1 element) plays a similar role (Phase 16.1).

---

## 11. Quick Self-Check Questions

1. What problem does `Optional` solve, and what does it make explicit?
2. What's the difference between `Optional.of`, `ofNullable`, and `empty`?
3. Why is calling `get()` discouraged?
4. What's the difference between `orElse` and `orElseGet`, and when does it matter?
5. What do `map`, `flatMap`, and `filter` do on an Optional?
6. When would you use `flatMap` instead of `map`?
7. List three places you should NOT use Optional.
8. How do you collect only the present values from a `Stream<Optional<T>>`?

---

## 12. Key Terms Glossary

- **`Optional<T>`:** a container that may or may not hold a value.
- **`of` / `ofNullable` / `empty`:** create from non-null / nullable / nothing.
- **`isPresent` / `isEmpty`:** presence checks.
- **`ifPresent` / `ifPresentOrElse`:** run actions conditionally.
- **`get`:** extract value (throws if empty — avoid).
- **`orElse` / `orElseGet` / `orElseThrow`:** value-or-default / lazy default / throw.
- **`map` / `flatMap` / `filter`:** transform / flatten-Optional / conditional.
- **`OptionalInt`/`OptionalLong`/`OptionalDouble`:** primitive variants.
- **Anti-patterns:** Optional fields/parameters/collections, blind `get()`.

---

*Previous topic: **Collectors & Parallel Streams**.*
*This completes **Section 1.8 — Functional Programming in Java**.*
*Next section in roadmap: **1.9 Java I/O**.*
