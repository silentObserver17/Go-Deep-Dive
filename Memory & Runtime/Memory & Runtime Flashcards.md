
### PHASE 1 — Normal Flashcards (Concept + Recall)

---

## 🔹 Stack vs Heap

**Q1. What are the two memory regions in Go?**  
A. Stack (per-goroutine, fast) and Heap (shared, GC-managed).

---

**Q2. Why is stack allocation fast?**  
A. Just moves the stack pointer (no GC, cache-friendly).

---

**Q3. Why is heap allocation slower?**  
A. Requires allocation + GC tracking.

---

**Q4. Who decides stack vs heap?**  
A. Compiler via escape analysis.

---

**Q5. When does a variable go to heap?**  
A. When it outlives the function (returned pointer, goroutine, etc.).

---

**Q6. Why are goroutines cheap?**  
A. Small dynamically growing stacks (~2KB).

---

---

## 🔹 Escape Analysis

**Q7. What is escape analysis?**  
A. Compiler decides whether variable lives on stack or heap.

---

**Q8. How to inspect escape decisions?**  
A.

```bash
go build -gcflags='-m'
```

---

**Q9. Name common escape triggers.**  
A.

- returning pointer
    
- interface boxing
    
- goroutine capture
    
- heap struct reference
    
- large slice/map allocation

---

**Q10. Why should you care about escape analysis?**  
A. Heap allocations → GC pressure → performance cost.

---

**Q11. What does `allocs/op` mean in benchmarks?**  
A. Number of heap allocations per operation.

---

---

## 🔹 Garbage Collector (GC)

**Q12. What GC algorithm does Go use?**  
A. Concurrent tri-color mark-sweep.

---

**Q13. What are the three colors in GC?**  
A.

- White → garbage
    
- Grey → discovered
    
- Black → fully scanned
    

---

**Q14. What are GC phases?**  
A.

- STW (mark setup)
    
- concurrent mark
    
- STW (mark termination)
    
- concurrent sweep
    

---

**Q15. What is a write barrier?**  
A. Ensures new pointers are marked during GC.

---

**Q16. What does `GOGC` control?**  
A. GC frequency.

---

**Q17. What does `GOMEMLIMIT` control?**  
A. Maximum memory usage (hard ceiling).

---

**Q18. Why is `GOMEMLIMIT` important in containers?**  
A. Prevents OOM kill by OS/Kubernetes.

---

---

## 🔹 sync.Pool

**Q19. What is `sync.Pool` used for?**  
A. Reusing allocations between GC cycles.

---

**Q20. What happens to pool objects during GC?**  
A. They are cleared.

---

**Q21. Is sync.Pool a cache?**  
A. ❌ No.

---

**Q22. Best use cases for sync.Pool?**  
A. Short-lived objects (buffers, request scratch space).

---

---

## 🔹 Value vs Pointer Receiver

**Q23. What does value receiver do?**  
A. Copies struct.

---

**Q24. What does pointer receiver do?**  
A. Passes reference, allows mutation.

---

**Q25. When to use pointer receiver?**  
A.

- large struct
    
- mutation needed
    
- consistency
    

---

**Q26. Why are value receivers safer in concurrency?**  
A. Work on copies (no shared state).

---

---

## 🔹 Memory Alignment

**Q27. What is memory alignment?**  
A. Placing data at addresses aligned to CPU expectations.

---

**Q28. Why does alignment matter?**  
A. Misalignment → slower memory access.

---

**Q29. How to reduce padding in structs?**  
A. Order fields largest → smallest.

---

**Q30. When does alignment optimization matter?**  
A. Large-scale allocations (millions of structs).

---

---

## 🔹 Zero-Copy Techniques

**Q31. What is zero-copy?**  
A. Avoiding memory duplication.

---

**Q32. Why use `json.NewDecoder` instead of `ReadAll`?**  
A. Streaming → avoids full buffer allocation.

---

**Q33. Why use `strings.Builder`?**  
A. Avoid repeated allocations.

---

**Q34. What does `sync.Pool` optimize?**  
A. Allocation reuse.

---

**Q35. What does `io.Pipe` do?**  
A. Connects reader/writer without buffering.

---

---

## 🔹 String ↔ Byte Conversion

**Q36. Why does `[]byte(s)` allocate?**  
A. Copies string → mutable slice.

---

**Q37. Why does `string(b)` allocate?**  
A. Copies slice → immutable string.

---

**Q38. When should you use `[]byte`?**  
A. I/O, mutation.

---

**Q39. When should you use `string`?**  
A. Text processing.

---

---

## 🔹 Goroutine Stack

**Q40. How does goroutine stack grow?**  
A. New larger stack allocated → old stack copied.

---

**Q41. Why are stack pointers not stable?**  
A. Stack can move during growth.

---

---

## 🔹 runtime.ReadMemStats

**Q42. What does `HeapAlloc` mean?**  
A. Live heap memory.

---

**Q43. What does `TotalAlloc` mean?**  
A. Total allocated since start.

---

**Q44. Does ReadMemStats cause STW?**  
A. ✅ Yes.

---

---

# ⚠️ PHASE 2 — Trap-Based Flashcards (INTERVIEW LEVEL)

---

## 🔥 Escape Trap

**Q1. Does this escape? Why?**

```go
func f() []int {
    s := make([]int, 10)
    return s
}
```

A. ✅ Yes — slice header returned → backing array escapes.

---

## 🔥 Interface Boxing Trap

**Q2. What happens here?**

```go
func g(v interface{}) {}
x := 10
g(x)
```

A. `x` escapes (boxed into interface).

---

## 🔥 Goroutine Capture Trap

**Q3. Does this escape?**

```go
func f() {
    x := 10
    go func() {
        fmt.Println(x)
    }()
}
```

A. ✅ Yes — captured by goroutine.

---

## 🔥 Slice Memory Leak Trap

**Q4. What’s wrong?**

```go
data := readBigFile()
small := data[:10]
```

A. Entire big array stays in memory.

---

**Q5. Fix?**

```go
small := append([]byte(nil), data[:10]...)
```

---

## 🔥 sync.Pool Trap

**Q6. Why is this wrong thinking?**

> “Pool stores objects for reuse forever”

A. GC clears pool every cycle.

---

## 🔥 GC Tuning Trap

**Q7. What happens if `GOGC=off` without limit?**  
A. Memory can grow → OOM crash.

---

**Q8. Correct setup?**  
A.

```bash
GOGC=off + GOMEMLIMIT
```

---

## 🔥 Pointer Receiver Trap

**Q9. Why is pointer receiver faster for large struct?**  
A. Avoids copying large memory.

---

**Q10. Why can pointer be slower sometimes?**  
A. Pointer chasing (cache miss).

---

## 🔥 Alignment Trap

**Q11. Which struct is better?**

```go
type A struct {
    bool
    int64
}
```

vs

```go
type B struct {
    int64
    bool
}
```

A. `B` — less padding.

---

## 🔥 String Conversion Trap

**Q12. What’s wrong here?**

```go
for _, d := range data {
    if strings.Contains(string(d), "x") {}
}
```

A. Allocates every iteration.

---

**Q13. Fix?**

```go
bytes.Contains(d, []byte("x"))
```

---

## 🔥 Unsafe Trap

**Q14. What’s dangerous here?**

```go
b := stringToBytes(s)
b[0] = 'x'
```

A. Violates string immutability → undefined behavior.

---

## 🔥 Stack Growth Trap

**Q15. Why is storing stack pointer dangerous?**  
A. Stack can move → pointer invalid.

---

## 🔥 Benchmark Trap

**Q16. What does this mean?**

```
48 B/op   1 allocs/op
```

A. Each operation allocates 48 bytes once.

---

## 🔥 ReadMemStats Trap

**Q17. Why not call in hot path?**  
A. Causes STW pause.

---

## 🔥 Zero-Copy Trap

**Q18. What’s wrong here?**

```go
body, _ := io.ReadAll(resp.Body)
```

A. Loads entire payload → high memory usage.

---

## 🔥 Slice Trim Trap

**Q19. Is this zero-copy?**

```go
return s[1:]
```

A. ✅ Yes — but may retain large underlying array.

---

---

# 🧠 Final Verdict (after update)

👉 Your notes are now:

- ✅ **Conceptually strong**
    
- ✅ **Performance-aware**
    
- ✅ **Interview-ready (with traps added)**
    
