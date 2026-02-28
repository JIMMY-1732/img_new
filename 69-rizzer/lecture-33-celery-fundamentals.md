# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ARCHITECTURE BEFORE CODE                                       │
│  ────────────────────────                                       │
│  Students must see the DISTRIBUTED SYSTEM before writing a      │
│  single line of Celery code. Three processes, not one.          │
│                                                                 │
│  THE FACTORY FLOOR ANALOGY                                      │
│  ─────────────────────────                                      │
│  Every Celery concept maps to a factory that receives orders,   │
│  builds products, and reports completion. Tangible.             │
│                                                                 │
│  THE SYNC SURPRISE                                              │
│  ─────────────────                                              │
│  After 10 weeks of async/await, students will instinctively     │
│  write "async def" for tasks. Celery tasks are SYNC.            │
│  We confront this head-on and explain WHY.                      │
│                                                                 │
│  CONNECT TO PRIOR KNOWLEDGE                                     │
│  ─────────────────────────                                      │
│  Redis (Week 10) → Celery uses it as the message broker         │
│  Decorators (Week 1) → @app.task is another decorator           │
│  Retry patterns (Week 8) → Same concept, new mechanism          │
│  Idempotent APIs (Week 4) → Same principle, higher stakes       │
│  BackgroundTasks (Lecture 1) → The problem Celery solves        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                    CELERY FUNDAMENTALS                          │
│                     (3-4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE DEATH TEST (25 min)                                │
│  ├─ 1.1 Killing Your Server Mid-Task (Demonstration)            │
│  ├─ 1.2 What We Actually Need                                   │
│  └─ 1.3 Enter Celery: Work That Survives                        │
│                                                                 │
│  PART 2: THE FACTORY FLOOR (35 min)                             │
│  ├─ 2.1 The Factory Analogy                                     │
│  ├─ 2.2 Three Components: Client, Broker, Worker                │
│  ├─ 2.3 Message Flow (How an Order Becomes a Product)           │
│  ├─ 2.4 Redis as Broker (Your Week 10 Knowledge Pays Off)       │
│  └─ 2.5 The Sync Surprise                                       │
│                                                                 │
│  PART 3: YOUR FIRST CELERY TASK (55 min)                        │
│  ├─ 3.1 Creating the Celery App                                 │
│  ├─ 3.2 Project Structure (Two Processes, One Codebase)         │
│  ├─ 3.3 Defining Tasks with @app.task                           │
│  ├─ 3.4 Running the Worker                                      │
│  ├─ 3.5 Calling Tasks: .delay() and .apply_async()              │
│  └─ 3.6 Serialization: Tasks Are Messages, Not Function Calls   │
│                                                                 │
│  PART 4: TRACKING WORK — RESULTS & STATES (25 min)             │
│  ├─ 4.1 AsyncResult — Your Tracking Number                      │
│  ├─ 4.2 The Task State Machine                                  │
│  ├─ 4.3 Checking Status from FastAPI                            │
│  └─ 4.4 When You Don't Need Results                             │
│                                                                 │
│  PART 5: WHEN THINGS GO WRONG — RETRIES (45 min)               │
│  ├─ 5.1 What Happens When a Task Fails?                         │
│  ├─ 5.2 bind=True — Giving the Task Self-Awareness              │
│  ├─ 5.3 Manual Retry with self.retry()                          │
│  ├─ 5.4 Declarative Retry with autoretry_for                    │
│  ├─ 5.5 Exponential Backoff (Week 8 Callback)                   │
│  └─ 5.6 After Max Retries: Accepting Failure                    │
│                                                                 │
│  PART 6: THE NON-NEGOTIABLE — IDEMPOTENCY (30 min)             │
│  ├─ 6.1 At-Least-Once Delivery (The Uncomfortable Truth)        │
│  ├─ 6.2 How Duplicates Happen                                   │
│  ├─ 6.3 Making Tasks Idempotent (Patterns)                      │
│  └─ 6.4 Database-Backed Idempotency Keys                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE DEATH TEST

## 1.1 Killing Your Server Mid-Task

**Last lecture, you learned WHY background processing matters and saw BackgroundTasks. Now watch what happens when the real world intervenes.**

```python
# demo_fragile.py — A FastAPI server with BackgroundTasks
from fastapi import FastAPI, BackgroundTasks
import time
import uvicorn

app = FastAPI()

emails_sent: list[str] = []

def send_email_slowly(to: str, subject: str) -> None:
    """Simulate sending an email (takes 3 seconds)"""
    print(f"📧 Sending email to {to}...")
    time.sleep(3)  # Simulate SMTP connection + send
    emails_sent.append(to)
    print(f"✅ Email sent to {to}! (Total sent: {len(emails_sent)})")

@app.post("/notify-all")
async def notify_all_users(background_tasks: BackgroundTasks):
    users = [
        "alice@example.com",
        "bob@example.com",
        "charlie@example.com",
        "diana@example.com",
        "eve@example.com",
    ]
    for user in users:
        background_tasks.add_task(send_email_slowly, user, "Important Update")
    
    return {"message": f"Queued {len(users)} emails", "status": "accepted"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Run it. Hit the endpoint. Then kill the server after 5 seconds.**

```
Terminal 1:                          Terminal 2:
$ python demo_fragile.py             $ curl -X POST localhost:8000/notify-all
                                     {"message":"Queued 5 emails","status":"accepted"}

📧 Sending email to alice@example.com...
✅ Email sent to alice@example.com! (Total sent: 1)
📧 Sending email to bob@example.com...

  *** You press Ctrl+C here ***

^C  Shutting down...
```

**Now ask the class:**

> "Five emails were queued. Alice got hers. Bob's was in progress. Charlie, Diana, Eve — never started. How many actually sent? You don't know for certain. Bob's might be half-sent. And here's the real question: **when you restart the server, do those 3 unsent emails come back?**"

Answer: **No. They're gone. Forever.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE DEATH TEST RESULT                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE CRASH:                                                  │
│  ├─ alice@example.com    ✅ Sent                                │
│  ├─ bob@example.com      ⚠️  In progress (corrupted?)           │
│  ├─ charlie@example.com  ❌ Never started                       │
│  ├─ diana@example.com    ❌ Never started                       │
│  └─ eve@example.com      ❌ Never started                       │
│                                                                 │
│  AFTER RESTART:                                                 │
│  ├─ emails_sent = []     ← Memory wiped                        │
│  ├─ No record of what was queued                                │
│  ├─ No record of what succeeded                                 │
│  └─ No way to resume                                            │
│                                                                 │
│  BackgroundTasks live in YOUR PROCESS MEMORY.                   │
│  When the process dies, the tasks die with it.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 What We Actually Need

**BackgroundTasks has three fatal limitations for work that matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                THREE FATAL LIMITATIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. NO PERSISTENCE                                              │
│     Tasks live in RAM. Process dies → tasks vanish.             │
│     You can't restart and resume.                               │
│                                                                 │
│  2. NO DISTRIBUTION                                             │
│     All tasks run in the SAME process as your web server.       │
│     100 emails to send? Your API slows down for everyone.       │
│     Can't spread work across multiple machines.                 │
│                                                                 │
│  3. NO VISIBILITY                                               │
│     Is the task running? Failed? Stuck? No way to know.         │
│     No retry. No monitoring. No status checking.                │
│                                                                 │
│  BackgroundTasks is fine for: "log this request" (fire-forget)  │
│  BackgroundTasks BREAKS for: "send this invoice" (must happen)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 Enter Celery: Work That Survives

**Celery is a distributed task queue. The key word is DISTRIBUTED — the work doesn't live in your web server.**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│    BackgroundTasks                    Celery                     │
│    ─────────────────                 ──────────────              │
│                                                                 │
│    ┌─────────────────┐               ┌──────────┐               │
│    │   FastAPI        │               │ FastAPI  │               │
│    │   ┌───────────┐ │               │ (sends   │               │
│    │   │ Background│ │               │  message)│               │
│    │   │ Tasks     │ │               └────┬─────┘               │
│    │   │ (in RAM)  │ │                    │                      │
│    │   └───────────┘ │                    ▼                      │
│    │   Process dies → │               ┌──────────┐              │
│    │   Tasks die      │               │  Redis   │ ← persisted  │
│    └─────────────────┘               │  (broker)│   on disk     │
│                                       └────┬─────┘              │
│    One process.                            │                    │
│    One machine.                            ▼                    │
│    No persistence.                    ┌──────────┐              │
│    No visibility.                     │  Celery  │              │
│                                       │  Worker  │ ← separate  │
│                                       │ (does    │   process    │
│                                       │  work)   │              │
│                                       └──────────┘              │
│                                                                 │
│                                       Separate process.         │
│                                       Can be separate machine.  │
│                                       Tasks survive crashes.    │
│                                       Full visibility.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Celery separates the REQUEST for work from the EXECUTION of work. Your web server says 'do this,' writes it down in a durable place, and walks away. A completely different process picks it up and does it. If either process crashes, the work order still exists."

---

# PART 2: THE FACTORY FLOOR

## 2.1 The Factory Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE FACTORY FLOOR                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine a factory that builds custom furniture.                │
│                                                                 │
│  THE SALES OFFICE (FastAPI)                                     │
│  ──────────────────────────                                     │
│  Customers walk in and place orders.                            │
│  The sales office does NOT build furniture.                     │
│  They write an ORDER FORM and put it in the factory's inbox.    │
│  Then they immediately help the next customer.                  │
│                                                                 │
│  THE INBOX CONVEYOR BELT (Redis Broker)                         │
│  ──────────────────────────────────────                         │
│  A conveyor belt carries order forms to the factory floor.      │
│  Orders sit in the queue until a worker picks one up.           │
│  If the sales office closes for the night, orders stay          │
│  on the belt — they don't disappear.                            │
│                                                                 │
│  THE FACTORY WORKERS (Celery Workers)                           │
│  ─────────────────────────────────────                          │
│  Workers stand at the belt, grab the next order form,           │
│  and build the product. One worker, one order at a time.        │
│  When finished, they put the product on the "completed" shelf   │
│  and grab the next order.                                       │
│                                                                 │
│  THE COMPLETED SHELF (Redis Result Backend)                     │
│  ──────────────────────────────────────────                     │
│  Finished products sit on a shelf with their order number.      │
│  The sales office can check: "Is order #472 done yet?"          │
│                                                                 │
│  THE ORDER NUMBER (task_id / AsyncResult)                       │
│  ─────────────────────────────────────────                      │
│  Every order gets a unique tracking number.                     │
│  Check status anytime: pending, in progress, done, failed.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to Celery:**

```
Factory                 │  Celery
────────────────────────│─────────────────────────
Sales office            │  FastAPI (your web server)
Customer order          │  Task call (send_email.delay(...))
Order form              │  Serialized message (JSON in Redis)
Inbox conveyor belt     │  Redis broker queue
Factory worker          │  Celery worker process
Building the product    │  Executing the task function
Order tracking number   │  task_id (UUID)
Completed shelf         │  Redis result backend
Checking order status   │  AsyncResult.status
Defective → redo        │  Task retry
Worker calls in sick    │  Worker process crashes (others continue)
```

---

## 2.2 Three Components: Client, Broker, Worker

**Every Celery system has exactly three components:**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE THREE COMPONENTS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐    │
│   │   CLIENT    │      │   BROKER    │      │   WORKER    │    │
│   │  (FastAPI)  │─────▶│   (Redis)   │─────▶│  (Celery)   │    │
│   │             │      │             │      │             │    │
│   │ "Do this    │      │ "I'll hold  │      │ "I'll do    │    │
│   │  task"      │      │  the order" │      │  the work"  │    │
│   └─────────────┘      └─────────────┘      └──────┬──────┘    │
│                                                     │           │
│                         ┌─────────────┐             │           │
│                         │   RESULT    │◀────────────┘           │
│                         │  BACKEND    │                         │
│                         │   (Redis)   │  "Here's the result"   │
│                         └─────────────┘                         │
│                                                                 │
│   PROCESS 1             SERVICE              PROCESS 2          │
│   (your API)           (already running)     (separate!)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Each component is INDEPENDENT:**

```
┌─────────────────────────────────────────────────────────────────┐
│              INDEPENDENCE MATTERS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FastAPI crashes?                                               │
│  └─ Workers keep processing. Queued tasks are safe in Redis.    │
│                                                                 │
│  A worker crashes?                                              │
│  └─ The task it was running goes back to the queue.             │
│     Other workers (or the restarted one) pick it up.            │
│                                                                 │
│  Redis restarts?                                                │
│  └─ Problem — but Redis persists to disk (AOF/RDB),            │
│     so queued tasks can survive. (You configured this Week 10.) │
│                                                                 │
│  Each piece can crash, restart, or scale INDEPENDENTLY.         │
│  That's the power of distributed systems.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Message Flow (How an Order Becomes a Product)

**Follow one task through the entire lifecycle:**

```
┌─────────────────────────────────────────────────────────────────┐
│              LIFECYCLE OF A SINGLE TASK                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: CLIENT SENDS MESSAGE                                   │
│  ─────────────────────────────                                  │
│                                                                 │
│  Your FastAPI code:                                             │
│      send_email.delay("alice@test.com", "Welcome!")             │
│                                                                 │
│  What actually happens:                                         │
│  ┌──────────────────────────────────────────┐                   │
│  │ {                                        │                   │
│  │   "id": "a1b2c3d4-...",                  │                   │
│  │   "task": "tasks.send_email",            │ ← JSON message   │
│  │   "args": ["alice@test.com", "Welcome!"],│   pushed to      │
│  │   "kwargs": {},                          │   Redis queue     │
│  │   "retries": 0                           │                   │
│  │ }                                        │                   │
│  └──────────────────────────────────────────┘                   │
│         │                                                       │
│         ▼                                                       │
│  Step 2: BROKER HOLDS THE MESSAGE                               │
│  ────────────────────────────────                               │
│                                                                 │
│  Redis receives the JSON and stores it in a list (queue).       │
│  The message sits there until a worker picks it up.             │
│  (This is LPUSH/BRPOP — Redis lists you know from Week 10.)    │
│         │                                                       │
│         ▼                                                       │
│  Step 3: WORKER PICKS UP THE MESSAGE                            │
│  ────────────────────────────────────                           │
│                                                                 │
│  A Celery worker process is listening on the queue.             │
│  It pops the message, reads: "task = tasks.send_email"          │
│  It IMPORTS the function, passes in the arguments, and runs it. │
│         │                                                       │
│         ▼                                                       │
│  Step 4: WORKER EXECUTES THE FUNCTION                           │
│  ─────────────────────────────────────                          │
│                                                                 │
│  The worker calls: send_email("alice@test.com", "Welcome!")     │
│  This is a normal, synchronous Python function call.            │
│  The worker is BLOCKED while it runs (and that's fine).         │
│         │                                                       │
│         ▼                                                       │
│  Step 5: RESULT STORED IN BACKEND                               │
│  ─────────────────────────────────                              │
│                                                                 │
│  When the function returns, the result (or exception) is        │
│  serialized to JSON and stored in Redis (result backend).       │
│  The client can check this anytime using the task_id.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Redis as Broker (Your Week 10 Knowledge Pays Off)

**You already know Redis. Celery just uses it differently.**

```
┌─────────────────────────────────────────────────────────────────┐
│             REDIS: TWO ROLES IN CELERY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ROLE 1: MESSAGE BROKER (redis://localhost:6379/0)              │
│  ──────────────────────────────────────────────────             │
│  Uses Redis LISTS as queues.                                    │
│  Producer (FastAPI) does LPUSH → pushes task to list.           │
│  Consumer (Worker) does BRPOP → blocks until task available.    │
│                                                                 │
│  You know these data types from Week 10.                        │
│  Celery just automates the push/pop pattern.                    │
│                                                                 │
│                                                                 │
│  ROLE 2: RESULT BACKEND (redis://localhost:6379/1)              │
│  ─────────────────────────────────────────────────              │
│  Uses Redis STRINGS with TTL.                                   │
│  Key: "celery-task-meta-{task_id}"                              │
│  Value: JSON with status + result                               │
│  TTL: auto-expires (no stale results eating memory)             │
│                                                                 │
│  You've done this exact pattern — cache with TTL — in Week 10.  │
│                                                                 │
│                                                                 │
│  WHY TWO DIFFERENT DATABASES? (db 0 vs db 1)                   │
│  ─────────────────────────────────────────────                  │
│  Separation of concerns. Broker data is transient (consumed     │
│  and deleted). Result data is read-back (queried by client).    │
│  Different access patterns, different databases.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 The Sync Surprise

**This will feel wrong after 10 weeks of async. It's not.**

```python
# ❌ What you EXPECT to write (after 10 weeks of async):
@app.task
async def send_email(to: str, subject: str) -> dict:
    await smtp_client.send(to, subject)
    return {"status": "sent"}

# ✅ What you ACTUALLY write in Celery:
@app.task
def send_email(to: str, subject: str) -> dict:
    smtp_client.send(to, subject)  # Regular sync call
    return {"status": "sent"}
```

**"Wait — we've been writing async for months. Why is this sync?"**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE SYNC SURPRISE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Remember the Async lecture in Week 1?                          │
│                                                                 │
│  ASYNC (Event Loop):                                            │
│  ──────────────────                                             │
│  ONE process handles MANY tasks by switching between them.      │
│  When Task A waits for I/O, Task B runs. Cooperative.           │
│                                                                 │
│     ┌────────────────────────────────────┐                      │
│     │  FastAPI Process                   │                      │
│     │  ┌─────────────────────────────┐   │                      │
│     │  │      Event Loop             │   │                      │
│     │  │  Req A ──┐   ┌── Req C     │   │                      │
│     │  │  Req B ──┤   ├── Req D     │   │                      │
│     │  │  (juggling all at once)     │   │                      │
│     │  └─────────────────────────────┘   │                      │
│     └────────────────────────────────────┘                      │
│     One waiter serving many tables.                             │
│                                                                 │
│                                                                 │
│  CELERY (Worker Pool):                                          │
│  ─────────────────────                                          │
│  MANY processes each handle ONE task. True parallelism.         │
│  Each worker is DEDICATED to its current task.                  │
│                                                                 │
│     ┌──────────────────────────────────────────────┐            │
│     │  Celery Worker Pool                          │            │
│     │  ┌────────────┐ ┌────────────┐ ┌────────────┐│            │
│     │  │ Worker 1   │ │ Worker 2   │ │ Worker 3   ││            │
│     │  │ (email     │ │ (report    │ │ (idle,     ││            │
│     │  │  task)     │ │  task)     │ │  waiting)  ││            │
│     │  └────────────┘ └────────────┘ └────────────┘│            │
│     └──────────────────────────────────────────────┘            │
│     Multiple workers, each building one product.                │
│                                                                 │
│                                                                 │
│  WHY SYNC MAKES SENSE FOR WORKERS:                              │
│  ──────────────────────────────────                             │
│  A factory worker doesn't need to juggle. They pick up ONE      │
│  order, build it start to finish, then pick up the next.        │
│  You scale by hiring MORE workers, not by making one worker     │
│  juggle. That's the Celery model.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The decision framework you already know from Week 1:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CONCURRENCY MODEL           │  WHERE IT'S USED                 │
│  ────────────────────────    │  ─────────────────               │
│  async/await (event loop)    │  FastAPI — handle many requests  │
│  multiprocessing (prefork)   │  Celery — execute many tasks     │
│                                                                 │
│  Same goal (do more work), different mechanism.                 │
│  FastAPI juggles. Celery hires more hands.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Think back to Week 1: concurrency (one worker switching between tasks) vs parallelism (multiple workers, truly simultaneous). FastAPI is concurrency. Celery is parallelism. You need both — FastAPI to handle thousands of incoming requests, Celery to grind through thousands of background jobs."

---

# PART 3: YOUR FIRST CELERY TASK

## 3.1 Creating the Celery App

**The Celery app is the equivalent of `FastAPI()` — the central object everything attaches to.**

```python
# celery_app.py
from celery import Celery

celery = Celery(
    "worker",                                  # Name of the worker module
    broker="redis://localhost:6379/0",         # Where to SEND messages
    backend="redis://localhost:6379/1",        # Where to STORE results
)

# Configuration — sensible defaults for production
celery.conf.update(
    task_serializer="json",          # Serialize task arguments as JSON
    accept_content=["json"],         # Only accept JSON (security)
    result_serializer="json",        # Serialize results as JSON
    timezone="UTC",                  # All timestamps in UTC
    enable_utc=True,                 # Enforce UTC
    task_track_started=True,         # Report when a task starts running
    task_acks_late=True,             # Acknowledge AFTER completion (crash safety)
    worker_prefetch_multiplier=1,    # Don't grab extra tasks (fair distribution)
)
```

**What each configuration does:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CONFIGURATION EXPLAINED                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  task_serializer = "json"                                       │
│  ─────────────────────────                                      │
│  Task arguments get converted to JSON before going to Redis.    │
│  This means: only JSON-serializable data can be passed.         │
│  (strings, numbers, lists, dicts — NOT objects, NOT models)     │
│                                                                 │
│  task_acks_late = True          ← CRITICAL FOR CRASH SAFETY     │
│  ──────────────────────                                         │
│  Default behavior: worker acknowledges (removes from queue)     │
│  the task BEFORE running it. If it crashes mid-task → lost.     │
│                                                                 │
│  With acks_late: worker acknowledges AFTER completing.          │
│  If it crashes mid-task → task goes back to the queue.          │
│                                                                 │
│  This is why we said "work that survives." The order form       │
│  stays on the conveyor belt until the product is FINISHED.      │
│                                                                 │
│  task_track_started = True                                      │
│  ──────────────────────────                                     │
│  Adds a STARTED state to the lifecycle.                         │
│  Without it: PENDING → SUCCESS. You can't tell "waiting"        │
│  from "running." With it: PENDING → STARTED → SUCCESS.          │
│                                                                 │
│  worker_prefetch_multiplier = 1                                 │
│  ───────────────────────────────                                │
│  Each worker grabs only 1 task at a time from the queue.        │
│  Prevents one greedy worker from hoarding all the orders.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Project Structure (Two Processes, One Codebase)

**This is the mental shift: your project now runs as TWO separate processes.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   PROJECT STRUCTURE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  project/                                                       │
│  ├── app/                                                       │
│  │   ├── main.py              ← FastAPI app (Process 1)         │
│  │   ├── routers/                                               │
│  │   │   ├── users.py                                           │
│  │   │   └── reports.py                                         │
│  │   ├── services/                                              │
│  │   │   └── user_service.py                                    │
│  │   └── schemas/                                               │
│  │       └── user.py                                            │
│  ├── worker/                                                    │
│  │   ├── celery_app.py        ← Celery instance                 │
│  │   └── tasks/                                                 │
│  │       ├── __init__.py                                        │
│  │       ├── email_tasks.py   ← Task definitions                │
│  │       └── report_tasks.py  ← Task definitions                │
│  ├── docker-compose.yml       ← Runs everything together        │
│  └── requirements.txt                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Two terminals, two processes:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Terminal 1 (FastAPI):                                          │
│  $ uvicorn app.main:app --reload                                │
│                                                                 │
│  Terminal 2 (Celery Worker):                                    │
│  $ celery -A worker.celery_app worker --loglevel=info           │
│                                                                 │
│  (Redis is already running — you've had it since Week 10.)      │
│                                                                 │
│                                                                 │
│  They share the SAME CODEBASE but run as SEPARATE processes.    │
│  They communicate ONLY through Redis.                           │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                   │
│  │ Terminal 1│    │  Redis   │    │ Terminal 2│                  │
│  │ FastAPI   │───▶│ (broker) │───▶│ Celery   │                  │
│  │ :8000     │    │ :6379    │    │ Worker   │                  │
│  └──────────┘    └──────────┘    └──────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Defining Tasks with @app.task

**A task is a regular Python function with a decorator. You've used decorators since Week 1.**

```python
# worker/tasks/email_tasks.py
import smtplib
from email.message import EmailMessage
from worker.celery_app import celery

@celery.task
def send_welcome_email(user_email: str, username: str) -> dict:
    """Send a welcome email to a newly registered user.
    
    This is a regular sync function. NOT async.
    The worker runs it in its own process.
    """
    msg = EmailMessage()
    msg["Subject"] = f"Welcome, {username}!"
    msg["From"] = "noreply@ourapp.com"
    msg["To"] = user_email
    msg.set_content(f"Hello {username}, welcome to our platform!")

    # This is a blocking call — and that's perfectly fine.
    # This worker process has nothing else to do right now.
    with smtplib.SMTP("smtp.mailserver.com", 587) as server:
        server.starttls()
        server.login("noreply@ourapp.com", "password")
        server.send_message(msg)

    return {"status": "sent", "to": user_email}
```

**What `@celery.task` actually does:**

```
┌─────────────────────────────────────────────────────────────────┐
│             WHAT THE DECORATOR DOES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You learned in Week 1 that decorators wrap functions.          │
│  @celery.task wraps your function with task-sending machinery.  │
│                                                                 │
│  BEFORE decorator:                                              │
│  ─────────────────                                              │
│  send_welcome_email is just a function.                         │
│  You can only call it directly: send_welcome_email("a@b.com")  │
│                                                                 │
│  AFTER decorator:                                               │
│  ────────────────                                               │
│  send_welcome_email is now a Task object with extra methods:    │
│                                                                 │
│  send_welcome_email("a@b.com", "alice")                         │
│  └─ Still works! Runs the function directly (useful for tests)  │
│                                                                 │
│  send_welcome_email.delay("a@b.com", "alice")                   │
│  └─ NEW! Sends a message to Redis. Returns immediately.         │
│                                                                 │
│  send_welcome_email.apply_async(args=["a@b.com", "alice"])      │
│  └─ NEW! Same as delay but with more options.                   │
│                                                                 │
│  send_welcome_email.name                                        │
│  └─ "worker.tasks.email_tasks.send_welcome_email"               │
│     (The full import path — this is how the worker finds it.)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Another task example — a heavy report:**

```python
# worker/tasks/report_tasks.py
import csv
import io
from datetime import datetime
from worker.celery_app import celery

@celery.task
def generate_monthly_report(org_id: int, month: int, year: int) -> dict:
    """Generate a CSV report for an organization.
    
    This might take 30 seconds — way too long for an API response.
    Perfect candidate for a background task.
    """
    # Connect to DB (sync driver — NOT async SQLAlchemy)
    # Workers have their OWN database connections.
    from sqlalchemy import create_engine, text
    engine = create_engine("postgresql://user:pass@localhost/mydb")
    
    with engine.connect() as conn:
        rows = conn.execute(
            text("""
                SELECT task_name, status, completed_at 
                FROM tasks 
                WHERE org_id = :org_id 
                  AND EXTRACT(MONTH FROM created_at) = :month
                  AND EXTRACT(YEAR FROM created_at) = :year
            """),
            {"org_id": org_id, "month": month, "year": year}
        ).fetchall()
    
    # Build CSV in memory
    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["Task", "Status", "Completed At"])
    for row in rows:
        writer.writerow(row)
    
    csv_content = output.getvalue()
    
    # In reality: upload to S3, save path to DB, etc.
    # For now, return metadata
    return {
        "status": "completed",
        "org_id": org_id,
        "period": f"{year}-{month:02d}",
        "row_count": len(rows),
        "generated_at": datetime.utcnow().isoformat(),
    }
```

> "Notice: we're using a **sync** database driver (`create_engine`, not `create_async_engine`). The worker is its own process — it doesn't share FastAPI's async engine or session. It creates its own connections. This is a completely separate world."

---

## 3.4 Running the Worker

**Start the worker and watch it come alive:**

```bash
$ celery -A worker.celery_app worker --loglevel=info
```

**What you'll see:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   WORKER STARTUP                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  -------------- celery@your-machine v5.4.0 (opalescent)        │
│  --- ***** -----                                                │
│  -- ******* ---- Linux-5.15.0-x86_64-with-glib2.35             │
│  - *** --- * ---                                                │
│  - ** ---------- [config]                                       │
│  - ** ---------- .> app:         worker:0x7f4a...               │
│  - ** ---------- .> transport:   redis://localhost:6379/0  ← ①  │
│  - ** ---------- .> results:     redis://localhost:6379/1  ← ②  │
│  - *** --- * --- .> concurrency: 4 (prefork)              ← ③  │
│  -- ******* ---- .> task events: OFF                            │
│  --- ***** -----                                                │
│   -------------- [queues]                                       │
│                  .> celery  exchange=celery(direct) key=celery   │
│                                                                 │
│  [tasks]                                                        │
│    . worker.tasks.email_tasks.send_welcome_email          ← ④   │
│    . worker.tasks.report_tasks.generate_monthly_report          │
│                                                                 │
│  [... INFO/MainProcess] Connected to redis://localhost:6379/0   │
│  [... INFO/MainProcess] celery@your-machine ready.        ← ⑤  │
│                                                                 │
│  ① Broker: where it listens for new tasks                       │
│  ② Backend: where it stores results                             │
│  ③ Concurrency: 4 worker processes (one per CPU core)           │
│  ④ Registered tasks: the functions this worker knows about      │
│  ⑤ Ready: now waiting for messages on the queue                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**If your task doesn't appear in [tasks], the worker can't run it:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  COMMON ISSUE: Task not registered                              │
│  ─────────────────────────────────                              │
│                                                                 │
│  If you don't see your task in the [tasks] list:                │
│                                                                 │
│  1. Check imports — Celery must discover your task modules      │
│                                                                 │
│  # In celery_app.py, tell Celery where to find tasks:           │
│  celery.autodiscover_tasks(["worker.tasks"])                    │
│                                                                 │
│  2. Or import tasks explicitly:                                 │
│  import worker.tasks.email_tasks    # Force import              │
│  import worker.tasks.report_tasks                               │
│                                                                 │
│  The worker can only run functions it has IMPORTED.              │
│  No import → not in [tasks] list → task will be UNREGISTERED.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Updated celery_app.py with autodiscovery:**

```python
# worker/celery_app.py
from celery import Celery

celery = Celery(
    "worker",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1",
)

celery.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
    task_track_started=True,
    task_acks_late=True,
    worker_prefetch_multiplier=1,
)

# Automatically find all @celery.task decorated functions
# in the listed packages
celery.autodiscover_tasks(["worker.tasks"])
```

---

## 3.5 Calling Tasks: .delay() and .apply_async()

**Two ways to send work to the factory. Both put an order on the conveyor belt.**

### .delay() — The Simple Way

```python
# app/routers/users.py
from fastapi import APIRouter, Depends
from worker.tasks.email_tasks import send_welcome_email

router = APIRouter()

@router.post("/users/", status_code=201)
async def create_user(user_data: UserCreate, db: AsyncSession = Depends(get_db)):
    # 1. Create user in database (fast, < 50ms)
    new_user = await user_service.create_user(db, user_data)
    
    # 2. Send welcome email in background (slow, 2-5 seconds)
    #    .delay() sends the message and returns IMMEDIATELY
    task_result = send_welcome_email.delay(new_user.email, new_user.username)
    
    # 3. Return response to the client NOW — don't wait for email
    return {
        "id": new_user.id,
        "email": new_user.email,
        "email_task_id": task_result.id,  # Client can check status later
    }
```

**What happens at `.delay()`:**

```
┌─────────────────────────────────────────────────────────────────┐
│            WHAT .delay() DOES                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  send_welcome_email.delay("alice@test.com", "alice")            │
│                           │                                     │
│                           ▼                                     │
│  1. Serializes arguments to JSON:                               │
│     {"args": ["alice@test.com", "alice"], "kwargs": {}}         │
│                           │                                     │
│                           ▼                                     │
│  2. Generates a unique task_id:                                 │
│     "f47ac10b-58cc-4372-a567-0e02b2c3d479"                      │
│                           │                                     │
│                           ▼                                     │
│  3. Pushes message to Redis queue:                              │
│     LPUSH "celery" <full message JSON>                          │
│                           │                                     │
│                           ▼                                     │
│  4. Returns immediately with an AsyncResult object:             │
│     AsyncResult(id="f47ac10b-...")                               │
│                                                                 │
│  TIME SPENT: ~1-5 milliseconds (just a Redis write)             │
│  The email hasn't been sent yet. A worker will do that later.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### .apply_async() — The Powerful Way

```python
# .delay(arg1, arg2) is shorthand for:
# .apply_async(args=[arg1, arg2])

# But apply_async gives you control:

# 1. Delay execution by 60 seconds (send email in 1 minute)
send_welcome_email.apply_async(
    args=["alice@test.com", "alice"],
    countdown=60,
)

# 2. Schedule for a specific time
from datetime import datetime, timedelta

send_welcome_email.apply_async(
    args=["alice@test.com", "alice"],
    eta=datetime.utcnow() + timedelta(hours=1),
)

# 3. Set a hard expiration (don't run if too old)
send_welcome_email.apply_async(
    args=["alice@test.com", "alice"],
    expires=300,  # Expire after 5 minutes
)

# 4. Route to a specific queue
send_welcome_email.apply_async(
    args=["alice@test.com", "alice"],
    queue="high_priority",
)

# 5. Custom task_id (useful for idempotency — more in Part 6)
send_welcome_email.apply_async(
    args=["alice@test.com", "alice"],
    task_id=f"welcome-email-{user_id}",
)
```

**When to use which:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  .delay()                │  .apply_async()                      │
│  ────────                │  ──────────────                      │
│  Simple. Just run it.    │  Need control over WHEN/WHERE/HOW.   │
│                          │                                      │
│  send_email.delay(       │  send_email.apply_async(             │
│      "a@b.com", "hi"    │      args=["a@b.com", "hi"],         │
│  )                       │      countdown=60,                   │
│                          │      queue="emails",                 │
│                          │      expires=600,                    │
│                          │  )                                   │
│                          │                                      │
│  Use for: fire and       │  Use for: delayed, scheduled,        │
│  forget, run ASAP.       │  prioritized, or expiring tasks.     │
│                          │                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.6 Serialization: Tasks Are Messages, Not Function Calls

**This is the most common source of bugs for beginners. Understand it now.**

When you call `.delay()`, you are NOT calling the function. You are putting a **message in a queue**. That message is **JSON**. The worker in a completely different process reads that JSON, imports the function, and calls it with the deserialized arguments.

```
┌─────────────────────────────────────────────────────────────────┐
│         TASKS ARE MESSAGES, NOT CALLS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What you THINK happens:                                        │
│  ─────────────────────────                                      │
│  FastAPI calls send_email("alice@test.com", "alice")            │
│  in another thread or something.                                │
│                                                                 │
│  What ACTUALLY happens:                                         │
│  ──────────────────────────                                     │
│  FastAPI writes JSON → Redis → Worker reads JSON → Worker       │
│  imports function → Worker calls function with parsed args.     │
│                                                                 │
│  The arguments travel as JSON through Redis.                    │
│  EVERYTHING must survive JSON serialization.                    │
│                                                                 │
│  ✅ CAN pass as arguments:                                      │
│  ├─ str, int, float, bool, None                                 │
│  ├─ list, dict (with JSON-safe contents)                        │
│  ├─ Nested combinations of the above                            │
│  └─ Anything json.dumps() can handle                            │
│                                                                 │
│  ❌ CANNOT pass as arguments:                                   │
│  ├─ SQLAlchemy model instances (User object)                    │
│  ├─ Pydantic model instances (UserSchema object)                │
│  ├─ Database sessions / connections                             │
│  ├─ File objects                                                │
│  ├─ FastAPI Request objects                                     │
│  ├─ datetime objects (pass ISO string instead)                  │
│  └─ Any custom class instance                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The pattern: pass IDs, not objects:**

```python
# ❌ WRONG: Passing a SQLAlchemy model
@router.post("/users/")
async def create_user(user_data: UserCreate, db: AsyncSession = Depends(get_db)):
    user = await user_service.create_user(db, user_data)
    send_welcome_email.delay(user)  # TypeError! Can't serialize User object
    return user

# ❌ WRONG: Passing a Pydantic model
@router.post("/users/")
async def create_user(user_data: UserCreate, db: AsyncSession = Depends(get_db)):
    user = await user_service.create_user(db, user_data)
    send_welcome_email.delay(user_data)  # Can't serialize Pydantic model
    return user

# ✅ CORRECT: Pass primitive values the task needs
@router.post("/users/")
async def create_user(user_data: UserCreate, db: AsyncSession = Depends(get_db)):
    user = await user_service.create_user(db, user_data)
    send_welcome_email.delay(user.email, user.username)  # Strings — JSON safe
    return user

# ✅ ALSO CORRECT: Pass the ID, let the task fetch what it needs
@router.post("/reports/")
async def request_report(org_id: int, db: AsyncSession = Depends(get_db)):
    task = generate_monthly_report.delay(org_id, 1, 2025)  # Ints — JSON safe
    return {"task_id": task.id}

# The task fetches data itself using the ID:
@celery.task
def generate_monthly_report(org_id: int, month: int, year: int) -> dict:
    # Worker creates its OWN database connection
    # and fetches the data it needs using the ID
    ...
```

**Think of it this way:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Factory analogy:                                               │
│                                                                 │
│  The order form says: "Build a chair for Customer #472"         │
│  The order form does NOT contain the customer standing there.   │
│  The order form does NOT contain the wood and nails.            │
│                                                                 │
│  It contains: instructions and identifiers.                     │
│  The factory worker gets their OWN wood and nails.              │
│                                                                 │
│  Same with Celery:                                              │
│  Pass IDs and simple values.                                    │
│  The worker fetches its own data, opens its own connections.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Also: NO FastAPI context inside tasks.**

```python
# ❌ WRONG: Using FastAPI's Depends() inside a task
@celery.task
def bad_task(user_id: int):
    db = Depends(get_db)  # This is meaningless here — no FastAPI request
    ...

# ❌ WRONG: Using the same async session from FastAPI
@celery.task
def bad_task(user_id: int):
    session = get_async_session()  # Worker doesn't have FastAPI's event loop
    await session.execute(...)     # Can't await in sync function anyway
    ...

# ✅ CORRECT: Task creates its own resources
@celery.task
def good_task(user_id: int) -> dict:
    # Sync engine — the worker's own connection
    engine = create_engine("postgresql://user:pass@localhost/mydb")
    with Session(engine) as session:
        user = session.query(User).filter_by(id=user_id).first()
        if not user:
            return {"error": "user_not_found"}
        # ... do work ...
        return {"status": "done"}
```

> "The worker is a completely separate process. It has no idea FastAPI exists. It doesn't share memory, sessions, or context. If the task needs something, it gets it itself — its own database connection, its own HTTP client, its own everything."

---

# PART 4: TRACKING WORK — RESULTS & STATES

## 4.1 AsyncResult — Your Tracking Number

**When you send a task, you get back a tracking number (AsyncResult). You can check it anytime.**

```python
from worker.tasks.email_tasks import send_welcome_email

# Send the task
result = send_welcome_email.delay("alice@test.com", "alice")

# result is an AsyncResult object
print(result.id)       # "f47ac10b-58cc-4372-a567-0e02b2c3d479"
print(result.status)   # "PENDING" (just queued, not started yet)
print(result.ready())  # False (not done yet)
```

**You can also reconstruct an AsyncResult from just the task_id:**

```python
from celery.result import AsyncResult
from worker.celery_app import celery

# If you saved the task_id earlier (e.g., in a database or returned to client)
task_id = "f47ac10b-58cc-4372-a567-0e02b2c3d479"

# Reconstruct the result object
result = AsyncResult(task_id, app=celery)
print(result.status)   # Check current state
print(result.result)   # Get the return value (if done)
```

> "This is like entering a tracking number on a shipping website. You don't need the original receipt — just the number."

---

## 4.2 The Task State Machine

**Every task moves through a defined set of states:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  TASK STATE MACHINE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      ┌─────────┐                                │
│                      │ PENDING │  Task is in the queue.          │
│                      │         │  No worker has picked it up.    │
│                      └────┬────┘                                │
│                           │                                     │
│                    Worker picks it up                            │
│                           │                                     │
│                           ▼                                     │
│                      ┌─────────┐                                │
│                      │ STARTED │  Worker is executing the        │
│                      │         │  function right now.            │
│                      └────┬────┘                                │
│                           │                                     │
│              ┌────────────┼────────────┐                        │
│              │            │            │                         │
│         Returned      Exception    self.retry()                 │
│         a value       raised       called                       │
│              │            │            │                         │
│              ▼            ▼            ▼                         │
│         ┌─────────┐ ┌─────────┐ ┌─────────┐                    │
│         │ SUCCESS │ │ FAILURE │ │  RETRY  │                     │
│         │         │ │         │ │         │                     │
│         │ .result │ │ .result │ │ Goes    │                     │
│         │ = return│ │ = the   │ │ back to │──▶ back to PENDING  │
│         │   value │ │ Exception│ │ queue   │                    │
│         └─────────┘ └─────────┘ └─────────┘                    │
│                                                                 │
│                                                                 │
│  Special state:                                                 │
│  ┌─────────┐                                                    │
│  │ REVOKED │  Task was cancelled before/during execution.       │
│  │         │  (task.revoke() was called)                        │
│  └─────────┘                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Inspecting each state:**

```python
from celery.result import AsyncResult
from worker.celery_app import celery

def check_task(task_id: str) -> dict:
    """Check the current state of any task."""
    result = AsyncResult(task_id, app=celery)
    
    response = {
        "task_id": task_id,
        "status": result.status,
        "ready": result.ready(),       # True if SUCCESS or FAILURE
        "successful": result.successful() if result.ready() else None,
    }
    
    if result.status == "SUCCESS":
        response["result"] = result.result       # The return value
    elif result.status == "FAILURE":
        response["error"] = str(result.result)   # The exception
        response["traceback"] = result.traceback  # Full traceback string
    elif result.status == "RETRY":
        response["retry_info"] = str(result.info) # Why it's retrying

    return response
```

---

## 4.3 Checking Status from FastAPI

**Common pattern: client submits work, polls for status.**

```python
# app/routers/reports.py
from fastapi import APIRouter
from celery.result import AsyncResult
from worker.celery_app import celery
from worker.tasks.report_tasks import generate_monthly_report

router = APIRouter(prefix="/reports", tags=["reports"])

@router.post("/", status_code=202)  # 202 Accepted — work is queued, not done
async def request_report(org_id: int, month: int, year: int):
    """Request a report generation. Returns immediately."""
    task = generate_monthly_report.delay(org_id, month, year)
    
    return {
        "task_id": task.id,
        "status": "accepted",
        "check_url": f"/reports/status/{task.id}",
    }

@router.get("/status/{task_id}")
async def get_report_status(task_id: str):
    """Poll this endpoint to check if the report is ready."""
    result = AsyncResult(task_id, app=celery)

    if result.status == "PENDING":
        return {"task_id": task_id, "status": "pending", "message": "In queue"}
    elif result.status == "STARTED":
        return {"task_id": task_id, "status": "processing"}
    elif result.status == "SUCCESS":
        return {
            "task_id": task_id,
            "status": "completed",
            "result": result.result,
        }
    elif result.status == "FAILURE":
        return {
            "task_id": task_id,
            "status": "failed",
            "error": str(result.result),
        }
    elif result.status == "RETRY":
        return {"task_id": task_id, "status": "retrying"}
    
    return {"task_id": task_id, "status": result.status}
```

**The HTTP flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│               CLIENT POLLING PATTERN                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client                  FastAPI                 Celery Worker   │
│    │                       │                         │          │
│    │ POST /reports/        │                         │          │
│    │──────────────────────▶│                         │          │
│    │                       │── delay() ──▶ Redis ──▶ │          │
│    │  202 Accepted         │                         │          │
│    │  {task_id: "abc"}     │                         │          │
│    │◀──────────────────────│                         │          │
│    │                       │                    (working...)     │
│    │ GET /reports/status/abc                         │          │
│    │──────────────────────▶│                         │          │
│    │  {"status": "pending"}│                         │          │
│    │◀──────────────────────│                         │          │
│    │                       │                    (still working)  │
│    │ GET /reports/status/abc                         │          │
│    │──────────────────────▶│                         │          │
│    │  {"status":"processing"}                        │          │
│    │◀──────────────────────│                         │          │
│    │                       │                    (done!)          │
│    │ GET /reports/status/abc                         │          │
│    │──────────────────────▶│                         │          │
│    │  {"status":"completed",                         │          │
│    │   "result": {...}}    │                         │          │
│    │◀──────────────────────│                         │          │
│    │                       │                         │          │
│                                                                 │
│  202 Accepted → "I received your order."                        │
│  Status endpoint → "Check the tracking page."                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice the 202 status code — not 200 or 201. 202 means 'Accepted — I've received your request and I'll process it, but it's not done yet.' You learned status code semantics in Week 3. This is where it matters."

---

## 4.4 When You Don't Need Results

**Not every task needs a return value. Fire-and-forget is often enough.**

```python
# If you never check the result, don't store it.
# It wastes Redis memory.

@celery.task(ignore_result=True)  # ← Don't store result in backend
def log_user_activity(user_id: int, action: str, metadata: dict) -> None:
    """Write to audit log. We don't need to check the result."""
    engine = create_engine(DATABASE_URL)
    with Session(engine) as session:
        session.add(AuditLog(
            user_id=user_id,
            action=action,
            metadata=metadata,
        ))
        session.commit()

# When calling — you can still get the task_id, but .result will be None
log_user_activity.delay(42, "login", {"ip": "1.2.3.4"})
```

**When to ignore results:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  KEEP RESULTS (default):           IGNORE RESULTS:              │
│  ───────────────────────           ────────────────             │
│  • Report generation               • Logging / auditing         │
│    (client needs the output)        (write-and-forget)          │
│  • File processing                 • Sending notifications      │
│    (need download URL)              (no response expected)       │
│  • Data import                     • Cache warming              │
│    (need success/row count)         (just populate cache)        │
│  • Any task where the caller       • Cleanup jobs               │
│    checks status or result          (delete old files)           │
│                                                                 │
│  ignore_result=True saves Redis memory and reduces overhead.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: WHEN THINGS GO WRONG — RETRIES

## 5.1 What Happens When a Task Fails?

**By default: the exception is captured, the task is marked FAILURE, and life moves on.**

```python
@celery.task
def fragile_task(url: str) -> dict:
    import httpx
    response = httpx.get(url, timeout=5)  # Might timeout!
    response.raise_for_status()           # Might be 500!
    return response.json()

# If this task raises an exception:
# 1. The worker catches it
# 2. Task state → FAILURE
# 3. result.result = the Exception object
# 4. result.traceback = full traceback string
# 5. The task is NOT retried (by default)
# 6. Worker moves on to the next task
```

**The worker log shows:**

```
[... ERROR/ForkPoolWorker-1] Task worker.tasks.fragile_task[abc-123] 
raised unexpected: ConnectTimeout('Connection timed out')
Traceback (most recent call last):
  File ".../worker/tasks/example.py", line 4, in fragile_task
    response = httpx.get(url, timeout=5)
  ...
httpx.ConnectTimeout: Connection timed out
```

> "The task fails. The worker logs the error. Nobody retries it. For some tasks, that's fine — if a cache warm fails, whatever. But for sending an invoice? You need retries."

---

## 5.2 bind=True — Giving the Task Self-Awareness

**Before we can retry, the task needs access to its own task object. That's what `bind=True` does.**

```python
# Without bind: the function is just a function
@celery.task
def simple_task(x: int, y: int) -> int:
    return x + y

# With bind: the function receives 'self' — the Task instance
@celery.task(bind=True)
def aware_task(self, x: int, y: int) -> int:
    print(f"Task ID: {self.request.id}")
    print(f"Retry count: {self.request.retries}")
    return x + y
```

**What `self` gives you access to:**

```
┌─────────────────────────────────────────────────────────────────┐
│             WHAT bind=True UNLOCKS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  self.request.id          → The unique task ID                  │
│  self.request.retries     → How many times this task retried    │
│  self.request.args        → The positional arguments            │
│  self.request.kwargs      → The keyword arguments               │
│  self.request.delivery_info → Queue info                        │
│                                                                 │
│  self.retry()             → Re-queue this task (retry it)       │
│  self.max_retries         → Max retry count from config         │
│  self.name                → Full task name string               │
│                                                                 │
│  Think of bind=True as giving the factory worker an employee    │
│  badge. Now they know their name, can check how many times      │
│  they've tried this order, and can officially request a redo.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice that `self` is the **first** parameter, before your actual arguments. When calling with `.delay()`, you still pass only your arguments — Celery injects `self` automatically. This is the same pattern as instance methods on classes."

```python
# Calling a bound task — same as before, 'self' is invisible to callers
aware_task.delay(3, 5)        # Just pass x and y — self is automatic
```

---

## 5.3 Manual Retry with self.retry()

**You decide WHEN to retry and with what delay.**

```python
@celery.task(bind=True, max_retries=3)
def send_notification(self, user_id: int, message: str) -> dict:
    """Send a push notification. Retry on transient failures."""
    import httpx
    
    try:
        response = httpx.post(
            "https://push-service.example.com/send",
            json={"user_id": user_id, "message": message},
            timeout=10,
        )
        response.raise_for_status()
        return {"status": "sent", "user_id": user_id}
    
    except httpx.TimeoutException:
        # Transient error — retry after a delay
        raise self.retry(
            countdown=60,  # Wait 60 seconds before retrying
            exc=httpx.TimeoutException("Push service timed out"),
        )
    
    except httpx.HTTPStatusError as exc:
        if exc.response.status_code == 503:
            # Service temporarily unavailable — retry
            raise self.retry(countdown=120, exc=exc)
        
        elif exc.response.status_code == 400:
            # Bad request — OUR fault, retrying won't help
            return {"status": "failed", "reason": "invalid_request"}
        
        else:
            # Unknown error — let it fail
            raise
```

**Critical pattern: raise self.retry(), do NOT call and ignore it:**

```python
# ❌ WRONG: calling self.retry() without raise
except TimeoutError:
    self.retry(countdown=60)  # This works but is confusing
    # Code below this STILL RUNS in some edge cases!
    print("This might execute!")

# ✅ CORRECT: raise self.retry()
except TimeoutError:
    raise self.retry(countdown=60)  # Raises Retry exception — clear exit
    # Code below this NEVER runs
```

**What self.retry() does internally:**

```
┌─────────────────────────────────────────────────────────────────┐
│               self.retry() INTERNALS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  raise self.retry(countdown=60, exc=original_exception)         │
│                           │                                     │
│                           ▼                                     │
│  1. Checks: retries < max_retries?                              │
│     ├─ YES: Continue to step 2                                  │
│     └─ NO: Raise MaxRetriesExceededError                        │
│                           │                                     │
│                           ▼                                     │
│  2. Sends a NEW message to the broker                           │
│     with same args/kwargs but incremented retry count           │
│     and ETA = now + countdown                                   │
│                           │                                     │
│                           ▼                                     │
│  3. Raises celery.exceptions.Retry                              │
│     (this stops the current execution cleanly)                  │
│                           │                                     │
│                           ▼                                     │
│  4. Task state → RETRY                                          │
│     The new message sits in the queue until countdown expires.   │
│     Then a worker picks it up and runs the function AGAIN.      │
│                                                                 │
│  It's like the factory worker saying: "This piece is defective. │
│  Put the order back on the belt. I'll try again in an hour."    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Retry with different arguments (advanced):**

```python
@celery.task(bind=True, max_retries=3)
def process_with_fallback(self, url: str, use_backup: bool = False) -> dict:
    try:
        response = httpx.get(url, timeout=10)
        return response.json()
    except httpx.TimeoutException:
        if not use_backup:
            # Retry with the backup URL flag
            raise self.retry(
                countdown=30,
                kwargs={"url": url, "use_backup": True},
            )
        else:
            # Already tried backup — give up
            raise
```

---

## 5.4 Declarative Retry with autoretry_for

**For simpler cases, you don't need the try/except at all. Declare which exceptions should trigger a retry.**

```python
@celery.task(
    bind=True,
    autoretry_for=(ConnectionError, TimeoutError, IOError),
    max_retries=5,
    retry_backoff=True,       # Exponential backoff (next section)
    retry_jitter=True,        # Add randomness to prevent thundering herd
)
def sync_external_data(self, source_id: int) -> dict:
    """Fetch data from external source and store locally.
    
    If ConnectionError, TimeoutError, or IOError is raised,
    Celery automatically retries — no try/except needed.
    """
    import httpx
    
    response = httpx.get(f"https://api.source.com/data/{source_id}", timeout=15)
    response.raise_for_status()
    data = response.json()
    
    engine = create_engine(DATABASE_URL)
    with Session(engine) as session:
        session.merge(ExternalData(source_id=source_id, payload=data))
        session.commit()
    
    return {"source_id": source_id, "records": len(data)}
```

**Comparison — same behavior, two styles:**

```python
# MANUAL RETRY (verbose but full control):
@celery.task(bind=True, max_retries=5)
def fetch_data_manual(self, url: str) -> dict:
    try:
        response = httpx.get(url, timeout=10)
        return response.json()
    except (ConnectionError, TimeoutError) as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)

# DECLARATIVE RETRY (concise, less control):
@celery.task(
    autoretry_for=(ConnectionError, TimeoutError),
    max_retries=5,
    retry_backoff=True,
)
def fetch_data_declarative(url: str) -> dict:
    response = httpx.get(url, timeout=10)
    return response.json()

# Same result. Choose based on whether you need custom logic per exception.
```

**When to use which:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  autoretry_for:                  │  self.retry():               │
│  ─────────────                   │  ─────────────               │
│  Same handling for all listed    │  Different handling per       │
│  exceptions. Retry all of them   │  exception type. Custom       │
│  with the same strategy.         │  countdown. Different args.   │
│                                  │                               │
│  Use for: network errors,        │  Use for: partial failures,   │
│  timeouts, external services     │  fallback logic, conditional  │
│  that are reliably transient.    │  retry decisions.             │
│                                  │                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Exponential Backoff (Week 8 Callback)

**You implemented this with tenacity in Week 8 for HTTP clients. Celery has it built in.**

```
┌─────────────────────────────────────────────────────────────────┐
│             EXPONENTIAL BACKOFF IN CELERY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  retry_backoff=True means:                                      │
│                                                                 │
│  Attempt 1 fails → wait 1 second                                │
│  Attempt 2 fails → wait 2 seconds                               │
│  Attempt 3 fails → wait 4 seconds                               │
│  Attempt 4 fails → wait 8 seconds                               │
│  Attempt 5 fails → wait 16 seconds                              │
│  ...                                                            │
│                                                                 │
│  Formula: delay = backoff_base ** retry_number                  │
│  Default backoff_base = 2 (doubles each time)                   │
│                                                                 │
│  Or specify a fixed base:                                       │
│  retry_backoff=3 → wait 3, 9, 27, 81, 243...                   │
│                                                                 │
│                                                                 │
│  retry_backoff_max=600 → Cap the delay at 600s (10 minutes)     │
│  Without a cap, delay grows forever. Don't wait 3 hours.        │
│                                                                 │
│                                                                 │
│  retry_jitter=True → Adds random ±variation to each delay       │
│  WHY? If 100 tasks all fail at the same time (e.g., DB is       │
│  down), they'd all retry at the same time → same overload.      │
│  Jitter spreads them out. Same reason as Week 8 tenacity.       │
│                                                                 │
│                                                                 │
│  Attempt timeline with jitter (approximate):                    │
│  ──────────────────────────────────────────                     │
│  0s      1.3s    3.7s      11.2s           42.1s                │
│  │ fail  │ fail  │ fail    │ fail          │ success!           │
│  ├───────┤───────┤─────────┤───────────────┤                    │
│  try 1   try 2   try 3    try 4           try 5                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The production-ready pattern:**

```python
@celery.task(
    bind=True,
    autoretry_for=(ConnectionError, TimeoutError),
    max_retries=5,
    retry_backoff=2,          # Base: 2 seconds
    retry_backoff_max=600,    # Cap: 10 minutes
    retry_jitter=True,        # Random spread
)
def resilient_task(self, data_id: int) -> dict:
    """A task that handles transient failures gracefully."""
    # ... do work ...
    pass
```

> "Same exponential backoff pattern from Week 8 with tenacity. There it was for HTTP client retries. Here it's for background task retries. Same principle: give the failing service time to recover, with increasing patience."

---

## 5.6 After Max Retries: Accepting Failure

**What happens when all retries are exhausted?**

```python
@celery.task(bind=True, max_retries=3)
def send_critical_email(self, to: str, subject: str, body: str) -> dict:
    try:
        smtp_send(to, subject, body)
        return {"status": "sent"}
    except SMTPConnectionError as exc:
        try:
            raise self.retry(exc=exc, countdown=60)
        except self.MaxRetriesExceededError:
            # All retries exhausted. This is our LAST CHANCE to handle it.
            # Log it, alert someone, save to a dead letter table.
            
            log_failed_email(to, subject, body, error=str(exc))
            notify_ops_team(f"Email to {to} failed after {self.max_retries} retries")
            
            # Return failure info instead of raising
            return {
                "status": "permanently_failed",
                "to": to,
                "error": str(exc),
                "retries_attempted": self.request.retries,
            }
```

**Or use a callback for failure handling:**

```python
def handle_task_failure(self, exc, task_id, args, kwargs, einfo):
    """Called when a task fails permanently (after all retries)."""
    # self = task instance, exc = exception, einfo = traceback
    print(f"Task {task_id} permanently failed: {exc}")
    # Log to database, send alert, etc.

@celery.task(
    bind=True,
    autoretry_for=(ConnectionError,),
    max_retries=5,
    retry_backoff=True,
    on_failure=handle_task_failure,  # ← Called after final failure
)
def task_with_failure_hook(self, item_id: int) -> dict:
    # ... do work ...
    pass
```

**The decision: retry or don't?**

```
┌─────────────────────────────────────────────────────────────────┐
│             RETRY OR DON'T? DECISION GUIDE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ RETRY (transient — might work next time):                   │
│  ├─ ConnectionError         (network blip)                      │
│  ├─ TimeoutError            (slow response)                     │
│  ├─ HTTP 502, 503, 504      (server temporarily down)           │
│  ├─ Database connection lost (pool exhausted, restart)           │
│  └─ Rate limit hit (429)    (wait and try again)                │
│                                                                 │
│  ❌ DO NOT RETRY (permanent — will NEVER work):                 │
│  ├─ ValueError              (bad data, won't change)            │
│  ├─ HTTP 400, 401, 403, 404 (client error, our fault)           │
│  ├─ ValidationError         (invalid input)                     │
│  ├─ IntegrityError          (duplicate key, constraint)         │
│  └─ PermissionError         (wrong credentials)                 │
│                                                                 │
│  Retrying a permanent error wastes resources and delays the     │
│  inevitable. Fail fast on errors that won't fix themselves.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 6: THE NON-NEGOTIABLE — IDEMPOTENCY

## 6.1 At-Least-Once Delivery (The Uncomfortable Truth)

**Remember idempotency from Week 4's API design lecture? You learned that PUT and DELETE are idempotent — calling them multiple times has the same effect as calling them once. Now idempotency becomes critical, because Celery delivers tasks AT LEAST ONCE.**

```
┌─────────────────────────────────────────────────────────────────┐
│           DELIVERY GUARANTEES                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Three possible guarantees:                                     │
│                                                                 │
│  AT-MOST-ONCE:   Message delivered 0 or 1 time.                 │
│                  Might lose messages. Never duplicates.          │
│                                                                 │
│  AT-LEAST-ONCE:  Message delivered 1 or MORE times.  ← CELERY  │
│                  Never loses messages. Might duplicate.          │
│                                                                 │
│  EXACTLY-ONCE:   Message delivered exactly 1 time.              │
│                  The holy grail. Extremely hard in distributed   │
│                  systems. Celery does NOT guarantee this.        │
│                                                                 │
│                                                                 │
│  Celery with acks_late=True gives you AT-LEAST-ONCE.            │
│  Your task WILL run. But it might run MORE THAN ONCE.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 How Duplicates Happen

**This isn't theoretical. It WILL happen in production.**

```
┌─────────────────────────────────────────────────────────────────┐
│              HOW TASKS GET DUPLICATED                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO 1: Worker crashes after completing, before acking     │
│  ─────────────────────────────────────────────────────────      │
│                                                                 │
│  Worker picks up task → Sends email → CRASH before acknowledge  │
│  Broker thinks task wasn't done → Redelivers to another worker  │
│  Second worker sends the email AGAIN.                           │
│                                                                 │
│     Worker 1: [pick up] → [send email] → [CRASH]               │
│     Broker:   "Task not acknowledged, redelivering..."          │
│     Worker 2: [pick up] → [send email] → [ack] ✅              │
│     Result:   Customer gets TWO emails.                         │
│                                                                 │
│                                                                 │
│  SCENARIO 2: Visibility timeout expires                         │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  Worker picks up a long task → Takes too long → Broker assumes  │
│  the worker is dead → Redelivers to another worker.             │
│  Now TWO workers are running the same task.                     │
│                                                                 │
│     Worker 1: [pick up] → [still running..........]             │
│     Broker:   "Too slow, must be dead. Redelivering..."         │
│     Worker 2: [pick up] → [running too...]                      │
│     Result:   Task runs TWICE concurrently.                     │
│                                                                 │
│                                                                 │
│  SCENARIO 3: Network partition                                  │
│  ─────────────────────────────                                  │
│                                                                 │
│  Worker completes task → Sends ACK → Network drops ACK          │
│  Broker never receives ACK → Redelivers.                        │
│                                                                 │
│                                                                 │
│  YOU CANNOT PREVENT DUPLICATES in a distributed system.         │
│  You can only make duplicates HARMLESS.                         │
│  That's idempotency.                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 Making Tasks Idempotent (Patterns)

**An idempotent task produces the same result whether it runs 1 time or 10 times.**

```python
# ❌ NOT IDEMPOTENT: Charges the customer again on every run
@celery.task
def charge_customer_bad(order_id: int, amount: float) -> dict:
    payment_gateway.charge(amount)  # Runs again on duplicate!
    return {"status": "charged"}

# Running twice = customer charged twice. Disaster.


# ✅ IDEMPOTENT: Checks if already charged before charging
@celery.task
def charge_customer_good(order_id: int, amount: float) -> dict:
    engine = create_engine(DATABASE_URL)
    with Session(engine) as session:
        # Check: has this order already been charged?
        existing = session.query(Payment).filter_by(
            order_id=order_id,
            status="completed",
        ).first()
        
        if existing:
            # Already charged — return the existing result
            return {
                "status": "already_charged",
                "payment_id": existing.id,
            }
        
        # Not yet charged — proceed
        result = payment_gateway.charge(amount)
        
        # Record the charge
        session.add(Payment(
            order_id=order_id,
            amount=amount,
            status="completed",
            gateway_id=result.transaction_id,
        ))
        session.commit()
        
        return {"status": "charged", "payment_id": result.transaction_id}

# Running twice = second run returns "already_charged." Customer charged once.
```

**Three idempotency patterns:**

```
┌─────────────────────────────────────────────────────────────────┐
│              IDEMPOTENCY PATTERNS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PATTERN 1: CHECK-THEN-ACT                                      │
│  ─────────────────────────                                      │
│  Before doing work, check if it was already done.               │
│  "Has this email been sent?" → If yes, skip.                    │
│  Best for: operations with observable side effects.             │
│                                                                 │
│  PATTERN 2: DATABASE UNIQUE CONSTRAINTS                         │
│  ──────────────────────────────────────                         │
│  Use a unique constraint to prevent duplicate records.          │
│  INSERT fails on duplicate → catch IntegrityError → return.     │
│  Best for: data creation tasks.                                 │
│                                                                 │
│  PATTERN 3: IDEMPOTENCY KEY IN REDIS                            │
│  ────────────────────────────────────                           │
│  Use SET NX (set-if-not-exists) to claim a lock before work.    │
│  If the key already exists, another run already did it.         │
│  Best for: quick checks before expensive operations.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.4 Database-Backed Idempotency Keys

**The most robust pattern for production: a dedicated table that records completed work.**

```python
# Pattern 3 in practice — Redis-backed idempotency

import redis

redis_client = redis.Redis(host="localhost", port=6379, db=2)

@celery.task(bind=True)
def send_welcome_email(self, user_id: int) -> dict:
    """Send welcome email — guaranteed to send at most once."""
    
    idempotency_key = f"task:welcome_email:{user_id}"
    
    # Try to claim this task. SET NX = Set if Not eXists.
    # TTL of 86400 = key expires after 24 hours (cleanup).
    was_set = redis_client.set(idempotency_key, "processing", nx=True, ex=86400)
    
    if not was_set:
        # Key already existed — this task was already processed (or is in progress)
        return {"status": "already_processed", "user_id": user_id}
    
    try:
        # We claimed the key. Do the work.
        engine = create_engine(DATABASE_URL)
        with Session(engine) as session:
            user = session.query(User).filter_by(id=user_id).first()
            if not user:
                return {"status": "user_not_found"}
            
            smtp_send(to=user.email, subject="Welcome!", body="...")
            
            # Mark as completed
            redis_client.set(idempotency_key, "completed", ex=86400)
            return {"status": "sent", "to": user.email}
    
    except Exception:
        # If we fail, DELETE the key so a retry can try again
        redis_client.delete(idempotency_key)
        raise  # Let Celery's retry mechanism handle it
```

**How this prevents duplicates:**

```
┌─────────────────────────────────────────────────────────────────┐
│           IDEMPOTENCY KEY FLOW                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Run 1 (first execution):                                       │
│  ─────────────────────────                                      │
│  SET NX "task:welcome_email:42" → True (key created)            │
│  Send email → Success                                           │
│  SET "task:welcome_email:42" = "completed"                      │
│                                                                 │
│  Run 2 (duplicate delivery):                                    │
│  ──────────────────────────                                     │
│  SET NX "task:welcome_email:42" → False (key already exists!)   │
│  Return {"status": "already_processed"}                         │
│  Email NOT sent again. ✅                                       │
│                                                                 │
│  Run after failure + retry:                                     │
│  ──────────────────────────                                     │
│  SET NX "task:welcome_email:42" → True (key was deleted on fail)│
│  Send email → Success                                           │
│  Retry works correctly. ✅                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The database version — for when you need a permanent audit trail:**

```python
# In your models (sync SQLAlchemy, used by workers)
class ProcessedTask(Base):
    __tablename__ = "processed_tasks"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    idempotency_key: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    task_name: Mapped[str] = mapped_column(String(255))
    result: Mapped[dict] = mapped_column(JSONB, nullable=True)
    processed_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)


@celery.task(bind=True)
def process_invoice(self, invoice_id: int) -> dict:
    """Process an invoice — exactly once, with database proof."""
    
    idempotency_key = f"invoice:process:{invoice_id}"
    
    engine = create_engine(DATABASE_URL)
    with Session(engine) as session:
        # Check if already processed
        existing = session.query(ProcessedTask).filter_by(
            idempotency_key=idempotency_key
        ).first()
        
        if existing:
            return existing.result  # Return the cached result
        
        # Do the actual work
        invoice = session.query(Invoice).filter_by(id=invoice_id).first()
        result = billing_service.process(invoice)
        
        # Record completion (unique constraint prevents races)
        try:
            session.add(ProcessedTask(
                idempotency_key=idempotency_key,
                task_name="process_invoice",
                result=result,
            ))
            session.commit()
        except IntegrityError:
            # Another worker beat us (race condition) — that's fine
            session.rollback()
            existing = session.query(ProcessedTask).filter_by(
                idempotency_key=idempotency_key
            ).first()
            return existing.result
        
        return result
```

> "Idempotency isn't optional. It's not a 'nice to have.' In a distributed system, tasks WILL be duplicated. The only question is whether you handle it gracefully or your users get charged twice."

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                  CELERY QUICK REFERENCE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CREATE CELERY APP:                                             │
│      celery = Celery("worker",                                  │
│          broker="redis://localhost:6379/0",                      │
│          backend="redis://localhost:6379/1")                     │
│                                                                 │
│  DEFINE A TASK:                                                 │
│      @celery.task                                               │
│      def my_task(arg1: str, arg2: int) -> dict:                 │
│          return {"result": "done"}                              │
│                                                                 │
│  DEFINE A BOUND TASK (for retries):                             │
│      @celery.task(bind=True, max_retries=3)                     │
│      def my_task(self, arg1: str) -> dict:                      │
│          ...                                                    │
│                                                                 │
│  CALL A TASK (async/background):                                │
│      result = my_task.delay(arg1, arg2)                         │
│      result = my_task.apply_async(args=[...], countdown=60)     │
│                                                                 │
│  CALL A TASK (sync/direct — for testing):                       │
│      result = my_task(arg1, arg2)                               │
│                                                                 │
│  CHECK RESULT:                                                  │
│      result = AsyncResult(task_id, app=celery)                  │
│      result.status    → "PENDING" / "STARTED" / "SUCCESS" /... │
│      result.ready()   → True if done (success or failure)       │
│      result.result    → Return value (if SUCCESS)               │
│                       → Exception (if FAILURE)                  │
│                                                                 │
│  RETRY (manual):                                                │
│      raise self.retry(countdown=60, exc=original_error)         │
│                                                                 │
│  RETRY (declarative):                                           │
│      @celery.task(                                              │
│          autoretry_for=(ConnectionError,),                      │
│          retry_backoff=True,                                    │
│          retry_backoff_max=600,                                 │
│          retry_jitter=True,                                     │
│          max_retries=5,                                         │
│      )                                                          │
│                                                                 │
│  RUN THE WORKER:                                                │
│      celery -A worker.celery_app worker --loglevel=info         │
│                                                                 │
│  TASK ARGUMENTS — MUST be JSON-serializable:                    │
│      ✅ str, int, float, bool, None, list, dict                 │
│      ❌ Objects, models, sessions, file handles, datetime       │
│      ↳ Pass IDs. Let the task fetch its own data.               │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ async def task     → Use regular def (sync)             │
│      ❌ Pass SQLAlchemy    → Pass the ID, query in the task     │
│         model as argument                                       │
│      ❌ Use Depends()      → Task has no FastAPI context         │
│         in a task                                               │
│      ❌ Forget to run      → celery -A ... worker is required   │
│         the worker                                              │
│      ❌ Non-idempotent     → Always check before acting         │
│         task                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CELERY = WORK THAT LIVES OUTSIDE YOUR WEB SERVER               │
│                                                                 │
│  Your FastAPI server's job:                                     │
│      Accept requests → Respond fast → Delegate heavy work       │
│                                                                 │
│  Celery workers' job:                                           │
│      Pick up work → Execute it → Report results                 │
│                                                                 │
│  They communicate through Redis.                                │
│  They don't share memory, connections, or context.              │
│                                                                 │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐               │
│   │  FastAPI  │────▶│  Redis   │────▶│  Celery  │               │
│   │  "Do this"│     │ "Holding │     │ "Done.   │               │
│   │          │◀─ ─ ─│  it"     │◀─ ─ │  Here's  │               │
│   │          │      │          │     │  result" │               │
│   └──────────┘      └──────────┘     └──────────┘               │
│   Process 1         Service          Process 2                  │
│   (async)           (persists)       (sync)                     │
│                                                                 │
│                                                                 │
│  THE FACTORY ANALOGY:                                           │
│  ├─ Sales office (FastAPI) takes orders                         │
│  ├─ Conveyor belt (Redis) carries order forms                   │
│  ├─ Factory workers (Celery) build products                     │
│  ├─ Order number (task_id) tracks progress                      │
│  ├─ Defective → redo (retry with backoff)                       │
│  └─ Stamp each order "DONE" only once (idempotency)             │
│                                                                 │
│                                                                 │
│  THREE RULES TO REMEMBER:                                       │
│                                                                 │
│  1. Tasks are SYNC functions (workers don't need event loops)   │
│  2. Arguments must be JSON-serializable (pass IDs, not objects) │
│  3. Tasks MUST be idempotent (they WILL run more than once)     │
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
│  WEEK 11 LECTURE 3 (next lecture):                               │
│  └─ Celery Beat — SCHEDULING tasks to run periodically          │
│     (cron-like: "run this report every Monday at 6 AM")         │
│     Flower — MONITORING your workers and tasks via web UI       │
│     You'll schedule and observe the tasks you defined today.    │
│                                                                 │
│  WEEK 11 LECTURE 4:                                             │
│  └─ Event-Driven Architecture — Pub/Sub with Redis              │
│     Compare: task queue (Celery) vs event broadcast (pub/sub)   │
│     Different patterns for different problems.                  │
│                                                                 │
│  WEEK 11 PROJECT:                                               │
│  └─ Background Processing Pipeline                              │
│     You'll build: scheduled data jobs, event-triggered          │
│     notifications, retry logic, idempotent tasks, monitoring.   │
│     Everything from this lecture, wired together.               │
│                                                                 │
│  WEEK 12 (WebSockets & Performance):                            │
│  └─ Celery tasks triggering WebSocket notifications             │
│     "Report done!" → push to connected clients in real-time     │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Background jobs in your SaaS: email on task assignment,     │
│     report generation, audit logging, cache warming.            │
│     Celery is a core component of your final project.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```