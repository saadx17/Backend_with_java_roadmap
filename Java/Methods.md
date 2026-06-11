A **method in Java** is a reusable block of code that executes only when called. Methods are declared within a class to perform specific actions, allow code reuse without retyping, and break down complex programs into manageable parts.

Method = reusable block of code that does ONE thing, function in other languages.

##### Why use methods?
- Avoid repeating code (DRY principle)
- Break complex problems into smaller pieces
- Easier to test, debug, maintain
- Makes code readable

# Method Declaration
A **method declaration** in Java defines a block of code that performs a specific task, outlining its attributes like visibility, return type, name, and input parameters. At minimum, a valid Java method declaration requires a **return type, a unique name, parentheses `()`, and a body enclosed in curly braces `{}`**.

### Anatomy of a Method
In Java, a method declaration defines the attributes and execution logic of a function within a class. According to the [Oracle Java Tutorials](https://docs.oracle.com/javase/tutorial/java/javaOO/methods.html), a full method declaration consists of **six distinct components** that must appear in a specific order.
```java title:anatomy.java
//  access    return
//  modifier  type    name        parameters
//  ↓         ↓       ↓           ↓
    public    int     add(        int a, int b) {
        return a + b;   // ← return statement
    }

// Full syntax
accessModifier returnType methodName(param1, param2, ...) {
    // method body
    return value;  // if returnType is not void
}
```

### Return Types
In Java, every method declaration **must explicitly define a return type**, which specifies the exact data type of the value the method sends back to its caller. The return type is positioned **directly before the method name** in a method signature.
```java title:return_types.java
// ─── void → returns nothing ───────────────────────────────
public static void greet(String name) {
    System.out.println("Hello, " + name);
    // no return needed (or use bare return; to exit early)
}

// ─── int → returns integer ────────────────────────────────
public static int add(int a, int b) {
    return a + b;  // MUST return int
}

// ─── String → returns String ──────────────────────────────
public static String fullName(String first, String last) {
    return first + " " + last;
}

// ─── boolean → returns true/false ────────────────────────
public static boolean isEven(int n) {
    return n % 2 == 0;
}

// ─── Array → returns array ────────────────────────────────
public static int[] createRange(int start, int end) {
    int[] arr = new int[end - start];
    for (int i = 0; i < arr.length; i++) {
        arr[i] = start + i;
    }
    return arr;
}

// ─── Early return (exit method before end) ────────────────
public static String classify(int n) {
    if (n < 0)  return "negative";  // exits here
    if (n == 0) return "zero";      // exits here
    return "positive";              // exits here
}

// Using them
greet("Alice");                                   // Hello, Alice
System.out.println(add(5, 3));                    // 8
System.out.println(fullName("John", "Doe"));      // John Doe
System.out.println(isEven(4));                    // true
System.out.println(Arrays.toString(createRange(1, 6))); // [1,2,3,4,5]
```

### Parameters vs Arguments
In Java, **parameters** are the variables defined in a method's declaration, while **arguments** are the actual values passed to those variables when the method is invoked.

```java title:para_arg.java
//          parameters (in definition)
//          ↓         ↓
public static int multiply(int a, int b) {
    return a * b;
}

//                 arguments (actual values passed)
//                 ↓    ↓
int result = multiply(5,  3);  // 15

// ─── No parameters ────────────────────────────────────────
public static void sayHello() {
    System.out.println("Hello!");
}
sayHello();  // called with no arguments

// ─── Multiple parameters ──────────────────────────────────
public static double calculateBMI(double weight, double height) {
    return weight / (height * height);
}
double bmi = calculateBMI(70.0, 1.75);  // order matters!
```

# Method Overloading
**Method overloading** in Java allows a class to have multiple methods with the **same name**, provided they have **different parameter lists**. It is a foundational component of object-oriented programming used to implement **compile-time polymorphism** (also known as static polymorphism or early binding).

### Why use Overloading?
**Method overloading** in Java is used to execute the same conceptual operation on different types or amounts of input data using a single, unified method name. It implements **compile-time polymorphism**, allowing the compiler to automatically select and map the correct method implementation based on the exact arguments passed at the call site.

### Key Benefits of Method Overloading
- **Eliminates Clunky Naming:** Without overloading, you would have to invent unique names for the same action, such as `printInt()`, `printString()`, and `printDouble()`. Overloading collapses these into a clean, single `print()` interface.
- **Improves Code Readability:** Code becomes intuitive and self-documenting because the focus remains on _what_ the method does, rather than the data type it consumes.
- **Provides API Flexibility:** It allows users of your class or library to supply input data in multiple formats (like an individual ID or a full username string) without needing a different workflow.
- **Facilitates Backward Compatibility:** Developers can safely introduce a new parameter configuration to an existing workflow by overloading the original method or constructor, reducing the risk of breaking legacy, external applications.

### Overloading Examples
```java title:method_overloading.java
public class Calculator {

    // ─── Different number of params ───────────────────────
    public static int add(int a, int b) {
        return a + b;
    }

    public static int add(int a, int b, int c) {
        return a + b + c;
    }

    public static int add(int a, int b, int c, int d) {
        return a + b + c + d;
    }

    // ─── Different types ──────────────────────────────────
    public static double add(double a, double b) {
        return a + b;
    }

    public static String add(String a, String b) {
        return a + b;   // concatenation
    }

    // ─── Different order of types ─────────────────────────
    public static void print(String s, int n) {
        System.out.println(s + " " + n);
    }

    public static void print(int n, String s) {
        System.out.println(n + " " + s);
    }

    public static void main(String[] args) {
        System.out.println(add(1, 2));           // 3       → add(int, int)
        System.out.println(add(1, 2, 3));         // 6       → add(int,int,int)
        System.out.println(add(1.5, 2.5));        // 4.0     → add(double,double)
        System.out.println(add("Hello", " World")); // Hello World

        print("Age:", 25);   // Age: 25   → print(String, int)
        print(25, "years");  // 25 years  → print(int, String)
    }
}
```

### Overloading Pitfalls
```java title:pitfalls.java
// ─── ❌ Return type alone is NOT enough ───────────────────
public static int getValue()    { return 1; }
public static double getValue() { return 1.0; }
// ❌ Compile error: Java can't tell which to call
// getValue(); → ambiguous!

// ─── Type promotion in overloading ───────────────────────
public static void show(int n) {
    System.out.println("int: " + n);
}
public static void show(double n) {
    System.out.println("double: " + n);
}

show(5);    // int: 5      → exact match int
show(5.0);  // double: 5.0 → exact match double
show('A');  // int: 65     → char promoted to int ⚠️

// ─── Ambiguous overloading ────────────────────────────────
public static void test(int a, double b) { }
public static void test(double a, int b) { }

test(1, 2);    // ❌ Ambiguous! compiler can't decide
test(1, 2.0);  // ✅ test(int, double)
test(1.0, 2);  // ✅ test(double, int)
```

# `static` & Instance Methods
In Java, the core difference is that **static methods belong to the class itself**, while **instance methods belong to an object (instance) of the class**.

| Feature            | Static Method                                       | Instance Method                               |
| ------------------ | --------------------------------------------------- | --------------------------------------------- |
| **Keyword**        | Requires the `static` keyword.                      | No specific keyword needed.                   |
| **How to Call**    | Directly via the class name (`Class.method()`).     | Via an object reference (`object.method()`).  |
| **Data Access**    | Only static variables and other static methods.     | Both static and instance variables/methods.   |
| **`this` Keyword** | Cannot use `this` or `super`.                       | Can use `this` and `super`.                   |
| **Polymorphism**   | Cannot be overridden (supports method hiding only). | Supports overriding and runtime polymorphism. |
| **Binding**        | Compile-time (static binding).                      | Runtime (dynamic binding).                    |

## `Static` Methods
A static method is stateless. It performs operations using only its input arguments and does not depend on the unique state of any specific object.

- **Use Case:** Utility functions, mathematical calculations, or factory methods.
- **Examples:** `Math.sqrt(25)`, `Integer.parseInt("10")`, or the standard `public static void main` entry point.

```java title:static_method.java
class Calculator {
    // Static method
    public static int add(int a, int b) {
        return a + b;
    }
}

// Calling without an object
int sum = Calculator.add(5, 10);
```

## Instance Methods
An instance method requires an object because its behavior depends on the data stored inside that specific object.

- **Use Case:** Modifying object attributes, getting/setting object state, or implementing behavior unique to a specific entity.
- **Examples:** `myList.add("item")` or `myString.toLowerCase()`

```java title:instance_method.java
class Car {
    private String model;

    public Car(String model) {
        this.model = model;
    }

    // Instance method
    public void displayModel() {
        System.out.println("Car model: " + this.model); // Accesses object-specific data
    }
}

// Calling requires object instantiation
Car myCar = new Car("Tesla");
myCar.displayModel(); 
```

## The Difference
```java title:difference.java
public class BankAccount {

    // ─── Instance variable ────────────────────────────────
    private String owner;
    private double balance;

    // ─── Constructor ──────────────────────────────────────
    public BankAccount(String owner, double balance) {
        this.owner   = owner;
        this.balance = balance;
    }

    // ─── Instance method → needs object to call ───────────
    // Works on THIS specific object's data
    public void deposit(double amount) {
        this.balance += amount;   // accesses instance variable
        System.out.printf("%s deposited $%.2f%n", owner, amount);
    }

    public double getBalance() {
        return this.balance;
    }

    public void printInfo() {
        System.out.printf("Owner: %s | Balance: $%.2f%n",
                           owner, balance);
    }

    // ─── Static method → belongs to CLASS not object ──────
    // Cannot access instance variables or instance methods!
    public static double calculateInterest(double principal,
                                           double rate,
                                           int years) {
        return principal * rate * years;  // no instance data needed
    }

    public static BankAccount createSavings(String owner) {
        return new BankAccount(owner, 100.0);  // factory method
    }

    public static void main(String[] args) {
        // ─── Static method → called on CLASS ──────────────
        double interest = BankAccount.calculateInterest(1000, 0.05, 3);
        System.out.println("Interest: $" + interest);  // Interest: $150.0

        // ─── Instance method → called on OBJECT ───────────
        BankAccount alice = new BankAccount("Alice", 500.0);
        alice.deposit(200.0);           // Alice deposited $200.00
        alice.printInfo();              // Owner: Alice | Balance: $700.00

        BankAccount bob = new BankAccount("Bob", 300.0);
        bob.deposit(100.0);             // Bob deposited $100.00

        // Static factory method
        BankAccount savings = BankAccount.createSavings("Charlie");
        savings.printInfo();  // Owner: Charlie | Balance: $100.00
    }
}
```

## Rules & Access
```java title:rules_access.java
public class MyClass {
    int instanceVar    = 10;        // belongs to object
    static int staticVar = 20;      // belongs to class (shared)

    // ─── Instance method → can access BOTH ────────────────
    void instanceMethod() {
        System.out.println(instanceVar);   // ✅
        System.out.println(staticVar);     // ✅
        staticMethod();                    // ✅
    }

    // ─── Static method → can ONLY access static stuff ─────
    static void staticMethod() {
        System.out.println(staticVar);     // ✅
        System.out.println(instanceVar);   // ❌ compile error!
        instanceMethod();                  // ❌ compile error!
        // no 'this' available in static context
    }
}

// ─── Calling them ─────────────────────────────────────────
MyClass obj = new MyClass();

obj.instanceMethod();    // ✅ via object
obj.staticMethod();      // ⚠️ works but bad practice
MyClass.staticMethod();  // ✅ correct way for static

// ─── Static variable → shared across ALL objects ──────────
public class Counter {
    static int count = 0;  // shared
    int id;                // unique per object

    Counter() {
        count++;           // increments shared counter
        id = count;        // unique id for this object
    }

    public static void main(String[] args) {
        Counter c1 = new Counter();
        Counter c2 = new Counter();
        Counter c3 = new Counter();

        System.out.println(Counter.count);  // 3 (shared)
        System.out.println(c1.id);          // 1
        System.out.println(c2.id);          // 2
        System.out.println(c3.id);          // 3
    }
}
```

## Common Gotchas

1. **The Static-to-Instance Error:** You cannot directly call an instance method or use an instance variable inside a static method. If you try, the compiler throws an error: `Non-static field/method cannot be referenced from a static context`. To fix this, you must explicitly create an object inside the static method first.
2. **Testing Constraints:** Code that relies heavily on static methods can be harder to unit test because static methods cannot be easily swapped out using traditional mocking frameworks.

## When to use what?

```
USE static when:
Method doesn't use instance variables
Utility/helper methods (Math.sqrt, Arrays.sort)
Factory methods (create objects)
Constants (static final PI = 3.14)
main() method (always static)
```

```
USE instance when:
Method uses or modifies object state
Behavior differs per object
Most methods in OOP classes
```

