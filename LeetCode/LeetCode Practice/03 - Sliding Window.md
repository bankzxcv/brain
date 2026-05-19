---
title: "LeetCode Practice: Sliding Window"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - sliding-window
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Sliding Window

12 problems · 2 sub-patterns: **fixed window** and **variable window**.

> [!abstract] Pattern recap
> **Fixed window** — size `k`. Slide one step at a time: add new, remove old.
> **Variable window** — `l` and `r` expand/shrink based on a condition. Generic template:
> ```
> l = 0
> for r in range(n):
>     add(nums[r])
>     while window is INVALID:
>         remove(nums[l])
>         l += 1
>     # window [l..r] is now valid → update answer
> ```

> [!warning] When sliding window FAILS
> Negatives break the "expand → sum increases" assumption. If sums aren't monotonic with window growth, use **prefix sum + hash** instead (see P9 in Arrays and Hashing).

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Best Time to Buy and Sell Stock | 121 | Easy | Track min, variable |
| P2 | Longest Substring Without Repeating Characters | 3 | Med | Variable + set |
| P3 | Longest Repeating Character Replacement | 424 | Med | Variable + max freq |
| P4 | Permutation in String | 567 | Med | Fixed + char count |
| P5 | Find All Anagrams in a String | 438 | Med | Fixed + char count |
| P6 | Minimum Window Substring | 76 | Hard | Variable + need count |
| P7 | Sliding Window Maximum | 239 | Hard | Fixed + monotonic deque |
| P8 | Maximum Average Subarray I | 643 | Easy | Fixed window |
| P9 | Subarrays with K Different Integers | 992 | Hard | At-most-k − at-most-(k−1) |
| P10 | Fruit Into Baskets | 904 | Med | Variable, ≤ 2 distinct |
| P11 | Max Consecutive Ones III | 1004 | Med | Variable, ≤ k zeros |
| P12 | Count Number of Nice Subarrays | 1248 | Med | At-most-k transformation |

---

## P1: Best Time to Buy and Sell Stock

**LC #121** · Easy

Max profit from one buy + one later sell.

**Edge cases:** monotonic decreasing (profit=0) · single price · two prices.

### 🧠 Pattern: Track Min-So-Far (Degenerate Window)

> One-pass: keep the cheapest price seen; profit at `i` = `price[i] - cheapest`.

### Trace

```
prices=[7,1,5,3,6,4]
min=7 profit=0
i=1 p=1 → min=1
i=2 p=5 → profit=max(0, 5-1)=4
i=3 p=3 → profit=4
i=4 p=6 → profit=max(4, 6-1)=5
i=5 p=4 → profit=5
return 5
```

> [!success]- JS
> ```js
> const maxProfit = (prices) => {
>   let lo = prices[0], best = 0;
>   for (let i = 1; i < prices.length; i++) {
>     lo = Math.min(lo, prices[i]);
>     best = Math.max(best, prices[i] - lo);
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def max_profit(prices):
>     lo, best = prices[0], 0
>     for p in prices[1:]:
>         lo = min(lo, p)
>         best = max(best, p - lo)
>     return best
> ```

**Variants:** Stock II (multiple transactions) · Stock III/IV (k transactions, DP) · with cooldown / fee.

**Key takeaway:** "Best diff in order" → track running min/max.

---

## P2: Longest Substring Without Repeating Characters

**LC #3** · Medium

Length of longest substring with all unique chars.

**Edge cases:** empty · all same chars · Unicode (clarify).

### 🧠 Pattern: Variable Window + Set

> Expand `r` while window valid (no repeat). On repeat, shrink `l` until valid again. Track max window size.

### Trace

```
s="abcabcbb"
l=0
r=0 'a' set={a}         best=1
r=1 'b' set={a,b}       best=2
r=2 'c' set={a,b,c}     best=3
r=3 'a' repeat → remove s[l]='a', l=1, set={b,c}; add 'a' set={b,c,a} best=3
r=4 'b' repeat → remove 'b', l=2; add 'b' set={c,a,b} best=3
... (continues)
return 3
```

> [!success]- JS
> ```js
> const lengthOfLongestSubstring = (s) => {
>   const set = new Set();
>   let l = 0, best = 0;
>   for (let r = 0; r < s.length; r++) {
>     while (set.has(s[r])) set.delete(s[l++]);
>     set.add(s[r]);
>     best = Math.max(best, r - l + 1);
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def length_of_longest_substring(s):
>     seen = set()
>     l = best = 0
>     for r, c in enumerate(s):
>         while c in seen:
>             seen.remove(s[l])
>             l += 1
>         seen.add(c)
>         best = max(best, r - l + 1)
>     return best
> ```

**Variants:** With at most k distinct chars · Longest Substring with At Least K Repeating Characters (divide & conquer).

**Key takeaway:** Variable window template; expand → shrink-while-invalid → update.

---

## P3: Longest Repeating Character Replacement

**LC #424** · Medium

Replace at most `k` chars; find longest substring of one repeated char.

**Edge cases:** k ≥ n (whole string) · all same already · k = 0.

### 🧠 Pattern: Variable Window + Max Frequency

> Window valid iff `windowLen - maxFreq ≤ k`. Track char counts and running `maxFreq`. **Don't need to decrement `maxFreq`** when shrinking — only the answer matters, and it can't shrink.

### Trace

```
s="AABABBA"  k=1
l=0 r=0 cnt[A]=1 maxF=1 winLen=1 valid (1-1=0≤1) best=1
   r=1 cnt[A]=2 maxF=2 winLen=2 valid best=2
   r=2 cnt[B]=1 maxF=2 winLen=3 valid best=3
   r=3 cnt[A]=3 maxF=3 winLen=4 valid best=4
   r=4 cnt[B]=2 maxF=3 winLen=5 invalid (5-3=2>1) → drop s[0]=A, l=1
   r=5 cnt[B]=3 maxF=3 winLen=5 invalid → drop s[1]=A, l=2
   r=6 cnt[A]=2 maxF=3 winLen=5 still=5  best=4
return 4
```

> [!success]- JS
> ```js
> const characterReplacement = (s, k) => {
>   const cnt = new Array(26).fill(0);
>   const A = 'A'.charCodeAt(0);
>   let l = 0, maxF = 0, best = 0;
>   for (let r = 0; r < s.length; r++) {
>     maxF = Math.max(maxF, ++cnt[s.charCodeAt(r) - A]);
>     if (r - l + 1 - maxF > k) cnt[s.charCodeAt(l++) - A]--;
>     best = Math.max(best, r - l + 1);
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def character_replacement(s, k):
>     cnt = {}
>     l = max_f = best = 0
>     for r, c in enumerate(s):
>         cnt[c] = cnt.get(c, 0) + 1
>         max_f = max(max_f, cnt[c])
>         if r - l + 1 - max_f > k:
>             cnt[s[l]] -= 1
>             l += 1
>         best = max(best, r - l + 1)
>     return best
> ```

**Key takeaway:** "Window is valid if `len - maxFreq ≤ k`". MaxFreq doesn't need decrementing.

---

## P4: Permutation in String

**LC #567** · Medium

Does `s2` contain any permutation of `s1` as a substring?

**Edge cases:** `len(s1) > len(s2)` (false) · empty s1 (true) · identical strings.

### 🧠 Pattern: Fixed Window + Frequency Match

> Window of size `len(s1)` over `s2`. Compare two 26-int arrays.

### Trace

```
s1="ab" s2="eidbaooo"
target[a]=1 target[b]=1
window 'ei' [a]=0 [b]=0  ≠
slide → 'id'           ≠
slide → 'db' [d]=1 [b]=1  ≠ (extra d)
slide → 'ba' [b]=1 [a]=1  = target ✓ → true
```

> [!success]- JS
> ```js
> const checkInclusion = (s1, s2) => {
>   if (s1.length > s2.length) return false;
>   const A = 'a'.charCodeAt(0);
>   const t = new Array(26).fill(0), w = new Array(26).fill(0);
>   for (let i = 0; i < s1.length; i++) {
>     t[s1.charCodeAt(i) - A]++;
>     w[s2.charCodeAt(i) - A]++;
>   }
>   const same = () => t.every((v, i) => v === w[i]);
>   if (same()) return true;
>   for (let i = s1.length; i < s2.length; i++) {
>     w[s2.charCodeAt(i) - A]++;
>     w[s2.charCodeAt(i - s1.length) - A]--;
>     if (same()) return true;
>   }
>   return false;
> };
> ```

> [!success]- Python
> ```python
> from collections import Counter
> def check_inclusion(s1, s2):
>     if len(s1) > len(s2): return False
>     t, w = Counter(s1), Counter(s2[:len(s1)])
>     if w == t: return True
>     for i in range(len(s1), len(s2)):
>         w[s2[i]] += 1
>         w[s2[i - len(s1)]] -= 1
>         if w[s2[i - len(s1)]] == 0: del w[s2[i - len(s1)]]
>         if w == t: return True
>     return False
> ```

**Variants:** Find All Anagrams (P5 — same trick, collect starts) · Substring with Concatenation of All Words.

**Key takeaway:** Fixed window + char counts. Maintain counts incrementally, not from scratch.

---

## P5: Find All Anagrams in a String

**LC #438** · Medium

Same as P4 but **return all starting indices**.

> [!success]- JS
> ```js
> const findAnagrams = (s, p) => {
>   const out = [];
>   if (p.length > s.length) return out;
>   const A = 'a'.charCodeAt(0);
>   const t = new Array(26).fill(0), w = new Array(26).fill(0);
>   for (let i = 0; i < p.length; i++) {
>     t[p.charCodeAt(i) - A]++;
>     w[s.charCodeAt(i) - A]++;
>   }
>   const same = () => t.every((v, i) => v === w[i]);
>   if (same()) out.push(0);
>   for (let i = p.length; i < s.length; i++) {
>     w[s.charCodeAt(i) - A]++;
>     w[s.charCodeAt(i - p.length) - A]--;
>     if (same()) out.push(i - p.length + 1);
>   }
>   return out;
> };
> ```

> [!success]- Python
> ```python
> from collections import Counter
> def find_anagrams(s, p):
>     out = []
>     if len(p) > len(s): return out
>     t, w = Counter(p), Counter(s[:len(p)])
>     if w == t: out.append(0)
>     for i in range(len(p), len(s)):
>         w[s[i]] += 1
>         w[s[i - len(p)]] -= 1
>         if w[s[i - len(p)]] == 0: del w[s[i - len(p)]]
>         if w == t: out.append(i - len(p) + 1)
>     return out
> ```

**Key takeaway:** P5 = P4 + collect indices. Same skeleton.

---

## P6: Minimum Window Substring

**LC #76** · Hard

Smallest substring of `s` containing every char of `t` (with multiplicity).

**Edge cases:** `len(t) > len(s)` (return "") · `t` has duplicates · all chars equal.

### 🧠 Pattern: Variable Window + Need Counter

> Track how many distinct chars still need their full count. Expand `r` until `need == 0`; then shrink `l` while still valid, recording min window.

### Trace

```
s="ADOBECODEBANC"  t="ABC"
need = {A:1, B:1, C:1}  required=3 (distinct chars matched)
r=0 'A' have[A]=1 matches A → required=2
r=3 'B' matches → required=1
r=5 'C' matches → required=0 ✓  window=ADOBEC (len 6) → start shrinking
  l=0 'A' have[A]=0<need → required=1; window invalid, stop
r=6..  expand until 'A' matches again
... best window = "BANC" (len 4)
```

> [!success]- JS
> ```js
> const minWindow = (s, t) => {
>   if (t.length > s.length) return "";
>   const need = new Map();
>   for (const c of t) need.set(c, (need.get(c) ?? 0) + 1);
>   let missing = t.length;
>   let l = 0, bestL = 0, bestLen = Infinity;
>   for (let r = 0; r < s.length; r++) {
>     if ((need.get(s[r]) ?? 0) > 0) missing--;
>     need.set(s[r], (need.get(s[r]) ?? 0) - 1);
>     while (missing === 0) {
>       if (r - l + 1 < bestLen) { bestLen = r - l + 1; bestL = l; }
>       need.set(s[l], need.get(s[l]) + 1);
>       if (need.get(s[l]) > 0) missing++;
>       l++;
>     }
>   }
>   return bestLen === Infinity ? "" : s.slice(bestL, bestL + bestLen);
> };
> ```

> [!success]- Python
> ```python
> from collections import Counter
> def min_window(s, t):
>     if len(t) > len(s): return ""
>     need = Counter(t)
>     missing = len(t)
>     l = best_l = 0
>     best_len = float('inf')
>     for r, c in enumerate(s):
>         if need[c] > 0: missing -= 1
>         need[c] -= 1
>         while missing == 0:
>             if r - l + 1 < best_len:
>                 best_len = r - l + 1
>                 best_l = l
>             need[s[l]] += 1
>             if need[s[l]] > 0: missing += 1
>             l += 1
>     return "" if best_len == float('inf') else s[best_l:best_l + best_len]
> ```

> [!tip] The `missing` trick
> Counting only chars `need[c] > 0` (not negatives from excess) lets you check validity with a single integer.

**Variants:** Substring with Concatenation of All Words · Smallest Range Covering K Lists (heap).

**Key takeaway:** Variable window with multiset condition → `need` counter + `missing` counter.

---

## P7: Sliding Window Maximum

**LC #239** · Hard

Max in each window of size `k`.

**Edge cases:** k = 1 · k = n · all equal.

### 🧠 Pattern: Monotonic Deque (Decreasing)

> Maintain a deque of **indices** whose values are decreasing. Head is always the max of current window. Push `r`: pop tail while smaller. Drop head if out of window.

### Trace

```
nums=[1,3,-1,-3,5,3,6,7]  k=3
r=0  push 0; deque=[0]   no output (window not full)
r=1  3 > nums[0]=1 → pop; push 1; deque=[1]
r=2 -1 < 3 → push 2; deque=[1,2]   window full → out[0]=nums[1]=3
r=3 -3 < -1 → push 3; deque=[1,2,3]  head=1 in range → out[1]=3
r=4  5 > all → pop 3,2,1; push 4; deque=[4]  out[2]=5
r=5  3 < 5 → push 5; deque=[4,5]  out[3]=5
r=6  6 > 3 → pop 5; 6 < 5? no, pop 4? 6>5 → pop; push 6; deque=[6]  out[4]=6
r=7  7 > 6 → pop 6; push 7; deque=[7]  out[5]=7
return [3,3,5,5,6,7]
```

> [!success]- JS
> ```js
> const maxSlidingWindow = (nums, k) => {
>   const dq = [], out = [];
>   for (let r = 0; r < nums.length; r++) {
>     while (dq.length && nums[dq[dq.length - 1]] < nums[r]) dq.pop();
>     dq.push(r);
>     if (dq[0] <= r - k) dq.shift();
>     if (r >= k - 1) out.push(nums[dq[0]]);
>   }
>   return out;
> };
> ```

> [!success]- Python
> ```python
> from collections import deque
> def max_sliding_window(nums, k):
>     dq, out = deque(), []
>     for r, x in enumerate(nums):
>         while dq and nums[dq[-1]] < x:
>             dq.pop()
>         dq.append(r)
>         if dq[0] <= r - k:
>             dq.popleft()
>         if r >= k - 1:
>             out.append(nums[dq[0]])
>     return out
> ```

**Variants:** Min in window · First Negative in Window · Shortest Subarray with Sum at Least K (deque on prefix sums).

**Key takeaway:** Window max in O(n) → monotonic decreasing deque of indices.

---

## P8: Maximum Average Subarray I

**LC #643** · Easy

Average of best window of size `k`.

### Approach

Classic fixed window: running sum, slide by 1.

> [!success]- JS
> ```js
> const findMaxAverage = (nums, k) => {
>   let sum = 0;
>   for (let i = 0; i < k; i++) sum += nums[i];
>   let best = sum;
>   for (let i = k; i < nums.length; i++) {
>     sum += nums[i] - nums[i - k];
>     best = Math.max(best, sum);
>   }
>   return best / k;
> };
> ```

> [!success]- Python
> ```python
> def find_max_average(nums, k):
>     s = sum(nums[:k])
>     best = s
>     for i in range(k, len(nums)):
>         s += nums[i] - nums[i - k]
>         best = max(best, s)
>     return best / k
> ```

**Key takeaway:** The textbook fixed window. Always maintain sum incrementally.

---

## P9: Subarrays with K Different Integers

**LC #992** · Hard

Count subarrays containing **exactly** k distinct ints.

### 🧠 Pattern: "Exactly K" = "At most K" − "At most K−1"

> Counting subarrays with **exactly** k distinct is hard directly. Define `atMost(k)` (variable window, O(n)). Then `exactly(k) = atMost(k) - atMost(k-1)`.

### Approach

> [!success]- JS
> ```js
> const subarraysWithKDistinct = (nums, k) => {
>   const atMost = (k) => {
>     const cnt = new Map();
>     let l = 0, total = 0;
>     for (let r = 0; r < nums.length; r++) {
>       cnt.set(nums[r], (cnt.get(nums[r]) ?? 0) + 1);
>       while (cnt.size > k) {
>         cnt.set(nums[l], cnt.get(nums[l]) - 1);
>         if (cnt.get(nums[l]) === 0) cnt.delete(nums[l]);
>         l++;
>       }
>       total += r - l + 1;
>     }
>     return total;
>   };
>   return atMost(k) - atMost(k - 1);
> };
> ```

> [!success]- Python
> ```python
> def subarrays_with_k_distinct(nums, k):
>     def at_most(k):
>         cnt = {}
>         l = total = 0
>         for r, x in enumerate(nums):
>             cnt[x] = cnt.get(x, 0) + 1
>             while len(cnt) > k:
>                 cnt[nums[l]] -= 1
>                 if cnt[nums[l]] == 0: del cnt[nums[l]]
>                 l += 1
>             total += r - l + 1
>         return total
>     return at_most(k) - at_most(k - 1)
> ```

> [!tip] `total += r - l + 1` per step
> Counts all new subarrays **ending at r** within the current valid window.

**Variants:** P12 (nice subarrays = exactly k odd) · Binary Subarrays with Sum.

**Key takeaway:** "Exactly K" → difference of two "at most" counts. Each `atMost` is variable window.

---

## P10: Fruit Into Baskets

**LC #904** · Medium

Longest subarray with at most 2 distinct values.

### Approach

Direct application: `atMost(2)` from P9 (but return max window len, not count).

> [!success]- JS
> ```js
> const totalFruit = (fruits) => {
>   const cnt = new Map();
>   let l = 0, best = 0;
>   for (let r = 0; r < fruits.length; r++) {
>     cnt.set(fruits[r], (cnt.get(fruits[r]) ?? 0) + 1);
>     while (cnt.size > 2) {
>       cnt.set(fruits[l], cnt.get(fruits[l]) - 1);
>       if (cnt.get(fruits[l]) === 0) cnt.delete(fruits[l]);
>       l++;
>     }
>     best = Math.max(best, r - l + 1);
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def total_fruit(fruits):
>     cnt = {}
>     l = best = 0
>     for r, f in enumerate(fruits):
>         cnt[f] = cnt.get(f, 0) + 1
>         while len(cnt) > 2:
>             cnt[fruits[l]] -= 1
>             if cnt[fruits[l]] == 0: del cnt[fruits[l]]
>             l += 1
>         best = max(best, r - l + 1)
>     return best
> ```

**Variants:** Longest Substring with At Most K Distinct Characters.

**Key takeaway:** "Longest with at most k distinct" → variable window + map.

---

## P11: Max Consecutive Ones III

**LC #1004** · Medium

Longest subarray of 1s after flipping **at most k zeros**.

### 🧠 Pattern: Variable Window with ≤ k "Violations"

### Approach

> [!success]- JS
> ```js
> const longestOnes = (nums, k) => {
>   let l = 0, zeros = 0, best = 0;
>   for (let r = 0; r < nums.length; r++) {
>     if (nums[r] === 0) zeros++;
>     while (zeros > k) {
>       if (nums[l] === 0) zeros--;
>       l++;
>     }
>     best = Math.max(best, r - l + 1);
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def longest_ones(nums, k):
>     l = zeros = best = 0
>     for r, x in enumerate(nums):
>         if x == 0: zeros += 1
>         while zeros > k:
>             if nums[l] == 0: zeros -= 1
>             l += 1
>         best = max(best, r - l + 1)
>     return best
> ```

**Variants:** Max Consecutive Ones · Longest Subarray of 1's After Deleting One Element (k=1, but must delete).

**Key takeaway:** Allow up to k bad elements → variable window with a counter.

---

## P12: Count Number of Nice Subarrays

**LC #1248** · Medium

Count subarrays with **exactly** k odd numbers.

### Approach

Same trick as P9: `exactly(k) = atMost(k) - atMost(k-1)`. Window counter = number of odds.

> [!success]- JS
> ```js
> const numberOfSubarrays = (nums, k) => {
>   const atMost = (k) => {
>     let l = 0, odds = 0, total = 0;
>     for (let r = 0; r < nums.length; r++) {
>       if (nums[r] % 2 === 1) odds++;
>       while (odds > k) {
>         if (nums[l] % 2 === 1) odds--;
>         l++;
>       }
>       total += r - l + 1;
>     }
>     return total;
>   };
>   return atMost(k) - atMost(k - 1);
> };
> ```

> [!success]- Python
> ```python
> def number_of_subarrays(nums, k):
>     def at_most(k):
>         l = odds = total = 0
>         for r, x in enumerate(nums):
>             if x % 2 == 1: odds += 1
>             while odds > k:
>                 if nums[l] % 2 == 1: odds -= 1
>                 l += 1
>             total += r - l + 1
>         return total
>     return at_most(k) - at_most(k - 1)
> ```

**Key takeaway:** Map "odd count" to "distinct count" — same atMost-difference trick generalizes.

---

> [!tip] After this drill
> Memorize the variable-window template. The `while invalid: shrink` line is the workhorse of 60% of string problems.
