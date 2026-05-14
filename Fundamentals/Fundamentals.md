# 1. Variables & Constants
---
## Variables
```go
var x int = 10
y := 20
```

### `:=`
- Only inside functions
- Performs type inference
- Most common style in Go
- `:=` inside `if`/`for` silently shadows outer variables

1. What is Shadowing in Go?

Variable shadowing occurs when a variable declared within a smaller, inner scope (like a block `{}` or `if` condition) has the same name as a variable in an outer scope. [shadowing](https://dev.to/rijultp/the-hidden-bug-in-go-variable-shadowing-explained-4e6f)

- **The Shadow:** The new inner variable.
- **The Shadowed:** The original outer variable (which becomes temporarily inaccessible or "hidden"). 

2. "Silently" Shadowing via `:=`

The `:=` operator declares and initializes a new variable. If you use it to create a variable that already exists outside the current scope, Go does not throw an error; it just creates a new one, shadowing the old one

```go
package main
import "fmt"

func main() {
    err := doSomething() // Outer variable 'err'
    if err != nil {
        // ERROR: We want to reassign the outer 'err',
        // but := creates a NEW 'err' only for this block.
        err := doSomethingElse() 
        fmt.Println(err) // Prints the new 'err'
    }
    // The outer 'err' is still nil, not the result of doSomethingElse()
    fmt.Println(err) 
}
```

---
## Zero Values

Every variable gets initialized automatically.

|Type|Zero Value|
|---|---|
|int|0|
|string|""|
|bool|false|
|pointer|nil|
|slice|nil|
|map|nil|
|interface|nil|
## Important
```go
var s []int
fmt.Println(s == nil) // true
```
Nil slice is valid.

You can:
```go
s = append(s, 1)
```

But nil map is NOT writable:

```go
var m map[string]intm["a"] = 1 // panic
```

Need:

```go
m = make(map[string]int)
```

---
## Constants
```go
const Pi = 3.14
```
### Untyped Constants

Untyped constants (`const N = 3`) are flexible — assignable to int32, int64, float64 etc. Typed ones are strict.
```go
const x = 3
```
Can behave as:
- int
- float
- complex
depending on context.

> Multi-assign `x, err := f()` is valid even if `x` exists, as long as at least one var is new.

**EXAMPLE:**
```go
var x int          // zero value: 0
var s string       // zero value: ""
var p *int         // zero value: nil
y := 42            // short decl — only inside functions

const MaxRetries = 3          // untyped — fits any numeric type
const Pi float64 = 3.14       // typed — only float64

type Direction int
const (
    North Direction = iota  // 0
    East                    // 1
    South                   // 2
)
```
# 2. Pointers
---
## Basics

```go
x := 10
p := &x
fmt.Println(*p)
```

## No Pointer Arithmetic

Unlike C/C++.

Safer memory model.

## Most Important Thing

Go passes EVERYTHING by value.

Even pointers.

```go
func update(p *int) {    *p = 20}
```

Pointer itself copied,  
but points to same memory.

---
## Why Use Pointers?

### Avoid copying large structs

``` go
func process(u *User)
```

### Mutate original value

### Method receiver optimization

---
## Escape Analysis

Critical concept.

```go
func test() *int {
    x := 10
    return &x
}
```

Compiler moves `x` to heap automatically.

You don't manage memory manually.

#### Breakdown of `n := new(int)`
---
- **`n`**: The variable `n` is created. Its type is `*int` (a pointer to an integer).
- **`new(int)`**: This allocates enough space on the heap to hold one integer, sets that memory to \(0\), and returns the address of that memory.
- **Result**: `n` points to an address where the value is \(0\).
```go
package main

import "fmt"

func main() {
    n := new(int)
    fmt.Println(n)  // Prints the address (e.g., 0x408010)
    fmt.Println(*n) // Prints the value: 0
    
    *n = 42         // You can now dereference and assign values
    fmt.Println(*n) // Prints: 42
}

```

**Why Use `new(int)`?**

It is used to ensure a pointer is not `nil`. If you declare `var ptr *int`, the value of `ptr` is `nil`, and trying to set `*ptr = 10` will cause a runtime panic. `new(int)` ensures `n` holds a valid memory address.

> _**Note:** `new` is rarely used just for integers, but it is often used for allocating structs or memory where a pointer to a zeroed type is required._ 

## Pointer Receiver vs Value Receiver
### Value Receiver

```go
func (u User) Print()
```

Copies struct.

---

### Pointer Receiver

```go
func (u *User) Update()
```

Mutates original.

---
#### Rule of Thumb

Use pointer receivers when:

- struct large
- mutation needed
- consistency with other methods
- Be consistent — if any method on a type is pointer receiver, make them all pointer receivers.

**Gotchas:**

- Classic loop variable capture bug: `for _, v := range s { go func() { use(&v) }() }` — all goroutines share the same `v`.
- Returning a pointer to a local variable is safe — Go moves it to the heap (escape analysis).

```go
x := 10
p := &x      // *int
*p = 20      // x is now 20

n := new(int)   // *int pointing to zeroed int

type Point struct{ X, Y int }
pt := &Point{1, 2}
pt.X = 99    // sugar for (*pt).X = 99
```
# 3. Arrays vs Slices
# Arrays

Fixed size.

```go
var arr [3]int
```

Size is part of type.

```go
[3]int != [4]int
```

Rarely used directly. Array of 3 integers and an array of 4 integers are **entirely different types**

# Slices

Most important data structure in Go.

```go
s := []int{1,2,3}
```

Slice internally:

```
ptr -> backing array
len
cap
```

## Length vs Capacity

```
s := make([]int, 2, 5)
```

- len = 2
- cap = 5
## Append Mechanics

```
s = append(s, 10)
```

The `append`function’s behavior depends on the slice’s capacity:

- **Within Capacity**: If the new length fits within the capacity, the shared array is modified, affecting the original slice (up to its length).
- **Beyond Capacity**: If the capacity is exceeded, a new array is allocated. The function’s slice points to this new array, and changes no longer affect the original.
## Example Code Analysis

### Scenario 1: Length=3, Capacity=4
```go
x := make([]int, 3, 4)  // [1, 2, 0, _]  
x[0], x[1] = 1, 2  
fmt.Println("Before:", x)  // [1 2 0]  
y := update(x)  
fmt.Println("After:", x, y)  // [1 99 88] [1 99 88 4]
```

Inside `update`:
```go
func update(x []int) []int {  
    x[1] = 99          // Array: [1, 99, 0, _]  
    x = append(x, 4)   // Array: [1, 99, 0, 4] (within capacity)  
    x[2] = 88          // Array: [1, 99, 88, 4]  
    return x  
}
```

Since `append` stays within capacity (4), the shared array is modified. The original `x` (length 3) shows `[1, 99, 88]`, and `y` (length 4) shows `[1, 99, 88, 4]`.

### Scenario 2: Length=3, Capacity=3
```go
x := make([]int, 3, 3)  // [1, 2, 3]  
x[0], x[1], x[2] = 1, 2, 3  
fmt.Println("Before:", x)  // [1 2 3]  
y := update(x)  
fmt.Println("After:", x, y)  // [1 99 3] [1 99 88 4]
```

Inside `update`:

1. `x[1] = 99` → `[1, 99, 3]`.
2. `x = append(x, 4)` → New array: `[1, 99, 3, 4]` (exceeds capacity).
3. `x[2] = 88` → New array: `[1, 99, 88, 4]`.

The original x stays `[1, 99, 3]` (old array), while `y` is `[1, 99, 88, 4]` (new array).

## Slice Sharing Bug
---
The **Slice Sharing Bug** (or "sneaky mutation") occurs because a slice in Go is just a "view" or a "header" that points to an **underlying array**. When you create multiple slices from the same original slice, they often share that same memory

The "bug" appears when you modify one slice and accidentally change another, or when an `append` operation behaves inconsistently

1. **The Direct Mutation Bug**
If you create a sub-slice, any change to its elements directly modifies the original because they share the same physical memory addresses

```go
original := []int{1, 2, 3, 4, 5}
sub := original[1:3] // View of {2, 3}

sub[0] = 99
fmt.Println(original) // [1, 99, 3, 4, 5] — The original is "mutated"!

```

2. **The "Ghost Append" Bug**
This is the most confusing version. When you `append` to a slice that has extra **capacity**, Go will overwrite the next slot in the underlying array instead of allocating new space

```go
s1 := []int{1, 2}           // Len: 2, Cap: 2 (usually)
s1 = append(s1, 3)          // Cap expands (e.g., to 4)
// s1 is now [1, 2, 3]

s2 := append(s1, 4)         // s2 uses s1's extra capacity; sets next slot to 4
s3 := append(s1, 5)         // s3 ALSO uses s1's extra capacity; overwrites 4 with 5!

fmt.Println(s2) // [1, 2, 3, 5] — The '4' disappeared!
fmt.Println(s3) // [1, 2, 3, 5]

```
In this case, `s2` and `s3` share the same capacity slot. Writing to `s3` overwrote the value in `s2` because the underlying array wasn't full yet.

3. **Memory Leak Bug**

If you take a tiny slice of a massive array (like reading 10 bytes from a 1GB file), the entire 1GB array stays in memory as long as that tiny slice exists. The Garbage Collector cannot free the 1GB because the tiny slice still "references" that shared memory. 

## Full Slice Expression
```go
b := a[:2:2]
```

###### **How it works**

The syntax is `a[low:high:max]`.

- **`high`** sets the **length** (how many elements you see).
- **`max`** sets the **capacity** (how much of the underlying array you "own").

By setting `high` and `max` to the same number (like `a[:2:2]`), you tell Go: "This slice is 2 elements long, and its capacity is also 2."

###### **Why it’s "Defensive"**

When you call `append()` on a slice, Go checks if there is room in the capacity.

1. **Without the 3rd index:** `b := a[:2]` likely has leftover capacity from `a`. If you `append` to `b`, it will overwrite the data in `a[2]`.
2. **With the 3rd index:** `b := a[:2:2]` has **no room**. If you `append` to `b`, Go is forced to:
    - Allocate a **brand new** underlying array.
    - Copy the data over.
    - Modify only the new array.
```go
a := []int{1, 2, 3, 4}
b := a[:2:2] // b is [1, 2], capacity is 2

b = append(b, 99) 

fmt.Println(a) // [1, 2, 3, 4] - SAFE! Original is untouched.
fmt.Println(b) // [1, 2, 99]    - New array created.
```

## Copying Slices

```go
copy(dst, src)
```

Needed for true isolation.

| Feature     | Array               | Slice                                     |
| ----------- | ------------------- | ----------------------------------------- |
| Size        | Fixed (`[5]int`)    | Dynamic                                   |
| Memory      | Contiguous, value   | Header (ptr + len + cap)                  |
| Declaration | `[3]int{1,2,3}`     | `[]int{1,2,3}` or `make([]int, len, cap)` |
| Passing     | Copies entire array | Copies header (cheap)                     |
> `nil` slice vs empty slice: `var s []int` is nil; `s := []int{}` is non-nil. `json.Marshal` encodes nil → `null`, empty → `[]`.

```go
// Array — fixed size, VALUE type
arr := [3]int{1, 2, 3}
arr2 := arr        // full copy — independent

// Slice — descriptor: (ptr, len, cap)
s := []int{1, 2, 3}
s2 := s[:2]        // shares underlying array!
s2[0] = 99         // mutates s[0] too

s = append(s, 4)   // may or may not reallocate

// Safe independent copy
dst := make([]int, len(s))
copy(dst, s)

// 3-index slice — cap the sub-slice to prevent silent overwrites
t := s[1:3:3]      // len=2, cap=2 — append will always allocate new

// Pre-allocate
result := make([]string, 0, 100)
```

# 4. Maps
## Internal Structure

Hash table.

Average:

- O(1) lookup
- O(1) insert

#### Creation

```go
m := make(map[string]int)
```

---
## Key Rules

Keys must be comparable.

Allowed:

- string
- int
- bool
- structs (if fields comparable)

NOT allowed:

- slices
- maps
- functions

- Can't take address of map value: `&m["key"]` is a compile error.
-  Struct values in maps are not addressable — you can't do `m["k"].Field = x`. Copy out, mutate, store back.
## Lookup

```
v, ok := m["x"]
```

Critical pattern.

---

## Map Iteration

Order NOT guaranteed.

```go
for k, v := range m
```

Never rely on ordering.

---
## Map Internals

Go maps:

- resize automatically
- use buckets
- may rehash

Concurrent writes are unsafe.

## Concurrency Danger

This:

```go
go m["x"] = 1
```

without synchronization can crash.

Need:
- mutex
- sync.Map
- channel ownership

```go
var m map[string]int  // nil map — reads OK, writes PANIC
m = make(map[string]int)

v := m["x"]           // zero value (0), never panics
v, ok := m["a"]       // comma-ok — existence check
delete(m, "a")

for k, v := range m { ... }  // iteration order is RANDOM
```

# 5. Structs
---
## Basics

```go
type User struct {
    Name string
    Age  int
}
```

---

## Structs are Value Types

Assignment copies entire struct.

```go
u2 := u1
```

---
## Anonymous Structs

```go
u := struct {
    Name string
}{}
```

Useful for:
- temporary JSON responses
- tests

## Tags

```go
type User struct {    Name string `json:"name"`}
```

Used via reflection.

## Composition Over Inheritance

Go prefers embedding.

>  Struct containing a slice/map: copying the struct shares the underlying data.

>   Unexported fields are invisible to `json.Marshal`.

```go
type User struct {
    ID     int
    Name   string   `json:"name" db:"name"`  // struct tags
    Address                                   // anonymous/embedded
    secret string                             // unexported
}

u := User{Name: "Alice", Address: Address{City: "NY"}}
u.City  // promoted field — same as u.Address.City

// Comparable only if all fields are comparable
type Point struct{ X, Y int }
p1 == p2  // valid

// Anonymous struct
cfg := struct{ Host string; Port int }{"localhost", 8080}
```

---
# 6. Interfaces
---
# MOST IMPORTANT GO CONCEPT

## Interface Definition

```go
type Writer interface {    
	Write([]byte) (int, error)
}
```

## Implicit Implementation

No `implements` keyword.

```go
type File struct {}

func (f File) Write(b []byte)(int,error)
```

Automatically satisfies interface.

## Interface Internals

Interface contains:
```
type
value
```

In Go, an interface is not just a single pointer. Under the hood, it is a small struct (often called an `iface` or `eface`) that contains two distinct fields:

1. **Type:** The concrete type of the value being stored (e.g., `*User`).
2. **Value:** The actual data or the address of the instance (e.g., `nil`).

## Nil Interface Trap

```go
var p *User = nil
var i interface{} = p

fmt.Println(i == nil) // false
```
For an interface to be truly `nil`, **both** fields must be empty.

The interface `i` is populated like this:

- **Type:** `*User`
- **Value:** `nil`

Because the **Type** field is now filled with `*User`, the interface struct itself is no longer "empty." When you check `i == nil`, Go checks the whole interface structure and sees that it’s not completely empty, so it returns `false`.

**Why this is a "Trap"**

This usually causes bugs in error handling:
```go
func GetError() error {
    var err *MyCustomError = nil 
    return err // Returns an interface with Type=*MyCustomError, Value=nil
}

func main() {
    err := GetError()
    if err != nil {
        // This code RUNS because the interface is not nil!
        fmt.Println("Error found!") 
    }
}

```

#### **How to avoid it**
- **Return `nil` directly:** Instead of returning a typed variable that happens to be nil, literally write `return nil`.
- **Type check:** If you must check for this, you have to use reflection or a type assertion to check if the underlying _value_ is nil.

## Empty Interface

```go
interface{}
```

Means:  
"any type"

Now replaced mostly by:

```go
any
```

## Best Practice

Interfaces should usually be:

- small
- behavior-focused

Good:

```go
io.Reader
```

Bad:

```go
UserServiceManagerRepositoryController
```

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}

// Satisfied implicitly
type FileWriter struct{}
func (f FileWriter) Write(p []byte) (int, error) { return len(p), nil }

// Interface internals: (type, value) pair
var w Writer           // (nil, nil) — truly nil
var f *FileWriter      // nil pointer of type *FileWriter
w = f                  // (type=*FileWriter, value=nil) — NOT nil!
fmt.Println(w == nil)  // false ← classic trap
```

# 7. Methods
---
## Method Syntax

```go
func (u User) Print()
```

Receiver not special.

Just syntax sugar.

Equivalent idea:

```go
Print(u)
```

## Method Sets
---
### Value Receiver Methods

Available on:

- value
- pointer

### Pointer Receiver Methods

Available only on:

- pointer

(but compiler may auto-take address)

#### 1. **The Method Set Rules**

| Receiver Type                 | Method Set of Value (`T`) | Method Set of Pointer (`*T`) |
| ----------------------------- | ------------------------- | ---------------------------- |
| **Value Receiver** `(t T)`    | ✅ Included                | ✅ Included                   |
| **Pointer Receiver** `(t *T)` | ❌ **NOT Included**        | ✅ Included                   |

#### 2. Why the Difference?

Go is designed for safety.

- **Pointer to Value:** If you have a pointer (`*T`) but the method needs a value (`T`), Go can easily grab the data at that address (dereference it) and pass a copy. **Safe.**
- **Value to Pointer:** If you have a value (`T`) but the method needs a pointer (`*T`), Go would need to take the address of that value (`&T`). However, many values are **not addressable** (like a value returned from a map or a temporary constant). To avoid inconsistent behavior, Go simply forbids values from using pointer-receiver methods in interfaces.

```go
type Shaper interface {
    Draw()
}

type Circle struct{ Radius int }

// Method has a POINTER receiver
func (c *Circle) Draw() { fmt.Println("Drawing...") }

func main() {
    var s Shaper
    
    cValue := Circle{5}
    s = cValue // ❌ COMPILE ERROR: Circle does not implement Shaper 
               // (Draw method has pointer receiver)

    cPtr := &Circle{5}
    s = cPtr   // ✅ WORKS: *Circle has Draw in its method set
}

```

Summary

- **Pointer receivers** are "strict." Only pointers can use them to satisfy interfaces.
- **Value receivers** are "flexible." Both values and pointers can use them to satisfy interfaces.

```go
type Counter struct{ n int }

func (c Counter) Value() int  { return c.n }     // value receiver — copy
func (c *Counter) Increment() { c.n++ }          // pointer receiver — mutates

c := Counter{}
c.Increment()   // Go auto-takes address: (&c).Increment()

// Method value (bound)
inc := c.Increment   // captures c

// Method expression (unbound)
valFn := Counter.Value   // func(Counter) int

// Methods on non-struct types
type Celsius float64
func (c Celsius) ToF() float64 { return float64(c)*9/5 + 32 }
```

# 8. Embedding
---
### 1. What is Composition?
Instead of saying a `Manager` "is-a" `Employee` (inheritance), you say a `Manager` "has-a" `Employee` (composition). In Go, you do this by including one struct inside another without giving it a field name.

```go
type Employee struct {
    Name string
}

func (e Employee) Work() { fmt.Println(e.Name, "is working") }

type Manager struct {
    Employee // <--- Anonymous/Embedded field
    Level    int
}
```

### 2. What are Promoted Methods?
When you embed a struct (like `Employee`) into a parent struct (like `Manager`), the methods of the embedded struct are **promoted** to the parent.

This means you can call `Work()` directly on a `Manager` instance, even though `Work()` is defined on `Employee`.

```go
m := Manager{
    Employee: Employee{Name: "Alice"},
    Level:    1,
}

m.Work() // Works! Prints: "Alice is working"

```

**Key features of promotion:**

- **Direct Access:** You don't have to write `m.Employee.Work()`; Go handles the shortcut for you.
- **Shadowing:** If `Manager` also defines a method called `Work()`, it "shadows" the embedded one. Calling `m.Work()` will run the Manager's version, but you can still access the original via `m.Employee.Work()`.
- **Interface Satisfaction:** If `Employee` satisfies an interface, `Manager` now satisfies it too because the methods are promoted.

### 3. Rules for Promoted Method Sets
Promotion follows the same **Method Set** rules we discussed earlier, based on how you embed the struct:

- **Embed by Value (`Employee`):**
    - Value methods of `Employee` are promoted to both `Manager` and `*Manager`.
    - Pointer methods of `Employee` are promoted **only** to `*Manager`.
- **Embed by Pointer (`*Employee`):**
    - All methods (value and pointer) of `Employee` are promoted to both `Manager` and `*Manager`.

## NOT Inheritance

No polymorphic inheritance hierarchy.

Embedding = field promotion.

## Common Usage

- reusable behavior
- middleware-like composition
- config layering

```go
type Animal struct{ Name string }
func (a Animal) Speak() string { return a.Name + " speaks" }

type Dog struct {
    Animal        // embedded — not a named field
    Breed string
}

d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Lab"}
d.Speak()         // promoted: calls d.Animal.Speak()
d.Name            // promoted field

// Override — Dog's own Speak shadows Animal's
func (d Dog) Speak() string { return "Woof!" }
d.Animal.Speak()  // still accessible explicitly
```

# 9. Type Assertions
---
Used with interfaces.

```go
v := i.(string)
```

Dangerous version:
- panics if wrong

## Safe Version

```go
v, ok := i.(string)
```

Preferred.

```go
var i interface{} = "hello"

s := i.(string)           // unsafe — panics if wrong type

s, ok := i.(string)       // safe — comma-ok
if !ok { /* handle */ }

// Assert to interface
if stringer, ok := i.(fmt.Stringer); ok {
    fmt.Println(stringer.String())
}

// For errors — prefer errors.As
var target *os.PathError
if errors.As(err, &target) {
    fmt.Println(target.Path)
}
```

**Gotchas:**

- Asserting on a **nil interface** always panics, even with comma-ok.
- `x.(T)` only compiles when `x` is an interface type.

# 10. Type Switches
Cleaner multiple assertions.

```go
switch v := i.(type) {
case string:
case int:
default:
}
```

---

## Very Common In

- error handling
- serialization
- middleware
- dynamic processing

```go
func describe(i interface{}) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("int: %d", v)
    case string:
        return fmt.Sprintf("string: %q", v)
    case bool, float64:     // multi-type case
        return "bool or float64"  // v is interface{} here
    case nil:
        return "nil"
    default:
        return fmt.Sprintf("unknown: %T", v)
    }
}
```

**Gotchas:**

- In a multi-type case (`case bool, float64`), `v` is typed as `interface{}`, not either concrete type.
- Prefer `errors.Is` / `errors.As` over type-switching on errors — type switches break with wrapped errors.

# 11. Generics
---
#### Basic Syntax
```go
func Print[T any](v T) {
    fmt.Println(v)
}
```

#### Constraints
```go
type Number interface {
    int | float64
}
```

## Why Generics Matter

Before generics:

- interface{}
- reflection
- code duplication

Now:

- type-safe reusable code

## Common Generic Use Cases

### Data structures

```go
Stack[T]
```

### Utility functions

```go
Map[T]Filter[T]
```

### Repositories

### Pipelines

---
```go
// Generic function
func Map[T, R any](s []T, f func(T) R) []R {
    result := make([]R, len(s))
    for i, v := range s { result[i] = f(v) }
    return result
}

// Union constraint
type Number interface { int | int32 | int64 | float64 }

// ~T — includes all types whose underlying type is T
type Signed interface { ~int | ~int8 | ~int16 | ~int32 | ~int64 }

// Generic struct
type Stack[T any] struct{ items []T }
func (s *Stack[T]) Push(v T)      { s.items = append(s.items, v) }
func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 { var z T; return z, false }
    n := len(s.items) - 1
    v := s.items[n]; s.items = s.items[:n]
    return v, true
}

// comparable constraint — needed for == and map keys
func Contains[T comparable](s []T, v T) bool {
    for _, x := range s { if x == v { return true } }
    return false
}
```

### **Gotchas:**
- **Zero value of T**: use `var z T; return z` — you can't use `nil` for a generic type.

- **Without `~` (Exact Match)**: If you define a constraint without the tilde, the compiler is strict. Only the literal type `int` will work.
```go
// Only accepts the literal type 'int'
func Add[T int](a, b T) T {
    return a + b
}

type MyInt int // Defined a custom type

func main() {
    var x int = 5
    var y MyInt = 10

    Add(x, x) // ✅ Works
    Add(y, y) // ❌ COMPILE ERROR: MyInt does not implement int
}

```

- **With `~` (Underlying Type Match)**
Adding the `~` tells Go: "Accept `int` **and** any type that was created with `int` as its foundation."
```go
// Accepts int AND any type whose underlying type is int
func Add[T ~int](a, b T) T {
    return a + b
}

type MyInt int

func main() {
    var y MyInt = 10
    Add(y, y) // ✅ Works! Because MyInt's underlying type is int.
}

```
In Go, `type MyInt int` creates a **new, distinct type**. Even though it's just an integer, it doesn't automatically "act" like one in generic constraints unless you use the approximation operator.


- While a **struct** can be generic, ==the **methods** attached to it cannot introduce _new_ generic types that aren't already defined by the struct==.
	1. **What you CANNOT do:**
		You cannot add a new type parameter to a method signature that wasn't declared at the struct level.

```go
type List[T any] struct {
    elements []T
}

// ❌ COMPILE ERROR: Methods cannot have type parameters
func (l *List[T]) Map[U any](f func(T) U) []U {
    // ...
}
```
In the example above, `T` is allowed because it belongs to the struct, but `U` is forbidden because it is introduced specifically for the `Map` method.
	**2. The Workaround: Top-level Functions**
		If you need a generic operation that involves a new type (like transforming a `List[int]` into a `List[string]`), you must use a **standalone function** instead of a method.

```go
// ✅ Use a standalone function instead of a method
func Map[T any, U any](list []T, f func(T) U) []U {
    result := make([]U, len(list))
    for i, v := range list {
        result[i] = f(v)
    }
    return result
}
```

>**Pro-tip:** If you find yourself needing a method with its own type parameter, it’s usually a signal that the logic should be a **standalone utility function**

- In Go generics, a **type parameter** is treated as a single, fixed type at compile-time after instantiation. Because of this, ==Go does not allow you to perform a **type switch** directly on a variable whose type is a type parameter== (e.g., `T`)

#### Why this restriction exists

###### 1. “Fixed Identity” (Simplified)
👉 Type switch is for **runtime type checking**

👉 Generics in Go are **compile-time**

> Inside the function, `T` is already decided.

So Go is like:

> “Why are you asking what type it is? You already know it.”

### Example

```go
func Print[T int | string](v T)
```

If called as:

```go
Print[int](10)
```

👉 Inside function:

```
v is int — ALWAYS
```

So this:

```go
switch v.(type)
```

##### 2. Why interfaces allow type switch

```go
func Print(v any) {    
	switch v.(type) {    
		case int:    
		case string:    
	}
}
```

👉 Here:
- `v` can be **anything at runtime**
- So type switch makes sense

###### 3. The confusing part (~ underlying types)
Example:
```go
type MyInt intfunc Test[T ~int](v T)
```

Now:

```go
Test(MyInt(10))
```

---

👉 Question:  
Should this match:

```go
case int:
```

OR

```go
case MyInt:
```

💥 Confusing.

Go designers said:

> “Let’s just NOT allow this confusion”


**The Workaround**

To "break out" of the generic constraint and inspect the type at runtime, you must first convert the variable to the **`any`** (empty interface) type
```go
func DoSomething[T any](v T) {
    // ❌ Error: cannot use type switch on type parameter value v
    /* 
    switch v.(type) {
    case string:
        fmt.Println("It's a string")
    }
    */

    // ✅ Correct: Convert to 'any' first
    switch any(v).(type) {
    case string:
        fmt.Println("It's a string")
    case int:
        fmt.Println("It's an int")
    }
}

```

