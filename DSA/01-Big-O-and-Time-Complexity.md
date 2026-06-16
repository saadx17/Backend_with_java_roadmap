# Big O Notation & Time Complexity

> **Phase 2 — Data Structures & Algorithms → 2.1 Complexity Analysis**
> Goal: Master Big O notation — the common complexity classes, best/worst/average case, and amortized analysis — the language for reasoning about algorithm efficiency.

---

## 0. The Big Picture

**Complexity analysis** measures how an algorithm's resource usage (time or space) **grows as the input grows**. It lets you compare algorithms *independently* of hardware, language, or compiler — the universal language of "is this fast enough?"

```
How does the work scale as n (input size) gets larger?
  10 items  -> fast
  1,000,000 items -> still fast? or does it grind to a halt?
```

> Big O is the foundation of all of Phase 2 — and it's what you use daily to judge whether code will scale (Phase 13). It's also the #1 interview topic.

---

## 1. What Big O Measures

**Big O notation** describes the **upper bound** on how an algorithm's running time (or space) grows relative to input size `n`, focusing on the **dominant term** as `n → ∞`.

### 1.1 Why "growth rate," not actual time?
Actual time depends on CPU, language, etc. — not comparable. Growth rate is what matters at scale:
```
Algorithm A: 1000n operations      (O(n))
Algorithm B: n² operations         (O(n²))
For small n, B might be faster. But as n grows, A wins decisively:
  n=10:        A=10,000   B=100        (B wins)
  n=10,000:    A=10M      B=100M       (A wins)
  n=1,000,000: A=1B       B=1,000,000B (A wins by a million-fold!)
```
> Big O captures the **asymptotic** behavior — what happens as input gets large. That's what determines if your system survives growth.

### 1.2 The rules for deriving Big O
1. **Drop constants:** `O(2n)` → `O(n)`; `O(n/2)` → `O(n)`. (Constants don't affect growth rate.)
2. **Keep only the dominant term:** `O(n² + n)` → `O(n²)`. (The fastest-growing term dominates.)
3. **Different inputs, different variables:** two separate inputs → `O(a + b)` or `O(a × b)`, not `O(n)`.
```
O(5n² + 100n + 50)  ->  O(n²)   (drop constants, keep dominant term)
```

---

## 2. The Common Complexity Classes (Memorize These)

From fastest-growing (best) to slowest-growing (worst):
| Big O | Name | Example | At n=1,000,000 |
|-------|------|---------|----------------|
| **O(1)** | Constant | Array index, HashMap get | 1 |
| **O(log n)** | Logarithmic | Binary search, balanced tree ops | ~20 |
| **O(n)** | Linear | Loop through an array | 1,000,000 |
| **O(n log n)** | Linearithmic | Efficient sorts (merge/quick) | ~20,000,000 |
| **O(n²)** | Quadratic | Nested loops, bubble sort | 1,000,000,000,000 |
| **O(2ⁿ)** | Exponential | Naive recursive Fibonacci, subsets | astronomically huge |
| **O(n!)** | Factorial | Permutations, brute-force TSP | unimaginably huge |

```
Growth (smaller = better):
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)
```

### 2.1 O(1) — Constant time
Time **independent** of input size:
```java
int first = array[0];           // direct index — O(1)
map.get(key);                    // HashMap lookup — O(1) average (Phase 1.6)
stack.push(x);                   // O(1)
```

### 2.2 O(log n) — Logarithmic
Time grows by a constant each time the input **doubles** — typically from **halving** the search space each step:
```java
// Binary search — halves the range each iteration:
int binarySearch(int[] arr, int target) {
    int lo = 0, hi = arr.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```
> O(log n) is excellent — doubling 1M to 2M items adds only **one** more step. Balanced trees (TreeMap/TreeSet — Phase 1.6) and binary search are O(log n). (Recall `git bisect`, Phase 0.4 — binary search in action.)

### 2.3 O(n) — Linear
Time proportional to input size — a single pass:
```java
int sum = 0;
for (int x : array) sum += x;    // touches every element once — O(n)
```

### 2.4 O(n log n) — Linearithmic
The best achievable for **comparison-based sorting**. Typically "do an O(log n) operation n times" or "divide-and-conquer":
```java
Arrays.sort(array);              // O(n log n) — merge/dual-pivot quicksort (Phase 2.3)
```

### 2.5 O(n²) — Quadratic
**Nested loops** over the same input — every element compared with every other:
```java
for (int i = 0; i < n; i++)
    for (int j = 0; j < n; j++)
        process(arr[i], arr[j]);  // n × n = O(n²)
```
> ⚠️ O(n²) is a common scalability killer. Fine for small n; disastrous for large n. Often a sign to use a better data structure (e.g., a HashMap to avoid the inner loop).

### 2.6 O(2ⁿ) and O(n!) — Exponential & Factorial
- **O(2ⁿ):** each element doubles the work — naive recursion exploring all subsets (e.g., naive Fibonacci — Phase 1.2):
```java
int fib(int n) {                 // O(2ⁿ) — recomputes the same values exponentially
    if (n < 2) return n;
    return fib(n-1) + fib(n-2);   // fix with memoization -> O(n), Phase 2.3 DP
}
```
- **O(n!):** all permutations (brute-force traveling salesman). Both are **intractable** beyond tiny n → need DP, greedy, or approximation.

---

## 3. Best, Worst & Average Case

The same algorithm can have **different complexities** depending on the input:
| Case | Meaning | Notation (often) |
|------|---------|------------------|
| **Best case** | The most favorable input | Ω (Omega) — lower bound |
| **Average case** | Expected over typical inputs | Θ (Theta) — tight bound |
| **Worst case** | The least favorable input | O (Big O) — upper bound |

### 3.1 Example: linear search
```java
int find(int[] arr, int target) {
    for (int i = 0; i < arr.length; i++)
        if (arr[i] == target) return i;
    return -1;
}
```
- **Best case:** target is the first element → **O(1)**.
- **Worst case:** target is last or absent → **O(n)**.
- **Average case:** target somewhere in the middle → **O(n)**.

### 3.2 Why we usually focus on worst case
> **Big O typically refers to the worst case** — it's the guarantee ("never slower than this"). Worst-case matters most for systems that must meet SLAs/latency targets (Phase 13). Some structures shine in average case but have bad worst cases (e.g., quicksort: O(n log n) average, O(n²) worst; HashMap: O(1) average, O(log n)/O(n) worst with bad hashing — recall Phase 1.6 treeification).

### 3.3 Big O / Omega / Theta (precise notation)
| Symbol | Bounds | Meaning |
|--------|--------|---------|
| **O** (Big O) | Upper | "at most this fast-growing" |
| **Ω** (Omega) | Lower | "at least this fast-growing" |
| **Θ** (Theta) | Tight | "exactly this growth" (upper = lower) |
> Casually people say "Big O" for the typical/worst case. Formally, Θ is the tight bound. For interviews, knowing O is usually enough; Θ shows precision.

---

## 4. Amortized Analysis

**Amortized analysis** gives the **average cost per operation over a sequence** — useful when *most* operations are cheap but *occasionally* one is expensive.

### 4.1 The classic example: ArrayList.add (recall Phase 1.6)
```java
list.add(x);   // usually O(1)... but occasionally O(n) when it resizes!
```
- Most `add`s just write to the array → **O(1)**.
- When full, the array **doubles** and copies all elements → **O(n)** for *that* add.
- But resizes are rare (each doubling postpones the next), so **averaged over many adds**, the cost is **amortized O(1)**.

### 4.2 Why amortized ≠ average case
- **Average case** = average over different *inputs* (probabilistic).
- **Amortized** = average over a *sequence of operations* on the same structure (guaranteed, not probabilistic).
> Amortized O(1) means: "any sequence of n adds costs O(n) total," so each add *averages* O(1) — even though a single add can spike to O(n). This is why ArrayList/HashMap are described as "amortized O(1)."

---

## 5. Practical Reference: What's "Fast Enough"?

Rough guide for ~1 second on typical hardware (interview/competitive context):
| Max n | Acceptable complexity |
|-------|------------------------|
| n ≤ 10–12 | O(n!) / O(2ⁿ) OK |
| n ≤ 100 | O(n³) |
| n ≤ 1,000–10,000 | O(n²) |
| n ≤ 1,000,000 | O(n log n) |
| n ≤ 100,000,000 | O(n) |
| huge | O(log n) / O(1) |
> If your data is large and your algorithm is O(n²), it likely won't scale — switch to a better structure/algorithm. This intuition guides design decisions (Phase 13 performance).

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Keeping constants/lower terms | Drop them: `O(2n+5)` → `O(n)` |
| Assuming O(1) is always faster than O(n) for small n | Constants matter at small scale; Big O is about growth |
| Using one variable for two inputs | `O(a + b)` / `O(a × b)`, not `O(n)` |
| Confusing average case with amortized | Inputs vs sequence-of-operations |
| Ignoring worst case for SLA-critical code | Worst case is the guarantee |
| Thinking nested loops are always O(n²) | Depends on bounds (e.g., inner loop to √n) |
| Forgetting HashMap is O(1) *average*, not guaranteed | Worst case can degrade (Phase 1.6) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Scalability (Phase 13):** an O(n²) algorithm on growing data is a classic cause of slow endpoints — Big O predicts it before it bites.
- **Data structure choice (Phase 1.6):** Big O is *why* you pick HashMap (O(1)) over a list scan (O(n)), or a TreeMap (O(log n)) for range queries.
- **Database queries (Phase 4):** the **N+1 problem** is an O(n) explosion of queries; indexing turns O(n) scans into O(log n) lookups.
- **Algorithm interviews:** every solution is judged on time/space complexity.
- **Caching (Phase 5.7):** trades O(n) recomputation for O(1) lookups.
- **Capacity estimation (Phase 16.8):** back-of-envelope sizing uses growth-rate reasoning.

---

## 8. Quick Self-Check Questions

1. What does Big O measure, and why growth rate instead of actual time?
2. State the three rules for deriving Big O from an expression.
3. List the common complexity classes from best to worst.
4. Give a code example of O(1), O(log n), O(n), O(n²).
5. Why is O(log n) so efficient (what causes it)?
6. What's the difference between best, worst, and average case? Which does Big O usually mean?
7. What is amortized analysis, and how does it differ from average case?
8. Why is ArrayList.add "amortized O(1)"?

---

## 9. Key Terms Glossary

- **Complexity analysis:** measuring how resource use grows with input size.
- **Big O (O):** upper bound on growth rate (worst case, typically).
- **Omega (Ω) / Theta (Θ):** lower bound / tight bound.
- **Asymptotic:** behavior as n → ∞.
- **Constant/log/linear/linearithmic/quadratic/exponential/factorial:** O(1), O(log n), O(n), O(n log n), O(n²), O(2ⁿ), O(n!).
- **Best / worst / average case:** complexity for favorable / unfavorable / typical inputs.
- **Amortized analysis:** average cost per operation over a sequence.
- **Dominant term:** the fastest-growing term (the one Big O keeps).

---

*This is the first note of **Section 2.1 — Complexity Analysis**.*
*Next topic: **Space Complexity & Analyzing Code**.*
