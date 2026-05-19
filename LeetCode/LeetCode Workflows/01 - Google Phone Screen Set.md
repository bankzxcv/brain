---
title: "Workflow 01: Google Phone Screen Set"
date: 2026-05-19
tags:
  - leetcode
  - workflow
  - mock-interview
  - google
parent: "[[LeetCode Study]]"
status: in-progress
---

# Workflow 01: Google Phone Screen Set

> [!abstract] Format
> **45 minutes** · **1 medium problem** · think out loud · shared doc, no autocomplete · Google interviewer expects: clarifying questions + brute force → optimization → code → trace → complexity → edge cases.

> [!warning] Set a real timer
> The point of a mock is realistic time pressure. Use a stopwatch. Stop at 45 minutes regardless of progress.

## The Round Protocol

```
0:00 - 0:02   Restate & clarify       (2 questions minimum)
0:02 - 0:05   Brute force out loud    (don't code)
0:05 - 0:08   Optimize: identify pattern, propose solution
0:08 - 0:20   Code the optimal
0:20 - 0:25   Trace + edge case check
0:25 - 0:30   Discuss complexity, optimizations
0:30 - 0:40   Follow-up question
0:40 - 0:45   Questions for interviewer (always ask 1-2)
```

---

## Problem: Longest Substring Without Repeating Characters

> [!example] Interviewer prompt
> "Given a string `s`, find the length of the longest substring without repeating characters."
> 
> Example: `s = "abcabcbb"` → 3 (`"abc"`)
> Example: `s = "bbbbb"` → 1
> Example: `s = "pwwkew"` → 3 (`"wke"`, not `"pwke"` — substring, not subsequence)

---

### 🎬 Interviewer's Hidden Script

> [!example]- What the interviewer is looking for
> **Must say:**
> - Clarifying questions (Unicode? case-sensitive? what's "substring" vs "subsequence"?)
> - Identify it as a sliding-window problem before coding
> - Mention brute force complexity O(n²) or O(n³) before jumping to optimization
> 
> **Red flags:**
> - Silent for >30s
> - Codes immediately without sketching
> - Says "I've memorized this" — interviewer will give a follow-up
> 
> **If you finish early (Google ~always does):** follow-up is usually:
> > "What if the input is a stream — chars arrive one at a time and you must return the answer at any moment?"

---

### Walkthrough — What a Great Answer Looks Like

**1. RESTATE (15s)**
> "So we need the length of the longest substring — contiguous chars — of `s` where every character is unique."

**2. CLARIFY (30s) — Ask 2-3:**
- "Are characters ASCII or could be Unicode?"
- "Is it case-sensitive? `'A'` and `'a'` distinct?"
- "Can `s` be empty? Return 0 then?"

**3. EXAMPLE WALK (30s)**
> Sketch `"pwwkew"`:
> - `p` unique → len 1
> - `pw` unique → len 2
> - `pww` duplicate `w`! shrink from left → `ww` still duplicate → `w`
> - continue `wk`, `wke` → len 3 ✓
> - etc.
> Observation: a sliding window with two pointers + a set/map of seen chars.

**4. BRUTE (1m, don't code)**
> "Brute force: for each (i, j) check if `s[i..j]` has all unique chars — O(n³). Or for each i, walk right until duplicate — O(n²)."

**5. OPTIMIZE (2m)**
> "The pattern here is **variable sliding window**. I'll keep `l, r`. Expand `r`. When `s[r]` is already in my set, shrink `l` until it's not. After each expansion, update the best length. Each char is added/removed at most once → O(n)."

**6. CODE (10m)**

> [!success]- Reference solution — Python
> ```python
> def length_of_longest_substring(s):
>     seen = set()
>     l = best = 0
>     for r, c in enumerate(s):
>         while c in seen:
>             seen.remove(s[l])
>             l += 1
>         seen.add(c)
>         best = max(best, r - l + 1)
>     return best
> ```

> [!success]- Reference solution — JS
> ```js
> const lengthOfLongestSubstring = (s) => {
>   const seen = new Set();
>   let l = 0, best = 0;
>   for (let r = 0; r < s.length; r++) {
>     while (seen.has(s[r])) seen.delete(s[l++]);
>     seen.add(s[r]);
>     best = Math.max(best, r - l + 1);
>   }
>   return best;
> };
> ```

**7. TRACE (2m)**
> Walk through `"abcabcbb"`:
> - `r=0 'a'` set={a} best=1
> - `r=1 'b'` set={a,b} best=2
> - `r=2 'c'` set={a,b,c} best=3
> - `r=3 'a'` duplicate → shrink: remove `a` (l=1), add `a` → set={b,c,a} best=3
> - ... best stays 3

**8. COMPLEXITY (30s)**
> "Time O(n) — l and r each move at most n steps. Space O(min(n, alphabet)) — set bounded by distinct chars."

**9. EDGE CASES (1m)**
> - Empty string → 0 ✓ (loop doesn't execute)
> - Single char → 1 ✓
> - All same char `"aaaa"` → 1 ✓
> - All unique → n ✓
> - Unicode? Need clarification on what counts; works as long as comparison is per-codepoint.

---

### 🔥 Follow-up: Stream Input

> [!example] Interviewer's follow-up
> "What if `s` is a stream — characters arrive one at a time, and at any moment we must be able to return the current best?"

**Discuss before coding:**
- Same algorithm! Just pump each new char into the loop.
- API: `Stream.next(c)` returns updated best.
- Need: a `Map<char, last_seen_index>` — replacing the set lets us jump `l` directly to `last_seen + 1` instead of stepping.

> [!success]- Stream solution
> ```python
> class LSStream:
>     def __init__(self):
>         self.last = {}      # char -> index
>         self.l = 0
>         self.i = -1
>         self.best = 0
>     def next(self, c):
>         self.i += 1
>         if c in self.last and self.last[c] >= self.l:
>             self.l = self.last[c] + 1
>         self.last[c] = self.i
>         self.best = max(self.best, self.i - self.l + 1)
>         return self.best
> ```

---

### Questions to Ask the Interviewer (5 min)

Always have 2 prepared:
1. "What's a typical day for you on this team?"
2. "What's the most challenging problem your team is working on right now?"
3. "How do you see this role evolving over the next year?"

---

> [!tip] After this mock
> Did you stay silent at any point? Did you code before optimizing? Did you forget edge cases? Repeat this round until you can hit every checkpoint within the time budget.

---

## Extensions

After mastering this round, try these as your "Phone Screen Set":
- LC #20 Valid Parentheses (warmup)
- LC #146 LRU Cache (design)
- LC #973 K Closest Points (heap)
- LC #200 Number of Islands (graph)
