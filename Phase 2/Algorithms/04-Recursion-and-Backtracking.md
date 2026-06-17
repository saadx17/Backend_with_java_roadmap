# Recursion & Backtracking

> **Phase 2 — Data Structures & Algorithms → 2.3 Algorithms**
> Goal: Master recursion fundamentals (recap + depth) and backtracking — the systematic technique for exploring all possibilities (N-Queens, Sudoku, permutations, combinations, subsets).

---

## 0. The Big Picture

**Recursion** (a method calling itself, recall Phase 1.2) is the foundation. **Backtracking** is a recursion-based technique that **builds candidates incrementally** and **abandons ("backtracks") a path** as soon as it can't lead to a solution — systematically exploring a tree of possibilities.

```
Backtracking = try a choice -> recurse -> if it fails or is exhausted, UNDO the choice and try the next
```

> Backtracking solves "find all/any valid configurations" problems: permutations, combinations, subsets, N-Queens, Sudoku, maze-solving, word search. It's a core interview pattern (Phase 2.4) and the basis for constraint-satisfaction.

---

## 1. Recursion Recap (from Phase 1.2)

Every recursion needs:
1. **Base case** — when to stop (no further recursion).
2. **Recursive case** — call itself on a **smaller** subproblem, progressing toward the base case.
```java
long factorial(int n) {
    if (n <= 1) return 1;            // base case
    return n * factorial(n - 1);    // recursive case (smaller n)
}
```

### 1.1 The call stack & space (recall Phase 2.1)
Each recursive call adds a **stack frame** → recursion space = **maximum depth** (Phase 0.1/2.1). Deep recursion risks **`StackOverflowError`**. Java has **no tail-call optimization** (Phase 1.2), so very deep recursion must be converted to iteration.

### 1.2 Types of recursion
| Type | Example |
|------|---------|
| **Linear** | factorial (one self-call) |
| **Tree** | Fibonacci, divide-and-conquer (multiple self-calls) |
| **Mutual** | A calls B, B calls A |
> ⚠️ Naive **tree recursion** can be exponential (naive Fibonacci is O(2ⁿ) — recomputes subproblems). Fix with **memoization** → Dynamic Programming (next note).

---

## 2. Thinking Recursively

The mindset: **trust the recursion**. Assume the recursive call solves the smaller problem correctly, then combine.
```
To solve problem(n):
  1. Handle the base case (smallest input).
  2. Assume problem(n-1) is solved.
  3. Use that result to solve problem(n).
```
**Example: reverse a string recursively**
```java
String reverse(String s) {
    if (s.isEmpty()) return s;                  // base case
    return reverse(s.substring(1)) + s.charAt(0);   // reverse the rest, then prepend first
}
```
> Recursion shines for **inherently recursive** structures: trees (Phase 2.2 traversals), graphs (DFS), divide-and-conquer (merge/quick sort), and nested data (JSON).

---

## 3. Backtracking — The Core Technique

Backtracking explores a **decision tree**: at each step, make a choice, recurse, then **undo the choice** to try alternatives. It's "DFS over the space of candidate solutions" with pruning.

### 3.1 The universal backtracking template
```
backtrack(current_state):
    if (current_state is a complete solution):
        record/return it
        return
    for each possible choice from current_state:
        if (choice is valid):           # PRUNE invalid choices early
            make the choice
            backtrack(new_state)        # recurse
            undo the choice             # BACKTRACK
```
> The three pillars: **choose** → **explore (recurse)** → **un-choose (backtrack)**. The "undo" step is what makes it backtracking. **Pruning** (skipping invalid branches early) is what makes it efficient.

---

## 4. Classic Backtracking Problems

### 4.1 Subsets (the power set)
Generate all subsets of `[1, 2, 3]` (each element is in or out):
```java
List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}
void backtrack(int[] nums, int start, List<Integer> current, List<List<Integer>> result) {
    result.add(new ArrayList<>(current));        // every state is a valid subset
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);                    // CHOOSE
        backtrack(nums, i + 1, current, result); // EXPLORE
        current.remove(current.size() - 1);      // UN-CHOOSE (backtrack)
    }
}
// Result: [], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]   -> 2ⁿ subsets
```

### 4.2 Permutations
All orderings of `[1, 2, 3]`:
```java
List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, new ArrayList<>(), new boolean[nums.length], result);
    return result;
}
void backtrack(int[] nums, List<Integer> current, boolean[] used, List<List<Integer>> result) {
    if (current.size() == nums.length) {          // complete permutation
        result.add(new ArrayList<>(current));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;                    // skip already-used (pruning)
        used[i] = true; current.add(nums[i]);     // CHOOSE
        backtrack(nums, current, used, result);   // EXPLORE
        used[i] = false; current.remove(current.size() - 1);  // UN-CHOOSE
    }
}
// 3! = 6 permutations
```

### 4.3 Combinations
Choose k of n elements (order doesn't matter) — same as subsets but with a size constraint and a `start` index to avoid duplicates.
```java
void backtrack(int n, int k, int start, List<Integer> current, List<List<Integer>> result) {
    if (current.size() == k) { result.add(new ArrayList<>(current)); return; }
    for (int i = start; i <= n; i++) {
        current.add(i);
        backtrack(n, k, i + 1, current, result);   // i+1 -> no reuse, no duplicates
        current.remove(current.size() - 1);
    }
}
```

### 4.4 N-Queens
Place N queens on an N×N board so none attack each other. Place one queen per row; for each row, try each column, **prune** if it conflicts with existing queens, recurse, backtrack.
```
For each row:
    for each column:
        if placing a queen here is safe (no conflict in column/diagonals):
            place it
            recurse to the next row
            remove it (backtrack)
```
> N-Queens is the canonical backtracking problem: the **safety check prunes** invalid placements early, dramatically cutting the search space from N^N.

### 4.5 Sudoku Solver
For each empty cell, try digits 1–9; if valid (no conflict in row/column/box), place it and recurse; if the recursion fails, backtrack and try the next digit.
```
solve(board):
    find an empty cell; if none -> solved!
    for digit 1..9:
        if valid(cell, digit):
            place digit
            if solve(board): return true    # success propagates up
            remove digit                     # backtrack
    return false                             # no digit worked -> backtrack further
```

### 4.6 Common backtracking problems summary
| Problem | What backtracking explores |
|---------|----------------------------|
| Subsets | In/out of each element (2ⁿ) |
| Permutations | All orderings (n!) |
| Combinations | k-of-n selections |
| N-Queens | Valid queen placements |
| Sudoku | Valid digit fillings |
| Word search / maze | Valid paths |
| Combination sum | Sets summing to a target |

---

## 5. Pruning (Making Backtracking Efficient)

Backtracking's worst case is exponential, but **pruning** — abandoning a branch the moment it can't yield a valid solution — often makes it fast in practice.
| Pruning technique | Example |
|-------------------|---------|
| **Constraint check** | Skip a column that conflicts (N-Queens) |
| **`used[]` / visited** | Skip already-chosen elements (permutations) |
| **Early termination** | Stop when sum exceeds target (combination sum) |
| **Ordering** | Try most-constrained choices first |
> ⚠️ Without pruning, backtracking degenerates to brute-force exponential search. Good pruning is the difference between "solves instantly" and "runs forever."

---

## 6. Recursion vs Iteration (recap Phase 1.2/2.1)

| | Recursion | Iteration |
|---|-----------|-----------|
| Readability | Elegant for trees/backtracking | Better for simple loops |
| Space | Call stack (O(depth)) | O(1) usually |
| Risk | StackOverflow (deep) | None |
| Convert? | Any recursion → iteration (with an explicit stack) | — |
> Backtracking is naturally recursive (it mirrors the decision tree). For very deep simple recursion, prefer iteration; for exploratory/tree-shaped problems, recursion is clearer.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Missing/unreachable base case | Ensure progress toward the base case |
| Forgetting to undo the choice (no backtrack) | Always un-choose after recursing |
| Adding a mutable reference instead of a copy | `new ArrayList<>(current)` when recording results |
| No pruning → exponential blowup | Prune invalid branches early |
| Deep recursion → StackOverflow | Convert to iteration / increase `-Xss` |
| Tree recursion recomputing subproblems | Memoize → DP (next note) |
| Duplicates in combinations/subsets | Use a `start` index; skip duplicates |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Tree/graph traversal** (Phase 2.2): DFS *is* recursion; backtracking is DFS with undo.
- **Parsing** (JSON, expressions, query languages): recursive descent parsers (Phase 5.3).
- **Dependency resolution** (Phase 3): exploring valid configurations.
- **Constraint solving / scheduling / configuration**: backtracking explores valid assignments.
- **Interview prep** (Phase 2.4): subsets/permutations/N-Queens/Sudoku are extremely common.
- **DP foundation** (next note): backtracking + memoization → dynamic programming.
- **Recursion limits**: production code must guard against deep recursion (StackOverflow, Phase 1.11).

---

## 9. Quick Self-Check Questions

1. What two parts must every recursion have, and what's the "trust the recursion" mindset?
2. Why can naive tree recursion be exponential, and what's the fix?
3. What is backtracking, and what are its three steps?
4. Write the universal backtracking template.
5. How do you generate all subsets? All permutations?
6. How does N-Queens use pruning to cut the search space?
7. Why is pruning essential to backtracking's practicality?
8. Why must you copy `current` when recording a result, not add the reference?

---

## 10. Key Terms Glossary

- **Recursion:** a method calling itself (base + recursive case).
- **Call stack / depth:** recursion's space cost (Phase 2.1).
- **Tree recursion:** multiple self-calls (can be exponential).
- **Backtracking:** incremental candidate building with undo (DFS over solutions).
- **Choose / explore / un-choose:** backtracking's three steps.
- **Pruning:** abandoning branches that can't succeed.
- **Decision tree:** the space of choices backtracking explores.
- **Subsets / permutations / combinations:** classic enumeration problems.
- **Constraint satisfaction:** problems like N-Queens/Sudoku.

---

*Previous topic: **Two Pointers & Sliding Window**.*
*Next topic: **Dynamic Programming**.*
