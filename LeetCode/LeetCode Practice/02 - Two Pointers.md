---
title: "LeetCode Practice: Two Pointers"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - two-pointers
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Two Pointers

12 problems · 3 sub-patterns: **opposite-direction**, **same-direction**, **fast/slow**.

> [!abstract] Pattern recap
> **Opposite** — start at `l=0, r=n-1`, move inward. For sorted arrays / palindromes / sum-target / container.
> **Same direction** — `slow` and `fast` move forward; `slow` writes, `fast` reads. For in-place filtering.
> **Fast/Slow (Floyd)** — fast moves 2 steps per slow 1. For cycle detection / midpoint / nth-from-end.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Valid Palindrome | 125 | Easy | Opposite |
| P2 | Valid Palindrome II | 680 | Easy | Opposite + try-skip |
| P3 | Two Sum II — Sorted Input | 167 | Med | Opposite |
| P4 | 3Sum | 15 | Med | Fix one + opposite |
| P5 | 3Sum Closest | 16 | Med | Fix one + opposite |
| P6 | Container With Most Water | 11 | Med | Opposite (move shorter) |
| P7 | Trapping Rain Water | 42 | **Hard** | Opposite + max-so-far |
| P8 | Remove Duplicates from Sorted Array | 26 | Easy | Same direction |
| P9 | Move Zeroes | 283 | Easy | Same direction (write-pointer) |
| P10 | Sort Colors | 75 | Med | 3 pointers (Dutch flag) |
| P11 | Linked List Cycle | 141 | Easy | Fast/Slow |
| P12 | Middle of the Linked List | 876 | Easy | Fast/Slow |

---

## P1: Valid Palindrome

**LC #125** · Easy

Ignore non-alphanumerics and case; is it a palindrome?

**Edge cases:** empty string (true) · all non-alphanumerics (true) · mixed case · single char.

### 🧠 Pattern: Opposite Pointers (Palindrome Skeleton)

> Walk from both ends inward. Skip filler chars; compare. Bail on mismatch.

### Approach Evolution

1. **Clean string + reverse compare** · O(n)/O(n). Easy but allocates.
2. **Two pointers in place — FINAL** · O(n)/O(1).

> [!example]- 📊 Visual: opposite pointers converging
> ```text
>   s = "A man, a plan, a canal: Panama"
> 
>   Logical view (after lowercasing & ignoring non-alphanumerics):
>      "amanaplanacanalpanama"
>       0 1 2 3 4 5 6 7 8 9 ...
> 
>   Pointers walk INWARD and compare at each step:
> 
>         ┌─────────────────────────────────────────┐
>         │ a m a n a p l a n a c a n a l p a n a m a │
>         └─────────────────────────────────────────┘
>           ↑                                       ↑
>           l                                       r
>           a ──────────── compare ─────────────── a   ✓
> 
>     after a few steps:
>                     ┌─────────────┐
>           ... ... a │ n a l p a n │ a m a
>                     └─────────────┘
>                     ↑           ↑
>                     l           r
> 
>     when l >= r:                  end. No mismatch → palindrome.
> 
>   On a real string, filler chars are SKIPPED, not compared:
> 
>      "A   m a n ,   a   p l a n ..."
>       ↑   ↑  on space/comma we advance the pointer past them
>       l→
> 
>   Cost: each char visited at most once by l or by r → O(n), O(1) space.
> ```

> [!info]- 🔍 Dry Run: s="A man, a plan, a canal: Panama"
> ```text
> Setup:
>   l = 0, r = 29   (length 30)
> 
> ─────────────────────────────────────────
> Step 1: l=0 'A', r=29 'a'
>   both alphanumeric, lowercase comparison
>   'a' == 'a'  → match
>   l=1, r=28
> 
> Step 2: l=1 ' ', r=28 'm'
>   s[l]=' ' is NOT alphanumeric → skip, l++
>   l=2 'm', r=28 'm'
>   match. l=3, r=27
> 
> Step 3: l=3 'a', r=27 'a'  (compressing the trace — same idea)
>   match. l=4, r=26
> 
> Step 4: l=4 'n', r=26 'n'
>   match. l=5, r=25
> 
> ... continue through ',', ' ', 'a', 'p', 'l', 'a', 'n', ...
> Each non-alphanumeric is skipped; each pair of letters matches.
> 
> Eventually l >= r → loop exits with no mismatch.
> 
> ✅ Answer: true
> ```

> [!success]- JS
> ```js
> const isPalindrome = (s) => {
>   const ok = (c) => /[a-z0-9]/i.test(c);
>   let l = 0, r = s.length - 1;
>   while (l < r) {
>     while (l < r && !ok(s[l])) l++;
>     while (l < r && !ok(s[r])) r--;
>     if (s[l].toLowerCase() !== s[r].toLowerCase()) return false;
>     l++; r--;
>   }
>   return true;
> };
> ```

> [!success]- Python
> ```python
> def is_palindrome(s):
>     l, r = 0, len(s) - 1
>     while l < r:
>         while l < r and not s[l].isalnum(): l += 1
>         while l < r and not s[r].isalnum(): r -= 1
>         if s[l].lower() != s[r].lower(): return False
>         l += 1; r -= 1
>     return True
> ```

**Variants:** P2 (allow one skip) · Longest Palindromic Substring (expand around center).

**Key takeaway:** Palindrome → two pointers from ends inward. O(1) space wins over clean-and-reverse.

---

## P2: Valid Palindrome II

**LC #680** · Easy

Palindrome if you may **delete at most one** char.

**Edge cases:** already palindrome · two consecutive mismatches in middle · empty.

### 🧠 Pattern: Opposite Pointers + Try-Skip

> Walk inward. On the first mismatch, try skipping `l` OR skipping `r` — if either remaining slice is a palindrome, return true. The "skip" branches into two O(n) checks, total still O(n).

### Approach Evolution

1. **Try every deletion** · O(n²). TLE.
2. **One mismatch → two slice checks — FINAL** · O(n)/O(1).

> [!example]- 📊 Visual: branch on first mismatch
> ```text
>   s = "a b c a"
>        0 1 2 3
> 
>   Walk inward as in P1:
> 
>      [ a , b , c , a ]
>        ↑           ↑
>        l           r        a == a ✓ → l++, r--
> 
>          [ b , c ]
>            ↑   ↑
>            l   r            b vs c  ✗ MISMATCH
> 
>   With one allowed deletion, branch into TWO checks:
> 
>     ┌────────────────────┐         ┌────────────────────┐
>     │ Option A: skip l   │         │ Option B: skip r   │
>     │   check s[l+1..r]  │         │   check s[l..r-1]  │
>     │     = "c"          │         │     = "b"          │
>     │   palindrome? YES  │         │   palindrome? YES  │
>     └────────────────────┘         └────────────────────┘
>                         ↓                       ↓
>                          ───── OR ─────
>                                 ↓
>                              return TRUE
> 
>   Each helper is O(n); we run at most 2 of them → O(n) total.
> 
>   Counter-example:  s = "abc"
>     mismatch at (l=0,r=2)
>       Option A: is_pal("bc")  → FALSE
>       Option B: is_pal("ab")  → FALSE
>     return FALSE
> ```

> [!info]- 🔍 Dry Run: s="abca"
> ```text
> Setup:
>   l=0, r=3
> 
> ─────────────────────────────────────────
> Step 1: l=0 'a', r=3 'a'
>   match. l=1, r=2
> 
> Step 2: l=1 'b', r=2 'c'
>   MISMATCH! Try two options:
> 
>   Option A: skip s[l] → check is_pal("ca") = is_pal(s[2..2])
>     l'=2, r'=2 → loop exits immediately (l>=r) → palindrome
>     return true via Option A
> 
>   (We don't even need Option B in this case.)
> 
> ✅ Answer: true
> 
> ─────────────────────────────────────────
> Counter-example: s="abc"
>   l=0 'a', r=2 'c' MISMATCH
>     Option A: is_pal(s[1..2]) = "bc"  → false
>     Option B: is_pal(s[0..1]) = "ab"  → false
>   Return false.
> ```

> [!success]- JS
> ```js
> const validPalindrome = (s) => {
>   const isPal = (l, r) => {
>     while (l < r) {
>       if (s[l] !== s[r]) return false;
>       l++; r--;
>     }
>     return true;
>   };
>   let l = 0, r = s.length - 1;
>   while (l < r) {
>     if (s[l] !== s[r]) return isPal(l + 1, r) || isPal(l, r - 1);
>     l++; r--;
>   }
>   return true;
> };
> ```

> [!success]- Python
> ```python
> def valid_palindrome(s):
>     def is_pal(l, r):
>         while l < r:
>             if s[l] != s[r]: return False
>             l += 1; r -= 1
>         return True
>     l, r = 0, len(s) - 1
>     while l < r:
>         if s[l] != s[r]:
>             return is_pal(l + 1, r) or is_pal(l, r - 1)
>         l += 1; r -= 1
>     return True
> ```

**Variants:** Valid Palindrome III (delete up to k → DP).

**Key takeaway:** One allowed mismatch → branch into two helper checks. Keep O(n).

---

## P3: Two Sum II — Input Array Is Sorted

**LC #167** · Medium · **1-indexed output, O(1) extra**

Return 1-indexed positions of two numbers summing to target.

**Edge cases:** negatives · duplicate target value · n = 2.

### 🧠 Pattern: Opposite Pointers on Sorted Array

> Sorted + pair sum = the canonical opposite-pointer setup. Sum too small → move `l` right; too big → move `r` left.

### Approach Evolution

1. **Hash map (Two Sum P1)** · O(n)/O(n) — works but ignores the sort.
2. **Binary search for complement** · O(n log n)/O(1) — overkill.
3. **Opposite pointers — FINAL** · O(n)/O(1).

> [!example]- 📊 Visual: sorted array + opposite pointers
> ```text
>   Sorted nums:    ┌────┬────┬────┬────┐    target = 9
>                   │  2 │  7 │ 11 │ 15 │
>                   └────┴────┴────┴────┘
>                     ↑              ↑
>                     l              r
> 
>   Number line (sorted = monotonic):
> 
>      small ─── 2 ── 7 ── 11 ── 15 ─── large
>                ↑                ↑
>                l                r
> 
>   Decision tree per step (sum = nums[l] + nums[r]):
> 
>             ┌──────────────────────────────┐
>     sum  >  │  too big  → r--   (shrink high)│
>             │                              │
>     sum  <  │  too small → l++  (grow low)  │
>             │                              │
>     sum  =  │  HIT → return [l+1, r+1]      │
>             └──────────────────────────────┘
> 
>   Trace:  l=0 r=3  sum=2+15=17 > 9 → r--
>           l=0 r=2  sum=2+11=13 > 9 → r--
>           l=0 r=1  sum=2+ 7 = 9 ✓ → return [1, 2]
> 
>   Why no missed pair? On each step we discard EXACTLY ONE row/column of
>   the implicit n×n sum matrix — but never the row/column containing the
>   answer. Each pointer moves at most n steps → O(n).
> ```

> [!info]- 🔍 Dry Run: nums=[2,7,11,15], target=9
> ```text
> Setup:
>   l = 0, r = 3
> 
> ─────────────────────────────────────────
> Step 1: l=0 (nums[0]=2), r=3 (nums[3]=15)
>   sum = 2 + 15 = 17
>   17 > target=9? YES → too big, decrement r
>   l=0, r=2
> 
> Step 2: l=0 (2), r=2 (nums[2]=11)
>   sum = 2 + 11 = 13
>   13 > 9? YES → r--
>   l=0, r=1
> 
> Step 3: l=0 (2), r=1 (nums[1]=7)
>   sum = 2 + 7 = 9
>   sum == target → MATCH
>   return [l+1, r+1] = [1, 2]      ← 1-indexed!
> 
> ✅ Answer: [1, 2]
> ```

> [!success]- JS
> ```js
> const twoSumSorted = (nums, target) => {
>   let l = 0, r = nums.length - 1;
>   while (l < r) {
>     const s = nums[l] + nums[r];
>     if (s === target) return [l + 1, r + 1];
>     if (s < target) l++; else r--;
>   }
> };
> ```

> [!success]- Python
> ```python
> def two_sum_sorted(nums, target):
>     l, r = 0, len(nums) - 1
>     while l < r:
>         s = nums[l] + nums[r]
>         if s == target: return [l + 1, r + 1]
>         if s < target: l += 1
>         else: r -= 1
> ```

**Variants:** 3Sum (P4) · 4Sum · Two Sum BST.

**Key takeaway:** Sorted array + pair → opposite pointers. O(1) space beats hash.

---

## P4: 3Sum

**LC #15** · Medium

Find all unique triplets summing to 0.

**Edge cases:** duplicates (skip them!) · all zeros · n < 3.

### 🧠 Pattern: Fix-One + Two-Pointer-on-Rest

> Sort first. For each `i`, run 2Sum (with target `-nums[i]`) on `nums[i+1..n-1]` using opposite pointers. **Skip duplicates** at every level to avoid duplicate triplets.

### Approach Evolution

1. **3 nested loops + dedup set** · O(n³). Slow.
2. **For each pair, hash for 3rd** · O(n²)/O(n). Works.
3. **Sort + two pointers — FINAL** · O(n²)/O(1).

> [!example]- 📊 Visual: fix one, 2-sum the rest
> ```text
>   nums (sorted):  [-4, -1, -1, 0, 1, 2]
>                     0   1   2  3  4  5
> 
>   Outer loop fixes i. Inner two-pointer scans nums[i+1 .. n-1]:
> 
>          ┌────┬────┬────┬───┬───┬───┐
>     i=1: │ -4 │░-1░│ -1 │ 0 │ 1 │ 2 │      target for inner = -nums[i] = 1
>          └────┴────┴────┴───┴───┴───┘
>                 ▲    ↑           ↑
>                fix   l           r
> 
>     l=2 r=5  -1+2 = 1 ✓ MATCH → push [-1,-1,2]
>              skip dups on both sides, then l++, r--
>     l=3 r=4   0+1 = 1 ✓ MATCH → push [-1, 0,1]
>              l++ r-- → cross → stop inner
> 
>   Dedup discipline (CRUCIAL):
>     • outer: if nums[i] == nums[i-1] → continue
>     • on match: while nums[l]==nums[l+1] → l++; while nums[r]==nums[r-1] → r--
> 
>   Why sort?
>     • makes two-pointer work (monotonic sums)
>     • makes dedup easy (duplicates are adjacent)
> 
>   Total work:  O(n log n) sort  +  O(n) inner × n outer  =  O(n²).
> ```

> [!info]- 🔍 Dry Run: nums=[-1,0,1,2,-1,-4]
> ```text
> Phase 1 — Sort:
>   nums sorted = [-4, -1, -1, 0, 1, 2]
> 
> ─────────────────────────────────────────
> i=0, nums[i]=-4
>   target for inner pair = -nums[i] = 4
>   l=1, r=5
>   sum = -1 + 2 = 1 < 4 → l++
>   l=2 r=5: -1+2 = 1 < 4 → l++
>   l=3 r=5:  0+2 = 2 < 4 → l++
>   l=4 r=5:  1+2 = 3 < 4 → l++
>   l=5 r=5: stop
>   No triplet here.
> 
> ─────────────────────────────────────────
> i=1, nums[i]=-1
>   target = 1
>   l=2, r=5: -1+2 = 1 ✓ MATCH
>     append [-1, -1, 2]
>     skip dups: nums[l]=-1, nums[l+1]=0 → no dup; l++. nums[r]=2, nums[r-1]=1 → no dup; r--
>     l=3, r=4: 0+1 = 1 ✓ MATCH
>       append [-1, 0, 1]
>       l++, r-- → l=4, r=3 → exit
> 
> ─────────────────────────────────────────
> i=2, nums[i]=-1
>   SKIP: nums[i] == nums[i-1]  (same outer value → would produce dup triplets)
> 
> ─────────────────────────────────────────
> i=3, nums[i]=0
>   target = 0
>   l=4, r=5: 1+2 = 3 > 0 → r--
>   l=4, r=4: stop
>   No triplet.
> 
> ─────────────────────────────────────────
> i=4: only 1 element after — stop outer loop (need ≥ 2)
> 
> ✅ Answer: [[-1, -1, 2], [-1, 0, 1]]
> ```

> [!success]- JS
> ```js
> const threeSum = (nums) => {
>   nums.sort((a, b) => a - b);
>   const out = [];
>   for (let i = 0; i < nums.length - 2; i++) {
>     if (i > 0 && nums[i] === nums[i - 1]) continue;
>     let l = i + 1, r = nums.length - 1;
>     while (l < r) {
>       const s = nums[i] + nums[l] + nums[r];
>       if (s === 0) {
>         out.push([nums[i], nums[l], nums[r]]);
>         while (l < r && nums[l] === nums[l + 1]) l++;
>         while (l < r && nums[r] === nums[r - 1]) r--;
>         l++; r--;
>       } else if (s < 0) l++;
>       else r--;
>     }
>   }
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def three_sum(nums):
>     nums.sort()
>     out = []
>     for i in range(len(nums) - 2):
>         if i > 0 and nums[i] == nums[i - 1]: continue
>         l, r = i + 1, len(nums) - 1
>         while l < r:
>             s = nums[i] + nums[l] + nums[r]
>             if s == 0:
>                 out.append([nums[i], nums[l], nums[r]])
>                 while l < r and nums[l] == nums[l + 1]: l += 1
>                 while l < r and nums[r] == nums[r - 1]: r -= 1
>                 l += 1; r -= 1
>             elif s < 0: l += 1
>             else: r -= 1
>     return out
> ```

**Variants:** 3Sum Closest (P5) · 4Sum (fix two, then 2-pointer) · 3Sum Smaller (count).

**Key takeaway:** k-Sum reduces to (k-1)-Sum recursively. 3Sum = 2-pointer inside outer loop. Skip dups aggressively.

---

## P5: 3Sum Closest

**LC #16** · Medium

Return sum of triplet closest to `target`.

**Edge cases:** n = 3 (only one answer) · target very far from all sums.

### 🧠 Pattern: Same skeleton as 3Sum, track best diff.

### Approach (one-liner)

Sort. For each `i`, two-pointer on rest. Update `best` if `|target - s| < |target - best|`. Move pointer based on `s < target`.

> [!example]- 📊 Visual: minimize |target − sum| on a number line
> ```text
>   target = 1.  We're hunting on the number line for the closest triplet sum.
> 
>      ─────────────────────●─────────────────────  number line
>                         target=1
> 
>      candidate sums (from various triplets):
> 
>      ── -4 ─── -3 ─── -1 ── 0 ── 1 ── 2 ── 3 ──── ...
>            ●     ●      ●         ●    ●
>            sum=-4 sum=-3 sum=-1   ?    sum=2     ← distance to target
>            dist=5 dist=4 dist=2        dist=1   ★ best
> 
>   Update rule:
>      best ← s    iff   |target − s|  <  |target − best|
> 
>   Pointer movement (same as 3Sum, no equality stop):
>      if s < target:  l++   (try larger)
>      if s > target:  r--   (try smaller)
>      if s = target:  return immediately — can't beat 0 distance
> 
>   Visualisation of one outer step:
> 
>      sorted: [-4, -1, 1, 2]      i=0 (nums[i]=-4)
>               i  l        r
> 
>        l=1 r=3:  -4 + (-1) + 2 = -3       update best
>        l=2 r=3:  -4 +   1  + 2 = -1       closer, update best
>        l=3 r=3:  stop
> 
>      i=1 (nums[i]=-1)
>        l=2 r=3:  -1 + 1 + 2 = 2           |1-2|=1 closer, best=2
>        l=3 r=3:  stop
> 
>      answer = 2.
> ```

> [!info]- 🔍 Dry Run: nums=[-1,2,1,-4], target=1
> ```text
> Phase 1 — Sort:
>   nums = [-4, -1, 1, 2]
>   best = -4 + -1 + 1 = -4 (any initial triplet sum)
> 
> ─────────────────────────────────────────
> i=0, nums[i]=-4
>   l=1, r=3
>   s = -4 + -1 + 2 = -3
>   |1 - (-3)| = 4 vs |1 - (-4)| = 5  → -3 is closer
>   best = -3
>   s=-3 < 1 → l++
> 
>   l=2, r=3
>   s = -4 + 1 + 2 = -1
>   |1 - (-1)| = 2 vs |1 - (-3)| = 4 → -1 closer
>   best = -1
>   s=-1 < 1 → l++
> 
>   l=3, r=3 → stop
> 
> ─────────────────────────────────────────
> i=1, nums[i]=-1
>   l=2, r=3
>   s = -1 + 1 + 2 = 2
>   |1 - 2| = 1 vs |1 - (-1)| = 2  → 2 closer
>   best = 2
>   s=2 > 1 → r--
> 
>   l=2, r=2 → stop
> 
> ─────────────────────────────────────────
> i=2: only 1 element after — stop
> 
> ✅ Answer: 2
> ```

> [!success]- JS
> ```js
> const threeSumClosest = (nums, target) => {
>   nums.sort((a, b) => a - b);
>   let best = nums[0] + nums[1] + nums[2];
>   for (let i = 0; i < nums.length - 2; i++) {
>     let l = i + 1, r = nums.length - 1;
>     while (l < r) {
>       const s = nums[i] + nums[l] + nums[r];
>       if (Math.abs(target - s) < Math.abs(target - best)) best = s;
>       if (s < target) l++;
>       else if (s > target) r--;
>       else return s;
>     }
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def three_sum_closest(nums, target):
>     nums.sort()
>     best = nums[0] + nums[1] + nums[2]
>     for i in range(len(nums) - 2):
>         l, r = i + 1, len(nums) - 1
>         while l < r:
>             s = nums[i] + nums[l] + nums[r]
>             if abs(target - s) < abs(target - best): best = s
>             if s < target: l += 1
>             elif s > target: r -= 1
>             else: return s
>     return best
> ```

**Key takeaway:** Closest-X problems → same scaffold, track best `|target - s|`.

---

## P6: Container With Most Water

**LC #11** · Medium

Heights array; pick two lines, max water = `min(h_l, h_r) * (r - l)`.

**Edge cases:** all same height · n = 2 · zero heights.

### 🧠 Pattern: Opposite Pointers, Move the Shorter Side

> Width is `r - l`; height is `min(h[l], h[r])`. Moving the **taller** side inward can only **decrease** width without increasing the bounding height — guaranteed worse. Always move the shorter side; that's the only way the bounding height might improve.

### Approach Evolution

1. **Brute force pairs** · O(n²). TLE at n=10⁵.
2. **Two pointers (move shorter) — FINAL** · O(n)/O(1).

> [!example]- 📊 Visual: container with bars
> ```text
>   heights = [1, 8, 6, 2, 5, 4, 8, 3, 7]
> 
>   8 │   █           █             
>   7 │   █           █         █   
>   6 │   █  █        █         █   
>   5 │   █  █     █  █         █   
>   4 │   █  █     █  █  █      █   
>   3 │   █  █     █  █  █   █  █   
>   2 │   █  █  █  █  █  █   █  █   
>   1 │ █ █  █  █  █  █  █   █  █   
>     └────────────────────────────
>       0  1  2  3  4  5  6  7  8
> 
>   Two pointers — start at the ends, move the SHORTER inward:
> 
>   Iter 1: l=0 (h=1) and r=8 (h=7)
>           area = min(1,7) × (8-0) = 1 × 8 = 8     (shallow)
>           Move l (shorter) →
> 
>   Iter 2: l=1 (h=8) and r=8 (h=7)
>           area = min(8,7) × (8-1) = 7 × 7 = 49 ✓ ← BEST
> 
>              ┌──── width 7 ────┐
>              │                 │
>           7 ─┤                 ├─ water level = min(8,7) = 7
>              │░░░░░░░░░░░░░░░░░│
>              │░░░░░░░░░░░░░░░░░│
>              █                 █
>              1                 8
>          h=8                h=7
> 
>   Key insight: moving the TALLER side can never gain area —
>   width shrinks AND height is still bounded by the same shorter wall.
>   Only the shorter side might be replaced by something taller.
> ```

> [!info]- 🔍 Dry Run: h=[1,8,6,2,5,4,8,3,7]
> ```text
> Setup:
>   l=0, r=8, best=0
> 
> ─────────────────────────────────────────
> Step 1: l=0 (h=1), r=8 (h=7)
>   area = min(1,7) * (8-0) = 1 * 8 = 8
>   best = max(0, 8) = 8
>   move SHORTER side: h[l]=1 < h[r]=7 → l++
>   l=1, r=8
> 
> Step 2: l=1 (h=8), r=8 (h=7)
>   area = min(8,7) * 7 = 7 * 7 = 49
>   best = max(8, 49) = 49 ✓
>   move SHORTER side: h[r]=7 < h[l]=8 → r--
>   l=1, r=7
> 
> Step 3: l=1 (8), r=7 (h=3)
>   area = min(8,3) * 6 = 3 * 6 = 18
>   best = 49 (unchanged)
>   h[r]=3 shorter → r--
>   l=1, r=6
> 
> Step 4: l=1 (8), r=6 (h=8)
>   area = min(8,8) * 5 = 8 * 5 = 40
>   best = 49 (unchanged)
>   tie → move either; say l++
>   l=2, r=6
> 
> Step 5: l=2 (h=6), r=6 (h=8)
>   area = min(6,8) * 4 = 6 * 4 = 24
>   best = 49
>   h[l]=6 shorter → l++
>   l=3, r=6
> 
> Step 6: l=3 (h=2), r=6 (h=8)
>   area = 2 * 3 = 6
>   best = 49
>   l++
>   l=4, r=6
> 
> Step 7: l=4 (h=5), r=6 (h=8)
>   area = 5 * 2 = 10
>   best = 49
>   l++
>   l=5, r=6
> 
> Step 8: l=5 (h=4), r=6 (h=8)
>   area = 4 * 1 = 4
>   best = 49
>   l++ → l=r → exit
> 
> ✅ Answer: 49   (lines at index 1 and 8: width 7 × min(8,7))
> ```

> [!success]- JS
> ```js
> const maxArea = (h) => {
>   let l = 0, r = h.length - 1, best = 0;
>   while (l < r) {
>     best = Math.max(best, Math.min(h[l], h[r]) * (r - l));
>     if (h[l] < h[r]) l++; else r--;
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def max_area(h):
>     l, r, best = 0, len(h) - 1, 0
>     while l < r:
>         best = max(best, min(h[l], h[r]) * (r - l))
>         if h[l] < h[r]: l += 1
>         else: r -= 1
>     return best
> ```

**Variants:** Trapping Rain Water (P7) — same idea, different question.

**Key takeaway:** Two-pointer with **asymmetric move rule** based on which side is the binding constraint.

---

## P7: Trapping Rain Water

**LC #42** · **Hard**

Total water trapped between bars of varying heights.

**Edge cases:** non-decreasing (zero water) · single peak · n ≤ 2.

### 🧠 Pattern: Opposite Pointers + Max-So-Far on Each Side

> Water above index `i` = `min(maxLeft[i], maxRight[i]) - h[i]`. Two-pointer trick: track `leftMax`, `rightMax` as you go; whichever side's max is smaller, that side **bounds** water and can be settled now.

### Approach Evolution

1. **Brute force** · O(n²). For each i, scan left/right max.
2. **Precompute leftMax[] and rightMax[]** · O(n)/O(n).
3. **Monotonic stack** · O(n)/O(n). Also works (see Monotonic Stack topic).
4. **Two pointers — FINAL** · O(n)/O(1).

> [!example]- 📊 Visual: trapped water
> ```text
>   heights = [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]
> 
>   Bars + trapped water (~ = water trapped above):
> 
>   3 │                  █                  
>   2 │       █          █  █     █         
>   1 │    █  █~ █~ █~ █ █  █  █~ █  █      
>   0 │ █  █  █  █  █  █ █  █  █  █  █  █   
>     └──────────────────────────────────
>       0  1  2  3  4  5 6  7  8  9  10 11
> 
>   Water at each index = min(left_max, right_max) − height[i]
> 
>   index:    0  1  2  3  4  5  6  7  8  9  10 11
>   l_max:    0  1  1  2  2  2  2  3  3  3  3  3
>   r_max:    3  3  3  3  3  3  3  3  2  2  2  1
>   bound:    0  1  1  2  2  2  2  3  2  2  2  1
>   height:   0  1  0  2  1  0  1  3  2  1  2  1
>   water:    0  0  1  0  1  2  1  0  0  1  0  0  = 6 total
> 
>   Two-pointer trick: whichever side's running max is SMALLER bounds
>   the water on that side — settle it first, move that side inward.
> ```

> [!info]- 🔍 Dry Run: h=[0,1,0,2,1,0,1,3,2,1,2,1]
> ```text
> Setup:
>   l=0, r=11
>   l_max=0, r_max=0
>   water=0
> 
> ─────────────────────────────────────────
> Step 1: h[l]=0, h[r]=1
>   h[l] < h[r] → process LEFT side
>     l_max = max(0, 0) = 0
>     water += l_max - h[l] = 0 - 0 = 0
>     l++ → l=1
> 
> Step 2: h[l]=1, h[r]=1
>   h[l] < h[r]? NO (equal) → process RIGHT side (any tie-breaker works)
>     r_max = max(0, 1) = 1
>     water += r_max - h[r] = 1 - 1 = 0
>     r-- → r=10
> 
> Step 3: h[1]=1, h[10]=2
>   h[l] < h[r] → LEFT
>     l_max = max(0, 1) = 1
>     water += 1 - 1 = 0
>     l++ → l=2
> 
> Step 4: h[2]=0, h[10]=2
>   LEFT
>     l_max = max(1, 0) = 1
>     water += 1 - 0 = 1                        ← TRAP at index 2: 1 unit
>     l++ → l=3
> 
> Step 5: h[3]=2, h[10]=2
>   tie → RIGHT
>     r_max = max(1, 2) = 2
>     water += 2 - 2 = 0
>     r-- → r=9
> 
> Step 6: h[3]=2, h[9]=1
>   h[l] >= h[r] → RIGHT
>     r_max = max(2, 1) = 2
>     water += 2 - 1 = 1                        ← TRAP at index 9: 1 unit
>     r-- → r=8
> 
> Step 7: h[3]=2, h[8]=2
>   tie → RIGHT
>     r_max = 2; water += 0
>     r-- → r=7
> 
> Step 8: h[3]=2, h[7]=3
>   LEFT
>     l_max = 2; water += 2 - 2 = 0
>     l++ → l=4
> 
> Step 9: h[4]=1, h[7]=3
>   LEFT
>     l_max = max(2, 1) = 2
>     water += 2 - 1 = 1                         ← TRAP at index 4
>     l++ → l=5
> 
> Step 10: h[5]=0, h[7]=3
>   LEFT
>     l_max = 2; water += 2 - 0 = 2              ← TRAP at index 5
>     l++ → l=6
> 
> Step 11: h[6]=1, h[7]=3
>   LEFT
>     l_max = 2; water += 2 - 1 = 1              ← TRAP at index 6
>     l++ → l=7
> 
> l=r → exit
> 
> Total water: 0+0+0+1+0+1+0+0+1+2+1 = 6
> 
> ✅ Answer: 6
> ```

> [!success]- JS
> ```js
> const trap = (h) => {
>   let l = 0, r = h.length - 1, lMax = 0, rMax = 0, water = 0;
>   while (l < r) {
>     if (h[l] < h[r]) {
>       lMax = Math.max(lMax, h[l]);
>       water += lMax - h[l];
>       l++;
>     } else {
>       rMax = Math.max(rMax, h[r]);
>       water += rMax - h[r];
>       r--;
>     }
>   }
>   return water;
> };
> ```

> [!success]- Python
> ```python
> def trap(h):
>     l, r = 0, len(h) - 1
>     l_max = r_max = water = 0
>     while l < r:
>         if h[l] < h[r]:
>             l_max = max(l_max, h[l])
>             water += l_max - h[l]
>             l += 1
>         else:
>             r_max = max(r_max, h[r])
>             water += r_max - h[r]
>             r -= 1
>     return water
> ```

**Variants:** Trapping Rain Water II (2D, needs heap) · Container With Most Water (P6).

**Key takeaway:** Water-at-i bounded by `min(left_max, right_max)`. Two-pointer settles the smaller side first.

---

## P8: Remove Duplicates from Sorted Array

**LC #26** · Easy · **In place, return new length**

Mutate sorted `nums` so first k positions hold unique values.

**Edge cases:** empty · all duplicates (k=1) · already unique (k=n).

### 🧠 Pattern: Same-Direction (Write Pointer)

> `slow` writes; `fast` reads. Advance `slow` only when you see something new.

### Approach Evolution

1. **Build new array** · O(n)/O(n). Violates in-place.
2. **Write pointer — FINAL** · O(n)/O(1).

> [!example]- 📊 Visual: slow writes, fast reads
> ```text
>   Sorted in:  [1, 1, 2, 3, 3]
>                0  1  2  3  4
> 
>   Two pointers walk the SAME direction:
> 
>      ┌─────────────── reading region ───────────────┐
>      │   ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐          │
>      │   │ 1 │  │ 1 │  │ 2 │  │ 3 │  │ 3 │          │
>      │   └───┘  └───┘  └───┘  └───┘  └───┘          │
>      │     0     1      2      3      4              │
>      └────────────────────────────────────────────────┘
>             ↑                           ↑
>           slow                         fast
>          (next                       (cursor)
>          unique slot)
> 
>   Rule (sorted!): nums[fast] != nums[fast-1]  →  write at slow, slow++
>                   else                          →  skip
> 
>   Evolution:
> 
>     fast=1  same as prev  skip          nums: [1,1,2,3,3]  slow=1
>     fast=2  new (2)       write @1      nums: [1,2,2,3,3]  slow=2
>     fast=3  new (3)       write @2      nums: [1,2,3,3,3]  slow=3
>     fast=4  same as prev  skip          nums: [1,2,3,3,3]  slow=3
> 
>   ┌──── kept ────┐
>   │  [1, 2, 3]   │  garbage [...]   ← slow = 3 = new length
>   └──────────────┘
> 
>   slow always trails fast: slow ≤ fast. We never overwrite unread data.
> ```

> [!info]- 🔍 Dry Run: nums=[1,1,2,3,3]
> ```text
> Setup:
>   slow = 1   (slot to write next unique)
>   (treat nums[0] as already-placed; we compare against nums[slow-1])
> 
> ─────────────────────────────────────────
> fast=1: nums[fast]=1
>   nums[fast]=1 == nums[fast-1]=1?  → YES, duplicate
>   skip
>   State: nums=[1,1,2,3,3], slow=1
> 
> fast=2: nums[fast]=2
>   nums[fast]=2 != nums[fast-1]=1?  → YES, new
>   Action: nums[slow]=2; slow++
>   State: nums=[1,2,2,3,3], slow=2
> 
> fast=3: nums[fast]=3
>   nums[fast]=3 != nums[fast-1]=2?  → YES, new
>   Action: nums[slow]=3; slow++
>   State: nums=[1,2,3,3,3], slow=3
> 
> fast=4: nums[fast]=3
>   nums[fast]=3 == nums[fast-1]=3?  → YES, dup
>   skip
> 
> ✅ Answer: slow = 3
>   nums[0..2] = [1, 2, 3]  (positions 3..4 don't matter per problem)
> ```

> [!success]- JS
> ```js
> const removeDuplicates = (nums) => {
>   if (nums.length === 0) return 0;
>   let slow = 1;
>   for (let fast = 1; fast < nums.length; fast++) {
>     if (nums[fast] !== nums[fast - 1]) nums[slow++] = nums[fast];
>   }
>   return slow;
> };
> ```

> [!success]- Python
> ```python
> def remove_duplicates(nums):
>     if not nums: return 0
>     slow = 1
>     for fast in range(1, len(nums)):
>         if nums[fast] != nums[fast - 1]:
>             nums[slow] = nums[fast]
>             slow += 1
>     return slow
> ```

**Variants:** Remove Duplicates II (allow each value twice) · Remove Element.

**Key takeaway:** In-place filter on sorted → slow/fast pointers, slow as write head.

---

## P9: Move Zeroes

**LC #283** · Easy · **In place, preserve order**

Move all zeroes to end; keep relative order of non-zeroes.

**Edge cases:** all zeros · no zeros · single element.

### 🧠 Pattern: Same-Direction (Write Pointer + Final Fill)

### Approach

Two-pass: walk `fast`, copy non-zero to `slow`, increment slow. After loop, fill `slow..end` with zeros. (Or one-pass with swap.)

> [!example]- 📊 Visual: partition non-zeros to the front
> ```text
>   nums:  [ 0 , 1 , 0 , 3 , 12 ]
>            0   1   2   3    4
> 
>   Pass 1 — copy non-zeros to the front (preserving order):
> 
>      ┌──── front (non-zeros) ────┐┌──── tail (junk, will overwrite) ────┐
>      │                            ││                                     │
>      └────────────────────────────┘└─────────────────────────────────────┘
>          ↑                              ↑
>        slow                           fast
>      (write here)                  (reading cursor)
> 
>     fast=0 x=0   skip
>     fast=1 x=1   nums[slow=0] = 1 → [1,1,0,3,12]  slow=1
>     fast=2 x=0   skip
>     fast=3 x=3   nums[slow=1] = 3 → [1,3,0,3,12]  slow=2
>     fast=4 x=12  nums[slow=2] = 12→ [1,3,12,3,12] slow=3
> 
>   After pass 1:
>      ┌────────────────────┐
>      │  1   3   12 │  ?   ?   ← positions ≥ slow are garbage
>      └────────────────────┘
>            non-zeros      tail to zero out
> 
>   Pass 2 — zero-fill positions [slow..n):
> 
>      ┌─────────────────────┐
>      │  1   3   12   0   0 │   ← done
>      └─────────────────────┘
> 
>   Order of non-zeros is preserved because we COPY in scan order.
> ```

> [!info]- 🔍 Dry Run: nums=[0,1,0,3,12]
> ```text
> Setup:
>   slow = 0   (write position)
> 
> ─────────────────────────────────────────
> Phase 1 — Copy non-zeros to front:
> 
>   fast=0  nums[0]=0  → skip
>   fast=1  nums[1]=1  → nums[slow=0] = 1, slow=1
>           State: nums=[1,1,0,3,12], slow=1
>   fast=2  nums[2]=0  → skip
>   fast=3  nums[3]=3  → nums[slow=1] = 3, slow=2
>           State: nums=[1,3,0,3,12], slow=2
>   fast=4  nums[4]=12 → nums[slow=2] = 12, slow=3
>           State: nums=[1,3,12,3,12], slow=3
> 
> ─────────────────────────────────────────
> Phase 2 — Fill trailing positions with zeros:
> 
>   nums[3] = 0  →  nums=[1,3,12,0,12]
>   nums[4] = 0  →  nums=[1,3,12,0,0]
> 
> ✅ Answer: nums = [1, 3, 12, 0, 0]
> ```

> [!success]- JS
> ```js
> const moveZeroes = (nums) => {
>   let slow = 0;
>   for (let fast = 0; fast < nums.length; fast++) {
>     if (nums[fast] !== 0) nums[slow++] = nums[fast];
>   }
>   while (slow < nums.length) nums[slow++] = 0;
> };
> ```

> [!success]- Python
> ```python
> def move_zeroes(nums):
>     slow = 0
>     for fast in range(len(nums)):
>         if nums[fast] != 0:
>             nums[slow] = nums[fast]
>             slow += 1
>     for i in range(slow, len(nums)):
>         nums[i] = 0
> ```

**Variants:** Remove Element · Move negatives to one side.

**Key takeaway:** "Partition in place, preserve order" → write pointer + final fill.

---

## P10: Sort Colors

**LC #75** · Medium · **One pass, in place** · **Dutch National Flag**

Sort an array of 0s/1s/2s in place.

**Edge cases:** all same color · empty · already sorted.

### 🧠 Pattern: 3-Way Partition (Dutch National Flag)

> Three pointers — `low` (next 0 slot), `mid` (cursor), `high` (next 2 slot). Invariant: `[0..low-1]=0`, `[low..mid-1]=1`, `[high+1..end]=2`. As `mid` walks, dispatch based on value.

### Approach Evolution

1. **Counting sort (2 passes)** · O(n)/O(1). Works.
2. **Dutch flag (one pass) — FINAL** · O(n)/O(1).

> [!example]- 📊 Visual: 3-way partition
> ```text
>   Invariant during sweep:
> 
>     ┌──────┬─────────┬──────────────┬──────┐
>     │ 0s   │   1s    │   ??? unseen │  2s  │
>     └──────┴─────────┴──────────────┴──────┘
>     ↑      ↑         ↑              ↑      ↑
>     0     low       mid            high   n-1
> 
>   Three regions:
>     [0..low-1]   : all 0s         (sorted)
>     [low..mid-1] : all 1s         (sorted)
>     [mid..high]  : unprocessed
>     [high+1..n-1]: all 2s         (sorted)
> 
>   At each step, dispatch on nums[mid]:
>     0 → swap(low, mid); low++, mid++   (extend 0-region)
>     1 → mid++                          (extend 1-region)
>     2 → swap(mid, high); high--        (don't advance mid! new value unseen)
> 
>   Final state: all three regions correctly partitioned → sorted.
> ```

> [!info]- 🔍 Dry Run: nums=[2,0,2,1,1,0]
> ```text
> Setup:
>   low=0, mid=0, high=5
> 
> ─────────────────────────────────────────
> Step 1: nums[mid=0]=2
>   Value 2 → swap(mid, high), high--
>     swap(0, 5): nums=[0,0,2,1,1,2]
>     high=4
>   Do NOT advance mid! New value at mid is unprocessed.
>   State: nums=[0,0,2,1,1,2], low=0, mid=0, high=4
> 
> Step 2: nums[mid=0]=0
>   Value 0 → swap(low, mid), low++, mid++
>     swap(0, 0): no-op
>     low=1, mid=1
>   State: nums=[0,0,2,1,1,2], low=1, mid=1, high=4
> 
> Step 3: nums[mid=1]=0
>   Value 0 → swap(1, 1) no-op; low=2, mid=2
> 
> Step 4: nums[mid=2]=2
>   Value 2 → swap(mid, high), high--
>     swap(2, 4): nums=[0,0,1,1,2,2]
>     high=3
>   mid stays at 2.
> 
> Step 5: nums[mid=2]=1
>   Value 1 → just advance: mid=3
> 
> Step 6: nums[mid=3]=1
>   Value 1 → mid=4
> 
> mid=4 > high=3 → exit
> 
> ✅ Answer: nums=[0,0,1,1,2,2]
> ```

> [!success]- JS
> ```js
> const sortColors = (nums) => {
>   let low = 0, mid = 0, high = nums.length - 1;
>   while (mid <= high) {
>     if (nums[mid] === 0) {
>       [nums[low], nums[mid]] = [nums[mid], nums[low]];
>       low++; mid++;
>     } else if (nums[mid] === 2) {
>       [nums[mid], nums[high]] = [nums[high], nums[mid]];
>       high--;
>     } else mid++;
>   }
> };
> ```

> [!success]- Python
> ```python
> def sort_colors(nums):
>     low = mid = 0
>     high = len(nums) - 1
>     while mid <= high:
>         if nums[mid] == 0:
>             nums[low], nums[mid] = nums[mid], nums[low]
>             low += 1; mid += 1
>         elif nums[mid] == 2:
>             nums[mid], nums[high] = nums[high], nums[mid]
>             high -= 1
>         else:
>             mid += 1
> ```

> [!warning] Don't increment `mid` after swapping with `high`
> The swapped-in value is unprocessed.

**Variants:** Wiggle Sort · Partition by Pivot (Quickselect).

**Key takeaway:** 3-way partition → 3 pointers. Crystal-clear invariants.

---

## P11: Linked List Cycle

**LC #141** · Easy

Detect if a linked list has a cycle.

**Edge cases:** empty list · single node (no cycle / self-cycle) · 2 nodes.

### 🧠 Pattern: Floyd's Tortoise & Hare

> Two pointers, fast moves 2× speed. If there's a cycle, fast laps slow inside it — they must meet. If no cycle, fast hits `null` first.

### Approach Evolution

1. **Hash set of visited nodes** · O(n)/O(n).
2. **Floyd — FINAL** · O(n)/O(1).

> [!example]- 📊 Visual: tortoise and hare on the rho shape
> ```text
>   A cyclic linked list looks like the Greek letter ρ:
> 
>         1 ──▶ 2 ──▶ 3 ──▶ 4 ──▶ 5
>                ▲                  │
>                └──────────────────┘
>                 (5.next loops back to 2)
> 
>      tail (no cycle):  •──•──•   ←  fast hits null  → no cycle
>      rho (with cycle): same, plus a closed loop.
> 
>   Two pointers, two speeds:
> 
>      slow ── moves 1 step
>      fast ── moves 2 steps
> 
>   In a cycle, the GAP between fast and slow shrinks by 1 each tick.
>   So eventually gap = 0  →  they collide.
> 
>   Snapshot during traversal:
> 
>      tick 0:   slow=1   fast=1                       •
>      tick 1:   slow=2   fast=3                       │
>      tick 2:   slow=3   fast=5                       │ both in loop
>      tick 3:   slow=4   fast=3   (3 because 5→2→3)   │
>      tick 4:   slow=5   fast=5   ←  COLLISION ✓      ▼
> 
>   If there's NO cycle, fast (or fast.next) hits null first → return false.
> 
>   Why O(1) space? Just two pointers — no visited set needed.
> ```

> [!info]- 🔍 Dry Run: 1 → 2 → 3 → 4 → 5 → (back to 2)
> ```text
> Setup:
>   slow = head = node(1)
>   fast = head = node(1)
> 
> ─────────────────────────────────────────
> Step 1:
>   slow = slow.next       → node(2)
>   fast = fast.next.next  → node(3)
>   slow == fast?  → 2 vs 3, NO
> 
> Step 2:
>   slow = node(3)
>   fast = node(5)
>   slow == fast?  → 3 vs 5, NO
> 
> Step 3:
>   slow = node(4)
>   fast = fast.next.next = (5).next.next = (2).next = node(3)
>   slow == fast?  → 4 vs 3, NO
> 
> Step 4:
>   slow = node(5)
>   fast = (3).next.next = (4).next = node(5)
>   slow == fast?  → 5 == 5  → YES!
> 
> ✅ Answer: true (cycle detected)
> ```

> [!success]- JS
> ```js
> const hasCycle = (head) => {
>   let slow = head, fast = head;
>   while (fast && fast.next) {
>     slow = slow.next;
>     fast = fast.next.next;
>     if (slow === fast) return true;
>   }
>   return false;
> };
> ```

> [!success]- Python
> ```python
> def has_cycle(head):
>     slow = fast = head
>     while fast and fast.next:
>         slow = slow.next
>         fast = fast.next.next
>         if slow is fast:
>             return True
>     return False
> ```

**Variants:** Linked List Cycle II (find cycle start — math trick) · Happy Number (Floyd on integers).

**Key takeaway:** Cycle detection in O(1) space → Floyd. Stop check `fast && fast.next`.

---

## P12: Middle of the Linked List

**LC #876** · Easy

Return middle node. If two middles, return the second.

**Edge cases:** 1 node · 2 nodes · even/odd length.

### 🧠 Pattern: Fast/Slow → slow ends at middle

> Fast moves 2x. When fast hits end, slow is at the middle.

> [!example]- 📊 Visual: speed-2 pointer pins the middle
> ```text
>   Linked list:    1 ──▶ 2 ──▶ 3 ──▶ 4 ──▶ 5 ──▶ null
> 
>   When fast traverses 2n nodes, slow traverses n nodes
>   ⇒ slow is at the HALFWAY mark.
> 
>   Odd length (5):
> 
>      step 0:  slow=1   fast=1                  
>               ▼                                    
>               1 ─ 2 ─ 3 ─ 4 ─ 5                   
>               ▲                                    
> 
>      step 1:  slow=2   fast=3                  
>                   ▼      ▼                          
>               1 ─ 2 ─ 3 ─ 4 ─ 5                   
> 
>      step 2:  slow=3   fast=5                  
>                       ▼          ▼                  
>               1 ─ 2 ─ 3 ─ 4 ─ 5                   
> 
>      fast.next = null → STOP. slow at node 3 (the middle). ✓
> 
>   Even length (4):
> 
>      step 0:  slow=1   fast=1
>      step 1:  slow=2   fast=3
>      step 2:  slow=3   fast=null
>                       ▼
>               1 ─ 2 ─ 3 ─ 4
> 
>      STOP. slow=3 = "second middle" (problem spec).
> 
>   Same skeleton as cycle detection — fast/slow is a Swiss-army knife:
>     • cycle detection
>     • midpoint
>     • nth-from-end (gap-pointer variant)
> ```

> [!info]- 🔍 Dry Run: 1 → 2 → 3 → 4 → 5
> ```text
> Setup:
>   slow = fast = node(1)
> 
> ─────────────────────────────────────────
> Step 1:
>   fast && fast.next? fast=1, fast.next=2 → YES, continue
>   slow = node(2)
>   fast = fast.next.next = node(3)
> 
> Step 2:
>   fast=3, fast.next=4 → continue
>   slow = node(3)
>   fast = node(5)
> 
> Step 3:
>   fast=5, fast.next=null → STOP loop
> 
> ✅ Answer: slow = node(3)        (the middle of 1,2,3,4,5)
> 
> ─────────────────────────────────────────
> Even-length example: 1 → 2 → 3 → 4
>   slow=1, fast=1
>   slow=2, fast=3
>   slow=3, fast=null (3.next is 4, 4.next is null, but check fast && fast.next at top of loop)
>   
>   Actually: after first iter slow=2,fast=3. Check: fast=3, fast.next=4 → continue.
>             after second iter slow=3, fast=3.next.next=null.
>   Loop top: fast=null → STOP.
>   Return slow=3 (the SECOND middle, as required).
> ```

> [!success]- JS
> ```js
> const middleNode = (head) => {
>   let slow = head, fast = head;
>   while (fast && fast.next) {
>     slow = slow.next;
>     fast = fast.next.next;
>   }
>   return slow;
> };
> ```

> [!success]- Python
> ```python
> def middle_node(head):
>     slow = fast = head
>     while fast and fast.next:
>         slow = slow.next
>         fast = fast.next.next
>     return slow
> ```

**Variants:** Reorder List (find middle + reverse second + interleave) · Palindrome Linked List.

**Key takeaway:** Fast/slow is your **midpoint, length-detector, and cycle-detector** all in one.

---

> [!tip] After this drill
> When you see "pair sum / palindrome / in-place filter" → opposite pointers. When you see "linked list / cycle / midpoint" → fast/slow. Pattern recognition should be < 5 seconds.
