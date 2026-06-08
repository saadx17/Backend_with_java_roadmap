**Exception handling** in Java is a powerful mechanism that intercepts runtime errors to prevent application crashes and maintain a consistent program execution flow. When an error occurs within a method, the Java Virtual Machine (JVM) creates an "Exception object" containing the error type, description, and state of the program, which is then thrown into the system execution context.

Exception Handling lets you CATCH errors gracefully instead of crashing.
**Real world:**
- User enters text where number expected
- File doesn't exist
- Network connection drops
- Dividing by zero
- Accessing null object

## Exception Hierarchy

```
Throwable
├── Error                        ← JVM problems, don't catch these
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
│
└── Exception                    ← YOU handle these
    ├── RuntimeException         ← Unchecked (don't NEED try-catch)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ClassCastException
    │   ├── ArithmeticException
    │   ├── NumberFormatException
    │   └── IllegalArgumentException
    │
    └── IOException              ← Checked (MUST use try-catch)
        ├── FileNotFoundException
        ├── SQLException
        └── ParseException
```

### Checked vs Unchecked
The primary difference between checked and unchecked exceptions is that,
1. checked exceptions are verified by the compiler at compile-time.
2. unchecked exceptions are detected at runtime.

```java title:checked.java
// CHECKED → Exception subclasses (not Runtime)
// Compiler FORCES you to handle them
// External factors you can't always control

File f = new File("data.txt");
Scanner sc = new Scanner(f);   // ❌ won't compile!
// Must handle FileNotFoundException

// ✅ compiler happy now
try {
    Scanner sc = new Scanner(f);
} catch (FileNotFoundException e) {
    System.out.println("File not found");
}
```

```java title:unchecked.java
// UNCHECKED → RuntimeException subclasses
// Compiler doesn't force you to handle them
// Caused by programming mistakes

int[] arr = new int[5];
arr[10] = 1;              // ArrayIndexOutOfBoundsException
// No try-catch required by compiler, but will crash at runtime
```

## 1. `try` - `catch`

In Java, a **`try-catch` block** is the fundamental mechanism used to handle exceptions, preventing your application from crashing unexpectedly when runtime errors occur.

```java title:syntax.java
try {
    // code that might throw exception
} catch (ExceptionType e) {
    // handle the exception
}
```

```java title:without_try-catch.java
// (CRASHES)
int result = 10 / 0;                 // ArithmeticException
System.out.println(result);          // never reaches here
```

```java title:with_try-catch.java
// (HANDLED)
try {
    int result = 10 / 0;             // exception thrown here
    System.out.println(result);      // skipped
} catch (ArithmeticException e) {
    System.out.println("Cannot divide by zero!");  // runs
}
System.out.println("Program continues...");        // runs
```

### Exception Object (e)
In Java, **`e` is a variable name used to reference an Exception object** inside a `catch` block. When your program encounters a runtime error, the Java Virtual Machine (JVM) creates a data-filled object containing information about that specific failure. It then passes this instance directly to your `catch` block, binding it to the variable `e`.

```java title:syntax.java
try {
    int[] arr = new int[3];
    arr[10] = 50;                         // throws exception
} catch (ArrayIndexOutOfBoundsException e) {
    // e is the exception object
    System.out.println(e.getMessage());   // Index 10 out of bounds for length 3
    System.out.println(e.toString());     // java.lang.ArrayIndexOutOfBoundsException...
    e.printStackTrace();                  // full stack trace (best for debugging)
}

// Common Exception Messages
try {
    String s = null;
    s.length();
} catch (NullPointerException e) {
    System.out.println(e.getMessage());   // Cannot invoke "String.length()"...
}

try {
    int n = Integer.parseInt("abc");
} catch (NumberFormatException e) {
    System.out.println(e.getMessage());   // For input string: "abc"
}
```

### Multiple catch Blocks
In Java, **multiple catch blocks** allow a single `try` block to handle different types of exceptions separately. When a specific exception occurs inside the `try` block, Java passes control to the corresponding `catch` block, preventing the program from crashing.

```java title:syntax.java
// Catch different exceptions differently
try {
    Scanner sc = new Scanner(System.in);
    System.out.print("Enter index: ");
    int index = sc.nextInt();        // could be NumberFormatException

    int[] arr = {1, 2, 3, 4, 5};
    System.out.println(arr[index]);  // could be ArrayIndexOutOfBoundsException

    int result = 10 / arr[index];    // could be ArithmeticException
    System.out.println(result);

} catch (ArrayIndexOutOfBoundsException e) {
    System.out.println("Index out of range!");
} catch (ArithmeticException e) {
    System.out.println("Division by zero!");
} catch (Exception e) {
    System.out.println("Something went wrong: " + e.getMessage());
}
```
**RULE:** catch more specific FIRST, general LAST.

```java title:wrong-order.java
// WRONG ORDER → compiler error
try {
    int x = 10 / 0;
} catch (Exception e) {              // too broad, catches everything
    System.out.println("error");
} catch (ArithmeticException e) {   // unreachable! compiler error
    System.out.println("math error");
}
```

```java title:correct-order.java
// CORRECT ORDER → specific first
try {
    int x = 10 / 0;
} catch (ArithmeticException e) {   // specific first
    System.out.println("math error");
} catch (Exception e) {             // general last
    System.out.println("error");
}
```

### Multi-catch (Java 7+)
A **multi-catch block** in Java allows a single `catch` block to handle multiple types of exceptions using the pipe (`|`) operator. Introduced in Java 7, this feature drastically reduces code duplication when different exceptions require the exact same handling logic.

```java title:multi-catch.java
// Catch multiple exceptions with same handler
try {
    // some code
} catch (ArithmeticException | NullPointerException e) {
    System.out.println("Math or null error: " + e.getMessage());
} catch (ArrayIndexOutOfBoundsException | NumberFormatException e) {
    System.out.println("Array or format error: " + e.getMessage());
}
```

```java title:real_Example.java
public static int parseAndDivide(String numStr, int divisor) {
    try {
        int num = Integer.parseInt(numStr);
        return num / divisor;
    } catch (NumberFormatException | ArithmeticException e) {
        System.out.println("Error: " + e.getMessage());
        return -1;
    }
}
```

## 2. `try` with resources (Java 7+)

The **`try-with-resources` statement** in Java is a feature introduced in Java 7 that **automatically closes resources** (like files, database connections, or network streams) at the end of the statement block. This eliminates the need for manual cleanup in a `finally` block and safely prevents memory or resource leaks.

**Note:** Automatically closes resources (no need for finally!), works with anything that implements [AutoCloseable](https://docs.oracle.com/javase/8/docs/api/java/lang/AutoCloseable.html).

```java title:old-way.java
// Old way (verbose)
Scanner sc = null;
try {
    sc = new Scanner(new File("data.txt"));
    System.out.println(sc.nextLine());
} catch (FileNotFoundException e) {
    e.printStackTrace();
} finally {
    if (sc != null) sc.close();   // manual close
}
```

```java title:new-way.java
// New way (try-with-resources)
try (Scanner sc = new Scanner(new File("data.txt"))) {
    System.out.println(sc.nextLine());
} catch (FileNotFoundException e) {
    e.printStackTrace();
}
// sc automatically closed after try block
```

```java title:multi-resource.java
// Multiple resources
try (
    Scanner input  = new Scanner(new File("input.txt"));
    PrintWriter output = new PrintWriter("output.txt")
) {
    while (input.hasNextLine()) {
        output.println(input.nextLine());
    }
} catch (FileNotFoundException e) {
    e.printStackTrace();
}
// Both closed automatically
```

## 3. `finally` Block

The **`finally` block** in Java is a code block that **always executes after the `try` and `catch` blocks finish**, regardless of whether an exception is thrown, caught, or completely unhandled.

**Note:** `finally` block ALWAYS runs, regardless of exception or not and use for cleanup code (close files, DB connections, etc.)
```java title:finally.java
try {
    // risky code
} catch (Exception e) {
    // handle error
} finally {
    // ALWAYS runs
    // runs after try (no exception)
    // runs after catch (exception handled)
    // runs even if return is called!
}
```

### finally Always Runs
```java title:no-except.java
// No exception → finally still runs
try {
    System.out.println("try block");
    int x = 10 / 2;                      // no exception
    System.out.println("result: " + x);
} catch (ArithmeticException e) {
    System.out.println("catch block");    // skipped
} finally {
    System.out.println("finally block");  // always runs
}
// Output:
// try block
// result: 5
// finally block
```

```java title:with-execpt.java
// Exception → finally still runs
try {
    System.out.println("try block");
    int x = 10 / 0;                       // exception!
    System.out.println("never runs");
} catch (ArithmeticException e) {
    System.out.println("catch block");    // runs
} finally {
    System.out.println("finally block");  // always runs
}
// Output:
// try block
// catch block
// finally block
```

```java title:with-return.java
// Even with return → finally runs
public static int test() {
    try {
        return 1;                          // return called
    } finally {
        System.out.println("finally!");    // still runs before return!
    }
}
// Output: finally!
// Returns: 1
```

### finally for Cleanup
```java title:cleanup.java
// Classic pattern: always close resources

Scanner sc = null;
try {
    sc = new Scanner(new File("data.txt"));
    while (sc.hasNextLine()) {
        System.out.println(sc.nextLine());
    }
} catch (FileNotFoundException e) {
    System.out.println("File not found: " + e.getMessage());
} finally {
    if (sc != null) {
        sc.close();   // always close even if exception occurred
        System.out.println("Scanner closed");
    }
}
```

### When Finally Doesn'T Run
```java title:doesntrun.java
// Only case: JVM is forcefully shut down
try {
    System.exit(0);      // kills JVM immediately
} finally {
    System.out.println("Never runs!");  // ← skipped
}
```

## 4. `throw` Block

In Java, the `throw` keyword is used to explicitly clear and manually trigger a single exception from a method or any block of code. It breaks the normal application flow and transfers control directly to the nearest enclosing `try-catch` block.

`throw` → manually trigger an exception, You decide when something is wrong.
```java title:syntax.java
throw new ExceptionType("message");
```

```java title:example.java
public static double divide(int a, int b) {
    if (b == 0) {
        throw new ArithmeticException("Divisor cannot be zero");
    }
    return (double) a / b;
}
```

```java title:validate_input.java
public static void setAge(int age) {
    if (age < 0 || age > 150) {
        throw new IllegalArgumentException("Invalid age: " + age);
    }
    System.out.println("Age set to: " + age);
}
```

```java title:calling_it.java
try {
    setAge(-5);
} catch (IllegalArgumentException e) {
    System.out.println("Error: " + e.getMessage());  // Error: Invalid age: -5
}
```

```java title:null-check.java
public static void greet(String name) {
    if (name == null) {
        throw new NullPointerException("Name cannot be null");
    }
    System.out.println("Hello " + name);
}
```

## 5. `throws` (Declaring Exceptions)

The **`throws` keyword in Java** is used in a **method signature** to declare that the method might throw one or more specific exceptions during its execution. It delegates the responsibility of handling the exception to the **calling method** (the caller) rather than dealing with it internally via a `try-catch` block.

`throws` → tells caller "this method MIGHT throw this exception", You're passing responsibility to the caller.
```java title:without_throws.java
// Without throws → compiler error for checked exceptions
public static void readFile(String path) {
    Scanner sc = new Scanner(new File(path));  // ❌ compile error
}
```

```java title:with-throws.java
// With throws → compiler happy
public static void readFile(String path) throws FileNotFoundException {
    Scanner sc = new Scanner(new File(path));  // ✅
    while (sc.hasNextLine()) {
        System.out.println(sc.nextLine());
    }
    sc.close();
}
```

```java title:syntax.java
// Caller MUST handle it
public static void main(String[] args) {
    try {
        readFile("data.txt");
    } catch (FileNotFoundException e) {
        System.out.println("File not found!");
    }
}
```

```java title:multi-except.java
// Multiple exceptions
public static void process(String path)
        throws FileNotFoundException, IOException, SQLException {
    // might throw any of these
}
```

### throw vs throws
```
┌──────────┬────────────────────────────────────────────┐
│ throw    │ actually THROWS an exception right now     │
│          │ used inside method body                    │
│          │ throw new IllegalArgumentException(...)    │
├──────────┼────────────────────────────────────────────┤
│ throws   │ DECLARES method might throw exception      │
│          │ used in method signature                   │
│          │ public void read() throws IOException      │
└──────────┴────────────────────────────────────────────┘

throw  → action  (happens now)
throws → warning (might happen)
```

## 6. Custom Exceptions
To create a custom exception in Java, you must **create a new class that extends either `Exception` (for checked exceptions) or `RuntimeException` (for unchecked exceptions)**. Custom exceptions allow you to capture business-logic or domain-specific errors with clear, meaningful names.

```java title:custom-except.java
// Create your own exception types
// Makes code more readable and specific

// Custom unchecked exception
public class InsufficientFundsException extends RuntimeException {
    private double amount;

    public InsufficientFundsException(double amount) {
        super("Insufficient funds. Short by: $" + amount);
        this.amount = amount;
    }

    public double getAmount() {
        return amount;
    }
}

// Custom checked exception
public class InvalidAgeException extends Exception {
    public InvalidAgeException(String message) {
        super(message);
    }
}

// Using custom exceptions
public class BankAccount {
    private double balance;

    public BankAccount(double balance) {
        this.balance = balance;
    }

    public void withdraw(double amount) {
        if (amount > balance) {
            throw new InsufficientFundsException(amount - balance);
        }
        balance -= amount;
        System.out.printf("Withdrew $%.2f | Balance: $%.2f%n", amount, balance);
    }
}
```

```java title:main.java
// Main
try {
    BankAccount account = new BankAccount(100.0);
    account.withdraw(150.0);
} catch (InsufficientFundsException e) {
    System.out.println(e.getMessage());       // Insufficient funds. Short by: $50.0
    System.out.println("Short by: $" + e.getAmount());  // Short by: $50.0
}
```

## 7.Exception Chaining
**Exception Chaining** in Java allows you to associate one exception with another, where the first exception describes the **underlying root cause** of the second one. Introduced in Java 1.4, this design pattern helps developers wrap low-level system errors into higher-level business or abstraction-layer exceptions without losing vital stack trace debugging information.

**Note:** Wrap original exception inside new one, preserves the original cause for debugging.

```java title:except-chaining.java
public static void connectToDatabase() throws Exception {
    try {
        // simulating connection failure
        throw new RuntimeException("Connection refused at port 5432");
    } catch (RuntimeException e) {
        // wrap original exception
        throw new Exception("Database connection failed", e);  // e = cause
    }
}

try {
    connectToDatabase();
} catch (Exception e) {
    System.out.println(e.getMessage());         // Database connection failed
    System.out.println(e.getCause());           // original cause
    e.printStackTrace();                         // shows full chain
}
```

## Real-World Example
```java title:main.java
import java.util.Scanner;

// Custom exception
class InsufficientFundsException extends RuntimeException {
    public InsufficientFundsException(String msg) { super(msg); }
}



class BankAccount {
    private String owner;
    private double balance;

    public BankAccount(String owner, double initialBalance) {
        if (initialBalance < 0)
            throw new IllegalArgumentException("Initial balance can't be negative");
        this.owner = owner;
        this.balance = initialBalance;
    }

    public void deposit(double amount) {
        if (amount <= 0)
            throw new IllegalArgumentException("Deposit must be positive");
        balance += amount;
        System.out.printf("Deposited $%.2f | New balance: $%.2f%n", amount, balance);
    }

    public void withdraw(double amount) {
        if (amount <= 0)
            throw new IllegalArgumentException("Withdrawal must be positive");
        if (amount > balance)
            throw new InsufficientFundsException(
                String.format("Cannot withdraw $%.2f. Balance: $%.2f", amount, balance)
            );
        balance -= amount;
        System.out.printf("Withdrew $%.2f | New balance: $%.2f%n", amount, balance);
    }

    public double getBalance() { return balance; }
}



public class BankApp {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        BankAccount account = null;

        // Setup account
        try {
            account = new BankAccount("Alice", 500.0);
            System.out.println("Account created for Alice | Balance: $500.00");
        } catch (IllegalArgumentException e) {
            System.out.println("Failed to create account: " + e.getMessage());
            return;
        }

        // Simulate operations
        String[] operations = {"deposit 200", "withdraw 100",
                               "withdraw 700", "deposit -50"};

        for (String op : operations) {
            try {
                String[] parts = op.split(" ");
                String  action = parts[0];
                double  amount = Double.parseDouble(parts[1]);

                switch (action) {
                    case "deposit"  -> account.deposit(amount);
                    case "withdraw" -> account.withdraw(amount);
                    default         -> System.out.println("Unknown operation");
                }

            } catch (InsufficientFundsException e) {
                System.out.println("❌ " + e.getMessage());
            } catch (IllegalArgumentException e) {
                System.out.println("❌ Invalid input: " + e.getMessage());
            } catch (Exception e) {
                System.out.println("❌ Unexpected error: " + e.getMessage());
            }
        }

        System.out.printf("%nFinal Balance: $%.2f%n", account.getBalance());
        sc.close();
    }
}

// Output:
// Account created for Alice | Balance: $500.00
// Deposited $200.00 | New balance: $700.00
// Withdrew $100.00  | New balance: $600.00
// ❌ Cannot withdraw $700.00. Balance: $600.00
// ❌ Invalid input: Deposit must be positive
//
// Final Balance: $600.00
```