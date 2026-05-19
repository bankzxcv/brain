---
title: "LeetCode Practice: Intervals"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - intervals
  - sweep-line
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Intervals

10 problems · merge, insert, overlap, scheduling.

> [!abstract] Pattern recap
> Interval problems have 3 reliable techniques:
> 1. **Sort by start** (then sweep merging overlaps)
> 2. **Sort by end** (then greedy pick non-overlapping)
> 3. **Sweep line** — convert intervals to start/end events, sort, scan
> 
> **Overlap test**: `[a, b]` and `[c, d]` overlap iff `max(a, c) < min(b, d)`.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Merge Intervals | 56 | Med | Sort by start, sweep |
| P2 | Insert Interval | 57 | Med | Three-phase merge |
| P3 | Non-overlapping Intervals | 435 | Med | Sort by end, greedy |
| P4 | Meeting Rooms | 252 | Easy | Sort + adjacent overlap |
| P5 | Meeting Rooms II | 253 | Med | Min-heap of ends |
| P6 | Min Arrows to Burst Balloons | 452 | Med | Sort by end, greedy |
| P7 | Interval List Intersections | 986 | Med | Two pointers |
| P8 | Car Pooling | 1094 | Med | Sweep line / prefix sum |
| P9 | Employee Free Time | 759 | **Hard** | Heap or merge intervals |
| P10 | Max Events You Can Attend | 1353 | Med | Heap of ends |

---

## P1: Merge Intervals

**LC #56** · Medium

> [!example]- 📊 Visual: timeline merge
> ```text
>   Input intervals (after sort by start):
> 
>    1───3
>      2─────6
>                    8──10
>                                15─────18
> 
>   Sweep left-to-right; if current overlaps top of result, extend top:
> 
>   Step 1: take [1,3]:                   [1───3]
>   Step 2: [2,6] starts at 2 ≤ 3 →       [1─────6]    (merged, extend end to 6)
>   Step 3: [8,10] starts at 8 > 6 →      [1─────6] [8─10]   (new segment)
>   Step 4: [15,18] starts at 15 > 10 →   [1─────6] [8─10] [15───18]
> 
>   Final timeline:
> 
>    ▓▓▓▓▓▓     ▓▓▓        ▓▓▓▓
>    1    6     8 10       15 18
> 
>   Result = [[1, 6], [8, 10], [15, 18]]
> 
>   Overlap test (for half-closed convention):
>     [a, b] and [c, d] overlap iff  c ≤ b  (sorted by start, so a ≤ c)
> ```

> [!info]- 🔍 Dry Run: intervals=[[1,3],[2,6],[8,10],[15,18]]
> ```text
> Sort by start: already sorted.
> 
> out = [[1, 3]]
> 
> ─────────────────────────────────────────
> Process [2, 6]:
>   2 ≤ out[-1].end (3)? YES → merge
>   out[-1].end = max(3, 6) = 6
>   out = [[1, 6]]
> 
> Process [8, 10]:
>   8 ≤ 6? NO → start new
>   out = [[1, 6], [8, 10]]
> 
> Process [15, 18]:
>   15 ≤ 10? NO → start new
>   out = [[1, 6], [8, 10], [15, 18]]
> 
> ✅ Answer: [[1, 6], [8, 10], [15, 18]]
> ```

> [!success]- Python
> ```python
> def merge(intervals):
>     intervals.sort(key=lambda x: x[0])
>     out = [intervals[0]]
>     for s, e in intervals[1:]:
>         if s <= out[-1][1]:
>             out[-1][1] = max(out[-1][1], e)
>         else:
>             out.append([s, e])
>     return out
> ```

**Key takeaway:** Sort by start. Merge is the canonical interval skeleton.

---

## P2: Insert Interval

**LC #57** · Medium · intervals pre-sorted

> [!info]- 🔍 Dry Run: intervals=[[1,3],[6,9]], new=[2,5]
> ```text
> Phase 1 — append all intervals ending BEFORE new starts:
>   i=0, [1,3]: 3 < 2? NO → stop
>   out = []
> 
> Phase 2 — merge all overlapping with new:
>   i=0, [1,3]: 1 ≤ new.end=5? YES → merge
>     new.start = min(2, 1) = 1
>     new.end = max(5, 3) = 5
>     i++
>   i=1, [6,9]: 6 ≤ 5? NO → stop
>   out.append([1, 5])
> 
> Phase 3 — append remaining:
>   i=1, [6,9]: append
>   out = [[1, 5], [6, 9]]
> 
> ✅ Answer: [[1, 5], [6, 9]]
> 
> ─────────────────────────────────────────
> Example: intervals=[[1,2],[3,5],[6,7],[8,10],[12,16]], new=[4,8]
>   Phase 1: [1,2] (2 < 4) → append. out=[[1,2]]. Then [3,5]: 5<4 NO → stop.
>   Phase 2: [3,5] overlaps → merge new = [min(4,3), max(8,5)] = [3,8]. i=2.
>            [6,7]: 6 ≤ 8 → merge new=[3, max(8,7)]=[3,8]. i=3.
>            [8,10]: 8 ≤ 8 → merge new=[3, max(8,10)]=[3,10]. i=4.
>            [12,16]: 12 ≤ 10? NO → stop.
>   out.append([3,10]).
>   Phase 3: append [12,16].
>   Result: [[1,2],[3,10],[12,16]]
> ```

> [!success]- Python
> ```python
> def insert(intervals, new):
>     out, i, n = [], 0, len(intervals)
>     while i < n and intervals[i][1] < new[0]:
>         out.append(intervals[i]); i += 1
>     while i < n and intervals[i][0] <= new[1]:
>         new[0] = min(new[0], intervals[i][0])
>         new[1] = max(new[1], intervals[i][1])
>         i += 1
>     out.append(new)
>     while i < n:
>         out.append(intervals[i]); i += 1
>     return out
> ```

**Key takeaway:** Pre-sorted? Do **one** linear pass with three explicit phases. No re-sort needed.

---

## P3: Non-overlapping Intervals

**LC #435** · Medium

> [!info]- 🔍 Dry Run: intervals=[[1,2],[2,3],[3,4],[1,3]]
> ```text
> Sort by END: [[1,2],[2,3],[1,3],[3,4]]
> 
> end = -∞, removed = 0
> 
> [1,2]: start=1 ≥ -∞ → keep. end = 2
> [2,3]: start=2 ≥ 2 → keep. end = 3
> [1,3]: start=1 ≥ 3? NO → remove. removed++ = 1
> [3,4]: start=3 ≥ 3 → keep. end = 4
> 
> ✅ Answer: 1   (remove [1,3] to leave [[1,2],[2,3],[3,4]] non-overlapping)
> ```

> [!success]- Python
> ```python
> def erase_overlap_intervals(intervals):
>     intervals.sort(key=lambda x: x[1])
>     removed, end = 0, float('-inf')
>     for s, e in intervals:
>         if s >= end:
>             end = e
>         else:
>             removed += 1
>     return removed
> ```

**Key takeaway:** "Max non-overlapping" / "Min to remove" → sort by end, greedy keep.

---

## P4: Meeting Rooms

**LC #252** · Easy

> [!info]- 🔍 Dry Run: intervals=[[0,30],[5,10],[15,20]]
> ```text
> Sort by start: [[0,30],[5,10],[15,20]]
> 
> i=1: intervals[1].start (5) < intervals[0].end (30)? YES → overlap → return false
> 
> ✅ Answer: false
> 
> Example with no overlap: [[7,10],[2,4]]
>   sort: [[2,4],[7,10]]
>   i=1: 7 < 4? NO → continue
>   loop ends → true
> ```

> [!success]- Python
> ```python
> def can_attend_meetings(intervals):
>     intervals.sort(key=lambda x: x[0])
>     for i in range(1, len(intervals)):
>         if intervals[i][0] < intervals[i-1][1]:
>             return False
>     return True
> ```

**Key takeaway:** Sorted + adjacent overlap check. The simplest interval problem.

---

## P5: Meeting Rooms II

**LC #253** · Medium

> [!example]- 📊 Visual: meeting timeline
> ```text
>   intervals = [[0,30], [5,10], [15,20]]
> 
>   Timeline (concurrent overlap = rooms needed):
> 
>     [0────────────────30]      ← Room A
>          [5─10]                ← Room B (overlaps A)
>                  [15──20]      ← reuses Room B
> 
>     Time:   0   5   10  15  20      30
>     Rooms:  1   2   1   2   1       0
>                 ↑           
>             peak = 2 rooms needed
> 
>   Sweep approach: two sorted arrays of events
> 
>     starts: 0 ─── 5 ────────── 15
>     ends:                  10 ────── 20 ────── 30
> 
>     At each "start" event:
>       if start ≥ earliest pending end → reuse (advance ends pointer)
>       else                            → new room
> 
>   Min heap variant: heap holds end times of rooms currently in use.
>     When a new meeting starts: top.end ≤ start → pop (release), then push new end.
> ```

> [!info]- 🔍 Dry Run: intervals=[[0,30],[5,10],[15,20]]
> ```text
> starts sorted = [0, 5, 15]
> ends sorted = [10, 20, 30]
> 
> j = 0, rooms = 0, best = 0
> 
> s=0: starts[0]=0 ≥ ends[j=0]=10? NO → rooms++=1. best=1
> s=5: 5 ≥ 10? NO → rooms++=2. best=2
> s=15: 15 ≥ 10? YES → j++. rooms stays at 2.
>   (We reuse the room that ended at 10.)
>   
> ✅ Answer: 2
> ```

> [!success]- Python
> ```python
> def min_meeting_rooms(intervals):
>     starts = sorted(i[0] for i in intervals)
>     ends = sorted(i[1] for i in intervals)
>     rooms = best = 0
>     j = 0
>     for s in starts:
>         if s >= ends[j]:
>             j += 1
>         else:
>             rooms += 1
>         best = max(best, rooms)
>     return best
> ```

**Key takeaway:** Min rooms = max concurrent meetings. Two sorted arrays of events = sweep line.

---

## P6: Minimum Number of Arrows to Burst Balloons

**LC #452** · Medium

> [!info]- 🔍 Dry Run: points=[[10,16],[2,8],[1,6],[7,12]]
> ```text
> Sort by end: [[1,6],[2,8],[7,12],[10,16]]
> 
> arrows = 1; end = points[0][1] = 6
> 
> [2,8]: start=2 > end=6? NO → covered by same arrow at x=6
> [7,12]: 7 > 6? YES → need new arrow; arrows=2; end=12
> [10,16]: 10 > 12? NO → covered
> 
> ✅ Answer: 2
>   Arrow 1 at x=6 hits balloons [1,6] and [2,8].
>   Arrow 2 at x=12 hits [7,12] and [10,16].
> ```

> [!success]- Python
> ```python
> def find_min_arrow_shots(points):
>     points.sort(key=lambda x: x[1])
>     arrows = 1
>     end = points[0][1]
>     for s, e in points[1:]:
>         if s > end:
>             arrows += 1
>             end = e
>     return arrows
> ```

**Key takeaway:** Same skeleton as P3. "Min cover / max non-overlap" — sort by end.

---

## P7: Interval List Intersections

**LC #986** · Medium

> [!info]- 🔍 Dry Run: A=[[0,2],[5,10],[13,23],[24,25]], B=[[1,5],[8,12],[15,24],[25,26]]
> ```text
> i=0, j=0:
>   lo = max(0, 1) = 1
>   hi = min(2, 5) = 2
>   1 ≤ 2 → out.append([1, 2])
>   A[0].end=2 < B[0].end=5 → i++
> 
> i=1, j=0:
>   lo = max(5, 1) = 5
>   hi = min(10, 5) = 5
>   5 ≤ 5 → out.append([5, 5])
>   A[1].end=10 > B[0].end=5 → j++
> 
> i=1, j=1:
>   lo = max(5, 8) = 8
>   hi = min(10, 12) = 10
>   8 ≤ 10 → out.append([8, 10])
>   A[1].end=10 < B[1].end=12 → i++
> 
> i=2, j=1:
>   lo = max(13, 8) = 13
>   hi = min(23, 12) = 12
>   13 ≤ 12? NO → no intersection
>   A[2].end=23 > B[1].end=12 → j++
> 
> i=2, j=2:
>   lo = max(13, 15) = 15
>   hi = min(23, 24) = 23
>   15 ≤ 23 → out.append([15, 23])
>   A[2].end=23 < B[2].end=24 → i++
> 
> i=3, j=2:
>   lo = max(24, 15) = 24
>   hi = min(25, 24) = 24
>   out.append([24, 24])
>   A[3].end=25 > B[2].end=24 → j++
> 
> i=3, j=3:
>   lo = max(24, 25) = 25
>   hi = min(25, 26) = 25
>   out.append([25, 25])
>   A[3].end=25 < B[3].end=26 → i++  → i out of bounds, exit
> 
> ✅ Answer: [[1,2], [5,5], [8,10], [15,23], [24,24], [25,25]]
> ```

> [!success]- Python
> ```python
> def interval_intersection(A, B):
>     out, i, j = [], 0, 0
>     while i < len(A) and j < len(B):
>         lo = max(A[i][0], B[j][0])
>         hi = min(A[i][1], B[j][1])
>         if lo <= hi:
>             out.append([lo, hi])
>         if A[i][1] < B[j][1]: i += 1
>         else: j += 1
>     return out
> ```

**Key takeaway:** Two sorted, non-overlapping lists → merge-style two pointers.

---

## P8: Car Pooling

**LC #1094** · Medium

> [!info]- 🔍 Dry Run: trips=[[2,1,5],[3,3,7]], capacity=4
> ```text
> diff array (size 1001): all zeros
> 
> trip [p=2, from=1, to=5]:
>   diff[1] += 2 → diff[1] = 2
>   diff[5] -= 2 → diff[5] = -2
> 
> trip [p=3, from=3, to=7]:
>   diff[3] += 3 → 3
>   diff[7] -= 3 → -3
> 
> diff (relevant): index 1=2, 3=3, 5=-2, 7=-3
> 
> Sweep:
>   cur = 0
>   i=0: cur=0
>   i=1: cur=2 ≤ 4 OK
>   i=2: cur=2
>   i=3: cur=5 > 4 → return false
> 
> ✅ Answer: false
>   At time 3 we'd have 2+3=5 passengers, exceeds capacity 4.
> ```

> [!success]- Python
> ```python
> def car_pooling(trips, capacity):
>     diff = [0] * 1001
>     for p, f, t in trips:
>         diff[f] += p
>         diff[t] -= p
>     cur = 0
>     for x in diff:
>         cur += x
>         if cur > capacity: return False
>     return True
> ```

**Key takeaway:** "Concurrent at point T" → difference array (bounded) or sweep with sorted events.

---

## P9: Employee Free Time

**LC #759** · **Hard**

> [!info]- 🔍 Dry Run: schedule=[[(1,2),(5,6)], [(1,3)], [(4,10)]]
> ```text
> Flatten and sort: [(1,2), (1,3), (4,10), (5,6)]
> 
> Merge:
>   merged = [(1,2)]
>   (1,3): 1 ≤ 2 (merged top end) → merge: top.end = max(2,3) = 3
>     merged = [(1,3)]
>   (4,10): 4 ≤ 3? NO → append
>     merged = [(1,3), (4,10)]
>   (5,6): 5 ≤ 10? YES → merge: top.end = max(10, 6) = 10
>     merged = [(1,3), (4,10)]
> 
> Free time = gaps between consecutive merged intervals:
>   gap = (merged[0].end, merged[1].start) = (3, 4) → free interval [3, 4]
> 
> ✅ Answer: [[3, 4]]
> ```

> [!success]- Python
> ```python
> def employee_free_time(schedule):
>     all_intervals = sorted([iv for emp in schedule for iv in emp], key=lambda x: x.start)
>     merged = [all_intervals[0]]
>     for iv in all_intervals[1:]:
>         if iv.start <= merged[-1].end:
>             merged[-1].end = max(merged[-1].end, iv.end)
>         else:
>             merged.append(iv)
>     return [Interval(merged[i].end, merged[i+1].start) for i in range(len(merged) - 1)]
> ```

**Key takeaway:** Free time = "negative space" of merged busy intervals.

---

## P10: Maximum Number of Events You Can Attend

**LC #1353** · Medium

> [!info]- 🔍 Dry Run: events=[[1,2],[2,3],[3,4]]
> ```text
> Sort by start: [[1,2],[2,3],[3,4]]
> 
> i=0, heap = [], day=0, attended=0
> 
> Loop while i < n or heap:
>   heap empty? YES; day = events[0].start = 1
> 
>   Add all events with start ≤ day=1: events[0]=[1,2] → push end=2; i=1
>   heap = [2]
>   Pop expired (end < day=1): heap[0]=2 ≥ 1 → keep
>   Attend: pop 2 → attended=1; day=2
> 
>   Add events with start ≤ 2: events[1]=[2,3] → push 3; i=2
>   heap = [3]
>   Pop expired: 3 ≥ 2 → keep
>   Attend: pop 3 → attended=2; day=3
> 
>   Add events with start ≤ 3: events[2]=[3,4] → push 4; i=3
>   heap = [4]
>   Pop expired: 4 ≥ 3 → keep
>   Attend: pop 4 → attended=3; day=4
> 
>   Loop exit (i=3=n, heap empty)
> 
> ✅ Answer: 3
> ```

> [!success]- Python
> ```python
> import heapq
> def max_events(events):
>     events.sort()
>     i = 0
>     h = []
>     attended = 0
>     day = 0
>     while i < len(events) or h:
>         if not h:
>             day = events[i][0]
>         while i < len(events) and events[i][0] <= day:
>             heapq.heappush(h, events[i][1])
>             i += 1
>         while h and h[0] < day:
>             heapq.heappop(h)
>         if h:
>             heapq.heappop(h)
>             attended += 1
>             day += 1
>     return attended
> ```

**Key takeaway:** "Schedule one per day, deadline-driven" → min-heap on deadline, advance day.

---

> [!tip] After this drill
> Three knobs: sort by start (merge) vs sort by end (greedy keep) vs sweep line (concurrent count). Knob choice = problem flavor.
