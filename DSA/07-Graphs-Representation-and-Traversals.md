# Graphs: Representation & Traversals

> **Phase 2 — Data Structures & Algorithms → 2.2 Data Structures**
> Goal: Master graph fundamentals — representations (adjacency matrix, list, edge list) and traversals (BFS, DFS) — the foundation for the graph algorithms in the next note.

---

## 0. The Big Picture

A **graph** is a set of **vertices (nodes)** connected by **edges**. Unlike trees (a special kind of graph), graphs can have **cycles**, **disconnected parts**, and **arbitrary connections**. Graphs model networks: social networks, maps, dependencies, the web, microservice calls.

```
   A --- B
   |   / |
   |  /  |
   C --- D        (vertices A,B,C,D; edges connect them)
```

> Graphs are the most general structure and appear constantly: shortest paths (maps/routing), dependency resolution (build tools, Maven — Phase 3), social/recommendation systems, network topology. This note covers representation + traversal; algorithms come next.

---

## 1. Graph Terminology

| Term | Meaning |
|------|---------|
| **Vertex (node)** | A point in the graph |
| **Edge** | A connection between two vertices |
| **Directed / undirected** | Edges have direction (A→B) or not (A—B) |
| **Weighted / unweighted** | Edges have a cost/weight or not |
| **Degree** | Number of edges at a vertex (in-degree/out-degree for directed) |
| **Path** | A sequence of vertices connected by edges |
| **Cycle** | A path that returns to its start |
| **Connected** | Every vertex reachable from every other (undirected) |
| **DAG** | Directed Acyclic Graph (directed, no cycles) |

### 1.1 Types of graphs
| Type | Example |
|------|---------|
| **Undirected** | Friendship (mutual) |
| **Directed (digraph)** | Twitter follows (one-way), task dependencies |
| **Weighted** | Road map (distances), network latency |
| **DAG** | Build dependencies, course prerequisites, Maven module graph (Phase 3) |
| **Cyclic** | Most real networks |

> A **tree is a special graph**: connected, undirected (or rooted directed), acyclic, with n−1 edges. Graphs generalize this.

---

## 2. Graph Representations

How you store a graph in memory affects which operations are fast. Three main options:

### 2.1 Adjacency Matrix
A 2D array `matrix[i][j]` = 1 (or the weight) if there's an edge from i to j, else 0.
```
Vertices A,B,C,D (indices 0-3):
       A  B  C  D
    A [0, 1, 1, 0]
    B [1, 0, 0, 1]      matrix[A][B]=1 -> edge A-B
    C [1, 0, 0, 1]
    D [0, 1, 1, 0]
```
```java
int[][] matrix = new int[n][n];
matrix[a][b] = 1;        // undirected: also matrix[b][a] = 1
boolean hasEdge = matrix[a][b] == 1;   // O(1) edge check
```
| Pros | Cons |
|------|------|
| O(1) edge lookup | **O(V²) space** (wasteful for sparse graphs) |
| Simple | O(V) to find a vertex's neighbors |
> Use for **dense** graphs (many edges) or when you frequently check "is there an edge between X and Y?"

### 2.2 Adjacency List (most common)
Each vertex stores a **list of its neighbors** — space-efficient for **sparse** graphs (most real graphs).
```
A -> [B, C]
B -> [A, D]
C -> [A, D]
D -> [B, C]
```
```java
Map<Integer, List<Integer>> adj = new HashMap<>();
adj.computeIfAbsent(a, k -> new ArrayList<>()).add(b);   // edge a->b
adj.computeIfAbsent(b, k -> new ArrayList<>()).add(a);   // undirected: also b->a
// weighted: List<int[]> of {neighbor, weight}, or a list of Edge objects
```
| Pros | Cons |
|------|------|
| **O(V + E) space** (efficient for sparse) | O(degree) edge lookup |
| Fast neighbor iteration | |
> **Default to adjacency lists** — most real graphs are sparse, and you usually iterate neighbors (for traversal/algorithms). (Recall linked lists, Phase 2.2 — adjacency lists use lists per vertex.)

### 2.3 Edge List
Just a list of all edges as pairs (or triples with weight).
```java
List<int[]> edges = List.of(new int[]{0,1}, new int[]{0,2}, new int[]{1,3});  // {from, to, [weight]}
```
| Pros | Cons |
|------|------|
| Simple; compact | Slow neighbor/edge lookup |
| Good for some algorithms (Kruskal's MST — next note) | |
> Used by algorithms that process **all edges** (e.g., Kruskal's, Bellman-Ford — next note).

### 2.4 Comparison
| | Adjacency Matrix | Adjacency List | Edge List |
|---|------------------|----------------|-----------|
| Space | O(V²) | O(V + E) | O(E) |
| Edge lookup | O(1) | O(degree) | O(E) |
| Neighbor iteration | O(V) | O(degree) | O(E) |
| Best for | Dense graphs | **Sparse graphs (default)** | Edge-centric algorithms |

---

## 3. Breadth-First Search (BFS)

**BFS** explores the graph **level by level** — all neighbors first, then their neighbors. Uses a **queue** (recall Phase 2.2 queues). Finds the **shortest path in unweighted graphs**.
```java
void bfs(Map<Integer, List<Integer>> adj, int start) {
    Queue<Integer> queue = new ArrayDeque<>();
    Set<Integer> visited = new HashSet<>();   // CRUCIAL: avoid revisiting (cycles!)
    queue.offer(start);
    visited.add(start);
    while (!queue.isEmpty()) {
        int node = queue.poll();
        process(node);
        for (int neighbor : adj.getOrDefault(node, List.of())) {
            if (!visited.contains(neighbor)) {
                visited.add(neighbor);
                queue.offer(neighbor);
            }
        }
    }
}
```
### 3.1 Key points
- Uses a **queue** (FIFO) → processes nearest nodes first.
- **`visited` set is essential** — graphs have cycles; without it you loop forever.
- **Finds shortest path** (fewest edges) in an **unweighted** graph.
- Complexity: **O(V + E)** time, O(V) space.
> **Uses:** shortest path (unweighted), level-order processing, finding connected components, web crawling by distance, "degrees of separation."

---

## 4. Depth-First Search (DFS)

**DFS** explores **as deep as possible** along each branch before backtracking. Uses **recursion** (the call stack) or an explicit **stack** (recall Phase 2.2 stacks).
```java
// Recursive DFS:
void dfs(Map<Integer, List<Integer>> adj, int node, Set<Integer> visited) {
    if (visited.contains(node)) return;
    visited.add(node);
    process(node);
    for (int neighbor : adj.getOrDefault(node, List.of()))
        dfs(adj, neighbor, visited);          // recurse deeper
}

// Iterative DFS (explicit stack — avoids StackOverflow on deep graphs):
void dfsIterative(Map<Integer, List<Integer>> adj, int start) {
    Deque<Integer> stack = new ArrayDeque<>();
    Set<Integer> visited = new HashSet<>();
    stack.push(start);
    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (!visited.add(node)) continue;     // already visited?
        process(node);
        for (int neighbor : adj.getOrDefault(node, List.of()))
            if (!visited.contains(neighbor)) stack.push(neighbor);
    }
}
```
### 4.1 Key points
- Uses **recursion/stack** (LIFO) → goes deep first.
- Also needs a **`visited` set** (cycles).
- Complexity: **O(V + E)** time, O(V) space (recursion depth/visited).
- ⚠️ Recursive DFS on a very deep graph risks **StackOverflow** (Phase 2.1) → use the iterative version.
> **Uses:** cycle detection, topological sort, connected components, pathfinding, maze solving, backtracking (Phase 2.3).

---

## 5. BFS vs DFS

| | **BFS** | **DFS** |
|---|---------|---------|
| Data structure | Queue (FIFO) | Stack / recursion (LIFO) |
| Explores | Level by level (wide) | Deep first (then backtrack) |
| Shortest path (unweighted) | ✅ Yes | ❌ No (not guaranteed) |
| Memory | O(width) — can be large | O(depth) |
| Best for | Shortest path, nearest nodes | Cycles, topological sort, all paths |
| Complexity | O(V + E) | O(V + E) |
> **Both are O(V + E)** and both need a **visited set**. Choose BFS for shortest-unweighted-path/nearest; DFS for exhaustive exploration, cycles, ordering.

---

## 6. The Critical Detail: Avoiding Infinite Loops

> ⚠️ **Always track visited vertices.** Unlike trees (no cycles), graphs can have cycles — without a `visited` set, BFS/DFS loop forever. This is the #1 graph traversal bug.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| No `visited` set → infinite loop | Always track visited |
| Adjacency matrix for sparse graphs | Use an adjacency list (O(V+E) space) |
| Using DFS for shortest unweighted path | Use BFS |
| Recursive DFS on deep graphs → StackOverflow | Use iterative DFS |
| Forgetting both directions for undirected edges | Add edge both ways in the adjacency list |
| Confusing directed vs undirected handling | Model edges correctly |
| Forgetting disconnected components | Loop over all vertices as start points |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Dependency graphs:** Maven/Gradle module resolution, Spring bean dependency ordering — both are DAGs resolved via topological sort (next note, Phase 3, 5.1).
- **Microservice call graphs / distributed tracing** (Phase 12.7) are graphs.
- **Shortest path / routing** (maps, network latency) — BFS/Dijkstra (next note).
- **Social/recommendation features**, "friends of friends," connected components.
- **Cycle detection:** circular dependencies in beans/modules (Spring fails fast on bean cycles).
- **Web crawling, link analysis** (BFS by distance).

---

## 9. Quick Self-Check Questions

1. What's the difference between a graph and a tree?
2. Define directed/undirected, weighted/unweighted, and DAG.
3. Compare adjacency matrix, adjacency list, and edge list (space & lookup).
4. Which representation is the default and why?
5. How does BFS work, what structure does it use, and what path does it find?
6. How does DFS work, and why prefer the iterative version sometimes?
7. Why is a `visited` set essential in graph traversal?
8. When choose BFS vs DFS?

---

## 10. Key Terms Glossary

- **Graph:** vertices connected by edges.
- **Vertex / edge:** node / connection.
- **Directed / undirected / weighted:** edge properties.
- **DAG:** directed acyclic graph.
- **Degree / in-degree / out-degree:** edge counts at a vertex.
- **Path / cycle / connected:** sequence / loop / reachability.
- **Adjacency matrix / list / edge list:** graph representations.
- **BFS:** breadth-first (queue, shortest unweighted path).
- **DFS:** depth-first (stack/recursion).
- **Visited set:** tracks explored vertices (prevents infinite loops).

---

*Previous topic: **Heaps, Tries & Specialized Trees**.*
*Next topic: **Graph Algorithms & Union-Find**.*
