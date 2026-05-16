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

