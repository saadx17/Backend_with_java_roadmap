# Greedy Algorithms & Divide and Conquer

> **Phase 2 — Data Structures & Algorithms → 2.3 Algorithms**
> Goal: Master two major algorithm paradigms — greedy algorithms (make the locally optimal choice) and divide and conquer (split, solve, combine) — and know when each applies.

---

## 0. The Big Picture

These are two fundamental **algorithm design paradigms** (alongside DP and backtracking):
- **Greedy** — make the **locally optimal** choice at each step, hoping it leads to a global optimum. Simple and fast — *when it works*.
- **Divide and conquer** — **split** a problem into smaller independent subproblems, **solve** each recursively, **combine** the results.

```
Greedy:           pick the best-looking option NOW, never reconsider
Divide & conquer: break -> solve parts -> merge
```

> Both are core paradigms tested in interviews and used throughout systems. The key skill: knowing **when greedy is correct** (it isn't always!) and recognizing divide-and-conquer structure.

---

## 1. Greedy Algorithms

A **greedy algorithm** builds a solution step by step, always choosing the option that looks best **right now** (locally optimal), without reconsidering past choices.

### 1.1 When greedy works (the greedy-choice property)
Greedy gives the **correct global optimum only if** the problem has:
| Property | Meaning |
|----------|---------|
| **Greedy-choice property** | A locally optimal choice leads to a globally optimal solution |
| **Optimal substructure** | The optimal solution contains optimal subsolutions (also needed for DP) |
> ⚠️ **Greedy doesn't always work!** It's fast and simple, but you must **prove** (or trust a known result) that the greedy choice is safe. When it fails, you usually need DP (previous note) or backtracking. This is the #1 greedy pitfall.

### 1.2 Example where greedy WORKS: Activity Selection
Select the maximum number of non-overlapping activities. **Greedy choice:** always pick the activity that **ends earliest** (leaves the most room for others).
```java
int maxActivities(int[][] activities) {        // activities[i] = {start, end}
    Arrays.sort(activities, Comparator.comparingInt(a -> a[1]));  // sort by END time
    int count = 0, lastEnd = Integer.MIN_VALUE;
    for (int[] a : activities) {
        if (a[0] >= lastEnd) {                  // doesn't overlap the last chosen
            count++;
            lastEnd = a[1];                     // greedily take it
        }
    }
    return count;
}
```
> Sorting by earliest finish time is *provably* optimal here — the greedy choice is safe.

### 1.3 Example where greedy WORKS: Huffman Coding
Builds optimal prefix-free codes for **data compression**. **Greedy choice:** repeatedly merge the **two least-frequent** symbols (using a min-heap, Phase 2.2) into a subtree.
```
1. Put all symbols (by frequency) in a min-heap.
2. Repeatedly: extract the two smallest, merge into a node with their combined frequency, re-insert.
3. The resulting tree gives variable-length codes (frequent symbols -> short codes).
```
> Huffman coding underlies compression (ZIP, JPEG, etc.) and is a classic greedy success — using a priority queue (Phase 2.2 heaps).

### 1.4 Example where greedy FAILS: Coin Change (general)
For arbitrary coin systems, greedy (always take the largest coin ≤ remaining) can be **wrong**:
```
Coins {1, 3, 4}, make 6:
  Greedy: 4 + 1 + 1 = 3 coins
  Optimal: 3 + 3 = 2 coins   <- greedy FAILED!
(This is why coin change needs DP — previous note.)
```
> ⚠️ Greedy works for *some* coin systems (like standard currency) but not all → you must verify. When greedy fails, use **DP**.

### 1.5 Other greedy successes
| Problem | Greedy choice |
|---------|---------------|
| **Dijkstra's shortest path** (Phase 2.2) | Always expand the closest unvisited node |
| **Kruskal's / Prim's MST** (Phase 2.2) | Always add the cheapest valid edge |
| **Interval scheduling** | Earliest finish time |
| **Fractional knapsack** | Highest value/weight ratio first |
> Note: **0/1 knapsack** needs DP (can't take fractions), but **fractional** knapsack is greedy — a subtle but important distinction.

### 1.6 Greedy approach summary
```
1. Sort / order by some greedy criterion (often the key step).
2. Iterate, making the locally best choice each time.
3. (Critically) ensure the greedy choice is provably optimal for THIS problem.
```

---

## 2. Divide and Conquer

**Divide and conquer** solves a problem by:
1. **Divide** — break it into smaller, **independent** subproblems.
2. **Conquer** — solve each subproblem **recursively**.
3. **Combine** — merge the subproblem results into the final answer.
```
            problem
           /        \
      subproblem  subproblem      (divide)
       /    \       /    \
     ...recursively solve...      (conquer)
           \        /
            combine                (combine)
```

### 2.1 Classic examples (mostly seen already)
| Algorithm | Divide | Combine |
|-----------|--------|---------|
| **Merge sort** (Phase 2.3) | Split in half | Merge sorted halves |
| **Quick sort** (Phase 2.3) | Partition around a pivot | (already in place) |
| **Binary search** (Phase 2.3) | Halve the range | (pick one half) |
| **Fast exponentiation** | `x^n = (x^(n/2))²` | Square the result |

### 2.2 Example: fast power (x^n in O(log n))
```java
double power(double x, int n) {
    if (n == 0) return 1;
    double half = power(x, n / 2);             // DIVIDE: solve half the exponent
    if (n % 2 == 0) return half * half;        // COMBINE: square it
    else return half * half * x;               // odd: one extra multiply
}
// O(log n) instead of O(n) naive multiplication
```

### 2.3 Example: maximum subarray (divide & conquer view)
The max subarray is either entirely in the left half, entirely in the right half, or **crosses the middle** — solve each and combine. (Kadane's algorithm solves it in O(n) with DP, but the D&C view is instructive.)

### 2.4 The Master Theorem (awareness)
Divide-and-conquer recurrences often have the form `T(n) = a·T(n/b) + O(n^d)`. The **Master Theorem** gives the complexity:
```
Merge sort: T(n) = 2T(n/2) + O(n)   -> O(n log n)
Binary search: T(n) = T(n/2) + O(1) -> O(log n)
```
> You don't need to memorize the full theorem, but recognize that splitting into `a` subproblems of size `n/b` plus `O(n^d)` combine work determines the complexity. (Merge sort: 2 halves + O(n) merge = O(n log n).)

---

## 3. Divide & Conquer vs Dynamic Programming

Both break problems into subproblems — the difference is **subproblem overlap**:
| | Divide & Conquer | Dynamic Programming |
|---|------------------|---------------------|
| Subproblems | **Independent** (no overlap) | **Overlapping** (reused) |
| Caching | Not needed | Essential (memoization) |
| Examples | Merge sort, binary search | Fibonacci, LCS, knapsack |
> If subproblems **overlap** (the same one is solved repeatedly), it's a **DP** problem (cache them). If they're **independent**, it's plain divide and conquer (no caching needed). This is *the* distinguishing question.

---

## 4. The Four Paradigms Compared
| Paradigm | Idea | Optimal? | When |
|----------|------|----------|------|
| **Brute force / Backtracking** | Try all (with pruning) | Yes | Enumerate / constraints |
| **Divide & Conquer** | Split, solve, combine | Yes | Independent subproblems |
| **Dynamic Programming** | Cache overlapping subproblems | Yes | Overlapping subproblems |
| **Greedy** | Locally best choice | Only if greedy property holds | Provably-safe local choices |
> **Decision flow:** Independent subproblems → D&C. Overlapping subproblems → DP. A safe local choice each step → Greedy. Otherwise → backtracking (exhaustive).

---

## 5. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Assuming greedy always works | Prove the greedy-choice property; else use DP |
| Greedy coin change for arbitrary coins | Use DP (greedy fails for some systems) |
| Confusing 0/1 vs fractional knapsack | 0/1 = DP; fractional = greedy |
| Caching independent subproblems (D&C) | No caching needed if no overlap |
| Not caching overlapping subproblems | That's DP — memoize |
| Deep divide-and-conquer recursion | Watch stack depth (usually O(log n), fine) |
| Forgetting the "combine" step in D&C | Combining subresults is half the work |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- **Greedy:** Dijkstra/MST (routing, network design — Phase 2.2/12), scheduling, **rate limiting** (token bucket is greedy), load balancing heuristics, Huffman compression.
- **Divide & conquer:** sorting (Phase 2.3), parallel processing (**Fork/Join**, Phase 1.10 — split work, combine), **MapReduce** / distributed data processing (Phase 16), binary search (Phase 2.3).
- **Paradigm recognition:** choosing the right approach is a core engineering & interview skill (Phase 2.4).
- **Fast exponentiation** appears in cryptography (modular exponentiation — Phase 15) and hashing.
- **Combine step** mirrors the **combiner** in parallel streams/Fork-Join (Phase 1.8/1.10).

---

## 7. Quick Self-Check Questions

1. What is a greedy algorithm, and when is it guaranteed correct?
2. Give an example where greedy works and one where it fails.
3. What's the difference between 0/1 and fractional knapsack regarding greedy vs DP?
4. What are the three steps of divide and conquer?
5. Give three divide-and-conquer algorithms you already know.
6. How do you compute x^n in O(log n)?
7. What distinguishes divide & conquer from dynamic programming?
8. How do you decide between greedy, DP, D&C, and backtracking?

---

## 8. Key Terms Glossary

- **Greedy algorithm:** locally optimal choices toward a global optimum.
- **Greedy-choice property:** local optimum → global optimum (must hold).
- **Activity selection / Huffman coding:** classic greedy successes.
- **Fractional vs 0/1 knapsack:** greedy vs DP.
- **Divide and conquer:** divide → conquer (recurse) → combine.
- **Independent subproblems:** non-overlapping (no caching needed).
- **Master Theorem:** complexity of divide-and-conquer recurrences.
- **Fast exponentiation:** x^n via repeated squaring, O(log n).
- **Combine step:** merging subproblem results.

---

*Previous topic: **Dynamic Programming**.*
*Next topic: **Bit Manipulation & String Algorithms**.*
