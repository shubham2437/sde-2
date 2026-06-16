# Database Indexing — Complete Study Guide

A deep dive into everything you need to know about indexing for interviews: what indexes are, how they work internally, their tradeoffs, the different types, and how to read query plans.

---

## 1. What Is an Index?

An **index** is a separate data structure that the database maintains *in addition to* the table, whose only job is to make **finding rows faster**.

### The library analogy
Imagine a book with no index. To find every mention of "indexing," you'd read every page — that's a **full table scan**. An index at the back of the book lets you jump straight to the right pages. A database index works the same way: it stores the indexed column's values in a sorted, searchable structure along with a pointer to the actual row.

### What problem it solves
Without an index, finding `WHERE email = 'a@b.com'` forces the DB to check **every single row** — O(n). With an index on `email`, the DB navigates a sorted tree structure — roughly **O(log n)**. On a million-row table that's the difference between ~1,000,000 checks and ~20.

### The core tradeoff
Indexes are not free. Every index you add has costs:

| Benefit | Cost |
|---------|------|
| **Faster reads** (SELECT, WHERE, JOIN, ORDER BY) | **Slower writes** — every INSERT/UPDATE/DELETE must also update the index |
| Faster lookups, range scans, sorting | **Extra storage** — the index is a copy of the indexed columns + pointers |
| Faster JOINs on indexed columns | **Maintenance overhead** — indexes can fragment and need rebuilding |

> **Key interview line:** "An index trades write speed and storage for read speed. You index columns you frequently search/filter/join on, not every column."

This is why you **don't index everything**. A table with 15 indexes will have painfully slow writes because each write fans out into 16 updates (the table + 15 indexes).

---

## 2. Clustered vs Non-Clustered Indexes

This is one of the most common indexing interview topics. The distinction is about **where the actual row data physically lives** relative to the index.

### Clustered Index
- **Determines the physical order of rows** in the table on disk.
- The index *is* the table — the leaf nodes of the index **are** the actual data rows.
- A table can have **only ONE clustered index** (data can only be physically sorted one way).
- In most databases (SQL Server, MySQL/InnoDB), the **primary key automatically becomes the clustered index**.

Think of a phone book sorted by last name — the data itself is stored in that order.

### Non-Clustered Index
- A **separate structure** from the table data.
- Its leaf nodes contain the indexed value plus a **pointer** (the row's location, or in InnoDB, the clustered key) back to the actual row.
- A table can have **many** non-clustered indexes.
- Looking up a row often requires an extra step: find the entry in the index, then follow the pointer to fetch the full row. This second step is sometimes called a **bookmark lookup** or **key lookup**.

Think of the index at the back of a book — separate from the content, pointing to page numbers.

### Side-by-side

| Aspect | Clustered | Non-Clustered |
|--------|-----------|---------------|
| Physical data order | Defines it | Doesn't affect it |
| Number per table | One | Many |
| Storage | No separate copy (it *is* the data) | Separate structure with pointers |
| Read speed | Faster (data is right there) | Slightly slower (extra pointer hop) |
| Default | Usually the primary key | Created manually |

### Covering index (a key optimization)
If a non-clustered index contains **all the columns a query needs** (in the key or as "included" columns), the DB never has to hop back to the table — the index alone "covers" the query. This avoids the expensive key lookup and is a major performance win.

> **Interview tip:** "A covering index includes every column the query touches, so the engine answers entirely from the index without reading the base table."

---

## 3. Index Data Structures: B-Tree vs Hash

The structure behind an index determines what kinds of queries it can speed up.

### B-Tree (and B+Tree) — the default
The **B-tree** (and its variant B+tree, which is what most real databases use) is the workhorse index structure.

- A balanced tree that keeps data **sorted**.
- Search, insert, and delete are all **O(log n)**.
- In a **B+tree**, all actual values live in the leaf nodes, and leaves are linked together in a sorted list — which makes **range scans** very efficient.

**What B-trees are great at:**
- Equality lookups: `WHERE id = 5`
- Range queries: `WHERE age BETWEEN 20 AND 30`, `WHERE price > 100`
- Sorting: `ORDER BY` on the indexed column (data is already sorted)
- Prefix matching: `WHERE name LIKE 'Jo%'` (but **not** `LIKE '%jo'`)
- `MIN()` / `MAX()` (just read the first/last leaf)

This versatility is why B-trees are the **default index type** in virtually every relational database.

### Hash Index
A **hash index** stores a hash of the column value mapping to the row location.

- Lookups are **O(1)** average — even faster than B-trees for exact matches.
- **Only works for equality** (`WHERE x = 5`). It **cannot** do ranges, sorting, or prefix matches, because hashing destroys order.
- Used by in-memory engines and certain databases (e.g., PostgreSQL has hash indexes; MySQL MEMORY tables use them).

### Comparison

| Feature | B-Tree | Hash |
|---------|--------|------|
| Equality (`=`) | Yes (O(log n)) | Yes (O(1), fastest) |
| Range (`<`, `>`, `BETWEEN`) | Yes | **No** |
| Sorting / `ORDER BY` | Yes | **No** |
| Prefix `LIKE 'abc%'` | Yes | **No** |
| Default choice | **Yes** | Niche |

> **Interview line:** "Use a B-tree unless you have a pure equality-lookup workload where a hash index's O(1) gives a measurable edge — and even then only if the engine supports it well."

### Composite (Multi-Column) Indexes
A **composite index** indexes two or more columns together, e.g. `INDEX(last_name, first_name)`.

The critical concept is the **leftmost prefix rule**: a composite index on `(A, B, C)` can be used for queries filtering on:
- `A`
- `A, B`
- `A, B, C`

But **NOT** for queries that filter only on `B`, only on `C`, or on `B, C` — because the index is sorted by A first, then B within each A, then C. It's like a phone book sorted by last name then first name: useless if you only know someone's first name.

**Column order matters enormously.** Put the most selective / most frequently filtered column first, and put equality-filter columns before range-filter columns.

```sql
-- Good for: WHERE country = 'US' AND city = 'NYC'
-- Good for: WHERE country = 'US'
-- BAD for:  WHERE city = 'NYC'   (skips the leftmost column)
CREATE INDEX idx_loc ON users (country, city);
```

---

## 4. When Indexes DON'T Help (or Actively Hurt)

Knowing when *not* to rely on an index is what separates junior from senior answers.

### Indexes are ignored or useless when:

1. **Low selectivity / low cardinality columns.** Indexing a `gender` or `is_active` boolean column is usually pointless — if 50% of rows match, the DB decides a full scan is cheaper than jumping back and forth through an index. Indexes shine on **high-cardinality** columns (many distinct values, like email or user_id).

2. **Functions or operations on the indexed column.** Wrapping the column in a function breaks index usage:
   ```sql
   WHERE YEAR(created_at) = 2024   -- index on created_at NOT used
   WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'  -- index USED
   ```
   The fix is to rewrite the query so the column stays "bare," or use a function-based/expression index.

3. **Leading wildcard `LIKE '%term'`.** A B-tree can't use an index when the unknown part is at the front.

4. **Small tables.** If a table has 50 rows, scanning all of them is trivially fast and the optimizer skips the index.

5. **Querying a large fraction of the table.** If you're returning 80% of rows anyway, a sequential scan beats millions of index lookups.

6. **Type mismatch / implicit conversion.** Comparing an indexed integer column to a string can silently disable the index.

7. **Too many indexes hurting writes.** As noted, every extra index slows down INSERT/UPDATE/DELETE and bloats storage. Unused indexes are pure cost.

### Other downsides to remember
- **Write amplification** — heavy-write tables suffer with many indexes.
- **Index fragmentation** — over time, indexes get fragmented and need rebuilding/reorganizing.
- **Storage** — indexes can collectively be larger than the table itself.

---

## 5. Reading a Query Execution Plan (EXPLAIN)

`EXPLAIN` (and `EXPLAIN ANALYZE`) shows you **how the database plans to execute a query** — and whether it's actually using your indexes. This is the practical, hands-on skill interviewers love to probe.

### How to run it
```sql
EXPLAIN SELECT * FROM users WHERE email = 'a@b.com';
-- Postgres: EXPLAIN ANALYZE actually runs it and shows real timings
-- MySQL:    EXPLAIN ... or EXPLAIN ANALYZE in newer versions
```

### Key things to look for

**Access type / scan type — the single most important field:**
- **Seq Scan / Full Table Scan / `type: ALL`** — reading every row. A red flag on big tables. ⚠️
- **Index Scan / Range Scan / `type: range`** — using an index to scan a range. Good.
- **Index Seek / Ref / `type: ref`** — using an index for a lookup. Good.
- **Index Only Scan / "Using index"** — a covering index answered the whole query without touching the table. Best. ✅

**Other important fields:**
- **`rows` (estimated)** — how many rows the planner thinks it'll examine. A huge number where you expect few signals a missing or unused index.
- **`key` / Index Name** — which index (if any) was actually chosen. `NULL`/none means no index used.
- **`cost`** — the planner's relative estimate of effort (startup cost..total cost). Lower is better; used to compare plans.
- **`Extra` (MySQL)** — watch for `Using filesort` (sorting in memory/disk, often fixable with an index on the ORDER BY column) and `Using temporary` (temp table created — can be expensive).
- **Actual time vs estimated rows** (with `EXPLAIN ANALYZE`) — big gaps between estimated and actual rows mean **stale statistics**; run `ANALYZE`/`UPDATE STATISTICS` to fix the planner's estimates.

### A practical workflow
1. Run `EXPLAIN` on a slow query.
2. Spot a `Seq Scan` / full scan on a large table where you filter on a column.
3. Add an appropriate index on that column (or composite columns).
4. Re-run `EXPLAIN` and confirm it now shows an Index Scan/Seek and far fewer estimated rows.
5. Verify with `EXPLAIN ANALYZE` that real execution time dropped.

> **Interview gold:** "I'd `EXPLAIN` the query, look at the access type — if it's a sequential scan on a large filtered column, I'd add an index, then re-check that the plan switched to an index scan and the estimated rows dropped."

---

## Quick Revision Cheat Sheet

- **Index** = extra sorted structure → faster reads, slower writes, more storage.
- **Clustered** = data physically sorted by it, one per table (usually the PK). **Non-clustered** = separate structure with pointers, many per table.
- **Covering index** = contains all columns a query needs → no table lookup.
- **B-tree** = default, handles equality + ranges + sorting. **Hash** = O(1) equality only, no ranges.
- **Composite index** = leftmost-prefix rule; column order matters; equality columns before range columns.
- **Indexes don't help** on low-cardinality columns, functions on columns, leading `%wildcards`, small tables, or when most rows are returned.
- **EXPLAIN** = check the access type; full/seq scan on a big filtered table = add an index; aim for index scan/seek and a low estimated row count.

---

## Common Interview Questions to Self-Test

1. What's the difference between a clustered and non-clustered index? How many of each can a table have?
2. Why shouldn't you index every column?
3. When would a hash index beat a B-tree, and what can't it do?
4. Explain the leftmost-prefix rule for composite indexes with an example.
5. Give three reasons a query might ignore an existing index.
6. What is a covering index and why is it fast?
7. You see a `Seq Scan` in your EXPLAIN output on a 10M-row table — walk me through what you'd do.
8. What's the tradeoff of adding an index to a write-heavy table?
