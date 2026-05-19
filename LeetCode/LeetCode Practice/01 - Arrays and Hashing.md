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

16 progressive problems · drill the **Complement Lookup** and **Prefix Sum** patterns.

> [!abstract] Pattern recap
> **Complement Lookup** — for pair/triplet problems, store what you've seen in a hash map and look up `target - x` in O(1).
> **Prefix Sum** — replace range/subarray sum questions with O(1) lookups via cumulative sums; combine with hash map for "subarray summing to K".

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

---

## P1: Two Sum

**LC #1** · Easy

Given `nums[]` and `target`, return indices of two numbers that sum to target. Exactly one solution; can't use same index twice.

**Edge cases:** duplicates `[3,3]` t=6 · negatives · target = 2x · n < 2.

---

### 🧠 Pattern: Complement Lookup

> Need a **pair** with `a + b = target`? Don't search for both — search for the *complement* (`target - a`) of what you've already seen. Hash map turns "have I seen X?" from O(n) into O(1).

**Recognize when:** "find two/pair such that..." and order doesn't matter.

### Approach Evolution

1. **Brute force — nested loops** · O(n²)/O(1). At n=10⁵ → 10¹⁰ ops → **TLE**.
2. **Sort + two pointers** · O(n log n)/O(1). Destroys original indices → **disqualified**.
3. **Hash map, one pass — FINAL** · O(n)/O(n). Walk once; for each `x` check if `target-x` already seen.

### Trace

```
nums=[2,7,11,15] target=9
i=0 x=2  need=7  seen={}        → seen={2:0}
i=1 x=7  need=2  seen={2:0} HIT → return [0,1]
```

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

### Trace

```
nums=[1,2,3,1]
seen={}      add 1 → {1}
seen={1}     add 2 → {1,2}
seen={1,2}   add 3 → {1,2,3}
seen={1,2,3} 1 in seen? YES → return true
```

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

### Trace

```
s="anagram"  t="nagaram"
counts (a-z, only nonzero):
  after s: a:3 g:1 m:1 n:1 r:1
  after t: a:0 g:0 m:0 n:0 r:0   → true
```

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

**Variants:** Group Anagrams (P4) · Find All Anagrams in a String (sliding window) · Valid Anagram with Unicode (must use map).

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

### Trace

```
strs=["eat","tea","tan","ate","nat","bat"]
sig("eat")="aet" → groups={aet:["eat"]}
sig("tea")="aet" → groups={aet:["eat","tea"]}
sig("tan")="ant" → groups={aet:[...], ant:["tan"]}
...
result = list of groups.values()
```

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

### Trace

```
nums=[1,1,1,2,2,3]  k=2
count = {1:3, 2:2, 3:1}
buckets index = freq:
  buckets[1]=[3]  buckets[2]=[2]  buckets[3]=[1]
Walk i=n..1, collect 1 then 2 → [1,2]
```

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

### Trace

```
nums = [1, 2, 3, 4]

Pass 1 (prefix into out):
  out = [1, 1, 2, 6]   (out[i] = product of nums[0..i-1])

Pass 2 (suffix as scalar `r`, right-to-left):
  r=1  out[3]=6*1=6   r=4
  r=4  out[2]=2*4=8   r=12
  r=12 out[1]=1*12=12 r=24
  r=24 out[0]=1*24=24 

result = [24, 12, 8, 6]
```

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

### Trace

```
encode(["leet","code","#hi"])
  → "4#leet" + "4#code" + "3##hi"
  → "4#leet4#code3##hi"

decode walks left-to-right:
  read until '#' → "4", skip '#', take 4 chars "leet"
  read until '#' → "4", skip '#', take 4 chars "code"
  read until '#' → "3", skip '#', take 3 chars "#hi"
```

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

### Trace

```
nums=[100,4,200,1,3,2]  set={100,4,200,1,3,2}

x=100  100-1=99 not in set → start. Extend: 101? no. len=1
x=4    4-1=3 in set → skip (not a start)
x=200  199 not in set → start. len=1
x=1    0 not in set → start. Extend: 2,3,4. len=4 ✓
x=3,2  prev in set → skip
return 4
```

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

### Trace

```
nums=[1,1,1]  k=2
prefix=0, count={0:1}  total=0
i=0 prefix=1  need=1-2=-1 ∉  count={0:1,1:1}
i=1 prefix=2  need=2-2= 0 HIT ×1 → total=1   count={0:1,1:1,2:1}
i=2 prefix=3  need=3-2= 1 HIT ×1 → total=2   ...
return 2
```

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

### Trace

```
nums=[23,2,4,6,7]  k=6
remainder map = {0:-1}
i=0 prefix=23  rem=5  store {0:-1, 5:0}
i=1 prefix=25  rem=1  store {0:-1, 5:0, 1:1}
i=2 prefix=29  rem=5  HIT! prev_idx=0, i-prev=2 ≥ 2 → true
```

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

### Trace

```
nums   = [-2, 0, 3, -5, 2, -1]
prefix = [0, -2, -2, 1, -4, -2, -3]   // prefix[0]=0 sentinel

sumRange(0,2) = prefix[3] - prefix[0] = 1 - 0 = 1
sumRange(2,5) = prefix[6] - prefix[2] = -3 - (-2) = -1
```

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

### Trace

```
nums=[-2,1,-3,4,-1,2,1,-5,4]
i  x  cur=max(x,cur+x)  best
0 -2       -2            -2
1  1   max(1,-1)=1        1
2 -3   max(-3,-2)=-2      1
3  4   max(4,2)=4         4
4 -1   max(-1,3)=3        4
5  2   max(2,5)=5         5
6  1   max(1,6)=6         6   ← [4,-1,2,1]
7 -5   max(-5,1)=1        6
8  4   max(4,5)=5         6
return 6
```

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

### Trace

```
nums=[4,3,2,7,8,2,3,1]

x=4  nums[3]=7→-7
x=3  nums[2]=2→-2
x=2  nums[1]=3→-3
x=7  nums[6]=3→-3 ... already negative? wait, |7|-1=6, nums[6]=3>0 → -3
x=8  nums[7]=1→-1
x=|-2|=2  nums[1]=-3 < 0 → 2 is duplicate ✓
x=|-3|=3  nums[2]=-2 < 0 → 3 is duplicate ✓
x=|-1|=1  nums[0]=4>0 → -4

result=[2,3]
```

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

**LC #41** · Hard · **O(n) time, O(1) extra space**

Find smallest missing positive integer.

**Edge cases:** all negatives → 1 · `[1,2,3]` → 4 · contains huge numbers (ignore them).

### 🧠 Pattern: Cyclic Sort (Index Slot)

> Answer is in `[1, n+1]`. Place each value `v` into index `v-1` via swaps (only if `1 ≤ v ≤ n` and `nums[v-1] != v`). Then scan: first index where `nums[i] != i+1` ⇒ missing is `i+1`. If all match, answer is `n+1`.

### Approach Evolution

1. **Hash set** · O(n)/O(n). Easy but violates space.
2. **Sort** · O(n log n).
3. **Cyclic sort — FINAL** · O(n)/O(1). Each swap places one number into its home slot; O(n) total swaps.

### Trace

```
nums=[3,4,-1,1]   n=4
i=0 v=3, want at slot 2. swap with nums[2]:  [-1,4,3,1]
i=0 v=-1, out of range → i++
i=1 v=4, want at slot 3. swap with nums[3]:  [-1,1,3,4]
i=1 v=1, want at slot 0. swap with nums[0]:  [1,-1,3,4]
i=1 v=-1, out of range → i++
i=2 nums[2]=3 → at home → i++
i=3 nums[3]=4 → at home → i++

scan: nums[0]=1 ✓  nums[1]=-1 ≠ 2 → return 2
```

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

> [!warning] Loop without `i++`
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

### Trace

```
A=[1,2] B=[-2,-1] C=[-1,2] D=[0,2]

AB sums: 1+(-2)=-1, 1+(-1)=0, 2+(-2)=0, 2+(-1)=1
ab_count = {-1:1, 0:2, 1:1}

For each (c,d):
 -1+0=-1 → need 1 → ab_count[1]=1     total=1
 -1+2= 1 → need -1 → ab_count[-1]=1   total=2
  2+0= 2 → need -2 → 0
  2+2= 4 → need -4 → 0
return 2
```

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

### Trace

```
insert(1): arr=[1]       idx={1:0}
insert(2): arr=[1,2]     idx={1:0, 2:1}
insert(3): arr=[1,2,3]   idx={1:0, 2:1, 3:2}

remove(2):
  i = idx[2] = 1
  last = arr[-1] = 3
  arr[1] = 3                  → arr=[1,3,3]
  idx[3] = 1                  → idx={1:0, 2:1, 3:1}
  arr.pop()                   → arr=[1,3]
  delete idx[2]               → idx={1:0, 3:1}

getRandom(): random index in [0..len-1]
```

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

> [!tip] After finishing this drill
> You should be able to write Two Sum and Group Anagrams in under 3 minutes from scratch, and recognize "prefix sum + hash" the instant a problem says "subarray sum equals K".
