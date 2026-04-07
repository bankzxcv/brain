---
title: "Go Practice: Generics"
date: 2026-04-07
tags:
  - golang
  - practice
  - generics
parent: "[[Golang Study]]"
---

# Generics

Progressive drills for Go generics (type parameters, constraints, generic data structures). Write each solution from scratch.

---

## P1: Generic Min

Write a generic function `Min[T cmp.Ordered](a, b T) T` that returns the smaller of two values. Test it with `int`, `float64`, and `string`.

**Expected output:**
```
Min(3, 7) = 3
Min(2.5, 1.1) = 1.1
Min("apple", "banana") = apple
```

> [!hint]- Hint
> Import `cmp` from the standard library (Go 1.21+). The `cmp.Ordered` constraint covers all numeric types and strings.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"cmp"
> 	"fmt"
> )
> 
> func Min[T cmp.Ordered](a, b T) T {
> 	if a < b {
> 		return a
> 	}
> 	return b
> }
> 
> func main() {
> 	fmt.Printf("Min(3, 7) = %d\n", Min(3, 7))
> 	fmt.Printf("Min(2.5, 1.1) = %.1f\n", Min(2.5, 1.1))
> 	fmt.Printf("Min(\"apple\", \"banana\") = %s\n", Min("apple", "banana"))
> }
> ```

---

## P2: Generic Contains

Write `Contains[T comparable](slice []T, target T) bool` that returns true if the target is in the slice.

**Expected output:**
```
Contains [1,2,3] 2: true
Contains [1,2,3] 5: false
Contains ["go","rust"] "go": true
```

> [!hint]- Hint
> The `comparable` constraint allows using `==`. Loop through the slice and compare each element.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func Contains[T comparable](slice []T, target T) bool {
> 	for _, v := range slice {
> 		if v == target {
> 			return true
> 		}
> 	}
> 	return false
> }
> 
> func main() {
> 	fmt.Println("Contains [1,2,3] 2:", Contains([]int{1, 2, 3}, 2))
> 	fmt.Println("Contains [1,2,3] 5:", Contains([]int{1, 2, 3}, 5))
> 	fmt.Println("Contains [\"go\",\"rust\"] \"go\":", Contains([]string{"go", "rust"}, "go"))
> }
> ```

---

## P3: Generic Map (Transform)

Write `Map[T any, R any](slice []T, fn func(T) R) []R` that transforms each element. Use it to:
1. Convert `[]int{1,2,3}` to their squares.
2. Convert `[]string{"go", "rust"}` to their lengths.

**Expected output:**
```
squares: [1 4 9]
lengths: [2 4]
```

> [!hint]- Hint
> Allocate a result slice of length `len(slice)`. Iterate and apply `fn` to each element.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func Map[T any, R any](slice []T, fn func(T) R) []R {
> 	result := make([]R, len(slice))
> 	for i, v := range slice {
> 		result[i] = fn(v)
> 	}
> 	return result
> }
> 
> func main() {
> 	squares := Map([]int{1, 2, 3}, func(n int) int { return n * n })
> 	fmt.Println("squares:", squares)
> 
> 	lengths := Map([]string{"go", "rust"}, func(s string) int { return len(s) })
> 	fmt.Println("lengths:", lengths)
> }
> ```

---

## P4: Generic Filter

Write `Filter[T any](slice []T, predicate func(T) bool) []T`. Use it to:
1. Keep only even numbers from `[]int{1,2,3,4,5,6}`.
2. Keep only strings longer than 3 characters from `[]string{"go", "java", "rust", "c"}`.

**Expected output:**
```
evens: [2 4 6]
long: [java rust]
```

> [!hint]- Hint
> Start with an empty (nil) slice. Append elements where the predicate returns true.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func Filter[T any](slice []T, predicate func(T) bool) []T {
> 	var result []T
> 	for _, v := range slice {
> 		if predicate(v) {
> 			result = append(result, v)
> 		}
> 	}
> 	return result
> }
> 
> func main() {
> 	evens := Filter([]int{1, 2, 3, 4, 5, 6}, func(n int) bool { return n%2 == 0 })
> 	fmt.Println("evens:", evens)
> 
> 	long := Filter([]string{"go", "java", "rust", "c"}, func(s string) bool { return len(s) > 3 })
> 	fmt.Println("long:", long)
> }
> ```

---

## P5: Generic Reduce

Write `Reduce[T any, R any](slice []T, initial R, fn func(R, T) R) R`. Use it to:
1. Sum `[]int{1,2,3,4,5}`.
2. Concatenate `[]string{"Hello", " ", "World"}`.

**Expected output:**
```
sum: 15
concat: Hello World
```

> [!hint]- Hint
> Start with `acc := initial`. For each element, `acc = fn(acc, element)`. Return `acc`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func Reduce[T any, R any](slice []T, initial R, fn func(R, T) R) R {
> 	acc := initial
> 	for _, v := range slice {
> 		acc = fn(acc, v)
> 	}
> 	return acc
> }
> 
> func main() {
> 	sum := Reduce([]int{1, 2, 3, 4, 5}, 0, func(acc, v int) int { return acc + v })
> 	fmt.Println("sum:", sum)
> 
> 	concat := Reduce([]string{"Hello", " ", "World"}, "", func(acc, v string) string { return acc + v })
> 	fmt.Println("concat:", concat)
> }
> ```

---

## P6: Generic Stack

Implement a `Stack[T any]` with methods:
- `Push(val T)`
- `Pop() (T, bool)`
- `Peek() (T, bool)`
- `Size() int`

Test with both `int` and `string`.

**Expected output:**
```
push 1, 2, 3
pop: 3
peek: 2
size: 2
push "a", "b"
pop: b
```

> [!hint]- Hint
> Use a slice as the backing store. `Pop` returns the last element. For the zero value on empty stack, use `var zero T`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Stack[T any] struct {
> 	items []T
> }
> 
> func (s *Stack[T]) Push(val T) {
> 	s.items = append(s.items, val)
> }
> 
> func (s *Stack[T]) Pop() (T, bool) {
> 	if len(s.items) == 0 {
> 		var zero T
> 		return zero, false
> 	}
> 	val := s.items[len(s.items)-1]
> 	s.items = s.items[:len(s.items)-1]
> 	return val, true
> }
> 
> func (s *Stack[T]) Peek() (T, bool) {
> 	if len(s.items) == 0 {
> 		var zero T
> 		return zero, false
> 	}
> 	return s.items[len(s.items)-1], true
> }
> 
> func (s *Stack[T]) Size() int {
> 	return len(s.items)
> }
> 
> func main() {
> 	// int stack
> 	var s Stack[int]
> 	s.Push(1)
> 	s.Push(2)
> 	s.Push(3)
> 	fmt.Println("push 1, 2, 3")
> 
> 	v, _ := s.Pop()
> 	fmt.Println("pop:", v)
> 
> 	v, _ = s.Peek()
> 	fmt.Println("peek:", v)
> 	fmt.Println("size:", s.Size())
> 
> 	// string stack
> 	var ss Stack[string]
> 	ss.Push("a")
> 	ss.Push("b")
> 	fmt.Println("push \"a\", \"b\"")
> 
> 	sv, _ := ss.Pop()
> 	fmt.Println("pop:", sv)
> }
> ```

---

## P7: Generic Linked List

Implement a singly-linked list `LinkedList[T any]` with:
- `Append(val T)`
- `Prepend(val T)`
- `Print()` — prints all values separated by `" -> "`
- `Len() int`

**Expected output:**
```
0 -> 1 -> 2 -> 3
length: 4
```

> [!hint]- Hint
> Define a `Node[T any]` with `Value T` and `Next *Node[T]`. `LinkedList` holds a `head *Node[T]`. `Append` walks to the end.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Node[T any] struct {
> 	Value T
> 	Next  *Node[T]
> }
> 
> type LinkedList[T any] struct {
> 	head *Node[T]
> 	size int
> }
> 
> func (ll *LinkedList[T]) Append(val T) {
> 	newNode := &Node[T]{Value: val}
> 	ll.size++
> 	if ll.head == nil {
> 		ll.head = newNode
> 		return
> 	}
> 	current := ll.head
> 	for current.Next != nil {
> 		current = current.Next
> 	}
> 	current.Next = newNode
> }
> 
> func (ll *LinkedList[T]) Prepend(val T) {
> 	newNode := &Node[T]{Value: val, Next: ll.head}
> 	ll.head = newNode
> 	ll.size++
> }
> 
> func (ll *LinkedList[T]) Print() {
> 	current := ll.head
> 	for current != nil {
> 		if current.Next != nil {
> 			fmt.Printf("%v -> ", current.Value)
> 		} else {
> 			fmt.Printf("%v", current.Value)
> 		}
> 		current = current.Next
> 	}
> 	fmt.Println()
> }
> 
> func (ll *LinkedList[T]) Len() int {
> 	return ll.size
> }
> 
> func main() {
> 	var ll LinkedList[int]
> 	ll.Append(1)
> 	ll.Append(2)
> 	ll.Append(3)
> 	ll.Prepend(0)
> 	ll.Print()
> 	fmt.Println("length:", ll.Len())
> }
> ```

---

## P8: Type Constraint with Multiple Methods

Define a `Sortable` interface constraint that requires both `cmp.Ordered` and a `String() string` method. Create a `NamedInt` type satisfying it. Write a generic `SortAndPrint[T Sortable]` that sorts a slice and prints each element using `String()`.

**Expected output:**
```
one (1)
three (3)
five (5)
```

> [!hint]- Hint
> Interface constraints can embed other interfaces and list method signatures: `type Sortable interface { cmp.Ordered; String() string }`. But note: `cmp.Ordered` is a union of types, so you need a custom approach. Use `~int` or define your own constraint with `<` support.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sort"
> )
> 
> // Constraint: must support ordering and have a String method
> type Sortable interface {
> 	~int | ~float64 | ~string
> 	String() string
> }
> 
> type NamedInt struct {
> 	Name string
> 	Val  int
> }
> 
> // We can't directly use operators with interface methods easily,
> // so let's use a different approach: constraint with a Less method.
> 
> type Ordered interface {
> 	Less(other Ordered) bool
> 	String() string
> }
> 
> // Simpler practical approach using sort.Interface pattern with generics:
> 
> type HasLessAndString[T any] interface {
> 	Less(T) bool
> 	fmt.Stringer
> }
> 
> func SortAndPrint[T HasLessAndString[T]](items []T) {
> 	sort.Slice(items, func(i, j int) bool {
> 		return items[i].Less(items[j])
> 	})
> 	for _, item := range items {
> 		fmt.Println(item.String())
> 	}
> }
> 
> type RankedItem struct {
> 	Name string
> 	Rank int
> }
> 
> func (r RankedItem) Less(other RankedItem) bool {
> 	return r.Rank < other.Rank
> }
> 
> func (r RankedItem) String() string {
> 	return fmt.Sprintf("%s (%d)", r.Name, r.Rank)
> }
> 
> func main() {
> 	items := []RankedItem{
> 		{Name: "three", Rank: 3},
> 		{Name: "one", Rank: 1},
> 		{Name: "five", Rank: 5},
> 	}
> 	SortAndPrint(items)
> }
> ```

---

## P9: Generic Binary Search

Write `BinarySearch[T cmp.Ordered](sorted []T, target T) (int, bool)` that returns the index and whether the target was found.

**Expected output:**
```
search 5 in [1 3 5 7 9]: index=2, found=true
search 4 in [1 3 5 7 9]: index=0, found=false
search "go" in [c go python rust]: index=1, found=true
```

> [!hint]- Hint
> Standard binary search: maintain `lo` and `hi` indices. Compare `sorted[mid]` with `target`. Use `<` and `>` operators which work with `cmp.Ordered`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"cmp"
> 	"fmt"
> )
> 
> func BinarySearch[T cmp.Ordered](sorted []T, target T) (int, bool) {
> 	lo, hi := 0, len(sorted)-1
> 	for lo <= hi {
> 		mid := lo + (hi-lo)/2
> 		switch {
> 		case sorted[mid] == target:
> 			return mid, true
> 		case sorted[mid] < target:
> 			lo = mid + 1
> 		default:
> 			hi = mid - 1
> 		}
> 	}
> 	return 0, false
> }
> 
> func main() {
> 	nums := []int{1, 3, 5, 7, 9}
> 	idx, found := BinarySearch(nums, 5)
> 	fmt.Printf("search 5 in %v: index=%d, found=%v\n", nums, idx, found)
> 
> 	idx, found = BinarySearch(nums, 4)
> 	fmt.Printf("search 4 in %v: index=%d, found=%v\n", nums, idx, found)
> 
> 	strs := []string{"c", "go", "python", "rust"}
> 	idx, found = BinarySearch(strs, "go")
> 	fmt.Printf("search \"go\" in %v: index=%d, found=%v\n", strs, idx, found)
> }
> ```

---

## P10: Generic Cache with TTL

Implement `Cache[K comparable, V any]` with:
- `Set(key K, value V, ttl time.Duration)`
- `Get(key K) (V, bool)` — returns false if expired or missing

Test: set a value with 500ms TTL, read immediately (hit), sleep 600ms, read again (miss).

**Expected output:**
```
get "x": val=42, ok=true
... 600ms later ...
get "x": val=0, ok=false
```

> [!hint]- Hint
> Store entries as `struct { Value V; ExpiresAt time.Time }`. On `Get`, check if `time.Now().After(entry.ExpiresAt)`. Use a `sync.RWMutex` for thread safety.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sync"
> 	"time"
> )
> 
> type entry[V any] struct {
> 	Value     V
> 	ExpiresAt time.Time
> }
> 
> type Cache[K comparable, V any] struct {
> 	mu    sync.RWMutex
> 	items map[K]entry[V]
> }
> 
> func NewCache[K comparable, V any]() *Cache[K, V] {
> 	return &Cache[K, V]{
> 		items: make(map[K]entry[V]),
> 	}
> }
> 
> func (c *Cache[K, V]) Set(key K, value V, ttl time.Duration) {
> 	c.mu.Lock()
> 	defer c.mu.Unlock()
> 	c.items[key] = entry[V]{
> 		Value:     value,
> 		ExpiresAt: time.Now().Add(ttl),
> 	}
> }
> 
> func (c *Cache[K, V]) Get(key K) (V, bool) {
> 	c.mu.RLock()
> 	defer c.mu.RUnlock()
> 	e, ok := c.items[key]
> 	if !ok || time.Now().After(e.ExpiresAt) {
> 		var zero V
> 		return zero, false
> 	}
> 	return e.Value, true
> }
> 
> func main() {
> 	cache := NewCache[string, int]()
> 
> 	cache.Set("x", 42, 500*time.Millisecond)
> 
> 	val, ok := cache.Get("x")
> 	fmt.Printf("get \"x\": val=%d, ok=%v\n", val, ok)
> 
> 	time.Sleep(600 * time.Millisecond)
> 	fmt.Println("... 600ms later ...")
> 
> 	val, ok = cache.Get("x")
> 	fmt.Printf("get \"x\": val=%d, ok=%v\n", val, ok)
> }
> ```

---

## P11: Generic Result Type

Implement a `Result[T any]` type that represents either a success value or an error (like Rust's `Result`). Provide:
- `Ok[T](val T) Result[T]`
- `Err[T](err error) Result[T]`
- `IsOk() bool`
- `Unwrap() T` (panics on error)
- `UnwrapOr(def T) T`
- `Map(fn func(T) T) Result[T]`

**Expected output:**
```
result: Ok(42)
mapped: Ok(84)
error result: Err(something failed)
unwrap or: 0
```

> [!hint]- Hint
> `Result[T]` has fields `value T`, `err error`, and `ok bool`. `Map` applies the function only if `ok` is true.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> )
> 
> type Result[T any] struct {
> 	value T
> 	err   error
> 	ok    bool
> }
> 
> func Ok[T any](val T) Result[T] {
> 	return Result[T]{value: val, ok: true}
> }
> 
> func Err[T any](err error) Result[T] {
> 	return Result[T]{err: err, ok: false}
> }
> 
> func (r Result[T]) IsOk() bool {
> 	return r.ok
> }
> 
> func (r Result[T]) Unwrap() T {
> 	if !r.ok {
> 		panic(fmt.Sprintf("called Unwrap on Err: %v", r.err))
> 	}
> 	return r.value
> }
> 
> func (r Result[T]) UnwrapOr(def T) T {
> 	if !r.ok {
> 		return def
> 	}
> 	return r.value
> }
> 
> func (r Result[T]) Map(fn func(T) T) Result[T] {
> 	if !r.ok {
> 		return r
> 	}
> 	return Ok(fn(r.value))
> }
> 
> func (r Result[T]) String() string {
> 	if r.ok {
> 		return fmt.Sprintf("Ok(%v)", r.value)
> 	}
> 	return fmt.Sprintf("Err(%v)", r.err)
> }
> 
> func main() {
> 	r := Ok(42)
> 	fmt.Println("result:", r)
> 
> 	mapped := r.Map(func(v int) int { return v * 2 })
> 	fmt.Println("mapped:", mapped)
> 
> 	e := Err[int](fmt.Errorf("something failed"))
> 	fmt.Println("error result:", e)
> 	fmt.Println("unwrap or:", e.UnwrapOr(0))
> }
> ```

---

## P12: Generic Binary Search Tree

Implement a `BST[T cmp.Ordered]` with:
- `Insert(val T)`
- `Search(val T) bool`
- `InOrder() []T` — returns values in sorted order

**Expected output:**
```
insert: 5, 3, 7, 1, 4
search 4: true
search 6: false
in-order: [1 3 4 5 7]
```

> [!hint]- Hint
> Each node has `Value T`, `Left *node[T]`, `Right *node[T]`. Insert recursively: go left if `val < node.Value`, right if greater. InOrder: left, self, right.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"cmp"
> 	"fmt"
> )
> 
> type node[T cmp.Ordered] struct {
> 	Value T
> 	Left  *node[T]
> 	Right *node[T]
> }
> 
> type BST[T cmp.Ordered] struct {
> 	root *node[T]
> }
> 
> func (t *BST[T]) Insert(val T) {
> 	t.root = insert(t.root, val)
> }
> 
> func insert[T cmp.Ordered](n *node[T], val T) *node[T] {
> 	if n == nil {
> 		return &node[T]{Value: val}
> 	}
> 	if val < n.Value {
> 		n.Left = insert(n.Left, val)
> 	} else if val > n.Value {
> 		n.Right = insert(n.Right, val)
> 	}
> 	return n
> }
> 
> func (t *BST[T]) Search(val T) bool {
> 	return search(t.root, val)
> }
> 
> func search[T cmp.Ordered](n *node[T], val T) bool {
> 	if n == nil {
> 		return false
> 	}
> 	if val == n.Value {
> 		return true
> 	}
> 	if val < n.Value {
> 		return search(n.Left, val)
> 	}
> 	return search(n.Right, val)
> }
> 
> func (t *BST[T]) InOrder() []T {
> 	var result []T
> 	inOrder(t.root, &result)
> 	return result
> }
> 
> func inOrder[T cmp.Ordered](n *node[T], result *[]T) {
> 	if n == nil {
> 		return
> 	}
> 	inOrder(n.Left, result)
> 	*result = append(*result, n.Value)
> 	inOrder(n.Right, result)
> }
> 
> func main() {
> 	var tree BST[int]
> 	for _, v := range []int{5, 3, 7, 1, 4} {
> 		tree.Insert(v)
> 	}
> 	fmt.Println("insert: 5, 3, 7, 1, 4")
> 
> 	fmt.Println("search 4:", tree.Search(4))
> 	fmt.Println("search 6:", tree.Search(6))
> 	fmt.Println("in-order:", tree.InOrder())
> }
> ```

---

## P13: Generic Pipeline Builder

Build a `Pipeline[T any]` that chains transformations. Provide:
- `NewPipeline[T any]() *Pipeline[T]`
- `Then(fn func(T) T) *Pipeline[T]` — adds a transformation step
- `Execute(input T) T` — runs all steps in order

Test: build a pipeline for `int` that adds 1, doubles, then subtracts 3. Execute with input 5. Expected: `((5 + 1) * 2) - 3 = 9`.

**Expected output:**
```
result: 9
```

Also test with `string`: trim spaces, convert to uppercase.

**Expected output:**
```
result: HELLO WORLD
```

> [!hint]- Hint
> Store `[]func(T) T` in the pipeline. `Then` appends to the slice and returns the pipeline (for chaining). `Execute` applies each function in order.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> type Pipeline[T any] struct {
> 	steps []func(T) T
> }
> 
> func NewPipeline[T any]() *Pipeline[T] {
> 	return &Pipeline[T]{}
> }
> 
> func (p *Pipeline[T]) Then(fn func(T) T) *Pipeline[T] {
> 	p.steps = append(p.steps, fn)
> 	return p
> }
> 
> func (p *Pipeline[T]) Execute(input T) T {
> 	result := input
> 	for _, step := range p.steps {
> 		result = step(result)
> 	}
> 	return result
> }
> 
> func main() {
> 	// int pipeline: add 1, double, subtract 3
> 	intPipe := NewPipeline[int]().
> 		Then(func(n int) int { return n + 1 }).
> 		Then(func(n int) int { return n * 2 }).
> 		Then(func(n int) int { return n - 3 })
> 
> 	fmt.Println("result:", intPipe.Execute(5)) // ((5+1)*2)-3 = 9
> 
> 	// string pipeline: trim, uppercase
> 	strPipe := NewPipeline[string]().
> 		Then(strings.TrimSpace).
> 		Then(strings.ToUpper)
> 
> 	fmt.Println("result:", strPipe.Execute("  hello world  ")) // HELLO WORLD
> }
> ```
