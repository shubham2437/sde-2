# Index Types & Variants — Complete Catalogue

There are far more index variants than "B-tree on one column." This guide catalogues them all, organized four ways: by **number of columns**, by **what they index**, by **internal structure**, and by **constraint role**. Multi-valued and multi-column (composite) indexes get the deepest treatment since those are the most commonly under-explained.

---

## A. By Number of Columns

### 1. Single-Column Index
The basic case — one index on one column. Good when you filter/sort by that column alone.
```sql
CREATE INDEX idx_email ON users (email);
```

### 2. Composite / Multi-Column Index (deep dive)
An index built on **two or more columns together**, sorted first by the first column, then by the second *within* each value of the first, and so on.
```sql
CREATE INDEX idx_name ON users (last_name, first_name);
```

**Mental model:** a phone book sorted by last name, then first name. The ordering is hierarchical.

#### The leftmost-prefix rule (the single most important concept)
An index on `(A, B, C)` can serve queries that filter on a **left prefix** of the columns:

| Query filters on | Can use `(A, B, C)`? |
|------------------|----------------------|
| `A` | ✅ Yes |
| `A, B` | ✅ Yes |
| `A, B, C` | ✅ Yes (fully) |
| `A, C` (skipping B) | ⚠️ Partial — uses A only, then filters C |
| `B` alone | ❌ No |
| `C` alone | ❌ No |
| `B, C` | ❌ No |

Why? Because the data is sorted by A first. Knowing only `B` is like knowing only someone's first name in a phone book sorted by last name — you'd have to scan everything.

#### Column-ordering rules (senior-level design)
1. **Equality columns before range columns.** For `WHERE a = ? AND b > ?`, use `(a, b)`. Once a range (`>`, `<`, `BETWEEN`, `LIKE 'x%'`) is used on a column, columns *after* it in the index can't be used for seeking — only for filtering. So put the range column **last**.
2. **Most selective / most-frequently-filtered column first** (with the caveat that supporting `ORDER BY` and the equality-first rule can override pure selectivity).
3. **Match `ORDER BY` to enable sort elimination.** If the index order matches the query's sort, the engine skips the sort step entirely (no `filesort` / no separate Sort node). For `ORDER BY a, b DESC` you may need a mixed-direction index `(a ASC, b DESC)`.

#### Composite vs index merge
Two single-column indexes on `a` and `b` *can* be combined by the optimizer (index merge / bitmap-and) for `WHERE a=? AND b=?`, but a single well-ordered composite `(a, b)` is almost always faster because it does one traversal instead of two plus a merge.

#### Redundancy
A standalone index on `(a)` is **redundant** if you already have `(a, b)` — the composite serves all left-prefix `a` queries. Redundant indexes are pure write + storage cost; audit and drop them.

### 3. Covering Index
Any index (usually composite) that contains **every column a query needs**, so the engine answers entirely from the index without touching the table (an **index-only scan**). You can extend coverage without bloating the search key using `INCLUDE`:
```sql
-- key stays narrow on customer_id; amount/status just ride in the leaf
CREATE INDEX idx ON orders (customer_id) INCLUDE (amount, status);
```

---

## B. By What They Index

### 4. Multi-Valued Index (deep dive — MySQL 8.0.17+)
This is the one you asked about, and it's special: **a single data row can produce multiple index entries.**

Normal indexes have a **1:1 relationship** — one index record per row. A **multi-valued index** has a **1:N relationship** — one row whose column holds an *array* gets **one index entry per array element**. It's designed to index **JSON arrays**.

#### How it's defined
You index an array extracted from a JSON column using `CAST(... AS <type> ARRAY)` as a functional key part:
```sql
CREATE TABLE customers (
  id        BIGINT PRIMARY KEY,
  custinfo  JSON,
  INDEX zips ( (CAST(custinfo->'$.zipcodes' AS UNSIGNED ARRAY)) )
);
```
If a row's `zipcodes` is `[94507, 94582, 94100]`, that **one row creates three index entries** — one per zip.

#### Queries that use it
A multi-valued index is used by three JSON functions:
```sql
-- "is this value a member of the array?"
SELECT * FROM customers
WHERE 94507 MEMBER OF (custinfo->'$.zipcodes');

-- "does the array contain all of these?"
SELECT * FROM customers
WHERE JSON_CONTAINS(custinfo->'$.zipcodes', CAST('[94507,94582]' AS JSON));

-- "does the array contain any of these?"
SELECT * FROM customers
WHERE JSON_OVERLAPS(custinfo->'$.zipcodes', CAST('[94507,94582]' AS JSON));
```

#### Key rules / limitations
- An index can have **only one multi-valued key part** (you can't have two array key parts in the same index).
- It can be **combined with ordinary scalar key parts** in a composite (e.g., one normal column + one array part).
- Supported types for the array cast include integer types, `DECIMAL`, `DATE/DATETIME/TIME`, `CHAR/VARCHAR/BINARY`.
- The array can't contain `NULL`; empty arrays index nothing for that row.
- It's a **secondary index only** — can't be a primary key, can't be unique, can't be used for `ORDER BY`/`GROUP BY`.
- Conceptually it's MySQL's relational equivalent of what Postgres does with **GIN indexes on arrays/JSONB** (see #11).

> **Interview line:** "A multi-valued index breaks the usual one-entry-per-row rule — it stores one entry per element of a JSON array, so `MEMBER OF` / `JSON_CONTAINS` / `JSON_OVERLAPS` can use an index instead of scanning and parsing JSON for every row."

### 5. Prefix Index (MySQL)
Index only the **first N characters** of a long string column to save space:
```sql
CREATE INDEX idx_name ON users (name(10));   -- index first 10 chars
```
- Pros: smaller index, cheaper for long `TEXT`/`VARCHAR`.
- Cons: **can't be a covering index** (it stores a truncated value), can't fully satisfy `ORDER BY`, and you must pick a prefix length long enough to stay **selective** (measure distinct-prefix ratio).

### 6. Expression / Functional Index
Index the **result of an expression** so a transformed predicate stays index-usable (sargable):
```sql
CREATE INDEX idx_lower_email ON users (LOWER(email));
-- enables: WHERE LOWER(email) = 'a@b.com'
```
Without it, `WHERE LOWER(email) = ...` would ignore a plain `email` index.

### 7. Partial / Filtered Index
Index only the **subset of rows** matching a condition — smaller, faster, cheaper to maintain:
```sql
-- Postgres
CREATE INDEX idx_pending ON orders (created_at) WHERE shipped = false;
```
Ideal for hot subsets: pending jobs, active users, `WHERE deleted_at IS NULL`. (SQL Server calls these *filtered indexes*; MySQL has no direct equivalent — you emulate with a generated column.)

### 8. Generated / Computed Column Index
Materialize a computed value into a (stored or virtual) generated column, then index it. The standard MySQL trick to index a specific JSON scalar path:
```sql
ALTER TABLE users
  ADD COLUMN city VARCHAR(50) AS (data->>'$.city') VIRTUAL,
  ADD INDEX idx_city (city);
```

---

## C. By Internal Structure

### 9. B-Tree / B+Tree
The default. Handles equality, ranges, sorting, prefix matches. Covered exhaustively elsewhere — this is what `CREATE INDEX` builds unless you say otherwise.

### 10. Hash Index
O(1) average for **equality only** — no ranges, no sorting, no prefix. Used by in-memory engines (MySQL `MEMORY` tables) and available explicitly in Postgres.

### 11. Inverted Index (Full-Text / GIN)
Maps each **token/element → the list of rows containing it**. This is how full-text search and array/JSONB containment work:
- **MySQL:** `FULLTEXT` indexes (`MATCH ... AGAINST`).
- **Postgres GIN:** full-text (`tsvector`), `jsonb`, and **array containment** — like a multi-valued index, one entry per element. GIN is conceptually the Postgres counterpart to MySQL's multi-valued index for arrays/JSON.

### 12. Spatial / R-Tree Index
For geometric/geographic data — indexes bounding boxes for "within / intersects / nearest" queries.
- **MySQL:** `SPATIAL INDEX` (R-tree) on geometry columns.
- **Postgres/PostGIS:** **GiST** indexes.

### 13. BRIN (Block Range Index — Postgres)
Stores **min/max per block range** instead of per row. Tiny footprint; only effective when the column's values **correlate with physical row order** (e.g., an append-only timestamp on a huge time-series table).

### 14. Bitmap Index
A bitmap per distinct value; excellent for **low-cardinality** columns in **OLAP/data-warehouse** workloads and for combining many `AND`/`OR` predicates. **Poor for OLTP** because updates lock large bitmaps. Common in Oracle and columnar engines.

### 15. Clustered vs Non-Clustered (structural placement)
- **Clustered:** the index *is* the table; rows physically ordered by it; one per table (usually the PK).
- **Non-clustered:** a separate structure pointing back to the row; many per table.
(Full treatment in the earlier guides.)

---

## D. By Constraint Role

### 16. Primary Key Index
Automatically created, unique + not null; usually the clustered index.

### 17. Unique Index
Enforces uniqueness **and** speeds lookups. Can be composite (`UNIQUE (a, b)` — the *combination* must be unique). Can be **partial unique** to enforce "unique among active rows":
```sql
-- only one active email allowed, soft-deleted duplicates fine
CREATE UNIQUE INDEX uq_active_email ON users (email) WHERE deleted_at IS NULL;
```

### 18. Foreign-Key Supporting Index
Many engines do **not** auto-index the *referencing* FK column. Without it, deletes/updates on the parent take broad locks and scan the child — a classic source of slow operations and **deadlocks**. Always index FK columns on the child side.

---

## Quick Decision Cheat Sheet

| Need | Reach for |
|------|-----------|
| Filter/sort on one column | Single-column B-tree |
| Filter on several columns together | **Composite** (equality cols first, range last, leftmost-prefix) |
| Avoid touching the table at all | **Covering** index (`INCLUDE`) |
| Search inside a JSON **array** (MySQL) | **Multi-valued** index + `MEMBER OF`/`JSON_CONTAINS`/`JSON_OVERLAPS` |
| Array/JSONB containment (Postgres) | **GIN** index |
| Long string column, save space | **Prefix** index |
| Query on a transformed value | **Expression/functional** index |
| Only a hot subset of rows matters | **Partial/filtered** index |
| Full-text search | **Inverted** (FULLTEXT / GIN tsvector) |
| Geospatial | **Spatial** (R-tree / GiST) |
| Huge naturally-ordered table | **BRIN** |
| Low-cardinality, OLAP, read-mostly | **Bitmap** |
| Pure equality, in-memory | **Hash** |

---

## Self-Test Questions

1. What makes a multi-valued index different from every other index type? Give the three functions that use it.
2. Why can't an index on `(a, b, c)` serve a query that filters only on `b`?
3. In a composite index, why do equality columns go before range columns?
4. When is a single-column index on `(a)` redundant?
5. What's the downside of a prefix index, and how do you choose the prefix length?
6. How would you index a specific value *inside* a JSON object in MySQL (not an array)?
7. What's the Postgres equivalent of MySQL's multi-valued index for arrays?
8. When is BRIN a good choice and when is it useless?
9. Why is a bitmap index great for OLAP but bad for OLTP?
10. How do you enforce "unique email among non-deleted rows only"?
