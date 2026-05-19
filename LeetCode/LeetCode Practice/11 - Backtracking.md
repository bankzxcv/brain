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

12 problems · subsets, permutations, combinations, and constraint-satisfaction.

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
| P11 | N-Queens | 51 | Hard | Column + diagonal sets |
| P12 | Restore IP Addresses | 93 | Med | Bounded segmentation |

---

## P1: Subsets

**LC #78** · Medium

All subsets of distinct `nums`.

### 🧠 Pattern: Include / Exclude via Start Index

> At index `i`, snapshot the current path, then for each `j ≥ i`, include `nums[j]` and recurse with `j+1`.

### Trace

```
nums=[1,2,3]
path=[]      → [[]]
  include 1, path=[1]    → [[],[1]]
    include 2, path=[1,2]   → [[],[1],[1,2]]
      include 3, path=[1,2,3] → ...,[1,2,3]
      pop 3
    pop 2
    include 3, path=[1,3]
    ...
```

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

> [!tip] Pruning
> `range(start, n + 1)` can be tightened to `range(start, n - (k - len(path)) + 2)` — stop when not enough remaining slots.

**Key takeaway:** Combinations = Subsets restricted to size k. Same skeleton with size-check goal.

---

## P6: Combination Sum

**LC #39** · Medium · Reuse allowed

Find all combos summing to target. Each number can be reused.

### 🧠 Pattern: Same Start (Allow Reuse)

> Critical: on the recursive call, pass `i` (not `i+1`) — you may pick the same number again.

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

> [!tip] Speedup
> Precompute `pal[i][j]` table with DP for O(1) palindrome check.

**Key takeaway:** "All partitions where each part is X" → backtrack over cut points + validate.

---

## P11: N-Queens

**LC #51** · Hard

### 🧠 Pattern: Three Constraint Sets

> Place queens row by row. Track used columns + two diagonal sets:
> - **Column index `c`**
> - **`r + c`** (anti-diagonal, "/")
> - **`r - c`** (main diagonal, "\")
> 
> Adding a queen at `(r, c)` is valid iff `c`, `r+c`, `r-c` are unused.

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

### Approach

4 segments. Backtrack choosing segment length 1, 2, or 3. Validate (no leading 0, ≤ 255).

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

> [!tip] After this drill
> Memorize the **choose → recurse → unchoose** skeleton. Most variations are: where to start (start-index vs used-set), how to dedup (sort + same-level skip), and what's the goal (size vs sum vs full).
