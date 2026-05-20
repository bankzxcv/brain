---
title: "LeetCode Practice: Arrays and Hashing"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - arrays
  - hashing
  - prefix-sum
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Arrays and Hashing

17 progressive problems · drill the **Complement Lookup** and **Prefix Sum** patterns.

> [!abstract] Pattern recap
> **Complement Lookup** — for pair/triplet problems, store what you've seen in a hash map and look up `target - x` in O(1).
> **Prefix Sum** — replace range/subarray sum questions with O(1) lookups via cumulative sums; combine with hash map for "subarray summing to K".

> [!tip] How to use the Dry Run boxes
> Each problem has a 🔍 **Dry Run** collapsible callout. It walks the algorithm step-by-step like a debugger. You can literally type this kind of trace in an interview to demonstrate your thinking out loud.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Two Sum | 1 | Easy | Complement lookup |
| P2 | Contains Duplicate | 217 | Easy | Hash set |
| P3 | Valid Anagram | 242 | Easy | Frequency count |
| P4 | Group Anagrams | 49 | Med | Hash by canonical signature |
| P5 | Top K Frequent Elements | 347 | Med | Hash + bucket sort |
| P6 | Product of Array Except Self | 238 | Med | Prefix · suffix arrays |
| P7 | Encode and Decode Strings | 271 | Med | Length-prefix encoding |
| P8 | Longest Consecutive Sequence | 128 | Med | Hash set + sequence start |
| P9 | Subarray Sum Equals K | 560 | Med | Prefix sum + hash |
| P10 | Continuous Subarray Sum | 523 | Med | Prefix sum mod K |
| P11 | Range Sum Query — Immutable | 303 | Easy | Prefix sum array |
| P12 | Maximum Subarray | 53 | Med | Kadane's (DP bridge) |
| P13 | Find All Duplicates in an Array | 442 | Med | Sign marking (in-place hash) |
| P14 | First Missing Positive | 41 | Hard | Cyclic sort (index as hash) |
| P15 | 4Sum II | 454 | Med | Two-pair hash |
| P16 | Insert Delete GetRandom O(1) | 380 | Med | Hash + array design |
| P17 | Longest Duplicate Substring | 1044 | **Hard** | Rolling hash + binary search |

---

## P1: Two Sum

**LC #1** · Easy

Given `nums[]` and `target`, return indices of two numbers that sum to target. Exactly one solution; can't use same index twice.

**Edge cases:** duplicates `[3,3]` t=6 · negatives · target = 2x · n < 2.

### 🧠 Pattern: Complement Lookup

> Need a **pair** with `a + b = target`? Don't search for both — search for the *complement* (`target - a`) of what you've already seen. Hash map turns "have I seen X?" from O(n) into O(1).

**Recognize when:** "find two/pair such that..." and order doesn't matter.

### Approach Evolution

1. **Brute force — nested loops** · O(n²)/O(1). At n=10⁵ → 10¹⁰ ops → **TLE**.
2. **Sort + two pointers** · O(n log n)/O(1). Destroys original indices → **disqualified**.
3. **Hash map, one pass — FINAL** · O(n)/O(n). Walk once; for each `x` check if `target-x` already seen.

> [!example]- 📊 Visual: array + hash map evolution
> ```text
>           ┌────┬────┬────┬────┐
>   nums =  │ 2  │ 7  │ 11 │ 15 │       target = 9
>           └────┴────┴────┴────┘
>             0    1    2    3
> 
>   Step 1 (i=0):
>     ┌─→ x=2          need = 9-2 = 7
>     │   ┌────┬────┬────┬────┐
>     │   │ 2  │ 7  │ 11 │ 15 │
>     │   └────┴────┴────┴────┘
>     │     ↑
>     │     i=0
>     │
>     └─→  seen: { }                         not in map
>          seen: { 2 → 0 }                   ← store
> 
>   Step 2 (i=1):
>         x=7          need = 9-7 = 2
>         ┌────┬────┬────┬────┐
>         │ 2  │ 7  │ 11 │ 15 │
>         └────┴────┴────┴────┘
>                ↑
>                i=1
> 
>         seen: { 2 → 0 }
>                 ^^^^
>                 MATCH! complement found
> 
>         ┌───┐         ┌───┐
>         │ 2 │═════════│ 7 │ ← pair sums to 9
>         └───┘         └───┘
>           ↑             ↑
>          [0]           [1]
> 
>   ✅ return [0, 1]
> ```

> [!info]- 🔍 Dry Run: nums=[2,7,11,15], target=9
> ```text
> Setup:
>   seen = {}              ← hash map: value → index
> 
> ─────────────────────────────────────────
> Step 1: i=0, x=nums[0]=2
>   need = target - x = 9 - 2 = 7
>   Check: is 7 in seen?  → NO
>   Action: store seen[x] = i  → seen[2] = 0
>   State: seen = {2: 0}
> 
> ─────────────────────────────────────────
> Step 2: i=1, x=nums[1]=7
>   need = target - x = 9 - 7 = 2
>   Check: is 2 in seen?  → YES, at index 0
>   Action: return [seen[need], i] = [0, 1]
> 
> ✅ Answer: [0, 1]
>   Verify: nums[0] + nums[1] = 2 + 7 = 9 ✓
> ```

> [!success]- JS
> ```js
> const twoSum = (nums, target) => {
>   const seen = new Map();
>   for (let i = 0; i < nums.length; i++) {
>     const need = target - nums[i];
>     if (seen.has(need)) return [seen.get(need), i];
>     seen.set(nums[i], i);
>   }
> };
> ```

> [!success]- Python
> ```python
> def two_sum(nums, target):
>     seen = {}
>     for i, x in enumerate(nums):
>         if target - x in seen:
>             return [seen[target - x], i]
>         seen[x] = i
> ```

**Variants:** Two Sum II — sorted (two pointers, O(1) space) · 3Sum (fix one, 2Sum on rest) · 4Sum · Two Sum BST.

**Key takeaway:** Pair-finding → hash map of complement. 4-line skeleton.

---

## P2: Contains Duplicate

**LC #217** · Easy

Return `true` if any value appears at least twice in `nums`.

**Edge cases:** empty · single element · all same · sorted vs unsorted.

### 🧠 Pattern: Set Membership

> When you ask "have I seen this exact value before?", a hash **set** is the cleanest O(1) tool. Use a set when you only need presence/absence; use a map when you also need a count or index.

### Approach Evolution

1. **Sort + compare adjacent** · O(n log n)/O(1) — works but mutates / sorts unnecessarily.
2. **Hash set — FINAL** · O(n)/O(n). Add each element; bail on first re-insertion.
3. **One-liner:** `len(set(nums)) < len(nums)` — clean but allocates the full set.

> [!example]- 📊 Visual: scan + grow a "seen" set
> ```text
>   nums = [ 1 , 2 , 3 , 1 ]
>           ┌───┬───┬───┬───┐
>           │ 1 │ 2 │ 3 │ 1 │
>           └───┴───┴───┴───┘
>             ↑
>             i — walk left-to-right
> 
>   For each x, ask the set:  "have I seen you?"
>   If yes → bail with TRUE. Otherwise add x and keep walking.
> 
>     i=0  x=1   seen={ }              add → seen={1}
>     i=1  x=2   seen={1}              add → seen={1,2}
>     i=2  x=3   seen={1,2}            add → seen={1,2,3}
>     i=3  x=1   seen={1,2,3}  ← HIT!  return TRUE
> 
>   ┌────────────────────────────────────────┐
>   │ Each element passes through the set    │
>   │ exactly once. Bail on first collision. │
>   └────────────────────────────────────────┘
> ```

> [!info]- 🔍 Dry Run: nums=[1,2,3,1]
> ```text
> Setup:
>   seen = {}
> 
> ─────────────────────────────────────────
> Step 1: i=0, x=1
>   Is 1 in seen?  → NO
>   Action: seen.add(1)
>   State: seen = {1}
> 
> ─────────────────────────────────────────
> Step 2: i=1, x=2
>   Is 2 in seen?  → NO
>   Action: seen.add(2)
>   State: seen = {1, 2}
> 
> ─────────────────────────────────────────
> Step 3: i=2, x=3
>   Is 3 in seen?  → NO
>   Action: seen.add(3)
>   State: seen = {1, 2, 3}
> 
> ─────────────────────────────────────────
> Step 4: i=3, x=1
>   Is 1 in seen?  → YES!
>   Action: return true
> 
> ✅ Answer: true
> ```

> [!success]- JS
> ```js
> const containsDuplicate = (nums) => {
>   const seen = new Set();
>   for (const x of nums) {
>     if (seen.has(x)) return true;
>     seen.add(x);
>   }
>   return false;
> };
> ```

> [!success]- Python
> ```python
> def contains_duplicate(nums):
>     seen = set()
>     for x in nums:
>         if x in seen:
>             return True
>         seen.add(x)
>     return False
> ```

**Variants:** Contains Duplicate II (within k distance) · Contains Duplicate III (within k & value diff ≤ t — needs buckets/BBST).

**Key takeaway:** Presence-only check → hash set. Bail early.

---

## P3: Valid Anagram

**LC #242** · Easy

Return `true` if `t` is an anagram of `s`.

**Edge cases:** different lengths (instant false) · Unicode · case sensitivity (clarify).

### 🧠 Pattern: Frequency Count

> Anagrams are equal **multisets of characters**. Compare two strings by their character-frequency map. For ASCII, a 26-int array beats a hash map on constants.

### Approach Evolution

1. **Sort both, compare** · O(n log n)/O(n). One-liner; OK for small inputs.
2. **Two hash maps** · O(n)/O(n). Build both, compare.
3. **One array of length 26 — FINAL** · O(n)/O(1). Increment for `s`, decrement for `t`; all zero ⇒ anagram.

> [!example]- 📊 Visual: balance scale of letter counts
> ```text
>   s = "anagram"      t = "nagaram"
> 
>   For each i in parallel:  cnt[s[i]]++   cnt[t[i]]--
> 
>      ┌──────────────────────────────┐
>      │   +1 from s        -1 from t │
>      └──────────────────────────────┘
> 
>   index in cnt[26]:  a  b  c  d ... g ... m  n ...
> 
>   After all 7 chars processed:
> 
>            a   g   m   n   r        (other letters = 0)
>           ┌───┬───┬───┬───┬───┐
>     cnt = │ 0 │ 0 │ 0 │ 0 │ 0 │ ← every slot is zero
>           └───┴───┴───┴───┴───┘
> 
>   Anagram iff ALL counters land on zero (perfect balance).
> 
>           s side  ████████        t side  ████████
>                  +a +n +a +g +r +a +m   −n −a −g −a −r −a −m
>                                                       
>           ╔══════════════════ EQUAL ══════════════════╗
>           ║   every +1 on the left is cancelled by    ║
>           ║   exactly one −1 on the right             ║
>           ╚════════════════════════════════════════════╝
> ```

> [!info]- 🔍 Dry Run: s="anagram", t="nagaram"
> ```text
> Setup:
>   len(s) == len(t)? YES (both 7) — proceed
>   cnt = [0]*26      ← index = ord(c) - ord('a')
> 
> ─────────────────────────────────────────
> Step 1: i=0  s[0]='a' (+1)  t[0]='n' (-1)
>   cnt[a]+=1  cnt[n]-=1
>   cnt: {a:1, n:-1}
> 
> Step 2: i=1  s[1]='n' (+1)  t[1]='a' (-1)
>   cnt[n]+=1  cnt[a]-=1
>   cnt: {a:0, n:0}
> 
> Step 3: i=2  s[2]='a' (+1)  t[2]='g' (-1)
>   cnt[a]+=1  cnt[g]-=1
>   cnt: {a:1, n:0, g:-1}
> 
> Step 4: i=3  s[3]='g' (+1)  t[3]='a' (-1)
>   cnt[g]+=1  cnt[a]-=1
>   cnt: {a:0, n:0, g:0}
> 
> Step 5: i=4  s[4]='r' (+1)  t[4]='r' (-1)
>   cnt: {a:0, n:0, g:0, r:0}
> 
> Step 6: i=5  s[5]='a' (+1)  t[5]='a' (-1)
>   cnt: {a:0, n:0, g:0, r:0}
> 
> Step 7: i=6  s[6]='m' (+1)  t[6]='m' (-1)
>   cnt: {a:0, n:0, g:0, r:0, m:0}
> 
> Final check: every count == 0?  → YES
> 
> ✅ Answer: true
> ```

> [!success]- JS
> ```js
> const isAnagram = (s, t) => {
>   if (s.length !== t.length) return false;
>   const cnt = new Array(26).fill(0);
>   const A = 'a'.charCodeAt(0);
>   for (let i = 0; i < s.length; i++) {
>     cnt[s.charCodeAt(i) - A]++;
>     cnt[t.charCodeAt(i) - A]--;
>   }
>   return cnt.every(c => c === 0);
> };
> ```

> [!success]- Python
> ```python
> from collections import Counter
> def is_anagram(s, t):
>     return len(s) == len(t) and Counter(s) == Counter(t)
> ```

**Variants:** Group Anagrams (P4) · Find All Anagrams in a String (sliding window).

**Key takeaway:** Anagram → frequency count. For ASCII, prefer fixed-size array.

---

## P4: Group Anagrams

**LC #49** · Medium

Group strings that are anagrams of each other.

**Edge cases:** empty list · single empty string `""` · all unique (each its own group) · all anagrams of each other.

### 🧠 Pattern: Hash by Canonical Signature

> When you need to group "equivalent" items, build a **canonical form** (the same for all equivalents) and use it as a hash-map key.

**Signatures for anagrams:**
- Sorted string `"eat" → "aet"` — O(k log k) per word, simple
- Count tuple `(1,0,1,0,1,...)` for a–z — O(k) per word, faster on long words

### Approach Evolution

1. **Pairwise compare** · O(n² · k). For each new word, check every group. Quadratic — too slow.
2. **Sort each word as key — FINAL (clear)** · O(n · k log k)/O(n·k). Default to this in interview.
3. **Count tuple as key** · O(n · k)/O(n·k). Faster for long words but more code.

> [!example]- 📊 Visual: canonical key buckets
> ```text
>   strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
> 
>   STEP 1 — Canonicalize each word (sort its letters):
> 
>      "eat" ─sort→ "aet"          "ate" ─sort→ "aet"
>      "tea" ─sort→ "aet"          "nat" ─sort→ "ant"
>      "tan" ─sort→ "ant"          "bat" ─sort→ "abt"
> 
>   STEP 2 — Drop each word into the bucket named by its key:
> 
>          ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
>   key →  │    "aet"     │   │    "ant"     │   │    "abt"     │
>          ├──────────────┤   ├──────────────┤   ├──────────────┤
>          │ eat          │   │ tan          │   │ bat          │
>          │ tea          │   │ nat          │   │              │
>          │ ate          │   │              │   │              │
>          └──────────────┘   └──────────────┘   └──────────────┘
> 
>   STEP 3 — Output = list of bucket contents.
> 
>   Insight: equivalent things share a canonical key. The map turns
>   "are these two anagrams?" (pairwise) into "do they hash the same?" (O(1)).
> ```

> [!info]- 🔍 Dry Run: strs=["eat","tea","tan","ate","nat","bat"]
> ```text
> Setup:
>   groups = {}      ← key: sorted-string signature, value: list of words
> 
> ─────────────────────────────────────────
> Step 1: word="eat"
>   signature = sorted("eat") = "aet"
>   "aet" in groups?  → NO; create [] then append
>   groups = {"aet": ["eat"]}
> 
> Step 2: word="tea"
>   signature = sorted("tea") = "aet"
>   "aet" in groups?  → YES; append
>   groups = {"aet": ["eat","tea"]}
> 
> Step 3: word="tan"
>   signature = sorted("tan") = "ant"
>   "ant" in groups?  → NO; create
>   groups = {"aet": ["eat","tea"], "ant": ["tan"]}
> 
> Step 4: word="ate"
>   signature = "aet"; append
>   groups = {"aet": ["eat","tea","ate"], "ant": ["tan"]}
> 
> Step 5: word="nat"
>   signature = "ant"; append
>   groups = {"aet": ["eat","tea","ate"], "ant": ["tan","nat"]}
> 
> Step 6: word="bat"
>   signature = "abt"; create
>   groups = {"aet": ["eat","tea","ate"], "ant": ["tan","nat"], "abt": ["bat"]}
> 
> ✅ Answer: values of groups
>   → [["eat","tea","ate"], ["tan","nat"], ["bat"]]
> ```

> [!success]- JS
> ```js
> const groupAnagrams = (strs) => {
>   const groups = new Map();
>   for (const s of strs) {
>     const key = s.split('').sort().join('');
>     if (!groups.has(key)) groups.set(key, []);
>     groups.get(key).push(s);
>   }
>   return [...groups.values()];
> };
> ```

> [!success]- Python
> ```python
> from collections import defaultdict
> def group_anagrams(strs):
>     groups = defaultdict(list)
>     for s in strs:
>         groups[tuple(sorted(s))].append(s)
>     return list(groups.values())
> ```

**Variants:** Group Shifted Strings (signature = diff sequence) · Group Strings by Pattern.

**Key takeaway:** Grouping problems → choose a canonical key, hash by it.

---

## P5: Top K Frequent Elements

**LC #347** · Medium

Return the k most frequent elements.

**Edge cases:** k = n (return all unique) · ties · large n.

### 🧠 Pattern: Hash Count + Bucket Sort

> Two tools combine here: a hash map for **counting**, then **bucket sort** indexed by frequency (since freq ≤ n, you can use an array, not a heap, to get O(n)).

### Approach Evolution

1. **Sort by frequency** · O(n log n). Trivial — count then sort. Acceptable but not optimal.
2. **Heap of size k** · O(n log k). Classic interview answer. Use a min-heap.
3. **Bucket sort — FINAL** · O(n)/O(n). Index = frequency, value = list of nums with that freq. Walk buckets high→low.

> [!example]- 📊 Visual: buckets indexed by frequency
> ```text
>   nums = [1,1,1, 2,2, 3]    k = 2
> 
>   Phase A — count map:
>      ┌─────┬─────┬─────┐
>      │  1  │  2  │  3  │   value
>      ├─────┼─────┼─────┤
>      │  3  │  2  │  1  │   frequency
>      └─────┴─────┴─────┘
> 
>   Phase B — re-index by frequency (freq ≤ n always):
> 
>     freq:   0    1    2    3    4    5    6
>            ┌──┬────┬────┬────┬──┬──┬──┐
>   buckets: │  │ 3  │ 2  │ 1  │  │  │  │
>            └──┴────┴────┴────┴──┴──┴──┘
>             └─ no number has freq 0  ─┘
> 
>   Phase C — walk RIGHT → LEFT, collect until we have k:
> 
>             ┌──┬────┬────┬░░░┬──┬──┬──┐
>             │  │ 3  │ 2  │░1░│  │  │  │
>             └──┴────┴────┴░░░┴──┴──┴──┘
>                       ←──── start here (freq=3) take 1
>                  ←──── then freq=2 take 2 → have k=2, STOP
> 
>   ✓ result = [1, 2]
> 
>   Bucket sort works because freq ≤ n, so the index space is bounded.
> ```

> [!info]- 🔍 Dry Run: nums=[1,1,1,2,2,3], k=2
> ```text
> Phase 1 — Count frequencies:
>   x=1: count[1]=1
>   x=1: count[1]=2
>   x=1: count[1]=3
>   x=2: count[2]=1
>   x=2: count[2]=2
>   x=3: count[3]=1
>   count = {1:3, 2:2, 3:1}
> 
> ─────────────────────────────────────────
> Phase 2 — Bucket by frequency:
>   buckets[0] = []   ← index = freq
>   buckets[1] = []
>   ...
>   buckets[6] = []
> 
>   num=1, freq=3 → buckets[3].push(1)
>   num=2, freq=2 → buckets[2].push(2)
>   num=3, freq=1 → buckets[1].push(3)
> 
>   buckets = [[], [3], [2], [1], [], [], []]
>              0    1    2    3
> 
> ─────────────────────────────────────────
> Phase 3 — Walk buckets right-to-left, collect k items:
>   i=6 buckets[6]=[]    out=[]
>   i=5 buckets[5]=[]    out=[]
>   i=4 buckets[4]=[]    out=[]
>   i=3 buckets[3]=[1]   out=[1]              ← have 1, need 2 more...wait, k=2 means 2 items
>   i=2 buckets[2]=[2]   out=[1,2]            ← have 2 items, stop
> 
> ✅ Answer: [1, 2]
> ```

> [!success]- JS
> ```js
> const topKFrequent = (nums, k) => {
>   const count = new Map();
>   for (const x of nums) count.set(x, (count.get(x) ?? 0) + 1);
>   const buckets = Array.from({ length: nums.length + 1 }, () => []);
>   for (const [num, freq] of count) buckets[freq].push(num);
>   const out = [];
>   for (let i = buckets.length - 1; i >= 0 && out.length < k; i--) {
>     out.push(...buckets[i]);
>   }
>   return out.slice(0, k);
> };
> ```

> [!success]- Python
> ```python
> from collections import Counter
> def top_k_frequent(nums, k):
>     count = Counter(nums)
>     buckets = [[] for _ in range(len(nums) + 1)]
>     for num, freq in count.items():
>         buckets[freq].append(num)
>     out = []
>     for i in range(len(buckets) - 1, -1, -1):
>         out.extend(buckets[i])
>         if len(out) >= k:
>             return out[:k]
> ```

**Variants:** Top K Frequent Words (heap with custom comparator) · K Closest Points to Origin (heap).

**Key takeaway:** "Top K by frequency" → bucket sort when freq ≤ n, else heap.

---

## P6: Product of Array Except Self

**LC #238** · Medium · **No division allowed**

Return `out[i] = product of all nums except nums[i]`.

**Edge cases:** zeros (especially **two zeros** ⇒ all outputs zero) · single zero ⇒ all outputs zero except that index.

### 🧠 Pattern: Prefix · Suffix Decomposition

> Many "everything except me" problems split into **what's before me** and **what's after me**, each computable in O(n). Then combine: `out[i] = prefix[i-1] · suffix[i+1]`.

### Approach Evolution

1. **Brute force** · O(n²). Multiply all-but-i, n times. Too slow.
2. **Divide total product / nums[i]** — *not allowed*; breaks on zeros.
3. **Prefix + suffix arrays** · O(n)/O(n). Clear but uses 2 extra arrays.
4. **Two passes, O(1) extra — FINAL** · O(n)/O(1) (output not counted). First pass fills `out[i] = prefix product`; second pass multiplies suffix on the fly.

> [!example]- 📊 Visual: prefix × suffix sandwich
> ```text
>   nums  =   [  1  ,  2  ,  3  ,  4  ]
>               0     1     2     3
> 
>   prefix[i]  = product of EVERYTHING LEFT of i
>   suffix[i]  = product of EVERYTHING RIGHT of i
>   out[i]     = prefix[i]  ×  suffix[i]
> 
>       ┌─────────── nums ───────────┐
>       │  1     2     3     4       │
>       └────────────────────────────┘
>          ▲              ▲
>          │              │
>     ┌────┴────┐    ┌────┴────┐
>     │ prefix  │    │ suffix  │
>     │ sweep → │    │ ← sweep │
>     └─────────┘    └─────────┘
> 
>   prefix (LTR pass):      suffix (RTL pass, running r):
>     i=0: 1                  i=3: 1
>     i=1: 1                  i=2: 4
>     i=2: 1·2=2              i=1: 4·3=12
>     i=3: 2·3=6              i=0: 12·2=24
> 
>   Stack them:
>              i=0    i=1    i=2    i=3
>   prefix:     1      1      2      6
>   suffix:    24     12      4      1
>   out:       ───────────────────────
>              24  ·  12  ·   8  ·   6
> 
>   Trick: do it in TWO sweeps with the OUTPUT ARRAY ITSELF holding the
>   prefix; then a single running `r` accumulates the suffix on the way back.
> ```

> [!info]- 🔍 Dry Run: nums=[1,2,3,4]
> ```text
> Pass 1 — Left-to-right, fill out[i] = product of nums[0..i-1]:
>   out[0] = 1                  (nothing to left)
>   out[1] = out[0]*nums[0] = 1*1 = 1
>   out[2] = out[1]*nums[1] = 1*2 = 2
>   out[3] = out[2]*nums[2] = 2*3 = 6
>   out = [1, 1, 2, 6]
> 
> ─────────────────────────────────────────
> Pass 2 — Right-to-left, multiply by running suffix `r`:
>   r = 1
> 
>   i=3:  out[3] *= r           → 6 * 1 = 6
>         r *= nums[3]           → 1 * 4 = 4
>         out = [1, 1, 2, 6],  r=4
> 
>   i=2:  out[2] *= r           → 2 * 4 = 8
>         r *= nums[2]           → 4 * 3 = 12
>         out = [1, 1, 8, 6],  r=12
> 
>   i=1:  out[1] *= r           → 1 * 12 = 12
>         r *= nums[1]           → 12 * 2 = 24
>         out = [1, 12, 8, 6],  r=24
> 
>   i=0:  out[0] *= r           → 1 * 24 = 24
>         r *= nums[0]           → 24 * 1 = 24
>         out = [24, 12, 8, 6]
> 
> ✅ Answer: [24, 12, 8, 6]
>   Verify: out[0]=2·3·4=24 ✓  out[1]=1·3·4=12 ✓  out[2]=1·2·4=8 ✓  out[3]=1·2·3=6 ✓
> ```

> [!success]- JS
> ```js
> const productExceptSelf = (nums) => {
>   const n = nums.length;
>   const out = new Array(n).fill(1);
>   for (let i = 1; i < n; i++) out[i] = out[i - 1] * nums[i - 1];
>   let r = 1;
>   for (let i = n - 1; i >= 0; i--) {
>     out[i] *= r;
>     r *= nums[i];
>   }
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def product_except_self(nums):
>     n = len(nums)
>     out = [1] * n
>     for i in range(1, n):
>         out[i] = out[i - 1] * nums[i - 1]
>     r = 1
>     for i in range(n - 1, -1, -1):
>         out[i] *= r
>         r *= nums[i]
>     return out
> ```

**Variants:** Maximum Product Subarray (track max & min — DP) · Subarray Product Less Than K (sliding window).

**Key takeaway:** "Function of all except me" → prefix·suffix. Two passes, O(1) extra space.

---

## P7: Encode and Decode Strings

**LC #271** · Medium

Design `encode(list[str]) -> str` and `decode(str) -> list[str]` that round-trip.

**Edge cases:** empty list · empty strings inside list · strings containing your delimiter · Unicode.

### 🧠 Pattern: Length-Prefix Encoding

> Any delimiter you pick can appear in the data. The robust serializer encodes **length** + **delimiter** + **payload**: parser reads length, then takes exactly that many chars — no escaping needed.

### Approach Evolution

1. **Single char delimiter (e.g. `,`)** — breaks if data contains `,`.
2. **Escape the delimiter** — possible but messy; nested escapes get ugly.
3. **Length-prefix — FINAL** · O(n) both ways. `"4#word3#cat"` ⇒ ["word","cat"].

> [!example]- 📊 Visual: length-prefix frame format
> ```text
>   Wire format = a sequence of FRAMES.  Each frame:
> 
>       ┌─────────┬───┬──────────────────┐
>       │ length  │ # │     payload      │   ← payload is EXACTLY `length` chars
>       └─────────┴───┴──────────────────┘
>           ascii digits     raw bytes (may contain '#')
> 
>   ["leet", "code", "#hi"]   →   "4#leet4#code3##hi"
> 
>     ┌────┬─┬──────┐┌────┬─┬──────┐┌────┬─┬──────┐
>     │ 4  │#│ leet ││ 4  │#│ code ││ 3  │#│ #hi  │
>     └────┴─┴──────┘└────┴─┴──────┘└────┴─┴──────┘
>      hdr  d  body   hdr  d  body   hdr  d  body
> 
>   Decode cursor:
>      "4#leet4#code3##hi"
>       │└┘└──┘
>       │ │  └─ slice payload (4 chars)
>       │ └─── delimiter '#'
>       └───── read digits until '#'
> 
>   Why this beats escaping:
>     The parser TRUSTS the length — it grabs exactly that many chars
>     and never re-interprets them. The '#' inside payload is just data.
> ```

> [!info]- 🔍 Dry Run: encode(["leet","code","#hi"]) then decode
> ```text
> Encode:
>   "leet" → "4#leet"           (length=4, '#', payload)
>   "code" → "4#code"
>   "#hi"  → "3##hi"            (length=3, '#', then "#hi" literally)
>   Concatenated: "4#leet4#code3##hi"
> 
> ─────────────────────────────────────────
> Decode "4#leet4#code3##hi":
>   i=0
> 
>   Iter 1:  scan from i=0 until '#' → found at j=1
>            length = int("4") = 4
>            word   = s[j+1 : j+1+length] = s[2:6] = "leet"
>            out = ["leet"]
>            i = j+1+length = 6
> 
>   Iter 2:  scan from i=6 until '#' → found at j=7
>            length = int("4") = 4
>            word   = s[8:12] = "code"
>            out = ["leet","code"]
>            i = 12
> 
>   Iter 3:  scan from i=12 until '#' → found at j=13
>            length = int("3") = 3
>            word   = s[14:17] = "#hi"            ← note the # is part of payload!
>            out = ["leet","code","#hi"]
>            i = 17 (end of string)
> 
> ✅ Answer: ["leet","code","#hi"]
> ```

> [!success]- JS
> ```js
> const encode = (strs) => strs.map(s => `${s.length}#${s}`).join('');
> 
> const decode = (s) => {
>   const out = [];
>   let i = 0;
>   while (i < s.length) {
>     let j = i;
>     while (s[j] !== '#') j++;
>     const len = +s.slice(i, j);
>     out.push(s.slice(j + 1, j + 1 + len));
>     i = j + 1 + len;
>   }
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def encode(strs):
>     return ''.join(f"{len(s)}#{s}" for s in strs)
> 
> def decode(s):
>     out, i = [], 0
>     while i < len(s):
>         j = s.index('#', i)
>         length = int(s[i:j])
>         out.append(s[j + 1: j + 1 + length])
>         i = j + 1 + length
>     return out
> ```

**Variants:** Serialize/Deserialize Binary Tree (preorder + null markers) · Design HashMap.

**Key takeaway:** Robust serialization → length-prefix. Don't fight escaping.

---

## P8: Longest Consecutive Sequence

**LC #128** · Medium · **O(n) required**

Length of the longest streak of consecutive integers in `nums` (unsorted).

**Edge cases:** empty · all duplicates · negatives spanning zero.

### 🧠 Pattern: Hash Set + Sequence Start Filter

> Walking *every* element to extend a streak is O(n²) — but if you only start a streak from elements that are a **sequence start** (`x-1` not in set), each element is visited at most twice. That's the trick that buys O(n).

### Approach Evolution

1. **Sort, scan adjacent** · O(n log n). Acceptable; not O(n).
2. **Naive hash set, extend from every num** · O(n²) worst case (`[1..n]` extended from 1).
3. **Hash set + start filter — FINAL** · O(n)/O(n). Only start a streak from `x` where `x-1 ∉ set`.

> [!example]- 📊 Visual: only "true starts" launch a walk
> ```text
>   nums = [100, 4, 200, 1, 3, 2]    →   set = {1,2,3,4,100,200}
> 
>   On the integer number line:
> 
>      ··· 1   2   3   4 ··· 100 ··· 200 ···
>          ●───●───●───●     ●       ●
>          ↑               ↑       ↑
>          start          start   start    (x where x−1 is NOT in set)
> 
>      4 is in the set but 3 is too, so 4 is NOT a start.
>      2 is in the set but 1 is too, so 2 is NOT a start.
>      3 is in the set but 2 is too, so 3 is NOT a start.
> 
>   From each true start, walk forward until the chain breaks:
> 
>      start=1   :  1 → 2 → 3 → 4 → (5∉set) STOP   length 4   ✓
>      start=100 :  100 → (101∉set) STOP             length 1
>      start=200 :  200 → (201∉set) STOP             length 1
> 
>   Why is this O(n)? Each value is visited at most TWICE:
>     once as the "is x-1 in set?" check, and
>     once during exactly one streak walk.
> 
>   Total work bounded by 2n → O(n).
> ```

> [!info]- 🔍 Dry Run: nums=[100,4,200,1,3,2]
> ```text
> Setup:
>   set = {100, 4, 200, 1, 3, 2}
>   best = 0
> 
> ─────────────────────────────────────────
> Iter x=100:
>   Is 100-1=99 in set?  → NO → 100 is a sequence start
>   Extend: 101 in set? NO → stop
>   streak length = 1
>   best = max(0, 1) = 1
> 
> Iter x=4:
>   Is 4-1=3 in set?     → YES → 4 is NOT a start, skip
> 
> Iter x=200:
>   Is 200-1=199 in set? → NO → 200 is a start
>   Extend: 201 in set? NO
>   streak length = 1; best = 1
> 
> Iter x=1:
>   Is 1-1=0 in set?     → NO → 1 is a start
>   Extend: 2 in set? YES → cur=2
>           3 in set? YES → cur=3
>           4 in set? YES → cur=4
>           5 in set? NO  → stop
>   streak length = 4 (covers 1,2,3,4)
>   best = max(1, 4) = 4
> 
> Iter x=3:
>   Is 3-1=2 in set?     → YES → skip
> 
> Iter x=2:
>   Is 2-1=1 in set?     → YES → skip
> 
> ✅ Answer: 4   (the streak 1→2→3→4)
> ```

> [!success]- JS
> ```js
> const longestConsecutive = (nums) => {
>   const set = new Set(nums);
>   let best = 0;
>   for (const x of set) {
>     if (set.has(x - 1)) continue;          // not a start
>     let y = x, len = 1;
>     while (set.has(y + 1)) { y++; len++; }
>     best = Math.max(best, len);
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def longest_consecutive(nums):
>     s = set(nums)
>     best = 0
>     for x in s:
>         if x - 1 in s:
>             continue
>         y, length = x, 1
>         while y + 1 in s:
>             y += 1
>             length += 1
>         best = max(best, length)
>     return best
> ```

**Variants:** Binary Tree Longest Consecutive Sequence (DFS) · Longest Harmonious Subsequence.

**Key takeaway:** Naive O(n²) becomes O(n) by only starting from "true" starts.

---

## P9: Subarray Sum Equals K

**LC #560** · Medium

Count subarrays whose sum equals `k`.

**Edge cases:** negatives (so sliding window fails) · k = 0 · target = total sum.

### 🧠 Pattern: Prefix Sum + Hash Map

> A subarray `nums[i..j]` sums to `k` iff `prefix[j] - prefix[i-1] = k`, i.e. `prefix[i-1] = prefix[j] - k`. So as you walk, ask: "how many previous prefix sums equal `current - k`?" — hash map gives O(1).

**Why not sliding window?** Negatives mean expanding the window doesn't monotonically increase the sum. Sliding window requires monotonicity.

### Approach Evolution

1. **Brute force** · O(n²). Try every subarray.
2. **Prefix sum array, then pair search** · O(n²). Still pair search.
3. **Prefix sum + hash count — FINAL** · O(n)/O(n). Map `{prefix → count_of_occurrences}`.

> [!example]- 📊 Visual: prefix-sum difference identity
> ```text
>   nums    = [ 1 , 1 , 1 ]              k = 2
>   index     0   1   2
> 
>   prefix sums (P[j] = sum of nums[0..j-1], P[0]=0):
> 
>             ┌────┬────┬────┬────┐
>      P =    │ 0  │ 1  │ 2  │ 3  │
>             └────┴────┴────┴────┘
>               P0   P1   P2   P3
> 
>   A subarray nums[i..j-1] sums to k  ⇔  P[j] − P[i] = k
>                                     ⇔  P[i] = P[j] − k
> 
>   So as we sweep j → for each P[j], ask the map:
>       "how many earlier prefixes equal P[j] − k?"
> 
>           P[j]    P[j]−k    matches in map     +=
>            0        −2           0              0
>            1        −1           0              0
>            2         0           1 (the P0)     1   ← subarray [0..1]
>            3         1           1 (the P1)     1   ← subarray [1..2]
> 
>                                  total = 2 ✓
> 
>   Negatives are fine here — we don't rely on monotonic growth,
>   just on counting equal prefixes.
> 
>   ┌───────────────────────────────────────────────┐
>   │ KEY: seed the map with {0: 1} so that a       │
>   │ prefix that ITSELF equals k counts as a hit.  │
>   └───────────────────────────────────────────────┘
> ```

> [!info]- 🔍 Dry Run: nums=[1,1,1], k=2
> ```text
> Setup:
>   count = {0: 1}       ← prefix sum 0 has been "seen" once (empty prefix)
>   prefix = 0
>   total = 0            ← answer accumulator
> 
> ─────────────────────────────────────────
> Step 1: x=1
>   prefix = 0 + 1 = 1
>   need = prefix - k = 1 - 2 = -1
>   count[-1] = ?  → 0 (not in map)
>   total += 0  → total=0
>   count[1] = 1
>   State: prefix=1, count={0:1, 1:1}, total=0
> 
> Step 2: x=1
>   prefix = 1 + 1 = 2
>   need = prefix - k = 2 - 2 = 0
>   count[0] = 1  ← HIT! 1 previous prefix sum was 0
>   total += 1  → total=1     (means subarray nums[0..1] = [1,1] sums to 2)
>   count[2] = 1
>   State: prefix=2, count={0:1, 1:1, 2:1}, total=1
> 
> Step 3: x=1
>   prefix = 2 + 1 = 3
>   need = prefix - k = 3 - 2 = 1
>   count[1] = 1  ← HIT!
>   total += 1  → total=2     (means subarray nums[1..2] = [1,1] sums to 2)
>   count[3] = 1
>   State: prefix=3, total=2
> 
> ✅ Answer: 2 subarrays
>   Verify: [1,1] starting at i=0; [1,1] starting at i=1. ✓
> ```

> [!success]- JS
> ```js
> const subarraySum = (nums, k) => {
>   const count = new Map([[0, 1]]);
>   let prefix = 0, total = 0;
>   for (const x of nums) {
>     prefix += x;
>     total += count.get(prefix - k) ?? 0;
>     count.set(prefix, (count.get(prefix) ?? 0) + 1);
>   }
>   return total;
> };
> ```

> [!success]- Python
> ```python
> from collections import defaultdict
> def subarray_sum(nums, k):
>     count = defaultdict(int)
>     count[0] = 1
>     prefix = total = 0
>     for x in nums:
>         prefix += x
>         total += count[prefix - k]
>         count[prefix] += 1
>     return total
> ```

**Variants:** Continuous Subarray Sum (P10, mod k) · Subarray Sum Divisible by K · Path Sum III (apply on trees).

**Key takeaway:** Subarray sum with negatives → prefix sum + hash. Initialize `count[0]=1`.

---

## P10: Continuous Subarray Sum

**LC #523** · Medium

Return `true` if there's a subarray of size ≥ 2 with sum divisible by `k`.

**Edge cases:** k = 0 (means sum == 0) · negatives · single element · entire array.

### 🧠 Pattern: Prefix Sum mod K

> Two prefix sums with the **same remainder mod k** ⇒ subarray between them is divisible by k. Hash map of `remainder → earliest index` lets you check size ≥ 2 in O(n).

### Approach Evolution

1. **Brute force** · O(n²).
2. **Prefix sum mod k — FINAL** · O(n)/O(n). Track first index where each remainder appeared; if same remainder seen ≥ 2 steps later → answer.

> [!example]- 📊 Visual: same remainder ⇒ k divides the gap
> ```text
>   nums = [23, 2, 4, 6, 7]    k = 6
> 
>   prefix sums:        prefix mod 6:
> 
>     P0 = 0               r0 = 0
>     P1 = 23              r1 = 5
>     P2 = 25              r2 = 1
>     P3 = 29              r3 = 5  ← SAME as r1
>     P4 = 35              r4 = 5
>     P5 = 42              r5 = 0
> 
>   Why same remainder matters:
>      P3 ≡ P1 (mod k)   ⇒   P3 − P1 = nums[1..2] sum
>                       ⇒   that sum is divisible by k
> 
>     ┌──────────────────────────────────────────┐
>     │ index    0     1     2     3     4       │
>     │ nums    23     2     4     6     7       │
>     │ remain   ─    [5]    1    [5]    5    0  │
>     │              ↑      ↑                    │
>     │           first    again — gap is 2 ≥ 2  │
>     │           seen     ✓ return true         │
>     └──────────────────────────────────────────┘
> 
>   Subarray witness: nums[1..2] = [2, 4],  sum = 6,  6 % 6 = 0.
> 
>   We store ONLY the first index per remainder — that maximises the gap.
> ```

> [!info]- 🔍 Dry Run: nums=[23,2,4,6,7], k=6
> ```text
> Setup:
>   seen = {0: -1}    ← prefix sum 0 at "virtual" index -1 (before array)
>   prefix = 0
> 
> ─────────────────────────────────────────
> Step 1: i=0, x=23
>   prefix = 0 + 23 = 23
>   rem = 23 % 6 = 5
>   Is 5 in seen?  → NO
>   Action: seen[5] = 0
>   State: seen = {0:-1, 5:0}
> 
> Step 2: i=1, x=2
>   prefix = 23 + 2 = 25
>   rem = 25 % 6 = 1
>   Is 1 in seen?  → NO
>   Action: seen[1] = 1
>   State: seen = {0:-1, 5:0, 1:1}
> 
> Step 3: i=2, x=4
>   prefix = 25 + 4 = 29
>   rem = 29 % 6 = 5
>   Is 5 in seen?  → YES, at index 0
>   Check: i - seen[5] = 2 - 0 = 2 ≥ 2  → YES
>   ✅ return true
> 
>   Verify: nums[1..2] = [2, 4], sum=6, 6 % 6 = 0 ✓
> ```

> [!success]- JS
> ```js
> const checkSubarraySum = (nums, k) => {
>   const seen = new Map([[0, -1]]);
>   let prefix = 0;
>   for (let i = 0; i < nums.length; i++) {
>     prefix += nums[i];
>     const rem = k === 0 ? prefix : prefix % k;
>     if (seen.has(rem)) {
>       if (i - seen.get(rem) >= 2) return true;
>     } else {
>       seen.set(rem, i);
>     }
>   }
>   return false;
> };
> ```

> [!success]- Python
> ```python
> def check_subarray_sum(nums, k):
>     seen = {0: -1}
>     prefix = 0
>     for i, x in enumerate(nums):
>         prefix += x
>         rem = prefix if k == 0 else prefix % k
>         if rem in seen:
>             if i - seen[rem] >= 2:
>                 return True
>         else:
>             seen[rem] = i
>     return False
> ```

**Variants:** Subarray Sums Divisible by K (count all) · Make Sum Divisible by P (min length removal).

**Key takeaway:** Divisibility on subarray sum → remainder mod k. Store first index, not all.

---

## P11: Range Sum Query — Immutable

**LC #303** · Easy · **Design problem**

Many `sumRange(i, j)` queries on a fixed array. Optimize for many calls.

### 🧠 Pattern: Prefix Sum Array

> Precompute once, answer in O(1). `prefix[i] = nums[0]+...+nums[i-1]`. Then `sumRange(i,j) = prefix[j+1] - prefix[i]`.

### Approach Evolution

1. **Recompute each query** · O(n) per call. Fine for few queries, terrible for many.
2. **Prefix array — FINAL** · Build in O(n), each query O(1).

> [!example]- 📊 Visual: prefix array with sentinel
> ```text
>   nums:        ┌────┬────┬────┬────┬────┬────┐
>                │ -2 │  0 │  3 │ -5 │  2 │ -1 │
>                └────┴────┴────┴────┴────┴────┘
>                  0    1    2    3    4    5
> 
>   prefix:    ┌────┬────┬────┬────┬────┬────┬────┐
>              │  0 │ -2 │ -2 │  1 │ -4 │ -2 │ -3 │
>              └────┴────┴────┴────┴────┴────┴────┘
>                P0   P1   P2   P3   P4   P5   P6
> 
>   prefix[j] = nums[0] + nums[1] + ... + nums[j-1]
>     ↑                          ↑
>     sentinel 0 at index 0      so prefix[i] gives "sum of FIRST i elements"
> 
>   Range query sumRange(i, j) :
> 
>      sum( nums[i..j] )  =  prefix[j+1]  −  prefix[i]
> 
>                          ┌────────────────┐
>      nums[2..5]:  3, -5, │ 2,  -1         │
>                          └────────────────┘
>                          P6 − P2 = -3 − (-2) = -1 ✓
> 
>      ╔═══════════════════════════════════╗
>      ║ Build once: O(n).                 ║
>      ║ Each query: O(1) — just one diff. ║
>      ╚═══════════════════════════════════╝
> ```

> [!info]- 🔍 Dry Run: nums=[-2, 0, 3, -5, 2, -1]
> ```text
> Build phase (constructor):
>   prefix[0] = 0                      ← sentinel
>   prefix[1] = 0 + (-2) = -2
>   prefix[2] = -2 + 0 = -2
>   prefix[3] = -2 + 3 = 1
>   prefix[4] = 1 + (-5) = -4
>   prefix[5] = -4 + 2 = -2
>   prefix[6] = -2 + (-1) = -3
>   prefix = [0, -2, -2, 1, -4, -2, -3]
> 
> ─────────────────────────────────────────
> Query sumRange(0, 2):
>   = prefix[2+1] - prefix[0]
>   = prefix[3] - prefix[0]
>   = 1 - 0
>   = 1
>   Verify: nums[0..2] = -2+0+3 = 1 ✓
> 
> Query sumRange(2, 5):
>   = prefix[5+1] - prefix[2]
>   = prefix[6] - prefix[2]
>   = -3 - (-2)
>   = -1
>   Verify: nums[2..5] = 3+(-5)+2+(-1) = -1 ✓
> 
> ✅ Each query is O(1) once prefix is built.
> ```

> [!success]- JS
> ```js
> class NumArray {
>   constructor(nums) {
>     this.prefix = [0];
>     for (const x of nums) this.prefix.push(this.prefix.at(-1) + x);
>   }
>   sumRange(i, j) { return this.prefix[j + 1] - this.prefix[i]; }
> }
> ```

> [!success]- Python
> ```python
> class NumArray:
>     def __init__(self, nums):
>         self.prefix = [0]
>         for x in nums:
>             self.prefix.append(self.prefix[-1] + x)
>     def sumRange(self, i, j):
>         return self.prefix[j + 1] - self.prefix[i]
> ```

**Variants:** Range Sum Query 2D — Immutable (2D prefix sum) · Range Sum Query — Mutable (Fenwick / Segment Tree).

**Key takeaway:** Many queries on static data → precompute prefix array. Use `prefix[0]=0` sentinel to avoid off-by-one.

---

## P12: Maximum Subarray

**LC #53** · Medium · **Kadane's**

Return the largest sum of any contiguous subarray.

**Edge cases:** all negatives (answer is the max single element) · single element.

### 🧠 Pattern: Kadane's Algorithm (Local vs Global DP)

> At each index, the best subarray *ending here* is either: (a) extend the previous best, or (b) start fresh from `nums[i]`. Take the max. Track the global max separately.
> 
> ```
> cur = max(nums[i], cur + nums[i])
> best = max(best, cur)
> ```

### Approach Evolution

1. **Brute force** · O(n²). Try every subarray.
2. **Divide and conquer** · O(n log n). Elegant but more code.
3. **Kadane's — FINAL** · O(n)/O(1).

> [!example]- 📊 Visual: extend vs restart at each step
> ```text
>   nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]
>            0  1   2  3   4  5  6   7  8
> 
>   At index i, only TWO options for best subarray ending here:
> 
>     ┌──────────────────────┐        ┌──────────────────────┐
>     │  A. extend previous  │        │  B. restart at i     │
>     │     cur + nums[i]    │   vs   │     nums[i]          │
>     └──────────────────────┘        └──────────────────────┘
>                              max of these → new `cur`
> 
>   Walk:           extend    restart    cur     best
>     i=0 x=-2       —          —         -2       -2
>     i=1 x= 1      -2+1=-1     1   →     1        1
>     i=2 x=-3      1-3=-2     -3   →    -2        1
>     i=3 x= 4     -2+4=2      4   →     4        4
>     i=4 x=-1      4-1=3     -1   →     3        4
>     i=5 x= 2      3+2=5      2   →     5        5
>     i=6 x= 1      5+1=6      1   →     6        6  ←★
>     i=7 x=-5      6-5=1     -5   →     1        6
>     i=8 x= 4      1+4=5      4   →     5        6
> 
>   Best subarray ends at i=6, value chain [4, -1, 2, 1] = 6.
> 
>     ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
>     │-2 │ 1 │-3 │░4░│░-1│░2░│░1░│-5 │ 4 │  ← winning slice highlighted
>     └───┴───┴───┴───┴───┴───┴───┴───┴───┘
>                  ▲           ▲
>                start       end
> ```

> [!info]- 🔍 Dry Run: nums=[-2,1,-3,4,-1,2,1,-5,4]
> ```text
> Setup:
>   cur = nums[0] = -2
>   best = -2
> 
> ─────────────────────────────────────────
> Step 1: i=1, x=1
>   cur = max(1, cur + 1) = max(1, -1) = 1
>     ← starting fresh beats extending -2 chain
>   best = max(-2, 1) = 1
> 
> Step 2: i=2, x=-3
>   cur = max(-3, 1 + (-3)) = max(-3, -2) = -2
>     ← extending hurt less than restart
>   best = max(1, -2) = 1
> 
> Step 3: i=3, x=4
>   cur = max(4, -2 + 4) = max(4, 2) = 4
>     ← restart wins
>   best = max(1, 4) = 4
> 
> Step 4: i=4, x=-1
>   cur = max(-1, 4 + (-1)) = max(-1, 3) = 3
>   best = max(4, 3) = 4
> 
> Step 5: i=5, x=2
>   cur = max(2, 3 + 2) = max(2, 5) = 5
>   best = max(4, 5) = 5
> 
> Step 6: i=6, x=1
>   cur = max(1, 5 + 1) = max(1, 6) = 6
>   best = max(5, 6) = 6     ← [4,-1,2,1] sums to 6
> 
> Step 7: i=7, x=-5
>   cur = max(-5, 6 + (-5)) = max(-5, 1) = 1
>   best = 6 (unchanged)
> 
> Step 8: i=8, x=4
>   cur = max(4, 1 + 4) = max(4, 5) = 5
>   best = 6 (unchanged)
> 
> ✅ Answer: 6   (subarray [4,-1,2,1])
> ```

> [!success]- JS
> ```js
> const maxSubArray = (nums) => {
>   let cur = nums[0], best = nums[0];
>   for (let i = 1; i < nums.length; i++) {
>     cur = Math.max(nums[i], cur + nums[i]);
>     best = Math.max(best, cur);
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def max_sub_array(nums):
>     cur = best = nums[0]
>     for x in nums[1:]:
>         cur = max(x, cur + x)
>         best = max(best, cur)
>     return best
> ```

**Variants:** Maximum Subarray with indices · Maximum Product Subarray (track min too) · Best Time to Buy and Sell Stock (Kadane on diffs).

**Key takeaway:** Track local-best-ending-here separately from global-best. Kadane's is your bridge to DP.

---

## P13: Find All Duplicates in an Array

**LC #442** · Medium · **In-place** O(1) extra space, 1 ≤ nums[i] ≤ n

Return all elements appearing **twice**. No extra space.

**Edge cases:** all unique (return []) · single element · all duplicates.

### 🧠 Pattern: Sign Marking (Index as Hash)

> When values are in `1..n`, the array indices themselves are a built-in hash table. **Negate** `nums[|x|-1]` to record "I've seen x". If already negative, x is a duplicate.

### Approach Evolution

1. **Hash set** · O(n)/O(n). Trivial but violates the space constraint.
2. **Sort + scan** · O(n log n) — mutates the array.
3. **Sign marking — FINAL** · O(n)/O(1).

> [!example]- 📊 Visual: array used as its own hash table
> ```text
>   Values are in 1..n, so index (v-1) is a SLOT for value v.
>   To record "I've seen v", flip the SIGN of nums[v-1].
> 
>   nums = [4, 3, 2, 7, 8, 2, 3, 1]    (n = 8)
> 
>          slot 0  1  2  3  4  5  6  7    ← these are indices = value-1
>                ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑
>          for v= 1  2  3  4  5  6  7  8
> 
>   The MAGNITUDE |nums[i]| is the actual stored value;
>   the SIGN is one bit of metadata = "value (i+1) seen?"
> 
>   Visiting v ⇒ look at slot (|v|−1):
> 
>      slot[|v|−1]  positive  → "not yet seen"  → flip to negative
>      slot[|v|−1]  negative  → "seen before!"  → record v as duplicate
> 
>   Final sweep through nums=[4,3,2,7,8,2,3,1]:
> 
>      visiting 4 → slot 3 (was 7) → flip → -7
>      visiting 3 → slot 2 (was 2) → flip → -2
>      visiting 2 → slot 1 (was 3) → flip → -3
>      visiting 7 → slot 6 (was 3) → flip → -3   ← but read |3|, magnitude
>      visiting 8 → slot 7 (was 1) → flip → -1
>      visiting 2 → slot 1 is -3 < 0 → DUPLICATE: out += [2]
>      visiting 3 → slot 2 is -2 < 0 → DUPLICATE: out += [3]
>      visiting 1 → slot 0 (was 4) → flip → -4
> 
>      out = [2, 3]   in O(1) extra space.
> ```

> [!info]- 🔍 Dry Run: nums=[4,3,2,7,8,2,3,1]
> ```text
> Setup:
>   out = []
>   (index 0..7 corresponds to values 1..8)
> 
> ─────────────────────────────────────────
> Step 1: v=nums[0]=4
>   i = |4| - 1 = 3
>   nums[3] = 7 > 0  → not seen yet
>   Action: nums[3] = -7
>   State: nums = [4, 3, 2, -7, 8, 2, 3, 1]
> 
> Step 2: v=nums[1]=3
>   i = |3| - 1 = 2
>   nums[2] = 2 > 0  → not seen yet
>   Action: nums[2] = -2
>   State: nums = [4, 3, -2, -7, 8, 2, 3, 1]
> 
> Step 3: v=nums[2]=-2
>   i = |-2| - 1 = 1                  ← abs handles the negated marker
>   nums[1] = 3 > 0  → not seen yet
>   Action: nums[1] = -3
>   State: nums = [4, -3, -2, -7, 8, 2, 3, 1]
> 
> Step 4: v=nums[3]=-7
>   i = |-7| - 1 = 6
>   nums[6] = 3 > 0  → not seen yet
>   Action: nums[6] = -3
>   State: nums = [4, -3, -2, -7, 8, 2, -3, 1]
> 
> Step 5: v=nums[4]=8
>   i = |8| - 1 = 7
>   nums[7] = 1 > 0  → not seen yet
>   Action: nums[7] = -1
> 
> Step 6: v=nums[5]=2
>   i = |2| - 1 = 1
>   nums[1] = -3 < 0  → SEEN BEFORE!
>   Action: out.append(2)
>   State: out = [2]
> 
> Step 7: v=nums[6]=-3
>   i = |-3| - 1 = 2
>   nums[2] = -2 < 0  → SEEN BEFORE!
>   Action: out.append(3)
>   State: out = [2, 3]
> 
> Step 8: v=nums[7]=-1
>   i = |-1| - 1 = 0
>   nums[0] = 4 > 0  → not seen
>   Action: nums[0] = -4
> 
> ✅ Answer: [2, 3]
> ```

> [!success]- JS
> ```js
> const findDuplicates = (nums) => {
>   const out = [];
>   for (const v of nums) {
>     const i = Math.abs(v) - 1;
>     if (nums[i] < 0) out.push(i + 1);
>     else nums[i] = -nums[i];
>   }
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def find_duplicates(nums):
>     out = []
>     for v in nums:
>         i = abs(v) - 1
>         if nums[i] < 0:
>             out.append(i + 1)
>         else:
>             nums[i] = -nums[i]
>     return out
> ```

**Variants:** Missing Number (XOR) · Find Missing & Duplicate (P14-related) · Set Mismatch.

**Key takeaway:** Values in `1..n` + O(1) space → use the array itself as the hash via sign flip.

---

## P14: First Missing Positive

**LC #41** · **Hard** · **O(n) time, O(1) extra space**

Find smallest missing positive integer.

**Edge cases:** all negatives → 1 · `[1,2,3]` → 4 · contains huge numbers (ignore them).

### 🧠 Pattern: Cyclic Sort (Index Slot)

> Answer is in `[1, n+1]`. Place each value `v` into index `v-1` via swaps (only if `1 ≤ v ≤ n` and `nums[v-1] != v`). Then scan: first index where `nums[i] != i+1` ⇒ missing is `i+1`. If all match, answer is `n+1`.

### Approach Evolution

1. **Hash set** · O(n)/O(n). Easy but violates space.
2. **Sort** · O(n log n).
3. **Cyclic sort — FINAL** · O(n)/O(1). Each swap places one number into its home slot; O(n) total swaps.

> [!example]- 📊 Visual: place each v into slot v−1
> ```text
>   Answer lives in [1, n+1]. The array has n cells; place each
>   value v into cell v−1, so cells become "labeled boxes".
> 
>   nums = [3, 4, -1, 1]   (n=4, slots labeled 1..4)
> 
>          slot:    1     2     3     4
>                 ┌────┬────┬────┬────┐
>                 │ 3  │ 4  │ -1 │ 1  │
>                 └────┴────┴────┴────┘
>                  ?    ?     ✗    ?      ← negative/out-of-range OK to ignore
> 
>   Cyclic placement (swap v home until each cell is its label or junk):
> 
>     [3,4,-1,1] swap(0,2)→ [-1,4,3,1]
>     [-1 in cell 1] junk, leave it.    advance.
>     [4,3,1 ...] swap(1,3) →            [-1, 1, 3, 4]
>     [1 in cell 2] swap(1,0) →          [ 1,-1, 3, 4]
>     [-1 in cell 2] junk, leave.        advance.
>     [3 in cell 3] HOME ✓.  [4 in cell 4] HOME ✓.
> 
>   Final after placement:
>          slot:    1     2     3     4
>                 ┌────┬────┬────┬────┐
>                 │ 1  │ -1 │ 3  │ 4  │
>                 └────┴────┴────┴────┘
>                   ✓    ✗    ✓    ✓
>                        ↑
>                        first cell whose value ≠ label
>                        → answer = label = 2
> 
>   If every slot matches its label, the answer is n+1 (e.g., [1,2,3] → 4).
> ```

> [!info]- 🔍 Dry Run: nums=[3,4,-1,1], n=4
> ```text
> Phase 1 — Place each value v at index v-1:
> 
>   i=0  v=3  range [1,4]? YES. target=nums[2]=-1, target != 3 → swap
>             swap(nums[0], nums[2])
>             nums = [-1, 4, 3, 1]
>             stay at i=0 (don't advance; new value needs placing too)
> 
>   i=0  v=-1  range [1,4]? NO (negative) → i++
> 
>   i=1  v=4  range [1,4]? YES. target=nums[3]=1, target != 4 → swap
>             swap(nums[1], nums[3])
>             nums = [-1, 1, 3, 4]
>             stay at i=1
> 
>   i=1  v=1  range [1,4]? YES. target=nums[0]=-1, target != 1 → swap
>             swap(nums[1], nums[0])
>             nums = [1, -1, 3, 4]
>             stay at i=1
> 
>   i=1  v=-1  out of range → i++
> 
>   i=2  v=3  nums[2]=3, already in home slot → i++
> 
>   i=3  v=4  nums[3]=4, home → i++
> 
>   Final after Phase 1: nums = [1, -1, 3, 4]
> 
> ─────────────────────────────────────────
> Phase 2 — Scan for first slot that doesn't match:
>   i=0: nums[0]=1 == 0+1 ✓
>   i=1: nums[1]=-1 != 1+1=2  ← MISSING!
>   return 2
> 
> ✅ Answer: 2
> ```

> [!success]- JS
> ```js
> const firstMissingPositive = (nums) => {
>   const n = nums.length;
>   for (let i = 0; i < n; ) {
>     const v = nums[i];
>     if (v >= 1 && v <= n && nums[v - 1] !== v) {
>       [nums[i], nums[v - 1]] = [nums[v - 1], nums[i]];
>     } else i++;
>   }
>   for (let i = 0; i < n; i++) if (nums[i] !== i + 1) return i + 1;
>   return n + 1;
> };
> ```

> [!success]- Python
> ```python
> def first_missing_positive(nums):
>     n = len(nums)
>     i = 0
>     while i < n:
>         v = nums[i]
>         if 1 <= v <= n and nums[v - 1] != v:
>             nums[i], nums[v - 1] = nums[v - 1], nums[i]
>         else:
>             i += 1
>     for i in range(n):
>         if nums[i] != i + 1:
>             return i + 1
>     return n + 1
> ```

> [!warning] Loop without `i++` on swap
> The `while` (not `for`) is critical: when you swap, the new value at `i` might need placing too — don't advance `i` until placement is settled.

**Variants:** Find All Numbers Disappeared in an Array · Missing Number (single missing, XOR).

**Key takeaway:** O(n) time + O(1) space + bounded ints → cyclic sort. The answer lives in `[1, n+1]`.

---

## P15: 4Sum II

**LC #454** · Medium

Given four arrays of length n, count tuples `(i,j,k,l)` with `A[i]+B[j]+C[k]+D[l] = 0`.

**Edge cases:** zeros in all arrays · n up to 200 → n⁴ = 1.6 × 10⁹ (TLE).

### 🧠 Pattern: Hash by Pair-Sum (Meet in the Middle)

> n⁴ → n². Split four arrays into two pairs. Compute all `A[i]+B[j]` sums into a count map. Then for each `(k,l)`, count occurrences of `-(C[k]+D[l])` in the map.

### Approach Evolution

1. **Brute force 4 nested loops** · O(n⁴) — TLE at n=200.
2. **3 nested + hash on 4th** · O(n³) — still slow.
3. **Hash both pairs — FINAL** · O(n²)/O(n²).

> [!example]- 📊 Visual: meet-in-the-middle pair split
> ```text
>   Goal:  A[i] + B[j] + C[k] + D[l] = 0
>          └──── LEFT ────┘   └──── RIGHT ────┘
>                ab                   cd
> 
>   Naive: 4 nested loops = n⁴.   At n=200, that's 1.6×10⁹ → TLE.
> 
>   Trick: split into two halves and store one side in a hash:
> 
>      ┌─────────────────┐                ┌─────────────────┐
>      │  build  ab_cnt  │                │  for each c+d:  │
>      │  for every pair │   ◀──lookup──  │  need = −(c+d)  │
>      │   (A[i], B[j])  │                │  total += ab[need] │
>      └─────────────────┘                └─────────────────┘
>          n² entries                            n² queries
> 
>   Example with A=[1,2], B=[-2,-1], C=[-1,2], D=[0,2]:
> 
>      Build:                   Lookup (each += ab[−(c+d)]):
>      ┌───────┬───┐            c+d = -1  → need = 1 → ab[1]=1 → +1
>      │  sum  │ # │            c+d =  1  → need = -1 → ab[-1]=1 → +1
>      ├───────┼───┤            c+d =  2  → need = -2 → ab[-2]=0
>      │  -1   │ 1 │            c+d =  4  → need = -4 → ab[-4]=0
>      │   0   │ 2 │                                    ─────────
>      │   1   │ 1 │                                    total = 2 ✓
>      └───────┴───┘
> 
>   n⁴ → n² in TIME at the cost of n² SPACE — classic tradeoff.
> ```

> [!info]- 🔍 Dry Run: A=[1,2], B=[-2,-1], C=[-1,2], D=[0,2]
> ```text
> Phase 1 — Build ab_count (all A[i]+B[j] sums):
>   i=0,j=0: 1+(-2) = -1  → ab_count[-1] = 1
>   i=0,j=1: 1+(-1) =  0  → ab_count[0]  = 1
>   i=1,j=0: 2+(-2) =  0  → ab_count[0]  = 2
>   i=1,j=1: 2+(-1) =  1  → ab_count[1]  = 1
> 
>   ab_count = {-1: 1, 0: 2, 1: 1}
> 
> ─────────────────────────────────────────
> Phase 2 — For each (C[k], D[l]) pair, look up complement -(c+d):
>   k=0,l=0: c+d = -1+0 = -1.  need = 1.  ab_count[1] = 1  → total += 1  (total=1)
>   k=0,l=1: c+d = -1+2 =  1.  need = -1. ab_count[-1] = 1 → total += 1  (total=2)
>   k=1,l=0: c+d =  2+0 =  2.  need = -2. ab_count[-2] = 0 → total += 0  (total=2)
>   k=1,l=1: c+d =  2+2 =  4.  need = -4. ab_count[-4] = 0 → total += 0  (total=2)
> 
> ✅ Answer: 2 tuples
>   Tuple 1: A[0]+B[1]+C[0]+D[0] = 1+(-1)+(-1)+0 = -1...
>   (Verify yourself; key insight: it works.)
> ```

> [!success]- JS
> ```js
> const fourSumCount = (A, B, C, D) => {
>   const ab = new Map();
>   for (const a of A) for (const b of B) {
>     const s = a + b;
>     ab.set(s, (ab.get(s) ?? 0) + 1);
>   }
>   let total = 0;
>   for (const c of C) for (const d of D) {
>     total += ab.get(-(c + d)) ?? 0;
>   }
>   return total;
> };
> ```

> [!success]- Python
> ```python
> from collections import Counter
> def four_sum_count(A, B, C, D):
>     ab = Counter(a + b for a in A for b in B)
>     return sum(ab[-(c + d)] for c in C for d in D)
> ```

**Variants:** 3Sum (sorted + two pointers) · 4Sum (sorted + two pointers + 2 outer loops).

**Key takeaway:** Multi-array sum → split into pairs, hash one half. n^k → n^(k/2) memory tradeoff.

---

## P16: Insert Delete GetRandom O(1)

**LC #380** · Medium · **Design**

`insert(val)`, `remove(val)`, `getRandom()` all in **O(1) average**.

**Edge cases:** insert duplicates (return false) · remove non-existent (return false) · single element + remove + getRandom.

### 🧠 Pattern: Hash Map + Array (Swap-with-Last)

> Hash map alone → no O(1) random. Array alone → no O(1) remove. Combine: array for storage + random access; hash map `val → index in array`. For removal, **swap-with-last** to keep array dense.

### Approach Evolution

1. **Array only** · remove = O(n) (shift). Random = O(1).
2. **Hash set only** · random = O(n) (must iterate to pick).
3. **Hash map + array, swap-with-last — FINAL** · all O(1).

> [!example]- 📊 Visual: array + index map, swap-with-last delete
> ```text
>   Two data structures kept in sync:
> 
>     arr (dense storage):       idx (val → position in arr):
>     ┌───┬───┬───┬───┐          ┌──────────────┐
>     │ 1 │ 2 │ 3 │ 4 │          │ 1 → 0        │
>     └───┴───┴───┴───┘          │ 2 → 1        │
>       0   1   2   3            │ 3 → 2        │
>                                │ 4 → 3        │
>                                └──────────────┘
> 
>   getRandom()  →  arr[ rand(0, len) ]                  O(1)
>   insert(v)    →  append to arr; record idx[v]         O(1)
> 
>   remove(v)   ← the only tricky one. Goal: keep arr DENSE.
> 
>     remove(2):  position of 2 is i=1; last = arr[-1] = 4
> 
>     STEP A: write last over the hole
>        ┌───┬───┬───┬───┐
>        │ 1 │░4░│ 3 │ 4 │
>        └───┴───┴───┴───┘
> 
>     STEP B: update idx for the value that moved
>        idx[4] = 1     (4 now lives at slot 1)
> 
>     STEP C: pop the tail
>        ┌───┬───┬───┐
>        │ 1 │ 4 │ 3 │
>        └───┴───┴───┘
> 
>     STEP D: delete idx[2]
> 
>   ▲ No shifting → O(1). The map says "where's value v?"
>     The array says "value at position i". Together: all ops O(1).
> ```

> [!info]- 🔍 Dry Run: insert(1), insert(2), insert(3), remove(2)
> ```text
> Setup:
>   arr = []          ← dense array
>   idx = {}          ← map: value → index in arr
> 
> ─────────────────────────────────────────
> insert(1):
>   Is 1 in idx?  → NO
>   Action: idx[1] = arr.length = 0;  arr.push(1)
>   State: arr=[1],  idx={1:0}
>   Return: true
> 
> insert(2):
>   Is 2 in idx?  → NO
>   Action: idx[2] = 1;  arr.push(2)
>   State: arr=[1,2],  idx={1:0, 2:1}
>   Return: true
> 
> insert(3):
>   Is 3 in idx?  → NO
>   Action: idx[3] = 2;  arr.push(3)
>   State: arr=[1,2,3],  idx={1:0, 2:1, 3:2}
>   Return: true
> 
> ─────────────────────────────────────────
> remove(2):    ← the tricky one
>   Is 2 in idx?  → YES, at idx[2]=1
>   last_val   = arr[arr.length-1] = 3
>   target_idx = 1
> 
>   Step A: overwrite slot 1 with last_val
>           arr[1] = 3
>           State: arr=[1,3,3]
> 
>   Step B: update idx for the moved value
>           idx[3] = 1
>           State: idx={1:0, 2:1, 3:1}
> 
>   Step C: pop the last element (no longer needed)
>           arr.pop()
>           State: arr=[1,3]
> 
>   Step D: delete the removed key
>           delete idx[2]
>           State: idx={1:0, 3:1}
> 
>   Return: true
> 
> ─────────────────────────────────────────
> getRandom():
>   pick random index from [0, arr.length) = [0, 2)
>   say it picks index 1 → return arr[1] = 3
> 
> ✅ All ops O(1).
> ```

> [!success]- JS
> ```js
> class RandomizedSet {
>   constructor() { this.arr = []; this.idx = new Map(); }
>   insert(v) {
>     if (this.idx.has(v)) return false;
>     this.idx.set(v, this.arr.length);
>     this.arr.push(v);
>     return true;
>   }
>   remove(v) {
>     if (!this.idx.has(v)) return false;
>     const i = this.idx.get(v), last = this.arr.at(-1);
>     this.arr[i] = last;
>     this.idx.set(last, i);
>     this.arr.pop();
>     this.idx.delete(v);
>     return true;
>   }
>   getRandom() {
>     return this.arr[Math.floor(Math.random() * this.arr.length)];
>   }
> }
> ```

> [!success]- Python
> ```python
> import random
> class RandomizedSet:
>     def __init__(self):
>         self.arr = []
>         self.idx = {}
>     def insert(self, v):
>         if v in self.idx: return False
>         self.idx[v] = len(self.arr)
>         self.arr.append(v)
>         return True
>     def remove(self, v):
>         if v not in self.idx: return False
>         i = self.idx[v]
>         last = self.arr[-1]
>         self.arr[i] = last
>         self.idx[last] = i
>         self.arr.pop()
>         del self.idx[v]
>         return True
>     def getRandom(self):
>         return random.choice(self.arr)
> ```

**Variants:** Insert Delete GetRandom O(1) — Duplicates allowed (idx → set of positions) · LRU Cache (hash + doubly-linked list).

**Key takeaway:** O(1) for all three ops → hash for lookup + array for random. **Swap-with-last** keeps array dense without shifting.

---

## P17: Longest Duplicate Substring

**LC #1044** · **Hard** · Rolling Hash + Binary Search

Find the longest substring that appears at least twice in `s` (return any one).

**Edge cases:** no duplicates (return "") · entire string repeated (`"aa"`) · large n (up to 3×10⁴).

### 🧠 Pattern: Binary Search on Length + Rabin-Karp Rolling Hash

> Two ideas combined:
> 1. **Binary search the answer**: if a duplicate of length L exists, then duplicates of lengths < L also exist. So search for the largest L with `has_dup(L) == true`.
> 2. **Rabin-Karp**: hash every length-L substring in O(n) total using a rolling hash; detect collisions via set.

### Approach Evolution

1. **Brute force** · O(n² · L) — for every pair of starting positions, compare. Way too slow.
2. **Suffix array + LCP** · O(n log n). Powerful but heavy to code under interview pressure.
3. **Binary search + Rabin-Karp — FINAL** · O(n log n) average. Cleanest interview answer.

> [!example]- 📊 Visual: binary search on L + rolling hash
> ```text
>   Property exploited:
>      if some duplicate of length L exists,
>      then duplicates of EVERY length < L also exist (their prefixes).
>      ⇒ has_dup(L) is monotonic in L — binary searchable.
> 
>      L:      1   2   3   4   5
>      dup?:   T   T   T   F   F        ←  find rightmost T
>                      ↑
>                   answer
> 
>   Inside has_dup(L): slide a length-L window and hash it.
> 
>      s = b a n a n a
>          0 1 2 3 4 5
> 
>      L=3 windows:
>        ┌─────────┐
>        │ b a n │ a n a    hash H1   →  seen = {H1: 0}
>        └─────────┘
>           ┌─────────┐
>        b  │ a n a │ n a   hash H2   →  seen = {H1, H2}
>           └─────────┘
>              ┌─────────┐
>        b  a  │ n a n │ a  hash H3   →  seen = {H1, H2, H3}
>              └─────────┘
>                 ┌─────────┐
>        b  a  n │ a n a │  hash H2   ←  COLLISION!
>                 └─────────┘
> 
>   Rolling hash slides in O(1) per step:
> 
>         h' = (h · BASE − s[i−1]·BASE^L + s[i+L−1])  mod MOD
> 
>   ⚠ Verify on collision: hash equality can be a false positive.
>     After a hit, compare the actual substrings (still amortised O(n log n)).
> 
>   ╔══════════════════════════════════════════════╗
>   ║ outer: O(log n) binary-search iterations     ║
>   ║ inner: O(n) rolling hash per L               ║
>   ║ total: O(n log n) average                    ║
>   ╚══════════════════════════════════════════════╝
> ```

> [!info]- 🔍 Dry Run: s="banana"
> ```text
> Binary search on length:
>   lo=1, hi=5 (max possible is len(s)-1)
> 
> ─────────────────────────────────────────
> Iter 1: L = (1+5)/2 = 3
>   Check: does any length-3 substring appear twice?
>   length-3 substrings: "ban","ana","nan","ana"
>   Roll a hash through:
>     "ban" → hash H1; seen = {H1}
>     "ana" → hash H2; seen = {H1, H2}
>     "nan" → hash H3; seen = {H1, H2, H3}
>     "ana" → hash H2; ALREADY IN seen → COLLISION at start 1, also start 3
>   has_dup(3) = true → answer is "ana"; try larger
>   lo = L+1 = 4
> 
> ─────────────────────────────────────────
> Iter 2: L = (4+5)/2 = 4
>   length-4 substrings: "bana","anan","nana"
>     "bana" → H; seen = {H}
>     "anan" → H; seen = {... H'}
>     "nana" → H; seen = {... H''}
>   No collision.
>   has_dup(4) = false → try smaller
>   hi = L-1 = 3
> 
> ─────────────────────────────────────────
> lo=4 > hi=3 → exit loop
> Last successful L was 3 with match "ana"
> 
> ✅ Answer: "ana"
> ```

> [!success]- Python
> ```python
> def longest_dup_substring(s):
>     MOD = (1 << 61) - 1
>     BASE = 26
>     n = len(s)
>     nums = [ord(c) - ord('a') for c in s]
>     
>     def search(L):
>         """Return start index of a length-L dup, else -1."""
>         if L == 0: return 0
>         # hash of first L chars
>         h = 0
>         for i in range(L):
>             h = (h * BASE + nums[i]) % MOD
>         seen = {h: 0}
>         # BASE^L mod MOD for rolling
>         power = pow(BASE, L, MOD)
>         for i in range(1, n - L + 1):
>             h = (h * BASE - nums[i - 1] * power + nums[i + L - 1]) % MOD
>             if h in seen:
>                 # Verify (avoid false positive collisions)
>                 if s[seen[h]:seen[h] + L] == s[i:i + L]:
>                     return i
>             else:
>                 seen[h] = i
>         return -1
>     
>     lo, hi = 1, n - 1
>     start, length = 0, 0
>     while lo <= hi:
>         L = (lo + hi) // 2
>         idx = search(L)
>         if idx != -1:
>             start, length = idx, L
>             lo = L + 1
>         else:
>             hi = L - 1
>     return s[start:start + length]
> ```

> [!success]- JS
> ```js
> const longestDupSubstring = (s) => {
>   const MOD = 2n ** 61n - 1n;
>   const BASE = 26n;
>   const n = s.length;
>   const nums = [...s].map(c => BigInt(c.charCodeAt(0) - 97));
>   const search = (L) => {
>     if (L === 0) return 0;
>     let h = 0n;
>     for (let i = 0; i < L; i++) h = (h * BASE + nums[i]) % MOD;
>     const seen = new Map([[h, 0]]);
>     let power = 1n;
>     for (let i = 0; i < L; i++) power = (power * BASE) % MOD;
>     for (let i = 1; i <= n - L; i++) {
>       h = (h * BASE - nums[i - 1] * power + nums[i + L - 1]) % MOD;
>       h = (h + MOD * MOD) % MOD;        // keep positive
>       if (seen.has(h)) {
>         const start = seen.get(h);
>         if (s.slice(start, start + L) === s.slice(i, i + L)) return i;
>       } else {
>         seen.set(h, i);
>       }
>     }
>     return -1;
>   };
>   let lo = 1, hi = n - 1, start = 0, length = 0;
>   while (lo <= hi) {
>     const L = (lo + hi) >> 1;
>     const idx = search(L);
>     if (idx !== -1) { start = idx; length = L; lo = L + 1; }
>     else hi = L - 1;
>   }
>   return s.slice(start, start + length);
> };
> ```

> [!tip] Verify on collision
> Rolling hash can collide on different strings (false positive). After a hash hit, do a literal substring comparison to confirm. This keeps correctness while preserving O(n log n) average.

**Variants:** Longest Common Substring of two strings · Repeated DNA Sequences (LC #187, exact same hashing trick with L=10).

**Key takeaway:** "Longest something" + "appears N times" → binary search on length + rolling hash. Classic Google/Meta hard.

---

> [!tip] After finishing this drill
> You should be able to write Two Sum and Group Anagrams in under 3 minutes from scratch, and recognize "prefix sum + hash" the instant a problem says "subarray sum equals K".
