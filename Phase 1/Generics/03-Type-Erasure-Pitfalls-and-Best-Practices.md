# Type Erasure, Pitfalls & Best Practices

> **Phase 1 — Java Language Mastery → 1.7 Generics**
> Goal: Understand type erasure (what happens at compile vs runtime), reifiable vs non-reifiable types, bridge methods, generic array pitfalls, heap pollution, and why to avoid raw types.

---

## 0. The Big Picture

Java generics are a **compile-time** feature. At runtime, the type information is largely **erased** — this is **type erasure**. Understanding it explains nearly every "weird" thing about generics: why you can't do `new T[]`, why `List<String>` and `List<Integer>` are the "same" class at runtime, and why some operations are forbidden.

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
a.getClass() == b.getClass();   // TRUE! — both are just ArrayList at runtime
```

> Generics exist for the **compiler** to check types; the **JVM** mostly doesn't see them. This design kept Java backward-compatible when generics were added in Java 5.

---

## 1. Type Erasure — What Happens

The compiler uses generics to **check types**, then **erases** them, replacing type parameters with their **bounds** (or `Object` if unbounded) and inserting casts where needed.

### 1.1 What the compiler does
```java
// What you write:
class Box<T> {
    private T value;
    public T get() { return value; }
    public void set(T value) { this.value = value; }
}

// What the JVM sees after erasure (roughly):
class Box {
    private Object value;                 // T -> Object (unbounded)
    public Object get() { return value; }
    public void set(Object value) { this.value = value; }
}
```
For a **bounded** type parameter, T is erased to its **bound**:
```java
class Stats<T extends Number> { ... }    // T -> Number after erasure
```

### 1.2 Casts are inserted automatically
```java
// You write:
Box<String> box = new Box<>();
String s = box.get();

// Compiler generates:
String s = (String) box.get();   // the cast you'd have written manually pre-generics
```
> So generics give you **compile-time safety + automatic casts**, but at runtime it's the same old `Object` + casts. The safety is enforced *before* erasure.

---

## 2. Compile-Time vs Runtime

| | Compile time | Runtime |
|---|--------------|---------|
| Type info present? | ✅ Full generic types | ❌ Erased (mostly) |
| Type checking | ✅ Enforced by compiler | ❌ Not checked |
| `List<String>` vs `List<Integer>` | Different types | **Same class** (`ArrayList`) |

```java
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();
System.out.println(strings.getClass() == ints.getClass());  // true
```

---

## 3. Reifiable vs Non-Reifiable Types

- **Reifiable type:** a type whose full info **is available at runtime** (not erased) — e.g., `String`, `int[]`, `List` (raw), `List<?>`.
- **Non-reifiable type:** a type whose info is **erased** at runtime — e.g., `List<String>`, `Map<K,V>`, `T`.

### 3.1 Consequences (things you CAN'T do)
Because generic type info is erased, several operations are **forbidden**:
```java
// ❌ Can't use instanceof with a parameterized type:
if (obj instanceof List<String>) { }   // COMPILE ERROR
if (obj instanceof List<?>) { }         // OK (unbounded wildcard is reifiable)
if (obj instanceof List) { }            // OK (raw, reifiable)

// ❌ Can't create an instance of a type parameter:
T t = new T();                          // COMPILE ERROR (T is erased)

// ❌ Can't create a generic array directly (see §5):
T[] arr = new T[10];                    // COMPILE ERROR

// ❌ Can't use primitives as type arguments:
List<int> list;                         // COMPILE ERROR — use List<Integer>

// ❌ Static fields can't use the class's type parameter:
class Box<T> { static T value; }        // COMPILE ERROR (T is per-instance)
```

---

## 4. Bridge Methods (awareness)

When a generic class is extended/overridden, the compiler sometimes generates synthetic **bridge methods** to preserve polymorphism after erasure.
```java
class Node<T> {
    T data;
    void set(T data) { this.data = data; }
}
class StringNode extends Node<String> {
    @Override void set(String data) { ... }   // overrides set(String)
}
// After erasure, Node.set takes Object. The compiler adds a BRIDGE method in
// StringNode: void set(Object data) { set((String) data); }  -> delegates to set(String)
```
> You rarely write or see these directly, but they explain odd entries in stack traces / reflection (`isBridge()`). They exist to make overriding work correctly across erasure.

---

## 5. Generic Arrays Pitfalls

> **You cannot create a generic array** (`new T[]` or `new List<String>[]`). This is one of the most surprising restrictions.

### 5.1 Why generic arrays are forbidden
Arrays are **covariant and reified** (they know their element type at runtime); generics are **invariant and erased**. Mixing them would break type safety:
```java
// If this were allowed (it's NOT):
List<String>[] arr = new List<String>[10];   // COMPILE ERROR
Object[] objs = arr;                          // arrays are covariant
objs[0] = List.of(1, 2, 3);                   // a List<Integer> — no runtime check!
String s = arr[0].get(0);                     // ClassCastException — type safety broken
```
Arrays check element types at runtime (`ArrayStoreException`), but erasure means the generic element type can't be checked → the language forbids it outright.

### 5.2 Workarounds
```java
// Use a List instead of a generic array (preferred):
List<List<String>> list = new ArrayList<>();

// Or an array of the raw/unbounded type with a cast (with @SuppressWarnings):
@SuppressWarnings("unchecked")
List<String>[] arr = (List<String>[]) new List[10];   // unchecked but works

// Generic methods needing an array often take a Class<T> or use reflection:
@SuppressWarnings("unchecked")
static <T> T[] newArray(Class<T> type, int size) {
    return (T[]) java.lang.reflect.Array.newInstance(type, size);
}
```
> **Prefer collections (`List<T>`) over arrays when generics are involved** — it sidesteps the whole problem (*Effective Java* Item 28: "prefer lists to arrays").

---

## 6. Heap Pollution

**Heap pollution** occurs when a variable of a parameterized type refers to an object that is **not** of that type — typically via unchecked casts or raw types. It can cause `ClassCastException` later, far from the cause.
```java
List<String> strings = new ArrayList<>();
List raw = strings;          // raw type — bypasses checking
raw.add(42);                 // heap pollution! an Integer in a List<String>
String s = strings.get(0);   // ClassCastException — but here, not where the bug was
```

### 6.1 Varargs + generics: `@SafeVarargs`
Generic varargs methods can cause heap pollution (varargs create an array, which conflicts with erasure). The compiler warns; if the method is genuinely safe, annotate it:
```java
@SafeVarargs
static <T> List<T> listOf(T... items) {   // safe: only reads items
    return new ArrayList<>(Arrays.asList(items));
}
```
> `@SafeVarargs` asserts "this generic varargs method doesn't pollute the heap." Only use it when the method truly is safe (doesn't store into or expose the varargs array).

---

## 7. Raw Types — Why to Avoid Them

A **raw type** is a generic type used **without** type arguments (`List` instead of `List<String>`). They exist only for **backward compatibility** with pre-generics code. **Avoid them.**

### 7.1 Why raw types are dangerous
```java
List raw = new ArrayList();   // raw type — no type safety
raw.add("hello");
raw.add(42);                   // compiles! no checking
String s = (String) raw.get(1);  // ClassCastException at runtime
```
- Raw types **disable** all generic type checking → you lose the entire benefit of generics.
- Using a raw type also disables generics for **other** operations on it (not just the unparameterized one).

### 7.2 Raw type vs `<?>`
| | Raw type `List` | `List<?>` |
|---|-----------------|-----------|
| Type-safe? | ❌ No (anything goes) | ✅ Yes (compiler enforces) |
| Can add elements? | Yes (unsafe) | No (except null) — safe |
| Use? | Avoid (legacy only) | When type is unknown but you want safety |

> **Always parameterize.** If you don't care about the type, use `<?>` (the unbounded wildcard), **not** the raw type — it keeps type safety.

---

## 8. Best Practices Summary

```
[ ] Always use parameterized types — never raw types
[ ] Use <?> when the type is unknown (not the raw type)
[ ] Apply PECS for flexible APIs (producer extends, consumer super)
[ ] Prefer List<T> over generic arrays
[ ] Add bounds (<T extends X>) only when you need X's behavior
[ ] Minimize @SuppressWarnings("unchecked"); scope it tightly & comment why
[ ] Use @SafeVarargs only when the method is genuinely safe
[ ] Let type inference (diamond/var) reduce verbosity
[ ] Remember: type checks happen at COMPILE time; nothing at runtime
```

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Expecting generic types at runtime (`instanceof List<String>`) | Erased — use `List<?>` or pass `Class<T>` |
| `new T[]` / `new List<String>[]` | Use `List<T>` / a cast with `@SuppressWarnings` |
| Using raw types | Always parameterize; use `<?>` if unknown |
| `new T()` inside a generic class | Pass a `Supplier<T>` or `Class<T>` and use reflection |
| Static field of type `T` | Not allowed (T is per-instance) |
| Heap pollution via raw types/unchecked casts | Avoid raw types; verify casts |
| Blanket `@SuppressWarnings("unchecked")` | Scope it to the smallest block, comment the reason |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Reflection & frameworks (Phase 1.14, 5):** because generics are erased, frameworks (Jackson, Spring) need `Class<T>`, `TypeReference`, or `ParameterizedTypeReference` to recover generic types at runtime (e.g., deserializing `List<User>` from JSON).
- **`ResponseEntity<List<Dto>>` / `ParameterizedTypeReference`** in `RestTemplate`/`WebClient` exist precisely to work around erasure (Phase 5.9).
- **Avoiding raw types** keeps Spring/JPA code type-safe.
- **Prefer lists over arrays** applies throughout entity/DTO modeling.
- **Generic repository/service base classes** use `Class<T>` to instantiate or query by type (Phase 5.4), a direct consequence of erasure.

---

## 11. Quick Self-Check Questions

1. What is type erasure, and why does Java do it?
2. What is a type parameter erased to (unbounded vs bounded)?
3. Why do `List<String>` and `List<Integer>` have the same runtime class?
4. What's the difference between reifiable and non-reifiable types? Give examples.
5. Why can't you write `new T[10]` or `instanceof List<String>`?
6. What is a bridge method and why does the compiler generate one?
7. Why are generic arrays forbidden, and what should you use instead?
8. What is heap pollution, and when is `@SafeVarargs` appropriate?
9. Why are raw types dangerous, and what should you use instead?

---

## 12. Key Terms Glossary

- **Type erasure:** removal of generic type info at compile time.
- **Reifiable type:** type whose info exists at runtime (`String`, `List<?>`).
- **Non-reifiable type:** type whose info is erased (`List<String>`, `T`).
- **Bridge method:** synthetic method generated to preserve overriding after erasure.
- **Generic array:** `new T[]` — forbidden because of erasure/covariance conflict.
- **Heap pollution:** a parameterized variable referencing an incompatible object.
- **`@SafeVarargs`:** asserts a generic varargs method doesn't pollute the heap.
- **Raw type:** a generic type used without type arguments (avoid).
- **`@SuppressWarnings("unchecked")`:** silences unchecked-cast warnings (use sparingly).
- **`ParameterizedTypeReference` / `Class<T>`:** runtime helpers to recover erased types.

---

*Previous topic: **Wildcards & the PECS Principle**.*
*This completes **Section 1.7 — Generics**.*
*Next section in roadmap: **1.8 Functional Programming in Java**.*
