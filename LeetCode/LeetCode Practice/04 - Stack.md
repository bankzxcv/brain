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

12 problems · LIFO mechanics — matching, evaluation, design.

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
| P11 | Basic Calculator | 224 | **Hard** | Sign stack with parens |
| P12 | Longest Valid Parentheses | 32 | **Hard** | Stack of indices |

---

## P1: Valid Parentheses

**LC #20** · Easy

`(){}[]` — return true if all brackets are matched and nested correctly.

**Edge cases:** empty (true) · odd length (false) · single bracket (false) · `]` first (false).

### 🧠 Pattern: Push Open, Pop on Close

> Push every open bracket. On a close, the top **must** be its matching open. Empty stack at end = valid.

> [!info]- 🔍 Dry Run: s="([{}])"
> ```text
> Setup:
>   pair = {')':'(', ']':'[', '}':'{'}
>   st = []
> 
> ─────────────────────────────────────────
> c='(' (open) → push
>   st = ['(']
> 
> c='[' (open) → push
>   st = ['(', '[']
> 
> c='{' (open) → push
>   st = ['(', '[', '{']
> 
> c='}' (close)
>   expected = pair['}'] = '{'
>   st.pop() = '{', matches expected → OK
>   st = ['(', '[']
> 
> c=']' (close)
>   expected = '['
>   pop = '[' ✓
>   st = ['(']
> 
> c=')' (close)
>   expected = '('
>   pop = '(' ✓
>   st = []
> 
> End of string, st empty?  → YES
> 
> ✅ Answer: true
> 
> ─────────────────────────────────────────
> Counter-example: s="(]"
>   push '(' → st=['(']
>   ']' close, expected='[', pop='(' ≠ '[' → return false
> ```

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

**Variants:** Minimum Add to Make Parentheses Valid · Score of Parentheses · Longest Valid Parentheses (P12).

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

> [!info]- 🔍 Dry Run: push(-2), push(0), push(-3), getMin, pop, top, getMin
> ```text
> Stack stores (value, running_min) pairs.
> 
> ─────────────────────────────────────────
> push(-2):
>   running min if empty = x itself = -2
>   st = [(-2, -2)]
> 
> push(0):
>   prev min = top.min = -2
>   new running min = min(0, -2) = -2
>   st = [(-2,-2), (0,-2)]
> 
> push(-3):
>   prev min = -2
>   new running min = min(-3, -2) = -3
>   st = [(-2,-2), (0,-2), (-3,-3)]
> 
> getMin() → top.min = -3 ✓
> 
> pop()  → removes (-3,-3)
>   st = [(-2,-2), (0,-2)]
> 
> top()  → top.val = 0
> 
> getMin() → top.min = -2 ✓
> ```

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

> [!info]- 🔍 Dry Run: tokens=["2","1","+","3","*"]
> ```text
> Setup:
>   st = []
> 
> ─────────────────────────────────────────
> token="2"  (number) → push 2
>   st = [2]
> 
> token="1"  → push 1
>   st = [2, 1]
> 
> token="+"  (operator) → pop two, compute, push
>   b = st.pop() = 1   ← right operand FIRST
>   a = st.pop() = 2   ← left operand
>   compute: a + b = 2 + 1 = 3
>   push 3
>   st = [3]
> 
> token="3"  → push
>   st = [3, 3]
> 
> token="*"
>   b = 3, a = 3
>   compute: 3 * 3 = 9
>   push 9
>   st = [9]
> 
> End of tokens. Result = st[0] = 9
> 
> ✅ Answer: 9   (expression "(2+1)*3" = 9)
> ```

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

**Variants:** Infix → Postfix (Shunting Yard) · Basic Calculator (P11).

**Key takeaway:** RPN ↔ stack are isomorphic. Pop right operand first.

---

## P4: Generate Parentheses

**LC #22** · Medium

Generate all valid combinations of `n` pairs of parens.

### 🧠 Pattern: Backtracking with Counts (Implicit Stack via Recursion)

> Track `open` and `close` used so far. You may add `(` if `open < n`, and `)` if `close < open`. The recursion stack IS the stack.

> [!info]- 🔍 Dry Run: n=2
> ```text
> dfs(cur, open, close):
>   if len(cur) == 2n: emit
>   if open < n:  recurse with cur+'(', open+1, close
>   if close < open: recurse with cur+')', open, close+1
> 
> ─────────────────────────────────────────
> dfs("", 0, 0)
>   open=0<2 → dfs("(", 1, 0)
>     open=1<2 → dfs("((", 2, 0)
>       open=2 not < n, skip
>       close=0<2 → dfs("(()", 2, 1)
>         close=1<2 → dfs("(())", 2, 2)
>           len=4=2n → emit "(())"          ← result #1
>     close=0<1 → dfs("()", 1, 1)
>       open=1<2 → dfs("()(", 2, 1)
>         close=1<2 → dfs("()()", 2, 2)
>           emit "()()"                      ← result #2
>       close=1 not < open=1, skip
>   close=0 not < open=0, skip
> 
> ✅ Answer: ["(())", "()()"]
> ```

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

> [!info]- 🔍 Dry Run: push(1), push(2), pop, push(3), pop
> ```text
> Setup:
>   in = []
>   out = []
> 
> ─────────────────────────────────────────
> push(1):  in.push(1)
>   in = [1], out = []
> 
> push(2):  in.push(2)
>   in = [1, 2], out = []
> 
> pop():
>   out empty → transfer all from `in` to `out` (reverses)
>     pop 2 from in → push to out: out=[2]
>     pop 1 from in → push to out: out=[2, 1]
>   in = [], out = [2, 1]
>   pop from out: returns 1   ✓ (FIFO: pushed 1 first, comes out first)
>   in = [], out = [2]
> 
> push(3):  in.push(3)
>   in = [3], out = [2]
> 
> pop():
>   out NOT empty → just pop from out
>   returns 2
>   in = [3], out = []
> 
> ✅ FIFO order preserved: 1, 2, then 3 would come out next.
> ```

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

> [!info]- 🔍 Dry Run: push(1), push(2), push(3), pop, top
> ```text
> Setup:
>   q = []  (queue, front on left)
> 
> ─────────────────────────────────────────
> push(1):
>   enqueue 1 → q = [1]
>   rotate (size-1)=0 times → no-op
>   q = [1]
> 
> push(2):
>   enqueue 2 → q = [1, 2]
>   rotate 1 time: deque front (1), enqueue back → q = [2, 1]
>   q = [2, 1]                    ← top of "stack" is now front of queue
> 
> push(3):
>   enqueue 3 → q = [2, 1, 3]
>   rotate 2 times:
>     iter 1: deque 2, enqueue → q = [1, 3, 2]
>     iter 2: deque 1, enqueue → q = [3, 2, 1]
>   q = [3, 2, 1]                ← stack top is 3, then 2, then 1
> 
> pop() → dequeue front = 3 ✓
>   q = [2, 1]
> 
> top() → front = 2 ✓
> ```

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

> [!info]- 🔍 Dry Run: asteroids=[5, 10, -5]
> ```text
> Setup:
>   st = []
> 
> ─────────────────────────────────────────
> x=5: positive → push (no collision possible with empty stack)
>   st = [5]
> 
> x=10: positive → push
>   st = [5, 10]
> 
> x=-5: negative
>   Check: st.top=10 > 0 AND x<0?  → COLLISION possible
>   Compare |x|=5 vs top=10:
>     5 < 10 → incoming destroyed, top survives → alive=false
>   x=-5 not pushed.
>   st = [5, 10]
> 
> ✅ Answer: [5, 10]
> 
> ─────────────────────────────────────────
> Counter-example: [8, -8]
>   push 8 → st=[8]
>   x=-8: st.top=8>0, x<0 → collision
>     |x|=8, top=8: EQUAL → both destroyed
>       st.pop() → st=[]
>       alive=false; don't push x
>   Result: []
> ```

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

> [!info]- 🔍 Dry Run: s="3[a2[c]]"
> ```text
> Setup:
>   counts = [], strs = []
>   cur = "", k = 0
> 
> ─────────────────────────────────────────
> c='3'  digit → k = 0*10 + 3 = 3
>   State: k=3, cur=""
> 
> c='['  → push k & cur, reset
>   counts.push(3), strs.push("")
>   k = 0, cur = ""
>   counts=[3], strs=[""]
> 
> c='a'  → cur += 'a'
>   cur = "a"
> 
> c='2'  → k = 2
> 
> c='['  → push
>   counts.push(2), strs.push("a")
>   k=0, cur=""
>   counts=[3,2], strs=["", "a"]
> 
> c='c'  → cur = "c"
> 
> c=']'  → pop and combine
>   prev_k = counts.pop() = 2
>   prev_s = strs.pop() = "a"
>   cur = prev_s + cur * prev_k = "a" + "c"*2 = "a" + "cc" = "acc"
>   counts=[3], strs=[""]
>   cur="acc"
> 
> c=']'
>   prev_k = 3, prev_s = ""
>   cur = "" + "acc"*3 = "accaccacc"
>   counts=[], strs=[]
> 
> ✅ Answer: "accaccacc"
> ```

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

> [!info]- 🔍 Dry Run: path="/a/./b/../../c/"
> ```text
> Setup:
>   split by '/': parts = ['', 'a', '.', 'b', '..', '..', 'c', '']
>   st = []
> 
> ─────────────────────────────────────────
> ''     → skip (empty from leading/duplicate slashes)
> 'a'    → push     → st=['a']
> '.'    → skip (current dir)
> 'b'    → push     → st=['a', 'b']
> '..'   → pop if non-empty: pop 'b' → st=['a']
> '..'   → pop 'a' → st=[]
> 'c'    → push    → st=['c']
> ''     → skip (trailing slash)
> 
> Final: '/' + '/'.join(st) = '/' + 'c' = '/c'
> 
> ✅ Answer: "/c"
> ```

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

> [!info]- 🔍 Dry Run: s="3+2*2"
> ```text
> Setup:
>   st = []
>   num = 0
>   op = '+'  (pretend a leading +)
> 
> ─────────────────────────────────────────
> i=0, c='3'  digit → num = 0*10 + 3 = 3
> 
> i=1, c='+'  operator → apply PREVIOUS op (which is '+') with current num
>   op was '+' → st.push(num=3)
>   st = [3]
>   op = '+'   (the new one)
>   num = 0
> 
> i=2, c='2'  → num = 2
> 
> i=3, c='*'  → apply previous op '+'
>   st.push(2) → st=[3, 2]
>   op = '*'
>   num = 0
> 
> i=4, c='2'  → num = 2
>   At end of string (i == n-1) → apply prev op '*'
>   op '*' → pop top, multiply by num: 2 * 2 = 4 → push
>   st = [3, 4]
> 
> Sum st = 3 + 4 = 7
> 
> ✅ Answer: 7
> 
> ─────────────────────────────────────────
> Example with /: s="14-3/2"
>   num=14, '-': st=[14], op='-'
>   num=3, '/': st=[14,-3], op='/'
>   num=2, end: op='/' → pop -3, int(-3/2)=-1, push → st=[14,-1]
>   Sum = 13
>   (Python: int(-3/2) truncates toward 0 → -1, not -2)
> ```

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

**Variants:** Basic Calculator (parens, P11) · Basic Calculator III (parens + all ops).

**Key takeaway:** Expression eval without parens → delayed-operator stack pattern.

---

## P11: Basic Calculator

**LC #224** · **Hard**

Eval `s` with `+ -` and **parentheses** (no `*` `/`).

**Edge cases:** unary minus `-(2+3)` · spaces · deeply nested parens.

### 🧠 Pattern: Sign Stack with Parentheses

> Walk left-to-right. Track `result`, current `num`, and a running `sign` (+1 or -1). On `(`, push current `result` and `sign` onto stack and reset. On `)`, fold: `result = popped_sign * result + popped_result`.

> [!info]- 🔍 Dry Run: s="1+(2-(3+4))"
> ```text
> Setup:
>   result = 0, num = 0, sign = +1
>   st = []
> 
> ─────────────────────────────────────────
> c='1'  num=1
> 
> c='+'  apply: result += sign * num = 0 + 1*1 = 1
>        sign = +1, num = 0
>        State: result=1, sign=+1
> 
> c='('  push (result, sign) and reset
>        st.push(result=1), st.push(sign=+1)
>        st = [1, +1]
>        result = 0, sign = +1, num = 0
> 
> c='2'  num=2
> 
> c='-'  result += sign*num = 0 + 1*2 = 2
>        sign = -1, num = 0
>        State: result=2, sign=-1
> 
> c='('  push and reset
>        st.push(result=2), st.push(sign=-1)
>        st = [1, +1, 2, -1]
>        result=0, sign=+1, num=0
> 
> c='3'  num=3
> 
> c='+'  result += 1*3 = 3
>        sign=+1, num=0
>        State: result=3
> 
> c='4'  num=4
> 
> c=')'  apply pending num: result += 1*4 = 7
>        Now fold parenthesis:
>          popped_sign = st.pop() = -1
>          popped_res  = st.pop() = 2
>          result = popped_sign * result + popped_res
>                 = -1 * 7 + 2 = -5
>        st = [1, +1]
>        sign=+1, num=0
>        State: result=-5
> 
> c=')'  fold:
>          popped_sign = +1
>          popped_res  = 1
>          result = 1 * -5 + 1 = -4
>        st = []
>        State: result=-4
> 
> End of string. (No pending num since last was ')'.)
> 
> ✅ Answer: -4
>   Verify: 1 + (2 - (3 + 4)) = 1 + (2 - 7) = 1 + (-5) = -4 ✓
> ```

> [!success]- JS
> ```js
> const calculate1 = (s) => {
>   const st = [];
>   let result = 0, num = 0, sign = 1;
>   for (const c of s) {
>     if (/\d/.test(c)) {
>       num = num * 10 + +c;
>     } else if (c === '+' || c === '-') {
>       result += sign * num;
>       num = 0;
>       sign = c === '+' ? 1 : -1;
>     } else if (c === '(') {
>       st.push(result);
>       st.push(sign);
>       result = 0; sign = 1;
>     } else if (c === ')') {
>       result += sign * num;
>       num = 0;
>       const popSign = st.pop();
>       const popRes = st.pop();
>       result = popSign * result + popRes;
>     }
>   }
>   return result + sign * num;
> };
> ```

> [!success]- Python
> ```python
> def calculate1(s):
>     st = []
>     result, num, sign = 0, 0, 1
>     for c in s:
>         if c.isdigit():
>             num = num * 10 + int(c)
>         elif c in '+-':
>             result += sign * num
>             num = 0
>             sign = 1 if c == '+' else -1
>         elif c == '(':
>             st.append(result)
>             st.append(sign)
>             result, sign = 0, 1
>         elif c == ')':
>             result += sign * num
>             num = 0
>             result = st.pop() * result + st.pop()
>     return result + sign * num
> ```

> [!tip] Order of pops
> When you pop for `)`, sign comes off first (most recently pushed), then result. Mirror the push order.

**Variants:** Basic Calculator III (`+ - * /` plus parens) — combine P10 and P11.

**Key takeaway:** Parens + sign → push the **outside context** (result so far + applied sign), reset, recurse. Stack saves snapshots.

---

## P12: Longest Valid Parentheses

**LC #32** · **Hard**

Longest substring of properly matched parentheses.

**Edge cases:** all `(` (zero) · all `)` (zero) · `"()(()"` (best is 2, not 4).

### 🧠 Pattern: Stack of Indices with a Sentinel

> Push **indices** (not chars). Initialize with `-1` as a sentinel "base of valid run". On `(`, push index. On `)`: pop top; if stack now empty, push current index as new base; else, current valid length = `i - top of stack`.

> [!info]- 🔍 Dry Run: s=")()())"
> ```text
> Setup:
>   st = [-1]      ← sentinel: base of "current run"
>   best = 0
> 
> ─────────────────────────────────────────
> i=0, c=')'
>   pop top (-1) → st = []
>   st empty → push i=0 as new base → st=[0]
>   (no valid run continued)
> 
> i=1, c='('
>   push 1 → st=[0, 1]
> 
> i=2, c=')'
>   pop top (1) → st=[0]
>   not empty: top is 0 (current base)
>   valid length = i - top = 2 - 0 = 2
>   best = max(0, 2) = 2
> 
> i=3, c='('
>   push 3 → st=[0, 3]
> 
> i=4, c=')'
>   pop 3 → st=[0]
>   length = 4 - 0 = 4
>   best = max(2, 4) = 4   ✓
> 
> i=5, c=')'
>   pop 0 → st=[]
>   st empty → push 5 as new base → st=[5]
> 
> ✅ Answer: 4   (the substring "()()" at positions 1..4)
> 
> ─────────────────────────────────────────
> Counter-example: s="((()"
>   st=[-1]
>   i=0 '(' push: st=[-1,0]
>   i=1 '(' push: st=[-1,0,1]
>   i=2 '(' push: st=[-1,0,1,2]
>   i=3 ')' pop 2: st=[-1,0,1]
>      length = 3 - 1 = 2 → best=2
>   Answer: 2
> ```

> [!success]- JS
> ```js
> const longestValidParentheses = (s) => {
>   const st = [-1];
>   let best = 0;
>   for (let i = 0; i < s.length; i++) {
>     if (s[i] === '(') {
>       st.push(i);
>     } else {
>       st.pop();
>       if (!st.length) st.push(i);
>       else best = Math.max(best, i - st.at(-1));
>     }
>   }
>   return best;
> };
> ```

> [!success]- Python
> ```python
> def longest_valid_parentheses(s):
>     st = [-1]
>     best = 0
>     for i, c in enumerate(s):
>         if c == '(':
>             st.append(i)
>         else:
>             st.pop()
>             if not st:
>                 st.append(i)
>             else:
>                 best = max(best, i - st[-1])
>     return best
> ```

> [!tip] Two-pass O(1)-space alternative
> Sweep left-to-right tracking `open`/`close` counts; if `close > open`, reset; if equal, update best with `2*open`. Then sweep right-to-left similarly. Catches the `"(()"` case the single sweep misses.

**Variants:** Generate Parentheses (P4) · Score of Parentheses · Min Add to Make Parens Valid.

**Key takeaway:** "Longest valid" → stack of indices, sentinel base. Length = `current_index - top_of_stack` always.

---

> [!tip] After this drill
> See `[` `(` `{` → stack. See "evaluate / parse with precedence" → stack. See "process now, finalize later when ?" → stack.
