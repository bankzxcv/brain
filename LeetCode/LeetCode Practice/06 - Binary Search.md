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
| P11 | Median of Two Sorted Arrays | 4 | **Hard** | Partition |
| P12 | Koko Eating Bananas | 875 | Med | Search on answer |
| P13 | Capacity to Ship Packages | 1011 | Med | Search on answer |
| P14 | Split Array Largest Sum | 410 | **Hard** | Search on answer |

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

> [!example]- 📊 Visual: search space shrinking
> ```text
>   nums = [1, 3, 5, 7, 9, 11]   target = 7
>   idx:    0  1  2  3  4  5
> 
>   Iter 1:   l=0                            r=5
>             ▼                              ▼
>           ┌───┬───┬───┬───┬───┬───┐
>           │ 1 │ 3 │ 5 │ 7 │ 9 │11 │
>           └───┴───┴───┴───┴───┴───┘
>                       ▲ mid=2 (val 5)
>             5 < 7 → search RIGHT half
> 
>   Iter 2:                  l=3       r=5
>                            ▼         ▼
>           ┌───┬───┬───┬───┬───┬───┐
>           │░░░│░░░│░░░│ 7 │ 9 │11 │
>           └───┴───┴───┴───┴───┴───┘
>                                ▲ mid=4 (val 9)
>             9 > 7 → search LEFT half (within current range)
> 
>   Iter 3:                  l=3 
>                            r=3
>                            ▼
>           ┌───┬───┬───┬───┬───┬───┐
>           │░░░│░░░│░░░│ 7 │░░░│░░░│
>           └───┴───┴───┴───┴───┴───┘
>                            ▲ mid=3 (val 7) → MATCH
> 
>   Each iteration HALVES the search space → log₂(n) iterations max.
>   For n=10⁶: only ~20 iterations.
> ```

> [!info]- 🔍 Dry Run: nums=[1,3,5,7,9,11], target=7
> ```text
> Setup:
>   l=0, r=5
> 
> ─────────────────────────────────────────
> Iter 1:
>   mid = (0+5)/2 = 2
>   nums[2] = 5
>   5 vs target=7:  5 < 7 → answer is on the right
>   l = mid + 1 = 3
>   State: l=3, r=5
> 
> Iter 2:
>   mid = (3+5)/2 = 4
>   nums[4] = 9
>   9 > 7 → answer on the left
>   r = mid - 1 = 3
>   State: l=3, r=3
> 
> Iter 3:
>   mid = 3
>   nums[3] = 7
>   7 == target → MATCH
>   return 3
> 
> ✅ Answer: 3
> 
> ─────────────────────────────────────────
> Counter-example: target=4 (not present)
>   l=0 r=5 mid=2 nums[2]=5 > 4 → r=1
>   l=0 r=1 mid=0 nums[0]=1 < 4 → l=1
>   l=1 r=1 mid=1 nums[1]=3 < 4 → l=2
>   l=2 > r=1 → exit → return -1
> ```

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

> [!info]- 🔍 Dry Run: nums=[1,3,5,6], target=5
> ```text
> Setup:
>   l=0, r=4 (note: r=len, not len-1, in half-open)
> 
> ─────────────────────────────────────────
> Iter 1:
>   mid = (0+4)/2 = 2
>   nums[2] = 5
>   5 < target=5? NO → answer ≤ mid
>   r = mid = 2
>   State: l=0, r=2
> 
> Iter 2:
>   mid = 1
>   nums[1] = 3
>   3 < 5? YES → answer > mid
>   l = mid + 1 = 2
>   State: l=2, r=2
> 
> Loop exits (l == r).
> 
> ✅ Answer: l = 2   (position where 5 already is — first index ≥ 5)
> 
> ─────────────────────────────────────────
> Example target=2:
>   l=0 r=4 mid=2 nums[2]=5 >=2 → r=2
>   l=0 r=2 mid=1 nums[1]=3 >=2 → r=1
>   l=0 r=1 mid=0 nums[0]=1 < 2 → l=1
>   l=1 r=1 exit → return 1 (insert position: between nums[0]=1 and nums[1]=3)
> ```

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

> [!info]- 🔍 Dry Run: n=5, first bad = 4 (versions 1,2,3 good; 4,5 bad)
> ```text
> Setup:
>   l=1, r=5
> 
> ─────────────────────────────────────────
> Iter 1:
>   mid = (1+5)/2 = 3
>   isBadVersion(3) → false  (good)
>   l = mid + 1 = 4
>   State: l=4, r=5
> 
> Iter 2:
>   mid = (4+5)/2 = 4
>   isBadVersion(4) → true  (bad)
>   r = mid = 4
>   State: l=4, r=4
> 
> Loop exits.
> 
> ✅ Answer: 4
> ```

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

> [!info]- 🔍 Dry Run: nums=[5,7,7,8,8,10], target=8
> ```text
> First call lower(8) — smallest index with nums[i] ≥ 8:
>   l=0 r=6 mid=3 nums[3]=8 ≥ 8 → r=3
>   l=0 r=3 mid=1 nums[1]=7 < 8 → l=2
>   l=2 r=3 mid=2 nums[2]=7 < 8 → l=3
>   l=3 r=3 exit → lo = 3
> 
> Check: nums[3]=8 == 8?  YES → first = 3
> 
> ─────────────────────────────────────────
> Second call lower(9) — smallest index with nums[i] ≥ 9:
>   l=0 r=6 mid=3 nums[3]=8 < 9 → l=4
>   l=4 r=6 mid=5 nums[5]=10 ≥ 9 → r=5
>   l=4 r=5 mid=4 nums[4]=8 < 9 → l=5
>   l=5 r=5 exit → 5
> 
> last = lower(9) - 1 = 4
> 
> ✅ Answer: [3, 4]
>   Verify: nums[3..4] = [8, 8] ✓
> ```

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

> [!info]- 🔍 Dry Run: nums=[4,5,6,7,0,1,2], target=0
> ```text
> Setup:
>   l=0, r=6
> 
> ─────────────────────────────────────────
> Iter 1:
>   mid = 3, nums[3] = 7
>   7 == 0? NO
>   Is LEFT [l..mid] sorted? nums[l=0]=4 ≤ nums[mid]=7 → YES, left is sorted
>     Is target=0 in [4, 7)? NO (0 < 4)
>     → search RIGHT: l = mid + 1 = 4
>   State: l=4, r=6
> 
> Iter 2:
>   mid = 5, nums[5] = 1
>   1 == 0? NO
>   Is LEFT [4..5] sorted? nums[4]=0 ≤ nums[5]=1 → YES
>     Is target=0 in [0, 1)? YES (0 ≤ 0 < 1)
>     → search LEFT: r = mid - 1 = 4
>   State: l=4, r=4
> 
> Iter 3:
>   mid = 4, nums[4] = 0
>   0 == 0 → MATCH
>   return 4
> 
> ✅ Answer: 4
> ```

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

> [!info]- 🔍 Dry Run: nums=[4,5,6,7,0,1,2]
> ```text
> Setup:
>   l=0, r=6
> 
> ─────────────────────────────────────────
> Iter 1:
>   mid = 3, nums[3] = 7, nums[r=6] = 2
>   7 > 2 → min in RIGHT half (strictly after mid)
>   l = mid + 1 = 4
>   State: l=4, r=6
> 
> Iter 2:
>   mid = 5, nums[5] = 1, nums[r=6] = 2
>   1 > 2? NO → min in LEFT half (including mid)
>   r = mid = 5
>   State: l=4, r=5
> 
> Iter 3:
>   mid = 4, nums[4] = 0, nums[r=5] = 1
>   0 > 1? NO → r = mid = 4
>   State: l=4, r=4
> 
> Loop exits.
> 
> ✅ Answer: nums[l] = nums[4] = 0
> ```

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

> [!info]- 🔍 Dry Run: matrix=[[1,3,5,7],[10,11,16,20],[23,30,34,60]], target=3
> ```text
> rows=3, cols=4 → total 12 cells, flat indices 0..11
>   flat[0..3]  = [1, 3, 5, 7]      (row 0)
>   flat[4..7]  = [10, 11, 16, 20]  (row 1)
>   flat[8..11] = [23, 30, 34, 60]  (row 2)
> 
> ─────────────────────────────────────────
> l=0, r=11
> 
> Iter 1:
>   mid = 5
>   r=mid/cols=1, c=mid%cols=1 → matrix[1][1] = 11
>   11 == 3? NO
>   11 > 3 → r = 4
>   State: l=0, r=4
> 
> Iter 2:
>   mid = 2
>   r=0, c=2 → matrix[0][2] = 5
>   5 > 3 → r = 1
>   State: l=0, r=1
> 
> Iter 3:
>   mid = 0
>   matrix[0][0] = 1
>   1 < 3 → l = 1
>   State: l=1, r=1
> 
> Iter 4:
>   mid = 1
>   matrix[0][1] = 3
>   3 == 3 → MATCH return true
> 
> ✅ Answer: true
> ```

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

> [!info]- 🔍 Dry Run: matrix=[[1,4,7,11],[2,5,8,12],[3,6,9,16],[10,13,14,17]], target=5
> ```text
> Setup:
>   r = 0 (top), c = 3 (rightmost)
> 
> ─────────────────────────────────────────
> Step 1: r=0, c=3, m[0][3] = 11
>   11 > 5? YES → go LEFT → c--
>   State: r=0, c=2
> 
> Step 2: m[0][2] = 7
>   7 > 5? YES → c--
>   State: r=0, c=1
> 
> Step 3: m[0][1] = 4
>   4 > 5? NO; 4 < 5? YES → go DOWN → r++
>   State: r=1, c=1
> 
> Step 4: m[1][1] = 5
>   5 == 5 → MATCH return true
> 
> ✅ Answer: true
> ```

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

> [!info]- 🔍 Dry Run: set("foo","bar",1), set("foo","bar2",4), get("foo",3), get("foo",5)
> ```text
> After sets:
>   data["foo"] = [(1,"bar"), (4,"bar2")]  (sorted by ts)
> 
> ─────────────────────────────────────────
> get("foo", 3):
>   binary-search upper bound of 3 in timestamps [1, 4]:
>     l=0 r=2 mid=1: ts[1]=4 ≤ 3? NO → r=1
>     l=0 r=1 mid=0: ts[0]=1 ≤ 3? YES → l=1
>     exit: i=1
>   return data["foo"][i-1=0].value = "bar"
> 
> ─────────────────────────────────────────
> get("foo", 5):
>   upper bound of 5: [1,4] both ≤ 5 → i=2
>   return data["foo"][1].value = "bar2"
> 
> ─────────────────────────────────────────
> get("foo", 0):
>   upper bound: 0 < 1 → i=0
>   i==0 → no value at/before this ts → return ""
> ```

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

> [!info]- 🔍 Dry Run: nums=[1,2,3,1]
> ```text
> Setup:
>   l=0, r=3
> 
> ─────────────────────────────────────────
> Iter 1:
>   mid = 1
>   nums[1]=2, nums[mid+1=2]=3
>   2 < 3? YES → slope going UP → peak is to the right
>   l = mid + 1 = 2
>   State: l=2, r=3
> 
> Iter 2:
>   mid = 2
>   nums[2]=3, nums[3]=1
>   3 < 1? NO → slope going DOWN → peak is here or to the left
>   r = mid = 2
>   State: l=2, r=2
> 
> Loop exits.
> 
> ✅ Answer: index 2 (value 3)
> ```

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

**LC #4** · **Hard** · O(log(min(m,n)))

### 🧠 Pattern: Binary Search the Partition Point

> Pick `i` in A. Then `j = (m+n+1)/2 - i` in B. The split `[A[0..i-1] | A[i..]]` and `[B[0..j-1] | B[j..]]` is correct iff `A[i-1] ≤ B[j]` and `B[j-1] ≤ A[i]`. BS on `i` until both hold.

> [!info]- 🔍 Dry Run: A=[1,3], B=[2,4,5,6]
> ```text
> Ensure A is the smaller: A=[1,3] (m=2), B=[2,4,5,6] (n=4). A is smaller ✓
> half = (m+n+1)/2 = 3
> 
> ─────────────────────────────────────────
> Iter 1: BS on i in [0..2]
>   mid i=1, j = 3-1 = 2
>   Partition:
>     A_left = [1], A_right = [3]
>     B_left = [2,4], B_right = [5,6]
>     aL=1, aR=3; bL=4, bR=5
>   Check: aL ≤ bR (1 ≤ 5 ✓) AND bL ≤ aR (4 ≤ 3? NO)
>   bL > aR → A_left is too small; need more from A
>   l = i + 1 = 2
> 
> Iter 2: l=2, r=2 → i=2
>   j = 3 - 2 = 1
>   aL = A[1]=3, aR = INF (i==m)
>   bL = B[0]=2, bR = B[1]=4
>   Check: aL ≤ bR (3 ≤ 4 ✓) AND bL ≤ aR (2 ≤ INF ✓)
>   Partition valid!
>   Total length m+n=6 even → median = (max(aL, bL) + min(aR, bR)) / 2
>                            = (max(3,2) + min(INF,4)) / 2
>                            = (3 + 4) / 2 = 3.5
> 
> ✅ Answer: 3.5
>   Sanity: merged sorted = [1,2,3,4,5,6], median = (3+4)/2 = 3.5 ✓
> ```

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

> [!info]- 🔍 Dry Run: piles=[3,6,7,11], h=8
> ```text
> Setup:
>   l=1, r=max(piles)=11
> 
> ─────────────────────────────────────────
> Iter 1:
>   mid k=6
>   time(6) = ceil(3/6)+ceil(6/6)+ceil(7/6)+ceil(11/6)
>           = 1 + 1 + 2 + 2 = 6
>   6 ≤ 8? YES → can do with k=6, try smaller
>   r = 6
>   State: l=1, r=6
> 
> Iter 2:
>   mid k=3
>   time(3) = ceil(3/3)+ceil(6/3)+ceil(7/3)+ceil(11/3)
>           = 1 + 2 + 3 + 4 = 10
>   10 ≤ 8? NO → can't do with k=3, try larger
>   l = 4
>   State: l=4, r=6
> 
> Iter 3:
>   mid k=5
>   time(5) = 1+2+2+3 = 8
>   8 ≤ 8? YES → r=5
>   State: l=4, r=5
> 
> Iter 4:
>   mid k=4
>   time(4) = 1+2+2+3 = 8
>   8 ≤ 8 → r=4
>   State: l=4, r=4
> 
> Loop exits.
> 
> ✅ Answer: k = 4
> ```

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

> [!info]- 🔍 Dry Run: weights=[1,2,3,4,5,6,7,8,9,10], days=5
> ```text
> Setup:
>   l = max(w) = 10  (must fit the biggest single package)
>   r = sum(w) = 55  (ship everything in one day)
> 
> ─────────────────────────────────────────
> Iter 1:
>   mid cap = 32
>   Simulate: d=1, cur=0
>     w=1: cur=1
>     w=2: cur=3
>     ... all fit in one load? 1+2+...+10=55 > 32
>     Going through: load 1+2+3+4+5+6+7=28 (next 8 would overflow), d=2, cur=8
>     8+9=17, 17+10=27, OK. Total d=2.
>   2 ≤ 5? YES → r=32
> 
> Iter 2:
>   mid = (10+32)/2 = 21
>   Simulate: 1+2+...+6=21 fits, then need d=2 for 7,8 (15), d=3 for 9 (9), d=4 for 10 (10). d=4.
>   4 ≤ 5? YES → r=21
> 
> Iter 3:
>   mid = (10+21)/2 = 15
>   Simulate: 1+2+3+4+5=15 d=1. 6+7=13 then +8=21>15 → 6 alone? 6+7+? 6+7=13, +8=21>15. So 6+7=13 d=2. Then 8 alone (d=3). 9+? 9+10=19>15. 9 alone d=4. 10 alone d=5. Total d=5.
>   Actually let me redo: simulate(15):
>     cur=0, d=1
>     +1=1, +2=3, +3=6, +4=10, +5=15, +6=21>15 → d=2,cur=6
>     +7=13, +8=21>15 → d=3,cur=8
>     +9=17>15 → d=4,cur=9
>     +10=19>15 → d=5,cur=10
>     end: d=5
>   5 ≤ 5? YES → r=15
> 
> Iter 4:
>   mid = (10+15)/2 = 12
>   simulate(12):
>     +1=1, +2=3, +3=6, +4=10, +5=15>12 → d=2,cur=5
>     +6=11, +7=18>12 → d=3,cur=7
>     +8=15>12 → d=4, cur=8
>     +9=17>12 → d=5, cur=9
>     +10=19>12 → d=6, cur=10
>     d=6
>   6 ≤ 5? NO → l=13
> 
> Iter 5:
>   mid = 14
>   simulate(14):
>     1+2+3+4=10, +5=15>14 → d=2,cur=5
>     5+6=11, +7=18>14 → d=3,cur=7
>     7+8=15>14 → d=4,cur=8
>     8+9=17>14 → d=5,cur=9
>     9+10=19>14 → d=6,cur=10
>     d=6
>   6 ≤ 5? NO → l=15
> 
> l=15, r=15 exit.
> 
> ✅ Answer: 15
> ```

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

**LC #410** · **Hard**

Split `nums` into `m` non-empty subarrays. Minimize the **maximum subarray sum**.

### Approach

Same as P13. `can(maxSum) = "can we split into ≤ m parts each ≤ maxSum"`.

> [!info]- 🔍 Dry Run: nums=[7,2,5,10,8], m=2
> ```text
> Setup:
>   l = max(nums) = 10
>   r = sum(nums) = 32
> 
> ─────────────────────────────────────────
> Iter 1:
>   mid cap = 21
>   Simulate: parts=1, cur=0
>     +7=7, +2=9, +5=14, +10=24>21 → parts=2, cur=10
>     +8=18 ✓
>     parts = 2
>   2 ≤ m=2? YES → r=21
> 
> Iter 2:
>   mid = 15
>   Simulate:
>     +7=7, +2=9, +5=14, +10=24>15 → parts=2, cur=10
>     +8=18>15 → parts=3, cur=8
>     parts=3
>   3 ≤ 2? NO → l=16
> 
> Iter 3:
>   mid = 18
>   Simulate:
>     +7=7, +2=9, +5=14, +10=24>18 → parts=2, cur=10
>     +8=18 ✓
>     parts=2
>   2 ≤ 2? YES → r=18
> 
> Iter 4:
>   mid = 17
>   Simulate:
>     7+2+5=14, +10=24>17 → parts=2, cur=10
>     10+8=18>17 → parts=3, cur=8
>     parts=3
>   3 ≤ 2? NO → l=18
> 
> l=18, r=18 exit.
> 
> ✅ Answer: 18
>   (Split: [7,2,5,...] and [10,8] giving sums 14 and 18 → max 18)
>   Actually let's verify: cap=18 simulation said parts=[7+2+5+? no... 7+2+5=14, +10=24 > 18 → first part = [7,2,5] sum=14; second part starts with 10, +8=18 sum=18.] ✓
> ```

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
