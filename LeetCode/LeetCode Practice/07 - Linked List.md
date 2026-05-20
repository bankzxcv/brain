---
title: "LeetCode Practice: Linked List"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - linked-list
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Linked List

14 problems · pointer manipulation, cycle detection, merging, in-place rewiring.

> [!abstract] Pattern recap
> Three core techniques:
> 1. **Dummy head** for clean head-insertion / removal logic
> 2. **Two pointers** (fast/slow or N-ahead) for cycle/midpoint/Nth-from-end
> 3. **Reverse in place** — `prev/curr/next` three-pointer dance

> [!warning] Always draw before you code
> Linked-list problems are paper-friendly. Draw 3 nodes, label `prev/curr/next` at every step. Coding without drawing = bugs.

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Reverse Linked List | 206 | Easy | In-place reverse |
| P2 | Merge Two Sorted Lists | 21 | Easy | Dummy + interleave |
| P3 | Linked List Cycle | 141 | Easy | Floyd |
| P4 | Linked List Cycle II | 142 | Med | Floyd + math reset |
| P5 | Remove Nth Node From End | 19 | Med | N-ahead pointer |
| P6 | Add Two Numbers | 2 | Med | Carry + dummy |
| P7 | Copy List with Random Pointer | 138 | Med | Hash map (or weave-clone) |
| P8 | Reorder List | 143 | Med | Midpoint + reverse + merge |
| P9 | Reverse Nodes in k-Group | 25 | **Hard** | Reverse with anchor |
| P10 | LRU Cache | 146 | Med | Hash + doubly linked list |
| P11 | Remove Linked List Elements | 203 | Easy | Dummy filter |
| P12 | Palindrome Linked List | 234 | Easy | Mid + reverse + compare |
| P13 | Intersection of Two Linked Lists | 160 | Easy | Length difference |
| P14 | Merge k Sorted Lists | 23 | **Hard** | Min-heap |

---

## P1: Reverse Linked List

**LC #206** · Easy

Reverse a singly linked list.

**Edge cases:** empty · single node · two nodes.

### 🧠 Pattern: Three-Pointer Reverse

> `prev`, `curr`, `next`. Save `next`, point `curr.next ← prev`, advance both. Return `prev` (new head).

> [!example]- 📊 Visual: pointer dance
> ```text
>   Initial:
>     prev      curr
>      ↓         ↓
>     null     [ 1 ] → [ 2 ] → [ 3 ] → null
> 
>   After step 1 (flip 1's arrow):
>                       prev      curr
>                        ↓         ↓
>     null ← [ 1 ]     [ 2 ] → [ 3 ] → null
> 
>   After step 2 (flip 2's arrow):
>                                  prev      curr
>                                   ↓         ↓
>     null ← [ 1 ] ← [ 2 ]        [ 3 ] → null
> 
>   After step 3 (flip 3's arrow):
>                                            prev      curr
>                                             ↓         ↓
>     null ← [ 1 ] ← [ 2 ] ← [ 3 ]          null
> 
>   Return prev = node(3). Reversed list:  3 → 2 → 1 → null
> ```

> [!info]- 🔍 Dry Run: 1 → 2 → 3 → null
> ```text
> Setup:
>   prev = null
>   curr = head = node(1)
> 
> ─────────────────────────────────────────
> Iter 1: curr = node(1)
>   nxt = curr.next = node(2)
>   curr.next = prev    (i.e., node(1).next = null)
>   prev = curr         → prev = node(1)
>   curr = nxt          → curr = node(2)
>   List now: null ← 1   ;   curr=2 → 3 → null
> 
> Iter 2: curr = node(2)
>   nxt = node(3)
>   node(2).next = prev = node(1)
>   prev = node(2)
>   curr = node(3)
>   List now: null ← 1 ← 2   ;   curr=3 → null
> 
> Iter 3: curr = node(3)
>   nxt = null
>   node(3).next = node(2)
>   prev = node(3)
>   curr = null
>   List now: null ← 1 ← 2 ← 3
> 
> Loop exit (curr is null).
> 
> ✅ Answer: prev = node(3) (new head)
>   Reversed list: 3 → 2 → 1 → null
> ```

> [!success]- JS
> ```js
> const reverseList = (head) => {
>   let prev = null, curr = head;
>   while (curr) {
>     const nxt = curr.next;
>     curr.next = prev;
>     prev = curr;
>     curr = nxt;
>   }
>   return prev;
> };
> ```

> [!success]- Python
> ```python
> def reverse_list(head):
>     prev, curr = None, head
>     while curr:
>         nxt = curr.next
>         curr.next = prev
>         prev = curr
>         curr = nxt
>     return prev
> ```

**Variants:** Reverse between l and r · Reverse k-group (P9).

**Key takeaway:** The 5-line three-pointer dance. Memorize.

---

## P2: Merge Two Sorted Lists

**LC #21** · Easy

### 🧠 Pattern: Dummy Head + Tail Pointer

> Dummy avoids special-casing the head. Walk both lists, append the smaller; advance.

> [!example]- 📊 Visual: dummy + tail interleave
> ```text
>   l1:   [ 1 ] → [ 2 ] → [ 4 ] → null
>   l2:   [ 1 ] → [ 3 ] → [ 4 ] → null
> 
>   dummy → null         tail
>     ↑                    ↑
>   dummy                dummy
> 
>   Pick smaller head; append to tail.next; advance that list.
> 
>   Step 1: l1.val(1) <= l2.val(1) → take l1
>     dummy → [1] → null
>                ↑
>              tail
>     l1 = [2] → [4]
> 
>   Step 2: l1.val(2) > l2.val(1) → take l2
>     dummy → [1] → [1] → null
>                       ↑
>                     tail
> 
>   Step 3: 2 <= 3 → take l1
>     dummy → [1] → [1] → [2] → null
>                              ↑
>                            tail
> 
>   ... continue until one list empties ...
> 
>   Splice leftover tail (whichever list still has nodes):
>     dummy → 1 → 1 → 2 → 3 → 4 → 4 → null
> 
>   Return dummy.next  (skips the sentinel)
> ```

> [!info]- 🔍 Dry Run: l1=1→2→4, l2=1→3→4
> ```text
> Setup:
>   dummy = node(0)
>   tail  = dummy
> 
> ─────────────────────────────────────────
> Iter 1: l1=1, l2=1
>   l1.val(1) <= l2.val(1) → take l1
>   tail.next = l1                   (dummy → 1)
>   l1 = l1.next = node(2)
>   tail = node(1)
>   State: dummy → 1
> 
> Iter 2: l1=2, l2=1
>   l1.val(2) <= l2.val(1)? NO → take l2
>   tail.next = l2                   (dummy → 1 → 1)
>   l2 = node(3)
>   tail = node(1) [the second one]
>   State: dummy → 1 → 1
> 
> Iter 3: l1=2, l2=3
>   2 <= 3 → take l1
>   tail.next = l1
>   l1 = node(4)
>   tail = node(2)
>   State: dummy → 1 → 1 → 2
> 
> Iter 4: l1=4, l2=3
>   4 <= 3? NO → take l2
>   tail.next = l2
>   l2 = node(4)
>   tail = node(3)
>   State: dummy → 1 → 1 → 2 → 3
> 
> Iter 5: l1=4, l2=4
>   4 <= 4 → take l1
>   tail.next = l1
>   l1 = null
>   tail = node(4)
>   State: dummy → 1 → 1 → 2 → 3 → 4
> 
> Loop exits (l1 is null).
> 
> tail.next = l2 (or l1, whichever has remaining)
>             = node(4) (from l2)
>   State: dummy → 1 → 1 → 2 → 3 → 4 → 4
> 
> ✅ Answer: dummy.next = 1 → 1 → 2 → 3 → 4 → 4
> ```

> [!success]- JS
> ```js
> const mergeTwoLists = (l1, l2) => {
>   const dummy = { next: null };
>   let tail = dummy;
>   while (l1 && l2) {
>     if (l1.val <= l2.val) { tail.next = l1; l1 = l1.next; }
>     else { tail.next = l2; l2 = l2.next; }
>     tail = tail.next;
>   }
>   tail.next = l1 ?? l2;
>   return dummy.next;
> };
> ```

> [!success]- Python
> ```python
> def merge_two_lists(l1, l2):
>     dummy = ListNode(0)
>     tail = dummy
>     while l1 and l2:
>         if l1.val <= l2.val:
>             tail.next = l1; l1 = l1.next
>         else:
>             tail.next = l2; l2 = l2.next
>         tail = tail.next
>     tail.next = l1 or l2
>     return dummy.next
> ```

**Key takeaway:** Dummy + tail = the cleanest list-building idiom.

---

## P3: Linked List Cycle

**LC #141** · Easy

(See [[02 - Two Pointers]] P11 for the full discussion of Floyd's tortoise & hare.)

> [!example]- 📊 Visual: tortoise and hare on a ρ-shape
> ```text
>   List with a cycle (shape resembles the Greek letter ρ "rho"):
> 
>     head → [3] → [2] → [0] → [-4]
>                   ↑           │
>                   └───────────┘   ← back-edge forms the loop
> 
>   slow steps 1×, fast steps 2×. Trace from head:
> 
>     t=0:  slow=3,  fast=3
>     t=1:  slow=2,  fast=0
>     t=2:  slow=0,  fast=2     (fast wrapped around once)
>     t=3:  slow=-4, fast=-4    ← MEET inside the cycle ✅
> 
>   If there were no cycle, fast (or fast.next) would hit null:
> 
>     head → [1] → [2] → [3] → null
>                                ↑
>                            fast eventually here → return False
> 
>   Mental model:
>     • On a circular track, the faster runner laps the slower one.
>     • On a straight road, the faster runner just exits the road.
> ```

> [!success]- Python
> ```python
> def has_cycle(head):
>     slow = fast = head
>     while fast and fast.next:
>         slow = slow.next
>         fast = fast.next.next
>         if slow is fast: return True
>     return False
> ```

---

## P4: Linked List Cycle II

**LC #142** · Medium

Return the node where the cycle begins, or None.

### 🧠 Pattern: Floyd + Math Reset

> After they meet inside the cycle, reset one pointer to head and step both 1× at a time. They meet at the cycle start.
> 
> **Why?** Let `H` = distance from head to cycle start, `D` = distance from cycle start to meeting point, `C` = cycle length. Slow traveled `H+D`; fast traveled `H+D+kC`; fast = 2× slow → `H+D = kC` → `H = kC - D`. So `H` steps from head and `H` steps from meeting point both land at start.

> [!example]- 📊 Visual: cycle geometry
> ```text
>             ┌─── H ───┐
>     head → [A] → [B] → [C] ← cycle entry
>                         ↓
>                        [D]
>                         ↓
>                        [E]              cycle length C
>                         ↓
>                        [F]
>                         ↓
>     meeting → [M] ← ← ← [G]
>                ↑
>                slow and fast first meet HERE
> 
>   Walking distances at meeting:
>     slow  walks: H + D       (D = entry → meeting along cycle)
>     fast  walks: H + D + kC  (some extra full loops)
>     fast = 2 × slow
>     →  H + D + kC = 2(H + D)
>     →  H = kC - D
>     →  H ≡ -D  (mod C)
>     →  H steps from M (going forward in cycle) lands at entry C
> 
>   Phase 2 — reset one pointer to head, step both 1× each:
>     p1 from head:       walks H steps, lands at cycle entry
>     p2 from meeting:    walks H steps, also lands at cycle entry
>     They MEET at the cycle entry!  ← return this node.
> ```

> [!info]- 🔍 Dry Run: 3 → 2 → 0 → -4 → (back to 2)
> ```text
> Nodes: A=3, B=2, C=0, D=-4; D.next = B (forms cycle 2→0→-4→2)
> 
> Phase 1: Floyd until meeting
>   slow=A, fast=A
>   step 1: slow=B, fast=C
>   step 2: slow=C, fast=B    (D→B is cycle back)
>   step 3: slow=D, fast=D    MEET at node D
> 
> ─────────────────────────────────────────
> Phase 2: Reset, step both 1x
>   p = head = A
>   slow stays at D (the meeting point)
> 
>   iter: p=A, slow=D
>     p != slow → advance both
>     p = B, slow = D.next = B
>   iter: p=B, slow=B → MATCH
> 
> ✅ Answer: cycle starts at node B (value 2)
> ```

> [!success]- JS
> ```js
> const detectCycle = (head) => {
>   let slow = head, fast = head;
>   while (fast && fast.next) {
>     slow = slow.next;
>     fast = fast.next.next;
>     if (slow === fast) {
>       let p = head;
>       while (p !== slow) { p = p.next; slow = slow.next; }
>       return p;
>     }
>   }
>   return null;
> };
> ```

> [!success]- Python
> ```python
> def detect_cycle(head):
>     slow = fast = head
>     while fast and fast.next:
>         slow = slow.next
>         fast = fast.next.next
>         if slow is fast:
>             p = head
>             while p is not slow:
>                 p = p.next
>                 slow = slow.next
>             return p
>     return None
> ```

**Key takeaway:** Memorize the math; this is a Google favorite.

---

## P5: Remove Nth Node From End

**LC #19** · Medium

### 🧠 Pattern: N-Ahead Pointer + Dummy

> Move `fast` n+1 steps ahead. Then walk both until `fast` is null — `slow` is at the node **before** the one to remove.

> [!example]- 📊 Visual: N-ahead pointer
> ```text
>   List: 1 → 2 → 3 → 4 → 5,  n=2  (remove 2nd-from-end → remove 4)
> 
>   Goal: `slow` should land on node 3 (the one BEFORE target node 4).
>         If slow is at 3, then slow.next = 4 (remove) and slow.next = slow.next.next.
> 
>   Use a DUMMY → 1 → 2 → 3 → 4 → 5
>   slow = dummy   fast = dummy
> 
>   Phase 1: move fast (n+1)=3 steps ahead:
> 
>     dummy ─ 1 ─ 2 ─ 3 ─ 4 ─ 5
>       ↑           ↑
>      slow        fast      (gap of 3 nodes)
> 
>   Phase 2: walk BOTH until fast hits null:
> 
>     step 1:   slow=1,  fast=4
>     dummy ─ 1 ─ 2 ─ 3 ─ 4 ─ 5
>             ↑                    ↑
>            slow                  fast (kept the same offset)
> 
>     step 2:   slow=2,  fast=5
>     step 3:   slow=3,  fast=null  ← STOP, slow is the predecessor
> 
>     dummy ─ 1 ─ 2 ─ 3 ─ 4 ─ 5
>                     ↑   ╳ remove this
>                    slow
> 
>   Action:  slow.next = slow.next.next     →   1 ─ 2 ─ 3 ─ 5
> 
>   The dummy elegantly handles the case "remove the head".
> ```

> [!info]- 🔍 Dry Run: 1→2→3→4→5, n=2
> ```text
> Goal: remove the 2nd-from-end → remove node(4).
> 
> Setup:
>   dummy → 1 → 2 → 3 → 4 → 5
>   slow = dummy, fast = dummy
> 
> ─────────────────────────────────────────
> Phase 1 — Move fast (n+1)=3 steps ahead:
>   fast = node(1)
>   fast = node(2)
>   fast = node(3)
> 
>   Now fast is 3 nodes ahead of slow (dummy → ... → 3).
> 
> ─────────────────────────────────────────
> Phase 2 — Move both until fast is null:
>   iter: slow=node(1), fast=node(4)
>   iter: slow=node(2), fast=node(5)
>   iter: slow=node(3), fast=null  → exit
> 
> slow is at node(3) — the one BEFORE the target.
> 
> Phase 3 — Remove slow.next:
>   slow.next = slow.next.next     (node(3).next = node(5))
>   State: dummy → 1 → 2 → 3 → 5
> 
> ✅ Answer: 1 → 2 → 3 → 5
> ```

> [!success]- JS
> ```js
> const removeNthFromEnd = (head, n) => {
>   const dummy = { next: head };
>   let slow = dummy, fast = dummy;
>   for (let i = 0; i <= n; i++) fast = fast.next;
>   while (fast) { slow = slow.next; fast = fast.next; }
>   slow.next = slow.next.next;
>   return dummy.next;
> };
> ```

> [!success]- Python
> ```python
> def remove_nth_from_end(head, n):
>     dummy = ListNode(0, head)
>     slow = fast = dummy
>     for _ in range(n + 1):
>         fast = fast.next
>     while fast:
>         slow = slow.next
>         fast = fast.next
>     slow.next = slow.next.next
>     return dummy.next
> ```

**Key takeaway:** N-from-end → fast-pointer offset trick. Dummy handles "remove head" edge case.

---

## P6: Add Two Numbers

**LC #2** · Medium

Two lists represent reversed integers. Add and return as linked list.

### Approach

Walk both, sum digits + carry, append.

> [!example]- 📊 Visual: digit-by-digit with carry
> ```text
>   Numbers stored REVERSED (least-significant digit first):
> 
>     l1:  [ 2 ] → [ 4 ] → [ 3 ]      represents  342
>     l2:  [ 5 ] → [ 6 ] → [ 4 ]      represents  465
> 
>   Column-add like grade school, but starting from the left
>   (because LSB is on the left here):
> 
>         2   4   3
>       + 5   6   4
>       ─────────────
>     carry → propagates right ──►
> 
>     col 0:  2 + 5 + 0 = 7    digit=7  carry=0
>     col 1:  4 + 6 + 0 = 10   digit=0  carry=1
>     col 2:  3 + 4 + 1 = 8    digit=8  carry=0
> 
>   Build result via dummy + tail (one node per column):
> 
>     dummy → [7]
>     dummy → [7] → [0]
>     dummy → [7] → [0] → [8]      represents 807  ✅
> 
>   Edge cases shown by the carry rail:
>     • Lists of UNEQUAL length: missing digit treated as 0.
>     • Final carry: e.g. 5 + 5 → [0] → [1]  (extra node!).
>       Loop condition: `while l1 or l2 or carry`.
> ```

> [!info]- 🔍 Dry Run: l1=2→4→3 (=342), l2=5→6→4 (=465); expected 807 → 7→0→8
> ```text
> Setup:
>   dummy = node(0), tail = dummy
>   carry = 0
> 
> ─────────────────────────────────────────
> Iter 1: l1=2, l2=5, carry=0
>   sum = 2 + 5 + 0 = 7
>   carry = 7 // 10 = 0
>   digit = 7 % 10 = 7
>   append node(7); tail moves
>   State: dummy → 7
> 
> Iter 2: l1=4, l2=6, carry=0
>   sum = 4 + 6 + 0 = 10
>   carry = 1, digit = 0
>   append node(0)
>   State: dummy → 7 → 0
> 
> Iter 3: l1=3, l2=4, carry=1
>   sum = 3 + 4 + 1 = 8
>   carry = 0, digit = 8
>   append node(8)
>   State: dummy → 7 → 0 → 8
> 
> Loop exits (both lists end, carry = 0).
> 
> ✅ Answer: 7 → 0 → 8   (reads as 807)
> ```

> [!success]- JS
> ```js
> const addTwoNumbers = (l1, l2) => {
>   const dummy = { next: null };
>   let tail = dummy, carry = 0;
>   while (l1 || l2 || carry) {
>     const s = (l1?.val ?? 0) + (l2?.val ?? 0) + carry;
>     carry = Math.floor(s / 10);
>     tail.next = { val: s % 10, next: null };
>     tail = tail.next;
>     l1 = l1?.next; l2 = l2?.next;
>   }
>   return dummy.next;
> };
> ```

> [!success]- Python
> ```python
> def add_two_numbers(l1, l2):
>     dummy = ListNode(0)
>     tail = dummy
>     carry = 0
>     while l1 or l2 or carry:
>         s = (l1.val if l1 else 0) + (l2.val if l2 else 0) + carry
>         carry, digit = divmod(s, 10)
>         tail.next = ListNode(digit)
>         tail = tail.next
>         l1 = l1.next if l1 else None
>         l2 = l2.next if l2 else None
>     return dummy.next
> ```

**Key takeaway:** Carry-propagation in one loop. Don't forget the trailing carry.

---

## P7: Copy List with Random Pointer

**LC #138** · Medium

Each node has `next` AND `random`. Deep copy.

### 🧠 Pattern: Hash Map old→new

> Pass 1: clone each node, map `old → new`. Pass 2: wire next/random via the map.

> [!example]- 📊 Visual: clone + rewire via old→new map
> ```text
>   Original list with `next` (straight) and `random` (curved) pointers:
> 
>          ┌──────── random ────────┐
>          │                        ▼
>     [ A:7 ] ──next──► [ B:13 ] ──next──► [ C:11 ] ──► null
>          ▲                                   │
>          └────────── random ─────────────────┘
>          B.random = A
>          C.random = B
>          A.random = null
> 
>   ─ Pass 1: clone each node (values only, no pointers yet) ─
> 
>     map:  A ──► A'(7)
>           B ──► B'(13)
>           C ──► C'(11)
> 
>     Cloned nodes initially disconnected:
> 
>           [A':7]      [B':13]      [C':11]
> 
>   ─ Pass 2: wire next & random by LOOKING UP in the map ─
> 
>     For each old node p:
>       map[p].next   = map[p.next]    (or null)
>       map[p].random = map[p.random]  (or null)
> 
>          ┌──────── random ────────┐
>          │                        ▼
>     [A':7 ] ──next──► [B':13] ──next──► [C':11] ──► null
>          ▲                                   │
>          └────────── random ─────────────────┘
> 
>   Why this works:
>     The hash map gives O(1) lookup from any OLD node to its CLONE,
>     so wiring random pointers (which can jump anywhere) is trivial.
> ```

> [!info]- 🔍 Dry Run: Original A(7)→B(13)→C(11) with B.random=A, C.random=B
> ```text
> Pass 1 — clone each node, build old→new map:
>   visit A: create A' (val 7); map[A] = A'
>   visit B: create B' (val 13); map[B] = B'
>   visit C: create C' (val 11); map[C] = C'
>   map = {A: A', B: B', C: C'}
> 
> ─────────────────────────────────────────
> Pass 2 — wire pointers via map:
>   A':  A'.next   = map[A.next]   = map[B] = B'
>        A'.random = map[A.random] = map[null] = null
> 
>   B':  B'.next   = map[B.next]   = map[C] = C'
>        B'.random = map[B.random] = map[A] = A'
> 
>   C':  C'.next   = map[C.next]   = map[null] = null
>        C'.random = map[C.random] = map[B] = B'
> 
> ✅ Answer: A' → B' → C' with proper random pointers preserved
> ```

> [!success]- JS
> ```js
> const copyRandomList = (head) => {
>   if (!head) return null;
>   const map = new Map();
>   for (let p = head; p; p = p.next) map.set(p, { val: p.val, next: null, random: null });
>   for (let p = head; p; p = p.next) {
>     map.get(p).next = map.get(p.next) ?? null;
>     map.get(p).random = map.get(p.random) ?? null;
>   }
>   return map.get(head);
> };
> ```

> [!success]- Python
> ```python
> def copy_random_list(head):
>     if not head: return None
>     mp = {}
>     p = head
>     while p:
>         mp[p] = Node(p.val)
>         p = p.next
>     p = head
>     while p:
>         mp[p].next = mp.get(p.next)
>         mp[p].random = mp.get(p.random)
>         p = p.next
>     return mp[head]
> ```

> [!tip] O(1) extra space alternative
> Weave clones into the original list (A→A'→B→B'→...), set randoms via `p.next`, then unweave.

**Key takeaway:** When auxiliary connections need cloning, hash original→clone.

---

## P8: Reorder List

**LC #143** · Medium

`L0 → L1 → ... → Ln` becomes `L0 → Ln → L1 → Ln-1 → ...`.

### 🧠 Pattern: Midpoint + Reverse + Interleave

> 1. Find midpoint (fast/slow). 2. Reverse second half. 3. Merge two halves alternating.

> [!example]- 📊 Visual: split → reverse → interleave
> ```text
>   Input:    1 → 2 → 3 → 4 → 5
> 
>   ┌─ Step 1: find midpoint (slow/fast pointers) ──────────┐
>   │                                                       │
>   │  After fast/slow runs:                                │
>   │                                                       │
>   │      head    slow                                     │
>   │       ↓       ↓                                       │
>   │       1 → 2 → 3 → 4 → 5                               │
>   │                                                       │
>   └───────────────────────────────────────────────────────┘
> 
>   ┌─ Step 2: split into two halves, reverse the second ───┐
>   │                                                       │
>   │      Half A:    1 → 2 → 3 → null                      │
>   │      Half B:    4 → 5 → null                          │
>   │                                                       │
>   │   Reverse B:    5 → 4 → null                          │
>   │                                                       │
>   └───────────────────────────────────────────────────────┘
> 
>   ┌─ Step 3: interleave A and reversed B ─────────────────┐
>   │                                                       │
>   │      A: 1 → 2 → 3                                     │
>   │      B: 5 → 4                                         │
>   │                                                       │
>   │   Pick alternately:                                   │
>   │      1 → 5 → 2 → 4 → 3                                │
>   │                                                       │
>   └───────────────────────────────────────────────────────┘
> 
>   Result: 1 → 5 → 2 → 4 → 3 → null
> 
>   Three composable subroutines, none of them new:
>     midpoint    (fast/slow)
>     reverse     (3-pointer dance)
>     merge       (dummy + tail)
> ```

> [!info]- 🔍 Dry Run: 1 → 2 → 3 → 4 → 5
> ```text
> Phase 1 — Find midpoint (slow ends at middle):
>   slow=1, fast=1
>   slow=2, fast=3
>   slow=3, fast=5  (fast.next=null, stop)
>   slow = node(3) — the middle.
> 
> ─────────────────────────────────────────
> Phase 2 — Reverse second half (after slow):
>   Take slow.next = 4 → 5
>   Reverse: 5 → 4 → null
>   Cut: slow.next = null
>   Now: first half = 1 → 2 → 3 → null
>        second half (prev) = 5 → 4 → null
> 
> ─────────────────────────────────────────
> Phase 3 — Interleave:
>   a = 1, b = 5
>   iter 1:
>     a_n = a.next = 2
>     b_n = b.next = 4
>     a.next = b           →  1 → 5
>     b.next = a_n         →  5 → 2
>     a = a_n = 2, b = b_n = 4
>     State: 1 → 5 → 2 → 3 → null   ;   b=4 → null
> 
>   iter 2:
>     a_n = 3
>     b_n = null
>     a.next = b           →  2 → 4
>     b.next = a_n         →  4 → 3
>     a=3, b=null
>     State: 1 → 5 → 2 → 4 → 3 → null
> 
>   b is null → loop ends.
> 
> ✅ Answer: 1 → 5 → 2 → 4 → 3
> ```

> [!success]- JS
> ```js
> const reorderList = (head) => {
>   if (!head || !head.next) return;
>   let slow = head, fast = head;
>   while (fast.next && fast.next.next) { slow = slow.next; fast = fast.next.next; }
>   let prev = null, curr = slow.next;
>   slow.next = null;
>   while (curr) { const n = curr.next; curr.next = prev; prev = curr; curr = n; }
>   let a = head, b = prev;
>   while (b) {
>     const aN = a.next, bN = b.next;
>     a.next = b; b.next = aN;
>     a = aN; b = bN;
>   }
> };
> ```

> [!success]- Python
> ```python
> def reorder_list(head):
>     if not head or not head.next: return
>     slow = fast = head
>     while fast.next and fast.next.next:
>         slow = slow.next
>         fast = fast.next.next
>     prev, curr = None, slow.next
>     slow.next = None
>     while curr:
>         nxt = curr.next; curr.next = prev; prev = curr; curr = nxt
>     a, b = head, prev
>     while b:
>         a_n, b_n = a.next, b.next
>         a.next = b; b.next = a_n
>         a, b = a_n, b_n
> ```

**Key takeaway:** Compose 3 sub-skills (mid, reverse, merge). Each subroutine is reusable.

---

## P9: Reverse Nodes in k-Group

**LC #25** · **Hard**

Reverse every k consecutive nodes; leave remainder if < k.

### 🧠 Pattern: Reverse-Between with Anchor

> Use a dummy. For each group: find the group's last node; if not enough, stop. Reverse the group between anchor and group-end. Move anchor to (originally) first node of the group (now last).

> [!example]- 📊 Visual: reverse one k-group between anchors
> ```text
>   List with k=3:
> 
>     dummy → [1] → [2] → [3] → [4] → [5] → [6] → [7] → null
>       ↑     └──── group ────┘ └──── group ────┘  └ leftover
>     anchor
>     (groupPrev)
> 
>   ─ Step A: identify the group's last node (kth) ─
> 
>     dummy → [1] → [2] → [3] → [4] → [5] → ...
>       ↑                  ↑      ↑
>     anchor              kth    groupNext = kth.next
> 
>   ─ Step B: reverse only the segment [groupPrev.next .. kth] ─
> 
>     Reverse using 3-pointer dance bounded by groupNext on the right.
>     Before:  anchor → 1 → 2 → 3 → 4 ...
>     After :  anchor → 3 → 2 → 1 → 4 ...
> 
>       dummy → [3] → [2] → [1] → [4] → [5] → [6] → [7] → null
>         ↑                  ↑
>      old anchor       new anchor (tmp = old groupPrev.next)
> 
>   ─ Step C: slide anchor to the new tail of just-reversed group ─
> 
>     Now the next round starts here:
> 
>       dummy → 3 → 2 → 1 → [4] → [5] → [6] → [7] → null
>                        ↑    └────── next group ──┘
>                      anchor
> 
>   ─ Step D: if fewer than k nodes remain, STOP (leave as-is) ─
> 
>     Final (k=3, input 1..7):  3 → 2 → 1 → 6 → 5 → 4 → 7
>                                                     ↑ leftover untouched
> ```

> [!info]- 🔍 Dry Run: 1→2→3→4→5, k=2
> ```text
> dummy → 1 → 2 → 3 → 4 → 5
> 
> groupPrev = dummy
> 
> ─────────────────────────────────────────
> ROUND 1: identify group [1, 2]
>   walk k=2 steps from groupPrev → kth = node(2)
>   kth not null → can reverse
>   groupNext = kth.next = node(3)
> 
>   Reverse from groupPrev.next=node(1) up to (not including) groupNext=node(3):
>     prev=node(3), curr=node(1)
>     iter: nxt=node(2); curr.next=prev → 1→3; prev=1, curr=2
>     iter: nxt=node(3); curr.next=prev → 2→1; prev=2, curr=3 (==groupNext, stop)
>   After reverse: 2 → 1 → 3 → 4 → 5
> 
>   tmp = groupPrev.next = node(1)  (the original first; now last in group)
>   groupPrev.next = kth = node(2)  (link dummy to new head of group)
>   groupPrev = tmp = node(1)        (new anchor)
> 
>   State: dummy → 2 → 1 → 3 → 4 → 5
> 
> ─────────────────────────────────────────
> ROUND 2: from groupPrev=node(1), walk k=2 → kth=node(4)
>   groupNext = node(5)
>   Reverse [3, 4]:
>     prev=node(5), curr=node(3)
>     iter: nxt=4; 3.next=5; prev=3, curr=4
>     iter: nxt=5; 4.next=3; prev=4, curr=5 stop
>   After: 4 → 3 → 5
>   tmp = node(3); groupPrev.next = node(4); groupPrev = node(3)
>   State: dummy → 2 → 1 → 4 → 3 → 5
> 
> ─────────────────────────────────────────
> ROUND 3: walk k=2 from node(3) → 1 step to node(5), next is null at step 2 → kth=null
>   Not enough nodes → stop.
> 
> ✅ Answer: 2 → 1 → 4 → 3 → 5
> ```

> [!success]- JS
> ```js
> const reverseKGroup = (head, k) => {
>   const dummy = { next: head };
>   let groupPrev = dummy;
>   while (true) {
>     let kth = groupPrev;
>     for (let i = 0; i < k && kth; i++) kth = kth.next;
>     if (!kth) break;
>     const groupNext = kth.next;
>     let prev = groupNext, curr = groupPrev.next;
>     while (curr !== groupNext) {
>       const nxt = curr.next;
>       curr.next = prev;
>       prev = curr;
>       curr = nxt;
>     }
>     const tmp = groupPrev.next;
>     groupPrev.next = kth;
>     groupPrev = tmp;
>   }
>   return dummy.next;
> };
> ```

> [!success]- Python
> ```python
> def reverse_k_group(head, k):
>     dummy = ListNode(0, head)
>     group_prev = dummy
>     while True:
>         kth = group_prev
>         for _ in range(k):
>             kth = kth.next if kth else None
>             if not kth: return dummy.next
>         group_next = kth.next
>         prev, curr = group_next, group_prev.next
>         while curr is not group_next:
>             nxt = curr.next
>             curr.next = prev
>             prev = curr
>             curr = nxt
>         tmp = group_prev.next
>         group_prev.next = kth
>         group_prev = tmp
> ```

**Key takeaway:** Reverse a sub-segment between fixed anchors. The 3-pointer dance is the same, just bounded.

---

## P10: LRU Cache

**LC #146** · Medium · Design · O(1) get/put

### 🧠 Pattern: Hash Map + Doubly Linked List

> Doubly linked list orders recency (head = most recent). Hash map gives O(1) lookup of the node. Move-to-front on `get` and `put`. Evict tail on overflow.

> [!example]- 📊 Visual: hash + DLL structure
> ```text
>   ┌──────────────────────────────────────────────────────┐
>   │ map: { 1: ●─┐                                        │
>   │         3: ●─┼─┐                                     │
>   │         5: ●─┼─┼─┐                                   │
>   │       }     │ │ │                                    │
>   └─────────────┼─┼─┼────────────────────────────────────┘
>                 │ │ │
>      ┌─────┐    ▼ │ │   ┌─────┐         ┌─────┐    ┌─────┐
>      │HEAD │↔ [3,X]↔─── [1,Y] ↔──────── [5,Z] ↔──│TAIL │
>      └─────┘     ↑                                └─────┘
>                  │ ◀── most recent (just touched)
>                  
>      head ↔ (most recent) ↔ ... ↔ (least recent) ↔ tail
> 
>   Operations (all O(1)):
> 
>     get(k):                   put(k, v) new key:
>       node = map[k]              create node n
>       remove(node)               if size == cap:
>       addFront(node)               lru = tail.prev
>       return node.val              remove(lru)
>                                    delete map[lru.key]
>     put(k, v) existing:          addFront(n)
>       node = map[k]              map[k] = n
>       node.val = v
>       remove(node)
>       addFront(node)
> 
>   Sentinel head and tail nodes (dummies) avoid null-edge cases.
> ```

> [!info]- 🔍 Dry Run: cap=2; put(1,1), put(2,2), get(1), put(3,3), get(2)
> ```text
> Structure: doubly linked list  head ↔ ... ↔ tail
>   head and tail are sentinels (no data)
> 
> ─────────────────────────────────────────
> put(1, 1):
>   key 1 not in map → create node N1=(1,1)
>   addFront(N1): head ↔ N1 ↔ tail
>   map = {1: N1}
> 
> put(2, 2):
>   create N2=(2,2)
>   addFront: head ↔ N2 ↔ N1 ↔ tail
>   map = {1: N1, 2: N2}
> 
> get(1):
>   key 1 in map → N1
>   remove(N1), addFront(N1):
>     head ↔ N1 ↔ N2 ↔ tail
>   return N1.val = 1
> 
> put(3, 3):
>   key 3 not in map; size (2) == cap (2) → EVICT
>     lru = tail.prev = N2
>     remove(N2); delete map[2]
>     map = {1: N1}
>   create N3=(3,3); addFront(N3)
>     head ↔ N3 ↔ N1 ↔ tail
>   map = {1: N1, 3: N3}
> 
> get(2):
>   key 2 NOT in map → return -1
> 
> ✅ Final state:
>   list: head ↔ N3(3,3) ↔ N1(1,1) ↔ tail
>   N3 is most recently used; N1 is LRU.
> ```

> [!success]- JS
> ```js
> class LRUCache {
>   constructor(cap) {
>     this.cap = cap; this.map = new Map();
>     this.head = { key: 0, val: 0 };
>     this.tail = { key: 0, val: 0 };
>     this.head.next = this.tail; this.tail.prev = this.head;
>   }
>   _remove(n) { n.prev.next = n.next; n.next.prev = n.prev; }
>   _addFront(n) {
>     n.next = this.head.next; n.prev = this.head;
>     this.head.next.prev = n; this.head.next = n;
>   }
>   get(key) {
>     if (!this.map.has(key)) return -1;
>     const n = this.map.get(key);
>     this._remove(n); this._addFront(n);
>     return n.val;
>   }
>   put(key, val) {
>     if (this.map.has(key)) {
>       const n = this.map.get(key);
>       n.val = val;
>       this._remove(n); this._addFront(n);
>       return;
>     }
>     if (this.map.size === this.cap) {
>       const lru = this.tail.prev;
>       this._remove(lru); this.map.delete(lru.key);
>     }
>     const n = { key, val };
>     this._addFront(n); this.map.set(key, n);
>   }
> }
> ```

> [!success]- Python
> ```python
> from collections import OrderedDict
> class LRUCache:
>     def __init__(self, cap):
>         self.cap = cap
>         self.data = OrderedDict()
>     def get(self, key):
>         if key not in self.data: return -1
>         self.data.move_to_end(key)
>         return self.data[key]
>     def put(self, key, val):
>         if key in self.data:
>             self.data.move_to_end(key)
>         self.data[key] = val
>         if len(self.data) > self.cap:
>             self.data.popitem(last=False)
> ```

**Variants:** LFU Cache · LRU Cache with TTL.

**Key takeaway:** O(1) for both lookup AND ordered eviction → hash + doubly linked list. Sentinels remove edge cases.

---

## P11: Remove Linked List Elements

**LC #203** · Easy

Remove all nodes with `val == target`.

### Approach

Dummy + walk; skip matches.

> [!example]- 📊 Visual: dummy + skip-over filter (val=6)
> ```text
>   Input (remove all 6s):
> 
>     dummy → [1] → [2] → [6] → [3] → [4] → [5] → [6] → null
>       ↑           target ✗                       target ✗
>       p
> 
>   Walk with `p`; check p.next.val:
>     • If match → splice it out: p.next = p.next.next   (p STAYS)
>     • Else      → advance: p = p.next
> 
>   ─ Snapshot when p reaches node(2), p.next = [6] ─
> 
>     dummy → 1 → [2] → ╳[6]╳ → [3] → 4 → 5 → 6 → null
>                  ↑      │
>                  p      └──► p.next = p.next.next  (skip the 6)
> 
>     Result: dummy → 1 → 2 → 3 → 4 → 5 → 6 → null
>                          ↑
>                          p stays here — recheck new p.next
> 
>   ─ Snapshot when p reaches node(5), p.next = [6] ─
> 
>     ... → [5] → ╳[6]╳ → null   →   ... → [5] → null
>            ↑
>            p
> 
>   Why DUMMY matters:
>     If the head itself is the target (e.g. head=6→1→2),
>     the dummy lets us treat it uniformly — no special case.
> 
>   Final: dummy.next = 1 → 2 → 3 → 4 → 5 → null
> ```

> [!info]- 🔍 Dry Run: 1→2→6→3→4→5→6, val=6
> ```text
> dummy → 1 → 2 → 6 → 3 → 4 → 5 → 6
> p = dummy
> 
> ─────────────────────────────────────────
> p=dummy, p.next=1
>   1 == 6? NO → p = 1
> 
> p=1, p.next=2
>   2 == 6? NO → p = 2
> 
> p=2, p.next=6
>   6 == 6? YES → p.next = p.next.next  (skip the 6)
>   State: 1 → 2 → 3 → 4 → 5 → 6  (relative to dummy)
>   p stays at 2 (re-check new p.next)
> 
> p=2, p.next=3
>   3 == 6? NO → p = 3
> 
> p=3, p.next=4 → p=4
> p=4, p.next=5 → p=5
> p=5, p.next=6
>   YES → skip
>   State: ... → 5 → null
>   p stays at 5
> 
> p=5, p.next=null → exit
> 
> ✅ Answer: 1 → 2 → 3 → 4 → 5
> ```

> [!success]- JS
> ```js
> const removeElements = (head, val) => {
>   const dummy = { next: head };
>   let p = dummy;
>   while (p.next) {
>     if (p.next.val === val) p.next = p.next.next;
>     else p = p.next;
>   }
>   return dummy.next;
> };
> ```

> [!success]- Python
> ```python
> def remove_elements(head, val):
>     dummy = ListNode(0, head)
>     p = dummy
>     while p.next:
>         if p.next.val == val:
>             p.next = p.next.next
>         else:
>             p = p.next
>     return dummy.next
> ```

**Key takeaway:** "Remove all X" → dummy + walk + conditional next-skip.

---

## P12: Palindrome Linked List

**LC #234** · Easy · O(1) extra

### Approach

Find midpoint, reverse second half, compare.

> [!example]- 📊 Visual: fold the list and compare halves
> ```text
>   Input:    [1] → [2] → [2] → [1] → null
> 
>   ─ Step 1: find midpoint (fast/slow) ─
> 
>     head             slow
>      ↓                ↓
>     [1] → [2] → [2] → [1] → null
>                                ↑
>                              fast (or fast.next) is null
> 
>   ─ Step 2: reverse from slow onward ─
> 
>     Before:  head → 1 → 2          ;          slow → 2 → 1 → null
>     After :  head → 1 → 2 → null   ;          prev → 1 → 2 → null
> 
>   ─ Step 3: fold and compare ─
> 
>          a (forward)                  b (reversed second half)
>          ↓                            ↓
>         [1] → [2] → null            [1] → [2] → null
> 
>     Compare position by position:
> 
>         a.val:  1   2
>                 ║   ║      ← matches
>         b.val:  1   2
> 
>     All match while b is non-null → PALINDROME ✅
> 
>   Counter-example (1→2→3→4):
>     after split/reverse: a=1→2, b=4→3
>         1 vs 4 → MISMATCH → not a palindrome
> 
>   Odd-length lists (1→2→3→2→1):
>     slow lands on the MIDDLE node (3). The middle is symmetric to
>     itself and we just stop when b runs out — no special-casing.
> ```

> [!info]- 🔍 Dry Run: 1 → 2 → 2 → 1
> ```text
> Phase 1 — Midpoint:
>   slow=1, fast=1
>   slow=2, fast=2 (the third node, val 2)
>   slow=2 (third), fast=null (fourth.next=null but stops at top check)
>   Actually trace: fast=1; iter1: fast=fast.next.next=2(3rd); iter2: fast=2(3rd).next.next=null. Stop.
>   slow advanced twice: slow=2 (3rd node).
> 
> Phase 2 — Reverse from slow=node(3rd "2"):
>   prev=null, slow=node3=2
>   iter: nxt=node4=1; node3.next=null; prev=node3; slow=node4
>   iter: nxt=null; node4.next=node3 (=2); prev=node4 (=1); slow=null
>   prev now heads: 1 → 2 → null (reversed second half)
> 
> Phase 3 — Compare:
>   a = original head = node1=1
>   b = prev = node4=1
>   iter: a.val=1, b.val=1 → match; a=node2=2, b=node3=2
>   iter: a.val=2, b.val=2 → match; a=node3=2, b=null (we cut at slow earlier... actually we didn't cut, but b advances along reversed half)
>   b=null → exit
> 
> ✅ Answer: true
> ```

> [!success]- JS
> ```js
> const isPalindromeLL = (head) => {
>   let slow = head, fast = head;
>   while (fast && fast.next) { slow = slow.next; fast = fast.next.next; }
>   let prev = null;
>   while (slow) { const n = slow.next; slow.next = prev; prev = slow; slow = n; }
>   let a = head, b = prev;
>   while (b) {
>     if (a.val !== b.val) return false;
>     a = a.next; b = b.next;
>   }
>   return true;
> };
> ```

> [!success]- Python
> ```python
> def is_palindrome_ll(head):
>     slow = fast = head
>     while fast and fast.next:
>         slow = slow.next
>         fast = fast.next.next
>     prev = None
>     while slow:
>         nxt = slow.next
>         slow.next = prev
>         prev = slow
>         slow = nxt
>     a, b = head, prev
>     while b:
>         if a.val != b.val: return False
>         a, b = a.next, b.next
>     return True
> ```

**Key takeaway:** Composed solution: midpoint + reverse + compare.

---

## P13: Intersection of Two Linked Lists

**LC #160** · Easy · O(1) extra

### 🧠 Pattern: Length Difference Reset

> When pointer A finishes, jump to head of B. When B finishes, jump to head of A. They traverse `m+n` total; if intersection exists, they meet there. Else, both reach `null` simultaneously.

> [!example]- 📊 Visual: switch heads to equalize path length
> ```text
>   Two lists that merge into one shared tail:
> 
>     A:  [1] → [9] → [1] ──┐
>                            ▼
>                           [X] → [Y] → [Z] → null
>                            ▲
>     B:        [3] ────────┘
> 
>     lenA = 6,  lenB = 4,  shared tail length = 3
>     intersection node = X
> 
>   ─ Strategy: pointer p walks A then B; pointer q walks B then A ─
> 
>     Combined path each traverses: lenA + lenB = 10 steps total.
>     If an intersection exists, they sync up at X.
> 
>   ─ Step trace ─
> 
>     t=0:  p=A0(1)   q=B0(3)
>     t=1:  p=A1(9)   q=X
>     t=2:  p=A2(1)   q=Y
>     t=3:  p=X       q=Z
>     t=4:  p=Y       q=null → q jumps to A0(1)
>     t=5:  p=Z       q=A1(9)
>     t=6:  p=null → p jumps to B0(3);   q=A2(1)
>     t=7:  p=X       q=X         ← MATCH ✅
> 
>   Why it works (visual proof):
> 
>     p walks:  [A: 1,9,1,X,Y,Z]  then  [B: 3,X,Y,Z]  → meets at X
>     q walks:  [B: 3,X,Y,Z]      then  [A: 1,9,1,X,Y,Z] → meets at X
> 
>     Both paths have length lenA + lenB. The trailing common
>     segment lines up — they land on the SAME node simultaneously.
> 
>   No intersection? Both pointers become null at the same step → return null.
> ```

> [!info]- 🔍 Dry Run: A=[1,9,1,X,Y,Z], B=[3,X,Y,Z] (X,Y,Z are shared)
> ```text
> Lengths: A=6 nodes (1,9,1,X,Y,Z), B=4 nodes (3,X,Y,Z). Intersection at X.
> 
> p=A0(1), q=B0(3)
> 
> ─────────────────────────────────────────
> step 1: p=9, q=X        (not equal: 9 vs X)
> step 2: p=1, q=Y
> step 3: p=X, q=Z
> step 4: p=Y, q=null      → q jumps to head A → q=1(A0)
> step 5: p=Z, q=9
> step 6: p=null → jump to head B → p=3, q=1
> step 7: p=X, q=X         ← MATCH!
> 
> ✅ Answer: X (the intersection node)
> ```

> [!success]- JS
> ```js
> const getIntersectionNode = (a, b) => {
>   let p = a, q = b;
>   while (p !== q) {
>     p = p ? p.next : b;
>     q = q ? q.next : a;
>   }
>   return p;
> };
> ```

> [!success]- Python
> ```python
> def get_intersection_node(a, b):
>     p, q = a, b
>     while p is not q:
>         p = p.next if p else b
>         q = q.next if q else a
>     return p
> ```

**Key takeaway:** Beautiful trick — equalize traversal lengths by switching heads. No counting needed.

---

## P14: Merge k Sorted Lists

**LC #23** · **Hard**

### 🧠 Pattern: Min-Heap of Heads

> Push all k heads into a min-heap keyed on val. Pop the smallest, append, push its next. O(N log k).

### Approach Evolution

1. **Merge two at a time, k-1 times** · O(Nk).
2. **Divide & conquer pairs** · O(N log k).
3. **Min-heap — FINAL** · O(N log k)/O(k).

> [!example]- 📊 Visual: k lists feeding a min-heap
> ```text
>   k sorted input lists (drawn vertically by their heads):
> 
>     list 0:  [ 1 ] → [ 4 ] → [ 5 ] → null
>     list 1:  [ 1 ] → [ 3 ] → [ 4 ] → null
>     list 2:  [ 2 ] → [ 6 ] → null
> 
>   ─ Seed: push EACH list's head into a min-heap ─
> 
>                ┌─────────────────────────────┐
>                │  min-heap (key = node.val)  │
>                │                             │
>                │            (1, list0)       │  ← root = smallest
>                │           /        \        │
>                │     (1, list1)  (2, list2)  │
>                └─────────────────────────────┘
> 
>   ─ Repeat: pop min, append to result, push popped.next ─
> 
>     pop (1,list0) → result: dummy → 1
>     push list0.next = 4
>     heap = { (1,list1), (2,list2), (4,list0) }
> 
>     pop (1,list1) → result: dummy → 1 → 1
>     push list1.next = 3
>     heap = { (2,list2), (4,list0), (3,list1) }
> 
>     pop (2,list2) → result: ... → 2 ;  push 6
>     pop (3,list1) → result: ... → 3 ;  push 4
>     pop (4,list0) → result: ... → 4 ;  push 5
>     pop (4,list1) → result: ... → 4 ;  (list1 exhausted)
>     pop (5,list0) → result: ... → 5 ;  (list0 exhausted)
>     pop (6,list2) → result: ... → 6 ;  (list2 exhausted)
> 
>   ─ Final ─
> 
>     dummy → 1 → 1 → 2 → 3 → 4 → 4 → 5 → 6 → null
> 
>   Complexity:
>     • Each of N total nodes is pushed/popped once.
>     • Heap holds ≤ k entries → push/pop is O(log k).
>     • Total: O(N log k) time, O(k) space.
> 
>   Tie-breaker tip (Python): heap entries are (val, idx, node).
>   The `idx` prevents Python from comparing ListNode objects on val ties.
> ```

> [!info]- 🔍 Dry Run: lists=[[1,4,5],[1,3,4],[2,6]]
> ```text
> Setup:
>   Push each list's head into min-heap (tuples: (val, list_idx, node)):
>     heap = [(1, 0, n1a), (1, 1, n1b), (2, 2, n2c)]
> 
> ─────────────────────────────────────────
> Iter 1: pop (1, 0, n1a)
>   append to result: dummy → 1(a)
>   push next from list 0: n1a.next = 4 → push (4, 0, n4a)
>   heap = [(1,1,n1b), (2,2,n2c), (4,0,n4a)]
> 
> Iter 2: pop (1, 1, n1b)
>   append: dummy → 1 → 1(b)
>   push n1b.next = 3 → (3, 1, n3b)
>   heap = [(2,2,n2c), (4,0,n4a), (3,1,n3b)]
> 
> Iter 3: pop (2, 2, n2c)
>   append: ...→ 1 → 1 → 2
>   push 6 → (6, 2, ...)
>   heap = [(3,1,...), (4,0,...), (6,2,...)]
> 
> Iter 4: pop (3, 1, ...)
>   append → 3
>   push 4 → (4, 1, ...)
>   heap = [(4,0,...), (4,1,...), (6,2,...)]
> 
> Iter 5: pop (4, 0, ...) [or (4,1) depending on tiebreaker]
>   append → 4
>   push 5 → (5, 0, ...)
>   heap = [(4,1,...), (6,2,...), (5,0,...)]
> 
> Iter 6: pop (4, 1, ...)
>   append → 4
>   nothing left in list 1, don't push
>   heap = [(5,0,...), (6,2,...)]
> 
> Iter 7: pop (5, 0, ...)
>   append → 5
>   no more in list 0
>   heap = [(6,2,...)]
> 
> Iter 8: pop (6, 2, ...)
>   append → 6
>   no more
>   heap = []
> 
> ✅ Answer: 1 → 1 → 2 → 3 → 4 → 4 → 5 → 6
> ```

> [!success]- JS
> ```js
> // assumes a MinHeap available
> const mergeKLists = (lists) => {
>   const heap = new MinHeap((a, b) => a.val - b.val);
>   for (const node of lists) if (node) heap.push(node);
>   const dummy = { next: null };
>   let tail = dummy;
>   while (heap.size()) {
>     const n = heap.pop();
>     tail.next = n; tail = n;
>     if (n.next) heap.push(n.next);
>   }
>   return dummy.next;
> };
> ```

> [!success]- Python
> ```python
> import heapq
> def merge_k_lists(lists):
>     heap = []
>     for i, node in enumerate(lists):
>         if node:
>             heapq.heappush(heap, (node.val, i, node))
>     dummy = ListNode(0)
>     tail = dummy
>     while heap:
>         v, i, n = heapq.heappop(heap)
>         tail.next = n; tail = n
>         if n.next:
>             heapq.heappush(heap, (n.next.val, i, n.next))
>     return dummy.next
> ```

> [!tip] Tie-breaker `i`
> Heaps compare full tuples. If values tie, Python tries to compare nodes (errors). Add `i` as the tiebreaker.

**Key takeaway:** k-way merge → min-heap. Always seed with k initial heads.

---

> [!tip] After this drill
> The 5-line reverse is muscle memory. Dummy node is your default. Fast/slow is your Swiss army knife.
