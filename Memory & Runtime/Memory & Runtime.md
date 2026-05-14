# 1. Stack vs Heap
---
Go manages two regions of memory:

**Stack** — per-goroutine, grows and shrinks automatically. Allocation is essentially free (CPU just move the stack pointer and  It is also extremely cache-friendly.). Each goroutine starts with a small stack (~8KB) and grows as needed via stack copying (more on that in runtime internals).
		- No GC involvement
	
**Heap** — shared, GC-managed. Allocation is more expensive and creates GC pressure(==performance issue where an application creates objects faster than the garbage collector (GC) can reclaim memory, causing frequent GC cycles, high CPU usage, and application pauses==).

```go
func add(a, b int) int {
    result := a + b   // 'result' lives on the stack
    return result     // copied out on return, stack frame released
}

func newUser() *User {
    u := User{Name: "Alice"}  // escapes to heap — pointer returned
    return &u
}
```

The compiler decides where a variable lives through **escape analysis**. You don’t control stack vs heap. The rule of thumb:

- If a variable's lifetime is bounded to the function call → stack.
- If it outlives the function (returned pointer, stored in heap object, captured by goroutine, etc.) → heap.

**Goroutine stacks are not OS threads.** They start small and are copied when they grow — this is why you can have millions of goroutines cheaply.

| Stack        | Heap              |
| ------------ | ----------------- |
| Fast         | Slower            |
| No GC        | GC overhead       |
| Limited size | Large allocations |
👉 Excess heap usage = **performance degradation**

# 2. Escape Analysis
---
Compiler decides:

> “Can this variable safely live on stack, or must it escape to heap?”

 You can inspect decisions with:
```bash
go build -gcflags='-m' ./...
# More verbose:
go build -gcflags='-m -m' ./...
```

Output looks like:

```
./main.go:10:6: moved to heap: u
./main.go:15:2: result does not escape
```

**Common escape triggers:**
```go
// 1. Returning a pointer to a local variable
func f() *int {
    x := 42
    return &x   // x escapes — compiler moves to heap
}

// 2. Storing in an interface
func g(v interface{}) {
    // value inside interface{} is heap-allocated
}
var x int = 42
g(x)   // x is boxed — escapes

// 3. Captured by a goroutine or closure
func h() {
    x := 10
    go func() {
        fmt.Println(x)  // x escapes — goroutine outlives h()
    }()
}

// 4. Stored in a heap-allocated struct
type Node struct{ val *int }
func i() {
    x := 42
    n := &Node{val: &x}  // x escapes — referenced by heap object
    _ = n
}

// 5. Slice/map allocation
s := make([]int, 0, 1000)  // backing array goes to heap
```

**What doesn't escape:**
```go
func sum(nums []int) int {
    total := 0              // stays on stack
    for _, n := range nums {
        total += n
    }
    return total
}
```

**Why this matters for performance:** every heap allocation costs time (malloc + GC tracking). Tight loops that allocate inside them → GC pressure → stop-the-world pauses (even if tiny). Write benchmarks and use `go test -benchmem` to see allocations per operation.
```bash
go test -bench=. -benchmem
# BenchmarkFoo-8   1000000   1203 ns/op   48 B/op   1 allocs/op
```

## 🚀 Real-World Impact

- Returning pointers → heap
- Interface usage → often heap
- Closures → often heap

👉 Overuse = hidden performance cost

# 3. Garbage Collector (GC)
---
Go uses a **concurrent tri-color mark-sweep GC** that runs mostly alongside your program (not stop-the-world like Java's old GC).

**Tri-color algorithm:**

| Color | Meaning                                          |
| ----- | ------------------------------------------------ |
| White | Not yet visited — candidate for collection       |
| Grey  | Discovered, but children not yet scanned         |
| Black | Fully scanned — reachable, will not be collected |
The cycle:

1. **Mark roots** (globals, stack vars) as grey.
2. **Scan grey objects** — mark their references grey, mark themselves black.
3. **Repeat** until no grey objects remain.
4. **Sweep** — everything still white is garbage, reclaimed.

#### **Write barrier** —
---
###### **The Problem: Concurrent Marking**

Go uses a **Concurrent Mark-and-Sweep** GC. This means the GC is walking through your memory to find "live" objects (the **Mark phase**) while your program is still running and changing pointers.

Imagine this scenario:

1. The GC scans **Object A** and sees it points to **Object B**.
2. Your program quickly moves the pointer so **Object A** now points to a new **Object C**, and **Object B** is removed.
3. Because the GC already "passed" Object A, it might miss Object C entirely and think it's garbage.

##### **The Solution: The Write Barrier**

When the GC is in its "Marking" phase, it turns on the **Write Barrier**. It's essentially a small piece of "guard code" that sits in front of every pointer update.

Whenever your code does: `object.ptr = newPointer`, the write barrier stops it for a nanosecond and says:  
_"Wait! The GC is running. I need to mark `newPointer` as 'reachable' right now just in case the GC hasn't seen it yet."_

> This is a small overhead on every pointer write during GC.

**GC phases in practice:**

```
STW: mark setup   → very short (~100µs)
Concurrent mark   → runs alongside your code
STW: mark term    → very short (~100µs)
Concurrent sweep  → background, amortized
```

The two STW (stop-the-world) pauses are typically sub-millisecond in modern Go.

## 🔹 Tuning

```bash
GOGC=100
```

- Higher → less frequent GC
- Lower → more frequent GC
### GOMEMLIMIT (Go 1.19+)

`GOGC` controls GC _frequency_ (relative to live heap size). But it can't prevent OOM(Out Of Memory) — if your heap grows faster than GC can reclaim, the process dies.

`GOMEMLIMIT` sets a **hard ceiling** on total Go memory usage. GC will run aggressively to stay under it.
```bash
GOMEMLIMIT=512MiB go run main.go
```
Or in code:
```go
import "runtime/debug"
debug.SetMemoryLimit(512 << 20)  // 512 MB
```

**Why it matters:**

- Before 1.19: you'd set `GOGC=off` for throughput, but risk OOM in bursty traffic
- Now: set `GOGC=off` + `GOMEMLIMIT=<safe ceiling>` — GC only kicks in when actually needed, not on a fixed heap-growth schedule

**Practical rule:**

- Set `GOMEMLIMIT` to ~90% of your container's memory limit
- Prevents the container from being OOM-killed by the OS/Kubernetes before Go's GC has a chance to react

```bash
# container has 1GB RAM
GOMEMLIMIT=900MiB GOGC=off ./server
```

## 🔥 Pro Tip

Reduce GC pressure by:

- avoiding unnecessary allocations
- reusing objects (sync.Pool)
- preferring stack allocations

**Observing GC:**

``` bash
GODEBUG=gctrace=1 go run main.go
# gc 1 @0.012s 3%: 0.035+1.2+0.051 ms clock, ...
```

### sync.Pool and GC Interaction

The critical behavior most people miss:

> **sync.Pool objects are CLEARED on every GC cycle.**

Pool is not a cache. It's a way to reuse allocations _between_ GC cycles — nothing more.
```go
var pool = sync.Pool{
    New: func() any { return &MyBuffer{} },
}

// Between GC cycles: Get() may return a previously Put() object
// After a GC: pool is drained, Get() calls New() again

obj := pool.Get().(*MyBuffer)
obj.Reset()
defer pool.Put(obj)
```

**Implications:**

- Don't store objects in Pool that are expensive to recreate — GC will destroy them regularly
- Pool is ideal for short-lived, frequently allocated objects: byte buffers, `bytes.Buffer`, request-scoped scratch space
- Don't rely on Pool for object _lifetime_ — only for allocation _reuse_

**What Pool actually solves:**

```
Without Pool: allocate → use → GC → allocate → use → GC → ...
With Pool:    allocate → use → return to pool → reuse → ... → GC drains → allocate again
```

Net effect: fewer allocations per GC cycle = less GC pressure = fewer pauses.
# 4. Value vs Pointer Receivers
---
You know the mechanics — here's the memory-focused view.

**Value receiver:**

```go
func (u User) Name() string { return u.Name }
```

- Copies the entire struct on every call.
- `User` has 3 string fields (each string = 16 bytes on 64-bit: ptr + len) → 48 bytes copied per call.
- Copy stays on the stack if it doesn't escape.

**Pointer receiver:**

```go
func (u *User) SetName(n string) { u.Name = n }
```

- Passes 8 bytes (pointer size on 64-bit) regardless of struct size.
- No copy, but introduces indirection (pointer chase).

**Decision matrix:**

| Condition                         | Use                             |
| --------------------------------- | ------------------------------- |
| Struct > ~40 bytes                | Pointer receiver                |
| Need to mutate                    | Pointer receiver                |
| Concurrency safe reads            | Value receiver (immutable copy) |
| Tiny struct (e.g. Point{X,Y int}) | Value receiver                  |
| Interface implementation          | Be careful — see method sets    |

**Interface + receiver interaction:**
```go
type Namer interface { Name() string }

type User struct{ name string }
func (u User) Name() string { return u.name }  // value receiver

var n Namer = User{"Alice"}   // OK — T satisfies interface
var n Namer = &User{"Alice"}  // OK — *T also satisfies (value receiver in both sets)

func (u *User) Name() string { ... }  // pointer receiver

var n Namer = User{"Alice"}   // COMPILE ERROR — T's method set doesn't include pointer receivers
var n Namer = &User{"Alice"}  // OK
```

# 5. Memory Alignment
---
CPU prefers data aligned in memory.

Misalignment = slower access.

The CPU accesses memory most efficiently in aligned chunks — a 64-bit value should sit at an address divisible by 8, a 32-bit value at an address divisible by 4, etc. Go does this automatically, but struct field ordering determines how much **padding** gets inserted.

```go
// Poorly ordered — 24 bytes due to padding
type Bad struct {
    A bool     // 1 byte
               // 7 bytes padding
    B int64    // 8 bytes
    C bool     // 1 byte
               // 7 bytes padding  ← to align next field
}
// Total: 1 + 7 + 8 + 1 + 7 = 24 bytes

// Optimally ordered — 16 bytes, no wasted padding
type Good struct {
    B int64    // 8 bytes
    A bool     // 1 byte
    C bool     // 1 byte
               // 6 bytes padding (struct size rounded to largest field alignment)
}
// Total: 8 + 1 + 1 + 6 = 16 bytes
```

**Rule of thumb — order fields largest to smallest:**
```go
type Optimized struct {
    // 8-byte fields first
    ID        int64
    Timestamp time.Time    // 24 bytes internally, but starts aligned

    // 4-byte fields
    Count     int32
    Flags     uint32

    // 2-byte fields
    Code      uint16

    // 1-byte fields
    Active    bool
    Deleted   bool
}
```

**Checking struct size:**
```go
import "unsafe"

fmt.Println(unsafe.Sizeof(Bad{}))    // 24
fmt.Println(unsafe.Sizeof(Good{}))   // 16
fmt.Println(unsafe.Alignof(int64(0))) // 8
fmt.Println(unsafe.Offsetof(Bad{}.B)) // 8 (after padding)
```

**When it actually matters:** if you're allocating millions of these structs (e.g. in a slice for a hot data path), shaving 8 bytes per struct = 8MB per million instances. For normal application structs it's premature optimization. For protocol headers, cache-line-sensitive structs, or high-throughput serialization paths — it matters.

**False sharing** (advanced): two goroutines writing to different fields of the same struct may contend on the same CPU cache line (64 bytes). Use `//go:noescape` or padding to separate hot fields into different cache lines.

# 6. Zero-Copy Techniques
---
A "copy" here means `memcpy` — moving bytes from one memory region to another. Zero-copy means you reuse the same memory.

**6a. `io.Reader` / `io.Writer` streaming — avoid buffering entire payload:**

```go
// BAD — reads entire response into memory
body, _ := io.ReadAll(resp.Body)
json.Unmarshal(body, &result)

// GOOD — streaming decode, no full buffer
json.NewDecoder(resp.Body).Decode(&result)
```

**6b. `bytes.Buffer` and `strings.Builder` — avoid repeated concatenation:**

```go
// BAD — each += allocates a new string (O(n²))
s := ""
for _, part := range parts {
    s += part
}

// GOOD — single allocation
var b strings.Builder
b.Grow(estimatedSize)   // pre-allocate
for _, part := range parts {
    b.WriteString(part)
}
result := b.String()
```

**6c. `sync.Pool` — reuse allocations:**

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 4096)
    },
}

func handleRequest(data []byte) {
    buf := bufPool.Get().([]byte)
    buf = buf[:0]                  // reset length, keep capacity
    defer bufPool.Put(buf)

    buf = append(buf, data...)     // no new allocation if cap sufficient
    process(buf)
}
```

**6d. `io.Pipe` — connect reader and writer without buffering:**

```go
pr, pw := io.Pipe()
go func() {
    json.NewEncoder(pw).Encode(payload)
    pw.Close()
}()
http.Post(url, "application/json", pr)  // reads directly from encoder output
```

**6e. `mmap` via `syscall` — map file directly into address space:**

```go
f, _ := os.Open("large_file.bin")
fi, _ := f.Stat()
data, _ := syscall.Mmap(int(f.Fd()), 0, int(fi.Size()),
    syscall.PROT_READ, syscall.MAP_SHARED)
// data is []byte — no file read, no copy, OS handles paging
defer syscall.Munmap(data)
```

**6f. Slice tricks — avoid copying when trimming:**

```go
// Instead of allocating a new slice, reslice in place
func trimFirst(s []int) []int {
    return s[1:]   // no copy — moves the start pointer
}
```

# 7. String ↔ Byte Conversion
---
This is a very common source of hidden allocations in Go.
#### 🔹 Key Fact

- `string` = immutable
- `[]byte` = mutable

**The fundamental issue:**

```go
s := "hello"
b := []byte(s)   // ALLOCATES — copies the string's bytes into a new slice
s2 := string(b)  // ALLOCATES — copies the slice's bytes into a new string
```

A `string` in Go is `(ptr, len)` — immutable. A `[]byte` is `(ptr, len, cap)` — mutable. The conversion copies because the memory models are incompatible (mutability guarantee).

**The compiler is smart about eliminating copies in some cases:**
```go
// No allocation — compiler knows the []byte won't outlive the comparison
m := map[string]int{"hello": 1}
key := []byte("hello")
_ = m[string(key)]   // compiler elides the copy here

// No allocation — string(b) in a concat is optimized
b := []byte("world")
_ = "hello " + string(b)   // may be optimized

// No allocation — ranging over []byte(s) avoids full copy in some cases
for i, c := range []byte(s) { ... }
```

## When to Use What

| Use Case        | Type   |
| --------------- | ------ |
| Text processing | string |
| I/O, network    | []byte |
| Mutation        | []byte |

**`strings` vs `bytes` packages:**

```go
// If you already have a []byte, operate on it directly
// instead of converting to string
bytes.Contains(data, []byte("token"))  // no alloc
bytes.Split(data, []byte("\n"))        // no alloc

// vs
strings.Contains(string(data), "token")  // allocates
```

**The unsafe zero-copy conversion (use deliberately):**

```go
import "unsafe"

func stringToBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
    // Returns a []byte pointing at the string's memory
    // DO NOT mutate the result — strings are immutable
}

func bytesToString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
    // Returns a string header over the slice's memory
    // DO NOT modify b after this — string is supposed to be immutable
}
```

**When to actually use unsafe conversions:**

- Hot path that processes millions of strings per second (e.g. log formatter, HTTP request parser).
- You control the lifetime — the original `[]byte` must not be modified and must outlive the string.
- Benchmarks confirm the allocation is the bottleneck (`go test -benchmem`).

### Goroutine Stack Growth (Contiguous Stack Model)

Each goroutine starts with a small stack (~2KB in recent Go versions, historically 8KB). It grows dynamically as needed.

**How growth works:**

1. On every function call, Go checks if there's enough stack space
2. If not, it allocates a **new larger contiguous stack** (typically 2×)
3. **Copies the entire old stack** to the new one
4. Updates all pointers that pointed into the old stack
5. Frees the old stack

**Implications:**

- Pointers into the stack are not stable — never store a pointer to a stack variable in a long-lived heap object (the GC/escape analysis prevents this, but good to understand why)
- This is why millions of goroutines are cheap — starting at 2KB vs OS thread stacks at ~1-8MB
- Stack shrinks too: if usage drops significantly, Go copies to a smaller stack

```go
// This is fine — Go handles stack growth transparently
func deepRecursion(n int) int {
    if n == 0 { return 0 }
    return deepRecursion(n - 1)  // stack grows automatically
}
```

### `runtime.ReadMemStats` — Observing Memory Programmatically
---
`GODEBUG=gctrace=1` is great for development. For production observability, use `runtime.ReadMemStats`:

```go
import "runtime"

var stats runtime.MemStats
runtime.ReadMemStats(&stats)

fmt.Printf("Heap in use:    %d KB\n", stats.HeapInuse / 1024)
fmt.Printf("Heap allocated: %d KB\n", stats.HeapAlloc / 1024)
fmt.Printf("Total allocs:   %d\n",    stats.TotalAlloc)
fmt.Printf("GC cycles:      %d\n",    stats.NumGC)
fmt.Printf("Last GC pause:  %v\n",    time.Duration(stats.PauseNs[(stats.NumGC+255)%256]))
fmt.Printf("Next GC target: %d KB\n", stats.NextGC / 1024)
```

**Key fields:**

| Field        | What it tells you                                                  |
| ------------ | ------------------------------------------------------------------ |
| `HeapAlloc`  | Currently live heap bytes                                          |
| `HeapInuse`  | Heap memory held by Go (may exceed HeapAlloc due to fragmentation) |
| `TotalAlloc` | Cumulative bytes allocated — always increases                      |
| `NumGC`      | Total GC cycles since start                                        |
| `PauseNs`    | Ring buffer of last 256 GC pause durations                         |
| `NextGC`     | Heap size that will trigger the next GC                            |

**Important caveat:** `ReadMemStats` triggers an STW pause to collect consistent data. Don't call it in a hot loop — use it for periodic metrics export (every 30s, or on a `/debug/metrics` endpoint).

For production, expose these via Prometheus or your metrics library of choice rather than logging them directly.
### Summary — Mental Model

| Topic                      | Key Insight                                                                                    |
| -------------------------- | ---------------------------------------------------------------------------------------------- |
| Stack vs Heap              | Stack = free, scoped. Heap = GC-managed, survives function.                                    |
| Escape analysis            | Pointer returned / interface boxing / goroutine capture → heap. Use `-gcflags='-m'` to verify. |
| GC                         | Concurrent tri-color mark-sweep. Two tiny STW pauses. Tune via GOGC + GOMEMLIMIT.              |
| Value vs pointer receivers | Value = copy (safe for small/immutable). Pointer = shared (mutate or large structs).           |
| Memory alignment           | Order fields largest → smallest to minimize padding. Use `unsafe.Sizeof` to verify.            |
| Zero-copy                  | Stream with `io.Reader`, reuse with `sync.Pool`, avoid full-buffer patterns.                   |
| String/byte                | Every `string(b)` / `[]byte(s)` allocates. Use `unsafe` on hot paths carefully.                |

