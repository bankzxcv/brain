---
title: "LeetCode Practice: Backtracking"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - backtracking
  - dfs
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Backtracking

13 problems · subsets, permutations, combinations, and constraint-satisfaction.

> [!abstract] Pattern recap
> **Backtracking** = DFS that **builds and unbuilds** a solution. Three moves: **choose** → **recurse** → **un-choose**.
> 
> ```
> def backtrack(path, choices):
>     if goal(path):
>         out.append(path[:])     # snapshot!
>         return
>     for c in choices:
>         if not allowed(c, path): continue
>         path.append(c)
>         backtrack(path, choices)
>         path.pop()              # un-choose
> ```

> [!warning] Two perennial bugs
> 1. **Forgot to snapshot** `path` → all results point to same mutated list. Always `path[:]` or `list(path)`.
> 2. **Forgot to un-choose** → state leaks across branches.

> [!tip] Subsets vs combinations vs permutations
> - **Subsets**: pick or don't pick each item, **any size**.
> - **Combinations**: choose k items, **order doesn't matter**.
> - **Permutations**: order **matters**; visit every arrangement.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Subsets | 78 | Med | Include/exclude |
| P2 | Subsets II (dups) | 90 | Med | Sort + skip dup at same level |
| P3 | Permutations | 46 | Med | Used-set |
| P4 | Permutations II (dups) | 47 | Med | Sort + skip dup |
| P5 | Combinations | 77 | Med | Start index |
| P6 | Combination Sum | 39 | Med | Reuse allowed |
| P7 | Combination Sum II | 40 | Med | No reuse + dups |
| P8 | Letter Combinations of Phone Number | 17 | Med | Cartesian product |
| P9 | Word Search | 79 | Med | DFS on grid |
| P10 | Palindrome Partitioning | 131 | Med | Cut after every palindrome |
| P11 | N-Queens | 51 | **Hard** | Column + diagonal sets |
| P12 | Restore IP Addresses | 93 | Med | Bounded segmentation |
| P13 | Sudoku Solver | 37 | **Hard** | DFS + 3 constraint sets |

---

## P1: Subsets

**LC #78** · Medium

All subsets of distinct `nums`.

### 🧠 Pattern: Include / Exclude via Start Index

> At index `i`, snapshot the current path, then for each `j ≥ i`, include `nums[j]` and recurse with `j+1`.

> [!example]- 📊 Visual: recursion tree
> ```text
>   nums = [1, 2, 3]
> 
>                          [ ]                 ← snapshot at each node
>                       /   |    \
>                      /    |     \
>                  pick 1  pick 2  pick 3
>                    /       |       \
>                  [1]      [2]      [3]       ← snapshot
>                  / \       |
>                 /   \      |
>             pick 2  pick 3 pick 3
>               /       \      \
>             [1,2]   [1,3]   [2,3]            ← snapshot
>              /
>            pick 3
>            /
>         [1,2,3]                              ← snapshot
> 
>   Snapshots collected (each node emits its current path):
>     [ ], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]
> 
>   Each recursion uses `start_index` to avoid re-picking earlier elements
>   (so we never produce permutations, only subsets).
> 
>   Total subsets = 2^n = 8 ✓
> ```

> [!info]- 🔍 Dry Run: nums=[1,2,3]
> ```text
> Setup: out=[], path=[]
> 
> dfs(0):
>   out.append([]) → out=[[]]
>   j=0: push 1, path=[1]
>     dfs(1):
>       out.append([1]) → out=[[], [1]]
>       j=1: push 2, path=[1,2]
>         dfs(2):
>           out.append([1,2])
>           j=2: push 3, path=[1,2,3]
>             dfs(3):
>               out.append([1,2,3])
>               (loop body doesn't execute, j=3 not < 3)
>             pop 3, path=[1,2]
>         pop 2, path=[1]
>       j=2: push 3, path=[1,3]
>         dfs(3): out.append([1,3])
>         pop 3, path=[1]
>     pop 1, path=[]
>   j=1: push 2, path=[2]
>     dfs(2):
>       out.append([2])
>       j=2: push 3, path=[2,3]
>         dfs(3): out.append([2,3])
>         pop
>     pop
>   j=2: push 3, path=[3]
>     dfs(3): out.append([3])
>     pop
> 
> ✅ Answer: [[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]]
> ```

> [!success]- JS
> ```js
> const subsets = (nums) => {
>   const out = [], path = [];
>   const dfs = (i) => {
>     out.push([...path]);
>     for (let j = i; j < nums.length; j++) {
>       path.push(nums[j]);
>       dfs(j + 1);
>       path.pop();
>     }
>   };
>   dfs(0);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def subsets(nums):
>     out, path = [], []
>     def dfs(i):
>         out.append(path[:])
>         for j in range(i, len(nums)):
>             path.append(nums[j])
>             dfs(j + 1)
>             path.pop()
>     dfs(0)
>     return out
> ```

**Key takeaway:** Subsets = snapshot at every node. Start-index avoids visiting same subset twice.

---

## P2: Subsets II (with Duplicates)

**LC #90** · Medium

### 🧠 Pattern: Sort + Skip Duplicates at Same Level

> Sort first. In the loop, if `nums[j] == nums[j-1]` AND `j > i` (not the first sibling), skip — we've already explored this branch.

> [!example]- 📊 Visual: skip dup at SAME level, not within path
> ```text
>   nums = [1, 2, 2]   (sorted)
> 
>                            [ ]
>                       /     |      \
>                    pick 1  pick 2  pick 2 ✗ SKIP
>                                    (dup at same level)
>                    /        |
>                  [1]       [2]
>                  / |        |
>             pick 2 pick 2✗  pick 2     ← deeper "pick 2" is OK
>               |   (same lvl  │           (different level, in path)
>              [1,2] dup skip) [2,2]
>               |
>             pick 2          ← second 2 in the SAME path is fine
>               |
>             [1,2,2]
> 
>   Rule:  skip if  j > i  AND  nums[j] == nums[j-1]
>          (means: not the first sibling, and equals left sibling)
> 
>   Results: [], [1], [1,2], [1,2,2], [2], [2,2]    ← no duplicates
> ```

> [!info]- 🔍 Dry Run: nums=[1,2,2]
> ```text
> Sort: [1, 2, 2] (already sorted)
> 
> dfs(0):
>   out.append([])
>   j=0: push 1, path=[1]
>     dfs(1):
>       out.append([1])
>       j=1: push 2, path=[1,2]
>         dfs(2):
>           out.append([1,2])
>           j=2: dup check: j>start? 2>2? NO → don't skip
>             push 2, path=[1,2,2]
>             dfs(3): append [1,2,2]
>             pop
>         pop
>       j=2: dup check: j>start (1)? 2>1 YES; nums[2]==nums[1]? YES → SKIP
>     pop 1
>   j=1: push 2, path=[2]
>     dfs(2):
>       out.append([2])
>       j=2: dup check: 2>2? NO → don't skip
>         push 2, path=[2,2]
>         dfs(3): append [2,2]
>         pop
>     pop
>   j=2: dup check: 2>0? YES; nums[2]==nums[1] → SKIP
> 
> ✅ Answer: [[], [1], [1,2], [1,2,2], [2], [2,2]]
> ```

> [!success]- JS
> ```js
> const subsetsWithDup = (nums) => {
>   nums.sort((a, b) => a - b);
>   const out = [], path = [];
>   const dfs = (i) => {
>     out.push([...path]);
>     for (let j = i; j < nums.length; j++) {
>       if (j > i && nums[j] === nums[j - 1]) continue;
>       path.push(nums[j]);
>       dfs(j + 1);
>       path.pop();
>     }
>   };
>   dfs(0);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def subsets_with_dup(nums):
>     nums.sort()
>     out, path = [], []
>     def dfs(i):
>         out.append(path[:])
>         for j in range(i, len(nums)):
>             if j > i and nums[j] == nums[j - 1]: continue
>             path.append(nums[j])
>             dfs(j + 1)
>             path.pop()
>     dfs(0)
>     return out
> ```

**Key takeaway:** "Same-level duplicate skip" is the canonical dedup. **Skip at same level, not same path.**

---

## P3: Permutations

**LC #46** · Medium

### 🧠 Pattern: Used Set + Try All Indices

> Different from subsets: we visit ALL indices in every recursion, skipping those already used.

> [!example]- 📊 Visual: permutation decision tree
> ```text
>   nums = [1, 2, 3]
> 
>                           [ ]
>                  /         |         \
>              pick 1     pick 2      pick 3
>                /          |            \
>             [1]          [2]           [3]
>             / \          / \           / \
>          pick2 pick3  pick1 pick3  pick1 pick2
>          /      \      /     \      /     \
>       [1,2]  [1,3]  [2,1]  [2,3]  [3,1]  [3,2]
>         |      |      |      |      |      |
>       pick3  pick2  pick3  pick1  pick2  pick1
>         |      |      |      |      |      |
>      [1,2,3][1,3,2][2,1,3][2,3,1][3,1,2][3,2,1]
> 
>   At each level: every UNUSED index is a child branch.
>   `used[]` tracks which positions are already in the path.
>   Leaves (depth = n) = n! = 6 full permutations.
> ```

> [!info]- 🔍 Dry Run: nums=[1,2,3]
> ```text
> dfs() with path=[], used=[F,F,F]:
>   i=0 used? F → use(0), path=[1]
>     dfs(): path=[1], used=[T,F,F]
>       i=0 used → skip
>       i=1 free → use, path=[1,2]
>         dfs(): used=[T,T,F]
>           i=0 used → skip
>           i=1 used → skip
>           i=2 free → use, path=[1,2,3]
>             dfs(): len==3 → out.append([1,2,3]); return
>           un-use 2, path=[1,2]
>         (loop done)
>       un-use 1, path=[1]
>       i=2 free → use, path=[1,3]
>         dfs():
>           i=0 used → skip
>           i=1 free → use, path=[1,3,2]
>             dfs(): append [1,3,2]
>           un-use
>       un-use
>     un-use 0
>   i=1 free → use, path=[2]
>     dfs():
>       i=0 free → path=[2,1] → dfs() → use 2 → [2,1,3]; un-use 2; un-use 0
>       i=2 free → path=[2,3,1] (similar)
>     un-use
>   i=2 free → produce [3,1,2] and [3,2,1]
> 
> ✅ Answer: [[1,2,3], [1,3,2], [2,1,3], [2,3,1], [3,1,2], [3,2,1]]
> ```

> [!success]- JS
> ```js
> const permute = (nums) => {
>   const out = [], path = [];
>   const used = new Array(nums.length).fill(false);
>   const dfs = () => {
>     if (path.length === nums.length) {
>       out.push([...path]);
>       return;
>     }
>     for (let i = 0; i < nums.length; i++) {
>       if (used[i]) continue;
>       used[i] = true;
>       path.push(nums[i]);
>       dfs();
>       path.pop();
>       used[i] = false;
>     }
>   };
>   dfs();
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def permute(nums):
>     out, path = [], []
>     used = [False] * len(nums)
>     def dfs():
>         if len(path) == len(nums):
>             out.append(path[:])
>             return
>         for i in range(len(nums)):
>             if used[i]: continue
>             used[i] = True
>             path.append(nums[i])
>             dfs()
>             path.pop()
>             used[i] = False
>     dfs()
>     return out
> ```

**Key takeaway:** Permutations need a `used` flag — no start index, you can revisit earlier positions on different branches.

---

## P4: Permutations II (with Duplicates)

**LC #47** · Medium

### Approach

Same as P3 + dedup: sort; in the loop, skip if `nums[i] == nums[i-1] && !used[i-1]` (the previous duplicate hasn't been used yet → using this one creates a dup permutation).

> [!example]- 📊 Visual: force canonical order on equal values
> ```text
>   nums = [1a, 1b, 2]   (sort; treat the two 1s as 1a, 1b)
> 
>                          [ ]
>                /          |         \
>            pick 1a     pick 1b ✗    pick 2
>             (used[0]=F,             (used[2]=T)
>              i=1: nums[1]==nums[0]
>              & !used[0] → SKIP)
> 
>            /          \
>        pick 1b      pick 2
>        (used[0]=T,
>         dup rule OK)
>          /              \
>     pick 2             pick 1a
>      [1a,1b,2]    \ /     │
>                    X    pick 1b ✗ (used[0]=F → SKIP)
>                         │
>                       [2,1a,1b]
> 
>   Rule:  if nums[i] == nums[i-1] AND used[i-1] == False → SKIP
>          (the earlier dup hasn't been placed yet; using THIS one
>           would create a permutation we'd reach via the earlier dup later)
> 
>   Effectively: equal values must be picked in their original index order.
>   Results: [1,1,2], [1,2,1], [2,1,1]   ← no duplicates
> ```

> [!info]- 🔍 Dry Run: nums=[1,1,2]
> ```text
> Sort: [1, 1, 2]
> 
> dfs() with used=[F,F,F]:
>   i=0 nums[0]=1 free; (i>0 check skipped since i=0) → use(0), path=[1]
>     dfs() used=[T,F,F]:
>       i=0 used → skip
>       i=1 nums[1]=1; dup rule: nums[1]==nums[0] AND used[0]? T → OK to use
>         use(1), path=[1,1]
>         dfs(): i=0 used, i=1 used, i=2 free → path=[1,1,2] → append. pop. (un-use)
>         un-use 1
>       i=2 free → path=[1,2]
>         dfs(): i=0 used, i=1 free, used[0]=T → OK; use → path=[1,2,1] → append. ...
>         un-use 2
>     un-use 0
>   
>   i=1 nums[1]=1; dup rule: nums[1]==nums[0] AND used[0]=F? → SKIP
>     (This is the dedup: if we already explored "use first 1 then ...", we must not also explore "use second 1 first" — it'd duplicate.)
>   
>   i=2 free → path=[2]
>     dfs() used=[F,F,T]:
>       i=0 free → path=[2,1]
>         dfs(): i=1 free, dup rule: nums[1]==nums[0] AND used[0]=T → OK; use → path=[2,1,1] → append.
>       
>       i=1 free; dup rule: used[0]=F → SKIP (dedup)
> 
> ✅ Answer: [[1,1,2], [1,2,1], [2,1,1]]
> ```

> [!success]- JS
> ```js
> const permuteUnique = (nums) => {
>   nums.sort((a, b) => a - b);
>   const out = [], path = [];
>   const used = new Array(nums.length).fill(false);
>   const dfs = () => {
>     if (path.length === nums.length) {
>       out.push([...path]);
>       return;
>     }
>     for (let i = 0; i < nums.length; i++) {
>       if (used[i]) continue;
>       if (i > 0 && nums[i] === nums[i - 1] && !used[i - 1]) continue;
>       used[i] = true;
>       path.push(nums[i]);
>       dfs();
>       path.pop();
>       used[i] = false;
>     }
>   };
>   dfs();
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def permute_unique(nums):
>     nums.sort()
>     out, path = [], []
>     used = [False] * len(nums)
>     def dfs():
>         if len(path) == len(nums):
>             out.append(path[:])
>             return
>         for i in range(len(nums)):
>             if used[i]: continue
>             if i > 0 and nums[i] == nums[i - 1] and not used[i - 1]:
>                 continue
>             used[i] = True
>             path.append(nums[i])
>             dfs()
>             path.pop()
>             used[i] = False
>     dfs()
>     return out
> ```

**Key takeaway:** Permutation dedup rule: `nums[i] == nums[i-1] && !used[i-1]` — force a canonical order on equal values.

---

## P5: Combinations

**LC #77** · Medium

Choose `k` items from `1..n`.

> [!example]- 📊 Visual: combinations decision tree (start-index)
> ```text
>   n = 4, k = 2     pool = [1, 2, 3, 4]
> 
>                            [ ]
>            /         /         \         \
>         pick 1   pick 2     pick 3     pick 4
>           |         |          |          |
>          [1]       [2]        [3]        [4]
>         / | \      / \         |          
>        2  3  4    3   4        4         (nothing to pick after 4)
>        |  |  |    |   |        |
>      [1,2][1,3][1,4][2,3][2,4][3,4]
> 
>   start_index keeps strictly increasing: at depth d picking index i,
>   the next level starts at i+1. This ensures combos (order doesn't matter).
> 
>   Leaves at depth k = 2 → C(4,2) = 6 combinations.
> ```

> [!info]- 🔍 Dry Run: n=4, k=2
> ```text
> dfs(start=1) with path=[]:
>   len(path) == k? 0 != 2
>   i=1: push 1, path=[1]
>     dfs(2): i=2 → path=[1,2] → len 2 → append. pop.
>             i=3 → path=[1,3] → append. pop.
>             i=4 → path=[1,4] → append. pop.
>     pop 1
>   i=2: push 2, path=[2]
>     dfs(3): path=[2,3]→append; path=[2,4]→append
>     pop
>   i=3: push 3, path=[3]
>     dfs(4): path=[3,4]→append
>     pop
>   i=4: push 4, path=[4]
>     dfs(5): no iterations (i=5 > n=4)
>     pop
> 
> ✅ Answer: [[1,2],[1,3],[1,4],[2,3],[2,4],[3,4]]
> ```

> [!success]- JS
> ```js
> const combine = (n, k) => {
>   const out = [], path = [];
>   const dfs = (start) => {
>     if (path.length === k) {
>       out.push([...path]);
>       return;
>     }
>     for (let i = start; i <= n; i++) {
>       path.push(i);
>       dfs(i + 1);
>       path.pop();
>     }
>   };
>   dfs(1);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def combine(n, k):
>     out, path = [], []
>     def dfs(start):
>         if len(path) == k:
>             out.append(path[:])
>             return
>         for i in range(start, n + 1):
>             path.append(i)
>             dfs(i + 1)
>             path.pop()
>     dfs(1)
>     return out
> ```

**Key takeaway:** Combinations = Subsets restricted to size k. Same skeleton with size-check goal.

---

## P6: Combination Sum

**LC #39** · Medium · Reuse allowed

Find all combos summing to target. Each number can be reused.

### 🧠 Pattern: Same Start (Allow Reuse)

> Critical: on the recursive call, pass `i` (not `i+1`) — you may pick the same number again.

> [!example]- 📊 Visual: reuse-allowed tree, recurse on SAME index
> ```text
>   candidates = [2, 3, 6, 7]   target = 7
>   Each node label = remaining target after subtracting picks.
> 
>                       (rem=7)
>          ┌──────────┬──────────┬──────────┐
>        pick 2     pick 3     pick 6     pick 7
>         (5)        (4)        (1)        (0) ✓ → [7]
>         │           │          │
>      ┌──┼──┐     ┌──┼──┐    (prune: 6>1, 7>1)
>    pick2 3  ...  pick3 ...
>     (3)  (2)     (1)
>      │    │       (prune)
>     pick2 prune
>     (1)
>      prune
> 
>     pick2 again from rem=3:
>       → rem=1 → prune
>     pick3 from rem=3:
>       → rem=0 ✓ → [2, 2, 3]
> 
>   KEY: at "pick X" we recurse with start = i (NOT i+1),
>        which keeps the same X available next level (reuse).
>        Going only forward (no backwards picks) avoids order-permutations.
> 
>   Result: [[2,2,3], [7]]
> ```

> [!info]- 🔍 Dry Run: candidates=[2,3,6,7], target=7
> ```text
> Sort: [2,3,6,7]
> 
> dfs(i=0, remain=7) path=[]:
>   j=0 c=2: push 2, path=[2]
>     dfs(0, 5):       ← pass j=0 (reuse allowed)
>       j=0 c=2: push, path=[2,2]
>         dfs(0, 3):
>           j=0 c=2: push, path=[2,2,2]
>             dfs(0, 1):
>               j=0 c=2 > 1 → break (pruned due to sort)
>             pop
>           j=1 c=3: push, path=[2,2,3]
>             dfs(1, 0): remain==0 → append [2,2,3]
>             pop
>           j=2 c=6 > 3 → break
>         pop
>       j=1 c=3: push, path=[2,3]
>         dfs(1, 2):
>           j=1 c=3 > 2 → break
>         pop
>       j=2,3 too big → break
>     pop
>   j=1 c=3: push, path=[3]
>     dfs(1, 4):
>       j=1 c=3: push, path=[3,3]
>         dfs(1, 1): j=1 c=3 > 1 → break
>         pop
>       j=2 c=6 > 4 → break
>     pop
>   j=2 c=6: push, path=[6]
>     dfs(2, 1): j=2 c=6 > 1 break
>     pop
>   j=3 c=7: push, path=[7]
>     dfs(3, 0): remain==0 → append [7]
>     pop
> 
> ✅ Answer: [[2,2,3], [7]]
> ```

> [!success]- JS
> ```js
> const combinationSum = (candidates, target) => {
>   candidates.sort((a, b) => a - b);
>   const out = [], path = [];
>   const dfs = (i, remain) => {
>     if (remain === 0) {
>       out.push([...path]);
>       return;
>     }
>     for (let j = i; j < candidates.length; j++) {
>       if (candidates[j] > remain) break;   // pruned by sort
>       path.push(candidates[j]);
>       dfs(j, remain - candidates[j]);       // j, not j+1
>       path.pop();
>     }
>   };
>   dfs(0, target);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def combination_sum(candidates, target):
>     out, path = [], []
>     candidates.sort()
>     def dfs(i, remain):
>         if remain == 0:
>             out.append(path[:])
>             return
>         for j in range(i, len(candidates)):
>             if candidates[j] > remain: break   # pruned by sort
>             path.append(candidates[j])
>             dfs(j, remain - candidates[j])      # j, not j+1
>             path.pop()
>     dfs(0, target)
>     return out
> ```

**Key takeaway:** Reuse allowed → pass `j` in recursion (not `j+1`). Sort + break = early prune.

---

## P7: Combination Sum II

**LC #40** · Medium · No reuse, with duplicates

> [!example]- 📊 Visual: no-reuse (i+1) + same-level dup skip
> ```text
>   candidates = [1, 1, 2, 5, 6, 7, 10]  (sorted)   target = 8
> 
>                            (rem=8)
>            ┌────────┬────────┬────────┬─── ...
>          pick 1[0]  pick 1[1]✗  pick 2  pick 5  ...
>            (7)      SKIP        (6)     (3)
>                  same level                
>                  dup '1'
>                  
>   At rem=7, start=1 (no reuse → i+1 from index 0 → start at 1):
>            ┌────────┬────────┬────────┐
>          pick 1[1]  pick 2   pick 5   pick 7
>            (6)       (5)      (2)      (0) ✓ [1,7]
>          (different lvl,
>           NOT a skip)
>   
>   Two skip rules combined:
>     (1) no reuse  →  pass i+1 in recursion
>     (2) dup skip  →  if j > start AND candidates[j] == candidates[j-1] → skip
>     (3) sort prune → if candidates[j] > rem → break
> 
>   Result: [[1,1,6], [1,2,5], [1,7], [2,6]]
> ```

> [!info]- 🔍 Dry Run: candidates=[10,1,2,7,6,1,5], target=8
> ```text
> Sort: [1,1,2,5,6,7,10]
> 
> dfs(i=0, remain=8) path=[]:
>   j=0 c=1: push, path=[1]
>     dfs(1, 7):
>       j=1 c=1 (dup check: j>i=1? 1>1 NO → don't skip; use): push, path=[1,1]
>         dfs(2, 6):
>           j=2 c=2: push, path=[1,1,2]
>             dfs(3, 4): j=3 c=5 > 4 break; pop
>           j=3 c=5: push, path=[1,1,5]
>             dfs(4, 1): break (6>1) pop
>           j=4 c=6: push, path=[1,1,6]
>             dfs(5, 0): append [1,1,6]; pop
>           j=5 c=7 > 6 break
>         pop
>       j=2 c=2: push, path=[1,2]
>         dfs(3, 5):
>           j=3 c=5: push, path=[1,2,5]
>             dfs(4, 0): append [1,2,5]; pop
>         pop
>       j=3 c=5: push, path=[1,5]
>         dfs(4, 2): break (6>2); pop
>       j=4 c=6: push, path=[1,6]
>         dfs(5, 1): break (7>1); pop
>       j=5 c=7: push, path=[1,7]
>         dfs(6, 0): append [1,7]; pop
>     pop
>   j=1 c=1 (dup check: 1>0? YES, nums[1]==nums[0] → SKIP)
>   j=2 c=2: push, path=[2]
>     dfs(3, 6):
>       j=3 c=5: push, path=[2,5]; dfs(4,1) break; pop
>       j=4 c=6: push, path=[2,6]; dfs(5,0) → append [2,6]; pop
>     pop
>   ... (continue with j=3,4,5,6; some yield more combos)
> 
> ✅ Answer: [[1,1,6], [1,2,5], [1,7], [2,6]]
> ```

> [!success]- JS
> ```js
> const combinationSum2 = (candidates, target) => {
>   candidates.sort((a, b) => a - b);
>   const out = [], path = [];
>   const dfs = (i, remain) => {
>     if (remain === 0) {
>       out.push([...path]);
>       return;
>     }
>     for (let j = i; j < candidates.length; j++) {
>       if (j > i && candidates[j] === candidates[j - 1]) continue;
>       if (candidates[j] > remain) break;
>       path.push(candidates[j]);
>       dfs(j + 1, remain - candidates[j]);
>       path.pop();
>     }
>   };
>   dfs(0, target);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def combination_sum2(candidates, target):
>     candidates.sort()
>     out, path = [], []
>     def dfs(i, remain):
>         if remain == 0:
>             out.append(path[:])
>             return
>         for j in range(i, len(candidates)):
>             if j > i and candidates[j] == candidates[j - 1]: continue
>             if candidates[j] > remain: break
>             path.append(candidates[j])
>             dfs(j + 1, remain - candidates[j])
>             path.pop()
>     dfs(0, target)
>     return out
> ```

**Key takeaway:** No reuse → `j+1`. Plus same-level dup skip.

---

## P8: Letter Combinations of a Phone Number

**LC #17** · Medium

`"23"` → `["ad","ae","af","bd","be","bf","cd","ce","cf"]`.

> [!example]- 📊 Visual: Cartesian product via DFS
> ```text
>   digits = "23"
>     2 → "abc"
>     3 → "def"
> 
>                         [ ]
>                 ┌────────┼────────┐
>                 a        b        c        ← choose letter for digit '2'
>                /|\      /|\      /|\
>               d e f    d e f    d e f      ← choose letter for digit '3'
>               | | |    | | |    | | |
>             ad ae af  bd be bf  cd ce cf
> 
>   Depth = len(digits).
>   Branching at depth d = |letters of digits[d]|.
>   Total leaves = product of branch factors  =  3 × 3  = 9.
> ```

> [!info]- 🔍 Dry Run: digits="23"
> ```text
> Map: 2→"abc", 3→"def"
> 
> dfs(i=0) with path=[]:
>   c='a': path=['a']
>     dfs(1):
>       c='d': path=['a','d'] → append "ad"; pop
>       c='e': → append "ae"; pop
>       c='f': → append "af"; pop
>     pop 'a'
>   c='b':
>     dfs(1): → "bd","be","bf"
>   c='c':
>     → "cd","ce","cf"
> 
> ✅ Answer: ["ad","ae","af","bd","be","bf","cd","ce","cf"]
> ```

> [!success]- JS
> ```js
> const letterCombinations = (digits) => {
>   if (!digits) return [];
>   const mp = { '2':'abc','3':'def','4':'ghi','5':'jkl','6':'mno','7':'pqrs','8':'tuv','9':'wxyz' };
>   const out = [], path = [];
>   const dfs = (i) => {
>     if (i === digits.length) {
>       out.push(path.join(''));
>       return;
>     }
>     for (const c of mp[digits[i]]) {
>       path.push(c);
>       dfs(i + 1);
>       path.pop();
>     }
>   };
>   dfs(0);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def letter_combinations(digits):
>     if not digits: return []
>     mp = {'2':'abc','3':'def','4':'ghi','5':'jkl','6':'mno','7':'pqrs','8':'tuv','9':'wxyz'}
>     out, path = [], []
>     def dfs(i):
>         if i == len(digits):
>             out.append(''.join(path))
>             return
>         for c in mp[digits[i]]:
>             path.append(c)
>             dfs(i + 1)
>             path.pop()
>     dfs(0)
>     return out
> ```

**Key takeaway:** Cartesian product via DFS. Position-by-position commit.

---

## P9: Word Search

**LC #79** · Medium

DFS on a grid; visit each cell at most once per path.

### Approach

For each starting cell, DFS through the word. Mark visited by temporarily mutating the cell; restore on backtrack.

> [!example]- 📊 Visual: trace the word along the grid
> ```text
>   word = "ABCCED"
> 
>   Board:                     Path (numbered = order of visits):
>     A  B  C  E                  ①  ②  ③  .
>     S  F  C  S                  .  .  ④  .
>     A  D  E  E                  .  ⑥  ⑤  .
> 
>     ┌──────────────────────────────────┐
>     │  A → B → C   → C   → E   → D     │
>     │ (0,0)(0,1)(0,2)(1,2)(2,2)(2,1)   │
>     │  ①   ②   ③    ④    ⑤    ⑥      │
>     └──────────────────────────────────┘
> 
>   Mark each visited cell with '#' during recursion; restore on backtrack:
> 
>     #  #  #  E       ← in the middle of the search
>     S  F  #  S
>     A  #  #  E
> 
>   At each step: 4 neighbors (up/down/left/right) → DFS branch.
>   Mismatched char or out-of-bounds → prune.
>   i == len(word) → success.
> ```

> [!info]- 🔍 Dry Run: board=[["A","B","C","E"],["S","F","C","S"],["A","D","E","E"]], word="ABCCED"
> ```text
> Try every cell as start until match found.
> 
> Start (0,0) board='A' == word[0]='A' ✓
>   mark (0,0) = '#'; board=[["#","B","C","E"], ...]
>   dfs(1,0,1): board[1][0]='S' != word[1]='B' → false
>   dfs(-1,0,1): out of bounds → false
>   dfs(0,1,1): board[0][1]='B' == word[1]='B' ✓
>     mark (0,1) = '#'
>     dfs(1,1,2): board[1][1]='F' != 'C' → false
>     dfs(-1,1,2): oob → false
>     dfs(0,0,2): board[0][0]='#' != 'C' → false
>     dfs(0,2,2): board[0][2]='C' == 'C' ✓
>       mark (0,2)
>       dfs(1,2,3): board[1][2]='C' == word[3]='C' ✓
>         mark (1,2)
>         dfs(2,2,4): board[2][2]='E' == 'E' ✓
>           mark (2,2)
>           dfs(*,*,5) for 'D':
>             (2,1): board[2][1]='D' == 'D' ✓
>               mark
>               dfs(*,*,6): i==6==len(word), return TRUE
>             unmark; return TRUE
>           unmark; return TRUE
>         unmark; return TRUE
>       unmark; return TRUE
>     unmark
>   unmark
> 
> ✅ Answer: true   (path: (0,0)→(0,1)→(0,2)→(1,2)→(2,2)→(2,1))
> ```

> [!success]- JS
> ```js
> const exist = (board, word) => {
>   const R = board.length, C = board[0].length;
>   const dfs = (r, c, i) => {
>     if (i === word.length) return true;
>     if (r < 0 || r >= R || c < 0 || c >= C || board[r][c] !== word[i]) return false;
>     const saved = board[r][c];
>     board[r][c] = '#';
>     const found = dfs(r + 1, c, i + 1) || dfs(r - 1, c, i + 1)
>                || dfs(r, c + 1, i + 1) || dfs(r, c - 1, i + 1);
>     board[r][c] = saved;
>     return found;
>   };
>   for (let r = 0; r < R; r++) {
>     for (let c = 0; c < C; c++) {
>       if (dfs(r, c, 0)) return true;
>     }
>   }
>   return false;
> };
> ```

> [!success]- Python
> ```python
> def exist(board, word):
>     R, C = len(board), len(board[0])
>     def dfs(r, c, i):
>         if i == len(word): return True
>         if r < 0 or r >= R or c < 0 or c >= C or board[r][c] != word[i]: return False
>         board[r][c] = '#'
>         found = (dfs(r+1,c,i+1) or dfs(r-1,c,i+1)
>                  or dfs(r,c+1,i+1) or dfs(r,c-1,i+1))
>         board[r][c] = word[i]
>         return found
>     for r in range(R):
>         for c in range(C):
>             if dfs(r, c, 0): return True
>     return False
> ```

**Key takeaway:** Grid backtracking → mutate-to-mark, restore-on-return.

---

## P10: Palindrome Partitioning

**LC #131** · Medium

Partition `s` so every piece is a palindrome; return all partitions.

### Approach

For each cut point: if `s[i..j]` is a palindrome, include and recurse on `j+1`.

> [!example]- 📊 Visual: cut-point decision tree
> ```text
>   s = "aab"
> 
>   At each i, try every j ≥ i; if s[i..j] is a palindrome, take it.
> 
>                             dfs(0)            s = "aab"
>              ┌──────────────┴─────────────┐
>          take "a"                      take "aa"
>          (s[0:1] ✓)                    (s[0:2] ✓)
>          dfs(1)                        dfs(2)
>          ┌─────┴─────┐                 │
>       take "a"   take "ab"           take "b"
>       (s[1:2] ✓) (s[1:3] ✗ skip)     (s[2:3] ✓)
>       dfs(2)                          dfs(3) ✓ → ["aa","b"]
>          │
>       take "b"
>       (s[2:3] ✓)
>       dfs(3) ✓ → ["a","a","b"]
> 
>   String laid out:    a   a   b
>                       │   │   │
>   Cut markers:    |   ▲   ▲   ▲   |       (between chars)
>                   0   1   2   3
>   A partition = choose a subset of cut markers; every piece between
>   consecutive chosen cuts must be a palindrome.
> 
>   Result: [["a","a","b"], ["aa","b"]]
> ```

> [!info]- 🔍 Dry Run: s="aab"
> ```text
> dfs(i=0) path=[]:
>   j=0: s[0:1]="a" palindrome ✓ → path=["a"]
>     dfs(1):
>       j=1: s[1:2]="a" pal ✓ → path=["a","a"]
>         dfs(2):
>           j=2: s[2:3]="b" pal ✓ → path=["a","a","b"]
>             dfs(3): i==len(s) → append ["a","a","b"]; return
>           pop
>         (loop done)
>       pop
>       j=2: s[1:3]="ab" pal? a!=b → no, skip
>     pop
>   j=1: s[0:2]="aa" pal ✓ → path=["aa"]
>     dfs(2):
>       j=2: s[2:3]="b" ✓ → path=["aa","b"]
>         dfs(3): append ["aa","b"]; return
>       pop
>     pop
>   j=2: s[0:3]="aab" pal? a!=b skip
> 
> ✅ Answer: [["a","a","b"], ["aa","b"]]
> ```

> [!success]- JS
> ```js
> const partition = (s) => {
>   const out = [], path = [];
>   const isPal = (l, r) => {
>     while (l < r) {
>       if (s[l] !== s[r]) return false;
>       l++; r--;
>     }
>     return true;
>   };
>   const dfs = (i) => {
>     if (i === s.length) {
>       out.push([...path]);
>       return;
>     }
>     for (let j = i; j < s.length; j++) {
>       if (isPal(i, j)) {
>         path.push(s.slice(i, j + 1));
>         dfs(j + 1);
>         path.pop();
>       }
>     }
>   };
>   dfs(0);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def partition(s):
>     out, path = [], []
>     def is_pal(l, r):
>         while l < r:
>             if s[l] != s[r]: return False
>             l += 1; r -= 1
>         return True
>     def dfs(i):
>         if i == len(s):
>             out.append(path[:])
>             return
>         for j in range(i, len(s)):
>             if is_pal(i, j):
>                 path.append(s[i:j + 1])
>                 dfs(j + 1)
>                 path.pop()
>     dfs(0)
>     return out
> ```

**Key takeaway:** "All partitions where each part is X" → backtrack over cut points + validate.

---

## P11: N-Queens

**LC #51** · **Hard**

### 🧠 Pattern: Three Constraint Sets

> Place queens row by row. Track used columns + two diagonal sets:
> - **Column index `c`**
> - **`r + c`** (anti-diagonal, "/")
> - **`r - c`** (main diagonal, "\")
> 
> Adding a queen at `(r, c)` is valid iff `c`, `r+c`, `r-c` are unused.

> [!example]- 📊 Visual: diagonal identities
> ```text
>   Anti-diagonal (/) — constant r+c:
> 
>     c=0  c=1  c=2  c=3
>     ─────────────────
>     r+c=0  1   2   3       r=0
>     r+c=1  2   3   4       r=1
>     r+c=2  3   4   5       r=2
>     r+c=3  4   5   6       r=3
> 
>     Each anti-diagonal has a unique r+c value.
>     If queen at (1,2) has r+c=3, any other (r,c) with r+c=3 attacks it.
> 
>   Main diagonal (\) — constant r-c:
> 
>     c=0  c=1  c=2  c=3
>     ─────────────────
>     r-c= 0  -1  -2  -3     r=0
>     r-c= 1   0  -1  -2     r=1
>     r-c= 2   1   0  -1     r=2
>     r-c= 3   2   1   0     r=3
> 
>   So for any new queen at (r, c):
>     check that column c is unused
>     check that r+c is unused
>     check that r-c is unused
>   If all three pass → safe to place.
> 
>   One of the two valid 4-Queens placements:
> 
>     . Q . .       cols     = {0:_, 1:Q@row 0, 2:_, 3:_}
>     . . . Q       r+c set  = {0+1=1, 1+3=4, 2+0=2, 3+2=5}
>     Q . . .       r-c set  = {0-1=-1, 1-3=-2, 2-0=2, 3-2=1}
>     . . Q .
> ```

> [!info]- 🔍 Dry Run: n=4
> ```text
> dfs(r=0) with empty board, cols={}, d1={}, d2={}:
>   try c=0: place Q at (0,0). cols={0}, d1={0}, d2={0}
>     dfs(1):
>       try c=0: 0 in cols → skip
>       try c=1: 1 in cols? NO. 1+1=2 in d1? NO. 1-1=0 in d2? YES → skip
>       try c=2: 2 free, 1+2=3 free, 1-2=-1 free → place. cols={0,2}, d1={0,3}, d2={0,-1}
>         dfs(2):
>           try c=0: 0 in cols → skip
>           try c=1: 2+1=3 in d1 → skip
>           try c=2: in cols
>           try c=3: 2+3=5 free, 2-3=-1 in d2 → skip
>           → no valid; backtrack
>         (un-place 2 at (1,2))
>       try c=3: 1+3=4 free, 1-3=-2 free → place. cols={0,3}, d1={0,4}, d2={0,-2}
>         dfs(2):
>           c=0,3 in cols
>           c=1: 2+1=3 free, 2-1=1 free → place; cols={0,1,3}, d1={0,3,4}, d2={0,-2,1}
>             dfs(3):
>               c=2: 3+2=5 free, 3-2=1 in d2 → skip
>               others all blocked → backtrack
>           c=2: 2+2=4 in d1 → skip
>         (backtrack all)
>     (no valid → backtrack outer)
>   try c=1: place at (0,1). cols={1}, d1={1}, d2={-1}
>     dfs(1):
>       c=3: 1+3=4 free, 1-3=-2 free → place; cols={1,3}, d1={1,4}, d2={-1,-2}
>         dfs(2):
>           c=0: 2+0=2 free, 2-0=2 free → place; cols={0,1,3}, d1={1,2,4}, d2={-1,-2,2}
>             dfs(3):
>               c=2: 3+2=5 free, 3-2=1 in d1 → skip
>               others blocked
>             (backtrack)
>           (un-place 0)
>           c=2: 2+2=4 in d1 → skip
>         (backtrack)
>       (un-place 3)
>     ... eventually find solution ".Q..","...Q","Q...",".Q.." wait that has duplicate column
>     
> Solutions found (n=4):
>   .Q..      ..Q.
>   ...Q      Q...
>   Q...      ...Q
>   ..Q.      .Q..
> 
> ✅ Answer: 2 valid placements
> ```

> [!success]- JS
> ```js
> const solveNQueens = (n) => {
>   const out = [];
>   const cols = new Set(), diag1 = new Set(), diag2 = new Set();
>   const board = Array.from({ length: n }, () => Array(n).fill('.'));
>   const dfs = (r) => {
>     if (r === n) {
>       out.push(board.map((row) => row.join('')));
>       return;
>     }
>     for (let c = 0; c < n; c++) {
>       if (cols.has(c) || diag1.has(r + c) || diag2.has(r - c)) continue;
>       cols.add(c); diag1.add(r + c); diag2.add(r - c);
>       board[r][c] = 'Q';
>       dfs(r + 1);
>       cols.delete(c); diag1.delete(r + c); diag2.delete(r - c);
>       board[r][c] = '.';
>     }
>   };
>   dfs(0);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def solve_n_queens(n):
>     out = []
>     cols, diag1, diag2 = set(), set(), set()
>     board = [['.'] * n for _ in range(n)]
>     def dfs(r):
>         if r == n:
>             out.append([''.join(row) for row in board])
>             return
>         for c in range(n):
>             if c in cols or (r + c) in diag1 or (r - c) in diag2: continue
>             cols.add(c); diag1.add(r + c); diag2.add(r - c)
>             board[r][c] = 'Q'
>             dfs(r + 1)
>             cols.remove(c); diag1.remove(r + c); diag2.remove(r - c)
>             board[r][c] = '.'
>     dfs(0)
>     return out
> ```

**Variants:** N-Queens II (count solutions).

**Key takeaway:** Encode each constraint as a set keyed by an arithmetic identity (`r+c`, `r-c`).

---

## P12: Restore IP Addresses

**LC #93** · Medium

Insert 3 dots into `s` so each segment is 0..255 (no leading zeros).

> [!example]- 📊 Visual: 4-level bounded segmentation tree
> ```text
>   s = "25525511135"   (len 11; need exactly 4 segments)
> 
>   At each level, take length 1, 2, or 3 chars (segment 0..255).
> 
>            dfs(start=0, parts=[])
>      ┌─────────┬─────────┬─────────┐
>   len=1 "2"  len=2 "25" len=3 "255"
>     ↓          ↓           ↓
>    ...        ...        dfs(start=3, parts=["255"])
>                          ┌──────┬──────┬──────┐
>                       "2"      "25"   "255"
>                        ↓        ↓       ↓
>                      ...      dfs(5, ["255","25"])
>                                  ↓
>                              ... (continue)
> 
>   Valid path found:
>      "255" . "255" . "11"  . "135"
>     [0..3) [3..6) [6..8) [8..11)   ← exactly fills s
>     │      │      │       │
>     part 1 part 2 part 3  part 4
> 
>   Pruning:
>     • segment > 255  →  skip
>     • leading '0' (and length > 1)  →  skip
>     • when parts == 4: must have start == len(s); else fail
>     • start + length > len(s)  →  break (no more chars)
> 
>   Result: ["255.255.11.135", "255.255.111.35"]
> ```

> [!info]- 🔍 Dry Run: s="25525511135"
> ```text
> dfs(start=0, parts=[]):
>   try length=1: seg="2", ok (2<=255) → dfs(1, ["2"])
>     try length=1: "5" → dfs(2, ["2","5"])
>       try length=1: "5" → dfs(3, ["2","5","5"])
>         len(parts)==3; remaining is "25511135" → too long for one part (max 3 chars) → skip if length 4+
>         try lengths 1..3:
>           length=1: "2" → not last 4th part, only 4 parts means we need start+length==len(s)
>             Actually rule: len(parts)==4 must mean start==len(s).
>             So at this stage we recurse to dfs(4, ["2","5","5","2"]); parts len=4 but start=4 != len("25525511135")=11 → not valid; continue.
>           length=2: "25" → dfs(5, ["2","5","5","25"]); len=4 but start=5 != 11 skip
>           length=3: "255" → dfs(6, ...); start=6 != 11 skip
>           (all fail)
>         backtrack
>       length=2: "52" → dfs(...); etc.
>       length=3: "525" → 525 > 255 SKIP
>     (similar for other branches)
> 
> Eventually find valid:
>   "255.255.11.135"
>   "255.255.111.35"
> 
> ✅ Answer: ["255.255.11.135", "255.255.111.35"]
> ```

> [!success]- JS
> ```js
> const restoreIpAddresses = (s) => {
>   const out = [];
>   const dfs = (start, parts) => {
>     if (parts.length === 4) {
>       if (start === s.length) out.push(parts.join('.'));
>       return;
>     }
>     for (const length of [1, 2, 3]) {
>       if (start + length > s.length) break;
>       const seg = s.slice(start, start + length);
>       if ((seg.startsWith('0') && length > 1) || Number(seg) > 255) continue;
>       dfs(start + length, [...parts, seg]);
>     }
>   };
>   dfs(0, []);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def restore_ip_addresses(s):
>     out = []
>     def dfs(start, parts):
>         if len(parts) == 4:
>             if start == len(s):
>                 out.append('.'.join(parts))
>             return
>         for length in (1, 2, 3):
>             if start + length > len(s): break
>             seg = s[start:start + length]
>             if (seg.startswith('0') and length > 1) or int(seg) > 255: continue
>             dfs(start + length, parts + [seg])
>     dfs(0, [])
>     return out
> ```

**Key takeaway:** Bounded segmentation → DFS with length 1..k. Validate at each step.

---

## P13: Sudoku Solver

**LC #37** · **Hard**

Fill a 9×9 Sudoku grid in place. Standard rules (each row, col, 3×3 box has digits 1-9 exactly once).

### 🧠 Pattern: DFS + Three Constraint Sets

> Track used digits per row, per column, per 3×3 box. For each empty cell, try digits 1-9 that aren't in any of its row/col/box sets. Recurse.

> [!example]- 📊 Visual: three constraint sets per cell
> ```text
>   For cell (i, j), the three sets it must avoid:
> 
>        col j
>          │
>    ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─ ─       ┌─────┐         ┌─────┐
>    . . X . . . . . .            │ . . │ . . .  │ . . │
>    . . X . . . . . .            │ . . │ . . .  │ . . │
>    X X * X X X X X X  ← row i   │ . * │ . . .  │ . . │  ← box (i//3, j//3)
>    . . X . . . . . .            └─────┘         └─────┘
>    . . X . . . . . .            (3x3 block
>                                   containing *)
> 
>   For cell *:
>     rows[i]  = digits already in row i
>     cols[j]  = digits already in col j
>     boxes[b] = digits already in the box b = (i//3)*3 + (j//3)
> 
>   Candidate digit d is valid ⟺ d ∉ rows[i] ∧ d ∉ cols[j] ∧ d ∉ boxes[b]
> 
>   DFS over the list of empty cells:
>     for d in '1'..'9':
>       if all three sets allow d:
>         place d; add to 3 sets
>         recurse on next empty cell
>         if success → return
>         else: remove from 3 sets; restore '.'  (backtrack)
> 
>   The three sets are the "ledger"; updates and rollbacks mirror exactly.
> ```

> [!info]- 🔍 Dry Run (conceptual; trace for one cell)
> ```text
> Setup:
>   rows[i] = set of digits used in row i
>   cols[j] = set of digits used in column j
>   box[k] = set used in box k where k = (i//3)*3 + (j//3)
>   empties = list of (i, j) for each '.' cell
> 
> Pre-populate sets from filled cells.
> 
> ─────────────────────────────────────────
> dfs(empty_idx=0):
>   (i, j) = first empty cell, say (0, 2)
>   box_id = (0//3)*3 + (2//3) = 0
> 
>   for d in '1'..'9':
>     try d='1':
>       Is '1' in rows[0]?     check row 0
>       Is '1' in cols[2]?
>       Is '1' in box[0]?
>       If ANY → skip
>       
>       Suppose all clear → place '1':
>         board[0][2] = '1'
>         add to rows[0], cols[2], box[0]
>         dfs(empty_idx+1):
>           if it succeeds, return true
>           else, un-place: remove from sets, restore '.'
>     try d='2': ... similar
> 
>   If no d worked → return false (backtrack)
> 
> ─────────────────────────────────────────
> Base case: empty_idx == len(empties) → all cells filled → return true
> 
> Example: starting from a near-complete board, only one digit fits at each remaining cell → linear; harder boards require deep backtracking but the algorithm is the same.
> 
> ✅ Final state: board is filled; sets are consistent.
> ```

> [!success]- JS
> ```js
> const solveSudoku = (board) => {
>   const rows = Array.from({ length: 9 }, () => new Set());
>   const cols = Array.from({ length: 9 }, () => new Set());
>   const boxes = Array.from({ length: 9 }, () => new Set());
>   const empties = [];
>   for (let i = 0; i < 9; i++) {
>     for (let j = 0; j < 9; j++) {
>       const c = board[i][j];
>       if (c === '.') {
>         empties.push([i, j]);
>       } else {
>         rows[i].add(c);
>         cols[j].add(c);
>         boxes[Math.floor(i / 3) * 3 + Math.floor(j / 3)].add(c);
>       }
>     }
>   }
>   const dfs = (idx) => {
>     if (idx === empties.length) return true;
>     const [i, j] = empties[idx];
>     const b = Math.floor(i / 3) * 3 + Math.floor(j / 3);
>     for (const d of "123456789") {
>       if (rows[i].has(d) || cols[j].has(d) || boxes[b].has(d)) continue;
>       board[i][j] = d;
>       rows[i].add(d); cols[j].add(d); boxes[b].add(d);
>       if (dfs(idx + 1)) return true;
>       rows[i].delete(d); cols[j].delete(d); boxes[b].delete(d);
>       board[i][j] = '.';
>     }
>     return false;
>   };
>   dfs(0);
> };
> ```

> [!success]- Python
> ```python
> def solve_sudoku(board):
>     rows = [set() for _ in range(9)]
>     cols = [set() for _ in range(9)]
>     boxes = [set() for _ in range(9)]
>     empties = []
>     for i in range(9):
>         for j in range(9):
>             c = board[i][j]
>             if c == '.':
>                 empties.append((i, j))
>             else:
>                 rows[i].add(c)
>                 cols[j].add(c)
>                 boxes[(i // 3) * 3 + (j // 3)].add(c)
>     def dfs(idx):
>         if idx == len(empties): return True
>         i, j = empties[idx]
>         b = (i // 3) * 3 + (j // 3)
>         for d in "123456789":
>             if d in rows[i] or d in cols[j] or d in boxes[b]: continue
>             board[i][j] = d
>             rows[i].add(d); cols[j].add(d); boxes[b].add(d)
>             if dfs(idx + 1): return True
>             rows[i].remove(d); cols[j].remove(d); boxes[b].remove(d)
>             board[i][j] = '.'
>         return False
>     dfs(0)
> ```

> [!tip] MRV heuristic
> For harder puzzles, sort `empties` by the cell with the fewest legal candidates first → prune drastically faster. Optional for interview.

**Variants:** Valid Sudoku (just check current state) · 9x9 KenKen (constraint propagation).

**Key takeaway:** Constraint satisfaction → backtracking with explicit constraint sets. Updates and rollbacks must mirror.

---

> [!tip] After this drill
> Memorize the **choose → recurse → unchoose** skeleton. Most variations are: where to start (start-index vs used-set), how to dedup (sort + same-level skip), and what's the goal (size vs sum vs full).
