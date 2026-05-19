---
title: "Workflow 05: Onsite Round — System-Adjacent (LRU, Trie, etc.)"
date: 2026-05-19
tags:
  - leetcode
  - workflow
  - mock-interview
  - design
parent: "[[LeetCode Study]]"
status: in-progress
---

# Workflow 05: Onsite Round — System-Adjacent

> [!abstract] Format
> **45 minutes** · **1 design-flavored coding problem** · the round between pure algorithms and full system design. You'll be asked to design a small but real component — pick data structures wisely, write clean code, discuss tradeoffs.

> [!tip] What "system-adjacent" means
> It's NOT system design (no boxes-and-arrows, no load balancers). It's a CODING problem with **strong constraints**: time complexity targets, memory limits, API design, multi-method classes. Examples: LRU Cache, Trie, Tweet feed, Iterator design.

---

## Problem: LRU Cache

**LC #146** · Medium-Hard

> [!example] Prompt
> "Implement an LRU (Least Recently Used) cache with O(1) average for both `get(key)` and `put(key, value)`. Evict the LRU entry when over capacity."

### 🎬 Hidden Script

> [!example]- What they're testing
> - Identify the two requirements: O(1) lookup AND O(1) ordered eviction
> - Recognize the data structure: **hash map + doubly linked list**
> - Implement cleanly with sentinels (head/tail nodes) to avoid edge cases
> - Discuss thread safety / concurrency as follow-up
> 
> **Red flag:** using `OrderedDict` without explaining the underlying structure. Or only one data structure → won't be O(1) for both.

---

## Walkthrough

**Clarify (1 min):**
- "Capacity > 0?"
- "`get` on missing key → return -1?"
- "Should `put` on existing key update value AND mark as most recent?"
- "Thread-safe required?"

**Design Discussion (3 min):**

> "I need O(1) for both `get` and `put`. Let me think about what each requires:
> 
> - **Lookup by key** → hash map gives O(1).
> - **Insertion at 'most recent' position** → need to add to one end.
> - **Eviction of 'least recent'** → need to remove from the other end.
> - **Move an existing entry to most-recent** → need to remove from middle, add to end.
> 
> Doubly linked list gives O(1) remove given a node reference (it has `prev` pointers). Hash map will store key → node. Combining the two gives:
> 
> ```
> get(k):  map[k] = node  →  remove(node)  →  addFront(node)  →  return node.val
> put(k,v): if existing: remove + addFront. If new: addFront + map; evict tail if over capacity.
> ```

**Architecture Sketch (1 min):**

```
       head ←→ A ←→ B ←→ C ←→ tail
                       ↑
        map: 'a'→A, 'b'→B, 'c'→C
                       ↓
                   most recent
```

- `head` and `tail` are sentinel nodes (dummy, never store data)
- New nodes inserted between `head` and `head.next` (front = most recent)
- Eviction = remove `tail.prev` (the LRU node)

**Code (10 min):**

> [!success]- Python — full DIY implementation
> ```python
> class Node:
>     __slots__ = ('key', 'val', 'prev', 'next')
>     def __init__(self, key=0, val=0):
>         self.key = key
>         self.val = val
>         self.prev = None
>         self.next = None
> 
> class LRUCache:
>     def __init__(self, capacity):
>         self.cap = capacity
>         self.map = {}
>         # sentinels
>         self.head = Node()
>         self.tail = Node()
>         self.head.next = self.tail
>         self.tail.prev = self.head
>     
>     def _remove(self, node):
>         node.prev.next = node.next
>         node.next.prev = node.prev
>     
>     def _add_front(self, node):
>         node.next = self.head.next
>         node.prev = self.head
>         self.head.next.prev = node
>         self.head.next = node
>     
>     def get(self, key):
>         if key not in self.map: return -1
>         node = self.map[key]
>         self._remove(node)
>         self._add_front(node)
>         return node.val
>     
>     def put(self, key, value):
>         if key in self.map:
>             node = self.map[key]
>             node.val = value
>             self._remove(node)
>             self._add_front(node)
>             return
>         if len(self.map) == self.cap:
>             lru = self.tail.prev
>             self._remove(lru)
>             del self.map[lru.key]
>         node = Node(key, value)
>         self._add_front(node)
>         self.map[key] = node
> ```

> [!success]- Python — using `OrderedDict` (mention this as the production shortcut)
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

> [!success]- JS — DIY
> ```js
> class LRUCache {
>   constructor(capacity) {
>     this.cap = capacity;
>     this.map = new Map();
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

**Trace (2 min):**
```
cap=2
put(1,1)  → map={1:N1}, list: H ↔ N1 ↔ T
put(2,2)  → map={1:N1, 2:N2}, list: H ↔ N2 ↔ N1 ↔ T
get(1)    → move N1 front; list: H ↔ N1 ↔ N2 ↔ T; return 1
put(3,3)  → over capacity; evict tail.prev=N2; list: H ↔ N3 ↔ N1 ↔ T
get(2)    → not in map → -1
```

**Complexity:** O(1) for both ops. O(capacity) space.

**Edge cases:**
- `cap = 1` — sentinels handle this correctly (only 1 node ever between head and tail)
- Repeated `put` of same key — must NOT count as new entry; just update + move

---

## Follow-ups (10 min — discuss, don't necessarily code)

### Q1: Thread safety

> "If multiple threads call `get` and `put` concurrently, the linked list could corrupt — pointer updates in `_remove` and `_addFront` are not atomic."
> 
> **Options:**
> - **Lock the whole cache** — simple, but kills concurrency.
> - **Striped locking** — N locks, hash key to a lock. Better, but list-level operations still need a global lock.
> - **Lock-free** — extremely hard for doubly linked lists. Use a CLHM (ConcurrentLinkedHashMap) like Caffeine's design — ring buffer of access events, batched async reordering.

### Q2: Approximate LRU (when exact ordering is too slow)

> "For very high QPS, even O(1) work per access is too much. Use **CLOCK** algorithm: ring buffer of entries with a `referenced` bit. On access, set bit. On eviction, scan and clear bits until finding an entry with bit=0."

### Q3: Persistence

> "If the cache must survive process restart, periodically snapshot the linked list order + values to disk. On restart, rebuild. Append-only log for incremental writes."

### Q4: TTL (Time-to-Live)

> "Add `expires_at` to each node. On `get`, if expired → treat as miss + remove. Also add a separate min-heap of expirations for background cleanup. Combined structure now has hash + DLL + heap."

### Q5: LFU instead of LRU

> "Different! For LFU, we evict the **least frequently** used. Need: hash key→node, hash freq→DLL of nodes at that freq, plus minFreq pointer. More complex; O(1) achievable but trickier code."

---

## Self-Eval Checklist

- [ ] Did I justify the data structure choice from requirements?
- [ ] Did I draw the layout (map + DLL with sentinels) before coding?
- [ ] Did I implement `_remove` and `_addFront` as helpers (not inline)?
- [ ] Did I trace the eviction case?
- [ ] Did I discuss at least one follow-up (thread safety / TTL / LFU)?

> [!tip] System-adjacent rounds reward "engineer instincts"
> Talk like a senior engineer: "I'd use X because of constraint Y; the tradeoff is Z." This round rewards holistic thinking over just code correctness.

---

## Other System-Adjacent Problems

- **LC #208 Implement Trie** — words/prefixes design
- **LC #211 Add and Search Words** — trie + wildcard
- **LC #355 Design Twitter** — heap + hashmap
- **LC #295 Find Median from Data Stream** — two heaps design
- **LC #1797 Authentication Manager** — TTL hash map
- **LC #460 LFU Cache** — harder LRU variant
- **LC #341 Flatten Nested List Iterator** — iterator design with stack
- **LC #380 Insert Delete GetRandom O(1)** — hash + array design
