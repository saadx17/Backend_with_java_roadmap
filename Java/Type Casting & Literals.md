# Type Casting
Type casting is **converting a value from one type to another**. There are 2 types of type casting.
1. [[#==Implicit== Casting|Implicit]] Casting (Widening)
2. [[#==Explicit== Casting|Explicit]] Casting (Narrowing)

##### ==Implicit== Casting
Java **automatically** converts a **smaller type to a larger type**.  
No data is lost. No special syntax needed.

```
byte → short → int → long → float → double (widening direction →)
```

**Syntax:**
```
int myInt = 100;
long myLong = myInt; // int → long (automatic, safe)
double myDouble = myLong; // long → double (automatic, safe)

System.out.println(myInt); // 100
System.out.println(myLong); // 100
System.out.println(myDouble); // 100.0
```

**Another example:**
```
byte b = 42;
short s = b; // byte → short (implicit)
int i = s; // short → int (implicit)
long l = i; // int → long (implicit)
float f = l; // long → float (implicit)
double d = f; // float → double (implicit)
```

##### ==Explicit== Casting
**Manually** converting a **larger type to a smaller type**.  
You must tell Java explicitly. **Data loss can occur.**
```
double → float → long → int → short → byte (narrowing direction ←)
```

**Syntax: `(targetType) value`**
```
double myDouble = 9.99;
int myInt = (int) myDouble; // Explicit cast required 

System.out.println(myDouble); // 9.99
System.out.println(myInt); // 9 ← decimal part is TRUNCATED (not rounded)
```

**More examples:**
```
long bigNumber = 1234567890123L;
int smallNumber = (int) bigNumber; // Data LOSS occurs System.out.println(smallNumber); // Some garbage value (overflow)

double price = 19.99;
int truncated = (int) price;
System.out.println(truncated); // 19 (not 20 — it truncates, not rounds)

// char ↔ int casting
char ch = 'A';
int ascii = (int) ch;
System.out.println(ascii); // 65

int num = 66;
char letter = (char) num;
System.out.println(letter); // B
```

##### Casting Summary:

|Type|Safe?|Syntax|Risk|
|---|---|---|---|
|Implicit (widening)|✅ Yes|Automatic|None|
|Explicit (narrowing)|⚠️ Maybe|`(type) value`|Data loss|

# Type Conversion vs. Type Casting
In Java, both **Type Conversion** and **Type Casting** involve changing a variable from one data type to another. The difference comes down to who does the work (the compiler or you) and whether there is a risk of losing data.

##### 1. Type Conversion (Implicit / Widening)
Type conversion happens **automatically** by the Java compiler. It occurs when you assign a value from a smaller data type to a larger data type.
Because you are putting a small item into a bigger box, there is zero risk of the data overflowing or getting lost. The compiler handles it silently behind the scenes.

- **Direction:** `byte` → `short` → `int` → `long` → `float` → `double`
- **Safety:** 100% safe. No data is lost.

**Example:**
```
int smallBox = 100;
// Automatic conversion: putting a 32-bit int into a 64-bit double
double bigBox = smallBox; 

System.out.println(bigBox); // Prints: 100.0
```

##### 2. Type Casting (Explicit / Narrowing)
Type casting is a **manual** process. It is required when you want to assign a value from a larger data type to a smaller data type.
Because you are trying to cram a large item into a smaller box, there is a real risk that the data won't fit, resulting in **data loss** (like chopping off decimals). The Java compiler refuses to do this automatically because it wants you to confirm that you know what you are doing. You force it by putting the target data type in parentheses `( )` before the value.

- **Direction:** `double` → `float` → `long` → `int` → `short` → `byte`
- **Safety:** Risky. Decimals will be truncated (cut off), and numbers too large for the new type will overflow into completely wrong values.

**Example:**
```
double bigBox = 99.99;

// Manual casting: forcing a 64-bit double into a 32-bit int
// int smallBox = bigBox; <-- This causes a compiler ERROR
int smallBox = (int) bigBox; // <-- This works, but truncates the data

System.out.println(smallBox); // Prints: 99 (The .99 is permanently lost)
```

##### Comparison
|**Feature**|**Type Conversion (Widening)**|**Type Casting (Narrowing)**|
|---|---|---|
|**Who does it?**|Automatic (Java Compiler)|Manual (The Programmer)|
|**Size change**|Smaller type to Larger type|Larger type to Smaller type|
|**Data Loss Risk**|None|High (Decimals lost, overflow possible)|
|**Syntax Required**|None|`(targetType) variable`|

# Literals
In Java, a **literal** is the actual, raw data or fixed value that you type directly into your source code to assign to a variable. You aren't calculating it or pulling it from a database; you are hardcoding the exact value.

#### 1. Integer Literals
By default, any whole number typed into Java is treated as a standard base-10 (decimal) `int`. However, Java allows you to represent whole numbers in several different number systems using specific prefixes.

1. **Decimal (Base 10):** The standard numbers we use every day.
- `int standard = 10;`

2. **Hexadecimal (Base 16):** Starts with `0x` or `0X`. Commonly used in memory addressing and color codes.
- `int hex = 0x1A;` _(1A in hex equals 26 in decimal)_

3. **Binary (Base 2):** Starts with `0b` or `0B`. Useful for low-level bitwise operations.
- `int binary = 0b101;` _(101 in binary equals 5 in decimal)_

4. **The Underscore Feature:** To make large numbers easier to read, Java allows you to place underscores between digits. The compiler completely ignores them.
- `int oneMillion = 1_000_000;`

#### 2. Floating-Point Literals
These are numbers that contain a decimal point. In Java, any decimal number is treated as a `double` (which takes up 64 bits of memory) by default.

1. **Float Literals (`f` or `F`):** If you specifically want to save memory and use a 32-bit `float`, you **must** append an `f` to the end of the number. Otherwise, the compiler will throw an error because it thinks you are trying to cram a massive 64-bit `double` into a smaller 32-bit `float` box.
- `float pi = 3.14f;`

2. **Double Literals:** Because `double` is the default, you can just write the number. (You can append a `d` or `D`, but it's rarely done).
- `double precisePi = 3.14159265359;`

#### 3. Character Literals
A character literal represents a single character and must always be enclosed in **single quotes** (`' '`).

1. **Standard Characters:**
- `char letter = 'A';`

2. **Escape Sequences:** These are special character combinations that start with a backslash (`\`) to represent things you can't easily type, like formatting commands.
- `'\n'` : Newline (drops the cursor to the next line)
- `'\t'` : Tab (inserts a horizontal tab space)
- `'\\'` : Backslash (allows you to print an actual backslash)

3. **Unicode Characters:** You can represent any character from any language using its Unicode value by starting with `\u`.
- `char omega = '\u03A9';` _(Prints the Greek letter Ω)_

#### 4. String Literals
While `String` is technically a reference data type, Java allows you to create them using literals for convenience. A string literal must always be enclosed in **double quotes** (`" "`).
- `String message = "Hello";`

Because it relies on double quotes, if you ever need to print a literal quote inside your text, you must use an escape sequence:
- `String quote = "He said, \"Java is fun!\"";`

#### 5. Boolean Literals
Boolean literals are the simplest. They represent the concept of truth and can only ever be one of two values. They must be entirely lowercase and are not surrounded by quotes.
- `boolean isOnline = true;`
- `boolean hasError = false;`
