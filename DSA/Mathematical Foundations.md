

## CONNECTION TO DSA
```
┌─────────────────────────────────────────────────────────────┐
│                    MATH → DSA CONNECTION MAP                │
├──────────────────────┬──────────────────────────────────────┤
│ Math Concept         │ Where You'll Use It in DSA           │
├──────────────────────┼──────────────────────────────────────┤
│ Logarithms           │ Binary Search O(log n)               │
│                      │ Tree height analysis                 │
│                      │ Heap operations O(log n)             │
│                      │ Divide & Conquer analysis            │
├──────────────────────┼──────────────────────────────────────┤
│ Powers of 2          │ Tree node counts                     │
│                      │ Bit manipulation                     │
│                      │ Subset generation (2^n)              │
│                      │ Dynamic array resizing               │
├──────────────────────┼──────────────────────────────────────┤
│ Summation Formulas   │ Sorting complexity O(n²)             │
│                      │ Loop analysis                        │
│                      │ Graph edge counts n(n-1)/2           │
├──────────────────────┼──────────────────────────────────────┤
│ Modular Arithmetic   │ Hash table indexing                  │
│                      │ Circular arrays/queues               │
│                      │ Overflow prevention                  │
│                      │ Cycle detection                      │
├──────────────────────┼──────────────────────────────────────┤
│ Combinatorics        │ Backtracking complexity              │
│                      │ DP on subsets                        │
│                      │ Counting paths in graphs             │
│                      │ Catalan numbers (BST counting)       │
├──────────────────────┼──────────────────────────────────────┤
│ Probability          │ Hash collision analysis              │
│                      │ Randomized algorithms                │
│                      │ Expected time complexity             │
│                      │ Load factor in HashMap               │
├──────────────────────┼──────────────────────────────────────┤
│ Proof by Induction   │ Understanding recursion              │
│                      │ Proving algorithm correctness        │
│                      │ Thinking about base cases            │
└──────────────────────┴──────────────────────────────────────┘
```

## REFERENCE CHEAT SHEET
```
LOGARITHMS:
  log₂(n) = "how many times halve n to reach 1"
  log₂(1024) = 10
  Binary Search → O(log n) steps

POWERS OF 2:
  2^10 = 1024 (≈ 1K)
  2^20 = 1M
  2^30 = 1B
  int max = 2^31 - 1

SUMMATION:
  1+2+...+n = n(n+1)/2
  1+2+4+...+2^n = 2^(n+1) - 1

MODULO:
  a % m = remainder
  (a+b)%m = ((a%m)+(b%m))%m
  Negative fix: ((n%m)+m)%m
  Circular next: (i+1)%n

COMBINATORICS:
  Permutation P(n,r) = n!/(n-r)!  [order matters]
  Combination C(n,r) = n!/(r!(n-r)!) [order doesn't]
  Subsets of n elements = 2^n
  n! grows EXTREMELY fast

PROBABILITY:
  Collision happens faster than you think!
  Java HashMap default load factor = 0.75

INDUCTION = RECURSION:
  Base case → base case
  Inductive step → recursive call
  Trust it works for smaller input!
```