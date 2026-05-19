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
> **Overlap test**: `[a, b]` and `[c, d]` overlap iff `max(a, c) < min(b, d)` (or `≤` if endpoints touching counts).

> [!tip] Inclusive vs exclusive endpoints
> Clarify in the interview. `[1,3]` and `[3,5]` may or may not overlap depending on convention. Use `<=` for closed/closed; `<` for half-open.

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
| P9 | Employee Free Time | 759 | Hard | Heap or merge intervals |
| P10 | Max Events You Can Attend | 1353 | Med | Heap of ends |

---

## P1: Merge Intervals

**LC #56** · Medium

### Approach

Sort by start. Walk: if current.start ≤ prev.end → merge (extend end). Else, push new.

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

### Approach

3 phases: append all that end before newInterval starts; merge all that overlap with newInterval; append the rest.

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

Min intervals to remove so the rest don't overlap.

### 🧠 Pattern: Sort by END, Greedy Earliest-End

> Pick intervals with smallest end first; this leaves the most room. Remove anything that overlaps the last kept.

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

Can a single person attend all meetings (any overlap → no)?

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

**LC #253** · Medium · (Heap version in [[10 - Heap and Priority Queue]] P10)

### Sweep-line alternative

Sort starts and ends separately. Two pointers. Increment rooms on each start; decrement on end if start has passed. Max running.

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

### Approach

Sort by end. Shoot arrow at the smallest end; this destroys all balloons starting before it. Repeat.

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

Both lists pre-sorted and non-overlapping internally.

### 🧠 Pattern: Two Pointers

> Overlap = `[max(a.start, b.start), min(a.end, b.end)]` if max ≤ min. Advance the one with smaller end.

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

Trip = `[passengers, from, to]`. Capacity check.

### 🧠 Pattern: Sweep Line via Difference Array

> Mark `+passengers` at `from`, `-passengers` at `to`. Sweep; running sum must never exceed capacity.

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

> [!tip] Difference array vs heap
> Diff array works when coordinates are bounded integers (`≤ 1000` here). Else use heap of events.

**Key takeaway:** "Concurrent at point T" → difference array (bounded) or sweep with sorted events.

---

## P9: Employee Free Time

**LC #759** · Hard

Given each employee's working intervals, find common free time.

### Approach

Flatten all intervals, sort by start, sweep merging — gaps between merged intervals = free time.

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

**Key takeaway:** Free time = "negative space" of merged busy intervals. Reduce to P1 + take gaps.

---

## P10: Maximum Number of Events You Can Attend

**LC #1353** · Medium

Each event `[start, end]`, attend any one day in range. Each day attend one event.

### 🧠 Pattern: Min-Heap of End Days, Sweep Per Day

> Sort by start. Walk days 1..max_day. Push all events starting today. Pop events whose end < today. Attend the one with smallest end day (it dies soonest).

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
