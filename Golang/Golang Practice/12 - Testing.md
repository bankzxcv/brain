---
title: "Go Practice: Testing"
date: 2026-04-07
tags:
  - golang
  - practice
  - testing
parent: "[[Golang Study]]"
---

# Testing

Progressive drills for Go testing: table-driven tests, benchmarks, httptest, race detection, and more. Write each solution from scratch.

---

## P1: Basic Test for Add

Write a function `Add(a, b int) int` in `math.go`. Write a test in `math_test.go` that verifies `Add(2, 3) == 5`.

Run with: `go test -v ./...`

> [!hint]- Hint
> Test functions must start with `Test` and take `*testing.T`. Use `t.Errorf` to report failures.

> [!success]- Solution
> `math.go`:
> ```go
> package math
> 
> func Add(a, b int) int {
> 	return a + b
> }
> ```
> 
> `math_test.go`:
> ```go
> package math
> 
> import "testing"
> 
> func TestAdd(t *testing.T) {
> 	got := Add(2, 3)
> 	want := 5
> 	if got != want {
> 		t.Errorf("Add(2, 3) = %d, want %d", got, want)
> 	}
> }
> ```

---

## P2: Table-Driven Tests

Write table-driven tests for a `Divide(a, b float64) (float64, error)` function that returns an error when dividing by zero.

Test cases:
| a   | b   | want | wantErr |
|-----|-----|------|---------|
| 10  | 2   | 5.0  | false   |
| -6  | 3   | -2.0 | false   |
| 7   | 0   | 0    | true    |
| 0   | 5   | 0.0  | false   |

> [!hint]- Hint
> Define a slice of anonymous structs: `[]struct{ a, b, want float64; wantErr bool }`. Loop and call `t.Run` for each case.

> [!success]- Solution
> `calc.go`:
> ```go
> package calc
> 
> import "errors"
> 
> func Divide(a, b float64) (float64, error) {
> 	if b == 0 {
> 		return 0, errors.New("division by zero")
> 	}
> 	return a / b, nil
> }
> ```
> 
> `calc_test.go`:
> ```go
> package calc
> 
> import (
> 	"fmt"
> 	"testing"
> )
> 
> func TestDivide(t *testing.T) {
> 	tests := []struct {
> 		a, b    float64
> 		want    float64
> 		wantErr bool
> 	}{
> 		{10, 2, 5.0, false},
> 		{-6, 3, -2.0, false},
> 		{7, 0, 0, true},
> 		{0, 5, 0.0, false},
> 	}
> 
> 	for _, tt := range tests {
> 		name := fmt.Sprintf("%.0f/%.0f", tt.a, tt.b)
> 		t.Run(name, func(t *testing.T) {
> 			got, err := Divide(tt.a, tt.b)
> 			if (err != nil) != tt.wantErr {
> 				t.Errorf("Divide(%.1f, %.1f) error = %v, wantErr %v", tt.a, tt.b, err, tt.wantErr)
> 				return
> 			}
> 			if !tt.wantErr && got != tt.want {
> 				t.Errorf("Divide(%.1f, %.1f) = %.1f, want %.1f", tt.a, tt.b, got, tt.want)
> 			}
> 		})
> 	}
> }
> ```

---

## P3: Subtests with t.Run

Write a `Greet(name, lang string) string` function that returns:
- `"Hello, <name>!"` for `"en"`
- `"Hola, <name>!"` for `"es"`
- `"Bonjour, <name>!"` for `"fr"`
- `"Hello, <name>!"` for unknown languages

Group subtests by language using `t.Run`.

> [!hint]- Hint
> Use nested `t.Run`: outer level groups by language, inner level tests specific names.

> [!success]- Solution
> `greet.go`:
> ```go
> package greet
> 
> import "fmt"
> 
> func Greet(name, lang string) string {
> 	switch lang {
> 	case "es":
> 		return fmt.Sprintf("Hola, %s!", name)
> 	case "fr":
> 		return fmt.Sprintf("Bonjour, %s!", name)
> 	default:
> 		return fmt.Sprintf("Hello, %s!", name)
> 	}
> }
> ```
> 
> `greet_test.go`:
> ```go
> package greet
> 
> import "testing"
> 
> func TestGreet(t *testing.T) {
> 	t.Run("English", func(t *testing.T) {
> 		t.Run("Alice", func(t *testing.T) {
> 			if got := Greet("Alice", "en"); got != "Hello, Alice!" {
> 				t.Errorf("got %q", got)
> 			}
> 		})
> 		t.Run("Bob", func(t *testing.T) {
> 			if got := Greet("Bob", "en"); got != "Hello, Bob!" {
> 				t.Errorf("got %q", got)
> 			}
> 		})
> 	})
> 
> 	t.Run("Spanish", func(t *testing.T) {
> 		if got := Greet("Carlos", "es"); got != "Hola, Carlos!" {
> 			t.Errorf("got %q", got)
> 		}
> 	})
> 
> 	t.Run("French", func(t *testing.T) {
> 		if got := Greet("Marie", "fr"); got != "Bonjour, Marie!" {
> 			t.Errorf("got %q", got)
> 		}
> 	})
> 
> 	t.Run("Unknown defaults to English", func(t *testing.T) {
> 		if got := Greet("Test", "jp"); got != "Hello, Test!" {
> 			t.Errorf("got %q", got)
> 		}
> 	})
> }
> ```

---

## P4: Test Helper with t.Helper()

Write a test helper `assertEq[T comparable](t *testing.T, got, want T)` that calls `t.Helper()` so that failures point to the calling line. Use it in a test for a `Max(a, b int) int` function.

> [!hint]- Hint
> `t.Helper()` must be the first call in the helper function. This makes the error report show the line in the test function, not inside the helper.

> [!success]- Solution
> `max.go`:
> ```go
> package max
> 
> func Max(a, b int) int {
> 	if a > b {
> 		return a
> 	}
> 	return b
> }
> ```
> 
> `max_test.go`:
> ```go
> package max
> 
> import "testing"
> 
> func assertEqual[T comparable](t *testing.T, got, want T) {
> 	t.Helper()
> 	if got != want {
> 		t.Errorf("got %v, want %v", got, want)
> 	}
> }
> 
> func TestMax(t *testing.T) {
> 	assertEqual(t, Max(1, 2), 2)
> 	assertEqual(t, Max(5, 3), 5)
> 	assertEqual(t, Max(-1, -5), -1)
> 	assertEqual(t, Max(7, 7), 7)
> }
> ```

---

## P5: Testing Error Cases

Write a `ParseAge(s string) (int, error)` function that:
- Returns an error if the string is not a valid integer
- Returns an error if the age is negative
- Returns an error if the age is > 150

Write tests that verify the correct error message is returned for each case.

> [!hint]- Hint
> Use `errors.Is` or check `err.Error()` contains the expected substring. Consider defining sentinel errors like `var ErrNegativeAge = errors.New("age cannot be negative")`.

> [!success]- Solution
> `age.go`:
> ```go
> package age
> 
> import (
> 	"errors"
> 	"fmt"
> 	"strconv"
> )
> 
> var (
> 	ErrInvalidFormat = errors.New("invalid age format")
> 	ErrNegativeAge   = errors.New("age cannot be negative")
> 	ErrAgeTooLarge   = errors.New("age cannot exceed 150")
> )
> 
> func ParseAge(s string) (int, error) {
> 	n, err := strconv.Atoi(s)
> 	if err != nil {
> 		return 0, fmt.Errorf("%w: %s", ErrInvalidFormat, s)
> 	}
> 	if n < 0 {
> 		return 0, ErrNegativeAge
> 	}
> 	if n > 150 {
> 		return 0, ErrAgeTooLarge
> 	}
> 	return n, nil
> }
> ```
> 
> `age_test.go`:
> ```go
> package age
> 
> import (
> 	"errors"
> 	"testing"
> )
> 
> func TestParseAge(t *testing.T) {
> 	tests := []struct {
> 		name    string
> 		input   string
> 		want    int
> 		wantErr error
> 	}{
> 		{"valid", "25", 25, nil},
> 		{"zero", "0", 0, nil},
> 		{"max valid", "150", 150, nil},
> 		{"not a number", "abc", 0, ErrInvalidFormat},
> 		{"negative", "-5", 0, ErrNegativeAge},
> 		{"too large", "200", 0, ErrAgeTooLarge},
> 		{"empty string", "", 0, ErrInvalidFormat},
> 	}
> 
> 	for _, tt := range tests {
> 		t.Run(tt.name, func(t *testing.T) {
> 			got, err := ParseAge(tt.input)
> 			if tt.wantErr != nil {
> 				if !errors.Is(err, tt.wantErr) {
> 					t.Errorf("ParseAge(%q) error = %v, want %v", tt.input, err, tt.wantErr)
> 				}
> 				return
> 			}
> 			if err != nil {
> 				t.Errorf("ParseAge(%q) unexpected error: %v", tt.input, err)
> 				return
> 			}
> 			if got != tt.want {
> 				t.Errorf("ParseAge(%q) = %d, want %d", tt.input, got, tt.want)
> 			}
> 		})
> 	}
> }
> ```

---

## P6: Benchmark a Function

Write a `Fibonacci(n int) int` function (recursive or iterative). Write a benchmark for it with n=20 and n=30.

Run with: `go test -bench=. -benchmem`

> [!hint]- Hint
> Benchmark functions start with `Benchmark`, take `*testing.B`, and loop `b.N` times. Use `b.Run` for sub-benchmarks with different inputs.

> [!success]- Solution
> `fib.go`:
> ```go
> package fib
> 
> func Fibonacci(n int) int {
> 	if n <= 1 {
> 		return n
> 	}
> 	a, b := 0, 1
> 	for i := 2; i <= n; i++ {
> 		a, b = b, a+b
> 	}
> 	return b
> }
> ```
> 
> `fib_test.go`:
> ```go
> package fib
> 
> import (
> 	"fmt"
> 	"testing"
> )
> 
> func TestFibonacci(t *testing.T) {
> 	tests := []struct {
> 		n, want int
> 	}{
> 		{0, 0}, {1, 1}, {2, 1}, {5, 5}, {10, 55},
> 	}
> 	for _, tt := range tests {
> 		if got := Fibonacci(tt.n); got != tt.want {
> 			t.Errorf("Fibonacci(%d) = %d, want %d", tt.n, got, tt.want)
> 		}
> 	}
> }
> 
> func BenchmarkFibonacci(b *testing.B) {
> 	for _, n := range []int{20, 30} {
> 		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
> 			for i := 0; i < b.N; i++ {
> 				Fibonacci(n)
> 			}
> 		})
> 	}
> }
> ```

---

## P7: Setup / Teardown with t.Cleanup

Write a test that creates a temporary file, writes test data, reads it back, and verifies the content. Use `t.Cleanup` to remove the file after the test.

> [!hint]- Hint
> `os.CreateTemp("", "test-*.txt")` creates a temp file. Register `t.Cleanup(func() { os.Remove(f.Name()) })` immediately after creation.

> [!success]- Solution
> `fileutil.go`:
> ```go
> package fileutil
> 
> import "os"
> 
> func WriteAndRead(path, content string) (string, error) {
> 	if err := os.WriteFile(path, []byte(content), 0644); err != nil {
> 		return "", err
> 	}
> 	data, err := os.ReadFile(path)
> 	if err != nil {
> 		return "", err
> 	}
> 	return string(data), nil
> }
> ```
> 
> `fileutil_test.go`:
> ```go
> package fileutil
> 
> import (
> 	"os"
> 	"testing"
> )
> 
> func TestWriteAndRead(t *testing.T) {
> 	// Create temp file
> 	f, err := os.CreateTemp("", "test-*.txt")
> 	if err != nil {
> 		t.Fatal(err)
> 	}
> 	f.Close()
> 
> 	// Register cleanup
> 	t.Cleanup(func() {
> 		os.Remove(f.Name())
> 	})
> 
> 	// Test
> 	want := "hello, testing!"
> 	got, err := WriteAndRead(f.Name(), want)
> 	if err != nil {
> 		t.Fatalf("WriteAndRead failed: %v", err)
> 	}
> 	if got != want {
> 		t.Errorf("got %q, want %q", got, want)
> 	}
> }
> ```

---

## P8: HTTP Handler Test with httptest

Write an HTTP handler `HealthHandler` that returns status 200 with body `{"status": "ok"}`. Test it using `httptest.NewRecorder` and `http.NewRequest`.

> [!hint]- Hint
> Create a `httptest.NewRecorder()` and an `http.NewRequest("GET", "/health", nil)`. Call the handler directly. Check `rr.Code` and `rr.Body.String()`.

> [!success]- Solution
> `server.go`:
> ```go
> package server
> 
> import (
> 	"encoding/json"
> 	"net/http"
> )
> 
> func HealthHandler(w http.ResponseWriter, r *http.Request) {
> 	w.Header().Set("Content-Type", "application/json")
> 	json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
> }
> ```
> 
> `server_test.go`:
> ```go
> package server
> 
> import (
> 	"encoding/json"
> 	"net/http"
> 	"net/http/httptest"
> 	"testing"
> )
> 
> func TestHealthHandler(t *testing.T) {
> 	req := httptest.NewRequest(http.MethodGet, "/health", nil)
> 	rr := httptest.NewRecorder()
> 
> 	HealthHandler(rr, req)
> 
> 	// Check status code
> 	if rr.Code != http.StatusOK {
> 		t.Errorf("status = %d, want %d", rr.Code, http.StatusOK)
> 	}
> 
> 	// Check content type
> 	ct := rr.Header().Get("Content-Type")
> 	if ct != "application/json" {
> 		t.Errorf("Content-Type = %q, want application/json", ct)
> 	}
> 
> 	// Check body
> 	var body map[string]string
> 	if err := json.NewDecoder(rr.Body).Decode(&body); err != nil {
> 		t.Fatalf("failed to decode body: %v", err)
> 	}
> 	if body["status"] != "ok" {
> 		t.Errorf("body status = %q, want \"ok\"", body["status"])
> 	}
> }
> ```

---

## P9: Testing with Time (Interface Approach)

Write a `Cache` struct whose `IsExpired()` method depends on the current time. Instead of calling `time.Now()` directly, inject a `Clock` interface so tests can control time.

> [!hint]- Hint
> Define `type Clock interface { Now() time.Time }`. Create `RealClock` and `FakeClock` implementations. The `Cache` takes a `Clock` in its constructor.

> [!success]- Solution
> `cache.go`:
> ```go
> package cache
> 
> import "time"
> 
> type Clock interface {
> 	Now() time.Time
> }
> 
> type RealClock struct{}
> 
> func (RealClock) Now() time.Time { return time.Now() }
> 
> type Entry struct {
> 	Value     string
> 	ExpiresAt time.Time
> }
> 
> type Cache struct {
> 	clock   Clock
> 	entries map[string]Entry
> }
> 
> func NewCache(clock Clock) *Cache {
> 	return &Cache{
> 		clock:   clock,
> 		entries: make(map[string]Entry),
> 	}
> }
> 
> func (c *Cache) Set(key, value string, ttl time.Duration) {
> 	c.entries[key] = Entry{
> 		Value:     value,
> 		ExpiresAt: c.clock.Now().Add(ttl),
> 	}
> }
> 
> func (c *Cache) Get(key string) (string, bool) {
> 	entry, ok := c.entries[key]
> 	if !ok {
> 		return "", false
> 	}
> 	if c.clock.Now().After(entry.ExpiresAt) {
> 		delete(c.entries, key)
> 		return "", false
> 	}
> 	return entry.Value, true
> }
> ```
> 
> `cache_test.go`:
> ```go
> package cache
> 
> import (
> 	"testing"
> 	"time"
> )
> 
> type FakeClock struct {
> 	current time.Time
> }
> 
> func (f *FakeClock) Now() time.Time {
> 	return f.current
> }
> 
> func (f *FakeClock) Advance(d time.Duration) {
> 	f.current = f.current.Add(d)
> }
> 
> func TestCacheExpiration(t *testing.T) {
> 	clock := &FakeClock{current: time.Date(2026, 1, 1, 0, 0, 0, 0, time.UTC)}
> 	c := NewCache(clock)
> 
> 	c.Set("key", "value", 5*time.Minute)
> 
> 	// Before expiration
> 	val, ok := c.Get("key")
> 	if !ok || val != "value" {
> 		t.Errorf("expected cache hit, got ok=%v val=%q", ok, val)
> 	}
> 
> 	// Advance past expiration
> 	clock.Advance(6 * time.Minute)
> 
> 	val, ok = c.Get("key")
> 	if ok {
> 		t.Errorf("expected cache miss after expiration, got val=%q", val)
> 	}
> }
> ```

---

## P10: Test for Race Conditions

Write a `Counter` struct with `Increment()` and `Value() int` methods. Write a test that launches 100 goroutines incrementing concurrently. Run with `go test -race` to detect races. Fix the race with a mutex.

Run with: `go test -race -v`

> [!hint]- Hint
> First write the test WITHOUT a mutex. `go test -race` will report the race. Then add `sync.Mutex` to the `Counter`. Re-run the race detector to confirm the fix.

> [!success]- Solution
> `counter.go`:
> ```go
> package counter
> 
> import "sync"
> 
> type Counter struct {
> 	mu    sync.Mutex
> 	value int
> }
> 
> func (c *Counter) Increment() {
> 	c.mu.Lock()
> 	defer c.mu.Unlock()
> 	c.value++
> }
> 
> func (c *Counter) Value() int {
> 	c.mu.Lock()
> 	defer c.mu.Unlock()
> 	return c.value
> }
> ```
> 
> `counter_test.go`:
> ```go
> package counter
> 
> import (
> 	"sync"
> 	"testing"
> )
> 
> func TestCounterConcurrent(t *testing.T) {
> 	c := &Counter{}
> 	var wg sync.WaitGroup
> 
> 	const n = 100
> 	for i := 0; i < n; i++ {
> 		wg.Add(1)
> 		go func() {
> 			defer wg.Done()
> 			c.Increment()
> 		}()
> 	}
> 
> 	wg.Wait()
> 
> 	if got := c.Value(); got != n {
> 		t.Errorf("Value() = %d, want %d", got, n)
> 	}
> }
> ```
> 
> Run: `go test -race -v ./...` — should pass with no race detected.

---

## P11: Testable Example (ExampleXxx)

Write a `Reverse(s string) string` function. Write an `ExampleReverse` function with an `// Output:` comment so `go test` verifies the output automatically.

> [!hint]- Hint
> Example functions go in `_test.go` files. The `// Output:` comment must match the exact output. Example functions have no parameters.

> [!success]- Solution
> `stringutil.go`:
> ```go
> package stringutil
> 
> func Reverse(s string) string {
> 	runes := []rune(s)
> 	for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
> 		runes[i], runes[j] = runes[j], runes[i]
> 	}
> 	return string(runes)
> }
> ```
> 
> `stringutil_test.go`:
> ```go
> package stringutil
> 
> import "fmt"
> 
> func ExampleReverse() {
> 	fmt.Println(Reverse("hello"))
> 	fmt.Println(Reverse("Go語言"))
> 	// Output:
> 	// olleh
> 	// 言語oG
> }
> ```

---

## P12: Test Coverage

Write a `Classify(n int) string` function that returns:
- `"negative"` if n < 0
- `"zero"` if n == 0
- `"small"` if n > 0 && n <= 10
- `"large"` if n > 10

Write tests that cover only 3 of 4 branches. Run `go test -cover` to see the gap. Then add the missing test to reach 100%.

Run with: `go test -cover -coverprofile=coverage.out && go tool cover -func=coverage.out`

> [!hint]- Hint
> Start by testing negative, zero, and small. The coverage report will show the `"large"` branch is uncovered. Add a test for n=100.

> [!success]- Solution
> `classify.go`:
> ```go
> package classify
> 
> func Classify(n int) string {
> 	switch {
> 	case n < 0:
> 		return "negative"
> 	case n == 0:
> 		return "zero"
> 	case n <= 10:
> 		return "small"
> 	default:
> 		return "large"
> 	}
> }
> ```
> 
> `classify_test.go` (initially missing the "large" case):
> ```go
> package classify
> 
> import "testing"
> 
> func TestClassify(t *testing.T) {
> 	tests := []struct {
> 		input int
> 		want  string
> 	}{
> 		{-5, "negative"},
> 		{0, "zero"},
> 		{5, "small"},
> 		// Uncomment to reach 100% coverage:
> 		// {100, "large"},
> 	}
> 
> 	for _, tt := range tests {
> 		got := Classify(tt.input)
> 		if got != tt.want {
> 			t.Errorf("Classify(%d) = %q, want %q", tt.input, got, tt.want)
> 		}
> 	}
> }
> ```
> 
> Run `go test -cover` to see ~75% coverage. Uncomment the last test case and re-run to see 100%.

---

## P13: Integration Test - REST API End-to-End

Build a small REST API with two endpoints:
- `POST /items` — accepts `{"name": "foo"}`, stores it, returns `201` with `{"id": 1, "name": "foo"}`
- `GET /items/{id}` — returns the item or `404`

Write an integration test using `httptest.NewServer` that:
1. POSTs an item
2. GETs it back by ID
3. GETs a non-existent ID and checks for 404

> [!hint]- Hint
> Use `httptest.NewServer(handler)` to start a real HTTP server. Use `http.Post` and `http.Get` with `ts.URL + "/items"`. Parse JSON responses to verify correctness.

> [!success]- Solution
> `api.go`:
> ```go
> package api
> 
> import (
> 	"encoding/json"
> 	"net/http"
> 	"strconv"
> 	"strings"
> 	"sync"
> )
> 
> type Item struct {
> 	ID   int    `json:"id"`
> 	Name string `json:"name"`
> }
> 
> type Store struct {
> 	mu    sync.Mutex
> 	items []Item
> }
> 
> func NewStore() *Store {
> 	return &Store{}
> }
> 
> func (s *Store) Add(name string) Item {
> 	s.mu.Lock()
> 	defer s.mu.Unlock()
> 	item := Item{ID: len(s.items) + 1, Name: name}
> 	s.items = append(s.items, item)
> 	return item
> }
> 
> func (s *Store) Get(id int) (Item, bool) {
> 	s.mu.Lock()
> 	defer s.mu.Unlock()
> 	for _, item := range s.items {
> 		if item.ID == id {
> 			return item, true
> 		}
> 	}
> 	return Item{}, false
> }
> 
> func NewHandler(store *Store) http.Handler {
> 	mux := http.NewServeMux()
> 
> 	mux.HandleFunc("POST /items", func(w http.ResponseWriter, r *http.Request) {
> 		var req struct {
> 			Name string `json:"name"`
> 		}
> 		if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
> 			http.Error(w, "bad request", http.StatusBadRequest)
> 			return
> 		}
> 		item := store.Add(req.Name)
> 		w.Header().Set("Content-Type", "application/json")
> 		w.WriteHeader(http.StatusCreated)
> 		json.NewEncoder(w).Encode(item)
> 	})
> 
> 	mux.HandleFunc("GET /items/{id}", func(w http.ResponseWriter, r *http.Request) {
> 		idStr := r.PathValue("id")
> 		id, err := strconv.Atoi(idStr)
> 		if err != nil {
> 			http.Error(w, "bad id", http.StatusBadRequest)
> 			return
> 		}
> 		item, ok := store.Get(id)
> 		if !ok {
> 			http.Error(w, "not found", http.StatusNotFound)
> 			return
> 		}
> 		w.Header().Set("Content-Type", "application/json")
> 		json.NewEncoder(w).Encode(item)
> 	})
> 
> 	return mux
> }
> 
> // For Go versions before 1.22 (no method+pattern routing), use this instead:
> func NewHandlerCompat(store *Store) http.Handler {
> 	mux := http.NewServeMux()
> 
> 	mux.HandleFunc("/items", func(w http.ResponseWriter, r *http.Request) {
> 		if r.Method == http.MethodPost {
> 			var req struct {
> 				Name string `json:"name"`
> 			}
> 			if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
> 				http.Error(w, "bad request", http.StatusBadRequest)
> 				return
> 			}
> 			item := store.Add(req.Name)
> 			w.Header().Set("Content-Type", "application/json")
> 			w.WriteHeader(http.StatusCreated)
> 			json.NewEncoder(w).Encode(item)
> 			return
> 		}
> 		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
> 	})
> 
> 	mux.HandleFunc("/items/", func(w http.ResponseWriter, r *http.Request) {
> 		if r.Method != http.MethodGet {
> 			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
> 			return
> 		}
> 		idStr := strings.TrimPrefix(r.URL.Path, "/items/")
> 		id, err := strconv.Atoi(idStr)
> 		if err != nil {
> 			http.Error(w, "bad id", http.StatusBadRequest)
> 			return
> 		}
> 		item, ok := store.Get(id)
> 		if !ok {
> 			http.Error(w, "not found", http.StatusNotFound)
> 			return
> 		}
> 		w.Header().Set("Content-Type", "application/json")
> 		json.NewEncoder(w).Encode(item)
> 	})
> 
> 	return mux
> }
> ```
> 
> `api_test.go`:
> ```go
> package api
> 
> import (
> 	"bytes"
> 	"encoding/json"
> 	"fmt"
> 	"net/http"
> 	"net/http/httptest"
> 	"testing"
> )
> 
> func TestAPIIntegration(t *testing.T) {
> 	store := NewStore()
> 	ts := httptest.NewServer(NewHandler(store))
> 	defer ts.Close()
> 
> 	// 1. POST /items
> 	body := bytes.NewBufferString(`{"name":"widget"}`)
> 	resp, err := http.Post(ts.URL+"/items", "application/json", body)
> 	if err != nil {
> 		t.Fatalf("POST failed: %v", err)
> 	}
> 	defer resp.Body.Close()
> 
> 	if resp.StatusCode != http.StatusCreated {
> 		t.Fatalf("POST status = %d, want 201", resp.StatusCode)
> 	}
> 
> 	var created Item
> 	json.NewDecoder(resp.Body).Decode(&created)
> 	if created.Name != "widget" || created.ID != 1 {
> 		t.Fatalf("unexpected item: %+v", created)
> 	}
> 
> 	// 2. GET /items/1
> 	resp2, err := http.Get(fmt.Sprintf("%s/items/%d", ts.URL, created.ID))
> 	if err != nil {
> 		t.Fatalf("GET failed: %v", err)
> 	}
> 	defer resp2.Body.Close()
> 
> 	if resp2.StatusCode != http.StatusOK {
> 		t.Fatalf("GET status = %d, want 200", resp2.StatusCode)
> 	}
> 
> 	var fetched Item
> 	json.NewDecoder(resp2.Body).Decode(&fetched)
> 	if fetched != created {
> 		t.Errorf("GET item = %+v, want %+v", fetched, created)
> 	}
> 
> 	// 3. GET /items/999 — not found
> 	resp3, err := http.Get(ts.URL + "/items/999")
> 	if err != nil {
> 		t.Fatalf("GET 999 failed: %v", err)
> 	}
> 	defer resp3.Body.Close()
> 
> 	if resp3.StatusCode != http.StatusNotFound {
> 		t.Errorf("GET 999 status = %d, want 404", resp3.StatusCode)
> 	}
> }
> ```
