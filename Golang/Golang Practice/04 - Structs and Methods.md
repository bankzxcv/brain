---
title: "Go Practice: Structs and Methods"
date: 2026-04-07
tags:
  - golang
  - practice
  - structs
  - methods
parent: "[[Golang Study]]"
---

# Go Practice: Structs and Methods

14 progressive problems. Write each solution from scratch to build muscle memory.

---

## P1: Define a Struct and Create Instances

Define a `Book` struct with `Title string`, `Author string`, and `Pages int`. Create one instance with named fields and one with positional initialization. Print both.

**Expected output:**
```
{Title:The Go Programming Language Author:Donovan & Kernighan Pages:380}
{Title:Concurrency in Go Author:Katherine Cox-Buday Pages:238}
```

> [!hint]- Hint
> Named: `Book{Title: "...", Author: "...", Pages: 380}`. Positional: `Book{"...", "...", 238}`. Positional init requires all fields in order and is fragile -- prefer named fields.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Book struct {
> 	Title  string
> 	Author string
> 	Pages  int
> }
> 
> func main() {
> 	b1 := Book{
> 		Title:  "The Go Programming Language",
> 		Author: "Donovan & Kernighan",
> 		Pages:  380,
> 	}
> 
> 	b2 := Book{"Concurrency in Go", "Katherine Cox-Buday", 238}
> 
> 	fmt.Printf("%+v\n", b1)
> 	fmt.Printf("%+v\n", b2)
> }
> ```

---

## P2: Value Receiver - String() Method

Add a `String() string` method to the `Book` struct from P1 (value receiver). This makes `Book` satisfy `fmt.Stringer`. Print a book using `fmt.Println`.

**Expected output:**
```
"The Go Programming Language" by Donovan & Kernighan (380 pages)
```

> [!hint]- Hint
> Implement `func (b Book) String() string` and use `fmt.Sprintf` to format the output. When you pass a `Book` to `fmt.Println`, it automatically calls `String()`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Book struct {
> 	Title  string
> 	Author string
> 	Pages  int
> }
> 
> func (b Book) String() string {
> 	return fmt.Sprintf("%q by %s (%d pages)", b.Title, b.Author, b.Pages)
> }
> 
> func main() {
> 	b := Book{
> 		Title:  "The Go Programming Language",
> 		Author: "Donovan & Kernighan",
> 		Pages:  380,
> 	}
> 	fmt.Println(b)
> }
> ```

---

## P3: Pointer Receiver - Update Price

Define a `Product` struct with `Name string` and `Price float64`. Add a method `ApplyDiscount(percent float64)` with a pointer receiver that reduces the price. Show that the original product is modified.

**Expected output:**
```
Before: Laptop $999.99
After 10% discount: Laptop $899.99
After 50% discount: Laptop $450.00
```

> [!hint]- Hint
> Use a pointer receiver `func (p *Product) ApplyDiscount(percent float64)` so the method modifies the original struct. The formula is `p.Price -= p.Price * percent / 100`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Product struct {
> 	Name  string
> 	Price float64
> }
> 
> func (p *Product) ApplyDiscount(percent float64) {
> 	p.Price -= p.Price * percent / 100
> }
> 
> func (p Product) String() string {
> 	return fmt.Sprintf("%s $%.2f", p.Name, p.Price)
> }
> 
> func main() {
> 	p := Product{Name: "Laptop", Price: 999.99}
> 	fmt.Printf("Before: %s\n", p)
> 
> 	p.ApplyDiscount(10)
> 	fmt.Printf("After 10%% discount: %s\n", p)
> 
> 	p.ApplyDiscount(50)
> 	fmt.Printf("After 50%% discount: %s\n", p)
> }
> ```

---

## P4: Constructor Pattern - NewUser with Validation

Define a `User` struct with `Name string`, `Email string`, and `Age int`. Write a `NewUser(name, email string, age int) (*User, error)` constructor that validates:
- Name must not be empty
- Email must contain `@`
- Age must be >= 0

**Expected output:**
```
User: &{Name:Alice Email:alice@example.com Age:30}
Error: name cannot be empty
Error: email must contain '@'
Error: age cannot be negative
```

> [!hint]- Hint
> Go has no real constructors -- the convention is a `NewXxx` function that returns a pointer and an error. Use `strings.Contains(email, "@")` for a simple email check.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> type User struct {
> 	Name  string
> 	Email string
> 	Age   int
> }
> 
> func NewUser(name, email string, age int) (*User, error) {
> 	if name == "" {
> 		return nil, fmt.Errorf("name cannot be empty")
> 	}
> 	if !strings.Contains(email, "@") {
> 		return nil, fmt.Errorf("email must contain '@'")
> 	}
> 	if age < 0 {
> 		return nil, fmt.Errorf("age cannot be negative")
> 	}
> 	return &User{Name: name, Email: email, Age: age}, nil
> }
> 
> func main() {
> 	testCases := []struct {
> 		name, email string
> 		age         int
> 	}{
> 		{"Alice", "alice@example.com", 30},
> 		{"", "bob@example.com", 25},
> 		{"Charlie", "charlie.example.com", 20},
> 		{"Dave", "dave@example.com", -5},
> 	}
> 
> 	for _, tc := range testCases {
> 		user, err := NewUser(tc.name, tc.email, tc.age)
> 		if err != nil {
> 			fmt.Printf("Error: %v\n", err)
> 		} else {
> 			fmt.Printf("User: %+v\n", user)
> 		}
> 	}
> }
> ```

---

## P5: Embedding - Employee Embeds Person

Define `Person` with `Name` and `Age`, and `Employee` that embeds `Person` and adds `Company` and `Role`. Show that you can access `Name` and `Age` directly on `Employee` (promoted fields).

**Expected output:**
```
Name: Alice (directly promoted)
Age: 30 (directly promoted)
Company: Gopher Inc
Full: {Person:{Name:Alice Age:30} Company:Gopher Inc Role:Engineer}
```

> [!hint]- Hint
> Embedding syntax: just put the type name without a field name. `type Employee struct { Person; Company string; Role string }`. The embedded fields are "promoted" and accessible directly.

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
> type Employee struct {
> 	Person
> 	Company string
> 	Role    string
> }
> 
> func main() {
> 	e := Employee{
> 		Person:  Person{Name: "Alice", Age: 30},
> 		Company: "Gopher Inc",
> 		Role:    "Engineer",
> 	}
> 
> 	// Promoted fields -- accessed directly
> 	fmt.Printf("Name: %s (directly promoted)\n", e.Name)
> 	fmt.Printf("Age: %d (directly promoted)\n", e.Age)
> 	fmt.Printf("Company: %s\n", e.Company)
> 	fmt.Printf("Full: %+v\n", e)
> }
> ```

---

## P6: Method Override with Embedding

Define `Animal` with a `Speak() string` method that returns `"..."`. Define `Dog` that embeds `Animal` and overrides `Speak()` to return `"Woof!"`. Show both the overridden and the original method.

**Expected output:**
```
Dog says: Woof!
Dog's inner Animal says: ...
```

> [!hint]- Hint
> When a method is defined on both the outer and embedded struct with the same name, the outer struct's method "shadows" the embedded one. You can still call the embedded one via `d.Animal.Speak()`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Animal struct{}
> 
> func (a Animal) Speak() string {
> 	return "..."
> }
> 
> type Dog struct {
> 	Animal
> 	Name string
> }
> 
> func (d Dog) Speak() string {
> 	return "Woof!"
> }
> 
> func main() {
> 	d := Dog{Name: "Rex"}
> 
> 	fmt.Printf("Dog says: %s\n", d.Speak())
> 	fmt.Printf("Dog's inner Animal says: %s\n", d.Animal.Speak())
> }
> ```

---

## P7: Struct Tags - JSON Serialization

Define a `Config` struct with fields:
- `Host` (json: "host")
- `Port` (json: "port")
- `Debug` (json: "debug", omitempty)
- `Secret` (json: "-", meaning excluded from JSON)

Marshal to JSON, then unmarshal back. Print both.

**Expected output:**
```
JSON: {"host":"localhost","port":8080}
Parsed: {Host:localhost Port:8080 Debug:false Secret:}
```

> [!hint]- Hint
> Struct tags go after the type: `Host string \`json:"host"\``. The tag `json:"-"` excludes the field. The `omitempty` option skips zero-value fields. Use `json.Marshal` and `json.Unmarshal`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"encoding/json"
> 	"fmt"
> )
> 
> type Config struct {
> 	Host   string `json:"host"`
> 	Port   int    `json:"port"`
> 	Debug  bool   `json:"debug,omitempty"`
> 	Secret string `json:"-"`
> }
> 
> func main() {
> 	cfg := Config{
> 		Host:   "localhost",
> 		Port:   8080,
> 		Debug:  false,
> 		Secret: "super-secret-key",
> 	}
> 
> 	// Marshal to JSON
> 	data, err := json.Marshal(cfg)
> 	if err != nil {
> 		panic(err)
> 	}
> 	fmt.Printf("JSON: %s\n", data)
> 
> 	// Unmarshal back
> 	var parsed Config
> 	err = json.Unmarshal(data, &parsed)
> 	if err != nil {
> 		panic(err)
> 	}
> 	fmt.Printf("Parsed: %+v\n", parsed)
> }
> ```

---

## P8: Anonymous Structs for Test Cases

Write a test-like function that uses a slice of anonymous structs to define test cases for an `add(a, b int) int` function. Each test case has `name string`, `a int`, `b int`, and `expected int`.

**Expected output:**
```
PASS: positive numbers
PASS: negative numbers
PASS: zeros
PASS: mixed signs
```

> [!hint]- Hint
> Define the slice inline: `tests := []struct { name string; a, b, expected int }{ {...}, {...} }`. No need to declare a named struct type for one-off usage.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func add(a, b int) int {
> 	return a + b
> }
> 
> func main() {
> 	tests := []struct {
> 		name     string
> 		a, b     int
> 		expected int
> 	}{
> 		{"positive numbers", 2, 3, 5},
> 		{"negative numbers", -1, -2, -3},
> 		{"zeros", 0, 0, 0},
> 		{"mixed signs", -5, 10, 5},
> 	}
> 
> 	for _, tt := range tests {
> 		result := add(tt.a, tt.b)
> 		if result == tt.expected {
> 			fmt.Printf("PASS: %s\n", tt.name)
> 		} else {
> 			fmt.Printf("FAIL: %s (got %d, want %d)\n", tt.name, result, tt.expected)
> 		}
> 	}
> }
> ```

---

## P9: Implement Comparable Structs

Define a `Point` struct with `X, Y float64`. Implement `Equals(other Point) bool` and `Distance(other Point) float64`. Test equality and distance between points.

**Expected output:**
```
p1 == p2: true
p1 == p3: false
Distance p1 to p3: 5.00
```

> [!hint]- Hint
> For floating point equality, use a small epsilon: `math.Abs(p.X - other.X) < 1e-9`. For distance, use `math.Sqrt((dx*dx) + (dy*dy))` or `math.Hypot(dx, dy)`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"math"
> )
> 
> type Point struct {
> 	X, Y float64
> }
> 
> func (p Point) Equals(other Point) bool {
> 	const epsilon = 1e-9
> 	return math.Abs(p.X-other.X) < epsilon &&
> 		math.Abs(p.Y-other.Y) < epsilon
> }
> 
> func (p Point) Distance(other Point) float64 {
> 	dx := p.X - other.X
> 	dy := p.Y - other.Y
> 	return math.Hypot(dx, dy)
> }
> 
> func main() {
> 	p1 := Point{3, 4}
> 	p2 := Point{3, 4}
> 	p3 := Point{6, 8}
> 
> 	fmt.Printf("p1 == p2: %t\n", p1.Equals(p2))
> 	fmt.Printf("p1 == p3: %t\n", p1.Equals(p3))
> 	fmt.Printf("Distance p1 to p3: %.2f\n", p1.Distance(p3))
> }
> ```

---

## P10: Builder Pattern

Implement a builder for an `HTTPRequest` struct with fields: `Method`, `URL`, `Headers map[string]string`, `Body string`, `Timeout time.Duration`. Each builder method returns `*RequestBuilder` for chaining. The final `Build()` returns the `HTTPRequest`.

**Expected output:**
```
Request:
  Method: POST
  URL: https://api.example.com/users
  Headers: map[Authorization:Bearer token123 Content-Type:application/json]
  Body: {"name":"alice"}
  Timeout: 30s
```

> [!hint]- Hint
> The builder struct holds the in-progress request. Each method like `WithMethod(m string)` sets a field and returns the builder pointer. `Build()` returns the final struct.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"time"
> )
> 
> type HTTPRequest struct {
> 	Method  string
> 	URL     string
> 	Headers map[string]string
> 	Body    string
> 	Timeout time.Duration
> }
> 
> type RequestBuilder struct {
> 	request HTTPRequest
> }
> 
> func NewRequestBuilder() *RequestBuilder {
> 	return &RequestBuilder{
> 		request: HTTPRequest{
> 			Method:  "GET",
> 			Headers: make(map[string]string),
> 			Timeout: 10 * time.Second,
> 		},
> 	}
> }
> 
> func (b *RequestBuilder) Method(m string) *RequestBuilder {
> 	b.request.Method = m
> 	return b
> }
> 
> func (b *RequestBuilder) URL(url string) *RequestBuilder {
> 	b.request.URL = url
> 	return b
> }
> 
> func (b *RequestBuilder) Header(key, value string) *RequestBuilder {
> 	b.request.Headers[key] = value
> 	return b
> }
> 
> func (b *RequestBuilder) Body(body string) *RequestBuilder {
> 	b.request.Body = body
> 	return b
> }
> 
> func (b *RequestBuilder) Timeout(d time.Duration) *RequestBuilder {
> 	b.request.Timeout = d
> 	return b
> }
> 
> func (b *RequestBuilder) Build() HTTPRequest {
> 	return b.request
> }
> 
> func main() {
> 	req := NewRequestBuilder().
> 		Method("POST").
> 		URL("https://api.example.com/users").
> 		Header("Content-Type", "application/json").
> 		Header("Authorization", "Bearer token123").
> 		Body(`{"name":"alice"}`).
> 		Timeout(30 * time.Second).
> 		Build()
> 
> 	fmt.Println("Request:")
> 	fmt.Printf("  Method: %s\n", req.Method)
> 	fmt.Printf("  URL: %s\n", req.URL)
> 	fmt.Printf("  Headers: %v\n", req.Headers)
> 	fmt.Printf("  Body: %s\n", req.Body)
> 	fmt.Printf("  Timeout: %s\n", req.Timeout)
> }
> ```

---

## P11: Linked List with Methods

Implement a linked list with these methods:
- `Push(val int)` - add to front
- `Pop() (int, bool)` - remove from front
- `Len() int`
- `String() string` - pretty print

**Expected output:**
```
Empty list: [] (len=0)
After Push 1,2,3: [3 -> 2 -> 1] (len=3)
Pop: 3, ok=true
Pop: 2, ok=true
After pops: [1] (len=1)
Pop: 1, ok=true
Pop from empty: ok=false
```

> [!hint]- Hint
> `Push` creates a new node pointing to the current head, then updates head. `Pop` returns the head's value and advances head to the next node. Track length with a counter.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> type node struct {
> 	value int
> 	next  *node
> }
> 
> type LinkedList struct {
> 	head *node
> 	len  int
> }
> 
> func (ll *LinkedList) Push(val int) {
> 	ll.head = &node{value: val, next: ll.head}
> 	ll.len++
> }
> 
> func (ll *LinkedList) Pop() (int, bool) {
> 	if ll.head == nil {
> 		return 0, false
> 	}
> 	val := ll.head.value
> 	ll.head = ll.head.next
> 	ll.len--
> 	return val, true
> }
> 
> func (ll *LinkedList) Len() int {
> 	return ll.len
> }
> 
> func (ll *LinkedList) String() string {
> 	if ll.head == nil {
> 		return "[]"
> 	}
> 	var parts []string
> 	current := ll.head
> 	for current != nil {
> 		parts = append(parts, fmt.Sprintf("%d", current.value))
> 		current = current.next
> 	}
> 	return "[" + strings.Join(parts, " -> ") + "]"
> }
> 
> func main() {
> 	ll := &LinkedList{}
> 	fmt.Printf("Empty list: %s (len=%d)\n", ll, ll.Len())
> 
> 	ll.Push(1)
> 	ll.Push(2)
> 	ll.Push(3)
> 	fmt.Printf("After Push 1,2,3: %s (len=%d)\n", ll, ll.Len())
> 
> 	val, ok := ll.Pop()
> 	fmt.Printf("Pop: %d, ok=%t\n", val, ok)
> 	val, ok = ll.Pop()
> 	fmt.Printf("Pop: %d, ok=%t\n", val, ok)
> 	fmt.Printf("After pops: %s (len=%d)\n", ll, ll.Len())
> 
> 	val, ok = ll.Pop()
> 	fmt.Printf("Pop: %d, ok=%t\n", val, ok)
> 	_, ok = ll.Pop()
> 	fmt.Printf("Pop from empty: ok=%t\n", ok)
> }
> ```

---

## P12: Composition with Multiple Embeds

Define three structs: `Person` (Name, Age), `Address` (Street, City, Country), `Employment` (Company, Role, Salary). Create `FullEmployee` that embeds all three. Demonstrate accessing promoted fields and handling field name conflicts.

**Expected output:**
```
Name: Alice, Age: 30
Street: 123 Go St, City: Gophertown
Company: Gopher Inc, Role: Engineer
Salary: $120000.00
Full: {Person:{Name:Alice Age:30} Address:{Street:123 Go St City:Gophertown Country:US} Employment:{Company:Gopher Inc Role:Engineer Salary:120000}}
```

> [!hint]- Hint
> When embedding multiple structs, all their fields and methods are promoted. If two embedded structs have a field with the same name, you must qualify it (e.g., `fe.Person.Name` vs `fe.Address.Name`). As long as names are unique, they promote cleanly.

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
> type Address struct {
> 	Street  string
> 	City    string
> 	Country string
> }
> 
> type Employment struct {
> 	Company string
> 	Role    string
> 	Salary  float64
> }
> 
> type FullEmployee struct {
> 	Person
> 	Address
> 	Employment
> }
> 
> func main() {
> 	fe := FullEmployee{
> 		Person:     Person{Name: "Alice", Age: 30},
> 		Address:    Address{Street: "123 Go St", City: "Gophertown", Country: "US"},
> 		Employment: Employment{Company: "Gopher Inc", Role: "Engineer", Salary: 120000},
> 	}
> 
> 	// All fields are promoted since names are unique
> 	fmt.Printf("Name: %s, Age: %d\n", fe.Name, fe.Age)
> 	fmt.Printf("Street: %s, City: %s\n", fe.Street, fe.City)
> 	fmt.Printf("Company: %s, Role: %s\n", fe.Company, fe.Role)
> 	fmt.Printf("Salary: $%.2f\n", fe.Salary)
> 	fmt.Printf("Full: %+v\n", fe)
> }
> ```

---

## P13: Immutable Struct Pattern

Define a `Settings` struct with `Theme string`, `FontSize int`, and `Language string`. Make it "immutable" by:
- Using unexported fields
- Providing getter methods
- Providing `WithXxx` methods that return a *new* `Settings` (not modifying the original)

**Expected output:**
```
Original: {theme:dark fontSize:14 language:en}
Modified: {theme:dark fontSize:18 language:en}
Original unchanged: {theme:dark fontSize:14 language:en}
```

> [!hint]- Hint
> Unexported fields (lowercase) prevent direct modification from outside the package. `With` methods copy the struct, change one field, and return the copy. This is like a functional update pattern.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Settings struct {
> 	theme    string
> 	fontSize int
> 	language string
> }
> 
> func NewSettings(theme string, fontSize int, language string) Settings {
> 	return Settings{theme: theme, fontSize: fontSize, language: language}
> }
> 
> func (s Settings) Theme() string    { return s.theme }
> func (s Settings) FontSize() int    { return s.fontSize }
> func (s Settings) Language() string  { return s.language }
> 
> func (s Settings) WithTheme(theme string) Settings {
> 	s.theme = theme
> 	return s
> }
> 
> func (s Settings) WithFontSize(size int) Settings {
> 	s.fontSize = size
> 	return s
> }
> 
> func (s Settings) WithLanguage(lang string) Settings {
> 	s.language = lang
> 	return s
> }
> 
> func (s Settings) String() string {
> 	return fmt.Sprintf("{theme:%s fontSize:%d language:%s}", s.theme, s.fontSize, s.language)
> }
> 
> func main() {
> 	original := NewSettings("dark", 14, "en")
> 	fmt.Printf("Original: %s\n", original)
> 
> 	modified := original.WithFontSize(18)
> 	fmt.Printf("Modified: %s\n", modified)
> 	fmt.Printf("Original unchanged: %s\n", original)
> }
> ```

---

## P14: Simple ORM-like Struct with Reflect

Build a simple function `ToMap(v any) map[string]any` that reads a struct's fields using `reflect` and returns a map keyed by the `db` struct tag (or field name if no tag). Also write `Columns(v any) []string` that returns the column names.

Test with a `User` struct:
```go
type User struct {
    ID    int    `db:"id"`
    Name  string `db:"name"`
    Email string `db:"email"`
    Age   int    // no db tag -- use field name
}
```

**Expected output:**
```
Columns: [id name email Age]
Map: map[Age:30 email:alice@example.com id:1 name:Alice]
```

> [!hint]- Hint
> Use `reflect.TypeOf(v)` to get the type and iterate fields with `.NumField()` and `.Field(i)`. Use `.Tag.Get("db")` to read the struct tag. Use `reflect.ValueOf(v).Field(i).Interface()` to get the value.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"reflect"
> )
> 
> type User struct {
> 	ID    int    `db:"id"`
> 	Name  string `db:"name"`
> 	Email string `db:"email"`
> 	Age   int    // no db tag
> }
> 
> func columnName(f reflect.StructField) string {
> 	tag := f.Tag.Get("db")
> 	if tag != "" {
> 		return tag
> 	}
> 	return f.Name
> }
> 
> func Columns(v any) []string {
> 	t := reflect.TypeOf(v)
> 	if t.Kind() == reflect.Ptr {
> 		t = t.Elem()
> 	}
> 
> 	cols := make([]string, t.NumField())
> 	for i := 0; i < t.NumField(); i++ {
> 		cols[i] = columnName(t.Field(i))
> 	}
> 	return cols
> }
> 
> func ToMap(v any) map[string]any {
> 	val := reflect.ValueOf(v)
> 	typ := reflect.TypeOf(v)
> 	if typ.Kind() == reflect.Ptr {
> 		val = val.Elem()
> 		typ = typ.Elem()
> 	}
> 
> 	result := make(map[string]any)
> 	for i := 0; i < typ.NumField(); i++ {
> 		col := columnName(typ.Field(i))
> 		result[col] = val.Field(i).Interface()
> 	}
> 	return result
> }
> 
> func main() {
> 	user := User{
> 		ID:    1,
> 		Name:  "Alice",
> 		Email: "alice@example.com",
> 		Age:   30,
> 	}
> 
> 	fmt.Printf("Columns: %v\n", Columns(user))
> 	fmt.Printf("Map: %v\n", ToMap(user))
> }
> ```
