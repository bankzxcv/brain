---
title: "Go Practice: Goroutines and Channels"
date: 2026-04-07
tags:
  - golang
  - practice
  - concurrency
parent: "[[Golang Study]]"
---

# Goroutines and Channels

Progressive drills for goroutines, channels, and the select statement. Write each solution from scratch.

---

## P1: Hello from a Goroutine

Launch a goroutine that prints `"hello from goroutine"`. Use `time.Sleep` in main to wait long enough for it to finish.

**Expected output:**
```
main: starting
hello from goroutine
main: done
```

> [!hint]- Hint
> Use the `go` keyword before a function call. `time.Sleep(100 * time.Millisecond)` gives the goroutine time to run.

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
> 	fmt.Println("main: starting")
> 
> 	go func() {
> 		fmt.Println("hello from goroutine")
> 	}()
> 
> 	time.Sleep(100 * time.Millisecond)
> 	fmt.Println("main: done")
> }
> ```

---

## P2: Done Channel Instead of Sleep

Refactor P1: instead of `time.Sleep`, use a `chan bool` to signal that the goroutine has finished.

**Expected output:**
```
main: starting
hello from goroutine
main: done
```

> [!hint]- Hint
> Create `done := make(chan bool)`. Send `true` at the end of the goroutine. Receive from `done` in main before printing "done".

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	fmt.Println("main: starting")
> 
> 	done := make(chan bool)
> 
> 	go func() {
> 		fmt.Println("hello from goroutine")
> 		done <- true
> 	}()
> 
> 	<-done
> 	fmt.Println("main: done")
> }
> ```

---

## P3: Send and Receive Values

Create an unbuffered channel of `int`. Launch a goroutine that sends the numbers 1 through 5, then closes the channel. In main, receive and print each value.

**Expected output:**
```
1
2
3
4
5
```

> [!hint]- Hint
> The sender goroutine should `close(ch)` after the loop. The receiver can use `for v := range ch` to read until closed.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	ch := make(chan int)
> 
> 	go func() {
> 		for i := 1; i <= 5; i++ {
> 			ch <- i
> 		}
> 		close(ch)
> 	}()
> 
> 	for v := range ch {
> 		fmt.Println(v)
> 	}
> }
> ```

---

## P4: Buffered Channel

Create a **buffered** channel with capacity 5. In a single goroutine (no extra goroutines), send the values 10, 20, 30, 40, 50 into it without blocking. Then read and print them all.

**Expected output:**
```
10
20
30
40
50
```

> [!hint]- Hint
> `make(chan int, 5)` creates a buffered channel. You can send up to 5 values before any receive, because the buffer absorbs them.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	ch := make(chan int, 5)
> 
> 	ch <- 10
> 	ch <- 20
> 	ch <- 30
> 	ch <- 40
> 	ch <- 50
> 	close(ch)
> 
> 	for v := range ch {
> 		fmt.Println(v)
> 	}
> }
> ```

---

## P5: Producer / Consumer with Range

Write a `producer` function that sends squares of 1..10 on a channel, then closes it. Write a `consumer` function that ranges over the channel and prints each value. Wire them together in main.

**Expected output:**
```
1
4
9
16
25
36
49
64
81
100
```

> [!hint]- Hint
> Run `producer` as a goroutine so that the consumer can range over the channel in main. The producer must close the channel when done.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func producer(ch chan<- int) {
> 	for i := 1; i <= 10; i++ {
> 		ch <- i * i
> 	}
> 	close(ch)
> }
> 
> func consumer(ch <-chan int) {
> 	for v := range ch {
> 		fmt.Println(v)
> 	}
> }
> 
> func main() {
> 	ch := make(chan int)
> 	go producer(ch)
> 	consumer(ch)
> }
> ```

---

## P6: Directional Channels in Function Signatures

Write two functions:
- `generate(ch chan<- string)` — sends `"a"`, `"b"`, `"c"` then closes.
- `print All(ch <-chan string)` — receives and prints all values.

The compiler should reject any attempt to read from a send-only channel or write to a receive-only channel.

**Expected output:**
```
a
b
c
```

> [!hint]- Hint
> `chan<-` is send-only, `<-chan` is receive-only. A bidirectional `chan string` can be passed to either.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func generate(ch chan<- string) {
> 	ch <- "a"
> 	ch <- "b"
> 	ch <- "c"
> 	close(ch)
> }
> 
> func printAll(ch <-chan string) {
> 	for v := range ch {
> 		fmt.Println(v)
> 	}
> }
> 
> func main() {
> 	ch := make(chan string)
> 	go generate(ch)
> 	printAll(ch)
> }
> ```

---

## P7: Select Statement

Create two channels. Launch two goroutines: one sends `"from ch1"` after 100ms, the other sends `"from ch2"` after 200ms. Use a `select` in a loop to print messages as they arrive. Exit after both have sent.

**Expected output (order may vary):**
```
from ch1
from ch2
```

> [!hint]- Hint
> Use a counter or check the `ok` value on receive. `select` picks whichever channel is ready first.

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
> 	ch1 := make(chan string)
> 	ch2 := make(chan string)
> 
> 	go func() {
> 		time.Sleep(100 * time.Millisecond)
> 		ch1 <- "from ch1"
> 	}()
> 
> 	go func() {
> 		time.Sleep(200 * time.Millisecond)
> 		ch2 <- "from ch2"
> 	}()
> 
> 	received := 0
> 	for received < 2 {
> 		select {
> 		case msg := <-ch1:
> 			fmt.Println(msg)
> 			received++
> 		case msg := <-ch2:
> 			fmt.Println(msg)
> 			received++
> 		}
> 	}
> }
> ```

---

## P8: Select with Timeout

Launch a goroutine that takes 2 seconds to compute a result. Use `select` with `time.After(1 * time.Second)` to implement a timeout. Print `"timeout!"` if the operation takes too long.

**Expected output:**
```
timeout!
```

> [!hint]- Hint
> `time.After(d)` returns a `<-chan time.Time` that fires once after duration `d`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"time"
> )
> 
> func slowOperation(ch chan<- string) {
> 	time.Sleep(2 * time.Second)
> 	ch <- "result"
> }
> 
> func main() {
> 	ch := make(chan string)
> 	go slowOperation(ch)
> 
> 	select {
> 	case result := <-ch:
> 		fmt.Println("got:", result)
> 	case <-time.After(1 * time.Second):
> 		fmt.Println("timeout!")
> 	}
> }
> ```

---

## P9: Done / Quit Channel for Graceful Shutdown

Write a goroutine that generates random numbers on a channel forever. Main reads 5 numbers and then signals a `quit` channel. The generator should detect the quit signal using `select` and print `"generator: shutting down"` before returning.

**Expected output (numbers will vary):**
```
received: 81
received: 87
received: 47
received: 59
received: 81
generator: shutting down
```

> [!hint]- Hint
> Inside the generator, use `select` with two cases: one to send a value, one to receive from `quit`. In main, after reading 5 values, send on `quit` and give the goroutine time to print.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"math/rand"
> )
> 
> func generator(out chan<- int, quit <-chan struct{}) {
> 	for {
> 		select {
> 		case out <- rand.Intn(100):
> 		case <-quit:
> 			fmt.Println("generator: shutting down")
> 			return
> 		}
> 	}
> }
> 
> func main() {
> 	out := make(chan int)
> 	quit := make(chan struct{})
> 
> 	go generator(out, quit)
> 
> 	for i := 0; i < 5; i++ {
> 		fmt.Println("received:", <-out)
> 	}
> 
> 	quit <- struct{}{}
> 	// Give generator time to print its shutdown message.
> 	// In production code you'd use a done channel instead.
> 	<-out // This will block because generator returned — 
> 	      // actually, let's use a proper done signal:
> }
> ```
> 
> Cleaner version with a done acknowledgement:
> 
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"math/rand"
> )
> 
> func generator(out chan<- int, quit <-chan struct{}, done chan<- struct{}) {
> 	defer func() { done <- struct{}{} }()
> 	for {
> 		select {
> 		case out <- rand.Intn(100):
> 		case <-quit:
> 			fmt.Println("generator: shutting down")
> 			return
> 		}
> 	}
> }
> 
> func main() {
> 	out := make(chan int)
> 	quit := make(chan struct{})
> 	done := make(chan struct{})
> 
> 	go generator(out, quit, done)
> 
> 	for i := 0; i < 5; i++ {
> 		fmt.Println("received:", <-out)
> 	}
> 
> 	quit <- struct{}{}
> 	<-done
> }
> ```

---

## P10: Fan-Out

Write a `worker(id int, jobs <-chan int, results chan<- string)` function. Launch 3 workers. Send 9 jobs (numbers 1..9) into the jobs channel. Collect and print all 9 results. Each result should be `"worker <id> processed <job>"`.

**Expected output (order may vary):**
```
worker 1 processed 1
worker 2 processed 2
worker 3 processed 3
worker 1 processed 4
...
```

> [!hint]- Hint
> All workers read from the same `jobs` channel. Close `jobs` after sending all values. Use a loop to receive exactly 9 results.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sync"
> )
> 
> func worker(id int, jobs <-chan int, results chan<- string, wg *sync.WaitGroup) {
> 	defer wg.Done()
> 	for job := range jobs {
> 		results <- fmt.Sprintf("worker %d processed %d", id, job)
> 	}
> }
> 
> func main() {
> 	jobs := make(chan int, 9)
> 	results := make(chan string, 9)
> 	var wg sync.WaitGroup
> 
> 	for w := 1; w <= 3; w++ {
> 		wg.Add(1)
> 		go worker(w, jobs, results, &wg)
> 	}
> 
> 	for j := 1; j <= 9; j++ {
> 		jobs <- j
> 	}
> 	close(jobs)
> 
> 	go func() {
> 		wg.Wait()
> 		close(results)
> 	}()
> 
> 	for r := range results {
> 		fmt.Println(r)
> 	}
> }
> ```

---

## P11: Fan-In

Write two generator functions: `genEvens` sends 2, 4, 6, 8, 10 and `genOdds` sends 1, 3, 5, 7, 9 (each on its own channel). Write a `fanIn` function that merges both channels into one output channel. Print all 10 values from the merged channel.

**Expected output (order between evens/odds may vary):**
```
2
1
4
3
...
```

> [!hint]- Hint
> `fanIn` launches one goroutine per input channel. Each goroutine forwards values to the output channel. Use a `sync.WaitGroup` to know when to close the output.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sync"
> )
> 
> func genEvens(ch chan<- int) {
> 	for i := 2; i <= 10; i += 2 {
> 		ch <- i
> 	}
> 	close(ch)
> }
> 
> func genOdds(ch chan<- int) {
> 	for i := 1; i <= 9; i += 2 {
> 		ch <- i
> 	}
> 	close(ch)
> }
> 
> func fanIn(channels ...<-chan int) <-chan int {
> 	out := make(chan int)
> 	var wg sync.WaitGroup
> 
> 	for _, ch := range channels {
> 		wg.Add(1)
> 		go func(c <-chan int) {
> 			defer wg.Done()
> 			for v := range c {
> 				out <- v
> 			}
> 		}(ch)
> 	}
> 
> 	go func() {
> 		wg.Wait()
> 		close(out)
> 	}()
> 
> 	return out
> }
> 
> func main() {
> 	evens := make(chan int)
> 	odds := make(chan int)
> 
> 	go genEvens(evens)
> 	go genOdds(odds)
> 
> 	merged := fanIn(evens, odds)
> 	for v := range merged {
> 		fmt.Println(v)
> 	}
> }
> ```

---

## P12: Ping-Pong

Create two goroutines: `ping` and `pong`. They pass an `int` back and forth on two channels. Start with 0. Each goroutine increments the value and prints its name and the new value. Stop after the value reaches 10.

**Expected output:**
```
ping: 1
pong: 2
ping: 3
pong: 4
ping: 5
pong: 6
ping: 7
pong: 8
ping: 9
pong: 10
```

> [!hint]- Hint
> Use two channels: `pingCh` and `pongCh`. `ping` reads from `pingCh`, increments, prints, sends to `pongCh`. `pong` does the reverse. Seed the first channel with 0.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	pingCh := make(chan int)
> 	pongCh := make(chan int)
> 	done := make(chan struct{})
> 
> 	// ping goroutine
> 	go func() {
> 		for v := range pingCh {
> 			v++
> 			fmt.Printf("ping: %d\n", v)
> 			if v >= 10 {
> 				close(done)
> 				return
> 			}
> 			pongCh <- v
> 		}
> 	}()
> 
> 	// pong goroutine
> 	go func() {
> 		for v := range pongCh {
> 			v++
> 			fmt.Printf("pong: %d\n", v)
> 			if v >= 10 {
> 				close(done)
> 				return
> 			}
> 			pingCh <- v
> 		}
> 	}()
> 
> 	pingCh <- 0 // start the rally
> 	<-done
> }
> ```

---

## P13: Stoppable Ticker

Create a goroutine that prints `"tick"` every 500ms using `time.NewTicker`. After 2.5 seconds, stop the ticker gracefully and print `"ticker stopped"`.

**Expected output:**
```
tick
tick
tick
tick
tick
ticker stopped
```

> [!hint]- Hint
> Use `time.NewTicker(500ms)` and `time.After(2500ms)` with a `select` loop. Call `ticker.Stop()` before exiting.

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
> 	ticker := time.NewTicker(500 * time.Millisecond)
> 	defer ticker.Stop()
> 
> 	timeout := time.After(2600 * time.Millisecond)
> 
> 	for {
> 		select {
> 		case <-ticker.C:
> 			fmt.Println("tick")
> 		case <-timeout:
> 			fmt.Println("ticker stopped")
> 			return
> 		}
> 	}
> }
> ```

---

## P14: Fibonacci Generator

Write a `fibonacci` function that returns a `<-chan int`. It should launch a goroutine that sends fibonacci numbers (0, 1, 1, 2, 3, 5, 8, ...) on the channel indefinitely. Main reads the first 10 values and prints them. Use a `quit` channel so the generator doesn't leak.

**Expected output:**
```
0
1
1
2
3
5
8
13
21
34
```

> [!hint]- Hint
> The generator pattern: return a receive-only channel. Inside, use `select` to either send the next value or detect a quit signal.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func fibonacci(quit <-chan struct{}) <-chan int {
> 	ch := make(chan int)
> 	go func() {
> 		defer close(ch)
> 		a, b := 0, 1
> 		for {
> 			select {
> 			case ch <- a:
> 				a, b = b, a+b
> 			case <-quit:
> 				return
> 			}
> 		}
> 	}()
> 	return ch
> }
> 
> func main() {
> 	quit := make(chan struct{})
> 	fib := fibonacci(quit)
> 
> 	for i := 0; i < 10; i++ {
> 		fmt.Println(<-fib)
> 	}
> 
> 	close(quit)
> }
> ```

---

## P15: Concurrent-Safe Counter (No Mutex)

Build a counter that can be safely incremented and read from multiple goroutines, using **only channels** (no `sync.Mutex`). The counter should support `Increment()` and `Value() int` operations. Launch 1000 goroutines that each call `Increment()` once, then print the final value.

**Expected output:**
```
final value: 1000
```

> [!hint]- Hint
> Run a single goroutine that owns the counter value. It listens on an `increment` channel and a `value` request/response channel pair. All mutations go through that single goroutine.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"sync"
> )
> 
> type Counter struct {
> 	incCh chan struct{}
> 	valCh chan int
> }
> 
> func NewCounter() *Counter {
> 	c := &Counter{
> 		incCh: make(chan struct{}),
> 		valCh: make(chan int),
> 	}
> 	go c.run()
> 	return c
> }
> 
> func (c *Counter) run() {
> 	count := 0
> 	for {
> 		select {
> 		case <-c.incCh:
> 			count++
> 		case c.valCh <- count:
> 		}
> 	}
> }
> 
> func (c *Counter) Increment() {
> 	c.incCh <- struct{}{}
> }
> 
> func (c *Counter) Value() int {
> 	return <-c.valCh
> }
> 
> func main() {
> 	counter := NewCounter()
> 	var wg sync.WaitGroup
> 
> 	for i := 0; i < 1000; i++ {
> 		wg.Add(1)
> 		go func() {
> 			defer wg.Done()
> 			counter.Increment()
> 		}()
> 	}
> 
> 	wg.Wait()
> 	fmt.Println("final value:", counter.Value())
> }
> ```
