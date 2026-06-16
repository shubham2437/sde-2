# Database Indexing & Scans — Practical Deep Dive (Postgres)

A detailed walkthrough based on AkashSDas's article *"Database Indexing,"* expanded with explanations of each scan type and indexing concept. Everything below is rephrased and elaborated for study; read the original for the full hands-on examples.

**Source:** [Database Indexing — AkashSDas (Medium)](https://medium.com/@akashsdas_dev/database-indexing-e10362624ed3)

---

## 0. Why This Matters

Databases store and fetch data well, but as tables grow into the millions of rows, naive queries slow down badly — slow reads, blocked writes, unhappy users. Indexing is the main lever to fix this, turning second-long queries into millisecond ones. But indexes are a double-edged sword: the wrong ones, or too many, can hurt more than help. The article's whole premise is that you have to understand *how the engine actually executes a query* to index well.

---

## 1. Building a Realistic Test Dataset (1M rows)

To feel indexing effects you need volume. In Postgres you can fabricate a million-row table cheaply using `generate_series()` and `random()`:

```sql
CREATE TABLE temperature (temp INT);

INSERT INTO temperature(temp)
SELECT (random() * 100)::INT
FROM generate_series(1, 1000000);

SELECT COUNT(*) FROM temperature;
```

This pattern — `generate_series` to manufacture rows — is the standard trick for benchmarking index behavior locally.

---

## 2. What an Index Is (and Postgres's two structures)

An index is a separate data structure layered on top of a table that lets the engine jump to rows instead of reading every one — exactly like the index at the back of a book. The article highlights two structures:

- **B-Tree** — Postgres's default, great for equality *and* range lookups. The focus of the whole article.
- **LSM-Tree** — more common in write-heavy NoSQL stores like Cassandra; optimized for fast writes.

---

## 3. `EXPLAIN` and `EXPLAIN ANALYZE` — Your Microscope

`EXPLAIN` shows the **plan** the optimizer intends to use without running the query. `EXPLAIN ANALYZE` actually **runs** it and reports real timings, so it's the debugging tool of choice.

```sql
EXPLAIN ANALYZE SELECT id FROM employees WHERE id = 20;
```

A caution the article repeats: **caching** (OS-level and Postgres-level) makes repeated runs of the same query look faster than they truly are, so don't trust a single warm run.

---

## 4. The Scans — The Core of the Article

This is the part you asked about most. Postgres has three main strategies for getting rows, and the planner picks based on **how many rows it expects to match.**

### 4a. Sequential Scan (Full Table Scan)
The engine reads the table top to bottom, every row. You'll see `Seq Scan on <table>` in the plan. It happens when:
- there's no filter,
- there's no usable index, **or**
- the planner calculates that scanning everything is *cheaper* than using an index (because the query matches a large fraction of rows).

That last point is critical: **an index existing does not mean it gets used.** If you're going to touch most of the table anyway, sequential reads (which are fast and linear) beat thousands of random index jumps. Postgres can even run a **Parallel Seq Scan** across threads, but it's still fundamentally expensive on big tables.

### 4b. Index Scan
When the query matches only a **small subset** of rows and a usable index exists, the engine walks the index to find matching entries, then **follows pointers back to the table (the "heap")** to fetch any columns not in the index. You'll see `Index Scan using <index> on <table>`.

> Index Scan = use index to locate rows → hop to the heap to fetch the rest of the columns.

### 4c. Index-Only Scan (the fastest)
If **every column the query needs is already in the index**, Postgres never touches the heap at all. This is the fastest path. Two ways to get it:
- Select only indexed columns (`SELECT id ... WHERE id = 7` when `id` is indexed), or
- Add the extra columns as **non-key INCLUDE columns** (see §6).

```sql
SELECT id FROM grades WHERE id = 7;   -- Index-Only Scan: id is in the index
SELECT name FROM grades WHERE id = 7; -- Index Scan: name must come from the heap
```

### 4d. Bitmap Index Scan (Postgres's clever middle ground)
Used when a query matches **too many rows for an efficient Index Scan but too few to justify a full Seq Scan.** It works in two phases:
1. **Bitmap Index Scan** — scan the index and mark which *table pages* contain matches in an in-memory bitmap.
2. **Bitmap Heap Scan** — visit those marked pages **in physical order, in bulk**, then re-check the condition (you'll see a `Recheck Cond` step) to drop rows that don't actually qualify.

The payoff: it converts scattered **random** disk access into batched **sequential-ish** access, which is far cheaper.

```
Bitmap Index Scan on grades_grade_idx
Bitmap Heap Scan on grades
   Recheck Cond: (grade > 90)
```

### 4e. Combining indexes in a bitmap (AND / OR)
Bitmap scans can fuse **multiple indexes** for one query:
- `WHERE grade > 90 AND id < 10000` → build a bitmap per condition, **BitmapAnd** them, then fetch.
- `WHERE a = 70 OR b = 70` → **BitmapOr** the two bitmaps. But `OR` usually returns more rows, so Postgres often falls back to a (parallel) sequential scan; even when it uses BitmapOr it's slower than the `AND` case because of the larger result set and the recheck.

### Scan selection, in one line
> Few rows → **Index Scan** (or Index-Only). Medium number of rows → **Bitmap Index Scan**. Large fraction of the table → **Sequential Scan**.

---

## 5. Reading the `EXPLAIN` Cost Output

A line like:

```
Seq Scan on employees  (cost=0.00..12.25 rows=1000 width=64)
```

decodes as:
- **`cost=0.00..12.25`** → two numbers: estimated cost to return the **first row** (`0.00`) and to return **all rows** (`12.25`). A high start-up cost means a slow-to-first-byte UI.
- **`rows=1000`** → how many rows the planner *estimates* (from statistics), not a guarantee.
- **`width=64`** → estimated average row size in bytes; wide rows cost more memory and bandwidth.

Practical takeaways the article stresses:
- Plans are read **bottom-to-top** in Postgres — the lowest node runs first.
- Use an index to support `ORDER BY`, otherwise the engine fetches everything then sorts (expensive).
- Avoid `SELECT *`, especially with large/blob columns — you ship unnecessary bytes and you forfeit index-only scans.
- Use `BIGINT` for primary keys on tables that might exceed ~2 billion rows (an `INTEGER` PK will overflow).

---

## 6. Key vs Non-Key (INCLUDE) Index Columns

- **Key columns** are the columns you actually index and filter on — the B-tree is built on them. Lookups follow pointers from the index back to the heap for anything else.
- **Non-key / included columns** are extra columns stored *in the index leaf* purely so the index can satisfy more queries without a heap trip:

```sql
CREATE INDEX grade_idx ON students(grade) INCLUDE (id);
```

Now `SELECT id, grade ... WHERE grade = ...` is answered entirely from the index → **Index-Only Scan**. The tradeoff: included columns **bloat the index**, and a bigger index may not fit in memory, which can claw back the gains. Rule of thumb from the article: *use INCLUDE to dodge the heap, but don't over-stuff the index.*

---

## 7. Composite (Multi-Column) Indexes

A composite index covers several columns and shines for `AND` filters across them:

```sql
CREATE INDEX ON test (a, b);
```

The crucial rule, demonstrated in the article on a 12M-row table:
- `WHERE a = 70` → **uses** the index (`a` is the **left-most** column).
- `WHERE a = 70 AND b = 70` → **uses** it efficiently (Index Scan).
- `WHERE b = 70` → **Sequential Scan**, because Postgres can only read the composite **left-to-right**, never right-to-left. To filter by `b` alone you need a separate index on `b`.

Also shown: result-set size flips the strategy even on the same index — `WHERE a = 70` may use a Bitmap Index Scan, but `WHERE a = 70 LIMIT 10` switches to a plain Index Scan because the small `LIMIT` makes building a bitmap not worth the overhead.

---

## 8. How the Optimizer Decides Which Index(es) to Use

Postgres uses a **cost-based optimizer**: it simulates candidate plans, estimates rows for each, and picks the cheapest. The article walks three outcomes for `WHERE f1 = 1 AND f2 = 4` with indexes on both columns:

1. **Use both indexes** — find matching row IDs from each index, then **BitmapAnd** them. Chosen when the result set is medium: too big for one index, too small for a full scan.
2. **Use only one index** — e.g. use `f1_idx`, then check `f2 = 4` against the heap. Chosen when one column is highly selective (few rows) or is the primary key, making the second index not worth it.
3. **Use no index (Seq Scan)** — chosen when the expected match count is very large, *or* when statistics are stale/missing so the planner can't reason well.

### Keep statistics fresh
After bulk loads, Postgres doesn't auto-refresh stats, so it can make terrible choices (e.g. thinking a 300M-row table has a handful of rows). Run:

```sql
VACUUM (VERBOSE) table_name;
ANALYZE table_name;
```

### Nudging the planner
Postgres lacks Oracle/MySQL-style hints, but you can: rewrite to filter on the most selective condition first, add **expression indexes** for computed predicates, tweak planner cost settings (`enable_seqscan = off`) in edge cases, or use the `pg_hint_plan` extension. Use sparingly — the optimizer is usually right.

### Things that *kill* index usage
- **Leading-wildcard `LIKE '%Zs%'`** — even an indexed column falls back to a full scan, because a substring match can't use the sorted B-tree.
- **Selecting non-indexed columns** forces a heap hop (Index Scan instead of Index-Only).

---

## 9. `CREATE INDEX CONCURRENTLY` (production safety)

A plain `CREATE INDEX` **blocks all writes** (INSERT/UPDATE/DELETE) on the table while it builds — fine in dev, an outage in production. The safe form:

```sql
CREATE INDEX CONCURRENTLY grade_idx ON grades (grade);
```

It avoids the write lock by scanning the table multiple times and waiting for in-flight transactions to finish. Tradeoffs: it's **slower**, uses **more resources**, and for a **unique** index it can fail (leaving an invalid index) if duplicates appear mid-build — requiring a manual `DROP INDEX` and retry. The Postgres team has even discussed making concurrent builds the default because plain builds are so risky in live systems.

---

## 10. Bloom Filters (avoiding pointless DB hits)

A **Bloom filter** is a tiny, in-memory, probabilistic structure that answers "**is X in this set?**" with two possible replies:
- **Definitely NO** (100% certain), or
- **Maybe yes** (small false-positive chance) → then you actually query the DB.

It never stores the data itself; it only tests membership. Mechanically, you hash the value into bit positions and set those bits; on lookup, if any required bit is `0`, the item is certainly absent. This lets a service skip the database entirely for known-absent keys (e.g., "does this username exist?"). Real implementations use **multiple hash functions** and large bit arrays; **Counting Bloom filters** add deletion support. Cassandra uses them to avoid needless disk reads. The danger: if you undersize it, too many bits get set, everything returns "maybe," and you lose the benefit — so size and tune (hash count + array size) carefully.

---

## 11. Taming Billion-Row Tables

Three broad approaches, from blunt to smart:

- **Brute force (big-data style):** chunk the table and process in parallel (Hadoop/MapReduce style). Expensive, high latency, not great for frequent queries.
- **Smart subset processing (usually what you want):** reduce how much data you touch via layered tactics —
  - **Indexing:** still operate on the whole table, but jump straight to relevant rows.
  - **Partitioning:** split the table into smaller chunks on disk (by range or hash); each partition has its own indexes, so you scan fewer of them.
  - **Sharding:** spread partitions across multiple machines (horizontal scaling). Powerful but introduces shard-routing logic and painful cross-shard joins/transactions/aggregations.
  - Mental hierarchy: **Sharding → Partitioning → Indexes → Disk Pages.**
- **Rethink the data model:** ask *why* the table is so huge. Archive old rows, set retention policies, use summary tables / materialized views, or pre-aggregate. Design schemas so stale data ages out instead of accumulating forever.

---

## 12. The Cost of Long-Running Transactions (MVCC + VACUUM)

Postgres uses **MVCC (Multi-Version Concurrency Control)**:
- Every `UPDATE`/`DELETE` creates a **new row version (tuple)**; the old one lingers until cleaned up.
- Indexes pointing at the row must be updated to the new version — **unless** the change fits on the same page and the columns aren't indexed, in which case a **HOT (Heap-Only Tuple)** update skips index maintenance entirely (this depends on free page space, governed by **fill factor**).

**On rollback** of a transaction that touched millions of rows: all those new versions become **dead tuples** but stay on disk/memory — Postgres cleans up **lazily**, not immediately.

**`VACUUM`** is the background process that scans for dead tuples and reclaims their space. Until it runs, pages are **bloated**: a read might pull a page that's 1 live row and 999 dead ones, wasting I/O, and every read pays a **visibility check** ("is this tuple from a committed transaction?"). The guidance: run `VACUUM` and `ANALYZE` aggressively on write-heavy systems and watch dead-tuple counts via `pg_stat_user_tables`.

**Contrast with eager-cleanup engines** (Oracle, SQL Server, MySQL InnoDB): they use **undo logs** to roll back immediately and won't fully start until rollback completes — cleaner on-disk state, but longer startup/downtime.

---

## Quick Revision — Scan Decision Table

| Situation | Scan chosen | Plan keyword |
|-----------|-------------|--------------|
| No index / matches most rows | Sequential Scan | `Seq Scan` |
| Few matching rows, needs heap columns | Index Scan | `Index Scan using ...` |
| Few matching rows, all columns in index | Index-Only Scan | `Index Only Scan` |
| Medium match count | Bitmap Index Scan | `Bitmap Index Scan` + `Bitmap Heap Scan` |
| Multiple conditions (`AND`/`OR`) with separate indexes | Bitmap combine | `BitmapAnd` / `BitmapOr` |

---

## Self-Test Questions

1. Why does Postgres sometimes ignore a perfectly good index and do a Seq Scan?
2. Walk through the two phases of a Bitmap Index Scan and what `Recheck Cond` does.
3. What's the exact difference between an Index Scan and an Index-Only Scan?
4. In a composite index on `(a, b)`, why does `WHERE b = 70` cause a sequential scan?
5. What do the two numbers in `cost=0.00..12.25` mean?
6. How do `INCLUDE` (non-key) columns enable an index-only scan, and what's the downside?
7. Why does `CREATE INDEX CONCURRENTLY` exist and what are its tradeoffs?
8. How does a Bloom filter let you skip the database, and when does it stop helping?
9. What is a dead tuple, how does it appear, and how does `VACUUM` address it?
10. Order these by scope: indexing, sharding, partitioning, disk pages.

---

**Source / further reading:** [Database Indexing — AkashSDas, Medium (Apr 2025)](https://medium.com/@akashsdas_dev/database-indexing-e10362624ed3)
