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
| P4 | Largest Rectangle in Histogram | 84 | **Hard** | NS-prev + NS-next |
| P5 | Maximal Rectangle | 85 | **Hard** | LRH per row |
| P6 | Trapping Rain Water | 42 | **Hard** | Stack of decreasing bars |
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

> [!info]- 🔍 Dry Run: nums1=[4,1,2], nums2=[1,3,4,2]
> ```text
> Phase 1 — Build NGE map by scanning nums2 with a decreasing stack:
>   nge = {}
>   st = []  (decreasing, holds values)
> 
>   x=1: st empty → push
>     st = [1]
> 
>   x=3: st.top=1 < 3 → pop, nge[1]=3
>     st = [] → push 3
>     st = [3]
> 
>   x=4: st.top=3 < 4 → pop, nge[3]=4
>     st = [] → push 4
>     st = [4]
> 
>   x=2: st.top=4 < 2? NO → just push
>     st = [4, 2]
> 
>   Remaining in st: 4 and 2 have no greater to the right → nge[4]=-1, nge[2]=-1
>   nge = {1:3, 3:4, 4:-1, 2:-1}
> 
> ─────────────────────────────────────────
> Phase 2 — Look up answers for nums1:
>   nums1=[4,1,2]
>     nge[4] = -1
>     nge[1] = 3
>     nge[2] = -1
> 
> ✅ Answer: [-1, 3, -1]
> ```

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

> [!info]- 🔍 Dry Run: nums=[1,2,1]
> ```text
> Setup:
>   n = 3, out = [-1, -1, -1]
>   st = []   (indices)
> 
> Iterate i=0..2n-1=5, using nums[i % n]:
> 
> ─────────────────────────────────────────
> i=0  x=nums[0]=1
>   st empty → push 0 (within first n, so we track it)
>   st = [0]
> 
> i=1  x=nums[1]=2
>   nums[st.top=0]=1 < 2 → pop 0, out[0]=2
>   st = [] → push 1
>   st = [1]
> 
> i=2  x=nums[2]=1
>   nums[1]=2 < 1? NO → just push (i<n so push)
>   st = [1, 2]
> 
> i=3  x=nums[3%3=0]=1
>   nums[2]=1 < 1? NO → don't push (i >= n)
>   st = [1, 2]
> 
> i=4  x=nums[4%3=1]=2
>   nums[2]=1 < 2 → pop 2, out[2]=2
>   nums[1]=2 < 2? NO → don't push (i >= n)
>   st = [1]
> 
> i=5  x=nums[5%3=2]=1
>   nums[1]=2 < 1? NO → don't push
> 
> Remaining: index 1 with no greater to its "right" → out[1]=-1 (already)
> 
> ✅ Answer: [2, -1, 2]
> ```

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

> [!example]- 📊 Visual: temperatures + stack
> ```text
>   T = [73, 74, 75, 71, 69, 72, 76, 73]
> 
>   Bars (top axis = days, height = temp):
> 
>   76 │                       █
>   75 │       █              ██
>   74 │     ███             ███
>   73 │   █████              ███  █
>   72 │   █████          █  ███  █
>   71 │   █████      █  ██  ███  █
>   ...
>      └────────────────────────────
>        0   1  2   3   4  5   6   7
> 
>   Decreasing stack of indices waiting for a HOTTER day:
> 
>   day 0 (73):  push       stack = [0]
>   day 1 (74):  74 > T[0]=73 → pop 0, out[0]=1-0=1.  push 1.  stack=[1]
>   day 2 (75):  pop 1, out[1]=2-1=1. push 2. stack=[2]
>   day 3 (71):  71 < 75 → push.  stack=[2,3]
>   day 4 (69):  69 < 71 → push.  stack=[2,3,4]
>   day 5 (72):  72 > 69 pop 4 (out[4]=1)
>                72 > 71 pop 3 (out[3]=2)
>                72 < 75 → push.   stack=[2,5]
>   day 6 (76):  76 > 72 pop 5 (out[5]=1)
>                76 > 75 pop 2 (out[2]=4)
>                push.            stack=[6]
>   day 7 (73):  73 < 76 → push.   stack=[6,7]
> 
>   Remaining stack [6,7] never found a warmer day → out[6]=out[7]=0.
> 
>   Each index pushed/popped at most once → O(n).
> ```

> [!info]- 🔍 Dry Run: T=[73,74,75,71,69,72,76,73]
> ```text
> Setup:
>   out = [0,0,0,0,0,0,0,0]
>   st = []  (indices, decreasing temperature top→bottom)
> 
> ─────────────────────────────────────────
> i=0 T=73:  st empty → push 0
>   st = [0]
> 
> i=1 T=74:  T[st.top=0]=73 < 74 → pop 0, out[0] = 1-0 = 1
>   st = [] → push 1
>   out = [1,0,0,0,0,0,0,0]
> 
> i=2 T=75:  T[1]=74 < 75 → pop 1, out[1] = 2-1 = 1
>   push 2
>   out = [1,1,0,0,0,0,0,0]
> 
> i=3 T=71:  T[2]=75 < 71? NO → push
>   st = [2, 3]
> 
> i=4 T=69:  T[3]=71 < 69? NO → push
>   st = [2, 3, 4]
> 
> i=5 T=72:
>   T[4]=69 < 72 → pop 4, out[4] = 5-4 = 1
>   T[3]=71 < 72 → pop 3, out[3] = 5-3 = 2
>   T[2]=75 < 72? NO → push 5
>   st = [2, 5]
>   out = [1,1,0,2,1,0,0,0]
> 
> i=6 T=76:
>   T[5]=72 < 76 → pop 5, out[5] = 6-5 = 1
>   T[2]=75 < 76 → pop 2, out[2] = 6-2 = 4
>   push 6
>   st = [6]
>   out = [1,1,4,2,1,1,0,0]
> 
> i=7 T=73:  T[6]=76 < 73? NO → push
>   st = [6, 7]
> 
> End. Remaining indices in st (6, 7) never found a hotter day → out stays 0 there.
> 
> ✅ Answer: [1, 1, 4, 2, 1, 1, 0, 0]
> ```

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

**LC #84** · **Hard** · **The canonical mono-stack problem**

Find largest rectangle in histogram of bar heights.

### 🧠 Pattern: For Each Bar, Find Left and Right "Less" Boundaries

> Width of rectangle bounded by bar `i` = `right_less[i] - left_less[i] - 1`. Both computable in O(n) with one increasing stack.

### Approach Evolution

1. **For each pair (l,r), min × width** · O(n²).
2. **For each bar, scan left/right for first lower** · O(n²).
3. **One increasing stack — FINAL** · O(n)/O(n). On pop, current `i` is right boundary, new top is left boundary.

> [!example]- 📊 Visual: histogram + largest rectangle
> ```text
>   heights = [2, 1, 5, 6, 2, 3]
> 
>   6 │          █
>   5 │       ███████
>   4 │       ███████
>   3 │       ███████      █
>   2 │ ███   ███████  ███████
>   1 │ █████ ███████  ███████
>   0 └────────────────────────
>       0  1  2  3  4  5
> 
>   For each bar, ask: how wide can I extend left/right while everyone is ≥ me?
> 
>   Bar 0 (h=2): blocked by bar 1 (h=1<2). Width = 1. Area = 2.
>   Bar 1 (h=1): can extend over the whole array! Width = 6. Area = 6.
>   Bar 2 (h=5): blocked by bar 4 (h=2). Width = 2 (bars 2-3). Area = 10.
>   Bar 3 (h=6): blocked immediately. Width = 1. Area = 6.
>   Bar 4 (h=2): can extend to bar 5 (h=3 ≥ 2) and back to bar 2... but blocked left by h=1.
>                Actually width = 2 (bars 4-5). Area = 4.
>   Bar 5 (h=3): width = 1. Area = 3.
> 
>   Max = 10 (bars 2-3, height 5, width 2):
> 
>   6 │          █
>   5 │       ▓▓▓▓▓▓▓        ← winning rectangle
>   4 │       ▓▓▓▓▓▓▓
>   3 │       ▓▓▓▓▓▓▓      █
>   2 │ ███   ▓▓▓▓▓▓▓  ███████
>   1 │ █████ ▓▓▓▓▓▓▓  ███████
>     └────────────────────────
>       0  1  2  3  4  5
> 
>   Mono stack finds left/right "first smaller" in O(n) total.
> ```

> [!info]- 🔍 Dry Run: heights=[2,1,5,6,2,3]
> ```text
> Setup:
>   Append sentinel 0 at end → h = [2,1,5,6,2,3,0]
>   st = []         (indices with increasing heights)
>   best = 0
> 
> ─────────────────────────────────────────
> i=0 h=2: st empty → push
>   st = [0]
> 
> i=1 h=1: h[st.top=0]=2 > 1 → pop 0
>   popped bar height=2; right boundary i=1; left boundary: st empty → effective left = -1
>   width = i - (-1) - 1 = 1   (or use formula: i if st empty else i - st[-1] - 1)
>   area = 2 * 1 = 2
>   best = max(0, 2) = 2
>   Continue: st empty, push 1
>   st = [1]
> 
> i=2 h=5: 1 < 5 → push
>   st = [1, 2]
> 
> i=3 h=6: 5 < 6 → push
>   st = [1, 2, 3]
> 
> i=4 h=2:
>   h[3]=6 > 2 → pop 3
>     popped height 6; right=4; left = st.top=2 (h=5)
>     width = 4 - 2 - 1 = 1
>     area = 6 * 1 = 6
>     best = 6
>   h[2]=5 > 2 → pop 2
>     popped height 5; right=4; left = st.top=1 (h=1)
>     width = 4 - 1 - 1 = 2
>     area = 5 * 2 = 10  ✓
>     best = 10
>   h[1]=1 > 2? NO → push 4
>   st = [1, 4]
> 
> i=5 h=3: h[4]=2 < 3 → push
>   st = [1, 4, 5]
> 
> i=6 h=0 (sentinel):
>   h[5]=3 > 0 → pop 5
>     height 3; right=6; left=4 (h=2)
>     width = 6 - 4 - 1 = 1; area = 3
>   h[4]=2 > 0 → pop 4
>     height 2; right=6; left=1 (h=1)
>     width = 6 - 1 - 1 = 4; area = 8
>   h[1]=1 > 0 → pop 1
>     height 1; right=6; left = st empty → effective -1
>     width = 6 - (-1) - 1 = 6; area = 6
>   push 6
> 
> ✅ Answer: 10   (the 5,6 pair forms 5*2=10)
> ```

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

**LC #85** · **Hard**

In a binary matrix, find the largest rectangle of all 1s.

### 🧠 Pattern: LRH Applied Row by Row

> Treat each row as the base of a histogram: `heights[c] += 1` if `matrix[r][c]=1` else `0`. Run LRH on the heights each row. Max across rows wins.

> [!info]- 🔍 Dry Run: matrix=[["1","0","1","0","0"],["1","0","1","1","1"],["1","1","1","1","1"],["1","0","0","1","0"]]
> ```text
> Per-row heights (cumulative '1's column-wise, reset on '0'):
> 
> Row 0: ["1","0","1","0","0"]  → heights = [1, 0, 1, 0, 0]
>   LRH([1,0,1,0,0]) → best so far = 1
> 
> Row 1: ["1","0","1","1","1"]  → heights = [2, 0, 2, 1, 1]
>   LRH([2,0,2,1,1]) → best so far = 3   (the rectangle of width 3 height 1 in row 1 starting at col 2)
> 
> Row 2: ["1","1","1","1","1"]  → heights = [3, 1, 3, 2, 2]
>   LRH([3,1,3,2,2]) → 6   (height=2 across cols 2..4: 2*3=6)
>   best = 6 ✓
> 
> Row 3: ["1","0","0","1","0"]  → heights = [4, 0, 0, 3, 0]
>   LRH → 4 (just the lone tall column 0)
>   best = 6 (unchanged)
> 
> ✅ Answer: 6
> ```

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

**LC #42** · **Hard**

(Two-pointer version in [[02 - Two Pointers]] P7. Here, mono-stack approach.)

### 🧠 Pattern: Decreasing Stack, Compute Water on Pop

> Maintain decreasing stack of bar indices. When current `h[i]` > h[top], that bar is **right boundary** for the popped bar. New top is **left boundary**. Water above popped = `(min(h[left], h[i]) - h[popped]) * (i - left - 1)`.

> [!info]- 🔍 Dry Run: h=[0,1,0,2,1,0,1,3,2,1,2,1]
> ```text
> Setup:
>   st = []  (decreasing bar heights top→bottom; holds indices)
>   water = 0
> 
> ─────────────────────────────────────────
> i=0 h=0: st empty → push
>   st=[0]
> 
> i=1 h=1: h[0]=0 < 1 → pop 0
>   After pop, st empty → no left bound → skip (can't trap)
>   push 1
>   st=[1]
> 
> i=2 h=0: h[1]=1 < 0? NO → push 2
>   st=[1, 2]
> 
> i=3 h=2:
>   h[2]=0 < 2 → pop 2
>     left bound = st.top=1, right=3
>     bounded = min(h[1]=1, h[3]=2) - h[2]=0 = 1
>     dist = 3 - 1 - 1 = 1
>     water += 1 * 1 = 1
>   h[1]=1 < 2 → pop 1
>     After pop, st empty → no left → skip
>   push 3
>   st=[3], water=1
> 
> i=4 h=1: h[3]=2 < 1? NO → push 4
>   st=[3, 4]
> 
> i=5 h=0: h[4]=1 < 0? NO → push 5
>   st=[3, 4, 5]
> 
> i=6 h=1:
>   h[5]=0 < 1 → pop 5
>     left=4 (h=1), right=6 (h=1)
>     bounded = min(1, 1) - 0 = 1
>     dist = 6 - 4 - 1 = 1
>     water += 1 * 1 = 1; water=2
>   h[4]=1 < 1? NO → push 6
>   st=[3, 4, 6]
> 
> i=7 h=3:
>   h[6]=1 < 3 → pop 6
>     left=4 (h=1), right=7 (h=3)
>     bounded = min(1, 3) - 1 = 0; skip (no water)
>   h[4]=1 < 3 → pop 4
>     left=3 (h=2), right=7
>     bounded = min(2, 3) - 1 = 1
>     dist = 7 - 3 - 1 = 3
>     water += 1 * 3 = 3; water=5
>   h[3]=2 < 3 → pop 3
>     st empty → skip
>   push 7
>   st=[7], water=5
> 
> i=8 h=2: 3<2? NO → push 8
> i=9 h=1: 2<1? NO → push 9
> i=10 h=2:
>   h[9]=1 < 2 → pop 9
>     left=8 (h=2), right=10 (h=2)
>     bounded = min(2,2)-1=1; dist=10-8-1=1; water += 1; water=6
>   h[8]=2 < 2? NO → push 10
>   st=[7, 8, 10]
> 
> i=11 h=1: 2<1? NO → push 11
> 
> End. Remaining stack: bars with no closing greater on the right → no more water.
> 
> ✅ Answer: 6
> ```

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

> [!info]- 🔍 Dry Run: nums=[3,1,2,4]
> ```text
> For each i, want:
>   left[i]  = #consecutive positions to the LEFT (including i) where nums[i] is the min strictly
>   right[i] = #consecutive positions to the RIGHT (including i) where nums[i] is the min (with tie-break)
>   contribution = nums[i] * left[i] * right[i]
> 
> ─────────────────────────────────────────
> Compute prev-less indices (strict <):
>   i=0 (3): no prev-less → left[0] = 0 - (-1) = 1
>   i=1 (1): no prev strictly less than 1 in {3} → left[1] = 1 - (-1) = 2
>   i=2 (2): prev less is index 1 (val 1) → left[2] = 2 - 1 = 1
>   i=3 (4): prev less is index 2 (val 2) → left[3] = 3 - 2 = 1
> 
> Compute next-less-or-equal indices:
>   i=0 (3): next ≤ is index 1 (val 1) → right[0] = 1 - 0 = 1
>   i=1 (1): no next ≤ → right[1] = n - 1 = 4 - 1 = 3
>   i=2 (2): no next ≤ → right[2] = 4 - 2 = 2
>   i=3 (4): no next ≤ → right[3] = 4 - 3 = 1
> 
> Contributions:
>   i=0: 3 * 1 * 1 = 3
>   i=1: 1 * 2 * 3 = 6
>   i=2: 2 * 1 * 2 = 4
>   i=3: 4 * 1 * 1 = 4
>   Sum = 3+6+4+4 = 17
> 
> ✅ Answer: 17
>   Verify by enumeration:
>     [3]=3, [3,1]=1, [3,1,2]=1, [3,1,2,4]=1, [1]=1, [1,2]=1, [1,2,4]=1, [2]=2, [2,4]=2, [4]=4
>     Sum = 3+1+1+1+1+1+1+2+2+4 = 17 ✓
> ```

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

> [!info]- 🔍 Dry Run: num="1432219", k=3
> ```text
> Setup:
>   st = []   (increasing stack)
>   k = 3
> 
> ─────────────────────────────────────────
> '1':  st empty → push
>   st = ['1']
> 
> '4':  4 > top=1 → push (st stays increasing)
>   st = ['1','4']
> 
> '3':  3 < top=4 AND k>0 → pop '4', k=2
>     st = ['1']
>     3 < top=1? NO → stop popping
>     push '3'
>   st = ['1','3']
> 
> '2':  2 < 3 AND k>0 → pop '3', k=1
>     2 < 1? NO → stop
>     push '2'
>   st = ['1','2']
> 
> '2':  2 < 2? NO (not strictly less) → push
>   st = ['1','2','2']
> 
> '1':  1 < 2 AND k>0 → pop '2', k=0
>     k=0, stop popping
>     push '1'
>   st = ['1','2','1']
> 
> '9':  9 > 1 → push
>   st = ['1','2','1','9']
> 
> k remaining = 0, nothing to trim from end.
> 
> Join: "1219"
> Strip leading zeros: still "1219"
> 
> ✅ Answer: "1219"
> ```

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

> [!info]- 🔍 Dry Run: calls next(100), next(80), next(60), next(70), next(60), next(75), next(85)
> ```text
> Setup: st = []
> 
> ─────────────────────────────────────────
> next(100):
>   span = 1 (today)
>   no smaller-or-equal prices on top
>   push (100, 1) → st = [(100, 1)]
>   return 1
> 
> next(80):
>   span = 1
>   top.price=100 ≤ 80? NO → don't absorb
>   push (80, 1) → st = [(100,1), (80,1)]
>   return 1
> 
> next(60):
>   span = 1
>   top=80 ≤ 60? NO → push
>   st = [(100,1), (80,1), (60,1)]
>   return 1
> 
> next(70):
>   span = 1
>   top=60 ≤ 70? YES → pop, absorb: span += 1 = 2
>   top=80 ≤ 70? NO → stop
>   push (70, 2)
>   st = [(100,1), (80,1), (70,2)]
>   return 2
> 
> next(60):
>   top=70 ≤ 60? NO → push
>   st = [(100,1), (80,1), (70,2), (60,1)]
>   return 1
> 
> next(75):
>   top=60 ≤ 75? YES → pop, span+=1 → span=2
>   top=70 ≤ 75? YES → pop, span+=2 → span=4
>   top=80 ≤ 75? NO → stop
>   push (75, 4)
>   st = [(100,1), (80,1), (75,4)]
>   return 4
> 
> next(85):
>   top=75 ≤ 85 → pop, span+=4 → 5
>   top=80 ≤ 85 → pop, span+=1 → 6
>   top=100 ≤ 85? NO → stop
>   push (85, 6)
>   return 6
> 
> ✅ Returns: [1, 1, 1, 2, 1, 4, 6]
> ```

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

> [!info]- 🔍 Dry Run: target=12, position=[10,8,0,5,3], speed=[2,4,1,1,3]
> ```text
> Compute arrival times for each car (descending by position):
>   pos=10 speed=2: time=(12-10)/2 = 1.0
>   pos=8  speed=4: time=(12-8)/4 = 1.0
>   pos=5  speed=1: time=(12-5)/1 = 7.0
>   pos=3  speed=3: time=(12-3)/3 = 3.0
>   pos=0  speed=1: time=(12-0)/1 = 12.0
> 
> Sorted by position desc → walking in this order:
> 
> ─────────────────────────────────────────
> Car (pos=10, t=1.0):
>   st empty → push 1.0
>   st = [1.0]
> 
> Car (pos=8, t=1.0):
>   st.top = 1.0; t=1.0 > 1.0? NO (not strictly greater)
>   This car catches up to the leader → joins same fleet, don't push.
>   st = [1.0]
> 
> Car (pos=5, t=7.0):
>   st.top = 1.0; 7.0 > 1.0? YES → new fleet (this car is slower)
>   push 7.0
>   st = [1.0, 7.0]
> 
> Car (pos=3, t=3.0):
>   st.top = 7.0; 3.0 > 7.0? NO → catches up to fleet ahead, join
>   st = [1.0, 7.0]
> 
> Car (pos=0, t=12.0):
>   st.top = 7.0; 12.0 > 7.0? YES → new fleet
>   push 12.0
>   st = [1.0, 7.0, 12.0]
> 
> ✅ Answer: len(st) = 3 fleets
> ```

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
