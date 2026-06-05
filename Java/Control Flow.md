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

