# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BREAK IT FIRST, FIX IT SECOND                                  │
│  ────────────────────────────                                   │
│  Students will watch three things explode:                      │
│  connections exhausted, money vanished, imports crawling.        │
│  THEN we teach the solutions.                                   │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Databases are abstract at scale. We use a BANK analogy.        │
│  Teller windows, joint accounts, payroll batches.               │
│  Every concept maps to something students can picture.          │
│                                                                 │
│  ALWAYS ASYNC, ALWAYS SQLALCHEMY 2.0                            │
│  ───────────────────────────────────                            │
│  Students have used AsyncSession since Week 6.                  │
│  Every example uses the async patterns they already know.       │
│  No legacy 1.x syntax. Legacy is acknowledged, never taught.    │
│                                                                 │
│  CONNECT TO THE PROJECT                                         │
│  ─────────────────────                                          │
│  The Task Manager they've been building is the substrate.       │
│  Examples use tasks, users, organizations — their domain.       │
│  This isn't abstract theory. It's next week's commit.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                   ADVANCED DATABASE PATTERNS                    │
│                      (3.5-4 Hour Lecture)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEMS (30 min)                                  │
│  ├─ 1.1 Explosion #1: Connection Exhaustion                     │
│  ├─ 1.2 Explosion #2: The Vanishing Update                      │
│  ├─ 1.3 Explosion #3: The 10,000-Row Crawl                      │
│  └─ 1.4 The Bank Analogy                                        │
│                                                                 │
│  PART 2: CONNECTION POOLING (40 min)                            │
│  ├─ 2.1 What Is a Connection Pool?                               │
│  ├─ 2.2 Pool Parameters (The Dials You Turn)                     │
│  ├─ 2.3 Configuring the Async Engine (Week 6 Upgrade)            │
│  ├─ 2.4 pool_pre_ping (The Health Check)                         │
│  └─ 2.5 Monitoring and Diagnosing Pool Problems                  │
│                                                                 │
│  PART 3: LOCKING (50 min)                                       │
│  ├─ 3.1 Why Rows Need Locks                                     │
│  ├─ 3.2 Pessimistic Locking (Lock the Door)                     │
│  ├─ 3.3 Optimistic Locking (Check Before You Save)              │
│  ├─ 3.4 When to Use Which                                       │
│  └─ 3.5 Deadlocks (The Deadly Embrace)                           │
│                                                                 │
│  PART 4: BULK OPERATIONS (45 min)                               │
│  ├─ 4.1 The N+1 Write Problem                                   │
│  ├─ 4.2 session.add_all() — ORM-Managed Bulk Inserts            │
│  ├─ 4.3 Core Bulk INSERT (execute + insert)                     │
│  ├─ 4.4 Core Bulk UPDATE (execute + update)                     │
│  ├─ 4.5 Choosing the Right Tool                                 │
│  └─ 4.6 Legacy Awareness (What You'll See in Old Code)           │
│                                                                 │
│  PART 5: RAW SQL WHEN ORM ISN'T ENOUGH (30 min)                │
│  ├─ 5.1 text() — The Escape Hatch                               │
│  ├─ 5.2 Hybrid Properties (Python + SQL in One)                 │
│  └─ 5.3 When to Break Out of the ORM                            │
│                                                                 │
│  PART 6: MULTI-TENANCY & SEEDING (25 min)                      │
│  ├─ 6.1 Multi-Tenancy: One Database, Many Organizations         │
│  ├─ 6.2 Row-Based Isolation (The Practical Approach)            │
│  ├─ 6.3 Schema-Based Isolation (The Nuclear Approach)           │
│  └─ 6.4 Database Seeding (Populating Development Data)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEMS

## 1.1 Explosion #1: Connection Exhaustion

**Start with a demonstration. Blow it up.**

Suppose your Task Manager API becomes popular. Fifty users hit it at once. Each request needs a database connection. Your engine has a default pool of 5 connections. Watch what happens.

```python
# demo_pool_explosion.py
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from sqlalchemy import text

# Small pool — simulates a misconfigured production server
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost:5432/taskmanager",
    pool_size=3,        # Only 3 connections available
    max_overflow=0,     # No extras allowed
    pool_timeout=5,     # Wait 5 seconds, then explode
)

AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession)

async def simulate_request(request_id: int):
    """Each 'request' holds a connection for 3 seconds"""
    async with AsyncSessionLocal() as session:
        print(f"Request {request_id}: got connection")
        await session.execute(text("SELECT pg_sleep(3)"))  # Simulate slow query
        print(f"Request {request_id}: done")

async def main():
    # 10 concurrent requests, 3 connections. Do the math.
    tasks = [simulate_request(i) for i in range(10)]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Request {i}: FAILED — {type(result).__name__}: {result}")

asyncio.run(main())
```

**Run it. Watch it choke.**

```
Request 0: got connection
Request 1: got connection
Request 2: got connection
Request 0: done
Request 3: got connection
Request 1: done
Request 4: got connection
Request 2: done
Request 5: got connection
...
Request 7: FAILED — TimeoutError: QueuePool limit of size 3 ...
Request 8: FAILED — TimeoutError: QueuePool limit of size 3 ...
Request 9: FAILED — TimeoutError: QueuePool limit of size 3 ...
```

**Ask the class:**

> "3 connections. 10 requests. Each holds a connection for 3 seconds. What did you expect? The pool ran dry. Requests 7, 8, 9 waited 5 seconds for a free connection, didn't get one, and crashed. In production, that's 500 Internal Server Error for real users."

---

## 1.2 Explosion #2: The Vanishing Update

**Two users open the same task in your Task Manager. Both edit it. One person's changes vanish.**

```python
# demo_lost_update.py
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from sqlalchemy import select, text

# ... (engine setup same as your Week 6 code) ...

async def user_edits_task(session: AsyncSession, user_name: str):
    """Simulate a user reading a task, thinking, then saving"""
    # Step 1: Read the task
    result = await session.execute(
        text("SELECT id, title, description FROM tasks WHERE id = 1")
    )
    task = result.mappings().one()
    print(f"{user_name} reads task: title='{task['title']}'")

    # Step 2: User thinks for a bit (network delay, typing, etc.)
    await asyncio.sleep(1)

    # Step 3: Save their edit — OVERWRITES whatever is in the DB now
    if user_name == "Alice":
        new_title = "Alice's Updated Title"
    else:
        new_title = "Bob's Updated Title"

    await session.execute(
        text("UPDATE tasks SET title = :title WHERE id = 1"),
        {"title": new_title},
    )
    await session.commit()
    print(f"{user_name} saved: title='{new_title}'")

async def main():
    # Both users start at the SAME TIME
    async with AsyncSessionLocal() as session_a, AsyncSessionLocal() as session_b:
        await asyncio.gather(
            user_edits_task(session_a, "Alice"),
            user_edits_task(session_b, "Bob"),
        )

    # Check final state
    async with AsyncSessionLocal() as session:
        result = await session.execute(
            text("SELECT title FROM tasks WHERE id = 1")
        )
        print(f"\nFinal title in DB: '{result.scalar_one()}'")

asyncio.run(main())
```

**Output:**

```
Alice reads task: title='Original Title'
Bob reads task: title='Original Title'
Alice saved: title='Alice's Updated Title'
Bob saved: title='Bob's Updated Title'

Final title in DB: 'Bob's Updated Title'
```

**Ask the class:**

> "Alice's edit is gone. Completely overwritten. She has no idea. No error, no warning. The database did exactly what we told it to — write this value. But we didn't tell it to CHECK whether someone else had already changed it. This is called a **lost update**, and it is one of the most common and insidious bugs in web applications."

---

## 1.3 Explosion #3: The 10,000-Row Crawl

**Your Task Manager needs to import tasks from a CSV. 10,000 rows. One at a time.**

```python
# demo_slow_inserts.py
import asyncio
import time
from sqlalchemy import insert
from models import Task  # Your SQLAlchemy model from Week 6

async def import_tasks_slow(session: AsyncSession):
    """Insert tasks one at a time. Feel the pain."""
    start = time.time()

    for i in range(10_000):
        task = Task(
            title=f"Imported Task {i}",
            description=f"Description for task {i}",
            category_id=1,
        )
        session.add(task)
        await session.flush()  # Sends INSERT to DB, one at a time

    await session.commit()
    elapsed = time.time() - start
    print(f"Inserted 10,000 tasks in {elapsed:.1f}s")

# Run it...
```

**Output:**

```
Inserted 10,000 tasks in 27.3s
```

**27 seconds for 10,000 rows.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHY SO SLOW?                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Each session.flush() does this:                                │
│                                                                 │
│  Your App               Network             PostgreSQL          │
│  ────────               ───────             ──────────          │
│     │                                                           │
│     │─── INSERT INTO tasks ──────────────────▶│                 │
│     │                                         │ (parse, plan,   │
│     │                                         │  execute, ack)  │
│     │◀──────────── OK ───────────────────────│                 │
│     │                                                           │
│     │─── INSERT INTO tasks ──────────────────▶│                 │
│     │◀──────────── OK ───────────────────────│                 │
│     │                                                           │
│     ... (repeat 9,998 more times)                               │
│                                                                 │
│  10,000 network round trips.                                    │
│  10,000 individual parse-plan-execute cycles.                   │
│  10,000 transaction log writes (if committing each).            │
│                                                                 │
│  The database spends more time on OVERHEAD than actual work.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Later in this lecture, we'll do the same import in under 1 second. Same data. Same database. Different technique."

---

## 1.4 The Bank Analogy

**This analogy will carry through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE BANK ANALOGY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your DATABASE is a BANK.                                       │
│  Your APPLICATION is a line of CUSTOMERS.                       │
│                                                                 │
│  CONNECTION POOL = TELLER WINDOWS                               │
│  ─────────────────────────────────                              │
│  The bank doesn't hire one teller per customer.                 │
│  It has a fixed number of windows. Customers queue.             │
│  Rush hour? Open a few temporary extra windows.                 │
│  Teller gets tired? Replace them with a fresh one.              │
│                                                                 │
│  LOCKING = JOINT BANK ACCOUNT                                   │
│  ─────────────────────────────                                  │
│  Two people share an account. Both check the balance.           │
│  Both withdraw. One withdrawal vanishes.                        │
│  Solution: either LOCK the account while one person is at the   │
│  window, or CHECK the balance hasn't changed before saving.     │
│                                                                 │
│  BULK OPERATIONS = PAYROLL PROCESSING                           │
│  ─────────────────────────────────────                          │
│  A company doesn't send one employee's paycheck at a time.      │
│  They hand the bank a BATCH: "Here are 500 deposits."           │
│  One trip. One transaction. Done.                               │
│                                                                 │
│  RAW SQL = TALKING DIRECTLY TO THE VAULT MANAGER                │
│  ────────────────────────────────────────────────               │
│  Usually you interact through standard procedures (the ORM).    │
│  But sometimes you need a custom instruction only the vault     │
│  manager understands. You write it by hand.                     │
│                                                                 │
│  MULTI-TENANCY = MULTIPLE COMPANIES IN ONE BANK                 │
│  ───────────────────────────────────────────────                │
│  The bank serves many companies. Company A must NEVER see       │
│  Company B's accounts. Same building, separate ledgers.         │
│                                                                 │
│  SEEDING = SETTING UP TRAINING ACCOUNTS                         │
│  ────────────────────────────────────────                       │
│  Before training new employees, you fill the system             │
│  with fake but realistic accounts to practice with.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to code:**

```
Bank                        │  SQLAlchemy / PostgreSQL
────────────────────────────│──────────────────────────────────
Bank vault                  │  PostgreSQL server
Teller windows              │  Database connections
Number of open windows      │  pool_size
Temporary extra windows     │  max_overflow
Customer gives up waiting   │  pool_timeout
Replace a tired teller      │  pool_recycle
Joint account withdrawal    │  Concurrent UPDATE on same row
Lock the account            │  SELECT ... FOR UPDATE
Check balance before save   │  Version column (optimistic lock)
Payroll batch               │  Bulk INSERT / UPDATE
Custom vault instruction    │  text() raw SQL
Multiple companies          │  tenant_id column / separate schemas
Training accounts           │  Database seed scripts
```

---

# PART 2: CONNECTION POOLING

## 2.1 What Is a Connection Pool?

**A database connection is expensive to create.**

```
┌─────────────────────────────────────────────────────────────────┐
│              CREATING A CONNECTION (The Expensive Way)           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your App                              PostgreSQL               │
│  ────────                              ──────────               │
│     │                                                           │
│     │── TCP handshake (SYN/ACK) ──────────▶│                   │
│     │◀──────────────────────────────────── │                   │
│     │── SSL negotiation (if enabled) ─────▶│                   │
│     │◀──────────────────────────────────── │                   │
│     │── Authentication (user/pass) ───────▶│                   │
│     │◀── Auth OK ──────────────────────── │                   │
│     │── Set session parameters ───────────▶│                   │
│     │◀── Ready ────────────────────────── │                   │
│     │                                                           │
│     │  NOW you can run a query. (~5-20ms just for setup)        │
│     │                                                           │
│     │── SELECT * FROM tasks ──────────────▶│                   │
│     │◀── Results ──────────────────────── │                   │
│     │                                                           │
│     │── Close connection (tear down) ─────▶│                   │
│     │                                                           │
│  And then the NEXT request does ALL of this again.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A connection pool solves this by keeping connections alive and reusing them:**

```
┌─────────────────────────────────────────────────────────────────┐
│               CONNECTION POOL (The Smart Way)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                  ┌──────────────────┐                           │
│                  │  CONNECTION POOL  │                           │
│                  │  (QueuePool)      │                           │
│                  │                  │                           │
│                  │  ┌──┐ ┌──┐ ┌──┐ │                           │
│   Request A ────▶│  │C1│ │C2│ │C3│ │◀──── PostgreSQL           │
│   Request B ────▶│  └──┘ └──┘ └──┘ │      (persistent          │
│   Request C ────▶│  │C4│ │C5│      │       connections)        │
│                  │  └──┘ └──┘      │                           │
│                  └──────────────────┘                           │
│                                                                 │
│  1. Request arrives → Borrow a connection from the pool         │
│  2. Run queries                                                 │
│  3. Request done → Return connection to the pool                │
│  4. Next request → Reuse that SAME connection (no setup cost)   │
│                                                                 │
│  THE TELLER WINDOW ANALOGY:                                     │
│  The bank doesn't hire/fire a teller for each customer.         │
│  Tellers stay at their windows. Customers come and go.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key insight:**

> "You've been using a connection pool since Week 6. Every time you wrote `async with session`, you were borrowing from the pool and returning it when the `async with` block ended. `create_async_engine()` creates the pool automatically. You just didn't know the dials you could turn."

---

## 2.2 Pool Parameters (The Dials You Turn)

**Five parameters control the pool's behavior:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE FIVE DIALS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                               │
│  │  pool_size   │  How many permanent teller windows            │
│  │  (default 5) │  are open at all times.                       │
│  └──────────────┘                                               │
│         │                                                       │
│         │  "We keep 5 tellers on staff, always."                │
│         │                                                       │
│  ┌──────────────┐                                               │
│  │ max_overflow  │  How many EXTRA temporary windows            │
│  │ (default 10)  │  can open during rush hour.                  │
│  └──────────────┘                                               │
│         │                                                       │
│         │  "If all 5 are busy, we can open up to 10 more        │
│         │   temporary windows. They close when rush ends."      │
│         │                                                       │
│         │  Total max connections = pool_size + max_overflow      │
│         │  Default: 5 + 10 = 15 simultaneous connections        │
│         │                                                       │
│  ┌──────────────┐                                               │
│  │ pool_timeout  │  How long (seconds) a customer waits         │
│  │ (default 30)  │  in line before giving up.                   │
│  └──────────────┘                                               │
│         │                                                       │
│         │  "If you wait 30 seconds and no window opens,         │
│         │   you get a TimeoutError. Go home."                   │
│         │                                                       │
│  ┌──────────────┐                                               │
│  │ pool_recycle  │  How often (seconds) to replace a            │
│  │ (default -1)  │  connection with a fresh one.                │
│  └──────────────┘                                               │
│         │                                                       │
│         │  "Every 3600 seconds (1 hour), retire the teller      │
│         │   and bring in a fresh one. Prevents stale            │
│         │   connections. -1 means never recycle."               │
│         │                                                       │
│  ┌───────────────┐                                              │
│  │ pool_pre_ping  │  Ping the connection before lending it.     │
│  │ (default False)│  "Are you still alive?"                     │
│  └───────────────┘                                              │
│         │                                                       │
│         │  "Before assigning a teller to a customer, check:     │
│         │   'Hey, are you awake?' If not, replace them."        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize the pool under load:**

```
┌─────────────────────────────────────────────────────────────────┐
│          POOL LIFECYCLE (pool_size=3, max_overflow=2)           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TIME   POOL STATE              WHAT HAPPENS                   │
│  ────   ──────────              ─────────────                  │
│                                                                 │
│  t=0    [C1:idle] [C2:idle] [C3:idle]                           │
│         3 permanent connections ready.                          │
│                                                                 │
│  t=1    [C1:BUSY] [C2:BUSY] [C3:idle]                           │
│         Request A takes C1, Request B takes C2.                 │
│                                                                 │
│  t=2    [C1:BUSY] [C2:BUSY] [C3:BUSY]                           │
│         Request C takes C3. Pool is full.                       │
│                                                                 │
│  t=3    [C1:BUSY] [C2:BUSY] [C3:BUSY] [C4:BUSY]                │
│         Request D arrives. No permanent slots!                  │
│         Overflow: create temporary C4.                          │
│                                                                 │
│  t=4    [C1:BUSY] [C2:BUSY] [C3:BUSY] [C4:BUSY] [C5:BUSY]     │
│         Request E: overflow #2. MAX REACHED (3+2=5).            │
│                                                                 │
│  t=5    Request F arrives → WAITS in queue...                   │
│         (up to pool_timeout seconds)                            │
│                                                                 │
│  t=6    [C1:idle] [C2:BUSY] [C3:BUSY] [C4:BUSY] [C5:BUSY]     │
│         Request A finishes → C1 returned to pool.               │
│         Request F → gets C1. Crisis averted.                    │
│                                                                 │
│  t=10   [C1:idle] [C2:idle] [C3:idle]                           │
│         Rush hour ends. Overflow connections (C4, C5)           │
│         are CLOSED. Only permanent ones stay.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Configuring the Async Engine (Week 6 Upgrade)

**In Week 6, your engine setup was minimal. Now we tune it:**

```python
# database.py — BEFORE (Week 6, basic setup)
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/taskmanager")
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

```python
# database.py — AFTER (Week 7, production-tuned)
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/taskmanager",

    # ── THE POOL DIALS ──────────────────────────────────────────

    pool_size=10,          # 10 permanent connections
                           # Rule of thumb: start with 2-4x your CPU cores
                           # A 4-core server → 10-20 pool_size

    max_overflow=20,       # 20 additional during spikes
                           # Total max: 10 + 20 = 30 connections
                           # WARNING: PostgreSQL default max_connections = 100
                           # Don't exceed it across all your app instances!

    pool_timeout=30,       # Wait 30s for a connection before raising error
                           # For APIs: 30s is generous. User already left.
                           # Consider 5-10s for user-facing endpoints.

    pool_recycle=3600,     # Replace connections every hour (3600 seconds)
                           # Prevents stale connections from:
                           #   - Cloud DB proxies closing idle connections
                           #   - Network infrastructure with timeout policies
                           #   - PostgreSQL's idle session timeout

    pool_pre_ping=True,    # Check connection health before using it
                           # Adds ~1ms overhead per query.
                           # Prevents: "connection was closed" errors
                           # Worth it. Always enable in production.
)

AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

**What about PostgreSQL's connection limit?**

```
┌─────────────────────────────────────────────────────────────────┐
│               WATCH YOUR TOTAL CONNECTIONS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PostgreSQL default: max_connections = 100                      │
│                                                                 │
│  If you run 3 instances of your API:                            │
│  ┌────────────────┐  pool_size=10, max_overflow=20  = 30 max   │
│  │   Instance 1   │──────────────────────────────────┐         │
│  └────────────────┘                                  │         │
│  ┌────────────────┐  pool_size=10, max_overflow=20  = 30 max   │
│  │   Instance 2   │──────────────────────────────┐  │         │
│  └────────────────┘                              │  │         │
│  ┌────────────────┐  pool_size=10, max_overflow=20  = 30 max   │
│  │   Instance 3   │────────────────────────┐    │  │         │
│  └────────────────┘                        │    │  │         │
│                                            ▼    ▼  ▼         │
│                                    ┌──────────────────┐       │
│                                    │   PostgreSQL      │       │
│                                    │   max: 100 conn   │       │
│                                    │   used: up to 90  │       │
│                                    └──────────────────┘       │
│                                                                 │
│  3 instances × 30 max each = 90. Getting close to 100.         │
│  Add a 4th instance? Boom. Connection refused.                  │
│                                                                 │
│  RULE: Sum of (pool_size + max_overflow) across ALL instances   │
│        must be LESS than PostgreSQL's max_connections.           │
│        Leave headroom for admin connections, migrations, etc.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 pool_pre_ping (The Health Check)

**Why connections go stale:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  STALE CONNECTIONS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your pool has connection C1, idle for 2 hours.                 │
│  Meanwhile, the database restarted (maintenance, crash, etc.)   │
│                                                                 │
│  WITHOUT pool_pre_ping:                                         │
│  ─────────────────────                                          │
│  Request → Borrows C1 → "SELECT * FROM tasks"                   │
│  → ERROR: "connection was closed" ← C1 is dead!                │
│  → User sees 500 error. You get paged at 3am.                  │
│                                                                 │
│  WITH pool_pre_ping=True:                                       │
│  ────────────────────────                                       │
│  Request → Pool pings C1 → "SELECT 1" → No response!           │
│  → Pool discards C1, creates fresh C2                          │
│  → Request uses C2 → "SELECT * FROM tasks" → Works!             │
│  → User never notices.                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Always enable `pool_pre_ping=True` in production. The ~1ms overhead is invisible compared to the hours of debugging stale connection errors at 3 AM."

---

## 2.5 Monitoring and Diagnosing Pool Problems

**When something's wrong, check the pool:**

```python
# Check pool status (useful in health check endpoints)
from sqlalchemy import event

# Your engine from database.py
pool = engine.pool

# Check current pool state
def get_pool_status() -> dict:
    return {
        "pool_size": pool.size(),             # Configured permanent slots
        "checked_in": pool.checkedin(),       # Connections idle in pool
        "checked_out": pool.checkedout(),     # Connections currently in use
        "overflow": pool.overflow(),          # Overflow connections in use
        "total": pool.checkedout() + pool.checkedin(),
    }

# Expose in a health/debug endpoint (Week 3 FastAPI knowledge):
# @router.get("/debug/pool")
# async def pool_health():
#     return get_pool_status()
```

**Symptoms and diagnoses:**

```
┌─────────────────────────────────────────────────────────────────┐
│              POOL PROBLEM DIAGNOSIS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SYMPTOM                        LIKELY CAUSE                   │
│  ───────                        ────────────                   │
│                                                                 │
│  TimeoutError: QueuePool        Pool exhausted. Either:         │
│  limit reached                  • pool_size too small           │
│                                 • Queries too slow (optimize!)  │
│                                 • Connection leak (see below)   │
│                                                                 │
│  checked_out keeps growing,     CONNECTION LEAK. Code is        │
│  never decreases                borrowing connections but       │
│                                 never returning them.           │
│                                 Missing: async with session     │
│                                                                 │
│  Intermittent "connection       Stale connections. Enable       │
│  closed" errors                 pool_pre_ping=True and set      │
│                                 pool_recycle=3600               │
│                                                                 │
│  All connections busy but       Slow queries. Use EXPLAIN       │
│  app seems idle                 (Lecture 1 this week).          │
│                                 Add indexes. Optimize.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The most dangerous bug — connection leaks:**

```python
# ❌ CONNECTION LEAK: session never returned to pool
async def leaky_function():
    session = AsyncSessionLocal()       # Borrows from pool
    result = await session.execute(...)
    return result
    # session.close() never called! Connection stuck in "checked_out" forever.
    # Do this enough times and the pool runs dry.

# ✅ ALWAYS use async with (you've been doing this since Week 6)
async def safe_function():
    async with AsyncSessionLocal() as session:  # Borrows from pool
        result = await session.execute(...)
        return result
    # async with guarantees session.close() runs, even if exceptions occur.
    # Connection returned to pool. Always.
```

> "This is why we drilled `async with` in Week 6. It's not just clean code — it's leak prevention. A single missing `async with` can crash your production server under load."

---

# PART 3: LOCKING

## 3.1 Why Rows Need Locks

**Recall the bank demo from Part 1. Let's formalize the problem.**

The lost update happens because of a **read-modify-write race condition**:

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE LOST UPDATE PROBLEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Time   Alice (Transaction A)       Bob (Transaction B)        │
│  ────   ─────────────────────       ────────────────────       │
│                                                                 │
│  t=0    BEGIN                       BEGIN                       │
│                                                                 │
│  t=1    SELECT balance              SELECT balance              │
│         FROM accounts               FROM accounts               │
│         WHERE id = 1;               WHERE id = 1;               │
│         → balance = 1000            → balance = 1000            │
│                                                                 │
│  t=2    (calculate: 1000 - 300)     (calculate: 1000 - 500)    │
│                                                                 │
│  t=3    UPDATE accounts             UPDATE accounts             │
│         SET balance = 700           SET balance = 500           │
│         WHERE id = 1;               WHERE id = 1;               │
│                                                                 │
│  t=4    COMMIT ✅                                               │
│                                                                 │
│  t=5                                COMMIT ✅                   │
│                                                                 │
│  FINAL BALANCE: 500                                             │
│  EXPECTED BALANCE: 1000 - 300 - 500 = 200                      │
│                                                                 │
│  Alice's $300 withdrawal VANISHED.                              │
│  The bank just lost $300.                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**This is not a database bug. The database did exactly what we asked. The bug is in our logic: we read, computed in Python, then wrote — without protecting the gap between read and write.**

Two solutions exist. They have different philosophies.

---

## 3.2 Pessimistic Locking (Lock the Door)

**Philosophy: "Assume conflict WILL happen. Lock the row BEFORE reading."**

```
┌─────────────────────────────────────────────────────────────────┐
│                  PESSIMISTIC LOCKING                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BANK ANALOGY:                                                  │
│  When Alice walks up to the teller, the teller puts a           │
│  "WINDOW CLOSED — IN USE" sign on her account.                  │
│  Bob arrives, tries to access the same account →                │
│  "Please wait. Account is locked."                              │
│  Alice finishes → sign removed → Bob can proceed.               │
│                                                                 │
│  SQL: SELECT ... FOR UPDATE                                     │
│                                                                 │
│  Time   Alice                            Bob                    │
│  ────   ─────                            ───                    │
│  t=0    BEGIN                            BEGIN                  │
│                                                                 │
│  t=1    SELECT balance                                          │
│         FROM accounts                                           │
│         WHERE id = 1                                            │
│         FOR UPDATE;  ← LOCKS ROW                                │
│         → balance = 1000                                        │
│                                                                 │
│  t=2                                     SELECT balance         │
│                                          FROM accounts          │
│                                          WHERE id = 1           │
│                                          FOR UPDATE;            │
│                                          → ⏳ BLOCKED (waiting) │
│                                                                 │
│  t=3    UPDATE SET balance = 700;                               │
│         COMMIT; ← RELEASES LOCK                                │
│                                                                 │
│  t=4                                     → balance = 700 ✅      │
│                                          (NOW Bob sees the      │
│                                           updated balance)      │
│                                                                 │
│  t=5                                     UPDATE SET bal = 200;  │
│                                          COMMIT;                │
│                                                                 │
│  FINAL: 200 ✅ CORRECT                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In SQLAlchemy (using the Task Manager domain):**

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from models import Task  # Your model from Week 6


async def assign_task(
    session: AsyncSession,
    task_id: int,
    assignee_id: int,
) -> Task:
    """
    Assign a task to a user. Only one person can assign at a time.
    Uses pessimistic locking to prevent race conditions.
    """
    # FOR UPDATE: lock this row until we commit/rollback.
    # No other transaction can read (with FOR UPDATE) or write this row.
    stmt = (
        select(Task)
        .where(Task.id == task_id)
        .with_for_update()  # ← This is SELECT ... FOR UPDATE
    )
    result = await session.execute(stmt)
    task = result.scalar_one_or_none()

    if task is None:
        raise TaskNotFoundError(f"Task {task_id} not found")

    if task.assignee_id is not None:
        raise TaskAlreadyAssignedError(
            f"Task {task_id} is already assigned to user {task.assignee_id}"
        )

    # Safe to modify — we hold the lock
    task.assignee_id = assignee_id
    await session.commit()  # Lock released here

    return task
```

**Variations of FOR UPDATE:**

```python
# Standard: block until the lock is available
select(Task).with_for_update()

# NOWAIT: don't wait — raise error immediately if locked
select(Task).with_for_update(nowait=True)
# Use when: you'd rather fail fast than wait

# SKIP LOCKED: skip rows that are locked, return the rest
select(Task).with_for_update(skip_locked=True)
# Use when: processing a work queue — grab any unlocked task
# "Give me a task to work on, but skip ones someone else grabbed."
```

**The work queue pattern (SKIP LOCKED):**

```python
async def claim_next_task(session: AsyncSession, worker_id: int) -> Task | None:
    """Claim the next available unassigned task for processing.
    Multiple workers can call this concurrently without conflict."""
    stmt = (
        select(Task)
        .where(Task.status == "pending")
        .order_by(Task.created_at)
        .limit(1)
        .with_for_update(skip_locked=True)  # Skip tasks another worker just grabbed
    )
    result = await session.execute(stmt)
    task = result.scalar_one_or_none()

    if task is None:
        return None  # No pending tasks available

    task.status = "in_progress"
    task.worker_id = worker_id
    await session.commit()
    return task
```

---

## 3.3 Optimistic Locking (Check Before You Save)

**Philosophy: "Assume conflict is RARE. Don't lock anything. But CHECK when saving."**

```
┌─────────────────────────────────────────────────────────────────┐
│                  OPTIMISTIC LOCKING                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BANK ANALOGY:                                                  │
│  Alice and Bob both get deposit slips with "Current Balance:    │
│  $1000" printed on them. Both fill in their withdrawals.        │
│  Alice submits first. The teller checks: "Balance is still      │
│  $1000? Yes." → Processes it. Balance now $700.                 │
│  Bob submits next. The teller checks: "Balance is still         │
│  $1000? No, it's $700." → REJECTED. "Get a new slip."          │
│  Bob gets a fresh slip ($700), recalculates, resubmits.         │
│                                                                 │
│  The "deposit slip balance" is the VERSION NUMBER.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Step 1: Add a version column to your model.**

```python
# models.py — add version column
from sqlalchemy.orm import Mapped, mapped_column, DeclarativeBase
from sqlalchemy import String, Text


class Base(DeclarativeBase):
    pass


class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[str] = mapped_column(Text, default="")
    assignee_id: Mapped[int | None] = mapped_column(default=None)

    # The version column — starts at 1, increments on every update
    version: Mapped[int] = mapped_column(default=1)
```

Remember to create an Alembic migration for this new column (Week 6 Lecture 3).

**Step 2: Check the version when updating.**

```python
from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession
from models import Task


class OptimisticLockError(Exception):
    """Raised when the row was modified by another transaction."""
    pass


async def update_task(
    session: AsyncSession,
    task_id: int,
    new_title: str,
    expected_version: int,  # The version the client read earlier
) -> Task:
    """
    Update a task's title, but only if no one else changed it
    since we last read it. Uses optimistic locking.
    """
    stmt = (
        update(Task)
        .where(
            Task.id == task_id,
            Task.version == expected_version,  # ← THE KEY CHECK
        )
        .values(
            title=new_title,
            version=expected_version + 1,  # Bump version on success
        )
        .returning(Task)  # PostgreSQL RETURNING — get the updated row back
    )
    result = await session.execute(stmt)
    updated_task = result.scalar_one_or_none()

    if updated_task is None:
        # Either the task doesn't exist, or the version didn't match.
        # Check which one:
        check = await session.execute(select(Task).where(Task.id == task_id))
        existing = check.scalar_one_or_none()

        if existing is None:
            raise TaskNotFoundError(f"Task {task_id} not found")

        raise OptimisticLockError(
            f"Task {task_id} was modified by another user. "
            f"Your version: {expected_version}, current version: {existing.version}. "
            f"Please re-read and try again."
        )

    await session.commit()
    return updated_task
```

**How the client uses this:**

```python
# In a FastAPI route (Week 3-4 knowledge):
from pydantic import BaseModel

class TaskUpdate(BaseModel):
    title: str
    version: int  # Client MUST send the version they read

@router.patch("/tasks/{task_id}")
async def update_task_endpoint(
    task_id: int,
    body: TaskUpdate,
    session: AsyncSession = Depends(get_session),
):
    try:
        task = await update_task(session, task_id, body.title, body.version)
        return task
    except OptimisticLockError as e:
        raise HTTPException(status_code=409, detail=str(e))  # 409 Conflict
```

**The flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│              OPTIMISTIC LOCKING FLOW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Client reads task:                                          │
│     GET /tasks/1                                                │
│     → { "id": 1, "title": "Old", "version": 3 }               │
│                                                                 │
│  2. Client edits and sends update WITH version:                 │
│     PATCH /tasks/1                                              │
│     { "title": "New Title", "version": 3 }                     │
│                                                                 │
│  3. Server does:                                                │
│     UPDATE tasks SET title='New Title', version=4               │
│     WHERE id=1 AND version=3                                    │
│              ▲                                                  │
│              │                                                  │
│              └── "Only update if no one else changed it."       │
│                                                                 │
│  4a. Success (rowcount=1): version was still 3 → updated ✅     │
│  4b. Failure (rowcount=0): someone else already bumped it → 409 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 When to Use Which

```
┌─────────────────────────────────────────────────────────────────┐
│           PESSIMISTIC VS OPTIMISTIC — THE DECISION              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      PESSIMISTIC                                │
│                      (FOR UPDATE)                               │
│                      ────────────                               │
│  Conflicts:          Frequent / guaranteed                      │
│  Lock duration:      Short (within single DB transaction)       │
│  User experience:    Other users wait (or get NOWAIT error)     │
│  Complexity:         Low (database handles it)                  │
│  Risk:               Deadlocks (if locking multiple rows)       │
│  Performance:        Worse under low contention (unnecessary    │
│                      locks). Better under HIGH contention       │
│                      (prevents wasted work from retries).       │
│                                                                 │
│  USE WHEN:                                                      │
│  • Financial transactions (withdrawals, transfers)              │
│  • Work queue processing (claim a job)                          │
│  • Any operation where conflicts are common                     │
│  • Inventory decrement (selling last item in stock)             │
│                                                                 │
│                                                                 │
│                      OPTIMISTIC                                 │
│                      (Version Column)                           │
│                      ────────────────                           │
│  Conflicts:          Rare                                       │
│  Lock duration:      None (no row lock held)                    │
│  User experience:    "Conflict detected, please retry"          │
│  Complexity:         Higher (version tracking, retry logic)     │
│  Risk:               Stale reads if retry logic is wrong        │
│  Performance:        Better under low contention (no locks).    │
│                      Worse under HIGH contention (many retries).│
│                                                                 │
│  USE WHEN:                                                      │
│  • Collaborative editing (Google Docs-style)                    │
│  • CMS updates (multiple editors, rare same-page edits)        │
│  • Web forms with long think-time between read and write        │
│  • Any case where holding a DB lock for seconds is too long     │
│                                                                 │
│                                                                 │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  YOUR TASK MANAGER:                                   │      │
│  │  • Assigning tasks → Pessimistic (multiple users,     │      │
│  │    one task, fast operation)                           │      │
│  │  • Editing task title → Optimistic (users read, think │      │
│  │    for minutes, then save — can't hold lock that long)│      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Deadlocks (The Deadly Embrace)

**Pessimistic locking has a trap. It's called a deadlock.**

```
┌─────────────────────────────────────────────────────────────────┐
│                       DEADLOCK                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BANK ANALOGY:                                                  │
│  Alice locks Account #1, then tries to lock Account #2.         │
│  Bob locks Account #2, then tries to lock Account #1.           │
│  Both are holding one lock and waiting for the other.           │
│  Neither can proceed. Frozen forever.                           │
│                                                                 │
│                                                                 │
│  Transaction A:                Transaction B:                   │
│  ─────────────                 ──────────────                   │
│  1. LOCK task #10  ✅           1. LOCK task #20  ✅             │
│  2. LOCK task #20  ⏳ waiting   2. LOCK task #10  ⏳ waiting     │
│     (B has it!)                    (A has it!)                   │
│                                                                 │
│         ┌──────────┐         ┌──────────┐                       │
│         │    A     │◀─waits──│    B     │                       │
│         │ holds #10│         │ holds #20│                       │
│         │ wants #20│──waits─▶│ wants #10│                       │
│         └──────────┘         └──────────┘                       │
│                                                                 │
│              DEADLOCK. Neither can proceed.                      │
│                                                                 │
│  PostgreSQL detects this (after ~1 second) and KILLS one         │
│  transaction with: ERROR: deadlock detected                     │
│  The killed transaction gets rolled back.                       │
│  The surviving transaction proceeds.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Prevention: always lock rows in a consistent order.**

```python
async def transfer_between_tasks(
    session: AsyncSession,
    task_id_a: int,
    task_id_b: int,
):
    """Move resources between two tasks. Lock in consistent order."""
    # ✅ ALWAYS lock by ascending ID — every transaction does this
    first_id, second_id = sorted([task_id_a, task_id_b])

    stmt = (
        select(Task)
        .where(Task.id.in_([first_id, second_id]))
        .order_by(Task.id)          # Sorted order
        .with_for_update()          # Lock both, in the same order always
    )
    result = await session.execute(stmt)
    tasks = {t.id: t for t in result.scalars().all()}

    # Now safely modify both tasks...
    # No deadlock possible because every caller locks in ID order.
    await session.commit()
```

> "Simple rule: if you need to lock multiple rows, always lock them in the same deterministic order (typically by primary key). This makes deadlocks impossible because circular waits can never form."

---

# PART 4: BULK OPERATIONS

## 4.1 The N+1 Write Problem

**You know the N+1 read problem from Week 6 (eager vs lazy loading). There's a write equivalent, and it's just as brutal.**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE N+1 WRITE PROBLEM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INSERTING 1,000 TASKS ONE AT A TIME:                           │
│                                                                 │
│  for task_data in thousand_tasks:     # 1,000 iterations        │
│      session.add(Task(**task_data))   # Stages one object       │
│      await session.flush()            # Sends 1 INSERT to DB   │
│                                                                 │
│  App ─── INSERT ──▶ DB ──▶ OK                                   │
│  App ─── INSERT ──▶ DB ──▶ OK                                   │
│  App ─── INSERT ──▶ DB ──▶ OK                                   │
│  ... (997 more round trips) ...                                 │
│                                                                 │
│  = 1,000 network round trips                                    │
│  = 1,000 query parse/plan cycles                                │
│  = ~25-30 seconds                                               │
│                                                                 │
│                                                                 │
│  INSERTING 1,000 TASKS IN ONE BATCH:                            │
│                                                                 │
│  await session.execute(                                         │
│      insert(Task), thousand_tasks     # 1 statement, all data   │
│  )                                                              │
│                                                                 │
│  App ─── INSERT 1000 rows ──▶ DB ──▶ OK                         │
│                                                                 │
│  = 1 network round trip                                         │
│  = 1 query parse/plan cycle                                     │
│  = ~0.3-0.8 seconds                                             │
│                                                                 │
│  PAYROLL ANALOGY:                                               │
│  Don't drive to the bank 1,000 times.                           │
│  Hand them one list of 1,000 deposits.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 session.add_all() — ORM-Managed Bulk Inserts

**The simplest approach. You create ORM objects, add them all at once.**

```python
from models import Task

async def import_tasks_orm(
    session: AsyncSession,
    tasks_data: list[dict],
) -> list[Task]:
    """Bulk insert using ORM objects. Simple, but not the fastest."""

    # Create ORM objects
    tasks = [Task(**data) for data in tasks_data]

    # Add all at once
    session.add_all(tasks)
    await session.commit()

    # All objects now have their IDs populated
    return tasks
```

**What `add_all` actually does:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   session.add_all()                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Adds all objects to the session's identity map              │
│  2. On commit, SQLAlchemy batches them into INSERT statements   │
│  3. Objects get their generated IDs back from the DB            │
│                                                                 │
│  ADVANTAGES:                                                    │
│  ├─ ORM events fire (if you have any)                           │
│  ├─ Objects are tracked by the session (can modify them after)  │
│  ├─ Default values from Python (like datetime.now()) work       │
│  └─ Relationships are resolved automatically                    │
│                                                                 │
│  DISADVANTAGES:                                                 │
│  ├─ Creates Python objects for every row (memory for huge sets) │
│  ├─ Session tracks each object (overhead)                       │
│  └─ Still does individual INSERT statements under the hood      │
│     (though batched into fewer round trips by asyncpg)          │
│                                                                 │
│  BEST FOR: Hundreds of rows where you need ORM features.        │
│  NOT FOR: 100,000 rows. Use Core bulk operations instead.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Complete example with error handling:**

```python
async def import_tasks_from_csv(
    session: AsyncSession,
    csv_rows: list[dict],
) -> int:
    """Import tasks from parsed CSV data with validation."""
    tasks: list[Task] = []

    for i, row in enumerate(csv_rows):
        try:
            task = Task(
                title=row["title"],
                description=row.get("description", ""),
                category_id=int(row["category_id"]),
            )
            tasks.append(task)
        except (KeyError, ValueError) as e:
            # Log bad rows, skip them, don't crash the whole import
            print(f"Skipping row {i}: {e}")
            continue

    if tasks:
        session.add_all(tasks)
        await session.commit()

    return len(tasks)
```

---

## 4.3 Core Bulk INSERT (execute + insert)

**The fast path. Skip ORM object creation. Talk to the database directly.**

```python
from sqlalchemy import insert
from models import Task

async def import_tasks_fast(
    session: AsyncSession,
    tasks_data: list[dict],
) -> int:
    """
    Bulk insert using Core. Fastest method.
    Bypasses ORM object creation — sends data directly to PostgreSQL.
    """
    if not tasks_data:
        return 0

    # Each dict maps column names to values
    # [{"title": "Task 1", "description": "...", "category_id": 1}, ...]
    await session.execute(insert(Task), tasks_data)
    await session.commit()

    return len(tasks_data)
```

**What this generates:**

```sql
-- SQLAlchemy + asyncpg turn this into an efficient batch operation.
-- Under the hood (simplified):
INSERT INTO tasks (title, description, category_id)
VALUES
    ('Task 1', '...', 1),
    ('Task 2', '...', 2),
    ('Task 3', '...', 1),
    ... (all rows in one statement)
```

**The difference is dramatic:**

```python
# Benchmark: inserting 10,000 tasks

# Method 1: add one at a time (DO NOT DO THIS)
# for data in tasks_data:
#     session.add(Task(**data))
#     await session.flush()
# → 27.3 seconds

# Method 2: add_all (ORM batch)
# session.add_all([Task(**d) for d in tasks_data])
# await session.commit()
# → 4.2 seconds

# Method 3: Core bulk insert
# await session.execute(insert(Task), tasks_data)
# await session.commit()
# → 0.6 seconds
```

```
┌─────────────────────────────────────────────────────────────────┐
│             10,000 ROW INSERT COMPARISON                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Method                Time       Speedup                      │
│  ──────                ────       ───────                      │
│  Individual add+flush  27.3s      1x (baseline)                │
│  add_all + commit      4.2s       6.5x faster                  │
│  Core execute(insert)  0.6s       45x faster                   │
│                                                                 │
│  VERIFY THIS YOURSELF with EXPLAIN ANALYZE (Lecture 1).         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Important: Core bulk insert does NOT return ORM objects.**

```python
# ✅ If you need the inserted IDs back, use RETURNING:
from sqlalchemy import insert
from models import Task

stmt = insert(Task).returning(Task.id)
result = await session.execute(stmt, tasks_data)
new_ids = result.scalars().all()  # [1, 2, 3, ...]
await session.commit()
```

---

## 4.4 Core Bulk UPDATE (execute + update)

**Same principle, for updates. Bulk-update many rows at once by primary key.**

```python
from sqlalchemy import update
from models import Task


async def bulk_update_status(
    session: AsyncSession,
    updates: list[dict],
) -> int:
    """
    Update many tasks at once.
    Each dict MUST include the primary key and the columns to change.
    
    updates = [
        {"id": 1, "status": "completed"},
        {"id": 2, "status": "in_progress"},
        {"id": 3, "status": "completed"},
    ]
    """
    if not updates:
        return 0

    await session.execute(update(Task), updates)
    await session.commit()
    return len(updates)
```

**How SQLAlchemy handles this under the hood:**

```
┌─────────────────────────────────────────────────────────────────┐
│              BULK UPDATE MECHANICS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  When you call: session.execute(update(Task), list_of_dicts)   │
│                                                                 │
│  SQLAlchemy matches each dict's "id" key to the primary key    │
│  and generates individual UPDATE statements, but batches them   │
│  efficiently into a single database round trip.                 │
│                                                                 │
│  Input:                                                         │
│  [{"id": 1, "status": "done"}, {"id": 2, "status": "done"}]   │
│                                                                 │
│  Becomes (conceptually):                                        │
│  UPDATE tasks SET status='done' WHERE id=1;                     │
│  UPDATE tasks SET status='done' WHERE id=2;                     │
│  (batched into one network call by asyncpg)                     │
│                                                                 │
│  NOTE: Each dict MUST have the primary key column.              │
│  The PK tells SQLAlchemy WHICH row to update.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**For bulk updates where ALL rows get the same change, use a WHERE clause instead:**

```python
from sqlalchemy import update
from models import Task

async def archive_old_tasks(session: AsyncSession) -> int:
    """Mark all completed tasks older than 90 days as archived."""
    from datetime import datetime, timedelta

    cutoff = datetime.utcnow() - timedelta(days=90)

    stmt = (
        update(Task)
        .where(Task.status == "completed", Task.updated_at < cutoff)
        .values(status="archived")
    )
    result = await session.execute(stmt)
    await session.commit()

    return result.rowcount  # Number of rows actually updated
```

This generates a single SQL statement, no matter how many rows match. Far more efficient than loading thousands of ORM objects and modifying them one by one.

---

## 4.5 Choosing the Right Tool

```
┌─────────────────────────────────────────────────────────────────┐
│              WHICH BULK METHOD DO I USE?                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────┐                      │
│  │  Need ORM features (events,           │                      │
│  │  relationships, Python defaults)?     │                      │
│  └────────────────┬──────────────────────┘                      │
│             Yes   │   No                                        │
│              │    │    │                                         │
│              ▼    │    ▼                                         │
│  ┌──────────────┐ │  ┌────────────────────────┐                 │
│  │ add_all()    │ │  │ How many rows?          │                 │
│  │ + commit     │ │  └──────────┬─────────────┘                 │
│  │              │ │        │         │                           │
│  │ ~Hundreds OK │ │    <1,000    >1,000                         │
│  └──────────────┘ │        │         │                           │
│                   │        ▼         ▼                           │
│                   │  ┌──────────┐  ┌───────────────────┐        │
│                   │  │ execute  │  │ execute(insert)   │        │
│                   │  │ (insert) │  │ in BATCHES of     │        │
│                   │  │ one shot │  │ 1,000-5,000       │        │
│                   │  └──────────┘  └───────────────────┘        │
│                   │                                             │
│                                                                 │
│  SAME ROW VALUE FOR ALL?                                        │
│  ─────────────────────                                          │
│  YES → update().where(...).values(...)    (single statement)    │
│  NO  → execute(update(Model), list_dicts) (per-row values)     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Batching very large inserts:**

```python
async def import_massive_dataset(
    session: AsyncSession,
    rows: list[dict],
    batch_size: int = 5000,
) -> int:
    """Insert hundreds of thousands of rows in manageable batches."""
    total = 0

    for i in range(0, len(rows), batch_size):
        batch = rows[i : i + batch_size]
        await session.execute(insert(Task), batch)
        await session.commit()  # Commit each batch — don't hold a huge transaction
        total += len(batch)
        print(f"Inserted {total}/{len(rows)} rows...")

    return total
```

> "Why batch? If you send 500,000 rows in one statement, you risk: memory exhaustion on the database, transaction log bloat, and a very long-running transaction that blocks other operations. Batches of 1,000–5,000 are a sweet spot."

---

## 4.6 Legacy Awareness (What You'll See in Old Code)

**You will encounter these in existing codebases. Do NOT use them in new code.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 LEGACY BULK METHODS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ session.bulk_save_objects([obj1, obj2, ...])                │
│     SQLAlchemy 1.x approach. Deprecated in 2.0.                 │
│     You will see this in Stack Overflow answers and old code.   │
│     Migrate to: session.add_all() or execute(insert(Model))    │
│                                                                 │
│  ❌ session.bulk_insert_mappings(Model, [dict1, dict2, ...])    │
│     Another 1.x approach. Deprecated.                           │
│     Migrate to: session.execute(insert(Model), list_of_dicts)  │
│                                                                 │
│  ❌ session.bulk_update_mappings(Model, [dict1, dict2, ...])    │
│     Deprecated update equivalent.                               │
│     Migrate to: session.execute(update(Model), list_of_dicts)  │
│                                                                 │
│  If you see these in a codebase:                                │
│  1. They still work (backward compatible, for now).             │
│  2. Don't add more. Use the 2.0 patterns from this lecture.    │
│  3. Refactor them when you touch that code.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: RAW SQL WHEN ORM ISN'T ENOUGH

## 5.1 text() — The Escape Hatch

**The ORM is powerful. But sometimes it generates bad SQL, or can't express what you need. That's when you use `text()`.**

You saw raw SQL briefly in Week 5 when learning SQL fundamentals. Now you're combining it with SQLAlchemy's session and the async engine.

```python
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


async def get_task_statistics(session: AsyncSession) -> dict:
    """
    Complex aggregate query. The ORM CAN express this, but the
    raw SQL is clearer and more maintainable for complex analytics.
    """
    query = text("""
        SELECT
            c.name AS category_name,
            COUNT(t.id) AS total_tasks,
            COUNT(t.id) FILTER (WHERE t.status = 'completed') AS completed,
            ROUND(
                COUNT(t.id) FILTER (WHERE t.status = 'completed') * 100.0
                / NULLIF(COUNT(t.id), 0),
                1
            ) AS completion_rate
        FROM tasks t
        JOIN categories c ON c.id = t.category_id
        GROUP BY c.name
        ORDER BY total_tasks DESC
    """)

    result = await session.execute(query)
    rows = result.mappings().all()
    # rows = [{"category_name": "Backend", "total_tasks": 45, ...}, ...]

    return {"categories": [dict(row) for row in rows]}
```

**CRITICAL: Always use bind parameters. Never string-format SQL.**

```python
# ❌ SQL INJECTION VULNERABILITY — NEVER DO THIS
user_input = "'; DROP TABLE tasks; --"
query = text(f"SELECT * FROM tasks WHERE title = '{user_input}'")
# Congratulations, your tasks table is gone.

# ✅ SAFE: Use bind parameters (SQLAlchemy parameterizes them)
query = text("SELECT * FROM tasks WHERE title = :title")
result = await session.execute(query, {"title": user_input})
# The :title placeholder is parameterized — user input is ESCAPED.
# No injection possible.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   BIND PARAMETERS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SYNTAX:  :parameter_name in the SQL string                    │
│  VALUES:  passed as a dict to session.execute()                │
│                                                                 │
│  text("SELECT * FROM tasks WHERE id = :id AND status = :s")   │
│  session.execute(query, {"id": 42, "s": "completed"})         │
│                                                                 │
│  PostgreSQL receives:                                           │
│  SELECT * FROM tasks WHERE id = $1 AND status = $2             │
│  Parameters: [42, 'completed']                                  │
│                                                                 │
│  The values are NEVER interpolated into the SQL string.         │
│  The database treats them as DATA, not as SQL code.             │
│  This is what makes parameterized queries injection-proof.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Combining raw SQL results with Pydantic (Week 3 connection):**

```python
from pydantic import BaseModel


class CategoryStats(BaseModel):
    category_name: str
    total_tasks: int
    completed: int
    completion_rate: float


async def get_stats(session: AsyncSession) -> list[CategoryStats]:
    query = text("""
        SELECT c.name AS category_name, COUNT(t.id) AS total_tasks,
               COUNT(t.id) FILTER (WHERE t.status = 'completed') AS completed,
               ROUND(COUNT(t.id) FILTER (WHERE t.status = 'completed') * 100.0
                     / NULLIF(COUNT(t.id), 0), 1) AS completion_rate
        FROM tasks t
        JOIN categories c ON c.id = t.category_id
        GROUP BY c.name
    """)
    result = await session.execute(query)
    rows = result.mappings().all()

    return [CategoryStats(**row) for row in rows]
```

---

## 5.2 Hybrid Properties (Python + SQL in One)

**A hybrid property is a computed attribute that works BOTH in Python AND in SQL queries.**

```python
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy import String, case
from models import Base


class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    total_subtasks: Mapped[int] = mapped_column(default=0)
    completed_subtasks: Mapped[int] = mapped_column(default=0)
    status: Mapped[str] = mapped_column(String(20), default="pending")

    @hybrid_property
    def progress_percent(self) -> float:
        """Works on Python objects — regular calculation."""
        if self.total_subtasks == 0:
            return 0.0
        return (self.completed_subtasks / self.total_subtasks) * 100

    @progress_percent.expression
    @classmethod
    def progress_percent(cls):
        """Works in SQL queries — translated to SQL expression."""
        return case(
            (cls.total_subtasks == 0, 0.0),
            else_=(
                cls.completed_subtasks * 100.0 / cls.total_subtasks
            ),
        )

    @hybrid_property
    def is_overdue(self) -> bool:
        """Python side: check if past due date."""
        from datetime import datetime
        if self.due_date is None:
            return False
        return datetime.utcnow() > self.due_date

    @is_overdue.expression
    @classmethod
    def is_overdue(cls):
        """SQL side: compare in the database."""
        from sqlalchemy import func
        return func.now() > cls.due_date
```

**Now you can use these in queries:**

```python
from sqlalchemy import select
from models import Task

# Filter by computed property — runs IN THE DATABASE as SQL
stmt = select(Task).where(Task.progress_percent > 75.0)
result = await session.execute(stmt)
almost_done = result.scalars().all()

# Also works on an individual Python object
task = almost_done[0]
print(task.progress_percent)  # 80.0 — calculated in Python

# Find overdue tasks
overdue_stmt = select(Task).where(Task.is_overdue == True)
overdue_tasks = (await session.execute(overdue_stmt)).scalars().all()
```

```
┌─────────────────────────────────────────────────────────────────┐
│              HYBRID PROPERTY — TWO FACES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ON A PYTHON OBJECT:                                            │
│  ────────────────────                                           │
│  task = Task(total_subtasks=10, completed_subtasks=8)           │
│  task.progress_percent  → 80.0                                  │
│  Calls the @hybrid_property method. Pure Python.                │
│                                                                 │
│  IN A SQL QUERY:                                                │
│  ────────────────                                               │
│  select(Task).where(Task.progress_percent > 75)                │
│  → SELECT * FROM tasks                                          │
│    WHERE (completed_subtasks * 100.0 / total_subtasks) > 75    │
│  Calls the @expression method. Translated to SQL.               │
│                                                                 │
│  SAME ATTRIBUTE, TWO BEHAVIORS.                                 │
│  Python code and SQL queries both use .progress_percent         │
│  but each side gets the right implementation.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 When to Break Out of the ORM

```
┌─────────────────────────────────────────────────────────────────┐
│              ORM vs RAW SQL — THE DECISION                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE THE ORM (SQLAlchemy select/insert/update) WHEN:           │
│  ├─ Standard CRUD operations                                    │
│  ├─ Simple to moderate queries (joins, filters, ordering)       │
│  ├─ You need relationship loading (joinedload, selectinload)    │
│  ├─ You want Python-level validation and ORM events             │
│  └─ Portability across databases matters                        │
│                                                                 │
│  USE text() RAW SQL WHEN:                                       │
│  ├─ Complex aggregation / analytics (GROUP BY with many calcs)  │
│  ├─ PostgreSQL-specific features (window functions, CTEs, JSONB │
│  │   operators, full-text search ranking)                       │
│  ├─ The ORM generates inefficient SQL (check with echo=True)   │
│  ├─ Performance-critical queries where you want full control    │
│  └─ One-off data migration or reporting scripts                 │
│                                                                 │
│  USE HYBRID PROPERTIES WHEN:                                    │
│  ├─ A computed value is needed both in Python AND SQL queries   │
│  ├─ You want to filter/sort by a derived attribute              │
│  └─ The calculation is simple enough to express in SQL          │
│                                                                 │
│  ⚠️  CAUTION:                                                   │
│  Raw SQL bypasses the ORM's type checking and relationship      │
│  management. You're responsible for correctness.                │
│  Always use bind parameters. Always.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 6: MULTI-TENANCY & SEEDING

## 6.1 Multi-Tenancy: One Database, Many Organizations

**Your Task Manager serves multiple companies. Company A must NEVER see Company B's tasks.**

This is called multi-tenancy: multiple tenants (organizations) sharing one application and one database, with strict data isolation between them. You will build this in the capstone project (Week 13-14).

```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-TENANCY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BANK ANALOGY:                                                  │
│  Multiple companies bank at the same branch.                    │
│  Same vault, same tellers, same systems.                        │
│  But Company A can NEVER see Company B's accounts.              │
│                                                                 │
│  TWO APPROACHES:                                                │
│                                                                 │
│  ROW-BASED ISOLATION                                            │
│  ─────────────────────                                          │
│  Same tables. Every row has an organization_id column.          │
│  Every query filters by organization_id.                        │
│                                                                 │
│  ┌──────────────────────────────────────────────────┐           │
│  │  tasks table                                     │           │
│  │  id │ title          │ org_id │ status            │           │
│  │  1  │ "Fix login"    │   1    │ "pending"         │           │
│  │  2  │ "Add search"   │   1    │ "done"            │           │
│  │  3  │ "Deploy v2"    │   2    │ "pending"  ← Org 2│           │
│  │  4  │ "Write docs"   │   2    │ "done"     ← Org 2│           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
│                                                                 │
│  SCHEMA-BASED ISOLATION                                         │
│  ──────────────────────                                         │
│  Same database. Separate PostgreSQL schemas per org.            │
│  Completely separate tables — org1.tasks, org2.tasks.           │
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │ schema: org_1   │    │ schema: org_2   │                     │
│  │ ┌─────────────┐ │    │ ┌─────────────┐ │                     │
│  │ │ tasks       │ │    │ │ tasks       │ │                     │
│  │ │ users       │ │    │ │ users       │ │                     │
│  │ │ categories  │ │    │ │ categories  │ │                     │
│  │ └─────────────┘ │    │ └─────────────┘ │                     │
│  └─────────────────┘    └─────────────────┘                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 Row-Based Isolation (The Practical Approach)

**This is what you'll use 95% of the time. Simpler, proven at scale.**

**Step 1: Add organization_id to every tenant-scoped model.**

```python
# models.py
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column, relationship
from models import Base


class Organization(Base):
    __tablename__ = "organizations"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))


class Task(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[str] = mapped_column(default="")
    status: Mapped[str] = mapped_column(String(20), default="pending")

    # EVERY tenant-scoped table gets this column
    organization_id: Mapped[int] = mapped_column(
        ForeignKey("organizations.id"),
        index=True,  # Always index — you filter by this on every query
    )

    organization: Mapped[Organization] = relationship()
```

**Step 2: ALWAYS filter by organization_id in every query.**

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from models import Task


class TaskRepository:
    """Repository that enforces tenant isolation."""

    def __init__(self, session: AsyncSession, org_id: int):
        self._session = session
        self._org_id = org_id  # Set once, used in EVERY method

    async def get_all(self) -> list[Task]:
        stmt = (
            select(Task)
            .where(Task.organization_id == self._org_id)  # ← ALWAYS
            .order_by(Task.created_at.desc())
        )
        result = await self._session.execute(stmt)
        return list(result.scalars().all())

    async def get_by_id(self, task_id: int) -> Task | None:
        stmt = select(Task).where(
            Task.id == task_id,
            Task.organization_id == self._org_id,  # ← ALWAYS
            # Without this, a user from Org 1 could read Org 2's tasks
            # just by guessing the task ID!
        )
        result = await self._session.execute(stmt)
        return result.scalar_one_or_none()

    async def create(self, title: str, description: str = "") -> Task:
        task = Task(
            title=title,
            description=description,
            organization_id=self._org_id,  # ← ALWAYS set on create
        )
        self._session.add(task)
        await self._session.commit()
        return task
```

**The danger:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  ⚠️  THE ONE BUG THAT LEAKS DATA               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  If you FORGET the organization_id filter in even ONE query:    │
│                                                                 │
│  # ❌ MISSING FILTER — returns ALL organizations' tasks!        │
│  stmt = select(Task).where(Task.id == task_id)                 │
│                                                                 │
│  A user from Org 1 sends: GET /tasks/42                         │
│  Task #42 belongs to Org 2.                                     │
│  Without the org filter → they see it.                          │
│  → DATA BREACH.                                                 │
│                                                                 │
│  This is why the repository takes org_id in __init__.           │
│  It's impossible to call get_by_id() without the org_id         │
│  already being set. The architecture prevents the mistake.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 Schema-Based Isolation (The Nuclear Approach)

**For when row-based isn't enough. Separate PostgreSQL schemas per tenant.**

```
┌─────────────────────────────────────────────────────────────────┐
│              SCHEMA-BASED — OVERVIEW ONLY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HOW IT WORKS:                                                  │
│  ─────────────                                                  │
│  PostgreSQL supports multiple schemas in one database.          │
│  Each organization gets its own schema: org_1, org_2, etc.     │
│  Each schema has its own copy of every table.                   │
│                                                                 │
│  -- Org 1's tasks:                                              │
│  SELECT * FROM org_1.tasks;                                     │
│                                                                 │
│  -- Org 2's tasks:                                              │
│  SELECT * FROM org_2.tasks;                                     │
│                                                                 │
│  You switch schemas at the connection level:                    │
│  SET search_path TO org_1;  -- All queries now use org_1's     │
│                              -- tables automatically            │
│                                                                 │
│  ADVANTAGES:                                                    │
│  ├─ Complete isolation (no accidental cross-tenant access)      │
│  ├─ Per-tenant schema customization possible                    │
│  └─ Easier per-tenant backup/restore                            │
│                                                                 │
│  DISADVANTAGES:                                                 │
│  ├─ Alembic migrations must run per-schema (complex)            │
│  ├─ Connection pool complexity (schema switching)               │
│  ├─ Harder to query across tenants (admin/analytics)            │
│  └─ More operational overhead (1000 orgs = 1000 schemas)        │
│                                                                 │
│  FOR THIS COURSE:                                               │
│  We use ROW-BASED in all projects. Schema-based is here so      │
│  you recognize it when you encounter it professionally.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.4 Database Seeding (Populating Development Data)

**When developing locally, you need realistic data. Not 3 rows — hundreds.**

**Step 1: Factory functions that generate realistic data.**

```python
# seed/factories.py
from faker import Faker
from datetime import datetime, timedelta
import random

fake = Faker()


def make_organization(**overrides) -> dict:
    """Generate one organization's data."""
    defaults = {
        "name": fake.company(),
    }
    return {**defaults, **overrides}


def make_task(organization_id: int, category_id: int, **overrides) -> dict:
    """Generate one task's data."""
    created = fake.date_time_between(start_date="-90d", end_date="now")
    defaults = {
        "title": fake.sentence(nb_words=5)[:-1],  # Remove trailing period
        "description": fake.paragraph(nb_sentences=3),
        "status": random.choice(["pending", "in_progress", "completed"]),
        "organization_id": organization_id,
        "category_id": category_id,
        "created_at": created,
        "updated_at": created + timedelta(days=random.randint(0, 30)),
    }
    return {**defaults, **overrides}
```

**Step 2: A seed script that populates the database.**

```python
# seed/run_seed.py
import asyncio
from sqlalchemy import insert
from database import engine, AsyncSessionLocal
from models import Organization, Category, Task
from seed.factories import make_organization, make_task


async def seed():
    async with AsyncSessionLocal() as session:
        # 1. Create organizations
        orgs_data = [make_organization() for _ in range(5)]
        result = await session.execute(
            insert(Organization).returning(Organization.id), orgs_data
        )
        org_ids = result.scalars().all()
        print(f"Created {len(org_ids)} organizations")

        # 2. Create categories (shared across orgs)
        categories_data = [
            {"name": "Backend"},
            {"name": "Frontend"},
            {"name": "DevOps"},
            {"name": "Design"},
        ]
        result = await session.execute(
            insert(Category).returning(Category.id), categories_data
        )
        cat_ids = result.scalars().all()
        print(f"Created {len(cat_ids)} categories")

        # 3. Create tasks — 200 per org, using bulk insert
        import random

        all_tasks = []
        for org_id in org_ids:
            for _ in range(200):
                all_tasks.append(
                    make_task(
                        organization_id=org_id,
                        category_id=random.choice(cat_ids),
                    )
                )

        # Bulk insert all 1,000 tasks at once (Part 4 of this lecture!)
        await session.execute(insert(Task), all_tasks)
        await session.commit()
        print(f"Created {len(all_tasks)} tasks across {len(org_ids)} orgs")


if __name__ == "__main__":
    asyncio.run(seed())
```

**Run it:**

```
$ uv run python -m seed.run_seed

Created 5 organizations
Created 4 categories
Created 1000 tasks across 5 orgs
```

**Why seed data matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHY SEED?                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. REALISTIC TESTING                                           │
│     Empty databases hide bugs. Pagination? Doesn't matter       │
│     with 3 rows. Performance? Looks fine with 3 rows.           │
│     1,000 rows reveal real behavior.                            │
│                                                                 │
│  2. DEMO-READY                                                  │
│     When showing your app to others (or in job interviews),     │
│     a populated app is 10x more impressive than an empty one.   │
│                                                                 │
│  3. ONBOARDING                                                  │
│     New developer joins your team. They run one command:        │
│     uv run python -m seed.run_seed                              │
│     Now they have a working local environment with data.        │
│                                                                 │
│  4. QUERY OPTIMIZATION                                          │
│     You can't run EXPLAIN ANALYZE (Lecture 1) meaningfully      │
│     on an empty table. The query planner behaves differently    │
│     with 10 rows vs 10,000 rows.                                │
│                                                                 │
│  RULE: Your seed script should be checked into Git.             │
│  It's part of your codebase, not a one-off throwaway.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Install Faker:**

```bash
uv add faker --dev  # Development dependency only — never in production
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              ADVANCED PATTERNS QUICK REFERENCE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONNECTION POOLING:                                            │
│      create_async_engine(                                       │
│          url,                                                   │
│          pool_size=10,                                          │
│          max_overflow=20,                                       │
│          pool_recycle=3600,                                     │
│          pool_timeout=30,                                       │
│          pool_pre_ping=True,                                    │
│      )                                                          │
│                                                                 │
│  PESSIMISTIC LOCK (hold row, block others):                     │
│      select(Model).with_for_update()                            │
│      select(Model).with_for_update(nowait=True)                 │
│      select(Model).with_for_update(skip_locked=True)            │
│                                                                 │
│  OPTIMISTIC LOCK (version check on update):                     │
│      update(Model)                                              │
│        .where(Model.id == id, Model.version == expected)        │
│        .values(data=new_data, version=expected + 1)             │
│      if result.rowcount == 0: raise OptimisticLockError()       │
│                                                                 │
│  BULK INSERT (Core — fastest):                                  │
│      await session.execute(insert(Model), [dict1, dict2, ...]) │
│                                                                 │
│  BULK INSERT (ORM — with tracking):                             │
│      session.add_all([obj1, obj2, ...])                         │
│      await session.commit()                                     │
│                                                                 │
│  BULK UPDATE (same value for all matching rows):                │
│      update(Model).where(...).values(status="done")             │
│                                                                 │
│  BULK UPDATE (different value per row):                         │
│      await session.execute(update(Model), [dicts_with_pk...])  │
│                                                                 │
│  RAW SQL:                                                       │
│      from sqlalchemy import text                                │
│      await session.execute(                                     │
│          text("SELECT * FROM t WHERE id = :id"), {"id": 42}    │
│      )                                                          │
│      ⚠️  Always use :param bind parameters. Never f-strings.    │
│                                                                 │
│  HYBRID PROPERTY:                                               │
│      @hybrid_property                                           │
│      def computed(self) -> T:  (Python side)                    │
│      @computed.expression                                       │
│      @classmethod                                               │
│      def computed(cls):  (SQL side)                             │
│                                                                 │
│  MULTI-TENANCY (row-based):                                     │
│      Every model: organization_id column (indexed)              │
│      Every query: .where(Model.organization_id == org_id)       │
│      Repository: takes org_id in __init__, enforces everywhere  │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Missing async with → connection leak                    │
│      ❌ pool_size > DB max_connections → refused connections    │
│      ❌ time.sleep() in async code → blocks the pool            │
│      ❌ Forgetting org_id filter → cross-tenant data leak      │
│      ❌ String-formatting SQL → injection vulnerability        │
│      ❌ Individual inserts in a loop → N+1 writes              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  YOUR DATABASE IS A SHARED RESOURCE UNDER PRESSURE.             │
│  THESE PATTERNS PROTECT IT.                                     │
│                                                                 │
│  ┌──────────────┐    ┌────────────┐    ┌────────────────────┐   │
│  │  CONNECTION   │    │  LOCKING   │    │  BULK OPERATIONS   │   │
│  │  POOLING      │    │            │    │                    │   │
│  │              │    │            │    │                    │   │
│  │  Manage HOW  │    │  Manage    │    │  Manage HOW        │   │
│  │  MANY can    │    │  WHO can   │    │  EFFICIENTLY       │   │
│  │  talk to     │    │  change    │    │  you write         │   │
│  │  the DB at   │    │  the same  │    │  data              │   │
│  │  once        │    │  data      │    │                    │   │
│  └──────────────┘    └────────────┘    └────────────────────┘   │
│         │                  │                     │               │
│         │                  │                     │               │
│         └──────────────────┴─────────────────────┘               │
│                            │                                    │
│                   ┌────────────────┐                             │
│                   │  A WELL-TUNED  │                             │
│                   │  DATABASE LAYER│                             │
│                   └────────────────┘                             │
│                                                                 │
│  THE BANK ANALOGY:                                              │
│  ├─ Connection Pool = Teller windows (limited, reused)          │
│  ├─ Pessimistic Lock = "Account locked while in use"            │
│  ├─ Optimistic Lock = "Check your balance slip is current"      │
│  ├─ Bulk Operations = Payroll batch (one trip, many deposits)   │
│  ├─ Raw SQL = Custom vault instruction (when standard won't do) │
│  ├─ Multi-tenancy = Separate ledgers per company                │
│  └─ Seeding = Training accounts for new employees               │
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
│  WEEK 7 (Lecture 3, NoSQL):                                     │
│  └─ Redis and MongoDB — different data stores, different        │
│     trade-offs. Connection pooling concepts carry over.         │
│                                                                 │
│  WEEK 7 PROJECT COMPLETION:                                     │
│  └─ Apply pool tuning and bulk operations to your Task Manager. │
│     Document before/after performance with EXPLAIN ANALYZE.     │
│                                                                 │
│  WEEK 8 (External APIs):                                        │
│  └─ Fetching from external APIs + storing results in your DB.   │
│     Bulk insert external data. Rate limiting uses Redis         │
│     counters (similar to pool counting).                        │
│                                                                 │
│  WEEK 9 (Authentication):                                       │
│  └─ User registration = INSERT with uniqueness checking.        │
│     Optimistic locking concepts apply to user profiles.         │
│     Multi-tenancy enforced through auth dependencies.           │
│                                                                 │
│  WEEK 10 (Redis Caching):                                       │
│  └─ Connection pooling for Redis follows the same principles.   │
│     Cache invalidation on write connects to bulk update events. │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Multi-tenancy is a CORE REQUIREMENT.                        │
│     Row-based isolation from this lecture is the foundation.    │
│     Bulk operations for data import/export features.            │
│     Connection pool tuning for load testing.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```