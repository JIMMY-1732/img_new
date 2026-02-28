# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PAIN FIRST, ORM SECOND                                         │
│  ───────────────────────                                        │
│  Students wrote raw SQL last week. They built a Python API      │
│  in Week 3-4. Now we FORCE them to imagine connecting the two   │
│  by hand — and let them feel how awful that would be.           │
│                                                                 │
│  BRIDGE, DON'T REPLACE                                          │
│  ────────────────────                                           │
│  The ORM does NOT replace SQL knowledge. It generates SQL.      │
│  We show the generated SQL for EVERYTHING so the ORM never      │
│  feels like magic. echo=True is always on.                      │
│                                                                 │
│  EVOLUTION, NOT REVOLUTION                                      │
│  ─────────────────────────                                      │
│  dataclass → Pydantic BaseModel → SQLAlchemy Model.             │
│  Each step builds on the last. Nothing is truly new —           │
│  it's the same idea applied to a different problem.             │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Type hints → Mapped[int] is just type hints for columns        │
│  Context managers → Session uses `with` exactly like files      │
│  Error handling → Database errors follow the same patterns      │
│  Dataclasses → Models ARE decorated classes with fields          │
│  Raw SQL (Week 5) → ORM generates the SQL you already know      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                 SQLALCHEMY ORM FUNDAMENTALS                     │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 Two Worlds Collide (Demonstration)                      │
│  ├─ 1.2 What is Object-Relational Mapping?                      │
│  ├─ 1.3 The Impedance Mismatch                                  │
│  └─ 1.4 SQLAlchemy as The Translator                            │
│                                                                 │
│  PART 2: THE ENGINE (35 min)                                    │
│  ├─ 2.1 SQLAlchemy Architecture (Two Layers)                    │
│  ├─ 2.2 The Engine (Your Connection Factory)                    │
│  ├─ 2.3 Connection URLs (Addressing the Database)               │
│  └─ 2.4 Creating Your First Engine                              │
│                                                                 │
│  PART 3: DEFINING MODELS (60 min)                               │
│  ├─ 3.1 DeclarativeBase — The Foundation                        │
│  ├─ 3.2 Your First Model (Mapped + mapped_column)               │
│  ├─ 3.3 Column Types (Python Types → SQL Types)                 │
│  ├─ 3.4 Column Constraints (Rules on Your Data)                 │
│  └─ 3.5 Dataclass → Pydantic → SQLAlchemy (The Evolution)       │
│                                                                 │
│  PART 4: THE SESSION (50 min)                                   │
│  ├─ 4.1 What is a Session? (The Shopping Cart)                  │
│  ├─ 4.2 Session as Context Manager (Connection to Week 1)       │
│  ├─ 4.3 CRUD Operations with Session                            │
│  ├─ 4.4 The Session Lifecycle (add → flush → commit)            │
│  └─ 4.5 Error Handling with Sessions (Connection to Week 1)     │
│                                                                 │
│  PART 5: CREATING TABLES & LOOKING AHEAD (25 min)              │
│  ├─ 5.1 metadata.create_all() — Development Only               │
│  ├─ 5.2 Inspecting Generated SQL (echo=True)                   │
│  ├─ 5.3 Raw SQL vs ORM: Side-by-Side                           │
│  └─ 5.4 Why NOT create_all in Production (Alembic Preview)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 Two Worlds Collide

**Set the scene. They've lived in two separate worlds for weeks.**

> "In Week 3-4, you built a Task Manager API with Python, FastAPI, and Pydantic. Your data lived in a Python dict. In Week 5, you designed schemas and wrote raw SQL against PostgreSQL. Two worlds. Today, we crash them together — and discover why that's painful."

**Show what happens when you try to connect them by hand:**

```python
# raw_pain.py — What your Task Manager would look like with raw SQL
import psycopg2
from datetime import datetime
from typing import Optional

# Your Pydantic model from Week 3-4 (unchanged)
from pydantic import BaseModel

class TaskResponse(BaseModel):
    id: int
    title: str
    description: Optional[str]
    is_completed: bool
    created_at: datetime


def get_connection():
    """Get a database connection. Every. Single. Time."""
    return psycopg2.connect(
        host="localhost",
        port=5432,
        dbname="taskmanager",
        user="postgres",
        password="postgres",
    )


def get_task(task_id: int) -> Optional[TaskResponse]:
    conn = get_connection()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "SELECT id, title, description, is_completed, created_at "
            "FROM tasks WHERE id = %s",
            (task_id,),
        )
        row = cursor.fetchone()

        if row is None:
            return None

        # Manual mapping — positional indexing. What is row[3] again?
        return TaskResponse(
            id=row[0],
            title=row[1],
            description=row[2],
            is_completed=row[3],
            created_at=row[4],
        )
    finally:
        cursor.close()
        conn.close()


def create_task(
    title: str, description: Optional[str] = None
) -> TaskResponse:
    conn = get_connection()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO tasks (title, description, is_completed) "
            "VALUES (%s, %s, %s) "
            "RETURNING id, title, description, is_completed, created_at",
            (title, description, False),
        )
        row = cursor.fetchone()
        conn.commit()

        # The SAME manual mapping. Again. For every function.
        return TaskResponse(
            id=row[0],
            title=row[1],
            description=row[2],
            is_completed=row[3],
            created_at=row[4],
        )
    except Exception:
        conn.rollback()
        raise
    finally:
        cursor.close()
        conn.close()


def update_task(task_id: int, title: str) -> Optional[TaskResponse]:
    conn = get_connection()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE tasks SET title = %s WHERE id = %s "
            "RETURNING id, title, description, is_completed, created_at",
            (title, task_id),
        )
        row = cursor.fetchone()
        conn.commit()

        if row is None:
            return None

        # The SAME mapping. AGAIN.
        return TaskResponse(
            id=row[0],
            title=row[1],
            description=row[2],
            is_completed=row[3],
            created_at=row[4],
        )
    except Exception:
        conn.rollback()
        raise
    finally:
        cursor.close()
        conn.close()
```

**Let that sink in. Then ask the class:**

> "Three functions. The exact same try/finally block, the exact same row-to-object mapping, copied three times. And we only have ONE table. Your Week 5 schema had four or five. Now imagine 20 endpoints, 5 tables, each with 10+ columns. What happens when you add a column to the tasks table?"

Answer: **Every single function that touches that table breaks. You have to find them all, update the SQL string, update the positional index, and pray you didn't miss one.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE RAW SQL PAIN POINTS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. CONNECTION BOILERPLATE                                      │
│     connect → try → finally → close.                            │
│     Every function. Every time.                                 │
│                                                                 │
│  2. SQL AS STRINGS                                              │
│     No syntax checking. No autocomplete. Typos are runtime      │
│     errors. "SELCT * FORM users" — Python won't catch that.     │
│                                                                 │
│  3. POSITIONAL INDEXING                                         │
│     row[0], row[1], row[3]... What is row[3]?                   │
│     Swap two columns in SELECT → everything silently breaks.    │
│                                                                 │
│  4. REPEATED MAPPING                                            │
│     The same row → object conversion, duplicated in every       │
│     function. Violates DRY (Don't Repeat Yourself).             │
│                                                                 │
│  5. SCHEMA CHANGES ARE LANDMINES                                │
│     Add a column → update EVERY query, EVERY mapping.           │
│     Miss one → silent bug or crash in production.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now show the promise:**

```python
# The same operations with SQLAlchemy ORM (preview — we'll build up to this)
from sqlalchemy import select
from sqlalchemy.orm import Session

def get_task(session: Session, task_id: int) -> Optional[Task]:
    return session.get(Task, task_id)

def create_task(session: Session, title: str, description: Optional[str] = None) -> Task:
    task = Task(title=title, description=description)
    session.add(task)
    session.commit()
    session.refresh(task)
    return task

def update_task(session: Session, task_id: int, title: str) -> Optional[Task]:
    task = session.get(Task, task_id)
    if task is None:
        return None
    task.title = title
    session.commit()
    return task
```

> "Same three operations. No SQL strings. No positional indexing. No connection boilerplate. Add a column to Task? You change the model once. Every query still works."

---

## 1.2 What is Object-Relational Mapping?

**Define the core concept:**

```
┌─────────────────────────────────────────────────────────────────┐
│              OBJECT-RELATIONAL MAPPING (ORM)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  An ORM is a technique that maps:                               │
│                                                                 │
│      Python classes    ←→    Database tables                    │
│      Class instances   ←→    Table rows                         │
│      Class attributes  ←→    Table columns                      │
│      Python methods    ←→    SQL queries                        │
│                                                                 │
│                                                                 │
│  You define the mapping ONCE. The ORM handles the               │
│  translation in BOTH directions — forever.                      │
│                                                                 │
│                                                                 │
│  PYTHON WORLD              MAPPING              DATABASE WORLD  │
│  ────────────              ───────              ──────────────  │
│                                                                 │
│  class User          ─────────────▶       CREATE TABLE users    │
│      id: int         ─────────────▶           id INTEGER        │
│      name: str       ─────────────▶           name VARCHAR      │
│      email: str      ─────────────▶           email VARCHAR     │
│                                                                 │
│  user = User(...)    ─────────────▶       INSERT INTO users...  │
│  user.name           ◀─────────────       SELECT name FROM...   │
│  user.name = "Bob"   ─────────────▶       UPDATE users SET...   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Impedance Mismatch

**Why can't we just use SQL directly? Why is this mapping hard?**

> "Python and SQL were designed for completely different purposes. They think about data in fundamentally incompatible ways. This is called the **object-relational impedance mismatch** — a fancy term for 'these two worlds don't naturally fit together.'"

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE IMPEDANCE MISMATCH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PYTHON THINKS IN:              SQL THINKS IN:                  │
│  ─────────────────              ─────────────                   │
│  Objects with behavior          Rows in flat tables             │
│  Nested structures              Normalized relations            │
│  Inheritance hierarchies        No inheritance concept          │
│  References (object.other)      Foreign keys (integer IDs)      │
│  In-memory, ephemeral           On-disk, persistent             │
│  Rich types (Any class)         Fixed types (INT, VARCHAR...)   │
│  Identity by id()               Identity by primary key         │
│                                                                 │
│                                                                 │
│  Example of the clash:                                          │
│                                                                 │
│  PYTHON:                        SQL:                            │
│  ────────                       ─────                           │
│  task.category.name             SELECT c.name                   │
│                                 FROM tasks t                    │
│  (one dot access)               JOIN categories c               │
│                                 ON t.category_id = c.id         │
│                                 WHERE t.id = 1                  │
│                                                                 │
│                                 (a whole JOIN query)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "This mismatch is unavoidable. The ORM doesn't eliminate it — it manages it. Behind every `task.category.name`, there is still a JOIN happening. The ORM just writes that JOIN for you. And that's why you MUST understand the SQL underneath. An ORM without SQL knowledge is a loaded gun."

---

## 1.4 SQLAlchemy as The Translator

**The analogy that will carry us through the lecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE TRANSLATOR ANALOGY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You speak Python. Your database speaks SQL.                    │
│  SQLAlchemy is your professional translator.                    │
│                                                                 │
│                                                                 │
│  WITHOUT ORM (doing it yourself):                               │
│  ─────────────────────────────────                              │
│                                                                 │
│  You: 🧑 (speaks Python)                                        │
│       │                                                         │
│       │ ── hand-writes SQL strings ──▶  📋 raw SQL              │
│       │                                    │                    │
│       │                                    ▼                    │
│       │                               🗄️ Database               │
│       │                                    │                    │
│       │ ◀── manually parses rows ─────  📋 raw results          │
│       │                                                         │
│  Slow. Error-prone. Exhausting.                                 │
│                                                                 │
│                                                                 │
│  WITH ORM (SQLAlchemy translates):                              │
│  ─────────────────────────────────                              │
│                                                                 │
│  You: 🧑 (speaks Python)                                        │
│       │                                                         │
│       │ ── session.add(user) ──▶  🔄 SQLAlchemy                 │
│       │                               │                         │
│       │                               │ ── INSERT INTO ... ──▶  │
│       │                               │                     🗄️  │
│       │                               │ ◀── result rows ─────  │
│       │                               │                         │
│       │ ◀── User object ───────── 🔄                            │
│       │                                                         │
│  Fast. Type-safe. One mapping definition.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to concepts we'll learn today:**

```
Translator Analogy       │  SQLAlchemy Concept
─────────────────────────│───────────────────────────────
Teaching the translator  │  Defining models
your vocabulary          │  (class User, class Task)
                         │
Setting up the phone     │  Creating the Engine
line to the database     │  (create_engine)
                         │
One conversation /       │  The Session
translation session      │  (with Session() as session)
                         │
"Send my message"        │  session.commit()
                         │
"Unsend everything"      │  session.rollback()
                         │
"Repeat that back to me" │  session.refresh()
```

---

# PART 2: THE ENGINE

## 2.1 SQLAlchemy Architecture (Two Layers)

**SQLAlchemy is NOT just an ORM. It has two distinct layers:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SQLALCHEMY ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────┐          │
│  │                  YOUR CODE                        │          │
│  └───────────────────────┬───────────────────────────┘          │
│                          │                                      │
│                          ▼                                      │
│  ┌───────────────────────────────────────────────────┐          │
│  │              ORM LAYER (High Level)               │          │
│  │                                                   │          │
│  │  What we learn TODAY:                             │          │
│  │  • Model classes (User, Task)                     │          │
│  │  • Session (add, commit, query)                   │          │
│  │  • Mapped, mapped_column                          │          │
│  │                                                   │          │
│  │  You speak Python. The ORM translates.            │          │
│  │                                                   │          │
│  └───────────────────────┬───────────────────────────┘          │
│                          │                                      │
│                          ▼                                      │
│  ┌───────────────────────────────────────────────────┐          │
│  │              CORE LAYER (Low Level)               │          │
│  │                                                   │          │
│  │  Under the hood:                                  │          │
│  │  • Engine (connection management)                 │          │
│  │  • SQL Expression Language (build SQL in Python)  │          │
│  │  • Connection pooling                             │          │
│  │  • Dialect system (PostgreSQL, MySQL, SQLite)     │          │
│  │                                                   │          │
│  │  The ORM is BUILT ON TOP of Core.                 │          │
│  │                                                   │          │
│  └───────────────────────┬───────────────────────────┘          │
│                          │                                      │
│                          ▼                                      │
│  ┌───────────────────────────────────────────────────┐          │
│  │              DATABASE DRIVER (DBAPI)              │          │
│  │                                                   │          │
│  │  The actual wire protocol:                        │          │
│  │  • psycopg2 (PostgreSQL, sync)                    │          │
│  │  • asyncpg (PostgreSQL, async — Week 6, Lec 4)   │          │
│  │  • sqlite3 (SQLite, built-in)                     │          │
│  │                                                   │          │
│  └───────────────────────┬───────────────────────────┘          │
│                          │                                      │
│                          ▼                                      │
│  ┌───────────────────────────────────────────────────┐          │
│  │              DATABASE SERVER                      │          │
│  │              (PostgreSQL in Docker — Week 5)      │          │
│  └───────────────────────────────────────────────────┘          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why this matters:**

> "You can use SQLAlchemy Core without the ORM — writing SQL expressions in Python without classes or sessions. But you can't use the ORM without Core. The ORM sits on top and calls Core underneath. Today we learn the ORM layer, but we'll always peek at what it generates underneath. No black boxes."

---

## 2.2 The Engine (Your Connection Factory)

**The Engine is the starting point for all SQLAlchemy operations.**

```
┌─────────────────────────────────────────────────────────────────┐
│                       THE ENGINE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The Engine is a CONNECTION FACTORY.                            │
│                                                                 │
│  It does NOT hold a single connection.                          │
│  It manages a POOL of connections.                              │
│                                                                 │
│                    ┌──────────────┐                              │
│                    │    ENGINE    │                              │
│                    │              │                              │
│                    │  ┌────────┐  │                              │
│                    │  │ Pool   │  │                              │
│                    │  │        │  │                              │
│                    │  │ conn 1 │  │──────▶  🗄️ PostgreSQL       │
│                    │  │ conn 2 │  │──────▶  🗄️ (in Docker)     │
│                    │  │ conn 3 │  │──────▶                      │
│                    │  │  ...   │  │                              │
│                    │  └────────┘  │                              │
│                    │              │                              │
│                    └──────────────┘                              │
│                                                                 │
│  CREATE ONCE at application startup.                            │
│  REUSE throughout the application's lifetime.                   │
│  DISPOSE at application shutdown.                               │
│                                                                 │
│  Analogy: The Engine is like a phone company account.           │
│  You sign up once. Then you make calls (Sessions) whenever.     │
│  The company handles the phone lines (connection pool) for you. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Connection URLs (Addressing the Database)

**The Engine needs to know WHERE your database is. You tell it with a URL.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONNECTION URL FORMAT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  postgresql+psycopg2://user:password@host:port/dbname           │
│  ──────── ─────────── ──── ──────── ──── ──── ──────           │
│     │         │         │      │      │    │     │              │
│     │         │         │      │      │    │     └─ Database    │
│     │         │         │      │      │    └─ Port (5432)       │
│     │         │         │      │      └─ Host (localhost)       │
│     │         │         │      └─ Password                      │
│     │         │         └─ Username                             │
│     │         └─ Driver (which library talks to the DB)         │
│     └─ Dialect (which SQL flavor)                               │
│                                                                 │
│                                                                 │
│  EXAMPLES:                                                      │
│  ─────────                                                      │
│                                                                 │
│  # PostgreSQL (what we use in this course)                      │
│  "postgresql://postgres:postgres@localhost:5432/taskmanager"     │
│                                                                 │
│  # PostgreSQL with explicit driver                              │
│  "postgresql+psycopg2://postgres:postgres@localhost:5432/taskdb" │
│                                                                 │
│  # SQLite (file-based, great for quick tests)                   │
│  "sqlite:///./test.db"                                          │
│                                                                 │
│  # SQLite in-memory (perfect for unit tests)                    │
│  "sqlite:///:memory:"                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The dialect+driver combo determines how SQLAlchemy talks to your specific database:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   DIALECT + DRIVER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Dialect        Driver          URL Prefix         Mode         │
│  ──────────     ──────────      ──────────         ──────       │
│  PostgreSQL     psycopg2        postgresql://       Sync        │
│  PostgreSQL     asyncpg         postgresql+asyncpg  Async       │
│  SQLite         sqlite3         sqlite://            Sync       │
│  MySQL          pymysql         mysql+pymysql://     Sync       │
│                                                                 │
│  TODAY: postgresql:// (sync, psycopg2)                          │
│  LECTURE 4: postgresql+asyncpg:// (async)                       │
│                                                                 │
│  If you omit the driver, SQLAlchemy picks the default:          │
│  "postgresql://..." defaults to psycopg2                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Creating Your First Engine

**Let's create an Engine that connects to the PostgreSQL you set up in Week 5:**

```python
# database.py — Your database configuration module
from sqlalchemy import create_engine

# Your PostgreSQL from Week 5 (running in Docker)
DATABASE_URL = "postgresql://postgres:postgres@localhost:5432/taskmanager"

# Create the engine — do this ONCE
engine = create_engine(
    DATABASE_URL,
    echo=True,  # ← CRITICAL: prints every SQL statement SQLAlchemy generates
)
```

**What is `echo=True`?**

```
┌─────────────────────────────────────────────────────────────────┐
│                     echo=True                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  echo=True makes SQLAlchemy print EVERY SQL statement           │
│  it generates to the console.                                   │
│                                                                 │
│  WHY WE ALWAYS USE IT DURING LEARNING:                          │
│                                                                 │
│  You write:     session.get(User, 1)                            │
│                                                                 │
│  Console shows: SELECT users.id, users.name, users.email        │
│                 FROM users                                      │
│                 WHERE users.id = %(pk_1)s                       │
│                 [parameters: {'pk_1': 1}]                       │
│                                                                 │
│  The ORM is NOT magic. It writes the SAME SQL you learned       │
│  in Week 5. echo=True proves it.                                │
│                                                                 │
│  ⚠️  Turn it OFF in production (echo=False). It's noisy.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Verify the connection:**

```python
# test_connection.py — Verify your engine works
from sqlalchemy import create_engine, text

DATABASE_URL = "postgresql://postgres:postgres@localhost:5432/taskmanager"
engine = create_engine(DATABASE_URL, echo=True)

# Use the engine to run a simple query
with engine.connect() as conn:
    result = conn.execute(text("SELECT 1"))
    print(result.scalar())  # Output: 1
    print("Connection successful!")
```

```
# Console output:
# SELECT 1
# [raw sql] ()
# 1
# Connection successful!
```

**The `text()` function:**

> "Notice `text('SELECT 1')`. When you pass raw SQL strings to SQLAlchemy, you wrap them in `text()`. This tells SQLAlchemy 'I'm giving you raw SQL, don't try to parse it.' You already know this SQL from Week 5 — `text()` is just the wrapper."

**Connection test failed? Check these:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CONNECTION TROUBLESHOOTING                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "connection refused"                                           │
│  └─ Is Docker running? Is the Postgres container up?            │
│     docker ps                                                   │
│     docker start your_postgres_container                        │
│                                                                 │
│  "password authentication failed"                               │
│  └─ Check your username/password in the URL.                    │
│     Match them with your Docker Compose or docker run command.  │
│                                                                 │
│  "database does not exist"                                      │
│  └─ Create it first:                                            │
│     docker exec -it postgres psql -U postgres                   │
│     CREATE DATABASE taskmanager;                                │
│                                                                 │
│  "no module named psycopg2"                                     │
│  └─ Install the driver:                                         │
│     pip install psycopg2-binary                                 │
│     (use psycopg2-binary for development, psycopg2 for prod)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: DEFINING MODELS

## 3.1 DeclarativeBase — The Foundation

**Every SQLAlchemy ORM model needs a base class. In SQLAlchemy 2.0, that's `DeclarativeBase`:**

```python
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

**That's it. Three lines. But what does it do?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    DeclarativeBase                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DeclarativeBase is the ANCESTOR of all your models.            │
│                                                                 │
│  When you create class User(Base):                              │
│  SQLAlchemy automatically:                                      │
│                                                                 │
│  1. Registers the class as a database table                     │
│  2. Reads your type hints to determine column types             │
│  3. Generates an __init__ method (like dataclass!)              │
│  4. Tracks all models in Base.metadata (the catalog)            │
│                                                                 │
│                                                                 │
│        ┌──────────────┐                                         │
│        │     Base     │ ← DeclarativeBase                       │
│        │  (metadata)  │    Holds the catalog of all tables      │
│        └──────┬───────┘                                         │
│               │                                                 │
│        ┌──────┴──────┐                                          │
│        │             │                                          │
│        ▼             ▼                                          │
│  ┌──────────┐  ┌──────────┐                                     │
│  │  User    │  │  Task    │                                     │
│  │  (model) │  │  (model) │   ← Your models inherit from Base  │
│  └──────────┘  └──────────┘                                     │
│                                                                 │
│  Base.metadata knows about ALL of them.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Think of `Base` as the translator's master dictionary. Every model you define is a new entry. The translator (SQLAlchemy) can look up any of them when it needs to generate SQL.**

**Convention:** Define `Base` once in a dedicated file (e.g., `database.py` or `models/base.py`). Import it everywhere.

```python
# database.py — Single source of truth
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase

DATABASE_URL = "postgresql://postgres:postgres@localhost:5432/taskmanager"
engine = create_engine(DATABASE_URL, echo=True)

class Base(DeclarativeBase):
    pass
```

---

## 3.2 Your First Model (Mapped + mapped_column)

**A model is a Python class that maps to a database table.**

```python
# models.py
from datetime import datetime
from typing import Optional

from sqlalchemy import String, Text
from sqlalchemy.orm import Mapped, mapped_column

from database import Base


class User(Base):
    __tablename__ = "users"    # ← Name of the table in PostgreSQL
    
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)
    bio: Mapped[Optional[str]] = mapped_column(Text)
    is_active: Mapped[bool] = mapped_column(default=True)
    
    def __repr__(self) -> str:
        return f"User(id={self.id!r}, username={self.username!r})"
```

**Let's break down every piece:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  ANATOMY OF A MODEL                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  class User(Base):                                              │
│  ─────────────────                                              │
│  Inherits from Base → SQLAlchemy knows this is a table.         │
│                                                                 │
│                                                                 │
│  __tablename__ = "users"                                        │
│  ───────────────────────                                        │
│  The actual table name in PostgreSQL.                           │
│  Convention: class singular (User), table plural (users).       │
│  This is REQUIRED (unless auto-generating, which we won't).     │
│                                                                 │
│                                                                 │
│  id: Mapped[int] = mapped_column(primary_key=True)              │
│  ──── ──────────   ─────────────────────────────────            │
│   │      │              │                                       │
│   │      │              └─ Column configuration                 │
│   │      │                 (constraints, defaults, etc.)        │
│   │      │                                                      │
│   │      └─ The TYPE of this column.                            │
│   │         Mapped[int] → INTEGER in SQL.                       │
│   │         Mapped[str] → needs String(n) or Text.              │
│   │         Mapped[Optional[str]] → nullable column.            │
│   │                                                             │
│   └─ The Python attribute name.                                 │
│      user.id gives you this column's value.                     │
│                                                                 │
│                                                                 │
│  def __repr__(self) -> str:                                     │
│  ──────────────────────────                                     │
│  Optional but STRONGLY recommended.                             │
│  Makes debugging bearable (Week 2 callback!).                   │
│  Without it: <models.User object at 0x7f...>                    │
│  With it: User(id=1, username='alice')                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The two key imports:**

```python
from sqlalchemy.orm import Mapped, mapped_column
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 Mapped vs mapped_column                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Mapped[type]                                                   │
│  ─────────────                                                  │
│  A TYPE ANNOTATION. Tells Python and SQLAlchemy                 │
│  "this attribute exists and its Python type is X."              │
│  It determines the SQL column type.                             │
│                                                                 │
│  mapped_column(...)                                             │
│  ──────────────────                                             │
│  A COLUMN CONFIGURATION. Tells SQLAlchemy HOW to               │
│  create and behave with this column:                            │
│  primary key? unique? default value? max length?                │
│                                                                 │
│                                                                 │
│  Together:                                                      │
│                                                                 │
│  username: Mapped[str] = mapped_column(String(50), unique=True) │
│            ──────────    ────────────────────────────────────    │
│            "It's a str"  "VARCHAR(50), must be unique"           │
│                                                                 │
│  Mapped says WHAT. mapped_column says HOW.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What SQL does this model generate?**

> "Remember: `echo=True` shows us everything. This User model, when SQLAlchemy creates the table, generates exactly the SQL you would have written in Week 5:"

```sql
-- SQLAlchemy generates this (you'll see it in your console):
CREATE TABLE users (
    id SERIAL NOT NULL,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(255) NOT NULL,
    bio TEXT,
    is_active BOOLEAN NOT NULL,
    PRIMARY KEY (id),
    UNIQUE (username),
    UNIQUE (email)
)
```

> "Look familiar? That's the same CREATE TABLE you wrote by hand last week. The ORM wrote it for you from the Python class."

---

## 3.3 Column Types (Python Types → SQL Types)

**`Mapped[type]` determines the SQL column type. Here's the translation table:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  TYPE MAPPING TABLE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python Type        SQLAlchemy Type      PostgreSQL Type        │
│  (in Mapped[])      (in mapped_column)   (generated SQL)        │
│  ──────────────     ──────────────────   ──────────────         │
│  int                Integer              INTEGER / SERIAL       │
│  str                String(n)            VARCHAR(n)             │
│  str                Text                 TEXT                   │
│  bool               Boolean              BOOLEAN                │
│  float              Float                FLOAT / DOUBLE         │
│  datetime           DateTime             TIMESTAMP              │
│  date               Date                 DATE                   │
│  time               Time                 TIME                   │
│  uuid.UUID          Uuid                 UUID                   │
│                                                                 │
│  Optional[str]      (auto nullable)      VARCHAR / NULL         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When does `Mapped[type]` need an explicit SQLAlchemy type in `mapped_column()`?**

```python
from datetime import datetime, date
from uuid import UUID
from typing import Optional

from sqlalchemy import String, Text, DateTime, Uuid
from sqlalchemy.orm import Mapped, mapped_column

class Task(Base):
    __tablename__ = "tasks"
    
    # int, bool, float → SQLAlchemy infers the SQL type automatically.
    # No need for explicit type in mapped_column.
    id: Mapped[int] = mapped_column(primary_key=True)
    is_completed: Mapped[bool] = mapped_column(default=False)
    priority: Mapped[int] = mapped_column(default=0)
    
    # str → You MUST specify String(n) or Text.
    # Why? Because "str" could mean VARCHAR(50) or VARCHAR(255) or TEXT.
    # SQLAlchemy can't guess the length you need.
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[Optional[str]] = mapped_column(Text)
    
    # datetime → SQLAlchemy infers DateTime, but you may want to be explicit.
    created_at: Mapped[datetime] = mapped_column(DateTime)
    
    # uuid → Use Uuid type
    external_id: Mapped[UUID] = mapped_column(Uuid)
```

**The `Optional` rule — this is important:**

```
┌─────────────────────────────────────────────────────────────────┐
│               NULLABLE vs NOT NULL                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Mapped[str]                  → NOT NULL in SQL                 │
│  Mapped[Optional[str]]        → nullable in SQL (NULL allowed)  │
│                                                                 │
│                                                                 │
│  EXAMPLES:                                                      │
│                                                                 │
│  # This column CANNOT be NULL — database enforces it            │
│  title: Mapped[str] = mapped_column(String(200))                │
│  # → title VARCHAR(200) NOT NULL                                │
│                                                                 │
│  # This column CAN be NULL — it's optional                      │
│  description: Mapped[Optional[str]] = mapped_column(Text)       │
│  # → description TEXT  (nullable by default)                    │
│                                                                 │
│                                                                 │
│  This is the SAME Optional you learned in Week 1, Lecture 1!    │
│  It means the same thing: "this value might not exist."         │
│  In type hints: could be None.                                  │
│  In database: could be NULL.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Column Constraints (Rules on Your Data)

**Constraints go inside `mapped_column()`. You already know these from Week 5 — now see them in Python:**

```python
from datetime import datetime
from typing import Optional
from uuid import uuid4, UUID

from sqlalchemy import String, Text, DateTime, Uuid, func
from sqlalchemy.orm import Mapped, mapped_column

from database import Base


class Task(Base):
    __tablename__ = "tasks"
    
    # PRIMARY KEY — uniquely identifies each row (Week 5)
    id: Mapped[int] = mapped_column(primary_key=True)
    
    # UNIQUE — no two rows can have the same value
    slug: Mapped[str] = mapped_column(String(200), unique=True)
    
    # NOT NULL — enforced by Mapped[str] (no Optional)
    title: Mapped[str] = mapped_column(String(200))
    
    # NULLABLE — enforced by Mapped[Optional[str]]
    description: Mapped[Optional[str]] = mapped_column(Text)
    
    # PYTHON DEFAULT — Python sets this value before INSERT
    is_completed: Mapped[bool] = mapped_column(default=False)
    
    # SERVER DEFAULT — PostgreSQL sets this value during INSERT
    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()
    )
    
    # INDEX — makes lookups on this column faster (Week 5, Lecture 3!)
    external_id: Mapped[UUID] = mapped_column(
        Uuid, default=uuid4, index=True
    )
    
    def __repr__(self) -> str:
        return f"Task(id={self.id!r}, title={self.title!r})"
```

**`default` vs `server_default` — a critical distinction:**

```
┌─────────────────────────────────────────────────────────────────┐
│              default vs server_default                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  default=False                                                  │
│  ──────────────                                                 │
│  PYTHON sets the value before sending the INSERT to the DB.     │
│                                                                 │
│  Python:     task = Task(title="Buy milk")                      │
│              # task.is_completed is already False in Python      │
│  SQL sent:   INSERT INTO tasks (title, is_completed)            │
│              VALUES ('Buy milk', false)                          │
│                                                                 │
│                                                                 │
│  server_default=func.now()                                      │
│  ─────────────────────────                                      │
│  POSTGRESQL sets the value when it processes the INSERT.        │
│                                                                 │
│  Python:     task = Task(title="Buy milk")                      │
│              # task.created_at is None in Python (not set yet!) │
│  SQL sent:   INSERT INTO tasks (title)                          │
│              VALUES ('Buy milk')                                │
│              -- PostgreSQL fills in created_at = NOW()           │
│                                                                 │
│  After commit + refresh:                                        │
│              task.created_at = 2025-03-15 14:30:00  ← from DB   │
│                                                                 │
│                                                                 │
│  WHEN TO USE WHICH:                                             │
│  ├─ default      → Python values (uuid4, False, 0, "draft")    │
│  └─ server_default → DB-generated (NOW(), gen_random_uuid())    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Dataclass → Pydantic → SQLAlchemy (The Evolution)

**Connection to what you've learned:**

> "You've already defined structured data in three ways this course. SQLAlchemy models are the next evolution — not something entirely new."

```python
# STEP 1: Dataclass (Week 1, Lecture 2)
# Purpose: Structure data in plain Python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class TaskDC:
    title: str
    description: Optional[str] = None
    is_completed: bool = False
    id: Optional[int] = field(default=None)


# STEP 2: Pydantic BaseModel (Week 3, Lecture 3)
# Purpose: Validate data at API boundaries
from pydantic import BaseModel, Field

class TaskSchema(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    description: Optional[str] = None
    is_completed: bool = False


# STEP 3: SQLAlchemy Model (TODAY)
# Purpose: Map data to database tables
from sqlalchemy import String, Text
from sqlalchemy.orm import Mapped, mapped_column

class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[Optional[str]] = mapped_column(Text)
    is_completed: Mapped[bool] = mapped_column(default=False)
```

**Compare them side by side:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE EVOLUTION OF STRUCTURED DATA                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                  dataclass    Pydantic       SQLAlchemy          │
│                  ─────────    ────────       ──────────          │
│  Purpose:        Structure    Validate       Map to DB           │
│  Decorator:      @dataclass   (inherits)     (inherits)          │
│  Base class:     None         BaseModel      Base                │
│  Type hints:     ✅ Yes       ✅ Yes         ✅ Yes (Mapped[])   │
│  Defaults:       = value      = value        default=value       │
│  Validation:     ❌ No        ✅ Yes         ❌ No*              │
│  DB awareness:   ❌ No        ❌ No          ✅ Yes              │
│  Auto __init__:  ✅ Yes       ✅ Yes         ✅ Yes              │
│                                                                 │
│  * SQLAlchemy models don't validate — Pydantic does.            │
│    They work TOGETHER. More on this in Lecture 4.               │
│                                                                 │
│                                                                 │
│  IN YOUR APPLICATION, YOU'LL HAVE BOTH:                         │
│                                                                 │
│  API request ──▶ Pydantic model ──▶ SQLAlchemy model ──▶ DB     │
│  (validates)      (clean data)       (persists data)            │
│                                                                 │
│  DB ──▶ SQLAlchemy model ──▶ Pydantic model ──▶ API response    │
│         (loads data)          (serializes)       (sends JSON)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This is the key insight: Pydantic guards the **door** (API boundary). SQLAlchemy guards the **vault** (database). Different jobs, same data. In Lecture 4 this week, we'll wire them together."

---

# PART 4: THE SESSION

## 4.1 What is a Session? (The Shopping Cart)

**The Session is your workspace for talking to the database.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      THE SESSION                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Think of the Session as a SHOPPING CART at a store.            │
│                                                                 │
│  1. You grab a cart (open a Session)                            │
│  2. You add items, remove items, change quantities              │
│     (add, delete, modify objects)                               │
│  3. You go to checkout and pay (commit)                         │
│     → Changes are saved to the database                        │
│  4. OR you abandon the cart (rollback)                          │
│     → Nothing happened. No changes saved.                      │
│  5. You leave the store (close the Session)                     │
│                                                                 │
│                                                                 │
│  The Session:                                                   │
│  ├─ Borrows a connection from the Engine's pool                 │
│  ├─ Tracks all objects you've loaded or added                   │
│  ├─ Accumulates changes in memory                               │
│  ├─ Sends changes to DB on commit                               │
│  ├─ Undoes all changes on rollback                              │
│  └─ Returns the connection to the pool when closed              │
│                                                                 │
│                                                                 │
│        ┌──────────┐                                             │
│        │  Engine   │                                            │
│        │  (pool)   │                                            │
│        └─────┬─────┘                                            │
│              │ borrows connection                                │
│              ▼                                                   │
│        ┌──────────────────────────────┐                          │
│        │         SESSION              │                          │
│        │                              │                          │
│        │  ┌─────────────────────────┐ │                          │
│        │  │  Identity Map           │ │                          │
│        │  │  (tracks loaded objects)│ │                          │
│        │  │                         │ │                          │
│        │  │  User(id=1) ───────────▶│ │ only one copy per PK    │
│        │  │  Task(id=5) ───────────▶│ │                          │
│        │  └─────────────────────────┘ │                          │
│        │                              │                          │
│        │  Pending changes:            │                          │
│        │  ├─ new: [User(id=None)]     │ ← added, not yet saved  │
│        │  ├─ dirty: [Task(id=5)]      │ ← modified              │
│        │  └─ deleted: []              │                          │
│        │                              │                          │
│        └──────────────────────────────┘                          │
│              │                                                   │
│              │ commit() → sends SQL → returns connection         │
│              ▼                                                   │
│           🗄️ Database                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Identity Map — the detail that catches beginners:**

> "The Session keeps an **identity map**: a dictionary of every object it knows about, keyed by primary key. If you load `User(id=1)` twice in the same Session, you get the **same Python object** both times — not two copies. This prevents inconsistencies."

```python
# Identity map in action:
with Session(engine) as session:
    user_a = session.get(User, 1)  # Loads from DB
    user_b = session.get(User, 1)  # Returns SAME object from identity map
    
    print(user_a is user_b)  # True — same object in memory!
```

---

## 4.2 Session as Context Manager (Connection to Week 1)

**Connection to what you've learned:**

> "Remember context managers from Week 1, Lecture 2? The `with` statement ensures cleanup happens even if an error occurs. Session uses this exact pattern."

```python
# Week 1, Lecture 2 — Context managers for file cleanup
with open("data.txt") as f:
    data = f.read()
# File is automatically closed, even if an exception occurred


# Week 6, Lecture 1 — Context managers for session cleanup
from sqlalchemy.orm import Session

with Session(engine) as session:
    user = session.get(User, 1)
    print(user)
# Session is automatically closed, connection returned to pool
```

```
┌─────────────────────────────────────────────────────────────────┐
│             SESSION AS CONTEXT MANAGER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  with Session(engine) as session:                               │
│       │                                                         │
│       ├── Borrows a connection from the pool                    │
│       │                                                         │
│       │   ... your database operations ...                      │
│       │                                                         │
│       └── On exit:                                              │
│           ├── If no exception: closes session normally           │
│           ├── If exception: rolls back, then closes             │
│           └── Always: returns connection to pool                │
│                                                                 │
│  EXACTLY the same pattern as files, network connections,        │
│  and every other resource you've used with `with`.              │
│                                                                 │
│  ⚠️  NOTE: `with Session(engine) as session:` auto-CLOSES       │
│  the session, but does NOT auto-commit. You must call           │
│  session.commit() explicitly when you want to save changes.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The `sessionmaker` factory — the production pattern:**

```python
# For real applications, don't create Session(engine) everywhere.
# Create a factory ONCE, then use it repeatedly.

from sqlalchemy.orm import sessionmaker

# Create the factory — do this ONCE alongside your engine
SessionLocal = sessionmaker(bind=engine)

# Use the factory — do this every time you need a session
with SessionLocal() as session:
    user = session.get(User, 1)

# Why a factory?
# 1. Configure once, use everywhere (consistent settings)
# 2. Easy to swap for testing (inject a test session factory)
# 3. FastAPI integration needs it (Lecture 4 this week)
```

**Updated database.py (our single source of truth):**

```python
# database.py — Complete setup
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

DATABASE_URL = "postgresql://postgres:postgres@localhost:5432/taskmanager"

engine = create_engine(DATABASE_URL, echo=True)

SessionLocal = sessionmaker(bind=engine)

class Base(DeclarativeBase):
    pass
```

---

## 4.3 CRUD Operations with Session

**CREATE — Adding new objects:**

```python
from database import SessionLocal
from models import User

# Create a single user
with SessionLocal() as session:
    new_user = User(
        username="alice",
        email="alice@example.com",
        bio="Backend developer",
    )
    
    print(new_user.id)   # None — not in the database yet!
    
    session.add(new_user)           # Put in the shopping cart
    session.commit()                # Checkout — sends INSERT to DB
    session.refresh(new_user)       # Reload from DB to get generated values
    
    print(new_user.id)   # 1 — database assigned the ID!
    print(new_user.is_active)  # True — Python default was applied
```

```
# echo=True shows you the generated SQL:
#
# INSERT INTO users (username, email, bio, is_active)
# VALUES (%(username)s, %(email)s, %(bio)s, %(is_active)s)
# RETURNING users.id
# [parameters: {'username': 'alice', 'email': 'alice@example.com',
#               'bio': 'Backend developer', 'is_active': True}]
```

**Create multiple objects:**

```python
with SessionLocal() as session:
    users = [
        User(username="bob", email="bob@example.com"),
        User(username="charlie", email="charlie@example.com"),
        User(username="diana", email="diana@example.com"),
    ]
    
    session.add_all(users)  # Add a list at once
    session.commit()
```

**READ — Querying objects:**

```python
from sqlalchemy import select
from database import SessionLocal
from models import User

with SessionLocal() as session:
    
    # GET BY PRIMARY KEY — the simplest query
    user = session.get(User, 1)
    # SQL: SELECT users.id, users.username, users.email, ...
    #      FROM users WHERE users.id = 1
    
    if user:
        print(user.username)  # "alice" — access like any Python object!
    
    
    # SELECT with WHERE — for anything beyond primary key
    stmt = select(User).where(User.username == "alice")
    user = session.scalars(stmt).first()
    # SQL: SELECT users.id, users.username, ...
    #      FROM users WHERE users.username = 'alice'
    #      LIMIT 1
    
    
    # SELECT ALL
    stmt = select(User).where(User.is_active == True)
    active_users = session.scalars(stmt).all()
    # SQL: SELECT ... FROM users WHERE users.is_active = true
    # Returns: list[User]
    
    
    # SELECT with ORDER BY and LIMIT
    stmt = (
        select(User)
        .where(User.is_active == True)
        .order_by(User.username)
        .limit(10)
    )
    users = session.scalars(stmt).all()
    # SQL: SELECT ... FROM users
    #      WHERE users.is_active = true
    #      ORDER BY users.username
    #      LIMIT 10
```

**The `select()` + `session.scalars()` pattern:**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE QUERY PATTERN                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Build the statement                                    │
│  ─────────────────────────────                                  │
│  stmt = select(User).where(User.is_active == True)              │
│                                                                 │
│  This builds a SQL statement OBJECT. No SQL is executed yet.    │
│  Think of it as writing a question on paper.                    │
│                                                                 │
│                                                                 │
│  STEP 2: Execute and get results                                │
│  ───────────────────────────────                                │
│  result = session.scalars(stmt)                                 │
│                                                                 │
│  NOW the SQL runs. scalars() returns an iterable of             │
│  model objects (not raw rows).                                  │
│                                                                 │
│                                                                 │
│  STEP 3: Consume results                                        │
│  ────────────────────────                                       │
│  .first()    → First result or None                             │
│  .one()      → Exactly one result (raises if 0 or 2+)          │
│  .one_or_none() → One result or None (raises if 2+)            │
│  .all()      → List of all results                              │
│                                                                 │
│                                                                 │
│  WHY NOT session.query()? That's the OLD SQLAlchemy 1.x style. │
│  select() is the 2.0 way. It's more explicit, more composable, │
│  and works identically in sync and async.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**UPDATE — Modifying objects:**

```python
with SessionLocal() as session:
    # Load the user
    user = session.get(User, 1)
    
    if user:
        # Just change the attribute — like any Python object!
        user.bio = "Senior backend developer"
        user.is_active = False
        
        # The Session TRACKS these changes automatically.
        # It knows user has been modified.
        
        session.commit()
        # SQL: UPDATE users SET bio=%(bio)s, is_active=%(is_active)s
        #      WHERE users.id = %(users_id)s
        #      [parameters: {'bio': 'Senior backend developer',
        #                    'is_active': False, 'users_id': 1}]
```

> "This is the ORM's superpower. You don't write UPDATE statements. You change Python attributes. The Session detects the changes and generates the UPDATE SQL for you."

**DELETE — Removing objects:**

```python
with SessionLocal() as session:
    user = session.get(User, 1)
    
    if user:
        session.delete(user)
        session.commit()
        # SQL: DELETE FROM users WHERE users.id = %(id)s
        #      [parameters: {'id': 1}]
```

---

## 4.4 The Session Lifecycle (add → flush → commit)

**There are distinct stages in the Session's lifecycle. Understanding them prevents confusion.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SESSION LIFECYCLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│   session.add(obj)                                              │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────────────────┐                                       │
│   │  PENDING            │  Object is in the session's cart      │
│   │  (in memory only)   │  but NO SQL has been sent yet.        │
│   └─────────┬───────────┘                                       │
│             │                                                   │
│             │  session.flush()                                   │
│             │  (or auto-flush before queries)                    │
│             ▼                                                    │
│   ┌─────────────────────┐                                       │
│   │  FLUSHED            │  SQL was SENT to the database          │
│   │  (SQL sent, not     │  (INSERT/UPDATE/DELETE executed)       │
│   │   yet permanent)    │  but the TRANSACTION is still open.    │
│   └─────────┬───────────┘  You can still ROLLBACK.              │
│             │                                                   │
│        ┌────┴─────┐                                             │
│        │          │                                             │
│        ▼          ▼                                             │
│   ┌─────────┐  ┌──────────┐                                     │
│   │ COMMIT  │  │ ROLLBACK │                                     │
│   │         │  │          │                                     │
│   │ Changes │  │ Changes  │                                     │
│   │ are     │  │ are      │                                     │
│   │ SAVED   │  │ UNDONE   │                                     │
│   │ forever │  │ as if    │                                     │
│   │         │  │ nothing  │                                     │
│   │         │  │ happened │                                     │
│   └─────────┘  └──────────┘                                     │
│                                                                 │
│                                                                 │
│  KEY INSIGHT:                                                   │
│  flush ≠ commit.                                                │
│  flush sends SQL but keeps the transaction open.                │
│  commit finalizes. This matches COMMIT from Week 5, Lecture 3.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**See it in action:**

```python
with SessionLocal() as session:
    task = Task(title="Learn SQLAlchemy")
    session.add(task)
    
    print(task.id)       # None — not flushed yet, DB hasn't assigned an ID
    
    session.flush()      # SQL sent: INSERT INTO tasks (...) RETURNING id
    print(task.id)       # 1 — DB assigned the ID, but transaction is open!
    
    # At this point:
    # - The INSERT has been sent to PostgreSQL
    # - PostgreSQL has reserved the row
    # - But the transaction is NOT committed
    # - Another database connection CANNOT see this row yet
    #   (remember isolation levels from Week 5, Lecture 3?)
    
    session.commit()     # Transaction committed. Row is permanent.
    # Now every other connection can see this row.
```

**Auto-flush — SQLAlchemy's convenience:**

```python
with SessionLocal() as session:
    task = Task(title="Learn SQLAlchemy")
    session.add(task)
    
    # We query WITHOUT manually flushing first.
    # SQLAlchemy auto-flushes before queries so results are consistent.
    stmt = select(Task).where(Task.title == "Learn SQLAlchemy")
    found = session.scalars(stmt).first()
    
    print(found is task)  # True — same object, found via auto-flush + query
    
    session.commit()
```

**`session.refresh()` — get database-generated values:**

```python
with SessionLocal() as session:
    task = Task(title="Learn SQLAlchemy")
    session.add(task)
    session.commit()          # Row is saved
    
    print(task.created_at)    # ⚠️ May be None! (if server_default was used)
    
    session.refresh(task)     # Re-loads ALL columns from the database
    print(task.created_at)    # 2025-03-15 14:30:00 — now we have it
```

> "After `commit()`, SQLAlchemy expires all loaded attributes to avoid stale data. If you need to read database-generated values like `created_at` or auto-increment `id`, call `session.refresh(obj)` to reload them from the database."

---

## 4.5 Error Handling with Sessions (Connection to Week 1)

**Connection to what you've learned:**

> "Remember custom exceptions and error handling from Week 1, Lecture 2? Database operations can fail too — duplicate keys, constraint violations, connection errors. The patterns are the same."

```python
from sqlalchemy.exc import IntegrityError
from database import SessionLocal
from models import User

def create_user(username: str, email: str) -> User:
    with SessionLocal() as session:
        try:
            user = User(username=username, email=email)
            session.add(user)
            session.commit()
            session.refresh(user)
            return user
        
        except IntegrityError as e:
            session.rollback()  # Undo the failed INSERT
            
            # Determine WHICH constraint failed
            error_msg = str(e.orig)
            
            if "users_username_key" in error_msg:
                raise ValueError(f"Username '{username}' already taken")
            elif "users_email_key" in error_msg:
                raise ValueError(f"Email '{email}' already registered")
            else:
                raise  # Unknown constraint — re-raise (Week 1 pattern!)
```

**Common SQLAlchemy exceptions:**

```
┌─────────────────────────────────────────────────────────────────┐
│                SQLALCHEMY EXCEPTIONS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  from sqlalchemy.exc import ...                                 │
│                                                                 │
│  IntegrityError                                                 │
│  ├─ Unique constraint violated (duplicate username/email)       │
│  ├─ Foreign key constraint violated (referencing missing row)   │
│  ├─ NOT NULL constraint violated (missing required field)       │
│  └─ Check constraint violated                                   │
│                                                                 │
│  OperationalError                                               │
│  ├─ Database unreachable (connection refused)                   │
│  ├─ Connection timeout                                          │
│  └─ Database server crashed                                     │
│                                                                 │
│  NoResultFound                                                  │
│  └─ .one() called but query returned zero rows                  │
│                                                                 │
│  MultipleResultsFound                                           │
│  └─ .one() called but query returned more than one row          │
│                                                                 │
│                                                                 │
│  HIERARCHY (Week 1 Lecture 2 — exception hierarchies!):         │
│                                                                 │
│  SQLAlchemyError                                                │
│  ├─ StatementError                                              │
│  │   └─ IntegrityError                                          │
│  │   └─ OperationalError                                        │
│  │   └─ ProgrammingError                                        │
│  └─ NoResultFound                                               │
│  └─ MultipleResultsFound                                        │
│                                                                 │
│  Catch specific first, generic last. Same rule as Week 1.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The `session.begin()` shortcut — auto-commit / auto-rollback:**

```python
# Instead of manual try/except/rollback:
with SessionLocal() as session:
    with session.begin():
        user = User(username="alice", email="alice@example.com")
        session.add(user)
    # If no exception → auto-committed
    # If exception → auto-rolled back, then re-raised
```

```
┌─────────────────────────────────────────────────────────────────┐
│              session.begin() PATTERN                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MANUAL APPROACH:                                               │
│  ─────────────────                                              │
│  with SessionLocal() as session:                                │
│      try:                                                       │
│          session.add(user)                                      │
│          session.commit()                                       │
│      except Exception:                                          │
│          session.rollback()                                     │
│          raise                                                  │
│                                                                 │
│                                                                 │
│  session.begin() APPROACH:                                      │
│  ──────────────────────────                                     │
│  with SessionLocal() as session:                                │
│      with session.begin():                                      │
│          session.add(user)                                      │
│      # commit or rollback handled automatically                 │
│                                                                 │
│                                                                 │
│  SAME behavior, less boilerplate.                               │
│  The nested `with` is ANOTHER context manager — Week 1!         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: CREATING TABLES & LOOKING AHEAD

## 5.1 metadata.create_all() — Development Only

**You've defined models. Now you need actual tables in PostgreSQL.**

```python
# create_tables.py — Run ONCE during development setup
from database import engine, Base
from models import User, Task  # Import all models so Base knows about them!

# Create all tables that Base knows about
Base.metadata.create_all(bind=engine)
```

```
# echo=True output:

# CREATE TABLE users (
#     id SERIAL NOT NULL,
#     username VARCHAR(50) NOT NULL,
#     email VARCHAR(255) NOT NULL,
#     bio TEXT,
#     is_active BOOLEAN NOT NULL,
#     PRIMARY KEY (id),
#     UNIQUE (username),
#     UNIQUE (email)
# )

# CREATE TABLE tasks (
#     id SERIAL NOT NULL,
#     slug VARCHAR(200) NOT NULL,
#     title VARCHAR(200) NOT NULL,
#     description TEXT,
#     is_completed BOOLEAN NOT NULL,
#     created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT now() NOT NULL,
#     external_id UUID NOT NULL,
#     PRIMARY KEY (id),
#     UNIQUE (slug)
# )
# CREATE INDEX ix_tasks_external_id ON tasks (external_id)
```

> "Look at that generated SQL. It's the exact CREATE TABLE syntax from Week 5. The only difference: you didn't write it. SQLAlchemy read your model definitions and wrote it for you."

**What `metadata.create_all()` does and does NOT do:**

```
┌─────────────────────────────────────────────────────────────────┐
│              metadata.create_all() BEHAVIOR                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ DOES:                                                       │
│  ├─ Creates tables that DON'T EXIST yet                         │
│  ├─ Creates indexes and constraints                             │
│  └─ Is safe to call multiple times (skips existing tables)      │
│                                                                 │
│  ❌ DOES NOT:                                                   │
│  ├─ Modify existing tables (add columns, change types)          │
│  ├─ Drop and recreate tables                                    │
│  ├─ Track schema changes over time                              │
│  └─ Help you deploy schema changes to production                │
│                                                                 │
│                                                                 │
│  THIS IS WHY IT'S DEVELOPMENT ONLY.                             │
│                                                                 │
│  Changed your model? Added a column?                            │
│  create_all() WON'T update the existing table.                  │
│  You'd have to DROP and recreate — losing all data!             │
│                                                                 │
│  Solution: ALEMBIC (Lecture 3 this week).                       │
│  Alembic tracks changes and generates migration scripts         │
│  that ALTER existing tables safely. Production-ready.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Inspecting Generated SQL (echo=True)

**`echo=True` is your X-ray vision into the ORM. Let's use it deliberately.**

```python
# Every operation shows its SQL. Let's trace a full workflow:
from database import SessionLocal
from models import User

with SessionLocal() as session:
    # 1. CREATE
    user = User(username="alice", email="alice@example.com")
    session.add(user)
    session.commit()
```
```
# echo output:
# BEGIN (implicit)
# INSERT INTO users (username, email, bio, is_active)
#   VALUES (%(username)s, %(email)s, %(bio)s, %(is_active)s)
#   RETURNING users.id
# [generated in 0.00023s] {'username': 'alice', 'email': 'alice@example.com',
#                           'bio': None, 'is_active': True}
# COMMIT
```

```python
    # 2. READ
    session.refresh(user)
```
```
# echo output:
# BEGIN (implicit)
# SELECT users.id, users.username, users.email, users.bio, users.is_active
#   FROM users
#   WHERE users.id = %(pk_1)s
# [generated in 0.00018s] {'pk_1': 1}
```

```python
    # 3. UPDATE
    user.bio = "Python developer"
    session.commit()
```
```
# echo output:
# UPDATE users SET bio=%(bio)s WHERE users.id = %(users_id)s
# [generated in 0.00015s] {'bio': 'Python developer', 'users_id': 1}
# COMMIT
```

```python
    # 4. DELETE
    session.delete(user)
    session.commit()
```
```
# echo output:
# DELETE FROM users WHERE users.id = %(id)s
# [generated in 0.00012s] {'id': 1}
# COMMIT
```

> "Every line of SQL you see here is SQL you already know from Week 5. The ORM just writes it for you. If the generated SQL ever surprises you, **that's a sign you need to understand what your Python code is actually doing**. Never ignore the echo output while learning."

---

## 5.3 Raw SQL vs ORM: Side-by-Side

**The grand comparison. Same operations, both approaches.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  RAW SQL vs ORM                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ─── CREATE ───                                                 │
│                                                                 │
│  RAW SQL (Week 5):                                              │
│  cursor.execute(                                                │
│      "INSERT INTO users (username, email) VALUES (%s, %s)"      │
│      " RETURNING id",                                           │
│      ("alice", "alice@example.com")                              │
│  )                                                              │
│  row = cursor.fetchone()                                        │
│  user_id = row[0]                                               │
│                                                                 │
│  ORM (Today):                                                   │
│  user = User(username="alice", email="alice@example.com")       │
│  session.add(user)                                              │
│  session.commit()                                               │
│  # user.id is now set                                           │
│                                                                 │
│                                                                 │
│  ─── READ ───                                                   │
│                                                                 │
│  RAW SQL:                                                       │
│  cursor.execute(                                                │
│      "SELECT id, username, email FROM users WHERE id = %s",     │
│      (1,)                                                       │
│  )                                                              │
│  row = cursor.fetchone()                                        │
│  user = {"id": row[0], "username": row[1], "email": row[2]}     │
│                                                                 │
│  ORM:                                                           │
│  user = session.get(User, 1)                                    │
│  # user.username, user.email — real Python object               │
│                                                                 │
│                                                                 │
│  ─── UPDATE ───                                                 │
│                                                                 │
│  RAW SQL:                                                       │
│  cursor.execute(                                                │
│      "UPDATE users SET email = %s WHERE id = %s",               │
│      ("new@example.com", 1)                                     │
│  )                                                              │
│  conn.commit()                                                  │
│                                                                 │
│  ORM:                                                           │
│  user.email = "new@example.com"                                 │
│  session.commit()                                               │
│                                                                 │
│                                                                 │
│  ─── FILTER ───                                                 │
│                                                                 │
│  RAW SQL:                                                       │
│  cursor.execute(                                                │
│      "SELECT id, username FROM users"                           │
│      " WHERE is_active = %s ORDER BY username LIMIT %s",        │
│      (True, 10)                                                 │
│  )                                                              │
│  rows = cursor.fetchall()                                       │
│  users = [{"id": r[0], "username": r[1]} for r in rows]         │
│                                                                 │
│  ORM:                                                           │
│  stmt = (                                                       │
│      select(User)                                               │
│      .where(User.is_active == True)                             │
│      .order_by(User.username)                                   │
│      .limit(10)                                                 │
│  )                                                              │
│  users = session.scalars(stmt).all()                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Why NOT create_all in Production (Alembic Preview)

**This is a critical mindset shift.**

> "In development, you can drop and recreate tables freely — there's no real data to lose. In production, your tables have actual user data. You can't drop a table with 100,000 users just to add a column."

```
┌─────────────────────────────────────────────────────────────────┐
│           DEVELOPMENT vs PRODUCTION SCHEMA CHANGES              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DEVELOPMENT (what we do now):                                  │
│  ──────────────────────────────                                 │
│  1. Change your model                                           │
│  2. Drop all tables: Base.metadata.drop_all(engine)             │
│  3. Recreate: Base.metadata.create_all(engine)                  │
│  4. Re-seed test data                                           │
│  ✅ Fine. No real data.                                         │
│                                                                 │
│                                                                 │
│  PRODUCTION (what you need):                                    │
│  ────────────────────────────                                   │
│  1. Change your model                                           │
│  2. Generate a MIGRATION script that ALTERs the table           │
│     ALTER TABLE users ADD COLUMN phone VARCHAR(20);             │
│  3. Test the migration                                          │
│  4. Apply it in production                                      │
│  5. Rollback if something goes wrong                            │
│  ✅ Safe. Data preserved.                                       │
│                                                                 │
│                                                                 │
│  THE TOOL: Alembic (Lecture 3 this week)                        │
│  Alembic watches your models, detects changes, and generates    │
│  migration scripts automatically. It's git for your database    │
│  schema.                                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              SQLALCHEMY ORM QUICK REFERENCE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SETUP:                                                         │
│      from sqlalchemy import create_engine, String, select       │
│      from sqlalchemy.orm import (                               │
│          DeclarativeBase, Mapped, mapped_column,                │
│          Session, sessionmaker,                                 │
│      )                                                          │
│                                                                 │
│  ENGINE:                                                        │
│      engine = create_engine("postgresql://...", echo=True)      │
│      SessionLocal = sessionmaker(bind=engine)                   │
│                                                                 │
│  BASE:                                                          │
│      class Base(DeclarativeBase):                               │
│          pass                                                   │
│                                                                 │
│  MODEL:                                                         │
│      class User(Base):                                          │
│          __tablename__ = "users"                                │
│          id: Mapped[int] = mapped_column(primary_key=True)      │
│          name: Mapped[str] = mapped_column(String(100))         │
│          bio: Mapped[Optional[str]] = mapped_column(Text)       │
│                                                                 │
│  CREATE:                                                        │
│      session.add(user)                                          │
│      session.commit()                                           │
│                                                                 │
│  READ (by PK):                                                  │
│      user = session.get(User, 1)                                │
│                                                                 │
│  READ (query):                                                  │
│      stmt = select(User).where(User.name == "Alice")            │
│      user = session.scalars(stmt).first()                       │
│                                                                 │
│  UPDATE:                                                        │
│      user.name = "Bob"                                          │
│      session.commit()                                           │
│                                                                 │
│  DELETE:                                                        │
│      session.delete(user)                                       │
│      session.commit()                                           │
│                                                                 │
│  CREATE TABLES (dev only):                                      │
│      Base.metadata.create_all(bind=engine)                      │
│                                                                 │
│  NULLABLE RULE:                                                 │
│      Mapped[str]            → NOT NULL                          │
│      Mapped[Optional[str]]  → nullable                          │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Forgetting session.commit()  → changes never saved      │
│      ❌ Forgetting session.refresh() → generated values missing  │
│      ❌ Forgetting to import models  → create_all misses tables  │
│      ❌ Using create_all in production → use Alembic instead     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ORM = SPEAK PYTHON, LET SQLALCHEMY WRITE THE SQL              │
│                                                                 │
│  You define the mapping once (models).                          │
│  SQLAlchemy translates every operation to SQL.                  │
│  echo=True proves it's the same SQL you learned in Week 5.      │
│                                                                 │
│                                                                 │
│  THREE THINGS TO REMEMBER:                                      │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   ENGINE     │    │   MODEL      │    │   SESSION    │       │
│  │              │    │              │    │              │       │
│  │ Creates and  │    │ Maps Python  │    │ Tracks your  │       │
│  │ pools your   │    │ classes to   │    │ changes and  │       │
│  │ database     │    │ database     │    │ talks to the │       │
│  │ connections  │    │ tables       │    │ database     │       │
│  │              │    │              │    │              │       │
│  │ CREATE ONCE  │    │ DEFINE ONCE  │    │ USE PER      │       │
│  │              │    │              │    │ OPERATION    │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│                                                                 │
│                                                                 │
│  THE TRANSLATOR ANALOGY:                                        │
│  ├─ Engine = The phone line to the database                     │
│  ├─ Models = The vocabulary (your data definitions)             │
│  ├─ Session = One conversation (open, talk, close)              │
│  ├─ commit = "Send my message" (persist changes)                │
│  ├─ rollback = "Unsend everything" (discard changes)            │
│  └─ echo=True = Listen in on the translator's SQL              │
│                                                                 │
│                                                                 │
│  THE SHOPPING CART ANALOGY:                                     │
│  ├─ session.add() = Put item in cart                            │
│  ├─ session.flush() = Items scanned but not paid                │
│  ├─ session.commit() = Pay and leave (changes saved)            │
│  └─ session.rollback() = Abandon cart (nothing happened)        │
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
│  WEEK 6, LECTURE 2: Relationships & Querying                    │
│  └─ Today you mapped single tables. Next: connect them.         │
│     User has many Tasks. Task belongs to a Category.            │
│     Relationships, foreign keys, JOINs in ORM.                  │
│                                                                 │
│  WEEK 6, LECTURE 3: Alembic Migrations                          │
│  └─ Today you used create_all(). Next: the production way.      │
│     Alembic tracks every schema change and generates            │
│     migration scripts. Never drop a table again.                │
│                                                                 │
│  WEEK 6, LECTURE 4: Async SQLAlchemy & FastAPI Integration      │
│  └─ Today you used sync Session. Next: AsyncSession.            │
│     Same concepts, async/await syntax (Week 1, Lecture 3!).     │
│     Plus: wire SQLAlchemy into FastAPI as a dependency.          │
│                                                                 │
│  WEEK 6 PROJECT: Refactor Task Manager to PostgreSQL            │
│  └─ Replace your in-memory dict storage with everything         │
│     you learn this week. Same API, real database.               │
│                                                                 │
│  WEEK 7: Advanced Database Patterns                             │
│  └─ Query optimization with EXPLAIN (Week 5 callback).          │
│     Connection pooling, bulk operations, advanced patterns.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```