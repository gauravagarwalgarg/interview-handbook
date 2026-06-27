# Go Coding & Output Questions

> Real Go interview gotchas from Google, Uber, Cloudflare, and distributed systems roles.
> Focus: goroutines, channels, interfaces, slices, and "What does this print?"

---

### Q1: Goroutine Leak Level: SDE 2

**Question:**
```go
package main

import (
    "fmt"
    "time"
)

func fetch(url string) <-chan string {
    ch := make(chan string)
    go func() {
        time.Sleep(2 * time.Second)
        ch <- "result from " + url
    }()
    return ch
}

func main() {
    ch1 := fetch("url1")
    ch2 := fetch("url2")
    // Only use the first result
    select {
    case r := <-ch1:
        fmt.Println(r)
    case r := <-ch2:
        fmt.Println(r)
    }
    // What happens to the other goroutine?
    time.Sleep(3 * time.Second)
}
```

**Answer:**
The "losing" goroutine **leaks** it blocks forever on `ch <- "result"` because nobody reads from the unbuffered channel after `select` picks one.

**Fix:** Use buffered channels `make(chan string, 1)` so the goroutine can send and exit even if nobody reads.

**Follow-up:** How to detect goroutine leaks in tests? → Use `goleak` package from Uber: `defer goleak.VerifyNone(t)`.

---

### Q2: Channel Deadlock Level: SDE 2

**Question:**
```go
package main

import "fmt"

func main() {
    ch := make(chan int)
    ch <- 42
    fmt.Println(<-ch)
}
```

**Answer:**
**Deadlock.** `ch <- 42` blocks because unbuffered channels require a concurrent receiver. Main goroutine is the only goroutine → all goroutines blocked → `fatal error: all goroutines are asleep`.

**Fix:** Either use `go func() { ch <- 42 }()` or `make(chan int, 1)`.

**Follow-up:** Does the Go runtime detect all deadlocks? → Only when ALL goroutines are blocked. If one goroutine is in `time.Sleep`, it won't detect it.

---

### Q3: Slice Append Shared Backing Array Level: SDE 2

**Question:**
```go
package main

import "fmt"

func main() {
    s := make([]int, 0, 5)
    s = append(s, 1, 2, 3)
    
    a := append(s, 4)
    b := append(s, 5)
    
    fmt.Println(a)
    fmt.Println(b)
    fmt.Println(s)
}
```

**Answer:**
```
[1 2 3 5]
[1 2 3 5]
[1 2 3]
```
Both `a` and `b` share the same backing array (capacity 5, length 3 → room to append without reallocation). `b`'s append overwrites `a`'s element at index 3.

**Fix:** `a := append(s[:len(s):len(s)], 4)` set capacity = length to force copy.

**Follow-up:** When does `append` allocate a new array? → When `len == cap`.

---

### Q4: nil Interface vs nil Pointer Level: SDE 3

**Question:**
```go
package main

import "fmt"

type MyError struct{}

func (e *MyError) Error() string { return "error" }

func getError() error {
    var err *MyError = nil
    return err  // returning typed nil pointer as interface
}

func main() {
    err := getError()
    fmt.Println(err == nil)
    fmt.Printf("type=%T, value=%v\n", err, err)
}
```

**Answer:**
```
false
type=*main.MyError, value=<nil>
```
An interface is `(type, value)`. Returning a typed nil `*MyError` creates interface `(*MyError, nil)` which is **not** equal to `nil` interface `(nil, nil)`.

**Fix:** Return `nil` explicitly: `if err == nil { return nil }`.

**Follow-up:** How to check if an interface holds a nil value? → Use `reflect.ValueOf(err).IsNil()` (panics if not pointer type).

---

### Q5: Defer Evaluation Order Level: SDE 2

**Question:**
```go
package main

import "fmt"

func main() {
    x := 0
    defer fmt.Println("defer 1:", x)  // x captured by value NOW
    x++
    defer fmt.Println("defer 2:", x)
    x++
    fmt.Println("main:", x)
}
```

**Answer:**
```
main: 2
defer 2: 1
defer 1: 0
```
Defers execute LIFO. Arguments are evaluated at the `defer` statement, not at execution time. So `x` is captured as 0 and 1 respectively.

**Follow-up:** How to capture the final value? → Use closure: `defer func() { fmt.Println(x) }()`.

---

### Q6: Map Concurrent Write Panic Level: SDE 2

**Question:**
```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    m := make(map[int]int)
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            m[n] = n * n  // concurrent write!
        }(i)
    }
    wg.Wait()
    fmt.Println(len(m))
}
```

**Answer:**
**`fatal error: concurrent map writes`** Go's runtime detects concurrent map access and panics deliberately (since Go 1.6).

**Fix:** Use `sync.Mutex` to protect the map, or use `sync.Map` for read-heavy workloads.

**Follow-up:** When is `sync.Map` better than mutex+map? → Many goroutines reading, few writing; or disjoint key sets per goroutine.

---

### Q7: Range Loop Pointer Aliasing (pre-1.22) Level: SDE 2

**Question:**
```go
package main

import "fmt"

type Item struct{ Name string }

func main() {
    items := []Item{{"a"}, {"b"}, {"c"}}
    ptrs := make([]*Item, 0)
    
    for _, item := range items {
        ptrs = append(ptrs, &item)  // Bug!
    }
    
    for _, p := range ptrs {
        fmt.Println(p.Name)
    }
}
```

**Answer (Go < 1.22):**
```
c
c
c
```
`item` is a single loop variable reused each iteration. All pointers point to the same address (last value).

**Go 1.22+:** Each iteration gets a new variable. Output: `a b c`.

**Fix (pre-1.22):** `item := item` inside the loop, or `ptrs = append(ptrs, &items[i])`.

**Follow-up:** How does Go 1.22 implement per-iteration variables? → Compiler allocates a new variable per iteration.

---

### Q8: Interface Method Sets Value vs Pointer Receiver Level: SDE 3

**Question:**
```go
package main

import "fmt"

type Speaker interface {
    Speak() string
}

type Dog struct{ Name string }

func (d *Dog) Speak() string { return d.Name + " barks" }

func main() {
    var s Speaker
    d := Dog{Name: "Rex"}
    // s = d   // Does this compile?
    s = &d     // Does this compile?
    fmt.Println(s.Speak())
}
```

**Answer:**
- `s = d` → **compile error**: `Dog does not implement Speaker` (method has pointer receiver)
- `s = &d` → ✅ compiles

A **pointer receiver** method is only in the method set of `*T`, not `T`. A **value receiver** method is in the method set of both `T` and `*T`.

**Follow-up:** Why this asymmetry? → A value might not be addressable (e.g., return value, map element). Can't get pointer to call pointer receiver method.

---

### Q9: Context Cancellation Propagation Level: SDE 3

**Question:**
```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context, id int) {
    select {
    case <-time.After(5 * time.Second):
        fmt.Printf("worker %d: done\n", id)
    case <-ctx.Done():
        fmt.Printf("worker %d: cancelled (%v)\n", id, ctx.Err())
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()
    
    go worker(ctx, 1)
    go worker(ctx, 2)
    
    time.Sleep(2 * time.Second)
}
```

**Answer:**
```
worker 1: cancelled (context deadline exceeded)
worker 2: cancelled (context deadline exceeded)
```
(order may vary)

Context timeout fires after 1s. Both workers receive cancellation signal on `ctx.Done()`.

**Follow-up:** Does `time.After` leak here? → Yes! It creates a timer that isn't GC'd until it fires. Use `time.NewTimer` + `timer.Stop()`.

---

### Q10: select Random Case Selection Level: SDE 2

**Question:**
```go
package main

import "fmt"

func main() {
    ch1 := make(chan string, 1)
    ch2 := make(chan string, 1)
    ch1 <- "one"
    ch2 <- "two"
    
    select {
    case msg := <-ch1:
        fmt.Println(msg)
    case msg := <-ch2:
        fmt.Println(msg)
    }
}
```

**Answer:**
Either `"one"` or `"two"` **non-deterministic**. When multiple cases are ready, Go's `select` picks one at random (uniformly).

**Follow-up:** How to prioritize one channel? → Use nested selects or check the priority channel in a `default`-free select first.

---

### Q11: Struct Embedding and Method Promotion Level: SDE 2

**Question:**
```go
package main

import "fmt"

type Logger struct{}
func (l Logger) Log(msg string) { fmt.Println("LOG:", msg) }

type Server struct {
    Logger  // embedded
    Name string
}

func (s Server) Log(msg string) { fmt.Println("SERVER:", msg) }

func main() {
    s := Server{Name: "app"}
    s.Log("starting")
    s.Logger.Log("starting")
}
```

**Answer:**
```
SERVER: starting
LOG: starting
```
`Server.Log` shadows the promoted `Logger.Log`. Direct access via `s.Logger.Log` bypasses the shadow.

**Follow-up:** Is this inheritance? → No, it's composition with syntactic sugar. No polymorphism `Logger` doesn't know about `Server`.

---

### Q12: Goroutine Scheduling with GOMAXPROCS=1 Level: SDE 3

**Question:**
```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    runtime.GOMAXPROCS(1)
    
    go func() {
        for i := 0; i < 5; i++ {
            fmt.Print("A")
        }
    }()
    
    for i := 0; i < 5; i++ {
        fmt.Print("B")
    }
}
```

**Answer:**
Most likely: `BBBBB` (or `BBBBBAAAAA` etc.). With GOMAXPROCS=1, only one goroutine runs at a time. The main goroutine runs first; the spawned goroutine may not get scheduled before main exits.

**Key insight:** There's no guarantee the goroutine runs at all before `main` returns.

**Follow-up:** Add `runtime.Gosched()` in the loop what changes? → Yields to scheduler, giving the other goroutine a chance to run.

---

### Q13: sync.WaitGroup Misuse Level: SDE 2

**Question:**
```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        go func(n int) {
            wg.Add(1)  // Bug: Add inside goroutine!
            defer wg.Done()
            fmt.Println(n)
        }(i)
    }
    wg.Wait()
}
```

**Answer:**
**Race condition.** `wg.Wait()` might return before all `wg.Add(1)` calls execute, because goroutines may not be scheduled yet.

**Fix:** Call `wg.Add(1)` before `go func()`:
```go
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(n int) { defer wg.Done(); fmt.Println(n) }(i)
}
```

**Follow-up:** What happens if `Done()` is called more times than `Add()`? → Panic: negative WaitGroup counter.

---

### Q14: Error Wrapping and Unwrapping Level: SDE 2

**Question:**
```go
package main

import (
    "errors"
    "fmt"
)

var ErrNotFound = errors.New("not found")

func findUser(id int) error {
    return fmt.Errorf("findUser(%d): %w", id, ErrNotFound)
}

func main() {
    err := findUser(42)
    
    fmt.Println(err)
    fmt.Println(errors.Is(err, ErrNotFound))
    
    var target *MyError
    fmt.Println(errors.As(err, &target))
}

type MyError struct{ Code int }
func (e *MyError) Error() string { return "my error" }
```

**Answer:**
```
findUser(42): not found
true
false
```
`%w` wraps the error, preserving the chain. `errors.Is` walks the chain checking equality. `errors.As` walks the chain checking type fails because no `*MyError` in the chain.

**Follow-up:** Difference between `%w` and `%v` in `fmt.Errorf`? → `%v` creates a new error string; `%w` wraps preserving the original for `Is`/`As`.

---

### Q15: Channel Direction Constraints Level: SDE 2

**Question:**
```go
package main

import "fmt"

func producer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)
}

func consumer(ch <-chan int) {
    for v := range ch {
        fmt.Print(v, " ")
    }
    fmt.Println()
}

func main() {
    ch := make(chan int, 5)
    producer(ch)
    consumer(ch)
}
```

**Answer:**
```
0 1 2 3 4
```
Bidirectional channel implicitly converts to directional. `chan<-` is send-only, `<-chan` is receive-only. Compile-time enforcement prevents misuse.

**Follow-up:** Can you close a `<-chan`? → No, compile error. Only the sender (or owner) should close a channel.

---

### Q16: Panic/Recover in Goroutines Level: SDE 3

**Question:**
```go
package main

import (
    "fmt"
    "time"
)

func safeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Println("recovered:", r)
            }
        }()
        fn()
    }()
}

func main() {
    safeGo(func() {
        panic("something went wrong")
    })
    
    time.Sleep(100 * time.Millisecond)
    fmt.Println("main continues")
}
```

**Answer:**
```
recovered: something went wrong
main continues
```
`recover()` only works inside a deferred function in the same goroutine. `safeGo` wraps each goroutine with panic recovery.

**Follow-up:** What if panic happens in main goroutine without recover? → Program crashes with stack trace. What if panic in a child goroutine without recover? → Entire program crashes.

---

### Q17: Interface Nil Check After Type Assertion Level: SDE 3

**Question:**
```go
package main

import "fmt"

type Validator interface {
    Validate() error
}

type EmailValidator struct{}

func (e *EmailValidator) Validate() error { return nil }

func getValidator(kind string) Validator {
    switch kind {
    case "email":
        return &EmailValidator{}
    }
    return nil
}

func main() {
    v := getValidator("unknown")
    if v != nil {
        fmt.Println("has validator")
        fmt.Println(v.Validate())  // safe?
    } else {
        fmt.Println("no validator")
    }
}
```

**Answer:**
```
no validator
```
Here `getValidator` returns a bare `nil` (nil interface), so the check works correctly. The bug from Q4 only happens when returning a typed nil pointer.

**Follow-up:** Rewrite to return `*EmailValidator` would the nil check still work? → Depends on whether the caller's variable is typed as interface or concrete.

---

### Q18: Mutex Copy Bug Level: SDE 2

**Question:**
```go
package main

import (
    "fmt"
    "sync"
)

type SafeCounter struct {
    mu sync.Mutex
    count int
}

func (c SafeCounter) Increment() {  // value receiver = COPY
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

func main() {
    c := SafeCounter{}
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            c.Increment()
        }()
    }
    wg.Wait()
    fmt.Println(c.count)
}
```

**Answer:**
`0` (or some small number). Value receiver copies the entire struct including the mutex. Each goroutine locks a **different copy** of the mutex, and increments a copy of count that's discarded.

**Fix:** Use pointer receiver: `func (c *SafeCounter) Increment()`.

**Follow-up:** How does `go vet` help? → `go vet` detects copying of `sync.Mutex`.

---

### Q19: Slice Header vs Underlying Array Level: SDE 2

**Question:**
```go
package main

import "fmt"

func modify(s []int) {
    s[0] = 99
    s = append(s, 100)
    s[1] = 88
}

func main() {
    data := []int{1, 2, 3}
    modify(data)
    fmt.Println(data)
}
```

**Answer:**
```
[99 2 3]
```
- `s[0] = 99` modifies shared backing array → affects `data`.
- `append(s, 100)` triggers reallocation (len 3, cap 3 → new array). After this, `s` points to a new array.
- `s[1] = 88` modifies the new array, not `data`'s.

**Follow-up:** What if the slice had extra capacity? → `append` wouldn't reallocate; `s[1] = 88` would affect `data`.

---

### Q20: Type Switch vs Interface Assertion Level: SDE 2

**Question:**
```go
package main

import "fmt"

func describe(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("int: %d\n", v)
    case string:
        fmt.Printf("string: %q\n", v)
    case bool:
        fmt.Printf("bool: %t\n", v)
    default:
        fmt.Printf("unknown: %T\n", v)
    }
}

func main() {
    describe(42)
    describe("hello")
    describe(true)
    describe(3.14)
}
```

**Answer:**
```
int: 42
string: "hello"
bool: true
unknown: float64
```
Type switch extracts the concrete type. `v` is automatically the correct type in each case branch. The `default` case shows `float64` (Go's default for float literals).

**Follow-up:** What's the difference between `i.(int)` and type switch? → Assertion panics on failure (or returns ok=false with comma-ok). Type switch is safe.

---

### Q21: Goroutine Closure Variable Capture Level: SDE 2

**Question:**
```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Print(i, " ")  // captures loop variable!
        }()
    }
    wg.Wait()
    fmt.Println()
}
```

**Answer (Go < 1.22):**
Likely `5 5 5 5 5` (all goroutines see `i=5` after loop completes).

**Go 1.22+:** `0 1 2 3 4` (order varies) loop variable is per-iteration.

**Fix (pre-1.22):** Pass as argument: `go func(n int) { fmt.Print(n) }(i)`

**Follow-up:** Why did Go change this in 1.22? → The old behavior was the most common source of goroutine bugs.

---

### Q22: Empty Struct Channel Signaling Level: SDE 2

**Question:**
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    done := make(chan struct{})
    
    go func() {
        time.Sleep(1 * time.Second)
        fmt.Println("work done")
        close(done)
    }()
    
    <-done  // blocks until closed
    <-done  // does this panic?
    fmt.Println("all done")
}
```

**Answer:**
```
work done
all done
```
Reading from a closed channel returns the zero value immediately (`struct{}{}` in this case). Multiple reads from a closed channel are safe and non-blocking.

**Follow-up:** Why `chan struct{}` instead of `chan bool`? → Zero memory allocation. `struct{}` has size 0.

---

### Q23: init() Function Execution Order Level: SDE 2

**Question:**
```go
// main.go
package main

import (
    "fmt"
    _ "myapp/pkg" // blank import
)

func init() {
    fmt.Println("main init 1")
}

func init() {
    fmt.Println("main init 2")
}

func main() {
    fmt.Println("main")
}

// pkg/pkg.go
package pkg

import "fmt"

func init() {
    fmt.Println("pkg init")
}
```

**Answer:**
```
pkg init
main init 1
main init 2
main
```
`init()` functions run: imported packages first (depth-first), then current package in source order. Multiple `init()` in one file run in order.

**Follow-up:** Can `init()` be called explicitly? → No, it's only called by the runtime automatically.

---

### Q24: Deferred Closures and Named Returns Level: SDE 3

**Question:**
```go
package main

import "fmt"

func calculate() (result int) {
    defer func() {
        result *= 2
    }()
    return 5
}

func main() {
    fmt.Println(calculate())
}
```

**Answer:**
```
10
```
`return 5` sets `result = 5`. The deferred function then modifies the named return value: `result *= 2 → 10`. Named returns + defer = post-processing pattern.

**Follow-up:** Use case? → Error enrichment: `defer func() { if err != nil { err = fmt.Errorf("wrap: %w", err) } }()`.

---

### Q25: String Immutability and Conversion Cost Level: SDE 2

**Question:**
```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    s := "hello"
    b := []byte(s)
    b[0] = 'H'
    
    fmt.Println(s)
    fmt.Println(string(b))
    fmt.Println(unsafe.Sizeof(s))
}
```

**Answer:**
```
hello
Hello
16
```
`[]byte(s)` creates a **copy** strings are immutable. Modifying `b` doesn't affect `s`. `unsafe.Sizeof(string)` = 16 bytes (pointer + length on 64-bit).

**Follow-up:** How to avoid copy for read-only access? → `unsafe.Slice(unsafe.StringData(s), len(s))` (Go 1.20+), but dangerous.

---

### Q26: Ticker Leak Level: SDE 2

**Question:**
```go
package main

import (
    "fmt"
    "time"
)

func processForever() {
    for range time.Tick(1 * time.Second) {  // LEAK!
        fmt.Println("tick")
    }
}

func main() {
    go processForever()
    time.Sleep(3500 * time.Millisecond)
    // processForever goroutine now leaked
    fmt.Println("exiting")
}
```

**Answer:**
```
tick
tick
tick
exiting
```
`time.Tick` creates a Ticker that's **never stopped** it leaks. The goroutine also leaks since nothing signals it to stop.

**Fix:** Use `time.NewTicker` + `defer ticker.Stop()` + context for cancellation.

**Follow-up:** How many resources does a leaked Ticker consume? → One goroutine + one channel + periodic timer wakeups.
