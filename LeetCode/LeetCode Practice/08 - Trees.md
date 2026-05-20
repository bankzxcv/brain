---
title: "LeetCode Practice: Trees"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - trees
  - dfs
  - bfs
  - bst
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Trees

18 problems · the most common topic in Google interviews. Sub-patterns: **DFS recursion**, **BFS level-order**, **BST property**.

> [!abstract] Pattern recap
> **DFS** — recurse left + right; combine results. Templates: top-down (pass info DOWN), bottom-up (return info UP).
> **BFS** — queue, process by level. Use a level-size loop to know when each level ends.
> **BST** — `left.val < root.val < right.val`. In-order traversal = sorted. Use this for validation, kth, search, insert.

> [!tip] DFS skeleton
> ```python
> def dfs(node):
>     if not node: return base_case
>     l = dfs(node.left)
>     r = dfs(node.right)
>     return combine(node, l, r)
> ```

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Invert Binary Tree | 226 | Easy | DFS swap |
| P2 | Maximum Depth | 104 | Easy | DFS bottom-up |
| P3 | Same Tree | 100 | Easy | DFS parallel |
| P4 | Subtree of Another Tree | 572 | Easy | DFS + sameTree |
| P5 | Diameter of Binary Tree | 543 | Easy | DFS w/ global |
| P6 | Balanced Binary Tree | 110 | Easy | DFS w/ early exit |
| P7 | Symmetric Tree | 101 | Easy | DFS mirror |
| P8 | Level Order Traversal | 102 | Med | BFS by level |
| P9 | Right Side View | 199 | Med | BFS last-of-level |
| P10 | Path Sum | 112 | Easy | DFS top-down |
| P11 | Path Sum II | 113 | Med | DFS + backtracking |
| P12 | LCA of BST | 235 | Easy | BST property |
| P13 | LCA of Binary Tree | 236 | Med | DFS post-order |
| P14 | Validate BST | 98 | Med | DFS with range |
| P15 | Kth Smallest in BST | 230 | Med | In-order traversal |
| P16 | Build Tree from Preorder + Inorder | 105 | Med | Recursive construction |
| P17 | Binary Tree Max Path Sum | 124 | **Hard** | DFS w/ global + gain |
| P18 | Serialize and Deserialize | 297 | **Hard** | Preorder + null markers |

---

## P1: Invert Binary Tree

**LC #226** · Easy

Mirror the tree (swap left/right at every node).

### 🧠 Pattern: DFS Swap (Post-order or Pre-order both work)

> [!example]- 📊 Visual: original vs mirrored
> ```text
>   Original                Mirrored
>        4                       4
>       / \                     / \
>      2   7        ──►        7   2
>     /|   |\                  /|   |\
>    1 3   6 9                9 6   3 1
> 
>   At each node, swap left ↔ right children:
> 
>        ●n●                    ●n●
>        / \      swap         / \
>       L   R     ───►        R   L
> 
>   Recursion direction (post-order shown):
>     dfs(4) ──┬── dfs(2) ──┬── dfs(1)  ← leaf, swap nulls
>              │            └── dfs(3)
>              └── dfs(7) ──┬── dfs(6)
>                           └── dfs(9)
>     ▲ swap happens AFTER both children return
> ```

> [!info]- 🔍 Dry Run
> ```text
> Input:
>         4
>        / \
>       2   7
>      /|   |\
>     1 3   6 9
> 
> invertTree(4):
>   left  = invertTree(2)
>     left  = invertTree(1) → returns node(1) with both children null (after recursive swaps that do nothing)
>     right = invertTree(3) → returns node(3)
>     swap: node(2).left, node(2).right = invertTree(right), invertTree(left)
>          = node(3), node(1)
>     return node(2)  (now has children 3 on left, 1 on right)
>   right = invertTree(7)
>     similar: swap 6 ↔ 9
>     return node(7) with children 9, 6
>   swap: node(4).left, node(4).right = node(7), node(2)
>   return node(4)
> 
> ✅ Output:
>         4
>        / \
>       7   2
>      /|   |\
>     9 6   3 1
> ```

> [!success]- JS
> ```js
> const invertTree = (root) => {
>   if (!root) return null;
>   [root.left, root.right] = [invertTree(root.right), invertTree(root.left)];
>   return root;
> };
> ```

> [!success]- Python
> ```python
> def invert_tree(root):
>     if not root: return None
>     root.left, root.right = invert_tree(root.right), invert_tree(root.left)
>     return root
> ```

**Key takeaway:** Tree recursion = base case + recurse children + combine. Pattern unchanged across 80% of tree problems.

---

## P2: Maximum Depth of Binary Tree

**LC #104** · Easy

### Approach

`depth(node) = 1 + max(depth(left), depth(right))`. Base `null → 0`.

> [!example]- 📊 Visual: depth-from-root labels
> ```text
>   Depths (1-indexed at each node):
> 
>            3       ← depth 1
>           / \
>          9   20    ← depth 2
>             /  \
>            15   7  ← depth 3   ✓ max depth = 3
> 
>   Bottom-up recursion returns subtree height:
> 
>            ●3     returns 1 + max(1, 2) = 3
>           / \
>          ●9   ●20  returns 1 + max(1, 1) = 2
>          (1)  / \
>             ●15  ●7   each returns 1
>             (1) (1)
> 
>   Nulls return 0. Each parent adds 1 for itself + max(child heights).
> ```

> [!info]- 🔍 Dry Run
> ```text
> Tree:
>         3
>        / \
>       9   20
>           |\
>          15 7
> 
> depth(3):
>   depth(9):
>     depth(null) = 0
>     depth(null) = 0
>     return 1 + max(0, 0) = 1
>   depth(20):
>     depth(15):
>       depth(null) = 0
>       depth(null) = 0
>       return 1
>     depth(7):
>       return 1 (same)
>     return 1 + max(1, 1) = 2
>   return 1 + max(1, 2) = 3
> 
> ✅ Answer: 3
> ```

> [!success]- JS
> ```js
> const maxDepth = (r) => r ? 1 + Math.max(maxDepth(r.left), maxDepth(r.right)) : 0;
> ```

> [!success]- Python
> ```python
> def max_depth(r):
>     return 0 if not r else 1 + max(max_depth(r.left), max_depth(r.right))
> ```

**Key takeaway:** Bottom-up DFS — children return facts, parent combines + adds itself.

---

## P3: Same Tree

**LC #100** · Easy

> [!example]- 📊 Visual: walking two pointers in parallel
> ```text
>   p tree         q tree
> 
>      ●1            ●1     ← compare (p, q): vals equal ✓
>     /  \          /  \
>    ●2  ●3        ●2  ●3   ← recurse pairs in lockstep
> 
>   Two pointers traverse the SAME positions:
> 
>      (p,q)         step 1: compare roots
>      ╱   ╲
>   (pL,qL) (pR,qR)  step 2: compare left pair AND right pair
> 
>   Mismatch path (counter-example p=[1,2,1], q=[1,1,2]):
> 
>      ●1   ●1       ✓ equal
>     / \   / \
>    ●2  ●1 ●1  ●2   ← p.left=2 vs q.left=1: MISMATCH ✗ → false
>    ↑       ↑
>    └───────┘ short-circuit, no further recursion
> ```

> [!info]- 🔍 Dry Run: p=[1,2,3], q=[1,2,3]
> ```text
> Trees:
>     1       1
>    / \     / \
>   2   3   2   3
> 
> sameTree(p=1, q=1):
>   both not null, p.val==q.val (1==1) ✓
>   sameTree(p.left=2, q.left=2):
>     1==1 wait, vals are 2==2 ✓
>     sameTree(null, null) → true
>     sameTree(null, null) → true
>     return true
>   sameTree(p.right=3, q.right=3):
>     similar → true
>   return true && true = true
> 
> ✅ Answer: true
> 
> ─────────────────────────────────────────
> Counter-example: p=[1,2,1] vs q=[1,1,2]
>   p.val==q.val (1==1) ✓
>   sameTree(p.left=2, q.left=1): vals 2!=1 → false
>   short-circuit → false
> ```

> [!success]- JS
> ```js
> const isSameTree = (a, b) => {
>   if (!a && !b) return true;
>   if (!a || !b || a.val !== b.val) return false;
>   return isSameTree(a.left, b.left) && isSameTree(a.right, b.right);
> };
> ```

> [!success]- Python
> ```python
> def is_same_tree(a, b):
>     if not a and not b: return True
>     if not a or not b or a.val != b.val: return False
>     return is_same_tree(a.left, b.left) and is_same_tree(a.right, b.right)
> ```

**Key takeaway:** Parallel DFS on two trees. Base cases first; values next; recurse.

---

## P4: Subtree of Another Tree

**LC #572** · Easy

Is `subRoot` a subtree of `root`?

### Approach

DFS over root: at each node, check `isSameTree(node, subRoot)`. Recurse children.

> [!example]- 📊 Visual: matching subtree highlighted
> ```text
>   root:                        sub:
>          3                       4
>         / \                     / \
>     ╔══●═══╗   5                1   2
>     ║  4   ║                   
>     ║ / \  ║                   
>     ║●1  ●2║                   
>     ╚══════╝
>     ↑
>     matched! this subtree == sub
> 
>   Walk every node of root; at each, ask "does sameTree(node, sub) hold?"
> 
>     isSubtree(3) ──┬── sameTree(3, 4)? ✗
>                    ├── isSubtree(4) ──┬── sameTree(4, 4)? ✓ ★ FOUND
>                    │                  └── short-circuit
>                    └── (skipped — already true)
> 
>   Time: O(m·n) — m nodes in root, each may trigger an O(n) sameTree check.
> ```

> [!info]- 🔍 Dry Run: root=[3,4,5,1,2], sub=[4,1,2]
> ```text
> root:           sub:
>     3            4
>    / \          / \
>   4   5        1   2
>  / \
> 1   2
> 
> isSubtree(3, sub):
>   isSameTree(3, sub=4)? 3!=4 → false
>   isSubtree(3.left=4, sub):
>     isSameTree(4, 4)?
>       4==4 ✓
>       isSameTree(1, 1) → true
>       isSameTree(2, 2) → true
>       → true!
>     return true
>   short-circuit → return true
> 
> ✅ Answer: true
> ```

> [!success]- JS
> ```js
> const isSubtree = (root, sub) => {
>   if (!root) return false;
>   if (isSameTree(root, sub)) return true;
>   return isSubtree(root.left, sub) || isSubtree(root.right, sub);
> };
> ```

> [!success]- Python
> ```python
> def is_subtree(root, sub):
>     if not root: return False
>     if is_same_tree(root, sub): return True
>     return is_subtree(root.left, sub) or is_subtree(root.right, sub)
> ```

**Key takeaway:** Reuse sub-skills (P3) inside larger DFS. O(m·n) acceptable here.

---

## P5: Diameter of Binary Tree

**LC #543** · Easy

Longest path **(in edges)** between any two nodes.

### 🧠 Pattern: DFS Returns Depth, Global Tracks Best

> A path through node `n` has length `leftDepth + rightDepth`. Update global; return `1 + max(leftDepth, rightDepth)` to the parent.

> [!example]- 📊 Visual: longest path through a node
> ```text
>   Tree:                          
>            1                     
>           / \                    
>          2   3                   
>         / \                      
>        4   5                     
> 
>   At each node, gather:
>     L = longest downward path in left subtree
>     R = longest downward path in right subtree
>     "through-me" path length = L + R   (in edges)
>     return to parent: 1 + max(L, R)    (single-path going up)
> 
>   ┌──────── path through node 2: 4→2→5  (length 2) ────────┐
>   │                                                         │
>   │   L=1 (4)        R=1 (5)                                │
>   │   leaf→2          leaf→2                                │
>   │   contrib 2 = 2                                         │
>   │                                                         │
>   └─────────────────────────────────────────────────────────┘
> 
>   ┌──── path through node 1: 4→2→1→3  (length 3) ✓ best ────┐
>   │                                                         │
>   │   L=2 (4→2)      R=1 (3→1)                              │
>   │   contrib 1 = 3 ← global best updated                   │
>   │                                                         │
>   └─────────────────────────────────────────────────────────┘
> 
>   Visualize the winning path:
> 
>            1 ●━━━━━●
>           / ┃     ┃ \
>          2 ●       ● 3
>         / ┃
>        4 ●   5
> 
>   The diameter (3 edges) doesn't have to go through the root —
>   but every diameter passes through SOME node as its apex. Try each.
> ```

> [!info]- 🔍 Dry Run: [1,2,3,4,5]
> ```text
> Tree:
>     1
>    / \
>   2   3
>  / \
> 4   5
> 
> best = 0  (global)
> 
> dfs(1):
>   dfs(2):
>     dfs(4):
>       l = dfs(null) = 0
>       r = dfs(null) = 0
>       best = max(0, 0+0) = 0
>       return 1 + max(0,0) = 1
>     dfs(5): similarly returns 1; best stays 0 (path through 5 = 0+0 = 0)
>     l=1, r=1
>     best = max(0, 1+1) = 2     ← path 4→2→5 = 2 edges
>     return 1 + max(1,1) = 2
>   dfs(3):
>     l=0, r=0
>     best stays 2
>     return 1
>   l=2, r=1
>   best = max(2, 2+1) = 3        ← path 4→2→1→3 (or 5→2→1→3) = 3 edges
>   return 1 + max(2,1) = 3
> 
> ✅ Answer: 3
> ```

> [!success]- JS
> ```js
> const diameterOfBinaryTree = (root) => {
>   let best = 0;
>   const dfs = (n) => {
>     if (!n) return 0;
>     const l = dfs(n.left), r = dfs(n.right);
>     best = Math.max(best, l + r);
>     return 1 + Math.max(l, r);
>   };
>   dfs(root);
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def diameter_of_binary_tree(root):
>     best = 0
>     def dfs(n):
>         nonlocal best
>         if not n: return 0
>         l, r = dfs(n.left), dfs(n.right)
>         best = max(best, l + r)
>         return 1 + max(l, r)
>     dfs(root)
>     return best
> ```

**Key takeaway:** Path-through-node problems → DFS returns one thing (depth), updates global with another (sum of children).

---

## P6: Balanced Binary Tree

**LC #110** · Easy

Every node: `|leftDepth - rightDepth| ≤ 1`.

### 🧠 Pattern: DFS Returns Depth, -1 Sentinel for "Imbalanced"

> Avoid recomputing depth at each level. Return `-1` to short-circuit.

> [!example]- 📊 Visual: balanced vs unbalanced (heights + abs-diff)
> ```text
>   ✓ Balanced                       ✗ Unbalanced
> 
>           ●3  h=3                          ●1  h=4
>          / \                              /
>        ●9   ●20  h=2                    ●2  h=3
>        h=1  / \                         /
>           ●15  ●7  h=1                 ●3  h=2
>           h=1  h=1                     /
>                                       ●4  h=1
>     check at each node:
>       3:  |1-2|=1 ✓                    check at 1: |3-0|=3 → -1 (FAIL)
>       9:  |0-0|=0 ✓                    short-circuit upward
>       20: |1-1|=0 ✓
>       15,7: leaves ✓
> 
>   Sentinel propagation:
> 
>        ●               return -1 if |L-R| > 1
>       / \              else return 1 + max(L, R)
>      L   R
>     (h)  (h)
> 
>     Any -1 from a child → immediately propagate -1.
> ```

> [!info]- 🔍 Dry Run: [3,9,20,null,null,15,7] (balanced)
> ```text
> Tree:
>     3
>    / \
>   9   20
>       |\
>      15 7
> 
> dfs(3):
>   l = dfs(9):
>     dfs(null)=0, dfs(null)=0
>     |0-0|=0 ≤ 1 OK
>     return 1 + max(0,0) = 1
>   r = dfs(20):
>     dfs(15)=1, dfs(7)=1
>     |1-1|=0 OK
>     return 2
>   l=1, r=2
>   |1-2|=1 ≤ 1 OK
>   return 3
> 
> ✅ Answer: true (dfs didn't return -1)
> 
> ─────────────────────────────────────────
> Counter-example: [1,2,2,3,3,null,null,4,4] (left-heavy, unbalanced)
>   At some point, l=3 r=1, diff=2 > 1 → return -1
>   Short-circuits up → final -1 → false
> ```

> [!success]- JS
> ```js
> const isBalanced = (root) => {
>   const dfs = (n) => {
>     if (!n) return 0;
>     const l = dfs(n.left); if (l === -1) return -1;
>     const r = dfs(n.right); if (r === -1) return -1;
>     if (Math.abs(l - r) > 1) return -1;
>     return 1 + Math.max(l, r);
>   };
>   return dfs(root) !== -1;
> };
> ```

> [!success]- Python
> ```python
> def is_balanced(root):
>     def dfs(n):
>         if not n: return 0
>         l = dfs(n.left)
>         if l == -1: return -1
>         r = dfs(n.right)
>         if r == -1: return -1
>         if abs(l - r) > 1: return -1
>         return 1 + max(l, r)
>     return dfs(root) != -1
> ```

**Key takeaway:** -1 sentinel for fast early exit. Naïve recurse-and-check is O(n²) worst case.

---

## P7: Symmetric Tree

**LC #101** · Easy

Is the tree a mirror of itself?

### Approach

Two-pointer DFS: `mirror(a, b) = (a.val == b.val) && mirror(a.l, b.r) && mirror(a.r, b.l)`.

> [!example]- 📊 Visual: mirror line down the middle
> ```text
>                  ●1
>                 / │ \
>                /  │  \
>              ●2   │   ●2
>             / \   │   / \
>           ●3  ●4  │  ●4  ●3
>                   │
>                ── mirror line ──
> 
>   Pairs to compare (drawn with ⇔ across the mirror):
> 
>         ●2  ⇔  ●2          (root.left  ⇔  root.right)
>         ●3  ⇔  ●3          (L.left     ⇔  R.right)
>         ●4  ⇔  ●4          (L.right    ⇔  R.left)
> 
>   Recursion mirrors child positions:
> 
>          mirror(a,b)
>            /      \
>     mirror(a.L,b.R)  mirror(a.R,b.L)   ← note the CROSS
> 
>   Reflect, don't repeat. The "swap" is in the recursive call args.
> ```

> [!info]- 🔍 Dry Run: [1,2,2,3,4,4,3]
> ```text
> Tree:
>        1
>       / \
>      2   2
>     /|   |\
>    3 4   4 3
> 
> mirror(left=2, right=2):
>   2==2 ✓
>   mirror(L.left=3, R.right=3):
>     3==3 ✓; both children null → return true
>   mirror(L.right=4, R.left=4):
>     4==4 ✓; → true
>   return true
> 
> ✅ Answer: true (symmetric)
> ```

> [!success]- JS
> ```js
> const isSymmetric = (root) => {
>   const mirror = (a, b) => {
>     if (!a && !b) return true;
>     if (!a || !b || a.val !== b.val) return false;
>     return mirror(a.left, b.right) && mirror(a.right, b.left);
>   };
>   return !root || mirror(root.left, root.right);
> };
> ```

> [!success]- Python
> ```python
> def is_symmetric(root):
>     def mirror(a, b):
>         if not a and not b: return True
>         if not a or not b or a.val != b.val: return False
>         return mirror(a.left, b.right) and mirror(a.right, b.left)
>     return not root or mirror(root.left, root.right)
> ```

**Key takeaway:** Like SameTree but recurse to **mirror** child pairs.

---

## P8: Binary Tree Level Order Traversal

**LC #102** · Medium

Return values grouped by level.

### 🧠 Pattern: BFS with Level Size Loop

> Queue. Each outer iteration, snapshot `len(queue)` and process exactly that many — that's one level.

> [!example]- 📊 Visual: BFS by level
> ```text
>   Tree:
>            3        ← Level 0
>           / \
>          9   20     ← Level 1
>              | \
>             15  7   ← Level 2
> 
>   BFS expansion (queue snapshot per layer):
> 
>     start:    queue = [3]              process 1 node    → out = [[3]]
>               ↓ enqueue children
>     layer 1:  queue = [9, 20]          process 2 nodes   → out = [[3], [9,20]]
>               ↓ enqueue children (9 has none; 20 has 15,7)
>     layer 2:  queue = [15, 7]          process 2 nodes   → out = [[3],[9,20],[15,7]]
>               ↓ no children
>               queue = []                end
> 
>   Trick: snapshot `n = len(queue)` BEFORE the inner loop so newly-enqueued
>          children (next level) don't leak into the current iteration.
> 
>   ┌─────────────────────────────────┐
>   │  for _ in range(len(q)):        │  ← lock the level size here
>   │      node = q.popleft()         │
>   │      level.append(node.val)     │
>   │      if node.left:  q.append(...│
>   │      if node.right: q.append(...│
>   └─────────────────────────────────┘
> ```

> [!info]- 🔍 Dry Run: [3,9,20,null,null,15,7]
> ```text
> Tree:
>     3
>    / \
>   9   20
>       |\
>      15 7
> 
> queue = [node(3)], out = []
> 
> ─────────────────────────────────────────
> Outer iter 1: n = len(queue) = 1
>   level = []
>   pop node(3): level = [3]; enqueue 9, 20
>   queue = [9, 20]
>   out.append([3]) → out=[[3]]
> 
> Outer iter 2: n = 2
>   level = []
>   pop 9: level = [9]; no children to enqueue (both null)
>   pop 20: level = [9, 20]; enqueue 15, 7
>   queue = [15, 7]
>   out.append([9, 20]) → out=[[3],[9,20]]
> 
> Outer iter 3: n = 2
>   level = []
>   pop 15: level = [15]; no children
>   pop 7:  level = [15, 7]; no children
>   queue = []
>   out.append([15, 7]) → out=[[3],[9,20],[15,7]]
> 
> Queue empty → exit.
> 
> ✅ Answer: [[3], [9, 20], [15, 7]]
> ```

> [!success]- JS
> ```js
> const levelOrder = (root) => {
>   if (!root) return [];
>   const out = [], q = [root];
>   while (q.length) {
>     const level = [], n = q.length;
>     for (let i = 0; i < n; i++) {
>       const node = q.shift();
>       level.push(node.val);
>       if (node.left) q.push(node.left);
>       if (node.right) q.push(node.right);
>     }
>     out.push(level);
>   }
>   return out;
> };
> ```

> [!success]- Python
> ```python
> from collections import deque
> def level_order(root):
>     if not root: return []
>     out, q = [], deque([root])
>     while q:
>         level = []
>         for _ in range(len(q)):
>             node = q.popleft()
>             level.append(node.val)
>             if node.left: q.append(node.left)
>             if node.right: q.append(node.right)
>         out.append(level)
>     return out
> ```

**Key takeaway:** "Process by level" → BFS + level-size loop. The most reusable BFS template.

---

## P9: Binary Tree Right Side View

**LC #199** · Medium

Values visible from the right at each level.

### Approach

BFS by level; last node of each level.

> [!example]- 📊 Visual: rightmost-per-level circled
> ```text
>   Tree:                            What you see from the RIGHT:
> 
>          1                                  ◉1     ← level 0
>         / \                                /
>        2   ◉3   ← level 1 last           (2)  ◉3
>         \   \                              \   \
>          5   ◉4  ← level 2 last           (5)  ◉4
> 
>   ◉ = visible from right       (·) = blocked by something to its right
> 
>   BFS keeps level groups; capture i == n-1 (last index):
> 
>     Level 0: [1]        → output 1   ◉
>     Level 1: [2, 3]     → output 3   ◉
>     Level 2: [5, 4]     → output 4   ◉
> 
>   ✓ Answer: [1, 3, 4]
> 
>   Alternative DFS trick: visit right-first, record first node seen at each depth.
> ```

> [!info]- 🔍 Dry Run: [1,2,3,null,5,null,4]
> ```text
> Tree:
>     1
>    / \
>   2   3
>    \   \
>     5   4
> 
> queue=[1]
> Iter 1: n=1
>   i=0: node=1; this is i==n-1 → out.append(1)
>        enqueue 2 (left), 3 (right)
>   queue = [2, 3]
> 
> Iter 2: n=2
>   i=0: node=2; not last; enqueue null... wait, only enqueue non-null
>        node(2).left=null, .right=node(5) → enqueue 5
>   i=1: node=3; last (i==n-1) → out.append(3)
>        enqueue null, node(4)
>   queue = [5, 4]
> 
> Iter 3: n=2
>   i=0: node=5; not last; no children
>   i=1: node=4; last → out.append(4); no children
>   queue = []
> 
> ✅ Answer: [1, 3, 4]
>   (rightmost visible nodes per level)
> ```

> [!success]- JS
> ```js
> const rightSideView = (root) => {
>   if (!root) return [];
>   const out = [], q = [root];
>   while (q.length) {
>     const n = q.length;
>     for (let i = 0; i < n; i++) {
>       const node = q.shift();
>       if (i === n - 1) out.push(node.val);
>       if (node.left) q.push(node.left);
>       if (node.right) q.push(node.right);
>     }
>   }
>   return out;
> };
> ```

> [!success]- Python
> ```python
> from collections import deque
> def right_side_view(root):
>     if not root: return []
>     out, q = [], deque([root])
>     while q:
>         n = len(q)
>         for i in range(n):
>             node = q.popleft()
>             if i == n - 1: out.append(node.val)
>             if node.left: q.append(node.left)
>             if node.right: q.append(node.right)
>     return out
> ```

**Key takeaway:** "View / per-level extreme" → BFS, capture index-specific node.

---

## P10: Path Sum

**LC #112** · Easy

Root-to-leaf path summing to `target`?

### Approach

Top-down DFS: subtract `node.val` from target. Leaf with `target == 0` → true.

> [!example]- 📊 Visual: valid root-to-leaf path (target=22)
> ```text
>                ●5━━━┓               remaining: 22 - 5 = 17
>               /     ┃
>             ●4━━━━━━┫               remaining: 17 - 4 = 13
>             /       ┃   \
>          ●11━━━━━━━━┫    13         remaining: 13 - 11 = 2
>          /  ┃        \    \
>         7   ●2━━━━━━━┛    4         leaf hit: 2 == 2 ✓ → TRUE
>             ▲
>             goal leaf
> 
>   Subtracting accumulator:
> 
>     target=22 ──► 17 ──► 13 ──► 2 ──► leaf reached, 2-2=0 ✓
> 
>   Each step: t' = t - node.val. At a leaf, check t' == 0  (equiv. node.val == t).
> 
>   ━━━ = the winning path
> ```

> [!info]- 🔍 Dry Run: tree=[5,4,8,11,null,13,4,7,2,null,null,null,1], target=22
> ```text
> Path 5→4→11→2 sums to 22.
> 
> hasPathSum(5, 22):
>   not leaf
>   hasPathSum(4, 22-5=17):
>     not leaf
>     hasPathSum(11, 17-4=13):
>       not leaf
>       hasPathSum(7, 13-11=2):
>         leaf; 7 == 2? NO → false
>       hasPathSum(2, 2):
>         leaf; 2 == 2 ✓ → true
>       return true
>     return true (short-circuit)
>   return true
> 
> ✅ Answer: true
> ```

> [!success]- JS
> ```js
> const hasPathSum = (root, target) => {
>   if (!root) return false;
>   if (!root.left && !root.right) return root.val === target;
>   const t = target - root.val;
>   return hasPathSum(root.left, t) || hasPathSum(root.right, t);
> };
> ```

> [!success]- Python
> ```python
> def has_path_sum(root, target):
>     if not root: return False
>     if not root.left and not root.right:
>         return root.val == target
>     t = target - root.val
>     return has_path_sum(root.left, t) or has_path_sum(root.right, t)
> ```

**Key takeaway:** Top-down DFS — accumulator passes down, decision at leaves.

---

## P11: Path Sum II

**LC #113** · Medium

All root-to-leaf paths summing to target.

### 🧠 Pattern: DFS + Backtracking

> Push node onto path; recurse; pop on return.

> [!example]- 📊 Visual: two paths summing to 22
> ```text
>                  ●5━━━━━━━━━━━━━━━╗
>                 ╱ ┃               ╲ ┃
>               ●4━━┫              ●8━┫
>              ╱    ┃             ╱ ┃ ╲
>           ●11━━━━━┫           13 ●4━┫
>          ╱  ┃      ╲              ╱ ┃ ╲
>         7   ●2━━━━┛━━━━━╗      ●5━┛  1
>         ▲   ▲           ║      ▲
>         ╲   ╲           ║      ╲
>          path A: 5→4→11→2 = 22  ━━━ (double line)
>                              path B: 5→8→4→5 = 22  ═══ (different line)
> 
>   Backtracking stack snapshot (printable trace):
> 
>     path=[5]
>       path=[5,4]
>         path=[5,4,11]
>           path=[5,4,11,7]   leaf, 27 ≠ 22  ✗   pop 7
>           path=[5,4,11,2]   leaf, 22 = 22  ✓   record copy   pop 2
>         pop 11
>       pop 4
>       path=[5,8]
>         path=[5,8,4]
>           path=[5,8,4,5]   leaf, 22 ✓   record copy   pop 5
>           ...
> 
>   Push on entry, pop on exit — list reused, snapshots taken via path[:].
> ```

> [!info]- 🔍 Dry Run: tree=[5,4,8,11,null,13,4,7,2,null,null,5,1], target=22
> ```text
> Same tree as P10 but with two valid paths now.
> 
> dfs(5, 22): path=[5]
>   dfs(4, 17): path=[5,4]
>     dfs(11, 13): path=[5,4,11]
>       dfs(7, 2): leaf; 7!=2 → skip. pop 7. path=[5,4,11]
>       dfs(2, 2): leaf; 2==2 ✓ → out.append([5,4,11,2]). pop 2.
>     pop 11
>   pop 4
>   dfs(8, 17): path=[5,8]
>     dfs(13, 9): path=[5,8,13]
>       leaf? (let's assume yes per modified test). 13!=9 → skip.
>     pop 13
>     dfs(4, 9): path=[5,8,4]
>       dfs(5, 5): leaf; 5==5 ✓ → out.append([5,8,4,5]). pop.
>       dfs(1, 5): leaf; 1!=5 → skip
>     pop 4
>   pop 8
> 
> ✅ Answer: [[5,4,11,2], [5,8,4,5]]
> ```

> [!success]- JS
> ```js
> const pathSum = (root, target) => {
>   const out = [], path = [];
>   const dfs = (n, t) => {
>     if (!n) return;
>     path.push(n.val);
>     if (!n.left && !n.right && t === n.val) out.push([...path]);
>     dfs(n.left, t - n.val);
>     dfs(n.right, t - n.val);
>     path.pop();
>   };
>   dfs(root, target);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def path_sum(root, target):
>     out, path = [], []
>     def dfs(n, t):
>         if not n: return
>         path.append(n.val)
>         if not n.left and not n.right and t == n.val:
>             out.append(path[:])
>         dfs(n.left, t - n.val)
>         dfs(n.right, t - n.val)
>         path.pop()
>     dfs(root, target)
>     return out
> ```

**Key takeaway:** Whenever you want **all** paths, push/pop on the path list (backtracking). Snapshot with `[:]`.

---

## P12: Lowest Common Ancestor of BST

**LC #235** · Easy

### 🧠 Pattern: BST Property

> If both p, q < root → LCA in left. If both > root → LCA in right. Else, root is the split point = LCA.

> [!example]- 📊 Visual: BST split point for p=2, q=8
> ```text
>                  ◎6   ◀── SPLIT! p=2 < 6 < q=8 → 6 is the LCA
>                 ╱   ╲
>               ●2     ●8
>              ╱  ╲   ╱  ╲
>             0    4 7    9
>                 ╱ ╲
>                3   5
> 
>   Walking down the BST, the FIRST node where p and q part ways = LCA.
> 
>   Decision at each node:
>     ┌──────────────────────────────────────────────────┐
>     │  p.val < node.val AND q.val < node.val → go LEFT │
>     │  p.val > node.val AND q.val > node.val → go RIGHT│
>     │  otherwise (one each side, or hit) → THIS IS LCA │
>     └──────────────────────────────────────────────────┘
> 
>   Another case: p=2, q=4 (both ≤ 6, then split at 2)
> 
>                  6   both 2,4 < 6 → go LEFT
>                 ╱
>               ◎2   ◀── p hits self; q=4 is on right of 2 → SPLIT → LCA=2
> ```

> [!info]- 🔍 Dry Run: root=[6,2,8,0,4,7,9,null,null,3,5], p=2, q=8
> ```text
> Tree:
>          6
>         / \
>        2   8
>       /|   |\
>      0 4   7 9
>       /|
>      3 5
> 
> lca(6, p=2, q=8):
>   p.val=2 < 6, q.val=8 > 6 → split point → return 6
> 
> ✅ Answer: 6
> 
> ─────────────────────────────────────────
> p=2, q=4:
>   lca(6): both 2,4 < 6 → go left
>   lca(2): p.val=2 (not <), q.val=4 (not >) → split! return 2
> 
> ✅ Answer: 2
> ```

> [!success]- JS
> ```js
> const lowestCommonAncestorBST = (root, p, q) => {
>   if (p.val < root.val && q.val < root.val) return lowestCommonAncestorBST(root.left, p, q);
>   if (p.val > root.val && q.val > root.val) return lowestCommonAncestorBST(root.right, p, q);
>   return root;
> };
> ```

> [!success]- Python
> ```python
> def lca_bst(root, p, q):
>     if p.val < root.val and q.val < root.val: return lca_bst(root.left, p, q)
>     if p.val > root.val and q.val > root.val: return lca_bst(root.right, p, q)
>     return root
> ```

**Key takeaway:** BST = sorted-by-position. Always use the ordering property.

---

## P13: Lowest Common Ancestor of Binary Tree

**LC #236** · Medium

Not a BST — general binary tree.

### 🧠 Pattern: DFS Returns "Found Either"

> If current node is p or q → return self. Else, recurse children. If both sides return non-null → current is LCA. Else, return the non-null side.

> [!example]- 📊 Visual: LCA logic at each node
> ```text
>   Tree:                Looking for LCA of p=5 and q=1
> 
>            3
>           / \
>          5   1
>         / \
>        6   2
>           / \
>          7   4
> 
>   Post-order DFS returns one of:
>     • node itself if it IS p or q
>     • LCA if found below
>     • null otherwise
> 
>   Recursion bubbling up:
> 
>            3 ◀── L=5(non-null), R=1(non-null) → SPLIT → return 3 (LCA!) ✓
>           / \
>      5 →●   ●→ 1
>         ↑      ↑
>         "I am p,        "I am q,
>          return self"    return self"
>         /\
>        6  2
>         (subtree of 5 doesn't matter — once we returned 5, parent decides)
> 
>   ─────────────────────────────────────────
>   Different case: p=5, q=4 (q is descendant of p)
> 
>   gain(5) sees: 5 IS p → return 5 (don't look further)
>   gain(3): L=5, R=null → return L=5
> 
>   Answer = 5.  The "stop at first match" rule auto-handles
>   the ancestor-descendant case correctly.
> ```

> [!info]- 🔍 Dry Run: root=[3,5,1,6,2,0,8,null,null,7,4], p=5, q=1
> ```text
> Tree:
>         3
>        / \
>       5   1
>      /|   |\
>     6 2   0 8
>      /|
>     7 4
> 
> lca(3, p=5, q=1):
>   root != p, root != q
>   L = lca(5, 5, 1) = 5    (node 5 itself is p, returns 5)
>   R = lca(1, 5, 1) = 1    (node 1 is q)
>   both L and R non-null → root=3 is LCA → return 3
> 
> ✅ Answer: 3
> 
> ─────────────────────────────────────────
> p=5, q=4 (q is descendant of p):
>   lca(3,5,4):
>     L=lca(5,5,4): 5 is p → return 5    (doesn't look further; q is below)
>     R=lca(1,5,4): both descendants not p/q → both null → return null
>     L non-null only → return 5
> 
> ✅ Answer: 5
> ```

> [!success]- JS
> ```js
> const lowestCommonAncestor = (root, p, q) => {
>   if (!root || root === p || root === q) return root;
>   const l = lowestCommonAncestor(root.left, p, q);
>   const r = lowestCommonAncestor(root.right, p, q);
>   if (l && r) return root;
>   return l ?? r;
> };
> ```

> [!success]- Python
> ```python
> def lowest_common_ancestor(root, p, q):
>     if not root or root is p or root is q: return root
>     l = lowest_common_ancestor(root.left, p, q)
>     r = lowest_common_ancestor(root.right, p, q)
>     if l and r: return root
>     return l or r
> ```

**Key takeaway:** Post-order DFS returns "subtree contains p or q (or LCA)". Both sides non-null = split = LCA.

---

## P14: Validate BST

**LC #98** · Medium

> [!warning] Common mistake
> Checking only `node.left.val < node.val < node.right.val` is **wrong** — fails when a great-grandchild violates the global range. Must pass `[lo, hi]` down.

### 🧠 Pattern: DFS with Range Constraint

> Recurse with `(lo, hi)`. At node, must have `lo < node.val < hi`. Left child gets `(lo, node.val)`; right gets `(node.val, hi)`.

> [!example]- 📊 Visual: local OK, global FAIL + (lo, hi) range narrowing
> ```text
>   Tricky tree (locally each parent > left, < right — but invalid overall):
> 
>            ●5        range: (-∞, +∞)
>           ╱  ╲
>          1    ●4     ◀── 4 < 5 locally OK, but must be > 5! globally fails
>              ╱ ╲
>             3   6
> 
>   Narrow (lo, hi) as we recurse:
> 
>     visit 5: (-∞, +∞)   ✓ -∞ < 5 < +∞
>       L → visit 1: (-∞, 5)   ✓ -∞ < 1 < 5
>       R → visit 4: (5, +∞)   ✗ requires 5 < 4 — FALSE → return false
> 
>   Range propagation rule:
> 
>               ●n  (lo, hi)
>              ╱  ╲
>     (lo, n) L   R (n, hi)
> 
>     • going left:   new hi = n.val  (everything left must be < n)
>     • going right:  new lo = n.val  (everything right must be > n)
> 
>   Window shrinks monotonically as you descend.
> ```

> [!info]- 🔍 Dry Run: [5,1,4,null,null,3,6]
> ```text
> Tree:        5
>             / \
>            1   4
>               / \
>              3   6
> 
> NOTE: 3 is in right subtree of 5; should be > 5 but is 3 — invalid BST.
> 
> isValidBST(5, lo=-INF, hi=INF):
>   -INF < 5 < INF ✓
>   isValidBST(1, lo=-INF, hi=5):
>     -INF < 1 < 5 ✓
>     null children OK → true
>   isValidBST(4, lo=5, hi=INF):
>     5 < 4? NO → return false
>   short-circuit → return false
> 
> ✅ Answer: false
> 
> ─────────────────────────────────────────
> Valid example: [2,1,3]
>   isValidBST(2, -INF, INF): ✓
>     isValidBST(1, -INF, 2): -INF<1<2 ✓ → true
>     isValidBST(3, 2, INF): 2<3<INF ✓ → true
>   → true
> ```

> [!success]- JS
> ```js
> const isValidBST = (root, lo = -Infinity, hi = Infinity) => {
>   if (!root) return true;
>   if (root.val <= lo || root.val >= hi) return false;
>   return isValidBST(root.left, lo, root.val) && isValidBST(root.right, root.val, hi);
> };
> ```

> [!success]- Python
> ```python
> def is_valid_bst(root, lo=float('-inf'), hi=float('inf')):
>     if not root: return True
>     if not (lo < root.val < hi): return False
>     return is_valid_bst(root.left, lo, root.val) and is_valid_bst(root.right, root.val, hi)
> ```

**Variants:** In-order iterative + track prev — equivalent.

**Key takeaway:** BST validity needs **global** range, not local.

---

## P15: Kth Smallest Element in BST

**LC #230** · Medium

### 🧠 Pattern: In-Order Traversal (yields sorted)

> Iterative in-order with stack. Count nodes visited; return on k.

> [!example]- 📊 Visual: in-order numbering = sorted order
> ```text
>   BST:                  In-order visit numbers (1, 2, 3, ...):
> 
>          3                       3 ③
>         ╱ ╲                     ╱ ╲
>        1   4                  ① 1   4 ④
>         ╲                       ╲
>          2                       2 ②
> 
>   In-order: left → node → right
>   Visit sequence: 1, 2, 3, 4   (sorted!)
> 
>   For k=1 → answer is 1 (stop at first pop)
>   For k=3 → answer is 3
> 
>   Iterative-with-stack walk for k=3:
> 
>     push 3, push 1                       stack: [3, 1]
>     pop 1  (visit #1, k=2)               cur → 1.right = 2
>     push 2                               stack: [3, 2]
>     pop 2  (visit #2, k=1)               cur → 2.right = null
>     pop 3  (visit #3, k=0)  ✓ return 3
> 
>   Stop as soon as the k-th pop happens — don't traverse the rest.
> ```

> [!info]- 🔍 Dry Run: root=[3,1,4,null,2], k=1
> ```text
> Tree:
>     3
>    / \
>   1   4
>    \
>     2
> 
> Iterative in-order:
>   st = [], cur = 3
> 
>   Outer loop: cur or st
>     while cur: push, go left
>       push 3, cur=1
>       push 1, cur=null
>     cur = st.pop() = 1
>     k -= 1 → k=0. k==0 → return cur.val = 1
> 
> ✅ Answer: 1
> 
> ─────────────────────────────────────────
> Same tree, k=3:
>   push 3, push 1 (going left)
>   pop 1, k=2 not 0; cur = 1.right = 2
>     push 2, push nothing (2 has no left)
>     pop 2, k=1; cur = 2.right = null
>   cur null, st has [3]
>   pop 3, k=0 → return 3
> 
> ✅ Answer: 3
> ```

> [!success]- JS
> ```js
> const kthSmallest = (root, k) => {
>   const st = [];
>   let cur = root;
>   while (cur || st.length) {
>     while (cur) { st.push(cur); cur = cur.left; }
>     cur = st.pop();
>     if (--k === 0) return cur.val;
>     cur = cur.right;
>   }
> };
> ```

> [!success]- Python
> ```python
> def kth_smallest(root, k):
>     st, cur = [], root
>     while cur or st:
>         while cur:
>             st.append(cur)
>             cur = cur.left
>         cur = st.pop()
>         k -= 1
>         if k == 0: return cur.val
>         cur = cur.right
> ```

**Key takeaway:** In-order on BST = sorted iteration. Iterative with stack lets you stop early.

---

## P16: Construct Binary Tree from Preorder and Inorder

**LC #105** · Medium

### 🧠 Pattern: Preorder Root + Inorder Split

> Preorder gives the **root** (first elem). Inorder lets you split into left/right subtrees by finding root's index. Recurse.

> [!example]- 📊 Visual: preorder root + inorder split
> ```text
>   preorder = [ ●3 , 9, 20, 15, 7 ]    ← FIRST element is the root
>                ▲
>                root of whole tree
> 
>   inorder  = [ 9 │ ●3 │ 15, 20, 7 ]   ← find root, split here
>                ▲      ▲
>             left      right
>             subtree   subtree
> 
>   Recurse on each side:
> 
>     LEFT  subtree → inorder [9]            preorder next = 9  → leaf
>     RIGHT subtree → inorder [15, 20, 7]    preorder next = 20 → root
> 
>     20's split:    inorder [15 │ ●20 │ 7]
>                              ▲        ▲
>                            left      right
> 
>   Building bottom-up gives:
> 
>            3
>           / \
>          9   20
>             /  \
>            15   7
> 
>   Trick: precompute a hashmap value → inorder-index for O(1) splits → O(n) total.
> ```

> [!info]- 🔍 Dry Run: preorder=[3,9,20,15,7], inorder=[9,3,15,20,7]
> ```text
> idx map (value → inorder index):
>   {9:0, 3:1, 15:2, 20:3, 7:4}
> 
> p (preorder cursor) = 0
> 
> build(l=0, r=4):    inorder range covering all
>   val = preorder[p=0] = 3; p=1
>   m = idx[3] = 1
>   left = build(0, 0):
>     val = preorder[1] = 9; p=2
>     m = 0
>     left = build(0, -1) → null (l > r)
>     right = build(1, 0) → null
>     return node(9, null, null)
>   right = build(2, 4):
>     val = preorder[2] = 20; p=3
>     m = idx[20] = 3
>     left = build(2, 2):
>       val = preorder[3] = 15; p=4
>       m = 2
>       left = build(2, 1) → null
>       right = build(3, 2) → null
>       return node(15)
>     right = build(4, 4):
>       val = preorder[4] = 7; p=5
>       m = 4
>       null, null
>       return node(7)
>     return node(20, 15, 7)
>   return node(3, 9, node(20, 15, 7))
> 
> ✅ Answer:
>         3
>        / \
>       9   20
>           |\
>          15 7
> ```

> [!success]- JS
> ```js
> const buildTree = (preorder, inorder) => {
>   const idx = new Map();
>   inorder.forEach((v, i) => idx.set(v, i));
>   let p = 0;
>   const build = (l, r) => {
>     if (l > r) return null;
>     const val = preorder[p++];
>     const node = { val, left: null, right: null };
>     const m = idx.get(val);
>     node.left = build(l, m - 1);
>     node.right = build(m + 1, r);
>     return node;
>   };
>   return build(0, inorder.length - 1);
> };
> ```

> [!success]- Python
> ```python
> def build_tree(preorder, inorder):
>     idx = {v: i for i, v in enumerate(inorder)}
>     it = iter(preorder)
>     def build(l, r):
>         if l > r: return None
>         val = next(it)
>         m = idx[val]
>         return TreeNode(val, build(l, m - 1), build(m + 1, r))
>     return build(0, len(inorder) - 1)
> ```

**Variants:** From Inorder + Postorder (postorder gives root last; iterate reverse).

**Key takeaway:** Preorder = "what is root?"; Inorder = "split left/right". Map lookups make it O(n).

---

## P17: Binary Tree Maximum Path Sum

**LC #124** · **Hard**

Max sum of any path (any node to any node, edges may go up-and-down at one apex).

### 🧠 Pattern: DFS Returns "Best Downward Gain", Global Tracks "Through-Me"

> Each node: best path going down = `node.val + max(0, leftGain, rightGain)` returned upward (clamp negatives to 0 — don't take them). Update global with `node.val + max(0,L) + max(0,R)` (through-me path).

> [!example]- 📊 Visual: max path through a node
> ```text
>   Tree:
>           -10              gain(-10) = -10 + max(9, 35) = 25 (returned up)
>           /  \              through-me = -10 + 9 + 35 = 34
>          9    20            
>              /  \           gain(20) = 20 + max(15, 7) = 35
>            15    7           through-me = 20 + 15 + 7 = 42  ✓ best
> 
>   Winning path: 15 ─→ 20 ─→ 7    (sum = 42)
> 
>                  -10
>                  /  \
>                 9    ●20●━━●7
>                      ┃
>                      ●15
> 
>   Two values per recursive call — easy to confuse:
> 
>     RETURNED upward:  node.val + max(0, L, R)    "best single-path going up"
>                       ↑ at most ONE branch contributes
>     UPDATE global:    node.val + max(0,L) + max(0,R)   "best path with me as apex"
>                       ↑ BOTH branches contribute (the V-shape)
> 
>   clamp(0): a negative gain only HURTS — drop that subtree.
> ```

> [!info]- 🔍 Dry Run: [-10,9,20,null,null,15,7]
> ```text
> Tree:
>     -10
>     / \
>    9   20
>        |\
>       15 7
> 
> best = -INF (global)
> 
> gain(-10):
>   L = max(0, gain(9))
>     gain(9):
>       L = max(0, gain(null)) = 0
>       R = max(0, gain(null)) = 0
>       through-me: 9+0+0 = 9; best = max(-INF, 9) = 9
>       return 9 + max(0,0) = 9
>     → L_outer = 9
>   R = max(0, gain(20))
>     gain(20):
>       L = max(0, gain(15)):
>         gain(15): both children 0; through=15; best=max(9, 15)=15; return 15
>         → 15
>       R = max(0, gain(7)):
>         gain(7): through=7; best=max(15, 7)=15 (unchanged); return 7
>         → 7
>       through-me: 20 + 15 + 7 = 42; best = max(15, 42) = 42
>       return 20 + max(15, 7) = 35
>     → R_outer = 35
>   through-me: -10 + 9 + 35 = 34; best = max(42, 34) = 42
>   return -10 + max(9, 35) = 25
> 
> ✅ Answer: 42 (path 15 → 20 → 7)
> ```

> [!success]- JS
> ```js
> const maxPathSum = (root) => {
>   let best = -Infinity;
>   const gain = (n) => {
>     if (!n) return 0;
>     const l = Math.max(0, gain(n.left));
>     const r = Math.max(0, gain(n.right));
>     best = Math.max(best, n.val + l + r);
>     return n.val + Math.max(l, r);
>   };
>   gain(root);
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def max_path_sum(root):
>     best = float('-inf')
>     def gain(n):
>         nonlocal best
>         if not n: return 0
>         l = max(0, gain(n.left))
>         r = max(0, gain(n.right))
>         best = max(best, n.val + l + r)
>         return n.val + max(l, r)
>     gain(root)
>     return best
> ```

**Key takeaway:** Returned-up value ≠ updated-global value. Clamp negatives to 0.

---

## P18: Serialize and Deserialize Binary Tree

**LC #297** · **Hard**

### 🧠 Pattern: Preorder + Null Markers

> Serialize: preorder DFS, output `val,` for nodes, `#,` for null. Deserialize: split, consume tokens with a generator.

> [!example]- 📊 Visual: tree ↔ serialized string
> ```text
>   Tree:                              Serialized (preorder + # for null):
> 
>           ●1                          "1,2,#,#,3,4,#,#,5,#,#"
>          ╱  ╲                          │ │ │ │ │ │ │ │ │ │ │
>        ●2    ●3                        ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼
>             ╱  ╲                     ┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
>          ●4    ●5                    │1│2│#│#│3│4│#│#│5│#│#│
>                                      └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
>                                       ▲   ▲   ▲   ▲   ▲   ▲
>   Encoding order (preorder DFS):       │   │   │   │   │   │
>                                  root─┘   │   │   │   │   │
>     1 ─► 2 ─► (null 2.L) ─► (null 2.R) ──┴───┘   │   │   │
>       ─► 3 ─► 4 ─► (null) ─► (null) ──────────┘   │   │
>             ─► 5 ─► (null) ─► (null) ─────────────┴───┘
> 
>   Why preorder + null markers?
>     • Root-first lets you build top-down on the way back in
>     • Null markers eliminate ambiguity (no need for a second order)
>     • A single linear consumer (iterator) rebuilds in O(n)
> 
>     build():
>       tok = next(tokens)
>       if tok == '#': return None
>       return Node(int(tok), build(), build())   ← left first, then right
> ```

> [!info]- 🔍 Dry Run
> ```text
> Tree:        1
>             / \
>            2   3
>               / \
>              4   5
> 
> Serialize (preorder with nulls):
>   visit 1 → "1"
>   visit 2 → "1,2"
>   2.left=null → "1,2,#"
>   2.right=null → "1,2,#,#"
>   visit 3 → "1,2,#,#,3"
>   visit 4 → "1,2,#,#,3,4"
>   4.left null → "1,2,#,#,3,4,#"
>   4.right null → "1,2,#,#,3,4,#,#"
>   visit 5 → "1,2,#,#,3,4,#,#,5"
>   5.left null → "...,#"
>   5.right null → "...,#"
> 
> Output: "1,2,#,#,3,4,#,#,5,#,#"
> 
> ─────────────────────────────────────────
> Deserialize "1,2,#,#,3,4,#,#,5,#,#":
>   iter tokens, build recursive:
>     build(): v="1" → node(1)
>       node.left = build(): v="2" → node(2)
>         node(2).left = build(): v="#" → null
>         node(2).right = build(): v="#" → null
>         return node(2)
>       node.right = build(): v="3" → node(3)
>         node(3).left = build(): v="4" → node(4)
>           node(4).left = null
>           node(4).right = null
>           return node(4)
>         node(3).right = build(): v="5" → node(5)
>           null, null → return node(5)
>         return node(3)
>       return node(1) with kids
> 
> ✅ Round-trip successful.
> ```

> [!success]- JS
> ```js
> const serialize = (root) => {
>   const out = [];
>   const dfs = (n) => {
>     if (!n) { out.push('#'); return; }
>     out.push(n.val);
>     dfs(n.left); dfs(n.right);
>   };
>   dfs(root);
>   return out.join(',');
> };
> 
> const deserialize = (data) => {
>   const tokens = data.split(',');
>   let i = 0;
>   const build = () => {
>     const v = tokens[i++];
>     if (v === '#') return null;
>     return { val: +v, left: build(), right: build() };
>   };
>   return build();
> };
> ```

> [!success]- Python
> ```python
> def serialize(root):
>     out = []
>     def dfs(n):
>         if not n: out.append('#'); return
>         out.append(str(n.val))
>         dfs(n.left); dfs(n.right)
>     dfs(root)
>     return ','.join(out)
> 
> def deserialize(data):
>     it = iter(data.split(','))
>     def build():
>         v = next(it)
>         if v == '#': return None
>         return TreeNode(int(v), build(), build())
>     return build()
> ```

**Key takeaway:** Preorder + explicit nulls = round-trippable encoding. Iterator/index pattern for parsing.

---

> [!tip] After this drill
> See "tree problem" → 90% chance DFS with combine. See "by level / shortest" → BFS. See "BST + search/validate" → use the ordering. Master the DFS skeleton and you've solved most tree problems.
