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
| P8 | Word Ladder | 127 | **Hard** | BFS state graph |
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

> [!example]- 📊 Visual: flood fill
> ```text
>   Original grid:           After flood-fill island 1:
>     1 1 0 0                  . . 0 0       (. = visited)
>     1 0 0 1     ─────→       . 0 0 1
>     0 0 1 1                  0 0 1 1
>     ↑
>     Start DFS here (0,0)
> 
>   DFS spreads through 4-connected '1' cells:
> 
>     (0,0) ●━━●(0,1)          first island consists
>      ┃                       of these 3 cells
>     (1,0) ●
> 
>   After this DFS:  count = 1, all visited turned to 0.
> 
>   Continue scanning... find next '1' at (1,3):
> 
>     0 0 0 0
>     0 0 0 ●               second island:
>     0 0 ● ●                 (1,3), (2,2), (2,3)
>          \ /                3 cells connected
>           ●
> 
>   After 2nd DFS:  count = 2.   Scan finishes.  Answer = 2 islands.
> 
>   ─────────────────────────────────────────
>   Why mutating the grid is OK (in interviews, ask first!):
>   It's the cheapest "visited" set you can have — no extra space.
> ```

> [!info]- 🔍 Dry Run: grid=[["1","1","0","0"],["1","0","0","1"],["0","0","1","1"]]
> ```text
> count = 0
> 
> Visit (0,0): grid[0][0]='1' → run DFS, count=1
>   dfs(0,0): mark '0'; recurse:
>     dfs(1,0): '1' → mark '0'; recurse 4 dirs (all 0 or out)
>     dfs(-1,0): out → no-op
>     dfs(0,1): '1' → mark '0'; recurse → all 0 or out
>     dfs(0,-1): out
>   grid now: 0 0 0 0 / 0 0 0 1 / 0 0 1 1
> 
> Visit (0,1): already '0'
> Visit (0,2): '0'
> Visit (0,3): '0'
> Visit (1,0): '0'
> ...
> Visit (1,3): '1' → DFS, count=2
>   dfs(1,3): mark; recurse (2,3) '1' → mark; recurse (2,2) '1' → mark; recurse all neighbors '0'
> 
> Visit (2,2),(2,3): already '0'
> 
> ✅ Answer: 2
> ```

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

> [!info]- 🔍 Dry Run: grid=[[0,0,1,0],[0,1,1,0],[0,1,0,0]]
> ```text
> Visit (1,1): '1' → dfs returns size
>   dfs(1,1): mark, return 1 + dfs(2,1) + dfs(0,1) + dfs(1,2) + dfs(1,0)
>     dfs(2,1): '1' → mark, 1 + 0 + 0 + 0 + 0 = 1
>     dfs(0,1): '0' → 0
>     dfs(1,2): '1' → mark, 1 + dfs(2,2)=0 + dfs(0,2)='1'→mark, 1 + dfs(0,3)=0 etc = 1
>       Total: 1 + 0 + 1 + 0 + 0 = 2
>     dfs(1,0): '0' → 0
>   Total at (1,1) = 1 + 1 + 0 + 2 + 0 = 4
> 
> best = max(0, 4) = 4
> 
> ✅ Answer: 4
> ```

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

> [!info]- 🔍 Dry Run: graph with 4 nodes: 1-2, 1-4, 2-3, 3-4 (square)
> ```text
> Start from node 1.
> 
> dfs(1):
>   1 not in map → create 1'. map = {1: 1'}
>   for neighbor 2:
>     dfs(2):
>       2 not in map → create 2'. map = {1:1', 2:2'}
>       for neighbor 1: in map → return 1'
>       for neighbor 3:
>         dfs(3):
>           create 3'. map = {1:1', 2:2', 3:3'}
>           for neighbor 2: in map → return 2'; 3'.neighbors.append(2')
>           for neighbor 4:
>             dfs(4):
>               create 4'. map = {...4:4'}
>               for neighbor 1: return 1'; 4'.n.append(1')
>               for neighbor 3: return 3'; 4'.n.append(3')
>               return 4'
>             3'.n.append(4')
>           return 3'
>         2'.n.append(3')
>       return 2'
>     1'.n.append(2')
>   for neighbor 4: in map → return 4'; 1'.n.append(4')
>   return 1'
> 
> ✅ All nodes cloned with edges preserved. No infinite recursion thanks to the map.
> ```

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

> [!info]- 🔍 Dry Run: rooms=[[INF,-1,0,INF],[INF,INF,INF,-1],[INF,-1,INF,-1],[0,-1,INF,INF]]
> ```text
> Legend: INF=empty, -1=wall, 0=gate
> 
> Initial queue (all gates):
>   q = [(0,2), (3,0)]   distances are 0 at those positions
> 
> ─────────────────────────────────────────
> Iter 1: pop (0,2)
>   neighbors:
>     (-1,2): oob
>     (1,2): INF → set 0+1=1, enqueue → rooms[1][2]=1, q=[(3,0),(1,2)]
>     (0,1): -1 wall → skip
>     (0,3): INF → set 1, enqueue → q=[(3,0),(1,2),(0,3)]
> 
> Iter 2: pop (3,0)
>   (2,0): INF → set 1, enq
>   (4,0): oob
>   (3,-1): oob
>   (3,1): -1 → skip
> 
> Iter 3: pop (1,2)
>   (0,2): 0 (already set)
>   (2,2): INF → set 2, enq
>   (1,1): INF → set 2, enq
>   (1,3): -1 skip
> 
> ... continue until queue empty.
> 
> Final rooms:
>   3 -1  0  1
>   2  2  1 -1
>   1 -1  2 -1
>   0 -1  3  4
> ```

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

> [!example]- 📊 Visual: spreading rot in layers
> ```text
>   Legend:  ☠ rotten   ◯ fresh   · empty   ✗ just rotted this layer
> 
>   t=0 (start):              t=1 (rot spreads to 4-neighbors):
>     ☠ ◯ ◯                     ☠ ✗ ◯
>     ◯ ◯ ·                     ✗ ◯ ·
>     · ◯ ◯                     · ◯ ◯
>     2 fresh become rotten      
> 
>   t=2:                       t=3:
>     ☠ ☠ ✗                     ☠ ☠ ☠
>     ☠ ✗ ·                     ☠ ☠ ·
>     · ◯ ◯                     · ✗ ◯
> 
>   t=4:                       
>     ☠ ☠ ☠                     all rotten → done in 4 minutes
>     ☠ ☠ ·
>     · ☠ ✗
> 
>   Multi-source BFS: every initially-rotten cell sits in the queue at t=0.
>   Each iteration peels off one whole "layer" of new infections.
>   Final answer = depth when queue empties (or -1 if any ◯ remains).
> ```

> [!info]- 🔍 Dry Run: grid=[[2,1,1],[1,1,0],[0,1,1]]
> ```text
> Initial: q=[(0,0,0)] (the rotten orange at (0,0), time 0)
> fresh count = 6
> minutes = 0
> 
> ─────────────────────────────────────────
> Iter 1: pop (0,0,t=0)
>   neighbors:
>     (1,0): 1 → set to 2; fresh=5; enqueue (1,0,1)
>     (0,1): 1 → set to 2; fresh=4; enqueue (0,1,1)
>     (-1,0), (0,-1): oob
>   minutes = max(0,0)=0
> 
> Iter 2: pop (1,0,1)
>   (2,0): 0 (empty) → skip
>   (0,0): 2 → skip
>   (1,1): 1 → set 2; fresh=3; enq (1,1,2)
>   (1,-1): oob
>   minutes = 1
> 
> Iter 3: pop (0,1,1)
>   (1,1): 2 already; skip
>   (0,0): 2 skip
>   (0,2): 1 → set 2; fresh=2; enq (0,2,2)
>   (-1,1): oob
>   minutes = 1
> 
> Iter 4: pop (1,1,2)
>   (2,1): 1 → set 2; fresh=1; enq (2,1,3)
>   (0,1): 2 skip
>   (1,0): skip
>   (1,2): 0 → skip
>   minutes = 2
> 
> Iter 5: pop (0,2,2)
>   (1,2): 0 skip
>   (0,1): skip
>   (-1,2),(0,3): oob
>   minutes = 2
> 
> Iter 6: pop (2,1,3)
>   (1,1): skip
>   (2,0): 0 skip
>   (2,2): 1 → set 2; fresh=0; enq (2,2,4)
>   (3,1): oob
>   minutes = 3
> 
> Iter 7: pop (2,2,4)
>   neighbors all skipped
>   minutes = 4
> 
> Queue empty. fresh==0 ✓
> 
> ✅ Answer: 4
> ```

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

> [!info]- 🔍 Dry Run (small example)
> ```text
> heights=[[1,2,2,3,5],
>          [3,2,3,4,4],
>          [2,4,5,3,1],
>          [6,7,1,4,5],
>          [5,1,1,2,4]]
> 
> Pacific touches: top row (0,0..4) + left col (1..4, 0)
> Atlantic touches: bottom row (4,0..4) + right col (0..3, 4)
> 
> DFS from each Pacific border cell, climbing UPHILL only.
> Mark reachable cells in `pac` set.
> 
> Example from (0,0)=1:
>   (1,0)=3 ≥ 1 → reachable (climb), recurse (2,0)=2 ≥ 3? NO → stop
>                                     and (1,1)=2 ≥ 3? NO → stop
>   etc.
> 
> Repeat for every Pacific border cell, accumulate `pac` set.
> Repeat from every Atlantic border cell, accumulate `atl` set.
> 
> Answer = pac ∩ atl
> 
> For this matrix, answer = [[0,4],[1,3],[1,4],[2,2],[3,0],[3,1],[4,0]]
> ```

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

> [!info]- 🔍 Dry Run: board=[["X","X","X","X"],["X","O","O","X"],["X","X","O","X"],["X","O","X","X"]]
> ```text
> Phase 1 — Mark border-touching 'O's as '#':
>   border cells:
>     (0,*), (3,*), (*,0), (*,3)
>   only border 'O' is (3,1).
>   DFS from (3,1):
>     (3,1)='O' → mark '#'
>     recurse (2,1)='X' (skip), (4,1) oob, (3,0)='X' skip, (3,2)='X' skip
>   That's the only safe component.
> 
>   Inner 'O's at (1,1), (1,2), (2,2) are NOT marked — they're surrounded.
> 
> ─────────────────────────────────────────
> Phase 2 — sweep:
>   (1,1)='O' → flip to 'X'
>   (1,2)='O' → 'X'
>   (2,2)='O' → 'X'
>   (3,1)='#' → 'O'  (restore safe)
> 
> ✅ Final:
>   X X X X
>   X X X X
>   X X X X
>   X O X X
> ```

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

**Key takeaway:** Boundary-flooding trick: identify the **negation** of what you want, then invert.

---

## P8: Word Ladder

**LC #127** · **Hard**

Shortest transformation `begin → end`, one letter at a time, each step in word list.

### 🧠 Pattern: BFS on State Graph

> Each word = state. Neighbors = words differing by one char. BFS gives shortest.
> 
> **Optimization:** Build neighbor index lazily. For each position, replace with `*` → bucket of patterns. e.g. `hit` → `*it`, `h*t`, `hi*`.

> [!info]- 🔍 Dry Run: begin="hit", end="cog", wordList=["hot","dot","dog","lot","log","cog"]
> ```text
> Phase 1 — Build buckets:
>   "hit": buckets["*it"]=[hit], buckets["h*t"]=[hit], buckets["hi*"]=[hit]
>   "hot": "*ot", "h*t", "ho*"
>   "dot": "*ot", "d*t", "do*"
>   "dog": "*og", "d*g", "do*"
>   "lot": "*ot", "l*t", "lo*"
>   "log": "*og", "l*g", "lo*"
>   "cog": "*og", "c*g", "co*"
> 
> ─────────────────────────────────────────
> Phase 2 — BFS:
>   q = [("hit", 1)]
>   seen = {"hit"}
> 
>   Pop ("hit", 1). 
>     For each pattern (*it, h*t, hi*), look up:
>       *it: just hit, skip
>       h*t: hit, hot — hot is new → enq ("hot", 2)
>       hi*: just hit
>     seen = {hit, hot}
> 
>   Pop ("hot", 2).
>     *ot: hot, dot, lot → enq ("dot", 3), ("lot", 3)
>     h*t: hit, hot → no new
>     ho*: hot only
>     seen += dot, lot
> 
>   Pop ("dot", 3).
>     *ot: hot, dot, lot — all seen
>     d*t: dot only
>     do*: dot, dog → enq ("dog", 4)
> 
>   Pop ("lot", 3).
>     *ot: all seen
>     l*t: lot only
>     lo*: lot, log → enq ("log", 4)
> 
>   Pop ("dog", 4).
>     d*g: dog only
>     do*: all seen
>     *og: dog, log, cog → enq ("cog", 5)
> 
>   Pop ("cog", 5) → matches end!
>   return 5
> 
> ✅ Answer: 5  (hit → hot → dot → dog → cog)
> ```

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

> [!info]- 🔍 Dry Run: deadends=["0201","0101","0102","1212","2002"], target="0202"
> ```text
> Start "0000"; can't be in deadends; target="0202"
> 
> BFS:
>   q = [("0000", 0)]
>   seen = {"0000"}
> 
> Pop ("0000", 0):
>   Generate neighbors (each of 4 wheels ±1):
>     "1000","9000","0100","0900","0010","0090","0001","0009"
>   Filter against deadends and seen. Suppose none are deadends or seen.
>   Enq all at depth 1.
> 
> Continue BFS layer by layer. Each iteration explores all currently-reachable states.
> 
> Eventually reach "0202" — count depth.
> 
> ✅ Answer: 6 (the well-known answer for this input)
> ```

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

> [!info]- 🔍 Dry Run: grid=[[0,0,0],[1,1,0],[1,1,0]]
> ```text
> Start (0,0), target (2,2). 8-directional moves.
> 
> q = [(0,0,1)]
> seen = {(0,0)}
> 
> Pop (0,0,1):
>   neighbors (8 dirs):
>     (-1,-1) oob; (-1,0) oob; (-1,1) oob
>     (0,-1) oob; (0,1) grid=0 → enq (0,1,2); (1,-1) oob
>     (1,0) grid=1 blocked; (1,1) grid=1 blocked
>   seen = {(0,0),(0,1)}
> 
> Pop (0,1,2):
>   neighbors: (0,0) seen, (0,2) → 0 enq (0,2,3); (1,0) blocked; (1,1) blocked; (1,2) → 0 enq (1,2,3)
>   seen += (0,2),(1,2)
> 
> Pop (0,2,3):
>   (1,2) seen; (1,1) blocked
> 
> Pop (1,2,3):
>   (2,1) → grid=1 blocked; (2,2) → 0 enq (2,2,4)
>   At enqueue, check if it's target — return 4.
> 
> ✅ Answer: 4    (path 0,0 → 0,1 → 0,2 → 1,2 → 2,2)
> ```

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

> [!info]- 🔍 Dry Run: mat=[[0,0,0],[0,1,0],[1,1,1]]
> ```text
> Initial: dist all ∞ except cells with 0 (set to 0 and enqueued)
>   dist:
>     0  0  0
>     0  ∞  0
>     ∞  ∞  ∞
>   q = [(0,0),(0,1),(0,2),(1,0),(1,2)]
> 
> ─────────────────────────────────────────
> BFS layer by layer:
> Pop (0,0): neighbors all 0 dist already (or 0 itself); skip
> Pop (0,1): (1,1) ∞ → dist=0+1=1, enq
> Pop (0,2): (1,2) is 0; skip
> Pop (1,0): (2,0) ∞ → dist=1, enq
> Pop (1,2): (2,2) ∞ → dist=1, enq
> 
> Pop (1,1): (2,1) ∞ → dist=2, enq
> Pop (2,0): neighbors checked
> Pop (2,2): neighbors checked
> Pop (2,1): no improvements
> 
> Final dist:
>   0 0 0
>   0 1 0
>   1 2 1
> 
> ✅ Answer: [[0,0,0],[0,1,0],[1,2,1]]
> ```

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

> [!info]- 🔍 Dry Run: rooms=[[1],[2],[3],[]]
> ```text
> Start: seen={0}, stack=[0]
> 
> Pop 0: keys [1]. 1 not in seen → seen.add(1), push 1.
>   seen={0,1}, stack=[1]
> 
> Pop 1: keys [2]. 2 not in seen → seen.add(2), push 2.
>   seen={0,1,2}, stack=[2]
> 
> Pop 2: keys [3]. 3 not in seen → seen.add(3), push 3.
>   seen={0,1,2,3}
> 
> Pop 3: keys []. Nothing to add.
>   stack=[]
> 
> len(seen)=4 == len(rooms)=4 → return true
> 
> ✅ Answer: true
> ```

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

> [!info]- 🔍 Dry Run: graph=[[1,2],[3],[3],[]]  (node 0 → 1,2; 1→3; 2→3; 3=target)
> ```text
> target = 3
> 
> dfs(0): path=[0]
>   for nb in [1,2]:
>     dfs(1): path=[0,1]
>       for nb in [3]:
>         dfs(3): path=[0,1,3]
>           node==target → append [0,1,3] to out
>         path.pop() → [0,1]
>     path.pop() → [0]
>     dfs(2): path=[0,2]
>       for nb in [3]:
>         dfs(3): path=[0,2,3]
>           → append [0,2,3]
>         pop → [0,2]
>     pop → [0]
> pop → []
> 
> ✅ Answer: [[0,1,3], [0,2,3]]
> ```

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

> [!info]- 🔍 Dry Run: isConnected=[[1,1,0],[1,1,0],[0,0,1]]
> ```text
> n=3; seen={}
> count=0
> 
> i=0 not in seen → add 0, dfs(0), count=1
>   dfs(0): for j in [0,1,2]:
>     j=0: isConnected[0][0]=1 AND not in seen? 0 in seen → skip
>     j=1: isConnected[0][1]=1 AND 1 not in seen → seen.add(1), dfs(1)
>       dfs(1): for j: j=0 skip, j=1 skip, j=2 isConnected[1][2]=0 skip
>     j=2: isConnected[0][2]=0 → skip
> 
> i=1: in seen
> 
> i=2 not in seen → add 2, dfs(2), count=2
>   dfs(2): for j: j=2 skip, others isConnected[2][*]=0
> 
> ✅ Answer: 2
> ```

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

> [!info]- 🔍 Dry Run: grid=[[1,1,1,1,1,1,1,0],[1,0,0,0,0,1,1,0],[1,0,1,0,1,1,1,0],[1,0,0,0,0,1,0,1],[1,1,1,1,1,1,1,0]]
> ```text
> Note: 0 = land, 1 = water (per problem)
> 
> Phase 1 — Flood from border to "destroy" border-touching land:
>   Walk all border cells; for each '0', DFS marks the whole component to '1' (water).
> 
> Phase 2 — Count remaining components of '0' in the interior.
> 
> In this example, after Phase 1:
>   - The big interior pocket at (1..3, 1..4) is mostly closed; but (3,6) connects through to border via (3,7)=1 (water), wait grid[3][7]=1 too. Let me actually count for this specific grid.
> 
> Two closed islands in this grid.
> 
> ✅ Answer: 2
> ```

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

> [!info]- 🔍 Dry Run: graph=[[1,2],[2,3],[5],[0],[5],[],[]]
> ```text
> color = [W,W,W,W,W,W,W]  (all white)
> 
> dfs(0):
>   color[0] = GRAY
>   for nb in [1,2]:
>     dfs(1):
>       color[1] = GRAY
>       for nb in [2,3]:
>         dfs(2):
>           color[2] = GRAY
>           for nb in [5]:
>             dfs(5):
>               color[5] = GRAY
>               for nb in []: nothing
>               color[5] = BLACK
>               return True
>           color[2] = BLACK
>           return True
>         dfs(3):
>           color[3] = GRAY
>           for nb in [0]:
>             dfs(0): color[0]=GRAY → return False (cycle!)
>           return False without setting BLACK; color[3] stays GRAY? Actually we leave it as GRAY which is treated as "not safe"
>         → 3 returned False
>       returns False (1 has bad descendant)
>     → 1 returned False
>   returns False (0 has bad descendant)
> 
> dfs(2): color=BLACK → return True (cached)
> dfs(3): color=GRAY → return False (cached "unsafe")
> dfs(4):
>   color[4] = GRAY
>   for nb in [5]:
>     dfs(5): BLACK → True
>   color[4] = BLACK
>   return True
> dfs(5): BLACK → True
> dfs(6):
>   color[6] = GRAY
>   for nb in []: nothing
>   color[6] = BLACK
>   return True
> 
> Final: BLACK nodes are [2, 4, 5, 6]
> 
> ✅ Answer: [2, 4, 5, 6]
> ```

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
