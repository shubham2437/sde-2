# Database Indexing & Scans — Explained With Examples (Postgres)

An expanded, example-driven walkthrough based on AkashSDas's *"Database Indexing."* Every concept below has a concrete worked example, and there's a dedicated section explaining the term **heap**, which everything else depends on.

**Source:** [Database Indexing — AkashSDas (Medium)](https://medium.com/@akashsdas_dev/database-indexing-e10362624ed3)

---

## 0. First: What Does "Heap" Mean? (read this before anything else)

The word **heap** appears constantly in this topic, so let's nail it down.

> **The heap is the actual table — the raw, unordered collection of data pages where your real row data is stored on disk.**

In PostgreSQL, every ordinary table is stored as a **heap**: when you `INSERT` a row, Postgres drops it into the first data page that has free space, in **no particular order**. It's literally a "heap" (a pile) of rows. An index, by contrast, is a *separate*, *sorted* structure that points **into** that pile.

### A concrete picture

Say you have this table:

```sql
CREATE TABLE employees (
  id   INT,
  name TEXT,
  city TEXT
);
```

After some inserts, the **heap** (the table file) physically looks like an unordered pile of pages:

```
HEAP (the employees table on disk)
┌─ Page 0 ──────────────────────┐
│ (1, 'Asha',  'Pune')          │   ← each row has a physical
│ (5, 'Ben',   'Delhi')         │     address called ctid =
├─ Page 1 ──────────────────────┤     (page number, row offset)
│ (2, 'Carl',  'Mumbai')        │     e.g. 'Asha' is at ctid (0,1)
│ (9, 'Dina',  'Pune')          │
├─ Page 2 ──────────────────────┤
│ (7, 'Eve',   'Delhi')         │
└───────────────────────────────┘
```

Notice the `id`s are **not sorted** — that's the heap. To find `id = 7` without an index, Postgres must read every page (a **sequential scan / full table scan**).

Now build an index on `id`:

```sql
CREATE INDEX id_idx ON employees(id);
```

The **index** is a separate sorted structure (a B+tree) that maps each `id` to the heap address (`ctid`) where that row lives:

```
INDEX id_idx (sorted by id)        HEAP (the table)
  1 ──────────────► ctid (0,0)  →  (1,'Asha','Pune')
  2 ──────────────► ctid (1,0)  →  (2,'Carl','Mumbai')
  5 ──────────────► ctid (0,1)  →  (5,'Ben','Delhi')
  7 ──────────────► ctid (2,0)  →  (7,'Eve','Delhi')
  9 ──────────────► ctid (1,1)  →  (9,'Dina','Pune')
```

So whenever you read **"go back to the heap"** or **"heap fetch,"** it means: *the index gave us the row's address, now jump to the table file and read the full row.* That second hop is exactly what makes an Index Scan slower than an Index-Only Scan (which never touches the heap).

> **One-line definition for interviews:** "The heap is the unordered table storage where rows actually live; an index is a sorted pointer structure into the heap. Touching the heap means fetching the real row after the index located it."

**Nuance worth mentioning:** This heap model is the Postgres way. In **MySQL/InnoDB** there is *no separate heap* — the table is stored *as* the primary-key index (an "index-organized" / clustered table), so the row data sits in the leaf of the PK B+tree. That's why this whole article is Postgres-flavored.

---

## 1. Why This Matters

As tables grow into millions of rows, naive queries slow down — slow reads, blocked writes, unhappy users. Indexing turns second-long queries into millisecond ones, but the wrong indexes (or too many) hurt. You must understand **how the engine executes a query** to index well.

**Example of the pain:** `SELECT * FROM orders WHERE customer_id = 42` on a 10-million-row table with no index reads all 10M rows to find maybe 8. With an index on `customer_id`, it reads ~3 index pages + 8 heap rows. Same query, ~1,000,000× less work.

---

## 2. What an Index Is (and Postgres's two structures)

An index is a separate, sorted data structure layered on the table — like the index at the back of a book that sends you straight to the right page instead of reading the whole book.

- **B-Tree** — Postgres's default; great for equality *and* range lookups. (`WHERE id = 7`, `WHERE id BETWEEN 5 AND 50`.)
- **LSM-Tree** — common in write-heavy NoSQL stores like Cassandra; optimized for fast writes.

**Example:** `CREATE INDEX id_idx ON employees(id);` builds a B-tree so `WHERE id = 7` jumps straight to the row instead of scanning the heap.

---

## 3. `EXPLAIN` and `EXPLAIN ANALYZE` — Your Microscope

`EXPLAIN` shows the **plan** without running the query. `EXPLAIN ANALYZE` actually **runs** it and reports real timings.

**Example:**

```sql
EXPLAIN ANALYZE SELECT id FROM employees WHERE id = 20;
```

might print:

```
Index Only Scan using id_idx on employees
  (cost=0.29..8.31 rows=1 width=4) (actual time=0.025..0.027 rows=1 loops=1)
  Index Cond: (id = 20)
Planning Time: 0.1 ms
Execution Time: 0.05 ms
```

That tells you it used `id_idx`, expected 1 row, and finished in 0.05 ms.

**Caution:** caching (OS- and Postgres-level) makes a repeated query look faster than a cold run, so don't trust a single warm timing.

---

## 4. The Scans — The Core Concept

Postgres has a few strategies to fetch rows; the planner chooses based on **how many rows it expects to match.**

### 4a. Sequential Scan (Full Table Scan)
Reads every row in the heap, top to bottom. Shows as `Seq Scan on <table>`.

**Example — happens because there's no index on `name`:**
```sql
EXPLAIN ANALYZE SELECT id FROM employees WHERE name = 'Eve';
-- Seq Scan on employees  (reads all rows, checks each name)
```
It also happens when the query matches **most** of the table — e.g. `WHERE id > 0` returns everything, so scanning beats index-hopping. **An index existing doesn't mean it gets used.**

**A Seq Scan isn't as dumb as it sounds.** Modern engines (PostgreSQL, [IBM Informix](https://www.ibm.com/docs/en/informix-servers/14.10.0?topic=io-sequential-scans)) apply two big optimizations that make scanning a large table far cheaper than a naive "read one page at a time" loop:

- **Parallel Scans** — multiple CPU **worker** processes split the table among themselves and read different chunks at the same time, then merge the results. This drastically cuts wall-clock time on large datasets, because the work is divided across cores instead of done by one process.
  ```sql
  EXPLAIN ANALYZE SELECT count(*) FROM big_table WHERE temp > 50;
  -- Gather  (workers planned: 4)
  --   ->  Parallel Seq Scan on big_table
  --         Filter: (temp > 50)
  ```
  Here the `Gather` node collects results from 4 `Parallel Seq Scan` workers, each scanning roughly a quarter of the heap. (Postgres decides the worker count from table size and `max_parallel_workers_per_gather`.)

- **Read-Ahead (Prefetching)** — because a sequential scan reads pages in a predictable order, the engine (and the OS) **anticipate the pattern and pull upcoming pages into memory *before* they're actually requested.** So while the CPU is processing page N, page N+1 (and beyond) is already being fetched from disk in the background. This hides disk latency and keeps the scan running at near memory speed instead of stalling on every page read.

Together these are why a Seq Scan over a few million rows can still be fast: the work is **parallelized across cores** and the disk reads are **prefetched ahead of time**, turning what looks like a brute-force full-table read into a streamlined, pipelined operation. This is also part of why the planner sometimes *prefers* a Seq Scan — a prefetched, parallel sequential read of the whole table can beat millions of scattered random index lookups.

### 4b. Index Scan
Matches a **small** subset; the engine walks the index, then **hops to the heap** for columns not in the index. Shows as `Index Scan using <index> on <table>`.

**Example:**
```sql
EXPLAIN ANALYZE SELECT name FROM employees WHERE id = 7;
-- Index Scan using id_idx on employees
--   Index Cond: (id = 7)
```
Why Index *Scan* and not Index-*Only*? Because `name` isn't in `id_idx`, so after the index finds `id = 7` it must **go back to the heap** to read `name`.

> Index Scan = use index to locate the row → hop to the heap for the missing columns.

### 4c. Index-Only Scan (the fastest)
Every column the query needs is already **in the index**, so Postgres never touches the heap.

**Example:**
```sql
SELECT id FROM employees WHERE id = 7;  -- Index-Only Scan: only 'id' needed, and it's in the index
SELECT name FROM employees WHERE id = 7; -- Index Scan: 'name' forces a heap fetch
```

You can *make* the first stay index-only even when selecting more columns by adding them as INCLUDE columns (see §6).

### 4d. Bitmap Index Scan (the middle ground)
Used when a query matches **too many rows for a tidy Index Scan but too few for a full Seq Scan.** Two phases:

1. **Bitmap Index Scan** — scan the index, mark which *heap pages* contain matches in an in-memory bitmap.
2. **Bitmap Heap Scan** — visit those marked pages **in physical order, in bulk**, then **`Recheck Cond`** to drop rows that don't actually qualify.

**Example:**
```sql
EXPLAIN ANALYZE SELECT name FROM employees WHERE city = 'Pune';
-- Bitmap Heap Scan on employees
--   Recheck Cond: (city = 'Pune')
--   ->  Bitmap Index Scan on city_idx
--         Index Cond: (city = 'Pune')
```
The payoff: it turns scattered **random** heap reads into batched **sequential-ish** reads.

### 4e. Combining indexes in a bitmap (AND / OR)
**Example AND:**
```sql
SELECT name FROM grades WHERE grade > 90 AND id < 10000;
-- builds a bitmap for each condition, BitmapAnd them, then one heap fetch
```
**Example OR:**
```sql
SELECT name FROM grades WHERE a = 70 OR b = 70;
-- BitmapOr of two index bitmaps; OR returns more rows, so often slower (or falls back to Seq Scan)
```

### Scan selection, in one line
> **Few rows → Index Scan** (or Index-Only). **Medium → Bitmap Index Scan**. **Large fraction → Sequential Scan.**

---

## 5. Reading the `EXPLAIN` Cost Output

**Example line:**
```
Seq Scan on employees  (cost=0.00..12.25 rows=1000 width=64)
```
Decoded:
- **`cost=0.00..12.25`** → cost to return the **first row** (`0.00`) and **all rows** (`12.25`). High start-up cost = slow-to-first-byte UI.
- **`rows=1000`** → the planner's *estimate* (from statistics), not a guarantee.
- **`width=64`** → estimated average row size in bytes; wide rows cost more memory/bandwidth.

**Practical example of `width` biting you:** `SELECT *` on a table with a 2 KB `description` column makes `width` huge and ships megabytes you didn't need. `SELECT id, name` keeps `width` tiny.

Other takeaways: Postgres reads plans **bottom-to-top**; index an `ORDER BY` column to avoid a costly sort; use `BIGINT` PKs for tables that may exceed ~2 billion rows.

---

## 6. Key vs Non-Key (INCLUDE) Index Columns

- **Key columns** — the columns the index is built/sorted on and that you filter by.
- **Non-key / INCLUDE columns** — extra columns stored in the **index leaf** purely so a query can be answered **without a heap fetch**.

**Example — turning an Index Scan into an Index-Only Scan:**
```sql
-- Before: name lives only in the heap
CREATE INDEX id_idx ON employees(id);
SELECT name FROM employees WHERE id = 7;          -- Index Scan (heap hop for name)

-- After: name rides along inside the index
DROP INDEX id_idx;
CREATE INDEX id_idx ON employees(id) INCLUDE (name);
SELECT name FROM employees WHERE id = 7;          -- Index-Only Scan (no heap hop!)
```
**Tradeoff:** INCLUDE columns **bloat the index**; a bigger index may not fit in memory, clawing back the win. Use it to dodge the heap on hot queries, don't over-stuff.

---

## 7. Composite (Multi-Column) Indexes

A composite index covers several columns and shines for `AND` filters.

**Example setup:** `CREATE INDEX ab_idx ON test (a, b);`

- `WHERE a = 70` → **uses** it (`a` is the **leftmost** column). ✅
- `WHERE a = 70 AND b = 70` → **uses** it efficiently. ✅
- `WHERE b = 70` → **Seq Scan**, because the index is sorted by `a` first; knowing only `b` is like knowing only a first name in a phone book sorted by last name. ❌ (Fix: a separate index on `b`.)

**Example of result-size flipping the strategy on the *same* index:**
```sql
SELECT c FROM test WHERE a = 70;            -- Bitmap Index Scan (many rows)
SELECT c FROM test WHERE a = 70 LIMIT 10;   -- plain Index Scan (LIMIT makes a bitmap not worth building)
```

---

## 8. How the Optimizer Decides Which Index(es) to Use

Postgres uses a **cost-based optimizer**: it simulates plans, estimates rows, picks the cheapest. For `WHERE f1 = 1 AND f2 = 4` with indexes on both:

1. **Use both** — get row IDs from `f1_idx` and `f2_idx`, **BitmapAnd** them. (Result set is medium-sized.)
2. **Use one** — use `f1_idx`, then check `f2 = 4` against the heap. (One column is far more selective or is the PK.)
3. **Use none → Seq Scan** — expected matches are huge, *or* statistics are stale.

**Example of stale stats causing a bad plan:**
```sql
-- You just bulk-loaded 300M rows; Postgres still thinks the table is tiny
ANALYZE big_table;   -- refresh stats so the planner stops choosing dumb plans
```

### Things that *kill* index usage (with examples)
- **Leading wildcard:** `WHERE name LIKE '%Eve%'` → Seq Scan (a sorted B-tree can't seek a substring). But `WHERE name LIKE 'Ev%'` → can use the index.
- **Function on the column:** `WHERE LOWER(name) = 'eve'` ignores a plain `name` index — fix with an expression index `CREATE INDEX ON employees(LOWER(name))`.
- **Selecting non-indexed columns** forces a heap hop (Index Scan instead of Index-Only).

---

## 9. `CREATE INDEX CONCURRENTLY` (production safety)

A plain `CREATE INDEX` **blocks all writes** while building — fine in dev, an outage in production.

**Example:**
```sql
-- DANGER in prod: locks INSERT/UPDATE/DELETE until done
CREATE INDEX grade_idx ON grades (grade);

-- SAFE: writes continue during the build
CREATE INDEX CONCURRENTLY grade_idx ON grades (grade);
```
Tradeoffs: `CONCURRENTLY` is **slower**, uses more resources, and for a **unique** index can fail (leaving an invalid index) if duplicates sneak in mid-build — then you `DROP INDEX` and retry.

---

## 10. Bloom Filters (avoiding pointless DB hits)

A **Bloom filter** is a tiny, in-memory, probabilistic structure answering "**is X in this set?**" with: **definitely NO** (100% sure) or **maybe yes** (then actually query the DB). It never stores the data — only membership bits.

**Worked example (8-bit filter, checking usernames):**
```
Insert "paul":  hash("paul") % 8 = 3   → set bit 3 to 1   →  [0 0 0 1 0 0 0 0]
Insert "asha":  hash("asha") % 8 = 6   → set bit 6 to 1   →  [0 0 0 1 0 0 1 0]

Lookup "mike":  hash("mike") % 8 = 5   → bit 5 is 0  → DEFINITELY NOT in DB → skip the query
Lookup "paul":  hash("paul") % 8 = 3   → bit 3 is 1  → MAYBE in DB → go query to confirm
```
You avoid the database entirely for known-absent keys. Real ones use **multiple hash functions** and large bit arrays; **Counting Bloom filters** allow deletes. Cassandra uses them to skip needless disk reads. Danger: undersize it and everything returns "maybe," defeating the point.

---

## 11. Taming Billion-Row Tables

- **Brute force (big-data):** chunk + parallel process (Hadoop/MapReduce). Expensive, high latency.
- **Smart subset processing (usually what you want):**
  - **Indexing** — jump to relevant rows (still one table). *Example:* `WHERE created_at > now() - interval '1 day'` with an index on `created_at`.
  - **Partitioning** — split one table into chunks by range/hash, each with its own indexes. *Example:* partition `orders` by month so a query for January only scans the January partition.
  - **Sharding** — spread partitions across machines (horizontal scaling). Powerful but adds shard-routing and painful cross-shard joins.
  - Hierarchy: **Sharding → Partitioning → Indexes → Disk Pages.**
- **Rethink the model:** archive old rows, set retention policies, use summary tables / materialized views, pre-aggregate.

---

## 12. The Cost of Long-Running Transactions (MVCC + VACUUM)

Postgres uses **MVCC**: every `UPDATE`/`DELETE` writes a **new row version (tuple)** and leaves the old one behind as a **dead tuple** until cleanup.

**Example of bloat from a rollback:**
```sql
BEGIN;
UPDATE accounts SET balance = balance * 1.05;  -- touches 5,000,000 rows → 5M new tuples
ROLLBACK;                                       -- all 5M new tuples are now DEAD but still on disk
```
Those dead tuples sit in the heap until **`VACUUM`** reclaims them. Until then, a page might hold 1 live row + 999 dead, wasting I/O, and every read does a **visibility check** ("is this tuple from a committed transaction?").

**HOT (Heap-Only Tuple) optimization:** if an update changes only **non-indexed** columns and the new version fits on the **same page**, Postgres skips updating indexes — a reason *not* to over-index churning columns.

**Maintenance example:**
```sql
VACUUM (VERBOSE) accounts;   -- reclaim dead tuples
ANALYZE accounts;            -- refresh planner statistics
-- monitor: SELECT relname, n_dead_tup FROM pg_stat_user_tables;
```
**Contrast:** Oracle / SQL Server / MySQL InnoDB use **undo logs** for *eager* rollback — cleaner disk, but longer startup/downtime after a big rollback.

---

## Quick Revision — Scan Decision Table

| Situation | Scan chosen | Plan keyword | Touches heap? |
|-----------|-------------|--------------|---------------|
| No index / matches most rows | Sequential Scan | `Seq Scan` | Yes (whole table) |
| Few rows, needs non-indexed columns | Index Scan | `Index Scan using ...` | Yes (per row) |
| Few rows, all columns in index | Index-Only Scan | `Index Only Scan` | **No** |
| Medium match count | Bitmap Index Scan | `Bitmap Index Scan` + `Bitmap Heap Scan` | Yes (in bulk) |
| Multiple `AND`/`OR` with separate indexes | Bitmap combine | `BitmapAnd` / `BitmapOr` | Yes (in bulk) |

---

## Self-Test Questions

1. **In your own words, what is the heap, and what does "going back to the heap" mean?**
2. Why does Postgres sometimes ignore a perfectly good index and do a Seq Scan?
3. Walk through the two phases of a Bitmap Index Scan and what `Recheck Cond` does.
4. What's the exact difference between an Index Scan and an Index-Only Scan (in terms of the heap)?
5. In a composite index on `(a, b)`, why does `WHERE b = 70` cause a sequential scan?
6. What do the two numbers in `cost=0.00..12.25` mean?
7. How do `INCLUDE` columns enable an index-only scan, and what's the downside?
8. Why does `CREATE INDEX CONCURRENTLY` exist and what are its tradeoffs?
9. How does a Bloom filter let you skip the database, and when does it stop helping?
10. What is a dead tuple, how does it appear, and how does `VACUUM` address it?
11. Why does MySQL/InnoDB not have a separate "heap" the way Postgres does?

---

**Source / further reading:** [Database Indexing — AkashSDas, Medium (Apr 2025)](https://medium.com/@akashsdas_dev/database-indexing-e10362624ed3)
