### How This Section Is Structured

1. **Session 1A** — Goroutines + G-M-P scheduler in depth
2. **Session 1B** — Channels in full depth including internals and all edge cases
3. **Session 1C** — Select statement
4. **Session 2A** — All four core patterns with canonical implementations
5. **Session 2B** — Sync primitives revisited in concurrency context
6. **Session 2C** — Timeouts and cancellation in goroutine context
7. **Session 3A** — Failure modes: leaks, deadlocks, races
8. **Session 3B** — Production: backpressure, limiting, errgroup

# Session 1A — Goroutines and the G-M-P scheduler
---
### The Mental Model First

Most languages give you threads. A thread is an OS-level construct — the kernel knows about it, schedules it, and when you create one, you're making a syscall that allocates a kernel stack (typically 1-8MB), registers the thread with the OS scheduler, and from that point the OS decides when it runs.

Go does something different. Goroutines are **user-space threads** managed entirely by the Go runtime. The OS has no idea goroutines exist. From the OS's perspective, your Go program is just a handful of OS threads doing work. The Go runtime is a scheduler that runs inside your process, multiplexing thousands or millions of goroutines onto those few OS threads.

This is why goroutines are cheap. Creating an OS thread costs a syscall and 1-8MB of stack. Creating a goroutine costs no syscall and starts with a 2-8KB stack. You can have a million goroutines in a program that uses only a few hundred MB of memory. The same program using OS threads would need terabytes.

The machinery that makes this work is the **G-M-P model**.

### The G-M-P Model

Three entities, three distinct roles.
```
G — Goroutine    the unit of work
M — Machine      an OS thread
P — Processor    a logical CPU with a run queue
```

Think of it this way before going into detail:
```
G is the work.
P is the permission to run work.
M is the thing that actually executes work.

M needs P to run Gs.
P holds the queue of Gs waiting to run.
M picks Gs from P's queue and executes them.
```

Without a P, an M cannot run Go code. This is the key constraint that controls parallelism.

#### G — Goroutine
---
A G is a struct inside the Go runtime that represents a goroutine. It contains:

```
stack pointer        — current top of the goroutine's stack
stack bounds         — lo and hi addresses of the stack
program counter      — where execution is currently at
status               — running, runnable, waiting, dead, etc.
reference to M       — which machine is running it (if any)
reference to P       — which processor owns it (if any)
```

**Goroutine status transitions:**

```
_Gidle       — just allocated, not initialized
_Grunnable   — on a run queue, ready to run, not currently running
_Grunning    — currently executing on an M
_Gsyscall    — executing a syscall, not on a P
_Gwaiting    — blocked (channel, mutex, timer, IO) — not on any run queue
_Gdead       — finished, can be reused
```

The transition that matters most in practice:
```
_Grunnable → _Grunning    when M picks G from P's run queue
_Grunning  → _Gwaiting    when G blocks on channel send/recv, mutex, etc.
_Gwaiting  → _Grunnable   when the block is resolved (data arrived, lock released)
_Grunning  → _Gsyscall    when G makes a syscall
_Gsyscall  → _Grunning    when syscall completes and P is available
```

When a goroutine blocks on a channel receive, it doesn't block the OS thread. The runtime parks the goroutine (status → `_Gwaiting`), saves its state, and the M picks up the next runnable G from the P's queue. The blocked G sits in a wait list associated with the channel. When data arrives on the channel, the runtime moves it back to `_Grunnable` and puts it on a run queue.

#### P — Processor
---
A P is a logical processor. The number of Ps is controlled by `GOMAXPROCS` — by default it equals the number of CPU cores available to the process.

Each P has:
```
run queue (local)    — a circular buffer of runnable Gs, capacity 256
reference to M       — which machine is currently using this P
mcache               — per-P memory allocator cache (avoids contention on global heap)
```

The number of Ps is the **true concurrency limit** for Go code. If `GOMAXPROCS=4`, at most 4 goroutines are running Go code simultaneously. You can have millions of goroutines but only 4 are executing at any given instant.

This is important to internalise: **goroutine count is not parallelism**. Parallelism is bounded by GOMAXPROCS. Goroutine count is the number of concurrent logical tasks, most of which are waiting at any point in time.

```go
runtime.GOMAXPROCS(runtime.NumCPU())  // default — use all CPUs
runtime.GOMAXPROCS(1)                 // single-threaded — useful for debugging races
n := runtime.GOMAXPROCS(0)           // query current value without changing it
```

#### M — Machine
---
An M is an OS thread. The runtime creates and manages Ms. The number of Ms is not fixed — the runtime creates new Ms when needed and parks them when not.

**Each M:**

- Must hold a P to execute Go code
- Can exist without a P while blocked in a syscall
- Has a small per-M stack for runtime code
- Has a reference to the currently running G

The relationship:
```
M without P  →  parked, waiting for a P to become available
M with P     →  executing G from P's run queue
M in syscall →  temporarily surrenders P so another M can use it
```

There is a limit on the number of Ms: `runtime/debug.SetMaxThreads`, default 10,000. In practice you rarely approach this unless you have goroutines stuck in long syscalls.

### The Full Picture
```
GOMAXPROCS = 4  →  4 Ps exist

P0 [G1, G2, G3...]    P1 [G4, G5...]    P2 [G6...]    P3 []
 |                      |                 |              |
 M0 (running G1)       M1 (running G4)   M2 (syscall)  M3 (idle)

Global run queue: [G7, G8, G9, G10...]   ← overflow from local queues
```

Each P has its own local run queue. There is also a global run queue that P's check periodically or when their local queue is empty. This two-level queue design reduces contention — most of the time Ps work from their local queues without any synchronisation.

### Work Stealing

This is how the runtime keeps all Ps busy even when work is unevenly distributed.

The scenario: P0's local queue has 200 goroutines. P1 just finished its last goroutine and has nothing to do. Rather than P1 sitting idle while P0 is overloaded, P1 **steals half of P0's run queue**.

```
Before stealing:
P0 [G1,G2,G3,G4,G5,G6,G7,G8]   P1 []

After P1 steals from P0:
P0 [G1,G2,G3,G4]                P1 [G5,G6,G7,G8]
```

Work stealing happens when:

1. A P's local run queue is empty
2. The global run queue is empty
3. There is no network poller work available

The P picks a random victim P and attempts to steal half its run queue. "Random victim" is important — it avoids all idle Ps piling onto the same busy P simultaneously.

This is why goroutines don't need affinity to a specific CPU. The scheduler moves work to wherever there is capacity. It also means you generally should not worry about which CPU a goroutine runs on — the scheduler handles load balancing automatically.


### Syscalls and the Scheduler

This is where the design gets clever. When a goroutine makes a blocking syscall (file read, network accept, etc.), it would normally block the OS thread, preventing that M from running other goroutines.

Go handles this in two ways depending on the syscall type.

**Network I/O — non-blocking with netpoller:**

Go's network layer uses non-blocking syscalls under the hood. When a goroutine does `conn.Read()`, the runtime:
```
1. Tries the non-blocking read
2. If data available  → returns immediately, G stays _Grunning
3. If no data yet     → parks the G (_Gwaiting), registers the fd with netpoller
4. M picks next G from P's queue and continues
5. When data arrives  → netpoller wakes up, moves G back to _Grunnable
```

The netpoller is a background thread that uses `epoll` (Linux), `kqueue` (macOS), or `IOCP` (Windows) to monitor file descriptors. Your goroutine blocks from its own perspective, but the OS thread never blocks.

**Blocking syscalls (file I/O, CGo, etc.):**

For truly blocking syscalls that can't be made non-blocking:
```
1. G enters syscall → status becomes _Gsyscall
2. P is handed off to another M (or a new M is created if none available)
3. M0 (with G in syscall) keeps running, but without a P
4. When syscall completes:
   a. M0 tries to reclaim its P
   b. If P is still free → reclaim it, G goes back to _Grunning
   c. If P was taken → put G on global run queue, M0 parks itself
```

This handoff is why blocking syscalls don't stall the entire program. The P is freed to run other goroutines while the syscall is in flight. The cost is that you might create a new OS thread — which is why programs with many goroutines doing blocking file I/O can end up with many OS threads.

```go
// You can observe this
import "runtime"

fmt.Println(runtime.NumGoroutine())   // current goroutine count
// There's no runtime.NumThreads() but you can see it in:
// /proc/<pid>/status   →  Threads: field (Linux)
// pprof goroutine profile
```

### Goroutine Lifecycle
```go
// 1. Created — go keyword
go func() {
    doWork()
}()
// New G allocated, pushed to current P's local run queue
// No syscall, no OS involvement — extremely cheap

// 2. Running — scheduler picks it up
// M dequeues G from P's run queue, sets G status to _Grunning
// Execution begins

// 3. Preemption — scheduler can preempt running goroutines
// Before Go 1.14: only at function call boundaries (cooperative)
// Go 1.14+: asynchronous preemption via signals — goroutines preempted
//            even in tight loops with no function calls

// 4. Blocking — G encounters a blocking operation
// Channel recv with no sender
// Mutex.Lock() when locked
// time.Sleep
// network I/O
// → G parked, M picks next G

// 5. Unblocking — blocking condition resolved
// Data arrives on channel, mutex released, timer fires
// → G moved back to _Grunnable, put on run queue

// 6. Termination — function returns
// G status → _Gdead
// Stack memory returned to pool for reuse
// G struct may be reused for a future goroutine (object pool)
```

### Stack Growth

Goroutines start with a small stack (2KB in current Go, was 8KB in older versions) and grow automatically. This is how millions of goroutines fit in reasonable memory.

**How it works — stack copying:**
```
1. G starts with 2KB stack
2. On every function call, runtime checks: is there enough stack space?
3. If yes → proceed normally
4. If no  → allocate a NEW, larger stack (double the current size)
            copy entire existing stack to new stack
            update all pointers that pointed into old stack
            free old stack
            continue execution on new stack
```

Stack copying is why you can't have Go pointers to stack variables that persist across function calls in certain situations — the stack can move. The GC and runtime handle this transparently for normal Go code.

**Implications:**
```go
// This is fine — no stack pinning needed
func recursive(n int) int {
    if n == 0 { return 0 }
    return recursive(n - 1)  // stack grows as needed, no stack overflow panic
}                             // unless you recurse millions of levels deep

// Stack size is unbounded by default — a goroutine that recurses
// infinitely will consume memory until OOM, not stack overflow
// (unless you hit the goroutine stack size limit — default 1GB)

// You can change the max stack size
debug.SetMaxStack(64 * 1024 * 1024)  // 64MB max per goroutine stack
```

The maximum goroutine stack size (default 1GB on 64-bit) exists to catch infinite recursion. If a goroutine's stack grows beyond this, the program panics with "stack overflow".

### Preemption — Before and After Go 1.14

This is subtle but important for understanding scheduler behaviour.

**Before Go 1.14 — cooperative preemption:**

The scheduler could only preempt a goroutine at function call boundaries. A goroutine in a tight loop with no function calls could hold an M indefinitely, starving other goroutines on that P.
```go
// Pre-1.14: this goroutine could starve others on the same P
go func() {
    for {
        i++  // no function calls — never preempted
    }
}()
```

**Go 1.14+ — asynchronous preemption:**

The runtime sends `SIGURG` signals to OS threads at regular intervals (~10ms). The signal handler checks if the current goroutine has been running too long and if so, preempts it — suspends it and puts it back on the run queue, allowing other goroutines to run.

This means even a tight infinite loop can be preempted. The practical implication: you no longer need `runtime.Gosched()` calls in tight loops to be a good citizen, though it still exists for explicit yielding.

```go
runtime.Gosched()  // voluntarily yield — let other goroutines run
                   // rarely needed post-1.14 but still valid
```

### Creating Goroutines — Mechanics and Gotchas

```go
// Basic
go doWork()

// Closure — captures variables from enclosing scope
x := 10
go func() {
    fmt.Println(x)  // captures x by reference — x could change before this runs
}()

// Safe closure — pass value as argument
go func(x int) {
    fmt.Println(x)  // x is a copy — safe
}(x)

// Method on value
go obj.Method()      // obj is copied if Method has value receiver

// Method on pointer
go obj.PointerMethod()  // safe — pointer is captured, not copied
```

**The loop variable capture bug** — appears constantly in real code:
```go
// BROKEN — all goroutines print the same value (last value of i)
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i)  // captures i by reference — loop may finish before goroutines run
    }()
}
// Possible output: 5 5 5 5 5

// FIXED — pass as argument
for i := 0; i < 5; i++ {
    go func(i int) {
        fmt.Println(i)  // i is a copy of the loop variable at this iteration
    }(i)
}
// Output: 0 1 2 3 4 (in some order)

// Go 1.22+ — loop variable semantics changed
// Each iteration gets its own variable — no capture bug
// But pre-1.22 codebases still have this pattern everywhere
```

### `runtime.NumGoroutine` — observing the scheduler

```go
import "runtime"

// Current number of goroutines (running + blocked + waiting)
n := runtime.NumGoroutine()

// Useful for detecting goroutine leaks in tests
func TestNoLeaks(t *testing.T) {
    before := runtime.NumGoroutine()

    runSomethingThatSpawnsGoroutines()

    // Give goroutines time to finish
    time.Sleep(100 * time.Millisecond)

    after := runtime.NumGoroutine()
    if after > before {
        t.Errorf("goroutine leak: started with %d, ended with %d", before, after)
    }
}

// goleak library — more robust leak detection
import "go.uber.org/goleak"

func TestMyFunction(t *testing.T) {
    defer goleak.VerifyNone(t)  // fails if any goroutines leaked after test
    // ...
}
```

### GOMAXPROCS in Practice

```go
// Default: number of available CPUs
// This is almost always correct — don't change it without profiling

// When you might reduce it:
// - Running in a container with CPU limits
//   (Go 1.5+ reads CPU quota from cgroups automatically — usually handled)
// - Sharing a machine with latency-sensitive processes
// - Debugging concurrency bugs (GOMAXPROCS=1 makes races more deterministic)

// When you might increase it:
// - Almost never — adding more Ps than CPUs causes context switching overhead
//   with no benefit

// The container case — automatic since Go 1.5 with automaxprocs library
import _ "go.uber.org/automaxprocs"
// Sets GOMAXPROCS to match container CPU quota at startup
// Without this, a container limited to 2 CPUs but running on a 64-core host
// sets GOMAXPROCS=64 — creating 64 Ps when only 2 can run
```

### Visualising G-M-P Together

Here is the full picture of what happens when your server handles concurrent requests:

```
Incoming requests: [req1, req2, req3, req4, req5, req6, req7, req8]

GOMAXPROCS=4 → 4 Ps

P0 [G_req3, G_req7]     P1 [G_req4, G_req8]
 |                        |
 M0 executing G_req1     M1 executing G_req2
 (reading from conn)      (querying DB — waiting for response)
    |                        |
    G_req1 blocks           G_req2 blocks
    on network read         on DB response
    ↓                       ↓
    parked (_Gwaiting)      parked (_Gwaiting)
    M0 picks G_req3         M1 picks G_req4

P2 []                    P3 [G_req6]
 |                        |
 M2 executing G_req5     M3 executing G_req6
 (CPU work — computing)   (writing response — syscall)
                              |
                              P3 handed off to M4
                              M3 continues syscall without P
                              M4 picks G from P3 queue

Global run queue: [G_req9, G_req10] ← overflow
Netpoller: waiting on G_req1 (network), G_req2 (DB socket)
```

Every request that's waiting for I/O is parked and costs essentially nothing. The 4 Ms are always doing real work. When the DB responds, G_req2 moves from the netpoller back to a run queue and gets picked up as soon as an M is free.

This is why Go can handle tens of thousands of concurrent connections on modest hardware with a small fixed thread pool. The goroutines waiting for I/O are essentially free.

### Gotchas Summary

|Gotcha|What happens|Fix|
|---|---|---|
|Loop variable capture|All goroutines share the last value of loop variable|Pass as function argument or use Go 1.22+|
|Assuming goroutine order|Goroutines run in non-deterministic order|Never rely on scheduling order|
|GOMAXPROCS in containers|Default reads host CPU count, not container quota|Use `automaxprocs` library|
|Tight loops pre-1.14|Could starve other goroutines on same P|Now handled by async preemption, but call `runtime.Gosched()` in very tight CPU-bound loops to be safe|
|Goroutine with no exit condition|Runs forever, leaks memory and scheduler resources|Every goroutine needs a way out — via channel, context, or return condition|
|Creating goroutines in a loop without bounds|Spawns millions of goroutines under load|Bound with worker pool or semaphore|
|CGo blocking syscalls|Each blocks an M, new Ms created, can exhaust thread limit|Limit CGo goroutines or use thread pool|
|Assuming stack size is fixed|Stack grows dynamically — no fixed limit per goroutine|Max stack defaults to 1GB — infinite recursion OOMs before stack overflow|
### Summary

|Concept|Key Point|
|---|---|
|G|Goroutine — lightweight, starts at 2KB stack, user-space thread|
|M|OS thread — needs a P to run Go code|
|P|Logical processor — holds run queue, count = GOMAXPROCS|
|Work stealing|Idle P steals half the run queue from a busy P|
|Blocking I/O|Goroutine parked, M picks next G — OS thread never blocks|
|Blocking syscall|P handed off to another M so work continues|
|Stack growth|Stacks copy and grow dynamically — no pre-allocation needed|
|Preemption|Async since Go 1.14 — no tight loop starvation|
|GOMAXPROCS|True parallelism limit — defaults to CPU count|

---

# **Session 1B** — Channels in full depth
---
Most concurrency primitives are about **protecting shared state** — you have memory that multiple goroutines can access, and you put locks around it to prevent simultaneous access. Mutexes, semaphores, read-write locks — they all follow this model. The shared memory exists first, and the synchronisation is layered on top to make access safe.


Channels are a different philosophy entirely. The idea, borrowed from Tony Hoare's **Communicating Sequential Processes**, is:

> > Don't communicate by sharing memory. Share memory by communicating.

Instead of having shared state that goroutines all reach into, you have goroutines that are completely independent and pass ownership of data between each other through channels. The goroutine that sends data is done with it. The goroutine that receives data now owns it. There is no shared access — there is **transfer of ownership**.

This mental model has a concrete consequence: when you design a concurrent system in Go, your first question should not be "what mutex do I need here?" It should be "which goroutine owns this data at each point in time, and when does ownership transfer?"

Channels are the mechanism for that transfer. They also serve a second purpose: **synchronisation**. The act of sending and receiving is itself a synchronisation point — the sender and receiver coordinate at the channel regardless of what data is exchanged. Sometimes the data is what matters. Sometimes the synchronisation is what matters and the data is incidental (the `struct{}` signal pattern).

### What a Channel Actually Is

A channel is a reference type backed by a runtime structure called `hchan`. Understanding its internals explains all the behaviour.

```go
// Simplified view of hchan in the runtime
type hchan struct {
    qcount   uint          // number of elements currently in the buffer
    dataqsiz uint          // capacity of the circular buffer (0 for unbuffered)
    buf      unsafe.Pointer // pointer to circular buffer array
    elemsize uint16        // size of each element in bytes
    closed   uint32        // 0 = open, 1 = closed
    elemtype *_type        // type information for elements

    sendx uint             // send index in circular buffer
    recvx uint             // receive index in circular buffer

    recvq    waitq         // list of goroutines blocked waiting to receive
    sendq    waitq         // list of goroutines blocked waiting to send

    lock     mutex         // protects all fields above
}
```

The two wait queues — `recvq` and `sendq` — are the key to understanding blocking behaviour. When a goroutine tries to receive from an empty channel, it parks itself and adds its own goroutine struct to `recvq`. When a goroutine tries to send to a full channel, it parks itself and adds to `sendq`. When the blocking condition resolves, the runtime directly unparks the waiting goroutine and hands it the data.

This direct handoff is more efficient than the alternative. When a goroutine is waiting to receive and a sender arrives, the runtime copies data directly from the sender's stack into the receiver's stack — bypassing the channel buffer entirely. The receiver is unblocked and the sender never blocks at all.

### Channel Creation

```go
// Unbuffered channel — capacity 0
ch := make(chan int)
ch := make(chan int, 0)  // explicit, same thing

// Buffered channel — capacity N
ch := make(chan int, 10)
ch := make(chan struct{}, 1)

// Channel of channels — uncommon but valid
ch := make(chan chan int)

// Channel of functions
ch := make(chan func())

// Nil channel — declared but not initialised
var ch chan int  // ch is nil
```

### Unbuffered Channels — the rendezvous

An unbuffered channel has zero capacity. There is no buffer. A send and a receive must happen simultaneously — they meet at the channel.
```
Goroutine A:  ch <- value      ← blocks here until someone receives
Goroutine B:      <- ch        ← blocks here until someone sends
```

When A's send and B's receive meet at the channel, two things happen atomically: the value is transferred from A to B, and both goroutines are unblocked. This is called a **rendezvous** — neither can proceed without the other.

```go
ch := make(chan int)

go func() {
    fmt.Println("sender: about to send")
    ch <- 42                        // blocks until receiver is ready
    fmt.Println("sender: sent")     // only prints after receiver has the value
}()

time.Sleep(time.Second)             // receiver is delayed
fmt.Println("receiver: about to receive")
v := <-ch                          // unblocks sender
fmt.Println("receiver: got", v)

// Output (guaranteed order):
// sender: about to send
// receiver: about to receive  ← after 1 second delay
// sender: sent                ← both happen simultaneously at the rendezvous
// receiver: got 42
```

The synchronisation guarantee is strong: when the send completes, you know the receiver has the value. This is why unbuffered channels are sometimes called **synchronous channels**.

### Buffered Channels — asynchronous up to capacity

A buffered channel has a circular buffer. A send succeeds immediately if there is space in the buffer. A receive succeeds immediately if there is data in the buffer. The sender and receiver don't need to be present simultaneously — they only block when the buffer is full (sender) or empty (receiver).

```
Buffer capacity = 3, currently has [A, B]:

Send C:  [A, B, C]  ← succeeds immediately, sender not blocked
Send D:             ← blocks, buffer full, sender waits in sendq
Recv:    [B, C]     ← returns A, succeeds immediately
```

```go
ch := make(chan int, 3)

ch <- 1   // doesn't block — buffer has space
ch <- 2   // doesn't block
ch <- 3   // doesn't block
// ch <- 4 would block here — buffer full

fmt.Println(<-ch)  // 1 — channels are FIFO
fmt.Println(<-ch)  // 2
fmt.Println(<-ch)  // 3
// <-ch would block here — buffer empty
```

The buffered channel provides **decoupling** between producer and consumer speed. If the producer is occasionally faster than the consumer, the buffer absorbs the burst without blocking. But the buffer is not infinite — when it fills, the producer blocks and backpressure propagates naturally.

### The Critical Difference — when to use each

This is the most important design decision when working with channels.

**Use unbuffered when:**

- You need a synchronisation guarantee — you need to know the receiver has the value before proceeding.
- You are passing ownership and want to know transfer is complete.
- You want to rate-limit a producer to the speed of the consumer.
- You are building a done/signal channel.

**Use buffered when:**

- Producer and consumer run at different speeds and you want to smooth bursts.
- You are implementing a semaphore (buffer size = concurrency limit).
- You know exactly how many items will be sent (fixed-size pipeline).
- You want a goroutine to send and continue without waiting (fire and forget with bounded capacity).

**The most common mistake** — using a buffered channel to "fix" a blocking problem without understanding why it's blocking. Buffering hides the symptom. If a channel is blocking because the consumer is too slow, buffering buys time but doesn't solve the underlying speed mismatch. Under sustained load, the buffer fills and you're back to blocking.

### Send and Receive Operations

```go
ch := make(chan int, 2)

// Send
ch <- 42          // blocking send — blocks if channel full (or no receiver for unbuffered)

// Receive
v := <-ch         // blocking receive — blocks if channel empty (or no sender for unbuffered)
v, ok := <-ch     // receive with ok — ok is false if channel is closed and empty

// Non-blocking send and receive — via select with default
select {
case ch <- 42:
    // sent
default:
    // channel full or no receiver — did not send
}

select {
case v := <-ch:
    // received v
default:
    // channel empty or no sender — did not receive
}
```

The `ok` idiom in receive is critical and deserves special attention. `ok` is `false` only when two conditions are both true:

1. The channel is closed
2. The buffer is empty (all remaining values have been received)

If the channel is closed but still has values in the buffer, `ok` is `true` for those remaining receives. The channel drains completely before `ok` becomes `false`.

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3
close(ch)

v, ok := <-ch  // v=1, ok=true   — still has data
v, ok  = <-ch  // v=2, ok=true
v, ok  = <-ch  // v=3, ok=true
v, ok  = <-ch  // v=0, ok=false  — closed and empty
v, ok  = <-ch  // v=0, ok=false  — stays this way forever
```

### Channel Directions — type safety for channel roles

You can declare channels with a direction to restrict what a function can do with them. This documents intent and catches bugs at compile time.

```go
// Bidirectional — can send and receive
var ch chan int

// Send-only — can only send into it
var sendOnly chan<- int

// Receive-only — can only receive from it
var recvOnly <-chan int

// Bidirectional converts to directional implicitly
ch := make(chan int)
var s chan<- int = ch   // OK — restrict to send-only
var r <-chan int = ch   // OK — restrict to receive-only

// Directional does NOT convert back to bidirectional
var ch2 chan int = s    // COMPILE ERROR — cannot go back
```

The pattern in practice: functions that produce values take a `chan<- T` parameter. Functions that consume values take a `<-chan T`. The `make` call at the point of creation uses the bidirectional type, and the directional types are used everywhere else.

```go
// Producer — can only send
func generate(done <-chan struct{}, out chan<- int) {
    defer close(out)
    for i := 0; ; i++ {
        select {
        case out <- i:
        case <-done:
            return
        }
    }
}

// Consumer — can only receive
func process(in <-chan int, out chan<- string) {
    for v := range in {
        out <- fmt.Sprintf("processed %d", v)
    }
}

// Wiring — only main/orchestrator has bidirectional channels
func main() {
    done := make(chan struct{})
    numbers := make(chan int, 10)
    results := make(chan string, 10)

    go generate(done, numbers)   // numbers passed as chan<-
    go process(numbers, results) // numbers passed as <-chan
}
```

### Closing Channels — the rules and what happens

Closing is one-directional and irreversible. Understanding the exact semantics prevents a category of panics and incorrect logic.

#### The three rules of closing

**Rule 1: Only the sender should close a channel.**
This is a convention, not enforced by the compiler. But it's a strong convention because only the sender knows when there is no more data to send. A receiver closing a channel would be closing it from the consumer side — the producer doesn't know to stop sending, and sending to a closed channel panics.

**Rule 2: Closing a nil channel panics.**
```go
var ch chan int
close(ch)  // panic: close of nil channel
```

**Rule 3: Closing an already-closed channel panics.**
```go
ch := make(chan int)
close(ch)
close(ch)  // panic: close of closed channel
```

This second rule is where multiple-producer patterns get complicated. If multiple goroutines are all sending and any of them might close the channel, you need coordination to ensure only one close happens.

#### What receiving from a closed channel gives you
```go
ch := make(chan int, 2)
ch <- 10
ch <- 20
close(ch)

// Buffered values drain first
fmt.Println(<-ch)         // 10
fmt.Println(<-ch)         // 20

// Then zero values forever
fmt.Println(<-ch)         // 0  ← zero value of int
fmt.Println(<-ch)         // 0
fmt.Println(<-ch)         // 0  ← never blocks, never panics, returns forever
```

A closed channel with no buffer (or empty buffer) returns the zero value of the element type immediately, forever, without blocking. This behaviour is intentional and is used as a **broadcast signal** — closing a channel simultaneously unblocks all goroutines waiting to receive from it.

```go
// Broadcast pattern — closing signals ALL waiting goroutines
done := make(chan struct{})

for i := 0; i < 10; i++ {
    go func(id int) {
        <-done  // all 10 goroutines block here
        fmt.Printf("goroutine %d unblocked\n", id)
    }(i)
}

time.Sleep(time.Second)
close(done)  // all 10 goroutines unblock simultaneously
             // this would NOT work if you sent 10 values — you'd need exactly 10 receivers
```

This is fundamentally different from sending a value. Sending a value unblocks exactly one receiver. Closing the channel unblocks all current and future receivers. This is the canonical way to signal shutdown to multiple goroutines.

#### `range` over a channel

`range` receives from a channel until it is closed and empty:
```go
ch := make(chan int, 5)
for i := 1; i <= 5; i++ {
    ch <- i
}
close(ch)  // range needs the channel to be closed to terminate

for v := range ch {
    fmt.Println(v)  // 1, 2, 3, 4, 5 then loop ends
}
// Without close(ch), range blocks forever after draining the buffer
```

The `range` loop is equivalent to:

```go
for {
    v, ok := <-ch
    if !ok {
        break
    }
    // use v
}
```

The most common `range` over channel pattern — producer sends, closes, consumer ranges:

```go
func producer(ch chan<- int) {
    defer close(ch)  // close when done — signals consumer to stop ranging
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

func consumer(ch <-chan int) {
    for v := range ch {  // stops when ch is closed and empty
        process(v)
    }
}
```

`defer close(ch)` in the producer is the idiomatic pattern — the channel is always closed when the function exits, regardless of how it exits (normal return, panic, early return).

### The Nil Channel — the most useful thing you've never used

A nil channel blocks forever on both send and receive. This sounds useless but is actually one of the most elegant tools in Go's concurrency model.

```go
var ch chan int  // nil

ch <- 1          // blocks forever
<-ch             // blocks forever
```

In a `select` statement, a nil channel case is **never selected**. This lets you dynamically disable a case without restructuring your select.

```go
// Pattern: disable a channel case by setting it to nil
func merge(a, b <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for a != nil || b != nil {
            select {
            case v, ok := <-a:
                if !ok {
                    a = nil  // channel closed — disable this case
                    continue
                }
                out <- v
            case v, ok := <-b:
                if !ok {
                    b = nil  // channel closed — disable this case
                    continue
                }
                out <- v
            }
        }
    }()
    return out
}
```

Without the nil trick, once `a` closes, `case v, ok := <-a` would fire continuously returning `0, false` — spinning forever. Setting `a = nil` makes that case permanently blocked, so only `b` is selected until it also closes.

### The select Statement — channel multiplexing

`select` waits on multiple channel operations simultaneously. It is to channels what a `switch` is to values — except all cases are evaluated concurrently and any ready case can proceed.

```go
select {
case v := <-ch1:
    // received from ch1
case v := <-ch2:
    // received from ch2
case ch3 <- val:
    // sent to ch3
case <-done:
    // done signal received
default:
    // none of the above are ready — non-blocking
}
```

**When multiple cases are ready simultaneously**, Go picks one **uniformly at random**. Not the first one. Not the last one. Random. This prevents starvation of any particular case.

```go
// Demonstrating random selection
ch1 := make(chan string, 1)
ch2 := make(chan string, 1)
ch1 <- "one"
ch2 <- "two"

// Run this many times — sometimes "one", sometimes "two"
select {
case v := <-ch1:
    fmt.Println(v)
case v := <-ch2:
    fmt.Println(v)
}
```

### Patterns Built on These Primitives

#### Done channel — the shutdown signal
```go
func worker(done <-chan struct{}) {
    for {
        select {
        case <-done:
            fmt.Println("worker stopping")
            return
        default:
            doWork()
        }
    }
}

done := make(chan struct{})
go worker(done)

time.Sleep(time.Second)
close(done)  // signals all workers simultaneously
```

#### Timeout with select

```go
func callWithTimeout(ctx context.Context) (Result, error) {
    resultCh := make(chan Result, 1)
    errCh    := make(chan error, 1)

    go func() {
        result, err := expensiveCall()
        if err != nil {
            errCh <- err
            return
        }
        resultCh <- result
    }()

    select {
    case result := <-resultCh:
        return result, nil
    case err := <-errCh:
        return Result{}, err
    case <-ctx.Done():
        return Result{}, ctx.Err()
    case <-time.After(5 * time.Second):
        return Result{}, errors.New("timed out")
    }
}
```

#### Semaphore — limit concurrency with a buffered channel

```go
// Buffer size = max concurrent operations
sem := make(chan struct{}, 10)

for _, job := range jobs {
    job := job
    sem <- struct{}{}   // acquire — blocks when 10 goroutines are running
    go func() {
        defer func() { <-sem }()  // release on exit
        process(job)
    }()
}

// Wait for all to finish — fill the semaphore completely
for i := 0; i < cap(sem); i++ {
    sem <- struct{}{}
}
```

#### Pipeline — chain of stages

```go
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func main() {
    // Pipeline: generate → square → print
    for v := range square(generate(1, 2, 3, 4, 5)) {
        fmt.Println(v)  // 1, 4, 9, 16, 25
    }
}
```

Each stage is independent. Each runs in its own goroutine. Data flows through channels between stages. Adding a new transformation is adding a new function and inserting it in the chain.

### Gotchas Summary

|Gotcha|What happens|Fix|
|---|---|---|
|Send to closed channel|Panic immediately|Only sender closes; use sync if multiple senders|
|Close nil channel|Panic immediately|Always initialise channels with make|
|Close already-closed channel|Panic immediately|Coordinate with sync.Once or mutex|
|Receive from nil channel|Blocks forever|Ensure channels are initialised before use|
|Forgetting to close|`range` loops hang forever waiting|Always `defer close(ch)` in the producer|
|Receiving zero values from closed channel without ok check|Silently processes zero values as real data|Always use `v, ok := <-ch` or `range`|
|Buffered channel masking slow consumer|Buffer fills under load, blocking resumes|Treat buffer as burst absorber not a fix|
|Goroutine waiting on channel with no sender|Goroutine leaks forever|Every blocked goroutine needs a guaranteed unblock path|
|Assuming select case order|Deterministic selection never guaranteed|Design so any case order is correct|
|Nil channel in select forgotten|Appears disabled but you need the data|Only nil a channel when you're genuinely done with it|

---

### Summary

|Concept|Key Point|
|---|---|
|Channel philosophy|Transfer ownership, don't share memory|
|Unbuffered|Synchronous rendezvous — sender blocks until receiver ready|
|Buffered|Async up to capacity — sender blocks only when buffer full|
|`hchan`|Circular buffer + two wait queues (sendq, recvq) + lock|
|Directional channels|`chan<-` send-only, `<-chan` receive-only — enforced at compile time|
|Closing|Sender closes, never receiver. Panic on nil or double-close|
|Closed channel receive|Returns zero value immediately, forever — used for broadcast|
|`range` over channel|Drains until closed — always need `close()` in producer|
|Nil channel|Blocks forever — disables select cases dynamically|
|`select` with multiple ready|Picks uniformly at random — no starvation guarantee|

# Session 1C — Select in depth
---

`select` is the heart of Go's concurrency model. Every non-trivial concurrent program uses it. But most developers understand it superficially — "it's like a switch for channels" — and that surface understanding breaks down when behaviour gets subtle.

The deeper mental model is this: `select` is a **coordination primitive**. It lets a single goroutine participate in multiple channel operations simultaneously, proceeding with whichever one becomes possible first. Without `select`, a goroutine can only wait on one channel at a time. With `select`, a goroutine can watch any number of channels and respond to whichever fires.

This is the fundamental tool for expressing "wait for any of these things to happen." Every timeout pattern, every cancellation pattern, every fan-in pattern, every backpressure mechanism in Go is built on this single primitive.

### How select Works — Mechanically

When the Go runtime executes a `select` statement, it does the following:

**Step 1 — Evaluate all channel expressions and values.** All channel operands and send values are evaluated exactly once before any case is checked. This happens left to right, top to bottom. The evaluation is not conditional.

```go
select {
case ch1 <- computeValue():  // computeValue() is called unconditionally
case v := <-ch2:
}
```

**Step 2 — Check which cases are immediately ready.** A case is ready if:

- A send case: the channel has space (buffered with room) or a receiver is waiting
- A receive case: the channel has data or a sender is waiting
- A receive-from-closed case: always ready

**Step 3 — If one or more cases are ready, pick one uniformly at random and execute it.**

**Step 4 — If no cases are ready and there is a default case, execute default immediately.**

**Step 5 — If no cases are ready and there is no default, block until at least one case becomes ready, then pick one uniformly at random.**

The randomness in steps 3 and 5 is deliberate and important. It prevents any one case from being permanently starved. If two channels are always ready, over many iterations each gets selected approximately half the time.

### Basic select — waiting on multiple channels

```go
func fanIn(ch1, ch2 <-chan string) <-chan string {
    out := make(chan string)
    go func() {
        defer close(out)
        for {
            select {
            case v, ok := <-ch1:
                if !ok {
                    ch1 = nil  // disable this case
                    continue
                }
                out <- v
            case v, ok := <-ch2:
                if !ok {
                    ch2 = nil
                    continue
                }
                out <- v
            }
            if ch1 == nil && ch2 == nil {
                return
            }
        }
    }()
    return out
}
```

Notice the nil channel trick from Session 1B in action here — when a channel closes, we set it to nil to disable that case, rather than breaking out of the loop prematurely.

#### The default Case — non-blocking operations

A `select` with a `default` case never blocks. If no channel operation is immediately ready, `default` executes.

```go
// Non-blocking receive
func tryReceive(ch <-chan int) (int, bool) {
    select {
    case v := <-ch:
        return v, true
    default:
        return 0, false  // nothing available right now
    }
}

// Non-blocking send
func trySend(ch chan<- int, v int) bool {
    select {
    case ch <- v:
        return true
    default:
        return false  // channel full or no receiver
    }
}
```

**The most common misuse of default** — polling in a tight loop:

```go
// BAD — spins the CPU doing nothing useful
for {
    select {
    case v := <-ch:
        process(v)
    default:
        // nothing to do... keep spinning
    }
}

// GOOD — block until there is work
for v := range ch {
    process(v)
}

// GOOD — block with a timeout if you need periodic checks
ticker := time.NewTicker(100 * time.Millisecond)
defer ticker.Stop()
for {
    select {
    case v := <-ch:
        process(v)
    case <-ticker.C:
        doPeriodicCheck()
    }
}
```

`default` is correct when you have genuinely useful work to do when no channel is ready, or when you want exactly one non-blocking attempt. It is wrong when you use it to avoid blocking and end up in a spin loop.

### Timeout Pattern
This is the most frequently used select pattern in production code. Every outbound call — HTTP, database, gRPC — should have a timeout.

#### With `time.After`

```go
func fetchWithTimeout(ch <-chan Result) (Result, error) {
    select {
    case result := <-ch:
        return result, nil
    case <-time.After(5 * time.Second):
        return Result{}, errors.New("operation timed out")
    }
}
```

`time.After(d)` returns a `<-chan time.Time` that receives the current time after duration `d`. The channel is created fresh on every call. This is fine for one-off timeouts. It is a problem in loops — each iteration allocates a new timer that lives until it fires, even if the select chose a different case first.

```go
// LEAKS in a loop — new timer every iteration
for {
    select {
    case job := <-jobs:
        process(job)
    case <-time.After(30 * time.Second):  // new allocation every iteration
        checkHealth()
    }
}

// CORRECT in a loop — reuse the timer
timer := time.NewTimer(30 * time.Second)
defer timer.Stop()
for {
    select {
    case job := <-jobs:
        if !timer.Stop() {
            select {
            case <-timer.C:
            default:
            }
        }
        timer.Reset(30 * time.Second)
        process(job)
    case <-timer.C:
        checkHealth()
        timer.Reset(30 * time.Second)
    }
}
```

#### With context deadline — the production pattern

In real code, you rarely use raw `time.After` for timeouts. You use context:

```go
func fetchUser(ctx context.Context, id int) (*User, error) {
    resultCh := make(chan *User, 1)
    errCh    := make(chan error, 1)

    go func() {
        user, err := db.QueryUser(id)
        if err != nil {
            errCh <- err
            return
        }
        resultCh <- user
    }()

    select {
    case user := <-resultCh:
        return user, nil
    case err := <-errCh:
        return nil, err
    case <-ctx.Done():
        // Context cancelled or deadline exceeded
        // The goroutine above may still be running — this is a goroutine leak
        // unless db.QueryUser respects context too
        return nil, ctx.Err()
    }
}
```

Notice the comment about goroutine leaks. This is a real problem. The goroutine calling `db.QueryUser` has no way to know the caller gave up. It will run to completion even though nobody is reading the result. If `db.QueryUser` accepts a context, pass it:

```go
go func() {
    user, err := db.QueryUser(ctx, id)  // respects cancellation
    if err != nil {
        errCh <- err
        return
    }
    resultCh <- user
}()
```

Now when the context is cancelled, `db.QueryUser` returns early, the goroutine exits, and there is no leak. This is why context propagation is not optional — it is the mechanism that makes cancellation actually work through the entire call stack.

### Cancellation Pattern — done channel

Before context existed, the done channel pattern was the standard way to signal cancellation to goroutines. It is still used in lower-level code and is worth understanding thoroughly because context is built on the same concept.

```go
// Done channel — closed to signal all goroutines to stop
func startWorkers(done <-chan struct{}, jobs <-chan Job) {
    for i := 0; i < 5; i++ {
        go func() {
            for {
                select {
                case <-done:
                    return  // clean exit
                case job, ok := <-jobs:
                    if !ok {
                        return  // jobs channel closed
                    }
                    process(job)
                }
            }
        }()
    }
}

done := make(chan struct{})
jobs := make(chan Job, 100)

startWorkers(done, jobs)

// Later — stop all workers
close(done)  // broadcasts to all workers simultaneously
```

The key insight: `close(done)` is a broadcast. All goroutines blocked on `case <-done:` unblock simultaneously. This is why the done channel is always a `chan struct{}` closed rather than a `chan bool` sent to — you want broadcast semantics, and closing a channel achieves that; sending a value wakes only one receiver.

### For-Select Loop — the canonical goroutine body

Almost every long-running goroutine in Go has this exact structure:
```go
func worker(ctx context.Context, jobs <-chan Job, results chan<- Result) {
    for {
        select {
        case <-ctx.Done():
            // Cleanup if needed
            return

        case job, ok := <-jobs:
            if !ok {
                // Jobs channel closed — no more work
                return
            }
            result, err := process(job)
            if err != nil {
                log.Printf("processing job %v: %v", job.ID, err)
                continue
            }
            // Send result — also needs to respect cancellation
            select {
            case results <- result:
            case <-ctx.Done():
                return
            }
        }
    }
}
```

Notice the inner select for the result send. If you just write `results <- result`, and the results channel is full and the context gets cancelled, the goroutine is stuck. Every channel operation in a long-running goroutine that might block should check cancellation.

### Nil Channel in select — disabling cases dynamically

Covered in Session 1B but critical enough to revisit in the context of select. A nil channel case is never selected — it is as if that case doesn't exist. This lets you enable and disable cases dynamically at runtime.

```go
// Rate-limited pipeline — pause input when output is backed up
func rateLimit(in <-chan int, out chan<- int, limit int) {
    ticker := time.NewTicker(time.Second / time.Duration(limit))
    defer ticker.Stop()

    var (
        pending   int
        inputCh   = in        // starts enabled
        outputCh  chan<- int  // starts disabled (nil)
    )

    for {
        select {
        case v, ok := <-inputCh:
            if !ok {
                inputCh = nil  // disable input when closed
                continue
            }
            pending++
            _ = v
            // Enable output when we have work
            if outputCh == nil {
                outputCh = out
            }

        case <-ticker.C:
            if pending > 0 {
                outputCh <- pending  // send
                pending--
                if pending == 0 {
                    outputCh = nil  // disable output when nothing to send
                }
            }
        }

        if inputCh == nil && pending == 0 {
            return
        }
    }
}
```

The nil pattern turns a select from a static structure into a dynamic one. You can add and remove "active" channels at runtime by setting them to nil or restoring them to their real value.

### Priority Select — when random selection isn't good enough

The runtime selects randomly among ready cases. This is usually correct. But sometimes you have a genuine priority — checking cancellation should always take precedence over processing work, for example.

Random selection does not guarantee this. Under heavy load, the cancellation case might get starved:

```go
// This is NOT reliable for priority
select {
case <-ctx.Done():  // might not be selected even if ready
    return
case job := <-jobs:
    process(job)
}
```

The solution is a nested select or a double-check:

```go
// Pattern 1 — check high priority first with a dedicated select
for {
    // Always check cancellation first, non-blocking
    select {
    case <-ctx.Done():
        return
    default:
    }

    // Then do the normal select
    select {
    case <-ctx.Done():
        return
    case job := <-jobs:
        process(job)
    }
}
```

The first non-blocking select on `ctx.Done()` guarantees that if the context is already cancelled, we exit immediately without processing another job. The second select handles the blocking wait. This double-check pattern is the idiomatic way to express "cancel takes priority."

```go
// Pattern 2 — explicit priority via channel draining
for {
    select {
    case <-ctx.Done():
        return
    default:
    }

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
```

Both patterns have the same structure. The first non-blocking select is the priority check. The second blocking select is the normal wait.

### Select with Multiple Sends — the output side

Sending results also needs to participate in select when the output channel might block:

```go
func pipeline(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            result := transform(v)
            select {
            case out <- result:     // send result
            case <-ctx.Done():      // or cancel if context done
                return
            }
        }
    }()
    return out
}
```

Without the select on the send, this goroutine can block indefinitely on `out <- result` if the downstream consumer stops reading. Adding `ctx.Done()` to the send select gives the goroutine a way out.

### Empty select — blocking forever

```go
select {}  // blocks forever — goroutine is parked, zero CPU
```

This is occasionally useful in `main()` to keep the program running while goroutines do work in the background, or in tests to park a goroutine until the test ends. It is more expressive than `time.Sleep(math.MaxInt64)` and doesn't consume CPU.

```go
func main() {
    startBackgroundWorkers()
    // Keep main alive — let workers run until OS signal
    select {}
}

// Better — respond to OS signals for graceful shutdown
func main() {
    startBackgroundWorkers()
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit  // block until signal
    shutdown()
}
```

### Select Evaluation Subtlety — all expressions evaluated first

This catches people off guard. All channel expressions in a select are evaluated before the select begins waiting:
```go
func getChannel(name string) chan int {
    fmt.Println("evaluating", name)
    return make(chan int)
}

select {
case <-getChannel("first"):   // "evaluating first" printed
case <-getChannel("second"):  // "evaluating second" printed
}
// Both print even though only one case will ever be selected
```

And for send cases, the value to send is also evaluated:
```go
func expensiveValue() int {
    fmt.Println("computing")
    return 42
}

select {
case ch1 <- expensiveValue():  // expensiveValue() called unconditionally
case <-ch2:
}
```

`expensiveValue()` is called even if `ch2` fires and `ch1` is never used. If evaluating these expressions has side effects, those side effects happen regardless of which case is selected. Design accordingly.

### Common select Patterns — reference

```go
// 1. Wait for result or timeout
select {
case result := <-workCh:
    use(result)
case <-time.After(timeout):
    handleTimeout()
}

// 2. Wait for result or cancellation
select {
case result := <-workCh:
    use(result)
case <-ctx.Done():
    return ctx.Err()
}

// 3. Non-blocking check
select {
case v := <-ch:
    use(v)
default:
    // nothing ready
}

// 4. Fan-in two channels
select {
case v := <-ch1:
    handle(v)
case v := <-ch2:
    handle(v)
}

// 5. Send or cancel
select {
case out <- result:
case <-ctx.Done():
    return
}

// 6. Periodic work with cancellation
ticker := time.NewTicker(interval)
defer ticker.Stop()
for {
    select {
    case <-ctx.Done():
        return
    case <-ticker.C:
        doWork()
    case job := <-jobs:
        process(job)
    }
}

// 7. Drain a channel on shutdown
func drain(ch <-chan Job) {
    for {
        select {
        case _, ok := <-ch:
            if !ok {
                return
            }
        default:
            return  // nothing left
        }
    }
}
```

### Gotchas Summary

|Gotcha|What happens|Fix|
|---|---|---|
|`time.After` in a loop|New timer allocation every iteration, old ones live until they fire|Reuse `time.NewTimer`, reset after each use|
|`default` case in a tight loop|100% CPU spin doing nothing|Remove default, block properly, or use a ticker|
|Single case select no default|Equivalent to direct channel op — select adds no value|Just use the direct channel operation|
|Not checking cancellation on send|Goroutine blocks on output send when context cancelled|Always select on both `out <- v` and `ctx.Done()`|
|Relying on select case order for priority|Random selection — low-priority case runs when high-priority case is ready|Double-check pattern with preliminary non-blocking select|
|All expressions evaluated upfront|Side-effectful function called even if its case isn't chosen|Keep channel expressions side-effect free|
|Empty select for sleeping|Works but obscures intent|Use `<-quit` with a signal channel for main, or `time.Sleep` for delays|
|select in goroutine with no exit case|Goroutine can only exit when a channel case fires|Always include `ctx.Done()` or done channel case in long-running selects|

---

### Summary

| Concept                      | Key Point                                                            |
| ---------------------------- | -------------------------------------------------------------------- |
| select mechanics             | Evaluates all expressions first, picks randomly among ready cases    |
| No ready cases, no default   | Blocks until any case is ready                                       |
| No ready cases, with default | Executes default immediately — non-blocking                          |
| Multiple ready cases         | Uniform random selection — no starvation, no priority                |
| Timeout pattern              | `time.After` for one-shot, `time.NewTimer` in loops                  |
| Cancellation pattern         | `case <-ctx.Done()` in every blocking select                         |
| Nil channel in select        | Never selected — disables a case dynamically                         |
| Priority select              | Double-check: non-blocking select first, then blocking select        |
| Empty select                 | Blocks forever, zero CPU — useful for keeping main alive             |
| Send also needs select       | `case out <- v` with `case <-ctx.Done()` to avoid blocking on output |
