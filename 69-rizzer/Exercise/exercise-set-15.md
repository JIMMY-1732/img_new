# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "PostgreSQL features are just convenient shortcuts. JSONB is storing JSON as a string with some helper functions. Arrays work like Python lists. Full-text search is a fancier LIKE. Indexes make everything faster. Transactions are just optional wrappers around SQL statements."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | `->>` returns the original JSON type (number, boolean) | `->>` **always** returns TEXT, even for JSON numbers and booleans |
| 2 | `->` and `->>` are interchangeable in WHERE clauses | `->` returns JSONB (can't compare to text); `->>` returns TEXT (can't do numeric comparison without cast) |
| 3 | The `\|\|` merge operator does deep/recursive merge on nested JSONB | `\|\|` does **shallow** merge — nested objects are **replaced entirely**, not merged |
| 4 | PostgreSQL arrays are 0-indexed like Python and JSONB | PostgreSQL arrays are **1-indexed**; `arr[0]` returns NULL, not the first element |
| 5 | LIKE with wildcards is "good enough" for search | LIKE has no stemming, no ranking, cannot use standard indexes, and scans every row |
| 6 | Concatenating columns for full-text search is safe | If ANY concatenated column is NULL, the **entire expression** becomes NULL — rows vanish silently |
| 7 | Adding an index always speeds up the matching query | The index expression must **exactly match** the query expression; different function arguments = index ignored |
| 8 | After an error inside a transaction, you can continue with other statements | The transaction enters an **aborted state**; ALL subsequent statements are rejected until ROLLBACK |
| 9 | COMMIT after a transaction error saves the statements that succeeded | COMMIT on an aborted transaction acts as **ROLLBACK** — nothing is saved |
| 10 | EXPLAIN cost numbers are in milliseconds | Cost is in **arbitrary planner units**, not time; only EXPLAIN ANALYZE shows real milliseconds |

---

## The Main Purpose (Essence)

**Core Insight:** PostgreSQL-specific features solve problems where standard SQL hits a wall. Each feature exists because the "obvious" SQL approach either **fails outright**, **performs terribly**, or **cannot guarantee data integrity**. The features are not optional conveniences — they are solutions to specific, painful failure modes.

**The Problems Being Solved:**

| Feature | Without It (Standard SQL) | With It (PostgreSQL Extension) |
|---------|--------------------------|-------------------------------|
| JSONB | Column explosion: 12+ nullable columns, most rows 90% NULL | One flexible column, each row stores only what it needs |
| Arrays | Extra junction table for simple string lists | Simple lists without schema overhead |
| Full-Text Search | LIKE scans every row, no stemming, no ranking | Language-aware, indexable, ranked search |
| EXPLAIN + Indexes | Debugging performance blind; queries mysteriously slow | X-ray of query execution; targeted speedups |
| Transactions | Partial updates corrupt data; no atomicity guarantee | All-or-nothing operations; data integrity guaranteed |

**The Key Distinctions:**
- `->` gives you **JSONB** (for chaining deeper). `->>` gives you **TEXT** (for comparing and displaying). These are not interchangeable.
- PostgreSQL arrays are **1-indexed**. JSONB arrays are **0-indexed**. In the same query, the same number means different positions.
- LIKE is **substring matching**. Full-text search is **language understanding**. "authenticate" and "authentication" are the same word to FTS but completely different strings to LIKE.
- EXPLAIN shows the **plan** (estimates). EXPLAIN ANALYZE shows **reality** (actual execution). The plan can be wrong.
- A transaction error doesn't "skip" the bad statement — it **kills the entire transaction**.

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "`->>` preserves numeric types" | `metadata->>'priority' > '5'` returns priorities 7, 10, 25, 100 | Returns ONLY priority 7 | `->>` returns TEXT; `'10' < '5'` in text because `'1' < '5'` |
| "Arrays are 0-indexed like Python" | `tags[0]` returns first tag | Returns NULL | PostgreSQL arrays are 1-indexed; index 0 is out of bounds |
| "String concat is safe for search" | `title \|\| ' ' \|\| description` always produces a searchable string | Returns NULL when description is NULL | In SQL, `anything \|\| NULL = NULL` |
| "`\|\|` merges nested JSONB recursively" | Merging `{"env": {"browser": "Firefox"}}` updates only the browser | Replaces the **entire** `env` object, losing all other nested keys | `\|\|` is shallow; it replaces top-level keys, not nested values |
| "COMMIT saves whatever succeeded" | After error, COMMIT saves pre-error UPDATEs | COMMIT acts as ROLLBACK; ALL changes lost | Transaction is in aborted state; COMMIT = ROLLBACK |
| "Index on column speeds up any query on that column" | FTS index on `to_tsvector('english', col)` speeds up `to_tsvector('simple', col)` | Seq Scan; index completely ignored | Index expression must match query expression exactly |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Priority Paradox (JSONB ->> Type Trap)

**Setup** (run after lecture demo):

```sql
-- Add numeric priorities to task metadata
UPDATE tasks SET metadata = jsonb_set(COALESCE(metadata, '{}'), '{priority}', '3')   WHERE id = 1;
UPDATE tasks SET metadata = jsonb_set(COALESCE(metadata, '{}'), '{priority}', '10')  WHERE id = 2;
UPDATE tasks SET metadata = jsonb_set(COALESCE(metadata, '{}'), '{priority}', '7')   WHERE id = 3;
UPDATE tasks SET metadata = jsonb_set(COALESCE(metadata, '{}'), '{priority}', '25')  WHERE id = 6;
UPDATE tasks SET metadata = jsonb_set(COALESCE(metadata, '{}'), '{priority}', '100') WHERE id = 7;
```

**The Query:**

```sql
-- "Show me all high-priority tasks (priority above 5), highest first"
SELECT id, title, metadata->>'priority' AS priority
FROM tasks
WHERE metadata ? 'priority'
  AND metadata->>'priority' > '5'
ORDER BY metadata->>'priority' DESC;
```

**Questions:**

1. **Predict the output.** Specifically:
   - Which task IDs appear in the result?
   - What is the sort order?
   - A junior developer expected tasks 2, 3, 6, and 7 (priorities 10, 7, 25, 100) to all appear, sorted as 100, 25, 10, 7. Are they correct?

2. **Explain why.** The `->>` operator extracts the JSON number `10` as the text string `'10'`. Walk through the comparison `'10' > '5'` as PostgreSQL performs it. What rule does text comparison follow? Why does this make `'100'` sort before `'25'`?

3. **Fix the query.** Write a corrected version that performs true numeric comparison and produces the expected result of priorities 100, 25, 10, 7 in descending order.

---

## Exercise 1.2: The Off-By-One Database (Array vs JSONB Indexing)

**Setup:** Task id=4 has been modified during the lecture. Verify its state:

```sql
-- From lecture: task 4 has these values
-- tags (PostgreSQL TEXT[]): {'bug', 'frontend', 'urgent'}
-- metadata (JSONB) includes:
--   "steps_to_reproduce": ["Open the login page", "Leave password field empty",
--                          "Click Submit", "App crashes with 500 error"]
```

**The Query:**

```sql
SELECT
    title,
    -- Attempt to get the first tag
    tags[0]                                AS tag_at_0,
    tags[1]                                AS tag_at_1,
    -- Attempt to get the first step
    metadata->'steps_to_reproduce'->>0     AS step_at_0,
    metadata->'steps_to_reproduce'->>1     AS step_at_1
FROM tasks
WHERE id = 4;
```

**Questions:**

1. **Predict the output** for all four extracted values (`tag_at_0`, `tag_at_1`, `step_at_0`, `step_at_1`). Be precise about which is NULL and which has a value.

2. **Explain the inconsistency.** Why do PostgreSQL arrays and JSONB arrays use different indexing? A student says: "This is insane — index 0 means different things in the same query." What concrete advice would you give them to avoid bugs?

3. **Write a general rule.** Complete this sentence: "When accessing elements, PostgreSQL TEXT[] arrays start at index ___, while JSONB arrays accessed via `->` start at index ___. If you get NULL from an array access, the first thing to check is ___."

---

## Exercise 1.3: The Invisible Tasks (NULL Propagation in Full-Text Search)

**Setup:**

```sql
-- Insert tasks with varying description completeness
INSERT INTO tasks (id, project_id, title, description, status, assigned_to) VALUES
    (500001, 1, 'Fix database connection pooling', NULL, 'todo', 1),
    (500002, 1, 'Database migration for user preferences',
     'Create Alembic migration to add a preferences JSONB column to the users table', 'todo', 2),
    (500003, 1, 'Optimize database query performance', NULL, 'in_progress', 3);
```

**The Query:**

```sql
-- "Find all tasks related to 'database'"
SELECT id, title
FROM tasks
WHERE to_tsvector('english', title || ' ' || description)
      @@ to_tsquery('english', 'database')
  AND id IN (500001, 500002, 500003)
ORDER BY id;
```

**Questions:**

1. **Predict the output.** All three tasks have the word "database" in their title. How many rows does this query return? Which specific IDs appear?

2. **Trace the failure.** For task 500001, evaluate the expression `title || ' ' || description` step by step. What is the result of `'Fix database connection pooling' || ' ' || NULL`? What does `to_tsvector('english', NULL)` return? What does `NULL @@ to_tsquery(...)` evaluate to in a WHERE clause?

3. **Fix the query** so that tasks with NULL descriptions are still found when the search term appears in their title. The fix should be minimal — do not restructure the entire query.

---

## Exercise 1.4: The Optimistic Recovery (Transaction Error State)

**Setup:** Verify starting state:

```sql
SELECT id, status FROM tasks WHERE id IN (1, 2) ORDER BY id;
-- Assume: id=1 status='done', id=2 status='in_progress'
```

**The Script:**

```sql
BEGIN;

-- Step 1: Mark task 1 as 'blocked'
UPDATE tasks SET status = 'blocked' WHERE id = 1;

-- Step 2: Accidentally insert a task with a duplicate id
INSERT INTO tasks (id, title, project_id) VALUES (1, 'Duplicate task', 1);
-- ERROR: duplicate key value violates unique constraint "tasks_pkey"

-- Step 3: Developer thinks "OK that failed, let me continue"
UPDATE tasks SET status = 'done' WHERE id = 2;

-- Step 4: Save the work
COMMIT;
```

**Then check results:**

```sql
SELECT id, status FROM tasks WHERE id IN (1, 2) ORDER BY id;
```

**Questions:**

1. **Predict the final status** of task 1 and task 2 after the COMMIT. A developer expects: task 1 = 'blocked' (Step 1 succeeded), task 2 = unchanged (Step 3 might fail), or task 2 = 'done' (if Step 3 executes). What actually happens?

2. **What does Step 3 return?** When the developer runs the UPDATE in Step 3 after the error in Step 2, what does PostgreSQL respond with? Does the UPDATE execute?

3. **What does COMMIT do** when the transaction is in an error (aborted) state? Does it save the Step 1 UPDATE that was successful before the error?

4. **Rewrite the script** using SAVEPOINT so that: Step 1 is preserved, the failed INSERT is contained, and Step 3 can execute. Both the Step 1 and Step 3 UPDATEs should be committed.

---

# Level 1 Solutions

## Solution 1.1: The Priority Paradox

**1. Predicted output:**

```
 id |              title               | priority
----+----------------------------------+----------
  3 | Write API test suite             | 7
(1 row)
```

Only task id=3 (priority '7') appears. Tasks with priorities 10, 25, and 100 are **all missing**. The sort order is irrelevant — there's only one row.

**2. Explain why:**

The `->>` operator **always** returns TEXT, regardless of the original JSON type. Even though `10` is stored as a JSON number, `metadata->>'priority'` extracts it as the text string `'10'`.

Text comparison uses **lexicographic (character-by-character) ordering**, not numeric ordering. PostgreSQL compares the first character of each string first:

```
'3'   > '5'  →  '3' < '5'  →  FALSE  (3 < 5 in ASCII)
'10'  > '5'  →  '1' < '5'  →  FALSE  (compares first char only!)
'7'   > '5'  →  '7' > '5'  →  TRUE   ✅
'25'  > '5'  →  '2' < '5'  →  FALSE
'100' > '5'  →  '1' < '5'  →  FALSE
```

The text string `'100'` sorts BEFORE `'5'` because `'1'` comes before `'5'` in the alphabet. This is the same reason `'apple'` comes before `'banana'` — only the first differing character matters.

Similarly, ORDER BY on text would produce: `'7', '3', '25', '100', '10'` — completely nonsensical from a numeric perspective.

**3. The fix — cast to integer:**

```sql
SELECT id, title, (metadata->>'priority')::int AS priority
FROM tasks
WHERE metadata ? 'priority'
  AND (metadata->>'priority')::int > 5
ORDER BY (metadata->>'priority')::int DESC;
```

```
 id |              title                        | priority
----+-------------------------------------------+----------
  7 | <task title>                              |      100
  6 | <task title>                              |       25
  2 | Implement JWT authentication              |       10
  3 | Write API test suite                      |        7
```

**The Key Insight:** `->>` is a TEXT extraction operator, not a "give me the value" operator. For any non-text comparison (numeric, boolean, date), you **must** explicitly cast: `(column->>'key')::int`, `(column->>'key')::boolean`, etc.

---

## Solution 1.2: The Off-By-One Database

**1. Predicted output:**

```
              title               | tag_at_0 | tag_at_1 |       step_at_0        |          step_at_1
----------------------------------+----------+----------+------------------------+----------------------------
 Fix login crash on empty password| NULL     | bug      | Open the login page    | Leave password field empty
```

- `tags[0]` → **NULL** (PostgreSQL arrays are 1-indexed; index 0 is out of bounds)
- `tags[1]` → `'bug'` (the actual first element)
- `metadata->'steps_to_reproduce'->>0` → `'Open the login page'` (JSONB arrays are 0-indexed)
- `metadata->'steps_to_reproduce'->>1` → `'Leave password field empty'` (second element)

**2. The inconsistency explained:**

PostgreSQL arrays follow SQL standard convention (1-indexed), which predates most modern programming languages. JSONB follows the JSON/JavaScript specification (0-indexed). This is a historical artifact — PostgreSQL didn't invent either convention but must support both.

**Advice:** Never assume the index. When you get NULL from an array access, your first instinct should be "Am I using the wrong starting index?" — not "Is the data missing?" Add a comment at the query site: `tags[1] -- PG arrays are 1-indexed` as documentation for yourself and teammates.

**3. The general rule:**

"When accessing elements, PostgreSQL TEXT[] arrays start at index **1**, while JSONB arrays accessed via `->` start at index **0**. If you get NULL from an array access, the first thing to check is **whether you're using the correct indexing convention for that data type**."

---

## Solution 1.3: The Invisible Tasks

**1. Predicted output:**

```
   id   |                  title
--------+------------------------------------------
 500002 | Database migration for user preferences
(1 row)
```

Only **1 row** out of 3. Tasks 500001 and 500003 are invisible even though "database" is in their titles.

**2. Trace the failure for task 500001:**

```
Step 1: title || ' ' || description
      = 'Fix database connection pooling' || ' ' || NULL
      = NULL
      (In SQL, ANY concatenation with NULL produces NULL)

Step 2: to_tsvector('english', NULL)
      = NULL
      (Function on NULL input returns NULL)

Step 3: NULL @@ to_tsquery('english', 'database')
      = NULL
      (Any comparison involving NULL returns NULL, not true/false)

Step 4: WHERE NULL
      = Row excluded (WHERE only includes rows where condition is TRUE)
```

The task is eliminated at Step 1. The NULL description poisons the entire concatenation, making the title invisible to search.

**3. The fix — wrap nullable columns in COALESCE:**

```sql
SELECT id, title
FROM tasks
WHERE to_tsvector('english', title || ' ' || COALESCE(description, ''))
      @@ to_tsquery('english', 'database')
  AND id IN (500001, 500002, 500003)
ORDER BY id;
```

`COALESCE(description, '')` replaces NULL with an empty string, so the concatenation produces `'Fix database connection pooling '` instead of NULL.

**The Key Insight:** In SQL, NULL is not an empty string — it is the **absence of a value**. It propagates through concatenation, function calls, and comparisons like a virus. Any time you combine columns for full-text search, every nullable column **must** be wrapped in COALESCE. This is not optional defensive coding — it is required for correctness.

---

## Solution 1.4: The Optimistic Recovery

**1. Final status of both tasks:** **Unchanged.** Task 1 retains whatever status it had before BEGIN. Task 2 also retains its original status. No modifications from this script persist.

**2. Step 3 returns:**

```
ERROR:  current transaction is aborted, commands ignored until end of transaction block
```

The UPDATE does **not** execute. PostgreSQL refuses to run any statement after an error inside a transaction. The transaction is in a permanently aborted state.

**3. What COMMIT does:**

On an aborted transaction, COMMIT is silently converted to ROLLBACK. PostgreSQL may emit:

```
ROLLBACK
```

or:

```
WARNING:  there is no transaction in progress
```

It does **not** save the Step 1 UPDATE. The Step 1 UPDATE was in the same transaction as the failed INSERT, and transactions are atomic — all or nothing. "All" failed, so "nothing" is saved.

**4. Rewrite with SAVEPOINT:**

```sql
BEGIN;

-- Step 1: Mark task 1 as 'blocked'
UPDATE tasks SET status = 'blocked' WHERE id = 1;

-- Create a safety checkpoint before the risky operation
SAVEPOINT before_insert;

-- Step 2: Attempt the insert (will fail)
INSERT INTO tasks (id, title, project_id) VALUES (1, 'Duplicate task', 1);
-- ERROR: duplicate key value violates unique constraint

-- Roll back only to the savepoint — Step 1 is preserved
ROLLBACK TO SAVEPOINT before_insert;

-- Step 3: Now this executes successfully
UPDATE tasks SET status = 'done' WHERE id = 2;

-- Step 4: Commit everything — Steps 1 and 3 are saved
COMMIT;
```

After this script, task 1 = 'blocked' and task 2 = 'done'.

**The Key Insight:** Without SAVEPOINT, an error inside a transaction is a death sentence for the entire transaction. SAVEPOINT creates "checkpoints" that allow partial rollback without killing the whole transaction. Think of them like exception handling in Python: the SAVEPOINT is the `try`, ROLLBACK TO SAVEPOINT is the `except`, and the code after it is recovery logic.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Merge That Ate Your Data (JSONB Shallow Merge)

**Context:** A developer is building a system to track detailed environment info for bug reports. The code "works" — no errors, correct field updated — but something is silently going wrong.

**Setup:**

```sql
-- Set up detailed bug metadata for task 4
UPDATE tasks SET metadata = '{
    "type": "bug",
    "severity": "critical",
    "environment": {
        "browser": "Chrome 120",
        "os": "Windows 11",
        "screen_resolution": "1920x1080",
        "node_version": "18.17.0"
    },
    "steps_to_reproduce": [
        "Open the login page",
        "Leave password field empty",
        "Click Submit",
        "App crashes with 500 error"
    ]
}'::jsonb
WHERE id = 4;
```

**Verify the starting state:**

```sql
SELECT jsonb_pretty(metadata) FROM tasks WHERE id = 4;
```

**The Bug Code:**

```sql
-- "A user reported this also happens on Firefox. Just update the browser."
UPDATE tasks
SET metadata = metadata || '{"environment": {"browser": "Firefox 121"}}'::jsonb
WHERE id = 4;

-- "Done! Browser updated. Let's verify."
SELECT metadata->>'type' AS type,
       metadata->'environment'->>'browser' AS browser,
       metadata->'environment'->>'os' AS os,
       metadata->'environment'->>'screen_resolution' AS resolution,
       metadata->'environment'->>'node_version' AS node_version
FROM tasks WHERE id = 4;
```

---

**Task 2a: Identify the Flaw**

Run the code. The browser was correctly updated to "Firefox 121". But examine the `os`, `resolution`, and `node_version` columns in the output.

1. What values do they show?
2. What values SHOULD they show?
3. The developer says "I only changed the browser — why did the other fields disappear?"

Do NOT fix the code yet.

---

**Task 2b: Explain the Mechanism**

The `||` (concatenation/merge) operator merges two JSONB objects. But HOW does it merge?

1. Is `||` a deep merge (recursively merges nested objects) or a shallow merge (replaces top-level keys)?
2. What specifically happens when `metadata || '{"environment": {"browser": "Firefox 121"}}'` executes? Walk through what the `||` operator does with the `"environment"` key.
3. How many key-value pairs does the new `"environment"` object have after the merge?

---

**Task 2c: Instrument to Prove Diagnosis**

Write queries that demonstrate the data loss. Show the full metadata before and after the merge:

```sql
-- Before the merge: show the environment object
SELECT jsonb_pretty(metadata->'environment') FROM tasks WHERE id = 4;

-- After the merge: show the environment object
-- (Reset and re-run the bug code first)
SELECT jsonb_pretty(metadata->'environment') FROM tasks WHERE id = 4;

-- Count the keys in the environment object before and after
SELECT jsonb_object_keys(metadata->'environment') FROM tasks WHERE id = 4;
```

---

**Task 2d: Fix the Code**

Rewrite the UPDATE so that ONLY the browser field is changed, while preserving all other fields inside the `"environment"` object. You have two approaches — use the one that is most surgical:

- Approach A: Use `jsonb_set()` to target the specific nested key
- Approach B: Merge at the nested level instead of the top level

Implement both approaches and verify that `os`, `screen_resolution`, and `node_version` are preserved.

---

**Task 2e: Write Verification Test**

Write a query that acts as a "test" — it returns a boolean indicating whether the update was safe:

```sql
-- This query should return TRUE if the fix is correct, FALSE if data was lost
SELECT
    metadata->'environment'->>'browser' = 'Firefox 121' AS browser_updated,
    metadata->'environment'->>'os' = 'Windows 11' AS os_preserved,
    metadata->'environment'->>'screen_resolution' = '1920x1080' AS resolution_preserved,
    metadata->'environment'->>'node_version' = '18.17.0' AS node_preserved,
    -- All four must be TRUE
    (metadata->'environment'->>'browser' = 'Firefox 121'
     AND metadata->'environment'->>'os' IS NOT NULL
     AND metadata->'environment'->>'screen_resolution' IS NOT NULL
     AND metadata->'environment'->>'node_version' IS NOT NULL) AS all_data_intact
FROM tasks WHERE id = 4;
```

Run this against both the broken code and your fixed code. The broken code should show `all_data_intact = FALSE`. Your fix should show `TRUE`.

---

## Exercise 2.2: The Phantom Index (Expression Mismatch)

**Context:** A developer created a full-text search index to speed up task search. The index was created successfully. But EXPLAIN ANALYZE shows the query is still doing a sequential scan on 500,000 rows.

**Setup:**

```sql
-- Developer creates an FTS index on task descriptions
CREATE INDEX idx_tasks_desc_fts
ON tasks USING gin(to_tsvector('english', COALESCE(description, '')));
```

**The Bug Code:**

```sql
-- Query 1: Developer searches for tasks about "authentication"
EXPLAIN ANALYZE
SELECT id, title
FROM tasks
WHERE to_tsvector('english', description) @@ to_tsquery('english', 'authentication');
```

Output shows:

```
 Seq Scan on tasks  (cost=0.00..158241.00 rows=2500 width=44)
                     (actual time=0.042..487.231 rows=... loops=1)
   Filter: (to_tsvector('english'::regconfig, description) @@ ...
 Execution Time: 492.118 ms
```

```sql
-- Query 2: Developer tries searching across title AND description
EXPLAIN ANALYZE
SELECT id, title
FROM tasks
WHERE to_tsvector('english', title || ' ' || COALESCE(description, ''))
      @@ to_tsquery('english', 'authentication');
```

Output also shows Seq Scan.

**The developer says:** "I created a GIN index! Why is PostgreSQL ignoring it?"

---

**Task 2a: Identify the Flaw**

Look carefully at the index definition and the two queries. Why doesn't PostgreSQL use the index for either query?

1. What **exact expression** was the index built on?
2. What expression does Query 1 use in its WHERE clause?
3. What is the difference between `to_tsvector('english', COALESCE(description, ''))` and `to_tsvector('english', description)`?

---

**Task 2b: Explain the Mechanism**

PostgreSQL expression indexes work by pre-computing and storing the result of an expression. The query planner will only use the index if the query's expression **exactly matches** the stored index expression.

1. Why does adding `COALESCE` in the index but not in the query cause a mismatch?
2. Why does Query 2 fail even though it DOES use COALESCE? (Hint: the index is on `description` only, not `title || ' ' || description`.)
3. Would creating a second index on `to_tsvector('english', description)` (without COALESCE) make Query 1 work? What problem would it have with NULL descriptions?

---

**Task 2c: Instrument to Prove Diagnosis**

Write a query that uses the **exact** index expression and verify the index is used:

```sql
EXPLAIN ANALYZE
SELECT id, title
FROM tasks
WHERE to_tsvector('english', COALESCE(description, ''))
      @@ to_tsquery('english', 'authentication');
```

Show the output. Confirm it uses `Bitmap Index Scan` or `Index Scan` on `idx_tasks_desc_fts`.

---

**Task 2d: Fix the Code**

The developer actually wants to search across **both** title and description. Provide a solution:

1. Drop the existing index.
2. Create a new index that covers the actual search expression (title + description, with COALESCE).
3. Write the query that matches the new index expression exactly.
4. Show EXPLAIN ANALYZE output confirming the index is used.

---

**Task 2e: Write Verification Test**

Write a script that verifies the index is being used. The script should:

```sql
-- Step 1: Run EXPLAIN (not ANALYZE) and check for 'Index' in the plan
EXPLAIN
SELECT id, title FROM tasks
WHERE to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(description, ''))
      @@ to_tsquery('english', 'authentication');

-- The output MUST contain 'Bitmap Index Scan' or 'Index Scan'.
-- If it shows 'Seq Scan', the index is not being used and the fix failed.
```

Also write a functional verification that the search returns correct results:

```sql
-- This must return true: the search finds tasks about authentication
SELECT EXISTS (
    SELECT 1 FROM tasks
    WHERE to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(description, ''))
          @@ to_tsquery('english', 'authentication')
) AS search_returns_results;
```

---

# Level 2 Solutions

## Solution 2.1: The Merge That Ate Your Data

**Task 2a:** After the merge, `os`, `resolution`, and `node_version` are all **NULL**. They should be `'Windows 11'`, `'1920x1080'`, and `'18.17.0'`. The entire `environment` object was replaced with `{"browser": "Firefox 121"}` — a single-key object.

**Task 2b:**

1. `||` is a **shallow merge**. It operates only on top-level keys. When two JSONB objects share a top-level key, the right-hand value **completely replaces** the left-hand value.

2. The execution steps:
    ```
    metadata || '{"environment": {"browser": "Firefox 121"}}'
    
    Step 1: The || operator sees the key "environment" exists in BOTH objects.
    Step 2: It replaces the LEFT value with the RIGHT value entirely.
    Step 3: metadata["environment"] was {"browser":"Chrome 120", "os":"Windows 11", ...}
    Step 4: metadata["environment"] is now {"browser":"Firefox 121"}
    Step 5: The os, screen_resolution, node_version keys are GONE.
    ```

3. The new `"environment"` object has **1** key-value pair (only `browser`). The previous 4 key-value pairs were all destroyed.

**Task 2c:**

```sql
-- Before (reset the metadata first):
SELECT jsonb_pretty(metadata->'environment') FROM tasks WHERE id = 4;
```
```json
{
    "browser": "Chrome 120",
    "os": "Windows 11",
    "screen_resolution": "1920x1080",
    "node_version": "18.17.0"
}
```

```sql
-- After the || merge:
SELECT jsonb_pretty(metadata->'environment') FROM tasks WHERE id = 4;
```
```json
{
    "browser": "Firefox 121"
}
```

Three key-value pairs lost. The `jsonb_object_keys` query returns 4 keys before, 1 key after.

**Task 2d:**

**Approach A — `jsonb_set` (recommended for single field updates):**

```sql
UPDATE tasks
SET metadata = jsonb_set(metadata, '{environment, browser}', '"Firefox 121"')
WHERE id = 4;
```

The path `'{environment, browser}'` navigates into the nested `environment` object and changes only the `browser` key. All sibling keys are untouched.

**Approach B — Nested `||` merge (merge at the correct level):**

```sql
UPDATE tasks
SET metadata = jsonb_set(
    metadata,
    '{environment}',
    (metadata->'environment') || '{"browser": "Firefox 121"}'::jsonb
)
WHERE id = 4;
```

This extracts the existing `environment` object, merges the new browser value into it (shallow merge is fine here because all keys at this level are leaf values, not nested objects), then sets the merged result back.

**Task 2e:** Running the verification query against the broken code shows `all_data_intact = FALSE` (os, resolution, and node_version are NULL). Running it against the `jsonb_set` fix shows `all_data_intact = TRUE` with all four fields intact.

**The Key Insight:** `||` is safe for merging **flat** JSONB objects (no nesting). The moment you have nested objects and want to update a nested field, use `jsonb_set` with a path that navigates to the exact key. Think of `||` as "replace this top-level key" and `jsonb_set` as "surgically edit this nested path."

---

## Solution 2.2: The Phantom Index

**Task 2a:**

1. The index is built on: `to_tsvector('english', COALESCE(description, ''))`
2. Query 1 uses: `to_tsvector('english', description)` — **no COALESCE**
3. The difference: `COALESCE(description, '')` converts NULL to `''` before passing to `to_tsvector`. Without it, `to_tsvector('english', NULL)` returns NULL. These are different expressions, so PostgreSQL's planner does not recognize Query 1 as matching the index.

**Task 2b:**

1. PostgreSQL treats the index expression as an opaque formula. `to_tsvector('english', COALESCE(description, ''))` and `to_tsvector('english', description)` are as different to the planner as `x + 1` and `x + 2`. It does not reason about whether they produce the same results.

2. Query 2 fails because the index covers `COALESCE(description, '')` — just the description column. The query uses `title || ' ' || COALESCE(description, '')` — a different expression that includes the title column. The index cannot help.

3. A second index on `to_tsvector('english', description)` would make Query 1 use the index. However, rows where `description IS NULL` would have NULL tsvector entries in the index, and those rows would never match any search query — effectively invisible.

**Task 2c:**

```sql
EXPLAIN ANALYZE
SELECT id, title
FROM tasks
WHERE to_tsvector('english', COALESCE(description, ''))
      @@ to_tsquery('english', 'authentication');
```

Output now shows:

```
 Bitmap Heap Scan on tasks  (cost=52.08..8124.34 rows=2500 width=44)
   Recheck Cond: (to_tsvector('english'::regconfig, COALESCE(description, ''::text)) @@ ...)
   ->  Bitmap Index Scan on idx_tasks_desc_fts  (cost=0.00..51.46 rows=2500 width=0)
         Index Cond: (to_tsvector('english'::regconfig, COALESCE(description, ''::text)) @@ ...)
```

The index IS used when the expression matches exactly.

**Task 2d:**

```sql
-- Step 1: Drop the old index
DROP INDEX idx_tasks_desc_fts;

-- Step 2: Create a new index covering title + description
CREATE INDEX idx_tasks_title_desc_fts
ON tasks USING gin(
    to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(description, ''))
);

-- Step 3: Query that matches the index expression EXACTLY
EXPLAIN ANALYZE
SELECT id, title
FROM tasks
WHERE to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(description, ''))
      @@ to_tsquery('english', 'authentication');
```

Output should now show Bitmap Index Scan on `idx_tasks_title_desc_fts`.

**The Key Insight:** An expression index is a contract: "I pre-computed THIS EXACT FORMULA for every row." If your query uses a formula that differs by even one function call, one COALESCE wrapper, or one additional column, the index is invisible. When building expression indexes, decide on the canonical form first, then ensure every query uses that exact form. Many teams define this as a SQL view or a generated column to enforce consistency.

---

# LEVEL 2.5: EXTEND EXISTING CODE (The Integration)

## Exercise 2.5: Building a Multi-Criteria Task Search

**Given:** The following working search system. It searches task titles only and returns results ranked by relevance.

```sql
-- Working code: Basic title search with ranking
-- Prerequisite: Index exists
CREATE INDEX IF NOT EXISTS idx_tasks_title_fts
ON tasks USING gin(to_tsvector('english', COALESCE(title, '')));

-- The working search query
SELECT
    id,
    title,
    status,
    ts_rank(
        to_tsvector('english', COALESCE(title, '')),
        to_tsquery('english', 'database')
    ) AS relevance
FROM tasks
WHERE to_tsvector('english', COALESCE(title, ''))
      @@ to_tsquery('english', 'database')
ORDER BY relevance DESC
LIMIT 10;
```

**Verification queries** (these must continue to return correct results after every extension):

```sql
-- Test A: Basic search returns results
SELECT count(*) > 0 AS basic_search_works
FROM tasks
WHERE to_tsvector('english', COALESCE(title, ''))
      @@ to_tsquery('english', 'database');

-- Test B: Results are ranked (first result has highest relevance)
SELECT relevance >= all_relevances.min_r AS ranking_works
FROM (
    SELECT ts_rank(to_tsvector('english', COALESCE(title, '')),
                   to_tsquery('english', 'database')) AS relevance
    FROM tasks
    WHERE to_tsvector('english', COALESCE(title, ''))
          @@ to_tsquery('english', 'database')
    ORDER BY relevance DESC LIMIT 1
) top,
(
    SELECT MIN(ts_rank(to_tsvector('english', COALESCE(title, '')),
                        to_tsquery('english', 'database'))) AS min_r
    FROM tasks
    WHERE to_tsvector('english', COALESCE(title, ''))
          @@ to_tsquery('english', 'database')
) all_relevances;
```

---

**Extension 1: Expand Search Scope (Moderate)**

Modify the search to cover **both** title and description. Requirements:

- Title matches should rank **higher** than description-only matches. (Hint: `ts_rank` accepts weight vectors, or you can multiply the title rank by a factor.)
- Tasks with NULL descriptions must still appear in results if their title matches.
- Test A and Test B must still pass (title-only matches still found and ranked).

Write the new query AND create an appropriate index to support it.

---

**Extension 2: Add JSONB Metadata Filtering (Advanced)**

Extend your search to allow filtering by task type stored in JSONB metadata. The user wants to search for "database" but only within tasks whose `metadata->>'type'` is `'bug'`.

- The search must still work for tasks that have NO metadata (metadata is NULL or `'{}'`).
- If no type filter is provided, all tasks are searched (backward compatible with Extension 1).
- Write the query as two variants: one with the type filter, one without.

---

**Extension 3: Performance Verification (Expert)**

- Run EXPLAIN ANALYZE on both the filtered and unfiltered search queries.
- If either shows Seq Scan, create the necessary indexes to eliminate it.
- Create a GIN index on the metadata column for the containment operator.
- Show before/after EXPLAIN ANALYZE output proving the index is used.
- Ensure Test A and Test B still pass.

---

# Level 2.5 Solutions

## Solution: Extension 1

```sql
-- Drop old index, create combined index
DROP INDEX IF EXISTS idx_tasks_title_fts;
CREATE INDEX idx_tasks_combined_fts
ON tasks USING gin(
    to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(description, ''))
);

-- Search with title-weighted ranking
SELECT
    id, title, status,
    (ts_rank(to_tsvector('english', COALESCE(title, '')),
             to_tsquery('english', 'database')) * 2.0
     +
     ts_rank(to_tsvector('english', COALESCE(description, '')),
             to_tsquery('english', 'database'))
    ) AS relevance
FROM tasks
WHERE to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(description, ''))
      @@ to_tsquery('english', 'database')
ORDER BY relevance DESC
LIMIT 10;
```

The title rank is multiplied by 2.0, giving title matches double the weight. COALESCE on both columns prevents NULL propagation. The combined index supports the WHERE clause.

## Solution: Extension 2

```sql
-- With type filter (search "database" within bugs only)
SELECT id, title, metadata->>'type' AS task_type
FROM tasks
WHERE to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(description, ''))
      @@ to_tsquery('english', 'database')
  AND metadata @> '{"type": "bug"}'
ORDER BY ts_rank(
    to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(description, '')),
    to_tsquery('english', 'database')
) DESC
LIMIT 10;

-- Without type filter (backward compatible)
-- Same query as Extension 1 — the AND clause is simply omitted.
```

Tasks with NULL metadata won't match `metadata @> '{"type": "bug"}'` (which is correct — they're not bugs). Tasks with `'{}'` metadata also won't match. Both behaviors are correct for this filter.

## Solution: Extension 3

```sql
-- GIN index on metadata for containment queries
CREATE INDEX IF NOT EXISTS idx_tasks_metadata ON tasks USING gin(metadata);

-- Verify combined search + filter uses indexes
EXPLAIN ANALYZE
SELECT id, title FROM tasks
WHERE to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(description, ''))
      @@ to_tsquery('english', 'database')
  AND metadata @> '{"type": "bug"}';
```

The plan should show two Bitmap Index Scans (one for FTS, one for metadata containment) combined with a BitmapAnd node. If metadata has low cardinality for `type`, PostgreSQL might choose to only use the FTS index and filter metadata post-scan — this is acceptable. The key verification is that the FTS index is used (no full Seq Scan on 500K rows).

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS (The Application)

## Exercise 3: Atomic Task Transfer with Audit Trail

**Scenario:** You are building the "Move Task" feature for the Task Manager API. When a user moves a task to a different project, the system must:

1. Update the task's project_id
2. Add a JSONB entry to the task's metadata recording the transfer
3. Log the action in the activity_log table
4. If the target project does not exist, nothing should change

**Strict Constraints:**
- **Atomicity:** All three modifications succeed together or none persist. No partial updates.
- **Auditability:** After a successful transfer, the task's metadata must include a `"transfer_history"` array with the transfer details.
- **Safety:** If the target project_id does not reference an existing project, the transfer must fail cleanly with no side effects.
- **Idempotency safeguard:** If someone accidentally runs the transfer twice, the second run must not corrupt the transfer_history (no duplicate entries).

**Verification Tests (you must make ALL of these pass):**

```sql
-- TEST 1: Successful transfer
-- After moving task 3 from project 1 to project 2:
SELECT
    (SELECT project_id FROM tasks WHERE id = 3) = 2 AS task_moved,
    (SELECT count(*) FROM activity_log WHERE task_id = 3 AND action LIKE '%transfer%') > 0
        AS log_exists,
    (SELECT metadata->'transfer_history' IS NOT NULL FROM tasks WHERE id = 3)
        AS history_recorded
;
-- All three must be TRUE

-- TEST 2: Failed transfer (bad project_id)
-- Attempt to move task 5 to project 99999 (does not exist):
-- After the attempt:
SELECT
    (SELECT project_id FROM tasks WHERE id = 5) AS project_unchanged,
    (SELECT count(*) FROM activity_log
     WHERE task_id = 5 AND action LIKE '%transfer%') = 0 AS no_log_entry,
    (SELECT metadata->'transfer_history' IS NULL
     OR jsonb_array_length(metadata->'transfer_history') = 0
     FROM tasks WHERE id = 5) AS no_history_corruption
;
-- project_unchanged should be original value, other two TRUE

-- TEST 3: Performance — the project existence check must NOT do a sequential scan
EXPLAIN SELECT id FROM projects WHERE id = 2;
-- Must show Index Scan (using projects_pkey), NOT Seq Scan
```

---

**Task 3a: Implement Core (Pass TEST 1)**

Write a transaction that moves task 3 from project 1 to project 2. The transaction must:

1. Verify the target project exists (SELECT before UPDATE)
2. Update the task's project_id
3. Update the task's metadata to include transfer info in a `transfer_history` JSONB array
4. Insert an activity_log entry

Use BEGIN/COMMIT. If any step fails, everything rolls back.

Hint for the metadata update: if `transfer_history` doesn't exist yet, you need to create it. Consider using `jsonb_set` with a default plus array concatenation: `COALESCE(metadata->'transfer_history', '[]'::jsonb) || '[{"new_entry": ...}]'::jsonb`.

---

**Task 3b: Handle Failure Safely (Pass TEST 2)**

Modify your transaction to handle the case where the target project doesn't exist. After the project existence check, if the project is not found, the transaction must ROLLBACK and leave no trace.

You have two approaches:
- **Approach A:** Check first, only proceed if valid. Use a DO block or conditional logic.
- **Approach B:** Use SAVEPOINT to attempt the update and roll back if needed.

Implement **Approach A** and demonstrate that TEST 2 passes.

---

**Task 3c: Prevent Duplicate History Entries**

Add protection so that running the exact same transfer twice (task 3, project 1 → project 2) does not create a duplicate entry in `transfer_history`.

Hint: Before appending to the array, check if an entry with the same `from_project` and `to_project` already exists. One approach: use a `WHERE NOT EXISTS` subquery or check the JSONB array contents with `@>`.

Verify that running your transfer script twice results in only ONE transfer_history entry, not two.

---

# Level 3 Solutions

## Solution 3a: Core Transfer

```sql
BEGIN;

-- Step 1: Verify target project exists
DO $$
DECLARE
    target_exists BOOLEAN;
    old_project_id INTEGER;
BEGIN
    SELECT EXISTS(SELECT 1 FROM projects WHERE id = 2) INTO target_exists;

    IF NOT target_exists THEN
        RAISE EXCEPTION 'Target project does not exist';
    END IF;

    -- Step 2: Capture old project_id for audit
    SELECT project_id INTO old_project_id FROM tasks WHERE id = 3;

    -- Step 3: Update project_id
    UPDATE tasks SET project_id = 2 WHERE id = 3;

    -- Step 4: Add transfer history to metadata
    UPDATE tasks
    SET metadata = jsonb_set(
        COALESCE(metadata, '{}'::jsonb),
        '{transfer_history}',
        COALESCE(metadata->'transfer_history', '[]'::jsonb)
            || jsonb_build_array(jsonb_build_object(
                'from_project', old_project_id,
                'to_project', 2,
                'transferred_at', NOW()::text
            ))
    )
    WHERE id = 3;

    -- Step 5: Log the action
    INSERT INTO activity_log (task_id, action, performed_by)
    VALUES (3, 'transfer: project ' || old_project_id || ' → project 2', 1);
END $$;

COMMIT;
```

This passes TEST 1. The DO block allows procedural logic within the transaction. If the RAISE EXCEPTION fires, the entire transaction aborts — no partial updates.

## Solution 3b: Failure Handling

The DO block's `RAISE EXCEPTION` in 3a already handles this. If `target_exists` is false, the exception aborts the entire transaction. No UPDATE, no metadata change, no activity_log entry. TEST 2 passes because nothing was modified.

For an approach without DO blocks (pure SQL with SAVEPOINT):

```sql
BEGIN;

SAVEPOINT before_transfer;

-- Check if project 99999 exists
-- If this returns 0 rows, we know the project is invalid
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM projects WHERE id = 99999) THEN
        RAISE EXCEPTION 'Target project % does not exist', 99999;
    END IF;
END $$;

-- (Everything below only runs if the project exists)
UPDATE tasks SET project_id = 99999 WHERE id = 5;
-- ... metadata and logging ...

COMMIT;

-- If the exception fired, we catch it and rollback:
-- In psql: the DO block exception aborts the transaction.
-- The final state is: nothing changed. TEST 2 passes.
```

## Solution 3c: Idempotency Protection

Replace the metadata update with a check-then-append:

```sql
-- Only append if this exact transfer is not already recorded
UPDATE tasks
SET metadata = jsonb_set(
    COALESCE(metadata, '{}'::jsonb),
    '{transfer_history}',
    COALESCE(metadata->'transfer_history', '[]'::jsonb)
        || jsonb_build_array(jsonb_build_object(
            'from_project', old_project_id,
            'to_project', 2,
            'transferred_at', NOW()::text
        ))
)
WHERE id = 3
  AND NOT (
      COALESCE(metadata->'transfer_history', '[]'::jsonb)
      @> jsonb_build_array(jsonb_build_object('from_project', old_project_id, 'to_project', 2))
  );
```

The `NOT @>` clause checks whether the transfer_history array already contains an object with matching `from_project` and `to_project`. If it does, the WHERE clause excludes the row and no update happens. This uses the `@>` containment operator — when checking array containment, `@>` performs element-wise matching, where the contained array elements need only match a subset of keys in the containing array's elements.

**The Key Insight:** Idempotency in database operations requires explicit checks. Unlike HTTP PUT (which naturally replaces), JSONB array append always adds. You must build the "already exists?" check into the query itself.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE (The Distinction)

## Exercise 4: When PostgreSQL Features Hurt You

**Task:** For each feature below, describe a **specific, realistic scenario** where using it is **worse** than the simpler alternative. Write a small SQL snippet demonstrating the failure mode.

---

**4a: When JSONB is worse than regular columns**

A team decides to store **every** task field in JSONB to "avoid running ALTER TABLE." All queries filter by status, sort by created_at, and join on assigned_to.

Write two versions of the same query — one against a JSONB-only schema, one against proper columns — and explain why the JSONB version is worse. Consider: type safety, indexing efficiency, foreign key constraints, and query readability.

---

**4b: When an index hurts instead of helps**

Your task_manager application processes a nightly batch import of 50,000 new tasks. The tasks table has 8 indexes (on status, assigned_to, project_id, created_at, tags, metadata, description FTS, and a composite).

Explain and demonstrate (using concepts, not necessarily benchmarks) why these indexes make the batch import significantly slower. What is the write amplification factor? When should you consider dropping indexes and rebuilding them?

---

**4c: When full-text search is worse than LIKE**

A developer replaces every LIKE query with full-text search "because it's better." One of the original queries was:

```sql
SELECT * FROM tasks WHERE title LIKE 'Fix%';
```

Explain why full-text search **cannot** replicate this query's behavior. What does `to_tsquery('english', 'Fix')` do to the search term? What prefix-matching capability does LIKE have that FTS does not support in its basic form?

---

# Level 4 Solutions

## Solution 4a: JSONB vs Columns

```sql
-- JSONB-only schema (anti-pattern)
CREATE TABLE tasks_jsonb (
    id SERIAL PRIMARY KEY,
    data JSONB
);

-- Query: Find all in-progress tasks for user 5, sorted by creation date
SELECT data->>'title' AS title
FROM tasks_jsonb
WHERE data->>'status' = 'in_progress'
  AND (data->>'assigned_to')::int = 5
ORDER BY (data->>'created_at')::timestamp DESC;

-- Problems:
-- 1. No type safety: data->>'assigned_to' could be "banana" — PostgreSQL won't stop you
-- 2. No foreign key: data->>'assigned_to' = 5 but user 5 was deleted — no constraint violation
-- 3. Cast overhead: every row requires text→int and text→timestamp conversion at query time
-- 4. Index inefficiency: a GIN index on the whole JSONB can't efficiently handle range queries
--    on created_at. A B-tree on a proper TIMESTAMP column can.
-- 5. Readability: (data->>'created_at')::timestamp DESC vs created_at DESC

-- Proper schema equivalent:
SELECT title FROM tasks
WHERE status = 'in_progress' AND assigned_to = 5
ORDER BY created_at DESC;
-- Type-safe, FK-constrained, directly indexable, readable.
```

**Rule:** If a field appears on **every** row, is filtered/sorted **frequently**, or requires **referential integrity**, it must be a column. JSONB is for the variable, sparse, rarely-filtered remainder.

## Solution 4b: Index Write Amplification

Each index on the tasks table must be updated for every INSERT. With 8 indexes, a single INSERT triggers approximately 9 write operations (1 for the table heap + 8 for the indexes). A batch import of 50,000 tasks becomes ~450,000 write operations instead of ~50,000.

Additionally, B-tree indexes may need page splits during heavy inserts, and GIN indexes accumulate a pending list that must be flushed. The total overhead can make batch imports 3-10x slower depending on index complexity.

**Mitigation:** For very large batch imports, the standard approach is: (1) drop non-essential indexes, (2) perform the bulk INSERT, (3) recreate the indexes. Index creation on existing data is much faster than incremental index maintenance because PostgreSQL can sort all the entries and build the tree bottom-up.

## Solution 4c: FTS Cannot Do Prefix Matching

```sql
-- LIKE 'Fix%' matches: "Fix login", "Fix database", "Fixed bug"
-- It matches any title STARTING with "Fix" — a prefix scan.
-- With a B-tree index on title, this is extremely fast (index range scan).

-- Full-text search equivalent attempt:
SELECT * FROM tasks
WHERE to_tsvector('english', title) @@ to_tsquery('english', 'Fix');
-- Problem 1: 'Fix' is stemmed to 'fix'. It matches "fixing", "fixed", "fixes" — 
--   but also doesn't distinguish prefix from occurrence anywhere in the title.
-- Problem 2: FTS has no concept of "starts with". It matches word PRESENCE, not position.
-- Problem 3: The query to_tsquery('english', 'Fix') matches 'fix' ANYWHERE in the title,
--   not just at the beginning.

-- LIKE 'Fix%' can use a B-tree index (with appropriate operator class).
-- FTS is fundamentally about CONTENT search, not PATTERN search.
-- For prefix matching on structured fields, LIKE with B-tree is superior.
```

**The Key Insight:** LIKE and full-text search solve **different problems**. LIKE is pattern matching on character sequences. FTS is semantic matching on language tokens. Replacing LIKE everywhere with FTS is like replacing a scalpel with a chainsaw — they're both cutting tools, but the use cases don't overlap completely.

---

# LEVEL 5: THE TEACH BACK (Synthesis)

## Exercise 5: The Three Misconceptions

**Scenario:** A junior developer on your team submits a pull request for the Task Manager. In the PR description, they write:

> "I stored all the task metadata in JSONB since it's more flexible. I also added a JSONB column called `task_data` that holds the title, description, status, and created_at — this way we never need schema migrations when requirements change. For search, I replaced the old LIKE queries with `LIKE '%search_term%'` using ILIKE for case insensitivity — it works great. I didn't add any indexes because the table only has 500 rows in dev and everything is fast. Also, I'm not using transactions because each of my API endpoints only does one INSERT or UPDATE at a time."

**Task:** Write a code review response addressing **each** of the five claims. For each one:

1. Identify the specific misconception
2. Explain what will go wrong and **when** (not just "it's bad practice" — give concrete failure scenarios)
3. Provide the correct approach with a brief code example

Your review should be professional and educational — not condescending. You're teaching a colleague, not grading a student.

---

# Level 5 Solution

## Model Code Review Response

**Re: Storing title, status, created_at in JSONB**

The variable metadata like bug severity or feature priority is a perfect fit for JSONB — well done on that. However, storing `title`, `description`, `status`, and `created_at` in JSONB removes PostgreSQL's ability to enforce types, constraints, and foreign keys on those fields. In production, nothing stops a buggy client from setting `status` to `7` or `created_at` to `"yesterday"`. More critically, when we add `WHERE status = 'todo'` to an API endpoint, a B-tree index on a TEXT column is far more efficient than a GIN index on JSONB for equality lookups on 500,000 rows.

```sql
-- Keep structured, universal fields as columns:
status TEXT NOT NULL CHECK (status IN ('todo', 'in_progress', 'done', 'blocked')),
created_at TIMESTAMP NOT NULL DEFAULT NOW(),
-- Keep variable, sparse fields as JSONB:
metadata JSONB DEFAULT '{}'
```

**Re: ILIKE '%search_term%' for search**

ILIKE with wildcards on both sides has three production-killing limitations: (1) no stemming — searching "deploy" won't find "deployment" or "deployed," (2) no relevance ranking — you can't sort by how well results match, and (3) it forces a sequential scan on every query because `%prefix` patterns can't use B-tree indexes. At 500 rows this is invisible. At 500,000 rows (a small-to-medium SaaS), every search takes 100-500ms scanning every single row. Use PostgreSQL's full-text search instead:

```sql
-- Replace: WHERE title ILIKE '%authentication%'
-- With:
WHERE to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(description, ''))
      @@ to_tsquery('english', 'authentication')
-- Plus a GIN index to make it fast
```

**Re: Not adding indexes because dev is fast**

Dev performance is irrelevant — you have 500 rows. Production will have hundreds of thousands. A query that takes 1ms on 500 rows can take 500ms on 500,000 rows with a Seq Scan. Add indexes on columns you filter by (`status`, `assigned_to`, `project_id`) and verify with `EXPLAIN ANALYZE` on a representative dataset. Don't add them everywhere — indexes slow writes — but every `WHERE` clause in a frequently-hit endpoint should have index support.

**Re: Not using transactions because each endpoint does one statement**

You're actually already using transactions — PostgreSQL automatically wraps every single statement in an implicit transaction. For single-statement endpoints, you're fine. But the moment an endpoint does two related writes (e.g., "create task and log the action," "transfer task and update audit trail"), you need an explicit `BEGIN/COMMIT`. Without it, a crash between the two writes leaves your data half-updated. Check the "Move Task" feature specifically — if that does more than one write, it needs a transaction. Plan for this now, because multi-statement endpoints are inevitable as the product grows.

**The Key Insight for the Junior:** The core theme across all five points is **optimize for production, not development**. Every shortcut that's invisible at 500 rows becomes a production incident at 500,000. JSONB flexibility is a tool, not an architecture. ILIKE is a prototype, not a search engine. Missing indexes are invisible until the day they aren't. And transactions are insurance you never regret buying.