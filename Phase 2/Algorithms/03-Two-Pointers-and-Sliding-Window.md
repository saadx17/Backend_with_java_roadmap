# Two Pointers & Sliding Window

> **Phase 2 — Data Structures & Algorithms → 2.3 Algorithms**
> Goal: Master two of the most common interview/algorithm patterns — the two-pointer technique (same & opposite direction) and the sliding window (fixed & variable size).

---

## 0. The Big Picture

**Two pointers** and **sliding window** are *techniques* (not data structures) that turn many **O(n²) brute-force** solutions into **O(n)** by cleverly moving one or two indices through an array/string instead of using nested loops.

```
Brute force: nested loops over pairs/subarrays -> O(n²)
Two pointers / sliding window: one pass with smart pointer movement -> O(n)
```

> These are among the **most frequently tested** interview patterns (Phase 2.4) and appear constantly in array/string processing. Recognizing when to apply them is the key skill.

---

## 1. The Two-Pointer Technique

Use **two indices** that move through the data, replacing a nested loop. Two main variants based on pointer movement.

### 1.1 Opposite-direction (converging) pointers
Two pointers start at the **ends** and move **toward each other**. Works on **sorted** arrays or for symmetry checks.

**Example: two-sum on a sorted array** (O(n) instead of O(n²)):
```java
int[] twoSumSorted(int[] arr, int target) {     // arr is SORTED
    int lo = 0, hi = arr.length - 1;
    while (lo < hi) {
        int sum = arr[lo] + arr[hi];
        if (sum == target) return new int[]{lo, hi};
        else if (sum < target) lo++;             // need bigger -> move left pointer right
        else hi--;                                // need smaller -> move right pointer left
    }
    return new int[]{-1, -1};
}
```
> The insight: in a *sorted* array, if the sum is too small, the only way to increase it is to move `lo` right; if too big, move `hi` left. Each step eliminates one element → O(n). (Compare: the HashMap two-sum from Phase 2.1 is O(n) time + O(n) space; this is O(n) time + O(1) space but needs sorting.)

**Example: valid palindrome** (compare ends moving inward):
```java
boolean isPalindrome(String s) {
    int lo = 0, hi = s.length() - 1;
    while (lo < hi)
        if (s.charAt(lo++) != s.charAt(hi--)) return false;
    return true;
}
```

**Example: reverse in place** (recall Phase 2.2 arrays):
```java
void reverse(int[] arr) {
    int lo = 0, hi = arr.length - 1;
    while (lo < hi) { swap(arr, lo++, hi--); }   // O(1) space
}
```

### 1.2 Same-direction (fast & slow) pointers
Two pointers move in the **same direction** at **different speeds** or for **different purposes** (often one "reads," one "writes").

**Example: remove duplicates from a sorted array in place** (O(1) space):
```java
int removeDuplicates(int[] arr) {
    if (arr.length == 0) return 0;
    int slow = 0;                                 // position to write the next unique value
    for (int fast = 1; fast < arr.length; fast++) {
        if (arr[fast] != arr[slow]) {
            arr[++slow] = arr[fast];              // write the new unique value
        }
    }
    return slow + 1;                              // length of the unique prefix
}
```
> The **fast/slow** pattern also powers **linked-list cycle detection** and **find-middle** (Floyd's, recall Phase 2.2 linked lists) — the same idea applied to lists.

### 1.3 When to use two pointers
| Signal | Pattern |
|--------|---------|
| Sorted array + find a pair/triplet | Opposite-direction |
| Palindrome / symmetry | Opposite-direction |
| In-place array modification (dedupe, partition) | Same-direction (fast/slow) |
| Linked-list cycle/middle | Fast/slow pointers |
| Merging two sorted sequences | Two pointers (one per sequence) |

---

## 2. The Sliding Window Technique

A **sliding window** maintains a contiguous **subarray/substring** (a "window") that **expands and contracts** as it moves through the data — ideal for problems about **contiguous** subarrays/substrings.

```
[a b c d e f]   window expands by moving 'right', contracts by moving 'left'
 ^   ^
left right
```

### 2.1 Fixed-size window
The window stays a constant size `k`; slide it across, updating the result incrementally instead of recomputing.

**Example: maximum sum of any subarray of size k** (O(n) instead of O(n×k)):
```java
int maxSumSubarray(int[] arr, int k) {
    int windowSum = 0;
    for (int i = 0; i < k; i++) windowSum += arr[i];   // first window
    int maxSum = windowSum;
    for (int i = k; i < arr.length; i++) {
        windowSum += arr[i] - arr[i - k];              // add new, remove old (slide!)
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}
```
> The key efficiency: when the window slides, **add the entering element and subtract the leaving one** — O(1) per slide instead of re-summing O(k). Total O(n). (Recompute-from-scratch would be O(n×k).)

### 2.2 Variable-size window
The window **grows and shrinks** based on a condition — typically "expand right until invalid, then shrink left until valid again."

**Example: longest substring without repeating characters** (O(n)):
```java
int longestUniqueSubstring(String s) {
    Set<Character> window = new HashSet<>();
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        while (window.contains(c)) {                   // shrink until valid
            window.remove(s.charAt(left++));
        }
        window.add(c);                                  // expand
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
```
> The pattern: **expand `right`** to include new elements; when the window violates the constraint, **shrink `left`** until it's valid again. Each element enters and leaves the window at most once → O(n) overall, even with the inner `while` (amortized — Phase 2.1).

### 2.3 The general variable-window template
```
left = 0
for right in 0..n-1:
    add arr[right] to the window
    while (window is invalid):
        remove arr[left] from the window
        left++
    update the answer using the current valid window [left..right]
```

### 2.4 When to use a sliding window
| Signal | Likely sliding window |
|--------|----------------------|
| "Contiguous subarray/substring" | ✅ |
| "Longest/shortest/maximum subarray with condition X" | ✅ |
| "Subarray sum equals / at most K" | ✅ |
| Fixed-size window (average, max sum of k) | ✅ fixed window |
> ⚠️ Sliding window works for **contiguous** ranges. For **subsequences** (non-contiguous) or unsorted-pair problems, you usually need hashing or DP instead.

---

## 3. Two Pointers vs Sliding Window

They're related (a window *is* two pointers), but:
| | Two pointers (general) | Sliding window |
|---|------------------------|----------------|
| Pointers | Often move independently / oppositely | Both move forward; maintain a window |
| Problem type | Pairs, palindromes, in-place edits, merging | Contiguous subarray/substring with a condition |
| Window concept | Not necessarily | Always a contiguous range |
> Sliding window is a *specialized* same-direction two-pointer pattern where the pointers bound a contiguous window.

---

## 4. Related Pattern: Monotonic Deque (sliding window maximum)
For "maximum/minimum in every window of size k," a plain sliding window is O(n×k); a **monotonic deque** (recall Phase 2.2 deques) keeps candidates in decreasing order for O(n):
```java
int[] maxSlidingWindow(int[] arr, int k) {
    Deque<Integer> dq = new ArrayDeque<>();   // stores indices, values decreasing
    int[] result = new int[arr.length - k + 1];
    for (int i = 0; i < arr.length; i++) {
        if (!dq.isEmpty() && dq.peekFirst() <= i - k) dq.pollFirst();   // drop out-of-window
        while (!dq.isEmpty() && arr[dq.peekLast()] < arr[i]) dq.pollLast();  // keep decreasing
        dq.offerLast(i);
        if (i >= k - 1) result[i - k + 1] = arr[dq.peekFirst()];        // front = max
    }
    return result;
}
```
> A nice example of combining a *technique* (sliding window) with a *structure* (deque) for an optimal solution. (Awareness — recognize it exists.)

---

## 5. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Two pointers (opposite) on unsorted data | Sort first, or use a different approach |
| Recomputing the window from scratch | Update incrementally (add new, remove old) |
| Sliding window for non-contiguous subsequences | Use DP/hashing instead |
| Off-by-one in window size (`right - left + 1`) | Carefully track inclusive bounds |
| Forgetting to shrink the window when invalid | The `while` shrink step is essential |
| Not updating the answer at the right time | Update after the window is valid |
| Thinking the inner `while` makes it O(n²) | Each element enters/leaves once → amortized O(n) |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- **Stream/data processing** (Project 2): windowed aggregations, deduplication, running computations.
- **Rate limiting** (Phase 12.3): the **sliding-window** algorithm for request counting is literally this pattern over time.
- **Time-series / metrics** (Phase 9): moving averages, max-in-window.
- **Log processing**: scanning contiguous ranges efficiently.
- **Interview prep** (Phase 2.4): two pointers and sliding window are among the highest-frequency patterns.
- **In-place array ops** (fast/slow) avoid extra allocation (memory efficiency, Phase 13).

---

## 7. Quick Self-Check Questions

1. How do opposite-direction two pointers solve sorted two-sum in O(n) / O(1) space?
2. What's the fast/slow (same-direction) pattern good for? Give two examples.
3. How does a fixed-size sliding window achieve O(n) instead of O(n×k)?
4. What's the general template for a variable-size sliding window?
5. Why is a variable sliding window O(n) despite the inner `while` loop?
6. When does sliding window NOT apply?
7. What's the difference between two pointers and a sliding window?
8. What's a monotonic deque used for?

---

## 8. Key Terms Glossary

- **Two-pointer technique:** using two indices to replace a nested loop.
- **Opposite-direction (converging) pointers:** start at ends, move inward (sorted/symmetry).
- **Same-direction (fast/slow) pointers:** move forward at different speeds/roles.
- **Sliding window:** a contiguous range that expands/contracts across the data.
- **Fixed / variable window:** constant size / condition-driven size.
- **Window invariant:** the condition the window must satisfy.
- **Monotonic deque:** a deque kept in sorted order for window min/max.
- **Amortized O(n):** each element processed a constant number of times overall.

---

*Previous topic: **Searching**.*
*Next topic: **Recursion & Backtracking**.*
