An **operator** performs an operation on one or more **operands** (values/variables) to produce a result. Java's operators fall into these groups:

| Group | Operators |
|-------|-----------|
| Arithmetic | `+` `-` `*` `/` `%` |
| Comparison (relational) | `==` `!=` `<` `>` `<=` `>=` |
| Logical | `&&` `||` `!` |
| Bitwise | `&` `|` `^` `~` `<<` `>>` `>>>` |
| Assignment | `=` `+=` `-=` `*=` ... |
| Unary | `+` `-` `++` `--` `!` `~` |
| Ternary | `? :` |
| Type | `instanceof` |

## 1. Arithmetic Operators
These behave exactly as they do in basic math, but the JVM has strict memory boundaries for primitives. Used to perform **mathematical calculations**.

| Operator | Name                | Example  | Result |
| -------- | ------------------- | -------- | ------ |
| `+`      | Addition            | `10 + 3` | `13`   |
| `-`      | Subtraction         | `10 - 3` | `7`    |
| `*`      | Multiplication      | `10 * 3` | `30`   |
| `/`      | Division            | `10 / 3` | `3`    |
| `%`      | Modulus (remainder) | `10 % 3` | `1`    |

```java title:arithmetic.java
int a = 10;
int b = 3;
System.out.println(a + b); // 13
System.out.println(a - b); // 7
System.out.println(a * b); // 30
System.out.println(a / b); // 3 ← NOT 3.333
System.out.println(a % b); // 1 ← remainder of 10 ÷ 3
```

### 1.1 Integer division & Modulus
When you divide two **integers**, Java **discards the decimal part** (truncates).
```java title:integer.java
int a = 10;
int b = 3;
System.out.println(a / b); // 3 ← decimal is thrown away
System.out.println(10 / 4); // 2 ← NOT 2.5
System.out.println(7 / 2); // 3 ← NOT 3.5
```

To get decimal result, at least one operand must be `double`/`float`.
```java title:decimal.java
System.out.println(10.0 / 3); // 3.3333333333333335
System.out.println(10 / 3.0); // 3.3333333333333335
System.out.println((double) a / b); // 3.3333333333333335 ← cast to double first
```

#### Modulus `%`
Returns the **remainder** after division. Very useful in real programming.
```java title:modulus.java
System.out.println(10 % 3); // 1 (10 = 3×3 + 1)
System.out.println(15 % 5); // 0 (15 = 5×3 + 0)
System.out.println(7 % 2); // 1 (7 = 2×3 + 1)
System.out.println(4 % 7); // 4 (4 = 7×0 + 4)
```

### 1.2 Increment / Decrement

| Operator | Name           | Meaning            |
| -------- | -------------- | ------------------ |
| `++x`    | Pre-increment  | Increment THEN use |
| `x++`    | Post-increment | Use THEN increment |
| `--x`    | Pre-decrement  | Decrement THEN use |
| `x--`    | Post-decrement | Use THEN decrement |

```java title:incdec.java
int x = 5;

// Post-increment
System.out.println(x++); // prints 5, THEN x becomes 6
System.out.println(x); // 6


// Pre-increment
int y = 5;
System.out.println(++y); // y becomes 6 FIRST, THEN prints 6
System.out.println(y); // 6
```

**The difference becomes clear here:**
```java title:diff_clear.java
int a = 5;
int b = a++; // b = 5, a = 6 (b gets the OLD value)

int c = 5;
int d = ++c; // d = 6, c = 6 (d gets the NEW value)

System.out.println("b=" + b + ", a=" + a); // b=5, a=6 System.out.println("d=" + d + ", c=" + c); // d=6, c=6
```
**Note:** When used alone on its own line, `x++` and `++x` are identical. The difference only matters when used **inside an expression**.

### 1.3 String concatenation with `+`
The `+` operator is **overloaded** for Strings, it concatenates.
```java title:string.java
String firstName = "Ahmed";
String lastName = "Ali";
String fullName = firstName + " " + lastName;
System.out.println(fullName); // Ahmed Ali

// + with numbers and Strings
System.out.println("Age: " + 25); // Age: 25
System.out.println("Sum: " + (10 + 5)); // Sum: 15
System.out.println("Sum: " + 10 + 5); // Sum: 105 ← GOTCHA! left to right
```

**Gotcha explained:**
```java title:gotcha.java
"Sum: " + 10 + 5
= ("Sum: " + 10) + 5 // left to right evaluation
= "Sum: 10" + 5 // String + int = String
= "Sum: 105" // NOT 15!
// Fix: use parentheses
"Sum: " + (10 + 5)
= "Sum: " + 15
= "Sum: 15"
```

## 2. Comparison (Relational) Operators
**Relational operators** in Java are binary operators used to compare two values or expressions. They always return a **boolean value (`true` or `false`)** based on whether the specified relationship holds true. These operators form the foundation of control flow statements like `if-else` blocks, `while` loops, and `for` loops.

|Operator|Meaning|Example|Result|
|---|---|---|---|
|`==`|Equal to|`5 == 5`|`true`|
|`!=`|Not equal to|`5 != 3`|`true`|
|`>`|Greater than|`5 > 3`|`true`|
|`<`|Less than|`5 < 3`|`false`|
|`>=`|Greater than or equal|`5 >= 5`|`true`|
|`<=`|Less than or equal|`3 <= 5`|`true`|

Return a `boolean`:
```java title:relational.java
int a = 10;
int b = 20;

System.out.println(a == b); // false
System.out.println(a != b); // true
System.out.println(a > b); // false
System.out.println(a < b); // true
System.out.println(a >= 10); // true
System.out.println(b <= 20); // true

// Result can be stored in a boolean
boolean isAdult = age >= 18;
boolean isEqual = a == b;
```

### 2.1 The `==` trap for objects
- For **primitives**, `==` compares values.
- For **objects**, `==` compares **references** (identity), not content - use `.equals()` for content (recall the Object methods + Integer cache notes)

```java title:values.java
// Primitives — == compares VALUES
int x = 5;
int y = 5;
System.out.println(x == y); // true
```

```java title:ref.java
// Strings — == compares REFERENCES
String a = new String("hello");
String b = new String("hello");
System.out.println(a == b); // false (different objects in memory)
System.out.println(a.equals(b)); // true (same content)
```
**Rule:** *Use `==` for primitives. Use `.equals()` for objects.*

## 3. Logical Operators
**Logical operators** in Java are symbols used to combine or invert boolean expressions. They form the backbone of decision-making structures like `if` statements and loops.

|Operator|Name|Meaning|Example|
|---|---|---|---|
|`&&`|AND|Both must be true|`a > 0 && b > 0`|
|`\|`|OR|At least one must be true|`a > 0 \| b > 0`|
|`!`|NOT|Inverts the boolean|`!isLoggedIn`|

**Operate on booleans:**
```java
a && b   // logical AND — short-circuits
a || b   // logical OR  — short-circuits
!a       // logical NOT
```

### 3.1 Short-circuit evaluation
Java is **lazy**, it stops evaluating as soon as the result is determined.
- `&&`: if the left is `false`, the right is **not evaluated** (result is already `false`).
- `||`: if the left is `true`, the right is **not evaluated**.

```java
if (obj != null && obj.isValid()) { ... }   // safe: isValid() only runs if obj != null

if (list.isEmpty() || list.get(0) == null) { ... }  // get(0) skipped if empty
```
> Short-circuiting is used constantly for **null-safety** and to avoid expensive/unsafe calls. The non-short-circuit `&`/`|` (below) always evaluate both sides. This is not just an optimization, it is used intentionally in real code to avoid null pointer exceptions and division by zero.

## 4. Bitwise Operators
**Bitwise operators** in Java are symbols that perform operations directly on the binary representations (individual bits, `0` and `1`) of integer data types like `byte`, `short`, `int`, `long`, and `char`. They evaluate values bit-by-bit to deliver high-performance computation required for competitive programming, cryptography, optimization, and low-level system communication.
```java title:bitwise.java
int a = 5;   // 0101
int b = 3;   // 0011

// & (AND)  → bit is 1 only if BOTH bits are 1
System.out.println(a & b);   // 0001 = 1

// | (OR)   → bit is 1 if EITHER bit is 1
System.out.println(a | b);   // 0111 = 7

// ^ (XOR)  → bit is 1 if bits are DIFFERENT
System.out.println(a ^ b);   // 0110 = 6

// ~ (NOT)  → flips all bits
System.out.println(~a);      // -(a+1) = -6
```

**`Bit` Shift Operators:**
```java title:shift.java
int a = 8;  // 00001000 in binary

// << Left shift  → multiply by 2^n  (fast multiplication)
System.out.println(a << 1);  // 00010000 = 16  (8 × 2)
System.out.println(a << 2);  // 00100000 = 32  (8 × 4)
System.out.println(a << 3);  // 01000000 = 64  (8 × 8)

// >> Right shift → divide by 2^n  (fast division)
System.out.println(a >> 1);  // 00000100 = 4   (8 / 2)
System.out.println(a >> 2);  // 00000010 = 2   (8 / 4)

// >>> Unsigned right shift → always fills with 0 from left
int negative = -8;
System.out.println(negative >> 1);   // -4  (preserves sign)
System.out.println(negative >>> 1);  // large positive (fills 0)
```

### 4.1 Common uses
```java
n << 1;        // multiply by 2     (fast)
n >> 1;        // divide by 2
(n & 1) == 0;  // check if n is even (last bit)
flags |= FLAG; // set a flag bit
flags &= ~FLAG;// clear a flag bit
flags & FLAG;  // test a flag bit
```

### 4.2 `>>` vs `>>>` (the signed/unsigned distinction)
```java
-8 >> 1;    // -4  (signed: fills with the sign bit, 1s)
-8 >>> 1;   // 2147483644  (unsigned: fills with 0s) — only meaningful for the bit pattern
```
> `>>>` exists *because* of two's-complement signed integers (Phase 0). Java has no unsigned types, so `>>>` gives you logical shifting.

### 4.3 `&`/`|` as non-short-circuit logical operators
On booleans, `&` and `|` work like `&&`/`||` but **always evaluate both sides** (no short-circuit). Rarely needed; usually a bug if you meant `&&`/`||`.

## 5. Assignment Operators
An **assignment operator** in Java is used to assign a value to a variable. The most common assignment operator is the single equal sign (`=`), which copies the value from its right-hand side to the variable on its left-hand side.

|Operator|Equivalent To|Example|Result (if x=10)|
|---|---|---|---|
|`=`|assign|`x = 5`|x = 5|
|`+=`|`x = x + n`|`x += 3`|x = 13|
|`-=`|`x = x - n`|`x -= 3`|x = 7|
|`*=`|`x = x * n`|`x *= 3`|x = 30|
|`/=`|`x = x / n`|`x /= 3`|x = 3|
|`%=`|`x = x % n`|`x %= 3`|x = 1|

```java title:assignment.java
int x = 10;

x += 5; // x = x + 5 → x = 15
x -= 3; // x = x - 3 → x = 12
x *= 2; // x = x * 2 → x = 24
x /= 4; // x = x / 4 → x = 6
x %= 4; // x = x % 4 → x = 2

System.out.println(x); // 2
```

### 5.1 Compound assignment does an implicit cast
```java
byte b = 10;
// b = b + 5;   // COMPILE ERROR: b+5 is int, needs explicit cast
b += 5;         // OK! compound assignment implicitly casts back to byte
```
> `+=` etc. silently narrow the result back to the variable's type — convenient but can hide truncation/overflow.

## 6. The Ternary Operator (`? :`)
The **ternary operator** (`?:`) in Java is a shorthand, single-line alternative to traditional `if-else` statements. It is the only operator in Java that accepts **three operands**: a boolean condition, a value for when that condition is true, and a value for when it is false.
A compact conditional **expression** that returns a value:
```java title:syntax.java
condition ? valueIfTrue : valueIfFalse
```

```java title:ternary.java
int age = 20;

// Normal if-else
String status;
if (age >= 18) {
    status = "adult";
} else {
    status = "minor";
}

// Ternary → same thing, one line
String status = (age >= 18) ? "adult" : "minor";
System.out.println(status);  // adult
```
- Unlike `if` (a **statement**), the ternary is an **expression** (produces a value).
- Both branches must produce **compatible types**.
- Don't nest deeply - it hurts readability.

## 7. The `instanceof` Operator
Tests whether an object is an instance of a given type (returns `boolean`):
```java
Object o = "hello";
if (o instanceof String) { ... }   // true
o instanceof Integer;              // false
null instanceof String;            // false (null is never an instance of anything)
```

### 7.1 Pattern matching for `instanceof` (Java 16+)
Test and bind in one step - no separate cast (recall Polymorphism note):
```java
if (o instanceof String s) {       // checks AND binds 's' if true
    System.out.println(s.length()); // use s directly, no cast
}
```
This replaces the old verbose `if (o instanceof String) { String s = (String) o; ... }`.

## 8. Operator Precedence
When multiple operators appear in one expression, Java follows **precedence rules** (like BODMAS/PEMDAS in math).

```
Higher precedence (evaluated first) ───────────────────────────────────
1. () Parentheses
2. ++ -- Post/Pre increment
3. ! Logical NOT
4. * / % Multiplication, Division, Modulus
5. + - Addition, Subtraction
6. > < >= <= Comparison
7. == != Equality
8. && Logical AND
9. || Logical OR
10. = += -= *= /= %= Assignment
───────────────────────────────────
Lower precedence (evaluated last)
```

```java title:example.java
int result = 2 + 3 * 4; // 14, not 20 (* before +)
int result2 = (2 + 3) * 4; // 20 (parentheses first)

boolean r = 5 > 3 && 10 < 20; // true && true = true
boolean r2 = 5 + 3 > 6 && !false; // 8 > 6 && true = true && true = true
```
> **When in doubt, use parentheses.** They make intent explicit and prevent precedence bugs.

## 9. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Integer division truncating (`5/2 == 2`) | Make an operand a double: `5.0/2` |
| Integer `/ 0` throwing vs float giving `Infinity` | Know the difference; guard divisors |
| `==` comparing object references | Use `.equals()` for content |
| Confusing pre/post increment | Be explicit; avoid combining with other ops |
| Using `&`/`|` where `&&`/`||` intended | Use short-circuit versions for booleans |
| `>>` vs `>>>` on negative numbers | Use `>>>` for logical/unsigned shifting |
| Precedence surprises | Use parentheses |
| Compound assignment hiding narrowing | Be aware `+=` casts implicitly |

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Short-circuit `&&`/`||`** is essential for **null-safe** checks throughout backend code.
- **`.equals()` vs `==`** matters constantly for IDs, strings, and enums (recall Integer cache).
- **Bitwise operators** appear in flags/permissions, hashing, Bloom filters, and `EnumSet` internals (Phase 2, 16).
- **Pattern-matching `instanceof`** cleans up type handling and deserialization logic (modern Java).
- **Ternary** keeps DTO mapping and defaulting concise.
- **`BigDecimal`** (not `/`) for money arithmetic to avoid float errors (Phase 4/5).

## 11. Quick Self-Check Questions

1. What does `5 / 2` evaluate to, and how do you get `2.5`?
2. What's the difference between integer and floating-point division by zero?
3. Explain short-circuit evaluation and give a null-safety example.
4. What's the difference between `>>` and `>>>`?
5. How do you set, clear, and test a flag bit with bitwise operators?
6. What's the difference between `x++` and `++x`?
7. Why does `byte b = 10; b += 5;` compile but `b = b + 5;` not?
8. How does pattern-matching `instanceof` improve on the classic form?
9. What does `2 + 3 * 4` evaluate to, and why?

## 12. Key Terms Glossary

- **Operator / operand:** the action / the value(s) it acts on.
- **Arithmetic operators:** `+ - * / %`.
- **Integer division:** division truncating the fractional part.
- **Modulus (`%`):** remainder operator.
- **Short-circuit evaluation:** `&&`/`||` skipping the right side when the result is determined.
- **Bitwise operators:** operate on individual bits.
- **Signed/unsigned shift:** `>>` (sign-extending) vs `>>>` (zero-fill).
- **Compound assignment:** `+=`, `-=`, etc. (with implicit cast).
- **Ternary operator:** `? :` conditional expression.
- **`instanceof`:** runtime type test (with optional pattern binding).
- **Operator precedence:** the order operators are evaluated.