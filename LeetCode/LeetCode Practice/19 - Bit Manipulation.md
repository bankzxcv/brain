---
title: "LeetCode Practice: Bit Manipulation"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - bit-manipulation
  - xor
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Bit Manipulation

9 problems · XOR tricks, masks, counting bits.

> [!abstract] Pattern recap
> Memorize these bit identities:
> - `x ^ x = 0` · `x ^ 0 = x` · XOR cancels duplicates
> - `x & (x - 1)` → clears the lowest set bit
> - `x & -x` → isolates the lowest set bit
> - `(x >> i) & 1` → i-th bit
> - `x | (1 << i)` → set i-th bit
> - `x & ~(1 << i)` → clear i-th bit
> - `x ^ (1 << i)` → toggle i-th bit

> [!tip] Why bit manipulation wins
> Compact representation (subsets in 32 bits), O(1) operations, can replace boolean arrays and small sets.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Single Number | 136 | Easy | XOR all |
| P2 | Single Number II | 137 | Med | Bit counting mod 3 |
| P3 | Single Number III | 260 | Med | XOR + partition by bit |
| P4 | Number of 1 Bits | 191 | Easy | `x & (x-1)` loop |
| P5 | Counting Bits | 338 | Easy | DP on bits |
| P6 | Missing Number | 268 | Easy | XOR or sum |
| P7 | Reverse Bits | 190 | Easy | Bit by bit |
| P8 | Sum of Two Integers | 371 | Med | XOR for sum, AND for carry |
| P9 | Maximum XOR With an Element From Array | 1707 | **Hard** | Offline queries + bit trie |

---

## P1: Single Number

**LC #136** · Easy

> [!info]- 🔍 Dry Run: nums=[4,1,2,1,2]
> ```text
> out = 0
> 
> x=4: out = 0 ^ 4 = 4
> x=1: out = 4 ^ 1 = 5
> x=2: out = 5 ^ 2 = 7
> x=1: out = 7 ^ 1 = 6
> x=2: out = 6 ^ 2 = 4
> 
> ✅ Answer: 4
>   The duplicates 1^1=0 and 2^2=0 cancel; only 4 survives.
> ```

> [!success]- Python
> ```python
> def single_number(nums):
>     out = 0
>     for x in nums:
>         out ^= x
>     return out
> ```

**Key takeaway:** "Find singleton among pairs" → XOR. Memorize `x ^ x = 0`.

---

## P2: Single Number II

**LC #137** · Medium

> [!info]- 🔍 Dry Run: nums=[2,2,3,2]
> ```text
> For each bit position i in [0, 31]:
>   sum the i-th bit across all numbers
>   bit of result = sum mod 3
> 
> ─────────────────────────────────────────
> nums in binary (4 bits for illustration):
>   2 = 0010
>   2 = 0010
>   3 = 0011
>   2 = 0010
> 
> Bit position 0 (rightmost):
>   sum = 0+0+1+0 = 1
>   1 % 3 = 1 → result bit 0 = 1
> 
> Bit position 1:
>   sum = 1+1+1+1 = 4
>   4 % 3 = 1 → result bit 1 = 1
> 
> Bit positions 2..31: sum=0, 0%3=0
> 
> result = 0...0 11 = 3
> 
> ✅ Answer: 3
>   The triplicate 2 contributes 3 to each set bit; mod 3 wipes it out.
> ```

> [!success]- Python
> ```python
> def single_number_ii(nums):
>     result = 0
>     for i in range(32):
>         s = sum((x >> i) & 1 for x in nums) % 3
>         if i == 31 and s:   # sign bit, handle negatives
>             result -= (1 << 31)
>         else:
>             result |= s << i
>     return result
> ```

**Key takeaway:** Generalize XOR trick: for "appears k times except one", sum each bit mod k.

---

## P3: Single Number III

**LC #260** · Medium

> [!info]- 🔍 Dry Run: nums=[1,2,1,3,2,5]
> ```text
> XOR all: 1^2^1^3^2^5 = (1^1)^(2^2)^3^5 = 0^0^3^5 = 3^5 = 6 (binary 110)
> So a ^ b = 6 (the two singletons combined).
> 
> Find a distinguishing bit (lowest set bit of 6):
>   6 & -6 = 6 & (~6+1) = 6 & ...11111010 = 0010 (binary) = 2
>   So bit at position 1 distinguishes a and b.
> 
> Partition nums by that bit:
>   Group A (bit set): {2, 2}   (2 has bit 1 set)
>   Group B (bit clear): {1, 1, 3, 5}
> 
> Wait, let me re-check: 1=001, 2=010, 3=011, 5=101.
>   Bit 1 set: 2 (010), 3 (011) → group A = {2, 2, 3}
>   Bit 1 clear: 1, 1, 5 → group B = {1, 1, 5}
> 
> XOR each group:
>   A: 2^2^3 = 3
>   B: 1^1^5 = 5
> 
> ✅ Answer: [3, 5]
> ```

> [!success]- Python
> ```python
> def single_number_iii(nums):
>     xor_all = 0
>     for x in nums: xor_all ^= x
>     diff_bit = xor_all & -xor_all
>     a = b = 0
>     for x in nums:
>         if x & diff_bit:
>             a ^= x
>         else:
>             b ^= x
>     return [a, b]
> ```

**Key takeaway:** Beautiful application of XOR: partition by a bit on which a and b differ.

---

## P4: Number of 1 Bits

**LC #191** · Easy

> [!info]- 🔍 Dry Run: n=11 (binary 1011)
> ```text
> count = 0
> 
> Iter 1: n=1011
>   n & (n-1) = 1011 & 1010 = 1010
>   n = 1010
>   count = 1
> 
> Iter 2: n=1010
>   n & (n-1) = 1010 & 1001 = 1000
>   n = 1000
>   count = 2
> 
> Iter 3: n=1000
>   n & (n-1) = 1000 & 0111 = 0000
>   n = 0
>   count = 3
> 
> Loop exits.
> 
> ✅ Answer: 3
>   Check: 11 = 8+2+1, three 1-bits. ✓
> ```

> [!success]- Python
> ```python
> def hamming_weight(n):
>     count = 0
>     while n:
>         n &= n - 1
>         count += 1
>     return count
> ```

**Key takeaway:** `x & (x-1)` = "drop the rightmost 1 bit". O(set-bit count), not O(32).

---

## P5: Counting Bits

**LC #338** · Easy

> [!info]- 🔍 Dry Run: n=5
> ```text
> bits[0] = 0
> 
> i=1: bits[1] = bits[1>>1=0] + (1 & 1) = 0 + 1 = 1
> i=2: bits[2] = bits[2>>1=1] + (2 & 1) = 1 + 0 = 1
> i=3: bits[3] = bits[3>>1=1] + (3 & 1) = 1 + 1 = 2
> i=4: bits[4] = bits[4>>1=2] + (4 & 1) = 1 + 0 = 1
> i=5: bits[5] = bits[5>>1=2] + (5 & 1) = 1 + 1 = 2
> 
> Final bits = [0, 1, 1, 2, 1, 2]
> 
> ✅ Answer: [0, 1, 1, 2, 1, 2]
> ```

> [!success]- Python
> ```python
> def count_bits(n):
>     bits = [0] * (n + 1)
>     for i in range(1, n + 1):
>         bits[i] = bits[i >> 1] + (i & 1)
>     return bits
> ```

**Key takeaway:** popcount recursion via halving. O(n) overall.

---

## P6: Missing Number

**LC #268** · Easy

> [!info]- 🔍 Dry Run: nums=[3,0,1] (n=3, should contain 0..3 but missing 2)
> ```text
> result = len(nums) = 3   (initialize with n)
> 
> i=0 x=3: result ^= i (0) ^ x (3) → result = 3 ^ 0 ^ 3 = 0
> i=1 x=0: result ^= 1 ^ 0 → result = 0 ^ 1 = 1
> i=2 x=1: result ^= 2 ^ 1 → result = 1 ^ 2 ^ 1 = 2
> 
> ✅ Answer: 2
>   Logic: XOR all of 0..n and all elements; duplicates cancel; missing remains.
>   We started with result=n=3 (representing the index n that has no corresponding nums entry),
>   then for each i in 0..n-1, XOR'd both i and nums[i].
>   Every number that appears in both 0..n and nums cancels; the missing one is what's left.
> ```

> [!success]- Python
> ```python
> def missing_number(nums):
>     result = len(nums)
>     for i, x in enumerate(nums):
>         result ^= i ^ x
>     return result
> ```

**Variants:** Sum formula `n*(n+1)/2 - sum(nums)` (risk: overflow in some languages).

**Key takeaway:** XOR pairing trick generalizes to "0..n missing one".

---

## P7: Reverse Bits

**LC #190** · Easy

> [!info]- 🔍 Dry Run: n=43261596 (binary 00000010100101000001111010011100)
> ```text
> result = 0
> 
> Iter 1: shift result left, append bit 0 of n
>   bit 0 of n = 0
>   result = (0 << 1) | 0 = 0
>   n >>= 1
> 
> Iter 2: bit 0 of new n = 0
>   result = (0 << 1) | 0 = 0
>   n >>= 1
> 
> Iter 3: bit 0 = 1
>   result = (0 << 1) | 1 = 1
>   n >>= 1
> 
> ... continue 32 times.
> 
> After 32 iterations, result holds the bit-reversed value.
> 
> ✅ Answer: 964176192 (binary 00111001011110000010100101000000)
> ```

> [!success]- Python
> ```python
> def reverse_bits(n):
>     result = 0
>     for i in range(32):
>         result = (result << 1) | (n & 1)
>         n >>= 1
>     return result
> ```

**Key takeaway:** Bit-by-bit shift in, shift out. Classic O(32).

---

## P8: Sum of Two Integers

**LC #371** · Medium

> [!info]- 🔍 Dry Run: a=2, b=3
> ```text
> Binary: a=010, b=011
> 
> Iter 1: b≠0
>   carry = (a & b) << 1 = (010 & 011) << 1 = 010 << 1 = 100
>   a = a ^ b = 010 ^ 011 = 001
>   b = carry = 100
>   State: a=001, b=100
> 
> Iter 2: b≠0
>   carry = (001 & 100) << 1 = 0
>   a = 001 ^ 100 = 101
>   b = 0
>   State: a=101, b=0
> 
> Loop exits.
> 
> ✅ Answer: 101 = 5
>   Check: 2 + 3 = 5 ✓
> 
> ─────────────────────────────────────────
> Python negatives: requires masking to 32 bits because Python ints are unbounded.
> ```

> [!success]- Python
> ```python
> def get_sum(a, b):
>     MASK = 0xFFFFFFFF
>     MAX_INT = 0x7FFFFFFF
>     while b:
>         carry = ((a & b) << 1) & MASK
>         a = (a ^ b) & MASK
>         b = carry
>     return a if a <= MAX_INT else ~(a ^ MASK)
> ```

**Key takeaway:** Adders use XOR (sum) + AND (carry).

---

## P9: Maximum XOR With an Element From Array

**LC #1707** · **Hard**

Given `nums` and queries `[xi, mi]`, return max `xi XOR nums[j]` for each query where `nums[j] ≤ mi` (or -1 if no such j).

### 🧠 Pattern: Offline Queries + Sorted Insertion into Bit Trie

> Process queries in increasing order of `mi`. Sort `nums` ascending. For each query, insert into a binary trie all nums ≤ mi (incrementally). Then perform max-XOR query against the trie (P6 in Tries).

> [!info]- 🔍 Dry Run: nums=[0,1,2,3,4], queries=[[3,1],[1,3],[5,6]]
> ```text
> Sort nums: [0, 1, 2, 3, 4]
> 
> Sort queries with original indices:
>   q0 = (3, 1, orig_idx=0)
>   q1 = (1, 3, orig_idx=1)
>   q2 = (5, 6, orig_idx=2)
> 
> Sort by mi:
>   (3, 1, 0), (1, 3, 1), (5, 6, 2)
> 
> trie = {}
> answers = [None, None, None]
> nums_idx = 0
> 
> ─────────────────────────────────────────
> Query (xi=3, mi=1, orig=0):
>   Insert nums[nums_idx]=0 while nums[nums_idx] ≤ mi=1:
>     Insert 0 (binary 000...) into trie. nums_idx=1
>     Insert 1 (binary 001) into trie. nums_idx=2
>     nums[2]=2 > 1 → stop.
>   Query max XOR(3, anything in trie):
>     3 (binary ...011) want to XOR with bits 100 (opposite).
>     Best available: 0 → 3 XOR 0 = 3
>     Or 1 → 3 XOR 1 = 2
>     Max = 3
>   answers[0] = 3
> 
> Query (xi=1, mi=3, orig=1):
>   Insert more:
>     nums[2]=2 ≤ 3 → insert. nums_idx=3
>     nums[3]=3 ≤ 3 → insert. nums_idx=4
>     nums[4]=4 > 3 → stop
>   trie now has {0,1,2,3}
>   Query XOR(1):
>     1=001; want bits 110 (opposite). Best is 2 (010) — XOR with 1 = 011 = 3. Or 3=011 XOR 1 = 010=2.
>     Actually: 1 XOR 0=1, 1 XOR 1=0, 1 XOR 2=3, 1 XOR 3=2. Max=3.
>   answers[1] = 3
> 
> Query (xi=5, mi=6, orig=2):
>   Insert nums[4]=4 ≤ 6 → insert
>   trie has {0,1,2,3,4}
>   Query XOR(5):
>     5=101; XOR with 0=5, 1=4, 2=7, 3=6, 4=1. Max=7.
>   answers[2] = 7
> 
> Return answers in original order: [3, 3, 7]
> 
> ✅ Answer: [3, 3, 7]
> ```

> [!success]- Python
> ```python
> def maximize_xor(nums, queries):
>     nums.sort()
>     # attach original index
>     q = sorted([(m, x, i) for i, (x, m) in enumerate(queries)])
>     # bit trie
>     BITS = 31
>     root = {}
>     def insert(n):
>         node = root
>         for i in range(BITS, -1, -1):
>             b = (n >> i) & 1
>             node = node.setdefault(b, {})
>     def max_xor(x):
>         if not root: return -1
>         node = root
>         out = 0
>         for i in range(BITS, -1, -1):
>             b = (x >> i) & 1
>             toggle = 1 - b
>             if toggle in node:
>                 out |= (1 << i)
>                 node = node[toggle]
>             elif b in node:
>                 node = node[b]
>             else:
>                 return -1
>         return out
>     answers = [0] * len(queries)
>     ni = 0
>     for m, x, orig in q:
>         while ni < len(nums) and nums[ni] <= m:
>             insert(nums[ni])
>             ni += 1
>         answers[orig] = max_xor(x)
>     return answers
> ```

> [!tip] Offline processing
> When queries can be answered in any order, sorting them often unlocks a much faster algorithm. Always ask: "Can I reorder the queries?"

**Variants:** Maximum XOR of Two Numbers (LC 421, simpler — no upper bound on second operand).

**Key takeaway:** Offline queries sorted + incremental data structure (here: bit trie) = O((n+q) log n · BITS).

---

> [!tip] After this drill
> Memorize the 6 bit identities at the top. They turn many problems from O(n) auxiliary space into O(1). XOR is the workhorse for pair-cancellation.
