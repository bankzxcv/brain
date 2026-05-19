---
title: "LeetCode Practice: Math and Geometry"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - math
  - geometry
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Math and Geometry

8 problems · matrix manipulation, modular arithmetic, fast exponentiation, weighted random.

> [!abstract] Pattern recap
> Not a unified pattern. Recurring tricks:
> - **In-place 2D rotation** — transpose + reverse
> - **Spiral / boundary walk** — track boundaries, shrink each pass
> - **Bit tricks for "in-place flag"** — use first row/col as markers
> - **Fast exponentiation** — `pow(x, n) = pow(x, n/2)²` with parity adjustment
> - **Prefix sum + binary search** — for weighted random

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Rotate Image | 48 | Med | Transpose + reverse |
| P2 | Spiral Matrix | 54 | Med | Boundary walk |
| P3 | Set Matrix Zeroes | 73 | Med | First row/col as flags |
| P4 | Happy Number | 202 | Easy | Floyd on numbers |
| P5 | Pow(x, n) | 50 | Med | Fast exponentiation |
| P6 | Multiply Strings | 43 | Med | Manual long multiply |
| P7 | Plus One | 66 | Easy | Carry propagation |
| P8 | Random Pick with Weight | 528 | Med | Prefix sum + BS |

---

## P1: Rotate Image

**LC #48** · Medium · **In place**

Rotate n×n matrix 90° clockwise.

### 🧠 Pattern: Transpose + Reverse Each Row

> Clockwise rotation = transpose (swap `M[i][j]` ↔ `M[j][i]` for `i < j`), then reverse each row.

### Trace

```
[1,2,3]      [1,4,7]      [7,4,1]
[4,5,6]  →   [2,5,8]   →  [8,5,2]
[7,8,9]      [3,6,9]      [9,6,3]
   original   transposed    rows reversed
```

> [!success]- Python
> ```python
> def rotate(M):
>     n = len(M)
>     for i in range(n):
>         for j in range(i + 1, n):
>             M[i][j], M[j][i] = M[j][i], M[i][j]
>     for row in M:
>         row.reverse()
> ```

**Variants:** Counter-clockwise = reverse rows first, then transpose.

**Key takeaway:** Geometric transforms decompose into simpler steps. Memorize the 90° = transpose + reverse identity.

---

## P2: Spiral Matrix

**LC #54** · Medium

Return matrix elements in spiral order.

### 🧠 Pattern: Four Boundaries, Shrink Each Lap

> Maintain `top, bottom, left, right`. Walk one direction; shrink that boundary; rotate direction.

> [!success]- Python
> ```python
> def spiral_order(M):
>     out = []
>     top, bottom = 0, len(M) - 1
>     left, right = 0, len(M[0]) - 1
>     while top <= bottom and left <= right:
>         for c in range(left, right + 1): out.append(M[top][c])
>         top += 1
>         for r in range(top, bottom + 1): out.append(M[r][right])
>         right -= 1
>         if top <= bottom:
>             for c in range(right, left - 1, -1): out.append(M[bottom][c])
>             bottom -= 1
>         if left <= right:
>             for r in range(bottom, top - 1, -1): out.append(M[r][left])
>             left += 1
>     return out
> ```

**Key takeaway:** Boundary walk on grid. The inner two `if`s prevent re-visiting on thin matrices.

---

## P3: Set Matrix Zeroes

**LC #73** · Medium · **In place, O(1) extra**

If a cell is 0, set entire row and column to 0.

### 🧠 Pattern: First Row/Col as Markers + Two Flags

> Use the matrix's own first row and column to mark which rows/cols should be zeroed. Two extra booleans for the first row/col themselves.

> [!success]- Python
> ```python
> def set_zeroes(M):
>     R, C = len(M), len(M[0])
>     first_row = any(M[0][c] == 0 for c in range(C))
>     first_col = any(M[r][0] == 0 for r in range(R))
>     for r in range(1, R):
>         for c in range(1, C):
>             if M[r][c] == 0:
>                 M[r][0] = 0
>                 M[0][c] = 0
>     for r in range(1, R):
>         for c in range(1, C):
>             if M[r][0] == 0 or M[0][c] == 0:
>                 M[r][c] = 0
>     if first_row:
>         for c in range(C): M[0][c] = 0
>     if first_col:
>         for r in range(R): M[r][0] = 0
> ```

**Key takeaway:** O(1) auxiliary space in 2D problems often involves overloading existing rows/columns.

---

## P4: Happy Number

**LC #202** · Easy

Replace n by sum of squares of digits, repeat. 1 ⇒ happy.

### 🧠 Pattern: Floyd Cycle on Number Sequence

> Either reaches 1 (happy) or enters a cycle. Detect cycle with fast/slow.

> [!success]- Python
> ```python
> def is_happy(n):
>     def next_n(x):
>         s = 0
>         while x:
>             d = x % 10
>             s += d * d
>             x //= 10
>         return s
>     slow, fast = n, next_n(n)
>     while fast != 1 and slow != fast:
>         slow = next_n(slow)
>         fast = next_n(next_n(fast))
>     return fast == 1
> ```

**Key takeaway:** Cycle detection isn't only for linked lists — any state-machine iteration with bounded states.

---

## P5: Pow(x, n)

**LC #50** · Medium

Compute `x^n` in O(log n).

### 🧠 Pattern: Fast Exponentiation (Binary)

> `x^n = (x²)^(n/2)` if n even; `x · x^(n-1)` if n odd. Recursion depth = log n.

> [!success]- Python
> ```python
> def my_pow(x, n):
>     if n < 0:
>         x = 1 / x
>         n = -n
>     result = 1
>     while n:
>         if n & 1:
>             result *= x
>         x *= x
>         n >>= 1
>     return result
> ```

**Key takeaway:** Halve the exponent each step → O(log n). Iterative form avoids stack overflow.

---

## P6: Multiply Strings

**LC #43** · Medium

Multiply two non-negative integers represented as strings (no built-in big int).

### Approach

Long multiplication. Result has at most `m + n` digits. `result[i + j + 1] += (digits[i] * digits[j])`; propagate carry.

> [!success]- Python
> ```python
> def multiply(num1, num2):
>     if num1 == "0" or num2 == "0": return "0"
>     m, n = len(num1), len(num2)
>     res = [0] * (m + n)
>     for i in range(m - 1, -1, -1):
>         for j in range(n - 1, -1, -1):
>             mul = (ord(num1[i]) - 48) * (ord(num2[j]) - 48)
>             p1, p2 = i + j, i + j + 1
>             total = mul + res[p2]
>             res[p2] = total % 10
>             res[p1] += total // 10
>     s = ''.join(map(str, res)).lstrip('0')
>     return s or "0"
> ```

**Key takeaway:** Schoolbook algorithm by index. `i+j` and `i+j+1` are carry / units positions.

---

## P7: Plus One

**LC #66** · Easy

Add one to a number represented as digit array.

> [!success]- Python
> ```python
> def plus_one(digits):
>     for i in range(len(digits) - 1, -1, -1):
>         if digits[i] < 9:
>             digits[i] += 1
>             return digits
>         digits[i] = 0
>     return [1] + digits
> ```

**Key takeaway:** Carry propagation in reverse. If you walk off the front, prepend 1.

---

## P8: Random Pick with Weight

**LC #528** · Medium · **Design**

Pick index `i` with probability `weight[i] / sum(weight)`.

### 🧠 Pattern: Prefix Sum + Binary Search

> Build prefix sum (which forms cumulative probability buckets). Pick random in `[0, total)`; binary-search to find the bucket.

> [!success]- Python
> ```python
> import bisect, random
> class Solution:
>     def __init__(self, w):
>         self.prefix = []
>         s = 0
>         for x in w:
>             s += x
>             self.prefix.append(s)
>         self.total = s
>     def pickIndex(self):
>         r = random.random() * self.total
>         return bisect.bisect_left(self.prefix, r)
> ```

**Variants:** Reservoir Sampling (random k from stream).

**Key takeaway:** "Weighted sampling" → cumulative bucket via prefix sum, search into buckets.

---

> [!tip] After this drill
> Memorize: rotate = transpose + reverse · pow = halve exponent · weighted random = prefix + binary search · in-place flags = use first row/col.
