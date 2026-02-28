# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BREAK IT FIRST, FIX IT SECOND                                  │
│  ─────────────────────────────                                  │
│  Students will watch real data vanish. They'll see              │
│  metadata.create_all silently do NOTHING when a column changes. │
│  Pain is the best teacher for why migrations exist.             │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  They already know Git from Week 1. Alembic IS Git for your    │
│  database. Every concept maps to something they've already      │
│  internalized. We exploit that foundation ruthlessly.           │
│                                                                 │
│  BUILD MENTAL MODEL BEFORE COMMANDS                             │
│  ──────────────────────────────────                             │
│  The migration chain diagram comes before `alembic revision`.   │
│  Understand what a migration IS before creating one.            │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  SQLAlchemy models (Lecture 1) → what Alembic reads             │
│  metadata.create_all (Lecture 1) → the problem Alembic solves   │
│  Git workflow (Week 1) → same mental model, different target    │
│  PostgreSQL (Week 5) → the actual database being migrated       │
│  Testing (Week 2) → migrations must be tested too               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                    ALEMBIC MIGRATIONS                           │
│                     (3.5 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 The Schema Change That Kills (Demonstration)            │
│  ├─ 1.2 Why metadata.create_all Is Not Enough                   │
│  ├─ 1.3 The Real World: Teams, Staging, Production              │
│  └─ 1.4 The Analogy: Git for Your Database                      │
│                                                                 │
│  PART 2: THE MENTAL MODEL (30 min)                              │
│  ├─ 2.1 What Is a Migration?                                    │
│  ├─ 2.2 The Migration Chain                                     │
│  ├─ 2.3 Upgrade vs Downgrade                                    │
│  └─ 2.4 How Alembic Tracks State                                │
│                                                                 │
│  PART 3: SETUP & FIRST MIGRATION (60 min)                       │
│  ├─ 3.1 Installing and Initializing Alembic                     │
│  ├─ 3.2 The File Structure (What Got Generated)                 │
│  ├─ 3.3 Configuration (alembic.ini and env.py)                  │
│  ├─ 3.4 Your First Migration (by Hand)                          │
│  ├─ 3.5 Running Migrations (upgrade / downgrade)                │
│  └─ 3.6 Inspecting State (current, history)                     │
│                                                                 │
│  PART 4: AUTOGENERATION (45 min)                                │
│  ├─ 4.1 How Autogenerate Works (The Diff Engine)                │
│  ├─ 4.2 Autogenerate in Practice                                │
│  ├─ 4.3 What Autogenerate CAN Detect                            │
│  ├─ 4.4 What Autogenerate CANNOT Detect                         │
│  └─ 4.5 The Golden Rule: Always Review                          │
│                                                                 │
│  PART 5: ADVANCED PATTERNS & BEST PRACTICES (45 min)            │
│  ├─ 5.1 Data Migrations (Not Just Schema Changes)               │
│  ├─ 5.2 Rollback Strategies                                     │
│  ├─ 5.3 Migrations in a Team (Conflicts and Merges)             │
│  ├─ 5.4 Best Practices Checklist                                │
│  ├─ 5.5 Migrations in CI (Preview for Week 15)                  │
│  └─ 5.6 Common Mistakes and How to Avoid Them                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Schema Change That Kills

**Start with the models they built in Lecture 1 and 2 this week.**

We have a working Task Manager with real data in PostgreSQL. Life is good.

```python
# models.py — what we built in Lecture 1
from datetime import datetime
from sqlalchemy import String, DateTime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    completed: Mapped[bool] = mapped_column(default=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.utcnow
    )
```

```python
# seed.py — let's pretend we have real users and real data
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
from models import Base, Task

engine = create_engine("postgresql://user:pass@localhost:5432/taskmanager")
Base.metadata.create_all(engine)  # Creates the table. Works fine first time.

with Session(engine) as session:
    session.add_all([
        Task(title="Deploy v1 to production"),
        Task(title="Write API documentation"),
        Task(title="Fix authentication bug", completed=True),
        Task(title="Set up monitoring dashboard"),
        Task(title="Review pull request #42"),
    ])
    session.commit()
    print("Seeded 5 tasks!")
```

**Run it. Verify the data is there:**

```sql
taskmanager=# SELECT * FROM tasks;
 id |             title              | completed |         created_at
----+--------------------------------+-----------+----------------------------
  1 | Deploy v1 to production        | f         | 2025-06-15 10:30:00.000000
  2 | Write API documentation        | f         | 2025-06-15 10:30:00.000000
  3 | Fix authentication bug         | t         | 2025-06-15 10:30:00.000000
  4 | Set up monitoring dashboard    | f         | 2025-06-15 10:30:00.000000
  5 | Review pull request #42        | f         | 2025-06-15 10:30:00.000000
(5 rows)
```

**Now: the product manager walks in.**

> "We need a priority field on tasks. High, medium, low. Can you add that by Thursday?"

Easy, right? Just add the column to the model:

```python
# models.py — updated
class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    completed: Mapped[bool] = mapped_column(default=False)
    priority: Mapped[str] = mapped_column(String(10), default="medium")  # NEW!
    created_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.utcnow
    )
```

**Run the app. Watch it fail:**

```python
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session
from models import Base, Task

engine = create_engine("postgresql://user:pass@localhost:5432/taskmanager")
Base.metadata.create_all(engine)  # This should handle it... right?

with Session(engine) as session:
    # Try to query tasks
    tasks = session.execute(select(Task)).scalars().all()
    for task in tasks:
        print(f"{task.title} — priority: {task.priority}")
```

```
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedColumn)
column tasks.priority does not exist
LINE 1: SELECT tasks.id, tasks.title, tasks.completed, tasks.priorit...
```

**Ask the class:**

> "We added `priority` to our Python model. We called `metadata.create_all()`. Why did it crash?"

---

## 1.2 Why metadata.create_all Is Not Enough

**The answer: `create_all` only CREATES. It never ALTERS.**

```
┌─────────────────────────────────────────────────────────────────┐
│              metadata.create_all() — THE TRUTH                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What it DOES:                                                  │
│  ├─ Checks if a table EXISTS                                    │
│  ├─ If NO  → CREATE TABLE (with all columns)  ✅               │
│  └─ If YES → Do absolutely nothing            😐               │
│                                                                 │
│  What it DOES NOT do:                                           │
│  ├─ Compare your model to the existing table   ❌               │
│  ├─ Add new columns                            ❌               │
│  ├─ Remove old columns                         ❌               │
│  ├─ Change column types                        ❌               │
│  ├─ Add or remove constraints                  ❌               │
│  └─ Anything related to modifying structure     ❌               │
│                                                                 │
│  It's a one-shot tool. Build once, then it's done.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**"Okay, I'll just DROP the table and recreate it."**

```python
# The desperate developer's solution:
Base.metadata.drop_all(engine)    # DROP TABLE tasks
Base.metadata.create_all(engine)  # CREATE TABLE tasks (with priority!)
```

**Run it. Check the database:**

```sql
taskmanager=# SELECT * FROM tasks;
 id | title | completed | priority | created_at
----+-------+-----------+----------+------------
(0 rows)
```

**All 5 tasks. Gone. Obliterated.**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│                     YOUR DATA                                   │
│                                                                 │
│    BEFORE drop_all:              AFTER drop_all:                │
│                                                                 │
│    ┌────────────────────┐        ┌────────────────────┐         │
│    │ Deploy v1          │        │                    │         │
│    │ Write API docs     │        │                    │         │
│    │ Fix auth bug       │───────▶│     (empty)        │         │
│    │ Set up monitoring  │  💀    │                    │         │
│    │ Review PR #42      │        │                    │         │
│    └────────────────────┘        └────────────────────┘         │
│                                                                 │
│    5 tasks                       0 tasks                        │
│                                                                 │
│    In development: annoying.                                    │
│    In production: career-ending.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The point:**

> "You have exactly two options with `metadata.create_all`: do nothing, or destroy everything. Neither is acceptable when real data exists. We need a third option: **change the table structure while keeping the data intact.**"

---

## 1.3 The Real World: Teams, Staging, Production

**The problem is bigger than one developer.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE REAL-WORLD PROBLEM                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your app doesn't run on one laptop. It runs on:                │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                   │
│  │  Dev DB  │    │ Staging  │    │Production│                   │
│  │ (yours)  │    │   DB     │    │   DB     │                   │
│  └──────────┘    └──────────┘    └──────────┘                   │
│   Your laptop     Test server     Real users                    │
│   Fake data       Test data       Real data                     │
│   Can destroy     Should match    CANNOT lose                   │
│                   production                                    │
│                                                                 │
│  AND your teammate has THEIR own dev database:                  │
│                                                                 │
│  ┌──────────┐                                                   │
│  │  Dev DB  │    ← They pulled your code.                       │
│  │ (theirs) │      Their database is still the OLD schema.      │
│  └──────────┘      How do they get the new column?              │
│                                                                 │
│                                                                 │
│  Questions that MUST have answers:                              │
│                                                                 │
│  1. How does your teammate get the `priority` column?           │
│  2. How does staging get it?                                    │
│  3. How does production get it WITHOUT losing data?             │
│  4. If the change breaks something, how do you UNDO it?         │
│  5. How do you know WHICH changes have been applied WHERE?      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Write the SQL yourself? Dangerous.**

```sql
-- You could write this manually...
ALTER TABLE tasks ADD COLUMN priority VARCHAR(10) DEFAULT 'medium';

-- But:
-- Who tracks which ALTER statements have been run?
-- What if you run it twice? (ERROR: column already exists)
-- What if your teammate writes a DIFFERENT alter for the same table?
-- What if you need to undo it?
-- What about 50 changes over 6 months?
```

**This is a version control problem. And you already know how to solve version control problems.**

---

## 1.4 The Analogy: Git for Your Database

**You learned Git in Week 1. Alembic is the same idea for database schemas.**

```
┌─────────────────────────────────────────────────────────────────┐
│                GIT vs ALEMBIC — SAME BRAIN                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│       Git (Code)                 Alembic (Database)             │
│       ──────────                 ─────────────────              │
│                                                                 │
│  git init                    alembic init                       │
│  └─ Start tracking           └─ Start tracking                  │
│     code changes                schema changes                  │
│                                                                 │
│  git commit -m "msg"         alembic revision -m "msg"          │
│  └─ Record a code change     └─ Record a schema change          │
│                                                                 │
│  git log                     alembic history                    │
│  └─ See all past commits     └─ See all past migrations         │
│                                                                 │
│  git status                  alembic current                    │
│  └─ Where am I now?          └─ Which migration is applied?     │
│                                                                 │
│  git checkout <commit>       alembic upgrade <revision>         │
│  └─ Move to a specific       └─ Move to a specific              │
│     point in history            schema version                  │
│                                                                 │
│  git revert                  alembic downgrade                  │
│  └─ Undo a change            └─ Undo a migration                │
│                                                                 │
│  .git/ directory             alembic_version table              │
│  └─ Stores history locally   └─ Stores current version          │
│                                                                 │
│  Merge conflicts             Migration conflicts                │
│  └─ Two people changed       └─ Two people created              │
│     the same file               migrations at once              │
│                                                                 │
│  git diff (auto-detect)      alembic --autogenerate             │
│  └─ Show what changed        └─ Detect model changes            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "If Git had existed in the 1980s but only for code, and you asked 'how do I track database changes?' — someone would have invented Alembic. It's the same problem, the same solution, applied to a different target."

---

# PART 2: THE MENTAL MODEL

## 2.1 What Is a Migration?

**A migration is a single, atomic change to your database schema.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHAT IS A MIGRATION?                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A migration is a Python file that contains TWO functions:      │
│                                                                 │
│  ┌────────────────────────────────────────────────┐             │
│  │                                                │             │
│  │   def upgrade():                               │             │
│  │       """How to APPLY this change"""            │             │
│  │       # Add the priority column                │             │
│  │                                                │             │
│  │   def downgrade():                             │             │
│  │       """How to UNDO this change"""             │             │
│  │       # Remove the priority column             │             │
│  │                                                │             │
│  └────────────────────────────────────────────────┘             │
│                                                                 │
│  Every migration is REVERSIBLE.                                 │
│  upgrade() moves FORWARD.                                       │
│  downgrade() moves BACKWARD.                                    │
│                                                                 │
│  Think of it like a Git commit with a built-in revert.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Each migration describes ONE change, not the whole schema:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ NOT a migration:                                            │
│     "Here's the entire database, all tables, all columns"       │
│     (That's metadata.create_all)                                │
│                                                                 │
│  ✅ A migration:                                                │
│     "Add a 'priority' column to the 'tasks' table"              │
│                                                                 │
│  ✅ Another migration:                                          │
│     "Create a 'categories' table with id and name"              │
│                                                                 │
│  ✅ Another migration:                                          │
│     "Add foreign key 'category_id' to 'tasks' table"            │
│                                                                 │
│  Each one is SMALL, SPECIFIC, and REVERSIBLE.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 The Migration Chain

**Migrations form a linked chain. Each one knows its predecessor.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE MIGRATION CHAIN                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Each migration has:                                            │
│  ├─ A unique ID (revision hash, like a short git commit hash)   │
│  ├─ A pointer to its PARENT (down_revision)                     │
│  └─ A human-readable message                                    │
│                                                                 │
│                                                                 │
│  (empty DB)                                                     │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────┐                            │
│  │  a1b2c3: "create tasks table"   │                            │
│  │  down_revision = None           │  ← First migration         │
│  └──────────────┬──────────────────┘                            │
│                 │                                                │
│                 ▼                                                │
│  ┌─────────────────────────────────┐                            │
│  │  d4e5f6: "add priority column"  │                            │
│  │  down_revision = a1b2c3         │  ← Points to parent        │
│  └──────────────┬──────────────────┘                            │
│                 │                                                │
│                 ▼                                                │
│  ┌─────────────────────────────────┐                            │
│  │  g7h8i9: "create categories"    │                            │
│  │  down_revision = d4e5f6         │  ← Points to parent        │
│  └──────────────┬──────────────────┘                            │
│                 │                                                │
│                 ▼                                                │
│             (head)                                               │
│                                                                 │
│  Just like git commits — a linear history of changes.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Compare directly to what you already know:**

```
Git log:                          Alembic history:

commit c3f2a1                     g7h8i9 (head)
  "Fix styling"                     "create categories"
       │                                 │
commit b2e1d0                     d4e5f6
  "Add login page"                  "add priority column"
       │                                 │
commit a1c0b9                     a1b2c3
  "Initial commit"                  "create tasks table"
       │                                 │
    (empty)                           (empty DB)
```

---

## 2.3 Upgrade vs Downgrade

**Moving forward and backward through the chain:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 UPGRADE AND DOWNGRADE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      upgrade ──────────▶                        │
│                                                                 │
│  (empty)    a1b2c3         d4e5f6         g7h8i9                │
│    DB   ──▶ create   ──▶  add       ──▶  create                │
│             tasks         priority        categories            │
│                                                                 │
│                      ◀────────── downgrade                      │
│                                                                 │
│                                                                 │
│  UPGRADE runs the upgrade() function:                           │
│  ├─ Applies the change to the database                          │
│  └─ Moves forward in history                                    │
│                                                                 │
│  DOWNGRADE runs the downgrade() function:                       │
│  ├─ Reverses the change                                         │
│  └─ Moves backward in history                                   │
│                                                                 │
│                                                                 │
│  EXAMPLES:                                                      │
│                                                                 │
│  alembic upgrade head       → Apply ALL pending migrations      │
│  alembic upgrade +1         → Apply next ONE migration          │
│  alembic upgrade d4e5f6     → Upgrade TO that specific revision │
│                                                                 │
│  alembic downgrade -1       → Undo the LAST migration           │
│  alembic downgrade a1b2c3   → Downgrade TO that revision        │
│  alembic downgrade base     → Undo ALL migrations (empty DB)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Walk through a concrete scenario:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   SCENARIO: ROLLBACK                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Current state: all 3 migrations applied                        │
│                                                                 │
│  DB has: tasks table (with priority), categories table          │
│          ▲ You are here (g7h8i9)                                │
│                                                                 │
│  Problem: categories table has a bug. Need to undo it.          │
│                                                                 │
│  $ alembic downgrade -1                                         │
│                                                                 │
│  What happens:                                                  │
│  1. Alembic looks at g7h8i9's downgrade() function              │
│  2. Runs: DROP TABLE categories                                 │
│  3. Updates alembic_version to d4e5f6                           │
│                                                                 │
│  DB now has: tasks table (with priority). No categories.        │
│              ▲ You are here (d4e5f6)                            │
│                                                                 │
│  Data in tasks table: UNTOUCHED. Safe. ✅                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 How Alembic Tracks State

**Alembic stores exactly ONE thing in your database: the current revision.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE alembic_version TABLE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alembic creates a tiny table in your database:                 │
│                                                                 │
│  ┌──────────────────────┐                                       │
│  │   alembic_version    │                                       │
│  ├──────────────────────┤                                       │
│  │ version_num: VARCHAR │                                       │
│  ├──────────────────────┤                                       │
│  │ "d4e5f6"             │  ← Current migration applied          │
│  └──────────────────────┘                                       │
│                                                                 │
│  That's it. One table. One row. One column.                     │
│                                                                 │
│                                                                 │
│  HOW ALEMBIC KNOWS WHAT TO DO:                                  │
│                                                                 │
│  1. Read version_num from database      → "d4e5f6"              │
│  2. Read migration files from disk      → chain of revisions    │
│  3. Compare: where am I vs where to go?                         │
│                                                                 │
│     Database says:  d4e5f6  (2nd migration)                     │
│     You said:       "upgrade head"                              │
│     Head is:        g7h8i9  (3rd migration)                     │
│                                                                 │
│  4. Run upgrade() for every migration between d4e5f6 → g7h8i9  │
│     In this case: just g7h8i9's upgrade()                       │
│                                                                 │
│                                                                 │
│  Like Git's HEAD pointer — it tells you where you are           │
│  in the history, and Alembic figures out what to run.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**This is how your teammate gets your changes:**

```
┌─────────────────────────────────────────────────────────────────┐
│               TEAM WORKFLOW WITH ALEMBIC                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOU:                                                           │
│  1. Change SQLAlchemy model (add priority column)               │
│  2. Generate migration file                                     │
│  3. Run migration on YOUR database                              │
│  4. Commit migration file to Git (it's just a .py file!)        │
│  5. Push to remote                                              │
│                                                                 │
│  YOUR TEAMMATE:                                                 │
│  1. git pull (gets your migration file)                         │
│  2. alembic upgrade head                                        │
│  3. Done. Their database now matches yours. ✅                  │
│                                                                 │
│  STAGING/PRODUCTION:                                            │
│  1. Deploy new code (includes migration files)                  │
│  2. Run: alembic upgrade head                                   │
│  3. Database updated. No data lost. ✅                          │
│                                                                 │
│  Everyone runs the SAME migration files.                        │
│  Everyone's database ends up in the SAME state.                 │
│  The migration files ARE the single source of truth.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: SETUP & FIRST MIGRATION

## 3.1 Installing and Initializing Alembic

**Installation (you already have SQLAlchemy):**

```bash
# In your virtual environment (Week 1, Lecture 4 — you remember)
pip install alembic
```

**Initialization:**

```bash
# Navigate to your project root
cd task-manager/

# Initialize Alembic
alembic init alembic
```

That's it. Two words: `alembic init`, and a folder name.

---

## 3.2 The File Structure (What Got Generated)

**After `alembic init alembic`, your project looks like this:**

```
task-manager/
├── alembic/                      ← NEW: Alembic directory
│   ├── env.py                    ← The brain: how Alembic connects to DB
│   ├── README                    ← Just a note, you can ignore
│   ├── script.py.mako            ← Template for new migration files
│   └── versions/                 ← Where migration files live
│       └── (empty for now)
├── alembic.ini                   ← NEW: Alembic configuration file
├── src/
│   ├── models.py                 ← Your SQLAlchemy models
│   └── ...
└── pyproject.toml
```

**Map this to what you already know:**

```
┌─────────────────────────────────────────────────────────────────┐
│                GIT STRUCTURE vs ALEMBIC STRUCTURE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Git                              Alembic                       │
│  ───                              ───────                       │
│  .git/                            alembic/                      │
│  .git/config                      alembic.ini                   │
│  .git/objects/ (commits)          alembic/versions/ (migrations)│
│  .gitignore                       (nothing to ignore — commit   │
│                                    ALL migration files!)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Every migration file in `versions/` MUST be committed to Git. They are as important as your source code. If someone doesn't have a migration file, they can't apply that change."

---

## 3.3 Configuration (alembic.ini and env.py)

**There are two files to configure. Let's take them one at a time.**

### alembic.ini — The Connection String

This file lives at your project root. The critical line:

```ini
# alembic.ini
# Find this line (around line 63):
sqlalchemy.url = driver://user:pass@localhost/dbname

# Change it to your actual PostgreSQL URL:
sqlalchemy.url = postgresql://user:pass@localhost:5432/taskmanager
```

```
┌─────────────────────────────────────────────────────────────────┐
│                      ⚠️  WARNING                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NEVER commit real database credentials to Git.                 │
│                                                                 │
│  For now (development), hardcoding in alembic.ini is FINE.      │
│  In Week 15 (pydantic-settings), you'll learn to use            │
│  environment variables instead.                                 │
│                                                                 │
│  Quick preview of the proper way:                               │
│                                                                 │
│  # alembic.ini                                                  │
│  sqlalchemy.url =       ← Leave it BLANK                        │
│                                                                 │
│  # env.py                                                       │
│  config.set_main_option(                                        │
│      "sqlalchemy.url",                                          │
│      os.environ["DATABASE_URL"]  ← Read from environment        │
│  )                                                              │
│                                                                 │
│  For this week: hardcode it. Learn it properly in Week 15.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### env.py — Connecting Alembic to Your Models

This is the important file. Alembic needs to know about your SQLAlchemy models so it can compare them to the database. Open `alembic/env.py`:

```python
# alembic/env.py — the KEY change

# Find this line (near the top):
# target_metadata = None

# Replace it with:
from src.models import Base  # Import YOUR Base class
target_metadata = Base.metadata
```

**Why this matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY target_metadata MATTERS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  When you run autogenerate (Part 4), Alembic does this:         │
│                                                                 │
│  1. Reads target_metadata → "What SHOULD the DB look like?"     │
│     (Your SQLAlchemy models define the desired state)           │
│                                                                 │
│  2. Inspects actual database → "What DOES the DB look like?"    │
│     (Connects and reads real table structures)                  │
│                                                                 │
│  3. Computes the DIFF → "What changed?"                         │
│     (New columns? Removed tables? Type changes?)                │
│                                                                 │
│  4. Generates a migration file with the changes                 │
│                                                                 │
│  If target_metadata = None → Alembic can't compare.             │
│  Autogenerate produces empty migrations. Useless.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The full env.py you need (simplified, focused on what matters):**

```python
# alembic/env.py — cleaned up version
from logging.config import fileConfig

from sqlalchemy import engine_from_config, pool
from alembic import context

# Alembic Config object (reads alembic.ini)
config = context.config

# Set up logging
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# THIS IS THE CRITICAL LINE — import your models' metadata
from src.models import Base
target_metadata = Base.metadata


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode — generates SQL without connecting."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    """Run migrations in 'online' mode — connects to database."""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
        )
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

**Don't panic about the boilerplate:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "That's a lot of code I didn't write."                         │
│                                                                 │
│  Alembic generated 95% of env.py for you.                       │
│  You changed exactly ONE thing:                                 │
│                                                                 │
│    from src.models import Base                                   │
│    target_metadata = Base.metadata                               │
│                                                                 │
│  The rest is boilerplate that handles:                           │
│  ├─ Online mode (connected to real DB — you'll use this)        │
│  ├─ Offline mode (generates SQL text — rarely used)             │
│  └─ Logging configuration                                       │
│                                                                 │
│  You can understand it later. For now: import your Base,        │
│  set target_metadata, move on.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Your First Migration (by Hand)

**Let's create a migration manually first, before we learn autogenerate.**

Why manually? Same reason you learned raw SQL in Week 5 before using SQLAlchemy — **understand what the tool generates before trusting it to generate for you.**

**Create the initial migration that builds the tasks table:**

```bash
alembic revision -m "create tasks table"
```

Output:
```
Generating alembic/versions/a1b2c3d4e5f6_create_tasks_table.py ... done
```

**Look at the file name:**

```
a1b2c3d4e5f6_create_tasks_table.py
│              │
│              └─ Your message (slugified)
└─ Revision ID (random hash, like a git commit hash)
```

**Open the generated file:**

```python
# alembic/versions/a1b2c3d4e5f6_create_tasks_table.py

"""create tasks table

Revision ID: a1b2c3d4e5f6
Revises:
Create Date: 2025-06-15 14:30:00.000000
"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = "a1b2c3d4e5f6"
down_revision: Union[str, None] = None  # ← First migration, no parent
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    pass  # ← EMPTY. You fill this in.


def downgrade() -> None:
    pass  # ← EMPTY. You fill this in.
```

**The file has two empty functions. You write the SQL operations:**

```python
def upgrade() -> None:
    """Create the tasks table."""
    op.create_table(
        "tasks",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("title", sa.String(200), nullable=False),
        sa.Column("completed", sa.Boolean(), server_default="false"),
        sa.Column("created_at", sa.DateTime(), server_default=sa.func.now()),
    )


def downgrade() -> None:
    """Remove the tasks table."""
    op.drop_table("tasks")
```

**Understand the two key tools:**

```
┌─────────────────────────────────────────────────────────────────┐
│               op AND sa — YOUR TWO TOOLS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  op (alembic.op) — THE OPERATIONS                               │
│  ─────────────────────────────────                              │
│  Actions you perform on the database structure:                  │
│                                                                 │
│  op.create_table("name", columns...)   Create a new table       │
│  op.drop_table("name")                 Delete a table           │
│  op.add_column("table", column)        Add a column             │
│  op.drop_column("table", "col_name")   Remove a column          │
│  op.alter_column("table", "col", ...)  Change a column          │
│  op.create_index("name", "table", ..)  Add an index             │
│  op.drop_index("name")                 Remove an index          │
│  op.create_foreign_key(...)            Add a foreign key        │
│  op.execute("RAW SQL")                 Run any SQL directly     │
│                                                                 │
│                                                                 │
│  sa (sqlalchemy) — THE COLUMN TYPES                             │
│  ──────────────────────────────────                             │
│  Defining what columns look like:                                │
│                                                                 │
│  sa.Column("name", sa.Integer())       Integer column           │
│  sa.Column("name", sa.String(100))     Varchar column           │
│  sa.Column("name", sa.Boolean())       Boolean column           │
│  sa.Column("name", sa.DateTime())      Timestamp column         │
│  sa.Column("name", sa.Text())          Unlimited text           │
│                                                                 │
│  These are the same types you used in SQLAlchemy models          │
│  in Lecture 1. Same library, same types.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "Look at upgrade and downgrade. Notice they are EXACT OPPOSITES. upgrade creates the table, downgrade drops it. This must ALWAYS be true. If upgrade adds a column, downgrade must remove it. If upgrade creates an index, downgrade must drop it. They are mirrors."

---

## 3.5 Running Migrations (upgrade / downgrade)

**Apply the migration:**

```bash
# Apply all pending migrations up to the latest ("head")
alembic upgrade head
```

Output:
```
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> a1b2c3d4e5f6, create tasks table
```

**What just happened:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHAT upgrade head DID                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Connected to PostgreSQL                                     │
│  2. Checked: does alembic_version table exist?                  │
│     └─ No → Created it                                          │
│  3. Read version_num from alembic_version                       │
│     └─ Empty → No migrations have been applied                  │
│  4. Found pending migration: a1b2c3d4e5f6                       │
│  5. Ran a1b2c3d4e5f6's upgrade() function                       │
│     └─ Executed: CREATE TABLE tasks (...)                        │
│  6. Updated alembic_version to "a1b2c3d4e5f6"                   │
│                                                                 │
│                                                                 │
│  Database now contains:                                         │
│  ├─ tasks table (with id, title, completed, created_at)         │
│  └─ alembic_version table (version_num = "a1b2c3d4e5f6")       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Verify in PostgreSQL:**

```sql
taskmanager=# \dt
              List of relations
 Schema |      Name       | Type  | Owner
--------+-----------------+-------+-------
 public | alembic_version | table | user
 public | tasks           | table | user
(2 rows)

taskmanager=# SELECT * FROM alembic_version;
 version_num
--------------
 a1b2c3d4e5f6
(1 row)

taskmanager=# \d tasks
                                     Table "public.tasks"
   Column   |          Type          | Nullable |              Default
------------+------------------------+----------+-----------------------------------
 id         | integer                | not null | nextval('tasks_id_seq'::regclass)
 title      | character varying(200) | not null |
 completed  | boolean                |          | false
 created_at | timestamp              |          | now()
```

**Now add some data, then practice downgrade:**

```sql
INSERT INTO tasks (title) VALUES ('Deploy v1'), ('Write docs'), ('Fix auth bug');
```

```bash
# Undo the last migration
alembic downgrade -1
```

Output:
```
INFO  [alembic.runtime.migration] Running downgrade a1b2c3d4e5f6 -> , create tasks table
```

**The tasks table is gone. So is the data in it.** That's what `drop_table` does.

```bash
# Apply it again
alembic upgrade head
```

**The table is back, but empty. The data we inserted is gone forever.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CRITICAL LESSON                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Downgrading is safe for STRUCTURE.                             │
│  Downgrading can DESTROY DATA.                                  │
│                                                                 │
│  drop_table → data gone                                         │
│  drop_column → column data gone                                 │
│                                                                 │
│  This is expected. But it means:                                │
│  ├─ Test downgrades on staging BEFORE production                │
│  ├─ Back up data before destructive downgrades                  │
│  └─ Sometimes downgrade is intentionally NOT implemented        │
│     (we'll cover this in Part 5)                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now let's do the change that started this whole lecture. Add the priority column:**

```bash
alembic revision -m "add priority column to tasks"
```

```python
# alembic/versions/d4e5f6g7h8i9_add_priority_column_to_tasks.py

"""add priority column to tasks

Revision ID: d4e5f6g7h8i9
Revises: a1b2c3d4e5f6
Create Date: 2025-06-15 15:00:00.000000
"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


revision: str = "d4e5f6g7h8i9"
down_revision: Union[str, None] = "a1b2c3d4e5f6"  # ← Points to parent!
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    """Add priority column with default value."""
    op.add_column(
        "tasks",
        sa.Column("priority", sa.String(10), server_default="medium", nullable=False),
    )


def downgrade() -> None:
    """Remove priority column."""
    op.drop_column("tasks", "priority")
```

**Apply it:**

```bash
alembic upgrade head
```

```
INFO  [alembic.runtime.migration] Running upgrade a1b2c3d4e5f6 -> d4e5f6g7h8i9, add priority column to tasks
```

**Check the database:**

```sql
taskmanager=# \d tasks
                                     Table "public.tasks"
   Column   |          Type          | Nullable |              Default
------------+------------------------+----------+-----------------------------------
 id         | integer                | not null | nextval('tasks_id_seq'::regclass)
 title      | character varying(200) | not null |
 completed  | boolean                |          | false
 created_at | timestamp              |          | now()
 priority   | character varying(10)  | not null | 'medium'::character varying

taskmanager=# SELECT * FROM alembic_version;
 version_num
--------------
 d4e5f6g7h8i9
```

**The column exists. If there had been existing data, every row would get `'medium'` as the priority. No data lost.**

> "THIS is what we couldn't do before. We ALTERED the table — added a column — while keeping everything intact. That one line, `op.add_column(...)`, is the difference between losing your users' data and not."

---

## 3.6 Inspecting State (current, history)

**Know where you are and where you've been:**

```bash
# Where is my database right now?
alembic current
```

```
d4e5f6g7h8i9 (head)
```

```bash
# Show the full migration history
alembic history
```

```
a1b2c3d4e5f6 -> d4e5f6g7h8i9 (head), add priority column to tasks
<base> -> a1b2c3d4e5f6, create tasks table
```

```bash
# Verbose history with dates
alembic history --verbose
```

```
Rev: d4e5f6g7h8i9 (head)
Parent: a1b2c3d4e5f6
Path: alembic/versions/d4e5f6g7h8i9_add_priority_column_to_tasks.py

    add priority column to tasks

    Revision ID: d4e5f6g7h8i9
    Revises: a1b2c3d4e5f6
    Create Date: 2025-06-15 15:00:00.000000

Rev: a1b2c3d4e5f6
Parent: <base>
Path: alembic/versions/a1b2c3d4e5f6_create_tasks_table.py

    create tasks table

    Revision ID: a1b2c3d4e5f6
    Revises:
    Create Date: 2025-06-15 14:30:00.000000
```

**Compare to Git:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  alembic current     ≈    git log --oneline -1  (HEAD)          │
│  alembic history     ≈    git log --oneline                     │
│  alembic history -v  ≈    git log (full details)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: AUTOGENERATION

## 4.1 How Autogenerate Works (The Diff Engine)

**Writing migrations by hand works, but it's tedious and error-prone. Alembic can detect changes automatically.**

```
┌─────────────────────────────────────────────────────────────────┐
│              AUTOGENERATE: THE DIFF ENGINE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Think of it like `git diff`, but for database schemas:         │
│                                                                 │
│                                                                 │
│  ┌─────────────────────┐     ┌─────────────────────┐            │
│  │   Your SQLAlchemy   │     │   Your Actual        │           │
│  │   Models (Python)   │     │   Database (PG)      │           │
│  │                     │     │                      │           │
│  │  "What SHOULD       │     │  "What DOES          │           │
│  │   the DB look like" │     │   the DB look like"  │           │
│  └──────────┬──────────┘     └──────────┬───────────┘           │
│             │                           │                       │
│             └─────────┬─────────────────┘                       │
│                       │                                         │
│                       ▼                                         │
│               ┌───────────────┐                                 │
│               │   COMPARE     │                                 │
│               │   (diff)      │                                 │
│               └───────┬───────┘                                 │
│                       │                                         │
│                       ▼                                         │
│               ┌───────────────┐                                 │
│               │  Migration    │                                 │
│               │  file with    │                                 │
│               │  the changes  │                                 │
│               └───────────────┘                                 │
│                                                                 │
│                                                                 │
│  This is why target_metadata in env.py matters!                 │
│  Without it, Alembic has nothing to compare against.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Autogenerate in Practice

**Let's add a `Category` model and a foreign key, then let Alembic figure it out.**

First, update your models:

```python
# src/models.py — add Category and update Task
from datetime import datetime
from typing import Optional

from sqlalchemy import ForeignKey, String, DateTime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Category(Base):
    """NEW table — Alembic should detect this."""
    __tablename__ = "categories"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True)

    # Relationship (Lecture 2 this week)
    tasks: Mapped[list["Task"]] = relationship(back_populates="category")


class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    completed: Mapped[bool] = mapped_column(default=False)
    priority: Mapped[str] = mapped_column(String(10), server_default="medium")
    created_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.utcnow
    )

    # NEW: foreign key to categories
    category_id: Mapped[Optional[int]] = mapped_column(
        ForeignKey("categories.id"), nullable=True
    )

    # NEW: relationship
    category: Mapped[Optional["Category"]] = relationship(
        back_populates="tasks"
    )
```

**Now let Alembic detect the changes:**

```bash
alembic revision --autogenerate -m "add categories table and task category FK"
```

Output:
```
INFO  [alembic.autogenerate.compare] Detected added table 'categories'
INFO  [alembic.autogenerate.compare] Detected added column 'tasks.category_id'
INFO  [alembic.autogenerate.compare] Detected added foreign key
  (category_id)(id) on table tasks
Generating alembic/versions/j1k2l3m4_add_categories_table_and_task_category_fk.py ... done
```

**Alembic detected THREE changes automatically.**

**Open the generated file:**

```python
# alembic/versions/j1k2l3m4_add_categories_table_and_task_category_fk.py

"""add categories table and task category FK

Revision ID: j1k2l3m4
Revises: d4e5f6g7h8i9
Create Date: 2025-06-15 16:00:00.000000
"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


revision: str = "j1k2l3m4"
down_revision: Union[str, None] = "d4e5f6g7h8i9"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.create_table(
        "categories",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("name", sa.String(length=100), nullable=False),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("name"),
    )
    op.add_column(
        "tasks",
        sa.Column("category_id", sa.Integer(), nullable=True),
    )
    op.create_foreign_key(
        op.f("fk_tasks_category_id_categories"),  # constraint name
        "tasks",            # source table
        "categories",       # target table
        ["category_id"],    # source columns
        ["id"],             # target columns
    )
    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_constraint(
        op.f("fk_tasks_category_id_categories"),
        "tasks",
        type_="foreignkey",
    )
    op.drop_column("tasks", "category_id")
    op.drop_table("categories")
    # ### end Alembic commands ###
```

**Study the generated code carefully:**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHAT AUTOGENERATE PRODUCED                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  upgrade():                                                     │
│  1. Create the categories table      ← New model detected       │
│  2. Add category_id column to tasks  ← New column detected      │
│  3. Create foreign key constraint    ← ForeignKey detected      │
│                                                                 │
│  downgrade():                                                   │
│  1. Drop the foreign key constraint  ← Reverse of step 3        │
│  2. Drop the category_id column      ← Reverse of step 2        │
│  3. Drop the categories table        ← Reverse of step 1        │
│                                                                 │
│  ORDER MATTERS:                                                 │
│  ├─ upgrade: create table BEFORE adding FK (table must exist)   │
│  └─ downgrade: drop FK BEFORE dropping table (constraint first) │
│                                                                 │
│  Alembic figured out the correct order automatically.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Apply it:**

```bash
alembic upgrade head
```

```
INFO  [alembic.runtime.migration] Running upgrade d4e5f6g7h8i9 -> j1k2l3m4,
  add categories table and task category FK
```

**Verify:**

```sql
taskmanager=# \dt
              List of relations
 Schema |      Name       | Type  | Owner
--------+-----------------+-------+-------
 public | alembic_version | table | user
 public | categories      | table | user
 public | tasks           | table | user
(3 rows)

taskmanager=# \d tasks
   Column      |          Type          | Nullable |              Default
---------------+------------------------+----------+----------------------------------
 id            | integer                | not null | nextval('tasks_id_seq'::regclass)
 title         | character varying(200) | not null |
 completed     | boolean                |          | false
 created_at    | timestamp              |          | now()
 priority      | character varying(10)  | not null | 'medium'::character varying
 category_id   | integer                |          |
Foreign-key constraints:
    "fk_tasks_category_id_categories" FOREIGN KEY (category_id) REFERENCES categories(id)
```

---

## 4.3 What Autogenerate CAN Detect

```
┌─────────────────────────────────────────────────────────────────┐
│              AUTOGENERATE CAN DETECT ✅                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TABLE-LEVEL CHANGES:                                           │
│  ├─ New table added (new model class)                           │
│  └─ Table removed (model class deleted)                         │
│                                                                 │
│  COLUMN-LEVEL CHANGES:                                          │
│  ├─ Column added to existing table                              │
│  ├─ Column removed from existing table                          │
│  ├─ Column nullable changed (nullable=True → False)             │
│  └─ Column type changed (String → Text) *sometimes*             │
│                                                                 │
│  CONSTRAINT CHANGES:                                            │
│  ├─ Foreign key added or removed                                │
│  ├─ Unique constraint added or removed                          │
│  └─ Primary key changes                                         │
│                                                                 │
│  INDEX CHANGES:                                                 │
│  ├─ Index added (if defined in model)                           │
│  └─ Index removed                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 What Autogenerate CANNOT Detect

**This is critical. Autogenerate is NOT omniscient.**

```
┌─────────────────────────────────────────────────────────────────┐
│              AUTOGENERATE CANNOT DETECT ❌                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TABLE/COLUMN RENAMES:                                          │
│  ├─ Renaming a table        → Detected as DROP old + CREATE new │
│  └─ Renaming a column       → Detected as DROP old + ADD new    │
│                                                                 │
│     Model change:  title → name                                 │
│     Autogenerate:  DROP COLUMN title, ADD COLUMN name           │
│     Reality:       You wanted RENAME, not drop+add!             │
│     Impact:        ALL DATA IN "title" IS LOST.                 │
│                                                                 │
│                                                                 │
│  DATA CHANGES:                                                  │
│  ├─ Populating new columns with calculated values               │
│  ├─ Moving data between tables                                  │
│  └─ Transforming existing data                                  │
│                                                                 │
│  SOME COLUMN PROPERTY CHANGES:                                  │
│  ├─ server_default changes (depends on backend)                 │
│  └─ Comment changes                                             │
│                                                                 │
│  CONDITIONAL LOGIC:                                             │
│  ├─ "Add column only if table has > 1000 rows"                  │
│  └─ "Change type based on database vendor"                      │
│                                                                 │
│  POSTGRESQL-SPECIFIC FEATURES:                                  │
│  ├─ Enum type changes (adding/removing values)                  │
│  ├─ Trigger creation                                            │
│  └─ Stored procedures                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The rename trap — visualized:**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE RENAME TRAP                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What you DID in Python:                                        │
│      title: Mapped[str]  →  name: Mapped[str]                   │
│      (just renamed the column)                                  │
│                                                                 │
│  What autogenerate THINKS happened:                             │
│      "title column was removed, name column was added"          │
│                                                                 │
│  What autogenerate GENERATES:                                   │
│      op.drop_column("tasks", "title")   ← 💀 DATA DESTROYED    │
│      op.add_column("tasks",                                     │
│          sa.Column("name", sa.String(200)))                     │
│                                                                 │
│  What you ACTUALLY NEEDED:                                      │
│      op.alter_column("tasks",                                   │
│          "title",                                               │
│          new_column_name="name")         ← ✅ DATA PRESERVED    │
│                                                                 │
│  Autogenerate cannot know the difference between                │
│  "removed one thing and added another" and "renamed."           │
│  YOU must catch this during review.                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 The Golden Rule: Always Review

**Never blindly apply an autogenerated migration.**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│              ⚠️  THE GOLDEN RULE OF AUTOGENERATE                │
│                                                                 │
│     ALWAYS read the generated migration before running it.      │
│                                                                 │
│     Every. Single. Time.                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Your autogenerate workflow should be:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE CORRECT AUTOGENERATE WORKFLOW                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Change your SQLAlchemy models                               │
│                                                                 │
│  2. Generate:                                                   │
│     $ alembic revision --autogenerate -m "description"          │
│                                                                 │
│  3. ⚠️  OPEN THE FILE AND READ IT                               │
│     ├─ Does upgrade() do what you intended?                     │
│     ├─ Does downgrade() properly reverse it?                    │
│     ├─ Are there any DROP + ADD that should be RENAME?          │
│     ├─ Are column types correct?                                │
│     └─ Does order of operations make sense?                     │
│                                                                 │
│  4. Edit if needed (you CAN and SHOULD modify autogenerated     │
│     code — it's a starting point, not gospel)                   │
│                                                                 │
│  5. TEST: Apply and verify                                      │
│     $ alembic upgrade head                                      │
│     $ # check database structure                                │
│     $ alembic downgrade -1                                      │
│     $ # verify downgrade works too                              │
│     $ alembic upgrade head                                      │
│     $ # re-apply — full round trip                              │
│                                                                 │
│  6. Commit the migration file to Git                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Think of the autogenerate comment in every generated file:**

```python
# ### commands auto generated by Alembic - please adjust! ###
```

That comment says "please adjust." It's not saying "this is perfect." It's saying "I did my best — you verify."

---

# PART 5: ADVANCED PATTERNS & BEST PRACTICES

## 5.1 Data Migrations (Not Just Schema Changes)

**Sometimes you need to change DATA, not just structure.**

Scenario: You have a `priority` column with string values (`"high"`, `"medium"`, `"low"`). You want to change it to an integer (`3`, `2`, `1`) for sorting.

This is a **three-step migration** — schema change alone isn't enough:

```python
"""convert priority from string to integer

Revision ID: x1y2z3
Revises: j1k2l3m4
"""
from alembic import op
import sqlalchemy as sa


# We need to talk to the data, so we need a reference to the table
# This is a lightweight way to query inside a migration without
# importing your full model (which may have already changed further)

def upgrade() -> None:
    # STEP 1: Add the new integer column
    op.add_column(
        "tasks",
        sa.Column("priority_int", sa.Integer(), nullable=True),
    )

    # STEP 2: Migrate the DATA
    # Map old string values to new integer values
    op.execute("""
        UPDATE tasks SET priority_int = CASE
            WHEN priority = 'high'   THEN 3
            WHEN priority = 'medium' THEN 2
            WHEN priority = 'low'    THEN 1
            ELSE 2
        END
    """)

    # STEP 3: Remove old column, rename new one
    op.drop_column("tasks", "priority")
    op.alter_column("tasks", "priority_int", new_column_name="priority")
    op.alter_column(
        "tasks", "priority",
        nullable=False,
        server_default="2",
    )


def downgrade() -> None:
    # Reverse: integer back to string
    op.add_column(
        "tasks",
        sa.Column("priority_str", sa.String(10), nullable=True),
    )

    op.execute("""
        UPDATE tasks SET priority_str = CASE
            WHEN priority = 3 THEN 'high'
            WHEN priority = 2 THEN 'medium'
            WHEN priority = 1 THEN 'low'
            ELSE 'medium'
        END
    """)

    op.drop_column("tasks", "priority")
    op.alter_column("tasks", "priority_str", new_column_name="priority")
    op.alter_column(
        "tasks", "priority",
        nullable=False,
        server_default="'medium'",
    )
```

**Visualize the three-step process:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DATA MIGRATION: THREE STEPS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE:                                                        │
│  ┌────┬──────────────────┬──────────┐                           │
│  │ id │ title            │ priority │                           │
│  ├────┼──────────────────┼──────────┤                           │
│  │  1 │ Deploy v1        │ "high"   │                           │
│  │  2 │ Write docs       │ "medium" │                           │
│  │  3 │ Fix auth bug     │ "low"    │                           │
│  └────┴──────────────────┴──────────┘                           │
│                                                                 │
│  STEP 1 — Add new column:                                       │
│  ┌────┬──────────────────┬──────────┬──────────────┐            │
│  │ id │ title            │ priority │ priority_int │            │
│  ├────┼──────────────────┼──────────┼──────────────┤            │
│  │  1 │ Deploy v1        │ "high"   │ NULL         │            │
│  │  2 │ Write docs       │ "medium" │ NULL         │            │
│  │  3 │ Fix auth bug     │ "low"    │ NULL         │            │
│  └────┴──────────────────┴──────────┴──────────────┘            │
│                                                                 │
│  STEP 2 — Populate new column:                                  │
│  ┌────┬──────────────────┬──────────┬──────────────┐            │
│  │ id │ title            │ priority │ priority_int │            │
│  ├────┼──────────────────┼──────────┼──────────────┤            │
│  │  1 │ Deploy v1        │ "high"   │ 3            │            │
│  │  2 │ Write docs       │ "medium" │ 2            │            │
│  │  3 │ Fix auth bug     │ "low"    │ 1            │            │
│  └────┴──────────────────┴──────────┴──────────────┘            │
│                                                                 │
│  STEP 3 — Drop old, rename new:                                 │
│  ┌────┬──────────────────┬──────────┐                           │
│  │ id │ title            │ priority │                           │
│  ├────┼──────────────────┼──────────┤                           │
│  │  1 │ Deploy v1        │ 3        │                           │
│  │  2 │ Write docs       │ 2        │                           │
│  │  3 │ Fix auth bug     │ 1        │                           │
│  └────┴──────────────────┴──────────┘                           │
│                                                                 │
│  Data PRESERVED. Type CHANGED. No data loss.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key principle:**

> "Autogenerate will NEVER create data migrations. It only sees structure. If you need to move, transform, or populate data, you MUST write that migration by hand. This is why even teams that use autogenerate still need developers who understand manual migrations."

---

## 5.2 Rollback Strategies

**Not every migration can be cleanly undone.**

```
┌─────────────────────────────────────────────────────────────────┐
│                ROLLBACK STRATEGIES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EASY TO REVERSE (fully reversible):                            │
│  ├─ Add column       → downgrade: drop column                   │
│  ├─ Create table     → downgrade: drop table                    │
│  ├─ Add index        → downgrade: drop index                    │
│  └─ Add constraint   → downgrade: drop constraint               │
│                                                                 │
│  DANGEROUS TO REVERSE (data loss risk):                         │
│  ├─ Drop column      → downgrade: add column (data is GONE)     │
│  ├─ Drop table       → downgrade: create table (data is GONE)   │
│  └─ Change type      → downgrade: change back (data truncated?) │
│                                                                 │
│  IMPOSSIBLE TO REVERSE PERFECTLY:                               │
│  ├─ Drop column with no backup   → data cannot be recovered     │
│  ├─ Lossy type conversion        → precision lost forever       │
│  └─ Data deletion migrations     → deleted data cannot return   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What do you do when downgrade can't fully undo the upgrade?**

Option 1: Implement a "best effort" downgrade and document the limitation.

```python
def downgrade() -> None:
    # WARNING: This re-creates the column but data is lost.
    # Data must be restored from backup if needed.
    op.add_column(
        "tasks",
        sa.Column("description", sa.Text(), nullable=True),
    )
```

Option 2: Raise an explicit error so nobody runs it accidentally.

```python
def downgrade() -> None:
    raise RuntimeError(
        "This migration cannot be reversed without data loss. "
        "Restore from backup instead. "
        "See: docs/migrations/m3_rollback_plan.md"
    )
```

**The right approach depends on the situation:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Development environment:                                       │
│  └─ Downgrade freely. Data doesn't matter. Speed matters.       │
│                                                                 │
│  Staging environment:                                           │
│  └─ Test downgrades here. Verify they work. Then plan for prod. │
│                                                                 │
│  Production environment:                                        │
│  └─ RARELY downgrade. Usually: fix forward with a NEW migration │
│     instead of rolling back. Backup before any destructive ops. │
│                                                                 │
│  "Fix forward" means: if migration 5 has a bug, don't           │
│  downgrade to 4. Instead, create migration 6 that fixes         │
│  the bug. The history stays clean and moves in one direction.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Migrations in a Team (Conflicts and Merges)

**What happens when two developers create migrations at the same time?**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE BRANCHING PROBLEM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alice and Bob both start from migration d4e5f6:                │
│                                                                 │
│  d4e5f6 (current head)                                          │
│    ├─── Alice: "add categories table" (aaaa1111)                │
│    └─── Bob:   "add tags table"       (bbbb2222)                │
│                                                                 │
│  Both have down_revision = d4e5f6                               │
│  Alembic now has TWO heads. It can't resolve this.              │
│                                                                 │
│                                                                 │
│  $ alembic heads                                                │
│  aaaa1111 (head)                                                │
│  bbbb2222 (head)                                                │
│                                                                 │
│  $ alembic upgrade head                                         │
│  ERROR: Multiple heads detected!                                │
│                                                                 │
│  Sound familiar? It's a MERGE CONFLICT — same as Git.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to fix it: the merge migration.**

```bash
# Alembic can create a merge point (like a git merge commit)
alembic merge heads -m "merge alice and bob migrations"
```

This generates:

```python
"""merge alice and bob migrations

Revision ID: cccc3333
Revises: aaaa1111, bbbb2222
"""

revision: str = "cccc3333"
down_revision = ("aaaa1111", "bbbb2222")  # ← TWO parents!


def upgrade() -> None:
    pass  # Nothing to do — just joining the two branches


def downgrade() -> None:
    pass
```

```
┌─────────────────────────────────────────────────────────────────┐
│                AFTER MERGE                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  d4e5f6                                                         │
│    ├─── aaaa1111 (Alice: categories) ───┐                       │
│    └─── bbbb2222 (Bob: tags) ───────────┤                       │
│                                         ▼                       │
│                                    cccc3333 (merge)             │
│                                         │                       │
│                                       (head)                    │
│                                                                 │
│  Now there's one head again. Crisis averted.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Prevention is better than cure:**

```
┌─────────────────────────────────────────────────────────────────┐
│            AVOIDING MIGRATION CONFLICTS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Pull before you generate                                    │
│     $ git pull                                                  │
│     $ alembic upgrade head     ← Apply teammate's migrations    │
│     $ alembic revision --autogenerate -m "my change"            │
│                                                                 │
│  2. Communicate with your team                                  │
│     "I'm about to create a migration for the users table."      │
│     Prevents two people touching the same tables simultaneously │
│                                                                 │
│  3. Keep migrations small                                       │
│     One migration per logical change.                           │
│     Smaller changes = fewer conflicts.                          │
│                                                                 │
│  4. Merge quickly                                               │
│     Don't let migration branches live on feature branches       │
│     for weeks. Merge frequently.                                │
│                                                                 │
│  Same principles as Git branch management from Week 1.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Best Practices Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│              MIGRATION BEST PRACTICES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NAMING:                                                        │
│  ├─ Use descriptive messages: "add priority column to tasks"    │
│  ├─ NOT: "update models", "fix stuff", "migration 3"           │
│  └─ Future-you will read this. Be kind to future-you.           │
│     (Same principle as conventional commits from Week 1.)       │
│                                                                 │
│  ONE CHANGE PER MIGRATION:                                      │
│  ├─ Add categories table → one migration                        │
│  ├─ Add FK from tasks to categories → another migration         │
│  ├─ NOT: one giant migration that creates 5 tables              │
│  └─ Small migrations are easier to debug and rollback           │
│                                                                 │
│  ALWAYS TEST BOTH DIRECTIONS:                                   │
│  ├─ $ alembic upgrade head      (apply)                         │
│  ├─ $ alembic downgrade -1      (undo)                          │
│  └─ $ alembic upgrade head      (re-apply)                      │
│  If any step fails, fix it BEFORE committing.                   │
│                                                                 │
│  NEVER EDIT APPLIED MIGRATIONS:                                 │
│  ├─ Once a migration is applied on ANY environment, it's frozen │
│  ├─ Don't fix a bug by editing the file — create a NEW one      │
│  └─ Editing breaks the contract: teammates already ran it       │
│                                                                 │
│  ALWAYS COMMIT MIGRATION FILES:                                 │
│  ├─ Migration files go in Git. Always.                          │
│  └─ They are source code, not generated artifacts.              │
│                                                                 │
│  USE server_default FOR NEW NOT-NULL COLUMNS:                   │
│  ├─ Adding a NOT NULL column to a table with existing data?     │
│  ├─ MUST provide server_default so existing rows get a value    │
│  └─ Without it: ERROR. Existing rows violate NOT NULL.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The `server_default` trap — demonstrate it:**

```python
# ❌ WRONG: Adding NOT NULL column without default
def upgrade() -> None:
    op.add_column(
        "tasks",
        sa.Column("assignee", sa.String(100), nullable=False),
    )
# ERROR: column "assignee" of relation "tasks" contains null values
# Existing rows have no value for "assignee"!


# ✅ CORRECT: Provide a server_default for existing rows
def upgrade() -> None:
    op.add_column(
        "tasks",
        sa.Column(
            "assignee",
            sa.String(100),
            nullable=False,
            server_default="unassigned",
        ),
    )
# All existing rows get "unassigned". New rows need explicit value.
# Optionally, remove the default afterward if you don't want it permanent:
# op.alter_column("tasks", "assignee", server_default=None)
```

---

## 5.5 Migrations in CI (Preview for Week 15)

**A brief look ahead at how this fits into real deployment pipelines:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MIGRATIONS IN CI/CD (PREVIEW)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In Week 15 (CI/CD), you'll set up pipelines that:              │
│                                                                 │
│  1. On every Pull Request:                                      │
│     ├─ Spin up a test database (Docker)                         │
│     ├─ Run: alembic upgrade head (apply all migrations)         │
│     ├─ Run: alembic downgrade base (test all downgrades)        │
│     ├─ Run: alembic upgrade head (verify round-trip)            │
│     └─ Run: your test suite against the migrated database       │
│                                                                 │
│  2. On deploy to staging/production:                            │
│     ├─ alembic upgrade head (apply pending migrations)          │
│     ├─ If migration fails → deployment stops                    │
│     └─ Health check confirms application works                  │
│                                                                 │
│  This is why downgrade MUST work:                               │
│  CI tests it automatically. Broken downgrades fail the build.   │
│                                                                 │
│  For now: just know that the habits you build today             │
│  (testing both directions, reviewing generated code)            │
│  will be ENFORCED by your CI pipeline later.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.6 Common Mistakes and How to Avoid Them

### Mistake 1: Editing a migration that's already been applied

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Scenario: You applied migration aaaa1111, then realized        │
│  you forgot to add an index.                                    │
│                                                                 │
│  ❌ WRONG: Edit aaaa1111 and add the index there                │
│                                                                 │
│     Why it breaks:                                              │
│     ├─ Your database already ran aaaa1111's upgrade()            │
│     ├─ alembic_version says "aaaa1111 is done"                  │
│     ├─ Alembic will NEVER re-run aaaa1111                       │
│     ├─ Your database never gets the index                       │
│     ├─ Your teammate runs upgrade head — they get the index     │
│     └─ Now YOUR database differs from THEIRS. Silent bug.       │
│                                                                 │
│  ✅ CORRECT: Create a NEW migration                             │
│                                                                 │
│     $ alembic revision -m "add index on tasks.priority"         │
│     # Write the index in this new migration                     │
│     # Everyone applies the new migration. All databases match.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 2: Using metadata.create_all alongside Alembic

```python
# ❌ WRONG: Mixing two schema management strategies
engine = create_engine(DATABASE_URL)
Base.metadata.create_all(engine)   # Creates tables WITHOUT Alembic knowing!

# Now alembic_version says "no migrations applied"
# But tables already exist.
# Next: alembic upgrade head → ERROR: table "tasks" already exists

# ✅ CORRECT: Let Alembic be the ONLY way tables are created
# Delete metadata.create_all from your application code.
# Use: alembic upgrade head to create tables.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Once you adopt Alembic, metadata.create_all is RETIRED.        │
│                                                                 │
│  Alembic owns the schema. Period.                               │
│                                                                 │
│  The only exception: test databases (Week 2 testing patterns).  │
│  In tests, you MAY use create_all for speed because the         │
│  test database is disposable. But your real database:            │
│  Alembic only.                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 3: Not specifying server_default for NOT NULL columns

```python
# ❌ Table has 10,000 rows. This migration FAILS:
def upgrade() -> None:
    op.add_column(
        "tasks",
        sa.Column("status", sa.String(20), nullable=False),
        # No server_default → existing rows have NULL
        # NULL violates NOT NULL → ERROR
    )

# ✅ Provide a default for existing rows:
def upgrade() -> None:
    op.add_column(
        "tasks",
        sa.Column(
            "status", sa.String(20),
            nullable=False,
            server_default="pending",
        ),
    )
```

### Mistake 4: Forgetting to commit migration files

```bash
# You generated a migration, applied it locally, pushed your code.
# But you forgot to git add the migration file.

# Your teammate:
$ git pull
$ alembic upgrade head
# "No new migrations." ← They don't have the file!
# Their database is out of sync. Your app crashes on their machine.

# ALWAYS:
$ git add alembic/versions/
$ git commit -m "feat: add priority column migration"
```

### Mistake 5: Making autogenerate with stale database

```bash
# Your database is behind. You haven't applied recent migrations.

# ❌ WRONG:
$ alembic revision --autogenerate -m "add tags"
# Autogenerate compares models to the CURRENT database state.
# If your DB is 3 migrations behind, autogenerate will include
# changes from those 3 migrations AGAIN. Duplicate operations!

# ✅ CORRECT:
$ alembic upgrade head          # ← Get database up to date FIRST
$ alembic revision --autogenerate -m "add tags"  # NOW generate
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                  ALEMBIC QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SETUP (once per project):                                      │
│      pip install alembic                                        │
│      alembic init alembic                                       │
│      # Edit alembic.ini → set sqlalchemy.url                    │
│      # Edit alembic/env.py → set target_metadata = Base.metadata│
│                                                                 │
│  CREATE MIGRATION (manual):                                     │
│      alembic revision -m "description"                          │
│      # Edit the generated file: fill in upgrade() & downgrade() │
│                                                                 │
│  CREATE MIGRATION (autogenerate):                               │
│      alembic revision --autogenerate -m "description"           │
│      # ⚠️ REVIEW the generated file before applying!            │
│                                                                 │
│  APPLY MIGRATIONS:                                              │
│      alembic upgrade head     Apply all pending                 │
│      alembic upgrade +1       Apply next one                    │
│      alembic upgrade abc123   Apply up to specific revision     │
│                                                                 │
│  UNDO MIGRATIONS:                                               │
│      alembic downgrade -1     Undo last one                     │
│      alembic downgrade abc123 Downgrade to specific revision    │
│      alembic downgrade base   Undo everything                   │
│                                                                 │
│  INSPECT STATE:                                                 │
│      alembic current          What's applied now?               │
│      alembic history          Show all migrations               │
│      alembic heads            Show latest migration(s)          │
│                                                                 │
│  RESOLVE CONFLICTS:                                             │
│      alembic merge heads -m "merge branch migrations"           │
│                                                                 │
│  KEY OPERATIONS (inside migration files):                       │
│      op.create_table("name", columns...)                        │
│      op.drop_table("name")                                      │
│      op.add_column("table", sa.Column(...))                     │
│      op.drop_column("table", "column_name")                     │
│      op.alter_column("table", "col", new_column_name="new")     │
│      op.create_index("idx_name", "table", ["col1", "col2"])     │
│      op.drop_index("idx_name")                                  │
│      op.create_foreign_key("fk_name", "src", "tgt", [...],[...])│
│      op.drop_constraint("name", "table", type_="foreignkey")    │
│      op.execute("RAW SQL HERE")                                 │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Edit applied migration   → Create a new one instead     │
│      ❌ metadata.create_all      → Alembic owns the schema      │
│      ❌ Skip reviewing autogen   → Always read before applying  │
│      ❌ NOT NULL without default → Provide server_default       │
│      ❌ Forget to git add        → Migration files are code      │
│      ❌ Autogen on stale DB      → upgrade head first            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ALEMBIC = VERSION CONTROL FOR YOUR DATABASE SCHEMA             │
│                                                                 │
│  Your database evolves over time, just like your code.          │
│  Alembic tracks each change as a migration — a reversible,      │
│  ordered record of what changed and how to undo it.             │
│                                                                 │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                    │
│  │ Model   │     │ Alembic │     │  Real   │                    │
│  │ (Python)│────▶│  (diff) │────▶│   DB    │                    │
│  └─────────┘     └─────────┘     └─────────┘                    │
│  What SHOULD       Computes       What DOES                     │
│  the DB look       the change     the DB look                   │
│  like?             needed         like?                         │
│                                                                 │
│                                                                 │
│  THE GIT ANALOGY:                                               │
│  ├─ alembic init        = git init                              │
│  ├─ alembic revision    = git commit                            │
│  ├─ alembic upgrade     = git checkout (forward)                │
│  ├─ alembic downgrade   = git revert                            │
│  ├─ alembic history     = git log                               │
│  ├─ alembic current     = git log -1                            │
│  ├─ alembic merge       = git merge                             │
│  └─ alembic_version     = .git/HEAD                             │
│                                                                 │
│  GOLDEN RULES:                                                  │
│  ├─ Alembic owns the schema. Retire metadata.create_all.        │
│  ├─ Always review autogenerated migrations.                     │
│  ├─ Never edit an applied migration. Create a new one.          │
│  ├─ Test both upgrade AND downgrade.                            │
│  └─ Commit every migration file to Git.                         │
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
│  WEEK 6, LECTURE 4 (Next lecture):                               │
│  └─ Async SQLAlchemy & FastAPI Integration                      │
│     You'll connect Alembic-managed tables to real API           │
│     endpoints. Migrations handle the schema,                    │
│     async SQLAlchemy handles the queries.                       │
│                                                                 │
│  WEEK 6 PROJECT: Refactor Task Manager to PostgreSQL            │
│  └─ Your Task Manager currently stores data in memory.          │
│     You'll create Alembic migrations for the full schema:       │
│     tasks, categories, tags — all with foreign keys.            │
│     No more metadata.create_all. Alembic from day one.          │
│                                                                 │
│  WEEK 7: Advanced Database Patterns                             │
│  └─ Query optimization with indexes. You'll create index        │
│     migrations and measure before/after performance             │
│     using EXPLAIN (Week 5 knowledge + Alembic).                 │
│                                                                 │
│  WEEK 9: Authentication                                         │
│  └─ Adding users and roles tables to your existing database.    │
│     This is EXACTLY the "evolving schema" problem:              │
│     add new tables alongside existing ones without breaking     │
│     anything. Migration skills are essential.                   │
│                                                                 │
│  WEEK 15: CI/CD                                                 │
│  └─ Your migration tests become automated.                      │
│     GitHub Actions will run upgrade + downgrade + upgrade       │
│     on every pull request. Broken migrations = blocked merge.   │
│                                                                 │
│  WEEK 13-14: Capstone                                           │
│  └─ Multi-tenant SaaS backend with 10+ tables.                  │
│     Your migration history will be 15-20 files long.            │
│     Clean migration habits NOW pay off massively THEN.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```