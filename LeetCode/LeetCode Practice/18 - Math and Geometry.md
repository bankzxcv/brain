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

9 problems · matrix manipulation, modular arithmetic, fast exponentiation, weighted random.

> [!abstract] Pattern recap
> Not a unified pattern. Recurring tricks:
> - **In-place 2D rotation** — transpose + reverse
> - **Spiral / boundary walk** — track boundaries, shrink each pass
> - **Bit tricks for "in-place flag"** — use first row/col as markers
> - **Fast exponentiation** — `pow(x, n) = pow(x, n/2)²` with parity adjustment

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
| P9 | Reverse Pairs | 493 | **Hard** | Merge sort with counting |

---

## P1: Rotate Image

**LC #48** · Medium · **In place**

> [!example]- 📊 Visual: transpose + reverse-each-row = 90° rotation
> ```text
>   Original          After transpose         After row-reverse
>   1 2 3             1 4 7                   7 4 1
>   4 5 6     ──→     2 5 8       ──→         8 5 2
>   7 8 9             3 6 9                   9 6 3
> 
>   Transpose: swap M[i][j] ↔ M[j][i] for i < j
>     (mirrors the matrix across the main diagonal ↘)
> 
>          ↘ diag
>      1 . .
>      ↕ 5 .
>      ↕ ↕ 9     swaps: (0,1)↔(1,0), (0,2)↔(2,0), (1,2)↔(2,1)
> 
>   Then reverse each row → final rotation.
> 
>   Why this works:
>     A 90° clockwise rotation maps (r, c) → (c, n-1-r).
>     Transpose maps (r, c) → (c, r).
>     Then reversing row r maps (c, r) → (c, n-1-r). ✓
> ```

> [!info]- 🔍 Dry Run: matrix=[[1,2,3],[4,5,6],[7,8,9]]
> ```text
> Step 1 — Transpose (swap M[i][j] ↔ M[j][i] for i<j):
> 
> Original:        After transpose:
>   1 2 3            1 4 7
>   4 5 6     →      2 5 8
>   7 8 9            3 6 9
> 
> Swaps performed:
>   (0,1) ↔ (1,0): 2↔4
>   (0,2) ↔ (2,0): 3↔7
>   (1,2) ↔ (2,1): 6↔8
> 
> Step 2 — Reverse each row:
>   row 0: [1,4,7] → [7,4,1]
>   row 1: [2,5,8] → [8,5,2]
>   row 2: [3,6,9] → [9,6,3]
> 
> Final:
>   7 4 1
>   8 5 2
>   9 6 3
> 
> ✅ Answer: matrix rotated 90° clockwise.
> ```

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

**Key takeaway:** Geometric transforms decompose. Memorize: 90° clockwise = transpose + reverse-each-row.

---

## P2: Spiral Matrix

**LC #54** · Medium

> [!example]- 📊 Visual: shrink boundaries on each lap
> ```text>
>   Starting matrix:
>      1 → 2 → 3
>                ↓
>      4 → 5    6
>      ↑        ↓
>      7 ← 8 ← 9
> 
>   Path: 1 2 3 6 9 8 7 4 5
> 
>   Boundary walk:
>     ┌─ top
>     │  ┌──────────┐
>     │  │ 1  2  3  │  ← walk left→right, then top++
>     │  │ 4  5  6  │
>     │  │ 7  8  9  │  ← walk right→left, then bottom--
>     │  └──────────┘
>     │  ↑          ↑
>     │  left     right
>     │              walk top→bottom on right col, then right--
>     │              walk bottom→top on left col, then left++
> 
>   After 1st lap: top=1, bottom=1, left=1, right=1   (only [5] left)
>   2nd lap visits just 5.
> 
>   The two inner `if`s (top ≤ bottom, left ≤ right) prevent revisits
>   when the matrix collapses to a single row or column mid-spiral.
> ```

> [!info]- 🔍 Dry Run: M=[[1,2,3],[4,5,6],[7,8,9]]
> ```text
> Boundaries: top=0, bottom=2, left=0, right=2
> out = []
> 
> ─────────────────────────────────────────
> While top ≤ bottom AND left ≤ right:
> 
> Iteration 1:
>   Walk top row left→right: out += [1,2,3]; top++ = 1
>   Walk right col top→bottom: out += [6,9]; right-- = 1
>   top ≤ bottom? 1 ≤ 2 ✓ → walk bottom row right→left: out += [8,7]; bottom-- = 1
>   left ≤ right? 0 ≤ 1 ✓ → walk left col bottom→top: out += [4]; left++ = 1
> 
> Iteration 2:
>   top=1, bottom=1, left=1, right=1
>   Walk top row: out += [5]; top++ = 2
>   Walk right col: range(top=2, bottom=1+1=2) empty → skip
>   top ≤ bottom? 2 ≤ 1 NO → skip bottom walk
>   loop check: top(2) > bottom(1) → exit
> 
> ✅ Answer: [1, 2, 3, 6, 9, 8, 7, 4, 5]
> ```

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

> [!example]- 📊 Visual: first row/col as flag bits
> ```text
>   Input:                Stash whether row0/col0 have any 0:
>     1 1 1                  first_row_has_0 = false
>     1 0 1                  first_col_has_0 = false
>     1 1 1
> 
>   Step A — scan inner cells (r≥1, c≥1).
>   For each 0 found at (r,c), MARK by setting M[r][0]=0 and M[0][c]=0:
> 
>             c=0 c=1 c=2
>     r=0:    1   ◦  1     ← col flags live in row 0
>     r=1:    ◦  [0] 1     ← row flags live in col 0
>     r=2:    1   1  1
> 
>     (◦ = marker bit overlaid on first row/col)
> 
>   Step B — for every inner cell (r≥1, c≥1):
>     if M[r][0]==0 or M[0][c]==0  → set M[r][c]=0
> 
>            1 0 1
>            0 0 0
>            1 0 1
> 
>   Step C — apply the two stashed flags to first row/col.
>     (Here both were false, so nothing extra to do.)
> 
>   We reused the existing matrix as O(1) extra storage — only TWO scalars
>   needed (the two stashed booleans).
> ```

> [!info]- 🔍 Dry Run: matrix=[[1,1,1],[1,0,1],[1,1,1]]
> ```text
> Check first row has any 0: row 0 = [1,1,1] → first_row_zero = False
> Check first col has any 0: col 0 = [1,1,1] → first_col_zero = False
> 
> Sweep (1..R-1, 1..C-1) marking first row/col as flags:
>   (1,1): val=0 → mark matrix[1][0]=0 AND matrix[0][1]=0
>   (others all 1)
> 
> After marking:
>   1 0 1
>   0 0 1
>   1 1 1
> 
> Sweep again (1..R-1, 1..C-1) and zero out cells whose row-marker OR col-marker is 0:
>   (1,1): matrix[1][0]=0 → zero out (already 0)
>   (1,2): matrix[1][0]=0 → zero out
>   (2,1): matrix[0][1]=0 → zero out
>   (2,2): both markers fine (matrix[2][0]=1, matrix[0][2]=1)
> 
> After:
>   1 0 1
>   0 0 0
>   1 0 1
> 
> Finally: first_row_zero was False → don't zero row 0; first_col_zero False → don't zero col 0
> 
> ✅ Final:
>   1 0 1
>   0 0 0
>   1 0 1
> ```

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

> [!example]- 📊 Visual: digit-square sequence → reaches 1 or cycles
> ```text
>   sum_of_digit_squares(x):  e.g.  19 → 1² + 9² = 1 + 81 = 82
> 
>   n=19 trajectory:
>     19 → 82 → 68 → 100 → 1 → 1 → 1 …       HAPPY ✓
> 
>     19  ──▶  82  ──▶  68  ──▶  100  ──▶  1  ↻
> 
>   n=2 trajectory:
>     2 → 4 → 16 → 37 → 58 → 89 → 145 → 42 → 20 → 4 → 16 → …
> 
>          ┌─────────────────────────────────────┐
>          ▼                                     │
>     2 → 4 ─→ 16 ─→ 37 ─→ 58 ─→ 89 ─→ 145 ─→ 42 ─→ 20
>          ↑___________________________________________│  (cycle, never reaches 1)
> 
>   Floyd's tortoise & hare on the integer state graph:
>     - If fast lands on 1     → happy (true)
>     - If slow == fast (≠ 1)  → cycle detected → not happy
> 
>   Why is the sequence bounded? For any x, the next value is at most
>   81·(#digits of x), which shrinks fast. State space is finite → must cycle.
> ```

> [!info]- 🔍 Dry Run: n=19
> ```text
> Helper: next_n(x) sums squares of digits.
>   next_n(19) = 1² + 9² = 1 + 81 = 82
>   next_n(82) = 64 + 4 = 68
>   next_n(68) = 36 + 64 = 100
>   next_n(100) = 1
>   next_n(1) = 1
> 
> Floyd cycle on number sequence:
>   slow=19, fast=next_n(19)=82
> 
> Iter 1:
>   slow = next_n(19) = 82
>   fast = next_n(next_n(82)) = next_n(68) = 100
>   fast != 1 AND slow != fast
> 
> Iter 2:
>   slow = next_n(82) = 68
>   fast = next_n(next_n(100)) = next_n(1) = 1
>   fast == 1 → exit loop
> 
> Return fast == 1 → True
> 
> ✅ Answer: true
> 
> ─────────────────────────────────────────
> Counter-example: n=2
>   2 → 4 → 16 → 37 → 58 → 89 → 145 → 42 → 20 → 4 (cycle!)
>   Floyd will detect: at some point slow == fast inside the cycle.
>   Return false.
> ```

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

> [!example]- 📊 Visual: binary exponentiation tree
> ```text
>   Compute  2^10.   Binary of 10 = 1010₂  →  10 = 8 + 2.
> 
>   So  2^10 = 2^8 · 2^2.
> 
>   Build successive squares of x  (each is x^(2^k)):
> 
>          x = 2
>          ↓ square
>          x² = 4              ← used (bit 1 of n is 1)
>          ↓ square
>          x⁴ = 16
>          ↓ square
>          x⁸ = 256            ← used (bit 3 of n is 1)
> 
>   Multiply the chosen ones:
> 
>     result = x² · x⁸  =  4 · 256  =  1024
> 
>   Trace of the while loop (n in binary, low bit first):
>     n=1010   bit0=0  skip mult,   x: 2→4,   n→101
>     n= 101   bit0=1  result*=x→4, x: 4→16,  n→ 10
>     n=  10   bit0=0  skip mult,   x:16→256, n→  1
>     n=   1   bit0=1  result*=x→1024, n→0    EXIT
> 
>   O(log n) multiplications instead of n. Negative n: flip x=1/x, n=-n.
> ```

> [!info]- 🔍 Dry Run: x=2.0, n=10
> ```text
> Iterative fast exponentiation:
>   result = 1
>   x = 2, n = 10
> 
> Iter 1: n=10 binary=1010
>   n & 1 = 0 → don't multiply result
>   x = x² = 4
>   n >>= 1 → n = 5
> 
> Iter 2: n=5 (=101 binary)
>   n & 1 = 1 → result *= x → result = 1 * 4 = 4
>   x = x² = 16
>   n >>= 1 → n = 2
> 
> Iter 3: n=2 (=10)
>   n & 1 = 0
>   x = 16² = 256
>   n = 1
> 
> Iter 4: n=1
>   n & 1 = 1 → result *= x → result = 4 * 256 = 1024
>   x = x² = ...
>   n = 0 → exit
> 
> ✅ Answer: 1024
>   Check: 2^10 = 1024 ✓
> 
> ─────────────────────────────────────────
> Negative n: pow(2.0, -2) = 1 / pow(2, 2) = 1/4 = 0.25
>   We flip: x = 1/x, n = -n at the start, then proceed as above.
> ```

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

> [!example]- 📊 Visual: schoolbook long-multiplication grid
> ```text
>   num1 = "123"   num2 = "45"     answer = "5535"
> 
>   Schoolbook layout:
> 
>           1   2   3
>         ×     4   5
>         ─────────────
>           1   2   3        ← (5 · "123") column-shifted to units place
>         · 5  10  15
>         (carries fold in: 5·3=15→5 carry1, 5·2+1=11→1 carry1, 5·1+1=6)
>         →     6   1   5
>           4   8  12         ← (4 · "123") shifted one place left
>         (4·3=12→2 c1, 4·2+1=9, 4·1=4)
>         →   4   9   2 _
>         ─────────────────
>           5   5   3   5
> 
>   In-array indexing trick: digit num1[i] · num2[j] contributes to result
>   positions (i+j, i+j+1).   p2 holds units; p1 holds carry.
> 
>     index in res :    0     1     2     3     4
>     for i=2,j=1 :                       p1=3  p2=4
>     for i=2,j=0 :                 p1=2  p2=3
>     for i=1,j=1 :                 p1=2  p2=3
>     for i=1,j=0 :           p1=1  p2=2
>     for i=0,j=1 :           p1=1  p2=2
>     for i=0,j=0 :     p1=0  p2=1
> 
>   After all contributions + carry propagation:
>     res = [0, 5, 5, 3, 5]  →  strip leading 0  →  "5535"
> ```

> [!info]- 🔍 Dry Run: num1="123", num2="45"
> ```text
> m=3, n=2, result array of size m+n=5: [0,0,0,0,0]
> 
> i=2 (digit '3'), j=1 ('5'):
>   mul = 3*5 = 15
>   p1 = i+j = 3, p2 = i+j+1 = 4
>   total = mul + result[p2=4] = 15 + 0 = 15
>   result[p2] = 15 % 10 = 5
>   result[p1] += 15 // 10 = 1
>   result = [0, 0, 0, 1, 5]
> 
> i=2 ('3'), j=0 ('4'):
>   mul = 3*4 = 12
>   p1=2, p2=3
>   total = 12 + result[3]=1 = 13
>   result[3] = 13 % 10 = 3
>   result[2] += 13 // 10 = 1
>   result = [0, 0, 1, 3, 5]
> 
> i=1 ('2'), j=1 ('5'):
>   mul = 10
>   p1=2, p2=3
>   total = 10 + result[3]=3 = 13
>   result[3] = 3
>   result[2] += 1 = 2
>   result = [0, 0, 2, 3, 5]
> 
> i=1 ('2'), j=0 ('4'):
>   mul = 8
>   p1=1, p2=2
>   total = 8 + result[2]=2 = 10
>   result[2] = 0
>   result[1] += 1
>   result = [0, 1, 0, 3, 5]
> 
> i=0 ('1'), j=1 ('5'):
>   mul = 5
>   p1=1, p2=2
>   total = 5 + 0 = 5
>   result[2] = 5
>   result[1] += 0
>   result = [0, 1, 5, 3, 5]
> 
> i=0 ('1'), j=0 ('4'):
>   mul = 4
>   p1=0, p2=1
>   total = 4 + 1 = 5
>   result[1] = 5
>   result[0] += 0
>   result = [0, 5, 5, 3, 5]
> 
> Strip leading zeros: "5535"
> 
> ✅ Answer: "5535"
>   Check: 123 * 45 = 5535 ✓
> ```

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

> [!example]- 📊 Visual: carry propagates right-to-left
> ```text
>   digits = [1, 2, 9]    add 1
> 
>     [ 1 ][ 2 ][ 9 ]
>                 ▲
>                 │  9 + 1 = 10  →  write 0, carry 1
>     [ 1 ][ 2 ][ 0 ]
>           ▲
>           │  2 + 1 = 3  →  write 3, done (no carry)
>     [ 1 ][ 3 ][ 0 ]      → 130 ✓
> 
>   ─────────────────────────────────────────
>   Worst case digits = [9, 9, 9]:
> 
>     [ 9 ][ 9 ][ 9 ]
>     [ 9 ][ 9 ][ 0 ]  carry
>     [ 9 ][ 0 ][ 0 ]  carry
>     [ 0 ][ 0 ][ 0 ]  carry off the front → PREPEND 1
>   [1][ 0 ][ 0 ][ 0 ]      → 1000 ✓
> 
>   In code: if any digit is < 9 we increment and return; else set to 0 and
>   continue. If we exit the loop we walked off the front → prepend 1.
> ```

> [!info]- 🔍 Dry Run: digits=[1,2,9]
> ```text
> i=2: digits[2]=9. Not < 9 → set to 0; continue loop with carry implicit
>   digits = [1, 2, 0]
> i=1: digits[1]=2. < 9 → digits[1] += 1 = 3; return digits = [1, 3, 0]
> 
> ✅ Answer: [1, 3, 0]
>   Check: 129 + 1 = 130 ✓
> 
> ─────────────────────────────────────────
> Edge: digits=[9,9,9]
>   i=2: 9 → set to 0; continue
>   i=1: 9 → set to 0; continue
>   i=0: 9 → set to 0; continue (loop ends)
>   digits = [0, 0, 0]
>   prepend 1 → [1, 0, 0, 0]
> ```

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

**LC #528** · Medium · Design

> [!example]- 📊 Visual: weights as variable-width bins on a number line
> ```text
>   weights w = [1, 3, 2, 4]      total = 10
>   prefix    = [1, 4, 6, 10]
> 
>   Place buckets back-to-back on the number line [0, 10):
> 
>     index:    0       1            2         3
>            ┌───┬─────────────┬─────────┬─────────────────┐
>            │ 1 │      3      │    2    │        4        │
>            └───┴─────────────┴─────────┴─────────────────┘
>     0      1                 4         6                 10
> 
>   To sample with the desired weights:
>     1. r = uniform[0, 10)
>     2. answer = bisect_left(prefix, r)
>        (first prefix value strictly greater than r → that bucket)
> 
>   Example draws:
>     r = 0.4  → falls in bin 0  → return 0   (prob 1/10)
>     r = 2.3  → falls in bin 1  → return 1   (prob 3/10)
>     r = 5.7  → falls in bin 2  → return 2   (prob 2/10)
>     r = 8.0  → falls in bin 3  → return 3   (prob 4/10)
> 
>   O(n) build; O(log n) per query (binary search into prefix array).
> ```

> [!info]- 🔍 Dry Run: w=[1, 3, 2, 4], pickIndex
> ```text
> Init: build prefix sums and total
>   prefix = [1, 4, 6, 10]
>   total = 10
> 
> The "weighted bucket" interpretation:
>   index 0 owns range [0, 1)
>   index 1 owns range [1, 4)   (length 3)
>   index 2 owns range [4, 6)   (length 2)
>   index 3 owns range [6, 10)  (length 4)
> 
> ─────────────────────────────────────────
> Call pickIndex():
>   r = random in [0, 10)  →  say r = 5.7
>   bisect_left(prefix, 5.7):
>     prefix=[1, 4, 6, 10]
>     5.7 vs 1 → go right
>     5.7 vs 4 → go right
>     5.7 vs 6 → go left
>     5.7 vs 4 → already past
>     return index 2
>   return 2
> 
> Call again, r=2.3:
>   bisect_left(prefix, 2.3) → between 1 and 4 → index 1
>   return 1
> 
> Over many calls, frequency ratio ≈ weights ratio 1:3:2:4
> ```

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

## P9: Reverse Pairs

**LC #493** · **Hard**

Count pairs `(i, j)` with `i < j` and `nums[i] > 2 * nums[j]`.

### 🧠 Pattern: Modified Merge Sort

> Pure brute force = O(n²). Merge sort gives O(n log n) by counting "reverse pairs" during the **merge** step: when we have two sorted halves, for each i in left, count how many j in right satisfy `left[i] > 2*right[j]`. Two pointers — both monotonic.

> [!example]- 📊 Visual: count cross-pairs during merge sort
> ```text
>   nums = [1, 3, 2, 3, 1]    count pairs (i<j) with nums[i] > 2·nums[j]
> 
>   Recursion tree (split, then merge):
> 
>                  [1 3 2 3 1]
>                 ╱           ╲
>             [1 3 2]         [3 1]
>             ╱     ╲          ╱  ╲
>          [1 3]   [2]       [3]  [1]
>          ╱ ╲
>        [1] [3]
> 
>   Bottom-up — at each MERGE, count pairs using two sorted halves:
> 
>     merge [1] | [3]:
>          left=[1]  right=[3]
>          1 > 2·3=6 ?  no.   +0
>          sorted: [1,3]
> 
>     merge [1,3] | [2]:
>          1 > 2·2=4 ?  no
>          3 > 4    ?  no   +0
>          sorted: [1,2,3]
> 
>     merge [3] | [1]:
>          3 > 2·1=2 ? YES   +1
>          sorted: [1,3]
> 
>     merge [1,2,3] | [1,3]:
>          For each i in left, advance j in right while left[i] > 2·right[j]:
>            i=1: not > 2·1=2
>            i=2: not > 2
>            i=3: > 2  → j=1 (right[1]=3, 3 > 6? no, stop)   +1
>          +1 total at this level
> 
>   Grand total = 0 + 0 + 1 + 1 = 2   ✓
> 
>   Key insight: when both halves are SORTED, the j-pointer only moves forward
>   across all i — amortized O(n) at each merge → O(n log n) overall.
> ```

> [!info]- 🔍 Dry Run: nums=[1,3,2,3,1]
> ```text
> Brute force expected pairs:
>   (i=1, j=4): 3 > 2*1=2 ✓
>   (i=3, j=4): 3 > 2*1=2 ✓
>   That's 2 reverse pairs.
> 
> ─────────────────────────────────────────
> Merge sort approach:
> 
> Split: [1,3,2,3,1] → [1,3,2] and [3,1]
> Recurse left: [1,3,2]
>   Split: [1,3] and [2]
>   Recurse [1,3]: split [1], [3]. Both base. Count pairs: left=[1], right=[3]. 1 > 2*3? NO. count=0. Merge → [1,3].
>   Recurse [2]: base.
>   Count pairs across [1,3] and [2]:
>     i=0 (1): how many j with 1 > 2*right[j]? right=[2]: 1 > 4? NO. count_local=0.
>     i=1 (3): 3 > 2*2=4? NO. count_local=0.
>   Total subarray pairs = 0. Merge → [1, 2, 3].
> 
> Recurse right: [3,1]
>   Split [3], [1]. Both base.
>   Count pairs: i=0 (3): 3 > 2*1=2? YES! count=1.
>   Merge → [1, 3].
> 
> Count pairs across [1,2,3] and [1,3]:
>   i=0 (1): 1 > 2*1=2? NO. j stays at 0.
>   i=1 (2): 2 > 2*1=2? NO. j stays.
>   i=2 (3): 3 > 2*1=2? YES → j++ to 1. 3 > 2*3=6? NO. j stays.
>     # for this i, j ended at 1, so count += 1.
>   Total subarray pairs = 1.
> 
> Merge two sorted halves → [1, 1, 2, 3, 3]
> 
> Total reverse pairs = 0 (left subtree) + 1 (right subtree) + 1 (cross) = 2
> 
> ✅ Answer: 2
> ```

> [!success]- Python
> ```python
> def reverse_pairs(nums):
>     def merge_sort(lo, hi):
>         if lo >= hi: return 0
>         mid = (lo + hi) // 2
>         count = merge_sort(lo, mid) + merge_sort(mid + 1, hi)
>         # count cross pairs (i in left, j in right)
>         j = mid + 1
>         for i in range(lo, mid + 1):
>             while j <= hi and nums[i] > 2 * nums[j]:
>                 j += 1
>             count += j - (mid + 1)
>         # merge two sorted halves
>         nums[lo:hi+1] = sorted(nums[lo:hi+1])
>         return count
>     return merge_sort(0, len(nums) - 1)
> ```

> [!tip] Sort the subarray with built-in
> The merge step is the canonical two-pointer merge, but for clarity Python's `sorted(slice)` does it concisely. For optimal performance, write the merge manually.

**Variants:** Count of Smaller Numbers After Self · Reverse Pairs with different multipliers.

**Key takeaway:** Counting pairs with a comparative condition → modified merge sort. O(n log n) using sorted-half monotonicity.

---

> [!tip] After this drill
> Memorize: rotate = transpose + reverse · pow = halve exponent · weighted random = prefix + binary search · counting reverse pairs = merge sort.
