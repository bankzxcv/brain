---
title: "Go Practice: Defer Panic Recover"
date: 2026-04-07
tags:
  - golang
  - practice
  - defer-panic-recover
parent: "[[Golang Study]]"
---

# Go Practice: Defer Panic Recover

12 progressive problems covering defer, panic, and recover - from basic ordering to production patterns.

---

## P1: Basic Defer - Predict the Output

Without running it, predict the output of this program. Then verify by running it.

```go
package main

import "fmt"

func main() {
    fmt.Println("start")
    defer fmt.Println("first defer")
    defer fmt.Println("second defer")
    defer fmt.Println("third defer")
    fmt.Println("end")
}
```

**What prints?**

> [!hint]- Hint
> Deferred calls execute in LIFO (last in, first out) order - like a stack. They run after the surrounding function returns, but before it actually returns to the caller.

> [!success]- Solution
> **Output:**
> ```
> start
> end
> third defer
> second defer
> first defer
> ```
> 
> Defers execute in reverse order (stack). Non-deferred statements execute normally. All defers run after `fmt.Println("end")` but before `main` returns.
> 
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	fmt.Println("start")
> 	defer fmt.Println("first defer")
> 	defer fmt.Println("second defer")
> 	defer fmt.Println("third defer")
> 	fmt.Println("end")
> }
> ```

---

## P2: Defer for Resource Cleanup

Write a function that creates a temporary file, writes some data to it, and ensures the file is closed properly using `defer` - even if the write fails.

**Expected output:**
```
Created temp file: /tmp/example-XXXXXX
Writing data...
File closed successfully
File contents: Hello from Go!
```

> [!hint]- Hint
> Place `defer f.Close()` immediately after opening the file, before doing any work. This guarantees cleanup regardless of what happens next. Use `os.CreateTemp` for the temp file.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"os"
> )
> 
> func writeToTempFile(data string) (string, error) {
> 	f, err := os.CreateTemp("", "example-")
> 	if err != nil {
> 		return "", err
> 	}
> 	defer func() {
> 		f.Close()
> 		fmt.Println("File closed successfully")
> 	}()
> 
> 	fmt.Println("Created temp file:", f.Name())
> 	fmt.Println("Writing data...")
> 
> 	_, err = f.WriteString(data)
> 	if err != nil {
> 		return "", err
> 	}
> 
> 	return f.Name(), nil
> }
> 
> func main() {
> 	filename, err := writeToTempFile("Hello from Go!")
> 	if err != nil {
> 		fmt.Println("Error:", err)
> 		return
> 	}
> 
> 	// Read back to verify
> 	data, _ := os.ReadFile(filename)
> 	fmt.Println("File contents:", string(data))
> 
> 	// Clean up
> 	os.Remove(filename)
> }
> ```

---

## P3: Defer Evaluates Arguments Immediately

Predict the output. The key insight is when defer evaluates its arguments.

```go
package main

import "fmt"

func main() {
    x := 10
    defer fmt.Println("deferred x =", x)
    x = 20
    fmt.Println("current x =", x)
}
```

**What prints?**

> [!hint]- Hint
> Arguments to a deferred function are evaluated when the `defer` statement is executed, NOT when the deferred function runs. The value of `x` is captured at the time of the defer call.

> [!success]- Solution
> **Output:**
> ```
> current x = 20
> deferred x = 10
> ```
> 
> Even though `x` is changed to 20 before the function returns, the deferred call captured `x = 10` at the time the `defer` statement was executed.
> 
> **To capture the latest value, use a closure:**
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	x := 10
> 
> 	// Captures x by value at defer time -> prints 10
> 	defer fmt.Println("deferred x =", x)
> 
> 	// Captures x by reference (closure) -> prints 20
> 	defer func() {
> 		fmt.Println("closure x =", x)
> 	}()
> 
> 	x = 20
> 	fmt.Println("current x =", x)
> }
> ```
> 
> Output with closure:
> ```
> current x = 20
> closure x = 20
> deferred x = 10
> ```

---

## P4: Defer in a Loop - The Leak Problem

The following code opens files in a loop with defer. Explain why this leaks file descriptors, and write the fixed version.

```go
// BUGGY version - DO NOT USE
func processFiles(filenames []string) error {
    for _, name := range filenames {
        f, err := os.Open(name)
        if err != nil {
            return err
        }
        defer f.Close() // BUG: defers pile up until function returns
        
        // process file...
    }
    return nil
}
```

> [!hint]- Hint
> `defer` runs when the enclosing **function** returns, not when the enclosing **block** (loop iteration) ends. So all file handles stay open until `processFiles` returns. Fix it by extracting the loop body into a separate function, or close explicitly.

> [!success]- Solution
> **The problem:** All `defer f.Close()` calls stack up and only execute when `processFiles` returns. If there are 1000 files, all 1000 are open simultaneously.
> 
> **Fix 1: Extract to a function**
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"os"
> )
> 
> func processFile(name string) error {
> 	f, err := os.Open(name)
> 	if err != nil {
> 		return err
> 	}
> 	defer f.Close() // Runs when processFile returns (each iteration)
> 
> 	// process file...
> 	fmt.Printf("Processing: %s\n", name)
> 	return nil
> }
> 
> func processFiles(filenames []string) error {
> 	for _, name := range filenames {
> 		if err := processFile(name); err != nil {
> 			return err
> 		}
> 	}
> 	return nil
> }
> 
> func main() {
> 	// Create temp files for demonstration
> 	var names []string
> 	for i := 0; i < 3; i++ {
> 		f, _ := os.CreateTemp("", fmt.Sprintf("test-%d-", i))
> 		names = append(names, f.Name())
> 		f.Close()
> 	}
> 
> 	err := processFiles(names)
> 	if err != nil {
> 		fmt.Println("Error:", err)
> 	}
> 
> 	// Cleanup
> 	for _, name := range names {
> 		os.Remove(name)
> 	}
> }
> ```
> 
> **Fix 2: Use an anonymous function**
> ```go
> func processFiles(filenames []string) error {
> 	for _, name := range filenames {
> 		err := func() error {
> 			f, err := os.Open(name)
> 			if err != nil {
> 				return err
> 			}
> 			defer f.Close() // Runs when anonymous func returns
> 
> 			fmt.Printf("Processing: %s\n", name)
> 			return nil
> 		}()
> 		if err != nil {
> 			return err
> 		}
> 	}
> 	return nil
> }
> ```

---

## P5: Defer with Named Return Values

Predict the output. Then write a function that uses defer to modify a named return value (a common pattern for error annotation).

```go
package main

import "fmt"

func double(x int) (result int) {
    defer func() {
        result *= 2
    }()
    return x
}
```

What does `double(5)` return?

> [!hint]- Hint
> When you use `return x` with a named return value, it first assigns `result = x`, then runs deferred functions, then returns. The defer can modify the named return value before it's actually returned to the caller.

> [!success]- Solution
> `double(5)` returns **10**.
> 
> The sequence is:
> 1. `return x` sets `result = 5`
> 2. Deferred function runs: `result *= 2` makes `result = 10`
> 3. Function returns `result` which is now 10
> 
> **Practical example - annotating errors with context:**
> ```go
> package main
> 
> import (
> 	"fmt"
> )
> 
> func double(x int) (result int) {
> 	defer func() {
> 		result *= 2
> 	}()
> 	return x
> }
> 
> func doWork(input string) (err error) {
> 	defer func() {
> 		if err != nil {
> 			err = fmt.Errorf("doWork(%q): %w", input, err)
> 		}
> 	}()
> 
> 	if input == "" {
> 		return fmt.Errorf("empty input")
> 	}
> 	return nil
> }
> 
> func main() {
> 	fmt.Println("double(5) =", double(5))
> 
> 	err := doWork("")
> 	fmt.Println("Error:", err)
> 	// Output: Error: doWork(""): empty input
> }
> ```

---

## P6: Panic and Recover - Safe Division

Write a `safeDivide` function that uses `recover` to catch a panic from integer division by zero. Return an error instead of crashing.

**Expected output:**
```
10 / 3 = 3, err: <nil>
10 / 0 = 0, err: division by zero: runtime error: integer divide by zero
```

> [!hint]- Hint
> `recover()` must be called inside a deferred function. It returns `nil` if no panic occurred. Use a named return value for the error so the deferred function can set it.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func safeDivide(a, b int) (result int, err error) {
> 	defer func() {
> 		if r := recover(); r != nil {
> 			err = fmt.Errorf("division by zero: %v", r)
> 		}
> 	}()
> 
> 	return a / b, nil
> }
> 
> func main() {
> 	result, err := safeDivide(10, 3)
> 	fmt.Printf("10 / 3 = %d, err: %v\n", result, err)
> 
> 	result, err = safeDivide(10, 0)
> 	fmt.Printf("10 / 0 = %d, err: %v\n", result, err)
> }
> ```

---

## P7: Recover in Goroutine

Launch 3 goroutines. One of them panics. Demonstrate that without recovery, the whole program crashes. Then add recovery so the other goroutines continue running.

**Expected output:**
```
Worker 0: starting
Worker 1: starting
Worker 2: starting
Worker 1: recovered from panic: something went wrong in worker 1
Worker 0: done
Worker 2: done
All workers finished
```

> [!hint]- Hint
> A panic in a goroutine will crash the entire program if not recovered in that same goroutine. You cannot recover a panic from a different goroutine. Each goroutine must have its own `defer/recover`.

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
> func worker(id int, wg *sync.WaitGroup) {
> 	defer wg.Done()
> 	defer func() {
> 		if r := recover(); r != nil {
> 			fmt.Printf("Worker %d: recovered from panic: %v\n", id, r)
> 		}
> 	}()
> 
> 	fmt.Printf("Worker %d: starting\n", id)
> 
> 	if id == 1 {
> 		time.Sleep(50 * time.Millisecond)
> 		panic(fmt.Sprintf("something went wrong in worker %d", id))
> 	}
> 
> 	time.Sleep(100 * time.Millisecond)
> 	fmt.Printf("Worker %d: done\n", id)
> }
> 
> func main() {
> 	var wg sync.WaitGroup
> 
> 	for i := 0; i < 3; i++ {
> 		wg.Add(1)
> 		go worker(i, &wg)
> 	}
> 
> 	wg.Wait()
> 	fmt.Println("All workers finished")
> }
> ```

---

## P8: Transaction Pattern with Defer

Implement a transaction-like pattern where operations are automatically rolled back on error. Use `defer` to check if the transaction should commit or rollback.

**Expected output:**
```
=== Successful transaction ===
BEGIN transaction
  Inserting user: alice
  Inserting email: alice@go.dev
COMMIT transaction

=== Failed transaction ===
BEGIN transaction
  Inserting user: bob
  Inserting email: (empty - will fail)
  Error: email cannot be empty
ROLLBACK transaction
```

> [!hint]- Hint
> Use a named error return. In the deferred function, check if `err != nil`. If yes, rollback. If no, commit. The `Transaction` struct can track operations to undo.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> )
> 
> type Transaction struct {
> 	operations []string
> 	committed  bool
> }
> 
> func NewTransaction() *Transaction {
> 	fmt.Println("BEGIN transaction")
> 	return &Transaction{}
> }
> 
> func (tx *Transaction) Execute(op string) {
> 	tx.operations = append(tx.operations, op)
> 	fmt.Printf("  %s\n", op)
> }
> 
> func (tx *Transaction) Commit() {
> 	tx.committed = true
> 	fmt.Println("COMMIT transaction")
> }
> 
> func (tx *Transaction) Rollback() {
> 	fmt.Println("ROLLBACK transaction")
> }
> 
> func (tx *Transaction) Finish(err *error) {
> 	if *err != nil {
> 		tx.Rollback()
> 	} else {
> 		tx.Commit()
> 	}
> }
> 
> func createUser(name, email string) (err error) {
> 	tx := NewTransaction()
> 	defer tx.Finish(&err)
> 
> 	tx.Execute(fmt.Sprintf("Inserting user: %s", name))
> 
> 	if email == "" {
> 		fmt.Println("  Error: email cannot be empty")
> 		return fmt.Errorf("email cannot be empty")
> 	}
> 	tx.Execute(fmt.Sprintf("Inserting email: %s", email))
> 
> 	return nil
> }
> 
> func main() {
> 	fmt.Println("=== Successful transaction ===")
> 	createUser("alice", "alice@go.dev")
> 
> 	fmt.Println("\n=== Failed transaction ===")
> 	createUser("bob", "")
> }
> ```

---

## P9: Timing Function Execution with Defer

Write a `timeIt` function that measures how long a function takes. Use `defer` so the timing call is a one-liner at the start of any function.

**Expected output:**
```
slowOperation took 200ms
fastOperation took 50ms
```

> [!hint]- Hint
> The trick is that `timeIt` records `time.Now()` when called, and returns a function that computes the elapsed time. Use `defer timeIt("name")()` - note the extra `()` which calls the returned function when the enclosing function exits.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"time"
> )
> 
> func timeIt(name string) func() {
> 	start := time.Now()
> 	return func() {
> 		elapsed := time.Since(start)
> 		fmt.Printf("%s took %dms\n", name, elapsed.Milliseconds())
> 	}
> }
> 
> func slowOperation() {
> 	defer timeIt("slowOperation")()
> 	time.Sleep(200 * time.Millisecond)
> }
> 
> func fastOperation() {
> 	defer timeIt("fastOperation")()
> 	time.Sleep(50 * time.Millisecond)
> }
> 
> func main() {
> 	slowOperation()
> 	fastOperation()
> }
> ```
> 
> **Key insight:** `defer timeIt("name")()` works in two steps:
> 1. `timeIt("name")` executes immediately (captures start time), returns a closure
> 2. `()` is deferred - the returned closure runs when the function exits (computes elapsed time)

---

## P10: Defer Stack Trace Logger

Write a `trace` function that logs function entry and exit. Use it to trace a call chain: `main -> a -> b -> c`.

**Expected output:**
```
entering: main
  entering: a
    entering: b
      entering: c
      leaving: c
    leaving: b
  leaving: a
leaving: main
```

> [!hint]- Hint
> Use a package-level `depth` variable to track indentation. `trace` increments it on entry and decrements (via defer) on exit. Return a function from `trace` and call it with `defer trace("name")()`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> var depth int
> 
> func trace(name string) func() {
> 	indent := strings.Repeat("  ", depth)
> 	fmt.Printf("%sentering: %s\n", indent, name)
> 	depth++
> 	return func() {
> 		depth--
> 		indent := strings.Repeat("  ", depth)
> 		fmt.Printf("%sleaving: %s\n", indent, name)
> 	}
> }
> 
> func c() {
> 	defer trace("c")()
> 	// do some work
> }
> 
> func b() {
> 	defer trace("b")()
> 	c()
> }
> 
> func a() {
> 	defer trace("a")()
> 	b()
> }
> 
> func main() {
> 	defer trace("main")()
> 	a()
> }
> ```

---

## P11: Cleanup Registry

Implement a `CleanupRegistry` that lets you register multiple cleanup functions. All registered cleanups run (in reverse order) when `RunAll` is called. Even if one cleanup panics, the others still run.

**Expected output:**
```
Registering cleanup: close database
Registering cleanup: flush logs
Registering cleanup: remove temp files
Running all cleanups...
  Running: remove temp files... done
  Running: flush logs... done
  Running: close database... done
All cleanups completed
```

Then test with a panicking cleanup:
```
  Running: cleanup 3... done
  Running: bad cleanup... PANIC: oh no (recovered, continuing)
  Running: cleanup 1... done
All cleanups completed (1 panic(s) recovered)
```

> [!hint]- Hint
> Store cleanup functions in a `[]func()` slice. Run them in reverse order. Wrap each call in a `func()` with `defer/recover` so one panic doesn't stop the others. Count recovered panics.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type CleanupRegistry struct {
> 	cleanups []namedCleanup
> }
> 
> type namedCleanup struct {
> 	name string
> 	fn   func()
> }
> 
> func NewCleanupRegistry() *CleanupRegistry {
> 	return &CleanupRegistry{}
> }
> 
> func (r *CleanupRegistry) Register(name string, fn func()) {
> 	fmt.Printf("Registering cleanup: %s\n", name)
> 	r.cleanups = append(r.cleanups, namedCleanup{name: name, fn: fn})
> }
> 
> func (r *CleanupRegistry) RunAll() {
> 	fmt.Println("Running all cleanups...")
> 	panics := 0
> 
> 	// Run in reverse order (LIFO)
> 	for i := len(r.cleanups) - 1; i >= 0; i-- {
> 		c := r.cleanups[i]
> 		func() {
> 			defer func() {
> 				if r := recover(); r != nil {
> 					fmt.Printf(" PANIC: %v (recovered, continuing)\n", r)
> 					panics++
> 				}
> 			}()
> 			fmt.Printf("  Running: %s...", c.name)
> 			c.fn()
> 			fmt.Println(" done")
> 		}()
> 	}
> 
> 	if panics > 0 {
> 		fmt.Printf("All cleanups completed (%d panic(s) recovered)\n", panics)
> 	} else {
> 		fmt.Println("All cleanups completed")
> 	}
> }
> 
> func main() {
> 	// Normal case
> 	fmt.Println("=== Normal cleanups ===")
> 	reg := NewCleanupRegistry()
> 	reg.Register("close database", func() { /* simulate work */ })
> 	reg.Register("flush logs", func() { /* simulate work */ })
> 	reg.Register("remove temp files", func() { /* simulate work */ })
> 	reg.RunAll()
> 
> 	// Case with a panicking cleanup
> 	fmt.Println("\n=== With panicking cleanup ===")
> 	reg2 := NewCleanupRegistry()
> 	reg2.Register("cleanup 1", func() {})
> 	reg2.Register("bad cleanup", func() { panic("oh no") })
> 	reg2.Register("cleanup 3", func() {})
> 	reg2.RunAll()
> }
> ```

---

## P12: Safe Worker with Panic Recovery and Error Channel

Build a `safeWorker` that:
1. Runs a task function in a goroutine
2. Recovers from any panics
3. Reports errors (including recovered panics) through an error channel
4. Reports success through a result channel

Run 5 workers where workers 1 and 3 panic, and worker 4 returns an error.

**Expected output:**
```
Starting 5 workers...
Worker 0: success -> result: "processed: task-0"
Worker 1: panic recovered -> "unexpected failure in task-1"
Worker 2: success -> result: "processed: task-2"
Worker 3: panic recovered -> "nil pointer in task-3"
Worker 4: error -> "task-4: known error condition"
All workers finished. Success: 2, Errors: 1, Panics: 2
```

> [!hint]- Hint
> Define a `WorkerResult` struct with `ID int`, `Value string`, `Err error`, `Panicked bool`. Each worker sends its result through a single channel. Use `defer/recover` inside the goroutine to catch panics and convert them to `WorkerResult` with `Panicked: true`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sync"
> )
> 
> type WorkerResult struct {
> 	ID       int
> 	Value    string
> 	Err      error
> 	Panicked bool
> 	PanicVal any
> }
> 
> func safeWorker(id int, task func() (string, error), results chan<- WorkerResult, wg *sync.WaitGroup) {
> 	defer wg.Done()
> 
> 	// This goroutine-level recover prevents program crash
> 	defer func() {
> 		if r := recover(); r != nil {
> 			results <- WorkerResult{
> 				ID:       id,
> 				Panicked: true,
> 				PanicVal: r,
> 			}
> 		}
> 	}()
> 
> 	val, err := task()
> 	results <- WorkerResult{
> 		ID:    id,
> 		Value: val,
> 		Err:   err,
> 	}
> }
> 
> func main() {
> 	tasks := []func() (string, error){
> 		// Worker 0: success
> 		func() (string, error) {
> 			return "processed: task-0", nil
> 		},
> 		// Worker 1: panic
> 		func() (string, error) {
> 			panic("unexpected failure in task-1")
> 		},
> 		// Worker 2: success
> 		func() (string, error) {
> 			return "processed: task-2", nil
> 		},
> 		// Worker 3: panic
> 		func() (string, error) {
> 			panic("nil pointer in task-3")
> 		},
> 		// Worker 4: error
> 		func() (string, error) {
> 			return "", fmt.Errorf("task-4: known error condition")
> 		},
> 	}
> 
> 	fmt.Printf("Starting %d workers...\n", len(tasks))
> 
> 	results := make(chan WorkerResult, len(tasks))
> 	var wg sync.WaitGroup
> 
> 	for i, task := range tasks {
> 		wg.Add(1)
> 		go safeWorker(i, task, results, &wg)
> 	}
> 
> 	// Close results channel after all workers finish
> 	go func() {
> 		wg.Wait()
> 		close(results)
> 	}()
> 
> 	// Collect results
> 	collected := make([]WorkerResult, 0, len(tasks))
> 	for r := range results {
> 		collected = append(collected, r)
> 	}
> 
> 	// Sort by ID for deterministic output
> 	sorted := make([]WorkerResult, len(tasks))
> 	for _, r := range collected {
> 		sorted[r.ID] = r
> 	}
> 
> 	successes, errors, panics := 0, 0, 0
> 	for _, r := range sorted {
> 		switch {
> 		case r.Panicked:
> 			fmt.Printf("Worker %d: panic recovered -> %q\n", r.ID, r.PanicVal)
> 			panics++
> 		case r.Err != nil:
> 			fmt.Printf("Worker %d: error -> %q\n", r.ID, r.Err.Error())
> 			errors++
> 		default:
> 			fmt.Printf("Worker %d: success -> result: %q\n", r.ID, r.Value)
> 			successes++
> 		}
> 	}
> 
> 	fmt.Printf("All workers finished. Success: %d, Errors: %d, Panics: %d\n",
> 		successes, errors, panics)
> }
> ```
