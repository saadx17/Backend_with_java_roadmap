# Lambda Expressions & Functional Interfaces

> **Phase 1 — Java Language Mastery → 1.8 Functional Programming**
> Goal: Master lambda expressions (syntax, variable capture, scoping) and functional interfaces (the built-in `Function`, `Predicate`, `Consumer`, `Supplier`, and friends).

---

## 0. The Big Picture

**Functional programming** in Java (since Java 8) lets you treat **behavior as data** — passing functions (lambdas) as arguments, returning them, and composing them. This enables concise, declarative code, especially with the Stream API (later notes).

```java
// Imperative (how):
List<String> result = new ArrayList<>();
for (String s : names) if (s.length() > 3) result.add(s);

// Functional (what):
List<String> result = names.stream().filter(s -> s.length() > 3).toList();
```

> The foundation is the **lambda** — a concise anonymous function — and the **functional interface** it implements. Everything else (streams, method references, `Optional`) builds on these.

---

## 1. Lambda Expressions

A **lambda** is a short block of code that takes parameters and returns a value — an **anonymous function**. It's a concise way to implement a **functional interface** (an interface with one abstract method — recall Phase 1.3).

```java
// Before (anonymous class — verbose):
Runnable r = new Runnable() {
    @Override public void run() { System.out.println("hi"); }
};

// After (lambda):
Runnable r = () -> System.out.println("hi");
```

### 1.1 Syntax forms
```java
(parameters) -> expression
(parameters) -> { statements; }
```
| Form | Example |
|------|---------|
| No parameters | `() -> 42` |
| One parameter (parens optional) | `x -> x * 2` or `(x) -> x * 2` |
| Multiple parameters | `(a, b) -> a + b` |
| Explicit types | `(int a, int b) -> a + b` |
| Block body (needs `return`) | `(a, b) -> { int s = a + b; return s; }` |
| Expression body (implicit return) | `(a, b) -> a + b` |

```java
Comparator<String> byLength = (a, b) -> Integer.compare(a.length(), b.length());
Predicate<Integer> isEven = n -> n % 2 == 0;
Function<String, Integer> length = s -> s.length();
Runnable task = () -> System.out.println("running");
```

### 1.2 A lambda is an implementation of a functional interface
A lambda has no type by itself — its **target type** is a functional interface inferred from context:
```java
Runnable r = () -> System.out.println("hi");   // target type: Runnable
// The lambda body becomes the implementation of Runnable.run()
```

---

## 2. Variable Capture & "Effectively Final"

Lambdas can **capture** variables from their enclosing scope, but only **`final` or effectively final** ones (recall Inner Classes note, Phase 1.3):
```java
int factor = 10;                    // effectively final (never reassigned)
Function<Integer, Integer> multiply = x -> x * factor;   // captures 'factor'
// factor = 20;                     // if uncommented -> COMPILE ERROR (no longer effectively final)
```

### 2.1 Why this restriction?
A lambda may outlive the method's stack frame, so Java **captures the value** (not a live reference to the stack slot). Requiring effective-finality prevents confusing situations where the captured copy and the original could diverge. (Same rule as anonymous/local classes — Phase 1.3.)

### 2.2 Workaround for "mutable" capture
You can't reassign a captured local, but you can mutate a referenced object's contents, or use an array/atomic as a holder:
```java
int[] counter = {0};
list.forEach(x -> counter[0]++);   // mutate array contents (the reference is final)
// Better in concurrent code: AtomicInteger (Phase 1.10)
```

### 2.3 Lexical scoping & `this`
A lambda **does not introduce a new scope** — it's lexically scoped to its enclosing method. Crucially, **`this` inside a lambda refers to the enclosing instance** (unlike an anonymous class, where `this` is the anonymous object — recall Phase 1.3):
```java
class Service {
    String name = "svc";
    void run() {
        Runnable r = () -> System.out.println(this.name);  // 'this' = the Service instance
    }
}
```

---

## 3. Functional Interfaces

A **functional interface** has exactly **one abstract method (SAM)** — so a lambda can implement it (recall Phase 1.3 Interfaces). The `@FunctionalInterface` annotation makes the compiler enforce this.
```java
@FunctionalInterface
interface Calculator {
    int apply(int a, int b);     // the single abstract method
    // default/static methods don't count and are allowed
}
Calculator add = (a, b) -> a + b;
add.apply(2, 3);   // 5
```

> The lambda's parameter/return types must match the single abstract method's signature.

---

## 4. The Built-in Functional Interfaces (`java.util.function`)

Java provides a standard set in `java.util.function` — learn these; they're used throughout the Stream API and modern Java. The **four core** ones:

| Interface | Abstract method | Takes → Returns | Mnemonic |
|-----------|-----------------|-----------------|----------|
| **`Function<T,R>`** | `R apply(T t)` | T → R | transform |
| **`Predicate<T>`** | `boolean test(T t)` | T → boolean | test/filter |
| **`Consumer<T>`** | `void accept(T t)` | T → (nothing) | consume/side-effect |
| **`Supplier<T>`** | `T get()` | (nothing) → T | supply/produce |

```java
Function<String, Integer> length = s -> s.length();      // length.apply("abc") -> 3
Predicate<Integer> isPositive = n -> n > 0;              // isPositive.test(5) -> true
Consumer<String> printer = s -> System.out.println(s);   // printer.accept("hi")
Supplier<Double> random = () -> Math.random();           // random.get()
```

### 4.1 Function — `apply`, `compose`, `andThen`
`Function` can be **composed**:
```java
Function<Integer, Integer> doubleIt = x -> x * 2;
Function<Integer, Integer> addOne   = x -> x + 1;

doubleIt.andThen(addOne).apply(5);   // (5*2)+1 = 11   (doubleIt FIRST, then addOne)
doubleIt.compose(addOne).apply(5);   // (5+1)*2 = 12   (addOne FIRST, then doubleIt)
Function.identity();                  // x -> x
```

### 4.2 Predicate — `test`, `and`, `or`, `negate`
`Predicate`s combine like booleans:
```java
Predicate<Integer> positive = n -> n > 0;
Predicate<Integer> even     = n -> n % 2 == 0;

positive.and(even).test(4);    // true  (positive AND even)
positive.or(even).test(-2);    // true  (even)
positive.negate().test(-1);    // true  (NOT positive)
Predicate.not(even).test(3);   // true  (static helper, Java 11+)
```

### 4.3 Consumer — `accept`, `andThen`
```java
Consumer<String> log   = s -> System.out.println("LOG: " + s);
Consumer<String> save  = s -> database.store(s);
log.andThen(save).accept("data");   // log first, then save (chained side-effects)
```

### 4.4 Supplier — `get`
Used for **lazy** value production (compute only when needed):
```java
Supplier<ExpensiveObject> lazy = () -> new ExpensiveObject();
// Not created until lazy.get() is called — basis of Optional.orElseGet, logging, etc.
```

---

## 5. More Built-in Functional Interfaces

### 5.1 Operators (input and output same type)
| Interface | Equivalent to | Use |
|-----------|---------------|-----|
| `UnaryOperator<T>` | `Function<T,T>` | T → T (e.g., `x -> x*2`) |
| `BinaryOperator<T>` | `BiFunction<T,T,T>` | (T,T) → T (e.g., `(a,b) -> a+b`) |

```java
UnaryOperator<String> upper = String::toUpperCase;   // (method ref — next note)
BinaryOperator<Integer> sum  = Integer::sum;
```

### 5.2 Bi-variants (two arguments)
| Interface | Method | Takes → Returns |
|-----------|--------|-----------------|
| `BiFunction<T,U,R>` | `apply(T,U)` | (T,U) → R |
| `BiPredicate<T,U>` | `test(T,U)` | (T,U) → boolean |
| `BiConsumer<T,U>` | `accept(T,U)` | (T,U) → void |

```java
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
BiConsumer<String, Integer> printEntry = (k, v) -> System.out.println(k + "=" + v);
map.forEach(printEntry);   // Map.forEach takes a BiConsumer
```

### 5.3 Primitive specializations (avoid boxing!)
To avoid autoboxing overhead (recall Phase 1.2), there are primitive versions:
| Interface | Avoids boxing of |
|-----------|------------------|
| `IntFunction<R>`, `IntPredicate`, `IntConsumer`, `IntSupplier`, `IntUnaryOperator` | `int` |
| `ToIntFunction<T>` | returns `int` |
| `LongFunction`, `DoubleFunction`, ... | `long`, `double` |

```java
IntPredicate isEven = n -> n % 2 == 0;        // no Integer boxing
ToIntFunction<String> len = String::length;    // String -> int (unboxed)
```
> In hot paths / streams, primitive specializations matter for performance (no boxing per element).

---

## 6. Why Lambdas Over Anonymous Classes (recap)

| | Anonymous class | Lambda |
|---|-----------------|--------|
| Verbosity | High | Low |
| Works for | Any interface/class | **Functional interfaces only** |
| `this` | The anonymous instance | The **enclosing** instance |
| Extra class file | Yes (`Outer$1.class`) | No (uses `invokedynamic`) |
| State/fields | Can have | Cannot |

> Use a **lambda** for functional interfaces (the common case); use an anonymous class only when you need multiple methods, fields, or to subclass a non-functional type (recall Phase 1.3 Inner Classes).

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Reassigning a captured local variable | Captured locals must be effectively final |
| Expecting `this` to mean the lambda | In a lambda, `this` = enclosing instance |
| Side effects in lambdas passed to streams | Keep lambdas pure (no shared mutable state) |
| Boxing in numeric lambdas/streams | Use primitive specializations (`IntPredicate`, etc.) |
| Overusing block-body lambdas | Keep lambdas short; extract complex logic to methods |
| Adding a 2nd abstract method to `@FunctionalInterface` | Keep exactly one (defaults/statics are fine) |
| `compose` vs `andThen` confusion | `andThen` runs *this* first; `compose` runs the *argument* first |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Stream API** (next notes) takes lambdas everywhere — `filter`, `map`, `forEach`.
- **Spring callbacks:** `JdbcTemplate` (`RowMapper`), `RestTemplate`, `TransactionTemplate`, event listeners all take functional interfaces/lambdas.
- **`Optional`** methods (`map`, `filter`, `orElseGet`) take functions/suppliers (Phase 1.8 last note).
- **`CompletableFuture`** (Phase 1.10) chains lambdas (`thenApply`, `thenCompose`).
- **Comparators** (Phase 1.6) are built with lambdas/method references.
- **Reactive programming** (Phase 16) is functional to the core (Reactor operators are lambdas).

---

## 9. Quick Self-Check Questions

1. What is a lambda, and what does it implement?
2. Write a lambda for: no args returning 42; one arg doubling it; two args summing.
3. What does "effectively final" mean for captured variables, and why is it required?
4. What does `this` refer to inside a lambda vs an anonymous class?
5. Name the four core functional interfaces and their methods.
6. What's the difference between `Function.andThen` and `Function.compose`?
7. How do you combine predicates with AND/OR/NOT?
8. Why use primitive specializations like `IntPredicate`?
9. When should you use an anonymous class instead of a lambda?

---

## 10. Key Terms Glossary

- **Functional programming:** treating behavior/functions as data.
- **Lambda expression:** a concise anonymous function.
- **Target type:** the functional interface a lambda implements (from context).
- **Functional interface (SAM):** interface with one abstract method.
- **`@FunctionalInterface`:** annotation enforcing the SAM rule.
- **Variable capture / effectively final:** using enclosing variables (assigned once).
- **`Function` / `Predicate` / `Consumer` / `Supplier`:** the four core functional interfaces.
- **`andThen` / `compose`:** chaining functions (this-first / argument-first).
- **`UnaryOperator` / `BinaryOperator`:** same-type operators.
- **Bi-variants:** two-argument functional interfaces.
- **Primitive specialization:** boxing-free functional interfaces (`IntPredicate`, etc.).

---

*This is the first note of **Section 1.8 — Functional Programming**.*
*Next topic: **Method References**.*
