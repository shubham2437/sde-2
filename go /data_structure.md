# Go Data Structures — Create · Search · Update · Delete

A complete reference to **all 17 data structures in Go**. Each one shows the four core operations:
**Create**, **Search**, **Update**, and **Delete** (plus loop where useful).

> Run any snippet inside `func main()` in `main.go` → `go run main.go`.
> A few helpers (`slices.*`, `maps.*`) need **Go 1.21+** — check with `go version`.

---

# Built-in (part of the language)

## 1. Array — `[N]T` (fixed size)

Fixed length baked into the type; `[3]int` ≠ `[4]int`. Copied by value.

```go
// CREATE
var a [3]int              // [0 0 0]
b := [3]int{1, 2, 3}
c := [...]int{10, 20, 30} // length inferred

// SEARCH (linear — arrays can't grow, so no built-in search)
func indexOf(arr [3]int, x int) int {
    for i, v := range arr {
        if v == x { return i }
    }
    return -1
}

// UPDATE
b[0] = 99                 // set by index

// DELETE (can't shrink an array — zero the slot instead)
b[1] = 0                  // reset to zero value
```

---

## 2. Slice — `[]T` (dynamic list, used most)

```go
// CREATE
var s []int               // nil, len 0
s = append(s, 1, 2, 3)
t := []int{4, 5, 6}
u := make([]int, 0, 10)   // len 0, cap 10

// SEARCH
import "slices"
slices.Contains(s, 2)     // true/false   (Go 1.21+)
slices.Index(s, 2)        // index or -1
// sorted -> binary:
import "sort"
sort.Ints(s)
i := sort.SearchInts(s, 2)

// UPDATE
s[0] = 100                // by index
s = append(s, 7)          // add to end

// DELETE
// keep order:
s = append(s[:i], s[i+1:]...)
// fast, order not kept:
s[i] = s[len(s)-1]; s = s[:len(s)-1]
// Go 1.21+: slices.Delete(s, i, i+1)
```

---

## 3. Map — `map[K]V` (hashmap / dictionary)

```go
// CREATE
m := make(map[string]int)
scores := map[string]int{"alice": 90, "bob": 75}

// SEARCH (comma-ok idiom)
v, ok := scores["alice"]  // ok == false if missing
if ok { fmt.Println(v) }

// UPDATE (same syntax as add)
scores["alice"] = 95
scores["carol"]++         // missing key starts at zero value

// DELETE
delete(scores, "bob")
```

Iteration order is **random**; sort keys if you need order.

---

## 4. Struct — grouped named fields (object / record)

```go
// CREATE
type User struct {
    Name string
    Age  int
}
u := User{Name: "Alice", Age: 30}
p := &User{Name: "Bob"}   // pointer to struct

// SEARCH (in a slice of structs)
import "slices"
users := []User{{"Alice", 30}, {"Bob", 25}}
i := slices.IndexFunc(users, func(x User) bool { return x.Name == "Bob" })

// UPDATE
u.Age = 31                // by field
p.Name = "Bobby"          // through pointer (auto-deref)

// DELETE a field? Not possible — fields are fixed.
// To "delete" a struct from a slice, delete the slice element:
users = append(users[:i], users[i+1:]...)
```

---

## 5. String — immutable UTF-8 bytes

Strings can't be modified in place — "update/delete" means building a new string.

```go
// CREATE
s := "héllo"
b := []byte(s)            // editable byte slice
r := []rune(s)            // editable rune slice (Unicode-safe)

// SEARCH
import "strings"
strings.Contains(s, "ll") // true
strings.Index(s, "l")     // first byte index, -1 if absent
strings.Count(s, "l")     // 2

// UPDATE (produce a new string)
s2 := strings.Replace(s, "l", "L", -1)   // or strings.ReplaceAll
s3 := s + " world"                       // concatenate
r[0] = 'H'; s4 := string(r)              // edit via runes

// DELETE (remove parts -> new string)
s5 := strings.ReplaceAll(s, "l", "")     // remove all "l"
s6 := s[:2] + s[3:]                       // drop byte at index 2
```

---

## 6. Pointer — `*T` (memory address)

```go
// CREATE
x := 10
p := &x                   // pointer to x
q := new(int)             // pointer to fresh zero int
var r *int                // nil pointer

// SEARCH / read (dereference)
fmt.Println(*p)           // 10

// UPDATE (change the pointed-to value)
*p = 20                   // x is now 20

// DELETE (drop the reference)
p = nil                   // no longer points anywhere
```

No pointer arithmetic in Go — it's memory-safe.

---

## 7. Channel — `chan T` (queue between goroutines)

Channels are consumed in order (FIFO), not searched or randomly updated.

```go
// CREATE
ch := make(chan int)      // unbuffered
buf := make(chan int, 3)  // buffered, cap 3

// ADD (send) / READ (receive)
ch <- 42                  // send  (in a goroutine)
v := <-ch                 // receive
v, ok := <-ch             // ok == false if closed & empty

// LOOP until closed
for v := range ch { fmt.Println(v) }

// "DELETE" / finish — close the channel
close(ch)                 // no more sends allowed
```

There is no update or random delete on a channel — you drain it in order.

---

## 8. Interface — `interface{...}` / `any`

Holds any value whose type implements its methods. The empty interface `any` holds anything.

```go
// CREATE
type Shape interface { Area() float64 }
type Circle struct{ R float64 }
func (c Circle) Area() float64 { return 3.14 * c.R * c.R }
var s Shape = Circle{R: 2}

// SEARCH (find the real type)
if c, ok := s.(Circle); ok { fmt.Println(c.R) }  // type assertion
switch v := s.(type) {                            // type switch
case Circle: fmt.Println("circle", v.R)
default:     fmt.Println("other")
}

// UPDATE (assign a different value/type)
s = Circle{R: 5}
var a any = "hi"; a = 42  // any can hold a new type

// DELETE (clear it)
s = nil
```

---

## 9. Function type — functions are values

You can store, pass, update, and drop functions like any other value.

```go
// CREATE
add := func(a, b int) int { return a + b }
var op func(int, int) int          // nil function value

// SEARCH — use in a map/registry to look one up
ops := map[string]func(int, int) int{
    "add": func(a, b int) int { return a + b },
    "sub": func(a, b int) int { return a - b },
}
f, ok := ops["add"]                // look up by name
if ok { fmt.Println(f(2, 3)) }     // 5

// UPDATE (reassign)
add = func(a, b int) int { return a + b + 1 }
ops["add"] = func(a, b int) int { return a * b }

// DELETE (from a registry map)
delete(ops, "sub")
op = nil                           // clear a function variable
```

---

# Derived / commonly built (composed from the above)

## 10. Set — `map[T]struct{}` or `map[T]bool`

Go has no native set; use a map with empty-struct values (zero memory per key).

```go
// CREATE
set := make(map[string]struct{})

// ADD
set["apple"] = struct{}{}

// SEARCH (membership)
_, ok := set["apple"]              // ok == true if present
if ok { fmt.Println("in set") }

// UPDATE — a set only stores presence; "update" = add or remove
// (bool version lets you toggle: set["apple"] = false)

// DELETE
delete(set, "apple")
```

---

## 11. Stack — LIFO from a slice

```go
// CREATE
var stack []int

// PUSH (create/add)
stack = append(stack, 1)
stack = append(stack, 2)

// SEARCH / peek top
top := stack[len(stack)-1]

// UPDATE top
stack[len(stack)-1] = 99

// POP (delete top)
stack = stack[:len(stack)-1]
```

---

## 12. Queue / Deque — FIFO from a slice

```go
// CREATE
var q []int

// ENQUEUE (add to back)
q = append(q, 1)
q = append(q, 2)

// SEARCH / peek front
front := q[0]

// UPDATE front
q[0] = 99

// DEQUEUE (delete from front)
q = q[1:]

// DEQUE extras: push/pop both ends
q = append([]int{0}, q...)  // push front
q = q[:len(q)-1]            // pop back
```

---

## 13. Linked list — `container/list` (doubly linked)

```go
import "container/list"

// CREATE
l := list.New()

// ADD
l.PushBack(1)              // to end
l.PushFront(0)             // to front

// SEARCH
var found *list.Element
for e := l.Front(); e != nil; e = e.Next() {
    if e.Value == 1 { found = e; break }
}

// UPDATE
if found != nil { found.Value = 42 }

// DELETE
l.Remove(found)
```

---

## 14. Heap / Priority queue — `container/heap`

Implement `heap.Interface`, then the package keeps heap order.

```go
import "container/heap"

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] } // min-heap
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() any {
    old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

// CREATE
h := &IntHeap{5, 2, 8}
heap.Init(h)

// ADD
heap.Push(h, 1)

// SEARCH — peek min (root) without removing
min := (*h)[0]

// UPDATE an element then re-heapify
(*h)[0] = 100
heap.Fix(h, 0)

// DELETE — pop the min, or remove at index i
smallest := heap.Pop(h).(int)
// heap.Remove(h, i)
```

---

## 15. Ring buffer — `container/ring` (circular, fixed size)

```go
import "container/ring"

// CREATE
r := ring.New(3)          // 3 slots

// ADD / UPDATE (set values as you move around)
for i := 0; i < r.Len(); i++ {
    r.Value = i
    r = r.Next()
}

// SEARCH / LOOP
r.Do(func(v any) { fmt.Println(v) })

// DELETE (unlink n elements after r)
r.Unlink(1)
```

Ring size is fixed; you overwrite values rather than grow it.

---

## 16. Tree / Graph — structs + pointers/slices

### Binary Search Tree
```go
type Node struct {
    Val         int
    Left, Right *Node
}

// CREATE + ADD (insert)
func insert(n *Node, v int) *Node {
    if n == nil { return &Node{Val: v} }
    if v < n.Val { n.Left = insert(n.Left, v) } else { n.Right = insert(n.Right, v) }
    return n
}

// SEARCH
func search(n *Node, v int) *Node {
    if n == nil || n.Val == v { return n }
    if v < n.Val { return search(n.Left, v) }
    return search(n.Right, v)
}

// UPDATE — find the node, change its Val (or delete + re-insert to keep order)
if node := search(root, 5); node != nil { node.Val = 6 }
```

### Graph (adjacency list) + BFS search
```go
graph := map[int][]int{1: {2, 3}, 2: {4}, 3: {4}, 4: {}}

// ADD edge
graph[1] = append(graph[1], 5)

// SEARCH (BFS)
func reachable(g map[int][]int, start, target int) bool {
    seen := map[int]struct{}{}
    q := []int{start}
    for len(q) > 0 {
        n := q[0]; q = q[1:]
        if n == target { return true }
        if _, ok := seen[n]; ok { continue }
        seen[n] = struct{}{}
        q = append(q, g[n]...)
    }
    return false
}

// DELETE a node
delete(graph, 4)
```

---

## 17. Concurrent map — `sync.Map`

Safe for many goroutines without extra locking.

```go
import "sync"

// CREATE
var m sync.Map

// ADD / UPDATE
m.Store("key", 1)

// SEARCH
v, ok := m.Load("key")           // ok == false if missing
actual, loaded := m.LoadOrStore("key", 2) // read or insert atomically

// LOOP
m.Range(func(k, v any) bool { fmt.Println(k, v); return true })

// DELETE
m.Delete("key")
```

---

# CRUD cheat sheet (one line each)

| # | Structure | Create | Search | Update | Delete |
|---|-----------|--------|--------|--------|--------|
| 1 | Array | `[3]int{...}` | linear loop | `a[i]=x` | `a[i]=0` (zero it) |
| 2 | Slice | `append(s, x)` | `slices.Index` | `s[i]=x` | `slices.Delete` |
| 3 | Map | `make(map...)` | `v,ok:=m[k]` | `m[k]=x` | `delete(m,k)` |
| 4 | Struct | `T{...}` | `slices.IndexFunc` | `s.Field=x` | remove slice elem |
| 5 | String | `"..."` | `strings.Index` | `Replace`/rebuild | `ReplaceAll("")` |
| 6 | Pointer | `&x` / `new(T)` | `*p` | `*p=x` | `p=nil` |
| 7 | Channel | `make(chan T)` | `<-ch` (FIFO) | — | `close(ch)` |
| 8 | Interface | `var s I = v` | type assert/switch | reassign | `s=nil` |
| 9 | Func type | `func(){...}` | map registry | reassign | `f=nil`/`delete` |
| 10 | Set | `map[T]struct{}` | `_,ok:=set[k]` | add/remove | `delete(set,k)` |
| 11 | Stack | `append` | `s[len-1]` | `s[len-1]=x` | `s=s[:len-1]` |
| 12 | Queue | `append` | `q[0]` | `q[0]=x` | `q=q[1:]` |
| 13 | Linked list | `list.New()` | loop `Next()` | `e.Value=x` | `l.Remove(e)` |
| 14 | Heap | `heap.Init` | `h[0]` (root) | `heap.Fix` | `heap.Pop` |
| 15 | Ring | `ring.New(n)` | `r.Do` | `r.Value=x` | `r.Unlink(n)` |
| 16 | Tree/Graph | structs+ptrs | recurse/BFS | node.Val=x | `delete`/unlink |
| 17 | sync.Map | `var m sync.Map` | `m.Load` | `m.Store` | `m.Delete` |
