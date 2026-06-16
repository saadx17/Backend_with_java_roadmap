**Conditional statements** in Java allow your program to evaluate boolean expressions and execute different paths of code based on whether those expressions are `true` or `false`.
This note covers **conditional statements** & [[Control Flow (Loops)]] are the next note.

Java's tools:

| Construct | Use |
|-----------|-----|
| `if` / `else if` / `else` | General boolean branching |
| `switch` statement | Multi-way branch on a value |
| `switch` expression (Java 14+) | Multi-way branch that **returns a value** |
| Pattern matching in `switch` (Java 17+/21) | Branch on **type** and deconstruct |

## 1. `if-else`
The statement in Java is a fundamental decision-making control structure used to execute different blocks of code based on whether a boolean condition evaluates to `true` or `false`.

### 1.1 `if` Statement
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

### 1.2 `if-else` Statement
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

### 1.3 `if-else` Chain
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

### 1.4 Nested `if` Statements
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

### 1.5 Rules
- The condition **must** be a `boolean` (Java has no "truthy" numbers like C/JS - `if (1)` won't compile).
- Braces `{}` are optional for single statements but **always use them** (prevents the "dangling else" / accidental-statement bugs):
  ```java
  if (x > 0)
      doA();
      doB();   // BUG: NOT part of the if — always runs! Braces would prevent this.
  ```
- `else if` is just a nested `if` in the `else` branch.

### 1.6 Guard clauses (clean-code style)
Prefer early returns over deep nesting:
```java
// Instead of deeply nested ifs:
public void process(User u) {
    if (u == null) return;                 // guard
    if (!u.isActive()) return;             // guard
    // main logic here, un-nested
}
```

## 2. `switch` Statement
`switch` is used when you have **one variable** and want to compare it against **multiple specific values**.

### 2.1 Traditional `switch`
Branches on a single value, matching it against `case` labels:
```java
switch (day) {
    case 1:
        System.out.println("Monday");
        break;                 // REQUIRED to stop fall-through
    case 6:
    case 7:                    // stacked cases (1-6 fall into 7)
        System.out.println("Weekend");
        break;
    default:
        System.out.println("Weekday");
}
```

####  The fall-through trap
> Without `break`, execution **falls through** to the next case, a classic bug.
```java
switch (x) {
    case 1: System.out.println("one");    // no break -> falls through!
    case 2: System.out.println("two");    // runs too if x==1
}
```
**Note:** *Fall-through can be used **intentionally**.*

#### Allowed switch types

`byte`, `short`, `char`, `int` (and their wrappers), `String` (Java 7+), and **enums**. Not `long`, `float`, `double`, or `boolean`.

#### `switch` on enums
```java
switch (status) {
    case ACTIVE:   ... break;   // no enum-type prefix needed
    case INACTIVE: ... break;
}
```

##### Traditional `:` Colon `yield`
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

### 2.2 Enhanced `switch` (Java 14+)
The new switch is cleaner, **no break needed**, can return a value.

#### Arrow labels - no fall-through, no break
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
- `->` runs only the matched branch - **no `break` needed**, **no fall-through**.
- Multiple labels per case, comma-separated.

**Multiple values per case:**
```java title:switch_case.java
int day = 6;

String type = switch (day) {
  case 1, 2, 3, 4, 5 -> "Weekday";
  case 6, 7 -> "Weekend";
  default -> "Invalid";
};

System.out.println(type); // Weekend
```

#### `switch` as an expression
 Returns a value.
```java title:expression_switch.java
String type = switch (day) {
    case SATURDAY, SUNDAY -> "Weekend";
    case MONDAY           -> "Start";
    default               -> "Midweek";
};                       // note the semicolon — it's an expression/assignment
```

#### The `yield` keyword
In Java, the **`yield` keyword** is used inside **`switch` expressions** to return a value from a multi-line code block. Finalized in **Java 14**, it acts like a localized `return` statement tailored exclusively for your `switch` logic.

For multi-statement branches, when a branch needs multiple statements, use a block and `yield` to return its value:
```java title:yield.java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY -> 7;
    default -> {
        int len = day.toString().length();   // multi-statement block
        yield len;                            // 'yield' returns the block's value
    }
};
```
### 2.3 Exhaustiveness
A switch **expression must be exhaustive**, cover all possible values or include a `default`. For **enums** and **sealed types**, the compiler can verify all cases are covered, allowing you to omit `default` (recall Sealed/Enums notes) and it flags you if a new constant/subtype is added.

| | Traditional `switch` | Enhanced `switch` |
|---|---------------------|-------------------|
| Syntax | `case x:` + `break` | `case x ->` |
| Fall-through | Yes (bug-prone) | No |
| Returns a value | No (statement only) | Yes (expression) |
| Multi-value case | Stacked `case`s | `case a, b, c ->` |
| Multi-statement value | n/a | block + `yield` |

## 3. Pattern Matching in `switch`
**Pattern Matching for Switch** simplifies code by allowing you to check the **class type of an object** directly inside `case` labels, eliminating verbose `instanceof` checks and manual casting. Introduced as a preview in **Java 17 (JEP 406)**, it became a standard feature in **Java 21 (JEP 441**).

### 3.1 Evolution of Switch in Java
```
Java 1   → switch statement (int, char only)
Java 7   → switch with String support
Java 14  → switch expressions (arrow syntax, yield) [preview]
Java 14  → switch expressions STANDARD
Java 17  → pattern matching for switch [preview]
Java 21  → pattern matching for switch STANDARD ✅
```

**Progression:**
- Old:     switch (x) { case 1: ...; break; }
- Modern:  switch (x) { case 1 -> ...; }
- Newest:  switch (obj) { case String s when s.length() > 5 -> ...; }

### 3.2 The Problem it Solves
Pattern Matching for Switch solves the problems of verbose `instanceof` chains, unsafe manual type-casting, and rigid restriction on data types in traditional Java `switch` statements. It transforms the `switch` construct from a simple primitive value-wrapper into a powerful tool for complex, data-driven control flow.

#### Before Java 17 → ugly `instanceof` chain
```java title:ugly_syntax.java
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

#### After Java 17+ → type pattern matching in switch
```java title:clean_syntax.java
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

### 3.3 Type patterns
```java
static String describe(Object o) {
    return switch (o) {
        case Integer i -> "int: " + i;   // matches type AND binds 'i'
        case String s  -> "string of length " + s.length();
        case int[] arr -> "array of " + arr.length;
        case null      -> "null";        // can handle null explicitly
        default        -> "unknown";
    };
}
```
- Each `case Type var` checks the runtime type and binds the variable.
- `case null` lets `switch` handle null without throwing (traditionally switch throws NPE on null).

### 3.4 Guarded patterns (`when`)
Add a condition to a pattern:
```java
String size = switch (shape) {
    case Circle c when c.radius() > 10 -> "big circle";
    case Circle c                       -> "small circle";
    case Square s                       -> "square";
    default                             -> "other";
};
```

### 3.5 With sealed types + records (the powerful combo)
Recall the Sealed and Records notes - this is where it all comes together:
```java
sealed interface Shape permits Circle, Square {}
record Circle(double r) implements Shape {}
record Square(double s) implements Shape {}

double area(Shape shape) {
    return switch (shape) {                  // exhaustive — no default needed
        case Circle c -> Math.PI * c.r() * c.r();
        case Square s -> s.s() * s.s();
    };
}
```
**Record deconstruction patterns** (Java 21) go further, destructuring components directly:
```java
case Circle(double r) -> Math.PI * r * r;    // bind 'r' straight from the record
```

## 5. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Missing `break` → fall-through | Use enhanced `switch` (`->`) or add `break` |
| Omitting braces in `if` | Always use `{}` |
| Using `==` for objects in conditions | Use `.equals()` |
| `switch` expression not exhaustive | Cover all cases or add `default` |
| Forgetting `yield` in multi-statement arrow branch | Use `yield` to return the value |
| Switching on a `null` (NPE) | Handle `case null` (pattern switch) or null-check first |
| Deeply nested `if`s | Use guard clauses / early returns |

## 6. Connection to Backend / Spring (Why This Matters Later)

- **Enhanced switch expressions** make mapping enums → values/DTOs concise and safe (no fall-through bugs).
- **Pattern matching + sealed types + records** model and handle **result/event/state** types exhaustively — increasingly idiomatic in modern Java backends (Phases 11, 12).
- **Exhaustiveness checking** is a safety net: adding a new state/variant forces you to update every switch.
- **Guard clauses** keep service methods flat and readable (Phase 14 clean code).
- Pattern matching replaces verbose **visitor patterns** and `instanceof` chains in domain logic.

## 7. Quick Self-Check Questions

1. Why won't `if (1)` compile in Java?
2. What is switch fall-through, and how does the enhanced switch avoid it?
3. What types can a traditional `switch` operate on?
4. What's the difference between a switch statement and a switch expression?
5. When do you need `yield`?
6. When can you omit `default` in a switch expression?
7. How does pattern matching in switch improve on `if-else instanceof` chains?
8. How do `when` guards and record deconstruction patterns work?

## 8. Key Terms

- **Control flow:** statements that determine which code executes.
- **`if`/`else`:** boolean-based branching.
- **Guard clause:** an early return that avoids nesting.
- **`switch` statement:** multi-way branch on a value.
- **Fall-through:** continuing into the next case without `break`.
- **Enhanced switch (`->`):** arrow-label switch; no fall-through; can be an expression.
- **Switch expression:** a switch that produces a value.
- **`yield`:** returns a value from a multi-statement switch branch.
- **Exhaustiveness:** covering all possible cases (compiler-checked for enums/sealed).
- **Pattern matching in switch:** branching on type (and deconstructing) within a switch.
- **Guarded pattern (`when`):** a pattern with an extra boolean condition.
- **Record deconstruction pattern:** destructuring a record's components in a pattern.