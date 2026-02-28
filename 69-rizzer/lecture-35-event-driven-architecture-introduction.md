# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PAIN FIRST, PATTERN SECOND                                     │
│  ──────────────────────────                                     │
│  Students will see their OWN code pattern from the past         │
│  10 weeks collapse under new requirements. The coupling         │
│  explosion must be FELT before we introduce the cure.           │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Radio Station (broadcast, nobody cares who listens)            │
│  vs Postal Service (directed, someone must receive).            │
│  Every concept maps to one of these two models.                 │
│                                                                 │
│  CONCEPT BEFORE IMPLEMENTATION                                  │
│  ─────────────────────────────                                  │
│  The pub/sub diagram comes before redis.publish().              │
│  Architecture patterns come before tool selection.              │
│  Understand the WHY before choosing the WHAT.                   │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Redis (Week 10) → now messaging, not just caching              │
│  Celery (Lecture 2-3) → repositioned among alternatives         │
│  Async (Week 1) → async-native tools leverage your knowledge   │
│  BackgroundTasks (Lecture 1) → simplest tool revisited          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│          EVENT-DRIVEN ARCHITECTURE & TASK QUEUE SELECTION        │
│                       (3-4 Hour Lecture)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE COUPLING PROBLEM (40 min)                          │
│  ├─ 1.1 The Feature Request That Breaks Everything              │
│  ├─ 1.2 Commands vs Events (Two Ways to Communicate)            │
│  ├─ 1.3 The Radio Station Analogy                               │
│  └─ 1.4 The Pub/Sub Pattern                                     │
│                                                                 │
│  PART 2: REDIS PUB/SUB IN PRACTICE (50 min)                    │
│  ├─ 2.1 How Redis Pub/Sub Works (The Mechanism)                 │
│  ├─ 2.2 Publishing Events                                       │
│  ├─ 2.3 Subscribing to Channels                                 │
│  ├─ 2.4 Pattern Subscriptions (Wildcards)                       │
│  ├─ 2.5 Pub/Sub in a FastAPI Application                        │
│  └─ 2.6 The Critical Limitation: Messages Are Ephemeral         │
│                                                                 │
│  PART 3: ARCHITECTURE PATTERNS — AWARENESS (35 min)             │
│  ├─ 3.1 Event Sourcing — Recording History, Not Just State      │
│  ├─ 3.2 CQRS — Separate Reads from Writes                      │
│  └─ 3.3 When You'd Need Kafka or RabbitMQ (And When You Don't) │
│                                                                 │
│  PART 4: CHOOSING THE RIGHT TOOL (50 min)                       │
│  ├─ 4.1 The Landscape (What Exists and Why)                     │
│  ├─ 4.2 FastAPI BackgroundTasks (Recap & Positioning)           │
│  ├─ 4.3 Celery (Recap & Positioning)                            │
│  ├─ 4.4 Taskiq — The Async-Native Alternative                   │
│  ├─ 4.5 ARQ — The Lightweight Option                            │
│  └─ 4.6 The Decision Framework                                  │
│                                                                 │
│  PART 5: COMMON MISTAKES AND MISCONCEPTIONS (15 min)            │
│  ├─ 5.1 Using Pub/Sub When You Need Guaranteed Delivery         │
│  ├─ 5.2 Blocking the Event Loop Inside a Subscriber             │
│  ├─ 5.3 Unstructured Events (No Schema)                         │
│  ├─ 5.4 Over-Engineering (Kafka for 100 Events/Day)             │
│  └─ 5.5 Confusing Task Queues with Event Streams                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE COUPLING PROBLEM

## 1.1 The Feature Request That Breaks Everything

**Start with code they recognize. This is THEIR code pattern.**

Over the past 10 weeks you've been building something like this. A service function that does the core thing and then handles side effects one by one:

```python
# services/task_service.py — What your code probably looks like right now
from datetime import datetime

async def complete_task(
    task_id: int,
    db: AsyncSession,
    redis: Redis,
) -> Task:
    """Mark a task as completed."""
    task = await task_repo.get(db, task_id)
    task.status = "completed"
    task.completed_at = datetime.utcnow()
    await db.commit()
    await db.refresh(task)

    # Invalidate cache (you added this in Week 10)
    await redis.delete(f"task:{task_id}")

    return task
```

**Clean. Readable. Works. Now the requirements start rolling in.**

> "Boss says: send an email when a task is completed."

```python
async def complete_task(
    task_id: int,
    db: AsyncSession,
    redis: Redis,
) -> Task:
    task = await task_repo.get(db, task_id)
    task.status = "completed"
    task.completed_at = datetime.utcnow()
    await db.commit()
    await db.refresh(task)

    await redis.delete(f"task:{task_id}")

    # NEW: Send email notification
    send_notification_email.delay(task.assignee_id, task.title)

    return task
```

> "Boss says: update the project progress percentage when tasks complete."

```python
async def complete_task(
    task_id: int,
    db: AsyncSession,
    redis: Redis,
) -> Task:
    task = await task_repo.get(db, task_id)
    task.status = "completed"
    task.completed_at = datetime.utcnow()
    await db.commit()
    await db.refresh(task)

    await redis.delete(f"task:{task_id}")
    send_notification_email.delay(task.assignee_id, task.title)

    # NEW: Update project progress
    await update_project_progress(db, task.project_id)

    return task
```

> "Boss says: write an audit log for compliance. Also notify admins via WebSocket. Also invalidate the user's task list cache. Also trigger a Celery job to regenerate the weekly report..."

```python
async def complete_task(
    task_id: int,
    db: AsyncSession,
    redis: Redis,
    ws_manager: ConnectionManager,
) -> Task:
    task = await task_repo.get(db, task_id)
    task.status = "completed"
    task.completed_at = datetime.utcnow()
    await db.commit()
    await db.refresh(task)

    # Side effect 1: Cache invalidation
    await redis.delete(f"task:{task_id}")
    await redis.delete(f"user:{task.assignee_id}:tasks")
    await redis.delete(f"project:{task.project_id}:stats")

    # Side effect 2: Email notification
    send_notification_email.delay(task.assignee_id, task.title)

    # Side effect 3: Project progress
    await update_project_progress(db, task.project_id)

    # Side effect 4: Audit log
    await audit_log.record(
        action="task_completed",
        entity_id=task_id,
        user_id=task.assignee_id,
    )

    # Side effect 5: WebSocket broadcast
    await ws_manager.broadcast_to_project(
        task.project_id,
        {"event": "task_completed", "task_id": task_id},
    )

    # Side effect 6: Report regeneration
    regenerate_weekly_report.delay(task.project_id)

    return task
```

**Now ask the class:**

> "What was this function supposed to do?"

Answer: **Mark a task as completed.** One line of business logic. But now it's 6 side effects bolted on, and every new feature request forces you to open this function, add more parameters, add more imports, add more failure modes.

**Visualize the problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE COUPLING EXPLOSION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                  complete_task()                                 │
│                       │                                         │
│         ┌─────────────┼──────────────┐                          │
│         │             │              │                          │
│         ▼             ▼              ▼                          │
│    ┌─────────┐  ┌──────────┐  ┌──────────┐                     │
│    │  Redis  │  │  Email   │  │ Project  │                     │
│    │  Cache  │  │  Service │  │ Progress │                     │
│    └─────────┘  └──────────┘  └──────────┘                     │
│         │             │              │                          │
│         ▼             ▼              ▼                          │
│    ┌─────────┐  ┌──────────┐  ┌──────────┐                     │
│    │  Audit  │  │ WebSocket│  │  Report  │                     │
│    │  Log    │  │ Broadcast│  │  Regen   │                     │
│    └─────────┘  └──────────┘  └──────────┘                     │
│                                                                 │
│  complete_task() now KNOWS about 6 systems.                     │
│  It DEPENDS on 6 systems.                                       │
│  If ANY of those 6 change, this function changes.               │
│  If ANY of those 6 fail, this function must handle it.          │
│                                                                 │
│  Next month there will be 9 side effects.                       │
│  The month after that: 12.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Three specific problems with this approach:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  PROBLEM 1: SHOTGUN SURGERY                                     │
│  Every new side effect → edit complete_task()                   │
│  Every new side effect → add new parameter                      │
│  Every new side effect → add new import                         │
│  The function grows forever.                                    │
│                                                                 │
│  PROBLEM 2: CASCADING FAILURE                                   │
│  If the email service is down, does complete_task() fail?       │
│  Should the user see a 500 error because email is broken?       │
│  The task WAS completed. The core job IS done.                  │
│  Side effects are dragging the core operation into their mess.  │
│                                                                 │
│  PROBLEM 3: TESTING NIGHTMARE                                   │
│  To test complete_task() you now need to mock:                  │
│  - AsyncSession       - Redis                                   │
│  - Email service       - Audit logger                           │
│  - WebSocket manager   - Report generator                       │
│  Six mocks for a function that should just flip a status bit.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 Commands vs Events (Two Ways to Communicate)

**There are two fundamentally different ways for code to communicate.**

```
┌─────────────────────────────────────────────────────────────────┐
│               COMMANDS VS EVENTS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  COMMAND                             EVENT                      │
│  ───────                             ─────                      │
│  "Do this."                          "This happened."           │
│                                                                 │
│  Directed at a specific receiver.    Broadcast to anyone        │
│                                      who cares.                 │
│                                                                 │
│  The sender KNOWS who will           The sender DOES NOT KNOW   │
│  handle it.                          who will handle it.        │
│                                                                 │
│  The sender EXPECTS a result.        The sender does NOT WAIT   │
│                                      for results.               │
│                                                                 │
│                                                                 │
│  Example:                            Example:                   │
│  "Hey EmailService,                  "A task was completed.     │
│   send this email!"                   Just so you all know."    │
│                                                                 │
│  send_email(to, subject, body)       publish("task.completed",  │
│  ▲                                            {task_id: 42})    │
│  │                                   ▲                          │
│  └── I know exactly who              └── I have no idea who     │
│      I'm talking to.                     is listening.           │
│                                          And I don't care.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What you've been writing so far is COMMAND-driven:**

```python
# Commands — you tell specific systems what to do
send_notification_email.delay(user_id, title)     # "EmailService, send!"
await update_project_progress(db, project_id)      # "ProgressService, update!"
await audit_log.record(action, entity_id, user_id) # "AuditService, record!"
```

**The event-driven alternative:**

```python
# Events — you announce what happened, you don't tell anyone what to do
await publish("task.completed", {"task_id": 42, "project_id": 7})
# "A task was completed. I'm done. Whoever cares, deal with it."
```

**The function after this shift:**

```python
async def complete_task(
    task_id: int,
    db: AsyncSession,
    event_bus: EventBus,            # Only ONE extra dependency
) -> Task:
    task = await task_repo.get(db, task_id)
    task.status = "completed"
    task.completed_at = datetime.utcnow()
    await db.commit()
    await db.refresh(task)

    # Announce what happened. That's it.
    await event_bus.publish("task.completed", {
        "task_id": task.id,
        "project_id": task.project_id,
        "assignee_id": task.assignee_id,
        "title": task.title,
    })

    return task
```

**Visualize the difference:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  BEFORE (COMMAND-DRIVEN)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│               complete_task()                                   │
│              ╱   │    │    ╲                                     │
│            ╱     │    │      ╲                                   │
│          ╱       │    │        ╲                                 │
│        ▼         ▼    ▼          ▼                               │
│     Email    Cache  Audit    WebSocket                          │
│                                                                 │
│   Function must know about ALL consumers.                       │
│   Add a consumer → modify the function.                         │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                  AFTER (EVENT-DRIVEN)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│               complete_task()                                   │
│                     │                                           │
│                     ▼                                           │
│              ┌─────────────┐                                    │
│              │  Event Bus  │                                    │
│              │  (Pub/Sub)  │                                    │
│              └──────┬──────┘                                    │
│              ╱   │    │    ╲                                     │
│            ╱     │    │      ╲                                   │
│          ▼       ▼    ▼        ▼                                 │
│       Email   Cache  Audit  WebSocket                          │
│                                                                 │
│   Function knows about NOTHING except the event bus.            │
│   Add a consumer → subscribe to the event. Function untouched. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The function went from knowing about 6 systems to knowing about 0 systems. It just says what happened. The event bus does the rest."

---

## 1.3 The Radio Station Analogy

**This analogy will carry us through the rest of the lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE RADIO STATION ANALOGY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  COMMAND = POSTAL SERVICE                                       │
│  ────────────────────────                                       │
│                                                                 │
│  You write a letter addressed to a specific person.             │
│  The postal service delivers it to THAT person.                 │
│  If they're not home, the letter waits in the mailbox.          │
│  You KNOW who you're sending to.                                │
│  You EXPECT the letter to arrive.                               │
│                                                                 │
│       [You] ──letter──▶ [Postal Service] ──▶ [Alice]           │
│                                                                 │
│  This is Celery. You send a task TO a specific worker.          │
│                                                                 │
│                                                                 │
│  EVENT = RADIO STATION                                          │
│  ────────────────────                                           │
│                                                                 │
│  The station broadcasts on a frequency.                         │
│  ANYONE with a radio can tune in.                               │
│  The station doesn't know who's listening.                      │
│  The station doesn't CARE who's listening.                      │
│  If nobody tunes in, the broadcast still happens — into air.   │
│                                                                 │
│       [Station] ──broadcast──▶ 📻 [Alice]                      │
│                            ──▶ 📻 [Bob]                        │
│                            ──▶ 📻 [Charlie]                    │
│                            ──▶ 📻 [... anyone]                 │
│                                                                 │
│  This is Pub/Sub. You publish an event. Whoever subscribes,    │
│  receives it. You never know or care who that is.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to programming:**

```
Radio Station                  │  Event-Driven System
───────────────────────────────│──────────────────────────────────
Radio station (broadcaster)    │  Publisher (your service function)
Broadcast frequency / channel  │  Event channel / topic name
Radio listener                 │  Subscriber (event handler)
The broadcast itself           │  The published event (data payload)
Tuning your radio to 101.5 FM │  Subscribing to "task.completed"
Station doesn't know listeners │  Publisher doesn't know subscribers
Broadcast lost if nobody       │  Redis Pub/Sub: message lost if
  tunes in                     │    no subscriber is connected
```

**And the postal analogy to Celery:**

```
Postal Service                 │  Task Queue (Celery)
───────────────────────────────│──────────────────────────────────
You (sender)                   │  Producer (your endpoint)
Addressed letter               │  Task with arguments
Postal service                 │  Broker (Redis as Celery broker)
Recipient's mailbox            │  Worker queue
Letter waits until opened      │  Task waits until worker picks up
You KNOW the recipient         │  You KNOW the task function
Delivery confirmation          │  Task result / acknowledgment
```

> "Through the rest of this lecture, whenever I say 'broadcast,' think radio station. Whenever I say 'dispatch,' think postal service. These are two completely different communication models, and choosing the wrong one is a real-world mistake."

---

## 1.4 The Pub/Sub Pattern

**Formalize the pattern.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE PUB/SUB PATTERN                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PUBLISHER                                                      │
│  ─────────                                                      │
│  The code that announces "something happened."                  │
│  It does NOT know who will receive the message.                 │
│  It does NOT wait for anyone to handle it.                      │
│                                                                 │
│  CHANNEL (also called Topic)                                    │
│  ───────                                                        │
│  A named stream that publishers write to and subscribers        │
│  read from. Like a radio frequency.                             │
│  Examples: "task.completed", "user.registered", "order.paid"    │
│                                                                 │
│  SUBSCRIBER                                                     │
│  ──────────                                                     │
│  The code that listens on a channel and reacts.                 │
│  It does NOT know who published the message.                    │
│  Multiple subscribers can listen to the same channel.           │
│                                                                 │
│  BROKER                                                         │
│  ──────                                                         │
│  The infrastructure that routes messages from publishers        │
│  to subscribers. Redis, RabbitMQ, Kafka — this is the           │
│  radio tower that carries the signal.                           │
│                                                                 │
│                                                                 │
│  ┌───────────┐    publish     ┌───────────┐    deliver          │
│  │ Publisher  │ ────────────▶ │  Channel  │ ────────────▶ Sub A │
│  │           │  "task.done"  │ (Broker)  │ ────────────▶ Sub B │
│  └───────────┘               └───────────┘ ────────────▶ Sub C │
│                                                                 │
│  Publisher: 1        Channel: 1        Subscribers: N           │
│  (or many)                             (zero or many)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The three guarantees of pub/sub:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. DECOUPLING                                                  │
│     Publisher and subscriber do not know about each other.      │
│     They only know about the channel.                           │
│                                                                 │
│  2. ONE-TO-MANY                                                 │
│     One event can trigger multiple independent reactions.       │
│     Adding a new reaction = adding a new subscriber.            │
│     No existing code changes.                                   │
│                                                                 │
│  3. FIRE-AND-FORGET (from the publisher's perspective)          │
│     The publisher sends and moves on.                           │
│     It does not wait. It does not check.                        │
│     It's the subscriber's job to handle the message.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now map this back to our task completion problem:**

```python
# PUBLISHERS — they announce what happened
# (inside complete_task)
await event_bus.publish("task.completed", {
    "task_id": 42,
    "project_id": 7,
    "assignee_id": 3,
    "title": "Fix login bug",
})

# SUBSCRIBERS — each handles ONE concern, independently
# They are defined ELSEWHERE. complete_task() doesn't know they exist.

# subscriber_email.py
async def on_task_completed_send_email(event: dict):
    user = await get_user(event["assignee_id"])
    await send_email(user.email, f"Task done: {event['title']}")

# subscriber_cache.py
async def on_task_completed_invalidate_cache(event: dict):
    await redis.delete(f"task:{event['task_id']}")
    await redis.delete(f"project:{event['project_id']}:stats")

# subscriber_audit.py
async def on_task_completed_audit_log(event: dict):
    await audit_log.record("task_completed", entity_id=event["task_id"])

# subscriber_ws.py
async def on_task_completed_notify_ws(event: dict):
    await ws_manager.broadcast_to_project(
        event["project_id"],
        {"event": "task_completed", "task_id": event["task_id"]},
    )

# subscriber_report.py
async def on_task_completed_regen_report(event: dict):
    regenerate_weekly_report.delay(event["project_id"])
```

> "Five subscribers. Five files. Five independent concerns. And `complete_task()` knows about NONE of them. Want to add a sixth reaction next month? Write a new subscriber, subscribe to the channel. Zero changes to `complete_task()`."

---

# PART 2: REDIS PUB/SUB IN PRACTICE

## 2.1 How Redis Pub/Sub Works (The Mechanism)

**You already know Redis as a cache (Week 10). Now meet Redis as a message broker.**

Redis Pub/Sub is a built-in messaging system that is completely separate from the key-value operations you've been using. No `SET`, no `GET`, no `TTL`. Different mechanism entirely.

```
┌─────────────────────────────────────────────────────────────────┐
│                  REDIS PUB/SUB — HOW IT WORKS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REDIS SERVER                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │  Key-Value Store (what you've used)                     │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │ "task:42"  → {"status": "completed"}            │    │    │
│  │  │ "user:3"   → {"name": "Alice"}                  │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  │                                                         │    │
│  │  Pub/Sub Engine (what we're learning now)               │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │ Channel: "task.completed"                       │    │    │
│  │  │   └─ Subscribers: [Client A, Client B]          │    │    │
│  │  │ Channel: "user.registered"                      │    │    │
│  │  │   └─ Subscribers: [Client C]                    │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  │                                                         │    │
│  │  These are TWO SEPARATE SYSTEMS inside the same Redis.  │    │
│  │  Pub/Sub does NOT store data. No keys. No TTL. No GET.  │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The lifecycle of a pub/sub message:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MESSAGE LIFECYCLE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Subscriber connects and SUBSCRIBES to a channel             │
│     Client A ──▶ SUBSCRIBE "task.completed"                     │
│     Client B ──▶ SUBSCRIBE "task.completed"                     │
│                                                                 │
│  2. Publisher sends a message to that channel                   │
│     Client X ──▶ PUBLISH "task.completed" '{"task_id": 42}'    │
│                                                                 │
│  3. Redis IMMEDIATELY forwards to ALL connected subscribers     │
│     Redis ──▶ Client A: '{"task_id": 42}'                      │
│     Redis ──▶ Client B: '{"task_id": 42}'                      │
│                                                                 │
│  4. Message is GONE. Not stored. Not retrievable.               │
│     If Client C subscribes 1 second later, it missed it.       │
│                                                                 │
│  ⚠️  This is FUNDAMENTALLY different from Celery.               │
│     Celery stores tasks in the broker until a worker picks up.  │
│     Redis Pub/Sub delivers live and forgets immediately.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Publishing Events

**Publishing is the easy side. One function call.**

You're using `redis.asyncio` — the same client from Week 10.

```python
import redis.asyncio as aioredis
import json
from datetime import datetime, timezone


async def publish_event(
    redis: aioredis.Redis,
    channel: str,
    data: dict,
) -> int:
    """
    Publish an event to a Redis Pub/Sub channel.

    Returns the number of subscribers who received the message.
    """
    message = json.dumps({
        "event": channel,
        "data": data,
        "published_at": datetime.now(timezone.utc).isoformat(),
    })
    receiver_count = await redis.publish(channel, message)
    return receiver_count
```

**Using it in your service:**

```python
async def complete_task(
    task_id: int,
    db: AsyncSession,
    redis: aioredis.Redis,
) -> Task:
    task = await task_repo.get(db, task_id)
    task.status = "completed"
    task.completed_at = datetime.now(timezone.utc)
    await db.commit()
    await db.refresh(task)

    # Publish the event — that's it
    receiver_count = await publish_event(redis, "task.completed", {
        "task_id": task.id,
        "project_id": task.project_id,
        "assignee_id": task.assignee_id,
        "title": task.title,
    })
    # receiver_count tells you how many subscribers got it
    # If 0 → nobody was listening. The message is gone.

    return task
```

**What `redis.publish()` returns:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  await redis.publish("task.completed", message)                 │
│                                                                 │
│  Returns: int — the number of clients that received it          │
│                                                                 │
│  Returns 0  → No one was subscribed. Message lost forever.      │
│  Returns 1  → One subscriber got it.                            │
│  Returns 3  → Three subscribers got it.                         │
│                                                                 │
│  The publisher does NOT get confirmation that subscribers       │
│  successfully PROCESSED the message. Only that they RECEIVED.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Subscribing to Channels

**The subscriber runs a persistent loop, listening for messages.**

```python
import redis.asyncio as aioredis
import json


async def subscribe_to_events(redis: aioredis.Redis) -> None:
    """
    Long-running subscriber that listens for task events.
    This function runs FOREVER (until cancelled).
    """
    pubsub = redis.pubsub()
    await pubsub.subscribe("task.completed", "task.created")

    async for message in pubsub.listen():
        # Redis sends several message types
        if message["type"] != "message":
            # "subscribe" confirmation, "unsubscribe", etc. — skip
            continue

        channel = message["channel"]  # bytes, e.g. b"task.completed"
        raw_data = message["data"]    # bytes, e.g. b'{"event": ...}'

        # Decode (Redis gives us bytes)
        if isinstance(channel, bytes):
            channel = channel.decode()
        if isinstance(raw_data, bytes):
            raw_data = raw_data.decode()

        event = json.loads(raw_data)
        print(f"Received on '{channel}': {event}")

        # Route to the right handler
        await handle_event(channel, event)


async def handle_event(channel: str, event: dict) -> None:
    """Route events to their handlers."""
    handlers: dict[str, list] = {
        "task.completed": [
            on_task_completed_invalidate_cache,
            on_task_completed_send_email,
            on_task_completed_log_audit,
        ],
        "task.created": [
            on_task_created_notify_project,
        ],
    }

    for handler in handlers.get(channel, []):
        try:
            await handler(event["data"])
        except Exception as e:
            # One handler failing should NOT kill the others
            print(f"Handler {handler.__name__} failed: {e}")
```

**Visualize the subscriber loop:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SUBSCRIBER LOOP                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   subscribe_to_events()                                         │
│          │                                                      │
│          ▼                                                      │
│   ┌─── pubsub.subscribe("task.completed") ◀──── register       │
│   │                                                             │
│   │    ┌───────────────────┐                                    │
│   └──▶ │   Wait for next   │ ◀─────────────────┐               │
│        │   message...      │                    │               │
│        └────────┬──────────┘                    │               │
│                 │ message arrives                │               │
│                 ▼                                │               │
│        ┌───────────────────┐                    │               │
│        │ Is type "message"?│                    │               │
│        └────┬─────────┬────┘                    │               │
│           Yes         No                        │               │
│            │          └── skip ──────────────────┤               │
│            ▼                                    │               │
│        ┌───────────────────┐                    │               │
│        │  Decode JSON      │                    │               │
│        │  Route to handler │                    │               │
│        │  Execute handler  │                    │               │
│        └────────┬──────────┘                    │               │
│                 │                               │               │
│                 └───────────────────────────────┘               │
│                                                                 │
│   This loop runs FOREVER. It's a long-lived process.           │
│   Like a radio that's always on, always listening.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Message types from Redis Pub/Sub:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    MESSAGE TYPES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "subscribe"     → Confirmation that you subscribed             │
│                    {"type": "subscribe",                        │
│                     "channel": "task.completed",                │
│                     "data": 1}  ← number of active subs        │
│                                                                 │
│  "message"       → An actual published message                  │
│                    {"type": "message",                          │
│                     "channel": "task.completed",                │
│                     "data": '{"task_id": 42}'}                  │
│                                                                 │
│  "unsubscribe"   → Confirmation that you unsubscribed           │
│                    {"type": "unsubscribe",                      │
│                     "channel": "task.completed",                │
│                     "data": 0}  ← remaining subs               │
│                                                                 │
│  You only care about "message". Filter the rest.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Pattern Subscriptions (Wildcards)

**Instead of subscribing to exact channel names, subscribe to patterns.**

```python
pubsub = redis.pubsub()

# Exact subscription — only "task.completed"
await pubsub.subscribe("task.completed")

# Pattern subscription — any channel starting with "task."
await pubsub.psubscribe("task.*")
# Matches: "task.completed", "task.created", "task.deleted"
# Does NOT match: "user.registered", "order.paid"
```

**This is like tuning your radio to a range of frequencies instead of one.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  PATTERN SUBSCRIPTIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pattern              Matches                                   │
│  ───────              ───────                                   │
│  "task.*"             task.completed, task.created               │
│  "*.completed"        task.completed, order.completed            │
│  "project.*.updated"  project.settings.updated,                 │
│                       project.members.updated                   │
│                                                                 │
│  Use case:                                                      │
│  An audit log subscriber that needs to record ALL events:       │
│                                                                 │
│  await pubsub.psubscribe("*")  ← listen to EVERYTHING          │
│                                                                 │
│  A project-level dashboard subscriber:                          │
│                                                                 │
│  await pubsub.psubscribe("project.42.*")  ← all events for     │
│                                              project 42         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Pattern subscriptions produce `"pmessage"` type, not `"message"`:**

```python
async for message in pubsub.listen():
    if message["type"] == "pmessage":
        pattern = message["pattern"]   # b"task.*"
        channel = message["channel"]   # b"task.completed"
        data = message["data"]         # b'{"task_id": 42}'
```

---

## 2.5 Pub/Sub in a FastAPI Application

**How do you actually wire this into the app you've been building?**

The publisher side is straightforward — call `redis.publish()` from any endpoint or service function. The subscriber side is the challenge: you need a long-running loop that starts when your app starts and stops when your app stops.

**Use FastAPI's `lifespan` — a context manager that wraps your app's lifecycle.**

If you've set up your Redis connection pool, you've already seen this pattern. It builds on the context manager concept from Week 1, Lecture 2.

```python
# main.py
import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI
import redis.asyncio as aioredis

from app.events.subscriber import subscribe_to_events


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ── STARTUP ──
    app.state.redis = aioredis.from_url("redis://localhost:6379")

    # Start the subscriber as a background task
    subscriber_task = asyncio.create_task(
        subscribe_to_events(app.state.redis)
    )

    yield  # ← App is running, handling requests

    # ── SHUTDOWN ──
    subscriber_task.cancel()
    try:
        await subscriber_task
    except asyncio.CancelledError:
        pass  # Expected — we cancelled it
    await app.state.redis.aclose()


app = FastAPI(lifespan=lifespan)
```

**Visualize the architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 FASTAPI + REDIS PUB/SUB                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FastAPI Process                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │  ┌──────────────────────┐   ┌────────────────────────┐  │    │
│  │  │   Request Handlers   │   │   Event Subscriber     │  │    │
│  │  │   (your endpoints)   │   │   (background task)    │  │    │
│  │  │                      │   │                        │  │    │
│  │  │  POST /tasks/42/     │   │   Listening on:        │  │    │
│  │  │    complete          │   │   - "task.completed"   │  │    │
│  │  │      │               │   │   - "task.created"     │  │    │
│  │  │      │ publish ──────│───│──▶ handle_event()      │  │    │
│  │  │      │               │   │     ├─ send email      │  │    │
│  │  │      ▼               │   │     ├─ clear cache     │  │    │
│  │  │  return response     │   │     └─ log audit       │  │    │
│  │  └──────────────────────┘   └────────────────────────┘  │    │
│  │            │                          ▲                  │    │
│  │            │         ┌────────────────┘                  │    │
│  │            ▼         │                                   │    │
│  │       ┌──────────────────┐                               │    │
│  │       │   Redis Server   │                               │    │
│  │       │   (Pub/Sub +     │                               │    │
│  │       │    Cache)        │                               │    │
│  │       └──────────────────┘                               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Both publisher and subscriber live in the SAME process.        │
│  The event goes: endpoint → Redis → back to same process.       │
│  This seems silly — but it DECOUPLES the code.                  │
│  And when you scale to multiple processes, it still works.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A more structured event system for production:**

```python
# app/events/schemas.py
from pydantic import BaseModel
from datetime import datetime, timezone


class TaskCompletedEvent(BaseModel):
    task_id: int
    project_id: int
    assignee_id: int
    title: str


class EventEnvelope(BaseModel):
    """Every event gets wrapped in a standard envelope."""
    event_type: str
    data: dict
    published_at: datetime

    @classmethod
    def create(cls, event_type: str, data: BaseModel) -> "EventEnvelope":
        return cls(
            event_type=event_type,
            data=data.model_dump(),
            published_at=datetime.now(timezone.utc),
        )
```

```python
# app/events/publisher.py
import json
import redis.asyncio as aioredis
from app.events.schemas import EventEnvelope


class EventPublisher:
    def __init__(self, redis: aioredis.Redis) -> None:
        self._redis = redis

    async def publish(self, channel: str, envelope: EventEnvelope) -> int:
        """Publish a structured event. Returns subscriber count."""
        message = envelope.model_dump_json()
        return await self._redis.publish(channel, message)
```

```python
# app/events/handlers.py
from app.events.schemas import EventEnvelope


async def handle_task_completed_email(envelope: EventEnvelope) -> None:
    data = envelope.data
    # ... send email using data["assignee_id"], data["title"]


async def handle_task_completed_cache(envelope: EventEnvelope) -> None:
    data = envelope.data
    # ... invalidate cache for data["task_id"], data["project_id"]
```

```python
# app/events/subscriber.py
import json
import redis.asyncio as aioredis
from app.events.schemas import EventEnvelope
from app.events.handlers import (
    handle_task_completed_email,
    handle_task_completed_cache,
)

# Registry: channel → list of handler functions
HANDLERS: dict[str, list] = {
    "task.completed": [
        handle_task_completed_email,
        handle_task_completed_cache,
    ],
}


async def subscribe_to_events(redis: aioredis.Redis) -> None:
    pubsub = redis.pubsub()
    await pubsub.subscribe(*HANDLERS.keys())

    async for message in pubsub.listen():
        if message["type"] != "message":
            continue

        channel = message["channel"]
        if isinstance(channel, bytes):
            channel = channel.decode()

        raw = message["data"]
        if isinstance(raw, bytes):
            raw = raw.decode()

        envelope = EventEnvelope.model_validate_json(raw)

        for handler in HANDLERS.get(channel, []):
            try:
                await handler(envelope)
            except Exception as e:
                # Log error, but don't crash the subscriber loop
                print(f"Handler {handler.__name__} failed: {e}")
```

> "Notice: Pydantic validates the event structure. Type hints everywhere. The handler registry is a simple dict. Adding a new handler? Add one line to HANDLERS and write the function. Zero changes to the publisher."

---

## 2.6 The Critical Limitation: Messages Are Ephemeral

**This is the most important thing to understand about Redis Pub/Sub.**

```
┌─────────────────────────────────────────────────────────────────┐
│            ⚠️  REDIS PUB/SUB = FIRE AND FORGET                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  There is NO message persistence. None.                         │
│                                                                 │
│  If a message is published and NO subscriber is connected:      │
│  → The message is GONE. FOREVER. No retry. No queue. Nothing.   │
│                                                                 │
│  If a subscriber crashes mid-processing:                        │
│  → The message is GONE. It was already delivered. No re-read.   │
│                                                                 │
│  If your app restarts (deployment, crash, scaling):             │
│  → All messages published during the restart gap are GONE.      │
│                                                                 │
│                                                                 │
│  RADIO STATION ANALOGY:                                         │
│  ──────────────────────                                         │
│  If you turn your radio off, you miss the broadcast.            │
│  The station doesn't record it for you.                         │
│  You can't rewind. You can't ask for a replay.                  │
│  It was broadcast. It's over.                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When is this acceptable?**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ✅ EPHEMERAL IS FINE FOR:                                      │
│  ├─ Cache invalidation (miss one? Cache just stays stale        │
│  │   a bit longer — it has TTL anyway)                          │
│  ├─ Real-time UI updates (user refreshes page, gets new data)   │
│  ├─ Live dashboards (next data point comes in seconds)          │
│  └─ Development/debugging event monitoring                      │
│                                                                 │
│  ❌ EPHEMERAL IS DANGEROUS FOR:                                 │
│  ├─ Email notifications (user NEVER gets the email)             │
│  ├─ Billing events (payment not recorded → money lost)          │
│  ├─ Audit logs (compliance failure, legal liability)            │
│  └─ Anything where losing a message has BUSINESS consequences   │
│                                                                 │
│  RULE: If losing ONE message keeps you up at night,             │
│  Redis Pub/Sub is the wrong tool. Use Celery, a persistent      │
│  message broker, or Redis Streams instead.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This is the single most common mistake people make with Redis Pub/Sub. They use it for critical business events because 'Redis is fast' and then discover months later that they've been silently losing messages during every deployment. Don't be that person."

---

# PART 3: ARCHITECTURE PATTERNS — AWARENESS

> "The following two patterns are concepts you'll encounter in architecture discussions, technical blogs, and system design interviews. We are NOT implementing them. You are learning to RECOGNIZE them, understand when they make sense, and know enough to participate in a conversation about them."

## 3.1 Event Sourcing — Recording History, Not Just State

**The normal approach (what you've been doing):**

```
┌─────────────────────────────────────────────────────────────────┐
│                   STATE-BASED STORAGE                           │
│                   (What you do now)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You store the CURRENT state of an entity.                      │
│                                                                 │
│  ┌─────────────────────────────┐                                │
│  │  tasks table                │                                │
│  ├─────────────────────────────┤                                │
│  │  id: 42                     │                                │
│  │  title: "Fix login bug"     │                                │
│  │  status: "completed"        │ ← Only the current state      │
│  │  assignee_id: 3             │                                │
│  │  updated_at: 2026-02-16     │                                │
│  └─────────────────────────────┘                                │
│                                                                 │
│  Question: Who changed the status from "in_progress"            │
│  to "completed"? When? Was it ever changed back?                │
│  Was it "blocked" at some point?                                │
│                                                                 │
│  Answer: You don't know. You overwrote the history.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The event sourcing approach:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   EVENT SOURCING                                │
│                   (Alternative approach)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You store EVERY EVENT that happened to the entity.             │
│  Current state = replay all events from the beginning.          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  task_events table                                      │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │  1. TaskCreated    {title: "Fix login", assignee: 3}    │    │
│  │  2. TaskStarted    {started_by: 3}                      │    │
│  │  3. TaskBlocked    {reason: "waiting on design"}        │    │
│  │  4. TaskUnblocked  {unblocked_by: 5}                    │    │
│  │  5. TaskReassigned {from: 3, to: 7}                     │    │
│  │  6. TaskCompleted  {completed_by: 7}                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Current state? Replay events 1→6:                              │
│    title="Fix login", status="completed", assignee=7            │
│                                                                 │
│  State at event 3? Replay events 1→3:                           │
│    title="Fix login", status="blocked", assignee=3              │
│                                                                 │
│  Full audit trail? Read the events. It's ALL there.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Think of it like a bank account:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  STATE-BASED BANK:                                              │
│  ┌──────────────────────┐                                       │
│  │  Account #12345      │                                       │
│  │  Balance: $500       │  ← That's all you know                │
│  └──────────────────────┘                                       │
│                                                                 │
│  EVENT-SOURCED BANK:                                            │
│  ┌──────────────────────────────────────────────┐               │
│  │  Jan 01: AccountOpened      +$0     = $0     │               │
│  │  Jan 05: DepositMade        +$1000  = $1000  │               │
│  │  Jan 12: WithdrawalMade     -$300   = $700   │               │
│  │  Jan 20: TransferReceived   +$200   = $900   │               │
│  │  Feb 01: WithdrawalMade     -$400   = $500   │               │
│  └──────────────────────────────────────────────┘               │
│                                                                 │
│  Same balance. But the event-sourced version can answer:        │
│  • "What was my balance on Jan 15?" → $700                      │
│  • "How much did I withdraw total?" → $700                      │
│  • "Show me all deposits." → Jan 05, Jan 20                     │
│                                                                 │
│  The state-based version can answer: "$500. That's it."         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When event sourcing makes sense:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ✅ USE WHEN:                                                   │
│  ├─ Audit trails are legally required (finance, healthcare)     │
│  ├─ You need "time travel" (undo, replay, reconstruct state)    │
│  ├─ Multiple systems derive different views from the same data  │
│  └─ Debugging production issues ("what happened to order #X?")  │
│                                                                 │
│  ❌ DON'T USE WHEN:                                             │
│  ├─ Simple CRUD is enough (most apps, including your project)   │
│  ├─ You don't need history (overwrite is fine)                  │
│  └─ Your team is small and complexity would slow you down       │
│                                                                 │
│  EVENT SOURCING IS POWERFUL BUT COMPLEX.                        │
│  It changes how you think about data fundamentally.             │
│  For this course: know it exists, know when it shines,          │
│  don't implement it unless the domain demands it.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 CQRS — Separate Reads from Writes

**CQRS stands for Command Query Responsibility Segregation.**

The core idea: the model you use to WRITE data doesn't have to be the same model you use to READ data.

```
┌─────────────────────────────────────────────────────────────────┐
│                     TRADITIONAL APPROACH                        │
│                     (What you do now)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│               ┌──────────────┐                                  │
│   Writes ──▶  │              │  ◀── Reads                       │
│               │  SAME MODEL  │                                  │
│   (CREATE,    │  SAME TABLE  │  (SELECT, JOIN,                  │
│    UPDATE,    │  SAME SCHEMA │   aggregate, filter)             │
│    DELETE)    │              │                                  │
│               └──────────────┘                                  │
│                                                                 │
│  Your SQLAlchemy model is used for both writing and reading.    │
│  This is fine for most applications.                            │
│                                                                 │
│  But what if your write model (normalized, consistent)          │
│  is SLOW to read (complex JOINs, aggregations)?                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                      CQRS APPROACH                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│             COMMAND SIDE            QUERY SIDE                   │
│            (Write Model)           (Read Model)                 │
│                                                                 │
│   Writes ──▶ ┌────────────┐       ┌────────────┐ ──▶ Reads     │
│              │ Normalized │       │Denormalized│                │
│              │ tables     │──sync─▶│ views /    │                │
│              │ (3NF)      │       │ projections│                │
│              └────────────┘       └────────────┘                │
│                                                                 │
│   Optimized for:                  Optimized for:                │
│   • Data integrity                • Fast queries                │
│   • Consistency                   • No JOINs needed             │
│   • Normalization                 • Pre-computed aggregates     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A concrete example with your Task Manager:**

```
┌─────────────────────────────────────────────────────────────────┐
│               CQRS EXAMPLE: PROJECT DASHBOARD                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WRITE SIDE (normalized — what you have now):                   │
│                                                                 │
│  tasks: id, title, status, assignee_id, project_id             │
│  users: id, name, email                                        │
│  projects: id, name, org_id                                    │
│                                                                 │
│  To build a dashboard you need to SELECT + JOIN + GROUP BY:     │
│  "For project 7, give me task count by status,                  │
│   assignee names, completion percentage, overdue count."        │
│  → Complex query. Multiple JOINs. Slow if tables are large.    │
│                                                                 │
│                                                                 │
│  QUERY SIDE (denormalized — CQRS optimization):                 │
│                                                                 │
│  project_dashboard: project_id, total_tasks, completed_tasks,   │
│                     in_progress_tasks, overdue_tasks,           │
│                     completion_pct, top_assignees_json,         │
│                     last_updated_at                             │
│                                                                 │
│  Dashboard query: SELECT * FROM project_dashboard WHERE id = 7  │
│  → One table. No JOINs. Instant.                               │
│                                                                 │
│  The "sync" step: whenever a task changes, an event triggers   │
│  a handler that updates the project_dashboard row.              │
│  (See how pub/sub feeds into this?)                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When CQRS makes sense:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ✅ USE WHEN:                                                   │
│  ├─ Read and write patterns are drastically different           │
│  ├─ Read-heavy systems (100 reads per 1 write)                 │
│  ├─ Complex read queries that slow down under load             │
│  └─ Different teams own the read and write paths               │
│                                                                 │
│  ❌ DON'T USE WHEN:                                             │
│  ├─ Reads and writes use similar models (most CRUD apps)       │
│  ├─ Data must be instantly consistent (no sync delay)          │
│  └─ The added complexity isn't justified by performance needs  │
│                                                                 │
│  KEY TRADEOFF: Consistency vs Performance.                      │
│  Your read model is "eventually consistent" — there's a tiny   │
│  delay between writing data and seeing it in the read model.    │
│  For a real-time trading system? Probably unacceptable.         │
│  For a project dashboard? Nobody notices a 500ms delay.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice how event sourcing and CQRS connect to pub/sub. Event sourcing generates events. Pub/Sub distributes them. CQRS subscribers build optimized read models from those events. These patterns are building blocks that compose together. You don't need all of them. But when you do, they're a natural fit."

---

## 3.3 When You'd Need Kafka or RabbitMQ (And When You Don't)

**You've now seen two messaging tools: Redis Pub/Sub (this lecture) and Celery with Redis as broker (Lectures 2-3). When are these not enough?**

```
┌─────────────────────────────────────────────────────────────────┐
│              MESSAGE BROKER LANDSCAPE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REDIS PUB/SUB                                                  │
│  ─────────────                                                  │
│  Delivery: At-most-once (fire and forget)                       │
│  Persistence: None                                              │
│  Replay: Impossible                                             │
│  Scale: Single Redis instance (millions of msg/sec possible)    │
│  Best for: Real-time notifications, cache invalidation          │
│                                                                 │
│  🔑 Think: LIVE radio broadcast. Miss it, it's gone.           │
│                                                                 │
│                                                                 │
│  RABBITMQ                                                       │
│  ────────                                                       │
│  Delivery: At-least-once, at-most-once, exactly-once possible   │
│  Persistence: Yes (messages survive broker restart)              │
│  Replay: No (message is deleted once acknowledged)              │
│  Scale: Clustering, but not infinite                            │
│  Best for: Reliable task routing, complex routing rules,        │
│            when messages MUST be processed                       │
│                                                                 │
│  🔑 Think: Postal service with tracking.                        │
│     Guaranteed delivery. Signed receipt.                         │
│                                                                 │
│                                                                 │
│  KAFKA                                                          │
│  ─────                                                          │
│  Delivery: At-least-once (exactly-once with transactions)       │
│  Persistence: Yes (messages stored for days/weeks/forever)      │
│  Replay: Yes! Any consumer can re-read from any point in time.  │
│  Scale: Massive horizontal scaling (partitions, consumer groups)│
│  Best for: Event streaming at scale, event sourcing,            │
│            log aggregation, data pipelines                       │
│                                                                 │
│  🔑 Think: Newspaper archive. Read today's paper.               │
│     Read last week's paper. Read from any date you want.        │
│     The papers don't disappear when you read them.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When do you ACTUALLY need these?**

```
┌─────────────────────────────────────────────────────────────────┐
│             DO I NEED KAFKA / RABBITMQ?                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You probably DON'T if:                                         │
│  ├─ Your app has < 1000 users                                  │
│  ├─ Redis Pub/Sub + Celery cover your needs                    │
│  ├─ Message loss is tolerable (cache invalidation, live UI)    │
│  ├─ You're a small team (< 5 engineers)                        │
│  └─ You're not doing event sourcing                            │
│                                                                 │
│  You might need RABBITMQ if:                                    │
│  ├─ Messages MUST be delivered (billing, notifications)         │
│  ├─ You need complex routing (message goes to different        │
│  │   queues based on content, headers, patterns)               │
│  ├─ Multiple consumers need to SHARE load (work queues)        │
│  └─ Celery's features aren't enough and you want more control  │
│                                                                 │
│  You might need KAFKA if:                                       │
│  ├─ You process millions of events per day                     │
│  ├─ Multiple independent services need the SAME events         │
│  ├─ You need to REPLAY events (rebuild state, fix bugs)        │
│  ├─ You're building data pipelines (analytics, ML features)    │
│  └─ You're doing event sourcing at scale                       │
│                                                                 │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  REALITY CHECK:                                          │   │
│  │                                                          │   │
│  │  Most startups and small-to-medium applications will     │   │
│  │  NEVER need Kafka. Redis + Celery handles an enormous    │   │
│  │  range of use cases. Don't add infrastructure complexity │   │
│  │  you don't need yet.                                     │   │
│  │                                                          │   │
│  │  If you're handling 100 events per day and you reach     │   │
│  │  for Kafka, you're building a rocket to cross the        │   │
│  │  street.                                                 │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: CHOOSING THE RIGHT TOOL

## 4.1 The Landscape (What Exists and Why)

> "You now know Celery and FastAPI BackgroundTasks from this week. You just learned Redis Pub/Sub. Now I'm going to show you two more tools — Taskiq and ARQ — because the Python async ecosystem has matured, and you should know what's available. Then we'll build a decision framework so you never have to guess."

```
┌─────────────────────────────────────────────────────────────────┐
│              THE PYTHON BACKGROUND TASK LANDSCAPE               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│            Simple ◀──────────────────────────▶ Complex          │
│                                                                 │
│  BackgroundTasks    ARQ        Taskiq        Celery             │
│       │              │           │             │                │
│       ▼              ▼           ▼             ▼                │
│  ┌──────────┐  ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │In-process│  │Redis-only│ │Async-    │ │Industry  │          │
│  │No broker │  │Lightweight│ │native   │ │standard  │          │
│  │No persist│  │Async-    │ │Multiple  │ │Battle-   │          │
│  │Fire-and- │  │first     │ │brokers   │ │tested    │          │
│  │forget    │  │Simple API│ │Type-safe │ │Sync-first│          │
│  └──────────┘  └──────────┘ └──────────┘ └──────────┘          │
│                                                                 │
│  "I need a          "I need         "I need        "I need     │
│   quick side         something       async-first    proven,     │
│   effect."           simple."        + typed."      reliable."  │
│                                                                 │
│                                                                 │
│  Also in this space (not task queues):                          │
│  ├─ Redis Pub/Sub  → Event broadcasting (no persistence)       │
│  ├─ RabbitMQ       → Enterprise message broker                 │
│  └─ Kafka          → Distributed event streaming               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 FastAPI BackgroundTasks (Recap & Positioning)

**You learned this in Lecture 1. Here's where it fits.**

```python
from fastapi import BackgroundTasks

@app.post("/tasks/{task_id}/complete")
async def complete_task(
    task_id: int,
    background_tasks: BackgroundTasks,
):
    task = await do_complete(task_id)

    # Side effect runs AFTER response is sent
    background_tasks.add_task(send_notification, task.assignee_id)

    return task
```

```
┌─────────────────────────────────────────────────────────────────┐
│              FASTAPI BACKGROUNDTASKS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Strengths:                                                  │
│  ├─ Zero setup. No broker. No worker process.                  │
│  ├─ Built into FastAPI. Import and use.                        │
│  ├─ Perfect for "send email after responding to user."         │
│  └─ Async or sync functions both work.                         │
│                                                                 │
│  ❌ Weaknesses:                                                 │
│  ├─ NO persistence. If the process crashes, task is lost.      │
│  ├─ NO retry logic. Fails once → gone forever.                 │
│  ├─ NO scheduling. Cannot run "every Monday at 9am."           │
│  ├─ NO monitoring. No way to see pending/failed tasks.         │
│  ├─ Runs in the SAME process as your API.                      │
│  │   Heavy background work slows down your request handling.   │
│  └─ NO distributed execution. One server only.                 │
│                                                                 │
│  📍 Position: The "good enough for simple things" option.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Celery (Recap & Positioning)

**You've spent two lectures on Celery. Here's the honest assessment.**

```python
# What you've been writing:
@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def send_notification_email(self, user_id: int, title: str) -> None:
    try:
        user = get_user_sync(user_id)  # ← sync call
        send_email_sync(user.email, f"Task completed: {title}")
    except Exception as exc:
        self.retry(exc=exc)
```

```
┌─────────────────────────────────────────────────────────────────┐
│                        CELERY                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Strengths:                                                  │
│  ├─ Industry standard. Used at Instagram, Mozilla, Robinhood.  │
│  ├─ Battle-tested for 15+ years. Known failure modes.          │
│  ├─ Rich ecosystem: retry, scheduling (Beat), monitoring       │
│  │   (Flower), result backends, rate limiting.                 │
│  ├─ Multiple broker support (Redis, RabbitMQ, SQS).            │
│  ├─ Excellent documentation and community support.             │
│  └─ Tasks persist in broker until a worker picks them up.      │
│                                                                 │
│  ❌ Weaknesses:                                                 │
│  ├─ SYNC-FIRST ARCHITECTURE.                                   │
│  │   Workers run synchronous code by default.                  │
│  │   Running async code in Celery requires workarounds.        │
│  │   This is awkward when your ENTIRE APP is async.            │
│  ├─ Heavy setup: separate worker process, broker, Beat.        │
│  ├─ No type hints on task signatures (task.delay(x, y)         │
│  │   gives no autocomplete, no type checking).                 │
│  ├─ Task arguments must be serializable (JSON/pickle).         │
│  │   No Pydantic models, no SQLAlchemy objects.                │
│  └─ Configuration is complex (many settings to tune).          │
│                                                                 │
│  📍 Position: The safe choice. Never wrong. Sometimes clunky.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The async problem with Celery — why alternatives emerged:**

```python
# Your FastAPI app is async. Everything is async.
# Async endpoint → async SQLAlchemy → async Redis → async httpx

# But Celery tasks are sync:
@celery_app.task
def process_data(data_id: int):
    # To use async code here, you need workarounds:
    import asyncio
    result = asyncio.run(async_process(data_id))
    # This creates a NEW event loop per task.
    # It works, but it's inelegant and loses the benefits
    # of connection sharing, async context, etc.
```

> "Celery was designed before Python had async/await. It's sync at its core. Your entire application is async. This mismatch is real, and it's the reason tools like Taskiq and ARQ exist."

---

## 4.4 Taskiq — The Async-Native Alternative

**Taskiq is what you'd get if you redesigned Celery with async and type safety as first-class priorities.**

```
┌─────────────────────────────────────────────────────────────────┐
│                        TASKIQ                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CORE IDEA: Celery, but async-native and type-hinted.           │
│                                                                 │
│  • Tasks are async functions (no workarounds needed)            │
│  • Task parameters have type hints (IDE autocomplete works)     │
│  • Return types are preserved (type-checked results)            │
│  • FastAPI integration via taskiq-fastapi                       │
│  • Multiple broker support (Redis, RabbitMQ, Kafka, etc.)       │
│  • Middleware system (logging, metrics, error handling)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Setup and task definition:**

```python
# broker.py
from taskiq_redis import ListQueueBroker, RedisAsyncResultBackend

# Same Redis you're already running
result_backend = RedisAsyncResultBackend(
    redis_url="redis://localhost:6379/1",
)

broker = ListQueueBroker(
    url="redis://localhost:6379/0",
).with_result_backend(result_backend)
```

```python
# tasks.py
from broker import broker


@broker.task(retry_on_error=True, max_retries=3)
async def send_notification_email(user_id: int, title: str) -> bool:
    """
    Notice:
    1. This is an async function. Your async DB, Redis, httpx — all work.
    2. Parameters are type-hinted. IDE knows user_id is int.
    3. Return type is declared. Result is typed.
    """
    user = await async_get_user(user_id)    # Async DB call — natural
    await async_send_email(                  # Async HTTP call — natural
        to=user.email,
        subject=f"Task completed: {title}",
    )
    return True


@broker.task
async def generate_report(project_id: int) -> dict:
    async with get_async_session() as db:    # Async SQLAlchemy — natural
        stats = await compute_project_stats(db, project_id)
    return stats
```

**Calling tasks — compare with Celery:**

```python
# Celery — you've been doing this:
send_notification_email.delay(user_id=3, title="Fix bug")
#                       ^^^^^ No type hints. No autocomplete.
#                             IDE doesn't know the args.

# Taskiq — kiq (kick) instead of delay:
await send_notification_email.kiq(user_id=3, title="Fix bug")
#                              ^^^ Async call.
#                                  Type-checked by IDE.
#                                  Autocomplete works.
```

**Getting results:**

```python
# Celery:
result = send_notification_email.delay(3, "Fix bug")
value = result.get(timeout=10)  # Blocks the calling thread!

# Taskiq:
task_handle = await send_notification_email.kiq(user_id=3, title="Fix bug")
result = await task_handle.wait_result(timeout=10)  # Async wait!

if result.is_err:
    print(f"Task failed: {result.error}")
else:
    print(f"Task result: {result.return_value}")  # Typed as bool
```

**FastAPI integration:**

```python
# With taskiq-fastapi, tasks can access FastAPI dependencies
from taskiq_fastapi import InjectDependency
from app.dependencies import get_db, get_redis


@broker.task
async def process_with_deps(
    task_id: int,
    db: AsyncSession = InjectDependency(get_db),      # ← FastAPI dep!
    redis: Redis = InjectDependency(get_redis),        # ← FastAPI dep!
) -> None:
    """Task has access to the same dependencies as your endpoints."""
    task = await task_repo.get(db, task_id)
    await redis.delete(f"task:{task_id}")
```

**Running a Taskiq worker:**

```bash
# Very similar to Celery:
# Celery:  celery -A app.celery_app worker --loglevel=info
# Taskiq:  taskiq worker app.broker:broker
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ✅ Strengths:                                                  │
│  ├─ Async-native: tasks are async def, no workarounds.         │
│  ├─ Type-hinted: IDE autocomplete, type checking.              │
│  ├─ FastAPI integration: shared dependencies.                  │
│  ├─ Multiple brokers: Redis, RabbitMQ, Kafka, NATS.            │
│  └─ Middleware system for cross-cutting concerns.              │
│                                                                 │
│  ❌ Weaknesses:                                                 │
│  ├─ Younger ecosystem. Less battle-tested than Celery.         │
│  ├─ Smaller community. Fewer Stack Overflow answers.           │
│  ├─ Scheduling (cron) is less mature than Celery Beat.         │
│  └─ Less tooling (no Flower equivalent yet).                   │
│                                                                 │
│  📍 Position: The modern choice for async-first Python apps.    │
│     Growing fast. Production-ready. Not yet "default."          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 ARQ — The Lightweight Option

**ARQ is the minimalist's task queue. Redis-only, async-only, simple.**

Built by Samuel Colvin — the same person who created Pydantic.

```
┌─────────────────────────────────────────────────────────────────┐
│                          ARQ                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CORE IDEA: What if a task queue was just...                    │
│             async functions + Redis? That's it.                 │
│                                                                 │
│  • No broker abstraction. Redis IS the broker.                  │
│  • Tasks are plain async functions with a ctx parameter.        │
│  • Pessimistic execution: assumes tasks can fail.               │
│  • At-least-once delivery: guarantees processing.               │
│  • Built on Redis streams (not Pub/Sub — persistent).           │
│  • Cron-style scheduling built in.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Task definition — just write async functions:**

```python
# tasks.py

async def send_notification_email(
    ctx: dict,                 # ← ARQ always passes a context dict
    user_id: int,
    title: str,
) -> bool:
    """
    This is a plain async function. Not decorated.
    The ctx dict contains: redis connection, job metadata.
    """
    redis = ctx["redis"]  # ARQ provides a Redis connection
    user_data = await redis.get(f"user:{user_id}")
    # ... send email ...
    return True


async def generate_report(ctx: dict, project_id: int) -> dict:
    """Another plain async function."""
    # ... async work ...
    return {"project_id": project_id, "status": "done"}
```

**Worker configuration — a single class:**

```python
# worker.py
from arq import cron
from arq.connections import RedisSettings

from tasks import send_notification_email, generate_report


class WorkerSettings:
    """All worker config in one place."""

    # Register available task functions
    functions = [send_notification_email, generate_report]

    # Cron jobs (like Celery Beat, but built-in)
    cron_jobs = [
        cron(generate_report, hour=9, minute=0, project_id=1),  # Daily at 9am
    ]

    # Redis connection
    redis_settings = RedisSettings(
        host="localhost",
        port=6379,
        database=0,
    )

    # Timeouts
    max_jobs = 10           # Max concurrent jobs
    job_timeout = 300       # 5 minutes per job
    retry_jobs = True       # Retry failed jobs
    max_tries = 3           # Up to 3 attempts
```

**Enqueuing jobs:**

```python
from arq.connections import ArqRedis, create_pool, RedisSettings


async def enqueue_example() -> None:
    pool: ArqRedis = await create_pool(RedisSettings())

    # Enqueue a job — type-safe pool call
    job = await pool.enqueue_job(
        "send_notification_email",  # Function name as string
        user_id=3,
        title="Fix login bug",
    )

    print(f"Job ID: {job.job_id}")

    # Optionally wait for result
    result = await job.result(timeout=30)
    print(f"Result: {result}")
```

**Running the worker:**

```bash
arq worker.WorkerSettings
```

**The "pessimistic execution" model:**

```
┌─────────────────────────────────────────────────────────────────┐
│              PESSIMISTIC EXECUTION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ARQ assumes the worst:                                         │
│                                                                 │
│  "This task will probably fail. Let me prepare for that."       │
│                                                                 │
│  1. Job is enqueued → stored in Redis stream (persistent)       │
│  2. Worker picks up job → marks it as "in progress"             │
│  3. If worker crashes → job is NOT acknowledged                 │
│     → Redis knows it wasn't completed                          │
│     → Another worker can pick it up (at-least-once delivery)   │
│  4. If job succeeds → result stored, job marked complete        │
│  5. If job fails → retry according to max_tries                 │
│                                                                 │
│  Compare to Redis Pub/Sub:                                      │
│  Pub/Sub assumes the best: "Someone will hear this."           │
│  ARQ assumes the worst: "This might fail. Let me be safe."     │
│                                                                 │
│  "At-least-once" means: a job might run MORE than once          │
│  (if the worker crashes after completing but before             │
│  acknowledging). Your tasks should be IDEMPOTENT                │
│  (Lecture 2 callback — idempotent task design).                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ✅ Strengths:                                                  │
│  ├─ Dead simple. Minimal API, easy to understand.              │
│  ├─ Async-first. Tasks are plain async functions.              │
│  ├─ Built-in cron scheduling (no separate Beat process).       │
│  ├─ Persistent: uses Redis streams, not Pub/Sub.               │
│  ├─ At-least-once delivery. Jobs don't silently vanish.        │
│  └─ Built by the Pydantic creator (quality, thoughtful API).   │
│                                                                 │
│  ❌ Weaknesses:                                                 │
│  ├─ Redis-only. Cannot use RabbitMQ, SQS, or Kafka.           │
│  ├─ Smaller ecosystem than Celery (fewer plugins, middleware). │
│  ├─ Task name is a string at enqueue time (no autocomplete).   │
│  ├─ Less flexible routing (no priority queues, exchanges).     │
│  └─ Smaller community. Less documentation.                     │
│                                                                 │
│  📍 Position: The "simple and correct" option.                   │
│     When Celery is overkill and BackgroundTasks isn't enough.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.6 The Decision Framework

**Now the question that matters: WHICH ONE DO I USE?**

Evaluate along three axes:

```
┌─────────────────────────────────────────────────────────────────┐
│               THREE-AXIS DECISION FRAMEWORK                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AXIS 1: COMPLEXITY OF YOUR NEEDS                              │
│  ─────────────────────────────────                              │
│  Low:  "Run this after the response."                           │
│  Med:  "Retry on failure, schedule weekly."                     │
│  High: "Priority queues, rate limiting, complex routing,        │
│         multiple worker types, result chaining."                │
│                                                                 │
│  AXIS 2: ASYNC-NATIVENESS                                       │
│  ────────────────────────                                       │
│  Low:  "Doesn't matter, sync tasks are fine."                   │
│  Med:  "Nice to have, but I can work around it."                │
│  High: "My entire stack is async. I don't want sync anywhere." │
│                                                                 │
│  AXIS 3: ECOSYSTEM MATURITY                                     │
│  ──────────────────────────                                     │
│  Low:  "I'm comfortable being an early adopter."                │
│  Med:  "I want stable with decent docs."                        │
│  High: "I need stack overflow answers, corporate backing,       │
│         and a tool that's been in production for 10+ years."    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The decision matrix:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│                    Complexity    Async-    Ecosystem             │
│  Tool              of Needs     Native    Maturity              │
│  ────              ────────     ──────    ─────────             │
│  BackgroundTasks   Low          ✅ Yes    ✅ FastAPI built-in    │
│  ARQ               Low-Med      ✅ Yes    ⚠️  Small community    │
│  Taskiq            Med-High     ✅ Yes    ⚠️  Growing fast       │
│  Celery            Med-High     ❌ No     ✅ Industry standard   │
│                                                                 │
│                                                                 │
│  QUICK PICKS:                                                   │
│  ────────────                                                   │
│                                                                 │
│  "I just need a simple side effect after a response."           │
│  → FastAPI BackgroundTasks. Done.                               │
│                                                                 │
│  "I need retries, scheduling, and my app is fully async."       │
│  → Taskiq if you want typed + async.                            │
│  → ARQ if you want simple + async and only use Redis.           │
│                                                                 │
│  "I'm at a company. We need proven. We need Flower.             │
│   We need 50 Stack Overflow answers per problem."               │
│  → Celery. It's not sexy, but it's reliable.                    │
│                                                                 │
│  "I just need to broadcast events, not assign tasks."           │
│  → Redis Pub/Sub. Different tool, different purpose.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**And the visual decision tree:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     PICK YOUR TOOL                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      START HERE                                 │
│                          │                                      │
│                          ▼                                      │
│              ┌───────────────────────┐                           │
│              │ Do I need persistence │                           │
│              │ (survive crashes)?    │                           │
│              └───────────┬───────────┘                           │
│                   │            │                                │
│                  NO           YES                               │
│                   │            │                                │
│                   ▼            ▼                                │
│         ┌──────────────┐  ┌───────────────────┐                 │
│         │ Is it a task  │  │ Is your entire    │                 │
│         │ or an event?  │  │ stack async?      │                 │
│         └───────┬───────┘  └─────────┬─────────┘                │
│             │       │            │         │                    │
│           Task    Event        YES        NO                   │
│             │       │            │         │                    │
│             ▼       ▼            ▼         ▼                    │
│      Background  Redis       ┌──────┐  ┌──────┐                │
│        Tasks     Pub/Sub     │ Need │  │      │                │
│                              │complex│  │Celery│                │
│                              │features│  │      │                │
│                              └──┬───┘  └──────┘                │
│                              │     │                           │
│                            YES     NO                          │
│                              │     │                           │
│                              ▼     ▼                           │
│                           Taskiq  ARQ                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Side-by-side comparison — same task, four tools:**

```python
# ── THE TASK: Send a notification email with retry ──

# 1. FastAPI BackgroundTasks (no retry possible)
@app.post("/tasks/{task_id}/complete")
async def complete(task_id: int, bg: BackgroundTasks):
    task = await do_complete(task_id)
    bg.add_task(send_email, task.assignee_id, task.title)
    return task

# 2. Celery (sync task, sync email)
@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def send_email_task(self, user_id: int, title: str):
    try:
        send_email_sync(user_id, title)
    except Exception as exc:
        self.retry(exc=exc)

# 3. Taskiq (async task, async email)
@broker.task(retry_on_error=True, max_retries=3)
async def send_email_task(user_id: int, title: str) -> bool:
    await send_email_async(user_id, title)
    return True

# 4. ARQ (async function, retry via config)
async def send_email_task(ctx: dict, user_id: int, title: str):
    await send_email_async(user_id, title)
# retry configured in WorkerSettings: max_tries=3
```

---

# PART 5: COMMON MISTAKES AND MISCONCEPTIONS

### Mistake 1: Using Redis Pub/Sub when you need guaranteed delivery

```python
# ❌ DANGEROUS: Billing event via Pub/Sub
async def process_payment(order_id: int, redis: Redis):
    await charge_credit_card(order_id)

    # If NO subscriber is connected right now, this event is LOST.
    # The customer was charged, but the order is never marked as paid.
    await redis.publish("order.paid", json.dumps({"order_id": order_id}))

# ✅ CORRECT: Use a persistent task queue for critical events
await process_order_payment.kiq(order_id=order_id)  # Taskiq
# or
process_order_payment.delay(order_id)                # Celery
```

> "If losing the message means losing money, losing compliance, or losing data — it needs a persistent queue, not Pub/Sub."

---

### Mistake 2: Blocking the event loop inside a subscriber

```python
# ❌ WRONG: Blocking call inside async subscriber handler
async def on_task_completed_send_email(event: dict):
    user = requests.get(f"/api/users/{event['assignee_id']}")  # BLOCKS!
    # All other event handlers are frozen while this completes.
    # If you have 10 events queued, they ALL wait for this HTTP call.

# ✅ CORRECT: Use async libraries inside async handlers
async def on_task_completed_send_email(event: dict):
    async with httpx.AsyncClient() as client:                  # Non-blocking
        response = await client.get(f"/api/users/{event['assignee_id']}")
```

This is the same lesson from Week 1, Lecture 3 — blocking calls in async code freeze the event loop. Inside a pub/sub subscriber, this is especially damaging because it stalls ALL event processing, not just one request.

---

### Mistake 3: Unstructured events (no schema)

```python
# ❌ BAD: Different publishers send different shapes
# Publisher A:
await redis.publish("task.completed", json.dumps({"id": 42}))
# Publisher B:
await redis.publish("task.completed", json.dumps({"task_id": 42, "name": "Fix bug"}))
# Subscriber has no idea what fields to expect.

# ✅ GOOD: Enforce schema with Pydantic (you already know this)
class TaskCompletedEvent(BaseModel):
    task_id: int
    project_id: int
    assignee_id: int
    title: str

# Publisher:
event = TaskCompletedEvent(task_id=42, project_id=7, assignee_id=3, title="Fix bug")
await redis.publish("task.completed", event.model_dump_json())

# Subscriber:
event = TaskCompletedEvent.model_validate_json(raw_data)
# Now you KNOW what fields exist. Pydantic validates it.
```

---

### Mistake 4: Over-engineering (Kafka for 100 events/day)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Your traffic:     100 events / day                             │
│  Kafka designed:   millions of events / second                  │
│                                                                 │
│  Kafka requires:                                                │
│  ├─ ZooKeeper (or KRaft) cluster                               │
│  ├─ Broker cluster (minimum 3 nodes for replication)           │
│  ├─ Schema registry                                            │
│  ├─ Ops knowledge to manage partitions, replication, ISR       │
│  └─ At least one person whose job is "Kafka admin"             │
│                                                                 │
│  Redis Pub/Sub + Celery requires:                               │
│  ├─ Redis (you already have it)                                │
│  └─ That's it.                                                 │
│                                                                 │
│  Don't build infrastructure for problems you don't have.        │
│  Scale is a GOOD problem. Solve it when it arrives.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Mistake 5: Confusing task queues with event streams

```
┌─────────────────────────────────────────────────────────────────┐
│          TASK QUEUE ≠ EVENT STREAM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TASK QUEUE (Celery, Taskiq, ARQ):                              │
│  ─────────                                                      │
│  "Hey worker, DO this specific thing."                          │
│  One task → one worker processes it → done.                     │
│  The task is CONSUMED. No one else sees it.                     │
│                                                                 │
│  Analogy: Postal letter → one recipient.                        │
│                                                                 │
│                                                                 │
│  EVENT STREAM (Pub/Sub, Kafka):                                 │
│  ────────────                                                   │
│  "Hey everyone, this thing HAPPENED."                           │
│  One event → ALL subscribers see it → each reacts differently.  │
│  The event is BROADCAST, not consumed by one.                   │
│                                                                 │
│  Analogy: Radio broadcast → all listeners.                      │
│                                                                 │
│                                                                 │
│  COMMON MISTAKE:                                                │
│  Using Pub/Sub to distribute tasks → multiple subscribers       │
│  all process the same task → duplicate work.                    │
│                                                                 │
│  Using a task queue to broadcast events → only one worker       │
│  processes the event → other systems never learn about it.      │
│                                                                 │
│  Use the right tool for the communication pattern.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│               EVENT-DRIVEN QUICK REFERENCE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PUBLISH EVENT (Redis Pub/Sub):                                 │
│      await redis.publish("channel.name", json_string)           │
│                                                                 │
│  SUBSCRIBE TO EVENTS:                                           │
│      pubsub = redis.pubsub()                                    │
│      await pubsub.subscribe("channel.name")                     │
│      async for msg in pubsub.listen():                          │
│          if msg["type"] == "message": ...                       │
│                                                                 │
│  PATTERN SUBSCRIBE:                                             │
│      await pubsub.psubscribe("task.*")                          │
│                                                                 │
│                                                                 │
│  TOOL CHEAT SHEET:                                              │
│                                                                 │
│  "Simple side effect, no retry"                                 │
│      → FastAPI BackgroundTasks                                  │
│                                                                 │
│  "Persistent tasks, proven ecosystem, sync OK"                  │
│      → Celery                                                   │
│                                                                 │
│  "Persistent tasks, async-native, typed"                        │
│      → Taskiq                                                   │
│                                                                 │
│  "Persistent tasks, async, Redis-only, simple"                  │
│      → ARQ                                                      │
│                                                                 │
│  "Broadcast events, no persistence needed"                      │
│      → Redis Pub/Sub                                            │
│                                                                 │
│  "Broadcast events, persistence + replay needed"                │
│      → Kafka (but do you really need it?)                       │
│                                                                 │
│                                                                 │
│  KEY PATTERNS:                                                  │
│  ├─ Pub/Sub: decouple producers from consumers                 │
│  ├─ Event Sourcing: store events, derive state                 │
│  ├─ CQRS: separate read model from write model                 │
│  └─ Commands vs Events: "do this" vs "this happened"           │
│                                                                 │
│  CRITICAL RULE:                                                 │
│  If losing a message has business consequences,                 │
│  do NOT use Redis Pub/Sub. Use a persistent queue.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  EVENT-DRIVEN = DON'T CALL. ANNOUNCE.                           │
│                                                                 │
│  Instead of your code calling every system that cares,          │
│  your code announces what happened. Systems that care           │
│  subscribe and handle it themselves.                            │
│                                                                 │
│                                                                 │
│  ┌───────────┐  "task done"  ┌───────────┐                      │
│  │ Your Code │ ────────────▶ │ Event Bus │                      │
│  │           │               │ (broker)  │                      │
│  └───────────┘               └─────┬─────┘                      │
│                                    │                            │
│                    ┌───────────────┼───────────────┐            │
│                    │               │               │            │
│                    ▼               ▼               ▼            │
│              ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│              │  Email   │   │  Cache   │   │  Audit   │         │
│              │  Handler │   │  Handler │   │  Handler │         │
│              └──────────┘   └──────────┘   └──────────┘         │
│                                                                 │
│  THE RADIO STATION ANALOGY:                                     │
│  ├─ Commands = Postal service ("you, specifically, do this")    │
│  ├─ Events = Radio broadcast ("this happened, FYI")             │
│  ├─ Pub/Sub = Live radio (miss it, it's gone)                   │
│  ├─ Kafka = Newspaper archive (read any past issue)             │
│  ├─ Celery = Postal service (guaranteed delivery, directed)     │
│  └─ Taskiq/ARQ = Modern postal (same delivery, async staff)     │
│                                                                 │
│  THERE IS NO "BEST" TOOL. THERE IS ONLY "RIGHT FOR THE JOB."  │
│  Match the tool to the problem, not the other way around.       │
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
│  WEEK 11 PROJECT (This week):                                   │
│  └─ Background Processing Pipeline                              │
│     You'll implement event-triggered notifications,             │
│     scheduled jobs, and choose between task queue tools.        │
│     The decision framework from 4.6 is your guide.             │
│                                                                 │
│  WEEK 12: WebSockets & Real-Time                                │
│  └─ Redis Pub/Sub becomes your backbone for scaling WebSockets. │
│     When a task is completed on Server A, the event goes        │
│     through Pub/Sub to Server B which has the WebSocket         │
│     connection to the user. This pattern is EVERYWHERE.         │
│                                                                 │
│  WEEK 12: Performance & Load Testing                            │
│  └─ You'll measure the impact of moving side effects out        │
│     of the request path. Response times drop dramatically       │
│     when your endpoint publishes an event instead of doing      │
│     6 side effects synchronously.                               │
│                                                                 │
│  WEEK 13-14: Capstone SaaS Platform                             │
│  └─ Your capstone requires real-time notifications              │
│     (WebSocket + Pub/Sub), background jobs (Celery or           │
│     Taskiq), and audit logging. Everything from today           │
│     is load-bearing architecture in your final project.         │
│                                                                 │
│  WEEK 16: System Design                                         │
│  └─ Event-driven architecture, CQRS, and event sourcing        │
│     are staple topics in system design interviews.              │
│     Today gave you the vocabulary and mental models.            │
│     Week 16 teaches you how to draw them on a whiteboard.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```