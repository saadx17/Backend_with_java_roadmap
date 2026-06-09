Here, we are going to discuss:

1. [[#What is a Variable?|Variable]]
2. [[#What is Data Types?|Datatypes]]

# What is a Variable?
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
        System.out.println("int: " + playerLevel);     // Prints: 0
        System.out.println("boolean: " + isAlive);     // Prints: false
        System.out.println("String: " + playerName);   // Prints: null
        System.out.println("double: " + health);       // Prints: 0.0

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

# What is Data Types?
A classification that specifies the type of value a variable can hold, the amount of memory it requires, and the operations that can be performed on it. [Java data types](https://www.w3schools.com/java/java_data_types.asp) are divided into ==two main categories==: **Primitive** and **Non-Primitive (Reference).**

| Feature       | Primitive      | Non-Primitive              |
| ------------- | -------------- | -------------------------- |
| Stores        | Actual value   | Reference (memory address) |
| Memory        | Stack          | Heap                       |
| Default value | 0, false, etc. | `null`                     |
| Has methods?  | No             | Yes                        |
| Example       | `int`, `char`  | `String`, `Arrays`         |

## What are Primitive Types?
Primitive types are the **most basic data types** built into Java.  
They store **raw values directly in memory** (not objects).  
There are exactly **8 primitive types** in Java.

#### 1. Integer Types (whole numbers)

##### ==byte==
In Java, the **byte** data type is ==an 8-bit signed two's complement integer==. It is the smallest integer type in the language and is primarily used to save memory in large arrays or when working with raw binary data from files or network streams.

- Size: **8 bits (1 byte)**
- Range: **-128 to 127**
- Use when: saving memory with small numbers

```java title:syntax.java
byte temperature = 36;
byte minVal = -128;
byte maxVal = 127;
```

##### ==short==
In Java, **`short`** refers to both a primitive data type and a corresponding wrapper class used to store small integer values.

- Size: **16 bits (2 bytes)**
- Range: **-32,768 to 32,767**
- Use when: working with legacy systems or saving memory

```java title:syntax.java
short year = 2024;
short population = 32000;
```

###### `short` as wrapper class:
In Java, the [**`Short`**](https://www.javaguides.net/2018/08/short-wrapper-class-in-java.html) class is the wrapper class for the primitive data type `short [12, 13]`. It encapsulates a single 16-bit signed integer value within an object, allowing you to treat a `short` value as an object `[13, 27]`
##### ==**int**==
In Java, **`int`** is a primitive data type used to store signed 32-bit whole numbers. It is the most commonly used numeric type for integers.

- Size: **32 bits (4 bytes)**
- Range: **-2,147,483,648 to 2,147,483,647** (about ±2 billion)
- **Default choice for whole numbers in Java**
- Use when: counting, indexing, general purpose integers

```java title:syntax.java
int age = 25;
int score = 1000000;
int negative = -500;
```

##### ==**long**==
In Java, [`long`](https://www.javaguides.net/2018/08/long-wrapper-class-in-java.html) is a primitive data type and `Long`  is its corresponding wrapper class. They are used for storing 64-bit signed integers when the range of a standard 32-bit `int` is insufficient.

- Size: **64 bits (8 bytes)**
- Range: **-9,223,372,036,854,775,808 to 9,223,372,036,854,775,807**
- Must add **L** at the end of the value
- Use when: numbers exceed int range (population of earth, timestamps, IDs)

```java title:syntax.java
long worldPopulation = 8000000000L;  // L is required
long timestamp = 1712345678901L;
long bankAccountBalance = 9999999999L;
```

**Note:** **Why L?** Without it, Java treats the number as an int literal first, which overflows before being assigned to long.

#### 2. Decimal Types (floating point numbers)

##### ==**float**==
In Java, `float` is a primitive data type used to store single-precision 32-bit [**IEEE 754**](https://www.geeksforgeeks.org/computer-organization-architecture/ieee-standard-754-floating-point-numbers/) floating-point numbers. It is primarily used when memory conservation is a priority in large arrays of fractional numbers and when 6 to 7 decimal digits of precision are sufficient.

- Size: **32 bits (4 bytes)**
- Precision: **~6-7 decimal digits**
- Must add **f** at the end
- Use when: memory is tight and precision is not critical

```java title:syntax.java
float price = 9.99f;   // f is required as L
float temperature = 36.6f;
float pi = 3.14159f;
```

##### ==**double**==
In Java, `double` is a primitive data type used to store fractional (floating-point) numbers with high precision.

- Size: **64 bits (8 bytes)**
- Precision: **~15-16 decimal digits**
- **Default choice for decimal numbers in Java**
- Use when: scientific calculations, financial calculations (with care)

```java title:syntax.java
double salary = 75000.50;
double pi = 3.141592653589793;
double gpa = 3.85;
```


##### **float vs double:**
Always prefer `double`, as it is *more precise* and is Java's default decimal type. Only use `float` when memory is *critically limited*.

**Warning about floating point:** Never use `float` or `double` for precise financial calculations. Use `BigDecimal` instead.

```java title:syntax.java
// This is a problem with floating point precision

System.out.println(0.1 + 0.2);

// Output: 0.30000000000000004
```

###### What is `BigDecimal?`
`BigDecimal` is a class (primarily in Java's `java.math` package) designed to provide immutable, arbitrary-precision signed decimal numbers, essential for financial calculations. It enables exact accuracy by avoiding the rounding errors inherent in `float` and `double` types, offering precise control over scale and rounding behavior.

#### 3. ==`char`== (Single Character)
In Java, the **`char`** data type is a primitive used to store a single 16-bit Unicode character.

- Size: **16 bits (2 bytes)**
- Stores a **single character** using **single quotes**.
- Java uses **Unicode**, so it can store any character from any language.
- Range: 0 to 65,535 (unsigned)

```java title:syntax.java
char letter = 'A';
char digit = '5';
char symbol = '@';
char space = ' ';
char unicode = '\u0041'; // Unicode for 'A'
```

**Note:** *Single quotes for char, double quotes for String*
`'A'` → char (one character)  
`"A"` → String (object containing one character)

```java title:syntax.java
// char is actually a number underneath
char ch = 'A';
System.out.println(ch); // Output: A
System.out.println((int) ch); // Output: 65  (ASCII/Unicode value)
```

#### 4. ==`boolean`== (True or False)
In Java, a **boolean** is a data type that represents one of two logical values: `true` or `false`. It is primarily used for decision-making in conditional statements and loops.

- Size: **1 bit logically** (JVM may use more internally)
- Only two possible values: `true` or `false`
- **Never use 0/1 in Java like in C, only true/false**
- Use when: flags, conditions, switches

```java title:syntax.java
boolean isLoggedIn = true;
boolean hasPermission = false;
boolean isAdult = age >= 18; // result of a condition is boolean
boolean isEmpty = name.isEmpty(); // method returning boolean
```

#### Complete Primitive Types Reference Table:

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

## What are Non-Primitive Types?
Non-primitive data types, also known as **reference types**, differ from primitive types because they refer to objects rather than storing actual values directly. While primitive types are predefined by Java, non-primitive types are created by the programmer (except for `String`), they "point" or "refer" to a much larger, more complex object sitting in the JVM's [[JVM Architecture#3. Heap (Shared)|Heap]] memory.

##### 1. ==String==
[[String]] is a **non-primitive type**, it is a **class** in Java (`java.lang.String`).  
It represents a **sequence of characters**.  
Uses **double quotes**.

```java title:syntax.java
String name = "Ahmed";
String greeting = "Hello, World!";
String empty = "";
String nullString = null; // String can be null (primitives cannot)
```

###### String is Immutable
Once created, **a String cannot be changed**.  
Any operation that seems to modify a String actually **creates a new String**.

```java title:syntax.java
String name = "Ahmed";
name.toUpperCase(); // Does NOT change name
System.out.println(name); // Still prints: Ahmed
String upper = name.toUpperCase(); // Creates a NEW String
System.out.println(upper); // Prints: AHMED
```

###### String Pool
Java optimizes Strings using a **String Pool** in memory:

```java title:syntax.java
String a = "Hello"; // Goes into String Pool
String b = "Hello"; // Reuses same object from pool String
c = new String("Hello"); // Forces a NEW object in Heap (avoid this)

System.out.println(a == b); // true (same reference in pool)
System.out.println(a == c); // false (different objects)
System.out.println(a.equals(c)); // true (same content)
```

**NOTE:** *Always use `.equals()` to compare Strings, never `==`*

##### 2. ==Arrays==
An [[Arrays|array]] is a **fixed-size collection of elements of the same type**.
###### Declaring and creating arrays:
```java title:syntax.java
// Method 1: Declare then initialize
int[] numbers = new int[5]; // Array of 5 integers (all default to 0)
numbers[0] = 10;
numbers[1] = 20;
numbers[2] = 30;

// Method 2: Declare and initialize together
int[] scores = {95, 87, 76, 92, 88};

// Method 3: new keyword with values
String[] names = new String[]{"Alice", "Bob", "Charlie"};
```

###### Accessing array elements:
```java title:syntax.java
int[] scores = {95, 87, 76, 92, 88};

System.out.println(scores[0]); // 95 (first element, index starts at 0)
System.out.println(scores[4]); // 88 (last element)
System.out.println(scores.length); // 5 (number of elements)
```

###### Array index visualization:
```java title:syntax.java
int[] scores = {95, 87, 76, 92, 88};

Index: 0 1 2 3 4

       ┌────┬────┬────┬────┬────┐
Value: │ 95 │ 87 │ 76 │ 92 │ 88 │
       └────┴────┴────┴────┴────┘
       
scores[0] = 95
scores[4] = 88
scores[5] = ❌ ArrayIndexOutOfBoundsException
```

###### Iterating over an array:
```java title:syntax.java
int[] scores = {95, 87, 76, 92, 88};

// Regular for loop
for (int i = 0; i < scores.length; i++) { System.out.println("Index " + i + ": " + scores[i]); }

// Enhanced for loop (for-each) — cleaner, no index needed
for (int score : scores) {
System.out.println(score);
}
```

###### 2D Arrays:
```java title:syntax.java
int[][] matrix = {
{1, 2, 3},
{4, 5, 6},
{7, 8, 9}
};

System.out.println(matrix[0][0]); // 1 (row 0, col 0)
System.out.println(matrix[1][2]); // 6 (row 1, col 2)
System.out.println(matrix[2][1]); // 8 (row 2, col 1)
```

##### 3. Classes (User-Defined Objects)
A Class is a blueprint that you create. When you declare a variable using a Class type, you are creating a reference that will point to a specific "instance" (Object) of that blueprint.

###### Declaration & Initialization
Assuming you have already written a `Player` class somewhere else in your code:
```java title:syntax.java
// 1. Declare the reference variable
Player playerOne; 

// 2. Initialize it by calling the class constructor with 'new'
playerOne = new Player(); 

// All-in-one declaration and initialization
Player playerTwo = new Player("Luigi", 100);
```

##### 4. Interfaces
An Interface is a purely abstract "contract" that dictates what methods a class must have, but not how they work. You cannot instantiate an interface directly (you can't say `new MyInterface()`). Instead, you declare a reference variable of the Interface type, and point it to a Class that _implements_ that interface.

###### Declaration & Initialization
Assume `Playable` is an interface, and `AudioTrack` and `VideoClip` are classes that implement it.
```java title:syntax.java
// Declare a reference of the Interface type
Playable currentMedia;

// Point it to a specific object that implements the interface
currentMedia = new AudioTrack("song.mp3");

// Later, you can point that exact same reference to a different object type, 
// as long as it also implements the interface!
currentMedia = new VideoClip("movie.mp4");
```


