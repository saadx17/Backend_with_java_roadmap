# Generics Basics & Bounded Type Parameters

> **Phase 1 — Java Language Mastery → 1.7 Generics**
> Goal: Understand why generics exist, how to write generic classes/methods/interfaces, bounded type parameters, and type inference (the diamond operator).

---

## 0. The Big Picture

**Generics** let you write classes, interfaces, and methods that work with **any type**, while keeping **compile-time type safety**. Instead of coding against `Object` and casting everywhere, you parameterize types.

```java
List<String> names = new ArrayList<>();   // <String> = type parameter
names.add("Sam");
String s = names.get(0);                   // no cast needed — guaranteed a String
// names.add(42);                          // COMPILE ERROR — type safety
```

> Generics are the reason the Collections Framework (Phase 1.6) is type-safe. You use them constantly; understanding them removes a whole class of bugs and casts.

---

## 1. Why Generics Exist

### 1.1 Before generics (Java < 5): casting hell
```java
List list = new ArrayList();      // raw type — holds Object
list.add("hello");
list.add(42);                     // compiles! no type checking
String s = (String) list.get(0);  // must cast
String bad = (String) list.get(1); // ClassCastException at RUNTIME!
```
Problems: no type safety, manual casts everywhere, errors found only at runtime.

### 1.2 With generics: safety + clarity
```java
List<String> list = new ArrayList<>();
list.add("hello");
// list.add(42);                   // COMPILE ERROR — caught immediately
String s = list.get(0);            // no cast; compiler guarantees String
```

### 1.3 The two benefits
| Benefit | Meaning |
|---------|---------|
| **Type safety** | Type errors caught at **compile time**, not runtime |
| **Eliminate casts** | The compiler knows the type → no manual casting |

> Generics move errors from **runtime** (`ClassCastException`) to **compile time** — far cheaper to fix. They also make code self-documenting (`Map<UserId, User>` says exactly what's stored).

---

## 2. Generic Classes

A **generic class** declares one or more **type parameters** in angle brackets after its name:
```java
public class Box<T> {              // T is a type parameter
    private T content;
    public void set(T content) { this.content = content; }
    public T get() { return content; }
}

Box<String> sb = new Box<>();
sb.set("hello");
String s = sb.get();              // T is String here

Box<Integer> ib = new Box<>();
ib.set(42);                        // T is Integer here
```

### 2.1 Type parameter naming conventions
| Letter | Conventional meaning |
|--------|----------------------|
| `T` | Type |
| `E` | Element (collections) |
| `K`, `V` | Key, Value (maps) |
| `N` | Number |
| `R` | Return type |
| `S`, `U` | Additional types |

### 2.2 Multiple type parameters
```java
public class Pair<K, V> {
    private final K key;
    private final V value;
    public Pair(K key, V value) { this.key = key; this.value = value; }
    public K getKey()   { return key; }
    public V getValue() { return value; }
}
Pair<String, Integer> p = new Pair<>("age", 30);
```

---

## 3. Generic Interfaces

Interfaces can be generic too — `Comparable<T>` and `Comparator<T>` (Phase 1.6) are key examples:
```java
public interface Repository<T, ID> {      // generic interface
    T findById(ID id);
    List<T> findAll();
    void save(T entity);
}

class UserRepository implements Repository<User, Long> {  // bind the type parameters
    public User findById(Long id) { ... }
    public List<User> findAll() { ... }
    public void save(User entity) { ... }
}
```
> This is exactly how **Spring Data's `JpaRepository<T, ID>`** works (Phase 5.4) — a generic interface you parameterize with your entity and ID types.

---

## 4. Generic Methods

A **generic method** declares its own type parameter(s) **before the return type** — independent of any class-level generics:
```java
public class Utils {
    public static <T> T firstOrNull(List<T> list) {   // <T> declared before return type
        return list.isEmpty() ? null : list.get(0);
    }

    public static <T> void swap(T[] array, int i, int j) {
        T temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }

    public static <K, V> Map<V, K> invert(Map<K, V> map) {  // multiple params
        Map<V, K> result = new HashMap<>();
        map.forEach((k, v) -> result.put(v, k));
        return result;
    }
}

String s = Utils.firstOrNull(List.of("a", "b"));   // T inferred as String
```

### 4.1 Key points
- The `<T>` **before the return type** declares the method's own type parameter.
- The type is usually **inferred** from the arguments (you rarely specify it explicitly).
- A generic method can be `static` (unlike class-level type parameters, which need an instance).
- Explicit type witness (rare): `Utils.<String>firstOrNull(list)`.

---

## 5. Bounded Type Parameters

Sometimes you need to **restrict** what types a parameter can be — e.g., "must be a Number" so you can call numeric methods. Use **`extends`**.

### 5.1 Upper bound (`extends`)
```java
public class Stats<T extends Number> {        // T must be Number or a subtype
    private final List<T> nums;
    public Stats(List<T> nums) { this.nums = nums; }

    public double sum() {
        double total = 0;
        for (T n : nums) total += n.doubleValue();  // can call Number methods!
        return total;
    }
}
Stats<Integer> s = new Stats<>(List.of(1, 2, 3));   // OK (Integer extends Number)
// Stats<String> bad = ...;                          // COMPILE ERROR (String isn't a Number)
```
> `<T extends Number>` means **T is Number or any subclass** — so you can call `Number`'s methods (`doubleValue()`, etc.) on `T`. Without the bound, `T` is treated as `Object` and you couldn't do that.

### 5.2 `extends` works for both classes and interfaces
Despite the keyword `extends`, it covers implementing interfaces too:
```java
public <T extends Comparable<T>> T max(List<T> list) {   // T must be Comparable
    T result = list.get(0);
    for (T item : list)
        if (item.compareTo(result) > 0) result = item;    // can call compareTo
    return result;
}
```

### 5.3 Multiple bounds (`&`)
A type parameter can have several bounds joined with `&` (at most one may be a class, and it must come first):
```java
public <T extends Comparable<T> & Serializable> void process(T item) {
    item.compareTo(other);   // Comparable method
    // item is also guaranteed Serializable
}
```

### 5.4 Recursive type bounds
A common pattern where the bound refers to the type parameter itself — used for "comparable to itself":
```java
public static <T extends Comparable<T>> T max(Collection<T> coll) { ... }
//                       ^^^^^^^^^^^^^ T must be comparable to its own type
```
> `<T extends Comparable<T>>` reads as "T is a type that can be compared to other Ts" — exactly what you need for sorting/min/max generically.

---

## 6. Type Inference & the Diamond Operator

### 6.1 The diamond operator (`<>`)
Since Java 7, you don't repeat the type on the right side — the compiler **infers** it:
```java
Map<String, List<Integer>> map = new HashMap<>();   // <> infers <String, List<Integer>>
// pre-Java 7: new HashMap<String, List<Integer>>()  (verbose, redundant)
```

### 6.2 Inference in method calls
```java
List<String> list = List.of("a", "b");    // T inferred as String
var nums = List.of(1, 2, 3);               // var + inference (Phase 1.2)
Map<String, Integer> m = Map.of("a", 1);   // K=String, V=Integer inferred
```

### 6.3 Inference limits
The compiler can't always infer — sometimes you need an explicit type:
```java
List<String> empty = Collections.emptyList();           // inferred from target type
List<String> e2 = Collections.<String>emptyList();      // explicit (rarely needed)
```

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using **raw types** (`List` instead of `List<String>`) | Always parameterize (next note covers why raw is dangerous) |
| Thinking `extends` is only for classes | It also means "implements" for interfaces |
| Confusing class type params with method type params | Method declares its own `<T>` before return type |
| Trying to use a class type parameter in a static method | Use a generic **method** (`static <T>`) |
| Forgetting bounds when you need type's methods | Add `<T extends X>` to call X's methods |
| Over-specifying types when inference works | Use the diamond `<>` / `var` |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Collections** (Phase 1.6) are all generic — `List<User>`, `Map<String, Order>`.
- **Spring Data** `JpaRepository<T, ID>` is a generic interface you parameterize (Phase 5.4).
- **Generic service/DTO base classes** and `ResponseEntity<T>` (Phase 5.3) rely on generics.
- **`Optional<T>`, `CompletableFuture<T>`, `Stream<T>`** are generic (Phase 1.8, 1.10).
- **Type safety** prevents `ClassCastException` in production and makes APIs self-documenting (Phase 7).
- **Bounded types** appear in generic algorithms and frameworks (e.g., `<T extends Comparable<T>>`).

---

## 9. Quick Self-Check Questions

1. What two main benefits do generics provide?
2. What problem did raw types/`Object` collections have before generics?
3. How do you declare a generic class with two type parameters?
4. Where do you declare a generic method's type parameter, and can it be static?
5. What does `<T extends Number>` allow you to do with `T`?
6. Does `extends` in a bound also cover interfaces? How do you specify multiple bounds?
7. What is a recursive type bound (`<T extends Comparable<T>>`) and why use it?
8. What does the diamond operator do?

---

## 10. Key Terms Glossary

- **Generics:** parameterizing types for compile-time type safety.
- **Type parameter:** a placeholder type (`T`, `E`, `K`, `V`).
- **Generic class/interface/method:** one declaring type parameters.
- **Type argument:** the concrete type supplied (`String` in `List<String>`).
- **Bounded type parameter:** restricting a type (`<T extends Number>`).
- **Upper bound (`extends`):** the type must be a subtype of the bound.
- **Multiple bounds (`&`):** several constraints on one type parameter.
- **Recursive type bound:** a bound referencing its own type parameter.
- **Type inference:** the compiler deducing type arguments.
- **Diamond operator (`<>`):** infers the type on the right side.

---

*This is the first note of **Section 1.7 — Generics**.*
*Next topic: **Wildcards & the PECS Principle**.*
