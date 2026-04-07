---
title: "Go Practice: Pointers"
date: 2026-04-07
tags:
  - golang
  - practice
  - pointers
parent: "[[Golang Study]]"
---

# Go Practice: Pointers

13 progressive problems. Write each solution from scratch to build muscle memory.

---

## P1: Create, Dereference, and Modify Through a Pointer

Declare an `int` variable `x := 10`. Create a pointer `p` to `x`. Print the pointer address, the dereferenced value, modify `x` through `p`, and print `x` again.

**Expected output (address will vary):**
```
Address: 0xc0000b6010
Value via pointer: 10
After modification: 20
```

> [!hint]- Hint
> `p := &x` creates a pointer. `*p` dereferences it. `*p = 20` modifies `x` through the pointer.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	x := 10
> 	p := &x
> 
> 	fmt.Printf("Address: %p\n", p)
> 	fmt.Printf("Value via pointer: %d\n", *p)
> 
> 	*p = 20
> 	fmt.Printf("After modification: %d\n", x)
> }
> ```

---

## P2: Swap Two Integers Using Pointers

Write `swap(a, b *int)` that swaps the values of two integers using pointers. Demonstrate that the original variables are swapped after the call.

**Expected output:**
```
Before: a=10, b=20
After:  a=20, b=10
```

> [!hint]- Hint
> Inside the function, dereference both pointers and use a temporary variable: `*a, *b = *b, *a` or the classic three-step swap.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func swap(a, b *int) {
> 	*a, *b = *b, *a
> }
> 
> func main() {
> 	a, b := 10, 20
> 	fmt.Printf("Before: a=%d, b=%d\n", a, b)
> 
> 	swap(&a, &b)
> 	fmt.Printf("After:  a=%d, b=%d\n", a, b)
> }
> ```

---

## P3: Nil Pointer Safe Dereference

Write `safeDeref(p *int) int` that returns the value pointed to by `p`, or `0` if `p` is nil. Test with both a valid pointer and a nil pointer.

**Expected output:**
```
safeDeref(&42) = 42
safeDeref(nil) = 0
```

> [!hint]- Hint
> Check `if p == nil` before dereferencing. This is a common pattern in Go to avoid panics.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func safeDeref(p *int) int {
> 	if p == nil {
> 		return 0
> 	}
> 	return *p
> }
> 
> func main() {
> 	x := 42
> 	fmt.Printf("safeDeref(&42) = %d\n", safeDeref(&x))
> 	fmt.Printf("safeDeref(nil) = %d\n", safeDeref(nil))
> }
> ```

---

## P4: Pointer to Struct

Define a `Person` struct with `Name` and `Age` fields. Write a function `birthday(p *Person)` that increments the person's age. Demonstrate that the original struct is modified.

**Expected output:**
```
Before: {Alice 30}
After:  {Alice 31}
```

> [!hint]- Hint
> With a struct pointer, you can access fields directly: `p.Age++` (Go automatically dereferences struct pointers). This is syntactic sugar for `(*p).Age++`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Person struct {
> 	Name string
> 	Age  int
> }
> 
> func birthday(p *Person) {
> 	p.Age++
> }
> 
> func main() {
> 	alice := Person{Name: "Alice", Age: 30}
> 	fmt.Printf("Before: %+v\n", alice)
> 
> 	birthday(&alice)
> 	fmt.Printf("After:  %+v\n", alice)
> }
> ```

---

## P5: new() vs &T{}

Create a `Config` struct with `Host string` and `Port int`. Allocate one instance using `new()` and another using `&Config{}`. Set their fields and print both. Explain the difference (or lack thereof).

**Expected output:**
```
c1: &{localhost 8080}
c2: &{0.0.0.0 9090}
```

> [!hint]- Hint
> `new(Config)` returns `*Config` with zero values. `&Config{Host: "0.0.0.0", Port: 9090}` returns `*Config` with the fields initialized. Both give you a pointer to a heap-allocated struct.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Config struct {
> 	Host string
> 	Port int
> }
> 
> func main() {
> 	// Using new() - returns pointer with zero values
> 	c1 := new(Config)
> 	c1.Host = "localhost"
> 	c1.Port = 8080
> 	fmt.Printf("c1: %+v\n", c1)
> 
> 	// Using &T{} - returns pointer with initialized values
> 	c2 := &Config{Host: "0.0.0.0", Port: 9090}
> 	fmt.Printf("c2: %+v\n", c2)
> 
> 	// Both are *Config — functionally equivalent.
> 	// &T{} is preferred when you want to set initial values.
> 	// new() is rarely used in practice.
> }
> ```

---

## P6: Method with Pointer Receiver

Define a `Counter` struct with a `count int` field. Add methods `Increment()` (pointer receiver) and `Value()` (value receiver). Show that `Increment` modifies the original while demonstrating the difference.

**Expected output:**
```
Initial: 0
After 3 increments: 3
After 2 more: 5
```

> [!hint]- Hint
> Pointer receiver: `func (c *Counter) Increment()`. Value receiver: `func (c Counter) Value() int`. The pointer receiver can modify the struct; the value receiver works on a copy.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Counter struct {
> 	count int
> }
> 
> func (c *Counter) Increment() {
> 	c.count++
> }
> 
> func (c Counter) Value() int {
> 	return c.count
> }
> 
> func main() {
> 	c := Counter{}
> 	fmt.Printf("Initial: %d\n", c.Value())
> 
> 	c.Increment()
> 	c.Increment()
> 	c.Increment()
> 	fmt.Printf("After 3 increments: %d\n", c.Value())
> 
> 	c.Increment()
> 	c.Increment()
> 	fmt.Printf("After 2 more: %d\n", c.Value())
> }
> ```

---

## P7: Linked List - Create Nodes, Traverse, Print

Define a `Node` struct with `Value int` and `Next *Node`. Create a linked list of 3 nodes (1 -> 2 -> 3). Write a function that traverses the list and prints each value.

**Expected output:**
```
1 -> 2 -> 3 -> nil
```

> [!hint]- Hint
> Build from the tail: `n3 := &Node{Value: 3}`, `n2 := &Node{Value: 2, Next: n3}`, `n1 := &Node{Value: 1, Next: n2}`. Traverse with `for current != nil`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Node struct {
> 	Value int
> 	Next  *Node
> }
> 
> func printList(head *Node) {
> 	current := head
> 	for current != nil {
> 		fmt.Printf("%d -> ", current.Value)
> 		current = current.Next
> 	}
> 	fmt.Println("nil")
> }
> 
> func main() {
> 	n3 := &Node{Value: 3}
> 	n2 := &Node{Value: 2, Next: n3}
> 	n1 := &Node{Value: 1, Next: n2}
> 
> 	printList(n1)
> }
> ```

---

## P8: Slices and Pointers - Why Append Works

Write a program that demonstrates:
1. Passing a slice to a function that modifies existing elements (works without pointer)
2. Passing a slice to a function that appends (the caller does NOT see the append)
3. Passing a `*[]int` to fix the append problem

Explain why in comments.

**Expected output:**
```
After modify: [100 2 3]
After appendBroken: [1 2 3]
After appendFixed: [1 2 3 4]
```

> [!hint]- Hint
> A slice is a struct with (pointer, length, capacity). Passing by value copies the struct but both copies point to the same underlying array. Modifying elements works, but `append` may change the length/pointer of the copy without affecting the original.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> // Modifying elements works because both the original and copy
> // share the same underlying array.
> func modify(s []int) {
> 	s[0] = 100
> }
> 
> // Append does NOT affect the caller's slice because append may
> // create a new underlying array, and the new slice header
> // (with updated len/cap) is only in this function's copy.
> func appendBroken(s []int) {
> 	s = append(s, 4)
> 	_ = s // s is local; caller never sees it
> }
> 
> // Passing a pointer to the slice lets us update the caller's
> // slice header (pointer, len, cap).
> func appendFixed(s *[]int) {
> 	*s = append(*s, 4)
> }
> 
> func main() {
> 	s1 := []int{1, 2, 3}
> 	modify(s1)
> 	fmt.Println("After modify:", s1)
> 
> 	s2 := []int{1, 2, 3}
> 	appendBroken(s2)
> 	fmt.Println("After appendBroken:", s2)
> 
> 	s3 := []int{1, 2, 3}
> 	appendFixed(&s3)
> 	fmt.Println("After appendFixed:", s3)
> }
> ```

---

## P9: Double Pointer

Write a function `allocate(pp **int, value int)` that allocates a new `int`, sets it to `value`, and assigns it through the double pointer. Demonstrate that the caller's pointer variable is updated.

**Expected output:**
```
Before: p = <nil>
After:  p = 42
```

> [!hint]- Hint
> `pp` is a pointer to a pointer. `*pp` gives you the `*int` that the caller holds. Assign `*pp = &value` (but beware: `value` is a local copy, use `new(int)` instead to be safe).

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func allocate(pp **int, value int) {
> 	n := new(int)
> 	*n = value
> 	*pp = n
> }
> 
> func main() {
> 	var p *int
> 	fmt.Printf("Before: p = %v\n", p)
> 
> 	allocate(&p, 42)
> 	fmt.Printf("After:  p = %d\n", *p)
> }
> ```

---

## P10: Simple Object Pool

Build a `Pool` struct that pre-allocates a slice of `*Buffer` objects (where `Buffer` has a `Data []byte` field). Implement `Get() *Buffer` (returns a buffer from the pool or creates a new one) and `Put(b *Buffer)` (returns a buffer to the pool, resetting its data).

**Expected output:**
```
Got buffer, len=0
After write, len=5
Returned to pool
Got same buffer back, len=0
Pool size: 0
```

> [!hint]- Hint
> Use a `[]*Buffer` slice as the pool. `Get` pops from the end of the slice (or creates new). `Put` resets `Data` to `nil` or `Data[:0]` and appends back.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Buffer struct {
> 	Data []byte
> }
> 
> type Pool struct {
> 	buffers []*Buffer
> }
> 
> func NewPool(size int) *Pool {
> 	p := &Pool{
> 		buffers: make([]*Buffer, 0, size),
> 	}
> 	for i := 0; i < size; i++ {
> 		p.buffers = append(p.buffers, &Buffer{})
> 	}
> 	return p
> }
> 
> func (p *Pool) Get() *Buffer {
> 	if len(p.buffers) == 0 {
> 		return &Buffer{}
> 	}
> 	// Pop from end
> 	b := p.buffers[len(p.buffers)-1]
> 	p.buffers = p.buffers[:len(p.buffers)-1]
> 	return b
> }
> 
> func (p *Pool) Put(b *Buffer) {
> 	b.Data = b.Data[:0] // Reset length but keep capacity
> 	p.buffers = append(p.buffers, b)
> }
> 
> func (p *Pool) Size() int {
> 	return len(p.buffers)
> }
> 
> func main() {
> 	pool := NewPool(2)
> 
> 	buf := pool.Get()
> 	fmt.Printf("Got buffer, len=%d\n", len(buf.Data))
> 
> 	buf.Data = append(buf.Data, []byte("hello")...)
> 	fmt.Printf("After write, len=%d\n", len(buf.Data))
> 
> 	pool.Put(buf)
> 	fmt.Println("Returned to pool")
> 
> 	buf2 := pool.Get()
> 	fmt.Printf("Got same buffer back, len=%d\n", len(buf2.Data))
> 	fmt.Printf("Pool size: %d\n", pool.Size())
> }
> ```

---

## P11: Binary Search Tree with Insert

Define a `TreeNode` struct with `Value int`, `Left *TreeNode`, `Right *TreeNode`. Implement `Insert(root **TreeNode, value int)` that inserts into a BST. Write `InOrder(root *TreeNode)` to print the tree in sorted order.

**Expected output:**
```
Inserting: 5, 3, 7, 1, 4, 6, 8
In-order: 1 3 4 5 6 7 8
```

> [!hint]- Hint
> Use a double pointer `**TreeNode` for insert so you can assign to a nil root. If `*root == nil`, create a new node. Otherwise, recurse left or right based on value comparison.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type TreeNode struct {
> 	Value int
> 	Left  *TreeNode
> 	Right *TreeNode
> }
> 
> func Insert(root **TreeNode, value int) {
> 	if *root == nil {
> 		*root = &TreeNode{Value: value}
> 		return
> 	}
> 	if value < (*root).Value {
> 		Insert(&(*root).Left, value)
> 	} else {
> 		Insert(&(*root).Right, value)
> 	}
> }
> 
> func InOrder(root *TreeNode) {
> 	if root == nil {
> 		return
> 	}
> 	InOrder(root.Left)
> 	fmt.Printf("%d ", root.Value)
> 	InOrder(root.Right)
> }
> 
> func main() {
> 	values := []int{5, 3, 7, 1, 4, 6, 8}
> 	fmt.Printf("Inserting: ")
> 	for i, v := range values {
> 		if i > 0 {
> 			fmt.Print(", ")
> 		}
> 		fmt.Print(v)
> 	}
> 	fmt.Println()
> 
> 	var root *TreeNode
> 	for _, v := range values {
> 		Insert(&root, v)
> 	}
> 
> 	fmt.Print("In-order: ")
> 	InOrder(root)
> 	fmt.Println()
> }
> ```

---

## P12: Pointer Aliasing

Create a struct `Account` with a `Balance int` field. Create two pointers to the same account. Modify the balance through one pointer and observe through the other. Then demonstrate a subtle bug: what happens when you reassign one pointer to a new account?

**Expected output:**
```
Both point to same account:
  p1.Balance = 100
  p2.Balance = 100

Modify through p1:
  p1.Balance = 200
  p2.Balance = 200

Reassign p1 to new account:
  p1.Balance = 500
  p2.Balance = 200
```

> [!hint]- Hint
> When two pointers point to the same struct, changes through either are visible through both. But reassigning a pointer variable (`p1 = &Account{...}`) only changes what `p1` points to; `p2` still points to the original.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Account struct {
> 	Balance int
> }
> 
> func main() {
> 	acct := &Account{Balance: 100}
> 	p1 := acct
> 	p2 := acct
> 
> 	fmt.Println("Both point to same account:")
> 	fmt.Printf("  p1.Balance = %d\n", p1.Balance)
> 	fmt.Printf("  p2.Balance = %d\n", p2.Balance)
> 
> 	// Modify through p1 — visible through p2
> 	p1.Balance = 200
> 	fmt.Println("\nModify through p1:")
> 	fmt.Printf("  p1.Balance = %d\n", p1.Balance)
> 	fmt.Printf("  p2.Balance = %d\n", p2.Balance)
> 
> 	// Reassign p1 to a NEW account — p2 still points to the original
> 	p1 = &Account{Balance: 500}
> 	fmt.Println("\nReassign p1 to new account:")
> 	fmt.Printf("  p1.Balance = %d\n", p1.Balance)
> 	fmt.Printf("  p2.Balance = %d\n", p2.Balance)
> }
> ```

---

## P13: Ring Buffer Using Pointers

Implement a ring buffer (circular buffer) of fixed size using a struct with a slice, head/tail indices, and a count. Implement:
- `NewRingBuffer(size int) *RingBuffer`
- `Push(val int) bool` (returns false if full)
- `Pop() (int, bool)` (returns false if empty)
- `String() string` for debug display

**Expected output:**
```
Push 1, 2, 3: [1 2 3] (len=3)
Push 4 (full): false
Pop: 1, ok=true
Pop: 2, ok=true
State: [3] (len=1)
Push 4, 5: [3 4 5] (len=3)
Pop all: 3 4 5
Pop from empty: ok=false
```

> [!hint]- Hint
> Use modular arithmetic for wrapping: `rb.tail = (rb.tail + 1) % rb.size`. Track the count separately to distinguish full from empty. The underlying storage is a fixed-size `[]int`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> type RingBuffer struct {
> 	data []int
> 	head int // read position
> 	tail int // write position
> 	count int
> 	size int
> }
> 
> func NewRingBuffer(size int) *RingBuffer {
> 	return &RingBuffer{
> 		data: make([]int, size),
> 		size: size,
> 	}
> }
> 
> func (rb *RingBuffer) Push(val int) bool {
> 	if rb.count == rb.size {
> 		return false
> 	}
> 	rb.data[rb.tail] = val
> 	rb.tail = (rb.tail + 1) % rb.size
> 	rb.count++
> 	return true
> }
> 
> func (rb *RingBuffer) Pop() (int, bool) {
> 	if rb.count == 0 {
> 		return 0, false
> 	}
> 	val := rb.data[rb.head]
> 	rb.head = (rb.head + 1) % rb.size
> 	rb.count--
> 	return val, true
> }
> 
> func (rb *RingBuffer) Len() int {
> 	return rb.count
> }
> 
> func (rb *RingBuffer) String() string {
> 	if rb.count == 0 {
> 		return "[]"
> 	}
> 	var parts []string
> 	idx := rb.head
> 	for i := 0; i < rb.count; i++ {
> 		parts = append(parts, fmt.Sprintf("%d", rb.data[idx]))
> 		idx = (idx + 1) % rb.size
> 	}
> 	return "[" + strings.Join(parts, " ") + "]"
> }
> 
> func main() {
> 	rb := NewRingBuffer(3)
> 
> 	rb.Push(1)
> 	rb.Push(2)
> 	rb.Push(3)
> 	fmt.Printf("Push 1, 2, 3: %s (len=%d)\n", rb, rb.Len())
> 
> 	ok := rb.Push(4)
> 	fmt.Printf("Push 4 (full): %t\n", ok)
> 
> 	val, ok := rb.Pop()
> 	fmt.Printf("Pop: %d, ok=%t\n", val, ok)
> 	val, ok = rb.Pop()
> 	fmt.Printf("Pop: %d, ok=%t\n", val, ok)
> 
> 	fmt.Printf("State: %s (len=%d)\n", rb, rb.Len())
> 
> 	rb.Push(4)
> 	rb.Push(5)
> 	fmt.Printf("Push 4, 5: %s (len=%d)\n", rb, rb.Len())
> 
> 	fmt.Print("Pop all: ")
> 	for {
> 		val, ok := rb.Pop()
> 		if !ok {
> 			break
> 		}
> 		fmt.Printf("%d ", val)
> 	}
> 	fmt.Println()
> 
> 	_, ok = rb.Pop()
> 	fmt.Printf("Pop from empty: ok=%t\n", ok)
> }
> ```
