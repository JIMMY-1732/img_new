# PHASE 0: TOPIC ANALYSIS

## The Main Opponent (Prior Mental Model)

**Naive Belief:** "Connection pooling is invisible infrastructure that takes care of itself. Database transactions automatically prevent data conflicts. Inserting records in a loop is fine — the ORM optimizes it under the hood. The `organization_id` filter is a convenience for searching; skipping it on a single row lookup only affects what you see, not your security boundary."

**Specific Misconceptions:**

| # | Misconception | Reality |
|---|---|---|
| 1 | `async with AsyncSessionLocal()` creates a brand new database connection each call | It borrows from a fixed pool; `async with` is what **guarantees the return** |
| 2 | `asyncio.gather()` means coroutines "share" connections efficiently — nobody starves | All gathered coroutines attempt to check out connections **simultaneously**; async does not conjure new connections |
| 3 | `execute(update(Task).where(...))` raises an exception when 0 rows match | It silently succeeds with `rowcount == 0`; the developer must check this manually |
| 4 | Optimistic locking **retries automatically** when a version conflict occurs | The developer checks `rowcount`, raises an error, and implements any retry logic themselves |
| 5 | `SELECT ... FOR UPDATE` locks the entire table | It locks only the **specific rows returned** by that query |
| 6 | `session.add_all(objects)` sends one batched `INSERT` statement | It stages Python objects; actual SQL behavior depends heavily on whether `flush()` is called in a loop |
| 7 | `execute(insert(Model), rows)` returns a list of ORM model instances | It returns a `CursorResult`; no ORM objects exist unless you use `.returning()` |
| 8 | Skipping `organization_id` in `get_by_id` only shows the wrong row — it cannot cause mutations | Any method that **delegates to an unfiltered** `get_by_id` (update, delete) also crosses tenant boundaries |
| 9 | `pool_size=80` is a safe fix for `QueuePool limit reached` errors | Each connection × instances must stay under PostgreSQL's `max_connections`; greedier pools cause complete outages |

---

## The Main Purpose (Essence)

**Core Insight:** Every pattern in this lecture is the surgical answer to exactly one production failure mode.

| Failure Mode | Pattern | Mechanism |
|---|---|---|
| Server crashes under load | Connection Pooling | Reuse and queue connections; timeout and report excess requests |
| Data corruption under concurrency | Locking | Pessimistic: block at read. Optimistic: detect at write, check `rowcount` |
| Catastrophically slow writes | Bulk Operations | One network round-trip for N rows instead of N round-trips |
| Cross-tenant data breach | Multi-Tenancy Isolation | Every query must carry `organization_id`; architecture must make forgetting impossible |

**Key Distinctions:**
- **Pessimistic** (`FOR UPDATE`): "Conflict is likely. Block others now." Lock held during your operation — milliseconds max.
- **Optimistic** (version column): "Conflict is rare. Check at write time." No lock held; detect conflict only via `result.rowcount == 0`.
- **`add_all()`**: Creates tracked ORM objects, returns them with populated IDs. Best for hundreds of rows where you need ORM features.
- **`execute(insert(Model), [...])`**: Bypasses ORM tracking entirely, fastest path, returns `CursorResult` — not ORM objects — unless you call `.returning()`.
- **`organization_id` filter**: Not a search convenience — it is a **security boundary**. Missing it in one query is a data breach, not a bug.

---

## Prediction Collision Map

| Naive Belief | Leads to Prediction | Actual Behavior | Root Cause |
|---|---|---|---|
| "Async shares connections fairly" | 3 concurrent jobs with pool_size=2 all succeed | 3rd job raises `TimeoutError` after `pool_timeout` | Async concurrency ≠ connection sharing |
| "execute(update) raises if nothing matches" | Code crashes on version mismatch | Silently returns `rowcount=0`; caller returns `True` | SQL `UPDATE` matching 0 rows is not an error |
| "add_all + flush per batch is efficient" | Batch loop = fast | Each `flush()` call = one network round-trip | `flush()` sends SQL NOW, not at commit |
| "hybrid_property filters work everywhere" | `where(Task.progress >= 50)` works in SQL | `NotImplementedError` — no `.expression` defined | Python method cannot be translated to SQL without explicit expression |
| "org_id filter optional for single-row lookup" | Missing filter shows wrong row, nothing worse | `delete_task(999)` deletes Org B's data from Org A's repo | `update` and `delete` delegate to unfiltered `get_by_id` |

---

# LEVEL 1: PREDICT & EXPLAIN

## Exercise 1.1: The Async Pool Trap

```python
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from sqlalchemy import text

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/taskmanager",
    pool_size=2,
    max_overflow=0,
    pool_timeout=2,  # Wait at most 2 seconds for a free connection
)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession)


async def export_report(report_id: int) -> str:
    async with AsyncSessionLocal() as session:
        # Simulate a slow analytics query that holds the connection open
        await session.execute(text("SELECT pg_sleep(3)"))
        return f"Report {report_id} ready"


async def main():
    results = await asyncio.gather(
        export_report(1),
        export_report(2),
        export_report(3),
        return_exceptions=True,
    )
    for i, result in enumerate(results, 1):
        print(f"Job {i}: {result}")


asyncio.run(main())
```

**Questions:**

1. **Predict the output.** `pool_size=2`, `max_overflow=0`, `pool_timeout=2`. Three export jobs fire concurrently via `asyncio.gather()`. Each holds a connection for 3 seconds. Which jobs succeed? Which fail? Write the predicted output line by line.

2. **Explain why.** A classmate argues: *"Python's async model means all three coroutines run concurrently and share the two connections efficiently — they take turns, so nobody should fail."* Break down exactly why this reasoning is wrong. At what precise moment does each job attempt to acquire a connection, and how is "acquiring a connection" different from "waiting for I/O"?

3. **Diagnose the fix.** Without changing `pg_sleep(3)` — the query cannot be made faster — name the **minimum change** to the engine configuration that allows all 3 jobs to complete. State the tradeoff your fix introduces in a production environment running 5 API instances.

---

## Exercise 1.2: The Silent Overwrite

**Setup:** A task currently exists in the database with `id=1, title="Draft Report", version=3`.

```python
from sqlalchemy import update
from sqlalchemy.ext.asyncio import AsyncSession
from models import Task


async def save_task_edit(
    session: AsyncSession,
    task_id: int,
    new_title: str,
    client_version: int,
) -> dict:
    stmt = (
        update(Task)
        .where(Task.id == task_id, Task.version == client_version)
        .values(title=new_title, version=client_version + 1)
    )
    await session.execute(stmt)
    await session.commit()
    return {"saved": True, "new_version": client_version + 1}


async def simulate_concurrent_edits():
    # Alice saves first with version=3 (she read it at version=3)
    async with AsyncSessionLocal() as session_a:
        result_alice = await save_task_edit(
            session_a, task_id=1, new_title="Alice's Final Version", client_version=3
        )

    # Bob saves a moment later — also with version=3 (he also read it at version=3)
    async with AsyncSessionLocal() as session_b:
        result_bob = await save_task_edit(
            session_b, task_id=1, new_title="Bob's Final Version", client_version=3
        )

    print(f"Alice: {result_alice}")
    print(f"Bob:   {result_bob}")

    async with AsyncSessionLocal() as session:
        task = await session.get(Task, 1)
        print(f"Final DB: title='{task.title}', version={task.version}")
```

**Questions:**

1. **Predict the output.** Trace through both saves. What does `result_alice` contain? What does `result_bob` contain? What are the final `title` and `version` in the database after both commits?

2. **Explain the bug.** Both Alice and Bob receive `{"saved": True, ...}`. Identify the critical check that is missing. Why is returning `True` unconditionally dangerous — specifically, what does `session.execute(update(...))` return when zero rows are matched, and where does `save_task_edit` fail to inspect this?

3. **Rewrite the function.** Fix `save_task_edit` so it raises an `OptimisticLockError` when a version conflict is detected. Include the exact condition you check and why it proves a conflict occurred.

---

## Exercise 1.3: Counting Round Trips

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import insert
from models import Task


async def import_with_batches(session: AsyncSession, all_rows: list[dict]) -> int:
    BATCH_SIZE = 500

    for start in range(0, len(all_rows), BATCH_SIZE):
        batch = all_rows[start : start + BATCH_SIZE]
        objects = [Task(**row) for row in batch]
        session.add_all(objects)
        await session.flush()  # Developer comment: "commits each batch safely"

    await session.commit()
    return len(all_rows)
```

**Questions:**

1. **Predict the network behavior.** `all_rows` contains 3,000 rows. `BATCH_SIZE=500`. Answer specifically:
   - How many times does `flush()` get called?
   - How many SQL statements does SQLAlchemy send to PostgreSQL?
   - How many network round-trips to the database does this produce?

2. **Identify the misconception.** The developer added `flush()` inside the loop believing it prevents SQLAlchemy from "holding too much in memory at once." Explain what `flush()` actually does and why calling it in a loop is not a memory solution — and is, in fact, a performance regression.

3. **Rank three approaches.** Without running the code, rank these three implementations from **slowest to fastest** for importing 3,000 rows, and explain the mechanical reason for each ranking:
   - **Method A**: `session.add(Task(**row))` + `await session.flush()` per row, in a `for` loop
   - **Method B**: The `import_with_batches` function above (add_all + flush per 500)
   - **Method C**: `await session.execute(insert(Task), all_rows)` + `await session.commit()` once

---

## Exercise 1.4: The Hybrid That Can Only See Half

```python
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import Mapped, mapped_column, DeclarativeBase
from sqlalchemy import String


class Base(DeclarativeBase):
    pass


class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    total_steps: Mapped[int] = mapped_column(default=0)
    completed_steps: Mapped[int] = mapped_column(default=0)

    @hybrid_property
    def progress_percent(self) -> float:
        if self.total_steps == 0:
            return 0.0
        return (self.completed_steps / self.total_steps) * 100


# ── Usage A: On a Python object ────────────────────────────────
task = Task(total_steps=10, completed_steps=7)
print(f"Task progress: {task.progress_percent}%")

# ── Usage B: As a SQL filter ───────────────────────────────────
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

async def get_near_complete_tasks(session: AsyncSession) -> list[Task]:
    stmt = select(Task).where(Task.progress_percent >= 75.0)
    result = await session.execute(stmt)
    return list(result.scalars().all())
```

**Questions:**

1. **Predict Usage A.** Does `task.progress_percent` work? What does it print? Explain which part of the class definition makes this work.

2. **Predict Usage B.** Does `Task.progress_percent >= 75.0` inside `select().where(...)` work or raise an error? If it raises an error, what kind and why? Be specific about what SQLAlchemy is trying and failing to do.

3. **Fix the class.** You cannot change the database schema or add new columns. Fix the `Task` class so that `Task.progress_percent >= 75.0` works as a valid SQL `WHERE` clause. Write only the addition needed — not the whole class.

---

# Level 1 Solutions

## Solution 1.1: The Async Pool Trap

**1. Predicted output:**
```
Job 1: Report 1 ready
Job 2: Report 2 ready
Job 3: TimeoutError: QueuePool limit of size 2 overflow 0 reached,
       connection timed out, timeout 2.00
```
Jobs 1 and 2 immediately acquire the 2 available connections. Job 3 enters the queue and waits up to `pool_timeout=2` seconds for a free connection. Since each connection is held for 3 seconds (the `pg_sleep` duration), no connection is freed within 2 seconds. Job 3 raises `TimeoutError`.

**2. Why "sharing efficiently" is wrong:**
`asyncio.gather()` launches all three coroutines, and all three reach `async with AsyncSessionLocal() as session` at nearly the same instant. The `async with` block calls `session.__aenter__()`, which immediately checks out a connection from the pool. This checkout is synchronous from the pool's perspective — it either acquires a slot right now or waits. There is no "turn-taking." A connection occupied by a running query is not available to any other coroutine. Async concurrency allows coroutines to suspend while waiting for I/O, but a connection is a serial, exclusive resource. Two queries need two connections, always.

**3. Minimum fix and tradeoff:**
```python
engine = create_async_engine(
    "...",
    pool_size=2,
    max_overflow=1,   # Allow 1 extra temporary connection → total capacity = 3
    pool_timeout=2,
)
```
Alternatively: `pool_timeout=5` gives Job 3 enough time to wait for Job 1 or 2 to release their connection.

**Tradeoff of increasing `max_overflow`:** In a multi-instance deployment, each overflow connection consumes a slot in PostgreSQL's `max_connections=100`. With 5 API instances each set to `pool_size=2, max_overflow=1`, you have up to 15 simultaneous connections. The "minimum fix" is safe here, but it sets a precedent for reaching into overflow settings without calculating the total across all instances.

**The Key Insight:** `async/await` enables concurrency by suspending while *waiting for I/O responses*. It does not allow multiple coroutines to share a single active connection. The moment a coroutine calls `async with AsyncSessionLocal()`, it claims one slot. Async cannot conjure connections that don't exist in the pool.

---

## Solution 1.2: The Silent Overwrite

**1. Predicted output:**
```
Alice: {'saved': True, 'new_version': 4}
Bob:   {'saved': True, 'new_version': 4}
Final DB: title="Alice's Final Version", version=4
```
Alice's update succeeds: `WHERE id=1 AND version=3` matches, sets `version=4`. After Alice commits, the database row has `version=4`. Bob's update runs `WHERE id=1 AND version=3` — but the row now has `version=4`. Zero rows are matched. PostgreSQL returns `rowcount=0`. The function never inspects this and unconditionally returns `{"saved": True, "new_version": 4}`. Bob's client believes his save succeeded. His title was silently discarded.

**2. The missing check:**
`session.execute(update(...))` returns a `CursorResult`. When a `WHERE` clause matches zero rows, PostgreSQL executes the statement successfully and reports `rowcount=0`. This is **not** a database error — it is a valid SQL outcome. SQLAlchemy raises no exception. `save_task_edit` never reads `result.rowcount`, so it cannot distinguish "1 row updated" from "0 rows updated." Returning `True` unconditionally means a real data loss event is reported to the user as a success.

**3. Fixed function:**
```python
class OptimisticLockError(Exception):
    pass


async def save_task_edit(
    session: AsyncSession,
    task_id: int,
    new_title: str,
    client_version: int,
) -> dict:
    stmt = (
        update(Task)
        .where(Task.id == task_id, Task.version == client_version)
        .values(title=new_title, version=client_version + 1)
    )
    result = await session.execute(stmt)

    if result.rowcount == 0:
        # Either the task doesn't exist, or someone else already modified it.
        raise OptimisticLockError(
            f"Task {task_id} was modified since version {client_version}. "
            "Re-read the task and try again."
        )

    await session.commit()
    return {"saved": True, "new_version": client_version + 1}
```

**The Key Insight:** `execute(update(...))` is designed to succeed silently when nothing matches. The version-column check is your application-level contract, and `result.rowcount` is the only receipt the database gives you to verify it was honored. Every optimistic lock implementation must read `rowcount`. There is no shortcut.

---

## Solution 1.3: Counting Round Trips

**1. Network behavior for 3,000 rows with BATCH_SIZE=500:**
- `flush()` calls: **6** (one per batch: 3000 ÷ 500)
- SQL statements sent: SQLAlchemy's ORM with `add_all` may issue individual `INSERT` statements per object. In the worst case: **3,000 INSERT statements**. With asyncpg pipelining, they may be batched into fewer round-trips, but each `flush()` forces transmission of all pending statements before continuing.
- Network round-trips: **at minimum 6** (one per flush), but the overhead is dominated by the 3,000 individual statement executions on the PostgreSQL side, regardless of pipelining.

**2. What `flush()` actually does:**
`flush()` instructs SQLAlchemy: "Send all pending changes to the database right now, but keep the transaction open." It does not reduce memory usage — the Python ORM objects created by `Task(**row)` still exist in memory whether you flush or not. Calling `flush()` inside the loop forces 6 separate synchronization points with the database. Each `flush()` is a network call. The developer has paid for 6 round-trips in exchange for zero memory benefit.

**3. Ranking — slowest to fastest:**

1. **Method A** (slowest): 3,000 `flush()` calls = 3,000 network round-trips. Each round-trip includes the full SQL parse-plan-execute cycle on PostgreSQL. This is the N+1 write problem.

2. **Method B** (middle): 6 `flush()` calls = 6 network round-trips. Significantly better than A, but still creates 3,000 Python ORM objects, and the ORM sends individual INSERT statements within each flush block. The batch reduction helps, but the per-object ORM overhead remains.

3. **Method C** (fastest): 1 `commit()` = 1 network round-trip. SQLAlchemy Core generates a single `INSERT INTO tasks (...) VALUES ($1,$2), ($3,$4)...` statement. No ORM object creation, no per-row overhead, no intermediate flushes. One statement, one trip.

**The Key Insight:** `session.add_all()` and `session.add()` differ only in Python-side staging. From the database's perspective, both still send individual INSERT statements unless you use Core's `execute(insert(Model), rows)`. The only path to a genuinely batched single-statement INSERT is the Core API.

---

## Solution 1.4: The Hybrid That Can Only See Half

**1. Usage A — Python object:**
It works correctly and prints `Task progress: 70.0%`. When accessed on a Python instance, `@hybrid_property` calls the method body directly: `(7 / 10) * 100 = 70.0`. No SQL involved.

**2. Usage B — SQL filter:**
It raises `NotImplementedError` (or an `ArgumentError` depending on SQLAlchemy version). When `Task.progress_percent` is accessed on the *class* (not an instance), SQLAlchemy looks for the `.expression` class method to translate the property into a SQL clause. Since no `@progress_percent.expression` is defined, SQLAlchemy has no SQL equivalent for the Python calculation and cannot include it in a `WHERE` clause.

**3. The fix — add the `.expression` side:**
```python
from sqlalchemy import case
from sqlalchemy.ext.hybrid import hybrid_property

# Add this to the Task class, immediately after the existing @hybrid_property block:

@progress_percent.expression
@classmethod
def progress_percent(cls):
    """SQL translation: used when Task.progress_percent appears in a query."""
    return case(
        (cls.total_steps == 0, 0.0),
        else_=(cls.completed_steps * 100.0 / cls.total_steps),
    )
```

**The Key Insight:** A hybrid property has two personalities — a Python method and a SQL expression. The `@hybrid_property` defines the Python side; the `@<name>.expression` class method defines the SQL side. Without the expression side, the property exists only in Python memory and cannot be evaluated inside a database query. They must both be present for the property to be truly "hybrid."

---

# LEVEL 2: DEBUG, FIX, & VERIFY

## Exercise 2.1: The Pool That Slowly Starves

**Context:** A developer wrote a helper to fetch a task with retry logic. Under normal load it works fine. After running in production for a few hours, the API begins throwing `TimeoutError: QueuePool limit reached` errors across all endpoints — not just ones using this function.

```python
import asyncio
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from models import Task


async def fetch_task_safe(task_id: int) -> Task | None:
    """Fetch a task, retrying once on transient connection errors."""
    for attempt in range(2):
        session = AsyncSessionLocal()
        try:
            result = await session.execute(
                select(Task).where(Task.id == task_id)
            )
            task = result.scalar_one_or_none()
            await session.close()
            return task
        except Exception:
            if attempt == 0:
                await asyncio.sleep(0.5)
                continue  # Retry — but what happened to the session?
            raise
    return None
```

**Task 2a: Identify the flaw**

Read the function carefully. Under what **exact condition** does a connection get borrowed from the pool and **never returned**? Name the specific code path and the line that is never reached.

---

**Task 2b: Explain the mechanism**

1. If `fetch_task_safe` is called 1,000 times and each call triggers the retry path (i.e., `session.execute()` raises on the first attempt), what happens to `engine.pool.checkedout()` over those 1,000 calls?
2. Why does Python's garbage collector not fix this automatically, even though the abandoned `session` object eventually goes out of scope?

---

**Task 2c: Instrument to prove diagnosis**

Fill in the instrumentation below and show the expected output that proves the leak is occurring:

```python
async def demonstrate_leak(num_calls: int = 5):
    print(f"Before calls: checked_out = {engine.pool.checkedout()}")

    for i in range(num_calls):
        try:
            # Assume this triggers the retry path (execute raises on attempt 0)
            await fetch_task_safe(task_id=i)
        except Exception:
            pass

    print(f"After calls:  checked_out = {engine.pool.checkedout()}")
    # If leaking:    "After calls:  checked_out = ?"
    # If not leaking: "After calls:  checked_out = ?"
```

State the expected values for both the leaking and non-leaking versions.

---

**Task 2d: Fix the code**

Rewrite `fetch_task_safe` so that the connection is returned to the pool in **every possible code path** — normal return, exception on first attempt, and exception on second attempt.

---

**Task 2e: Write a verification test**

Write a test that fails on the original code and passes on your fixed version:

```python
@pytest.mark.asyncio
async def test_no_connection_leak_on_retry():
    initial_checked_out = engine.pool.checkedout()

    # Make 10 calls that each trigger the retry path
    # (Use unittest.mock to make session.execute raise on the first attempt)
    for _ in range(10):
        try:
            await fetch_task_safe(task_id=-99)
        except Exception:
            pass

    # Your assertion here
    assert engine.pool.checkedout() == ???
```

---

## Exercise 2.2: The Repository With a Hole

**Context:** A team built a `TaskRepository` enforcing multi-tenant data isolation. During a security audit, you are asked to review it. Read all four methods carefully before answering.

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from models import Task


class TaskRepository:
    def __init__(self, session: AsyncSession, org_id: int):
        self._session = session
        self._org_id = org_id

    async def list_tasks(self) -> list[Task]:
        stmt = (
            select(Task)
            .where(Task.organization_id == self._org_id)
            .order_by(Task.created_at.desc())
        )
        result = await self._session.execute(stmt)
        return list(result.scalars().all())

    async def get_task(self, task_id: int) -> Task | None:
        """Fetch a single task by ID."""
        stmt = select(Task).where(Task.id == task_id)
        result = await self._session.execute(stmt)
        return result.scalar_one_or_none()

    async def update_task_status(
        self, task_id: int, new_status: str
    ) -> Task | None:
        task = await self.get_task(task_id)
        if task is None:
            return None
        task.status = new_status
        await self._session.commit()
        return task

    async def delete_task(self, task_id: int) -> bool:
        task = await self.get_task(task_id)
        if task is None:
            return False
        await self._session.delete(task)
        await self._session.commit()
        return True
```

**Task 2a: Identify the flaw**

Which single method contains the security vulnerability? What specific line is missing from it?

---

**Task 2b: Explain the mechanism**

Trace the following attack scenario step by step, showing what executes at each line:
- `org1_repo = TaskRepository(session, org_id=1)`
- Task `id=999` belongs to `organization_id=2`
- Attack call: `await org1_repo.delete_task(999)`

Does the deletion succeed? What is the final state of task 999 in the database?

---

**Task 2c: Instrument to prove diagnosis**

Complete the test assertions below to document the vulnerability:

```python
@pytest.mark.asyncio
async def test_cross_tenant_delete_vulnerability():
    async with AsyncSessionLocal() as session:
        # Setup: Create a task for org 2
        org2_task = Task(title="Org 2 Secret", organization_id=2, status="pending")
        session.add(org2_task)
        await session.commit()
        await session.refresh(org2_task)

        # Attack: Use org 1's repository to delete org 2's task
        org1_repo = TaskRepository(session, org_id=1)
        result = await org1_repo.delete_task(org2_task.id)

        # Fill in expected assertions — what actually happens:
        assert result == ???       # Did delete_task() return True or False?

    # Verify database state in a fresh session
    async with AsyncSessionLocal() as verify_session:
        check = await verify_session.get(Task, org2_task.id)
        assert check is ???        # Does the task still exist?
```

---

**Task 2d: Fix the code**

Fix the repository so that:
1. `get_task()` enforces tenant isolation
2. `update_task_status()` and `delete_task()` are automatically protected without any changes to their code
3. The fix lives in **exactly one place** (the architecture propagates it)

---

**Task 2e: Write a verification test**

```python
@pytest.mark.asyncio
async def test_cross_tenant_access_fully_blocked():
    """After the fix, Org 1 cannot read, modify, or delete Org 2's tasks."""
    async with AsyncSessionLocal() as session:
        org2_task = Task(title="Org 2 Private", organization_id=2, status="pending")
        session.add(org2_task)
        await session.commit()
        await session.refresh(org2_task)

        org1_repo = TaskRepository(session, org_id=1)

        # All three operations must behave as if the task does not exist
        assert await org1_repo.get_task(org2_task.id) is None
        assert await org1_repo.update_task_status(org2_task.id, "done") is None
        assert await org1_repo.delete_task(org2_task.id) is False

    # Verify the task is completely untouched
    async with AsyncSessionLocal() as verify_session:
        intact = await verify_session.get(Task, org2_task.id)
        assert intact is not None
        assert intact.status == "pending"   # Not modified
        assert intact.title == "Org 2 Private"
```

---

# Level 2 Solutions

## Solution 2.1: The Pool That Slowly Starves

**Task 2a — The leak condition:**
The connection is leaked when `session.execute()` raises an exception on the **first attempt** (`attempt == 0`). The code path:
1. `session = AsyncSessionLocal()` → connection checked out from pool
2. `await session.execute(...)` → raises an exception
3. `except Exception:` → caught
4. `if attempt == 0: continue` → jumps back to the top of the for loop
5. `await session.close()` is **never reached** — it is inside the `try` block after the line that raised

On the next loop iteration, a brand new `session = AsyncSessionLocal()` is created, checking out a second connection. The first connection is lost.

**Task 2b — Mechanism:**
After 1,000 retry-triggering calls, `engine.pool.checkedout()` grows to 1,000 (or until pool exhaustion begins raising `TimeoutError` for other endpoints). Python's garbage collector does not reliably fix this because:
- `async` context managers require `__aexit__` to be explicitly called. GC's `__del__` is not guaranteed to call it promptly, or correctly, for async resources.
- When the GC does collect an `AsyncSession` object, the connection return path through `asyncpg` may not execute in the right async context to safely return the connection. This makes GC-based cleanup unreliable for pooled async resources.

**Task 2c — Instrumentation output:**
```
# Leaking version:
Before calls: checked_out = 0
After calls:  checked_out = 5

# Fixed version:
Before calls: checked_out = 0
After calls:  checked_out = 0
```

**Task 2d — Fixed code:**
```python
async def fetch_task_safe(task_id: int) -> Task | None:
    """Fetch a task, retrying once on transient errors. No connection leaks."""
    last_exception: Exception | None = None

    for attempt in range(2):
        # async with guarantees __aexit__ runs in ALL exit paths:
        # normal return, exception, even asyncio.CancelledError.
        try:
            async with AsyncSessionLocal() as session:
                result = await session.execute(
                    select(Task).where(Task.id == task_id)
                )
                return result.scalar_one_or_none()
                # Connection returned to pool here, via async with __aexit__
        except Exception as e:
            last_exception = e
            if attempt == 0:
                await asyncio.sleep(0.5)
                continue
            raise

    return None
```

**Task 2e — Verification test:**
```python
@pytest.mark.asyncio
async def test_no_connection_leak_on_retry():
    initial_checked_out = engine.pool.checkedout()

    for _ in range(10):
        try:
            await fetch_task_safe(task_id=-99)
        except Exception:
            pass

    assert engine.pool.checkedout() == initial_checked_out, (
        f"Connection leak: {engine.pool.checkedout() - initial_checked_out} "
        "connections not returned to pool"
    )
```

**The Key Insight:** `async with` is not stylistic preference — it is a correctness guarantee. It binds `__aexit__` to exit via any mechanism: normal return, exception, `return` mid-block, even `asyncio.CancelledError`. Manual `.close()` calls inside `try/except` always leave edge cases where they are skipped. The invariant is simple: every session must be created with `async with`. No exceptions, no exceptions to the rule.

---

## Solution 2.2: The Repository With a Hole

**Task 2a — The flaw:**
`get_task()` queries `WHERE id = :task_id` without `AND organization_id = :org_id`. It returns any task with that ID, regardless of which tenant owns it.

**Task 2b — Attack trace:**
```
1. org1_repo.delete_task(999) called
2. → await self.get_task(999)
3.   → SELECT * FROM tasks WHERE id = 999
        ← NO organization_id filter — returns Org 2's task object
4. → task is not None  ← security boundary silently crossed
5. → await self._session.delete(task)  ← deletes Org 2's data
6. → await self._session.commit()      ← permanently deleted
7. → returns True                      ← caller believes it succeeded
```
The deletion succeeds completely. Task 999 is gone from the database. Org 2's data has been destroyed by Org 1's repository instance.

**Task 2c — Assertion values:**
```python
assert result == True   # delete_task returns True — "succeeded"

check = await verify_session.get(Task, org2_task.id)
assert check is None    # Task was permanently deleted
```

**Task 2d — Fix:**
```python
async def get_task(self, task_id: int) -> Task | None:
    """Fetch a single task by ID — scoped to this organization."""
    stmt = select(Task).where(
        Task.id == task_id,
        Task.organization_id == self._org_id,  # ← The one line that was missing
    )
    result = await self._session.execute(stmt)
    return result.scalar_one_or_none()
```
`update_task_status()` and `delete_task()` delegate to `get_task()`, so fixing `get_task()` automatically secures both with zero changes to their code. This is the architectural value of the delegation pattern — the fix propagates by design.

**The Key Insight:** The `organization_id` filter is not a search constraint. It is a **security boundary**. A `get_by_id` without a tenant filter is not "slightly wrong" — it is cross-tenant data access. The repository takes `org_id` in `__init__` precisely to create one place where the boundary is enforced. If you can call `get_task()` without passing an org, the architecture has already failed. The invariant must be structural, not behavioral.

---

# LEVEL 2.5: EXTEND EXISTING CODE

## Exercise 2.5: The Bulk Import Pipeline

**Background:** Your Task Manager imports tasks from external sources. The following function is correct, passes all tests, and is in production.

```python
from sqlalchemy import insert
from sqlalchemy.ext.asyncio import AsyncSession
from models import Task


async def bulk_import_tasks(
    session: AsyncSession,
    tasks_data: list[dict],
    organization_id: int,
) -> int:
    """
    Bulk import tasks from an external data source.
    Returns the number of tasks inserted.
    """
    if not tasks_data:
        return 0

    rows = [
        {**data, "organization_id": organization_id}
        for data in tasks_data
    ]

    await session.execute(insert(Task), rows)
    await session.commit()
    return len(rows)


# ── Existing tests — all must still pass after every extension ─────────

@pytest.mark.asyncio
async def test_bulk_import_inserts_correct_count():
    async with AsyncSessionLocal() as session:
        count = await bulk_import_tasks(
            session,
            [{"title": "Task A", "status": "pending"},
             {"title": "Task B", "status": "pending"}],
            organization_id=1,
        )
        assert count == 2


@pytest.mark.asyncio
async def test_bulk_import_empty_list_returns_zero():
    async with AsyncSessionLocal() as session:
        count = await bulk_import_tasks(session, [], organization_id=1)
        assert count == 0


@pytest.mark.asyncio
async def test_bulk_import_injects_org_id():
    async with AsyncSessionLocal() as session:
        await bulk_import_tasks(
            session,
            [{"title": "Org Task", "status": "pending"}],
            organization_id=42,
        )

    async with AsyncSessionLocal() as verify:
        result = await verify.execute(
            select(Task).where(Task.title == "Org Task")
        )
        task = result.scalar_one()
        assert task.organization_id == 42
```

---

**Extension 1: Handle Massive Imports Without Blocking**

**Problem:** The current function sends all rows in one statement. For 500,000 rows this creates a transaction that can run for minutes, blocking all concurrent writes to the `tasks` table and potentially exhausting database memory.

**Task:** Add a `batch_size: int = 2000` parameter. When `len(tasks_data) > batch_size`, split the import into batches and commit after each batch. When `len(tasks_data) <= batch_size`, behavior must be identical to the current implementation.

**New test to pass:**
```python
@pytest.mark.asyncio
async def test_large_import_uses_multiple_commits():
    commit_count = 0
    original_commit = AsyncSession.commit

    async def counting_commit(self):
        nonlocal commit_count
        commit_count += 1
        return await original_commit(self)

    with patch.object(AsyncSession, "commit", counting_commit):
        async with AsyncSessionLocal() as session:
            data = [{"title": f"T{i}", "status": "pending"} for i in range(5000)]
            count = await bulk_import_tasks(
                session, data, organization_id=1, batch_size=1000
            )

    assert count == 5000
    assert commit_count == 5  # 5000 ÷ 1000 = 5 commits
```

---

**Extension 2: Handle Duplicate External Records Gracefully**

**Problem:** The `Task` model has an optional `external_id: str` column with a composite unique constraint on `(organization_id, external_id)`. Re-running the same import currently raises an `IntegrityError` that aborts the entire batch.

**Task:** Add an `on_conflict: str = "error"` parameter. When `on_conflict="skip"`, rows that would violate the unique constraint are silently ignored using PostgreSQL's `ON CONFLICT DO NOTHING`. When `on_conflict="error"` (the default), existing behavior is preserved.

**New test to pass:**
```python
@pytest.mark.asyncio
async def test_duplicate_external_ids_are_skipped():
    async with AsyncSessionLocal() as session:
        # First import — all rows inserted
        first_count = await bulk_import_tasks(
            session,
            [{"title": "Sync Task", "status": "pending", "external_id": "ext-001"}],
            organization_id=1,
            on_conflict="skip",
        )
        assert first_count == 1

        # Second import — same external_id, should not raise, should skip
        second_count = await bulk_import_tasks(
            session,
            [{"title": "Sync Task Updated", "status": "done", "external_id": "ext-001"}],
            organization_id=1,
            on_conflict="skip",
        )
        assert second_count == 1  # Function still reports 1 row submitted

    # Verify only one row exists and original title is preserved
    async with AsyncSessionLocal() as verify:
        result = await verify.execute(
            select(Task).where(Task.external_id == "ext-001")
        )
        tasks = result.scalars().all()
        assert len(tasks) == 1
        assert tasks[0].title == "Sync Task"  # Original, not overwritten
```

---

**Extension 3: Return Inserted IDs for Audit Logging**

**Problem:** Downstream audit logging needs the database-assigned `id` of each newly inserted task to create audit log entries.

**Task:** Add a `return_ids: bool = False` parameter. When `True`, return a `list[int]` of newly inserted task IDs. When `False` (the default), return the count as before. The function's return type becomes `int | list[int]`.

**New test to pass:**
```python
@pytest.mark.asyncio
async def test_return_ids_matches_database_records():
    async with AsyncSessionLocal() as session:
        returned_ids = await bulk_import_tasks(
            session,
            [{"title": "Audit Task 1", "status": "pending"},
             {"title": "Audit Task 2", "status": "pending"}],
            organization_id=1,
            return_ids=True,
        )

    assert isinstance(returned_ids, list)
    assert len(returned_ids) == 2
    assert all(isinstance(i, int) for i in returned_ids)

    # Verify the returned IDs actually exist in the database
    async with AsyncSessionLocal() as verify:
        for task_id in returned_ids:
            task = await verify.get(Task, task_id)
            assert task is not None
```

---

# Level 2.5 Solution

## Solution 2.5: The Bulk Import Pipeline

**Extension 1 — Batch Large Imports:**
```python
async def bulk_import_tasks(
    session: AsyncSession,
    tasks_data: list[dict],
    organization_id: int,
    batch_size: int = 2000,
) -> int:
    if not tasks_data:
        return 0

    rows = [
        {**data, "organization_id": organization_id}
        for data in tasks_data
    ]

    total = 0
    for i in range(0, len(rows), batch_size):
        batch = rows[i : i + batch_size]
        await session.execute(insert(Task), batch)
        await session.commit()
        total += len(batch)

    return total
```

**Extension 2 — Duplicate Handling:**
```python
from sqlalchemy.dialects.postgresql import insert as pg_insert

async def bulk_import_tasks(
    session: AsyncSession,
    tasks_data: list[dict],
    organization_id: int,
    batch_size: int = 2000,
    on_conflict: str = "error",
) -> int:
    if not tasks_data:
        return 0

    rows = [
        {**data, "organization_id": organization_id}
        for data in tasks_data
    ]

    total = 0
    for i in range(0, len(rows), batch_size):
        batch = rows[i : i + batch_size]

        if on_conflict == "skip":
            stmt = pg_insert(Task).on_conflict_do_nothing(
                index_elements=["organization_id", "external_id"]
            )
        else:
            stmt = insert(Task)

        await session.execute(stmt, batch)
        await session.commit()
        total += len(batch)

    return total
```

Note: `pg_insert` from `sqlalchemy.dialects.postgresql` is used instead of the generic `insert` because `ON CONFLICT DO NOTHING` is a PostgreSQL-specific clause. This is a case where the ORM abstraction is insufficient and dialect-specific SQL is required.

**Extension 3 — Return Inserted IDs:**
```python
async def bulk_import_tasks(
    session: AsyncSession,
    tasks_data: list[dict],
    organization_id: int,
    batch_size: int = 2000,
    on_conflict: str = "error",
    return_ids: bool = False,
) -> int | list[int]:
    if not tasks_data:
        return [] if return_ids else 0

    rows = [
        {**data, "organization_id": organization_id}
        for data in tasks_data
    ]

    all_ids: list[int] = []
    total = 0
    for i in range(0, len(rows), batch_size):
        batch = rows[i : i + batch_size]

        if on_conflict == "skip":
            stmt = pg_insert(Task).on_conflict_do_nothing(
                index_elements=["organization_id", "external_id"]
            )
        else:
            stmt = insert(Task)

        if return_ids:
            stmt = stmt.returning(Task.id)
            result = await session.execute(stmt, batch)
            all_ids.extend(result.scalars().all())
        else:
            await session.execute(stmt, batch)

        await session.commit()
        total += len(batch)

    return all_ids if return_ids else total
```

**The Key Insight for Extension 3:** `execute(insert(Task), rows)` returns a `CursorResult`, not ORM objects. Without `.returning(Task.id)`, the database performs the insert and discards the generated IDs from PostgreSQL's perspective. This is exactly why Core bulk insert is faster than `add_all()` — it skips ORM identity tracking entirely. Speed and ID visibility are a direct tradeoff. When you need the IDs back, `.returning()` is the explicit opt-in.

---

# LEVEL 3: DESIGN UNDER CONSTRAINTS

## Exercise 3: The Concurrent Task Claimer

**Scenario:** You are building a background job infrastructure for the Task Manager. Multiple worker processes claim pending tasks from a shared queue and process them independently.

**Business Rules:**
- Each task must be claimed by **exactly one** worker — no double-claiming
- Claiming is atomic: find-and-mark must happen in a single database operation
- If no tasks are available, the worker returns `None` immediately without waiting

**Hard Constraints:**
- Must work correctly with **20 fully concurrent workers**
- Each claim operation must complete in **under 100ms**
- Must work correctly with **any number of concurrent workers**, not just 20

**Data model:**
```python
from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy import String, ForeignKey
from datetime import datetime


class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    status: Mapped[str] = mapped_column(String(20), default="pending")
    worker_id: Mapped[str | None] = mapped_column(String(100), default=None)
    claimed_at: Mapped[datetime | None] = mapped_column(default=None)
    organization_id: Mapped[int] = mapped_column(ForeignKey("organizations.id"), index=True)
```

**Provided test suite (all must pass):**

```python
@pytest.mark.asyncio
async def test_single_worker_claims_task():
    task_id = await create_pending_task(title="Test Task", org_id=1)
    claimed = await claim_next_task(worker_id="worker-1", org_id=1)

    assert claimed is not None
    assert claimed.id == task_id
    assert claimed.status == "in_progress"
    assert claimed.worker_id == "worker-1"
    assert claimed.claimed_at is not None


@pytest.mark.asyncio
async def test_no_tasks_returns_none():
    result = await claim_next_task(worker_id="worker-1", org_id=999)
    assert result is None


@pytest.mark.asyncio
async def test_20_workers_no_duplicate_claims():
    """The critical test: 20 workers race for 5 tasks.
    Exactly 5 succeed. No task is claimed twice."""
    task_ids = set()
    for i in range(5):
        tid = await create_pending_task(title=f"Task {i}", org_id=1)
        task_ids.add(tid)

    results = await asyncio.gather(
        *[claim_next_task(worker_id=f"worker-{i}", org_id=1) for i in range(20)],
        return_exceptions=True,
    )

    claimed = [r for r in results if isinstance(r, Task)]
    claimed_ids = [t.id for t in claimed]

    assert len(claimed) == 5, f"Expected 5 claims, got {len(claimed)}"
    assert len(set(claimed_ids)) == 5, f"Duplicate claims: {claimed_ids}"
    assert set(claimed_ids) == task_ids


@pytest.mark.asyncio
async def test_claimed_task_unavailable_to_other_workers():
    task_id = await create_pending_task(title="Exclusive Task", org_id=1)
    first = await claim_next_task(worker_id="worker-A", org_id=1)
    assert first is not None

    second = await claim_next_task(worker_id="worker-B", org_id=1)
    if second is not None:
        assert second.id != task_id
```

---

**Task 3a: Implement `claim_next_task`**

```python
async def claim_next_task(
    worker_id: str,
    org_id: int,
) -> Task | None:
    """
    Atomically claim the next available pending task for this worker.
    Returns the claimed Task, or None if no tasks are available.
    """
    ...
```

Before writing code, answer: which locking strategy prevents double-claiming under 20 concurrent workers — `with_for_update()`, `with_for_update(skip_locked=True)`, or no lock? Explain your choice, then implement.

---

**Task 3b: Explain the alternatives that fail**

Your implementation from 3a should pass `test_20_workers_no_duplicate_claims`. Now explain what would go wrong with each rejected approach:

1. **`with_for_update()` without `skip_locked`**: Describe the specific failure mode under 20 workers competing for 5 tasks. Would `test_20_workers_no_duplicate_claims` pass or fail? Would the system be correct but slow, or actually incorrect?

2. **No locking at all** (plain `select` then `update`): Why would `test_20_workers_no_duplicate_claims` fail? Describe the exact sequence of events for two workers that produces a duplicate claim.

---

**Task 3c: Add stale claim recovery**

Workers can crash mid-task. Add a `reclaim_stale_tasks()` function that re-queues any task stuck in `in_progress` for longer than `stale_after_minutes`.

```python
async def reclaim_stale_tasks(
    org_id: int,
    stale_after_minutes: int = 5,
) -> int:
    """Re-queue tasks stuck in_progress too long. Returns count re-queued."""
    ...
```

**New tests to pass:**
```python
@pytest.mark.asyncio
async def test_stale_task_is_requeued():
    from datetime import timezone
    stale_time = datetime.now(timezone.utc) - timedelta(minutes=10)
    task_id = await create_inprogress_task(
        worker_id="crashed-worker", claimed_at=stale_time, org_id=1
    )

    requeued = await reclaim_stale_tasks(org_id=1, stale_after_minutes=5)
    assert requeued >= 1

    reclaimed = await claim_next_task(worker_id="new-worker", org_id=1)
    assert reclaimed is not None
    assert reclaimed.id == task_id
    assert reclaimed.worker_id == "new-worker"


@pytest.mark.asyncio
async def test_active_task_not_requeued():
    from datetime import timezone
    task_id = await create_inprogress_task(
        worker_id="active-worker", claimed_at=datetime.now(timezone.utc), org_id=1
    )

    await reclaim_stale_tasks(org_id=1, stale_after_minutes=5)

    async with AsyncSessionLocal() as session:
        task = await session.get(Task, task_id)
        assert task.status == "in_progress"
        assert task.worker_id == "active-worker"
```

---

# Level 3 Solution

## Solution 3: The Concurrent Task Claimer

**Task 3a — Implementation:**

The correct choice is `with_for_update(skip_locked=True)`.

- **No lock**: Two workers read the same `pending` row before either commits. Both update it to `in_progress`. Double-claim.
- **`with_for_update()` (no skip)**: Correct, but Workers 2–20 all queue behind Worker 1's lock. They unblock sequentially, but they're all waiting for the same row. Throughput collapses. Under 20 workers competing for 5 tasks, 15 workers end up waiting in line even though there's nothing for them.
- **`with_for_update(skip_locked=True)`**: Correct AND concurrent. A locked row is invisible to other workers. Each worker skips past any row currently being claimed and takes the next available one. 5 workers claim 5 tasks in parallel. 15 workers find nothing available and return `None` immediately.

```python
from sqlalchemy import select, update
from datetime import datetime, timezone


async def claim_next_task(
    worker_id: str,
    org_id: int,
) -> Task | None:
    async with AsyncSessionLocal() as session:
        # Step 1: Find and lock one pending task.
        # SKIP LOCKED makes locked rows invisible — other workers skip past them.
        stmt = (
            select(Task)
            .where(
                Task.status == "pending",
                Task.organization_id == org_id,
            )
            .order_by(Task.id)                   # Deterministic ordering
            .limit(1)
            .with_for_update(skip_locked=True)   # The critical flag
        )
        result = await session.execute(stmt)
        task = result.scalar_one_or_none()

        if task is None:
            return None  # No unclaimed tasks available

        # Step 2: Claim it. Still within the same transaction as the lock.
        task.status = "in_progress"
        task.worker_id = worker_id
        task.claimed_at = datetime.now(timezone.utc)

        # Step 3: Commit releases the lock. Other workers can now see this row
        # as in_progress and skip it naturally via the WHERE status='pending' filter.
        await session.commit()
        await session.refresh(task)
        return task
```

**Task 3b — Why the alternatives fail:**

**`with_for_update()` without `skip_locked`:**
Worker 1 locks row `task_id=1`. Workers 2–20 all call `SELECT ... FOR UPDATE` and reach `task_id=1` in the ordering. All 19 are blocked, waiting for Worker 1's lock to release. Worker 1 commits, Worker 2 unblocks and gets `task_id=2` (the row it locked). Workers 3–20 are now waiting for Worker 2... and so on. The system is correct — no duplicate claims — but throughput is nearly serial. 20 workers process tasks one at a time.

`test_20_workers_no_duplicate_claims` would still pass (no duplicates), but performance would be unacceptably slow.

**No locking:**
Worker 1: `SELECT ... WHERE status='pending' LIMIT 1` → reads `task_id=1`.
Worker 2: `SELECT ... WHERE status='pending' LIMIT 1` → also reads `task_id=1` (Worker 1 hasn't committed yet).
Both workers run `UPDATE tasks SET status='in_progress', worker_id=... WHERE id=1`.
Both commits succeed. Both workers believe they claimed `task_id=1`.

`test_20_workers_no_duplicate_claims` fails: `len(set(claimed_ids)) < 5`.

**Task 3c — Stale claim recovery:**
```python
async def reclaim_stale_tasks(
    org_id: int,
    stale_after_minutes: int = 5,
) -> int:
    from datetime import timedelta, timezone
    cutoff = datetime.now(timezone.utc) - timedelta(minutes=stale_after_minutes)

    async with AsyncSessionLocal() as session:
        stmt = (
            update(Task)
            .where(
                Task.organization_id == org_id,
                Task.status == "in_progress",
                Task.claimed_at < cutoff,     # Claimed before the cutoff time
            )
            .values(
                status="pending",
                worker_id=None,
                claimed_at=None,
            )
        )
        result = await session.execute(stmt)
        await session.commit()
        return result.rowcount
```

Note: `update().where().values()` is the correct tool here. We are not selecting specific objects — we are applying a uniform state change to all rows matching a condition. This generates one SQL statement regardless of how many rows match, which is exactly what Part 4 of the lecture covered.

**The Key Insight:** `SKIP LOCKED` is a work-queue-specific optimization built on the observation that work queue items are **fungible** — a worker doesn't need task #42 specifically, it just needs *any* unclaimed task. This fungibility is what makes `SKIP LOCKED` safe. When items are NOT fungible (e.g., a user needs to update a specific bank account), `SKIP LOCKED` would be wrong — you'd skip the one row you actually needed. Match your locking strategy to your semantics, not just your concurrency requirements.

---

# LEVEL 4: ADVERSARIAL COUNTER-EXAMPLE

## Exercise 4: When Pessimistic Locks Become a Bottleneck

A senior engineer reviews your PR where you've added `with_for_update()` to the task editing flow. Their comment reads:

> "For this specific pattern, pessimistic locking is going to turn your edit endpoint into a single-threaded queue at peak hours. You're solving the right problem with the wrong tool."

**Tasks:**

1. **Describe the failure scenario.** Identify a specific user interaction pattern in the Task Manager where `SELECT ... FOR UPDATE` would create a serious performance problem. Be precise about what the user is doing during the time the lock is held, and what happens to all other users attempting the same operation.

2. **Annotate the broken code.** The snippet below uses pessimistic locking for task editing. Add comments showing the timeline of the lock and why it is held for an unacceptable duration:

```python
@router.get("/tasks/{task_id}/edit")
async def open_task_for_editing(
    task_id: int,
    session: AsyncSession = Depends(get_session),
):
    stmt = select(Task).where(Task.id == task_id).with_for_update()
    result = await session.execute(stmt)
    task = result.scalar_one_or_none()
    return task
    # What is the lock state after this response is sent?

@router.patch("/tasks/{task_id}")
async def submit_task_edit(
    task_id: int,
    body: TaskUpdate,
    session: AsyncSession = Depends(get_session),
):
    task = await session.get(Task, task_id)
    task.title = body.title
    await session.commit()
    return task
```

3. **Write the optimistic version.** Implement the same editing flow using a version column instead. Show the request/response cycle, explain what the user experience looks like when a conflict occurs, and explain why this trade-off is correct for this scenario even though it requires more code.

---

# Level 4 Solution

## Solution 4: When Pessimistic Locks Become a Bottleneck

**1. The failure scenario — Task Description Editing:**

When a user opens a task to edit its title or description, the interaction spans three distinct phases:
1. `GET /tasks/42` — user opens the edit form (~10ms round trip)
2. User **reads, thinks, and types** — 30 to 300 seconds of human time
3. `PATCH /tasks/42` — user saves (~10ms round trip)

If pessimistic locking is applied at phase 1 and the lock is held until phase 3, the lock is held for **30–300 seconds**. Every other user attempting to edit the same task during that window is either blocked (waiting for the lock) or receives an immediate error. In a team environment where a popular task might have 5 users opening it simultaneously, 4 of them are paralyzed for minutes by one person's typing speed.

**2. Annotated broken code:**

```python
@router.get("/tasks/{task_id}/edit")
async def open_task_for_editing(
    task_id: int,
    session: AsyncSession = Depends(get_session),
):
    # 🔒 LOCK ACQUIRED at t=0 when the user opens the edit form.
    stmt = select(Task).where(Task.id == task_id).with_for_update()
    result = await session.execute(stmt)
    task = result.scalar_one_or_none()
    return task
    # ⚠️ HTTP response is sent. But what about the lock?
    # The FastAPI dependency injects a session scoped to the REQUEST.
    # The session commits/closes at the end of THIS request — not when
    # the user finishes editing. The lock is RELEASED here, at t~10ms.
    # This means FOR UPDATE on a GET request accomplishes NOTHING —
    # the lock is released before the user even reads the response.
    # The pattern is architecturally broken: HTTP is stateless.
    # A database lock CANNOT span two separate HTTP requests.

@router.patch("/tasks/{task_id}")
async def submit_task_edit(
    task_id: int,
    body: TaskUpdate,
    session: AsyncSession = Depends(get_session),
):
    # By the time this runs, it's been 30-300 seconds since GET.
    # The FOR UPDATE lock from the GET request was released immediately.
    # This PATCH operates on a completely fresh session with no lock.
    # The attempt to use pessimistic locking here was silent no-op.
    task = await session.get(Task, task_id)
    task.title = body.title
    await session.commit()
    return task
```

The key architectural insight: **pessimistic locks cannot span HTTP request boundaries**. A `FastAPI` session dependency scopes the transaction to a single request. Any lock acquired in request A is released before request B begins. Pessimistic locking for user-facing edit flows is not merely slow — it is structurally impossible in a stateless HTTP API.

**3. The optimistic version:**

```python
# GET: Send the task with its version. No lock acquired.
@router.get("/tasks/{task_id}")
async def get_task(task_id: int, session: AsyncSession = Depends(get_session)):
    task = await session.get(Task, task_id)
    if not task:
        raise HTTPException(404)
    # Client receives: {"id": 42, "title": "...", "version": 7}
    # They hold onto version=7 while they type.
    return task

# PATCH: Client sends back the version they read.
@router.patch("/tasks/{task_id}")
async def update_task(
    task_id: int,
    body: TaskUpdate,  # body contains: title + version
    session: AsyncSession = Depends(get_session),
):
    stmt = (
        update(Task)
        .where(Task.id == task_id, Task.version == body.version)
        .values(title=body.title, version=body.version + 1)
    )
    result = await session.execute(stmt)

    if result.rowcount == 0:
        raise HTTPException(
            status_code=409,
            detail="This task was edited by someone else while you were typing. "
                   "Please reload and reapply your changes."
        )

    await session.commit()
    return {"saved": True}
```

**User experience on conflict:** The user gets a `409 Conflict` and must reload the page to see the current version. They may need to retype their changes. This is mildly inconvenient in the rare case where two people edit the same task simultaneously. But in the overwhelmingly common case where only one person edits at a time, the experience is instant and nobody is blocked.

**The Key Insight:** Pessimistic locking is correct when lock duration is measured in **milliseconds** and contention is high — e.g., decrementing inventory stock, processing a work queue, transferring money. Optimistic locking is correct when "lock duration" would span **user think time** (seconds to minutes) — e.g., form editing, collaborative documents. The mechanism must match the timescale. Using `FOR UPDATE` for a human editing session is solving a millisecond problem with a tool designed for millisecond durations.

---

# LEVEL 5: THE TEACH BACK

## Exercise 5: The Pool Size Debate

A junior developer opens a pull request. The PR description reads:

> **Fix: Increase pool_size to resolve production TimeoutErrors**
>
> We've been getting `TimeoutError: QueuePool limit of size 10 overflow 20 reached` in production during peak hours. I've increased `pool_size` from `10` to `80` and set `max_overflow` to `0`. Our PostgreSQL server has `max_connections = 100`, so this uses the full available capacity directly — overflow is wasteful anyway. Tested locally: no errors seen.

You are the senior engineer reviewing this PR. The junior developer is confident this is correct.

**Task:**

Write your code review comment. Address all four of the following points — skip any one of them and your review is incomplete:

1. **Why the arithmetic is dangerous in a multi-instance deployment.** Walk through the math for 3 running API instances. What is the total potential connection demand? What happens when you push a 4th instance?

2. **What the `TimeoutError` is actually diagnosing.** The error is a symptom. Name the three most likely root causes they have not investigated — none of which are "pool_size is too small."

3. **How this change could cause a complete outage instead of a partial one.** Describe the specific failure mode when total connections across instances exceeds PostgreSQL's `max_connections`.

4. **A concrete, actionable investigation checklist** — three specific things to check before touching a single pool configuration value.

---

# Level 5 Solution

## Solution 5: The Pool Size Debate

**Code Review Comment:**

---

Thanks for debugging this — `TimeoutError` under peak load is a real problem worth fixing. But this specific change has a critical flaw and is very likely treating the symptom rather than the cause. Let me go through each issue.

**1. The arithmetic is dangerous in a multi-instance deployment.**

Our production environment runs 3 API instances (and temporarily more during a rolling deploy). With the proposed change:
- Each instance: `pool_size=80`, `max_overflow=0` → max 80 connections per instance
- 3 instances × 80 = **240 simultaneous connections**
- PostgreSQL `max_connections = 100`

At full load, all three instances attempt to hold 240 connections against a server that only accepts 100. PostgreSQL begins rejecting connections with `FATAL: sorry, too many clients already`. This is worse than our current behavior. We go from "some requests timeout under peak load" (partial outage) to "all requests fail with connection refused" (complete outage). During a deploy when a 4th instance briefly runs, we hit 320 connection attempts — 220% over the limit.

The invariant to follow: `(pool_size + max_overflow) × max_instances` must stay comfortably below `max_connections`, with headroom for monitoring, migrations, and admin connections.

**2. What the `TimeoutError` is actually telling you.**

`QueuePool limit reached` means all pool slots were occupied and none freed within `pool_timeout` seconds. This has three common root causes, none of which are "the pool is too small":

- **Slow queries holding connections hostage.** If your average endpoint query takes 3–5 seconds instead of 50ms, each connection is occupied 60–100× longer. 10 pool slots support ~200 req/s at 50ms/query, but only ~2 req/s at 3s/query. Check `EXPLAIN ANALYZE` on your slowest endpoints before anything else.

- **Connection leak.** Some code path creates a session without `async with` and never returns the connection to the pool. `pool.checkedout()` grows over time and never returns to `pool_size`. Add a `/debug/pool` endpoint exposing `checkedout()` and watch it during load — if it climbs and never drops, you have a leak.

- **Genuine traffic that outpaces the current configuration.** Only valid after the above two are ruled out. And even then, the solution is to add app instances behind a load balancer, not to make each instance greedier with connections.

**3. How this causes a complete outage.**

When total connections across all instances exceeds PostgreSQL's `max_connections`, new connection attempts receive `FATAL: sorry, too many clients already`. This is not a timeout — it is an immediate hard refusal. Every request that needs a database connection fails instantly. The application is completely non-functional. Our current `TimeoutError` affects only the requests that couldn't get a pool slot within 30 seconds. A connection-refused error affects every single request. We trade intermittent slowness for total unavailability.

**4. Investigation checklist before touching pool settings:**

Before changing a single number, check these three things:

1. **Query profiling.** Enable SQLAlchemy's `echo=True` temporarily or add a slow-query log middleware. Find every query slower than 200ms. Run `EXPLAIN ANALYZE` on each one. A missing index that drops a 2s query to 5ms reclaims 99.75% of the connection time it was consuming. This alone often resolves `TimeoutError` issues entirely.

2. **Pool telemetry over time.** Add this endpoint and watch it during load testing:
   ```python
   @router.get("/debug/pool")
   async def pool_status():
       return {
           "checked_out": engine.pool.checkedout(),
           "checked_in": engine.pool.checkedin(),
           "overflow": engine.pool.overflow(),
       }
   ```
   If `checked_out` is growing monotonically and `checked_in` never recovers, you have a leak — not a capacity issue.

3. **Request duration distribution.** Add timing middleware that logs requests taking longer than 500ms. If p95 request time is 3 seconds, pool exhaustion under modest concurrency is mathematically inevitable regardless of pool size. The problem is in the application layer, not the pool configuration.

If all three are clean and a capacity increase is genuinely warranted, the correct change is:
```python
engine = create_async_engine(
    DATABASE_URL,
    pool_size=10,        # Per instance: keep low to leave PostgreSQL headroom
    max_overflow=10,     # Total: 20 per instance × 3 instances = 60 max (within 100)
    pool_timeout=30,
    pool_recycle=3600,
    pool_pre_ping=True,
)
```
And if 60 total connections is genuinely insufficient, the right scaling lever is more app instances behind a load balancer — not a larger pool per instance.

---

**The Key Insight:** `pool_size` is not a volume dial for handling more load. It is a carefully budgeted allocation based on `max_connections ÷ num_instances`. The `TimeoutError` is PostgreSQL's way of telling you that something in your application is holding connections longer than it should — either slow queries, a leak, or true traffic growth requiring more instances. Increasing `pool_size` without investigating the root cause is equivalent to fixing a leaking pipe by buying a larger bucket.