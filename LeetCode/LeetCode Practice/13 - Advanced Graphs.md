---
title: "LeetCode Practice: Advanced Graphs"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - graphs
  - topological-sort
  - union-find
  - dijkstra
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Advanced Graphs

12 problems · Topological sort, Union-Find, Dijkstra, Bellman-Ford, MST.

> [!abstract] Pattern recap
> **Topological Sort** — linear order of DAG nodes such that every edge `u → v` means `u` comes before `v`. Two methods: **Kahn's** (BFS + indegrees) and **DFS post-order reverse**. Detects cycles for free.
> **Union-Find (DSU)** — dynamic connectivity. `find(x)` (with path compression) and `union(a, b)` (with rank). Near-O(1) per op.
> **Dijkstra** — shortest path in **non-negative-weighted** graph. Min-heap.
> **Bellman-Ford / SPFA** — handles negatives. O(V·E).
> **MST** — Kruskal (sort edges + DSU) or Prim (Dijkstra-like with min-heap).

> [!tip] Decision tree
> Unweighted shortest path → BFS.
> Non-negative weighted → Dijkstra.
> Negative weights or hop-limited → Bellman-Ford.
> Dynamic connectivity / # components → Union-Find.
> Ordering with prereqs → Topo sort.
> All-pairs shortest → Floyd-Warshall (rare in interviews).

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Course Schedule | 207 | Med | Topo sort (cycle check) |
| P2 | Course Schedule II | 210 | Med | Topo sort (order) |
| P3 | Alien Dictionary | 269 | Hard | Topo from constraints |
| P4 | Minimum Height Trees | 310 | Med | Peeling leaves |
| P5 | Network Delay Time | 743 | Med | Dijkstra |
| P6 | Path with Min Effort | 1631 | Med | Dijkstra on grid |
| P7 | Swim in Rising Water | 778 | Hard | Dijkstra (min-max) |
| P8 | Cheapest Flights Within K Stops | 787 | Med | Bellman-Ford (k+1 rounds) |
| P9 | Number of Connected Components | 323 | Med | Union-Find |
| P10 | Graph Valid Tree | 261 | Med | Union-Find (no cycle + connected) |
| P11 | Redundant Connection | 684 | Med | Union-Find (find cycle edge) |
| P12 | Min Cost to Connect All Points | 1584 | Med | MST (Prim or Kruskal) |

---

## P1: Course Schedule

**LC #207** · Medium

Can you finish all courses given prerequisites? = Is the prereq graph a DAG?

### 🧠 Pattern: Topological Sort via Kahn's Algorithm

> Build adjacency + in-degree. Queue all nodes with `indeg == 0`. Pop one, decrement neighbors' indeg, enqueue when they hit 0. If all nodes popped → no cycle. Else → cycle.

### Trace

```
n=4, prereqs=[[1,0],[2,1],[3,2]]
graph: 0→1, 1→2, 2→3
indeg: 0:0, 1:1, 2:1, 3:1

queue=[0]   pop 0, indeg[1]=0 → queue=[1]
            pop 1, indeg[2]=0 → queue=[2]
            pop 2, indeg[3]=0 → queue=[3]
            pop 3 → done. all 4 popped → no cycle → true
```

> [!success]- Python
> ```python
> from collections import deque
> def can_finish(num_courses, prerequisites):
>     graph = [[] for _ in range(num_courses)]
>     indeg = [0] * num_courses
>     for a, b in prerequisites:
>         graph[b].append(a)
>         indeg[a] += 1
>     q = deque(i for i in range(num_courses) if indeg[i] == 0)
>     done = 0
>     while q:
>         u = q.popleft()
>         done += 1
>         for v in graph[u]:
>             indeg[v] -= 1
>             if indeg[v] == 0:
>                 q.append(v)
>     return done == num_courses
> ```

**Variants:** DFS coloring approach (same as [[12 - Graphs]] P16).

**Key takeaway:** "Can we order with prereqs?" = "Is graph a DAG?" → topo sort. Kahn's is BFS + in-degree.

---

## P2: Course Schedule II

**LC #210** · Medium

Return one valid order (empty if impossible).

### Approach

Same as P1 but record the popped order.

> [!success]- Python
> ```python
> from collections import deque
> def find_order(num_courses, prerequisites):
>     graph = [[] for _ in range(num_courses)]
>     indeg = [0] * num_courses
>     for a, b in prerequisites:
>         graph[b].append(a)
>         indeg[a] += 1
>     q = deque(i for i in range(num_courses) if indeg[i] == 0)
>     out = []
>     while q:
>         u = q.popleft()
>         out.append(u)
>         for v in graph[u]:
>             indeg[v] -= 1
>             if indeg[v] == 0:
>                 q.append(v)
>     return out if len(out) == num_courses else []
> ```

**Key takeaway:** Topo sort output = the BFS order itself.

---

## P3: Alien Dictionary

**LC #269** · Hard

Given a sorted list of words in an unknown alphabet, return the character order (or "" if invalid).

### Approach

For each adjacent pair, find first differing chars → edge `a → b`. Topo sort. Validate "prefix anomaly" (e.g., `["abc", "ab"]` is invalid).

> [!success]- Python
> ```python
> from collections import deque, defaultdict
> def alien_order(words):
>     graph = defaultdict(set)
>     indeg = {c: 0 for w in words for c in w}
>     for a, b in zip(words, words[1:]):
>         minlen = min(len(a), len(b))
>         if a[:minlen] == b[:minlen] and len(a) > len(b):
>             return ""  # invalid: prefix anomaly
>         for i in range(minlen):
>             if a[i] != b[i]:
>                 if b[i] not in graph[a[i]]:
>                     graph[a[i]].add(b[i])
>                     indeg[b[i]] += 1
>                 break
>     q = deque([c for c in indeg if indeg[c] == 0])
>     out = []
>     while q:
>         c = q.popleft()
>         out.append(c)
>         for nb in graph[c]:
>             indeg[nb] -= 1
>             if indeg[nb] == 0:
>                 q.append(nb)
>     return ''.join(out) if len(out) == len(indeg) else ""
> ```

> [!warning] Don't forget single-letter constraints
> Every distinct character should appear in `indeg` even if it has no inbound/outbound edges.

**Key takeaway:** Extract constraints from data → topo sort. Watch for "prefix anomaly" edge case.

---

## P4: Minimum Height Trees

**LC #310** · Medium

Find roots that minimize tree height.

### 🧠 Pattern: Peel Leaves Layer by Layer

> Trees have ≤ 2 centroids. Repeatedly trim degree-1 nodes; the last 1 or 2 remaining are the answer.

> [!success]- Python
> ```python
> from collections import deque, defaultdict
> def find_min_height_trees(n, edges):
>     if n == 1: return [0]
>     graph = defaultdict(set)
>     for a, b in edges:
>         graph[a].add(b)
>         graph[b].add(a)
>     leaves = deque(i for i in range(n) if len(graph[i]) == 1)
>     remaining = n
>     while remaining > 2:
>         size = len(leaves)
>         remaining -= size
>         for _ in range(size):
>             leaf = leaves.popleft()
>             nb = graph[leaf].pop()
>             graph[nb].remove(leaf)
>             if len(graph[nb]) == 1:
>                 leaves.append(nb)
>     return list(leaves)
> ```

**Key takeaway:** Tree centroid = peel leaves. ≤ 2 survive.

---

## P5: Network Delay Time

**LC #743** · Medium · Dijkstra

Min time for signal from `k` to reach all nodes.

### 🧠 Pattern: Dijkstra's Algorithm

> Min-heap of `(dist, node)`. Pop smallest, relax neighbors. Skip if already finalized.

> [!success]- Python
> ```python
> import heapq
> from collections import defaultdict
> def network_delay_time(times, n, k):
>     graph = defaultdict(list)
>     for u, v, w in times:
>         graph[u].append((v, w))
>     dist = {}
>     h = [(0, k)]
>     while h:
>         d, u = heapq.heappop(h)
>         if u in dist: continue
>         dist[u] = d
>         for v, w in graph[u]:
>             if v not in dist:
>                 heapq.heappush(h, (d + w, v))
>     return max(dist.values()) if len(dist) == n else -1
> ```

> [!tip] Dijkstra ≠ BFS
> BFS implicitly assumes weight=1. Dijkstra generalizes to non-negative weights by always popping the smallest distance.

**Key takeaway:** Non-negative weighted shortest path → Dijkstra (PQ-based BFS).

---

## P6: Path With Minimum Effort

**LC #1631** · Medium

Grid; cost of a path = max abs diff between adjacent cells. Min cost from top-left to bottom-right.

### 🧠 Pattern: Dijkstra Where "Distance" = Path's Maximum Edge

> Replace `+` with `max` in relaxation: new_dist = `max(dist[u], |h[u] - h[v]|)`. Same PQ skeleton.

> [!success]- Python
> ```python
> import heapq
> def minimum_effort_path(heights):
>     R, C = len(heights), len(heights[0])
>     INF = float('inf')
>     dist = [[INF]*C for _ in range(R)]
>     dist[0][0] = 0
>     h = [(0, 0, 0)]
>     while h:
>         d, r, c = heapq.heappop(h)
>         if (r, c) == (R-1, C-1): return d
>         if d > dist[r][c]: continue
>         for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
>             nr, nc = r+dr, c+dc
>             if 0 <= nr < R and 0 <= nc < C:
>                 nd = max(d, abs(heights[nr][nc] - heights[r][c]))
>                 if nd < dist[nr][nc]:
>                     dist[nr][nc] = nd
>                     heapq.heappush(h, (nd, nr, nc))
>     return 0
> ```

**Key takeaway:** Dijkstra's relaxation is parameterizable. Replace `dist + w` with any monotone "cost combiner" like `max`.

---

## P7: Swim in Rising Water

**LC #778** · Hard

Grid; cost = max cell-height along path. Min cost from top-left to bottom-right.

### Approach

Same as P6 — Dijkstra with `max` as the combiner. Or binary-search on answer (cleaner).

> [!success]- Python
> ```python
> import heapq
> def swim_in_water(grid):
>     N = len(grid)
>     h = [(grid[0][0], 0, 0)]
>     seen = {(0, 0)}
>     while h:
>         t, r, c = heapq.heappop(h)
>         if (r, c) == (N-1, N-1): return t
>         for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
>             nr, nc = r+dr, c+dc
>             if 0 <= nr < N and 0 <= nc < N and (nr, nc) not in seen:
>                 seen.add((nr, nc))
>                 heapq.heappush(h, (max(t, grid[nr][nc]), nr, nc))
> ```

**Key takeaway:** Cousin of P6. Whenever cost = "max along path", Dijkstra still works with `max`.

---

## P8: Cheapest Flights Within K Stops

**LC #787** · Medium

### 🧠 Pattern: Bellman-Ford with K+1 Relaxation Rounds

> Each Bellman-Ford round captures one additional hop. Run K+1 rounds.
> 
> **Why not Dijkstra?** Dijkstra finalizes nodes; can miss the right answer when the cheapest path uses more hops than the local optimum.

> [!success]- Python
> ```python
> def find_cheapest_price(n, flights, src, dst, k):
>     INF = float('inf')
>     prices = [INF] * n
>     prices[src] = 0
>     for _ in range(k + 1):
>         snapshot = prices[:]
>         for u, v, w in flights:
>             if snapshot[u] + w < prices[v]:
>                 prices[v] = snapshot[u] + w
>     return -1 if prices[dst] == INF else prices[dst]
> ```

> [!tip] Snapshot per round
> Use the *previous* round's prices for relaxation; otherwise you'd allow multi-hop progress within a single round.

**Key takeaway:** Hop-limited shortest path → Bellman-Ford bounded to k+1 rounds.

---

## P9: Number of Connected Components in an Undirected Graph

**LC #323** · Medium

### 🧠 Pattern: Union-Find (DSU)

> Start with n separate components. Each edge → union. Final count = remaining distinct roots.

> [!success]- Python
> ```python
> def count_components(n, edges):
>     parent = list(range(n))
>     def find(x):
>         while parent[x] != x:
>             parent[x] = parent[parent[x]]   # path compression
>             x = parent[x]
>         return x
>     def union(a, b):
>         ra, rb = find(a), find(b)
>         if ra == rb: return False
>         parent[ra] = rb
>         return True
>     count = n
>     for a, b in edges:
>         if union(a, b):
>             count -= 1
>     return count
> ```

**Key takeaway:** DSU template. **Path compression** in find, **decrement count** only on successful union.

---

## P10: Graph Valid Tree

**LC #261** · Medium

Tree iff: connected AND no cycle. With n nodes, exactly n-1 edges + no cycle ⇒ tree.

### Approach

DSU: every union should connect two different components. Cycle = unioning already-connected nodes.

> [!success]- Python
> ```python
> def valid_tree(n, edges):
>     if len(edges) != n - 1: return False
>     parent = list(range(n))
>     def find(x):
>         while parent[x] != x:
>             parent[x] = parent[parent[x]]
>             x = parent[x]
>         return x
>     for a, b in edges:
>         ra, rb = find(a), find(b)
>         if ra == rb: return False   # cycle
>         parent[ra] = rb
>     return True
> ```

**Key takeaway:** Tree validation = "n-1 edges AND no cycle". DSU detects the cycle cleanly.

---

## P11: Redundant Connection

**LC #684** · Medium

Find the extra edge that creates a cycle. Return the one that appears **last** in the input.

### Approach

DSU: process edges in order. The first edge that connects two already-connected nodes is the answer.

> [!success]- Python
> ```python
> def find_redundant_connection(edges):
>     n = len(edges)
>     parent = list(range(n + 1))
>     def find(x):
>         while parent[x] != x:
>             parent[x] = parent[parent[x]]
>             x = parent[x]
>         return x
>     for a, b in edges:
>         ra, rb = find(a), find(b)
>         if ra == rb: return [a, b]
>         parent[ra] = rb
> ```

**Key takeaway:** "Find the redundant edge" → DSU sweep; first cycle-creating edge wins.

---

## P12: Min Cost to Connect All Points

**LC #1584** · Medium · MST

Connect all points; cost = Manhattan distance. Min total cost.

### 🧠 Pattern: Minimum Spanning Tree (Prim or Kruskal)

> Prim's: grow MST one node at a time using min-heap.
> Kruskal's: sort all edges, union-find to add if doesn't cycle.

> [!success]- Python (Prim)
> ```python
> import heapq
> def min_cost_connect_points(points):
>     n = len(points)
>     in_mst = [False] * n
>     h = [(0, 0)]
>     total = 0
>     count = 0
>     while count < n:
>         d, u = heapq.heappop(h)
>         if in_mst[u]: continue
>         in_mst[u] = True
>         total += d
>         count += 1
>         x1, y1 = points[u]
>         for v in range(n):
>             if not in_mst[v]:
>                 x2, y2 = points[v]
>                 heapq.heappush(h, (abs(x1-x2) + abs(y1-y2), v))
>     return total
> ```

> [!success]- Python (Kruskal)
> ```python
> def min_cost_connect_points_kruskal(points):
>     n = len(points)
>     edges = []
>     for i in range(n):
>         for j in range(i + 1, n):
>             d = abs(points[i][0]-points[j][0]) + abs(points[i][1]-points[j][1])
>             edges.append((d, i, j))
>     edges.sort()
>     parent = list(range(n))
>     def find(x):
>         while parent[x] != x:
>             parent[x] = parent[parent[x]]
>             x = parent[x]
>         return x
>     total = count = 0
>     for d, a, b in edges:
>         ra, rb = find(a), find(b)
>         if ra == rb: continue
>         parent[ra] = rb
>         total += d
>         count += 1
>         if count == n - 1: break
>     return total
> ```

**Key takeaway:** MST: Prim for dense graphs, Kruskal for sparse with explicit edges. Both heavily reuse skills from earlier topics.

---

> [!tip] After this drill
> Build a tiny decision tree in your head: "Weighted shortest path? Non-neg → Dijkstra; negatives → Bellman-Ford. Connectivity → DSU. Order with deps → topo. MST → Prim/Kruskal."
