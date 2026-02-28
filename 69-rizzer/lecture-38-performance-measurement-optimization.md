# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FEEL THE WRONG GUESS FIRST                                     │
│  ──────────────────────────                                     │
│  Students will guess where a bottleneck is. They'll be wrong.   │
│  That failure is the lesson. You can't intuit performance.      │
│                                                                 │
│  DIAGNOSE BEFORE YOU PRESCRIBE                                  │
│  ─────────────────────────────                                  │
│  The doctor analogy carries this entire lecture.                 │
│  Measurement is not optional — it is the FIRST step.            │
│                                                                 │
│  TOOLS SERVE METHODOLOGY, NOT THE REVERSE                       │
│  ─────────────────────────────────────────                      │
│  We don't teach Locust for its own sake.                        │
│  We teach a systematic optimization CYCLE that uses tools.      │
│                                                                 │
│  CONNECT TO THE ENTIRE COURSE                                   │
│  ────────────────────────────                                   │
│  This lecture ties together almost every prior topic:            │
│  Async (W1), FastAPI (W3), SQLAlchemy (W6), EXPLAIN (W7),       │
│  Redis (W10), Celery (W11), WebSockets (W12 L1-2).             │
│  Performance is where it all converges.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│              PERFORMANCE MEASUREMENT & OPTIMIZATION             │
│                      (3–4 Hour Lecture)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM — "It Feels Slow" (45 min)                │
│  ├─ 1.1 The Guessing Game (Demonstration)                       │
│  ├─ 1.2 Why Intuition Fails                                     │
│  ├─ 1.3 The Doctor Analogy                                      │
│  └─ 1.4 Anatomy of a Request (Where Time Hides)                │
│                                                                 │
│  PART 2: REQUEST TIMING MIDDLEWARE (40 min)                     │
│  ├─ 2.1 Middleware vs Dependencies (Brief)                      │
│  ├─ 2.2 Building a Timing Middleware                            │
│  ├─ 2.3 Structured Performance Logging                          │
│  └─ 2.4 Server-Timing Header                                   │
│                                                                 │
│  PART 3: DATABASE QUERY PROFILING (45 min)                      │
│  ├─ 3.1 SQLAlchemy Query Logging                                │
│  ├─ 3.2 Counting Queries Per Request                            │
│  ├─ 3.3 Catching N+1 in Production                              │
│  └─ 3.4 Query Count Assertions in Tests                         │
│                                                                 │
│  PART 4: RESPONSE OPTIMIZATION TECHNIQUES (30 min)              │
│  ├─ 4.1 Response Compression (GzipMiddleware)                   │
│  ├─ 4.2 Parallel Operations in Endpoints                        │
│  └─ 4.3 The Optimization Toolbox                                │
│                                                                 │
│  PART 5: LOAD TESTING WITH LOCUST (45 min)                      │
│  ├─ 5.1 Why Load Test? (The Empty Restaurant Problem)           │
│  ├─ 5.2 Your First Locust Test                                  │
│  ├─ 5.3 Writing Realistic Scenarios                             │
│  └─ 5.4 Reading Results and the Optimization Cycle              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM — "It Feels Slow"

## 1.1 The Guessing Game

**Start with a piece of code. Make them commit to a wrong answer.**

Show this endpoint on screen. Tell students: this is a dashboard endpoint from a SaaS application like your capstone project. Users are complaining it's slow. **Read it carefully, then tell me: which part is the bottleneck?**

```python
# routes/dashboard.py — "Users say this is slow"

@router.get("/organizations/{org_id}/dashboard")
async def get_org_dashboard(
    org_id: int,
    db: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
):
    # --- Section A: Load org data ---
    org = await db.get(Organization, org_id)
    if not org:
        raise HTTPException(status_code=404, detail="Org not found")

    # Load all projects for this org
    stmt = select(Project).where(Project.org_id == org_id)
    result = await db.execute(stmt)
    projects = result.scalars().all()

    # For each project, load tasks and their assignees
    project_data = []
    for project in projects:
        tasks_stmt = select(Task).where(Task.project_id == project.id)
        tasks_result = await db.execute(tasks_stmt)
        tasks = tasks_result.scalars().all()

        task_details = []
        for task in tasks:
            assignee = await db.get(User, task.assignee_id)
            task_details.append({
                "id": task.id,
                "title": task.title,
                "status": task.status,
                "assignee": assignee.name if assignee else None,
            })

        project_data.append({
            "project_name": project.name,
            "task_count": len(tasks),
            "tasks": task_details,
        })

    # --- Section B: External enrichment ---
    async with httpx.AsyncClient(timeout=5.0) as client:
        analytics_resp = await client.get(
            f"https://analytics.example.com/api/org/{org_id}/summary"
        )
    async with httpx.AsyncClient(timeout=5.0) as client:
        weather_resp = await client.get(
            f"https://api.weather.example.com/current?city={org.city}"
        )

    # --- Section C: Build response ---
    return {
        "organization": org.name,
        "member_count": org.member_count,
        "projects": project_data,
        "analytics": analytics_resp.json(),
        "weather": weather_resp.json(),
    }
```

**Now poll the class:**

> "Raise your hand: who thinks Section A (database queries) is the bottleneck?"
> "Who thinks Section B (external API calls) is the bottleneck?"
> "Who thinks Section C (building the response) is the bottleneck?"

**Most students will pick Section B** — the external API calls. They *look* slow. They go over the network. They have a 5-second timeout. Surely that's the problem.

**Now reveal the actual timing data:**

```
┌─────────────────────────────────────────────────────────────────┐
│           ACTUAL TIMING BREAKDOWN (10 projects, 8 tasks each)   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Section A — Database queries:                                  │
│  ├─ 1 query: get org                           5ms              │
│  ├─ 1 query: get all projects                  6ms              │
│  ├─ 10 queries: get tasks per project       10×5ms = 50ms       │
│  ├─ 80 queries: get assignee per task       80×5ms = 400ms  🔴  │
│  ├─ ─────────────────────────────────────────────               │
│  └─ SUBTOTAL: 92 queries                    ≈ 461ms             │
│                                                                 │
│  Section B — External API calls:                                │
│  ├─ Analytics API                             280ms             │
│  ├─ Weather API                               320ms             │
│  ├─ (run SEQUENTIALLY)                                          │
│  ├─ ─────────────────────────────────────────────               │
│  └─ SUBTOTAL: 2 requests                   ≈ 600ms             │
│                                                                 │
│  Section C — Response serialization:                            │
│  └─ SUBTOTAL:                               ≈  40ms             │
│                                                                 │
│  ═══════════════════════════════════════════════════             │
│  TOTAL:                                     ≈ 1,101ms           │
│                                                                 │
│  THE REAL BOTTLENECK:                                           │
│  92 database queries.  Not the 2 API calls.                     │
│  (And it gets WORSE with more projects/tasks.)                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask:**

> "How many of you picked Section B? You were wrong — but you were wrong for an interesting reason. The external API calls are *visibly* slow. You can point at them and say 'that's network, that's slow.' But the 92 invisible little database queries each took only 5 milliseconds. They don't *look* dangerous. They don't *feel* slow. But 5ms × 92 = 461ms. **Death by a thousand paper cuts.**"

> "And here's the truly dangerous part: with 10 projects and 8 tasks each, it's 461ms. What happens when a customer has 50 projects and 20 tasks each? That's 1 + 1 + 50 + 1000 = **1,052 queries**. At 5ms each: **5.2 seconds** — just from the database. The external APIs are still 600ms. Your 'slow API calls' are now 10% of the total time."

```
┌─────────────────────────────────────────────────────────────────┐
│               SCALING BEHAVIOR                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Projects  Tasks/Project  DB Queries  DB Time   API Time  Total │
│  ────────  ────────────   ─────────   ───────   ────────  ───── │
│       5            4           27      135ms     600ms    775ms │
│      10            8           92      461ms     600ms   1101ms │
│      20           10          222     1110ms     600ms   1750ms │
│      50           20        1,052     5260ms     600ms   5900ms │
│                                                                 │
│  Section A: grows as O(projects × tasks)  ← QUADRATIC  🔴      │
│  Section B: constant at ~600ms            ← FLAT       🟢      │
│                                                                 │
│  The "small" problem becomes the DOMINANT problem at scale.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The lesson:**

> "You cannot intuit performance. You must MEASURE it. Every time someone says 'I think the bottleneck is X,' the correct response is: 'Show me the numbers.' That's what this lecture is about."

---

## 1.2 Why Intuition Fails

**Three reasons your gut feeling about performance is unreliable:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY YOUR INTUITION IS WRONG                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REASON 1: VISIBILITY BIAS                                      │
│  ─────────────────────────                                      │
│  You notice what's OBVIOUS.                                     │
│                                                                 │
│  "External API call" LOOKS slow → you suspect it.               │
│  "92 tiny queries in a loop" looks like normal code.            │
│  The biggest problems often hide in innocent-looking lines.     │
│                                                                 │
│                                                                 │
│  REASON 2: SINGLE-REQUEST THINKING                              │
│  ─────────────────────────────────                              │
│  You test with 1 user. Production has 500.                      │
│                                                                 │
│  An endpoint at 200ms with 1 user might be 3,000ms              │
│  with 200 concurrent users (connection pool exhaustion,         │
│  lock contention, Redis pipeline saturation).                   │
│  You'll never see this from your browser.                       │
│                                                                 │
│                                                                 │
│  REASON 3: AVERAGES LIE                                         │
│  ────────────────────                                           │
│  "Average response time: 150ms" — sounds great!                 │
│                                                                 │
│  Reality:                                                       │
│  ├─ 95 requests at 50ms                                         │
│  ├─  4 requests at 500ms                                        │
│  └─  1 request at 5,200ms   ← This user is FURIOUS             │
│                                                                 │
│  Average: (95×50 + 4×500 + 1×5200) / 100 = 119ms               │
│                                                                 │
│  The average represents NOBODY's actual experience.             │
│  We'll learn a better way to measure in Part 5 (percentiles).   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Doctor Analogy

**This analogy carries through the rest of the lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE DOCTOR ANALOGY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BAD DOCTOR (Guess-Based Optimization)                          │
│  ─────────────────────────────────────                          │
│                                                                 │
│  Patient: "I feel tired."                                       │
│  Doctor:  "Here, take these antibiotics."                       │
│  Patient: "But you didn't run any tests—"                       │
│  Doctor:  "Antibiotics fix most things. Trust me."              │
│                                                                 │
│  Outcome: Patient still tired. Now also has side effects.       │
│                                                                 │
│                                                                 │
│  GOOD DOCTOR (Measurement-Based Optimization)                   │
│  ────────────────────────────────────────────                   │
│                                                                 │
│  Patient: "I feel tired."                                       │
│  Doctor:  "Let's run some tests first."                         │
│           ├─ Blood work      (→ timing middleware)              │
│           ├─ Heart monitor   (→ database profiling)             │
│           └─ Stress test     (→ load testing)                   │
│  Doctor:  "Your iron is low. Here's the targeted treatment."    │
│                                                                 │
│  Outcome: Root cause found. Targeted fix. Verified recovery.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map every tool in this lecture to a diagnostic:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Medicine                 │  Performance Engineering            │
│  ─────────────────────────│──────────────────────────           │
│  "I feel sick"            │  "The app is slow"                  │
│  Taking temperature       │  Timing middleware (Part 2)         │
│  Blood work               │  Database query profiling (Part 3)  │
│  Stress test (treadmill)  │  Load testing with Locust (Part 5)  │
│  Diagnosis                │  Identifying the bottleneck         │
│  Targeted prescription    │  Specific optimization (Part 4)     │
│  Follow-up visit          │  Measure AGAIN after the fix        │
│                                                                 │
│  THE GOLDEN RULE:                                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │  Never prescribe without a diagnosis.                   │    │
│  │  Never optimize without a measurement.                  │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The optimization cycle (our "treatment plan"):**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE OPTIMIZATION CYCLE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│           ┌──────────┐                                          │
│    ┌─────▶│ MEASURE  │──────┐                                   │
│    │      └──────────┘      │                                   │
│    │                        ▼                                   │
│    │                 ┌────────────┐                              │
│    │                 │ IDENTIFY   │                              │
│    │                 │ bottleneck │                              │
│    │                 └─────┬──────┘                              │
│    │                       │                                    │
│    │                       ▼                                    │
│    │                 ┌──────────┐                                │
│    │                 │   FIX    │                                │
│    │                 │ (target) │                                │
│    │                 └─────┬────┘                                │
│    │                       │                                    │
│    │                       ▼                                    │
│    │                 ┌──────────┐                                │
│    └─────────────────│ MEASURE  │                               │
│                      │  AGAIN   │                               │
│                      └──────────┘                               │
│                                                                 │
│  "Did it actually help? By how much? Is there a new             │
│   bottleneck now? If yes, go around again."                     │
│                                                                 │
│  EVERY optimization in your Week 12 project MUST have           │
│  before/after numbers. No numbers, no credit.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.4 Anatomy of a Request (Where Time Hides)

**Before we can measure, we need to know WHERE to look.**

When a request hits your FastAPI server, it passes through many layers. Each layer adds latency. Understanding the anatomy tells you where to point your thermometer.

```
┌─────────────────────────────────────────────────────────────────┐
│            ANATOMY OF A REQUEST (Time Budget)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client sends HTTP request                                      │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────┐                                            │
│  │   MIDDLEWARE     │  1-5ms typically                           │
│  │   (CORS, auth,  │  Can spike if middleware does I/O          │
│  │    timing, etc.) │                                           │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  DEPENDENCIES   │  Varies wildly                             │
│  │  (get_db,       │  DB session: <1ms                          │
│  │   get_current_  │  Auth token verify: 1-5ms                  │
│  │   user, etc.)   │  Redis lookup: 1-2ms                       │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  ROUTE HANDLER  │  This is where most time goes              │
│  │  ┌────────────┐ │                                            │
│  │  │ DB queries │ │  5-500ms per query (depends on query)      │
│  │  ├────────────┤ │                                            │
│  │  │ Cache ops  │ │  1-5ms (Redis round-trip)                  │
│  │  ├────────────┤ │                                            │
│  │  │ External   │ │  100-5000ms (network + remote server)      │
│  │  │ API calls  │ │                                            │
│  │  ├────────────┤ │                                            │
│  │  │ Business   │ │  <1ms (usually), unless CPU-intensive      │
│  │  │ logic      │ │                                            │
│  │  └────────────┘ │                                            │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  SERIALIZATION  │  1-50ms (depends on response size)         │
│  │  (Pydantic →    │  Large nested objects = slow               │
│  │   JSON)         │                                            │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  TRANSFER       │  Depends on payload size + network         │
│  │  (network to    │  1KB = instant, 5MB = noticeable           │
│  │   client)       │  Compression helps here (Part 4)           │
│  └─────────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The performance budget concept:**

> "Think of each request as a budget. Your total budget is the response time your users will tolerate — let's say 200ms. Every layer of the request spends some of that budget. Your job is to figure out where the money goes, then cut spending on the most expensive line items."

```
┌─────────────────────────────────────────────────────────────────┐
│              EXAMPLE PERFORMANCE BUDGET                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BUDGET: 200ms total                                            │
│                                                                 │
│  ├─ Middleware:        5ms   ██                                  │
│  ├─ Dependencies:      8ms   ███                                │
│  ├─ DB queries:       82ms   █████████████████████████████      │
│  ├─ Redis cache:       3ms   █                                  │
│  ├─ Business logic:    2ms   █                                  │
│  ├─ Serialization:    15ms   ██████                             │
│  └─ Transfer:         10ms   ████                               │
│                      ──────                                     │
│                      125ms   ✅ Under budget (75ms headroom)    │
│                                                                 │
│  If DB queries grow to 400ms, you're at 525ms.                  │
│  The DB line item is where you need to cut.                     │
│  Don't waste time optimizing serialization (15ms → 10ms         │
│  saves you almost nothing).                                     │
│                                                                 │
│  ALWAYS OPTIMIZE THE BIGGEST LINE ITEM FIRST.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Now we know what to measure and where to look. Let's build the tools."

---

# PART 2: REQUEST TIMING MIDDLEWARE

## 2.1 Middleware vs Dependencies (Brief)

**Quick distinction — you know both concepts, but they serve different purposes here:**

```
┌─────────────────────────────────────────────────────────────────┐
│          MIDDLEWARE VS DEPENDENCIES FOR PERFORMANCE              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MIDDLEWARE (@app.middleware)                                    │
│  ───────────────────────────                                    │
│  Wraps EVERY request, before AND after the route handler.       │
│  Perfect for cross-cutting concerns: timing, logging, headers.  │
│                                                                 │
│  Request ──▶ [Middleware BEFORE] ──▶ Route ──▶ [Middleware AFTER]│
│                    │                              │              │
│                    └─ Start timer                  └─ Stop timer │
│                                                                 │
│  You've already used middleware: CORSMiddleware (Week 9).       │
│  Now you'll BUILD one.                                          │
│                                                                 │
│                                                                 │
│  DEPENDENCY (Depends())                                         │
│  ──────────────────────                                         │
│  Runs per-endpoint. Can be selective. Has access to params.     │
│  Better for endpoint-specific profiling or conditional logic.   │
│                                                                 │
│                                                                 │
│  FOR TIMING: Use middleware. You want EVERY request timed,      │
│  not just specific endpoints.                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Building a Timing Middleware

**The `@app.middleware("http")` pattern:**

A middleware function in FastAPI receives two things: the incoming `request` object, and a `call_next` function. You call `call_next(request)` to pass the request to the next layer (eventually reaching your route handler), and it returns the `response`. Everything *before* `call_next` runs on the way in. Everything *after* runs on the way out.

```python
# middleware/timing.py

import time
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware

# Approach 1: Using the @app.middleware decorator (simplest)
# You register this on your app object directly.

async def add_timing_middleware(request: Request, call_next) -> Response:
    """Measure and log the duration of every request."""
    
    start_time = time.perf_counter()
    # ▲ time.perf_counter() — NOT time.time()
    # perf_counter is monotonic and high-resolution.
    # time.time() can jump forward/backward (NTP adjustments).
    # For measuring durations, always use perf_counter.

    response: Response = await call_next(request)
    # ▲ This is where your route handler runs.
    # Everything between start and here IS the request duration.

    duration_ms = (time.perf_counter() - start_time) * 1000

    # Attach timing info to response header (visible in DevTools)
    response.headers["X-Process-Time-Ms"] = f"{duration_ms:.2f}"

    return response
```

**Register it on your app:**

```python
# main.py

from fastapi import FastAPI
from middleware.timing import add_timing_middleware

app = FastAPI()

# Register the middleware
app.middleware("http")(add_timing_middleware)

# Or equivalently, using the decorator directly:
#
# @app.middleware("http")
# async def timing_middleware(request: Request, call_next):
#     ...  (same body as above)
```

**What this gives you:**

```
┌─────────────────────────────────────────────────────────────────┐
│                WHAT TIMING MIDDLEWARE CAPTURES                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    start_time                                   │
│                        │                                        │
│  Request in ──────────▶│                                        │
│                        ▼                                        │
│              ┌───────────────────┐                               │
│              │  await call_next  │  ← entire handler + deps     │
│              │  ┌─────────────┐  │                               │
│              │  │ Depends()   │  │                               │
│              │  │ Route logic │  │                               │
│              │  │ DB queries  │  │                               │
│              │  │ API calls   │  │                               │
│              │  │ Serialize   │  │                               │
│              │  └─────────────┘  │                               │
│              └───────────────────┘                               │
│                        │                                        │
│                        ▼                                        │
│                    end_time                                     │
│                                                                 │
│  duration = end_time - start_time                               │
│                                                                 │
│  This captures EVERYTHING from the moment the request           │
│  enters your application to the moment the response is ready.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why `time.perf_counter()` and not `time.time()`?**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  time.time()                                                    │
│  ├─ Wall-clock time (seconds since epoch)                       │
│  ├─ CAN GO BACKWARDS (NTP sync, leap seconds, DST)             │
│  ├─ Resolution: ~1ms on most systems                            │
│  └─ Use for: timestamps ("when did this happen?")               │
│                                                                 │
│  time.perf_counter()                                            │
│  ├─ Monotonic high-resolution counter                           │
│  ├─ NEVER GOES BACKWARDS                                        │
│  ├─ Resolution: nanoseconds on most systems                     │
│  └─ Use for: measuring DURATIONS ("how long did this take?")    │
│                                                                 │
│  For performance measurement, ALWAYS use perf_counter().        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 Structured Performance Logging

**Printing to console isn't enough. You need structured, searchable logs.**

You were introduced to structlog in Week 3. Now we use it for performance data. Structured logs produce JSON that can be filtered, searched, and aggregated by log management tools in production.

```python
# middleware/timing.py (enhanced with structured logging)

import time
import structlog
from fastapi import Request, Response

logger = structlog.get_logger()


async def add_timing_middleware(request: Request, call_next) -> Response:
    start_time = time.perf_counter()

    response: Response = await call_next(request)

    duration_ms = (time.perf_counter() - start_time) * 1000

    # Structured log — every field is searchable/filterable
    await logger.ainfo(
        "request_completed",
        method=request.method,
        path=request.url.path,
        status_code=response.status_code,
        duration_ms=round(duration_ms, 2),
        # Tag slow requests so they're easy to find
        slow=duration_ms > 500,
    )

    # Warn on truly slow requests
    if duration_ms > 1000:
        await logger.awarn(
            "slow_request",
            method=request.method,
            path=request.url.path,
            duration_ms=round(duration_ms, 2),
        )

    response.headers["X-Process-Time-Ms"] = f"{duration_ms:.2f}"
    return response
```

**What this produces in your logs:**

```json
{
  "event": "request_completed",
  "method": "GET",
  "path": "/organizations/42/dashboard",
  "status_code": 200,
  "duration_ms": 1101.34,
  "slow": true,
  "timestamp": "2026-02-14T10:23:45.123Z"
}
```

```json
{
  "event": "slow_request",
  "method": "GET",
  "path": "/organizations/42/dashboard",
  "duration_ms": 1101.34,
  "timestamp": "2026-02-14T10:23:45.123Z",
  "level": "warning"
}
```

**Why structured JSON logs matter:**

```
┌─────────────────────────────────────────────────────────────────┐
│             UNSTRUCTURED VS STRUCTURED LOGS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UNSTRUCTURED (print / basic logging):                          │
│  ──────────────────────────────────────                         │
│  INFO: GET /dashboard completed in 1101ms                       │
│                                                                 │
│  ❌ How do you find all requests slower than 500ms?             │
│  ❌ How do you calculate average time for /dashboard?           │
│  ❌ How do you alert on 5xx responses?                          │
│  Answer: painful regex parsing.                                 │
│                                                                 │
│                                                                 │
│  STRUCTURED (structlog → JSON):                                 │
│  ──────────────────────────────                                 │
│  {"event":"request_completed","path":"/dashboard",              │
│   "duration_ms":1101.34,"status_code":200}                      │
│                                                                 │
│  ✅ Find slow requests: filter where duration_ms > 500          │
│  ✅ Average time: aggregate duration_ms where path="/dashboard" │
│  ✅ Alert on 5xx: filter where status_code >= 500               │
│  Answer: simple queries.                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Filtering slow requests (the doctor reading blood work results):**

```python
# In production, you'd use a log aggregation tool (Datadog, ELK, etc.)
# For development, you can filter your JSON logs with jq:

# Find all requests slower than 500ms:
#   cat logs.json | jq 'select(.duration_ms > 500)'

# Find the 10 slowest endpoints:
#   cat logs.json | jq -s 'sort_by(-.duration_ms) | .[0:10]'

# Average response time per endpoint:
#   cat logs.json | jq -s 'group_by(.path) | map({
#     path: .[0].path,
#     avg_ms: (map(.duration_ms) | add / length)
#   })'
```

---

## 2.4 Server-Timing Header

**A standard HTTP mechanism for reporting backend timing to the browser.**

The `Server-Timing` header is part of the W3C specification. Browsers (Chrome DevTools → Network tab → Timing section) can read it and display a breakdown of where time was spent on the server. This means your frontend colleagues (or you, debugging) can see the backend's timing breakdown without reading logs.

```python
# middleware/timing.py (with Server-Timing header)

import time
from contextvars import ContextVar
from fastapi import Request, Response

# Context variable to collect timing data from anywhere in the request
# (We'll use this in Part 3 for database timing too)
request_timings: ContextVar[dict[str, float]] = ContextVar(
    "request_timings", default={}
)


async def add_timing_middleware(request: Request, call_next) -> Response:
    # Reset timings for this request
    timings: dict[str, float] = {}
    request_timings.set(timings)

    start_time = time.perf_counter()

    response: Response = await call_next(request)

    total_ms = (time.perf_counter() - start_time) * 1000
    timings["total"] = total_ms

    # Build Server-Timing header
    # Format: metric;desc="Description";dur=123.45
    timing_parts = []
    for name, duration in timings.items():
        timing_parts.append(f'{name};dur={duration:.2f}')

    response.headers["Server-Timing"] = ", ".join(timing_parts)
    response.headers["X-Process-Time-Ms"] = f"{total_ms:.2f}"
    return response
```

**Adding timing from inside your route handler:**

```python
# Now from ANYWHERE in the request lifecycle, you can record timings:

import time
from middleware.timing import request_timings

async def get_org_dashboard(
    org_id: int,
    db: AsyncSession = Depends(get_db),
):
    timings = request_timings.get()

    # Time the database work
    db_start = time.perf_counter()
    org = await db.get(Organization, org_id)
    projects = await fetch_projects(db, org_id)
    timings["db"] = (time.perf_counter() - db_start) * 1000

    # Time the external API calls
    api_start = time.perf_counter()
    analytics = await fetch_analytics(org_id)
    timings["external_api"] = (time.perf_counter() - api_start) * 1000

    # Time serialization
    serial_start = time.perf_counter()
    result = build_dashboard_response(org, projects, analytics)
    timings["serialization"] = (time.perf_counter() - serial_start) * 1000

    return result
```

**What the browser sees:**

```
┌─────────────────────────────────────────────────────────────────┐
│        SERVER-TIMING IN BROWSER DEVTOOLS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Response Headers:                                              │
│  Server-Timing: db;dur=461.23, external_api;dur=600.45,         │
│                 serialization;dur=12.30, total;dur=1101.34      │
│                                                                 │
│  Chrome DevTools → Network → select request → Timing:           │
│                                                                 │
│  Server Timing                                                  │
│  ├─ db                 461.23ms  ██████████████████████          │
│  ├─ external_api       600.45ms  ████████████████████████████    │
│  ├─ serialization       12.30ms  █                              │
│  └─ total             1101.34ms                                 │
│                                                                 │
│  Now the BROWSER tells you where the time went.                 │
│  No log searching needed for quick debugging.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key detail: `ContextVar` for request-scoped state**

> "Notice we used `ContextVar` (from Python's `contextvars` module), not a global variable. Why? In async code, multiple requests are in-flight concurrently on the same thread. A global dictionary would be shared by ALL concurrent requests — total chaos. `ContextVar` gives each async task chain its own isolated copy. Think of it as a variable that 'follows' the request through all the `await` calls without leaking between concurrent requests."

```
┌─────────────────────────────────────────────────────────────────┐
│                   WHY ContextVar?                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ GLOBAL DICT:                                                │
│                                                                 │
│  timings = {}   ← shared by ALL concurrent requests!            │
│                                                                 │
│  Request A writes: timings["db"] = 50ms                         │
│  Request B writes: timings["db"] = 300ms   ← overwrites A!     │
│  Request A reads:  timings["db"] → 300ms   ← WRONG             │
│                                                                 │
│                                                                 │
│  ✅ ContextVar:                                                 │
│                                                                 │
│  request_timings: ContextVar[dict] = ContextVar(...)            │
│                                                                 │
│  Request A sees: {"db": 50ms}      ← its own copy              │
│  Request B sees: {"db": 300ms}     ← its own copy              │
│  No interference.                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: DATABASE QUERY PROFILING

> "The timing middleware tells you which ENDPOINTS are slow. But it doesn't tell you WHY. For database-heavy applications — which is nearly every backend — the next diagnostic is query profiling. This is the blood work."

## 3.1 SQLAlchemy Query Logging

**The simplest diagnostic: see every SQL query that runs.**

You've used `echo=True` during development (Week 6). That prints every query to the console. But in production-like profiling, we need something more targeted. SQLAlchemy provides an **event system** that lets us hook into query execution.

```python
# db/profiling.py

import time
import logging
from sqlalchemy import event
from sqlalchemy.ext.asyncio import AsyncEngine

logger = logging.getLogger("sqlalchemy.profiling")


def setup_query_logging(engine: AsyncEngine) -> None:
    """Attach profiling listeners to a SQLAlchemy engine."""

    sync_engine = engine.sync_engine  
    # ▲ Event listeners attach to the sync engine underneath
    # the async wrapper. This is a SQLAlchemy implementation detail.

    @event.listens_for(sync_engine, "before_cursor_execute")
    def before_cursor_execute(
        conn, cursor, statement, parameters, context, executemany
    ):
        # Store the start time on the connection object
        conn.info["query_start_time"] = time.perf_counter()

    @event.listens_for(sync_engine, "after_cursor_execute")
    def after_cursor_execute(
        conn, cursor, statement, parameters, context, executemany
    ):
        duration_ms = (
            time.perf_counter() - conn.info["query_start_time"]
        ) * 1000

        # Log every query with its duration
        logger.debug(
            "Query executed: duration=%.2fms | %s",
            duration_ms,
            statement[:200],  # Truncate long queries
        )

        # Warn on slow queries
        if duration_ms > 100:
            logger.warning(
                "SLOW QUERY: duration=%.2fms | %s",
                duration_ms,
                statement[:500],
            )
```

**Register it when creating your engine:**

```python
# db/session.py

from sqlalchemy.ext.asyncio import create_async_engine
from db.profiling import setup_query_logging

engine = create_async_engine(settings.DATABASE_URL)
setup_query_logging(engine)  # ← Attach the profiling listeners
```

**What you see in your logs:**

```
DEBUG: Query executed: duration=4.23ms | SELECT org.id, org.name ... WHERE org.id = $1
DEBUG: Query executed: duration=5.67ms | SELECT project.id, ... WHERE project.org_id = $1
DEBUG: Query executed: duration=4.12ms | SELECT task.id, ... WHERE task.project_id = $1
DEBUG: Query executed: duration=3.89ms | SELECT user.id, ... WHERE user.id = $1
DEBUG: Query executed: duration=4.01ms | SELECT user.id, ... WHERE user.id = $1
DEBUG: Query executed: duration=3.95ms | SELECT user.id, ... WHERE user.id = $1
... (89 more lines)

WARNING: 92 queries in this request!  ← Now the N+1 is VISIBLE
```

---

## 3.2 Counting Queries Per Request

**Knowing individual query times isn't enough. You need to know how many queries each request triggers.**

This is the single most important diagnostic for database performance. Combine the timing middleware from Part 2 with a query counter using `ContextVar`:

```python
# db/profiling.py (enhanced with per-request counting)

import time
from contextvars import ContextVar
from dataclasses import dataclass, field
from sqlalchemy import event
from sqlalchemy.ext.asyncio import AsyncEngine


@dataclass
class QueryStats:
    """Track query statistics for a single request."""
    count: int = 0
    total_duration_ms: float = 0.0
    queries: list[dict] = field(default_factory=list)

    def record(self, statement: str, duration_ms: float) -> None:
        self.count += 1
        self.total_duration_ms += duration_ms
        self.queries.append({
            "sql": statement[:200],
            "duration_ms": round(duration_ms, 2),
        })


# ContextVar: each request gets its own QueryStats
request_query_stats: ContextVar[QueryStats | None] = ContextVar(
    "request_query_stats", default=None
)


def setup_query_profiling(engine: AsyncEngine) -> None:
    sync_engine = engine.sync_engine

    @event.listens_for(sync_engine, "before_cursor_execute")
    def before_cursor_execute(
        conn, cursor, statement, parameters, context, executemany
    ):
        conn.info["query_start_time"] = time.perf_counter()

    @event.listens_for(sync_engine, "after_cursor_execute")
    def after_cursor_execute(
        conn, cursor, statement, parameters, context, executemany
    ):
        duration_ms = (
            time.perf_counter() - conn.info["query_start_time"]
        ) * 1000

        # Record to the current request's stats (if tracked)
        stats = request_query_stats.get()
        if stats is not None:
            stats.record(statement, duration_ms)
```

**Integrate with your timing middleware:**

```python
# middleware/timing.py (enhanced with query stats)

import structlog
from db.profiling import QueryStats, request_query_stats
from middleware.timing import request_timings

logger = structlog.get_logger()


async def add_performance_middleware(request: Request, call_next) -> Response:
    # Initialize per-request tracking
    timings: dict[str, float] = {}
    request_timings.set(timings)

    stats = QueryStats()
    request_query_stats.set(stats)  # ← Start counting queries

    start_time = time.perf_counter()

    response: Response = await call_next(request)

    total_ms = (time.perf_counter() - start_time) * 1000

    # Log with query count information
    await logger.ainfo(
        "request_completed",
        method=request.method,
        path=request.url.path,
        status_code=response.status_code,
        duration_ms=round(total_ms, 2),
        query_count=stats.count,                          # ← NEW
        query_total_ms=round(stats.total_duration_ms, 2), # ← NEW
        slow=total_ms > 500,
    )

    # Warn on high query count — the N+1 alarm
    if stats.count > 10:
        await logger.awarn(
            "high_query_count",
            path=request.url.path,
            query_count=stats.count,
            query_total_ms=round(stats.total_duration_ms, 2),
            queries=stats.queries,  # Full list for debugging
        )

    # Add to Server-Timing header
    response.headers["Server-Timing"] = (
        f'db;dur={stats.total_duration_ms:.2f},'
        f'total;dur={total_ms:.2f}'
    )
    response.headers["X-Query-Count"] = str(stats.count)
    response.headers["X-Process-Time-Ms"] = f"{total_ms:.2f}"

    return response
```

**What you get in your logs now:**

```json
{
  "event": "request_completed",
  "path": "/organizations/42/dashboard",
  "duration_ms": 1101.34,
  "query_count": 92,
  "query_total_ms": 461.23,
  "slow": true
}
```

```json
{
  "event": "high_query_count",
  "path": "/organizations/42/dashboard",
  "query_count": 92,
  "query_total_ms": 461.23,
  "queries": [
    {"sql": "SELECT org.id, ...", "duration_ms": 4.23},
    {"sql": "SELECT project.id, ...", "duration_ms": 5.67},
    "... (90 more)"
  ]
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  NOW YOU CAN SEE:                                               │
│                                                                 │
│  1. Which endpoints are slow        (duration_ms)               │
│  2. Whether the DB is the cause     (query_total_ms / total)    │
│  3. Whether N+1 is the pattern      (query_count > 10)          │
│  4. Exactly which queries ran       (queries list)              │
│                                                                 │
│  This is your blood work report. Now you can diagnose.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Catching N+1 in Production

You learned about N+1 in Week 6 and how to fix it with `joinedload` and `selectinload`. The problem isn't that you don't know the fix — it's that **N+1 sneaks back in** every time someone writes a new query or modifies an endpoint. You need a way to CATCH it automatically.

**The pattern: recognize N+1 from query logs.**

```
┌─────────────────────────────────────────────────────────────────┐
│             HOW TO SPOT N+1 IN QUERY LOGS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  N+1 has a FINGERPRINT:                                         │
│  The same query structure repeated many times with              │
│  different parameter values.                                    │
│                                                                 │
│  NORMAL (2 queries):                                            │
│  ──────────────────                                             │
│  SELECT project.* FROM project WHERE project.org_id = 42        │
│  SELECT task.* FROM task WHERE task.project_id IN (1,2,3,4,5)   │
│                                                                 │
│  N+1 (6 queries — 1 + 5):                                      │
│  ─────────────────────────                                      │
│  SELECT project.* FROM project WHERE project.org_id = 42        │
│  SELECT task.* FROM task WHERE task.project_id = 1   ← same     │
│  SELECT task.* FROM task WHERE task.project_id = 2   ← query    │
│  SELECT task.* FROM task WHERE task.project_id = 3   ← repeats  │
│  SELECT task.* FROM task WHERE task.project_id = 4   ← with     │
│  SELECT task.* FROM task WHERE task.project_id = 5   ← diff id  │
│                                                                 │
│  THE FIX (Week 6 refresher — use eager loading):                │
│                                                                 │
│  # Before (N+1):                                                │
│  stmt = select(Project).where(Project.org_id == org_id)         │
│                                                                 │
│  # After (2 queries with selectinload):                         │
│  stmt = (                                                       │
│      select(Project)                                            │
│      .where(Project.org_id == org_id)                           │
│      .options(selectinload(Project.tasks))                      │
│  )                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Demonstration — before and after with real numbers:**

```python
# BEFORE: N+1 (from Part 1's dashboard endpoint)
# 10 projects, 8 tasks each = 92 queries, ~461ms

@router.get("/organizations/{org_id}/dashboard")
async def get_org_dashboard(org_id: int, db: AsyncSession = Depends(get_db)):
    stmt = select(Project).where(Project.org_id == org_id)
    result = await db.execute(stmt)
    projects = result.scalars().all()

    for project in projects:  # ← N+1 STARTS HERE
        tasks_stmt = select(Task).where(Task.project_id == project.id)
        tasks = (await db.execute(tasks_stmt)).scalars().all()

        for task in tasks:  # ← NESTED N+1
            assignee = await db.get(User, task.assignee_id)
            ...
```

```python
# AFTER: Eager loading (2 queries, ~15ms)

@router.get("/organizations/{org_id}/dashboard")
async def get_org_dashboard(org_id: int, db: AsyncSession = Depends(get_db)):
    stmt = (
        select(Project)
        .where(Project.org_id == org_id)
        .options(
            selectinload(Project.tasks)       # Load tasks in 1 batch query
            .selectinload(Task.assignee)       # Load assignees in 1 batch query
        )
    )
    result = await db.execute(stmt)
    projects = result.scalars().unique().all()
    # ▲ .unique() is needed when using joined/selectin loading
    #   to deduplicate parent rows. You saw this in Week 6.

    # Now project.tasks and task.assignee are already loaded.
    # No additional queries needed.
    for project in projects:
        for task in project.tasks:
            assignee_name = task.assignee.name  # ← No query! Already loaded.
            ...
```

**The results (doctor's follow-up):**

```
┌─────────────────────────────────────────────────────────────────┐
│               BEFORE vs AFTER (N+1 fix)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  10 projects, 8 tasks each:                                     │
│                                                                 │
│                    BEFORE              AFTER                    │
│  Queries:          92                  3                        │
│  DB time:          461ms               15ms                     │
│  Total endpoint:   1,101ms             655ms                    │
│  Improvement:                          30× fewer queries        │
│                                        ~40% faster overall      │
│                                                                 │
│  50 projects, 20 tasks each:                                    │
│                                                                 │
│                    BEFORE              AFTER                    │
│  Queries:          1,052               3                        │
│  DB time:          5,260ms             42ms                     │
│  Total endpoint:   5,900ms             682ms                    │
│  Improvement:                          350× fewer queries       │
│                                        ~88% faster overall      │
│                                                                 │
│  The fix was ONE LINE (adding .options(selectinload(...))).     │
│  But you could only justify and verify it because you           │
│  MEASURED the query count before and after.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Query Count Assertions in Tests

**The permanent safeguard: make N+1 regressions fail your test suite.**

This is the most powerful technique in this entire lecture for long-term health. You write a test that asserts: "this endpoint must not execute more than N queries." If someone adds a loop that causes N+1, the test fails in CI before it reaches production.

```python
# tests/conftest.py

import pytest
from db.profiling import QueryStats, request_query_stats


@pytest.fixture
def query_counter():
    """Fixture that tracks query count during a test."""
    stats = QueryStats()
    request_query_stats.set(stats)
    yield stats
    # After test, stats.count contains the total queries executed
```

**Using it in your tests:**

```python
# tests/test_dashboard.py

import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_dashboard_query_count(
    async_client: AsyncClient,
    query_counter: QueryStats,
    seed_org_with_projects,   # Fixture: creates org with 10 projects, 8 tasks each
):
    """Dashboard endpoint must not trigger N+1 queries."""
    response = await async_client.get("/organizations/1/dashboard")
    assert response.status_code == 200

    # THE ASSERTION THAT CATCHES N+1:
    assert query_counter.count <= 5, (
        f"Dashboard executed {query_counter.count} queries! "
        f"Expected ≤5. Likely N+1 regression.\n"
        f"Queries:\n"
        + "\n".join(
            f"  {q['sql'][:100]}" for q in query_counter.queries
        )
    )


@pytest.mark.asyncio
async def test_dashboard_with_many_projects(
    async_client: AsyncClient,
    query_counter: QueryStats,
    seed_org_with_many_projects,  # 50 projects, 20 tasks each
):
    """Query count must NOT scale with data volume."""
    response = await async_client.get("/organizations/1/dashboard")
    assert response.status_code == 200

    # Same budget as the small test — that's the point!
    # If this fails, queries scale with data volume = N+1.
    assert query_counter.count <= 5, (
        f"Query count scales with data! Got {query_counter.count} queries."
    )
```

**Why this is so powerful:**

```
┌─────────────────────────────────────────────────────────────────┐
│         QUERY COUNT ASSERTIONS IN CI                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without query count tests:                                     │
│  ──────────────────────────                                     │
│  1. Developer adds new field to dashboard                       │
│  2. Adds a loop to fetch related data                           │
│  3. Tests pass (correctness is fine)                            │
│  4. Code review doesn't catch it (the code looks fine)          │
│  5. Deployed to production                                      │
│  6. 2 weeks later: "Why is the dashboard slow?"                 │
│  7. Debug, find N+1, fix, deploy, apologize                     │
│                                                                 │
│                                                                 │
│  With query count tests:                                        │
│  ───────────────────────                                        │
│  1. Developer adds new field to dashboard                       │
│  2. Adds a loop to fetch related data                           │
│  3. CI FAILS: "Dashboard executed 47 queries! Expected ≤5."     │
│  4. Developer sees the problem immediately                      │
│  5. Adds selectinload, test passes                              │
│  6. Deployed with zero performance regression                   │
│                                                                 │
│  The test caught the bug before a human had to.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: RESPONSE OPTIMIZATION TECHNIQUES

> "Parts 2 and 3 were the diagnosis. Now we prescribe. But remember: **targeted treatments only.** We know exactly where the time goes. We fix exactly those things."

## 4.1 Response Compression (GzipMiddleware)

**Reduce what you send over the wire.**

JSON responses from APIs can be large, especially for list endpoints that return many records. Gzip compression typically reduces JSON payloads by 70–90%. The CPU cost of compression is almost always worth the reduced transfer time, especially for clients on slower networks.

```python
# main.py

from starlette.middleware.gzip import GzipMiddleware

app = FastAPI()

# Add gzip compression — responses larger than 500 bytes get compressed
app.add_middleware(GzipMiddleware, minimum_size=500)
# ▲ minimum_size: don't bother compressing tiny responses.
#   The compression overhead isn't worth it for small payloads.
```

That's it. Two lines. Starlette handles it.

**How it works:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  GZIP COMPRESSION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT COMPRESSION:                                           │
│                                                                 │
│  Server ──── 250KB JSON ────────────────────────▶ Client        │
│              ▲                                                  │
│              │ Full payload over the wire                        │
│              │ Slow on mobile/weak connections                   │
│                                                                 │
│                                                                 │
│  WITH GZIP:                                                     │
│                                                                 │
│  Server ─┬── Compress ──── 35KB gzipped ────▶ Client            │
│          │   (~2ms CPU)    ▲                   │                 │
│          │                 │ 86% smaller!      ├── Decompress    │
│          │                 │                   │   (browser      │
│          │                 │                   │    does this     │
│          │                 │                   │    automatically)│
│                                                                 │
│  TRADEOFF:                                                      │
│  ├─ Cost:   ~2-5ms CPU per response (compression)               │
│  ├─ Saving: 70-90% less data transferred                        │
│  └─ Net:    Almost always a win for JSON payloads > 1KB         │
│                                                                 │
│  WHEN IT DOESN'T HELP:                                          │
│  ├─ Tiny responses (< 500 bytes) — overhead > savings           │
│  ├─ Already compressed content (images, binary files)           │
│  └─ Internal microservice calls on fast networks                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Verifying it works — check response headers:**

```python
# In a test or curl:
# curl -H "Accept-Encoding: gzip" http://localhost:8000/organizations/1/dashboard -v
#
# Response headers should include:
#   Content-Encoding: gzip
#   Content-Length: 35412  (vs 250000 uncompressed)

# Or in your test suite:
@pytest.mark.asyncio
async def test_gzip_compression(async_client: AsyncClient):
    response = await async_client.get(
        "/organizations/1/dashboard",
        headers={"Accept-Encoding": "gzip"},
    )
    assert response.status_code == 200
    # httpx automatically decompresses, but we can check
    # the raw response headers were set
    assert response.headers.get("content-encoding") == "gzip"
```

---

## 4.2 Parallel Operations in Endpoints

**You learned `asyncio.gather` in Week 1. Here's where it pays off.**

Look at Section B from the opening demonstration again:

```python
# BEFORE: Sequential external calls (600ms)

async with httpx.AsyncClient(timeout=5.0) as client:
    analytics_resp = await client.get(f"https://analytics.example.com/...")
    # ▲ Waits 280ms. CPU idle. Nothing else happening.
    
async with httpx.AsyncClient(timeout=5.0) as client:
    weather_resp = await client.get(f"https://api.weather.example.com/...")
    # ▲ Waits 320ms AFTER analytics finishes. Total: 600ms.
    
# Also: creating a new AsyncClient per call is wasteful.
# Each one opens a new connection pool. Use a shared client.
```

```python
# AFTER: Parallel external calls + shared client (320ms)

# Shared client (injected via Depends, or module-level)
# You learned httpx.AsyncClient in Week 8.

async def get_org_dashboard(
    org_id: int,
    db: AsyncSession = Depends(get_db),
    http_client: httpx.AsyncClient = Depends(get_http_client),
):
    # ... DB queries (already optimized) ...

    # Fetch external data IN PARALLEL
    analytics_resp, weather_resp = await asyncio.gather(
        http_client.get(f"https://analytics.example.com/org/{org_id}/summary"),
        http_client.get(f"https://api.weather.example.com/current?city={org.city}"),
    )
    # ▲ Both requests start at the same time.
    # Total time = max(280ms, 320ms) = 320ms, not 600ms.
```

**Visualize the difference:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SEQUENTIAL:                                                    │
│                                                                 │
│  Time: 0ms     280ms          600ms                             │
│        │        │               │                               │
│  Analytics:  [══════════]                                       │
│  Weather:                [══════════]                            │
│                                                                 │
│  Total: 600ms                                                   │
│                                                                 │
│                                                                 │
│  PARALLEL (asyncio.gather):                                     │
│                                                                 │
│  Time: 0ms          320ms                                       │
│        │              │                                         │
│  Analytics:  [════════]                                         │
│  Weather:    [══════════]                                       │
│                                                                 │
│  Total: 320ms (time of the slower call)                         │
│                                                                 │
│  Savings: 280ms (47% faster)                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When can you parallelize?**

```
┌─────────────────────────────────────────────────────────────────┐
│          CAN I RUN THESE IN PARALLEL?                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ YES — if operations are INDEPENDENT:                        │
│  ├─ Fetch analytics + Fetch weather (no dependency)             │
│  ├─ Cache lookup + DB query (if you don't need cache to skip    │
│  │   the DB query — in that case they ARE dependent!)           │
│  ├─ Send notification + Log audit event                         │
│  └─ Fetch from 3 different external APIs                        │
│                                                                 │
│  ❌ NO — if one operation DEPENDS on another's result:          │
│  ├─ Get user → then get user's permissions (need user_id first) │
│  ├─ Check cache → if miss, THEN query DB (conditional)          │
│  └─ Validate input → then process (process needs valid input)   │
│                                                                 │
│  THE RULE:                                                      │
│  "If operation B does NOT need the result of operation A,       │
│   they can run in parallel."                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Combining parallel I/O with error handling:**

```python
# Production pattern: parallel calls with individual error handling

async def get_org_dashboard(
    org_id: int,
    db: AsyncSession = Depends(get_db),
    http_client: httpx.AsyncClient = Depends(get_http_client),
):
    projects = await fetch_projects_optimized(db, org_id)

    # Parallel external calls with graceful degradation
    analytics_resp, weather_resp = await asyncio.gather(
        http_client.get(f"https://analytics.example.com/org/{org_id}"),
        http_client.get(f"https://api.weather.example.com/?city={org.city}"),
        return_exceptions=True,  # ← Don't crash if one fails
    )

    # Handle individual failures gracefully
    analytics = (
        analytics_resp.json()
        if isinstance(analytics_resp, httpx.Response)
           and analytics_resp.status_code == 200
        else None  # Degrade gracefully: dashboard still works
    )

    weather = (
        weather_resp.json()
        if isinstance(weather_resp, httpx.Response)
           and weather_resp.status_code == 200
        else None
    )

    return {
        "projects": projects,
        "analytics": analytics,   # null if analytics service was down
        "weather": weather,        # null if weather service was down
    }
```

---

## 4.3 The Optimization Toolbox

**A quick reference of every technique available to you after this course:**

```
┌─────────────────────────────────────────────────────────────────┐
│           THE OPTIMIZATION TOOLBOX                              │
│           (ordered by: try this first → try this last)          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. FIX N+1 QUERIES (Week 6 + this lecture)                     │
│     Effort: Low  │  Impact: Massive (10x-100x fewer queries)    │
│     How: selectinload / joinedload / subqueryload               │
│     Always try this FIRST. It's almost always the biggest win.  │
│                                                                 │
│  2. ADD DATABASE INDEXES (Week 7)                               │
│     Effort: Low  │  Impact: High (10x-1000x on filtered reads) │
│     How: EXPLAIN ANALYZE → find seq scans → add index           │
│     Second thing to check. You already know how.                │
│                                                                 │
│  3. ADD CACHING (Week 10)                                       │
│     Effort: Medium  │  Impact: High (eliminate repeated work)   │
│     How: Redis cache-aside for expensive/repeated queries       │
│     Third: if the same data is requested often, cache it.       │
│                                                                 │
│  4. PARALLELIZE I/O (Week 1 + this lecture)                     │
│     Effort: Low  │  Impact: Medium (depends on # of operations) │
│     How: asyncio.gather for independent operations              │
│     Quick win when you have multiple independent I/O calls.     │
│                                                                 │
│  5. COMPRESS RESPONSES (this lecture)                            │
│     Effort: Minimal  │  Impact: Low-Medium (network-bound)      │
│     How: GzipMiddleware, 2 lines                                │
│     Easy, helps most for large JSON responses.                  │
│                                                                 │
│  6. OPTIMIZE QUERIES (Week 7)                                   │
│     Effort: High  │  Impact: Varies                             │
│     How: Rewrite slow queries, denormalize, materialized views  │
│     Only after the above are exhausted.                         │
│                                                                 │
│  7. SCALE INFRASTRUCTURE (Week 16)                              │
│     Effort: High  │  Impact: High (but expensive)               │
│     How: Read replicas, horizontal scaling, CDN                 │
│     Last resort. You'll learn this in system design.            │
│                                                                 │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ RULE: Exhaust the cheap optimizations before reaching   │    │
│  │ for the expensive ones. A missing index (5 minutes to   │    │
│  │ add) often beats adding a cache layer (2 days of work). │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: LOAD TESTING WITH LOCUST

## 5.1 Why Load Test? (The Empty Restaurant Problem)

**Everything so far measures one request at a time. Production has hundreds.**

```
┌─────────────────────────────────────────────────────────────────┐
│            THE EMPTY RESTAURANT PROBLEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You test your new restaurant with ONE customer.                │
│                                                                 │
│  Customer sits down. Waiter comes immediately.                  │
│  Food arrives in 10 minutes. Customer is happy. ⭐⭐⭐⭐⭐          │
│                                                                 │
│  "Our restaurant is fast! Ready for opening night!"             │
│                                                                 │
│  Opening night: 200 customers arrive.                           │
│                                                                 │
│  ├─ 3 waiters for 200 people (understaffed)                     │
│  ├─ Kitchen can only cook 5 meals at a time (bottleneck)        │
│  ├─ Tables too close together, movement is slow (contention)    │
│  ├─ Customer #150 waits 45 minutes. Leaves a 1-star review.    │
│  └─ Customer #200 can't even get in (connection refused).       │
│                                                                 │
│  WHAT WENT WRONG:                                               │
│  You tested with 1 customer. Your system performed under        │
│  ZERO CONTENTION. Real performance only shows under LOAD.       │
│                                                                 │
│  WHAT LOAD TESTING REVEALS:                                     │
│  ├─ Connection pool exhaustion (all DB connections in use)      │
│  ├─ Lock contention (multiple requests hitting same rows)       │
│  ├─ Memory leaks (usage growing with each request)              │
│  ├─ Event loop blocking (one slow sync call freezes everyone)   │
│  └─ Cascading failures (one slow service backs up everything)   │
│                                                                 │
│  These problems are INVISIBLE to single-request testing.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Your First Locust Test

**Locust is a Python-based load testing tool. You write test scenarios as Python code.**

That matters: it's not a config file or a GUI tool. It's Python. You already know Python. You can use logic, loops, conditionals, and everything you know.

**Install it:**

```bash
pip install locust
```

**Write your first locustfile:**

```python
# locustfile.py — place in your project root

from locust import HttpUser, task, between


class DashboardUser(HttpUser):
    """Simulates a user who logs in and views dashboards."""

    # Wait 1-3 seconds between tasks (simulates think time)
    # Real users don't click every millisecond.
    wait_time = between(1, 3)

    # ─── TASKS ────────────────────────────────────────────

    @task(3)  # Weight: 3x more likely than weight-1 tasks
    def view_dashboard(self):
        """Most common action: view the org dashboard."""
        self.client.get(
            "/organizations/1/dashboard",
            headers=self.auth_headers,
            name="/organizations/{id}/dashboard",
            # ▲ 'name' groups URLs with different IDs into
            #   one stat line. Without it, /orgs/1/dashboard
            #   and /orgs/2/dashboard would be separate entries.
        )

    @task(2)
    def list_projects(self):
        """Second most common: browse projects."""
        self.client.get(
            "/organizations/1/projects",
            headers=self.auth_headers,
            name="/organizations/{id}/projects",
        )

    @task(1)
    def create_task(self):
        """Least common: create a new task."""
        self.client.post(
            "/organizations/1/projects/1/tasks",
            json={
                "title": "Load test task",
                "description": "Created during load testing",
                "status": "todo",
            },
            headers=self.auth_headers,
            name="/organizations/{id}/projects/{id}/tasks",
        )

    # ─── LIFECYCLE ────────────────────────────────────────

    def on_start(self):
        """Called when a simulated user starts. Log in once."""
        response = self.client.post(
            "/auth/login",
            json={"email": "loadtest@example.com", "password": "testpass123"},
        )
        token = response.json()["access_token"]
        self.auth_headers = {"Authorization": f"Bearer {token}"}
```

**Anatomy of a locustfile:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  LOCUST ANATOMY                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  class DashboardUser(HttpUser):                                 │
│  ▲                    ▲                                         │
│  │                    │                                         │
│  Your user type       Built-in base class with HTTP client      │
│                                                                 │
│  wait_time = between(1, 3)                                      │
│  ▲                                                              │
│  How long a "user" waits between actions.                       │
│  Simulates real human think time.                               │
│  Without this, locust hammers your API unrealistically.         │
│                                                                 │
│  @task(3)                                                       │
│  ▲     ▲                                                        │
│  │     Weight: this task is picked 3x more often                │
│  Marks this method as a user action                             │
│                                                                 │
│  self.client.get(...)                                           │
│  ▲                                                              │
│  Built-in HTTP client (similar to httpx/requests).              │
│  Locust automatically tracks timing and success/failure.        │
│                                                                 │
│  on_start(self):                                                │
│  ▲                                                              │
│  Runs once when a virtual user spawns.                          │
│  Use it for login, setup, getting tokens.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Run it:**

```bash
# Start locust (assumes your FastAPI app is running on :8000)
locust -f locustfile.py --host=http://localhost:8000

# Locust starts a web UI at http://localhost:8089
# From the web UI, you configure:
#   - Number of users (e.g., 50)
#   - Spawn rate (e.g., 5 users/second)
#   - Then click "Start swarming"
```

```
┌─────────────────────────────────────────────────────────────────┐
│                LOCUST WEB UI                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────┐          │
│  │  Number of users: [50    ]                        │          │
│  │  Spawn rate:      [5     ] users/sec              │          │
│  │  Host:            http://localhost:8000            │          │
│  │                                                   │          │
│  │  [ Start swarming ]                               │          │
│  └───────────────────────────────────────────────────┘          │
│                                                                 │
│  This means:                                                    │
│  - 50 simulated users will be active                            │
│  - They spawn at 5 per second (takes 10 seconds to reach 50)   │
│  - Each user runs tasks in a loop with random wait_time         │
│  - Locust tracks every request's timing and status              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Writing Realistic Scenarios

**A bad load test uses fake patterns. A good one models real behavior.**

```
┌─────────────────────────────────────────────────────────────────┐
│             REALISTIC VS UNREALISTIC LOAD TESTS                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ UNREALISTIC:                                                │
│  ├─ All users hit the same endpoint                             │
│  ├─ No think time (machine-gun requests)                        │
│  ├─ No authentication (skips auth overhead)                     │
│  ├─ No write operations (read-only is easier to scale)          │
│  └─ All same data (unrealistic cache hit rate)                  │
│                                                                 │
│  ✅ REALISTIC:                                                  │
│  ├─ Mix of endpoints weighted by real usage                     │
│  ├─ Think time between actions (between(1, 3))                  │
│  ├─ Authenticated requests (includes token validation cost)     │
│  ├─ Mix of reads and writes (80/20 is typical for SaaS)         │
│  └─ Varied data (different org IDs, project IDs)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A more realistic scenario with multiple user types:**

```python
# locustfile.py — realistic multi-user scenario

import random
from locust import HttpUser, task, between


class RegularMember(HttpUser):
    """80% of traffic: regular team members browsing and updating tasks."""
    weight = 8  # 8 out of 10 users will be this type

    wait_time = between(2, 5)  # Casual browsing pace

    def on_start(self):
        resp = self.client.post("/auth/login", json={
            "email": f"member{random.randint(1, 50)}@example.com",
            "password": "testpass123",
        })
        self.auth_headers = {
            "Authorization": f"Bearer {resp.json()['access_token']}"
        }
        self.org_id = random.choice([1, 2, 3])  # Spread across orgs

    @task(5)
    def browse_tasks(self):
        project_id = random.randint(1, 10)
        self.client.get(
            f"/organizations/{self.org_id}/projects/{project_id}/tasks",
            headers=self.auth_headers,
            name="/orgs/{id}/projects/{id}/tasks",
        )

    @task(3)
    def view_task_detail(self):
        task_id = random.randint(1, 100)
        self.client.get(
            f"/tasks/{task_id}",
            headers=self.auth_headers,
            name="/tasks/{id}",
        )

    @task(1)
    def update_task_status(self):
        task_id = random.randint(1, 100)
        self.client.patch(
            f"/tasks/{task_id}",
            json={"status": random.choice(["todo", "in_progress", "done"])},
            headers=self.auth_headers,
            name="/tasks/{id}",
        )


class AdminUser(HttpUser):
    """20% of traffic: admins checking dashboards and managing users."""
    weight = 2  # 2 out of 10 users will be this type

    wait_time = between(3, 8)  # Admins spend more time reading

    def on_start(self):
        resp = self.client.post("/auth/login", json={
            "email": "admin@example.com",
            "password": "adminpass123",
        })
        self.auth_headers = {
            "Authorization": f"Bearer {resp.json()['access_token']}"
        }

    @task(3)
    def view_dashboard(self):
        self.client.get(
            "/organizations/1/dashboard",
            headers=self.auth_headers,
            name="/orgs/{id}/dashboard",
        )

    @task(1)
    def list_members(self):
        self.client.get(
            "/organizations/1/members",
            headers=self.auth_headers,
            name="/orgs/{id}/members",
        )
```

**How `weight` works:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  If you spawn 100 simulated users:                              │
│                                                                 │
│  RegularMember (weight=8): ~80 users browsing and updating      │
│  AdminUser     (weight=2): ~20 users viewing dashboards         │
│                                                                 │
│  This models a real SaaS app where most traffic is              │
│  regular users, with a smaller portion of admin activity.       │
│                                                                 │
│  Combined with @task weights WITHIN each user class:            │
│  RegularMember does browse(5):detail(3):update(1) ≈ 56%:33%:11%│
│  AdminUser does dashboard(3):members(1) ≈ 75%:25%              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Reading Results and the Optimization Cycle

**Locust gives you a statistics table and real-time charts. Here's how to read them.**

```
┌─────────────────────────────────────────────────────────────────┐
│           LOCUST RESULTS TABLE (example)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Name                           Reqs  Fails  Avg  Med  p95  p99│
│  ──────────────────────────────  ────  ─────  ───  ───  ───  ───│
│  GET  /orgs/{id}/dashboard        892   12    683  520  1800 3200│
│  GET  /orgs/{id}/projects/{id}   1847    3    145  102   380  820│
│  GET  /tasks/{id}                2104    1     52   45    95  180│
│  PATCH /tasks/{id}                523    8    210  180   450  900│
│  POST /auth/login                  50    0    320  310   380  410│
│  ──────────────────────────────  ────  ─────  ───  ───  ───  ───│
│  Total                           5416   24    198  105   680 1400│
│                                                                 │
│  All times in milliseconds.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What do these columns mean?**

```
┌─────────────────────────────────────────────────────────────────┐
│                    UNDERSTANDING PERCENTILES                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine sorting ALL response times from fastest to slowest.    │
│                                                                 │
│  892 requests to /dashboard, sorted:                            │
│  [23ms, 25ms, 30ms, ... ... ... 1800ms, 2200ms, 3200ms]        │
│   ▲                       ▲             ▲               ▲       │
│   fastest               middle          │               slowest │
│                           │             │                       │
│                                                                 │
│  Avg (Average): 683ms                                           │
│  ├─ Sum of all times / count                                    │
│  ├─ MISLEADING: pulled up by a few very slow requests           │
│  └─ Does not represent any typical user experience              │
│                                                                 │
│  Med (Median / p50): 520ms                                      │
│  ├─ The MIDDLE value: 50% of requests were faster than this     │
│  ├─ "The typical user experience"                               │
│  └─ More useful than average for skewed distributions           │
│                                                                 │
│  p95: 1800ms                                                    │
│  ├─ 95% of requests were faster than this                       │
│  ├─ "The bad experience that 1 in 20 users has"                 │
│  └─ This is the SLA target most teams use                       │
│                                                                 │
│  p99: 3200ms                                                    │
│  ├─ 99% of requests were faster than this                       │
│  ├─ "The worst experience except true outliers"                 │
│  └─ At 10,000 requests/day, 100 users hit this. That matters.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why averages lie — visually:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Imagine 100 requests:                                          │
│                                                                 │
│   Fast ◀────────────────────────────────────────────▶ Slow      │
│                                                                 │
│   ████████████████████████████████████████░░░░░▓▓▓██            │
│   ▲                                      ▲    ▲   ▲            │
│   Most requests                          p50  p95 p99          │
│   clustered here                                               │
│   (40-80ms)                              │    │   │            │
│                                         52ms 380 820ms         │
│                                                                 │
│   Average: 145ms ← WHERE IS THIS? It's in the GAP              │
│   between the cluster and the tail. NOBODY is at 145ms.         │
│                                                                 │
│   The average is a number that describes NO ACTUAL USER.        │
│   The median (p50) describes the TYPICAL user.                  │
│   The p95 describes the UNLUCKY user (but not rare).            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to read the Locust results from our example:**

```
┌─────────────────────────────────────────────────────────────────┐
│           DIAGNOSING FROM LOCUST RESULTS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  /orgs/{id}/dashboard:  p50=520ms, p95=1800ms, p99=3200ms      │
│  └─ 🔴 PROBLEM: p95 is 1.8 seconds. 1 in 20 users waits >1.8s │
│     p99 at 3.2s means some users are having a terrible time.    │
│     The gap between p50 (520ms) and p95 (1800ms) suggests       │
│     the endpoint degrades badly under contention.               │
│     INVESTIGATE: connection pool exhaustion? Lock contention?   │
│                                                                 │
│  /orgs/{id}/projects/{id}:  p50=102ms, p95=380ms               │
│  └─ 🟡 ACCEPTABLE but watch: p95 is 3.7× the median.           │
│     Some variance, probably normal for DB queries.              │
│                                                                 │
│  /tasks/{id}:  p50=45ms, p95=95ms                               │
│  └─ 🟢 HEALTHY: low latency, small variance. No action needed. │
│                                                                 │
│  PATCH /tasks/{id}:  8 failures out of 523                      │
│  └─ 🔴 PROBLEM: 1.5% failure rate under load.                  │
│     Writes are failing. Possible causes: lock contention,       │
│     unique constraint violations under concurrency,             │
│     connection pool exhaustion.                                 │
│                                                                 │
│  POST /auth/login:  0 failures, p95=380ms                       │
│  └─ 🟢 FINE: Only runs once per user (on_start). Stable.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Headless mode for CI/automation (no web UI needed):**

```bash
# Run locust without the web UI — useful in CI pipelines
locust \
  -f locustfile.py \
  --host=http://localhost:8000 \
  --headless \
  --users 50 \
  --spawn-rate 5 \
  --run-time 2m \
  --csv=results/loadtest

# This produces CSV files:
#   results/loadtest_stats.csv        — summary statistics
#   results/loadtest_stats_history.csv — time-series data
#   results/loadtest_failures.csv     — failed request details
```

**The complete optimization cycle (putting it all together):**

```
┌─────────────────────────────────────────────────────────────────┐
│          THE COMPLETE OPTIMIZATION CYCLE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: BASELINE MEASUREMENT                                   │
│  ─────────────────────────────                                  │
│  Run Locust with your current code. Record the numbers.         │
│  This is your "before."                                         │
│                                                                 │
│  Example: /dashboard p95=1800ms, 12 failures, 92 queries/req    │
│                                                                 │
│                                                                 │
│  STEP 2: IDENTIFY BOTTLENECK                                    │
│  ─────────────────────────────                                  │
│  Use timing middleware → find slow endpoints                    │
│  Use query profiling  → find N+1 and slow queries               │
│  Use Server-Timing    → find which layer is expensive           │
│                                                                 │
│  Finding: 92 queries per request. N+1 on tasks → assignees.     │
│                                                                 │
│                                                                 │
│  STEP 3: TARGETED FIX                                           │
│  ────────────────────                                           │
│  Apply ONE optimization. Not five at once.                      │
│  If you change five things and it improves, you don't know      │
│  WHICH change helped. One change at a time.                     │
│                                                                 │
│  Fix: Added selectinload(Task.assignee). 92 queries → 3.        │
│                                                                 │
│                                                                 │
│  STEP 4: MEASURE AGAIN                                          │
│  ─────────────────────                                          │
│  Run Locust with IDENTICAL configuration.                       │
│  Compare before/after.                                          │
│                                                                 │
│  Result: /dashboard p95=480ms (was 1800ms), 0 failures          │
│  Improvement: 73% reduction in p95 latency, failures eliminated │
│                                                                 │
│                                                                 │
│  STEP 5: DOCUMENT                                               │
│  ────────────────                                               │
│  Write it down. Your Week 12 project requires this.             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ OPTIMIZATION #1: Fix N+1 in dashboard endpoint      │        │
│  │                                                     │        │
│  │ Problem: 92 DB queries per request due to N+1       │        │
│  │ Fix: selectinload(Project.tasks.assignee)           │        │
│  │ Before: p95=1800ms, 12 failures/892 requests        │        │
│  │ After:  p95=480ms,   0 failures/940 requests        │        │
│  │ Impact: 73% p95 reduction, 100% failure elimination │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│                                                                 │
│  STEP 6: IS IT GOOD ENOUGH?                                     │
│  ──────────────────────────                                     │
│  Compare against your target (e.g., p95 < 200ms).              │
│  If not → go back to Step 2. There's a new bottleneck now.     │
│  If yes → move on. Don't over-optimize.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              PERFORMANCE QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GOLDEN RULE:                                                   │
│      Never optimize without measuring first.                    │
│      Never claim an improvement without before/after numbers.   │
│                                                                 │
│  TIMING MIDDLEWARE:                                              │
│      @app.middleware("http")                                    │
│      → time.perf_counter() before and after call_next()         │
│      → Log duration_ms, query_count with structlog              │
│      → Add Server-Timing header for browser visibility          │
│                                                                 │
│  DATABASE PROFILING:                                            │
│      @event.listens_for(engine, "before_cursor_execute")        │
│      @event.listens_for(engine, "after_cursor_execute")         │
│      → ContextVar[QueryStats] for per-request counting          │
│      → Warn when query_count > 10                               │
│      → Assert query budgets in tests                            │
│                                                                 │
│  RESPONSE OPTIMIZATION:                                         │
│      app.add_middleware(GzipMiddleware, minimum_size=500)        │
│      asyncio.gather(call_a(), call_b()) for parallel I/O        │
│                                                                 │
│  LOAD TESTING:                                                  │
│      locust -f locustfile.py --host=http://localhost:8000       │
│      → HttpUser, @task(weight), wait_time=between(a, b)         │
│      → Read p50, p95, p99 — NOT averages                        │
│      → Use --headless --csv for CI automation                   │
│                                                                 │
│  PERCENTILES:                                                   │
│      p50 (median) = typical user experience                     │
│      p95 = worst experience for 1 in 20 users (SLA target)     │
│      p99 = worst experience for 1 in 100 users (tail latency)  │
│                                                                 │
│  THE CYCLE:                                                     │
│      MEASURE → IDENTIFY → FIX (one thing) → MEASURE AGAIN      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  PERFORMANCE = DIAGNOSIS, NOT GUESSING                          │
│                                                                 │
│  You are a doctor, not a fortune teller.                        │
│                                                                 │
│  ┌───────────┐     ┌───────────┐     ┌───────────┐              │
│  │  Symptom  │────▶│ Diagnostic│────▶│ Targeted  │              │
│  │  "Slow"   │     │  Tools    │     │    Fix    │              │
│  └───────────┘     └───────────┘     └───────────┘              │
│       │                 │                  │                     │
│       │            ┌────┴────┐             │                     │
│       │            │ Timing  │             │                     │
│       │            │ Query   │             │                     │
│       │            │ Locust  │             │                     │
│       │            └────┬────┘             │                     │
│       │                 │                  │                     │
│       │                 ▼                  │                     │
│       │          ┌─────────────┐           │                     │
│       │          │  Measure    │◀──────────┘                     │
│       │          │  AGAIN      │                                │
│       │          └─────────────┘                                │
│       │                 │                                       │
│       │                 ▼                                       │
│       │          Did it work?                                   │
│       │          YES → document, move on                        │
│       └───────── NO  → new bottleneck, go around again          │
│                                                                 │
│                                                                 │
│  THREE TOOLS YOU BUILT TODAY:                                   │
│  ├─ Timing middleware     → "taking the patient's temperature"  │
│  ├─ Query profiling       → "running blood work"                │
│  └─ Locust load testing   → "stress test on the treadmill"     │
│                                                                 │
│  ONE RULE TO REMEMBER:                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │         MEASURE, DON'T GUESS.                           │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Lectures and Projects

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 12 LECTURE 4 (next):                                      │
│  └─ Rate limiting with slowapi                                  │
│     Uses timing middleware to understand request patterns.      │
│     Uses p95/p99 to set rate limit thresholds.                  │
│     Adds performance regression tests to your CI pipeline.      │
│                                                                 │
│  WEEK 12 PROJECT:                                               │
│  └─ "At least 3 documented optimizations with                   │
│      before/after metrics. Load test report."                   │
│     You now have every tool to produce this deliverable.        │
│     Use the optimization cycle from Part 5 for each one.        │
│                                                                 │
│  WEEK 13-14 CAPSTONE:                                           │
│  └─ "Load tested with documented results"                       │
│     Apply these tools from day one of the capstone.             │
│     Build with the timing middleware already in place.           │
│     Write query count tests as you write endpoints.             │
│                                                                 │
│  WEEK 15 (CI/CD):                                               │
│  └─ Locust headless mode in your CI pipeline                    │
│     Automatically catch performance regressions                 │
│     before they reach production.                               │
│                                                                 │
│  WEEK 16 (System Design):                                       │
│  └─ Understanding bottlenecks informs scaling decisions.        │
│     "Should we add a cache or a read replica?"                  │
│     The answer depends on WHERE the bottleneck is —             │
│     and now you know how to find out.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```