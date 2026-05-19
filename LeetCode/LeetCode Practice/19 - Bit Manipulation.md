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

8 problems · XOR tricks, masks, counting bits.

> [!abstract] Pattern recap
> Memorize these bit identities:
> - `x ^ x = 0` · `x ^ 0 = x` · XOR is commutative + associative → cancels duplicates
> - `x & (x - 1)` → clears the lowest set bit (used for counting bits, power-of-2 check)
> - `x & -x` → isolates the lowest set bit
> - `(x >> i) & 1` → i-th bit
> - `x | (1 << i)` → set i-th bit
> - `x & ~(1 << i)` → clear i-th bit
> - `x ^ (1 << i)` → toggle i-th bit

> [!tip] Why bit manipulation wins
> Compact representation (subsets in 32 bits), O(1) operations, can replace boolean arrays and small sets. Common in interviews even if "bit manipulation" isn't called out.

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

---

## P1: Single Number

**LC #136** · Easy

Every element appears twice except one. Find it. **O(n) time, O(1) space.**

### 🧠 Pattern: XOR Cancels Duplicates

> XOR all elements. Pairs cancel, leaving the singleton.

> [!success]- Python
> ```python
> def single_number(nums):
>     out = 0
>     for x in nums:
>         out ^= x
>     return out
> ```

**Key takeaway:** "Find singleton among pairs" → XOR. Memorize the identity `x ^ x = 0`.

---

## P2: Single Number II

**LC #137** · Medium

Every element appears 3× except one. Find it.

### 🧠 Pattern: Bit Counting Mod 3

> For each bit position, sum the bit across all numbers, take mod 3. The result is the singleton's bit at that position.

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

> [!tip] Bitwise state machine (advanced)
> `ones`, `twos` registers: `ones = (ones ^ x) & ~twos`, `twos = (twos ^ x) & ~ones`. O(1) space, no mod. Memorable only if you derive it.

**Key takeaway:** Generalize XOR trick: for "appears k times except one", sum each bit mod k.

---

## P3: Single Number III

**LC #260** · Medium

Two singletons, everything else appears twice. Return both.

### 🧠 Pattern: XOR + Partition by Distinguishing Bit

> XOR all → `a ^ b` (the two singletons combined). Pick any set bit (e.g., lowest with `x & -x`); it differs between a and b. Partition input by that bit, XOR each group → recover a and b.

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

### 🧠 Pattern: `x & (x - 1)` Clears Lowest Set Bit

> Each iteration removes one set bit. Count iterations until x = 0.

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

Return `[popcount(0), popcount(1), ..., popcount(n)]`.

### 🧠 Pattern: DP on Bits

> `bits[i] = bits[i >> 1] + (i & 1)`. (Equivalently `bits[i] = bits[i & (i-1)] + 1`.)

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

`nums` contains `n` distinct numbers from `0..n`. Find missing.

### Approach (XOR)

XOR all of `0..n` and all of `nums`. Duplicates cancel, missing remains.

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

Reverse the bits of a 32-bit unsigned integer.

> [!success]- Python
> ```python
> def reverse_bits(n):
>     result = 0
>     for i in range(32):
>         result = (result << 1) | (n & 1)
>         n >>= 1
>     return result
> ```

**Variants:** Reverse using lookup table for chunks of 8 bits.

**Key takeaway:** Bit-by-bit shift in, shift out. Classic O(32).

---

## P8: Sum of Two Integers

**LC #371** · Medium · **No `+` or `-`**

### 🧠 Pattern: XOR for Sum Without Carry; AND<<1 for Carry

> `sum_no_carry = a ^ b`. `carry = (a & b) << 1`. Loop until carry is 0. (In Python you must mask to 32 bits because Python ints are arbitrary precision.)

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

> [!warning] Python negative numbers
> Python has unbounded ints. Without masking, you'd loop forever on negative inputs. Mask + manually two's-complement the result.

**Key takeaway:** Adders use XOR (sum) + AND (carry). The mask trick is necessary for Python.

---

> [!tip] After this drill
> Memorize the 6 bit identities at the top. They turn many problems from O(n) auxiliary space into O(1). XOR is the workhorse for pair-cancellation.
