# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Async SQLAlchemy is just sync SQLAlchemy with `await` sprinkled in front of every call. I can take my working sync code, swap imports, add `await`, and everything works the same way — lazy loading, attribute access after commit, session management — it's all identical under the hood."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---------------|---------|
| 1 | `expire_on_commit=False` makes all attributes accessible forever | It only preserves **already-loaded** column values; relationships that were never eagerly loaded still trigger `MissingGreenlet` |
| 2 | `await session.flush()` saves data to the database permanently | `flush()` sends SQL and assigns IDs but does NOT commit; a subsequent rollback or session close discards everything |
| 3 | `await session.refresh(task)` reloads the entire object including relationships | `refresh()` reloads **column values** only; relationship access still triggers lazy loading → crash in async |
| 4 | `session.add(obj)` needs `await` because it "writes to the database" | `add()` is a purely local Python operation — it stages the object in the session's identity map. No I/O occurs. |
| 5 | Yield dependency cleanup runs **before** the route's return value is serialized | FastAPI serializes the return value first, **then** resumes the dependency generator for cleanup |
| 6 | SQL queries inside route handlers are fine since "the app is small" | Duplicated queries across routes, untestable DB logic, and invisible N+1 problems compound immediately |
| 7 | A shared global session works if you "just `await` everything" | Concurrent requests on a shared session interleave `add()`, `commit()`, and `rollback()`, corrupting each other's transactions |

**The Prediction Gap:** When students see `expire_on_commit=False` and `await session.refresh(task)`, they are confident that every attribute on `task` — columns AND relationships — is now safely accessible. The actual behavior is that only scalar column values are refreshed. Accessing `task.category` still triggers a synchronous lazy load inside the async session, which crashes with `MissingGreenlet`. The student's mental model conflates "refreshed" with "fully loaded," when in reality these are two completely different operations.

---

## The Main Purpose (Essence)

**Core Insight:** Async SQLAlchemy does not just add `await` to your code — it fundamentally changes the **data access contract**. In sync SQLAlchemy, the ORM silently fetches data you didn't ask for (lazy loading). In async SQLAlchemy, you must **declare everything you need upfront**. This feels restrictive, but it is actually a feature: it makes your data access patterns explicit, prevents hidden N+1 queries, and forces you to think about what data each endpoint actually requires.

**The Problems Being Solved:**

| Layer | Without It | With It |
|-------|-----------|---------|
| Async Engine + Session | Sync DB call blocks event loop; 10 concurrent requests behave like 1 sequential chain | Each `await` yields control; all 10 requests overlap their I/O waits |
| Yield Dependency | Manual try/finally in every route; one forgotten `close()` = connection leak | Session lifecycle is automatic: create → yield → commit/rollback → close, guaranteed |
| Repository Pattern | Identical SQL queries copy-pasted across 5+ routes; untestable DB logic | One method, one query, tested in isolation; routes say WHAT, repository says HOW |
| Pydantic Bridge | SQLAlchemy internals leak into API responses (relationship proxies, internal IDs) | Clean separation: database-facing model vs. client-facing contract |

**The Key Distinction:**
- **Sync SQLAlchemy** = "Ask for data whenever you want; the ORM silently fetches it behind the scenes."
- **Async SQLAlchemy** = "Declare what you need upfront; any implicit fetch attempt crashes immediately."
- This distinction propagates through every decision: session setup (`expire_on_commit`), query design (eager loading), response serialization (from_attributes), and testing strategy (real DB vs mocks).

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "`expire_on_commit=False` = all attributes safe" | `task.category.name` works after commit | `MissingGreenlet` crash | expire_on_commit only preserves attributes that were **already loaded into Python memory**; it cannot load something that was never fetched |
| "`flush()` = data is saved" | Task with assigned ID exists in DB after exception | Task is gone; ID is "wasted" | `flush()` sends SQL within the current transaction but `commit()` makes it permanent; without commit, rollback discards everything |
| "`refresh()` reloads the full object" | `task.category` accessible after refresh | `MissingGreenlet` crash | `refresh()` re-executes `SELECT` for the object's **columns only**; it does not follow relationships |
| "`session.add()` needs await" | Removing await from `session.add(task)` breaks code | Code works identically | `add()` is a local dictionary insertion in Python; no network I/O occurs |
| "Yield dependency cleanup runs first" | Session committed before response is serialized | Response serialized **before** dependency resumes | FastAPI captures return value → serializes → then resumes generator past yield |
| "Global session is fine with await" | Two concurrent requests operate independently | Request B's rollback undoes Request A's staged writes | Both requests share the same transaction state on one session object |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Lifecycle Puzzle

```python
from typing import AsyncGenerator
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession

# Assume async_session_factory is properly configured with:
# expire_on_commit=False, bound to a valid async engine

app = FastAPI()

async def get_async_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            print("DEP-1: Yielding session")
            yield session
            print("DEP-2: Route succeeded, committing")
            await session.commit()
        except Exception:
            print("DEP-3: Route raised, rolling back")
            await session.rollback()
            raise
        finally:
            print("DEP-4: Finally block executing")


@app.post("/tasks", status_code=201)
async def create_task(
    session: AsyncSession = Depends(get_async_session),
):
    print("ROUTE-1: Entering route")
    task = Task(title="Test Task")
    session.add(task)
    await session.flush()
    print(f"ROUTE-2: Flushed, task.id = {task.id}")
    return {"id": task.id, "title": task.title}
```

**Scenario A:** A POST request arrives and the route executes successfully (no exceptions).

**Scenario B:** The `await session.flush()` call raises an `IntegrityError` (e.g., a unique constraint violation on the title).

**Questions:**

1. **Predict the exact order of print statements for Scenario A.** Write each DEP-* and ROUTE-* label in the order they appear in the server log.

2. **Predict the exact order of print statements for Scenario B.** Which DEP-* statements execute? Does DEP-4 execute even though an exception was raised?

3. **Explain the mechanism.** A student claims: "DEP-2 runs before ROUTE-1 because the dependency is resolved first." Why is this wrong? What does `yield` do in an `async def` dependency?

4. **Critical thinking.** In Scenario A, at the moment `return {"id": task.id, "title": task.title}` executes, has `session.commit()` been called yet? Why does this matter for the data in the response?

---

## Exercise 1.2: The False Safety Net

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
)
from sqlalchemy import select
from sqlalchemy.orm import selectinload

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/taskdb")

async_session_factory = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,   # <-- Students see this and feel safe
)


async def demonstrate():
    async with async_session_factory() as session:
        # Load a task WITHOUT eager loading the category
        result = await session.execute(
            select(Task).where(Task.id == 1)
        )
        task = result.scalar_one()

        # Access a column attribute before commit — works fine
        title_before = task.title
        print(f"A: Before commit, title = {title_before}")

        await session.commit()

        # Access the SAME column attribute after commit
        title_after = task.title
        print(f"B: After commit, title = {title_after}")

        # Now access the relationship after commit
        category_name = task.category.name
        print(f"C: After commit, category = {category_name}")
```

**Assume** the database contains a Task with `id=1`, `title="Buy milk"`, and a related Category with `name="Shopping"`.

**Questions:**

1. **Predict the output.** Does line A print? Does line B print? Does line C print? If any line crashes, state the exception type and explain what triggers it.

2. **Explain the distinction.** `expire_on_commit=False` is set. A student argues: "That means nothing expires after commit, so all attributes are accessible." Identify the flaw in this reasoning. What is the difference between an **expired** attribute and a **never-loaded** attribute?

3. **Mental Trace.** Walk through what data exists in Python memory after `result.scalar_one()` returns. Specifically: which attributes of `task` are populated, and which are not? What does the `category` attribute contain at this point?

4. **Fix the code** so that line C works. You may change exactly ONE line in the query. Do not remove `expire_on_commit=False`.

---

## Exercise 1.3: The Phantom Record

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession


async def create_categorized_task(session_factory) -> int | None:
    """Create a category and a task, return the task's ID."""
    async with session_factory() as session:
        category = Category(name="Urgent")
        session.add(category)
        print(f"After add: category.id = {category.id}")

        await session.flush()
        print(f"After flush: category.id = {category.id}")

        task = Task(title="Fix critical bug", category_id=category.id)
        session.add(task)
        await session.flush()
        print(f"After second flush: task.id = {task.id}")

        saved_task_id = task.id

        # Simulate a validation error AFTER both flushes
        if len(task.title) > 10:
            raise ValueError("Title too long for quick tasks!")

        await session.commit()
        print("Committed successfully")
        return saved_task_id


async def check_database(session_factory, task_id: int):
    """Check if the task actually exists."""
    async with session_factory() as session:
        result = await session.execute(
            select(Task).where(Task.id == task_id)
        )
        task = result.scalar_one_or_none()
        print(f"Task with id={task_id} exists: {task is not None}")


async def main():
    try:
        task_id = await create_categorized_task(async_session_factory)
    except ValueError as e:
        print(f"Caught error: {e}")
        task_id = 1  # We saw the ID was assigned!

    await check_database(async_session_factory, task_id)
```

**Questions:**

1. **Predict the full output** of `main()`, including all print statements and any caught exceptions. Specifically:
   - What value does `category.id` have after `add()`? After `flush()`?
   - What value does `task.id` have after the second `flush()`?
   - Does the "Committed successfully" line print?
   - Does `check_database` find the task?

2. **Explain the paradox.** The task was assigned a real database ID (e.g., `id=1`) by PostgreSQL during `flush()`. PostgreSQL actually processed the `INSERT` statement. So why doesn't the task exist when we check with a fresh session?

3. **Mental model.** A student says: "If PostgreSQL assigned an ID, the row must be in the database." Correct this mental model. What is the relationship between `flush()`, the database's transaction log, and `commit()`?

4. **Consequence.** After this code runs, what happens to the ID that was assigned to the task? If you create another task later, will it get ID `1` (reused) or ID `2` (skipped)?

---

## Exercise 1.4: The Refresh Illusion

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession


async def create_and_inspect(session: AsyncSession):
    """Create a task with a category, then try to read the category name."""

    # First, ensure the category exists
    result = await session.execute(
        select(Category).where(Category.name == "Work")
    )
    work_category = result.scalar_one()

    # Create a new task in the "Work" category
    task = Task(title="Write report", category_id=work_category.id)
    session.add(task)
    await session.flush()
    print(f"A: task.id = {task.id}")
    print(f"B: task.category_id = {task.category_id}")

    # Refresh to get DB-generated fields
    await session.refresh(task)
    print(f"C: task.id = {task.id}")
    print(f"D: task.title = {task.title}")

    # Now access the relationship
    print(f"E: task.category.name = {task.category.name}")
```

**Assume** a "Work" category exists in the database with `id=1`.

**Questions:**

1. **Predict the output.** Which lines print successfully? Does line E work, or does it crash? If it crashes, what is the exception type?

2. **Explain the mechanism.** A student reasons: "I called `refresh(task)`, which reloads the object from the database. The database row has `category_id=1`, and the Category table has a row with `id=1` and `name='Work'`. So `task.category` should be loaded." Identify the flaw in this reasoning. What exactly does `refresh()` reload?

3. **The irony.** We already have `work_category` loaded in memory — a Python variable holding the exact Category object we need. The data is RIGHT THERE. But `task.category` doesn't point to it. Why? What would SQLAlchemy need to do to make `task.category` resolve to `work_category`, and why doesn't it do this automatically in async?

4. **Fix the code.** Provide TWO different fixes:
   - **Fix A:** Change the query/loading strategy so that `task.category` is available after the flush.
   - **Fix B:** Don't load the relationship at all — instead, use the `work_category` object you already have to build the response data manually.

---

# Level 1 Solutions

## Solution 1.1: The Lifecycle Puzzle

**1. Scenario A (success) — Order of prints:**
```
DEP-1: Yielding session
ROUTE-1: Entering route
ROUTE-2: Flushed, task.id = <some_integer>
DEP-2: Route succeeded, committing
DEP-4: Finally block executing
```

**2. Scenario B (IntegrityError on flush) — Order of prints:**
```
DEP-1: Yielding session
ROUTE-1: Entering route
DEP-3: Route raised, rolling back
DEP-4: Finally block executing
```

ROUTE-2 never prints because the exception occurs at `await session.flush()` before that line. DEP-4 executes in **both** scenarios because it is in a `finally` block, which runs regardless of whether the `try` or `except` branch was taken. This is the same `finally` guarantee from Week 1, Lecture 2 (context managers and error handling).

**3. The mechanism:**

The student's claim is wrong because `yield` **suspends** the generator, it doesn't "resolve and continue." Here is the execution flow:

1. FastAPI calls `get_async_session()` — this creates an async generator object (it does NOT start executing).
2. FastAPI advances the generator to the first `yield` — this runs the code up to and including `print("DEP-1")`, then suspends at `yield session`.
3. FastAPI takes the yielded `session` and injects it into the route handler.
4. The route handler runs (ROUTE-1, ROUTE-2).
5. After the route handler returns (or raises), FastAPI **resumes the generator** past the `yield`.
6. If the route succeeded: execution continues after `yield` → DEP-2 → commit → finally → DEP-4.
7. If the route raised: FastAPI throws the exception into the generator → the `except` block catches it → DEP-3 → rollback → re-raise → finally → DEP-4.

The key insight: **`yield` is a suspension point**. Code before `yield` = setup. Code after `yield` = cleanup. The route runs *in between*.

**4. Critical thinking:**

No, `session.commit()` has NOT been called yet when the `return` statement executes. The commit happens in DEP-2, which runs AFTER the route returns its value.

This matters because the data in the response dictionary `{"id": task.id, "title": task.title}` is read from the Python object's in-memory attributes — NOT from a committed database row. This is precisely why `expire_on_commit=False` is critical: if it were `True`, the attributes would be expired at commit time, and any attempt to read them during response serialization (which happens before the dependency cleanup) could trigger a lazy load. But since the commit hasn't happened yet at return time, `expire_on_commit` doesn't even come into play here — the concern is about **what happens when FastAPI serializes the response**, which happens after `return` but before the dependency resumes.

---

## Solution 1.2: The False Safety Net

**1. Output:**
```
A: Before commit, title = Buy milk
B: After commit, title = Buy milk
```
Then the program **crashes** on line C with:

```
sqlalchemy.exc.MissingGreenlet: greenlet_spawn has not been called;
can't call await_only() here.
```

Line A works: `task.title` was loaded during `session.execute()`, so it's in Python memory. Line B works: `expire_on_commit=False` prevents the commit from expiring already-loaded column values, so `task.title` is still accessible. Line C crashes: `task.category` was **never loaded** — it's not an "expired" attribute, it's an attribute that SQLAlchemy would need to **lazy-load** from the database. Lazy loading requires a synchronous database roundtrip, which is impossible inside an `AsyncSession`.

**2. The distinction:**

The student's reasoning confuses two entirely different states:

- **Expired attribute:** Was loaded into Python memory, then marked as "stale" by `commit()`. With `expire_on_commit=True`, accessing it triggers a refresh query. With `expire_on_commit=False`, accessing it returns the stale-but-still-present Python value.

- **Never-loaded attribute (lazy proxy):** Was never fetched from the database at all. The attribute holds a SQLAlchemy proxy object that knows HOW to fetch the data, but hasn't done so yet. Accessing it triggers a **lazy load** — a new `SELECT` query. In sync SQLAlchemy, this happens silently. In async SQLAlchemy, this crashes because a lazy load is a synchronous I/O operation, and `AsyncSession` cannot perform synchronous I/O.

`expire_on_commit=False` has **zero effect** on never-loaded attributes. It only governs what happens to data that was already in memory.

**3. Mental trace after `result.scalar_one()` returns:**

When SQLAlchemy executes `select(Task).where(Task.id == 1)`, it sends a SQL query like:
```sql
SELECT tasks.id, tasks.title, tasks.description, tasks.completed,
       tasks.created_at, tasks.category_id
FROM tasks WHERE tasks.id = 1
```

This loads all **columns** of the `tasks` table into the Python object. So `task.id`, `task.title`, `task.description`, `task.completed`, `task.created_at`, and `task.category_id` are all populated in memory.

However, `task.category` is a **relationship** attribute. It points to a row in the `categories` table. That row was NOT included in the SQL query. The `category` attribute holds a lazy-loading proxy — an object that says "I know the foreign key is `category_id=1`, but I haven't fetched the `Category` row yet."

**4. Fix:**

Change the query to eagerly load the relationship:

```python
result = await session.execute(
    select(Task)
    .where(Task.id == 1)
    .options(selectinload(Task.category))   # <-- Add this
)
```

This causes SQLAlchemy to issue a second query (or a JOIN, depending on the strategy) that fetches the related `Category` row and populates `task.category` in memory. After this, line C works because the data is already in Python — no lazy load needed.

---

## Solution 1.3: The Phantom Record

**1. Full output:**
```
After add: category.id = None
After flush: category.id = 1
After second flush: task.id = 1
Caught error: Title too long for quick tasks!
Task with id=1 exists: False
```

(Exact ID values may vary, but the pattern holds.)

- After `add()`: `category.id = None` because `add()` is a local Python operation. No SQL was sent. The database has not assigned an ID.
- After `flush()`: `category.id = 1` (or similar integer) because `flush()` sent the `INSERT` statement to PostgreSQL, which assigned the ID via its sequence.
- After second `flush()`: `task.id = 1` for the same reason.
- "Committed successfully" does **NOT** print because the `ValueError` is raised before `commit()`.
- `check_database` reports `False` — the task does not exist.

**2. The paradox:**

PostgreSQL did process the `INSERT` statements. It did assign IDs. The rows existed **within the transaction**. But `commit()` was never called. When the `async with` block exits (due to the exception), the session closes, and the underlying database connection's transaction is **rolled back**. PostgreSQL discards the inserted rows as if they never happened.

The key concept is **transaction isolation** (Week 5, Lecture 3). Rows inserted within a transaction are only visible to that transaction until `COMMIT` is executed. From every other session's perspective, those rows never existed. Rolling back erases them completely.

`flush()` = "Send the SQL to the database **within the current transaction**."
`commit()` = "Tell the database to make the transaction **permanent**."
Without `commit()`, the transaction is ephemeral.

**3. Mental model correction:**

The correct model: PostgreSQL's sequence (the auto-incrementing ID generator) is **not transactional**. When the `INSERT` is flushed, the sequence advances from 0 to 1 and assigns that ID. When the transaction rolls back, the row is deleted, but the sequence **does not rewind**. This is by design — sequences must be non-transactional to avoid locking issues under high concurrency.

**4. Consequence:**

The ID is **skipped**, not reused. The next task created will get ID `2`. The sequence has a "gap" where ID `1` was consumed but never committed. This is normal and expected in PostgreSQL. Gaps in ID sequences are not a bug — they're a consequence of transaction isolation applied to sequence generators.

---

## Solution 1.4: The Refresh Illusion

**1. Output:**
```
A: task.id = <integer, e.g., 1>
B: task.category_id = 1
C: task.id = <same integer>
D: task.title = Write report
```
Then line E **crashes** with `MissingGreenlet`.

Lines A-D all work. After `flush()`, the `INSERT` has been sent and `task.id` is assigned. After `refresh()`, the columns are reloaded from the database (confirming `id`, `title`, `category_id`, etc.). But `task.category` — the relationship — was never loaded, and `refresh()` does not load relationships.

**2. The mechanism flaw:**

`refresh()` executes a SQL query like:
```sql
SELECT tasks.id, tasks.title, tasks.description, tasks.completed,
       tasks.created_at, tasks.category_id
FROM tasks WHERE tasks.id = 1
```

This is a `SELECT` on the `tasks` table only. It reloads the **column values** of the `Task` row. It does NOT join to the `categories` table. It does NOT populate the `task.category` relationship proxy.

The student's confusion is that "reload from database" sounds like "reload everything about this entity." In reality, `refresh()` means "re-execute the SELECT for this row's columns." Relationships are a separate concern that require explicit eager loading via `selectinload()` or `joinedload()`.

**3. The irony:**

We have `work_category` sitting right there — a fully loaded `Category` object in Python memory. But SQLAlchemy's `task.category` doesn't magically point to it. In sync SQLAlchemy, accessing `task.category` would trigger a lazy load that queries the database, finds the category with `id=1`, and returns it. SQLAlchemy would even be smart enough to return the same `work_category` object if it's still in the session's identity map. But in async, the lazy load attempt crashes before any of that can happen.

To make `task.category` resolve automatically, SQLAlchemy would need to issue an async SELECT to the categories table. Lazy loading was designed for sync sessions where blocking I/O is acceptable. In async, there's no mechanism to transparently await a database call when you simply access a Python attribute like `.category`.

**4. Two fixes:**

**Fix A — Eager loading after creation:**
```python
# After flushing the new task, reload it with the relationship
await session.flush()
result = await session.execute(
    select(Task)
    .where(Task.id == task.id)
    .options(selectinload(Task.category))
)
task = result.scalar_one()
print(f"E: task.category.name = {task.category.name}")  # ✅ Works
```

**Fix B — Use what you already have:**
```python
# Don't access the relationship at all. Build the response manually.
await session.flush()
await session.refresh(task)
print(f"E: category name = {work_category.name}")  # ✅ Works — it's already loaded!

# In a real route, you'd return something like:
# return {"task_id": task.id, "title": task.title, "category": work_category.name}
```

Fix B is often the pragmatic choice: you already have the data in Python, so why ask the database again? Fix A is the correct approach when you're loading existing data (e.g., `GET /tasks/{id}`) and need the full object graph.

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Session That Silently Leaks

**Context:** A developer set up the database dependency for their FastAPI application. It works perfectly in development — every test passes, every endpoint responds. But under load in staging, the application freezes after ~20 requests and eventually throws `TimeoutError: QueuePool limit ... reached`.

```python
# database.py
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
)
from typing import AsyncGenerator

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/taskdb"

engine = create_async_engine(
    DATABASE_URL,
    pool_size=5,
    max_overflow=10,
)

async_session_factory = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


async def get_async_session() -> AsyncGenerator[AsyncSession, None]:
    session = async_session_factory()
    yield session
    await session.commit()
    await session.close()
```

```python
# routes.py
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from database import get_async_session
from models import Task

app = FastAPI()


@app.get("/tasks/{task_id}")
async def get_task(
    task_id: int,
    session: AsyncSession = Depends(get_async_session),
):
    result = await session.execute(
        select(Task).where(Task.id == task_id)
    )
    task = result.scalar_one_or_none()
    if task is None:
        raise HTTPException(status_code=404, detail="Task not found")
    return {"id": task.id, "title": task.title}


@app.post("/tasks", status_code=201)
async def create_task(
    session: AsyncSession = Depends(get_async_session),
):
    task = Task(title="New Task")
    session.add(task)
    await session.flush()
    await session.refresh(task)
    return {"id": task.id, "title": task.title}
```

**The symptom:** In development and testing (one request at a time), everything works. Under load (50+ concurrent requests), the app freezes and crashes with a connection pool exhaustion error after about 15-20 requests.

---

**Task 2a: Identify the Flaw**

The code "works" in every happy-path test. But there is a critical flaw in `get_async_session()` that causes connections to leak under specific conditions.

1. Examine `get_async_session()`. What happens to the session when `GET /tasks/999` is requested (a task that doesn't exist)?
2. Trace the execution: when `HTTPException` is raised in the route, which lines of `get_async_session()` actually execute?
3. Why does this problem only manifest under load, not during single-request testing?

Do NOT fix the code yet. Just identify the exact failure mode.

---

**Task 2b: Explain the Mechanism**

1. In a Python `async def` function that uses `yield`, what happens to the code **after** `yield` when the caller throws an exception into the generator? Does it execute automatically?
2. Why does the current `get_async_session()` fail to execute `await session.close()` when the route raises?
3. Connection pools have a limited number of connections (`pool_size=5, max_overflow=10` = 15 max connections). Explain why leaked sessions eventually exhaust this pool.

---

**Task 2c: Instrument to Prove Diagnosis**

Add logging to `get_async_session()` that reveals whether cleanup actually runs:

```python
async def get_async_session() -> AsyncGenerator[AsyncSession, None]:
    session = async_session_factory()
    session_id = id(session)
    print(f"[{session_id}] Session CREATED")
    yield session
    # Add instrumentation here to prove the leak
    print(f"[{session_id}] Session COMMITTED")
    await session.commit()
    print(f"[{session_id}] Session CLOSED")
    await session.close()
```

Run the following test scenario and show what the logs reveal:

```python
# Test: Request an existing task, then a non-existing task
# GET /tasks/1    (exists)
# GET /tasks/999  (does not exist → HTTPException)
```

What pattern do you see in the logs that proves the leak?

---

**Task 2d: Fix the Code**

Rewrite `get_async_session()` to guarantee that:
1. The session is **always** closed, even if the route raises.
2. The session is **committed** only if the route succeeds.
3. The session is **rolled back** if the route raises an exception.
4. The exception is **re-raised** so FastAPI can return the appropriate HTTP response.

---

**Task 2e: Write Verification Test**

Write a test that **fails** with the original dependency and **passes** with your fixed version. The test must demonstrate that the session is properly cleaned up even when an exception occurs.

```python
import pytest
from httpx import ASGITransport, AsyncClient
from unittest.mock import AsyncMock, patch


@pytest.mark.asyncio
async def test_session_closed_on_404():
    """
    Verify that the database session is properly closed
    even when the route raises HTTPException.
    
    This test should:
    - FAIL with the original (broken) get_async_session
    - PASS with the fixed version
    """
    # Your test here.
    # Hint: Track whether session.close() is called by patching
    # or wrapping the session factory.
    pass
```

---

## Exercise 2.2: The N+1 Time Bomb

**Context:** A developer wrote a "task dashboard" endpoint that returns all tasks with their category names. It works correctly — every response has the right data. But when the database grows to 500 tasks, this single endpoint takes **12 seconds** to respond.

```python
# schemas.py
from pydantic import BaseModel, ConfigDict
from datetime import datetime


class TaskWithCategoryResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str
    completed: bool
    created_at: datetime
    category_name: str | None


# routes.py
@app.get("/dashboard/tasks", response_model=list[TaskWithCategoryResponse])
async def task_dashboard(
    session: AsyncSession = Depends(get_async_session),
):
    # Step 1: Get all tasks
    result = await session.execute(
        select(Task).order_by(Task.created_at.desc())
    )
    tasks = result.scalars().all()

    # Step 2: Build response with category names
    response = []
    for task in tasks:
        category_name = None
        if task.category_id is not None:
            cat_result = await session.execute(
                select(Category).where(Category.id == task.category_id)
            )
            category = cat_result.scalar_one_or_none()
            if category:
                category_name = category.name

        response.append(
            TaskWithCategoryResponse(
                id=task.id,
                title=task.title,
                completed=task.completed,
                created_at=task.created_at,
                category_name=category_name,
            )
        )

    return response
```

---

**Task 2a: Identify the Flaw**

The response data is 100% correct. Every task has the right category name. But the endpoint is unacceptably slow.

1. Count the number of SQL queries this endpoint executes for a database with `N` tasks where `M` tasks have a `category_id`. Express as a formula.
2. For `N=500` with `M=400`, how many queries are executed?
3. Why does this work fine in development (10 tasks) but fail in production (500 tasks)?

---

**Task 2b: Explain the Mechanism**

This is the **N+1 query problem** from Week 6 Lecture 2 — but in a new disguise. In the sync lecture, N+1 happened through lazy loading. Here, the developer avoided lazy loading (they manually query each category) but recreated the same problem.

1. Why is manually querying inside a loop just as bad as lazy loading?
2. What is the fundamental principle being violated?
3. In async code, there's an additional cost beyond just "more queries." What is it? (Hint: think about what `await` does in the loop.)

---

**Task 2c: Instrument to Prove Diagnosis**

Add timing instrumentation that demonstrates the sequential execution:

```python
import time

@app.get("/dashboard/tasks")
async def task_dashboard(session: AsyncSession = Depends(get_async_session)):
    start_total = time.perf_counter()
    query_count = 0

    # ... your instrumented code here ...

    elapsed = time.perf_counter() - start_total
    print(f"Total queries: {query_count}, Total time: {elapsed:.3f}s")
    return response
```

Show example output for `N=50` tasks with `M=40` having categories that proves the problem.

---

**Task 2d: Fix the Code**

Rewrite the endpoint to execute at most **2 SQL queries** regardless of how many tasks exist, using SQLAlchemy's eager loading. The response data must remain identical.

Your fix must:
1. Load all tasks with their categories in a single operation.
2. Build the response using Pydantic's `from_attributes` or manual attribute access on eagerly-loaded relationships.
3. Eliminate the per-task query loop entirely.

---

**Task 2e: Write Verification Test**

Write a test that:
- Creates 20 tasks across 3 categories in a test database
- Calls the endpoint
- Asserts the response is correct (right data, right count)
- Verifies the query count is at most 2

```python
@pytest.mark.asyncio
async def test_dashboard_no_n_plus_one(client: AsyncClient, db_session: AsyncSession):
    """
    Verify that the dashboard endpoint does not exhibit N+1 queries.
    Should FAIL on original code, PASS on fixed code.
    """
    # Setup: Create test data
    # ... create categories and tasks ...
    
    # Act: Call the endpoint
    # ... 
    
    # Assert: Correct data
    # ...
    
    # Assert: Query efficiency
    # Hint: Use SQLAlchemy's echo=True or event listeners to count queries
    pass
```

---

# Level 2 Solutions

## Solution 2.1: The Session That Silently Leaks

**Task 2a: Identify the Flaw**

1. When `GET /tasks/999` is requested, the route raises `HTTPException(status_code=404)`. This exception propagates out of the route handler.

2. In `get_async_session()`:
   ```python
   session = async_session_factory()     # ✅ Executes
   yield session                         # ✅ Executes (session injected into route)
   # --- Route runs here, then raises HTTPException ---
   await session.commit()                # ❌ NEVER EXECUTES
   await session.close()                 # ❌ NEVER EXECUTES
   ```

   When FastAPI throws the exception into the generator (via `.athrow()`), and the generator has no `try/except` around the `yield`, the exception propagates **out of the generator**. The code after `yield` is simply skipped. The session is never committed, never closed.

3. In single-request testing, the leaked session's connection is eventually garbage-collected. Python's garbage collector closes the connection, returning it to the pool. But under load, new requests arrive faster than the garbage collector runs, so connections accumulate until the pool's maximum is exceeded.

**Task 2b: Explain the Mechanism**

1. In an async generator, code after `yield` only executes if:
   - The caller advances the generator past the yield (calls `__anext__()` again), OR
   - The caller throws an exception that the generator catches with `try/except`.

   If the caller throws an exception and the generator does NOT have a `try/except` around the `yield`, the exception propagates and the generator terminates without running post-yield code.

2. FastAPI's dependency system throws the route's exception into the generator using `.athrow()`. Since `get_async_session()` has no `try/except` around `yield session`, the `HTTPException` propagates straight through. The `await session.commit()` and `await session.close()` lines are never reached.

3. Each leaked session holds a database connection borrowed from the pool. The pool has `pool_size=5` permanent connections and `max_overflow=10` temporary ones = 15 max. After 15 error-producing requests without garbage collection, every connection is leaked. The 16th request waits for a connection, times out, and raises `TimeoutError`.

**Task 2c: Instrumented output**

```
# GET /tasks/1 (exists)
[140234567890] Session CREATED
[140234567890] Session COMMITTED
[140234567890] Session CLOSED

# GET /tasks/999 (does not exist)
[140234567891] Session CREATED
# ... silence. No COMMITTED or CLOSED lines appear.
```

The pattern: for successful requests, all three log lines appear. For failed requests, only `CREATED` appears — proving the session is never committed or closed.

**Task 2d: Fix**

```python
async def get_async_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

This fix uses three layered guarantees:
1. `async with async_session_factory() as session:` — The async context manager ensures `session.close()` is called when the block exits, regardless of how it exits.
2. `try/except` around `yield` — Catches any exception thrown into the generator by FastAPI.
3. `except` block — Rolls back the transaction and re-raises, ensuring the database state is clean AND FastAPI still handles the error.

**Task 2e: Verification Test**

```python
import pytest
from httpx import ASGITransport, AsyncClient
from database import async_session_factory, get_async_session
from main import app


@pytest.mark.asyncio
async def test_session_closed_on_404():
    """Verify session cleanup on route exception."""
    sessions_created = []
    sessions_closed = []

    original_factory = async_session_factory

    class TrackingSession:
        """Wrapper that tracks session lifecycle."""
        def __init__(self, real_session):
            self._session = real_session
            self._id = id(real_session)
            sessions_created.append(self._id)

        async def close(self):
            sessions_closed.append(self._id)
            await self._session.close()

        def __getattr__(self, name):
            return getattr(self._session, name)

    async def tracked_get_session():
        async with original_factory() as session:
            tracked = TrackingSession(session)
            try:
                yield tracked
                await session.commit()
            except Exception:
                await session.rollback()
                raise

    app.dependency_overrides[get_async_session] = tracked_get_session

    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as client:
        # Request a non-existent task (triggers 404)
        response = await client.get("/tasks/99999")
        assert response.status_code == 404

    app.dependency_overrides.clear()

    # The critical assertion: every created session was closed
    assert len(sessions_created) > 0, "No sessions were created"
    assert set(sessions_created) == set(sessions_closed), (
        f"Session leak detected! Created: {sessions_created}, Closed: {sessions_closed}"
    )
```

---

## Solution 2.2: The N+1 Time Bomb

**Task 2a: Identify the Flaw**

1. Number of queries = **1 + M**, where `M` is the number of tasks with a non-null `category_id`.
   - 1 query to fetch all tasks.
   - 1 query per task that has a category (to fetch the `Category` row).

2. For `N=500, M=400`: **1 + 400 = 401 queries**.

3. With 10 tasks, you execute ~11 queries — barely noticeable. With 500 tasks, you execute ~401 queries. Each query has network latency (even on localhost, ~0.5-1ms per query). 401 queries × ~1ms = ~400ms of pure network overhead. On a remote database with ~25ms latency per query: 401 × 25ms = **~10 seconds**.

**Task 2b: Explain the Mechanism**

1. Manually querying inside a loop is functionally identical to lazy loading — both execute one SELECT per related object, inside a loop. The developer avoided the `MissingGreenlet` error by writing explicit queries, but recreated the exact same performance problem.

2. The fundamental principle being violated: **fetch the data you need in the fewest possible round-trips to the database.** Instead of making N separate "give me one category" requests, make one "give me all the categories I need" request.

3. The additional async cost: each `await session.execute()` in the loop is a **suspension point**. The coroutine yields control to the event loop, waits for the database response, then resumes. While the `await` overhead is small per call, 400 suspensions add scheduling overhead on top of network latency. More importantly, these queries are **serialized** — each `await` completes before the next one starts, so there's no concurrency benefit.

**Task 2c: Instrumented output** (for N=50, M=40)

```python
@app.get("/dashboard/tasks")
async def task_dashboard(session: AsyncSession = Depends(get_async_session)):
    start_total = time.perf_counter()
    query_count = 0

    result = await session.execute(
        select(Task).order_by(Task.created_at.desc())
    )
    query_count += 1
    tasks = result.scalars().all()

    response = []
    for task in tasks:
        category_name = None
        if task.category_id is not None:
            cat_result = await session.execute(
                select(Category).where(Category.id == task.category_id)
            )
            query_count += 1
            category = cat_result.scalar_one_or_none()
            if category:
                category_name = category.name
        response.append(
            TaskWithCategoryResponse(
                id=task.id, title=task.title, completed=task.completed,
                created_at=task.created_at, category_name=category_name,
            )
        )

    elapsed = time.perf_counter() - start_total
    print(f"Total queries: {query_count}, Total time: {elapsed:.3f}s")
    return response
```

Output:
```
Total queries: 41, Total time: 0.487s
```

41 queries for 50 tasks (1 initial + 40 category lookups). As N grows, this scales linearly.

**Task 2d: Fix**

```python
from sqlalchemy.orm import selectinload


@app.get("/dashboard/tasks", response_model=list[TaskWithCategoryResponse])
async def task_dashboard(
    session: AsyncSession = Depends(get_async_session),
):
    result = await session.execute(
        select(Task)
        .options(selectinload(Task.category))   # One extra query for ALL categories
        .order_by(Task.created_at.desc())
    )
    tasks = result.scalars().all()

    return [
        TaskWithCategoryResponse(
            id=task.id,
            title=task.title,
            completed=task.completed,
            created_at=task.created_at,
            category_name=task.category.name if task.category else None,
        )
        for task in tasks
    ]
```

This executes exactly **2 queries** regardless of task count:
1. `SELECT * FROM tasks ORDER BY created_at DESC`
2. `SELECT * FROM categories WHERE id IN (1, 2, 3, ...)` — one query for ALL referenced categories

With `selectinload`, SQLAlchemy collects all the `category_id` values from the tasks, issues a single `IN` query for all referenced categories, and populates `task.category` for every task in memory. The list comprehension then accesses `task.category.name` directly — no lazy load, no extra queries.

**Task 2e: Verification Test**

```python
import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy import event


@pytest.mark.asyncio
async def test_dashboard_no_n_plus_one(client: AsyncClient, db_session):
    """Dashboard must not produce N+1 queries."""
    # Setup: Create test data
    categories = [
        Category(name="Work"),
        Category(name="Personal"),
        Category(name="Urgent"),
    ]
    for cat in categories:
        db_session.add(cat)
    await db_session.flush()

    for i in range(20):
        task = Task(
            title=f"Task {i}",
            category_id=categories[i % 3].id,
        )
        db_session.add(task)
    await db_session.flush()

    # Track query count via SQLAlchemy event
    query_log = []

    @event.listens_for(test_engine.sync_engine, "before_cursor_execute")
    def log_query(conn, cursor, statement, parameters, context, executemany):
        query_log.append(statement)

    # Act
    response = await client.get("/dashboard/tasks")

    # Remove listener
    event.remove(test_engine.sync_engine, "before_cursor_execute", log_query)

    # Assert: Correct data
    assert response.status_code == 200
    data = response.json()
    assert len(data) == 20
    assert all(item["category_name"] in ["Work", "Personal", "Urgent"] for item in data)

    # Assert: Query efficiency — at most 2 SELECT queries for tasks+categories
    select_queries = [q for q in query_log if q.strip().upper().startswith("SELECT")]
    assert len(select_queries) <= 2, (
        f"N+1 detected! Expected ≤2 SELECT queries, got {len(select_queries)}. "
        f"Queries: {select_queries}"
    )
```

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Exercise 2.5: Evolving the Task Repository

**Background:** The following code is a working, fully-tested async Task Manager API. It has a repository, a yield dependency for session management, Pydantic schemas, and routes. All tests pass.

Your job: extend this codebase with three new features **without breaking any existing tests**.

### Given: Working Code

```python
# ─── models.py ────────────────────────────────────────────────
from datetime import datetime
from sqlalchemy import String, Text, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Category(Base):
    __tablename__ = "categories"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True)
    tasks: Mapped[list["Task"]] = relationship(back_populates="category")


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


# ─── database.py ──────────────────────────────────────────────
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import (
    create_async_engine, AsyncSession, async_sessionmaker,
)

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/taskdb"
engine = create_async_engine(DATABASE_URL)
async_session_factory = async_sessionmaker(
    bind=engine, class_=AsyncSession, expire_on_commit=False,
)

async def get_async_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


# ─── schemas.py ───────────────────────────────────────────────
from pydantic import BaseModel, Field, ConfigDict
from datetime import datetime


class TaskCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    description: str | None = None
    category_id: int | None = None

class TaskResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str
    description: str | None
    completed: bool
    created_at: datetime


# ─── repositories/task_repository.py ─────────────────────────
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from models import Task


class TaskRepository:
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def get_by_id(self, task_id: int) -> Task | None:
        stmt = select(Task).where(Task.id == task_id)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_all(self, limit: int = 50, offset: int = 0) -> list[Task]:
        stmt = select(Task).order_by(Task.created_at.desc()).limit(limit).offset(offset)
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def create(self, title: str, description: str | None = None,
                     category_id: int | None = None) -> Task:
        task = Task(title=title, description=description, category_id=category_id)
        self.session.add(task)
        await self.session.flush()
        await self.session.refresh(task)
        return task

    async def delete(self, task: Task) -> None:
        await self.session.delete(task)
        await self.session.flush()


# ─── dependencies.py ──────────────────────────────────────────
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from database import get_async_session
from repositories.task_repository import TaskRepository

async def get_task_repository(
    session: AsyncSession = Depends(get_async_session),
) -> TaskRepository:
    return TaskRepository(session)


# ─── routes.py ────────────────────────────────────────────────
from fastapi import FastAPI, Depends, HTTPException
from dependencies import get_task_repository
from repositories.task_repository import TaskRepository
from schemas import TaskCreate, TaskResponse

app = FastAPI()

@app.get("/tasks", response_model=list[TaskResponse])
async def list_tasks(repo: TaskRepository = Depends(get_task_repository)):
    return await repo.get_all()

@app.get("/tasks/{task_id}", response_model=TaskResponse)
async def get_task(task_id: int, repo: TaskRepository = Depends(get_task_repository)):
    task = await repo.get_by_id(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="Task not found")
    return task

@app.post("/tasks", response_model=TaskResponse, status_code=201)
async def create_task(body: TaskCreate, repo: TaskRepository = Depends(get_task_repository)):
    return await repo.create(
        title=body.title, description=body.description, category_id=body.category_id,
    )
```

### Given: Existing Test Suite (Must Continue Passing)

```python
# ─── tests/test_tasks.py ─────────────────────────────────────
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_create_task(client: AsyncClient):
    response = await client.post("/tasks", json={"title": "Buy milk"})
    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "Buy milk"
    assert data["completed"] is False
    assert "id" in data


@pytest.mark.asyncio
async def test_get_task(client: AsyncClient):
    create_resp = await client.post("/tasks", json={"title": "Find me"})
    task_id = create_resp.json()["id"]
    response = await client.get(f"/tasks/{task_id}")
    assert response.status_code == 200
    assert response.json()["title"] == "Find me"


@pytest.mark.asyncio
async def test_get_task_not_found(client: AsyncClient):
    response = await client.get("/tasks/99999")
    assert response.status_code == 404


@pytest.mark.asyncio
async def test_list_tasks(client: AsyncClient):
    await client.post("/tasks", json={"title": "Task A"})
    await client.post("/tasks", json={"title": "Task B"})
    response = await client.get("/tasks")
    assert response.status_code == 200
    assert len(response.json()) >= 2
```

---

### Extension A: Task Detail with Category (Eager Loading)

**Requirement:** Add a new endpoint `GET /tasks/{task_id}/detail` that returns the task with its category name included.

**New schema:**

```python
class CategoryInfo(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str

class TaskDetailResponse(TaskResponse):
    category: CategoryInfo | None
```

**Constraints:**
- Add a new repository method `get_by_id_with_category` that uses eager loading.
- The route must use this new method.
- `task.category` must be accessible without triggering lazy loading.
- All existing tests must still pass.

**Test your extension passes:**

```python
@pytest.mark.asyncio
async def test_task_detail_with_category(client: AsyncClient, db_session: AsyncSession):
    # Create a category
    category = Category(name="Work")
    db_session.add(category)
    await db_session.flush()

    # Create a task in that category
    response = await client.post(
        "/tasks", json={"title": "Write report", "category_id": category.id}
    )
    task_id = response.json()["id"]

    # Get detail
    detail_resp = await client.get(f"/tasks/{task_id}/detail")
    assert detail_resp.status_code == 200
    data = detail_resp.json()
    assert data["title"] == "Write report"
    assert data["category"]["name"] == "Work"
    assert data["category"]["id"] == category.id


@pytest.mark.asyncio
async def test_task_detail_without_category(client: AsyncClient):
    response = await client.post("/tasks", json={"title": "No category"})
    task_id = response.json()["id"]

    detail_resp = await client.get(f"/tasks/{task_id}/detail")
    assert detail_resp.status_code == 200
    assert detail_resp.json()["category"] is None
```

---

### Extension B: Batch Create (Transaction Atomicity)

**Requirement:** Add a `POST /tasks/batch` endpoint that creates multiple tasks in a single request. If **any** task fails validation or creation, **none** of them should be saved.

**New schema:**

```python
class BatchTaskCreate(BaseModel):
    tasks: list[TaskCreate] = Field(..., min_length=1, max_length=50)

class BatchTaskResponse(BaseModel):
    created_count: int
    tasks: list[TaskResponse]
```

**Constraints:**
- Add a `create_batch` method to `TaskRepository`.
- All tasks must be created within the same transaction (the existing yield dependency already handles this — think about why).
- If one task references a non-existent `category_id`, the entire batch must fail and no tasks should be created.
- All existing tests must still pass.

**Test your extension passes:**

```python
@pytest.mark.asyncio
async def test_batch_create_success(client: AsyncClient):
    response = await client.post("/tasks/batch", json={
        "tasks": [
            {"title": "Task 1"},
            {"title": "Task 2"},
            {"title": "Task 3"},
        ]
    })
    assert response.status_code == 201
    data = response.json()
    assert data["created_count"] == 3
    assert len(data["tasks"]) == 3


@pytest.mark.asyncio
async def test_batch_create_atomic_failure(client: AsyncClient):
    """If one task fails, none should be created."""
    # Create a batch where one task references a non-existent category
    response = await client.post("/tasks/batch", json={
        "tasks": [
            {"title": "Good Task 1"},
            {"title": "Bad Task", "category_id": 99999},  # FK violation
            {"title": "Good Task 2"},
        ]
    })
    # Should fail (4xx error)
    assert response.status_code >= 400

    # Verify NONE of the tasks were created
    list_resp = await client.get("/tasks")
    titles = [t["title"] for t in list_resp.json()]
    assert "Good Task 1" not in titles
    assert "Good Task 2" not in titles
```

---

### Extension C: Filtered List with Category Info

**Requirement:** Extend `GET /tasks` to support filtering by `completed` status and `category_id`, and return category info in the response.

**Constraints:**
- Modify `get_all` in the repository to accept optional filter parameters.
- The response for this endpoint must now use `TaskDetailResponse` (with category info).
- The query must use eager loading — NO N+1 queries, regardless of how many tasks are returned.
- Must work with zero filters (returns all tasks with categories), one filter, or both.
- All existing tests must still pass (existing tests that check `list[TaskResponse]` must still work — `TaskDetailResponse` extends `TaskResponse`, so this should be compatible).

**Test your extension passes:**

```python
@pytest.mark.asyncio
async def test_filter_by_completed(client: AsyncClient):
    await client.post("/tasks", json={"title": "Done task"})
    # Mark it completed via some mechanism (or create it completed if schema allows)
    await client.post("/tasks", json={"title": "Pending task"})

    response = await client.get("/tasks?completed=false")
    assert response.status_code == 200
    titles = [t["title"] for t in response.json()]
    assert "Pending task" in titles


@pytest.mark.asyncio
async def test_filter_by_category(client: AsyncClient, db_session: AsyncSession):
    cat = Category(name="Filtered")
    db_session.add(cat)
    await db_session.flush()

    await client.post("/tasks", json={"title": "In category", "category_id": cat.id})
    await client.post("/tasks", json={"title": "No category"})

    response = await client.get(f"/tasks?category_id={cat.id}")
    assert response.status_code == 200
    data = response.json()
    assert all(t["category"]["name"] == "Filtered" for t in data if t["category"])


@pytest.mark.asyncio
async def test_list_includes_category_info(client: AsyncClient, db_session: AsyncSession):
    cat = Category(name="TestCat")
    db_session.add(cat)
    await db_session.flush()

    await client.post("/tasks", json={"title": "With cat", "category_id": cat.id})
    response = await client.get("/tasks")
    data = response.json()

    tasks_with_cat = [t for t in data if t.get("category")]
    assert len(tasks_with_cat) >= 1
    assert tasks_with_cat[0]["category"]["name"] == "TestCat"
```

---

# Level 2.5 Solutions

## Solution: Extension A — Task Detail with Category

**Repository addition:**

```python
# In TaskRepository
from sqlalchemy.orm import selectinload

async def get_by_id_with_category(self, task_id: int) -> Task | None:
    stmt = (
        select(Task)
        .where(Task.id == task_id)
        .options(selectinload(Task.category))
    )
    result = await self.session.execute(stmt)
    return result.scalar_one_or_none()
```

**Route addition:**

```python
@app.get("/tasks/{task_id}/detail", response_model=TaskDetailResponse)
async def get_task_detail(
    task_id: int,
    repo: TaskRepository = Depends(get_task_repository),
):
    task = await repo.get_by_id_with_category(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="Task not found")
    return task
```

**Why this works:** `selectinload(Task.category)` tells SQLAlchemy to issue a second query that fetches the related `Category` row and populates `task.category` in memory. When FastAPI serializes the return value using `TaskDetailResponse` (which has `from_attributes=True`), Pydantic reads `task.category`, finds a loaded `Category` object, and converts it using `CategoryInfo` — also with `from_attributes=True`. No lazy loading occurs.

**Why existing tests don't break:** This is a new endpoint (`/tasks/{task_id}/detail`) and a new repository method. Nothing in the existing code is modified.

## Solution: Extension B — Batch Create

**Repository addition:**

```python
async def create_batch(self, tasks_data: list[dict]) -> list[Task]:
    tasks = []
    for data in tasks_data:
        task = Task(**data)
        self.session.add(task)
        tasks.append(task)

    # Single flush sends all INSERTs. If any FK violation occurs,
    # the flush raises IntegrityError — no tasks are committed
    # because the yield dependency rolls back on exception.
    await self.session.flush()

    for task in tasks:
        await self.session.refresh(task)

    return tasks
```

**Route addition:**

```python
from fastapi import HTTPException
from sqlalchemy.exc import IntegrityError

@app.post("/tasks/batch", response_model=BatchTaskResponse, status_code=201)
async def create_tasks_batch(
    body: BatchTaskCreate,
    repo: TaskRepository = Depends(get_task_repository),
):
    try:
        tasks = await repo.create_batch(
            [t.model_dump() for t in body.tasks]
        )
    except IntegrityError:
        raise HTTPException(
            status_code=422,
            detail="Batch failed: one or more tasks reference invalid data."
        )
    return BatchTaskResponse(created_count=len(tasks), tasks=tasks)
```

**Why atomicity works:** All tasks are added to the same session and flushed together. If `flush()` raises (e.g., FK violation on `category_id=99999`), the exception propagates to the route, then to the yield dependency, which calls `rollback()`. Since none of the tasks were committed, they all disappear. The single transaction wrapping the entire request guarantees all-or-nothing behavior WITHOUT any extra code.

## Solution: Extension C — Filtered List with Category

**Repository modification:**

```python
async def get_all(
    self,
    limit: int = 50,
    offset: int = 0,
    completed: bool | None = None,
    category_id: int | None = None,
) -> list[Task]:
    stmt = select(Task).options(selectinload(Task.category))

    if completed is not None:
        stmt = stmt.where(Task.completed == completed)
    if category_id is not None:
        stmt = stmt.where(Task.category_id == category_id)

    stmt = stmt.order_by(Task.created_at.desc()).limit(limit).offset(offset)
    result = await self.session.execute(stmt)
    return list(result.scalars().all())
```

**Route modification:**

```python
@app.get("/tasks", response_model=list[TaskDetailResponse])
async def list_tasks(
    completed: bool | None = None,
    category_id: int | None = None,
    repo: TaskRepository = Depends(get_task_repository),
):
    return await repo.get_all(completed=completed, category_id=category_id)
```

**Why existing tests don't break:** `TaskDetailResponse` extends `TaskResponse`, so it has all the same fields plus `category`. Existing tests that check for `title`, `completed`, `id`, etc. still find those fields. The new `category` field simply appears as additional data. The `selectinload` adds one extra query but doesn't change the result set.

**Why no N+1:** `selectinload(Task.category)` is applied once at query construction. SQLAlchemy issues: (1) `SELECT * FROM tasks WHERE ... ORDER BY ... LIMIT ...` and (2) `SELECT * FROM categories WHERE id IN (...)` — exactly 2 queries regardless of how many tasks are returned.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Exercise 3: The Order Processing System

**Scenario:** You're building an order processing API. An order contains multiple line items, each referencing a product. The system must guarantee data consistency: you cannot create an order with a non-existent product, and you cannot partially create an order (all items succeed or all fail).

### Models (Provided — Do NOT Modify)

```python
# models.py
from datetime import datetime
from sqlalchemy import String, Text, ForeignKey, Numeric
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "products"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    price: Mapped[float] = mapped_column(Numeric(10, 2))
    stock: Mapped[int] = mapped_column(default=0)


class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    customer_name: Mapped[str] = mapped_column(String(200))
    status: Mapped[str] = mapped_column(String(50), default="pending")
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    items: Mapped[list["OrderItem"]] = relationship(
        back_populates="order", cascade="all, delete-orphan",
    )


class OrderItem(Base):
    __tablename__ = "order_items"
    id: Mapped[int] = mapped_column(primary_key=True)
    order_id: Mapped[int] = mapped_column(ForeignKey("orders.id"))
    product_id: Mapped[int] = mapped_column(ForeignKey("products.id"))
    quantity: Mapped[int] = mapped_column()
    unit_price: Mapped[float] = mapped_column(Numeric(10, 2))
    order: Mapped["Order"] = relationship(back_populates="items")
    product: Mapped["Product"] = relationship()
```

### Database & Dependency Setup (Provided — Do NOT Modify)

```python
# database.py
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import (
    create_async_engine, AsyncSession, async_sessionmaker,
)

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/orderdb"
engine = create_async_engine(DATABASE_URL)
async_session_factory = async_sessionmaker(
    bind=engine, class_=AsyncSession, expire_on_commit=False,
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

---

### Task 3a: Implement Core — Order Repository + Create Endpoint

**Requirements:**
1. Create an `OrderRepository` class with a `create_order` method.
2. `create_order` accepts a customer name and a list of items (each item has `product_id` and `quantity`).
3. For each item, the repository must look up the product to get the current `unit_price` and verify the product exists.
4. If ANY product doesn't exist, raise a custom exception (not `HTTPException` — the repository should NOT know about HTTP).
5. Create the `Order` with all `OrderItem` objects in a single transaction.
6. Return the created `Order` object with items accessible.
7. Create the `POST /orders` route and Pydantic schemas.

**Schemas (you must create):**

```python
# At minimum:
class OrderItemCreate(BaseModel):
    product_id: int
    quantity: int = Field(..., gt=0)

class CreateOrderRequest(BaseModel):
    customer_name: str = Field(..., min_length=1)
    items: list[OrderItemCreate] = Field(..., min_length=1)

# Response schemas — you design these.
# Must include: order id, customer_name, status, items with product_name and unit_price
```

**Tests your code must pass:**

```python
@pytest.mark.asyncio
async def test_create_order_basic(client: AsyncClient, db_session: AsyncSession):
    """Basic order creation with valid products."""
    # Setup: Create products
    p1 = Product(name="Widget", price=9.99, stock=100)
    p2 = Product(name="Gadget", price=24.99, stock=50)
    db_session.add_all([p1, p2])
    await db_session.flush()

    # Act: Create order
    response = await client.post("/orders", json={
        "customer_name": "Alice",
        "items": [
            {"product_id": p1.id, "quantity": 2},
            {"product_id": p2.id, "quantity": 1},
        ],
    })

    # Assert
    assert response.status_code == 201
    data = response.json()
    assert data["customer_name"] == "Alice"
    assert data["status"] == "pending"
    assert len(data["items"]) == 2
    assert "id" in data


@pytest.mark.asyncio
async def test_create_order_invalid_product(client: AsyncClient):
    """Order with non-existent product must fail entirely."""
    response = await client.post("/orders", json={
        "customer_name": "Bob",
        "items": [
            {"product_id": 99999, "quantity": 1},
        ],
    })
    assert response.status_code == 422 or response.status_code == 404


@pytest.mark.asyncio
async def test_create_order_atomic(client: AsyncClient, db_session: AsyncSession):
    """If one item references a bad product, NO items should be created."""
    p1 = Product(name="Real Product", price=5.00, stock=10)
    db_session.add(p1)
    await db_session.flush()

    response = await client.post("/orders", json={
        "customer_name": "Charlie",
        "items": [
            {"product_id": p1.id, "quantity": 1},
            {"product_id": 99999, "quantity": 1},  # Bad product
        ],
    })
    assert response.status_code >= 400

    # Verify no order was created
    from sqlalchemy import select, func
    count_result = await db_session.execute(
        select(func.count()).select_from(Order)
    )
    assert count_result.scalar() == 0
```

---

### Task 3b: Get Order Detail — Efficient Query

**Requirements:**
1. Add a `get_order_detail` method to `OrderRepository` that fetches an order with all items AND each item's product information.
2. Create `GET /orders/{order_id}` route.
3. **Constraint: The endpoint must execute at most 3 SQL queries** (one for the order, one for items, one for products — or fewer with joins). It MUST NOT produce N+1 queries regardless of how many items are in the order.
4. The response must include product names and prices for each item.

**Tests your code must pass:**

```python
@pytest.mark.asyncio
async def test_get_order_detail(client: AsyncClient, db_session: AsyncSession):
    """GET /orders/{id} returns full detail with product info."""
    # Setup
    product = Product(name="Premium Widget", price=49.99, stock=100)
    db_session.add(product)
    await db_session.flush()

    create_resp = await client.post("/orders", json={
        "customer_name": "Diana",
        "items": [{"product_id": product.id, "quantity": 3}],
    })
    order_id = create_resp.json()["id"]

    # Act
    response = await client.get(f"/orders/{order_id}")

    # Assert
    assert response.status_code == 200
    data = response.json()
    assert data["customer_name"] == "Diana"
    assert len(data["items"]) == 1
    assert data["items"][0]["quantity"] == 3
    assert data["items"][0]["product_name"] == "Premium Widget"
    assert float(data["items"][0]["unit_price"]) == 49.99


@pytest.mark.asyncio
async def test_get_order_not_found(client: AsyncClient):
    response = await client.get("/orders/99999")
    assert response.status_code == 404


@pytest.mark.asyncio
async def test_get_order_no_n_plus_one(client: AsyncClient, db_session: AsyncSession):
    """Order detail with 10 items must not produce 10+ queries."""
    products = [Product(name=f"P{i}", price=i * 1.0, stock=100) for i in range(10)]
    for p in products:
        db_session.add(p)
    await db_session.flush()

    create_resp = await client.post("/orders", json={
        "customer_name": "Efficiency Test",
        "items": [{"product_id": p.id, "quantity": 1} for p in products],
    })
    order_id = create_resp.json()["id"]

    # Count queries via event listener
    from sqlalchemy import event
    query_log = []

    @event.listens_for(test_engine.sync_engine, "before_cursor_execute")
    def log_q(conn, cursor, stmt, params, context, executemany):
        query_log.append(stmt)

    response = await client.get(f"/orders/{order_id}")

    event.remove(test_engine.sync_engine, "before_cursor_execute", log_q)

    assert response.status_code == 200
    assert len(response.json()["items"]) == 10

    select_queries = [q for q in query_log if q.strip().upper().startswith("SELECT")]
    assert len(select_queries) <= 3, (
        f"N+1 detected: {len(select_queries)} SELECT queries for 10 items"
    )
```

---

### Task 3c: Cancel Order — Stock Restoration

**Requirements:**
1. Add `POST /orders/{order_id}/cancel` that changes the order's status to `"cancelled"`.
2. When an order is cancelled, each item's quantity must be **added back** to the product's `stock`.
3. The entire cancellation (status update + all stock restorations) must be **atomic** — if any stock update fails, the order status must NOT change.
4. Cancelling an already-cancelled order should return 409 Conflict.

**Tests your code must pass:**

```python
@pytest.mark.asyncio
async def test_cancel_order_restores_stock(client: AsyncClient, db_session: AsyncSession):
    """Cancelling an order must restore product stock."""
    product = Product(name="Stockable", price=10.00, stock=50)
    db_session.add(product)
    await db_session.flush()
    product_id = product.id

    # Create order (buying 5 items)
    create_resp = await client.post("/orders", json={
        "customer_name": "Eve",
        "items": [{"product_id": product_id, "quantity": 5}],
    })
    order_id = create_resp.json()["id"]

    # Cancel the order
    cancel_resp = await client.post(f"/orders/{order_id}/cancel")
    assert cancel_resp.status_code == 200
    assert cancel_resp.json()["status"] == "cancelled"

    # Verify stock was restored
    from sqlalchemy import select
    result = await db_session.execute(
        select(Product).where(Product.id == product_id)
    )
    updated_product = result.scalar_one()
    assert updated_product.stock == 50  # Back to original


@pytest.mark.asyncio
async def test_cancel_already_cancelled(client: AsyncClient, db_session: AsyncSession):
    """Cancelling an already-cancelled order returns 409."""
    product = Product(name="Item", price=1.00, stock=10)
    db_session.add(product)
    await db_session.flush()

    create_resp = await client.post("/orders", json={
        "customer_name": "Frank",
        "items": [{"product_id": product.id, "quantity": 1}],
    })
    order_id = create_resp.json()["id"]

    # Cancel once — OK
    resp1 = await client.post(f"/orders/{order_id}/cancel")
    assert resp1.status_code == 200

    # Cancel again — Conflict
    resp2 = await client.post(f"/orders/{order_id}/cancel")
    assert resp2.status_code == 409
```

---

# Level 3 Solutions

## Solution 3a: Core Order Creation

```python
# ─── exceptions.py ────────────────────────────────────────────
class ProductNotFoundError(Exception):
    """Raised when a referenced product does not exist."""
    def __init__(self, product_id: int):
        self.product_id = product_id
        super().__init__(f"Product with id={product_id} not found")


# ─── schemas.py ───────────────────────────────────────────────
from pydantic import BaseModel, Field, ConfigDict
from datetime import datetime


class OrderItemCreate(BaseModel):
    product_id: int
    quantity: int = Field(..., gt=0)

class CreateOrderRequest(BaseModel):
    customer_name: str = Field(..., min_length=1)
    items: list[OrderItemCreate] = Field(..., min_length=1)

class OrderItemResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    product_id: int
    quantity: int
    unit_price: float
    product_name: str | None = None  # Will be populated manually or via property

class OrderResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    customer_name: str
    status: str
    created_at: datetime
    items: list[OrderItemResponse]


# ─── repositories/order_repository.py ─────────────────────────
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload
from models import Order, OrderItem, Product
from exceptions import ProductNotFoundError


class OrderRepository:
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def create_order(
        self, customer_name: str, items_data: list[dict]
    ) -> Order:
        # 1. Look up ALL products in one query (avoid N+1)
        product_ids = [item["product_id"] for item in items_data]
        result = await self.session.execute(
            select(Product).where(Product.id.in_(product_ids))
        )
        products = {p.id: p for p in result.scalars().all()}

        # 2. Verify all products exist
        for item in items_data:
            if item["product_id"] not in products:
                raise ProductNotFoundError(item["product_id"])

        # 3. Create order and items
        order = Order(customer_name=customer_name)
        for item in items_data:
            product = products[item["product_id"]]
            order_item = OrderItem(
                product_id=product.id,
                quantity=item["quantity"],
                unit_price=float(product.price),
            )
            order.items.append(order_item)

        self.session.add(order)
        await self.session.flush()
        await self.session.refresh(order)
        return order


# ─── routes.py ─────────────────────────────────────────────────
from fastapi import FastAPI, Depends, HTTPException
from exceptions import ProductNotFoundError

app = FastAPI()

async def get_order_repository(
    session: AsyncSession = Depends(get_async_session),
) -> OrderRepository:
    return OrderRepository(session)


@app.post("/orders", status_code=201)
async def create_order(
    body: CreateOrderRequest,
    repo: OrderRepository = Depends(get_order_repository),
):
    try:
        order = await repo.create_order(
            customer_name=body.customer_name,
            items_data=[item.model_dump() for item in body.items],
        )
    except ProductNotFoundError as e:
        raise HTTPException(status_code=422, detail=str(e))

    # Build response manually since we need product_name
    # but didn't eagerly load products on the order items
    items_resp = []
    for item in order.items:
        items_resp.append(OrderItemResponse(
            id=item.id,
            product_id=item.product_id,
            quantity=item.quantity,
            unit_price=item.unit_price,
            product_name=None,  # Will be available in get_order_detail
        ))

    return OrderResponse(
        id=order.id,
        customer_name=order.customer_name,
        status=order.status,
        created_at=order.created_at,
        items=items_resp,
    )
```

**Why atomicity works without extra code:** The yield dependency wraps the entire request in one transaction. If `ProductNotFoundError` is raised, it propagates to the route, which raises `HTTPException`, which propagates to the dependency, which calls `rollback()`. No order or items are committed. The repository doesn't need to manage transactions — the dependency does.

**Why the repository raises a custom exception instead of `HTTPException`:** The repository is a data-access layer. It should not know about HTTP status codes. Raising `ProductNotFoundError` lets the route handler decide the appropriate HTTP response. This is the separation of concerns the repository pattern enforces.

## Solution 3b: Efficient Order Detail

```python
# In OrderRepository:
async def get_order_detail(self, order_id: int) -> Order | None:
    stmt = (
        select(Order)
        .where(Order.id == order_id)
        .options(
            selectinload(Order.items).selectinload(OrderItem.product)
        )
    )
    result = await self.session.execute(stmt)
    return result.scalar_one_or_none()
```

```python
# Route:
@app.get("/orders/{order_id}")
async def get_order(
    order_id: int,
    repo: OrderRepository = Depends(get_order_repository),
):
    order = await repo.get_order_detail(order_id)
    if order is None:
        raise HTTPException(status_code=404, detail="Order not found")

    items_resp = [
        OrderItemResponse(
            id=item.id,
            product_id=item.product_id,
            quantity=item.quantity,
            unit_price=item.unit_price,
            product_name=item.product.name,  # Safe — eagerly loaded!
        )
        for item in order.items
    ]

    return OrderResponse(
        id=order.id,
        customer_name=order.customer_name,
        status=order.status,
        created_at=order.created_at,
        items=items_resp,
    )
```

**Why `selectinload(Order.items).selectinload(OrderItem.product)` works:** This is chained eager loading. SQLAlchemy issues: (1) `SELECT * FROM orders WHERE id = ?`, (2) `SELECT * FROM order_items WHERE order_id IN (?)`, (3) `SELECT * FROM products WHERE id IN (...)`. Three queries maximum, regardless of how many items. Accessing `item.product.name` is safe because the product is already in memory.

## Solution 3c: Order Cancellation with Stock Restoration

```python
# Custom exception:
class OrderAlreadyCancelledError(Exception):
    def __init__(self, order_id: int):
        super().__init__(f"Order {order_id} is already cancelled")

# In OrderRepository:
async def cancel_order(self, order_id: int) -> Order:
    # Load order with items AND products (need product for stock update)
    order = await self.get_order_detail(order_id)
    if order is None:
        raise OrderNotFoundError(order_id)
    if order.status == "cancelled":
        raise OrderAlreadyCancelledError(order_id)

    # Restore stock for each item
    for item in order.items:
        item.product.stock += item.quantity

    # Update order status
    order.status = "cancelled"

    await self.session.flush()
    return order
```

```python
# Route:
@app.post("/orders/{order_id}/cancel")
async def cancel_order(
    order_id: int,
    repo: OrderRepository = Depends(get_order_repository),
):
    try:
        order = await repo.cancel_order(order_id)
    except OrderNotFoundError:
        raise HTTPException(status_code=404, detail="Order not found")
    except OrderAlreadyCancelledError:
        raise HTTPException(status_code=409, detail="Order already cancelled")

    return {"id": order.id, "status": order.status}
```

**Why atomicity is guaranteed:** Everything happens within a single session, within a single transaction managed by the yield dependency. The stock updates (`item.product.stock += item.quantity`) are tracked by SQLAlchemy's unit of work. The status change (`order.status = "cancelled"`) is also tracked. `flush()` sends all `UPDATE` statements together. If anything fails, the dependency rolls back ALL changes — both the status and the stock. You get all-or-nothing without writing any transaction management code.

**Why we reuse `get_order_detail`:** We need the products loaded so we can update their stock. The eager loading chain (`items → product`) gives us direct access to `item.product.stock` without lazy loading.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise 4: When the Repository Pattern Hurts

You've spent this lecture learning the repository pattern: encapsulate all database queries in a class, inject it via `Depends()`, keep routes clean. It's an excellent pattern for most API applications.

**Your task:** Describe a realistic scenario where using the repository pattern is **worse** than putting queries directly in the code. Then write a small code snippet demonstrating why the pattern fails in this case.

**Guiding constraints:**
1. Your scenario must be a **real programming task**, not a contrived example.
2. You must show code both **with** and **without** the repository pattern.
3. Explain concretely what the repository pattern adds in terms of complexity, indirection, or performance cost — and why that cost isn't justified in your scenario.
4. Your argument should NOT be "the app is small." Small apps grow. Find a scenario where the repository pattern is structurally wrong, not just premature.

---

# Level 4 Solution

One genuinely problematic scenario: **one-off data migration scripts**.

Suppose you need to write a script that runs once, reads from two tables, transforms data, and writes to a third:

```python
# ❌ WITH repository pattern: over-engineered for a throwaway script

class UserRepository:
    def __init__(self, session: AsyncSession): ...
    async def get_all_active(self) -> list[User]: ...

class LegacyOrderRepository:
    def __init__(self, session: AsyncSession): ...
    async def get_by_user_id(self, user_id: int) -> list[LegacyOrder]: ...

class MigratedOrderRepository:
    def __init__(self, session: AsyncSession): ...
    async def create(self, user_id: int, total: float) -> MigratedOrder: ...


async def migrate():
    async with async_session_factory() as session:
        user_repo = UserRepository(session)
        legacy_repo = LegacyOrderRepository(session)
        migrated_repo = MigratedOrderRepository(session)

        users = await user_repo.get_all_active()
        for user in users:
            old_orders = await legacy_repo.get_by_user_id(user.id)
            for order in old_orders:
                await migrated_repo.create(
                    user_id=user.id, total=order.amount
                )
        await session.commit()
```

```python
# ✅ WITHOUT repository: simple and direct for a one-off script

async def migrate():
    async with async_session_factory() as session:
        users = (await session.execute(
            select(User).where(User.is_active == True)
        )).scalars().all()

        for user in users:
            old_orders = (await session.execute(
                select(LegacyOrder).where(LegacyOrder.user_id == user.id)
            )).scalars().all()

            for order in old_orders:
                session.add(MigratedOrder(user_id=user.id, total=order.amount))

        await session.commit()
        print(f"Migrated orders for {len(users)} users")
```

**Why the repository pattern hurts here:**

The repository pattern's value comes from **reuse** (same query called from multiple routes), **testability** (test DB logic without HTTP), and **separation** (routes don't know about SQL). In a one-off migration script:

- **Reuse:** These repository methods will never be called again. You wrote three classes for code that runs once and gets deleted.
- **Testability:** The migration is verified by running it against a test database and checking the results, not by unit-testing individual repository methods.
- **Separation:** There are no routes. There's no HTTP layer. The "separation" is between you and the database, and the repository just adds an indirection layer that makes the data flow harder to read in a single file.

The deeper structural issue: repositories are designed for **entity-centric operations** (CRUD on one entity type). Data migrations often cross entity boundaries in non-standard ways — reading from legacy tables, joining across schemas, writing computed aggregates. Forcing these operations into entity-centric repositories fragments the migration logic across multiple classes, making it harder to understand the transformation as a whole.

**When to use repositories vs. direct queries:**

| Use Repository | Use Direct Queries |
|---|---|
| Long-lived API endpoints | One-off scripts and migrations |
| Same query reused across multiple routes | Query used exactly once |
| Team collaboration (clear interfaces) | Solo scripts |
| Complex applications where routes and DB must evolve independently | Ad-hoc reports and analytics queries |

---

# LEVEL 5: THE TEACH BACK

## Exercise 5: The Sync Shortcut Debate

**Scenario:** A junior developer on your team reviews the codebase and sends you this message:

> "I've been reading about FastAPI and found that if you use `def` instead of `async def` for your route handlers, FastAPI automatically runs them in a thread pool. That means I can use regular sync SQLAlchemy with lazy loading — no `MissingGreenlet` errors, no `selectinload` everywhere, no `expire_on_commit=False`. It's way simpler. Why are we jumping through all these async hoops? I'm going to rewrite our routes to use sync and save everyone the headache."

They provide this code as evidence:

```python
from fastapi import FastAPI, Depends
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session, sessionmaker

engine = create_engine("postgresql://user:pass@localhost/taskdb")
SessionLocal = sessionmaker(bind=engine)

app = FastAPI()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


@app.get("/tasks/{task_id}")     # <-- def, NOT async def
def get_task(task_id: int, db: Session = Depends(get_db)):
    task = db.query(Task).filter(Task.id == task_id).first()
    if not task:
        raise HTTPException(404)
    # Lazy loading works! No MissingGreenlet!
    category_name = task.category.name if task.category else None
    return {"id": task.id, "title": task.title, "category": category_name}
```

**Your task:**

1. **Acknowledge what they got right.** The junior developer isn't entirely wrong — their code does work. Explain the mechanism: why does `def` instead of `async def` allow sync SQLAlchemy to work in FastAPI?

2. **Explain what they got wrong.** Why is the "just use sync" approach problematic for a production backend that needs to handle hundreds of concurrent requests? Be specific about the performance implications. Use the "restaurant analogy" or the "concurrent requests timeline" from the lecture to make it concrete.

3. **Address the "it's simpler" argument.** The junior developer is right that async SQLAlchemy requires more setup (`expire_on_commit`, `selectinload`, two-step query pattern). But they're wrong that "simpler code" always means "better code." Explain what you GAIN from the explicit data-loading model that async enforces, even beyond performance.

4. **Give them a fair answer.** Are there situations where their approach (sync routes + sync SQLAlchemy) is genuinely the better choice? Be honest — don't just say "async is always better."

---

# Level 5 Solution

**1. What they got right:**

Their code is correct, and the mechanism is real. When you declare a route with `def` (not `async def`), FastAPI detects that it's a synchronous function and runs it in a **thread pool** (specifically, `anyio`'s default thread pool via `run_in_threadpool`). This means the sync route runs on a separate thread, NOT on the event loop thread. Sync SQLAlchemy's blocking calls block that worker thread, but the event loop remains free to handle other requests.

The lazy loading works because the sync `Session` makes sync database calls, which are perfectly legal inside a sync thread. There is no `MissingGreenlet` because there's no `AsyncSession` involved.

**2. What they got wrong:**

Thread pools have a **fixed size** (typically 40 threads in the default configuration). Each sync route handler occupies one thread for its entire duration — including the time spent waiting for database responses.

Consider their route handling 100 concurrent requests, each taking 50ms of database time:
- **Async approach:** All 100 requests share the event loop. During each `await session.execute()`, the coroutine yields control. The event loop juggles all 100 overlapping database waits. Total time ≈ 50-80ms for all 100 requests.
- **Sync thread pool approach:** The first 40 requests each claim a thread. The remaining 60 requests **queue** until a thread becomes free. Total time ≈ 50ms × (100/40) ≈ 125ms for the last requests in the queue. Under heavier load, the queue grows and response times degrade linearly.

The async approach achieves **concurrency through cooperative scheduling** (lightweight, thousands of coroutines). The sync approach achieves concurrency through **OS threads** (heavyweight, dozens at most). Threads consume memory (typically ~1-8MB per thread stack), have context-switch overhead, and are managed by the OS kernel rather than your application.

The restaurant analogy: sync routes give you 40 waiters who each stand at the kitchen window waiting for their order. Async gives you one waiter who takes 100 orders and gets notified as each one is ready. Under light load, both work. Under heavy load, you need 400 waiters (threads) or one very efficient waiter (event loop).

**3. The hidden benefit of async's "complexity":**

The junior developer frames `selectinload` and `expire_on_commit=False` as annoying boilerplate. But these are actually **forcing good architectural decisions**:

- **`selectinload` makes N+1 impossible to hide.** In sync code, lazy loading silently fires hundreds of queries that you never see until production monitoring reveals a 10-second endpoint. Async SQLAlchemy makes you declare your data needs upfront. If you forget a `selectinload`, you get a loud crash immediately — not a silent performance regression that shows up months later.

- **`expire_on_commit=False` makes session boundaries explicit.** In sync code, you can accidentally access attributes after the session is closed and get a "detached instance" error at runtime. In async, the strict model forces you to think about what data the response needs BEFORE you close the session.

- **The repository pattern exists because SQL in routes doesn't scale.** Async or sync, you'll eventually need it. The async approach forces you to adopt proper architecture from the start because the alternative (SQL scattered in routes with lazy loading) simply doesn't work.

**4. When sync IS the better choice:**

- **Prototyping and MVPs:** If you're building a proof-of-concept with <10 concurrent users, the async complexity genuinely isn't worth it. Ship fast, optimize later.
- **Background scripts and CLIs:** One-off data migrations, management commands, and scripts that run sequentially by nature. No concurrency = no benefit from async.
- **CPU-bound routes:** If your route mostly does computation (image processing, ML inference) rather than I/O, async provides no benefit — and the thread pool approach is arguably better because it actually parallelizes CPU work across cores (via OS scheduling), whereas async on one event loop thread cannot.
- **Teams unfamiliar with async:** If the team's debugging skills and mental models are built entirely around sync Python, introducing async creates a maintenance burden that may outweigh the performance gains for moderate-traffic applications.

The honest answer: for a backend serving <100 concurrent users with moderate database complexity, sync SQLAlchemy with `def` routes is a legitimate and pragmatic choice. For a backend that needs to scale to hundreds or thousands of concurrent connections — which is the target for most production SaaS backends — async is the right foundation, and the upfront complexity pays for itself in performance, explicit data patterns, and architectural discipline.