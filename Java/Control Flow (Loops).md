**Loops** repeat a block of code while a condition holds. Java has four loop forms plus control statements:

| Loop       | Use When                                             |
| ---------- | ---------------------------------------------------- |
| `for`      | You know exactly how many times to repeat            |
| `for-each` | You iterate over a collection or array               |
| `while`    | You repeat while a condition is true (count unknown) |
| `do-while` | You need to run at least once, then check condition  |

| Loop Control               | Effect                     |
| -------------------------- | -------------------------- |
| `break`                    | Exit the loop entirely     |
| `continue`                 | Skip to the next iteration |
| labeled `break`/`continue` | Target an outer loop       |

## 1. The `for` Loop
The most commonly used loop when the **number of iterations is known**.
```java title:syntax.java
for (int i = 0; i < 5; i++) {     // init; condition; update
    System.out.println(i);        // 0 1 2 3 4
}
```

**Execution order:**
```
1. init (once)           -> int i = 0
2. check condition       -> i < 5 ?  (false -> exit)
3. run body
4. update                -> i++
5. back to step 2
```

**Example:**
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

### 1.1 Variations
```java title:multi_variable.java
// Multiple variables in initialization and update
for (int i = 0, j = 10; i < j; i++, j--) {
  System.out.println("i=" + i + ", j=" + j);
}
// i=0, j=10
// i=1, j=9
// i=2, j=8
// i=3, j=7
// i=4, j=6
```

```java title:infinity_loop.java
// Infinite for loop (use with break)
for (;;) {
// runs forever until break
  System.out.println("Running...");
  break;
}
```

```java title:empty_body.java
// Empty body (rare but valid)
int i = 0;
for (; i < 5; i++); // i becomes 5
System.out.println(i); // 5
```
- The loop variable `i` is **scoped to the loop** (not visible after).

### 1.2 Nested `for` Loops
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

## 2. The Enhanced `for-each` Loop
Introduced in Java 5, this is the **industry standard** for iterating through [[Arrays]] and Collections (like Lists). It is cleaner, easier to read, and prevents "Index Out Of Bounds" errors.

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

### 2.1 When NOT to use for-each
- When you need the **index** (use a regular `for`).
- When you need to **modify the collection** during iteration (throws `ConcurrentModificationException` - use an `Iterator.remove()` or `removeIf()` instead — Phase 1.6):
  ```java
  for (String s : list) {
      // list.remove(s);   // ConcurrentModificationException!
  }
  list.removeIf(s -> s.isEmpty());   // safe alternative
  ```
- When you need to iterate **two structures in parallel** or **backwards**.

> for-each is read-only over the *iteration variable* — reassigning it doesn't change the collection (recall pass-by-value, Methods note).

## 3. The `while` Loop
Use `while` when you **don't know how many iterations** you need in advance.  
The condition is checked **before** each iteration.
```java title:syntax.java
while (condition) {
  // body
  // must update something to eventually make condition false!
}
```

**How it works:**
```
  Check condition
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
- If the condition is false initially, the body runs **zero** times.
- Forgetting to update the condition variable → **infinite loop**.

## 4. The `do-while` Loop
This is almost identical to the `while` loop, with one massive difference: **it guarantees the code will run at least once**, even if the condition is `false` from the very beginning. It checks the condition at the _end_ of the loop.
```java title:syntax.java
do {
  // body — runs at least once
}
while (condition); // ← semicolon required here!
```

**How it works:**
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

| | `while` | `do-while` |
|---|---------|-----------|
| Condition checked | Before body | After body |
| Minimum runs | 0 | 1 |
| Use for | "Maybe run" | "Run then check" (menus, retries, input) |

## 5. Loop Control: `break`, `continue`, Labels
Sometimes you need to interrupt a loop before it finishes naturally.

### 5.1 `break`
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

### 5.2 `continue`
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

### 5.3 Labeled loops
By default `break`/`continue` affect the **innermost** loop. A **label** lets you target an **outer** one:
```java
outer:                                  // label
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (i * j > 2) {
            break outer;                // breaks the OUTER loop, not just inner
        }
        System.out.println(i + "," + j);
    }
}
```
- `break label;` exits the labeled loop entirely.
- `continue label;` continues the next iteration of the labeled loop.
- Useful for breaking out of nested loops, but **use sparingly** - often a sign you should extract a method or restructure.

## 6. Loops and the Stream Alternative (preview)
For transforming/filtering collections, the **Stream API** (Phase 1.8) often replaces explicit loops with a more declarative style:
```java
// Imperative loop:
List<String> result = new ArrayList<>();
for (String s : names) {
    if (s.length() > 3) result.add(s.toUpperCase());
}

// Stream equivalent (declarative):
List<String> result = names.stream()
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase)
    .toList();
```
> Loops are fine and often faster for simple iteration; streams shine for complex pipelines and readability. Both are essential, covered fully in Phase 1.8.

## 7. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Infinite loop (condition never changes) | Ensure progress toward the exit condition |
| Off-by-one errors (`<=` vs `<`) | Carefully choose bounds; prefer for-each when possible |
| Modifying a collection during for-each | Use `Iterator.remove()` / `removeIf()` |
| `continue` skipping the update in a `while` | Update before `continue` |
| Overusing labeled break/continue | Extract a method or restructure |
| Heavy work / object creation inside loops | Hoist invariant work out of the loop |
| Using a loop where a stream is clearer | Consider streams for transform/filter pipelines |

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Batch processing** (reading large files/result sets) relies on condition-driven loops (`while (rs.next())`) — Phase 4 JDBC, Project 2.
- **`ConcurrentModificationException`** is a real bug when mutating collections during iteration — know `removeIf`/iterators (Phase 1.6).
- **Performance:** avoid expensive operations (DB calls, allocations) inside loops — a top cause of slow endpoints and the **N+1 problem** (Phase 5.4, 13).
- **Streams vs loops:** modern backend code uses streams heavily for data transformation (Phase 1.8) — but understand loops first.
- **Pagination loops** when consuming paged APIs/data (Phase 7).

## 9. Quick Self-Check Questions

1. When would you use a `for` loop vs a for-each loop?
2. What's the difference between `while` and `do-while`?
3. Why might a for-each loop throw `ConcurrentModificationException`, and how do you avoid it?
4. What's the difference between `break` and `continue`?
5. In a `for` loop, does `continue` run the update step? In a `while`?
6. What do labeled `break`/`continue` do, and when are they appropriate?
7. Give an example of converting a loop into a stream pipeline.

## 10. Key Terms Glossary

- **Loop:** a construct that repeats a block of code.
- **`for` loop:** index/counter-based iteration.
- **Enhanced for (for-each):** iterates elements of an array/`Iterable`.
- **`while` loop:** repeats while a condition is true (checked first).
- **`do-while` loop:** repeats with the condition checked after (runs ≥ once).
- **`break`:** exits the current loop.
- **`continue`:** skips to the next iteration.
- **Labeled loop:** a named loop targeted by `break`/`continue`.
- **Infinite loop:** a loop whose condition never becomes false.
- **`ConcurrentModificationException`:** thrown when modifying a collection during iteration.
- **Off-by-one error:** boundary mistake in loop bounds.