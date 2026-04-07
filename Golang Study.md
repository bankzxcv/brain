---
title: Golang Study
date: 2026-04-06
tags:
  - golang
  - programming
  - study
  - course
aliases:
  - Go Study
  - Go Language
  - Go Course
status: in-progress
---

# Golang Study

> [!tip] Why Go?
> Go is statically typed, compiled, and designed for simplicity and performance. Great for systems programming, CLIs, backend services, and cloud-native infrastructure.

> [!abstract] Course Structure
> **Reference** - This note covers all core concepts with deep explanations and gotchas.
> **Practice** - 12 topic drills with 12-15 progressive problems each. See [[#Practice Drills]].
> **Workflows** - 5 real-world projects combining all concepts. See [[#Workflow Projects]].

---

## 1. Variables & Types

```go
// var keyword - explicit type
var x int = 10
var s string = "hello"

// Short declaration - type inferred (only inside functions)
y := 20
name := "Go"

// Zero values - Go ALWAYS initializes variables
var i int       // 0
var f float64   // 0.0
var b bool      // false
var str string  // ""
var p *int      // nil
var sl []int    // nil (but len=0, safe to append)

// Constants
const Pi = 3.14159
const (
    StatusOK    = 200
    StatusError = 500
)

// iota - auto-incrementing constant generator
type Weekday int
const (
    Sunday    Weekday = iota  // 0
    Monday                     // 1
    Tuesday                    // 2
    Wednesday                  // 3
    Thursday                   // 4
    Friday                     // 5
    Saturday                   // 6
)

// Type conversions (Go has NO implicit conversions)
var a int = 42
var b float64 = float64(a)   // must be explicit
var c int = int(b)

// Type aliases vs type definitions
type Celsius float64        // new type, not interchangeable
type Temperature = float64  // alias, IS interchangeable
```

> [!warning] Common Gotcha: Short Declaration Shadowing
> `:=` creates a NEW variable. Inside an `if` or `for` block, it shadows the outer variable.
> ```go
> x := 10
> if true {
>     x := 20  // This is a NEW x, outer x is still 10
>     fmt.Println(x) // 20
> }
> fmt.Println(x) // 10 (unchanged!)
> ```

> [!info] Rune vs Byte
> - `byte` = `uint8` (ASCII, single byte)
> - `rune` = `int32` (Unicode code point)
> - `string` is a sequence of bytes, NOT runes. Use `[]rune(s)` for Unicode-safe operations.
> ```go
> s := "Hello, Thai"
> fmt.Println(len(s))         // byte count (may be > character count for Thai)
> fmt.Println(len([]rune(s))) // actual character count
> ```

**Practice:** [[Golang Practice/01 - Variables and Types]]

---

## 2. Functions

```go
// Basic function
func add(a, b int) int {
    return a + b
}

// Multiple return values
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Named return values (useful for documentation, enables naked return)
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return  // naked return - returns x, y
}

// Variadic functions
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
// Call: sum(1, 2, 3) or sum(slice...)

// First-class functions (functions are values)
func apply(f func(int, int) int, a, b int) int {
    return f(a, b)
}

// Closures - capture surrounding variables
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}
// c := counter(); c() -> 1; c() -> 2; c() -> 3

// Function types
type Transformer func(string) string

func process(s string, fn Transformer) string {
    return fn(s)
}

// init() - runs automatically before main, once per package
func init() {
    // setup code, no args, no return
}
```

> [!warning] Common Gotcha: Closure in Loops
> ```go
> // BUG: all goroutines capture the SAME variable i
> for i := 0; i < 5; i++ {
>     go func() {
>         fmt.Println(i)  // likely prints 5,5,5,5,5
>     }()
> }
>
> // FIX: pass i as argument
> for i := 0; i < 5; i++ {
>     go func(id int) {
>         fmt.Println(id)  // prints 0,1,2,3,4 (in any order)
>     }(i)
> }
> ```

**Practice:** [[Golang Practice/02 - Functions]]

---

## 3. Pointers

```go
// Pointer holds the memory address of a value
var x int = 42
var p *int = &x     // p points to x
fmt.Println(*p)     // 42 (dereference: read value at address)
*p = 100            // modify x through pointer
fmt.Println(x)      // 100

// new() allocates memory and returns pointer
ptr := new(int)     // *int pointing to zero-valued int
*ptr = 42

// Pointer to struct - Go auto-dereferences
type Point struct { X, Y int }
p := &Point{1, 2}
p.X = 10           // same as (*p).X = 10

// Pointers for mutation
func double(n *int) {
    *n *= 2
}
x := 5
double(&x)  // x is now 10

// When to use pointers:
// 1. Need to mutate the original value
// 2. Large struct (avoid copying)
// 3. Need to represent "absence" (nil)
// 4. Implementing interfaces with pointer receivers
```

> [!tip] Value vs Pointer Semantics
> - **Value types** (int, string, struct): copied on assignment/pass
> - **Reference types** (slice, map, channel): contain internal pointers, "cheap" to copy
> - Use pointer receivers when: mutating state, large struct, consistency (if one method needs pointer, use pointer for all)
> - Use value receivers when: small struct, read-only, immutable semantics

> [!warning] No Pointer Arithmetic in Go
> Unlike C/C++, Go does NOT allow pointer arithmetic. You cannot do `p++` or `p + 4`. This is by design for safety. Use `unsafe.Pointer` only as a last resort.

**Practice:** [[Golang Practice/03 - Pointers]]

---

## 4. Structs & Methods

```go
// Struct definition
type User struct {
    ID        int
    Name      string
    Email     string
    Active    bool
    CreatedAt time.Time
}

// Initialization
u1 := User{ID: 1, Name: "Alice"}         // named fields (preferred)
u2 := User{1, "Bob", "bob@x.com", true, time.Now()}  // positional (fragile)
u3 := new(User)                           // pointer to zero-valued User

// Value receiver (read-only, works on copy)
func (u User) FullInfo() string {
    return fmt.Sprintf("%s (%s)", u.Name, u.Email)
}

// Pointer receiver (can mutate, works on original)
func (u *User) Deactivate() {
    u.Active = false
}

// Embedding (composition over inheritance)
type Employee struct {
    User           // embedded - Employee "inherits" User's fields and methods
    Department string
    Salary     float64
}

e := Employee{
    User:       User{ID: 1, Name: "Alice"},
    Department: "Engineering",
    Salary:     100000,
}
e.Name            // promoted from User
e.FullInfo()      // promoted from User
e.User.Name       // explicit access also works

// Method set rules:
// - Value of type T:      can call methods with value receiver
// - Pointer to type T:    can call methods with BOTH value and pointer receivers
// - Interface satisfaction requires matching receiver type

// Struct tags (metadata for serialization)
type Config struct {
    Host    string `json:"host" yaml:"host" validate:"required"`
    Port    int    `json:"port" yaml:"port" validate:"min=1,max=65535"`
    Debug   bool   `json:"debug,omitempty"`
}

// Anonymous struct (one-off, no reuse)
point := struct{ X, Y int }{10, 20}

// Constructor pattern (Go has no constructors)
func NewUser(name, email string) *User {
    return &User{
        Name:      name,
        Email:     email,
        Active:    true,
        CreatedAt: time.Now(),
    }
}
```

> [!tip] Embedding vs Fields
> - Embedding (`User`) promotes all fields and methods - use for "is-a" or "has-a" with delegation
> - Named field (`user User`) requires explicit access via `e.user.Name` - use when you want encapsulation

**Practice:** [[Golang Practice/04 - Structs and Methods]]

---

## 5. Interfaces

```go
// Interface = set of method signatures
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Interface composition
type ReadWriter interface {
    Reader
    Writer
}

// Implicit implementation (no "implements" keyword!)
type MyBuffer struct {
    data []byte
}

func (b *MyBuffer) Read(p []byte) (int, error) {
    n := copy(p, b.data)
    return n, nil
}
// MyBuffer now satisfies Reader interface automatically

// Empty interface (accepts ANY type)
var anything interface{} = 42
anything = "hello"
anything = []int{1, 2, 3}

// any is an alias for interface{} (Go 1.18+)
var x any = "hello"

// Type assertion
val, ok := anything.(string)
if ok {
    fmt.Println("It's a string:", val)
}

// Type switch
func describe(i interface{}) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("int: %d", v)
    case string:
        return fmt.Sprintf("string: %q", v)
    case bool:
        return fmt.Sprintf("bool: %t", v)
    default:
        return fmt.Sprintf("unknown: %v", v)
    }
}

// Common standard library interfaces:
// fmt.Stringer    -> String() string
// error           -> Error() string
// io.Reader       -> Read([]byte) (int, error)
// io.Writer       -> Write([]byte) (int, error)
// io.Closer       -> Close() error
// sort.Interface  -> Len(), Less(i,j), Swap(i,j)
```

> [!warning] Interface Gotcha: nil vs nil interface
> ```go
> var p *MyBuffer = nil          // typed nil pointer
> var r Reader = p               // r is NOT nil!
> fmt.Println(r == nil)          // false (interface has type info but nil value)
>
> var r2 Reader = nil            // this IS nil
> fmt.Println(r2 == nil)         // true
> ```
> An interface is nil only when BOTH its type and value are nil.

> [!tip] Interface Design Principles
> - **Accept interfaces, return structs** - makes your code flexible for callers
> - **Keep interfaces small** - `io.Reader` has 1 method, that's perfect
> - **Define interfaces where they're used**, not where they're implemented
> - **Don't create interfaces for the sake of it** - if there's only one implementation, you probably don't need it yet

**Practice:** [[Golang Practice/05 - Interfaces]]

---

## 6. Slices & Maps

### Slices

```go
// Slice = dynamic view into an array (pointer + length + capacity)
s := []int{1, 2, 3, 4, 5}
fmt.Println(len(s), cap(s))  // 5, 5

// Make with length and capacity
s2 := make([]int, 3, 10)  // len=3, cap=10, values=[0,0,0]

// Append (may allocate new underlying array if cap exceeded)
s = append(s, 6, 7)
s = append(s, []int{8, 9}...)  // append another slice

// Slicing (creates a NEW slice sharing same array)
sub := s[1:4]   // elements at index 1,2,3
sub2 := s[:3]   // first 3
sub3 := s[2:]   // from index 2 to end

// Full slice expression (controls capacity)
sub4 := s[1:4:4]  // len=3, cap=3 (prevents accidental overwrite)

// Copy (independent copy)
dst := make([]int, len(s))
copy(dst, s)

// Delete element at index i (order preserved)
s = append(s[:i], s[i+1:]...)

// Delete element at index i (order NOT preserved, faster)
s[i] = s[len(s)-1]
s = s[:len(s)-1]

// Filter pattern
func filter(s []int, fn func(int) bool) []int {
    result := make([]int, 0, len(s))
    for _, v := range s {
        if fn(v) {
            result = append(result, v)
        }
    }
    return result
}

// slices package (Go 1.21+)
import "slices"
slices.Sort(s)
slices.Contains(s, 42)
slices.Index(s, 42)
slices.Reverse(s)
```

### Maps

```go
// Create
m := map[string]int{"a": 1, "b": 2}
m2 := make(map[string]int, 100)  // hint: preallocate for ~100 entries

// CRUD
m["c"] = 3               // create/update
val := m["a"]             // read (returns zero value if missing)
val, ok := m["x"]         // comma ok pattern
delete(m, "b")            // delete
clear(m)                  // clear all entries (Go 1.21+)

// Iteration (order is NOT guaranteed!)
for k, v := range m {
    fmt.Println(k, v)
}

// Map of slices (common pattern)
graph := make(map[string][]string)
graph["A"] = append(graph["A"], "B", "C")

// Set pattern (map with empty struct)
type Set map[string]struct{}
s := make(Set)
s["item"] = struct{}{}
_, exists := s["item"]

// maps package (Go 1.21+)
import "maps"
keys := maps.Keys(m)
values := maps.Values(m)
maps.Equal(m1, m2)
```

> [!warning] Slice Gotcha: Shared Underlying Array
> ```go
> original := []int{1, 2, 3, 4, 5}
> slice := original[1:3]  // [2, 3], shares memory!
> slice[0] = 99           // original is now [1, 99, 3, 4, 5]
> ```
> Use `copy()` or full slice expression `original[1:3:3]` to prevent this.

> [!warning] Map Gotcha: Concurrent Access
> Maps are NOT safe for concurrent read/write. Use `sync.Mutex` or `sync.Map`.

**Practice:** [[Golang Practice/06 - Slices and Maps]]

---

## 7. Error Handling

```go
// Errors are values (not exceptions!)
result, err := someFunction()
if err != nil {
    return fmt.Errorf("context: %w", err)  // wrap with %w
}

// Creating errors
err1 := errors.New("something failed")
err2 := fmt.Errorf("failed to process %s: %w", name, err1)

// Custom error type
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

// Sentinel errors (predefined, checked by callers)
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrTimeout      = errors.New("operation timed out")
)

// Error wrapping chain (Go 1.13+)
err := fmt.Errorf("save user: %w", fmt.Errorf("db insert: %w", ErrTimeout))
// Error message: "save user: db insert: operation timed out"

// errors.Is - check if any error in chain matches
if errors.Is(err, ErrTimeout) {
    // handle timeout (checks entire chain)
}

// errors.As - extract specific error type from chain
var valErr *ValidationError
if errors.As(err, &valErr) {
    fmt.Println("field:", valErr.Field)
}

// Multiple errors (Go 1.20+)
err := errors.Join(err1, err2, err3)

// Error handling patterns
// 1. Return early
func process() error {
    data, err := fetch()
    if err != nil {
        return fmt.Errorf("fetch: %w", err)
    }
    result, err := transform(data)
    if err != nil {
        return fmt.Errorf("transform: %w", err)
    }
    return save(result)
}

// 2. Must pattern (panic on error, for init/setup only)
func mustParseURL(raw string) *url.URL {
    u, err := url.Parse(raw)
    if err != nil {
        panic(fmt.Sprintf("invalid URL %q: %v", raw, err))
    }
    return u
}
```

> [!tip] Error Handling Best Practices
> - Always add context when wrapping: `fmt.Errorf("doing X: %w", err)`
> - Use `%w` (not `%v`) to preserve the error chain
> - Use sentinel errors for expected conditions (`ErrNotFound`)
> - Use custom types for errors that carry extra data
> - Never ignore errors. At minimum: `_ = f.Close()`
> - Check with `errors.Is`/`errors.As`, never compare error strings

**Practice:** [[Golang Practice/07 - Error Handling]]

---

## 8. Defer, Panic, Recover

```go
// Defer - schedules function call to run when enclosing function returns
// Runs in LIFO (last-in, first-out) order
func example() {
    defer fmt.Println("3rd - last defer runs first among defers, but after function body")
    defer fmt.Println("2nd")
    defer fmt.Println("1st")
    fmt.Println("function body")
}
// Output: function body, 1st, 2nd, 3rd

// Defer evaluates arguments immediately (but executes later)
x := 10
defer fmt.Println(x)  // will print 10, even if x changes later
x = 20

// Common pattern: resource cleanup
func readFile(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()  // guaranteed cleanup
    return io.ReadAll(f)
}

// Defer with mutex
func (s *SafeMap) Get(key string) string {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.data[key]
}

// Panic - for unrecoverable errors (like throwing an exception)
func mustDivide(a, b int) int {
    if b == 0 {
        panic("division by zero")  // stops normal execution
    }
    return a / b
}

// Recover - catches panics (only works inside defer)
func safeOperation() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recovered: %v", r)
        }
    }()
    riskyCode()
    return nil
}

// When to use:
// defer  -> ALWAYS for cleanup (files, locks, connections)
// panic  -> RARELY (truly unrecoverable: corrupted state, programmer error)
// recover -> middleware/boundaries (HTTP handlers, goroutine top-level)
```

> [!warning] Defer Gotcha: Loop Leaks
> ```go
> // BAD: all files stay open until function returns
> for _, path := range paths {
>     f, _ := os.Open(path)
>     defer f.Close()  // won't close until function ends!
> }
>
> // GOOD: wrap in closure
> for _, path := range paths {
>     func() {
>         f, _ := os.Open(path)
>         defer f.Close()
>         // process file
>     }()
> }
> ```

**Practice:** [[Golang Practice/08 - Defer Panic Recover]]

---

## 9. Goroutines & Channels

```go
// Goroutine = lightweight thread managed by Go runtime (~2KB stack)
go doWork()                     // fire-and-forget
go func() { /* inline */ }()   // anonymous goroutine

// Channel = typed conduit for communication between goroutines
ch := make(chan int)        // unbuffered (synchronous)
bch := make(chan int, 10)   // buffered (async up to capacity)

// Send and receive
ch <- 42        // send (blocks until receiver ready on unbuffered)
val := <-ch     // receive (blocks until value available)

// Directional channels (for function signatures)
func producer(out chan<- int) { out <- 1 }   // send-only
func consumer(in <-chan int)  { <-in }        // receive-only

// Close a channel (signals "no more values")
close(ch)

// Range over channel (reads until closed)
for val := range ch {
    fmt.Println(val)
}

// Check if channel is closed
val, ok := <-ch  // ok is false if closed and empty

// Select - multiplexes channels
select {
case msg := <-ch1:
    fmt.Println("from ch1:", msg)
case msg := <-ch2:
    fmt.Println("from ch2:", msg)
case ch3 <- value:
    fmt.Println("sent to ch3")
case <-time.After(5 * time.Second):
    fmt.Println("timeout")
default:
    fmt.Println("no channel ready")
}

// Done channel pattern
func worker(done chan struct{}) {
    defer close(done)
    // do work...
}
done := make(chan struct{})
go worker(done)
<-done  // blocks until worker finishes
```

> [!danger] Goroutine Leaks
> A goroutine that blocks forever (waiting on a channel nobody sends to) is a **memory leak**. Always ensure goroutines can exit:
> - Use `context.Context` with cancellation
> - Use `done` channels
> - Use buffered channels to prevent blocking on send

> [!tip] Channel Axioms
> | Operation | nil channel | closed channel | normal channel |
> |-----------|-------------|----------------|----------------|
> | send | blocks forever | **panic** | blocks/succeeds |
> | receive | blocks forever | returns zero | blocks/succeeds |
> | close | **panic** | **panic** | succeeds |

**Practice:** [[Golang Practice/09 - Goroutines and Channels]]

---

## 10. Concurrency Patterns

```go
// WaitGroup - wait for N goroutines to finish
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Println("worker", id)
    }(i)
}
wg.Wait()

// Mutex - protect shared state
type SafeCounter struct {
    mu sync.RWMutex
    v  map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.v[key]++
}

func (c *SafeCounter) Get(key string) int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.v[key]
}

// Worker pool pattern
func workerPool(jobs <-chan int, results chan<- int, numWorkers int) {
    var wg sync.WaitGroup
    for w := 0; w < numWorkers; w++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    wg.Wait()
    close(results)
}

// Fan-out, fan-in
func fanOut(input <-chan int, n int) []<-chan int {
    outputs := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        outputs[i] = worker(input)
    }
    return outputs
}

func fanIn(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int)
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                merged <- v
            }
        }(ch)
    }
    go func() {
        wg.Wait()
        close(merged)
    }()
    return merged
}

// Pipeline pattern
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}
// Usage: result := square(square(gen(1, 2, 3)))

// Semaphore pattern (limit concurrency)
sem := make(chan struct{}, 3)  // max 3 concurrent
for _, task := range tasks {
    sem <- struct{}{}  // acquire
    go func(t Task) {
        defer func() { <-sem }()  // release
        process(t)
    }(task)
}

// Context for cancellation
func longTask(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // do work
        }
    }
}

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
err := longTask(ctx)

// errgroup - WaitGroup + error handling
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(context.Background())
for _, url := range urls {
    url := url
    g.Go(func() error {
        return fetch(ctx, url)
    })
}
if err := g.Wait(); err != nil {
    log.Fatal(err)
}

// sync.Once - execute exactly once
var once sync.Once
var instance *DB

func GetDB() *DB {
    once.Do(func() {
        instance = connectDB()
    })
    return instance
}
```

**Practice:** [[Golang Practice/10 - Concurrency Patterns]]

---

## 11. Generics (Go 1.18+)

```go
// Generic function
func Map[T any, R any](s []T, fn func(T) R) []R {
    result := make([]R, len(s))
    for i, v := range s {
        result[i] = fn(v)
    }
    return result
}
// Usage: Map([]int{1,2,3}, func(n int) string { return strconv.Itoa(n) })

// Type constraints
type Number interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~float32 | ~float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

// ~ means "underlying type" (includes type definitions)
type MyInt int
Sum([]MyInt{1, 2, 3})  // works because of ~int

// comparable constraint (supports == and !=)
func Contains[T comparable](s []T, target T) bool {
    for _, v := range s {
        if v == target {
            return true
        }
    }
    return false
}

// Generic struct
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

// Ordered constraint (from golang.org/x/exp/constraints or cmp)
import "cmp"

func Max[T cmp.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

// Multiple type parameters with constraints
func Zip[A, B any](as []A, bs []B) []struct{ First A; Second B } {
    n := min(len(as), len(bs))
    result := make([]struct{ First A; Second B }, n)
    for i := 0; i < n; i++ {
        result[i] = struct{ First A; Second B }{as[i], bs[i]}
    }
    return result
}
```

> [!tip] When to Use Generics
> - Collection operations (map, filter, reduce, contains)
> - Data structures (stack, queue, tree, linked list)
> - Utility functions that work on multiple types
> - **Don't** use when `interface{}` or a specific type works fine
> - **Don't** over-abstract: if only 2 types, maybe just write 2 functions

**Practice:** [[Golang Practice/11 - Generics]]

---

## 12. Testing

```go
// File: math_test.go (must end in _test.go)
package math

import "testing"

// Basic test
func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2, 3) = %d, want 5", result)
    }
}

// Table-driven tests (idiomatic Go)
func TestAdd_Table(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
        {"mixed", -1, 5, 4},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.expected {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.expected)
            }
        })
    }
}

// Subtests with setup/teardown
func TestDatabase(t *testing.T) {
    db := setupTestDB(t)
    t.Cleanup(func() { db.Close() })

    t.Run("Insert", func(t *testing.T) { /* ... */ })
    t.Run("Query", func(t *testing.T) { /* ... */ })
}

// Benchmarks
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}

// Test helpers
func assertEqual[T comparable](t *testing.T, got, want T) {
    t.Helper()  // marks this as helper (error points to caller)
    if got != want {
        t.Errorf("got %v, want %v", got, want)
    }
}

// Testable examples (also shown in docs)
func ExampleAdd() {
    fmt.Println(Add(2, 3))
    // Output: 5
}

// TestMain - setup/teardown for entire package
func TestMain(m *testing.M) {
    setup()
    code := m.Run()
    teardown()
    os.Exit(code)
}

// httptest for HTTP handlers
func TestHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/api/user", nil)
    w := httptest.NewRecorder()
    handler(w, req)
    if w.Code != 200 {
        t.Errorf("status = %d, want 200", w.Code)
    }
}
```

```bash
go test ./...                  # run all tests
go test -v ./...               # verbose
go test -run TestAdd ./...     # run specific test
go test -count=1 ./...         # disable cache
go test -race ./...            # race detector
go test -cover ./...           # coverage
go test -bench=. ./...         # benchmarks
go test -benchmem ./...        # benchmark with memory stats
```

**Practice:** [[Golang Practice/12 - Testing]]

---

## Standard Library Highlights

| Package | Use | Key Functions |
|---|---|---|
| `fmt` | Formatted I/O | `Printf`, `Sprintf`, `Fprintf`, `Errorf` |
| `os` | OS interaction | `Open`, `Create`, `Getenv`, `Args`, `Exit` |
| `io` | I/O primitives | `Copy`, `ReadAll`, `TeeReader`, `Pipe` |
| `bufio` | Buffered I/O | `Scanner`, `NewReader`, `NewWriter` |
| `net/http` | HTTP client/server | `ListenAndServe`, `HandleFunc`, `Get` |
| `encoding/json` | JSON encode/decode | `Marshal`, `Unmarshal`, `NewDecoder` |
| `sync` | Synchronization | `Mutex`, `RWMutex`, `WaitGroup`, `Once`, `Map` |
| `context` | Cancellation/deadlines | `WithTimeout`, `WithCancel`, `WithValue` |
| `strings` | String manipulation | `Contains`, `Split`, `Join`, `Replace`, `Builder` |
| `strconv` | String conversions | `Itoa`, `Atoi`, `ParseFloat`, `FormatBool` |
| `time` | Time and duration | `Now`, `Since`, `NewTicker`, `After`, `Sleep` |
| `log/slog` | Structured logging | `Info`, `Error`, `With`, `NewJSONHandler` |
| `testing` | Unit tests | `T`, `B`, `Run`, `Cleanup`, `Helper` |
| `sort` | Sorting | `Slice`, `SliceStable`, `Search` |
| `regexp` | Regular expressions | `Compile`, `MatchString`, `FindAllString` |
| `path/filepath` | File paths | `Join`, `Walk`, `Glob`, `Ext` |
| `flag` | CLI flags | `String`, `Int`, `Bool`, `Parse` |
| `embed` | Embed files in binary | `//go:embed` directive |

---

## Tools & Commands

```bash
# Module management
go mod init <module-name>   # init module
go mod tidy                 # clean up go.mod/go.sum
go get <package>@latest     # add/update dependency
go get <package>@v1.2.3     # specific version

# Build & run
go run main.go              # compile and run
go run .                    # run package in current dir
go build ./...              # build all packages
go build -o myapp .         # build with custom output name
go install ./...            # build and install to $GOPATH/bin

# Quality
go test ./...               # test
go test -race ./...         # test with race detector
go vet ./...                # static analysis
go fmt ./...                # format (prefer gofmt -w .)
golangci-lint run           # comprehensive linter (external)

# Debugging & profiling
go test -bench=. -benchmem  # benchmarks with memory
go test -cpuprofile=cpu.out # CPU profiling
go tool pprof cpu.out       # analyze profile
GODEBUG=gctrace=1 ./myapp  # GC tracing

# Documentation
go doc fmt.Println          # view docs
godoc -http=:6060           # local doc server
```

---

## Practice Drills

> [!example] Asian-Style Progressive Practice
> Each topic has 12-15 problems. Problems build on each other - each one adds a new concept or twist. Write each solution from scratch (no copy-paste!) to build muscle memory.

| # | Topic | Link | Problems |
|---|---|---|---|
| 01 | Variables & Types | [[Golang Practice/01 - Variables and Types]] | 13 |
| 02 | Functions | [[Golang Practice/02 - Functions]] | 14 |
| 03 | Pointers | [[Golang Practice/03 - Pointers]] | 13 |
| 04 | Structs & Methods | [[Golang Practice/04 - Structs and Methods]] | 14 |
| 05 | Interfaces | [[Golang Practice/05 - Interfaces]] | 13 |
| 06 | Slices & Maps | [[Golang Practice/06 - Slices and Maps]] | 15 |
| 07 | Error Handling | [[Golang Practice/07 - Error Handling]] | 13 |
| 08 | Defer, Panic, Recover | [[Golang Practice/08 - Defer Panic Recover]] | 12 |
| 09 | Goroutines & Channels | [[Golang Practice/09 - Goroutines and Channels]] | 15 |
| 10 | Concurrency Patterns | [[Golang Practice/10 - Concurrency Patterns]] | 14 |
| 11 | Generics | [[Golang Practice/11 - Generics]] | 13 |
| 12 | Testing | [[Golang Practice/12 - Testing]] | 13 |

---

## Workflow Projects

> [!success] After finishing all practice drills, implement these real-world projects to cement your knowledge. Each project combines multiple concepts.

| # | Project | Link | Concepts Used |
|---|---|---|---|
| 01 | CLI Task Manager | [[Golang Workflows/01 - CLI Task Manager]] | structs, slices, JSON, file I/O, error handling, flags |
| 02 | REST API Server | [[Golang Workflows/02 - REST API Server]] | net/http, JSON, structs, maps, error handling, middleware |
| 03 | Concurrent File Processor | [[Golang Workflows/03 - Concurrent File Processor]] | goroutines, channels, WaitGroup, worker pool, file I/O |
| 04 | Web Scraper Pipeline | [[Golang Workflows/04 - Web Scraper Pipeline]] | goroutines, channels, pipeline, context, HTTP client |
| 05 | TCP Chat Server | [[Golang Workflows/05 - TCP Chat Server]] | net, goroutines, channels, maps, mutex, select |

---

## Resources

- [Official Go Tour](https://go.dev/tour/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go by Example](https://gobyexample.com)
- [pkg.go.dev](https://pkg.go.dev) - standard library docs
- [Go Proverbs](https://go-proverbs.github.io/)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- [100 Go Mistakes](https://100go.co/)
