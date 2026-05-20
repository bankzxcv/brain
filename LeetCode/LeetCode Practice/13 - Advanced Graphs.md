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

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Course Schedule | 207 | Med | Topo sort (cycle check) |
| P2 | Course Schedule II | 210 | Med | Topo sort (order) |
| P3 | Alien Dictionary | 269 | **Hard** | Topo from constraints |
| P4 | Minimum Height Trees | 310 | Med | Peeling leaves |
| P5 | Network Delay Time | 743 | Med | Dijkstra |
| P6 | Path with Min Effort | 1631 | Med | Dijkstra on grid |
| P7 | Swim in Rising Water | 778 | **Hard** | Dijkstra (min-max) |
| P8 | Cheapest Flights Within K Stops | 787 | Med | Bellman-Ford (k+1 rounds) |
| P9 | Number of Connected Components | 323 | Med | Union-Find |
| P10 | Graph Valid Tree | 261 | Med | Union-Find (no cycle + connected) |
| P11 | Redundant Connection | 684 | Med | Union-Find (find cycle edge) |
| P12 | Min Cost to Connect All Points | 1584 | Med | MST (Prim or Kruskal) |

---

## P1: Course Schedule

**LC #207** · Medium

Can you finish all courses given prerequisites?

### 🧠 Pattern: Topological Sort via Kahn's Algorithm

> Build adjacency + in-degree. Queue all nodes with `indeg == 0`. Pop one, decrement neighbors' indeg, enqueue when they hit 0. If all nodes popped → no cycle. Else → cycle.

> [!example]- 📊 Visual: Kahn's algorithm on a DAG
> ```text
>   Edges (prereq → course):  0 → 1 → 2 → 3
> 
>   Indegree per node:
>     [0]  [1]  [2]  [3]
>      0    1    1    1
> 
>   Kahn's pops indeg==0 nodes in waves:
> 
>     wave 1:   q=[0]   pop 0 → indeg[1]=0 → enq 1
>     wave 2:   q=[1]   pop 1 → indeg[2]=0 → enq 2
>     wave 3:   q=[2]   pop 2 → indeg[3]=0 → enq 3
>     wave 4:   q=[3]   pop 3
> 
>   Order popped: [0, 1, 2, 3]   (one valid linearization)
> 
>   ─────────────────────────────────────────
>   Cycle case: edges = 0 → 1, 1 → 0
> 
>             ┌────→─────┐
>           (0)         (1)
>             └──←──────┘
> 
>     indeg = [1, 1]  — no zero-indeg node to start!
>     queue stays empty → done < n → cycle detected → return false
> ```

> [!info]- 🔍 Dry Run: numCourses=4, prerequisites=[[1,0],[2,1],[3,2]]
> ```text
> Edge meaning: [a, b] means must take b before a → edge b → a
> 
> Build graph:
>   0 → 1
>   1 → 2
>   2 → 3
>   indeg = [0, 1, 1, 1]
> 
> ─────────────────────────────────────────
> Init queue with indeg==0: q = [0]
> done = 0
> 
> Pop 0: done=1
>   for v in graph[0]=[1]:
>     indeg[1] -= 1 → 0
>     0 == 0 → q.append(1)
>   q = [1]
> 
> Pop 1: done=2
>   for v in graph[1]=[2]:
>     indeg[2] -= 1 → 0
>     q.append(2)
>   q = [2]
> 
> Pop 2: done=3
>   for v in graph[2]=[3]:
>     indeg[3] -= 1 → 0
>     q.append(3)
>   q = [3]
> 
> Pop 3: done=4. No outgoing edges.
> 
> done==4==numCourses → no cycle → return true
> 
> ✅ Answer: true
> 
> ─────────────────────────────────────────
> Counter-example: prerequisites=[[1,0],[0,1]] → cycle 0↔1
>   graph: 0→1, 1→0; indeg = [1, 1]
>   q = []  (no indeg==0 nodes)
>   loop doesn't execute; done=0 ≠ 2 → return false
> ```

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

**Key takeaway:** "Can we order with prereqs?" = "Is graph a DAG?" → topo sort. Kahn's is BFS + in-degree.

---

## P2: Course Schedule II

**LC #210** · Medium

Return one valid order (empty if impossible).

> [!example]- 📊 Visual: branching DAG with shared sink
> ```text
>   Edges:                 DAG:           Indegrees:
>     0 → 1                       (0)           0: 0
>     0 → 2                      /   \          1: 1
>     1 → 3                   (1)   (2)         2: 1
>     2 → 3                      \   /          3: 2
>                                 (3)
> 
>   Kahn's waves:
> 
>     wave 1:   q=[0]            pop 0   → indeg[1]=0, indeg[2]=0
>                                          enq 1, enq 2
>     wave 2:   q=[1, 2]         pop 1   → indeg[3]=1 (not 0)
>                                pop 2   → indeg[3]=0 → enq 3
>     wave 3:   q=[3]            pop 3
> 
>   Output: [0, 1, 2, 3]
> 
>   ─────────────────────────────────────────
>   Note: order within a wave depends on queue order.
>   [0, 2, 1, 3] is also valid (different push order).
> ```

> [!info]- 🔍 Dry Run: numCourses=4, prerequisites=[[1,0],[2,0],[3,1],[3,2]]
> ```text
> Edges:
>   0 → 1
>   0 → 2
>   1 → 3
>   2 → 3
>   indeg = [0, 1, 1, 2]
> 
> q = [0], out = []
> 
> Pop 0: out=[0]
>   indeg[1]-=1 → 0; enq 1
>   indeg[2]-=1 → 0; enq 2
>   q = [1, 2]
> 
> Pop 1: out=[0,1]
>   indeg[3]-=1 → 1 (not 0, don't enq)
>   q = [2]
> 
> Pop 2: out=[0,1,2]
>   indeg[3]-=1 → 0; enq 3
>   q = [3]
> 
> Pop 3: out=[0,1,2,3]
>   no outgoing
> 
> len(out)=4 == numCourses → return out
> 
> ✅ Answer: [0, 1, 2, 3]
>   (Other valid orders exist; queue order determines which we get.)
> ```

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

**LC #269** · **Hard**

> [!example]- 📊 Visual: extract ordering edges from adjacent word pairs
> ```text
>   words = ["wrt", "wrf", "er", "ett", "rftt"]
> 
>   Compare adjacent pairs, find FIRST differing char:
> 
>     wrt vs wrf      → t comes before f       edge: t → f
>     wrf vs er       → w comes before e       edge: w → e
>     er  vs ett      → r comes before t       edge: r → t
>     ett vs rftt     → e comes before r       edge: e → r
> 
>   Constraint DAG:
> 
>         (w) → (e) → (r) → (t) → (f)
> 
>   Indegrees: w=0, e=1, r=1, t=1, f=1
> 
>   Kahn's pops in order: w, e, r, t, f
> 
>   Answer: "wertf"
> 
>   ─────────────────────────────────────────
>   Prefix anomaly: words = ["abc", "ab"]
>     "ab" is a prefix of "abc" but appears LATER → impossible → ""
> ```

> [!info]- 🔍 Dry Run: words=["wrt","wrf","er","ett","rftt"]
> ```text
> Find first differing char between adjacent pairs:
>   wrt vs wrf: first diff at index 2: t vs f → edge t → f
>   wrf vs er:  index 0: w vs e → edge w → e
>   er vs ett:  index 1: r vs t → edge r → t
>   ett vs rftt: index 0: e vs r → edge e → r
> 
> Collect all chars: {w, r, t, f, e}
> 
> Build graph + indeg:
>   t → f       indeg[f]=1
>   w → e       indeg[e]=1
>   r → t       indeg[t]=1
>   e → r       indeg[r]=1
>   indeg[w]=0
> 
> Init q = [w]   (only w has indeg 0)
> out = []
> 
> Pop w: out=[w]. graph[w]=[e]. indeg[e]-=1 → 0. enq e.
> Pop e: out=[w,e]. graph[e]=[r]. indeg[r]-=1 → 0. enq r.
> Pop r: out=[w,e,r]. graph[r]=[t]. indeg[t]-=1 → 0. enq t.
> Pop t: out=[w,e,r,t]. graph[t]=[f]. indeg[f]-=1 → 0. enq f.
> Pop f: out=[w,e,r,t,f].
> 
> All 5 chars processed → return "wertf"
> 
> ✅ Answer: "wertf"
> 
> ─────────────────────────────────────────
> Edge case prefix anomaly: words=["abc","ab"]
>   minlen=2, a[:2]="ab"==b[:2]="ab" AND len(a) > len(b) → return ""
> ```

> [!success]- Python
> ```python
> from collections import deque, defaultdict
> def alien_order(words):
>     graph = defaultdict(set)
>     indeg = {c: 0 for w in words for c in w}
>     for a, b in zip(words, words[1:]):
>         minlen = min(len(a), len(b))
>         if a[:minlen] == b[:minlen] and len(a) > len(b):
>             return ""
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

**Key takeaway:** Extract constraints from data → topo sort. Watch for "prefix anomaly" edge case.

---

## P4: Minimum Height Trees

**LC #310** · Medium

> [!example]- 📊 Visual: peel leaves to find tree centroid(s)
> ```text
>   Tree (n=6):
>                (0)
>                 │
>          (1)───(3)───(2)
>                 │
>                (4)
>                 │
>                (5)
> 
>   Round 0:  leaves (deg=1) = {0, 1, 2, 5}    remaining = 6
> 
>   Peel round 1: remove {0, 1, 2, 5}
>                                    │
>                                   (3)───(4)         ← new leaves
>                                    │
>                                  
>             remaining = 6 - 4 = 2 → stop (≤ 2 left)
> 
>   Centroids: {3, 4}   ← roots that minimize tree height
> 
>   ─────────────────────────────────────────
>   Intuition: longest path in tree has ≤2 midpoints. Those are
>   the centers. Peeling layer-by-layer converges on them.
> ```

> [!info]- 🔍 Dry Run: n=6, edges=[[0,3],[1,3],[2,3],[4,3],[5,4]]
> ```text
> Graph (undirected):
>   3: {0,1,2,4}
>   0: {3}
>   1: {3}
>   2: {3}
>   4: {3,5}
>   5: {4}
> 
> Leaves (degree 1): 0, 1, 2, 5
> 
> remaining = 6
> leaves queue = [0, 1, 2, 5]
> 
> ─────────────────────────────────────────
> Round 1: size=4, remaining=6-4=2 (≤2, exit before processing this round)
>   Wait, condition is "while remaining > 2", so we process this round only if remaining was > 2 BEFORE decrement.
> 
> Let me re-read: while remaining > 2.
>   Initial remaining=6 > 2, enter loop:
>     size=4 (current leaves count)
>     remaining -= 4 → 2
>     for each leaf, remove from graph; if neighbor becomes degree 1, push:
>       leaf 0: graph[3] removes 0 → 3 still has {1,2,4} (deg 3)
>       leaf 1: graph[3] removes 1 → deg 2
>       leaf 2: graph[3] removes 2 → deg 1 → push 3
>       leaf 5: graph[4] removes 5 → 4 has deg 1 → push 4
>     leaves new queue = [3, 4]
>   remaining=2 == 2, exit loop.
> 
> Return list(leaves) = [3, 4]
> 
> ✅ Answer: [3, 4]
> ```

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

> [!example]- 📊 Visual: Dijkstra heap evolution
> ```text
>   Weighted graph (k=2 = source):
> 
>            (1)        (4)
>            ↑           ↑
>           1│           │1
>            │           │
>           (2) ──1──→ (3)
> 
>   Dijkstra from node 2:
> 
>     step │ heap (sorted)        │ pop      │ dist updates
>     ─────┼──────────────────────┼──────────┼────────────────
>      0   │ [(0,2)]               │ (0, 2)   │ dist[2]=0
>      1   │ [(1,1), (1,3)]        │ (1, 1)   │ dist[1]=1
>      2   │ [(1,3)]               │ (1, 3)   │ dist[3]=1
>      3   │ [(2,4)]               │ (2, 4)   │ dist[4]=2
> 
>   Final dist table:
>     dist[1]=1   dist[2]=0   dist[3]=1   dist[4]=2
> 
>   Answer = max(dist values) = 2   (time for signal to reach all)
> 
>   "Lazy deletion": when popping a node already in dist, skip it.
> ```

> [!info]- 🔍 Dry Run: times=[[2,1,1],[2,3,1],[3,4,1]], n=4, k=2
> ```text
> Graph: 2 → 1 (w=1), 2 → 3 (w=1), 3 → 4 (w=1)
> 
> dist = {}
> h = [(0, 2)]
> 
> ─────────────────────────────────────────
> Pop (0, 2): dist[2] = 0
>   for (1, 1): not in dist → push (0+1, 1)
>   for (3, 1): not in dist → push (0+1, 3)
>   h = [(1,1), (1,3)]
> 
> Pop (1, 1): dist[1] = 1
>   no outgoing
> 
> Pop (1, 3): dist[3] = 1
>   for (4, 1): push (2, 4)
>   h = [(2,4)]
> 
> Pop (2, 4): dist[4] = 2
> 
> All 4 nodes reached.
> Result = max(dist.values()) = 2
> 
> ✅ Answer: 2
> ```

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

**Key takeaway:** Non-negative weighted shortest path → Dijkstra (PQ-based BFS).

---

## P6: Path With Minimum Effort

**LC #1631** · Medium

### 🧠 Pattern: Dijkstra Where "Distance" = Path's Maximum Edge

> [!example]- 📊 Visual: Dijkstra with max-combiner on a grid
> ```text
>   heights:                      Best "effort" to reach each cell:
>     1  2  2                       0  1  1
>     3  8  2                       2  6  1     (initial relaxation)
>     5  3  5                       ∞  ∞  ∞
> 
>   "Effort" of a path = MAX |diff| of any step on path.
>   Dijkstra with cost combiner: nd = max(d_so_far, |h_next - h_here|)
> 
>   Heap snapshot for the bottom-right corner:
> 
>     pop (0, 0,0)  effort 0
>     push (2,1,0), (1,0,1)
>     pop (1, 0,1)  push (1,0,2), (6,1,1)
>     pop (1, 0,2)  push (1,1,2)
>     pop (1, 1,2)  push (3,2,2)
>     pop (2, 1,0)  push (2,2,0)
>     pop (2, 2,0)  push (2,2,1)
>     pop (2, 2,1)  push (2,2,2)   ← better than 3 → updates dist[2,2]=2
>     pop (2, 2,2)  TARGET → return 2
> 
>   Answer: 2   (path 1→2→2→2→3 with max diff = 2 at the last step)
> ```

> [!info]- 🔍 Dry Run: heights=[[1,2,2],[3,8,2],[5,3,5]]
> ```text
> dist[][] init INF; dist[0][0]=0
> h = [(0, 0, 0)]
> 
> Pop (0, 0, 0):
>   neighbors:
>     (1,0): |3-1|=2 → nd = max(0, 2) = 2; push (2, 1, 0). dist[1][0]=2
>     (0,1): |2-1|=1 → nd = max(0, 1) = 1; push (1, 0, 1). dist[0][1]=1
> 
> Pop (1, 0, 1):
>   (0,0) seen-equivalent (dist 0, smaller); skip
>   (0,2): |2-2|=0 → max(1, 0) = 1; dist[0][2]=1; push (1, 0, 2)
>   (1,1): |8-2|=6 → max(1, 6) = 6; dist[1][1]=6; push (6, 1, 1)
> 
> Pop (1, 0, 2):
>   (1,2): |2-2|=0 → max(1, 0) = 1; dist[1][2]=1; push
> 
> Pop (1, 1, 2):  same effort 1
>   (2,2): |5-2|=3 → max(1, 3) = 3; dist[2][2]=3; push (3, 2, 2)
> 
> Pop (2, 1, 0):
>   (2,0): |5-3|=2 → max(2, 2) = 2; dist[2][0]=2; push
> 
> Pop (2, 2, 0):
>   (2,1): |3-5|=2 → max(2, 2) = 2; dist[2][1]=2; push
> 
> Pop (2, 2, 1): updated dist[2][2] possibly? dist[2][2] was 3.
>   (2,2): |5-3|=2 → max(2, 2) = 2 < 3 → update dist[2][2]=2; push (2, 2, 2)
> 
> Pop (2, 2, 2): target! return 2.
> 
> ✅ Answer: 2
> ```

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

**LC #778** · **Hard**

### Approach

Same as P6 — Dijkstra with `max` as the combiner.

> [!example]- 📊 Visual: rising water reveals lowest "tide" to traverse
> ```text
>   grid (elevation = time-to-submerge):
>     0  2
>     1  3
> 
>   You can step from a cell to neighbor when water level ≥ max(both elevs).
>   We want the MIN time t such that a path 0,0 → N-1,N-1 exists.
> 
>   Dijkstra with t = max(t_so_far, grid[next]):
> 
>     start (0,0,t=0)
>       ↓               ↘
>     (1,0,t=1)        (0,1,t=2)
>       ↓                 ↓
>     (1,1,t=3)        (1,1,t=3)
> 
>   Heap order of finalization:  (0,0)→(1,0)→(0,1)→(1,1) at t=3
> 
>   ─────────────────────────────────────────
>   Mental model — water level rises 0,1,2,3,...
>     t=0:  ▓ . . .       only (0,0) reachable
>     t=1:  ▓ ▓ . .       (1,0) joins
>     t=2:  ▓ ▓ ▓ .       (0,1) joins
>     t=3:  ▓ ▓ ▓ ▓       target reachable → answer = 3
> ```

> [!info]- 🔍 Dry Run: grid=[[0,2],[1,3]]
> ```text
> Start at (0,0) elevation 0. Target (1,1).
> 
> h = [(0, 0, 0)]
> seen = {(0,0)}
> 
> Pop (0, 0, 0): not target
>   neighbors:
>     (1,0): elev=1; cost = max(0, 1) = 1; push (1, 1, 0); seen add
>     (0,1): elev=2; cost = max(0, 2) = 2; push (2, 0, 1); seen add
> 
> Pop (1, 1, 0): not target
>   (1,1): elev=3; cost = max(1, 3) = 3; push (3, 1, 1); seen add
> 
> Pop (2, 0, 1): not target
>   (1,1): in seen
> 
> Pop (3, 1, 1): target → return 3
> 
> ✅ Answer: 3
> ```

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

> [!example]- 📊 Visual: Bellman-Ford rounds with snapshot
> ```text
>   Graph:
>         100        100
>     (0) ────→ (1) ────→ (2)
>      │                    ↑
>      └──────500───────────┘
> 
>   k=1 stops means at most 2 edges allowed.
>   Run k+1 = 2 relaxation rounds.
> 
>   prices array evolution:
> 
>     init:           [ 0,  ∞,  ∞ ]
> 
>     ─── Round 1 ─── (snapshot = init)
>       edge 0→1 (100):  snap[0]+100 = 100 < ∞   → prices[1] = 100
>       edge 1→2 (100):  snap[1] = ∞              → no update
>       edge 0→2 (500):  snap[0]+500 = 500 < ∞   → prices[2] = 500
>     prices:         [ 0, 100, 500 ]
> 
>     ─── Round 2 ─── (snapshot = [0,100,500])
>       edge 0→1 (100):  snap[0]+100 = 100, not < 100  → no
>       edge 1→2 (100):  snap[1]+100 = 200 < 500       → prices[2] = 200
>       edge 0→2 (500):  no improvement
>     prices:         [ 0, 100, 200 ]
> 
>   Why snapshot? Without it, edge 0→1 in this round and edge 1→2
>   could BOTH apply in one round → 2 hops in 1 round → over-relaxed.
> 
>   Answer: prices[2] = 200
> ```

> [!info]- 🔍 Dry Run: n=3, flights=[[0,1,100],[1,2,100],[0,2,500]], src=0, dst=2, k=1
> ```text
> prices = [INF, INF, INF]
> prices[0] = 0
> 
> Round 1 (allow 1 hop): snapshot prices=[0, INF, INF]
>   edge (0,1,100): snapshot[0]+100=100 < prices[1] → prices[1]=100
>   edge (1,2,100): snapshot[1]=INF, can't use
>   edge (0,2,500): snapshot[0]+500=500 < prices[2] → prices[2]=500
>   prices = [0, 100, 500]
> 
> Round 2 (allow 2 hops): snapshot prices=[0, 100, 500]
>   edge (0,1,100): 0+100=100, not < prices[1]=100; no update
>   edge (1,2,100): 100+100=200 < prices[2]=500 → prices[2]=200
>   edge (0,2,500): no improvement
>   prices = [0, 100, 200]
> 
> Note: k=1 stops means at most 2 edges. We ran k+1=2 rounds.
> 
> Return prices[dst=2] = 200
> 
> ✅ Answer: 200
> ```

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

> [!example]- 📊 Visual: DSU forest evolution
> ```text
>   n=5, edges = [(0,1), (1,2), (3,4)]
> 
>   Initial forest (each its own root):
>     0   1   2   3   4
>     ●   ●   ●   ●   ●
> 
>   After union(0,1):  (point 0's root → 1)
>         1
>         │       2   3   4
>         0       ●   ●   ●
> 
>   After union(1,2):  (point 1's root → 2)
>             2
>             │           3   4
>             1           ●   ●
>             │
>             0
> 
>   After union(3,4):  (point 3's root → 4)
>             2               4
>             │               │
>             1               3
>             │
>             0
> 
>   Roots remaining: {2, 4} → 2 connected components ✓
> 
>   count starts at n=5, decrements on each successful union.
>   5 → 4 → 3 → 2 = answer.
> ```

> [!info]- 🔍 Dry Run: n=5, edges=[[0,1],[1,2],[3,4]]
> ```text
> parent = [0,1,2,3,4]   (each its own root)
> count = 5
> 
> ─────────────────────────────────────────
> edge (0,1):
>   find(0)=0, find(1)=1; different → union
>   parent[0] = 1 (or parent[1]=0; either way)
>   count = 4
> 
> edge (1,2):
>   find(1)=1 (parent[1]=1)
>   find(2)=2; different → union
>   parent[1] = 2
>   count = 3
> 
> edge (3,4):
>   find(3)=3, find(4)=4; different → union
>   parent[3] = 4
>   count = 2
> 
> Final: components {0,1,2} and {3,4}. count = 2.
> 
> ✅ Answer: 2
> ```

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

> [!example]- 📊 Visual: tree = connected + acyclic
> ```text
>   Valid tree: n=5, edges = (0,1),(0,2),(0,3),(1,4)
> 
>          (4)
>           │
>          (1)
>           │
>          (0) ── (2)
>           │
>          (3)
> 
>   Check: |edges| = 4 == n-1 = 4 ✓
>   DSU sweep — every union joins two DIFFERENT components → no cycle.
>   Result: true
> 
>   ─────────────────────────────────────────
>   Invalid (cycle): n=5, edges = (0,1),(1,2),(2,3),(1,3),(1,4)
>     |edges| = 5 ≠ n-1=4 → reject immediately
> 
>          (4)
>           │
>          (1) ── (2)
>          /\      │
>        (0) (3) ──┘     ← extra edge closes cycle 1-2-3-1
> 
>   ─────────────────────────────────────────
>   Invalid (disconnected): n=4, edges = (0,1),(2,3)
>     |edges|=2 ≠ n-1=3 → reject
> 
>     (0)─(1)    (2)─(3)    ← two components
> ```

> [!info]- 🔍 Dry Run: n=5, edges=[[0,1],[0,2],[0,3],[1,4]]
> ```text
> Check: len(edges)=4 == n-1=4 ✓
> 
> parent = [0,1,2,3,4]
> 
> (0,1): find=0,1 diff → parent[0]=1; ok
> (0,2): find(0): parent[0]=1, parent[1]=1, so find(0)=1. find(2)=2; union; parent[1]=2.
> (0,3): find(0)=2; find(3)=3; union; parent[2]=3
> (1,4): find(1)=3; find(4)=4; union; parent[3]=4
> 
> All edges processed without cycle → return true
> 
> ✅ Answer: true
> 
> ─────────────────────────────────────────
> Cycle example: n=5, edges=[[0,1],[1,2],[2,3],[1,3],[1,4]]
>   len(edges)=5, n-1=4 → 5 != 4 → return false immediately
> ```

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

> [!example]- 📊 Visual: DSU spots the cycle-closing edge
> ```text
>   edges = [(1,2), (1,3), (2,3)]
> 
>   Process edges in order; on each, find roots of both endpoints:
> 
>     ─── (1,2) ───              parents: 1→2
>       find(1)=1, find(2)=2     forest:    2
>       different → union          │
>                                  1     3
> 
>     ─── (1,3) ───              parents: 2→3
>       find(1)=2, find(3)=3     forest:        3
>       different → union                       │
>                                                2
>                                                │
>                                                1
> 
>     ─── (2,3) ───              find(2)=3, find(3)=3
>       SAME root!               → adding this edge would create a cycle
>                                → return [2, 3]   ✓
> 
>   The first edge that connects two nodes already in the SAME tree
>   is the redundant one.
> ```

> [!info]- 🔍 Dry Run: edges=[[1,2],[1,3],[2,3]]
> ```text
> n = 3, parent = [0,1,2,3]   (1-indexed for convenience)
> 
> edge (1,2): find=1,2 diff → union; parent[1]=2
> edge (1,3): find(1)=2; find(3)=3; diff → union; parent[2]=3
> edge (2,3): find(2)=3; find(3)=3; SAME → cycle detected → return [2,3]
> 
> ✅ Answer: [2, 3]   (the edge that closes the cycle)
> ```

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

> [!example]- 📊 Visual: Prim's MST grows one edge at a time
> ```text
>   Points:  0=(0,0)  1=(2,2)  2=(3,10)  3=(5,2)  4=(7,0)
> 
>   All pairwise Manhattan distances (relevant subset):
>     0─1: 4    0─3: 7    0─4: 7
>     1─2: 9    1─3: 3    1─4: 7
>     2─3: 10   3─4: 4
> 
>   Prim from node 0 (heap of candidate edges, pop cheapest):
> 
>     in_mst = {0}        total = 0
>       push edges from 0: (4,1)(13,2)(7,3)(7,4)
> 
>     pop (4,1) → MST adds 0═══1   total = 4
>       push edges from 1: (9,2)(3,3)(7,4)
> 
>     pop (3,3) → MST adds 1═══3   total = 7
>       push edges from 3: (10,2)(4,4)
> 
>     pop (4,4) → MST adds 3═══4   total = 11
>       push edges from 4: (14,2)
> 
>     pop (7,3) → already in MST, skip
>     pop (7,4) → already in MST, skip
>     pop (9,2) → MST adds 1═══2   total = 20
> 
>   Final MST (double lines):
> 
>     (2)
>      ║ 9
>     (1)═══4═══(0)
>      ║ 3
>     (3)═══4═══(4)
> 
>   Total cost = 20.
> ```

> [!info]- 🔍 Dry Run: points=[[0,0],[2,2],[3,10],[5,2],[7,0]]
> ```text
> Use Prim's algorithm:
>   in_mst = [F,F,F,F,F]
>   h = [(0, 0)]   (start at node 0 with cost 0)
>   total = 0, count = 0
> 
> ─────────────────────────────────────────
> Pop (0, 0): not in_mst → in_mst[0]=T; total=0; count=1
>   Add edges from 0 to all others:
>     d(0,1)=|0-2|+|0-2|=4 → push (4, 1)
>     d(0,2)=|0-3|+|0-10|=13 → push (13, 2)
>     d(0,3)=|0-5|+|0-2|=7 → push (7, 3)
>     d(0,4)=|0-7|+|0-0|=7 → push (7, 4)
> 
> Pop (4, 1): not in_mst → add. total=4, count=2
>   Add edges from 1:
>     d(1,2)=|2-3|+|2-10|=9 → push
>     d(1,3)=|2-5|+|2-2|=3 → push
>     d(1,4)=|2-7|+|2-0|=7 → push
> 
> Pop (3, 3): not in_mst → add. total=7, count=3
>   Add edges from 3:
>     d(3,2)=|5-3|+|2-10|=10
>     d(3,4)=|5-7|+|2-0|=4
> 
> Pop (4, 4): not in_mst → add. total=11, count=4
>   Add edges from 4:
>     d(4,2)=|7-3|+|0-10|=14
> 
> Pop (7, 3): in_mst → skip
> Pop (7, 4): in_mst → skip
> Pop (9, 2): not in_mst → add. total=20, count=5
> 
> ✅ Answer: 20
> ```

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

**Key takeaway:** MST: Prim for dense graphs, Kruskal for sparse with explicit edges.

---

> [!tip] After this drill
> Build a tiny decision tree in your head: "Weighted shortest path? Non-neg → Dijkstra; negatives → Bellman-Ford. Connectivity → DSU. Order with deps → topo. MST → Prim/Kruskal."
