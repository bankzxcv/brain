---
title: "Go Practice: Functions"
date: 2026-04-07
tags:
  - golang
  - practice
  - functions
parent: "[[Golang Study]]"
---

# Go Practice: Functions

14 progressive problems. Write each solution from scratch to build muscle memory.

---

## P1: Max of Two Ints

Write a function `max(a, b int) int` that returns the larger of the two values. Call it with several test cases.

**Expected output:**
```
max(3, 7) = 7
max(10, 2) = 10
max(5, 5) = 5
```

> [!hint]- Hint
> A simple `if a > b` check is all you need. Go does not have a built-in `max` for ints in older versions (pre-1.21).

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func max(a, b int) int {
> 	if a > b {
> 		return a
> 	}
> 	return b
> }
> 
> func main() {
> 	fmt.Printf("max(3, 7) = %d\n", max(3, 7))
> 	fmt.Printf("max(10, 2) = %d\n", max(10, 2))
> 	fmt.Printf("max(5, 5) = %d\n", max(5, 5))
> }
> ```

---

## P2: Multiple Return Values - Divide with Error

Write `divide(a, b float64) (float64, error)` that returns an error when dividing by zero. Test with both valid and invalid inputs.

**Expected output:**
```
10 / 3 = 3.33
10 / 0 = error: division by zero
```

> [!hint]- Hint
> Return `0, fmt.Errorf("division by zero")` for the error case. The caller checks `if err != nil`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> )
> 
> func divide(a, b float64) (float64, error) {
> 	if b == 0 {
> 		return 0, fmt.Errorf("division by zero")
> 	}
> 	return a / b, nil
> }
> 
> func main() {
> 	result, err := divide(10, 3)
> 	if err != nil {
> 		fmt.Printf("10 / 3 = error: %v\n", err)
> 	} else {
> 		fmt.Printf("10 / 3 = %.2f\n", result)
> 	}
> 
> 	result, err = divide(10, 0)
> 	if err != nil {
> 		fmt.Printf("10 / 0 = error: %v\n", err)
> 	} else {
> 		fmt.Printf("10 / 0 = %.2f\n", result)
> 	}
> }
> ```

---

## P3: Named Return Values

Write `divmod(a, b int) (quotient, remainder int)` using named return values. Return the quotient and remainder of integer division.

**Expected output:**
```
17 / 5 = quotient: 3, remainder: 2
100 / 7 = quotient: 14, remainder: 2
```

> [!hint]- Hint
> Named return values are declared in the function signature. You can assign to them directly and use a bare `return`. Use `a / b` for quotient and `a % b` for remainder.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func divmod(a, b int) (quotient, remainder int) {
> 	quotient = a / b
> 	remainder = a % b
> 	return
> }
> 
> func main() {
> 	q, r := divmod(17, 5)
> 	fmt.Printf("17 / 5 = quotient: %d, remainder: %d\n", q, r)
> 
> 	q, r = divmod(100, 7)
> 	fmt.Printf("100 / 7 = quotient: %d, remainder: %d\n", q, r)
> }
> ```

---

## P4: Variadic Function - Find Minimum

Write `min(nums ...int) (int, error)` that returns the minimum value from any number of int arguments. Return an error if no arguments are provided.

**Expected output:**
```
min(3, 1, 4, 1, 5) = 1
min(42) = 42
min() = error: no arguments provided
```

> [!hint]- Hint
> The variadic parameter `nums ...int` gives you a `[]int` slice inside the function. Start with `nums[0]` as the minimum and iterate through the rest.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func min(nums ...int) (int, error) {
> 	if len(nums) == 0 {
> 		return 0, fmt.Errorf("no arguments provided")
> 	}
> 	m := nums[0]
> 	for _, n := range nums[1:] {
> 		if n < m {
> 			m = n
> 		}
> 	}
> 	return m, nil
> }
> 
> func main() {
> 	if v, err := min(3, 1, 4, 1, 5); err == nil {
> 		fmt.Printf("min(3, 1, 4, 1, 5) = %d\n", v)
> 	}
> 
> 	if v, err := min(42); err == nil {
> 		fmt.Printf("min(42) = %d\n", v)
> 	}
> 
> 	if _, err := min(); err != nil {
> 		fmt.Printf("min() = error: %v\n", err)
> 	}
> }
> ```

---

## P5: Function as Parameter

Write `apply(nums []int, fn func(int) int) []int` that applies `fn` to each element and returns a new slice. Test it with a "double" and a "square" function.

**Expected output:**
```
Original: [1 2 3 4 5]
Doubled:  [2 4 6 8 10]
Squared:  [1 4 9 16 25]
```

> [!hint]- Hint
> The second parameter type is `func(int) int`. Inside `apply`, create a new slice, iterate, apply `fn` to each element, and append. You can pass named functions or anonymous functions.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func apply(nums []int, fn func(int) int) []int {
> 	result := make([]int, len(nums))
> 	for i, n := range nums {
> 		result[i] = fn(n)
> 	}
> 	return result
> }
> 
> func double(n int) int { return n * 2 }
> func square(n int) int { return n * n }
> 
> func main() {
> 	nums := []int{1, 2, 3, 4, 5}
> 	fmt.Println("Original:", nums)
> 	fmt.Println("Doubled: ", apply(nums, double))
> 	fmt.Println("Squared: ", apply(nums, square))
> }
> ```

---

## P6: Return a Function (Closure) - Multiplier Factory

Write `multiplier(factor int) func(int) int` that returns a function which multiplies its argument by `factor`.

**Expected output:**
```
double(5) = 10
triple(5) = 15
times10(5) = 50
```

> [!hint]- Hint
> Return an anonymous function that captures `factor` from the enclosing scope. The returned function "closes over" `factor`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func multiplier(factor int) func(int) int {
> 	return func(n int) int {
> 		return n * factor
> 	}
> }
> 
> func main() {
> 	double := multiplier(2)
> 	triple := multiplier(3)
> 	times10 := multiplier(10)
> 
> 	fmt.Printf("double(5) = %d\n", double(5))
> 	fmt.Printf("triple(5) = %d\n", triple(5))
> 	fmt.Printf("times10(5) = %d\n", times10(5))
> }
> ```

---

## P7: Closure with State - Running Average

Write `newAverage() func(float64) float64` that returns a function. Each time the returned function is called with a new value, it returns the running average of all values seen so far.

**Expected output:**
```
add(10) -> avg = 10.00
add(20) -> avg = 15.00
add(30) -> avg = 20.00
add(0)  -> avg = 15.00
```

> [!hint]- Hint
> The closure needs to capture two variables: a running `sum` and a `count`. Each call adds to `sum`, increments `count`, and returns `sum / count`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func newAverage() func(float64) float64 {
> 	var sum float64
> 	var count int
> 
> 	return func(val float64) float64 {
> 		sum += val
> 		count++
> 		return sum / float64(count)
> 	}
> }
> 
> func main() {
> 	avg := newAverage()
> 
> 	fmt.Printf("add(10) -> avg = %.2f\n", avg(10))
> 	fmt.Printf("add(20) -> avg = %.2f\n", avg(20))
> 	fmt.Printf("add(30) -> avg = %.2f\n", avg(30))
> 	fmt.Printf("add(0)  -> avg = %.2f\n", avg(0))
> }
> ```

---

## P8: Recursive Fibonacci with Memoization

Write a recursive Fibonacci function. Then wrap it with memoization using a closure over a map. Compare call counts for `fib(10)` with and without memoization.

**Expected output (approximate):**
```
fib(10) = 55 (naive calls: 177)
fib(10) = 55 (memo calls: 19)
```

> [!hint]- Hint
> For memoization, use a `map[int]int` captured by the closure. Before computing, check if the value exists in the map. After computing, store it. Track call counts with a pointer to an int counter.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func fibNaive(n int, calls *int) int {
> 	*calls++
> 	if n <= 1 {
> 		return n
> 	}
> 	return fibNaive(n-1, calls) + fibNaive(n-2, calls)
> }
> 
> func fibMemo() func(int, *int) int {
> 	cache := map[int]int{}
> 
> 	var fib func(int, *int) int
> 	fib = func(n int, calls *int) int {
> 		*calls++
> 		if n <= 1 {
> 			return n
> 		}
> 		if v, ok := cache[n]; ok {
> 			return v
> 		}
> 		result := fib(n-1, calls) + fib(n-2, calls)
> 		cache[n] = result
> 		return result
> 	}
> 	return fib
> }
> 
> func main() {
> 	var naiveCalls int
> 	result := fibNaive(10, &naiveCalls)
> 	fmt.Printf("fib(10) = %d (naive calls: %d)\n", result, naiveCalls)
> 
> 	var memoCalls int
> 	fib := fibMemo()
> 	result = fib(10, &memoCalls)
> 	fmt.Printf("fib(10) = %d (memo calls: %d)\n", result, memoCalls)
> }
> ```

---

## P9: Function Type Alias

Define a type `Predicate func(string) bool`. Write `filter(items []string, pred Predicate) []string` that returns items where `pred` returns true. Test with predicates for "longer than 3 chars" and "starts with A".

**Expected output:**
```
All:     [Apple Bee Avocado Cat Ant]
Long:    [Apple Avocado]
StartsA: [Apple Avocado Ant]
```

> [!hint]- Hint
> `type Predicate func(string) bool` creates a named function type. You can use any function matching the signature where `Predicate` is expected.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> type Predicate func(string) bool
> 
> func filter(items []string, pred Predicate) []string {
> 	var result []string
> 	for _, item := range items {
> 		if pred(item) {
> 			result = append(result, item)
> 		}
> 	}
> 	return result
> }
> 
> func main() {
> 	items := []string{"Apple", "Bee", "Avocado", "Cat", "Ant"}
> 
> 	longerThan3 := Predicate(func(s string) bool {
> 		return len(s) > 3
> 	})
> 
> 	startsWithA := Predicate(func(s string) bool {
> 		return strings.HasPrefix(s, "A")
> 	})
> 
> 	fmt.Println("All:    ", items)
> 	fmt.Println("Long:   ", filter(items, longerThan3))
> 	fmt.Println("StartsA:", filter(items, startsWithA))
> }
> ```

---

## P10: Compose Two Functions

Write `compose(f, g func(int) int) func(int) int` that returns a new function computing `f(g(x))`. Test by composing "add 1" and "double".

**Expected output:**
```
addOne(5) = 6
double(5) = 10
doubleThemAddOne(5) = 11
addOneThenDouble(5) = 12
```

> [!hint]- Hint
> `compose` returns an anonymous function that calls `g(x)` first, then passes the result to `f`. Order matters: `compose(f, g)` means "apply g first, then f".

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func compose(f, g func(int) int) func(int) int {
> 	return func(x int) int {
> 		return f(g(x))
> 	}
> }
> 
> func main() {
> 	addOne := func(x int) int { return x + 1 }
> 	double := func(x int) int { return x * 2 }
> 
> 	fmt.Printf("addOne(5) = %d\n", addOne(5))
> 	fmt.Printf("double(5) = %d\n", double(5))
> 
> 	// f(g(x)): first double, then addOne
> 	doubleThenAddOne := compose(addOne, double)
> 	fmt.Printf("doubleThemAddOne(5) = %d\n", doubleThenAddOne(5))
> 
> 	// f(g(x)): first addOne, then double
> 	addOneThenDouble := compose(double, addOne)
> 	fmt.Printf("addOneThenDouble(5) = %d\n", addOneThenDouble(5))
> }
> ```

---

## P11: Retry Logic

Write a function `retry(attempts int, delay time.Duration, fn func() error) error` that calls `fn` up to `attempts` times, waiting `delay` between retries. It should return `nil` on the first success or the last error if all attempts fail. Test with a function that fails the first 2 times then succeeds.

**Expected output (approximate):**
```
Attempt 1: failed (simulated error)
Attempt 2: failed (simulated error)
Attempt 3: success!
Result: <nil>
```

> [!hint]- Hint
> Use a `for` loop from 0 to `attempts`. Call `fn()`, check if `err == nil`. If not nil and not the last attempt, `time.Sleep(delay)`. Return the error from the last attempt.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"time"
> )
> 
> func retry(attempts int, delay time.Duration, fn func() error) error {
> 	var err error
> 	for i := 0; i < attempts; i++ {
> 		err = fn()
> 		if err == nil {
> 			return nil
> 		}
> 		if i < attempts-1 {
> 			time.Sleep(delay)
> 		}
> 	}
> 	return err
> }
> 
> func main() {
> 	callCount := 0
> 
> 	unreliable := func() error {
> 		callCount++
> 		if callCount < 3 {
> 			fmt.Printf("Attempt %d: failed (simulated error)\n", callCount)
> 			return fmt.Errorf("simulated error")
> 		}
> 		fmt.Printf("Attempt %d: success!\n", callCount)
> 		return nil
> 	}
> 
> 	err := retry(5, 100*time.Millisecond, unreliable)
> 	fmt.Printf("Result: %v\n", err)
> }
> ```

---

## P12: Middleware Chain

Build a simple middleware system. Define `type Middleware func(http.Handler) http.Handler`. Write two middlewares: `Logger` (prints the request method and path) and `Timer` (prints how long the request took). Chain them together and test with a simple handler.

**Expected output (approximate):**
```
[Logger] GET /hello
[Timer] Request took 1ms
Hello, World!
```

> [!hint]- Hint
> Each middleware takes a handler and returns a new handler that wraps the original. To chain: `Timer(Logger(handler))`. Use `httptest.NewRecorder` and `httptest.NewRequest` for testing without starting a server.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"net/http"
> 	"net/http/httptest"
> 	"time"
> )
> 
> type Middleware func(http.Handler) http.Handler
> 
> func Logger(next http.Handler) http.Handler {
> 	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
> 		fmt.Printf("[Logger] %s %s\n", r.Method, r.URL.Path)
> 		next.ServeHTTP(w, r)
> 	})
> }
> 
> func Timer(next http.Handler) http.Handler {
> 	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
> 		start := time.Now()
> 		next.ServeHTTP(w, r)
> 		fmt.Printf("[Timer] Request took %v\n", time.Since(start).Round(time.Millisecond))
> 	})
> }
> 
> func Chain(handler http.Handler, middlewares ...Middleware) http.Handler {
> 	// Apply in reverse order so first middleware in the list is outermost
> 	for i := len(middlewares) - 1; i >= 0; i-- {
> 		handler = middlewares[i](handler)
> 	}
> 	return handler
> }
> 
> func main() {
> 	hello := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
> 		fmt.Fprintln(w, "Hello, World!")
> 	})
> 
> 	// Chain middlewares: Timer wraps Logger wraps hello
> 	handler := Chain(hello, Timer, Logger)
> 
> 	// Test without starting a real server
> 	req := httptest.NewRequest("GET", "/hello", nil)
> 	rec := httptest.NewRecorder()
> 	handler.ServeHTTP(rec, req)
> 
> 	fmt.Print(rec.Body.String())
> }
> ```

---

## P13: Simple Event System

Build a simple event system. Implement:
- `type EventBus struct` with a map of event names to slices of handler functions
- `On(event string, handler func(data any))` to register a handler
- `Emit(event string, data any)` to call all handlers for an event

Test by registering handlers for `"user:login"` and `"user:logout"` events.

**Expected output:**
```
[Logger] user:login: alice
[Greeter] Welcome, alice!
[Logger] user:logout: alice
```

> [!hint]- Hint
> Use `map[string][]func(any)` as the internal storage. `On` appends to the slice for that key. `Emit` iterates and calls each handler.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type EventBus struct {
> 	handlers map[string][]func(data any)
> }
> 
> func NewEventBus() *EventBus {
> 	return &EventBus{
> 		handlers: make(map[string][]func(data any)),
> 	}
> }
> 
> func (eb *EventBus) On(event string, handler func(data any)) {
> 	eb.handlers[event] = append(eb.handlers[event], handler)
> }
> 
> func (eb *EventBus) Emit(event string, data any) {
> 	for _, handler := range eb.handlers[event] {
> 		handler(data)
> 	}
> }
> 
> func main() {
> 	bus := NewEventBus()
> 
> 	// Register handlers
> 	bus.On("user:login", func(data any) {
> 		fmt.Printf("[Logger] user:login: %v\n", data)
> 	})
> 	bus.On("user:login", func(data any) {
> 		fmt.Printf("[Greeter] Welcome, %v!\n", data)
> 	})
> 	bus.On("user:logout", func(data any) {
> 		fmt.Printf("[Logger] user:logout: %v\n", data)
> 	})
> 
> 	// Emit events
> 	bus.Emit("user:login", "alice")
> 	bus.Emit("user:logout", "alice")
> }
> ```

---

## P14: Pipeline Builder with Error Handling

Build a `Pipeline` that chains transform functions. Each step has the signature `func(any) (any, error)`. The pipeline should stop at the first error. Implement:
- `type Step func(any) (any, error)`
- `type Pipeline struct` with a slice of steps
- `Add(step Step)` to add a step
- `Run(input any) (any, error)` to execute all steps sequentially

Test with a pipeline that: parses a string to int, doubles it, and converts back to string.

**Expected output:**
```
Pipeline("42") = "84", err=<nil>
Pipeline("abc") = <nil>, err=strconv.Atoi: parsing "abc": invalid syntax
```

> [!hint]- Hint
> `Run` loops through steps, passing each output as the next input. If any step returns an error, return immediately. Use type assertions (`val.(int)`) in each step.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strconv"
> )
> 
> type Step func(any) (any, error)
> 
> type Pipeline struct {
> 	steps []Step
> }
> 
> func NewPipeline() *Pipeline {
> 	return &Pipeline{}
> }
> 
> func (p *Pipeline) Add(step Step) *Pipeline {
> 	p.steps = append(p.steps, step)
> 	return p
> }
> 
> func (p *Pipeline) Run(input any) (any, error) {
> 	var err error
> 	current := input
> 	for _, step := range p.steps {
> 		current, err = step(current)
> 		if err != nil {
> 			return nil, err
> 		}
> 	}
> 	return current, nil
> }
> 
> func main() {
> 	parseToInt := func(v any) (any, error) {
> 		s, ok := v.(string)
> 		if !ok {
> 			return nil, fmt.Errorf("expected string, got %T", v)
> 		}
> 		return strconv.Atoi(s)
> 	}
> 
> 	doubleIt := func(v any) (any, error) {
> 		n, ok := v.(int)
> 		if !ok {
> 			return nil, fmt.Errorf("expected int, got %T", v)
> 		}
> 		return n * 2, nil
> 	}
> 
> 	toString := func(v any) (any, error) {
> 		return fmt.Sprintf("%v", v), nil
> 	}
> 
> 	pipe := NewPipeline().Add(parseToInt).Add(doubleIt).Add(toString)
> 
> 	result, err := pipe.Run("42")
> 	fmt.Printf("Pipeline(\"42\") = %q, err=%v\n", result, err)
> 
> 	result, err = pipe.Run("abc")
> 	fmt.Printf("Pipeline(\"abc\") = %v, err=%v\n", result, err)
> }
> ```
