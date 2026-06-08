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