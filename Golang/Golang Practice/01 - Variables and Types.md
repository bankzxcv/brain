---
title: "Go Practice: Variables and Types"
date: 2026-04-07
tags:
  - golang
  - practice
  - variables
  - types
parent: "[[Golang Study]]"
---

# Go Practice: Variables and Types

13 progressive problems. Write each solution from scratch to build muscle memory.

---

## P1: Declare Variables

Declare a `string`, an `int`, and a `bool` using both `var` and `:=` syntax. Print all six variables.

**Expected output:**
```
hello 42 true
world 99 false
```

> [!hint]- Hint
> `var name string = "hello"` is the long form. `name := "hello"` is the short form. Short form can only be used inside functions.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	var s1 string = "hello"
> 	var n1 int = 42
> 	var b1 bool = true
> 
> 	s2 := "world"
> 	n2 := 99
> 	b2 := false
> 
> 	fmt.Println(s1, n1, b1)
> 	fmt.Println(s2, n2, b2)
> }
> ```

---

## P2: Zero Values

Declare variables of type `int`, `float64`, `string`, `bool`, and `*int` without initializing them. Print each one and predict the output before running.

**Expected output:**
```
int: 0
float64: 0
string: ""
bool: false
pointer: <nil>
```

> [!hint]- Hint
> Every type in Go has a zero value. Numeric types are `0`, strings are `""`, booleans are `false`, pointers are `nil`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	var i int
> 	var f float64
> 	var s string
> 	var b bool
> 	var p *int
> 
> 	fmt.Printf("int: %d\n", i)
> 	fmt.Printf("float64: %g\n", f)
> 	fmt.Printf("string: %q\n", s)
> 	fmt.Printf("bool: %t\n", b)
> 	fmt.Printf("pointer: %v\n", p)
> }
> ```

---

## P3: Type Conversions

Given `x := 42` and `y := 3.14`, compute `z = x + y` as a `float64`. Then convert `z` to an `int` and to a `string` representation. Print all results.

**Expected output:**
```
z = 45.14
z as int = 45
z as string = "45.14"
```

> [!hint]- Hint
> Use `float64(x)` to convert int to float. Use `int(z)` to truncate. Use `fmt.Sprintf` or `strconv.FormatFloat` for float-to-string.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strconv"
> )
> 
> func main() {
> 	x := 42
> 	y := 3.14
> 
> 	z := float64(x) + y
> 	fmt.Printf("z = %.2f\n", z)
> 
> 	zInt := int(z)
> 	fmt.Printf("z as int = %d\n", zInt)
> 
> 	zStr := strconv.FormatFloat(z, 'f', 2, 64)
> 	fmt.Printf("z as string = %q\n", zStr)
> }
> ```

---

## P4: Constants and Iota

Define a set of constants for days of the week using `iota`, starting with `Sunday = 0`. Print the name and numeric value of `Wednesday`.

**Expected output:**
```
Wednesday = 3
```

> [!hint]- Hint
> Use a `const` block with `iota`. The first constant gets `iota` (which is 0), and subsequent ones auto-increment.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Day int
> 
> const (
> 	Sunday Day = iota
> 	Monday
> 	Tuesday
> 	Wednesday
> 	Thursday
> 	Friday
> 	Saturday
> )
> 
> func (d Day) String() string {
> 	names := [...]string{
> 		"Sunday", "Monday", "Tuesday", "Wednesday",
> 		"Thursday", "Friday", "Saturday",
> 	}
> 	if d < Sunday || d > Saturday {
> 		return "Unknown"
> 	}
> 	return names[d]
> }
> 
> func main() {
> 	fmt.Printf("%s = %d\n", Wednesday, int(Wednesday))
> }
> ```

---

## P5: String, []byte, []rune

Given the string `s := "Hello, Go!"`, convert it to `[]byte` and `[]rune`. Print the length of each. Then modify the byte slice to change `'G'` to `'N'` and print the resulting string.

**Expected output:**
```
string len: 10
byte len: 10
rune len: 10
Modified: Hello, No!
```

> [!hint]- Hint
> `[]byte(s)` and `[]rune(s)` perform the conversions. For ASCII strings, byte and rune lengths are equal. Find the index of `'G'` and replace it.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	s := "Hello, Go!"
> 
> 	b := []byte(s)
> 	r := []rune(s)
> 
> 	fmt.Printf("string len: %d\n", len(s))
> 	fmt.Printf("byte len: %d\n", len(b))
> 	fmt.Printf("rune len: %d\n", len(r))
> 
> 	// Change 'G' to 'N'
> 	for i, ch := range b {
> 		if ch == 'G' {
> 			b[i] = 'N'
> 			break
> 		}
> 	}
> 	fmt.Printf("Modified: %s\n", string(b))
> }
> ```

---

## P6: Count Unicode Characters

Given `s := "สวัสดีครับ"` (Thai for "hello"), print the number of bytes, the number of runes, and each rune with its byte position.

**Expected output (approximate):**
```
bytes: 27
runes: 9
[0] ส
[3] ว
[6] ั
[9] ส
[12] ด
[15] ี
[18] ค
[21] ร
[24] ั
[27] บ
```

> [!hint]- Hint
> Use `len(s)` for byte count and `utf8.RuneCountInString(s)` for rune count. A `for range` loop over a string iterates by rune, giving you the byte index and the rune value.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"unicode/utf8"
> )
> 
> func main() {
> 	s := "สวัสดีครับ"
> 
> 	fmt.Printf("bytes: %d\n", len(s))
> 	fmt.Printf("runes: %d\n", utf8.RuneCountInString(s))
> 
> 	for i, r := range s {
> 		fmt.Printf("[%d] %c\n", i, r)
> 	}
> }
> ```

---

## P7: Variable Shadowing

**Without running the code**, predict the output. Then verify by running it.

```go
package main

import "fmt"

func main() {
	x := 1
	fmt.Println("A:", x)
	{
		x := 2
		fmt.Println("B:", x)
		{
			x := x * 3
			fmt.Println("C:", x)
		}
		fmt.Println("D:", x)
	}
	fmt.Println("E:", x)
}
```

> [!hint]- Hint
> Each `:=` inside a new block creates a *new* variable that shadows the outer one. Changes to the inner `x` do not affect the outer `x`.

> [!success]- Solution
> **Output:**
> ```
> A: 1
> B: 2
> C: 6
> D: 2
> E: 1
> ```
> 
> Explanation:
> - `A`: outer `x = 1`
> - `B`: first inner block shadows `x = 2`
> - `C`: second inner block shadows `x = 2 * 3 = 6`
> - `D`: back to first inner block, `x` is still `2`
> - `E`: back to outer, `x` is still `1`

---

## P8: Multiple Assignment and Swap

Declare `a, b := 10, 20`. Swap their values **without** using a temporary variable. Print before and after.

**Expected output:**
```
Before: a=10, b=20
After:  a=20, b=10
```

> [!hint]- Hint
> Go supports multiple assignment in a single statement: `a, b = b, a`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	a, b := 10, 20
> 	fmt.Printf("Before: a=%d, b=%d\n", a, b)
> 
> 	a, b = b, a
> 	fmt.Printf("After:  a=%d, b=%d\n", a, b)
> }
> ```

---

## P9: Typed Constants with Custom Type

Define a `Celsius` type based on `float64`. Create constants for `FreezingPoint` (0) and `BoilingPoint` (100). Write a `ToFahrenheit()` method and print both points in Fahrenheit.

**Expected output:**
```
Freezing: 0.00°C = 32.00°F
Boiling:  100.00°C = 212.00°F
```

> [!hint]- Hint
> `type Celsius float64` creates a named type. You can define methods on it: `func (c Celsius) ToFahrenheit() float64`. The formula is `F = C*9/5 + 32`.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Celsius float64
> 
> const (
> 	FreezingPoint Celsius = 0
> 	BoilingPoint  Celsius = 100
> )
> 
> func (c Celsius) ToFahrenheit() float64 {
> 	return float64(c)*9.0/5.0 + 32.0
> }
> 
> func main() {
> 	fmt.Printf("Freezing: %.2f°C = %.2f°F\n", FreezingPoint, FreezingPoint.ToFahrenheit())
> 	fmt.Printf("Boiling:  %.2f°C = %.2f°F\n", BoilingPoint, BoilingPoint.ToFahrenheit())
> }
> ```

---

## P10: Bitwise Operations with Iota for Permission Flags

Define permission flags `Read`, `Write`, `Execute` using `iota` and bit shifting. Create a variable with `Read | Write` permission. Write functions to check and display which permissions are set.

**Expected output:**
```
Permissions: Read | Write
Has Read: true
Has Write: true
Has Execute: false
```

> [!hint]- Hint
> Use `1 << iota` to get bit flags: `1`, `2`, `4`. Check a flag with `perm & flag != 0`. Use a custom type like `type Permission uint8`.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> type Permission uint8
> 
> const (
> 	Read    Permission = 1 << iota // 1
> 	Write                          // 2
> 	Execute                        // 4
> )
> 
> func (p Permission) String() string {
> 	var parts []string
> 	if p&Read != 0 {
> 		parts = append(parts, "Read")
> 	}
> 	if p&Write != 0 {
> 		parts = append(parts, "Write")
> 	}
> 	if p&Execute != 0 {
> 		parts = append(parts, "Execute")
> 	}
> 	if len(parts) == 0 {
> 		return "None"
> 	}
> 	return strings.Join(parts, " | ")
> }
> 
> func (p Permission) Has(flag Permission) bool {
> 	return p&flag != 0
> }
> 
> func main() {
> 	perm := Read | Write
> 
> 	fmt.Printf("Permissions: %s\n", perm)
> 	fmt.Printf("Has Read: %t\n", perm.Has(Read))
> 	fmt.Printf("Has Write: %t\n", perm.Has(Write))
> 	fmt.Printf("Has Execute: %t\n", perm.Has(Execute))
> }
> ```

---

## P11: String Builder for Efficient Concatenation

Build a comma-separated string from the slice `[]string{"apple", "banana", "cherry", "date"}` using `strings.Builder`. Do not include a trailing comma.

**Expected output:**
```
apple, banana, cherry, date
```

> [!hint]- Hint
> `strings.Builder` has `WriteString` and `String()` methods. Track the index to know when to skip the comma (e.g., skip it for the first element, or don't add it after the last).

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> func main() {
> 	fruits := []string{"apple", "banana", "cherry", "date"}
> 
> 	var sb strings.Builder
> 	for i, fruit := range fruits {
> 		if i > 0 {
> 			sb.WriteString(", ")
> 		}
> 		sb.WriteString(fruit)
> 	}
> 
> 	fmt.Println(sb.String())
> }
> ```

---

## P12: Parse "KEY=VALUE" Strings

Write a function `parseEnv(s string) (key, value string, err error)` that splits a string like `"DATABASE_URL=postgres://localhost/mydb"` into key and value. Handle the case where there is no `=` sign (return an error). Handle values that contain `=` signs.

**Expected output:**
```
key="DATABASE_URL", value="postgres://localhost/mydb"
key="FEATURE_FLAGS", value="a=1&b=2"
Error: invalid format: no '=' found in "INVALID"
```

> [!hint]- Hint
> Use `strings.SplitN(s, "=", 2)` to split into at most 2 parts. This correctly handles values containing `=`. Check the length of the resulting slice.

> [!success]- Solution
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"strings"
> )
> 
> func parseEnv(s string) (key, value string, err error) {
> 	parts := strings.SplitN(s, "=", 2)
> 	if len(parts) != 2 {
> 		return "", "", fmt.Errorf("invalid format: no '=' found in %q", s)
> 	}
> 	return parts[0], parts[1], nil
> }
> 
> func main() {
> 	inputs := []string{
> 		"DATABASE_URL=postgres://localhost/mydb",
> 		"FEATURE_FLAGS=a=1&b=2",
> 		"INVALID",
> 	}
> 
> 	for _, input := range inputs {
> 		key, value, err := parseEnv(input)
> 		if err != nil {
> 			fmt.Printf("Error: %v\n", err)
> 			continue
> 		}
> 		fmt.Printf("key=%q, value=%q\n", key, value)
> 	}
> }
> ```

---

## P13: Mini Type System - Temperature Conversions

Create types `Celsius` and `Fahrenheit` (both based on `float64`). Implement methods:
- `Celsius.ToFahrenheit() Fahrenheit`
- `Fahrenheit.ToCelsius() Celsius`
- `String()` for both types (include the degree symbol and unit)

Write a `main` that converts 100°C to Fahrenheit, then back to Celsius, demonstrating the round-trip.

**Expected output:**
```
Start:     100.00°C
To F:      212.00°F
Back to C: 100.00°C
```

> [!hint]- Hint
> `C -> F: F = C*9/5 + 32` and `F -> C: C = (F-32)*5/9`. Make the methods return the other type so you can chain them.

> [!success]- Solution
> ```go
> package main
> 
> import "fmt"
> 
> type Celsius float64
> type Fahrenheit float64
> 
> func (c Celsius) ToFahrenheit() Fahrenheit {
> 	return Fahrenheit(c*9.0/5.0 + 32.0)
> }
> 
> func (c Celsius) String() string {
> 	return fmt.Sprintf("%.2f°C", float64(c))
> }
> 
> func (f Fahrenheit) ToCelsius() Celsius {
> 	return Celsius((f - 32.0) * 5.0 / 9.0)
> }
> 
> func (f Fahrenheit) String() string {
> 	return fmt.Sprintf("%.2f°F", float64(f))
> }
> 
> func main() {
> 	c := Celsius(100)
> 	fmt.Printf("Start:     %s\n", c)
> 
> 	f := c.ToFahrenheit()
> 	fmt.Printf("To F:      %s\n", f)
> 
> 	back := f.ToCelsius()
> 	fmt.Printf("Back to C: %s\n", back)
> }
> ```
