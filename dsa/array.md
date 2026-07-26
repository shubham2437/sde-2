# Arrays, Strings & Hashing — Complete Theory Guide

> **Start date:** \_\_\_\_\_\_\_\_\_\_
> **Completion date:** \_\_\_\_\_\_\_\_\_\_

This guide explains the *theory* behind the core array patterns. For each topic you get:

1. **What it is** — the core idea
2. **How it works** — with a diagram / dry run
3. **Pattern recognition** — *"how do I know a new question is this topic?"* (the trigger words & signals)
4. **Time & space complexity**

---

## 0. Foundation — how to think about arrays

An **array** is a block of contiguous memory. Element `i` lives at `base + i * size`, so **random access `arr[i]` is O(1)**. That single fact drives almost every trick below.

```
index:   0    1    2    3    4
        +----+----+----+----+----+
arr =   | 5  | 2  | 9  | 1  | 7  |
        +----+----+----+----+----+
memory: 100  104  108  112  116   (each int = 4 bytes)
```

Key operations and their cost:

| Operation                  | Cost   | Why                                  |
|----------------------------|--------|--------------------------------------|
| Access `arr[i]`            | O(1)   | Direct address math                  |
| Update `arr[i] = x`        | O(1)   | Same                                 |
| Search (unsorted)          | O(n)   | Must scan                            |
| Search (sorted)            | O(log n) | Binary search                      |
| Insert / delete middle     | O(n)   | Must shift elements                  |
| Insert / delete at end     | O(1)*  | Amortized for dynamic arrays         |

**The big four techniques** that solve most array problems:

- **Two pointers** — one array, indices moving toward/with each other.
- **Sliding window** — a moving sub-range, grow/shrink to keep a condition.
- **Prefix computation** — precompute cumulative info so each query is O(1).
- **Hashing** — trade memory for O(1) lookup ("have I seen this before?").

Keep these four in your head. Most "how do I recognize the topic" answers below reduce to *"which of the big four fits?"*

---

## 1. Maximum / Minimum Subarray

### What it is
Find the contiguous subarray with the largest (or smallest) **sum**. "Contiguous" = elements next to each other, no skipping.

### How it works
Brute force checks every subarray `(i, j)`:

```
arr = [-2, 1, -3, 4, -1, 2, 1, -5, 4]

Try all start/end pairs:
  [-2]            = -2
  [-2, 1]         = -1
  [-2, 1, -3]     = -4
  ...
  [4, -1, 2, 1]   =  6   <-- best
```

That is O(n²) or O(n³). The smart solution is **Kadane's algorithm** (next section), which is O(n).

### Pattern recognition — is a new question this topic?
Look for these signals:
- The word **"contiguous subarray"** + **"maximum/minimum sum"**.
- "Largest sum you can get from a consecutive stretch."
- Variants: *maximum product* subarray, *circular* maximum subarray, *max sum with at most k elements*.

If it says **"subsequence"** (skipping allowed) instead of subarray, it is a *different* family (usually DP), not this one.

### Complexity
- Brute force: **O(n²)** time, O(1) space.
- Kadane: **O(n)** time, **O(1)** space.

---

## 2. Kadane's Algorithm

### What it is
The O(n) way to find the maximum subarray sum. The core insight:

> At each index, the best subarray ending *here* is either **just this element**, or **this element plus the best subarray ending at the previous index.**

Formally: `curr = max(arr[i], curr + arr[i])`, and track the global best.

### How it works — dry run

```
arr = [-2, 1, -3, 4, -1, 2, 1, -5, 4]

i   arr[i]  curr = max(arr[i], curr+arr[i])   best
--------------------------------------------------
0    -2     max(-2, -2)      = -2             -2
1     1     max( 1, -2+1=-1) =  1              1
2    -3     max(-3,  1-3=-2) = -2              1
3     4     max( 4, -2+4=2)  =  4              4
4    -1     max(-1, 4-1=3)   =  3              4
5     2     max( 2, 3+2=5)   =  5              5
6     1     max( 1, 5+1=6)   =  6              6   <-- answer
7    -5     max(-5, 6-5=1)   =  1              6
8     4     max( 4, 1+4=5)   =  5              6
```

Answer = **6**, the subarray `[4, -1, 2, 1]`.

**Why "drop the past"?** If `curr` ever goes negative, carrying it forward only *hurts* the next element — so we reset to start fresh at `arr[i]`. That reset is the whole trick.

```
When running sum < 0:
   ...previous stuff...  |  fresh start here
   (dead weight, drop it)   ^ curr = arr[i]
```

### Pseudocode
```
curr = best = arr[0]
for i in 1..n-1:
    curr = max(arr[i], curr + arr[i])
    best = max(best, curr)
return best
```

```curr = best = arr[0]
start = 0          // start of the current running subarray
bestStart = 0      // start of the best subarray so far
bestEnd = 0        // end of the best subarray so far

for i in 1..n-1:
    if curr + arr[i] < arr[i]:
        curr = arr[i]       // drop the past, start fresh
        start = i           // <-- new run begins here
    else:
        curr = curr + arr[i]  // extend the current run

    if curr > best:
        best = curr
        bestStart = start   // lock in the winning window
        bestEnd = i

return arr[bestStart .. bestEnd]```

To also return the **indices**, remember where `curr` reset (start) and where `best` updated (end).

### Pattern recognition
- Any "maximum/minimum sum of a contiguous run" problem.
- Key mental cue: *"the best answer ending here depends only on the best answer ending just before."* That "depends only on previous" is the Kadane signature (it is 1-D dynamic programming).
- **Maximum product subarray** is a Kadane cousin — but you track *both* max and min (because a negative × negative flips to a large positive).

### Complexity
- **O(n)** time, **O(1)** space. One pass, two variables.

---

## 3. Prefix and Suffix Sums

### What it is
Precompute cumulative sums so that the sum of *any* range can be answered in **O(1)** instead of O(n).

`prefix[i]` = sum of `arr[0..i-1]` (everything before index `i`).

### How it works

```
arr    =    [ 3,  1,  4,  1,  5,  9 ]
index         0   1   2   3   4   5

prefix = [0, 3, 4, 8, 9, 14, 23]
          ^  ^
     prefix[0]=0 (empty), prefix[k] = arr[0]+...+arr[k-1]
```

Sum of range `[L, R]` (inclusive) = `prefix[R+1] - prefix[L]`.

```
Sum of arr[2..4] = prefix[5] - prefix[2]
                 = 14 - 4 = 10
Check: 4 + 1 + 5 = 10  ✓
```

Diagram of the subtraction idea:

```
prefix[R+1]:  |=========================|   (sum 0..R)
prefix[L]:    |=========|                    (sum 0..L-1)
              subtract → |==============|     (sum L..R)  ✓
```

A **suffix sum** is the mirror: `suffix[i]` = sum of `arr[i..n-1]`. Useful when a problem needs "everything to the right."

### Pattern recognition
- **"Many range-sum queries"** on a fixed array → build prefix once, answer each query O(1).
- Anything asking about "sum from index i to j", "average of a window", "how much is to the left/right of each element."
- 2-D version (prefix sum matrix) for **submatrix sum** queries.
- If you catch yourself re-summing overlapping ranges, prefix sums remove the repeat work.

### Complexity
- Build: **O(n)** time, **O(n)** space.
- Each query afterward: **O(1)**.

---

## 4. Subarray Sum Equals K

### What it is
Count (or find) contiguous subarrays whose sum equals a target `k`. This is where **prefix sum meets hashing** — one of the most important patterns to master.

### How it works — the key equation
If `prefix[j] - prefix[i] = k`, then the subarray `arr[i..j-1]` sums to `k`. Rearrange:

```
prefix[i] = prefix[j] - k
```

So while scanning, at each `j` we ask: *"how many earlier prefix values equal `currentPrefix - k`?"* Store prefix-sum frequencies in a **hash map** and look up in O(1).

### Dry run
```
arr = [1, 2, 3], k = 3

map = {0: 1}   (empty prefix seen once — crucial for subarrays starting at index 0)
running = 0, count = 0

j=0: running=1;  need 1-3=-2 → not in map. map={0:1, 1:1}
j=1: running=3;  need 3-3= 0 → in map (count+1=1). map={0:1,1:1,3:1}
j=2: running=6;  need 6-3= 3 → in map (count+1=2). map={...,6:1}

Answer = 2   → subarrays [1,2] and [3]
```

Visual of "need = running − k":

```
      <---- prefix[i] ---->
[==================|===== k =====]
                    running (prefix[j])
prefix[i] = running - k   ← look this up in the map
```

### Pattern recognition — very common triggers
- "Count / find subarrays with sum = k" (or product = k, or sum divisible by k).
- "Longest subarray with sum k."
- **Binary array**: "subarray with equal 0s and 1s" → map 0 to −1, then it becomes sum = 0.
- "Subarray sum divisible by k" → store `prefix % k` in the map.
- General cue: *contiguous* + *a target aggregate* + you need to avoid O(n²) → **prefix sum + hashmap**.
- ⚠️ If all numbers are **positive**, a **sliding window** also works (and uses O(1) space). If negatives are possible, you *must* use the prefix-sum + hashmap approach — the window can't shrink reliably with negatives.

### Complexity
- **O(n)** time (single pass), **O(n)** space (the hash map).

---

## 5. Product of Array Except Self

### What it is
For each index `i`, compute the product of *all other* elements — **without using division** and in O(n).

### How it works
`answer[i] = (product of everything to the left of i) × (product of everything to the right of i)`.

This is prefix/suffix thinking applied to products.

```
arr    = [ 1,  2,  3,  4 ]

left  (prefix products, exclusive):
  left[0]=1
  left[1]=1
  left[2]=1*2=2
  left[3]=1*2*3=6
        = [1, 1, 2, 6]

right (suffix products, exclusive):
  right[3]=1
  right[2]=4
  right[1]=4*3=12
  right[0]=4*3*2=24
        = [24, 12, 4, 1]

answer[i] = left[i] * right[i]
        = [24, 12, 8, 6]
```

Diagram for index 2:

```
arr:   [ 1   2 | (3) | 4 ]
         left=1*2=2    right=4
answer[2] = 2 * 4 = 8   ✓
```

**Space trick:** put the left products in the output array, then multiply by a running right-product in a second pass → **O(1) extra space** (excluding the output).

### Pattern recognition
- "Result for each index depends on everything *except* that index."
- "Do it without division" (division fails when a zero is present, and is often banned to test the prefix/suffix idea).
- Cue: *each output cell = combine(left side, right side)* → prefix + suffix pass.

### Complexity
- **O(n)** time, **O(1)** extra space (output array not counted).

---

## 6. Merge Overlapping Intervals

### What it is
Given intervals like `[1,3], [2,6], [8,10]`, merge every pair that overlaps into one.

### How it works
**Step 1 — sort by start time.** This is the whole trick: once sorted, overlapping intervals are adjacent, so a single left-to-right pass merges them.

**Step 2 — sweep.** Keep the last interval in the result. For each next interval:
- If it **overlaps** (its start ≤ current end) → extend the end.
- Else → it's a new separate interval, push it.

```
Input:  [1,3] [2,6] [8,10] [15,18]

Sort by start (already sorted here).

merged = [ [1,3] ]

[2,6]:  2 <= 3  → overlap → extend end to max(3,6)=6
        merged = [ [1,6] ]

[8,10]: 8 > 6   → no overlap → push
        merged = [ [1,6], [8,10] ]

[15,18]:15 > 10 → no overlap → push
        merged = [ [1,6], [8,10], [15,18] ]
```

Overlap condition on a number line:

```
   [1------3]
      [2--------6]      start(2) <= end(3) → overlap → merge to [1,6]
                 ...
                 [8---10]   start(8) > end(6) → gap → separate
```

### Pattern recognition
- Input is a **list of intervals / ranges** `[start, end]`.
- Words: "merge", "overlapping", "meeting rooms", "free time", "insert interval", "non-overlapping."
- Almost every interval problem starts with **sort by start (or end)** — that sort is the recognition signal itself.

### Complexity
- **O(n log n)** time (dominated by the sort), **O(n)** space for the output (O(1) extra if merging in place).

---

## 7. Rotate Array

### What it is
Shift every element `k` positions (usually to the right), wrapping around.

```
arr = [1,2,3,4,5,6,7], k = 3
result = [5,6,7,1,2,3,4]
```

### How it works — the reversal trick (elegant, O(1) space)
Rotating right by `k` = **reverse whole array, then reverse the first k, then reverse the rest.**

```
Start:            [1, 2, 3, 4, 5, 6, 7]   k=3

1) reverse all:   [7, 6, 5, 4, 3, 2, 1]
2) reverse first k=3:
                  [5, 6, 7, 4, 3, 2, 1]
3) reverse rest:  [5, 6, 7, 1, 2, 3, 4]   ✓
```

Why it works: the last `k` elements need to move to the front *in original order*. The full reverse puts them in front but backwards; the two partial reverses fix the order of each block.

**Always do `k = k % n` first** — rotating by `n` is a no-op, and `k` can exceed `n`.

Other methods: extra array (`newIndex = (i + k) % n`, O(n) space) or juggling by cycles.

### Pattern recognition
- "Rotate / shift / cyclically move by k."
- "Rotate a matrix 90°" is a cousin (transpose + reverse rows — see next topic).
- Cue: *elements wrap around* and you want it in place → think **reverse trick** or **index modulo n**.

### Complexity
- Reversal method: **O(n)** time, **O(1)** space.

---

## 8. Matrix Traversal and Rotation

### What it is
Two common 2-D array skills: (a) traverse a matrix in a specific order (e.g. spiral), and (b) rotate a matrix 90° in place.

### Rotate 90° clockwise — transpose + reverse rows
```
Original          Transpose         Reverse each row
1 2 3             1 4 7             7 4 1
4 5 6      →      2 5 8      →      8 5 2
7 8 9             3 6 9             9 6 3
```
- **Transpose**: swap `matrix[i][j]` with `matrix[j][i]` (flip across the main diagonal).
- **Reverse each row**: gives the clockwise rotation.
- (Counter-clockwise = transpose + reverse each *column*.)

### Spiral traversal — shrinking boundaries
Keep four boundaries `top, bottom, left, right` and peel layers:

```
→ → → →
        ↓
↑       ↓
↑ ← ← ←

1  2  3  4
5  6  7  8      spiral = 1 2 3 4 8 12 11 10 9 5 6 7
9 10 11 12
```
Go right along `top`, down `right`, left along `bottom`, up `left`, then move the boundaries inward and repeat.

### Pattern recognition
- Input is a **2-D grid / matrix**.
- Words: "rotate the image/matrix", "spiral order", "in place", "layer by layer", "diagonal traversal."
- Cue for rotation: think **transpose + reverse**. Cue for traversal: **four moving boundaries** or direction vectors.

### Complexity
- Rotation and full traversal: **O(m×n)** time (visit each cell once). Rotation is **O(1)** extra space when done in place.

---

## 9. Dutch National Flag (Sort 0s, 1s, 2s)

### What it is
Sort an array containing only **three distinct values** (classically 0, 1, 2) in a **single pass** with O(1) space. Named after the three-color Dutch flag.

### How it works — three pointers
Maintain three regions using pointers `low`, `mid`, `high`:

```
[ 0s | 1s | unknown | 2s ]
      ^    ^        ^
     low  mid      high

Region rules:
  arr[0 .. low-1]   = all 0s
  arr[low .. mid-1] = all 1s
  arr[mid .. high]  = unknown (still to process)
  arr[high+1 .. n-1]= all 2s
```

Loop while `mid <= high`:
- `arr[mid] == 0`: swap `arr[low], arr[mid]`; `low++`, `mid++`
- `arr[mid] == 1`: it's already in place; `mid++`
- `arr[mid] == 2`: swap `arr[mid], arr[high]`; `high--` (do **not** move mid — the swapped-in value is unknown)

### Dry run
```
arr = [2, 0, 2, 1, 1, 0]      low=0, mid=0, high=5

mid=0 val=2 → swap(0,5), high=4   [0,0,2,1,1,2]
mid=0 val=0 → swap(0,0), low=1,mid=1 [0,0,2,1,1,2]
mid=1 val=0 → swap(1,1), low=2,mid=2 [0,0,2,1,1,2]
mid=2 val=2 → swap(2,4), high=3   [0,0,1,1,2,2]
mid=2 val=1 → mid=3               [0,0,1,1,2,2]
mid=3 val=1 → mid=4               [0,0,1,1,2,2]
mid(4) > high(3) → stop

Result: [0,0,1,1,2,2]  ✓
```

### Pattern recognition
- Array has only **2 or 3 distinct categories** and you must group/sort them in one pass.
- Words: "sort 0s 1s 2s", "sort colors", "partition into ≤ pivot / = pivot / > pivot" (this is the heart of **quicksort's 3-way partition**).
- Cue: *"one pass, constant space, few categories"* → three-pointer partition.

### Complexity
- **O(n)** time (single pass), **O(1)** space.

---

## 10. Majority Element

### What it is
Find the element appearing **more than ⌊n/2⌋ times** (guaranteed to exist in the classic version).

### How it works — Boyer–Moore Voting
Intuition: the majority element occurs more than all others *combined*. So if we let each majority vote **+1** and every other vote **−1**, the total can never be cancelled out.

Keep a `candidate` and a `count`:
- If `count == 0`, adopt the current element as `candidate`.
- If `element == candidate`, `count++`; else `count--`.

```
arr = [2, 2, 1, 1, 1, 2, 2]

el  candidate count
------------------------
2      2       1
2      2       2
1      2       1
1      2       0   (tie — cancelled)
1      1       1   (count was 0 → adopt 1)
2      1       0
2      2       1   (adopt 2)
------------------------
candidate = 2   → majority element
```

Think of it as pairing off opposites: every non-majority element cancels one majority vote, but there aren't enough of them, so a majority vote always survives.

```
  2 2 2 2   (majority, 4 votes)
  1 1 2      (others, 3 votes)
  cancel 3 pairs → one 2 left standing → answer = 2
```

> If existence isn't guaranteed, do a **second pass to verify** the candidate actually exceeds n/2.

### Pattern recognition
- "Element appearing more than half / more than n/2 / n/3 times."
- "Dominant element", "majority vote."
- Simpler-but-heavier alternatives: **hash map counting** (O(n) time, O(n) space) or **sort and take the middle** (O(n log n)). Boyer–Moore is the O(1)-space winner.
- The **n/3 variant** tracks *two* candidates (at most two elements can exceed n/3).

### Complexity
- Boyer–Moore: **O(n)** time, **O(1)** space.

---

## Master Cheat-Sheet — recognizing the pattern from the question

| If the question says…                                            | Reach for…                        | Time      | Space |
|------------------------------------------------------------------|-----------------------------------|-----------|-------|
| "max/min sum of a **contiguous** subarray"                       | Kadane                            | O(n)      | O(1)  |
| "answer many **range-sum** queries"                              | Prefix sums                       | O(n)+O(1)/q | O(n) |
| "count/find **subarrays with sum = k**" (with negatives)         | Prefix sum + hashmap              | O(n)      | O(n)  |
| "subarray sum = k, all **positive** numbers"                     | Sliding window                    | O(n)      | O(1)  |
| "product of all **except self**", no division                   | Prefix × suffix products          | O(n)      | O(1)  |
| list of **[start,end] intervals**, "merge/overlap"               | Sort by start + sweep             | O(n log n)| O(n)  |
| "**rotate/shift** by k, in place"                                | Reverse trick / index mod n       | O(n)      | O(1)  |
| "**rotate matrix** 90°" / "spiral order"                         | Transpose+reverse / 4 boundaries  | O(m·n)    | O(1)  |
| only **0/1/2** (or 3 categories), one pass                       | Dutch flag (3 pointers)           | O(n)      | O(1)  |
| "element appearing **> n/2** times"                              | Boyer–Moore voting                | O(n)      | O(1)  |

### How to reason about time complexity (quick rules)
- **Single loop** over n → O(n).
- **Nested loops** (all pairs) → O(n²); triple nested → O(n³).
- **Sorting first** → at least O(n log n) — the sort dominates simple passes.
- **Hash map lookup / insert** → O(1) average, so a single pass with a map stays O(n).
- **Binary search / halving** → O(log n); a loop with binary search inside → O(n log n).
- **Space**: O(1) if you use a fixed number of variables/pointers; O(n) if you build a map/array proportional to input.

### A 4-question triage for any array problem
1. **Contiguous** subarray/window? → sliding window or Kadane or prefix sums.
2. Need **"seen before?" / counts / pairs**? → hash map/set.
3. **Sorted or can I sort**? → two pointers, binary search, or interval sweep.
4. **In-place, O(1) space** demanded? → pointer partitioning (Dutch flag), reversal trick, Boyer–Moore.

---

*Study tip: for each topic, first re-derive the diagram from memory, then code it from scratch, then solve 2–3 practice problems. The pattern-recognition table is what turns a new, unseen question into a known one.*
