# Arrays & Dynamic Arrays

> **Phase 2 — Data Structures & Algorithms → 2.2 Data Structures**
> Goal: Master arrays (the most fundamental structure) and dynamic arrays — their memory layout, operations & complexity, how resizing works, and when to use them.

---

## 0. The Big Picture

An **array** is a **contiguous block of memory** holding a fixed number of same-type elements, accessed by **zero-based index** in **O(1)**. A **dynamic array** (Java's `ArrayList`) wraps an array and **grows automatically**. Arrays are the foundation: most other structures (hash tables, heaps, stacks) are built on them.

```
Array (contiguous in memory):
  index:  0    1    2    3    4
        [ 10 ][ 20 ][ 30 ][ 40 ][ 50 ]
  address: base, base+4, base+8, ...   (each int = 4 bytes)
```

> Arrays are **cache-friendly** (contiguous → great locality, recall Phase 0.2) — which is why they often outperform "theoretically faster" pointer structures (recall Phase 1.6 ArrayList vs LinkedList).

---

## 1. Why Random Access Is O(1)

The element at index `i` lives at a **computable address**:
```
address(arr[i]) = base_address + (i × element_size)
```
The CPU computes this directly — no traversal needed → **O(1)** access. This is the array's superpower (recall Phase 1.2 arrays, Phase 0.1 memory).

---

## 2. Array Operations & Complexity

| Operation | Complexity | Why |
|-----------|-----------|-----|
| **Access** `arr[i]` | **O(1)** | Direct address computation |
| **Update** `arr[i] = x` | **O(1)** | Direct write |
| **Search** (unsorted) | O(n) | Must scan |
| **Search** (sorted) | O(log n) | Binary search |
| **Insert at end** (room available) | O(1) | Write to next slot |
| **Insert at middle/start** | **O(n)** | Shift elements right |
| **Delete at middle/start** | **O(n)** | Shift elements left |

### 2.1 Why insert/delete in the middle is O(n)
```
Insert 99 at index 1 in [10, 20, 30, 40]:
  Step 1: shift 20,30,40 right -> [10, _, 20, 30, 40]
  Step 2: place 99            -> [10, 99, 20, 30, 40]
  (shifting up to n elements -> O(n))
```
> Arrays are great for **access** but expensive for **insertion/deletion in the middle**. If you need frequent middle inserts, consider other structures — though in practice array shifting is fast due to cache locality (Phase 1.6).

---

## 3. Static Arrays in Java (recap from Phase 1.2)

```java
int[] arr = new int[5];              // fixed size 5, all 0
int[] nums = {10, 20, 30};            // literal
arr[0] = 99;                          // O(1) update
int x = arr[0];                       // O(1) access
int len = arr.length;                 // length is a FIELD, not a method
```
- **Fixed size** — can't grow/shrink (Phase 1.2).
- Elements default to `0`/`false`/`null`.
- `ArrayIndexOutOfBoundsException` on invalid index (Phase 1.2).

---

## 4. Dynamic Arrays (How ArrayList Works Internally)

A **dynamic array** stores elements in an internal fixed array and **resizes** when full. This is `ArrayList` (recall Phase 1.6) — let's implement one to understand it.

### 4.1 A simplified dynamic array implementation
```java
public class DynamicArray<T> {
    private Object[] data;
    private int size = 0;              // number of elements (≤ capacity)

    public DynamicArray() {
        data = new Object[10];        // initial capacity
    }

    public void add(T element) {
        if (size == data.length) {    // FULL -> resize
            resize();
        }
        data[size++] = element;       // amortized O(1)
    }

    @SuppressWarnings("unchecked")
    public T get(int index) {
        if (index < 0 || index >= size) throw new IndexOutOfBoundsException();
        return (T) data[index];       // O(1)
    }

    private void resize() {
        int newCapacity = data.length * 2;        // DOUBLE the capacity
        Object[] newData = new Object[newCapacity];
        System.arraycopy(data, 0, newData, 0, size);  // copy all -> O(n)
        data = newData;
    }

    public int size() { return size; }
}
```

### 4.2 The doubling strategy & amortized O(1)
```
Capacity grows: 10 -> 20 -> 40 -> 80 -> ...  (doubles each time it fills)
```
- Most `add`s are **O(1)** (write to next slot).
- When full, **resize is O(n)** (allocate bigger array + copy everything).
- But doubling means resizes are **rare** — each resize postpones the next → averaged over all adds, the cost is **amortized O(1)** (recall Phase 2.1 amortized analysis).
> **Why double (not add a fixed amount)?** Doubling gives amortized O(1). Growing by a constant (e.g., +10) would make n adds cost O(n²). Java's `ArrayList` grows by ~1.5×.

### 4.3 Pre-sizing (performance)
If you know the size, give an initial capacity to **avoid resizes** (recall Phase 1.6):
```java
List<Integer> list = new ArrayList<>(10_000);   // no repeated resizing
```

---

## 5. Multi-Dimensional Arrays (recap Phase 1.2)

Java has no true 2D arrays — they're **arrays of arrays**:
```java
int[][] grid = new int[3][4];        // 3 rows × 4 columns
grid[1][2] = 5;
int[][] jagged = new int[3][];        // rows can have different lengths
```
- `matrix.length` = rows; `matrix[0].length` = columns.
- Row-major in memory; iterating row-by-row is more cache-friendly than column-by-column.

---

## 6. Common Array Techniques (preview — Phase 2.3)

Arrays power many algorithm patterns you'll use constantly:
| Technique | Use |
|-----------|-----|
| **Two pointers** | Pair-finding, palindromes, in-place ops (Phase 2.3) |
| **Sliding window** | Subarray/substring problems (Phase 2.3) |
| **Prefix sums** | Range-sum queries in O(1) after O(n) preprocessing |
| **Binary search** | Sorted-array search in O(log n) |
| **In-place manipulation** | O(1) extra space (reverse, rotate) |

```java
// Prefix sum: precompute, then O(1) range queries
int[] prefix = new int[n + 1];
for (int i = 0; i < n; i++) prefix[i+1] = prefix[i] + arr[i];
int rangeSum = prefix[r+1] - prefix[l];   // sum of arr[l..r] in O(1)
```

---

## 7. Array Copying (recap Phase 1.2 — shallow vs deep)

```java
int[] copy = Arrays.copyOf(src, src.length);     // true copy (primitives)
int[] range = Arrays.copyOfRange(src, 1, 4);      // sub-range
System.arraycopy(src, 0, dst, 0, len);            // fast low-level copy
int[] b = a;                                       // NOT a copy — same array!
```
> For object/2D arrays, `clone()`/`copyOf` are **shallow** (elements shared) — deep-copy each element for independence (recall Phase 1.2/1.3).

---

## 8. When to Use Arrays vs Other Structures

| Use an array / ArrayList when... | Use something else when... |
|----------------------------------|-----------------------------|
| You need **fast index access** | You need fast key lookup → **HashMap** |
| You iterate frequently | You need uniqueness → **HashSet** |
| Size is known / append-mostly | Frequent middle insert/delete → consider **LinkedList** (rarely) |
| Cache locality matters (performance) | You need sorted order → **TreeMap/TreeSet** |
| Building other structures (heaps, hash tables) | FIFO/LIFO processing → **ArrayDeque** |

> **Default to `ArrayList`** for sequences (recall Phase 1.6) — it covers ~90% of needs and is cache-friendly.

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Expecting middle insert/delete to be cheap | It's O(n) (shifting) |
| `int[] b = a` thinking it copies | It's a reference; use `copyOf` |
| Off-by-one / `ArrayIndexOutOfBoundsException` | Indices are `0..length-1` |
| Not pre-sizing a large dynamic array | Pass initial capacity |
| Shallow-copying object/2D arrays | Deep-copy elements for independence |
| Growing a dynamic array by a constant | Double (amortized O(1)) |
| Column-major iteration of a 2D array | Iterate row-by-row (cache locality) |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **`ArrayList`** is the default collection in services, DTOs, query results (Phase 1.6, 5).
- **Cache locality** of arrays explains real performance vs pointer structures (Phase 13).
- **`byte[]`** for I/O, file handling, serialization, hashing (Phase 1.9).
- **Prefix sums / two pointers / sliding window** appear in data processing (Project 2) and interviews.
- Arrays back **hash tables, heaps, dynamic arrays** — the structures you'll use everywhere.
- **Defensive copying** of arrays protects encapsulation (Phase 1.3).

---

## 11. Quick Self-Check Questions

1. Why is array access O(1)? (Give the address formula.)
2. Why is inserting in the middle of an array O(n)?
3. How does a dynamic array (`ArrayList`) grow, and why is `add` amortized O(1)?
4. Why double the capacity instead of adding a fixed amount?
5. How do you avoid repeated resizing if you know the size?
6. How are 2D arrays represented in Java? What's a jagged array?
7. What is a prefix sum, and what does it enable?
8. When should you use an array vs a HashMap vs a TreeMap?

---

## 12. Key Terms Glossary

- **Array:** fixed-size contiguous block of same-type elements.
- **Random access:** O(1) access by index (address computation).
- **Dynamic array:** auto-resizing array (`ArrayList`).
- **Resizing / doubling:** allocating a bigger array and copying (amortized O(1)).
- **Amortized O(1):** average constant cost over many operations (Phase 2.1).
- **Cache locality:** contiguous memory → fast access (Phase 0.2).
- **Multi-dimensional / jagged array:** array of arrays / rows of differing lengths.
- **Prefix sum:** precomputed cumulative sums for O(1) range queries.
- **Shallow vs deep copy:** shared elements vs independent copies.

---

*This is the first note of **Section 2.2 — Data Structures**.*
*Next topic: **Linked Lists**.*
