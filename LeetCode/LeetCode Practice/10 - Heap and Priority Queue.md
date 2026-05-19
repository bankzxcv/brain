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
| P6 | Find Median from Data Stream | 295 | Hard | Two heaps |
| P7 | Task Scheduler | 621 | Med | Max-heap + cooldown queue |
| P8 | Reorganize String | 767 | Med | Max-heap by count |
| P9 | Sort Characters by Frequency | 451 | Easy | Heap or bucket |
| P10 | Meeting Rooms II | 253 | Med | Min-heap of end times |
| P11 | Schedule Course | 630 | Hard | Greedy + max-heap |
| P12 | IPO | 502 | Hard | Two structures |

---

## P1: Kth Largest Element in an Array

**LC #215** · Medium

### 🧠 Pattern: Min-Heap of Size K

> Push each element; if heap exceeds K, pop the smallest. After all, heap top = Kth largest.
> 
> **Why min-heap of size K for "K largest"?** The top of the heap is the smallest of your top K. Anything smaller than top gets discarded. Anything larger evicts top → top advances toward the true K-th largest.

### Approach Evolution

1. **Sort + index n-k** · O(n log n).
2. **Min-heap of size K — FINAL** · O(n log k)/O(k).
3. **Quickselect** · O(n) avg, O(n²) worst — mention but rarely chosen.

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

**LC #295** · Hard · Design

### 🧠 Pattern: Two Heaps

> `low` = max-heap of bottom half; `high` = min-heap of top half. Keep `|low| - |high|` ∈ {0, 1}.
> 
> Median = `top(low)` (odd total) or average of tops (even total).
> 
> **Add(x):**
> 1. Push to `low` (use negation for max-heap).
> 2. Rebalance: move `low`'s max into `high`.
> 3. If `high` is bigger, move `high`'s min back to `low`.

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

> [!tip] Why the "push to low → pop top → push to high" dance?
> It ensures the value lands in the correct half. If it was too big for `low`, it'll bubble out and into `high`.

**Variants:** Sliding Window Median (heaps + lazy delete or BBST).

**Key takeaway:** Running median = two heaps balanced in size and value. Keep `|low| - |high|` ∈ {0,1}.

---

## P7: Task Scheduler

**LC #621** · Medium

n-CPU scheduling: same-task cooldown `n`. Min CPU intervals to finish.

### 🧠 Pattern: Max-Heap of Counts + Cooldown Queue

> Always run the task with the highest remaining count. After running, push it into a cooldown queue with `time + n + 1`. Before each tick, return any cooled-down tasks to the heap.

### Approach

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

**Variants:** Rearrange so no two same within k distance (LC 358).

**Key takeaway:** "No-two-adjacent equal" → pop two most-frequent at a time. Feasibility check: `maxCount ≤ (n+1)/2`.

---

## P9: Sort Characters By Frequency

**LC #451** · Easy

### Approach

Count, then sort by count desc (or `heapq.nlargest`).

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

**LC #630** · Hard

Each course `(duration, lastDay)`. Max courses you can complete in order of `lastDay`.

### 🧠 Pattern: Greedy + Max-Heap Replacement

> Sort by `lastDay` (deadlines first). Try every course; if it fits, take it. If overflows, **drop** the longest course taken so far (max-heap on durations) — only if doing so frees enough time.

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

**LC #502** · Hard

Pick at most k projects to maximize capital. Each project requires `capital[i]`; gives `profit[i]`.

### 🧠 Pattern: Two Structures — Min-Heap by Capital, Max-Heap by Profit

> Sort projects by required capital. Push affordable projects (by current capital) into a max-heap on profit. Pick the top, add profit, repeat k times.

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
