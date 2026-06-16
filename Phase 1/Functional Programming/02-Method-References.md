# Method References

> **Phase 1 — Java Language Mastery → 1.8 Functional Programming**
> Goal: Master the four kinds of method references — static, bound instance, unbound instance, and constructor — and know when to use them instead of lambdas.

---

## 0. The Big Picture

A **method reference** is a shorthand for a lambda that **just calls an existing method**. When a lambda's only job is to invoke one method, a method reference (`::`) is more concise and readable.

```java
// Lambda that just calls a method:
names.forEach(s -> System.out.println(s));
// Method reference (same thing, cleaner):
names.forEach(System.out::println);
```

> Method references are pure syntactic sugar over lambdas — same target types (functional interfaces), just terser. The `::` operator is the giveaway.

---

## 1. When a Lambda Becomes a Method Reference

If a lambda **only delegates to an existing method** (no extra logic), you can usually replace it with a method reference:
```java
s -> s.toUpperCase()        ->  String::toUpperCase
x -> Math.abs(x)            ->  Math::abs
s -> System.out.println(s)  ->  System.out::println
() -> new ArrayList<>()     ->  ArrayList::new
```
> Rule of thumb: if the lambda is `x -> someMethod(x)` (passes its args straight through), it's a method reference candidate.

---

## 2. The Four Kinds of Method References

| # | Kind | Syntax | Example |
|---|------|--------|---------|
| 1 | **Static method** | `ClassName::staticMethod` | `Integer::parseInt` |
| 2 | **Instance method of a specific object** (bound) | `instance::method` | `System.out::println` |
| 3 | **Instance method of an arbitrary object** (unbound) | `ClassName::instanceMethod` | `String::toUpperCase` |
| 4 | **Constructor** | `ClassName::new` | `ArrayList::new` |

---

## 3. Type 1 — Static Method Reference

References a static method: `ClassName::staticMethod`.
```java
Function<String, Integer> parse = Integer::parseInt;
parse.apply("42");           // 42
// equivalent lambda: s -> Integer.parseInt(s)

BinaryOperator<Integer> max = Integer::max;     // (a,b) -> Integer.max(a,b)
Function<Double, Double> abs = Math::abs;       // x -> Math.abs(x)

stream.map(Integer::parseInt);   // parse each string
```

---

## 4. Type 2 — Bound Instance Method Reference (Specific Object)

References a method **on a particular object** that already exists: `instance::method`. The object is "bound" — fixed.
```java
String prefix = "Hello, ";
Function<String, String> greet = prefix::concat;   // x -> prefix.concat(x)
greet.apply("World");         // "Hello, World"

System.out::println;          // 'System.out' is the specific object
list.forEach(System.out::println);

Logger log = ...;
runnable = log::flush;        // bound to this specific logger
```
> "Bound" = the receiver object is already chosen (`System.out`, `prefix`, `log`). The lambda's parameters become the *method's* parameters.

---

## 5. Type 3 — Unbound Instance Method Reference (Arbitrary Object)

References an instance method where the **receiver is supplied as the first lambda parameter**: `ClassName::instanceMethod`. This is the trickiest one.
```java
Function<String, String> upper = String::toUpperCase;   // s -> s.toUpperCase()
upper.apply("hi");            // "HI"  — the String IS the receiver

// The first argument becomes the object the method is called on:
Comparator<String> byLen = Comparator.comparingInt(String::length);  // s -> s.length()
stream.map(String::trim);     // each element calls its own .trim()
stream.sorted(String::compareTo);   // (a,b) -> a.compareTo(b)
```
> **Bound vs unbound (key distinction):**
> - **Bound** `prefix::concat` → receiver is **fixed** (`prefix`); the lambda arg is the *method's* argument.
> - **Unbound** `String::toUpperCase` → receiver is the **first lambda argument** (a different String each call).

```
Bound:    instance::method      -> x          becomes  instance.method(x)
Unbound:  ClassName::method     -> (x)        becomes  x.method()
Unbound:  ClassName::method     -> (x, y)     becomes  x.method(y)
```

---

## 6. Type 4 — Constructor Reference

References a constructor: `ClassName::new`. Great for factories and stream collection.
```java
Supplier<ArrayList<String>> listFactory = ArrayList::new;   // () -> new ArrayList<>()
ArrayList<String> list = listFactory.get();

Function<String, StringBuilder> sbFactory = StringBuilder::new;  // s -> new StringBuilder(s)

// Common in streams — collect into a specific structure:
stream.collect(Collectors.toCollection(TreeSet::new));   // create the result container
names.stream().map(User::new).toList();   // User(String) constructor per name

// Array constructor reference:
Function<Integer, String[]> arrayMaker = String[]::new;   // n -> new String[n]
String[] arr = stream.toArray(String[]::new);             // very common!
```

---

## 7. Method Reference ↔ Lambda Equivalence Table

| Method reference | Equivalent lambda |
|------------------|-------------------|
| `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| `System.out::println` | `x -> System.out.println(x)` |
| `String::toUpperCase` | `s -> s.toUpperCase()` |
| `String::compareTo` | `(a, b) -> a.compareTo(b)` |
| `ArrayList::new` | `() -> new ArrayList<>()` |
| `User::new` | `name -> new User(name)` |
| `String[]::new` | `n -> new String[n]` |

---

## 8. When to Use Method References vs Lambdas

| Use a **method reference** when... | Use a **lambda** when... |
|------------------------------------|--------------------------|
| The lambda just calls one existing method | You need extra logic / multiple statements |
| It improves readability (`String::trim`) | The method ref would be obscure |
| Passing a method directly to a stream op | You transform args before calling |

```java
// Method reference (clear):
stream.map(String::toUpperCase)
// Lambda (needed — extra logic):
stream.map(s -> s.trim().toUpperCase())
```
> Prefer method references **when they're clearer**. If a method reference makes you pause to figure out which overload/binding applies, a lambda may be more readable. Don't force it.

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Confusing bound vs unbound instance refs | Bound = fixed receiver; unbound = receiver is the 1st arg |
| Using a method ref when the lambda adds logic | Use a lambda for `s -> s.trim().toUpperCase()` |
| Forgetting `ClassName::new` for constructors | Constructor refs are `::new` |
| Ambiguity with overloaded methods | Use a lambda to disambiguate if needed |
| Expecting method refs to do anything lambdas can't | They're equivalent — just sugar |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Streams everywhere** read better with method references: `.map(User::getName)`, `.filter(Objects::nonNull)`, `.sorted(comparingInt(Order::getTotal))`.
- **`toArray(User[]::new)`** is the idiomatic way to collect a stream to a typed array.
- **`Collectors.toCollection(TreeSet::new)`** and constructor refs for custom containers (next note).
- **Comparators** (Phase 1.6) read cleanly with `Comparator.comparing(Entity::getField)`.
- **Mapping/factory patterns** (DTO ↔ entity, Phase 7) use constructor references.
- Cleaner, more declarative service code overall.

---

## 11. Quick Self-Check Questions

1. What is a method reference, and what operator denotes it?
2. When can a lambda be rewritten as a method reference?
3. Name the four kinds of method references with an example of each.
4. What's the difference between a bound and an unbound instance method reference?
5. In `String::toUpperCase` used as `Function<String,String>`, where does the receiver come from?
6. Write the lambda equivalent of `User::new` and `String[]::new`.
7. When should you prefer a lambda over a method reference?

---

## 12. Key Terms Glossary

- **Method reference (`::`):** shorthand for a lambda that calls one method.
- **Static method reference:** `ClassName::staticMethod`.
- **Bound instance method reference:** `instance::method` (fixed receiver).
- **Unbound instance method reference:** `ClassName::method` (receiver = first arg).
- **Constructor reference:** `ClassName::new`.
- **Array constructor reference:** `Type[]::new`.
- **Receiver:** the object an instance method is called on.
- **Target type:** the functional interface the reference satisfies.

---

*Previous topic: **Lambdas & Functional Interfaces**.*
*Next topic: **Stream API — Creation & Operations**.*
