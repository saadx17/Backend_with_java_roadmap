# Type Casting
Type casting is **converting a value from one type to another**. There are 2 types of type casting.
1. Implicit Casting (Widening)
2. Explicit Casting (Narrowing)

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
```
Widening (Implicit) ──────────────────────────────────→
byte → short → int → long → float → double ←────────────────────────────────── Narrowing (Explicit)
```

|Type|Safe?|Syntax|Risk|
|---|---|---|---|
|Implicit (widening)|✅ Yes|Automatic|None|
|Explicit (narrowing)|⚠️ Maybe|`(type) value`|Data loss|

# Variable Naming Conventions
Java follows specific naming conventions. These are **not enforced by the compiler** but are **industry standards every Java developer follows**.
##### ==camelCase==
Used for variables. *Start with **lowercase**, capitalize each subsequent word.* camelCase (specifically `lowerCamelCase`) is a naming convention where the first word is entirely lowercase and each subsequent, compound word begins with a capital letter, with no spaces or punctuation, used for variables and methods to improve code readability. Examples include `firstName`, `calculateTotal`

```
// ✅ Correct
int age = 25;
String firstName = "Ahmed";
double monthlyIncome = 5000.0;
boolean isUserLoggedIn = true;
int numberOfStudents = 30;

// ❌ Wrong
int Age = 25; // No - looks like a class name

int first_name = 0; // No - this is snake_case (Python style, not Java)

int FIRSTNAME = 0; // No - this is for constants
```

##### ==PascalCase== (UpperCamelCase)
Used for classes. *Start with **uppercase**, capitalize each word.*
PascalCase a naming convention where every word starts with a capital letter, including the first, with no spaces or separators (e.g. `ClassMemberExample`).

```
// ✅ Correct
class HelloWorld {}
class BankAccount {}
class UserProfileService {}
class HttpRequestHandler {}

// ❌ Wrong
class helloWorld {} // No
class hello_world {} // No
```


##### ==Constants==
*All **uppercase** with **underscores** between words.*
In Java, constants are defined using `SCREAMING_SNAKE_CASE` (all uppercase letters with underscores separating words) to indicate that the value is immutable and `static final`.

```
// ✅ Correct

final int MAX_SIZE = 100;
final double PI = 3.14159;
final String APP_NAME = "MyApp";
final int MAX_RETRY_ATTEMPTS = 3;
```

##### ==Packages==
Package names are *written in all lowercase* to ensure consistency, prevent conflicts with class/interface names, and avoid issues with case-sensitive filesystems.

```
// ✅ Correct
package com.company.project.service;
package org.example.utils;
```

##### Summary Table:

| What     | Convention           | Example                             |
| -------- | -------------------- | ----------------------------------- |
| Variable | camelCase            | `firstName`, `totalScore`           |
| Method   | camelCase            | `getUserName()`, `calculateTotal()` |
| Class    | PascalCase           | `BankAccount`, `UserService`        |
| Constant | SCREAMING_SNAKE_CASE | `MAX_SIZE`, `PI`                    |
| Package  | lowercase            | `com.example.app`                   |


##### Other naming rules:
- Must start with a **letter, `$`, or `_`** (never a digit)
- Cannot use **reserved keywords** (`int`, `class`, `static`, etc.)
- Case sensitive (`age` ≠ `Age` ≠ `AGE`)
- Make names **meaningful**, never `x`, `temp`, `data` unless context is obvious.

```
// ❌ Bad naming
int x = 25;
String s = "Ahmed";
double d = 5000.0;

// ✅ Good naming
int age = 25;
String userName = "Ahmed";
double monthlySalary = 5000.0;
```
