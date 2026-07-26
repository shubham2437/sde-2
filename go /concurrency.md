# Go Concurrency — Beginner to Advanced
### (Goroutines, Channels, select, sync & context)

A friendly, detailed guide to doing many things at once in Go — the language built
for concurrency. If you've seen the Python concurrency guide, this is the real thing
Python is always compared *to*.

> **Part 1 (sections 0–10)** — beginner → intermediate foundation.
> **Part 2 (sections 11–22)** — advanced: patterns, context, sync toolkit, races,
> worker pools, pipelines, and a real project.
>
> Run any snippet inside `package main` + `func main()` → `go run main.go`.
> Detect race bugs with `go run -race main.go`.

---

## 0. The big picture first

Go's superpower: concurrency is built into the language, not bolted on.

- **Goroutine** — an incredibly cheap "thread" managed by the Go runtime (not the OS).
  You can run *hundreds of thousands* of them. Start one with just `go f()`.
- **Channel** — a typed pipe to safely send values between goroutines.
- **`select`** — wait on multiple channel operations at once.
- The runtime **multiplexes** goroutines onto real OS threads across all CPU cores —
  so you get concurrency **and** parallelism for free. No GIL, unlike Python.

Go's motto:
> **"Don't communicate by sharing memory; share memory by communicating."**
> i.e. prefer **channels** to pass data, over locks guarding shared variables.

---

## 1. Goroutines — `go f()`

A goroutine is a function running concurrently with everything else.

```go
package main

import (
    "fmt"
    "time"
)

func worker(name string) {
    fmt.Println(name, "starting")
    time.Sleep(time.Second)     // simulate work
    fmt.Println(name, "done")
}

func main() {
    go worker("A")              // launches concurrently, returns immediately
    go worker("B")
    time.Sleep(2 * time.Second) // crude wait so main doesn't exit first
}
```

**Critical beginner gotcha:** when `main()` returns, the program exits and **kills all
goroutines** — even unfinished ones. The `time.Sleep` above is a hack; the real way to
wait is a `WaitGroup` (next section). Never rely on `Sleep` for synchronization.

Goroutines are cheap: ~2 KB of stack to start (vs ~1 MB for an OS thread), so launching
10,000 is normal. This is why Python threads feel "heavy" by comparison.

---

## 2. Waiting properly — `sync.WaitGroup`

A `WaitGroup` counts running goroutines and lets you block until they all finish.
(This is Python's `t.join()` for all threads, or `asyncio.gather`.)

```go
import "sync"

func main() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)                 // +1 before launching
        go func(id int) {
            defer wg.Done()       // -1 when this goroutine finishes
            fmt.Println("job", id)
        }(i)                      // pass i as arg — see the closure trap below
    }

    wg.Wait()                     // blocks until the counter hits 0
    fmt.Println("all done")
}
```

Three rules: `Add` **before** `go`, `defer wg.Done()` as the first line inside, and
`Wait` once after the loop.

### The classic loop-variable trap
Before Go 1.22, `go func(){ use(i) }()` captured the *same* `i` — all goroutines saw the
final value. Fixes: pass `i` as an argument (as above), or on **Go 1.22+** the loop
variable is per-iteration so it's safe. When in doubt, pass it as an argument.

---

## 3. Channels — the heart of Go concurrency

A channel is a typed conduit. Send with `ch <- v`, receive with `v := <-ch`.
Both block until the other side is ready (for an unbuffered channel) — this is how
goroutines **synchronize**.

```go
func main() {
    ch := make(chan string)       // unbuffered channel of strings

    go func() {
        ch <- "hello from goroutine"   // send (blocks until received)
    }()

    msg := <-ch                   // receive (blocks until sent)
    fmt.Println(msg)
}
```

### Directions, closing, and ranging
```go
ch := make(chan int)

// a sender goroutine
go func() {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)                     // signal "no more values"
}()

// receive until closed
for v := range ch {               // loops until ch is closed & drained
    fmt.Println(v)
}

// the comma-ok receive tells you if the channel is closed:
v, ok := <-ch                     // ok == false if closed and empty
```
Rules: **only the sender closes** a channel, never the receiver. Sending on a closed
channel panics. Receiving from a closed channel returns the zero value immediately.

---

## 4. Buffered channels

A buffered channel holds up to N values without a receiver — sends only block when
it's full. Useful as a queue or to decouple producer/consumer speeds.

```go
ch := make(chan int, 3)   // capacity 3
ch <- 1                   // doesn't block
ch <- 2                   // doesn't block
ch <- 3                   // doesn't block
// ch <- 4                // WOULD block until someone receives
fmt.Println(<-ch)         // 1
fmt.Println(len(ch), cap(ch)) // 2 3
```

Unbuffered = synchronization (handoff). Buffered = queueing (decoupling). Choose based
on whether you want the sender to wait for the receiver.

---

## 5. `select` — wait on multiple channels

`select` blocks until *one* of its channel cases is ready. If several are ready, it
picks one at random. This is Go's signature concurrency tool (Python fakes it with
`asyncio.wait(FIRST_COMPLETED)`).

```go
select {
case v := <-ch1:
    fmt.Println("from ch1:", v)
case ch2 <- 5:
    fmt.Println("sent 5 to ch2")
case <-time.After(time.Second):     // timeout!
    fmt.Println("timed out after 1s")
default:
    fmt.Println("nothing ready right now")  // non-blocking with default
}
```

Common uses: **timeouts** (`time.After`), **non-blocking** send/receive (`default`),
and **quitting** (a `<-done` case).

---

## 6. Channels as signals — done & quit

An empty-struct channel (`chan struct{}`) costs zero memory and is the idiomatic way to
signal events. Closing it broadcasts to **all** receivers at once.

```go
done := make(chan struct{})

go func() {
    for {
        select {
        case <-done:                // fires when done is closed
            fmt.Println("stopping")
            return
        default:
            // do a chunk of work...
        }
    }
}()

time.Sleep(time.Second)
close(done)                         // broadcast "stop" to everyone watching
```

---

## 7. Sharing memory safely — `sync.Mutex`

Sometimes channels are overkill and a plain lock is clearer (e.g. a shared counter or
map). A `Mutex` ensures only one goroutine touches the data at a time.

```go
type Counter struct {
    mu sync.Mutex
    n  int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()   // always unlock, even on panic
    c.n++
}
```

Without the mutex, two goroutines doing `c.n++` race and lose updates. Prove it by
running with `go run -race main.go` — Go's built-in **race detector** flags it.

`sync.RWMutex` is a variant allowing many concurrent **readers** OR one **writer** —
use it when reads vastly outnumber writes.

---

## 8. Race conditions & the race detector

A data race = two goroutines access the same memory concurrently and at least one
writes. Results are undefined (corrupted values, crashes).

```go
// BROKEN
counter := 0
var wg sync.WaitGroup
for i := 0; i < 1000; i++ {
    wg.Add(1)
    go func() { defer wg.Done(); counter++ }()  // RACE on counter
}
wg.Wait()
fmt.Println(counter)   // rarely 1000
```
Fix with a mutex, or a channel, or `sync/atomic` (§14). Always test concurrent code
with **`-race`** — it's the single most valuable tool for Go concurrency.

---

## 9. How to choose your tool

```
Passing data / results between goroutines?      -> channels
Signaling an event / cancellation to many?      -> close(chan struct{}) or context
Guarding a shared variable/map/struct?          -> sync.Mutex (or atomic for counters)
Waiting for N goroutines to finish?             -> sync.WaitGroup
Waiting on several channels at once?            -> select
Run once (lazy init)?                            -> sync.Once
```

Rule of thumb: **reach for channels first** (share by communicating). Drop to a mutex
when you're just protecting a small piece of shared state and a channel would be clumsy.

---

## 10. Go vs Python concurrency cheat sheet

| Concept | Go | Python |
|---------|-----|--------|
| Launch concurrent work | `go f()` | `Thread(...).start()` / `asyncio.create_task` |
| Wait for all | `sync.WaitGroup` | `join()` / `asyncio.gather` |
| Channel send/recv | `ch <- v` / `<-ch` | `q.put(v)` / `q.get()` |
| Close/broadcast | `close(ch)` | put sentinel `None` / `Event.set()` |
| Buffered channel | `make(chan T, n)` | `queue.Queue(maxsize=n)` |
| Select | `select { }` | `asyncio.wait(FIRST_COMPLETED)` |
| Mutex | `sync.Mutex` | `threading.Lock` |
| Semaphore | buffered chan as tokens | `threading.Semaphore` |
| Cancellation | `context.Context` | `CancelledError` / `Event` |
| Multi-core | automatic | `multiprocessing` only |

**Biggest advantage over Python:** `go f()` gives concurrency AND multi-core parallelism
at once — no GIL, no choosing between threads/asyncio/processes. One model does it all.

---
---

# PART 2 — ADVANCED

---

## 11. `context.Context` — cancellation, deadlines, timeouts

`context` is how you cancel goroutines and propagate deadlines through a call tree —
the backbone of every real Go server. (Python analog: `asyncio` cancellation / timeouts.)

```go
import "context"

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():                 // cancelled or timed out
            fmt.Println(id, "stopping:", ctx.Err())
            return
        default:
            // do a unit of work
        }
    }
}

func main() {
    // cancel manually:
    ctx, cancel := context.WithCancel(context.Background())
    go worker(ctx, 1)
    time.Sleep(time.Second)
    cancel()                               // signals ctx.Done()

    // or auto-cancel after a timeout:
    ctx2, cancel2 := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel2()                        // ALWAYS defer cancel to avoid leaks
    go worker(ctx2, 2)
}
```
Golden rules: pass `ctx` as the **first argument** to functions, **always `defer cancel()`**,
and never store a context in a struct. This is Go's version of Python's `asyncio.timeout`
and `CancelledError`, but threaded explicitly through your code.

---

## 12. `errgroup` — concurrent tasks with error handling

`golang.org/x/sync/errgroup` runs goroutines concurrently, waits for all, and returns the
**first error** — cancelling the rest via context. This is Python's `asyncio.TaskGroup`.

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) error {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]string, len(urls))

    for i, url := range urls {
        i, url := i, url                   // capture (pre-1.22)
        g.Go(func() error {                // each returns an error
            body, err := fetch(ctx, url)
            if err != nil {
                return err                 // first error cancels the group's ctx
            }
            results[i] = body
            return nil
        })
    }
    return g.Wait()                        // waits for all; returns first error
}
```
`g.Go` ≈ Python's `tg.create_task`, and `g.Wait()` ≈ exiting the `TaskGroup` block.

---

## 13. Semaphore pattern — limit concurrency

"Run 1000 jobs but only N at a time." A **buffered channel used as tokens** is the
idiomatic Go semaphore. (Python: `asyncio.Semaphore`.)

```go
func crawl(urls []string, limit int) {
    sem := make(chan struct{}, limit)      // N tokens
    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            sem <- struct{}{}              // acquire (blocks if N in flight)
            defer func() { <-sem }()       // release
            fetch(u)
        }(url)
    }
    wg.Wait()
}
```
There's also `golang.org/x/sync/semaphore` for a weighted version with context support.

---

## 14. `sync/atomic` — lock-free counters

For simple numeric counters/flags, atomics are faster than a mutex. Go 1.19+ has typed
atomics that are hard to misuse.

```go
import "sync/atomic"

var ops atomic.Int64          // Go 1.19+ typed atomic

func worker() {
    ops.Add(1)                // atomic increment — no lock, no race
}
// read with ops.Load(); set with ops.Store(x); CAS with ops.CompareAndSwap(old,new)
```
Use atomics for counters and flags; use a mutex when you must update several fields
together as a unit.

---

## 15. The rest of the `sync` toolkit

```go
// sync.Once — run something exactly once (lazy singleton/init)
var once sync.Once
once.Do(func() { initConfig() })     // runs on first call only, even concurrently

// sync.RWMutex — many readers OR one writer
var mu sync.RWMutex
mu.RLock(); _ = cache["k"]; mu.RUnlock()   // concurrent reads OK
mu.Lock(); cache["k"] = 1; mu.Unlock()     // exclusive write

// sync.Pool — reuse temporary objects to reduce GC pressure
var bufPool = sync.Pool{New: func() any { return make([]byte, 1024) }}
buf := bufPool.Get().([]byte)
// ... use buf ...
bufPool.Put(buf)

// sync.Map — concurrent map for high-churn / disjoint-key workloads
var m sync.Map
m.Store("k", 1)
v, ok := m.Load("k")
m.Delete("k")
```

---

## 16. Pipelines — chaining stages with channels

A pipeline connects goroutines via channels, each stage transforming a stream. This is
Go's answer to Python async generators (`async for`).

```go
func gen(nums ...int) <-chan int {          // stage 1: source
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums { out <- n }
    }()
    return out
}

func square(in <-chan int) <-chan int {     // stage 2: transform
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in { out <- n * n }
    }()
    return out
}

func main() {
    for n := range square(gen(1, 2, 3, 4)) {   // 1 4 9 16
        fmt.Println(n)
    }
}
```
`<-chan int` is a **receive-only** channel type — a compile-time guarantee a stage can't
send on its input.

---

## 17. Fan-out / fan-in

**Fan-out:** multiple goroutines read from one channel to parallelize work.
**Fan-in:** merge multiple channels into one.

```go
// fan-in: merge several channels into one
func merge(cs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    for _, c := range cs {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c { out <- v }
        }(c)
    }
    go func() { wg.Wait(); close(out) }()   // close out once all inputs drain
    return out
}
```
Fan-out is just launching N goroutines that all `range` over the same input channel.

---

## 18. Worker pool — the workhorse pattern

Fixed number of workers pull jobs off a channel and push results to another. Bounds
resource use while staying fully concurrent. (Python: `ThreadPoolExecutor`.)

```go
func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for j := range jobs {                 // pulls until jobs is closed
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    var wg sync.WaitGroup

    for w := 1; w <= 3; w++ {             // 3 workers (fan-out)
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    for j := 1; j <= 9; j++ { jobs <- j } // send work
    close(jobs)                            // no more jobs

    go func() { wg.Wait(); close(results) }()  // close results when workers done
    for r := range results {               // collect (fan-in)
        fmt.Println(r)
    }
}
```
`chan<- int` is a **send-only** type. Note the pattern: close `jobs` to stop workers,
then close `results` after `wg.Wait()` so the collector loop ends.

---

## 19. Deadlocks & goroutine leaks

Go's runtime detects total deadlock and panics: `fatal error: all goroutines are asleep`.

```go
// DEADLOCK: unbuffered send with no receiver
ch := make(chan int)
ch <- 1        // blocks forever — nobody is receiving -> fatal error
```

More insidious: **goroutine leaks** — a goroutine blocked forever on a channel nobody
will ever signal. It won't crash; it just wastes memory silently.
```go
// LEAK: goroutine blocks on send, but main returns and never receives
func leak() {
    ch := make(chan int)
    go func() { ch <- 1 }()   // this goroutine is stuck forever
    // ... we return without ever doing <-ch
}
```
**Prevention:** always give blocked goroutines a way out — a `ctx.Done()` case in a
`select`, a closed channel, or a buffered channel sized so sends don't block. Consistent
lock ordering prevents mutex deadlocks, same as Python.

---

## 20. `GOMAXPROCS` & the scheduler

The Go runtime schedules goroutines onto OS threads. `GOMAXPROCS` sets how many run in
parallel (defaults to the number of CPU cores).

```go
import "runtime"
fmt.Println(runtime.NumCPU())        // cores available
fmt.Println(runtime.GOMAXPROCS(0))   // current setting (0 = just query)
```
You rarely change it. The key mental model: goroutines are **multiplexed** onto a small
pool of OS threads (the M:N scheduler), which is why millions of goroutines are feasible
and why blocking one (on real I/O) doesn't stall the others. This is exactly what
Python's GIL prevents — hence Go needs no `multiprocessing` equivalent for CPU work.

---

## 21. Testing concurrent code

```go
// Always run concurrency tests with the race detector:
//   go test -race ./...

// Benchmark parallel code:
func BenchmarkWork(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            work()
        }
    })
}
```
The **`-race`** flag instruments memory access to catch data races at runtime — it's the
one habit that saves you from the hardest Go bugs. Use it in CI.

---

## 22. Capstone — a concurrent web scraper

Ties it all together: worker pool, bounded concurrency, context timeout, error handling,
and result streaming. Compare it line-for-line with the Python asyncio capstone.

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

type Result struct {
    URL   string
    Size  int
    Err   error
}

func fetch(ctx context.Context, url string) (int, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return 0, err
    }
    resp, err := http.DefaultClient.Do(req)   // respects ctx timeout/cancel
    if err != nil {
        return 0, err
    }
    defer resp.Body.Close()
    body, err := io.ReadAll(resp.Body)
    return len(body), err
}

func main() {
    urls := make([]string, 50)
    for i := range urls {
        urls[i] = fmt.Sprintf("https://example.com/page/%d", i)
    }

    const concurrency = 10
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    jobs := make(chan string)
    results := make(chan Result)
    var wg sync.WaitGroup

    // fan-out: `concurrency` workers (this IS the semaphore — bounded pool)
    for i := 0; i < concurrency; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for url := range jobs {
                size, err := fetch(ctx, url)
                results <- Result{URL: url, Size: size, Err: err}
            }
        }()
    }

    // feed jobs
    go func() {
        defer close(jobs)
        for _, u := range urls {
            select {
            case jobs <- u:
            case <-ctx.Done():
                return
            }
        }
    }()

    // close results once all workers finish
    go func() { wg.Wait(); close(results) }()

    // fan-in: stream results as they arrive
    for r := range results {
        if r.Err != nil {
            fmt.Printf("✗ %s: %v\n", r.URL, r.Err)
        } else {
            fmt.Printf("✓ %s: %d bytes\n", r.URL, r.Size)
        }
    }
}
```
Map it to the concepts:
- **Worker pool** (§18) of 10 goroutines = bounded concurrency (the semaphore, §13).
- **`context.WithTimeout`** (§11) caps the whole run and is passed into every `fetch`.
- **`select` with `ctx.Done()`** (§5, §19) means the job feeder can't leak.
- Results **streamed** via a channel (§16–17), closed after `wg.Wait()`.

The Python asyncio version does the same with a Semaphore + `asyncio.timeout` +
`as_completed`. Same shape, different syntax — recognizing that mapping is the goal.

---

## Advanced topics — quick index

| Topic | Section | Python analog |
|-------|---------|---------------|
| context (cancel/deadline/timeout) | §11 | asyncio cancellation / `asyncio.timeout` |
| errgroup (concurrent + errors) | §12 | `asyncio.TaskGroup` |
| Semaphore (bounded concurrency) | §13 | `asyncio.Semaphore` |
| atomic counters | §14 | — (GIL makes some ops atomic) |
| Once / RWMutex / Pool / sync.Map | §15 | `Lock` / — / — / — |
| Pipelines | §16 | async generators / `async for` |
| Fan-out / fan-in | §17 | gather over tasks |
| Worker pool | §18 | `ThreadPoolExecutor` |
| Deadlocks & goroutine leaks | §19 | deadlocks / leaked tasks |
| GOMAXPROCS & scheduler | §20 | GIL / no-GIL build |
| Race detector & testing | §21 | — (no equivalent tooling) |
| Scraper capstone | §22 | asyncio scraper capstone |

---

## Setup notes

- `errgroup` and `x/sync/semaphore` live in an external module — install with:
  `go get golang.org/x/sync`
- Everything else (`sync`, `context`, `sync/atomic`, channels) is standard library.
- **Always** develop concurrent code with `go run -race` / `go test -race`.
