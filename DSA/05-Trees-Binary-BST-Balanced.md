# Trees: Binary Trees, BST & Balanced Trees

> **Phase 2 — Data Structures & Algorithms → 2.2 Data Structures**
> Goal: Master tree fundamentals — binary trees & traversals, binary search trees (insert/search/delete), and balanced BSTs (AVL, Red-Black) — the structures behind TreeMap, databases, and more.

---

## 0. The Big Picture

A **tree** is a hierarchical structure of **nodes** connected by **edges**, with one **root** and no cycles. Each node has children; nodes with no children are **leaves**. Trees model hierarchy (file systems, org charts, DOM) and enable **O(log n)** search when balanced.

```
        root
        /    \
      node   node
      /  \      \
   leaf  leaf   leaf
```

> Trees are everywhere: `TreeMap`/`TreeSet` (Red-Black trees, Phase 1.6), database indexes (B-trees), heaps (Phase 2.2), tries (Phase 2.2), parse trees, and the DOM. This note covers binary trees → BST → balanced trees.

---

## 1. Tree Terminology

| Term | Meaning |
|------|---------|
| **Root** | The top node (no parent) |
| **Parent / child** | A node and the nodes directly below it |
| **Leaf** | A node with no children |
| **Edge** | A connection between parent and child |
| **Depth** (of a node) | Distance from the root (root = 0) |
| **Height** (of a node/tree) | Longest path to a leaf |
| **Subtree** | A node and all its descendants |
| **Level** | All nodes at the same depth |

---

## 2. Binary Trees

A **binary tree** is a tree where each node has **at most two children** (left and right).

### 2.1 The node
```java
class TreeNode<T> {
    T val;
    TreeNode<T> left;
    TreeNode<T> right;
    TreeNode(T val) { this.val = val; }
}
```

### 2.2 Types of binary trees
| Type | Definition |
|------|-----------|
| **Full** | Every node has 0 or 2 children |
| **Complete** | All levels full except possibly the last, filled left-to-right (heaps! Phase 2.2) |
| **Perfect** | All internal nodes have 2 children; all leaves at the same level |
| **Balanced** | Height ≈ O(log n) (left/right subtree heights differ by ≤1) |
| **Degenerate** | Each node has one child → effectively a linked list (height = n) |

> ⚠️ A binary tree's efficiency depends on **balance**: a balanced tree is O(log n) deep; a degenerate (skewed) tree is O(n) deep → operations degrade to linked-list speed. Balancing (§5) prevents this.

---

## 3. Tree Traversals (Must-Know)

Visiting all nodes in a defined order. **Depth-first** (recursive) traversals come in three flavors based on *when you visit the node* relative to its subtrees:

### 3.1 The three DFS traversals
```java
// INORDER: Left -> Node -> Right   (for a BST, yields SORTED order!)
void inorder(TreeNode<Integer> n) {
    if (n == null) return;
    inorder(n.left);
    System.out.print(n.val + " ");   // visit node BETWEEN subtrees
    inorder(n.right);
}

// PREORDER: Node -> Left -> Right  (copy/serialize a tree)
void preorder(TreeNode<Integer> n) {
    if (n == null) return;
    System.out.print(n.val + " ");   // visit node FIRST
    preorder(n.left);
    preorder(n.right);
}

// POSTORDER: Left -> Right -> Node (delete a tree, evaluate expression trees)
void postorder(TreeNode<Integer> n) {
    if (n == null) return;
    postorder(n.left);
    postorder(n.right);
    System.out.print(n.val + " ");   // visit node LAST
}
```
| Traversal | Order | Common use |
|-----------|-------|-----------|
| **Inorder** | Left, Node, Right | **Sorted output of a BST** |
| **Preorder** | Node, Left, Right | Copy/serialize tree |
| **Postorder** | Left, Right, Node | Delete tree, evaluate expressions |

### 3.2 Level-order traversal (BFS) — uses a queue
```java
// LEVEL-ORDER: visit level by level (breadth-first) — uses a QUEUE
void levelOrder(TreeNode<Integer> root) {
    if (root == null) return;
    Queue<TreeNode<Integer>> queue = new ArrayDeque<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        TreeNode<Integer> n = queue.poll();
        System.out.print(n.val + " ");
        if (n.left != null) queue.offer(n.left);
        if (n.right != null) queue.offer(n.right);
    }
}
```
> **DFS uses recursion/stack; BFS uses a queue** (recall Phase 2.2 stacks/queues). Inorder traversal of a BST giving sorted order is a key insight — and a common interview trick.

---

## 4. Binary Search Tree (BST)

A **BST** is a binary tree with the **ordering invariant**: for every node, **all left-subtree values < node < all right-subtree values**. This enables O(log n) search (when balanced).
```
        50
       /  \
     30    70
    /  \   /  \
   20  40 60  80      (left < node < right, everywhere)
```

### 4.1 Search — O(log n) balanced, O(n) worst
```java
TreeNode<Integer> search(TreeNode<Integer> root, int target) {
    if (root == null || root.val == target) return root;
    if (target < root.val) return search(root.left, target);   // go left
    else return search(root.right, target);                     // go right
}
```
> Each comparison **halves** the remaining tree (like binary search, Phase 2.1) → O(log n) when balanced.

### 4.2 Insert
```java
TreeNode<Integer> insert(TreeNode<Integer> root, int val) {
    if (root == null) return new TreeNode<>(val);
    if (val < root.val) root.left = insert(root.left, val);
    else if (val > root.val) root.right = insert(root.right, val);
    return root;   // duplicates ignored here
}
```

### 4.3 Delete (three cases)
```java
TreeNode<Integer> delete(TreeNode<Integer> root, int val) {
    if (root == null) return null;
    if (val < root.val) root.left = delete(root.left, val);
    else if (val > root.val) root.right = delete(root.right, val);
    else {
        // Found the node to delete:
        if (root.left == null) return root.right;    // Case 1: no left child (or leaf)
        if (root.right == null) return root.left;    // Case 2: no right child
        // Case 3: two children -> replace with inorder successor (smallest in right subtree)
        TreeNode<Integer> successor = findMin(root.right);
        root.val = successor.val;
        root.right = delete(root.right, successor.val);
    }
    return root;
}
TreeNode<Integer> findMin(TreeNode<Integer> n) {
    while (n.left != null) n = n.left;
    return n;
}
```
| Delete case | Action |
|-------------|--------|
| **Leaf / one child** | Remove / replace with the child |
| **Two children** | Replace value with the **inorder successor** (min of right subtree), then delete that successor |

### 4.4 BST complexity
| Operation | Balanced | Worst (degenerate) |
|-----------|----------|--------------------|
| Search / insert / delete | **O(log n)** | O(n) |
> ⚠️ A plain BST can **degenerate** to O(n) if you insert sorted data (it becomes a linked list). **Balanced BSTs (§5) guarantee O(log n)**.

---

## 5. Balanced BSTs (Guaranteeing O(log n))

A **self-balancing BST** automatically keeps its height ≈ O(log n) via **rotations** during insert/delete, preventing degeneration.

### 5.1 Rotations (the balancing primitive)
A **rotation** rearranges nodes to reduce height while preserving the BST invariant:
```
Left rotation around x:           Right rotation around y:
    x                  y               y              x
     \      -->       / \             /     -->        \
      y              x   z           x                  y
       \                                                  
        z                                                
(rotations are O(1) and keep left<node<right intact)
```

### 5.2 AVL Trees
- **Strictly balanced:** for every node, the heights of left/right subtrees differ by **at most 1** (the "balance factor").
- After each insert/delete, rotations restore balance.
- **Faster lookups** (more rigidly balanced) but **more rotations** on insert/delete.
- Use when reads vastly outnumber writes.

### 5.3 Red-Black Trees (the practical default)
- **Loosely balanced** using node "colors" (red/black) and rules ensuring the longest path ≤ 2× the shortest.
- **Fewer rotations** than AVL on insert/delete → better for write-heavy workloads.
- **This is what Java's `TreeMap`/`TreeSet` use** (recall Phase 1.6), and many language libraries.

### 5.4 AVL vs Red-Black
| | AVL | Red-Black |
|---|-----|-----------|
| Balance | Strict (height diff ≤ 1) | Loose (≤ 2× shortest path) |
| Lookups | Faster (more balanced) | Slightly slower |
| Insert/delete | More rotations | Fewer rotations |
| Used by | DBs needing fast reads | **Java TreeMap/TreeSet**, Linux kernel |
> **Default to Red-Black** (it's what the JDK uses) — a good all-around balance. AVL when reads dominate. Both guarantee **O(log n)** for all operations.

---

## 6. Why Balanced Trees Matter (vs HashMap)

| | Hash table (HashMap) | Balanced BST (TreeMap) |
|---|----------------------|------------------------|
| Lookup | O(1) average | O(log n) |
| Ordering | None | **Sorted** |
| Range queries | No | **Yes** (floor/ceiling/subMap) |
| Worst case | O(n)/O(log n) | **O(log n) guaranteed** |
> Use a **hash table** for fastest unordered lookup; use a **balanced BST** when you need **sorted order, range queries, or guaranteed O(log n)** (recall Phase 1.6 TreeMap/TreeSet, NavigableMap).

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Plain BST degenerating on sorted input | Use a balanced BST (TreeMap) |
| Confusing the three DFS traversals | Inorder=L-N-R (sorted for BST); pre=N-L-R; post=L-R-N |
| Using recursion on very deep/unbalanced trees | Risk of StackOverflow (Phase 2.1) — iterative/BFS |
| Forgetting BST delete's two-children case | Replace with inorder successor |
| Expecting O(log n) without balancing | Only balanced trees guarantee it |
| Using a tree when a hash map suffices | TreeMap only if you need ordering/ranges |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **`TreeMap`/`TreeSet`** (Red-Black trees) for sorted keys & range queries (Phase 1.6).
- **Database indexes** use **B-trees/B+ trees** (Phase 2.2 next note, Phase 4.4) — generalized balanced trees for disk.
- **Inorder traversal** → sorted output; **BFS/level-order** → hierarchical processing.
- **Expression/parse trees** in query parsers and compilers.
- **DOM / JSON trees** in parsing (Phase 5.3 Jackson).
- **Interview staples:** traversals, BST validation, lowest common ancestor, tree DP (Phase 2.4).

---

## 9. Quick Self-Check Questions

1. Define root, leaf, height, and depth.
2. What's the difference between a complete, full, and balanced binary tree?
3. Describe the three DFS traversals and a use for each. Which gives sorted BST output?
4. What's the BST ordering invariant, and why does it give O(log n) search?
5. How do you delete a BST node with two children?
6. Why can a plain BST degenerate to O(n), and how do balanced trees prevent it?
7. What's a rotation, and how do AVL and Red-Black trees differ?
8. When use a TreeMap vs a HashMap?

---

## 10. Key Terms Glossary

- **Tree / node / edge / root / leaf:** hierarchical structure basics.
- **Depth / height:** distance from root / longest path to a leaf.
- **Binary tree:** ≤ 2 children per node.
- **Full/complete/perfect/balanced/degenerate:** tree shapes.
- **Traversal (inorder/preorder/postorder/level-order):** node-visiting orders.
- **DFS / BFS:** depth-first (stack/recursion) / breadth-first (queue).
- **BST:** binary search tree (left < node < right).
- **Inorder successor:** smallest node in the right subtree.
- **Balanced BST:** self-balancing tree guaranteeing O(log n).
- **Rotation:** O(1) rebalancing operation.
- **AVL / Red-Black tree:** strict / loose self-balancing BSTs.

---

*Previous topic: **Hash Tables**.*
*Next topic: **Heaps, Tries & Specialized Trees (Segment/Fenwick/B-Tree)**.*
