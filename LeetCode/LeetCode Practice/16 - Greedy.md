---
title: "LeetCode Practice: Greedy"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - greedy
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Greedy

10 problems · pick locally-optimal choice that leads to a globally-optimal solution.

> [!abstract] Pattern recap
> Greedy works when the problem has the **greedy choice property**: a local optimal choice leads to a global optimal solution. Proving greediness requires either:
> - **Exchange argument** — swapping to greedy can't hurt
> - **Cut-and-paste** — replace any optimal solution's step with greedy's, no worse
> - **Matroid theory** — rare in interviews
> 
> In an interview: state your greedy rule, sketch why it works, code it. Don't over-prove.

> [!warning] Greedy ≠ first guess
> Many "obvious" greedy approaches FAIL. Always check: does locally-best break a future-needed option? E.g., Coin Change with `[1, 3, 4]` target 6: greedy 4+1+1=3 coins; optimal 3+3=2.

> [!tip] When greedy DOES work
> - Interval scheduling: sort by END time
> - Activity selection: pick earliest-finish-first
> - Jump game: track furthest reachable
> - Gas station: skip stations that net negative
> - Huffman coding: pop two smallest

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Jump Game | 55 | Med | Track furthest reach |
| P2 | Jump Game II | 45 | Med | BFS-like greedy |
| P3 | Maximum Subarray | 53 | Med | Kadane = greedy view |
| P4 | Gas Station | 134 | Med | Net + earliest restart |
| P5 | Hand of Straights | 846 | Med | Sorted multiset |
| P6 | Merge Triplets to Form Target | 1899 | Med | Filter + componentwise max |
| P7 | Partition Labels | 763 | Med | Last-occurrence window |
| P8 | Valid Parenthesis String | 678 | Med | Two counters (min/max open) |
| P9 | Best Time to Buy/Sell Stock II | 122 | Easy | Sum positive diffs |
| P10 | Boats to Save People | 881 | Med | Two pointers + greedy |

---

## P1: Jump Game

**LC #55** · Medium

Can you reach the last index? `nums[i]` = max jump length.

### 🧠 Pattern: Track Furthest Reachable

> Walk left to right. At each `i`, if `i > furthest_reachable`, return false. Else update `furthest = max(furthest, i + nums[i])`.

> [!success]- Python
> ```python
> def can_jump(nums):
>     furthest = 0
>     for i, x in enumerate(nums):
>         if i > furthest: return False
>         furthest = max(furthest, i + x)
>     return True
> ```

**Key takeaway:** Reachability problems → "can I still get here?" greedy. Single-pass O(n).

---

## P2: Jump Game II

**LC #45** · Medium

Min number of jumps to reach the end.

### 🧠 Pattern: BFS-like Levels

> Treat each "jump range" as a level. Within current level, scan to find `farthest`. When you exhaust current level → increment jumps, move to next level.

> [!success]- Python
> ```python
> def jump(nums):
>     jumps = 0
>     cur_end = 0
>     farthest = 0
>     for i in range(len(nums) - 1):
>         farthest = max(farthest, i + nums[i])
>         if i == cur_end:
>             jumps += 1
>             cur_end = farthest
>     return jumps
> ```

**Key takeaway:** BFS in disguise — each "expansion of reach" is a level.

---

## P3: Maximum Subarray (Greedy View)

**LC #53** · Medium

(Also covered in [[01 - Arrays and Hashing]] P12 as Kadane.)

### Greedy framing

> If `cur < 0`, discard — it can only hurt future sums. Start fresh.

> [!success]- Python
> ```python
> def max_subarray(nums):
>     cur = best = nums[0]
>     for x in nums[1:]:
>         cur = x if cur < 0 else cur + x
>         best = max(best, cur)
>     return best
> ```

**Key takeaway:** Kadane is greedy: "negative running sum is dead weight, restart".

---

## P4: Gas Station

**LC #134** · Medium

Circular route. `gas[i]` and `cost[i]`. Find start index, or -1.

### 🧠 Pattern: Net + Earliest Restart

> If `sum(gas) < sum(cost)`: impossible (-1).
> Else: walk once. When tank goes negative, the answer **cannot** be any index from `start..current` (every earlier start had even more deficit by now). Reset `start = i+1`, `tank = 0`.

> [!success]- Python
> ```python
> def can_complete_circuit(gas, cost):
>     if sum(gas) < sum(cost): return -1
>     start, tank = 0, 0
>     for i in range(len(gas)):
>         tank += gas[i] - cost[i]
>         if tank < 0:
>             start = i + 1
>             tank = 0
>     return start
> ```

**Key takeaway:** "Earliest valid start" = restart after every failure. Linear pass once net is verified.

---

## P5: Hand of Straights

**LC #846** · Medium

Can you partition hand into groups of `groupSize` consecutive integers?

### 🧠 Pattern: Always Start a Group with the Smallest Remaining

> Count freq. Sorted iteration; for each min remaining, try to form a group; decrement counts.

> [!success]- Python
> ```python
> from collections import Counter
> def is_n_straight_hand(hand, groupSize):
>     if len(hand) % groupSize: return False
>     cnt = Counter(hand)
>     for x in sorted(cnt):
>         c = cnt[x]
>         if c == 0: continue
>         for i in range(groupSize):
>             if cnt[x + i] < c: return False
>             cnt[x + i] -= c
>     return True
> ```

**Key takeaway:** "Sorted multiset + greedy commit smallest" — pattern shared with Reorganize String.

---

## P6: Merge Triplets to Form Target Triplet

**LC #1899** · Medium

You can take componentwise max of any 2 triplets. Reach `target = [a, b, c]`?

### 🧠 Pattern: Filter + Compose

> Ignore any triplet that exceeds any component of target. From remaining, check if for each component, at least one triplet equals target's value there.

> [!success]- Python
> ```python
> def merge_triplets(triplets, target):
>     good = [False, False, False]
>     for t in triplets:
>         if any(t[i] > target[i] for i in range(3)): continue
>         for i in range(3):
>             if t[i] == target[i]:
>                 good[i] = True
>     return all(good)
> ```

**Key takeaway:** Filter aggressively (impossible candidates) then check feasibility on what remains.

---

## P7: Partition Labels

**LC #763** · Medium

Partition string so each letter appears in only one part. Return part sizes.

### 🧠 Pattern: Last-Occurrence Window

> Record each char's last index. Walk; extend current window to `max(end, last[ch])`. When `i == end`, cut.

### Trace

```
"ababcbacadefegdehijhklij"
last={ a:8, b:5, c:7, d:14, e:15, f:11, g:13, h:19, i:22, j:23, k:20, l:21 }

i=0 'a' end=max(0,8)=8
i=1 'b' end=max(8,5)=8
i=2 'a' end=8
... walk until i=8: end=8. Cut. Part size 8+1-0=9.
restart from 9...
```

> [!success]- Python
> ```python
> def partition_labels(s):
>     last = {c: i for i, c in enumerate(s)}
>     out = []
>     start = end = 0
>     for i, c in enumerate(s):
>         end = max(end, last[c])
>         if i == end:
>             out.append(end - start + 1)
>             start = i + 1
>     return out
> ```

**Key takeaway:** "Earliest cut that fully contains every letter so far" → expand window to last occurrence.

---

## P8: Valid Parenthesis String

**LC #678** · Medium

`*` may be `(`, `)`, or empty.

### 🧠 Pattern: Track Min and Max Possible Open Count

> `lo` = min open (treat * as `)` if possible). `hi` = max open (treat * as `(`). On `(`: both +1. On `)`: both -1. On `*`: lo-1, hi+1. Clamp `lo` at 0. If `hi < 0`: invalid. End: `lo == 0`.

> [!success]- Python
> ```python
> def check_valid_string(s):
>     lo = hi = 0
>     for c in s:
>         if c == '(': lo += 1; hi += 1
>         elif c == ')': lo -= 1; hi -= 1
>         else: lo -= 1; hi += 1
>         if hi < 0: return False
>         lo = max(lo, 0)
>     return lo == 0
> ```

**Key takeaway:** Wildcards → track range of states. Beautiful, hard to invent under pressure — memorize.

---

## P9: Best Time to Buy and Sell Stock II

**LC #122** · Easy

Multiple transactions allowed.

### 🧠 Pattern: Sum Positive Differences

> Greedy: any positive price diff = "buy yesterday, sell today". Sum them.

> [!success]- Python
> ```python
> def max_profit_ii(prices):
>     return sum(max(0, prices[i] - prices[i-1]) for i in range(1, len(prices)))
> ```

**Key takeaway:** "Many transactions" = greedy on adjacent diffs. Equivalent to picking every up-stretch.

---

## P10: Boats to Save People

**LC #881** · Medium

Each boat: limit weight, max 2 people. Min boats.

### Approach

Sort. Two pointers — lightest + heaviest. If they fit together, pair; else, heaviest goes alone.

> [!success]- Python
> ```python
> def num_rescue_boats(people, limit):
>     people.sort()
>     l, r = 0, len(people) - 1
>     boats = 0
>     while l <= r:
>         if people[l] + people[r] <= limit:
>             l += 1
>         r -= 1
>         boats += 1
>     return boats
> ```

**Key takeaway:** Sort + two pointers = classic greedy pattern when pairs need to be optimized.

---

> [!tip] After this drill
> Recognize: "sort + sweep" → 80% of greedy problems. The hard part is **proving** the greedy rule works — practice articulating the exchange argument.
