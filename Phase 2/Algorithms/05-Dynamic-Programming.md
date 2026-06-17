# Dynamic Programming

> **Phase 2 — Data Structures & Algorithms → 2.3 Algorithms**
> Goal: Master dynamic programming — memoization (top-down) and tabulation (bottom-up) — and the classic DP problems (Fibonacci, coin change, LCS, LIS, 0/1 knapsack, edit distance).

---

## 0. The Big Picture

**Dynamic Programming (DP)** solves problems by breaking them into **overlapping subproblems** and **storing** each subproblem's result so it's computed **only once**. It transforms exponential brute-force/recursion (Phase 2.3 backtracking) into polynomial time.

```
Naive recursion: recomputes the same subproblems exponentially -> O(2ⁿ)
DP: compute each subproblem ONCE, store and reuse -> O(n) or O(n²)
```

> DP is the most feared interview topic — but it's just **smart recursion with caching**. The hard part is *recognizing* a DP problem and *defining the state*. This builds directly on recursion (Phase 2.3) and the time-space trade-off (Phase 2.1).

---

## 1. When Does DP Apply? (Two Requirements)

A problem is a DP candidate if it has **both**:
| Requirement | Meaning |
|-------------|---------|
| **Overlapping subproblems** | The same subproblems are solved repeatedly (so caching helps) |
| **Optimal substructure** | The optimal solution is built from optimal solutions of subproblems |

### 1.1 The motivating example: Fibonacci
```java
// Naive recursion — O(2ⁿ): fib(5) recomputes fib(3), fib(2), ... many times
int fib(int n) {
    if (n < 2) return n;
    return fib(n-1) + fib(n-2);   // OVERLAPPING: fib(n-2) computed twice, etc.
}
```
```
fib(5) call tree (note repeated fib(3), fib(2), ...):
            fib(5)
           /      \
       fib(4)     fib(3)      <- fib(3) computed AGAIN
       /    \     /    \
   fib(3) fib(2) fib(2) fib(1)   <- exponential repetition
```
> The repeated subproblems (overlapping) are exactly what DP eliminates by **caching** — turning O(2ⁿ) into O(n).

---

## 2. The Two DP Approaches

### 2.1 Top-Down (Memoization) — recursion + cache
Write the natural recursion, but **cache** each result so it's computed once. "Top-down" because it starts from the original problem and recurses down.
```java
int fib(int n, int[] memo) {
    if (n < 2) return n;
    if (memo[n] != 0) return memo[n];          // already computed? return cached
    memo[n] = fib(n-1, memo) + fib(n-2, memo); // compute & CACHE
    return memo[n];
}
// O(n) time, O(n) space (cache + recursion stack)
```
- ✅ Easy: just add a cache to the recursion.
- ⚠️ Uses recursion stack (deep → StackOverflow, Phase 2.1).

### 2.2 Bottom-Up (Tabulation) — iterative table filling
Build a **table** of subproblem solutions iteratively, from the smallest subproblems up to the answer. "Bottom-up" because it starts from base cases.
```java
int fib(int n) {
    if (n < 2) return n;
    int[] dp = new int[n + 1];
    dp[0] = 0; dp[1] = 1;                       // base cases
    for (int i = 2; i <= n; i++)
        dp[i] = dp[i-1] + dp[i-2];             // build up from smaller subproblems
    return dp[n];
}
// O(n) time, O(n) space — no recursion
```

### 2.3 Space optimization
Often you only need the last few subproblems → reduce space to O(1):
```java
int fib(int n) {
    if (n < 2) return n;
    int prev = 0, cur = 1;
    for (int i = 2; i <= n; i++) {
        int next = prev + cur;
        prev = cur; cur = next;                // only keep the last two
    }
    return cur;
}
// O(n) time, O(1) space!
```

### 2.4 Top-down vs bottom-up
| | Memoization (top-down) | Tabulation (bottom-up) |
|---|------------------------|------------------------|
| Style | Recursion + cache | Iterative table |
| Intuition | Natural (mirrors recursion) | Requires ordering subproblems |
| Space | Cache + recursion stack | Table (often optimizable) |
| Risk | StackOverflow (deep) | None |
| Computes | Only needed subproblems | All subproblems (usually) |
> Both are O(same). Use **memoization** when the recursion is natural and not all subproblems are needed; use **tabulation** to avoid stack issues and enable space optimization.

---

## 3. The DP Problem-Solving Framework

A repeatable method for cracking DP problems:
```
1. DEFINE THE STATE: what does dp[i] (or dp[i][j]) represent? (the hardest step)
2. FIND THE RECURRENCE: how does dp[i] relate to smaller states?
3. IDENTIFY BASE CASES: the smallest subproblems' values.
4. DETERMINE THE ORDER: in what order to fill the table (so dependencies are ready).
5. (OPTIONAL) OPTIMIZE SPACE: keep only what's needed.
```
> The art of DP is **step 1 (defining the state)** and **step 2 (the recurrence)**. Once you have those, the code is mechanical.

---

## 4. Classic DP Problems

### 4.1 Coin Change (minimum coins to make an amount)
**State:** `dp[a]` = fewest coins to make amount `a`. **Recurrence:** `dp[a] = min(dp[a - coin] + 1)` over all coins.
```java
int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);               // "infinity" sentinel
    dp[0] = 0;                                  // base: 0 coins for amount 0
    for (int a = 1; a <= amount; a++)
        for (int coin : coins)
            if (coin <= a)
                dp[a] = Math.min(dp[a], dp[a - coin] + 1);
    return dp[amount] > amount ? -1 : dp[amount];
}
// O(amount × coins) time
```

### 4.2 0/1 Knapsack (max value within a weight limit, each item once)
**State:** `dp[i][w]` = max value using the first `i` items with capacity `w`. **Recurrence:** for each item, take it or skip it.
```java
int knapsack(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[][] dp = new int[n + 1][capacity + 1];
    for (int i = 1; i <= n; i++)
        for (int w = 0; w <= capacity; w++) {
            dp[i][w] = dp[i-1][w];             // SKIP item i
            if (weights[i-1] <= w)             // can we TAKE item i?
                dp[i][w] = Math.max(dp[i][w], dp[i-1][w - weights[i-1]] + values[i-1]);
        }
    return dp[n][capacity];
}
// O(n × capacity) time
```
> The "take it or leave it" choice is the heart of 0/1 knapsack — a template for many resource-allocation problems.

### 4.3 Longest Common Subsequence (LCS)
**State:** `dp[i][j]` = LCS length of the first `i` chars of A and first `j` of B. **Recurrence:** match → diagonal+1; else → max of skipping one char.
```java
int lcs(String a, String b) {
    int m = a.length(), n = b.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            if (a.charAt(i-1) == b.charAt(j-1))
                dp[i][j] = dp[i-1][j-1] + 1;            // chars match
            else
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);  // skip one
    return dp[m][n];
}
// O(m × n) time
```
> LCS is the basis of **diff tools** (recall `git diff`, Phase 0.4!) and DNA sequence alignment.

### 4.4 Longest Increasing Subsequence (LIS)
**State:** `dp[i]` = length of the LIS ending at index `i`.
```java
int lis(int[] nums) {
    int[] dp = new int[nums.length];
    Arrays.fill(dp, 1);
    int max = 1;
    for (int i = 1; i < nums.length; i++)
        for (int j = 0; j < i; j++)
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
                max = Math.max(max, dp[i]);
            }
    return max;
}
// O(n²) DP; can be optimized to O(n log n) with binary search (Phase 2.3.2)
```

### 4.5 Edit Distance (Levenshtein)
Minimum insert/delete/replace operations to transform string A into B. **State:** `dp[i][j]` = edit distance between prefixes.
```java
int editDistance(String a, String b) {
    int m = a.length(), n = b.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 0; i <= m; i++) dp[i][0] = i;          // delete all
    for (int j = 0; j <= n; j++) dp[0][j] = j;          // insert all
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            if (a.charAt(i-1) == b.charAt(j-1))
                dp[i][j] = dp[i-1][j-1];                // no op
            else
                dp[i][j] = 1 + Math.min(dp[i-1][j-1],   // replace
                            Math.min(dp[i-1][j],        // delete
                                     dp[i][j-1]));      // insert
    return dp[m][n];
}
// O(m × n) — powers spell-checkers, fuzzy search, diff
```

### 4.6 Classic DP problems summary
| Problem | State `dp[...]` | Complexity |
|---------|-----------------|------------|
| Fibonacci | `dp[i]` = i-th number | O(n) |
| Coin change | `dp[a]` = min coins for amount a | O(amount × coins) |
| 0/1 Knapsack | `dp[i][w]` = max value, i items, cap w | O(n × cap) |
| LCS | `dp[i][j]` = LCS of prefixes | O(m × n) |
| LIS | `dp[i]` = LIS ending at i | O(n²) → O(n log n) |
| Edit distance | `dp[i][j]` = edits between prefixes | O(m × n) |

---

## 5. DP vs Backtracking vs Greedy
| | Explores | Guarantees optimal? | Use when |
|---|----------|---------------------|----------|
| **Backtracking** (Phase 2.3) | All possibilities (with pruning) | Yes (finds all/valid) | Enumerate/constraint problems |
| **DP** | Overlapping subproblems (cached) | **Yes** | Overlapping subproblems + optimal substructure |
| **Greedy** (next note) | One locally-best choice each step | Only if the greedy property holds | When local optima → global optimum |
> If backtracking is too slow due to **repeated subproblems**, add **memoization** → it becomes DP. If a **locally optimal choice always works**, greedy is simpler and faster (next note).

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Not recognizing overlapping subproblems | Look for repeated recursive calls → memoize |
| Wrong state definition | The hardest step — be precise about what `dp[i]` means |
| Wrong fill order (tabulation) | Ensure dependencies are computed first |
| Missing/incorrect base cases | They anchor the recurrence |
| Forgetting space optimization | Often only the last row/few values are needed |
| Deep memoized recursion → StackOverflow | Use tabulation for large inputs |
| Using DP when greedy suffices | Greedy is simpler if the property holds |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Diff/merge** (`git diff`, code review — Phase 0.4/6): LCS/edit distance.
- **Fuzzy search / spell-check / autocomplete** (Phase 16.4): edit distance.
- **Resource allocation / optimization** (scheduling, capacity — Phase 16.8): knapsack-style DP.
- **Caching as memoization** (Phase 5.7): the same "compute once, reuse" principle at the system level.
- **Interview prep** (Phase 2.4): DP is a major, high-difficulty category.
- **Time-space trade-off** (Phase 2.1): DP trades memory for dramatically less time.

---

## 8. Quick Self-Check Questions

1. What two properties must a problem have for DP to apply?
2. Why is naive recursive Fibonacci O(2ⁿ), and how does DP fix it?
3. What's the difference between memoization and tabulation?
4. How can you often reduce DP space, and give an example?
5. What are the 5 steps of the DP framework?
6. Define the state and recurrence for coin change and 0/1 knapsack.
7. What real tools use LCS / edit distance?
8. How are backtracking, DP, and greedy related?

---

## 9. Key Terms Glossary

- **Dynamic programming (DP):** solving overlapping subproblems by caching results.
- **Overlapping subproblems:** the same subproblems recur.
- **Optimal substructure:** optimal solution built from optimal subsolutions.
- **Memoization (top-down):** recursion + cache.
- **Tabulation (bottom-up):** iterative table filling.
- **State:** what a DP cell represents (`dp[i]`).
- **Recurrence:** how a state relates to smaller states.
- **Base case:** smallest-subproblem value.
- **Space optimization:** keeping only needed prior states.
- **LCS / LIS / knapsack / edit distance:** classic DP problems.

---

*Previous topic: **Recursion & Backtracking**.*
*Next topic: **Greedy Algorithms & Divide and Conquer**.*
