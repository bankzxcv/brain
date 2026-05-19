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
| P17 | Binary Tree Max Path Sum | 124 | Hard | DFS w/ global + gain |
| P18 | Serialize and Deserialize | 297 | Hard | Preorder + null markers |

---

## P1: Invert Binary Tree

**LC #226** · Easy

Mirror the tree (swap left/right at every node).

### 🧠 Pattern: DFS Swap (Post-order or Pre-order both work)

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

### Approach

```
sameTree(a, b):
  if both null: true
  if one null: false
  if vals differ: false
  return sameTree(a.l, b.l) AND sameTree(a.r, b.r)
```

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

### Trace

```
        3
       / \
      9  20
         / \
        15  7

queue=[3]  level=[3]   queue=[9,20]
queue=[9,20]  level=[9,20]   queue=[15,7]
queue=[15,7]  level=[15,7]   queue=[]
return [[3],[9,20],[15,7]]
```

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

BFS by level; last node of each level. (Or DFS: visit right first; on first visit at a depth, record.)

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
> 
> Use a hash map of value → inorder index for O(1) lookup → O(n) total.

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

**LC #124** · Hard

Max sum of any path (any node to any node, edges may go up-and-down at one apex).

### 🧠 Pattern: DFS Returns "Best Downward Gain", Global Tracks "Through-Me"

> Each node: best path going down = `node.val + max(0, leftGain, rightGain)` returned upward (clamp negatives to 0 — don't take them). Update global with `node.val + max(0,L) + max(0,R)` (through-me path).

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

**LC #297** · Hard

### 🧠 Pattern: Preorder + Null Markers

> Serialize: preorder DFS, output `val,` for nodes, `#,` for null. Deserialize: split, consume tokens with a generator.

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
