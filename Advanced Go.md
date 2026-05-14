# 1. Context (Conceptual Mastery)
---
## What Problem It Solves

> How do you control **lifecycle, cancellation, deadlines, and request-scoped data** across goroutines?

## Core Ideas

- Propagates **cancellation signals**
- Carries **deadlines/timeouts**
- Passes **request-scoped metadata**

**The interface:**
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)  // when will it cancel?
    Done() <-chan struct{}                     // closed when cancelled
    Err() error                               // why was it cancelled?
    Value(key any) any                        // request-scoped values
}
```

**The four constructors:**
```go
// Root contexts — never cancelled on their own
ctx := context.Background()  // top-level, always use this as root
ctx := context.TODO()        // placeholder when you're not sure yet — same as Background internally

// Derived contexts — form a tree
ctx, cancel := context.WithCancel(parent)       // manual cancellation
ctx, cancel := context.WithTimeout(parent, 5*time.Second)  // deadline = now + duration
ctx, cancel := context.WithDeadline(parent, t) // explicit deadline time
ctx        := context.WithValue(parent, key, val)  // attach a value
```

**Always call cancel.** It releases resources even if the context expires on its own:

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()  // always defer immediately after creation
```

**How cancellation propagates — the tree:**
```
Background
    └── WithCancel (A)
            ├── WithTimeout (B)   ← cancelled when A cancelled OR timeout
            └── WithValue (C)     ← cancelled when A cancelled
```

Cancelling a parent cancels all children. Children cannot cancel parents.

**Checking cancellation in your code:**
```go
func doWork(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()  // context.Canceled or context.DeadlineExceeded
        default:
            // do actual work
            process()
        }
    }
}

// For blocking calls, pass ctx directly
rows, err := db.QueryContext(ctx, "SELECT ...")
resp, err := http.NewRequestWithContext(ctx, "GET", url, nil)
```

**`ctx.Err()` values:**
```go
context.Canceled          // parent called cancel()
context.DeadlineExceeded  // timeout hit
```

**Values in context — the right way:**
```go
// Define unexported key type to avoid collisions across packages
type contextKey string
const requestIDKey contextKey = "requestID"

func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}

func RequestIDFrom(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(requestIDKey).(string)
    return id, ok
}
```

**What context is NOT for:**

- Passing optional function arguments (use functional options for that).
- Passing database connections or loggers (put those in structs).
- Storing business logic data — only request-scoped cross-cutting concerns (request ID, auth token, trace ID).

**Gotchas:**

- **Never store context in a struct.** Pass it as the first argument to every function that needs it.
- `context.WithValue` uses interface key comparison — always use an unexported custom type as the key to prevent collision with other packages using the same string.
- A cancelled context passed to a DB query will cause the query to fail even if the DB has already done the work — the result is discarded. This can leave dangling DB-side operations. Be aware in write paths.

### Real-World Use

- HTTP request lifecycle
- DB query timeout
- Microservice communication
- Worker shutdown

# 2. Reflection
---
Reflection lets you **inspect and manipulate types and values at runtime** without knowing them at compile time. The entry point is the `reflect` package, but the concepts matter regardless.

**Two core types: `reflect.Type` and `reflect.Value`:**
```go
var x float64 = 3.14

t := reflect.TypeOf(x)    // reflect.Type  — describes the type
v := reflect.ValueOf(x)   // reflect.Value — holds the actual value

fmt.Println(t.Kind())     // float64
fmt.Println(v.Float())    // 3.14
```

**Kind vs Type:**
```go
type MyInt int

var m MyInt = 5
t := reflect.TypeOf(m)
fmt.Println(t.Name())   // "MyInt"  — the named type
fmt.Println(t.Kind())   // "int"    — the underlying kind
```

Kind is the underlying primitive category. Type is the declared name.

**Modifying values — requires pointer + Elem():**

```go
x := 42
v := reflect.ValueOf(x)
v.SetInt(99)  // PANICS — v is not addressable

// Correct way: pass pointer, call Elem()
v = reflect.ValueOf(&x).Elem()
v.SetInt(99)  // works — x is now 99
```

**Struct reflection — the most common use case:**

```go
type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    age   int    // unexported
}

u := User{Name: "Alice", Email: "a@b.com"}
t := reflect.TypeOf(u)
v := reflect.ValueOf(u)

for i := 0; i < t.NumField(); i++ {
    field := t.Field(i)             // reflect.StructField
    value := v.Field(i)            // reflect.Value

    fmt.Println(field.Name)        // "Name", "Email", "age"
    fmt.Println(field.Tag.Get("json"))  // "name", "email", ""
    fmt.Println(field.IsExported()) // true, true, false
    fmt.Println(value.Interface())  // panics on unexported field!
}
```

**Calling methods dynamically:**
```go
type Greeter struct{}
func (g Greeter) Hello(name string) string { return "Hello, " + name }

g := Greeter{}
v := reflect.ValueOf(g)
method := v.MethodByName("Hello")
result := method.Call([]reflect.Value{reflect.ValueOf("Alice")})
fmt.Println(result[0].String())  // "Hello, Alice"
```

**When reflection is appropriate:**

- Writing generic serializers/deserializers (like `encoding/json`).
- ORMs mapping structs to DB columns via struct tags.
- Dependency injection containers.
- Test helpers that need to compare arbitrary structs.

**When it is NOT appropriate:**

- Anywhere performance matters — reflection is 10–100x slower than direct calls.
- When generics can do the job (Go 1.18+) — prefer generics for type-safe abstractions.
- When it makes code hard to reason about — reflection errors are runtime panics, not compile errors.

**Gotchas:**

- `reflect.Value.Interface()` panics on unexported fields.
- `CanSet()` returns false if the value isn't addressable — always check before `Set*()`.
- `reflect.DeepEqual` is often the only reason people reach for reflection in tests — it works but is slow and has surprising behavior with nil slices vs empty slices.

# 3. Unsafe Package (Conceptual)
---
`unsafe` breaks Go's type safety. The compiler gives you raw memory access. There is no GC tracking of unsafe pointers — you are responsible.
### 1. The Pointer Bridge: `unsafe.Pointer`

In standard Go, you cannot convert a `*int` to a `*float64`. The compiler stops you because they are logically different things. `unsafe.Pointer` acts as a **Universal Translator**. It tells the compiler: _"Stop looking at the type, just give me the memory address."_

- **Standard Pointer:** Bound by type rules.
    
- **`unsafe.Pointer`:** A raw address that the Go Garbage Collector (GC) still recognizes.
    
- **`uintptr`:** A raw number (address) that the GC **ignores**.

### 2. The Three Pillars of Memory Geometry
| **Operation**       | **Purpose**                               | **Analogy**                                                      |
| ------------------- | ----------------------------------------- | ---------------------------------------------------------------- |
| **`Sizeof(x)`**     | How many bytes does this take?            | How much floor space does the furniture occupy?                  |
| **`Alignof(x)`**    | What "grid" does this live on?            | Does the furniture need to be pushed against a wall or centered? |
| **`Offsetof(s.f)`** | Where is field `f` relative to the start? | How many steps from the front door to the kitchen?               |
##### Example: Struct Navigation

If you have a struct, Go places the fields in a sequence in memory. By knowing the **Offset**, you can "teleport" a pointer directly to a specific field without using its name.

### 3. The Dangerous Game: Pointer Arithmetic

Go does not allow you to do `ptr + 1`. This is a safety feature to prevent you from wandering into memory you don't own. To do math on an address, you must follow this specific sequence:

1. **Convert** `unsafe.Pointer` (address) $\rightarrow$ `uintptr` (number).
    
2. **Add/Subtract** bytes to the number.
    
3. **Convert** `uintptr` (number) $\rightarrow$ back to `unsafe.Pointer` (address).
    

**The Golden Rule:** You must perform these steps in **one single expression**.

> **Why?** If you store a `uintptr` in a variable, the Garbage Collector might move the data or reclaim the memory because it doesn't think anyone is "using" it. A `uintptr` is just a number to the GC, not a reference.'

### 4. Real-World Use Case: Zero-Copy

The most common "legit" use of `unsafe` today is converting between `string` and `[]byte` without copying the underlying data.

Normally:

- `string(bytes)` creates a **copy** (to ensure the string remains immutable).
    
- `[]byte(str)` creates a **copy** (so you don't accidentally mutate the string).
    

With `unsafe`, you simply point a new header at the old memory. This is 100% faster for large buffers because 0 bytes are copied.

### 5. Summary of Risks
Using `unsafe` is like handling a live wire:

- **No Bounds Checks:** You can read past the end of an array and crash the program (Segfault).
    
- **Portability Issues:** Code that works on a 64-bit system might break on 32-bit due to different alignment/sizes.
    
- **Future-Proofing:** Code using `//go:linkname` or internal runtime tricks can (and will) break when you update your Go version.

# 4. Custom Errors
---
Go treats errors as **values**, not exceptions. It implement the `error` interface.

```go
type error interface {
    Error() string
}
```

**Sentinel errors — package-level values:**
```go
var ErrNotFound = errors.New("not found")
var ErrUnauthorized = errors.New("unauthorized")

// Usage
if err == ErrNotFound { ... }         // old style — fragile with wrapping
if errors.Is(err, ErrNotFound) { ... } // correct — unwraps chain
```

**Custom error type — carries structured data:**
```go
type AppError struct {
    Code    int
    Message string
    Err     error  // wrapped cause
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("[%d] %s: %v", e.Code, e.Message, e.Err)
    }
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}

// Implement Unwrap so errors.Is/As can traverse the chain
func (e *AppError) Unwrap() error { return e.Err }

// Constructor
func NewAppError(code int, msg string, cause error) *AppError {
    return &AppError{Code: code, Message: msg, Err: cause}
}

```

**Domain-specific error types:**

```go
type ValidationError struct {
    Field   string
    Message string
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

type DBError struct {
    Query string
    Err   error
}
func (e *DBError) Error() string { return fmt.Sprintf("db error in %q: %v", e.Query, e.Err) }
func (e *DBError) Unwrap() error { return e.Err }
```

**Returning errors — interface vs concrete type:**
```go
// WRONG — typed nil trap
func find() *AppError {
    if ok { return nil }  // returns (*AppError)(nil)
    // caller checks: err != nil → TRUE even though value is nil
}

// CORRECT — return the interface
func find() error {
    if ok { return nil }  // returns nil interface — safe
}
```

## Why Use Custom Errors

- structured error handling
- better debugging
- domain-specific logic

# 5. Wrapping / Unwrapping Errors
---
Go 1.13 introduced the error chain. The idea: errors wrap each other like a linked list. You can inspect the whole chain without losing context at each layer.
##### Problem
You lose context if you just return errors.

**Wrapping with `%w`:**
```go
if err := db.Query(q); err != nil {
    return fmt.Errorf("userRepo.Find: %w", err)
}
// Message: "userRepo.Find: connection refused"
// Chain:   *fmt.wrapError → original db error
```

**`errors.Is` — checks identity across the chain:**
```go
var ErrNotFound = errors.New("not found")

err := fmt.Errorf("service layer: %w", fmt.Errorf("repo layer: %w", ErrNotFound))

errors.Is(err, ErrNotFound)  // true — unwraps the whole chain
err == ErrNotFound           // false — only checks top level
```

`errors.Is` calls `Unwrap()` repeatedly until it finds a match or reaches nil. For custom equality, implement `Is(target error) bool` on your type:
```go
func (e *AppError) Is(target error) bool {
    t, ok := target.(*AppError)
    if !ok { return false }
    return e.Code == t.Code  // match by code, not pointer
}

errors.Is(err, &AppError{Code: 404})  // true if any error in chain has Code 404
```

**`errors.As` — extracts concrete type from chain:**
```go
var dbErr *DBError
if errors.As(err, &dbErr) {
    log.Printf("query was: %s", dbErr.Query)
    // retry logic, metrics, etc.
}
```

`errors.As` walks the chain calling `Unwrap()`, checks if each error can be assigned to the target type, and if so, sets the pointer and returns true.

**Multiple unwrap — Go 1.20 `errors.Join`:**
```go
err1 := errors.New("validation failed")
err2 := errors.New("db write failed")

combined := errors.Join(err1, err2)
// combined.Unwrap() returns []error — both are in the chain
errors.Is(combined, err1)  // true
errors.Is(combined, err2)  // true
```

**The unwrap chain visualized:**
```
fmt.Errorf("handler: %w", 
    fmt.Errorf("service: %w", 
        fmt.Errorf("repo: %w", 
            ErrNotFound)))

errors.Is(err, ErrNotFound):
  → unwrap handler layer  → "service: ..."
  → unwrap service layer  → "repo: ..."
  → unwrap repo layer     → ErrNotFound ✓ match
```

**Convention — always add call-site context when wrapping:**
```go
// Good — tells you exactly where in the call stack it failed
return fmt.Errorf("UserService.GetByID id=%d: %w", id, err)

// Bad — loses call site info
return err
```

# 6. Panic / Recover
---
In Go, `panic` and `recover` are the mechanism for handling "exceptional" errors—those unexpected, unrecoverable situations that shouldn't happen during normal program execution (like an index out of bounds or a nil pointer dereference).

Think of a **Panic** as a fire alarm that evacuates the building, and **Recover** as the fire suppression system that stops the building from burning down so people can go back inside.

#### 1. The Mechanics of Panic

When a function calls `panic`, the following happens:

1. The current function **stops executing** immediately.
    
2. Any functions called via the **`defer`** keyword are executed in Last-In-First-Out (LIFO) order.
    
3. The panic propagates up the call stack, running defers in every parent function.
    
4. If not caught, the program crashes and prints a stack trace.

## 2. The Mechanics of Recover

`recover` is a built-in function that regains control of a panicking goroutine.

- It is **only useful inside `defer` functions**.
    
- Calling `recover` inside a normal function does nothing and returns `nil`.
    
- If the current goroutine is panicking, `recover` will capture the value passed to `panic` and stop the "evacuation," allowing the program to continue from the point where the deferred function finished.

## 3. Practical Example: Protecting Your Program

Imagine a web server where one specific request handler crashes. You don't want the entire server to go down; you just want to log the error and keep serving other users.

```go
package main

import "fmt"

func safeDivision(a, b int) {
    // 1. Set up the "safety net"
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Recovered from error: %v\n", r)
        }
    }()

    // 2. This will trigger a runtime panic if b == 0
    fmt.Printf("Result: %d\n", a/b)
}

func main() {
    fmt.Println("Starting calculation...")
    
    safeDivision(10, 2) // Normal execution
    safeDivision(10, 0) // This would normally crash the program
    
    fmt.Println("Program finished gracefully!")
}
```
### Why this works:

- When `a/b` causes a division-by-zero, the `safeDivision` function panics.
    
- The `defer` block executes.
    
- Inside `defer`, `recover()` catches the panic. The program does **not** exit main.

##### **Another Example:**
```go
// 1. HTTP server — recover from handler panics, return 500 instead of crashing
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic recovered: %v\n%s", err, debug.Stack())
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// 2. Goroutine supervisor — panics in goroutines crash the whole program
func safeGo(f func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("goroutine panic: %v", r)
            }
        }()
        f()
    }()
}
```

**When to use panic vs error:**

| Situation                                          | Use                                          |
| -------------------------------------------------- | -------------------------------------------- |
| Expected failure (file not found, bad input)       | `error`                                      |
| Programming mistake (nil passed where not allowed) | `panic`                                      |
| Unrecoverable state (corrupt data structure)       | `panic`                                      |
| Library initialization failure                     | `panic` (in `init()` or constructors)        |
| Across API boundaries                              | Always convert back to `error` via `recover` |
**The `Must` pattern — panic in constructors:**
```go
// Common in stdlib: regexp.MustCompile, template.Must
func MustParse(s string) *Config {
    cfg, err := Parse(s)
    if err != nil {
        panic(fmt.Sprintf("MustParse: %v", err))
    }
    return cfg
}

// Used at package init time — panic is acceptable
var defaultConfig = MustParse(os.Getenv("CONFIG"))
```

**Gotchas:**
- A panic in a goroutine that's not recovered crashes the **entire program**, not just that goroutine.
- `recover()` only catches the panic in the current goroutine — you can't recover a panic from another goroutine.
- `debug.Stack()` inside a recover gives you the full stack trace for logging.

#### 💥 Important Insight

> Panic is for _bugs_, not _failures_


# 7. Functional Options Pattern
---
The problem: constructors with many optional parameters are painful in Go (no default arguments, no overloading).

**The naive solutions that don't scale:**
```go
// Config struct — caller must know zero values, easy to get wrong
NewServer(ServerConfig{Port: 8080})

// Telescoping constructors — combinatorial explosion
NewServer(addr string)
NewServerWithTimeout(addr string, timeout time.Duration)
NewServerWithTimeoutAndTLS(addr string, timeout time.Duration, cert tls.Certificate)
```

**Functional options — the idiomatic solution:**
```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
    maxConn int
    tls     *tls.Config
}

// Option is a function that mutates the server config
type Option func(*Server)

// Each option is a constructor function
func WithHost(host string) Option {
    return func(s *Server) { s.host = host }
}

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func WithTLS(cfg *tls.Config) Option {
    return func(s *Server) { s.tls = cfg }
}

// Constructor with sensible defaults
func NewServer(opts ...Option) *Server {
    s := &Server{           // defaults
        host:    "localhost",
        port:    8080,
        timeout: 30 * time.Second,
        maxConn: 100,
    }
    for _, opt := range opts {
        opt(s)              // apply each option
    }
    return s
}

// Usage — clean, readable, forward-compatible
srv := NewServer(
    WithPort(9090),
    WithTimeout(10 * time.Second),
    WithTLS(tlsCfg),
)
```

**Adding validation inside options:**
```go
func WithPort(port int) Option {
    return func(s *Server) {
        if port < 1 || port > 65535 {
            panic(fmt.Sprintf("invalid port: %d", port))
        }
        s.port = port
    }
}
```

**Error-returning variant — for when panicking isn't appropriate:**
```go
type Option func(*Server) error

func WithPort(port int) Option {
    return func(s *Server) error {
        if port < 1 || port > 65535 {
            return fmt.Errorf("invalid port: %d", port)
        }
        s.port = port
        return nil
    }
}

func NewServer(opts ...Option) (*Server, error) {
    s := &Server{host: "localhost", port: 8080}
    for _, opt := range opts {
        if err := opt(s); err != nil {
            return nil, fmt.Errorf("NewServer: %w", err)
        }
    }
    return s, nil
}
```

**Why this pattern is powerful:**

- **Backwards compatible** — add new options without breaking existing call sites.
- **Readable** — each option is self-documenting at the call site.
- **Composable** — options are just functions, you can combine, wrap, or conditionally apply them.
- **Testable** — you can build test-specific option sets.

```go
// Conditional option application
opts := []Option{WithPort(8080)}
if cfg.TLSEnabled {
    opts = append(opts, WithTLS(cfg.TLS))
}
srv := NewServer(opts...)
```

### Summary

|Topic|Core Idea|
|---|---|
|Context|Cancellation tree. Pass as first arg. Use unexported key types for values. Always defer cancel().|
|Reflection|Runtime type/value inspection. Slow. Use for frameworks/serializers. Prefer generics for new code.|
|Unsafe|Raw memory access. `unsafe.Pointer` bridges types. Never store `uintptr` as a variable.|
|Custom errors|Implement `error`. Add `Unwrap()` for chain. Return `error` interface, never concrete nil.|
|Wrapping|`%w` wraps. `errors.Is` checks identity. `errors.As` extracts type. Always add context on wrap.|
|Panic/Recover|Panic for programming errors. Recover in HTTP middleware and goroutine supervisors. Convert to error at API boundaries.|
|Functional options|`type Option func(*T)`. Defaults in constructor. Apply with a loop. Backwards-compatible and composable.|

| Concept            | Problem Solved         |
| ------------------ | ---------------------- |
| Context            | lifecycle management   |
| Reflection         | dynamic behavior       |
| Unsafe             | performance edge cases |
| Custom errors      | structured failure     |
| Error wrapping     | debugging & tracing    |
| Panic/recover      | crash handling         |
| Functional options | clean API design       |
