# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SLOW FIRST, FAST LATER                                         │
│  ──────────────────────                                         │
│  Students must EXPERIENCE a slow query before learning to       │
│  optimize it. We'll make them wait. Again.                      │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Query optimization is invisible. We use a library analogy      │
│  throughout. Every concept maps to finding books.               │
│                                                                 │
│  DIAGNOSE BEFORE PRESCRIBE                                      │
│  ────────────────────────                                       │
│  EXPLAIN comes before CREATE INDEX. You must SEE the            │
│  problem before applying the fix. No guessing.                  │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Week 5 EXPLAIN intro → now you master it                       │
│  Week 5 JSONB/Arrays/Full-text → now you learn to index them   │
│  Week 6 SQLAlchemy queries → the ORM generating these queries   │
│  Week 6 N+1 problem → one query problem, now we solve more     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                    QUERY OPTIMIZATION                           │
│                    (3–4 Hour Lecture)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 The Million-Row Wake-Up Call (Demonstration)            │
│  ├─ 1.2 Why Queries Get Slow (Sequential Scan)                  │
│  ├─ 1.3 The Library Analogy                                     │
│  └─ 1.4 The Fix Preview (Index Scan)                            │
│                                                                 │
│  PART 2: READING QUERY PLANS (60 min)                           │
│  ├─ 2.1 EXPLAIN — The Blueprint                                 │
│  ├─ 2.2 EXPLAIN ANALYZE — The Ground Truth                      │
│  ├─ 2.3 Anatomy of a Plan Node                                  │
│  ├─ 2.4 Scan Types (How PostgreSQL Finds Rows)                  │
│  ├─ 2.5 Join Strategies (How PostgreSQL Combines Tables)        │
│  └─ 2.6 When Estimates Lie (Statistics and ANALYZE)             │
│                                                                 │
│  PART 3: INDEX TYPES (50 min)                                   │
│  ├─ 3.1 What an Index Actually Is                               │
│  ├─ 3.2 B-tree — The Default Workhorse                          │
│  ├─ 3.3 Hash — Equality and Nothing Else                        │
│  ├─ 3.4 GIN — The Inside-Out Index                              │
│  ├─ 3.5 GiST — The Swiss Army Knife                            │
│  └─ 3.6 Choosing the Right Index Type                           │
│                                                                 │
│  PART 4: ADVANCED INDEX STRATEGIES (45 min)                     │
│  ├─ 4.1 Composite Indexes (Column Order Is Everything)          │
│  ├─ 4.2 Partial Indexes (Index Only What Matters)               │
│  ├─ 4.3 Expression Indexes (Index Computed Values)              │
│  ├─ 4.4 Index-Only Scans (The Holy Grail)                       │
│  └─ 4.5 The Hidden Cost of Indexes                              │
│                                                                 │
│  PART 5: QUERY PROFILING WITH SQLALCHEMY (30 min)               │
│  ├─ 5.1 SQL Logging with echo                                   │
│  ├─ 5.2 Event-Based Query Timing                                │
│  ├─ 5.3 Slow Query Detection in FastAPI                         │
│  └─ 5.4 The Optimization Workflow                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Million-Row Wake-Up Call

**Start with a demonstration. Make them feel the pain again.**

Your Task Manager has been working great in development with 50 tasks. Let's see what happens in the real world.

```sql
-- demo_setup.sql — Run this in your Docker PostgreSQL (psql)
-- This mirrors your Task Manager schema but with REAL volume.

CREATE TABLE demo_tasks (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'pending',
    priority INTEGER NOT NULL DEFAULT 0,
    user_id INTEGER NOT NULL,
    category_id INTEGER,
    metadata JSONB DEFAULT '{}',
    tags TEXT[] DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Insert 1,000,000 rows. This takes about 10-20 seconds.
INSERT INTO demo_tasks (
    title, description, status, priority,
    user_id, category_id, metadata, tags, created_at
)
SELECT
    'Task #' || i,
    'Description for task number ' || i,
    (ARRAY['pending','in_progress','completed','cancelled'])[1 + (random()*3)::int],
    (random() * 5)::int,
    (random() * 999 + 1)::int,
    (random() * 19 + 1)::int,
    jsonb_build_object(
        'source', (ARRAY['web','api','import','migration'])[1 + (random()*3)::int],
        'version', (random() * 5 + 1)::int
    ),
    ARRAY[(ARRAY['urgent','bug','feature','docs','refactor'])[1 + (random()*4)::int]],
    NOW() - ((random() * 365)::int || ' days')::interval
FROM generate_series(1, 1000000) AS i;
```

**Turn on timing, then run a query students would write in their project:**

```sql
\timing

-- "Show me all tasks for user 42"
SELECT * FROM demo_tasks WHERE user_id = 42;
```

```
  id   |    title    |      description       |  status   | priority | user_id | ...
-------+-------------+------------------------+-----------+----------+---------+----
 14023 | Task #14023 | Description for task... | pending   |        2 |      42 | ...
 28401 | Task #28401 | Description for task... | completed |        4 |      42 | ...
 ...
(approximately 1000 rows)

Time: 127.401 ms
```

**Now ask the class:**

> "127 milliseconds for one query. That doesn't sound bad, right? Now imagine 100 users hitting your API at the same time. Each request runs this query. That's 12.7 seconds of pure database work — on ONE endpoint. Your API starts timing out. Your users see spinners. What went wrong?"

**Scale it further:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  HOW 127ms DESTROYS YOUR API                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1 request  :  127ms    ← "That's fine"                         │
│  10 requests:  1.3s     ← "Getting slow..."                     │
│  100 requests: 12.7s    ← "API is timing out"                   │
│  1000 req/s :  💀       ← "Production is down"                  │
│                                                                 │
│  And this is ONE query. Your endpoint probably runs 3-4         │
│  queries per request (task + user + category + tags).           │
│                                                                 │
│  Your connection pool from Week 6 has a limited size.           │
│  When all connections are busy waiting for slow queries,        │
│  new requests queue up. This is how you get cascading           │
│  failures.                                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 Why Queries Get Slow

**Let's see what PostgreSQL actually did to answer that query.**

> "In Week 5, you saw EXPLAIN for the first time — a quick introduction. Now we use it for real."

```sql
EXPLAIN SELECT * FROM demo_tasks WHERE user_id = 42;
```

```
                          QUERY PLAN
--------------------------------------------------------------
 Seq Scan on demo_tasks  (cost=0.00..20834.00 rows=1002 width=169)
   Filter: (user_id = 42)
```

**Focus on two words: `Seq Scan`.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEQUENTIAL SCAN                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Seq Scan" means PostgreSQL is reading EVERY SINGLE ROW       │
│  in the table to find the ones where user_id = 42.             │
│                                                                 │
│                                                                 │
│  Row 1:      user_id = 7    → No. Skip.                        │
│  Row 2:      user_id = 483  → No. Skip.                        │
│  Row 3:      user_id = 42   → Yes! Keep.                       │
│  Row 4:      user_id = 891  → No. Skip.                        │
│  Row 5:      user_id = 42   → Yes! Keep.                       │
│  ...                                                            │
│  Row 999,999: user_id = 155 → No. Skip.                        │
│  Row 1,000,000: user_id = 2 → No. Skip.                        │
│                                                                 │
│                                                                 │
│  To find ~1,000 matching rows, PostgreSQL read all 1,000,000.  │
│  It checked every single row. 999,000 of those checks           │
│  were wasted.                                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Imagine searching for a friend's phone number by reading every name in a phonebook, starting from page 1, line 1, all the way to the end. That's what PostgreSQL just did."

---

## 1.3 The Library Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE LIBRARY ANALOGY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LIBRARY WITHOUT A CATALOG (Sequential Scan)                    │
│  ───────────────────────────────────────────                    │
│                                                                 │
│  "Find me every book by Author X."                              │
│                                                                 │
│  1. Walk to Shelf 1, check every book's author.                 │
│  2. Walk to Shelf 2, check every book's author.                 │
│  3. Walk to Shelf 3...                                          │
│  4. ... keep going through all 1,000 shelves.                   │
│  5. Found 8 books. Took 2 hours. 😤                             │
│                                                                 │
│                                                                 │
│  LIBRARY WITH A CARD CATALOG (Index Scan)                       │
│  ────────────────────────────────────────                       │
│                                                                 │
│  "Find me every book by Author X."                              │
│                                                                 │
│  1. Open the Author catalog drawer.                             │
│  2. Flip to "X." See 8 entries with shelf locations.            │
│  3. Walk directly to those 8 shelves.                           │
│  4. Grab the 8 books. Took 3 minutes. 😊                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to PostgreSQL:**

```
Library                     │  PostgreSQL
────────────────────────────│──────────────────────────────
Library building            │  Database
One bookshelf               │  Table
A single book               │  A row
Book's properties            │  Column values
  (title, author, ISBN)     │    (title, user_id, status)
Card catalog                │  Index
Walking every shelf         │  Sequential Scan (Seq Scan)
Look up catalog, go to shelf│  Index Scan
Read info from catalog card │  Index Only Scan
  without visiting shelf    │    (all data is in the index)
Librarian deciding how      │  Query Planner
  to find your book         │    (chooses the best strategy)
```

---

## 1.4 The Fix Preview

**Now add a card catalog (index) and watch what happens:**

```sql
-- Create an index on user_id
CREATE INDEX idx_demo_tasks_user_id ON demo_tasks(user_id);

-- Run the same query
\timing
SELECT * FROM demo_tasks WHERE user_id = 42;
```

```
(approximately 1000 rows)

Time: 1.842 ms
```

**127ms → 1.8ms. That's a 70x speedup.**

```sql
EXPLAIN SELECT * FROM demo_tasks WHERE user_id = 42;
```

```
                          QUERY PLAN
--------------------------------------------------------------
 Index Scan using idx_demo_tasks_user_id on demo_tasks
   (cost=0.42..1025.43 rows=1002 width=169)
   Index Cond: (user_id = 42)
```

**Two critical differences from before:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  BEFORE VS AFTER INDEX                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE:                                                        │
│  ────────                                                       │
│    Seq Scan on demo_tasks  (cost=0.00..20834.00 ...)            │
│      Filter: (user_id = 42)                                     │
│                 ▲                                               │
│                 └─ "Filter" means: read row FIRST, check AFTER │
│                    (read 1,000,000 rows, keep 1,000)            │
│                                                                 │
│                                                                 │
│  AFTER:                                                         │
│  ───────                                                        │
│    Index Scan using idx_demo_tasks_user_id ...                  │
│      Index Cond: (user_id = 42)                                 │
│                    ▲                                            │
│                    └─ "Index Cond" means: find in index FIRST  │
│                       (look up index, read only ~1,000 rows)    │
│                                                                 │
│                                                                 │
│  "Filter" = read everything, discard what doesn't match         │
│  "Index Cond" = look up matching rows first, read only those   │
│                                                                 │
│  This distinction is CRITICAL for reading query plans.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Now the question is: how do you read these query plans properly? How do you know when something is slow and why? And how do you choose the right type of index? That's what the rest of this lecture is about."

---

# PART 2: READING QUERY PLANS

## 2.1 EXPLAIN — The Blueprint

**`EXPLAIN` shows what PostgreSQL *plans* to do, without actually running the query.**

```sql
EXPLAIN SELECT * FROM demo_tasks WHERE status = 'pending';
```

```
                          QUERY PLAN
--------------------------------------------------------------
 Seq Scan on demo_tasks  (cost=0.00..20834.00 rows=250422 width=169)
   Filter: (status = 'pending'::text)
```

**Think of it as a construction blueprint:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 EXPLAIN = BLUEPRINT                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  An architect's blueprint tells you:                            │
│  • What will be built (scan type, join type)                    │
│  • Estimated cost (how expensive)                               │
│  • Estimated size (how many rows)                               │
│                                                                 │
│  But it does NOT tell you:                                      │
│  • How long construction actually takes (no real timing)        │
│  • Whether the estimate was accurate (no actual row counts)     │
│  • What resources were used (no buffer/IO stats)                │
│                                                                 │
│  EXPLAIN is SAFE: it never executes the query.                  │
│  Use it freely, even on DELETE or UPDATE — nothing happens.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When to use plain EXPLAIN:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ✅ Use plain EXPLAIN when:                                     │
│  ├─ You want to see the plan for a DELETE or UPDATE             │
│  │   without actually modifying data                            │
│  ├─ The query is very slow and you don't want to wait           │
│  └─ You just want to check if an index is being used            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 EXPLAIN ANALYZE — The Ground Truth

**`EXPLAIN ANALYZE` actually runs the query and shows real numbers.**

```sql
EXPLAIN ANALYZE SELECT * FROM demo_tasks WHERE user_id = 42;
```

```
                          QUERY PLAN
--------------------------------------------------------------
 Index Scan using idx_demo_tasks_user_id on demo_tasks
   (cost=0.42..1025.43 rows=1002 width=169)
   (actual time=0.031..1.614 rows=1003 loops=1)
   Index Cond: (user_id = 42)
 Planning Time: 0.095 ms
 Execution Time: 1.842 ms
```

**Now compare: plan BEFORE we added the index:**

```sql
-- Drop the index to see the difference (we'll recreate it)
DROP INDEX idx_demo_tasks_user_id;

EXPLAIN ANALYZE SELECT * FROM demo_tasks WHERE user_id = 42;
```

```
                          QUERY PLAN
--------------------------------------------------------------
 Seq Scan on demo_tasks
   (cost=0.00..20834.00 rows=1002 width=169)
   (actual time=0.012..127.109 rows=1003 loops=1)
   Filter: (user_id = 42)
   Rows Removed by Filter: 998997
 Planning Time: 0.078 ms
 Execution Time: 127.401 ms
```

**Side by side:**

```
┌─────────────────────────────────────────────────────────────────┐
│              SEQ SCAN VS INDEX SCAN — NUMBERS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    Seq Scan          Index Scan                  │
│                    ────────          ──────────                  │
│  Estimated cost:   0.00..20834.00   0.42..1025.43               │
│  Estimated rows:   1,002            1,002                       │
│  Actual time:      0.01..127.1 ms   0.03..1.6 ms               │
│  Actual rows:      1,003            1,003                       │
│  Rows examined:    1,000,000        ~1,003                      │
│  Execution time:   127.401 ms       1.842 ms                   │
│                                                                 │
│  Speedup: ~70x                                                  │
│                                                                 │
│  The Seq Scan read 998,997 rows that it threw away.            │
│  The Index Scan read only the rows it needed.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Recreate the index for the rest of the lecture:**

```sql
CREATE INDEX idx_demo_tasks_user_id ON demo_tasks(user_id);
```

**⚠️ WARNING: EXPLAIN ANALYZE actually executes the query!**

```
┌─────────────────────────────────────────────────────────────────┐
│                       ⚠️  DANGER                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EXPLAIN ANALYZE runs the query for real.                       │
│                                                                 │
│  This is SAFE for SELECT.                                       │
│                                                                 │
│  This is DANGEROUS for:                                         │
│    EXPLAIN ANALYZE DELETE FROM demo_tasks;  ← Deletes all rows!│
│    EXPLAIN ANALYZE UPDATE demo_tasks SET status = 'cancelled';  │
│                                                                 │
│  To safely analyze writes, wrap in a transaction:               │
│    BEGIN;                                                       │
│    EXPLAIN ANALYZE DELETE FROM demo_tasks WHERE id = 1;         │
│    ROLLBACK;  ← Undo the delete, keep the plan output          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Anatomy of a Plan Node

**Every line in a query plan is a "node." Let's dissect one:**

```
Index Scan using idx_demo_tasks_user_id on demo_tasks
  (cost=0.42..1025.43 rows=1002 width=169)
  (actual time=0.031..1.614 rows=1003 loops=1)
```

**Break it down field by field:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  ANATOMY OF A PLAN NODE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Index Scan using idx_demo_tasks_user_id on demo_tasks          │
│  ──────────       ──────────────────────    ──────────          │
│  Node type        Which index               Which table         │
│                                                                 │
│                                                                 │
│  (cost=0.42..1025.43 rows=1002 width=169)                       │
│        ────  ───────  ────────  ─────────                       │
│          │      │        │         │                            │
│          │      │        │         └─ Estimated average row     │
│          │      │        │            width in bytes             │
│          │      │        │                                      │
│          │      │        └─ Estimated number of rows            │
│          │      │           this node will produce               │
│          │      │                                               │
│          │      └─ Total cost to complete this node             │
│          │         (arbitrary units — not milliseconds!)         │
│          │                                                      │
│          └─ Startup cost: work before first row is returned    │
│                                                                 │
│                                                                 │
│  (actual time=0.031..1.614 rows=1003 loops=1)                   │
│              ─────  ─────  ────────  ───────                    │
│                │      │       │         │                       │
│                │      │       │         └─ How many times this  │
│                │      │       │            node executed         │
│                │      │       │                                 │
│                │      │       └─ ACTUAL number of rows          │
│                │      │          (compare to estimate above!)    │
│                │      │                                         │
│                │      └─ Actual total time (milliseconds)       │
│                │                                                │
│                └─ Actual startup time (milliseconds)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The most important comparisons:**

```
┌─────────────────────────────────────────────────────────────────┐
│               WHAT TO CHECK FIRST                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ESTIMATED ROWS vs ACTUAL ROWS                               │
│     ─────────────────────────────                               │
│     rows=1002  vs  rows=1003                                    │
│                                                                 │
│     Close? Good — the planner's statistics are accurate.        │
│     Way off? Bad — the planner made a wrong decision.           │
│     (We'll cover why in section 2.6)                            │
│                                                                 │
│                                                                 │
│  2. ACTUAL TIME                                                 │
│     ───────────                                                 │
│     actual time=0.031..1.614                                    │
│                                                                 │
│     The second number is total wall-clock time for this node.   │
│     This is what you care about most.                           │
│                                                                 │
│                                                                 │
│  3. ROWS REMOVED BY FILTER                                      │
│     ─────────────────────────                                   │
│     Rows Removed by Filter: 998997                              │
│                                                                 │
│     If this number is high, you're reading rows you don't need.│
│     This is the #1 sign you need an index.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**About "cost" units:**

> "The cost numbers are NOT milliseconds. They're arbitrary units used by the planner to compare different strategies. Think of them as 'effort points.' A plan with cost 1000 is roughly 10x more work than a plan with cost 100 — but the actual time depends on your hardware, data size, and cache state. Never try to convert cost to milliseconds. Always use EXPLAIN ANALYZE for real timings."

---

## 2.4 Scan Types (How PostgreSQL Finds Rows)

**There are four main ways PostgreSQL reads data from a table.**

### Sequential Scan (Seq Scan)

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEQUENTIAL SCAN                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Reads EVERY row in the table, from start to finish.            │
│                                                                 │
│  Library: Walking down every shelf, checking every book.        │
│                                                                 │
│  Table:  [ row1 ][ row2 ][ row3 ][ row4 ][ row5 ][ ... ]       │
│             ▲       ▲       ▲       ▲       ▲                  │
│             │       │       │       │       │                  │
│           check   check   check   check   check  ← every row  │
│             ✗       ✗       ✓       ✗       ✓    ← keep/skip  │
│                                                                 │
│  When PostgreSQL uses it:                                       │
│  ├─ No index exists for the filter column                       │
│  ├─ The query returns a LARGE fraction of the table             │
│  │   (e.g., 50%+ of rows match — index wouldn't help)           │
│  └─ Table is very small (faster to just read it all)            │
│                                                                 │
│  EXPLAIN shows:                                                 │
│    Seq Scan on demo_tasks                                       │
│      Filter: (user_id = 42)      ← post-read filtering        │
│      Rows Removed by Filter: N   ← wasted reads               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Index Scan

```
┌─────────────────────────────────────────────────────────────────┐
│                      INDEX SCAN                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Looks up matching entries in the index, then fetches those     │
│  rows from the table.                                           │
│                                                                 │
│  Library: Check the card catalog, walk to those shelves only.   │
│                                                                 │
│  Index:  [..., 42→row3, 42→row5, 42→row91, ...]                │
│                   │        │         │                          │
│                   ▼        ▼         ▼                          │
│  Table:  [   ][ row3 ][   ][ row5 ][   ]...[row91][   ]        │
│                  ▲           ▲              ▲                   │
│                fetch       fetch          fetch  ← only these  │
│                                                                 │
│  When PostgreSQL uses it:                                       │
│  ├─ An index exists for the filter column                       │
│  ├─ The query returns a SMALL fraction of the table             │
│  └─ The query needs columns NOT in the index (must visit table)│
│                                                                 │
│  EXPLAIN shows:                                                 │
│    Index Scan using idx_name on demo_tasks                      │
│      Index Cond: (user_id = 42)  ← efficient: condition in idx │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Index Only Scan

```
┌─────────────────────────────────────────────────────────────────┐
│                   INDEX ONLY SCAN                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  All needed data is IN the index. Never touches the table.      │
│                                                                 │
│  Library: The catalog card has all the info you need            │
│  (title, author, year). You don't need to go to the shelf.     │
│                                                                 │
│  Index:  [..., (42, "pending", 3), (42, "completed", 1), ...]  │
│                        │                    │                   │
│                    return this          return this              │
│                                                                 │
│  Table:  [never touched — huge savings!]                        │
│                                                                 │
│  When PostgreSQL uses it:                                       │
│  ├─ All columns in SELECT, WHERE, ORDER BY are in the index    │
│  └─ Visibility map confirms rows are visible (no recent updates)│
│                                                                 │
│  EXPLAIN shows:                                                 │
│    Index Only Scan using idx_name on demo_tasks                 │
│      Index Cond: (user_id = 42)                                 │
│      Heap Fetches: 0   ← 0 means truly index-only! Perfect.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Bitmap Index Scan + Bitmap Heap Scan

```
┌─────────────────────────────────────────────────────────────────┐
│                   BITMAP SCAN (Two-Phase)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Phase 1: Scan the index, build a "bitmap" of matching pages.   │
│  Phase 2: Read those pages from the table, in order.            │
│                                                                 │
│  Library: First, write down all the shelf numbers from the      │
│  catalog. Then visit shelves in order (efficient walking path). │
│                                                                 │
│  Why not just Index Scan?                                       │
│  When many rows match, Index Scan jumps around randomly in the  │
│  table (row3, row5, row91...). Random I/O is slow.              │
│  Bitmap Scan reads pages sequentially — much faster for disk.   │
│                                                                 │
│  Also: Bitmap Scan can COMBINE multiple indexes!                │
│                                                                 │
│  EXPLAIN shows:                                                 │
│    Bitmap Heap Scan on demo_tasks                               │
│      Recheck Cond: (user_id = 42)                               │
│      ->  Bitmap Index Scan on idx_demo_tasks_user_id            │
│            Index Cond: (user_id = 42)                           │
│                                                                 │
│  The arrow (->)  means "child node feeds into parent node."     │
│  Read plans BOTTOM-UP: index scan happens first, then heap scan.│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When does PostgreSQL choose each?**

```
┌─────────────────────────────────────────────────────────────────┐
│             SCAN TYPE SELECTION (Planner Logic)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  % of table rows matching:                                      │
│                                                                 │
│  0%          5%           20%          50%          100%         │
│  │           │            │            │            │           │
│  ├───────────┤            │            │            │           │
│  │Index Scan │            │            │            │           │
│  │           ├────────────┤            │            │           │
│  │           │Bitmap Scan │            │            │           │
│  │           │            ├────────────┴────────────┤           │
│  │           │            │      Seq Scan           │           │
│  │           │            │                         │           │
│                                                                 │
│  Few rows   → Index Scan (random I/O, but very few reads)      │
│  Medium     → Bitmap Scan (sorted I/O, moderate reads)          │
│  Many rows  → Seq Scan (read everything, no overhead)           │
│                                                                 │
│  The exact thresholds depend on table size, row width,          │
│  and PostgreSQL's random_page_cost / seq_page_cost settings.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Demonstrate with a query that matches many rows:**

```sql
-- ~25% of rows have status = 'pending'
EXPLAIN ANALYZE SELECT * FROM demo_tasks WHERE status = 'pending';
```

```
                          QUERY PLAN
--------------------------------------------------------------
 Seq Scan on demo_tasks
   (cost=0.00..20834.00 rows=250422 width=169)
   (actual time=0.013..104.552 rows=250108 loops=1)
   Filter: (status = 'pending'::text)
   Rows Removed by Filter: 749892
 Planning Time: 0.075 ms
 Execution Time: 117.231 ms
```

> "Even if we created an index on `status`, PostgreSQL would probably IGNORE it for this query. Returning 25% of the table means reading most of the data anyway — a Seq Scan is cheaper. The planner is smart. Not every slow query is fixed by adding an index."

---

## 2.5 Join Strategies (How PostgreSQL Combines Tables)

**In Week 5, you learned JOINs. Now see how PostgreSQL executes them.**

```sql
-- Create a small categories table for the demo
CREATE TABLE demo_categories (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);
INSERT INTO demo_categories (name)
SELECT 'Category ' || i FROM generate_series(1, 20) AS i;

-- Join tasks with categories
EXPLAIN ANALYZE
SELECT t.title, t.status, c.name as category
FROM demo_tasks t
JOIN demo_categories c ON t.category_id = c.id
WHERE t.user_id = 42;
```

```
                          QUERY PLAN
--------------------------------------------------------------
 Nested Loop  (cost=0.71..1034.17 rows=1002 width=46)
              (actual time=0.051..2.103 rows=1003 loops=1)
   ->  Index Scan using idx_demo_tasks_user_id on demo_tasks t
         (cost=0.42..1025.43 rows=1002 width=30)
         (actual time=0.032..1.587 rows=1003 loops=1)
         Index Cond: (user_id = 42)
   ->  Index Scan using demo_categories_pkey on demo_categories c
         (cost=0.14..0.16 rows=1 width=20)
         (actual time=0.000..0.000 rows=1 loops=1003)
         Index Cond: (id = t.category_id)
 Planning Time: 0.215 ms
 Execution Time: 2.312 ms
```

**Read plans bottom-up, inside-out:**

```
┌─────────────────────────────────────────────────────────────────┐
│               READING A JOIN PLAN                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: Inner scan (first child)                               │
│    Index Scan on demo_tasks WHERE user_id = 42                  │
│    → Finds 1,003 tasks for user 42                              │
│                                                                 │
│  Step 2: For EACH of those 1,003 rows...                        │
│    Index Scan on demo_categories WHERE id = t.category_id       │
│    → Look up the category name (loops=1003!)                    │
│                                                                 │
│  Step 3: Nested Loop combines them                              │
│    → Returns 1,003 joined rows                                  │
│                                                                 │
│  Notice "loops=1003" on the categories scan.                    │
│  That means this node ran 1,003 times — once per task.          │
│  Actual time per loop: ~0.001ms (fast because of PK index).    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**PostgreSQL has three join strategies:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THREE JOIN STRATEGIES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NESTED LOOP                                                    │
│  ───────────                                                    │
│  For each row in A, look up matching rows in B.                 │
│                                                                 │
│  Library: For each reader request, check one catalog.           │
│  Best when: One side is small, other side is indexed.           │
│  Worst when: Both sides are large (N × M lookups).              │
│                                                                 │
│                                                                 │
│  HASH JOIN                                                      │
│  ─────────                                                      │
│  Build a hash table from the smaller table.                     │
│  Scan the larger table, probe the hash for each row.            │
│                                                                 │
│  Library: Photocopy the small catalog into a quick-lookup       │
│  cheat sheet, then walk the shelves checking against it.        │
│  Best when: No useful index, one side fits in memory.           │
│                                                                 │
│                                                                 │
│  MERGE JOIN                                                     │
│  ──────────                                                     │
│  Sort both sides on the join key, then merge like a zipper.     │
│                                                                 │
│  Library: Sort both lists alphabetically, then walk             │
│  them side-by-side matching as you go.                          │
│  Best when: Both sides are already sorted (by index).           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You don't choose the join strategy — the planner does. But when you see a Hash Join where you expected a Nested Loop, that tells you an index might be missing."

---

## 2.6 When Estimates Lie

**Sometimes the planner makes bad decisions because its statistics are wrong.**

```sql
-- Delete half the rows but don't update statistics
DELETE FROM demo_tasks WHERE id > 500000;

-- The planner still thinks there are 1M rows!
EXPLAIN SELECT * FROM demo_tasks WHERE user_id = 42;
```

```
                          QUERY PLAN
--------------------------------------------------------------
 Index Scan using idx_demo_tasks_user_id on demo_tasks
   (cost=0.42..1025.43 rows=1002 width=169)
                              ▲
                              └── Still estimates ~1000 rows
                                  even though we deleted half!
```

**The fix: update statistics.**

```sql
-- Tell PostgreSQL to re-examine the table
ANALYZE demo_tasks;

-- Now the estimates match reality
EXPLAIN SELECT * FROM demo_tasks WHERE user_id = 42;
```

```
                          QUERY PLAN
--------------------------------------------------------------
 Index Scan using idx_demo_tasks_user_id on demo_tasks
   (cost=0.42..512.71 rows=501 width=169)
                            ▲
                            └── Now correctly estimates ~500
```

**Restore the demo data for the rest of the lecture:**

```sql
-- Re-populate to 1M rows
TRUNCATE demo_tasks;
INSERT INTO demo_tasks (
    title, description, status, priority,
    user_id, category_id, metadata, tags, created_at
)
SELECT
    'Task #' || i,
    'Description for task number ' || i,
    (ARRAY['pending','in_progress','completed','cancelled'])[1 + (random()*3)::int],
    (random() * 5)::int,
    (random() * 999 + 1)::int,
    (random() * 19 + 1)::int,
    jsonb_build_object(
        'source', (ARRAY['web','api','import','migration'])[1 + (random()*3)::int],
        'version', (random() * 5 + 1)::int
    ),
    ARRAY[(ARRAY['urgent','bug','feature','docs','refactor'])[1 + (random()*4)::int]],
    NOW() - ((random() * 365)::int || ' days')::interval
FROM generate_series(1, 1000000) AS i;

ANALYZE demo_tasks;
-- Recreate index after truncate
CREATE INDEX idx_demo_tasks_user_id ON demo_tasks(user_id);
```

```
┌─────────────────────────────────────────────────────────────────┐
│               WHEN ESTIMATES GO WRONG                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BAD ESTIMATES CAUSE BAD PLANS:                                 │
│                                                                 │
│  Planner thinks 10 rows    → picks Nested Loop                  │
│  Actual result: 100,000    → Nested Loop is disastrous          │
│                                                                 │
│  Planner thinks 1M rows    → picks Seq Scan                     │
│  Actual result: 5          → Seq Scan wastes time               │
│                                                                 │
│                                                                 │
│  HOW TO SPOT IT:                                                │
│                                                                 │
│  EXPLAIN ANALYZE shows BOTH estimated and actual rows.          │
│  If they differ by 10x or more, statistics are stale.           │
│                                                                 │
│  rows=1002  vs  rows=1003    ← Fine (within 1%)                │
│  rows=100   vs  rows=50000   ← Bad! Run ANALYZE.               │
│                                                                 │
│                                                                 │
│  PREVENTION:                                                    │
│                                                                 │
│  PostgreSQL runs autovacuum which periodically runs ANALYZE.    │
│  But after large data changes (bulk imports, deletes),          │
│  run ANALYZE manually.                                          │
│                                                                 │
│  ANALYZE demo_tasks;            -- one table                    │
│  ANALYZE;                       -- entire database              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: INDEX TYPES

## 3.1 What an Index Actually Is

**Before we learn index types, understand what ALL indexes share:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHAT IS AN INDEX?                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  An index is a SEPARATE DATA STRUCTURE that:                    │
│                                                                 │
│  1. Stores a copy of SOME columns (the indexed columns)         │
│  2. Organizes them for fast lookup                              │
│  3. Points back to the full row in the table                    │
│                                                                 │
│                                                                 │
│      TABLE (heap)              INDEX                            │
│    ┌─────────────────┐     ┌──────────────────┐                 │
│    │ id=1, user=7,   │     │ user=7  → row 1  │                 │
│    │ title="Fix..."  │     │ user=42 → row 3  │                 │
│    ├─────────────────┤     │ user=42 → row 5  │                 │
│    │ id=2, user=483, │     │ user=99 → row 4  │                 │
│    │ title="Add..."  │     │ user=483→ row 2  │                 │
│    ├─────────────────┤     └──────────────────┘                 │
│    │ id=3, user=42,  │       ▲                                  │
│    │ title="Bug..."  │       │ Sorted/organized for             │
│    ├─────────────────┤       │ fast lookup by user_id           │
│    │ id=4, user=99,  │                                          │
│    │ title="New..."  │     The table itself is NOT sorted.      │
│    ├─────────────────┤     Rows are stored in insertion order   │
│    │ id=5, user=42,  │     (heap). The index provides order.   │
│    │ title="Ref..."  │                                          │
│    └─────────────────┘                                          │
│                                                                 │
│  Key insight: The table stores ALL the data.                    │
│  The index stores KEYS + POINTERS. It's smaller but separate.  │
│  Every INSERT, UPDATE, DELETE must update BOTH.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Now the question is: HOW does the index organize the keys? Different organizations are good for different types of queries. That's why we have different index types."

---

## 3.2 B-tree — The Default Workhorse

**B-tree is what you get when you don't specify a type. It handles almost everything.**

```sql
-- These are equivalent:
CREATE INDEX idx_user ON demo_tasks(user_id);
CREATE INDEX idx_user ON demo_tasks USING btree(user_id);
```

**How a B-tree is organized:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    B-TREE STRUCTURE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A B-tree is a balanced tree where:                             │
│  • Internal nodes contain keys that guide the search            │
│  • Leaf nodes contain keys + pointers to table rows             │
│  • Leaf nodes are linked for range scans                        │
│                                                                 │
│                         [500]                                   │
│                        /     \                                  │
│                  [250]         [750]                             │
│                 /     \       /     \                            │
│             [125]   [375] [625]   [875]                         │
│              / \     / \   / \     / \                           │
│           ┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐┌──┐                     │
│           │  ││  ││  ││  ││  ││  ││  ││  │ ← Leaf nodes        │
│           └─┬┘└─┬┘└─┬┘└─┬┘└─┬┘└─┬┘└─┬┘└─┬┘   (linked list)    │
│             └──►└──►└──►└──►└──►└──►└──►┘                      │
│                                                                 │
│                                                                 │
│  Finding user_id = 42:                                          │
│    Root [500] → 42 < 500, go left                               │
│    Node [250] → 42 < 250, go left                               │
│    Node [125] → 42 < 125, go left                               │
│    Leaf node  → scan for 42 → found! Follow pointer to row.    │
│                                                                 │
│  Depth: ~3-4 levels for millions of rows.                       │
│  That means 3-4 page reads to find ANY value. O(log N).        │
│                                                                 │
│                                                                 │
│  Finding user_id BETWEEN 100 AND 200:                           │
│    Navigate to leaf with 100, then follow the linked list      │
│    through leaves until you pass 200. Sequential and fast.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What operations B-tree supports:**

```sql
-- ✅ All of these use the B-tree index efficiently:

-- Equality
SELECT * FROM demo_tasks WHERE user_id = 42;

-- Range
SELECT * FROM demo_tasks WHERE user_id > 500;
SELECT * FROM demo_tasks WHERE user_id BETWEEN 100 AND 200;

-- Comparison
SELECT * FROM demo_tasks WHERE user_id >= 42;
SELECT * FROM demo_tasks WHERE user_id <= 100;

-- Sorting (tree is already sorted!)
SELECT * FROM demo_tasks ORDER BY user_id LIMIT 10;

-- MIN/MAX (first/last leaf node)
SELECT MIN(user_id) FROM demo_tasks;
SELECT MAX(user_id) FROM demo_tasks;

-- IN (multiple equality lookups)
SELECT * FROM demo_tasks WHERE user_id IN (42, 99, 123);

-- LIKE with prefix (sorted, so prefix is a range)
CREATE INDEX idx_title ON demo_tasks(title);
SELECT * FROM demo_tasks WHERE title LIKE 'Task #42%';


-- ❌ These CANNOT use a B-tree efficiently:

-- Not equals (must scan almost everything)
SELECT * FROM demo_tasks WHERE user_id != 42;

-- LIKE with leading wildcard (no prefix to anchor the range)
SELECT * FROM demo_tasks WHERE title LIKE '%bug%';

-- Function on the indexed column (index stores raw values, not computed)
SELECT * FROM demo_tasks WHERE LOWER(title) = 'task #42';
-- (Expression indexes solve this — Part 4)
```

> "B-tree is the Swiss Army knife. When in doubt, B-tree. It handles equality, range, sorting, MIN, MAX, and prefix matching. You need a strong reason to choose something else."

---

## 3.3 Hash — Equality and Nothing Else

```sql
CREATE INDEX idx_status_hash ON demo_tasks USING hash(status);
```

```
┌─────────────────────────────────────────────────────────────────┐
│                      HASH INDEX                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A hash index computes a hash of the key value and stores       │
│  rows in hash buckets.                                          │
│                                                                 │
│     hash('pending')    = 0x3A → bucket 3A → [row2, row7, ...]  │
│     hash('completed')  = 0x7F → bucket 7F → [row1, row4, ...]  │
│     hash('in_progress')= 0x12 → bucket 12 → [row3, row8, ...]  │
│                                                                 │
│  ✅ Supports:           ❌ Does NOT support:                     │
│     WHERE x = 'value'     WHERE x > 'value'  (no ordering)     │
│                            WHERE x BETWEEN ... (no ordering)    │
│                            ORDER BY x         (no ordering)     │
│                            WHERE x IS NULL                      │
│                                                                 │
│  When to use Hash over B-tree:                                  │
│  Almost never. B-tree handles equality AND everything else.     │
│  Hash can be slightly smaller and faster for pure equality      │
│  on very large tables — but the difference is marginal.         │
│                                                                 │
│  In practice: just use B-tree.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```sql
-- Hash works for equality:
EXPLAIN SELECT * FROM demo_tasks WHERE status = 'pending';

-- May show (if planner chooses it):
-- Bitmap Heap Scan on demo_tasks
--   -> Bitmap Index Scan on idx_status_hash
--        Index Cond: (status = 'pending')

-- Hash CANNOT help with range:
EXPLAIN SELECT * FROM demo_tasks WHERE status > 'completed';
-- Will use Seq Scan — hash has no concept of "greater than"
```

> "I'm showing you Hash indexes because you'll see them mentioned in documentation and interviews. In practice, B-tree is almost always the better choice."

---

## 3.4 GIN — The Inside-Out Index

**GIN (Generalized Inverted Index) is built for values that contain multiple elements.**

> "In Week 5, you learned about JSONB columns and array types. You may have noticed that searching inside them was slow. GIN is the solution."

```
┌─────────────────────────────────────────────────────────────────┐
│                       GIN INDEX                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  B-tree: one key → one row                                      │
│  GIN:    one key → MANY rows  (inverted mapping)                │
│                                                                 │
│  Think of it as the INDEX AT THE BACK OF A TEXTBOOK:            │
│                                                                 │
│  Regular (B-tree):              Inverted (GIN):                 │
│  "Page 5 mentions: Python,     "Python" → pages 5, 12, 89, 201│
│   async, decorators"           "async"  → pages 5, 45, 89      │
│  "Page 12 mentions: Python,    "decorators" → pages 5, 67      │
│   FastAPI, routes"             "FastAPI" → pages 12, 45         │
│                                "routes"  → pages 12, 89        │
│                                                                 │
│  A textbook index maps:  term → [list of pages]                 │
│  A GIN index maps:       key  → [list of rows]                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**GIN for JSONB (your metadata column):**

```sql
-- Create GIN index on the JSONB column
CREATE INDEX idx_metadata_gin ON demo_tasks USING gin(metadata);
```

```sql
-- Without GIN (slow):
EXPLAIN ANALYZE
SELECT * FROM demo_tasks
WHERE metadata @> '{"source": "api"}';
```

```
 Seq Scan on demo_tasks
   (cost=0.00..27334.00 rows=1000 width=169)
   (actual time=0.019..185.223 rows=250341 loops=1)
   Filter: (metadata @> '{"source": "api"}'::jsonb)
   Rows Removed by Filter: 749659
 Execution Time: 198.445 ms
```

```sql
-- With GIN (fast):
EXPLAIN ANALYZE
SELECT * FROM demo_tasks
WHERE metadata @> '{"source": "api"}';
```

```
 Bitmap Heap Scan on demo_tasks
   (cost=286.25..15432.10 rows=250000 width=169)
   (actual time=42.113..98.776 rows=250341 loops=1)
   Recheck Cond: (metadata @> '{"source": "api"}'::jsonb)
   ->  Bitmap Index Scan on idx_metadata_gin
         (cost=0.00..223.75 rows=250000 width=0)
         (actual time=38.221..38.221 rows=250341 loops=1)
         Index Cond: (metadata @> '{"source": "api"}'::jsonb)
 Execution Time: 112.554 ms
```

> "The improvement is less dramatic here because 25% of rows match. But for selective queries on JSONB keys, GIN is the difference between Seq Scan and instant results."

**GIN for arrays (your tags column):**

```sql
-- Create GIN index on the array column
CREATE INDEX idx_tags_gin ON demo_tasks USING gin(tags);
```

```sql
-- "Find all tasks tagged 'urgent'"
EXPLAIN ANALYZE
SELECT * FROM demo_tasks WHERE tags @> ARRAY['urgent'];
```

```
 Bitmap Heap Scan on demo_tasks
   (cost=212.15..12543.20 rows=200000 width=169)
   (actual time=28.102..71.445 rows=199876 loops=1)
   Recheck Cond: (tags @> '{urgent}'::text[])
   ->  Bitmap Index Scan on idx_tags_gin
         (cost=0.00..162.15 rows=200000 width=0)
         (actual time=24.331..24.331 rows=199876 loops=1)
         Index Cond: (tags @> '{urgent}'::text[])
 Execution Time: 83.221 ms
```

**GIN index internals for the tags column:**

```
┌─────────────────────────────────────────────────────────────────┐
│              GIN INTERNALS FOR tags[]                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The GIN index "explodes" each array into its elements:         │
│                                                                 │
│  Row 1: tags = ['urgent', 'bug']                                │
│  Row 2: tags = ['feature']                                      │
│  Row 3: tags = ['urgent', 'feature']                            │
│  Row 4: tags = ['docs']                                         │
│                                                                 │
│  GIN builds:                                                    │
│                                                                 │
│    'bug'     → [row 1]                                          │
│    'docs'    → [row 4]                                          │
│    'feature' → [row 2, row 3]                                   │
│    'urgent'  → [row 1, row 3]                                   │
│                                                                 │
│  Query: WHERE tags @> ARRAY['urgent']                           │
│    → Look up 'urgent' → [row 1, row 3] → done!                 │
│                                                                 │
│  Query: WHERE tags @> ARRAY['urgent', 'feature']                │
│    → Look up 'urgent'  → [row 1, row 3]                         │
│    → Look up 'feature' → [row 2, row 3]                         │
│    → Intersect          → [row 3]  → done!                      │
│                                                                 │
│  Without GIN: scan ALL rows, unnest each array, check.          │
│  With GIN: direct lookup by element. Massive difference.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**GIN for full-text search** (callback to Week 5 Lecture 3):

```sql
-- Add a tsvector column and index it
ALTER TABLE demo_tasks ADD COLUMN search_vector tsvector;
UPDATE demo_tasks SET search_vector = to_tsvector('english', title || ' ' || description);
CREATE INDEX idx_search_gin ON demo_tasks USING gin(search_vector);

-- Fast full-text search:
EXPLAIN ANALYZE
SELECT * FROM demo_tasks
WHERE search_vector @@ to_tsquery('english', 'task & description');
```

---

## 3.5 GiST — The Swiss Army Knife

**GiST (Generalized Search Tree) handles types that don't fit neatly into B-tree or GIN.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      GiST INDEX                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GiST works on "lossy" representations — it approximates       │
│  whether a match exists, then rechecks.                         │
│                                                                 │
│  Library analogy: A floor map showing which AREA of the         │
│  library has cooking books. You walk to that area, then         │
│  search within it. Not exact, but much faster than walking      │
│  the entire library.                                            │
│                                                                 │
│                                                                 │
│  Best for:                                                      │
│  ├─ Range types (tsrange, daterange, int4range)                 │
│  ├─ Geometric data (points, boxes, polygons)                    │
│  ├─ Full-text search (tsvector — alternative to GIN)            │
│  └─ Network addresses (inet, cidr)                              │
│                                                                 │
│                                                                 │
│  GiST vs GIN for full-text search:                              │
│  ├─ GIN:  Faster reads, slower writes, larger index             │
│  └─ GiST: Faster writes, slower reads, smaller index            │
│           Use GIN unless writes are your bottleneck.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**GiST for range queries:**

```sql
-- Example: tasks with a scheduled date range
-- (not in our demo table, but illustrative)

CREATE TABLE bookings (
    id SERIAL PRIMARY KEY,
    room TEXT NOT NULL,
    during TSTZRANGE NOT NULL  -- timestamp range
);

-- GiST excels at "overlaps" and "contains" for ranges
CREATE INDEX idx_bookings_during ON bookings USING gist(during);

-- "Find all bookings that overlap with this time period"
SELECT * FROM bookings
WHERE during && tstzrange('2025-03-01', '2025-03-15');
-- The && operator means "overlaps" — GiST handles this efficiently.
-- B-tree CANNOT do this.
```

> "Most of you won't need GiST in your Task Manager project. But when you encounter range types, geometric data, or need to choose between GIN and GiST for full-text search, you'll know what it's for."

---

## 3.6 Choosing the Right Index Type

```
┌─────────────────────────────────────────────────────────────────┐
│                INDEX TYPE DECISION FRAMEWORK                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                     What are you indexing?                       │
│                            │                                    │
│            ┌───────────────┼───────────────┐                    │
│            │               │               │                    │
│            ▼               ▼               ▼                    │
│      Scalar values    Composite     Multi-valued                │
│    (int, text, date,  types          types                      │
│     uuid, bool)      (JSONB,        (JSONB, arrays,             │
│            │          ranges)        tsvector)                   │
│            │               │               │                    │
│            ▼               │         ┌─────┴─────┐              │
│        B-tree ✅           │         │           │              │
│     (default, handles      │         ▼           ▼              │
│      =, <, >, range,       │        GIN        GiST             │
│      sort, prefix)         │    (fast reads   (fast writes      │
│            │               │     @>, @@,       ranges,          │
│            │               │     ? operators)  geometric)       │
│            │               │                                    │
│            │               ▼                                    │
│            │          GiST for ranges                           │
│            │          GIN for JSONB @>                           │
│            │          B-tree for JSONB                           │
│            │            specific key extraction                  │
│            │                                                    │
│            ▼                                                    │
│     Need ONLY equality?                                         │
│     Hash exists but B-tree is almost always as good.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Quick reference:**

```
┌────────────┬──────────────────────────────────────────────────┐
│ Index Type │ Use When                                         │
├────────────┼──────────────────────────────────────────────────┤
│ B-tree     │ =, <, >, <=, >=, BETWEEN, IN, IS NULL,         │
│ (default)  │ LIKE 'prefix%', ORDER BY, MIN/MAX               │
├────────────┼──────────────────────────────────────────────────┤
│ Hash       │ = only (rarely better than B-tree)               │
├────────────┼──────────────────────────────────────────────────┤
│ GIN        │ JSONB @>, ?  ;  Array @>, &&                     │
│            │ Full-text @@  ;  trigram similarity               │
├────────────┼──────────────────────────────────────────────────┤
│ GiST       │ Range &&, @>  ;  Geometric operators             │
│            │ Full-text (write-heavy)  ;  Nearest-neighbor     │
└────────────┴──────────────────────────────────────────────────┘
```

---

# PART 4: ADVANCED INDEX STRATEGIES

## 4.1 Composite Indexes (Column Order Is Everything)

**A composite index is an index on multiple columns.**

```sql
CREATE INDEX idx_status_priority ON demo_tasks(status, priority);
```

**How it's organized (think: phonebook sorted by last name, then first name):**

```
┌─────────────────────────────────────────────────────────────────┐
│              COMPOSITE INDEX STRUCTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Index on (status, priority):                                   │
│                                                                 │
│  Sorted by status FIRST, then priority WITHIN each status:      │
│                                                                 │
│     ('cancelled', 0) → row 55                                   │
│     ('cancelled', 1) → row 23                                   │
│     ('cancelled', 2) → row 87                                   │
│     ('cancelled', 3) → row 12                                   │
│     ('completed', 0) → row 44                                   │
│     ('completed', 0) → row 91   ← same (status, priority)      │
│     ('completed', 1) → row 36                                   │
│     ...                                                         │
│     ('in_progress', 0) → row 7                                  │
│     ('in_progress', 1) → row 62                                 │
│     ...                                                         │
│     ('pending', 0) → row 3                                      │
│     ('pending', 1) → row 18                                     │
│     ('pending', 2) → row 5                                      │
│     ...                                                         │
│                                                                 │
│  Like a phonebook: entries sorted by (LastName, FirstName).     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The leftmost prefix rule — the most important rule of composite indexes:**

```
┌─────────────────────────────────────────────────────────────────┐
│                LEFTMOST PREFIX RULE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Index on (status, priority):                                   │
│                                                                 │
│  ✅ CAN use the index:                                          │
│                                                                 │
│  WHERE status = 'pending'                                       │
│    → Left column matches. Navigate tree by status.              │
│                                                                 │
│  WHERE status = 'pending' AND priority = 3                      │
│    → Both columns match. Navigate by status, then by priority.  │
│                                                                 │
│  WHERE status = 'pending' AND priority > 3                      │
│    → Left column equality + right column range. Works!          │
│                                                                 │
│  WHERE status IN ('pending', 'in_progress') AND priority = 3   │
│    → Multiple left values + right equality. Works!              │
│                                                                 │
│                                                                 │
│  ❌ CANNOT use the index efficiently:                           │
│                                                                 │
│  WHERE priority = 3                                             │
│    → SKIPPED the left column! The index is sorted by status     │
│      first. Priority=3 entries are scattered throughout.        │
│                                                                 │
│  WHERE priority > 3 AND status = 'pending'                      │
│    → Same columns, but PostgreSQL CAN reorder this: it uses    │
│      status= 'pending' from the index, then filters priority.  │
│      (The planner is smart about AND conditions.)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Phonebook analogy for the leftmost prefix rule:**

> "A phonebook is sorted by (LastName, FirstName). You can quickly find all people named 'Smith' (leftmost). You can quickly find 'Smith, John' (both). But you CANNOT quickly find all people named 'John' — Johnsons, Smiths, Williams all have Johns scattered throughout. You'd have to read the entire phonebook. Same with composite indexes."

**Demonstrate:**

```sql
-- Uses the index (status is leftmost):
EXPLAIN SELECT * FROM demo_tasks WHERE status = 'pending';
```

```
 Index Scan using idx_status_priority on demo_tasks
   (cost=0.42..8923.43 rows=250422 width=169)
   Index Cond: (status = 'pending'::text)
```

```sql
-- Uses the index (both columns):
EXPLAIN SELECT * FROM demo_tasks
WHERE status = 'pending' AND priority = 3;
```

```
 Index Scan using idx_status_priority on demo_tasks
   (cost=0.42..1823.43 rows=50102 width=169)
   Index Cond: ((status = 'pending'::text) AND (priority = 3))
```

```sql
-- CANNOT use the index efficiently (leftmost column missing):
EXPLAIN SELECT * FROM demo_tasks WHERE priority = 3;
```

```
 Seq Scan on demo_tasks
   (cost=0.00..20834.00 rows=200205 width=169)
   Filter: (priority = 3)
```

> "Seq Scan! Even though we have an index that INCLUDES priority, the planner can't use it because the leftmost column (status) isn't in the WHERE clause."

**Column order guideline:**

```
┌─────────────────────────────────────────────────────────────────┐
│           HOW TO ORDER COMPOSITE INDEX COLUMNS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Rule of thumb:                                                 │
│                                                                 │
│  1. Equality conditions first (WHERE status = 'pending')        │
│  2. Range conditions last (WHERE priority > 3)                  │
│  3. Most selective column first (if both are equality)          │
│                                                                 │
│                                                                 │
│  Example: You often query:                                      │
│    WHERE user_id = ? AND status = ? AND created_at > ?          │
│                                                                 │
│  Best index: (user_id, status, created_at)                      │
│                                                                 │
│    user_id = ?       ← equality, high selectivity → first       │
│    status = ?        ← equality, lower selectivity → second     │
│    created_at > ?    ← range → last                             │
│                                                                 │
│  The range condition should be last because once you enter      │
│  a range scan in the B-tree, subsequent columns can't use      │
│  the index for navigation.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Partial Indexes (Index Only What Matters)

**A partial index only includes rows that match a condition.**

```sql
-- Only index tasks that are NOT completed or cancelled
CREATE INDEX idx_active_tasks ON demo_tasks(user_id)
WHERE status IN ('pending', 'in_progress');
```

**Why this is powerful:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   PARTIAL INDEXES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Library analogy: Instead of a catalog for ALL 100,000 books,   │
│  keep a small catalog of just the 500 "Currently Available"     │
│  books. Faster to search, takes less space.                     │
│                                                                 │
│                                                                 │
│  Full index on user_id:         Partial index on user_id        │
│  (1,000,000 entries)            WHERE status IN                 │
│                                 ('pending', 'in_progress'):     │
│  ┌────────────────────┐         (500,000 entries)               │
│  │ ██████████████████ │                                         │
│  │ ██████████████████ │         ┌────────────────────┐          │
│  │ ██████████████████ │         │ █████████          │          │
│  │ ██████████████████ │         │ █████████          │          │
│  │ ██████████████████ │         └────────────────────┘          │
│  └────────────────────┘                                         │
│  Size: ~21 MB                   Size: ~10 MB                    │
│                                                                 │
│                                                                 │
│  Perfect when:                                                  │
│  ├─ You almost always query a subset of the data                │
│  │   (active tasks, recent orders, unread messages)             │
│  ├─ The subset is significantly smaller than the full table     │
│  └─ You want to save disk space and memory                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The query must match the partial condition:**

```sql
-- ✅ Uses the partial index (WHERE matches):
EXPLAIN
SELECT * FROM demo_tasks
WHERE user_id = 42 AND status = 'pending';
```

```
 Index Scan using idx_active_tasks on demo_tasks
   (cost=0.42..215.43 rows=251 width=169)
   Index Cond: (user_id = 42)
   Filter: (status = 'pending'::text)
```

```sql
-- ❌ Cannot use the partial index (status = 'completed' not in index):
EXPLAIN
SELECT * FROM demo_tasks
WHERE user_id = 42 AND status = 'completed';
```

```
 Index Scan using idx_demo_tasks_user_id on demo_tasks
   (cost=0.42..1025.43 rows=251 width=169)
   Index Cond: (user_id = 42)
   Filter: (status = 'completed'::text)
```

> "Notice it fell back to the full `idx_demo_tasks_user_id` index. The planner knows the partial index only covers active tasks."

**A common real-world pattern — unique email among active users:**

```sql
-- Soft-deleted users shouldn't block new registrations
CREATE UNIQUE INDEX idx_unique_active_email
ON users(email)
WHERE deleted_at IS NULL;

-- Two rows CAN have the same email if one is soft-deleted.
-- But two active users CANNOT share an email.
```

---

## 4.3 Expression Indexes (Index Computed Values)

**Sometimes you query by a computed value, not a raw column.**

```sql
-- This query CANNOT use a B-tree index on created_at:
SELECT * FROM demo_tasks
WHERE DATE(created_at) = '2025-01-15';

-- Why? The index stores the full timestamp values.
-- DATE(created_at) is a COMPUTATION. PostgreSQL must evaluate it
-- for every row — so it does a Seq Scan.
```

**The fix: index the expression itself.**

```sql
CREATE INDEX idx_created_date ON demo_tasks (DATE(created_at));
```

```sql
EXPLAIN
SELECT * FROM demo_tasks
WHERE DATE(created_at) = '2025-01-15';
```

```
 Bitmap Heap Scan on demo_tasks
   (cost=52.55..6234.20 rows=2740 width=169)
   Recheck Cond: (date(created_at) = '2025-01-15'::date)
   ->  Bitmap Index Scan on idx_created_date
         (cost=0.00..51.86 rows=2740 width=0)
         Index Cond: (date(created_at) = '2025-01-15'::date)
```

**More expression index examples:**

```sql
-- Case-insensitive search (common for email, usernames)
CREATE INDEX idx_lower_title ON demo_tasks (LOWER(title));

-- Now this is fast:
SELECT * FROM demo_tasks WHERE LOWER(title) = 'task #42';

-- Without the expression index, LOWER(title) forces a Seq Scan
-- even if title has a B-tree index.
```

```
┌─────────────────────────────────────────────────────────────────┐
│              EXPRESSION INDEX RULE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The expression in the WHERE clause must EXACTLY MATCH the      │
│  expression in the index.                                       │
│                                                                 │
│  Index on:  LOWER(title)                                        │
│                                                                 │
│  ✅ WHERE LOWER(title) = 'task'     ← exact match              │
│  ❌ WHERE UPPER(title) = 'TASK'     ← different function!      │
│  ❌ WHERE title = 'task'            ← no function              │
│                                                                 │
│  Library analogy: You built a catalog organized by              │
│  "publication decade." Searching by "publication year"          │
│  won't find it — the catalog doesn't work that way.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Index-Only Scans (The Holy Grail)

**If all the data you need is IN the index, PostgreSQL never touches the table.**

```sql
-- Create an index that COVERS this common query:
CREATE INDEX idx_covering_status
ON demo_tasks(status, user_id, priority);

-- Query that only needs columns in the index:
EXPLAIN ANALYZE
SELECT status, user_id, priority
FROM demo_tasks
WHERE status = 'pending';
```

```
 Index Only Scan using idx_covering_status on demo_tasks
   (cost=0.42..7523.42 rows=250422 width=16)
   (actual time=0.028..32.115 rows=250108 loops=1)
   Index Cond: (status = 'pending'::text)
   Heap Fetches: 0
 Execution Time: 44.897 ms
```

**`Heap Fetches: 0` — that's the magic number.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  INDEX ONLY SCAN                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Normal Index Scan:                                             │
│                                                                 │
│  Index          Table                                           │
│  ┌──────┐      ┌──────────────────┐                             │
│  │key=42├─────▶│ full row data    │  ← must visit table        │
│  └──────┘      └──────────────────┘    to get all columns      │
│                                                                 │
│  Two reads: index + table.                                      │
│                                                                 │
│                                                                 │
│  Index Only Scan:                                               │
│                                                                 │
│  Index (contains all needed columns)                            │
│  ┌──────────────────────────┐                                   │
│  │ status, user_id, priority│  ← everything you need            │
│  │ is right here            │     is right here                 │
│  └──────────────────────────┘                                   │
│                                                                 │
│  Table: [never touched]                                         │
│                                                                 │
│  One read: index only. Up to 2x faster.                         │
│                                                                 │
│                                                                 │
│  Library analogy: The catalog card has title, author, year,     │
│  and page count. If someone asks "How many pages is Book X?"    │
│  you read the catalog card and never walk to the shelf.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When Index Only Scan works:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ALL of these must be true:                                     │
│                                                                 │
│  1. Every column in SELECT is in the index                      │
│  2. Every column in WHERE is in the index                       │
│  3. Every column in ORDER BY is in the index                    │
│  4. The visibility map is up to date (VACUUM regularly)         │
│                                                                 │
│  If ANY column is NOT in the index, PostgreSQL falls back       │
│  to a regular Index Scan (which visits the table).              │
│                                                                 │
│  Watch Heap Fetches:                                            │
│  • Heap Fetches: 0      ← Perfect. True index-only.            │
│  • Heap Fetches: 500    ← Some rows needed table visit          │
│                            (recent updates, needs VACUUM).      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```sql
-- ❌ This CANNOT be index-only (title not in the index):
EXPLAIN
SELECT status, user_id, priority, title
FROM demo_tasks
WHERE status = 'pending';

-- Falls back to:
--   Index Scan using idx_covering_status on demo_tasks
--   (needs to visit table to get 'title')
```

> "Don't go creating indexes that include every column. That just duplicates your table. Index-only scans are most valuable for specific, high-frequency queries — like aggregation queries, dashboard counts, or hot API endpoints."

---

## 4.5 The Hidden Cost of Indexes

**Indexes are not free.**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE COST OF INDEXES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WRITE AMPLIFICATION                                            │
│  ───────────────────                                            │
│  Every INSERT must update: the table + EVERY index on it.       │
│  Every UPDATE on an indexed column must update those indexes.   │
│  Every DELETE must update every index.                           │
│                                                                 │
│  5 indexes on a table = 6 writes per INSERT (1 table + 5 idx). │
│                                                                 │
│                                                                 │
│  STORAGE                                                        │
│  ───────                                                        │
│  Each index takes disk space. On a 1M-row table:               │
│                                                                 │
│    B-tree on integer column:  ~21 MB                            │
│    B-tree on text column:     ~40 MB                            │
│    GIN on JSONB column:       ~60 MB                            │
│    Table itself:              ~170 MB                           │
│                                                                 │
│  5 indexes might add 200 MB. That's more than the table!       │
│                                                                 │
│                                                                 │
│  MEMORY PRESSURE                                                │
│  ────────────────                                               │
│  PostgreSQL caches frequently-used data in shared_buffers.      │
│  More indexes = more data competing for cache space.            │
│  An index that's never used wastes memory.                      │
│                                                                 │
│                                                                 │
│  MAINTENANCE                                                    │
│  ───────────                                                    │
│  Indexes can become bloated after many updates/deletes.         │
│  VACUUM helps, but index maintenance is real overhead.          │
│  Alembic migrations must manage index creation/deletion.        │
│                                                                 │
│                                                                 │
│  THE RULE:                                                      │
│  ──────────                                                     │
│  Add indexes to speed up queries you ACTUALLY RUN OFTEN.        │
│  Don't add indexes "just in case."                              │
│  Measure BEFORE and AFTER. Remove unused indexes.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to check for unused indexes:**

```sql
-- Show index usage statistics
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,         -- number of times this index was used
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_stat_user_indexes
WHERE tablename = 'demo_tasks'
ORDER BY idx_scan ASC;
```

```
 schemaname | tablename  |       indexname         | idx_scan |  size
------------+------------+-------------------------+----------+--------
 public     | demo_tasks | idx_status_hash         |        0 | 32 MB
 public     | demo_tasks | idx_created_date        |        1 | 21 MB
 public     | demo_tasks | idx_active_tasks        |        2 | 10 MB
 public     | demo_tasks | idx_demo_tasks_user_id  |      347 | 21 MB
 public     | demo_tasks | demo_tasks_pkey         |     1203 | 21 MB
```

> "`idx_scan = 0` for the hash index. It takes 32 MB but has never been used. That's a candidate for removal."

---

# PART 5: QUERY PROFILING WITH SQLALCHEMY

## 5.1 SQL Logging with echo

> "Everything we've done so far was in raw SQL. But your Task Manager uses SQLAlchemy. Let's see the SQL that your ORM generates."

**The simplest way — `echo=True`:**

```python
from sqlalchemy.ext.asyncio import create_async_engine

# During development, turn on echo to see all SQL
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/mydb",
    echo=True  # ← Logs every SQL statement to stdout
)
```

**What you'll see in the console:**

```
INFO sqlalchemy.engine.Engine SELECT tasks.id, tasks.title, tasks.status,
    tasks.priority, tasks.user_id, tasks.category_id, tasks.metadata,
    tasks.tags, tasks.created_at, tasks.updated_at
FROM tasks
WHERE tasks.user_id = $1::INTEGER
INFO sqlalchemy.engine.Engine [generated in 0.00023s] (42,)
```

**Levels of echo:**

```python
# Show all SQL statements
engine = create_async_engine(url, echo=True)

# Show SQL + connection pool events
engine = create_async_engine(url, echo="debug")

# Production: no logging (default)
engine = create_async_engine(url, echo=False)
```

> "Use `echo=True` during development to catch unexpected queries — especially the N+1 problem you learned about in Week 6. But turn it off in production. The logging overhead is real."

---

## 5.2 Event-Based Query Timing

**`echo=True` shows SQL, but doesn't show timing. Let's add that.**

```python
import time
import logging
from sqlalchemy import event
from sqlalchemy.engine import Engine

logger = logging.getLogger("query_profiler")

@event.listens_for(Engine, "before_cursor_execute")
def before_cursor_execute(
    conn, cursor, statement, parameters, context, executemany
):
    conn.info.setdefault("query_start_time", []).append(time.perf_counter())

@event.listens_for(Engine, "after_cursor_execute")
def after_cursor_execute(
    conn, cursor, statement, parameters, context, executemany
):
    total = time.perf_counter() - conn.info["query_start_time"].pop(-1)
    
    # Log slow queries (threshold: 100ms)
    if total > 0.1:
        logger.warning(
            "SLOW QUERY (%.2fms): %s | params: %s",
            total * 1000,
            statement[:200],  # Truncate long queries
            parameters
        )
    else:
        logger.debug(
            "Query (%.2fms): %s",
            total * 1000,
            statement[:100]
        )
```

**What this gives you:**

```
DEBUG query_profiler: Query (1.24ms): SELECT tasks.id, tasks.title ...
                      WHERE tasks.user_id = $1
DEBUG query_profiler: Query (0.87ms): SELECT categories.id, categories.name ...
                      WHERE categories.id = $1
WARNING query_profiler: SLOW QUERY (142.55ms): SELECT tasks.id, tasks.title ...
                        WHERE tasks.status = $1 | params: ('pending',)
```

> "Now you can spot slow queries without running EXPLAIN manually. The WARNING jumps out in your logs."

---

## 5.3 Slow Query Detection in FastAPI

**Connection to Week 6: Your Task Manager uses async SQLAlchemy with FastAPI.**

**Build a middleware that counts queries per request:**

```python
# middleware/query_counter.py
import time
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response
import logging

logger = logging.getLogger("api_profiler")

class QueryCountMiddleware(BaseHTTPMiddleware):
    """Logs query count and total DB time per request.
    
    Catches endpoints that trigger too many queries (N+1)
    or have slow individual queries.
    """
    
    async def dispatch(self, request: Request, call_next) -> Response:
        # Store counters in request state
        request.state.query_count = 0
        request.state.query_time_ms = 0.0
        
        start = time.perf_counter()
        response = await call_next(request)
        total = (time.perf_counter() - start) * 1000
        
        query_count = request.state.query_count
        query_time = request.state.query_time_ms
        
        # Log warnings for problematic requests
        if query_count > 10:
            logger.warning(
                "N+1 ALERT: %s %s made %d queries (%.1fms DB time)",
                request.method, request.url.path,
                query_count, query_time
            )
        elif query_time > 200:
            logger.warning(
                "SLOW DB: %s %s total %.1fms (%.1fms in DB, %d queries)",
                request.method, request.url.path,
                total, query_time, query_count
            )
        
        # Add headers for debugging
        response.headers["X-Query-Count"] = str(query_count)
        response.headers["X-Query-Time-Ms"] = f"{query_time:.1f}"
        
        return response
```

```python
# main.py
from fastapi import FastAPI
from middleware.query_counter import QueryCountMiddleware

app = FastAPI()
app.add_middleware(QueryCountMiddleware)
```

**What you see in dev:**

```
WARNING api_profiler: N+1 ALERT: GET /tasks made 52 queries (89.3ms DB time)
WARNING api_profiler: SLOW DB: GET /tasks/search total 312.4ms (287.1ms in DB, 1 queries)
```

> "The first line tells you: you're hitting the N+1 problem from Week 6. Add a `joinedload` or `selectinload`. The second tells you: one query is slow. Time to run EXPLAIN ANALYZE."

---

## 5.4 The Optimization Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│             THE QUERY OPTIMIZATION WORKFLOW                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  Step 1: IDENTIFY the slow query                                │
│  ────────────────────────────────                               │
│  • Query profiling middleware spots it                           │
│  • Or: user reports "this page is slow"                         │
│  • Get the actual SQL (from echo=True or logs)                  │
│                                                                 │
│                        │                                        │
│                        ▼                                        │
│                                                                 │
│  Step 2: EXPLAIN ANALYZE the query                              │
│  ─────────────────────────────────                              │
│  • Run in psql with the actual parameters                       │
│  • Look for: Seq Scan, high "Rows Removed by Filter",          │
│    large estimated-vs-actual row mismatch                       │
│                                                                 │
│                        │                                        │
│                        ▼                                        │
│                                                                 │
│  Step 3: DIAGNOSE the problem                                   │
│  ────────────────────────────                                   │
│  • Missing index? → Create one                                  │
│  • Wrong index type? → GIN for JSONB, composite for multi-col  │
│  • Stale statistics? → ANALYZE                                  │
│  • N+1 pattern? → Fix in SQLAlchemy (joinedload/selectinload)  │
│  • Query returns too many rows? → Add pagination (Week 4)       │
│                                                                 │
│                        │                                        │
│                        ▼                                        │
│                                                                 │
│  Step 4: APPLY the fix                                          │
│  ─────────────────────                                          │
│  • Add index via Alembic migration (Week 6)                     │
│  • Or: rewrite the query / fix the ORM call                     │
│                                                                 │
│                        │                                        │
│                        ▼                                        │
│                                                                 │
│  Step 5: VERIFY with EXPLAIN ANALYZE again                      │
│  ─────────────────────────────────────────                      │
│  • Confirm the index is used                                    │
│  • Compare before/after execution time                          │
│  • Document the improvement (for your project!)                 │
│                                                                 │
│                        │                                        │
│                        ▼                                        │
│                                                                 │
│  Step 6: MONITOR in production                                  │
│  ────────────────────────────                                   │
│  • Keep the query profiling middleware running                   │
│  • Watch for regressions after schema changes                   │
│                                                                 │
│                                                                 │
│  NEVER optimize without measuring first.                        │
│  NEVER assume an index helps without EXPLAIN ANALYZE.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              QUERY OPTIMIZATION QUICK REFERENCE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DIAGNOSTIC TOOLS:                                              │
│      EXPLAIN SELECT ...;                  -- plan only          │
│      EXPLAIN ANALYZE SELECT ...;          -- plan + real timing │
│      EXPLAIN (ANALYZE, BUFFERS) SELECT ...;  -- + cache stats  │
│                                                                 │
│  RED FLAGS IN EXPLAIN OUTPUT:                                   │
│      Seq Scan on large table              -- needs index?       │
│      Rows Removed by Filter: <huge>       -- reading too much   │
│      estimated rows ≠ actual rows (10x+)  -- run ANALYZE       │
│      Nested Loop with high loops count    -- missing index?     │
│      Heap Fetches: <high> on Index Only   -- run VACUUM         │
│                                                                 │
│  CREATE INDEX:                                                  │
│      B-tree:     CREATE INDEX idx ON tbl(col);                  │
│      Hash:       CREATE INDEX idx ON tbl USING hash(col);       │
│      GIN:        CREATE INDEX idx ON tbl USING gin(col);        │
│      GiST:       CREATE INDEX idx ON tbl USING gist(col);      │
│      Composite:  CREATE INDEX idx ON tbl(col1, col2);           │
│      Partial:    CREATE INDEX idx ON tbl(col) WHERE cond;       │
│      Expression: CREATE INDEX idx ON tbl(LOWER(col));           │
│                                                                 │
│  INDEX TYPE CHEAT SHEET:                                        │
│      Scalar =, <, >, range, sort  → B-tree (default)           │
│      JSONB @>, ?, ?|, ?&          → GIN                        │
│      Array @>, &&                 → GIN                        │
│      Full-text search @@          → GIN (reads) / GiST (writes)│
│      Range overlaps &&            → GiST                       │
│      Equality only                → B-tree (Hash is rarely better)│
│                                                                 │
│  COMPOSITE INDEX RULES:                                         │
│      Leftmost prefix must be in WHERE                           │
│      Equality columns first, range columns last                 │
│      Most selective equality column first                       │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Index on col, query on FUNCTION(col)  → Expression idx │
│      ❌ Composite (A, B), query on B only     → B not leftmost │
│      ❌ Index everything "just in case"        → Write penalty  │
│      ❌ Optimize without EXPLAIN ANALYZE       → Guessing       │
│      ❌ Forget ANALYZE after bulk data changes → Stale stats    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  QUERY OPTIMIZATION = HELPING POSTGRESQL FIND DATA FASTER       │
│                                                                 │
│  Without indexes, PostgreSQL reads every row in the table.      │
│  With the right index, it jumps directly to matching rows.      │
│                                                                 │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐                │
│  │ Identify │ ──▶ │ Diagnose │ ──▶ │  Fix &   │                │
│  │ (logs,   │     │ (EXPLAIN │     │  Verify  │                │
│  │  profile)│     │  ANALYZE)│     │ (index,  │                │
│  └──────────┘     └──────────┘     │  EXPLAIN)│                │
│                                    └──────────┘                │
│                                                                 │
│  THE LIBRARY ANALOGY:                                           │
│  ├─ Table = Library full of books                               │
│  ├─ Seq Scan = Walking every shelf                              │
│  ├─ Index = Card catalog                                        │
│  ├─ B-tree = Catalog sorted alphabetically (range lookups)      │
│  ├─ GIN = Back-of-textbook index (keyword → pages)              │
│  ├─ Partial Index = Small catalog for "Available Now" books     │
│  ├─ Index Only Scan = Answering from the catalog card alone     │
│  └─ EXPLAIN = Asking the librarian "how would you find this?"   │
│                                                                 │
│                                                                 │
│  THE GOLDEN RULE:                                               │
│                                                                 │
│     MEASURE → EXPLAIN → FIX → VERIFY → MONITOR                 │
│                                                                 │
│     Never guess. Never optimize without data.                   │
│     Always EXPLAIN ANALYZE before and after.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Lectures

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 7, LECTURE 2 (Next):                                      │
│  └─ Advanced database patterns                                  │
│     Connection pooling (pool_size affects how many queries      │
│     can run concurrently — slow queries exhaust the pool)       │
│     Bulk operations (when INSERT performance matters,           │
│     index overhead from 4.5 becomes critical)                   │
│     Optimistic locking (uses indexed columns for versioning)    │
│                                                                 │
│  WEEK 7, LECTURE 3:                                             │
│  └─ NoSQL awareness                                             │
│     When query optimization isn't enough because the data       │
│     model doesn't fit relational. Decision framework for        │
│     choosing the right database.                                │
│                                                                 │
│  WEEK 7 PROJECT:                                                │
│  └─ You must show at least 2 queries with EXPLAIN output        │
│     demonstrating index usage, with documented before/after     │
│     performance numbers. Today's lecture is your toolkit.       │
│                                                                 │
│  WEEK 10 (Redis):                                               │
│  └─ When even an optimized query isn't fast enough, cache       │
│     the result. But optimize the query FIRST — caching a bad   │
│     query just hides the problem until the cache expires.       │
│                                                                 │
│  WEEK 12 (Performance):                                         │
│  └─ Load testing with locust. Your EXPLAIN skills help you      │
│     diagnose WHY p95 latency is high under load.               │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Production-grade queries with real data volumes.            │
│     Query optimization is not optional in the capstone.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```