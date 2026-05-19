---
title: LeetCode Study
date: 2026-05-19
tags:
  - leetcode
  - algorithms
  - interview
  - study
  - course
aliases:
  - Coding Interview Prep
  - DSA Patterns
  - Algorithm Patterns
status: in-progress
---

# LeetCode Study

> [!tip] Why this course?
> Top-tech interviews (Google, Meta, Amazon, etc.) test **pattern recognition speed**, not LeetCode trivia. This course drills 19 patterns through ~230 progressive problems in JS and Python. Goal: when the interviewer states a problem, your brain auto-matches it to one of these patterns within 30 seconds.

> [!abstract] Course Structure
> **Reference (this note)** — 19-pattern cheat sheet + interview playbook + 8-week schedule.
> **Practice** — 19 topic drills, frequency-weighted. See [[#Practice Drills]].
> **Workflows** — 5 mock interview rounds simulating Google phone/onsite. See [[#Mock Interview Rounds]].

---

## Part 1 — The 23 Patterns Cheat Sheet

Memorize the **Recognize when...** column. That's the interview superpower.

| # | Pattern | Recognize when... | Tool | Time / Space |
|---|---|---|---|---|
| 1 | **Complement Lookup** | "find pair summing to X" | Hash map | O(n) / O(n) |
| 2 | **Prefix Sum** | "range sum / subarray sum equals K" | Cumulative sum + hash | O(n) / O(n) |
| 3 | **Sliding Window — fixed** | "max/min/sum of k-length subarray" | Two pointers + running stat | O(n) / O(1) |
| 4 | **Sliding Window — variable** | "longest/shortest substring with..." | Two pointers + map/set | O(n) / O(k) |
| 5 | **Two Pointers — opposite** | "sorted array, pair/triplet sum" | l, r pointers, move inward | O(n) / O(1) |
| 6 | **Two Pointers — same dir** | "remove dups in place, partition" | slow / fast pointers | O(n) / O(1) |
| 7 | **Fast / Slow Pointers** | "cycle / middle / nth from end" | tortoise & hare | O(n) / O(1) |
| 8 | **Binary Search — sorted** | "sorted, find target / insert pos" | l ≤ r, mid = (l+r)/2 | O(log n) / O(1) |
| 9 | **Binary Search — on answer** | "min/max value satisfying P(x)" | Search space = answer range | O(n log k) / O(1) |
| 10 | **Monotonic Stack** | "next greater/smaller element" | Stack of decreasing/increasing | O(n) / O(n) |
| 11 | **Tree DFS (recursion)** | "root-to-leaf, path sum, depth" | Recurse left + right | O(n) / O(h) |
| 12 | **Tree BFS (level order)** | "level-by-level, min depth, rightmost" | Queue | O(n) / O(w) |
| 13 | **BST Property** | "validate BST / kth smallest / search" | left < root < right; in-order = sorted | O(h) / O(h) |
| 14 | **Trie** | "prefix lookup / word dictionary" | Char-indexed tree, terminal flag | O(L) / O(N·L) |
| 15 | **Heap (Top-K)** | "K largest / smallest / merge K" | Min-heap of size K | O(n log k) / O(k) |
| 16 | **Backtracking** | "all permutations / subsets / combinations" | choose → recurse → un-choose | O(b^d) / O(d) |
| 17 | **Graph BFS** | "shortest path in unweighted graph" | Queue + visited set | O(V+E) / O(V) |
| 18 | **Graph DFS** | "connected components / cycle / paths" | Recurse / stack + visited | O(V+E) / O(V) |
| 19 | **Topological Sort** | "ordering with prerequisites / DAG" | Kahn's (BFS+indegree) or DFS post-order | O(V+E) / O(V) |
| 20 | **Union-Find (DSU)** | "dynamic connectivity / # components" | DSU + path compression + union by rank | O(α(n)) / O(n) |
| 21 | **DP — 1D** | "fib-like / one choice per step" | dp[i] from dp[i-1 .. i-k] | O(n) / O(n) |
| 22 | **DP — 2D** | "two sequences / grid paths" | dp[i][j] from dp[i-1][j], dp[i][j-1] | O(nm) / O(nm) |
| 23 | **Greedy + Intervals** | "sort then sweep / pick locally optimal" | Sort by start/end, sweep | O(n log n) / O(1) |

> [!tip] Pattern → Topic File Map
> Patterns 1–2 → [[LeetCode Practice/01 - Arrays and Hashing|01 Arrays and Hashing]]
> Patterns 5–7 → [[LeetCode Practice/02 - Two Pointers|02 Two Pointers]]
> Patterns 3–4 → [[LeetCode Practice/03 - Sliding Window|03 Sliding Window]]
> Pattern 10 → [[LeetCode Practice/05 - Monotonic Stack|05 Monotonic Stack]]
> Patterns 8–9 → [[LeetCode Practice/06 - Binary Search|06 Binary Search]]
> Patterns 11–13 → [[LeetCode Practice/08 - Trees|08 Trees]]
> Pattern 17–18 → [[LeetCode Practice/12 - Graphs|12 Graphs]]
> Patterns 19–20 → [[LeetCode Practice/13 - Advanced Graphs|13 Advanced Graphs]]
> Pattern 21 → [[LeetCode Practice/14 - 1D Dynamic Programming|14 1D DP]]
> Pattern 22 → [[LeetCode Practice/15 - 2D Dynamic Programming|15 2D DP]]
> Pattern 23 → [[LeetCode Practice/16 - Greedy|16 Greedy]] + [[LeetCode Practice/17 - Intervals|17 Intervals]]

---

## Part 2 — Interview Playbook

### 2.1 The 7-Step Think-Aloud Framework

Every interview problem. Same script. Drill until automatic.

```
1. RESTATE   (15s) — "So we need to ... return ..."
2. CLARIFY   (30s) — input bounds, edge cases, output format (2-3 Qs)
3. EXAMPLE   (30s) — walk through given example, draw it
4. BRUTE     (1m)  — state O(n²) or O(2^n) solution. DO NOT code it.
5. OPTIMIZE  (2m)  — identify the pattern, propose better. State complexity.
6. CODE      (10m) — code the optimal. Talk while typing.
7. TRACE+EDGE(2m)  — trace your code on example. Test edge cases.
```

### 2.2 Clarifying Questions Checklist

Always ask 2–3 of these (calibrate to the problem):

- **Input size?** ("n up to 10⁵?") — tells you the target complexity
- **Negative numbers / zeros / duplicates?**
- **Sorted / unsorted?**
- **Empty / single-element input?**
- **Modification allowed?** (can I mutate the input array?)
- **Output: index or value? unique answer or all?**
- **Memory constraint?** (for stream / huge input problems)
- **Unicode / ASCII?** (string problems)

### 2.3 When You're Stuck — 5-Step Recovery

```
1. SAY IT OUT LOUD — "I'm thinking through X. Let me look at a small example."
2. WORK A SMALLER EXAMPLE — n=3 or n=4 manually
3. CHECK PATTERNS — does this look like sliding window? two-pointer? DP?
4. RELAX A CONSTRAINT — solve a simpler version first
5. BRUTE FORCE THEN OPTIMIZE — code anything that works, then improve
```

> [!warning] Silence is the enemy
> Interviewers can't help you if you're silent. Narrate. Even "I'm not sure yet, let me think" is better than 30s of silence.

### 2.4 Complexity Quick Reference

| n | Acceptable complexity |
|---|---|
| n ≤ 10 | O(n!) — permutations, backtracking |
| n ≤ 20 | O(2^n) — subsets, bitmask DP |
| n ≤ 500 | O(n³) — Floyd-Warshall, naive DP |
| n ≤ 5,000 | O(n²) — nested loops, 2D DP |
| n ≤ 10⁶ | O(n log n) — sort, heap, segment tree |
| n ≤ 10⁸ | O(n), O(log n), O(1) |

### 2.5 Common Google Interview Themes

Google interviewers especially love:
- **Graphs on grids** — Number of Islands, Word Search, Walls and Gates
- **Tree recursion with state** — Lowest Common Ancestor, House Robber III
- **Interval problems** — Meeting Rooms, Merge Intervals, Calendar
- **Binary search on answer** — Capacity to Ship, Koko Eating Bananas
- **Trie + DFS** — Word Search II, Auto-complete
- **Heap top-K** — K Closest Points, Top K Frequent
- **DP with bitmask** — Traveling Salesman, Shortest Path Visiting All Nodes

### 2.6 Behavioral Framing — STAR

For "tell me about a time..." use:
- **S**ituation — context in 1 sentence
- **T**ask — your responsibility in 1 sentence
- **A**ction — what YOU did (60% of the answer)
- **R**esult — measurable outcome + lesson learned

---

## Part 3 — 8-Week Drill Schedule

Two patterns per week. Re-drill weak topics in the final week.

| Week | Focus | Topics |
|---|---|---|
| **1** | Foundations | [[LeetCode Practice/01 - Arrays and Hashing\|Arrays]] + [[LeetCode Practice/02 - Two Pointers\|Two Pointers]] |
| **2** | Windows & Stacks | [[LeetCode Practice/03 - Sliding Window\|Sliding Window]] + [[LeetCode Practice/04 - Stack\|Stack]] + [[LeetCode Practice/05 - Monotonic Stack\|Monotonic Stack]] |
| **3** | Search & Linear Structures | [[LeetCode Practice/06 - Binary Search\|Binary Search]] + [[LeetCode Practice/07 - Linked List\|Linked List]] |
| **4** | Trees | [[LeetCode Practice/08 - Trees\|Trees]] + [[LeetCode Practice/09 - Tries\|Tries]] |
| **5** | Heaps & Backtracking | [[LeetCode Practice/10 - Heap and Priority Queue\|Heap/PQ]] + [[LeetCode Practice/11 - Backtracking\|Backtracking]] |
| **6** | Graphs | [[LeetCode Practice/12 - Graphs\|Graphs]] + [[LeetCode Practice/13 - Advanced Graphs\|Advanced Graphs]] |
| **7** | DP | [[LeetCode Practice/14 - 1D Dynamic Programming\|1D DP]] + [[LeetCode Practice/15 - 2D Dynamic Programming\|2D DP]] |
| **8** | Polish + Mocks | [[LeetCode Practice/16 - Greedy\|Greedy]] + [[LeetCode Practice/17 - Intervals\|Intervals]] + [[LeetCode Practice/18 - Math and Geometry\|Math]] + [[LeetCode Practice/19 - Bit Manipulation\|Bit]] + all [[#Mock Interview Rounds\|workflows]] |

> [!tip] Daily cadence
> 4 problems/day, 30 min each, 5 days/week → 80 problems/week × 8 weeks ≈ full course twice (re-drilling = retention). Re-do every problem you fail. Don't move to the next topic until you can write the canonical solution from scratch in under 10 minutes.

---

## Practice Drills

19 topic files, **~238 problems total** (with **24+ Hard** problems for Google-level depth). Each problem has a **🔍 Dry Run** collapsible callout showing step-by-step debugger-style tracing — type these in interviews to demonstrate your thinking.

| # | Topic | Problems | Hard count | Patterns |
|---|---|---|---|---|
| 01 | [[LeetCode Practice/01 - Arrays and Hashing]] | 17 | 2 | Complement Lookup, Prefix Sum, Rolling Hash |
| 02 | [[LeetCode Practice/02 - Two Pointers]] | 12 | 1 | Opposite, Same-dir, Fast/Slow |
| 03 | [[LeetCode Practice/03 - Sliding Window]] | 12 | 3 | Fixed window, Variable window |
| 04 | [[LeetCode Practice/04 - Stack]] | 12 | 2 | LIFO mechanics, eval, parentheses |
| 05 | [[LeetCode Practice/05 - Monotonic Stack]] | 10 | 3 | Next greater/smaller |
| 06 | [[LeetCode Practice/06 - Binary Search]] | 14 | 2 | Sorted search, Search on answer |
| 07 | [[LeetCode Practice/07 - Linked List]] | 14 | 2 | Reverse, merge, cycle, manipulation |
| 08 | [[LeetCode Practice/08 - Trees]] | 18 | 2 | DFS, BFS, BST |
| 09 | [[LeetCode Practice/09 - Tries]] | 8 | 2 | Prefix tree, word search |
| 10 | [[LeetCode Practice/10 - Heap and Priority Queue]] | 12 | 3 | Top-K, merge-K, schedule |
| 11 | [[LeetCode Practice/11 - Backtracking]] | 13 | 2 | Subsets, permutations, combinations, Sudoku |
| 12 | [[LeetCode Practice/12 - Graphs]] | 16 | 1 | BFS/DFS on grid + adjacency |
| 13 | [[LeetCode Practice/13 - Advanced Graphs]] | 12 | 2 | Topo Sort, Union-Find, Dijkstra |
| 14 | [[LeetCode Practice/14 - 1D Dynamic Programming]] | 17 | 1 | Fib-like, Kadane, LIS, regex matching |
| 15 | [[LeetCode Practice/15 - 2D Dynamic Programming]] | 12 | 3 | LCS, edit distance, burst balloons |
| 16 | [[LeetCode Practice/16 - Greedy]] | 11 | 1 | Jump game family, scheduling, candy |
| 17 | [[LeetCode Practice/17 - Intervals]] | 10 | 1 | Merge, insert, overlap |
| 18 | [[LeetCode Practice/18 - Math and Geometry]] | 9 | 1 | Pow, rotation, reverse pairs |
| 19 | [[LeetCode Practice/19 - Bit Manipulation]] | 9 | 1 | XOR tricks, masks, bit trie |
| | **Total** | **238** | **35** | |

---

## Mock Interview Rounds

Real-time simulations. Set a timer, no autocomplete, narrate your thinking.

1. [[LeetCode Workflows/01 - Google Phone Screen Set]] — 45 min · 1 medium
2. [[LeetCode Workflows/02 - Onsite Round - Arrays and Strings]] — 45 min · 2 problems
3. [[LeetCode Workflows/03 - Onsite Round - Trees and Graphs]] — 45 min · 2 problems
4. [[LeetCode Workflows/04 - Onsite Round - DP and Backtracking]] — 45 min · 2 problems
5. [[LeetCode Workflows/05 - Onsite Round - System-Adjacent]] — 45 min · 1 design-flavored

---

## How to Use This Course

> [!example] Drill protocol for each problem
> 1. **Read** problem + edge cases. Close the file.
> 2. **Restate** out loud — what's the input/output?
> 3. **Identify the pattern** from the cheat sheet.
> 4. **Code from scratch** in JS or Python — don't peek.
> 5. **Reveal solution** — diff against yours.
> 6. **Mark status**: ✅ got it · 🟡 needed hint · ❌ couldn't solve
> 7. **Re-drill** any ❌ within 24 hours, any 🟡 within 3 days.

> [!warning] Anti-pattern: passive reading
> If you "just read" the solutions, you'll remember nothing in the interview. Every problem MUST be coded by hand, at least twice. Muscle memory > understanding.
