# What Are Loops?
A loop is a way to **execute a block of code repeatedly** without writing it multiple times.

**Syntax:**
```
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

###### Java has **4 types of loops**:

|Loop|Use When|
|---|---|
|`for`|You know exactly how many times to repeat|
|`while`|You repeat while a condition is true (count unknown)|
|`do-while`|You need to run at least once, then check condition|
|`for-each`|You iterate over a collection or array|

# The ==`for`== Loop
The most commonly used loop when the **number of iterations is known**.

Syntax:
```
for(initialization; condition; update){
  //body - runs each iteration
}
```

**The three parts:**
```
for (int i = 0; i < 5; i++) {
// └────┬────┘ └──┬──┘ └┬─┘
// initialization │ update
//            condition
```

|Part|Runs|Purpose|
|---|---|---|
|Initialization|Once at start|Create/set loop variable|
|Condition|Before each iteration|If false → loop stops|
|Update|After each iteration|Change loop variable|
Basic Examples:
```
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

**Visualizing Loop Execution:**
```
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

**Looping Over Arrays:**
```
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

**Common `for` Loop Patterns:**
```
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

**Variations of `for` Loop:**
```
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


# The ==`while`== Loop
Use `while` when you **don't know how many iterations** you need in advance.  
The condition is checked **before** each iteration.
```
while (condition) {
  // body
  // must update something to eventually make condition false!
}
```

**How it works:**
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

**Basic Example:**
```
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

# The ==`do-while`== Loop
This is almost identical to the `while` loop, with one massive difference: **it guarantees the code will run at least once**, even if the condition is `false` from the very beginning. It checks the condition at the _end_ of the loop.

**Syntax:**
```
do {
  // body — runs at least once
}
while (condition); // ← semicolon required here!
```

**Execution:**
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

**Example:**
```
int i = 1;
do {
  System.out.println(i); i++;
}
while (i <= 5); // Output: 1 2 3 4 5
```

# The ==`for-each`== Loop
Introduced in Java 5, this is the **industry standard** for iterating through Arrays and Collections (like Lists). It is cleaner, easier to read, and prevents "Index Out Of Bounds" errors.

**Syntax:**
```
for (type element : collection) {
  // use element
}

// Read as: "for each element in collection"
```

**Example:**
```
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

# Loop Control: ==`break`== and ==`continue`==
Sometimes you need to interrupt a loop before it finishes naturally.

- **`break`:** Completely **exits** the loop immediately. The program moves on to the code below the loop.
- **`continue`:** Skips the rest of the **current iteration** and jumps straight back to the top of the loop to evaluate the condition for the next round.

##### ==`break`== : Exit the Loop Immediately
`break` **completely stops** the loop and jumps to the code after it.

**Syntax:**
```
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

##### ==`continue`== : Skip Current Iteration, Continue Loop
`continue` **skips the rest of the current iteration** and jumps to the next one.

**Syntax:**
```
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

##### `break` vs `continue`
```
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

# ==Nested Loops==
A loop inside of another loop. The "inner loop" will execute all of its iterations for _each single iteration_ of the "outer loop".

- **Use Cases:** Highly common when working with 2D grids, matrices, or generating tables.
- **Performance Warning:** Be careful! If your outer loop runs 100 times, and your inner loop runs 100 times, your code just executed 10,000 times (this is known as $O(n^2)$ time complexity).

Syntax:
```
for (int i = 1; i <= 3; i++) {                 // outer loop
    for (int j = 1; j <= 3; j++) {             // inner loop
        System.out.println("i=" + i + ", j=" + j);
    }
}
```

**Example:**
```
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

# Summary

|Concept|Key Point|
|---|---|
|`for`|Use when number of iterations is known|
|`while`|Use when iterating based on a condition|
|`do-while`|Use when body must run at least once|
|`for-each`|Use for clean iteration over arrays/collections|
|`break`|Exits the loop immediately|
|`continue`|Skips current iteration, moves to next|
|Labeled break/continue|Controls which loop to break/continue|
|Nested loops|Inner loop completes fully for each outer iteration|
|Nested loop complexity|Multiply iterations: outer × inner|
|`continue` + `while`|Update variable BEFORE continue to avoid infinite loop|
|For-each limitation|Cannot modify array, cannot access index|
|Infinite loop|`while(true)` or `for(;;)` — always pair with `break`|
