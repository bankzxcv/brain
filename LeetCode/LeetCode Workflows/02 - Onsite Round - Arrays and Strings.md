---
title: "Workflow 02: Onsite Round — Arrays and Strings"
date: 2026-05-19
tags:
  - leetcode
  - workflow
  - mock-interview
  - arrays
  - strings
parent: "[[LeetCode Study]]"
status: in-progress
---

# Workflow 02: Onsite Round — Arrays and Strings

> [!abstract] Format
> **45 minutes** · **2 medium problems** · ~22 min each · narrate fully · whiteboard or shared doc.

## Round Protocol

```
0:00 - 0:01   Greeting, brief intro
0:01 - 0:22   Problem 1 (full protocol)
0:22 - 0:43   Problem 2 (full protocol)
0:43 - 0:45   Q&A
```

> [!tip] Time discipline
> If you stall on P1, MOVE ON at 22:00. Better to attempt both than master only one.

---

## Problem 1: 3Sum

**LC #15** · Medium

> [!example] Prompt
> "Given an integer array `nums`, return all unique triplets `[a, b, c]` such that `a + b + c = 0` and the indices are distinct."

### 🎬 What the Interviewer Watches For

> [!example]- Hidden script
> **Critical:**
> - Recognize "extension of 2Sum" → fix one + 2-pointer
> - Sort upfront (cost is "free" relative to the n² inner work)
> - **Dedup correctly** — both at the outer loop AND inside the 2-pointer body
> 
> **Red flag:** using a set of triplets to dedup. Works, but is hash-of-list cost, and shows poor sense of structure.

### Walkthrough

**Clarify:** "Distinct triplets — does that mean by index or by value?" (By value.) "Sorted output required?" (No.)

**Brute (1 min):** O(n³) triple loop + set for dedup. State and reject.

**Optimize (2 min):** Sort + iterate `i`, then 2-pointer on `[i+1, n-1]` with target `-nums[i]`. Skip duplicates at the outer level (`nums[i] == nums[i-1]`) and after finding a triplet (advance l/r past duplicates).

**Code (10 min):**

> [!success]- Python
> ```python
> def three_sum(nums):
>     nums.sort()
>     out = []
>     for i in range(len(nums) - 2):
>         if i > 0 and nums[i] == nums[i - 1]: continue
>         l, r = i + 1, len(nums) - 1
>         while l < r:
>             s = nums[i] + nums[l] + nums[r]
>             if s == 0:
>                 out.append([nums[i], nums[l], nums[r]])
>                 while l < r and nums[l] == nums[l + 1]: l += 1
>                 while l < r and nums[r] == nums[r - 1]: r -= 1
>                 l += 1; r -= 1
>             elif s < 0: l += 1
>             else: r -= 1
>     return out
> ```

**Trace (2 min):** `nums=[-1,0,1,2,-1,-4]` → sorted `[-4,-1,-1,0,1,2]` → walk i=0,1 producing `[-1,-1,2]` and `[-1,0,1]`.

**Complexity:** Time O(n²) — outer loop n times, inner two-pointer O(n). Space O(1) ignoring output.

**Edge cases:**
- `n < 3` → return `[]`
- All zeros `[0,0,0,0]` → return `[[0,0,0]]` once (dedup!)
- No solution → `[]`

---

## Problem 2: Group Anagrams

**LC #49** · Medium

> [!example] Prompt
> "Group strings into anagram groups. Return the groups (any order)."

### 🎬 What the Interviewer Watches For

> [!example]- Hidden script
> **Critical:**
> - Recognize "group by canonical form"
> - Choose a hash key: sorted string OR character count tuple
> - Discuss tradeoff: sorted is O(k log k) per word; count tuple is O(k)
> 
> **Follow-up:** "What if strings are very long (say avg k = 10⁶) but the set is small? Which key is better?"

### Walkthrough

**Clarify:** "Case-sensitive? `'a'` and `'A'` distinct?" (Assume yes.) "Are strings lowercase a-z only?" (Yes — enables count-tuple optimization.)

**Brute:** O(n² · k) — compare every pair. Reject.

**Optimize:** Hash map; key = canonical form of word. Two options:
- Sorted word as key — simple, O(k log k) per word
- 26-int tuple of counts — O(k) per word

Default to sorted; mention the alternative if time permits.

**Code:**

> [!success]- Python
> ```python
> from collections import defaultdict
> def group_anagrams(strs):
>     groups = defaultdict(list)
>     for s in strs:
>         groups[tuple(sorted(s))].append(s)
>     return list(groups.values())
> ```

> [!success]- Python (count tuple version)
> ```python
> from collections import defaultdict
> def group_anagrams_fast(strs):
>     groups = defaultdict(list)
>     for s in strs:
>         cnt = [0] * 26
>         for c in s: cnt[ord(c) - 97] += 1
>         groups[tuple(cnt)].append(s)
>     return list(groups.values())
> ```

**Complexity:** Time O(n · k log k) (sorted) or O(n · k) (count tuple). Space O(n · k).

**Edge cases:**
- Empty list → `[]`
- Single empty string `[""]` → `[[""]]`
- All unique (no group has >1) → each word is its own group

### Follow-up Discussion

**Q:** "What if strings are 10⁶ chars each?"

**A:** Sorting becomes O(k log k) = 20M ops per word — slow. Count tuple is O(k), 1M ops per word — better. **But** tuple of 26 ints has small hash cost, while sorted-string hashes scale with k. So count tuple wins on multiple fronts.

**Q:** "What if alphabet is Unicode?"

**A:** 26-array doesn't work. Use a hashmap of `(char: count)`, or fall back to sorted-string. Or for many same-length words, use a polynomial hash signature.

---

## Self-Eval Checklist

After the mock:
- [ ] Did I clarify before coding? (≥ 2 questions per problem)
- [ ] Did I state brute force complexity out loud?
- [ ] Did I name the pattern explicitly? ("This is fix-one + 2-pointer")
- [ ] Did I trace through an example before declaring "done"?
- [ ] Did I check edge cases (empty / 1-element / dup-heavy)?
- [ ] Did I finish in time?
- [ ] Did I stay silent for >30s at any point?

> [!warning] If you missed > 2 boxes
> Re-do this round tomorrow. The interview is not about whether you solved it — it's about whether your **process** is credible.

---

## Extensions (other "Arrays and Strings" rounds)

- LC #76 Minimum Window Substring + LC #239 Sliding Window Maximum
- LC #238 Product of Array Except Self + LC #128 Longest Consecutive Sequence
- LC #560 Subarray Sum Equals K + LC #424 Longest Repeating Character Replacement
