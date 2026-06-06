**Control flow** refers to the order in which statements within a program execute. While programs typically follow a **sequential flow** from top to bottom, there are scenarios where we need more flexibility.
```
By default Java runs top to bottom
Control flow lets you:
├── Make decisions   → if, switch
├── Repeat code      → loops
└── Jump around      → break, continue
```

# Types of Control Flow

| **Category**                | **Primary Purpose**                                                                    | **Common Keywords**                       | **Industry Context / Best Practice**                                                                                                                                  |
| --------------------------- | -------------------------------------------------------------------------------------- | ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sequential**              | The default mode. Executes code strictly from top to bottom, line by line.             | _None_                                    | The goal of good backend architecture is to keep the "happy path" as sequential and flat as possible.                                                                 |
| **Selection (Conditional)** | Branches the execution path based on boolean logic or pattern matching.                | `if`, `else if`, `else`, `switch`, `case` | Avoid deep nesting. Use **Guard Clauses** (early returns) for `if` statements. Prefer modern **Switch Expressions** for safety and exhaustiveness.                    |
| **Iteration (Looping)**     | Executes a block of code repeatedly as long as a specific condition remains true.      | `for`, `while`, `do-while`, `foreach`     | Beware of infinite loops blocking server threads. For collections, prefer `foreach` loops or the **Java 8 Streams API** over traditional `for` loops.                 |
| **Jump (Interruption)**     | Instantly transfers control to another part of the program, interrupting current flow. | `break`, `continue`, `return`, `yield`    | `return` is heavily used for early exits. `yield` is used in modern switch expressions. Avoid labels with `break`/`continue` as they create unreadable code.          |
| **Exception Handling**      | Diverts control flow specifically when an unexpected error or runtime anomaly occurs.  | `try`, `catch`, `finally`, `throw`        | Never use exceptions for normal business logic control flow (it is CPU heavy). Use them strictly for recovering from actual failures (like a database disconnecting). |

# Conditional Statements
**Conditional statements in Java** allow your program to evaluate boolean expressions and execute different paths of code based on whether those expressions are `true` or `false`. Java provides three primary control structures for handling conditional logic: **`if-else` blocks**, **`switch` blocks**, and the shorthand **ternary operator**.

## 1. `if-else`

The **`if-else` statement in Java** is a fundamental decision-making control structure used to execute different blocks of code based on whether a boolean condition evaluates to `true` or `false`.

#### `if` Statement
Runs a block of code **only if** the condition is `true`.
```java title:syntax.java
if (condition) {
   // runs if condition is true
}

// Example
int age = 20;
if (age >= 18) {
   System.out.println("You are an adult");
}
```

**How it works:**
```
Condition
    │
    ▼
  true? ── Yes ──→ Execute block
    │
    No
    │
    ▼
Skip block
```

#### `if-else` Statement
Runs one block if true, **another block if false**.
```java title:if-else_statement.java
int age = 15;
if (age >= 18) {
   System.out.println("Adult");
}
else {
   System.out.println("Minor"); // this runs
}
```

**How it works:**
```
Condition
    │
    ▼
  true? ──Yes──→ Execute if-block
    │
    No
    │
    ▼
Execute else-block
```

#### `if-else if-else` Chain
Check **multiple conditions** in sequence. The first true condition runs, rest are skipped.
```java title:chain.java
int score = 75;

if (score >= 90) {
   System.out.println("Grade: A");
}
else if (score >= 80) {
   System.out.println("Grade: B");
}
else if (score >= 70) {
   System.out.println("Grade: C"); // this runs
}
else if (score >= 60) {
   System.out.println("Grade: D");
}
else {
   System.out.println("Grade: F");
}
```

```java
// Conditions
Check score >= 90? → No
Check score >= 80? → No
Check score >= 70? → Yes → Print "Grade: C" → STOP (rest skipped)
```

#### Nested `if` Statements
`if` inside another `if`.
```java title:nested_if.java
int age = 25;
boolean hasLicense = true;
if (age >= 18) {
  if (hasLicense) {
     System.out.println("You can drive"); // this runs
}
else {
     System.out.println("Get a license first");
   }
}
else {
     System.out.println("Too young to drive");
}
```

**Note:** ***Avoid deep nesting**, more than 2-3 levels is a code smell. Refactor using early returns or combining conditions.*

```java title:deeplyNested.java
// Deeply nested (hard to read)
if (user != null) {
  if (user.isActive()) {
    if (user.hasPermission()) {
    // do something
    }
  }
}

// ✅ Early return (flat and clean)
if (user == null) return;
if (!user.isActive()) return;
if (!user.hasPermission()) return; // do something
```

#### Real-World `if-else` Example:
```java title:example.java
double accountBalance = 1500.0;
double withdrawAmount = 500.0;
boolean isAccountFrozen = false;

if (isAccountFrozen) {
  System.out.println("Account is frozen. Transaction denied.");
}
else if (withdrawAmount <= 0) {
  System.out.println("Invalid amount.");
}
else if (withdrawAmount > accountBalance) {
  System.out.println("Insufficient funds.");
}
else {
accountBalance -= withdrawAmount;
  System.out.println("Withdrawal successful. Balance: " + accountBalance);
}
```

## 2. `Switch`

`switch` is used when you have **one variable** and want to compare it against **multiple specific values**.

### Traditional `switch`
```java title:switch.java
switch (variable) {
  case value1:
      // code
      break;
  case value2:
      // code
      break;
  default:
      // code if no case matched
}
```

```java title:realUse.java
int day = 3;

switch (day) {
case 1: System.out.println("Monday");
break;
case 2: System.out.println("Tuesday");
break;
case 3: System.out.println("Wednesday"); // this runs
break;
case 4: System.out.println("Thursday");
break;
case 5: System.out.println("Friday");
break;
default: System.out.println("Weekend");
}
```

##### The `break` Keyword
Without `break`, Java **falls through** to the next case.
```java title:break.java
int day = 3;

switch (day) {
 case 3: System.out.println("Wednesday"); // runs //
         no break!
 case 4: System.out.println("Thursday"); // ALSO runs (fall-through!)
 case 5: System.out.println("Friday"); // ALSO runs (fall-through!)
         break;
 default: System.out.println("Other");
 }
 // Output:
 // Wednesday
 // Thursday
 // Friday
```
**Note:** *Fall-through can be used **intentionally**.*

#### `Switch` with `Strings`

```java title:switch_strings.java
String command = "start";

switch (command) {
  case "start": System.out.println("Starting the engine...");
                break;
  case "stop": System.out.println("Stopping the engine...");
                break;
  case "pause": System.out.println("Pausing...");
                break;
  default: System.out.println("Unknown command: " + command);
}
```

### `Switch` (Java 14+) 
The new switch is cleaner, **no break needed**, can return a value.
```java title:switch14.java
// Arrow syntax — no fall-through, no break needed

int day = 3;

String dayName = switch (day) {
  case 1 -> "Monday";
  case 2 -> "Tuesday";
  case 3 -> "Wednesday";
  case 4 -> "Thursday";
  case 5 -> "Friday";
  case 6 -> "Saturday";
  case 7 -> "Sunday";
  default -> "Invalid day";
};

System.out.println(dayName); // Wednesday
```

#### Multiple values per case
```java title:switch_case.java
int day = 6;

String type = switch (day) {
  case 1, 2, 3, 4, 5 -> "Weekday";
  case 6, 7 -> "Weekend";
  default -> "Invalid";
};

System.out.println(type); // Weekend
```

## `switch` vs `if-else`(When to use which)

|Situation|Use|
|---|---|
|Comparing one variable to specific values|`switch`|
|Complex conditions with ranges|`if-else`|
|Multiple conditions combined|`if-else`|
|Enum or known set of values|`switch`|
|Ranges like `score >= 90`|`if-else`|

```java title:switch_vs_if-else.java
// switch is perfect for this
switch (userRole) {
  case "ADMIN" -> showAdminPanel();
  case "USER" -> showUserDashboard();
  case "GUEST" -> showGuestPage();
  default -> showErrorPage();
}

// if-else is better for this
if (temperature > 35) {
  System.out.println("Very hot");
}
else if (temperature > 25) {
  System.out.println("Warm");
}
else if (temperature > 15) {
  System.out.println("Cool");
}
else { System.out.println("Cold"); }
```

## Pattern Matching for Switch (Java 17+)
**Pattern Matching for Switch** simplifies code by allowing you to check the **class type of an object** directly inside `case` labels, eliminating verbose `instanceof` checks and manual casting. Introduced as a preview in **Java 17 (JEP 406)**, it became a standard feature in **Java 21 (JEP 441**)

#### The Problem it Solves
Pattern Matching for Switch solves the problems of verbose `instanceof` chains, unsafe manual type-casting, and rigid restriction on data types in traditional Java `switch` statements. It transforms the `switch` construct from a simple primitive value-wrapper into a powerful tool for complex, data-driven control flow.

```java title:ugly_syntax.java
// Before Java 17 → ugly instanceof chain
Object obj = "Hello";

if (obj instanceof Integer) {
    Integer i = (Integer) obj;    // manual cast
    System.out.println(i * 2);
} else if (obj instanceof String) {
    String s = (String) obj;      // manual cast
    System.out.println(s.toUpperCase());
} else if (obj instanceof Double) {
    Double d = (Double) obj;      // manual cast
    System.out.println(d + 1.0);
}
```

```java title:clean_syntax.java
// Java 17+ → type pattern matching in switch
Object obj = "Hello";

String result = switch (obj) {
    case Integer i -> "Integer: " + (i * 2);
    case String  s -> "String: "  + s.toUpperCase();  // ← matches
    case Double  d -> "Double: "  + (d + 1.0);
    case null      -> "null value";
    default        -> "Unknown type";
};

System.out.println(result);  // String: HELLO
```

#### Guarded Patterns (when clause)
```java title:guarded_patterns.java
// Add conditions to pattern cases with 'when'
Object obj = 42;

String result = switch (obj) {
    case Integer i when i < 0  -> "Negative: " + i;
    case Integer i when i == 0 -> "Zero";
    case Integer i when i > 0  -> "Positive: " + i;   // ← matches
    case String  s when s.isEmpty() -> "Empty string";
    case String  s             -> "String: " + s;
    default                    -> "Other";
};

System.out.println(result);  // Positive: 42


// Sealed classes + pattern switch (powerful combo)
sealed interface Shape permits Circle, Rectangle, Triangle {}
record Circle(double radius) implements Shape {}
record Rectangle(double w, double h) implements Shape {}
record Triangle(double base, double height) implements Shape {}

Shape shape = new Circle(5.0);

double area = switch (shape) {
    case Circle    c             -> Math.PI * c.radius() * c.radius();
    case Rectangle r             -> r.w() * r.h();
    case Triangle  t             -> 0.5 * t.base() * t.height();
};
// No default needed → compiler knows all cases covered (sealed)
System.out.println("Area: " + area);  // Area: 78.53...
```

#### Evolution of Switch in Java
```
Java 1   → switch statement (int, char only)
Java 7   → switch with String support
Java 14  → switch expressions (arrow syntax, yield) [preview]
Java 14  → switch expressions STANDARD
Java 17  → pattern matching for switch [preview]
Java 21  → pattern matching for switch STANDARD ✅

// Progression:
Old:     switch (x) { case 1: ...; break; }
Modern:  switch (x) { case 1 -> ...; }
Newest:  switch (obj) { case String s when s.length() > 5 -> ...; }
```

```java title:complete_example.java
import java.util.Scanner;

public class ATM {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        double balance = 1000.0;
        int choice;

        System.out.println("=== ATM ===");

        do {
            System.out.println("\n1. Check Balance");
            System.out.println("2. Deposit");
            System.out.println("3. Withdraw");
            System.out.println("4. Exit");
            System.out.print("Choice: ");
            choice = sc.nextInt();

            switch (choice) {
                case 1 -> System.out.printf("Balance: $%.2f%n", balance);

                case 2 -> {
                    System.out.print("Deposit amount: $");
                    double amount = sc.nextDouble();
                    if (amount <= 0) {
                        System.out.println("Invalid amount");
                    } else {
                        balance += amount;
                        System.out.printf("Deposited $%.2f%n", amount);
                    }
                }

                case 3 -> {
                    System.out.print("Withdraw amount: $");
                    double amount = sc.nextDouble();
                    if (amount <= 0) {
                        System.out.println("Invalid amount");
                    } else if (amount > balance) {
                        System.out.println("Insufficient funds");
                    } else {
                        balance -= amount;
                        System.out.printf("Withdrew $%.2f%n", amount);
                    }
                }

                case 4 -> System.out.println("Thank you. Goodbye!");

                default -> System.out.println("Invalid choice");
            }

        } while (choice != 4);

        sc.close();
    }
}
```

## 3. Ternary Operator
The ternary operator is a **compact one-line if-else**.  It is the only operator in Java that takes **three operands**.

| Concept                | Key Point                                                |
| ---------------------- | -------------------------------------------------------- |
| `+ - * / %`            | Arithmetic — `/` truncates for integers                  |
| `%`                    | Remainder — useful for even/odd, wrapping, divisibility  |
| `++ --`                | Increment/decrement — pre vs post matters in expressions |
| `== != > < >= <=`      | Comparison — always returns boolean                      |
| `==` on objects        | Compares references — use `.equals()` for content        |
| `&&`                   | AND — both must be true — short-circuits on first false  |
| `\|`                   | OR — at least one true — short-circuits on first true    |
| `!`                    | NOT — flips boolean                                      |
| `+= -= *= /= %=`       | Compound assignment — shorthand for x = x op n           |
| `if-else if-else`      | First matching condition runs, rest skipped              |
| `switch`               | Compare one variable against specific values             |
| `break`                | Stops fall-through in switch                             |
| Switch expression `->` | Java 14+, no break needed, can return value              |
| Ternary `? :`          | One-line if-else, use for simple conditions only         |

**Syntax:**
```
result = (condition) ? valueIfTrue : valueIfFalse;
```

```java title:syntax.java
// Regular if-else
int age = 20;
String status;
if (age >= 18) {
  status = "Adult";
}
else { status = "Minor"; }

// Same thing with ternary
String status = (age >= 18) ? "Adult" : "Minor";
System.out.println(status); // Adult
```

#### Nested Ternary (Use Carefully)
```java title:nested_ternary.java
int score = 75;

// Nested ternary (hard to read, avoid when possible)
String grade = score >= 90 ? "A" :
score >= 80 ? "B" :
score >= 70 ? "C" :
score >= 60 ? "D" : "F";

System.out.println(grade); // C
```
**Note:** *Use ternary for simple, clear conditions. Use if-else for complex logic.*

#### When to Use Ternary vs `if-else`:
```java title:if_ter_else.java
// ✅ Good use of ternary — simple, readable
String label = isActive ? "Active" : "Inactive";
int fee = isMember ? 0 : 50;

// ❌ Bad use of ternary — too complex, use if-else
String result = (a > b) ? ((a > c) ? doSomethingComplex(a) : doOther(c)) : (b > c ? process(b) : fallback());
```

#### Example
```java title:ternary.java
public class LoanEligibility {
       public static void main(String[] args) {


// Applicant data
int age = 28;
double monthlyIncome = 6000.0;
int creditScore = 720;
boolean hasExistingLoan = false;
double requestedAmount = 50000.0;

// Determine interest rate based on credit score
double interestRate = switch (creditScore / 100) {
  case 8, 9, 10 -> 3.5;
  case 7 -> 5.0;
  case 6 -> 7.5;
  default -> -1.0; // ineligible
};

// Check eligibility
if (age < 18 || age > 65) {
   System.out.println("Ineligible: Age must be between 18 and 65.");
}
else if (monthlyIncome < 3000) {
   System.out.println("Ineligible: Insufficient income.");
}
else if (creditScore < 600) {
   System.out.println("Ineligible: Credit score too low.");
}
else if (hasExistingLoan) {
   System.out.println("Ineligible: Cannot have existing loan.");
}
else if (requestedAmount > monthlyIncome * 60) {
   System.out.println("Ineligible: Loan amount too high for your income.");
}
else {
  // Approved
  String riskLevel = (creditScore >= 750) ? "Low Risk" :
                     (creditScore >= 650) ? "Medium Risk" : "High Risk";
  System.out.println("✅ Loan Approved!");
  System.out.println("Amount: $" + requestedAmount);
  System.out.println("Interest Rate: " + interestRate + "%");
  System.out.println("Risk Level: " + riskLevel);
  
  double monthlyPayment = (requestedAmount * (interestRate / 100)) / 12;
  System.out.println("Est. Monthly Interest: $" + monthlyPayment);
   }
  }
 }
```

# Iteration
[Iteration](https://www.geeksforgeeks.org/java/iterators-in-java/) in Java refers to the repetitive execution of a block of code or the sequential traversal of data structures. Java provides multiple tools to accomplish this, ranging from standard **control flow loops to dedicated **[[Java Collections Framework|collection framework]] objects**.

| Loop       | Use When                                             |
| ---------- | ---------------------------------------------------- |
| `for`      | You know exactly how many times to repeat            |
| `while`    | You repeat while a condition is true (count unknown) |
| `do-while` | You need to run at least once, then check condition  |
| `for-each` | You iterate over a collection or array               

## 1. `for` Loop
The most commonly used loop when the **number of iterations is known**.
```java title:syntax.java
for(initialization; condition; update){
  //body - runs each iteration
}
```

```java title:example.java
// Without a loop — repetitive and unscalable
System.out.println("Hello");
System.out.println("Hello");
System.out.println("Hello");
System.out.println("Hello");
System.out.println("Hello");

// With a loop — clean and scalable
for (int i = 0; i < 5; i++) {
  System.out.println("Hello");
}
```
#### Understanding by parts
| Part           | Runs                  | Purpose                  |
| -------------- | --------------------- | ------------------------ |
| Initialization | Once at start         | Create/set loop variable |
| Condition      | Before each iteration | If false → loop stops    |
| Update         | After each iteration  | Change loop variable     |

```java title:part_by_syntax.java
for (int i = 0; i < 5; i++) {
// └────┬────┘ └──┬──┘ └┬─┘
// initialization │ update
//            condition
```

Basic Examples:
```java title:examples.java
// Count from 0 to 4
for (int i = 0; i < 5; i++) {
  System.out.println(i);
}
// Output: 0 1 2 3 4

// Count from 1 to 10
for (int i = 1; i <= 10; i++) {
  System.out.println(i);
}
// Output: 1 2 3 4 5 6 7 8 9 10

// Count downwards
for (int i = 10; i >= 1; i--) {
  System.out.println(i);
}
// Output: 10 9 8 7 6 5 4 3 2 1

// Step by 2
for (int i = 0; i <= 20; i += 2) {
  System.out.print(i + " ");
}
// Output: 0 2 4 6 8 10 12 14 16 18 20
```

#### Visualizing Loop Execution
```java title:visual.java
for (int i = 0; i < 4; i++) {
 System.out.println("i = " + i);
}
```

```
Step 1: i = 0 → condition: 0 < 4 → true → run body → print "i = 0" → i++
Step 2: i = 1 → condition: 1 < 4 → true → run body → print "i = 1" → i++
Step 3: i = 2 → condition: 2 < 4 → true → run body → print "i = 2" → i++
Step 4: i = 3 → condition: 3 < 4 → true → run body → print "i = 3" → i++
Step 5: i = 4 → condition: 4 < 4 → false → STOP

Output:
i = 0
i = 1
i = 2
i = 3
```

#### Looping Over Arrays
```java title:array_loop.java
int[] scores = {95, 87, 76, 92, 88};

// Using index to access each element
for (int i = 0; i < scores.length; i++) {
  System.out.println("scores[" + i + "] = " + scores[i]);
}
// Output:
// scores[0] = 95
// scores[1] = 87
// scores[2] = 76
// scores[3] = 92
// scores[4] = 88
```

#### Common `for` Loop Patterns
```java title:loop_patterns.java
// Sum of numbers 1 to 100
int sum = 0;
for (int i = 1; i <= 100; i++) {
  sum += i;
}
System.out.println("Sum: " + sum); // Sum: 5050


// Factorial of n
int n = 5;
int factorial = 1;
for (int i = 1; i <= n; i++) {
  factorial *= i;
}
System.out.println(n + "! = " + factorial); // 5! = 120


// Multiplication table
int num = 7;
for (int i = 1; i <= 10; i++) {
  System.out.println(num + " x " + i + " = " + (num * i));
}
// 7 x 1 = 7
// 7 x 2 = 14
// ...


// Print only even numbers in an array
int[] numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
for (int i = 0; i < numbers.length; i++) {
  if (numbers[i] % 2 == 0) {
    System.out.print(numbers[i] + " ");
    }
}
// Output: 2 4 6 8 10
```

#### Variations of `for` Loop
```java title:variation.java
// Multiple variables in initialization and update
for (int i = 0, j = 10; i < j; i++, j--) {
  System.out.println("i=" + i + ", j=" + j);
}
// i=0, j=10
// i=1, j=9
// i=2, j=8
// i=3, j=7
// i=4, j=6


// Infinite for loop (use with break)
for (;;) {
// runs forever until break
  System.out.println("Running...");
  break;
}

// Empty body (rare but valid)
int i = 0;
for (; i < 5; i++); // i becomes 5
System.out.println(i); // 5
```

#### Nested `for` Loops
A loop inside of another loop. The "inner loop" will execute all of its iterations for _each single iteration_ of the "outer loop".

- **Use Cases:** Highly common when working with 2D grids, matrices, or generating tables.
- **Performance Warning:** Be careful! If your outer loop runs 100 times, and your inner loop runs 100 times, your code just executed 10,000 times (this is known as $O(n^2)$ time complexity).

```java title:syntax.java
for (int i = 1; i <= 3; i++) {                 // outer loop
    for (int j = 1; j <= 3; j++) {             // inner loop
        System.out.println("i=" + i + ", j=" + j);
    }
}
```

```java title:example.java
// A simple 3x3 grid generator
for (int row = 1; row <= 3; row++) {          // Outer loop
    for (int col = 1; col <= 3; col++) {      // Inner loop
        System.out.print("(" + row + "," + col + ") ");
    }
    System.out.println(); // Moves to the next line after the inner loop finishes
}
/* Output:
(1,1) (1,2) (1,3) 
(2,1) (2,2) (2,3) 
(3,1) (3,2) (3,3) 
*/
```


## 2.`while` Loop
Use `while` when you **don't know how many iterations** you need in advance.  
The condition is checked **before** each iteration.
```java title:syntax.java
while (condition) {
  // body
  // must update something to eventually make condition false!
}
```

#### How it works
```Check condition
         │
         ▼
       true? ──Yes──→ Execute body → go back to check condition
         │
         No
         │
         ▼
     Exit loop
```

```java title:example.java
// Count from 1 to 5
int i = 1;
while (i <= 5) {
  System.out.println(i);
  i++; // ← CRITICAL: without this, infinite loop!
}
// Output: 1 2 3 4 5

// Same as for loop but written as while
int count = 0;
while (count < 5) {
  System.out.println("Count: " + count);
  count++;
}
```

## 3.`do-while` Loop
This is almost identical to the `while` loop, with one massive difference: **it guarantees the code will run at least once**, even if the condition is `false` from the very beginning. It checks the condition at the _end_ of the loop.
```java title:syntax.java
do {
  // body — runs at least once
}
while (condition); // ← semicolon required here!
```

#### How it works
```
Execute body (always runs at least once)
       │
       ▼
Check condition
       │
     true? ──Yes──→ Execute body again → check condition again
       │
       No
       │
       ▼
    Exit loop
```

```java title:example.java
int i = 1;
do {
  System.out.println(i); i++;
}
while (i <= 5); // Output: 1 2 3 4 5
```

## 4.`for-each` Loop
Introduced in Java 5, this is the **industry standard** for iterating through Arrays and Collections (like Lists). It is cleaner, easier to read, and prevents "Index Out Of Bounds" errors.
```java title:syntax.java
for (type element : collection) {
  // use element
}

// Read as: "for each element in collection"
```

```java title:example.java
// Array of integers
int[] scores = {95, 87, 76, 92, 88};

for (int score : scores) {
  System.out.println(score);
}
// Output: 95 87 76 92 88

// Array of Strings
String[] names = {"Alice", "Bob", "Charlie", "Diana"};

for (String name : names) {
  System.out.println("Hello, " + name + "!");
}
// Hello, Alice!
// Hello, Bob!
// Hello, Charlie!
// Hello, Diana!


// Works with char array
char[] vowels = {'a', 'e', 'i', 'o', 'u'};
for (char vowel : vowels) {
  System.out.print(vowel + " ");
}
// Output: a e i o u
```

# Loop Control
Sometimes you need to interrupt a loop before it finishes naturally.

- **`break`:** Completely **exits** the loop immediately. The program moves on to the code below the loop.
- **`continue`:** Skips the rest of the **current iteration** and jumps straight back to the top of the loop to evaluate the condition for the next round.
- `Yield`: returns a value from a specific branch or case.
- `return`: terminates the execution of the entire method.

## 1. `break`: Exit the Loop Immediately

`break` **completely stops** the loop and jumps to the code after it.
```java title:break.java
// Find first number divisible by 7
for (int i = 1; i <= 100; i++) {
  if (i % 7 == 0) {
    System.out.println("First divisible by 7: " + i);
    break; // stop immediately, don't continue to 100
  }
}
// Output: First divisible by 7: 7


// Search in array
int[] numbers = {3, 7, 2, 9, 4, 1, 8};
int target = 9;
boolean found = false;

for (int i = 0; i < numbers.length; i++) {
  if (numbers[i] == target) {
    System.out.println("Found " + target + " at index " + i);
    found = true;
    break; // no need to check rest
  }
}

if (!found) {
  System.out.println(target + " not found");
}
```

**How it works:**
```
Loop: i=0 → 3 == 9? No
      i=1 → 7 == 9? No
      i=2 → 2 == 9? No
      i=3 → 9 == 9? Yes → print → BREAK → exit loop
      (i=4, 5, 6 never checked)
```

## 2. `continue`: Skip Current Iteration, Continue Loop

`continue` **skips the rest of the current iteration** and jumps to the next one.
```java title:continue.java
// Print only odd numbers
for (int i = 1; i <= 10; i++) {
  if (i % 2 == 0) {
    continue; // skip even numbers
    }
    System.out.print(i + " "); // only reached for odd numbers
}
// Output: 1 3 5 7 9

// Skip negative numbers
int[] values = {5, -3, 8, -1, 12, -7, 4};
int sum = 0;

for (int val : values) {
  if (val < 0) {
    continue; // skip negatives
    }
    sum += val;
  }
System.out.println("Sum of positives: " + sum); // 29
```

**How it works:**
```
Iteration: val=5 → 5 < 0? No → sum += 5 → sum=5
           val=-3 → -3 < 0? Yes → CONTINUE → jump to next iteration
           val=8 → 8 < 0? No → sum += 8 → sum=13
           val=-1 → -1 < 0? Yes → CONTINUE
           val=12 → 12 < 0? No → sum += 12 → sum=25
           val=-7 → -7 < 0? Yes → CONTINUE
           val=4 → 4 < 0? No → sum += 4 → sum=29
```

#### `break` vs `continue`
```java title:break_vs_continue.java
// break
for (int i = 0; i < 10; i++) {
  if (i == 5) break;
  System.out.print(i + " ");
}
// Output: 0 1 2 3 4
// → stops completely at 5

// continue
for (int i = 0; i < 10; i++) {
  if (i == 5) continue;
  System.out.print(i + " ");
}
// Output: 0 1 2 3 4 6 7 8 9
// → skips 5, keeps going
```

## 3. `yield`: Multi-line code block
Before enhanced switch expressions were introduced, older `switch` statements were primarily used for control flow (directing which lines of code run), requiring you to manually assign values to variables and use `break` statements to stop the flow.

#### Modern `yield`
In Java, the **`yield` keyword** is used inside **`switch` expressions** to return a value from a multi-line code block. Finalized in **Java 14**, it acts like a localized `return` statement tailored exclusively for your `switch` logic.
```java title:switch_with_yield.java
int score = 85;

String grade = switch (score / 10) {
  case 10, 9 -> "A";
  case 8 -> "B";
  case 7 -> {
        System.out.println("Just passed with C");
        yield "C"; // yield returns the value in a block
  }
  case 6 -> "D";
  default -> "F";
  };

System.out.println(grade); // B
```

#### Traditional `:` Colon `yield`
You can use `yield` with the older colon-style `switch` syntax if you want it to behave like an expression. In this scenario, `yield` replaces the traditional `break` statement.
```java title:colon.java
int dayNumber = 2;
String type = switch (dayNumber) {
    case 1:
    case 7: 
        yield "Weekend"; // Terminates block and returns value
    default: 
        yield "Weekday";
};
```

**Note:** Do not confuse the `yield` keyword with the old `java.lang.Thread.yield()` method.

- **`Thread.yield()`** is a static native method used in multi-threading.
- It acts as a temporary "hint" to the CPU scheduler, stating that the current thread is willing to pause its execution to let other equal-priority threads run.
- The scheduler is completely free to ignore this request.

## 4. `Return`: Breaks out of all condition & loops
In Java, the `return` keyword acts as a **jump statement** that immediately **terminates the execution of the entire method** in which the loop resides.

#### Key Behavior
- **Exits Everything**: It breaks out of all nested loops and conditionals instantly.
- **Returns Value (Non-Void Methods)**: Passes a specific data type back to the caller.
- **Exits Early (Void Methods)**: Can be used completely on its own without a value to break out early.
- **Compiler Requirements**: If used conditionally inside a loop, the compiler still expects a default `return` statement _outside_ the loop block in case the loop never runs or the condition is never met.

#### In a Value-Returning Method
This is highly efficient for lookup algorithms. As soon as the match is found, the method stops executing entirely.
```java title:value_return.java
public class SearchExample {
    public static int findIndex(int[] numbers, int target) {
        for (int i = 0; i < numbers.length; i++) {
            if (numbers[i] == target) {
                return i; // Instantly stops the loop and exits the method
            }
        }
        return -1; // Fallback return required by the compiler
    }
}
```

#### In a Void Method
You can use `return` on its own to cancel operations early without returning a value.
```java title:void_example.java
public class VoidExample {
    public static void printUntilNegative(int[] numbers) {
        for (int num : numbers) {
            if (num < 0) {
                return; // Exits the method immediately, ignoring remaining items
            }
            System.out.println(num);
        }
        System.out.println("Loop finished naturally."); // Won't execute if return triggered
    }
}
```

#### `yield` vs `return`
You cannot use the standard `return` keyword inside a switch case. Doing so would instruct the entire parent method to finish executing rather than just returning the result of the switch operation. `yield` lets the switch statement provide its final value while allowing the rest of your code to keep running.

## `return` vs `break` vs `continue`

| Keyword        | What it stops              | Where control goes next                |
| -------------- | -------------------------- | -------------------------------------- |
| **`return`**   | The entire method          | Back to the caller of the method       |
| **`break`**    | The current loop block     | The very next line _after_ the loop    |
| **`continue`** | The current loop iteration | The loop's next iteration check/update |

# Exception Handling
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