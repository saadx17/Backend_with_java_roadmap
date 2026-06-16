# Bit Manipulation & String Algorithms

> **Phase 2 — Data Structures & Algorithms → 2.3 Algorithms**
> Goal: Master bit manipulation (AND, OR, XOR, shifts, bitmask tricks) and the essential string-matching algorithms (KMP, Rabin-Karp).

---

## 0. The Big Picture

Two specialized but important toolkits:
- **Bit manipulation** — operating directly on the binary representation of numbers (recall Phase 0.1 binary, Phase 1.2 bitwise operators) for fast, compact operations.
- **String algorithms** — efficient pattern matching (finding a substring) better than the naive O(n×m) approach.

> Bit tricks power flags, sets, hashing, and low-level optimizations. String matching underlies search, parsing, and text processing. Both appear in interviews (Phase 2.4) and real systems.

---

## 1. Bit Manipulation Fundamentals (recap Phase 1.2)

Numbers are stored in binary (Phase 0.1). Bitwise operators act on individual bits:
| Operator | Name | Example (a=`1100`, b=`1010`) |
|----------|------|------------------------------|
| `&` | AND | `a & b` = `1000` (8) |
| `\|` | OR | `a \| b` = `1110` (14) |
| `^` | XOR | `a ^ b` = `0110` (6) |
| `~` | NOT | flips all bits |
| `<<` | left shift | `a << 1` = `11000` (×2) |
| `>>` | signed right shift | `a >> 1` (÷2, keeps sign) |
| `>>>` | unsigned right shift | shifts in 0s (Phase 1.2) |

---

## 2. Essential Bit Tricks

These show up constantly in interviews and optimizations:
```java
// Check if the i-th bit is set:
boolean isSet = (n & (1 << i)) != 0;

// Set the i-th bit:
n = n | (1 << i);

// Clear the i-th bit:
n = n & ~(1 << i);

// Toggle the i-th bit:
n = n ^ (1 << i);

// Check if a number is even/odd (last bit):
boolean isEven = (n & 1) == 0;

// Multiply/divide by 2:
n << 1;   // × 2
n >> 1;   // ÷ 2

// Check if a number is a power of 2 (exactly one bit set):
boolean isPowerOf2 = n > 0 && (n & (n - 1)) == 0;

// Remove the lowest set bit:
n = n & (n - 1);

// Isolate the lowest set bit:
int lowest = n & (-n);

// Count set bits (population count):
int count = Integer.bitCount(n);   // built-in, fast
```
> The `n & (n-1)` trick (clears the lowest set bit) is a famous one — used to **count set bits** (loop until n is 0) and to check powers of 2. Memorize the set/clear/toggle/check patterns.

---

## 3. XOR — The Magic Operator

XOR (`^`) has special properties that enable elegant tricks:
| Property | Meaning |
|----------|---------|
| `x ^ x = 0` | A value XORed with itself is 0 |
| `x ^ 0 = x` | XOR with 0 is identity |
| Commutative & associative | Order doesn't matter |

### 3.1 Find the single non-duplicated number
Given an array where every number appears twice except one, XOR everything — duplicates cancel to 0, leaving the unique number:
```java
int findSingle(int[] nums) {
    int result = 0;
    for (int n : nums) result ^= n;   // pairs cancel (x^x=0), unique survives
    return result;
}
// [4, 1, 2, 1, 2] -> 4   (O(n) time, O(1) space — no hash map needed!)
```

### 3.2 Swap without a temp variable
```java
a ^= b; b ^= a; a ^= b;   // swaps a and b using XOR (rarely used, but classic)
```
> XOR's "cancellation" property powers single-number problems, finding missing numbers, and some cryptography. It's a favorite interview trick.

---

## 4. Bitmasking (Representing Sets with Bits)

A **bitmask** uses the bits of an integer to represent a **set** of items (each bit = "is this item present?"). Compact and fast — the basis of `EnumSet` (recall Phase 1.3/1.6).
```java
int mask = 0;
mask |= (1 << 2);              // add item 2 to the set
mask |= (1 << 5);             // add item 5
boolean has2 = (mask & (1 << 2)) != 0;   // is item 2 present?
mask &= ~(1 << 2);            // remove item 2
int size = Integer.bitCount(mask);        // number of items in the set
```

### 4.1 Iterating all subsets (bitmask subsets)
With n items, the integers `0` to `2ⁿ - 1` represent **all possible subsets** — each bit pattern is one subset:
```java
for (int mask = 0; mask < (1 << n); mask++) {     // all 2ⁿ subsets
    for (int i = 0; i < n; i++) {
        if ((mask & (1 << i)) != 0) {              // is item i in this subset?
            // item i is included
        }
    }
}
```
> This is the **bitmask alternative to backtracking subsets** (Phase 2.3) — iterating `0..2ⁿ-1`. It's also the foundation of **bitmask DP** (e.g., traveling salesman with `dp[mask][i]`).

### 4.2 Why bitmasks are useful
- **Compact:** a set of up to 32/64 items fits in one int/long.
- **Fast:** set operations are single CPU instructions (union = `|`, intersection = `&`, difference = `& ~`).
- **Used by:** `EnumSet` (Phase 1.3), feature flags, permissions, state in bitmask DP, graph subsets.
> ⚠️ Bit tricks are powerful but can hurt **readability**. In application code, prefer `EnumSet`/`BitSet` (clear intent); use raw bit manipulation for performance-critical or competitive code.

---

## 5. String Matching: The Problem

**Pattern matching:** find all occurrences of a pattern `P` (length m) in a text `T` (length n).
```
Text:    "ABABDABACDABABCABAB"
Pattern: "ABABC"
Find where the pattern occurs.
```

### 5.1 Naive approach — O(n × m)
Check the pattern at every position, comparing character by character:
```java
List<Integer> naiveSearch(String text, String pattern) {
    List<Integer> matches = new ArrayList<>();
    int n = text.length(), m = pattern.length();
    for (int i = 0; i <= n - m; i++) {
        int j = 0;
        while (j < m && text.charAt(i + j) == pattern.charAt(j)) j++;
        if (j == m) matches.add(i);
    }
    return matches;
}
```
- O(n × m) worst case — re-checks characters redundantly. The smarter algorithms avoid this redundancy.

---

## 6. KMP (Knuth-Morris-Pratt) — O(n + m)

**KMP** avoids re-comparing characters by precomputing a **failure function (LPS array)** — the length of the longest proper prefix that's also a suffix — so on a mismatch it knows how far to skip without backtracking in the text.
```
The key insight: when a mismatch occurs after matching some characters,
we already know those characters — use that info to skip ahead instead of restarting.
```

### 6.1 Building the LPS array
```java
int[] computeLPS(String pattern) {
    int m = pattern.length();
    int[] lps = new int[m];          // lps[i] = longest proper prefix that's also a suffix of pattern[0..i]
    int len = 0;
    for (int i = 1; i < m; ) {
        if (pattern.charAt(i) == pattern.charAt(len)) lps[i++] = ++len;
        else if (len > 0) len = lps[len - 1];   // fall back
        else lps[i++] = 0;
    }
    return lps;
}
```
### 6.2 KMP search
```java
List<Integer> kmpSearch(String text, String pattern) {
    int[] lps = computeLPS(pattern);
    List<Integer> matches = new ArrayList<>();
    int i = 0, j = 0;                 // i -> text, j -> pattern
    while (i < text.length()) {
        if (text.charAt(i) == pattern.charAt(j)) { i++; j++; }
        if (j == pattern.length()) { matches.add(i - j); j = lps[j - 1]; }  // match found
        else if (i < text.length() && text.charAt(i) != pattern.charAt(j)) {
            if (j > 0) j = lps[j - 1];   // skip using LPS (no text backtrack!)
            else i++;
        }
    }
    return matches;
}
```
- **Time: O(n + m)** — linear! The text pointer `i` never goes backward.
> KMP's brilliance: the LPS array lets it **never re-examine text characters**, turning O(n×m) into O(n+m).

---

## 7. Rabin-Karp — O(n + m) Average (Hashing)

**Rabin-Karp** uses **hashing** (recall Phase 2.2): compute a **rolling hash** of the pattern and of each text window; compare hashes (O(1)) instead of characters. On a hash match, verify with a direct comparison (to handle hash collisions).
```
1. Hash the pattern.
2. Hash the first window of the text.
3. Slide the window: update the hash in O(1) (a "rolling hash" — remove old char, add new char).
4. If window hash == pattern hash -> verify character by character (collision check).
```
- **Time:** O(n + m) average; O(n × m) worst (many collisions). Space: O(1).
- The **rolling hash** is the key — like a sliding window (Phase 2.3) where you update the hash incrementally rather than recomputing.

### 7.1 When to use which
| Algorithm | Best for |
|-----------|----------|
| **Naive** | Tiny inputs, simplicity |
| **KMP** | Single-pattern, guaranteed O(n+m) |
| **Rabin-Karp** | **Multiple patterns** (hash all patterns), plagiarism/duplicate detection |
> In practice, you'll usually use Java's built-in `String.indexOf` / `Pattern`/`Matcher` (regex, Phase 1.4) — which use efficient algorithms internally. Know KMP/Rabin-Karp for interviews and understanding.

---

## 8. Other String Concepts (recap/awareness)
- **Tries** (Phase 2.2) — prefix matching, autocomplete.
- **Regular expressions** (Phase 1.4) — `Pattern`/`Matcher` for flexible matching.
- **Suffix arrays / suffix trees** — advanced substring problems (awareness).
- **Anagrams / palindromes** — frequency counting (HashMap) or two pointers (Phase 2.3).

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `>>` vs `>>>` on negative numbers | Use `>>>` for logical/unsigned shift (Phase 1.2) |
| Overusing bit tricks (unreadable) | Prefer `EnumSet`/`BitSet` in app code |
| Forgetting Rabin-Karp's collision check | Always verify on a hash match |
| Hand-writing string search in production | Use `indexOf`/`Pattern` |
| Off-by-one in shifts/masks | Test bit indices carefully |
| Assuming bitmask scales beyond 32/64 items | int=32 bits, long=64 bits; use `BitSet` for more |
| Naive O(n×m) search on large text | Use KMP / built-in methods |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Bitmasks / flags:** `EnumSet` for permissions/feature flags (Phase 1.3/5), compact state representation.
- **Bloom filters** (Phase 2.2) use bit arrays and hashing — bit manipulation in action.
- **Hashing** (Rabin-Karp's rolling hash) connects to hash tables (Phase 2.2) and content hashing.
- **String search / matching:** `indexOf`, regex (Phase 1.4), full-text search (Elasticsearch, Phase 16.4).
- **XOR** in checksums, cryptography, error detection (Phase 15).
- **Performance:** bit operations are single CPU instructions (Phase 0.1) — used in hot paths.
- **Interview prep** (Phase 2.4): bit tricks and string matching are common categories.

---

## 11. Quick Self-Check Questions

1. How do you check, set, clear, and toggle the i-th bit?
2. How do you check if a number is a power of 2 in O(1)?
3. What does `n & (n-1)` do, and what's it used for?
4. How does XOR find the single non-duplicated number in O(1) space?
5. What is a bitmask, and how do you iterate all subsets with one?
6. Why is naive string search O(n×m), and how does KMP achieve O(n+m)?
7. What is the LPS array in KMP?
8. How does Rabin-Karp use hashing, and what's the "rolling hash"?

---

## 12. Key Terms Glossary

- **Bitwise operators:** `& | ^ ~ << >> >>>`.
- **Bit trick:** set/clear/toggle/check a bit; power-of-2 check.
- **`n & (n-1)`:** clears the lowest set bit (count bits, power-of-2).
- **XOR properties:** `x^x=0`, `x^0=x` (cancellation).
- **Bitmask:** integer bits representing set membership.
- **Bitmask subsets:** iterating `0..2ⁿ-1` for all subsets.
- **Pattern matching:** finding a substring in text.
- **Naive search:** O(n×m) brute-force matching.
- **KMP:** O(n+m) matching via the LPS/failure function.
- **LPS array:** longest proper prefix that is also a suffix.
- **Rabin-Karp:** hashing-based matching with a rolling hash.
- **Rolling hash:** incrementally updated window hash.

---

*Previous topic: **Greedy Algorithms & Divide and Conquer**.*
*This completes **Section 2.3 — Algorithms**.*
*Next section in roadmap: **2.4 Practice Strategy**.*
