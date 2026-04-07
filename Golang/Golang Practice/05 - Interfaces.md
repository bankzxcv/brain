---
title: "Go Practice: Interfaces"
date: 2026-04-07
tags:
  - golang
  - practice
  - interfaces
parent: "[[Golang Study]]"
---

# Go Practice: Interfaces

13 progressive problems to build intuition for Go interfaces - from basic definitions to real-world patterns.

---

## P1: Shape Interface - Area

Define a `Shape` interface with an `Area() float64` method. Implement it for `Rectangle` (width, height) and `Circle` (radius).

Create one of each, store them in a `[]Shape`, and print each area.

**Expected output:**
```
Rectangle area: 24.00
Circle area: 78.54
```

> [!hint]- Hint
> A type satisfies an interface implicitly - no `implements` keyword needed. Just define the method with the right signature. Use `math.Pi` for the circle.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"math"
> )
> 
> type Shape interface {
> 	Area() float64
> }
> 
> type Rectangle struct {
> 	Width, Height float64
> }
> 
> func (r Rectangle) Area() float64 {
> 	return r.Width * r.Height
> }
> 
> type Circle struct {
> 	Radius float64
> }
> 
> func (c Circle) Area() float64 {
> 	return math.Pi * c.Radius * c.Radius
> }
> 
> func main() {
> 	shapes := []Shape{
> 		Rectangle{Width: 4, Height: 6},
> 		Circle{Radius: 5},
> 	}
> 
> 	for _, s := range shapes {
> 		switch s.(type) {
> 		case Rectangle:
> 			fmt.Printf("Rectangle area: %.2f\n", s.Area())
> 		case Circle:
> 			fmt.Printf("Circle area: %.2f\n", s.Area())
> 		}
> 	}
> }
> ```

---

## P2: Implement fmt.Stringer

Create a `Person` struct with `Name` and `Age` fields. Implement the `fmt.Stringer` interface so that `fmt.Println(person)` prints a custom format.

**Expected output:**
```
Alice (30 years old)
```

> [!hint]- Hint
> The `fmt.Stringer` interface requires a `String() string` method. When you pass a value to `fmt.Println`, it calls `String()` automatically if the type implements `Stringer`.

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
> func (p Person) String() string {
> 	return fmt.Sprintf("%s (%d years old)", p.Name, p.Age)
> }
> 
> func main() {
> 	p := Person{Name: "Alice", Age: 30}
> 	fmt.Println(p)
> }
> ```

---

## P3: Interface Composition - ReadWriter

Define `Reader` and `Writer` interfaces, each with one method. Compose them into a `ReadWriter` interface. Implement all methods on a `File` struct that reads/writes to an internal buffer.

**Expected output:**
```
Written 13 bytes
Read: Hello, World!
```

> [!hint]- Hint
> Interface composition uses embedding: `type ReadWriter interface { Reader; Writer }`. Your `File` struct can use a `[]byte` as its internal buffer. `Read` should copy from the buffer, `Write` should append to it.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Reader interface {
> 	Read(p []byte) (n int, err error)
> }
> 
> type Writer interface {
> 	Write(p []byte) (n int, err error)
> }
> 
> type ReadWriter interface {
> 	Reader
> 	Writer
> }
> 
> type File struct {
> 	data   []byte
> 	offset int
> }
> 
> func (f *File) Write(p []byte) (int, error) {
> 	f.data = append(f.data, p...)
> 	return len(p), nil
> }
> 
> func (f *File) Read(p []byte) (int, error) {
> 	if f.offset >= len(f.data) {
> 		return 0, fmt.Errorf("EOF")
> 	}
> 	n := copy(p, f.data[f.offset:])
> 	f.offset += n
> 	return n, nil
> }
> 
> func main() {
> 	var rw ReadWriter = &File{}
> 
> 	n, _ := rw.Write([]byte("Hello, World!"))
> 	fmt.Printf("Written %d bytes\n", n)
> 
> 	buf := make([]byte, 100)
> 	n, _ = rw.Read(buf)
> 	fmt.Printf("Read: %s\n", string(buf[:n]))
> }
> ```

---

## P4: Empty Interface (any)

Write a function `printType` that accepts `any` (empty interface) and prints the value's type and its value.

Call it with an int, a string, a bool, and a slice.

**Expected output:**
```
Type: int, Value: 42
Type: string, Value: hello
Type: bool, Value: true
Type: []int, Value: [1 2 3]
```

> [!hint]- Hint
> Use `fmt.Sprintf("%T", v)` to get the type name as a string, or use `reflect.TypeOf(v)`. The `any` keyword is an alias for `interface{}` since Go 1.18.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func printType(v any) {
> 	fmt.Printf("Type: %T, Value: %v\n", v, v)
> }
> 
> func main() {
> 	printType(42)
> 	printType("hello")
> 	printType(true)
> 	printType([]int{1, 2, 3})
> }
> ```

---

## P5: Type Assertion

Write a function `getLength` that takes an `any` value. If it's a `string`, return its length. If it's a `[]int`, return its length. Otherwise, return `-1`.

Use type assertions (not a type switch) with the comma-ok idiom.

**Expected output:**
```
Length of "hello": 5
Length of [1 2 3 4]: 4
Length of 42: -1
```

> [!hint]- Hint
> The comma-ok form of a type assertion is `val, ok := v.(string)`. If the assertion fails, `ok` is `false` and `val` is the zero value. This avoids panics unlike the single-value form.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func getLength(v any) int {
> 	if s, ok := v.(string); ok {
> 		return len(s)
> 	}
> 	if sl, ok := v.([]int); ok {
> 		return len(sl)
> 	}
> 	return -1
> }
> 
> func main() {
> 	fmt.Printf("Length of %q: %d\n", "hello", getLength("hello"))
> 	fmt.Printf("Length of %v: %d\n", []int{1, 2, 3, 4}, getLength([]int{1, 2, 3, 4}))
> 	fmt.Printf("Length of %v: %d\n", 42, getLength(42))
> }
> ```

---

## P6: Type Switch

Write a `describe` function that uses a type switch to return a string description for `int`, `string`, `bool`, `[]string`, and a default case.

**Expected output:**
```
42 is an integer
hello is a string
true is a boolean
[go rust] is a list of strings
3.14 is something else (float64)
```

> [!hint]- Hint
> A type switch looks like:
> ```go
> switch v := val.(type) {
> case int:
>     // v is int here
> case string:
>     // v is string here
> default:
>     // v is the original interface type
> }
> ```

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func describe(val any) string {
> 	switch v := val.(type) {
> 	case int:
> 		return fmt.Sprintf("%d is an integer", v)
> 	case string:
> 		return fmt.Sprintf("%s is a string", v)
> 	case bool:
> 		return fmt.Sprintf("%t is a boolean", v)
> 	case []string:
> 		return fmt.Sprintf("%v is a list of strings", v)
> 	default:
> 		return fmt.Sprintf("%v is something else (%T)", v, v)
> 	}
> }
> 
> func main() {
> 	fmt.Println(describe(42))
> 	fmt.Println(describe("hello"))
> 	fmt.Println(describe(true))
> 	fmt.Println(describe([]string{"go", "rust"}))
> 	fmt.Println(describe(3.14))
> }
> ```

---

## P7: Implement sort.Interface

Create a `ByAge` type (a slice of `Person` structs). Implement `sort.Interface` (`Len`, `Less`, `Swap`) so you can sort people by age.

**Expected output:**
```
Before: [{Alice 30} {Bob 25} {Charlie 35}]
After:  [{Bob 25} {Alice 30} {Charlie 35}]
```

> [!hint]- Hint
> `sort.Interface` requires three methods:
> - `Len() int`
> - `Less(i, j int) bool`
> - `Swap(i, j int)`
> 
> Then call `sort.Sort(byAge)`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sort"
> )
> 
> type Person struct {
> 	Name string
> 	Age  int
> }
> 
> type ByAge []Person
> 
> func (a ByAge) Len() int           { return len(a) }
> func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
> func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
> 
> func main() {
> 	people := []Person{
> 		{"Alice", 30},
> 		{"Bob", 25},
> 		{"Charlie", 35},
> 	}
> 
> 	fmt.Println("Before:", people)
> 	sort.Sort(ByAge(people))
> 	fmt.Println("After: ", people)
> }
> ```

---

## P8: Custom Error with the error Interface

Create an `HTTPError` struct with `StatusCode int` and `Message string` fields. Implement the `error` interface. Write a function `fetchData` that returns `*HTTPError` when the status is not 200.

**Expected output:**
```
Error: 404 - resource not found
Status code: 404
```

> [!hint]- Hint
> The `error` interface is just `Error() string`. Return `*HTTPError` as an `error` from your function. The caller can use type assertion to access the extra fields like `StatusCode`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type HTTPError struct {
> 	StatusCode int
> 	Message    string
> }
> 
> func (e *HTTPError) Error() string {
> 	return fmt.Sprintf("%d - %s", e.StatusCode, e.Message)
> }
> 
> func fetchData(url string) (string, error) {
> 	// Simulate a 404
> 	if url == "/missing" {
> 		return "", &HTTPError{StatusCode: 404, Message: "resource not found"}
> 	}
> 	return "data", nil
> }
> 
> func main() {
> 	_, err := fetchData("/missing")
> 	if err != nil {
> 		fmt.Println("Error:", err)
> 
> 		if httpErr, ok := err.(*HTTPError); ok {
> 			fmt.Println("Status code:", httpErr.StatusCode)
> 		}
> 	}
> }
> ```

---

## P9: Compile-Time Interface Satisfaction Check

Given a `Notifier` interface with a `Notify(message string) error` method, and an `EmailNotifier` struct, write a compile-time check that guarantees `EmailNotifier` implements `Notifier`.

Then intentionally comment out the `Notify` method and observe the compile error.

> [!hint]- Hint
> The idiom is:
> ```go
> var _ Notifier = (*EmailNotifier)(nil)
> ```
> This declares a package-level variable (discarded with `_`) that assigns a nil pointer of `*EmailNotifier` to a variable of type `Notifier`. If the type doesn't satisfy the interface, it fails at compile time.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Notifier interface {
> 	Notify(message string) error
> }
> 
> type EmailNotifier struct {
> 	Address string
> }
> 
> // Compile-time check: EmailNotifier must implement Notifier
> var _ Notifier = (*EmailNotifier)(nil)
> 
> func (e *EmailNotifier) Notify(message string) error {
> 	fmt.Printf("Sending email to %s: %s\n", e.Address, message)
> 	return nil
> }
> 
> func main() {
> 	n := &EmailNotifier{Address: "alice@example.com"}
> 	n.Notify("Hello!")
> }
> ```
> 
> **Try this**: Comment out the `Notify` method. You will get:
> ```
> cannot use (*EmailNotifier)(nil) (value of type *EmailNotifier)
>     as Notifier value in variable declaration:
>     *EmailNotifier does not implement Notifier (missing method Notify)
> ```

---

## P10: Polymorphism - Process a Slice of Shapes

Extend the `Shape` interface from P1 by adding a `Perimeter() float64` method. Implement `Rectangle`, `Circle`, and a new `Triangle` (three sides). Write a `printReport` function that takes `[]Shape` and prints each shape's area and perimeter.

**Expected output:**
```
Rectangle  -> Area: 24.00, Perimeter: 20.00
Circle     -> Area: 78.54, Perimeter: 31.42
Triangle   -> Area: 6.00, Perimeter: 12.00
```

> [!hint]- Hint
> For the triangle, use Heron's formula for area:
> ```
> s = (a + b + c) / 2
> area = sqrt(s * (s-a) * (s-b) * (s-c))
> ```
> To get the type name for printing, use `fmt.Sprintf("%T", shape)` or a `Name()` method.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"math"
> )
> 
> type Shape interface {
> 	Area() float64
> 	Perimeter() float64
> 	Name() string
> }
> 
> type Rectangle struct {
> 	Width, Height float64
> }
> 
> func (r Rectangle) Area() float64      { return r.Width * r.Height }
> func (r Rectangle) Perimeter() float64 { return 2 * (r.Width + r.Height) }
> func (r Rectangle) Name() string       { return "Rectangle" }
> 
> type Circle struct {
> 	Radius float64
> }
> 
> func (c Circle) Area() float64      { return math.Pi * c.Radius * c.Radius }
> func (c Circle) Perimeter() float64 { return 2 * math.Pi * c.Radius }
> func (c Circle) Name() string       { return "Circle" }
> 
> type Triangle struct {
> 	A, B, C float64
> }
> 
> func (t Triangle) Area() float64 {
> 	s := (t.A + t.B + t.C) / 2
> 	return math.Sqrt(s * (s - t.A) * (s - t.B) * (s - t.C))
> }
> 
> func (t Triangle) Perimeter() float64 { return t.A + t.B + t.C }
> func (t Triangle) Name() string       { return "Triangle" }
> 
> func printReport(shapes []Shape) {
> 	for _, s := range shapes {
> 		fmt.Printf("%-10s -> Area: %.2f, Perimeter: %.2f\n",
> 			s.Name(), s.Area(), s.Perimeter())
> 	}
> }
> 
> func main() {
> 	shapes := []Shape{
> 		Rectangle{Width: 4, Height: 6},
> 		Circle{Radius: 5},
> 		Triangle{A: 3, B: 4, C: 5},
> 	}
> 	printReport(shapes)
> }
> ```

---

## P11: Interface Embedding with Method Conflict

Define two interfaces: `Saver` with `Save() error` and `Loader` with `Load() error`. Define a `Storage` interface that embeds both and adds `Close() error`.

Implement a `Database` struct that satisfies `Storage`. Then write a function that accepts a `Saver` and another that accepts a `Storage` - show that `Database` satisfies both.

**Expected output:**
```
Saving to database: users.db
Loading from database: users.db
Closing database: users.db
---
Save-only function: Saving to database: users.db
Full storage function: Saving to database: users.db, Loading from database: users.db, Closing database: users.db
```

> [!hint]- Hint
> Embedded interfaces work like embedded structs - the methods are "promoted." A type implementing `Storage` automatically satisfies `Saver` and `Loader` individually. No conflicts here because the method sets are disjoint. Conflicts only arise when two embedded interfaces define the same method signature.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Saver interface {
> 	Save() error
> }
> 
> type Loader interface {
> 	Load() error
> }
> 
> type Storage interface {
> 	Saver
> 	Loader
> 	Close() error
> }
> 
> type Database struct {
> 	Name string
> }
> 
> func (d *Database) Save() error {
> 	fmt.Printf("Saving to database: %s\n", d.Name)
> 	return nil
> }
> 
> func (d *Database) Load() error {
> 	fmt.Printf("Loading from database: %s\n", d.Name)
> 	return nil
> }
> 
> func (d *Database) Close() error {
> 	fmt.Printf("Closing database: %s\n", d.Name)
> 	return nil
> }
> 
> func saveOnly(s Saver) {
> 	fmt.Print("Save-only function: ")
> 	s.Save()
> }
> 
> func fullStorage(s Storage) {
> 	fmt.Print("Full storage function: ")
> 	s.Save()
> 	fmt.Print(", ")
> 	s.Load()
> 	fmt.Print(", ")
> 	s.Close()
> 	fmt.Println()
> }
> 
> func main() {
> 	db := &Database{Name: "users.db"}
> 	db.Save()
> 	db.Load()
> 	db.Close()
> 
> 	fmt.Println("---")
> 	saveOnly(db)
> 	fullStorage(db)
> }
> ```

---

## P12: Plugin System with Interfaces

Design a simple plugin system. Define a `Plugin` interface with:
- `Name() string`
- `Execute(input string) (string, error)`

Implement at least 3 plugins:
- `UpperPlugin` - converts input to uppercase
- `ReversePlugin` - reverses the string
- `CountPlugin` - returns the character count

Write a `PluginManager` that registers plugins and runs them all on a given input.

**Expected output:**
```
Running plugins on input: "Hello, Go!"
[upper]   -> HELLO, GO!
[reverse] -> !oG ,olleH
[count]   -> 10 characters
```

> [!hint]- Hint
> The `PluginManager` can store plugins in a `[]Plugin` slice. The `Run` method iterates over all plugins and calls `Execute` on each. For reversing a string, convert to `[]rune` first to handle Unicode properly.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> type Plugin interface {
> 	Name() string
> 	Execute(input string) (string, error)
> }
> 
> // UpperPlugin
> type UpperPlugin struct{}
> 
> func (p UpperPlugin) Name() string { return "upper" }
> func (p UpperPlugin) Execute(input string) (string, error) {
> 	return strings.ToUpper(input), nil
> }
> 
> // ReversePlugin
> type ReversePlugin struct{}
> 
> func (p ReversePlugin) Name() string { return "reverse" }
> func (p ReversePlugin) Execute(input string) (string, error) {
> 	runes := []rune(input)
> 	for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
> 		runes[i], runes[j] = runes[j], runes[i]
> 	}
> 	return string(runes), nil
> }
> 
> // CountPlugin
> type CountPlugin struct{}
> 
> func (p CountPlugin) Name() string { return "count" }
> func (p CountPlugin) Execute(input string) (string, error) {
> 	return fmt.Sprintf("%d characters", len([]rune(input))), nil
> }
> 
> // PluginManager
> type PluginManager struct {
> 	plugins []Plugin
> }
> 
> func (pm *PluginManager) Register(p Plugin) {
> 	pm.plugins = append(pm.plugins, p)
> }
> 
> func (pm *PluginManager) Run(input string) {
> 	fmt.Printf("Running plugins on input: %q\n", input)
> 	for _, p := range pm.plugins {
> 		result, err := p.Execute(input)
> 		if err != nil {
> 			fmt.Printf("[%-8s] -> ERROR: %v\n", p.Name(), err)
> 			continue
> 		}
> 		fmt.Printf("[%-8s] -> %s\n", p.Name(), result)
> 	}
> }
> 
> func main() {
> 	pm := &PluginManager{}
> 	pm.Register(UpperPlugin{})
> 	pm.Register(ReversePlugin{})
> 	pm.Register(CountPlugin{})
> 
> 	pm.Run("Hello, Go!")
> }
> ```

---

## P13: Middleware Chain with Interfaces

Build a middleware chain for an HTTP-like handler. Define:

```go
type Handler interface {
    Handle(request string) string
}
```

Implement:
- `BaseHandler` - returns a response based on the request
- `LoggerMiddleware` - logs the request, delegates, logs the response
- `AuthMiddleware` - checks if request contains "token:", rejects otherwise
- `RateLimitMiddleware` - allows only N requests, rejects after that

Chain them: `RateLimit -> Auth -> Logger -> BaseHandler`

**Expected output:**
```
[LOG] Request: token:get-users
[LOG] Response: Users: [alice, bob, charlie]
Result: Users: [alice, bob, charlie]

[LOG] Request: get-users
Result: AUTH DENIED: missing token

[LOG] Request: token:get-status
[LOG] Response: Status: OK
Result: Status: OK

Result: RATE LIMITED: too many requests
```

> [!hint]- Hint
> Each middleware wraps another `Handler`. Use struct composition:
> ```go
> type LoggerMiddleware struct {
>     next Handler
> }
> ```
> The chain is built by wrapping from inside out:
> ```go
> chain := NewRateLimit(2, NewAuth(NewLogger(NewBase())))
> ```

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> type Handler interface {
> 	Handle(request string) string
> }
> 
> // BaseHandler
> type BaseHandler struct{}
> 
> func (h BaseHandler) Handle(request string) string {
> 	req := strings.TrimPrefix(request, "token:")
> 	switch req {
> 	case "get-users":
> 		return "Users: [alice, bob, charlie]"
> 	case "get-status":
> 		return "Status: OK"
> 	default:
> 		return "Unknown request"
> 	}
> }
> 
> // LoggerMiddleware
> type LoggerMiddleware struct {
> 	next Handler
> }
> 
> func NewLogger(next Handler) *LoggerMiddleware {
> 	return &LoggerMiddleware{next: next}
> }
> 
> func (l *LoggerMiddleware) Handle(request string) string {
> 	fmt.Printf("[LOG] Request: %s\n", request)
> 	response := l.next.Handle(request)
> 	fmt.Printf("[LOG] Response: %s\n", response)
> 	return response
> }
> 
> // AuthMiddleware
> type AuthMiddleware struct {
> 	next Handler
> }
> 
> func NewAuth(next Handler) *AuthMiddleware {
> 	return &AuthMiddleware{next: next}
> }
> 
> func (a *AuthMiddleware) Handle(request string) string {
> 	if !strings.HasPrefix(request, "token:") {
> 		return "AUTH DENIED: missing token"
> 	}
> 	return a.next.Handle(request)
> }
> 
> // RateLimitMiddleware
> type RateLimitMiddleware struct {
> 	next      Handler
> 	maxCalls  int
> 	callCount int
> }
> 
> func NewRateLimit(max int, next Handler) *RateLimitMiddleware {
> 	return &RateLimitMiddleware{next: next, maxCalls: max}
> }
> 
> func (r *RateLimitMiddleware) Handle(request string) string {
> 	if r.callCount >= r.maxCalls {
> 		return "RATE LIMITED: too many requests"
> 	}
> 	r.callCount++
> 	return r.next.Handle(request)
> }
> 
> func main() {
> 	chain := NewRateLimit(2, NewAuth(NewLogger(BaseHandler{})))
> 
> 	// Request 1: valid token
> 	result := chain.Handle("token:get-users")
> 	fmt.Printf("Result: %s\n\n", result)
> 
> 	// Request 2: missing token
> 	result = chain.Handle("get-users")
> 	fmt.Printf("Result: %s\n\n", result)
> 
> 	// Request 3: valid but hits rate limit (this is the 3rd call)
> 	result = chain.Handle("token:get-status")
> 	fmt.Printf("Result: %s\n\n", result)
> 
> 	// Request 4: rate limited
> 	result = chain.Handle("token:get-data")
> 	fmt.Printf("Result: %s\n", result)
> }
> ```
> 
> **Note:** The rate limiter counts all attempts (including auth-denied ones). In the output above, request 3 succeeds because only requests 1 and 2 counted before it. Request 4 is denied by rate limiting. If you want auth-denied requests to not count, move `AuthMiddleware` before `RateLimitMiddleware` in the chain.
