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

11 problems · pick locally-optimal choice that leads to a globally-optimal solution.

> [!abstract] Pattern recap
> Greedy works when the problem has the **greedy choice property**: a local optimal choice leads to a global optimal solution.

> [!warning] Greedy ≠ first guess
> Many "obvious" greedy approaches FAIL. Always check: does locally-best break a future-needed option?

> [!tip] When greedy DOES work
> - Interval scheduling: sort by END time
> - Activity selection: pick earliest-finish-first
> - Jump game: track furthest reachable
> - Gas station: skip stations that net negative

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
| P11 | Candy | 135 | **Hard** | Two-pass greedy |

---

## P1: Jump Game

**LC #55** · Medium

> [!example]- 📊 Visual: furthest reachable index
> ```text
>   nums = [2, 3, 1, 1, 4]    can we reach the end?
>   idx     0  1  2  3  4
> 
>   At each i, track furthest_reach = max(furthest, i + nums[i]):
> 
>     i=0  ⇨   reach max = 0+2 = 2
>          ▼─────▶
>          0 1 2 3 4
> 
>     i=1  ⇨   reach max = max(2, 1+3) = 4
>            ▼───────▶
>          0 1 2 3 4   ← can already reach the end!
> 
>   Stop check: if i > furthest_reach at any step → STUCK → return false.
>   We're fine here because furthest stays ≥ i throughout.
> ```

> [!info]- 🔍 Dry Run: nums=[2,3,1,1,4]
> ```text
> furthest = 0
> 
> i=0, x=2:  i (0) ≤ furthest (0) → still reachable
>            furthest = max(0, 0+2) = 2
> i=1, x=3:  i (1) ≤ furthest (2) → reachable
>            furthest = max(2, 1+3) = 4
> i=2, x=1:  2 ≤ 4 OK; furthest = max(4, 2+1) = 4
> i=3, x=1:  3 ≤ 4 OK; furthest = max(4, 3+1) = 4
> i=4, x=4:  4 ≤ 4 OK; furthest = max(4, 4+4) = 8 (but doesn't matter, already past end)
> 
> Reached end without dropping out.
> 
> ✅ Answer: true
> 
> ─────────────────────────────────────────
> Counter-example: nums=[3,2,1,0,4]
>   i=0 x=3: furthest=3
>   i=1 x=2: 1≤3 OK; furthest=max(3,3)=3
>   i=2 x=1: 2≤3 OK; furthest=max(3,3)=3
>   i=3 x=0: 3≤3 OK; furthest=max(3,3)=3
>   i=4: i=4 > furthest=3 → return false
> ```

> [!success]- Python
> ```python
> def can_jump(nums):
>     furthest = 0
>     for i, x in enumerate(nums):
>         if i > furthest: return False
>         furthest = max(furthest, i + x)
>     return True
> ```

**Key takeaway:** Reachability problems → "can I still get here?" greedy.

---

## P2: Jump Game II

**LC #45** · Medium

> [!example]- 📊 Visual: BFS levels of reach
> ```text
>   nums = [2, 3, 1, 1, 4]    min jumps from idx 0 → last idx?
>   idx     0  1  2  3  4
> 
>   Think of "levels" — how far you can reach with k jumps:
> 
>     Level 0 (0 jumps): {0}
>          [0]
>           └── nums[0]=2 → can reach idx 1, 2
> 
>     Level 1 (1 jump):  {1, 2}     cur_end = 2
>          ┌────┐
>          0  [1 2]
>              ├── nums[1]=3 → reach up to idx 4
>              └── nums[2]=1 → reach up to idx 3
>          farthest = 4
> 
>     Level 2 (2 jumps): {3, 4}     cur_end = 4  ← contains last idx!
>          0  1 2 [3 4]
> 
>   Each time i hits cur_end we "expand" to farthest → jumps++.
>   Answer: 2 jumps   path: 0 → 1 → 4
> ```

> [!info]- 🔍 Dry Run: nums=[2,3,1,1,4]
> ```text
> jumps = 0
> cur_end = 0      ← end of "current level" reach
> farthest = 0     ← end of "next level" reach
> 
> i=0 (don't enter loop iter for last index; loop i in range(n-1)):
>   x=2 → farthest = max(0, 0+2) = 2
>   i == cur_end (0) → jumps++, cur_end = farthest = 2
>   jumps=1
> 
> i=1:
>   x=3 → farthest = max(2, 1+3) = 4
>   i (1) == cur_end (2)? NO
> 
> i=2:
>   x=1 → farthest = max(4, 2+1) = 4
>   i == cur_end (2)? YES → jumps++, cur_end = 4
>   jumps=2
> 
> i=3:
>   x=1 → farthest = max(4, 3+1) = 4
>   i == cur_end? NO
> 
> Loop ends.
> 
> ✅ Answer: 2 jumps   (0 → idx 1 → idx 4)
> ```

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

> [!example]- 📊 Visual: Kadane — restart whenever running sum goes negative
> ```text
>   nums  = [-2,  1, -3,  4, -1,  2,  1, -5,  4]
>   idx     0    1   2   3   4   5   6   7   8
> 
>   Walk left→right tracking cur (best ending HERE) and best (overall max):
> 
>     i=0  cur=-2          best=-2
>     i=1  cur<0 ⇒ RESET   cur =  1     best = 1
>     i=2  extend          cur = -2     best = 1
>     i=3  cur<0 ⇒ RESET   cur =  4     best = 4
>     i=4  extend          cur =  3     best = 4
>     i=5  extend          cur =  5     best = 5
>     i=6  extend          cur =  6     best = 6    ◀ peak
>     i=7  extend          cur =  1     best = 6
>     i=8  extend          cur =  5     best = 6
> 
>   Best subarray: nums[3..6] = [4, -1, 2, 1]   sum = 6
> 
>           ┌──────────────┐
>     -2  1 -3 │ 4 -1  2  1 │ -5  4
>           └──────────────┘
> 
>   Intuition: a negative prefix can never help any future suffix,
>   so drop it and start fresh.
> ```

> [!info]- 🔍 Dry Run: nums=[-2,1,-3,4,-1,2,1,-5,4]
> ```text
> cur = best = -2
> 
> x=1:  cur < 0 → reset to x: cur = 1; best = max(-2, 1) = 1
> x=-3: cur ≥ 0? No actually cur=1 ≥ 0, extend: cur = 1 + (-3) = -2; best = 1
> x=4:  cur < 0 → reset: cur = 4; best = max(1, 4) = 4
> x=-1: cur=4 ≥ 0, extend: cur = 3; best = 4
> x=2:  extend: cur = 5; best = 5
> x=1:  extend: cur = 6; best = 6
> x=-5: extend: cur = 1; best = 6
> x=4:  extend: cur = 5; best = 6
> 
> ✅ Answer: 6
> ```

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

> [!example]- 📊 Visual: net tank along the circuit
> ```text
>   gas  = [1, 2, 3, 4, 5]      cost = [3, 4, 5, 1, 2]
>   diff = [-2,-2,-2, 3, 3]     total = 0 ⇒ a solution exists
> 
>   Cumulative tank if we start at station 0:
> 
>     after 0:  -2  ▼
>     after 1:  -4  ▼▼
>     after 2:  -6  ▼▼▼    ← lowest dip
>     after 3:  -3
>     after 4:   0
> 
>             tank
>            +3 ┤                              ╭──
>             0 ┤────●──────────────────●──────╯
>            -2 ┤────╲────●
>            -4 ┤         ╲────●
>            -6 ┤              ╲──────────────  ← min reached AT end of station 2
>                  0   1    2    3    4
> 
>   Greedy: every time tank goes negative, the start must be AFTER that point.
>   "Earliest restart" → start = i + 1 = 3
> 
>   Verify from 3:   +3, +6, +4, +2, 0  (never goes below 0) ✓
> ```

> [!info]- 🔍 Dry Run: gas=[1,2,3,4,5], cost=[3,4,5,1,2]
> ```text
> Total gas = 15, total cost = 15. Equal → possible.
> 
> start = 0, tank = 0
> 
> i=0: tank += gas[0]-cost[0] = 1-3 = -2 → tank = -2
>      tank < 0 → reset: start = i+1 = 1, tank = 0
> i=1: tank += 2-4 = -2; tank=-2 < 0 → start=2, tank=0
> i=2: tank += 3-5 = -2; start=3, tank=0
> i=3: tank += 4-1 = 3; tank=3
> i=4: tank += 5-2 = 3; tank=6
> 
> Loop ends. total gas == total cost → start=3 is valid.
> 
> ✅ Answer: 3
>   Verify: from station 3: +4-1=3, station 4: 3+5-2=6, station 0: 6+1-3=4, station 1: 4+2-4=2, station 2: 2+3-5=0. Made it.
> ```

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

> [!example]- 📊 Visual: smallest-available anchors consecutive runs
> ```text>
>   hand = [1,2,3,6,2,3,4,7,8]    groupSize = 3
>   Counter:  1×1  2×2  3×2  4×1  6×1  7×1  8×1
> 
>   Sorted keys: 1 2 3 4 6 7 8
> 
>   Take smallest = 1 → MUST start a group [1,2,3]:
>       1 2 3 4 6 7 8
>       █─█─█           (cnt 1→0, 2→1, 3→1)
> 
>   Take smallest with cnt>0 = 2 → MUST start [2,3,4]:
>       1 2 3 4 6 7 8
>         █─█─█         (cnt 2→0, 3→0, 4→0)
> 
>   Take smallest with cnt>0 = 6 → MUST start [6,7,8]:
>       1 2 3 4 6 7 8
>               █─█─█   (cnt 6→0, 7→0, 8→0)
> 
>   All buckets emptied → true.
> 
>   Why "smallest first" is forced: if smallest x exists, no smaller value
>   is available to anchor it, so x MUST be the leftmost of its group.
> ```

> [!info]- 🔍 Dry Run: hand=[1,2,3,6,2,3,4,7,8], groupSize=3
> ```text
> len(hand)=9 % 3 = 0 ✓
> 
> Counter: {1:1, 2:2, 3:2, 6:1, 4:1, 7:1, 8:1}
> Sorted keys: [1, 2, 3, 4, 6, 7, 8]
> 
> Process x=1: c = cnt[1] = 1
>   for i=0..2 (groupSize): cnt[1+i] must be >= 1
>     cnt[1]=1 ≥ 1 ✓; cnt[1] -= 1 = 0
>     cnt[2]=2 ≥ 1 ✓; cnt[2] -= 1 = 1
>     cnt[3]=2 ≥ 1 ✓; cnt[3] -= 1 = 1
>   Took group [1,2,3]
> 
> Process x=2: cnt[2]=1 ≠ 0
>   for i=0..2:
>     cnt[2]=1 ≥ 1; cnt[2] -= 1 = 0
>     cnt[3]=1 ≥ 1; cnt[3] -= 1 = 0
>     cnt[4]=1 ≥ 1; cnt[4] -= 1 = 0
>   Took group [2,3,4]
> 
> Process x=3: cnt[3]=0 → skip
> Process x=4: cnt[4]=0 → skip
> 
> Process x=6: cnt[6]=1
>   for i=0..2: cnt[6]=1; cnt[7]=1; cnt[8]=1 → all ≥ 1, decrement
>   Took group [6,7,8]
> 
> Process x=7, x=8: cnt=0 → skip
> 
> All groups formed. ✅ Answer: true
> ```

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

> [!example]- 📊 Visual: componentwise filter + max
> ```text>
>   target = (2, 7, 5)
> 
>   triplet     pos0  pos1  pos2   verdict
>   ─────────────────────────────────────────────────
>   (2, 5, 3)    2     5     3     all ≤ target  ✓ keep
>   (1, 8, 4)    1    [8]    4     8 > 7  ✗ TOXIC (drop)
>   (1, 7, 5)    1     7     5     all ≤ target  ✓ keep
> 
>   Componentwise MAX of kept rows = candidate result:
> 
>       (2, 5, 3)
>     ⊔ (1, 7, 5)
>     ─────────
>       (2, 7, 5)  == target  ✓
> 
>   Track which positions have already matched target exactly:
>       good[0]=T  (from (2,5,3))
>       good[1]=T  (from (1,7,5))
>       good[2]=T  (from (1,7,5))
> 
>   Why drop "toxic" triplets? Their oversize component would poison the max
>   forever — and componentwise max only goes up.
> ```

> [!info]- 🔍 Dry Run: triplets=[[2,5,3],[1,8,4],[1,7,5]], target=[2,7,5]
> ```text
> good = [False, False, False]   (target hit at each position)
> 
> Triplet (2,5,3):
>   2 > 2? NO. 5 > 7? NO. 3 > 5? NO. Eligible.
>   Check matches:
>     2 == 2 → good[0] = True
>     5 == 7? NO
>     3 == 5? NO
> 
> Triplet (1,8,4):
>   1 > 2? NO. 8 > 7? YES → skip (this triplet has 8 in position 1 which exceeds 7)
> 
> Triplet (1,7,5):
>   1 > 2? NO. 7 > 7? NO. 5 > 5? NO. Eligible.
>   Check:
>     1 == 2? NO
>     7 == 7 → good[1] = True
>     5 == 5 → good[2] = True
> 
> all(good) = True
> 
> ✅ Answer: true
>   By componentwise-max of [2,5,3] and [1,7,5] = [2,7,5] = target.
> ```

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

> [!example]- 📊 Visual: expand window to last-occurrence
> ```text
>   s = a b a b c b a c a d e f e g d e h i j h k l i j
>       0 1 2 3 4 5 6 7 8 9 ...                        23
> 
>   last[c]:  a:8  b:5  c:7  d:14  e:15  f:11  g:13
>             h:19 i:22 j:23  k:20 l:21
> 
>   Walk i; end = max(end, last[s[i]]):
> 
>     i=0 'a'  end=8
>     i=1..7   end stays 8 (b,c all have last ≤ 8)
>     i=8 'a'  i == end → CUT here → part "ababcbaca" (size 9)
> 
>     i=9 'd'  end=14
>     i=10 'e' end=15
>     i=11..14 end stays 15
>     i=15 'e' i == end → CUT → part "defegde" (size 7)
> 
>     i=16 'h' end=19
>     i=17 'i' end=22
>     i=18 'j' end=23
>     i=23 'j' i == end → CUT → part "hijhklij" (size 8)
> 
>   Partitions:
>   │ a b a b c b a c a │ d e f e g d e │ h i j h k l i j │
>   └─────── 9 ────────┘└──── 7 ───────┘└──── 8 ────────┘
> 
>   Answer: [9, 7, 8]
> ```

> [!info]- 🔍 Dry Run: s="ababcbacadefegdehijhklij"
> ```text
> last index of each char:
>   a:8, b:5, c:7, d:14, e:15, f:11, g:13, h:19, i:22, j:23, k:20, l:21
> 
> start = 0, end = 0
> 
> i=0 'a': end = max(0, 8) = 8
> i=1 'b': end = max(8, 5) = 8
> i=2 'a': end = max(8, 8) = 8
> i=3 'b': end = 8
> i=4 'c': end = max(8, 7) = 8
> i=5 'b': end = 8
> i=6 'a': end = 8
> i=7 'c': end = 8
> i=8 'a': end = 8; i == end → cut! part size = 8-0+1 = 9; start = 9
> i=9 'd': end = max(9, 14) = 14
> i=10 'e': end = max(14, 15) = 15
> i=11 'f': end = max(15, 11) = 15
> i=12 'e': end = 15
> i=13 'g': end = max(15, 13) = 15
> i=14 'd': end = 15
> i=15 'e': end = 15; i == end → cut! part size = 15-9+1 = 7; start = 16
> i=16 'h': end = max(16, 19) = 19
> i=17 'i': end = max(19, 22) = 22
> i=18 'j': end = max(22, 23) = 23
> i=19 'h': end = 23
> i=20 'k': end = max(23, 20) = 23
> i=21 'l': end = max(23, 21) = 23
> i=22 'i': end = 23
> i=23 'j': end = 23; i == end → cut! size = 23-16+1 = 8
> 
> ✅ Answer: [9, 7, 8]
> ```

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

> [!example]- 📊 Visual: track range [lo, hi] of possible open-counts
> ```text
>   s = ( * ) )
> 
>   Treat '*' as could-be '('  '0'  or  ')'.
>   lo = "min open count assuming '*' = ')'"   (or skip)
>   hi = "max open count assuming '*' = '('"
> 
>     char    '('    '*'    ')'    ')'
>     lo:  0 → 1  →  0  →  -1  → clamp 0
>     hi:  0 → 1  →  2  →   1  →  0
> 
>   Visualize the feasible-open band:
> 
>        hi  ─┐       ╭─╮
>             │  ╭────╯ │
>             │  │      ╰─╮
>        lo  ─┴──┴────────╯─
>             (   *    )   )
> 
>   Rules:
>     - If hi < 0 mid-scan → impossible (too many ')')  return false
>     - lo is clamped at 0 (can't have negative open count)
>     - At end, lo == 0 means a valid assignment exists.
> 
>   Here lo ends at 0 → answer true.  e.g. treat '*' as '(' → "(()))"... wait,
>   actually treat '*' as '(' gives "(()" + ")" + ")" = "(())". ✓
> ```

> [!info]- 🔍 Dry Run: s="(*))"
> ```text
> lo = hi = 0
> 
> c='(': lo=1, hi=1
> c='*': lo=0 (treat as ')'), hi=2 (treat as '(')
>   But if lo would go negative, clamp at 0.
>   lo=max(0, 0) = 0
> c=')': lo=-1 → max(0)=0; hi=1
> c=')': lo=-1 → 0; hi=0
> 
> End check: lo == 0? YES → valid
> 
> ✅ Answer: true
>   (Interpretation: '*' becomes '(', giving "(())")
> 
> ─────────────────────────────────────────
> Counter-example: s="(("
>   '(': lo=1, hi=1
>   '(': lo=2, hi=2
>   End: lo=2 ≠ 0 → false
> 
> ─────────────────────────────────────────
> Edge case: hi goes negative → s="(()))"
>   '(': lo=1 hi=1
>   '(': lo=2 hi=2
>   ')': lo=1 hi=1
>   ')': lo=0 hi=0
>   ')': lo=-1→0 hi=-1
>   hi < 0 during processing → return false immediately
> ```

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

**Key takeaway:** Wildcards → track range of states.

---

## P9: Best Time to Buy and Sell Stock II

**LC #122** · Easy

> [!example]- 📊 Visual: sum positive adjacent diffs = sum of all up-runs
> ```text
>   prices = [7, 1, 5, 3, 6, 4]
> 
>          7 ●
>            ╲
>            ╲                6 ●
>             ╲     5 ●       ╱  ╲
>              ╲   ╱   ╲     ╱   ╲ 4 ●
>               ╲ ╱     ╲ 3 ●     ●
>            1 ●
>            day 0   1   2   3   4   5
> 
>   Diffs (next - prev):    -6  +4  -2  +3  -2
>   Keep only positives:        +4      +3        = 7
> 
>   Why this works:
>     Any multi-day rise  d0 → d1 → d2 → d3   (all up)
>     can be split into back-to-back day trades:
>       buy d0, sell d3  ==  (d1-d0) + (d2-d1) + (d3-d2)
>     Pocket every up-step; never pay for down-steps.
> ```

> [!info]- 🔍 Dry Run: prices=[7,1,5,3,6,4]
> ```text
> Sum of positive adjacent diffs:
>   day 1-0: 1-7 = -6 → 0
>   day 2-1: 5-1 = 4 → +4
>   day 3-2: 3-5 = -2 → 0
>   day 4-3: 6-3 = 3 → +3
>   day 5-4: 4-6 = -2 → 0
> 
> Total = 4 + 3 = 7
> 
> ✅ Answer: 7   (buy at 1 sell at 5, buy at 3 sell at 6 → 4+3=7)
> ```

> [!success]- Python
> ```python
> def max_profit_ii(prices):
>     return sum(max(0, prices[i] - prices[i-1]) for i in range(1, len(prices)))
> ```

**Key takeaway:** "Many transactions" = greedy on adjacent diffs.

---

## P10: Boats to Save People

**LC #881** · Medium

> [!example]- 📊 Visual: pair lightest ↔ heaviest
> ```text
>   people sorted = [1, 2, 2, 3]      limit = 3
>                    l        r
> 
>   Step 1:  l=0 (1), r=3 (3)   1+3 = 4 > 3
>            → heaviest goes ALONE.   [3]
>            r--                       boats = 1
> 
>   Step 2:  l=0 (1), r=2 (2)   1+2 = 3 ≤ 3
>            → pair them.       [1,2]
>            l++, r--                  boats = 2
> 
>   Step 3:  l=1 (2), r=1 (2)   same person, take alone.   [2]
>            r--                       boats = 3
> 
>           l=1 > r=0 → done
> 
>   Visual layout of boats:
>     boat A:  [        3 ]     (limit 3, exactly fits)
>     boat B:  [ 1    2   ]     (1+2=3)
>     boat C:  [    2     ]     (lone)
> 
>   Why pair lightest with heaviest?
>     If heaviest can ride with ANY remaining person, it's the lightest one.
>     If it can't ride with the lightest, it can't ride with anyone → alone.
> ```

> [!info]- 🔍 Dry Run: people=[3,2,2,1], limit=3
> ```text
> Sort: [1, 2, 2, 3]
> l=0, r=3, boats=0
> 
> Iter 1: 1+3=4 > 3 → heaviest alone; r-- to 2; boats=1
> Iter 2: 1+2=3 ≤ 3 → both go; l++ to 1; r-- to 1; boats=2
> 
> l==r=1: still one person left
>   1+2=3 ≤ 3? wait l=r so one boat for that single person:
>   Actually loop continues while l <= r.
>   l=1, r=1: 2+2=4 > 3 → boat alone; r--; boats=3
>   
> Wait let me redo carefully:
> 
> ─────────────────────────────────────────
> Re-trace: people=[3,2,2,1] sorted = [1,2,2,3]
> l=0, r=3, boats=0
> 
> Iter 1: people[0]+people[3] = 1+3 = 4 > 3 → only heaviest goes
>   r=2, boats=1
> 
> Iter 2: people[0]+people[2] = 1+2 = 3 ≤ 3 → both go
>   l=1, r=1, boats=2
> 
> Iter 3: people[1]+people[1] (same person): 2+2=4 > 3 → only one
>   r=0, boats=3
> 
> Now l=1 > r=0 → exit loop.
> 
> ✅ Answer: 3 boats
> ```

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

## P11: Candy

**LC #135** · **Hard**

Give candies to children with ratings: every child gets at least 1; higher-rated child gets more than each neighbor.

### 🧠 Pattern: Two-Pass Greedy (Left-to-Right + Right-to-Left)

> Each constraint involves only one neighbor. Solve left-neighbor and right-neighbor constraints independently with two sweeps, then take max.

> [!example]- 📊 Visual: two-pass max (left-pass + right-pass)
> ```text
>   ratings = [1, 3, 4, 5, 2]
>   idx         0  1  2  3  4
> 
>   Left-to-right pass (only fixes "i > i-1"):
>     start  : 1 1 1 1 1
>     i=1 3>1 → c[1]=c[0]+1=2
>     i=2 4>3 → c[2]=c[1]+1=3
>     i=3 5>4 → c[3]=c[2]+1=4
>     i=4 2>5? no
>     L =      [1, 2, 3, 4, 1]
> 
>   Right-to-left pass (only fixes "i > i+1"):
>     start  : 1 1 1 1 1
>     i=3 5>2 → c[3]=c[4]+1=2
>     i=2 4>5? no
>     i=1 3>4? no
>     i=0 1>3? no
>     R =      [1, 1, 1, 2, 1]
> 
>   Final = max(L, R) element-wise:
>     max:    [1, 2, 3, 4, 1]
> 
>   Bar view:
>     L:  ▁ ▂ ▃ ▄ ▁
>     R:  ▁ ▁ ▁ ▂ ▁
>     ─── max ─────
>         ▁ ▂ ▃ ▄ ▁     sum = 11
> 
>   Why two passes? Each pass enforces ONE neighbor constraint.
>   Taking the max satisfies BOTH simultaneously (any constraint asks ≥ X;
>   the max is ≥ both Xs).
> ```

> [!info]- 🔍 Dry Run: ratings=[1,0,2]
> ```text
> Init candies = [1, 1, 1]
> 
> Left-to-right sweep (ensure left-neighbor constraint):
>   i=1: ratings[1]=0 > ratings[0]=1? NO → skip
>   i=2: ratings[2]=2 > ratings[1]=0? YES → candies[2] = candies[1] + 1 = 2
>   candies = [1, 1, 2]
> 
> Right-to-left sweep (ensure right-neighbor constraint):
>   i=1: ratings[1]=0 > ratings[2]=2? NO → skip
>   i=0: ratings[0]=1 > ratings[1]=0? YES → candies[0] = max(candies[0]=1, candies[1]+1=2) = 2
>   candies = [2, 1, 2]
> 
> Sum = 2 + 1 + 2 = 5
> 
> ✅ Answer: 5
> 
> ─────────────────────────────────────────
> ratings=[1,2,2]:
>   init [1,1,1]
>   L→R: i=1: 2>1 → candies[1]=2. i=2: 2>2? NO. → [1,2,1]
>   R→L: i=1: 2>2 NO. i=0: 1>2 NO. → [1,2,1]
>   Sum = 4
>   Note: child 2 has ratings[2]==ratings[1]=2, so the "strictly higher" rule doesn't apply between them.
> ```

> [!success]- Python
> ```python
> def candy(ratings):
>     n = len(ratings)
>     candies = [1] * n
>     for i in range(1, n):
>         if ratings[i] > ratings[i - 1]:
>             candies[i] = candies[i - 1] + 1
>     for i in range(n - 2, -1, -1):
>         if ratings[i] > ratings[i + 1]:
>             candies[i] = max(candies[i], candies[i + 1] + 1)
>     return sum(candies)
> ```

**Variants:** Trapping Rain Water (two-sweep max), Best Time to Buy and Sell Stock with Cooldown (two passes).

**Key takeaway:** When a constraint has a "look left" AND "look right" dependency, two passes suffice. Each is independently greedy; the final `max` reconciles.

---

> [!tip] After this drill
> Recognize: "sort + sweep" → 80% of greedy problems. The hard part is **proving** the greedy rule works — practice articulating the exchange argument.
