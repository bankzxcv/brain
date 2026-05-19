---
title: "LeetCode Practice: 2D Dynamic Programming"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - dp
  - dynamic-programming
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: 2D Dynamic Programming

12 problems · grid paths, LCS, edit distance, palindromic substrings.

> [!abstract] Pattern recap
> 2D DP arises with **two indices** in the state:
> - **Two sequences** (LCS, edit distance, distinct subsequences)
> - **Grid** (unique paths, minimum path sum)
> - **Substring (i..j)** (palindromes, burst balloons)
> 
> ```
> # Generic 2D template:
> dp = [[0] * (n + 1) for _ in range(m + 1)]
> for i in ...:
>     for j in ...:
>         dp[i][j] = f(dp[i-1][j], dp[i][j-1], dp[i-1][j-1], a[i], b[j])
> return dp[m][n]
> ```

> [!tip] Space optimization
> Most 2D DPs use only the previous row → reduce to `dp[2][n]` rolling or even `dp[n]` with care. Save for later — get correctness first.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Unique Paths | 62 | Med | Grid count |
| P2 | Unique Paths II | 63 | Med | Grid + obstacles |
| P3 | Minimum Path Sum | 64 | Med | Grid min |
| P4 | Longest Common Subsequence | 1143 | Med | Two-string DP |
| P5 | Edit Distance | 72 | Med | Two-string DP w/ 3 ops |
| P6 | Longest Palindromic Substring | 5 | Med | Substring DP |
| P7 | Longest Palindromic Subsequence | 516 | Med | Substring DP |
| P8 | Distinct Subsequences | 115 | Hard | Two-string count |
| P9 | Interleaving String | 97 | Med | Two-string boolean |
| P10 | Maximal Square | 221 | Med | Grid min of neighbors |
| P11 | Best Time to Buy and Sell Stock IV | 188 | Hard | State × time |
| P12 | Burst Balloons | 312 | Hard | Interval DP |

---

## P1: Unique Paths

**LC #62** · Medium

Count paths in m×n grid from top-left to bottom-right, moving only right/down.

### Approach

`dp[i][j] = dp[i-1][j] + dp[i][j-1]`. Base: row 0 and col 0 are all 1.

> [!success]- Python
> ```python
> def unique_paths(m, n):
>     dp = [[1] * n for _ in range(m)]
>     for i in range(1, m):
>         for j in range(1, n):
>             dp[i][j] = dp[i-1][j] + dp[i][j-1]
>     return dp[m-1][n-1]
> ```

> [!tip] Combinatorial shortcut
> Answer = C(m+n-2, m-1). Useful sanity check.

**Key takeaway:** Grid path = sum of "from up" + "from left". The canonical 2D DP.

---

## P2: Unique Paths II

**LC #63** · Medium · Obstacles

> [!success]- Python
> ```python
> def unique_paths_with_obstacles(grid):
>     m, n = len(grid), len(grid[0])
>     dp = [[0] * n for _ in range(m)]
>     dp[0][0] = 0 if grid[0][0] else 1
>     for i in range(m):
>         for j in range(n):
>             if grid[i][j]: dp[i][j] = 0
>             elif i == 0 and j == 0: continue
>             else:
>                 dp[i][j] = (dp[i-1][j] if i else 0) + (dp[i][j-1] if j else 0)
>     return dp[m-1][n-1]
> ```

**Key takeaway:** Same recurrence; obstacles zero out cells.

---

## P3: Minimum Path Sum

**LC #64** · Medium

> [!success]- Python
> ```python
> def min_path_sum(grid):
>     m, n = len(grid), len(grid[0])
>     for i in range(m):
>         for j in range(n):
>             if i == 0 and j == 0: continue
>             up = grid[i-1][j] if i else float('inf')
>             left = grid[i][j-1] if j else float('inf')
>             grid[i][j] += min(up, left)
>     return grid[m-1][n-1]
> ```

**Key takeaway:** Same grid skeleton; `+` becomes `+ min(...)`.

---

## P4: Longest Common Subsequence

**LC #1143** · Medium

### 🧠 Pattern: Two-String DP

> `dp[i][j]` = LCS of `text1[:i]` and `text2[:j]`.
> - If `text1[i-1] == text2[j-1]`: `dp[i][j] = dp[i-1][j-1] + 1`
> - Else: `dp[i][j] = max(dp[i-1][j], dp[i][j-1])`

### Trace

```
text1="abcde"  text2="ace"

       ""  a  c  e
   ""  0   0  0  0
    a  0   1  1  1
    b  0   1  1  1
    c  0   1  2  2
    d  0   1  2  2
    e  0   1  2  3   ← answer
```

> [!success]- Python
> ```python
> def longest_common_subsequence(t1, t2):
>     m, n = len(t1), len(t2)
>     dp = [[0] * (n + 1) for _ in range(m + 1)]
>     for i in range(1, m + 1):
>         for j in range(1, n + 1):
>             if t1[i-1] == t2[j-1]:
>                 dp[i][j] = dp[i-1][j-1] + 1
>             else:
>                 dp[i][j] = max(dp[i-1][j], dp[i][j-1])
>     return dp[m][n]
> ```

**Key takeaway:** Two-string DP grids on string lengths. The classic char-equal-or-not branch.

---

## P5: Edit Distance

**LC #72** · Medium

Min operations (insert / delete / replace) to convert `word1` to `word2`.

### Approach

- `dp[i][0] = i` (delete all of word1)
- `dp[0][j] = j` (insert all of word2)
- If chars match: `dp[i][j] = dp[i-1][j-1]`
- Else: `dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])` (delete, insert, replace)

> [!success]- Python
> ```python
> def min_distance(w1, w2):
>     m, n = len(w1), len(w2)
>     dp = [[0] * (n + 1) for _ in range(m + 1)]
>     for i in range(m + 1): dp[i][0] = i
>     for j in range(n + 1): dp[0][j] = j
>     for i in range(1, m + 1):
>         for j in range(1, n + 1):
>             if w1[i-1] == w2[j-1]:
>                 dp[i][j] = dp[i-1][j-1]
>             else:
>                 dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
>     return dp[m][n]
> ```

**Key takeaway:** 3 ops = 3 transitions. Always think "what was the last op?".

---

## P6: Longest Palindromic Substring

**LC #5** · Medium

### 🧠 Pattern: Expand Around Center (Beats DP!)

> Every palindrome has a center (1 or 2 chars). Try every center → O(n²) time, **O(1) space**. DP also O(n²) but uses O(n²) space.

> [!success]- Python
> ```python
> def longest_palindrome(s):
>     start, end = 0, 0
>     def expand(l, r):
>         while l >= 0 and r < len(s) and s[l] == s[r]:
>             l -= 1; r += 1
>         return l + 1, r - 1
>     for i in range(len(s)):
>         a1, b1 = expand(i, i)
>         a2, b2 = expand(i, i + 1)
>         if b1 - a1 > end - start: start, end = a1, b1
>         if b2 - a2 > end - start: start, end = a2, b2
>     return s[start:end + 1]
> ```

> [!success]- Python (DP form)
> ```python
> def longest_palindrome_dp(s):
>     n = len(s)
>     dp = [[False]*n for _ in range(n)]
>     start, max_len = 0, 1
>     for i in range(n): dp[i][i] = True
>     for length in range(2, n + 1):
>         for i in range(n - length + 1):
>             j = i + length - 1
>             if s[i] == s[j] and (length == 2 or dp[i+1][j-1]):
>                 dp[i][j] = True
>                 if length > max_len:
>                     start, max_len = i, length
>     return s[start:start + max_len]
> ```

**Key takeaway:** Expand-around-center wins on space. DP form is good to know for palindromic substring **count**.

---

## P7: Longest Palindromic Subsequence

**LC #516** · Medium

### Approach

`dp[i][j]` = LPS in `s[i..j]`. If `s[i] == s[j]`: `dp[i][j] = dp[i+1][j-1] + 2`. Else: `max(dp[i+1][j], dp[i][j-1])`.

> Iterate by **substring length** (small → large) so dependencies exist.

> [!success]- Python
> ```python
> def longest_palindrome_subseq(s):
>     n = len(s)
>     dp = [[0]*n for _ in range(n)]
>     for i in range(n): dp[i][i] = 1
>     for length in range(2, n + 1):
>         for i in range(n - length + 1):
>             j = i + length - 1
>             if s[i] == s[j]:
>                 dp[i][j] = (dp[i+1][j-1] if length > 2 else 0) + 2
>             else:
>                 dp[i][j] = max(dp[i+1][j], dp[i][j-1])
>     return dp[0][n-1]
> ```

> [!tip] Equivalent shortcut
> LPS of s = LCS of s and reverse(s).

**Key takeaway:** Substring DP — iterate by length, use `dp[i][j]` for range `[i, j]`.

---

## P8: Distinct Subsequences

**LC #115** · Hard

Number of subsequences of `s` equal to `t`.

### Approach

`dp[i][j]` = #subseqs of `s[:i]` equal to `t[:j]`.
- `dp[i][0] = 1` (empty t)
- If `s[i-1] == t[j-1]`: `dp[i][j] = dp[i-1][j-1] + dp[i-1][j]` (use it or skip it)
- Else: `dp[i][j] = dp[i-1][j]` (must skip)

> [!success]- Python
> ```python
> def num_distinct(s, t):
>     m, n = len(s), len(t)
>     dp = [[0]*(n+1) for _ in range(m+1)]
>     for i in range(m+1): dp[i][0] = 1
>     for i in range(1, m+1):
>         for j in range(1, n+1):
>             if s[i-1] == t[j-1]:
>                 dp[i][j] = dp[i-1][j-1] + dp[i-1][j]
>             else:
>                 dp[i][j] = dp[i-1][j]
>     return dp[m][n]
> ```

**Key takeaway:** "Count subsequences matching pattern" → counting variant of LCS.

---

## P9: Interleaving String

**LC #97** · Medium

Is `s3` an interleaving of `s1` and `s2` (maintaining order)?

### Approach

`dp[i][j]` = True if `s3[:i+j]` is interleaving of `s1[:i]` and `s2[:j]`. Transition: come from s1 or from s2.

> [!success]- Python
> ```python
> def is_interleave(s1, s2, s3):
>     m, n = len(s1), len(s2)
>     if m + n != len(s3): return False
>     dp = [[False]*(n+1) for _ in range(m+1)]
>     dp[0][0] = True
>     for i in range(m+1):
>         for j in range(n+1):
>             if i > 0 and s1[i-1] == s3[i+j-1]:
>                 dp[i][j] = dp[i][j] or dp[i-1][j]
>             if j > 0 and s2[j-1] == s3[i+j-1]:
>                 dp[i][j] = dp[i][j] or dp[i][j-1]
>     return dp[m][n]
> ```

**Key takeaway:** "Boolean two-sequence merge" → check both predecessors.

---

## P10: Maximal Square

**LC #221** · Medium

In binary matrix, side of largest all-1 square.

### 🧠 Pattern: Min of Three Neighbors

> `dp[i][j]` = side of largest square ending at `(i, j)`. If `matrix[i][j] == '1'`: `dp[i][j] = 1 + min(dp[i-1][j-1], dp[i-1][j], dp[i][j-1])`.

> [!success]- Python
> ```python
> def maximal_square(matrix):
>     if not matrix: return 0
>     m, n = len(matrix), len(matrix[0])
>     dp = [[0]*(n+1) for _ in range(m+1)]
>     best = 0
>     for i in range(1, m+1):
>         for j in range(1, n+1):
>             if matrix[i-1][j-1] == '1':
>                 dp[i][j] = 1 + min(dp[i-1][j-1], dp[i-1][j], dp[i][j-1])
>                 best = max(best, dp[i][j])
>     return best * best
> ```

**Key takeaway:** Squares "anchored at bottom-right" → min of 3 corners. Beautiful structural insight.

---

## P11: Best Time to Buy and Sell Stock IV

**LC #188** · Hard

Up to k transactions. Max profit.

### 🧠 Pattern: State × Time DP

> `dp[t][i]` = max profit with at most `t` transactions, using prices through index `i`.
> `dp[t][i] = max(dp[t][i-1], max over j<i of prices[i] - prices[j] + dp[t-1][j-1])`.
> Optimize inner max by tracking `max_diff = dp[t-1][j-1] - prices[j]`.

> [!success]- Python
> ```python
> def max_profit_iv(k, prices):
>     n = len(prices)
>     if not n or k == 0: return 0
>     if k >= n // 2:
>         return sum(max(0, prices[i] - prices[i-1]) for i in range(1, n))
>     dp = [[0]*n for _ in range(k+1)]
>     for t in range(1, k+1):
>         max_diff = -prices[0]
>         for i in range(1, n):
>             dp[t][i] = max(dp[t][i-1], prices[i] + max_diff)
>             max_diff = max(max_diff, dp[t-1][i] - prices[i])
>     return dp[k][n-1]
> ```

**Key takeaway:** When state has two free dims (txn count × time), use 2D DP. The inner max optimization is the key trick.

---

## P12: Burst Balloons

**LC #312** · Hard · **Interval DP**

Burst balloons one at a time; gain = `nums[i-1] * nums[i] * nums[i+1]`. Max gain.

### 🧠 Pattern: Interval DP — Pick the LAST Balloon

> `dp[l][r]` = max coins from bursting all balloons in (l, r) **exclusive**. The trick: enumerate which balloon `k` is **burst last** in this range. When `k` bursts, its neighbors are `nums[l]` and `nums[r]` (since everything in between is gone). Recurse on (l, k) and (k, r).

> [!success]- Python
> ```python
> def max_coins(nums):
>     a = [1] + nums + [1]
>     n = len(a)
>     dp = [[0]*n for _ in range(n)]
>     for length in range(2, n):
>         for l in range(n - length):
>             r = l + length
>             for k in range(l + 1, r):
>                 dp[l][r] = max(dp[l][r], a[l]*a[k]*a[r] + dp[l][k] + dp[k][r])
>     return dp[0][n-1]
> ```

**Variants:** Strategy DP · Minimum Cost to Merge Stones.

**Key takeaway:** Interval DP often becomes tractable when you fix "the last move" (here: which balloon bursts last). Build up by interval length.

---

> [!tip] After this drill
> Recognize 2D DP triggers: two strings, two indices, grid traversal, substring `(i,j)`. Choose by what changes between subproblems.
