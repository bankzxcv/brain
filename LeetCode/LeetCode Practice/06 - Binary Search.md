---
title: "LeetCode Practice: Binary Search"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - binary-search
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Binary Search

14 problems · 2 sub-patterns: **search in sorted data** and **search on the answer space**.

> [!abstract] Pattern recap
> **Sorted data** — halve the range each step. O(log n).
> **Search on answer** — when "min/max value satisfying P(x)" + P is **monotonic** (true once, then always true; or vice versa), binary-search the answer range, not the array.

> [!warning] Three gotchas
> 1. `mid = l + ((r - l) >> 1)` to avoid overflow.
> 2. Decide upfront: `l ≤ r` with `r = mid - 1` (inclusive) **or** `l < r` with `r = mid` (half-open).
> 3. Off-by-one on boundary → trace 2-element and 3-element cases by hand.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Binary Search | 704 | Easy | Vanilla |
| P2 | Search Insert Position | 35 | Easy | Lower bound |
| P3 | First Bad Version | 278 | Easy | Boolean monotone |
| P4 | Find First and Last Position | 34 | Med | Two bounds |
| P5 | Search in Rotated Sorted Array | 33 | Med | Pivoted |
| P6 | Find Minimum in Rotated Sorted Array | 153 | Med | Pivot location |
| P7 | Search a 2D Matrix | 74 | Med | Flatten index |
| P8 | Search a 2D Matrix II | 240 | Med | Staircase (not BS) |
| P9 | Time Based Key-Value Store | 981 | Med | Floor lookup |
| P10 | Find Peak Element | 162 | Med | Direction of slope |
| P11 | Median of Two Sorted Arrays | 4 | Hard | Partition |
| P12 | Koko Eating Bananas | 875 | Med | Search on answer |
| P13 | Capacity to Ship Packages | 1011 | Med | Search on answer |
| P14 | Split Array Largest Sum | 410 | Hard | Search on answer |

---

## P1: Binary Search

**LC #704** · Easy

Find target in sorted array, return index or -1.

**Edge cases:** empty · single element · target smaller than min / larger than max · duplicates (return any index).

### 🧠 Pattern: Classic Halving

```
l, r = 0, n - 1
while l ≤ r:
    mid = (l + r) // 2
    if nums[mid] == target: return mid
    elif nums[mid] < target: l = mid + 1
    else: r = mid - 1
return -1
```

### Trace

```
nums=[1,3,5,7,9,11]  target=7
l=0 r=5 mid=2 nums[2]=5 <7 → l=3
l=3 r=5 mid=4 nums[4]=9 >7 → r=3
l=3 r=3 mid=3 nums[3]=7 = ✓ return 3
```

> [!success]- JS
> ```js
> const search = (nums, target) => {
>   let l = 0, r = nums.length - 1;
>   while (l <= r) {
>     const mid = (l + r) >> 1;
>     if (nums[mid] === target) return mid;
>     if (nums[mid] < target) l = mid + 1;
>     else r = mid - 1;
>   }
>   return -1;
> };
> ```

> [!success]- Python
> ```python
> def search(nums, target):
>     l, r = 0, len(nums) - 1
>     while l <= r:
>         mid = (l + r) // 2
>         if nums[mid] == target: return mid
>         elif nums[mid] < target: l = mid + 1
>         else: r = mid - 1
>     return -1
> ```

**Key takeaway:** Memorize the closed-interval template. Variants reuse this skeleton.

---

## P2: Search Insert Position

**LC #35** · Easy

Return index where `target` should be inserted to keep array sorted.

### 🧠 Pattern: Lower Bound

> Find smallest index `i` with `nums[i] ≥ target`. Same skeleton, return `l` when loop exits.

> [!success]- JS
> ```js
> const searchInsert = (nums, target) => {
>   let l = 0, r = nums.length;
>   while (l < r) {
>     const mid = (l + r) >> 1;
>     if (nums[mid] < target) l = mid + 1;
>     else r = mid;
>   }
>   return l;
> };
> ```

> [!success]- Python
> ```python
> def search_insert(nums, target):
>     l, r = 0, len(nums)
>     while l < r:
>         mid = (l + r) // 2
>         if nums[mid] < target: l = mid + 1
>         else: r = mid
>     return l
> ```

**Key takeaway:** Half-open template (`r = n`, `l < r`, `r = mid`) gives lower bound. Final `l` is insertion point.

---

## P3: First Bad Version

**LC #278** · Easy

Find first bad version. Sequence is `[good, good, ..., bad, bad, ...]` — monotone boolean.

### 🧠 Pattern: Boolean Monotone

> Reframe: find smallest `v` with `isBadVersion(v) = true`. Half-open template.

> [!success]- JS
> ```js
> const firstBadVersion = (n) => {
>   let l = 1, r = n;
>   while (l < r) {
>     const mid = l + ((r - l) >> 1);
>     if (isBadVersion(mid)) r = mid;
>     else l = mid + 1;
>   }
>   return l;
> };
> ```

> [!success]- Python
> ```python
> def first_bad_version(n):
>     l, r = 1, n
>     while l < r:
>         mid = (l + r) // 2
>         if isBadVersion(mid):
>             r = mid
>         else:
>             l = mid + 1
>     return l
> ```

**Key takeaway:** Binary search needs only a **monotonic predicate**, not sorted values. "First true" → half-open.

---

## P4: Find First and Last Position of Element in Sorted Array

**LC #34** · Medium

Return `[first, last]` indices of target.

### 🧠 Pattern: Two Bounds (Lower + Upper)

> first = lower_bound(target). last = lower_bound(target+1) - 1.

> [!success]- JS
> ```js
> const searchRange = (nums, target) => {
>   const lower = (t) => {
>     let l = 0, r = nums.length;
>     while (l < r) {
>       const mid = (l + r) >> 1;
>       if (nums[mid] < t) l = mid + 1; else r = mid;
>     }
>     return l;
>   };
>   const lo = lower(target);
>   if (lo === nums.length || nums[lo] !== target) return [-1, -1];
>   return [lo, lower(target + 1) - 1];
> };
> ```

> [!success]- Python
> ```python
> def search_range(nums, target):
>     def lower(t):
>         l, r = 0, len(nums)
>         while l < r:
>             mid = (l + r) // 2
>             if nums[mid] < t: l = mid + 1
>             else: r = mid
>         return l
>     lo = lower(target)
>     if lo == len(nums) or nums[lo] != target:
>         return [-1, -1]
>     return [lo, lower(target + 1) - 1]
> ```

**Key takeaway:** Range = `[lower(t), lower(t+1) - 1]`. Two calls, no edge-case logic.

---

## P5: Search in Rotated Sorted Array

**LC #33** · Medium · No duplicates

Sorted array rotated at some pivot. Find target.

### 🧠 Pattern: One Half Is Always Sorted

> At each step, `[l..mid]` or `[mid..r]` is sorted. Check which; if target in sorted half, narrow there; else go the other way.

### Trace

```
nums=[4,5,6,7,0,1,2]  target=0
l=0 r=6 mid=3 nums[3]=7
  left [4..7] sorted. Is 0 in [4,7]? no → go right. l=4
l=4 r=6 mid=5 nums[5]=1
  right [1,2] sorted. Is 0 in [1,2]? no → go left. r=4
l=4 r=4 mid=4 nums[4]=0 ✓ return 4
```

> [!success]- JS
> ```js
> const searchRotated = (nums, target) => {
>   let l = 0, r = nums.length - 1;
>   while (l <= r) {
>     const mid = (l + r) >> 1;
>     if (nums[mid] === target) return mid;
>     if (nums[l] <= nums[mid]) {
>       if (nums[l] <= target && target < nums[mid]) r = mid - 1;
>       else l = mid + 1;
>     } else {
>       if (nums[mid] < target && target <= nums[r]) l = mid + 1;
>       else r = mid - 1;
>     }
>   }
>   return -1;
> };
> ```

> [!success]- Python
> ```python
> def search_rotated(nums, target):
>     l, r = 0, len(nums) - 1
>     while l <= r:
>         mid = (l + r) // 2
>         if nums[mid] == target: return mid
>         if nums[l] <= nums[mid]:
>             if nums[l] <= target < nums[mid]: r = mid - 1
>             else: l = mid + 1
>         else:
>             if nums[mid] < target <= nums[r]: l = mid + 1
>             else: r = mid - 1
>     return -1
> ```

**Variants:** P5 with duplicates (LC #81 — degrades to O(n) worst case).

**Key takeaway:** Rotated sorted = "one half always sorted". Check sortedness, then check inclusion.

---

## P6: Find Minimum in Rotated Sorted Array

**LC #153** · Medium

### 🧠 Pattern: Compare mid to right

> If `nums[mid] > nums[r]`, min is in right half. Else, in left half (including mid).

> [!success]- JS
> ```js
> const findMin = (nums) => {
>   let l = 0, r = nums.length - 1;
>   while (l < r) {
>     const mid = (l + r) >> 1;
>     if (nums[mid] > nums[r]) l = mid + 1;
>     else r = mid;
>   }
>   return nums[l];
> };
> ```

> [!success]- Python
> ```python
> def find_min(nums):
>     l, r = 0, len(nums) - 1
>     while l < r:
>         mid = (l + r) // 2
>         if nums[mid] > nums[r]: l = mid + 1
>         else: r = mid
>     return nums[l]
> ```

**Key takeaway:** Comparison to `r` (not `l`) is more robust for finding the pivot.

---

## P7: Search a 2D Matrix

**LC #74** · Medium · Rows + columns globally sorted

### 🧠 Pattern: Flatten Index

> Treat matrix as 1D of length `m*n`. `mid` index `k` maps to `(k / cols, k % cols)`.

> [!success]- JS
> ```js
> const searchMatrix = (m, target) => {
>   const rows = m.length, cols = m[0].length;
>   let l = 0, r = rows * cols - 1;
>   while (l <= r) {
>     const mid = (l + r) >> 1;
>     const v = m[Math.floor(mid / cols)][mid % cols];
>     if (v === target) return true;
>     if (v < target) l = mid + 1; else r = mid - 1;
>   }
>   return false;
> };
> ```

> [!success]- Python
> ```python
> def search_matrix(m, target):
>     rows, cols = len(m), len(m[0])
>     l, r = 0, rows * cols - 1
>     while l <= r:
>         mid = (l + r) // 2
>         v = m[mid // cols][mid % cols]
>         if v == target: return True
>         if v < target: l = mid + 1
>         else: r = mid - 1
>     return False
> ```

**Key takeaway:** Globally sorted 2D → 1D BS via index flatten.

---

## P8: Search a 2D Matrix II

**LC #240** · Medium · Rows AND columns sorted (NOT globally)

### 🧠 Pattern: Staircase from Top-Right

> Start at top-right corner. If value > target → go left. If value < target → go down. Each step eliminates a row or column → O(m+n).

### Trace

```
m=[[1,4,7,11],[2,5,8,12],[3,6,9,16],[10,13,14,17]]  target=5
r=0 c=3 v=11>5 → c--
r=0 c=2 v=7>5  → c--
r=0 c=1 v=4<5  → r++
r=1 c=1 v=5 ✓ return true
```

> [!success]- JS
> ```js
> const searchMatrixII = (m, target) => {
>   let r = 0, c = m[0].length - 1;
>   while (r < m.length && c >= 0) {
>     if (m[r][c] === target) return true;
>     if (m[r][c] > target) c--;
>     else r++;
>   }
>   return false;
> };
> ```

> [!success]- Python
> ```python
> def search_matrix_ii(m, target):
>     r, c = 0, len(m[0]) - 1
>     while r < len(m) and c >= 0:
>         if m[r][c] == target: return True
>         if m[r][c] > target: c -= 1
>         else: r += 1
>     return False
> ```

**Key takeaway:** Not all "matrix search" is binary search. Staircase O(m+n) beats naive BS.

---

## P9: Time Based Key-Value Store

**LC #981** · Medium · Design

`set(k, v, ts)` and `get(k, ts)` returns the value set at the **largest timestamp ≤ ts**.

### 🧠 Pattern: Floor Lookup via Binary Search

> Per key, store sorted list of `(timestamp, value)`. `get` = upper_bound(ts) - 1.

> [!success]- JS
> ```js
> class TimeMap {
>   constructor() { this.data = new Map(); }
>   set(k, v, ts) {
>     if (!this.data.has(k)) this.data.set(k, []);
>     this.data.get(k).push([ts, v]);
>   }
>   get(k, ts) {
>     const arr = this.data.get(k);
>     if (!arr) return "";
>     let l = 0, r = arr.length;
>     while (l < r) {
>       const mid = (l + r) >> 1;
>       if (arr[mid][0] <= ts) l = mid + 1;
>       else r = mid;
>     }
>     return l === 0 ? "" : arr[l - 1][1];
>   }
> }
> ```

> [!success]- Python
> ```python
> from bisect import bisect_right
> class TimeMap:
>     def __init__(self): self.data = {}
>     def set(self, k, v, ts):
>         self.data.setdefault(k, []).append((ts, v))
>     def get(self, k, ts):
>         arr = self.data.get(k, [])
>         i = bisect_right(arr, (ts, chr(0x10FFFF)))
>         return arr[i - 1][1] if i else ""
> ```

**Key takeaway:** Floor / ceiling lookups in sorted lists → upper_bound / lower_bound. `bisect` is Python's friend.

---

## P10: Find Peak Element

**LC #162** · Medium · O(log n)

Find any peak (`nums[i] > nums[i-1] && nums[i] > nums[i+1]`). Edges treated as -∞.

### 🧠 Pattern: Direction of Slope

> If `nums[mid] < nums[mid+1]`, a peak exists on the **right** (slope is climbing). Else, on the left. Half-open BS converges to a peak.

> [!success]- JS
> ```js
> const findPeakElement = (nums) => {
>   let l = 0, r = nums.length - 1;
>   while (l < r) {
>     const mid = (l + r) >> 1;
>     if (nums[mid] < nums[mid + 1]) l = mid + 1;
>     else r = mid;
>   }
>   return l;
> };
> ```

> [!success]- Python
> ```python
> def find_peak_element(nums):
>     l, r = 0, len(nums) - 1
>     while l < r:
>         mid = (l + r) // 2
>         if nums[mid] < nums[mid + 1]: l = mid + 1
>         else: r = mid
>     return l
> ```

**Key takeaway:** Unsorted array, but **local monotonicity** still enables BS. Follow the climb.

---

## P11: Median of Two Sorted Arrays

**LC #4** · Hard · O(log(min(m,n)))

### 🧠 Pattern: Binary Search the Partition Point

> Pick `i` in A. Then `j = (m+n+1)/2 - i` in B. The split `[A[0..i-1] | A[i..]]` and `[B[0..j-1] | B[j..]]` is correct iff `A[i-1] ≤ B[j]` and `B[j-1] ≤ A[i]`. BS on `i` until both hold.

> [!success]- JS
> ```js
> const findMedianSortedArrays = (a, b) => {
>   if (a.length > b.length) [a, b] = [b, a];
>   const m = a.length, n = b.length;
>   const half = (m + n + 1) >> 1;
>   let l = 0, r = m;
>   while (l <= r) {
>     const i = (l + r) >> 1;
>     const j = half - i;
>     const aL = i === 0 ? -Infinity : a[i - 1];
>     const aR = i === m ? Infinity : a[i];
>     const bL = j === 0 ? -Infinity : b[j - 1];
>     const bR = j === n ? Infinity : b[j];
>     if (aL <= bR && bL <= aR) {
>       if ((m + n) % 2) return Math.max(aL, bL);
>       return (Math.max(aL, bL) + Math.min(aR, bR)) / 2;
>     } else if (aL > bR) r = i - 1;
>     else l = i + 1;
>   }
> };
> ```

> [!success]- Python
> ```python
> def find_median_sorted_arrays(a, b):
>     if len(a) > len(b): a, b = b, a
>     m, n = len(a), len(b)
>     half = (m + n + 1) // 2
>     l, r = 0, m
>     INF = float('inf')
>     while l <= r:
>         i = (l + r) // 2
>         j = half - i
>         aL = -INF if i == 0 else a[i - 1]
>         aR = INF if i == m else a[i]
>         bL = -INF if j == 0 else b[j - 1]
>         bR = INF if j == n else b[j]
>         if aL <= bR and bL <= aR:
>             if (m + n) % 2: return max(aL, bL)
>             return (max(aL, bL) + min(aR, bR)) / 2
>         elif aL > bR: r = i - 1
>         else: l = i + 1
> ```

**Key takeaway:** "Median of merged" = "partition so left half = right half in size, with crossing inequalities". BS on smaller array.

---

## P12: Koko Eating Bananas

**LC #875** · Medium

Min eating speed `k` so Koko finishes all piles within `h` hours. `time(k) = sum(ceil(pile / k))`.

### 🧠 Pattern: Binary Search on Answer

> `time(k)` is **monotonically decreasing** in `k`. Search smallest `k` with `time(k) ≤ h`.

> Range: `1` to `max(piles)`.

### Trace

```
piles=[3,6,7,11]  h=8
range [1, 11]
k=6 → ceil(3/6)+ceil(6/6)+ceil(7/6)+ceil(11/6) = 1+1+2+2=6 ≤8 → r=6
k=3 → 1+2+3+4=10 >8 → l=4
k=5 → 1+2+2+3=8 ≤8 → r=5
k=4 → 1+2+2+3=8 ≤8 → r=4
l=r=4 return 4
```

> [!success]- JS
> ```js
> const minEatingSpeed = (piles, h) => {
>   let l = 1, r = Math.max(...piles);
>   const can = (k) => piles.reduce((s, p) => s + Math.ceil(p / k), 0) <= h;
>   while (l < r) {
>     const mid = (l + r) >> 1;
>     if (can(mid)) r = mid;
>     else l = mid + 1;
>   }
>   return l;
> };
> ```

> [!success]- Python
> ```python
> from math import ceil
> def min_eating_speed(piles, h):
>     l, r = 1, max(piles)
>     def can(k):
>         return sum(ceil(p / k) for p in piles) <= h
>     while l < r:
>         mid = (l + r) // 2
>         if can(mid): r = mid
>         else: l = mid + 1
>     return l
> ```

**Variants:** Capacity to Ship (P13) · Split Array Largest Sum (P14) · Magnetic Force Between Two Balls.

**Key takeaway:** Problem says "min/max value that satisfies X" + X is monotonic → BS on the answer. Define `can(x)` simulator, search.

---

## P13: Capacity to Ship Packages Within D Days

**LC #1011** · Medium

Min ship capacity so all packages ship in ≤ D days, in given order.

### Approach

`can(cap) = days_needed(cap) ≤ D`. Range `[max(weights), sum(weights)]`. Same template as P12.

> [!success]- JS
> ```js
> const shipWithinDays = (w, days) => {
>   const can = (cap) => {
>     let d = 1, cur = 0;
>     for (const x of w) {
>       if (cur + x > cap) { d++; cur = 0; }
>       cur += x;
>     }
>     return d <= days;
>   };
>   let l = Math.max(...w), r = w.reduce((a, b) => a + b, 0);
>   while (l < r) {
>     const mid = (l + r) >> 1;
>     if (can(mid)) r = mid;
>     else l = mid + 1;
>   }
>   return l;
> };
> ```

> [!success]- Python
> ```python
> def ship_within_days(w, days):
>     def can(cap):
>         d, cur = 1, 0
>         for x in w:
>             if cur + x > cap:
>                 d += 1; cur = 0
>             cur += x
>         return d <= days
>     l, r = max(w), sum(w)
>     while l < r:
>         mid = (l + r) // 2
>         if can(mid): r = mid
>         else: l = mid + 1
>     return l
> ```

**Key takeaway:** `range = [max(item), sum(items)]` is the canonical bound for "min capacity" problems.

---

## P14: Split Array Largest Sum

**LC #410** · Hard

Split `nums` into `m` non-empty subarrays. Minimize the **maximum subarray sum**.

### Approach

Same as P13. `can(maxSum) = "can we split into ≤ m parts each ≤ maxSum"`.

> [!success]- JS
> ```js
> const splitArray = (nums, m) => {
>   const can = (cap) => {
>     let parts = 1, cur = 0;
>     for (const x of nums) {
>       if (cur + x > cap) { parts++; cur = 0; }
>       cur += x;
>     }
>     return parts <= m;
>   };
>   let l = Math.max(...nums), r = nums.reduce((a, b) => a + b, 0);
>   while (l < r) {
>     const mid = (l + r) >> 1;
>     if (can(mid)) r = mid;
>     else l = mid + 1;
>   }
>   return l;
> };
> ```

> [!success]- Python
> ```python
> def split_array(nums, m):
>     def can(cap):
>         parts, cur = 1, 0
>         for x in nums:
>             if cur + x > cap:
>                 parts += 1; cur = 0
>             cur += x
>         return parts <= m
>     l, r = max(nums), sum(nums)
>     while l < r:
>         mid = (l + r) // 2
>         if can(mid): r = mid
>         else: l = mid + 1
>     return l
> ```

**Key takeaway:** Identical pattern to P13. "Minimax" problems → BS on the answer + greedy `can`.

---

> [!tip] After this drill
> Memorize three templates: **closed** (`l ≤ r`), **half-open** (`l < r`), and **search-on-answer**. The third is what separates senior from junior interview candidates.
