# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PAIN FIRST, ARCHITECTURE SECOND                                │
│  ───────────────────────────────                                │
│  Students must see a realistic endpoint buckle under work       │
│  that should never have been there. Feel the latency.           │
│  Then — and only then — we introduce the escape hatch.          │
│                                                                 │
│  CONCEPT BEFORE TOOL                                            │
│  ───────────────────                                            │
│  This lecture teaches ZERO new libraries. No Celery, no         │
│  Flower, no Beat. Those are Lectures 2-3. Today we build       │
│  the mental model so thoroughly that the tools become obvious.  │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  A pizza shop analogy carries every concept. Cashier, ticket    │
│  board, kitchen staff, order number — each maps to a real       │
│  component students will implement next lecture.                │
│                                                                 │
│  BUILD ON WHAT THEY KNOW                                        │
│  ───────────────────────                                        │
│  Students already know async, Redis data structures, and        │
│  FastAPI. We use ALL of these to demystify "message broker"     │
│  into something they can build with 15 lines of code.           │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Async (Week 1) → Why async alone doesn't solve this problem   │
│  Redis (Week 10) → Redis lists ARE a job queue, literally      │
│  External APIs (Week 8) → Retry patterns reappear in workers   │
│  BackgroundTasks (Week 8) → The bridge from "quick fix" to     │
│    "real architecture"                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                BACKGROUND PROCESSING CONCEPTS                   │
│                     (3-3.5 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 The Overworked Endpoint (Demonstration)                 │
│  ├─ 1.2 Async Didn't Solve This                                 │
│  ├─ 1.3 Three Reasons to Go Background                          │
│  └─ 1.4 The Pizza Shop Analogy                                  │
│                                                                 │
│  PART 2: FASTAPI BACKGROUNDTASKS (35 min)                       │
│  ├─ 2.1 The Quick Fix                                           │
│  ├─ 2.2 How BackgroundTasks Works Under the Hood                │
│  ├─ 2.3 The Ceiling (Five Things It Cannot Do)                  │
│  └─ 2.4 When BackgroundTasks Is the Right Choice                │
│                                                                 │
│  PART 3: THE MESSAGE BROKER PATTERN (50 min)                    │
│  ├─ 3.1 Producer, Broker, Consumer                              │
│  ├─ 3.2 Why the Broker Exists                                   │
│  ├─ 3.3 Building a Tiny Queue with Redis (You Already Know How) │
│  ├─ 3.4 Job Lifecycle and States                                │
│  └─ 3.5 Workers: The Other Side of the Wall                     │
│                                                                 │
│  PART 4: JOB QUEUE VS PUB/SUB (30 min)                          │
│  ├─ 4.1 Two Patterns, Two Problems                              │
│  ├─ 4.2 Job Queue: "Make This Pizza"                            │
│  ├─ 4.3 Pub/Sub: "Order #47 Is Ready!"                         │
│  └─ 4.4 The Decision Framework                                  │
│                                                                 │
│  PART 5: EVENT-DRIVEN ARCHITECTURE OVERVIEW (25 min)            │
│  ├─ 5.1 From Commands to Events                                 │
│  ├─ 5.2 The Architecture Spectrum                               │
│  └─ 5.3 Where This Leads (Preview of Lecture 4)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Overworked Endpoint

**Start with a demonstration. A realistic FastAPI endpoint they'd actually write.**

```python
# demo_overworked.py — Run this with students watching
import asyncio
import time
from fastapi import FastAPI, Depends
from pydantic import BaseModel

app = FastAPI()

# --- Simulated services (each simulates real I/O latency) ---

async def get_order_from_db(order_id: int) -> dict:
    """Simulate database read"""
    await asyncio.sleep(0.05)  # 50ms — typical DB query
    return {"id": order_id, "user_email": "customer@example.com", "total": 99.99}

async def update_order_status(order_id: int, status: str) -> None:
    """Simulate database write"""
    await asyncio.sleep(0.05)  # 50ms — typical DB write

async def send_confirmation_email(email: str, order: dict) -> None:
    """Simulate SMTP server — sending an email"""
    await asyncio.sleep(3.0)  # 3 seconds — real SMTP latency

async def generate_invoice_pdf(order: dict) -> bytes:
    """Simulate PDF generation"""
    await asyncio.sleep(2.0)  # 2 seconds — rendering a PDF
    return b"%PDF-fake-content"

async def notify_warehouse_api(order: dict) -> None:
    """Simulate calling warehouse external API"""
    await asyncio.sleep(1.0)  # 1 second — external API call

async def update_analytics(order: dict) -> None:
    """Simulate pushing to analytics service"""
    await asyncio.sleep(0.5)  # 500ms — analytics ingestion


# --- The endpoint ---

@app.post("/orders/{order_id}/confirm")
async def confirm_order(order_id: int):
    start = time.time()

    # Step 1: Business logic (what the user NEEDS)
    order = await get_order_from_db(order_id)
    await update_order_status(order_id, "confirmed")

    # Step 2: Side effects (what the user DOESN'T need to wait for)
    await send_confirmation_email(order["user_email"], order)
    await generate_invoice_pdf(order)
    await notify_warehouse_api(order)
    await update_analytics(order)

    elapsed = time.time() - start
    print(f"Response time: {elapsed:.1f}s")

    return {"status": "confirmed", "order_id": order_id}
```

**Run it. Hit the endpoint. Watch the clock.**

```
$ curl -X POST http://localhost:8000/orders/42/confirm
```

```
Response time: 6.6s

{"status":"confirmed","order_id":42}
```

**Now ask the class:**

> "The order was confirmed in the database at 100 milliseconds. The customer stared at a loading spinner for 6.6 seconds. What were they waiting for?"

**Draw the timeline:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHAT THE USER WAITED FOR                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Time: 0ms    100ms                                   6600ms    │
│        │       │                                         │      │
│        │ DB ── │ ── Email (3s) ── PDF (2s) ── WH (1s) ──│      │
│        │read+  │                                  Anlytcs│      │
│        │write  │                                  (0.5s) │      │
│        │       │                                         │      │
│        ▼       ▼                                         ▼      │
│     Request  ORDER IS                              Response     │
│     arrives  CONFIRMED                             sent         │
│              HERE                                               │
│                                                                 │
│       ◀─ 100ms ─▶◀──────── 6500ms of work the ────────▶       │
│        User NEEDS    user does NOT need to see                  │
│         this part                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The punchline:**

> "The user needed ONE thing: confirmation that their order status changed. That took 100ms. The other 6.5 seconds? Email, PDF, warehouse notification, analytics — the user doesn't see any of that. They don't need it before the response. So why are we making them wait?"

---

## 1.2 Async Didn't Solve This

**Students will immediately think: "But we can use gather!"**

Let them. Show them.

```python
@app.post("/orders/{order_id}/confirm")
async def confirm_order_v2(order_id: int):
    start = time.time()

    order = await get_order_from_db(order_id)
    await update_order_status(order_id, "confirmed")

    # "Let's run them all concurrently!"
    await asyncio.gather(
        send_confirmation_email(order["user_email"], order),  # 3s ─┐
        generate_invoice_pdf(order),                          # 2s  │
        notify_warehouse_api(order),                          # 1s  ├─ 3s total
        update_analytics(order),                              # 0.5s┘
    )

    elapsed = time.time() - start
    print(f"Response time: {elapsed:.1f}s")  # 3.1 seconds

    return {"status": "confirmed", "order_id": order_id}
```

```
Response time: 3.1s
```

**Better. Still bad.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  GATHER HELPED, BUT...                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE (sequential):                                           │
│                                                                 │
│  [DB 100ms][Email 3s][PDF 2s][Warehouse 1s][Analytics 500ms]   │
│  Total: 6.6 seconds                                             │
│                                                                 │
│                                                                 │
│  AFTER (gather):                                                │
│                                                                 │
│  [DB 100ms][ Email 3s─────── ]                                 │
│            [ PDF 2s────  ]                                      │
│            [ WH 1s  ]                                           │
│            [ An 0.5s]                                           │
│  Total: 3.1 seconds (limited by slowest: email)                │
│                                                                 │
│                                                                 │
│  BUT THE USER STILL WAITS 3 SECONDS FOR NOTHING.               │
│  The order was confirmed at 100ms. Everything after that        │
│  is work the user doesn't need in the response.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**This is the critical distinction to nail:**

```
┌─────────────────────────────────────────────────────────────────┐
│            ASYNC ≠ BACKGROUND PROCESSING                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ASYNC (what you learned in Week 1):                            │
│  ────────────────────────────────────                           │
│  "Do multiple I/O tasks concurrently WITHIN a request."         │
│                                                                 │
│  The user STILL WAITS for all tasks to finish before            │
│  getting a response. Other users' requests aren't blocked,      │
│  but THIS user's response is held until everything completes.   │
│                                                                 │
│  Request ──▶ [do all the work concurrently] ──▶ Response        │
│              ◀──── user waits here ────────▶                    │
│                                                                 │
│                                                                 │
│  BACKGROUND PROCESSING (what we're learning today):             │
│  ──────────────────────────────────────────────────             │
│  "Do work OUTSIDE the request-response cycle entirely."         │
│                                                                 │
│  The user gets an IMMEDIATE response. The work happens          │
│  after, separately, potentially on a different machine.         │
│                                                                 │
│  Request ──▶ [core logic only] ──▶ Response (instant!)          │
│                     │                                           │
│                     └──▶ [email, PDF, etc.] (happens later)    │
│                          user is GONE, doesn't wait             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Async made your waiter efficient — they serve multiple tables without standing idle. Background processing is a completely different idea: some tasks should leave the dining room entirely and happen in the kitchen, after the customer has left."

---

## 1.3 Three Reasons to Go Background

**Not all background work is the same. There are three distinct motivations:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THREE REASONS TO GO BACKGROUND                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. DECOUPLED WORK (most common in web apps)                    │
│  ───────────────────────────────────────────                    │
│  Work that SHOULD happen, but the user doesn't need to          │
│  wait for it. It's a side effect of the user's action.          │
│                                                                 │
│  Examples:                                                      │
│  • Send confirmation email after order                          │
│  • Notify warehouse after purchase                              │
│  • Push event to analytics service                              │
│  • Resize uploaded image into thumbnails                        │
│  • Invalidate CDN cache after content update                    │
│                                                                 │
│  Pattern: User does X → respond immediately → do Y, Z later    │
│                                                                 │
│                                                                 │
│  2. LONG-RUNNING WORK                                           │
│  ────────────────────                                           │
│  Work that takes minutes, hours, or longer. No HTTP request     │
│  should stay open that long.                                    │
│                                                                 │
│  Examples:                                                      │
│  • Generate monthly financial report (5 min)                    │
│  • Export 500,000 rows to CSV (3 min)                           │
│  • Process uploaded video (10 min)                              │
│  • Retrain ML model on new data (1 hour)                        │
│  • Migrate user data between systems (30 min)                   │
│                                                                 │
│  Pattern: User requests X → "We're working on it" → poll/       │
│           notify when done                                      │
│                                                                 │
│                                                                 │
│  3. SCHEDULED WORK                                              │
│  ────────────────                                               │
│  Work that must happen at specific times or intervals.          │
│  No user triggers it at all. The clock triggers it.             │
│                                                                 │
│  Examples:                                                      │
│  • Daily database cleanup of expired sessions                   │
│  • Hourly sync from external data source                        │
│  • Weekly digest email to all users                             │
│  • Every 5 minutes: check health of external services           │
│  • Monthly: generate invoices for all subscribers               │
│                                                                 │
│  Pattern: Clock ticks → work starts → no user involved          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "Look at our broken endpoint. Which category does each task fall into?"

```
send_confirmation_email()  →  Decoupled (side effect of order)
generate_invoice_pdf()     →  Decoupled (side effect, maybe long-running)
notify_warehouse_api()     →  Decoupled (side effect of order)
update_analytics()         →  Decoupled (side effect of order)
```

> "All decoupled. None of these need to finish before we respond. Now imagine we also need to generate a monthly sales report for the admin dashboard. That's long-running AND scheduled. Background processing handles all three."

---

## 1.4 The Pizza Shop Analogy

**This analogy will carry us through the entire lecture and into Lectures 2-4.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE PIZZA SHOP ANALOGY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IMAGINE: You walk into a pizza shop and order a pizza.         │
│                                                                 │
│                                                                 │
│  BAD PIZZA SHOP (inline processing):                            │
│  ────────────────────────────────────                           │
│                                                                 │
│  1. You place your order at the counter                         │
│  2. The cashier walks to the kitchen                            │
│  3. The cashier MAKES the pizza themselves (20 min)             │
│  4. The cashier walks back to the counter                       │
│  5. The cashier hands you the pizza                             │
│  6. NOW the next customer can order                             │
│                                                                 │
│  Result: 3 customers = 60+ minutes. Line out the door. 😠       │
│                                                                 │
│                                                                 │
│  GOOD PIZZA SHOP (background processing):                       │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  1. You place your order at the counter                         │
│  2. Cashier writes order ticket, clips it to the TICKET RAIL   │
│  3. Cashier says: "Order #47! Ready in 20 min." (immediate!)   │
│  4. You sit down (or leave and come back)                       │
│  5. Kitchen staff PULLS ticket from rail, makes pizza           │
│  6. Counter calls: "#47, your pizza's ready!"                   │
│  7. Meanwhile, cashier has served 15 more customers             │
│                                                                 │
│  Result: Everyone served fast. Kitchen works in parallel. 😊    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to software:**

```
┌─────────────────────────────────────────────────────────────────┐
│               PIZZA SHOP → SOFTWARE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pizza Shop                │  Backend System                    │
│  ──────────────────────────│───────────────────────────────     │
│  Customer                  │  Client (browser, mobile app)      │
│  Cashier at counter        │  FastAPI endpoint                  │
│  "Order confirmed! #47"    │  HTTP 202 Accepted response        │
│  Order ticket              │  Job message (JSON payload)        │
│  Ticket rail               │  Message broker (Redis, RabbitMQ)  │
│  Kitchen staff             │  Worker processes                  │
│  One cook per pizza        │  One worker per job                │
│  Order number (#47)        │  Job ID (UUID)                     │
│  "Is my order ready?"      │  GET /jobs/{id}/status             │
│  "#47, pizza's ready!"     │  WebSocket notification / callback │
│  Adding more cooks         │  Scaling workers horizontally      │
│  Rush hour (20 orders/min) │  Traffic spike (queue absorbs it)  │
│  Burnt pizza → remake it   │  Failed job → retry                │
│  Morning prep (pre-open)   │  Scheduled tasks (cron jobs)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The crucial insight from the analogy:**

> "The cashier and the kitchen are SEPARATE. The cashier doesn't cook. The kitchen doesn't take orders. They communicate through the ticket rail. This decoupling is the entire point. In your system, the API endpoint and the background worker are separate processes — possibly on separate machines — communicating through a message queue."

---

# PART 2: FASTAPI BACKGROUNDTASKS

## 2.1 The Quick Fix

**You've seen BackgroundTasks briefly in Week 8. Let's use it properly.**

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()


# --- These functions do the actual work ---

async def send_confirmation_email(email: str, order: dict) -> None:
    """Simulate sending email — takes 3 seconds"""
    await asyncio.sleep(3.0)
    print(f"✉️  Email sent to {email} for order #{order['id']}")

async def generate_invoice_pdf(order: dict) -> None:
    """Simulate PDF generation — takes 2 seconds"""
    await asyncio.sleep(2.0)
    print(f"📄 Invoice PDF generated for order #{order['id']}")

async def notify_warehouse_api(order: dict) -> None:
    """Simulate warehouse notification — takes 1 second"""
    await asyncio.sleep(1.0)
    print(f"🏭 Warehouse notified for order #{order['id']}")

async def update_analytics(order: dict) -> None:
    """Simulate analytics push — takes 500ms"""
    await asyncio.sleep(0.5)
    print(f"📊 Analytics updated for order #{order['id']}")


# --- The FIXED endpoint ---

@app.post("/orders/{order_id}/confirm")
async def confirm_order(order_id: int, background_tasks: BackgroundTasks):
    start = time.time()

    # Step 1: The actual business logic
    order = await get_order_from_db(order_id)
    await update_order_status(order_id, "confirmed")

    # Step 2: Schedule side effects to run AFTER the response
    background_tasks.add_task(send_confirmation_email, order["user_email"], order)
    background_tasks.add_task(generate_invoice_pdf, order)
    background_tasks.add_task(notify_warehouse_api, order)
    background_tasks.add_task(update_analytics, order)

    elapsed = time.time() - start
    print(f"Response time: {elapsed:.2f}s")

    return {"status": "confirmed", "order_id": order_id}
```

**Run it. Hit the endpoint.**

```
Response time: 0.10s

{"status":"confirmed","order_id":42}

# Then, in the server logs AFTER the response:
✉️  Email sent to customer@example.com for order #42
📄 Invoice PDF generated for order #42
🏭 Warehouse notified for order #42
📊 Analytics updated for order #42
```

**0.10 seconds. Down from 6.6 seconds.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE DIFFERENCE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE (inline):                                               │
│                                                                 │
│  Request ▶ [DB][Email 3s][PDF 2s][WH 1s][An 0.5s] ▶ Response   │
│  User waits: ◀──────────── 6.6 seconds ──────────▶             │
│                                                                 │
│                                                                 │
│  AFTER (BackgroundTasks):                                       │
│                                                                 │
│  Request ▶ [DB] ▶ Response                                      │
│  User waits: ◀ 100ms ▶                                          │
│                  │                                               │
│                  └──▶ [Email 3s][PDF 2s][WH 1s][An 0.5s]       │
│                       (runs after response is sent)             │
│                       (user is already gone)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The cashier stopped trying to cook. They write the ticket and hand it off. The customer gets their order number in seconds."

---

## 2.2 How BackgroundTasks Works Under the Hood

**BackgroundTasks isn't magic. It's barely even infrastructure.**

```
┌─────────────────────────────────────────────────────────────────┐
│             BackgroundTasks INTERNALS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. You call add_task(func, *args)                              │
│     └─ Starlette appends (func, args) to an internal list      │
│                                                                 │
│  2. Your endpoint returns the response                          │
│     └─ FastAPI sends the HTTP response to the client            │
│                                                                 │
│  3. AFTER the response is sent, Starlette iterates the list     │
│     └─ For each task:                                           │
│        ├─ If async def → await it on the current event loop     │
│        └─ If regular def → run it in a thread pool              │
│                                                                 │
│  4. Tasks run SEQUENTIALLY (one after another, not gathered)    │
│     └─ Task 1 finishes, then Task 2 starts, then Task 3...     │
│                                                                 │
│                                                                 │
│  THERE IS NO:                                                   │
│  • Separate process        (same process as your API)           │
│  • Separate thread pool    (shares the API's resources)         │
│  • Queue on disk or Redis  (just a Python list in memory)       │
│  • Retry mechanism         (if it fails, it's gone)             │
│  • Monitoring dashboard    (you have no visibility)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize where the tasks live:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ┌──────────────── FastAPI Process (PID 1234) ───────────────┐  │
│  │                                                           │  │
│  │   Event Loop                                              │  │
│  │   ┌─────────────────────────────────────────────────┐     │  │
│  │   │                                                 │     │  │
│  │   │  Handling request ──▶ Sending response ──▶      │     │  │
│  │   │                                                 │     │  │
│  │   │  ──▶ Running background task 1 ──▶              │     │  │
│  │   │  ──▶ Running background task 2 ──▶              │     │  │
│  │   │  ──▶ Running background task 3 ──▶              │     │  │
│  │   │                                                 │     │  │
│  │   │  ──▶ Handling NEXT request ──▶ ...              │     │  │
│  │   │                                                 │     │  │
│  │   └─────────────────────────────────────────────────┘     │  │
│  │                                                           │  │
│  │   ⚠️  Everything is in ONE process, on ONE event loop.    │  │
│  │   Background tasks compete with incoming requests.        │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "BackgroundTasks is the cashier writing the order on a sticky note and putting it on the back of the cash register. If the cashier goes home? The sticky note is gone. If the shop gets busy? The cashier still has to look at those sticky notes between customers. It works for small things, but it's not a real kitchen system."

---

## 2.3 The Ceiling (Five Things It Cannot Do)

**BackgroundTasks has a hard ceiling. Here are five walls you'll hit:**

```
┌─────────────────────────────────────────────────────────────────┐
│               LIMITATION 1: NO DURABILITY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Scenario: User confirms order. Background email task starts.   │
│  2 seconds in... your server crashes (deploy, OOM, exception).  │
│                                                                 │
│  ┌──────────────────────────────────┐                           │
│  │ FastAPI Process                  │                           │
│  │                                  │                           │
│  │ Background: send_email(order)    │                           │
│  │             ██████░░░░░░ 60%     │                           │
│  │                                  │                           │
│  │          💥 CRASH / RESTART 💥    │                           │
│  │                                  │                           │
│  └──────────────────────────────────┘                           │
│                                                                 │
│  Email task was in MEMORY. Memory is gone. Task is gone.        │
│  Customer never gets their confirmation email.                  │
│  Nobody knows it failed. Nobody retries it.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│               LIMITATION 2: NO DISTRIBUTION                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In production, you run multiple API instances (you'll learn    │
│  this in Week 15 with load balancers).                          │
│                                                                 │
│      Load Balancer                                              │
│       ┌───┤├───┐                                                │
│       │        │                                                │
│       ▼        ▼                                                │
│  ┌─────────┐ ┌─────────┐                                       │
│  │ API #1  │ │ API #2  │                                       │
│  │ (busy!) │ │ (idle)  │                                       │
│  │ running │ │         │                                       │
│  │ 15 bg   │ │ 0 bg    │                                       │
│  │ tasks   │ │ tasks   │                                       │
│  └─────────┘ └─────────┘                                       │
│                                                                 │
│  All background tasks are stuck on the instance that received   │
│  the request. API #2 is idle but can't help.                    │
│  BackgroundTasks can't distribute work across machines.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│               LIMITATION 3: NO RETRY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  What if the email SMTP server is temporarily down?             │
│                                                                 │
│  async def send_confirmation_email(email, order):               │
│      response = await smtp_client.send(...)                     │
│      # ❌ ConnectionError: SMTP server unavailable              │
│      # Exception is logged... and that's it.                    │
│      # No retry. No backoff. The email is lost forever.         │
│                                                                 │
│  You COULD wrap it in try/except with manual retry logic...     │
│  but you'd be reinventing a job queue. Badly.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│               LIMITATION 4: NO SCHEDULING                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BackgroundTasks runs in response to a REQUEST.                 │
│  No request → no task.                                          │
│                                                                 │
│  You can't say:                                                 │
│  • "Run this every day at 3am"                                  │
│  • "Run this every 5 minutes"                                   │
│  • "Run this on the first of each month"                        │
│                                                                 │
│  Scheduled work has no trigger HTTP request. There's no         │
│  customer walking in — the kitchen just needs to prep           │
│  dough every morning before opening.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│               LIMITATION 5: NO VISIBILITY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  With BackgroundTasks, you cannot answer:                       │
│                                                                 │
│  • How many tasks are currently running?                        │
│  • How many have failed in the last hour?                       │
│  • What's the average completion time?                          │
│  • Is the email task for order #42 done yet?                    │
│  • Which tasks are stuck or taking too long?                    │
│                                                                 │
│  There's no dashboard, no API, no status tracking.              │
│  Tasks run in a void. You have server logs, if you wrote them.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Summary table:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Feature              BackgroundTasks         Real Job Queue    │
│  ────────────────     ───────────────         ──────────────    │
│  Survives restart     ❌ No (in memory)       ✅ Yes (broker)   │
│  Multi-machine        ❌ No (same process)    ✅ Yes (workers)  │
│  Retry on failure     ❌ No (fire-forget)     ✅ Yes (config)   │
│  Scheduled tasks      ❌ No (needs trigger)   ✅ Yes (cron)     │
│  Monitoring           ❌ No (invisible)       ✅ Yes (dashboard)│
│  Status tracking      ❌ No                   ✅ Yes (job ID)   │
│  Setup complexity     ✅ Zero (built in)      ❌ Needs infra    │
│  Dependencies         ✅ None                 ❌ Redis/RabbitMQ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 When BackgroundTasks Is the Right Choice

**BackgroundTasks is not obsolete. It's the right tool for the right job.**

```
┌─────────────────────────────────────────────────────────────────┐
│              USE BackgroundTasks WHEN:                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ The task is QUICK (< 30 seconds)                            │
│  ✅ Losing the task is ACCEPTABLE (non-critical)                │
│  ✅ No retry needed (best-effort is fine)                       │
│  ✅ You're running a SINGLE server instance                     │
│  ✅ You don't need to track progress or status                  │
│                                                                 │
│  Good examples:                                                 │
│  • Writing a log entry to a file                                │
│  • Pushing a non-critical metric to analytics                   │
│  • Sending a non-essential webhook                              │
│  • Cleaning up a temporary file after download                  │
│  • Updating a "last_seen" timestamp for a user                  │
│                                                                 │
│                                                                 │
│              USE A REAL JOB QUEUE WHEN:                         │
│  ───────────────────────────────────────                        │
│                                                                 │
│  ✅ The task MUST complete (emails, payments, notifications)    │
│  ✅ The task takes minutes or hours                              │
│  ✅ You need retry logic (external services fail)               │
│  ✅ You have multiple server instances                           │
│  ✅ You need scheduling (cron-like)                              │
│  ✅ You need to track task status and results                    │
│                                                                 │
│  Examples:                                                      │
│  • Sending transactional emails (order confirmation)            │
│  • Processing payments                                          │
│  • Generating reports (PDF, CSV exports)                        │
│  • Syncing data with third-party systems                        │
│  • Image/video processing                                       │
│  • Scheduled database maintenance                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "BackgroundTasks is the sticky note on the back of the cash register. Good for 'remember to wipe down Table 3.' Not good for 'make 200 pizzas for tomorrow's catering order.' For real work, you need a real kitchen system."

---

# PART 3: THE MESSAGE BROKER PATTERN

## 3.1 Producer, Broker, Consumer

**The fundamental pattern behind every job queue system in existence:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE MESSAGE BROKER PATTERN                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  ┌──────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │          │     │              │     │              │        │
│  │ PRODUCER │────▶│    BROKER    │────▶│   CONSUMER   │        │
│  │          │     │              │     │   (Worker)   │        │
│  └──────────┘     └──────────────┘     └──────────────┘        │
│                                                                 │
│  "I have         "I'll hold it      "I'll take the             │
│   work to do"     until someone      next job and               │
│                   picks it up"       do the work"               │
│                                                                 │
│                                                                 │
│  Pizza Shop:                                                    │
│  ┌──────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │          │     │              │     │              │        │
│  │ CASHIER  │────▶│ TICKET RAIL  │────▶│ KITCHEN COOK │        │
│  │          │     │              │     │              │        │
│  └──────────┘     └──────────────┘     └──────────────┘        │
│                                                                 │
│                                                                 │
│  Your System:                                                   │
│  ┌──────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │ FastAPI  │     │    Redis     │     │   Worker     │        │
│  │ endpoint │────▶│    (queue)   │────▶│  process     │        │
│  │          │     │              │     │              │        │
│  └──────────┘     └──────────────┘     └──────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Each component has exactly one responsibility:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  PRODUCER (your FastAPI app):                                   │
│  ─────────────────────────────                                  │
│  Creates a job message and puts it on the queue.                │
│  Doesn't know WHO will process it.                              │
│  Doesn't know WHEN it will be processed.                        │
│  Doesn't CARE. Its job is done once the message is queued.      │
│                                                                 │
│  BROKER (Redis, RabbitMQ, etc.):                                │
│  ─────────────────────────────────                              │
│  Holds messages durably until a consumer is ready.              │
│  Guarantees ordering (FIFO by default).                         │
│  Survives restarts (data on disk or in Redis with persistence). │
│  Doesn't know what the message MEANS. It's just a courier.     │
│                                                                 │
│  CONSUMER / WORKER (separate Python process):                   │
│  ─────────────────────────────────────────────                  │
│  Waits for messages. Picks one up. Does the work.               │
│  Reports success or failure.                                    │
│  Doesn't know WHO sent the message.                             │
│  Could be running on a completely different server.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The critical rule:**

> "The producer and consumer never talk to each other directly. All communication goes through the broker. This is the same principle as FastAPI's `Depends()` — inversion of control, decoupling. The producer says 'here's work,' the consumer says 'give me work,' and the broker connects them without either knowing the other exists."

---

## 3.2 Why the Broker Exists

**Students will ask: "Why not just call the worker function directly?"**

```
┌─────────────────────────────────────────────────────────────────┐
│             WHY NOT JUST CALL THE FUNCTION?                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Direct call" approach:                                        │
│                                                                 │
│  # In your FastAPI endpoint                                     │
│  from worker import send_email     # Import worker code         │
│  await send_email(order)           # Call it directly           │
│                                                                 │
│  Problems:                                                      │
│  1. You're back to inline processing (user waits again)         │
│  2. If you use create_task(), it's still in-process (no         │
│     durability, same limitations as BackgroundTasks)             │
│  3. The worker code runs in YOUR API process, consuming         │
│     YOUR API's CPU and memory                                   │
│  4. You can't scale workers independently of API instances      │
│  5. If the worker code crashes, it can take down your API       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The broker provides five guarantees that direct calls cannot:**

```
┌─────────────────────────────────────────────────────────────────┐
│            THE FIVE GUARANTEES OF A BROKER                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  1. DECOUPLING                                                  │
│  ─────────────                                                  │
│  Producer and consumer are independent processes.               │
│  Deploy, restart, scale them separately.                        │
│  The API doesn't import any worker code.                        │
│                                                                 │
│    API Process    ┆    Worker Process                            │
│    ┌──────────┐   ┆    ┌──────────┐                             │
│    │ FastAPI  │   ┆    │ Worker   │                             │
│    │ code     │   ┆    │ code     │                             │
│    └──────────┘   ┆    └──────────┘                             │
│    (no worker     ┆    (no FastAPI                               │
│     imports)      ┆     imports)                                 │
│                   ┆                                              │
│    They don't know each other exists.                           │
│                                                                 │
│                                                                 │
│  2. DURABILITY                                                  │
│  ─────────────                                                  │
│  Messages survive process restarts and crashes.                 │
│  Redis persists to disk (AOF/RDB — you know this from Week 10).│
│  RabbitMQ writes messages to disk by default.                   │
│                                                                 │
│    API crashes?    → Messages are safe in broker.               │
│    Worker crashes? → Unfinished job returns to queue.           │
│    Broker restarts?→ Persistent messages survive.               │
│                                                                 │
│                                                                 │
│  3. DISTRIBUTION                                                │
│  ────────────────                                               │
│  Workers can run on any machine that can connect to the broker. │
│  Scale by adding more workers, not more API instances.          │
│                                                                 │
│    ┌─────────┐                                                  │
│    │ API     │──┐                                               │
│    └─────────┘  │    ┌─────────┐    ┌──────────┐               │
│                 ├───▶│ Redis   │───▶│ Worker 1 │  (Server A)   │
│    ┌─────────┐  │    │ (broker)│    ├──────────┤               │
│    │ API     │──┘    └─────────┘───▶│ Worker 2 │  (Server B)   │
│    └─────────┘                  ───▶├──────────┤               │
│                                     │ Worker 3 │  (Server C)   │
│                                     └──────────┘               │
│                                                                 │
│                                                                 │
│  4. BUFFERING                                                   │
│  ────────────                                                   │
│  The queue absorbs traffic spikes. If 1000 jobs arrive in       │
│  one second, workers process them at their own pace.            │
│                                                                 │
│    Incoming rate:   ████████████████  (1000 jobs/sec burst)     │
│    Queue depth:     ████████████████████████  (growing)         │
│    Processing rate: ████  ████  ████  (steady 100 jobs/sec)     │
│                                                                 │
│    The queue acts as a shock absorber. No jobs are dropped.     │
│    Your API stays fast. Workers catch up over time.             │
│                                                                 │
│                                                                 │
│  5. ORDERING                                                    │
│  ────────────                                                   │
│  Messages are processed in FIFO order by default.               │
│  First job in, first job out. Fair and predictable.             │
│  (Some brokers support priority queues for urgent work.)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The ticket rail isn't just a piece of metal. It's what allows the pizza shop to survive a Friday night rush. Without it, the cashier is overwhelmed, orders are forgotten, and the kitchen has no idea what to cook next. The broker IS the system."

---

## 3.3 Building a Tiny Queue with Redis (You Already Know How)

**Here's the key insight: you already know how to build a message broker.**

Redis lists from Week 10. `LPUSH` to add. `BRPOP` to wait-and-remove. That's a job queue.

```python
# === producer.py (runs inside your FastAPI app) ===

import json
import uuid
import redis.asyncio as aioredis
from fastapi import FastAPI, Depends

app = FastAPI()

async def get_redis() -> aioredis.Redis:
    return aioredis.from_url("redis://localhost:6379")

@app.post("/orders/{order_id}/confirm")
async def confirm_order(
    order_id: int,
    redis: aioredis.Redis = Depends(get_redis),
):
    # 1. Do the core business logic
    order = await get_order_from_db(order_id)
    await update_order_status(order_id, "confirmed")

    # 2. Create a job message
    job = {
        "id": str(uuid.uuid4()),
        "task": "send_confirmation_email",
        "args": {
            "email": order["user_email"],
            "order_id": order_id,
            "order_total": order["total"],
        },
        "created_at": "2026-02-14T10:30:00Z",
    }

    # 3. Push it onto the Redis queue
    await redis.lpush("jobs:email", json.dumps(job))

    # 4. Respond immediately
    return {
        "status": "confirmed",
        "order_id": order_id,
        "email_job_id": job["id"],  # Client can track this
    }
```

```python
# === worker.py (a SEPARATE Python process — not part of FastAPI!) ===

import json
import redis
import time

def send_email(email: str, order_id: int, order_total: float) -> None:
    """Actually send the email (simulated)"""
    print(f"Sending email to {email} for order #{order_id}...")
    time.sleep(3)  # Real SMTP would take this long
    print(f"✉️  Email sent to {email}!")

def worker_loop():
    r = redis.from_url("redis://localhost:6379")
    print("Worker started. Waiting for jobs...")

    while True:
        # BRPOP: Block until a job is available, then pop it
        # This is the worker WAITING by the ticket rail for the next order
        _, job_json = r.brpop("jobs:email")

        job = json.loads(job_json)
        print(f"Picked up job {job['id']}: {job['task']}")

        try:
            send_email(
                email=job["args"]["email"],
                order_id=job["args"]["order_id"],
                order_total=job["args"]["order_total"],
            )
            print(f"✅ Job {job['id']} completed")
        except Exception as e:
            print(f"❌ Job {job['id']} failed: {e}")
            # In a real system: push to retry queue or dead letter queue

if __name__ == "__main__":
    worker_loop()
```

**Run both in separate terminals:**

```bash
# Terminal 1: Start your API
$ uvicorn producer:app --reload

# Terminal 2: Start your worker
$ python worker.py
Worker started. Waiting for jobs...
```

```bash
# Terminal 3: Hit the endpoint
$ curl -X POST http://localhost:8000/orders/42/confirm
{"status":"confirmed","order_id":42,"email_job_id":"a1b2c3..."}
# Response is INSTANT
```

```bash
# Terminal 2 (worker) shows:
Picked up job a1b2c3...: send_confirmation_email
Sending email to customer@example.com for order #42...
✉️  Email sent to customer@example.com!
✅ Job a1b2c3... completed
```

**Visualize the flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE COMPLETE FLOW                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Terminal 1 (API)        Terminal 3 (Client)                    │
│  ────────────────        ──────────────────                     │
│                          POST /orders/42/confirm                │
│       ◀──────────────────────────────────────                   │
│       │                                                         │
│   Read DB (50ms)                                                │
│   Write DB (50ms)                                               │
│   LPUSH job to Redis                                            │
│       │                                                         │
│       ──────────────────────────────────────▶                   │
│                          {"status":"confirmed"}  (100ms total)  │
│                                                                 │
│                                                                 │
│  Redis                   Terminal 2 (Worker)                    │
│  ─────                   ──────────────────                     │
│  jobs:email: [job_json]                                         │
│       │                                                         │
│       ──────────────────▶ BRPOP picks up job                    │
│                           send_email() runs (3 seconds)         │
│                           ✅ Done                                │
│                                                                 │
│  The client is LONG GONE by the time the email sends.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "15 lines in the producer. 15 lines in the worker. Redis in the middle. That's the entire message broker pattern. Everything else — Celery, RabbitMQ, AWS SQS — is just this pattern with better error handling, retry logic, monitoring, and scheduling bolted on top."

---

## 3.4 Job Lifecycle and States

**Every job goes through a lifecycle. Tracking it is how you answer "Is my pizza ready yet?"**

```
┌─────────────────────────────────────────────────────────────────┐
│                    JOB LIFECYCLE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │          │    │          │    │          │    │          │  │
│  │ PENDING  │───▶│ RECEIVED │───▶│ STARTED  │───▶│ SUCCESS  │  │
│  │          │    │          │    │          │    │          │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │                              │                          │
│       │              Job in queue,    │          Job completed   │
│       │              waiting for a    │          successfully    │
│  Job created,        worker to        │                          │
│  pushed to           pick it up       │                          │
│  broker                               │                          │
│                                       ▼                          │
│                                  ┌──────────┐                   │
│                                  │          │                   │
│                                  │ FAILURE  │                   │
│                                  │          │                   │
│                                  └─────┬────┘                   │
│                                        │                        │
│                                        ▼                        │
│                              ┌────────────────┐                 │
│                              │ Retry? (Y/N)   │                 │
│                              └───┬────────┬───┘                 │
│                                  │        │                     │
│                                Yes        No                    │
│                                  │        │                     │
│                                  ▼        ▼                     │
│                           ┌─────────┐ ┌──────────┐              │
│                           │ RETRY/  │ │  DEAD    │              │
│                           │ PENDING │ │  LETTER  │              │
│                           │ (back   │ │ (gave up)│              │
│                           │ in queue│ │          │              │
│                           └─────────┘ └──────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map to the pizza shop:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  PENDING    │  Ticket clipped on rail. No cook has seen it yet. │
│  RECEIVED   │  Cook pulled ticket from rail, reads it.          │
│  STARTED    │  Cook is actively making the pizza.               │
│  SUCCESS    │  Pizza is done. Ready for pickup.                 │
│  FAILURE    │  Cook dropped the pizza. 🍕💥                      │
│  RETRY      │  Cook starts over with a new dough.               │
│  DEAD LETTER│  Third pizza also dropped. Manager is called.     │
│             │  (investigate manually)                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Tracking state with Redis hashes (you already know this from Week 10):**

```python
# When producing a job, also store its status:
async def enqueue_job(redis: aioredis.Redis, job: dict) -> str:
    job_id = job["id"]

    # Store job metadata in a hash
    await redis.hset(f"job:{job_id}", mapping={
        "status": "PENDING",
        "task": job["task"],
        "created_at": job["created_at"],
        "result": "",
        "error": "",
    })

    # Push to the work queue
    await redis.lpush("jobs:email", json.dumps(job))

    return job_id


# Expose a status endpoint for clients:
@app.get("/jobs/{job_id}/status")
async def get_job_status(
    job_id: str,
    redis: aioredis.Redis = Depends(get_redis),
):
    status_data = await redis.hgetall(f"job:{job_id}")
    if not status_data:
        raise HTTPException(status_code=404, detail="Job not found")

    return {
        "job_id": job_id,
        "status": status_data[b"status"].decode(),
        "result": status_data[b"result"].decode() or None,
        "error": status_data[b"error"].decode() or None,
    }
```

```python
# In the worker, update status at each stage:
def process_job(r: redis.Redis, job: dict) -> None:
    job_id = job["id"]

    # Mark as STARTED
    r.hset(f"job:{job_id}", "status", "STARTED")

    try:
        result = send_email(
            email=job["args"]["email"],
            order_id=job["args"]["order_id"],
            order_total=job["args"]["order_total"],
        )
        # Mark as SUCCESS
        r.hset(f"job:{job_id}", mapping={
            "status": "SUCCESS",
            "result": "Email sent successfully",
        })

    except Exception as e:
        # Mark as FAILURE
        r.hset(f"job:{job_id}", mapping={
            "status": "FAILURE",
            "error": str(e),
        })
```

**The client can now poll:**

```
GET /jobs/a1b2c3/status  →  {"status": "PENDING"}
GET /jobs/a1b2c3/status  →  {"status": "STARTED"}
GET /jobs/a1b2c3/status  →  {"status": "SUCCESS", "result": "Email sent"}
```

> "Order #47 — you can call back and ask if your pizza is ready. The system knows where every job is at every moment. That's the power of explicit state tracking."

---

## 3.5 Workers: The Other Side of the Wall

**This is the single most important mental shift in this lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ⚠️  THE WORKER IS NOT PART OF YOUR FASTAPI APPLICATION.       │
│                                                                 │
│  It is a SEPARATE Python process.                               │
│  It runs in a SEPARATE terminal.                                │
│  It could run on a SEPARATE server.                             │
│  It does NOT import your FastAPI app.                           │
│  It does NOT handle HTTP requests.                              │
│  It does NOT know about endpoints.                              │
│                                                                 │
│  It is a completely independent program that:                   │
│  1. Connects to the same Redis as your API                     │
│  2. Waits for messages to appear                                │
│  3. Processes them                                              │
│  4. Reports results                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize your running system:**

```
┌─────────────────────────────────────────────────────────────────┐
│              YOUR SYSTEM IN PRODUCTION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  Server A                      Server B (could be same machine) │
│  ┌──────────────────────┐      ┌──────────────────────┐        │
│  │                      │      │                      │        │
│  │  $ uvicorn main:app  │      │  $ python worker.py  │        │
│  │                      │      │                      │        │
│  │  FastAPI Process      │      │  Worker Process      │        │
│  │  ├─ handles HTTP     │      │  ├─ NO HTTP server   │        │
│  │  ├─ validates input  │      │  ├─ loops forever    │        │
│  │  ├─ queries DB       │      │  ├─ BRPOP from Redis │        │
│  │  ├─ LPUSH to Redis   │      │  ├─ does the work   │        │
│  │  └─ returns response │      │  └─ updates status   │        │
│  │                      │      │                      │        │
│  └──────────┬───────────┘      └──────────┬───────────┘        │
│             │                              │                    │
│             │      ┌──────────────┐        │                    │
│             └─────▶│    REDIS     │◀───────┘                    │
│                    │              │                              │
│                    │ jobs:email   │                              │
│                    │ [msg1, msg2] │                              │
│                    │              │                              │
│                    │ job:a1b2c3   │                              │
│                    │ {status:...} │                              │
│                    └──────────────┘                              │
│                                                                 │
│                                                                 │
│  Also running (from Weeks 5-6):                                 │
│  ┌──────────────────────┐                                       │
│  │     PostgreSQL       │   ← Both API and Worker connect to   │
│  │                      │      the same database when needed    │
│  └──────────────────────┘                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Scaling workers is trivially easy:**

```
┌─────────────────────────────────────────────────────────────────┐
│              SCALING WORKERS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Friday night. 500 orders per minute. Email queue is backed up. │
│                                                                 │
│  Solution: start more workers. Each worker BRPOP from the       │
│  same queue. Redis ensures each job is given to exactly ONE.    │
│                                                                 │
│       ┌─────────┐                                               │
│       │  Redis   │                                              │
│       │ jobs:    │                                              │
│       │ email    │                                              │
│       └──┬──┬──┬─┘                                              │
│          │  │  │                                                │
│          ▼  ▼  ▼                                                │
│     ┌────┐┌────┐┌────┐                                         │
│     │ W1 ││ W2 ││ W3 │   ← Just start more instances!         │
│     └────┘└────┘└────┘                                         │
│                                                                 │
│  Monday morning. 5 orders per minute.                           │
│  Turn off W2 and W3. W1 handles it alone.                      │
│                                                                 │
│  The API code doesn't change. The broker doesn't change.        │
│  You ONLY change the number of workers.                         │
│                                                                 │
│  This is like adding more cooks during rush hour.               │
│  The cashier doesn't need to know how many cooks are back       │
│  there. The ticket rail doesn't change. Just more hands.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Multiple queue types for different work:**

```python
# Worker can listen to specific queues:
def email_worker():
    """This worker ONLY handles email jobs"""
    r = redis.from_url("redis://localhost:6379")
    while True:
        _, job_json = r.brpop("jobs:email")
        # ... process email ...

def pdf_worker():
    """This worker ONLY handles PDF generation"""
    r = redis.from_url("redis://localhost:6379")
    while True:
        _, job_json = r.brpop("jobs:pdf")
        # ... generate PDF ...

def general_worker():
    """This worker handles MULTIPLE queues (with priority)"""
    r = redis.from_url("redis://localhost:6379")
    while True:
        # BRPOP can listen to multiple lists — first one wins
        # This gives implicit priority: email > pdf > analytics
        queue_name, job_json = r.brpop(["jobs:email", "jobs:pdf", "jobs:analytics"])
        # ... route to appropriate handler ...
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 SPECIALIZED WORKERS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Like a real kitchen has stations:                              │
│                                                                 │
│    ┌─────────┐       ┌────────────┐                             │
│    │ jobs:   │──────▶│ Email      │  (2 workers)                │
│    │ email   │       │ Workers    │                              │
│    ├─────────┤       ├────────────┤                              │
│    │ jobs:   │──────▶│ PDF        │  (1 worker — heavy CPU)     │
│    │ pdf     │       │ Worker     │                              │
│    ├─────────┤       ├────────────┤                              │
│    │ jobs:   │──────▶│ Analytics  │  (1 worker — low priority)  │
│    │ anlytcs │       │ Worker     │                              │
│    └─────────┘       └────────────┘                              │
│                                                                 │
│  Pizza station. Salad station. Dessert station.                 │
│  Each specialized. Each scaled independently.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: JOB QUEUE VS PUB/SUB

## 4.1 Two Patterns, Two Problems

**There are two fundamentally different questions you can ask:**

```
┌─────────────────────────────────────────────────────────────────┐
│             TWO QUESTIONS, TWO PATTERNS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  QUESTION 1: "I have a task. Someone please do it."             │
│  ──────────────────────────────────────────────────             │
│  → JOB QUEUE                                                    │
│  → One message, ONE consumer processes it                       │
│  → The message is CONSUMED (removed from queue)                 │
│  → If 3 workers are listening, only ONE gets the job            │
│                                                                 │
│                                                                 │
│  QUESTION 2: "Something happened. Anyone who cares, react."     │
│  ──────────────────────────────────────────────────────────     │
│  → PUB/SUB                                                      │
│  → One message, ALL subscribers receive a copy                  │
│  → The message is BROADCAST (every listener gets it)            │
│  → If 3 services are subscribed, all 3 get the message          │
│                                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Job Queue: "Make This Pizza"

**A job queue is: one job, one worker, one outcome.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      JOB QUEUE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Customer orders a pepperoni pizza.                              │
│  ONE cook makes it. Not all cooks. ONE.                         │
│                                                                 │
│                                                                 │
│  Producer              Queue              Consumers             │
│  ─────────             ─────              ─────────             │
│                                                                 │
│  [Order] ───LPUSH───▶ ┌──────────────┐                          │
│                       │ Job A        │                          │
│                       │ Job B        │──BRPOP──▶ [Worker 1] ✅  │
│                       │ Job C        │                          │
│                       └──────────────┘──BRPOP──▶ [Worker 2] ✅  │
│                                                                 │
│                                                                 │
│  Job B goes to Worker 1 OR Worker 2. Never both.                │
│  Once a worker pops it, it's GONE from the queue.               │
│                                                                 │
│                                                                 │
│  USE FOR:                                                       │
│  • "Send this email"          (one email, not duplicated)       │
│  • "Generate this PDF"        (one PDF, not two copies)         │
│  • "Process this payment"     (charge once, not twice!)         │
│  • "Resize this image"        (one worker handles it)           │
│  • "Run this report query"    (one result set needed)           │
│                                                                 │
│  KEY PROPERTY: Exactly-once processing (ideally).               │
│  Each job should be done by ONE worker, ONE time.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You order ONE pizza. ONE cook makes it. If two cooks both grabbed the same ticket, you'd get two pizzas and be charged twice. Job queues prevent that."

---

## 4.3 Pub/Sub: "Order #47 Is Ready!"

**Pub/Sub is: one event, many listeners, each reacts independently.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      PUB/SUB                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The kitchen shouts: "ORDER #47 IS READY!"                      │
│  Multiple people react:                                         │
│  • Counter staff picks up the pizza                             │
│  • Display board updates order status                           │
│  • Manager tallies the completion for metrics                   │
│                                                                 │
│  The announcement goes to EVERYONE who's listening.             │
│                                                                 │
│                                                                 │
│  Publisher           Channel            Subscribers             │
│  ─────────           ───────            ───────────             │
│                                                                 │
│                      ┌──────────────┐                           │
│                      │              │──▶ [Email Service]        │
│  [OrderConfirmed] ──▶│   "orders"   │──▶ [Analytics]           │
│                      │   channel    │──▶ [Inventory]           │
│                      │              │──▶ [Audit Log]           │
│                      └──────────────┘                           │
│                                                                 │
│  ALL four services receive the event. Independently.            │
│  Each decides what to do with it on their own.                  │
│                                                                 │
│                                                                 │
│  USE FOR:                                                       │
│  • "User signed up"     → Welcome email + analytics + onboard  │
│  • "Order placed"       → Email + warehouse + analytics + audit│
│  • "Payment received"   → Receipt + accounting + notification  │
│  • "Article published"  → Cache invalidation + RSS + search    │
│                                                                 │
│  KEY PROPERTY: Fire-and-forget broadcast.                       │
│  Publisher doesn't know (or care) who's listening.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The kitchen doesn't shout '#47 is ready' AT the counter person. They shout it to the room. Whoever cares, reacts. If you add a new service next month — say, a loyalty points calculator — you just subscribe it to the channel. The publisher doesn't change. The kitchen doesn't know or care that loyalty points exist."

---

## 4.4 The Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│               WHICH PATTERN DO I USE?                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                     START HERE                                  │
│                         │                                       │
│                         ▼                                       │
│              ┌─────────────────────┐                            │
│              │ How many services   │                            │
│              │ need to act on this │                            │
│              │ message?            │                            │
│              └──────────┬──────────┘                            │
│                   │           │                                 │
│              EXACTLY ONE    MULTIPLE                            │
│                   │           │                                 │
│                   ▼           ▼                                 │
│           ┌────────────┐  ┌───────────────────┐                │
│           │ Must it     │  │                   │                │
│           │ definitely  │  │  PUB/SUB          │                │
│           │ complete?   │  │  Broadcast the    │                │
│           └──────┬──────┘  │  event.           │                │
│              │       │     │  Let listeners    │                │
│            YES       NO    │  decide.          │                │
│              │       │     │                   │                │
│              ▼       ▼     └───────────────────┘                │
│     ┌──────────┐ ┌──────────┐                                  │
│     │ JOB      │ │ PUB/SUB  │                                  │
│     │ QUEUE    │ │ might be │                                  │
│     │ with     │ │ fine     │                                  │
│     │ retry    │ │ (best    │                                  │
│     │ logic    │ │ effort)  │                                  │
│     └──────────┘ └──────────┘                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A concrete comparison using our order confirmation scenario:**

```
┌─────────────────────────────────────────────────────────────────┐
│           ORDER CONFIRMATION: WHICH PATTERN?                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Task                      Pattern       Why?                   │
│  ─────────────────         ──────────    ─────────────────────  │
│  Send confirmation email   Job Queue     MUST happen. One email │
│                                          per order. Retry if    │
│                                          SMTP fails.            │
│                                                                 │
│  Generate invoice PDF      Job Queue     MUST complete. One PDF │
│                                          per order. Result is   │
│                                          stored.                │
│                                                                 │
│  "Order confirmed" event   Pub/Sub       Multiple services care:│
│                                          analytics, warehouse,  │
│                                          audit log, dashboard.  │
│                                          Publisher doesn't know │
│                                          them all.              │
│                                                                 │
│                                                                 │
│  🔑 In practice, you often COMBINE both:                        │
│                                                                 │
│  1. Endpoint publishes event: "OrderConfirmed"                  │
│  2. Email service SUBSCRIBES, receives event                    │
│  3. Email service enqueues a JOB: "send_email(order_42)"        │
│  4. Email worker picks up the job, sends email with retry       │
│                                                                 │
│  Pub/Sub for fan-out. Job queue for reliable execution.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key difference in one diagram:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  JOB QUEUE:                                                     │
│                                                                 │
│       Message ────▶ [Queue] ────▶ ONE worker processes it       │
│                                   (message is consumed)         │
│                                                                 │
│                                                                 │
│  PUB/SUB:                                                       │
│                                                                 │
│                            ┌────▶ Subscriber A gets a copy      │
│       Message ────▶ [Channel] ──▶ Subscriber B gets a copy     │
│                            └────▶ Subscriber C gets a copy      │
│                                   (message is broadcast)        │
│                                                                 │
│                                                                 │
│  ⚠️  IMPORTANT PUB/SUB CAVEAT:                                  │
│  If NO subscriber is listening when the event fires,            │
│  the message is LOST. Pub/Sub has no memory.                    │
│  (This is why you sometimes need durable pub/sub —              │
│   we'll cover this in Lecture 4 with Redis Streams,             │
│   Kafka, and RabbitMQ.)                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: EVENT-DRIVEN ARCHITECTURE OVERVIEW

## 5.1 From Commands to Events

**There are two ways to think about communication between parts of your system:**

```
┌─────────────────────────────────────────────────────────────────┐
│              COMMANDS VS EVENTS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  COMMAND-DRIVEN (imperative):                                   │
│  ─────────────────────────────                                  │
│  "Do this specific thing."                                      │
│                                                                 │
│  The sender KNOWS the receiver.                                 │
│  The sender EXPECTS the receiver to act.                        │
│  The sender WAITS for (or depends on) the result.               │
│                                                                 │
│  Code looks like:                                               │
│  ┌──────────────────────────────────────────┐                   │
│  │  await email_service.send(email)         │                   │
│  │  await warehouse_api.notify(order)       │                   │
│  │  await analytics.track("order_confirmed")│                   │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
│  Problem: Every time you add a new side effect, you modify      │
│  the confirm_order() function. It grows and grows.              │
│                                                                 │
│                                                                 │
│  EVENT-DRIVEN (reactive):                                       │
│  ─────────────────────────                                      │
│  "This happened. React if you care."                            │
│                                                                 │
│  The sender DOESN'T KNOW the receivers.                         │
│  The sender DOESN'T CARE who acts.                              │
│  The sender's job is DONE after emitting the event.             │
│                                                                 │
│  Code looks like:                                               │
│  ┌──────────────────────────────────────────┐                   │
│  │  await event_bus.emit("OrderConfirmed", {│                   │
│  │      "order_id": 42,                     │                   │
│  │      "user_email": "customer@mail.com",  │                   │
│  │      "total": 99.99,                     │                   │
│  │  })                                      │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
│  Adding a new side effect? Add a new subscriber.                │
│  The confirm_order() function NEVER CHANGES.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize the structural difference:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  COMMAND-DRIVEN: confirm_order() knows everything               │
│                                                                 │
│      confirm_order()                                            │
│       ├──▶ email_service.send()                                 │
│       ├──▶ warehouse_api.notify()                               │
│       ├──▶ analytics.track()                                    │
│       ├──▶ audit_log.write()         ← Added in March           │
│       └──▶ loyalty_points.add()      ← Added in April           │
│                                                                 │
│  confirm_order() has been modified 5 times for 5 side effects.  │
│  It now imports 5 services. It's fragile and bloated.           │
│                                                                 │
│                                                                 │
│  EVENT-DRIVEN: confirm_order() knows nothing                    │
│                                                                 │
│      confirm_order()                                            │
│       └──▶ emit("OrderConfirmed")                               │
│                    │                                            │
│                    ├──▶ EmailSubscriber (hears event, sends)     │
│                    ├──▶ WarehouseSubscriber (hears event, calls) │
│                    ├──▶ AnalyticsSubscriber (hears event, tracks)│
│                    ├──▶ AuditSubscriber     ← Added in March     │
│                    └──▶ LoyaltySubscriber   ← Added in April     │
│                                                                 │
│  confirm_order() was NEVER modified after its initial write.    │
│  New behaviors are added by adding new subscribers.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Think about it through the lens of something you already know: the Open/Closed Principle. The endpoint is CLOSED for modification but OPEN for extension. Adding new reactions to an event doesn't change the code that produces the event. You experienced this principle with your Task Manager's error handling and Pydantic validation. Now it applies at the architecture level."

---

## 5.2 The Architecture Spectrum

**You're not choosing between two extremes. There's a spectrum.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE ARCHITECTURE SPECTRUM                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Simple ◀──────────────────────────────────────────▶ Complex    │
│                                                                 │
│                                                                 │
│  1. INLINE (where you started today)                            │
│  ─────────────────────────────────                              │
│  Everything happens in the request handler.                     │
│  No infrastructure needed. No reliability.                      │
│                                                                 │
│  [API] ──▶ do everything ──▶ respond                            │
│                                                                 │
│                                                                 │
│  2. BackgroundTasks (Part 2)                                    │
│  ───────────────────────────                                    │
│  Side effects run after response, same process.                 │
│  No extra infrastructure. No durability.                        │
│                                                                 │
│  [API] ──▶ respond ──▶ do side effects (same process)           │
│                                                                 │
│                                                                 │
│  3. JOB QUEUE (Part 3)                                          │
│  ─────────────────────                                          │
│  Side effects run in separate worker processes.                 │
│  Needs a broker (Redis). Durable. Retryable. Scalable.         │
│                                                                 │
│  [API] ──▶ [Broker] ──▶ [Worker]                                │
│                                                                 │
│         ▲                                                       │
│         └── YOU ARE HERE for the rest of this course            │
│                                                                 │
│                                                                 │
│  4. EVENT-DRIVEN                                                │
│  ────────────────                                               │
│  Components react to events. Fully decoupled.                   │
│  Needs more sophisticated broker (Redis Streams, RabbitMQ).     │
│                                                                 │
│  [API] ──▶ emit event ──▶ [Broker] ──▶ [Many Subscribers]      │
│                                                                 │
│                                                                 │
│  5. EVENT SOURCING + CQRS (advanced, awareness only)            │
│  ────────────────────────────────────────────────               │
│  Events ARE the source of truth. State is derived.              │
│  Read and write models are separated.                           │
│  Needs Kafka or similar. Complex. Powerful at scale.            │
│                                                                 │
│  [Command] ──▶ [Event Store] ──▶ [Read Model Projections]      │
│                                                                 │
│                                                                 │
│  ⚠️  DON'T start at level 5. START at level 3.                  │
│  Most applications NEVER need levels 4-5.                       │
│  Level 3 (job queue) solves 90% of real-world background        │
│  processing needs.                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Where This Leads (Preview of Lecture 4)

**Today was about building the mental model. Here's what comes next:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT WE COVERED VS WHAT'S COMING                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TODAY (Lecture 1): Concepts                                    │
│  ├─ Why background jobs exist                                   │
│  ├─ BackgroundTasks and its limits                               │
│  ├─ Producer / Broker / Consumer pattern                        │
│  ├─ Job queue vs pub/sub                                        │
│  └─ Event-driven thinking (awareness)                           │
│                                                                 │
│  LECTURE 2: Celery Fundamentals                                 │
│  ├─ Celery = the job queue pattern with batteries included      │
│  ├─ Everything from Part 3 today, but with retry, monitoring,   │
│  │   result tracking, and serialization built in                │
│  └─ You'll replace the hand-rolled Redis queue with Celery      │
│                                                                 │
│  LECTURE 3: Scheduling & Monitoring                              │
│  ├─ Celery Beat = the scheduled work from Section 1.3           │
│  ├─ Flower = the monitoring dashboard Section 2.3 said you      │
│  │   were missing                                               │
│  └─ Dead letter handling = the "gave up" state from Section 3.4 │
│                                                                 │
│  LECTURE 4: Event-Driven Architecture                           │
│  ├─ Redis Pub/Sub implementation (Part 4 concept → real code)   │
│  ├─ When to upgrade: Kafka, RabbitMQ                            │
│  ├─ Event sourcing + CQRS (Section 5.2, levels 4-5)            │
│  └─ Choosing: BackgroundTasks vs Celery vs message broker       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Misconceptions

```
┌─────────────────────────────────────────────────────────────────┐
│        MISCONCEPTION 1: "Async IS background processing"       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ WRONG:                                                      │
│  "My endpoint is async, so the work happens in the background"  │
│                                                                 │
│  ✅ CORRECT:                                                    │
│  Async lets your event loop handle OTHER requests while         │
│  THIS request waits for I/O. But THIS request still waits      │
│  for ALL its awaits to complete before responding.              │
│                                                                 │
│  Async = concurrent I/O within a request lifecycle              │
│  Background = work outside the request lifecycle entirely       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│        MISCONCEPTION 2: "BackgroundTasks is enough              │
│                          for production apps"                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ WRONG:                                                      │
│  "I'll just use BackgroundTasks for everything"                 │
│                                                                 │
│  ✅ CORRECT:                                                    │
│  BackgroundTasks is fine for non-critical, fast tasks.          │
│  For anything that MUST complete (emails, payments, exports):   │
│  use a real job queue with a broker.                            │
│                                                                 │
│  The question to ask: "If this task silently disappears,        │
│  does a customer get hurt?" If yes → job queue.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│        MISCONCEPTION 3: "More workers = always faster"         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ WRONG:                                                      │
│  "Email is slow? Add 50 workers!"                               │
│                                                                 │
│  ✅ CORRECT:                                                    │
│  If the bottleneck is the SMTP server rate limit (10 emails/    │
│  second), 50 workers will just hit the rate limit 50x harder.   │
│  You need to understand WHERE the slowness is.                  │
│                                                                 │
│  Remember circuit breakers from Week 8? Same principle.         │
│  Scaling workers only helps if the workers are the bottleneck.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│        MISCONCEPTION 4: "I need event-driven architecture      │
│                          for my app"                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ WRONG:                                                      │
│  "Let me set up Kafka and event sourcing from day one"          │
│                                                                 │
│  ✅ CORRECT:                                                    │
│  Start with a job queue (Celery + Redis). It solves 90% of     │
│  background processing needs. Move to event-driven IF and       │
│  WHEN you have multiple independent services that need to       │
│  react to the same events. Most apps never reach that point.    │
│                                                                 │
│  Premature architecture is as dangerous as premature            │
│  optimization.                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│        MISCONCEPTION 5: "The worker shares memory               │
│                          with my API"                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ WRONG:                                                      │
│  "I'll store the order in a Python dict and the worker          │
│   will just access it"                                          │
│                                                                 │
│  ✅ CORRECT:                                                    │
│  Workers are SEPARATE processes. Separate memory space.         │
│  Possibly on a different server. They communicate ONLY          │
│  through the broker (Redis) and shared databases (PostgreSQL).  │
│                                                                 │
│  Everything the worker needs must be in the job message         │
│  or fetchable from the database. You cannot pass Python         │
│  objects between processes — only serialized data (JSON).       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│             BACKGROUND PROCESSING QUICK REFERENCE               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THREE REASONS TO GO BACKGROUND:                                │
│      1. Decoupled (side effects: email, notifications)          │
│      2. Long-running (reports, exports, processing)             │
│      3. Scheduled (cron jobs, periodic cleanup)                 │
│                                                                 │
│  THE CORE QUESTION:                                             │
│      "Does the user need to wait for this                       │
│       before getting a response?"                               │
│       No? → Move it to the background.                          │
│                                                                 │
│  BackgroundTasks:                                               │
│      background_tasks.add_task(func, *args)                     │
│      Good for: Quick, non-critical, fire-and-forget             │
│      Bad for:  Anything that MUST complete                      │
│                                                                 │
│  MESSAGE BROKER PATTERN:                                        │
│      Producer ──▶ Broker ──▶ Consumer (Worker)                  │
│      Producer: LPUSH job to Redis list                          │
│      Consumer: BRPOP job from Redis list (separate process)     │
│                                                                 │
│  JOB QUEUE:                                                     │
│      One job → one worker. Consumed. Must complete.             │
│      "Do this specific task"                                    │
│                                                                 │
│  PUB/SUB:                                                       │
│      One event → many listeners. Broadcast. Best-effort.        │
│      "Something happened, react if you care"                    │
│                                                                 │
│  JOB STATES:                                                    │
│      PENDING → RECEIVED → STARTED → SUCCESS / FAILURE           │
│                                          └─→ RETRY or DEAD      │
│                                                                 │
│  ARCHITECTURE SPECTRUM:                                         │
│      Inline → BackgroundTasks → Job Queue → Event-Driven        │
│      (Start with Job Queue. Upgrade only if needed.)            │
│                                                                 │
│  THE GOLDEN RULE:                                               │
│      "If this task silently disappears,                         │
│       does a customer get hurt?"                                │
│       Yes → Real job queue with retry.                          │
│       No  → BackgroundTasks is fine.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  BACKGROUND PROCESSING = SEPARATION OF CONCERNS                 │
│                           FOR TIME                              │
│                                                                 │
│  Your API's job:                                                │
│  ┌───────────────────────────────────┐                          │
│  │ 1. Validate the request           │                          │
│  │ 2. Do the CORE business logic     │                          │
│  │ 3. Queue the side effects         │                          │
│  │ 4. Respond IMMEDIATELY            │                          │
│  └───────────────────────────────────┘                          │
│                                                                 │
│  Your workers' job:                                             │
│  ┌───────────────────────────────────┐                          │
│  │ 1. Pick up jobs from the queue    │                          │
│  │ 2. Do the work (email, PDF, etc.) │                          │
│  │ 3. Retry on failure               │                          │
│  │ 4. Report the result              │                          │
│  └───────────────────────────────────┘                          │
│                                                                 │
│  They communicate through a BROKER (Redis).                     │
│  They don't know each other.                                    │
│  They don't import each other.                                  │
│  They might not even be on the same machine.                    │
│                                                                 │
│                                                                 │
│  THE PIZZA SHOP ANALOGY:                                        │
│  ├─ Cashier = Your API endpoint                                 │
│  ├─ Ticket rail = Message broker (Redis queue)                  │
│  ├─ Kitchen staff = Worker processes                            │
│  ├─ Order number = Job ID                                       │
│  ├─ "Is my order ready?" = GET /jobs/{id}/status                │
│  ├─ Burnt pizza → remake = Failed job → retry                   │
│  └─ Adding more cooks = Scaling workers                         │
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
│  LECTURE 2 (Celery Fundamentals):                               │
│  └─ The hand-rolled Redis queue from Section 3.3 becomes        │
│     Celery: @celery.task, .delay(), AsyncResult.                │
│     Everything you learned today, with retry, serialization,    │
│     result backends, and concurrency control built in.          │
│                                                                 │
│  LECTURE 3 (Scheduling & Monitoring):                           │
│  └─ Scheduled work (Section 1.3) becomes Celery Beat.           │
│     The missing visibility (Section 2.3) becomes Flower.        │
│     Dead letter handling (Section 3.4) becomes real policy.     │
│                                                                 │
│  LECTURE 4 (Event-Driven Architecture):                         │
│  └─ Pub/Sub concept (Part 4) becomes Redis Pub/Sub code.        │
│     Event-driven overview (Part 5) becomes concrete patterns.   │
│     Choosing between BackgroundTasks / Celery / broker.         │
│                                                                 │
│  WEEK 11 PROJECT:                                               │
│  └─ You'll build a background processing pipeline using         │
│     everything from this week. Scheduled jobs, event-triggered  │
│     notifications, idempotent tasks, retry logic, monitoring.   │
│                                                                 │
│  WEEK 12 (WebSockets & Performance):                            │
│  └─ Background job completes → notify user in real time         │
│     via WebSocket. "Your report is ready for download."         │
│     The job queue and WebSocket layer work together.            │
│                                                                 │
│  CAPSTONE (Weeks 13-14):                                        │
│  └─ Email notifications, report generation, scheduled           │
│     cleanup — all background jobs in your SaaS platform.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```