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
| P7 | Trapping Rain Water | 42 | Hard | Opposite + max-so-far |
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

### Trace

```
"A man, a plan, a canal: Panama"
l=0 'A'  r=29 'a' → equal (lowercase)
l=1 ' '  skip → l=2
l=2 'm'  r=28 'm' → equal
... (continue)
return true
```

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

### Trace

```
"abca"
l=0 'a' r=3 'a' ✓  l=1 r=2
l=1 'b' r=2 'c' ✗ → try check(l+1..r) "ca"? no. check(l..r-1) "ab"? no.
                  Actually we need check(2,2)="c" palindrome ✓ → true
```

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

### Trace

```
nums=[2,7,11,15] target=9
l=0 r=3  2+15=17 > 9   r--
l=0 r=2  2+11=13 > 9   r--
l=0 r=1  2+7=9         return [1,2]
```

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

### Trace

```
nums=[-1,0,1,2,-1,-4] → sort → [-4,-1,-1,0,1,2]
i=0 -4   l=1 r=5  -4+(-1)+2=-3<0 l++
       l=2 r=5  -4+(-1)+2=-3<0 l++
       ... no match
i=1 -1   l=2 r=5  -1+(-1)+2=0 ✓ → [-1,-1,2]  l++ r-- (skip dup)
       l=3 r=4  -1+0+1=0 ✓ → [-1,0,1]  done
i=2 -1   skip (duplicate)
i=3 0    l=4 r=5  0+1+2=3>0 r--  done
return [[-1,-1,2],[-1,0,1]]
```

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

### Trace

```
h=[1,8,6,2,5,4,8,3,7]
l=0 r=8  min(1,7)*8 = 8                move l (shorter)
l=1 r=8  min(8,7)*7 = 49 ✓             move r
l=1 r=7  min(8,3)*6 = 18               move r
l=1 r=6  min(8,8)*5 = 40               move l OR r (pick l)
l=2 r=6  min(6,8)*4 = 24               move l
...
return 49
```

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

**LC #42** · Hard

Total water trapped between bars of varying heights.

**Edge cases:** non-decreasing (zero water) · single peak · n ≤ 2.

### 🧠 Pattern: Opposite Pointers + Max-So-Far on Each Side

> Water above index `i` = `min(maxLeft[i], maxRight[i]) - h[i]`. Two-pointer trick: track `leftMax`, `rightMax` as you go; whichever side's max is smaller, that side **bounds** water and can be settled now.

### Approach Evolution

1. **Brute force** · O(n²). For each i, scan left/right max.
2. **Precompute leftMax[] and rightMax[]** · O(n)/O(n).
3. **Monotonic stack** · O(n)/O(n). Also works (see Monotonic Stack topic).
4. **Two pointers — FINAL** · O(n)/O(1).

### Trace

```
h=[0,1,0,2,1,0,1,3,2,1,2,1]
l=0 r=11  leftMax=0 rightMax=1
  h[l]=0 < h[r]=1 → leftMax becomes max(0,0)=0; water += 0-0=0; l++
l=1 r=11  h[l]=1, leftMax=max(0,1)=1; h[l]≥h[r]? swap focus to right side
  h[r]=1, rightMax=max(1,1)=1; water += rightMax-h[r]=0; r--
... (continue)
total = 6
```

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

### Trace

```
nums=[1,1,2,3,3]
slow=1 (next write position)
fast=1 nums[1]=1 == nums[slow-1]=1 → skip
fast=2 nums[2]=2 != 1 → nums[1]=2; slow=2
fast=3 nums[3]=3 != 2 → nums[2]=3; slow=3
fast=4 nums[4]=3 == 3 → skip
return 3, nums starts [1,2,3,...]
```

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

### Trace

```
nums=[0,1,0,3,12]
fast=0 nums[0]=0 skip
fast=1 nums[1]=1 → nums[0]=1; slow=1
fast=2 nums[2]=0 skip
fast=3 nums[3]=3 → nums[1]=3; slow=2
fast=4 nums[4]=12 → nums[2]=12; slow=3
Fill nums[3..]=0 → [1,3,12,0,0]
```

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

### Trace

```
nums=[2,0,2,1,1,0]
low=0 mid=0 high=5
nums[mid]=2 → swap mid↔high: [0,0,2,1,1,2] high=4 (mid stays!)
nums[mid]=0 → swap low↔mid: [0,0,2,1,1,2] low=1 mid=1
nums[mid]=0 → swap low↔mid (same): [0,0,2,1,1,2] low=2 mid=2
nums[mid]=2 → swap mid↔high: [0,0,1,1,2,2] high=3
nums[mid]=1 → mid++: [0,0,1,1,2,2] mid=3
nums[mid]=1 → mid++: mid=4 > high stop
```

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

### Trace

```
1 → 2 → 3 → 4 → 5 → 2  (cycle back to 2)
slow=1 fast=1
step1: slow=2 fast=3
step2: slow=3 fast=5
step3: slow=4 fast=3
step4: slow=5 fast=5 ← MET → cycle
```

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

### Approach

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
