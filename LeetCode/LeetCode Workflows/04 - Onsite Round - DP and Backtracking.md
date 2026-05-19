---
title: "Workflow 04: Onsite Round — DP and Backtracking"
date: 2026-05-19
tags:
  - leetcode
  - workflow
  - mock-interview
  - dp
  - backtracking
parent: "[[LeetCode Study]]"
status: in-progress
---

# Workflow 04: Onsite Round — DP and Backtracking

> [!abstract] Format
> **45 minutes** · **2 problems (1 backtracking, 1 DP)** · the "hard-by-default" round. Recruiters often save this for the senior loop. Pattern recognition + state design are critical.

---

## Problem 1: Combination Sum

**LC #39** · Medium · Backtracking

> [!example] Prompt
> "Given distinct positive integers `candidates` and a `target`, return all unique combinations of candidates summing to target. The **same number may be chosen unlimited times**."

### 🎬 Hidden Script

> [!example]- What they're testing
> - Recognize "all combinations" → backtracking
> - Handle "reuse allowed" → on recursion, pass `j` not `j+1`
> - Prune via sort + early break
> 
> **Red flag:** code without the choose/un-choose pattern. Or generating permutations (different orderings of same combo).

### Walkthrough

**Clarify:** "Same combo in different order counted once? (Yes.)" "Negatives allowed? (No — would infinite loop.)" "Large target? (Be ready to discuss exponential blow-up.)"

**Approach:** Sort candidates (enables pruning). DFS with `(start_index, remaining)`. At each call, try each candidate from `start` onward. On reuse, recurse with same `start`. On exhaustion or excess, prune.

**Code:**

> [!success]- Python
> ```python
> def combination_sum(candidates, target):
>     candidates.sort()
>     out, path = [], []
>     def dfs(i, remain):
>         if remain == 0:
>             out.append(path[:])
>             return
>         for j in range(i, len(candidates)):
>             if candidates[j] > remain: break   # prune (sorted)
>             path.append(candidates[j])
>             dfs(j, remain - candidates[j])     # j, not j+1 → reuse OK
>             path.pop()
>     dfs(0, target)
>     return out
> ```

**Trace:** `candidates=[2,3,6,7]` `target=7`
```
dfs(0, 7)
  pick 2: path=[2] dfs(0, 5)
    pick 2: path=[2,2] dfs(0, 3)
      pick 2: path=[2,2,2] dfs(0, 1)
        2 > 1 → break
      pop 2
      pick 3: path=[2,2,3] dfs(1, 0) → APPEND
      pop 3, break (6>1)
    pop 2
    pick 3: path=[2,3] dfs(1, 2)
      3 > 2 → break
    pop 3
    pick 6,7 too big → break
  pop 2
  pick 3: path=[3] dfs(1, 4)
    pick 3: path=[3,3] dfs(1, 1)
      3>1 → break
    pop 3
    pick 6: 6>4 → break
  pop 3
  pick 7: path=[7] dfs(3, 0) → APPEND
return [[2,2,3], [7]]
```

**Complexity:** Time O(2^target / min_candidate) worst case; space O(target / min_candidate) recursion depth.

**Edge cases:**
- Target = 0 → return `[[]]`
- No combination → `[]`
- Single candidate that doesn't divide target → `[]`

### Follow-up

**Q:** "What if each number can only be used **once** (Combination Sum II)?"

**A:** Pass `j + 1` instead of `j`. And to dedup when there are duplicate candidates: sort + skip `nums[j] == nums[j-1]` when `j > i` (same-level skip).

**Q:** "What if we just want the **count**, not the combinations themselves?"

**A:** Use DP. `dp[t] = sum(dp[t - c] for c in candidates if c ≤ t)`. O(target × len(candidates)) time, O(target) space. Way faster.

---

## Problem 2: Word Break

**LC #139** · Medium · DP

> [!example] Prompt
> "Given `s` and a dictionary `wordDict`, return true if `s` can be segmented into a sequence of dictionary words."

### 🎬 Hidden Script

> [!example]- What they're testing
> - Try naive recursion first → mention exponential blow-up
> - Add memoization → DP
> - Recognize as a **substring DP** (state = prefix length)
> 
> **Red flag:** thinking it's a trie problem (it can be, but DP is the canonical answer).

### Walkthrough

**Clarify:** "Words can be reused? (Yes.)" "Case-sensitive? (Yes — clarify.)" "Empty s? (Trivially true.)"

**Approach Evolution:**
1. **Naive recursion** — for each prefix that's a word, recurse on suffix. Exponential.
2. **Memoize** — cache results per starting index.
3. **Bottom-up DP** — `dp[i] = True` iff `s[:i]` is segmentable. For each `i`, check all `j < i`: if `dp[j] && s[j:i] in dict`, then `dp[i] = True`.

**Code:**

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

> [!tip] The `max_w` optimization
> Without it, inner loop is O(n). With it, O(max word length). For `s` of size 10⁵ and short dict words, this is the difference between AC and TLE.

**Trace:** `s="leetcode"` `dict=["leet","code"]`
```
dp = [T, F, F, F, F, F, F, F, F]
i=4: j=0 dp[0]=T, "leet" in dict → dp[4]=T
i=8: j=4 dp[4]=T, "code" in dict → dp[8]=T
return dp[8] = True
```

**Complexity:** Time O(n · max_w · L) where L is hash cost of substring. Space O(n).

**Edge cases:**
- Empty s → True
- Single char in dict → trivial
- s longer than any dict word can build → False

### Follow-up

**Q:** "What if we want **all** valid segmentations (Word Break II)?"

**A:** Combine DP with backtracking. Use `dp[i] = list of decompositions of s[:i]`. Or first run dp to know which prefixes are valid; then DFS only down valid prefixes.

**Q:** "What if `s` is huge but the dictionary is small?"

**A:** **Trie + DFS** beats DP here. Insert dict into trie. DFS from index 0; at each position, walk trie + advance index; on `isEnd`, recurse on remaining string with memoization.

---

## Self-Eval Checklist

- [ ] Backtracking: did I draw the recursion tree before coding?
- [ ] Backtracking: did I show choose → recurse → un-choose explicitly?
- [ ] DP: did I state the recurrence in words before writing it?
- [ ] DP: did I identify base case and answer cell?
- [ ] Did I propose a brute force AND the optimization?

---

## Extensions

Other "DP / Backtracking" round combos:
- LC #51 N-Queens + LC #300 Longest Increasing Subsequence
- LC #79 Word Search + LC #322 Coin Change
- LC #131 Palindrome Partitioning + LC #72 Edit Distance
- LC #46 Permutations + LC #198 House Robber
