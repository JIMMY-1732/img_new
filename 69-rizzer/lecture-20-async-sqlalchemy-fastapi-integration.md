# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROGRESSIVE REFACTORING                                        │
│  ───────────────────────                                        │
│  Start with the mess. Feel the pain. Clean it up step by step.  │
│  Each pattern is motivated by a concrete problem, not dogma.    │
│                                                                 │
│  CONNECT THE DOTS                                               │
│  ────────────────                                               │
│  Students already know BOTH sides — async Python (Week 1) and   │
│  sync SQLAlchemy (Week 6 L1-L3). This lecture is the BRIDGE.    │
│  Every concept calls back to something they've already learned. │
│                                                                 │
│  LAYER BY LAYER                                                 │
│  ─────────────                                                  │
│  async engine → async session → dependency injection →          │
│  repository pattern → Pydantic bridge → testing.                │
│  Each layer solves one specific problem. Never introduce a      │
│  pattern without first showing the pain it removes.             │
│                                                                 │
│  THE LIBRARY ANALOGY                                            │
│  ───────────────────                                            │
│  A library system maps perfectly to our architecture.           │
│  Front desk = FastAPI. Archive = Database. Library card =       │
│  Session. Catalog system = Repository. Catalog cards =          │
│  Pydantic models. Physical books = SQLAlchemy models.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│              ASYNC SQLALCHEMY & FASTAPI INTEGRATION             │
│                      (3.5-4 Hour Lecture)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 The Sync Trap (Demonstration)                           │
│  ├─ 1.2 What Goes Wrong Under Load                              │
│  ├─ 1.3 The Three Problems We Must Solve                        │
│  └─ 1.4 The Library Analogy                                     │
│                                                                 │
│  PART 2: ASYNC ENGINE & SESSION (40 min)                        │
│  ├─ 2.1 asyncpg — The Async PostgreSQL Driver                   │
│  ├─ 2.2 create_async_engine (The New Connection)                │
│  ├─ 2.3 AsyncSession (The Async Conversation)                   │
│  └─ 2.4 Async Query Syntax (What Changes, What Stays)           │
│                                                                 │
│  PART 3: FASTAPI INTEGRATION (50 min)                           │
│  ├─ 3.1 Session Per Request (One Visit, One Card)               │
│  ├─ 3.2 Yield Dependencies for Sessions                         │
│  │       (Connection to Week 3, Lecture 4)                      │
│  ├─ 3.3 Transaction Boundaries                                  │
│  │       (Commit, Rollback, and the Request Lifecycle)          │
│  └─ 3.4 The Complete Wiring (End-to-End)                        │
│                                                                 │
│  PART 4: THE REPOSITORY PATTERN (40 min)                        │
│  ├─ 4.1 SQL in Routes — The Spaghetti Problem                   │
│  ├─ 4.2 What is a Repository?                                   │
│  ├─ 4.3 Building Your First Repository                          │
│  └─ 4.4 Repository as Dependency (Layered Architecture)         │
│                                                                 │
│  PART 5: PYDANTIC + SQLALCHEMY BRIDGE (30 min)                  │
│  ├─ 5.1 Two Model Systems, Two Jobs                             │
│  ├─ 5.2 from_attributes — Reading ORM Objects                   │
│  ├─ 5.3 The Full Data Flow                                      │
│  └─ 5.4 Common Patterns and Pitfalls                            │
│                                                                 │
│  PART 6: TESTING WITH A REAL DATABASE (30 min)                  │
│  ├─ 6.1 Mocks vs Real Database (When Each is Right)             │
│  ├─ 6.2 Test Database Setup                                     │
│  ├─ 6.3 Overriding Dependencies in Tests                        │
│  │       (Connection to Week 4, Lecture 1)                      │
│  └─ 6.4 Transaction Rollback for Test Isolation                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Sync Trap

**Start with code they already know. Make them watch it break.**

The students have been writing sync SQLAlchemy for the past three lectures. They have a FastAPI app from Weeks 3-4. The natural instinct is to combine them directly:

```python
# the_sync_trap.py — Run this with students watching
from fastapi import FastAPI
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session, sessionmaker

from models import Task  # Their model from Week 6 Lecture 1

# Sync engine — what they've been using
engine = create_engine("postgresql://user:pass@localhost/taskdb")
SessionLocal = sessionmaker(bind=engine)

app = FastAPI()


@app.get("/tasks")
async def get_tasks():
    """Looks reasonable. Feels familiar. Is BROKEN."""
    session = SessionLocal()
    try:
        stmt = select(Task).where(Task.completed == False)
        tasks = session.scalars(stmt).all()  # ← SYNC call inside ASYNC route
        return tasks
    finally:
        session.close()
```

**Ask the class:**

> "This works. You can test it, it returns tasks. So what's the problem? Think back to Week 1, Lecture 3. What did we say about blocking calls inside async code?"

**Let them sit with it. Then reveal:**

> "That `session.scalars()` call is SYNCHRONOUS. It talks to PostgreSQL over the network. The network call blocks. And because this is an `async def` route, it runs on the event loop thread. One blocked route = entire event loop frozen."

---

## 1.2 What Goes Wrong Under Load

**Draw the timeline. Make the damage visible.**

```
┌─────────────────────────────────────────────────────────────────┐
│              ONE REQUEST: LOOKS FINE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request 1: ──[route logic]──[DB QUERY 50ms]──[response]──▶    │
│                                                                 │
│  Total: ~50ms. No problem.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              TEN CONCURRENT REQUESTS: DISASTER                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Event loop thread (ONE thread handles ALL async routes):       │
│                                                                 │
│  Req 1: ──[DB BLOCKED 50ms]──▶                                  │
│  Req 2:   😴 waiting...       ──[DB BLOCKED 50ms]──▶            │
│  Req 3:   😴 waiting...          😴 waiting...     ──[DB]──▶    │
│  Req 4:   😴 😴 😴 ...                                          │
│  Req 5:   😴 😴 😴 😴 ...                                       │
│   ...                                                           │
│  Req 10:  😴 😴 😴 😴 😴 😴 😴 😴 😴 ...            ──[DB]──▶  │
│                                                                 │
│  Request 1:  50ms response  ← Acceptable                       │
│  Request 10: 500ms response ← Users notice. Unacceptable.      │
│                                                                 │
│  The sync DB call BLOCKS the event loop.                        │
│  Every other request queues up behind it.                       │
│  Your async server behaves like a sync server. 💀               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now show what SHOULD happen with async:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TEN CONCURRENT REQUESTS: ASYNC DB ✅               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Event loop thread (async DB calls yield control):              │
│                                                                 │
│  Req 1: ──[await DB]─────────────────[response]──▶              │
│  Req 2: ──[await DB]─────────────────[response]──▶              │
│  Req 3: ──[await DB]─────────────────[response]──▶              │
│  Req 4: ──[await DB]─────────────────[response]──▶              │
│  ...                                                            │
│  Req 10: ─[await DB]─────────────────[response]──▶              │
│                                                                 │
│  ALL requests: ~50-60ms response ← Everyone is happy 😊        │
│                                                                 │
│  Each `await` yields control to the event loop.                 │
│  While one request waits for PostgreSQL,                        │
│  the event loop handles other requests.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The callback to Week 1 Lecture 3:**

> "Remember the restaurant analogy? A sync waiter stands at the kitchen window waiting for one order. An async waiter takes more orders while the kitchen cooks. Sync SQLAlchemy is the waiter standing at the window. The event loop — our restaurant manager — can't assign other work because the waiter is STUCK."

---

## 1.3 The Three Problems We Must Solve

**Frame the rest of the lecture. Everything connects to solving one of these three:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THREE PROBLEMS TO SOLVE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM 1: BLOCKING                                            │
│  ────────────────────                                           │
│  Sync SQLAlchemy blocks the event loop.                         │
│  We need an ASYNC engine and ASYNC sessions.                    │
│  → Solved in PART 2                                             │
│                                                                 │
│  PROBLEM 2: SESSION LIFECYCLE                                   │
│  ────────────────────────────                                   │
│  Who creates the session? Who closes it? What happens           │
│  if the route throws an exception mid-query?                    │
│  We need session-per-request with automatic cleanup.            │
│  → Solved in PART 3                                             │
│                                                                 │
│  PROBLEM 3: SPAGHETTI                                           │
│  ────────────────────                                           │
│  SQL queries mixed into route handlers = unreadable,            │
│  untestable, unreusable code. We need SEPARATION.               │
│  → Solved in PARTS 4 & 5                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 The Library Analogy

**This analogy will carry us through the rest of the lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE LIBRARY ANALOGY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine a public library:                                      │
│                                                                 │
│  FRONT DESK (FastAPI routes)                                    │
│  ─────────────────────────────                                  │
│  Patrons walk in with requests:                                 │
│  "I need all books by Author X" = GET /tasks?author=X           │
│  "I want to donate a book"      = POST /tasks                   │
│                                                                 │
│  LIBRARY CARD (AsyncSession)                                    │
│  ─────────────────────────────                                  │
│  Each patron gets a card when they enter.                       │
│  The card grants access to the archive for THIS visit only.     │
│  When they leave, the card is returned. Always.                 │
│  Even if the fire alarm goes off. (Exception = fire alarm.)     │
│                                                                 │
│  THE ARCHIVE (PostgreSQL)                                       │
│  ─────────────────────────────                                  │
│  Where the physical books live.                                 │
│  Only accessible through proper channels, not directly.         │
│                                                                 │
│  CATALOG SYSTEM (Repository)                                    │
│  ─────────────────────────────                                  │
│  The organized system for finding and shelving books.            │
│  The front desk clerk doesn't wander the archive —              │
│  they ask the catalog system.                                   │
│                                                                 │
│  CATALOG CARDS (Pydantic models)                                │
│  ─────────────────────────────                                  │
│  What patrons see: title, author, summary.                      │
│  NOT the physical book with its shelf code, condition           │
│  notes, acquisition date, and internal tracking number.         │
│                                                                 │
│  PHYSICAL BOOKS (SQLAlchemy models)                             │
│  ─────────────────────────────                                  │
│  The full internal representation.                              │
│  Has everything: metadata, location, relationships              │
│  to other books, internal IDs. Patrons never see this.          │
│                                                                 │
│  MULTI-BOOK CHECKOUT (Transaction)                              │
│  ─────────────────────────────                                  │
│  Checking out 3 books = one transaction.                        │
│  If the system finds book #3 is damaged → ALL three              │
│  go back on the shelf. All succeed or all fail.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map it formally:**

```
Library                     │  Async SQLAlchemy + FastAPI
────────────────────────────│──────────────────────────────
Front desk                  │  FastAPI route handlers
Patron walking in           │  HTTP request arriving
Library card for this visit │  AsyncSession (per request)
Archive / vault             │  PostgreSQL database
Catalog system              │  Repository class
Catalog card                │  Pydantic response model
  (title, author, summary)  │    (selected fields, formatted)
Physical book               │  SQLAlchemy ORM model
  (full metadata, location) │    (all columns, relationships)
Multi-book checkout         │  Database transaction
  (all or nothing)          │    (commit or rollback)
Returning your card at exit │  session.close() in finally
```

---

# PART 2: ASYNC ENGINE & SESSION

## 2.1 asyncpg — The Async PostgreSQL Driver

**In Week 5, Lecture 2, you ran PostgreSQL with Docker. You connected with a SYNC driver. Now we need an ASYNC one.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  DATABASE DRIVERS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SYNC DRIVER (what you've been using):                          │
│  ─────────────────────────────────────                          │
│  psycopg2 (or psycopg)                                          │
│  ├─ Talks to PostgreSQL synchronously                           │
│  ├─ Blocks the thread while waiting for response                │
│  ├─ Connection string: postgresql://user:pass@host/db           │
│  └─ Fine for scripts, CLI tools, sync frameworks                │
│                                                                 │
│  ASYNC DRIVER (what we need now):                               │
│  ─────────────────────────────────                              │
│  asyncpg                                                        │
│  ├─ Talks to PostgreSQL asynchronously                          │
│  ├─ Yields control while waiting (event loop runs)              │
│  ├─ Connection string: postgresql+asyncpg://user:pass@host/db   │
│  └─ Built for async frameworks like FastAPI                     │
│                                                                 │
│                                                                 │
│  The ONLY difference SQLAlchemy needs to know:                  │
│  the connection string prefix changes.                          │
│                                                                 │
│  postgresql://          →  postgresql+asyncpg://                 │
│  ▲ sync driver              ▲ async driver                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Install it:**

```bash
pip install asyncpg
# or if using uv (from Week 1 Lecture 4):
uv add asyncpg
```

**In the library analogy:**

> "Switching from psycopg2 to asyncpg is like upgrading from a paper card catalog to a digital one. Same archive, same books — but now the front desk clerk doesn't have to walk to the catalog room and wait. They tap a screen and get notified when the result is ready."

---

## 2.2 create_async_engine (The New Connection)

**Side-by-side comparison. Spot the differences:**

```python
# ❌ SYNC (what you've been doing in Lectures 1-3)
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql://user:pass@localhost:5432/taskdb",
    echo=True,  # Log SQL statements
)
```

```python
# ✅ ASYNC (what we're switching to)
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost:5432/taskdb",
    echo=True,  # Log SQL statements — still works!
)
```

**What changed:**

```
┌─────────────────────────────────────────────────────────────────┐
│              SYNC → ASYNC: ENGINE CHANGES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Import path:                                                │
│     from sqlalchemy import create_engine                        │
│     ──▶                                                         │
│     from sqlalchemy.ext.asyncio import create_async_engine      │
│                                                                 │
│  2. Function name:                                              │
│     create_engine(...)                                          │
│     ──▶                                                         │
│     create_async_engine(...)                                    │
│                                                                 │
│  3. Connection string:                                          │
│     "postgresql://..."                                          │
│     ──▶                                                         │
│     "postgresql+asyncpg://..."                                  │
│                                                                 │
│  That's it. Everything else is the same.                        │
│  echo, pool_size, max_overflow — all the same parameters.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 AsyncSession (The Async Conversation)

**The session is your "conversation" with the database. In Lectures 1-2, you used sync sessions:**

```python
# ❌ SYNC sessions (Lectures 1-2)
from sqlalchemy.orm import Session, sessionmaker

SessionLocal = sessionmaker(bind=engine)

# Using it:
with Session(engine) as session:
    stmt = select(Task)
    tasks = session.scalars(stmt).all()
    # session.close() called automatically by context manager
```

**Now, the async equivalent:**

```python
# ✅ ASYNC sessions
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
)

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/taskdb")

async_session_factory = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,  # ← CRITICAL. We'll explain why.
)

# Using it:
async with async_session_factory() as session:
    stmt = select(Task)
    result = await session.execute(stmt)  # ← await!
    tasks = result.scalars().all()
    # session.close() called automatically by async context manager
```

**Connection to Week 1, Lecture 2 (Context Managers):**

> "Remember context managers? `with` ensures cleanup even if exceptions happen. `async with` is the same thing, but for async code. The session is a RESOURCE — it holds a database connection. The async context manager guarantees that connection is returned to the pool, even if your code crashes mid-query."

```
┌─────────────────────────────────────────────────────────────────┐
│              SYNC → ASYNC: SESSION CHANGES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IMPORT:                                                        │
│    sessionmaker          →  async_sessionmaker                  │
│    Session               →  AsyncSession                        │
│                                                                 │
│  CONTEXT MANAGER:                                               │
│    with session:         →  async with session:                  │
│                                                                 │
│  QUERIES:                                                       │
│    session.execute(...)  →  await session.execute(...)           │
│    session.commit()      →  await session.commit()              │
│    session.rollback()    →  await session.rollback()            │
│    session.refresh(obj)  →  await session.refresh(obj)          │
│    session.close()       →  await session.close()               │
│                                                                 │
│  UNCHANGED:                                                     │
│    select(), where(), order_by()  — Same!                       │
│    session.add(), session.delete() — Same! (not awaited)        │
│    Model definitions — Same!                                    │
│    Alembic migrations — Same!                                   │
│                                                                 │
│  Pattern: anything that talks to the database = await           │
│           anything that just modifies local state = no await    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**About `expire_on_commit=False` — this is CRITICAL:**

```python
async_session_factory = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,  # ← What does this do?
)
```

```
┌─────────────────────────────────────────────────────────────────┐
│              expire_on_commit=False                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DEFAULT BEHAVIOR (expire_on_commit=True):                      │
│  ──────────────────────────────────────────                     │
│  After session.commit(), SQLAlchemy "expires" all loaded        │
│  objects. Next time you access an attribute (like task.title),  │
│  it triggers a LAZY LOAD to refresh from the database.          │
│                                                                 │
│  In SYNC code: This works silently. The lazy load blocks,       │
│  fetches fresh data, and returns it.                            │
│                                                                 │
│  In ASYNC code: 💥 CRASH. Lazy loading requires a SYNC          │
│  database call. Async sessions cannot make sync calls.          │
│  You get: MissingGreenlet error.                                │
│                                                                 │
│  THE FIX: expire_on_commit=False                                │
│  ──────────────────────────────                                 │
│  Tells SQLAlchemy: "After commit, keep the data I already       │
│  loaded. Don't expire it. Don't try to lazy-load."              │
│                                                                 │
│  This is SAFE because:                                          │
│  • Each request gets its own session (we'll set this up)        │
│  • The session lives only for one request                       │
│  • No other request can modify "our" loaded objects             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Library analogy:**

> "Imagine you check out a book, the system logs it (commit), and then immediately FORGETS the book's title — forcing you to walk back to the archive just to read the cover. That's `expire_on_commit=True` in async: the system 'forgets' after committing and then crashes when it tries to silently re-read. `expire_on_commit=False` says: 'You already carried the book to the front desk. Just read the cover from what you're holding.'"

---

## 2.4 Async Query Syntax (What Changes, What Stays)

**Side-by-side. The query-building is identical. The execution changes.**

```python
from sqlalchemy import select
from models import Task

# ─────────────────────────────────────────────
# BUILDING QUERIES: Exactly the same as sync!
# ─────────────────────────────────────────────

# Select all
stmt = select(Task)

# With filter
stmt = select(Task).where(Task.completed == False)

# With ordering
stmt = select(Task).where(Task.completed == False).order_by(Task.created_at.desc())

# With limit
stmt = select(Task).limit(10).offset(20)

# Nothing above talks to the database.
# These just BUILD a statement object. No await needed.
```

```python
# ─────────────────────────────────────────────
# EXECUTING QUERIES: This is what changes.
# ─────────────────────────────────────────────

# SYNC (what you did in Lectures 1-2):
tasks = session.scalars(stmt).all()

# ASYNC (what we do now):
result = await session.execute(stmt)  # ← AWAIT here
tasks = result.scalars().all()         # ← No await here (local operation)
```

**Why the two-step in async?**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE TWO-STEP QUERY PATTERN                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Execute (talks to database → must await)               │
│  ─────────────────────────────────────────────────              │
│  result = await session.execute(stmt)                           │
│                                                                 │
│  This sends SQL to PostgreSQL, waits for the response,          │
│  and returns a Result object holding all the raw rows.          │
│                                                                 │
│  STEP 2: Extract (local memory operation → no await)            │
│  ─────────────────────────────────────────────────              │
│  tasks = result.scalars().all()                                 │
│                                                                 │
│  This reads from the Result object already in memory.           │
│  No database call. Just Python processing rows into objects.    │
│                                                                 │
│                                                                 │
│  Rule: await when talking to PostgreSQL.                        │
│        Don't await when processing data already in Python.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Common extraction methods (same as sync, just on the Result object):**

```python
# Get all rows as ORM objects
result = await session.execute(select(Task))
tasks = result.scalars().all()            # list[Task]

# Get first row or None
result = await session.execute(select(Task).where(Task.id == 1))
task = result.scalar_one_or_none()        # Task | None

# Get exactly one row (raises if 0 or 2+)
result = await session.execute(select(Task).where(Task.id == 1))
task = result.scalar_one()                # Task (or raises)

# Get as tuples (when selecting multiple columns)
result = await session.execute(select(Task.title, Task.completed))
rows = result.all()                       # list[tuple[str, bool]]
```

**Full CRUD example — async style:**

```python
from sqlalchemy import select, delete
from sqlalchemy.ext.asyncio import AsyncSession

from models import Task


# CREATE
async def create_task(session: AsyncSession, title: str, description: str | None = None) -> Task:
    task = Task(title=title, description=description)
    session.add(task)               # No await — just stages the object locally
    await session.commit()          # Await — sends INSERT to PostgreSQL
    await session.refresh(task)     # Await — reloads to get DB-generated fields (id, created_at)
    return task


# READ (one)
async def get_task(session: AsyncSession, task_id: int) -> Task | None:
    stmt = select(Task).where(Task.id == task_id)
    result = await session.execute(stmt)
    return result.scalar_one_or_none()


# READ (many)
async def get_tasks(session: AsyncSession, completed: bool | None = None) -> list[Task]:
    stmt = select(Task)
    if completed is not None:
        stmt = stmt.where(Task.completed == completed)
    stmt = stmt.order_by(Task.created_at.desc())
    result = await session.execute(stmt)
    return list(result.scalars().all())


# UPDATE
async def update_task(session: AsyncSession, task: Task, title: str) -> Task:
    task.title = title              # No await — modify local Python object
    session.add(task)               # No await — stage it
    await session.commit()          # Await — sends UPDATE to PostgreSQL
    await session.refresh(task)     # Await — reload fresh state
    return task


# DELETE
async def delete_task(session: AsyncSession, task: Task) -> None:
    await session.delete(task)      # Await — stages AND marks for deletion
    await session.commit()          # Await — sends DELETE to PostgreSQL
```

**Ask the class:**

> "Look at every line. Which ones have `await` and which don't? What's the pattern?"

Answer: **If it sends bytes to PostgreSQL → `await`. If it only touches Python memory → no `await`.**

```
┌─────────────────────────────────────────────────────────────────┐
│              AWAIT OR NOT? THE DECISION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NEEDS await (I/O with database):                               │
│  ├─ session.execute(stmt)        Sends SQL, waits for result   │
│  ├─ session.commit()             Writes changes to disk        │
│  ├─ session.rollback()           Undoes pending changes        │
│  ├─ session.refresh(obj)         Re-reads row from DB          │
│  ├─ session.delete(obj)          Marks + may trigger DB check  │
│  └─ session.close()              Returns connection to pool    │
│                                                                 │
│  NO await (local Python operations):                            │
│  ├─ session.add(obj)             Stages locally in memory      │
│  ├─ session.add_all([...])       Stages multiple locally       │
│  ├─ select(Task).where(...)      Builds SQL statement object   │
│  ├─ result.scalars().all()       Extracts from loaded result   │
│  └─ task.title = "new"           Modifies Python attribute     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: FASTAPI INTEGRATION

## 3.1 Session Per Request (One Visit, One Card)

**The fundamental rule of database sessions in web applications:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE SESSION-PER-REQUEST PATTERN                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RULE: Each HTTP request gets its OWN database session.         │
│        The session is CREATED when the request starts.          │
│        The session is CLOSED when the request ends.             │
│        No exceptions. No sharing.                               │
│                                                                 │
│                                                                 │
│  Request 1:  ──[ Session A ]────────────▶ close A              │
│  Request 2:  ────[ Session B ]──────────▶ close B              │
│  Request 3:  ──────[ Session C ]────────▶ close C              │
│                                                                 │
│  Each session is ISOLATED. Request 1's uncommitted changes      │
│  are invisible to Request 2. This is transaction isolation      │
│  from Week 5 Lecture 3.                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Library analogy:**

> "Every patron gets their own library card when they walk in. They don't share cards. When they leave — whether they checked out books successfully or got kicked out for being loud — the card is returned. If two patrons are both looking for the last copy of a book, the card system handles the conflict, not the front desk."

**Why NOT share a session across requests?**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY NOT SHARE SESSIONS? ❌                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SHARED SESSION (never do this):                                │
│                                                                 │
│  global_session = async_session_factory()  ← ONE session        │
│                                                                 │
│  Request 1: global_session.add(task_A)                          │
│  Request 2: global_session.add(task_B)                          │
│  Request 1: global_session.commit()  ← COMMITS BOTH!           │
│  Request 2: global_session.rollback() ← ROLLS BACK BOTH!       │
│                                                                 │
│  Result: Data corruption. Race conditions. Nightmares.          │
│                                                                 │
│                                                                 │
│  PER-REQUEST SESSION (correct):                                 │
│                                                                 │
│  Request 1: session_A.add(task_A)                               │
│  Request 2: session_B.add(task_B)                               │
│  Request 1: session_A.commit()  ← Only commits task_A          │
│  Request 2: session_B.rollback() ← Only rolls back task_B      │
│                                                                 │
│  Result: Each request is independent. Safe.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Yield Dependencies for Sessions (Connection to Week 3, Lecture 4)

**This is where everything you've learned converges.**

> "In Week 3, Lecture 4, you learned about FastAPI's `Depends()` and yield dependencies. We said yield dependencies are like context managers — they run setup code, yield a resource, and run cleanup code. In Week 1, Lecture 2, you learned context managers themselves — `with` statements for resource cleanup. NOW you see why we taught those things."

**The connection map:**

```
┌─────────────────────────────────────────────────────────────────┐
│              EVERYTHING CONNECTS HERE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Week 1 L2: Context managers                                    │
│       └─▶ "Resources need cleanup. with guarantees it."         │
│                                                                 │
│  Week 1 L3: Async / await                                       │
│       └─▶ "Use async with for async resources."                 │
│                                                                 │
│  Week 3 L4: Depends() and yield dependencies                    │
│       └─▶ "Yield = setup/teardown in FastAPI dependencies."     │
│                                                                 │
│  Week 6 L1: SQLAlchemy Session lifecycle                        │
│       └─▶ "Sessions must be opened and closed properly."        │
│                                                                 │
│             ┌─────────────────────────────┐                     │
│  ALL FOUR ─▶│ async yield dependency that │                     │
│             │ provides an AsyncSession    │                     │
│             └─────────────────────────────┘                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Build it step by step.**

First, recall yield dependencies from Week 3 Lecture 4:

```python
# Week 3, Lecture 4 — you wrote something like this:
from typing import Generator

def get_db_session() -> Generator[FakeDB, None, None]:
    db = FakeDB()       # Setup
    try:
        yield db        # Provide to route
    finally:
        db.close()      # Cleanup (always runs)
```

**Now, the REAL version with AsyncSession:**

```python
# database.py — The database configuration module
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
)
from typing import AsyncGenerator

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/taskdb"

engine = create_async_engine(DATABASE_URL, echo=True)

async_session_factory = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


async def get_async_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session           # ← Provide session to the route
            await session.commit()  # ← If no exception: commit
        except Exception:
            await session.rollback()  # ← If exception: rollback
            raise                     # ← Re-raise (Week 1 L2: re-raising!)
        # session.close() is called automatically by async with
```

**Break it down — what happens when a request arrives:**

```
┌─────────────────────────────────────────────────────────────────┐
│           LIFECYCLE OF get_async_session                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REQUEST ARRIVES                                                │
│       │                                                         │
│       ▼                                                         │
│  async with async_session_factory() as session:                 │
│       │  ← Session CREATED, DB connection borrowed from pool    │
│       │                                                         │
│       ▼                                                         │
│  yield session  ─────────▶  Route handler runs                  │
│       │                     │  Uses session for queries         │
│       │                     │  session.add(), execute(), etc.   │
│       │                     │                                   │
│       │  ◀──────────────────┘  Route handler returns            │
│       │                                                         │
│       ├─── No exception? ──▶ await session.commit()  ✅         │
│       │                                                         │
│       └─── Exception? ────▶ await session.rollback() ↩️         │
│                              raise (re-raise to FastAPI)        │
│       │                                                         │
│       ▼                                                         │
│  async with exits: session.close()                              │
│       │  ← DB connection returned to pool. Always.              │
│       │                                                         │
│       ▼                                                         │
│  RESPONSE SENT                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Library analogy:**

> "The yield dependency IS the library check-in/check-out system. Patron arrives → issue a card (create session). Patron does their business (route runs). Patron leaves normally → stamp the card as 'completed visit' (commit). Fire alarm goes off → put all books back on shelves (rollback). Either way, the card is returned (close)."

---

## 3.3 Transaction Boundaries (Commit, Rollback, and the Request Lifecycle)

**When does a transaction start? When does it end?**

```
┌─────────────────────────────────────────────────────────────────┐
│              TRANSACTION LIFECYCLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TRANSACTION STARTS:                                            │
│  When the session first talks to the database.                  │
│  (Not when the session is created — when the first              │
│  execute() or add()+flush() happens.)                           │
│                                                                 │
│  TRANSACTION ENDS:                                              │
│  When you call commit() or rollback().                          │
│                                                                 │
│  In our yield dependency:                                       │
│                                                                 │
│  Session created ─┐                                             │
│                   │  (no transaction yet)                       │
│  First query ─────┤                                             │
│                   │  ← Transaction begins                       │
│  More queries ────┤                                             │
│                   │  ← Same transaction                         │
│  Route returns ───┤                                             │
│                   │                                             │
│  commit() ────────┘  ← Transaction ends (changes saved)        │
│          OR                                                     │
│  rollback() ──────┘  ← Transaction ends (changes discarded)    │
│                                                                 │
│  Session closed ─────  ← Connection returned to pool           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to Week 5, Lecture 3 (Transactions):**

> "In Week 5, you learned `BEGIN`, `COMMIT`, `ROLLBACK` in raw SQL. SQLAlchemy does the same thing — it just manages those SQL commands for you automatically. Our yield dependency is the `BEGIN...COMMIT/ROLLBACK` wrapper."

**Why commit in the dependency, not in the route?**

```python
# ❌ FRAGILE: Committing inside each route
@app.post("/tasks")
async def create_task(
    body: TaskCreate,
    session: AsyncSession = Depends(get_async_session),
):
    task = Task(title=body.title)
    session.add(task)
    await session.commit()    # What if the next line fails?
    await session.refresh(task)
    return task               # If THIS fails, commit already happened!

# ✅ SAFE: Dependency handles commit/rollback
@app.post("/tasks")
async def create_task(
    body: TaskCreate,
    session: AsyncSession = Depends(get_async_session),
):
    task = Task(title=body.title)
    session.add(task)
    await session.flush()     # Send to DB but don't commit yet
    await session.refresh(task)
    return task
    # Dependency commits AFTER the route returns successfully.
    # If ANYTHING in the route raises, dependency rolls back.
```

**The difference between flush and commit:**

```
┌─────────────────────────────────────────────────────────────────┐
│              FLUSH vs COMMIT                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  session.flush():                                               │
│  ├─ Sends SQL to database (INSERT, UPDATE, etc.)                │
│  ├─ DB assigns IDs, checks constraints                          │
│  ├─ Changes are visible within THIS session/transaction         │
│  ├─ But NOT yet permanent — can still rollback                  │
│  └─ Think: "Write in pencil"                                    │
│                                                                 │
│  session.commit():                                              │
│  ├─ Calls flush() first (if not already flushed)                │
│  ├─ Then tells PostgreSQL: "Make it permanent"                  │
│  ├─ Changes now visible to other sessions/requests              │
│  ├─ Cannot be undone (except with new DELETE/UPDATE)            │
│  └─ Think: "Write in ink"                                       │
│                                                                 │
│  In our pattern:                                                │
│  Routes use flush() to get IDs and validate constraints.        │
│  The dependency uses commit() after the route succeeds.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 The Complete Wiring (End-to-End)

**Let's see the full picture — from HTTP request to database and back:**

```python
# database.py
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
)
from typing import AsyncGenerator

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/taskdb"

engine = create_async_engine(DATABASE_URL, echo=True)

async_session_factory = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


async def get_async_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

```python
# models.py — UNCHANGED from Lectures 1-2!
from sqlalchemy import String, Text, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from datetime import datetime


class Base(DeclarativeBase):
    pass


class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text, default=None)
    completed: Mapped[bool] = mapped_column(default=False)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    category_id: Mapped[int | None] = mapped_column(
        ForeignKey("categories.id"), default=None
    )

    category: Mapped["Category | None"] = relationship(back_populates="tasks")


class Category(Base):
    __tablename__ = "categories"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True)

    tasks: Mapped[list["Task"]] = relationship(back_populates="category")
```

```python
# routes.py — Simple first version (we'll improve in Part 4)
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from database import get_async_session
from models import Task

app = FastAPI()


@app.get("/tasks")
async def list_tasks(
    completed: bool | None = None,
    session: AsyncSession = Depends(get_async_session),
):
    stmt = select(Task)
    if completed is not None:
        stmt = stmt.where(Task.completed == completed)
    stmt = stmt.order_by(Task.created_at.desc())

    result = await session.execute(stmt)
    tasks = result.scalars().all()
    return tasks


@app.get("/tasks/{task_id}")
async def get_task(
    task_id: int,
    session: AsyncSession = Depends(get_async_session),
):
    stmt = select(Task).where(Task.id == task_id)
    result = await session.execute(stmt)
    task = result.scalar_one_or_none()

    if task is None:
        raise HTTPException(status_code=404, detail="Task not found")

    return task
```

**Trace a request through the system:**

```
┌─────────────────────────────────────────────────────────────────┐
│            REQUEST: GET /tasks?completed=false                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. HTTP request arrives at FastAPI                             │
│     └─ uvicorn receives, passes to app                          │
│                                                                 │
│  2. FastAPI sees Depends(get_async_session)                     │
│     └─ Calls get_async_session()                                │
│     └─ Session created, connection borrowed from pool           │
│     └─ yield session → session injected into route              │
│                                                                 │
│  3. Route handler runs                                          │
│     └─ stmt = select(Task).where(completed == False)            │
│     └─ result = await session.execute(stmt)                     │
│         └─ SQL sent to PostgreSQL over async connection         │
│         └─ Event loop free to handle other requests! ✅         │
│         └─ PostgreSQL responds with rows                        │
│     └─ tasks = result.scalars().all()                           │
│     └─ return tasks                                             │
│                                                                 │
│  4. Route returned successfully (no exception)                  │
│     └─ get_async_session resumes after yield                    │
│     └─ await session.commit()                                   │
│     └─ session closed, connection returned to pool              │
│                                                                 │
│  5. FastAPI serializes tasks to JSON, sends response            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**This works. But look at the route handler again:**

```python
@app.get("/tasks")
async def list_tasks(
    completed: bool | None = None,
    session: AsyncSession = Depends(get_async_session),
):
    stmt = select(Task)                              # ← SQL building
    if completed is not None:                        #
        stmt = stmt.where(Task.completed == completed)  # ← SQL building
    stmt = stmt.order_by(Task.created_at.desc())     # ← SQL building
    result = await session.execute(stmt)             # ← SQL execution
    tasks = result.scalars().all()                   # ← SQL extraction
    return tasks
```

> "Five of six lines are database logic. The route handler IS the database query. This is the spaghetti problem — and we solve it in Part 4."

---

# PART 4: THE REPOSITORY PATTERN

## 4.1 SQL in Routes — The Spaghetti Problem

**Why is SQL in routes bad? Three reasons:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE SPAGHETTI PROBLEM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM 1: DUPLICATION                                         │
│  ─────────────────────                                          │
│  "Get task by ID" appears in GET /tasks/{id},                   │
│  in PUT /tasks/{id}, in DELETE /tasks/{id}.                     │
│  Same query, copy-pasted three times.                           │
│  Change the query? Change it in three places.                   │
│  Miss one? Bug.                                                 │
│                                                                 │
│  PROBLEM 2: UNTESTABLE                                          │
│  ─────────────────────                                          │
│  Want to test your "get incomplete tasks" logic?                │
│  You must spin up a FastAPI TestClient, make HTTP requests,     │
│  parse JSON responses. You're testing HTTP + DB together.       │
│  You can't test the query ALONE.                                │
│                                                                 │
│  PROBLEM 3: UNREADABLE                                          │
│  ─────────────────────                                          │
│  A route handler should say WHAT it does, not HOW.              │
│  "Get incomplete tasks" should read like that —                 │
│  not like 6 lines of SQLAlchemy query building.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Concrete example of the duplication problem:**

```python
# ❌ SQL duplicated across routes
@app.get("/tasks/{task_id}")
async def get_task(task_id: int, session: AsyncSession = Depends(get_async_session)):
    stmt = select(Task).where(Task.id == task_id)
    result = await session.execute(stmt)
    task = result.scalar_one_or_none()
    if task is None:
        raise HTTPException(status_code=404, detail="Task not found")
    return task

@app.put("/tasks/{task_id}")
async def update_task(task_id: int, body: TaskUpdate, session: AsyncSession = Depends(get_async_session)):
    stmt = select(Task).where(Task.id == task_id)     # ← SAME query
    result = await session.execute(stmt)               # ← SAME execution
    task = result.scalar_one_or_none()                 # ← SAME extraction
    if task is None:                                   # ← SAME check
        raise HTTPException(status_code=404, detail="Task not found")  # ← SAME error
    # ... update logic

@app.delete("/tasks/{task_id}")
async def delete_task(task_id: int, session: AsyncSession = Depends(get_async_session)):
    stmt = select(Task).where(Task.id == task_id)     # ← SAME query AGAIN
    result = await session.execute(stmt)               # ← SAME execution AGAIN
    task = result.scalar_one_or_none()                 # ← SAME extraction AGAIN
    if task is None:                                   # ← SAME check AGAIN
        raise HTTPException(status_code=404, detail="Task not found")
    # ... delete logic
```

> "Three routes. Same five lines copy-pasted three times. You'll need 'get task by ID' in at least five more places before this project is done. This doesn't scale."

---

## 4.2 What is a Repository?

```
┌─────────────────────────────────────────────────────────────────┐
│              THE REPOSITORY PATTERN                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A REPOSITORY is a class that encapsulates all database         │
│  operations for a specific entity (e.g., Task).                 │
│                                                                 │
│  It provides a CLEAN INTERFACE:                                 │
│  ├─ get_by_id(id)          → Task or None                      │
│  ├─ get_all(filters)       → list of Tasks                     │
│  ├─ create(data)           → new Task                          │
│  ├─ update(task, data)     → updated Task                      │
│  └─ delete(task)           → None                              │
│                                                                 │
│  It HIDES the implementation:                                   │
│  ├─ The route doesn't know about select(), execute()            │
│  ├─ The route doesn't know about session.add(), flush()         │
│  └─ The route doesn't care if it's SQLAlchemy or raw SQL        │
│                                                                 │
│                                                                 │
│  ANALOGY:                                                       │
│  ──────────                                                     │
│  Library front desk clerk → "I need the book with ISBN X"       │
│  Catalog system handles  → finding the shelf, locating the      │
│                            book, checking availability           │
│  Clerk doesn't enter the archive. Clerk doesn't know shelf      │
│  codes. Clerk asks the catalog system (repository).             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visually:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WITHOUT REPOSITORY                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐       ┌─────────────┐       ┌────────────┐        │
│  │ FastAPI  │──────▶│ SQLAlchemy  │──────▶│ PostgreSQL │        │
│  │ Route    │       │ queries     │       │            │        │
│  │          │       │ (scattered  │       │            │        │
│  │          │       │  in route)  │       │            │        │
│  └─────────┘       └─────────────┘       └────────────┘        │
│                                                                 │
│  Route knows HOW to talk to the database. 😰                    │
│  SQL everywhere. Mixed concerns.                                │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│              WITH REPOSITORY                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐    ┌────────────┐    ┌───────────┐  ┌──────────┐  │
│  │ FastAPI  │───▶│ Repository │───▶│SQLAlchemy │─▶│PostgreSQL│  │
│  │ Route    │    │            │    │ queries   │  │          │  │
│  │          │    │ get_by_id  │    │ (all in   │  │          │  │
│  │          │    │ create     │    │  one place)│  │          │  │
│  │          │    │ update     │    │           │  │          │  │
│  └─────────┘    └────────────┘    └───────────┘  └──────────┘  │
│                                                                 │
│  Route knows WHAT it wants. Repository knows HOW. 😊            │
│  Clean separation. Each layer does one job.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Building Your First Repository

```python
# repositories/task_repository.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from models import Task


class TaskRepository:
    """Encapsulates all database operations for Tasks."""

    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def get_by_id(self, task_id: int) -> Task | None:
        """Fetch a single task by ID."""
        stmt = select(Task).where(Task.id == task_id)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_all(
        self,
        completed: bool | None = None,
        limit: int = 50,
        offset: int = 0,
    ) -> list[Task]:
        """Fetch tasks with optional filtering and pagination."""
        stmt = select(Task)

        if completed is not None:
            stmt = stmt.where(Task.completed == completed)

        stmt = stmt.order_by(Task.created_at.desc())
        stmt = stmt.limit(limit).offset(offset)

        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def get_by_id_with_category(self, task_id: int) -> Task | None:
        """Fetch task with its category eagerly loaded."""
        stmt = (
            select(Task)
            .where(Task.id == task_id)
            .options(selectinload(Task.category))  # Eager load!
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def create(self, title: str, description: str | None = None,
                     category_id: int | None = None) -> Task:
        """Create a new task and return it with DB-generated fields."""
        task = Task(
            title=title,
            description=description,
            category_id=category_id,
        )
        self.session.add(task)
        await self.session.flush()      # Send INSERT, get ID
        await self.session.refresh(task)  # Reload full object
        return task

    async def update(self, task: Task, **kwargs: object) -> Task:
        """Update specific fields on a task."""
        for key, value in kwargs.items():
            if hasattr(task, key):
                setattr(task, key, value)
        self.session.add(task)
        await self.session.flush()
        await self.session.refresh(task)
        return task

    async def delete(self, task: Task) -> None:
        """Delete a task."""
        await self.session.delete(task)
        await self.session.flush()
```

**Connection to Week 6, Lecture 2 (Eager vs Lazy Loading, N+1):**

> "Notice `get_by_id_with_category` uses `selectinload`. Remember the N+1 problem from Lecture 2? In async, this is even MORE critical because lazy loading is COMPLETELY BROKEN by default. You MUST eagerly load any relationship you plan to access."

```
┌─────────────────────────────────────────────────────────────────┐
│              LAZY LOADING IN ASYNC: BROKEN ❌                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  # In SYNC SQLAlchemy, this works silently:                     │
│  task = session.get(Task, 1)                                    │
│  print(task.category.name)   # Lazy load: silent DB query       │
│                                                                 │
│  # In ASYNC SQLAlchemy, this CRASHES:                           │
│  task = await session.get(Task, 1)                              │
│  print(task.category.name)   # 💥 MissingGreenlet error!       │
│                                                                 │
│  WHY: Lazy loading triggers a SYNC database call.               │
│  AsyncSession cannot make sync calls. No escape.                │
│                                                                 │
│  THE FIX: Always use eager loading for relationships            │
│  you need:                                                      │
│                                                                 │
│  stmt = select(Task).options(selectinload(Task.category))       │
│  result = await session.execute(stmt)                           │
│  task = result.scalar_one()                                     │
│  print(task.category.name)   # ✅ Already loaded. No lazy hit. │
│                                                                 │
│  This is not a limitation — it's a FEATURE.                     │
│  It forces you to be explicit about what data you need.         │
│  No more accidental N+1 queries hiding behind lazy loads.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Repository as Dependency (Layered Architecture)

**Make the repository injectable using FastAPI's `Depends`:**

```python
# dependencies.py
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from database import get_async_session
from repositories.task_repository import TaskRepository


async def get_task_repository(
    session: AsyncSession = Depends(get_async_session),
) -> TaskRepository:
    return TaskRepository(session)
```

**Now look at the routes — the transformation is dramatic:**

```python
# routes.py — CLEAN version with repository
from fastapi import FastAPI, Depends, HTTPException

from dependencies import get_task_repository
from repositories.task_repository import TaskRepository
from schemas import TaskCreate, TaskUpdate, TaskResponse

app = FastAPI()


@app.get("/tasks", response_model=list[TaskResponse])
async def list_tasks(
    completed: bool | None = None,
    repo: TaskRepository = Depends(get_task_repository),
):
    return await repo.get_all(completed=completed)


@app.get("/tasks/{task_id}", response_model=TaskResponse)
async def get_task(
    task_id: int,
    repo: TaskRepository = Depends(get_task_repository),
):
    task = await repo.get_by_id(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="Task not found")
    return task


@app.post("/tasks", response_model=TaskResponse, status_code=201)
async def create_task(
    body: TaskCreate,
    repo: TaskRepository = Depends(get_task_repository),
):
    return await repo.create(
        title=body.title,
        description=body.description,
        category_id=body.category_id,
    )


@app.put("/tasks/{task_id}", response_model=TaskResponse)
async def update_task(
    task_id: int,
    body: TaskUpdate,
    repo: TaskRepository = Depends(get_task_repository),
):
    task = await repo.get_by_id(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="Task not found")
    return await repo.update(task, **body.model_dump(exclude_unset=True))


@app.delete("/tasks/{task_id}", status_code=204)
async def delete_task(
    task_id: int,
    repo: TaskRepository = Depends(get_task_repository),
):
    task = await repo.get_by_id(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="Task not found")
    await repo.delete(task)
```

**Compare the before and after:**

```
┌─────────────────────────────────────────────────────────────────┐
│              BEFORE vs AFTER                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE (SQL in routes):                                        │
│  ───────────────────────                                        │
│  @app.get("/tasks")                                             │
│  async def list_tasks(                                          │
│      completed: bool | None = None,                             │
│      session: AsyncSession = Depends(get_async_session),        │
│  ):                                                             │
│      stmt = select(Task)                                        │
│      if completed is not None:                                  │
│          stmt = stmt.where(Task.completed == completed)         │
│      stmt = stmt.order_by(Task.created_at.desc())               │
│      result = await session.execute(stmt)                       │
│      tasks = result.scalars().all()                             │
│      return tasks                                               │
│                                                                 │
│  AFTER (repository):                                            │
│  ────────────────────                                           │
│  @app.get("/tasks")                                             │
│  async def list_tasks(                                          │
│      completed: bool | None = None,                             │
│      repo: TaskRepository = Depends(get_task_repository),       │
│  ):                                                             │
│      return await repo.get_all(completed=completed)             │
│                                                                 │
│  The route says WHAT. The repository knows HOW.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The dependency chain:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DEPENDENCY CHAIN                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FastAPI resolves dependencies from bottom up:                  │
│                                                                 │
│  Route: list_tasks(repo=Depends(get_task_repository))           │
│                           │                                     │
│                           ▼                                     │
│  get_task_repository(session=Depends(get_async_session))        │
│                               │                                 │
│                               ▼                                 │
│  get_async_session()  →  creates AsyncSession                   │
│                               │                                 │
│                               ▼                                 │
│  TaskRepository(session)  →  wraps session in clean interface   │
│                               │                                 │
│                               ▼                                 │
│  Route receives repository  →  calls repo.get_all()             │
│                                                                 │
│  After route returns:                                           │
│  get_async_session resumes → commit or rollback → close         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: PYDANTIC + SQLALCHEMY BRIDGE

## 5.1 Two Model Systems, Two Jobs

**Students often ask: "Why do we have TWO kinds of models?"**

```
┌─────────────────────────────────────────────────────────────────┐
│              TWO MODELS, TWO PURPOSES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SQLALCHEMY MODEL (Physical book in the archive)                │
│  ───────────────────────────────────────────────                │
│  Purpose: Represent a DATABASE ROW                              │
│  Lives in: models.py                                            │
│  Maps to: A table in PostgreSQL                                 │
│  Has: ALL columns, relationships, DB constraints                │
│  Used by: Repository, migrations, internal logic                │
│                                                                 │
│  class Task(Base):                                              │
│      id, title, description, completed, created_at,             │
│      updated_at, category_id, category (relationship)           │
│                                                                 │
│                                                                 │
│  PYDANTIC MODEL (Catalog card for patrons)                      │
│  ───────────────────────────────────────────────                │
│  Purpose: Validate and shape HTTP DATA                          │
│  Lives in: schemas.py                                           │
│  Maps to: JSON in request/response                              │
│  Has: Only the fields the client needs to see or send           │
│  Used by: Routes, API documentation, serialization              │
│                                                                 │
│  class TaskResponse(BaseModel):                                 │
│      id, title, description, completed, created_at              │
│      (no updated_at, no category_id — only what clients need)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why separate them?**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY NOT ONE MODEL FOR EVERYTHING?                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Can't I just return the SQLAlchemy model directly?"           │
│                                                                 │
│  You CAN, but you SHOULDN'T:                                    │
│                                                                 │
│  1. SECURITY                                                    │
│     Your DB model might have: password_hash, internal_notes,    │
│     admin_flags. Do you want those in API responses?            │
│                                                                 │
│  2. STABILITY                                                   │
│     Adding a DB column shouldn't change your API contract.      │
│     If clients parse your JSON, adding a field could            │
│     break their code.                                           │
│                                                                 │
│  3. VALIDATION                                                  │
│     Pydantic validates INCOMING data with constraints.          │
│     SQLAlchemy validates against DATABASE constraints.          │
│     Different concerns. Different rules.                        │
│                                                                 │
│  4. FLEXIBILITY                                                 │
│     Sometimes the API response combines data from multiple      │
│     tables. That doesn't map to any single DB model.            │
│                                                                 │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  SQLAlchemy models face the DATABASE.                           │
│  Pydantic models face the CLIENT.                               │
│  The repository is the translator between them.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 from_attributes — Reading ORM Objects

**Connection to Week 3, Lecture 3 (Pydantic Deep Dive):**

> "In Week 3, we briefly mentioned `from_attributes` in Pydantic's `model_config`. NOW is when you need it. It tells Pydantic: 'You're going to receive a SQLAlchemy ORM object, not a dictionary. Read attributes directly.'"

```python
# schemas.py
from pydantic import BaseModel, Field, ConfigDict
from datetime import datetime


# ─────────────────────────────────────────────
# REQUEST MODELS (what the client SENDS)
# ─────────────────────────────────────────────

class TaskCreate(BaseModel):
    """For POST /tasks — creating a new task."""
    title: str = Field(..., min_length=1, max_length=200)
    description: str | None = None
    category_id: int | None = None
    # No id, no created_at — the DATABASE generates those.


class TaskUpdate(BaseModel):
    """For PUT /tasks/{id} — updating an existing task."""
    title: str | None = Field(None, min_length=1, max_length=200)
    description: str | None = None
    completed: bool | None = None
    category_id: int | None = None
    # All fields optional — client sends only what they want to change.


# ─────────────────────────────────────────────
# RESPONSE MODELS (what the client RECEIVES)
# ─────────────────────────────────────────────

class TaskResponse(BaseModel):
    """For all task responses — reading task data."""
    model_config = ConfigDict(from_attributes=True)  # ← THE KEY LINE

    id: int
    title: str
    description: str | None
    completed: bool
    created_at: datetime
    # No category_id exposed — but we could include category details:


class TaskDetailResponse(TaskResponse):
    """For GET /tasks/{id} — includes related data."""
    category_name: str | None = None
    # We'll handle this custom mapping below
```

**How `from_attributes=True` works:**

```
┌─────────────────────────────────────────────────────────────────┐
│              from_attributes=True                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT from_attributes (default):                             │
│  Pydantic expects a dict:                                       │
│     TaskResponse(**{"id": 1, "title": "Buy milk", ...})         │
│                                                                 │
│  WITH from_attributes=True:                                     │
│  Pydantic can read from ANY object with matching attributes:    │
│     task = <SQLAlchemy Task object>                              │
│     TaskResponse.model_validate(task)                           │
│         │                                                       │
│         ├── task.id          → id: 1                            │
│         ├── task.title       → title: "Buy milk"                │
│         ├── task.description → description: null                │
│         ├── task.completed   → completed: false                 │
│         └── task.created_at  → created_at: "2025-..."           │
│                                                                 │
│  Pydantic reads attributes using getattr().                     │
│  It looks for task.id, task.title, etc.                         │
│  SQLAlchemy models have these as Python attributes.             │
│  It just works.                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In practice, with FastAPI's `response_model`:**

```python
@app.get("/tasks/{task_id}", response_model=TaskResponse)
async def get_task(
    task_id: int,
    repo: TaskRepository = Depends(get_task_repository),
):
    task = await repo.get_by_id(task_id)  # Returns SQLAlchemy Task
    if task is None:
        raise HTTPException(status_code=404, detail="Task not found")
    return task  # FastAPI sees response_model=TaskResponse
                 # Calls TaskResponse.model_validate(task)
                 # from_attributes=True lets it read task.id, task.title, etc.
                 # Serializes to JSON. Only the fields in TaskResponse appear.
```

> "You return a SQLAlchemy object, but the client sees a Pydantic-shaped JSON response. `from_attributes` is the translator. The physical book goes back to the archive; the patron walks out with the catalog card."

---

## 5.3 The Full Data Flow

**Trace the complete lifecycle. This is the full picture of everything connected:**

```
┌─────────────────────────────────────────────────────────────────┐
│          THE FULL DATA FLOW: POST /tasks                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CLIENT sends JSON:                                             │
│  {"title": "Buy milk", "description": "2% milk"}               │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────────────────┐                                    │
│  │ FastAPI parses body     │                                    │
│  │ Validates with Pydantic │                                    │
│  │ → TaskCreate object     │                                    │
│  └────────────┬────────────┘                                    │
│               │  body.title = "Buy milk"                        │
│               │  body.description = "2% milk"                   │
│               ▼                                                 │
│  ┌─────────────────────────┐                                    │
│  │ Route handler           │                                    │
│  │ repo.create(            │                                    │
│  │   title=body.title,     │                                    │
│  │   description=body.desc │                                    │
│  │ )                       │                                    │
│  └────────────┬────────────┘                                    │
│               │                                                 │
│               ▼                                                 │
│  ┌─────────────────────────┐                                    │
│  │ Repository.create()     │                                    │
│  │ task = Task(...)        │  ← Creates SQLAlchemy model        │
│  │ session.add(task)       │  ← Stages for INSERT               │
│  │ await session.flush()   │  ← Sends INSERT to PostgreSQL     │
│  │ await session.refresh() │  ← Gets back id, created_at       │
│  │ return task             │  ← Returns SQLAlchemy object       │
│  └────────────┬────────────┘                                    │
│               │  task.id = 42                                   │
│               │  task.title = "Buy milk"                        │
│               │  task.created_at = 2025-...                     │
│               ▼                                                 │
│  ┌─────────────────────────┐                                    │
│  │ Route returns task      │                                    │
│  │                         │                                    │
│  │ FastAPI sees            │                                    │
│  │ response_model=         │                                    │
│  │   TaskResponse          │                                    │
│  │                         │                                    │
│  │ Calls model_validate()  │  ← Pydantic reads ORM attributes  │
│  │ from_attributes=True    │                                    │
│  │ Serializes to JSON      │                                    │
│  └────────────┬────────────┘                                    │
│               │                                                 │
│               ▼                                                 │
│  ┌─────────────────────────┐                                    │
│  │ Yield dependency        │                                    │
│  │ resumes after route     │                                    │
│  │ → No exception          │                                    │
│  │ → await session.commit()│  ← Makes INSERT permanent         │
│  │ → session closes        │  ← Connection back to pool        │
│  └────────────┬────────────┘                                    │
│               │                                                 │
│               ▼                                                 │
│  CLIENT receives JSON:                                          │
│  {"id": 42, "title": "Buy milk", "description": "2% milk",     │
│   "completed": false, "created_at": "2025-..."}                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Common Patterns and Pitfalls

**Pattern: Nested response models for related data**

```python
# When you want to include category info in the task response:

class CategoryResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    name: str


class TaskWithCategoryResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    description: str | None
    completed: bool
    created_at: datetime
    category: CategoryResponse | None  # ← Nested Pydantic model


# In the route:
@app.get("/tasks/{task_id}", response_model=TaskWithCategoryResponse)
async def get_task_detail(
    task_id: int,
    repo: TaskRepository = Depends(get_task_repository),
):
    # MUST use eager loading — otherwise category is not loaded!
    task = await repo.get_by_id_with_category(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="Task not found")
    return task
    # Pydantic reads task.category (a Category ORM object)
    # Converts it to CategoryResponse using from_attributes
    # Nesting works automatically!
```

**Pitfall: Forgetting to eager load before nesting**

```python
# ❌ CRASH: category not loaded, Pydantic tries to access it,
#           SQLAlchemy attempts lazy load → MissingGreenlet!
@app.get("/tasks/{task_id}", response_model=TaskWithCategoryResponse)
async def get_task_detail(task_id: int, repo: TaskRepository = Depends(...)):
    task = await repo.get_by_id(task_id)  # ← No eager loading!
    return task  # 💥 When Pydantic reads task.category → crash

# ✅ CORRECT: use the method that eagerly loads the relationship
@app.get("/tasks/{task_id}", response_model=TaskWithCategoryResponse)
async def get_task_detail(task_id: int, repo: TaskRepository = Depends(...)):
    task = await repo.get_by_id_with_category(task_id)  # ← selectinload!
    return task  # ✅ task.category already loaded
```

---

# PART 6: TESTING WITH A REAL DATABASE

## 6.1 Mocks vs Real Database (When Each is Right)

**Connection to Week 2, Lecture 2 (Testing Fundamentals) and Week 4, Lecture 1 (Testing FastAPI):**

> "In Week 2, you learned about mocks and stubs. In Week 4, you used `dependency_overrides` to mock dependencies in FastAPI tests. Those skills still apply. But now we add a new tool: testing against a REAL database."

```
┌─────────────────────────────────────────────────────────────────┐
│              MOCKS vs REAL DATABASE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MOCKED DATABASE (unit tests):                                  │
│  ──────────────────────────────                                 │
│  ✅ Fast (no DB connection needed)                               │
│  ✅ Tests route logic in isolation                               │
│  ❌ Doesn't test SQL queries                                    │
│  ❌ Doesn't catch constraint violations                         │
│  ❌ Can pass even when query is wrong                           │
│                                                                 │
│  Use for: Testing route logic, HTTP status codes,               │
│           error handling, input validation.                     │
│                                                                 │
│                                                                 │
│  REAL TEST DATABASE (integration tests):                        │
│  ──────────────────────────────                                 │
│  ✅ Tests the ACTUAL queries against PostgreSQL                  │
│  ✅ Catches constraint violations, type mismatches               │
│  ✅ Verifies relationships, indexes, migrations                  │
│  ❌ Slower (DB round-trips)                                     │
│  ❌ Requires PostgreSQL running                                 │
│                                                                 │
│  Use for: Testing repository methods, data integrity,           │
│           full request → DB → response flows.                   │
│                                                                 │
│                                                                 │
│  YOUR PROJECT NEEDS BOTH.                                       │
│  Week 3-4's mocked tests → still useful for route logic.        │
│  Week 6 onward → add integration tests with real DB.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 Test Database Setup

**Never test against your development database. Use a separate one.**

```python
# tests/conftest.py
import asyncio
from typing import AsyncGenerator

import pytest
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
)

from models import Base
from database import get_async_session
from main import app  # Your FastAPI app

# ─────────────────────────────────────────────
# TEST DATABASE: Separate from development!
# ─────────────────────────────────────────────
TEST_DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/taskdb_test"
#                                                                       ^^^^
#                                                            DIFFERENT database!

test_engine = create_async_engine(TEST_DATABASE_URL, echo=True)

test_session_factory = async_sessionmaker(
    bind=test_engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


# ─────────────────────────────────────────────
# CREATE/DROP TABLES FOR TEST SESSION
# ─────────────────────────────────────────────
@pytest_asyncio.fixture(scope="session")
async def setup_database():
    """Create all tables before tests, drop after all tests."""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await test_engine.dispose()
```

---

## 6.3 Overriding Dependencies in Tests (Connection to Week 4, Lecture 1)

> "Remember `dependency_overrides` from Week 4? We used it to replace real dependencies with fake ones in tests. Now we use the same mechanism to swap the REAL database session with our TEST database session."

```python
# tests/conftest.py (continued)

# ─────────────────────────────────────────────
# PER-TEST SESSION WITH ROLLBACK
# ─────────────────────────────────────────────
@pytest_asyncio.fixture
async def db_session(setup_database) -> AsyncGenerator[AsyncSession, None]:
    """Provide a transactional session that rolls back after each test."""
    async with test_session_factory() as session:
        async with session.begin():   # Start transaction
            yield session
            await session.rollback()  # Rollback after test — clean slate!


# ─────────────────────────────────────────────
# OVERRIDE FastAPI's session dependency
# ─────────────────────────────────────────────
@pytest_asyncio.fixture
async def client(db_session: AsyncSession) -> AsyncGenerator[AsyncClient, None]:
    """Test client with overridden database session."""

    async def override_get_async_session() -> AsyncGenerator[AsyncSession, None]:
        yield db_session  # Use the TEST session, not the real one

    # Swap the dependency — Week 4, Lecture 1 pattern!
    app.dependency_overrides[get_async_session] = override_get_async_session

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac

    # Clean up override
    app.dependency_overrides.clear()
```

---

## 6.4 Transaction Rollback for Test Isolation

**Why rollback? So each test starts with a clean database.**

```
┌─────────────────────────────────────────────────────────────────┐
│              TRANSACTION ROLLBACK FOR ISOLATION                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT rollback:                                              │
│  ──────────────────                                             │
│  Test 1: Creates task "Buy milk"    → DB has 1 task            │
│  Test 2: Expects empty DB           → 💥 FAILS! "Buy milk"     │
│          still there                   is still there.          │
│  Test 3: Creates task "Walk dog"    → DB has 2 tasks           │
│  Test 4: Counts tasks, expects 1    → 💥 FAILS! Gets 2.       │
│                                                                 │
│  Tests depend on ORDER. Fragile. Flaky. Nightmare.              │
│                                                                 │
│                                                                 │
│  WITH rollback:                                                 │
│  ───────────────                                                │
│  Test 1: BEGIN → Create "Buy milk" → assert → ROLLBACK          │
│  Test 2: BEGIN → (DB is empty ✅)  → assert → ROLLBACK          │
│  Test 3: BEGIN → Create "Walk dog" → assert → ROLLBACK          │
│  Test 4: BEGIN → Create 1 task → count → 1 ✅ → ROLLBACK       │
│                                                                 │
│  Each test sees a CLEAN database. Order doesn't matter.         │
│  Tests are INDEPENDENT. Reliable. Fast.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Writing actual tests:**

```python
# tests/test_tasks.py
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_create_task(client: AsyncClient):
    """POST /tasks should create and return a task."""
    response = await client.post(
        "/tasks",
        json={"title": "Buy milk", "description": "2% milk"},
    )
    assert response.status_code == 201

    data = response.json()
    assert data["title"] == "Buy milk"
    assert data["description"] == "2% milk"
    assert data["completed"] is False
    assert "id" in data             # DB generated
    assert "created_at" in data     # DB generated


@pytest.mark.asyncio
async def test_create_task_validation(client: AsyncClient):
    """POST /tasks with invalid data should return 422."""
    response = await client.post(
        "/tasks",
        json={"title": ""},  # min_length=1 violated
    )
    assert response.status_code == 422


@pytest.mark.asyncio
async def test_get_task_not_found(client: AsyncClient):
    """GET /tasks/999 should return 404."""
    response = await client.get("/tasks/999")
    assert response.status_code == 404


@pytest.mark.asyncio
async def test_list_tasks_empty(client: AsyncClient):
    """GET /tasks on empty DB should return empty list."""
    response = await client.get("/tasks")
    assert response.status_code == 200
    assert response.json() == []   # Clean DB — rollback ensures this


@pytest.mark.asyncio
async def test_create_and_retrieve(client: AsyncClient):
    """Full cycle: create a task, then fetch it by ID."""
    # Create
    create_response = await client.post(
        "/tasks",
        json={"title": "Walk the dog"},
    )
    task_id = create_response.json()["id"]

    # Retrieve
    get_response = await client.get(f"/tasks/{task_id}")
    assert get_response.status_code == 200
    assert get_response.json()["title"] == "Walk the dog"
```

**Testing the repository directly (unit-ish test, with real DB):**

```python
# tests/test_repositories.py
import pytest
from sqlalchemy.ext.asyncio import AsyncSession

from repositories.task_repository import TaskRepository


@pytest.mark.asyncio
async def test_repo_create(db_session: AsyncSession):
    """Test repository create without HTTP layer."""
    repo = TaskRepository(db_session)
    task = await repo.create(title="Test task")

    assert task.id is not None
    assert task.title == "Test task"
    assert task.completed is False


@pytest.mark.asyncio
async def test_repo_get_by_id(db_session: AsyncSession):
    """Test repository get_by_id."""
    repo = TaskRepository(db_session)
    created = await repo.create(title="Find me")

    found = await repo.get_by_id(created.id)
    assert found is not None
    assert found.title == "Find me"


@pytest.mark.asyncio
async def test_repo_get_by_id_not_found(db_session: AsyncSession):
    """Test repository get_by_id returns None for missing ID."""
    repo = TaskRepository(db_session)
    result = await repo.get_by_id(9999)
    assert result is None
```

> "Notice: these tests hit a REAL PostgreSQL database. They test that your SQL queries actually work. The rollback fixture makes them fast and isolated. This is the integration test layer of the testing pyramid from Week 2, Lecture 2."

---

## Common Mistakes and Misconceptions

### Mistake 1: Using a sync driver with async engine

```python
# ❌ WRONG: psycopg2 is sync — it blocks the event loop!
engine = create_async_engine("postgresql://user:pass@localhost/db")
# Might not even error immediately — but will crash on first query.

# ✅ CORRECT: asyncpg is async
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
```

---

### Mistake 2: Accessing a lazy-loaded relationship

```python
# ❌ CRASH: MissingGreenlet error
async def get_task_with_category(session: AsyncSession, task_id: int):
    result = await session.execute(select(Task).where(Task.id == task_id))
    task = result.scalar_one()
    print(task.category.name)  # 💥 Lazy load attempt in async!

# ✅ CORRECT: Eagerly load what you need
async def get_task_with_category(session: AsyncSession, task_id: int):
    stmt = select(Task).where(Task.id == task_id).options(selectinload(Task.category))
    result = await session.execute(stmt)
    task = result.scalar_one()
    print(task.category.name)  # ✅ Already loaded
```

---

### Mistake 3: Forgetting expire_on_commit=False

```python
# ❌ DEFAULT: expire_on_commit=True
session_factory = async_sessionmaker(bind=engine, class_=AsyncSession)

async with session_factory() as session:
    task = Task(title="Test")
    session.add(task)
    await session.commit()
    print(task.title)  # 💥 Attribute expired, triggers lazy load → crash!

# ✅ CORRECT: Keep loaded data after commit
session_factory = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,  # ← Prevents this crash
)
```

---

### Mistake 4: Sharing a session between requests

```python
# ❌ NEVER: Global session
session = async_session_factory()  # Created once at startup

@app.get("/tasks")
async def get_tasks():
    result = await session.execute(select(Task))  # ALL requests share this!
    # Race conditions, data leaking between requests, chaos.

# ✅ CORRECT: Session per request via Depends
@app.get("/tasks")
async def get_tasks(session: AsyncSession = Depends(get_async_session)):
    result = await session.execute(select(Task))  # This request's own session
```

---

### Mistake 5: Committing inside the route AND the dependency

```python
# ❌ DOUBLE COMMIT: Confusing, can cause subtle bugs
@app.post("/tasks")
async def create_task(body: TaskCreate, session: AsyncSession = Depends(get_async_session)):
    task = Task(title=body.title)
    session.add(task)
    await session.commit()  # ← Commits here
    return task
    # THEN the dependency ALSO calls session.commit()
    # Double commit: wastes a round-trip, can mask errors

# ✅ CORRECT: Let the dependency handle commit
@app.post("/tasks")
async def create_task(body: TaskCreate, session: AsyncSession = Depends(get_async_session)):
    task = Task(title=body.title)
    session.add(task)
    await session.flush()      # ← Sends SQL but doesn't commit
    await session.refresh(task) # ← Gets generated fields
    return task
    # Dependency commits after route returns successfully
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              ASYNC SQLALCHEMY + FASTAPI QUICK REFERENCE         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ENGINE SETUP:                                                  │
│      from sqlalchemy.ext.asyncio import create_async_engine     │
│      engine = create_async_engine(                              │
│          "postgresql+asyncpg://user:pass@host/db"               │
│      )                                                          │
│                                                                 │
│  SESSION FACTORY:                                               │
│      from sqlalchemy.ext.asyncio import (                       │
│          AsyncSession, async_sessionmaker,                      │
│      )                                                          │
│      factory = async_sessionmaker(                              │
│          bind=engine,                                           │
│          class_=AsyncSession,                                   │
│          expire_on_commit=False,    ← ALWAYS set this           │
│      )                                                          │
│                                                                 │
│  DEPENDENCY (session per request):                              │
│      async def get_async_session():                             │
│          async with factory() as session:                       │
│              try:                                               │
│                  yield session                                  │
│                  await session.commit()                         │
│              except Exception:                                  │
│                  await session.rollback()                       │
│                  raise                                          │
│                                                                 │
│  QUERY PATTERN:                                                 │
│      result = await session.execute(stmt)   ← AWAIT             │
│      items = result.scalars().all()         ← NO await          │
│                                                                 │
│  WRITE PATTERN:                                                 │
│      session.add(obj)            ← NO await (local staging)     │
│      await session.flush()       ← AWAIT (sends SQL)            │
│      await session.refresh(obj)  ← AWAIT (reloads from DB)      │
│                                                                 │
│  RELATIONSHIPS:                                                 │
│      Always use selectinload() or joinedload() ← No lazy loads! │
│                                                                 │
│  PYDANTIC BRIDGE:                                               │
│      model_config = ConfigDict(from_attributes=True)            │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ postgresql:// with async  → Use postgresql+asyncpg://   │
│      ❌ task.category (lazy)      → Use selectinload()          │
│      ❌ expire_on_commit=True     → Set to False                │
│      ❌ Shared global session     → Use Depends() per request   │
│      ❌ Commit in route + dep     → Flush in route, commit in   │
│                                     dependency only             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ASYNC SQLALCHEMY + FASTAPI = LAYERED ARCHITECTURE              │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐       │
│  │                    CLIENT                            │       │
│  │              (sends JSON, receives JSON)              │       │
│  └────────────────────────┬─────────────────────────────┘       │
│                           │  HTTP Request / Response             │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────┐       │
│  │               FASTAPI ROUTE (routes.py)              │       │
│  │  Pydantic validates input, returns shaped response   │       │
│  │  Knows WHAT to do. Doesn't know HOW (DB details).   │       │
│  └────────────────────────┬─────────────────────────────┘       │
│                           │  Depends()                          │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────┐       │
│  │            REPOSITORY (repositories/*.py)            │       │
│  │  Encapsulates all SQLAlchemy queries.                │       │
│  │  Knows HOW to talk to the database.                  │       │
│  └────────────────────────┬─────────────────────────────┘       │
│                           │  AsyncSession                       │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────┐       │
│  │          SQLALCHEMY MODELS (models.py)               │       │
│  │  Maps Python classes to database tables.             │       │
│  └────────────────────────┬─────────────────────────────┘       │
│                           │  asyncpg driver                     │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────┐       │
│  │               POSTGRESQL DATABASE                    │       │
│  │          (the archive where data lives)              │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                 │
│  THE LIBRARY ANALOGY:                                           │
│  ├─ Client = Library patron                                     │
│  ├─ Route = Front desk ("What do you need?")                    │
│  ├─ Repository = Catalog system ("I know where to find it")     │
│  ├─ AsyncSession = Library card (one per visit, always returned)│
│  ├─ SQLAlchemy Model = Physical book (full internal data)       │
│  ├─ Pydantic Model = Catalog card (clean, patron-facing)        │
│  └─ Transaction = Multi-book checkout (all or nothing)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Lectures and the Project

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 6 PROJECT (starting NOW):                                 │
│  └─ Refactor your Task Manager from in-memory to PostgreSQL     │
│     You'll USE everything from this lecture:                    │
│     async engine, session dependency, repository pattern,       │
│     Pydantic bridge, Alembic migrations (Lecture 3),            │
│     and integration tests with a real test database.            │
│                                                                 │
│  WEEK 7: Advanced Database Patterns                             │
│  └─ Query optimization with EXPLAIN                             │
│     Your repository methods are where you'll optimize.          │
│     The separation we built today makes this easy:              │
│     change the query in ONE place, not in every route.          │
│                                                                 │
│  WEEK 8: External APIs                                          │
│  └─ httpx.AsyncClient fetching from external services           │
│     Same async pattern — await calls that talk to the network.  │
│     You'll combine external API data with database data,        │
│     both through their own repositories.                        │
│                                                                 │
│  WEEK 9: Authentication                                         │
│  └─ User model in the database, password hashing, JWT           │
│     get_current_user will be another Depends() in the chain.    │
│     Your dependency chain grows:                                │
│     session → repository → current_user → route                 │
│                                                                 │
│  WEEK 10: Redis Caching                                         │
│  └─ Cache layer sits BETWEEN route and repository               │
│     Check cache → miss → call repository → store in cache       │
│     The layered architecture we built today makes this          │
│     a clean addition, not a rewrite.                            │
│                                                                 │
│  WEEK 13-14: Capstone                                           │
│  └─ Multi-tenant SaaS backend                                   │
│     Multiple repositories, multiple models, complex queries.    │
│     The patterns from today are the FOUNDATION.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# File Organization Reference

```
┌─────────────────────────────────────────────────────────────────┐
│              WHERE EVERYTHING LIVES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  your_project/                                                  │
│  ├── main.py                 # FastAPI app, startup/shutdown    │
│  ├── database.py             # Engine, session factory,         │
│  │                           #   get_async_session dependency   │
│  ├── models.py               # SQLAlchemy models (Base, Task,   │
│  │                           #   Category)                      │
│  ├── schemas.py              # Pydantic models (TaskCreate,     │
│  │                           #   TaskResponse, etc.)            │
│  ├── dependencies.py         # get_task_repository and other    │
│  │                           #   dependency factories           │
│  ├── repositories/                                              │
│  │   ├── __init__.py                                            │
│  │   └── task_repository.py  # TaskRepository class             │
│  ├── routes/                                                    │
│  │   ├── __init__.py                                            │
│  │   └── tasks.py            # Task route handlers              │
│  ├── alembic/                # Migrations (Lecture 3)           │
│  │   └── versions/                                              │
│  └── tests/                                                     │
│      ├── conftest.py         # Test DB, fixtures, client        │
│      ├── test_tasks.py       # Integration tests (API)          │
│      └── test_repositories.py # Repository unit tests           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```