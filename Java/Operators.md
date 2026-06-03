An **operator** is a symbol that performs an operation on one or more values (called **operands**).
```
Operators tell Java WHAT TO DO with values

Types:
├── Arithmetic   → math operations
├── Assignment   → assigns values
├── Relational   → compare values
├── Logical      → combine conditions
├── Bitwise      → work on binary bits
└── Ternary      → shorthand if-else
```

# `Arithmetic` Operators
These behave exactly as they do in basic math, but the JVM has strict memory boundaries for primitives. Used to perform **mathematical calculations**.

|Operator|Name|Example|Result|
|---|---|---|---|
|`+`|Addition|`10 + 3`|`13`|
|`-`|Subtraction|`10 - 3`|`7`|
|`*`|Multiplication|`10 * 3`|`30`|
|`/`|Division|`10 / 3`|`3`|
|`%`|Modulus (remainder)|`10 % 3`|`1`|

```java title:arithmetic.java
int a = 10;
int b = 3;
System.out.println(a + b); // 13
System.out.println(a - b); // 7
System.out.println(a * b); // 30
System.out.println(a / b); // 3 ← NOT 3.333
System.out.println(a % b); // 1 ← remainder of 10 ÷ 3
```

##### Integer Division
When you divide two **integers**, Java **discards the decimal part** (truncates).
```java title:integer.java
int a = 10;
int b = 3;
System.out.println(a / b); // 3 ← decimal is thrown away
System.out.println(10 / 4); // 2 ← NOT 2.5
System.out.println(7 / 2); // 3 ← NOT 3.5
```

```java title:integer.java
// To get decimal result, at least one operand must be double/float
System.out.println(10.0 / 3); // 3.3333333333333335
System.out.println(10 / 3.0); // 3.3333333333333335
System.out.println((double) a / b); // 3.3333333333333335 ← cast to double first
```

##### Modulus `%`
Returns the **remainder** after division. Very useful in real programming.
```java title:modulus.java
System.out.println(10 % 3); // 1 (10 = 3×3 + 1)
System.out.println(15 % 5); // 0 (15 = 5×3 + 0)
System.out.println(7 % 2); // 1 (7 = 2×3 + 1)
System.out.println(4 % 7); // 4 (4 = 7×0 + 4)
```

**Real-world uses of `%`:**
```java title:remainder.java
// Check if a number is even or odd
int number = 17;
if (number % 2 == 0) {
   System.out.println("Even");
}
else {
   System.out.println("Odd"); // prints this
}

// Check if divisible by something
int n = 100;
if (n % 25 == 0) {
  System.out.println("Divisible by 25"); // prints this
}

// Wrap around (circular behavior)
int index = 7 % 5; // = 2 (useful in circular arrays or clock logic)

// Extract last digit of a number
int num = 12345;
int lastDigit = num % 10; // = 5
```

##### Increment & Decrement Operators
|Operator|Name|Meaning|
|---|---|---|
|`++x`|Pre-increment|Increment THEN use|
|`x++`|Post-increment|Use THEN increment|
|`--x`|Pre-decrement|Decrement THEN use|
|`x--`|Post-decrement|Use THEN decrement|

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

The difference becomes clear here:
```
int a = 5;
int b = a++; // b = 5, a = 6 (b gets the OLD value)

int c = 5;
int d = ++c; // d = 6, c = 6 (d gets the NEW value)

System.out.println("b=" + b + ", a=" + a); // b=5, a=6 System.out.println("d=" + d + ", c=" + c); // d=6, c=6
```

**Note:** When used alone on its own line, `x++` and `++x` are identical. The difference only matters when used **inside an expression**.

##### String Concatenation with `+`
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
```
"Sum: " + 10 + 5
= ("Sum: " + 10) + 5 // left to right evaluation
= "Sum: 10" + 5 // String + int = String
= "Sum: 105" // NOT 15!
// Fix: use parentheses
"Sum: " + (10 + 5)
= "Sum: " + 15
= "Sum: 15"
```

# `Assignment` Operators
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

**Real-world use:**
```java title:assignment.java
double total = 0;
total += 29.99; // add item 1
total += 49.99; // add item 2
total += 9.99; // add item 3
System.out.println("Total: " + total); // Total: 89.97

int score = 100;
score -= 10; // penalty
score += 25; // bonus
System.out.println("Score: " + score); // Score: 115
```

# `Relational` Operators
**Relational operators** in Java are binary operators used to compare two values or expressions. They always return a **boolean value (`true` or `false`)** based on whether the specified relationship holds true. These operators form the foundation of control flow statements like `if-else` blocks, `while` loops, and `for` loops.

|Operator|Meaning|Example|Result|
|---|---|---|---|
|`==`|Equal to|`5 == 5`|`true`|
|`!=`|Not equal to|`5 != 3`|`true`|
|`>`|Greater than|`5 > 3`|`true`|
|`<`|Less than|`5 < 3`|`false`|
|`>=`|Greater than or equal|`5 >= 5`|`true`|
|`<=`|Less than or equal|`3 <= 5`|`true`|

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

##### ==`==`== with Objects
`==` compares **references** (memory addresses) for objects, not content.
```java title:references.java
// Primitives — == compares VALUES
int x = 5;
int y = 5;
System.out.println(x == y); // true

// Strings — == compares REFERENCES
String a = new String("hello");
String b = new String("hello");
System.out.println(a == b); // false (different objects in memory)
System.out.println(a.equals(b)); // true (same content)
```

**Rule:** *Use `==` for primitives. Use `.equals()` for objects.*

# `Logical` Operators
**Logical operators** in Java are symbols used to combine or invert boolean expressions. They form the backbone of decision-making structures like `if` statements and loops.

|Operator|Name|Meaning|Example|
|---|---|---|---|
|`&&`|AND|Both must be true|`a > 0 && b > 0`|
|`\|`|OR|At least one must be true|`a > 0 \| b > 0`|
|`!`|NOT|Inverts the boolean|`!isLoggedIn`|

##### AND `&&`
Both conditions must be true.
```java title:and.java
int age = 25;
double salary = 60000;

boolean canGetLoan = age >= 18 && salary >= 50000; System.out.println(canGetLoan); // true (both are true)
boolean test1 = true && true; // true
boolean test2 = true && false; // false
boolean test3 = false && true; // false
boolean test4 = false && false; // false
```

**Truth table for `&&`:**
```
A       B     A && B
─────────────────────
true   true    true
true   false  false
false  true   false
false  false  false
```

##### OR `||`
At least one condition must be true.
```java title:or.java
boolean hasDriverLicense = false;
boolean hasPassport = true;
boolean hasValidId = hasDriverLicense || hasPassport;
System.out.println(hasValidId); // true (at least one is true)

boolean test1 = true || true; // true
boolean test2 = true || false; // true
boolean test3 = false || true; // true
boolean test4 = false || false; // false
```

**Truth table for `||`:**
```
A        B      A || B
──────────────────────
true    true     true
true    false    true
false    true    true
false   false    false
```

##### NOT `!`
Flips the boolean.
```java title:not.java
boolean isLoggedIn = false;

System.out.println(!isLoggedIn); // true System.out.println(!true); // false
System.out.println(!false); // true

// Real use case boolean isWeekend = true; if (!isWeekend) { System.out.println("Go to work"); } else { System.out.println("Rest!"); // prints this
}
```

##### Short-Circuit Evaluation
Java is **lazy**, it stops evaluating as soon as the result is determined.
```java title:short_circuit.java
// AND short-circuit: if first is false, second is NOT evaluated
int x = 0;
if (x != 0 && 10 / x > 1) { // 10/x would cause divide by zero!
   System.out.println("ok");
   }
// Safe! Because x != 0 is false → Java skips 10/x entirely


// OR short-circuit: if first is true, second is NOT evaluated
boolean result = true || someExpensiveMethod();

// someExpensiveMethod() is NEVER called because first is already true
```

**Note:** This is not just an optimization, it is used intentionally in real code to avoid null pointer exceptions and division by zero.

```java title:sc.java
// Common real-world pattern — null check before using object
String name = null;
if (name != null && name.length() > 0) { // Safe!
   System.out.println(name.toUpperCase());
}
// If name is null → name != null is false → name.length() is NEVER called → no crash
```

##### Combining Logical Operators
```java title:comb.java
int age = 20;
boolean hasTicket = true;
boolean isVIP = false;

// Can enter if: has ticket AND (is adult OR is VIP)
boolean canEnter = hasTicket && (age >= 18 || isVIP);
System.out.println(canEnter); // true

// Use parentheses to control evaluation order — never rely on precedence alone
boolean a = true || false && false; // true (&& before ||)
boolean b = (true || false) && false; // false (parentheses first)
```


# `Bitwise` Operators
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

##### `Bit` Shift Operators
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

**`Bitwise` Real World Uses:**
```java title:realworld.java
// ─── 1. Check if number is even or odd (faster than %) ───
int n = 7;
if ((n & 1) == 1) {
    System.out.println("odd");   // last bit is 1 → odd
} else {
    System.out.println("even");  // last bit is 0 → even
}

// ─── 2. Multiply/Divide by 2 without * or / ──────────────
int x = 5;
System.out.println(x << 1);  // 10  (x * 2)
System.out.println(x >> 1);  // 2   (x / 2)

// ─── 3. Swap without temp variable ───────────────────────
int p = 5, q = 9;
p = p ^ q;   // p = 5^9
q = p ^ q;   // q = 5^9^9 = 5
p = p ^ q;   // p = 5^9^5 = 9
System.out.println(p + " " + q);  // 9 5

// ─── 4. Check if bit is set (used in flags/permissions) ──
int permissions = 0b1010;  // binary literal
int READ_BIT    = 0b0010;

if ((permissions & READ_BIT) != 0) {
    System.out.println("Has read permission");  // ✅
}

// ─── 5. XOR to find unique number (famous interview problem)
// "Find the number that appears once, others appear twice"
int[] arr = {2, 3, 5, 3, 2};
int unique = 0;
for (int num : arr) {
    unique ^= num;  // duplicates cancel out (n^n = 0)
}
System.out.println(unique);  // 5
```


# `Ternary` Operator
The **ternary operator** (`?:`) in Java is a shorthand, single-line alternative to traditional `if-else` statements. It is the only operator in Java that accepts **three operands**: a boolean condition, a value for when that condition is true, and a value for when it is false.

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

**Real Example:**
```java title:terop.java
// Find maximum of two numbers
int a = 10, b = 20;
int max = (a > b) ? a : b;
System.out.println(max);  // 20

// Absolute value
int num = -5;
int abs = (num < 0) ? -num : num;
System.out.println(abs);  // 5

// Null check
String name = null;
String display = (name != null) ? name : "Anonymous";
System.out.println(display);  // Anonymous

// In print statements
int score = 75;
System.out.println("Result: " + (score >= 50 ? "PASS" : "FAIL"));

// Nested ternary (readable limit)
int marks = 85;
String grade = (marks >= 90) ? "A" :
               (marks >= 80) ? "B" :
               (marks >= 70) ? "C" : "F";
System.out.println(grade);  // B
```

**Note:** Don't over-nest, kills readability. Use if-else when logic gets complex.

# Operator `Precedence`
When multiple operators appear in one expression, Java follows **precedence rules** (like BODMAS/PEMDAS in math).

```java title:prec.java
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
10. = += -= *= /= %= Assignment ───────────────────────────────────
Lower precedence (evaluated last)
```

**Example:**
```java title:prec2.java
int result = 2 + 3 * 4; // 14, not 20 (* before +)
int result2 = (2 + 3) * 4; // 20 (parentheses first)

boolean r = 5 > 3 && 10 < 20; // true && true = true
boolean r2 = 5 + 3 > 6 && !false; // 8 > 6 && true = true && true = true
```

**Note:** Use **parentheses** to make your intent explicit. Don't rely on memorizing precedence - it makes code harder to read.
