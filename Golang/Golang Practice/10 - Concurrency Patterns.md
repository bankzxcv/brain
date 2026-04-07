---
title: "Go Practice: Concurrency Patterns"
date: 2026-04-07
tags:
  - golang
  - practice
  - concurrency-patterns
parent: "[[Golang Study]]"
---

# Concurrency Patterns

Progressive drills for real-world concurrency patterns: WaitGroup, Mutex, context, worker pools, and more. Write each solution from scratch.

---

## P1: WaitGroup - Wait for N Workers

Launch 5 goroutines. Each prints `"worker <id> done"`. Use `sync.WaitGroup` to wait for all of them before printing `"all workers finished"`.

**Expected output (order of workers may vary):**
```
worker 1 done
worker 2 done
worker 3 done
worker 4 done
worker 5 done
all workers finished
```

> [!hint]- Hint
> Call `wg.Add(1)` before launching each goroutine. Call `wg.Done()` (often via `defer`) inside the goroutine. Call `wg.Wait()` in main.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sync"
> )
> 
> func main() {
> 	var wg sync.WaitGroup
> 
> 	for i := 1; i <= 5; i++ {
> 		wg.Add(1)
> 		go func(id int) {
> 			defer wg.Done()
> 			fmt.Printf("worker %d done\n", id)
> 		}(i)
> 	}
> 
> 	wg.Wait()
> 	fmt.Println("all workers finished")
> }
> ```

---

## P2: Mutex - Protect a Shared Counter

Launch 1000 goroutines that each increment a shared `int` variable. Without a mutex this will race. Use `sync.Mutex` to protect the counter. Print the final value (should be 1000).

**Expected output:**
```
final: 1000
```

> [!hint]- Hint
> `mu.Lock()` before incrementing, `mu.Unlock()` after. Run with `go run -race .` to verify no data races.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sync"
> )
> 
> func main() {
> 	var (
> 		mu      sync.Mutex
> 		counter int
> 		wg      sync.WaitGroup
> 	)
> 
> 	for i := 0; i < 1000; i++ {
> 		wg.Add(1)
> 		go func() {
> 			defer wg.Done()
> 			mu.Lock()
> 			counter++
> 			mu.Unlock()
> 		}()
> 	}
> 
> 	wg.Wait()
> 	fmt.Println("final:", counter)
> }
> ```

---

## P3: RWMutex - Concurrent Readers, Exclusive Writer

Create a `SafeMap` struct wrapping `map[string]int` with `sync.RWMutex`. Implement `Get(key)`, `Set(key, val)`. Launch 5 reader goroutines (each reads key `"x"` in a loop 100 times) and 1 writer goroutine (increments `"x"` 100 times). Print the final value.

**Expected output:**
```
final value of x: 100
```

> [!hint]- Hint
> Readers call `mu.RLock()`/`mu.RUnlock()`. The writer calls `mu.Lock()`/`mu.Unlock()`. Multiple readers can hold the lock simultaneously.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sync"
> )
> 
> type SafeMap struct {
> 	mu sync.RWMutex
> 	m  map[string]int
> }
> 
> func NewSafeMap() *SafeMap {
> 	return &SafeMap{m: make(map[string]int)}
> }
> 
> func (s *SafeMap) Get(key string) int {
> 	s.mu.RLock()
> 	defer s.mu.RUnlock()
> 	return s.m[key]
> }
> 
> func (s *SafeMap) Set(key string, val int) {
> 	s.mu.Lock()
> 	defer s.mu.Unlock()
> 	s.m[key] = val
> }
> 
> func main() {
> 	sm := NewSafeMap()
> 	var wg sync.WaitGroup
> 
> 	// writer
> 	wg.Add(1)
> 	go func() {
> 		defer wg.Done()
> 		for i := 0; i < 100; i++ {
> 			sm.mu.Lock()
> 			sm.m["x"]++
> 			sm.mu.Unlock()
> 		}
> 	}()
> 
> 	// readers
> 	for r := 0; r < 5; r++ {
> 		wg.Add(1)
> 		go func() {
> 			defer wg.Done()
> 			for i := 0; i < 100; i++ {
> 				_ = sm.Get("x")
> 			}
> 		}()
> 	}
> 
> 	wg.Wait()
> 	fmt.Println("final value of x:", sm.Get("x"))
> }
> ```

---

## P4: Worker Pool

Create a worker pool with 3 workers processing jobs from a shared channel. Each job is an `int`; the result is `job * 2`. Send 10 jobs, collect all 10 results, and print them.

**Expected output (order may vary):**
```
job 1 -> 2
job 2 -> 4
...
job 10 -> 20
```

> [!hint]- Hint
> Define `Job` and `Result` structs. Workers range over the jobs channel. Use a WaitGroup to close the results channel when all workers finish.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sync"
> )
> 
> type Job struct {
> 	ID    int
> 	Value int
> }
> 
> type Result struct {
> 	JobID  int
> 	Output int
> }
> 
> func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
> 	defer wg.Done()
> 	for job := range jobs {
> 		results <- Result{JobID: job.ID, Output: job.Value * 2}
> 	}
> }
> 
> func main() {
> 	jobs := make(chan Job, 10)
> 	results := make(chan Result, 10)
> 	var wg sync.WaitGroup
> 
> 	// Start 3 workers
> 	for w := 0; w < 3; w++ {
> 		wg.Add(1)
> 		go worker(w, jobs, results, &wg)
> 	}
> 
> 	// Send 10 jobs
> 	for j := 1; j <= 10; j++ {
> 		jobs <- Job{ID: j, Value: j}
> 	}
> 	close(jobs)
> 
> 	// Close results when workers are done
> 	go func() {
> 		wg.Wait()
> 		close(results)
> 	}()
> 
> 	// Collect results
> 	for r := range results {
> 		fmt.Printf("job %d -> %d\n", r.JobID, r.Output)
> 	}
> }
> ```

---

## P5: Semaphore - Limit Concurrency

You have 20 tasks, but only 3 should run at a time. Use a **buffered channel** as a counting semaphore to limit concurrency. Each task prints its ID and sleeps for 100ms.

> [!hint]- Hint
> Create `sem := make(chan struct{}, 3)`. Before each goroutine's work, send to `sem` (acquire). After the work, receive from `sem` (release).

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
> func main() {
> 	sem := make(chan struct{}, 3)
> 	var wg sync.WaitGroup
> 
> 	for i := 1; i <= 20; i++ {
> 		wg.Add(1)
> 		go func(id int) {
> 			defer wg.Done()
> 
> 			sem <- struct{}{} // acquire
> 			defer func() { <-sem }() // release
> 
> 			fmt.Printf("task %d running\n", id)
> 			time.Sleep(100 * time.Millisecond)
> 		}(i)
> 	}
> 
> 	wg.Wait()
> 	fmt.Println("all tasks done")
> }
> ```

---

## P6: Context with Timeout

Write a function `slowQuery(ctx context.Context) (string, error)` that simulates a 5-second database query. In main, create a context with a 1-second timeout. Call `slowQuery` and handle the timeout error by printing `"query timed out"`.

**Expected output:**
```
query timed out: context deadline exceeded
```

> [!hint]- Hint
> Use `context.WithTimeout(context.Background(), 1*time.Second)`. Inside `slowQuery`, use `select` on `ctx.Done()` vs a `time.After` channel.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"context"
> 	"fmt"
> 	"time"
> )
> 
> func slowQuery(ctx context.Context) (string, error) {
> 	select {
> 	case <-time.After(5 * time.Second):
> 		return "data from db", nil
> 	case <-ctx.Done():
> 		return "", ctx.Err()
> 	}
> }
> 
> func main() {
> 	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
> 	defer cancel()
> 
> 	result, err := slowQuery(ctx)
> 	if err != nil {
> 		fmt.Println("query timed out:", err)
> 		return
> 	}
> 	fmt.Println("result:", result)
> }
> ```

---

## P7: Context with Cancel - Propagate Cancellation

Launch 3 goroutines that each run an infinite loop printing `"worker <id> working"` every 200ms. After 1 second, call `cancel()` on the shared context. Each worker should detect the cancellation and print `"worker <id> stopped"`.

> [!hint]- Hint
> Use `context.WithCancel`. Each worker checks `ctx.Done()` in a `select` alongside a `time.After` or `time.Ticker`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"context"
> 	"fmt"
> 	"sync"
> 	"time"
> )
> 
> func worker(ctx context.Context, id int, wg *sync.WaitGroup) {
> 	defer wg.Done()
> 	ticker := time.NewTicker(200 * time.Millisecond)
> 	defer ticker.Stop()
> 
> 	for {
> 		select {
> 		case <-ticker.C:
> 			fmt.Printf("worker %d working\n", id)
> 		case <-ctx.Done():
> 			fmt.Printf("worker %d stopped\n", id)
> 			return
> 		}
> 	}
> }
> 
> func main() {
> 	ctx, cancel := context.WithCancel(context.Background())
> 	var wg sync.WaitGroup
> 
> 	for i := 1; i <= 3; i++ {
> 		wg.Add(1)
> 		go worker(ctx, i, &wg)
> 	}
> 
> 	time.Sleep(1 * time.Second)
> 	cancel()
> 	wg.Wait()
> 	fmt.Println("all workers stopped")
> }
> ```

---

## P8: errgroup - Run Tasks, Fail on First Error

Use `golang.org/x/sync/errgroup` to run 5 tasks concurrently. Tasks 1-4 succeed after sleeping 100ms. Task 3 returns an error `"task 3 failed"`. Print the first error that occurs.

**Expected output:**
```
error: task 3 failed
```

> [!hint]- Hint
> `g, ctx := errgroup.WithContext(ctx)`. Call `g.Go(func() error { ... })` for each task. `g.Wait()` returns the first non-nil error. Other goroutines can check `ctx.Done()`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"context"
> 	"fmt"
> 	"time"
> 
> 	"golang.org/x/sync/errgroup"
> )
> 
> func main() {
> 	g, _ := errgroup.WithContext(context.Background())
> 
> 	for i := 1; i <= 5; i++ {
> 		id := i
> 		g.Go(func() error {
> 			time.Sleep(100 * time.Millisecond)
> 			if id == 3 {
> 				return fmt.Errorf("task %d failed", id)
> 			}
> 			fmt.Printf("task %d succeeded\n", id)
> 			return nil
> 		})
> 	}
> 
> 	if err := g.Wait(); err != nil {
> 		fmt.Println("error:", err)
> 	}
> }
> ```
> 
> **Note:** You need `go get golang.org/x/sync/errgroup` to use this package.

---

## P9: Pipeline with Cancellation

Build a 3-stage pipeline:
1. **gen** — produces integers 1..100
2. **square** — squares each integer
3. **filter** — keeps only values < 50

Use `context.Context` so the entire pipeline stops when main cancels after receiving 5 values.

**Expected output:**
```
1
4
9
16
25
```

> [!hint]- Hint
> Each stage is a function that takes a context and an input channel, returns an output channel. Use `select` on `ctx.Done()` in every stage. Cancel after reading 5 values.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"context"
> 	"fmt"
> )
> 
> func gen(ctx context.Context) <-chan int {
> 	out := make(chan int)
> 	go func() {
> 		defer close(out)
> 		for i := 1; i <= 100; i++ {
> 			select {
> 			case out <- i:
> 			case <-ctx.Done():
> 				return
> 			}
> 		}
> 	}()
> 	return out
> }
> 
> func square(ctx context.Context, in <-chan int) <-chan int {
> 	out := make(chan int)
> 	go func() {
> 		defer close(out)
> 		for v := range in {
> 			select {
> 			case out <- v * v:
> 			case <-ctx.Done():
> 				return
> 			}
> 		}
> 	}()
> 	return out
> }
> 
> func filter(ctx context.Context, in <-chan int, max int) <-chan int {
> 	out := make(chan int)
> 	go func() {
> 		defer close(out)
> 		for v := range in {
> 			if v < max {
> 				select {
> 				case out <- v:
> 				case <-ctx.Done():
> 					return
> 				}
> 			}
> 		}
> 	}()
> 	return out
> }
> 
> func main() {
> 	ctx, cancel := context.WithCancel(context.Background())
> 	defer cancel()
> 
> 	pipeline := filter(ctx, square(ctx, gen(ctx)), 50)
> 
> 	count := 0
> 	for v := range pipeline {
> 		fmt.Println(v)
> 		count++
> 		if count >= 5 {
> 			cancel()
> 			break
> 		}
> 	}
> }
> ```

---

## P10: Rate Limiter with time.Ticker

Write a function that processes 10 requests but limits throughput to 3 per second using `time.Ticker`. Print each request with a timestamp showing ~333ms spacing.

**Expected output (timestamps approximate):**
```
request 1  at 0ms
request 2  at 333ms
request 3  at 666ms
request 4  at 1000ms
...
```

> [!hint]- Hint
> Create `limiter := time.NewTicker(333 * time.Millisecond)`. Wait on `<-limiter.C` before processing each request.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"time"
> )
> 
> func main() {
> 	limiter := time.NewTicker(333 * time.Millisecond)
> 	defer limiter.Stop()
> 
> 	start := time.Now()
> 
> 	for i := 1; i <= 10; i++ {
> 		if i > 1 {
> 			<-limiter.C
> 		}
> 		elapsed := time.Since(start).Milliseconds()
> 		fmt.Printf("request %d  at %dms\n", i, elapsed)
> 	}
> }
> ```

---

## P11: sync.Once - Lazy Singleton

Create a `Config` struct with a `LoadConfig()` function that is expensive (simulated with a print and sleep). Use `sync.Once` to ensure `LoadConfig` runs exactly once even when called from 10 goroutines simultaneously.

**Expected output:**
```
loading config... (expensive)
config loaded (10 goroutines tried)
```

> [!hint]- Hint
> Store a package-level `var once sync.Once` and `var cfg *Config`. `GetConfig()` calls `once.Do(func() { ... })`.

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
> type Config struct {
> 	DBHost string
> }
> 
> var (
> 	once     sync.Once
> 	instance *Config
> )
> 
> func GetConfig() *Config {
> 	once.Do(func() {
> 		fmt.Println("loading config... (expensive)")
> 		time.Sleep(100 * time.Millisecond)
> 		instance = &Config{DBHost: "localhost:5432"}
> 	})
> 	return instance
> }
> 
> func main() {
> 	var wg sync.WaitGroup
> 
> 	for i := 0; i < 10; i++ {
> 		wg.Add(1)
> 		go func() {
> 			defer wg.Done()
> 			_ = GetConfig()
> 		}()
> 	}
> 
> 	wg.Wait()
> 	fmt.Println("config loaded (10 goroutines tried)")
> }
> ```

---

## P12: sync.Map vs Mutex Map

Write two implementations of a concurrent map:
1. Using `sync.Map`
2. Using a regular `map` with `sync.RWMutex`

Run a benchmark where 100 goroutines perform 1000 reads and 100 writes each. Print timing for both.

> [!hint]- Hint
> Use `testing.Benchmark` or just `time.Now()`/`time.Since()` for a quick comparison. `sync.Map` uses `Load`, `Store`, `Range`. The mutex version wraps a regular map.

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
> // Mutex-based map
> type MutexMap struct {
> 	mu sync.RWMutex
> 	m  map[string]int
> }
> 
> func NewMutexMap() *MutexMap {
> 	return &MutexMap{m: make(map[string]int)}
> }
> 
> func (mm *MutexMap) Get(key string) (int, bool) {
> 	mm.mu.RLock()
> 	defer mm.mu.RUnlock()
> 	v, ok := mm.m[key]
> 	return v, ok
> }
> 
> func (mm *MutexMap) Set(key string, val int) {
> 	mm.mu.Lock()
> 	defer mm.mu.Unlock()
> 	mm.m[key] = val
> }
> 
> func benchMutexMap() time.Duration {
> 	mm := NewMutexMap()
> 	var wg sync.WaitGroup
> 
> 	start := time.Now()
> 	for g := 0; g < 100; g++ {
> 		wg.Add(1)
> 		go func(id int) {
> 			defer wg.Done()
> 			key := fmt.Sprintf("key-%d", id%10)
> 			for i := 0; i < 1000; i++ {
> 				mm.Get(key) // read
> 			}
> 			for i := 0; i < 100; i++ {
> 				mm.Set(key, i) // write
> 			}
> 		}(g)
> 	}
> 	wg.Wait()
> 	return time.Since(start)
> }
> 
> func benchSyncMap() time.Duration {
> 	var sm sync.Map
> 	var wg sync.WaitGroup
> 
> 	start := time.Now()
> 	for g := 0; g < 100; g++ {
> 		wg.Add(1)
> 		go func(id int) {
> 			defer wg.Done()
> 			key := fmt.Sprintf("key-%d", id%10)
> 			for i := 0; i < 1000; i++ {
> 				sm.Load(key) // read
> 			}
> 			for i := 0; i < 100; i++ {
> 				sm.Store(key, i) // write
> 			}
> 		}(g)
> 	}
> 	wg.Wait()
> 	return time.Since(start)
> }
> 
> func main() {
> 	fmt.Println("MutexMap:", benchMutexMap())
> 	fmt.Println("sync.Map:", benchSyncMap())
> }
> ```

---

## P13: Publish / Subscribe

Implement a `Broker` that supports:
- `Subscribe(topic string) <-chan string` — returns a channel for receiving messages on a topic
- `Publish(topic string, msg string)` — sends msg to all subscribers of that topic
- `Unsubscribe(topic string, ch <-chan string)` — removes a subscriber

Test with 2 subscribers on topic `"news"`. Publish 3 messages. Both subscribers should receive all 3.

> [!hint]- Hint
> The `Broker` holds `map[string][]chan string`. `Publish` iterates over the slice and sends to each channel. Use a mutex to protect the map.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sync"
> )
> 
> type Broker struct {
> 	mu   sync.RWMutex
> 	subs map[string][]chan string
> }
> 
> func NewBroker() *Broker {
> 	return &Broker{subs: make(map[string][]chan string)}
> }
> 
> func (b *Broker) Subscribe(topic string) <-chan string {
> 	b.mu.Lock()
> 	defer b.mu.Unlock()
> 	ch := make(chan string, 10)
> 	b.subs[topic] = append(b.subs[topic], ch)
> 	return ch
> }
> 
> func (b *Broker) Publish(topic string, msg string) {
> 	b.mu.RLock()
> 	defer b.mu.RUnlock()
> 	for _, ch := range b.subs[topic] {
> 		ch <- msg
> 	}
> }
> 
> func (b *Broker) Close(topic string) {
> 	b.mu.Lock()
> 	defer b.mu.Unlock()
> 	for _, ch := range b.subs[topic] {
> 		close(ch)
> 	}
> 	delete(b.subs, topic)
> }
> 
> func main() {
> 	broker := NewBroker()
> 
> 	sub1 := broker.Subscribe("news")
> 	sub2 := broker.Subscribe("news")
> 
> 	var wg sync.WaitGroup
> 
> 	// Reader goroutines
> 	for i, sub := range []<-chan string{sub1, sub2} {
> 		wg.Add(1)
> 		go func(id int, ch <-chan string) {
> 			defer wg.Done()
> 			for msg := range ch {
> 				fmt.Printf("subscriber %d: %s\n", id+1, msg)
> 			}
> 		}(i, sub)
> 	}
> 
> 	// Publish 3 messages
> 	broker.Publish("news", "breaking: Go 1.24 released")
> 	broker.Publish("news", "update: new generics features")
> 	broker.Publish("news", "tip: use context for cancellation")
> 
> 	broker.Close("news")
> 	wg.Wait()
> }
> ```

---

## P14: In-Memory Job Queue with Workers, Retries, and Timeout

Build a job queue system with:
- A `Job` struct: `ID`, `Payload string`, `Retries int`, `MaxRetries int`
- 3 worker goroutines reading from a job channel
- A `process(job Job) error` function that fails 50% of the time
- If a job fails and has retries remaining, re-enqueue it
- If processing takes more than 500ms, cancel via context and count as a failure
- Print final status of each job (succeeded / permanently failed)

> [!hint]- Hint
> Use a buffered channel as the queue. Workers wrap `process` in `context.WithTimeout`. On failure, check `job.Retries < job.MaxRetries`, increment retries, and re-send to the queue. Track results in a shared map (protected by mutex).

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"context"
> 	"fmt"
> 	"math/rand"
> 	"sync"
> 	"time"
> )
> 
> type Job struct {
> 	ID         int
> 	Payload    string
> 	Retries    int
> 	MaxRetries int
> }
> 
> type Result struct {
> 	JobID   int
> 	Success bool
> 	Tries   int
> }
> 
> func process(ctx context.Context, job Job) error {
> 	// Simulate variable processing time
> 	duration := time.Duration(100+rand.Intn(600)) * time.Millisecond
> 
> 	select {
> 	case <-time.After(duration):
> 		// 50% failure rate
> 		if rand.Float64() < 0.5 {
> 			return fmt.Errorf("random failure")
> 		}
> 		return nil
> 	case <-ctx.Done():
> 		return ctx.Err()
> 	}
> }
> 
> func worker(id int, jobs chan Job, results chan<- Result, wg *sync.WaitGroup) {
> 	defer wg.Done()
> 	for job := range jobs {
> 		ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
> 		err := process(ctx, job)
> 		cancel()
> 
> 		if err != nil {
> 			if job.Retries < job.MaxRetries {
> 				job.Retries++
> 				fmt.Printf("worker %d: job %d failed (attempt %d/%d), retrying\n",
> 					id, job.ID, job.Retries, job.MaxRetries+1)
> 				// Re-enqueue (non-blocking with buffered channel)
> 				select {
> 				case jobs <- job:
> 				default:
> 					// Queue full, report failure
> 					results <- Result{JobID: job.ID, Success: false, Tries: job.Retries + 1}
> 				}
> 			} else {
> 				fmt.Printf("worker %d: job %d permanently failed after %d attempts\n",
> 					id, job.ID, job.Retries+1)
> 				results <- Result{JobID: job.ID, Success: false, Tries: job.Retries + 1}
> 			}
> 		} else {
> 			fmt.Printf("worker %d: job %d succeeded (attempt %d)\n",
> 				id, job.ID, job.Retries+1)
> 			results <- Result{JobID: job.ID, Success: true, Tries: job.Retries + 1}
> 		}
> 	}
> }
> 
> func main() {
> 	rand.New(rand.NewSource(time.Now().UnixNano()))
> 
> 	const numJobs = 5
> 	const numWorkers = 3
> 
> 	jobs := make(chan Job, numJobs*4) // extra capacity for retries
> 	results := make(chan Result, numJobs)
> 
> 	var wg sync.WaitGroup
> 
> 	// Start workers
> 	for w := 1; w <= numWorkers; w++ {
> 		wg.Add(1)
> 		go worker(w, jobs, results, &wg)
> 	}
> 
> 	// Send jobs
> 	for j := 1; j <= numJobs; j++ {
> 		jobs <- Job{
> 			ID:         j,
> 			Payload:    fmt.Sprintf("task-%d", j),
> 			MaxRetries: 2,
> 		}
> 	}
> 
> 	// Collect results
> 	go func() {
> 		collected := 0
> 		for r := range results {
> 			status := "SUCCEEDED"
> 			if !r.Success {
> 				status = "FAILED"
> 			}
> 			fmt.Printf("  -> Job %d: %s after %d tries\n", r.JobID, status, r.Tries)
> 			collected++
> 			if collected >= numJobs {
> 				close(jobs) // signal workers to stop
> 				return
> 			}
> 		}
> 	}()
> 
> 	wg.Wait()
> 	fmt.Println("\nAll jobs processed.")
> }
> ```
