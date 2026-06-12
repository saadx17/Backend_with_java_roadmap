A variable is a named **container that ==stores data== in memory**.

```java title:age.java
int age = 25
```

Every variable in Java has:
- A **type** → what kind of data it holds.
- A **name** → how you refer to it.
- A **value** → the actual data stored.

### Declaration & Initialization
You can create and fill a variable in two separate steps, or do it all at once.

###### 1. Declaration (Creating the box)
This is when you tell the Java compiler the variable's name and what data type it will hold. Once declared, the variable exists in memory, but it is technically empty.
```java title:Hello.java
int playerScore;   // Declaring a variable that will hold an integer
String playerName; // Declaring a variable that will hold text
```

###### 2. Initialization (Filling the box)
This is the very first time you assign a value to a declared variable using the equals sign (`=`), which is called the assignment operator.
```java title:Hello.java
playerScore = 0;       // Initializing the integer variable
playerName = "Mario";  // Initializing the String variable
```

##### 3. The All-In-One (Declaration + Initialization)
In most cases, it is cleaner and safer to do both at the same time on a single line.
```java title:Hello.java
int playerScore = 0;
String playerName = "Mario";
```

### Variable Naming Conventions
Java follows specific naming conventions. These are **not enforced by the compiler** but are **industry standards every Java developer follows**.

| What     | Convention           | Example                             |
| -------- | -------------------- | ----------------------------------- |
| Variable | camelCase            | `firstName`, `totalScore`           |
| Method   | camelCase            | `getUserName()`, `calculateTotal()` |
| Class    | PascalCase           | `BankAccount`, `UserService`        |
| Constant | SCREAMING_SNAKE_CASE | `MAX_SIZE`, `PI`                    |
| Package  | lowercase            | `com.example.app`                   |

##### ==camelCase==
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

##### ==PascalCase== (UpperCamelCase)
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


##### ==Constants==
*All **uppercase** with **underscores** between words.*
In Java, constants are defined using `SCREAMING_SNAKE_CASE` (all uppercase letters with underscores separating words) to indicate that the value is immutable and `static final`.

```java title:constants.java
// ✅ Correct

final int MAX_SIZE = 100;
final double PI = 3.14159;
final String APP_NAME = "MyApp";
final int MAX_RETRY_ATTEMPTS = 3;
```

##### ==Packages==
Package names are *written in all lowercase* to ensure consistency, prevent conflicts with class/interface names, and avoid issues with case-sensitive filesystems.

```java title:Hello.java
// ✅ Correct
package com.company.project.service;
package org.example.utils;
```



##### Other naming rules
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

## Instance, Local & Static variables
|**Feature**|**Local Variable**|**Instance Variable**|**Static (Class) Variable**|
|---|---|---|---|
|**Where is it declared?**|Inside a method, loop, or block|Inside a class, outside any method|Inside a class, using `static` keyword|
|**What does it belong to?**|The specific block of code|A specific Object (instance)|The Class itself (shared by all objects)|
|**Memory Location**|Stack Memory|Heap Memory|Method Area|
|**Default Value?**|No (Must be initialized manually)|Yes (`0`, `false`, `null`)|Yes (`0`, `false`, `null`)|
|**Lifetime**|Destroyed when method ends|Destroyed when Object is garbage collected|Destroyed when program ends|

#### 1. Local Variables
Local variables are short-lived, temporary storage used for immediate calculations.

- **Scope:** They only exist inside the specific `{ }` block, method, or loop where they are declared. The outside world (even other methods in the same class) cannot see them.

- **Initialization:** The compiler enforces strict rules here. A local variable **will not** be given a default value. If you try to use it before assigning a value, the code will not compile.

- **Memory:** Stored in the thread's **Stack** memory.

```java title:local_variable.java
public class Calculator {
    public void addNumbers() {
        int result; // Declared, but not initialized
        
        // System.out.println(result); <-- ERROR: variable result might not have been initialized
        
        result = 10 + 5; 
        System.out.println(result); // Works perfectly
    } // 'result' is instantly destroyed the moment the method ends
}
```

#### 2. Instance Variables
Instance variables represent the "state" or attributes of a specific object.

- **Scope:** They are tied to a specific object. Every time you create a new object using the `new` keyword, that object gets its own unique, independent copy of the instance variables.

- **Initialization:** You do not have to initialize them immediately. If you don't, Java automatically assigns them a default value (`0` for numbers, `false` for booleans, `null` for objects).

- **Memory:** Stored alongside the object in the **Heap** memory.

```java title:instance_variable.java
public class Player {
    // Instance variables: Every player gets their own name and score
    String name; 
    int score; 

    public static void main(String[] args) {
        Player p1 = new Player();
        p1.name = "Mario";
        p1.score = 100;

        Player p2 = new Player();
        p2.name = "Luigi";
        // p2.score is automatically 0 because it wasn't explicitly set
    }
}
```

#### 3. Static (Class) Variables
Static variables belong to the class itself, not to any individual object.

- **Scope:** There is only **one copy** of a static variable in existence, regardless of how many objects you create. All objects of that class share that single variable. If one object changes it, it changes for everyone.

- **Initialization:** Like instance variables, they get default values automatically.

- **Memory:** Stored in the **Method Area** (or Metaspace in modern JVMs) when the class is first loaded by the ClassLoader.

```java title:static_variable.java
public class Enemy {
    // Instance variable: Every enemy has its own health
    int health = 100; 
    
    // Static variable: Shared across ALL enemies
    static int totalEnemiesCreated = 0; 

    public Enemy() {
        totalEnemiesCreated++; // Increases the shared counter every time a new enemy is spawned
    }

    public static void main(String[] args) {
        Enemy e1 = new Enemy();
        Enemy e2 = new Enemy();
        Enemy e3 = new Enemy();

        // Access static variables using the Class name, not the object name
        System.out.println(Enemy.totalEnemiesCreated); // Prints: 3
    }
}
```

## Default Values of Variables
In Java, whether a variable gets a default value depends entirely on _where_ it is declared.

- **Instance and Static variables:** The JVM automatically assigns them a default value when the object or class is created, even if you don't assign one yourself.

- **Local variables:** The compiler **never** assigns default values to local variables. If you attempt to use a local variable before initializing it, your code will fail to compile.

Here are the exact default values assigned to Instance and Static variables by the JVM.

| **Data Type**                                     | **Default Value**                                              |
| ------------------------------------------------- | -------------------------------------------------------------- |
| `byte`                                            | `0`                                                            |
| `short`                                           | `0`                                                            |
| `int`                                             | `0`                                                            |
| `long`                                            | `0L`                                                           |
| `float`                                           | `0.0f`                                                         |
| `double`                                          | `0.0d`                                                         |
| `char`                                            | `'\u0000'` (The null character, which prints as a blank space) |
| `boolean`                                         | `false`                                                        |
| **All Reference Types** (String, Objects, Arrays) | `null`                                                         |
##### Seeing it in Action
Here is a quick demonstration showing how the JVM handles these defaults in memory compared to a local variable.

```java title:DefaultValuesDemo.java
public class DefaultValuesDemo {
    
    // Instance variables (Automatically initialized)
    int playerLevel;
    boolean isAlive;
    String playerName;
    double health;

    public void displayDefaults() {
        System.out.println("int: " + playerLevel);
                         // Prints: 0
        System.out.println("boolean: " + isAlive);
                         // Prints: false
        System.out.println("String: " + playerName);
                         // Prints: null
        System.out.println("double: " + health);
                         // Prints: 0.0

        // Local variable (No default value!)
        int currentScore; 
        
        // System.out.println(currentScore); 
        // ^ UNCOMMENTING THIS CAUSES A COMPILER ERROR: 
        // "variable currentScore might not have been initialized"
    }

    public static void main(String[] args) {
        DefaultValuesDemo demo = new DefaultValuesDemo();
        demo.displayDefaults();
    }
}
```

