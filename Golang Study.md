---
title: Golang Study
date: 2026-04-06
tags:
  - golang
  - programming
  - study
aliases:
  - Go Study
  - Go Language
status: in-progress
---

# Golang Study

> [!tip] Why Go?
> Go is statically typed, compiled, and designed for simplicity and performance. Great for systems programming, CLIs, and backend services.

## Core Concepts

### Variables & Types

```go
// Declaration
var x int = 10
y := 20  // short declaration (inferred)

// Basic types
var s string = "hello"
var b bool = true
var f float64 = 3.14
var bs []byte = []byte("data")
```

### Functions

```go
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

// Variadic
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
```

### Structs & Methods

```go
type Person struct {
    Name string
    Age  int
}

// Method with value receiver
func (p Person) Greet() string {
    return "Hi, I'm " + p.Name
}

// Method with pointer receiver (can mutate)
func (p *Person) Birthday() {
    p.Age++
}
```

### Interfaces

```go
type Animal interface {
    Sound() string
}

type Dog struct{}

func (d Dog) Sound() string { return "Woof" }

// Usage
var a Animal = Dog{}
fmt.Println(a.Sound())
```

### Goroutines & Channels

```go
// Goroutine
go func() {
    fmt.Println("running concurrently")
}()

// Channel
ch := make(chan int)
go func() { ch <- 42 }()
val := <-ch

// Buffered channel
bch := make(chan string, 3)

// Select
select {
case msg := <-ch1:
    fmt.Println(msg)
case msg := <-ch2:
    fmt.Println(msg)
default:
    fmt.Println("no message")
}
```

### Error Handling

```go
result, err := someFunction()
if err != nil {
    log.Fatal(err)
}

// Custom error
type MyError struct {
    Code    int
    Message string
}

func (e *MyError) Error() string {
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}
```

### Slices & Maps

```go
// Slices
s := []int{1, 2, 3}
s = append(s, 4)
sub := s[1:3]  // [2, 3]

// Make with length and capacity
s2 := make([]int, 0, 10)

// Maps
m := map[string]int{"a": 1, "b": 2}
m["c"] = 3
val, ok := m["a"]  // ok is false if key missing
delete(m, "b")
```

### Defer, Panic, Recover

```go
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)
        }
    }()
    return a / b, nil
}

// Defer runs LIFO on function exit
defer fmt.Println("last")
defer fmt.Println("first")
```

## Standard Library Highlights

| Package | Use |
|---|---|
| `fmt` | Formatted I/O |
| `os` | OS interaction, file I/O |
| `io` | I/O primitives |
| `net/http` | HTTP client/server |
| `encoding/json` | JSON encode/decode |
| `sync` | Mutexes, WaitGroups |
| `context` | Cancellation, deadlines |
| `strings` | String manipulation |
| `strconv` | String conversions |
| `time` | Time and duration |
| `log` | Logging |
| `testing` | Unit tests |

## Patterns

### WaitGroup

```go
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Println("worker", id)
    }(i)
}
wg.Wait()
```

### Context with Timeout

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := http.DefaultClient.Do(req)
```

### JSON Marshal/Unmarshal

```go
type User struct {
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`
}

// Marshal
data, _ := json.Marshal(User{Name: "Alice"})

// Unmarshal
var u User
json.Unmarshal(data, &u)
```

## Tools & Commands

```bash
go mod init <module-name>   # init module
go get <package>            # add dependency
go run main.go              # run
go build ./...              # build
go test ./...               # test
go vet ./...                # static analysis
gofmt -w .                  # format
golangci-lint run           # lint (external)
```

## Resources

- [Official Go Tour](https://go.dev/tour/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go by Example](https://gobyexample.com)
- [pkg.go.dev](https://pkg.go.dev) — standard library docs

## Topics to Explore

- [ ] Generics (Go 1.18+)
- [ ] `sync.Map` vs regular map
- [ ] Race detector (`go test -race`)
- [ ] Profiling with `pprof`
- [ ] HTTP server patterns
- [ ] gRPC with Go
- [ ] Testing strategies (table-driven tests)
