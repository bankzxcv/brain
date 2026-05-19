---
title: "LeetCode Practice: Graphs"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - graphs
  - bfs
  - dfs
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Graphs

16 problems · BFS / DFS on grids and adjacency lists.

> [!abstract] Pattern recap
> **DFS** — connected components, paths-existence, cycle detection. Stack or recursion.
> **BFS** — shortest path in **unweighted** graph, level/distance from source. Queue.
> 
> Always track **visited** to prevent revisits (and infinite loops in cyclic graphs).
> 
> ```
> # BFS template (shortest path):
> q = deque([(start, 0)])
> seen = {start}
> while q:
>     node, d = q.popleft()
>     if node == target: return d
>     for nb in neighbors(node):
>         if nb not in seen:
>             seen.add(nb)
>             q.append((nb, d + 1))
> ```

> [!tip] Grid as graph
> 4-directional grid → each cell is a node, 4 edges to neighbors. Same BFS/DFS skeleton.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Number of Islands | 200 | Med | DFS/BFS flood fill |
| P2 | Max Area of Island | 695 | Med | DFS counting |
| P3 | Clone Graph | 133 | Med | DFS + hash map |
| P4 | Walls and Gates | 286 | Med | Multi-source BFS |
| P5 | Rotting Oranges | 994 | Med | Multi-source BFS |
| P6 | Pacific Atlantic Water Flow | 417 | Med | Reverse-direction BFS |
| P7 | Surrounded Regions | 130 | Med | Boundary DFS |
| P8 | Word Ladder | 127 | Hard | BFS state graph |
| P9 | Open the Lock | 752 | Med | BFS state graph |
| P10 | Shortest Path in Binary Matrix | 1091 | Med | BFS 8-direction |
| P11 | 01 Matrix | 542 | Med | Multi-source BFS |
| P12 | Keys and Rooms | 841 | Med | DFS reachability |
| P13 | All Paths From Source to Target | 797 | Med | DFS all paths |
| P14 | Number of Provinces | 547 | Med | DFS components |
| P15 | Number of Closed Islands | 1254 | Med | Boundary-tainted DFS |
| P16 | Find Eventual Safe States | 802 | Med | DFS + 3 colors |

---

## P1: Number of Islands

**LC #200** · Medium

Count connected groups of `'1'` in a 2D grid.

### 🧠 Pattern: Flood Fill (DFS or BFS)

> For each unvisited `'1'`, run DFS/BFS to mark the entire island. Each launch = one island.

### Trace

```
grid=[
  ['1','1','0','0'],
  ['1','0','0','1'],
  ['0','0','1','1'],
]
(0,0) '1' → DFS marks (0,0),(0,1),(1,0)  count=1
(1,3) '1' → DFS marks (1,3),(2,3),(2,2)  count=2
return 2
```

> [!success]- JS
> ```js
> const numIslands = (grid) => {
>   const R = grid.length, C = grid[0].length;
>   let count = 0;
>   const dfs = (r, c) => {
>     if (r < 0 || r >= R || c < 0 || c >= C || grid[r][c] !== '1') return;
>     grid[r][c] = '0';
>     dfs(r + 1, c); dfs(r - 1, c); dfs(r, c + 1); dfs(r, c - 1);
>   };
>   for (let r = 0; r < R; r++) for (let c = 0; c < C; c++) {
>     if (grid[r][c] === '1') { dfs(r, c); count++; }
>   }
>   return count;
> };
> ```

> [!success]- Python
> ```python
> def num_islands(grid):
>     R, C = len(grid), len(grid[0])
>     count = 0
>     def dfs(r, c):
>         if r < 0 or r >= R or c < 0 or c >= C or grid[r][c] != '1':
>             return
>         grid[r][c] = '0'
>         dfs(r+1,c); dfs(r-1,c); dfs(r,c+1); dfs(r,c-1)
>     for r in range(R):
>         for c in range(C):
>             if grid[r][c] == '1':
>                 dfs(r, c)
>                 count += 1
>     return count
> ```

> [!tip] Mutating the grid as "visited" is fine in interviews
> Saves an extra `visited` array. Some interviewers prefer non-mutation — clarify.

**Key takeaway:** Connected components on grid = flood fill. Mutate `'1' → '0'` instead of a `visited` set.

---

## P2: Max Area of Island

**LC #695** · Medium

### Approach

Same DFS, return **size** as you go.

> [!success]- Python
> ```python
> def max_area_of_island(grid):
>     R, C = len(grid), len(grid[0])
>     def dfs(r, c):
>         if r < 0 or r >= R or c < 0 or c >= C or grid[r][c] != 1: return 0
>         grid[r][c] = 0
>         return 1 + dfs(r+1,c) + dfs(r-1,c) + dfs(r,c+1) + dfs(r,c-1)
>     best = 0
>     for r in range(R):
>         for c in range(C):
>             if grid[r][c] == 1:
>                 best = max(best, dfs(r, c))
>     return best
> ```

**Key takeaway:** DFS that returns counts/sums = aggregate over component. Same skeleton as P1.

---

## P3: Clone Graph

**LC #133** · Medium

Deep copy a connected undirected graph (each node has `val` and `neighbors[]`).

### 🧠 Pattern: DFS/BFS + Hash Map original → clone

> [!success]- Python
> ```python
> def clone_graph(node):
>     if not node: return None
>     mp = {}
>     def dfs(n):
>         if n in mp: return mp[n]
>         copy = Node(n.val)
>         mp[n] = copy
>         for nb in n.neighbors:
>             copy.neighbors.append(dfs(nb))
>         return copy
>     return dfs(node)
> ```

**Key takeaway:** Memoize by original node — prevents infinite recursion on cycles AND ensures each original maps to one clone.

---

## P4: Walls and Gates

**LC #286** · Medium · Multi-source BFS

Fill each empty room with distance to nearest gate.

### 🧠 Pattern: Multi-Source BFS

> Push ALL gates into the queue at distance 0. BFS expands outward layer by layer; each empty cell's distance = the first time it's visited.

> [!success]- Python
> ```python
> from collections import deque
> def walls_and_gates(rooms):
>     R, C = len(rooms), len(rooms[0])
>     q = deque()
>     for r in range(R):
>         for c in range(C):
>             if rooms[r][c] == 0:
>                 q.append((r, c))
>     while q:
>         r, c = q.popleft()
>         for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
>             nr, nc = r + dr, c + dc
>             if 0 <= nr < R and 0 <= nc < C and rooms[nr][nc] == 2147483647:
>                 rooms[nr][nc] = rooms[r][c] + 1
>                 q.append((nr, nc))
> ```

> [!tip] When to use multi-source
> "Distance from nearest of many sources" → seed BFS queue with ALL sources at level 0. Way faster than per-source BFS.

**Key takeaway:** Multi-source BFS = the trick when problem says "nearest X" with multiple X's.

---

## P5: Rotting Oranges

**LC #994** · Medium

Minutes until all fresh oranges rot (rot spreads to 4-neighbors per minute).

### Approach

Multi-source BFS from all initially-rotten oranges. Track minutes via BFS level.

> [!success]- Python
> ```python
> from collections import deque
> def oranges_rotting(grid):
>     R, C = len(grid), len(grid[0])
>     q = deque()
>     fresh = 0
>     for r in range(R):
>         for c in range(C):
>             if grid[r][c] == 2: q.append((r, c, 0))
>             elif grid[r][c] == 1: fresh += 1
>     minutes = 0
>     while q:
>         r, c, t = q.popleft()
>         minutes = max(minutes, t)
>         for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
>             nr, nc = r+dr, c+dc
>             if 0 <= nr < R and 0 <= nc < C and grid[nr][nc] == 1:
>                 grid[nr][nc] = 2
>                 fresh -= 1
>                 q.append((nr, nc, t + 1))
>     return minutes if fresh == 0 else -1
> ```

**Key takeaway:** "Time to spread" → multi-source BFS tracking levels.

---

## P6: Pacific Atlantic Water Flow

**LC #417** · Medium

Cells whose water can flow to BOTH oceans (touching N/W = Pacific; S/E = Atlantic).

### 🧠 Pattern: Reverse BFS / DFS from Each Ocean

> Naive: from each cell, BFS to see if reaches both — O((R·C)²).
> 
> **Better:** BFS from ocean cells inward (against gravity — climb upward). Mark all reachable. Intersect the two sets.

> [!success]- Python
> ```python
> def pacific_atlantic(heights):
>     R, C = len(heights), len(heights[0])
>     pac, atl = set(), set()
>     def dfs(r, c, seen, prev):
>         if (r, c) in seen or r < 0 or r >= R or c < 0 or c >= C or heights[r][c] < prev:
>             return
>         seen.add((r, c))
>         for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
>             dfs(r+dr, c+dc, seen, heights[r][c])
>     for r in range(R):
>         dfs(r, 0, pac, 0); dfs(r, C-1, atl, 0)
>     for c in range(C):
>         dfs(0, c, pac, 0); dfs(R-1, c, atl, 0)
>     return [[r, c] for r, c in pac & atl]
> ```

**Key takeaway:** **Reverse the direction** to avoid quadratic. Start from outside, walk inward, intersect.

---

## P7: Surrounded Regions

**LC #130** · Medium

Capture 'O's surrounded by 'X'. Edge-touching 'O's are safe.

### Approach

DFS from each border 'O', mark them as safe (temp char like '#'). Then sweep: '#' → 'O' (safe), 'O' → 'X' (captured).

> [!success]- Python
> ```python
> def solve(board):
>     if not board: return
>     R, C = len(board), len(board[0])
>     def dfs(r, c):
>         if r < 0 or r >= R or c < 0 or c >= C or board[r][c] != 'O': return
>         board[r][c] = '#'
>         dfs(r+1,c); dfs(r-1,c); dfs(r,c+1); dfs(r,c-1)
>     for r in range(R):
>         dfs(r, 0); dfs(r, C-1)
>     for c in range(C):
>         dfs(0, c); dfs(R-1, c)
>     for r in range(R):
>         for c in range(C):
>             board[r][c] = 'O' if board[r][c] == '#' else 'X'
> ```

**Key takeaway:** Boundary-flooding trick: identify the **negation** of what you want, then invert. Many "surrounded by X" problems reduce to this.

---

## P8: Word Ladder

**LC #127** · Hard

Shortest transformation `begin → end`, one letter at a time, each step in word list.

### 🧠 Pattern: BFS on State Graph

> Each word = state. Neighbors = words differing by one char. BFS gives shortest.
> 
> **Optimization:** Build neighbor index lazily. For each position, replace with `*` → bucket of patterns. e.g. `hit` → `*it`, `h*t`, `hi*`.

> [!success]- Python
> ```python
> from collections import deque, defaultdict
> def ladder_length(beginWord, endWord, wordList):
>     words = set(wordList)
>     if endWord not in words: return 0
>     buckets = defaultdict(list)
>     L = len(beginWord)
>     for w in words:
>         for i in range(L):
>             buckets[w[:i] + '*' + w[i+1:]].append(w)
>     q = deque([(beginWord, 1)])
>     seen = {beginWord}
>     while q:
>         w, d = q.popleft()
>         if w == endWord: return d
>         for i in range(L):
>             for nb in buckets[w[:i] + '*' + w[i+1:]]:
>                 if nb not in seen:
>                     seen.add(nb)
>                     q.append((nb, d + 1))
>     return 0
> ```

**Variants:** Word Ladder II (return all shortest paths — BFS + parents map + DFS reconstruct).

**Key takeaway:** Implicit graph search → bucket common patterns for O(1) neighbor lookup.

---

## P9: Open the Lock

**LC #752** · Medium

4-digit wheel; turn one wheel ±1 per move; can't land on deadends. Min moves to target.

### Approach

BFS on state space (10000 states). Skip deadends.

> [!success]- Python
> ```python
> from collections import deque
> def open_lock(deadends, target):
>     dead = set(deadends)
>     if "0000" in dead: return -1
>     if target == "0000": return 0
>     q = deque([("0000", 0)])
>     seen = {"0000"}
>     while q:
>         s, d = q.popleft()
>         for i in range(4):
>             for delta in (-1, 1):
>                 nb = s[:i] + str((int(s[i]) + delta) % 10) + s[i+1:]
>                 if nb in seen or nb in dead: continue
>                 if nb == target: return d + 1
>                 seen.add(nb)
>                 q.append((nb, d + 1))
>     return -1
> ```

**Key takeaway:** "Min steps in state space" → BFS. Encode states as strings.

---

## P10: Shortest Path in Binary Matrix

**LC #1091** · Medium · 8-directional

> [!success]- Python
> ```python
> from collections import deque
> def shortest_path_binary_matrix(grid):
>     N = len(grid)
>     if grid[0][0] or grid[N-1][N-1]: return -1
>     q = deque([(0, 0, 1)])
>     seen = {(0, 0)}
>     dirs = [(-1,-1),(-1,0),(-1,1),(0,-1),(0,1),(1,-1),(1,0),(1,1)]
>     while q:
>         r, c, d = q.popleft()
>         if r == N-1 and c == N-1: return d
>         for dr, dc in dirs:
>             nr, nc = r+dr, c+dc
>             if 0 <= nr < N and 0 <= nc < N and grid[nr][nc] == 0 and (nr,nc) not in seen:
>                 seen.add((nr, nc))
>                 q.append((nr, nc, d + 1))
>     return -1
> ```

**Key takeaway:** 8-direction = 4-direction template + 4 diagonals. Same BFS.

---

## P11: 01 Matrix

**LC #542** · Medium

Distance to nearest 0 from each cell.

### Approach

Multi-source BFS from all 0s. Initialize 1s to ∞; relax during BFS.

> [!success]- Python
> ```python
> from collections import deque
> def update_matrix(mat):
>     R, C = len(mat), len(mat[0])
>     INF = float('inf')
>     dist = [[INF] * C for _ in range(R)]
>     q = deque()
>     for r in range(R):
>         for c in range(C):
>             if mat[r][c] == 0:
>                 dist[r][c] = 0
>                 q.append((r, c))
>     while q:
>         r, c = q.popleft()
>         for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
>             nr, nc = r+dr, c+dc
>             if 0 <= nr < R and 0 <= nc < C and dist[nr][nc] > dist[r][c] + 1:
>                 dist[nr][nc] = dist[r][c] + 1
>                 q.append((nr, nc))
>     return dist
> ```

**Key takeaway:** P4, P5, P11 are the same pattern. Seed BFS with all sources.

---

## P12: Keys and Rooms

**LC #841** · Medium

Can you visit all rooms starting from room 0 (each room contains keys to other rooms)?

> [!success]- Python
> ```python
> def can_visit_all_rooms(rooms):
>     seen = {0}
>     stack = [0]
>     while stack:
>         r = stack.pop()
>         for k in rooms[r]:
>             if k not in seen:
>                 seen.add(k)
>                 stack.append(k)
>     return len(seen) == len(rooms)
> ```

**Key takeaway:** Reachability = DFS/BFS from source.

---

## P13: All Paths From Source to Target

**LC #797** · Medium · DAG

### Approach

DFS over a DAG (acyclic) — no need for visited.

> [!success]- Python
> ```python
> def all_paths_source_target(graph):
>     out, path = [], []
>     target = len(graph) - 1
>     def dfs(node):
>         path.append(node)
>         if node == target:
>             out.append(path[:])
>         else:
>             for nb in graph[node]:
>                 dfs(nb)
>         path.pop()
>     dfs(0)
>     return out
> ```

**Key takeaway:** DAG = no cycles → no visited set needed for "all paths".

---

## P14: Number of Provinces

**LC #547** · Medium

Adjacency matrix; count connected components.

> [!success]- Python
> ```python
> def find_circle_num(isConnected):
>     n = len(isConnected)
>     seen = set()
>     def dfs(i):
>         for j in range(n):
>             if isConnected[i][j] and j not in seen:
>                 seen.add(j)
>                 dfs(j)
>     count = 0
>     for i in range(n):
>         if i not in seen:
>             seen.add(i)
>             dfs(i)
>             count += 1
>     return count
> ```

**Key takeaway:** Adjacency matrix is just another graph rep. Count components = launches of DFS.

---

## P15: Number of Closed Islands

**LC #1254** · Medium

Islands not touching the border.

### Approach

First, **flood-fill all border-touching islands** (mark them out). Then count remaining components like P1.

> [!success]- Python
> ```python
> def closed_island(grid):
>     R, C = len(grid), len(grid[0])
>     def dfs(r, c):
>         if r < 0 or r >= R or c < 0 or c >= C or grid[r][c] != 0: return
>         grid[r][c] = 1
>         dfs(r+1,c); dfs(r-1,c); dfs(r,c+1); dfs(r,c-1)
>     for r in range(R):
>         dfs(r, 0); dfs(r, C-1)
>     for c in range(C):
>         dfs(0, c); dfs(R-1, c)
>     count = 0
>     for r in range(R):
>         for c in range(C):
>             if grid[r][c] == 0:
>                 dfs(r, c)
>                 count += 1
>     return count
> ```

**Key takeaway:** "Inner X" problems → first eliminate boundary-tainted X, then count.

---

## P16: Find Eventual Safe States

**LC #802** · Medium

A node is **safe** if every path leads to a terminal. Return all safe nodes (sorted).

### 🧠 Pattern: DFS with 3 Colors (White/Gray/Black)

> White = unvisited. Gray = on current DFS path. Black = fully verified safe. If DFS hits a Gray node → there's a cycle → not safe. Else, all descendants are black → mark this black.

> [!success]- Python
> ```python
> def eventual_safe_nodes(graph):
>     color = [0] * len(graph)  # 0 white, 1 gray, 2 black
>     def dfs(node):
>         if color[node] > 0: return color[node] == 2
>         color[node] = 1
>         for nb in graph[node]:
>             if not dfs(nb): return False
>         color[node] = 2
>         return True
>     return [i for i in range(len(graph)) if dfs(i)]
> ```

**Variants:** Cycle detection in directed graph (same coloring trick).

**Key takeaway:** "Cycle in directed graph" → 3-color DFS (white/gray/black). Gray-hit = cycle.

---

> [!tip] After this drill
> "Connected components / flood fill" → DFS launch per unvisited. "Shortest path unweighted" → BFS. "Distance from nearest X" → multi-source BFS. "Directed cycle detection" → 3-color DFS.
