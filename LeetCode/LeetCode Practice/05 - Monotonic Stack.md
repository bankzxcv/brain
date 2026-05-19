---
title: "LeetCode Practice: Monotonic Stack"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - monotonic-stack
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Monotonic Stack

10 problems · the "next greater / smaller / span" pattern.

> [!abstract] Pattern recap
> A **monotonic stack** keeps its contents in **strictly increasing** (or decreasing) order. When pushing, pop violators first. Each element pushed/popped at most once → **O(n) total**.
> 
> ```
> for i in range(n):
>     while stack and nums[stack[-1]] < nums[i]:   # decreasing stack
>         j = stack.pop()
>         next_greater[j] = nums[i]
>     stack.append(i)
> ```

> [!tip] Direction rules of thumb
> **Decreasing** stack → "next greater element"
> **Increasing** stack → "next smaller element"
> Walk **right→left** instead → "previous greater/smaller"

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Next Greater Element I | 496 | Easy | NGE map |
| P2 | Next Greater Element II | 503 | Med | NGE + circular |
| P3 | Daily Temperatures | 739 | Med | NGE → days-until |
| P4 | Largest Rectangle in Histogram | 84 | Hard | NS-prev + NS-next |
| P5 | Maximal Rectangle | 85 | Hard | LRH per row |
| P6 | Trapping Rain Water | 42 | Hard | Stack of decreasing bars |
| P7 | Sum of Subarray Minimums | 907 | Med | Contribution counting |
| P8 | Remove K Digits | 402 | Med | Greedy via inc. stack |
| P9 | Online Stock Span | 901 | Med | NGE-left online |
| P10 | Car Fleet | 853 | Med | Stack of arrival times |

---

## P1: Next Greater Element I

**LC #496** · Easy

For each `x` in `nums1` (subset of `nums2`), find next greater element in `nums2`, or -1.

**Edge cases:** strictly decreasing (all -1) · single element · `x` is the last element.

### 🧠 Pattern: Decreasing Stack

> Walk `nums2`. While stack top < current → pop, record current as its NGE. Push current. Look up answers from a map.

### Trace

```
nums2=[1,3,4,2]
i=0 x=1  st=[]; push → st=[1]
i=1 x=3  pop 1 (1<3) → nge[1]=3; push → st=[3]
i=2 x=4  pop 3 → nge[3]=4; push → st=[4]
i=3 x=2  2<4, push → st=[4,2]
end: nge[4]=-1, nge[2]=-1

nums1=[4,1,2] → [-1,3,2]   (wait nge[1]=3, nge[2]=-1)
Actually: [-1, 3, -1]
```

> [!success]- JS
> ```js
> const nextGreaterElement = (nums1, nums2) => {
>   const nge = new Map();
>   const st = [];
>   for (const x of nums2) {
>     while (st.length && st.at(-1) < x) nge.set(st.pop(), x);
>     st.push(x);
>   }
>   return nums1.map(x => nge.get(x) ?? -1);
> };
> ```

> [!success]- Python
> ```python
> def next_greater_element(nums1, nums2):
>     nge = {}
>     st = []
>     for x in nums2:
>         while st and st[-1] < x:
>             nge[st.pop()] = x
>         st.append(x)
>     return [nge.get(x, -1) for x in nums1]
> ```

**Key takeaway:** NGE in O(n): decreasing stack, pop-and-record on push.

---

## P2: Next Greater Element II

**LC #503** · Medium · **Circular**

Same as P1 but the array wraps.

### 🧠 Pattern: Double the Array (Virtually)

> Iterate `2n` indices, using `i % n`. Stack holds indices. Only set answer the first time (when index in `0..n-1`).

> [!success]- JS
> ```js
> const nextGreaterElements = (nums) => {
>   const n = nums.length;
>   const out = new Array(n).fill(-1);
>   const st = [];
>   for (let i = 0; i < 2 * n; i++) {
>     const x = nums[i % n];
>     while (st.length && nums[st.at(-1)] < x) out[st.pop()] = x;
>     if (i < n) st.push(i);
>   }
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def next_greater_elements(nums):
>     n = len(nums)
>     out = [-1] * n
>     st = []
>     for i in range(2 * n):
>         x = nums[i % n]
>         while st and nums[st[-1]] < x:
>             out[st.pop()] = x
>         if i < n:
>             st.append(i)
>     return out
> ```

**Key takeaway:** Circular array → iterate `2n` with modulo. Push only first half.

---

## P3: Daily Temperatures

**LC #739** · Medium

For each day, how many days until a warmer temperature?

### Approach

NGE on values, return **index differences** instead of values. Push indices.

### Trace

```
T=[73,74,75,71,69,72,76,73]
i=0  st=[0]
i=1  74>73 pop 0 → out[0]=1-0=1.  st=[1]
i=2  75>74 pop 1 → out[1]=1.       st=[2]
i=3  71<75 push.                   st=[2,3]
i=4  69<71 push.                   st=[2,3,4]
i=5  72>69 pop 4 → out[4]=1.  72>71 pop 3 → out[3]=2. 72<75 push. st=[2,5]
i=6  76>72 pop 5 → out[5]=1. 76>75 pop 2 → out[2]=4. push.  st=[6]
i=7  73<76 push.                   st=[6,7]
return [1,1,4,2,1,1,0,0]
```

> [!success]- JS
> ```js
> const dailyTemperatures = (T) => {
>   const out = new Array(T.length).fill(0);
>   const st = [];
>   for (let i = 0; i < T.length; i++) {
>     while (st.length && T[st.at(-1)] < T[i]) {
>       const j = st.pop();
>       out[j] = i - j;
>     }
>     st.push(i);
>   }
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def daily_temperatures(T):
>     out = [0] * len(T)
>     st = []
>     for i, t in enumerate(T):
>         while st and T[st[-1]] < t:
>             j = st.pop()
>             out[j] = i - j
>         st.append(i)
>     return out
> ```

**Key takeaway:** "Days/steps until..." → NGE pattern with index diffs.

---

## P4: Largest Rectangle in Histogram

**LC #84** · Hard · **The canonical mono-stack problem**

Find largest rectangle in histogram of bar heights.

### 🧠 Pattern: For Each Bar, Find Left and Right "Less" Boundaries

> Width of rectangle bounded by bar `i` = `right_less[i] - left_less[i] - 1`. Both computable in O(n) with one increasing stack.

### Approach Evolution

1. **For each pair (l,r), min × width** · O(n²).
2. **For each bar, scan left/right for first lower** · O(n²).
3. **One increasing stack — FINAL** · O(n)/O(n). On pop, current `i` is right boundary, new top is left boundary.

### Trace

```
heights=[2,1,5,6,2,3]   (sentinel 0 appended)
i=0 push 0       st=[0]   (h=2)
i=1 1<2 → pop 0, h=2, width=1-(-1)-1=1, area=2.  push 1.  st=[1]
i=2 5>1 push     st=[1,2]
i=3 6>5 push     st=[1,2,3]
i=4 2<6 → pop 3, h=6, w=4-2-1=1, area=6.
       2<5 → pop 2, h=5, w=4-1-1=2, area=10 ✓
       push 4.  st=[1,4]
i=5 3>2 push     st=[1,4,5]
i=6 0 (sentinel) → pop 5, h=3, w=6-4-1=1, area=3.
                  → pop 4, h=2, w=6-1-1=4, area=8.
                  → pop 1, h=1, w=6-(-1)-1=6, area=6.
best=10
```

> [!success]- JS
> ```js
> const largestRectangleArea = (heights) => {
>   const h = [...heights, 0];
>   const st = [];
>   let best = 0;
>   for (let i = 0; i < h.length; i++) {
>     while (st.length && h[st.at(-1)] > h[i]) {
>       const top = st.pop();
>       const width = st.length === 0 ? i : i - st.at(-1) - 1;
>       best = Math.max(best, h[top] * width);
>     }
>     st.push(i);
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def largest_rectangle_area(heights):
>     h = heights + [0]
>     st, best = [], 0
>     for i, x in enumerate(h):
>         while st and h[st[-1]] > x:
>             top = st.pop()
>             width = i if not st else i - st[-1] - 1
>             best = max(best, h[top] * width)
>         st.append(i)
>     return best
> ```

> [!tip] Sentinel
> Appending `0` forces the stack to flush at the end.

**Variants:** P5 (Maximal Rectangle in 2D) · Sum of Subarray Minimums (P7).

**Key takeaway:** "Largest rect bounded by my height" → mono-stack tells me my left/right "less" neighbors in O(n).

---

## P5: Maximal Rectangle

**LC #85** · Hard

In a binary matrix, find the largest rectangle of all 1s.

### 🧠 Pattern: LRH Applied Row by Row

> Treat each row as the base of a histogram: `heights[c] += 1` if `matrix[r][c]=1` else `0`. Run LRH on the heights each row. Max across rows wins.

> [!success]- JS
> ```js
> const maximalRectangle = (matrix) => {
>   if (!matrix.length) return 0;
>   const cols = matrix[0].length;
>   const heights = new Array(cols).fill(0);
>   let best = 0;
>   for (const row of matrix) {
>     for (let c = 0; c < cols; c++) heights[c] = row[c] === '1' ? heights[c] + 1 : 0;
>     best = Math.max(best, largestRectangleArea(heights));
>   }
>   return best;
> };
> // reuse largestRectangleArea from P4
> ```

> [!success]- Python
> ```python
> def maximal_rectangle(matrix):
>     if not matrix: return 0
>     cols = len(matrix[0])
>     heights = [0] * cols
>     best = 0
>     for row in matrix:
>         for c in range(cols):
>             heights[c] = heights[c] + 1 if row[c] == '1' else 0
>         best = max(best, largest_rectangle_area(heights))
>     return best
> ```

**Key takeaway:** Reduce 2D to 1D LRH per row. Reuse sub-solutions.

---

## P6: Trapping Rain Water — via Monotonic Stack

**LC #42** · Hard

(Two-pointer version in [[02 - Two Pointers]] P7. Here, mono-stack approach.)

### 🧠 Pattern: Decreasing Stack, Compute Water on Pop

> Maintain decreasing stack of bar indices. When current `h[i]` > h[top], that bar is **right boundary** for the popped bar. New top is **left boundary**. Water above popped = `(min(h[left], h[i]) - h[popped]) * (i - left - 1)`.

> [!success]- JS
> ```js
> const trapStack = (h) => {
>   const st = [];
>   let water = 0;
>   for (let i = 0; i < h.length; i++) {
>     while (st.length && h[st.at(-1)] < h[i]) {
>       const top = st.pop();
>       if (!st.length) break;
>       const left = st.at(-1);
>       const dist = i - left - 1;
>       const bounded = Math.min(h[left], h[i]) - h[top];
>       water += dist * bounded;
>     }
>     st.push(i);
>   }
>   return water;
> };
> ```

> [!success]- Python
> ```python
> def trap_stack(h):
>     st, water = [], 0
>     for i, x in enumerate(h):
>         while st and h[st[-1]] < x:
>             top = st.pop()
>             if not st: break
>             left = st[-1]
>             dist = i - left - 1
>             bounded = min(h[left], x) - h[top]
>             water += dist * bounded
>         st.append(i)
>     return water
> ```

**Key takeaway:** Two-pointer is cleaner here, but mono-stack solves the "lower bar bounded between two higher" pattern generically.

---

## P7: Sum of Subarray Minimums

**LC #907** · Medium · **mod 10^9+7**

Sum of `min` over all subarrays.

### 🧠 Pattern: Contribution Counting

> For each `nums[i]`, count subarrays where `nums[i]` is the min. Contribution = `nums[i] * left_count * right_count`. Use mono stacks for previous-less and next-less indices.

> [!tip] Tie-breaking
> Use **strict** `<` on one side and **non-strict** `≤` on the other to avoid double-counting equal values.

> [!success]- JS
> ```js
> const sumSubarrayMins = (nums) => {
>   const MOD = 1e9 + 7;
>   const n = nums.length;
>   const left = new Array(n), right = new Array(n);
>   let st = [];
>   for (let i = 0; i < n; i++) {
>     while (st.length && nums[st.at(-1)] > nums[i]) st.pop();
>     left[i] = st.length ? i - st.at(-1) : i + 1;
>     st.push(i);
>   }
>   st = [];
>   for (let i = n - 1; i >= 0; i--) {
>     while (st.length && nums[st.at(-1)] >= nums[i]) st.pop();
>     right[i] = st.length ? st.at(-1) - i : n - i;
>     st.push(i);
>   }
>   let sum = 0n;
>   for (let i = 0; i < n; i++) sum = (sum + BigInt(nums[i]) * BigInt(left[i]) * BigInt(right[i])) % BigInt(MOD);
>   return Number(sum);
> };
> ```

> [!success]- Python
> ```python
> def sum_subarray_mins(nums):
>     MOD = 10**9 + 7
>     n = len(nums)
>     left, right = [0] * n, [0] * n
>     st = []
>     for i in range(n):
>         while st and nums[st[-1]] > nums[i]: st.pop()
>         left[i] = i - st[-1] if st else i + 1
>         st.append(i)
>     st = []
>     for i in range(n - 1, -1, -1):
>         while st and nums[st[-1]] >= nums[i]: st.pop()
>         right[i] = st[-1] - i if st else n - i
>         st.append(i)
>     return sum(nums[i] * left[i] * right[i] for i in range(n)) % MOD
> ```

**Variants:** Sum of Subarray Ranges (max - min, two contribution problems).

**Key takeaway:** "Sum over all subarrays" → for each element, count when it's min/max — contribution counting via mono-stack.

---

## P8: Remove K Digits

**LC #402** · Medium

Remove exactly k digits from `num` to form smallest possible number.

### 🧠 Pattern: Greedy with Increasing Stack

> To stay smallest, when next digit is **less than** top, the top digit was a poor choice. Pop while possible (decrement k). At end, trim leading zeros.

### Trace

```
num="1432219" k=3
'1' st=[1]
'4' 4>1 push → [1,4]
'3' 3<4 pop 4 (k=2), 3>1 push → [1,3]
'2' 2<3 pop 3 (k=1), 2>1 push → [1,2]
'2' 2>=2 push → [1,2,2]
'1' 1<2 pop 2 (k=0), stop popping. push → [1,2,2,1]
'9' push → [1,2,2,1,9]
trim leading zeros → "1219"
```

> [!success]- JS
> ```js
> const removeKdigits = (num, k) => {
>   const st = [];
>   for (const d of num) {
>     while (k && st.length && st.at(-1) > d) { st.pop(); k--; }
>     st.push(d);
>   }
>   while (k--) st.pop();
>   return st.join('').replace(/^0+/, '') || '0';
> };
> ```

> [!success]- Python
> ```python
> def remove_k_digits(num, k):
>     st = []
>     for d in num:
>         while k and st and st[-1] > d:
>             st.pop(); k -= 1
>         st.append(d)
>     while k:
>         st.pop(); k -= 1
>     return ''.join(st).lstrip('0') or '0'
> ```

**Variants:** Find the Most Competitive Subsequence · Create Maximum Number.

**Key takeaway:** Greedy lex order → increasing stack. Always check "is there still k to spend?".

---

## P9: Online Stock Span

**LC #901** · Medium · **Streaming**

`next(price)` returns the consecutive days (including today) with prices ≤ today.

### 🧠 Pattern: Stack of (price, span)

> When popping, **absorb** the popped span into the current. Stack stays decreasing.

> [!success]- JS
> ```js
> class StockSpanner {
>   constructor() { this.st = []; }
>   next(price) {
>     let span = 1;
>     while (this.st.length && this.st.at(-1)[0] <= price) span += this.st.pop()[1];
>     this.st.push([price, span]);
>     return span;
>   }
> }
> ```

> [!success]- Python
> ```python
> class StockSpanner:
>     def __init__(self): self.st = []
>     def next(self, price):
>         span = 1
>         while self.st and self.st[-1][0] <= price:
>             span += self.st.pop()[1]
>         self.st.append((price, span))
>         return span
> ```

**Key takeaway:** Amortize popped work into pushed entry. Each price pushed/popped once → O(1) amortized.

---

## P10: Car Fleet

**LC #853** · Medium

Cars at positions heading to `target`. A faster car catching a slower one **fleets** with it. Count distinct fleets at arrival.

### 🧠 Pattern: Sort by Position, Stack of Arrival Times

> Sort cars by position **descending** (closer to target first). Compute each car's arrival time. Walk through and maintain a stack of arrival times: a slower (later) car ahead means current car joins that fleet (skip). Stack size = number of fleets.

### Trace

```
target=12  pos=[10,8,0,5,3]  speed=[2,4,1,1,3]
sort by pos desc → 
  (10, time=(12-10)/2=1)
  (8, time=(12-8)/4=1)
  (5, time=(12-5)/1=7)
  (3, time=(12-3)/3=3)
  (0, time=(12-0)/1=12)

times in order: [1,1,7,3,12]
st=[]; push 1
1 ≤ 1 → 8 catches 10, joins fleet (don't push)
push 7
3 < 7 → 3 catches 7's fleet, skip
push 12
return 3 (fleets: {10,8}, {5,3}, {0})
```

> [!success]- JS
> ```js
> const carFleet = (target, position, speed) => {
>   const cars = position.map((p, i) => [p, speed[i]]).sort((a, b) => b[0] - a[0]);
>   const st = [];
>   for (const [p, s] of cars) {
>     const t = (target - p) / s;
>     if (!st.length || t > st.at(-1)) st.push(t);
>   }
>   return st.length;
> };
> ```

> [!success]- Python
> ```python
> def car_fleet(target, position, speed):
>     cars = sorted(zip(position, speed), reverse=True)
>     st = []
>     for p, s in cars:
>         t = (target - p) / s
>         if not st or t > st[-1]:
>             st.append(t)
>     return len(st)
> ```

**Key takeaway:** Sort then sweep. Stack of "lead times" — current can't beat predecessor → joins their fleet.

---

> [!tip] After this drill
> "Next greater/smaller" or "first taller/shorter" → mono stack. "Largest rectangle / max area in array" → mono stack. "Stream span / running max-width" → mono stack with absorption.
