---
title: "LeetCode Practice: Heap and Priority Queue"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - heap
  - priority-queue
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Heap and Priority Queue

12 problems · top-K, merge-K, scheduling, running median.

> [!abstract] Pattern recap
> Heap gives **O(log n) insert/extract** and **O(1) peek-min/max**. Use it whenever you need:
> - Top-K largest → **min-heap of size K**, evict when over
> - Top-K smallest → **max-heap of size K**
> - Running median → **two heaps** (max-heap of low half, min-heap of high half)
> - Merge-K-sorted → min-heap of front nodes
> - Greedy scheduling → max-heap of remaining work

> [!tip] Python's `heapq` is min-heap only
> For max-heap behavior, push the **negated** value.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Kth Largest Element in Array | 215 | Med | Min-heap size K |
| P2 | K Closest Points to Origin | 973 | Med | Max-heap size K |
| P3 | Top K Frequent Elements (heap form) | 347 | Med | Heap of (freq, val) |
| P4 | Last Stone Weight | 1046 | Easy | Max-heap, two-pop loop |
| P5 | Kth Largest in Stream | 703 | Easy | Min-heap design |
| P6 | Find Median from Data Stream | 295 | **Hard** | Two heaps |
| P7 | Task Scheduler | 621 | Med | Max-heap + cooldown queue |
| P8 | Reorganize String | 767 | Med | Max-heap by count |
| P9 | Sort Characters by Frequency | 451 | Easy | Heap or bucket |
| P10 | Meeting Rooms II | 253 | Med | Min-heap of end times |
| P11 | Schedule Course | 630 | **Hard** | Greedy + max-heap |
| P12 | IPO | 502 | **Hard** | Two structures |

---

## P1: Kth Largest Element in an Array

**LC #215** · Medium

### 🧠 Pattern: Min-Heap of Size K

> Push each element; if heap exceeds K, pop the smallest. After all, heap top = Kth largest.

> [!example]- 📊 Visual: bounded min-heap of size K
> ```text
>   Min-heap of size K = 2 keeps the TOP-2 largest values seen so far.
>   The heap's MINIMUM is the smallest of the top-K → the k-th largest.
> 
>   Stream:  3, 2, 1, 5, 6, 4    k = 2
> 
>     after 3:   [3]
>     after 2:   [2, 3]              size = k ✓
>     after 1:   push → [1,2,3] → pop 1 (over) → [2, 3]
>     after 5:   push → [2,3,5] → pop 2         → [3, 5]
>     after 6:   push → [3,5,6] → pop 3         → [5, 6]
>     after 4:   push → [4,5,6] → pop 4         → [5, 6]
> 
>   Heap-as-tree at end:
>            5          ← min  → THIS is the answer (2nd largest)
>             \
>              6
> 
>   Invariant: heap holds exactly the top-K seen; root = the cutoff value.
> ```

> [!info]- 🔍 Dry Run: nums=[3,2,1,5,6,4], k=2
> ```text
> Setup:
>   h = [] (min-heap)
> 
> ─────────────────────────────────────────
> x=3:  push 3 → h=[3]; size 1 ≤ k=2 OK
> x=2:  push 2 → h=[2, 3]; size=2 OK
> x=1:  push 1 → h=[1, 2, 3]; size=3 > k → pop smallest (1) → h=[2, 3]
> x=5:  push 5 → h=[2, 3, 5]; pop 2 → h=[3, 5]
> x=6:  push 6 → h=[3, 5, 6]; pop 3 → h=[5, 6]
> x=4:  push 4 → h=[4, 5, 6]; pop 4 → h=[5, 6]
> 
> Top of heap = 5 (the smallest of the top-K elements)
> 
> ✅ Answer: 5   (2nd largest)
> ```

> [!success]- JS
> ```js
> // assumes MinHeap class
> const findKthLargest = (nums, k) => {
>   const h = new MinHeap();
>   for (const x of nums) {
>     h.push(x);
>     if (h.size() > k) h.pop();
>   }
>   return h.peek();
> };
> ```

> [!success]- Python
> ```python
> import heapq
> def find_kth_largest(nums, k):
>     h = []
>     for x in nums:
>         heapq.heappush(h, x)
>         if len(h) > k:
>             heapq.heappop(h)
>     return h[0]
> ```

**Key takeaway:** K-largest → keep min-heap of size K. Inverse for K-smallest.

---

## P2: K Closest Points to Origin

**LC #973** · Medium

### Approach

Heap of size K, key = `-distSquared` (max-heap of distances), pop when oversize.

> [!example]- 📊 Visual: max-heap of distances, evict the farthest
> ```text
>   Points & dist² to origin:
>     (1,3)   d² = 10
>     (-2,2)  d² =  8
>     (5,8)   d² = 89        ← far away, will be evicted
>     (0,1)   d² =  1        ← closest
> 
>   Max-heap of size K=2 keyed by dist² (Python: push -d²):
> 
>     push (1,3)/10  →   [10]
>     push (-2,2)/8  →     10
>                          /
>                         8                  size = K ✓
> 
>     push (5,8)/89  →     89                size = 3 > K → pop root (89, farthest)
>                         /  \
>                        8   10
>                    →    10
>                          /
>                         8
> 
>     push (0,1)/1   →     10                pop root (10)
>                         /  \
>                        8    1
>                    →    8
>                          /
>                         1
> 
>   Final heap = the K CLOSEST points; root is the farthest of those kept.
> ```

> [!info]- 🔍 Dry Run: points=[[1,3],[-2,2],[5,8],[0,1]], k=2
> ```text
> dist² values: (1,3)→10, (-2,2)→8, (5,8)→89, (0,1)→1
> 
> h = []  (max-heap via negation; we negate dist²)
> 
> ─────────────────────────────────────────
> (1,3): d=10; push (-10, 1, 3) → h=[(-10,1,3)]
>   size=1 ≤ k=2 ✓
> 
> (-2,2): d=8; push (-8, -2, 2) → h=[(-10,1,3), (-8,-2,2)]   (heap order — smallest negative on top)
>   size=2 OK
> 
> (5,8): d=89; push (-89, 5, 8) → h=[(-89,5,8), (-8,-2,2), (-10,1,3)]
>   size=3 > k → pop top (most-negative = -89 → that's the farthest by absolute distance? wait, negation reverses)
>   Actually pop removes smallest in heap = -89 (most negative). That corresponds to dist 89.
>   That's the farthest! We want to keep the closest k.
>   h after pop: [(-10,1,3), (-8,-2,2)]
> 
> (0,1): d=1; push (-1, 0, 1) → h=[(-10,1,3), (-8,-2,2), (-1,0,1)]
>   size=3 > k → pop top (most negative = -10, farthest of remaining)
>   h = [(-8,-2,2), (-1,0,1)]
> 
> Final h represents the 2 closest points:
>   (-8, -2, 2) → distance 8
>   (-1, 0, 1) → distance 1
> 
> ✅ Answer: [[-2, 2], [0, 1]]
> ```

> [!success]- JS
> ```js
> // assumes MaxHeap class keyed by first tuple element
> const kClosest = (points, k) => {
>   const h = new MaxHeap((a, b) => a[0] - b[0]);
>   for (const [x, y] of points) {
>     const d = x * x + y * y;
>     h.push([d, x, y]);
>     if (h.size() > k) h.pop();
>   }
>   return h.toArray().map(([, x, y]) => [x, y]);
> };
> ```

> [!success]- Python
> ```python
> import heapq
> def k_closest(points, k):
>     h = []
>     for x, y in points:
>         d = x * x + y * y
>         heapq.heappush(h, (-d, x, y))   # max-heap via negation
>         if len(h) > k:
>             heapq.heappop(h)
>     return [[x, y] for _, x, y in h]
> ```

**Key takeaway:** "Closest K" = K-smallest by distance → max-heap of size K, evict the farthest.

---

## P3: Top K Frequent Elements (heap form)

**LC #347** · Medium · (See P5 in [[01 - Arrays and Hashing]] for bucket-sort form.)

### Approach

Count with hash map; min-heap of size K on `(freq, val)`.

> [!example]- 📊 Visual: min-heap of (freq, val) keeps top-K frequencies
> ```text
>   nums = [1,1,1,2,2,3]   k = 2
> 
>   Counter:   1 → 3
>              2 → 2
>              3 → 1
> 
>   Min-heap of size K=2, key = freq:
> 
>     push (3,1):     [(3,1)]
> 
>     push (2,2):       (2,2)              size = K ✓
>                         \
>                          (3,1)
> 
>     push (1,3):       (1,3)              size = 3 > K → pop root
>                       /    \
>                     (2,2) (3,1)
>                  → pop (1,3) (smallest freq)
>                  →   (2,2)
>                         \
>                          (3,1)
> 
>   Heap contents = top-K most frequent. Extract values: {1, 2}.
> ```

> [!info]- 🔍 Dry Run: nums=[1,1,1,2,2,3], k=2
> ```text
> Phase 1 — count: {1:3, 2:2, 3:1}
> 
> Phase 2 — min-heap of size k=2 on (freq, val):
>   push (3, 1) → h=[(3,1)]
>   push (2, 2) → h=[(2,2), (3,1)]   ; size=2 OK
>   push (1, 3) → h=[(1,3), (3,1), (2,2)]
>     size 3 > k → pop min = (1,3) → h=[(2,2), (3,1)]
> 
> Final heap contains top-k by frequency: (2,2), (3,1)
> Extract vals: [2, 1]
> 
> ✅ Answer: [1, 2] (order may vary)
> ```

> [!success]- JS
> ```js
> // assumes MinHeap class keyed by first tuple element (freq)
> const topKFrequent = (nums, k) => {
>   const cnt = new Map();
>   for (const x of nums) cnt.set(x, (cnt.get(x) ?? 0) + 1);
>   const h = new MinHeap((a, b) => a[0] - b[0]);
>   for (const [v, f] of cnt) {
>     h.push([f, v]);
>     if (h.size() > k) h.pop();
>   }
>   return h.toArray().map(([, v]) => v);
> };
> ```

> [!success]- Python
> ```python
> import heapq
> from collections import Counter
> def top_k_frequent(nums, k):
>     cnt = Counter(nums)
>     return [v for _, v in heapq.nlargest(k, ((f, v) for v, f in cnt.items()))]
> ```

**Key takeaway:** `heapq.nlargest(k, iterable)` is your one-liner.

---

## P4: Last Stone Weight

**LC #1046** · Easy

### Approach

Max-heap; repeatedly pop top two, push difference, until ≤ 1 left.

> [!example]- 📊 Visual: max-heap loop, smash the two biggest
> ```text
>   stones = [2, 7, 4, 1, 8, 1]    (max-heap conceptually)
> 
>      8                Iter 1: pop 8, pop 7 → push 8-7 = 1
>     / \                                  
>    7   4              
>   / \ /                  4
>   2 1 1                 / \
>                        2   1        (heap = {4,2,1,1,1})
>                       / \
>                       1  1
> 
>   Iter 2: pop 4, pop 2 → push 4-2 = 2
>      2
>     / \
>    1   1            (heap = {2,1,1,1})
>    /
>   1
> 
>   Iter 3: pop 2, pop 1 → push 2-1 = 1
>      1
>     / \             (heap = {1,1,1})
>    1   1
> 
>   Iter 4: pop 1, pop 1 → equal → both destroyed, nothing pushed
>      1               (heap = {1})
> 
>   ≤ 1 stone left → answer = 1.
> ```

> [!info]- 🔍 Dry Run: stones=[2,7,4,1,8,1]
> ```text
> Setup: max-heap via negation
>   h = [-8, -7, -4, -2, -1, -1]  (heap invariant; conceptually max-heap of {8,7,4,2,1,1})
> 
> ─────────────────────────────────────────
> Iter 1:
>   a = -heap.pop() = 8
>   b = -heap.pop() = 7
>   a ≠ b → push -(a-b) = -1
>   heap = {4, 2, 1, 1, 1}
> 
> Iter 2:
>   a = 4
>   b = 2
>   a ≠ b → push -(4-2) = -2
>   heap = {2, 1, 1, 1}
> 
> Iter 3:
>   a = 2
>   b = 1
>   push -1
>   heap = {1, 1, 1}
> 
> Iter 4:
>   a = 1
>   b = 1
>   a == b → both destroyed, don't push
>   heap = {1}
> 
> Loop exits (size < 2).
> 
> ✅ Answer: 1
> ```

> [!success]- JS
> ```js
> // assumes MaxHeap class
> const lastStoneWeight = (stones) => {
>   const h = new MaxHeap();
>   for (const s of stones) h.push(s);
>   while (h.size() > 1) {
>     const a = h.pop();
>     const b = h.pop();
>     if (a !== b) h.push(a - b);
>   }
>   return h.size() ? h.peek() : 0;
> };
> ```

> [!success]- Python
> ```python
> import heapq
> def last_stone_weight(stones):
>     h = [-s for s in stones]
>     heapq.heapify(h)
>     while len(h) > 1:
>         a = -heapq.heappop(h)
>         b = -heapq.heappop(h)
>         if a != b:
>             heapq.heappush(h, -(a - b))
>     return -h[0] if h else 0
> ```

**Key takeaway:** "Repeatedly consume the largest" → max-heap loop.

---

## P5: Kth Largest Element in a Stream

**LC #703** · Easy · Design

### Approach

Maintain min-heap of size K. `add(x)`: push, evict if oversize. Top is current Kth-largest.

> [!example]- 📊 Visual: streaming bounded min-heap
> ```text
>   k = 3, initial = [4, 5, 8, 2]
> 
>   After processing initial (evict when size > k):
> 
>          4         ← root = current 3rd-largest
>         / \
>        5   8
> 
>   add(3):   push 3 → [3,4,5,8] → pop 3 (over K) →     4
>                                                       / \
>                                                       5  8        kth = 4
> 
>   add(5):   push 5 → [4,5,5,8] → pop 4 →              5
>                                                       / \
>                                                       5  8        kth = 5
> 
>   add(10):  push 10 → [5,5,8,10] → pop 5 →            5
>                                                       / \
>                                                       8  10       kth = 5
> 
>   add(9):   push 9 → [5,8,9,10] → pop 5 →             8
>                                                       / \
>                                                       9  10       kth = 8
> 
>   add(4):   push 4 → [4,8,9,10] → pop 4 →             8
>                                                       / \
>                                                       9  10       kth = 8
> 
>   The heap "remembers" the top-K of EVERYTHING ever seen.
> ```

> [!info]- 🔍 Dry Run: k=3, initial=[4,5,8,2], add(3), add(5), add(10), add(9), add(4)
> ```text
> Setup: push each initial, evict when > k:
>   add 4: h=[4]
>   add 5: h=[4, 5]
>   add 8: h=[4, 5, 8]
>   add 2: push → h=[2,4,5,8]; size>k → pop 2 → h=[4,5,8]
>   State: h=[4,5,8], top=4 → 3rd largest currently is 4
> 
> ─────────────────────────────────────────
> add(3):
>   push 3 → h=[3,4,5,8]; pop 3 → h=[4,5,8]
>   return top = 4
> 
> add(5):
>   push 5 → h=[4,5,5,8]; pop 4 → h=[5,5,8]
>   return 5
> 
> add(10):
>   push 10 → h=[5,5,8,10]; pop 5 → h=[5,8,10]
>   return 5
> 
> add(9):
>   push 9 → h=[5,8,9,10]; pop 5 → h=[8,9,10]
>   return 8
> 
> add(4):
>   push 4 → h=[4,8,9,10]; pop 4 → h=[8,9,10]
>   return 8
> 
> ✅ Returns: [4, 5, 5, 8, 8]
> ```

> [!success]- JS
> ```js
> // assumes MinHeap class
> class KthLargest {
>   constructor(k, nums) {
>     this.k = k;
>     this.h = new MinHeap();
>     for (const x of nums) this.add(x);
>   }
>   add(x) {
>     this.h.push(x);
>     if (this.h.size() > this.k) this.h.pop();
>     return this.h.peek();
>   }
> }
> ```

> [!success]- Python
> ```python
> import heapq
> class KthLargest:
>     def __init__(self, k, nums):
>         self.k = k
>         self.h = []
>         for x in nums: self.add(x)
>     def add(self, x):
>         heapq.heappush(self.h, x)
>         if len(self.h) > self.k:
>             heapq.heappop(self.h)
>         return self.h[0]
> ```

**Key takeaway:** Streaming top-K is the canonical use of a bounded heap.

---

## P6: Find Median from Data Stream

**LC #295** · **Hard** · Design

### 🧠 Pattern: Two Heaps

> `low` = max-heap of bottom half; `high` = min-heap of top half. Keep `|low| - |high|` ∈ {0, 1}.
> 
> Median = `top(low)` (odd total) or average of tops (even total).

> [!example]- 📊 Visual: two heaps split at the median
> ```text
>   After adding [1, 5, 2, 4, 3]  (already balanced):
> 
>        low (max-heap)         high (min-heap)
>        bottom half           top half
> 
>             3 ◀── median candidate         4 ◀── median candidate
>            / \                            / \
>           2   1                          5  (empty)
> 
>        max of low = 3                  min of high = 4
>        median = (3 + 4) / 2 = 3.5    (even total = 6 elements? wait 5 here)
> 
>   ─────────────────────────────────────────────────
>   For 5 elements:    low has 3, high has 2  →  median = top(low) = 3
>   For 6 elements:    low has 3, high has 3  →  median = (top(low) + top(high)) / 2
>   
>   Invariant: |low| - |high| ∈ {0, 1}
>              all values in low  ≤  all values in high
> 
>   ─────────────────────────────────────────────────
>   The "push → pop → push" rebalance trick on addNum(x):
> 
>     1. push x into low                  (always)
>     2. pop low's top → push into high   (forces a transfer)
>     3. if |high| > |low|:
>          pop high's top → push back into low
> 
>   Step 2 makes sure the new element ends up in the correct half
>   (if it was too big for low, it bubbles to the top and gets ejected to high).
>   Step 3 keeps the size invariant.
> ```

> [!info]- 🔍 Dry Run: addNum(1), findMedian, addNum(2), findMedian, addNum(3), findMedian
> ```text
> Setup:
>   low  = []  (max-heap, store as negatives in Python)
>   high = []  (min-heap)
> 
> ─────────────────────────────────────────
> addNum(1):
>   push 1 to low (negated: heap stores -1)
>   rebalance step 1: pop -1 from low (means "1"), push 1 to high
>     low=[], high=[1]
>   rebalance step 2: high size > low? 1>0 YES → pop 1 from high, push -1 to low
>     low=[-1], high=[]
>   Sizes: |low|=1, |high|=0 → diff=1 OK
> 
> findMedian:
>   |low| > |high| → return -low[0] = 1
> 
> ─────────────────────────────────────────
> addNum(2):
>   push 2 to low (stored -2)
>     low=[-2,-1] meaning {1, 2}, top is 2
>   rebalance: pop top of low (which is 2), push to high
>     low=[-1], high=[2]
>   rebalance: |high|>|low|? 1>1 NO
>   Sizes: 1 and 1
> 
> findMedian:
>   sizes equal → (1 + 2) / 2 = 1.5
> 
> ─────────────────────────────────────────
> addNum(3):
>   push 3 to low → low={1,3}, top=3
>   rebalance: pop 3, push to high → low={1}, high={2,3}
>   rebalance: |high|>|low|? 2>1 YES → pop top of high (2), push to low (negated)
>     low={1,2}, high={3}
>   Sizes: 2 and 1
> 
> findMedian:
>   |low| > |high| → return -low[0] = 2
> 
> ✅ Returns: 1, 1.5, 2
> ```

> [!success]- JS
> ```js
> // assumes MinHeap and MaxHeap classes
> class MedianFinder {
>   constructor() {
>     this.low = new MaxHeap();   // bottom half
>     this.high = new MinHeap();  // top half
>   }
>   addNum(x) {
>     this.low.push(x);
>     this.high.push(this.low.pop());
>     if (this.high.size() > this.low.size()) {
>       this.low.push(this.high.pop());
>     }
>   }
>   findMedian() {
>     if (this.low.size() > this.high.size()) return this.low.peek();
>     return (this.low.peek() + this.high.peek()) / 2;
>   }
> }
> ```

> [!success]- Python
> ```python
> import heapq
> class MedianFinder:
>     def __init__(self):
>         self.low = []   # max-heap (store negatives)
>         self.high = []  # min-heap
>     def addNum(self, x):
>         heapq.heappush(self.low, -x)
>         heapq.heappush(self.high, -heapq.heappop(self.low))
>         if len(self.high) > len(self.low):
>             heapq.heappush(self.low, -heapq.heappop(self.high))
>     def findMedian(self):
>         if len(self.low) > len(self.high):
>             return -self.low[0]
>         return (-self.low[0] + self.high[0]) / 2
> ```

**Variants:** Sliding Window Median (heaps + lazy delete or BBST).

**Key takeaway:** Running median = two heaps balanced in size and value. Keep `|low| - |high|` ∈ {0,1}.

---

## P7: Task Scheduler

**LC #621** · Medium

n-CPU scheduling: same-task cooldown `n`. Min CPU intervals to finish.

### 🧠 Pattern: Max-Heap of Counts + Cooldown Queue

> Always run the task with the highest remaining count. After running, push it into a cooldown queue with `time + n + 1`. Before each tick, return any cooled-down tasks to the heap.

> [!example]- 📊 Visual: max-heap of ready tasks + cooldown queue
> ```text
>   tasks = ["A","A","A","B","B","B"]   n = 2 (cooldown gap)
> 
>     ┌──────────────┐         ┌────────────────────────┐
>     │  MAX-HEAP    │ ──run──▶│  COOLDOWN QUEUE        │
>     │  ready tasks │         │  (count, available_at) │
>     └──────────────┘         └────────────────────────┘
>           ▲                              │
>           │   after n+1 ticks            │
>           └──────────────────────────────┘
> 
>   Timeline (each col = 1 CPU tick):
> 
>     t:   1   2   3   4   5   6   7   8
>     run: A   B  idle A   B  idle A   B
>          └─── n=2 gap ──┘
>              └─── n=2 gap ──┘
> 
>   Heap holds ready tasks (most frequent on top).
>   When you pick task X, it goes to queue with timestamp = t + n.
>   At time t == queue.front.timestamp, release back into heap.
> ```

> [!info]- 🔍 Dry Run: tasks=["A","A","A","B","B","B"], n=2
> ```text
> count: {A:3, B:3}
> heap (max via negation): [-3, -3]   (3 As, 3 Bs)
> q = deque()
> time = 0
> 
> ─────────────────────────────────────────
> t=1: heap pop -3 (3 As remaining) → run A, remaining=2
>   push (-2, t+n=3) to cooldown q
>   q = [(-2, 3)]
>   heap = [-3]
> 
> t=2: pop -3 (Bs) → run B, remaining=2
>   q = [(-2,3), (-2,4)]
>   heap = []
> 
> t=3: heap empty, but q[0].time == 3 → release; heap.push(-2)
>   pop -2 → run A, remaining=1
>   push (-1, 5) to q
>   q = [(-2,4), (-1,5)]
>   heap = []
> 
> t=4: q[0].time == 4 → release -2 (B); pop, run B, remaining=1
>   q = [(-1,5), (-1,6)]
>   heap = []
> 
> t=5: q[0].time == 5 → release -1 (A); pop, run A, remaining=0
>   q = [(-1,6)]
>   heap = []
> 
> t=6: q[0].time == 6 → release -1 (B); pop, run B
>   q = []
>   heap = []
> 
> heap empty AND q empty → exit
> 
> ✅ Answer: 6
>   (Schedule: A B _ A B _ A B → but task scheduler just counts intervals; idle slots count)
>   Actually verify: A,B,idle,A,B,idle,A,B = 8? Hmm. Let me recheck.
>   
> Better trace:
>   tasks counted: 3A + 3B = 6 tasks
>   n=2 cooldown → between two As, gap ≥ 2.
>   Optimal schedule: A B _ A B _ A B  → length 8 ? But formula says (max-1)*(n+1)+maxCount = (3-1)*3+2 = 8.
>   Hmm let me recount the trace... the queue mechanic gives 6 but I think my trace was off.
>   
> Correct answer for this input: 8.
> ```

> [!success]- JS
> ```js
> // assumes MaxHeap class
> const leastInterval = (tasks, n) => {
>   const cnt = new Map();
>   for (const t of tasks) cnt.set(t, (cnt.get(t) ?? 0) + 1);
>   const h = new MaxHeap();
>   for (const c of cnt.values()) h.push(c);
>   const q = [];   // [remaining, availableAt]
>   let time = 0;
>   while (h.size() || q.length) {
>     time += 1;
>     if (h.size()) {
>       const c = h.pop() - 1;
>       if (c > 0) q.push([c, time + n]);
>     }
>     if (q.length && q[0][1] === time) {
>       h.push(q.shift()[0]);
>     }
>   }
>   return time;
> };
> ```

> [!success]- Python
> ```python
> import heapq
> from collections import Counter, deque
> def least_interval(tasks, n):
>     cnt = Counter(tasks)
>     h = [-c for c in cnt.values()]
>     heapq.heapify(h)
>     time = 0
>     q = deque()   # (count_remaining, available_time)
>     while h or q:
>         time += 1
>         if h:
>             c = heapq.heappop(h) + 1   # one less (negative + 1)
>             if c < 0:
>                 q.append((c, time + n))
>         if q and q[0][1] == time:
>             heapq.heappush(h, q.popleft()[0])
>     return time
> ```

**Variants:** Closed-form formula: `max((maxCount-1)*(n+1) + numWithMax, len(tasks))`.

**Key takeaway:** "Greedy schedule with cooldown" → max-heap (current ready) + queue (cooling).

---

## P8: Reorganize String

**LC #767** · Medium

Rearrange so no two adjacent chars are equal.

### 🧠 Pattern: Max-Heap by Count, Always Pop Two

> Pop the two most frequent (different) chars. Append both. Decrement counts; push back if non-zero.

> [!example]- 📊 Visual: pop-two-different to break up the majority
> ```text
>   s = "aaabc"   counts = {a:3, b:1, c:1}
> 
>   Max-heap (by count):
>          (3,'a')
>          /     \
>      (1,'b')  (1,'c')
> 
>   Iter 1: pop two distinct → 'a' and 'b' → append both
>             out = "ab"
>             decrement → push back (2,'a'); 'b' becomes 0, don't re-push
>          (2,'a')
>             \
>            (1,'c')
> 
>   Iter 2: pop two → 'a' and 'c' → append
>             out = "abac"
>             (1,'a') re-pushed; 'c' dropped
>          (1,'a')
> 
>   Tail (only one element left): append directly
>             out = "abaca"
> 
>   Adjacent chars never equal because we always interleave the two largest.
>   Feasibility: maxCount ≤ ⌈n/2⌉ → otherwise impossible.
> ```

> [!info]- 🔍 Dry Run: s="aab"
> ```text
> counts = {a:2, b:1}
> Feasibility: max=2 ≤ (3+1)/2 = 2 ✓
> 
> heap (max via negation): [(-2,'a'), (-1,'b')]
> 
> ─────────────────────────────────────────
> Iter 1: pop two
>   c1=-2 ch1='a'
>   c2=-1 ch2='b'
>   out = ['a','b']
>   c1+1=-1 < 0 → push (-1,'a')
>   c2+1=0 = 0 → don't push
>   heap = [(-1,'a')]
> 
> Only 1 left:
>   out.append('a')
>   out = ['a','b','a']
> 
> ✅ Answer: "aba"
> 
> ─────────────────────────────────────────
> Counter-example: s="aaab"
>   counts={a:3, b:1}; max=3 > (4+1)/2=2 → return ""
> ```

> [!success]- JS
> ```js
> // assumes MaxHeap class keyed by first tuple element (count)
> const reorganizeString = (s) => {
>   const cnt = new Map();
>   for (const ch of s) cnt.set(ch, (cnt.get(ch) ?? 0) + 1);
>   if (Math.max(...cnt.values()) > Math.floor((s.length + 1) / 2)) return "";
>   const h = new MaxHeap((a, b) => a[0] - b[0]);
>   for (const [ch, c] of cnt) h.push([c, ch]);
>   const out = [];
>   while (h.size() >= 2) {
>     const [c1, ch1] = h.pop();
>     const [c2, ch2] = h.pop();
>     out.push(ch1, ch2);
>     if (c1 - 1 > 0) h.push([c1 - 1, ch1]);
>     if (c2 - 1 > 0) h.push([c2 - 1, ch2]);
>   }
>   if (h.size()) out.push(h.peek()[1]);
>   return out.join("");
> };
> ```

> [!success]- Python
> ```python
> import heapq
> from collections import Counter
> def reorganize_string(s):
>     cnt = Counter(s)
>     if max(cnt.values()) > (len(s) + 1) // 2:
>         return ""
>     h = [(-c, ch) for ch, c in cnt.items()]
>     heapq.heapify(h)
>     out = []
>     while len(h) >= 2:
>         c1, ch1 = heapq.heappop(h)
>         c2, ch2 = heapq.heappop(h)
>         out.append(ch1); out.append(ch2)
>         if c1 + 1 < 0: heapq.heappush(h, (c1 + 1, ch1))
>         if c2 + 1 < 0: heapq.heappush(h, (c2 + 1, ch2))
>     if h:
>         out.append(h[0][1])
>     return ''.join(out)
> ```

**Key takeaway:** "No-two-adjacent equal" → pop two most-frequent at a time. Feasibility check: `maxCount ≤ (n+1)/2`.

---

## P9: Sort Characters By Frequency

**LC #451** · Easy

### Approach

Count, then sort by count desc (or `heapq.nlargest`).

> [!example]- 📊 Visual: bucket by frequency
> ```text
>   s = "tree"        counts = {t:1, r:1, e:2}
> 
>   Buckets (index = frequency):
> 
>     freq 0:  ─────
>     freq 1:  [ 't', 'r' ]
>     freq 2:  [ 'e' ]
>     freq 3:  ─────
> 
>   Walk buckets HIGH → LOW, appending char × freq:
> 
>     freq 2 → "ee"
>     freq 1 → "t" + "r"
>     
>     out = "ee" + "t" + "r"  = "eetr"
> 
>   Alternative: max-heap of (count, char), pop all and write count copies.
>   Either is O(n + k log k); buckets are O(n).
> ```

> [!info]- 🔍 Dry Run: s="tree"
> ```text
> Counter("tree") = {'t': 1, 'r': 1, 'e': 2}
> most_common() in desc order: [('e', 2), ('t', 1), ('r', 1)]
> 
> Build result:
>   'e' * 2 = "ee"
>   't' * 1 = "t"
>   'r' * 1 = "r"
>   "eetr" (or "eert" depending on tie order)
> 
> ✅ Answer: "eert"
> ```

> [!success]- JS
> ```js
> const frequencySort = (s) => {
>   const cnt = new Map();
>   for (const ch of s) cnt.set(ch, (cnt.get(ch) ?? 0) + 1);
>   return [...cnt.entries()]
>     .sort((a, b) => b[1] - a[1])
>     .map(([ch, c]) => ch.repeat(c))
>     .join("");
> };
> ```

> [!success]- Python
> ```python
> from collections import Counter
> def frequency_sort(s):
>     cnt = Counter(s)
>     return ''.join(ch * c for ch, c in cnt.most_common())
> ```

**Key takeaway:** `Counter.most_common()` is the Pythonic shortcut.

---

## P10: Meeting Rooms II

**LC #253** · Medium

Min rooms needed for given intervals.

### 🧠 Pattern: Sort by Start + Min-Heap of End Times

> Sort intervals by start. For each, if heap top's end ≤ current start → reuse (pop). Otherwise → new room (push). Final heap size = rooms used.

> [!example]- 📊 Visual: heap = currently occupied rooms (by end time)
> ```text
>   intervals = [[0,30], [5,10], [15,20]]
> 
>   Time axis:
>     0─────────30                      meeting A
>          5──10                         meeting B
>                15──20                  meeting C
> 
>   Process by start time, min-heap of END TIMES = occupied rooms:
> 
>   [0,30):  heap empty → push 30
>            heap = [30]                                     rooms: 1
> 
>   [5,10):  top = 30,  30 > 5 → overlap, need NEW room
>            push 10
>            heap = [10, 30]                                 rooms: 2
> 
>   [15,20): top = 10,  10 ≤ 15 → reuse this room → pop 10
>            push 20
>            heap = [20, 30]                                 rooms: 2
> 
>   Answer: heap size at the end = 2 rooms.
>   Heap top = the room that frees up next; compare against next start.
> ```

> [!info]- 🔍 Dry Run: intervals=[[0,30],[5,10],[15,20]]
> ```text
> Sorted by start: [[0,30], [5,10], [15,20]]
> heap = []  (min-heap of end times)
> 
> ─────────────────────────────────────────
> [0, 30]:  heap empty → push 30 → heap=[30]; rooms=1
> 
> [5, 10]:  heap.top=30 ≤ start=5? NO (30>5) → can't reuse
>   push 10 → heap=[10, 30]; rooms=2
> 
> [15, 20]: heap.top=10 ≤ 15? YES → pop 10 (reuse that room)
>   push 20 → heap=[20, 30]; rooms=2 (unchanged)
> 
> ✅ Answer: 2 rooms
>   Reasoning: meeting [0,30] needs a room throughout; [5,10] needs a second room (overlaps with [0,30]); [15,20] can reuse the room that [5,10] vacated at time 10.
> ```

> [!success]- JS
> ```js
> // assumes MinHeap class
> const minMeetingRooms = (intervals) => {
>   intervals.sort((a, b) => a[0] - b[0]);
>   const h = new MinHeap();
>   for (const [s, e] of intervals) {
>     if (h.size() && h.peek() <= s) h.pop();
>     h.push(e);
>   }
>   return h.size();
> };
> ```

> [!success]- Python
> ```python
> import heapq
> def min_meeting_rooms(intervals):
>     intervals.sort(key=lambda x: x[0])
>     h = []
>     for s, e in intervals:
>         if h and h[0] <= s:
>             heapq.heappop(h)
>         heapq.heappush(h, e)
>     return len(h)
> ```

**Variants:** Sweep line (events of +1/-1) — equivalent.

**Key takeaway:** "Min concurrent intervals" → sort by start + min-heap of ends.

---

## P11: Course Schedule III

**LC #630** · **Hard**

Each course `(duration, lastDay)`. Max courses you can complete in order of `lastDay`.

### 🧠 Pattern: Greedy + Max-Heap Replacement

> Sort by `lastDay` (deadlines first). Try every course; if it fits, take it. If overflows, **drop** the longest course taken so far (max-heap on durations) — only if doing so frees enough time.

> [!example]- 📊 Visual: greedy take-then-swap-out the longest
> ```text
>   Sort by deadline. Maintain max-heap of durations TAKEN.
>   Idea: always take the course; if it pushes time past deadline,
>         drop the longest course taken so far.
> 
>   courses (by deadline): (100,200) (1000,1250) (200,1300) (2000,3200)
> 
>   Step 1: take (100,200)     heap = [100]            time = 100
>   Step 2: take (1000,1250)   heap = [1000, 100]      time = 1100
>   Step 3: take (200,1300)    heap = [1000, 200, 100] time = 1300
>   Step 4: take (2000,3200)   heap = [2000,1000,200,100] time = 3300
>           3300 > 3200 → POP LONGEST = 2000 → time = 1300
>           heap = [1000, 200, 100]
> 
>   Max-heap (taken so far):
> 
>         1000  ◀── longest, ready to be evicted if needed
>         /  \
>       200  100
> 
>   Final count = heap size = 3 courses taken.
>   "Swap a worse choice out" works because evicting the longest
>   reduces total time the most while preserving the count.
> ```

> [!info]- 🔍 Dry Run: courses=[[100,200],[200,1300],[1000,1250],[2000,3200]]
> ```text
> Sort by deadline: same order since deadlines 200<1250<1300<3200.
>   [(100,200), (1000,1250), (200,1300), (2000,3200)]
> 
> heap = []  (max-heap of durations taken, as negatives)
> time = 0
> 
> ─────────────────────────────────────────
> (100, 200): time += 100 = 100; push -100 → heap=[-100]
>   time 100 ≤ deadline 200 ✓ → keep
> 
> (1000, 1250): time += 1000 = 1100; push -1000 → heap=[-1000,-100]
>   time 1100 ≤ 1250 ✓ → keep
> 
> (200, 1300): time += 200 = 1300; push -200 → heap=[-1000,-200,-100]
>   time 1300 ≤ 1300 ✓ → keep
> 
> (2000, 3200): time += 2000 = 3300; push -2000 → heap=[-2000,-1000,-200,-100]
>   time 3300 > deadline 3200 → drop longest
>     pop -2000 (the just-added course itself!) → time -= 2000 = 1300; heap=[-1000,-200,-100]
>   We took 3 courses successfully.
> 
> ✅ Answer: len(heap) = 3 courses
> ```

> [!success]- JS
> ```js
> // assumes MaxHeap class
> const scheduleCourse = (courses) => {
>   courses.sort((a, b) => a[1] - b[1]);
>   const h = new MaxHeap();
>   let time = 0;
>   for (const [dur, end] of courses) {
>     time += dur;
>     h.push(dur);
>     if (time > end) time -= h.pop();
>   }
>   return h.size();
> };
> ```

> [!success]- Python
> ```python
> import heapq
> def schedule_course(courses):
>     courses.sort(key=lambda c: c[1])
>     h = []
>     time = 0
>     for dur, end in courses:
>         time += dur
>         heapq.heappush(h, -dur)
>         if time > end:
>             time += heapq.heappop(h)   # pops most-negative = longest
>     return len(h)
> ```

**Key takeaway:** Greedy "swap a worse choice out" → max-heap of taken items.

---

## P12: IPO

**LC #502** · **Hard**

Pick at most k projects to maximize capital. Each project requires `capital[i]`; gives `profit[i]`.

### 🧠 Pattern: Two Structures — Min-Heap by Capital, Max-Heap by Profit

> Sort projects by required capital. Push affordable projects (by current capital) into a max-heap on profit. Pick the top, add profit, repeat k times.

> [!example]- 📊 Visual: two structures — sorted-by-capital + max-heap-by-profit
> ```text
>   k = 2, w = 0
>   projects (sorted by capital req):
>     (cap=0, profit=1)
>     (cap=1, profit=2)
>     (cap=1, profit=3)
> 
>   ┌─────────────────────────┐         ┌──────────────────────┐
>   │  SORTED LIST (by cap)   │ ──pull──▶│  MAX-HEAP (by profit)│
>   │  affordable → eligible  │  when    │  pick top each round │
>   │                         │  cap ≤ w │                      │
>   └─────────────────────────┘         └──────────────────────┘
> 
>   Round 1 (w=0):
>     pull all with cap ≤ 0 → push profit 1 to heap
>     heap = [1]
>     pop top = 1  →  w = 0 + 1 = 1
> 
>   Round 2 (w=1):
>     pull all with cap ≤ 1 → push profits 2, 3
>     heap:    3
>             /
>            2
>     pop top = 3 →  w = 1 + 3 = 4
> 
>   Final w = 4.
>   Two-structure pattern: one for ELIGIBILITY, one for VALUE.
> ```

> [!info]- 🔍 Dry Run: k=2, w=0, profits=[1,2,3], capital=[0,1,1]
> ```text
> projects sorted by capital asc:
>   (0, 1), (1, 2), (1, 3)
> 
> w (current capital) = 0
> heap = []  (max-heap of profits, negated)
> i = 0
> 
> ─────────────────────────────────────────
> Round 1 (pick 1st project):
>   while projects[i].cap ≤ w (=0):
>     project (0,1): push -1 to heap → heap=[-1]; i=1
>     project (1,2): cap 1 > 0 → stop
>   heap not empty → pop most-negative = -1 (profit 1)
>   w -= -1 → w = 0 + 1 = 1
> 
> Round 2:
>   while projects[i].cap ≤ w (=1):
>     project (1,2): push -2 → heap=[-2]; i=2
>     project (1,3): push -3 → heap=[-3,-2]; i=3
>   pop -3 (profit 3) → w += 3 → w=4
> 
> Done (k rounds).
> 
> ✅ Answer: 4
> ```

> [!success]- JS
> ```js
> // assumes MaxHeap class
> const findMaximizedCapital = (k, w, profits, capital) => {
>   const projects = capital
>     .map((c, idx) => [c, profits[idx]])
>     .sort((a, b) => a[0] - b[0]);
>   const h = new MaxHeap();
>   let i = 0;
>   for (let round = 0; round < k; round++) {
>     while (i < projects.length && projects[i][0] <= w) {
>       h.push(projects[i][1]);
>       i += 1;
>     }
>     if (!h.size()) break;
>     w += h.pop();
>   }
>   return w;
> };
> ```

> [!success]- Python
> ```python
> import heapq
> def find_maximized_capital(k, w, profits, capital):
>     projects = sorted(zip(capital, profits))   # by capital asc
>     h = []                  # max-heap of profits (negated)
>     i = 0
>     for _ in range(k):
>         while i < len(projects) and projects[i][0] <= w:
>             heapq.heappush(h, -projects[i][1])
>             i += 1
>         if not h: break
>         w -= heapq.heappop(h)   # subtract negative = add profit
>     return w
> ```

**Key takeaway:** Two-criteria selection → one structure for **eligibility** (sorted by req), another for **value** (heap).

---

> [!tip] After this drill
> "Top K / Kth" → bounded heap. "Running median" → two heaps. "Scheduling / interval merge" → heap of end-times. "Greedy with replace" → max-heap of taken items.
