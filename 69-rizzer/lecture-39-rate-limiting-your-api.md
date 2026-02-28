# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DESTRUCTION FIRST, PROTECTION SECOND                           │
│  ────────────────────────────────────                           │
│  Students must BREAK their own API before learning to defend    │
│  it. We hammer an unprotected endpoint and watch it suffer.     │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  A nightclub bouncer analogy carries through the lecture.       │
│  Every algorithm maps to a bouncer counting strategy.           │
│                                                                 │
│  BOTH SIDES OF THE COIN                                         │
│  ────────────────────────                                       │
│  Week 8 taught RESPECTING rate limits as a client.              │
│  Now we ENFORCE them as a server. Full circle.                  │
│                                                                 │
│  NUMBERS DON'T LIE — BUT AVERAGES DO                            │
│  ─────────────────────────────────────                          │
│  Percentile latencies are taught through visceral examples.     │
│  Students learn why "average response time" is a trap.          │
│                                                                 │
│  CONNECT TO PRIOR KNOWLEDGE                                     │
│  ─────────────────────────                                      │
│  Decorators (Week 1) → @limiter.limit is a decorator           │
│  Redis (Week 10)     → Distributed rate limit storage           │
│  Middleware (Week 9, 12-L3) → Where the limiter lives           │
│  locust (Week 12-L3) → Testing rate limits under load           │
│  Auth (Week 9)       → Per-user limits, login protection        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

#=# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                  RATE LIMITING YOUR API                         │
│                   (3–4 Hour Lecture)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (35 min)                                   │
│  ├─ 1.1 The Unprotected API (Demonstration)                     │
│  ├─ 1.2 Three Reasons to Rate Limit                             │
│  ├─ 1.3 Who Gets Limited? (Identification)                      │
│  └─ 1.4 The Nightclub Bouncer Analogy                           │
│                                                                 │
│  PART 2: THE ALGORITHMS (35 min)                                │
│  ├─ 2.1 Fixed Window Counter                                    │
│  ├─ 2.2 Sliding Window Log                                      │
│  ├─ 2.3 Token Bucket                                            │
│  └─ 2.4 Choosing the Right Algorithm                            │
│                                                                 │
│  PART 3: THE IMPLEMENTATION (60 min)                            │
│  ├─ 3.1 slowapi Setup                                           │
│  ├─ 3.2 @limiter.limit() — Setting Limits                       │
│  ├─ 3.3 Key Functions — Identifying Clients                     │
│  ├─ 3.4 Per-Endpoint and Global Limits                          │
│  ├─ 3.5 Redis-Backed Rate Limiting (Connection to Week 10)      │
│  └─ 3.6 Custom 429 Handler                                      │
│                                                                 │
│  PART 4: THE PROTOCOL (25 min)                                  │
│  ├─ 4.1 Rate Limit Headers (X-RateLimit-*)                      │
│  ├─ 4.2 429 Too Many Requests + Retry-After                     │
│  └─ 4.3 Full Circle: Server ↔ Client (Connection to Week 8)     │
│                                                                 │
│  PART 5: MEASURING IMPACT (45 min)                              │
│  ├─ 5.1 Why Averages Lie                                        │
│  ├─ 5.2 Percentile Latencies (p50, p95, p99)                    │
│  ├─ 5.3 Load Testing with Rate Limits (Connection to L3)        │
│  ├─ 5.4 Performance Regression Testing in CI                    │
│  └─ 5.5 Setting Thresholds and Alerts                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Unprotected API

**Start with destruction. Let them watch their API get hammered.**

Your Task Manager API now has authentication, caching, background jobs, and WebSocket notifications. It's feature-complete. It handles real users gracefully.

But it's *defenseless*.

```python
# the_hammer.py — What a malicious client (or a buggy script) looks like
import asyncio
import httpx
import time

async def hammer_api():
    """Send 200 requests as fast as possible."""
    async with httpx.AsyncClient(base_url="http://localhost:8000") as client:
        start = time.time()

        # Fire 200 requests concurrently — you learned this in Week 8
        tasks = [
            client.get(
                "/api/v1/tasks",
                headers={"Authorization": "Bearer <valid_token>"}
            )
            for _ in range(200)
        ]
        responses = await asyncio.gather(*tasks, return_exceptions=True)

        elapsed = time.time() - start

        # Count what happened
        statuses: dict[int, int] = {}
        errors = 0
        for r in responses:
            if isinstance(r, Exception):
                errors += 1
            else:
                statuses[r.status_code] = statuses.get(r.status_code, 0) + 1

        print(f"200 requests in {elapsed:.1f}s")
        print(f"Status codes: {statuses}")
        print(f"Connection errors: {errors}")

asyncio.run(hammer_api())
```

**Run it. Watch the output.**

```
200 requests in 2.8s
Status codes: {200: 200}
Connection errors: 0
```

**Now ask the class:**

> "All 200 succeeded. Sounds great, right? So what's the problem?"

The problems are invisible — they're happening *behind* that 200 OK:

```
┌─────────────────────────────────────────────────────────────────┐
│              THE INVISIBLE DAMAGE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHILE THOSE 200 REQUESTS WERE BEING SERVED:                    │
│                                                                 │
│  💾 Database connection pool: EXHAUSTED                          │
│     └─ All 20 pool connections occupied by one client            │
│     └─ Other users' requests QUEUING behind this flood           │
│                                                                 │
│  🌐 External API quotas: BURNED                                  │
│     └─ Your OpenWeatherMap free tier: 60 calls/min               │
│     └─ This one client just consumed 200 calls                   │
│     └─ Every other user gets "503 Service Unavailable"           │
│                                                                 │
│  📊 Celery workers: BURIED                                       │
│     └─ 200 notification tasks queued simultaneously              │
│     └─ Legitimate background jobs delayed by minutes             │
│                                                                 │
│  🔋 Server resources: STRAINED                                   │
│     └─ CPU spike → OTHER users see slow responses                │
│     └─ Memory spike → OOM risk under sustained attack            │
│                                                                 │
│  💸 Your cloud bill: GROWING                                     │
│     └─ Every request costs compute, bandwidth, DB time           │
│     └─ Attacker pays nothing. YOU pay for their abuse.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The punchline:**

> "You authenticated this user. You validated their input with Pydantic. You cached their queries with Redis. You did everything right — except *control how often they can knock on your door.*"

---

## 1.2 Three Reasons to Rate Limit

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY RATE LIMIT?                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. FAIRNESS  🤝                                                 │
│     ─────────────                                               │
│     One client should not consume all your server's capacity.   │
│     100 users sharing an API deserve equal access.              │
│     Without limits, the fastest client wins and everyone        │
│     else starves.                                               │
│                                                                 │
│  2. PROTECTION  🛡️                                               │
│     ───────────────                                             │
│     Defense against:                                            │
│     • DDoS attacks (flood of requests to crash your server)     │
│     • Brute force (Week 9: we rate-limited login for this)      │
│     • Scrapers (bots harvesting your data)                      │
│     • Buggy clients (infinite loop sending requests)            │
│                                                                 │
│  3. COST CONTROL  💰                                             │
│     ─────────────────                                           │
│     Every request costs:                                        │
│     • CPU cycles (your cloud bill)                              │
│     • Database connections (pooled, finite)                     │
│     • External API calls (someone else's rate limits!)          │
│     • Bandwidth (data transfer charges)                         │
│     Rate limiting caps your worst-case cost.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 Who Gets Limited?

**Before you can limit someone, you need to identify them.**

> "Think of it this way: if 200 requests arrive, how do you know if it's one person sending 200 or 200 people sending one each?"

```
┌─────────────────────────────────────────────────────────────────┐
│              IDENTIFICATION STRATEGIES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BY IP ADDRESS                                                  │
│  ──────────────                                                 │
│  Simplest. Works for anonymous endpoints.                       │
│  Problem: Users behind a shared NAT/VPN share one IP.           │
│  All of a company's employees look like one client.             │
│                                                                 │
│  BY AUTHENTICATED USER                                          │
│  ──────────────────────                                         │
│  Use the JWT user ID (Week 9's get_current_user).               │
│  Most fair — limits follow the person, not the network.         │
│  Problem: Doesn't protect unauthenticated endpoints.            │
│                                                                 │
│  BY API KEY                                                     │
│  ──────────                                                     │
│  Common for third-party API providers.                          │
│  Each developer/app gets a key with its own quota.              │
│  Problem: Keys can be shared or stolen.                         │
│                                                                 │
│  COMBINED (Production reality)                                  │
│  ────────────────────────────                                   │
│  IP for anonymous endpoints (login, register)                   │
│  User ID for authenticated endpoints                            │
│  API key for third-party integrations                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

This concept of identifying the client is called a **key function** — you'll implement it in Part 3.

---

## 1.4 The Nightclub Bouncer Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE NIGHTCLUB BOUNCER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT A BOUNCER (No rate limiting)                           │
│  ────────────────────────────────────                           │
│                                                                 │
│  Everyone rushes the door at once.                              │
│  Club is overcrowded. Music lags. Drinks run out.               │
│  Fire code violated. EVERYONE has a terrible time.              │
│  One group of 50 friends ruins the night for 200 others.        │
│                                                                 │
│                                                                 │
│  WITH A BOUNCER (Rate limiting)                                 │
│  ──────────────────────────────                                 │
│                                                                 │
│  Bouncer stands at the door.                                    │
│  Counts people entering. "10 per minute, that's the rule."      │
│  If you're over the limit: "Sorry, come back in 5 minutes."     │
│  Club stays comfortable. Music is crisp. Everyone has fun.      │
│                                                                 │
│                                                                 │
│  THE MAPPING:                                                   │
│  ────────────                                                   │
│                                                                 │
│  Nightclub                │  Your API                           │
│  ─────────────────────────│──────────────────────────           │
│  Bouncer                  │  Rate limiter middleware             │
│  Club capacity            │  Server resources                   │
│  Patrons arriving         │  Incoming requests                  │
│  Counting heads           │  Tracking request count             │
│  "Come back later"        │  429 Too Many Requests              │
│  VIP wristband (skip)     │  Admin role → higher limits         │
│  Checking ID at door      │  Key function (IP / user ID)        │
│  Different rooms have     │  Different endpoints have           │
│    different capacities   │    different rate limits             │
│  Hourly head-count reset  │  Time window reset                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 2: THE ALGORITHMS

**The bouncer needs a *strategy* for counting. There are three main ones.**

## 2.1 Fixed Window Counter

**The simplest approach: count requests in fixed time chunks.**

```
┌─────────────────────────────────────────────────────────────────┐
│              FIXED WINDOW COUNTER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Rule: "10 requests per minute"                                 │
│                                                                 │
│  The bouncer resets his count at the top of every minute:       │
│                                                                 │
│  12:00:00 ──────────────── 12:01:00 ──────────────── 12:02:00   │
│  │      Window 1          │      Window 2          │            │
│  │                        │                        │            │
│  │  Counter: 0→1→2...→10  │  Counter: 0→1→2...     │            │
│  │  (resets at boundary)  │  (resets at boundary)  │            │
│                                                                 │
│  HOW IT WORKS:                                                  │
│  ─────────────                                                  │
│  1. Request arrives                                             │
│  2. Look up counter for current window                          │
│  3. If counter < limit → allow, increment counter               │
│  4. If counter >= limit → reject with 429                       │
│  5. Counter resets when new window begins                       │
│                                                                 │
│  PROS:                        CONS:                             │
│  • Dead simple                • Boundary burst problem          │
│  • Low memory (one counter)   • Uneven distribution             │
│  • Fast (single increment)    • (see below)                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The boundary burst problem — the fatal flaw of fixed windows:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE BOUNDARY BURST PROBLEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Rule: 10 requests per minute                                   │
│                                                                 │
│      Window 1                      Window 2                     │
│  ├─────────────────────────────┤─────────────────────────────┤  │
│  12:00             12:00:59    12:01:00             12:01:59    │
│                                                                 │
│  A clever (or unlucky) client sends:                            │
│                                                                 │
│  • 0 requests from 12:00:00 – 12:00:29                          │
│  • 10 requests from 12:00:30 – 12:00:59  ← Within limit ✅      │
│  • 10 requests from 12:01:00 – 12:01:30  ← Within limit ✅      │
│  • 0 requests from 12:01:30 – 12:01:59                          │
│                                                                 │
│  Result: 20 requests in 60 seconds!                             │
│          DOUBLE the intended rate!  😱                           │
│                                                                 │
│  Both bursts happen right at the window boundary.               │
│  Each window individually sees only 10 requests.                │
│  But the actual throughput across the boundary is 2x.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The bouncer resets his counter at midnight on the dot. A group of 10 rushes in at 11:59 PM. Counter resets. The same group's friends — another 10 — walk in at 12:01 AM. Both batches are 'within limits.' But 20 people entered in 2 minutes."

---

## 2.2 Sliding Window Log

**Instead of fixed boundaries, track the actual timestamps.**

```
┌─────────────────────────────────────────────────────────────────┐
│              SLIDING WINDOW LOG                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Rule: "10 requests per minute"                                 │
│                                                                 │
│  The bouncer writes down the EXACT TIME each person enters:     │
│                                                                 │
│  Log: [12:00:15, 12:00:22, 12:00:31, 12:00:45, ...]            │
│                                                                 │
│  When a new request arrives at 12:01:10:                        │
│  1. Look back exactly 60 seconds: window is 12:00:10–12:01:10  │
│  2. Count entries in that window                                │
│  3. Remove entries older than 12:00:10 (they've expired)        │
│  4. If count < 10 → allow, add timestamp to log                 │
│  5. If count >= 10 → reject with 429                            │
│                                                                 │
│                                                                 │
│  VISUALIZATION:                                                 │
│  ──────────────                                                 │
│  Time ───────────────────────────────────────────────▶          │
│        12:00:10                           12:01:10              │
│            │◄──── Sliding 60-second window ────►│               │
│            │                                    │               │
│            │  [x] [x] [x] [x] [x] [x] [x]     │               │
│            │  7 requests in window → ALLOW      │               │
│            │                                    │               │
│  Expired:  │                                                    │
│  [x] [x]──┘ (older than 60s, removed)                          │
│                                                                 │
│                                                                 │
│  PROS:                          CONS:                           │
│  • Precise (no boundary burst)  • Higher memory (stores log)    │
│  • Smooth enforcement           • More expensive per check      │
│  • Accurate over any interval   • Log can grow large            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This bouncer doesn't reset at midnight. He looks at his clipboard and counts everyone who entered in the *last 60 minutes from right now*. No gaming the boundary."

---

## 2.3 Token Bucket

**The most commonly used algorithm in production. Allows controlled bursts.**

```
┌─────────────────────────────────────────────────────────────────┐
│              TOKEN BUCKET                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Rule: Bucket holds max 10 tokens. Refills 1 token per second.  │
│                                                                 │
│  The bouncer has a bowl of wristbands (tokens):                 │
│  • Every person who enters takes one wristband                  │
│  • New wristbands are added to the bowl at a steady rate        │
│  • If the bowl is empty: "Sorry, no wristbands. Wait."          │
│  • Bowl never overflows past maximum capacity                   │
│                                                                 │
│                                                                 │
│  VISUALIZATION:                                                 │
│  ──────────────                                                 │
│                                                                 │
│  Time 0s:  [🟢🟢🟢🟢🟢🟢🟢🟢🟢🟢]  10 tokens (full bucket)     │
│                                                                 │
│  Burst! 6 requests arrive at once:                              │
│  Time 1s:  [🟢🟢🟢🟢⬚⬚⬚⬚⬚⬚]   4 tokens left                  │
│                 All 6 served ✅  (burst allowed!)                │
│                                                                 │
│  Time 2s:  [🟢🟢🟢🟢🟢⬚⬚⬚⬚⬚]   5 tokens (+1 refilled)        │
│  Time 3s:  [🟢🟢🟢🟢🟢🟢⬚⬚⬚⬚]   6 tokens (+1 refilled)        │
│  ...                                                            │
│  Time 7s:  [🟢🟢🟢🟢🟢🟢🟢🟢🟢🟢]  10 tokens (full again)      │
│                                                                 │
│  Another burst! 12 requests arrive:                             │
│  Time 8s:  [⬚⬚⬚⬚⬚⬚⬚⬚⬚⬚]   0 tokens                         │
│                 10 served ✅, 2 rejected 429 ❌                   │
│                                                                 │
│                                                                 │
│  KEY INSIGHT:                                                   │
│  Token bucket allows BURSTS up to the bucket size,              │
│  but SUSTAINED rate is limited to the refill rate.              │
│                                                                 │
│                                                                 │
│  PROS:                          CONS:                           │
│  • Allows controlled bursts     • Slightly more complex logic   │
│  • Smooth long-term rate        • Two parameters to tune        │
│  • Industry standard            │  (bucket size + refill rate)  │
│  • Memory efficient             │                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This bouncer is the smartest one. He doesn't just count — he rations. You get a burst when you first arrive, but if you keep coming back too fast, you'll find the bowl empty."

---

## 2.4 Choosing the Right Algorithm

```
┌─────────────────────────────────────────────────────────────────┐
│              ALGORITHM DECISION FRAMEWORK                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    Which algorithm?                              │
│                         │                                       │
│                         ▼                                       │
│              ┌─────────────────────┐                            │
│              │ Do you need to      │                            │
│              │ allow short bursts? │                            │
│              └──────────┬──────────┘                            │
│                   │          │                                  │
│                  YES         NO                                 │
│                   │          │                                  │
│                   ▼          ▼                                  │
│         ┌──────────────┐  ┌────────────────────┐               │
│         │ TOKEN BUCKET │  │ Does precision      │               │
│         │ ✅ Best for   │  │ matter more than    │               │
│         │ general APIs │  │ simplicity?          │               │
│         └──────────────┘  └──────────┬─────────┘               │
│                                │          │                     │
│                               YES         NO                    │
│                                │          │                     │
│                                ▼          ▼                     │
│                     ┌──────────────┐ ┌──────────────┐           │
│                     │ SLIDING      │ │ FIXED        │           │
│                     │ WINDOW       │ │ WINDOW       │           │
│                     │ ✅ Precise    │ │ ✅ Simplest   │           │
│                     │ No boundary  │ │ Fast check   │           │
│                     │ exploits     │ │ Low memory   │           │
│                     └──────────────┘ └──────────────┘           │
│                                                                 │
│  IN PRACTICE:                                                   │
│  Token bucket is the most common default.                       │
│  slowapi uses fixed window internally (from the limits lib)     │
│  but the principle is the same for your purposes.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "For a login endpoint that you want to protect from brute force, which algorithm would you prefer? What about a search endpoint that legitimate users might hit rapidly?"

---

# PART 3: THE IMPLEMENTATION

## 3.1 slowapi Setup

**slowapi is the standard rate limiting library for FastAPI.**

It's a rate limiting library for Starlette and FastAPI adapted from flask-limiter.[[3]](https://github.com/laurentS/slowapi) The actual rate limiting work is done by the `limits` library — slowapi is just a wrapper around it.[[2]](https://slowapi.readthedocs.io/)

```python
# Install it
# pip install slowapi
```

**Basic setup — 4 lines to protect your entire app:**

```python
# app/core/rate_limit.py
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

# Create the limiter — key_func decides HOW to identify clients
limiter = Limiter(key_func=get_remote_address)
```

```python
# app/main.py
from fastapi import FastAPI
from app.core.rate_limit import limiter
from slowapi.errors import RateLimitExceeded
from slowapi import _rate_limit_exceeded_handler

app = FastAPI()

# Two critical lines:
app.state.limiter = limiter                                         # 1. Store limiter in app state
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)  # 2. Handle 429s
```

**What each piece does:**

```
┌─────────────────────────────────────────────────────────────────┐
│              SETUP BREAKDOWN                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Limiter(key_func=get_remote_address)                           │
│  ─────────────────────────────────────                          │
│  Creates the bouncer. key_func tells the bouncer                │
│  HOW to identify each patron (by IP address here).              │
│                                                                 │
│  app.state.limiter = limiter                                    │
│  ────────────────────────────                                   │
│  Posts the bouncer at the door (attaches to the app).           │
│  slowapi looks for this exact attribute name.                   │
│                                                                 │
│  app.add_exception_handler(RateLimitExceeded, ...)              │
│  ──────────────────────────────────────────────────             │
│  Tells FastAPI: "When the bouncer rejects someone,              │
│  respond with a 429 status code."                               │
│  (Same custom exception handler pattern from Week 3)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 @limiter.limit() — Setting Limits

**Apply limits to individual endpoints with a decorator:**

```python
from fastapi import FastAPI, Request
from app.core.rate_limit import limiter

app = FastAPI()

# The route decorator MUST be above the limit decorator
@app.get("/api/v1/tasks")
@limiter.limit("30/minute")
async def get_tasks(request: Request):      # ← request MUST be a parameter!
    return {"tasks": []}

@app.post("/api/v1/tasks")
@limiter.limit("10/minute")
async def create_task(request: Request):    # ← request MUST be a parameter!
    return {"task": "created"}
```

**Two critical rules to remember:**

The request argument must be explicitly passed to your endpoint, or slowapi won't be able to hook into it. In other words, write `@limiter.limit("5/minute") async def myendpoint(request: Request)` and not `@limiter.limit("5/minute") async def myendpoint()`.[[3]](https://github.com/laurentS/slowapi)

```python
# ❌ WRONG: Missing request parameter — slowapi silently fails
@app.get("/items")
@limiter.limit("5/minute")
async def get_items():              # No request param!
    return {"items": []}

# ❌ WRONG: Decorators in wrong order
@limiter.limit("5/minute")         # Limit decorator on top
@app.get("/items")                  # Route decorator below
async def get_items(request: Request):
    return {"items": []}

# ✅ CORRECT: Route decorator first, request parameter present
@app.get("/items")
@limiter.limit("5/minute")
async def get_items(request: Request):
    return {"items": []}
```

**The rate limit string format:**

```
┌─────────────────────────────────────────────────────────────────┐
│              RATE LIMIT STRING FORMAT                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FORMAT: "{count}/{period}"                                     │
│                                                                 │
│  "5/minute"       →  5 requests per minute                      │
│  "100/hour"       →  100 requests per hour                      │
│  "1/second"       →  1 request per second                       │
│  "1000/day"       →  1000 requests per day                      │
│                                                                 │
│  MULTIPLE LIMITS (semicolon-separated):                         │
│  "5/minute;100/hour"                                            │
│  → Max 5 per minute AND max 100 per hour                        │
│  → Both must be satisfied. Whichever hits first, blocks.        │
│                                                                 │
│  ALTERNATIVE SYNTAX:                                            │
│  "5 per minute"   →  Same as "5/minute"                         │
│  "100 per hour"   →  Same as "100/hour"                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Key Functions — Identifying Clients

**Remember Section 1.3? The key function is *how* the bouncer identifies each patron.**

`get_remote_address` uses the client's IP. But for your authenticated API, you want per-user limits:

```python
# app/core/rate_limit.py
from fastapi import Request
from slowapi import Limiter
from slowapi.util import get_remote_address


def get_user_or_ip(request: Request) -> str:
    """
    Use the authenticated user's ID if available,
    fall back to IP address for anonymous requests.

    Connection to Week 9: request.state.user comes from
    your get_current_user dependency.
    """
    # If auth middleware has set a user on the request state
    if hasattr(request.state, "user") and request.state.user is not None:
        return str(request.state.user.id)

    # Fallback to IP for unauthenticated endpoints
    return get_remote_address(request)


# Use our custom key function
limiter = Limiter(key_func=get_user_or_ip)
```

**Per-endpoint key function override:**

```python
# Some endpoints need IP-based limiting regardless of auth
# (e.g., login — the user isn't authenticated YET)

@app.post("/api/v1/auth/login")
@limiter.limit("5/minute", key_func=get_remote_address)  # Override: always use IP
async def login(request: Request):
    # ... login logic from Week 9 ...
    pass


# Admin endpoints might get higher limits
def get_admin_key(request: Request) -> str:
    """Admins get their own separate limit bucket."""
    return f"admin:{get_remote_address(request)}"


@app.get("/api/v1/admin/users")
@limiter.limit("60/minute", key_func=get_admin_key)
async def admin_list_users(request: Request):
    # ... admin logic ...
    pass
```

**Visualize how key functions create separate buckets:**

```
┌─────────────────────────────────────────────────────────────────┐
│              KEY FUNCTIONS → SEPARATE BUCKETS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  get_remote_address:                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ 192.168.1.1  │  │ 10.0.0.42    │  │ 172.16.0.5   │          │
│  │ Count: 4/10  │  │ Count: 8/10  │  │ Count: 1/10  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  Each IP gets its own counter.                                  │
│                                                                 │
│  get_user_or_ip:                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ user:42      │  │ user:107     │  │ 10.0.0.99    │          │
│  │ Count: 4/10  │  │ Count: 8/10  │  │ Count: 1/10  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  Authed users tracked by ID. Anon tracked by IP.                │
│  User 42 can switch WiFi → still same bucket.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Per-Endpoint and Global Limits

**Different endpoints deserve different limits:**

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

# Global default: applies to ALL endpoints that don't have their own limit
limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["100/minute"]        # Global safety net
)
```

```python
# READS are cheap → higher limit
@app.get("/api/v1/tasks")
@limiter.limit("60/minute")
async def list_tasks(request: Request):
    return {"tasks": []}

# WRITES are expensive → lower limit
@app.post("/api/v1/tasks")
@limiter.limit("20/minute")
async def create_task(request: Request):
    return {"task": "created"}

# SEARCH hits the database hard → strict limit
@app.get("/api/v1/tasks/search")
@limiter.limit("10/minute")
async def search_tasks(request: Request):
    return {"results": []}

# AUTH endpoints: brute force protection → very strict
@app.post("/api/v1/auth/login")
@limiter.limit("5/minute;20/hour", key_func=get_remote_address)
async def login(request: Request):
    return {"token": "..."}

# HEALTH CHECK: no limit (monitoring tools poll this)
@app.get("/health")
@limiter.exempt                          # ← Skip rate limiting entirely
async def health_check():
    return {"status": "ok"}
```

**The tier pattern:**

```
┌─────────────────────────────────────────────────────────────────┐
│              ENDPOINT LIMIT TIERS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TIER        │ ENDPOINTS              │ LIMIT                   │
│  ────────────│────────────────────────│──────────────────       │
│  Unrestricted│ /health, /ready        │ @limiter.exempt          │
│  Generous    │ GET /tasks, GET /users │ 60/minute               │
│  Moderate    │ POST, PUT, PATCH       │ 20/minute               │
│  Strict      │ Search, export, report │ 10/minute               │
│  Fortress    │ Login, register, reset │ 5/minute; 20/hour       │
│                                                                 │
│  PRINCIPLE: The more expensive the operation                    │
│  (CPU, DB, external calls), the tighter the limit.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Redis-Backed Rate Limiting (Connection to Week 10)

**Remember the problem with in-memory rate limiting?**

> "Your Task Manager runs behind a load balancer with 3 server instances. In-memory counters are per-process. A client sends 10 requests — 3 hit instance A, 4 hit instance B, 3 hit instance C. Each instance sees under 10. *No one triggers the limit.* Your bouncer has amnesia across doors."

In a single-instance setup, SlowAPI keeps counters in memory. But once you scale horizontally (multiple containers or pods), each instance tracks limits independently — causing inconsistent throttling.[[7]](https://blog.schogini.com/static/html_files/multi-tenant-saas-with-redis-rate-limit.html)

Redis solves this by acting as a central store for counters and tokens.[[7]](https://blog.schogini.com/static/html_files/multi-tenant-saas-with-redis-rate-limit.html)

```
┌─────────────────────────────────────────────────────────────────┐
│              IN-MEMORY vs REDIS STORAGE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IN-MEMORY (default — dev only):                                │
│                                                                 │
│   Instance A          Instance B          Instance C            │
│  ┌───────────┐       ┌───────────┐       ┌───────────┐         │
│  │ Counter: 3│       │ Counter: 4│       │ Counter: 3│         │
│  │ (< 10 ✅) │       │ (< 10 ✅) │       │ (< 10 ✅) │         │
│  └───────────┘       └───────────┘       └───────────┘         │
│  Each thinks it's fine. Total: 10 requests. NONE blocked. ❌    │
│                                                                 │
│                                                                 │
│  REDIS (production):                                            │
│                                                                 │
│   Instance A          Instance B          Instance C            │
│  ┌───────────┐       ┌───────────┐       ┌───────────┐         │
│  │ Check ────│──┐    │ Check ────│──┐    │ Check ────│──┐      │
│  └───────────┘  │    └───────────┘  │    └───────────┘  │      │
│                 │                   │                   │      │
│                 ▼                   ▼                   ▼      │
│              ┌──────────────────────────────────────────┐       │
│              │         REDIS (shared counter)           │       │
│              │         Counter: 10 → BLOCKED ❌          │       │
│              └──────────────────────────────────────────┘       │
│  All instances share one counter. 10th request blocked. ✅      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Configuration — one line changes everything:**

```python
# app/core/rate_limit.py

# Development: in-memory (fine for single instance)
limiter = Limiter(key_func=get_user_or_ip)

# Production: Redis backend (Week 10 — you already have Redis running!)
limiter = Limiter(
    key_func=get_user_or_ip,
    storage_uri="redis://localhost:6379/1"   # Use DB 1 (DB 0 is your cache)
)
```

In production, pull from environment variables using `pydantic-settings` (same pattern you've used since Week 9):

```python
# app/core/config.py (your existing settings — just add one field)
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # ... your existing settings ...
    RATE_LIMIT_STORAGE_URI: str = "memory://"    # default: in-memory for dev

    model_config = {"env_file": ".env"}

settings = Settings()
```

```python
# app/core/rate_limit.py
from app.core.config import settings

limiter = Limiter(
    key_func=get_user_or_ip,
    storage_uri=settings.RATE_LIMIT_STORAGE_URI
)
```

```bash
# .env (production)
RATE_LIMIT_STORAGE_URI=redis://redis:6379/1

# .env (development)
RATE_LIMIT_STORAGE_URI=memory://
```

---

## 3.6 Custom 429 Handler

**The default `_rate_limit_exceeded_handler` returns a plain text response. For an API, you want structured JSON:**

```python
# app/core/rate_limit.py
from fastapi import Request
from fastapi.responses import JSONResponse
from slowapi.errors import RateLimitExceeded


async def custom_rate_limit_handler(
    request: Request,
    exc: RateLimitExceeded
) -> JSONResponse:
    """
    Return a structured 429 response that matches
    your API's error format (consistent with Week 3's
    custom exception handlers).
    """
    # exc.detail contains the limit that was exceeded, e.g. "5 per 1 minute"
    return JSONResponse(
        status_code=429,
        content={
            "error": "rate_limit_exceeded",
            "message": f"Rate limit exceeded: {exc.detail}",
            "retry_after": _get_retry_after(exc),
        },
        headers={
            "Retry-After": str(_get_retry_after(exc)),
        }
    )


def _get_retry_after(exc: RateLimitExceeded) -> int:
    """Extract or estimate seconds until the limit resets."""
    # Parse from the detail string — e.g., "5 per 1 minute" → 60 seconds
    detail = exc.detail.lower()
    if "second" in detail:
        return 1
    elif "minute" in detail:
        return 60
    elif "hour" in detail:
        return 3600
    return 60  # sensible default
```

```python
# app/main.py — use custom handler instead of default
from app.core.rate_limit import limiter, custom_rate_limit_handler
from slowapi.errors import RateLimitExceeded

app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, custom_rate_limit_handler)
```

**Now a rate-limited client sees:**

```json
HTTP/1.1 429 Too Many Requests
Retry-After: 60
Content-Type: application/json

{
    "error": "rate_limit_exceeded",
    "message": "Rate limit exceeded: 5 per 1 minute",
    "retry_after": 60
}
```

> "This is what a well-behaved API looks like. It doesn't just slam the door — it tells the client *when to come back*. That's the Retry-After header. We'll formalize this in Part 4."

---

# PART 4: THE PROTOCOL

## 4.1 Rate Limit Headers (X-RateLimit-*)

**Rate limiting isn't just about blocking — it's about *communicating*.**

A good API tells clients their quota status on *every response*, not just when they're blocked. The commonly used header field names are: `X-RateLimit-Limit` — the maximum number of requests you're permitted to make per hour, `X-RateLimit-Remaining` — the number of requests remaining in the current rate limit window, and `X-RateLimit-Reset` — the time at which the current rate limit window resets in UTC epoch seconds.[[6]](https://www.gvj-web.com/blog/rate-limiting-restful-api)

```
┌─────────────────────────────────────────────────────────────────┐
│              RATE LIMIT HEADERS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EVERY response (200, 201, 404, etc.) includes:                 │
│                                                                 │
│  X-RateLimit-Limit: 60        ← "Your quota is 60/minute"      │
│  X-RateLimit-Remaining: 42    ← "You have 42 requests left"    │
│  X-RateLimit-Reset: 1739548800← "Quota resets at this time"     │
│                                                                 │
│                                                                 │
│  WHEN BLOCKED (429 response) also includes:                     │
│                                                                 │
│  Retry-After: 35              ← "Try again in 35 seconds"      │
│                                                                 │
│                                                                 │
│  THINK OF IT LIKE:                                              │
│  ─────────────────                                              │
│  The bouncer stamps your hand each time you enter:              │
│                                                                 │
│  X-RateLimit-Limit:     "Club capacity is 60 people/hour"       │
│  X-RateLimit-Remaining: "42 spots left this hour"               │
│  X-RateLimit-Reset:     "Counter resets at 2:00 AM"             │
│  Retry-After:           "Come back in 35 minutes"               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Implementing custom headers with middleware:**

```python
# app/middleware/rate_limit_headers.py
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import Response
import time


class RateLimitHeaderMiddleware(BaseHTTPMiddleware):
    """
    Inject X-RateLimit-* headers into every response.
    
    slowapi's default handler adds some headers on 429,
    but we want them on EVERY response so clients can
    proactively throttle themselves.
    """

    async def dispatch(self, request: Request, call_next) -> Response:
        response = await call_next(request)

        # These headers might already be set by slowapi on 429s
        # We ensure they're present on ALL responses
        # In production, you'd read actual values from your limiter/Redis
        # This is the pattern — actual values depend on your storage layer
        if "X-RateLimit-Limit" not in response.headers:
            # Provide informational defaults from your config
            response.headers["X-RateLimit-Limit"] = "60"
            response.headers["X-RateLimit-Remaining"] = "unknown"
            response.headers["X-RateLimit-Reset"] = str(
                int(time.time()) + 60
            )

        return response
```

> "Note: there's currently an IETF draft working on standardizing these as `RateLimit-Policy` and `RateLimit` headers. The `X-` prefix headers are the de facto standard that the industry uses today."

The IETF draft defines a set of standard HTTP header fields: `RateLimit-Policy` — a quota policy defined by the server, and `RateLimit` — the currently remaining quota available for a specific policy.[[5]](https://greenbytes.de/tech/webdav/draft-ietf-httpapi-ratelimit-headers-latest.html) Commonly used header field names today are: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.[[1]](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)

---

## 4.2 429 Too Many Requests + Retry-After

**429 is the HTTP status code specifically designed for rate limiting.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE 429 RESPONSE                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP/1.1 429 Too Many Requests                                 │
│  Content-Type: application/json                                 │
│  Retry-After: 45                                                │
│  X-RateLimit-Limit: 10                                          │
│  X-RateLimit-Remaining: 0                                       │
│  X-RateLimit-Reset: 1739548845                                  │
│                                                                 │
│  {                                                              │
│      "error": "rate_limit_exceeded",                            │
│      "message": "Rate limit exceeded: 10 per 1 minute",        │
│      "retry_after": 45                                          │
│  }                                                              │
│                                                                 │
│                                                                 │
│  KEY: Retry-After tells the client EXACTLY when to try again.   │
│  A well-behaved client reads this and sleeps. A bad client      │
│  ignores it and gets blocked again. That's their problem.       │
│                                                                 │
│                                                                 │
│  Retry-After can be:                                            │
│  • Seconds (integer):  Retry-After: 45                          │
│  • HTTP date:          Retry-After: Fri, 14 Feb 2026 12:30:00   │
│                                                                 │
│  Prefer seconds — simpler for clients to parse.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Full Circle: Server ↔ Client (Connection to Week 8)

**Remember Week 8? You learned to RESPECT rate limits as a client. Now close the loop.**

```
┌─────────────────────────────────────────────────────────────────┐
│              WEEK 8 (Client) ←→ WEEK 12 (Server)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 8: You were the CLIENT                                    │
│  ────────────────────────────                                   │
│  • You read X-RateLimit-Remaining headers                       │
│  • You implemented exponential backoff (tenacity)               │
│  • You respected Retry-After                                    │
│  • You built client-side rate limiting (token bucket)           │
│                                                                 │
│  WEEK 12: You are now the SERVER                                │
│  ─────────────────────────────                                  │
│  • You SEND X-RateLimit-* headers                               │
│  • You RETURN 429 + Retry-After                                 │
│  • You ENFORCE server-side rate limiting                        │
│  • You PROTECT your resources from abusive clients              │
│                                                                 │
│                                                                 │
│  THE FULL PICTURE:                                              │
│                                                                 │
│  [Your Client Code]  ──request──▶  [External API]               │
│  (Week 8: respect                  (Their rate limiter)         │
│   their limits)                                                 │
│                                                                 │
│  [Someone's Client]  ──request──▶  [YOUR API]                   │
│  (Their responsibility             (Week 12: YOUR rate limiter) │
│   to back off)                                                  │
│                                                                 │
│  You've now built BOTH sides of the contract.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "In Week 8 you learned to be a good guest at someone else's party. Now you're the host. Set clear rules, communicate them in headers, and enforce them fairly."

---

# PART 5: MEASURING IMPACT

## 5.1 Why Averages Lie

**Before we measure rate limiting's impact, we need to understand HOW to measure.**

> "I'm about to show you why 'average response time' is the most dangerous metric in your dashboard."

```python
# 100 API requests. 99 are fast. 1 is a disaster.
response_times_ms = [12, 11, 13, 10, 14, 12, 11, 13, 10, 12,  # ... 
                     11, 13, 12, 10, 14, 11, 12, 13, 10, 11,
                     # ... (99 requests between 10-14ms)
                     4200]  # One request took 4.2 seconds

average = sum(response_times_ms) / len(response_times_ms)
# average ≈ 53ms
```

> "53ms average. Dashboard shows green. Ship it?"

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY AVERAGES LIE                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  100 requests:                                                  │
│  ├─ 99 requests at ~12ms                                        │
│  └─  1 request  at 4200ms                                       │
│                                                                 │
│  Average:  53ms      ← "Looks fine!" 😊                         │
│  Reality:  1 user waited 4.2 SECONDS                            │
│                                                                 │
│  Now scale it:                                                  │
│  • 10,000 requests/day at 1% failure = 100 terrible experiences │
│  • 1,000,000 requests/day at 1% = 10,000 angry users           │
│                                                                 │
│  The average HIDES the suffering of the tail.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

A handful of very slow outliers (GC pauses, cold starts, retries, network hiccups, lock contention) can inflate the average without affecting most users.[[1]](https://oneuptime.com/blog/post/2025-09-15-p50-vs-p95-vs-p99-latency-percentiles/view)

Don't trust averages — they lie. Percentiles tell you what your users are actually experiencing.[[4]](https://medium.com/@subodh.shetty87/not-all-slowness-is-equal-a-developers-guide-to-p50-p95-and-p99-latencies-c473b9ea6fb9)

---

## 5.2 Percentile Latencies (p50, p95, p99)

**Percentiles show you the DISTRIBUTION of response times, not just one misleading number.**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT PERCENTILES MEAN                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sort ALL response times from fastest to slowest:               │
│                                                                 │
│  User   1:  ██ 10ms                                             │
│  User   2:  ██ 11ms                                             │
│  ...                                                            │
│  User  50:  ███ 13ms            ← p50 (MEDIAN): 13ms           │
│  ...                               "Typical" user experience    │
│  User  95:  ██████ 45ms         ← p95: 45ms                    │
│  ...                               5% of users are slower       │
│  User  99:  █████████████ 180ms ← p99: 180ms                   │
│  User 100:  ██████████████████████████████ 4200ms               │
│                                                                 │
│                                                                 │
│  p50 (median):  13ms   "What a TYPICAL user sees"               │
│  p95:           45ms   "What an UNLUCKY user sees"              │
│  p99:          180ms   "What the WORST 1% experience"           │
│  Average:       53ms   "A number that describes NOBODY"         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to calculate them:**

```python
import numpy as np

response_times = [12, 11, 45, 13, 180, 10, 14, 12, 4200, 11, ...]  
# (pretend this is 100 values)

p50 = np.percentile(response_times, 50)   # Median
p95 = np.percentile(response_times, 95)   # 95th percentile
p99 = np.percentile(response_times, 99)   # 99th percentile

print(f"p50: {p50:.0f}ms  p95: {p95:.0f}ms  p99: {p99:.0f}ms")
```

**What each tells you:**

```
┌─────────────────────────────────────────────────────────────────┐
│              PERCENTILE CHEAT SHEET                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  p50 (median)                                                   │
│  ─────────────                                                  │
│  The "typical" experience. Half faster, half slower.            │
│  Use it to: Detect broad regressions affecting everyone.        │
│  If p50 is bad, your whole system is slow.                      │
│                                                                 │
│  p95                                                            │
│  ────                                                           │
│  The "bad luck" experience. 1 in 20 users sees this.            │
│  Use it to: Tune system performance. Set SLOs.                  │
│  This is usually the PRIMARY metric for SLOs.                   │
│                                                                 │
│  p99                                                            │
│  ────                                                           │
│  The "worst reasonable case." 1 in 100 users.                   │
│  Use it to: Expose architectural bottlenecks.                   │
│  If p99 is 10x worse than p50, something is very wrong.         │
│                                                                 │
│                                                                 │
│  HEALTHY SYSTEM:                                                │
│  p50: 20ms    p95: 40ms     p99: 80ms                           │
│  (Tight spread. Consistent. Good.)                              │
│                                                                 │
│  SICK SYSTEM:                                                   │
│  p50: 20ms    p95: 200ms    p99: 2000ms                         │
│  (Huge spread. Something is occasionally VERY slow.)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

A healthy system has percentiles that are relatively close together — p50 of 20ms, p95 of 40ms, p99 of 80ms. The slow requests are only a few times slower than the fast ones.[[8]](https://www.zoyla.app/resources/latency-percentiles-guide) A problematic system has a long tail — p50 of 20ms, p95 of 200ms, p99 of 2000ms. Something is causing occasional massive slowdowns — maybe a database query that sometimes hits a slow path, maybe garbage collection pauses, maybe external service timeouts. The gap between p50 and p99 tells you how consistent your performance is.[[8]](https://www.zoyla.app/resources/latency-percentiles-guide)

**Now ask the class:**

> "Your Task Manager API has p50 of 35ms but p99 of 3200ms. Is your API healthy? Where would you start investigating?"

The answer: If your p99 is bad but p50 is fine, you have an outlier problem — something is occasionally slow. Common causes: slow database queries that only trigger on certain data, external services with variable response times, and resource contention under load.[[8]](https://www.zoyla.app/resources/latency-percentiles-guide) They should check their EXPLAIN plans from Week 7, their Redis cache hit rates from Week 10, and their external API response times from Week 8.

---

## 5.3 Load Testing with Rate Limits (Connection to Lecture 3)

**In Lecture 3 you learned locust. Now use it to verify your rate limiting works.**

The test has two phases: measure WITHOUT limits (baseline), then WITH limits (protected).

```python
# tests/load/locustfile.py
from locust import HttpUser, task, between


class TaskManagerUser(HttpUser):
    """Simulates a typical user of our Task Manager API."""
    wait_time = between(0.5, 2)    # Random wait between requests
    host = "http://localhost:8000"

    def on_start(self):
        """Login once to get a token (Week 9 auth)."""
        response = self.client.post("/api/v1/auth/login", json={
            "email": "loadtest@example.com",
            "password": "testpassword123"
        })
        self.token = response.json()["access_token"]
        self.headers = {"Authorization": f"Bearer {self.token}"}

    @task(5)    # Weight: 5x more likely than create
    def list_tasks(self):
        self.client.get("/api/v1/tasks", headers=self.headers)

    @task(2)
    def get_single_task(self):
        self.client.get("/api/v1/tasks/1", headers=self.headers)

    @task(1)
    def create_task(self):
        self.client.post("/api/v1/tasks", headers=self.headers, json={
            "title": "Load test task",
            "description": "Created during load test",
            "priority": "medium"
        })

    @task(1)
    def search_tasks(self):
        self.client.get(
            "/api/v1/tasks/search?q=load",
            headers=self.headers
        )
```

**Run the load test and observe 429s:**

```bash
# Headless mode — outputs to terminal and CSV
locust --headless \
    --host=http://localhost:8000 \
    -u 50 -r 10 \
    --run-time 60s \
    --csv=results \
    -f tests/load/locustfile.py
```

**What to look for in the output:**

```
┌─────────────────────────────────────────────────────────────────┐
│              LOAD TEST: WITH vs WITHOUT RATE LIMITS             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT RATE LIMITS:                                           │
│  ─────────────────────                                          │
│  Requests:  2847 total, 0 failures                              │
│  p50: 35ms    p95: 180ms    p99: 890ms                          │
│  Status codes: {200: 2847}                                      │
│                                                                 │
│  → All requests served. But p99 is climbing.                    │
│    The server is straining. At 100 users it would buckle.       │
│                                                                 │
│                                                                 │
│  WITH RATE LIMITS (60/min per user):                            │
│  ────────────────────────────────────                           │
│  Requests:  2847 total, 1923 "failures" (429s)                  │
│  p50: 22ms    p95: 45ms     p99: 120ms                          │
│  Status codes: {200: 924, 429: 1923}                            │
│                                                                 │
│  → 429s are NOT failures — they're PROTECTION WORKING.          │
│    The served requests are faster (lower p99) because           │
│    the server isn't drowning.                                   │
│                                                                 │
│  THE INSIGHT:                                                   │
│  Rate limiting made the 200 responses FASTER for everyone.      │
│  Fewer requests served, but served BETTER.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Performance Regression Testing in CI

**You measured performance today. How do you ensure it doesn't degrade tomorrow?**

The answer: **run a lightweight load test in your CI pipeline and fail the build if performance drops.** You'll formalize your CI pipeline in Week 15 — this is the performance testing piece that slots into it.

```python
# scripts/check_performance.py
"""
Read locust CSV output and check against performance thresholds.
Exit with code 1 (fail the build) if any threshold is exceeded.
"""
import csv
import sys


THRESHOLDS = {
    "p95_ms": 200,          # p95 must be under 200ms
    "p99_ms": 500,          # p99 must be under 500ms
    "failure_pct": 5.0,     # Max 5% non-429 failures
}


def check_thresholds(csv_path: str) -> bool:
    passed = True

    with open(csv_path) as f:
        reader = csv.DictReader(f)
        for row in reader:
            if row["Name"] != "Aggregated":
                continue

            p95 = float(row["95%"])
            p99 = float(row["99%"])
            total = int(row["Request Count"])
            failures = int(row["Failure Count"])
            failure_pct = (failures / total) * 100 if total > 0 else 0

            # Check each threshold
            if p95 > THRESHOLDS["p95_ms"]:
                print(f"❌ FAIL: p95 = {p95:.0f}ms (threshold: {THRESHOLDS['p95_ms']}ms)")
                passed = False
            else:
                print(f"✅ PASS: p95 = {p95:.0f}ms")

            if p99 > THRESHOLDS["p99_ms"]:
                print(f"❌ FAIL: p99 = {p99:.0f}ms (threshold: {THRESHOLDS['p99_ms']}ms)")
                passed = False
            else:
                print(f"✅ PASS: p99 = {p99:.0f}ms")

            if failure_pct > THRESHOLDS["failure_pct"]:
                print(f"❌ FAIL: failure rate = {failure_pct:.1f}% (threshold: {THRESHOLDS['failure_pct']}%)")
                passed = False
            else:
                print(f"✅ PASS: failure rate = {failure_pct:.1f}%")

    return passed


if __name__ == "__main__":
    csv_path = sys.argv[1] if len(sys.argv) > 1 else "results_stats.csv"

    if not check_thresholds(csv_path):
        print("\n🚨 Performance regression detected!")
        sys.exit(1)     # Fail the CI build

    print("\n🎉 All performance thresholds met.")
    sys.exit(0)
```

**How this fits into CI (preview for Week 15):**

```
┌─────────────────────────────────────────────────────────────────┐
│              CI PIPELINE WITH PERFORMANCE GATE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  git push                                                       │
│    │                                                            │
│    ▼                                                            │
│  ┌─────────────────┐                                            │
│  │ 1. Lint (ruff)  │ ← Already doing this (Week 1)             │
│  └────────┬────────┘                                            │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │ 2. Type check   │ ← mypy (Week 1)                           │
│  │    (mypy)       │                                            │
│  └────────┬────────┘                                            │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │ 3. Unit tests   │ ← pytest (Week 2)                         │
│  │    (pytest)     │                                            │
│  └────────┬────────┘                                            │
│           ▼                                                     │
│  ┌─────────────────────────────────────────────┐                │
│  │ 4. Start services (docker compose up)       │ ← NEW         │
│  │    Run load test (locust --headless, 60s)   │                │
│  │    Check thresholds (check_performance.py)  │                │
│  └────────┬────────────────────────────────────┘                │
│           │                                                     │
│      PASS │  FAIL                                               │
│           │    │                                                │
│           ▼    ▼                                                │
│         Deploy  Block + Notify                                  │
│                                                                 │
│  The performance gate catches regressions BEFORE production.    │
│  "We don't ship code that makes the API slower."                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Setting Thresholds and Alerts

**How do you decide what "too slow" means?**

```
┌─────────────────────────────────────────────────────────────────┐
│              THRESHOLD GUIDELINES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: MEASURE YOUR BASELINE                                  │
│  ──────────────────────────────                                 │
│  Run your load test on the CURRENT healthy system.              │
│  Record p50, p95, p99. This is your baseline.                   │
│                                                                 │
│  Example baseline:                                              │
│    p50: 25ms    p95: 80ms    p99: 150ms                         │
│                                                                 │
│                                                                 │
│  STEP 2: SET THRESHOLDS WITH HEADROOM                           │
│  ──────────────────────────────────────                         │
│  Add 2x-3x margin above your baseline.                         │
│  Thresholds that are too tight cause false alarms.              │
│  Thresholds that are too loose miss real regressions.           │
│                                                                 │
│  Threshold recommendation:                                      │
│    p95 threshold: 200ms   (2.5x baseline)                       │
│    p99 threshold: 500ms   (3.3x baseline)                       │
│    Failure rate: < 1% (excluding 429s)                          │
│                                                                 │
│                                                                 │
│  STEP 3: WATCH THE RATIO                                        │
│  ────────────────────────                                       │
│  The gap between p50 and p99 is as important as                 │
│  the absolute values.                                           │
│                                                                 │
│  p99 / p50 < 5x  → Consistent. Healthy.                        │
│  p99 / p50 > 10x → Investigate! Something is occasionally      │
│                     very slow (missing index? cold cache?       │
│                     external API timeout?)                      │
│                                                                 │
│                                                                 │
│  STEP 4: DISTINGUISH 429s FROM REAL ERRORS                      │
│  ──────────────────────────────────────────                     │
│  In your load test results, 429 is NOT a failure.               │
│  It's your rate limiter WORKING AS DESIGNED.                    │
│  Filter 429s out when calculating error rate.                   │
│  Only 5xx and unexpected 4xx count as real failures.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Use p50 to detect broad regressions, p95 to tune system performance, p99 to expose architectural bottlenecks and outliers.[[1]](https://oneuptime.com/blog/post/2025-09-15-p50-vs-p95-vs-p99-latency-percentiles/view)

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              RATE LIMITING QUICK REFERENCE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INSTALL:                                                       │
│      pip install slowapi                                        │
│                                                                 │
│  SETUP:                                                         │
│      limiter = Limiter(key_func=get_remote_address)             │
│      app.state.limiter = limiter                                │
│      app.add_exception_handler(RateLimitExceeded, handler)      │
│                                                                 │
│  DECORATE (route decorator ABOVE limit decorator):              │
│      @app.get("/items")                                         │
│      @limiter.limit("30/minute")                                │
│      async def get_items(request: Request):  # ← Required!     │
│                                                                 │
│  LIMIT STRINGS:                                                 │
│      "5/minute"         "100/hour"         "1000/day"           │
│      "5/minute;100/hour"  (multiple, semicolon-separated)       │
│                                                                 │
│  KEY FUNCTIONS:                                                 │
│      get_remote_address          → limit by IP                  │
│      custom: return user.id      → limit by user                │
│      per-endpoint override       → key_func= argument           │
│                                                                 │
│  REDIS BACKEND:                                                 │
│      Limiter(key_func=..., storage_uri="redis://host:6379/1")   │
│                                                                 │
│  EXEMPT AN ENDPOINT:                                            │
│      @limiter.exempt                                            │
│                                                                 │
│  GLOBAL DEFAULT:                                                │
│      Limiter(key_func=..., default_limits=["100/minute"])       │
│                                                                 │
│  HEADERS TO SEND:                                               │
│      X-RateLimit-Limit     → quota total                        │
│      X-RateLimit-Remaining → requests left                      │
│      X-RateLimit-Reset     → when window resets (epoch)         │
│      Retry-After           → seconds until retry (on 429 only)  │
│                                                                 │
│  PERCENTILES:                                                   │
│      p50 → median ("typical" user)                              │
│      p95 → tail latency ("unlucky" user, use for SLOs)          │
│      p99 → critical tail ("worst 1%", find bottlenecks)         │
│                                                                 │
│  COMMON MISTAKES:                                               │
│      ❌ Missing request: Request in endpoint signature           │
│      ❌ Wrong decorator order (route must be above limit)        │
│      ❌ In-memory storage with multiple server instances         │
│      ❌ Trusting average response time instead of percentiles    │
│      ❌ Counting 429s as "failures" in load test reports         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  RATE LIMITING = CONTROLLED ACCESS                              │
│                                                                 │
│  Your API has finite resources. Rate limiting ensures            │
│  those resources are shared fairly, protected from abuse,       │
│  and consumed within your cost budget.                          │
│                                                                 │
│  ┌──────────┐     ┌──────────┐     ┌──────────────┐            │
│  │ Incoming │ ──▶ │ Rate     │ ──▶ │ Your API     │            │
│  │ Request  │     │ Limiter  │     │ (protected)  │            │
│  └──────────┘     └──────────┘     └──────────────┘            │
│       │                │                                        │
│       │          ┌─────┴─────┐                                  │
│       │          │   Under   │ YES → Allow through              │
│       │          │   limit?  │                                  │
│       │          └─────┬─────┘                                  │
│       │                │ NO                                     │
│       │                ▼                                        │
│       │          ┌───────────┐                                  │
│       │          │ 429 + Tell│                                  │
│       │          │ them when │                                  │
│       │          │ to retry  │                                  │
│       │          └───────────┘                                  │
│                                                                 │
│                                                                 │
│  THE NIGHTCLUB BOUNCER:                                         │
│  ├─ Key function   = How the bouncer identifies you             │
│  ├─ Algorithm      = How the bouncer counts                     │
│  ├─ slowapi        = The bouncer's toolkit                      │
│  ├─ Redis backend  = Shared clipboard across all doors          │
│  ├─ Rate headers   = Stamp on your hand showing quota left      │
│  ├─ 429 + Retry    = "Come back in 45 seconds"                  │
│  └─ Percentiles    = How you MEASURE the bouncer's impact       │
│                                                                 │
│                                                                 │
│  MEASURE WITH PERCENTILES, NOT AVERAGES:                        │
│  ├─ p50 = Typical experience                                    │
│  ├─ p95 = The SLO target (what you promise your users)          │
│  ├─ p99 = Where hidden problems live                            │
│  └─ avg = A liar. Never trust it alone.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming Content

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEK 12 PROJECT (this week):                                   │
│  └─ Load test your API. Target: 500 req/min, <200ms p95.       │
│     Rate limiting is one of the 3 documented optimizations.     │
│     Before/after metrics required.                              │
│                                                                 │
│  WEEK 13-14 (Capstone):                                         │
│  └─ Multi-tenant SaaS backend                                   │
│     Rate limits PER TENANT (org A gets 100/min,                 │
│     org B gets 500/min). Dynamic limits, not hardcoded.         │
│     This is real SaaS product design.                           │
│                                                                 │
│  WEEK 15 (CI/CD):                                               │
│  └─ Your check_performance.py script becomes a real             │
│     GitHub Actions step. Performance regression testing         │
│     runs on every pull request. Automated quality gates.        │
│                                                                 │
│  WEEK 16 (System Design):                                       │
│  └─ "Design a rate limiter" is a CLASSIC system design          │
│     interview question. You now understand the algorithms,      │
│     the distributed storage (Redis), the trade-offs,            │
│     and the protocol (headers). You've BUILT one.               │
│     Most candidates can only describe one.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```