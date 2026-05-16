# Advanced Go Flashcards — Core Revision

## 1. Context

### Flashcard 1

**Q:** What problem does Go's `context` package solve?

**A:** It solves:

- lifecycle management
    
- cancellation propagation
    
- deadlines/timeouts
    
- request-scoped metadata passing across goroutines and API boundaries.
    

It creates a cancellation tree where parent cancellation propagates to children.

---

### Flashcard 2

**Q:** What are the four main methods of the `Context` interface?

**A:**

```go
Deadline() (time.Time, bool)
Done() <-chan struct{}
Err() error
Value(key any) any
```

---

### Flashcard 3

**Q:** Difference between `context.Background()` and `context.TODO()`?

**A:**

- `Background()` → proper root context
    
- `TODO()` → temporary placeholder when unsure which context to use
    

Internally they behave the same.

---

### Flashcard 4

**Q:** Why should you always call `cancel()`?

**A:** To release internal resources/timers even if timeout expires naturally.

```go
ctx, cancel := context.WithTimeout(...)
defer cancel()
```

---

### Flashcard 5

**Q:** How does cancellation propagate in context trees?

**A:**

- Parent cancellation cancels all descendants.
    
- Child cancellation NEVER cancels parent.
    

---

### Flashcard 6

**Q:** What does `ctx.Done()` return?

**A:** A receive-only channel closed when context is cancelled.

Used inside `select`.

```go
select {
case <-ctx.Done():
    return ctx.Err()
}
```

---

### Flashcard 7

**Q:** What are the two standard values returned by `ctx.Err()`?

**A:**

```go
context.Canceled
context.DeadlineExceeded
```

---

### Flashcard 8

**Q:** Why should context keys use custom unexported types?

**A:** To avoid collisions across packages.

```go
type contextKey string
```

---

### Flashcard 9

**Q:** What should NOT be stored in context?

**A:**

- DB connections
    
- loggers
    
- optional params
    
- business/domain data
    

Only request-scoped cross-cutting metadata belongs there.

---

### Flashcard 10

**Q:** Why should context never be stored in structs?

**A:** Context is request-scoped and short-lived. Storing it in structs causes lifecycle confusion and stale contexts.

Always pass it explicitly as first parameter.

---

# 🚨 Trap Flashcards — Context

### Trap 1

**Q:** Can child contexts cancel parent contexts?

**A:** ❌ No.

Cancellation only propagates downward.

---

### Trap 2

**Q:** Is `context.WithValue` a good replacement for optional parameters?

**A:** ❌ No.

This is considered misuse.

---

### Trap 3

**Q:** If timeout expires naturally, is calling `cancel()` unnecessary?

**A:** ❌ Wrong.

Always call `cancel()` to free resources.

---

### Trap 4

**Q:** Can you safely use string keys in context values?

**A:** ❌ Dangerous.

Different packages may collide on the same key string.

Use custom unexported types.

---

---

# 2. Reflection

### Flashcard 11

**Q:** What are the two central reflection types?

**A:**

```go
reflect.Type
reflect.Value
```

- `Type` → metadata/type information
    
- `Value` → runtime value container
    

---

### Flashcard 12

**Q:** Difference between `Kind()` and `Name()`?

**A:**

- `Kind()` → underlying primitive category
    
- `Name()` → declared named type
    

Example:

```go
type MyInt int
```

- Name → `MyInt`
    
- Kind → `int`
    

---

### Flashcard 13

**Q:** Why does `reflect.Value.SetInt()` often panic?

**A:** Because the value is not addressable/settable.

Must pass pointer + use `.Elem()`.

```go
reflect.ValueOf(&x).Elem()
```

---

### Flashcard 14

**Q:** What does `CanSet()` check?

**A:** Whether reflection can mutate the value.

---

### Flashcard 15

**Q:** Why can reflection hurt performance?

**A:** Reflection:

- bypasses compile-time optimizations
    
- uses interface boxing/unboxing
    
- performs runtime metadata inspection
    

Usually 10–100x slower than direct code.

---

### Flashcard 16

**Q:** Common real-world uses of reflection?

**A:**

- ORMs
    
- serializers
    
- dependency injection
    
- testing utilities
    
- frameworks
    

---

### Flashcard 17

**Q:** Why are generics preferred over reflection in modern Go?

**A:** Generics provide:

- compile-time type safety
    
- better performance
    
- cleaner APIs
    
- no runtime panics
    

---

# 🚨 Trap Flashcards — Reflection

### Trap 5

**Q:** Can reflection access unexported struct fields safely?

**A:** ❌ Not fully.

`Interface()` on unexported fields can panic.

---

### Trap 6

**Q:** Does `reflect.DeepEqual(nilSlice, emptySlice)` return true?

**A:** ❌ No.

```go
nil != []T{}
```

This surprises many developers.

---

### Trap 7

**Q:** Does reflection operate at compile time?

**A:** ❌ No.

Reflection is purely runtime behavior.

---

---

# 3. Unsafe Package

### Flashcard 18

**Q:** What is `unsafe.Pointer`?

**A:** A universal pointer bridge allowing conversion between unrelated pointer types.

---

### Flashcard 19

**Q:** Difference between `unsafe.Pointer` and `uintptr`?

**A:**

- `unsafe.Pointer` → GC-tracked pointer
    
- `uintptr` → raw integer address NOT tracked by GC
    

---

### Flashcard 20

**Q:** What does `unsafe.Sizeof()` do?

**A:** Returns number of bytes occupied.

---

### Flashcard 21

**Q:** What does `unsafe.Offsetof()` do?

**A:** Returns byte offset of a struct field relative to struct start.

---

### Flashcard 22

**Q:** Why must pointer arithmetic happen in one expression?

**A:** Because storing `uintptr` separately can let GC move/reclaim memory.

---

### Flashcard 23

**Q:** What is a common legitimate use of `unsafe`?

**A:** Zero-copy conversion between:

```go
string <-> []byte
```

---

# 🚨 Trap Flashcards — Unsafe

### Trap 8

**Q:** Does Go support normal pointer arithmetic?

**A:** ❌ No.

You must manually convert:

```go
Pointer -> uintptr -> Pointer
```

---

### Trap 9

**Q:** Is `uintptr` considered a pointer by the GC?

**A:** ❌ No.

GC treats it as just an integer.

---

### Trap 10

**Q:** Is unsafe code portable across architectures?

**A:** ❌ Not guaranteed.

Alignment/size differ on 32-bit vs 64-bit systems.

---

---

# 4. Custom Errors

### Flashcard 24

**Q:** What philosophy does Go use for error handling?

**A:** Errors are values, not exceptions.

---

### Flashcard 25

**Q:** What is a sentinel error?

**A:** A package-level reusable error value.

```go
var ErrNotFound = errors.New("not found")
```

---

### Flashcard 26

**Q:** Why is `errors.Is()` preferred over `==`?

**A:** It unwraps the entire error chain.

---

### Flashcard 27

**Q:** Why implement `Unwrap()`?

**A:** So `errors.Is()` and `errors.As()` can traverse wrapped errors.

---

### Flashcard 28

**Q:** What is the typed nil trap?

**A:**

```go
var err *AppError = nil
return err
```

This becomes:

```go
(error)(*AppError(nil))
```

Which is NOT nil.

---

### Flashcard 29

**Q:** Best practice for return types in Go error APIs?

**A:** Return `error` interface, not concrete error types.

---

# 🚨 Trap Flashcards — Errors

### Trap 11

**Q:** Does `err == ErrNotFound` work reliably with wrapped errors?

**A:** ❌ No.

Use:

```go
errors.Is(err, ErrNotFound)
```

---

### Trap 12

**Q:** Can a nil concrete pointer inside an interface still make `err != nil`?

**A:** ✅ Yes.

Classic Go trap.

---

---

# 5. Wrapping & Unwrapping

### Flashcard 30

**Q:** What does `%w` do in `fmt.Errorf`?

**A:** Wraps an error into a new error layer.

---

### Flashcard 31

**Q:** What does `errors.As()` do?

**A:** Extracts concrete error types from an error chain.

---

### Flashcard 32

**Q:** What does `errors.Join()` do?

**A:** Combines multiple errors into one chain.

---

### Flashcard 33

**Q:** Why should wrapped errors include call-site context?

**A:** To preserve debugging trace information.

Example:

```go
return fmt.Errorf("UserService.GetByID: %w", err)
```

---

# 🚨 Trap Flashcards — Wrapping

### Trap 13

**Q:** Does wrapping preserve pointer equality?

**A:** ❌ No.

Wrapped errors are different objects.

---

### Trap 14

**Q:** Does `errors.As()` compare by identity?

**A:** ❌ No.

It checks assignability/type compatibility.

---

---

# 6. Panic / Recover

### Flashcard 34

**Q:** What happens during panic propagation?

**A:**

1. Current function stops
    
2. Deferred functions execute
    
3. Panic climbs stack
    
4. Program crashes if unrecovered
    

---

### Flashcard 35

**Q:** Where does `recover()` work?

**A:** ONLY inside deferred functions.

---

### Flashcard 36

**Q:** What is panic mainly meant for?

**A:** Programmer bugs/unrecoverable states.

NOT normal business failures.

---

### Flashcard 37

**Q:** Why are recovery middlewares important in servers?

**A:** Prevent one handler panic from crashing entire server.

---

### Flashcard 38

**Q:** Why is panic in goroutines dangerous?

**A:** Unrecovered panic in ANY goroutine crashes entire program.

---

### Flashcard 39

**Q:** What does `debug.Stack()` provide?

**A:** Full stack trace for panic logging/debugging.

---

# 🚨 Trap Flashcards — Panic

### Trap 15

**Q:** Can one goroutine recover panic from another goroutine?

**A:** ❌ No.

Recover only works within same goroutine.

---

### Trap 16

**Q:** Should panic be used for validation/business logic errors?

**A:** ❌ Never.

Use normal `error`.

---

### Trap 17

**Q:** Does `recover()` work outside deferred functions?

**A:** ❌ No.

Returns nil.

---

---

# 7. Functional Options Pattern

### Flashcard 40

**Q:** What problem does functional options solve?

**A:** Constructor explosion and optional parameter management.

---

### Flashcard 41

**Q:** Canonical functional option type?

**A:**

```go
type Option func(*Server)
```

---

### Flashcard 42

**Q:** Why are functional options backward-compatible?

**A:** New options can be added without changing constructor signature.

---

### Flashcard 43

**Q:** Why are functional options composable?

**A:** Because options are just functions.

---

### Flashcard 44

**Q:** What is the error-returning functional option variant?

**A:**

```go
type Option func(*Server) error
```

Useful when validation should not panic.

---

# 🚨 Trap Flashcards — Functional Options

### Trap 18

**Q:** Are functional options mainly about performance?

**A:** ❌ No.

They're about API ergonomics and extensibility.

---

### Trap 19

**Q:** Should defaults be applied inside options?

**A:** ❌ Usually defaults belong in constructor.

---

---

# 8. Closures & Higher-Order Functions

### Flashcard 45

**Q:** What is a closure?

**A:** A function that captures variables from surrounding lexical scope.

---

### Flashcard 46

**Q:** Do closures capture variables by value or by reference?

**A:** By reference.

This causes the classic loop-variable bug.

---

### Flashcard 47

**Q:** Why do captured variables often escape to heap?

**A:** Because closures may outlive their original stack frame.

---

### Flashcard 48

**Q:** What is a higher-order function (HOF)?

**A:** A function that:

- accepts functions
    
- returns functions
    
- or both
    

---

### Flashcard 49

**Q:** What are the classic functional trio patterns?

**A:**

- Map
    
- Filter
    
- Reduce
    

---

### Flashcard 50

**Q:** What is memoization?

**A:** Caching expensive computation results inside closures.

---

### Flashcard 51

**Q:** Why are closures powerful for middleware?

**A:** They encapsulate reusable behavior/state cleanly.

---

### Flashcard 52

**Q:** Why are closures useful for lazy evaluation?

**A:** They defer expensive computation until first use.

---

### Flashcard 53

**Q:** What causes the classic goroutine closure bug?

**A:** Goroutines capture shared loop variables.

---

### Flashcard 54

**Q:** How do you fix loop-variable closure bugs?

**A:**  
Either:

```go
i := i
```

Or pass as parameter:

```go
go func(v int) {}(i)
```

---

### Flashcard 55

**Q:** What is escape analysis?

**A:** Compiler analysis deciding stack vs heap allocation.

Use:

```bash
go build -gcflags="-m"
```

---

# 🚨 Trap Flashcards — Closures & HOF

### Trap 20

**Q:** Does each closure in a loop automatically get its own variable copy?

**A:** ❌ Not before Go 1.22.

All closures shared same loop variable.

---

### Trap 21

**Q:** Can closures increase GC pressure?

**A:** ✅ Yes.

Captured variables escaping to heap increase allocations.

---

### Trap 22

**Q:** Are closures always slower?

**A:** ❌ Not necessarily.

But heap escape can add overhead.

---

### Trap 23

**Q:** Is middleware primarily an OOP pattern?

**A:** ❌ No.

Middleware in Go is primarily functional/HOF-based.

---

# 🔥 Ultra-Important Interview Traps

### Trap 24

**Q:** Why is this dangerous?

```go
go func() {
    fmt.Println(i)
}()
```

inside loops?

**A:** Closure captures SAME loop variable shared across iterations.

---

### Trap 25

**Q:** Why is this dangerous?

```go
return (*MyError)(nil)
```

when return type is `error`?

**A:** Interface becomes non-nil due to type metadata.

---

### Trap 26

**Q:** Why is storing `uintptr` risky?

**A:** GC cannot track it as pointer reference.

---

### Trap 27

**Q:** Why is reflection avoided in performance-critical paths?

**A:** Runtime metadata lookups + boxing/unboxing + inability to optimize.

---

### Trap 28

**Q:** Why is `panic` discouraged for business failures?

**A:** Panic represents programmer bugs/unrecoverable corruption, not expected failures.

---

### Trap 29

**Q:** Why is `context.Context` always first parameter?

**A:** Standard Go convention for lifecycle propagation visibility.

---

### Trap 30

**Q:** Why can cancelled contexts still leave DB work running?

**A:** Cancellation discards result client-side; DB may still continue execution server-side.