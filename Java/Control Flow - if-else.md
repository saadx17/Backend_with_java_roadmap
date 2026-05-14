Control flow determines **which code runs** based on conditions. (`if`, `else if`, `else`)

##### ==`if`== Statement
Runs a block of code **only if** the condition is `true`.
```
// Syntax
if (condition) {
   // runs if condition is true
}

// Example
int age = 20;
if (age >= 18) {
   System.out.println("You are an adult");
}
```

```
Condition
    │
    ▼
  true? ──Yes──→ Execute block
    │
    No
    │
    ▼
Skip block
```

##### ==`if-else`== Statement
Runs one block if true, **another block if false**.
```
int age = 15;
if (age >= 18) {
   System.out.println("Adult");
}
else {
   System.out.println("Minor"); // this runs
}
```

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

##### ==`if-else if-else`== Chain
Check **multiple conditions** in sequence. The first true condition runs, rest are skipped.
```
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

```
Check score >= 90? → No
Check score >= 80? → No
Check score >= 70? → Yes → Print "Grade: C" → STOP (rest skipped)
```

##### ==Nested `if`== Statements
`if` inside another `if`.
```
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

```
// ❌ Deeply nested (hard to read)
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

##### ==Real-World== `if-else` Example:
```
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
