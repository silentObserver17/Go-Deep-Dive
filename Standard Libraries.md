## `context`
---
You already covered the API from the previous section. Here we go deeper — **internals, propagation mechanics, and production patterns**.

#### How the cancellation tree actually works

Every derived context holds a reference to its parent. When you call `WithCancel`, Go registers the child with the parent internally. When the parent cancels, it iterates its children and cancels them all — recursively.

```go
// Conceptually what WithCancel does internally:
type cancelCtx struct {
    parent   Context
    mu       sync.Mutex
    done     chan struct{}
    children map[canceller]struct{}  // all derived children
    err      error
}

func (c *cancelCtx) cancel(err error) {
    c.mu.Lock()
    c.err = err
    close(c.done)                        // unblocks all <-ctx.Done() receivers
    for child := range c.children {
        child.cancel(err)                // propagate down the tree
    }
    c.children = nil
    c.mu.Unlock()
}
```

The `Done()` channel is **lazily created** on first access. If you never read `Done()`, the channel is never allocated.

1. **Minimal Resource Usage**

When you create a new context (e.g., `context.WithCancel(parent)` or `context.WithTimeout(...)`), Go doesn't immediately allocate a channel object in memory. A channel is a relatively heavy structure. Instead, the context holds a placeholder or `nil` value for the channel until it is actually needed.

2. **"Lazy" Means "Only When Needed"** 

The channel is only allocated and properly initialized the first time you call the Done() method.

- **If you pass around a context** but never actually check `<-ctx.Done()` (e.g., in a `select` statement), the channel is never created.
- This saves memory and CPU time, particularly in applications that create thousands of contexts for request-scoped operations that finish quickly without needing to be cancelled.

3. **What if you never call Done()?**

If you never call `ctx.Done()`, that channel is never allocated, which avoids unnecessary allocations in the heap. 

#### **Summary Table**

| Scenario           | Does `Done()` trigger? | Who triggers it?       | Why?                                           |
| ------------------ | ---------------------- | ---------------------- | ---------------------------------------------- |
| **Success (Fast)** | No                     | No one                 | Request finished before any signal was needed. |
| **Ctrl+C**         | **Yes**                | Server Shutdown        | To stop all active work immediately.           |
| **Timeout**        | **Yes**                | Internal Timer         | To prevent the request from running forever.   |
| **Manual Cancel**  | **Yes**                | Your code (`cancel()`) | You decided to stop the work early.            |
#### Propagation through goroutines

Context doesn't automatically propagate into goroutines — you have to pass it:
```go
func (s *Service) ProcessOrder(ctx context.Context, orderID int) error {
    // WRONG — goroutine is completely detached from cancellation
    go func() {
        s.sendNotification(orderID)
    }()

    // CORRECT — goroutine respects caller's cancellation
    go func(ctx context.Context) {
        s.sendNotification(ctx, orderID)
    }(ctx)

    return nil
}
```

#### Cascading timeout pattern
Each layer of your stack should narrow the deadline, never widen it:

```go
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Incoming request already has a deadline from the server
    ctx := r.Context()

    // Service layer gets a tighter budget
    svcCtx, cancel := context.WithTimeout(ctx, 4*time.Second)
    defer cancel()

    // DB call gets even tighter
    dbCtx, dbCancel := context.WithTimeout(svcCtx, 2*time.Second)
    defer dbCancel()

    result, err := s.db.QueryContext(dbCtx, query)
    // ...
}
```

If the top-level context (from the HTTP server) cancels first, all children cancel automatically regardless of their own deadlines.

#### The select pattern — the core concurrency idiom with context
```go
func worker(ctx context.Context, jobs <-chan Job) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()  // clean exit

        case job, ok := <-jobs:
            if !ok {
                return nil  // channel closed, we're done
            }
            if err := process(ctx, job); err != nil {
                return err
            }
        }
    }
}
```

#### Detecting timeout vs explicit cancel
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

doSomething(ctx)

switch ctx.Err() {
case context.DeadlineExceeded:
    // timeout hit — maybe retry with backoff
case context.Canceled:
    // caller cancelled — don't retry
case nil:
    // no error
}
```

#### Context values — the right and wrong way
```go
// WRONG — string key collides across packages
ctx = context.WithValue(ctx, "userID", 123)

// WRONG — exported key type, still collision-prone
type Key string
ctx = context.WithValue(ctx, Key("userID"), 123)

// CORRECT — unexported type, guaranteed no collision
type ctxKey int
const (
    ctxKeyUserID ctxKey = iota
    ctxKeyTraceID
    ctxKeyRequestID
)

func WithUserID(ctx context.Context, id int) context.Context {
    return context.WithValue(ctx, ctxKeyUserID, id)
}

func UserIDFrom(ctx context.Context) (int, bool) {
    id, ok := ctx.Value(ctxKeyUserID).(int)
    return id, ok
}
```

#### Gotchas

- **`context.WithValue` lookup is O(n)** — it walks the parent chain comparing keys. Don't store many values; bundle them in a struct if needed.
-  **Never pass a cancelled context to cleanup work** — if your handler cancels but you still need to write a response or log something, use `context.Background()` or a fresh context for that work. Use `context.WithoutCancel`. (Go 1.21+) This creates a new context that keeps all the **values** (like your log-id or auth tokens) but **ignores the cancellation signal** of the parent
- **`http.Request.Context()`** is cancelled when the client disconnects. This means if your DB query is running and the client drops, the query gets killed via context — which is usually what you want.

```go
func MyHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    err := doWork(ctx)
    if err != nil {
        // DETACH here for cleanup
        cleanupCtx := context.WithoutCancel(ctx) 
        log.Info(cleanupCtx, "Work failed, cleaning up...") // This will now actually run!
        db.Exec(cleanupCtx, "UPDATE status SET failed=true")
    }
}

```

### CONTEXT IMPORTANT POINTS: 
---
#### 1. **context ownership**
The rule:
> The function that creates a derived context owns its cancellation.

```GO
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    service.Process(ctx)
}
```

Here:

- handler created the timeout
- handler owns `cancel`
- service should NEVER call `cancel`

This creates clean lifecycle boundaries.

A useful mental model:

```
creator == owner == canceller
```

#### 2. Important nuance: cancellation is cooperative

Context does NOT forcibly stop goroutines.
It only broadcasts:

```go
<-ctx.Done()
```

Your goroutines must voluntarily stop.

Bad:
```go
go func() {
    for {
        computeForever()
    }
}()
```

Good:
```go
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            compute()
        }
    }
}()
```

#### 3. leaked goroutines from ignored contexts

This is probably the #1 real-world concurrency bug.
**Example:**
```go
func worker(ctx context.Context, jobs <-chan Job) {
    for job := range jobs {
        process(job)
    }
}
```

**Problem:**
- worker ignores cancellation
- goroutine survives request death
- memory + CPU leak

Proper:

```go
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return

        case job, ok := <-jobs:
            if !ok {
                return
            }

            process(job)
        }
    }
}
```

Worth emphasizing heavily.

#### 4. Clarify: `WithValue` is NOT immutable data mutation
This misconception appears often.
This:
```go
ctx = context.WithValue(ctx, key, value)
```

does NOT mutate the existing context.

It creates:
```
new context -> parent context
```

Like:
```
ctx3 -> ctx2 -> ctx1 -> Background
```

Value lookup walks upward through the chain.

That explains:

- O(n) lookup
- shadowing behavior

**Example:**
```go
ctx1 := context.WithValue(ctx, "x", 1)
ctx2 := context.WithValue(ctx1, "x", 2)
ctx2.Value("x") // 2
```

because nearest parent wins.


#### 5. contexts should be short-lived

A subtle but important design principle.

Contexts are:
- request-scoped
- operation-scoped

NOT application state.

Bad:

```go
type App struct {    
	ctx context.Context
}
```

because:
- contexts expire
- deadlines become stale
- cancellation accidentally propagates forever

Contexts should flow downward temporarily.

#### 6.  Clarify timeout layering behavior

This is slightly subtle.

Child deadlines cannot extend parent deadlines.

**Example:**
```go
parent: 5s timeout
child: 30s timeout
```

Actual child deadline:

```go
5s
```

The earliest deadline always wins.

#### 7. `errgroup`

In modern Go concurrency, context and `errgroup` are deeply connected.
```go
g, ctx := errgroup.WithContext(ctx)

g.Go(func() error {
    return fetchUser(ctx)
})

g.Go(func() error {
    return fetchOrders(ctx)
})

g.Go(func() error {
    return fetchPayments(ctx)
})

if err := g.Wait(); err != nil {
    return err
}
```

Key idea:

- first goroutine error cancels shared context
- all sibling goroutines stop
- structured concurrency

This is a HUGE modern Go pattern.

##  `sync`  Coordination & Safety
---
#### What Problem It Solves

> Prevent **race conditions** when multiple goroutines access shared memory.

#### `sync.Mutex` and `sync.RWMutex`

```go
type SafeCounter struct {
    mu sync.Mutex
    v  map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.v[key]++
}

func (c *SafeCounter) Value(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.v[key]
}
```

`RWMutex` — multiple concurrent readers OR one writer:

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()            // multiple goroutines can hold RLock simultaneously
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

func (c *Cache) Set(key, val string) {
    c.mu.Lock()             // exclusive — blocks all readers and writers
    defer c.mu.Unlock()
    c.data[key] = val
}
```

**When to use RWMutex vs Mutex:**
- Read-heavy workloads with infrequent writes → `RWMutex` wins.
- Write-heavy or balanced → `Mutex` is simpler and often faster (RWMutex has more overhead).
- If in doubt, start with `Mutex`.

**Critical rules:**

- Never copy a Mutex after first use — it contains internal state. Always embed by value in a struct, pass struct by pointer.
- Lock/Unlock must always be paired. The `defer mu.Unlock()` pattern is idiomatic and prevents forgetting.
- Don't hold a lock across a channel send/receive or any blocking operation — deadlock risk.

```go
// DEADLOCK — holding lock while waiting for a channel
func bad(mu *sync.Mutex, ch chan int) {
    mu.Lock()
    ch <- 1     // blocks waiting for receiver — nobody can acquire mu
    mu.Unlock()
}
```

#### `sync.WaitGroup`
Waits for a collection of goroutines to finish.

```go
func processAll(items []Item) {
    var wg sync.WaitGroup

    for _, item := range items {
        wg.Add(1)                   // increment BEFORE starting goroutine
        go func(item Item) {
            defer wg.Done()         // decrement when done
            process(item)
        }(item)                     // pass item as arg — avoid loop capture
    }

    wg.Wait()                       // blocks until counter reaches 0
}
```

**With error collection:**
```go
func processAll(ctx context.Context, items []Item) error {
    var (
        wg   sync.WaitGroup
        mu   sync.Mutex
        errs []error
    )

    for _, item := range items {
        wg.Add(1)
        go func(item Item) {
            defer wg.Done()
            if err := process(ctx, item); err != nil {
                mu.Lock()
                errs = append(errs, err)
                mu.Unlock()
            }
        }(item)
    }

    wg.Wait()
    return errors.Join(errs...)  // Go 1.20+
}
```

**Gotchas:**

- `Add` must happen **before** the goroutine starts — if you call `Add` inside the goroutine, `Wait` might return before `Add` is even called.
- `Done` is `Add(-1)`. Calling `Done` more times than `Add` panics (counter goes negative).
- `WaitGroup` can be reused after `Wait` returns, but only after the counter reaches zero.

#### `sync.Once`

Guarantees a function runs exactly once, even across concurrent callers. The canonical singleton pattern in Go.

```go
type DB struct {
    once sync.Once
    conn *sql.DB
}

func (d *DB) getConn() *sql.DB {
    d.once.Do(func() {
        var err error
        d.conn, err = sql.Open("postgres", dsn)
        if err != nil {
            panic(err)
        }
    })
    return d.conn
}
```

**Important behaviour:**

- All goroutines calling `Do` concurrently will **block** until the first call completes.
- If the function passed to `Do` panics, `Once` considers it done — subsequent calls will **not** re-run the function.
- `Once` cannot be reset. If you need re-initialization, use a new `Once`.

#### `sync.Cond`
A condition variable — lets goroutines wait for a specific condition to become true, then get notified.

```go
type Queue struct {
    mu    sync.Mutex
    cond  *sync.Cond
    items []int
}

func NewQueue() *Queue {
    q := &Queue{}
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *Queue) Push(item int) {
    q.mu.Lock()
    q.items = append(q.items, item)
    q.mu.Unlock()
    q.cond.Signal()    // wake one waiting goroutine
}

func (q *Queue) Pop() int {
    q.mu.Lock()
    defer q.mu.Unlock()
    for len(q.items) == 0 {
        q.cond.Wait()  // atomically releases lock + sleeps; reacquires on wake
    }
    item := q.items[0]
    q.items = q.items[1:]
    return item
}
```

`Broadcast()` wakes all waiters. `Signal()` wakes one.

**In practice:** `sync.Cond` is rarely the right choice in Go. Channels are almost always cleaner. Use `Cond` when:

- You have a condition that depends on shared state that channels can't express naturally.
- You need `Broadcast` semantics (wake all waiters at once) — like a "start signal" for a pool of workers.

#### `sync.Map`

A concurrent map optimized for two specific patterns:

1. Write-once, read-many (stable keys after initial load).
2. Each goroutine only reads/writes its own disjoint set of keys.

```go
var m sync.Map

// Store
m.Store("key", "value")

// Load
val, ok := m.Load("key")

// LoadOrStore — atomic check-and-set
actual, loaded := m.LoadOrStore("key", "default")
// loaded=true if key existed, actual=existing value
// loaded=false if stored, actual="default"

// Delete
m.Delete("key")

// Iterate
m.Range(func(k, v interface{}) bool {
    fmt.Println(k, v)
    return true  // return false to stop iteration
})
```

**When NOT to use `sync.Map`:**

- General-purpose concurrent map — a plain `map` with `sync.RWMutex` is faster for mixed read/write workloads.
- When you need `Len()` — `sync.Map` has no length operation.
- When you need type safety — everything is `interface{}` / `any`.

#### `sync.Pool`

Object pool for reusing temporarily allocated objects. Reduces GC pressure. Covered in memory section — the key `sync`-specific point:
```go
var pool = sync.Pool{
    New: func() any { return &bytes.Buffer{} },
}

func handler() {
    buf := pool.Get().(*bytes.Buffer)
    buf.Reset()          // must reset state before use
    defer pool.Put(buf)  // return to pool

    buf.WriteString("hello")
    // use buf...
}
```

**GC behaviour:** the GC can clear the pool at any GC cycle. `Pool` is a cache, not a store. Objects in the pool may disappear between any two GC cycles. Never store objects you can't afford to lose.

## `sync/atomic` - Lock-Free Operations

Atomic operations are **lock-free** — they use CPU-level instructions (CAS, XADD, etc.) that are guaranteed to complete without interruption. Faster than mutexes for simple shared scalars.

```go
import "sync/atomic"

var counter int64

// Add and return new value
atomic.AddInt64(&counter, 1)
atomic.AddInt64(&counter, -1)  // decrement

// Load and Store — guaranteed atomic read/write
val := atomic.LoadInt64(&counter)
atomic.StoreInt64(&counter, 0)

// Swap — store new value, return old
old := atomic.SwapInt64(&counter, 100)

// CompareAndSwap (CAS) — the foundation of lock-free algorithms
// If *addr == old, set *addr = new, return true
swapped := atomic.CompareAndSwapInt64(&counter, 100, 200)
```

#### `atomic.Value` — store any value atomically
```go
var config atomic.Value

// Store — value must always be the same concrete type
config.Store(Config{Timeout: 5 * time.Second})

// Load
cfg := config.Load().(Config)

// Hot-reload config without stopping the server
go func() {
    for newCfg := range configUpdates {
        config.Store(newCfg)  // all goroutines reading cfg see this atomically
    }
}()
```

**Rules for `atomic.Value`:**

- Cannot store nil.
- Must always store the same concrete type — storing a different type panics.
- Useful for read-heavy shared config, routing tables, feature flags.

#### Go 1.19+ typed atomics — `atomic.Int64`, `atomic.Bool`, etc.

Much cleaner API, same semantics:
```go
var hits atomic.Int64
var ready atomic.Bool
var ptr atomic.Pointer[Config]

hits.Add(1)
hits.Load()
hits.Store(0)
hits.Swap(100)
hits.CompareAndSwap(100, 200)

ready.Store(true)
if ready.Load() { ... }

ptr.Store(&Config{Timeout: 5 * time.Second})
cfg := ptr.Load()  // returns *Config, typed
```

#### CAS loop — the foundation of lock-free data structures
```go
// Lock-free increment (same as AddInt64, but shows the pattern)
func increment(val *int64) {
    for {
        old := atomic.LoadInt64(val)
        new := old + 1
        if atomic.CompareAndSwapInt64(val, old, new) {
            return  // succeeded — nobody changed val between Load and CAS
        }
        // Failed — another goroutine changed val. Retry.
    }
}
```

This is the ABA problem foundation: load old value → compute new → CAS. If another goroutine mutated and restored the value between Load and CAS, you might not notice. For most counters this doesn't matter. For complex data structures, it's a real concern.

#### When to use atomic vs mutex

|Scenario|Use|
|---|---|
|Single integer counter|`atomic.Int64`|
|Single bool flag|`atomic.Bool`|
|Read-heavy config reload|`atomic.Value` / `atomic.Pointer[T]`|
|Multiple fields updated together|`sync.Mutex` — atomics can't group operations|
|Complex state transitions|`sync.Mutex`|
|Performance-critical hot path|Profile first, then consider atomics|
**Critical distinction:** atomics operate on a **single memory word** atomically. If you need to update two fields atomically (e.g. update both a counter and a timestamp together), you cannot do this with atomics alone — a mutex is required, otherwise a reader can see one field updated but not the other.

```go
// WRONG — reader can see hits updated but lastHit not yet updated
var hits atomic.Int64
var lastHit atomic.Int64

hits.Add(1)
lastHit.Store(time.Now().Unix())  // gap here — not atomic together

// CORRECT — both updated under one lock
type Stats struct {
    mu      sync.Mutex
    hits    int64
    lastHit time.Time
}

func (s *Stats) Record() {
    s.mu.Lock()
    s.hits++
    s.lastHit = time.Now()
    s.mu.Unlock()
}
```

#### Memory ordering — the thing most developers skip
Atomics don't just prevent data races — they also establish **memory ordering guarantees**. In Go, all atomic operations have **sequentially consistent** ordering (the strongest guarantee).

```go
var ready atomic.Bool
var data string

// Goroutine 1
data = "result"
ready.Store(true)   // acts as a memory barrier — data write is visible before this

// Goroutine 2
for !ready.Load() { runtime.Gosched() }
fmt.Println(data)   // safe — guaranteed to see "result"
```

Without the atomic, the CPU or compiler could reorder the writes and goroutine 2 might see `ready=true` but `data=""`. The atomic store/load acts as a **memory barrier** that prevents reordering across it.

This is why `sync/atomic` is the correct way to implement a "ready" flag even if you don't care about the atomic increment — you want the memory ordering.

### How they work together — a real pattern
Here's a worker pool that uses all three packages together, the way you'd actually write it in your backend services:

```go
type WorkerPool struct {
    ctx     context.Context
    cancel  context.CancelFunc
    wg      sync.WaitGroup
    jobs    chan Job
    mu      sync.RWMutex
    results map[string]Result
    done    atomic.Bool
}

func NewWorkerPool(parent context.Context, workers int) *WorkerPool {
    ctx, cancel := context.WithCancel(parent)
    p := &WorkerPool{
        ctx:     ctx,
        cancel:  cancel,
        jobs:    make(chan Job, workers*2),
        results: make(map[string]Result),
    }
    for i := 0; i < workers; i++ {
        p.wg.Add(1)
        go p.worker()
    }
    return p
}

func (p *WorkerPool) worker() {
    defer p.wg.Done()
    for {
        select {
        case <-p.ctx.Done():
            return
        case job, ok := <-p.jobs:
            if !ok {
                return
            }
            result := process(job)
            p.mu.Lock()
            p.results[job.ID] = result
            p.mu.Unlock()
        }
    }
}

func (p *WorkerPool) Submit(job Job) bool {
    if p.done.Load() {
        return false
    }
    select {
    case p.jobs <- job:
        return true
    case <-p.ctx.Done():
        return false
    }
}

func (p *WorkerPool) Shutdown() {
    p.done.Store(true)
    p.cancel()
    p.wg.Wait()
}

func (p *WorkerPool) Result(id string) (Result, bool) {
    p.mu.RLock()
    defer p.mu.RUnlock()
    r, ok := p.results[id]
    return r, ok
}
```

- **context** — cancellation signal flows from `Shutdown()` into every blocked worker.
- **sync.WaitGroup** — `Shutdown()` blocks until every worker exits cleanly.
- **sync.RWMutex** — protects the results map with concurrent reads.
- **atomic.Bool** — fast lock-free check on the hot `Submit` path before touching channels.

### Summary

| Primitive             | Purpose                     | Key Rule                         |
| --------------------- | --------------------------- | -------------------------------- |
| `context.WithCancel`  | Manual cancellation         | Always defer cancel()            |
| `context.WithTimeout` | Time-bound operations       | Each layer narrows deadline      |
| `context.WithValue`   | Request-scoped data         | Unexported key type only         |
| `sync.Mutex`          | Exclusive access            | Never copy after first use       |
| `sync.RWMutex`        | Read-heavy shared state     | RLock for reads, Lock for writes |
| `sync.WaitGroup`      | Wait for goroutine group    | Add before goroutine starts      |
| `sync.Once`           | One-time initialization     | Panic in Do = considered done    |
| `sync.Map`            | Write-once read-many map    | Not general purpose              |
| `sync.Pool`           | Reuse allocations           | GC can clear anytime             |
| `atomic.Int64` etc.   | Single-value lock-free ops  | Can't group multiple fields      |
| `atomic.Value`        | Lock-free config swap       | Same type every Store            |
| CAS loop              | Lock-free state transitions | Retry on failure                 |

# Networking & APIs
---
#### Big Picture First
```
Client → HTTP Server → Middleware → Handler → Business Logic → Response
                                 ↓
                           JSON Encode/Decode
```

You need to understand:

- how requests flow
- how responses are built
- where control points exist (middleware)
- how data is serialized/deserialized

# 1. `net/http` — The Engine
---
Before writing a single handler, understand what happens when a request arrives.

```
Client sends HTTP request
        ↓
net.Listener.Accept()        — TCP connection accepted
        ↓
http.Server reads request    — parses method, URL, headers, body
        ↓
ServeMux.ServeHTTP()         — route matching
        ↓
Handler.ServeHTTP()          — your code runs
        ↓
ResponseWriter writes        — headers flushed on first Write()
        ↓
Connection kept alive or closed (Connection: keep-alive)
```

1. **Client sends HTTP request**

A browser, curl, or another service opens a TCP connection and sends raw HTTP text — method, URL, headers, body.

2. **net.Listener.Accept()**

Go's server sits in a loop waiting for TCP connections. `Accept()` blocks until a client connects, then hands the connection to a new goroutine. **This is why Go handles thousands of concurrent requests** — each gets its own goroutine, not a thread.
```
goroutine per connection
```

3.  **http.Server parses the request**

Go reads and parses the raw bytes into a structured `*http.Request` — pulling out the method, URL, headers, and body into fields you can access.

```
method, URL, headers, body
```

4. **ServeMux.ServeHTTP() — Route Matching**

The router (ServeMux or chi) compares the URL to registered routes and picks the best match. If nothing matches, it returns a 404 automatically.
```
/users/42 → getUser handler
```

5. **Your Handler runs**

Your function gets called with a `ResponseWriter` (to write the response) and a `*Request` (to read the request). This is where your application logic lives.

6. **ResponseWriter sends the response**

The first call to `Write()` or `WriteHeader()` flushes the status line + headers to the client. After that, body bytes stream out as you write them.

7. **Connection kept alive or closed**

With HTTP/1.1 `keep-alive`, the same TCP connection is reused for the next request from the same client, avoiding the overhead of a new TCP handshake every time.

```
Connection: keep-alive
```

## The Two Core Interfaces

Everything in `net/http` revolves around two interfaces. Understand these deeply and the rest clicks into place.

```go
// We write to this - It build the HTTP Response
type ResponseWriter interface {
    Header() http.Header      
    // Get the response headers map. **Set headers here before calling WriteHeader.**

    WriteHeader(statusCode int) 
    // Flush the status code + all headers. Can only be called **once**. After this, headers are locked.
    
    Write([]byte) (int, error) 
    // Write body bytes. If you haven't called WriteHeader yet, it auto-calls it with 200 OK first.
}

// *http.Request
// You read FROM this — it contains the full HTTP request
type Request struct {
    Method     string // The HTTP verb: "GET", "POST", "PUT", "DELETE", etc.
    URL        *url.URL // Parsed URL struct — path, query params, scheme, host.
    Header     http.Header // Map of all incoming request headers (auth tokens, content-type, etc.)
    Body       io.ReadCloser  // The request body as a readable stream. **Always close it** when done.
    Context()  context.Context // Carries deadlines, cancellation signals, and request-scoped values.
    Form       url.Values     // populated after ParseForm()
    // ...
}
```


### The WriteHeader Trap — Most Common Mistake

This trips up nearly every Go developer at least once. Headers **must** be set before the response is sent — once you call `WriteHeader()` (or write a body), they're gone.

###### **❌ Wrong — Headers set too late**
```go
func handler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(201) // headers flushed HERE

    // TOO LATE — already sent!
    w.Header().Set("Content-Type", "application/json")
}
```

###### Correct — Headers set first
```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Set headers BEFORE writing
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(201)

    json.NewEncoder(w).Encode(response)
}
```

**Why doesn't Go error when you set headers too late?**

The mutation is silently ignored. Go won't panic or return an error — the header just doesn't appear in the response. This makes the bug easy to miss during development. Always set headers first, then write status, then write body.

## Configuring the Server Correctly

The shortcut `http.ListenAndServe(":8080", handler)` looks clean but is dangerous in production. It uses zero timeouts, which means a single slow or malicious client can hold open a goroutine forever.

```go
srv := &http.Server{
    Addr:    ":8080",
    Handler: router,

    // Time limit to read the FULL request (including body)
    ReadTimeout: 5 * time.Second,

    // Specifically for reading just the headers
    // Protects against "Slowloris" attack (slow header sending)
    ReadHeaderTimeout: 2 * time.Second,

    // Time limit to write the full response back to client
    WriteTimeout: 10 * time.Second,

    // How long to keep an idle keep-alive connection alive
    IdleTimeout: 120 * time.Second,

    // Max bytes for request headers (1MB)
    MaxHeaderBytes: 1 << 20,
}
```

>**Never use http.ListenAndServe in production**
	With no timeouts, a client sending 1 byte per minute holds a goroutine open forever. The Slowloris attack exploits exactly this — many slow connections that exhaust your server's resources. Always configure timeouts.

### What each timeout actually protects

1.  **ReadHeaderTimeout (2s)**

	The client must finish sending all HTTP headers within this time. Prevents the Slowloris attack where an attacker slowly drip-feeds headers to tie up your goroutine.

2. **ReadTimeout (5s)**

	The entire request — headers + body — must be read within this window. Prevents slow body uploads from holding connections open.

3. **WriteTimeout (10s)**

	Your handler + response writing must complete in this time. Prevents slow clients (or your own slow handlers) from holding connections open indefinitely.

4. **IdleTimeout (120s)**

	With keep-alive, a connection can sit idle between requests. This closes connections that haven't made a new request in 2 minutes, reclaiming resources.

## Graceful Shutdown

When you deploy a new version of your service, you need to stop the old server **without killing requests that are already in flight**. That's what graceful shutdown does.

```go
func run() error {
    srv := &http.Server{Addr: ":8080", Handler: buildRouter()}

    // Shutdown signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)

    // Start server in background
    errCh := make(chan error, 1)
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            errCh <- err
        }
    }()

    // Wait for signal or error
    select {
    case err := <-errCh:
        return err
    case <-quit:
    }

    // Give in-flight requests 30s to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    return srv.Shutdown(ctx)
}
```

##### **What Shutdown() actually does**

It immediately stops accepting new connections, closes idle keep-alive connections, then waits for all active request handlers to return. If your 30-second context expires before they finish, it returns an error and you can force-close.

## ServeMux and Routing

The router's job: look at the incoming URL and decide which handler to run. Go has two options — the built-in `ServeMux` and the popular `chi` library.
```go
mux := http.NewServeMux()

// Exact match — ONLY /health, not /health/check
mux.HandleFunc("/health", healthHandler)

// Exact match — ONLY /api/users
mux.HandleFunc("/api/users", usersHandler)

// Subtree match — /api/ AND everything under it
// Trailing slash = "match this path and all children"
mux.HandleFunc("/api/", apiHandler)

// No path parameters, no method matching
// All methods (GET/POST/etc) go to same handler
```

**GO(1.22+ upgraded)**
```go
mux := http.NewServeMux()

// Method + path in one string (Go 1.22+)
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // extract {id} from URL
    // id = "42" for request to /users/42
})

mux.HandleFunc("POST /users", createUser)
mux.HandleFunc("DELETE /users/{id}", deleteUser)

// Now nearly as capable as chi for basic routing
// But chi still wins for middleware composition
```

**Chi Router:** 
```go
r := chi.NewRouter()

// Clean method-based routing
r.Get("/users/{id}", getUser)
r.Post("/users", createUser)

// Grouped routes with shared middleware
r.Route("/admin", func(r chi.Router) {
    r.Use(AdminOnly) // only runs for /admin/* routes
    r.Get("/stats", statsHandler)
    r.Delete("/users/{id}", deleteUser)
})

// chi shines: middleware composition, groups,
// URL param extraction, nested routers
```


## Middleware — Wrapping Handlers

Middleware is code that runs **before and/or after** your handler — without modifying the handler itself. Common uses: logging, authentication, rate limiting, panic recovery.

The key insight: **middleware is just a function that takes a handler and returns a new handler**. It wraps behavior around the original.

```go
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {

        start := time.Now()

        // ↓ PRE-HANDLER: runs BEFORE your actual handler
        log.Printf("→ %s %s", r.Method, r.URL.Path)

        next.ServeHTTP(w, r)  // call the real handler

        // ↑ POST-HANDLER: runs AFTER your actual handler returns
        log.Printf("← %s %s took %v", r.Method, r.URL.Path, time.Since(start))
    })
}
```

### Request Flow Through the Middleware Chain

Middleware is applied right-to-left in code, but requests flow left-to-right at runtime. Think of it as concentric layers — the outermost middleware is the first to run.

```
Request Flow  →  LOGGING → AUTH → RATE LIMIT → HANDLER
Response Flow ←  LOGGING ← AUTH ← RATE LIMIT ← HANDLER
```

```go
// Apply right-to-left: request flows left-to-right through the chain
handler := LoggingMiddleware(
    AuthMiddleware(
        RateLimitMiddleware(
            myHandler,
        ),
    ),
)
```

>**Short-circuiting: the most important middleware concept**
	If middleware decides the request should be rejected (bad auth token, rate limit exceeded), it writes the error response and returns WITHOUT calling `next.ServeHTTP()`. The chain stops there — the actual handler never runs.

### Real middleware implementations

##### 1. **Auth middleware — JWT validation:**

```go
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return  // do NOT call next — short circuit
        }

        claims, err := validateJWT(strings.TrimPrefix(token, "Bearer "))
        if err != nil {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }

        // Attach claims to context for downstream handlers
        ctx := context.WithValue(r.Context(), ctxKeyUserID, claims.UserID)
        next.ServeHTTP(w, r.WithContext(ctx))  // pass enriched request
    })
}

// In handler — extract from context
func profileHandler(w http.ResponseWriter, r *http.Request) {
    userID, ok := r.Context().Value(ctxKeyUserID).(int)
    if !ok {
        http.Error(w, "unauthorized", http.StatusUnauthorized)
        return
    }
    // use userID
}
```

##### 2. **Rate limiting middleware:**
```go
func RateLimitMiddleware(rps int) func(http.Handler) http.Handler {
    limiter := rate.NewLimiter(rate.Limit(rps), rps)  // golang.org/x/time/rate
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                w.Header().Set("Retry-After", "1")
                http.Error(w, "too many requests", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

##### 3. **Recovery middleware:**
```go
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                stack := debug.Stack()
                log.Printf("PANIC: %v\n%s", err, stack)
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```


##### 4. **Request ID middleware — trace requests across services:**
```go
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = uuid.New().String()
        }
        ctx := context.WithValue(r.Context(), ctxKeyRequestID, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```


## HTTP Client — Making Outbound Requests

When your server needs to call another service, you use `http.Client`. The default client (`http.DefaultClient`) has no timeout — never use it in production.

```go
// Always use a custom client — never http.DefaultClient in production
client := &http.Client{
    Timeout: 10 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 5 * time.Second,
    },
}

// Always use request context
req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, body)
if err != nil {
    return fmt.Errorf("building request: %w", err)
}
req.Header.Set("Content-Type", "application/json")
req.Header.Set("Authorization", "Bearer "+token)

resp, err := client.Do(req)
if err != nil {
    return fmt.Errorf("executing request: %w", err)
}
defer resp.Body.Close()  // always close — even if you don't read the body

if resp.StatusCode != http.StatusOK {
    body, _ := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
    return fmt.Errorf("unexpected status %d: %s", resp.StatusCode, body)
}
```

**Gotchas:**

- `http.DefaultClient` has no timeout — a slow server will hang your goroutine forever.
- `resp.Body` must always be closed — even on error responses, even if you don't read it. Not closing leaks the TCP connection.
- `resp.Body` must be **fully read** before closing if you want the connection returned to the pool. If you close without reading, the connection is discarded.

```go
// Drain and close — return connection to pool
defer func() {
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
}()
```

##  `encoding/json` — Serialization Layer
---
### Struct Tags — The First Thing to Understand

When you define a Go struct, the backtick annotations control how it appears in JSON:
```go
type User struct {
    ID       int    `json:"id"`           // Go field "ID" → JSON key "id"
    Email    string `json:"email,omitempty"` // omit this field if it's empty string
    Password string `json:"-"`            // never show this in JSON, ever
    internal string // no tag needed — unexported fields are always ignored
}
```

Three things you'll use constantly:

- `json:"name"` — renames the field in JSON output
- `omitempty` — skips the field if it's the zero value (empty string, 0, false, nil)
- `json:"-"` — completely excludes the field (perfect for passwords, secrets)

### Marshal vs Unmarshal — The Basics

```go
// Struct → JSON
data, err := json.Marshal(user)

// JSON → Struct
var u User
err = json.Unmarshal(data, &u)
```

Simple enough. But in HTTP handlers, you rarely want these — you want the streaming versions.

### Encoder/Decoder — Use These for HTTP

`json.Marshal` creates the full JSON in memory first, then you'd write it to the response. For large payloads, that's wasteful.
**Instead, write directly to the response:**
```go
json.NewEncoder(w).Encode(user)  // streams directly to ResponseWriter
```

**And read directly from the request body:**
```go
json.NewDecoder(r.Body).Decode(&user)  // reads from the body stream
```

No intermediate buffer. For most APIs this difference is negligible, but it's the correct habit.

### Struct Embedding
If you embed a struct, its fields get "promoted" to the top level in JSON:

```go
type Base struct {
    ID        int    `json:"id"`
    CreatedAt string `json:"created_at"`
}

type User struct {
    Base              // embedded — not a named field
    Name string `json:"name"`
}
```
This marshals as `{"id":1,"created_at":"...","name":"Alice"}` — the Base fields appear at the top level, not nested under a `"base"` key. Useful for avoiding repetition across many struct types.

### Custom Marshaling

Sometimes the default behavior isn't what you want. Say you have a status stored as an `int` internally but want it as a string in JSON:
```go
type Status int

func (s Status) MarshalJSON() ([]byte, error) {
    switch s {
    case StatusActive:
        return []byte(`"active"`), nil
    case StatusInactive:
        return []byte(`"inactive"`), nil
    }
    return nil, fmt.Errorf("unknown status: %d", s)
}
```

Implement `MarshalJSON()` on the type and Go will call your function instead of the default behavior. Do the reverse with `UnmarshalJSON()` to parse `"active"` back to the integer constant. This keeps your API surface clean while your internals stay efficient.

### json.RawMessage — Deferred Decoding

When the shape of a JSON field depends on another field's value, you can't decode everything upfront:
```go
type Event struct {
    Type    string          `json:"type"`
    Payload json.RawMessage `json:"payload"` // stored as raw bytes, not decoded yet
}
```

`json.RawMessage` is just `[]byte` — it holds the raw JSON exactly as received. You decode the outer struct first, then look at `Type` to know which struct to decode `Payload` into. This is the standard pattern for event-driven systems and webhook handlers.

### map[string]interface{} and the float64 Gotcha

When you decode JSON into an untyped map, all numbers become `float64`. Not `int`, not `int64` — `float64`. This surprises almost everyone the first time:
```go
var m map[string]interface{}
json.Unmarshal(data, &m)

age := m["age"].(float64) // you have to do this, even if age is clearly an integer
```

If you need actual integers, use `json.Number`:
```go
dec := json.NewDecoder(bytes.NewReader(data))
dec.UseNumber()
dec.Decode(&m)

age, _ := m["age"].(json.Number).Int64() // now you get a proper int
```

### Pointer Fields for PATCH Endpoints

This is a pattern worth knowing well. For a PATCH endpoint, you need to tell the difference between "the client didn't send this field" and "the client sent this field as empty."

Regular `string` field: you can't tell the difference — it's `""` either way.

Pointer `*string` field: nil means not sent, `&""` means explicitly set to empty.

```go
type UpdateRequest struct {
    Name  *string `json:"name"`
    Email *string `json:"email"`
}
```

If the request body is `{"name":"Alice"}`, then `Name` is `&"Alice"` and `Email` is `nil`. Your update logic can then only touch the fields that aren't nil — leaving the rest unchanged in the database.

### Decoding Request Bodies Safely
A minimal helper you should have in every API:

```go
func decodeJSON(r *http.Request, v any) error {
    dec := json.NewDecoder(io.LimitReader(r.Body, 1<<20)) // cap at 1MB
    dec.DisallowUnknownFields() // reject fields not in your struct

    if err := dec.Decode(v); err != nil {
        // handle syntax errors, type mismatches, empty body separately
        return err
    }
    if dec.More() {
        return fmt.Errorf("body must contain a single JSON object")
    }
    return nil
}
```

Three things this does beyond bare `Decode`:

- **Size limit** — prevents someone sending a 500MB body and exhausting your memory
- **`DisallowUnknownFields`** — rejects requests with fields you don't recognize, useful for catching client bugs early
- **`dec.More()` check** — rejects bodies with multiple JSON values concatenated together

### A Consistent Response Envelope

Rather than writing ad-hoc JSON in every handler, define a standard shape once:
```go
type APIResponse[T any] struct {
    Success bool   `json:"success"`
    Data    T      `json:"data,omitempty"`
    Error   string `json:"error,omitempty"`
}
```

Then two helpers cover everything:

```go
writeSuccess(w, http.StatusOK, user)
writeError(w, http.StatusNotFound, "user not found")
```
Every response from your API looks the same. Clients know exactly what to expect.

### Performance

`encoding/json` uses reflection on every call — it inspects your struct at runtime to figure out what to encode. For typical CRUD APIs, this is never your bottleneck; your database queries will be 10–100x slower than JSON encoding.

If you genuinely profile and find JSON is the problem, drop-in replacements exist: `go-json` and `jsoniter` are 2–3x faster with zero code changes (same API). `easyjson` generates code at build time for zero reflection overhead.

### Full Request Lifecycle — putting it together
Here's how a real request flows through a production backend:

```
POST /api/v1/users
        ↓
[TCP] net.Listener accepts connection
        ↓
[TLS] handshake (if HTTPS)
        ↓
[HTTP] http.Server reads & parses request
        ↓
[Middleware 1] RequestID — attach/generate X-Request-ID
        ↓
[Middleware 2] RealIP — extract true client IP from X-Forwarded-For
        ↓
[Middleware 3] Logger — record method, path, start time
        ↓
[Middleware 4] Recovery — defer recover() for panics
        ↓
[Middleware 5] Auth — validate JWT, attach userID to ctx
        ↓
[Middleware 6] RateLimit — check per-user rate
        ↓
[Router] chi matches POST /api/v1/users → createUser handler
        ↓
[Handler] decodeJSON(r, &req) — decode + validate body
        ↓
[Handler] svc.CreateUser(ctx, req) — business logic
        ↓
[Service] repo.Insert(ctx, user) — DB call with context
        ↓
[Handler] writeSuccess(w, 201, user) — encode response
        ↓
[Middleware 3] Logger — record status code, duration
        ↓
[HTTP] response flushed to client
        ↓
[TCP] connection returned to keep-alive pool
```

### Gotchas Summary

**net/http:**

- Always set server timeouts — `ReadTimeout`, `WriteTimeout`, `IdleTimeout`.
- `http.DefaultClient` has no timeout — never use in production.
- `resp.Body` must always be closed AND drained to reuse TCP connections.
- Headers must be set before `WriteHeader` or `Write`.
- Panics in goroutines spawned by handlers are NOT caught by recovery middleware.

**encoding/json:**

- Numbers in `interface{}` decode as `float64` — use `json.Number` if you need integers.
- `omitempty` on a struct field never omits it (structs are never "empty") — use a pointer.
- `json.NewEncoder` appends a trailing newline — `json.Marshal` does not.
- `DisallowUnknownFields` is strict — good for internal APIs, usually too strict for public ones.
- Unexported fields are silently ignored — no error, just missing data.

```go
// omitempty on struct — doesn't work as expected
type Response struct {
    Meta Metadata `json:"meta,omitempty"`  // never omitted — structs aren't "empty"
}

// Fix — use pointer
type Response struct {
    Meta *Metadata `json:"meta,omitempty"`  // omitted when nil
}
```

## I/O System (Deep Understanding)
---
We’ll build this step by step:

```
os     → talks to OS (files, processes)
io     → defines universal interfaces (Reader/Writer)
bufio  → optimizes I/O (buffering layer)
```

### First: The Big Philosophy (VERY IMPORTANT)

Go follows a strong **Unix philosophy**:

> “Everything is a stream of bytes.”

That means:

- file = stream of bytes
- network = stream of bytes
- HTTP body = stream of bytes
- stdin/stdout = stream of bytes

👉 Go unifies all of this using **interfaces**, not classes.

The `io` package defines the **abstract interfaces** for this — what it means to be readable or writable, without caring what the underlying thing actually is. `bufio` sits on top of `io` and adds a **memory buffer** between your code and the underlying stream, so you're not making a system call for every single byte. `os` is the **concrete layer** — actual files on disk, actual environment variables, actual process signals.

Think of it as three layers:

```
Your code
    ↕
bufio     — buffer layer (amortizes system calls)
    ↕
io        — abstract interface layer (Reader, Writer, etc.)
    ↕
os        — concrete OS layer (files, stdin, stdout)
    ↕
Kernel    — actual syscalls (read, write, open, close)
```

Understanding why each layer exists makes all three packages click.

### `io` — The Interface Layer
---
The `io` package doesn't implement any real I/O. It only defines **what shapes things must have** to participate in the I/O ecosystem. This is the power of Go's interface system applied to data streaming.

The foundational insight is: if you write a function that accepts `io.Reader` instead of `*os.File`, that function automatically works with files, HTTP request bodies, in-memory byte buffers, gzip streams, network connections, test fakes — anything that knows how to produce bytes. You wrote the function once and got infinite flexibility for free.

#### `io.Reader` and `io.Writer` — the two atomic interfaces

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

`Read` says: "fill this byte slice with however many bytes you have right now, tell me how many you put in, and tell me if anything went wrong." The key word is **however many** — `Read` is allowed to return fewer bytes than `len(p)` even if more data is coming. This is called a short read and it's completely legal. Your calling code must handle it. This is why most code uses helper functions rather than calling `Read` directly.

`Write` says: "take these bytes and send them wherever you send things." It must either write all of `len(p)` bytes or return an error. Partial writes without an error are a bug in the Writer.

```go
// Read used directly — you have to handle short reads yourself
func readExactly(r io.Reader, n int) ([]byte, error) {
    buf := make([]byte, n)
    total := 0
    for total < n {
        read, err := r.Read(buf[total:])
        total += read
        if err == io.EOF {
            break
        }
        if err != nil {
            return nil, err
        }
    }
    return buf[:total], nil
}

// io.ReadFull does this for you — reads exactly len(buf) bytes or errors
buf := make([]byte, 1024)
n, err := io.ReadFull(r, buf)
// err is io.ErrUnexpectedEOF if stream ended before buf was filled
```

#### `io.EOF` — not really an error

`io.EOF` is a sentinel value that means "the stream ended normally, there's nothing more to read." The convention in Go is that when `Read` returns `n > 0` and `io.EOF` simultaneously, you should process the `n` bytes first, then handle EOF on the next call. Different implementations do this differently, so always check `n > 0` before checking the error.

```go
buf := make([]byte, 512)
for {
    n, err := r.Read(buf)
    if n > 0 {
        process(buf[:n])  // always process n bytes FIRST
    }
    if err == io.EOF {
        break             // clean end of stream
    }
    if err != nil {
        return err        // actual error
    }
}
```

#### The helper functions — what you'll actually call
Rather than calling `Read` and `Write` directly and managing short reads, use the standard helpers:

```go
// io.Copy — the workhorse. Reads from src, writes to dst, until EOF.
// Internally uses a 32KB buffer. Returns bytes copied.
n, err := io.Copy(dst, src)

// io.CopyN — copy exactly n bytes then stop
n, err := io.CopyN(dst, src, 1024*1024)  // copy 1MB

// io.ReadAll — read entire stream into memory. Use carefully on large streams.
data, err := io.ReadAll(r)

// io.ReadFull — read exactly len(buf) bytes, error if stream ends early
n, err := io.ReadFull(r, buf)

// io.LimitReader — wraps a reader, stops after n bytes
// Critical for security — prevents a malicious client sending infinite data
limited := io.LimitReader(r, 1<<20)  // max 1MB
data, err := io.ReadAll(limited)

// io.MultiReader — concatenate multiple readers into one sequential stream
// Useful for prepending data to a stream without copying
header := strings.NewReader("HEADER:")
body := strings.NewReader("actual content")
combined := io.MultiReader(header, body)
// Reading from combined gives: "HEADER:actual content"

// io.TeeReader — reads from r, simultaneously writes everything to w
// Like a T-junction in a pipe
var log bytes.Buffer
tee := io.TeeReader(resp.Body, &log)
json.NewDecoder(tee).Decode(&result)
// log.Bytes() now contains the raw JSON that was decoded
// Useful for debugging — log the raw body while also parsing it

// io.Pipe — synchronous in-memory pipe, connects a writer to a reader
// No buffering — Write blocks until Read consumes the data
pr, pw := io.Pipe()
go func() {
    json.NewEncoder(pw).Encode(largeObject)
    pw.Close()
}()
http.Post(url, "application/json", pr)
// The JSON encoder writes directly into the HTTP request body
// Nothing is buffered in memory — true streaming
```

#### The richer interfaces — capability discovery

Go's `io` package defines a family of interfaces that extend the two basics. The standard library uses these to discover whether an implementation has extra capabilities and use them if available:

```go
type ReadWriter interface {
    Reader
    Writer
}

type ReadCloser interface {
    Reader
    Closer  // Close() error
}

type WriteCloser interface {
    Writer
    Closer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// Seeking — jump to a position in the stream
type Seeker interface {
    Seek(offset int64, whence int) (int64, error)
    // whence: io.SeekStart=0, io.SeekCurrent=1, io.SeekEnd=2
}

// Reading at a specific offset without seeking (for concurrent access)
type ReaderAt interface {
    ReadAt(p []byte, off int64) (n int, err error)
}

// Writing at a specific offset
type WriterAt interface {
    WriteAt(p []byte, off int64) (n int, err error)
}

// Single byte reading — bufio uses this internally
type ByteReader interface {
    ReadByte() (byte, error)
}

// Writing a string efficiently (avoids []byte conversion)
type StringWriter interface {
    WriteString(s string) (n int, err error)
}
```

`io.Copy` for example checks whether the `src` implements `io.WriterTo` or the `dst` implements `io.ReaderFrom` — if either does, it uses that path instead of its generic buffer loop. `*os.File` on Linux implements `io.ReaderFrom` using the `sendfile` syscall, which transfers data from one file descriptor to another entirely in the kernel — zero copies into user space. You get this automatically just by using `io.Copy`.

#### `io.SectionReader` — read a slice of a larger stream
```go
// Read bytes 100–200 of a file without loading the whole file
f, _ := os.Open("large.bin")
section := io.NewSectionReader(f, 100, 100)  // offset=100, size=100
data, _ := io.ReadAll(section)
// SectionReader implements Reader, Seeker, ReaderAt
// Safe for concurrent use — each SectionReader has its own offset
```

### `bufio` — The Buffer Layer

#### Why buffering exists — the system call problem
Every call to `Read` or `Write` on an `*os.File` or a network connection is a **syscall** — your program crosses from user space into kernel space, does the work, and comes back. This context switch costs roughly 1–10 microseconds. That sounds tiny, but consider reading a 1MB file one byte at a time: that's 1,048,576 syscalls, potentially adding up to seconds of overhead for something that could be done with 32 syscalls using a 32KB buffer.

`bufio.Reader` solves this by doing one large read from the underlying `io.Reader` into an internal memory buffer (default 4096 bytes), then serving your small read requests from that buffer. Your code sees the same `Read` interface but most calls never touch the kernel — they just copy bytes from one memory region to another.

`bufio.Writer` works the same way in reverse: it accumulates your small writes into its buffer and only calls the underlying `Write` when the buffer is full, or when you explicitly call `Flush`.

```
Your code calls Read(1 byte)
    ↓
bufio.Reader checks its internal buffer
    → buffer has data: return 1 byte from buffer (no syscall)
    → buffer empty: call underlying Read(4096 bytes) once,
                    fill buffer, return 1 byte (1 syscall serves 4096 requests)
```

#### `bufio.Reader`
```go
// Wrap any io.Reader to add buffering
r := bufio.NewReader(file)
r := bufio.NewReaderSize(file, 65536)  // custom buffer size (64KB here)

// ReadByte — single byte, buffered
b, err := r.ReadByte()

// UnreadByte — put the last byte back (peek-and-decide pattern)
r.UnreadByte()

// ReadRune — reads a complete UTF-8 character (1-4 bytes)
ch, size, err := r.ReadRune()

// ReadSlice — read until delimiter, returns slice of internal buffer
// WARNING: returned slice is only valid until next read
line, err := r.ReadSlice('\n')

// ReadLine — reads a line, handles long lines by returning isPrefix=true
// Low-level — prefer ReadString or Scanner
line, isPrefix, err := r.ReadLine()

// ReadString — reads until delimiter inclusive, returns new string (safe, allocates)
line, err := r.ReadString('\n')
// err is io.EOF on last line with no trailing newline — still gives you the data

// ReadBytes — same but returns []byte
line, err := r.ReadBytes('\n')

// Peek — look ahead n bytes without consuming them
// Returns slice of internal buffer — only valid until next read
header, err := r.Peek(4)
if string(header) == "GZIP" { ... }
```

#### `bufio.Writer`
```go
w := bufio.NewWriter(file)
w := bufio.NewWriterSize(file, 65536)

// These writes go into the buffer, not to disk yet
w.WriteString("line 1\n")
w.WriteByte('\n')
w.WriteRune('€')
fmt.Fprintf(w, "formatted %d\n", 42)

// CRITICAL — Flush pushes buffered data to the underlying writer
// If you forget this, your last writes may be lost silently
if err := w.Flush(); err != nil {
    return fmt.Errorf("flushing: %w", err)
}

// Always flush, even on error paths — use defer carefully
// defer w.Flush() is tempting but its error is silently discarded
// Better pattern:
func writeData(path string, data []string) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer func() {
        if cerr := f.Close(); cerr != nil && err == nil {
            err = cerr
        }
    }()

    w := bufio.NewWriter(f)
    for _, line := range data {
        if _, err := w.WriteString(line + "\n"); err != nil {
            return err
        }
    }
    return w.Flush()  // explicit flush before close
}
```

#### `bufio.ReadWriter` — bidirectional buffering

```go
// Useful for TCP connections where you read and write on the same connection
rw := bufio.NewReadWriter(
    bufio.NewReader(conn),
    bufio.NewWriter(conn),
)
rw.WriteString("hello\n")
rw.Flush()
response, _ := rw.ReadString('\n')
```

#### `bufio.Scanner` — the right way to read line by line

`Scanner` is the high-level API for reading structured input. It hides all the EOF handling and delimiter complexity that makes `ReadString` error-prone.

```go
// Reading a file line by line — the idiomatic way
f, _ := os.Open("data.txt")
defer f.Close()

scanner := bufio.NewScanner(f)
// Default: splits on lines (strips the newline from the token)

for scanner.Scan() {          // Scan() advances and returns true if token found
    line := scanner.Text()    // current token as string (allocates)
    // or:
    line := scanner.Bytes()   // current token as []byte (references internal buffer — don't store)
    process(line)
}

if err := scanner.Err(); err != nil {  // check AFTER the loop, not inside
    return fmt.Errorf("scanning: %w", err)
}
// Note: io.EOF is NOT returned by scanner.Err() — EOF is normal termination
```

**Custom split functions** — Scanner is more powerful than just line splitting:

```go
// Built-in split functions
scanner.Split(bufio.ScanLines)   // default — lines without \n
scanner.Split(bufio.ScanWords)   // whitespace-delimited words
scanner.Split(bufio.ScanBytes)   // one byte at a time
scanner.Split(bufio.ScanRunes)   // one UTF-8 rune at a time

// Custom split — tokenize CSV fields for example
scanner.Split(func(data []byte, atEOF bool) (advance int, token []byte, err error) {
    // data: unprocessed bytes in the buffer
    // atEOF: true if no more data will come
    // advance: how many bytes to consume
    // token: the token to return (nil = no token yet, keep reading)
    // err: non-nil stops scanning

    if atEOF && len(data) == 0 {
        return 0, nil, nil  // done
    }
    if i := bytes.IndexByte(data, ','); i >= 0 {
        return i + 1, data[:i], nil  // found comma — return field before it
    }
    if atEOF {
        return len(data), data, nil  // last field with no trailing comma
    }
    return 0, nil, nil  // need more data
})
```

**Scanner buffer size** — the default is 64KB per token. If you have lines longer than 64KB, you'll get `bufio.ErrTooLong`:

```go
scanner := bufio.NewScanner(f)
buf := make([]byte, 0, 1024*1024)         // 1MB initial
scanner.Buffer(buf, 10*1024*1024)          // up to 10MB per token
```

### `os` — The Concrete Layer
#### Why `os` is designed the way it is

The `os` package provides a **platform-independent interface to OS features**. The same code works on Linux, macOS, and Windows because `os` hides the differences. But it's deliberately thin — it exposes OS concepts (files, processes, signals, environment) without abstracting over them. If you need the raw Unix syscall (`flock`, `mmap`, `ioctl`), you go to the `syscall` or `golang.org/x/sys` package.

#### Files — opening and creating
```go
// os.Open — read-only. Equivalent to OpenFile(name, O_RDONLY, 0)
f, err := os.Open("input.txt")
if err != nil {
    return fmt.Errorf("opening input: %w", err)
}
defer f.Close()

// os.Create — create or truncate for writing. Equivalent to OpenFile(name, O_RDWR|O_CREATE|O_TRUNC, 0666)
f, err := os.Create("output.txt")
defer f.Close()

// os.OpenFile — full control over flags and permissions
f, err := os.OpenFile("app.log",
    os.O_APPEND|os.O_CREATE|os.O_WRONLY,  // append to existing or create
    0644,                                  // rw-r--r-- permissions
)

// Common flag combinations:
// O_RDONLY                          — read only
// O_WRONLY | O_CREATE | O_TRUNC     — write, create fresh (os.Create)
// O_WRONLY | O_CREATE | O_APPEND    — append to log file
// O_RDWR                            — read and write
// O_EXCL | O_CREATE                 — fail if file exists (atomic creation)
```

#### Reading and writing files
```go
// Read entire file — convenience function
data, err := os.ReadFile("config.json")

// Write entire file atomically-ish — creates or truncates
err := os.WriteFile("output.txt", data, 0644)

// Streaming read with io.Copy
src, _ := os.Open("input.txt")
dst, _ := os.Create("output.txt")
defer src.Close()
defer dst.Close()
n, err := io.Copy(dst, src)  // uses sendfile on Linux — kernel-level copy

// Buffered read — wrap with bufio for line-by-line
f, _ := os.Open("large.log")
scanner := bufio.NewScanner(f)
for scanner.Scan() {
    process(scanner.Text())
}

// Seeking — jump to a specific position
f, _ := os.Open("data.bin")
f.Seek(1024, io.SeekStart)    // jump to byte 1024 from start
f.Seek(-100, io.SeekEnd)      // jump to 100 bytes before end
f.Seek(50, io.SeekCurrent)    // jump 50 bytes forward from current position

// ReadAt — read at offset without changing the file's position
// Safe for concurrent reads from multiple goroutines
buf := make([]byte, 100)
n, err := f.ReadAt(buf, 512)  // read 100 bytes starting at offset 512
```

#### File metadata
```go
// Stat — get file info
info, err := os.Stat("file.txt")
// info.Name()    — filename
// info.Size()    — bytes
// info.Mode()    — permissions + file type bits
// info.ModTime() — last modification time
// info.IsDir()   — true if directory
// info.Sys()     — underlying OS-specific data (*syscall.Stat_t on Unix)

// Lstat — like Stat but doesn't follow symlinks
info, err := os.Lstat("link")

// Check if file exists
_, err := os.Stat(path)
if os.IsNotExist(err) {
    // file doesn't exist
}
// Or more idiomatically with Go 1.13+:
if errors.Is(err, os.ErrNotExist) {
    // file doesn't exist
}
```

#### Directory operations
```go
// Create directory
err := os.Mkdir("mydir", 0755)       // single directory, fails if exists
err := os.MkdirAll("a/b/c", 0755)   // create full path, no-op if exists

// Remove
err := os.Remove("file.txt")         // file or empty directory
err := os.RemoveAll("dir/")          // directory tree recursively

// Rename — atomic on same filesystem (mv)
err := os.Rename("old.txt", "new.txt")

// Read directory contents
entries, err := os.ReadDir(".")
for _, entry := range entries {
    fmt.Println(entry.Name(), entry.IsDir(), entry.Type())
    info, _ := entry.Info()  // get full FileInfo if needed
}

// Walk a directory tree
err := filepath.WalkDir(".", func(path string, d fs.DirEntry, err error) error {
    if err != nil {
        return err  // error accessing path
    }
    if d.IsDir() && d.Name() == ".git" {
        return filepath.SkipDir  // skip entire .git directory
    }
    if !d.IsDir() && strings.HasSuffix(path, ".go") {
        process(path)
    }
    return nil
})
```

#### Temporary files and directories
```go
// TempFile — creates a file in dir with a random suffix
// Using "" for dir uses the OS default temp directory
f, err := os.CreateTemp("", "upload-*.tmp")
// f.Name() gives something like /tmp/upload-3829164729.tmp
defer os.Remove(f.Name())  // clean up when done
defer f.Close()

// TempDir — creates a temporary directory
dir, err := os.MkdirTemp("", "workspace-*")
defer os.RemoveAll(dir)
```

#### stdin, stdout, stderr — the standard streams
```go
// These are *os.File values — they implement io.Reader/Writer
os.Stdin   // *os.File — read user input or piped input
os.Stdout  // *os.File — normal output
os.Stderr  // *os.File — error/diagnostic output

// Reading from stdin
scanner := bufio.NewScanner(os.Stdin)
for scanner.Scan() {
    line := scanner.Text()
    fmt.Println("got:", line)
}

// Writing to stderr (for errors and diagnostics — don't mix with stdout)
fmt.Fprintln(os.Stderr, "error: something went wrong")

// Checking if stdin is a terminal or a pipe
info, _ := os.Stdin.Stat()
if info.Mode()&os.ModeCharDevice != 0 {
    // stdin is a terminal — interactive mode
} else {
    // stdin is a pipe or file — read from it
}
```

#### Environment variables
```go
// Read
val := os.Getenv("DATABASE_URL")  // "" if not set — can't distinguish missing from empty

// Read with existence check
val, ok := os.LookupEnv("DATABASE_URL")
if !ok {
    return errors.New("DATABASE_URL not set")
}

// Set and unset (affects current process and children)
os.Setenv("APP_ENV", "production")
os.Unsetenv("SECRET_KEY")

// Get all environment variables as []string ("KEY=VALUE" format)
for _, env := range os.Environ() {
    parts := strings.SplitN(env, "=", 2)
    key, value := parts[0], parts[1]
    _ = key; _ = value
}

// Expand environment variables in a string
// ${VAR} or $VAR syntax
expanded := os.ExpandEnv("connecting to $DATABASE_URL")

// Custom expansion function
result := os.Expand("hello ${NAME}", func(key string) string {
    if key == "NAME" { return "Alice" }
    return ""
})
```

#### Process and signals
```go
// Current process info
pid := os.Getpid()    // current process ID
ppid := os.Getppid()  // parent process ID

// Exit — runs deferred functions? NO. os.Exit does not run deferred functions.
// Use it only in main() after cleanup is done.
os.Exit(0)   // success
os.Exit(1)   // failure

// Signal handling — the correct way for graceful shutdown
quit := make(chan os.Signal, 1)  // buffered — don't miss the signal
signal.Notify(quit,
    os.Interrupt,    // Ctrl+C
    syscall.SIGTERM, // kill / docker stop / kubernetes pod termination
)

go func() {
    <-quit
    log.Println("shutting down...")
    // do cleanup
    os.Exit(0)
}()
```

#### `fs.FS` — the virtual filesystem abstraction (Go 1.16+)

`fs.FS` is a read-only filesystem interface that decouples code from the actual storage mechanism. A function accepting `fs.FS` works with real disk files, embedded files (`//go:embed`), zip archives, or a test mock equally well.

```go
// fs.FS interface — just one method
type FS interface {
    Open(name string) (File, error)
}

// Using embedded files — package-level variable populated at compile time
//go:embed templates/*
var templateFS embed.FS

//go:embed static
var staticFS embed.FS

// Both implement fs.FS — pass directly to http.FileServer
http.Handle("/static/", http.FileServer(http.FS(staticFS)))

// Read a file from an embedded FS
data, err := templateFS.ReadFile("templates/index.html")

// Walk an embedded FS
fs.WalkDir(templateFS, ".", func(path string, d fs.DirEntry, err error) error {
    fmt.Println(path)
    return nil
})

// Write functions that accept fs.FS — testable without real files
func loadConfig(fsys fs.FS, name string) (*Config, error) {
    f, err := fsys.Open(name)
    if err != nil {
        return nil, err
    }
    defer f.Close()
    var cfg Config
    return &cfg, json.NewDecoder(f).Decode(&cfg)
}

// In production
loadConfig(os.DirFS("/etc/myapp"), "config.json")

// In tests — no real files needed
loadConfig(fstest.MapFS{
    "config.json": &fstest.MapFile{Data: []byte(`{"port": 8080}`)},
}, "config.json")
```

### Real Patterns — How All Three Work Together

#### Pattern 1: Processing a large log file efficiently
```go
// Wrong — loads entire file into memory
data, _ := os.ReadFile("huge.log")
lines := strings.Split(string(data), "\n")

// Right — streams line by line, constant memory usage regardless of file size
func processLogFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return fmt.Errorf("opening log file: %w", err)
    }
    defer f.Close()

    // bufio.Scanner gives us line-by-line reading with buffering
    scanner := bufio.NewScanner(f)
    scanner.Buffer(make([]byte, 0, 64*1024), 1024*1024)  // up to 1MB lines

    lineNum := 0
    for scanner.Scan() {
        lineNum++
        line := scanner.Text()
        if err := processLine(lineNum, line); err != nil {
            return fmt.Errorf("line %d: %w", lineNum, err)
        }
    }
    return scanner.Err()
}
```

#### Pattern 2: Atomic file write — never corrupt on crash
```go
// Wrong — if the process crashes mid-write, the file is partially written
func saveConfig(path string, cfg Config) error {
    f, _ := os.Create(path)
    return json.NewEncoder(f).Encode(cfg)
}

// Right — write to temp file, rename atomically
// On Linux, rename is atomic — readers always see either old or new, never partial
func saveConfig(path string, cfg Config) error {
    // Create temp file in same directory (same filesystem = rename works)
    dir := filepath.Dir(path)
    tmp, err := os.CreateTemp(dir, ".config-*.tmp")
    if err != nil {
        return fmt.Errorf("creating temp file: %w", err)
    }
    tmpPath := tmp.Name()

    // Clean up temp file if anything goes wrong
    defer func() {
        tmp.Close()
        os.Remove(tmpPath)  // no-op if rename succeeded
    }()

    // Buffered write to temp file
    bw := bufio.NewWriter(tmp)
    if err := json.NewEncoder(bw).Encode(cfg); err != nil {
        return fmt.Errorf("encoding config: %w", err)
    }
    if err := bw.Flush(); err != nil {
        return fmt.Errorf("flushing: %w", err)
    }
    if err := tmp.Sync(); err != nil {  // fsync — flush OS buffer to disk
        return fmt.Errorf("syncing: %w", err)
    }
    if err := tmp.Close(); err != nil {
        return fmt.Errorf("closing temp: %w", err)
    }

    // Atomic rename — this is the moment the new config becomes visible
    return os.Rename(tmpPath, path)
}
```

#### Pattern 3: Streaming HTTP response to file
```go
// Download a large file without buffering it entirely in memory
func downloadFile(ctx context.Context, url, destPath string) error {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return fmt.Errorf("building request: %w", err)
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return fmt.Errorf("executing request: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("unexpected status: %d", resp.StatusCode)
    }

    f, err := os.Create(destPath)
    if err != nil {
        return fmt.Errorf("creating destination: %w", err)
    }
    defer f.Close()

    // bufio.Writer reduces syscalls from the os.File side
    bw := bufio.NewWriterSize(f, 256*1024)  // 256KB buffer

    // io.Copy streams from HTTP response body directly to disk
    // On Linux this may use splice() syscall — kernel-to-kernel transfer
    if _, err := io.Copy(bw, resp.Body); err != nil {
        return fmt.Errorf("downloading: %w", err)
    }

    return bw.Flush()
}
```

#### Pattern 4: Pipeline composition

This is the Unix philosophy in code — connect components through `io.Reader`/`io.Writer` interfaces into a pipeline where data flows from source through transforms to destination:

```go
func compressAndEncrypt(src io.Reader, dst io.Writer, key []byte) error {
    // Build the pipeline right-to-left (each wraps the next)
    // Data flow: src → gzip → AES-GCM encrypt → dst

    encWriter, err := newAESWriter(dst, key)   // encrypts, writes to dst
    if err != nil {
        return err
    }

    gzWriter := gzip.NewWriter(encWriter)      // compresses, writes to encWriter
    defer gzWriter.Close()

    // io.Copy pumps data through the entire pipeline
    // src → Read → gzWriter.Write → encWriter.Write → dst.Write
    _, err = io.Copy(gzWriter, src)
    return err
}

// Each component only knows about its immediate neighbor.
// You can swap out the encryption or compression algorithm without touching the rest.
// This is the io.Reader/Writer composition model.
```

### Gotchas Summary

**`io`:**

- `Read` can return fewer bytes than requested even with no error — always loop or use helpers like `io.ReadFull`.
- `io.EOF` returned alongside `n > 0` means "these are the last bytes" — process them before handling EOF.
- `io.ReadAll` loads the entire stream into memory — never use it on untrusted or potentially large input without `io.LimitReader`.
- `io.Pipe` has no internal buffer — `Write` blocks until `Read` consumes the data. If you forget to read from a pipe, the writer goroutine hangs forever.

**`bufio`:**

- `bufio.Writer` silently holds data in memory until `Flush` — forgetting `Flush` means silent data loss. Always flush explicitly, don't rely on `defer`.
- `scanner.Bytes()` returns a slice of the internal buffer — it's only valid until the next `Scan` call. If you need to keep it, call `scanner.Text()` or copy the bytes.
- `scanner.Err()` does not return `io.EOF` — that's normal termination. Check it after the loop, not inside.
- Default scanner token size is 64KB — set a larger buffer explicitly if your input lines could be longer.

**`os`:**

- `os.Exit` does not run deferred functions — do all cleanup before calling it.
- Always `defer f.Close()` immediately after a successful `os.Open` or `os.Create` — don't wait.
- File permissions in `os.Create` and `os.OpenFile` are modified by the process's `umask` — `0666` typically becomes `0644` in practice.
- `os.Rename` is only atomic within the same filesystem — renaming across filesystems falls back to copy+delete, which is not atomic.
- Closing a file does not guarantee data is on disk — use `f.Sync()` before closing for durability.

### Summary Table

| What you want                  | How to do it                                |
| ------------------------------ | ------------------------------------------- |
| Read whole file into memory    | `os.ReadFile`                               |
| Write whole file               | `os.WriteFile`                              |
| Stream large file line by line | `os.Open` + `bufio.NewScanner`              |
| Write large file efficiently   | `os.Create` + `bufio.NewWriter` + `Flush`   |
| Copy file efficiently          | `os.Open` + `os.Create` + `io.Copy`         |
| Read exactly N bytes           | `io.ReadFull`                               |
| Limit max bytes read           | `io.LimitReader`                            |
| Peek without consuming         | `bufio.Reader.Peek`                         |
| Pipe data between goroutines   | `io.Pipe`                                   |
| Read and log simultaneously    | `io.TeeReader`                              |
| Atomic file write              | Write to temp + `os.Rename`                 |
| Testable file code             | Accept `fs.FS`, use `fstest.MapFS` in tests |
| Concatenate readers            | `io.MultiReader`                            |
| Walk directory tree            | `filepath.WalkDir`                          |
| Embedded static files          | `//go:embed` + `embed.FS`                   |

## Phase 4: Time & Scheduling
---
### The Mental Model First

Time in computing has two fundamentally different concepts that most developers conflate, and Go's `time` package makes the distinction explicit once you know to look for it.

**Wall clock time** — what a clock on the wall shows. "It is 3:45 PM on May 12, 2026." This can jump forwards or backwards.
- **NTP Synchronization:** Computers constantly check in with "Network Time Protocol" servers. If your computer clock is 2 seconds fast, the system might suddenly snap it back 2 seconds to match the world standard.
    
- **Daylight Saving Time (DST):** When the clocks "fall back" in autumn, the time literally repeats itself. At 2:00 AM, it becomes 1:00 AM again.
    
- **Leap Seconds:** Occasional one-second adjustments added to UTC to keep Earth’s rotation in sync with atomic clocks.

**Never use wall clock time to measure how long a task took.**
Imagine you start a timer at 1:59:59 AM right before a DST "fall back" change:

- **Start Time:** 1:59:59 AM
    
- **End Time:** 1:00:05 AM (after the clock resets)
    
- **The Result:** Your calculation ($End - Start$) would show that the task took **negative 59 minutes**, even though it actually only took 6 seconds.

**Monotonic clock time** — a counter that only ever increases, starting from some arbitrary point (usually when the process started or when the OS booted). It has no relationship to wall time. You can't tell what time of day it is from the monotonic clock, but you can reliably measure how much time has elapsed between two readings because it never jumps.

Go's `time.Time` values carry **both** readings simultaneously since Go 1.9. When you call `time.Now()`, you get a `Time` that holds both the wall time and a monotonic reading. When you compute a duration between two `time.Now()` calls, Go automatically uses the monotonic readings, not the wall clock. When you serialize a time, compare it to a fixed timestamp, or do anything that involves a specific point in calendar time, Go uses the wall clock reading.

This is why `time.Since(t)` is always safe for measuring elapsed time, even if the system clock gets adjusted — it uses the monotonic component.

```go
start := time.Now()          // captures both wall and monotonic
doSomething()
elapsed := time.Since(start) // uses monotonic component — accurate regardless of NTP
```

### `time.Time` — the core type

#### Creation

```go
// Current time — wall + monotonic
now := time.Now()

// Specific point in time — wall only, no monotonic component
t := time.Date(2026, time.May, 12, 15, 30, 0, 0, time.UTC)
//             year  month      day  hr   min sec nanosec  location

// Parsing from a string — wall only
t, err := time.Parse(time.RFC3339, "2026-05-12T15:30:00Z")
t, err := time.Parse("2006-01-02", "2026-05-12")

// From a Unix timestamp
t := time.Unix(1715524200, 0)        // seconds since epoch
t := time.UnixMilli(1715524200000)   // milliseconds since epoch (Go 1.17+)
t := time.UnixMicro(1715524200000000) // microseconds since epoch (Go 1.17+)
```

#### The reference time — Go's most confusing design decision
Go uses a specific reference time for format strings instead of format codes like `%Y`, `%m`, `%d`. The reference time is:

```
Mon Jan 2 15:04:05 MST 2006
```

Or in numbers: `01/02 03:04:05PM '06 -0700`

Which is: month=1, day=2, hour=3, minute=4, second=5, year=6. This is a mnemonic — 1, 2, 3, 4, 5, 6, 7 — but it only works if you memorize the reference time. In practice just keep it somewhere as a reference.

```go
// To format, write out the reference time in the layout you want
t.Format("2006-01-02")                    // "2026-05-12"
t.Format("2006-01-02 15:04:05")           // "2026-05-12 15:30:00"
t.Format("January 2, 2006")              // "May 12, 2026"
t.Format("Mon, 02 Jan 2006 15:04:05 MST") // RFC1123
t.Format(time.RFC3339)                    // "2026-05-12T15:30:00Z" — use this for APIs
t.Format(time.RFC3339Nano)                // includes nanoseconds

// Parse uses the same reference time as the layout
t, err := time.Parse("2006-01-02", "2026-05-12")
t, err := time.Parse("01/02/2006", "05/12/2026")  // US date format
```

The most common mistake: using the wrong numbers in a format string. Writing `"2006-01-02"` is correct. Writing `"YYYY-MM-DD"` or `"2024-12-31"` as a format string is wrong — those are values, not format codes.

#### Accessing components

```go
t := time.Now()

t.Year()        // int — 2026
t.Month()       // time.Month — time.May (which is 5)
t.Day()         // int — 12
t.Hour()        // int — 0-23
t.Minute()      // int — 0-59
t.Second()      // int — 0-59
t.Nanosecond()  // int — 0-999999999
t.Weekday()     // time.Weekday — time.Monday etc.
t.YearDay()     // int — 1-366, day of year
t.ISOWeek()     // (year, week int) — ISO 8601 week number

// Unix timestamps
t.Unix()        // int64 — seconds since Jan 1, 1970 UTC
t.UnixMilli()   // int64 — milliseconds since epoch (Go 1.17+)
t.UnixMicro()   // int64 — microseconds since epoch (Go 1.17+)
t.UnixNano()    // int64 — nanoseconds since epoch (overflows in year 2262)

// Decompose into components at once
year, month, day := t.Date()
hour, min, sec := t.Clock()
```

#### Comparison and arithmetic
```go
a := time.Now()
b := a.Add(5 * time.Minute)

// Comparison — uses wall clock component
a.Before(b)  // true
a.After(b)   // false
a.Equal(b)   // false — same instant in time regardless of timezone

// Duration between two times
d := b.Sub(a)  // time.Duration — 5m0s
// For "time since X", prefer time.Since(a) over time.Now().Sub(a) — same thing, cleaner

// Add a duration
t2 := t.Add(24 * time.Hour)
t2 := t.Add(-30 * time.Minute)  // subtract by adding negative

// AddDate — calendar arithmetic (handles month lengths, leap years)
nextMonth := t.AddDate(0, 1, 0)   // add 1 month
nextYear  := t.AddDate(1, 0, 0)   // add 1 year
tomorrow  := t.AddDate(0, 0, 1)   // add 1 day
```

**Why `AddDate` exists and when you must use it:**
`24 * time.Hour` is always exactly 86,400 seconds. But "add one day in calendar terms" is not always 86,400 seconds — on days when clocks spring forward (23 hours) or fall back (25 hours), a calendar day is a different number of seconds. If you're working with dates in a specific timezone where DST applies and you want "same time tomorrow", use `AddDate(0, 0, 1)`, not `Add(24 * time.Hour)`.

```go
// In a timezone with DST:
// DST spring-forward night: Add(24h) gives 1:00 AM instead of 2:00 AM
// Use AddDate for calendar-correct "next day"
loc, _ := time.LoadLocation("America/New_York")
t := time.Date(2026, time.March, 8, 10, 0, 0, 0, loc)  // day before spring forward
nextDay := t.AddDate(0, 0, 1)  // correct — 10:00 AM next day
nextDay2 := t.Add(24 * time.Hour) // wrong — 11:00 AM next day (lost an hour)
```

### `time.Duration`

`Duration` is just an `int64` counting nanoseconds. This means it's a plain number you can do arithmetic on.
```go
type Duration int64

// Named constants for common durations
time.Nanosecond
time.Microsecond  // 1000 ns
time.Millisecond  // 1000 µs
time.Second       // 1000 ms
time.Minute       // 60 s
time.Hour         // 60 min

// Building durations — multiply the constant
5 * time.Second
100 * time.Millisecond
2*time.Hour + 30*time.Minute

// Decomposing a duration
d := 2*time.Hour + 37*time.Minute + 45*time.Second
d.Hours()        // float64 — 2.629...
d.Minutes()      // float64 — 157.75
d.Seconds()      // float64 — 9465.0
d.Milliseconds() // int64 — 9465000
d.Nanoseconds()  // int64 — 9465000000000
d.String()       // "2h37m45s"

// Truncate and Round
d.Truncate(time.Minute)  // 2h37m0s — floor to minute boundary
d.Round(time.Minute)     // 2h38m0s — round to nearest minute

// Parse a duration from string
d, err := time.ParseDuration("2h30m")
d, err := time.ParseDuration("100ms")
d, err := time.ParseDuration("1.5s")
```

**The integer multiplication gotcha:**
```go
// WRONG — 1000 is an untyped int, multiplied before converting to Duration
// 1000 * time.Millisecond works ONLY because of untyped constant rules
n := 1000
d := n * time.Millisecond  // COMPILE ERROR — can't multiply int and Duration

// CORRECT
d := time.Duration(n) * time.Millisecond
```

### Timezones and Locations

This is where most production bugs live.

#### The `time.Location` type

A `Location` represents a timezone, including all its historical DST rules. It is not just a UTC offset — it's a full timezone database entry.
```go
// UTC — no DST, always safe for storage and transmission
time.UTC

// Local — the timezone of the machine running the code
// DANGER: different on dev machine vs production server
time.Local

// Named timezone — the correct way for user-facing times
loc, err := time.LoadLocation("America/New_York")
loc, err := time.LoadLocation("Asia/Kolkata")    // IST — UTC+5:30
loc, err := time.LoadLocation("Europe/London")
loc, err := time.LoadLocation("UTC")

// Fixed offset — when you know the offset but not the timezone name
// Does NOT handle DST
ist := time.FixedZone("IST", 5*60*60 + 30*60)  // UTC+5:30
```

`time.LoadLocation` reads from the timezone database embedded in Go's binary (Go 1.15+ embeds it) or from the OS timezone database. On minimal Docker images (like `scratch` or `alpine` without `tzdata`), `LoadLocation` will fail with "unknown time zone" because the OS database is absent. Embed it:

```go
import _ "time/tzdata"  // blank import embeds the full tz database in your binary
```
This adds ~450KB to your binary but makes timezone loading work anywhere.

#### Converting between timezones
```go
t := time.Now().UTC()  // store everything in UTC

// Convert to user's timezone for display only
loc, _ := time.LoadLocation("Asia/Kolkata")
localTime := t.In(loc)
// t and localTime represent the SAME instant — just displayed differently

fmt.Println(t.UTC().Equal(localTime))  // true — same moment

// Extract just the date in a specific timezone
year, month, day := t.In(loc).Date()
```

#### The golden rule of timezone handling in backends

**Store in UTC. Display in local. Never store local time without the timezone.**

```go
// In your DB schema: use TIMESTAMPTZ in PostgreSQL (stores UTC internally)
// In your Go code: always work with UTC times in business logic

type Event struct {
    ID        int
    StartsAt  time.Time  // always UTC internally
    UserTZ    string     // "America/New_York" — store the IANA name, not "+05:30"
}

// When displaying to user
loc, err := time.LoadLocation(event.UserTZ)
displayTime := event.StartsAt.In(loc).Format("Mon Jan 2, 3:04 PM MST")
```

#### Parsing times with timezone information

```go
// time.Parse uses the layout's timezone literally — NO location conversion
t, _ := time.Parse(time.RFC3339, "2026-05-12T10:00:00+05:30")
// t is correct — it represents 10:00 IST = 04:30 UTC

// time.ParseInLocation parses and interprets the result in the given location
loc, _ := time.LoadLocation("Asia/Kolkata")
t, _ := time.ParseInLocation("2006-01-02 15:04:05", "2026-05-12 10:00:00", loc)
// Assumes the string is in IST, converts to that zone
```

### Timers, Tickers, and Scheduling

#### `time.Sleep` — pause the current goroutine
```go
time.Sleep(100 * time.Millisecond)
// The goroutine is parked — not spinning, not consuming CPU
// Other goroutines run normally during the sleep
```

#### `time.After` — one-shot future channel
```go
// Returns a channel that receives the current time after d
ch := time.After(5 * time.Second)
<-ch  // blocks for 5 seconds

// Most commonly used in select for timeouts
select {
case result := <-work:
    process(result)
case <-time.After(5 * time.Second):
    return errors.New("timed out")
}
```

**The leak problem with `time.After`:** every call to `time.After` allocates a `Timer` internally. The underlying timer is not garbage collected until it fires, even if the `select` case is never chosen (because another case was selected first). In a high-throughput loop, this creates a steady stream of timer allocations that only get cleaned up when they fire.

```go
// LEAKS in tight loops or frequently called functions
for {
    select {
    case msg := <-msgs:
        handle(msg)
    case <-time.After(30 * time.Second):  // new Timer allocated every iteration
        checkHealth()
    }
}

// CORRECT — reuse the timer
timer := time.NewTimer(30 * time.Second)
defer timer.Stop()
for {
    select {
    case msg := <-msgs:
        handle(msg)
        // Reset the timer for next iteration
        if !timer.Stop() {
            select {
            case <-timer.C:
            default:
            }
        }
        timer.Reset(30 * time.Second)
    case <-timer.C:
        checkHealth()
        timer.Reset(30 * time.Second)
    }
}
```

#### `time.Timer` — one-shot, controllable
```go
timer := time.NewTimer(5 * time.Second)

// Wait for it to fire
<-timer.C

// Or cancel it before it fires
if !timer.Stop() {
    // Stop returns false if the timer has already fired or been stopped.
    // If false, the channel may have a value — drain it to avoid a stale receive
    <-timer.C
}

// Reset — reuse the timer (only after Stop + drain)
timer.Reset(10 * time.Second)
```

The `Stop` + drain pattern is subtle and important. Here's why: `timer.Stop()` returns false if the timer already fired. If it fired, its channel has a value sitting in it. If you call `Reset` without draining that value first, the next `<-timer.C` will receive the stale fire immediately, not after the new duration.

```go
// Safe Stop + Reset pattern
func resetTimer(t *time.Timer, d time.Duration) {
    if !t.Stop() {
        select {
        case <-t.C:
        default:
        }
    }
    t.Reset(d)
}
```

#### `time.Ticker` — repeating timer

```go
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()  // always stop — prevents goroutine leak if ticker is abandoned

for {
    select {
    case t := <-ticker.C:
        fmt.Println("tick at", t)
    case <-done:
        return
    }
}
```

**Ticker is not a precise metronome.** The tick interval is the minimum time between ticks — if your handler takes longer than the interval, ticks pile up. But the channel is only buffered by 1, so missed ticks are dropped silently.

```go
// If your work might take longer than the interval, handle it explicitly
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()

for range ticker.C {
    // If this work takes 3 seconds, the next 2 ticks are dropped
    // You won't get a backlog of ticks — they're discarded
    doPeriodicWork()
}
```

If you need guaranteed execution intervals regardless of work duration (like a cron), you need an external job scheduler, not a ticker.

#### `time.AfterFunc` — run a function in a goroutine after a delay
```go
// Runs f in a new goroutine after d — non-blocking
timer := time.AfterFunc(5*time.Second, func() {
    fmt.Println("executed after 5s")
})

// Cancel before it fires
timer.Stop()
```

### Measuring Elapsed Time
```go
// For benchmarking/profiling
start := time.Now()
doWork()
elapsed := time.Since(start)
fmt.Printf("took %v\n", elapsed)  // "took 245.123ms"

// For timing with sub-millisecond precision
start := time.Now()
doWork()
ns := time.Since(start).Nanoseconds()

// Comparing to a threshold
if time.Since(start) > 2*time.Second {
    log.Warn("slow operation")
}
```

For extremely tight benchmarks where even `time.Now()` overhead matters, use `testing.B` — it handles the measurement correctly. For application-level timing (request duration, operation latency), `time.Since` is correct and appropriate.

### Truncate and Round — aligning to boundaries
```go
t := time.Date(2026, 5, 12, 15, 37, 45, 123456789, time.UTC)

// Truncate — floor to the nearest multiple of d (measured from zero time)
t.Truncate(time.Hour)          // 2026-05-12 15:00:00
t.Truncate(time.Minute)        // 2026-05-12 15:37:00
t.Truncate(24 * time.Hour)     // 2026-05-12 00:00:00 UTC — start of day IN UTC
                               // DANGER: not midnight in local time if not UTC

// Round — nearest multiple of d
t.Round(time.Hour)             // 2026-05-12 16:00:00 (rounds up — 37m > 30m)
t.Round(time.Minute)           // 2026-05-12 15:38:00 (rounds up — 45s > 30s)
```

**The Truncate midnight gotcha:** `t.Truncate(24 * time.Hour)` gives midnight UTC, not midnight in whatever timezone `t` is in. For "start of day in a specific timezone":

```go
func startOfDay(t time.Time, loc *time.Location) time.Time {
    y, m, d := t.In(loc).Date()
    return time.Date(y, m, d, 0, 0, 0, 0, loc)
}
```

### JSON, Database, and Serialization

#### JSON

`time.Time` implements `json.Marshaler` and `json.Unmarshaler` using RFC3339 format with nanosecond precision:
```go
type Event struct {
    Name      string    `json:"name"`
    StartsAt  time.Time `json:"starts_at"`  // marshals as "2026-05-12T15:30:00Z"
}

// Marshals to: {"name":"standup","starts_at":"2026-05-12T15:30:00Z"}
// Unmarshals RFC3339 strings back to time.Time automatically
```

If you need a different format in JSON (unix timestamp, custom format), use a custom type:

```go
type UnixTime struct{ time.Time }

func (t UnixTime) MarshalJSON() ([]byte, error) {
    return []byte(strconv.FormatInt(t.Unix(), 10)), nil
}

func (t *UnixTime) UnmarshalJSON(data []byte) error {
    sec, err := strconv.ParseInt(string(data), 10, 64)
    if err != nil {
        return err
    }
    t.Time = time.Unix(sec, 0).UTC()
    return nil
}
```

#### Database (PostgreSQL with `database/sql`)
```go
// time.Time scans directly from TIMESTAMPTZ columns
var t time.Time
err := row.Scan(&t)
// t is in UTC if column is TIMESTAMPTZ — PostgreSQL normalizes to UTC

// Inserting
_, err = db.Exec("INSERT INTO events (starts_at) VALUES ($1)", time.Now().UTC())

// Nullable timestamps — use *time.Time or sql.NullTime
type Event struct {
    CompletedAt *time.Time  // nil if not completed
}

var completedAt sql.NullTime
err := row.Scan(&completedAt)
if completedAt.Valid {
    use(completedAt.Time)
}
```

**Always pass UTC to the database.** PostgreSQL `TIMESTAMPTZ` stores UTC internally regardless of what you send, but `TIMESTAMP` (without zone) stores exactly what you give it. If you store a local time in a `TIMESTAMP` column and the server's timezone changes, your data is wrong.

### The Production Gotchas

#### 1. `time.Time` zero value is not what you expect

```go
var t time.Time
// t is January 1, year 1, 00:00:00 UTC
// NOT Unix epoch (Jan 1, 1970)
// NOT nil

t.IsZero()  // true — use this to check for unset times
t.Unix()    // -62135596800 — very negative number
```

Always use `.IsZero()` to check if a `time.Time` was set, never compare to a literal.

#### 2. Copying `time.Time` containing monotonic reading loses it

```go
t := time.Now()   // has monotonic reading
b, _ := json.Marshal(t)
json.Unmarshal(b, &t)
// t no longer has a monotonic reading — JSON only stores wall time
// time.Since(t) still works but now uses wall clock subtraction
```

Times deserialized from JSON, parsed from strings, or loaded from databases have no monotonic component. This is fine for measuring time since a stored timestamp — just be aware.

#### 3. `time.Local` is the machine's timezone

```go
// On your dev machine: Asia/Kolkata
// On your Linux prod server: UTC
// time.Local is different — behaviour differs between environments

t := time.Now()             // uses Local
t.Format("2006-01-02")     // gives different dates depending on where code runs

// Fix: always be explicit
t.UTC().Format("2006-01-02")
t.In(userLoc).Format("2006-01-02")
```

#### 4. Month is 1-based but Weekday is 0-based

```go
t.Month()   // time.January = 1, time.December = 12
t.Weekday() // time.Sunday = 0, time.Saturday = 6
```

#### 5. `time.Sleep` is not exact

The OS scheduler doesn't guarantee wakeup at exactly the requested time. In practice, sleeps are accurate to within ~1-5ms on Linux, but under heavy load can be much longer. Don't rely on sleep timing for anything that needs to be precise.

#### 6. Timer channels are buffered by 1

```go
timer := time.NewTimer(1 * time.Second)
timer.Stop()
// The timer fired before Stop — timer.C has a value
// If you don't drain it, next <-timer.C fires immediately
```

#### 7. Comparing times across timezones
```go
utc := time.Date(2026, 5, 12, 10, 0, 0, 0, time.UTC)
ist, _ := time.LoadLocation("Asia/Kolkata")
local := time.Date(2026, 5, 12, 15, 30, 0, 0, ist)  // 10:00 UTC = 15:30 IST

utc.Equal(local)  // TRUE — Equal compares the instant, not the representation
utc == local      // FALSE — == compares the struct including Location
```

**Always use `.Equal()` to compare times, never `==`.** The `==` operator compares the full struct including the Location field — two times representing the exact same instant in different timezones are `==` false but `.Equal()` true.

#### 8. Duration arithmetic overflow

`time.Duration` is `int64` nanoseconds. Maximum representable duration is ~292 years. In practice the issue is computing very large durations:
```go
// This overflows int64
d := time.Duration(math.MaxInt64 + 1)

// Storing "time since epoch" as Duration is wrong for historical dates
// time.Since(time.Unix(0, 0)) works fine — it's ~56 years, well within range
// But time.Since(time.Date(1700, ...)) may behave oddly
```

### Common Patterns in Production

#### Retry with timeout
```go
func withRetry(ctx context.Context, maxAttempts int, fn func() error) error {
    var err error
    for attempt := 0; attempt < maxAttempts; attempt++ {
        if ctx.Err() != nil {
            return ctx.Err()
        }
        if err = fn(); err == nil {
            return nil
        }
        // Exponential backoff with jitter
        backoff := time.Duration(1<<attempt) * 100 * time.Millisecond
        jitter := time.Duration(rand.Int63n(int64(backoff) / 2))
        wait := backoff + jitter

        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(wait):
        }
    }
    return fmt.Errorf("all %d attempts failed: %w", maxAttempts, err)
}
```

#### Rate limiting with a ticker
```go
func processWithRateLimit(ctx context.Context, items []Item, perSecond int) error {
    ticker := time.NewTicker(time.Second / time.Duration(perSecond))
    defer ticker.Stop()

    for _, item := range items {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if err := process(item); err != nil {
                return err
            }
        }
    }
    return nil
}
```

#### Periodic background work
```go
func startHealthChecker(ctx context.Context, interval time.Duration) {
    go func() {
        // Run once immediately, then on interval
        checkHealth()

        ticker := time.NewTicker(interval)
        defer ticker.Stop()

        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                checkHealth()
            }
        }
    }()
}
```

#### Cache with TTL
```go
type entry struct {
    value     any
    expiresAt time.Time
}

type TTLCache struct {
    mu      sync.RWMutex
    entries map[string]entry
}

func (c *TTLCache) Set(key string, value any, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.entries[key] = entry{
        value:     value,
        expiresAt: time.Now().Add(ttl),  // wall clock — correct here, we want calendar expiry
    }
}

func (c *TTLCache) Get(key string) (any, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    e, ok := c.entries[key]
    if !ok || time.Now().After(e.expiresAt) {
        return nil, false
    }
    return e.value, true
}
```

### Summary

|Concept|Key Point|
|---|---|
|Wall vs monotonic|`time.Now()` captures both. Duration measurement uses monotonic — safe from NTP jumps.|
|Reference time|`2006-01-02 15:04:05` — memorize it. Wrong numbers silently produce wrong output.|
|`Equal` vs `==`|Always use `.Equal()` for time comparison. `==` compares Location too.|
|Timezones|Store UTC. Display local. Import `time/tzdata` in Docker images.|
|`AddDate` vs `Add`|Calendar days → `AddDate`. Fixed duration → `Add`. DST zones make these different.|
|`time.After` leak|Allocates a timer that lives until it fires. Reuse `time.NewTimer` in loops.|
|Timer Stop + drain|`Stop` returns false if already fired. Drain the channel before `Reset`.|
|Zero value|`time.Time{}` is year 1, not epoch. Use `.IsZero()` to check.|
|`Truncate` midnight|Truncates to UTC midnight. Use `time.Date` for timezone-correct start of day.|
|DB storage|Use `TIMESTAMPTZ`. Always send UTC. Use `sql.NullTime` for nullable.|
|`time.Local` danger|Different on dev vs prod. Be explicit with `time.UTC` or a named location.|

## `database/sql`
---
`database/sql` is not a database driver. It is a **generic abstraction layer** that sits between your Go code and any relational database. The actual communication with PostgreSQL, MySQL, SQLite — that happens in a separate driver package. `database/sql` provides the connection pool, the transaction model, the query interface, and the scanning machinery. The driver provides the wire protocol implementation.

This separation is the key architectural insight. Your code talks to `database/sql` interfaces. The driver is registered once at startup and then stays invisible. This is why you can switch databases by changing one import and one connection string, leaving all your query code untouched — as long as your SQL is compatible.

```
Your Code
    ↓
database/sql     — pool management, transactions, scanning, prepared statements
    ↓
driver.Driver    — interface: Open(dsn) (Conn, error)
    ↓
database driver  — e.g. lib/pq, pgx/v5, go-sqlite3
    ↓
Database server  — actual TCP/Unix socket communication
```

The driver registers itself with `database/sql` via an `init()` function. That is why driver imports are blank imports — you import for the side effect of registration:
```go
import (
    "database/sql"
    _ "github.com/lib/pq"           // registers "postgres" driver
    _ "github.com/go-sql-driver/mysql"  // registers "mysql" driver
)

db, err := sql.Open("postgres", dsn)  // "postgres" matches the registered name
```

### `sql.Open` vs actual connection

This is the first thing most developers misunderstand. `sql.Open` does **not** open a connection to the database. It validates the driver name and DSN format, then returns a `*sql.DB` that represents a pool. No network activity happens.

```go
// This succeeds even if the database is completely unreachable
db, err := sql.Open("postgres", "postgres://localhost:5432/mydb?sslmode=disable")
if err != nil {
    // This only errors on invalid driver name or malformed DSN
    log.Fatal(err)
}

// This actually tries to connect — use this to verify connectivity at startup
if err := db.Ping(); err != nil {
    log.Fatalf("cannot reach database: %v", err)
}
```

The more explicit Go 1.15+ approach:

```go
// PingContext is better — respects context cancellation and deadlines
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
if err := db.PingContext(ctx); err != nil {
    log.Fatalf("cannot reach database: %v", err)
}
```

### `*sql.DB` — the Connection Pool

`*sql.DB` is not a single connection. It is a **thread-safe connection pool** that manages a set of live connections to the database. When you call `db.Query`, it borrows a connection from the pool, uses it, and returns it when done. You never manage individual connections directly in normal code.

```go
db, _ := sql.Open("postgres", dsn)

// Maximum number of open connections (idle + in-use)
// Default: 0 = unlimited — dangerous in production
// A connection to PostgreSQL costs ~5-10MB on the server side
// Set this based on: max_connections in postgresql.conf minus connections for other clients
db.SetMaxOpenConns(25)

// Maximum number of idle connections kept in the pool
// Default: 2
// Idle connections are ready to use immediately — no TCP handshake needed
// Setting this lower than MaxOpenConns means connections are closed after use if pool is full
// Setting equal to MaxOpenConns means all connections stay alive
db.SetMaxIdleConns(25)

// Maximum time a connection can be reused
// Default: 0 = forever
// Important for: load balancers that kill long-lived connections,
// PostgreSQL config changes, certificate rotation
db.SetConnMaxLifetime(5 * time.Minute)

// Maximum time a connection can sit idle in the pool
// Default: 0 = forever
// Prevents accumulation of stale connections
db.SetConnMaxIdleTime(5 * time.Minute)
```

**Why these settings matter in production:**
PostgreSQL has a hard limit on connections (`max_connections`, default 100). Each connection is a forked OS process on the PostgreSQL side consuming memory. If your Go service creates unlimited connections under load, you will exhaust PostgreSQL's connection limit and all queries start failing with "too many connections". With multiple instances of your service running (or multiple services sharing a database), you need to partition the connection budget carefully. A production setup often puts PgBouncer in front of PostgreSQL to multiplex many application connections onto fewer database connections.

```GO
// Reasonable defaults for a single service connecting to PostgreSQL
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(25)          // same as max to avoid teardown/rebuild overhead
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(10 * time.Minute)
```

#### Pool lifecycle — what actually happens
```
Request arrives
    ↓
db.QueryContext(ctx, ...) called
    ↓
pool has idle connection?
    YES → borrow it, execute query, return to pool
    NO  → open connections < MaxOpenConns?
            YES → open new connection, execute query, return to pool
            NO  → wait in queue until a connection becomes available
                  or context deadline exceeded → ErrConnDone
```

If `MaxOpenConns` is reached and all connections are busy, your query waits. If the context times out before a connection is available, you get a context error. This is why every query call should have a context with a deadline.

### Querying — the three methods

#### `db.QueryContext` — multiple rows
```go
func getUsers(ctx context.Context, db *sql.DB, active bool) ([]User, error) {
    rows, err := db.QueryContext(ctx,
        "SELECT id, name, email, created_at FROM users WHERE active = $1",
        active,
    )
    if err != nil {
        return nil, fmt.Errorf("querying users: %w", err)
    }
    defer rows.Close()  // ALWAYS defer Close immediately after checking err
                        // Close returns the connection to the pool
                        // If you forget, the connection leaks until GC

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt); err != nil {
            return nil, fmt.Errorf("scanning user: %w", err)
        }
        users = append(users, u)
    }

    // rows.Next() returns false on error OR on normal completion
    // Must check rows.Err() to distinguish the two
    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("iterating users: %w", err)
    }

    return users, nil
}
```

The `rows.Close()` + `rows.Err()` pattern is mandatory and often forgotten. When `rows.Next()` returns false because of an error mid-iteration, the error is stored in `rows.Err()`. If you only check the error from `QueryContext`, you miss errors that occur during row streaming.

#### `db.QueryRowContext` — exactly one row
```go
func getUserByID(ctx context.Context, db *sql.DB, id int) (*User, error) {
    var u User
    err := db.QueryRowContext(ctx,
        "SELECT id, name, email, created_at FROM users WHERE id = $1",
        id,
    ).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("user %d: %w", id, ErrNotFound)
        }
        return nil, fmt.Errorf("getting user %d: %w", id, err)
    }
    return &u, nil
}
```

`QueryRowContext` never returns an error directly — the error is deferred until `Scan`. This is by design: it makes the single-row case a one-liner. `sql.ErrNoRows` is the sentinel for "no result found" — always check for this explicitly and convert it to your domain error.

#### `db.ExecContext` — no rows returned
```go
func updateUserEmail(ctx context.Context, db *sql.DB, id int, email string) error {
    result, err := db.ExecContext(ctx,
        "UPDATE users SET email = $1, updated_at = NOW() WHERE id = $2",
        email, id,
    )
    if err != nil {
        return fmt.Errorf("updating email: %w", err)
    }

    // RowsAffected tells you how many rows were actually changed
    n, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("getting rows affected: %w", err)
    }
    if n == 0 {
        return fmt.Errorf("user %d: %w", id, ErrNotFound)
    }
    return nil
}
```

`RowsAffected` is critical for UPDATE and DELETE operations. Without checking it, you won't know if the WHERE clause matched anything. An update that affects zero rows is not a database error — it's a logic error in your application that you need to catch explicitly.

`LastInsertId` exists but is PostgreSQL-incompatible (PostgreSQL doesn't support it). For INSERT with PostgreSQL, use the `RETURNING` clause:
```go
func createUser(ctx context.Context, db *sql.DB, name, email string) (int, error) {
    var id int
    err := db.QueryRowContext(ctx,
        "INSERT INTO users (name, email, created_at) VALUES ($1, $2, NOW()) RETURNING id",
        name, email,
    ).Scan(&id)
    if err != nil {
        return 0, fmt.Errorf("creating user: %w", err)
    }
    return id, nil
}
```

### Scanning — mapping columns to Go types

`Scan` uses reflection to map database column values to Go variables. The mapping must match by position — column order in your SELECT must match argument order in Scan.

#### Nullable columns — the sql.Null* types

If a column can be NULL in the database and you scan into a non-pointer Go type, you get an error: "converting NULL to string is unsupported". You must use either a pointer or a `sql.Null*` type:

```go
// Using sql.Null* types
type User struct {
    ID        int
    Name      string
    Bio       sql.NullString   // nullable VARCHAR
    Age       sql.NullInt64    // nullable INTEGER
    Score     sql.NullFloat64  // nullable FLOAT
    Active    sql.NullBool     // nullable BOOLEAN
    DeletedAt sql.NullTime     // nullable TIMESTAMP (Go 1.15+)
}

var u User
err := row.Scan(&u.ID, &u.Name, &u.Bio, &u.Age)

// Access
if u.Bio.Valid {
    use(u.Bio.String)  // .String, .Int64, .Float64, .Bool, .Time
}

// Using pointer types — often cleaner in structs
type User struct {
    ID        int
    Name      string
    Bio       *string    // nil if NULL
    DeletedAt *time.Time // nil if NULL
}

var u User
err := row.Scan(&u.ID, &u.Name, &u.Bio, &u.DeletedAt)
// u.Bio is nil if column was NULL
```

#### Scanning into `[]byte` and `json.RawMessage`
```go
// PostgreSQL JSONB column
type Product struct {
    ID       int
    Metadata json.RawMessage  // scans raw JSON bytes
}

var p Product
err := row.Scan(&p.ID, &p.Metadata)
// p.Metadata contains the raw JSON — unmarshal as needed
var meta map[string]interface{}
json.Unmarshal(p.Metadata, &meta)

// Storing JSON
metadata, _ := json.Marshal(map[string]interface{}{"color": "red", "size": "L"})
db.ExecContext(ctx, "INSERT INTO products (metadata) VALUES ($1)", metadata)
```

#### Custom scanner — `sql.Scanner` interface

For types that don't scan natively, implement `sql.Scanner`:
```go
// Custom type that scans from a comma-separated string in DB
type Tags []string

func (t *Tags) Scan(src interface{}) error {
    switch v := src.(type) {
    case string:
        *t = strings.Split(v, ",")
        return nil
    case []byte:
        *t = strings.Split(string(v), ",")
        return nil
    case nil:
        *t = nil
        return nil
    default:
        return fmt.Errorf("cannot scan type %T into Tags", src)
    }
}

// Now Tags can be used directly in Scan
var tags Tags
row.Scan(&tags)
```

### Prepared Statements
A prepared statement is a query template that the database parses and plans once, then executes many times with different parameters. This gives two benefits: performance (no re-parsing) and safety (parameters are always escaped — no SQL injection possible).

In `database/sql`, **every parameterized query is automatically prepared behind the scenes**. When you call `db.QueryContext(ctx, "SELECT ... WHERE id = $1", id)`, the driver prepares the statement, executes it, and caches the prepared statement for reuse. You get the safety benefits automatically.

Explicit prepared statements give you control over when preparation happens:

```go
// Prepare once at service startup — fail fast if SQL is wrong
stmt, err := db.PrepareContext(ctx,
    "INSERT INTO events (user_id, action, created_at) VALUES ($1, $2, NOW())",
)
if err != nil {
    return fmt.Errorf("preparing event insert: %w", err)
}
defer stmt.Close()

// Execute many times — preparation cost paid once
for _, event := range events {
    if _, err := stmt.ExecContext(ctx, event.UserID, event.Action); err != nil {
        return fmt.Errorf("inserting event: %w", err)
    }
}
```

**Prepared statements and connection pools:** a prepared statement is prepared on a specific connection. When `database/sql` borrows a different connection from the pool to execute your prepared statement, it transparently re-prepares on that connection. This is handled automatically, but it means you can't assume a prepared statement is "zero cost" on every execution — there's occasional re-preparation overhead.

### Transactions

A transaction groups multiple operations into an atomic unit. Either all succeed and are committed, or any failure causes the entire group to be rolled back as if nothing happened.

#### Basic transaction pattern
```go
func transferFunds(ctx context.Context, db *sql.DB, fromID, toID int, amount float64) error {
    // Begin starts a transaction — borrows a connection that is held for the transaction's lifetime
    tx, err := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelReadCommitted,  // isolation level
        ReadOnly:  false,
    })
    if err != nil {
        return fmt.Errorf("beginning transaction: %w", err)
    }
    // Defer a rollback — if we return early with an error, this rolls back
    // If Commit was already called, Rollback is a no-op
    defer tx.Rollback()

    // Debit
    result, err := tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
        amount, fromID,
    )
    if err != nil {
        return fmt.Errorf("debiting account: %w", err)
    }
    if n, _ := result.RowsAffected(); n == 0 {
        return errors.New("insufficient funds or account not found")
    }

    // Credit
    if _, err := tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount, toID,
    ); err != nil {
        return fmt.Errorf("crediting account: %w", err)
    }

    // Commit — makes changes permanent
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("committing transfer: %w", err)
    }
    return nil
}
```

The `defer tx.Rollback()` pattern is the idiomatic Go approach. The logic is:

- If we reach `tx.Commit()` and it succeeds → subsequent `tx.Rollback()` is a no-op, harmless.
- If we return early due to any error → `tx.Rollback()` fires and undoes everything.
- If `tx.Commit()` itself fails → `tx.Rollback()` fires and undoes everything.

```go
sql.LevelDefault          // database default (usually ReadCommitted)
sql.LevelReadUncommitted  // dirty reads possible — almost never use
sql.LevelReadCommitted    // sees only committed data — PostgreSQL default
sql.LevelRepeatableRead   // same query returns same result within transaction
sql.LevelSerializable     // full isolation — most expensive, prevents all anomalies
```

For most CRUD operations, `ReadCommitted` (the default) is appropriate. For operations where you read data and then make decisions based on it within the same transaction, you may need `RepeatableRead` or `Serializable` to prevent phantom reads.

#### Select for update — pessimistic locking
```go
func claimJob(ctx context.Context, tx *sql.Tx, workerID int) (*Job, error) {
    var job Job
    err := tx.QueryRowContext(ctx, `
        SELECT id, payload
        FROM jobs
        WHERE status = 'pending'
        ORDER BY created_at
        LIMIT 1
        FOR UPDATE SKIP LOCKED
    `).Scan(&job.ID, &job.Payload)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, nil  // no jobs available
        }
        return nil, err
    }

    _, err = tx.ExecContext(ctx,
        "UPDATE jobs SET status = 'processing', worker_id = $1 WHERE id = $2",
        workerID, job.ID,
    )
    return &job, err
}
```

`FOR UPDATE` locks the selected rows. `SKIP LOCKED` skips rows already locked by other transactions. This pattern implements a distributed work queue at the database level — exactly what you'd use for a job processing system.

#### Savepoints — nested transaction logic

PostgreSQL supports savepoints within a transaction, allowing partial rollback:
```go
func processWithSavepoint(ctx context.Context, tx *sql.Tx) error {
    // Create a savepoint
    if _, err := tx.ExecContext(ctx, "SAVEPOINT sp1"); err != nil {
        return err
    }

    _, err := tx.ExecContext(ctx, "INSERT INTO risky_table VALUES ($1)", value)
    if err != nil {
        // Roll back to savepoint — outer transaction still alive
        tx.ExecContext(ctx, "ROLLBACK TO SAVEPOINT sp1")
        // Continue with alternative logic
        return handleFallback(ctx, tx)
    }

    tx.ExecContext(ctx, "RELEASE SAVEPOINT sp1")
    return nil
}
```

#### The transaction helper pattern

Repeating the `BeginTx` / `defer Rollback` / `Commit` boilerplate gets tiresome. Extract it:

```go
func withTx(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("beginning tx: %w", err)
    }
    defer tx.Rollback()

    if err := fn(tx); err != nil {
        return err
    }
    return tx.Commit()
}

// Usage
err := withTx(ctx, db, func(tx *sql.Tx) error {
    if err := debitAccount(ctx, tx, fromID, amount); err != nil {
        return err
    }
    return creditAccount(ctx, tx, toID, amount)
})
```

### The Repository Pattern — structuring database code

Raw `*sql.DB` calls scattered throughout your service create tight coupling and make testing hard. The repository pattern wraps database operations behind an interface:
```go
// The interface — defines what operations exist
type UserRepository interface {
    GetByID(ctx context.Context, id int) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    Create(ctx context.Context, params CreateUserParams) (*User, error)
    Update(ctx context.Context, id int, params UpdateUserParams) (*User, error)
    Delete(ctx context.Context, id int) error
    List(ctx context.Context, filter UserFilter) ([]User, error)
}

// The concrete implementation
type postgresUserRepo struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) UserRepository {
    return &postgresUserRepo{db: db}
}

func (r *postgresUserRepo) GetByID(ctx context.Context, id int) (*User, error) {
    var u User
    err := r.db.QueryRowContext(ctx,
        `SELECT id, name, email, created_at, updated_at
         FROM users WHERE id = $1 AND deleted_at IS NULL`,
        id,
    ).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt, &u.UpdatedAt)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNotFound
        }
        return nil, fmt.Errorf("userRepo.GetByID id=%d: %w", id, err)
    }
    return &u, nil
}
```

**Transactional repository — passing the transaction in:**

The challenge with repositories and transactions: sometimes you need to call two different repositories within one transaction. The cleanest approach is to accept either a `*sql.DB` or `*sql.Tx` via an interface:

```go
// Both *sql.DB and *sql.Tx implement this
type DBTX interface {
    ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error)
    QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error)
    QueryRowContext(ctx context.Context, query string, args ...interface{}) *sql.Row
    PrepareContext(ctx context.Context, query string) (*sql.Stmt, error)
}

type postgresUserRepo struct {
    db DBTX  // accepts both *sql.DB and *sql.Tx
}

// WithTx returns a new repo that uses the given transaction
func (r *postgresUserRepo) WithTx(tx *sql.Tx) *postgresUserRepo {
    return &postgresUserRepo{db: tx}
}

// Usage in service layer
func (s *Service) TransferWithAudit(ctx context.Context, from, to int, amount float64) error {
    return withTx(ctx, s.rawDB, func(tx *sql.Tx) error {
        userRepo := s.userRepo.WithTx(tx)
        auditRepo := s.auditRepo.WithTx(tx)

        if err := userRepo.DebitBalance(ctx, from, amount); err != nil {
            return err
        }
        if err := userRepo.CreditBalance(ctx, to, amount); err != nil {
            return err
        }
        return auditRepo.Log(ctx, from, to, amount)
        // All three operations in one transaction
    })
}
```

This `DBTX` interface pattern is exactly what `sqlc` generates — it's the standard approach for transactional repositories in Go.

### Bulk Operations

#### Bulk insert — `COPY` protocol

For inserting thousands of rows, the PostgreSQL `COPY` protocol is orders of magnitude faster than individual INSERTs. With `pgx`:
```go
import "github.com/jackc/pgx/v5"

func bulkInsertUsers(ctx context.Context, conn *pgx.Conn, users []User) error {
    _, err := conn.CopyFrom(
        ctx,
        pgx.Identifier{"users"},
        []string{"name", "email", "created_at"},
        pgx.CopyFromSlice(len(users), func(i int) ([]interface{}, error) {
            return []interface{}{
                users[i].Name,
                users[i].Email,
                users[i].CreatedAt,
            }, nil
        }),
    )
    return err
}
```

#### Bulk insert with standard `database/sql` — unnest trick

If you're using `database/sql` with `lib/pq`:
```go
func bulkInsertUsers(ctx context.Context, db *sql.DB, users []User) error {
    names := make([]string, len(users))
    emails := make([]string, len(users))
    times := make([]time.Time, len(users))

    for i, u := range users {
        names[i] = u.Name
        emails[i] = u.Email
        times[i] = u.CreatedAt
    }

    // unnest expands arrays into rows — single query, very fast
    _, err := db.ExecContext(ctx, `
        INSERT INTO users (name, email, created_at)
        SELECT * FROM unnest($1::text[], $2::text[], $3::timestamptz[])
    `,
        pq.Array(names),
        pq.Array(emails),
        pq.Array(times),
    )
    return err
}
```

#### IN clause with variable arguments
```go
func getUsersByIDs(ctx context.Context, db *sql.DB, ids []int) ([]User, error) {
    if len(ids) == 0 {
        return nil, nil
    }

    // Build $1,$2,$3,... placeholders
    placeholders := make([]string, len(ids))
    args := make([]interface{}, len(ids))
    for i, id := range ids {
        placeholders[i] = fmt.Sprintf("$%d", i+1)
        args[i] = id
    }

    query := fmt.Sprintf(
        "SELECT id, name, email FROM users WHERE id IN (%s)",
        strings.Join(placeholders, ","),
    )

    rows, err := db.QueryContext(ctx, query, args...)
    // ... rest of scanning
}
```

### Context and Cancellation

Every database method has a `Context` variant. Use them. Always.
```go
// What happens when context is cancelled mid-query:
// 1. database/sql sends a cancellation signal to the driver
// 2. The driver closes or cancels the in-flight query on the database side
// 3. The QueryContext / ExecContext call returns with context.Canceled or DeadlineExceeded
// 4. The connection is marked bad and not returned to pool (or cleaned up)

ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

rows, err := db.QueryContext(ctx, "SELECT * FROM large_table")
if err != nil {
    if errors.Is(err, context.DeadlineExceeded) {
        // query took too long — return 503 or retry
    }
    return err
}
```

**Cancelled context on writes:** if the context is cancelled during an INSERT or UPDATE, the operation may have already completed on the database side — Go just didn't get the confirmation back. This is important for write paths: a cancelled context doesn't mean the write didn't happen. Design write operations to be idempotent when possible.

### Error Handling and Postgres Error Codes

Database errors carry structured information. For PostgreSQL errors with `lib/pq`:
```go
import "github.com/lib/pq"

func isUniqueViolation(err error) bool {
    var pqErr *pq.Error
    if errors.As(err, &pqErr) {
        return pqErr.Code == "23505"  // unique_violation
    }
    return false
}

func isForeignKeyViolation(err error) bool {
    var pqErr *pq.Error
    if errors.As(err, &pqErr) {
        return pqErr.Code == "23503"  // foreign_key_violation
    }
    return false
}

// Usage in repository
func (r *postgresUserRepo) Create(ctx context.Context, email string) (*User, error) {
    var u User
    err := r.db.QueryRowContext(ctx,
        "INSERT INTO users (email) VALUES ($1) RETURNING id, email, created_at",
        email,
    ).Scan(&u.ID, &u.Email, &u.CreatedAt)

    if err != nil {
        if isUniqueViolation(err) {
            return nil, ErrEmailAlreadyExists
        }
        return nil, fmt.Errorf("creating user: %w", err)
    }
    return &u, nil
}
```

Common PostgreSQL error codes worth handling:

|Code|Name|When|
|---|---|---|
|23505|unique_violation|Duplicate key|
|23503|foreign_key_violation|Referenced row missing|
|23502|not_null_violation|NULL in NOT NULL column|
|40001|serialization_failure|Serializable transaction conflict|
|40P01|deadlock_detected|Deadlock|
|57014|query_canceled|Statement timeout hit|
|53300|too_many_connections|Connection limit reached|
### Observability — what to instrument
```go
type instrumentedDB struct {
    db      *sql.DB
    metrics *DBMetrics
}

func (d *instrumentedDB) QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error) {
    start := time.Now()
    rows, err := d.db.QueryContext(ctx, query, args...)
    duration := time.Since(start)

    // Record metrics
    d.metrics.QueryDuration.Observe(duration.Seconds())
    if err != nil {
        d.metrics.QueryErrors.Inc()
    }

    // Slow query log
    if duration > 100*time.Millisecond {
        log.Warn("slow query",
            zap.String("query", query),
            zap.Duration("duration", duration),
        )
    }
    return rows, err
}

// Pool stats — expose these to Prometheus
stats := db.Stats()
stats.OpenConnections   // current open connections
stats.InUse             // connections currently in use
stats.Idle              // idle connections in pool
stats.WaitCount         // total times waited for a connection
stats.WaitDuration      // total time spent waiting for connections
stats.MaxIdleClosed     // connections closed due to MaxIdleConns
stats.MaxLifetimeClosed // connections closed due to MaxConnLifetime
```

`WaitCount` and `WaitDuration` going up is the first sign your pool is undersized for your load.

### The Full Production Pattern
Here is how all the pieces fit together in a real service:

```go
package database

import (
    "context"
    "database/sql"
    "fmt"
    "time"

    _ "github.com/lib/pq"
)

type Config struct {
    DSN             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
    ConnMaxIdleTime time.Duration
}

func Connect(cfg Config) (*sql.DB, error) {
    db, err := sql.Open("postgres", cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("opening db: %w", err)
    }

    db.SetMaxOpenConns(cfg.MaxOpenConns)
    db.SetMaxIdleConns(cfg.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime)
    db.SetConnMaxIdleTime(cfg.ConnMaxIdleTime)

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := db.PingContext(ctx); err != nil {
        db.Close()
        return nil, fmt.Errorf("pinging db: %w", err)
    }

    return db, nil
}

// DBTX — both *sql.DB and *sql.Tx satisfy this
type DBTX interface {
    ExecContext(context.Context, string, ...interface{}) (sql.Result, error)
    QueryContext(context.Context, string, ...interface{}) (*sql.Rows, error)
    QueryRowContext(context.Context, string, ...interface{}) *sql.Row
}

// WithTx — generic transaction helper
func WithTx(ctx context.Context, db *sql.DB, fn func(DBTX) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("beginning tx: %w", err)
    }
    defer tx.Rollback()
    if err := fn(tx); err != nil {
        return err
    }
    return tx.Commit()
}
```

### Gotchas Summary

|Gotcha|What happens|Fix|
|---|---|---|
|`sql.Open` doesn't connect|No error even if DB is down|Call `db.PingContext` at startup|
|Forgetting `rows.Close()`|Connection never returned to pool, pool exhausted|`defer rows.Close()` immediately after err check|
|Ignoring `rows.Err()`|Silent data truncation on mid-iteration error|Always check after the loop|
|`sql.ErrNoRows` not handled|Returns internal error to caller|Explicitly check and map to domain error|
|`RowsAffected` not checked|Update/delete silently affects zero rows|Check after every UPDATE/DELETE|
|`SetMaxOpenConns` not set|Unlimited connections exhaust PostgreSQL|Always set in production|
|No context on queries|Queries run even after client disconnects|Always use `*Context` variants|
|`tx.Rollback` not deferred|Error paths leave transaction open|`defer tx.Rollback()` immediately after `BeginTx`|
|Cancelled context on writes|Write may have succeeded on DB side|Design writes to be idempotent|
|`LastInsertId` on PostgreSQL|Not supported — returns 0|Use `RETURNING id` clause instead|
|NULL columns without Null* types|Scan panics at runtime|Use `*Type` or `sql.NullString` etc.|
|`time.Local` in queries|Different timezone on dev vs prod|Always store and pass `time.UTC()`|

# `testing` 
---
### The Mental Model First

Go's testing philosophy is deliberately minimal. There is no test framework built into the language, no assertion library in the standard library, no magic dependency injection container. The `testing` package gives you exactly three things: a way to run functions as tests, a way to report failure, and a way to measure performance. Everything else — assertions, mocks, test data, HTTP faking — you either build yourself or bring in a small, focused library.

This minimalism is intentional. It pushes you toward writing tests that are just Go code, which means they are debuggable, readable, and don't require learning a framework DSL. The tradeoff is that you write slightly more boilerplate, but the tests you end up with are straightforward to reason about.

The other philosophical point: Go distinguishes between **marking a test as failed** and **stopping a test immediately**. `t.Error` and `t.Errorf` record a failure and let the test continue running — useful when you want to see all the failures in one run. `t.Fatal` and `t.Fatalf` record a failure and stop the current test function immediately — useful when subsequent steps are meaningless if an earlier one failed. Understanding when to use each is the first real skill in Go testing.

### `testing.T` — the core type
Every test function receives a `*testing.T`. This is your handle for everything.

```go
// Test functions must match this signature exactly
// Must start with Test, followed by a capital letter or underscore
func TestSomething(t *testing.T) { }
func Test_something(t *testing.T) { }  // underscore variant

// Run with:
// go test ./...                    — all packages
// go test ./internal/auth/...      — specific package tree
// go test -run TestSomething       — specific test by name
// go test -run TestUser/           — all subtests of TestUser
// go test -v                       — verbose output
// go test -count=1                 — disable test result caching
// go test -race                    — enable race detector
```

#### Failure methods
```go
func TestCalculate(t *testing.T) {
    result := add(2, 3)

    // Fail — record failure, continue running
    if result != 5 {
        t.Errorf("add(2,3) = %d, want 5", result)
    }

    // Fatal — record failure, stop this test immediately
    // Use when the rest of the test makes no sense without this passing
    got, err := parseConfig("invalid")
    if err == nil {
        t.Fatal("expected error for invalid config, got nil")
        // Lines after Fatal never execute in this test
    }

    // Logf — informational, only shown with -v or on failure
    t.Logf("intermediate value: %v", result)

    // Skip — mark test as skipped (not failed)
    if os.Getenv("INTEGRATION") == "" {
        t.Skip("skipping integration test — set INTEGRATION=1 to run")
    }
}
```

#### `t.Helper()` — critical for custom assertion functions
When a test fails inside a helper function, Go reports the failure at the line inside the helper — which is useless. `t.Helper()` tells Go to report the failure at the caller's line instead.

```go
// Without t.Helper() — failure reported inside assertEqual, not at the call site
func assertEqual(t *testing.T, got, want int) {
    if got != want {
        t.Errorf("got %d, want %d", got, want)  // points here — useless
    }
}

// With t.Helper() — failure reported at the line that called assertEqual
func assertEqual(t *testing.T, got, want int) {
    t.Helper()  // must be first line
    if got != want {
        t.Errorf("got %d, want %d", got, want)  // points to caller — useful
    }
}

func TestAdd(t *testing.T) {
    assertEqual(t, add(2, 3), 5)   // failure reported HERE when it fails
    assertEqual(t, add(0, 0), 0)
}
```

This is the single most important `testing.T` method that developers forget. Every custom assertion helper you write should start with `t.Helper()`.

### Table-Driven Tests

This is the idiomatic Go pattern for testing a function against multiple inputs. Instead of writing a separate test function for each case, you define a slice of test cases and loop over them. The benefits are: all cases are visible in one place, adding a new case is one line, and failures identify themselves by name.

```go
func TestParseAmount(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    float64
        wantErr bool
    }{
        {
            name:  "valid integer",
            input: "100",
            want:  100.0,
        },
        {
            name:  "valid decimal",
            input: "99.50",
            want:  99.50,
        },
        {
            name:  "negative value",
            input: "-10.00",
            want:  -10.0,
        },
        {
            name:    "empty string",
            input:   "",
            wantErr: true,
        },
        {
            name:    "non-numeric",
            input:   "abc",
            wantErr: true,
        },
        {
            name:    "overflow",
            input:   "999999999999999999999",
            wantErr: true,
        },
    }

    for _, tt := range tests {
        tt := tt  // capture loop variable — critical for parallel subtests
        t.Run(tt.name, func(t *testing.T) {
            got, err := parseAmount(tt.input)

            if tt.wantErr {
                if err == nil {
                    t.Fatalf("parseAmount(%q) expected error, got nil", tt.input)
                }
                return  // don't check got if we expected an error
            }

            if err != nil {
                t.Fatalf("parseAmount(%q) unexpected error: %v", tt.input, err)
            }

            if got != tt.want {
                t.Errorf("parseAmount(%q) = %v, want %v", tt.input, got, tt.want)
            }
        })
    }
}
```

#### When the test case needs richer comparison

For structs, use `reflect.DeepEqual` or the `cmp` package (`google/go-cmp`) — it gives better diffs than a bare `!=`:
```go
import "github.com/google/go-cmp/cmp"

tests := []struct {
    name  string
    input CreateUserRequest
    want  *User
}{
    {
        name:  "valid user",
        input: CreateUserRequest{Name: "Alice", Email: "a@b.com"},
        want:  &User{Name: "Alice", Email: "a@b.com", Active: true},
    },
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := createUser(tt.input)
        if err != nil {
            t.Fatal(err)
        }

        // cmp.Diff returns empty string if equal, otherwise a readable diff
        if diff := cmp.Diff(tt.want, got,
            cmpopts.IgnoreFields(User{}, "ID", "CreatedAt"),  // ignore generated fields
        ); diff != "" {
            t.Errorf("createUser() mismatch (-want +got):\n%s", diff)
        }
    })
}
```

The `cmp.Diff` output looks like a git diff — lines prefixed with `-` are what you wanted, `+` is what you got. Far more useful than "got X want Y" for complex structs.

### Subtests and Parallel Execution
`t.Run` creates a subtest with its own `*testing.T`. Subtests can be run in parallel, which dramatically speeds up test suites that do I/O.

```go
func TestUserService(t *testing.T) {
    // Setup shared state — runs once before any subtest
    db := setupTestDB(t)
    svc := NewUserService(db)

    t.Run("Create", func(t *testing.T) {
        t.Parallel()  // this subtest runs concurrently with other parallel subtests
        user, err := svc.Create(context.Background(), "Alice", "alice@example.com")
        // ...
    })

    t.Run("GetByEmail", func(t *testing.T) {
        t.Parallel()
        user, err := svc.GetByEmail(context.Background(), "alice@example.com")
        // ...
    })

    t.Run("Delete", func(t *testing.T) {
        t.Parallel()
        err := svc.Delete(context.Background(), 1)
        // ...
    })
}
```

**The loop variable capture issue** — this is a classic Go bug in parallel subtests:

```go
// BROKEN — all subtests see the last value of tt
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // By the time this runs, tt has been overwritten by loop
        doSomething(tt.input)  // all subtests use same tt
    })
}

// FIXED — capture the loop variable
for _, tt := range tests {
    tt := tt  // new variable scoped to this iteration
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        doSomething(tt.input)  // each subtest has its own tt
    })
}

// Go 1.22+ — loop variable capture fixed by default, no need for tt := tt
```

#### Parallel test levels
```go
func TestIntegration(t *testing.T) {
    // t.Parallel() here means this test runs parallel with OTHER top-level tests
    t.Parallel()

    t.Run("subtest A", func(t *testing.T) {
        t.Parallel()  // runs parallel with subtest B and C
        // ...
    })

    t.Run("subtest B", func(t *testing.T) {
        t.Parallel()
        // ...
    })
}
```

The parent test function blocks at `t.Run` subtests until they all complete — but parallel subtests run concurrently with each other, not with the parent.

### TestMain — package-level setup and teardown
When all tests in a package need shared setup (start a test database, spin up a test server), `TestMain` is the entry point:

```go
// TestMain runs instead of the tests directly
// You control when the tests actually run via m.Run()
func TestMain(m *testing.M) {
    // Setup — runs once before all tests in this package
    db, err := setupTestDatabase()
    if err != nil {
        log.Fatalf("setup failed: %v", err)
    }
    testDB = db  // package-level variable — tests access this

    // Run all tests
    code := m.Run()

    // Teardown — runs once after all tests
    testDB.Close()
    teardownTestDatabase()

    // Must call os.Exit with m.Run()'s return code
    os.Exit(code)
}

var testDB *sql.DB
```
**Important:** `defer` does not work reliably in `TestMain` because `os.Exit` bypasses deferred functions. Always do explicit cleanup before `os.Exit(code)`.

### Test Helpers and Fixtures
#### `t.Cleanup` — register cleanup that runs when test ends
```go
func setupUser(t *testing.T, db *sql.DB) *User {
    t.Helper()

    user, err := db.CreateUser(context.Background(), "test@example.com")
    if err != nil {
        t.Fatalf("creating test user: %v", err)
    }

    // Cleanup registered against THIS test's lifecycle
    // Runs when the test that called setupUser ends — even if test fails
    t.Cleanup(func() {
        db.DeleteUser(context.Background(), user.ID)
    })

    return user
}

func TestSomething(t *testing.T) {
    user := setupUser(t, testDB)  // user is cleaned up when TestSomething ends
    // ... test using user
}
```

`t.Cleanup` is the modern replacement for `defer` in tests. The advantage: cleanup functions registered on `t` run in LIFO order when the test ends, regardless of how it ends (pass, fail, skip). Multiple cleanups compose correctly.

#### Builder pattern for test data

Hardcoded test fixtures become brittle. A builder pattern gives you sensible defaults with easy overrides:
```go
type UserBuilder struct {
    user User
}

func NewUserBuilder() *UserBuilder {
    return &UserBuilder{
        user: User{
            Name:      "Test User",
            Email:     fmt.Sprintf("test-%d@example.com", time.Now().UnixNano()),
            Active:    true,
            CreatedAt: time.Now().UTC(),
        },
    }
}

func (b *UserBuilder) WithName(name string) *UserBuilder {
    b.user.Name = name
    return b
}

func (b *UserBuilder) WithEmail(email string) *UserBuilder {
    b.user.Email = email
    return b
}

func (b *UserBuilder) Inactive() *UserBuilder {
    b.user.Active = false
    return b
}

func (b *UserBuilder) Build() User {
    return b.user
}

func (b *UserBuilder) Insert(t *testing.T, db *sql.DB) User {
    t.Helper()
    created, err := insertUser(db, b.user)
    if err != nil {
        t.Fatalf("inserting test user: %v", err)
    }
    t.Cleanup(func() { deleteUser(db, created.ID) })
    return created
}

// Usage — clean, explicit, minimal noise
func TestGetActiveUsers(t *testing.T) {
    active := NewUserBuilder().WithName("Alice").Insert(t, testDB)
    inactive := NewUserBuilder().WithName("Bob").Inactive().Insert(t, testDB)

    users, err := svc.GetActiveUsers(context.Background())
    // assert active is in users, inactive is not
    _ = inactive
}
```

### Mocking and Interfaces

Go's approach to mocking is interface-based. Because Go interfaces are satisfied implicitly, any type that has the right methods satisfies the interface — including test doubles you write inline.

#### Manual mock — the simplest approach
```go
// Production interface
type EmailSender interface {
    Send(ctx context.Context, to, subject, body string) error
}

// Manual mock — just a struct with the methods and captured calls
type mockEmailSender struct {
    calls []struct {
        to      string
        subject string
        body    string
    }
    err error  // if set, Send returns this error
}

func (m *mockEmailSender) Send(ctx context.Context, to, subject, body string) error {
    m.calls = append(m.calls, struct {
        to      string
        subject string
        body    string
    }{to, subject, body})
    return m.err
}

func TestWelcomeEmail(t *testing.T) {
    sender := &mockEmailSender{}
    svc := NewUserService(sender)

    err := svc.RegisterUser(context.Background(), "alice@example.com")
    if err != nil {
        t.Fatal(err)
    }

    if len(sender.calls) != 1 {
        t.Fatalf("expected 1 email sent, got %d", len(sender.calls))
    }
    if sender.calls[0].to != "alice@example.com" {
        t.Errorf("email sent to %q, want alice@example.com", sender.calls[0].to)
    }
}

// Test error path
func TestWelcomeEmailFailure(t *testing.T) {
    sender := &mockEmailSender{err: errors.New("smtp unavailable")}
    svc := NewUserService(sender)

    err := svc.RegisterUser(context.Background(), "alice@example.com")
    if err == nil {
        t.Fatal("expected error when email fails, got nil")
    }
}
```

#### `testify/mock` — for more complex mocking needs

When you need call expectations, argument matching, and return value sequences:
```go
import "github.com/stretchr/testify/mock"

type MockUserRepo struct {
    mock.Mock
}

func (m *MockUserRepo) GetByID(ctx context.Context, id int) (*User, error) {
    args := m.Called(ctx, id)
    // Return typed values — handle nil carefully
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func TestUserService_GetProfile(t *testing.T) {
    repo := new(MockUserRepo)
    svc := NewUserService(repo)

    expectedUser := &User{ID: 1, Name: "Alice"}

    // Set expectation: when GetByID is called with ctx and 1, return expectedUser, nil
    repo.On("GetByID", mock.Anything, 1).Return(expectedUser, nil)

    user, err := svc.GetProfile(context.Background(), 1)
    if err != nil {
        t.Fatal(err)
    }

    if user.Name != "Alice" {
        t.Errorf("got name %q, want Alice", user.Name)
    }

    // Verify all expected calls were made
    repo.AssertExpectations(t)
}

// Test not found path
func TestUserService_GetProfile_NotFound(t *testing.T) {
    repo := new(MockUserRepo)
    svc := NewUserService(repo)

    repo.On("GetByID", mock.Anything, 99).Return(nil, ErrNotFound)

    _, err := svc.GetProfile(context.Background(), 99)
    if !errors.Is(err, ErrNotFound) {
        t.Errorf("expected ErrNotFound, got %v", err)
    }

    repo.AssertExpectations(t)
}
```

#### `testify/assert` and `testify/require`

The most widely used assertion library in Go. `assert` continues the test on failure, `require` stops immediately (like `t.Error` vs `t.Fatal`):
```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestCreateUser(t *testing.T) {
    user, err := svc.CreateUser(ctx, "Alice", "alice@example.com")

    // require — stop if this fails, rest of test is meaningless
    require.NoError(t, err)
    require.NotNil(t, user)

    // assert — check all of these, report all failures
    assert.Equal(t, "Alice", user.Name)
    assert.Equal(t, "alice@example.com", user.Email)
    assert.True(t, user.Active)
    assert.False(t, user.DeletedAt.Valid)
    assert.WithinDuration(t, time.Now(), user.CreatedAt, time.Second)

    // Error checking
    _, err = svc.CreateUser(ctx, "", "")
    assert.ErrorIs(t, err, ErrValidation)
    assert.ErrorContains(t, err, "name is required")
}
```

### `httptest` — testing HTTP handlers

The `net/http/httptest` package lets you test HTTP handlers without starting a real server.

#### `httptest.NewRecorder` — unit test a single handler

```go
func TestGetUserHandler(t *testing.T) {
    tests := []struct {
        name       string
        userID     string
        mockUser   *User
        mockErr    error
        wantStatus int
        wantBody   string
    }{
        {
            name:       "found",
            userID:     "1",
            mockUser:   &User{ID: 1, Name: "Alice"},
            wantStatus: http.StatusOK,
        },
        {
            name:       "not found",
            userID:     "99",
            mockErr:    ErrNotFound,
            wantStatus: http.StatusNotFound,
        },
        {
            name:       "invalid id",
            userID:     "abc",
            wantStatus: http.StatusBadRequest,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Build mock service
            svc := &mockUserService{
                user: tt.mockUser,
                err:  tt.mockErr,
            }
            h := NewUserHandler(svc)

            // Build request
            req := httptest.NewRequest(http.MethodGet, "/users/"+tt.userID, nil)
            req = req.WithContext(context.Background())

            // Recorder captures the response
            rec := httptest.NewRecorder()

            // Serve — synchronous, no network
            h.GetUser(rec, req)

            // Inspect result
            res := rec.Result()
            defer res.Body.Close()

            require.Equal(t, tt.wantStatus, res.StatusCode)

            if tt.wantStatus == http.StatusOK {
                var got User
                require.NoError(t, json.NewDecoder(res.Body).Decode(&got))
                assert.Equal(t, tt.mockUser.Name, got.Name)
            }
        })
    }
}
```

#### `httptest.NewServer` — integration test with a real HTTP server

When you need to test middleware chains, routing, or an actual HTTP client:

```go
func TestUserAPI(t *testing.T) {
    // Build the full router with middleware
    router := buildRouter(testDB)

    // Start a real HTTP server on a random port
    srv := httptest.NewServer(router)
    defer srv.Close()  // stops the server when test ends

    client := srv.Client()  // pre-configured HTTP client that trusts the test server

    t.Run("create and retrieve user", func(t *testing.T) {
        // Create
        body := strings.NewReader(`{"name":"Alice","email":"alice@example.com"}`)
        resp, err := client.Post(srv.URL+"/api/users", "application/json", body)
        require.NoError(t, err)
        defer resp.Body.Close()
        require.Equal(t, http.StatusCreated, resp.StatusCode)

        var created User
        require.NoError(t, json.NewDecoder(resp.Body).Decode(&created))
        require.NotZero(t, created.ID)

        // Retrieve
        resp2, err := client.Get(fmt.Sprintf("%s/api/users/%d", srv.URL, created.ID))
        require.NoError(t, err)
        defer resp2.Body.Close()
        require.Equal(t, http.StatusOK, resp2.StatusCode)

        var got User
        require.NoError(t, json.NewDecoder(resp2.Body).Decode(&got))
        assert.Equal(t, created.ID, got.ID)
        assert.Equal(t, "Alice", got.Name)
    })
}

// TLS server
func TestTLSEndpoint(t *testing.T) {
    srv := httptest.NewTLSServer(router)
    defer srv.Close()
    client := srv.Client()  // client pre-configured to trust the test cert
    // ...
}
```

#### Testing middleware specifically
```go
func TestAuthMiddleware(t *testing.T) {
    // A simple handler that confirms it was reached
    reached := false
    next := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        reached = true
        // Verify context was populated by middleware
        userID, ok := r.Context().Value(ctxKeyUserID).(int)
        assert.True(t, ok)
        assert.Equal(t, 42, userID)
        w.WriteHeader(http.StatusOK)
    })

    handler := AuthMiddleware(next)

    t.Run("valid token", func(t *testing.T) {
        reached = false
        token := generateTestJWT(t, 42)
        req := httptest.NewRequest("GET", "/", nil)
        req.Header.Set("Authorization", "Bearer "+token)
        rec := httptest.NewRecorder()

        handler.ServeHTTP(rec, req)

        assert.Equal(t, http.StatusOK, rec.Code)
        assert.True(t, reached, "handler should have been called")
    })

    t.Run("missing token", func(t *testing.T) {
        reached = false
        req := httptest.NewRequest("GET", "/", nil)
        rec := httptest.NewRecorder()

        handler.ServeHTTP(rec, req)

        assert.Equal(t, http.StatusUnauthorized, rec.Code)
        assert.False(t, reached, "handler should not have been called")
    })
}
```

### Testing the Database Layer
#### In-memory vs real database

There are two schools of thought. Running against a real PostgreSQL in tests (via Docker/testcontainers) gives you actual behaviour — real query plans, real constraint enforcement, real transaction semantics. An in-memory fake gives you speed but can hide bugs that only appear with real databases.

For backend development, the recommendation is: **use a real database for repository tests, use mocks for service/handler tests.**

```go
// testcontainers-go — spin up a real PostgreSQL in Docker for tests
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()

    ctx := context.Background()
    container, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(30*time.Second),
        ),
    )
    require.NoError(t, err)

    t.Cleanup(func() { container.Terminate(ctx) })

    dsn, err := container.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    db, err := sql.Open("postgres", dsn)
    require.NoError(t, err)
    t.Cleanup(func() { db.Close() })

    // Run migrations
    err = goose.Up(db, "../../migrations")
    require.NoError(t, err)

    return db
}
```

#### Transaction-based test isolation

Each test runs in a transaction that is rolled back at the end — the database is always clean for each test without truncating tables:
```go
func TestUserRepository(t *testing.T) {
    db := getSharedTestDB(t)  // shared across tests in this package

    t.Run("GetByID found", func(t *testing.T) {
        // Start a transaction for this test
        tx, err := db.Begin()
        require.NoError(t, err)
        t.Cleanup(func() { tx.Rollback() })  // always rollback — test isolation

        repo := NewUserRepository(tx)  // repo uses DBTX interface — accepts tx

        // Insert test data within the transaction
        _, err = tx.Exec(`INSERT INTO users (id, name, email) VALUES (1, 'Alice', 'a@b.com')`)
        require.NoError(t, err)

        // Test
        user, err := repo.GetByID(context.Background(), 1)
        require.NoError(t, err)
        assert.Equal(t, "Alice", user.Name)
        // tx.Rollback() in Cleanup removes the INSERT — no pollution between tests
    })

    t.Run("GetByID not found", func(t *testing.T) {
        tx, err := db.Begin()
        require.NoError(t, err)
        t.Cleanup(func() { tx.Rollback() })

        repo := NewUserRepository(tx)

        _, err = repo.GetByID(context.Background(), 9999)
        assert.ErrorIs(t, err, ErrNotFound)
    })
}
```

This pattern — begin tx in test, rollback in cleanup — is extremely fast and gives perfect isolation. Each test sees an empty database. The `DBTX` interface from the database layer chapter makes this possible — the repository doesn't care whether it has a `*sql.DB` or `*sql.Tx`.

### Benchmarks

Benchmarks measure performance. They run with `go test -bench=.` and are completely separate from tests.
```go
// Must start with Benchmark, receives *testing.B
func BenchmarkParseAmount(b *testing.B) {
    // b.N is set by the framework — it increases until timing is stable
    for i := 0; i < b.N; i++ {
        parseAmount("99.50")  // function under benchmark
    }
}

// With -benchmem, also reports allocations per operation
// BenchmarkParseAmount-8   5000000   234 ns/op   32 B/op   1 allocs/op
//                          ^N        ^time        ^bytes    ^alloc count
```

#### Setup outside the loop
```go
func BenchmarkUserSerialization(b *testing.B) {
    // Setup — not part of the measured time
    users := generateLargeUserList(1000)

    // Reset timer after setup
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        json.Marshal(users)  // only this is measured
    }
}
```

#### Sub-benchmarks — compare implementations
```go
func BenchmarkStringConcat(b *testing.B) {
    parts := []string{"hello", " ", "world", " ", "foo", " ", "bar"}

    b.Run("plus operator", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            s := ""
            for _, p := range parts { s += p }
            _ = s
        }
    })

    b.Run("strings.Builder", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var sb strings.Builder
            for _, p := range parts { sb.WriteString(p) }
            _ = sb.String()
        }
    })

    b.Run("strings.Join", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = strings.Join(parts, "")
        }
    })
}

// Run: go test -bench=BenchmarkStringConcat -benchmem
// Output:
// BenchmarkStringConcat/plus_operator-8    2000000   743 ns/op   216 B/op   6 allocs/op
// BenchmarkStringConcat/strings.Builder-8  5000000   289 ns/op    56 B/op   3 allocs/op
// BenchmarkStringConcat/strings.Join-8    10000000   134 ns/op    24 B/op   1 allocs/op
```

#### `b.ReportAllocs()` — show allocations without `-benchmem`
```go
func BenchmarkHotPath(b *testing.B) {
    b.ReportAllocs()  // always show allocation stats for this benchmark
    for i := 0; i < b.N; i++ {
        hotPathFunction()
    }
}
```

### Race Detector

The race detector is a dynamic analysis tool that detects data races at runtime. It instruments memory accesses and reports when two goroutines access the same memory concurrently without synchronization, and at least one access is a write.
```go
go test -race ./...
go run -race main.go
go build -race -o app_race .  # build race-enabled binary for staging
```

```go
// This test would catch the race in this function:
func TestConcurrentCounter(t *testing.T) {
    c := &Counter{}
    var wg sync.WaitGroup

    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            c.Increment()  // if Increment isn't synchronized, race detector catches it
        }()
    }

    wg.Wait()
    assert.Equal(t, 100, c.Value())
}
```

The race detector adds ~5-10x overhead. Run it in CI on every pull request, not in production. If it fires, treat it as a critical bug — races are non-deterministic and can manifest as data corruption, panics, or silent wrong results.

### Test Coverage
```bash
# Run tests with coverage
go test -cover ./...

# Generate coverage profile
go test -coverprofile=coverage.out ./...

# View in browser — color-coded by covered/not covered
go tool cover -html=coverage.out

# Coverage per function
go tool cover -func=coverage.out

# Fail if coverage drops below threshold (in CI)
go test -cover ./... | grep -E "coverage: [0-9]+\.[0-9]+" | \
    awk '{if ($2+0 < 80) exit 1}'
```

Coverage tells you which lines were executed by tests, not whether your tests are good. 100% coverage with trivial assertions is meaningless. Focus on covering the interesting paths — error cases, edge cases, concurrent code — not chasing a number.

### Fuzz Testing (Go 1.18+)

Fuzzing automatically generates inputs to find crashes and panics in your code.

```go
// Fuzz functions start with Fuzz, take *testing.F
func FuzzParseAmount(f *testing.F) {
    // Seed corpus — known interesting inputs
    f.Add("0")
    f.Add("100.00")
    f.Add("-1")
    f.Add("99999999.99")
    f.Add("")

    f.Fuzz(func(t *testing.T, input string) {
        // The fuzzer mutates input randomly
        // Your function must not panic — any panic is a bug
        result, err := parseAmount(input)
        if err != nil {
            return  // errors are fine
        }
        // Invariant: if no error, result must be a valid number
        if math.IsNaN(result) || math.IsInf(result, 0) {
            t.Errorf("parseAmount(%q) returned invalid float: %v", input, result)
        }
    })
}

// Run fuzzer (runs indefinitely until it finds a failure or you stop it)
// go test -fuzz=FuzzParseAmount
// go test -fuzz=FuzzParseAmount -fuzztime=30s  // run for 30 seconds
```

### Putting It All Together — a realistic test suite structure

Given your architecture (handler → service → repository), here is how testing fits each layer:

```
handlers/
    user_handler_test.go     — httptest.NewRecorder, mock service
service/
    user_service_test.go     — table-driven, mock repository
repository/
    user_repo_test.go        — real DB (testcontainers or shared test DB), tx rollback isolation
integration/
    user_flow_test.go        — httptest.NewServer, real DB, full stack
```

**Handler tests** — fast, no DB, mock the service interface, test HTTP concerns (status codes, headers, JSON encoding, middleware behaviour).

**Service tests** — fast, no DB, mock the repository interface, test business logic (validation, error mapping, orchestration).

**Repository tests** — slower, real DB, transaction rollback for isolation, test SQL correctness, constraint enforcement, NULL handling.

**Integration tests** — slowest, real everything, test full request flows end to end.
```go
// Service test — pure business logic, no infrastructure
func TestUserService_Register(t *testing.T) {
    tests := []struct {
        name        string
        email       string
        repoErr     error  // error the mock repo returns
        wantErr     bool
        wantErrType error
    }{
        {
            name:  "success",
            email: "alice@example.com",
        },
        {
            name:        "duplicate email",
            email:       "existing@example.com",
            repoErr:     ErrDuplicateEmail,
            wantErr:     true,
            wantErrType: ErrDuplicateEmail,
        },
        {
            name:        "invalid email",
            email:       "not-an-email",
            wantErr:     true,
            wantErrType: ErrValidation,
        },
        {
            name:        "empty email",
            email:       "",
            wantErr:     true,
            wantErrType: ErrValidation,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := &mockUserRepo{err: tt.repoErr}
            emailer := &mockEmailSender{}
            svc := NewUserService(repo, emailer)

            user, err := svc.Register(context.Background(), tt.email)

            if tt.wantErr {
                require.Error(t, err)
                if tt.wantErrType != nil {
                    assert.ErrorIs(t, err, tt.wantErrType)
                }
                return
            }

            require.NoError(t, err)
            assert.Equal(t, tt.email, user.Email)
            assert.True(t, user.Active)
            assert.Len(t, emailer.calls, 1)  // welcome email sent
        })
    }
}
```

### Gotchas Summary

|Gotcha|What happens|Fix|
|---|---|---|
|Missing `t.Helper()` in helpers|Failure reported inside helper, not at call site|Always first line of assertion helpers|
|Missing `tt := tt` in parallel table tests|All subtests use same loop variable|Capture loop variable before `t.Parallel()` (< Go 1.22)|
|`defer` in `TestMain`|`os.Exit` bypasses deferred functions — cleanup silently skipped|Explicit cleanup before `os.Exit(code)`|
|`assert` vs `require`|`assert` continues after failure — later code may panic on nil|Use `require` when subsequent steps depend on previous ones|
|`t.Log` only shown on failure|Debugging info invisible in passing tests|Add `-v` flag or use `t.Log` for failure-only context|
|No `-count=1` flag|Go caches test results — code change not reflected|Use `-count=1` to disable caching|
|Race detector not in CI|Races only appear under specific timing|Always run `-race` in CI|
|Testing time with `time.Now()`|Tests are non-deterministic — may fail at midnight|Inject time as a dependency or use `clockwork`/`timex` for fakeable clock|
|`httptest.NewRecorder` not flushing|Some handlers may buffer output|Call `rec.Result()` to get final response, not `rec.Body` directly|
|Forgetting `rows.Close()` in repo tests|Connection held open, pool exhausted if test creates many rows objects|Always `defer rows.Close()` in repository code|

---

### Summary

|Tool|Use For|
|---|---|
|`t.Error` / `t.Errorf`|Failure, keep running|
|`t.Fatal` / `t.Fatalf`|Failure, stop test|
|`t.Helper()`|Correct failure line in helpers|
|`t.Cleanup()`|Resource teardown tied to test lifecycle|
|`t.Run()`|Subtests and table-driven tests|
|`t.Parallel()`|Concurrent test execution|
|`TestMain`|Package-level setup and teardown|
|`httptest.NewRecorder`|Unit test HTTP handlers|
|`httptest.NewServer`|Integration test full HTTP stack|
|`testing.B`|Benchmarks and allocation measurement|
|`-race` flag|Detect data races|
|`-fuzz` flag|Random input generation|
|`cmp.Diff`|Readable struct diffs|
|`testify/assert`|Fluent assertions without stopping|
|`testify/require`|Fluent assertions that stop on failure|
|`testify/mock`|Expectation-based mocking|
|Transaction rollback|DB test isolation without truncating|
