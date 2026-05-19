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

16 problems · the most powerful pattern for "best/count/yes-no over sequential choices".

> [!abstract] Pattern recap
> DP = **memoizing decisions over subproblems**. To design:
> 
> 1. **State**: what does `dp[i]` mean? (e.g., "max profit using first i items")
> 2. **Transition**: `dp[i] = f(dp[i-1], dp[i-2], ...)` — express in terms of smaller states
> 3. **Base case**: trivial values for `dp[0]`, `dp[1]`
> 4. **Order**: iterate so dependencies are computed first (usually left-to-right)
> 5. **Answer**: extract from `dp[n]` or `max(dp)`

> [!tip] Two equivalent forms
> **Top-down (memoization):** recursive with cache. Easier to write, natural for tricky bases.
> **Bottom-up (tabulation):** iterative. Slightly faster (no recursion overhead), enables space optimization.
> 
> Convert between them by inverting the recursion. They have the same complexity.

> [!warning] Space optimization
> Most 1D DP only depends on the last 1-2 values. Replace `dp[]` with two variables — O(1) space.

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

---

## P1: Climbing Stairs

**LC #70** · Easy

Climb n stairs taking 1 or 2 steps at a time. How many ways?

### 🧠 Pattern: Fibonacci

> `dp[i] = dp[i-1] + dp[i-2]`. Last step was either 1 or 2.

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

Can't rob adjacent houses. Max sum.

### Approach

`dp[i] = max(dp[i-1], dp[i-2] + nums[i])`. Skip → carry forward; rob → add and use dp[i-2].

> [!success]- Python
> ```python
> def rob(nums):
>     prev2, prev1 = 0, 0
>     for x in nums:
>         prev2, prev1 = prev1, max(prev1, prev2 + x)
>     return prev1
> ```

**Key takeaway:** "Take or skip" decision → max over both choices. The cleanest 2-variable DP.

---

## P4: House Robber II

**LC #213** · Medium · Circular

### Approach

Either you skip house 0 (rob `[1..n-1]`), or skip house n-1 (rob `[0..n-2]`). Two passes of P3.

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

`A=1, B=2, ..., Z=26`. Count decodings.

### Approach

`dp[i] = (single digit valid ? dp[i-1] : 0) + (two digit valid ? dp[i-2] : 0)`. Zeros are tricky.

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

Min coins to make amount.

### 🧠 Pattern: Unbounded Knapsack

> `dp[a] = 1 + min(dp[a - c] for c in coins if c ≤ a)`. Base `dp[0] = 0`. Use ∞ sentinel for impossible.

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

**Variants:** Coin Change II (count combos — slightly different loop order).

**Key takeaway:** "Min items / count ways" with reusable items → unbounded knapsack.

---

## P7: Maximum Product Subarray

**LC #152** · Medium

### 🧠 Pattern: Track Max AND Min

> Negatives flip max ↔ min. Track both. At each step: new_max = max(x, x·prev_max, x·prev_min), and similarly new_min.

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

**Key takeaway:** When a transformation can flip sign, track both extremes. Classic Google trick.

---

## P8: Word Break

**LC #139** · Medium

Can `s` be split into dictionary words?

### Approach

`dp[i] = True` if some j ≤ i with `dp[j] && s[j:i] ∈ dict`.

> [!success]- Python
> ```python
> def word_break(s, word_dict):
>     words = set(word_dict)
>     dp = [False] * (len(s) + 1)
>     dp[0] = True
>     for i in range(1, len(s) + 1):
>         for j in range(i):
>             if dp[j] and s[j:i] in words:
>                 dp[i] = True
>                 break
>     return dp[len(s)]
> ```

> [!tip] Bound the inner loop
> Only consider `j ≥ i - max(word_length)` — skips dead lookups for long substrings.

**Variants:** Word Break II (return all splits — backtracking + memoization).

**Key takeaway:** Substring DP — `dp[i]` covers prefix `s[:i]`. Loop over split points.

---

## P9: Longest Increasing Subsequence

**LC #300** · Medium

### Approach Evolution

1. **O(n²)** `dp[i] = 1 + max(dp[j] for j<i if nums[j]<nums[i])`. Memorable.
2. **O(n log n) — patience sort** — keep `tails[]` of smallest possible tail per length; binary-search to extend or replace.

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

**Variants:** Russian Doll Envelopes · LIS with O(n²) but reconstruct path.

**Key takeaway:** Subsequence problems → O(n²) DP usually works. For O(n log n), think "patience sorting / replace with binary search".

---

## P10: Partition Equal Subset Sum

**LC #416** · Medium

Can we split `nums` into two equal-sum subsets?

### 🧠 Pattern: 0/1 Knapsack on Sums (Boolean DP)

> Target = total/2 (must be integer). `dp[s] = True` if subset sums to `s`. Iterate items; for each, sweep s **from high to low** to avoid reusing.

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

**Variants:** Last Stone Weight II (minimize diff) · Target Sum (count ways).

**Key takeaway:** Subset-sum boolean → 1D rolling DP, sweep right-to-left for 0/1.

---

## P11: Best Time to Buy and Sell Stock with Cooldown

**LC #309** · Medium

### 🧠 Pattern: State Machine DP

> 3 states per day: **held** (own stock), **sold** (just sold, cooldown), **rest** (free to buy). Transition:
> - held[i] = max(held[i-1], rest[i-1] - p[i])
> - sold[i] = held[i-1] + p[i]
> - rest[i] = max(rest[i-1], sold[i-1])

> [!success]- Python
> ```python
> def max_profit_cooldown(prices):
>     held, sold, rest = float('-inf'), 0, 0
>     for p in prices:
>         held, sold, rest = max(held, rest - p), held + p, max(rest, sold)
>     return max(sold, rest)
> ```

**Key takeaway:** Stock problems = state-machine DP. Enumerate states (own/free + cooldown/fee), write transitions.

---

## P12: Best Time to Buy and Sell Stock with Transaction Fee

**LC #714** · Medium

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

Min number of perfect squares summing to n.

### Approach

Unbounded knapsack with `coins = [1, 4, 9, 16, ...]`. Same template as P6.

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

**Key takeaway:** Same coin-change skeleton; the "coins" are the perfect squares.

---

## P14: Combination Sum IV

**LC #377** · Medium · Order matters

Count **sequences** (order matters) summing to target from `nums`.

### Approach

`dp[t] = sum over n in nums of dp[t-n]`. Outer loop on target (order matters → sequences), inner on nums.

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

**Key takeaway:** "Order matters" → outer loop iterates the sum, inner iterates items.

---

## P15: Delete and Earn

**LC #740** · Medium

Earn `x`. Deleting x also forces delete of all x-1 and x+1.

### Approach

Bucket sum by value (`bucket[v] = v · count(v)`). Then this is House Robber on `bucket[0..max]`.

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

**Key takeaway:** Spot the **reduction**. Many DP problems become trivial after the right transform (here: bucket by value).

---

## P16: Maximum Sum Circular Subarray

**LC #918** · Medium

### Approach

Either the max subarray is a non-wrap (Kadane's max) OR the wrap takes everything **except** the min subarray (total - Kadane's min). Edge case: all negatives → return Kadane max only.

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

**Key takeaway:** Circular array → either non-wrap (Kadane) or "everything except min slice". All-negative case is the edge.

---

> [!tip] After this drill
> Memorize: state + transition + base + order = a DP. Most 1D DPs are 5 lines. Practice writing them in under 3 minutes.
