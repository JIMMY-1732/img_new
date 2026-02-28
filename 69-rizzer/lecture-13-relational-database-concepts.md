# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, THEORY LATER                                    │
│  ────────────────────────────                                   │
│  Students must SEE their in-memory Task Manager break before    │
│  they'll care about databases. We'll destroy their data.        │
│  In front of them.                                              │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Databases are abstract infrastructure. We use a "Spreadsheet   │
│  That Grew Up" analogy throughout. Every concept maps to        │
│  something they've already touched in Excel or Google Sheets.   │
│                                                                 │
│  BUILD MENTAL MODEL BEFORE SQL                                  │
│  ────────────────────────────                                   │
│  Table diagrams come before CREATE TABLE. Understand the        │
│  structure before typing any SQL. SQL syntax is Lecture 2.      │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Type hints (Week 1) → Column data types                        │
│  Dataclasses (Week 1) → Table schemas (fields = columns)        │
│  Pydantic models (Week 3) → Constraints & validation            │
│  In-memory Task Manager (Week 3-4) → What we're replacing       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# UPDATED LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                 RELATIONAL DATABASE CONCEPTS                    │
│                  (4 Hour Lecture — Extended)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM                                            │
│  ├─ 1.1 Your Task Manager is a House of Cards (Demonstration)   │
│  ├─ 1.2 Five Ways In-Memory Storage Will Ruin Your Day          │
│  ├─ 1.3 What is a Database, Really?                             │
│  ├─ 1.4 The Spreadsheet Analogy (Our Guide for Today)           │
│  └─ 1.5 The Client-Server Architecture              [NEW]       │
│                                                                 │
│  PART 2: THE MENTAL MODEL                                       │
│  ├─ 2.1 Tables, Rows, and Columns                               │
│  ├─ 2.2 Primary Keys (Every Row Needs an Identity)              │
│  ├─ 2.3 Foreign Keys (The Bridges Between Tables)               │
│  ├─ 2.4 Relationship Types (One-to-Many, Many-to-Many)          │
│  └─ 2.5 How Separated Data Gets Reunited (Join Concept) [NEW]   │
│                                                                 │
│  PART 3: DATA TYPES                                             │
│  ├─ 3.1 Why Types Matter (Connection to Week 1)                 │
│  ├─ 3.2 Core PostgreSQL Types (The Everyday Toolkit)            │
│  ├─ 3.3 PostgreSQL Power Types (UUID, JSONB, Arrays)            │
│  ├─ 3.4 Choosing the Right Type (Decision Framework)            │
│  └─ 3.5 How Databases Find Data: The Index          [NEW]       │
│                                                                 │
│  PART 4: DATA INTEGRITY & ACID                                  │
│  ├─ 4.0 NULL: The Concept of Missing Data           [NEW]       │
│  ├─ 4.1 Constraints (The Database's Pydantic)                   │
│  ├─ 4.2 ACID Properties (The Four Promises)                     │
│  ├─ 4.3 What Happens Without ACID (The Horror Story)            │
│  ├─ 4.4 Transactions Preview (The Safety Net)                   │
│  └─ 4.5 Database Security & Access Control          [NEW]       │
│                                                                 │
│  PART 5: NORMALIZATION                                          │
│  ├─ 5.1 The Copy-Paste Problem                                  │
│  ├─ 5.2 First Normal Form (1NF): One Value Per Cell             │
│  ├─ 5.3 Second Normal Form (2NF): No Partial Dependencies       │
│  ├─ 5.4 Third Normal Form (3NF): No Transitive Dependencies     │
│  └─ 5.5 Denormalization (When to Break the Rules)               │
│                                                                 │
│  PART 6: ENTITY-RELATIONSHIP DIAGRAMS                           │
│  ├─ 6.1 Design Before Code (Blueprints, Not Guesswork)          │
│  ├─ 6.2 ER Notation (The Vocabulary)                            │
│  └─ 6.3 Your Task Manager as an ER Diagram                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 Your Task Manager is a House of Cards

**Start with a demonstration. Use their own project against them.**

```python
# What students built in Weeks 3-4 (simplified)
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

# The "database" — a Python dictionary
tasks_db: dict[int, dict] = {}
next_id: int = 1


class TaskCreate(BaseModel):
    title: str
    category: str


@app.post("/tasks")
async def create_task(task: TaskCreate) -> dict:
    global next_id
    new_task = {"id": next_id, "title": task.title, "category": task.category}
    tasks_db[next_id] = new_task
    next_id += 1
    return new_task


@app.get("/tasks")
async def list_tasks() -> list[dict]:
    return list(tasks_db.values())
```

**Run it. Create some tasks. Show it working.**

```
$ uvicorn main:app --reload

# In another terminal:
$ curl -X POST "http://localhost:8000/tasks" \
    -H "Content-Type: application/json" \
    -d '{"title": "Build login API", "category": "Backend"}'

{"id": 1, "title": "Build login API", "category": "Backend"}

$ curl -X POST "http://localhost:8000/tasks" \
    -H "Content-Type: application/json" \
    -d '{"title": "Write unit tests", "category": "Testing"}'

{"id": 2, "title": "Write unit tests", "category": "Testing"}

$ curl "http://localhost:8000/tasks"

[
  {"id": 1, "title": "Build login API", "category": "Backend"},
  {"id": 2, "title": "Write unit tests", "category": "Testing"}
]
```

**Now kill the server. Restart it.**

```
^C   # Ctrl+C — server stops

$ uvicorn main:app --reload   # Restart

$ curl "http://localhost:8000/tasks"

[]    ← EVERYTHING IS GONE.
```

**Now ask the class:**

> "You just spent 30 seconds creating data. Imagine a real user who spent 6 months managing their projects in your app. They refresh the page after a deploy. Everything gone. Would you use this product?"

Answer: **No. Because your data lives in RAM, and RAM is amnesia.**

---

## 1.2 Five Ways In-Memory Storage Will Ruin Your Day

```
┌─────────────────────────────────────────────────────────────────┐
│              FIVE FAILURE MODES OF dict-AS-A-DATABASE            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ 1. DATA LOSS ON RESTART                                     │
│     Server crashes, deploys, restarts = all data gone           │
│     Your dict lives in memory. Memory is volatile.              │
│                                                                 │
│  ❌ 2. NO RELATIONSHIPS                                         │
│     Task has a "category" string. But is "backend" the same     │
│     as "Backend" or "BACKEND"? No way to enforce consistency.   │
│     What if you need: tasks → assigned users → organizations?   │
│     Nested dicts become spaghetti fast.                         │
│                                                                 │
│  ❌ 3. NO DATA INTEGRITY                                        │
│     Nothing stops you from storing {"id": "banana"}.            │
│     Your Pydantic model validates INPUT, but the dict will      │
│     happily store anything. No constraints on the data itself.  │
│                                                                 │
│  ❌ 4. RACE CONDITIONS                                          │
│     Two requests arrive simultaneously.                         │
│     Both read next_id = 5. Both write id 5. One task vanishes.  │
│     Your async server handles concurrent requests — your dict   │
│     does not handle concurrent writes.                          │
│                                                                 │
│  ❌ 5. NO QUERYING POWER                                        │
│     "Give me all tasks assigned to Alice, in the Backend        │
│     category, created this week, sorted by priority."           │
│     With a dict: write 20 lines of Python filtering.            │
│     Every. Single. Time.                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Show the race condition — make it real:**

```python
# Simulate what happens with concurrent requests to your Task Manager

import asyncio

next_id: int = 1
tasks_db: dict[int, dict] = {}


async def create_task(title: str) -> dict:
    global next_id
    
    # Both coroutines read next_id at the same time
    current_id = next_id
    
    await asyncio.sleep(0.01)  # Simulates tiny async gap (I/O yield)
    
    # Both coroutines write to the SAME id
    tasks_db[current_id] = {"id": current_id, "title": title}
    next_id = current_id + 1
    return tasks_db[current_id]


async def main():
    # Two tasks created "at the same time"
    results = await asyncio.gather(
        create_task("Task A"),
        create_task("Task B"),
    )
    
    print(f"Results: {results}")
    print(f"Database: {tasks_db}")
    print(f"Expected 2 tasks, got: {len(tasks_db)}")


asyncio.run(main())
```

Output:
```
Results: [{'id': 1, 'title': 'Task A'}, {'id': 1, 'title': 'Task B'}]
Database: {1: {'id': 1, 'title': 'Task B'}}
Expected 2 tasks, got: 1
```

**Task A was silently overwritten. No error. No warning. Data just vanished.**

> "Your async FastAPI server can receive hundreds of concurrent requests. Your Python dictionary has no concept of safe concurrent access. Every request is a coin flip."

---

## 1.3 What is a Database, Really?

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHAT IS A DATABASE?                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  At its core, a database is:                                    │
│                                                                 │
│  1. PERSISTENT STORAGE                                          │
│     Data survives restarts, crashes, power outages.             │
│     Written to disk, not just memory.                           │
│                                                                 │
│  2. STRUCTURED ORGANIZATION                                     │
│     Data follows a defined shape (schema).                      │
│     Like type hints for your storage.                           │
│                                                                 │
│  3. QUERY ENGINE                                                │
│     A language (SQL) to ask complex questions.                  │
│     "Find all tasks due this week" — one line, not twenty.      │
│                                                                 │
│  4. INTEGRITY ENFORCER                                          │
│     Rules that your data MUST follow.                           │
│     "Every task MUST have an owner" — enforced automatically.   │
│                                                                 │
│  5. CONCURRENCY MANAGER                                         │
│     Handles multiple reads/writes simultaneously.               │
│     No more race conditions. No more lost data.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A RELATIONAL database specifically means:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "RELATIONAL" = Data is organized into TABLES that can          │
│                 RELATE to each other through shared keys.       │
│                                                                 │
│     ┌─────────┐          ┌─────────┐                            │
│     │  USERS  │─────────▶│  TASKS  │                            │
│     └─────────┘ "owns"   └─────────┘                            │
│                               │                                 │
│                          "tagged with"                          │
│                               │                                 │
│                          ┌─────────┐                            │
│                          │  TAGS   │                            │
│                          └─────────┘                            │
│                                                                 │
│  The RELATIONSHIPS between tables are the defining feature.     │
│  This is what your dict of dicts could never do cleanly.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why PostgreSQL?**

> "There are many relational databases — MySQL, SQLite, MariaDB, SQL Server. We use PostgreSQL because it's the most feature-rich open-source option, it's the industry default for backend development, and it handles everything from side projects to systems serving millions of users."

---

## 1.4 The Spreadsheet Analogy (Our Guide for Today)

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE SPREADSHEET THAT GREW UP                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You've used Google Sheets or Excel. A database is a            │
│  spreadsheet that EVOLVED to solve real problems.               │
│                                                                 │
│                                                                 │
│  SPREADSHEET                                                    │
│  ────────────                                                   │
│  • Anyone can type anything in any cell                         │
│  • Copy-paste data everywhere                                   │
│  • One person editing at a time (or chaos)                      │
│  • "Undo" is your only safety net                               │
│  • Searching means Ctrl+F and scrolling                         │
│                                                                 │
│                        │                                        │
│                        │ Evolution                              │
│                        ▼                                        │
│                                                                 │
│  RELATIONAL DATABASE                                            │
│  ────────────────────                                           │
│  • Every column has an enforced type (no "banana" in age)       │
│  • Data normalized — stored once, referenced everywhere         │
│  • Thousands of users reading and writing simultaneously        │
│  • Transactions guarantee all-or-nothing operations             │
│  • SQL queries replace scrolling and manual filtering           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to database terms:**

```
Spreadsheet                 │  Relational Database
────────────────────────────│──────────────────────────
A workbook file             │  A database
A sheet / tab               │  A table
A row in the sheet          │  A row (record)
A column header             │  A column (field)
A cell                      │  A value
Column format ("Number")    │  Data type (INTEGER)
"This cell can't be empty"  │  NOT NULL constraint
"No duplicates in column A" │  UNIQUE constraint
Row number                  │  Primary key
"See Sheet2, Row 5"         │  Foreign key
Data validation dropdown    │  CHECK constraint / ENUM
Multiple tabs related       │  Tables with relationships
  by a shared column        │   through foreign keys
```

> "Every concept we learn today, ask yourself: 'What's the spreadsheet version of this?' That's your anchor. Then ask: 'Why isn't the spreadsheet version good enough?' That's your motivation."

## 1.5 The Client-Server Architecture

**PostgreSQL is not a Python library. It does not live inside your application process. It is a server — a completely separate, continuously running program that your code connects to over a network socket, even when both are on the same machine.**

This distinction has immediate practical consequences: connections cost time, multiple services can share the same database, and each side can fail independently of the other.

```
┌─────────────────────────────────────────────────────────────────┐
│              THE CLIENT-SERVER ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────┐       │
│   │            CLIENT — Your Python Code                │       │
│   │  ┌───────────────────┐   ┌──────────────────────┐   │       │
│   │  │  FastAPI App      │   │  Export Script       │   │       │
│   │  │  (port 8000)      │   │  (runs on schedule)  │   │       │
│   │  └────────┬──────────┘   └──────────┬───────────┘   │       │
│   └───────────│─────────────────────────│───────────────┘       │
│               │                         │                       │
│               └───────────┬─────────────┘                       │
│                           │  TCP/IP socket                      │
│                           │  (localhost or network)             │
│                           │  Default port: 5432                 │
│                           ▼                                     │
│   ┌─────────────────────────────────────────────────────┐       │
│   │         SERVER — PostgreSQL Process                 │       │
│   │                                                     │       │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │       │
│   │  │  tasks   │  │  users   │  │  categories      │  │       │
│   │  │  table   │  │  table   │  │  table           │  │       │
│   │  └──────────┘  └──────────┘  └──────────────────┘  │       │
│   │                                                     │       │
│   │  Data lives on DISK. Independent from clients.      │       │
│   │  Survives app crashes, restarts, and deploys.        │       │
│   └─────────────────────────────────────────────────────┘       │
│                                                                 │
│  KEY FACTS:                                                     │
│  ├─ Both services connect to PostgreSQL on the SAME port 5432   │
│  ├─ Connecting takes time — it is a real network handshake      │
│  ├─ If your app crashes → DB keeps running, data is safe        │
│  └─ If the DB crashes → app cannot read/write, but no data loss │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The connection string is how your app locates and authenticates to the server:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 CONNECTION STRING ANATOMY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  postgresql://app_user:my_password@localhost:5432/taskmanager   │
│               │           │         │         │    │            │
│               │           │         │         │    └─ Database  │
│               │           │         │         │       name      │
│               │           │         │         └─ Port (default  │
│               │           │         │              = 5432)      │
│               │           │         └─ Host ("localhost" =      │
│               │           │              same machine)          │
│               │           └─ Password  ← NEVER hardcode this    │
│               └─ DB username (NOT your OS or app user)          │
│                                                                 │
│  COMMON HOST VARIANTS:                                          │
│  ├─ Same machine:   @localhost:5432                             │
│  ├─ Docker Compose: @db:5432   (service name in the network)    │
│  ├─ Remote server:  @db.prod.company.com:5432                   │
│  └─ Cloud-managed:  @taskmanager.xyz.rds.amazonaws.com:5432     │
│                                                                 │
│  Your code should never see this string directly.               │
│  It must come from an environment variable. (§ 4.5, Week 15)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT HAPPENS WHEN EACH SIDE FAILS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR FASTAPI APP CRASHES:                                      │
│  ├─ PostgreSQL process keeps running, completely unaffected     │
│  ├─ All committed data is intact on disk (Durability — § 4.2)  │
│  ├─ Any in-flight, uncommitted transaction is rolled back       │
│  └─ When app restarts, it reconnects — data is exactly as left  │
│                                                                 │
│  POSTGRESQL SERVER GOES DOWN:                                   │
│  ├─ Your app cannot read or write — connection attempts fail    │
│  ├─ Queries raise exceptions, not silent failures               │
│  ├─ No data is lost for writes that already committed           │
│  └─ Your code must handle this: try/except on DB calls,         │
│     health-check endpoints, graceful error responses            │
│                                                                 │
│  THEY FAIL INDEPENDENTLY. This is not obvious at first.         │
│  Treat "app is down" and "DB is down" as completely separate    │
│  failure modes with separate runbooks in production.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why don't we just reconnect on every request?**

Opening a database connection is not free. PostgreSQL must authenticate the user, allocate memory for the session, and perform a cryptographic handshake — costing 20–100ms. An API handling hundreds of requests per second cannot afford that overhead on every call. The solution is a **connection pool**: a set of pre-opened connections that requests borrow and return. You will build this in Week 6, Lecture 4. For now, simply understand that the architecture implies this cost exists.

```
┌─────────────────────────────────────────────────────────────────┐
│           CONNECTION COST — WHY POOLING EXISTS (PREVIEW)        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NAIVE (never do this in production):                           │
│  Request → Open new connection (20-100ms) → Query → Close       │
│                                                                 │
│  WITH CONNECTION POOL (Week 6, Lecture 4):                      │
│  App starts → Open 10 connections, keep alive                   │
│  Request → Borrow connection (< 1ms) → Query → Return           │
│                                                                 │
│  TECH DEBT → pool_size, max_overflow, asyncpg details           │
│              → Paid off: Week 6, Lecture 4                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏  PRACTICE CHECKPOINT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Scenario: Two services run on the same machine — a FastAPI     │
│  app (port 8000) and a nightly CSV export script.               │
│  Both need to read from the tasks table.                        │
│                                                                 │
│  1. How does each service reach PostgreSQL? What port?          │
│                                                                 │
│  2. The FastAPI app crashes mid-request. The user had just      │
│     submitted a new task that was being inserted. Is the task   │
│     saved in the database? Explain why or why not.              │
│                                                                 │
│  3. The PostgreSQL server goes down. The FastAPI app is still   │
│     running. A user requests GET /tasks. What happens?          │
│                                                                 │
│  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄   │
│  ▼ SOLUTION (attempt first, then check) ▼                       │
│  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄   │
│                                                                 │
│  1. Both connect via TCP/IP to localhost:5432. PostgreSQL        │
│     handles both connections simultaneously. They are           │
│     independent clients — neither knows the other exists.       │
│                                                                 │
│  2. NOT saved. The app crashed before the INSERT executed        │
│     (or before the transaction committed). PostgreSQL detects   │
│     the broken connection and rolls back any uncommitted work.  │
│     Atomicity (§ 4.2): partial writes never persist.            │
│                                                                 │
│  3. The app raises a database connection error. It should        │
│     catch this and return a 503 Service Unavailable response,   │
│     not a silent 200 with empty data. No data is lost — the     │
│     database is temporarily unreachable, not destroyed.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

# PART 2: THE MENTAL MODEL

## 2.1 Tables, Rows, and Columns

**A table is a structured collection of related data.**

```
┌─────────────────────────────────────────────────────────────────┐
│                         TABLE: tasks                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Think of it as one sheet in your spreadsheet.                  │
│  It has a NAME and a fixed set of COLUMNS.                      │
│                                                                 │
│  ┌────┬──────────────────┬──────────┬────────────┬───────────┐  │
│  │ id │ title            │ category │ is_done    │ created_at│  │
│  ├────┼──────────────────┼──────────┼────────────┼───────────┤  │
│  │  1 │ Build login API  │ Backend  │ false      │ 2025-01-15│  │
│  │  2 │ Write unit tests │ Testing  │ false      │ 2025-01-16│  │
│  │  3 │ Fix navbar bug   │ Frontend │ true       │ 2025-01-14│  │
│  └────┴──────────────────┴──────────┴────────────┴───────────┘  │
│                                                                 │
│  COLUMNS (vertical): Define WHAT data is stored.                │
│  ─────────────────── Each column has a name and a type.         │
│                      Like fields in a dataclass.                │
│                                                                 │
│  ROWS (horizontal):  The actual data. Each row is one record.   │
│  ──────────────────  Like one instance of a dataclass.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to what you already know (Week 1 — Dataclasses):**

```python
# What you learned in Week 1, Lecture 2
from dataclasses import dataclass
from datetime import date

@dataclass
class Task:
    id: int
    title: str
    category: str
    is_done: bool
    created_at: date
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  YOUR DATACLASS              │  A DATABASE TABLE                │
│  ────────────────            │  ────────────────                │
│                              │                                  │
│  class Task:                 │  Table "tasks":                  │
│      id: int           ───────▶  id       INTEGER              │
│      title: str        ───────▶  title    TEXT                  │
│      category: str     ───────▶  category TEXT                  │
│      is_done: bool     ───────▶  is_done  BOOLEAN              │
│      created_at: date  ───────▶  created_at DATE               │
│                              │                                  │
│  task = Task(                │  One ROW in the table:           │
│      id=1,             ───────▶  1 │ Build... │ Backend │ ...  │
│      title="Build..."        │                                  │
│      ...                     │                                  │
│  )                           │                                  │
│                              │                                  │
│  The class defines shape.    │  The table defines shape.        │
│  The instance holds data.    │  The row holds data.             │
│                              │                                  │
└─────────────────────────────────────────────────────────────────┘
```

> "If a dataclass defines the SHAPE of your data in Python, a table defines the SHAPE of your data on disk. Same idea, different medium."

**Key vocabulary:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     TERMINOLOGY                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FORMAL TERM     │  INFORMAL      │  SPREADSHEET   │  PYTHON   │
│  ────────────────│────────────────│────────────────│───────────│
│  Table           │  "a table"     │  Sheet/Tab     │  class    │
│  Row / Record    │  "a row"       │  Row           │  instance │
│  Column / Field  │  "a column"    │  Column header │  attribute│
│  Schema          │  "the shape"   │  Headers + fmt │  class def│
│  Value           │  "a cell"      │  Cell content  │  attr val │
│                                                                 │
│  You'll hear all of these interchangeably. They mean the same.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Primary Keys (Every Row Needs an Identity)

**The problem: how do you POINT to a specific row?**

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE IDENTITY PROBLEM                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine a spreadsheet of employees:                            │
│                                                                 │
│  ┌──────────────────┬────────────┬──────────────┐               │
│  │ name             │ department │ email        │               │
│  ├──────────────────┼────────────┼──────────────┤               │
│  │ Alice Chen       │ Backend    │ alice@co.com │               │
│  │ Bob Smith        │ Frontend   │ bob@co.com   │               │
│  │ Alice Chen       │ DevOps     │ achen@co.com │               │
│  └──────────────────┴────────────┴──────────────┘               │
│                                                                 │
│  "Delete Alice Chen's record."                                  │
│                                                                 │
│  WHICH ONE? There are two Alice Chens.                          │
│  Name is not unique. Department is not unique.                  │
│  You need something that UNIQUELY identifies each row.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The solution: a PRIMARY KEY.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     PRIMARY KEY                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A primary key is a column (or set of columns) that             │
│  UNIQUELY IDENTIFIES every row in a table.                      │
│                                                                 │
│  Rules:                                                         │
│  ├─ Must be UNIQUE — no two rows share the same value           │
│  ├─ Must be NOT NULL — every row MUST have a value              │
│  └─ Should be IMMUTABLE — never changes once assigned           │
│                                                                 │
│                                                                 │
│  ┌──────┬──────────────────┬────────────┬──────────────┐        │
│  │ id ★ │ name             │ department │ email        │        │
│  ├──────┼──────────────────┼────────────┼──────────────┤        │
│  │    1 │ Alice Chen       │ Backend    │ alice@co.com │        │
│  │    2 │ Bob Smith        │ Frontend   │ bob@co.com   │        │
│  │    3 │ Alice Chen       │ DevOps     │ achen@co.com │        │
│  └──────┴──────────────────┴────────────┴──────────────┘        │
│       ▲                                                         │
│       └── PRIMARY KEY: Always unique. "Delete row 3" is clear.  │
│                                                                 │
│  ★ = Primary Key column                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Compare to your in-memory Task Manager:**

```python
# You used this in Weeks 3-4:
tasks_db: dict[int, dict] = {}
next_id: int = 1

# That int key? That was your "primary key."
# But nothing stopped you from:
tasks_db[1] = {"title": "First task"}
tasks_db[1] = {"title": "Overwrite!"}  # Silently replaced! No error!

# A database would REJECT a duplicate primary key.
```

**Two common primary key strategies:**

```
┌─────────────────────────────────────────────────────────────────┐
│              PRIMARY KEY STRATEGIES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SERIAL / AUTO-INCREMENT                                        │
│  ────────────────────────                                       │
│  Database assigns: 1, 2, 3, 4, ...                              │
│  ├─ Simple and readable                                         │
│  ├─ Database handles uniqueness automatically                   │
│  ├─ ⚠ Predictable (user can guess id=5 exists)                  │
│  └─ ⚠ Problematic across multiple databases                     │
│                                                                 │
│  UUID                                                           │
│  ────                                                           │
│  Database assigns: "550e8400-e29b-41d4-a716-446655440000"       │
│  ├─ Universally unique — safe across systems                    │
│  ├─ Unpredictable (can't guess next ID)                         │
│  ├─ ⚠ Larger storage (16 bytes vs 4 bytes)                      │
│  └─ ⚠ Not human-readable                                       │
│                                                                 │
│  FOR THIS COURSE: We'll use both and discuss tradeoffs          │
│  in Lecture 4 (Database Design Workshop).                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Foreign Keys (The Bridges Between Tables)

**Now ask the class:**

> "In your Task Manager, tasks have a category — stored as a string. What happens when someone creates a task with category 'Backnd'? Or 'backend' vs 'Backend' vs 'BACKEND'? Who enforces consistency?"

Answer: **Nobody. Your Pydantic model accepts any string. Your dict stores any string. The user's typo becomes permanent data pollution.**

**Foreign keys solve this by CONNECTING tables:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     FOREIGN KEYS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A foreign key is a column in one table that REFERENCES         │
│  the primary key of another table.                              │
│                                                                 │
│  It says: "This value MUST exist in that other table."          │
│                                                                 │
│                                                                 │
│  Table: categories                Table: tasks                  │
│  ┌────┬────────────┐              ┌────┬─────────────┬───────┐  │
│  │ id │ name       │              │ id │ title       │cat_id │  │
│  ├────┼────────────┤              ├────┼─────────────┼───────┤  │
│  │  1 │ Backend    │◀─────────────│  1 │ Build API   │     1 │  │
│  │  2 │ Frontend   │◀────────┐    │  2 │ Write tests │     1 │  │
│  │  3 │ Testing    │         │    │  3 │ Fix navbar  │     2 │  │
│  └────┴────────────┘         │    │  4 │ Add CI/CD   │    99 │  │
│                              │    └────┴─────────────┴───────┘  │
│                              │                          ▲       │
│                              └──────────────────────────┘       │
│                                                   │             │
│                              cat_id = 99? ❌ REJECTED!          │
│                              No category with id 99 exists.     │
│                                                                 │
│  The database REFUSES to store a task pointing to a category    │
│  that doesn't exist. This is REFERENTIAL INTEGRITY.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The spreadsheet version would be like a dropdown validation:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SPREADSHEET:                                                   │
│  A dropdown cell that only allows values from another sheet.    │
│  But... anyone can just type over it. No real enforcement.      │
│                                                                 │
│  DATABASE FOREIGN KEY:                                          │
│  The database ENGINE ITSELF blocks invalid references.          │
│  No code can bypass it. No user can bypass it.                  │
│  It's enforced at the storage layer, not the application layer. │
│                                                                 │
│  THIS IS THE KEY INSIGHT:                                       │
│  ──────────────────────────                                     │
│  Pydantic validates data at the API boundary (input).           │
│  Foreign keys validate data at the STORAGE boundary (persist).  │
│  Both matter. Neither alone is enough.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What happens when you violate a foreign key?**

```
┌─────────────────────────────────────────────────────────────────┐
│                 FOREIGN KEY VIOLATIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO 1: Insert a task with non-existent category           │
│  ──────────                                                     │
│  INSERT task (title="New task", category_id=99)                 │
│  → ERROR: foreign key violation, key (category_id)=(99)         │
│           is not present in table "categories"                  │
│                                                                 │
│  SCENARIO 2: Delete a category that tasks still reference       │
│  ──────────                                                     │
│  DELETE category WHERE id = 1                                   │
│  → ERROR: update or delete on table "categories" violates       │
│           foreign key constraint on table "tasks"               │
│                                                                 │
│  The database protects you from creating ORPHANED DATA:         │
│  tasks pointing to categories that no longer exist.             │
│                                                                 │
│  Compare to your dict:                                          │
│  tasks_db[1] = {"category_id": 99}  ← No error. Silent chaos.  │
│  del categories_db[1]                ← Tasks now point nowhere. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Relationship Types (One-to-Many, Many-to-Many)

**Data relates to other data. There are patterns for how.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    RELATIONSHIP TYPES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ONE-TO-MANY (1:N)                                              │
│  ─────────────────                                              │
│  "One user owns MANY tasks."                                    │
│  "One category contains MANY tasks."                            │
│                                                                 │
│  The MANY side holds the foreign key.                           │
│                                                                 │
│   ┌──────────┐           ┌──────────┐                           │
│   │  users   │ 1      N  │  tasks   │                           │
│   │──────────│───────────│──────────│                           │
│   │ id ★     │           │ id ★     │                           │
│   │ name     │           │ title    │                           │
│   │ email    │           │ user_id ──────▶ users.id             │
│   └──────────┘           └──────────┘                           │
│                                                                 │
│   Alice (id=1) ──── "Build API"     (user_id=1)                │
│                ├─── "Write tests"   (user_id=1)                │
│                └─── "Deploy"        (user_id=1)                │
│   Bob   (id=2) ──── "Fix navbar"   (user_id=2)                │
│                                                                 │
│                                                                 │
│  MANY-TO-MANY (N:N)                                             │
│  ──────────────────                                             │
│  "A task can have MANY tags. A tag can belong to MANY tasks."   │
│                                                                 │
│  Cannot be represented with a single foreign key.               │
│  Requires a JUNCTION TABLE (also called "join table"            │
│  or "association table").                                       │
│                                                                 │
│   ┌──────────┐      ┌────────────┐      ┌──────────┐           │
│   │  tasks   │ 1  N │ task_tags  │ N  1 │  tags    │           │
│   │──────────│──────│────────────│──────│──────────│           │
│   │ id ★     │      │ task_id ─────────▶│ id ★     │           │
│   │ title    │      │ tag_id ──────────▶│ name     │           │
│   └──────────┘      └────────────┘      └──────────┘           │
│                                                                 │
│   "Build API" ──┬── python                                      │
│                 └── fastapi                                     │
│   "Fix navbar" ─┬── javascript                                  │
│                 ├── css                                         │
│                 └── frontend                                    │
│   "Write tests" ── python  (same tag as "Build API"!)           │
│                                                                 │
│                                                                 │
│  ONE-TO-ONE (1:1) — Less common                                 │
│  ────────────────                                               │
│  "One user has ONE profile." (separate table for large/optional │
│   data that you don't always need to load.)                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The junction table unpacked:**

```
┌─────────────────────────────────────────────────────────────────┐
│              JUNCTION TABLE: task_tags                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┬────────┐                                           │
│  │ task_id │ tag_id │                                           │
│  ├─────────┼────────┤                                           │
│  │    1    │    1   │  ← "Build API" has tag "python"           │
│  │    1    │    2   │  ← "Build API" has tag "fastapi"          │
│  │    2    │    3   │  ← "Fix navbar" has tag "javascript"      │
│  │    2    │    4   │  ← "Fix navbar" has tag "css"             │
│  │    3    │    1   │  ← "Write tests" has tag "python"         │
│  └─────────┴────────┘                                           │
│                                                                 │
│  Each row is one "link" between a task and a tag.               │
│  The junction table has NO data of its own —                    │
│  it only exists to connect two other tables.                    │
│                                                                 │
│  Both columns are foreign keys.                                 │
│  Together, they form a COMPOSITE PRIMARY KEY:                   │
│  (task_id=1, tag_id=1) can only appear once.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "In your Task Manager, you stored tags as a list of strings in each task: `tags: list[str]`. What's wrong with that approach compared to a junction table?"

Answer:
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  TAGS AS LIST OF STRINGS:        │  TAGS AS JUNCTION TABLE:     │
│  ─────────────────────────       │  ─────────────────────────   │
│  {"tags": ["python", "api"]}     │  task_tags + tags table      │
│                                  │                              │
│  ❌ "python" vs "Python"?         │  ✅ Tag id=1 is always       │
│     No consistency enforcement   │     "python". One source.    │
│                                  │                              │
│  ❌ "Find all tasks tagged        │  ✅ Simple join query.        │
│     python" = scan every task,   │     Milliseconds.            │
│     check every list             │                              │
│                                  │                              │
│  ❌ Rename "python" to "Python"   │  ✅ Update one row in tags    │
│     = update every task's list   │     table. Done.             │
│                                  │                              │
│  ❌ "How many tasks per tag?"     │  ✅ COUNT + GROUP BY.         │
│     = complex Python code        │     One line of SQL.         │
│                                  │                              │
└─────────────────────────────────────────────────────────────────┘
```

## 2.5 How Separated Data Gets Reunited (Join Concept)

**Now ask the class:**

> "We just spent two sections learning to store tasks and users in separate tables. A task stores `user_id = 1` instead of `'Alice'`. Your API response needs to show the assignee's name. How do you get it back? What operation retrieves 'Alice' when all you have is the number 1?"

The answer is a **JOIN** — the fundamental operation that makes normalization worth doing at all.

```
┌─────────────────────────────────────────────────────────────────┐
│              THE PROBLEM NORMALIZATION CREATES                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE normalization (your Week 3-4 Task Manager):             │
│  ─────────────────────────────────────────────────              │
│  task = {"title": "Build API", "assignee_name": "Alice",        │
│           "assignee_email": "alice@co.com"}                     │
│                                                                 │
│  Alice's email lives in EVERY task she owns. Rename Alice?      │
│  Update 400 rows. Miss one? Inconsistency forever.              │
│                                                                 │
│  AFTER normalization (separate tables, foreign key):            │
│  ─────────────────────────────────────────────────              │
│  tasks:  {id:1, title:"Build API", user_id:1}                   │
│  users:  {id:1, name:"Alice", email:"alice@co.com"}             │
│                                                                 │
│  Alice's email is stored ONCE. Change it in one place.          │
│  But now: how do you get "Build API was assigned to Alice"?     │
│                                                                 │
│  A JOIN: match rows from two tables where a shared key value    │
│  is equal, then COMBINE their columns into one result row.      │
│                                                                 │
│  The foreign key (tasks.user_id → users.id) IS the join         │
│  condition. Foreign keys don't just enforce integrity —         │
│  they are the bridges that JOINs travel across.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualising the row-matching process:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHAT A JOIN DOES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Table: tasks                   Table: users                    │
│  ┌────┬──────────────┬─────────┐  ┌────┬───────┬─────────────┐ │
│  │ id │ title        │ user_id │  │ id │ name  │ email       │ │
│  ├────┼──────────────┼─────────┤  ├────┼───────┼─────────────┤ │
│  │  1 │ Build API    │    1    │  │  1 │ Alice │ alice@co.com│ │
│  │  2 │ Write tests  │    1    │  │  2 │ Bob   │ bob@co.com  │ │
│  │  3 │ Fix navbar   │    2    │  └────┴───────┴─────────────┘ │
│  │  4 │ Orphan task  │  NULL   │                               │
│  └────┴──────────────┴─────────┘                               │
│                                                                 │
│  JOIN CONDITION: tasks.user_id = users.id                       │
│                                                                 │
│  Task 1: user_id=1 → find users.id=1 → Alice → combine rows    │
│  Task 2: user_id=1 → find users.id=1 → Alice → combine rows    │
│  Task 3: user_id=2 → find users.id=2 → Bob   → combine rows    │
│  Task 4: user_id=NULL → no match is possible                    │
│                           ▲                                     │
│                    What happens to Task 4 depends on            │
│                    the TYPE of join. See below.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  INNER JOIN vs LEFT JOIN                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INNER JOIN — "Only rows with a match on BOTH sides."           │
│  ──────────                                                     │
│  ┌──────────────┬───────┬─────────────┐                         │
│  │ title        │ name  │ email       │                         │
│  ├──────────────┼───────┼─────────────┤                         │
│  │ Build API    │ Alice │ alice@co.com│                         │
│  │ Write tests  │ Alice │ alice@co.com│                         │
│  │ Fix navbar   │ Bob   │ bob@co.com  │                         │
│  └──────────────┴───────┴─────────────┘                         │
│  "Orphan task" is DROPPED. user_id=NULL has no match.           │
│  Use when: you only want rows with a valid relationship.        │
│                                                                 │
│                                                                 │
│  LEFT JOIN — "All rows from the LEFT table (tasks), plus        │
│  ─────────    matching columns from the right — or NULL."       │
│  ┌──────────────┬───────┬─────────────┐                         │
│  │ title        │ name  │ email       │                         │
│  ├──────────────┼───────┼─────────────┤                         │
│  │ Build API    │ Alice │ alice@co.com│                         │
│  │ Write tests  │ Alice │ alice@co.com│                         │
│  │ Fix navbar   │ Bob   │ bob@co.com  │                         │
│  │ Orphan task  │ NULL  │ NULL        │                         │
│  └──────────────┴───────┴─────────────┘                         │
│  "Orphan task" is KEPT. user columns are NULL (no match).       │
│  Use when: you want ALL records, even unmatched ones.           │
│  e.g., "Show all tasks, including unassigned ones."             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What you were doing manually in Weeks 3-4:**

```python
# The manual "join" you wrote in your in-memory Task Manager:
def get_tasks_with_users(tasks_db: dict, users_db: dict) -> list[dict]:
    result = []
    for task in tasks_db.values():
        user = users_db.get(task["user_id"])
        if user:  # ← DROP tasks with no user = INNER JOIN behaviour
            result.append({
                "task_title": task["title"],
                "user_name": user["name"],
            })
    return result

# Works for 100 tasks.
# For 1,000,000 tasks: iterates every task, one dict.get() each.
# O(n) — linear scan, no optimisation possible in Python.

# A database JOIN uses an INDEX on users.id (see § 3.5) to find
# each matching row in O(log n). You write one instruction,
# not 10 lines of Python, and it works on millions of rows.

# You will write the actual SQL JOIN syntax in Lecture 2.
# Today: hold the mental model. A JOIN matches rows. That's it.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE CRITICAL INSIGHT                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Normalization and JOINs are two halves of one idea.            │
│                                                                 │
│  NORMALIZATION: Store each fact ONCE, in its own table.         │
│  JOIN:          Retrieve any combination of facts on demand.    │
│                                                                 │
│  You normalize to eliminate duplication and anomalies.          │
│  You JOIN to reconstruct any view of the data you need.         │
│                                                                 │
│  Without JOINs, normalization would just mean fragmented data   │
│  with no way to reassemble it. Without normalization, JOINs     │
│  are trivial — there's nothing worth joining.                   │
│                                                                 │
│  The spreadsheet analogy:                                       │
│  VLOOKUP = a JOIN across two sheets, one value at a time.       │
│  SQL JOIN = VLOOKUP for every row, both directions, optimised.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏  PRACTICE CHECKPOINT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Given these two tables:                                        │
│                                                                 │
│  categories:                  tasks:                            │
│  ┌────┬──────────┐            ┌────┬───────────────┬─────────┐  │
│  │ id │ name     │            │ id │ title         │ cat_id  │  │
│  ├────┼──────────┤            ├────┼───────────────┼─────────┤  │
│  │  1 │ Backend  │            │  1 │ Build API     │    1    │  │
│  │  2 │ Frontend │            │  2 │ Fix navbar    │    2    │  │
│  │  3 │ DevOps   │            │  3 │ Write docs    │  NULL   │  │
│  └────┴──────────┘            └────┴───────────────┴─────────┘  │
│                                                                 │
│  1. Describe in plain English what "joining tasks to            │
│     categories on tasks.cat_id = categories.id" does.           │
│                                                                 │
│  2. INNER JOIN: which tasks appear in the result?               │
│     Which is dropped, and why?                                  │
│                                                                 │
│  3. LEFT JOIN: which tasks appear? What values fill the         │
│     category columns for "Write docs"?                          │
│                                                                 │
│  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄   │
│  ▼ SOLUTION (attempt first, then check) ▼                       │
│  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄   │
│                                                                 │
│  1. For each task, find the category row whose id matches       │
│     that task's cat_id. Combine both rows' columns into a       │
│     single result row. Unmatched rows depend on join type.      │
│                                                                 │
│  2. INNER JOIN: "Build API" (Backend) and "Fix navbar"          │
│     (Frontend). "Write docs" is DROPPED — cat_id=NULL has       │
│     no matching row in categories.                              │
│                                                                 │
│  3. LEFT JOIN: all 3 tasks. "Write docs" appears with NULL      │
│     in the category id and name columns. The task row is kept;  │
│     the right-side columns simply have no value to fill in.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: DATA TYPES

## 3.1 Why Types Matter (Connection to Week 1)

**Connection to what you've learned:**

> "Remember type hints from Week 1, Lecture 1? You learned that `name: str` tells Python (and mypy) what kind of data a variable holds. Database columns work the same way — every column has a type, and the database ENFORCES it. But unlike Python, which lets you ignore type hints at runtime, a database will REJECT data that doesn't match the type. No exceptions."

```
┌─────────────────────────────────────────────────────────────────┐
│                  TYPE HINTS vs COLUMN TYPES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PYTHON TYPE HINTS (Week 1):                                    │
│  ────────────────────────────                                   │
│  age: int = "twenty"  ← Python RUNS this. mypy warns, but      │
│                          the code still executes. It's a hint.  │
│                                                                 │
│  DATABASE COLUMN TYPES:                                         │
│  ──────────────────────                                         │
│  Column "age" INTEGER → INSERT "twenty" into age                │
│  → ERROR: invalid input syntax for type integer: "twenty"       │
│                                                                 │
│  The database REFUSES. Not a hint. A RULE.                      │
│                                                                 │
│  Think of it this way:                                          │
│  ├─ Python type hints = "Please be an int" (advisory)           │
│  ├─ mypy = "This should be an int" (static check)              │
│  ├─ Pydantic = "This MUST be an int at API boundary" (runtime)  │
│  └─ Database type = "This MUST be an int on disk" (storage)     │
│                                                                 │
│  You now have validation at EVERY layer. Defense in depth.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Core PostgreSQL Types (The Everyday Toolkit)

```
┌─────────────────────────────────────────────────────────────────┐
│                 CORE POSTGRESQL TYPES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PostgreSQL Type  │  Python Equivalent  │  Use For              │
│  ─────────────────│─────────────────────│────────────────────── │
│  TEXT             │  str                │  Any string, no limit │
│  VARCHAR(n)       │  str (max length)   │  String with max len  │
│  INTEGER          │  int                │  Whole numbers        │
│  BIGINT           │  int                │  Very large numbers   │
│  SMALLINT         │  int                │  Small range (-32k    │
│                   │                     │    to +32k)           │
│  BOOLEAN          │  bool               │  true / false         │
│  NUMERIC(p, s)    │  Decimal            │  Exact decimals       │
│                   │                     │  (money!)             │
│  REAL / FLOAT     │  float              │  Approx decimals      │
│  DATE             │  date               │  Date only            │
│  TIME             │  time               │  Time only            │
│  TIMESTAMP        │  datetime           │  Date + time          │
│  TIMESTAMPTZ      │  datetime (tz-aware)│  Date + time + zone   │
│  SERIAL           │  (auto int)         │  Auto-increment PK    │
│  BIGSERIAL        │  (auto int)         │  Auto-increment (big) │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The ones that trip people up:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMON TYPE MISTAKES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ MISTAKE: Using FLOAT for money                               │
│                                                                 │
│     0.1 + 0.2 = 0.30000000000000004  (floating point math)     │
│     Your bank account shows $100.30000000000000004?             │
│                                                                 │
│  ✅ CORRECT: Use NUMERIC(10, 2) for money                       │
│     Exact decimal arithmetic. 0.1 + 0.2 = 0.30. Always.        │
│                                                                 │
│  ─────────────────────────────────────────────                  │
│                                                                 │
│  ❌ MISTAKE: Using TEXT for everything                            │
│                                                                 │
│     "Just store it as a string and parse later."                │
│     Now you can't sort numbers: "9" > "10" (string comparison)  │
│     And you can't do date math: "2025-01-15" + 7 = ???          │
│                                                                 │
│  ✅ CORRECT: Use the right type for the data                    │
│     INTEGER for counts, TIMESTAMPTZ for timestamps, etc.        │
│     The database can then sort, compare, and compute FOR you.   │
│                                                                 │
│  ─────────────────────────────────────────────────────          │
│                                                                 │
│  ❌ MISTAKE: Using TIMESTAMP without timezone                    │
│                                                                 │
│     Your server is in UTC. Your user is in Tokyo.               │
│     "2025-01-15 09:00:00" — whose 9 AM? Nobody knows.          │
│                                                                 │
│  ✅ CORRECT: Use TIMESTAMPTZ (timestamp with time zone)         │
│     Stores everything in UTC internally. Converts on display.   │
│     Always use TIMESTAMPTZ. Always. No exceptions.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 PostgreSQL Power Types (UUID, JSONB, Arrays)

**These are PostgreSQL-specific types that make it stand out:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  POSTGRESQL POWER TYPES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UUID                                                           │
│  ────                                                           │
│  "550e8400-e29b-41d4-a716-446655440000"                         │
│                                                                 │
│  128-bit universally unique identifier.                         │
│  Practically impossible to have collisions.                     │
│                                                                 │
│  Python:  import uuid                                           │
│           my_id = uuid.uuid4()                                  │
│                                                                 │
│  Use for: Primary keys (especially in distributed systems),     │
│           public-facing IDs (don't expose auto-increment),      │
│           any ID that must be unique across systems.            │
│                                                                 │
│  Why not always use UUID?                                       │
│  ├─ 16 bytes vs 4 bytes (INTEGER) — larger indexes              │
│  ├─ Random UUIDs fragment B-tree indexes (slower inserts)       │
│  └─ Harder to debug ("row 7" vs "550e8400-e29b...")            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  JSONB (Binary JSON)                                            │
│  ────                                                           │
│  Store a JSON object INSIDE a column.                           │
│                                                                 │
│  ┌────┬──────────┬──────────────────────────────────┐           │
│  │ id │ name     │ metadata (JSONB)                 │           │
│  ├────┼──────────┼──────────────────────────────────┤           │
│  │  1 │ Alice    │ {"role": "admin", "theme": "dark"│           │
│  │    │          │  "prefs": {"lang": "en"}}        │           │
│  │  2 │ Bob      │ {"role": "member"}               │           │
│  └────┴──────────┴──────────────────────────────────┘           │
│                                                                 │
│  Python:  metadata: dict  →  stored as JSONB                    │
│                                                                 │
│  PostgreSQL can QUERY INSIDE the JSON:                          │
│  "Find users where metadata->role = 'admin'"                    │
│  (You'll learn the syntax in Lecture 2)                         │
│                                                                 │
│  Use for: Flexible/optional attributes, external API data,      │
│           settings, metadata that varies between rows.          │
│                                                                 │
│  ⚠ WARNING: JSONB is powerful but seductive.                    │
│  If you put everything in JSONB, you've reinvented MongoDB      │
│  and lost all the benefits of a relational database.            │
│  Rule: If you QUERY a field often, it should be a COLUMN.       │
│  If it's rarely queried metadata, JSONB is fine.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ARRAY                                                          │
│  ─────                                                          │
│  Store a list of values in a single column.                     │
│                                                                 │
│  ┌────┬──────────┬──────────────────────┐                       │
│  │ id │ title    │ labels (TEXT[])       │                       │
│  ├────┼──────────┼──────────────────────┤                       │
│  │  1 │ Bug fix  │ {urgent, production} │                       │
│  │  2 │ Feature  │ {backlog}            │                       │
│  └────┴──────────┴──────────────────────┘                       │
│                                                                 │
│  Python:  labels: list[str]  →  stored as TEXT[]                │
│                                                                 │
│  Use for: Simple lists of scalars, tags (sometimes),            │
│           phone numbers, email aliases.                         │
│                                                                 │
│  ⚠ WARNING: If the items need their own attributes              │
│  (tag has name + color + created_at), use a separate table.     │
│  Arrays are for SIMPLE value lists only.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Choosing the Right Type (Decision Framework)

```
┌─────────────────────────────────────────────────────────────────┐
│               TYPE SELECTION DECISION TREE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "What kind of data am I storing?"                              │
│      │                                                          │
│      ├── Text ────────── Short, limited? ── VARCHAR(n)          │
│      │                   Unlimited? ─────── TEXT                │
│      │                                                          │
│      ├── Number ──────── Whole number? ──── INTEGER / BIGINT    │
│      │                   Money? ─────────── NUMERIC(p, s)       │
│      │                   Scientific? ────── REAL / DOUBLE       │
│      │                                                          │
│      ├── Yes / No ────── BOOLEAN                                │
│      │                                                          │
│      ├── Date / Time ─── Date only? ─────── DATE               │
│      │                   Time only? ─────── TIME               │
│      │                   Both? ──────────── TIMESTAMPTZ ★       │
│      │                                                          │
│      ├── Identifier ──── Internal only? ─── SERIAL / BIGSERIAL │
│      │                   Public-facing? ─── UUID               │
│      │                                                          │
│      ├── Flexible obj ── JSONB (last resort for structured)     │
│      │                                                          │
│      └── List of vals ── Simple scalars? ── ARRAY              │
│                          Complex items? ─── Separate table!     │
│                                                                 │
│  ★ Default to TIMESTAMPTZ for anything involving time.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 3.5 How Databases Find Data: The Index

In Part 1 (§ 1.3), this lecture promised that databases give you "querying power." In § 3.3, it mentioned that random UUIDs fragment B-trees. Both referenced a mechanism that was never explained: **how PostgreSQL actually finds rows efficiently**. Without understanding the index, the "Query Engine" is a black box.

```
┌─────────────────────────────────────────────────────────────────┐
│               THE SEARCH PROBLEM WITHOUT AN INDEX               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  tasks table: 1,000,000 rows                                    │
│                                                                 │
│  Query: "Find all tasks where category_id = 3"                  │
│                                                                 │
│  WITHOUT AN INDEX — PostgreSQL must do a full table scan:       │
│  ─────────────────────────────────────────────────────          │
│  Read row 1 → category_id = 1? No.                              │
│  Read row 2 → category_id = 7? No.                              │
│  Read row 3 → category_id = 3? YES — keep it.                   │
│  Read row 4 → category_id = 2? No.                              │
│  ... 999,996 more rows to check                                 │
│                                                                 │
│  Result: checked ALL 1,000,000 rows to find 50 matches.         │
│  The database had no way to skip the irrelevant rows.           │
│  This is O(n) — it scales with the size of the table.           │
│                                                                 │
│  Your Python dict had the same problem in § 1.2:                │
│  "Give me all tasks in category Backend" = scan every task.     │
│  The database without an index has the exact same weakness.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The book index analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE BOOK INDEX ANALOGY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You want to find all mentions of "transaction" in a            │
│  1,000-page textbook.                                           │
│                                                                 │
│  WITHOUT the index: read every page.                            │
│  WITH the index at the back: look up "transaction" →            │
│    pages 47, 213, 445 → go directly to those pages.            │
│                                                                 │
│  A database index works identically:                            │
│                                                                 │
│  BOOK:                         DATABASE:                        │
│  ──────────────────────────────────────────────────────         │
│  Back-of-book index            Index data structure on disk     │
│  Alphabetically sorted         Sorted by column value           │
│  Word → page numbers           Value → row locations            │
│  Separate from the book text   Separate from the table data     │
│  Must be updated when book     Must be updated on every         │
│  is revised                    INSERT / UPDATE / DELETE         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How the index changes the search:**

```
┌─────────────────────────────────────────────────────────────────┐
│           B-TREE INDEX — HOW POSTGRESQL FINDS ROWS              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A B-tree (Balanced Tree) index on tasks.category_id:           │
│                                                                 │
│  Conceptual structure (simplified):                             │
│                   [5]                                           │
│                  /   \                                          │
│               [2]     [8]                                       │
│              /   \   /   \                                      │
│           [1]   [3] [6]  [9]                                    │
│            ↓     ↓   ↓    ↓                                     │
│          row   row  row  row                                    │
│          ptrs  ptrs ptrs ptrs                                   │
│                                                                 │
│  "Find category_id = 3":                                        │
│  ├─ Compare 3 to root [5]: 3 < 5 → go left                     │
│  ├─ Compare 3 to [2]: 3 > 2 → go right                         │
│  ├─ Find leaf [3] → follow row pointers to matching rows        │
│  └─ 3 comparisons instead of 1,000,000 row scans               │
│                                                                 │
│  This is O(log n). Doubling the table size adds ONE comparison. │
│  At 1 billion rows, you still find results in ~30 comparisons.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Callback to § 3.3 — why UUID B-tree fragmentation matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│              UUID FRAGMENTATION (§ 3.3 REVISITED)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sequential PKs (1, 2, 3, 4, ...):                              │
│  New rows always insert at the RIGHTMOST leaf of the B-tree.   │
│  The tree grows predictably. Minimal reorganisation needed.     │
│  Inserts are fast.                                              │
│                                                                 │
│  Random UUIDs ("550e...", "b3f2...", "1a9c...", ...):           │
│  New rows insert at RANDOM positions throughout the B-tree.    │
│  The tree must constantly split and rebalance pages.            │
│  This is "B-tree fragmentation" — slower inserts over time,     │
│  fragmented disk pages, larger index size.                      │
│                                                                 │
│  Solution: UUID v7 (time-ordered UUIDs that avoid this).        │
│  TECH DEBT → UUID v7 details: Reference card / explore on demand│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What gets indexed automatically, and what you must do yourself:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHAT TO INDEX                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AUTOMATICALLY INDEXED (you do nothing):                        │
│  ────────────────────────────────────────                       │
│  ✅ Primary Key columns — always have an index automatically    │
│                                                                 │
│  YOU MUST INDEX MANUALLY:                                       │
│  ─────────────────────────                                      │
│  ⚠  Foreign Key columns — PostgreSQL does NOT auto-index them   │
│     (unlike MySQL/InnoDB, which does)                           │
│                                                                 │
│     tasks.user_id → users.id:  index tasks.user_id manually    │
│     tasks.category_id → ...: index tasks.category_id manually  │
│                                                                 │
│     Without this, every JOIN on that FK does a full scan.       │
│     With a million tasks and 50 users, that is 1,000,000        │
│     row reads per query, every time. This is a common and       │
│     painful production mistake.                                 │
│                                                                 │
│  ✅ Columns you filter by frequently (WHERE category_id = ?)   │
│  ✅ Columns you ORDER BY frequently                             │
│  ✅ Columns with UNIQUE constraints (auto-indexed)              │
│                                                                 │
│  Everything else: add indexes AFTER measuring performance.      │
│  "Index everything just in case" is not a strategy.             │
│  (EXPLAIN ANALYZE — how to measure — covered in Week 7, L1)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                     INDEX TRADEOFFS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Dramatically faster reads (O(log n) vs O(n))                │
│  ✅ Faster JOINs (indexed FK columns)                           │
│  ✅ Faster ORDER BY and WHERE on indexed columns                │
│                                                                 │
│  ❌ Slower writes — every INSERT, UPDATE, DELETE must also      │
│     update the index structure on disk                          │
│  ❌ Extra storage — the index is a separate structure on disk   │
│  ❌ Wrong indexes can mislead the query optimiser               │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  ├─ Every PRIMARY KEY → indexed automatically                   │
│  ├─ Every FOREIGN KEY → index manually, always                  │
│  └─ Everything else → measure first (Week 7, Lecture 1)         │
│                                                                 │
│  TECH DEBT → B-tree vs GIN vs GiST vs Hash, composite indexes, │
│  partial indexes, EXPLAIN ANALYZE → Week 7, Lecture 1           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏  PRACTICE CHECKPOINT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your tasks table has grown to 2,000,000 rows. The tasks table  │
│  has a foreign key: category_id → categories.id.                │
│  There is NO index on tasks.category_id.                        │
│                                                                 │
│  1. A query fetches all tasks where category_id = 5.            │
│     How many rows does PostgreSQL examine without an index?     │
│                                                                 │
│  2. You add a B-tree index on category_id. The same query       │
│     runs. Describe conceptually what changes in how             │
│     PostgreSQL finds the rows.                                  │
│                                                                 │
│  3. You now insert 10,000 new tasks in bulk. Does the index     │
│     make this INSERT faster, slower, or has no effect?          │
│                                                                 │
│  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄   │
│  ▼ SOLUTION (attempt first, then check) ▼                       │
│  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄   │
│                                                                 │
│  1. All 2,000,000 rows — full table scan. PostgreSQL has no     │
│     shortcut; it must check every row's category_id value.      │
│                                                                 │
│  2. PostgreSQL traverses the B-tree to the leaf node for        │
│     category_id=5 in ~21 comparisons (log₂ 2,000,000 ≈ 21),   │
│     then follows row pointers directly to only the matching     │
│     rows, skipping everything else entirely.                    │
│                                                                 │
│  3. SLOWER. Each of the 10,000 new rows must also update the    │
│     index structure. The B-tree must potentially rebalance.     │
│     Indexes improve reads at the cost of write speed.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: DATA INTEGRITY & ACID

## 4.0 NULL: The Concept of Missing Data

**Ask the class:**

> "What is the difference between a task with `priority = 0`, a task with `priority = ''` (empty string), and a task that has no priority value at all?"

These are three completely different states. Databases have a specific value that represents the third case: **NULL**. And NULL does not behave like anything you have encountered in Python. Getting this wrong is responsible for some of the most persistent, hardest-to-diagnose bugs in backend development.

```
┌─────────────────────────────────────────────────────────────────┐
│              NULL vs ZERO vs EMPTY STRING                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  priority = 0          → "The priority is known: it is zero."  │
│  title = ''            → "The title is known: it is empty."    │
│  due_date = NULL       → "The due date is UNKNOWN or absent."  │
│                                                                 │
│  NULL is NOT:                                                   │
│  ├─ The number zero                                             │
│  ├─ An empty string                                             │
│  ├─ False                                                       │
│  └─ "nothing" in any Python sense                               │
│                                                                 │
│  NULL means: "This value does not exist, or is not known."      │
│                                                                 │
│  Why does this matter?                                          │
│  ├─ A task with priority=0 might be legitimately low-priority   │
│  ├─ A task with priority=NULL has never been assigned one       │
│  └─ Treating them the same produces wrong query results         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│               THREE-VALUED LOGIC: TRUE, FALSE, UNKNOWN          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SQL conditions do not produce just TRUE or FALSE.              │
│  They produce three possible outcomes: TRUE, FALSE, UNKNOWN.    │
│                                                                 │
│  UNKNOWN is what NULL comparisons produce.                      │
│                                                                 │
│  "Is unknown equal to unknown?"   → You don't know. UNKNOWN.   │
│  "Is unknown equal to 5?"         → You don't know. UNKNOWN.   │
│  "Is unknown NOT equal to 5?"     → You don't know. UNKNOWN.   │
│                                                                 │
│  NULL =  NULL   → UNKNOWN  (not TRUE)                          │
│  NULL =  1      → UNKNOWN  (not FALSE)                         │
│  NULL != NULL   → UNKNOWN  (not TRUE)                          │
│  NULL != 1      → UNKNOWN  (not TRUE)                          │
│                                                                 │
│  In a WHERE clause: UNKNOWN means the row is EXCLUDED.          │
│  This is why WHERE col = NULL returns zero rows. Always.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The trap every Python developer falls into:**

```python
# In Python — None comparisons work as expected:
due_date = None
due_date == None    # True  ← Python behaves like you expect
due_date is None    # True  ← preferred Python idiom

# In SQL — NULL comparisons NEVER use =:
# ❌ WRONG — always returns zero rows, no error, no warning:
# WHERE due_date = NULL

# ✅ CORRECT — use IS NULL:
# WHERE due_date IS NULL

# ❌ WRONG — always returns zero rows:
# WHERE due_date != NULL

# ✅ CORRECT — use IS NOT NULL:
# WHERE due_date IS NOT NULL

# The silent failure is the danger:
# WHERE due_date = NULL  → runs without error, returns 0 rows.
# The database is not broken. Your query logic is wrong.
# You will not know unless you check your result count.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  NULL PROPAGATION IN OPERATIONS                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Any operation involving NULL produces NULL.                    │
│                                                                 │
│  Arithmetic:         10 + NULL         = NULL                   │
│  Arithmetic:         10 * NULL         = NULL                   │
│  String concat:      'hello' || NULL   = NULL                   │
│  Function call:      UPPER(NULL)       = NULL                   │
│                                                                 │
│  Practical consequence:                                         │
│  If a task has no priority, summing all priorities for a        │
│  report silently ignores NULL rows. Your totals may be wrong    │
│  without any error.                                             │
│                                                                 │
│  COALESCE(value, default) converts NULL to a fallback:          │
│  COALESCE(priority, 0) → uses 0 when priority is NULL           │
│  (Full aggregate function behaviour → Reference card)           │
│                                                                 │
│  EXCEPTION — logic short-circuits with NULL:                    │
│  TRUE  OR  NULL  = TRUE   (TRUE regardless of unknown side)     │
│  FALSE AND NULL  = FALSE  (FALSE regardless of unknown side)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│             NULL AND CONSTRAINTS — THE CONNECTION               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Now that you understand NULL, the NOT NULL constraint          │
│  (§ 4.1) makes complete sense:                                  │
│                                                                 │
│  title TEXT NOT NULL                                            │
│  → The database REJECTS any INSERT or UPDATE that would         │
│    leave this column without a value. NULL is forbidden here.  │
│                                                                 │
│  due_date DATE   (no NOT NULL)                                  │
│  → NULL is explicitly allowed. This task may have no due date. │
│                                                                 │
│  When designing a column, ask:                                  │
│  "Is the absence of this value EVER valid?"                     │
│  ├─ No (title, created_at) → NOT NULL constraint                │
│  └─ Yes (due_date, assigned_to) → allow NULL intentionally      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏  PRACTICE CHECKPOINT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Spot the two bugs in the following query descriptions.         │
│  For each one, state what the query actually returns, and       │
│  how to fix it.                                                 │
│                                                                 │
│  Bug 1:                                                         │
│  "Find all tasks that have no due date assigned."               │
│  Query used: WHERE due_date = NULL                              │
│                                                                 │
│  Bug 2:                                                         │
│  tasks table, 4 rows:                                           │
│  ┌────┬──────┬──────────┐                                       │
│  │ id │ name │ priority │                                       │
│  ├────┼──────┼──────────┤                                       │
│  │  1 │ T1   │    3     │                                       │
│  │  2 │ T2   │   NULL   │                                       │
│  │  3 │ T3   │    5     │                                       │
│  │  4 │ T4   │    2     │                                       │
│  └────┴──────┼──────────┘                                       │
│  Query: WHERE priority != 3                                     │
│  Expected: T2, T3, T4. How many rows does it actually return?   │
│                                                                 │
│  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄   │
│  ▼ SOLUTION (attempt first, then check) ▼                       │
│  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄   │
│                                                                 │
│  Bug 1: Returns ZERO rows. NULL = NULL → UNKNOWN → row          │
│  excluded. Fix: WHERE due_date IS NULL                          │
│                                                                 │
│  Bug 2: Returns only T3 and T4 — two rows, not three.           │
│  T2 (priority=NULL): NULL != 3 → UNKNOWN → row excluded.        │
│  T2 silently disappears from results. To include it:            │
│  WHERE priority != 3 OR priority IS NULL                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 4.1 Constraints (The Database's Pydantic)

**Connection to what you've learned:**

> "Remember Pydantic from Week 3? You used `Field(min_length=1)` to enforce that a title can't be empty, and `Field(gt=0)` to enforce positive numbers. Database constraints do the same thing — but at the storage layer, not the API layer."

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE CONSTRAINTS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pydantic (Week 3)            │  Database Constraint            │
│  ──────────────────           │  ─────────────────              │
│  title: str                   │  title TEXT NOT NULL             │
│    (required field)           │    (must have a value)           │
│                               │                                 │
│  email: str (unique check     │  email TEXT UNIQUE               │
│    in your code)              │    (DB enforces uniqueness)      │
│                               │                                 │
│  age: int = Field(gt=0)       │  age INTEGER CHECK (age > 0)    │
│                               │                                 │
│  status: Literal["open",     │  status TEXT CHECK               │
│    "closed"]                  │    (status IN ('open','closed')) │
│                               │                                 │
│  category_id: int             │  category_id INTEGER             │
│    (hope it's valid...)       │    REFERENCES categories(id)    │
│                               │    (GUARANTEED valid)            │
│                               │                                 │
│  priority: int = 0            │  priority INTEGER DEFAULT 0     │
│    (Python default)           │    (DB fills in if omitted)      │
│                               │                                 │
└─────────────────────────────────────────────────────────────────┘
```

**All constraints at a glance:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONSTRAINT TYPES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NOT NULL        Column must have a value. No empty cells.      │
│  ─────────       "Every task MUST have a title."                │
│                                                                 │
│  UNIQUE          No two rows can have the same value.           │
│  ──────          "No two users can have the same email."        │
│                                                                 │
│  PRIMARY KEY     UNIQUE + NOT NULL. The row's identity.         │
│  ───────────     "Every row has a unique ID."                   │
│                                                                 │
│  FOREIGN KEY     Must reference an existing row in another      │
│  ───────────     table. "This task's user_id must be a real     │
│                  user."                                         │
│                                                                 │
│  CHECK           Custom condition that must be true.            │
│  ─────           "Priority must be between 1 and 5."           │
│                                                                 │
│  DEFAULT         Auto-fill if no value is provided.             │
│  ───────         "Status defaults to 'open'."                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why both Pydantic AND database constraints?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEFENSE IN DEPTH                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│           Request from User                                     │
│                  │                                              │
│                  ▼                                              │
│         ┌────────────────┐                                      │
│    1.   │  Pydantic      │  Validates shape and types.          │
│         │  (API layer)   │  Fast, user-friendly error messages. │
│         └───────┬────────┘                                      │
│                 │  ✅ Passed                                     │
│                 ▼                                              │
│         ┌────────────────┐                                      │
│    2.   │  Your Code     │  Business logic checks.              │
│         │  (Service)     │  "Does this user have permission?"   │
│         └───────┬────────┘                                      │
│                 │  ✅ Passed                                     │
│                 ▼                                              │
│         ┌────────────────┐                                      │
│    3.   │  Database      │  FINAL guard. Constraints are        │
│         │  (Storage)     │  UNBYPASSABLE. Even if your code     │
│         └────────────────┘  has bugs, the DB catches it.        │
│                                                                 │
│   If a bug in your code skips Pydantic validation,              │
│   the database constraints are the last line of defense.        │
│   Never rely on only one layer.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 ACID Properties (The Four Promises)

**The most important concept in this lecture. This is what separates a real database from a file on disk.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ACID PROPERTIES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A database makes four promises about every operation:          │
│                                                                 │
│  A — ATOMICITY      "All or nothing."                           │
│  C — CONSISTENCY    "Rules are never broken."                   │
│  I — ISOLATION      "Concurrent users don't interfere."         │
│  D — DURABILITY     "Saved means saved. Forever."               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Let's unpack each one with a concrete example.**

**A — ATOMICITY: "All or nothing."**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ATOMICITY                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO: Transfer $200 from Alice to Bob.                     │
│                                                                 │
│  This requires TWO operations:                                  │
│  1. Subtract $200 from Alice's balance                          │
│  2. Add $200 to Bob's balance                                   │
│                                                                 │
│                                                                 │
│  WITHOUT ATOMICITY (your Python dict):                          │
│  ──────────────────────────────────────                         │
│                                                                 │
│  accounts["alice"] -= 200   ← ✅ Success. Alice: $800          │
│  # ⚡ SERVER CRASHES HERE ⚡                                     │
│  accounts["bob"] += 200     ← ❌ Never ran. Bob: $500          │
│                                                                 │
│  Result: $200 VANISHED from the system.                         │
│  Alice lost money. Bob never got it.                            │
│                                                                 │
│                                                                 │
│  WITH ATOMICITY (database transaction):                         │
│  ──────────────────────────────────────                         │
│                                                                 │
│  BEGIN TRANSACTION                                              │
│    Subtract $200 from Alice  ✅                                 │
│    # ⚡ SERVER CRASHES HERE ⚡                                   │
│    Add $200 to Bob           ❌ Never ran                       │
│  → ENTIRE TRANSACTION ROLLED BACK                               │
│                                                                 │
│  Result: Alice: $1000 (unchanged). Bob: $500 (unchanged).      │
│  Both operations succeed, or NEITHER does.                      │
│  No money disappears. Ever.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**C — CONSISTENCY: "Rules are never broken."**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONSISTENCY                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The database moves from one VALID state to another VALID       │
│  state. It never allows an operation that would violate         │
│  your constraints.                                              │
│                                                                 │
│  EXAMPLE:                                                       │
│  Rule: "Account balance must be >= 0" (CHECK constraint)        │
│                                                                 │
│  Alice has $100. She tries to transfer $150.                    │
│                                                                 │
│  WITHOUT CONSISTENCY:                                           │
│  ──────────────────────                                         │
│  accounts["alice"] -= 150   →  Alice: -$50  ← Allowed! 😱     │
│                                                                 │
│  WITH CONSISTENCY:                                              │
│  ─────────────────                                              │
│  UPDATE accounts SET balance = balance - 150 WHERE name='alice' │
│  → ERROR: CHECK constraint "balance_non_negative" violated      │
│  → Operation rejected. Alice still has $100.                    │
│                                                                 │
│  Constraints, foreign keys, types — ALL enforced at all times.  │
│  The database will NEVER be in an invalid state.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**I — ISOLATION: "Concurrent users don't interfere."**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ISOLATION                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Remember the race condition from Part 1?                       │
│  Two requests read next_id=5, both write id 5, one task lost.   │
│                                                                 │
│  Isolation means: each transaction BEHAVES as if it's the       │
│  only one running, even when thousands run simultaneously.      │
│                                                                 │
│                                                                 │
│  WITHOUT ISOLATION (your Python dict):                          │
│  ──────────────────────────────────────                         │
│                                                                 │
│  Time    Request A              Request B                       │
│  ────    ─────────              ─────────                       │
│  T1      Read balance: $1000                                    │
│  T2                             Read balance: $1000             │
│  T3      Withdraw $200                                          │
│  T4      Write: $800                                            │
│  T5                             Withdraw $300                   │
│  T6                             Write: $700   ← WRONG!         │
│                                                                 │
│  Expected: $1000 - $200 - $300 = $500                           │
│  Got: $700 (Request A's withdrawal was silently erased)         │
│                                                                 │
│                                                                 │
│  WITH ISOLATION (database):                                     │
│  ──────────────────────────                                     │
│                                                                 │
│  Request A and B each work in their own "bubble."               │
│  The database ensures the final result is correct: $500.        │
│  (The exact mechanism — locking, MVCC — is Week 7 material.)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**D — DURABILITY: "Saved means saved."**

```
┌─────────────────────────────────────────────────────────────────┐
│                    DURABILITY                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Once the database says "committed," the data SURVIVES:         │
│                                                                 │
│  ├─ Server restart              ✅ Data still there             │
│  ├─ Process crash               ✅ Data still there             │
│  ├─ Power outage                ✅ Data still there             │
│  └─ Your code throwing an      ✅ Data still there             │
│     unhandled exception                                         │
│                                                                 │
│  HOW? The database writes to DISK (not just memory) before      │
│  confirming the commit. It uses a Write-Ahead Log (WAL) to     │
│  ensure recoverability.                                         │
│                                                                 │
│                                                                 │
│  YOUR DICT:                     DATABASE:                       │
│  ──────────                     ─────────                       │
│  tasks_db[1] = {...}            INSERT INTO tasks (...)         │
│  # Stored in RAM                # Written to WAL, then disk     │
│  # Server restarts → GONE       # Server restarts → RECOVERED  │
│                                                                 │
│  This is the FIRST problem we demonstrated in Part 1.           │
│  Durability is what makes databases fundamentally different     │
│  from in-memory storage.                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 What Happens Without ACID (The Horror Story)

**A thought experiment to make it real:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE E-COMMERCE NIGHTMARE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine your Task Manager is actually an e-commerce backend.   │
│  Customer places an order. Your system must:                    │
│                                                                 │
│  1. Create the order record                                     │
│  2. Deduct inventory for each item                              │
│  3. Charge the customer's payment method                        │
│  4. Send confirmation email                                     │
│                                                                 │
│  WITHOUT ACID:                                                  │
│  ─────────────                                                  │
│  Step 1: Order created          ✅                              │
│  Step 2: Inventory deducted     ✅                              │
│  Step 3: Payment fails          ❌ Card declined!               │
│  Step 4: Never reached                                          │
│                                                                 │
│  Result:                                                        │
│  ├─ Order exists (but shouldn't)                                │
│  ├─ Inventory reduced (products "sold" but not paid)            │
│  ├─ Customer not charged (no payment)                           │
│  └─ No email (customer confused)                                │
│                                                                 │
│  Your data is now INCONSISTENT. Inventory is wrong.             │
│  Orders don't match payments. Trust is destroyed.               │
│                                                                 │
│                                                                 │
│  WITH ACID:                                                     │
│  ──────────                                                     │
│  BEGIN TRANSACTION                                              │
│    Step 1: Order created        ✅                              │
│    Step 2: Inventory deducted   ✅                              │
│    Step 3: Payment fails        ❌                              │
│  → ROLLBACK: Steps 1 and 2 are UNDONE automatically            │
│                                                                 │
│  Result:                                                        │
│  ├─ No order exists                                             │
│  ├─ Inventory unchanged                                         │
│  ├─ Customer not charged                                        │
│  └─ Clean state — ready to retry                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This is not an academic exercise. Every real backend you'll work on has operations that MUST succeed or fail as a unit. ACID is not a feature — it's a requirement."

---

## 4.4 Transactions Preview (The Safety Net)

**We'll cover transaction syntax in Lecture 3. For now, understand the concept:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    TRANSACTIONS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A TRANSACTION is a group of operations treated as ONE unit.    │
│                                                                 │
│  BEGIN;                  ← "Start recording"                    │
│      operation 1;                                               │
│      operation 2;                                               │
│      operation 3;                                               │
│  COMMIT;                 ← "Save everything" (all succeed)      │
│                                                                 │
│  or                                                             │
│                                                                 │
│  ROLLBACK;               ← "Undo everything" (all reverted)     │
│                                                                 │
│                                                                 │
│  Connection to Context Managers (Week 1, Lecture 2):            │
│  ──────────────────────────────────────────────────             │
│                                                                 │
│  Remember the `with` statement for resource cleanup?            │
│                                                                 │
│  with open("file.txt") as f:    # Setup                        │
│      data = f.read()            # Work                         │
│  # f is ALWAYS closed           # Cleanup (even on error)      │
│                                                                 │
│  Transactions work the same way in Python code:                 │
│                                                                 │
│  async with db.begin():         # BEGIN transaction             │
│      await db.execute(op1)      # Work                         │
│      await db.execute(op2)      # More work                    │
│  # Auto-COMMIT on success                                      │
│  # Auto-ROLLBACK on exception   # Cleanup (even on error)      │
│                                                                 │
│  The pattern is identical. Context managers were preparing      │
│  you for this exact concept.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 4.5 Database Security & Access Control

We have covered data integrity (correct data), ACID (safe operations), isolation (concurrent safety), and durability (persistence on disk). The final dimension of a trustworthy database is **access control**: who is even permitted to connect, and what are they allowed to do once connected.

```
┌─────────────────────────────────────────────────────────────────┐
│             POSTGRESQL'S OWN USER SYSTEM                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PostgreSQL has its own users and roles — entirely separate     │
│  from your operating system users and your application's        │
│  user accounts (Alice, Bob in your tasks table).                │
│                                                                 │
│  postgres (superuser — installed by default)                    │
│  ├─ Can CREATE and DROP any database                            │
│  ├─ Can CREATE and DROP any user                                │
│  ├─ Can bypass all permission checks                            │
│  ├─ Can read every table in every database                      │
│  └─ ❌ NEVER use this account for your application connection   │
│                                                                 │
│  app_user (restricted — what your FastAPI app should use)       │
│  ├─ Can SELECT, INSERT, UPDATE, DELETE on your tables           │
│  ├─ Cannot CREATE TABLE or DROP TABLE                           │
│  ├─ Cannot access other databases                               │
│  └─ If its credentials leak, damage is contained               │
│                                                                 │
│  readonly_user (reporting — even more restricted)               │
│  ├─ SELECT only — cannot modify any data                        │
│  └─ Useful for analytics tools, dashboards, data exports        │
│                                                                 │
│  PRINCIPLE OF LEAST PRIVILEGE:                                  │
│  Grant only the permissions actually required for the job.      │
│  No more. If a credential is compromised, minimal damage.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Creating a restricted user and granting privileges:**

```sql
-- Run these as the postgres superuser (just once, during setup):

-- 1. Create the restricted application user
CREATE USER app_user WITH PASSWORD 'a_strong_password_here';

-- 2. Allow it to connect to your database
GRANT CONNECT ON DATABASE taskmanager TO app_user;

-- 3. Allow it to use the public schema (where your tables live)
GRANT USAGE ON SCHEMA public TO app_user;

-- 4. Grant only the operations your app actually performs
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- Now app_user CAN do:
--   SELECT * FROM tasks;       ✅
--   INSERT INTO tasks (...);   ✅
--   UPDATE tasks SET ...;      ✅
--   DELETE FROM tasks WHERE ...; ✅

-- But app_user CANNOT do:
--   DROP TABLE tasks;          ❌ Permission denied
--   CREATE TABLE hacked;       ❌ Permission denied
--   SELECT * FROM pg_shadow;   ❌ Permission denied (system secrets)
--   Connect to other databases ❌ Permission denied
```

```
┌─────────────────────────────────────────────────────────────────┐
│           TWO COMPLETELY SEPARATE USER SYSTEMS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  POSTGRESQL USERS (database-level):                             │
│  ─────────────────────────────────                              │
│  Who can CONNECT to the PostgreSQL server process?              │
│  What SQL operations can they run once connected?               │
│  Examples: postgres, app_user, readonly_user                    │
│  Defined with: CREATE USER, GRANT, REVOKE                       │
│                                                                 │
│  YOUR APPLICATION USERS (row-level, in your users table):       │
│  ────────────────────────────────────────────────────           │
│  Alice, Bob — stored as rows in the users table                 │
│  Controlled by your application logic (JWT, RBAC)               │
│  PostgreSQL does not know them as DB users at all               │
│                                                                 │
│  HOW THEY RELATE:                                               │
│  Your FastAPI app connects as "app_user" (DB-level).            │
│  Inside that connection, your SQL filters by user_id to         │
│  enforce Alice only seeing Alice's tasks.                       │
│                                                                 │
│  app_user: "I am allowed to SELECT from tasks."                 │
│  Your code: "...WHERE user_id = alice.id"                       │
│  Together: Alice sees only her tasks.                           │
│                                                                 │
│  They operate at completely different layers.                   │
│  Do not conflate them.                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection string security — the credentials are a liability:**

```python
# ❌ NEVER hardcode credentials in your source code:
DATABASE_URL = "postgresql://app_user:my_password@localhost:5432/taskmanager"
# → This gets committed to git. It lives in git history forever.
# → Every developer with repo access sees production credentials.

# ❌ NEVER commit a .env file containing real credentials to git.

# ✅ Load from environment variables at runtime:
import os
DATABASE_URL = os.environ["DATABASE_URL"]
# If the variable is not set, raises KeyError immediately.
# Explicit failure is better than silent use of a bad default.

# ✅ For validated config (covered in Week 15, Lecture 2):
# from pydantic_settings import BaseSettings
# class Settings(BaseSettings):
#     database_url: str           ← validated, type-checked
#     class Config:
#         env_file = ".env"       ← loads .env in dev only
#
# TECH DEBT → pydantic-settings full config pattern: Week 15, L2
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   ✏  PRACTICE CHECKPOINT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your colleague says: "I'll just connect the app as postgres    │
│  for now — it's easier than setting up a separate user, and     │
│  we can lock it down later before going to production."         │
│                                                                 │
│  1. Name two specific risks of connecting the application as    │
│     the postgres superuser.                                     │
│                                                                 │
│  2. A new endpoint accidentally has a bug that executes         │
│     user-supplied SQL (an injection vulnerability).             │
│     Compare the damage if the app connects as:                  │
│     (a) postgres superuser  vs  (b) app_user with only          │
│         SELECT/INSERT/UPDATE/DELETE on the tasks table.         │
│                                                                 │
│  3. You add a new table (comments) after the initial GRANT.     │
│     Does app_user automatically have access to it?              │
│                                                                 │
│  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄   │
│  ▼ SOLUTION (attempt first, then check) ▼                       │
│  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄   │
│                                                                 │
│  1. (a) If credentials leak, the attacker has superuser access: │
│     can drop all tables, create backdoor accounts, read every   │
│     database on the server. (b) "Lock it down later" never      │
│     happens — the habit of committing superuser credentials     │
│     to .env files persists into production.                     │
│                                                                 │
│  2. (a) postgres: attacker can DROP TABLE users, CREATE USER,   │
│     exfiltrate all data, destroy all databases on the server.   │
│     Total loss. (b) app_user: attacker can read/write tasks     │
│     table only. Cannot drop tables, cannot access other DBs.    │
│     Serious — but the blast radius is contained.                │
│                                                                 │
│  3. No. The GRANT covered tables that existed at grant time.    │
│     You must re-run GRANT for the new table, or use             │
│     ALTER DEFAULT PRIVILEGES to auto-grant on future tables.   │
│     (TECH DEBT → Reference card / explore on demand)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: NORMALIZATION

## 5.1 The Copy-Paste Problem

**Start with messy data that students recognize:**

```
┌─────────────────────────────────────────────────────────────────┐
│           THE TASK MANAGER — UNNORMALIZED                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine storing EVERYTHING in one table:                       │
│                                                                 │
│  ┌────┬────────────┬──────────┬──────────────┬────────────────┐ │
│  │ id │ title      │ assignee │ assign_email │ tags           │ │
│  ├────┼────────────┼──────────┼──────────────┼────────────────┤ │
│  │  1 │ Build API  │ Alice    │ alice@co.com │ python,fastapi │ │
│  │  2 │ Write tests│ Alice    │ alice@co.com │ python,pytest  │ │
│  │  3 │ Fix navbar │ Bob      │ bob@co.com   │ react,css      │ │
│  │  4 │ Add CI/CD  │ Alice    │ alice@co.com │ devops,docker  │ │
│  │  5 │ Code review│ Bob      │ bob@co.com   │ python         │ │
│  └────┴────────────┴──────────┴──────────────┴────────────────┘ │
│                                                                 │
│  Looks fine, right? It's simple. One table. Easy to understand. │
│  But it's hiding THREE problems that will DESTROY your data.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Problem 1: UPDATE ANOMALY**

```
┌─────────────────────────────────────────────────────────────────┐
│                  UPDATE ANOMALY                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alice changes her email to alice.chen@company.com.             │
│                                                                 │
│  You must update EVERY ROW where Alice appears:                 │
│  ├─ Row 1: alice@co.com → alice.chen@company.com                │
│  ├─ Row 2: alice@co.com → alice.chen@company.com                │
│  └─ Row 4: alice@co.com → alice.chen@company.com                │
│                                                                 │
│  Miss one? Now Alice has TWO different emails in your database. │
│                                                                 │
│  With 10,000 tasks, how confident are you that you              │
│  updated them ALL? This is not a theoretical problem.           │
│  It happens in production. Constantly.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Problem 2: INSERT ANOMALY**

```
┌─────────────────────────────────────────────────────────────────┐
│                  INSERT ANOMALY                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  New employee Carol joins the company. She hasn't been          │
│  assigned any tasks yet.                                        │
│                                                                 │
│  How do you record Carol's existence?                           │
│                                                                 │
│  ┌────┬────────────┬──────────┬──────────────────┬────┐         │
│  │  ? │ ???        │ Carol    │ carol@co.com     │ ???│         │
│  └────┴────────────┴──────────┴──────────────────┴────┘         │
│                                                                 │
│  There's no task to assign her to. But the table requires       │
│  a task title (it's the whole point of the table).              │
│                                                                 │
│  You CAN'T store a user without a task.                         │
│  Users and tasks are MIXED in one table.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Problem 3: DELETE ANOMALY**

```
┌─────────────────────────────────────────────────────────────────┐
│                  DELETE ANOMALY                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Bob's only task ("Fix navbar") is completed and removed.       │
│                                                                 │
│  DELETE row 3.                                                  │
│                                                                 │
│  But row 3 was the ONLY place Bob's name and email were stored. │
│  By deleting the task, you've ALSO deleted Bob's information.   │
│                                                                 │
│  Bob still works at the company. But your system just forgot    │
│  he exists.                                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "All three problems share one root cause: **different concepts (users and tasks) are smashed into one table.** Normalization is the process of pulling them apart so each concept lives in its own table."

---

## 5.2 First Normal Form (1NF): One Value Per Cell

**Rule: Every cell contains ONE atomic (indivisible) value.**

```
┌─────────────────────────────────────────────────────────────────┐
│              FIRST NORMAL FORM (1NF)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ VIOLATES 1NF:                                               │
│                                                                 │
│  ┌────┬────────────┬────────────────┐                           │
│  │ id │ title      │ tags           │                           │
│  ├────┼────────────┼────────────────┤                           │
│  │  1 │ Build API  │ python,fastapi │  ← TWO values in one cell│
│  │  2 │ Fix navbar │ react,css      │  ← TWO values in one cell│
│  └────┴────────────┴────────────────┘                           │
│                                                                 │
│  Problems:                                                      │
│  ├─ "Find all tasks tagged 'python'" → string parsing needed    │
│  ├─ "How many tasks have 'css' tag?" → can't COUNT properly     │
│  └─ "Remove 'fastapi' tag from task 1" → string manipulation   │
│                                                                 │
│                                                                 │
│  ✅ SATISFIES 1NF (Option A — separate rows):                   │
│                                                                 │
│  ┌────┬────────────┬────────┐                                   │
│  │ id │ title      │ tag    │                                   │
│  ├────┼────────────┼────────┤                                   │
│  │  1 │ Build API  │ python │  ← ONE value per cell             │
│  │  1 │ Build API  │ fastapi│  ← ONE value per cell             │
│  │  2 │ Fix navbar │ react  │                                   │
│  │  2 │ Fix navbar │ css    │                                   │
│  └────┴────────────┴────────┘                                   │
│                                                                 │
│  Better — but now "Build API" is duplicated.                    │
│  We'll fix this in 2NF.                                         │
│                                                                 │
│                                                                 │
│  ✅ SATISFIES 1NF (Option B — separate table, the right way):   │
│                                                                 │
│  tasks:                      task_tags:     tags:               │
│  ┌────┬────────────┐         ┌───────┬──┐   ┌────┬────────┐    │
│  │ id │ title      │         │task_id│tag│   │ id │ name   │    │
│  ├────┼────────────┤         │  id   │   │   ├────┼────────┤    │
│  │  1 │ Build API  │         ├───────┼──┤   │  1 │ python │    │
│  │  2 │ Fix navbar │         │  1    │ 1│   │  2 │ fastapi│    │
│  └────┴────────────┘         │  1    │ 2│   │  3 │ react  │    │
│                              │  2    │ 3│   │  4 │ css    │    │
│                              │  2    │ 4│   └────┴────────┘    │
│                              └───────┴──┘                       │
│                                                                 │
│  Each cell: one value. Each table: one concept.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The Python parallel:**

```python
# ❌ 1NF violation — multiple values crammed into one field
@dataclass
class Task:
    id: int
    title: str
    tags: str  # "python,fastapi" — ugh

# ✅ 1NF satisfied — proper type modeling  
@dataclass
class Task:
    id: int
    title: str

@dataclass
class Tag:
    id: int
    name: str

@dataclass
class TaskTag:
    task_id: int
    tag_id: int
```

> "If you wouldn't store a comma-separated string in a well-typed Python dataclass, don't store it in a database column either."

---

## 5.3 Second Normal Form (2NF): No Partial Dependencies

**Rule: Every non-key column must depend on the ENTIRE primary key.**

```
┌─────────────────────────────────────────────────────────────────┐
│              SECOND NORMAL FORM (2NF)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Prerequisite: Table must already be in 1NF.                    │
│  Applies when: Primary key is COMPOSITE (multiple columns).     │
│                                                                 │
│                                                                 │
│  ❌ VIOLATES 2NF:                                               │
│                                                                 │
│  Table: enrollment                                              │
│  Primary Key: (student_id, course_id) — composite               │
│                                                                 │
│  ┌────────────┬───────────┬──────────────┬───────┐              │
│  │ student_id │ course_id │ student_name │ grade │              │
│  ├────────────┼───────────┼──────────────┼───────┤              │
│  │      1     │   101     │ Alice        │   A   │              │
│  │      1     │   102     │ Alice        │   B   │              │
│  │      2     │   101     │ Bob          │   C   │              │
│  └────────────┴───────────┴──────────────┴───────┘              │
│                                                                 │
│  "student_name" depends ONLY on student_id, not on course_id.   │
│  It's a PARTIAL dependency — it doesn't need the full key.      │
│                                                                 │
│  Problem: Alice's name is stored twice. Change it in one row    │
│  but not the other → inconsistency.                             │
│                                                                 │
│                                                                 │
│  ✅ SATISFIES 2NF: Split into separate tables.                  │
│                                                                 │
│  students:                  enrollments:                        │
│  ┌────┬──────┐              ┌────────────┬───────────┬───────┐  │
│  │ id │ name │              │ student_id │ course_id │ grade │  │
│  ├────┼──────┤              ├────────────┼───────────┼───────┤  │
│  │  1 │ Alice│              │      1     │   101     │   A   │  │
│  │  2 │ Bob  │              │      1     │   102     │   B   │  │
│  └────┴──────┘              │      2     │   101     │   C   │  │
│                             └────────────┴───────────┴───────┘  │
│                                                                 │
│  Now student_name lives in ONE place. Always consistent.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "2NF says: if a column only needs PART of the key to determine its value, it belongs in a separate table with that part of the key."

---

## 5.4 Third Normal Form (3NF): No Transitive Dependencies

**Rule: Non-key columns must depend ONLY on the primary key, not on other non-key columns.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THIRD NORMAL FORM (3NF)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Prerequisite: Table must already be in 2NF.                    │
│                                                                 │
│                                                                 │
│  ❌ VIOLATES 3NF:                                               │
│                                                                 │
│  Table: tasks                                                   │
│  ┌────┬────────────┬──────────────┬────────────────┐            │
│  │ id │ title      │ assignee_id  │ assignee_email │            │
│  ├────┼────────────┼──────────────┼────────────────┤            │
│  │  1 │ Build API  │      1       │ alice@co.com   │            │
│  │  2 │ Write tests│      1       │ alice@co.com   │            │
│  │  3 │ Fix navbar │      2       │ bob@co.com     │            │
│  └────┴────────────┴──────────────┴────────────────┘            │
│                                                                 │
│  assignee_email depends on assignee_id, not on the task's id.   │
│                                                                 │
│  Dependency chain:                                              │
│  task_id → assignee_id → assignee_email                         │
│            ▲                                                    │
│            └── TRANSITIVE: email depends on assignee_id,        │
│                which depends on task_id. Indirect dependency.   │
│                                                                 │
│  Problem: Alice's email stored in multiple rows. Again.         │
│                                                                 │
│                                                                 │
│  ✅ SATISFIES 3NF: Separate the transitive dependency.          │
│                                                                 │
│  tasks:                        users:                           │
│  ┌────┬────────────┬────────┐  ┌────┬───────┬──────────────┐   │
│  │ id │ title      │user_id │  │ id │ name  │ email        │   │
│  ├────┼────────────┼────────┤  ├────┼───────┼──────────────┤   │
│  │  1 │ Build API  │   1    │  │  1 │ Alice │ alice@co.com │   │
│  │  2 │ Write tests│   1    │  │  2 │ Bob   │ bob@co.com   │   │
│  │  3 │ Fix navbar │   2    │  └────┴───────┴──────────────┘   │
│  └────┴────────────┴────────┘                                   │
│                                                                 │
│  Alice's email is stored ONCE. Change it in one place.          │
│  This is the DRY principle (Don't Repeat Yourself)              │
│  applied to data.                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A simple way to remember:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1NF: "One value per cell."                                     │
│       No comma-separated lists. No arrays in a cell.            │
│                                                                 │
│  2NF: "Every column depends on the WHOLE key."                  │
│       If it only needs part of the key, split it out.           │
│                                                                 │
│  3NF: "Every column depends on NOTHING BUT the key."            │
│       If column A determines column B, B belongs elsewhere.     │
│                                                                 │
│                                                                 │
│  The classic mnemonic:                                          │
│  ┌─────────────────────────────────────────────────────┐        │
│  │                                                     │        │
│  │  "Each column depends on the key (1NF),             │        │
│  │   the WHOLE key (2NF),                              │        │
│  │   and NOTHING BUT the key (3NF)."                   │        │
│  │                                                     │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Denormalization (When to Break the Rules)

**Now ask the class:**

> "If normalization is so great, why not normalize EVERYTHING? Why not have a separate table for every single attribute?"

Answer: **Because joins have a cost.**

```
┌─────────────────────────────────────────────────────────────────┐
│              DENORMALIZATION TRADEOFFS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NORMALIZED (many tables, no duplication):                      │
│  ─────────────────────────────────────────                      │
│  ✅ No data duplication                                         │
│  ✅ Easy to update (change in one place)                        │
│  ✅ Small storage footprint                                     │
│  ❌ Complex queries (many JOINs)                                │
│  ❌ Slower reads (database must combine tables)                 │
│                                                                 │
│  DENORMALIZED (fewer tables, some duplication):                 │
│  ─────────────────────────────────────────────                  │
│  ✅ Simpler queries (fewer JOINs)                               │
│  ✅ Faster reads (data already combined)                        │
│  ❌ Data duplication (more storage)                              │
│  ❌ Update anomalies (must update in multiple places)            │
│  ❌ Risk of inconsistency                                       │
│                                                                 │
│                                                                 │
│  THE RULE OF THUMB:                                             │
│  ──────────────────                                             │
│  Start normalized (3NF). Only denormalize when you have         │
│  MEASURED a performance problem. Premature denormalization      │
│  is premature optimization — same mistake, different domain.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When denormalization is justified:**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHEN TO DENORMALIZE                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ READ-HEAVY workload (dashboards, reports, analytics)        │
│     If you read 1000x more than you write, precompute the join. │
│                                                                 │
│  ✅ Caching materialized data (storing a computed summary)       │
│     E.g., "task_count" column on the user table.                │
│     Updated on each task insert/delete.                         │
│                                                                 │
│  ✅ Archival / logging data (write-once, never updated)          │
│     Event logs often store denormalized snapshots because       │
│     the referenced data might change later.                     │
│                                                                 │
│  ❌ DON'T denormalize "just to avoid writing JOINs."            │
│     That's laziness, not engineering.                           │
│                                                                 │
│  ❌ DON'T denormalize before you have EVIDENCE of slow queries.  │
│     Profile first (EXPLAIN ANALYZE — Week 7).                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 6: ENTITY-RELATIONSHIP DIAGRAMS

## 6.1 Design Before Code (Blueprints, Not Guesswork)

```
┌─────────────────────────────────────────────────────────────────┐
│                 DESIGN BEFORE CODE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You wouldn't build a house by nailing boards together          │
│  and hoping it turns into a kitchen. You draw a blueprint.      │
│                                                                 │
│  The same applies to databases.                                 │
│                                                                 │
│  WRONG ORDER:                                                   │
│  ─────────────                                                  │
│  1. Start writing CREATE TABLE                                  │
│  2. Realize you need another table                              │
│  3. Add foreign keys haphazardly                                │
│  4. Realize the structure is wrong                              │
│  5. Rewrite everything (and migrate existing data)              │
│                                                                 │
│  RIGHT ORDER:                                                   │
│  ────────────                                                   │
│  1. Identify ENTITIES (what "things" exist?)                    │
│  2. Identify ATTRIBUTES (what describes each thing?)            │
│  3. Identify RELATIONSHIPS (how do things connect?)             │
│  4. Draw an ER diagram (visualize the whole structure)          │
│  5. THEN write CREATE TABLE statements                          │
│                                                                 │
│  Step 5 becomes trivial once Steps 1-4 are done right.         │
│                                                                 │
│  Connection to Week 3 (API Design):                             │
│  ──────────────────────────────────                             │
│  Remember "API contract design (OpenAPI spec before             │
│  implementation)"? Same principle.                              │
│  Design the structure before writing the code.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 ER Notation (The Vocabulary)

```
┌─────────────────────────────────────────────────────────────────┐
│                 ER DIAGRAM NOTATION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ENTITY (a table):                                              │
│  ┌──────────────┐                                               │
│  │  entity_name │                                               │
│  ├──────────────┤                                               │
│  │ ★ id         │  ★ = Primary Key                              │
│  │   attribute1 │                                               │
│  │   attribute2 │                                               │
│  └──────────────┘                                               │
│                                                                 │
│                                                                 │
│  RELATIONSHIPS:                                                 │
│                                                                 │
│  One-to-Many (1:N):                                             │
│  ┌────────┐         ┌────────┐                                  │
│  │ users  │ 1     N │ tasks  │                                  │
│  │────────│────────▶│────────│                                  │
│  │ ★ id   │         │ ★ id   │                                  │
│  │  name  │         │  title │                                  │
│  └────────┘         │  user_id (FK)                             │
│                     └────────┘                                  │
│  Read: "One user has many tasks."                               │
│  The FK goes on the "many" side.                                │
│                                                                 │
│                                                                 │
│  Many-to-Many (N:N):                                            │
│  ┌────────┐     ┌────────────┐     ┌────────┐                   │
│  │ tasks  │ 1 N │ task_tags  │ N 1 │ tags   │                   │
│  │────────│────▶│────────────│◀────│────────│                   │
│  │ ★ id   │     │ ★ task_id (FK)   │ ★ id   │                   │
│  │  title │     │ ★ tag_id  (FK)   │  name  │                   │
│  └────────┘     └────────────┘     └────────┘                   │
│  Read: "Many tasks have many tags."                             │
│  The junction table holds both FKs.                             │
│                                                                 │
│                                                                 │
│  One-to-One (1:1):                                              │
│  ┌────────┐         ┌──────────┐                                │
│  │ users  │ 1     1 │ profiles │                                │
│  │────────│────────▶│──────────│                                │
│  │ ★ id   │         │ ★ id     │                                │
│  │  name  │         │  bio     │                                │
│  └────────┘         │  avatar  │                                │
│                     │  user_id (FK, UNIQUE)                     │
│                     └──────────┘                                │
│  Read: "One user has one profile."                              │
│  The FK has a UNIQUE constraint.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 Your Task Manager as an ER Diagram

**Let's redesign the Task Manager from Weeks 3-4 as a proper relational schema:**

```
┌─────────────────────────────────────────────────────────────────┐
│           TASK MANAGER — ER DIAGRAM                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ ┌──────────────┐       ┌──────────────────┐                     │
│ │    users     │       │      tasks       │                     │
│ ├──────────────┤ 1   N ├──────────────────┤                     │
│ │ ★ id (UUID)  │──────▶│ ★ id (UUID)      │                     │
│ │   name       │       │   title          │                     │
│ │   email (UQ) │       │   description    │                     │
│ │   created_at │       │   is_done        │                     │
│ └──────────────┘       │   priority       │                     │
│                        │   created_at     │                     │
│                        │   updated_at     │                     │
│                        │   user_id (FK) ──────▶ users.id        │
│                        │   category_id (FK) ──▶ categories.id   │
│                        └──────────────────┘                     │
│                               │                                 │
│ ┌──────────────┐              │ N                                │
│ │  categories  │              │                                  │
│ ├──────────────┤ 1            │                                  │
│ │ ★ id         │◀─────────────┘                                 │
│ │   name (UQ)  │                                                │
│ │   color      │       ┌──────────────┐      ┌──────────────┐   │
│ └──────────────┘       │  task_tags   │      │    tags      │   │
│                        ├──────────────┤      ├──────────────┤   │
│                      N │ ★ task_id(FK)│──────│ ★ id         │ 1 │
│           tasks ──────▶│ ★ tag_id(FK) │──────│   name (UQ)  │◀──│
│                        └──────────────┘      └──────────────┘   │
│                                                                 │
│                                                                 │
│  ENTITIES:                                                      │
│  ├─ users — The people using the system                         │
│  ├─ tasks — The work items                                      │
│  ├─ categories — Classification buckets (Backend, Frontend...)  │
│  ├─ tags — Flexible labels (python, urgent, bug...)             │
│  └─ task_tags — Junction table for many-to-many                 │
│                                                                 │
│  RELATIONSHIPS:                                                 │
│  ├─ users → tasks: One-to-Many (one user owns many tasks)       │
│  ├─ categories → tasks: One-to-Many (one category, many tasks)  │
│  └─ tasks ↔ tags: Many-to-Many (via task_tags junction)         │
│                                                                 │
│                                                                 │
│  Compare this to your Weeks 3-4 Task Manager:                   │
│                                                                 │
│  BEFORE:                    AFTER:                              │
│  One Python dict.           Five related tables.                │
│  No relationships.          Clear, enforced relationships.      │
│  No integrity.              Constraints on every column.        │
│  Data in RAM.               Data on disk, ACID-protected.       │
│                                                                 │
│  This is what you'll BUILD in the rest of this phase.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes and Misconceptions

### Mistake 1: "I'll just store everything in one big table"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ THE "GOD TABLE" ANTI-PATTERN                                │
│                                                                 │
│  ┌────┬───────┬───────┬──────────┬──────┬────────┬─────┬────┐  │
│  │ id │ title │ descr │ user_name│ email│ cat    │ tags│ ...│  │
│  └────┴───────┴───────┴──────────┴──────┴────────┴─────┴────┘  │
│                                                                 │
│  "Fewer tables = simpler, right?"                               │
│                                                                 │
│  Wrong. You get update anomalies, insert anomalies, delete      │
│  anomalies, redundant data, and queries that scan everything.   │
│  "Simpler" now means "nightmare" in two months.                 │
│                                                                 │
│  ✅ One table per CONCEPT. Users are not tasks. Tags are not    │
│     categories. Separate them.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 2: "JSONB can hold everything, why use columns?"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ THE "EVERYTHING IS JSON" TRAP                               │
│                                                                 │
│  ┌────┬───────────────────────────────────────────┐             │
│  │ id │ data (JSONB)                              │             │
│  ├────┼───────────────────────────────────────────┤             │
│  │  1 │ {"title": "Build API", "user": "Alice",  │             │
│  │    │  "category": "Backend", "done": false}    │             │
│  └────┴───────────────────────────────────────────┘             │
│                                                                 │
│  "PostgreSQL has JSONB! It's like MongoDB but relational!"      │
│                                                                 │
│  You've just thrown away:                                       │
│  ├─ Column-level type enforcement (no type safety)              │
│  ├─ NOT NULL constraints (any field can be missing)             │
│  ├─ Foreign key relationships (no referential integrity)        │
│  ├─ Efficient indexing on individual fields                     │
│  └─ Every benefit of a relational database                      │
│                                                                 │
│  ✅ JSONB is for genuinely flexible, rarely-queried metadata.   │
│     If you query a field regularly, it MUST be a column.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 3: "I don't need foreign keys, I'll check in my application code"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ "APPLICATION-ENFORCED" INTEGRITY                            │
│                                                                 │
│  # "I'll just check in Python"                                  │
│  async def create_task(task: TaskCreate):                       │
│      user = await get_user(task.user_id)                        │
│      if not user:                                               │
│          raise HTTPException(404, "User not found")             │
│      # ... create task                                          │
│                                                                 │
│  Problems:                                                      │
│  ├─ Race condition: user deleted between check and insert       │
│  ├─ Bypass: another service writes directly to DB, skipping     │
│  │  your Python check entirely                                  │
│  ├─ Bugs: one developer forgets the check in a new endpoint     │
│  └─ Manual data fixes: someone runs SQL directly, no check      │
│                                                                 │
│  ✅ Foreign keys are enforced by the DATABASE ENGINE itself.     │
│     Nothing can bypass them. Not buggy code, not direct SQL,    │
│     not another service. They are the final line of defense.    │
│                                                                 │
│     (You ALSO check in application code for nice error          │
│      messages. Defense in depth. Same as Pydantic + DB types.)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 4: "I'll normalize later when it matters"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ "NORMALIZE LATER"                                           │
│                                                                 │
│  Once your unnormalized table has 100,000 rows:                 │
│  ├─ Splitting it into multiple tables requires migration        │
│  ├─ Migration requires downtime or complex logic                │
│  ├─ Every query in your application must be rewritten           │
│  ├─ Your test suite breaks everywhere                           │
│  └─ The corrupted data from months of anomalies must be cleaned │
│                                                                 │
│  ✅ Normalize from the START. It's 10x easier to denormalize    │
│     a well-structured database than to normalize a messy one.   │
│                                                                 │
│     Start at 3NF. Denormalize only when you measure a           │
│     performance problem. Never the other way around.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 5: "Using TIMESTAMP instead of TIMESTAMPTZ"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ TIMESTAMP WITHOUT TIMEZONE                                  │
│                                                                 │
│  "2025-06-15 09:00:00" — Is this UTC? New York? Tokyo?          │
│  Nobody knows. Your London user sees "9 AM" and your Tokyo      │
│  user also sees "9 AM" — but they're 8 hours apart.             │
│                                                                 │
│  ✅ ALWAYS use TIMESTAMPTZ.                                     │
│     PostgreSQL stores it as UTC internally.                     │
│     Converts to the client's timezone on display.               │
│     No ambiguity. No bugs when your users span time zones.      │
│                                                                 │
│     This is the ONLY correct choice. Not a preference.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 6: "I'll check for NULL with `= NULL`"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ NULL COMPARISON WITH =                                      │
│                                                                 │
│  # Python developer writes this intuitively:                    │
│  WHERE due_date = NULL     ← returns ZERO rows. Always.         │
│  WHERE due_date != NULL    ← returns ZERO rows. Always.         │
│                                                                 │
│  NULL = anything → UNKNOWN → row excluded from WHERE results.   │
│  No error is raised. The query runs silently and returns        │
│  nothing. This is one of the most common SQL bugs.              │
│                                                                 │
│  ✅ CORRECT: Use IS NULL and IS NOT NULL.                       │
│  WHERE due_date IS NULL      ← rows where due_date is absent    │
│  WHERE due_date IS NOT NULL  ← rows where due_date exists       │
│                                                                 │
│  Also: NULL != 3 → UNKNOWN → that row is excluded.              │
│  WHERE priority != 3 silently drops all NULL priority rows.     │
│  Fix: WHERE priority != 3 OR priority IS NULL                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 7: "The app connects as the postgres superuser"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ CONNECTING AS SUPERUSER                                     │
│                                                                 │
│  DATABASE_URL = "postgresql://postgres:pass@..."                │
│                              ▲                                  │
│                              Superuser. God mode.               │
│                                                                 │
│  Your application does not need to CREATE DATABASES,            │
│  DROP TABLES, or CREATE USERS at runtime. The superuser         │
│  account can do all of these — and so can anyone who            │
│  gets hold of your connection string.                           │
│                                                                 │
│  ✅ Create a restricted user for the application:               │
│  CREATE USER app_user WITH PASSWORD '...';                      │
│  GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES             │
│    IN SCHEMA public TO app_user;                                │
│                                                                 │
│  Superuser credentials = Superuser blast radius.                │
│  Principle of least privilege is not optional in production.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 8: "Foreign keys don't need indexes — the constraint handles it"

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ❌ UN-INDEXED FOREIGN KEY COLUMNS                              │
│                                                                 │
│  -- tasks.user_id references users.id                           │
│  -- tasks.category_id references categories.id                  │
│  -- Neither FK column has a manually created index.             │
│                                                                 │
│  The FK CONSTRAINT enforces referential integrity. Fine.        │
│  But it does NOT create an index on the FK column.              │
│  (PostgreSQL does not do this automatically. MySQL does.)       │
│                                                                 │
│  Every JOIN on tasks.user_id → full table scan of tasks.        │
│  With 1,000,000 tasks, that is 1,000,000 rows read per query.   │
│  Per request. Every time.                                       │
│                                                                 │
│  ✅ Index foreign key columns manually:                         │
│  CREATE INDEX idx_tasks_user_id ON tasks (user_id);             │
│  CREATE INDEX idx_tasks_category_id ON tasks (category_id);     │
│                                                                 │
│  You will write this SQL in Lecture 2 and formalise it          │
│  through SQLAlchemy in Week 6. Establish the habit now:         │
│  Every FK column gets an index. No exceptions.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              RELATIONAL DATABASE QUICK REFERENCE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STRUCTURE:                                                     │
│      Database → contains Tables                                 │
│      Table → has Columns (schema) and Rows (data)               │
│      Column → has a Name and a Type                             │
│      Row → one record (one instance of the schema)              │
│                                                                 │
│  KEYS:                                                          │
│      Primary Key → uniquely identifies each row (UNIQUE+NOT NULL)│
│      Foreign Key → references a primary key in another table    │
│      Composite Key → primary key spanning multiple columns      │
│                                                                 │
│  RELATIONSHIPS:                                                 │
│      One-to-Many (1:N) → FK on the "many" side                  │
│      Many-to-Many (N:N) → junction table with two FKs           │
│      One-to-One (1:1) → FK with UNIQUE constraint               │
│                                                                 │
│  CONSTRAINTS:                                                   │
│      NOT NULL → must have a value                               │
│      UNIQUE → no duplicates                                     │
│      CHECK → custom condition (e.g., age > 0)                   │
│      DEFAULT → auto-fill value                                  │
│      FOREIGN KEY → must reference existing row                  │
│                                                                 │
│  ACID:                                                          │
│      Atomicity → all-or-nothing operations                      │
│      Consistency → rules always enforced                        │
│      Isolation → concurrent access handled safely               │
│      Durability → committed data survives crashes               │
│                                                                 │
│  NORMALIZATION:                                                 │
│      1NF → one value per cell, no repeating groups              │
│      2NF → no partial dependencies (on composite key)           │
│      3NF → no transitive dependencies                           │
│      Mnemonic: "The key, the whole key, nothing but the key"    │
│                                                                 │
│  CORE POSTGRESQL TYPES:                                         │
│      Text: TEXT, VARCHAR(n)                                     │
│      Numbers: INTEGER, BIGINT, NUMERIC(p,s), REAL               │
│      Boolean: BOOLEAN                                           │
│      Time: TIMESTAMPTZ (always!), DATE, TIME                    │
│      Identity: SERIAL, UUID                                     │
│      Flexible: JSONB, ARRAY                                     │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ FLOAT for money          → Use NUMERIC(p,s)             │
│      ❌ TIMESTAMP without tz     → Use TIMESTAMPTZ              │
│      ❌ TEXT for everything       → Use proper types             │
│      ❌ One giant table           → Normalize to 3NF            │
│      ❌ JSONB for everything      → Use columns for queried data│
│      ❌ Skip foreign keys         → Always define them          │
│                                                                 │
┌─────────────────────────────────────────────────────────────────┐
│          QUICK REFERENCE — ADDITIONS (W5L1 new content)         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT-SERVER:                                                 │
│      App (client) ──TCP/IP──▶ PostgreSQL (server), port 5432   │
│      Connection string: postgresql://user:pass@host:port/db     │
│      App crash → DB unaffected, committed data safe             │
│      DB crash → app cannot read/write, no data loss             │
│      Always load credentials from environment variables         │
│                                                                 │
│  JOINS (Concept — SQL syntax in Lecture 2):                     │
│      JOIN recombines normalised tables by matching key values   │
│      INNER JOIN → only rows with a match on both sides          │
│      LEFT JOIN  → all left-table rows; NULL if no right match   │
│      Join condition: child_table.fk_col = parent_table.pk_col   │
│                                                                 │
│  INDEXES (Concept — EXPLAIN ANALYZE in Week 7):                 │
│      Primary Key → indexed automatically, always                │
│      Foreign Key → must be indexed MANUALLY in PostgreSQL       │
│      Index speeds reads O(log n) vs O(n) full table scan        │
│      Trade-off: slower writes (index must be maintained)        │
│                                                                 │
│  NULL SEMANTICS:                                                │
│      NULL ≠ zero, ≠ empty string, ≠ False ("unknown/absent")   │
│      NULL = NULL  → UNKNOWN (never TRUE)                        │
│      NULL != x    → UNKNOWN (row excluded from WHERE)           │
│      Use: IS NULL / IS NOT NULL (never = NULL or != NULL)       │
│      NULL in arithmetic → NULL (10 + NULL = NULL)               │
│                                                                 │
│  DATABASE SECURITY:                                             │
│      PostgreSQL users ≠ your app's user accounts (rows)         │
│      postgres = superuser (never use for app connection)        │
│      CREATE USER app_user; GRANT SELECT/INSERT/UPDATE/DELETE    │
│      Never commit connection strings to version control         │
│      Principle of least privilege: grant only what is needed    │
│                                                                 │
│  COMMON MISTAKES (additions):                                   │
│      ❌ WHERE col = NULL       → Use WHERE col IS NULL           │
│      ❌ Connect as postgres    → Use restricted app_user         │
│      ❌ No index on FK columns → Always index FK columns         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  A RELATIONAL DATABASE = A SPREADSHEET THAT GREW UP             │
│                                                                 │
│  Your in-memory dict:              A relational database:       │
│  ────────────────────              ──────────────────────       │
│  Data in RAM → volatile            Data on disk → durable       │
│  No schema → anything goes         Typed columns → enforced     │
│  No relationships → isolated       Foreign keys → connected     │
│  No constraints → trust your code  ACID → trust the database    │
│  dict.get() → manual filtering     SQL → powerful querying      │
│                                                                 │
│                                                                 │
│  THE LAYER MAP:                                                 │
│                                                                 │
│  ┌─────────────────────┐                                        │
│  │ Pydantic (API)      │  Validates input from users            │
│  ├─────────────────────┤                                        │
│  │ Python code (Logic) │  Business rules and workflows          │
│  ├─────────────────────┤                                        │
│  │ Database (Storage)  │  Final defense. ACID guarantees.       │
│  └─────────────────────┘  Data survives everything.             │
│                                                                 │
│  You've been building the top two layers for 4 weeks.           │
│  Now we add the foundation underneath.                          │
│                                                                 │
│                                                                 │
│  THE SPREADSHEET ANALOGY:                                       │
│  ├─ Database = The workbook                                     │
│  ├─ Table = A sheet/tab (with enforced column types)            │
│  ├─ Row = A row of data (one record)                            │
│  ├─ Primary Key = The unique row number                         │
│  ├─ Foreign Key = A cell referencing another sheet's row        │
│  ├─ Constraints = Data validation rules (but unbreakable)       │
│  └─ ACID = "Undo" that actually works, even after a crash       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│          UPDATED SPREADSHEET ANALOGY (new rows)                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Spreadsheet                 │  Relational Database             │
│  ────────────────────────────│──────────────────────────        │
│  VLOOKUP across sheets       │  JOIN (SQL syntax — Lecture 2)   │
│  Row number (built-in)       │  Primary Key (auto-indexed)      │
│  A-Z sort button on column   │  Index on a column               │
│  An empty cell               │  NULL (but ≠ zero or '')         │
│  Password-protected sheet    │  GRANT / REVOKE privileges        │
│  Separate Excel workbook     │  Separate PostgreSQL server       │
│    file on your drive        │    process, connected via TCP/IP  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│             THE COMPLETE ARCHITECTURE PICTURE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE (Weeks 1-4):                                            │
│  ─────────────────                                              │
│  One Python process. Data in memory. Volatile.                  │
│  Pydantic validates input. Your code validates logic.           │
│  Nothing else.                                                  │
│                                                                 │
│  AFTER (Week 5 onward):                                         │
│  ────────────────────                                           │
│  ┌──────────────────────────────────────────────────┐           │
│  │  CLIENT SIDE (your FastAPI process)              │           │
│  │  ├─ Pydantic: validates API input                │           │
│  │  ├─ Python code: business logic                  │           │
│  │  └─ Connects via TCP/IP to port 5432 as app_user │           │
│  └───────────────────────┬──────────────────────────┘           │
│                          │ TCP/IP                               │
│  ┌───────────────────────▼──────────────────────────┐           │
│  │  SERVER SIDE (PostgreSQL process)                │           │
│  │  ├─ Typed columns: rejects wrong types           │           │
│  │  ├─ Constraints: NOT NULL, UNIQUE, FK, CHECK     │           │
│  │  ├─ ACID: atomicity, consistency, isolation,     │           │
│  │  │         durability                            │           │
│  │  ├─ Indexes: O(log n) queries and JOINs          │           │
│  │  └─ Access control: app_user privileges          │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
│  Data is validated at the API boundary (Pydantic),              │
│  enforced at the storage boundary (DB constraints),             │
│  accessed efficiently (indexes), and protected at               │
│  the connection boundary (user privileges).                     │
│  Four independent layers. All required.                         │
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
│  WEEK 5, LECTURE 2 — SQL Fundamentals:                          │
│  └─ You'll WRITE the queries: SELECT, INSERT, UPDATE, DELETE.   │
│     Today was the "what" and "why." Next is the "how."          │
│     You'll also run PostgreSQL with Docker for the first time.  │
│                                                                 │
│  WEEK 5, LECTURE 3 — PostgreSQL-Specific Features:              │
│  └─ JSONB queries, full-text search, EXPLAIN, indexes.          │
│     The power types we introduced today, in depth.              │
│                                                                 │
│  WEEK 5, LECTURE 4 — Database Design Workshop:                  │
│  └─ Hands-on ER diagram design from requirements.               │
│     Junction tables, UUID vs serial, soft deletes.              │
│     You'll design the schema for your mini-project.             │
│                                                                 │
│  WEEK 6 — SQLAlchemy & Alembic:                                 │
│  └─ Map these tables to Python classes (ORM).                   │
│     Your dataclass knowledge (Week 1) + today's table           │
│     knowledge → SQLAlchemy models. Direct connection.           │
│                                                                 │
│  WEEK 6, LECTURE 4 — Async SQLAlchemy + FastAPI Integration:    │
│  └─ Replace your in-memory dict with real PostgreSQL.           │
│     Your Task Manager finally gets a real database.             │
│     Async (Week 1, Lecture 3) + database = async SQLAlchemy.    │
│                                                                 │
│  WEEK 7 — Advanced Database Patterns:                           │
│  └─ Query optimization, indexing, connection pooling.           │
│     Today's normalization decisions affect performance.         │
│     You'll learn to MEASURE and OPTIMIZE.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```