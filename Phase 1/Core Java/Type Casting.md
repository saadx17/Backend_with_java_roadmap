**Type casting** = converting a value from one type to another. Java has two contexts:
1. **Primitive casting** - converting between numeric types (`int` ↔ `double`, etc.).
2. **Reference casting** - viewing an object as a different type in its inheritance hierarchy (upcasting/downcasting).

Each context has a **safe automatic** form and a **risky explicit** form.

| | Safe / automatic | Risky / explicit |
|---|-----------------|------------------|
| **Primitives** | Widening (small → large) | Narrowing (large → small) |
| **References** | Upcasting (child → parent) | Downcasting (parent → child) |

## 1. Implicit Casting  Widening (Primitives)
**Widening** converts a smaller type to a larger one. It's **automatic** (no cast needed) because no information is lost, the larger type can always hold the smaller value.

```java
int i = 100;
long l = i;        // int -> long   (automatic)
double d = l;      // long -> double (automatic)
```

### 1.1 The widening order
```
byte → short → int → long → float → double
        char → int → ...
```
Any conversion **along this chain (left to right)** is automatic.

```java
byte b = 10;
int x = b;         // widening, automatic
double y = x;      // widening, automatic
float f = 100L;    // long -> float, automatic (though precision can drop, see §1.2)
```

### 1.2 Subtle precision loss in widening
Even "widening" `long → float` or `long → double` can lose **precision** (not magnitude), because floats have limited significant digits:
```java
long big = 9_223_372_036_854_775_807L;
float f = big;          // compiles (widening), but f is an approximation
```
> Widening never loses *magnitude/overflow*, but `int/long → float/double` can lose *precision*. Rare but worth knowing.

## 2. Explicit Casting - Narrowing (Primitives)

**Narrowing** converts a larger type to a smaller one. It **requires an explicit cast** because data can be **lost** (truncation/overflow).

```java
double d = 9.99;
int i = (int) d;       // explicit cast -> 9  (fractional part TRUNCATED, not rounded)

long l = 130L;
byte b = (byte) l;     // explicit cast -> -126  (overflow! 130 doesn't fit in byte)
```

### 2.1 What narrowing can do to your data
| Conversion | Risk |
|------------|------|
| `double`/`float` → integer | **Truncates** the decimal part (toward zero) |
| Larger int → smaller int | **Overflow** — keeps only the low-order bits |
| `int` → `char` | Reinterprets as a character code |

```java
(int) 3.99;        // 3   (truncates, doesn't round)
(int) -3.99;       // -3
(byte) 300;        // 44  (300 mod 256 = 44, low 8 bits)
(char) 65;         // 'A'
```
> To **round** instead of truncate, use `Math.round()`: `Math.round(3.99)` → `4`.

### 2.2 The compound-assignment exception (recap)
Compound operators implicitly narrow, hiding the cast:
```java
byte b = 10;
b += 5;            // OK -> implicit (byte) cast; b = 15
// b = b + 5;      // ERROR -> b+5 is int, needs explicit cast
```

---

## 3. Reference Casting
For objects, casting changes the **reference type** you view the object through, the object itself is unchanged (recall the Polymorphism note).

### 3.1 Upcasting (subtype → supertype) - implicit, safe
```java
Dog dog = new Dog();
Animal a = dog;            // upcast: automatic, always safe (Dog IS-A Animal)
```
- No cast needed; always valid.
- You lose access to subtype-specific members through the parent reference, but overridden methods still dispatch dynamically.

### 3.2 Downcasting (supertype → subtype) - explicit, risky
```java
Animal a = new Dog();
Dog d = (Dog) a;           // downcast: explicit; you assert "this really is a Dog"
d.bark();                  // now Dog-specific members accessible
```
- Required to access subtype members.
- **Throws `ClassCastException` at runtime** if the object isn't actually that type.

## 4. `ClassCastException`
Thrown when you downcast an object to a type it **isn't**:
```java
Animal a = new Cat();
Dog d = (Dog) a;           // compiles fine... but throws ClassCastException at runtime!
```
The compiler allows it (both are `Animal`s), but at runtime the actual object is a `Cat`, not a `Dog`.

### 4.1 Guard downcasts with `instanceof`
```java
if (a instanceof Dog) {
    Dog d = (Dog) a;
    d.bark();
}
```
Or, better, **pattern matching for `instanceof`** (Java 16+) — test and bind safely in one step:
```java
if (a instanceof Dog d) {  // only enters (and binds d) if a really is a Dog
    d.bark();
}
```

## 5. Other Common Conversions (not casts, but related)
These look like conversions but use **methods**, not the cast operator:

| Conversion | How |
|------------|-----|
| `String` → `int` | `Integer.parseInt("42")` |
| `int` → `String` | `String.valueOf(42)` or `Integer.toString(42)` or `"" + 42` |
| `String` → `double` | `Double.parseDouble("3.14")` |
| primitive ↔ wrapper | autoboxing/unboxing (Data Types note) |
| number formatting | `String.format`, `BigDecimal`, etc. |

```java
int n = Integer.parseInt("123");        // may throw NumberFormatException
String s = String.valueOf(123);
```
> Parsing invalid input throws **`NumberFormatException`** - always validate/catch when parsing external input (Phase 1.5, API input handling Phase 5).

## 6. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Expecting `(int) 3.99` to round | It truncates; use `Math.round()` |
| Overflow when narrowing (`(byte) 300`) | Ensure the value fits, or use a larger type |
| Unchecked downcast → `ClassCastException` | Guard with `instanceof` / pattern matching |
| `Integer.parseInt` on bad input | Catch `NumberFormatException`; validate |
| Assuming widening is always lossless | `long → float/double` can lose precision |
| Casting unrelated types | Only valid within an inheritance hierarchy |
| Forgetting compound assignment auto-casts | Be aware `+=` narrows implicitly |

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Parsing request params/JSON** (`String` → numbers) is constant in web APIs — handle `NumberFormatException` and validate input (Phase 5, 15).
- **Generics largely eliminate downcasting** in modern code — `List<String>` avoids casting elements (Phase 1.7). Excessive casting is a design smell.
- **`ClassCastException`** still appears with reflection, raw types, and deserialization — pattern matching makes handling safer.
- **`BigDecimal`** (not float casts) for money; **`Math.round`** for controlled rounding (Phase 4/5).
- **Pattern matching `instanceof`** streamlines polymorphic handling and event/result processing (sealed types — Phase 1.3).

## 8. Quick Self-Check Questions

1. What's the difference between widening and narrowing? Which needs an explicit cast?
2. What is the widening order of the primitive types?
3. What does `(int) 3.99` produce, and how do you round instead?
4. Why does `(byte) 300` give `44`?
5. What's the difference between upcasting and downcasting? Which can fail?
6. When is `ClassCastException` thrown, and how do you prevent it?
7. How do you convert a `String` to an `int`, and what exception can it throw?
8. Can widening ever lose information? Explain.

## 9. Key Terms Glossary

- **Type casting:** converting a value from one type to another.
- **Widening (implicit):** small → large primitive; automatic, no data loss in magnitude.
- **Narrowing (explicit):** large → small primitive; requires a cast; may lose data.
- **Truncation:** dropping the fractional part when casting float → integer.
- **Overflow:** value exceeding the target type's range when narrowing.
- **Upcasting:** treating a subtype as its supertype (implicit, safe).
- **Downcasting:** treating a supertype reference as a subtype (explicit, risky).
- **`ClassCastException`:** runtime error from an invalid reference downcast.
- **`NumberFormatException`:** thrown when parsing an invalid numeric string.
- **Pattern matching for `instanceof`:** test-and-bind in one expression.