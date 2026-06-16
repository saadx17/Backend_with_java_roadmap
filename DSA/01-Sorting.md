# Sorting Algorithms

> **Phase 2 — Data Structures & Algorithms → 2.3 Algorithms**
> Goal: Master the major sorting algorithms — simple O(n²) sorts, the efficient O(n log n) sorts (merge, quick, heap), non-comparison sorts, and what Java actually uses internally.

---

## 0. The Big Picture

**Sorting** arranges elements in order. It's one of the most-studied problems because (a) it's everywhere, and (b) it teaches key algorithmic ideas (divide-and-conquer, partitioning, heaps). Understanding sorting deeply makes you better at *all* algorithms.

```
Unsorted: [5, 2, 8, 1, 9]  -->  Sorted: [1, 2, 5, 8, 9]
```

> You'll rarely hand-write a sort in production (`Arrays.sort`/`Collections.sort` are excellent — Phase 1.6), but understanding them is essential for interviews and for knowing *what Java does under the hood* (§6).

---

## 1. Key Properties of Sorting Algorithms

When comparing sorts, three properties matter:
| Property | Meaning |
|----------|---------|
| **Time complexity** | How runtime scales (best/average/worst — Phase 2.1) |
| **Space complexity** | Extra memory (in-place = O(1) auxiliary) |
| **Stability** | Equal elements keep their **relative order** |

### 1.1 Why stability matters
A **stable** sort preserves the original order of equal elements:
```
Sort people by age (stable): [(Bob,30), (Amy,25), (Cal,30)]
  -> [(Amy,25), (Bob,30), (Cal,30)]   Bob still before Cal (both 30) ✓
```
> Stability matters when sorting by **multiple keys** in passes (e.g., sort by name, then stably by age → sorted by age, ties broken by name). Java's `Collections.sort`/`Arrays.sort` (objects) are **stable**.

---

## 2. Simple Sorts — O(n²)

These are easy to understand but slow for large n. Good for tiny inputs or teaching.

### 2.1 Bubble Sort
Repeatedly swap adjacent out-of-order elements; the largest "bubbles" to the end each pass.
```java
void bubbleSort(int[] arr) {
    for (int i = 0; i < arr.length - 1; i++) {
        boolean swapped = false;
        for (int j = 0; j < arr.length - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {        // swap adjacent if out of order
                int t = arr[j]; arr[j] = arr[j + 1]; arr[j + 1] = t;
                swapped = true;
            }
        }
        if (!swapped) break;                  // already sorted -> O(n) best case
    }
}
```
- Time: O(n²) avg/worst, O(n) best (already sorted). Space: O(1). **Stable.**
- ⚠️ Essentially never used in practice — slowest of the simple sorts. Educational only.

### 2.2 Selection Sort
Repeatedly find the minimum of the unsorted part and place it at the front.
```java
void selectionSort(int[] arr) {
    for (int i = 0; i < arr.length - 1; i++) {
        int min = i;
        for (int j = i + 1; j < arr.length; j++)
            if (arr[j] < arr[min]) min = j;   // find the minimum
        int t = arr[min]; arr[min] = arr[i]; arr[i] = t;   // swap into place
    }
}
```
- Time: O(n²) all cases. Space: O(1). **Not stable.**
- Does the **fewest swaps** (O(n)) — useful if swaps are expensive.

### 2.3 Insertion Sort
Build the sorted portion one element at a time, inserting each into its correct position (like sorting a hand of cards).
```java
void insertionSort(int[] arr) {
    for (int i = 1; i < arr.length; i++) {
        int key = arr[i], j = i - 1;
        while (j >= 0 && arr[j] > key) {      // shift larger elements right
            arr[j + 1] = arr[j];
            j--;
        }
        arr[j + 1] = key;                     // insert key
    }
}
```
- Time: O(n²) avg/worst, **O(n) best** (nearly sorted). Space: O(1). **Stable.**
- ⭐ **Excellent for small or nearly-sorted arrays** — which is *why* Java uses it for small subarrays inside its big sorts (§6).

### 2.4 Simple sorts comparison
| Sort | Best | Avg | Worst | Space | Stable | Note |
|------|------|-----|-------|-------|--------|------|
| Bubble | O(n) | O(n²) | O(n²) | O(1) | ✅ | Avoid |
| Selection | O(n²) | O(n²) | O(n²) | O(1) | ❌ | Fewest swaps |
| Insertion | **O(n)** | O(n²) | O(n²) | O(1) | ✅ | Great for small/nearly-sorted |

---

## 3. Efficient Sorts — O(n log n)

The workhorses for large data. O(n log n) is the **best possible for comparison-based sorting**.

### 3.1 Merge Sort (divide & conquer, stable)
**Divide** the array in half, **sort each half recursively**, then **merge** the two sorted halves.
```java
void mergeSort(int[] arr, int lo, int hi) {
    if (lo >= hi) return;                      // base case: 1 element
    int mid = lo + (hi - lo) / 2;
    mergeSort(arr, lo, mid);                   // sort left half
    mergeSort(arr, mid + 1, hi);              // sort right half
    merge(arr, lo, mid, hi);                  // merge them
}
void merge(int[] arr, int lo, int mid, int hi) {
    int[] temp = new int[hi - lo + 1];
    int i = lo, j = mid + 1, k = 0;
    while (i <= mid && j <= hi)
        temp[k++] = (arr[i] <= arr[j]) ? arr[i++] : arr[j++];   // <= keeps it STABLE
    while (i <= mid) temp[k++] = arr[i++];
    while (j <= hi)  temp[k++] = arr[j++];
    System.arraycopy(temp, 0, arr, lo, temp.length);
}
```
- Time: **O(n log n)** all cases (guaranteed!). Space: **O(n)** (the temp array). **Stable.**
- ✅ Predictable O(n log n) worst case + stable. ⚠️ Needs O(n) extra space.
- **Used by Java for sorting objects** (`Arrays.sort(Object[])`, `Collections.sort`) — TimSort, an optimized merge sort (§6).

### 3.2 Quick Sort (divide & conquer, in-place)
Pick a **pivot**, **partition** so smaller elements go left and larger go right, then recursively sort each side.
```java
void quickSort(int[] arr, int lo, int hi) {
    if (lo >= hi) return;
    int p = partition(arr, lo, hi);           // pivot now in its final position
    quickSort(arr, lo, p - 1);               // sort left of pivot
    quickSort(arr, p + 1, hi);               // sort right of pivot
}
int partition(int[] arr, int lo, int hi) {
    int pivot = arr[hi];                       // choose last element as pivot
    int i = lo - 1;
    for (int j = lo; j < hi; j++)
        if (arr[j] < pivot) { i++; swap(arr, i, j); }   // move smaller elements left
    swap(arr, i + 1, hi);                     // place pivot
    return i + 1;
}
void swap(int[] a, int i, int j) { int t = a[i]; a[i] = a[j]; a[j] = t; }
```
- Time: **O(n log n)** average, **O(n²) worst** (bad pivots — e.g., already-sorted with last-element pivot). Space: O(log n) (recursion). **Not stable.**
- ✅ Fast in practice (good cache locality, in-place). ⚠️ O(n²) worst case → mitigated by **random/median-of-three pivot selection**.
- **Used by Java for sorting primitives** (`Arrays.sort(int[])`) — dual-pivot quicksort (§6).

### 3.3 Heap Sort (uses a heap, in-place)
Build a **max-heap** (Phase 2.2), then repeatedly extract the max to the end.
```java
void heapSort(int[] arr) {
    int n = arr.length;
    for (int i = n / 2 - 1; i >= 0; i--) siftDown(arr, i, n);   // build max-heap: O(n)
    for (int i = n - 1; i > 0; i--) {
        swap(arr, 0, i);                       // move max to the end
        siftDown(arr, 0, i);                  // restore heap on the shrunken range
    }
}
void siftDown(int[] a, int i, int n) {
    while (2*i + 1 < n) {
        int child = 2*i + 1;
        if (child + 1 < n && a[child+1] > a[child]) child++;
        if (a[i] >= a[child]) break;
        swap(a, i, child); i = child;
    }
}
```
- Time: **O(n log n)** all cases. Space: **O(1)** (in-place!). **Not stable.**
- ✅ Guaranteed O(n log n) AND in-place. ⚠️ Poorer cache locality than quicksort → usually slower in practice.

### 3.4 Efficient sorts comparison
| Sort | Best | Avg | Worst | Space | Stable | Java uses |
|------|------|-----|-------|-------|--------|-----------|
| **Merge** | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ | Objects (TimSort) |
| **Quick** | O(n log n) | O(n log n) | **O(n²)** | O(log n) | ❌ | Primitives (dual-pivot) |
| **Heap** | O(n log n) | O(n log n) | O(n log n) | **O(1)** | ❌ | (PriorityQueue) |

---

## 4. Non-Comparison Sorts — Beat O(n log n)

Comparison sorts can't beat O(n log n). But if you **don't compare** elements — instead using their values directly — you can achieve **O(n)** under specific conditions (limited range of values).

### 4.1 Counting Sort
Count occurrences of each value, then reconstruct in order. Works when values are **small integers in a known range**.
```java
int[] countingSort(int[] arr, int maxVal) {
    int[] count = new int[maxVal + 1];
    for (int x : arr) count[x]++;             // count each value
    int[] result = new int[arr.length];
    int idx = 0;
    for (int v = 0; v <= maxVal; v++)
        while (count[v]-- > 0) result[idx++] = v;   // emit in order
    return result;
}
```
- Time: **O(n + k)** (k = value range). Space: O(k). **Stable** (careful impl).
- ⚠️ Only practical when **k is small** (e.g., ages 0–120, grades). If k ≫ n, it's wasteful.

### 4.2 Radix Sort
Sort numbers digit by digit (least significant first), using a stable sort (often counting sort) per digit.
- Time: **O(d × (n + k))** (d = number of digits). Space: O(n + k). **Stable.**
- Good for fixed-width integers/strings.

### 4.3 Bucket Sort
Distribute elements into buckets, sort each bucket, concatenate. Good for **uniformly distributed** data.
- Time: O(n) average (uniform), O(n²) worst. Space: O(n).

| Non-comparison sort | Time | When to use |
|---------------------|------|-------------|
| **Counting** | O(n + k) | Small integer range |
| **Radix** | O(d(n+k)) | Fixed-width integers/strings |
| **Bucket** | O(n) avg | Uniformly distributed values |
> These beat O(n log n) by exploiting structure in the data — but only under specific conditions (small range, uniform distribution). General-purpose data still needs comparison sorts.

---

## 5. Choosing a Sort

```
Small / nearly sorted        -> Insertion sort
Need guaranteed O(n log n)   -> Merge sort or Heap sort
Need stability               -> Merge sort (or TimSort)
In-place + fast on average   -> Quick sort
Memory-constrained           -> Heap sort (O(1) space)
Small integer range          -> Counting sort
Fixed-width integers         -> Radix sort
General use (just use Java!)  -> Arrays.sort / Collections.sort
```

---

## 6. What Java Actually Uses (Important)

You should almost always use the JDK's sorts — they're highly optimized hybrids:
| Method | Algorithm |
|--------|-----------|
| **`Arrays.sort(int[])`** (primitives) | **Dual-pivot quicksort** (fast, not stable — but primitives don't need stability) |
| **`Arrays.sort(Object[])`, `Collections.sort`, `list.sort`, `stream.sorted`** | **TimSort** — a stable, adaptive merge sort + insertion sort hybrid |

### 6.1 TimSort (the object sort)
TimSort is a real-world masterpiece (also Python's default sort):
- **Adaptive:** detects already-sorted "runs" and exploits them → **O(n)** on nearly-sorted data.
- Uses **insertion sort** for small runs (recall §2.3 — insertion sort is great for small/nearly-sorted).
- Uses **merge sort** to combine runs → **stable**, **O(n log n)** worst case.
> This is why §2.3 (insertion sort excels on small/nearly-sorted) and §3.1 (merge sort: stable O(n log n)) matter even though you use the library: **TimSort combines both.** Understanding the parts explains the whole.

### 6.2 Dual-pivot quicksort (the primitive sort)
Uses **two pivots** (partitioning into three regions) for better performance than classic single-pivot quicksort. Falls back to insertion sort for small arrays and heapsort if recursion gets too deep (to avoid O(n²)).

```java
// In practice:
Arrays.sort(intArray);                      // dual-pivot quicksort
Collections.sort(list);                     // TimSort (stable)
list.sort(Comparator.comparing(User::getName).thenComparing(User::getAge));  // Phase 1.6
List<Integer> sorted = stream.sorted().toList();   // TimSort (Phase 1.8)
```

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Hand-writing sorts in production | Use `Arrays.sort`/`Collections.sort` |
| Using quicksort on adversarial input | O(n²) — use random/median pivot or merge/heap |
| Assuming `Arrays.sort(int[])` is stable | It's dual-pivot quicksort (not stable) — but primitives don't need it |
| Counting sort with a huge value range | O(k) space — only for small k |
| Forgetting stability when sorting by multiple keys | Use a stable sort + `thenComparing` (Phase 1.6) |
| `a - b` comparator (overflow) | Use `Integer.compare` (Phase 1.6) |
| Thinking O(n) sorts beat O(n log n) generally | Only under special conditions (small range/uniform) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **`Collections.sort` / `Comparator`** for ordering query results, pagination, leaderboards (Phase 1.6, 4, 5).
- **Database `ORDER BY`** uses sort algorithms internally; indexes (B+ trees, Phase 2.2) can avoid sorting entirely.
- **Stability** matters for multi-key sorting in reports (Phase 5, 7).
- **External/merge sort** concepts apply to sorting data larger than memory (big data, Phase 16).
- **`PriorityQueue` (heap)** for streaming top-K / partial sorting without full sort (Phase 2.2).
- **Understanding TimSort** explains why `Collections.sort` is fast and stable.

---

## 9. Quick Self-Check Questions

1. What three properties characterize a sorting algorithm? Why does stability matter?
2. Which simple sort is best for small/nearly-sorted data, and why?
3. How does merge sort work, and what are its time/space/stability?
4. How does quicksort work, and what causes its O(n²) worst case? How is it mitigated?
5. How does heap sort achieve O(n log n) in O(1) space?
6. How can counting/radix sort beat O(n log n), and what's the catch?
7. What does Java use for `Arrays.sort(int[])` vs `Collections.sort`, and why the difference?
8. What is TimSort, and how does it combine insertion and merge sort?

---

## 10. Key Terms Glossary

- **Stable sort:** preserves relative order of equal elements.
- **In-place:** O(1) auxiliary space.
- **Bubble / selection / insertion sort:** simple O(n²) sorts.
- **Merge sort:** divide-and-conquer, stable, O(n log n), O(n) space.
- **Quick sort:** pivot-partition, O(n log n) avg / O(n²) worst, in-place.
- **Pivot / partition:** quicksort's chosen element / rearrangement step.
- **Heap sort:** heap-based, O(n log n), in-place, not stable.
- **Counting / radix / bucket sort:** non-comparison O(n)-ish sorts.
- **TimSort:** adaptive stable merge+insertion hybrid (Java objects, Python).
- **Dual-pivot quicksort:** Java's primitive sort.

---

*This is the first note of **Section 2.3 — Algorithms**.*
*Next topic: **Searching (Binary Search & Variations)**.*
