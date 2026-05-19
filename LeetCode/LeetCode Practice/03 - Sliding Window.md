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
| P6 | Minimum Window Substring | 76 | **Hard** | Variable + need count |
| P7 | Sliding Window Maximum | 239 | **Hard** | Fixed + monotonic deque |
| P8 | Maximum Average Subarray I | 643 | Easy | Fixed window |
| P9 | Subarrays with K Different Integers | 992 | **Hard** | At-most-k − at-most-(k−1) |
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

> [!info]- 🔍 Dry Run: prices=[7,1,5,3,6,4]
> ```text
> Setup:
>   lo   = prices[0] = 7
>   best = 0
> 
> ─────────────────────────────────────────
> Step 1: i=1, p=1
>   p < lo?  → 1 < 7, YES → lo = 1
>   profit = p - lo = 1 - 1 = 0
>   best = max(0, 0) = 0
> 
> Step 2: i=2, p=5
>   p < lo?  → 5 < 1, NO
>   profit = 5 - 1 = 4
>   best = max(0, 4) = 4
> 
> Step 3: i=3, p=3
>   p < lo?  → 3 < 1, NO
>   profit = 3 - 1 = 2
>   best = 4 (unchanged)
> 
> Step 4: i=4, p=6
>   profit = 6 - 1 = 5
>   best = max(4, 5) = 5  ✓
> 
> Step 5: i=5, p=4
>   profit = 4 - 1 = 3
>   best = 5 (unchanged)
> 
> ✅ Answer: 5   (buy at price 1 on day 1, sell at price 6 on day 4)
> ```

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

> [!info]- 🔍 Dry Run: s="abcabcbb"
> ```text
> Setup:
>   seen = {} (set)
>   l = 0, best = 0
> 
> ─────────────────────────────────────────
> Step 1: r=0, s[r]='a'
>   'a' in seen?  → NO
>   seen.add('a')  → seen={a}
>   window len = r-l+1 = 1
>   best = max(0, 1) = 1
> 
> Step 2: r=1, s[r]='b'
>   'b' in seen?  → NO
>   seen.add('b')  → seen={a,b}
>   window "ab", len 2
>   best = 2
> 
> Step 3: r=2, s[r]='c'
>   'c' in seen?  → NO
>   seen={a,b,c}, window "abc", best=3
> 
> Step 4: r=3, s[r]='a'
>   'a' in seen?  → YES → SHRINK
>     while 'a' in seen: remove s[l=0]='a', l=1; seen={b,c}
>   seen.add('a') → seen={b,c,a}, window "bca", len=3
>   best = 3 (unchanged)
> 
> Step 5: r=4, s[r]='b'
>   'b' in seen → SHRINK
>     remove s[l=1]='b', l=2; seen={c,a}
>   seen.add('b') → seen={c,a,b}, window "cab", len=3
>   best = 3
> 
> Step 6: r=5, s[r]='c'
>   'c' in seen → SHRINK
>     remove s[l=2]='c', l=3; seen={a,b}
>   seen.add('c') → seen={a,b,c}, window "abc", len=3
>   best = 3
> 
> Step 7: r=6, s[r]='b'
>   'b' in seen → SHRINK
>     remove s[l=3]='a', l=4; seen={b,c}
>     'b' still in seen → remove s[l=4]='b', l=5; seen={c}
>   seen.add('b') → seen={c,b}, window "cb", len=2
>   best = 3
> 
> Step 8: r=7, s[r]='b'
>   'b' in seen → SHRINK
>     remove s[l=5]='c', l=6; seen={b}
>     'b' still in seen → remove s[l=6]='b', l=7; seen={}
>   seen.add('b') → seen={b}, window "b", len=1
>   best = 3
> 
> ✅ Answer: 3   (one of "abc", "bca", "cab")
> ```

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

> [!info]- 🔍 Dry Run: s="AABABBA", k=1
> ```text
> Setup:
>   cnt = {} (Counter)
>   l = 0, maxF = 0, best = 0
> 
> Validity rule:  (r-l+1) - maxF ≤ k
> 
> ─────────────────────────────────────────
> r=0, c='A'
>   cnt[A]++ → cnt={A:1}, maxF = 1
>   winLen = 1, 1-1=0 ≤ k=1 → valid
>   best = max(0, 1) = 1
> 
> r=1, c='A'
>   cnt[A]++ → cnt={A:2}, maxF = 2
>   winLen = 2, 2-2=0 ≤ 1 → valid
>   best = 2
> 
> r=2, c='B'
>   cnt[B]++ → cnt={A:2,B:1}, maxF = 2 (unchanged)
>   winLen = 3, 3-2=1 ≤ 1 → valid
>   best = 3
> 
> r=3, c='A'
>   cnt[A]++ → cnt={A:3,B:1}, maxF = 3
>   winLen = 4, 4-3=1 ≤ 1 → valid
>   best = 4
> 
> r=4, c='B'
>   cnt[B]++ → cnt={A:3,B:2}, maxF = 3
>   winLen = 5, 5-3=2 > 1 → INVALID → shrink
>     drop s[l=0]='A': cnt[A]-- → cnt={A:2,B:2}, l=1
>     winLen now 4 (we only shrink once; maxF stays 3)
>   best = 4 (no improvement)
> 
> r=5, c='B'
>   cnt[B]++ → cnt={A:2,B:3}, maxF = 3
>   winLen = 5, 5-3=2 > 1 → INVALID → shrink
>     drop s[l=1]='A': cnt={A:1,B:3}, l=2
>     winLen=4 again
>   best = 4
> 
> r=6, c='A'
>   cnt[A]++ → cnt={A:2,B:3}, maxF = 3
>   winLen = 5, 5-3=2 > 1 → shrink
>     drop s[l=2]='B': cnt={A:2,B:2}, l=3
>     winLen=4
>   best = 4
> 
> ✅ Answer: 4
> ```

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

> [!info]- 🔍 Dry Run: s1="ab", s2="eidbaooo"
> ```text
> Setup:
>   target counts t = {a:1, b:1}     (from s1)
>   Initial window s2[0:2] = "ei"
>   window counts w = {e:1, i:1}
> 
> ─────────────────────────────────────────
> Window 0: "ei", w={e:1,i:1}
>   w == t?  → NO (different chars)
> 
> Slide to window 1: add s2[2]='d', drop s2[0]='e'
>   w[d]++, w[e]--
>   w = {i:1, d:1}        ← e dropped to 0
>   "id", w == t? NO
> 
> Slide to window 2: add 'b', drop 'i'
>   w = {d:1, b:1}
>   "db", w == t? NO
> 
> Slide to window 3: add 'a', drop 'd'
>   w = {b:1, a:1}
>   "ba", w == t={a:1,b:1}?  → YES!
> 
> ✅ Answer: true
> ```

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

> [!info]- 🔍 Dry Run: s="cbaebabacd", p="abc"
> ```text
> Setup:
>   t = Counter("abc") = {a:1, b:1, c:1}
>   Initial window s[0:3] = "cba", w = {c:1, b:1, a:1}
> 
> ─────────────────────────────────────────
> Window start 0: "cba"  w == t? YES → out.append(0)
> 
> Slide → add s[3]='e', drop s[0]='c'
>   w = {b:1, a:1, e:1}
> Window start 1: "bae"  w == t? NO
> 
> Slide → add 'b', drop 'b'
>   w = {a:1, e:1, b:1}                ← b dropped then added; net unchanged
>   wait: drop b first: w={a:1,e:1}; add b: w={a:1,e:1,b:1}
> Window start 2: "aeb"  w == t? NO (has e instead of c)
> 
> Slide → add 'a', drop 'a'
>   w = {e:1, b:1, a:1}
> Window 3: "eba"  NO
> 
> Slide → add 'b', drop 'e'
>   w = {b:2, a:1}
> Window 4: "bab"  NO
> 
> Slide → add 'a', drop 'b'
>   w = {b:1, a:2}
> Window 5: "aba"  NO
> 
> Slide → add 'c', drop 'a'
>   w = {b:1, a:1, c:1}
> Window 6: "bac"  w == t? YES → out.append(6)
> 
> Slide → add 'd', drop 'b'
>   w = {a:1, c:1, d:1}
> Window 7: "acd"  NO
> 
> ✅ Answer: [0, 6]
> ```

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

**LC #76** · **Hard**

Smallest substring of `s` containing every char of `t` (with multiplicity).

**Edge cases:** `len(t) > len(s)` (return "") · `t` has duplicates · all chars equal.

### 🧠 Pattern: Variable Window + Need Counter

> Track how many distinct chars still need their full count. Expand `r` until `need == 0`; then shrink `l` while still valid, recording min window.

> [!info]- 🔍 Dry Run: s="ADOBECODEBANC", t="ABC"
> ```text
> Setup:
>   need = {A:1, B:1, C:1}       ← positive entries = still need
>   missing = 3                  ← count of chars still needed
>   l = 0, best_len = ∞
> 
> ─────────────────────────────────────────
> r=0, c='A'
>   need[A] > 0 → missing-- → missing=2
>   need[A]-- → need = {A:0, B:1, C:1}
>   missing=2 ≠ 0 → no shrink
> 
> r=1, c='D'
>   need[D] > 0? NO (default 0)
>   need[D]-- → need[D]=-1
> 
> r=2, c='O'  similar → need[O]=-1
> 
> r=3, c='B'
>   need[B] > 0 → missing=1
>   need[B]=0
> 
> r=4, c='E'  → need[E]=-1
> 
> r=5, c='C'
>   need[C] > 0 → missing=0   ✓ found valid window!
>   need[C]=0
>   Now SHRINK while still valid:
>     window=[0..5]="ADOBEC" len=6, update best (best=6, best_l=0)
>     drop s[l=0]='A': need[A]++ → need[A]=1 > 0 → missing=1, STOP shrinking
>     l=1
> 
> r=6, c='O'  → need[O]=-2
> 
> r=7, c='D'  → need[D]=-2
> 
> r=8, c='E'  → need[E]=-2
> 
> r=9, c='B'
>   need[B] > 0? need[B]=0, NO
>   need[B]=-1
> 
> r=10, c='A'
>   need[A] > 0 → missing=0    ✓ valid again
>   need[A]=0
>   SHRINK while valid:
>     window=[1..10]="DOBECODEBA" len=10, not better than 6
>     drop s[1]='D': need[D]=-1, still ≤ 0 → still valid, l=2
>     drop 'O': need[O]=-1, valid, l=3
>     drop 'B': need[B]=0, still valid... wait, need[B] was -1, ++ → 0, not > 0, valid. l=4
>     drop 'E': need[E]=-1, valid, l=5
>     drop 'C': need[C]=1 > 0 → missing=1, STOP. l=6.
>   But wait — we should have recorded the smaller window before stopping.
>   At each shrink iteration before exiting: window len = r-l+1 = 10-5+1=6 — equal best.
>   (Actually it's 6 too; equal isn't better, keep best=6.)
> 
> r=11, c='N' → need[N]=-1
> 
> r=12, c='C'
>   need[C] > 0 → missing=0
>   need[C]=0
>   SHRINK:
>     window=[6..12]="ODEBANC" len=7, not better
>     drop 'O': l=7. valid. len=6.
>     drop 'D': l=8. valid. len=5.   ← better! best_len=5, best_l=8
>     drop 'E': l=9. valid. len=4.   ← better! best_len=4, best_l=9
>     drop 'B': need[B]=1 > 0 → missing=1, stop. l=10. window=[9..12]="BANC"
> 
> ✅ Answer: s[9:13] = "BANC"
> ```

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

**LC #239** · **Hard**

Max in each window of size `k`.

**Edge cases:** k = 1 · k = n · all equal.

### 🧠 Pattern: Monotonic Deque (Decreasing)

> Maintain a deque of **indices** whose values are decreasing. Head is always the max of current window. Push `r`: pop tail while smaller. Drop head if out of window.

> [!info]- 🔍 Dry Run: nums=[1,3,-1,-3,5,3,6,7], k=3
> ```text
> Setup:
>   dq = [] (deque of indices, values decreasing front→back)
>   out = []
> 
> ─────────────────────────────────────────
> r=0, x=1
>   dq empty → push 0
>   dq=[0]
>   r >= k-1=2? NO, don't output
> 
> r=1, x=3
>   nums[dq.back()=0]=1 < 3 → pop 0
>   dq=[] → push 1
>   dq=[1]
>   r >= 2? NO
> 
> r=2, x=-1
>   nums[dq.back()=1]=3 < -1? NO → don't pop
>   push 2 → dq=[1,2]
>   r==2, window full → out.append(nums[dq.front()=1]) = nums[1]=3
>   out=[3]
> 
> r=3, x=-3
>   nums[2]=-1 < -3? NO → don't pop
>   push 3 → dq=[1,2,3]
>   front=1, in window [r-k+1=1, r=3]? 1>=1 YES → keep
>   out.append(nums[1]) = 3
>   out=[3,3]
> 
> r=4, x=5
>   nums[dq.back()=3]=-3 < 5 → pop 3
>   nums[2]=-1 < 5 → pop 2
>   nums[1]=3 < 5 → pop 1
>   dq=[] → push 4
>   dq=[4]
>   r=4 in window [2,4], front=4 in range
>   out.append(nums[4]=5)
>   out=[3,3,5]
> 
> r=5, x=3
>   nums[4]=5 < 3? NO
>   push 5 → dq=[4,5]
>   front=4, window [3,5], in range
>   out.append(5) → out=[3,3,5,5]
> 
> r=6, x=6
>   nums[5]=3 < 6 → pop 5
>   nums[4]=5 < 6 → pop 4
>   push 6 → dq=[6]
>   out.append(6) → out=[3,3,5,5,6]
> 
> r=7, x=7
>   nums[6]=6 < 7 → pop 6
>   push 7 → dq=[7]
>   out.append(7) → out=[3,3,5,5,6,7]
> 
> ✅ Answer: [3, 3, 5, 5, 6, 7]
> ```

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

> [!info]- 🔍 Dry Run: nums=[1,12,-5,-6,50,3], k=4
> ```text
> Setup:
>   sum of first k=4: 1+12+(-5)+(-6) = 2
>   best = 2
> 
> ─────────────────────────────────────────
> i=4, slide window:
>   sum += nums[4] - nums[4-4] = 50 - 1 = 49
>   sum = 2 + 49 = 51
>   best = max(2, 51) = 51
> 
> i=5:
>   sum += nums[5] - nums[1] = 3 - 12 = -9
>   sum = 51 + (-9) = 42
>   best = 51 (unchanged)
> 
> ✅ Answer: 51 / 4 = 12.75
> ```

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

**LC #992** · **Hard**

Count subarrays containing **exactly** k distinct ints.

### 🧠 Pattern: "Exactly K" = "At most K" − "At most K−1"

> Counting subarrays with **exactly** k distinct is hard directly. Define `atMost(k)` (variable window, O(n)). Then `exactly(k) = atMost(k) - atMost(k-1)`.

> [!info]- 🔍 Dry Run: nums=[1,2,1,2,3], k=2
> ```text
> exactly(2) = atMost(2) - atMost(1)
> 
> ─────────────────────────────────────────
> Compute atMost(2):
>   cnt={}, l=0, total=0
> 
>   r=0 x=1: cnt={1:1}, len=1 ≤ 2 valid. total += r-l+1 = 1 → total=1
>   r=1 x=2: cnt={1:1,2:1}, len=2 ≤ 2 valid. total += 2 → total=3
>   r=2 x=1: cnt={1:2,2:1}, len=2 valid. total += 3 → total=6
>   r=3 x=2: cnt={1:2,2:2}, len=2 valid. total += 4 → total=10
>   r=4 x=3: cnt={1:2,2:2,3:1}, len=3 INVALID → shrink
>     drop nums[0]=1: cnt={1:1,2:2,3:1}, l=1; len=3 still
>     drop nums[1]=2: cnt={1:1,2:1,3:1}, l=2; len=3 still
>     drop nums[2]=1: cnt={2:1,3:1}, l=3; len=2 valid
>   total += r-l+1 = 4-3+1=2 → total=12
> 
>   atMost(2) = 12
> 
> ─────────────────────────────────────────
> Compute atMost(1):
>   r=0 x=1: cnt={1:1} len=1 valid. total=1
>   r=1 x=2: cnt={1:1,2:1} INVALID. shrink:
>     drop 1: cnt={2:1}, l=1, valid
>   total += 1 → total=2
>   r=2 x=1: cnt={1:1,2:1} INVALID. shrink:
>     drop 2: cnt={1:1}, l=2, valid
>   total += 1 → total=3
>   r=3 x=2: INVALID. shrink: drop 1, l=3. valid. total +=1 → total=4
>   r=4 x=3: INVALID. shrink: drop 2, l=4. valid. total +=1 → total=5
> 
>   atMost(1) = 5
> 
> ─────────────────────────────────────────
> exactly(2) = 12 - 5 = 7
> 
> ✅ Answer: 7
>   (subarrays: [1,2], [2,1], [1,2], [2,3], [1,2,1], [2,1,2], [1,2,1,2])
> ```

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

> [!info]- 🔍 Dry Run: fruits=[1,2,1,2,3,2,2]
> ```text
> Setup:
>   cnt={}, l=0, best=0
> 
> ─────────────────────────────────────────
> r=0 f=1: cnt={1:1}, 1 distinct, valid. best=max(0, 1)=1
> r=1 f=2: cnt={1:1,2:1}, valid. best=2
> r=2 f=1: cnt={1:2,2:1}, valid. best=3
> r=3 f=2: cnt={1:2,2:2}, valid. best=4
> r=4 f=3: cnt={1:2,2:2,3:1}, 3 distinct > 2 → shrink
>   drop fruits[0]=1: cnt={1:1,2:2,3:1}, l=1
>   drop fruits[1]=2: cnt={1:1,2:1,3:1}, l=2
>   drop fruits[2]=1: cnt={2:1,3:1}, l=3
>   2 distinct valid
>   window len = r-l+1 = 4-3+1=2; best=4 (unchanged)
> r=5 f=2: cnt={2:2,3:1}, valid. len=3. best=4
> r=6 f=2: cnt={2:3,3:1}, valid. len=4. best=4
> 
> ✅ Answer: 4   (subarray [1,2,1,2] or [2,3,2,2] — there are multiple of len 4)
> ```

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

> [!info]- 🔍 Dry Run: nums=[1,1,1,0,0,0,1,1,1,1,0], k=2
> ```text
> Setup:
>   l=0, zeros=0, best=0
> 
> ─────────────────────────────────────────
> r=0 x=1: zeros=0, valid, len=1, best=1
> r=1 x=1: zeros=0, valid, len=2, best=2
> r=2 x=1: zeros=0, valid, len=3, best=3
> r=3 x=0: zeros=1, valid (≤2), len=4, best=4
> r=4 x=0: zeros=2, valid, len=5, best=5
> r=5 x=0: zeros=3 > k=2 → INVALID → shrink
>   nums[l=0]=1, not zero → l=1
>   nums[1]=1 → l=2
>   nums[2]=1 → l=3
>   nums[3]=0, zero → zeros=2, l=4. now valid (zeros=2 ≤ 2)
>   len = r-l+1 = 5-4+1=2; best=5 (unchanged)
> r=6 x=1: zeros=2, valid, len=3, best=5
> r=7 x=1: len=4, best=5
> r=8 x=1: len=5, best=5
> r=9 x=1: len=6, best=6   ✓ updated
> r=10 x=0: zeros=3, INVALID → shrink
>   nums[4]=0, zero → zeros=2, l=5. valid.
>   len = 10-5+1=6; best=6 (no change)
> 
> ✅ Answer: 6
> ```

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

> [!info]- 🔍 Dry Run: nums=[1,1,2,1,1], k=3
> ```text
> exactly(3) = atMost(3) - atMost(2)
> 
> atMost(3):  count subarrays with ≤ 3 odd numbers
>   nums=[1,1,2,1,1], all combinations are ≤ 3 odd? Let me check.
>   The whole array has 4 odds: 1,1,1,1. So [0..4] is invalid.
> 
>   r=0 x=1 odds=1, valid. total += 1 = 1
>   r=1 x=1 odds=2, valid. total += 2 = 3
>   r=2 x=2 odds=2, valid. total += 3 = 6
>   r=3 x=1 odds=3, valid. total += 4 = 10
>   r=4 x=1 odds=4 > 3, INVALID → shrink
>     drop nums[0]=1: odds=3, l=1, valid
>   total += r-l+1 = 4-1+1=4 → total=14
>   atMost(3) = 14
> 
> atMost(2):
>   r=0 x=1 odds=1, valid. total=1
>   r=1 x=1 odds=2, valid. total=3
>   r=2 x=2 odds=2, valid. total=6
>   r=3 x=1 odds=3, INVALID → drop nums[0]=1, odds=2, l=1. valid.
>     total += 3 = 9
>   r=4 x=1 odds=3, INVALID → drop nums[1]=1, odds=2, l=2. valid.
>     total += 3 = 12
>   atMost(2) = 12
> 
> ─────────────────────────────────────────
> Answer: 14 - 12 = 2
> 
> ✅ Answer: 2
>   (subarrays with exactly 3 odds: [1,1,2,1] and [1,2,1,1])
> ```

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
