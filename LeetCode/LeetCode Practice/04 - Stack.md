---
title: "LeetCode Practice: Stack"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - stack
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Stack

10 problems · LIFO mechanics — matching, evaluation, design.

> [!abstract] Pattern recap
> Stack shines when you must **process now but resolve later** (matching `(`, evaluating operators with precedence, escaping nested structures). Push when you can't decide; pop when context tells you what to do.

> [!info] When you need a different stack
> - Need O(1) **min/max** of stack contents → auxiliary stack (P2)
> - Need "next greater/smaller" pattern → see [[05 - Monotonic Stack]]
> - Need **two stacks combining** for queue / calculator → P5, P6, P10

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Valid Parentheses | 20 | Easy | Pair matching |
| P2 | Min Stack | 155 | Med | Aux stack for min |
| P3 | Evaluate Reverse Polish Notation | 150 | Med | Operand stack |
| P4 | Generate Parentheses | 22 | Med | Implicit stack (recursion) |
| P5 | Implement Queue using Stacks | 232 | Easy | Two stacks |
| P6 | Implement Stack using Queues | 225 | Easy | Two queues |
| P7 | Asteroid Collision | 735 | Med | Stack with collision logic |
| P8 | Decode String | 394 | Med | Two stacks (count + string) |
| P9 | Simplify Path | 71 | Med | Stack of path parts |
| P10 | Basic Calculator II | 227 | Med | Operand stack + precedence |

---

## P1: Valid Parentheses

**LC #20** · Easy

`(){}[]` — return true if all brackets are matched and nested correctly.

**Edge cases:** empty (true) · odd length (false) · single bracket (false) · `]` first (false).

### 🧠 Pattern: Push Open, Pop on Close

> Push every open bracket. On a close, the top **must** be its matching open. Empty stack at end = valid.

### Trace

```
s="([{}])"
'('  push     stack=['(']
'['  push     stack=['(','[']
'{'  push     stack=['(','[','{']
'}'  pop '{', match ✓  stack=['(','[']
']'  pop '[', match ✓  stack=['(']
')'  pop '(', match ✓  stack=[]
return true
```

> [!success]- JS
> ```js
> const isValid = (s) => {
>   const pair = { ')': '(', ']': '[', '}': '{' };
>   const st = [];
>   for (const c of s) {
>     if (c in pair) {
>       if (st.pop() !== pair[c]) return false;
>     } else st.push(c);
>   }
>   return st.length === 0;
> };
> ```

> [!success]- Python
> ```python
> def is_valid(s):
>     pair = {')': '(', ']': '[', '}': '{'}
>     st = []
>     for c in s:
>         if c in pair:
>             if not st or st.pop() != pair[c]:
>                 return False
>         else:
>             st.append(c)
>     return not st
> ```

**Variants:** Minimum Add to Make Parentheses Valid · Score of Parentheses · Longest Valid Parentheses (stack of indices).

**Key takeaway:** Bracket matching = stack 101. Map close → expected open.

---

## P2: Min Stack

**LC #155** · Medium · **Design**

Stack with O(1) `push`, `pop`, `top`, `getMin`.

### 🧠 Pattern: Auxiliary Stack of Mins

> Mirror stack: each push, also push `min(x, currentMin)`. Pop both together. `getMin = topOfMins`.

### Approach Evolution

1. **Scan for min each `getMin`** · O(n) per call. Violates requirement.
2. **Pair (value, min-so-far) on one stack — FINAL** · O(1)/O(n).

### Trace

```
push(-2)  stack=[(-2,-2)]                    min=-2
push(0)   stack=[(-2,-2),(0,-2)]             min=-2
push(-3)  stack=[(-2,-2),(0,-2),(-3,-3)]     min=-3
getMin() → -3
pop()     stack=[(-2,-2),(0,-2)]             min=-2
top()  → 0
```

> [!success]- JS
> ```js
> class MinStack {
>   constructor() { this.s = []; }
>   push(x) {
>     const m = this.s.length ? Math.min(x, this.s.at(-1)[1]) : x;
>     this.s.push([x, m]);
>   }
>   pop() { this.s.pop(); }
>   top() { return this.s.at(-1)[0]; }
>   getMin() { return this.s.at(-1)[1]; }
> }
> ```

> [!success]- Python
> ```python
> class MinStack:
>     def __init__(self): self.s = []
>     def push(self, x):
>         m = min(x, self.s[-1][1]) if self.s else x
>         self.s.append((x, m))
>     def pop(self): self.s.pop()
>     def top(self): return self.s[-1][0]
>     def getMin(self): return self.s[-1][1]
> ```

**Variants:** Max Stack (also support `popMax` — needs doubly linked list).

**Key takeaway:** O(1) min/max on a stack → store running min with each frame.

---

## P3: Evaluate Reverse Polish Notation

**LC #150** · Medium

Eval postfix expression like `["2","1","+","3","*"]`.

**Edge cases:** division truncation toward 0 (Python `int(a/b)`) · single number · negative results.

### 🧠 Pattern: Operand Stack

> Number → push. Operator → pop two operands (right first!), apply, push result.

### Trace

```
tokens=["2","1","+","3","*"]
"2"  st=[2]
"1"  st=[2,1]
"+"  a=2 b=1, push 2+1=3, st=[3]
"3"  st=[3,3]
"*"  a=3 b=3, push 9, st=[9]
return 9
```

> [!success]- JS
> ```js
> const evalRPN = (tokens) => {
>   const st = [];
>   const op = {
>     '+': (a, b) => a + b,
>     '-': (a, b) => a - b,
>     '*': (a, b) => a * b,
>     '/': (a, b) => Math.trunc(a / b),
>   };
>   for (const t of tokens) {
>     if (t in op) {
>       const b = st.pop(), a = st.pop();
>       st.push(op[t](a, b));
>     } else st.push(+t);
>   }
>   return st[0];
> };
> ```

> [!success]- Python
> ```python
> def eval_rpn(tokens):
>     st = []
>     for t in tokens:
>         if t in "+-*/":
>             b, a = st.pop(), st.pop()
>             if t == '+': st.append(a + b)
>             elif t == '-': st.append(a - b)
>             elif t == '*': st.append(a * b)
>             else: st.append(int(a / b))   # truncate toward 0
>         else:
>             st.append(int(t))
>     return st[0]
> ```

> [!warning] Order matters
> Always `b = pop()` then `a = pop()`. `a - b`, `a / b` are NOT commutative.

**Variants:** Infix → Postfix (Shunting Yard) · Basic Calculator (P10).

**Key takeaway:** RPN ↔ stack are isomorphic. Pop right operand first.

---

## P4: Generate Parentheses

**LC #22** · Medium

Generate all valid combinations of `n` pairs of parens.

### 🧠 Pattern: Backtracking with Counts (Implicit Stack via Recursion)

> Track `open` and `close` used so far. You may add `(` if `open < n`, and `)` if `close < open`. The recursion stack IS the stack.

### Trace

```
n=2
""  open=0 close=0
  "("  open=1 close=0
    "(("  open=2 close=0
      "(()"  open=2 close=1
        "(())" ✓
    "()"  open=1 close=1
      "()("  open=2 close=1
        "()()" ✓
return ["(())","()()"]
```

> [!success]- JS
> ```js
> const generateParenthesis = (n) => {
>   const out = [];
>   const dfs = (cur, open, close) => {
>     if (cur.length === 2 * n) { out.push(cur); return; }
>     if (open < n) dfs(cur + '(', open + 1, close);
>     if (close < open) dfs(cur + ')', open, close + 1);
>   };
>   dfs('', 0, 0);
>   return out;
> };
> ```

> [!success]- Python
> ```python
> def generate_parenthesis(n):
>     out = []
>     def dfs(cur, open_, close):
>         if len(cur) == 2 * n:
>             out.append(cur); return
>         if open_ < n: dfs(cur + '(', open_ + 1, close)
>         if close < open_: dfs(cur + ')', open_, close + 1)
>     dfs('', 0, 0)
>     return out
> ```

**Key takeaway:** Some "stack" problems are pure backtracking. The constraint `close < open` is the validity invariant.

---

## P5: Implement Queue using Stacks

**LC #232** · Easy

Build FIFO queue with two LIFO stacks. Amortized O(1) per op.

### 🧠 Pattern: Input Stack + Output Stack

> Push → `in`. Pop/Peek → if `out` empty, transfer all from `in` (reverses order). Each element moves at most twice → amortized O(1).

> [!success]- JS
> ```js
> class MyQueue {
>   constructor() { this.in = []; this.out = []; }
>   push(x) { this.in.push(x); }
>   _shift() {
>     if (!this.out.length) while (this.in.length) this.out.push(this.in.pop());
>   }
>   pop() { this._shift(); return this.out.pop(); }
>   peek() { this._shift(); return this.out.at(-1); }
>   empty() { return !this.in.length && !this.out.length; }
> }
> ```

> [!success]- Python
> ```python
> class MyQueue:
>     def __init__(self):
>         self.in_ = []
>         self.out = []
>     def push(self, x): self.in_.append(x)
>     def _shift(self):
>         if not self.out:
>             while self.in_: self.out.append(self.in_.pop())
>     def pop(self): self._shift(); return self.out.pop()
>     def peek(self): self._shift(); return self.out[-1]
>     def empty(self): return not self.in_ and not self.out
> ```

**Key takeaway:** Reversing twice restores order. Lazy transfer = amortized O(1).

---

## P6: Implement Stack using Queues

**LC #225** · Easy

Build LIFO with FIFO queue(s).

### 🧠 Pattern: Push-and-Rotate

> Single queue. On `push(x)`: enqueue x, then rotate (deq → enq) `size-1` times so x ends up front. O(n) push, O(1) pop.

> [!success]- JS
> ```js
> class MyStack {
>   constructor() { this.q = []; }
>   push(x) {
>     this.q.push(x);
>     for (let i = 0; i < this.q.length - 1; i++) this.q.push(this.q.shift());
>   }
>   pop() { return this.q.shift(); }
>   top() { return this.q[0]; }
>   empty() { return !this.q.length; }
> }
> ```

> [!success]- Python
> ```python
> from collections import deque
> class MyStack:
>     def __init__(self): self.q = deque()
>     def push(self, x):
>         self.q.append(x)
>         for _ in range(len(self.q) - 1):
>             self.q.append(self.q.popleft())
>     def pop(self): return self.q.popleft()
>     def top(self): return self.q[0]
>     def empty(self): return not self.q
> ```

**Key takeaway:** Front-load the cost on push so pop stays O(1).

---

## P7: Asteroid Collision

**LC #735** · Medium

Asteroids moving + or − direction; collisions destroy smaller, both if equal. Return survivors.

**Edge cases:** all same direction (no collisions) · all collide pairwise · equal-size collisions.

### 🧠 Pattern: Stack with Pairwise Resolution

> Push asteroid. While stack top is positive AND incoming negative AND incoming bigger → pop. If equal-and-opposite → pop top and skip incoming.

### Trace

```
[5,10,-5]
push 5      st=[5]
push 10     st=[5,10]
-5: top=10 > 5? 10>5 → -5 dies. skip. → st=[5,10]
return [5,10]

[8,-8]
push 8      st=[8]
-8: top=8 == 8 → both die. → st=[]
```

> [!success]- JS
> ```js
> const asteroidCollision = (a) => {
>   const st = [];
>   for (const x of a) {
>     let alive = true;
>     while (alive && x < 0 && st.length && st.at(-1) > 0) {
>       if (st.at(-1) < -x) st.pop();
>       else if (st.at(-1) === -x) { st.pop(); alive = false; }
>       else alive = false;
>     }
>     if (alive) st.push(x);
>   }
>   return st;
> };
> ```

> [!success]- Python
> ```python
> def asteroid_collision(a):
>     st = []
>     for x in a:
>         alive = True
>         while alive and x < 0 and st and st[-1] > 0:
>             if st[-1] < -x: st.pop()
>             elif st[-1] == -x: st.pop(); alive = False
>             else: alive = False
>         if alive: st.append(x)
>     return st
> ```

**Variants:** Daily Temperatures (Monotonic Stack) · Remove K Digits.

**Key takeaway:** "Pairwise resolution with directionality" → stack. Watch out for the equal case.

---

## P8: Decode String

**LC #394** · Medium

`3[a2[c]]` → `accaccacc`. Decode nested-repeat strings.

### 🧠 Pattern: Two Stacks (Count + Current String)

> On `[`: push current count and current string, reset both. On `]`: pop previous string and count; new current = prev + count × current.

### Trace

```
"3[a2[c]]"
cur="" k=0
'3' k=3
'[' push (3,"") → stacks: counts=[3] strs=[""]; cur="" k=0
'a' cur="a"
'2' k=2
'[' push (2,"a") → counts=[3,2] strs=["","a"]; cur="" k=0
'c' cur="c"
']' pop count=2 str="a"; cur = "a" + "c"*2 = "acc"
']' pop count=3 str=""; cur = "" + "acc"*3 = "accaccacc"
return "accaccacc"
```

> [!success]- JS
> ```js
> const decodeString = (s) => {
>   const counts = [], strs = [];
>   let cur = '', k = 0;
>   for (const c of s) {
>     if (/\d/.test(c)) k = k * 10 + +c;
>     else if (c === '[') { counts.push(k); strs.push(cur); k = 0; cur = ''; }
>     else if (c === ']') { cur = strs.pop() + cur.repeat(counts.pop()); }
>     else cur += c;
>   }
>   return cur;
> };
> ```

> [!success]- Python
> ```python
> def decode_string(s):
>     counts, strs = [], []
>     cur, k = "", 0
>     for c in s:
>         if c.isdigit():
>             k = k * 10 + int(c)
>         elif c == '[':
>             counts.append(k); strs.append(cur)
>             k = 0; cur = ""
>         elif c == ']':
>             cur = strs.pop() + cur * counts.pop()
>         else:
>             cur += c
>     return cur
> ```

> [!warning] Multi-digit numbers
> `k = k * 10 + digit`. Don't reset `k` until you see `[`.

**Variants:** Basic Calculator (recursion on parentheses) · Number of Atoms.

**Key takeaway:** Nested structures → push state on `[`, restore on `]`.

---

## P9: Simplify Path

**LC #71** · Medium

Canonicalize Unix path. `/`, `.`, `..`, multiple slashes.

**Edge cases:** `/../` (stay at root) · `///` · `/...` (this is a real dir) · trailing slash.

### 🧠 Pattern: Stack of Path Components

> Split by `/`. For each part: `""` or `.` → skip; `..` → pop if non-empty; else → push. Join with `/`.

### Trace

```
"/a/./b/../../c/"
parts = ['', 'a', '.', 'b', '..', '..', 'c', '']
''   skip
'a'  push   st=['a']
'.'  skip
'b'  push   st=['a','b']
'..' pop    st=['a']
'..' pop    st=[]
'c'  push   st=['c']
''   skip
return "/c"
```

> [!success]- JS
> ```js
> const simplifyPath = (path) => {
>   const st = [];
>   for (const p of path.split('/')) {
>     if (p === '' || p === '.') continue;
>     if (p === '..') st.pop();
>     else st.push(p);
>   }
>   return '/' + st.join('/');
> };
> ```

> [!success]- Python
> ```python
> def simplify_path(path):
>     st = []
>     for p in path.split('/'):
>         if p == '' or p == '.':
>             continue
>         if p == '..':
>             if st: st.pop()
>         else:
>             st.append(p)
>     return '/' + '/'.join(st)
> ```

**Key takeaway:** Path canonicalization → stack of components. Split → filter → join.

---

## P10: Basic Calculator II

**LC #227** · Medium

Eval `s` with `+ - * /`, no parens, integer division truncates toward 0.

**Edge cases:** spaces · multi-digit numbers · leading `+` (no) but careful with negatives in division.

### 🧠 Pattern: Stack of Terms with Precedence

> Track `prev_op` and current `num`. On hitting next operator (or end), apply `prev_op` to push onto stack: `+num` push, `-num` push, `*` pop·num push, `/` pop÷num push. Final answer = sum of stack.

### Trace

```
s="3+2*2"
num=0 op='+'
'3' num=3
'+' apply prev '+': push 3 → st=[3]. op='+' num=0
'2' num=2
'*' apply prev '+': push 2 → st=[3,2]. op='*' num=0
'2' num=2
end apply prev '*': pop 2, push 2*2=4 → st=[3,4]
sum = 7
```

> [!success]- JS
> ```js
> const calculate = (s) => {
>   const st = [];
>   let num = 0, op = '+';
>   for (let i = 0; i < s.length; i++) {
>     const c = s[i];
>     if (/\d/.test(c)) num = num * 10 + +c;
>     if ((c !== ' ' && !/\d/.test(c)) || i === s.length - 1) {
>       if (op === '+') st.push(num);
>       else if (op === '-') st.push(-num);
>       else if (op === '*') st.push(st.pop() * num);
>       else st.push(Math.trunc(st.pop() / num));
>       op = c; num = 0;
>     }
>   }
>   return st.reduce((a, b) => a + b, 0);
> };
> ```

> [!success]- Python
> ```python
> def calculate(s):
>     st = []
>     num, op = 0, '+'
>     for i, c in enumerate(s):
>         if c.isdigit():
>             num = num * 10 + int(c)
>         if (c != ' ' and not c.isdigit()) or i == len(s) - 1:
>             if op == '+': st.append(num)
>             elif op == '-': st.append(-num)
>             elif op == '*': st.append(st.pop() * num)
>             else: st.append(int(st.pop() / num))
>             op = c; num = 0
>     return sum(st)
> ```

> [!tip] Delaying the operator
> You always apply the **previous** operator (with the current number). This naturally handles `*` and `/` having higher precedence: the multiplicand is the most recent thing on the stack.

**Variants:** Basic Calculator (parens, recursion) · Basic Calculator III (parens + all ops).

**Key takeaway:** Expression eval without parens → delayed-operator stack pattern.

---

> [!tip] After this drill
> See `[` `(` `{` → stack. See "evaluate / parse with precedence" → stack. See "process now, finalize later when ?" → stack.
