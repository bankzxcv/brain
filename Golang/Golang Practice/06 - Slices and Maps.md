---
title: "Go Practice: Slices and Maps"
date: 2026-04-07
tags:
  - golang
  - practice
  - slices-and-maps
parent: "[[Golang Study]]"
---

# Go Practice: Slices and Maps

15 progressive problems covering slices and maps - from basics to data structure implementation.

---

## P1: Slice Basics - Append, Length, Capacity

Create an empty int slice. Append the values 10, 20, 30, 40, 50 one at a time. After each append, print the length and capacity.

**Expected output (capacity may vary):**
```
After appending 10: len=1, cap=1
After appending 20: len=2, cap=2
After appending 30: len=3, cap=4
After appending 40: len=4, cap=4
After appending 50: len=5, cap=8
```

> [!hint]- Hint
> Use `append(slice, element)` and reassign the result. Go doubles the capacity (approximately) when the backing array is full. Use `len()` and `cap()` to inspect.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	var s []int
> 
> 	values := []int{10, 20, 30, 40, 50}
> 	for _, v := range values {
> 		s = append(s, v)
> 		fmt.Printf("After appending %d: len=%d, cap=%d\n", v, len(s), cap(s))
> 	}
> }
> ```

---

## P2: Slice from Array - Shared Backing Array

Create an array `[5]int{1, 2, 3, 4, 5}`. Take a slice of elements at indices 1 through 3. Modify the slice's first element. Print both the array and the slice to show the array changed.

**Expected output:**
```
Array before: [1 2 3 4 5]
Slice: [2 3 4]
Modifying slice[0] = 99
Array after:  [1 99 3 4 5]
Slice after:  [99 3 4]
```

> [!hint]- Hint
> `arr[1:4]` creates a slice viewing elements 1, 2, 3 of the array. The slice shares memory with the array, so modifying one modifies the other.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	arr := [5]int{1, 2, 3, 4, 5}
> 	fmt.Println("Array before:", arr)
> 
> 	sl := arr[1:4]
> 	fmt.Println("Slice:", sl)
> 
> 	fmt.Println("Modifying slice[0] = 99")
> 	sl[0] = 99
> 
> 	fmt.Println("Array after: ", arr)
> 	fmt.Println("Slice after: ", sl)
> }
> ```

---

## P3: Copy a Slice Independently

Given `original := []int{1, 2, 3, 4, 5}`, create an independent copy. Modify the copy and show the original is unchanged.

**Expected output:**
```
Original: [1 2 3 4 5]
Copy:     [1 2 3 4 5]
After modifying copy[0] = 99:
Original: [1 2 3 4 5]
Copy:     [99 2 3 4 5]
```

> [!hint]- Hint
> Use the built-in `copy(dst, src)` function. You must create the destination slice with `make([]int, len(original))` first. Alternatively, `append([]int{}, original...)` also works.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	original := []int{1, 2, 3, 4, 5}
> 
> 	// Method 1: copy()
> 	cpy := make([]int, len(original))
> 	copy(cpy, original)
> 
> 	// Method 2 (alternative): append
> 	// cpy := append([]int{}, original...)
> 
> 	fmt.Println("Original:", original)
> 	fmt.Println("Copy:    ", cpy)
> 
> 	cpy[0] = 99
> 
> 	fmt.Println("After modifying copy[0] = 99:")
> 	fmt.Println("Original:", original)
> 	fmt.Println("Copy:    ", cpy)
> }
> ```

---

## P4: Delete an Element from a Slice

Given `s := []int{10, 20, 30, 40, 50}`, write two functions:
1. `deleteOrdered(s []int, i int) []int` - removes element at index `i`, preserving order
2. `deleteUnordered(s []int, i int) []int` - removes element at index `i`, does NOT preserve order (fast)

Delete the element at index 2 (value 30) using both approaches.

**Expected output:**
```
Original: [10 20 30 40 50]
Order-preserving delete index 2: [10 20 40 50]
Unordered delete index 2:        [10 20 50 40]
```

> [!hint]- Hint
> Order-preserving: `append(s[:i], s[i+1:]...)`
> Unordered (fast): swap the element with the last one, then truncate: `s[i] = s[len(s)-1]; s[:len(s)-1]`
> 
> Both approaches modify the original slice's backing array. Make copies if you need both results.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func deleteOrdered(s []int, i int) []int {
> 	return append(s[:i], s[i+1:]...)
> }
> 
> func deleteUnordered(s []int, i int) []int {
> 	s[i] = s[len(s)-1]
> 	return s[:len(s)-1]
> }
> 
> func main() {
> 	original := []int{10, 20, 30, 40, 50}
> 	fmt.Println("Original:", original)
> 
> 	// Make copies since both approaches mutate the backing array
> 	s1 := make([]int, len(original))
> 	copy(s1, original)
> 	s1 = deleteOrdered(s1, 2)
> 	fmt.Println("Order-preserving delete index 2:", s1)
> 
> 	s2 := make([]int, len(original))
> 	copy(s2, original)
> 	s2 = deleteUnordered(s2, 2)
> 	fmt.Println("Unordered delete index 2:       ", s2)
> }
> ```

---

## P5: Filter a Slice with a Predicate

Write a generic function `filter[T any](s []T, pred func(T) bool) []T` that returns a new slice containing only elements where `pred` returns true.

Use it to filter even numbers from `[]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}`.

**Expected output:**
```
Original: [1 2 3 4 5 6 7 8 9 10]
Evens:    [2 4 6 8 10]
```

> [!hint]- Hint
> Use Go 1.18+ generics with `[T any]`. Inside the function, iterate and append matching elements to a new slice. The predicate for even numbers is `func(n int) bool { return n%2 == 0 }`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func filter[T any](s []T, pred func(T) bool) []T {
> 	var result []T
> 	for _, v := range s {
> 		if pred(v) {
> 			result = append(result, v)
> 		}
> 	}
> 	return result
> }
> 
> func main() {
> 	nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
> 	fmt.Println("Original:", nums)
> 
> 	evens := filter(nums, func(n int) bool {
> 		return n%2 == 0
> 	})
> 	fmt.Println("Evens:   ", evens)
> }
> ```

---

## P6: Map (Transform) a Slice

Write a generic function `mapSlice[T, U any](s []T, fn func(T) U) []U` that transforms each element.

Use it to convert `[]string{"hello", "world", "go"}` to uppercase.

**Expected output:**
```
Original:  [hello world go]
Uppercase: [HELLO WORLD GO]
```

> [!hint]- Hint
> The function creates a result slice of type `[]U` with the same length as the input. Use `strings.ToUpper` as the transform function. Named `mapSlice` to avoid conflict with the built-in `map` type.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> func mapSlice[T, U any](s []T, fn func(T) U) []U {
> 	result := make([]U, len(s))
> 	for i, v := range s {
> 		result[i] = fn(v)
> 	}
> 	return result
> }
> 
> func main() {
> 	words := []string{"hello", "world", "go"}
> 	fmt.Println("Original: ", words)
> 
> 	upper := mapSlice(words, strings.ToUpper)
> 	fmt.Println("Uppercase:", upper)
> }
> ```

---

## P7: Maps - Create, Access, Check Existence

Create a map of country codes to country names. Add entries, look up an existing key, look up a missing key (using comma-ok idiom), and delete an entry.

**Expected output:**
```
US -> United States
Looking up "JP": Japan (found: true)
Looking up "XX": (not found)
After deleting "DE": map[JP:Japan US:United States]
```

> [!hint]- Hint
> Use the comma-ok idiom: `val, ok := m[key]`. If `ok` is false, the key doesn't exist. Use `delete(m, key)` to remove an entry.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	countries := map[string]string{
> 		"US": "United States",
> 		"JP": "Japan",
> 		"DE": "Germany",
> 	}
> 
> 	fmt.Println("US ->", countries["US"])
> 
> 	if val, ok := countries["JP"]; ok {
> 		fmt.Printf("Looking up \"JP\": %s (found: %t)\n", val, ok)
> 	}
> 
> 	if _, ok := countries["XX"]; !ok {
> 		fmt.Println("Looking up \"XX\": (not found)")
> 	}
> 
> 	delete(countries, "DE")
> 	fmt.Println("After deleting \"DE\":", countries)
> }
> ```

---

## P8: Map of Slices - Adjacency List Graph

Build a directed graph as an adjacency list using `map[string][]string`. Add edges:
- A -> B, C
- B -> D
- C -> D, E
- D -> E

Print the neighbors of each node.

**Expected output:**
```
A -> [B C]
B -> [D]
C -> [D E]
D -> [E]
```

> [!hint]- Hint
> Use `graph[from] = append(graph[from], to)` to add edges. The map value is a `[]string`, so you can append to it directly. Iterate with `for node, neighbors := range graph` but note map iteration order is random - sort the keys first for deterministic output.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sort"
> )
> 
> func main() {
> 	graph := make(map[string][]string)
> 
> 	// Add edges
> 	addEdge := func(from, to string) {
> 		graph[from] = append(graph[from], to)
> 	}
> 
> 	addEdge("A", "B")
> 	addEdge("A", "C")
> 	addEdge("B", "D")
> 	addEdge("C", "D")
> 	addEdge("C", "E")
> 	addEdge("D", "E")
> 
> 	// Print in sorted order for deterministic output
> 	keys := make([]string, 0, len(graph))
> 	for k := range graph {
> 		keys = append(keys, k)
> 	}
> 	sort.Strings(keys)
> 
> 	for _, node := range keys {
> 		fmt.Printf("%s -> %v\n", node, graph[node])
> 	}
> }
> ```

---

## P9: Set Using map[T]struct{}

Implement a generic `Set[T comparable]` with methods:
- `Add(item T)`
- `Contains(item T) bool`
- `Remove(item T)`
- `Size() int`
- `Items() []T`

Demonstrate adding, checking membership, and removing.

**Expected output:**
```
Set after adding go, rust, python: {go, python, rust}
Contains "go": true
Contains "java": false
After removing "rust": {go, python}
Size: 2
```

> [!hint]- Hint
> Use `map[T]struct{}` as the backing store. `struct{}` takes zero bytes. To add: `m[item] = struct{}{}`. To check: `_, ok := m[item]`. To delete: `delete(m, item)`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sort"
> 	"strings"
> )
> 
> type Set[T comparable] struct {
> 	m map[T]struct{}
> }
> 
> func NewSet[T comparable]() *Set[T] {
> 	return &Set[T]{m: make(map[T]struct{})}
> }
> 
> func (s *Set[T]) Add(item T) {
> 	s.m[item] = struct{}{}
> }
> 
> func (s *Set[T]) Contains(item T) bool {
> 	_, ok := s.m[item]
> 	return ok
> }
> 
> func (s *Set[T]) Remove(item T) {
> 	delete(s.m, item)
> }
> 
> func (s *Set[T]) Size() int {
> 	return len(s.m)
> }
> 
> func (s *Set[T]) Items() []T {
> 	items := make([]T, 0, len(s.m))
> 	for k := range s.m {
> 		items = append(items, k)
> 	}
> 	return items
> }
> 
> // String for display (specific to string sets for sorted output)
> func (s *Set[T]) String() string {
> 	items := s.Items()
> 	strs := make([]string, len(items))
> 	for i, item := range items {
> 		strs[i] = fmt.Sprint(item)
> 	}
> 	sort.Strings(strs)
> 	return "{" + strings.Join(strs, ", ") + "}"
> }
> 
> func main() {
> 	s := NewSet[string]()
> 	s.Add("go")
> 	s.Add("rust")
> 	s.Add("python")
> 	fmt.Printf("Set after adding go, rust, python: %s\n", s)
> 
> 	fmt.Printf("Contains \"go\": %t\n", s.Contains("go"))
> 	fmt.Printf("Contains \"java\": %t\n", s.Contains("java"))
> 
> 	s.Remove("rust")
> 	fmt.Printf("After removing \"rust\": %s\n", s)
> 	fmt.Printf("Size: %d\n", s.Size())
> }
> ```

---

## P10: Word Frequency Counter

Write a function `wordFrequency(text string) map[string]int` that counts how many times each word appears. Convert words to lowercase. Print the results sorted by frequency (descending).

**Input:** `"the quick brown fox jumps over the lazy dog the fox"`

**Expected output:**
```
the: 3
fox: 2
brown: 1
dog: 1
jumps: 1
lazy: 1
over: 1
quick: 1
```

> [!hint]- Hint
> Use `strings.Fields(text)` to split on whitespace and `strings.ToLower()` to normalize. To sort by frequency, create a slice of key-value pairs and sort with `sort.Slice`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sort"
> 	"strings"
> )
> 
> func wordFrequency(text string) map[string]int {
> 	freq := make(map[string]int)
> 	for _, word := range strings.Fields(text) {
> 		freq[strings.ToLower(word)]++
> 	}
> 	return freq
> }
> 
> func main() {
> 	text := "the quick brown fox jumps over the lazy dog the fox"
> 	freq := wordFrequency(text)
> 
> 	// Sort by frequency (desc), then alphabetically
> 	type wordCount struct {
> 		word  string
> 		count int
> 	}
> 
> 	pairs := make([]wordCount, 0, len(freq))
> 	for w, c := range freq {
> 		pairs = append(pairs, wordCount{w, c})
> 	}
> 
> 	sort.Slice(pairs, func(i, j int) bool {
> 		if pairs[i].count != pairs[j].count {
> 			return pairs[i].count > pairs[j].count
> 		}
> 		return pairs[i].word < pairs[j].word
> 	})
> 
> 	for _, p := range pairs {
> 		fmt.Printf("%s: %d\n", p.word, p.count)
> 	}
> }
> ```

---

## P11: Group Items by Category

Given a list of products with name and category, group them into a `map[string][]string` where the key is the category and the value is a list of product names.

**Input data:**
```
{Name: "iPhone", Category: "Electronics"}
{Name: "Banana", Category: "Food"}
{Name: "MacBook", Category: "Electronics"}
{Name: "Apple", Category: "Food"}
{Name: "Shirt", Category: "Clothing"}
{Name: "Headphones", Category: "Electronics"}
```

**Expected output:**
```
Clothing: [Shirt]
Electronics: [iPhone MacBook Headphones]
Food: [Banana Apple]
```

> [!hint]- Hint
> Iterate over the products and `append` each product name to the slice under its category key. The zero value of a map value (`nil` slice) works with `append`, so no need to check if the key exists first.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sort"
> )
> 
> type Product struct {
> 	Name     string
> 	Category string
> }
> 
> func groupByCategory(products []Product) map[string][]string {
> 	groups := make(map[string][]string)
> 	for _, p := range products {
> 		groups[p.Category] = append(groups[p.Category], p.Name)
> 	}
> 	return groups
> }
> 
> func main() {
> 	products := []Product{
> 		{"iPhone", "Electronics"},
> 		{"Banana", "Food"},
> 		{"MacBook", "Electronics"},
> 		{"Apple", "Food"},
> 		{"Shirt", "Clothing"},
> 		{"Headphones", "Electronics"},
> 	}
> 
> 	groups := groupByCategory(products)
> 
> 	// Print sorted by category for deterministic output
> 	categories := make([]string, 0, len(groups))
> 	for cat := range groups {
> 		categories = append(categories, cat)
> 	}
> 	sort.Strings(categories)
> 
> 	for _, cat := range categories {
> 		fmt.Printf("%s: %v\n", cat, groups[cat])
> 	}
> }
> ```

---

## P12: Stack and Queue Using Slices

Implement:
1. A `Stack[T]` with `Push`, `Pop`, `Peek`, `IsEmpty`, `Size`
2. A `Queue[T]` with `Enqueue`, `Dequeue`, `Front`, `IsEmpty`, `Size`

Demonstrate both with a sequence of operations.

**Expected output:**
```
=== Stack ===
Push: 1, 2, 3
Peek: 3
Pop: 3
Pop: 2
Size: 1

=== Queue ===
Enqueue: a, b, c
Front: a
Dequeue: a
Dequeue: b
Size: 1
```

> [!hint]- Hint
> Stack: append to push, slice `[:len-1]` to pop, last element to peek.
> Queue: append to enqueue, first element to dequeue and reslice `[1:]`. Note that this queue implementation leaks memory over time in production - for practice it's fine.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> // Stack
> type Stack[T any] struct {
> 	items []T
> }
> 
> func (s *Stack[T]) Push(item T) {
> 	s.items = append(s.items, item)
> }
> 
> func (s *Stack[T]) Pop() (T, bool) {
> 	if len(s.items) == 0 {
> 		var zero T
> 		return zero, false
> 	}
> 	item := s.items[len(s.items)-1]
> 	s.items = s.items[:len(s.items)-1]
> 	return item, true
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
> func (s *Stack[T]) IsEmpty() bool { return len(s.items) == 0 }
> func (s *Stack[T]) Size() int     { return len(s.items) }
> 
> // Queue
> type Queue[T any] struct {
> 	items []T
> }
> 
> func (q *Queue[T]) Enqueue(item T) {
> 	q.items = append(q.items, item)
> }
> 
> func (q *Queue[T]) Dequeue() (T, bool) {
> 	if len(q.items) == 0 {
> 		var zero T
> 		return zero, false
> 	}
> 	item := q.items[0]
> 	q.items = q.items[1:]
> 	return item, true
> }
> 
> func (q *Queue[T]) Front() (T, bool) {
> 	if len(q.items) == 0 {
> 		var zero T
> 		return zero, false
> 	}
> 	return q.items[0], true
> }
> 
> func (q *Queue[T]) IsEmpty() bool { return len(q.items) == 0 }
> func (q *Queue[T]) Size() int     { return len(q.items) }
> 
> func main() {
> 	fmt.Println("=== Stack ===")
> 	s := &Stack[int]{}
> 	s.Push(1)
> 	s.Push(2)
> 	s.Push(3)
> 	fmt.Println("Push: 1, 2, 3")
> 
> 	if val, ok := s.Peek(); ok {
> 		fmt.Printf("Peek: %d\n", val)
> 	}
> 	if val, ok := s.Pop(); ok {
> 		fmt.Printf("Pop: %d\n", val)
> 	}
> 	if val, ok := s.Pop(); ok {
> 		fmt.Printf("Pop: %d\n", val)
> 	}
> 	fmt.Printf("Size: %d\n", s.Size())
> 
> 	fmt.Println("\n=== Queue ===")
> 	q := &Queue[string]{}
> 	q.Enqueue("a")
> 	q.Enqueue("b")
> 	q.Enqueue("c")
> 	fmt.Println("Enqueue: a, b, c")
> 
> 	if val, ok := q.Front(); ok {
> 		fmt.Printf("Front: %s\n", val)
> 	}
> 	if val, ok := q.Dequeue(); ok {
> 		fmt.Printf("Dequeue: %s\n", val)
> 	}
> 	if val, ok := q.Dequeue(); ok {
> 		fmt.Printf("Dequeue: %s\n", val)
> 	}
> 	fmt.Printf("Size: %d\n", q.Size())
> }
> ```

---

## P13: Merge Two Sorted Slices

Write a function `mergeSorted(a, b []int) []int` that merges two already-sorted slices into a single sorted slice in O(n+m) time.

**Expected output:**
```
A: [1 3 5 7 9]
B: [2 4 6 8 10]
Merged: [1 2 3 4 5 6 7 8 9 10]
```

> [!hint]- Hint
> Use two pointers, one for each slice. Compare the current elements. Append the smaller one and advance that pointer. When one slice is exhausted, append the remainder of the other.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func mergeSorted(a, b []int) []int {
> 	result := make([]int, 0, len(a)+len(b))
> 	i, j := 0, 0
> 
> 	for i < len(a) && j < len(b) {
> 		if a[i] <= b[j] {
> 			result = append(result, a[i])
> 			i++
> 		} else {
> 			result = append(result, b[j])
> 			j++
> 		}
> 	}
> 
> 	// Append remaining elements
> 	result = append(result, a[i:]...)
> 	result = append(result, b[j:]...)
> 
> 	return result
> }
> 
> func main() {
> 	a := []int{1, 3, 5, 7, 9}
> 	b := []int{2, 4, 6, 8, 10}
> 
> 	fmt.Println("A:", a)
> 	fmt.Println("B:", b)
> 	fmt.Println("Merged:", mergeSorted(a, b))
> }
> ```

---

## P14: Duplicates and Unique Elements

Given two slices, write functions to find:
1. `intersection(a, b []int) []int` - elements in both
2. `union(a, b []int) []int` - all unique elements from both
3. `difference(a, b []int) []int` - elements in `a` but not in `b`

**Expected output:**
```
A: [1 2 3 4 5]
B: [4 5 6 7 8]
Intersection: [4 5]
Union: [1 2 3 4 5 6 7 8]
Difference (A-B): [1 2 3]
```

> [!hint]- Hint
> Build a `map[int]bool` from one slice, then iterate the other to check membership. For union, add all elements from both slices to a set, then convert back. Sort the results for deterministic output.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sort"
> )
> 
> func intersection(a, b []int) []int {
> 	set := make(map[int]bool)
> 	for _, v := range a {
> 		set[v] = true
> 	}
> 
> 	var result []int
> 	for _, v := range b {
> 		if set[v] {
> 			result = append(result, v)
> 		}
> 	}
> 	sort.Ints(result)
> 	return result
> }
> 
> func union(a, b []int) []int {
> 	set := make(map[int]bool)
> 	for _, v := range a {
> 		set[v] = true
> 	}
> 	for _, v := range b {
> 		set[v] = true
> 	}
> 
> 	result := make([]int, 0, len(set))
> 	for v := range set {
> 		result = append(result, v)
> 	}
> 	sort.Ints(result)
> 	return result
> }
> 
> func difference(a, b []int) []int {
> 	set := make(map[int]bool)
> 	for _, v := range b {
> 		set[v] = true
> 	}
> 
> 	var result []int
> 	for _, v := range a {
> 		if !set[v] {
> 			result = append(result, v)
> 		}
> 	}
> 	sort.Ints(result)
> 	return result
> }
> 
> func main() {
> 	a := []int{1, 2, 3, 4, 5}
> 	b := []int{4, 5, 6, 7, 8}
> 
> 	fmt.Println("A:", a)
> 	fmt.Println("B:", b)
> 	fmt.Println("Intersection:", intersection(a, b))
> 	fmt.Println("Union:", union(a, b))
> 	fmt.Println("Difference (A-B):", difference(a, b))
> }
> ```

---

## P15: LRU Cache

Implement an LRU (Least Recently Used) cache with:
- `NewLRUCache(capacity int) *LRUCache`
- `Get(key string) (string, bool)` - returns value and moves to front
- `Put(key, value string)` - adds/updates and evicts oldest if at capacity

Use a `map` for O(1) lookups and a slice (or `container/list`) to track access order.

**Expected output:**
```
Put a=1, b=2, c=3 (capacity 2)
Get b: 2 (found: true)
Get a: (found: false) -- evicted when c was added
Put d=4 -- evicts c (least recently used, b was accessed after c)
Get c: (found: false)
Get b: 2 (found: true)
Get d: 4 (found: true)
```

> [!hint]- Hint
> Use `container/list` for a doubly linked list. Each list element stores a key-value pair. The map maps keys to list elements for O(1) access. On `Get` or `Put`, move the element to the front. On eviction, remove from the back.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"container/list"
> 	"fmt"
> )
> 
> type entry struct {
> 	key   string
> 	value string
> }
> 
> type LRUCache struct {
> 	capacity int
> 	items    map[string]*list.Element
> 	order    *list.List // front = most recent, back = least recent
> }
> 
> func NewLRUCache(capacity int) *LRUCache {
> 	return &LRUCache{
> 		capacity: capacity,
> 		items:    make(map[string]*list.Element),
> 		order:    list.New(),
> 	}
> }
> 
> func (c *LRUCache) Get(key string) (string, bool) {
> 	if elem, ok := c.items[key]; ok {
> 		c.order.MoveToFront(elem)
> 		return elem.Value.(*entry).value, true
> 	}
> 	return "", false
> }
> 
> func (c *LRUCache) Put(key, value string) {
> 	// Update existing
> 	if elem, ok := c.items[key]; ok {
> 		c.order.MoveToFront(elem)
> 		elem.Value.(*entry).value = value
> 		return
> 	}
> 
> 	// Evict if at capacity
> 	if c.order.Len() >= c.capacity {
> 		oldest := c.order.Back()
> 		if oldest != nil {
> 			c.order.Remove(oldest)
> 			delete(c.items, oldest.Value.(*entry).key)
> 		}
> 	}
> 
> 	// Add new
> 	e := &entry{key: key, value: value}
> 	elem := c.order.PushFront(e)
> 	c.items[key] = elem
> }
> 
> func main() {
> 	cache := NewLRUCache(2)
> 
> 	cache.Put("a", "1")
> 	cache.Put("b", "2")
> 	cache.Put("c", "3") // evicts "a"
> 	fmt.Println("Put a=1, b=2, c=3 (capacity 2)")
> 
> 	if val, ok := cache.Get("b"); ok {
> 		fmt.Printf("Get b: %s (found: %t)\n", val, ok)
> 	}
> 
> 	if _, ok := cache.Get("a"); !ok {
> 		fmt.Println("Get a: (found: false) -- evicted when c was added")
> 	}
> 
> 	cache.Put("d", "4") // evicts "c" (b was recently accessed)
> 	fmt.Println("Put d=4 -- evicts c (least recently used, b was accessed after c)")
> 
> 	if _, ok := cache.Get("c"); !ok {
> 		fmt.Println("Get c: (found: false)")
> 	}
> 
> 	if val, ok := cache.Get("b"); ok {
> 		fmt.Printf("Get b: %s (found: %t)\n", val, ok)
> 	}
> 
> 	if val, ok := cache.Get("d"); ok {
> 		fmt.Printf("Get d: %s (found: %t)\n", val, ok)
> 	}
> }
> ```
