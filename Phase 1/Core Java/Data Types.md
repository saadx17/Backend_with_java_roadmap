Java is a **statically typed** language: every variable has a declared type known at compile time. Java has **two categories**:

| Category            | Examples                            | Stored as                              | Lives where                             |
| ------------------- | ----------------------------------- | -------------------------------------- | --------------------------------------- |
| **Primitive types** | `int`, `double`, `boolean`, ...     | The actual value                       | Stack (local vars) or inline in objects |
| **Reference types** | `String`, arrays, objects, wrappers | A reference (address) to a heap object | Reference on stack, object on heap      |

## 1. The 8 Primitive Types
Java has exactly **8 built-in primitive types**. Their sizes are **fixed across all platforms** (part of "write once, run anywhere").

| Type      | Size    | Range                 | Default Value | Example             |
| --------- | ------- | --------------------- | ------------- | ------------------- |
| `byte`    | 8 bits  | -128 to 127           | 0             | `byte b = 10;`      |
| `short`   | 16 bits | -32,768 to 32,767     | 0             | `short s = 1000;`   |
| `int`     | 32 bits | ~±2 billion           | 0             | `int i = 25;`       |
| `long`    | 64 bits | ~±9.2 quintillion     | 0L            | `long l = 999L;`    |
| `float`   | 32 bits | ~6-7 decimal digits   | 0.0f          | `float f = 3.14f;`  |
| `double`  | 64 bits | ~15-16 decimal digits | 0.0d          | `double d = 3.14;`  |
| `char`    | 16 bits | 0 to 65,535           | '\u0000'      | `char c = 'A';`     |
| `boolean` | 1 bit   | true / false          | false         | `boolean b = true;` |

### 1.1 Literals and suffixes
```java
int    i = 100;
long   l = 100L;          // L suffix (or l) — required for values > int range
float  f = 3.14f;         // f suffix — required (else treated as double)
double d = 3.14;          // double is the default for decimals
char   c = 'A';           // single quotes; also 'A' == 65
boolean b = true;

long big   = 9_000_000_000L;   // underscores for readability (Java 7+)
int  hex   = 0xFF;             // hexadecimal (255)
int  binary= 0b1010;          // binary (10)
int  octal = 010;             // octal (8)  -- careful: leading 0!
```

### 1.2 Default values
- **Instance/static fields** are auto-initialized to the defaults in the table above.
- **Local variables are NOT** - they must be explicitly initialized before use, or the compiler errors:
```java title:default_values.java
int x;
// System.out.println(x);   // COMPILE ERROR: variable x might not have been initialized
```

### 1.3 Overflow & floating-point gotchas
```java
int max = Integer.MAX_VALUE;     // 2147483647
System.out.println(max + 1);     // -2147483648  -> silent overflow (wraps around)

System.out.println(0.1 + 0.2);   // 0.30000000000000004  -> float imprecision!
```
- **Integer overflow is silent** (wraps via two's complement) — use `long`, or `Math.addExact()` to detect.
- **`float`/`double` are imprecise** (binary fractions). **Never use them for money** - use `BigDecimal` (Phase 4/5).

## 2. Wrapper Classes
A [wrapper class](https://www.geeksforgeeks.org/java/wrapper-classes-java/) in Java is a special class that "wraps" or encapsulates a primitive data type into an object.
While primitive types (like `int` or `char`) are efficient for performance, Java is an object-oriented language where many advanced features, such as collection and generics , only work with objects.

Each primitive has a corresponding **wrapper class** - an object form, living in `java.lang`:

```
Primitive    →    Wrapper Class
─────────────────────────────────
byte         →    Byte
short        →    Short
int          →    Integer
long         →    Long
float        →    Float
double       →    Double
char         →    Character
boolean      →    Boolean
```

### 2.1 Why wrappers exist
- **Generics & collections** work only with objects, not primitives: `List<Integer>` is legal, `List<int>` is not.
- Wrappers can be **`null`** (a primitive can't represent "no value").
- They provide **utility methods/constants**: `Integer.parseInt("42")`, `Integer.MAX_VALUE`, `Integer.toBinaryString(10)`, `Double.isNaN(x)`.

```java title:wrapper.java
List<Integer> nums = new ArrayList<>();   // must use wrapper in generics
int parsed = Integer.parseInt("123");     // String -> int
Integer maybeNull = null;                 // wrappers can be null
```

### 2.2 Wrappers are immutable
Like `String`, wrapper objects are **immutable** - `Integer.valueOf(5)` always represents 5.

### 2.3 Autoboxing and Unboxing
Java automatically converts between primitives and wrappers (since Java 5):
- **Autoboxing:** primitive → wrapper (`int` → `Integer`)
- **Unboxing:** wrapper → primitive (`Integer` → `int`)

```java title:auto_boxing.java
// AUTOBOXING → primitive automatically becomes object
Integer num = 100;       // compiler converts to: Integer.valueOf(100)
Double  d   = 3.14;      // compiler converts to: Double.valueOf(3.14)
```

```java title:unboxing.java
// UNBOXING → object automatically becomes primitive
int result = num;        // compiler converts to: num.intValue()
double val = d;          // compiler converts to: d.doubleValue()

// Happens automatically in operations too
Integer x = 10;
Integer y = 20;
int sum = x + y;         // both unboxed, then added → 30
```

### 2.4 The `NullPointerException`
Unboxing a `null` wrapper throws **NPE**:
```java title:nullpointer.java
// Wrapper can be null, primitive CANNOT
Integer num = null;

// This will CRASH with NullPointerException!
int x = num;              // tries to unbox null → 💥 NPE

// Always check null before unboxing
if (num != null) {
    int x = num;          // safe
}

// Real world example - dangerous!
public int add(Integer a, Integer b) {
    return a + b;    // if a or b is null → NPE at runtime
}
```
> One of the most common subtle bugs. Be careful unboxing values that might be `null`.

### 2.5 Performance note
Auto-boxing in tight loops creates many wrapper objects (heap allocation + GC pressure):
```java title:performance_issue.java
Long sum = 0L;
for (long i = 0; i < 1_000_000; i++) sum += i;  // BAD: boxes/unboxes every iteration
// Fix: use primitive long for sum
```

### 2.6 Integer Caching (−128 to 127)
> **A classic interview topic and real bug source.**

Java **caches** `Integer` objects for values **−128 to 127**. `Integer.valueOf()` (and autoboxing) returns the **same cached object** for values in this range, but **new objects** outside it.

```java
Integer a = 100, b = 100;
System.out.println(a == b);   // true  -> both from the cache (same object)

Integer c = 200, d = 200;
System.out.println(c == d);   // false -> outside cache, different objects!

System.out.println(c.equals(d)); // true -> value comparison (always correct)
```
**RULE:** Always use `.equals()` to compare Wrapper objects. Never use `==` for value comparison.

### 2.7 Useful Utility Methods
```java title:Integer_utilities.java
// ─── Integer Utilities ───────────────────────────────

// String → int conversion
int a = Integer.parseInt("123");        // 123
int b = Integer.parseInt("FF", 16);     // 255 (hex to decimal)

// int → String
String s = Integer.toString(42);        // "42"
String s2 = String.valueOf(42);         // "42"

// Min / Max values
System.out.println(Integer.MAX_VALUE);  // 2147483647
System.out.println(Integer.MIN_VALUE);  // -2147483648

// Compare
int result = Integer.compare(10, 20);   // -1 (10 < 20)

// Binary, Hex, Octal
System.out.println(Integer.toBinaryString(10)); // "1010"
System.out.println(Integer.toHexString(255));   // "ff"
System.out.println(Integer.toOctalString(8));   // "10"
```

```java title:double_utilities.java
// ─── Double Utilities ────────────────────────────────

double d = Double.parseDouble("3.14");  // String → double
System.out.println(Double.MAX_VALUE);   // 1.7976931348623157E308
System.out.println(Double.isNaN(0.0 / 0.0));      // true
System.out.println(Double.isInfinite(1.0 / 0.0)); // true
```

```java title:char_utilities.java
// ─── Character Utilities ─────────────────────────────

char c = 'A';
System.out.println(Character.isLetter(c));     // true
System.out.println(Character.isDigit(c));      // false
System.out.println(Character.isWhitespace(' ')); // true
System.out.println(Character.toUpperCase('a')); // A
System.out.println(Character.toLowerCase('A')); // a
System.out.println(Character.isLetterOrDigit('3')); // true
```

## 3. Primitives vs Wrappers
When to Use Which?

| Use **primitives** when... | Use **wrappers** when... |
|----------------------------|--------------------------|
| Performance/memory matters (no object overhead) | You need them in generics/collections (`List<Integer>`) |
| The value always exists | The value can be **absent** (`null`) |
| Local computation, loops, counters | API/DTO fields where "not set" is meaningful |

> **Default rule:** prefer **primitives** for locals and computations; use wrappers only when you must (collections, nullability). Avoid unnecessary boxing in hot paths.

## 4. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Comparing wrappers with `==` | Use `.equals()` (Integer cache trap) |
| Unboxing a possibly-`null` wrapper | Null-check first / avoid wrappers where value always exists |
| Using `float`/`double` for money | Use `BigDecimal` |
| Forgetting `L`/`f` suffixes | `long` needs `L`; `float` needs `f` |
| Silent integer overflow | Use `long` or `Math.*Exact()` |
| Leading `0` making an octal literal | `010` is 8, not 10 |
| Using uninitialized local variables | Initialize before use (no auto-default for locals) |
| Boxing in tight loops | Use primitives for accumulators |

## 5. Connection to Backend / Spring (Why This Matters Later)

- **DTO/entity fields** often use wrappers (`Integer`, `Long`) so "not provided"/"null in DB" is representable — but watch for NPEs on unboxing.
- **JPA IDs** are typically `Long` (wrapper) so a new, unsaved entity can have a `null` id.
- **`BigDecimal` for money** is mandatory in financial/e-commerce backends (Projects 5, 7).
- **Integer cache `==` bug** appears in real code comparing IDs/counts — use `.equals()`.
- **Boxing performance** matters in high-throughput services and stream pipelines (use `IntStream`/`LongStream` — Phase 1.8).

## 6. Quick Self-Check Questions

1. Name the 8 primitives with their sizes. Which is unsigned?
2. What are the default values of fields vs local variables?
3. Why do wrapper classes exist? Give two reasons.
4. What's the difference between autoboxing and unboxing? What NPE trap does unboxing create?
5. Why does `Integer a=100,b=100; a==b` print `true` but `200==200` print `false`?
6. How should you always compare wrapper objects, and why?
7. Why must you never use `double` for currency?
8. When should you prefer primitives over wrappers?

## 7. Key Terms

- **Primitive type:** built-in type holding a value directly (8 of them).
- **Reference type:** type holding a reference to a heap object.
- **Wrapper class:** object form of a primitive (`Integer`, `Double`, ...).
- **Autoboxing / unboxing:** automatic primitive↔wrapper conversion.
- **Integer cache:** pre-created `Integer` objects for −128..127.
- **Literal:** a fixed value in source code (`100`, `3.14f`, `'A'`).
- **Overflow:** value exceeding a type's range, wrapping around.
- **`BigDecimal`:** arbitrary-precision decimal for exact (money) math.
- **Default value:** auto-assigned value for uninitialized fields.