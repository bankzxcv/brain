---
title: "Go Practice: Error Handling"
date: 2026-04-07
tags:
  - golang
  - practice
  - error-handling
parent: "[[Golang Study]]"
---

# Go Practice: Error Handling

13 progressive problems covering Go's error handling patterns - from basic checks to building a validation framework.

---

## P1: Basic Error Return and Check

Write a function `divide(a, b float64) (float64, error)` that returns an error when dividing by zero. The caller should check `if err != nil` and handle it.

**Expected output:**
```
10 / 3 = 3.33
10 / 0 = error: division by zero
```

> [!hint]- Hint
> Use `errors.New("division by zero")` or `fmt.Errorf("division by zero")` to create the error. Return the zero value for the result when returning an error.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"errors"
> 	"fmt"
> )
> 
> func divide(a, b float64) (float64, error) {
> 	if b == 0 {
> 		return 0, errors.New("division by zero")
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

## P2: Custom Error Type

Create a `ParseError` struct with fields `Line int`, `Col int`, and `Message string`. Implement the `error` interface. Write a function `parseLine` that returns a `*ParseError` on failure.

**Expected output:**
```
Parse error at line 10, col 25: unexpected token '}'
```

> [!hint]- Hint
> Implement `Error() string` on `*ParseError`. Format the fields into a descriptive string. Return `*ParseError` as an `error` from your function.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type ParseError struct {
> 	Line    int
> 	Col     int
> 	Message string
> }
> 
> func (e *ParseError) Error() string {
> 	return fmt.Sprintf("Parse error at line %d, col %d: %s",
> 		e.Line, e.Col, e.Message)
> }
> 
> func parseLine(input string) error {
> 	// Simulating a parse failure
> 	if input == "}" {
> 		return &ParseError{
> 			Line:    10,
> 			Col:     25,
> 			Message: "unexpected token '}'",
> 		}
> 	}
> 	return nil
> }
> 
> func main() {
> 	err := parseLine("}")
> 	if err != nil {
> 		fmt.Println(err)
> 	}
> }
> ```

---

## P3: Wrapping Errors with fmt.Errorf and %w

Write a three-layer call chain: `readConfig` calls `openFile` calls `checkPermission`. Each layer wraps the error from the layer below using `fmt.Errorf("context: %w", err)`. Print the final error to show the full chain.

**Expected output:**
```
readConfig: openFile "secret.yaml": checkPermission: access denied
```

> [!hint]- Hint
> The `%w` verb in `fmt.Errorf` wraps the error so that `errors.Is` and `errors.As` can traverse the chain. Each layer adds context about what it was trying to do.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"errors"
> 	"fmt"
> )
> 
> func checkPermission(path string) error {
> 	return errors.New("access denied")
> }
> 
> func openFile(path string) error {
> 	err := checkPermission(path)
> 	if err != nil {
> 		return fmt.Errorf("checkPermission: %w", err)
> 	}
> 	return nil
> }
> 
> func readConfig(path string) error {
> 	err := openFile(path)
> 	if err != nil {
> 		return fmt.Errorf("readConfig: openFile %q: %w", path, err)
> 	}
> 	return nil
> }
> 
> func main() {
> 	err := readConfig("secret.yaml")
> 	if err != nil {
> 		fmt.Println(err)
> 	}
> }
> ```

---

## P4: errors.Is - Checking the Error Chain

Using the wrapped error chain from P3, add a sentinel error `ErrAccessDenied`. Use `errors.Is` to check if the wrapped error chain contains `ErrAccessDenied`, even though it was wrapped multiple times.

**Expected output:**
```
Error: readConfig: openFile "secret.yaml": checkPermission: access denied
Is access denied? true
```

> [!hint]- Hint
> Define `var ErrAccessDenied = errors.New("access denied")` at the package level. Return this sentinel from `checkPermission`. `errors.Is(err, ErrAccessDenied)` unwraps the chain and compares each error.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"errors"
> 	"fmt"
> )
> 
> var ErrAccessDenied = errors.New("access denied")
> 
> func checkPermission(path string) error {
> 	return ErrAccessDenied
> }
> 
> func openFile(path string) error {
> 	err := checkPermission(path)
> 	if err != nil {
> 		return fmt.Errorf("checkPermission: %w", err)
> 	}
> 	return nil
> }
> 
> func readConfig(path string) error {
> 	err := openFile(path)
> 	if err != nil {
> 		return fmt.Errorf("readConfig: openFile %q: %w", path, err)
> 	}
> 	return nil
> }
> 
> func main() {
> 	err := readConfig("secret.yaml")
> 	if err != nil {
> 		fmt.Println("Error:", err)
> 		fmt.Println("Is access denied?", errors.Is(err, ErrAccessDenied))
> 	}
> }
> ```

---

## P5: errors.As - Extracting a Custom Error Type

Extend the `ParseError` from P2. Wrap it in another error. Use `errors.As` to extract the `*ParseError` from the chain and access its `Line` and `Col` fields.

**Expected output:**
```
Error: compileTemplate: parse "header.tmpl": Parse error at line 5, col 12: unclosed tag
Extracted ParseError -> Line: 5, Col: 12
```

> [!hint]- Hint
> `errors.As(err, &target)` unwraps the chain looking for an error that can be assigned to `target`. Declare `var pe *ParseError` and pass `&pe` to `errors.As`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"errors"
> 	"fmt"
> )
> 
> type ParseError struct {
> 	Line    int
> 	Col     int
> 	Message string
> }
> 
> func (e *ParseError) Error() string {
> 	return fmt.Sprintf("Parse error at line %d, col %d: %s",
> 		e.Line, e.Col, e.Message)
> }
> 
> func parseTemplate(name string) error {
> 	return &ParseError{Line: 5, Col: 12, Message: "unclosed tag"}
> }
> 
> func compileTemplate(name string) error {
> 	err := parseTemplate(name)
> 	if err != nil {
> 		return fmt.Errorf("compileTemplate: parse %q: %w", name, err)
> 	}
> 	return nil
> }
> 
> func main() {
> 	err := compileTemplate("header.tmpl")
> 	if err != nil {
> 		fmt.Println("Error:", err)
> 
> 		var pe *ParseError
> 		if errors.As(err, &pe) {
> 			fmt.Printf("Extracted ParseError -> Line: %d, Col: %d\n",
> 				pe.Line, pe.Col)
> 		}
> 	}
> }
> ```

---

## P6: Sentinel Errors

Define sentinel errors `ErrNotFound` and `ErrAlreadyExists`. Write a simple in-memory user store with `AddUser` and `GetUser` that return these sentinels. Handle each case differently in the caller.

**Expected output:**
```
Added user: alice
Error adding alice again: user already exists
Found user: alice (email: alice@example.com)
Error getting bob: user not found
```

> [!hint]- Hint
> Sentinel errors are package-level variables: `var ErrNotFound = errors.New("user not found")`. Use `errors.Is(err, ErrNotFound)` to check which error was returned. The convention is to prefix with `Err`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"errors"
> 	"fmt"
> )
> 
> var (
> 	ErrNotFound      = errors.New("user not found")
> 	ErrAlreadyExists = errors.New("user already exists")
> )
> 
> type User struct {
> 	Name  string
> 	Email string
> }
> 
> type UserStore struct {
> 	users map[string]User
> }
> 
> func NewUserStore() *UserStore {
> 	return &UserStore{users: make(map[string]User)}
> }
> 
> func (s *UserStore) AddUser(name, email string) error {
> 	if _, exists := s.users[name]; exists {
> 		return ErrAlreadyExists
> 	}
> 	s.users[name] = User{Name: name, Email: email}
> 	return nil
> }
> 
> func (s *UserStore) GetUser(name string) (User, error) {
> 	user, ok := s.users[name]
> 	if !ok {
> 		return User{}, ErrNotFound
> 	}
> 	return user, nil
> }
> 
> func main() {
> 	store := NewUserStore()
> 
> 	// Add user
> 	if err := store.AddUser("alice", "alice@example.com"); err != nil {
> 		fmt.Println("Error:", err)
> 	} else {
> 		fmt.Println("Added user: alice")
> 	}
> 
> 	// Try adding again
> 	if err := store.AddUser("alice", "alice2@example.com"); err != nil {
> 		if errors.Is(err, ErrAlreadyExists) {
> 			fmt.Printf("Error adding alice again: %v\n", err)
> 		}
> 	}
> 
> 	// Get existing user
> 	if user, err := store.GetUser("alice"); err != nil {
> 		fmt.Println("Error:", err)
> 	} else {
> 		fmt.Printf("Found user: %s (email: %s)\n", user.Name, user.Email)
> 	}
> 
> 	// Get missing user
> 	if _, err := store.GetUser("bob"); err != nil {
> 		if errors.Is(err, ErrNotFound) {
> 			fmt.Printf("Error getting bob: %v\n", err)
> 		}
> 	}
> }
> ```

---

## P7: Rich Error with Multiple Fields

Create a `ValidationError` struct with fields `Field`, `Value`, and `Tag` (e.g., "required", "min", "max"). Implement `Error()`. Write a `validateAge` function that returns a `ValidationError` for invalid input.

**Expected output:**
```
Validation failed:
  Field: "Age", Value: "-5", Tag: "min" -> Age must be >= 0
  Field: "Age", Value: "200", Tag: "max" -> Age must be <= 150
```

> [!hint]- Hint
> The `ValidationError` can include a computed `Message` or generate it from the fields in the `Error()` method. You can test multiple validation rules and collect errors.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type ValidationError struct {
> 	Field   string
> 	Value   string
> 	Tag     string
> 	Message string
> }
> 
> func (e *ValidationError) Error() string {
> 	return fmt.Sprintf("Field: %q, Value: %q, Tag: %q -> %s",
> 		e.Field, e.Value, e.Tag, e.Message)
> }
> 
> func validateAge(age int) []error {
> 	var errs []error
> 
> 	if age < 0 {
> 		errs = append(errs, &ValidationError{
> 			Field:   "Age",
> 			Value:   fmt.Sprintf("%d", age),
> 			Tag:     "min",
> 			Message: "Age must be >= 0",
> 		})
> 	}
> 
> 	if age > 150 {
> 		errs = append(errs, &ValidationError{
> 			Field:   "Age",
> 			Value:   fmt.Sprintf("%d", age),
> 			Tag:     "max",
> 			Message: "Age must be <= 150",
> 		})
> 	}
> 
> 	return errs
> }
> 
> func main() {
> 	testAges := []int{-5, 200}
> 
> 	fmt.Println("Validation failed:")
> 	for _, age := range testAges {
> 		errs := validateAge(age)
> 		for _, err := range errs {
> 			fmt.Printf("  %s\n", err)
> 		}
> 	}
> }
> ```

---

## P8: Multi-Error Collection from Batch Operations

Write a function `processBatch(items []string) error` that processes each item. Some items fail. Collect all errors and return them as a combined error. The caller should see all failures, not just the first.

**Input:** `["valid1", "", "valid2", "", "valid3"]`

**Expected output:**
```
Processing batch of 5 items...
Batch completed with errors:
  - item 1: empty string not allowed
  - item 3: empty string not allowed
Succeeded: 3, Failed: 2
```

> [!hint]- Hint
> Create a custom `BatchError` that holds a `[]error` slice. Its `Error()` method formats all collected errors. You could also use `strings.Builder` for efficient string building.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> type BatchError struct {
> 	Errors []error
> }
> 
> func (e *BatchError) Error() string {
> 	var b strings.Builder
> 	for _, err := range e.Errors {
> 		fmt.Fprintf(&b, "  - %s\n", err)
> 	}
> 	return b.String()
> }
> 
> func processItem(item string) error {
> 	if item == "" {
> 		return fmt.Errorf("empty string not allowed")
> 	}
> 	return nil
> }
> 
> func processBatch(items []string) (succeeded int, failed int, err error) {
> 	var batchErr BatchError
> 
> 	for i, item := range items {
> 		if e := processItem(item); e != nil {
> 			batchErr.Errors = append(batchErr.Errors,
> 				fmt.Errorf("item %d: %w", i, e))
> 			failed++
> 		} else {
> 			succeeded++
> 		}
> 	}
> 
> 	if len(batchErr.Errors) > 0 {
> 		return succeeded, failed, &batchErr
> 	}
> 	return succeeded, failed, nil
> }
> 
> func main() {
> 	items := []string{"valid1", "", "valid2", "", "valid3"}
> 	fmt.Printf("Processing batch of %d items...\n", len(items))
> 
> 	succeeded, failed, err := processBatch(items)
> 	if err != nil {
> 		fmt.Println("Batch completed with errors:")
> 		fmt.Print(err)
> 	}
> 	fmt.Printf("Succeeded: %d, Failed: %d\n", succeeded, failed)
> }
> ```

---

## P9: errors.Join (Go 1.20+)

Use `errors.Join` to combine multiple errors into one. Then use `errors.Is` to check if individual sentinel errors are present in the joined error.

**Expected output:**
```
Combined error:
connection timeout
authentication failed
rate limit exceeded

Contains timeout? true
Contains auth failure? true
Contains not found? false
```

> [!hint]- Hint
> `errors.Join(err1, err2, err3)` combines errors. The result supports `errors.Is` for each of the joined errors. Nil errors in the arguments are skipped.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"errors"
> 	"fmt"
> )
> 
> var (
> 	ErrTimeout  = errors.New("connection timeout")
> 	ErrAuth     = errors.New("authentication failed")
> 	ErrRateLimit = errors.New("rate limit exceeded")
> 	ErrNotFound = errors.New("not found")
> )
> 
> func main() {
> 	// Simulate collecting errors from multiple subsystems
> 	combined := errors.Join(ErrTimeout, ErrAuth, ErrRateLimit)
> 
> 	fmt.Println("Combined error:")
> 	fmt.Println(combined)
> 
> 	fmt.Println()
> 	fmt.Println("Contains timeout?", errors.Is(combined, ErrTimeout))
> 	fmt.Println("Contains auth failure?", errors.Is(combined, ErrAuth))
> 	fmt.Println("Contains not found?", errors.Is(combined, ErrNotFound))
> }
> ```

---

## P10: Must Pattern

Write a `must[T any](val T, err error) T` generic function that panics if `err` is not nil, otherwise returns `val`. Use it to parse URLs at program startup where failure should be fatal.

**Expected output:**
```
Base URL: https://api.example.com
Parsed host: api.example.com
Caught panic for invalid URL: parse "://bad url": missing protocol scheme
```

> [!hint]- Hint
> The `must` pattern is common for initialization code where an error is truly unrecoverable. `net/url.Parse` returns `(*url.URL, error)`. Wrap with `must` for one-liner usage. Use `recover()` to demonstrate what happens with a bad URL.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"net/url"
> )
> 
> func must[T any](val T, err error) T {
> 	if err != nil {
> 		panic(err)
> 	}
> 	return val
> }
> 
> func main() {
> 	// Successful parse
> 	baseURL := must(url.Parse("https://api.example.com"))
> 	fmt.Println("Base URL:", baseURL)
> 	fmt.Println("Parsed host:", baseURL.Host)
> 
> 	// Demonstrate panic with bad URL (recovered)
> 	func() {
> 		defer func() {
> 			if r := recover(); r != nil {
> 				fmt.Printf("Caught panic for invalid URL: %v\n", r)
> 			}
> 		}()
> 		_ = must(url.Parse("://bad url"))
> 	}()
> }
> ```

---

## P11: Generic Result Type (Rust-style)

Implement a `Result[T any]` type that holds either a value or an error. Provide methods:
- `Ok(val T) Result[T]`
- `Err[T](err error) Result[T]`
- `IsOk() bool`
- `Unwrap() T` (panics if error)
- `UnwrapOr(defaultVal T) T`
- `Map(fn func(T) T) Result[T]`

**Expected output:**
```
Result 1: Ok(42)
Result 2: Err(something went wrong)
r1.Unwrap(): 42
r2.UnwrapOr(0): 0
r1.Map(double): Ok(84)
```

> [!hint]- Hint
> Store both `val T` and `err error` in the struct. Use a boolean or check `err == nil` to determine the state. `Unwrap` should `panic` when the result holds an error.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Result[T any] struct {
> 	val T
> 	err error
> }
> 
> func Ok[T any](val T) Result[T] {
> 	return Result[T]{val: val}
> }
> 
> func Err[T any](err error) Result[T] {
> 	return Result[T]{err: err}
> }
> 
> func (r Result[T]) IsOk() bool {
> 	return r.err == nil
> }
> 
> func (r Result[T]) IsErr() bool {
> 	return r.err != nil
> }
> 
> func (r Result[T]) Unwrap() T {
> 	if r.err != nil {
> 		panic(fmt.Sprintf("called Unwrap on Err: %v", r.err))
> 	}
> 	return r.val
> }
> 
> func (r Result[T]) UnwrapOr(defaultVal T) T {
> 	if r.err != nil {
> 		return defaultVal
> 	}
> 	return r.val
> }
> 
> func (r Result[T]) Map(fn func(T) T) Result[T] {
> 	if r.err != nil {
> 		return r
> 	}
> 	return Ok(fn(r.val))
> }
> 
> func (r Result[T]) String() string {
> 	if r.err != nil {
> 		return fmt.Sprintf("Err(%v)", r.err)
> 	}
> 	return fmt.Sprintf("Ok(%v)", r.val)
> }
> 
> func main() {
> 	r1 := Ok(42)
> 	r2 := Err[int](fmt.Errorf("something went wrong"))
> 
> 	fmt.Printf("Result 1: %s\n", r1)
> 	fmt.Printf("Result 2: %s\n", r2)
> 
> 	fmt.Printf("r1.Unwrap(): %d\n", r1.Unwrap())
> 	fmt.Printf("r2.UnwrapOr(0): %d\n", r2.UnwrapOr(0))
> 
> 	double := func(n int) int { return n * 2 }
> 	fmt.Printf("r1.Map(double): %s\n", r1.Map(double))
> }
> ```

---

## P12: Error Handling in HTTP Handlers

Write an `AppError` type with `StatusCode`, `Message`, and an underlying `Err`. Write a handler-like function that returns `*AppError`. Map the errors to appropriate HTTP status codes.

**Expected output:**
```
GET /users/123 -> 200: {"name": "alice"}
GET /users/999 -> 404: user not found
POST /users    -> 400: validation failed: name is required
DELETE /admin  -> 403: forbidden: admin access required
```

> [!hint]- Hint
> Create helper constructors: `NotFound(msg)`, `BadRequest(msg)`, `Forbidden(msg)` that return `*AppError` with the right status code. This pattern maps cleanly to real HTTP handler code.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type AppError struct {
> 	StatusCode int
> 	Message    string
> 	Err        error
> }
> 
> func (e *AppError) Error() string {
> 	if e.Err != nil {
> 		return fmt.Sprintf("%s: %v", e.Message, e.Err)
> 	}
> 	return e.Message
> }
> 
> func (e *AppError) Unwrap() error {
> 	return e.Err
> }
> 
> func NotFound(msg string) *AppError {
> 	return &AppError{StatusCode: 404, Message: msg}
> }
> 
> func BadRequest(msg string, err error) *AppError {
> 	return &AppError{StatusCode: 400, Message: msg, Err: err}
> }
> 
> func Forbidden(msg string) *AppError {
> 	return &AppError{StatusCode: 403, Message: msg}
> }
> 
> // Simulate HTTP handlers
> func getUser(id string) (string, *AppError) {
> 	if id == "123" {
> 		return `{"name": "alice"}`, nil
> 	}
> 	return "", NotFound("user not found")
> }
> 
> func createUser(name string) *AppError {
> 	if name == "" {
> 		return BadRequest("validation failed",
> 			fmt.Errorf("name is required"))
> 	}
> 	return nil
> }
> 
> func deleteAdmin(role string) *AppError {
> 	if role != "admin" {
> 		return Forbidden("forbidden: admin access required")
> 	}
> 	return nil
> }
> 
> func handleResult(method, path string, body string, appErr *AppError) {
> 	if appErr != nil {
> 		fmt.Printf("%-14s -> %d: %s\n", method+" "+path,
> 			appErr.StatusCode, appErr.Error())
> 	} else {
> 		fmt.Printf("%-14s -> 200: %s\n", method+" "+path, body)
> 	}
> }
> 
> func main() {
> 	body, err := getUser("123")
> 	handleResult("GET", "/users/123", body, err)
> 
> 	body, err = getUser("999")
> 	handleResult("GET", "/users/999", body, err)
> 
> 	err2 := createUser("")
> 	handleResult("POST", "/users", "", err2)
> 
> 	err3 := deleteAdmin("user")
> 	handleResult("DELETE", "/admin", "", err3)
> }
> ```

---

## P13: Validation Framework

Build a mini validation framework. Define a `Validator` function type. Implement validators: `Required`, `MinLen`, `MaxLen`, `InRange`. Write a `ValidateStruct` function that runs all validators on a struct and collects all errors.

**Expected output:**
```
Validating User{Name: "", Email: "a", Age: 200}:
  - Name: required field is empty
  - Email: length 1 is less than minimum 5
  - Age: value 200 is out of range [0, 150]

Validating User{Name: "Alice", Email: "alice@go.dev", Age: 30}:
  All validations passed!
```

> [!hint]- Hint
> Define `type Validator func(fieldName string, value any) error`. Each validator is a function that returns a `Validator`. For example, `Required()` returns a `Validator` that checks if a string is empty. Use a map of field name to validators, and run all of them.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> type FieldError struct {
> 	Field   string
> 	Message string
> }
> 
> func (e FieldError) Error() string {
> 	return fmt.Sprintf("%s: %s", e.Field, e.Message)
> }
> 
> type Validator func(fieldName string, value any) *FieldError
> 
> func Required() Validator {
> 	return func(field string, value any) *FieldError {
> 		if s, ok := value.(string); ok && strings.TrimSpace(s) == "" {
> 			return &FieldError{Field: field, Message: "required field is empty"}
> 		}
> 		return nil
> 	}
> }
> 
> func MinLen(min int) Validator {
> 	return func(field string, value any) *FieldError {
> 		if s, ok := value.(string); ok {
> 			if len(s) < min {
> 				return &FieldError{
> 					Field:   field,
> 					Message: fmt.Sprintf("length %d is less than minimum %d", len(s), min),
> 				}
> 			}
> 		}
> 		return nil
> 	}
> }
> 
> func MaxLen(max int) Validator {
> 	return func(field string, value any) *FieldError {
> 		if s, ok := value.(string); ok {
> 			if len(s) > max {
> 				return &FieldError{
> 					Field:   field,
> 					Message: fmt.Sprintf("length %d exceeds maximum %d", len(s), max),
> 				}
> 			}
> 		}
> 		return nil
> 	}
> }
> 
> func InRange(min, max int) Validator {
> 	return func(field string, value any) *FieldError {
> 		if n, ok := value.(int); ok {
> 			if n < min || n > max {
> 				return &FieldError{
> 					Field:   field,
> 					Message: fmt.Sprintf("value %d is out of range [%d, %d]", n, min, max),
> 				}
> 			}
> 		}
> 		return nil
> 	}
> }
> 
> type FieldRule struct {
> 	Name       string
> 	Value      any
> 	Validators []Validator
> }
> 
> func Validate(rules []FieldRule) []error {
> 	var errs []error
> 	for _, rule := range rules {
> 		for _, v := range rule.Validators {
> 			if err := v(rule.Name, rule.Value); err != nil {
> 				errs = append(errs, err)
> 				break // one error per field is enough
> 			}
> 		}
> 	}
> 	return errs
> }
> 
> type User struct {
> 	Name  string
> 	Email string
> 	Age   int
> }
> 
> func validateUser(u User) []error {
> 	return Validate([]FieldRule{
> 		{"Name", u.Name, []Validator{Required(), MinLen(2), MaxLen(50)}},
> 		{"Email", u.Email, []Validator{Required(), MinLen(5), MaxLen(100)}},
> 		{"Age", u.Age, []Validator{InRange(0, 150)}},
> 	})
> }
> 
> func main() {
> 	// Invalid user
> 	u1 := User{Name: "", Email: "a", Age: 200}
> 	fmt.Printf("Validating User{Name: %q, Email: %q, Age: %d}:\n",
> 		u1.Name, u1.Email, u1.Age)
> 	errs := validateUser(u1)
> 	for _, err := range errs {
> 		fmt.Printf("  - %s\n", err)
> 	}
> 
> 	// Valid user
> 	fmt.Println()
> 	u2 := User{Name: "Alice", Email: "alice@go.dev", Age: 30}
> 	fmt.Printf("Validating User{Name: %q, Email: %q, Age: %d}:\n",
> 		u2.Name, u2.Email, u2.Age)
> 	errs = validateUser(u2)
> 	if len(errs) == 0 {
> 		fmt.Println("  All validations passed!")
> 	}
> }
> ```
