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
| P8 | Distinct Subsequences | 115 | **Hard** | Two-string count |
| P9 | Interleaving String | 97 | Med | Two-string boolean |
| P10 | Maximal Square | 221 | Med | Grid min of neighbors |
| P11 | Best Time to Buy and Sell Stock IV | 188 | **Hard** | State × time |
| P12 | Burst Balloons | 312 | **Hard** | Interval DP |

---

## P1: Unique Paths

**LC #62** · Medium

> [!info]- 🔍 Dry Run: m=3, n=3
> ```text
> dp[i][j] = paths to reach (i,j); init all 1.
> 
> Initial (i=0 row, j=0 col all 1):
>   1 1 1
>   1 . .
>   1 . .
> 
> i=1, j=1: dp[1][1] = dp[0][1] + dp[1][0] = 1 + 1 = 2
> i=1, j=2: dp[1][2] = dp[0][2] + dp[1][1] = 1 + 2 = 3
> i=2, j=1: dp[2][1] = dp[1][1] + dp[2][0] = 2 + 1 = 3
> i=2, j=2: dp[2][2] = dp[1][2] + dp[2][1] = 3 + 3 = 6
> 
> Final:
>   1 1 1
>   1 2 3
>   1 3 6
> 
> ✅ Answer: 6
> ```

> [!success]- Python
> ```python
> def unique_paths(m, n):
>     dp = [[1] * n for _ in range(m)]
>     for i in range(1, m):
>         for j in range(1, n):
>             dp[i][j] = dp[i-1][j] + dp[i][j-1]
>     return dp[m-1][n-1]
> ```

**Key takeaway:** Grid path = sum of "from up" + "from left". The canonical 2D DP.

---

## P2: Unique Paths II

**LC #63** · Medium · Obstacles

> [!info]- 🔍 Dry Run: grid=[[0,0,0],[0,1,0],[0,0,0]]
> ```text
> Obstacles marked '1'; we cannot pass them.
> 
> Initialize dp[0][0] = 1 (start is open)
> 
> Row 0: dp[0][1] = dp[0][0] = 1; dp[0][2] = 1 (no obstacles in row 0)
> Col 0: dp[1][0] = dp[0][0] = 1; dp[2][0] = 1
> 
> Now fill rest:
>   (1,1): obstacle → dp[1][1] = 0
>   (1,2): dp[1][2] = dp[0][2] + dp[1][1] = 1 + 0 = 1
>   (2,1): dp[2][1] = dp[1][1] + dp[2][0] = 0 + 1 = 1
>   (2,2): dp[2][2] = dp[1][2] + dp[2][1] = 1 + 1 = 2
> 
> Final:
>   1 1 1
>   1 0 1
>   1 1 2
> 
> ✅ Answer: 2
> ```

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

> [!info]- 🔍 Dry Run: grid=[[1,3,1],[1,5,1],[4,2,1]]
> ```text
> In-place update (overwrite grid as DP table):
> 
> Original:
>   1 3 1
>   1 5 1
>   4 2 1
> 
> Row 0: no left → dp[0][j] = grid[0][0..j] cumulative
>   dp[0] = [1, 4, 5]
> 
> Col 0: no up → dp[i][0] = cumulative
>   dp = [[1,4,5], [2,5,1], [6,2,1]]   (just dp[1][0]=1+1=2, dp[2][0]=2+4=6)
> 
> i=1, j=1: dp[1][1] = grid[1][1] + min(up=dp[0][1]=4, left=dp[1][0]=2) = 5 + 2 = 7
> i=1, j=2: dp[1][2] = 1 + min(dp[0][2]=5, dp[1][1]=7) = 1 + 5 = 6
> i=2, j=1: dp[2][1] = 2 + min(dp[1][1]=7, dp[2][0]=6) = 2 + 6 = 8
> i=2, j=2: dp[2][2] = 1 + min(dp[1][2]=6, dp[2][1]=8) = 1 + 6 = 7
> 
> ✅ Answer: 7
>   (Path: 1→3→1→1→1 = 7, or 1→1→5→1→1... actually best path: 1→3→1→1→1)
>   Verify: top-right path: 1+3+1+1+1=7 ✓
> ```

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

> [!info]- 🔍 Dry Run: text1="abcde", text2="ace"
> ```text
> dp[i][j] = LCS of text1[:i] and text2[:j]
> 
>             ""  a  c  e
>     ""      0   0  0  0
>      a      0   1  1  1
>      b      0   1  1  1
>      c      0   1  2  2
>      d      0   1  2  2
>      e      0   1  2  3
> 
> Fill explanation:
>   (a,a) match → dp[1][1] = dp[0][0]+1 = 1
>   (a,c) no match → max(dp[0][2], dp[1][1]) = max(0,1) = 1
>   ... etc
>   (c,c) match → dp[3][2] = dp[2][1]+1 = 1+1 = 2
>   (e,e) match → dp[5][3] = dp[4][2]+1 = 2+1 = 3
> 
> Return dp[5][3] = 3
> 
> ✅ Answer: 3   (LCS = "ace")
> ```

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

> [!info]- 🔍 Dry Run: word1="horse", word2="ros"
> ```text
> dp[i][j] = min ops to convert word1[:i] to word2[:j]
> 
>             ""  r  o  s
>     ""      0   1  2  3      (insert j chars)
>      h      1   1  2  3
>      o      2   2  1  2
>      r      3   2  2  2
>      s      4   3  3  2
>      e      5   4  4  3
> 
> Fill explanation:
>   dp[i][0] = i  (delete all of word1)
>   dp[0][j] = j  (insert)
>   (h,r) no match → 1 + min(dp[0][1]=1, dp[1][0]=1, dp[0][0]=0) = 1 + 0 = 1
>   (o,o) match → dp[2][2] = dp[1][1] = 1
>   ...
> 
> Return dp[5][3] = 3
> 
> ✅ Answer: 3
>   Operations: horse → rorse (replace h→r) → rose (delete r) → ros (delete e). 3 ops. ✓
> ```

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

> [!info]- 🔍 Dry Run: s="babad"
> ```text
> For each center, expand outward while chars match:
> 
> ─────────────────────────────────────────
> i=0 'b':
>   expand(0, 0): odd-len center
>     l=0 r=0: 'b'='b' → expand. l=-1, r=1 → exit (l<0)
>     return a=0, b=0 (range [0,0])
>     Best so far: "b" len 1
>   expand(0, 1): even-len center
>     l=0 r=1: 'b' vs 'a' → no match, exit
>     return a=1, b=0 (empty range)
> 
> i=1 'a':
>   expand(1, 1): odd
>     l=1 r=1: match. l=0 r=2: 'b'='b' match. l=-1 r=3: exit.
>     return a=0, b=2 (range [0,2] = "bab" len 3)
>     Update best: "bab" len 3
>   expand(1, 2): even
>     l=1 r=2: 'a' vs 'b' no match
> 
> i=2 'b':
>   expand(2, 2): odd
>     match. expand: 'a'='a' (l=1 r=3) match. expand: 'b' vs 'd' (l=0 r=4) no.
>     range [1, 3] = "aba" len 3
>   expand(2, 3): even
>     'b' vs 'a' no
> 
> i=3 'a': expand → "a" or maybe " " — no longer than 3
> i=4 'd': "d"
> 
> ✅ Answer: "bab" (or "aba" — both are valid len-3 palindromes)
> ```

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

**Key takeaway:** Expand-around-center wins on space.

---

## P7: Longest Palindromic Subsequence

**LC #516** · Medium

> [!info]- 🔍 Dry Run: s="bbbab"
> ```text
> dp[i][j] = LPS in s[i..j] (inclusive)
> 
> Base: dp[i][i] = 1 (single char is palindrome)
> 
> Iterate by substring length:
> 
> Length 2:
>   dp[0][1]: s[0]='b',s[1]='b' match → 0+2 = 2
>   dp[1][2]: 'b','b' → 2
>   dp[2][3]: 'b','a' no → max(dp[3][3], dp[2][2]) = 1
>   dp[3][4]: 'a','b' no → max(1,1) = 1
> 
> Length 3:
>   dp[0][2]: 'b','b' match → dp[1][1]+2 = 3
>   dp[1][3]: 'b','a' no → max(dp[2][3]=1, dp[1][2]=2) = 2
>   dp[2][4]: 'b','b' match → dp[3][3]+2 = 3
> 
> Length 4:
>   dp[0][3]: 'b','a' no → max(dp[1][3]=2, dp[0][2]=3) = 3
>   dp[1][4]: 'b','b' match → dp[2][3]+2 = 1+2 = 3
> 
> Length 5:
>   dp[0][4]: 'b','b' match → dp[1][3]+2 = 2+2 = 4
> 
> Return dp[0][n-1] = 4
> 
> ✅ Answer: 4   (subsequence "bbbb")
> ```

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

**Key takeaway:** Substring DP — iterate by length, use `dp[i][j]` for range `[i, j]`.

---

## P8: Distinct Subsequences

**LC #115** · **Hard**

> [!info]- 🔍 Dry Run: s="rabbbit", t="rabbit"
> ```text
> dp[i][j] = #subseqs of s[:i] equal to t[:j]
> 
>             ""  r  a  b  b  i  t
>     ""      1   0  0  0  0  0  0
>      r      1   1  0  0  0  0  0
>      a      1   1  1  0  0  0  0
>      b      1   1  1  1  0  0  0
>      b      1   1  1  2  1  0  0   ← (b,b) match: dp[3][2]+dp[3][3] = 1+1=2; (b,b) at j=4: dp[3][3]+dp[3][4]=1+0=1
>      b      1   1  1  3  3  0  0   ← (b,b) at j=3: 1+2=3; at j=4: 2+1=3
>      i      1   1  1  3  3  3  0   ← (i,i): dp[5][4]+dp[5][5]=3+0=3
>      t      1   1  1  3  3  3  3   ← (t,t): dp[6][5]+dp[6][6]=3+0=3
> 
> Return dp[7][6] = 3
> 
> ✅ Answer: 3
>   Verify: "rabbit" can be formed from "rabbbit" by choosing 3 different bs.
>   Positions of bs in "rabbbit": 2, 3, 4. We need 2 of them (rabbit has 2 bs in a row).
>   Pairs: (2,3), (2,4), (3,4) → 3 ways. ✓
> ```

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

> [!info]- 🔍 Dry Run: s1="aabcc", s2="dbbca", s3="aadbbcbcac"
> ```text
> Check len: 5 + 5 == 10 ✓
> 
> dp[i][j] = is s3[:i+j] an interleaving of s1[:i] and s2[:j]?
> 
> Selected cells (table 6×6):
>   dp[0][0] = T
>   dp[1][0]: s1[0]='a' == s3[0]='a' AND dp[0][0]=T → T
>   dp[2][0]: s1[1]='a' == s3[1]='a' AND dp[1][0]=T → T
>   dp[3][0]: s1[2]='b' vs s3[2]='d' → F
>   dp[0][1]: s2[0]='d' vs s3[0]='a' → F
>   dp[0][2]: F (since dp[0][1]=F)
>   dp[2][1]: try s2[0]='d' vs s3[2]='d' AND dp[2][0]=T → T
>   dp[3][1]: s1[2]='b' vs s3[3]='b' AND dp[2][1]=T → T (and other paths possible)
>   ...
> 
> Eventually dp[5][5] = T
> 
> ✅ Answer: true
> ```

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

> [!info]- 🔍 Dry Run: matrix=[["1","0","1","0","0"],["1","0","1","1","1"],["1","1","1","1","1"],["1","0","0","1","0"]]
> ```text
> dp[i][j] = side length of largest all-1 square ending at (i-1, j-1) in matrix.
> Padded with zero row/col on top/left for convenience.
> 
> Row 1 (matrix row 0): "10100"
>   dp[1][1]: matrix '1' → 1 + min(0,0,0) = 1; best=1
>   dp[1][2]: '0' → 0
>   dp[1][3]: '1' → 1; best=1
>   dp[1][4], dp[1][5]: 0
> 
> Row 2 (matrix row 1): "10111"
>   dp[2][1]: '1' → 1 + min(dp[1][0]=0, dp[1][1]=1, dp[2][0]=0) = 1
>   dp[2][2]: '0' → 0
>   dp[2][3]: '1' → 1 + min(dp[1][2]=0, dp[1][3]=1, dp[2][2]=0) = 1
>   dp[2][4]: '1' → 1 + min(dp[1][3]=1, dp[1][4]=0, dp[2][3]=1) = 1
>   dp[2][5]: '1' → 1 + min(dp[1][4]=0, dp[1][5]=0, dp[2][4]=1) = 1
> 
> Row 3 (matrix row 2): "11111"
>   dp[3][1]: '1' → 1 + min(0, 1, 0) = 1
>   dp[3][2]: '1' → 1 + min(dp[2][1]=1, dp[2][2]=0, dp[3][1]=1) = 1
>   dp[3][3]: '1' → 1 + min(dp[2][2]=0, dp[2][3]=1, dp[3][2]=1) = 1
>   dp[3][4]: '1' → 1 + min(dp[2][3]=1, dp[2][4]=1, dp[3][3]=1) = 2; best=2
>   dp[3][5]: '1' → 1 + min(dp[2][4]=1, dp[2][5]=1, dp[3][4]=2) = 2
> 
> Row 4 (matrix row 3): "10010"
>   dp[4][1]: '1' → 1 + min(0, 1, 0) = 1
>   dp[4][2]: '0' → 0
>   dp[4][3]: '0' → 0
>   dp[4][4]: '1' → 1 + min(dp[3][3]=1, dp[3][4]=2, dp[4][3]=0) = 1
>   dp[4][5]: '0' → 0
> 
> Best side length = 2; answer = 2² = 4
> 
> ✅ Answer: 4
> ```

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

**Key takeaway:** Squares "anchored at bottom-right" → min of 3 corners.

---

## P11: Best Time to Buy and Sell Stock IV

**LC #188** · **Hard**

> [!info]- 🔍 Dry Run: k=2, prices=[2,4,1]
> ```text
> dp[t][i] = max profit with ≤ t transactions, using prices[:i+1]
> n=3
> 
> For t=1:
>   max_diff starts as -prices[0] = -2
>   dp[1][0] = 0
>   i=1: dp[1][1] = max(dp[1][0]=0, prices[1]+max_diff=4-2=2) = 2
>        max_diff = max(-2, dp[0][1]-prices[1]=0-4=-4) = -2
>   i=2: dp[1][2] = max(dp[1][1]=2, prices[2]+max_diff=1-2=-1) = 2
>        max_diff = max(-2, dp[0][2]-prices[2]=0-1=-1) = -1
> 
> For t=2:
>   max_diff starts as -prices[0] = -2
>   dp[2][0] = 0
>   i=1: dp[2][1] = max(0, 4-2=2) = 2
>        max_diff = max(-2, dp[1][1]-prices[1]=2-4=-2) = -2
>   i=2: dp[2][2] = max(2, 1-2=-1) = 2
>        max_diff = max(-2, dp[1][2]-prices[2]=2-1=1) = 1
> 
> Return dp[2][2] = 2
> 
> ✅ Answer: 2   (Buy at 2, sell at 4; profit 2. Second transaction not useful.)
> ```

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

**LC #312** · **Hard** · **Interval DP**

> [!info]- 🔍 Dry Run: nums=[3,1,5,8]
> ```text
> Padded: a = [1, 3, 1, 5, 8, 1]  (with sentinel 1s)
> 
> dp[l][r] = max coins from bursting all balloons in (l, r) exclusive.
> 
> length=2 (balloons strictly between l and r — just one balloon: at index l+1):
>   dp[0][2]: k=1: a[0]*a[1]*a[2] = 1*3*1 = 3 → dp[0][2]=3
>   dp[1][3]: k=2: a[1]*a[2]*a[3] = 3*1*5 = 15 → dp[1][3]=15
>   dp[2][4]: k=3: a[2]*a[3]*a[4] = 1*5*8 = 40 → dp[2][4]=40
>   dp[3][5]: k=4: a[3]*a[4]*a[5] = 5*8*1 = 40 → dp[3][5]=40
> 
> length=3 (two balloons):
>   dp[0][3]: try last=k=1: a[0]*a[1]*a[3] + dp[0][1] + dp[1][3]
>                          = 1*3*5 + 0 + 15 = 30
>             try last=k=2: a[0]*a[2]*a[3] + dp[0][2] + dp[2][3]
>                          = 1*1*5 + 3 + 0 = 8
>             max = 30
>   dp[1][4]: try k=2: a[1]*a[2]*a[4] + dp[1][2] + dp[2][4] = 3*1*8 + 0 + 40 = 64
>             try k=3: a[1]*a[3]*a[4] + dp[1][3] + dp[3][4] = 3*5*8 + 15 + 0 = 135
>             max = 135
>   dp[2][5]: try k=3: 1*5*1 + 0 + dp[3][5]=40 → 45
>             try k=4: 1*8*1 + dp[2][4]=40 + 0 → 48
>             max = 48
> 
> length=4:
>   dp[0][4]: try k=1: a[0]*a[1]*a[4] + dp[0][1]=0 + dp[1][4]=135 = 1*3*8+135 = 159
>             try k=2: a[0]*a[2]*a[4] + dp[0][2]=3 + dp[2][4]=40 = 1*1*8+43 = 51
>             try k=3: a[0]*a[3]*a[4] + dp[0][3]=30 + dp[3][4]=0 = 1*5*8+30 = 70
>             max = 159
>   dp[1][5]: try k=2: 3*1*1 + 0 + dp[2][5]=48 = 51
>             try k=3: 3*5*1 + dp[1][3]=15 + dp[3][5]=40 = 70
>             try k=4: 3*8*1 + dp[1][4]=135 + 0 = 159
>             max = 159
> 
> length=5:
>   dp[0][5]: try k=1: 1*3*1 + 0 + dp[1][5]=159 = 162
>             try k=2: 1*1*1 + dp[0][2]=3 + dp[2][5]=48 = 52
>             try k=3: 1*5*1 + dp[0][3]=30 + dp[3][5]=40 = 75
>             try k=4: 1*8*1 + dp[0][4]=159 + 0 = 167
>             max = 167
> 
> ✅ Answer: 167
> ```

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
