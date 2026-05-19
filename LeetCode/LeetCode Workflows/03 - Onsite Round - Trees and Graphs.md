---
title: "Workflow 03: Onsite Round — Trees and Graphs"
date: 2026-05-19
tags:
  - leetcode
  - workflow
  - mock-interview
  - trees
  - graphs
parent: "[[LeetCode Study]]"
status: in-progress
---

# Workflow 03: Onsite Round — Trees and Graphs

> [!abstract] Format
> **45 minutes** · **2 problems (1 tree, 1 graph)** · Google's most common round — heavy on recursion, BFS/DFS, topological reasoning.

---

## Problem 1: Course Schedule II

**LC #210** · Medium

> [!example] Prompt
> "There are `numCourses` courses labeled `0..n-1`. Some have prerequisites: `[a, b]` means you must take `b` before `a`. Return a valid order to take all courses, or `[]` if impossible."

### 🎬 Hidden Script

> [!example]- What they're testing
> - Recognize "prereq ordering" → **topological sort**
> - Know two methods: Kahn's (BFS + indegree) or DFS post-order
> - Detect cycles (means impossible)
> 
> **Red flag:** brute-force trying permutations.

### Walkthrough

**Clarify:** "Multi-edge possible? Self-loops? (Both edge cases — self-loop = cycle, impossible.)"

**Approach:** Topological sort via Kahn's algorithm:
1. Build adjacency + indegree
2. Queue all `indeg == 0` nodes
3. Pop node, append to result, decrement neighbors' indeg, enqueue if 0
4. If processed count < n → cycle exists → return `[]`

**Code:**

> [!success]- Python (Kahn's)
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

**Trace:** `n=4, prereqs=[[1,0],[2,1],[3,2]]`
- Graph: 0→1, 1→2, 2→3. Indeg: 0:0, 1:1, 2:1, 3:1
- queue=[0]; pop 0 → out=[0], indeg[1]=0 → queue=[1]
- pop 1 → out=[0,1], indeg[2]=0 → queue=[2]
- ...
- Result: `[0,1,2,3]`

**Complexity:** O(V + E) time, O(V) space.

**Edge cases:**
- No prereqs → return `[0..n-1]`
- Cycle `[[0,1],[1,0]]` → return `[]`
- Multiple valid orders → any one is fine; Kahn's depends on queue order

### Follow-up

**Q:** "What if courses can be taken in parallel — find min semesters?"

**A:** Modify Kahn's to process in **levels**. Each iteration's queue snapshot = one semester. Track depth. Same O(V+E).

---

## Problem 2: Serialize and Deserialize Binary Tree

**LC #297** · Hard

> [!example] Prompt
> "Design an algorithm to serialize a binary tree to a string and deserialize back to the same tree."

### 🎬 Hidden Script

> [!example]- What they're testing
> - Pick a traversal: preorder (cleanest), or level-order
> - **Handle nulls explicitly** — otherwise can't reconstruct shape
> - Iterator/index pattern for parsing
> 
> **Red flag:** trying inorder without metadata (insufficient to reconstruct).

### Walkthrough

**Clarify:** "Values: integers only? Can they be negative / very large? Any uniqueness guarantee?" (Just be defensive — comma is a fine delimiter.)

**Approach:** Preorder DFS. Emit `val,` for nodes, `#,` for nulls. Deserialize via iterator that builds recursively.

**Code:**

> [!success]- Python
> ```python
> def serialize(root):
>     out = []
>     def dfs(n):
>         if not n:
>             out.append('#'); return
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

**Trace:**
```
Tree:    1
        / \
       2   3
          / \
         4   5

Preorder w/ nulls:
  visit 1: out=[1]
  visit 2: out=[1,2]
  null left of 2:  out=[1,2,#]
  null right of 2: out=[1,2,#,#]
  visit 3: out=[1,2,#,#,3]
  visit 4: ...
  ...
Result: "1,2,#,#,3,4,#,#,5,#,#"
```

**Complexity:** O(n) time and space for both directions.

**Edge cases:**
- Empty tree → `"#"` (or just one null marker)
- Single node → `"1,#,#"`
- All left-skewed → `"1,2,3,...n,#,#,#,...,#,#"`

### Follow-up

**Q:** "What if we want a more compact representation for sparse trees?"

**A:** Level-order BFS with `null` placeholders, trim trailing nulls. For balanced trees, preorder is fine; for skewed, BFS without trailing nulls is similar size.

**Q:** "What about for BSTs specifically?"

**A:** Preorder alone is enough — BST property lets you reconstruct via `[lo, hi]` ranges. No null markers needed. O(n) deserialize using a "next value > previous" check.

---

## Self-Eval Checklist

- [ ] Did I name the pattern (topo sort / preorder-with-nulls) BEFORE coding?
- [ ] Did I draw the tree / graph on a whiteboard before coding?
- [ ] Did I handle empty inputs? Single-node? Cycles?
- [ ] Did I propose 2 approaches for the second problem (preorder vs BFS) before picking one?
- [ ] Did I leave time for the follow-up?

---

## Extensions

Other "Trees and Graphs" round combos:
- LC #207 Course Schedule + LC #543 Diameter of Binary Tree
- LC #200 Number of Islands + LC #236 LCA of Binary Tree
- LC #743 Network Delay Time + LC #98 Validate BST
- LC #133 Clone Graph + LC #124 Binary Tree Maximum Path Sum
