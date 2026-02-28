# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Queries are slow because of hardware. Adding an index to a column makes every query on that column fast. More indexes means better performance. `Seq Scan` is always bad. The `cost` numbers in EXPLAIN are milliseconds."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | Index on a column means ANY query on that column is fast | Function wrapping (`LOWER(col)`), composite leftmost-prefix violations, and low selectivity all prevent index usage |
| 2 | More indexes = faster reads | Each index slows every INSERT/UPDATE/DELETE; unused indexes waste memory and disk |
| 3 | `Seq Scan` is always the wrong plan | For queries returning >20-30% of the table, Seq Scan's sequential I/O beats Index Scan's random I/O |
| 4 | EXPLAIN `cost` = milliseconds | Cost is in arbitrary planner units; only `EXPLAIN ANALYZE` shows real wall-clock time |
| 5 | Composite index `(A, B, C)` helps queries on any subset of those columns | Only the **leftmost prefix** can navigate the B-tree: `A`, `AB`, `ABC` — not `B`, `C`, `BC` |
| 6 | If estimated and actual rows differ, it's just a cosmetic issue | Bad estimates cause the planner to choose catastrophically wrong strategies (Nested Loop on 500K rows) |
| 7 | A `LIMIT 10` means PostgreSQL only processes 10 rows | LIMIT restricts **output**, not intermediate work. A missing index can force sorting all matching rows before the limit applies |

---

## The Main Purpose (Essence)

**Core Insight:** Query optimization is a **diagnostic discipline**, not a guessing game. You measure the problem (`EXPLAIN ANALYZE`), identify what PostgreSQL is doing and why (scan types, row estimates, join strategies), apply a targeted fix (specific index type, statistics update, query rewrite), and verify the fix actually worked. The workflow is always: **MEASURE → DIAGNOSE → FIX → VERIFY**.

**The Key Distinctions:**

| Concept | What Students Think | What It Actually Is |
|---------|--------------------|--------------------|
| Index | "Makes queries fast" | A separate, sorted data structure that trades write speed and storage for read speed on specific access patterns |
| `EXPLAIN` cost | "How slow the query is" | The planner's internal estimate of relative work, used to compare candidate plans — not a time measurement |
| `Seq Scan` | "A failure to optimize" | A deliberate planner choice when sequential disk I/O is cheaper than index-based random I/O |
| Composite Index | "An index on multiple columns" | A B-tree sorted by column A, then B within A, then C within B — order determines which queries it serves |
| `ANALYZE` | "Optional maintenance" | The only way to update the statistics the planner uses to make every decision; stale stats → wrong plans |

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "Index on `title`, query `LOWER(title)` uses index" | Index Scan | **Seq Scan** | B-tree stores raw values; `LOWER()` wrapping computes a new value the index never saw |
| "Composite `(A,B,C)`, query on `B` and `C` uses index" | Index Scan on B, C | **Seq Scan** | B-tree sorts by A first; without A, entries for B and C are scattered across all A-groups |
| "Index on `status`, 25% match → Index Scan" | Index Scan | **Seq Scan** or **Bitmap Scan** | 250K random page fetches via index is slower than one sequential pass of the whole table |
| "EXPLAIN cost = 5000 means 5 seconds" | Optimize the "5000 cost" query first | Cost has no unit relationship to time | Cost depends on `seq_page_cost`, `random_page_cost` parameters — not real hardware speed |
| "Same query, same index → same plan forever" | Index Scan (as always) | **Seq Scan** after data distribution shifts | Planner re-evaluates strategy based on statistics; bulk data changes shift the optimal plan |
| "`LIMIT 10` means only 10 rows processed" | Fast regardless of matching rows | **All** matching rows fetched + sorted, then 10 returned | Without an ORDER-matching index, PostgreSQL must see all candidates before selecting the top N |

---

# LEVEL 1: PREDICT & EXPLAIN

> **Prerequisite:** All exercises use the `demo_tasks` table from the lecture (1,000,000 rows). Before starting, ensure you have run the lecture's demo setup SQL, including `CREATE INDEX idx_demo_tasks_user_id ON demo_tasks(user_id);`. Drop all other indexes from lecture experiments:
> ```sql
> -- Clean slate (keep only PK and user_id index)
> DROP INDEX IF EXISTS idx_status_hash;
> DROP INDEX IF EXISTS idx_created_date;
> DROP INDEX IF EXISTS idx_active_tasks;
> DROP INDEX IF EXISTS idx_status_priority;
> DROP INDEX IF EXISTS idx_covering_status;
> DROP INDEX IF EXISTS idx_tags_gin;
> DROP INDEX IF EXISTS idx_metadata_gin;
> DROP INDEX IF EXISTS idx_search_gin;
> DROP INDEX IF EXISTS idx_title;
> DROP INDEX IF EXISTS idx_lower_title;
> ```

---

## Exercise 1.1: The Composite Riddle

**Setup:**

```sql
-- Remove the single-column user_id index
DROP INDEX IF EXISTS idx_demo_tasks_user_id;

-- Create a 3-column composite index
CREATE INDEX idx_composite ON demo_tasks(user_id, status, created_at);

ANALYZE demo_tasks;
```

You now have ONE index: a composite B-tree on `(user_id, status, created_at)`.

**Run each query through `EXPLAIN` (not `EXPLAIN ANALYZE`). Before running, predict for each:**
- Will PostgreSQL use `idx_composite`?
- Which columns appear as `Index Cond` vs `Filter`?
- Is there a `Sort` node?

**Query A:**
```sql
EXPLAIN
SELECT * FROM demo_tasks
WHERE user_id = 42
  AND status = 'pending'
  AND created_at > NOW() - INTERVAL '30 days';
```

**Query B:**
```sql
EXPLAIN
SELECT * FROM demo_tasks
WHERE status = 'pending'
  AND created_at > NOW() - INTERVAL '30 days';
```

**Query C:**
```sql
EXPLAIN
SELECT * FROM demo_tasks
WHERE user_id = 42
  AND created_at > NOW() - INTERVAL '30 days';
```

**Query D:**
```sql
EXPLAIN
SELECT id, title, priority, created_at
FROM demo_tasks
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 10;
```

**Questions:**

1. **Predict the scan type and Index Cond/Filter breakdown** for all four queries. Fill in a table:

   | Query | Scan Type? | Index Cond columns? | Filter columns? | Sort node? |
   |-------|-----------|-------------------|----------------|-----------|
   | A | ? | ? | ? | ? |
   | B | ? | ? | ? | ? |
   | C | ? | ? | ? | ? |
   | D | ? | ? | ? | ? |

2. **Query C** includes `user_id` and `created_at` — both are in the composite index. A junior developer says: *"Two out of three columns are in the WHERE clause, so the index should be used efficiently for both."* Explain why `created_at` cannot serve as an `Index Cond` in this query.

3. **Query D** has `ORDER BY created_at DESC` and a `LIMIT 10`. The index includes `created_at` as its third column. Does the index eliminate the `Sort` node? Why or why not? What single change to the index definition would eliminate the sort?

---

## Exercise 1.2: The Sort Trap

**Setup:**

```sql
-- Restore the single-column user_id index for this exercise
DROP INDEX IF EXISTS idx_composite;
CREATE INDEX idx_user_id ON demo_tasks(user_id);
ANALYZE demo_tasks;
```

**A developer writes this query for the "Recent Activity" page:**

```sql
EXPLAIN ANALYZE
SELECT id, title, status, created_at
FROM demo_tasks
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 10;
```

The developer sees the result returns 10 rows in about 2ms and says: *"Great, it's fast! The index on `user_id` found the 10 most recent tasks directly."*

**Questions:**

1. **Predict the full plan structure.** Specifically:
   - How many rows does the `Index Scan` on `user_id` return? (Hint: user 42 has ~1,000 tasks.)
   - Is there a `Sort` node between the Index Scan and the `Limit`?
   - How many rows does PostgreSQL actually sort?

2. **The developer claims "LIMIT 10 means PostgreSQL only looks at 10 rows."** Explain why this is wrong for this specific query plan. What does the `Sort` node need to see before it can emit the first row?

3. **Scale test:** If user 42 had 500,000 tasks instead of 1,000, what happens to the sort? Estimate the performance impact. (The sort spills to disk when it exceeds `work_mem`.)

4. **Design a single index** that allows PostgreSQL to answer this query with:
   - No `Sort` node
   - Only 10 rows read from the index (not 1,000)
   - Write the `CREATE INDEX` statement and predict the `EXPLAIN` output.

---

## Exercise 1.3: The Data Migration Surprise

**Context:** Your team ran a data migration last night. The migration inserted 900,000 new tasks with `status = 'archived'` (a new status value that didn't exist before). The team did **not** run `ANALYZE` after the migration. Autovacuum has not triggered yet.

**Before migration:** 1,000,000 rows. Status distribution: `pending` ~250K, `in_progress` ~250K, `completed` ~250K, `cancelled` ~250K. Statistics are fresh. The value `'archived'` has never appeared in the table.

**After migration:** 1,900,000 rows. 900,000 of them have `status = 'archived'`.

**Assume there is an index on `status`:**
```sql
CREATE INDEX idx_status ON demo_tasks(status);
```

**The planner's estimated plan (EXPLAIN, not ANALYZE):**

```
Index Scan using idx_status on demo_tasks
  (cost=0.42..8.44 rows=1 width=169)
  Index Cond: (status = 'archived'::text)
```

**Questions:**

1. The planner estimates **1 row**. The actual result will contain **900,000 rows**. Why is the estimate so catastrophically wrong? What specific PostgreSQL component holds the data that the planner is using, and why is it stale?

2. The planner chose `Index Scan` because it estimated 1 row. For 900,000 rows out of 1,900,000 (47% of the table), is `Index Scan` the optimal strategy? What scan type would be better, and why?

3. **Explain the specific performance impact.** `Index Scan` reads rows by following pointers from the index to the table. For 900,000 rows, this means 900,000 random page fetches. Compare this to what `Seq Scan` would do. Which involves more total I/O? Which involves more *expensive* I/O?

4. **What single command** fixes the planner's estimates? After running it, what scan type would the planner choose for this query?

---

## Exercise 1.4: The Function Barrier

**Setup:**
```sql
CREATE INDEX idx_title ON demo_tasks(title);
ANALYZE demo_tasks;
```

There is a B-tree index on the `title` column. Predict the scan type for each of these three queries:

**Query A:**
```sql
EXPLAIN SELECT * FROM demo_tasks WHERE title = 'Task #42';
```

**Query B:**
```sql
EXPLAIN SELECT * FROM demo_tasks WHERE LOWER(title) = 'task #42';
```

**Query C:**
```sql
EXPLAIN SELECT * FROM demo_tasks WHERE title = LOWER('TASK #42');
```

**Questions:**

1. **Predict the scan type** for all three queries. Which use the index? Which don't?

2. Queries B and C both involve `LOWER()`. A student says: *"They're equivalent — both compare a lowercased version of the title to 'task #42'."* Are they equivalent in terms of semantics? Are they equivalent in terms of **index usage**? Explain the critical difference.

3. **Mental trace for Query B:** Walk through what PostgreSQL does for *each row* in the table. At what point does it compute `LOWER(title)`? Why can't the B-tree index help with this computation?

4. **Mental trace for Query C:** What does PostgreSQL do with `LOWER('TASK #42')` *before* the scan begins? After that evaluation, what does the WHERE clause effectively become?

5. **Design an index** that makes Query B use an index scan. Write the `CREATE INDEX` statement. After creating it, what must the `WHERE` clause in future queries look like to use this index — `LOWER(title)`, `UPPER(title)`, or either?

---

# Level 1 Solutions

## Solution 1.1: The Composite Riddle

**1. Prediction Table (Correct Answers):**

| Query | Scan Type | Index Cond columns | Filter columns | Sort node? |
|-------|-----------|-------------------|----------------|-----------|
| A | Index Scan using idx_composite | `user_id`, `status`, `created_at` — all three | None | No |
| B | Seq Scan | None | `status`, `created_at` | No |
| C | Index Scan using idx_composite | `user_id` only | `created_at` | No |
| D | Index Scan using idx_composite + **Sort** | `user_id` only | None | **Yes** |

**Query A output:**
```
Index Scan using idx_composite on demo_tasks
  (cost=0.42..215.43 rows=84 width=169)
  Index Cond: ((user_id = 42) AND (status = 'pending'::text)
              AND (created_at > ...))
```

All three columns are in `Index Cond` because they follow the leftmost prefix exactly: `user_id` (1st) → `status` (2nd) → `created_at` (3rd). The B-tree can navigate directly to the narrow range of matching entries.

**Query B output:**
```
Seq Scan on demo_tasks
  (cost=0.00..20834.00 rows=... width=169)
  Filter: ((status = 'pending'::text) AND (created_at > ...))
```

The **leftmost column** (`user_id`) is missing from the WHERE clause. The composite B-tree is sorted by `user_id` first. Without `user_id`, `status = 'pending'` entries are scattered across every `user_id` group. PostgreSQL cannot navigate the tree, so it falls back to `Seq Scan`.

**Query C output:**
```
Index Scan using idx_composite on demo_tasks
  (cost=0.42..1025.43 rows=335 width=169)
  Index Cond: (user_id = 42)
  Filter: (created_at > ...)
  Rows Removed by Filter: ~668
```

This is the subtlest case. `user_id` IS the leftmost column, so the index IS used to navigate to all of user 42's entries. But `status` (the 2nd column) is skipped. Within user_id = 42, entries are sorted by `status` first, then `created_at` within each status group. Since `status` is unconstrained, `created_at` values are **not monotonically ordered** across status groups. PostgreSQL reads ALL ~1,000 entries for user_id = 42 and applies `created_at > ...` as a post-read `Filter`.

Key insight: `Index Cond` means "used for B-tree navigation." `Filter` means "applied after reading the row from the index." The index reduces the scan from 1,000,000 rows to ~1,000 rows (via `user_id`), but the `created_at` filter still discards ~668 of those 1,000.

**2. Why `created_at` cannot be an Index Cond in Query C:**

Think of the phonebook analogy. The composite index `(user_id, status, created_at)` is like a phonebook sorted by (LastName, FirstName, DateOfBirth).

Query A is: "Find all Smiths named John born after 1990." You flip to Smith, then within Smith to John, then within John scan from 1990 forward. All three levels of the sort order are used.

Query C is: "Find all Smiths born after 1990" (skipping FirstName). You flip to Smith, but now births-after-1990 are scattered: Adams-1995, Chen-1988, Chen-1992, Davis-2001... The DateOfBirth is not in sequence because FirstName (the middle sort key) was skipped. You must check every Smith.

**3. Query D — does the index eliminate Sort?**

No. The `ORDER BY created_at DESC` cannot use the composite index for ordering because `status` (the middle column) is unconstrained. Within `user_id = 42`, entries are ordered as:

```
(42, 'cancelled', 2025-01-15)
(42, 'cancelled', 2025-06-20)
(42, 'completed', 2025-02-01)
(42, 'completed', 2025-08-14)
(42, 'pending',   2025-03-10)
(42, 'pending',   2025-09-01)
```

The `created_at` values are NOT in global descending order — they're only sorted within each status group. PostgreSQL must collect all ~1,000 rows, then sort them by `created_at DESC`, then emit the top 10.

**To eliminate the Sort**, create an index where `created_at DESC` is the second column (immediately after `user_id`, with no gap):
```sql
CREATE INDEX idx_user_created ON demo_tasks(user_id, created_at DESC);
```
Now entries for `user_id = 42` are directly sorted by `created_at DESC`. PostgreSQL reads the first 10 entries and stops — no Sort node needed.

---

## Solution 1.2: The Sort Trap

**1. Full plan structure:**

```
Limit  (rows=10)
  ->  Sort  (actual time=1.102..1.105 rows=10)
        Sort Key: created_at DESC
        Sort Method: top-N heapsort  Memory: 27kB
        ->  Index Scan using idx_user_id on demo_tasks
              (actual time=0.031..0.892 rows=1003 loops=1)
              Index Cond: (user_id = 42)
Planning Time: 0.095 ms
Execution Time: 1.234 ms
```

The Index Scan returns **all ~1,003 rows** for user 42. The Sort node receives all 1,003 rows, sorts them by `created_at DESC`, then the Limit node takes the top 10. PostgreSQL uses "top-N heapsort" — an optimization that only tracks the top N items during the sort — but it still **reads and evaluates all 1,003 rows**.

**2. Why "LIMIT 10 means PostgreSQL only looks at 10 rows" is wrong:**

The `LIMIT` node sits on TOP of the plan tree. It limits the output of whatever node feeds into it. But the Sort node below it needs to see **all input rows** before it knows which 10 have the highest `created_at`. You can't know the top 10 of an unsorted dataset without examining every item.

The Sort node must receive all 1,003 rows, identify the 10 with the largest `created_at`, and only then emit them. The Limit merely tells Sort to stop after emitting 10.

**3. Scale impact with 500,000 tasks:**

With 1,003 rows, the sort runs in-memory (27kB). With 500,000 rows, the sort would consume roughly `500,000 × 169 bytes ≈ 80MB`. If `work_mem` is the default 4MB, the sort **spills to disk** — dramatically slower. What took 1ms now takes hundreds of milliseconds or more, because temporary files are written and re-read. The `Sort Method` would change from `quicksort Memory: 27kB` to `external merge Disk: 80MB`.

**4. Index to eliminate the Sort:**

```sql
CREATE INDEX idx_user_created_desc ON demo_tasks(user_id, created_at DESC);
```

Predicted EXPLAIN:
```
Limit  (rows=10)
  ->  Index Scan using idx_user_created_desc on demo_tasks
        (cost=0.42..12.87 rows=10 width=169)
        Index Cond: (user_id = 42)
```

No Sort node. PostgreSQL navigates to `user_id = 42` in the B-tree, then reads entries in `created_at DESC` order (the index is pre-sorted this way). After 10 entries, it stops. Total rows read from the index: **10** (not 1,003). This is the difference between O(N log N) and O(1) for the sort operation.

---

## Solution 1.3: The Data Migration Surprise

**1. Why the estimate is catastrophically wrong:**

PostgreSQL's query planner reads table statistics stored in `pg_statistic` (accessed via `pg_stats` view). These statistics are updated only when `ANALYZE` runs (either manually or via autovacuum). The statistics include a `most_common_vals` list and a `histogram_bounds` array for each column.

Before the migration, `'archived'` never appeared in the data, so it's absent from `most_common_vals` for the `status` column. When the planner encounters a value not in its statistics, it falls back to a generic selectivity estimate — often assuming very few rows match (approximately `1 / n_distinct` or a minimum default). Hence the estimate of 1 row.

After inserting 900,000 `'archived'` rows without running `ANALYZE`, the statistics are completely stale. The planner has no idea 47% of the table now matches this condition.

**2. Is Index Scan optimal for 900K out of 1.9M rows?**

No. At 47% selectivity, `Seq Scan` is dramatically better. Index Scan performs a random page fetch for each matching row. With 900,000 rows, that's up to 900,000 random I/O operations. Sequential Scan reads the entire table in one linear pass — far fewer I/O operations because sequential disk reads are 10-100x faster than random reads.

The optimal plan for 47% selectivity would be `Seq Scan` with a `Filter`.

**3. Specific performance impact:**

| Strategy | I/O Pattern | Approximate I/O Operations | Why |
|----------|-------------|---------------------------|-----|
| Index Scan (chosen) | **Random** page fetches | Up to 900,000 random reads | Each index entry points to a different page location |
| Seq Scan (optimal) | **Sequential** page reads | ~24,000 sequential reads | Read all ~24K pages of the table in order |

Random I/O on spinning disk is ~100x slower than sequential. Even on SSD, random reads are 5-10x slower due to command overhead. The Index Scan might take 10-30 seconds; the Seq Scan would take under 1 second.

**4. Fix:**

```sql
ANALYZE demo_tasks;
```

After running this, the planner updates its statistics. It now knows `'archived'` accounts for 47% of rows. It chooses `Seq Scan`:

```
Seq Scan on demo_tasks
  (cost=0.00..39584.00 rows=900000 width=169)
  Filter: (status = 'archived'::text)
```

---

## Solution 1.4: The Function Barrier

**1. Scan types:**

| Query | Scan Type | Uses `idx_title`? |
|-------|-----------|-------------------|
| A: `WHERE title = 'Task #42'` | **Index Scan** | Yes |
| B: `WHERE LOWER(title) = 'task #42'` | **Seq Scan** | No |
| C: `WHERE title = LOWER('TASK #42')` | **Index Scan** | Yes |

**2. B and C are semantically similar but NOT equivalent for index usage.**

- **Query B** applies `LOWER()` to the **column** (`LOWER(title)`). For every row, PostgreSQL must compute `LOWER(title)` and compare the result. The index stores raw `title` values, not lowercased versions. PostgreSQL cannot look up `LOWER(title)` in the B-tree because the B-tree has never seen those computed values.

- **Query C** applies `LOWER()` to a **constant** (`LOWER('TASK #42')`). PostgreSQL evaluates `LOWER('TASK #42')` → `'task #42'` once during query planning. The WHERE clause becomes `WHERE title = 'task #42'` — a simple equality on the raw column. The index stores raw values, and `'task #42'` is a raw value to look up.

The critical rule: **functions on columns block index usage. Functions on values are evaluated to constants at plan time.**

**3. Mental trace for Query B:**

```
Row 1:   Read title='Task #1'       → compute LOWER('Task #1') → 'task #1'    → 'task #1' = 'task #42'? No.
Row 2:   Read title='Task #2'       → compute LOWER('Task #2') → 'task #2'    → 'task #2' = 'task #42'? No.
...
Row 42:  Read title='Task #42'      → compute LOWER('Task #42') → 'task #42'  → 'task #42' = 'task #42'? Yes!
...
Row 1M:  Read title='Task #999999'  → compute LOWER(...)        → ...         → No.
```

PostgreSQL must visit every single row, compute `LOWER()`, then compare. The B-tree index has entries for `'Task #1'`, `'Task #42'`, etc. — but it has no entry for `'task #42'` (lowercased). The index is useless.

**4. Mental trace for Query C:**

```
Planning phase: evaluate LOWER('TASK #42') → 'task #42' (constant folding)
WHERE clause becomes: title = 'task #42'

Execution: look up 'task #42' in idx_title B-tree
  → Navigate: root → internal node → leaf
  → Found? If a row has title = 'task #42' (exact case), yes.
  → If all titles are 'Task #42' (capital T), then 0 results — but the INDEX was used.
```

Note: Query C finds rows where `title` is literally `'task #42'` (all lowercase). If all titles are `'Task #42'`, it returns nothing. This is semantically different from Query B, which finds rows where the **lowercased** title equals `'task #42'` (which matches `'Task #42'`, `'TASK #42'`, etc.).

**5. Expression index for Query B:**

```sql
CREATE INDEX idx_lower_title ON demo_tasks(LOWER(title));
```

The expression in the query's WHERE clause must **exactly match** the expression in the index:
- `WHERE LOWER(title) = 'task #42'` → ✅ uses `idx_lower_title`
- `WHERE UPPER(title) = 'TASK #42'` → ❌ different function, cannot use it
- `WHERE title = 'task #42'` → ❌ no function on column, uses `idx_title` instead

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Dashboard That Grinds

**Context:** A developer built an endpoint for the Task Manager's admin dashboard that returns the 20 most recent active tasks for a given user, along with category names. It works, but takes 400ms in production with 1,000,000 rows.

**The developer's SQLAlchemy query (simplified to raw SQL for diagnosis):**

```sql
-- This is the SQL that SQLAlchemy generates (from echo=True)
SELECT t.id, t.title, t.status, t.priority, t.created_at, c.name AS category
FROM demo_tasks t
JOIN demo_categories c ON t.category_id = c.id
WHERE t.user_id = 42
  AND t.status IN ('pending', 'in_progress')
ORDER BY t.created_at DESC
LIMIT 20;
```

**The developer's current indexes** (besides the primary key):

```sql
CREATE INDEX idx_status ON demo_tasks(status);
CREATE INDEX idx_created ON demo_tasks(created_at);
```

**Setup:** Run this to reproduce the state:

```sql
-- Clean slate
DROP INDEX IF EXISTS idx_composite;
DROP INDEX IF EXISTS idx_user_id;
DROP INDEX IF EXISTS idx_title;
DROP INDEX IF EXISTS idx_lower_title;
DROP INDEX IF EXISTS idx_status;
DROP INDEX IF EXISTS idx_created;

-- Ensure categories table exists (from lecture)
-- (skip if already created during lecture demo)

-- Developer's indexes
CREATE INDEX idx_status ON demo_tasks(status);
CREATE INDEX idx_created ON demo_tasks(created_at);
ANALYZE demo_tasks;
```

**Run the query with EXPLAIN ANALYZE:**

```sql
EXPLAIN ANALYZE
SELECT t.id, t.title, t.status, t.priority, t.created_at, c.name AS category
FROM demo_tasks t
JOIN demo_categories c ON t.category_id = c.id
WHERE t.user_id = 42
  AND t.status IN ('pending', 'in_progress')
ORDER BY t.created_at DESC
LIMIT 20;
```

---

**Task 2a: Identify the Flaws**

Examine the EXPLAIN ANALYZE output. You should find **three** performance problems:

1. What scan type is used on `demo_tasks`? Why doesn't either existing index help with this specific query?
2. Is there a `Sort` node? How many rows is it sorting?
3. Look at the join strategy and its `loops` count. Is this efficient?

Do NOT fix anything yet. Just identify and name each problem.

---

**Task 2b: Explain the Mechanism**

For each flaw you identified:

1. Why did the planner choose `Seq Scan` (or `Bitmap Scan`) on `demo_tasks` instead of a more targeted index scan? Consider what columns the WHERE clause actually filters on.
2. The `idx_status` index exists and `status IN ('pending', 'in_progress')` is in the WHERE clause. Why doesn't the planner use it? (Hint: what percentage of the table matches that condition?)
3. Explain why `idx_created` doesn't help even though `ORDER BY created_at DESC` is in the query.

---

**Task 2c: Instrument to Prove Diagnosis**

Add `EXPLAIN (ANALYZE, BUFFERS)` to get buffer statistics. Run:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT t.id, t.title, t.status, t.priority, t.created_at, c.name AS category
FROM demo_tasks t
JOIN demo_categories c ON t.category_id = c.id
WHERE t.user_id = 42
  AND t.status IN ('pending', 'in_progress')
ORDER BY t.created_at DESC
LIMIT 20;
```

Answer these from the output:
1. How many `shared hit` + `shared read` buffers does the `Seq Scan` (or `Bitmap Scan`) touch? This represents how many 8KB pages PostgreSQL read.
2. What is the total `Execution Time`?
3. How many rows does `Rows Removed by Filter` show? This is the waste.

---

**Task 2d: Fix the Code**

Design a **single index** that addresses all three problems simultaneously:
- Eliminates the Seq Scan on `demo_tasks` (targets `user_id` and `status`)
- Eliminates the Sort node (provides `created_at DESC` ordering)
- Minimizes the number of rows PostgreSQL touches

Write the `CREATE INDEX` statement. Then run `EXPLAIN ANALYZE` again on the same query.

---

**Task 2e: Write Verification**

After applying your fix, verify by running:

```sql
-- Verification 1: Plan check
EXPLAIN
SELECT t.id, t.title, t.status, t.priority, t.created_at, c.name AS category
FROM demo_tasks t
JOIN demo_categories c ON t.category_id = c.id
WHERE t.user_id = 42
  AND t.status IN ('pending', 'in_progress')
ORDER BY t.created_at DESC
LIMIT 20;
```

Confirm:
- [ ] Output shows `Index Scan` (not `Seq Scan`)
- [ ] NO `Sort` node in the plan
- [ ] `Limit` feeds directly from the join (or Index Scan)
- [ ] `Nested Loop` join with low `loops` count on categories

```sql
-- Verification 2: Performance check
EXPLAIN ANALYZE
/* same query */;
```

Confirm:
- [ ] `Execution Time` is under **5ms** (was ~400ms before)
- [ ] `Rows Removed by Filter` is 0 or very small

---

## Exercise 2.2: The Partial Index Mismatch

**Context:** A developer heard about partial indexes from the lecture and excitedly added one for "active tasks" — the most common query pattern in the Task Manager. But the performance improvement is smaller than expected for some queries and nonexistent for others.

**Setup:**

```sql
-- Clean indexes
DROP INDEX IF EXISTS idx_status;
DROP INDEX IF EXISTS idx_created;

-- The developer's "clever" partial index
CREATE INDEX idx_active_by_user ON demo_tasks(user_id, created_at DESC)
WHERE status = 'pending' OR status = 'in_progress';

-- Also add a regular user_id index for comparison
CREATE INDEX idx_user_id ON demo_tasks(user_id);

ANALYZE demo_tasks;
```

**The developer tests with three queries:**

```sql
-- Query A: Active tasks for user 42 (the intended use case)
EXPLAIN ANALYZE
SELECT * FROM demo_tasks
WHERE user_id = 42
  AND status IN ('pending', 'in_progress')
ORDER BY created_at DESC
LIMIT 20;

-- Query B: All tasks for user 42 (not just active)
EXPLAIN ANALYZE
SELECT * FROM demo_tasks
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;

-- Query C: Active tasks for user 42, different syntax
EXPLAIN ANALYZE
SELECT * FROM demo_tasks
WHERE user_id = 42
  AND (status = 'pending' OR status = 'in_progress')
ORDER BY created_at DESC
LIMIT 20;
```

The developer says: *"Query A uses the partial index perfectly — fast! But Query B is slow. I expected the partial index to help there too since most of user 42's tasks are active anyway. And Query C is weird — sometimes it uses the partial index, sometimes it doesn't."*

---

**Task 2a: Identify the Flaws**

Run all three queries with `EXPLAIN ANALYZE`.

1. Which index does each query use? Does `idx_active_by_user` appear in all three plans?
2. Why can't Query B use the partial index?
3. Query C's WHERE clause is **semantically identical** to Query A's (`IN` vs `OR`). Does PostgreSQL recognize them as equivalent for partial index matching?

---

**Task 2b: Explain the Mechanism**

1. What is the **exact rule** for when PostgreSQL can use a partial index? It's not just "the query filters on the same columns." What must be true about the query's WHERE clause relative to the index's WHERE clause?
2. The developer expected the partial index to help Query B "since most of user 42's tasks are active anyway." Why is this reasoning flawed? What guarantee does the partial index make to the planner?
3. For Query B, which index does PostgreSQL actually use? Why does `ORDER BY created_at DESC` work with `idx_user_id` but produce a `Sort` node?

---

**Task 2c: Instrument to Prove Diagnosis**

Run these to quantify the difference:

```sql
-- Compare plan for Query A vs Query B
EXPLAIN (ANALYZE, BUFFERS) /* Query A */;
EXPLAIN (ANALYZE, BUFFERS) /* Query B */;
```

Record:
| Metric | Query A (active only) | Query B (all tasks) |
|--------|------|------|
| Index used | ? | ? |
| Sort node? | ? | ? |
| Execution Time | ? | ? |
| Rows processed | ? | ? |

---

**Task 2d: Fix the Code**

The developer wants **both** queries (A and B) to be fast — under 5ms each, without a Sort node. Currently they have two indexes. Redesign the index strategy:

1. Can you make both queries fast with the **same** two indexes? If so, which two? If not, why?
2. Write the `CREATE INDEX` statements for your solution.

---

**Task 2e: Write Verification**

```sql
-- Test: Query A uses partial index, no Sort, under 5ms
EXPLAIN ANALYZE
SELECT * FROM demo_tasks
WHERE user_id = 42 AND status IN ('pending', 'in_progress')
ORDER BY created_at DESC LIMIT 20;
-- ✅ Must show idx_active_by_user, no Sort, Execution Time < 5ms

-- Test: Query B uses an index, no Sort, under 5ms
EXPLAIN ANALYZE
SELECT * FROM demo_tasks
WHERE user_id = 42
ORDER BY created_at DESC LIMIT 20;
-- ✅ Must show Index Scan, no Sort, Execution Time < 5ms

-- Test: Total index count (excluding PK) is at most 2
SELECT indexname FROM pg_indexes
WHERE tablename = 'demo_tasks' AND indexname != 'demo_tasks_pkey';
-- ✅ Must return at most 2 rows
```

---

# Level 2 Solutions

## Solution 2.1: The Dashboard That Grinds

**Task 2a: Three flaws:**

1. **Seq Scan on `demo_tasks`.** Neither `idx_status` nor `idx_created` targets `user_id`, which is the most selective filter in this query (0.1% of rows). `idx_status` covers `status IN ('pending', 'in_progress')` — roughly 50% of rows — far too many for an efficient index scan. `idx_created` is on `created_at` which isn't in the WHERE clause at all.

2. **Sort node.** PostgreSQL sorts all matching rows (~500) by `created_at DESC` before applying `LIMIT 20`. The `idx_created` index sorts by `created_at`, but it covers the *entire* table — not just user 42's active tasks. The planner can't use it to avoid sorting a filtered subset.

3. **Nested Loop join with high loops.** The join to `demo_categories` runs once per task row. With ~500 matching tasks, the categories primary key index is probed 500 times. While each probe is fast (<0.01ms), it adds up.

**Task 2b: Mechanisms:**

1. The WHERE clause filters on `user_id = 42` AND `status IN (...)`. Neither existing index covers `user_id`. The planner's best option is Seq Scan (read everything, filter on both conditions) or Bitmap Scan on `idx_status` (but 50% selectivity makes this barely worthwhile).

2. `idx_status` matches the `status` condition but ~500K rows match `status IN ('pending', 'in_progress')`. At 50% selectivity, the planner correctly determines that reading half the table through an index (random I/O) is worse than reading the whole table sequentially.

3. `idx_created` is ordered by `created_at`, which matches `ORDER BY created_at DESC`. But using this index would mean scanning `created_at` entries in reverse order and checking `user_id = 42 AND status IN (...)` for each — essentially scanning the entire index backwards until 20 matches are found. With user 42 being 0.1% of tasks, PostgreSQL would need to check ~20,000 entries to find 20 matches. This is less efficient than Seq Scan + Sort for the planner's cost model.

**Task 2d: The Fix:**

```sql
CREATE INDEX idx_user_active_recent ON demo_tasks(user_id, status, created_at DESC);
```

Why this works:
- `user_id = 42` → navigates to user 42 (leftmost prefix)
- `status IN ('pending', 'in_progress')` → within user 42, navigates to each status group
- `created_at DESC` → within each (user_id, status) group, entries are already in the right order

However, because `status IN (...)` covers two values, the entries across the two status groups are not globally sorted by `created_at DESC`. PostgreSQL may still need a small sort to merge them.

For guaranteed Sort elimination, a **partial composite index** is superior:

```sql
DROP INDEX idx_user_active_recent;

CREATE INDEX idx_dashboard ON demo_tasks(user_id, created_at DESC)
WHERE status IN ('pending', 'in_progress');
```

This partial index only includes "active" rows. Within each `user_id`, entries are sorted directly by `created_at DESC` with no intermediate status grouping. The Sort node is eliminated entirely.

**After fix — Expected EXPLAIN ANALYZE:**
```
Limit (actual rows=20)
  -> Nested Loop (actual rows=20)
     -> Index Scan using idx_dashboard on demo_tasks t
          Index Cond: (user_id = 42)
          actual rows=20   ← only 20 rows read!
     -> Index Scan using demo_categories_pkey on demo_categories c
          Index Cond: (id = t.category_id)
          loops=20         ← only 20 lookups, not 500
Planning Time: ~0.2 ms
Execution Time: ~0.5 ms
```

**Before:** ~400ms, 500+ rows processed, Sort node.
**After:** ~0.5ms, 20 rows processed, no Sort. **~800x improvement.**

---

## Solution 2.2: The Partial Index Mismatch

**Task 2a: Which index each query uses:**

| Query | Index Used | Sort Node? |
|-------|-----------|-----------|
| A: `WHERE user_id = 42 AND status IN ('pending','in_progress')` | `idx_active_by_user` ✅ | No ✅ |
| B: `WHERE user_id = 42` | `idx_user_id` (regular index) | **Yes** ❌ |
| C: `WHERE user_id = 42 AND (status = 'pending' OR status = 'in_progress')` | `idx_active_by_user` ✅ (PostgreSQL recognizes this is equivalent to `IN`) | No ✅ |

Query B **cannot** use the partial index because its WHERE clause doesn't imply the partial index's condition. The query asks for ALL tasks for user 42, including `'completed'` and `'cancelled'` — rows that are not in the partial index.

**Task 2b: Mechanisms:**

1. **Exact rule:** PostgreSQL can use a partial index only when the query's WHERE clause **logically implies** the index's WHERE predicate. In other words, the planner must be able to prove that every row the query could possibly return is guaranteed to exist in the index. For Query A, `status IN ('pending', 'in_progress')` exactly matches the index condition. For Query B, `user_id = 42` says nothing about status — user 42 might have `'completed'` tasks, which the partial index doesn't contain.

2. **Why "most tasks are active" reasoning is flawed:** The planner doesn't know or care about the actual data distribution for this decision. The partial index makes a structural guarantee: "I contain only rows where status is pending or in_progress." The planner asks: "Can this query return a row with status = 'completed'?" For Query B, the answer is yes — so the partial index is disqualified. Even if 99% of user 42's tasks are active, that 1% of completed/cancelled tasks would be missed.

3. For Query B, PostgreSQL uses `idx_user_id` to find all of user 42's tasks (~1,000 rows), then sorts them by `created_at DESC`. This requires a Sort node because `idx_user_id` is a single-column index on `user_id` — it doesn't include `created_at` and thus provides no ordering guarantee.

**Task 2d: Fix:**

Keep both indexes. The partial index handles Query A (active tasks, sorted, no Sort). For Query B (all tasks, sorted), replace `idx_user_id` with a composite that includes `created_at DESC`:

```sql
DROP INDEX idx_user_id;
CREATE INDEX idx_user_all ON demo_tasks(user_id, created_at DESC);
```

Final index set:
| Index | Purpose |
|-------|---------|
| `idx_active_by_user` | Query A: active tasks per user, sorted by recency |
| `idx_user_all` | Query B: all tasks per user, sorted by recency |

Query A uses the partial index (smaller, faster, covers only active rows). Query B uses the full composite index (covers all statuses). Both queries: no Sort node, under 5ms, LIMIT reads only 20 rows from the index.

There is an acceptable redundancy: `idx_user_all` could technically serve Query A too (with a Filter on status). But the partial index is smaller and faster for the active-only case. With only 2 indexes, the write overhead is minimal.

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Exercise 2.5: Extending the Query Profiler

**Background:** In Part 5 of the lecture, you learned about SQLAlchemy event-based query timing. Below is a working, tested `QueryProfiler` class. Your task is to extend it with three new capabilities — without breaking the existing test suite.

**Given Code:**

```python
# query_profiler.py
import time
import logging
from typing import Any
from sqlalchemy import event
from sqlalchemy.engine import Engine, Connection

logger = logging.getLogger("query_profiler")


class QueryProfiler:
    """Tracks query execution times and detects performance issues."""

    def __init__(self, slow_threshold_ms: float = 100.0) -> None:
        self._slow_threshold_ms = slow_threshold_ms
        self._query_log: list[dict[str, Any]] = []

    def attach(self, engine: Engine) -> None:
        """Attach profiler to a SQLAlchemy engine."""
        event.listen(engine, "before_cursor_execute", self._before_execute)
        event.listen(engine, "after_cursor_execute", self._after_execute)

    def _before_execute(
        self, conn: Connection, cursor: Any, statement: str,
        parameters: Any, context: Any, executemany: bool,
    ) -> None:
        conn.info.setdefault("query_start_time", []).append(time.perf_counter())

    def _after_execute(
        self, conn: Connection, cursor: Any, statement: str,
        parameters: Any, context: Any, executemany: bool,
    ) -> None:
        start = conn.info["query_start_time"].pop(-1)
        duration_ms = (time.perf_counter() - start) * 1000

        entry: dict[str, Any] = {
            "statement": statement[:500],
            "duration_ms": round(duration_ms, 3),
            "timestamp": time.time(),
            "is_slow": duration_ms > self._slow_threshold_ms,
        }
        self._query_log.append(entry)

        if entry["is_slow"]:
            logger.warning(
                "SLOW QUERY (%.2fms): %s", duration_ms, statement[:200],
            )

    def get_log(self) -> list[dict[str, Any]]:
        """Return a copy of all logged queries."""
        return list(self._query_log)

    def get_slow_queries(self) -> list[dict[str, Any]]:
        """Return only queries that exceeded the slow threshold."""
        return [q for q in self._query_log if q["is_slow"]]

    def clear(self) -> None:
        """Clear the query log."""
        self._query_log.clear()
```

**Existing Test Suite (must continue to pass):**

```python
# test_profiler.py
import pytest
from sqlalchemy import create_engine, text
from query_profiler import QueryProfiler


@pytest.fixture
def engine():
    eng = create_engine("postgresql://user:pass@localhost/testdb")
    yield eng
    eng.dispose()


@pytest.fixture
def profiler(engine):
    p = QueryProfiler(slow_threshold_ms=50.0)
    p.attach(engine)
    yield p
    p.clear()


def test_logs_queries(profiler, engine):
    with engine.connect() as conn:
        conn.execute(text("SELECT 1"))
        conn.execute(text("SELECT 2"))
    assert len(profiler.get_log()) >= 2


def test_measures_duration(profiler, engine):
    with engine.connect() as conn:
        conn.execute(text("SELECT 1"))
    entry = profiler.get_log()[-1]
    assert "duration_ms" in entry
    assert entry["duration_ms"] >= 0


def test_detects_slow_query(profiler, engine):
    with engine.connect() as conn:
        conn.execute(text("SELECT pg_sleep(0.1)"))  # 100ms > 50ms threshold
    slow = profiler.get_slow_queries()
    assert len(slow) >= 1
    assert slow[0]["is_slow"] is True


def test_clear_resets_log(profiler, engine):
    with engine.connect() as conn:
        conn.execute(text("SELECT 1"))
    assert len(profiler.get_log()) > 0
    profiler.clear()
    assert len(profiler.get_log()) == 0
```

---

### Extension 1: Top-N Slowest Queries Report

**Task:** Add a method `get_top_slow(n: int) -> list[dict[str, Any]]` that returns the N slowest queries from the log, sorted by duration descending.

**Constraint:** Must not modify any existing methods. Existing tests must pass.

**New tests that must also pass:**

```python
def test_top_slow_returns_sorted(profiler, engine):
    with engine.connect() as conn:
        conn.execute(text("SELECT pg_sleep(0.01)"))   # ~10ms
        conn.execute(text("SELECT pg_sleep(0.05)"))   # ~50ms
        conn.execute(text("SELECT pg_sleep(0.02)"))   # ~20ms

    top = profiler.get_top_slow(2)
    assert len(top) == 2
    assert top[0]["duration_ms"] >= top[1]["duration_ms"]
    # The slowest (~50ms) should be first
    assert top[0]["duration_ms"] > 30


def test_top_slow_with_fewer_queries_than_n(profiler, engine):
    with engine.connect() as conn:
        conn.execute(text("SELECT 1"))

    top = profiler.get_top_slow(10)
    assert len(top) == 1  # Only 1 query logged, so return 1
```

---

### Extension 2: Request-Scoped Query Counter

**Task:** Add a context manager method `request_scope(endpoint: str, max_queries: int = 10)` that tracks how many queries fire within the block. If the count exceeds `max_queries`, log a warning about potential N+1.

**Expected usage:**

```python
with profiler.request_scope(endpoint="/api/tasks", max_queries=5) as scope:
    with engine.connect() as conn:
        for i in range(20):
            conn.execute(text(f"SELECT {i}"))

print(scope.query_count)  # 20
print(scope.total_ms)     # sum of all 20 query durations
# WARNING logged: "N+1 SUSPECT: /api/tasks executed 20 queries (max 5)"
```

**New tests that must pass:**

```python
def test_request_scope_counts_queries(profiler, engine):
    with profiler.request_scope(endpoint="/test", max_queries=50) as scope:
        with engine.connect() as conn:
            conn.execute(text("SELECT 1"))
            conn.execute(text("SELECT 2"))
            conn.execute(text("SELECT 3"))
    assert scope.query_count == 3
    assert scope.total_ms >= 0


def test_request_scope_warns_on_excess(profiler, engine, caplog):
    import logging
    with caplog.at_level(logging.WARNING, logger="query_profiler"):
        with profiler.request_scope(endpoint="/slow", max_queries=2) as scope:
            with engine.connect() as conn:
                conn.execute(text("SELECT 1"))
                conn.execute(text("SELECT 2"))
                conn.execute(text("SELECT 3"))
    assert scope.query_count == 3
    assert "N+1" in caplog.text or "/slow" in caplog.text
```

**Constraint:** The `request_scope` context manager must integrate with the existing `_after_execute` hook. Do not duplicate the timing logic.

---

### Extension 3: Duplicate Statement Detection

**Task:** Enhance `request_scope` so that `scope` exposes a `get_duplicates() -> dict[str, int]` method that returns a mapping from SQL statement text to the number of times it was executed within the scope. Only include statements executed more than once.

This detects the N+1 pattern: the same `SELECT ... WHERE id = $1` fires repeatedly with different parameters.

**Important:** Two executions of `SELECT * FROM tasks WHERE id = $1` with different parameter values (`1` and `2`) should be treated as the **same statement** (the parameterized SQL is identical; only the bound values differ).

**New test that must pass:**

```python
def test_detects_duplicate_statements(profiler, engine):
    with profiler.request_scope(endpoint="/tasks") as scope:
        with engine.connect() as conn:
            # Simulate N+1: same statement, different params
            for i in range(5):
                conn.execute(
                    text("SELECT * FROM demo_tasks WHERE id = :id"),
                    {"id": i},
                )
            # One unique query
            conn.execute(text("SELECT COUNT(*) FROM demo_tasks"))

    dupes = scope.get_duplicates()
    # The parameterized SELECT ran 5 times
    assert any(count >= 5 for count in dupes.values())
    # The COUNT query ran once — should NOT appear in duplicates
    assert all("COUNT" not in stmt for stmt in dupes.keys())
```

---

# Level 2.5 Solutions

## Extension 1 Solution: Top-N Slowest

```python
def get_top_slow(self, n: int) -> list[dict[str, Any]]:
    """Return the N slowest queries, sorted by duration descending."""
    sorted_log = sorted(self._query_log, key=lambda q: q["duration_ms"], reverse=True)
    return sorted_log[:n]
```

**Why:** This is straightforward — sort a copy of the log by `duration_ms` descending and slice. The key design choice is sorting a copy (`sorted()` returns a new list), not mutating `_query_log`, which would break the chronological order that other methods depend on.

---

## Extension 2 Solution: Request-Scoped Counter

```python
from contextlib import contextmanager

# Add this helper class
class RequestScope:
    def __init__(self, endpoint: str, max_queries: int) -> None:
        self.endpoint = endpoint
        self.max_queries = max_queries
        self.query_count: int = 0
        self.total_ms: float = 0.0
        self._statements: list[str] = []

    def record(self, statement: str, duration_ms: float) -> None:
        self.query_count += 1
        self.total_ms += duration_ms
        self._statements.append(statement[:500])


# Add this method to QueryProfiler
@contextmanager
def request_scope(self, endpoint: str, max_queries: int = 10):
    scope = RequestScope(endpoint=endpoint, max_queries=max_queries)
    self._active_scope = scope  # Store reference for _after_execute to see
    try:
        yield scope
    finally:
        self._active_scope = None
        if scope.query_count > scope.max_queries:
            logger.warning(
                "N+1 SUSPECT: %s executed %d queries (max %d), %.1fms total",
                endpoint, scope.query_count, scope.max_queries, scope.total_ms,
            )
```

Then modify `_after_execute` to also record into the active scope:

```python
def _after_execute(self, conn, cursor, statement, parameters, context, executemany):
    start = conn.info["query_start_time"].pop(-1)
    duration_ms = (time.perf_counter() - start) * 1000

    entry = {
        "statement": statement[:500],
        "duration_ms": round(duration_ms, 3),
        "timestamp": time.time(),
        "is_slow": duration_ms > self._slow_threshold_ms,
    }
    self._query_log.append(entry)

    # NEW: record into active scope if one exists
    active = getattr(self, "_active_scope", None)
    if active is not None:
        active.record(statement, duration_ms)

    if entry["is_slow"]:
        logger.warning("SLOW QUERY (%.2fms): %s", duration_ms, statement[:200])
```

**Why this design:** The scope object is a lightweight container. The profiler holds a reference to the "active" scope during the `with` block. The existing `_after_execute` hook gains a single `if` check — minimal impact on the hot path. The warning fires in the `finally` block (after all queries complete), so the count is final.

Initialize `_active_scope = None` in `__init__` to avoid `AttributeError` if `request_scope` is never used:
```python
def __init__(self, slow_threshold_ms: float = 100.0) -> None:
    self._slow_threshold_ms = slow_threshold_ms
    self._query_log: list[dict[str, Any]] = []
    self._active_scope: RequestScope | None = None
```

---

## Extension 3 Solution: Duplicate Detection

Add `get_duplicates` to `RequestScope`:

```python
from collections import Counter

class RequestScope:
    # ... existing __init__ and record ...

    def get_duplicates(self) -> dict[str, int]:
        """Return statements executed more than once, with their counts."""
        counts = Counter(self._statements)
        return {stmt: count for stmt, count in counts.items() if count > 1}
```

**Why this works for N+1 detection:** SQLAlchemy's parameterized queries use placeholders (`$1`, `:id`). The `statement` string passed to the event listener contains the parameterized SQL, not the interpolated values. So `SELECT * FROM tasks WHERE id = :id` appears identically for every invocation regardless of the actual `id` value. The `Counter` groups them correctly.

**Why this is the right abstraction:** The developer doesn't need to know the parameter values — they need to know that the same query template fired 50 times. That pattern screams "N+1: you should use `joinedload` or `selectinload` instead of lazy loading."

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Exercise 3: The Dashboard Index Budget

**Scenario:**

Your Task Manager's dashboard page fires **three queries** on every load. With 1,000,000 rows in `demo_tasks`, each query currently takes 200-500ms (all `Seq Scan`). The product team requires every query to complete in under **5ms**. Your DBA adds a constraint: **maximum 3 custom indexes** (excluding the primary key) because the table receives 10,000 INSERTs per hour and write performance is critical.

**Setup:**

```sql
-- Clean all indexes except PK
DO $$
DECLARE idx RECORD;
BEGIN
    FOR idx IN
        SELECT indexname FROM pg_indexes
        WHERE tablename = 'demo_tasks' AND indexname != 'demo_tasks_pkey'
    LOOP
        EXECUTE format('DROP INDEX IF EXISTS %I', idx.indexname);
    END LOOP;
END $$;

ANALYZE demo_tasks;
```

**The three dashboard queries:**

```sql
-- QUERY 1: "Active tasks for current user, most urgent first"
SELECT id, title, status, priority, created_at
FROM demo_tasks
WHERE user_id = 42
  AND status IN ('pending', 'in_progress')
ORDER BY priority DESC, created_at DESC
LIMIT 20;

-- QUERY 2: "Task counts by status" (summary widget)
SELECT status, COUNT(*) AS task_count
FROM demo_tasks
GROUP BY status
ORDER BY status;

-- QUERY 3: "Tasks from API source" (integration monitor)
SELECT id, title, status, metadata
FROM demo_tasks
WHERE metadata @> '{"source": "api"}'
  AND user_id = 42;
```

**Verification Script** (run after each sub-task):

```sql
-- ═══════════════════════════════════════════════
-- VERIFICATION SCRIPT
-- ═══════════════════════════════════════════════

-- CHECK 1: Index budget
SELECT COUNT(*) AS custom_index_count
FROM pg_indexes
WHERE tablename = 'demo_tasks' AND indexname != 'demo_tasks_pkey';
-- PASS: custom_index_count <= 3

-- CHECK 2: Query 1 plan
EXPLAIN (COSTS OFF)
SELECT id, title, status, priority, created_at
FROM demo_tasks
WHERE user_id = 42
  AND status IN ('pending', 'in_progress')
ORDER BY priority DESC, created_at DESC
LIMIT 20;
-- PASS: Contains 'Index Scan' (not 'Seq Scan')
-- BONUS: No 'Sort' node

-- CHECK 3: Query 1 timing
EXPLAIN ANALYZE
SELECT id, title, status, priority, created_at
FROM demo_tasks
WHERE user_id = 42
  AND status IN ('pending', 'in_progress')
ORDER BY priority DESC, created_at DESC
LIMIT 20;
-- PASS: Execution Time < 5.0 ms

-- CHECK 4: Query 2 plan
EXPLAIN (COSTS OFF)
SELECT status, COUNT(*) AS task_count
FROM demo_tasks GROUP BY status ORDER BY status;
-- PASS: Contains 'Index Only Scan' (not 'Seq Scan')

-- CHECK 5: Query 2 timing
EXPLAIN ANALYZE
SELECT status, COUNT(*) AS task_count
FROM demo_tasks GROUP BY status ORDER BY status;
-- PASS: Execution Time < 200 ms (Index Only Scan on 1M rows is fast but not instant)

-- CHECK 6: Query 3 plan
EXPLAIN (COSTS OFF)
SELECT id, title, status, metadata
FROM demo_tasks
WHERE metadata @> '{"source": "api"}' AND user_id = 42;
-- PASS: Does NOT contain 'Seq Scan'

-- CHECK 7: Query 3 timing
EXPLAIN ANALYZE
SELECT id, title, status, metadata
FROM demo_tasks
WHERE metadata @> '{"source": "api"}' AND user_id = 42;
-- PASS: Execution Time < 5.0 ms
```

---

### Task 3a: Design Indexes for Queries 1 and 2

Create indexes that optimize Query 1 and Query 2. Use at most 2 indexes.

For each index, write:
1. The `CREATE INDEX` statement
2. A one-sentence justification: why this index type? Why these columns in this order?

Run the verification script (Checks 1–5). All must pass.

**Design question to answer:** Query 2 scans the entire table to count rows per status. Can you make it an `Index Only Scan`? What does that require about the columns in the index?

---

### Task 3b: Add Index for Query 3

Query 3 filters on a JSONB containment operator (`@>`). Add a third index that makes Query 3 fast.

Run the full verification script (Checks 1–7). All must pass.

**Design question:** What index type is required for the `@>` operator? Can a B-tree index handle this? Why or why not?

---

### Task 3c: New Requirement — Case-Insensitive Title Search

The product team adds a fourth dashboard query:

```sql
-- QUERY 4: "Search tasks by title" (search bar)
SELECT id, title, status
FROM demo_tasks
WHERE LOWER(title) LIKE 'task #42%'
  AND user_id = 42;
```

This query must also use an index scan. But **you still can only have 3 custom indexes.** You must drop one of your existing indexes and replace it with one that serves Query 4.

**Tasks:**
1. Decide which existing index to sacrifice. Justify your choice: what is the performance tradeoff? What does the sacrificed query's plan become?
2. Write the `CREATE INDEX` statement for the replacement index.
3. Run all 7 verification checks plus this new one:

```sql
-- CHECK 8: Query 4 plan
EXPLAIN (COSTS OFF)
SELECT id, title, status FROM demo_tasks
WHERE LOWER(title) LIKE 'task #42%' AND user_id = 42;
-- PASS: Does NOT contain 'Seq Scan'

-- CHECK 9: Query 4 timing
EXPLAIN ANALYZE
SELECT id, title, status FROM demo_tasks
WHERE LOWER(title) LIKE 'task #42%' AND user_id = 42;
-- PASS: Execution Time < 10 ms
```

4. Write a brief **tradeoff analysis** (3-4 sentences): What improved? What degraded? Would you recommend this tradeoff to the product team?

---

# Level 3 Solutions

## Solution 3a: Indexes for Queries 1 and 2

**Index 1 — for Query 1:**

```sql
CREATE INDEX idx_q1_active_tasks
ON demo_tasks(user_id, priority DESC, created_at DESC)
WHERE status IN ('pending', 'in_progress');
```

**Justification:** Partial composite index. `WHERE` clause matches the query's status filter. `user_id` is leftmost (most selective, used for equality lookup). `priority DESC, created_at DESC` matches the `ORDER BY` exactly, enabling Sort elimination. The partial condition keeps the index small (~50% of rows excluded).

**Index 2 — for Query 2:**

```sql
CREATE INDEX idx_q2_status_count
ON demo_tasks(status);
```

**Justification:** B-tree on `status` enables `Index Only Scan` for `SELECT status, COUNT(*) GROUP BY status ORDER BY status`. The query only needs `status` — which is the indexed column. The B-tree is already sorted by status, so `ORDER BY status` comes free. With a recent `VACUUM`, `Heap Fetches: 0` and the table itself is never touched.

**Query 2 design answer:** For `Index Only Scan`, every column in `SELECT`, `WHERE`, `GROUP BY`, and `ORDER BY` must exist in the index. Query 2 only needs `status` — so a single-column index suffices. If the query also selected `user_id`, we'd need `(status, user_id)` in the index.

---

## Solution 3b: Index for Query 3

```sql
CREATE INDEX idx_q3_metadata
ON demo_tasks USING gin(metadata);
```

**Justification:** The `@>` (containment) operator on JSONB requires a **GIN** index. B-tree cannot handle `@>` because it operates on the *contents* of a JSON document, not on the document as a scalar value. GIN builds an inverted index: each JSON key-value pair points to the list of rows containing it.

With both `idx_q1_active_tasks` (has `user_id`) and `idx_q3_metadata` (GIN on JSONB), PostgreSQL might use a **BitmapAnd** to combine both indexes for Query 3, or it might use the GIN index alone with a Filter on `user_id`. Either way, the Seq Scan is eliminated.

All 7 checks pass. Index count: 3 (at budget).

---

## Solution 3c: Adding Query 4 Under Budget

**1. Which index to sacrifice:** Drop `idx_q2_status_count`.

**Reasoning:** Query 2 (`SELECT status, COUNT(*) GROUP BY status`) falls back to `Seq Scan` with `HashAggregate` — reading all 1M rows. This takes ~100-150ms instead of ~40ms with the Index Only Scan. It's slower, but:
- It's a dashboard summary widget, typically loaded once per page view
- 150ms is noticeable but not broken (unlike 500ms for an interactive search)
- The title search (Query 4) is triggered on every keystroke — latency matters more

```sql
DROP INDEX idx_q2_status_count;

CREATE INDEX idx_q4_title_search
ON demo_tasks(user_id, LOWER(title) text_pattern_ops);
```

**Why `text_pattern_ops`:** The `LIKE 'prefix%'` operator requires `text_pattern_ops` operator class for expression indexes when the database locale isn't `C`. Without it, PostgreSQL may not use the index for `LIKE` queries.

**Alternative approach** — if `text_pattern_ops` doesn't apply in the student's setup:
```sql
CREATE INDEX idx_q4_title_search ON demo_tasks(user_id, LOWER(title));
```
This still works for equality (`LOWER(title) = 'task #42'`) and may work for `LIKE 'prefix%'` depending on locale settings.

**2. Verification:**

After the swap:
- Check 1: 3 indexes ✅
- Check 2-3: Query 1 still fast (index unchanged) ✅
- Check 4-5: Query 2 now uses Seq Scan ❌ (degraded, ~120ms)
- Check 6-7: Query 3 still fast (GIN unchanged) ✅
- Check 8-9: Query 4 uses Index Scan ✅ (new!)

**3. Tradeoff analysis:**

By dropping the status-count index, the dashboard summary widget degrades from ~40ms (Index Only Scan) to ~120ms (Seq Scan with HashAggregate). This is acceptable because the summary loads once per page view. In exchange, the title search bar now responds in under 5ms instead of 300ms+ — a critical improvement for an interactive feature triggered on every keystroke. If the product team later reports the summary widget feels slow, we could add Redis caching (Week 10) rather than reclaiming the index slot. If the DBA increases the budget to 4 indexes, restore `idx_q2_status_count` immediately.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise 4: When Indexes Backfire

**Task:** Describe a realistic scenario where adding an index makes **overall system performance worse**, not better. Then prove it with a benchmark.

**Specific requirements:**

1. **Describe the scenario** in 3-4 sentences. What is the workload? What index is added? Why does it hurt?

2. **Write a benchmark** that demonstrates the degradation. Use this template on the `demo_tasks` table:

```sql
-- Step 1: Measure INSERT performance WITHOUT the index
-- (Drop all custom indexes first)
DO $$
DECLARE idx RECORD;
BEGIN
    FOR idx IN
        SELECT indexname FROM pg_indexes
        WHERE tablename = 'demo_tasks' AND indexname != 'demo_tasks_pkey'
    LOOP
        EXECUTE format('DROP INDEX IF EXISTS %I', idx.indexname);
    END LOOP;
END $$;

\timing

-- Insert 50,000 rows and record the time
INSERT INTO demo_tasks (title, description, status, priority, user_id, category_id, metadata, tags, created_at)
SELECT
    'Bench #' || i,
    'Benchmark task ' || i,
    (ARRAY['pending','in_progress','completed','cancelled'])[1 + (random()*3)::int],
    (random() * 5)::int,
    (random() * 999 + 1)::int,
    (random() * 19 + 1)::int,
    jsonb_build_object('source', 'benchmark'),
    ARRAY['bench'],
    NOW()
FROM generate_series(1, 50000) AS i;

-- Record: Time without indexes: _____ms

-- Step 2: Now add YOUR indexes and re-measure
-- [Student adds indexes here]

-- Delete the benchmark rows to make a fair re-test
DELETE FROM demo_tasks WHERE tags @> ARRAY['bench'];

-- Re-insert the same 50,000 rows
INSERT INTO demo_tasks (title, description, status, priority, user_id, category_id, metadata, tags, created_at)
SELECT /* same query as above */
FROM generate_series(1, 50000) AS i;

-- Record: Time with indexes: _____ms
-- Calculate: slowdown factor = time_with / time_without
```

3. **Answer this question:** A junior developer says *"I'll just add indexes for every column since reads are more important than writes."* Using your benchmark results, calculate: if the table receives 10,000 INSERTs per hour, how many additional seconds of write overhead per hour do your indexes add?

4. **Identify the threshold:** At what ratio of reads-to-writes would the index become worthwhile? (i.e., how many read queries would need to benefit from the index to offset the write overhead?)

---

# Level 4 Solution

## Solution 4: When Indexes Backfire

**1. Scenario:**

A task management system with a write-heavy workload — an ETL pipeline imports 10,000 tasks per hour from external sources. A developer adds 5 indexes (on `user_id`, `status`, `created_at`, a GIN on `metadata`, and a GIN on `tags`) to speed up dashboard queries. Each INSERT now updates 6 data structures (the heap + 5 indexes) instead of 1. The INSERT throughput drops by 3-5x, causing the ETL pipeline to fall behind. The dashboard queries (run 50 times/hour) save 200ms each — a total of 10 seconds saved on reads — while the indexes add 300+ seconds of write overhead per hour.

**2. Expected benchmark results** (approximate, varies by hardware):

| Metric | No Custom Indexes | 5 Custom Indexes |
|--------|------------------|-----------------|
| 50K INSERT time | ~800ms | ~3,500ms |
| Slowdown factor | 1.0x | ~4.4x |
| Per-INSERT overhead | ~0.016ms | ~0.070ms |

The GIN indexes on `metadata` and `tags` are particularly expensive: GIN maintains an inverted index structure that must be updated for every new JSONB document and every new array element. A single INSERT with `metadata = '{"source": "benchmark"}'` requires inserting entries for both the key `"source"` and the value `"benchmark"` into the GIN index.

**3. Hourly write overhead:**

```
Extra time per INSERT:  0.070ms - 0.016ms = 0.054ms
INSERTs per hour:       10,000
Extra overhead/hour:    10,000 × 0.054ms = 540ms ≈ 0.5 seconds
```

With 5 indexes, that's modest. But with larger rows, more indexes, or concurrent writes competing for locks on the same index pages, the overhead compounds. At 100K INSERTs/hour (a realistic batch processing load), the overhead becomes ~5.4 seconds/hour of pure index maintenance.

**4. Read-write breakeven:**

Each index-accelerated read query saves ~200ms. Each INSERT costs an extra ~0.054ms.

```
Breakeven: (extra_write_cost × writes_per_hour) = (read_savings × reads_per_hour)
           0.054ms × 10,000 = 200ms × R
           540ms = 200ms × R
           R = 2.7 reads per hour
```

If an index serves at least 3 read queries per hour, it pays for itself. This seems like a low bar — but it applies **per index**. An index with `idx_scan = 0` in `pg_stat_user_indexes` serves zero reads and has zero benefit. Its entire write overhead is pure waste.

**The real lesson:** The question isn't "are indexes good or bad?" It's "is THIS specific index earning its keep?" Use `pg_stat_user_indexes` to check `idx_scan` counts and drop indexes with zero or near-zero usage.

---

# LEVEL 5: THE TEACH BACK

## Exercise 5: The Cost Confusion

**Scenario:** A junior developer on your team runs EXPLAIN on two slow endpoints and comes to you with this analysis:

> *"I figured out which query to optimize first! Look:*
>
> *Query A (user search):*
> ```
> Seq Scan on users  (cost=0.00..4231.00 rows=50000 width=128)
> ```
>
> *Query B (task dashboard):*
> ```
> Index Scan using idx_tasks_user on tasks  (cost=0.42..892.15 rows=500 width=169)
> ```
>
> *Query A has cost 4231 and Query B has cost 892. So Query A is about 5x slower than Query B. I should optimize Query A first — it probably takes 4 seconds. Also, Query B already uses an Index Scan, so it's already optimized and I can ignore it."*

**The junior developer is making at least four conceptual errors.** Your task:

1. **Identify each error** in the developer's reasoning. List them.

2. **Explain why EXPLAIN cost cannot be compared across queries** to determine which is "slower." What do the cost numbers actually represent? What are they used for?

3. **The developer says "Query B uses Index Scan, so it's optimized."** Construct a specific counter-example: describe a scenario where an `Index Scan` query is still slow (over 500ms) despite using an index. What would EXPLAIN ANALYZE reveal that plain EXPLAIN would not?

4. **What should the developer do instead?** Walk them through the correct diagnostic workflow for comparing the performance of two queries. Be specific about which commands to run and what numbers to compare.

5. **Write the response** you would give this junior developer. Keep it respectful and educational — they took initiative by running EXPLAIN, which is good. But redirect them to the correct mental model. Maximum 8 sentences.

---

# Level 5 Solution

## Solution 5: The Cost Confusion

**1. Four errors:**

- **Error 1:** Comparing cost across queries. Cost 4231 vs 892 does NOT mean Query A is 5x slower. EXPLAIN costs are relative estimates used by the planner to compare candidate plans for the **same** query. They are not comparable across different queries because the two queries scan different tables with different sizes, widths, and statistical distributions.

- **Error 2:** Treating cost as milliseconds. "It probably takes 4 seconds" assumes cost ≈ milliseconds. Cost units are dimensionless and depend on PostgreSQL configuration parameters (`seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`). A cost of 4231 could correspond to 50ms or 5 seconds depending on hardware and configuration. Only `EXPLAIN ANALYZE` provides real wall-clock time.

- **Error 3:** Assuming Index Scan = "already optimized." An Index Scan can still be slow if: the index has poor selectivity (returns many rows), the query needs columns not in the index (many heap fetches), the ORDER BY requires a sort after the scan, or the estimated rows are wrong and the index scan is actually the wrong plan choice.

- **Error 4:** Using `EXPLAIN` (without `ANALYZE`) to assess performance. `EXPLAIN` shows the plan and **estimated** costs. It does NOT run the query. The developer has no actual performance data — only guesses from the planner. The estimates could be wildly wrong if statistics are stale.

**2. What cost numbers actually represent:**

Cost is measured in arbitrary "planner units" calibrated by `seq_page_cost` (default 1.0) and `random_page_cost` (default 4.0). One cost unit roughly represents "the effort of reading one sequential page from disk." The planner uses cost to compare candidate plans for the same query: "Should I Seq Scan (cost 4231) or use this index (cost 1892) for THIS query?" The lower-cost plan wins. But comparing costs between Query A (scanning `users`) and Query B (scanning `tasks`) is meaningless — the tables are different sizes, the row widths are different, and the I/O patterns are different.

**3. Counter-example where Index Scan is still slow:**

Scenario: `idx_tasks_user` is on `(user_id)`. The query is:
```sql
SELECT * FROM tasks WHERE user_id = 42 ORDER BY created_at DESC LIMIT 20;
```
The index finds user 42's rows quickly (Index Scan), but user 42 has 500,000 tasks. PostgreSQL reads all 500,000 via the index, sorts them by `created_at DESC`, then returns 20. The Sort node on 500K rows can take over 1 second.

`EXPLAIN` (without ANALYZE) would show: `cost=0.42..892.15 rows=500` — the planner estimates 500 rows. Looks fast.

`EXPLAIN ANALYZE` would reveal: `actual time=0.031..1,247.89 rows=500000` — the actual row count is 1000x the estimate. The Sort node shows `Sort Method: external merge Disk: 85MB`. Execution time: 1.4 seconds. The plan is catastrophically wrong because statistics are stale, but plain `EXPLAIN` hid this entirely.

**4. Correct diagnostic workflow:**

```sql
-- Step 1: Run EXPLAIN ANALYZE (not just EXPLAIN) on BOTH queries
EXPLAIN ANALYZE SELECT ... /* Query A */;
EXPLAIN ANALYZE SELECT ... /* Query B */;

-- Step 2: Compare ACTUAL Execution Time (the last line of output)
-- Query A Execution Time: ___ms
-- Query B Execution Time: ___ms
-- NOW you know which is actually slower.

-- Step 3: For the slower query, examine:
--   a. estimated rows vs actual rows (are they close?)
--   b. "Rows Removed by Filter" (reading wasted rows?)
--   c. Sort nodes (spilling to disk?)
--   d. Nested Loop loops count (N+1 in the plan?)

-- Step 4: If estimates are wrong, run ANALYZE on the table
-- Step 5: If a Seq Scan is the bottleneck, consider an index
-- Step 6: After any change, run EXPLAIN ANALYZE again to verify
```

**5. Response to the junior developer:**

"Good instinct reaching for EXPLAIN — that's the right tool. But EXPLAIN cost can't be compared across different queries; it's only used internally to pick the best plan for one query. To know which query is actually slower, run `EXPLAIN ANALYZE` on both — that gives you real execution times in milliseconds. Also, Index Scan doesn't automatically mean 'fast' — if the estimates are wrong or there's a Sort hiding underneath, an Index Scan can still take seconds. Check `Execution Time` and compare estimated vs actual row counts. Start with whichever has the higher actual execution time."