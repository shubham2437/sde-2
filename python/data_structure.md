# Python Data Structures — Create · Search · Update · Delete

A complete reference to Python's data structures. Each shows the four core operations:
**Create**, **Search**, **Update**, **Delete** (plus loop where useful).

> Run any snippet with `python3 file.py` or paste into a REPL (`python3`).
> Built-ins need nothing; others show their `import`.

---

# Built-in (core language types)

## 1. List — `[]` (dynamic array, used most)

Ordered, mutable, allows duplicates.

```python
# CREATE
a = []
b = [1, 2, 3]
c = list(range(5))          # [0,1,2,3,4]
d = [0] * 3                 # [0,0,0]

# SEARCH
2 in b                      # True/False (membership)
b.index(2)                  # position -> 1 (ValueError if absent)
b.count(2)                  # occurrences
next((i for i,v in enumerate(b) if v == 2), -1)  # index or -1

# UPDATE
b[0] = 99                   # by index
b.append(4)                 # add to end
b.insert(1, 50)             # insert at index
b.extend([5, 6])            # add many
b[1:3] = [10, 20]           # slice assignment

# DELETE
b.remove(99)                # by value (first match)
del b[0]                    # by index
x = b.pop()                 # remove & return last
x = b.pop(0)                # remove & return at index
b.clear()                   # empty it
```

---

## 2. Tuple — `()` (immutable list)

Ordered, **cannot be changed** after creation. Good for fixed records / dict keys.

```python
# CREATE
t = (1, 2, 3)
single = (1,)               # note the comma!
empty = ()

# SEARCH
2 in t                      # True
t.index(2)                  # 1
t.count(2)                  # 1

# UPDATE — not allowed. Build a new tuple instead:
t2 = t[:1] + (99,) + t[2:]  # replace index 1

# DELETE — not allowed. Build a new tuple without the element:
t3 = t[:1] + t[2:]          # drop index 1
```

Tuples are hashable (if their contents are), so they can be dict keys / set members.

---

## 3. Dict — `{}` (hashmap / dictionary)

Key → value store, insertion-ordered (Python 3.7+), O(1) lookup.

```python
# CREATE
m = {}
scores = {"alice": 90, "bob": 75}
m2 = dict(x=1, y=2)

# SEARCH
"alice" in scores           # True (checks KEYS)
scores.get("carol")         # None if missing (no error)
scores.get("carol", 0)      # default 0 if missing
v = scores["alice"]         # KeyError if missing

# UPDATE (same syntax as add)
scores["alice"] = 95
scores["carol"] = 88        # add new key
scores.update({"dave": 60}) # merge in
scores.setdefault("eve", 0) # add only if absent

# DELETE
del scores["bob"]
v = scores.pop("alice")     # remove & return
scores.pop("ghost", None)   # safe: no error if absent
scores.clear()

# LOOP
for k, v in scores.items(): ...
```

---

## 4. Set — `set()` (unique items, hashmap-based)

Unordered, no duplicates, fast membership.

```python
# CREATE
s = set()                   # NOTE: {} is an empty DICT, not a set
s = {1, 2, 3}
s = set([1, 2, 2, 3])       # {1,2,3}

# SEARCH
2 in s                      # True — O(1)

# UPDATE (add)
s.add(4)
s.update([5, 6])            # add many

# DELETE
s.remove(2)                 # KeyError if absent
s.discard(99)               # safe, no error if absent
x = s.pop()                 # remove arbitrary element
s.clear()

# SET MATH
a, b = {1,2,3}, {2,3,4}
a | b                       # union {1,2,3,4}
a & b                       # intersection {2,3}
a - b                       # difference {1}
a ^ b                       # symmetric diff {1,4}
```

---

## 5. Frozenset — immutable set

Like a set but unchangeable → hashable, usable as a dict key / set member.

```python
# CREATE
fs = frozenset([1, 2, 3])

# SEARCH
2 in fs                     # True

# UPDATE / DELETE — not allowed. Build a new one:
fs2 = fs | {4}              # "add" -> new frozenset
fs3 = fs - {1}              # "remove" -> new frozenset
```

---

## 6. String — `str` (immutable text)

Ordered, immutable sequence of characters.

```python
# CREATE
s = "hello"
s2 = str(123)               # "123"
s3 = "a" * 3                # "aaa"

# SEARCH
"ell" in s                  # True
s.find("l")                 # first index, -1 if absent
s.index("l")                # first index, ValueError if absent
s.count("l")                # 2
s.startswith("he")          # True

# UPDATE — immutable, so build a new string:
s4 = s.replace("l", "L")    # "heLLo"
s5 = s + " world"
s6 = s[:1] + "E" + s[2:]    # change index 1

# DELETE (remove parts -> new string)
s7 = s.replace("l", "")     # "heo"
s8 = s[:2] + s[3:]          # drop index 2
```

---

## 7. Bytes / Bytearray — binary data

`bytes` is immutable; `bytearray` is mutable.

```python
# CREATE
b = b"hello"                # bytes (immutable)
ba = bytearray(b"hello")    # bytearray (mutable)

# SEARCH
b"ell" in b                 # True
b.find(b"l")                # index

# UPDATE (bytearray only)
ba[0] = 72                  # 'H'  (must be 0-255 int)
ba.append(33)               # add byte

# DELETE (bytearray only)
del ba[0]
ba.pop()
```

---

# Standard library containers

## 8. `collections.deque` — fast queue / stack / deque

O(1) appends and pops from BOTH ends (lists are O(n) at the front).

```python
from collections import deque

# CREATE
dq = deque([1, 2, 3])
dq = deque(maxlen=5)        # bounded ring buffer

# SEARCH
2 in dq                     # True
dq.index(2)                 # position

# UPDATE (add)
dq.append(4)                # right
dq.appendleft(0)            # left
dq.rotate(1)                # rotate right

# DELETE
dq.pop()                    # from right
dq.popleft()                # from left  (this is the fast FIFO dequeue)
dq.remove(2)                # by value
```

Use as a **stack** (append/pop) or **queue** (append/popleft).

---

## 9. `collections.OrderedDict`

A dict that remembers insertion order with extra order-aware methods. (Plain dicts are ordered since 3.7, but this adds `move_to_end` etc.)

```python
from collections import OrderedDict

# CREATE
od = OrderedDict([("a", 1), ("b", 2)])

# SEARCH / UPDATE / DELETE — same as dict
od["c"] = 3                 # add/update
"a" in od                   # search
del od["a"]                 # delete

# ORDER controls
od.move_to_end("b")         # send to end
od.popitem(last=False)      # pop from front (LRU-style)
```

---

## 10. `collections.defaultdict`

A dict that auto-creates a default value for missing keys.

```python
from collections import defaultdict

# CREATE
dd = defaultdict(int)       # missing key -> 0
dl = defaultdict(list)      # missing key -> []

# UPDATE (no KeyError on missing key)
dd["x"] += 1                # starts at 0 -> 1
dl["y"].append(5)           # starts at [] -> [5]

# SEARCH / DELETE — same as dict
"x" in dd
del dd["x"]
```

---

## 11. `collections.Counter` — counting multiset

A dict subclass for tallying occurrences.

```python
from collections import Counter

# CREATE
c = Counter("banana")       # {'a':3,'n':2,'b':1}
c = Counter([1, 1, 2, 3])

# SEARCH
c["a"]                      # 3  (0 for missing, no error)
c.most_common(2)            # [('a',3),('n',2)]

# UPDATE
c["a"] += 1
c.update("aa")              # add more counts

# DELETE
del c["b"]
c.subtract("a")             # decrement counts
```

---

## 12. `collections.namedtuple` — lightweight record

An immutable tuple with named fields (like a small struct/object).

```python
from collections import namedtuple

# CREATE
Point = namedtuple("Point", ["x", "y"])
p = Point(1, 2)

# SEARCH / read
p.x                         # 1  (by name)
p[0]                        # 1  (by index)

# UPDATE — immutable; make a modified copy:
p2 = p._replace(x=99)       # Point(x=99, y=2)

# DELETE — not applicable (fixed fields)
```

For a mutable version, use `@dataclass` (below).

---

## 13. `dataclass` — mutable object / record

The modern "struct/object" — auto-generates `__init__`, `__repr__`, etc.

```python
from dataclasses import dataclass, field

# CREATE
@dataclass
class User:
    name: str
    age: int = 0
    tags: list = field(default_factory=list)

u = User("Alice", 30)

# SEARCH (in a list of objects)
users = [User("Alice", 30), User("Bob", 25)]
found = next((x for x in users if x.name == "Bob"), None)

# UPDATE
u.age = 31                  # by attribute

# DELETE (remove object from a list)
users.remove(found)
```

---

## 14. `heapq` — heap / priority queue

Turns a plain list into a **min-heap** (smallest first).

```python
import heapq

# CREATE
h = [5, 2, 8]
heapq.heapify(h)            # in-place -> valid heap

# UPDATE (add)
heapq.heappush(h, 1)

# SEARCH — peek smallest (root)
smallest = h[0]

# DELETE — pop smallest
x = heapq.heappop(h)        # 1
# push+pop in one step:
heapq.heapreplace(h, 7)

# priority queue: push (priority, item) tuples
heapq.heappush(h, (2, "task"))
# n smallest/largest
heapq.nsmallest(3, h); heapq.nlargest(3, h)
```

For a max-heap, negate values: `heapq.heappush(h, -x)`.

---

## 15. `queue.Queue` — thread-safe queue

For passing items between threads (like Go channels).

```python
import queue

# CREATE
q = queue.Queue()           # FIFO
lq = queue.LifoQueue()      # stack
pq = queue.PriorityQueue()  # priority

# ADD
q.put(1)

# READ / consume (in order — not searched)
item = q.get()              # blocks until available
q.task_done()

# STATE
q.qsize(); q.empty()
```

No random search/update/delete — it's a synchronized pipe you drain in order.

---

## 16. `array.array` — compact typed array

Space-efficient array of one numeric C type (unlike a list of objects).

```python
from array import array

# CREATE
arr = array("i", [1, 2, 3])  # 'i' = signed int

# SEARCH
2 in arr
arr.index(2)

# UPDATE
arr[0] = 99
arr.append(4)

# DELETE
arr.remove(99)
del arr[0]
arr.pop()
```

---

# Built-your-own structures

## 17. Stack (using a list)

```python
# CREATE
stack = []

# PUSH / peek / POP
stack.append(1)             # push
top = stack[-1]             # peek
stack[-1] = 99              # update top
x = stack.pop()             # pop (delete top)
```

## 18. Queue (use deque, not list)

```python
from collections import deque
q = deque()
q.append(1)                 # enqueue
front = q[0]                # peek
x = q.popleft()             # dequeue (O(1))
```

## 19. Linked list (custom nodes)

```python
class Node:
    def __init__(self, val, nxt=None):
        self.val = val
        self.next = nxt

# CREATE
head = Node(1, Node(2, Node(3)))

# SEARCH
def find(head, target):
    cur = head
    while cur:
        if cur.val == target:
            return cur
        cur = cur.next
    return None

# UPDATE
node = find(head, 2)
if node: node.val = 99

# DELETE (unlink a value)
def delete(head, target):
    dummy = Node(0, head)
    cur = dummy
    while cur.next:
        if cur.next.val == target:
            cur.next = cur.next.next
            break
        cur = cur.next
    return dummy.next
head = delete(head, 99)
```

## 20. Tree / Graph

```python
# Binary Search Tree
class TreeNode:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

def insert(root, v):        # CREATE / add
    if root is None: return TreeNode(v)
    if v < root.val: root.left = insert(root.left, v)
    else: root.right = insert(root.right, v)
    return root

def search(root, v):        # SEARCH
    if root is None or root.val == v: return root
    return search(root.left, v) if v < root.val else search(root.right, v)

# Graph as adjacency dict
graph = {1: [2, 3], 2: [4], 3: [4], 4: []}
graph[1].append(5)          # add edge
del graph[4]                # delete node

def bfs(g, start, target):  # SEARCH (BFS)
    from collections import deque
    seen, q = set(), deque([start])
    while q:
        n = q.popleft()
        if n == target: return True
        if n in seen: continue
        seen.add(n)
        q.extend(g.get(n, []))
    return False
```

---

# CRUD cheat sheet (one line each)

| # | Structure | Create | Search | Update | Delete |
|---|-----------|--------|--------|--------|--------|
| 1 | list | `[1,2,3]` | `x in l` / `.index` | `l[i]=x` / `.append` | `.remove` / `del l[i]` / `.pop` |
| 2 | tuple | `(1,2,3)` | `x in t` | rebuild (immutable) | rebuild (immutable) |
| 3 | dict | `{"k":v}` | `k in d` / `.get` | `d[k]=v` | `del d[k]` / `.pop` |
| 4 | set | `{1,2,3}` | `x in s` | `.add` | `.remove` / `.discard` |
| 5 | frozenset | `frozenset(..)` | `x in fs` | rebuild | rebuild |
| 6 | str | `"..."` | `in` / `.find` | `.replace` (new) | `.replace("")` (new) |
| 7 | bytearray | `bytearray(b"..")` | `in` / `.find` | `ba[i]=n` | `del ba[i]` |
| 8 | deque | `deque([..])` | `x in dq` | `.append`/`.appendleft` | `.pop`/`.popleft` |
| 9 | OrderedDict | `OrderedDict()` | `k in od` | `od[k]=v` | `del od[k]` |
| 10 | defaultdict | `defaultdict(int)` | `k in dd` | `dd[k]+=1` | `del dd[k]` |
| 11 | Counter | `Counter(..)` | `c[k]` | `c[k]+=1` | `del c[k]` |
| 12 | namedtuple | `NT(...)` | `.field` | `._replace` (new) | n/a |
| 13 | dataclass | `@dataclass` | attr / list search | `obj.attr=x` | remove from list |
| 14 | heapq | `heapify(l)` | `l[0]` (min) | `heappush` | `heappop` |
| 15 | queue.Queue | `Queue()` | `.get` (FIFO) | `.put` | `.get` |
| 16 | array | `array("i",..)` | `in` / `.index` | `arr[i]=x` | `.remove`/`del` |
| 17 | Stack (list) | `[]` | `s[-1]` | `s[-1]=x` | `.pop()` |
| 18 | Queue (deque) | `deque()` | `q[0]` | `.append` | `.popleft()` |
| 19 | Linked list | custom `Node` | walk `.next` | `node.val=x` | unlink node |
| 20 | Tree/Graph | structs/dict | recurse / BFS | `node.val=x` | `del`/unlink |

---

## Python vs Go quick note

- Python's **list** = Go slice; Python **dict** = Go map; Python **set** = Go `map[T]struct{}`.
- Python **tuple/frozenset/str** are immutable (like Go strings) — "update/delete" rebuilds them.
- Python has **no built-in typed fixed array** like Go's `[N]T`; use `array.array` or just a list.
- `collections` and `heapq`/`queue` give you deque, Counter, heap, and thread-safe queues out of the box.
