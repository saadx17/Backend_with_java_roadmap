Java follows specific naming conventions. These are **not enforced by the compiler** but are **industry standards every Java developer follows**.

| What     | Convention           | Example                             |
| -------- | -------------------- | ----------------------------------- |
| Variable | camelCase            | `firstName`, `totalScore`           |
| Method   | camelCase            | `getUserName()`, `calculateTotal()` |
| Class    | PascalCase           | `BankAccount`, `UserService`        |
| Constant | SCREAMING_SNAKE_CASE | `MAX_SIZE`, `PI`                    |
| Package  | lowercase            | `com.example.app`                   |

#### camelCase
Used for variables. *Start with **lowercase**, capitalize each subsequent word.* camelCase (specifically `lowerCamelCase`) is a naming convention where the first word is entirely lowercase and each subsequent, compound word begins with a capital letter, with no spaces or punctuation, used for variables and methods to improve code readability. Examples include `firstName`, `calculateTotal`

```java title:Hello.java
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

#### PascalCase (UpperCamelCase)
Used for classes. *Start with **uppercase**, capitalize each word.*
PascalCase a naming convention where every word starts with a capital letter, including the first, with no spaces or separators (e.g. `ClassMemberExample`).

```java title:Hello.java
// ✅ Correct
class HelloWorld {}
class BankAccount {}
class UserProfileService {}
class HttpRequestHandler {}

// ❌ Wrong
class helloWorld {} // No
class hello_world {} // No
```


#### Constants
*All **uppercase** with **underscores** between words.*
In Java, constants are defined using `SCREAMING_SNAKE_CASE` (all uppercase letters with underscores separating words) to indicate that the value is immutable and `static final`.

```java title:constants.java
// ✅ Correct

final int MAX_SIZE = 100;
final double PI = 3.14159;
final String APP_NAME = "MyApp";
final int MAX_RETRY_ATTEMPTS = 3;
```

#### Packages
Package names are *written in all lowercase* to ensure consistency, prevent conflicts with class/interface names, and avoid issues with case-sensitive filesystems.

```java title:Hello.java
// ✅ Correct
package com.company.project.service;
package org.example.utils;
```



#### Other naming rules
- Must start with a **letter, `$`, or `_`** (never a digit)
- Cannot use **reserved keywords** (`int`, `class`, `static`, etc.)
- Case sensitive (`age` ≠ `Age` ≠ `AGE`)
- Make names **meaningful**, never `x`, `temp`, `data` unless context is obvious.

```cpp title:Nameing_rule.java
// ❌ Bad naming
int x = 25;
String s = "Ahmed";
double d = 5000.0;

// ✅ Good naming
int age = 25;
String userName = "Ahmed";
double monthlySalary = 5000.0;
```