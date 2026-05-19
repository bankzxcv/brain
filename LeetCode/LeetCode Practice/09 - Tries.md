---
title: "LeetCode Practice: Tries"
date: 2026-05-19
tags:
  - leetcode
  - practice
  - trie
  - prefix-tree
parent: "[[LeetCode Study]]"
status: in-progress
---

# LeetCode Practice: Tries

8 problems · prefix-indexed tree for dictionary / autocomplete / XOR problems.

> [!abstract] Pattern recap
> A **trie** is a tree where each edge is a character. Each path root→node spells a prefix. Mark `isEnd` at nodes that complete a word.
> 
> **Operations** (k = word length):
> - `insert(word)` → O(k)
> - `search(word)` → O(k)
> - `startsWith(prefix)` → O(k)
> 
> **Space** — bounded by total characters across all words.

> [!tip] Two implementations
> - **Map of children**: `{char: node}` — flexible, handles any alphabet
> - **26-array of children** — faster for lowercase a-z

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Implement Trie | 208 | Med | Insert / search / prefix |
| P2 | Add and Search Words | 211 | Med | `.` wildcard via DFS |
| P3 | Word Search II | 212 | Hard | Trie + DFS pruning |
| P4 | Replace Words | 648 | Med | Shortest prefix match |
| P5 | Longest Word in Dictionary | 720 | Med | DFS over trie |
| P6 | Maximum XOR of Two Numbers | 421 | Med | Binary trie (bit-indexed) |
| P7 | Stream of Characters | 1032 | Hard | Reverse trie |
| P8 | Map Sum Pairs | 677 | Med | Trie + subtree sum |

---

## P1: Implement Trie (Prefix Tree)

**LC #208** · Medium

Implement `insert`, `search`, `startsWith`.

### Approach

Node = `{children: Map, isEnd: bool}`. Walk one char at a time, creating nodes as needed.

> [!success]- JS
> ```js
> class Trie {
>   constructor() { this.root = { children: new Map(), isEnd: false }; }
>   insert(word) {
>     let n = this.root;
>     for (const c of word) {
>       if (!n.children.has(c)) n.children.set(c, { children: new Map(), isEnd: false });
>       n = n.children.get(c);
>     }
>     n.isEnd = true;
>   }
>   _walk(s) {
>     let n = this.root;
>     for (const c of s) {
>       if (!n.children.has(c)) return null;
>       n = n.children.get(c);
>     }
>     return n;
>   }
>   search(word) { const n = this._walk(word); return !!n && n.isEnd; }
>   startsWith(prefix) { return !!this._walk(prefix); }
> }
> ```

> [!success]- Python
> ```python
> class Trie:
>     def __init__(self):
>         self.root = {}
>     def insert(self, word):
>         n = self.root
>         for c in word:
>             n = n.setdefault(c, {})
>         n['$'] = True   # end marker
>     def _walk(self, s):
>         n = self.root
>         for c in s:
>             if c not in n: return None
>             n = n[c]
>         return n
>     def search(self, word):
>         n = self._walk(word)
>         return n is not None and n.get('$', False)
>     def startsWith(self, prefix):
>         return self._walk(prefix) is not None
> ```

**Key takeaway:** Trie = dict-of-dicts. `isEnd` flag distinguishes "word ends here" from "prefix only".

---

## P2: Design Add and Search Words Data Structure

**LC #211** · Medium

`addWord(w)` and `search(w)` where `w` may contain `.` (matches any char).

### 🧠 Pattern: Trie + DFS on Wildcards

> On `.`, try all children — recurse.

> [!success]- JS
> ```js
> class WordDictionary {
>   constructor() { this.root = { children: new Map(), isEnd: false }; }
>   addWord(word) {
>     let n = this.root;
>     for (const c of word) {
>       if (!n.children.has(c)) n.children.set(c, { children: new Map(), isEnd: false });
>       n = n.children.get(c);
>     }
>     n.isEnd = true;
>   }
>   search(word) {
>     const dfs = (n, i) => {
>       if (i === word.length) return n.isEnd;
>       const c = word[i];
>       if (c === '.') {
>         for (const child of n.children.values()) if (dfs(child, i + 1)) return true;
>         return false;
>       }
>       if (!n.children.has(c)) return false;
>       return dfs(n.children.get(c), i + 1);
>     };
>     return dfs(this.root, 0);
>   }
> }
> ```

> [!success]- Python
> ```python
> class WordDictionary:
>     def __init__(self):
>         self.root = {}
>     def addWord(self, word):
>         n = self.root
>         for c in word: n = n.setdefault(c, {})
>         n['$'] = True
>     def search(self, word):
>         def dfs(n, i):
>             if i == len(word): return n.get('$', False)
>             c = word[i]
>             if c == '.':
>                 return any(dfs(child, i + 1) for k, child in n.items() if k != '$')
>             if c not in n: return False
>             return dfs(n[c], i + 1)
>         return dfs(self.root, 0)
> ```

**Key takeaway:** Wildcards in trie search → DFS branch on all children.

---

## P3: Word Search II

**LC #212** · Hard · Trie + DFS over grid

Given an `m×n` board and a list of words, return words present on the board.

### 🧠 Pattern: Build Trie of Words, DFS Board Through Trie

> Naive: run Word Search I (DFS) for each word → O(words × m·n × 4^k). Too slow.
> 
> **Better:** Insert all words into a trie. Walk every starting cell; descend trie + grid together, pruning when no child matches.

> [!success]- Python
> ```python
> def find_words(board, words):
>     root = {}
>     for w in words:
>         n = root
>         for c in w: n = n.setdefault(c, {})
>         n['$'] = w
>     R, C = len(board), len(board[0])
>     out = []
>     def dfs(r, c, n):
>         ch = board[r][c]
>         if ch not in n: return
>         nxt = n[ch]
>         if '$' in nxt:
>             out.append(nxt['$'])
>             del nxt['$']  # avoid duplicates
>         board[r][c] = '#'
>         for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
>             nr, nc = r + dr, c + dc
>             if 0 <= nr < R and 0 <= nc < C and board[nr][nc] != '#':
>                 dfs(nr, nc, nxt)
>         board[r][c] = ch
>         if not nxt: del n[ch]   # prune dead branches
>     for r in range(R):
>         for c in range(C):
>             dfs(r, c, root)
>     return out
> ```

> [!tip] Trie pruning makes this fast
> When a leaf is fully consumed, delete it from its parent — future searches skip it.

**Key takeaway:** Many-pattern search on a grid → trie + DFS + pruning. The trie tells you which paths are still viable.

---

## P4: Replace Words

**LC #648** · Medium

Replace each word in a sentence with its **shortest root** from a dictionary.

### Approach

Build trie of roots. For each word, walk trie; on `isEnd`, output that prefix.

> [!success]- Python
> ```python
> def replace_words(dictionary, sentence):
>     root = {}
>     for w in dictionary:
>         n = root
>         for c in w: n = n.setdefault(c, {})
>         n['$'] = True
>     def shortest(word):
>         n = root
>         for i, c in enumerate(word):
>             if c not in n: return word
>             n = n[c]
>             if n.get('$'): return word[:i + 1]
>         return word
>     return ' '.join(shortest(w) for w in sentence.split())
> ```

**Key takeaway:** "Shortest matching prefix" → walk trie, stop at first `isEnd`.

---

## P5: Longest Word in Dictionary

**LC #720** · Medium

Longest word that can be built **one char at a time**, each prefix in dict. Ties: lex-smallest.

### Approach

Trie of words. DFS preferring lex order; only descend if `isEnd`. Track longest path.

> [!success]- Python
> ```python
> def longest_word(words):
>     root = {}
>     for w in words:
>         n = root
>         for c in w: n = n.setdefault(c, {})
>         n['$'] = w
>     best = ""
>     def dfs(n, path):
>         nonlocal best
>         if '$' in n:
>             if len(path) > len(best) or (len(path) == len(best) and path < best):
>                 best = path
>         for c in sorted(k for k in n if k != '$'):
>             child = n[c]
>             if '$' in child:
>                 dfs(child, path + c)
>     dfs(root, "")
>     return best
> ```

**Key takeaway:** "Build incrementally, each step must be a valid word" → DFS over trie, only descend through `isEnd` nodes.

---

## P6: Maximum XOR of Two Numbers in an Array

**LC #421** · Medium

Max `a XOR b` over all pairs `a, b ∈ nums`.

### 🧠 Pattern: Binary Trie + Greedy Bit Matching

> Insert each number as a 32-bit binary string into a trie (root → MSB → LSB). For each number, traverse trie greedily preferring the **opposite** bit at each level (maximizes XOR). The path gives the best pair.

> [!success]- Python
> ```python
> def find_maximum_xor(nums):
>     root = {}
>     for x in nums:
>         n = root
>         for i in range(31, -1, -1):
>             b = (x >> i) & 1
>             n = n.setdefault(b, {})
>     best = 0
>     for x in nums:
>         n, cur = root, 0
>         for i in range(31, -1, -1):
>             b = (x >> i) & 1
>             toggle = 1 - b
>             if toggle in n:
>                 cur |= (1 << i)
>                 n = n[toggle]
>             else:
>                 n = n[b]
>         best = max(best, cur)
>     return best
> ```

**Key takeaway:** Binary trie is a powerful trick for bitwise problems. Greedy → flip every bit if possible.

---

## P7: Stream of Characters

**LC #1032** · Hard

`query(c)` returns true if recent suffix matches any word in dictionary.

### 🧠 Pattern: Reverse Trie + Recent Buffer

> Insert each word **reversed** into a trie. For `query`, walk recent chars **backwards** through the trie; if any prefix (= original suffix) hits `isEnd`, return true.

> [!success]- Python
> ```python
> class StreamChecker:
>     def __init__(self, words):
>         self.root = {}
>         for w in words:
>             n = self.root
>             for c in reversed(w):
>                 n = n.setdefault(c, {})
>             n['$'] = True
>         self.buf = []
>     def query(self, c):
>         self.buf.append(c)
>         n = self.root
>         for ch in reversed(self.buf):
>             if ch not in n: return False
>             n = n[ch]
>             if n.get('$'): return True
>         return False
> ```

> [!tip] Memory note
> Cap `buf` at max word length to bound space.

**Key takeaway:** Suffix matching on a stream → reverse trie + backward walk on recent buffer.

---

## P8: Map Sum Pairs

**LC #677** · Medium

`insert(key, val)` and `sum(prefix)` = sum of values whose key starts with prefix.

### Approach

Trie node stores `total` (sum of all subtree values). On insert, walk and **add delta** = newVal − oldVal at every node on the path.

> [!success]- Python
> ```python
> class MapSum:
>     def __init__(self):
>         self.root = {'_total': 0}
>         self.vals = {}
>     def insert(self, key, val):
>         delta = val - self.vals.get(key, 0)
>         self.vals[key] = val
>         n = self.root
>         n['_total'] += delta
>         for c in key:
>             n = n.setdefault(c, {'_total': 0})
>             n['_total'] += delta
>     def sum(self, prefix):
>         n = self.root
>         for c in prefix:
>             if c not in n: return 0
>             n = n[c]
>         return n['_total']
> ```

**Key takeaway:** Trie can carry **aggregates** at each node. Update with deltas to allow value mutation.

---

> [!tip] After this drill
> "Prefix lookup / dictionary search / autocomplete" → trie. "XOR max" → binary trie. "Suffix matching" → reverse trie. Tries trade O(N·L) space for O(L) queries — almost always worth it for many-query workloads.
