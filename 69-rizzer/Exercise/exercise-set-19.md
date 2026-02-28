# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Alembic is a smart synchronization tool that automatically keeps my database in sync with my SQLAlchemy models. I change the model, run autogenerate, apply the result, and everything is handled. Migration files are generated artifacts — like `__pycache__` — that I don't really need to read, understand, or treat as source code."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | Autogenerate detects column renames | It sees DROP old column + ADD new column — data is silently destroyed |
| 2 | `alembic_version` stores a log of all applied migrations | It stores exactly ONE row: the current head revision, nothing else |
| 3 | Editing an applied migration file fixes the bug | Alembic never re-runs applied revisions; the edit is completely invisible |
| 4 | Adding a NOT NULL column to an existing table "just works" | Existing rows have no value for the new column → constraint violation → migration crashes |
| 5 | `downgrade()` perfectly reverses `upgrade()` | Destructive operations (DROP COLUMN, DROP TABLE) lose data that cannot be recovered |
| 6 | Autogenerate produces correct migrations regardless of database state | Running it against a stale database includes already-handled changes as duplicates |
| 7 | Migration files are generated artifacts like compiled output | They ARE source code — must be committed, reviewed, and never modified after application |

**The Prediction Gap:** When told that autogenerate "detects differences between models and the database," students assume the detection is *semantic* — that Alembic understands developer intent (e.g., recognizing a rename). In reality, autogenerate performs a *structural* diff between two snapshots. It cannot distinguish "column renamed" from "old column deleted, different column added." The migration it generates for a rename will silently destroy every value in the original column with no warning.

---

## The Main Purpose (Essence)

**Core Insight:** Database schemas evolve over time just like code. But unlike code — which can be fully rebuilt from source files — a database contains *irreplaceable state* (data). Alembic solves the fundamental problem that declarative definitions (SQLAlchemy models) describe the desired end state but cannot express the *transitions* needed to reach that state safely. Migrations encode the **HOW**: the ordered, atomic, reversible steps to move a database from one known state to another while preserving data.

**The Problems Being Solved:**

| Approach | What It Does | What It Cannot Do |
|----------|-------------|-------------------|
| `metadata.create_all` | Creates tables that don't exist yet | Alter existing tables, add columns, change types — anything that modifies existing structure |
| `DROP + CREATE` | Rebuilds tables from scratch with new schema | Preserve any existing data — everything is destroyed |
| Manual `ALTER TABLE` | Makes specific changes to table structure | Track which changes have been applied, by whom, in what order, or how to undo them |
| **Alembic migration** | Records a specific, reversible schema change with full history | — |

**The Key Distinction:**
- `metadata.create_all` says "here is what the database SHOULD look like" → declarative, one-shot, ignores existing state
- An Alembic migration says "here is HOW to get from state A to state B" → imperative, incremental, preserves data
- Autogenerate bridges the two by computing the structural diff, but that diff is a *starting point* that must be reviewed and sometimes manually corrected — it is never a final answer

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "Autogenerate detects renames" | Renaming `role` → `access_level` produces `ALTER COLUMN RENAME` | Produces `DROP COLUMN role` + `ADD COLUMN access_level`; all role data destroyed | Autogenerate compares two structural snapshots; it cannot infer semantic intent from differences |
| "`alembic_version` is a history log" | Table contains 5 rows after 5 migrations are applied | Table contains 1 row with only the current head revision ID | Alembic reconstructs history from migration files on disk; it only needs to know "where am I now" |
| "Editing a migration re-applies it" | Fixing a file and running `upgrade head` applies the fix to the database | Nothing happens; Alembic sees the version is already applied and skips it | Alembic checks `version_num` match, never re-reads file contents for already-applied revisions |
| "NOT NULL column just works" | Adding NOT NULL column to populated table succeeds | Migration fails: existing rows violate the NOT NULL constraint | PostgreSQL enforces constraints on ALL existing rows immediately upon `ALTER TABLE` |
| "Downgrade restores everything" | `downgrade → upgrade` round-trip preserves all data | Column data destroyed by DROP; re-ADD creates column with NULL/default values only | `DROP COLUMN` is physically destructive; SQL has no built-in undo for data deletion |
| "Autogenerate on any DB is correct" | Running autogenerate on a stale DB produces migration with only new changes | Migration includes both new changes AND already-existing migration changes as duplicates | Autogenerate compares models to the ACTUAL database state, not to the migration file history |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Rename That Destroys

You have a working `users` table managed by Alembic. The current migration history has one migration that created the table. The database contains real data:

```sql
-- Current database state (migration 001 applied)
SELECT * FROM users;

 id |  name   |  role   |       email
----+---------+---------+-------------------
  1 | Alice   | admin   | alice@company.com
  2 | Bob     | editor  | bob@company.com
  3 | Charlie | viewer  | charlie@company.com
(3 rows)
```

Your tech lead asks you to rename `role` to `access_level` for consistency with the new permissions system. You update the model:

```python
# models.py — BEFORE
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    role: Mapped[str] = mapped_column(String(50))
    email: Mapped[str] = mapped_column(String(200))


# models.py — AFTER (column renamed)
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    access_level: Mapped[str | None] = mapped_column(String(50), nullable=True)
    email: Mapped[str] = mapped_column(String(200))
```

You run `alembic revision --autogenerate -m "rename role to access_level"`. Alembic generates this:

```python
"""rename role to access_level

Revision ID: b7c8d9e0
Revises: a1b2c3d4
"""
from alembic import op
import sqlalchemy as sa


revision: str = "b7c8d9e0"
down_revision: str = "a1b2c3d4"


def upgrade() -> None:
    op.add_column("users", sa.Column("access_level", sa.String(50), nullable=True))
    op.drop_column("users", "role")


def downgrade() -> None:
    op.add_column("users", sa.Column("role", sa.String(50), nullable=True))
    op.drop_column("users", "access_level")
```

You review the migration file, say "looks right — it removes `role` and adds `access_level`," and run `alembic upgrade head`.

**Questions:**

1. **Predict the output.** After `alembic upgrade head`, what does `SELECT id, name, access_level, email FROM users;` return? Specifically: what values are in the `access_level` column for each row?

2. **Explain why.** Your migration message says "rename role to access_level." Did a rename actually happen? What two *separate* operations did the database execute, and why does that distinction matter?

3. **Mental Trace:** Walk through what Alembic's autogenerate did when it compared your new model to the existing database. It sees the model has `access_level` but no `role`; the database has `role` but no `access_level`. Why can't autogenerate know this was a rename? What would the *correct* manual migration look like?

---

## Exercise 1.2: The Phantom Fix

Your team has a migration that created a `tasks` table. It was applied to your local database, your teammate's database, and the staging server two weeks ago.

```python
# Migration 001 — applied everywhere 2 weeks ago
"""create tasks table

Revision ID: aaa11111
Revises: None
"""
from alembic import op
import sqlalchemy as sa

revision: str = "aaa11111"
down_revision = None


def upgrade() -> None:
    op.create_table(
        "tasks",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("title", sa.String(50), nullable=False),   # BUG: should be 200!
        sa.Column("completed", sa.Boolean(), server_default="false"),
    )
    # Missing: should have created an index on 'title'


def downgrade() -> None:
    op.drop_table("tasks")
```

You notice two problems: `title` is `String(50)` but should be `String(200)`, and there's no index on `title`. You "fix" the migration file:

```python
def upgrade() -> None:
    op.create_table(
        "tasks",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("title", sa.String(200), nullable=False),   # "Fixed"!
        sa.Column("completed", sa.Boolean(), server_default="false"),
    )
    op.create_index("ix_tasks_title", "tasks", ["title"])     # "Added"!
```

You save the file and run these commands:

```bash
$ alembic current
aaa11111 (head)

$ alembic upgrade head
```

**Questions:**

1. **Predict the outcome.** After running `alembic upgrade head`:
   - Is the `title` column now `String(200)`?
   - Does the index `ix_tasks_title` exist?
   - What output did `alembic upgrade head` produce?

2. **Explain why.** You edited the migration file. You ran `upgrade head`. Why did nothing happen? What specific check does Alembic perform, and why does it skip this migration?

3. **Mental Trace:** Now consider what happens when a *new* teammate clones the repo tomorrow, creates a fresh database, and runs `alembic upgrade head` from scratch. Do THEY get `String(200)` and the index? Why does this create a situation where your database and their database have different schemas despite running the same migration files?

---

## Exercise 1.3: The Lost Pointer

Your project has three migrations applied to a production database:

```
Migration chain:
  (base) → aaa111 "create users" → bbb222 "add email column" → ccc333 "create posts" (head)
```

The database has real data: 50 users, 30 posts. Everything works perfectly.

During a late-night debugging session, a panicked developer runs this SQL directly against the database, thinking it would "reset" Alembic so they could start fresh:

```sql
DELETE FROM alembic_version;
```

The next morning, without realizing what happened, you run the standard deployment command:

```bash
$ alembic upgrade head
```

**Questions:**

1. **Predict the outcome.** What happens when `alembic upgrade head` runs? Does it:
   - (a) Do nothing, because it detects the tables already exist?
   - (b) Run all 3 migrations from scratch?
   - (c) Crash with an error? If so, on which migration, and what error?

2. **Explain why.** Alembic has migration files on disk that describe the full schema. The database already has the correct schema and data. Why can't Alembic figure out that everything is already in place?

3. **Mental Trace:** Walk through Alembic's decision process step-by-step:
   - Step 1: Read `alembic_version` table. What does it find?
   - Step 2: Determine what needs to be applied. Which migrations does it think are pending?
   - Step 3: Execute the first pending migration's `upgrade()`. What SQL does this generate? What happens when PostgreSQL tries to execute it?

---

## Exercise 1.4: The Stale Autogenerate

Your team has three migrations:

```python
# Migration 001 (aaa111): create tasks table
# Migration 002 (bbb222): add priority column
# Migration 003 (ccc333): add is_active column
def upgrade() -> None:
    op.add_column(
        "tasks",
        sa.Column("is_active", sa.Boolean(), server_default="true", nullable=False),
    )
```

Your teammate pushed migration 003 yesterday. You pulled the code (so you have the migration FILE), but you forgot to run `alembic upgrade head` on your local database.

Your database is at migration `bbb222` — it does NOT have the `is_active` column yet.

Now you need to add a `due_date` column. You update the model:

```python
# models.py — current state (includes all team changes + your new change)
class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    completed: Mapped[bool] = mapped_column(default=False)
    priority: Mapped[str] = mapped_column(String(10), server_default="medium")
    is_active: Mapped[bool] = mapped_column(default=True)            # From migration 003
    due_date: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)  # YOUR new column
```

You run:
```bash
$ alembic revision --autogenerate -m "add due_date to tasks"
```

**Questions:**

1. **Predict what the generated migration contains.** Does it contain:
   - (a) Only the `due_date` column addition?
   - (b) Both the `is_active` AND `due_date` column additions?
   - (c) Something else entirely?

2. **Explain why.** You only changed `due_date`, and `is_active` already has its own migration (003). Why would autogenerate include `is_active` in the new migration?

3. **The downstream disaster.** Suppose you don't notice the extra `is_active` in the generated migration. You apply your migration (004) locally — it adds both columns, which works because your DB didn't have either. Then you push the migration file. Your teammate (whose database already has `is_active` from migration 003) runs `alembic upgrade head` to get your migration 004. What happens?

---

# Level 1 Solutions

## Solution 1.1: The Rename That Destroys

**1. Predicted Output:**

```sql
SELECT id, name, access_level, email FROM users;

 id |  name   | access_level |       email
----+---------+--------------+-------------------
  1 | Alice   | NULL         | alice@company.com
  2 | Bob     | NULL         | bob@company.com
  3 | Charlie | NULL         | charlie@company.com
(3 rows)
```

Every single `access_level` value is `NULL`. The original role data — `admin`, `editor`, `viewer` — is gone. Permanently.

**2. Explain Why:**

No rename happened. The database executed two completely independent operations:

1. `ALTER TABLE users ADD COLUMN access_level VARCHAR(50)` → a brand-new empty column is added. All existing rows get `NULL` because no `server_default` was specified.
2. `ALTER TABLE users DROP COLUMN role` → the old column is physically removed from the table, along with every value it contained.

These two operations have no awareness of each other. PostgreSQL doesn't know that `access_level` was "supposed to" receive `role`'s data. From the database's perspective, you deleted one thing and created a completely different thing.

**3. Mental Trace:**

```
Autogenerate's comparison process:

1. Read model (Python):
   - users.id (Integer, PK)
   - users.name (String(100))
   - users.access_level (String(50))    ← in model
   - users.email (String(200))

2. Read database (PostgreSQL):
   - users.id (Integer, PK)
   - users.name (VARCHAR(100))
   - users.role (VARCHAR(50))            ← in database
   - users.email (VARCHAR(200))

3. Compute diff:
   - access_level: in model, NOT in database → ADD COLUMN
   - role: in database, NOT in model → DROP COLUMN
   
   Autogenerate has NO concept of "rename."
   It sees two independent facts: something is missing, something is extra.
```

**The correct manual migration for a rename:**

```python
def upgrade() -> None:
    op.alter_column("users", "role", new_column_name="access_level")

def downgrade() -> None:
    op.alter_column("users", "access_level", new_column_name="role")
```

This single `ALTER COLUMN RENAME` operation changes the column name in-place. All data is preserved. Autogenerate will **never** produce this — you must write it by hand.

**The Key Insight:**
Autogenerate compares snapshots, not intent. It can tell you WHAT is different between model and database, but not WHY. Any time the "why" matters (renames, data transformations, intentional restructuring), you must write the migration manually.

---

## Solution 1.2: The Phantom Fix

**1. Predicted Outcome:**

```bash
$ alembic upgrade head
# OUTPUT: (nothing — no migrations to run)
```

- The `title` column is still `String(50)`. **Not fixed.**
- The index `ix_tasks_title` does **not** exist. **Not added.**
- `alembic upgrade head` produces no output because Alembic believes there is nothing to do.

**2. Explain Why:**

Alembic's decision process:

```
1. Read alembic_version table → current version: "aaa11111"
2. Read migration files → head is "aaa11111"
3. Compare: current (aaa11111) == head (aaa11111)
4. Conclusion: "Database is already at head. Nothing to do."
```

Alembic **never re-reads the contents** of a migration file that has already been applied. It only checks the `version_num` string. Since `aaa11111` is recorded in `alembic_version`, Alembic considers it "done" regardless of whether the file's code has changed.

This is the same principle as Git commits: a Git commit hash identifies a snapshot. Changing a file after committing doesn't retroactively change the commit. In Alembic, the revision ID identifies a "database change event." Once that event is recorded as applied, modifying the script has no effect on databases that already ran it.

**3. Mental Trace — The Divergence Problem:**

```
YOUR database (ran original migration 2 weeks ago):
  - title: VARCHAR(50)
  - no index on title
  - alembic_version: "aaa11111"

NEW TEAMMATE'S database (clones repo, runs upgrade tomorrow):
  - title: VARCHAR(200)        ← They get the "fixed" version!
  - index ix_tasks_title exists ← They get the index!
  - alembic_version: "aaa11111"

SAME revision ID. DIFFERENT schema. Silent divergence.
```

Both databases report being at `aaa11111`, but they have fundamentally different structures. This divergence can cause bugs that only appear on some machines, making them extraordinarily difficult to diagnose.

**The Key Insight:**
Never edit an applied migration. The correct fix is to create a NEW migration:

```bash
$ alembic revision -m "fix tasks title length and add index"
```

```python
def upgrade() -> None:
    op.alter_column("tasks", "title", type_=sa.String(200))
    op.create_index("ix_tasks_title", "tasks", ["title"])

def downgrade() -> None:
    op.drop_index("ix_tasks_title", "tasks")
    op.alter_column("tasks", "title", type_=sa.String(50))
```

Now every database — old and new — applies the fix through the same forward path.

---

## Solution 1.3: The Lost Pointer

**1. Predicted Outcome:**

**(c) Crash with an error on the FIRST migration.**

```
INFO  [alembic.runtime.migration] Running upgrade  -> aaa111, create users
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.DuplicateTable)
relation "users" already exists
```

**2. Explain Why:**

Alembic does **not** inspect the actual database schema to decide what to run. It uses one — and only one — source of truth: the `alembic_version` table.

The migration files on disk describe what SHOULD happen. The `alembic_version` table records what HAS happened. Without `alembic_version`, Alembic has no memory. It's like deleting Git's `.git/HEAD` file — Git would have no idea what branch you're on or what's been committed.

The database has the correct tables, the correct columns, the correct data — but Alembic doesn't check any of that. It only checks `alembic_version`. Since that's empty, Alembic concludes: "No migrations have ever been applied. I must run everything from the beginning."

**3. Mental Trace:**

```
Step 1: Read alembic_version
  → Table exists but is EMPTY (row was deleted)
  → Alembic concludes: current_version = None (base)

Step 2: Determine pending migrations
  → Chain: (base) → aaa111 → bbb222 → ccc333
  → Current is (base), target is ccc333
  → Pending: ALL THREE migrations

Step 3: Execute aaa111.upgrade()
  → op.create_table("users", ...)
  → PostgreSQL: CREATE TABLE users (...)
  → PostgreSQL: ERROR — table "users" already exists!
  → Migration ABORTS. alembic_version remains empty.
  
  The 50 users and 30 posts are safe (nothing was dropped),
  but Alembic is now stuck — it can't move forward or backward.
```

**The Fix** — if this happens, you use `alembic stamp` to tell Alembic "the database is at this revision" without running any migrations:

```bash
# Tell Alembic: "Trust me, this database is at revision ccc333"
$ alembic stamp ccc333
```

This writes `ccc333` into `alembic_version` without executing any `upgrade()` functions. Alembic can now operate normally again.

**The Key Insight:**
`alembic_version` is a simple pointer, not a log. It doesn't record history — it records position. If the pointer is wrong or missing, Alembic will attempt to re-apply changes to a database that already has them, which will crash. The `stamp` command exists precisely for this recovery scenario: setting the pointer without running migrations.

---

## Solution 1.4: The Stale Autogenerate

**1. Predicted Contents:**

**(b) Both `is_active` AND `due_date` column additions.**

The generated migration looks like this:

```python
def upgrade() -> None:
    op.add_column("tasks", sa.Column("is_active", sa.Boolean(),
                                      server_default=sa.text("true"), nullable=False))
    op.add_column("tasks", sa.Column("due_date", sa.DateTime(), nullable=True))
```

**2. Explain Why:**

Autogenerate's comparison process:

```
MODEL says tasks should have:
  id, title, completed, priority, is_active, due_date

YOUR DATABASE actually has:
  id, title, completed, priority
  (is_active is NOT here — migration 003 was never applied locally!)

DIFF:
  - is_active: in model, not in database → ADD COLUMN
  - due_date: in model, not in database → ADD COLUMN
```

Autogenerate compares models to the **actual database**, not to the migration file history. It doesn't know or care that migration 003 already handles `is_active`. It sees the column is missing from your database and includes it.

**3. The Downstream Disaster:**

```
YOUR database (stale, at bbb222):
  $ alembic upgrade head
  → Runs migration 003: adds is_active  ✅
  → Runs migration 004: adds is_active AGAIN → ERROR! Column already exists!
```

OR, if you applied 004 locally first (skipping 003):

```
TEAMMATE'S database (up-to-date, at ccc333):
  $ alembic upgrade head
  → Runs migration 004: tries to add is_active
  → ERROR: column "is_active" of relation "tasks" already exists
```

Either way, someone's deployment breaks.

**The Key Insight:**
The correct workflow is ALWAYS:

```bash
$ git pull                                  # Get teammate's code
$ alembic upgrade head                      # Apply their migrations FIRST
$ # NOW your database matches the models minus your new change
$ alembic revision --autogenerate -m "add due_date to tasks"
```

By upgrading BEFORE generating, your database already has `is_active`, so autogenerate correctly produces only the `due_date` change.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Production Bomb

**Context:** A developer wrote this migration to support a new task assignment feature. It passed code review because the logic looks correct. It was tested locally on a fresh development database (created with `alembic upgrade head` from scratch, no data). It worked perfectly. It was deployed to production on Friday evening.

```python
"""add assignment fields to tasks

Revision ID: f1e2d3c4
Revises: prev_revision
"""
from alembic import op
import sqlalchemy as sa


revision: str = "f1e2d3c4"
down_revision: str = "prev_revision"


def upgrade() -> None:
    op.add_column(
        "tasks",
        sa.Column("assigned_to", sa.String(100), nullable=False),
    )
    op.add_column(
        "tasks",
        sa.Column("assigned_at", sa.DateTime(), nullable=False),
    )
    op.create_index(
        "ix_tasks_assigned_to",
        "tasks",
        ["assigned_to"],
    )


def downgrade() -> None:
    op.drop_index("ix_tasks_assigned_to", "tasks")
    op.drop_column("tasks", "assigned_at")
    op.drop_column("tasks", "assigned_to")
```

Production has 85,000 tasks. The deployment fails:

```
sqlalchemy.exc.IntegrityError: (psycopg2.errors.NotNullViolation)
column "assigned_to" of relation "tasks" contains null values
```

---

**Task 2a: Identify the Flaw**

1. Why does this migration succeed on an empty database but fail on production?
2. Which specific column(s) cause the failure, and what constraint is violated?
3. Why did `assigned_at` also fail (or would it fail if `assigned_to` didn't crash first)?

Do NOT fix the code yet. Just diagnose the problem precisely.

---

**Task 2b: Explain the Mechanism**

1. When PostgreSQL executes `ALTER TABLE tasks ADD COLUMN assigned_to VARCHAR(100) NOT NULL`, what does it attempt to do to all 85,000 existing rows?
2. Why is `nullable=False` without `server_default` a time bomb for populated tables?
3. What is the difference between Python-level defaults (in the SQLAlchemy model) and `server_default` (in the migration)? Which one matters here, and why?

---

**Task 2c: Instrument to Prove Diagnosis**

Before running the migration, write a diagnostic query that would predict whether this migration will succeed or fail on a given database:

```sql
-- Write a query that checks if the migration can safely run
-- Your query should return a clear "SAFE" or "UNSAFE" result
-- Hint: what conditions make NOT NULL columns dangerous?
```

Show what this query returns on an empty database vs. on production.

---

**Task 2d: Fix the Code**

Fix the migration using the **three-phase approach** so that:
- All 85,000 existing tasks get a sensible default value
- New tasks inserted after the migration require explicit values
- The migration works on BOTH empty databases and populated databases
- The `downgrade()` properly reverses all changes

Write the corrected migration file.

---

**Task 2e: Write Verification Test**

Write a test that:
- **FAILS** against the original (broken) migration
- **PASSES** against your fixed migration

The test must:
1. Create the tasks table (using a preceding migration or raw SQL)
2. Insert sample data (at least 3 rows, no `assigned_to` values)
3. Apply the migration
4. Verify the column exists with correct defaults for existing rows
5. Verify the round-trip: `upgrade → downgrade → upgrade`

```python
# Skeleton — fill in the implementation
import pytest
from sqlalchemy import create_engine, text, inspect
from alembic.config import Config
from alembic import command


def test_migration_handles_existing_data(test_db_engine):
    """Migration must succeed when table already contains data."""
    with test_db_engine.connect() as conn:
        # Step 1: Create tasks table with data (simulating pre-migration state)
        conn.execute(text("""
            CREATE TABLE IF NOT EXISTS tasks (
                id SERIAL PRIMARY KEY,
                title VARCHAR(200) NOT NULL,
                completed BOOLEAN DEFAULT false
            )
        """))
        conn.execute(text("""
            INSERT INTO tasks (title) VALUES
            ('Task A'), ('Task B'), ('Task C')
        """))
        conn.commit()

        # Step 2: Apply migration — this is where the original FAILS
        # YOUR CODE: apply the migration here

        # Step 3: Verify existing rows got default values
        result = conn.execute(text(
            "SELECT assigned_to, assigned_at FROM tasks"
        ))
        rows = result.fetchall()
        for row in rows:
            assert row[0] is not None, "assigned_to must not be NULL"
            assert row[1] is not None, "assigned_at must not be NULL"

        # Step 4: Verify round-trip works
        # YOUR CODE: downgrade, then upgrade again
```

---

## Exercise 2.2: The Silent Data Corruption

**Context:** Your team is converting the `priority` column from free-text `VARCHAR` to a strict `INTEGER` for better sorting and validation. A developer wrote this data migration. It was applied to staging and production. No errors were reported.

Three weeks later, a product manager notices that 847 tasks show "No priority assigned" in the UI — but the original CSV import confirmed all 10,000 tasks had priority values. Something silently went wrong.

```python
"""convert priority from string to integer

Revision ID: d5e6f7g8
Revises: c4d5e6f7
"""
from alembic import op
import sqlalchemy as sa


revision: str = "d5e6f7g8"
down_revision: str = "c4d5e6f7"


def upgrade() -> None:
    # Step 1: Add new integer column
    op.add_column(
        "tasks",
        sa.Column("priority_int", sa.Integer(), nullable=True),
    )

    # Step 2: Migrate data from string to integer
    op.execute("""
        UPDATE tasks SET priority_int = CASE
            WHEN priority = 'high' THEN 3
            WHEN priority = 'medium' THEN 2
            WHEN priority = 'low' THEN 1
        END
    """)

    # Step 3: Swap columns
    op.drop_column("tasks", "priority")
    op.alter_column("tasks", "priority_int", new_column_name="priority")


def downgrade() -> None:
    op.alter_column("tasks", "priority", new_column_name="priority_int")
    op.add_column(
        "tasks",
        sa.Column("priority", sa.String(10), nullable=True),
    )
    op.execute("""
        UPDATE tasks SET priority = CASE
            WHEN priority_int = 3 THEN 'high'
            WHEN priority_int = 2 THEN 'medium'
            WHEN priority_int = 1 THEN 'low'
        END
    """)
    op.drop_column("tasks", "priority_int")
```

Here is a sample of the actual production data *before* the migration was applied:

```sql
SELECT DISTINCT priority, COUNT(*) FROM tasks GROUP BY priority;

 priority  | count
-----------+-------
 high      |  2103
 medium    |  4950
 low       |  2100
 High      |   312
 MEDIUM    |    89
 urgent    |   201
 critical  |   145
 NULL      |    47
            |    53    -- (empty string)
(9 rows)
```

---

**Task 2a: Identify the Flaw**

1. The migration applies without any error on this data. Why?
2. How many of the 10,000 tasks end up with `NULL` in the `priority` (integer) column after the migration? Which rows are affected, and why?
3. Why is "silent success" more dangerous than a crash here?

---

**Task 2b: Explain the Mechanism**

1. Walk through what the SQL `CASE` statement does for a row where `priority = 'High'` (capital H). What value does `priority_int` receive?
2. What does a SQL `CASE` statement return when none of the `WHEN` conditions match and there is no `ELSE` clause?
3. Why doesn't PostgreSQL raise an error or warning when rows get NULL from a missing `ELSE`?

---

**Task 2c: Instrument to Prove Diagnosis**

Write a diagnostic query that you should run BETWEEN Step 1 and Step 3 (after the UPDATE but before the DROP) to detect the corruption before it becomes permanent:

```sql
-- Write a query that shows rows where data migration failed
-- This query should be part of every data migration's verification step
```

Show exactly what this query would output for the production data above.

---

**Task 2d: Fix the Code**

Rewrite the migration so that:
- Case-insensitive matching handles `'High'`, `'HIGH'`, `'hIgH'`, etc.
- Legacy values (`'urgent'`, `'critical'`) are mapped to appropriate integers
- NULL values are explicitly handled
- Empty strings are explicitly handled
- **No row is left with an unintentional NULL after conversion**
- A verification step INSIDE the migration detects and aborts if any rows fall through the cracks
- The downgrade properly reverses the conversion, including the legacy values

---

**Task 2e: Write Verification Test**

Write a test that:
- **FAILS** against the original migration (detects the silent NULLs)
- **PASSES** against your fixed migration

```python
def test_no_data_lost_in_priority_conversion(test_db_engine):
    """Every non-NULL priority string must convert to a non-NULL integer."""
    with test_db_engine.connect() as conn:
        # Setup: create table with messy real-world data
        conn.execute(text("""
            CREATE TABLE tasks (
                id SERIAL PRIMARY KEY,
                title VARCHAR(200) NOT NULL,
                priority VARCHAR(10)
            )
        """))
        conn.execute(text("""
            INSERT INTO tasks (title, priority) VALUES
            ('Task 1', 'high'),
            ('Task 2', 'High'),
            ('Task 3', 'MEDIUM'),
            ('Task 4', 'urgent'),
            ('Task 5', 'critical'),
            ('Task 6', ''),
            ('Task 7', NULL)
        """))
        conn.commit()

        # Apply migration
        # YOUR CODE: apply the migration

        # Verify: every row that HAD a non-null, non-empty priority
        # MUST have a non-null integer priority after conversion
        result = conn.execute(text("""
            SELECT COUNT(*) FROM tasks
            WHERE priority IS NULL
        """))
        null_count = result.scalar()

        # Only Task 6 (empty string) and Task 7 (NULL) should be NULL
        # If null_count > 2, the migration silently lost data
        assert null_count <= 2, (
            f"Expected at most 2 NULL priorities (empty string + original NULL), "
            f"got {null_count}. Data was silently corrupted!"
        )
```

---

# Level 2 Solutions

## Solution 2.1: The Production Bomb

**Task 2a: Identify the Flaw**

1. On an **empty** database, `ALTER TABLE ADD COLUMN ... NOT NULL` succeeds because there are no rows to violate the constraint. On production, 85,000 rows have no value for `assigned_to` — PostgreSQL immediately checks every existing row and finds them all violating `NOT NULL`.

2. `assigned_to` causes the crash first (columns are processed in order). `assigned_at` would cause the same crash if it ran — it's also `nullable=False` with no `server_default`.

3. Both columns have the same defect: `NOT NULL` with no `server_default` means existing rows have no valid value for the new column, which PostgreSQL rejects.

**Task 2b: Explain the Mechanism**

1. PostgreSQL processes `ADD COLUMN ... NOT NULL` by logically checking every existing row. Each of the 85,000 rows would need a value for `assigned_to`. Since no `server_default` is provided, the value for all existing rows would be `NULL`, which violates `NOT NULL`.

2. `nullable=False` without `server_default` is safe only for brand-new tables (no rows to check). The moment any row exists, PostgreSQL enforces the constraint retroactively on all existing rows.

3. Python-level defaults (like `default=` in a SQLAlchemy model) only apply when you create objects through the ORM. Migrations don't use the ORM — they issue raw DDL. `server_default` tells PostgreSQL itself to provide a value, which is what `ALTER TABLE` uses.

**Task 2c: Diagnostic Query**

```sql
-- Run BEFORE applying the migration
SELECT
    CASE
        WHEN COUNT(*) = 0 THEN 'SAFE: table is empty, NOT NULL without default will work'
        ELSE 'UNSAFE: table has ' || COUNT(*) || ' rows — NOT NULL without server_default will FAIL'
    END AS migration_risk
FROM tasks;
```

On empty database: `SAFE: table is empty, NOT NULL without default will work`
On production: `UNSAFE: table has 85000 rows — NOT NULL without server_default will FAIL`

**Task 2d: Fixed Migration**

```python
from datetime import datetime, timezone

def upgrade() -> None:
    # PHASE 1: Add columns as NULLABLE (safe for existing rows)
    op.add_column(
        "tasks",
        sa.Column("assigned_to", sa.String(100), nullable=True),
    )
    op.add_column(
        "tasks",
        sa.Column("assigned_at", sa.DateTime(), nullable=True),
    )

    # PHASE 2: Backfill existing rows with sensible defaults
    op.execute("""
        UPDATE tasks
        SET assigned_to = 'unassigned',
            assigned_at = created_at
        WHERE assigned_to IS NULL
    """)

    # PHASE 3: Now that all rows have values, enforce NOT NULL
    op.alter_column("tasks", "assigned_to", nullable=False)
    op.alter_column("tasks", "assigned_at", nullable=False)

    # PHASE 4: Add index (after data is populated)
    op.create_index("ix_tasks_assigned_to", "tasks", ["assigned_to"])


def downgrade() -> None:
    op.drop_index("ix_tasks_assigned_to", "tasks")
    op.drop_column("tasks", "assigned_at")
    op.drop_column("tasks", "assigned_to")
```

**Why this works:**
- Phase 1: Nullable columns always succeed — existing rows simply get `NULL`.
- Phase 2: `UPDATE` replaces all `NULL`s with explicit values.
- Phase 3: `ALTER COLUMN SET NOT NULL` now succeeds because no `NULL`s remain.
- Phase 4: Index creation is the same as before.

The three-phase pattern (add nullable → backfill → set NOT NULL) is the standard safe approach for adding required columns to populated tables.

**Task 2e: Verification Test**

```python
def test_migration_handles_existing_data(test_db_engine):
    """Migration must succeed when table already contains data."""
    with test_db_engine.connect() as conn:
        # Step 1: Create pre-existing state
        conn.execute(text("""
            CREATE TABLE IF NOT EXISTS tasks (
                id SERIAL PRIMARY KEY,
                title VARCHAR(200) NOT NULL,
                completed BOOLEAN DEFAULT false,
                created_at TIMESTAMP DEFAULT now()
            )
        """))
        conn.execute(text("""
            INSERT INTO tasks (title) VALUES
            ('Existing Task A'), ('Existing Task B'), ('Existing Task C')
        """))
        conn.commit()

        # Step 2: Apply the fixed migration
        # (In a real test, use Alembic's command API)
        # For this test, we simulate the migration directly:
        conn.execute(text(
            "ALTER TABLE tasks ADD COLUMN assigned_to VARCHAR(100)"
        ))
        conn.execute(text(
            "ALTER TABLE tasks ADD COLUMN assigned_at TIMESTAMP"
        ))
        conn.execute(text("""
            UPDATE tasks
            SET assigned_to = 'unassigned',
                assigned_at = created_at
            WHERE assigned_to IS NULL
        """))
        conn.execute(text(
            "ALTER TABLE tasks ALTER COLUMN assigned_to SET NOT NULL"
        ))
        conn.execute(text(
            "ALTER TABLE tasks ALTER COLUMN assigned_at SET NOT NULL"
        ))
        conn.commit()

        # Step 3: Verify no NULLs remain
        result = conn.execute(text("""
            SELECT COUNT(*) FROM tasks
            WHERE assigned_to IS NULL OR assigned_at IS NULL
        """))
        null_count = result.scalar()
        assert null_count == 0, f"Found {null_count} rows with NULL assignment fields"

        # Step 4: Verify existing rows got sensible defaults
        result = conn.execute(text("""
            SELECT assigned_to FROM tasks WHERE title = 'Existing Task A'
        """))
        assert result.scalar() == "unassigned"

        # Step 5: Verify NOT NULL is enforced for new inserts
        with pytest.raises(Exception):
            conn.execute(text(
                "INSERT INTO tasks (title) VALUES ('New Task Without Assignment')"
            ))

    print("Test PASSED: Migration handles existing data correctly.")
```

---

## Solution 2.2: The Silent Data Corruption

**Task 2a: Identify the Flaw**

1. The migration applies without error because a SQL `CASE` expression with no matching `WHEN` and no `ELSE` clause silently returns `NULL`. PostgreSQL doesn't raise warnings for this.

2. **847 tasks** end up with `NULL`:
   - `'High'` (312 rows): Case-sensitive mismatch — `'High' != 'high'`
   - `'MEDIUM'` (89 rows): Case-sensitive mismatch
   - `'urgent'` (201 rows): Not in any `WHEN` clause at all
   - `'critical'` (145 rows): Not in any `WHEN` clause at all
   - `''` empty string (53 rows): Not in any `WHEN` clause
   - `NULL` (47 rows): `WHEN priority = 'high'` never matches `NULL` (NULL != anything)
   - Total: 312 + 89 + 201 + 145 + 53 + 47 = **847 rows with NULL**

3. A crash is visible, immediate, and blocks deployment. Silent corruption passes every check, gets deployed, and poisons data for weeks before anyone notices. By then, the original `VARCHAR` column is long gone (dropped in Step 3), so recovery requires a database backup.

**Task 2b: Explain the Mechanism**

1. For `priority = 'High'`: The `CASE` evaluates `'High' = 'high'` → `FALSE` (SQL string comparison is case-sensitive by default in PostgreSQL). Then `'High' = 'medium'` → `FALSE`. Then `'High' = 'low'` → `FALSE`. No match. No `ELSE`. Result: `NULL`.

2. The SQL standard defines: a `CASE` expression with no matching `WHEN` and no `ELSE` clause returns `NULL`. This is by design, not an error.

3. PostgreSQL follows the standard faithfully. `NULL` is a valid value for a nullable column. No rule is broken, so no error is raised. The data is silently wrong, but structurally valid.

**Task 2c: Diagnostic Query**

```sql
-- Run AFTER the UPDATE but BEFORE the DROP
-- This should be a mandatory checkpoint in every data migration

SELECT
    priority AS original_value,
    priority_int AS converted_value,
    COUNT(*) AS row_count
FROM tasks
WHERE priority_int IS NULL
  AND priority IS NOT NULL
  AND priority != ''
GROUP BY priority, priority_int
ORDER BY row_count DESC;
```

**Output on production data:**

```
 original_value | converted_value | row_count
----------------+-----------------+-----------
 High           | NULL            |       312
 urgent         | NULL            |       201
 critical       | NULL            |       145
 MEDIUM         | NULL            |        89
(4 rows)
```

This immediately reveals that 747 rows with actual priority values failed to convert. If this query returns any rows, the migration should be aborted before the destructive `drop_column` step.

**Task 2d: Fixed Migration**

```python
def upgrade() -> None:
    # Step 1: Add new integer column
    op.add_column(
        "tasks",
        sa.Column("priority_int", sa.Integer(), nullable=True),
    )

    # Step 2: Migrate data — case-insensitive, handles all known values
    op.execute("""
        UPDATE tasks SET priority_int = CASE
            WHEN LOWER(TRIM(priority)) = 'high'     THEN 3
            WHEN LOWER(TRIM(priority)) = 'critical'  THEN 3
            WHEN LOWER(TRIM(priority)) = 'urgent'    THEN 3
            WHEN LOWER(TRIM(priority)) = 'medium'    THEN 2
            WHEN LOWER(TRIM(priority)) = 'low'       THEN 1
            WHEN priority IS NULL                     THEN NULL
            WHEN TRIM(priority) = ''                  THEN NULL
            ELSE -1
        END
    """)

    # Step 3: VERIFICATION — abort if any unexpected values exist
    # The ELSE -1 acts as a sentinel: if any rows have -1,
    # we have unhandled priority values that need mapping.
    op.execute("""
        DO $$
        DECLARE
            bad_count INTEGER;
        BEGIN
            SELECT COUNT(*) INTO bad_count
            FROM tasks WHERE priority_int = -1;

            IF bad_count > 0 THEN
                RAISE EXCEPTION 'Data migration failed: % rows have unmapped priority values. '
                    'Query: SELECT DISTINCT priority FROM tasks WHERE priority_int = -1',
                    bad_count;
            END IF;
        END $$;
    """)

    # Step 4: Swap columns (only reached if verification passes)
    op.drop_column("tasks", "priority")
    op.alter_column("tasks", "priority_int", new_column_name="priority")


def downgrade() -> None:
    op.alter_column("tasks", "priority", new_column_name="priority_int")
    op.add_column(
        "tasks",
        sa.Column("priority", sa.String(10), nullable=True),
    )
    op.execute("""
        UPDATE tasks SET priority = CASE
            WHEN priority_int = 3 THEN 'high'
            WHEN priority_int = 2 THEN 'medium'
            WHEN priority_int = 1 THEN 'low'
            ELSE NULL
        END
    """)
    op.drop_column("tasks", "priority_int")
```

**Key improvements:**
- `LOWER(TRIM(...))` handles case variations and whitespace
- Legacy values (`urgent`, `critical`) are explicitly mapped
- `NULL` and empty strings are explicitly handled
- `ELSE -1` sentinel catches ANY unmapped value
- Verification step **inside the migration** aborts before the destructive DROP if any data fell through

**Task 2e: Verification Test**

```python
def test_no_data_lost_in_priority_conversion(test_db_engine):
    """Every non-NULL, non-empty priority string must convert to a non-NULL integer."""
    with test_db_engine.connect() as conn:
        conn.execute(text("""
            CREATE TABLE tasks (
                id SERIAL PRIMARY KEY,
                title VARCHAR(200) NOT NULL,
                priority VARCHAR(10)
            )
        """))
        conn.execute(text("""
            INSERT INTO tasks (title, priority) VALUES
            ('Task 1', 'high'),
            ('Task 2', 'High'),
            ('Task 3', 'MEDIUM'),
            ('Task 4', 'urgent'),
            ('Task 5', 'critical'),
            ('Task 6', ''),
            ('Task 7', NULL),
            ('Task 8', ' medium ')
        """))
        conn.commit()

        # Apply migration (simulate or use Alembic command API)
        # ... apply migration steps here ...

        # Verify: rows with meaningful priorities must have non-NULL integer
        result = conn.execute(text("""
            SELECT COUNT(*) FROM tasks
            WHERE priority IS NULL
        """))
        null_count = result.scalar()

        # Only Task 6 (empty) and Task 7 (NULL) should remain NULL = 2
        assert null_count == 2, (
            f"Expected 2 NULL priorities, got {null_count}. "
            f"Data was silently corrupted!"
        )

        # Verify specific conversions
        result = conn.execute(text(
            "SELECT priority FROM tasks WHERE title = 'Task 2'"
        ))
        assert result.scalar() == 3, "'High' should convert to 3"

        result = conn.execute(text(
            "SELECT priority FROM tasks WHERE title = 'Task 4'"
        ))
        assert result.scalar() == 3, "'urgent' should convert to 3"

        result = conn.execute(text(
            "SELECT priority FROM tasks WHERE title = 'Task 8'"
        ))
        assert result.scalar() == 2, "' medium ' (with spaces) should convert to 2"

    print("Test PASSED: No data lost in priority conversion.")
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Setup: The Working Task Manager Migrations

You have a working Task Manager with three migrations and a test suite that passes. Your job is to **extend** this system with new features, **without breaking existing tests or migrations.**

### Existing Migration Chain

```python
# ============================================================
# Migration 001 (aaa11111): create tasks table
# ============================================================
"""create tasks table"""
revision: str = "aaa11111"
down_revision = None

def upgrade() -> None:
    op.create_table(
        "tasks",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("title", sa.String(200), nullable=False),
        sa.Column("completed", sa.Boolean(), server_default="false"),
        sa.Column("created_at", sa.DateTime(), server_default=sa.func.now()),
    )

def downgrade() -> None:
    op.drop_table("tasks")


# ============================================================
# Migration 002 (bbb22222): add priority column
# ============================================================
"""add priority column to tasks"""
revision: str = "bbb22222"
down_revision: str = "aaa11111"

def upgrade() -> None:
    op.add_column(
        "tasks",
        sa.Column("priority", sa.String(10), server_default="medium", nullable=False),
    )

def downgrade() -> None:
    op.drop_column("tasks", "priority")


# ============================================================
# Migration 003 (ccc33333): create categories
# ============================================================
"""create categories table and add FK to tasks"""
revision: str = "ccc33333"
down_revision: str = "bbb22222"

def upgrade() -> None:
    op.create_table(
        "categories",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("name", sa.String(100), nullable=False, unique=True),
    )
    op.add_column(
        "tasks",
        sa.Column("category_id", sa.Integer(), nullable=True),
    )
    op.create_foreign_key(
        "fk_tasks_category_id", "tasks", "categories",
        ["category_id"], ["id"],
    )

def downgrade() -> None:
    op.drop_constraint("fk_tasks_category_id", "tasks", type_="foreignkey")
    op.drop_column("tasks", "category_id")
    op.drop_table("categories")
```

### Existing Test Suite (Must Continue to Pass)

```python
# test_migrations.py — DO NOT MODIFY THESE TESTS
import pytest
from sqlalchemy import create_engine, text, inspect

@pytest.fixture
def migrated_engine(tmp_postgres_db):
    """Apply all migrations to a fresh database, return engine."""
    engine = create_engine(tmp_postgres_db)
    # (Apply all migrations using Alembic command API)
    return engine


def test_tasks_table_structure(migrated_engine):
    """Tasks table must have expected columns."""
    inspector = inspect(migrated_engine)
    columns = {c["name"] for c in inspector.get_columns("tasks")}
    assert columns >= {"id", "title", "completed", "created_at", "priority", "category_id"}


def test_categories_table_exists(migrated_engine):
    """Categories table must exist with correct structure."""
    inspector = inspect(migrated_engine)
    columns = {c["name"] for c in inspector.get_columns("categories")}
    assert columns >= {"id", "name"}


def test_foreign_key_exists(migrated_engine):
    """Tasks must have FK to categories."""
    inspector = inspect(migrated_engine)
    fks = inspector.get_foreign_keys("tasks")
    fk_columns = [fk["constrained_columns"] for fk in fks]
    assert ["category_id"] in fk_columns


def test_full_round_trip(migrated_engine):
    """upgrade head → downgrade base → upgrade head must work."""
    # (Run alembic downgrade base)
    # (Run alembic upgrade head)
    # (Verify tasks and categories tables exist)
    inspector = inspect(migrated_engine)
    tables = inspector.get_table_names()
    assert "tasks" in tables
    assert "categories" in tables


def test_data_survives_new_migrations(migrated_engine):
    """Seed data must persist through new migrations."""
    with migrated_engine.connect() as conn:
        conn.execute(text("""
            INSERT INTO categories (name) VALUES ('Work'), ('Personal')
        """))
        conn.execute(text("""
            INSERT INTO tasks (title, priority, category_id)
            VALUES ('Test task', 'high', 1)
        """))
        conn.commit()

        result = conn.execute(text("SELECT COUNT(*) FROM tasks"))
        assert result.scalar() == 1
        result = conn.execute(text("SELECT COUNT(*) FROM categories"))
        assert result.scalar() == 2
```

---

### Extension A: Add a Tags System (Many-to-Many)

**Requirement:** Tasks should support multiple tags (e.g., "bug", "feature", "documentation"). A tag can be applied to multiple tasks, and a task can have multiple tags. This requires a `tags` table and a `task_tags` junction table.

Write **one migration** (migration 004) that:
1. Creates a `tags` table with `id` (Integer, PK) and `name` (String(50), unique, NOT NULL)
2. Creates a `task_tags` junction table with:
   - `task_id` (Integer, FK → tasks.id, NOT NULL)
   - `tag_id` (Integer, FK → tags.id, NOT NULL)
   - Composite primary key on `(task_id, tag_id)`
3. Has a working `downgrade()` — consider: which constraint must be dropped first?

**Your test for Extension A:**
```python
def test_tags_many_to_many(migrated_engine):
    """Tags system must support many-to-many relationship."""
    with migrated_engine.connect() as conn:
        conn.execute(text("INSERT INTO categories (name) VALUES ('Work')"))
        conn.execute(text(
            "INSERT INTO tasks (title, priority, category_id) VALUES ('Task 1', 'high', 1)"
        ))
        conn.execute(text("INSERT INTO tags (name) VALUES ('bug'), ('feature')"))
        conn.execute(text("INSERT INTO task_tags (task_id, tag_id) VALUES (1, 1), (1, 2)"))
        conn.commit()

        result = conn.execute(text("""
            SELECT t.name FROM tags t
            JOIN task_tags tt ON tt.tag_id = t.id
            WHERE tt.task_id = 1
            ORDER BY t.name
        """))
        tag_names = [row[0] for row in result]
        assert tag_names == ["bug", "feature"]
```

---

### Extension B: Add Due Date with Data Backfill

**Requirement:** Tasks need a `due_date` column. Existing tasks should have `due_date` set to `created_at + 7 days` (a sensible default for historical data). New tasks should be allowed to have NULL due dates.

Write **one migration** (migration 005) that:
1. Adds `due_date` (DateTime, nullable) to tasks
2. Backfills existing rows: `due_date = created_at + INTERVAL '7 days'`
3. Does NOT make the column NOT NULL (new tasks may not have a due date)
4. Has a working `downgrade()` that drops the column

**Constraint:** This is a data migration. Autogenerate cannot produce the backfill. You must write it by hand.

**Your test for Extension B:**
```python
def test_due_date_backfill(migrated_engine):
    """Existing tasks must have due_date = created_at + 7 days."""
    with migrated_engine.connect() as conn:
        # Insert a task BEFORE migration 005
        # (In a real test, apply migrations up to 004, insert data, then apply 005)
        conn.execute(text("""
            INSERT INTO categories (name) VALUES ('Test')
        """))
        conn.execute(text("""
            INSERT INTO tasks (title, priority, category_id, created_at)
            VALUES ('Old task', 'medium', 1, '2025-01-01 00:00:00')
        """))
        conn.commit()

        # Apply migration 005 backfill
        conn.execute(text("""
            UPDATE tasks SET due_date = created_at + INTERVAL '7 days'
            WHERE due_date IS NULL
        """))
        conn.commit()

        result = conn.execute(text(
            "SELECT due_date FROM tasks WHERE title = 'Old task'"
        ))
        due_date = result.scalar()
        assert str(due_date) == "2025-01-08 00:00:00", f"Expected 2025-01-08, got {due_date}"
```

---

### Extension C: Add Soft Delete with Partial Index

**Requirement:** Instead of permanently deleting tasks, we want to "soft delete" them — set a `deleted_at` timestamp. To keep queries fast, create a partial index that covers only non-deleted tasks.

Write **one migration** (migration 006) that:
1. Adds `deleted_at` (DateTime, nullable, default NULL) to tasks
2. Creates a partial index: `CREATE INDEX ix_tasks_active ON tasks (id) WHERE deleted_at IS NULL`
3. Has a working `downgrade()`

**Constraint:** The `op.create_index()` function in Alembic supports partial indexes through the `postgresql_where` parameter. You must use `op.create_index()` (not raw SQL) for the partial index.

**Your test for Extension C:**
```python
def test_soft_delete_index(migrated_engine):
    """Partial index must exist on non-deleted tasks."""
    inspector = inspect(migrated_engine)
    indexes = inspector.get_indexes("tasks")
    partial_indexes = [idx for idx in indexes if idx.get("name") == "ix_tasks_active"]
    assert len(partial_indexes) == 1, "Partial index ix_tasks_active must exist"

def test_soft_delete_workflow(migrated_engine):
    """Soft-deleted tasks should be excluded by the index."""
    with migrated_engine.connect() as conn:
        conn.execute(text("INSERT INTO categories (name) VALUES ('Work')"))
        conn.execute(text("""
            INSERT INTO tasks (title, priority, category_id) VALUES
            ('Active task', 'high', 1),
            ('Deleted task', 'low', 1)
        """))
        conn.execute(text("""
            UPDATE tasks SET deleted_at = now() WHERE title = 'Deleted task'
        """))
        conn.commit()

        result = conn.execute(text(
            "SELECT COUNT(*) FROM tasks WHERE deleted_at IS NULL"
        ))
        assert result.scalar() == 1, "Only one task should be 'active'"
```

---

# Level 2.5 Solutions

## Solution A: Tags System

```python
"""add tags system with many-to-many

Revision ID: ddd44444
Revises: ccc33333
"""
revision: str = "ddd44444"
down_revision: str = "ccc33333"


def upgrade() -> None:
    # 1. Create tags table
    op.create_table(
        "tags",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("name", sa.String(50), nullable=False, unique=True),
    )

    # 2. Create junction table with composite PK
    op.create_table(
        "task_tags",
        sa.Column("task_id", sa.Integer(), sa.ForeignKey("tasks.id"), nullable=False),
        sa.Column("tag_id", sa.Integer(), sa.ForeignKey("tags.id"), nullable=False),
        sa.PrimaryKeyConstraint("task_id", "tag_id"),
    )


def downgrade() -> None:
    # Drop junction table FIRST (has FKs pointing to tags and tasks)
    op.drop_table("task_tags")
    # Then drop tags table (now nothing references it)
    op.drop_table("tags")
```

**Why the downgrade order matters:** `task_tags` has foreign keys pointing to `tags`. If you try to drop `tags` first, PostgreSQL blocks it: "cannot drop table tags because other objects depend on it." The junction table must be dropped first to remove the FK dependencies.

This is the same principle from the lecture: when building UP, create referenced tables first (tags before task_tags). When tearing DOWN, drop referencing tables first (task_tags before tags).

---

## Solution B: Due Date with Backfill

```python
"""add due_date column with backfill

Revision ID: eee55555
Revises: ddd44444
"""
revision: str = "eee55555"
down_revision: str = "ddd44444"


def upgrade() -> None:
    # 1. Add nullable column (safe for existing data)
    op.add_column(
        "tasks",
        sa.Column("due_date", sa.DateTime(), nullable=True),
    )

    # 2. Backfill: existing tasks get due_date = created_at + 7 days
    # This is a DATA migration — autogenerate cannot produce this
    op.execute("""
        UPDATE tasks
        SET due_date = created_at + INTERVAL '7 days'
        WHERE due_date IS NULL AND created_at IS NOT NULL
    """)


def downgrade() -> None:
    op.drop_column("tasks", "due_date")
```

**Why not make it NOT NULL?** The requirement says new tasks may not have a due date. If we set `nullable=False`, every future INSERT would require a `due_date`, which contradicts the business requirement. The backfill is a one-time historical correction, not an ongoing constraint.

**Why autogenerate can't help:** Autogenerate would produce `op.add_column(...)` but never the `UPDATE` statement. The backfill is business logic ("7 days after creation") that no tool can infer from a model definition.

---

## Solution C: Soft Delete with Partial Index

```python
"""add soft delete support with partial index

Revision ID: fff66666
Revises: eee55555
"""
revision: str = "fff66666"
down_revision: str = "eee55555"


def upgrade() -> None:
    op.add_column(
        "tasks",
        sa.Column("deleted_at", sa.DateTime(), nullable=True),
    )

    # Partial index: only indexes rows WHERE deleted_at IS NULL
    # This makes "get all active tasks" queries fast without indexing deleted rows
    op.create_index(
        "ix_tasks_active",
        "tasks",
        ["id"],
        postgresql_where=sa.text("deleted_at IS NULL"),
    )


def downgrade() -> None:
    op.drop_index("ix_tasks_active", "tasks")
    op.drop_column("tasks", "deleted_at")
```

**Why a partial index?** Without it, every query for active tasks (`WHERE deleted_at IS NULL`) scans the full table. The partial index tells PostgreSQL: "pre-compute which rows have `deleted_at IS NULL` and maintain a small, fast index over just those rows." As tasks are soft-deleted, they automatically fall out of the index.

**Why `postgresql_where` instead of raw SQL?** Using `op.create_index` with `postgresql_where` is the Alembic-native way to create partial indexes. It generates the correct `CREATE INDEX ... WHERE ...` SQL while keeping the migration portable and auditable. Alembic can also properly detect and drop this index in the downgrade.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## The Legacy Database Adoption

**Scenario:** You've been hired to modernize a 5-year-old application. The existing database was created with raw SQL scripts — there is no migration history, no Alembic, no SQLAlchemy. The database has been running in production for years with real data.

Here is the existing schema and sample data:

```sql
-- The legacy schema (created years ago, no Alembic)
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price VARCHAR(50),           -- "$19.99" format — string, not numeric!
    category TEXT,               -- Freeform text, not normalized
    in_stock VARCHAR(5)          -- "yes" or "no" instead of BOOLEAN
);

-- Real production data (5 representative rows of 10,000)
INSERT INTO products (name, price, category, in_stock) VALUES
    ('Wireless Mouse',     '$29.99',  'Electronics',    'yes'),
    ('USB Cable',          '$5.50',   'electronics',    'no'),
    ('Desk Lamp',          '',        'Home & Garden',  'yes'),
    ('Standing Desk',      NULL,      'OFFICE',         'yes'),
    ('Mystery Widget',     '$0.00',   NULL,             NULL);
```

**Constraints:**
- **Zero data loss.** Every row must survive all migrations intact.
- **Each migration must have a working `downgrade()`.** Round-trip must work.
- **Handle all edge cases.** NULLs, empty strings, case mismatches, unexpected formats.
- **Use Alembic properly.** No `metadata.create_all`, no raw scripts outside of migrations.
- **All provided tests must pass.**

---

### Task 3a: Adopt the Legacy Database

**Problem:** The products table already exists in production, but Alembic has never been used. If you create a migration that does `op.create_table("products", ...)` and run it, PostgreSQL will say "table already exists."

Write a migration that:
1. Describes the **existing** schema (so a fresh database can be built from scratch with `upgrade head`)
2. Can be "stamped" on the production database (using `alembic stamp`) so Alembic recognizes the current state
3. Has a working `downgrade()` for fresh databases

**Hint:** The migration's `upgrade()` should create the table normally. On production, you will use `alembic stamp 001` instead of `alembic upgrade head` — this writes the revision to `alembic_version` without executing the `upgrade()` function.

**Tests for 3a:**

```python
def test_fresh_database_creates_table(fresh_engine):
    """On a brand-new database, upgrade creates the products table."""
    # Apply migration 001 via alembic upgrade
    inspector = inspect(fresh_engine)
    assert "products" in inspector.get_table_names()
    columns = {c["name"] for c in inspector.get_columns("products")}
    assert columns >= {"id", "name", "price", "category", "in_stock"}


def test_stamp_on_existing_database(legacy_engine):
    """On the legacy database, stamp records the version without changing anything."""
    with legacy_engine.connect() as conn:
        # Verify data exists BEFORE stamp
        result = conn.execute(text("SELECT COUNT(*) FROM products"))
        assert result.scalar() == 5

    # Run: alembic stamp 001 (simulated)

    with legacy_engine.connect() as conn:
        # Verify data still exists AFTER stamp
        result = conn.execute(text("SELECT COUNT(*) FROM products"))
        assert result.scalar() == 5

        # Verify alembic_version is set
        result = conn.execute(text("SELECT version_num FROM alembic_version"))
        assert result.scalar() is not None
```

---

### Task 3b: Convert Price from String to Numeric

**Problem:** The `price` column stores strings like `"$29.99"`. You need it as `NUMERIC(10, 2)` for calculations.

Write a migration that:
1. Adds a `price_numeric` column (NUMERIC(10, 2), nullable)
2. Converts all existing prices: strip `$`, handle empty strings, handle NULLs
3. Drops the old `price` column and renames `price_numeric` to `price`
4. Has a working `downgrade()` that converts back to the `"$xx.xx"` string format

**Edge cases you must handle:**
- `'$29.99'` → `29.99`
- `'$5.50'` → `5.50`
- `''` (empty string) → `NULL`
- `NULL` → `NULL`
- `'$0.00'` → `0.00` (valid price, not NULL!)

**Tests for 3b:**

```python
def test_price_conversion_accuracy(migrated_engine):
    """All prices must be converted correctly."""
    with migrated_engine.connect() as conn:
        result = conn.execute(text(
            "SELECT name, price FROM products ORDER BY name"
        ))
        prices = {row[0]: row[1] for row in result}

        assert prices["Wireless Mouse"] == pytest.approx(29.99)
        assert prices["USB Cable"] == pytest.approx(5.50)
        assert prices["Desk Lamp"] is None        # empty string → NULL
        assert prices["Standing Desk"] is None     # NULL → NULL
        assert prices["Mystery Widget"] == pytest.approx(0.00)  # $0.00 is valid!


def test_price_is_numeric_type(migrated_engine):
    """Price column must be NUMERIC type."""
    inspector = inspect(migrated_engine)
    columns = {c["name"]: c for c in inspector.get_columns("products")}
    assert "NUMERIC" in str(columns["price"]["type"]).upper()


def test_price_round_trip(migrated_engine):
    """Upgrade then downgrade must preserve price data (as strings)."""
    # After downgrade, prices should be back in "$xx.xx" format
    with migrated_engine.connect() as conn:
        # (simulate downgrade)
        result = conn.execute(text(
            "SELECT name, price FROM products WHERE name = 'Wireless Mouse'"
        ))
        row = result.fetchone()
        assert row[1] == "$29.99" or row[1] == "29.99"  # Accept either format
```

---

### Task 3c: Normalize Categories

**Problem:** The `category` column contains freeform text with inconsistent casing (`'Electronics'`, `'electronics'`, `'OFFICE'`). You need to normalize this into a proper `categories` table with a foreign key.

Write a migration that:
1. Creates a `categories` table (id, name — unique)
2. Extracts unique categories from the existing data (case-insensitive deduplication)
3. Adds `category_id` (FK → categories.id) to products
4. Populates `category_id` for all existing products
5. Drops the old `category` text column
6. Has a working `downgrade()`

**Edge cases:**
- `'Electronics'` and `'electronics'` must become the SAME category
- `NULL` category → `category_id` should be `NULL`
- The canonical form should be title case: `'electronics'` → `'Electronics'`

**Tests for 3c:**

```python
def test_categories_normalized(migrated_engine):
    """Duplicate categories (case-insensitive) must be deduplicated."""
    with migrated_engine.connect() as conn:
        result = conn.execute(text("SELECT COUNT(*) FROM categories"))
        # 'Electronics', 'electronics' → 1 category
        # 'Home & Garden' → 1 category
        # 'OFFICE' → 1 category
        # NULL → no category row created
        count = result.scalar()
        assert count == 3, f"Expected 3 unique categories, got {count}"


def test_products_linked_to_categories(migrated_engine):
    """All products with a category must be linked via FK."""
    with migrated_engine.connect() as conn:
        result = conn.execute(text("""
            SELECT p.name, c.name as category_name
            FROM products p
            LEFT JOIN categories c ON p.category_id = c.id
            ORDER BY p.name
        """))
        rows = {row[0]: row[1] for row in result}

        assert rows["Wireless Mouse"] == "Electronics"
        assert rows["USB Cable"] == "Electronics"  # was 'electronics' → normalized
        assert rows["Desk Lamp"] == "Home & Garden"
        assert rows["Standing Desk"] == "Office"    # was 'OFFICE' → title case
        assert rows["Mystery Widget"] is None       # NULL category → NULL FK


def test_null_category_preserved(migrated_engine):
    """Products with NULL category must have NULL category_id."""
    with migrated_engine.connect() as conn:
        result = conn.execute(text("""
            SELECT category_id FROM products WHERE name = 'Mystery Widget'
        """))
        assert result.scalar() is None
```

---

# Level 3 Solutions

## Solution 3a: Adopt the Legacy Database

```python
"""initial schema — adopt legacy products table

Revision ID: leg00001
Revises: None
"""
revision: str = "leg00001"
down_revision = None


def upgrade() -> None:
    # This migration describes the EXISTING legacy schema.
    # On a fresh database: creates the table normally.
    # On the legacy database: use `alembic stamp leg00001` instead of upgrade.
    op.create_table(
        "products",
        sa.Column("id", sa.Integer(), autoincrement=True, primary_key=True),
        sa.Column("name", sa.String(200), nullable=False),
        sa.Column("price", sa.String(50), nullable=True),
        sa.Column("category", sa.Text(), nullable=True),
        sa.Column("in_stock", sa.String(5), nullable=True),
    )


def downgrade() -> None:
    op.drop_table("products")
```

**Deployment workflow:**

```bash
# On the LEGACY PRODUCTION database:
$ alembic stamp leg00001
# Writes "leg00001" to alembic_version WITHOUT running upgrade().
# Database is untouched. Alembic now knows "we're at leg00001."

# On a FRESH development database:
$ alembic upgrade head
# Creates the products table from scratch. Works normally.
```

**Why this works:** `alembic stamp` is the bridge between "database created without Alembic" and "database managed by Alembic." It writes the version pointer without executing any operations. From this point forward, all new migrations work on both legacy and fresh databases.

---

## Solution 3b: Convert Price to Numeric

```python
"""convert price from string to numeric

Revision ID: leg00002
Revises: leg00001
"""
revision: str = "leg00002"
down_revision: str = "leg00001"


def upgrade() -> None:
    # Step 1: Add new numeric column
    op.add_column(
        "products",
        sa.Column("price_numeric", sa.Numeric(10, 2), nullable=True),
    )

    # Step 2: Convert data with full edge-case handling
    op.execute("""
        UPDATE products SET price_numeric = CASE
            WHEN price IS NULL THEN NULL
            WHEN TRIM(price) = '' THEN NULL
            ELSE CAST(REPLACE(TRIM(price), '$', '') AS NUMERIC(10, 2))
        END
    """)

    # Step 3: Verification — detect conversion failures
    op.execute("""
        DO $$
        DECLARE
            unexpected_nulls INTEGER;
        BEGIN
            SELECT COUNT(*) INTO unexpected_nulls
            FROM products
            WHERE price IS NOT NULL
              AND TRIM(price) != ''
              AND price_numeric IS NULL;

            IF unexpected_nulls > 0 THEN
                RAISE EXCEPTION
                    '% rows had non-empty price but failed numeric conversion',
                    unexpected_nulls;
            END IF;
        END $$;
    """)

    # Step 4: Swap columns
    op.drop_column("products", "price")
    op.alter_column("products", "price_numeric", new_column_name="price")


def downgrade() -> None:
    # Reverse: numeric back to "$xx.xx" string format
    op.alter_column("products", "price", new_column_name="price_numeric")
    op.add_column(
        "products",
        sa.Column("price", sa.String(50), nullable=True),
    )
    op.execute("""
        UPDATE products SET price = CASE
            WHEN price_numeric IS NULL THEN NULL
            ELSE '$' || TO_CHAR(price_numeric, 'FM999999990.00')
        END
    """)
    op.drop_column("products", "price_numeric")
```

**Key details:**
- `TRIM(price)` handles whitespace around values
- `REPLACE(..., '$', '')` strips the dollar sign
- Empty strings and NULLs are both mapped to `NULL` explicitly
- `$0.00` correctly converts to `0.00` (the CASE's ELSE clause handles it — it's not NULL and not empty)
- Verification step catches any rows where conversion silently failed
- Downgrade uses `TO_CHAR(..., 'FM999999990.00')` to format numeric back to string (`FM` prefix strips leading spaces)

---

## Solution 3c: Normalize Categories

```python
"""normalize categories into separate table

Revision ID: leg00003
Revises: leg00002
"""
revision: str = "leg00003"
down_revision: str = "leg00002"


def upgrade() -> None:
    # Step 1: Create categories table
    op.create_table(
        "categories",
        sa.Column("id", sa.Integer(), autoincrement=True, primary_key=True),
        sa.Column("name", sa.String(100), nullable=False, unique=True),
    )

    # Step 2: Extract unique categories (case-insensitive, title-cased)
    op.execute("""
        INSERT INTO categories (name)
        SELECT DISTINCT INITCAP(LOWER(TRIM(category)))
        FROM products
        WHERE category IS NOT NULL AND TRIM(category) != ''
    """)

    # Step 3: Add FK column to products
    op.add_column(
        "products",
        sa.Column("category_id", sa.Integer(), nullable=True),
    )

    # Step 4: Link products to normalized categories
    op.execute("""
        UPDATE products p
        SET category_id = c.id
        FROM categories c
        WHERE LOWER(TRIM(p.category)) = LOWER(c.name)
    """)

    # Step 5: Verification
    op.execute("""
        DO $$
        DECLARE
            orphaned INTEGER;
        BEGIN
            SELECT COUNT(*) INTO orphaned
            FROM products
            WHERE category IS NOT NULL
              AND TRIM(category) != ''
              AND category_id IS NULL;

            IF orphaned > 0 THEN
                RAISE EXCEPTION
                    '% products have a category string but no category_id link',
                    orphaned;
            END IF;
        END $$;
    """)

    # Step 6: Add FK constraint
    op.create_foreign_key(
        "fk_products_category_id", "products", "categories",
        ["category_id"], ["id"],
    )

    # Step 7: Drop old column (data is now in categories table)
    op.drop_column("products", "category")


def downgrade() -> None:
    # Reverse: bring back the freeform category column
    op.add_column(
        "products",
        sa.Column("category", sa.Text(), nullable=True),
    )

    # Populate from normalized table
    op.execute("""
        UPDATE products p
        SET category = c.name
        FROM categories c
        WHERE p.category_id = c.id
    """)

    # Drop FK and column
    op.drop_constraint("fk_products_category_id", "products", type_="foreignkey")
    op.drop_column("products", "category_id")

    # Drop categories table
    op.drop_table("categories")
```

**Key details:**
- `INITCAP(LOWER(TRIM(category)))` normalizes to title case: `'OFFICE'` → `'Office'`, `'electronics'` → `'Electronics'`
- The `INSERT INTO categories ... SELECT DISTINCT` extracts unique normalized names in one statement
- The `UPDATE ... FROM` join uses `LOWER()` on both sides for case-insensitive matching
- Products with `NULL` category correctly get `NULL` `category_id` (the `WHERE` clause in the UPDATE won't match them)
- The FK constraint is added AFTER data is populated (to avoid constraint violations during the UPDATE)
- Downgrade restores the original freeform text from the normalized categories table

**Why this can't be done with autogenerate:** Autogenerate can detect "create categories table" and "add category_id column" and "drop category column." But it will NEVER generate the data extraction, linking, or verification steps. The entire data migration must be written by hand. This is why the lecture says: "even teams that use autogenerate still need developers who understand manual migrations."

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## When Autogenerate Is the Wrong Tool

**Task:** Describe a realistic scenario where running `alembic revision --autogenerate` produces a migration that, if applied without review, causes catastrophic data loss or corruption in production. Your answer must include:

1. The model change you made (before and after)
2. The migration that autogenerate produces
3. What happens to production data if you blindly apply it
4. The correct manual migration that should have been written
5. A general rule for when to NEVER trust autogenerate

---

# Level 4 Solution

**Scenario: Splitting a Column Into Two**

You have a `users` table with a `full_name` column (`"Alice Johnson"`). The product team wants separate `first_name` and `last_name` columns.

**1. Model Change:**
```python
# BEFORE
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    full_name: Mapped[str] = mapped_column(String(200))
    email: Mapped[str] = mapped_column(String(200))

# AFTER
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    first_name: Mapped[str] = mapped_column(String(100))
    last_name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(200))
```

**2. What Autogenerate Produces:**
```python
def upgrade() -> None:
    op.add_column("users", sa.Column("first_name", sa.String(100), nullable=False))
    op.add_column("users", sa.Column("last_name", sa.String(100), nullable=False))
    op.drop_column("users", "full_name")
```

**3. Catastrophe:** If blindly applied to a production database with 50,000 users:
- `add_column ... NOT NULL` immediately fails (no `server_default`, existing rows). If the developer "fixes" this by making them nullable:
- `drop_column("full_name")` destroys all 50,000 names permanently
- `first_name` and `last_name` are both `NULL` — no way to recover the original names

**4. The Correct Manual Migration:**
```python
def upgrade() -> None:
    # Step 1: Add new nullable columns
    op.add_column("users", sa.Column("first_name", sa.String(100), nullable=True))
    op.add_column("users", sa.Column("last_name", sa.String(100), nullable=True))

    # Step 2: Split existing data
    op.execute("""
        UPDATE users SET
            first_name = SPLIT_PART(full_name, ' ', 1),
            last_name = CASE
                WHEN POSITION(' ' IN full_name) > 0
                THEN SUBSTRING(full_name FROM POSITION(' ' IN full_name) + 1)
                ELSE ''
            END
    """)

    # Step 3: Now enforce NOT NULL
    op.alter_column("users", "first_name", nullable=False)
    op.alter_column("users", "last_name", nullable=False)

    # Step 4: Drop old column (data has been migrated)
    op.drop_column("users", "full_name")
```

**5. The General Rule:**

**Never trust autogenerate when the semantic intent involves data transformation.** Autogenerate sees structural changes (column added, column removed) but cannot understand that data needs to *move* from one column to another, be *split*, be *merged*, be *converted*, or be *normalized*. Any time your model change is more than "add something new" or "remove something unused," you must write the migration by hand.

Specific scenarios where autogenerate should be immediately distrusted:
- Column renames → will generate DROP + ADD (data loss)
- Column splits or merges → will generate DROP + ADD (data loss)
- Type changes that require data transformation → will generate type change without data conversion
- Table renames → will generate DROP TABLE + CREATE TABLE (data loss)
- Structural refactoring (normalization, denormalization) → will never generate the data migration steps

---

# LEVEL 5: THE TEACH BACK

## The Slack Message

A junior developer on your team sends you this message:

> *"Hey, I found a bug in migration `005_add_description.py`. The column was created as `String(50)` but it should be `String(500)`. I opened the file, changed `50` to `500`, and ran `alembic upgrade head`. But when I check the database, the column is still `VARCHAR(50)`. I ran it three times. Is Alembic broken? Also, I pushed the fix to the repo because I figured the corrected file should be in Git."*

**Task:** Write a detailed response that covers:

1. Why their approach didn't work
2. What Alembic actually does when it processes `upgrade head`
3. The immediate damage their pushed "fix" will cause for the team
4. The correct way to fix this problem
5. An analogy to something they already understand (Git) that makes the behavior intuitive

---

# Level 5 Solution

**Your Response:**

Alembic isn't broken — but you've created a problem we need to fix quickly. Let me explain what happened and how to correct it.

**1. Why Nothing Changed**

When you run `alembic upgrade head`, Alembic does NOT re-read or re-execute migration files that have already been applied. The process is:

```
Step 1: Read alembic_version table → "005 is applied"
Step 2: Read migration files → "005 is the head"
Step 3: Compare → "005 == 005, nothing to do"
Step 4: Exit with no changes
```

Alembic only checks if the version ID **matches**, not whether the file's contents have changed. Once a migration is recorded in `alembic_version`, that migration is "done" forever. Editing the file is invisible to Alembic on any database that has already run it.

**2. The Git Analogy**

Think about Git commits. If you amend a commit that's already been pushed and pulled by teammates, you create a divergence — their repo has the old version, yours has the new version, and Git doesn't automatically fix that. Same principle here. Migration `005` is like a commit that has already been "applied" to your database. Rewriting the file is like rewriting a pushed commit — the change only affects people who haven't applied it yet, and it creates silent inconsistencies for everyone who has.

**3. The Damage From Pushing**

By pushing the edited file, you've created a schema divergence trap:

- **Your database:** `VARCHAR(50)` — the edited file was already applied before your edit, so the edit had no effect
- **Staging / production:** `VARCHAR(50)` — same reason; they ran the original `005` weeks ago
- **A new teammate who joins tomorrow:** `VARCHAR(500)` — they run `upgrade head` from scratch and get the edited version

Now you have databases at the same `alembic_version` revision but with different column sizes. This is a silent bug that can cause data truncation on some environments and not others.

**4. The Correct Fix**

First, revert the file edit (restore the original `005`). Then create a NEW migration:

```bash
$ alembic revision -m "fix description column length from 50 to 500"
```

```python
def upgrade() -> None:
    op.alter_column(
        "your_table",
        "description",
        type_=sa.String(500),
        existing_type=sa.String(50),
    )

def downgrade() -> None:
    op.alter_column(
        "your_table",
        "description",
        type_=sa.String(50),
        existing_type=sa.String(500),
    )
```

This new migration (`006`) applies the fix through Alembic's normal forward path. Every database — old, new, staging, production — will apply `006` and end up with `VARCHAR(500)`. No divergence.

**5. The Rule to Remember**

Once a migration has been applied on ANY environment — your local database, a teammate's database, staging, production, anywhere — that migration file is **frozen**. Never edit it. If it has a bug, fix it by moving forward with a new migration. This is the same principle as "don't rewrite published Git history" — once something is shared, you correct by adding, not by rewriting.