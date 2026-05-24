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

> [!tip] Two implementations
> - **Map of children**: `{char: node}` — flexible, handles any alphabet
> - **26-array of children** — faster for lowercase a-z

## Index

| # | Problem | LC# | Diff | Sub-pattern |
|---|---|---|---|---|
| P1 | Implement Trie | 208 | Med | Insert / search / prefix |
| P2 | Add and Search Words | 211 | Med | `.` wildcard via DFS |
| P3 | Word Search II | 212 | **Hard** | Trie + DFS pruning |
| P4 | Replace Words | 648 | Med | Shortest prefix match |
| P5 | Longest Word in Dictionary | 720 | Med | DFS over trie |
| P6 | Maximum XOR of Two Numbers | 421 | Med | Binary trie (bit-indexed) |
| P7 | Stream of Characters | 1032 | **Hard** | Reverse trie |
| P8 | Map Sum Pairs | 677 | Med | Trie + subtree sum |

---

## P1: Implement Trie (Prefix Tree)

**LC #208** · Medium

Implement `insert`, `search`, `startsWith`.

### Approach

Node = `{children: Map, isEnd: bool}`. Walk one char at a time, creating nodes as needed.

> [!example]- 📊 Visual: trie structure
> ```text
>   Inserted words: "apple", "app", "ape"
> 
>             root
>              │
>              a
>              │
>              p
>             / \
>            p   e($)         "ape" ends here
>           / \
>          l   ($)            "app" ends here
>          │
>          e($)               "apple" ends here
> 
>   '$' marker = word ends at this node.
>   Each edge is one character; each path root→node spells a prefix.
>   search(w) walks chars; success requires final node to carry '$'.
>   startsWith(p) walks chars; success means the walk didn't fail.
> ```

> [!info]- 🔍 Dry Run: insert("apple"), search("apple"), search("app"), startsWith("app"), insert("app"), search("app")
> ```text
> Setup:
>   root = {}    (each node is a dict; '$' key signals "word ends here")
> 
> ─────────────────────────────────────────
> insert("apple"):
>   walk 'a': not in root → create root['a'] = {}
>   walk 'p': not in root['a'] → create
>   walk 'p': create
>   walk 'l': create
>   walk 'e': create
>   mark end: leaf['$'] = True
>   Tree (conceptual):
>     root → a → p → p → l → e($)
> 
> ─────────────────────────────────────────
> search("apple"):
>   walk a→p→p→l→e — all exist
>   final node has '$'? YES → return true
> 
> ─────────────────────────────────────────
> search("app"):
>   walk a→p→p — all exist
>   final node has '$'? NO (it's just a prefix) → return false
> 
> ─────────────────────────────────────────
> startsWith("app"):
>   walk a→p→p — all exist
>   return true (don't care about $)
> 
> ─────────────────────────────────────────
> insert("app"):
>   walk a→p→p (all exist)
>   mark second-p's '$' = True
> 
> ─────────────────────────────────────────
> search("app") again:
>   walk to second p; node['$']=true → return true
> 
> ✅ All operations correct.
> ```

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

> [!example]- 📊 Visual: wildcard DFS branches every child
> ```text
>   After addWord("bad"), ("dad"), ("mad"):
> 
>            root
>           / | \
>          b  d  m
>          │  │  │
>          a  a  a
>          │  │  │
>          d($) d($) d($)
> 
>   search(".ad"):
>                 dfs(root, i=0)
>                 c='.' → try ALL children
>                /        |        \
>            try 'b'   try 'd'   try 'm'
>             ↓          ↓          ↓
>           dfs(b,1)   dfs(d,1)   dfs(m,1)
>           c='a' ✓    c='a' ✓    c='a' ✓
>             ↓          ↓          ↓
>           dfs(_a,2)  dfs(_a,2)  dfs(_a,2)
>           c='d'+$    c='d'+$    c='d'+$
>             TRUE ─── short-circuit
> 
>   '.' = wildcard frontier; literal char = single branch only.
> ```

> [!info]- 🔍 Dry Run: addWord("bad"), addWord("dad"), addWord("mad"), search("pad"), search(".ad"), search("b..")
> ```text
> After adds:
>   root
>    ├ b → a → d($)
>    ├ d → a → d($)
>    └ m → a → d($)
> 
> ─────────────────────────────────────────
> search("pad"):
>   dfs(root, i=0): c='p'. root has 'p'? NO → return false
> 
> ─────────────────────────────────────────
> search(".ad"):
>   dfs(root, i=0): c='.' → try every child
>     try 'b': dfs(b_node, i=1)
>       c='a'. b_node has 'a' ✓ → dfs(a_node, i=2)
>         c='d'. a_node has 'd' ✓ → dfs(d_node, i=3)
>           i==len → check isEnd. d_node has '$' ✓ → return true!
>     short-circuit: return true
> 
> ─────────────────────────────────────────
> search("b.."):
>   dfs(root, 0): c='b'. exists → dfs(b_node, 1)
>     c='.': try all children of b_node
>       try 'a': dfs(a_node, 2)
>         c='.': try all children of a_node
>           try 'd': dfs(d_node, 3)
>             i==len → d_node has '$' ✓ → true
>           return true
>         return true
>       return true
>     return true
> 
> ✅ "pad"→false, ".ad"→true, "b.."→true
> ```

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

**LC #212** · **Hard** · Trie + DFS over grid

Given an `m×n` board and a list of words, return words present on the board.

### 🧠 Pattern: Build Trie of Words, DFS Board Through Trie

> Naive: run Word Search I (DFS) for each word → O(words × m·n × 4^k). Too slow.
> 
> **Better:** Insert all words into a trie. Walk every starting cell; descend trie + grid together, pruning when no child matches.

> [!example]- 📊 Visual: trie + grid walk in lockstep
> ```text
>   words = ["oath", "eat"]    →    trie:
> 
>                root
>               /    \
>              o      e
>              │      │
>              a      a
>              │      │
>              t      t($, "eat")
>              │
>              h($, "oath")
> 
>   Grid:                         Trie pointer follows the path:
>     o  a  a  n
>     e  t  a  e                  (0,0)='o' → trie:o
>     i  h  k  r                  (0,1)='a' → trie:o.a
>     i  f  l  v                  (1,1)='t' → trie:o.a.t
>                                 (2,1)='h' → trie:o.a.t.h ($) ✓ "oath"
> 
>   Both pointers advance together:
>     • Grid pointer = where we are on the board (mark cell visited)
>     • Trie pointer = which prefix we've matched
>   If trie has no child for current grid char → PRUNE this branch.
>   After harvesting a word, delete its '$' to avoid duplicates; if node
>   becomes empty, delete from parent → prune dead twigs.
> ```

> [!info]- 🔍 Dry Run: board=[["o","a","a","n"],["e","t","a","e"],["i","h","k","r"],["i","f","l","v"]], words=["oath","pea","eat","rain"]
> ```text
> Trie of words:
>   root
>    ├ o → a → t → h($, full word "oath")
>    ├ p → e → a($)
>    ├ e → a → t($)
>    └ r → a → i → n($)
> 
> Walk every cell (r,c) and try to descend trie:
> 
> Start (0,0) 'o': trie has 'o' → enter
>   At trie:o, at grid (0,0)
>   Mark (0,0) as visited (replace with '#')
>   Try neighbors: (1,0)='e' — trie:o has 'a' child? Doesn't match 'e' → skip
>     (0,1)='a' — matches trie:o → 'a' → descend
>       At trie:o.a, grid (0,1)
>       Try (1,1)='t' — matches trie:o.a → 't' → descend
>         At trie:o.a.t, grid (1,1)
>         Try (2,1)='h' — matches trie:o.a.t → 'h' → descend
>           At trie:o.a.t.h, end marker '$' = "oath"
>           Found! out.append("oath")
> 
> Start (1,3) 'e': trie has 'e' → enter trie:e
>   (0,3)='n' — trie:e has 'a'? No match
>   (2,3)='r' — no
>   (1,2)='a' — match! descend to trie:e.a
>     check '$': YES at trie:e.a... wait, "ea" is not a word.
>     Actually "eat" is, "pea" is. Path: e→a doesn't have $.
>     Continue:
>     (0,2)='a' — but trie:e.a expects 't' next (for "eat"). Doesn't match.
>     (2,2)='k' — no
>     (1,1)='t' — match! descend to trie:e.a.t which has '$' → "eat"
>     Found! out.append("eat")
> 
> ... (similar exploration finds nothing else; "pea" and "rain" not on the board)
> 
> ✅ Answer: ["oath", "eat"]
> ```

> [!tip] Trie pruning makes this fast
> When a leaf is fully consumed, delete it from its parent — future searches skip it.

> [!success]- JS
> ```js
> const findWords = (board, words) => {
>   const root = {};
>   for (const w of words) {
>     let n = root;
>     for (const c of w) {
>       if (!(c in n)) n[c] = {};
>       n = n[c];
>     }
>     n.$ = w;
>   }
>   const R = board.length, C = board[0].length;
>   const out = [];
>   const dfs = (r, c, n) => {
>     const ch = board[r][c];
>     if (!(ch in n)) return;
>     const nxt = n[ch];
>     if ('$' in nxt) {
>       out.push(nxt.$);
>       delete nxt.$;
>     }
>     board[r][c] = '#';
>     for (const [dr, dc] of [[-1,0],[1,0],[0,-1],[0,1]]) {
>       const nr = r + dr, nc = c + dc;
>       if (nr >= 0 && nr < R && nc >= 0 && nc < C && board[nr][nc] !== '#') {
>         dfs(nr, nc, nxt);
>       }
>     }
>     board[r][c] = ch;
>     if (Object.keys(nxt).length === 0) delete n[ch];
>   };
>   for (let r = 0; r < R; r++) for (let c = 0; c < C; c++) dfs(r, c, root);
>   return out;
> };
> ```

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

**Key takeaway:** Many-pattern search on a grid → trie + DFS + pruning. The trie tells you which paths are still viable.

---

## P4: Replace Words

**LC #648** · Medium

Replace each word in a sentence with its **shortest root** from a dictionary.

### Approach

Build trie of roots. For each word, walk trie; on `isEnd`, output that prefix.

> [!example]- 📊 Visual: stop at the first '$' along the walk
> ```text
>   roots = ["cat", "bat", "rat"]      word = "cattle"
> 
>            root
>           / | \
>          c  b  r
>          │  │  │
>          a  a  a
>          │  │  │
>          t($)t($)t($)
> 
>   Walk "cattle" through trie:
> 
>     'c' ─→ trie:c             (no $)
>     'a' ─→ trie:c.a           (no $)
>     't' ─→ trie:c.a.t   ($!)  STOP — emit "cat"
>     ───── never look at 't','l','e' ─────
> 
>   Replace "cattle" with the shortest matching root "cat".
>   If the walk hits a dead end before any '$' → keep the word unchanged.
> ```

> [!info]- 🔍 Dry Run: dictionary=["cat","bat","rat"], sentence="the cattle was rattled by the battery"
> ```text
> Trie of roots:
>   root
>    ├ c → a → t($)
>    ├ b → a → t($)
>    └ r → a → t($)
> 
> Process each word:
> 
>   "the":
>     walk trie: 't' not in root → no root prefix → keep "the"
> 
>   "cattle":
>     walk trie:
>       i=0 'c' ✓ enter
>       i=1 'a' ✓ enter
>       i=2 't' ✓ enter
>       check $: YES → return "cat" (the shortest root)
>     → replace "cattle" with "cat"
> 
>   "was": walk trie 'w' not in root → keep "was"
> 
>   "rattled": walk → 'r','a','t' → '$' hit → "rat"
> 
>   "by": keep
>   "the": keep
> 
>   "battery": 'b','a','t' → '$' → "bat"
> 
> Join: "the cat was rat by the bat"
> 
> ✅ Answer: "the cat was rat by the bat"
> ```

> [!success]- JS
> ```js
> const replaceWords = (dictionary, sentence) => {
>   const root = {};
>   for (const w of dictionary) {
>     let n = root;
>     for (const c of w) {
>       if (!(c in n)) n[c] = {};
>       n = n[c];
>     }
>     n.$ = true;
>   }
>   const shortest = (word) => {
>     let n = root;
>     for (let i = 0; i < word.length; i++) {
>       const c = word[i];
>       if (!(c in n)) return word;
>       n = n[c];
>       if (n.$) return word.slice(0, i + 1);
>     }
>     return word;
>   };
>   return sentence.split(' ').map(shortest).join(' ');
> };
> ```

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

> [!example]- 📊 Visual: descend only through '$' nodes
> ```text
>   words = ["w","wo","wor","worl","world"]
> 
>            root
>             │
>             w($)         ← every node along path has '$'
>             │
>             o($)
>             │
>             r($)
>             │
>             l($)
>             │
>             d($)         ← deepest valid chain
> 
>   DFS rule:  child must have '$' to be visited.
>   Path so far:  w → wo → wor → worl → world   (best = "world")
> 
>   Counter-example: words=["a","banana"]
>            root
>           /  \
>          a($) b           ← 'b' has NO $
>                │
>                a          ← unreachable: dfs refuses non-$ children
> 
>   So "banana" can't be "built one char at a time" → best = "a".
> ```

> [!info]- 🔍 Dry Run: words=["w","wo","wor","worl","world"]
> ```text
> Trie:
>   root
>    └ w($) → o($) → r($) → l($) → d($)
> 
> All prefixes are valid words ('$' at every level).
> 
> dfs(root, ""):
>   children sorted: ['w']
>   for c='w': child has '$' → recurse dfs(w_node, "w")
>     check '$' at w_node: YES, len("w")=1 > best=0 → best="w"
>     children sorted: ['o']
>     for c='o': child has '$' → dfs(o_node, "wo")
>       best update: len 2 > 1 → best="wo"
>       continue: 'r' has '$' → dfs("wor")
>         best="wor"
>         'l' has '$' → dfs("worl")
>           best="worl"
>           'd' has '$' → dfs("world")
>             best="world" (len 5)
>             d has no more children
> 
> ✅ Answer: "world"
> 
> ─────────────────────────────────────────
> words=["a","banana","app","appl","ap","apply","apple"]
> Trie path 'app' has $ at a,ap,app,appl,apple, and apply.
>   apple len 5
>   apply len 5
> Tie → lex smallest → "apple"
> ```

> [!success]- JS
> ```js
> const longestWord = (words) => {
>   const root = {};
>   for (const w of words) {
>     let n = root;
>     for (const c of w) {
>       if (!(c in n)) n[c] = {};
>       n = n[c];
>     }
>     n.$ = w;
>   }
>   let best = "";
>   const dfs = (n, path) => {
>     if ('$' in n) {
>       if (path.length > best.length || (path.length === best.length && path < best)) {
>         best = path;
>       }
>     }
>     const keys = Object.keys(n).filter(k => k !== '$').sort();
>     for (const c of keys) {
>       const child = n[c];
>       if ('$' in child) dfs(child, path + c);
>     }
>   };
>   dfs(root, "");
>   return best;
> };
> ```

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

> [!example]- 📊 Visual: binary trie + greedy opposite-bit walk
> ```text
>   Nums (5-bit illustration):  5=00101, 25=11001
> 
>   Bit position →  4   3   2   1   0
> 
>                  root
>                  / \
>             bit 0   bit 1                 (MSB)
>                /        \
>             ...           ...
>           (for 5)       (for 25)
> 
>   For x = 5 (00101), greedy walk wanting OPPOSITE bit each level:
> 
>     level 4:  x-bit=0, want 1.  '1' exists (25-path) → take.  cur += 16
>     level 3:  x-bit=0, want 1.  '1' exists       → take.  cur +=  8
>     level 2:  x-bit=1, want 0.  '0' exists       → take.  cur +=  4
>     level 1:  x-bit=0, want 1.  only '0' here    → fallback.
>     level 0:  x-bit=1, want 0.  only '1' here    → fallback.
> 
>     XOR achieved = 16 + 8 + 4 = 28   ( = 5 XOR 25 ✓ )
> 
>   At each level, prefer the toggle-branch if it exists; else follow same-bit.
> ```

> [!info]- 🔍 Dry Run: nums=[3,10,5,25,2,8]
> ```text
> Binary (truncated to 5 bits for illustration):
>    3 = 00011
>   10 = 01010
>    5 = 00101
>   25 = 11001
>    2 = 00010
>    8 = 01000
> 
> Build trie (MSB first):
>   root
>    ├ 0 → 0 → 0 → 1 → 1   (3)
>    ├ 0 → 1 → 0 → 1 → 0   (10)
>    ├ 0 → 0 → 1 → 0 → 1   (5)
>    ├ 1 → 1 → 0 → 0 → 1   (25)
>    ├ 0 → 0 → 0 → 1 → 0   (2)
>    └ 0 → 1 → 0 → 0 → 0   (8)
> 
> For x=5 (00101), walk trie preferring opposite bit:
>   bit 4 of 5 = 0 → want 1 → trie has '1' (path for 25)? YES → cur += 16, descend right
>   bit 3 = 0 → want 1; at this node, only the 25 path is here → '1' present → cur += 8
>   bit 2 = 1 → want 0; 25 path has '0' here → cur += 4
>   bit 1 = 0 → want 1; 25 path has '0' (bit 1 of 25 is 0) → fall back to '0' → cur += 0
>   bit 0 = 1 → want 0; 25 path has '1' (bit 0) → fall back → cur += 0
>   final cur = 16+8+4 = 28
> 
> Check: 5 XOR 25 = 00101 XOR 11001 = 11100 = 28 ✓
> 
> Try other x's... the max across all is 28.
> 
> ✅ Answer: 28
> ```

> [!success]- JS
> ```js
> const findMaximumXOR = (nums) => {
>   const root = {};
>   for (const x of nums) {
>     let n = root;
>     for (let i = 31; i >= 0; i--) {
>       const b = (x >> i) & 1;
>       if (!(b in n)) n[b] = {};
>       n = n[b];
>     }
>   }
>   let best = 0;
>   for (const x of nums) {
>     let n = root, cur = 0;
>     for (let i = 31; i >= 0; i--) {
>       const b = (x >> i) & 1;
>       const toggle = 1 - b;
>       if (toggle in n) {
>         cur |= (1 << i);
>         n = n[toggle];
>       } else {
>         n = n[b];
>       }
>     }
>     best = Math.max(best, cur);
>   }
>   return best;
> };
> ```

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

**LC #1032** · **Hard**

`query(c)` returns true if recent suffix matches any word in dictionary.

### 🧠 Pattern: Reverse Trie + Recent Buffer

> Insert each word **reversed** into a trie. For `query`, walk recent chars **backwards** through the trie; if any prefix (= original suffix) hits `isEnd`, return true.

> [!example]- 📊 Visual: reverse trie + backward walk on buffer
> ```text
>   dictionary = ["cd", "f", "kl"]
> 
>   Insert REVERSED words:    "dc",   "f",   "lk"
> 
>            root
>           / | \
>          d  f($)  l
>          │       │
>          c($)    k($)
> 
>   Stream so far: [... a, b, c, d]
>                                  ↑ newest char (we just appended 'd')
> 
>   Walk buffer BACKWARDS through reverse trie:
> 
>     newest 'd' → root has 'd' ✓ → descend          (no $)
>     prev    'c' → trie:d has 'c' ✓ → descend     ($ → TRUE) → matched "cd"
> 
>   Reading buffer right→left through reverse trie is equivalent
>   to checking "does any dictionary word end exactly here?"
>   Cap buffer length at max-word-length to bound memory.
> ```

> [!info]- 🔍 Dry Run: words=["cd","f","kl"], queries: 'a','b','c','d','e','f','g','h','i','j','k','l'
> ```text
> Insert reversed:
>   "cd" reversed = "dc" → trie path d → c($)
>   "f" reversed  = "f"  → trie path f($)
>   "kl" reversed = "lk" → trie path l → k($)
> 
> Trie:
>   root
>    ├ d → c($)
>    ├ f($)
>    └ l → k($)
> 
> Buffer accumulates each char appended.
> 
> ─────────────────────────────────────────
> query('a'): buf=['a']
>   reverse walk: 'a' → root has 'a'? NO → return false
> 
> query('b'): buf=['a','b']
>   walk reversed: 'b' → no → false
> 
> query('c'): buf=[...,'c']
>   walk reversed: 'c' → root has 'c'? NO → false
> 
> query('d'): buf=[...,'d']
>   walk reversed: 'd' → root has 'd' ✓ → enter; check '$'? NO
>     next char back: 'c' → root.d has 'c' ✓ → check '$' YES → return TRUE
>   (we matched "cd" — the suffix of stream ending at this query)
> 
> query('e'): walk 'e' → false
> 
> query('f'): walk 'f' → root has 'f', '$' YES → TRUE (matched "f")
> 
> ... (continue similarly; 'k' alone won't match because we need 'l' after it)
> 
> query('k'): walk back 'k' → root has 'k'? NO → false   (note: 'kl' reversed starts with 'l')
> 
> query('l'): walk back 'l' → root has 'l' ✓, '$' NO
>             next back 'k' → root.l has 'k' ✓, '$' YES → TRUE
> ```

> [!success]- JS
> ```js
> class StreamChecker {
>   constructor(words) {
>     this.root = {};
>     for (const w of words) {
>       let n = this.root;
>       for (let i = w.length - 1; i >= 0; i--) {
>         const c = w[i];
>         if (!(c in n)) n[c] = {};
>         n = n[c];
>       }
>       n.$ = true;
>     }
>     this.buf = [];
>   }
>   query(c) {
>     this.buf.push(c);
>     let n = this.root;
>     for (let i = this.buf.length - 1; i >= 0; i--) {
>       const ch = this.buf[i];
>       if (!(ch in n)) return false;
>       n = n[ch];
>       if (n.$) return true;
>     }
>     return false;
>   }
> }
> ```

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

> [!example]- 📊 Visual: aggregate sum lives on every node of the path
> ```text
>   After insert("apple", 3):
> 
>     root[_total=3]
>       │
>       a[_total=3]
>       │
>       p[_total=3]
>       │
>       p[_total=3]
>       │
>       l[_total=3]
>       │
>       e[_total=3]
> 
>   Now insert("app", 2):    delta = +2, walk 'a','p','p'.
> 
>     root[5]
>       │
>       a[5]
>       │
>       p[5]
>       │
>       p[5]      ← "app" ends here (implicit)
>       │
>       l[3]      ← unaffected — beyond "app"
>       │
>       e[3]
> 
>   sum("ap")  → walk to p; return node._total = 5  (covers apple + app)
>   sum("app") → walk to second p; return 5
>   sum("apple") → walk to e; return 3
> 
>   Update via DELTA (= newVal − oldVal) so re-inserting same key works.
> ```

> [!info]- 🔍 Dry Run: insert("apple", 3), sum("ap"), insert("app", 2), sum("ap")
> ```text
> Setup: root = {_total: 0}, vals = {}
> 
> ─────────────────────────────────────────
> insert("apple", 3):
>   delta = 3 - 0 = 3
>   vals["apple"] = 3
>   walk and add delta to every node:
>     root._total += 3 → 3
>     'a' node: create, _total=3
>     'p' node: create, _total=3
>     'p' node: create, _total=3
>     'l' node: create, _total=3
>     'e' node: create, _total=3
> 
> ─────────────────────────────────────────
> sum("ap"):
>   walk root → a → p
>   return p._total = 3
> 
> ─────────────────────────────────────────
> insert("app", 2):
>   delta = 2 - 0 = 2 (key "app" was not present)
>   vals["app"] = 2
>   walk a, p, p (existing): add 2 to each + root
>     root._total = 5
>     a._total = 5
>     p._total = 5
>     p._total = 5     ← this is where "app" ends; we mark it implicitly
>   ('app' has 3 chars; walk and increment every level by 2)
> 
> ─────────────────────────────────────────
> sum("ap"):
>   walk root → a → p
>   return p._total = 5     (covers both "apple" and "app" descendants)
> 
> ─────────────────────────────────────────
> If we did insert("apple", 7) later:
>   delta = 7 - 3 = 4
>   walk and add 4 to all nodes on "apple" path.
>   root._total: 5+4=9
> ```

> [!success]- JS
> ```js
> class MapSum {
>   constructor() {
>     this.root = { _total: 0 };
>     this.vals = new Map();
>   }
>   insert(key, val) {
>     const delta = val - (this.vals.get(key) ?? 0);
>     this.vals.set(key, val);
>     let n = this.root;
>     n._total += delta;
>     for (const c of key) {
>       if (!(c in n)) n[c] = { _total: 0 };
>       n = n[c];
>       n._total += delta;
>     }
>   }
>   sum(prefix) {
>     let n = this.root;
>     for (const c of prefix) {
>       if (!(c in n)) return 0;
>       n = n[c];
>     }
>     return n._total;
>   }
> }
> ```

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
