# Python Concurrency — Beginner to Advanced (Threads, Async & Processes)
### (with Go goroutine comparisons)

> **Part 1 (sections 0–10)** is the beginner → intermediate foundation.
> **Part 2 (sections 11–22)** is the advanced material: threading primitives,
> modern asyncio (TaskGroup, cancellation, rate-limiting), multiprocessing shared
> memory, deadlocks, the no-GIL build, and a full real-world project.

A friendly, detailed guide to doing more than one thing at a time in Python.
Because you're learning Go, each section links the Python idea back to **goroutines & channels**.

> Run examples with `python3 file.py`. Everything here is standard library — no installs.

---

## 0. The big picture first

"Concurrency" means *structuring* your program so multiple tasks make progress.
"Parallelism" means tasks *literally run at the same instant* on multiple CPU cores.

Python gives you **three** tools, and picking the right one is 90% of the battle:

| Tool | Best for | Real parallelism? | Go equivalent |
|------|----------|-------------------|---------------|
| `threading` | **I/O-bound** work (network, disk, waiting) | No (blocked by the GIL) | goroutines doing I/O |
| `asyncio` | **Many** I/O tasks, one thread, `async/await` | No (single thread) | goroutines + channels |
| `multiprocessing` | **CPU-bound** work (math, crunching) | ✅ Yes (many processes) | goroutines on multiple cores |

The one thing to burn into memory as a beginner:

> **I/O-bound** (waiting on network/disk/database) → use **threads** or **asyncio**.
> **CPU-bound** (heavy computation) → use **multiprocessing**.

---

## 1. Why Python is different from Go here (the GIL)

Go was built for concurrency: you type `go doWork()` and the Go runtime schedules
thousands of cheap **goroutines** across all your CPU cores automatically.

Python has the **GIL** (Global Interpreter Lock): a lock that lets only **one thread
run Python bytecode at a time**, even on an 8-core machine. So:

- Python **threads do NOT speed up CPU-heavy work** — they take turns.
- But threads DO help **I/O-bound** work, because a thread waiting on the network
  *releases* the GIL, letting another thread run. The waiting is what overlaps.
- To get true multi-core parallelism in Python, you use **processes** (each has its own GIL).

Go has no GIL, so goroutines give you both concurrency *and* parallelism for free.
Keep this difference in mind — it explains every design choice below.

*(Note: Python 3.13+ has an experimental "free-threaded" no-GIL build, but the default
Python you'll use still has the GIL. Learn the GIL model first.)*

---

## 2. Threads — `threading` (best for I/O-bound)

A thread is a separate line of execution inside your program. Think of it as the
closest thing to a goroutine, but heavier and GIL-limited.

### The simplest thread
```python
import threading
import time

def worker(name):
    print(f"{name} starting")
    time.sleep(1)          # simulate I/O (network/disk wait)
    print(f"{name} done")

t = threading.Thread(target=worker, args=("A",))
t.start()                  # like `go worker("A")` in Go
t.join()                   # wait for it to finish
```

`t.start()` launches it; `t.join()` blocks until it's finished (Go uses a
`sync.WaitGroup` for this).

### Running many at once
```python
threads = []
for i in range(5):
    t = threading.Thread(target=worker, args=(f"job-{i}",))
    t.start()
    threads.append(t)

for t in threads:          # wait for all — like sync.WaitGroup.Wait()
    t.join()
```

### The easier, modern way: `ThreadPoolExecutor`
You rarely manage threads by hand. Use a pool — it's cleaner and reuses threads.

```python
from concurrent.futures import ThreadPoolExecutor

def fetch(url):
    time.sleep(1)          # pretend to download
    return f"got {url}"

urls = ["a.com", "b.com", "c.com"]

with ThreadPoolExecutor(max_workers=5) as pool:
    results = list(pool.map(fetch, urls))   # runs concurrently
print(results)
```

All three "downloads" overlap → ~1 second total instead of 3.

### Returning values / handling as they finish
```python
from concurrent.futures import ThreadPoolExecutor, as_completed

with ThreadPoolExecutor() as pool:
    futures = [pool.submit(fetch, u) for u in urls]
    for f in as_completed(futures):     # yields each as it completes
        print(f.result())
```

### Go comparison
```go
// Go: the same 3 concurrent fetches
var wg sync.WaitGroup
for _, u := range urls {
    wg.Add(1)
    go func(url string) {
        defer wg.Done()
        fetch(url)
    }(u)
}
wg.Wait()
```
`go func(){}` ≈ `Thread(...).start()`, and `wg.Wait()` ≈ `t.join()` for all.

---

## 3. Sharing data safely — Locks

When two threads touch the same variable, you get **race conditions**. Example bug:

```python
counter = 0
def bad():
    global counter
    for _ in range(100000):
        counter += 1        # NOT atomic → threads clobber each other

threads = [threading.Thread(target=bad) for _ in range(2)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)              # usually NOT 200000 — race condition!
```

Fix it with a **Lock** (a mutex — same idea as Go's `sync.Mutex`):

```python
counter = 0
lock = threading.Lock()

def good():
    global counter
    for _ in range(100000):
        with lock:          # only one thread inside at a time
            counter += 1

# now counter == 200000 reliably
```

Go's philosophy: *"Don't communicate by sharing memory; share memory by communicating"*
— i.e. prefer channels over locks. Python can do that too, with a **Queue**.

---

## 4. `queue.Queue` — Python's answer to Go channels

A thread-safe queue is the cleanest way to hand work between threads — the closest
thing Python has to a Go **channel**.

```python
import queue, threading

q = queue.Queue()

def producer():
    for i in range(5):
        q.put(i)            # like  ch <- i
    q.put(None)             # sentinel = "no more items" (like close(ch))

def consumer():
    while True:
        item = q.get()      # like  v := <-ch  (blocks until available)
        if item is None:
            break
        print("processing", item)
        q.task_done()

t1 = threading.Thread(target=producer)
t2 = threading.Thread(target=consumer)
t1.start(); t2.start()
t1.join(); t2.join()
```

| Go channel | Python `queue.Queue` |
|------------|----------------------|
| `ch := make(chan int)` | `q = queue.Queue()` |
| `ch <- v` (send) | `q.put(v)` |
| `v := <-ch` (receive) | `v = q.get()` |
| `close(ch)` | put a sentinel like `None` |
| buffered `make(chan int, 3)` | `queue.Queue(maxsize=3)` |

---

## 5. Async programming — `asyncio` (best for MANY I/O tasks)

`asyncio` runs many tasks **concurrently on a single thread** using `async`/`await`.
No locks needed (one thread!), and it scales to thousands of tasks cheaply — this is
the closest *feel* to lightweight goroutines.

### Core keywords
- `async def` — defines a **coroutine** (a pausable function).
- `await` — pause here, let other tasks run, resume when the awaited thing is ready.
- The **event loop** — the scheduler that runs coroutines (like Go's runtime scheduler).

### Hello, async
```python
import asyncio

async def worker(name, secs):
    print(f"{name} starting")
    await asyncio.sleep(secs)      # non-blocking wait (releases control)
    print(f"{name} done after {secs}s")
    return name

async def main():
    # run three concurrently and wait for all:
    results = await asyncio.gather(
        worker("A", 1),
        worker("B", 2),
        worker("C", 1),
    )
    print(results)

asyncio.run(main())                # entry point — starts the event loop
```
Total time ≈ 2s (the longest), not 4s — they overlap.

### Launching tasks (fire them off like `go f()`)
```python
async def main():
    task = asyncio.create_task(worker("bg", 3))  # starts immediately
    print("task launched, doing other work...")
    await asyncio.sleep(1)
    result = await task                          # await it when you need the result
```
`asyncio.create_task(...)` is the closest analog to `go f()` — it schedules the
coroutine to run concurrently right away.

### A realistic pattern: many downloads
```python
async def fetch(session, url):
    async with session.get(url) as resp:   # needs the `aiohttp` library
        return await resp.text()

async def main():
    import aiohttp
    async with aiohttp.ClientSession() as session:
        urls = ["https://example.com"] * 100
        results = await asyncio.gather(*(fetch(session, u) for u in urls))
    print(len(results), "pages fetched")
```
100 requests overlap on one thread — very efficient for I/O.

### async queue (channel-like, for async code)
```python
async def main():
    q = asyncio.Queue()

    async def producer():
        for i in range(5):
            await q.put(i)
        await q.put(None)

    async def consumer():
        while True:
            item = await q.get()
            if item is None:
                break
            print("got", item)

    await asyncio.gather(producer(), consumer())
```

### The golden rule of asyncio
**Never call a blocking function inside async code** (e.g. `time.sleep()`,
regular `requests.get()`) — it freezes the whole event loop. Use the async
version (`await asyncio.sleep()`, `aiohttp`) or push blocking work to a thread:
```python
await asyncio.to_thread(blocking_function, arg)   # runs it in a thread pool
```

---

## 6. Multiprocessing — `multiprocessing` (best for CPU-bound)

To *actually* use multiple CPU cores in Python (bypassing the GIL), run separate
**processes**. Each process is its own Python with its own GIL → true parallelism.
This is what most resembles goroutines spread across cores for heavy computation.

```python
from concurrent.futures import ProcessPoolExecutor

def heavy(n):
    total = 0
    for i in range(n):
        total += i * i          # CPU-bound crunching
    return total

if __name__ == "__main__":      # REQUIRED guard on Windows/macOS
    with ProcessPoolExecutor() as pool:
        results = list(pool.map(heavy, [10_000_000] * 4))
    print(results)              # runs on 4 cores in parallel
```

Trade-offs vs threads:
- ✅ Real parallelism, uses all cores.
- ❌ Heavier to start; data passed between processes must be **pickled** (copied),
  so avoid sharing big objects. Communicate via `multiprocessing.Queue` or `Pipe`.

> If your bottleneck is math/CPU → processes. If it's waiting on I/O → threads or asyncio.

---

## 7. How to choose — a decision guide

```
Is your task waiting on I/O (network, disk, DB, APIs)?
│
├── YES ──> Do you have a LOT of them (hundreds/thousands)?
│           ├── YES ──> asyncio   (most scalable)
│           └── NO  ──> threading / ThreadPoolExecutor (simplest)
│
└── NO (it's heavy CPU work: math, image/video, parsing)
            └────────> multiprocessing / ProcessPoolExecutor
```

Quick gut check:
- Scraping 500 web pages → **asyncio**
- Downloading 5 files → **threads**
- Resizing 1000 images / crunching numbers → **multiprocessing**

---

## 8. Python ↔ Go concurrency cheat sheet

| Concept | Go | Python |
|---------|-----|--------|
| Launch concurrent work | `go f()` | `Thread(target=f).start()` or `asyncio.create_task(f())` |
| Wait for all | `sync.WaitGroup` | `t.join()` / `asyncio.gather()` |
| Channel (send/recv) | `ch <- v` / `<-ch` | `q.put(v)` / `q.get()` (or `asyncio.Queue`) |
| Close channel | `close(ch)` | put a sentinel (`None`) |
| Mutex | `sync.Mutex` | `threading.Lock()` |
| Select on channels | `select { ... }` | `asyncio.wait(..., FIRST_COMPLETED)` |
| Buffered channel | `make(chan T, n)` | `queue.Queue(maxsize=n)` |
| True multi-core | built-in (no GIL) | `multiprocessing` (separate processes) |
| Lightweight tasks | goroutines (cheap) | asyncio tasks (cheap, 1 thread) |

**Biggest mental shift coming from Go:** in Go, `go f()` gives you concurrency *and*
multi-core parallelism at once. In Python you must decide *which kind* you need —
threads/asyncio for overlapping I/O, or multiprocessing for real CPU parallelism —
because the GIL stops threads from running Python code truly in parallel.

---

## 9. Tiny runnable comparison (I/O-bound: threads vs asyncio)

```python
import time, threading, asyncio

def blocking_task(n):
    time.sleep(1)               # pretend I/O
    return n * 2

# --- sequential (slow: ~3s) ---
start = time.perf_counter()
seq = [blocking_task(i) for i in range(3)]
print("sequential:", time.perf_counter() - start, "s")

# --- threads (~1s) ---
from concurrent.futures import ThreadPoolExecutor
start = time.perf_counter()
with ThreadPoolExecutor() as pool:
    thr = list(pool.map(blocking_task, range(3)))
print("threads:", time.perf_counter() - start, "s")

# --- asyncio (~1s) ---
async def atask(n):
    await asyncio.sleep(1)
    return n * 2

async def main():
    return await asyncio.gather(*(atask(i) for i in range(3)))

start = time.perf_counter()
asyncio.run(main())
print("asyncio:", time.perf_counter() - start, "s")
```

You'll see sequential ≈ 3s while threads and asyncio ≈ 1s — the overlap is the whole point.

---

## 10. Common beginner mistakes

1. **Using threads for CPU work** and expecting a speedup — the GIL prevents it. Use processes.
2. **Calling `time.sleep()` / `requests.get()` inside `async` code** — it blocks the whole
   event loop. Use `await asyncio.sleep()` / an async library, or `asyncio.to_thread()`.
3. **Forgetting `if __name__ == "__main__":`** with multiprocessing — causes crashes/infinite
   spawning on Windows and macOS.
4. **Sharing mutable state without a Lock** across threads — race conditions and corrupted data.
5. **Mixing the models** — don't mentally treat threads and asyncio the same; asyncio is
   cooperative (tasks yield at `await`), threads are preemptive (the OS switches them).
6. **Expecting Python threads to be as cheap as goroutines** — they're OS threads (heavier).
   For thousands of concurrent I/O tasks, prefer asyncio.

---
---

# PART 2 — ADVANCED

Everything above gets you productive. This part covers the tools you reach for in
real systems: fine-grained synchronization, modern structured asyncio, true shared
memory across processes, avoiding deadlocks, and a complete project.

---

## 11. Advanced threading primitives

`Lock` is only the beginning. The `threading` module has a whole toolkit.

### RLock — reentrant lock (a lock the same thread can acquire twice)
A plain `Lock` deadlocks if the *same* thread tries to acquire it again (e.g. a
recursive function or a method calling another locked method). `RLock` allows it.
```python
import threading
lock = threading.RLock()

def outer():
    with lock:
        inner()          # would deadlock with a plain Lock

def inner():
    with lock:           # same thread re-acquires — OK with RLock
        ...
```

### Semaphore — allow up to N at once (rate/concurrency limiting)
A Lock allows 1; a Semaphore allows N. Perfect for "max 3 downloads at a time".
```python
sem = threading.Semaphore(3)          # at most 3 concurrent

def download(url):
    with sem:                         # blocks if 3 already inside
        ...                           # only 3 threads run this block at once
```
Go analog: a **buffered channel** used as a token bucket (`make(chan struct{}, 3)`).

### Event — one-time (or reusable) broadcast flag
Signal many threads that "something happened" (like a broadcast channel close).
```python
ready = threading.Event()

def waiter():
    ready.wait()                      # blocks until the event is set
    print("go!")

def starter():
    ready.set()                       # wakes ALL waiters at once
# ready.clear() resets it; ready.is_set() checks it
```
Go analog: closing a channel to broadcast to all receivers (`close(done)`).

### Condition — wait for a custom predicate (producer/consumer by hand)
When "is there work?" is more complex than a simple flag.
```python
cond = threading.Condition()
items = []

def consumer():
    with cond:
        cond.wait_for(lambda: len(items) > 0)   # releases lock while waiting
        item = items.pop()

def producer():
    with cond:
        items.append(42)
        cond.notify()                            # wake one waiter (notify_all for all)
```

### Barrier — make N threads all wait for each other
Everyone stops at the barrier until all N arrive, then all proceed together.
```python
barrier = threading.Barrier(3)

def phase_worker():
    do_phase_1()
    barrier.wait()        # blocks until all 3 threads reach here
    do_phase_2()          # all start phase 2 together
```

### Thread-local storage — per-thread private data
Each thread sees its own copy — no locking needed.
```python
local = threading.local()

def handler():
    local.request_id = compute_id()   # private to this thread
    process()                         # can read local.request_id safely
```

### Daemon threads — background threads that don't block exit
```python
t = threading.Thread(target=loop, daemon=True)
t.start()   # program can exit even if this is still running (no join needed)
```

---

## 12. Deadlocks — the #1 advanced-threading bug

A deadlock: thread A holds lock 1 and wants lock 2; thread B holds lock 2 and wants
lock 1. Both wait forever.
```python
# BROKEN — classic deadlock
lock_a, lock_b = threading.Lock(), threading.Lock()

def t1():
    with lock_a:
        with lock_b: ...    # wants B while holding A

def t2():
    with lock_b:
        with lock_a: ...    # wants A while holding B  -> DEADLOCK
```
**Fixes:**
1. **Lock ordering** — always acquire locks in the same global order everywhere.
2. **Timeouts** — `lock.acquire(timeout=1)` and back off if it fails.
3. **Avoid nested locks** — prefer a single lock, or a `queue.Queue` (message passing)
   instead of shared state. This is the Go philosophy: channels over shared memory.

---

## 13. Modern asyncio — `TaskGroup` (the 3.11+ way)

`asyncio.gather` is older. Since Python 3.11, **`TaskGroup`** is the recommended way to
run tasks concurrently. It's *structured concurrency*: if any task fails, the rest are
cancelled automatically, and the group waits for all before exiting.
```python
import asyncio

async def worker(n):
    await asyncio.sleep(1)
    return n * 2

async def main():
    async with asyncio.TaskGroup() as tg:      # 3.11+
        t1 = tg.create_task(worker(1))
        t2 = tg.create_task(worker(2))
    # on exiting the block, ALL tasks are guaranteed done
    print(t1.result(), t2.result())

asyncio.run(main())
```
Why it beats `gather`: with `gather`, if one task raises, the others keep running in the
background (leaks). `TaskGroup` cancels siblings and propagates errors cleanly — this is
the closest Python has to Go's structured error handling with `errgroup`.

---

## 14. Cancellation & timeouts (asyncio)

This is where async gets powerful — you can cancel work that's taking too long.

### Timeout on an operation
```python
async def main():
    try:
        async with asyncio.timeout(2):        # 3.11+; cancels after 2s
            await slow_operation()
    except TimeoutError:
        print("gave up after 2s")

# older style:
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=2)
    except asyncio.TimeoutError:
        ...
```

### Cancelling a task and handling it gracefully
```python
async def cancellable():
    try:
        while True:
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print("cleaning up before exit")      # do cleanup
        raise                                  # re-raise! (don't swallow it)

async def main():
    task = asyncio.create_task(cancellable())
    await asyncio.sleep(2.5)
    task.cancel()                              # request cancellation
    try:
        await task
    except asyncio.CancelledError:
        print("task was cancelled")
```
Go analog: `context.Context` with `ctx.Done()` — you cancel a context and goroutines
watch for it. `CancelledError` is Python's version of a cancelled `ctx`.

---

## 15. Rate-limiting async with a Semaphore

The most common real-world async need: "make 1000 requests but only 10 at a time" so
you don't overwhelm the server or your machine.
```python
async def fetch_limited(sem, session, url):
    async with sem:                            # only N run this block at once
        async with session.get(url) as resp:
            return await resp.text()

async def main():
    import aiohttp
    sem = asyncio.Semaphore(10)                # max 10 concurrent
    async with aiohttp.ClientSession() as session:
        urls = ["https://example.com"] * 1000
        results = await asyncio.gather(
            *(fetch_limited(sem, session, u) for u in urls)
        )
```

---

## 16. Async generators & `async for` / `async with`

Stream results as they arrive instead of collecting them all.
```python
async def stream_pages(n):
    for i in range(n):
        await asyncio.sleep(0.1)               # simulate paged API
        yield f"page {i}"                       # async generator

async def main():
    async for page in stream_pages(5):          # consume as they come
        print(page)
```
`async with` powers async context managers (open/close a connection pool, acquire a
lock) — you saw it with `aiohttp.ClientSession()` and `asyncio.Semaphore`.

---

## 17. Mixing async with blocking code

Real apps have blocking libraries (a DB driver, `requests`, heavy CPU). Don't let them
freeze the event loop — offload them.
```python
async def main():
    loop = asyncio.get_running_loop()

    # blocking I/O -> run in a thread pool:
    result = await asyncio.to_thread(blocking_io_function, arg)

    # CPU-bound -> run in a PROCESS pool so it actually parallelizes:
    from concurrent.futures import ProcessPoolExecutor
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy_function, arg)
```
Rule of thumb: **I/O → `to_thread`**, **CPU → `run_in_executor(ProcessPoolExecutor)`**.

---

## 18. Advanced multiprocessing — sharing data between processes

Processes don't share memory by default (each is a separate Python). To share, you
have three options, from simplest to fastest.

### Shared values & arrays
```python
from multiprocessing import Process, Value, Array

def worker(counter, arr):
    with counter.get_lock():          # built-in lock
        counter.value += 1
    arr[0] = 99

if __name__ == "__main__":
    counter = Value("i", 0)           # shared int
    arr = Array("i", [1, 2, 3])       # shared int array
    p = Process(target=worker, args=(counter, arr))
    p.start(); p.join()
    print(counter.value, arr[:])
```

### Manager — shared dict/list/etc. (higher level, slower)
```python
from multiprocessing import Manager, Process

def worker(shared):
    shared["done"] = True
    shared_list = shared["items"]
    shared_list.append(1)

if __name__ == "__main__":
    with Manager() as mgr:
        shared = mgr.dict()
        shared["items"] = mgr.list()
        p = Process(target=worker, args=(shared,))
        p.start(); p.join()
        print(dict(shared))
```

### shared_memory — fastest, zero-copy (3.8+, great with NumPy)
```python
from multiprocessing import shared_memory
import numpy as np

# create a block backed by shared memory
shm = shared_memory.SharedMemory(create=True, size=1000)
arr = np.ndarray((250,), dtype=np.int32, buffer=shm.buf)
arr[0] = 42
# ... other processes attach with SharedMemory(name=shm.name) ...
shm.close(); shm.unlink()             # always clean up
```

### Communicating: `multiprocessing.Queue` and `Pipe`
```python
from multiprocessing import Process, Queue

def producer(q):
    for i in range(5):
        q.put(i)
    q.put(None)

if __name__ == "__main__":
    q = Queue()                       # process-safe queue (pickles items)
    p = Process(target=producer, args=(q,))
    p.start()
    while (item := q.get()) is not None:
        print(item)
    p.join()
```

---

## 19. `Pool` vs `ProcessPoolExecutor`, and start methods

`multiprocessing.Pool` is the older API; `ProcessPoolExecutor` is the modern one and
integrates with `concurrent.futures` (futures, `as_completed`). Prefer the executor
unless you need `Pool`-specific features like `imap_unordered` for streaming huge inputs.
```python
from multiprocessing import Pool
with Pool(4) as p:
    for r in p.imap_unordered(heavy, range(1000)):   # stream results, low memory
        print(r)
```

**Start methods** matter on the details:
- `fork` (Linux default) — fast, copies the parent process. Can be unsafe with threads.
- `spawn` (Windows/macOS default) — clean fresh process, but re-imports your module,
  which is *why* you need `if __name__ == "__main__":`.
```python
import multiprocessing as mp
mp.set_start_method("spawn")          # set once, at program start
```

---

## 20. The no-GIL future (free-threaded Python 3.13+)

Python 3.13 introduced an experimental **free-threaded build** (PEP 703) where the GIL
can be disabled, letting threads run Python bytecode truly in parallel — much closer to
Go's goroutine model. Caveats today:
- It's a **separate build** (`python3.13t`), not the default you get from python.org installers.
- Many C-extension packages aren't ready yet, and single-thread performance can be slightly lower.
- The `threading` API is unchanged — your code looks the same; it just parallelizes.

For now: learn the GIL model (it's still the reality for 99% of deployments), but know
that this is where Python is heading. It won't change *when to use processes vs threads*
overnight, but over time threads will become viable for CPU-bound work too.

---

## 21. `concurrent.futures` deep dive — the unified API

`concurrent.futures` gives you ONE interface over both threads and processes. Swap one
line to switch execution model — great for experimenting.
```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor, as_completed

Executor = ThreadPoolExecutor        # <- change to ProcessPoolExecutor for CPU work

with Executor(max_workers=4) as pool:
    futures = {pool.submit(work, x): x for x in inputs}
    for fut in as_completed(futures):
        x = futures[fut]
        try:
            print(x, "->", fut.result(timeout=5))
        except Exception as e:
            print(x, "failed:", e)   # exceptions surface via .result()
```
Key `Future` methods: `.result()` (blocks, re-raises exceptions), `.done()`,
`.cancel()`, `.add_done_callback(fn)`. This is Python's equivalent of collecting
goroutine results through a channel with error handling.

---

## 22. Capstone project — a real concurrent web scraper

Ties it all together: bounded concurrency (Semaphore), timeouts, retries, structured
error handling, and streaming results. This is production-shaped async code.
```python
import asyncio
import aiohttp          # pip install aiohttp

async def fetch_with_retry(sem, session, url, retries=3):
    """Fetch one URL: rate-limited, timed out, retried with backoff."""
    for attempt in range(1, retries + 1):
        try:
            async with sem:                              # max-N concurrency
                async with asyncio.timeout(10):          # per-request timeout
                    async with session.get(url) as resp:
                        resp.raise_for_status()
                        text = await resp.text()
                        return url, len(text), None       # success
        except (aiohttp.ClientError, asyncio.TimeoutError) as e:
            if attempt == retries:
                return url, 0, f"failed after {retries} tries: {e}"
            await asyncio.sleep(2 ** attempt)            # exponential backoff

async def scrape(urls, concurrency=10):
    sem = asyncio.Semaphore(concurrency)
    async with aiohttp.ClientSession() as session:
        tasks = [
            asyncio.create_task(fetch_with_retry(sem, session, u))
            for u in urls
        ]
        for coro in asyncio.as_completed(tasks):         # stream results as they finish
            url, size, error = await coro
            if error:
                print(f"✗ {url}: {error}")
            else:
                print(f"✓ {url}: {size} bytes")

if __name__ == "__main__":
    urls = [f"https://example.com/page/{i}" for i in range(50)]
    asyncio.run(scrape(urls, concurrency=10))
```
What to notice, mapped to everything above:
- **Semaphore** (§15) bounds concurrency to 10 even with 50 tasks.
- **`asyncio.timeout`** (§14) caps each request; **retries with backoff** handle flakiness.
- **`as_completed`** (§16) streams results instead of waiting for the slowest.
- Structured error returns instead of exceptions crashing the batch.

In Go this same shape would be goroutines + a buffered channel as a semaphore +
`context.WithTimeout` + a `WaitGroup` — recognizing that mapping is the goal.

---

## Advanced topics — quick index

| Topic | Section | Go analog |
|-------|---------|-----------|
| RLock / Semaphore / Event / Condition / Barrier | §11 | mutex / buffered chan / close(chan) / cond / — |
| Deadlock avoidance (lock ordering, timeouts) | §12 | same principles |
| TaskGroup (structured concurrency) | §13 | `errgroup.Group` |
| Cancellation & timeouts | §14 | `context.Context` |
| Async rate-limiting | §15 | buffered channel token bucket |
| Async generators / `async for` / `async with` | §16 | range over channel |
| Offloading blocking/CPU work from async | §17 | — |
| Shared memory across processes | §18 | shared memory (rare in Go) |
| Pool vs Executor, start methods | §19 | — |
| No-GIL free-threaded Python | §20 | goroutines (native parallel) |
| `concurrent.futures` / Future objects | §21 | channels for results |
| Real scraper capstone | §22 | goroutines + sem chan + context |
