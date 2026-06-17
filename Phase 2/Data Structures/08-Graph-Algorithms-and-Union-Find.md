# Graph Algorithms & Union-Find

> **Phase 2 — Data Structures & Algorithms → 2.2 Data Structures**
> Goal: Master the essential graph algorithms — shortest paths (Dijkstra, Bellman-Ford, Floyd-Warshall), minimum spanning trees (Kruskal, Prim), topological sort, cycle detection, Union-Find, and strongly connected components.

---

## 0. The Big Picture

Building on graph representation & traversal (previous note), this note covers the **algorithms** that solve real graph problems: finding shortest routes, cheapest networks, valid orderings, and detecting cycles/clusters. Each has a clear use case and complexity.

> These are interview staples (Phase 2.4) and underpin real systems: routing, dependency resolution, network design, scheduling. Know what each does, when to use it, and its complexity.

---

## 1. Shortest Path Algorithms

Finding the minimum-cost path between vertices. Choice depends on edge weights and how many pairs you need.

### 1.1 BFS — unweighted shortest path (recall previous note)
For **unweighted** graphs, plain **BFS** finds the shortest path (fewest edges) in **O(V + E)**. Use it when edges have no weights.

### 1.2 Dijkstra's Algorithm — weighted, non-negative
Finds the shortest path from a source to all vertices in a **weighted graph with non-negative edges**. Uses a **priority queue (min-heap)** (recall Phase 2.2 heaps) to always expand the closest unvisited vertex.
```java
int[] dijkstra(Map<Integer, List<int[]>> adj, int n, int src) {  // adj: node -> [neighbor, weight]
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[1]));
    pq.offer(new int[]{src, 0});            // {node, distance}
    while (!pq.isEmpty()) {
        int[] cur = pq.poll();
        int node = cur[0], d = cur[1];
        if (d > dist[node]) continue;        // stale entry
        for (int[] edge : adj.getOrDefault(node, List.of())) {
            int next = edge[0], weight = edge[1];
            if (dist[node] + weight < dist[next]) {   // relaxation
                dist[next] = dist[node] + weight;
                pq.offer(new int[]{next, dist[next]});
            }
        }
    }
    return dist;
}
```
- **Complexity:** O((V + E) log V) with a binary heap.
- ⚠️ **Fails with negative edges** (it assumes once a node is finalized, no shorter path exists).
> **The most used shortest-path algorithm** — routing, maps, network latency, "cheapest path." Uses a min-heap to greedily pick the closest node (greedy — Phase 2.3).

### 1.3 Bellman-Ford — handles negative edges
Finds shortest paths from a source, **allowing negative edge weights**, and **detects negative cycles**. Relaxes all edges V−1 times.
- **Complexity:** O(V × E) — slower than Dijkstra.
- Use when **negative weights** exist (e.g., currency arbitrage) or you need negative-cycle detection.

### 1.4 Floyd-Warshall — all pairs shortest paths
Computes shortest paths between **every pair** of vertices using dynamic programming (Phase 2.3).
- **Complexity:** O(V³) time, O(V²) space.
- Use for **small dense graphs** where you need all-pairs distances.

### 1.5 Shortest-path comparison
| Algorithm | Graph | Negative edges? | Complexity | Use |
|-----------|-------|-----------------|------------|-----|
| **BFS** | Unweighted | n/a | O(V+E) | Fewest-edges path |
| **Dijkstra** | Weighted | ❌ No | O((V+E) log V) | Single-source, non-negative (default) |
| **Bellman-Ford** | Weighted | ✅ Yes | O(V·E) | Negative edges / cycle detection |
| **Floyd-Warshall** | Weighted | ✅ Yes | O(V³) | All-pairs (small graphs) |

---

## 2. Minimum Spanning Tree (MST)

An **MST** connects all vertices with the **minimum total edge weight** and **no cycles** (a tree spanning all vertices). Use for designing cheapest networks (cabling, roads, pipelines).

### 2.1 Kruskal's Algorithm (edge-based, uses Union-Find)
Sort all edges by weight; add each edge if it **doesn't form a cycle** (checked via Union-Find, §4).
```
1. Sort edges by weight (ascending)
2. For each edge (u, v): if u and v are in different components (Union-Find), add it & union them
3. Stop when V-1 edges added
```
- **Complexity:** O(E log E) (dominated by sorting).
- Works well on **sparse** graphs; needs an **edge list** (Phase 2.2 representations).

### 2.2 Prim's Algorithm (vertex-based, uses a heap)
Grow the MST from a start vertex, repeatedly adding the **cheapest edge** connecting the tree to a new vertex (via a min-heap).
- **Complexity:** O((V + E) log V) with a heap.
- Works well on **dense** graphs; uses an **adjacency list**.

### 2.3 Kruskal vs Prim
| | Kruskal | Prim |
|---|---------|------|
| Approach | Sort edges, add if no cycle | Grow from a vertex (cheapest edge) |
| Data structure | Union-Find + edge list | Min-heap + adjacency list |
| Best for | Sparse graphs | Dense graphs |
| Complexity | O(E log E) | O((V+E) log V) |
> Both are **greedy** (Phase 2.3) and produce a valid MST. Choose by graph density and available representation.

---

## 3. Topological Sort (Ordering a DAG)

A **topological sort** linearly orders the vertices of a **DAG** (directed acyclic graph) so that for every edge u→v, **u comes before v**. It answers "what order can I do these tasks given dependencies?"
```
Dependencies: shoes need socks; jacket needs shirt
Valid topological order: socks, shoes, shirt, jacket
```

### 3.1 Kahn's Algorithm (BFS-based, using in-degrees)
```java
List<Integer> topoSort(Map<Integer, List<Integer>> adj, int n) {
    int[] inDegree = new int[n];
    for (var edges : adj.values()) for (int v : edges) inDegree[v]++;
    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < n; i++) if (inDegree[i] == 0) queue.offer(i);  // no dependencies
    List<Integer> order = new ArrayList<>();
    while (!queue.isEmpty()) {
        int node = queue.poll();
        order.add(node);
        for (int next : adj.getOrDefault(node, List.of()))
            if (--inDegree[next] == 0) queue.offer(next);   // dependency satisfied
    }
    return order.size() == n ? order : List.of();   // empty -> a CYCLE exists!
}
```
- Also done via **DFS** (postorder, reversed).
- **Complexity:** O(V + E).
- **Detects cycles:** if you can't order all vertices, there's a cycle.
> **Uses:** build systems (Maven/Gradle module order — Phase 3), task scheduling, course prerequisites, **Spring bean dependency ordering** (Phase 5.1), package managers. A cycle = a circular dependency error.

---

## 4. Union-Find (Disjoint Set Union — DSU)

**Union-Find** tracks a partition of elements into **disjoint sets** and answers "are these two in the same set?" in **near-O(1)** (amortized). It's the engine behind Kruskal's MST and cycle detection in undirected graphs.

### 4.1 The two operations
| Operation | Meaning |
|-----------|---------|
| **find(x)** | Which set does x belong to? (returns the set's representative/root) |
| **union(x, y)** | Merge the sets containing x and y |

### 4.2 Implementation with optimizations
```java
public class UnionFind {
    private int[] parent, rank;

    public UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;   // each element its own set
    }
    // find with PATH COMPRESSION — flattens the tree for future speed
    public int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);   // compress: point directly to root
        return parent[x];
    }
    // union by RANK — attach the smaller tree under the larger
    public boolean union(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return false;             // already connected (a cycle if adding this edge!)
        if (rank[rx] < rank[ry]) { int t = rx; rx = ry; ry = t; }
        parent[ry] = rx;
        if (rank[rx] == rank[ry]) rank[rx]++;
        return true;
    }
}
```
### 4.3 The two optimizations (crucial)
- **Path compression:** during `find`, point nodes directly to the root → flattens the tree.
- **Union by rank/size:** attach the shorter tree under the taller → keeps trees shallow.
- Together → amortized **O(α(n))** per operation (α = inverse Ackermann, effectively constant).
> **Uses:** Kruskal's MST, **cycle detection in undirected graphs** (if `union` returns false, the edge would form a cycle), **connected components**, "are these accounts/network nodes connected?", grouping/clustering.

---

## 5. Cycle Detection

Detecting cycles depends on whether the graph is directed:
| Graph | Method |
|-------|--------|
| **Undirected** | Union-Find (edge connects already-connected vertices → cycle), or DFS |
| **Directed** | DFS with a "recursion stack" (back edge → cycle), or Kahn's topo sort (can't order all → cycle) |
```java
// Directed cycle detection via DFS (3 colors / recursion stack):
boolean hasCycle(int node, Map<Integer,List<Integer>> adj, int[] state) {
    state[node] = 1;                          // 1 = in current path (gray)
    for (int next : adj.getOrDefault(node, List.of())) {
        if (state[next] == 1) return true;    // back edge -> CYCLE
        if (state[next] == 0 && hasCycle(next, adj, state)) return true;
    }
    state[node] = 2;                          // 2 = done (black)
    return false;
}
```
> **Uses:** detecting circular dependencies (Spring beans, Maven modules — Phase 3/5.1), deadlock detection, validating DAGs.

---

## 6. Strongly Connected Components (SCC)

In a **directed** graph, an **SCC** is a maximal set of vertices where **every vertex is reachable from every other** within the set. SCCs partition a digraph into clusters.
```
Algorithms: Tarjan's (one DFS pass) or Kosaraju's (two DFS passes).  Complexity: O(V + E).
```
> **Uses:** finding tightly-coupled clusters (modules that mutually depend — a refactoring smell), analyzing web link structure, detecting groups in directed networks. (Awareness-level; know the concept and that it's O(V+E).)

---

## 7. Algorithm Selection Cheat Sheet

| Problem | Algorithm |
|---------|-----------|
| Shortest path, unweighted | **BFS** |
| Shortest path, weighted (non-negative) | **Dijkstra** |
| Shortest path, negative edges | **Bellman-Ford** |
| All-pairs shortest paths | **Floyd-Warshall** |
| Cheapest connecting network (MST) | **Kruskal / Prim** |
| Order tasks with dependencies | **Topological sort** |
| Are X and Y connected? / cycle in undirected | **Union-Find** |
| Cycle in directed graph | **DFS (recursion stack)** / Kahn's |
| Clusters in a digraph | **SCC (Tarjan/Kosaraju)** |

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Dijkstra with negative edges | Use Bellman-Ford |
| Forgetting Union-Find optimizations | Add path compression + union by rank |
| Topo sort on a graph with cycles | It's only for DAGs (detects cycles by failing) |
| Confusing directed vs undirected cycle detection | Different methods (recursion stack vs Union-Find) |
| Floyd-Warshall on large graphs | O(V³) — only for small graphs |
| Wrong representation for the algorithm | Kruskal needs edge list; Prim/Dijkstra need adjacency list |
| Not handling disconnected graphs | Run from all unvisited starts |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Topological sort** = dependency resolution: **Maven/Gradle build order** (Phase 3), **Spring bean initialization order** (Phase 5.1), task scheduling (Phase 12).
- **Cycle detection** = circular dependency errors (Spring beans fail fast on cycles; Maven detects dependency cycles).
- **Dijkstra/shortest path** = routing, network latency optimization, recommendation distances.
- **Union-Find** = connectivity/clustering ("are these users/accounts related?"), fraud rings.
- **Graph algorithms at scale** = distributed graph processing, social/recommendation systems.
- **Microservice dependency graphs** (Phase 12) — ordering, cycle detection, blast-radius analysis.

---

## 10. Quick Self-Check Questions

1. Which shortest-path algorithm for: unweighted? non-negative weighted? negative edges? all-pairs?
2. Why does Dijkstra fail with negative edges?
3. What is an MST, and how do Kruskal's and Prim's differ?
4. What is a topological sort, and what does it require? How does it detect cycles?
5. What are Union-Find's two operations and two key optimizations?
6. How does Union-Find detect a cycle in an undirected graph?
7. How does directed cycle detection differ from undirected?
8. What is a strongly connected component?

---

## 11. Key Terms Glossary

- **Dijkstra / Bellman-Ford / Floyd-Warshall:** single-source non-negative / negative-edge / all-pairs shortest paths.
- **Relaxation:** updating a shorter known distance to a vertex.
- **MST (minimum spanning tree):** min-weight cycle-free spanning subgraph.
- **Kruskal / Prim:** edge-based (Union-Find) / vertex-based (heap) MST.
- **Topological sort:** dependency ordering of a DAG.
- **Kahn's algorithm:** BFS-based topo sort via in-degrees.
- **Union-Find (DSU):** disjoint-set structure (find/union).
- **Path compression / union by rank:** Union-Find optimizations (near-O(1)).
- **Cycle detection:** finding loops (different for directed/undirected).
- **SCC:** strongly connected component (mutually reachable cluster).

---

*Previous topic: **Graphs: Representation & Traversals**.*
*Next topic: **Bloom Filter & LRU Cache**.*
