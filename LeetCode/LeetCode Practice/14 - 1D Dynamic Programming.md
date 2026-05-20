---
title: "LeetCode Practice: 1D Dynamic Programming"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - dp
  - dynamic-programming
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: 1D Dynamic Programming

17 problems · the most powerful pattern for "best/count/yes-no over sequential choices".

> [!abstract] Pattern recap
> DP = **memoizing decisions over subproblems**. To design:
> 
> 1. **State**: what does `dp[i]` mean?
> 2. **Transition**: `dp[i] = f(dp[i-1], dp[i-2], ...)`
> 3. **Base case**: trivial values
> 4. **Order**: iterate so dependencies are computed first
> 5. **Answer**: extract from `dp[n]` or `max(dp)`

> [!tip] Two equivalent forms
> **Top-down (memoization):** recursive with cache. Easier for tricky bases.
> **Bottom-up (tabulation):** iterative. Enables space optimization.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Climbing Stairs | 70 | Easy | Fibonacci |
| P2 | Min Cost Climbing Stairs | 746 | Easy | Take min of last two |
| P3 | House Robber | 198 | Med | Take or skip |
| P4 | House Robber II | 213 | Med | Circular → two passes |
| P5 | Decode Ways | 91 | Med | One- or two-digit |
| P6 | Coin Change | 322 | Med | Unbounded knapsack |
| P7 | Maximum Product Subarray | 152 | Med | Track max AND min |
| P8 | Word Break | 139 | Med | Substring DP |
| P9 | Longest Increasing Subsequence | 300 | Med | O(n²) or O(n log n) |
| P10 | Partition Equal Subset Sum | 416 | Med | 0/1 knapsack on sums |
| P11 | Stock with Cooldown | 309 | Med | State machine |
| P12 | Stock with Transaction Fee | 714 | Med | State machine |
| P13 | Perfect Squares | 279 | Med | Unbounded knapsack |
| P14 | Combination Sum IV | 377 | Med | Order matters → permutations |
| P15 | Delete and Earn | 740 | Med | Reduce to house robber |
| P16 | Maximum Sum Circular Subarray | 918 | Med | Kadane × 2 |
| P17 | Regular Expression Matching | 10 | **Hard** | Char/pattern DP |

---

## P1: Climbing Stairs

**LC #70** · Easy

### 🧠 Pattern: Fibonacci

> `dp[i] = dp[i-1] + dp[i-2]`. Last step was either 1 or 2.

> [!example]- 📊 Visual: Fibonacci recurrence
> ```text
>   dp[i] = ways to reach step i = dp[i-1] + dp[i-2]
> 
>     i:    0    1    2    3    4    5
>   dp[i]:  1    1    2    3    5    8       ← Fibonacci!
>           ▲    ▲    ▲    ▲    ▲    ▲
>           │    │    │    │    │    │
>          base base  └─┬──┘    │    │       dp[2] = dp[0]+dp[1] = 1+1 = 2
>                       │       │    │       dp[3] = dp[1]+dp[2] = 1+2 = 3
>                       └─┬─────┘    │
>                         │          │       dp[5] = dp[3]+dp[4] = 3+5 = 8
>                         └────┬─────┘
> 
>   Two ways to reach step i: come from i-1 (took 1 step) or i-2 (took 2 steps).
> ```

> [!info]- 🔍 Dry Run: n=5
> ```text
> Base: dp[1]=1 (one way), dp[2]=2 (1+1 or 2)
> 
> a, b = 1, 2
> 
> i=3: new = a+b = 1+2 = 3; a, b = 2, 3
> i=4: new = 2+3 = 5; a, b = 3, 5
> i=5: new = 3+5 = 8; a, b = 5, 8
> 
> Return b = 8
> 
> ✅ Answer: 8 ways to climb 5 stairs
>   (1+1+1+1+1, 1+1+1+2, 1+1+2+1, 1+2+1+1, 2+1+1+1, 1+2+2, 2+1+2, 2+2+1)
> ```

> [!success]- Python
> ```python
> def climb_stairs(n):
>     if n <= 2: return n
>     a, b = 1, 2
>     for _ in range(3, n + 1):
>         a, b = b, a + b
>     return b
> ```

**Key takeaway:** The simplest DP. Get this template down cold.

---

## P2: Min Cost Climbing Stairs

**LC #746** · Easy

> [!example]- 📊 Visual: min over last two steps
> ```text
>   cost = [10, 15, 20]   (cost to LEAVE that step)
> 
>   dp[i] = min cost to REACH step i
> 
>     i:       0    1     2          top(3)
>   cost[i]:  10   15    20            —
>   dp[i]:    0    0    10            15
>             ▲    ▲     ▲             ▲
>             │    │     │             │
>            base base   │             │
>                        │             │
>                        min(dp[0]+cost[0],   min(dp[1]+cost[1],
>                            dp[1]+cost[1])       dp[2]+cost[2])
>                      = min(0+10, 0+15)    = min(0+15, 10+20)
>                      = 10                 = 15
> 
>   Start free at step 0 or 1; pay cost[i] to step off. Aim past last index.
> ```

> [!info]- 🔍 Dry Run: cost=[10,15,20]
> ```text
> a, b = 0, 0    (cost to reach step 0 and step 1, both 0 since we can start there for free)
> 
> i=2: min cost to reach step 2 = min(reach step 0 + cost[0], reach step 1 + cost[1])
>      = min(0+10, 0+15) = 10
>      a, b = 0, 10
> 
> i=3 (top, beyond the array): min cost to reach top
>      = min(reach step 1 + cost[1], reach step 2 + cost[2])
>      = min(0+15, 10+20) = 15
> 
> ✅ Answer: 15   (path: step 1 → top, pay cost[1]=15)
> ```

> [!success]- Python
> ```python
> def min_cost_climbing_stairs(cost):
>     a, b = 0, 0   # cost to reach step 0, step 1
>     for i in range(2, len(cost) + 1):
>         a, b = b, min(a + cost[i-2], b + cost[i-1])
>     return b
> ```

**Key takeaway:** Same shape as P1, replace `+` with `min(+cost)`.

---

## P3: House Robber

**LC #198** · Medium

> [!example]- 📊 Visual: take-or-skip
> ```text
>   nums:  [2, 7, 9, 3, 1]
>   idx:    0  1  2  3  4
> 
>   At each house, choose:
>     SKIP →  dp[i] = dp[i-1]
>     ROB  →  dp[i] = dp[i-2] + nums[i]
>     dp[i] = max(skip, rob)
> 
>   dp:    [2, 7, 11, 11, 12]
>           ▲   ▲   ▲    ▲    ▲
>           │   │   │    │    │
>           │   │   │    │    rob 1+dp[2]=12 > skip dp[3]=11 → rob
>           │   │   │    │
>           │   │   │    skip dp[2]=11 vs rob 3+dp[1]=10 → skip
>           │   │   │
>           │   │   rob 9+dp[0]=11 > skip dp[1]=7 → rob
>           │   │
>           │   skip dp[0]=2 vs rob 7+0=7 → rob
>           │
>          base
> 
>   Adjacent constraint: skipping breaks the chain — that's why dp[i-2] is used when robbing.
> ```

> [!info]- 🔍 Dry Run: nums=[2,7,9,3,1]
> ```text
> prev2, prev1 = 0, 0
> 
> x=2:  new = max(prev1, prev2 + x) = max(0, 0+2) = 2
>       prev2, prev1 = 0, 2
> 
> x=7:  new = max(2, 0+7) = 7
>       prev2, prev1 = 2, 7
> 
> x=9:  new = max(7, 2+9) = 11      ← rob houses 1 (val 2) and 3 (val 9)
>       prev2, prev1 = 7, 11
> 
> x=3:  new = max(11, 7+3) = 11
>       prev2, prev1 = 11, 11
> 
> x=1:  new = max(11, 11+1) = 12    ← rob 1, 3, 5
>       prev2, prev1 = 11, 12
> 
> ✅ Answer: 12
> ```

> [!success]- Python
> ```python
> def rob(nums):
>     prev2, prev1 = 0, 0
>     for x in nums:
>         prev2, prev1 = prev1, max(prev1, prev2 + x)
>     return prev1
> ```

**Key takeaway:** "Take or skip" decision → max over both choices.

---

## P4: House Robber II

**LC #213** · Medium · Circular

> [!example]- 📊 Visual: break the circle two ways
> ```text
>   Houses in a circle: house 0 and house n-1 are NEIGHBORS.
> 
>           ┌──── 0 ────┐
>           │           │
>           4           1
>           │           │
>           └─ 3 ─── 2 ─┘
> 
>   Run linear-rob TWICE on two slices, then take max:
> 
>     Scenario A: SKIP house 0 (free to take last)
>       slice = nums[1 : n]  ─────►  [─, ●, ●, ●, ●]
>                                       1  2  3  4
> 
>     Scenario B: SKIP house n-1 (free to take first)
>       slice = nums[0 : n-1] ─────►  [●, ●, ●, ●, ─]
>                                       0  1  2  3
> 
>     answer = max( rob_linear(A), rob_linear(B) )
> 
>   Either way, the two "wrap-around neighbors" are never both robbed.
> ```

> [!info]- 🔍 Dry Run: nums=[2,3,2]
> ```text
> Two scenarios (since 1st and last are neighbors):
>   A: rob houses [1, n-1] (skip house 0):
>      sub-array [3, 2]
>      rob_linear: prev2=0,prev1=0; x=3 → 0,3; x=2 → 3,3
>      result = 3
>   B: rob houses [0, n-2] (skip last):
>      sub-array [2, 3]
>      rob_linear: x=2 → 0,2; x=3 → 2,3
>      result = 3
> 
> max(A, B) = 3
> 
> ✅ Answer: 3
> ```

> [!success]- Python
> ```python
> def rob_circular(nums):
>     if len(nums) == 1: return nums[0]
>     def rob_linear(arr):
>         prev2, prev1 = 0, 0
>         for x in arr:
>             prev2, prev1 = prev1, max(prev1, prev2 + x)
>         return prev1
>     return max(rob_linear(nums[1:]), rob_linear(nums[:-1]))
> ```

**Key takeaway:** Circular constraint → run linear DP twice on the two slices.

---

## P5: Decode Ways

**LC #91** · Medium

> [!example]- 📊 Visual: one- or two-digit grouping
> ```text
>   s = "2 2 6"          A=1 … Z=26
>        0 1 2
> 
>   At each position i, two possible last decodes:
> 
>     ┌─── single ───┐     ┌──── pair ────┐
>     │   s[i]       │     │  s[i-1:i+1]  │
>     │  valid if    │     │  valid if    │
>     │  '1'..'9'    │     │  10 ≤ n ≤ 26 │
>     └──────┬───────┘     └──────┬───────┘
>            │                    │
>            ▼                    ▼
>          dp[i-1]              dp[i-2]
> 
>   dp[i] = (single? dp[i-1] : 0) + (pair? dp[i-2] : 0)
> 
>     i:     0    1    2    3
>            ""   "2"  "22" "226"
>   dp[i]:    1    1    2     3
>             ▲    ▲    ▲     ▲
>            base base  │     │
>                       │     │
>            single '2' + pair "22" = 1+1=2
>                                  │
>            single '6' + pair "26" = 2+1=3    ← "2,2,6"·"22,6"·"2,26"
> ```

> [!info]- 🔍 Dry Run: s="226"
> ```text
> prev2, prev1 = 1, 1   (dp[0]=1 empty string, dp[1]=1 for "2")
> 
> i=1, s[1]='2':
>   cur = 0
>   single '2' valid (1-9) → cur += prev1 (=1) → cur=1
>   two-digit s[0:2]="22" → 22 in [10,26] → cur += prev2 (=1) → cur=2
>   prev2, prev1 = 1, 2
> 
> i=2, s[2]='6':
>   cur = 0
>   single '6' valid → cur += prev1 = 2 → cur=2
>   two-digit "26" → 26 in [10,26] → cur += prev2 = 1 → cur=3
>   prev2, prev1 = 2, 3
> 
> ✅ Answer: 3
>   Decodings: "2,2,6"=BBF, "22,6"=VF, "2,26"=BZ
> ```

> [!success]- Python
> ```python
> def num_decodings(s):
>     if not s or s[0] == '0': return 0
>     prev2, prev1 = 1, 1
>     for i in range(1, len(s)):
>         cur = 0
>         if s[i] != '0': cur += prev1
>         two = int(s[i-1:i+1])
>         if 10 <= two <= 26: cur += prev2
>         prev2, prev1 = prev1, cur
>     return prev1
> ```

**Key takeaway:** Two transitions per step (one or two digits). Validate each before summing.

---

## P6: Coin Change

**LC #322** · Medium · Unbounded knapsack

> [!example]- 📊 Visual: unbounded knapsack on amount axis
> ```text
>   coins = [1, 2, 5]    amount = 11
> 
>   dp[a] = min #coins to make amount a    (∞ if impossible)
> 
>     a:     0  1  2  3  4  5  6  7  8  9  10  11
>   dp[a]:   0  1  1  2  2  1  2  2  3  3   2   3
>            ▲                 ▲           ▲    ▲
>            │                 │           │    │
>           base               │           │    │
>                              │           │    │
>             dp[5] = 1 + min(dp[4], dp[3], dp[0])
>                       coin   c=1   c=2    c=5
>                              =3    =3     =1   → 1
>                              
>             dp[10] = 1 + min(dp[9], dp[8], dp[5]) = 1 + min(3,3,1) = 2
>             dp[11] = 1 + min(dp[10], dp[9], dp[6]) = 1 + min(2,3,2) = 3
> 
>   Transition: dp[a] = min over coins c of  dp[a-c] + 1
>   "Unbounded" → coins reusable → forward sweep on a.
> ```

> [!info]- 🔍 Dry Run: coins=[1,2,5], amount=11
> ```text
> dp[0]=0; dp[1..11]=INF
> 
> a=1: try c=1: dp[1]=min(INF, dp[0]+1)=1; c=2,5 too big.
> a=2: c=1: dp[2]=min(INF, dp[1]+1)=2; c=2: dp[2]=min(2, dp[0]+1)=1; c=5 skip
> a=3: c=1: dp[3]=dp[2]+1=2; c=2: dp[3]=min(2, dp[1]+1)=2; c=5 skip
> a=4: c=1: dp[4]=dp[3]+1=3; c=2: dp[4]=min(3, dp[2]+1)=2
> a=5: c=1: dp[5]=dp[4]+1=3; c=2: dp[5]=min(3, dp[3]+1)=3; c=5: dp[5]=min(3, dp[0]+1)=1
> a=6: c=1: =2; c=2: min(2, dp[4]+1=3)=2; c=5: min(2, dp[1]+1=2)=2
> a=7: c=1:3, c=2:2 (dp[5]+1), c=5:2(dp[2]+1)
> a=8: c=1:3, c=2:3 (dp[6]+1), c=5:3 (dp[3]+1)
> a=9: c=1:4, c=2:3, c=5:3 (dp[4]+1)
> a=10: c=1:4, c=2:3 (dp[8]+1), c=5: min(3, dp[5]+1=4)=2? actually dp[5]+1=2 → dp[10]=2
> a=11: c=1: dp[10]+1=3; c=2: dp[9]+1=4; c=5: dp[6]+1=3
>   dp[11] = 3
> 
> ✅ Answer: 3   (e.g., 5+5+1)
> ```

> [!success]- Python
> ```python
> def coin_change(coins, amount):
>     INF = amount + 1
>     dp = [INF] * (amount + 1)
>     dp[0] = 0
>     for a in range(1, amount + 1):
>         for c in coins:
>             if c <= a:
>                 dp[a] = min(dp[a], dp[a - c] + 1)
>     return dp[amount] if dp[amount] <= amount else -1
> ```

**Key takeaway:** "Min items / count ways" with reusable items → unbounded knapsack.

---

## P7: Maximum Product Subarray

**LC #152** · Medium

> [!example]- 📊 Visual: track both max AND min (sign flips!)
> ```text
>   nums = [2, 3, -2, 4]
> 
>   Track cur_max, cur_min ending at index i.
>   When x < 0, multiplying flips sign:  big positive ↔ big negative.
>   So SWAP cur_max ↔ cur_min before updating with a negative x.
> 
>     i:           0     1      2      3
>     x:           2     3     -2      4
>   cur_max:       2     6     -2      4         ← keep best running product
>   cur_min:       2     3    -12    -48         ← min becomes "potential" max
>     best:        2     6      6      6
>                                      ▲
>                                      │
>                       x=4, +ve: no swap
>                       cur_max = max(4, -2·4=-8) = 4
>                       cur_min = min(4, -12·4=-48) = -48
> 
>   Why min matters:
>     A LARGE-negative cur_min can become a LARGE-positive cur_max as soon as
>     another negative number arrives.   [-2, -3] → 6, even though either alone is negative.
> ```

> [!info]- 🔍 Dry Run: nums=[2,3,-2,4]
> ```text
> cur_max = cur_min = best = 2
> 
> x=3:
>   cur_max = max(3, 2*3, 2*3) = 6
>   cur_min = min(3, 2*3, 2*3) = 3
>   best = max(2, 6) = 6
> 
> x=-2:
>   x is negative → swap cur_max and cur_min FIRST:
>     temp logic: cur_max, cur_min = cur_min (=3), cur_max (=6)
>   cur_max = max(-2, 3*-2) = -2
>   cur_min = min(-2, 6*-2) = -12
>   best = max(6, -2) = 6
> 
> x=4:
>   no swap (positive)
>   cur_max = max(4, -2*4) = 4
>   cur_min = min(4, -12*4) = -48
>   best = max(6, 4) = 6
> 
> Hmm, expected 6? Let me verify: max subarray product of [2,3,-2,4]:
>   [2]=2, [2,3]=6, [2,3,-2]=-12, [3]=3, [3,-2]=-6, [-2,4]=-8, [4]=4
>   Max = 6 ✓
> 
> ✅ Answer: 6
> ```

> [!success]- Python
> ```python
> def max_product(nums):
>     cur_max = cur_min = best = nums[0]
>     for x in nums[1:]:
>         if x < 0:
>             cur_max, cur_min = cur_min, cur_max
>         cur_max = max(x, cur_max * x)
>         cur_min = min(x, cur_min * x)
>         best = max(best, cur_max)
>     return best
> ```

**Key takeaway:** When a transformation can flip sign, track both extremes.

---

## P8: Word Break

**LC #139** · Medium

> [!example]- 📊 Visual: cut points along the string
> ```text
>   s        = " l e e t c o d e "
>   index i:   0 1 2 3 4 5 6 7 8
>   dict     = {"leet", "code"}
> 
>   dp[i] = True iff s[:i] can be segmented into dictionary words.
> 
>     i:    0    1    2    3    4    5    6    7    8
>   dp[i]:  T    F    F    F    T    F    F    F    T
>           ▲              ▲              ▲              ▲
>           │              │                             │
>          base            │                             │
>                          │                             │
>                          j=0, dp[0]=T,                 │
>                          s[0:4]="leet" ∈ dict → dp[4]=T│
>                                                        │
>                                              j=4, dp[4]=T,
>                                              s[4:8]="code" ∈ dict → dp[8]=T
> 
>   Cut visualisation:
>       l e e t │ c o d e
>       └─leet─┘ └─code─┘     ← two valid words → True
> 
>   For each i, scan all cut points j<i: dp[i] |= dp[j] AND s[j:i] in dict.
> ```

> [!info]- 🔍 Dry Run: s="leetcode", wordDict=["leet","code"]
> ```text
> words = {"leet", "code"}
> dp = [T, F, F, F, F, F, F, F, F]   (dp[0]=True empty prefix)
> 
> i=1: check j=0..0: dp[0]=T && s[0:1]="l" in words? NO
>      dp[1]=F
> i=2: check j=0,1: s[0:2]="le" no; s[1:2]="e" no (and dp[1]=F so irrelevant)
>      dp[2]=F
> i=3: similar — no prefix match. dp[3]=F
> i=4: j=0: dp[0]=T, s[0:4]="leet" in words → YES! dp[4]=T break
> i=5: j=0 dp[0]=T, "leetc" not in words; j=4 dp[4]=T, "c" not in words; etc. dp[5]=F
> i=6: similar. dp[6]=F
> i=7: similar. dp[7]=F
> i=8: j=0 "leetcode" no; j=4 dp[4]=T, s[4:8]="code" in words → YES dp[8]=T
> 
> Return dp[8] = True
> 
> ✅ Answer: true
> ```

> [!success]- Python
> ```python
> def word_break(s, word_dict):
>     words = set(word_dict)
>     max_w = max((len(w) for w in word_dict), default=0)
>     n = len(s)
>     dp = [False] * (n + 1)
>     dp[0] = True
>     for i in range(1, n + 1):
>         for j in range(max(0, i - max_w), i):
>             if dp[j] and s[j:i] in words:
>                 dp[i] = True
>                 break
>     return dp[n]
> ```

**Key takeaway:** Substring DP — `dp[i]` covers prefix `s[:i]`. Loop over split points.

---

## P9: Longest Increasing Subsequence

**LC #300** · Medium

> [!example]- 📊 Visual: patience-sort tails
> ```text
>   nums = [10, 9, 2, 5, 3, 7, 101, 18]
> 
>   Maintain `tails[k]` = smallest possible tail of an LIS of length k+1
> 
>   After each step:
> 
>     after 10:     tails = [10]                                    LIS len 1
>     after 9:      tails = [ 9]                replace 10 (smaller now possible)
>     after 2:      tails = [ 2]                replace 9
>     after 5:      tails = [ 2,  5]            extend (5 > 2)       LIS len 2
>     after 3:      tails = [ 2,  3]            replace 5 (smaller)
>     after 7:      tails = [ 2,  3,  7]        extend               LIS len 3
>     after 101:    tails = [ 2,  3,  7, 101]   extend               LIS len 4
>     after 18:     tails = [ 2,  3,  7,  18]   replace 101
> 
>   final len(tails) = 4   (LIS length = 4; one such LIS: 2,3,7,18 or 2,3,7,101)
> 
>   tails[] is NOT the actual LIS — just monitors length via binary search insertion.
> ```

> [!info]- 🔍 Dry Run: nums=[10,9,2,5,3,7,101,18] (O(n log n) patience method)
> ```text
> tails = []  (tails[i] = smallest possible tail of an LIS of length i+1)
> 
> x=10: bisect_left([],10)=0; tails empty → append 10
>   tails=[10]
> 
> x=9: bisect_left([10],9)=0; replace tails[0]=9
>   tails=[9]
> 
> x=2: bisect_left([9],2)=0; replace tails[0]=2
>   tails=[2]
> 
> x=5: bisect_left([2],5)=1; out of bounds → append 5
>   tails=[2, 5]
> 
> x=3: bisect_left([2,5],3)=1; replace tails[1]=3
>   tails=[2, 3]
> 
> x=7: bisect_left([2,3],7)=2; append 7
>   tails=[2, 3, 7]
> 
> x=101: bisect=3; append
>   tails=[2, 3, 7, 101]
> 
> x=18: bisect=3; replace tails[3]=18
>   tails=[2, 3, 7, 18]
> 
> Length of LIS = len(tails) = 4
> 
> ✅ Answer: 4   (e.g., subsequence [2,3,7,18] or [2,3,7,101])
> 
> Note: tails is NOT the actual LIS; just tracks lengths.
> ```

> [!success]- Python (O(n log n))
> ```python
> from bisect import bisect_left
> def length_of_lis(nums):
>     tails = []
>     for x in nums:
>         i = bisect_left(tails, x)
>         if i == len(tails):
>             tails.append(x)
>         else:
>             tails[i] = x
>     return len(tails)
> ```

> [!success]- Python (O(n²) for clarity)
> ```python
> def length_of_lis_n2(nums):
>     dp = [1] * len(nums)
>     for i in range(1, len(nums)):
>         for j in range(i):
>             if nums[j] < nums[i]:
>                 dp[i] = max(dp[i], dp[j] + 1)
>     return max(dp)
> ```

**Key takeaway:** Subsequence problems → O(n²) DP usually works. For O(n log n), think "patience sorting".

---

## P10: Partition Equal Subset Sum

**LC #416** · Medium

> [!example]- 📊 Visual: 0/1 subset-sum reachability
> ```text
>   nums = [1, 5, 11, 5]    total = 22    target = 11
> 
>   dp[s] = True iff some subset sums to s. Sweep RIGHT→LEFT to enforce 0/1 use.
> 
>   sums:     0  1  2  3  4  5  6  7  8  9  10 11
>   init   :  T  .  .  .  .  .  .  .  .  .   .  .
>   x=1    :  T  T  .  .  .  .  .  .  .  .   .  .         (dp[1] |= dp[0])
>   x=5    :  T  T  .  .  .  T  T  .  .  .   .  .         (5,6 reachable)
>   x=11   :  T  T  .  .  .  T  T  .  .  .   .  T   ←   ★ target hit (11 = {11})
>   x=5    :  T  T  .  .  .  T  T  .  .  .   T  T         (also 10={5,5}, 11={5,11-but-already})
> 
>   Right-to-left iteration: when updating dp[s] from dp[s-x], the dp[s-x] must
>   still be the OLD value (before x was used) so x is not counted twice.
> 
>     for x in nums:
>         for s in range(target, x-1, -1):    ← reverse!
>             dp[s] |= dp[s - x]
> ```

> [!info]- 🔍 Dry Run: nums=[1,5,11,5]
> ```text
> total = 22; even → target = 11
> dp = [T, F, F, F, F, F, F, F, F, F, F, F]  (indices 0..11)
> 
> ─────────────────────────────────────────
> Process x=1:  iterate s from 11 down to 1:
>   s=1: dp[1] = dp[1] or dp[1-1]=dp[0]=T → dp[1]=T
> dp = [T, T, F, F, F, F, F, F, F, F, F, F]
> 
> Process x=5:  iterate s from 11 down to 5:
>   s=11: dp[11] |= dp[6] (F) → F
>   s=10: dp[10] |= dp[5] (F)
>   ...
>   s=6:  dp[6] |= dp[1]=T → T
>   s=5:  dp[5] |= dp[0]=T → T
> dp = [T, T, F, F, F, T, T, F, F, F, F, F]
> 
> Process x=11: s from 11 down:
>   s=11: dp[11] |= dp[0]=T → T   ← found!
>   (also: s=12 out of range; s=10..0 update accordingly)
>   ...
> dp[11] = T
> 
> ✅ Answer: true
> ```

> [!success]- Python
> ```python
> def can_partition(nums):
>     total = sum(nums)
>     if total % 2: return False
>     target = total // 2
>     dp = [False] * (target + 1)
>     dp[0] = True
>     for x in nums:
>         for s in range(target, x - 1, -1):
>             dp[s] = dp[s] or dp[s - x]
>     return dp[target]
> ```

> [!warning] Reverse iteration is critical
> Forward iteration would let one item be used twice. Reverse ensures 0/1.

**Key takeaway:** Subset-sum boolean → 1D rolling DP, sweep right-to-left for 0/1.

---

## P11: Best Time to Buy and Sell Stock with Cooldown

**LC #309** · Medium

> [!example]- 📊 Visual: 3-state machine (held / sold / rest)
> ```text
>                  buy (-price)
>           ┌──────────────────────────┐
>           │                          ▼
>      ┌────────┐                  ┌────────┐
>      │  REST  │ ◄──────────────  │  SOLD  │      (cooldown: must rest after selling)
>      │ (free) │   one day later  │ (just  │
>      └────────┘                  │ sold)  │
>           ▲                      └────┬───┘
>           │                           ▲
>           │       hold (do nothing)   │
>           │                           │
>           │                      sell (+price)
>           │                           │
>           │                       ┌───┴────┐
>           └───────hold◄──────────►│  HELD  │
>                                    │ (own) │
>                                    └────────┘
> 
>   Transitions per day:
>     new_held = max(held,  rest - price)         buy from REST, or keep holding
>     new_sold = held + price                     sell from HELD
>     new_rest = max(rest,  sold)                 stay rest, or graduate from SOLD
> 
>   Answer = max(sold, rest) at the end (can't end while still holding).
> ```

> [!info]- 🔍 Dry Run: prices=[1,2,3,0,2]
> ```text
> held = -INF (own stock)
> sold = 0   (just sold, in cooldown)
> rest = 0   (free to buy)
> 
> p=1:
>   new_held = max(-INF, 0 - 1) = -1
>   new_sold = -INF + 1 = -INF   (can't sell without holding)
>   new_rest = max(0, 0) = 0
>   State: held=-1, sold=-INF, rest=0
> 
> p=2:
>   new_held = max(-1, 0 - 2) = -1
>   new_sold = -1 + 2 = 1
>   new_rest = max(0, -INF) = 0
>   State: held=-1, sold=1, rest=0
> 
> p=3:
>   new_held = max(-1, 0 - 3) = -1
>   new_sold = -1 + 3 = 2
>   new_rest = max(0, 1) = 1
>   State: held=-1, sold=2, rest=1
> 
> p=0:
>   new_held = max(-1, 1 - 0) = 1     ← rest=1 (post-cooldown), buy at 0
>   new_sold = -1 + 0 = -1
>   new_rest = max(1, 2) = 2
>   State: held=1, sold=-1, rest=2
> 
> p=2:
>   new_held = max(1, 2 - 2) = 1
>   new_sold = 1 + 2 = 3
>   new_rest = max(2, -1) = 2
>   State: held=1, sold=3, rest=2
> 
> Return max(sold, rest) = max(3, 2) = 3
> 
> ✅ Answer: 3   (buy at p=1, sell at p=3 → profit 2; cooldown; buy at p=0, sell at p=2 → profit 2; ... wait total 4? Let me recount)
> Actually answer is 3 per spec. Best: buy at 1, sell at 2 (+1); cooldown; buy at 0, sell at 2 (+2). Total +3 ✓
> ```

> [!success]- Python
> ```python
> def max_profit_cooldown(prices):
>     held, sold, rest = float('-inf'), 0, 0
>     for p in prices:
>         held, sold, rest = max(held, rest - p), held + p, max(rest, sold)
>     return max(sold, rest)
> ```

**Key takeaway:** Stock problems = state-machine DP.

---

## P12: Best Time to Buy and Sell Stock with Transaction Fee

**LC #714** · Medium

> [!example]- 📊 Visual: 2-state machine with fee on sell
> ```text
>                     buy (-price)
>           ┌────────────────────────────────┐
>           ▼                                │
>      ┌────────┐                       ┌────┴───┐
>      │  HELD  │ ─── sell (+p-fee) ──► │  FREE  │
>      │ (own)  │                       │ (cash) │
>      └───┬────┘                       └────┬───┘
>          │                                 │
>          └── hold (no-op) ──┐    ┌── hold ─┘
>                             ▼    ▼
>                          stay in current state
> 
>   Transitions per day p:
>     new_held = max(held,           free - p)         keep holding, OR buy
>     new_free = max(free,           held + p - fee)   keep cash,    OR sell-with-fee
> 
>   Fee bites once per round-trip (paid when realising profit on sell).
> 
>   prices = [1, 3, 2, 8, 4, 9]   fee = 2
>     day:   0   1   2   3   4   5
>    held:  -1  -1  -1  -1   1   1     (best capital if holding at end of day)
>    free:   0   0   0   5   5   8     (best capital if cash at end of day)
>                          ▲           ▲
>                          │           │
>             sell at 8: -1+8-2 = 5   sell at 9: 1+9-2 = 8
> ```

> [!info]- 🔍 Dry Run: prices=[1,3,2,8,4,9], fee=2
> ```text
> held = -1 (bought at p=1)
> free = 0
> 
> p=3:
>   held = max(-1, 0-3) = -1
>   free = max(0, -1+3-2) = 0    ← sell at 3, gain 3-1-2=0, no improvement
> 
> p=2:
>   held = max(-1, 0-2) = -1
>   free = max(0, -1+2-2) = 0
> 
> p=8:
>   held = max(-1, 0-8) = -1
>   free = max(0, -1+8-2) = 5    ← sell at 8, profit 5
> 
> p=4:
>   held = max(-1, 5-4) = 1     ← rebought at 4 with current capital 5
>   free = max(5, -1+4-2) = 5
> 
> p=9:
>   held = max(1, 5-9) = 1
>   free = max(5, 1+9-2) = 8    ← sell at 9
> 
> ✅ Answer: 8
> ```

> [!success]- Python
> ```python
> def max_profit_fee(prices, fee):
>     held, free = -prices[0], 0
>     for p in prices[1:]:
>         held = max(held, free - p)
>         free = max(free, held + p - fee)
>     return free
> ```

**Key takeaway:** Two states (held/free) + fee applied on sell.

---

## P13: Perfect Squares

**LC #279** · Medium

> [!example]- 📊 Visual: unbounded knapsack of perfect squares
> ```text
>   n = 12   squares = [1, 4, 9]
> 
>   dp[a] = min #squares summing to a
> 
>     a:      0  1  2  3  4  5  6  7  8  9  10  11  12
>   dp[a]:    0  1  2  3  1  2  3  4  2  1   2   3   3
>             ▲           ▲           ▲      ▲       ▲
>             │           │           │      │       │
>            base         │           │      │       │
>                         │           │      │       │
>            dp[4]  = 1 + dp[0]   (sq=4)
>            dp[8]  = 1 + dp[4]   (sq=4)  → 2
>            dp[9]  = 1 + dp[0]   (sq=9)  → 1
>            dp[12] = 1 + min(dp[11]=3, dp[8]=2, dp[3]=3) = 3   (4+4+4)
> 
>   Same coin-change skeleton: dp[a] = min over sq of dp[a - sq] + 1.
> ```

> [!info]- 🔍 Dry Run: n=12
> ```text
> squares = [1, 4, 9]  (perfect squares ≤ 12)
> dp[0]=0, dp[1..12]=INF
> 
> a=1: c=1: dp[1]=dp[0]+1=1
> a=2: c=1: dp[2]=dp[1]+1=2
> a=3: c=1: dp[3]=dp[2]+1=3
> a=4: c=1: dp[4]=dp[3]+1=4; c=4: dp[4]=min(4, dp[0]+1=1)=1
> a=5: c=1: dp[5]=dp[4]+1=2; c=4: dp[5]=min(2, dp[1]+1=2)=2
> a=6: c=1: dp[6]=dp[5]+1=3; c=4: dp[6]=min(3, dp[2]+1=3)=3
> a=7: c=1: dp[7]=dp[6]+1=4; c=4: dp[7]=min(4, dp[3]+1=4)=4
> a=8: c=1: dp[8]=dp[7]+1=5; c=4: dp[8]=min(5, dp[4]+1=2)=2
> a=9: c=1:3; c=4: dp[9]=min(3, dp[5]+1=3); c=9: dp[9]=min(3, dp[0]+1=1)=1
> a=10: c=1:2 (dp[9]+1); c=4: dp[10]=min(2, dp[6]+1=4)=2; c=9: dp[10]=min(2, dp[1]+1=2)=2
> a=11: c=1:3; c=4: dp[11]=min(3, dp[7]+1=5)=3; c=9: dp[11]=min(3, dp[2]+1=3)=3
> a=12: c=1: dp[12]=dp[11]+1=4; c=4: dp[12]=min(4, dp[8]+1=3)=3; c=9: dp[12]=min(3, dp[3]+1=4)=3
> 
> ✅ Answer: 3    (e.g., 4 + 4 + 4)
> ```

> [!success]- Python
> ```python
> def num_squares(n):
>     dp = [0] + [float('inf')] * n
>     i = 1
>     squares = []
>     while i * i <= n:
>         squares.append(i * i)
>         i += 1
>     for a in range(1, n + 1):
>         for sq in squares:
>             if sq > a: break
>             dp[a] = min(dp[a], dp[a - sq] + 1)
>     return dp[n]
> ```

**Key takeaway:** Same coin-change skeleton; "coins" are perfect squares.

---

## P14: Combination Sum IV

**LC #377** · Medium · Order matters

> [!example]- 📊 Visual: order-matters (permutations) recurrence
> ```text
>   nums = [1, 2, 3]    target = 4
> 
>   dp[t] = # ORDERED sequences from nums summing to t
> 
>     t:      0  1  2  3  4
>   dp[t]:    1  1  2  4  7
>             ▲              ▲
>             │              │
>            base            │
>                            │
>            dp[4] = dp[4-1] + dp[4-2] + dp[4-3]
>                  = dp[3]  + dp[2]  + dp[1]
>                  =   4    +   2    +   1   = 7
> 
>   "Last number in the sequence" branches over nums →
>   target loop OUTER, nums loop INNER ⇒ counts permutations.
> 
>   Swap loops (nums outer, target inner) ⇒ counts combinations (each item once-only path).
> 
>   The 7 sequences:  (1,1,1,1) (1,1,2) (1,2,1) (2,1,1) (2,2) (1,3) (3,1)
> ```

> [!info]- 🔍 Dry Run: nums=[1,2,3], target=4
> ```text
> dp[0] = 1  (empty sequence)
> 
> t=1: dp[1] = dp[1-1] + dp[1-2 (skip n>t)] + dp[1-3 (skip)] = dp[0] = 1
> t=2: dp[2] = dp[1] + dp[0] + (skip) = 1+1 = 2
> t=3: dp[3] = dp[2] + dp[1] + dp[0] = 2+1+1 = 4
> t=4: dp[4] = dp[3] + dp[2] + dp[1] = 4+2+1 = 7
> 
> ✅ Answer: 7
>   Sequences: (1,1,1,1), (1,1,2), (1,2,1), (2,1,1), (2,2), (1,3), (3,1)
> ```

> [!success]- Python
> ```python
> def combination_sum4(nums, target):
>     dp = [0] * (target + 1)
>     dp[0] = 1
>     for t in range(1, target + 1):
>         for n in nums:
>             if n <= t:
>                 dp[t] += dp[t - n]
>     return dp[target]
> ```

> [!tip] Loop order matters
> Outer = target / Inner = nums → counts **permutations** (order matters).
> Outer = nums / Inner = target → counts **combinations** (order doesn't matter).

**Key takeaway:** "Order matters" → outer loop iterates the sum.

---

## P15: Delete and Earn

**LC #740** · Medium

> [!example]- 📊 Visual: bucket-by-value → House Robber
> ```text
>   nums = [2, 2, 3, 3, 3, 4]
> 
>   Step 1: bucket[v] = total points earned by picking ALL copies of value v
> 
>     v:         0  1  2  3  4
>   bucket[v]:   0  0  4  9  4         (two 2s → 4; three 3s → 9; one 4 → 4)
> 
>   Step 2: picking v erases all v-1 and v+1 ⇒ NO ADJACENT values in v-axis.
>           ↳ EXACTLY House Robber on bucket[].
> 
>     v:         0  1  2  3  4
>   bucket[v]:   0  0  4  9  4
>     dp[v]:     0  0  4  9  9
>                ▲     ▲  ▲  ▲
>                │     │  │  │
>               base  rob skip(9) vs rob(4+dp[2]=8) → 9
>                     4   take 9   skip 9 vs rob 4+4=8 → 9
> 
>   Answer = max dp = 9   (take all 3s, lock out 2s and 4s)
> ```

> [!info]- 🔍 Dry Run: nums=[2,2,3,3,3,4]
> ```text
> M = max = 4
> bucket = [0]*(4+1) = [0,0,0,0,0]
> bucket[2] += 2 (twice) = 4   (two 2s, each worth 2)
> bucket[3] += 3 (thrice) = 9
> bucket[4] += 4
> 
> bucket = [0, 0, 4, 9, 4]
> 
> Now run House Robber on this bucket:
>   prev2, prev1 = 0, 0
>   v=0: new = max(0, 0+0) = 0; → prev2=0, prev1=0
>   v=0: same → 0
>   v=4: new = max(0, 0+4) = 4; prev2=0, prev1=4
>   v=9: new = max(4, 0+9) = 9; prev2=4, prev1=9
>   v=4: new = max(9, 4+4) = 9; prev2=9, prev1=9
> 
> Return 9
> 
> ✅ Answer: 9   (take all 3s for sum 9; can't take 2s or 4s)
> ```

> [!success]- Python
> ```python
> def delete_and_earn(nums):
>     if not nums: return 0
>     M = max(nums)
>     bucket = [0] * (M + 1)
>     for x in nums:
>         bucket[x] += x
>     prev2, prev1 = 0, 0
>     for v in bucket:
>         prev2, prev1 = prev1, max(prev1, prev2 + v)
>     return prev1
> ```

**Key takeaway:** Spot the **reduction**. Many DP problems become trivial after the right transform.

---

## P16: Maximum Sum Circular Subarray

**LC #918** · Medium

> [!example]- 📊 Visual: non-wrap vs wrap = total − min slice
> ```text
>   nums = [1, -2, 3, -2]      (visualised as a CIRCLE)
> 
>             ┌───  1 ───┐
>             │          │
>            -2          -2
>             │          │
>             └─── 3 ────┘
> 
>   Two candidates for the answer:
> 
>     A) Non-wrap → ordinary Kadane MAX
>          best contiguous slice in the linear array
>          here: [3]   sum = 3
> 
>     B) Wrap → take EVERYTHING except a contiguous MIN slice
>          ╭──────────────────────────╮
>          │   total  −  min-slice    │  = sum(all) − Kadane_MIN
>          ╰──────────────────────────╯
>          here: total = 0,  min-slice = [-2] = -2  →  0 − (-2) = 2
> 
>   answer = max(A, B) = 3
> 
>   Edge case: if best_max < 0 (all negatives), the wrap formula would pick the
>   empty subarray — invalid. Return best_max directly.
> ```

> [!info]- 🔍 Dry Run: nums=[1,-2,3,-2]
> ```text
> Kadane max: track cur_max, best_max
>   x=1: cur_max=1, best_max=1
>   x=-2: cur_max=max(-2, -1)=-1, best_max=1
>   x=3: cur_max=max(3, 2)=3, best_max=3
>   x=-2: cur_max=max(-2, 1)=1, best_max=3
> 
> Kadane min: track cur_min, best_min
>   x=1: cur_min=1, best_min=1
>   x=-2: cur_min=min(-2, -1)=-2, best_min=-2
>   x=3: cur_min=min(3, 1)=1, best_min=-2
>   x=-2: cur_min=min(-2, -1)=-2, best_min=-2
> 
> total = 0
> 
> Non-wrap max = best_max = 3
> Wrap max = total - best_min = 0 - (-2) = 2
> 
> Since best_max > 0 (not all negatives), answer = max(3, 2) = 3
> 
> ✅ Answer: 3
> ```

> [!success]- Python
> ```python
> def max_subarray_circular(nums):
>     cur_max = best_max = nums[0]
>     cur_min = best_min = nums[0]
>     total = nums[0]
>     for x in nums[1:]:
>         cur_max = max(x, cur_max + x); best_max = max(best_max, cur_max)
>         cur_min = min(x, cur_min + x); best_min = min(best_min, cur_min)
>         total += x
>     if best_max < 0: return best_max
>     return max(best_max, total - best_min)
> ```

**Key takeaway:** Circular array → either non-wrap (Kadane) or "everything except min slice".

---

## P17: Regular Expression Matching

**LC #10** · **Hard**

Match `s` against pattern `p` with `.` (any single char) and `*` (zero or more of preceding element).

### 🧠 Pattern: 2D DP `dp[i][j] = "s[:i] matches p[:j]"`

> Wait — this is a "1D rolling over a 2D matrix" so naturally a 2D DP. Still in 1D-DP topic for grouping. The state machine:
> 
> - If `p[j-1]` is not `*`:
>   - `dp[i][j] = (s[i-1] == p[j-1] or p[j-1] == '.') AND dp[i-1][j-1]`
> - If `p[j-1] == '*'`:
>   - `dp[i][j] = dp[i][j-2]`  (zero occurrences of preceding char)
>   - OR `dp[i][j] = dp[i-1][j]` if `s[i-1]` matches `p[j-2]` (one more occurrence)

> [!example]- 📊 Visual: 2D pattern-vs-text table with `*` two-way branch
> ```text
>   s = "aab"     p = "c*a*b"
> 
>   dp[i][j] = does s[:i] match p[:j]?
> 
>            ""   c   *   a   *   b
>       ""    T   F   T   F   T   F     ← c*, a*  match empty
>        a    F   F   F   T   T   F
>        a    F   F   F   F   T   F
>        b    F   F   F   F   F  ┌T┐    ← s,p both consumed exactly
>                                 └─┘
> 
>   Transitions at (i, j):
> 
>     if p[j-1] == '*':
>          ┌─────────────────┐         ┌──────────────────────────────┐
>          │  ZERO of prev   │   OR    │  ONE MORE of prev  (if match)│
>          │  dp[i][j-2]     │         │  dp[i-1][j]                  │
>          └─────────────────┘         └──────────────────────────────┘
>                  ⇧                              ⇧
>                  │ (skip "c*" pair)             │ (consume one 'a' against a*)
> 
>     else (literal or '.'):
>          ┌─────────────────────────────────────┐
>          │ dp[i-1][j-1]   IF chars match       │
>          └─────────────────────────────────────┘
> 
>   '.' matches any single char; '*' modifies the char BEFORE it.
> ```

> [!info]- 🔍 Dry Run: s="aab", p="c*a*b"
> ```text
> Build dp[m+1][n+1] where m=3 (s), n=5 (p).
> 
> Base: dp[0][0] = True
> Initialize first row (s empty):
>   dp[0][j] = dp[0][j-2] if p[j-1]=='*' else False
>   dp[0][0]=T
>   dp[0][1]: p[0]='c' not *, False
>   dp[0][2]: p[1]='*' → dp[0][0]=T → True (c* matches empty)
>   dp[0][3]: p[2]='a' not *, False
>   dp[0][4]: p[3]='*' → dp[0][2]=T → True (a* matches empty)
>   dp[0][5]: p[4]='b' not *, False
>   First row: [T, F, T, F, T, F]
> 
> i=1 s[0]='a':
>   j=1 p[0]='c': match 'a' vs 'c'? no → dp[1][1]=F
>   j=2 p[1]='*': 
>     option zero: dp[1][0]=F (s='a' not empty)
>     option one-more: matches p[0]='c' vs s[0]='a'? no → option not applicable
>     → F
>   j=3 p[2]='a': match 'a'='a' → dp[1][3] = dp[0][2] = T
>   j=4 p[3]='*':
>     option zero: dp[1][2] = F
>     option one-more: p[2]='a' matches s[0]='a' → dp[0][4] = T → T
>   j=5 p[4]='b': match 'b' vs 'a'? no → F
>   Row 1: [F, F, F, T, T, F]
> 
> i=2 s[1]='a':
>   j=1: 'c' vs 'a' no → F
>   j=2 '*':
>     option zero: dp[2][0]=F
>     option one-more: p[0]='c' vs s[1]='a' no
>     → F
>   j=3 'a' vs 'a' match: dp[1][2]=F → F
>   j=4 '*':
>     option zero: dp[2][2]=F
>     option one-more: p[2]='a' vs s[1]='a' → dp[1][4]=T → T
>   j=5 'b' vs 'a' no → F
>   Row 2: [F, F, F, F, T, F]
> 
> i=3 s[2]='b':
>   j=1 'c' vs 'b' no → F
>   j=2 '*':
>     zero: dp[3][0]=F
>     one-more: 'c' vs 'b' no
>     → F
>   j=3 'a' vs 'b' no → F
>   j=4 '*':
>     zero: dp[3][2]=F
>     one-more: p[2]='a' vs s[2]='b' no
>     → F
>   j=5 'b' vs 'b' match: dp[2][4]=T → T
>   Row 3: [F, F, F, F, F, T]
> 
> Return dp[3][5] = T
> 
> ✅ Answer: true
> ```

> [!success]- Python
> ```python
> def is_match(s, p):
>     m, n = len(s), len(p)
>     dp = [[False] * (n + 1) for _ in range(m + 1)]
>     dp[0][0] = True
>     for j in range(1, n + 1):
>         if p[j - 1] == '*':
>             dp[0][j] = dp[0][j - 2]
>     for i in range(1, m + 1):
>         for j in range(1, n + 1):
>             if p[j - 1] == '*':
>                 dp[i][j] = dp[i][j - 2]   # zero occurrences
>                 if p[j - 2] == '.' or p[j - 2] == s[i - 1]:
>                     dp[i][j] = dp[i][j] or dp[i - 1][j]
>             else:
>                 if p[j - 1] == '.' or p[j - 1] == s[i - 1]:
>                     dp[i][j] = dp[i - 1][j - 1]
>     return dp[m][n]
> ```

**Variants:** Wildcard Matching (LC #44 — `?` and `*` with simpler semantics).

**Key takeaway:** Pattern matching → 2D DP, with `*` as a special two-option transition (zero or one-more).

---

> [!tip] After this drill
> Memorize: state + transition + base + order = a DP. Most 1D DPs are 5 lines. Practice writing them in under 3 minutes.
