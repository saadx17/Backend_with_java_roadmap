# Searching Algorithms

> **Phase 2 — Data Structures & Algorithms → 2.3 Algorithms**
> Goal: Master searching — linear search, binary search (iterative & recursive), and the crucial binary search variations (first/last occurrence, floor/ceiling, search in rotated arrays).

---

## 0. The Big Picture

**Searching** finds an element (or a position) in a collection. The two fundamental approaches: **linear search** (check each element, O(n)) and **binary search** (halve the search space each step, O(log n) — but requires **sorted** data). Binary search is one of the most important and most-tested algorithms.

```
Linear:  scan one by one          -> O(n),     works on anything
Binary:  halve the range each step -> O(log n), requires SORTED data
```

> Binary search underlies database index lookups (B-trees, Phase 2.2), `Collections.binarySearch`, `git bisect` (Phase 0.4), and countless interview problems. Its **variations** (finding boundaries) are where it gets powerful — and tricky.

---

## 1. Linear Search

Check each element until you find the target. Works on **any** collection (sorted or not).
```java
int linearSearch(int[] arr, int target) {
    for (int i = 0; i < arr.length; i++)
        if (arr[i] == target) return i;   // found
    return -1;                            // not found
}
```
- Time: **O(n)** (O(1) best, O(n) worst — Phase 2.1). Space: O(1).
- Use when: data is **unsorted**, the collection is **small**, or you can't index (e.g., a linked list, a stream).
> `list.contains()`, `list.indexOf()` are linear search — O(n). Inside a loop, that's O(n²) (recall the hidden-cost trap, Phase 2.1) → use a `HashSet` for O(1) membership instead.

---

## 2. Binary Search (the essential algorithm)

Binary search finds a target in a **sorted** array by repeatedly **halving** the search range — O(log n).

### 2.1 How it works
```
Compare target to the MIDDLE element:
  equal      -> found
  target <   -> search the LEFT half
  target >   -> search the RIGHT half
Repeat, halving the range each time.
```

### 2.2 Iterative implementation (preferred — O(1) space)
```java
int binarySearch(int[] arr, int target) {
    int lo = 0, hi = arr.length - 1;
    while (lo <= hi) {                        // note: <=
        int mid = lo + (hi - lo) / 2;        // avoids integer overflow (NOT (lo+hi)/2!)
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) lo = mid + 1;   // search right
        else hi = mid - 1;                            // search left
    }
    return -1;                               // not found
}
```

### 2.3 Recursive implementation
```java
int binarySearch(int[] arr, int target, int lo, int hi) {
    if (lo > hi) return -1;                   // base case: not found
    int mid = lo + (hi - lo) / 2;
    if (arr[mid] == target) return mid;
    else if (arr[mid] < target) return binarySearch(arr, target, mid + 1, hi);
    else return binarySearch(arr, target, lo, mid - 1);
}
```
- Recursive uses O(log n) **stack space** (Phase 2.1); iterative uses O(1). Prefer **iterative**.

### 2.4 The two classic bugs
> ⚠️ **Bug 1 — overflow:** use `mid = lo + (hi - lo) / 2`, **not** `(lo + hi) / 2` — the latter can overflow `int` for large indices (recall Phase 1.2 integer overflow). This was a real bug in the JDK's binary search for years.
> ⚠️ **Bug 2 — infinite loop / off-by-one:** get the loop condition (`<=` vs `<`) and the updates (`mid + 1`, `mid - 1`) right, or you loop forever or miss elements. Binary search is famously easy to get subtly wrong.

### 2.5 Complexity
- Time: **O(log n)**. Space: O(1) iterative, O(log n) recursive.
- **Requires sorted data** — that's the trade-off (sort once O(n log n), then search many times O(log n)).

---

## 3. Binary Search in Java (use the library)
```java
int idx = Arrays.binarySearch(sortedArray, target);     // array must be sorted!
int idx2 = Collections.binarySearch(sortedList, target);
// Returns the index if found; otherwise -(insertion point) - 1 (negative!)
```
> ⚠️ Java's `binarySearch` returns a **negative** value `-(insertionPoint) - 1` when not found — not just `-1`. You can recover the insertion point with `-(result) - 1`. Also, the array/list **must already be sorted** or results are undefined.

---

## 4. Binary Search Variations (Where It Gets Powerful)

The real power of binary search is finding **boundaries** in sorted data with duplicates — not just "does it exist?" These are common interview problems.

### 4.1 First occurrence (leftmost)
Find the **first** index of a target that may appear multiple times:
```java
int findFirst(int[] arr, int target) {
    int lo = 0, hi = arr.length - 1, result = -1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) {
            result = mid;                    // record, but KEEP searching left
            hi = mid - 1;
        } else if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return result;
}
```

### 4.2 Last occurrence (rightmost)
```java
int findLast(int[] arr, int target) {
    int lo = 0, hi = arr.length - 1, result = -1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) {
            result = mid;                    // record, but KEEP searching right
            lo = mid + 1;
        } else if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return result;
}
```
> The trick: on a match, **don't stop** — record it and keep searching in the direction of the boundary. (Count occurrences = last − first + 1.)

### 4.3 Floor and Ceiling
- **Floor:** largest element **≤** target. **Ceiling:** smallest element **≥** target.
```java
int floor(int[] arr, int target) {           // largest value <= target
    int lo = 0, hi = arr.length - 1, result = -1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] <= target) { result = arr[mid]; lo = mid + 1; }   // candidate; go right
        else hi = mid - 1;
    }
    return result;
}
```
> (Recall `TreeSet`/`TreeMap` provide `floor`/`ceiling` directly — Phase 1.6 — backed by this idea on a Red-Black tree.)

### 4.4 Search in a rotated sorted array
A sorted array rotated at some pivot (e.g., `[4,5,6,7,0,1,2]`). Still O(log n) — at each step, one half is guaranteed sorted; decide which half to search:
```java
int searchRotated(int[] arr, int target) {
    int lo = 0, hi = arr.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) return mid;
        if (arr[lo] <= arr[mid]) {            // LEFT half is sorted
            if (arr[lo] <= target && target < arr[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else {                               // RIGHT half is sorted
            if (arr[mid] < target && target <= arr[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    return -1;
}
```

### 4.5 Binary search on the answer (a key pattern)
Binary search isn't only for arrays — you can binary-search over a **range of possible answers** when there's a monotonic condition (e.g., "minimum capacity to ship in D days," "smallest divisor"). If `feasible(x)` is monotonic (false…false, true…true), binary-search the smallest `x` where it's true.
```
lo, hi = bounds of possible answers
while lo < hi:
    mid = lo + (hi - lo) / 2
    if feasible(mid): hi = mid      // try smaller
    else: lo = mid + 1
return lo
```
> This "binary search on the answer" pattern is extremely common in interviews and optimization problems — recognize when the answer space is monotonic.

---

## 5. Searching Other Structures (recap)

| Structure | Search | Complexity |
|-----------|--------|------------|
| Unsorted array/list | Linear | O(n) |
| Sorted array | Binary search | O(log n) |
| HashMap/HashSet | Hash lookup | O(1) average (Phase 2.2) |
| TreeMap/TreeSet (BST) | Tree search | O(log n) (Phase 2.2) |
| Trie | Prefix walk | O(L) (Phase 2.2) |
| Graph | BFS/DFS | O(V+E) (Phase 2.2) |
> **Choose the structure for your search needs:** O(1) exact lookups → HashMap; sorted/range → TreeMap or sorted array + binary search; prefix → Trie.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Binary search on unsorted data | Sort first, or use linear/hash |
| `mid = (lo + hi) / 2` overflow | Use `lo + (hi - lo) / 2` |
| Off-by-one / infinite loop | Carefully set `<=`, `mid±1` |
| Expecting `Arrays.binarySearch` to return -1 when absent | It returns `-(insertionPoint)-1` |
| Stopping at first match when finding boundaries | Keep searching toward the boundary |
| `list.contains` in a loop (O(n²)) | Use a HashSet (Phase 2.1) |
| Recursive binary search on huge ranges | Use iterative (O(1) space) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Database indexes** (B+ trees, Phase 2.2/4.4) turn O(n) table scans into O(log n) lookups — binary search at scale on disk.
- **`Arrays.binarySearch`/`Collections.binarySearch`** and `TreeMap` floor/ceiling for sorted lookups (Phase 1.6).
- **`git bisect`** (Phase 0.4) is binary search over commits.
- **"Binary search on the answer"** appears in capacity planning, rate limiting, and optimization (Phase 13, 16.8).
- **Choosing the right structure** (HashMap vs TreeMap vs sorted array) is a recurring design decision (Phase 1.6).
- **Pagination/range queries** rely on sorted access + binary-search-like boundaries.

---

## 8. Quick Self-Check Questions

1. When use linear search vs binary search?
2. Why does binary search require sorted data, and what's its complexity?
3. What's the correct way to compute `mid`, and why?
4. What two classic bugs plague binary search?
5. What does Java's `binarySearch` return when the element isn't found?
6. How do you find the first vs last occurrence of a duplicated target?
7. How do you binary-search a rotated sorted array in O(log n)?
8. What is "binary search on the answer," and when does it apply?

---

## 9. Key Terms Glossary

- **Linear search:** check each element, O(n), works on anything.
- **Binary search:** halve a sorted range, O(log n).
- **Sorted precondition:** binary search requires sorted data.
- **`mid = lo + (hi - lo) / 2`:** overflow-safe midpoint.
- **First/last occurrence:** leftmost/rightmost index of a duplicated value.
- **Floor / ceiling:** largest ≤ / smallest ≥ target.
- **Rotated sorted array:** a sorted array rotated at a pivot.
- **Binary search on the answer:** searching a monotonic answer space.
- **Insertion point:** where a missing element would go (Java's negative return).

---

*Previous topic: **Sorting Algorithms**.*
*Next topic: **Two Pointers & Sliding Window**.*
