This is _perfect_ material for flashcards — your notes are already structured well, so I’ll convert them into **high-retention Q → A cards** (mix of conceptual + tricky gotchas + code-based recall).

I’ll group them by topic so you can revise in chunks.

---

# 🧠 Go Fundamentals Flashcards

---

## 🔹 1. Variables & Constants

**Q1. Where can `:=` be used in Go?**  
A. Only inside functions.

---

**Q2. What does `:=` do?**  
A. Declares + initializes + type inference.

---

**Q3. What is variable shadowing?**  
A. Inner scope variable hides outer variable with same name.

---

**Q4. Why is `:=` dangerous inside `if`?**  
A. It can silently create a new variable instead of updating the outer one.

---

**Q5. What are zero values in Go?**  
A. Default initialized values for types (e.g., int → 0, string → "", pointer → nil).

---

**Q6. Is a nil slice usable?**  
A. Yes, you can append to it.

---

**Q7. Is a nil map writable?**  
A. ❌ No — writing causes panic.

---

**Q8. How do you initialize a map?**  
A. `make(map[keyType]valueType)`

---

**Q9. What is special about untyped constants?**  
A. They adapt to context (int, float, etc.).

---

**Q10. When is `x, err := f()` valid if `x` exists?**  
A. If at least one variable (e.g., `err`) is new.

---

## 🔹 2. Pointers

**Q11. Does Go support pointer arithmetic?**  
A. ❌ No.

---

**Q12. Does Go pass by reference?**  
A. ❌ No — everything is pass-by-value (even pointers).

---

**Q13. Why do pointers still allow mutation?**  
A. Because copied pointer still points to same memory.

---

**Q14. When should you use pointers?**  
A. Avoid copying, mutate data, large structs.

---

**Q15. What is escape analysis?**  
A. Compiler decides whether variable goes to heap or stack.

---

**Q16. Is returning pointer to local variable safe?**  
A. ✅ Yes — moved to heap automatically.

---

**Q17. What does `new(int)` return?**  
A. Pointer to zero-initialized int.

---

**Q18. Why use `new()` instead of `var ptr *int`?**  
A. `new()` gives valid memory; `var ptr` is nil.

---

**Q19. Pointer receiver vs value receiver?**  
A. Pointer → mutate, Value → copy.

---

**Q20. Rule of thumb for receivers?**  
A. If one method uses pointer → all should.

---

## 🔹 3. Arrays vs Slices

**Q21. What is the key difference between array and slice?**  
A. Array = fixed size, Slice = dynamic.

---

**Q22. What is inside a slice?**  
A. Pointer + length + capacity.

---

**Q23. What happens when append exceeds capacity?**  
A. New array is allocated.

---

**Q24. What happens when append is within capacity?**  
A. Modifies underlying array.

---

**Q25. What is the slice sharing bug?**  
A. Multiple slices modify same underlying array.

---

**Q26. Why does this happen?**  
A. Slices are views, not copies.

---

**Q27. What is the “ghost append bug”?**  
A. Appending overwrites shared capacity unexpectedly.

---

**Q28. How to prevent slice mutation bugs?**  
A. Use `copy()` or full slice expression.

---

**Q29. What is full slice expression?**  
A. `a[low:high:max]` — controls capacity.

---

**Q30. Why use `a[:2:2]`?**  
A. Forces append to allocate new array.

---

**Q31. Difference: nil slice vs empty slice?**  
A. nil → `null` in JSON, empty → `[]`.

---

## 🔹 4. Maps

**Q32. What is time complexity of map lookup?**  
A. Average O(1).

---

**Q33. What types cannot be map keys?**  
A. Slices, maps, functions.

---

**Q34. How to safely check key existence?**  
A. `v, ok := m[key]`

---

**Q35. Is map iteration ordered?**  
A. ❌ No — random order.

---

**Q36. Can you take address of map value?**  
A. ❌ No.

---

**Q37. Can you modify struct field directly in map?**  
A. ❌ No — must copy → modify → reassign.

---

**Q38. Are maps thread-safe?**  
A. ❌ No.

---

**Q39. What happens if you write to nil map?**  
A. Panic.

---

## 🔹 5. Structs

**Q40. Are structs value or reference types?**  
A. Value types.

---

**Q41. What happens on struct assignment?**  
A. Full copy.

---

**Q42. What are struct tags used for?**  
A. Reflection (JSON, DB, etc.)

---

**Q43. What is Go’s alternative to inheritance?**  
A. Composition (embedding).

---

**Q44. Are structs comparable?**  
A. Yes, if all fields are comparable.

---

**Q45. What happens if struct contains slice/map and is copied?**  
A. Underlying data is shared.

---

## 🔹 6. Interfaces

**Q46. Does Go require explicit interface implementation?**  
A. ❌ No — implicit.

---

**Q47. What does an interface store internally?**  
A. (type, value)

---

**Q48. When is an interface truly nil?**  
A. When both type and value are nil.

---

**Q49. Why is this dangerous?**

```go
var p *User = nil
var i interface{} = p
```

A. `i != nil` because type is set.

---

**Q50. What is `any`?**  
A. Alias for `interface{}`.

---

**Q51. Best practice for interfaces?**  
A. Small and behavior-focused.

---

## 🔹 7. Methods

**Q52. What is a method in Go?**  
A. Function with receiver.

---

**Q53. Can value call pointer receiver method?**  
A. Sometimes — compiler auto-addresses.

---

**Q54. Can pointer call value receiver method?**  
A. ✅ Yes.

---

**Q55. Why can't value satisfy interface with pointer receiver method?**  
A. Method not in value's method set.

---

## 🔹 8. Embedding

**Q56. What is embedding?**  
A. Including struct without field name.

---

**Q57. What are promoted methods?**  
A. Methods of embedded struct accessible directly.

---

**Q58. Is embedding inheritance?**  
A. ❌ No — composition.

---

**Q59. What happens if both structs define same method?**  
A. Outer struct shadows inner.

---

## 🔹 9. Type Assertions

**Q60. What does `v := i.(T)` do?**  
A. Extracts concrete type.

---

**Q61. What happens if assertion is wrong?**  
A. Panic.

---

**Q62. Safe version?**  
A. `v, ok := i.(T)`

---

## 🔹 10. Type Switch

**Q63. What is type switch used for?**  
A. Handling multiple possible types.

---

**Q64. Where is it commonly used?**  
A. Errors, middleware, serialization.

---

## 🔹 11. Generics

**Q65. Why were generics introduced?**  
A. Avoid interface{}, reflection, duplication.

---

**Q66. What is constraint?**  
A. Restriction on type parameter.

---

**Q67. What does `~int` mean?**  
A. Any type with underlying type int.

---

**Q68. Can methods have new type parameters?**  
A. ❌ No.

---

**Q69. Workaround for generic methods?**  
A. Use standalone functions.

---

**Q70. Why no type switch on generic type?**  
A. Because type is fixed at compile time.

---

**Q71. How to do type switch in generics?**  
A. Convert to `any` first.

---

**Q72. How to return zero value of generic type?**  
A.

```go
var z T
return z
```

---

Now we’re getting into the _real_ learning zone 🔥 — these are the kinds of questions that actually expose gaps in understanding during interviews.

I’ll focus on **“looks easy but breaks your brain”** scenarios.

---

# ⚠️ Trap-Based Go Flashcards

---

## 🔥 Variables & Shadowing Traps

**Q1. What will this print?**

```go
err := doSomething()
if err != nil {
    err := doSomethingElse()
    fmt.Println(err)
}
fmt.Println(err)
```

A.

- Inside `if` → result of `doSomethingElse()`
    
- Outside → original `err` (unchanged)
    

👉 Trap: `:=` created a new variable (shadowing)

---

**Q2. Why is this dangerous in production?**  
A. You think you're updating error, but you're not — leads to silent bugs.

---

## 🔥 Nil vs Empty Traps

**Q3. What does this print?**

```go
var s []int
fmt.Println(s == nil)
```

A. `true`

---

**Q4. What about this?**

```go
s := []int{}
fmt.Println(s == nil)
```

A. `false`

---

**Q5. Why does this matter?**  
A. JSON encoding:

- nil → `null`
    
- empty → `[]`
    

---

## 🔥 Map Panic Trap

**Q6. What happens here?**

```go
var m map[string]int
m["a"] = 1
```

A. 💥 Panic

---

**Q7. Why?**  
A. Map is nil — not initialized.

---

**Q8. Why does this NOT panic?**

```go
v := m["a"]
```

A. Reads return zero value.

---

## 🔥 Pointer Illusion Trap

**Q9. What does this print?**

```go
func update(p *int) {
    p = new(int)
    *p = 100
}

func main() {
    x := 10
    update(&x)
    fmt.Println(x)
}
```

A. `10`

---

**Q10. Why didn’t it change?**  
A. Pointer itself was copied — reassignment didn’t affect original.

---

## 🔥 Slice Mutation Trap

**Q11. What will this print?**

```go
s := []int{1,2,3}
t := s[:2]
t[0] = 99
fmt.Println(s)
```

A. `[99 2 3]`

---

**Q12. Why?**  
A. Both share same underlying array.

---

## 🔥 Ghost Append Trap (VERY IMPORTANT)

**Q13. What will this print?**

```go
s1 := []int{1,2,3}
s2 := append(s1, 4)
s3 := append(s1, 5)

fmt.Println(s2)
fmt.Println(s3)
```

A. BOTH may print:

```
[1 2 3 5]
```

---

**Q14. Why did `4` disappear?**  
A. Shared capacity → overwrite.

---

## 🔥 Full Slice Defense Trap

**Q15. How to fix previous bug?**  
A.

```go
s2 := append(s1[:len(s1):len(s1)], 4)
```

👉 Forces new allocation

---

## 🔥 Struct Copy Trap

**Q16. What happens here?**

```go
type User struct {
    Scores []int
}

u1 := User{Scores: []int{1,2}}
u2 := u1

u2.Scores[0] = 99
fmt.Println(u1.Scores)
```

A. `[99 2]`

---

**Q17. Why?**  
A. Slice inside struct is shared.

---

## 🔥 Map Value Trap

**Q18. Why does this fail?**

```go
m["user"].Age = 30
```

A. Map values are not addressable.

---

**Q19. Correct approach?**  
A.

```go
u := m["user"]
u.Age = 30
m["user"] = u
```

---

## 🔥 Interface Nil Trap (MOST ASKED)

**Q20. What does this print?**

```go
var p *int = nil
var i interface{} = p

fmt.Println(i == nil)
```

A. `false`

---

**Q21. Why?**  
A. Interface = (type, value) → type is set.

---

## 🔥 Error Trap (Real-world bug)

**Q22. What happens?**

```go
func getError() error {
    var e *MyError = nil
    return e
}

if err != nil {
    fmt.Println("Error")
}
```

A. `"Error"` gets printed

---

**Q23. Fix?**  
A. Return `nil` explicitly.

---

## 🔥 Method Set Trap

**Q24. What happens here?**

```go
type S struct{}
func (s *S) Do() {}

var i interface{ Do() }

s := S{}
i = s
```

A. ❌ Compile error

---

**Q25. Why?**  
A. Value `S` does NOT have pointer receiver methods.

---

## 🔥 Loop Variable Capture Trap (VERY COMMON)

**Q26. What’s wrong?**

```go
for _, v := range arr {
    go func() {
        fmt.Println(v)
    }()
}
```

A. All goroutines print same value.

---

**Q27. Fix?**  
A.

```go
for _, v := range arr {
    v := v
    go func() {
        fmt.Println(v)
    }()
}
```

---

## 🔥 Range Pointer Trap

**Q28. What’s wrong here?**

```go
for _, v := range arr {
    ptrs = append(ptrs, &v)
}
```

A. All pointers point to same variable.

---

## 🔥 Append + Function Trap

**Q29. What will this print?**

```go
func add(s []int) {
    s = append(s, 4)
}

s := []int{1,2,3}
add(s)
fmt.Println(s)
```

A. `[1 2 3]`

---

**Q30. Why?**  
A. Slice header copied — append result not returned.

---

## 🔥 Generic Zero Value Trap

**Q31. Why is this invalid?**

```go
return nil
```

A. Generic type `T` may not be nil.

---

**Q32. Correct way?**

```go
var z T
return z
```

---

## 🔥 Type Switch on Generics Trap

**Q33. Why is this invalid?**

```go
switch v.(type)
```

A. Cannot type switch on generic type.

---

**Q34. Fix?**

```go
switch any(v).(type)
```

---

Now we’re getting into the _real_ learning zone 🔥 — these are the kinds of questions that actually expose gaps in understanding during interviews.

I’ll focus on **“looks easy but breaks your brain”** scenarios.

---

