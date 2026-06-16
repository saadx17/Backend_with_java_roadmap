# Space Complexity & Analyzing Code

> **Phase 2 — Data Structures & Algorithms → 2.1 Complexity Analysis**
> Goal: Master space complexity (including recursion's call-stack cost) and develop a systematic method for analyzing any piece of code for its time and space complexity.

---

## 0. The Big Picture

Time complexity (previous note) measures **how long**; **space complexity** measures **how much extra memory** an algorithm uses as input grows. Together they let you evaluate the full cost — and often there's a **trade-off** between them (the classic time-space trade-off).

```
Time complexity:  how the running time grows with n
Space complexity: how the extra MEMORY grows with n
```

> Space matters as much as time in real systems — an algorithm that's fast but uses O(n²) memory can OOM your service (recall Phase 1.11). And the call stack of deep recursion is space too (Phase 1.2/0.1).

---

## 1. What Space Complexity Measures

Space complexity is the **extra (auxiliary) memory** an algorithm needs **beyond the input itself**, as a function of input size `n`.

### 1.1 Auxiliary space vs total space
| | Meaning |
|---|---------|
| **Auxiliary space** | Extra memory used by the algorithm (not counting the input) |
| **Total space** | Auxiliary + the input itself |
> Usually we mean **auxiliary space** — the input is given; we care about what the *algorithm* adds. ("This sort uses O(n) extra space" = auxiliary.)

### 1.2 What counts as space
- Variables, data structures created (`new` — heap, Phase 1.11).
- The **recursion call stack** (each frame uses stack space — Phase 0.1/1.2).
- Not the input (for auxiliary space).

---

## 2. Common Space Complexities

| Space | Meaning | Example |
|-------|---------|---------|
| **O(1)** | Constant extra space | A few variables, in-place swap |
| **O(log n)** | Logarithmic | Recursion depth of balanced divide-and-conquer |
| **O(n)** | Linear | A new array/list/map proportional to input |
| **O(n²)** | Quadratic | A 2D matrix of size n×n |

### 2.1 O(1) space — in place
Uses a fixed amount of extra memory regardless of input:
```java
void reverse(int[] arr) {                 // O(1) space — reverses in place
    int lo = 0, hi = arr.length - 1;
    while (lo < hi) {
        int temp = arr[lo];               // just a few variables
        arr[lo++] = arr[hi];
        arr[hi--] = temp;
    }
}
```

### 2.2 O(n) space — new structure proportional to input
```java
int[] doubled = new int[arr.length];      // O(n) extra space
for (int i = 0; i < arr.length; i++)
    doubled[i] = arr[i] * 2;

Set<Integer> seen = new HashSet<>();       // grows up to O(n)
```

### 2.3 O(n²) space — 2D structures
```java
int[][] matrix = new int[n][n];           // O(n²) space (n rows × n columns)
```

---

## 3. Recursion and the Call Stack (Crucial)

> **Recursion uses space** — each recursive call adds a **stack frame** (Phase 0.1/1.2). The space complexity of recursion is proportional to its **maximum depth**.

### 3.1 Linear recursion → O(n) space
```java
long factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);          // depth = n -> O(n) stack space
}
```
Even though it does O(n) *time*, it also uses O(n) *space* on the call stack (n frames deep). Deep recursion risks **`StackOverflowError`** (recall Phase 0.1 guard page, Phase 1.2).

### 3.2 Iteration → O(1) space
The iterative version of the same logic uses constant space (no stack frames):
```java
long factorial(int n) {
    long result = 1;
    for (int i = 2; i <= n; i++) result *= i;   // O(1) space
    return result;
}
```
> **Converting recursion to iteration trades stack space for O(1)** — a common optimization for deep recursion (Phase 1.2).

### 3.3 Divide-and-conquer recursion → O(log n) space
Balanced recursion (like binary search or merge sort's recursion tree depth) goes only O(log n) deep:
```java
// Binary search recursion: depth = log n -> O(log n) stack space
```
> Be careful counting recursion space: it's the **maximum depth** of the stack at any one time, not the total number of calls. (A tree-recursive function making 2ⁿ calls may still only be O(n) deep.)

---

## 4. The Time-Space Trade-off

Often you can **use more memory to save time**, or vice versa. Recognizing this trade-off is a key skill.

### 4.1 Classic example: two-sum
```java
// Brute force: O(n²) time, O(1) space — check every pair:
for (int i = 0; i < n; i++)
    for (int j = i+1; j < n; j++)
        if (arr[i] + arr[j] == target) return new int[]{i, j};

// HashMap: O(n) time, O(n) space — trade space for speed:
Map<Integer, Integer> seen = new HashMap<>();
for (int i = 0; i < n; i++) {
    int need = target - arr[i];
    if (seen.containsKey(need)) return new int[]{seen.get(need), i};
    seen.put(arr[i], i);                  // uses O(n) memory to avoid the inner loop
}
```
> The HashMap version uses O(n) **space** to cut **time** from O(n²) to O(n). **Caching/memoization** (Phase 2.3 DP) is the same idea: store results to avoid recomputation. Most real systems lean toward spending memory to save time (caching — Phase 5.7).

---

## 5. How to Analyze Code (A Systematic Method)

A repeatable process for finding the complexity of any code:

### 5.1 Time complexity steps
```
1. Identify the input size (n) — and watch for MULTIPLE inputs.
2. Find loops: each loop over n -> O(n). Nested loops MULTIPLY.
3. Find recursion: depth × work-per-call (or use a recurrence).
4. Account for hidden costs: library calls (sort = O(n log n), contains on a list = O(n)).
5. Add sequential parts; multiply nested parts.
6. Drop constants & keep the dominant term.
```

### 5.2 Worked examples
```java
// Example 1: single loop -> O(n)
for (int i = 0; i < n; i++) sum += arr[i];

// Example 2: nested loops -> O(n²)
for (int i = 0; i < n; i++)
    for (int j = 0; j < n; j++) process();

// Example 3: SEQUENTIAL loops -> O(n) + O(n) = O(n) (not O(n²)!)
for (int i = 0; i < n; i++) a();          // O(n)
for (int j = 0; j < n; j++) b();          // O(n)   -> total O(n)

// Example 4: inner loop bounded by i -> still O(n²)
for (int i = 0; i < n; i++)
    for (int j = 0; j < i; j++) c();      // 0+1+2+...+(n-1) = n(n-1)/2 = O(n²)

// Example 5: loop that halves -> O(log n)
for (int i = n; i > 0; i /= 2) d();       // halving -> log n iterations

// Example 6: TWO different inputs -> O(a × b), NOT O(n²)
for (int i = 0; i < a; i++)
    for (int j = 0; j < b; j++) e();      // O(a × b)
```

### 5.3 Watch for hidden costs (a common mistake)
Library calls have their own complexity — don't treat them as O(1):
```java
for (int i = 0; i < n; i++) {
    if (list.contains(x)) ...   // list.contains is O(n)! -> total O(n²)!
}
// Fix: use a HashSet (contains is O(1)) -> total O(n)

for (int i = 0; i < n; i++) {
    Collections.sort(sub);      // sort is O(k log k) -> multiplies in!
}

String s = "";
for (...) s += x;               // String += in a loop is O(n²)! (Phase 1.4)
```
> ⚠️ **Hidden costs inside loops** are the most common analysis error: `list.contains` (O(n)), `String +=` (O(n²) — Phase 1.4), sorting inside a loop, or a DB query per iteration (the N+1 problem — Phase 5.4).

### 5.4 Space analysis steps
```
1. Count extra data structures created (relative to n).
2. Add recursion stack depth.
3. Don't count the input (for auxiliary space).
4. Keep the dominant term.
```

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Adding nested-loop complexities instead of multiplying | Nested = multiply; sequential = add |
| Forgetting recursion's stack space | Count max depth as space |
| Treating library calls as O(1) | `contains`/`sort`/`String +=` have real costs |
| Counting total recursive calls as space | It's the max **depth**, not total calls |
| Ignoring multiple inputs | Use separate variables (`O(a×b)`) |
| Optimizing time while ignoring space (or vice versa) | Consider both; mind the trade-off |
| Assuming "in place" means O(1) when recursion is used | Recursion adds stack space |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Memory safety (Phase 1.11, 13):** an O(n²)-space algorithm on large data → `OutOfMemoryError`. Loading a whole dataset into memory vs streaming it (Phase 1.9 `Files.lines`) is a space decision.
- **Caching/memoization (Phase 5.7, 2.3):** deliberately spend memory to save time — the time-space trade-off applied.
- **N+1 problem (Phase 5.4):** the canonical "hidden cost in a loop" (a query per element) → O(n) DB round-trips; fix with joins/batching.
- **`String +=` / `contains` in loops:** real performance bugs the analysis method catches (Phase 1.4, 1.6).
- **Recursion limits:** deep recursion → `StackOverflowError`; convert to iteration for large inputs (Phase 1.2).
- **Capacity planning (Phase 16.8):** estimating memory/throughput uses this growth reasoning.

---

## 8. Quick Self-Check Questions

1. What does space complexity measure, and what's auxiliary vs total space?
2. Why does recursion use space, and what determines its space complexity?
3. What's the space complexity of recursive vs iterative factorial?
4. Explain the time-space trade-off with the two-sum example.
5. For nested loops, do you add or multiply complexities? For sequential loops?
6. Why is `for (j=0; j<i; j++)` inside `for (i=0; i<n; i++)` still O(n²)?
7. What are "hidden costs," and name three common ones inside loops.
8. How do two different input sizes affect the complexity expression?

---

## 9. Key Terms Glossary

- **Space complexity:** how extra memory grows with input size.
- **Auxiliary space:** extra memory beyond the input.
- **Total space:** auxiliary + input.
- **In-place:** O(1) auxiliary space (mutates the input).
- **Call stack space:** memory used by recursion frames (∝ max depth).
- **Time-space trade-off:** using more memory to save time (or vice versa).
- **Memoization:** caching results to avoid recomputation (trades space for time).
- **Hidden cost:** non-obvious complexity of library calls inside loops.
- **Sequential vs nested:** add complexities vs multiply them.

---

*Previous topic: **Big O Notation & Time Complexity**.*
*This completes **Section 2.1 — Complexity Analysis**.*
*Next section in roadmap: **2.2 Data Structures**.*
