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
> 
> ```
> # In-place reverse skeleton:
> prev, curr = None, head
> while curr:
>     nxt = curr.next
>     curr.next = prev
>     prev = curr
>     curr = nxt
> return prev
> ```

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
| P9 | Reverse Nodes in k-Group | 25 | Hard | Reverse with anchor |
| P10 | LRU Cache | 146 | Med | Hash + doubly linked list |
| P11 | Remove Linked List Elements | 203 | Easy | Dummy filter |
| P12 | Palindrome Linked List | 234 | Easy | Mid + reverse + compare |
| P13 | Intersection of Two Linked Lists | 160 | Easy | Length difference |
| P14 | Merge k Sorted Lists | 23 | Hard | Min-heap |

---

## P1: Reverse Linked List

**LC #206** · Easy

Reverse a singly linked list.

**Edge cases:** empty · single node · two nodes.

### 🧠 Pattern: Three-Pointer Reverse

> `prev`, `curr`, `next`. Save `next`, point `curr.next ← prev`, advance both. Return `prev` (new head).

### Trace

```
1 → 2 → 3 → null
prev=null curr=1
  nxt=2  1.next=null  prev=1  curr=2     null←1   2→3→null
  nxt=3  2.next=1     prev=2  curr=3     null←1←2 3→null
  nxt=null 3.next=2   prev=3  curr=null  null←1←2←3
return 3 (new head)
```

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

(See [[02 - Two Pointers]] P11 for the full discussion of Floyd's tortoise & hare.)

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

### Trace

```
1→2→3→4→5→3(back to)
slow & fast meet at 5 (say)
reset slow=1
step both:
  slow=2 fast=3
  slow=3 fast=3 ✓ → cycle start = 3
```

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

**LC #25** · Hard

Reverse every k consecutive nodes; leave remainder if < k.

### 🧠 Pattern: Reverse-Between with Anchor

> Use a dummy. For each group: find the group's last node; if not enough, stop. Reverse the group between anchor and group-end. Move anchor to (originally) first node of the group (now last).

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

> [!success]- JS
> ```js
> class LRUCache {
>   constructor(cap) {
>     this.cap = cap; this.map = new Map();
>     this.head = { key: 0, val: 0 };  // sentinels
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

> [!tip] Production note
> Python's `OrderedDict` IS hash+DLL under the hood. In an interview, ask before using it — they may want you to implement the DLL.

**Variants:** LFU Cache · LRU Cache with TTL.

**Key takeaway:** O(1) for both lookup AND ordered eviction → hash + doubly linked list. Sentinels remove edge cases.

---

## P11: Remove Linked List Elements

**LC #203** · Easy

Remove all nodes with `val == target`.

### Approach

Dummy + walk; skip matches.

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

**LC #23** · Hard

### 🧠 Pattern: Min-Heap of Heads

> Push all k heads into a min-heap keyed on val. Pop the smallest, append, push its next. O(N log k).

### Approach Evolution

1. **Merge two at a time, k-1 times** · O(Nk).
2. **Divide & conquer pairs** · O(N log k).
3. **Min-heap — FINAL** · O(N log k)/O(k).

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
