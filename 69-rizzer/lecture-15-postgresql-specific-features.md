# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, FEATURE SECOND                                  │
│  ─────────────────────────────                                  │
│  Every PostgreSQL feature is introduced by the pain it solves.  │
│  Students must hit the wall before we show them the door.       │
│                                                                 │
│  SEE IT TO BELIEVE IT                                           │
│  ────────────────────                                           │
│  Live psql demonstrations. Students watch queries execute,      │
│  see timing differences, and read EXPLAIN output in real time.  │
│  Nothing is theoretical — every feature is shown, not told.     │
│                                                                 │
│  ONE SCHEMA, MANY FEATURES                                      │
│  ─────────────────────────                                      │
│  We use the same project management schema throughout.          │
│  Every feature applies to the Task Manager they are building.   │
│  No contrived examples. Everything maps to their project.       │
│                                                                 │
│  CONNECT TO THE FULL STACK                                      │
│  ─────────────────────────                                      │
│  JSONB → Pydantic models they wrote in Week 3                   │
│  Transactions → Context managers they learned in Week 1         │
│  Indexes → The performance they'll optimize in Week 7           │
│  Every feature links backward and forward.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                 POSTGRESQL-SPECIFIC FEATURES                    │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 Demo Setup (The Data)                                   │
│  ├─ 1.2 Pain #1: Rigid Schemas for Flexible Data               │
│  ├─ 1.3 Pain #2: Search That Doesn't Work                      │
│  └─ 1.4 Pain #3: Data That Corrupts Itself                     │
│                                                                 │
│  PART 2: JSONB — FLEXIBLE DATA (45 min)                         │
│  ├─ 2.1 JSON vs JSONB (Why JSONB Always Wins)                   │
│  ├─ 2.2 Storing JSONB Data                                      │
│  ├─ 2.3 Querying JSONB (The Operators)                          │
│  ├─ 2.4 Updating JSONB Data                                     │
│  └─ 2.5 When to Use JSONB (The Decision Framework)              │
│                                                                 │
│  PART 3: ARRAY TYPES (20 min)                                   │
│  ├─ 3.1 Multiple Values, One Column                             │
│  ├─ 3.2 Array Operations                                        │
│  └─ 3.3 Arrays vs Junction Tables (The Tradeoff)               │
│                                                                 │
│  PART 4: FULL-TEXT SEARCH (25 min)                              │
│  ├─ 4.1 Why LIKE Fails                                          │
│  ├─ 4.2 tsvector and tsquery (The Engine Under the Hood)        │
│  ├─ 4.3 Practical Full-Text Queries                             │
│  └─ 4.4 When to Outgrow PostgreSQL FTS                         │
│                                                                 │
│  PART 5: EXPLAIN & INDEXES (45 min)                             │
│  ├─ 5.1 The Blind Performance Problem                           │
│  ├─ 5.2 EXPLAIN — X-Ray Your Queries                            │
│  ├─ 5.3 EXPLAIN ANALYZE (Estimates vs Reality)                  │
│  ├─ 5.4 Indexes — The Speed Multiplier                          │
│  ├─ 5.5 Creating and Verifying Indexes                          │
│  └─ 5.6 The Cost of Indexes (Nothing Is Free)                   │
│                                                                 │
│  PART 6: TRANSACTIONS (35 min)                                  │
│  ├─ 6.1 The Partial Update Problem                              │
│  ├─ 6.2 BEGIN, COMMIT, ROLLBACK                                 │
│  ├─ 6.3 Transactions and Error Recovery                         │
│  ├─ 6.4 Isolation Levels (Introduction)                         │
│  └─ 6.5 The Context Manager Connection (Link to Week 1)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 Demo Setup (The Data)

**Before we begin, seed the database students have been using since Lecture 2.**

> "We're going to use the same project management domain you've been building since Week 3. Same users, same tasks. But now we're going to push PostgreSQL to do things that standard SQL can't."

```sql
-- demo_setup.sql — Run this together with students
-- Uses the Docker PostgreSQL from Lecture 2

-- Start fresh for today's demos
DROP TABLE IF EXISTS tasks CASCADE;
DROP TABLE IF EXISTS projects CASCADE;
DROP TABLE IF EXISTS users CASCADE;

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    email TEXT UNIQUE NOT NULL,
    bio TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    owner_id INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id),
    title TEXT NOT NULL,
    description TEXT,
    status TEXT DEFAULT 'todo',
    assigned_to INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Small sample data (for JSONB and Transaction demos)
INSERT INTO users (username, email, bio) VALUES
    ('alice', 'alice@example.com', 'Backend developer who loves Python'),
    ('bob', 'bob@example.com', 'Full-stack engineer and FastAPI fan'),
    ('charlie', 'charlie@example.com', 'Database nerd and PostgreSQL enthusiast');

INSERT INTO projects (name, description, owner_id) VALUES
    ('Task Manager API', 'A RESTful API for managing tasks and projects', 1),
    ('Weather App', 'Real-time weather data aggregation service', 2);

INSERT INTO tasks (project_id, title, description, status, assigned_to) VALUES
    (1, 'Set up database schema', 'Create PostgreSQL tables with Alembic migrations for all entities', 'done', 1),
    (1, 'Implement JWT authentication', 'Add login and registration with JWT tokens and refresh token rotation', 'in_progress', 2),
    (1, 'Write API test suite', 'Add pytest suite covering all endpoints with 80 percent coverage target', 'todo', 3),
    (1, 'Fix login crash on empty password', 'App crashes with 500 when user submits empty password field', 'todo', 1),
    (1, 'Add password reset feature', 'Users should be able to reset password via email link', 'todo', 2),
    (2, 'Integrate OpenWeatherMap API', 'Connect to weather API with retry logic and rate limiting', 'in_progress', 2),
    (2, 'Add Redis caching layer', 'Cache API responses to reduce external calls', 'todo', 1);
```

**Now generate a large dataset for the performance demos later:**

```sql
-- Generate 1000 users
INSERT INTO users (username, email)
SELECT
    'user_' || i,
    'user_' || i || '@example.com'
FROM generate_series(4, 1000) AS i;

-- Generate 100 projects
INSERT INTO projects (name, description, owner_id)
SELECT
    'Project ' || i,
    'Description for project number ' || i,
    (random() * 999 + 1)::int
FROM generate_series(3, 100) AS i;

-- Generate 500,000 tasks (this takes a few seconds)
INSERT INTO tasks (project_id, title, description, status, assigned_to, created_at)
SELECT
    (random() * 99 + 1)::int,
    (ARRAY['Fix','Implement','Review','Test','Deploy','Refactor'])[floor(random()*6+1)]
        || ' ' ||
        (ARRAY['login page','dashboard API','user service','cache layer',
               'search endpoint','auth middleware'])[floor(random()*6+1)],
    'This task involves ' ||
        (ARRAY['implementing','fixing','reviewing','testing',
               'deploying','optimizing'])[floor(random()*6+1)]
        || ' the ' ||
        (ARRAY['authentication system','database queries','API endpoints',
               'caching layer','search feature','background jobs'])[floor(random()*6+1)]
        || ' for the project. Requires careful attention to ' ||
        (ARRAY['performance','security','reliability',
               'scalability','maintainability','test coverage'])[floor(random()*6+1)]
        || '.',
    (ARRAY['todo','in_progress','done','blocked'])[floor(random()*4+1)],
    (random() * 999 + 1)::int,
    NOW() - (random() * 365 || ' days')::interval
FROM generate_series(1, 500000) AS i;
```

**Run it. Let students see the timing.**

```
INSERT 0 997
INSERT 0 98
INSERT 0 500000   -- Takes 5-10 seconds depending on machine
```

> "You now have half a million tasks in your database. This is a realistic size for a small-to-medium SaaS application. Everything we demonstrate today will use this data."

---

## 1.2 Pain #1: Rigid Schemas for Flexible Data

**Set the scenario:**

> "Your product manager walks in. New requirement: different task types need different metadata. Bug reports need severity and steps to reproduce. Feature requests need priority and business value. Research tasks need a resource list. How do you model this?"

**The naive approach — add columns for everything:**

```sql
-- Approach 1: A column for every possible field
ALTER TABLE tasks ADD COLUMN severity TEXT;
ALTER TABLE tasks ADD COLUMN steps_to_reproduce TEXT;
ALTER TABLE tasks ADD COLUMN priority_level INTEGER;
ALTER TABLE tasks ADD COLUMN business_value TEXT;
ALTER TABLE tasks ADD COLUMN acceptance_criteria TEXT;
ALTER TABLE tasks ADD COLUMN resources TEXT;
ALTER TABLE tasks ADD COLUMN deadline_flexible BOOLEAN;
-- ... and 15 more for the next task type.
```

```sql
SELECT title, severity, priority_level, resources
FROM tasks LIMIT 5;
```

```
       title                 | severity | priority_level | resources
-----------------------------+----------+----------------+-----------
 Set up database schema      | NULL     |           NULL | NULL
 Implement JWT authentication| NULL     |           NULL | NULL
 Write API test suite        | NULL     |           NULL | NULL
 Fix login crash             | NULL     |           NULL | NULL
 Add password reset feature  | NULL     |           NULL | NULL
```

**Look at that. Five rows, almost every column is NULL.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE COLUMN EXPLOSION PROBLEM                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Bug Reports need:        Feature Requests need:                │
│  ├─ severity              ├─ priority_level                     │
│  ├─ steps_to_reproduce    ├─ business_value                     │
│  └─ browser_info          └─ acceptance_criteria                │
│                                                                 │
│  Research Tasks need:     Deployment Tasks need:                │
│  ├─ resources             ├─ target_environment                 │
│  ├─ deadline_flexible     ├─ rollback_plan                      │
│  └─ related_papers        └─ downtime_window                    │
│                                                                 │
│  That is 12 columns, and EVERY row uses at most 3.             │
│  The rest? NULL, NULL, NULL, NULL, NULL, NULL...                │
│                                                                 │
│  And when the next task type arrives? MORE columns.             │
│  This does not scale.                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "What if we could store ONLY the fields each task actually needs? A bug report stores severity and steps. A feature request stores priority and business value. No NULLs. No wasted columns. What would that look like?"

**Undo the damage (we'll solve this properly in Part 2):**

```sql
ALTER TABLE tasks DROP COLUMN IF EXISTS severity;
ALTER TABLE tasks DROP COLUMN IF EXISTS steps_to_reproduce;
ALTER TABLE tasks DROP COLUMN IF EXISTS priority_level;
ALTER TABLE tasks DROP COLUMN IF EXISTS business_value;
ALTER TABLE tasks DROP COLUMN IF EXISTS acceptance_criteria;
ALTER TABLE tasks DROP COLUMN IF EXISTS resources;
ALTER TABLE tasks DROP COLUMN IF EXISTS deadline_flexible;
```

---

## 1.3 Pain #2: Search That Doesn't Work

**The scenario:**

> "A user types 'authentication' into the search bar. Let's find matching tasks."

```sql
-- The obvious approach
SELECT title, description
FROM tasks
WHERE description LIKE '%authentication%'
LIMIT 5;
```

```
              title              |                    description
---------------------------------+---------------------------------------------------
 Implement auth middleware        | This task involves implementing the authentication
                                 | system for the project. Requires careful attention
                                 | to security.
 ...
```

> "OK, that works. But now the user types 'authenticate' — they mean the same thing."

```sql
SELECT title, description
FROM tasks
WHERE description LIKE '%authenticate%'
LIMIT 5;
```

```
 title | description
-------+-------------
(0 rows)
```

**Zero results. The word 'authenticate' does not appear anywhere, even though 'authentication' does.**

> "And how fast was that search?"

```sql
EXPLAIN ANALYZE
SELECT title, description
FROM tasks
WHERE description LIKE '%authentication%';
```

```
 Seq Scan on tasks  (cost=0.00..15675.00 rows=2174 width=150)
                     (actual time=0.028..112.456 rows=2231 loops=1)
   Filter: (description ~~ '%authentication%'::text)
   Rows Removed by Filter: 497776
 Planning Time: 0.071 ms
 Execution Time: 113.234 ms
```

**Point at the output:**

> "Seq Scan. It read ALL 500,000 rows to find 2,231 matches. 113 milliseconds. And that's on your local machine with no other load. In production, with hundreds of concurrent users? This is a disaster."

```
┌─────────────────────────────────────────────────────────────────┐
│                    LIKE SEARCH PROBLEMS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. NO STEMMING                                                 │
│     "authenticate" ≠ "authentication" ≠ "authenticating"       │
│     LIKE treats them as completely different strings.            │
│                                                                 │
│  2. NO RANKING                                                  │
│     Which result is MOST relevant? LIKE has no idea.            │
│     It either matches or it doesn't.                            │
│                                                                 │
│  3. ALWAYS SLOW                                                 │
│     LIKE '%word%' CANNOT use a regular index.                   │
│     It MUST read every single row. Every time.                  │
│                                                                 │
│  4. NO MULTI-WORD INTELLIGENCE                                  │
│     Searching for "fix authentication"?                         │
│     LIKE '%fix authentication%' only matches that EXACT phrase. │
│     It won't find "authentication fix" or "fix the auth".      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 Pain #3: Data That Corrupts Itself

**The scenario:**

> "We need to transfer a task from Alice to Bob's project. This means two things: update the task's project_id AND create an activity log so we have an audit trail."

```sql
-- Create a simple activity log table for this demo
CREATE TABLE activity_log (
    id SERIAL PRIMARY KEY,
    task_id INTEGER,
    action TEXT,
    performed_by INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);
```

```sql
-- Step 1: Move task #4 to project 2
UPDATE tasks SET project_id = 2 WHERE id = 4;

-- ⚡ IMAGINE: Server crashes right here. Connection drops. Power goes out. ⚡

-- Step 2: Log what we did (NEVER EXECUTES)
INSERT INTO activity_log (task_id, action, performed_by)
VALUES (4, 'Moved to Project 2', 1);
```

**Now check the state:**

```sql
-- The task WAS moved
SELECT id, title, project_id FROM tasks WHERE id = 4;
```

```
 id |             title             | project_id
----+-------------------------------+------------
  4 | Fix login crash on empty password |          2
```

```sql
-- But there's NO record of who moved it or why
SELECT * FROM activity_log WHERE task_id = 4;
```

```
 id | task_id | action | performed_by | created_at
----+---------+--------+--------------+------------
(0 rows)
```

> "The task moved, but nobody knows why. No audit trail. In a financial system, this is the equivalent of money disappearing. In your SaaS app, this means broken accountability."

**Now ask the class:**

> "How do we guarantee that BOTH operations happen, or NEITHER happens? How do we make a group of SQL statements atomic — all or nothing?"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Today we solve all three problems:                             │
│                                                                 │
│  PAIN #1: Rigid schemas    →  JSONB and Array types             │
│  PAIN #2: Broken search    →  Full-text search                  │
│  PAIN #2b: Blind slowness  →  EXPLAIN and Indexes              │
│  PAIN #3: Partial updates  →  Transactions                      │
│                                                                 │
│  These are PostgreSQL EXTENSIONS beyond standard SQL.           │
│  This is why we use PostgreSQL, not just "any SQL database."   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Reset the demo data before we continue:**

```sql
UPDATE tasks SET project_id = 1 WHERE id = 4;
DELETE FROM activity_log;
```

---

# PART 2: JSONB — FLEXIBLE DATA IN A RIGID WORLD

## 2.1 JSON vs JSONB (Why JSONB Always Wins)

**PostgreSQL has TWO JSON types. Use JSONB. Always.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     JSON vs JSONB                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JSON                             JSONB                         │
│  ────                             ─────                         │
│  Stores as raw text               Stores as binary (parsed)     │
│  Preserves whitespace             Strips whitespace             │
│  Preserves key order              May reorder keys              │
│  Preserves duplicate keys         Removes duplicate keys        │
│  Slower to query (must parse)     Fast to query (pre-parsed)    │
│  Cannot be indexed                CAN be indexed (GIN)          │
│                                                                 │
│  When to use JSON:   Almost never.                              │
│  When to use JSONB:  Almost always.                             │
│                                                                 │
│  If you need to preserve the exact original text (rare),        │
│  use JSON. For everything else, JSONB.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Think of JSON as storing a paper document in a folder — to find anything, you have to open it and read from the start every time. JSONB is like scanning the document into a searchable database — it takes a moment to ingest, but then finding data inside it is fast."

---

## 2.2 Storing JSONB Data

**Add a JSONB column to our tasks table:**

```sql
ALTER TABLE tasks ADD COLUMN metadata JSONB DEFAULT '{}';
```

**Now store different metadata for different task types:**

```sql
-- Bug report: severity, steps to reproduce
UPDATE tasks
SET metadata = '{
    "type": "bug",
    "severity": "critical",
    "browser": "Chrome 120",
    "steps_to_reproduce": [
        "Open the login page",
        "Leave password field empty",
        "Click Submit",
        "App crashes with 500 error"
    ]
}'
WHERE id = 4;

-- Feature request: priority, business value
UPDATE tasks
SET metadata = '{
    "type": "feature",
    "priority": 5,
    "business_value": "high",
    "acceptance_criteria": [
        "User receives reset email within 30 seconds",
        "Reset link expires after 24 hours",
        "User is logged in automatically after reset"
    ]
}'
WHERE id = 5;

-- Regular task: minimal metadata
UPDATE tasks
SET metadata = '{
    "type": "task",
    "estimated_hours": 8,
    "requires_review": true
}'
WHERE id = 3;
```

**Look at the result:**

```sql
SELECT id, title, metadata
FROM tasks
WHERE id IN (3, 4, 5);
```

```
 id |              title               |                   metadata
----+----------------------------------+-----------------------------------------------
  3 | Write API test suite             | {"type": "task", "estimated_hours": 8,
    |                                  |  "requires_review": true}
  4 | Fix login crash on empty password| {"type": "bug", "severity": "critical",
    |                                  |  "browser": "Chrome 120",
    |                                  |  "steps_to_reproduce": ["Open the login..."]}
  5 | Add password reset feature       | {"type": "feature", "priority": 5,
    |                                  |  "business_value": "high",
    |                                  |  "acceptance_criteria": ["User receives..."]}
```

```
┌─────────────────────────────────────────────────────────────────┐
│              REGULAR COLUMNS vs JSONB                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITH SEPARATE COLUMNS (from Part 1):                           │
│  ┌────────────────┬──────────┬──────────┬────────────┬─────┐   │
│  │ title          │ severity │ priority │ browser    │ ... │   │
│  ├────────────────┼──────────┼──────────┼────────────┼─────┤   │
│  │ Write tests    │ NULL     │ NULL     │ NULL       │ ... │   │
│  │ Fix login crash│ critical │ NULL     │ Chrome 120 │ ... │   │
│  │ Password reset │ NULL     │ 5        │ NULL       │ ... │   │
│  └────────────────┴──────────┴──────────┴────────────┴─────┘   │
│  ❌ Sparse. Mostly NULL. Schema changes = ALTER TABLE.          │
│                                                                 │
│                                                                 │
│  WITH JSONB:                                                    │
│  ┌────────────────┬──────────────────────────────────────────┐ │
│  │ title          │ metadata (JSONB)                         │ │
│  ├────────────────┼──────────────────────────────────────────┤ │
│  │ Write tests    │ {"type":"task", "estimated_hours":8}     │ │
│  │ Fix login crash│ {"type":"bug", "severity":"critical"...} │ │
│  │ Password reset │ {"type":"feature", "priority":5...}      │ │
│  └────────────────┴──────────────────────────────────────────┘ │
│  ✅ Each row stores ONLY what it needs. Zero NULLs.            │
│  ✅ New task type? Just store new keys. No ALTER TABLE.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Querying JSONB (The Operators)

**This is where JSONB gets powerful. There are two essential operators to learn first.**

**The `->` and `->>` operators:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE TWO ARROW OPERATORS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ->   Returns JSONB                                             │
│       Use for: chaining, nested access, arrays                  │
│       Think: "give me the JSON object at this key"              │
│                                                                 │
│  ->>  Returns TEXT                                              │
│       Use for: comparison, display, WHERE clauses               │
│       Think: "give me the text VALUE at this key"               │
│                                                                 │
│                                                                 │
│  RULE: Use ->> when you need to COMPARE or DISPLAY.             │
│        Use ->  when you need to go DEEPER into nested JSON.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**See the difference:**

```sql
-- -> returns JSONB (notice the quotes around the value)
SELECT metadata->'severity' AS severity_json
FROM tasks WHERE id = 4;
```

```
 severity_json
───────────────
 "critical"
```

```sql
-- ->> returns TEXT (no quotes — it's a plain string now)
SELECT metadata->>'severity' AS severity_text
FROM tasks WHERE id = 4;
```

```
 severity_text
───────────────
 critical
```

**Why this matters:**

```sql
-- ❌ WRONG: Comparing JSONB to text (type mismatch!)
SELECT title FROM tasks
WHERE metadata->'severity' = 'critical';
-- ERROR: operator does not exist: jsonb = text

-- ✅ CORRECT: ->> gives you TEXT, compare with text
SELECT title FROM tasks
WHERE metadata->>'severity' = 'critical';
```

```
              title
──────────────────────────────────
 Fix login crash on empty password
```

**Accessing nested data and arrays:**

```sql
-- Get the first step to reproduce (arrays are 0-indexed in JSONB!)
SELECT metadata->'steps_to_reproduce'->0 AS first_step
FROM tasks WHERE id = 4;
```

```
        first_step
───────────────────────────
 "Open the login page"
```

```sql
-- As text (no quotes):
SELECT metadata->'steps_to_reproduce'->>0 AS first_step
FROM tasks WHERE id = 4;
```

```
     first_step
─────────────────────
 Open the login page
```

```sql
-- Deep nesting: use #>> with a path array
-- Equivalent to metadata->'steps_to_reproduce'->>2
SELECT metadata #>> '{steps_to_reproduce, 2}' AS third_step
FROM tasks WHERE id = 4;
```

```
      third_step
─────────────────────
 Click Submit
```

**The containment operator `@>` — does this JSONB contain that subset?**

```sql
-- Find all critical bugs
SELECT id, title
FROM tasks
WHERE metadata @> '{"severity": "critical"}';
```

```
 id |              title
────+──────────────────────────────────
  4 | Fix login crash on empty password
```

```sql
-- Find all feature requests
SELECT id, title
FROM tasks
WHERE metadata @> '{"type": "feature"}';
```

```
 id |           title
────+────────────────────────────
  5 | Add password reset feature
```

**The existence operator `?` — does this key exist?**

```sql
-- Find all tasks that HAVE a severity field (regardless of value)
SELECT id, title
FROM tasks
WHERE metadata ? 'severity';
```

```
 id |              title
────+──────────────────────────────────
  4 | Fix login crash on empty password
```

**Quick reference for all operators:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   JSONB OPERATORS CHEAT SHEET                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OPERATOR    RETURNS    PURPOSE              EXAMPLE            │
│  ─────────   ────────   ──────────────────   ─────────────────  │
│  ->          JSONB      Get field by key     data->'name'       │
│  ->>         TEXT       Get field as text    data->>'name'      │
│  ->          JSONB      Get array element    data->'arr'->0     │
│  ->>         TEXT       Array elem as text   data->'arr'->>0    │
│  #>          JSONB      Get by path          data#>'{a,b}'      │
│  #>>         TEXT       Path as text         data#>>'{a,b}'     │
│  @>          BOOLEAN    Contains subset?     data @> '{"k":1}'  │
│  ?           BOOLEAN    Key exists?          data ? 'key'       │
│  ?|          BOOLEAN    Any key exists?      data ?| array      │
│  ?&          BOOLEAN    All keys exist?      data ?& array      │
│                                                                 │
│  REMEMBER:  ->> for WHERE comparisons                           │
│             ->  for chaining deeper into JSON                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Updating JSONB Data

**Three ways to modify JSONB:**

**`jsonb_set()` — Update a specific field:**

```sql
-- Change severity from "critical" to "high"
UPDATE tasks
SET metadata = jsonb_set(metadata, '{severity}', '"high"')
WHERE id = 4;

-- Verify
SELECT metadata->>'severity' FROM tasks WHERE id = 4;
```

```
 severity
──────────
 high
```

**Understand `jsonb_set` parameters:**

```
jsonb_set(target, path, new_value, create_if_missing)
           │       │        │              │
           │       │        │              └─ Default true. Set false to
           │       │        │                 only update EXISTING keys.
           │       │        └─ Must be valid JSONB. Strings need
           │       │           double quotes: '"value"' (not 'value')
           │       └─ Array of keys: '{severity}' or '{nested,key}'
           └─ The JSONB column
```

**`||` — Merge two JSONB objects:**

```sql
-- Add new fields (merge)
UPDATE tasks
SET metadata = metadata || '{"assigned_team": "backend", "reviewed": false}'
WHERE id = 4;

SELECT metadata FROM tasks WHERE id = 4;
```

```
 {"type": "bug", "severity": "high", "browser": "Chrome 120",
  "assigned_team": "backend", "reviewed": false,
  "steps_to_reproduce": ["Open the login page", ...]}
```

> "The `||` operator MERGES. Existing keys are overwritten, new keys are added. It never removes keys."

**`-` — Remove a key:**

```sql
-- Remove the browser field
UPDATE tasks
SET metadata = metadata - 'browser'
WHERE id = 4;
```

```sql
-- Remove a nested element (by index for arrays)
UPDATE tasks
SET metadata = metadata #- '{steps_to_reproduce, 0}'
WHERE id = 4;
-- Removes the first step from the array
```

---

## 2.5 When to Use JSONB (The Decision Framework)

**This is the most important part of JSONB. Not HOW, but WHEN.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  JSONB DECISION FRAMEWORK                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE JSONB WHEN:                                                │
│  ───────────────                                                │
│  ✅ Schema varies by record                                     │
│     (bug metadata ≠ feature metadata ≠ research metadata)       │
│                                                                 │
│  ✅ Fields are optional and sparse                              │
│     (most records use only a few of the possible fields)        │
│                                                                 │
│  ✅ You're storing configuration or preferences                 │
│     (user settings that change over time)                       │
│                                                                 │
│  ✅ You're caching external API responses                       │
│     (you don't control the schema)                              │
│                                                                 │
│  ✅ The data is queried occasionally, not in every query        │
│                                                                 │
│                                                                 │
│  USE REGULAR COLUMNS WHEN:                                      │
│  ──────────────────────────                                     │
│  ❌ You filter/sort on this field in most queries               │
│     (status, created_at, assigned_to → always columns)          │
│                                                                 │
│  ❌ The field exists on every record                            │
│     (title, description → always columns)                       │
│                                                                 │
│  ❌ You need foreign key constraints                            │
│     (project_id → must be a column for referential integrity)   │
│                                                                 │
│  ❌ You need strict type enforcement at the DB level            │
│     (JSONB stores anything — no column-level CHECK constraint)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The rule of thumb:**

> "If you would put it in a Pydantic `Field(...)` with strict constraints that every record must have — make it a column. If different records need different shapes — make it JSONB."

**Connection to Week 3 Lecture 3 — Pydantic:**

> "Remember Pydantic models with optional fields? JSONB is the database equivalent. Your Pydantic model validates the JSONB data at the application layer, and PostgreSQL stores it flexibly at the database layer."

```python
# How this maps to your Pydantic models (Week 3):
from pydantic import BaseModel

class BugMetadata(BaseModel):
    type: str = "bug"
    severity: str
    browser: str | None = None
    steps_to_reproduce: list[str] = []

class FeatureMetadata(BaseModel):
    type: str = "feature"
    priority: int
    business_value: str
    acceptance_criteria: list[str] = []

# These different models → stored in the SAME JSONB column
# Pydantic validates the shape. PostgreSQL stores it flexibly.
```

**Anti-pattern — do NOT do this:**

```sql
-- ❌ WRONG: Storing everything in JSONB
CREATE TABLE tasks_bad (
    id SERIAL PRIMARY KEY,
    data JSONB  -- Title, description, status, everything in here
);

-- You just reinvented a NoSQL document store...
-- inside a relational database. You lost:
-- ✗ Type safety on columns
-- ✗ Foreign key constraints
-- ✗ Fast equality lookups
-- ✗ The entire point of using PostgreSQL
```

> "JSONB is a tool for specific situations, not a replacement for schema design. If you find yourself putting everything into JSONB, you should be using MongoDB instead — and probably still shouldn't be."

---

# PART 3: ARRAY TYPES

## 3.1 Multiple Values, One Column

**Sometimes you need a simple list. Not a JSON document — just a list.**

```sql
-- Add a tags column to tasks
ALTER TABLE tasks ADD COLUMN tags TEXT[] DEFAULT '{}';
```

```sql
-- Tag some tasks
UPDATE tasks SET tags = ARRAY['backend', 'database', 'urgent']
WHERE id = 1;

UPDATE tasks SET tags = ARRAY['backend', 'security', 'auth']
WHERE id = 2;

UPDATE tasks SET tags = ARRAY['testing', 'backend']
WHERE id = 3;

UPDATE tasks SET tags = ARRAY['bug', 'frontend', 'urgent']
WHERE id = 4;

UPDATE tasks SET tags = ARRAY['feature', 'backend', 'auth']
WHERE id = 5;
```

```sql
SELECT id, title, tags FROM tasks WHERE id <= 5;
```

```
 id |              title               |           tags
────+──────────────────────────────────+──────────────────────────
  1 | Set up database schema           | {backend,database,urgent}
  2 | Implement JWT authentication     | {backend,security,auth}
  3 | Write API test suite             | {testing,backend}
  4 | Fix login crash on empty password| {bug,frontend,urgent}
  5 | Add password reset feature       | {feature,backend,auth}
```

---

## 3.2 Array Operations

**Finding tasks by tag:**

```sql
-- Contains this element? Use ANY()
SELECT id, title, tags FROM tasks
WHERE 'urgent' = ANY(tags);
```

```
 id |              title               |           tags
────+──────────────────────────────────+────────────────────────
  1 | Set up database schema           | {backend,database,urgent}
  4 | Fix login crash on empty password| {bug,frontend,urgent}
```

```sql
-- Contains ALL of these? Use @> (contains operator)
SELECT id, title, tags FROM tasks
WHERE tags @> ARRAY['backend', 'auth'];
```

```
 id |            title             |          tags
────+──────────────────────────────+──────────────────────────
  2 | Implement JWT authentication | {backend,security,auth}
  5 | Add password reset feature   | {feature,backend,auth}
```

```sql
-- Contains ANY of these? Use && (overlap operator)
SELECT id, title, tags FROM tasks
WHERE tags && ARRAY['security', 'testing'];
```

```
 id |            title             |         tags
────+──────────────────────────────+────────────────────────
  2 | Implement JWT authentication | {backend,security,auth}
  3 | Write API test suite         | {testing,backend}
```

**Other useful operations:**

```sql
-- Array length
SELECT title, array_length(tags, 1) AS tag_count
FROM tasks WHERE id <= 5;

-- Append to array
UPDATE tasks SET tags = array_append(tags, 'reviewed')
WHERE id = 1;

-- Remove from array
UPDATE tasks SET tags = array_remove(tags, 'urgent')
WHERE id = 1;

-- Unnest: expand array into rows (useful for aggregation)
SELECT DISTINCT unnest(tags) AS tag
FROM tasks
WHERE id <= 5
ORDER BY tag;
```

```
    tag
───────────
 auth
 backend
 bug
 database
 feature
 frontend
 security
 testing
 urgent
```

**Array operator quick reference:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   ARRAY OPERATORS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  'x' = ANY(arr)      Element is in array                       │
│  arr @> ARRAY[...]    Array contains ALL these elements         │
│  arr && ARRAY[...]    Arrays overlap (share any element)        │
│  arr[1]               Access element (1-indexed, not 0!)        │
│  array_length(arr,1)  Number of elements                        │
│  array_append(arr,x)  Add element                               │
│  array_remove(arr,x)  Remove element                            │
│  unnest(arr)          Expand array to rows                      │
│                                                                 │
│  ⚠️ PostgreSQL arrays are 1-INDEXED, not 0-indexed!             │
│     arr[1] is the first element. arr[0] returns NULL.           │
│     (JSONB arrays ARE 0-indexed. Yes, this is confusing.)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Arrays vs Junction Tables (The Tradeoff)

**Now ask the class:**

> "We could have modeled tags with a junction table — a `tags` table and a `task_tags` table. Why didn't we? When would we?"

```
┌─────────────────────────────────────────────────────────────────┐
│                ARRAYS VS JUNCTION TABLES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE ARRAYS WHEN:                                               │
│  ────────────────                                               │
│  ✅ Values are simple (strings, numbers)                        │
│  ✅ No extra metadata per item (just the value)                 │
│  ✅ Small lists (< 100 items typically)                         │
│  ✅ Order matters and you want to preserve it                   │
│  ✅ You don't need to JOIN on the values                        │
│  ✅ Values don't need their own identity (no tag descriptions)  │
│                                                                 │
│  USE JUNCTION TABLES WHEN:                                      │
│  ──────────────────────────                                     │
│  ✅ Tags are entities with their own attributes                 │
│     (name, color, description, created_by)                      │
│  ✅ You need to query: "all tasks with tag X" frequently        │
│  ✅ You need referential integrity (tag must exist)             │
│  ✅ You need to list all unique tags with counts                │
│  ✅ The relationship has metadata (tagged_by, tagged_at)        │
│                                                                 │
│                                                                 │
│  REAL DECISION:                                                 │
│  ──────────────                                                 │
│  tags = ["backend", "urgent"]           → Array is fine         │
│  tags = [{id: 1, name: "backend",       → Junction table        │
│           color: "#ff0", created_by: 5}]                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to Week 5 Lecture 1 — Normalization:**

> "Remember normalization rules from Lecture 1? Arrays technically violate First Normal Form — you're storing multiple values in a single column. But remember we also discussed denormalization tradeoffs. Arrays are a controlled denormalization. Use them when the simplicity gain outweighs the normalization loss."

---

# PART 4: FULL-TEXT SEARCH

## 4.1 Why LIKE Fails

**We saw this in Part 1. Let's be precise about why:**

```sql
-- LIKE is substring matching. It is NOT search.
-- These are fundamentally different operations.

-- Match: "authentication"
SELECT count(*) FROM tasks WHERE description LIKE '%authentication%';
-- Returns some results

-- No match: "authenticate" (same concept, different word form)
SELECT count(*) FROM tasks WHERE description LIKE '%authenticate%';
-- Returns 0!

-- No match: "auth" (abbreviation)
SELECT count(*) FROM tasks WHERE description LIKE '%auth%';
-- Returns DIFFERENT results (matches "auth" anywhere, including "author")
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   LIKE vs FULL-TEXT SEARCH                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LIKE '%authentication%'          FULL-TEXT SEARCH              │
│  ─────────────────────           ────────────────               │
│  Exact substring match            Language-aware matching       │
│  No stemming                      Stems: authenticate →         │
│                                     authentication ✅           │
│                                     authenticating ✅           │
│                                     authenticated ✅            │
│  No ranking                       Relevance ranking             │
│  Cannot use standard index        Uses GIN index (fast!)        │
│  Case-sensitive by default        Case-insensitive              │
│                                                                 │
│  LIKE is a hammer.                                              │
│  Full-text search is a search engine.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 tsvector and tsquery (The Engine Under the Hood)

**Full-text search has two core concepts: the DOCUMENT and the QUERY.**

```
┌─────────────────────────────────────────────────────────────────┐
│              FULL-TEXT SEARCH PIPELINE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR TEXT                                                      │
│  "Implement JWT-based authentication for the project"           │
│          │                                                      │
│          ▼  to_tsvector('english', ...)                         │
│                                                                 │
│  TSVECTOR (stemmed, normalized tokens with positions)           │
│  'authent':4 'base':3 'implement':1 'jwt':2 'project':6        │
│  │                                                              │
│  │  Notice: "the" and "for" removed (stop words)                │
│  │  "authentication" → "authent" (stemmed)                      │
│  │  "Implement" → "implement" (lowercased)                      │
│  │  Numbers = position in original text                         │
│          │                                                      │
│          │               YOUR SEARCH                            │
│          │               "authenticate"                         │
│          │                    │                                  │
│          │                    ▼  to_tsquery('english', ...)     │
│          │                                                      │
│          │               TSQUERY (stemmed search term)          │
│          │               'authent'                              │
│          │                    │                                  │
│          ▼                    ▼                                  │
│        ┌────────── @@ ──────────┐                               │
│        │       MATCH!           │                               │
│        │  'authent' found in    │                               │
│        │  tsvector at pos 4     │                               │
│        └────────────────────────┘                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**See it in action:**

```sql
-- Convert text to tsvector (tokens)
SELECT to_tsvector('english',
    'Implement JWT-based authentication for the project');
```

```
                    to_tsvector
────────────────────────────────────────────────
 'authent':4 'base':3 'implement':1 'jwt':2 'project':6
```

> "Notice four things: (1) 'authentication' was STEMMED to 'authent'. (2) 'the' and 'for' were REMOVED — they're stop words. (3) Everything is LOWERCASE. (4) Each token has a POSITION number."

```sql
-- Convert search query to tsquery
SELECT to_tsquery('english', 'authenticate');
```

```
 to_tsquery
────────────
 'authent'
```

> "'authenticate' was stemmed to 'authent' — the SAME stem as 'authentication'. This is why the match works. The search engine understands that these are the same word."

```sql
-- The match operator: @@
SELECT to_tsvector('english', 'Implement JWT-based authentication for the project')
       @@
       to_tsquery('english', 'authenticate');
```

```
 ?column?
──────────
 t
```

**`t` = true. It matches.**

---

## 4.3 Practical Full-Text Queries

**Search across real data:**

```sql
-- Find tasks about authentication
SELECT id, title
FROM tasks
WHERE to_tsvector('english', description) @@ to_tsquery('english', 'authentication');
```

**Multi-word search — AND and OR:**

```sql
-- AND: Both words must appear (use & operator)
SELECT id, title
FROM tasks
WHERE to_tsvector('english', description)
      @@ to_tsquery('english', 'security & testing');

-- OR: Either word (use | operator)
SELECT id, title
FROM tasks
WHERE to_tsvector('english', description)
      @@ to_tsquery('english', 'security | testing');

-- NOT: Exclude a term (use ! operator)
SELECT id, title
FROM tasks
WHERE to_tsvector('english', description)
      @@ to_tsquery('english', 'database & !testing');
```

**Search across multiple columns (title AND description):**

```sql
-- Combine columns with ||
SELECT id, title
FROM tasks
WHERE to_tsvector('english', title || ' ' || COALESCE(description, ''))
      @@ to_tsquery('english', 'authentication');
```

> "We use `COALESCE(description, '')` because if description is NULL, concatenating with NULL gives NULL — and the entire search breaks. This is a common gotcha."

**Relevance ranking with `ts_rank()`:**

```sql
SELECT
    id,
    title,
    ts_rank(
        to_tsvector('english', title || ' ' || COALESCE(description, '')),
        to_tsquery('english', 'database')
    ) AS rank
FROM tasks
WHERE to_tsvector('english', title || ' ' || COALESCE(description, ''))
      @@ to_tsquery('english', 'database')
ORDER BY rank DESC
LIMIT 10;
```

> "ts_rank returns a score — higher means more relevant. This is how you sort search results by relevance instead of by date or ID."

---

## 4.4 When to Outgrow PostgreSQL FTS

```
┌─────────────────────────────────────────────────────────────────┐
│              POSTGRESQL FTS vs DEDICATED SEARCH                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  POSTGRESQL FTS IS GREAT FOR:                                   │
│  ├─ Basic to moderate search needs                              │
│  ├─ < 10M rows of searchable text                               │
│  ├─ No extra infrastructure (it is IN your database!)           │
│  ├─ Simple ranking requirements                                 │
│  └─ When you don't want to maintain a separate search service  │
│                                                                 │
│  CONSIDER ELASTICSEARCH/MEILISEARCH WHEN:                       │
│  ├─ You need fuzzy matching ("autentication" → "authentication")│
│  ├─ You need complex faceted search (filter by multiple fields) │
│  ├─ You need search suggestions / autocomplete                  │
│  ├─ You need to search across multiple tables seamlessly        │
│  ├─ You need better relevance tuning (boosting, synonyms)       │
│  └─ Search is a CORE feature, not a convenience                 │
│                                                                 │
│  FOR THIS COURSE: PostgreSQL FTS is perfect.                    │
│  Most SaaS apps start here and migrate later IF needed.         │
│  Don't add infrastructure you don't need yet.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: EXPLAIN & INDEXES

## 5.1 The Blind Performance Problem

**The scenario:**

> "Your Task Manager API has been running fine for weeks. One morning, the `/tasks?status=todo` endpoint starts taking 3 seconds. Users are complaining. Your code hasn't changed. What happened?"

> "Without a way to see HOW PostgreSQL executes your queries, you're debugging blind. You don't even know if the problem is the query, the network, or the application code."

**This is what EXPLAIN solves.**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  YOUR QUERY                                                     │
│  "SELECT * FROM tasks WHERE status = 'todo'"                    │
│          │                                                      │
│          │  You see: results (eventually)                        │
│          │  You DON'T see: HOW PostgreSQL found them             │
│          │                                                      │
│          ▼                                                      │
│  ┌─────────────────────────────────────────┐                    │
│  │         PostgreSQL's Brain              │                    │
│  │   ┌─────────────────────────────────┐   │                    │
│  │   │  Query Planner                  │   │                    │
│  │   │  "How should I find this data?" │   │                    │
│  │   │                                 │   │                    │
│  │   │  Option A: Read every row       │   │                    │
│  │   │  Option B: Use an index         │   │                    │
│  │   │  Option C: Some other strategy  │   │                    │
│  │   │                                 │   │                    │
│  │   │  → Picks the cheapest option    │   │                    │
│  │   └─────────────────────────────────┘   │                    │
│  └─────────────────────────────────────────┘                    │
│          │                                                      │
│          ▼                                                      │
│  EXPLAIN shows you what the planner DECIDED.                    │
│  It is an X-ray of your query's execution strategy.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 EXPLAIN — X-Ray Your Queries

**Run EXPLAIN on a simple query:**

```sql
EXPLAIN SELECT * FROM tasks WHERE status = 'todo';
```

```
                          QUERY PLAN
──────────────────────────────────────────────────────
 Seq Scan on tasks  (cost=0.00..15675.00 rows=125562 width=252)
   Filter: (status = 'todo'::text)
```

**Now let's read every piece of this:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 READING EXPLAIN OUTPUT                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Seq Scan on tasks  (cost=0.00..15675.00 rows=125562 width=252)│
│  ─────────────────   ─────────  ────────  ──────────  ─────────│
│         │                │         │          │           │     │
│         │                │         │          │           │     │
│   SCAN TYPE        STARTUP    TOTAL COST   ESTIMATED   AVG ROW │
│                      COST                    ROWS       SIZE    │
│                                                        (bytes) │
│                                                                 │
│  Seq Scan = Sequential Scan                                     │
│             Read EVERY row in the table.                        │
│             Check each one against the filter.                  │
│             Like reading every page of a book to find one word. │
│                                                                 │
│  cost=0.00   = Cost before returning the first row              │
│  ..15675.00  = Total estimated cost (arbitrary units, not time) │
│                Higher number = more work = slower               │
│                                                                 │
│  rows=125562 = PostgreSQL ESTIMATES it will return ~125K rows   │
│                (this is a guess based on table statistics)       │
│                                                                 │
│  width=252   = Average row size is 252 bytes                    │
│                                                                 │
│   Filter: (status = 'todo'::text)                               │
│   └─ The condition applied to EACH row during the scan          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "Seq Scan means PostgreSQL is reading EVERY SINGLE ROW in the table — all 500,000 of them — to find the ones where status = 'todo'. It's the brute-force approach. For a 500,000-row table, this is expensive."

**A faster query for comparison:**

```sql
EXPLAIN SELECT * FROM tasks WHERE id = 42;
```

```
                                QUERY PLAN
──────────────────────────────────────────────────────────────────
 Index Scan using tasks_pkey on tasks  (cost=0.42..8.44 rows=1 width=252)
   Index Cond: (id = 42)
```

> "Notice: Index Scan, not Seq Scan. Cost is 8.44, not 15,675. It found one row by jumping straight to it via the primary key index — like looking up a page number in a book's index."

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMON SCAN TYPES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Seq Scan                                                       │
│  ─────────                                                      │
│  Reads every row. Checks the filter on each one.                │
│  Like reading every page of a book.                             │
│  Slow on big tables. Sometimes unavoidable.                     │
│                                                                 │
│  Index Scan                                                     │
│  ──────────                                                     │
│  Uses an index to jump directly to matching rows.               │
│  Like using a book's index to find page numbers.                │
│  Fast for queries returning few rows.                           │
│                                                                 │
│  Bitmap Heap Scan + Bitmap Index Scan                           │
│  ─────────────────────────────────────                          │
│  Uses the index to build a "map" of matching rows,              │
│  then fetches them in bulk.                                     │
│  A middle ground: faster than Seq Scan,                         │
│  efficient when many rows match.                                │
│                                                                 │
│  Seq Scan: Read all.   Cost: 💰💰💰                             │
│  Bitmap:   Map, then fetch bulk.   Cost: 💰💰                   │
│  Index:    Jump to exact rows.     Cost: 💰                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 EXPLAIN ANALYZE (Estimates vs Reality)

**EXPLAIN alone shows the PLAN. EXPLAIN ANALYZE actually RUNS the query and shows real numbers.**

```sql
-- ⚠️ CAUTION: EXPLAIN ANALYZE RUNS the query!
-- For SELECT: safe, just slower (it actually executes)
-- For UPDATE/DELETE: ACTUALLY MODIFIES DATA. Wrap in a transaction!

EXPLAIN ANALYZE SELECT * FROM tasks WHERE status = 'todo';
```

```
                                                    QUERY PLAN
──────────────────────────────────────────────────────────────────────────────────────────
 Seq Scan on tasks  (cost=0.00..15675.00 rows=125562 width=252)
                     (actual time=0.018..94.371 rows=124923 loops=1)
   Filter: (status = 'todo'::text)
   Rows Removed by Filter: 375084
 Planning Time: 0.065 ms
 Execution Time: 101.842 ms
```

**Now annotate the new fields:**

```
┌─────────────────────────────────────────────────────────────────┐
│                EXPLAIN ANALYZE — EXTRA FIELDS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  (actual time=0.018..94.371 rows=124923 loops=1)                │
│               │         │        │          │                   │
│               │         │        │          └─ How many times   │
│               │         │        │             this node ran    │
│               │         │        └─ ACTUAL rows returned        │
│               │         │           (compare to estimate!)      │
│               │         └─ Total time in milliseconds           │
│               └─ Time to first row in milliseconds              │
│                                                                 │
│  Rows Removed by Filter: 375084                                 │
│  └─ Rows that were READ but didn't match the WHERE clause.     │
│     PostgreSQL read 375,084 + 124,923 = ~500,007 rows total.   │
│     It read ALL of them to find the ~125K matches.              │
│                                                                 │
│  Planning Time: 0.065 ms                                        │
│  └─ Time spent deciding HOW to run the query.                   │
│                                                                 │
│  Execution Time: 101.842 ms                                     │
│  └─ Total time ACTUALLY running the query.                      │
│     This is the number you care about most.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Compare estimates to actual:**

```
Estimated rows: 125,562
Actual rows:    124,923
                ──────
Difference:     ~0.5%    ← PostgreSQL's estimate was very close!
```

> "When estimates and actuals are wildly different, the query planner made a bad decision. This can happen with stale statistics. Running `ANALYZE tasks;` refreshes them. We'll go deeper into this in Week 7."

**CAUTION with EXPLAIN ANALYZE on write queries:**

```sql
-- ⚠️ This ACTUALLY deletes rows!
EXPLAIN ANALYZE DELETE FROM tasks WHERE id = 999;

-- ✅ Safe way: Wrap in a transaction and rollback
BEGIN;
EXPLAIN ANALYZE DELETE FROM tasks WHERE id = 999;
ROLLBACK;  -- Undo the delete
```

---

## 5.4 Indexes — The Speed Multiplier

**We've seen the problem: Seq Scan reads every row. Indexes are the solution.**

**The book index analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE BOOK INDEX ANALOGY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT AN INDEX (Seq Scan):                                   │
│                                                                 │
│  "Find every mention of 'authentication' in this 500-page book"│
│   → Read page 1. Nope.                                          │
│   → Read page 2. Nope.                                          │
│   → Read page 3. Yes! Note it.                                  │
│   → Read page 4. Nope.                                          │
│   → ... read ALL 500 pages ...                                  │
│   → Done. Took forever.                                         │
│                                                                 │
│                                                                 │
│  WITH AN INDEX (Index Scan):                                    │
│                                                                 │
│  "Find every mention of 'authentication' in this 500-page book"│
│   → Open the INDEX at the back of the book.                     │
│   → Look up "authentication" → Pages 3, 47, 182, 301           │
│   → Go directly to those pages.                                 │
│   → Done. Instant.                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How a B-tree index works (simplified):**

```
┌─────────────────────────────────────────────────────────────────┐
│                     B-TREE INDEX                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  B-tree = Balanced tree. Data is sorted. Lookups are O(log n).  │
│                                                                 │
│  Finding status = 'done' in 500,000 rows:                       │
│                                                                 │
│                      [in_progress]                              │
│                       /         \                               │
│                [done]             [todo]                         │
│                /    \             /    \                         │
│         [blocked] [done]    [in_prog] [todo]                    │
│               ↓                                                 │
│          Leaf nodes point to actual table rows                   │
│                                                                 │
│  Steps to find 'done':                                          │
│  1. Root: 'done' < 'in_progress' → go left                     │
│  2. Found 'done' → follow pointer to table rows                │
│                                                                 │
│  2 steps instead of 500,000 row reads!                          │
│                                                                 │
│  B-tree works for: =, <, >, <=, >=, BETWEEN, ORDER BY          │
│  B-tree is the DEFAULT index type in PostgreSQL.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Creating and Verifying Indexes

**The demo. This is the payoff.**

**Step 1 — Measure the problem:**

```sql
EXPLAIN ANALYZE SELECT * FROM tasks WHERE status = 'todo';
```

```
 Seq Scan on tasks  (cost=0.00..15675.00 rows=125562 width=252)
                     (actual time=0.019..94.371 rows=124923 loops=1)
   Filter: (status = 'todo'::text)
   Rows Removed by Filter: 375084
 Execution Time: 101.842 ms
```

**101 ms. Seq Scan. Reading every row.**

**Step 2 — Add an index:**

```sql
CREATE INDEX idx_tasks_status ON tasks(status);
```

**Step 3 — Measure again:**

```sql
EXPLAIN ANALYZE SELECT * FROM tasks WHERE status = 'todo';
```

```
 Bitmap Heap Scan on tasks  (cost=1401.23..9856.45 rows=125562 width=252)
                              (actual time=8.123..47.891 rows=124923 loops=1)
   Recheck Cond: (status = 'todo'::text)
   Heap Blocks: exact=8347
   ->  Bitmap Index Scan on idx_tasks_status  (cost=0.00..1369.84 rows=125562 width=0)
                                                (actual time=5.678..5.678 rows=124923 loops=1)
         Index Cond: (status = 'todo'::text)
 Execution Time: 54.234 ms
```

**Before and after side by side:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    BEFORE vs AFTER INDEX                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE (no index):                                             │
│  ├─ Scan type:      Seq Scan (read ALL 500,000 rows)           │
│  ├─ Total cost:     15,675                                      │
│  ├─ Execution time: 101 ms                                      │
│  └─ Rows examined:  500,007 (all of them)                       │
│                                                                 │
│  AFTER (with index):                                            │
│  ├─ Scan type:      Bitmap Index Scan → Bitmap Heap Scan       │
│  ├─ Total cost:     9,856                                       │
│  ├─ Execution time: 54 ms                                       │
│  └─ Rows examined:  124,923 (only matching rows)                │
│                                                                 │
│  IMPROVEMENT: ~47% faster, ~75% fewer rows examined             │
│                                                                 │
│  NOTE: Why Bitmap Scan, not Index Scan?                         │
│  Because ~25% of rows match. That is a LOT of rows.            │
│  For large result sets, Bitmap Scan is more efficient than      │
│  jumping back and forth with Index Scan.                        │
│  For small result sets (1-5% of rows), you'd see Index Scan.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now try a more selective query:**

```sql
-- Only a few tasks are "blocked" — very selective
EXPLAIN ANALYZE SELECT * FROM tasks WHERE status = 'blocked';
```

```
 Index Scan using idx_tasks_status on tasks  (cost=0.42..4521.67 rows=125048 width=252)
                                               (actual time=0.024..32.456 rows=124876 loops=1)
   Index Cond: (status = 'blocked'::text)
 Execution Time: 38.912 ms
```

> "NOW we see Index Scan. When the result set is smaller relative to the table, PostgreSQL jumps directly to each row via the index. When many rows match, it uses Bitmap Scan to batch the lookups."

**Which columns already have indexes?**

```sql
-- Every PRIMARY KEY automatically gets an index!
-- Foreign keys do NOT automatically get indexes in PostgreSQL (common surprise)

-- See existing indexes on the tasks table:
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'tasks';
```

```
      indexname       |                       indexdef
──────────────────────+────────────────────────────────────────────
 tasks_pkey           | CREATE UNIQUE INDEX tasks_pkey ON tasks USING btree (id)
 idx_tasks_status     | CREATE INDEX idx_tasks_status ON tasks USING btree (status)
```

---

## 5.6 The Cost of Indexes (Nothing Is Free)

**Indexes are not "just add them everywhere." They have costs.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE COST OF INDEXES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  READ  FASTER  ✅     Queries that use the index are faster.    │
│  WRITE SLOWER  ❌     Every INSERT, UPDATE, DELETE must also    │
│                       update the index. More indexes = slower   │
│                       writes.                                   │
│                                                                 │
│  DISK SPACE    📀     Each index takes disk space. On a         │
│                       500K-row table, one index might be 10-50MB│
│                       Add 10 indexes? That's significant.       │
│                                                                 │
│  MAINTENANCE   🔧     PostgreSQL must keep indexes in sync.     │
│                       Dead tuples accumulate. VACUUM must clean │
│                       up both table AND indexes.                │
│                                                                 │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  ──────────────                                                 │
│  Index columns that appear in WHERE clauses of frequent queries.│
│  Do NOT index columns you rarely filter on.                     │
│  Do NOT index tables with < 1000 rows (Seq Scan is fine).       │
│  Do NOT index columns with very few distinct values on small    │
│  tables (boolean columns with only true/false — low payoff).    │
│                                                                 │
│  When in doubt: EXPLAIN ANALYZE before and after.               │
│  Let the numbers decide, not your intuition.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What about indexes on our other new features?**

```sql
-- Index on JSONB: use GIN (Generalized Inverted Index)
CREATE INDEX idx_tasks_metadata ON tasks USING gin(metadata);

-- Now this is fast:
EXPLAIN ANALYZE SELECT * FROM tasks WHERE metadata @> '{"type": "bug"}';

-- Index on Array: also uses GIN
CREATE INDEX idx_tasks_tags ON tasks USING gin(tags);

-- Now this is fast:
EXPLAIN ANALYZE SELECT * FROM tasks WHERE tags @> ARRAY['urgent'];

-- Index for Full-Text Search: GIN on tsvector
CREATE INDEX idx_tasks_description_fts
ON tasks USING gin(to_tsvector('english', description));

-- Now this is fast:
EXPLAIN ANALYZE
SELECT * FROM tasks
WHERE to_tsvector('english', description) @@ to_tsquery('english', 'authentication');
```

> "We'll explore GIN, GiST, partial indexes, composite indexes, and expression indexes in depth in Week 7. For now: B-tree is the default for normal columns, and GIN is used for JSONB, arrays, and full-text search."

**Connection to Week 7:**

> "Today is the introduction. In Week 7, you'll learn to read complex multi-node EXPLAIN plans, understand composite indexes where column order matters, create partial indexes that only cover certain rows, and prove optimizations with documented before/after performance numbers. Today: learn to read the X-ray. Week 7: learn to operate."

---

# PART 6: TRANSACTIONS

## 6.1 The Partial Update Problem

**Back to the scenario from Part 1.**

> "We need to move a task to a different project AND log the action. Both must happen, or neither. This is the textbook case for a transaction."

```
┌─────────────────────────────────────────────────────────────────┐
│               THE PARTIAL UPDATE PROBLEM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT TRANSACTION:                                           │
│                                                                 │
│  Statement 1: UPDATE tasks SET project_id = 2 WHERE id = 4;    │
│               ✅ Success — task moved                           │
│                                                                 │
│               ⚡ CRASH / ERROR / DISCONNECT ⚡                  │
│                                                                 │
│  Statement 2: INSERT INTO activity_log (...) VALUES (...);      │
│               ❌ Never executed — no audit trail!               │
│                                                                 │
│  RESULT: Inconsistent data. Task moved but nobody knows why.   │
│                                                                 │
│                                                                 │
│  WITH TRANSACTION:                                              │
│                                                                 │
│  BEGIN;                                                         │
│  Statement 1: UPDATE tasks SET project_id = 2 WHERE id = 4;    │
│  Statement 2: INSERT INTO activity_log (...) VALUES (...);      │
│  COMMIT;                                                        │
│                                                                 │
│  If ANYTHING fails between BEGIN and COMMIT:                    │
│  → ALL changes are rolled back automatically.                   │
│  → Database is in the SAME state as before BEGIN.               │
│  → No partial updates. No inconsistency.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The bank transfer analogy:**

> "This is the classic example: transferring $100 from Alice to Bob. Deduct from Alice, add to Bob. If only the deduction happens, money vanishes from the system. Transactions guarantee both happen or neither happens."

---

## 6.2 BEGIN, COMMIT, ROLLBACK

**The three commands:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TRANSACTION LIFECYCLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│       BEGIN                                                     │
│         │         "Start a transaction.                         │
│         │          Nothing is permanent until COMMIT."          │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Transaction │                                                │
│  │   Active    │─── SQL statements execute here ───┐            │
│  │             │   (INSERT, UPDATE, DELETE, SELECT) │            │
│  └──────┬──────┘                                   │            │
│         │                                          │            │
│    ┌────┴────┐                               ┌─────┴─────┐      │
│    │         │                               │           │      │
│    ▼         ▼                               ▼           │      │
│ ┌────────┐ ┌──────────┐               ┌──────────┐      │      │
│ │ COMMIT │ │ ROLLBACK │               │  ERROR   │──────┘      │
│ │        │ │          │               │          │  (auto-     │
│ │ Make   │ │ Undo     │               │ Marks tx │   aborted)  │
│ │ all    │ │ all      │               │ as failed│             │
│ │ changes│ │ changes  │               │ Must     │             │
│ │ final  │ │ as if    │               │ ROLLBACK │             │
│ │        │ │ nothing  │               └──────────┘             │
│ │        │ │ happened │                                         │
│ └────────┘ └──────────┘                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Live demonstration — follow along in psql:**

```sql
-- Check current state
SELECT id, title, project_id FROM tasks WHERE id = 4;
```

```
 id |              title               | project_id
────+──────────────────────────────────+────────────
  4 | Fix login crash on empty password|          1
```

```sql
-- Start a transaction
BEGIN;

-- Move the task
UPDATE tasks SET project_id = 2 WHERE id = 4;

-- Log it
INSERT INTO activity_log (task_id, action, performed_by)
VALUES (4, 'Moved from Project 1 to Project 2', 1);

-- Inside the transaction, we can see the changes:
SELECT id, title, project_id FROM tasks WHERE id = 4;
```

```
 id |              title               | project_id
────+──────────────────────────────────+────────────
  4 | Fix login crash on empty password|          2
```

> "The change is visible to US — inside our transaction. But if another user queries right now, they still see project_id = 1. The change is not committed yet."

```sql
-- We changed our mind. Undo everything.
ROLLBACK;

-- Check: everything is back to original
SELECT id, title, project_id FROM tasks WHERE id = 4;
```

```
 id |              title               | project_id
────+──────────────────────────────────+────────────
  4 | Fix login crash on empty password|          1
```

> "ROLLBACK undid both the UPDATE and the INSERT. The database is exactly as it was before BEGIN. Nothing happened."

**Now do it for real:**

```sql
BEGIN;

UPDATE tasks SET project_id = 2 WHERE id = 4;

INSERT INTO activity_log (task_id, action, performed_by)
VALUES (4, 'Moved from Project 1 to Project 2', 1);

-- This time, we're sure. Make it permanent.
COMMIT;

-- Both changes are now permanent
SELECT id, title, project_id FROM tasks WHERE id = 4;
SELECT * FROM activity_log WHERE task_id = 4;
```

```
 id |              title               | project_id
────+──────────────────────────────────+────────────
  4 | Fix login crash on empty password|          2

 id | task_id |              action              | performed_by |         created_at
────+---------+──────────────────────────────────+--------------+────────────────────────
  1 |       4 | Moved from Project 1 to Project 2|            1 | 2025-02-12 10:30:00
```

> "BOTH operations persisted. If the server had crashed between the two statements (but before COMMIT), NEITHER would have persisted. That's the guarantee."

---

## 6.3 Transactions and Error Recovery

**What happens when an error occurs INSIDE a transaction?**

```sql
BEGIN;

UPDATE tasks SET project_id = 1 WHERE id = 4;

-- This will cause an error (violating a constraint)
INSERT INTO tasks (id, title, project_id) VALUES (4, 'Duplicate!', 1);
-- ERROR: duplicate key value violates unique constraint "tasks_pkey"
```

> "At this point, PostgreSQL marks the transaction as ABORTED. You CANNOT run any more statements — you MUST ROLLBACK."

```sql
-- Try to do something else after the error:
SELECT * FROM tasks WHERE id = 4;
-- ERROR: current transaction is aborted,
-- commands ignored until end of transaction block

-- The only option is:
ROLLBACK;
```

```
┌─────────────────────────────────────────────────────────────────┐
│               ERRORS IN TRANSACTIONS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ⚠️ IMPORTANT:                                                  │
│  Once an error occurs inside a transaction, the transaction     │
│  is DEAD. All subsequent statements are rejected.               │
│  You MUST issue ROLLBACK before you can do anything else.       │
│                                                                 │
│  BEGIN;                                                         │
│  UPDATE ... ✅                                                  │
│  INSERT ... ❌ ERROR                                            │
│  SELECT ... ❌ "current transaction is aborted"                 │
│  UPDATE ... ❌ "current transaction is aborted"                 │
│  ROLLBACK;  ← The ONLY valid command now                        │
│                                                                 │
│  The UPDATE that succeeded? Also rolled back.                   │
│  EVERYTHING since BEGIN is undone.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Savepoints — partial rollback within a transaction:**

```sql
BEGIN;

UPDATE tasks SET status = 'in_progress' WHERE id = 3;

-- Create a savepoint before the risky operation
SAVEPOINT before_risky;

-- Try something that might fail
INSERT INTO tasks (id, title, project_id) VALUES (4, 'Duplicate!', 1);
-- ERROR: duplicate key value violates unique constraint

-- Roll back ONLY to the savepoint (not the entire transaction!)
ROLLBACK TO SAVEPOINT before_risky;

-- The UPDATE from earlier is STILL in effect!
-- We can continue the transaction:
UPDATE tasks SET status = 'done' WHERE id = 1;

COMMIT;
-- Both UPDATEs are committed. The failed INSERT is gone.
```

> "Savepoints are like checkpoints in a video game. If you fail, you respawn at the checkpoint — you don't restart the entire game."

---

## 6.4 Isolation Levels (Introduction)

**The question: when you're inside a transaction, what do you see from OTHER transactions?**

```
┌─────────────────────────────────────────────────────────────────┐
│              ISOLATION LEVELS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User A                          User B                         │
│  ───────                         ───────                        │
│  BEGIN;                          BEGIN;                          │
│  UPDATE tasks SET status =                                      │
│    'done' WHERE id = 1;                                         │
│                                  SELECT status FROM tasks       │
│                                  WHERE id = 1;                  │
│                                                                 │
│  QUESTION: What does User B see?                                │
│                                                                 │
│  ┌──────────────────────────────────────────────────┐           │
│  │  READ COMMITTED (PostgreSQL default):            │           │
│  │  User B sees 'todo' (the old value).             │           │
│  │  B only sees COMMITTED changes.                  │           │
│  │  After User A COMMITs:                           │           │
│  │  User B's NEXT query sees 'done'.                │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
│  ┌──────────────────────────────────────────────────┐           │
│  │  REPEATABLE READ:                                │           │
│  │  User B sees 'todo' always — even after A        │           │
│  │  commits. B's view is FROZEN at the start of     │           │
│  │  B's transaction. Consistent snapshot.           │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
│  ┌──────────────────────────────────────────────────┐           │
│  │  SERIALIZABLE:                                   │           │
│  │  Strictest level. Transactions behave AS IF they │           │
│  │  ran one at a time. If a conflict is detected,   │           │
│  │  PostgreSQL aborts one transaction. Your code    │           │
│  │  must retry.                                     │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
│  FOR THIS COURSE: READ COMMITTED (the default) is correct      │
│  for 99% of web applications. Know the others exist.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to set isolation level:**

```sql
-- Set for a single transaction
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ... your statements ...
COMMIT;

-- Check current default
SHOW default_transaction_isolation;
-- "read committed"
```

> "Isolation levels are about TRADEOFFS. Higher isolation = fewer weird behaviors, but more contention and possible aborted transactions. READ COMMITTED is the sweet spot for most web APIs. If you need stronger guarantees (financial transactions, inventory management), step up to SERIALIZABLE — but be prepared to handle retries."

---

## 6.5 The Context Manager Connection (Link to Week 1)

**Connection to Week 1 Lecture 2 — Context Managers:**

> "Remember context managers? `with open('file') as f:` — the file is guaranteed to close, even if an exception occurs. Transactions follow the exact same pattern."

```python
# Week 1: Context managers guarantee cleanup
with open("data.txt") as f:
    data = f.read()
    process(data)
# File is ALWAYS closed — even if process() raises an exception

# Transactions are the database equivalent:
# BEGIN  → "opening" the transaction
# COMMIT → "closing" successfully (saving the changes)
# ROLLBACK → "closing" on failure (undoing the changes)
```

```
┌─────────────────────────────────────────────────────────────────┐
│           CONTEXT MANAGERS ↔ TRANSACTIONS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONTEXT MANAGER                  TRANSACTION                   │
│  ───────────────                  ───────────                   │
│  with open(f) as file:            BEGIN;                        │
│      file.write(data)                UPDATE ...;               │
│      file.write(more)                INSERT ...;               │
│  # auto-close on exit             COMMIT;                      │
│  # auto-close on exception        # auto-ROLLBACK on error     │
│                                                                 │
│  Same pattern:                                                  │
│  "Do a group of operations. If anything fails,                  │
│   clean up automatically. Never leave things half-done."        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Preview — this is EXACTLY how you'll use transactions in Week 6 with SQLAlchemy:**

```python
# Week 6 Preview: SQLAlchemy transactions ARE context managers
async with async_session() as session:
    async with session.begin():       # ← BEGIN
        session.add(new_task)         # ← INSERT
        session.add(activity_log)     # ← INSERT
    # ← COMMIT happens automatically here

    # If any exception occurs inside the block:
    # ← ROLLBACK happens automatically
```

> "When you see this in Week 6, it should feel familiar. It's context managers from Week 1 wrapping transactions from today. The same mental model you already have."

---

**Important detail before we move on: implicit transactions.**

```sql
-- ⚠️ PostgreSQL wraps EVERY statement in a transaction automatically!
-- When you write:
INSERT INTO tasks (title, project_id) VALUES ('New task', 1);

-- PostgreSQL actually runs:
BEGIN;
INSERT INTO tasks (title, project_id) VALUES ('New task', 1);
COMMIT;

-- This means: single statements are ALWAYS atomic.
-- You only need explicit BEGIN/COMMIT when you have
-- MULTIPLE statements that must succeed or fail together.
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              POSTGRESQL FEATURES QUICK REFERENCE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JSONB:                                                         │
│      Store:   '{"key": "value"}'                                │
│      Read:    column->>'key'          (returns TEXT)             │
│      Read:    column->'key'           (returns JSONB)            │
│      Filter:  WHERE col @> '{"k":1}'  (contains)                │
│      Exists:  WHERE col ? 'key'       (key exists)              │
│      Update:  jsonb_set(col, '{key}', '"val"')                  │
│      Merge:   col || '{"new_key": 1}'                           │
│      Remove:  col - 'key'                                       │
│                                                                 │
│  ARRAYS:                                                        │
│      Declare: column_name TEXT[]                                 │
│      Insert:  ARRAY['a', 'b', 'c']                              │
│      Element: 'x' = ANY(column)                                 │
│      All:     column @> ARRAY['a','b']  (contains all)          │
│      Overlap: column && ARRAY['a','b']  (shares any)            │
│      ⚠️ 1-indexed! arr[1] is first element.                     │
│                                                                 │
│  FULL-TEXT SEARCH:                                              │
│      Convert: to_tsvector('english', text)                      │
│      Query:   to_tsquery('english', 'word')                     │
│      Match:   tsvector @@ tsquery                               │
│      AND:     to_tsquery('word1 & word2')                       │
│      OR:      to_tsquery('word1 | word2')                       │
│      Rank:    ts_rank(tsvector, tsquery)                        │
│                                                                 │
│  EXPLAIN:                                                       │
│      Plan only:  EXPLAIN SELECT ...                             │
│      With timing: EXPLAIN ANALYZE SELECT ...                    │
│      ⚠️ ANALYZE runs the query! Wrap writes in transaction.     │
│      Key scans:  Seq Scan (slow) → Index Scan (fast)            │
│                                                                 │
│  INDEXES:                                                       │
│      Create:  CREATE INDEX name ON table(column);               │
│      GIN:     CREATE INDEX name ON table USING gin(column);     │
│      Drop:    DROP INDEX name;                                  │
│      List:    SELECT * FROM pg_indexes WHERE tablename = '...'; │
│      ⚠️ Speed up reads, slow down writes. Don't over-index.     │
│                                                                 │
│  TRANSACTIONS:                                                  │
│      Start:    BEGIN;                                           │
│      Save:     COMMIT;                                          │
│      Undo:     ROLLBACK;                                        │
│      Partial:  SAVEPOINT name; / ROLLBACK TO SAVEPOINT name;    │
│      ⚠️ Error in tx = tx aborted. Must ROLLBACK.                │
│      ⚠️ Every single statement is auto-wrapped if no BEGIN.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  POSTGRESQL = SQL + A TOOLKIT                                   │
│                                                                 │
│  Standard SQL stores data in rigid rows and columns.            │
│  PostgreSQL extends this with specialized tools:                │
│                                                                 │
│  ┌─────────────┐  ┌──────────┐  ┌─────────────┐                │
│  │   JSONB     │  │  Arrays  │  │  Full-Text  │                │
│  │             │  │          │  │   Search    │                │
│  │  Flexible   │  │ Simple   │  │             │                │
│  │  data in a  │  │ lists    │  │  Language-  │                │
│  │  rigid      │  │ without  │  │  aware      │                │
│  │  world      │  │ extra    │  │  search     │                │
│  │             │  │ tables   │  │  inside     │                │
│  │             │  │          │  │  your DB    │                │
│  └─────────────┘  └──────────┘  └─────────────┘                │
│                                                                 │
│  ┌─────────────────────────┐  ┌─────────────────────────┐      │
│  │   EXPLAIN + Indexes     │  │    Transactions         │      │
│  │                         │  │                         │      │
│  │  See HOW your queries   │  │  All-or-nothing.        │      │
│  │  execute. Make them     │  │  Never leave your       │      │
│  │  faster with surgical   │  │  data half-updated.     │      │
│  │  precision.             │  │  The database's         │      │
│  │                         │  │  context manager.       │      │
│  └─────────────────────────┘  └─────────────────────────┘      │
│                                                                 │
│  THE DECISION FRAMEWORK:                                        │
│  ├─ Data varies per record?         → JSONB                     │
│  ├─ Need a simple list?             → Array                     │
│  ├─ Need real search (not LIKE)?    → Full-Text Search          │
│  ├─ Query slow and don't know why?  → EXPLAIN                   │
│  ├─ Seq Scan on a big table?        → Add an Index              │
│  └─ Multiple ops that must all      → Transaction               │
│     succeed or all fail?                                        │
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
│  WEEK 5, LECTURE 4 (Next lecture):                               │
│  └─ Database Design Workshop                                    │
│     You'll design schemas using JSONB, arrays, and proper       │
│     indexing strategies for a multi-entity system.               │
│                                                                 │
│  WEEK 5, MINI-PROJECT:                                          │
│  └─ Design and populate a database                              │
│     You'll use JSONB for flexible metadata, write raw SQL       │
│     with full-text search, and document your schema with        │
│     an ER diagram. EXPLAIN output will prove your indexes work. │
│                                                                 │
│  WEEK 6 (SQLAlchemy & Alembic):                                 │
│  └─ Everything from today maps to Python ORM code.              │
│     JSONB → mapped_column(JSONB)                                │
│     Transactions → async with session.begin()                   │
│     Indexes → Index() in SQLAlchemy model metadata              │
│     The SQL you write today becomes the ORM code of next week.  │
│                                                                 │
│  WEEK 7 (Advanced Database Patterns):                           │
│  └─ Deep dive into query optimization.                          │
│     EXPLAIN ANALYZE becomes your daily tool.                    │
│     Composite indexes, partial indexes, GIN vs GiST.           │
│     You will document before/after performance numbers.         │
│                                                                 │
│  WEEK 9 (Authentication):                                       │
│  └─ User tables with JSONB preferences.                         │
│     Transactions for atomic user registration.                  │
│     Indexes on email and username for fast login lookups.        │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Multi-tenant SaaS with full-text search,                    │
│     JSONB metadata, optimized queries with EXPLAIN proof,       │
│     and transactions for every multi-step operation.            │
│     Everything from today will be used.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```