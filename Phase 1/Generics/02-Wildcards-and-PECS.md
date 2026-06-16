# Wildcards & the PECS Principle

> **Phase 1 — Java Language Mastery → 1.7 Generics**
> Goal: Master wildcards — unbounded `<?>`, upper-bounded `<? extends T>`, lower-bounded `<? super T>` — and the PECS rule (Producer Extends, Consumer Super) for designing flexible generic APIs.

---

## 0. The Big Picture

Wildcards (`?`) make generic APIs **flexible** about which parameterized types they accept. The motivating problem: **generics are invariant** — `List<Integer>` is **not** a subtype of `List<Number>`, even though `Integer` is a subtype of `Number`. Wildcards relax this in controlled ways.

```java
List<Number> nums = new ArrayList<Integer>();   // COMPILE ERROR! generics are invariant
```

> Wildcards are the trickiest part of generics for most people. The **PECS rule** (§5) is the practical takeaway that makes it click.

---

## 1. Why Wildcards: Generics Are Invariant

In Java, **`List<Integer>` is NOT a `List<Number>`** — generic types don't inherit the way their type arguments do:
```java
List<Integer> ints = new ArrayList<>();
// List<Number> nums = ints;     // COMPILE ERROR — invariance
```
### 1.1 Why invariance is necessary (it prevents bugs)
If it *were* allowed, you could break type safety:
```java
List<Number> nums = ints;   // (pretend this compiled)
nums.add(3.14);             // adding a Double into a List<Integer>!
Integer i = ints.get(0);    // BOOM — it's actually a Double -> ClassCastException
```
> Invariance protects you — but it makes generic methods rigid. **Wildcards** restore flexibility safely.

---

## 2. Unbounded Wildcard `<?>`

`<?>` means "a list of **some unknown type**." Use it when you don't care about the type — only about operations that don't depend on it.
```java
public void printSize(List<?> list) {       // accepts List of ANY type
    System.out.println(list.size());        // size() doesn't depend on the element type
}
printSize(List.of("a", "b"));   // OK
printSize(List.of(1, 2, 3));    // OK
```

### 2.1 What you can and can't do with `<?>`
```java
List<?> list = ...;
list.size();          // OK — type-independent
Object o = list.get(0); // OK — read as Object (the only safe assumption)
// list.add("x");     // COMPILE ERROR — can't add (we don't know the type!)
list.add(null);       // the ONLY value you can add (null fits any type)
```
> With `<?>` you can **read as `Object`** and check size, but you **can't add** (except `null`) because the compiler doesn't know what type is safe to insert.

### 2.2 `<?>` vs raw type `List`
`List<?>` is **type-safe** (compiler enforces you can't add wrong things); the raw type `List` is **not** (next note explains why raw types are dangerous). Always prefer `<?>` over raw types.

---

## 3. Upper-Bounded Wildcard `<? extends T>` (Producers)

`<? extends T>` means "some unknown type that **is T or a subtype of T**." Use it to **read** (consume from) a structure as `T`.
```java
public double sum(List<? extends Number> list) {   // accepts List<Integer>, List<Double>, ...
    double total = 0;
    for (Number n : list) {        // safe: every element IS-A Number (read as Number)
        total += n.doubleValue();
    }
    return total;
}
sum(List.of(1, 2, 3));        // List<Integer> — OK!
sum(List.of(1.5, 2.5));       // List<Double>  — OK!
```

### 3.1 The rule for `extends` wildcards
```java
List<? extends Number> list = new ArrayList<Integer>();   // OK now!
Number n = list.get(0);    // READ as Number — safe (it's at least a Number)
// list.add(1);            // COMPILE ERROR — can't ADD (which subtype? unknown)
// list.add(1.0);          // COMPILE ERROR
list.add(null);            // only null
```
> `<? extends T>` lets you **read** elements as `T`, but you **cannot add** (except null) — the compiler doesn't know the exact subtype. This is a **producer**: it *produces* values for you to consume.

---

## 4. Lower-Bounded Wildcard `<? super T>` (Consumers)

`<? super T>` means "some unknown type that **is T or a supertype of T**." Use it to **write** (add T's into) a structure.
```java
public void addNumbers(List<? super Integer> list) {   // accepts List<Integer>, List<Number>, List<Object>
    list.add(1);          // safe: an Integer fits in a List of Integer-or-supertype
    list.add(2);
}
addNumbers(new ArrayList<Integer>());   // OK
addNumbers(new ArrayList<Number>());    // OK
addNumbers(new ArrayList<Object>());    // OK
```

### 4.1 The rule for `super` wildcards
```java
List<? super Integer> list = new ArrayList<Number>();
list.add(1);            // WRITE — safe (Integer fits any supertype-of-Integer list)
list.add(2);
// Integer i = list.get(0);  // COMPILE ERROR — can only read as Object
Object o = list.get(0);  // reading gives Object (the only guaranteed supertype)
```
> `<? super T>` lets you **add** T (and subtypes of T), but reading gives only `Object`. This is a **consumer**: it *consumes* the values you give it.

---

## 5. PECS: Producer Extends, Consumer Super

> **The golden rule of wildcards (from *Effective Java*, Item 31):**
> **PECS — "Producer Extends, Consumer Super."**

| If the parameter... | Use | Because |
|---------------------|-----|---------|
| **Produces** values you read out | `<? extends T>` | You read elements as T (out of the structure) |
| **Consumes** values you put in | `<? super T>` | You write T into the structure |
| Both / neither (just T operations) | exact `T` or `<?>` | — |

### 5.1 The canonical example: `Collections.copy`
```java
public static <T> void copy(List<? super T> dest,    // CONSUMER -> super (we write into dest)
                            List<? extends T> src) {  // PRODUCER -> extends (we read from src)
    for (int i = 0; i < src.size(); i++) {
        dest.set(i, src.get(i));   // read from src (extends), write to dest (super)
    }
}
```
- `src` **produces** elements (we read them) → `? extends T`.
- `dest` **consumes** elements (we write them) → `? super T`.

### 5.2 Another example
```java
// A Stack that can push all elements FROM a producer:
public void pushAll(Collection<? extends E> src) {   // src produces E's -> extends
    for (E e : src) push(e);
}
// ...and pop all elements INTO a consumer:
public void popAll(Collection<? super E> dst) {      // dst consumes E's -> super
    while (!isEmpty()) dst.add(pop());
}
```

### 5.3 Quick mnemonic
```
You READ from it (it gives you values)  -> it's a PRODUCER -> extends
You WRITE to it  (you give it values)   -> it's a CONSUMER -> super
```

---

## 6. Wildcards vs Type Parameters (when to use which)

| Use a **type parameter `<T>`** | Use a **wildcard `<?>`** |
|--------------------------------|---------------------------|
| You need to refer to the type multiple times / relate inputs and outputs | The type is used once and you don't need to name it |
| `<T> T first(List<T> list)` (return relates to input) | `void printAll(List<?> list)` (don't care about the type) |
| Defining a generic class/method | Flexible method parameters |

> Rule of thumb: if a type appears **only once** in a method signature and you don't need to name it, use a **wildcard**; otherwise use a **type parameter**.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Assuming `List<Integer>` is a `List<Number>` | Generics are **invariant**; use wildcards |
| Trying to `add` to a `<? extends T>` list | You can't (except null) — it's a producer |
| Trying to read a specific type from `<? super T>` | You only get `Object` — it's a consumer |
| Forgetting PECS when designing APIs | Producer→extends, Consumer→super |
| Using a wildcard when you need to name the type | Use a type parameter `<T>` |
| Using raw types instead of `<?>` | `<?>` is type-safe; raw types aren't |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Flexible APIs:** methods taking `List<? extends Entity>` accept lists of any subtype — common in generic services and utilities.
- **Library design (Phase 7, 14):** PECS makes your generic APIs maximally usable while staying type-safe.
- **Stream/Collector signatures** (Phase 1.8) use bounded wildcards heavily (e.g., `Collector<? super T, A, R>`) — understanding PECS helps you read them.
- **Spring's generic types** (e.g., `ApplicationListener<? extends ApplicationEvent>`) use wildcards.
- **Avoiding raw types** (`<?>` over raw) keeps generic code safe (next note).

---

## 9. Quick Self-Check Questions

1. Why is `List<Integer>` not a `List<Number>`? What problem does invariance prevent?
2. What does `<?>` mean, and what can/can't you do with a `List<?>`?
3. What does `<? extends T>` allow — reading or writing? Why?
4. What does `<? super T>` allow — reading or writing? Why?
5. State the PECS rule and explain "producer" vs "consumer."
6. In `Collections.copy(dest, src)`, which gets `extends` and which gets `super`? Why?
7. When should you use a wildcard vs a named type parameter?
8. Why prefer `<?>` over a raw type?

---

## 10. Key Terms Glossary

- **Invariance:** `List<Sub>` is not a subtype of `List<Super>`.
- **Wildcard (`?`):** an unknown type argument.
- **Unbounded wildcard (`<?>`):** any type; read-as-Object, add-only-null.
- **Upper-bounded wildcard (`<? extends T>`):** T or a subtype; a **producer** (read as T).
- **Lower-bounded wildcard (`<? super T>`):** T or a supertype; a **consumer** (write T).
- **PECS:** Producer Extends, Consumer Super.
- **Producer / consumer:** a structure you read from / write to.
- **Type parameter vs wildcard:** named reusable type vs single-use unknown type.

---

*Previous topic: **Generics Basics & Bounded Type Parameters**.*
*Next topic: **Type Erasure, Pitfalls & Best Practices**.*
