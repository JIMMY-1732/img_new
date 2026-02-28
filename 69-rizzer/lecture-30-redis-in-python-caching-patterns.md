# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, SOLUTION EARNED                                 │
│  ──────────────────────────────                                 │
│  Students will watch 100 identical database queries hammer      │
│  PostgreSQL before they write a single line of cache code.      │
│  Make the waste VISIBLE.                                        │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Caching is abstract. We use a Bookshelf vs Library analogy     │
│  throughout. Every concept maps to something physical.          │
│                                                                 │
│  BRIDGE FROM CLI TO CODE                                        │
│  ──────────────────────                                         │
│  Students learned Redis CLI in the previous lecture.            │
│  Every Python operation maps 1:1 to a CLI command they know.   │
│  Old knowledge is the scaffold for new knowledge.               │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  async/await (Wk 1)    → redis.asyncio is async-native         │
│  Depends() (Wk 3)      → Redis injected like DB session        │
│  Pydantic (Wk 3)       → model_dump_json / model_validate_json │
│  Repository (Wk 6)     → Cache wraps the repository layer      │
│  Conn pooling (Wk 7)   → Same concept, different datastore     │
│                                                                 │
│  PATTERNS BEFORE SYNTAX                                         │
│  ──────────────────────                                         │
│  Understand WHY cache-aside works before typing the code.       │
│  Understand WHY invalidation is hard before trying to solve it. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│               REDIS IN PYTHON & CACHING PATTERNS                │
│                     (3–4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (35 min)                                   │
│  ├─ 1.1 The Repeated Query (Demonstration)                      │
│  ├─ 1.2 What is Caching? (The Bookshelf Analogy)                │
│  ├─ 1.3 The Speed Hierarchy                                     │
│  └─ 1.4 Cache Hit vs Cache Miss                                 │
│                                                                 │
│  PART 2: REDIS IN PYTHON (50 min)                               │
│  ├─ 2.1 redis.asyncio — Your Async Redis Client                 │
│  ├─ 2.2 Connection Pooling (Reuse, Don't Reconnect)             │
│  ├─ 2.3 CLI → Python (The Translation Table)                    │
│  ├─ 2.4 Serialization (Python Objects ↔ Redis Strings)          │
│  └─ 2.5 Redis as a FastAPI Dependency                           │
│                                                                 │
│  PART 3: THE CACHE-ASIDE PATTERN (50 min)                       │
│  ├─ 3.1 The Fundamental Pattern                                 │
│  ├─ 3.2 Cache Key Design (The Art of Naming)                    │
│  ├─ 3.3 Full Read Implementation                                │
│  └─ 3.4 What Happens on Writes?                                 │
│                                                                 │
│  PART 4: CACHE INVALIDATION (40 min)                            │
│  ├─ 4.1 The Hardest Problem in Computer Science                 │
│  ├─ 4.2 Time-Based Invalidation (TTL)                           │
│  ├─ 4.3 Event-Based Invalidation (Active Deletion)              │
│  └─ 4.4 The Staleness Trade-off (A Business Decision)           │
│                                                                 │
│  PART 5: WHEN CACHING BREAKS (35 min)                           │
│  ├─ 5.1 Cache Stampede (The Thundering Herd)                    │
│  ├─ 5.2 Cache Penetration (The Lookup That Never Succeeds)      │
│  ├─ 5.3 Graceful Degradation (Redis is Down, Now What?)         │
│  └─ 5.4 Monitoring: Is Your Cache Actually Helping?             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Repeated Query

**Start with a demonstration. Make them see the waste.**

```python
# demo_no_cache.py — Run this with students watching
import asyncio
import time

async def db_get_task(task_id: int) -> dict:
    """Simulate a PostgreSQL query — JOIN across 3 tables"""
    await asyncio.sleep(0.05)  # 50ms — realistic for a JOIN
    return {
        "id": task_id,
        "title": "Design caching layer",
        "assignee": "Alice",
        "org": "Acme Corp",
        "tags": ["backend", "performance"],
    }

async def handle_request(task_id: int) -> dict:
    """One API request → one database query"""
    return await db_get_task(task_id)

async def main():
    task_id = 42  # Same task, every time.

    start = time.time()

    # Simulate 200 users requesting the same popular task
    results = await asyncio.gather(
        *[handle_request(task_id) for _ in range(200)]
    )

    elapsed = time.time() - start
    print(f"Requests:      200")
    print(f"Total time:    {elapsed:.2f}s")
    print(f"Avg per req:   {elapsed / 200 * 1000:.1f}ms")
    print(f"DB queries:    200")  # ← THIS is the problem

asyncio.run(main())
```

**Run it. Read the numbers aloud.**

```
Requests:      200
Total time:    0.52s
Avg per req:   2.6ms
DB queries:    200
```

**Now ask the class:**

> "200 requests. 200 database queries. Every single query returns THE SAME data. The task didn't change between request #1 and request #200. So why did we ask PostgreSQL the same question 200 times?"

Pause. Let the silence sit.

> "Imagine this is a real product. A task goes viral — the CEO assigns it in a company all-hands. 10,000 employees open it within 60 seconds. That's 10,000 identical database queries in one minute. Your PostgreSQL server is sweating. Your response times spike. Users see loading spinners. All because you kept asking a question you already knew the answer to."

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE WASTE                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request 1 → SELECT ... FROM tasks JOIN ... → { task data }    │
│  Request 2 → SELECT ... FROM tasks JOIN ... → { task data }    │
│  Request 3 → SELECT ... FROM tasks JOIN ... → { task data }    │
│      ...            (same query, same result)                   │
│  Request 200 → SELECT ... FROM tasks JOIN ... → { task data }  │
│                                                                 │
│  SAME QUESTION. SAME ANSWER. 200 TIMES.                         │
│                                                                 │
│  Database work:  200 queries × 50ms = 10,000ms of DB time      │
│  Network trips:  200 round-trips to PostgreSQL                  │
│  Connection use: 200 sessions from your pool                    │
│                                                                 │
│  What if we just... remembered the answer?                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now show what happens WITH a cache:**

```python
# demo_with_cache.py — same scenario, one change
import asyncio
import time

# Simulated cache — just a dict for now
_cache: dict[str, dict] = {}

async def db_get_task(task_id: int) -> dict:
    """Still 50ms per query"""
    await asyncio.sleep(0.05)
    return {
        "id": task_id,
        "title": "Design caching layer",
        "assignee": "Alice",
        "org": "Acme Corp",
        "tags": ["backend", "performance"],
    }

db_query_count = 0

async def handle_request(task_id: int) -> dict:
    global db_query_count
    cache_key = f"tasks:{task_id}"

    # Check cache first
    if cache_key in _cache:
        return _cache[cache_key]  # Instant — no DB trip

    # Cache miss — go to database
    db_query_count += 1
    result = await db_get_task(task_id)
    _cache[cache_key] = result  # Remember for next time
    return result

async def main():
    global db_query_count
    task_id = 42

    start = time.time()

    results = await asyncio.gather(
        *[handle_request(task_id) for _ in range(200)]
    )

    elapsed = time.time() - start
    print(f"Requests:      200")
    print(f"Total time:    {elapsed:.4f}s")
    print(f"Avg per req:   {elapsed / 200 * 1000:.2f}ms")
    print(f"DB queries:    {db_query_count}")  # ← Watch this

asyncio.run(main())
```

```
Requests:      200
Total time:    0.0503s
Avg per req:   0.25ms
DB queries:    1
```

**Let the numbers speak:**

```
┌─────────────────────────────────────────────────────────────────┐
│                BEFORE vs AFTER                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                        No Cache         With Cache              │
│                        ────────         ──────────              │
│  DB Queries:           200              1                       │
│  Total time:           0.52s            0.05s                   │
│  Avg latency:          2.6ms            0.25ms                  │
│  DB load:              100%             0.5%                    │
│                                                                 │
│  10× faster. 200× fewer DB queries. Same correct result.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "That's what caching buys you. One database query instead of two hundred. The other 199 requests never touch PostgreSQL. The database barely notices."

---

## 1.2 What is Caching? (The Bookshelf Analogy)

**We'll use this analogy for the rest of the lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE BOOKSHELF ANALOGY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You're writing a research paper. You need facts constantly.    │
│                                                                 │
│                                                                 │
│  THE CITY LIBRARY (PostgreSQL)                                  │
│  ─────────────────────────────                                  │
│  • Has EVERY book ever published                                │
│  • Always up-to-date (librarians maintain it)                   │
│  • 20-minute drive to get there                                 │
│  • Comprehensive, authoritative, slow to access                 │
│                                                                 │
│                                                                 │
│  YOUR BOOKSHELF AT HOME (Redis Cache)                           │
│  ─────────────────────────────────────                          │
│  • Holds 50 books maximum (limited space)                       │
│  • Only has books YOU chose to bring home                       │
│  • 5-second walk to reach                                       │
│  • Fast, limited, might have outdated editions                  │
│                                                                 │
│                                                                 │
│  YOUR WORKFLOW:                                                  │
│  1. Need a fact → Check your bookshelf first (5 seconds)        │
│  2. Book is there → Use it (CACHE HIT)                          │
│  3. Book is NOT there → Drive to library, get it,               │
│     put a copy on your shelf, use it (CACHE MISS)               │
│  4. Bookshelf full? Remove the book you haven't touched          │
│     in months to make room (EVICTION)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy:**

```
Bookshelf                │  Caching System
─────────────────────────│───────────────────────────
City Library             │  PostgreSQL (source of truth)
Your bookshelf           │  Redis (cache layer)
A book on your shelf     │  Cached value
Walking to your shelf    │  Redis GET (~1ms)
Driving to the library   │  PostgreSQL query (~5-50ms)
Book found on shelf      │  Cache hit
Book not on shelf        │  Cache miss
Bringing a book home     │  Writing to cache after DB query
Shelf has limited space  │  Redis has limited memory
Book gets new edition    │  Data changes in the database
Removing your old copy   │  Cache invalidation
Sticky note: "re-check   │  TTL (Time-To-Live)
  after March 2026"      │
```

**The core principle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CACHING = Keeping a COPY of expensive-to-get data             │
│            in a FAST-to-access location                         │
│            for a LIMITED amount of time.                        │
│                                                                 │
│  Every word matters:                                            │
│  • COPY     → Cache is never the source of truth               │
│  • EXPENSIVE → Don't cache what's already fast                  │
│  • FAST     → Cache must be faster than the original source     │
│  • LIMITED  → Cached data expires or gets evicted               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Speed Hierarchy

**Why Redis is fast — it's about WHERE data lives:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE SPEED HIERARCHY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Location              Access Time     Analogy                  │
│  ─────────────────     ───────────     ────────────────         │
│  CPU Register          < 1 ns          Your own brain           │
│  L1/L2 Cache           1–10 ns         Sticky note on monitor   │
│  RAM                   50–100 ns       Bookshelf in your room   │
│  SSD (local)           10–100 μs       Filing cabinet downstairs│
│  Network (Redis)       100–500 μs      Neighbor's bookshelf     │
│  Network (PostgreSQL)  1–50 ms         City Library             │
│  External API          50–500 ms       Library in another city  │
│                                                                 │
│  ────────────────────────────────────────────────────────       │
│  Redis lives in RAM → ~100–1000× faster than PostgreSQL         │
│  PostgreSQL lives on disk + network                             │
│                                                                 │
│  That's why our demo went from 2.6ms to 0.25ms per request.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Redis is fast for the same reason your bookshelf is fast: the data is RIGHT THERE, already in memory, no disk seeks, no query planning, no JOIN operations. Just a key lookup in a hash table."

---

## 1.4 Cache Hit vs Cache Miss

**Every cache lookup has exactly two outcomes:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE TWO PATHS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                     Request arrives                             │
│                          │                                      │
│                          ▼                                      │
│                  ┌───────────────┐                               │
│                  │ Check Cache   │                               │
│                  │ (Redis GET)   │                               │
│                  └───────┬───────┘                               │
│                          │                                      │
│                ┌─────────┴─────────┐                            │
│                │                   │                            │
│            HIT ▼               MISS ▼                           │
│       ┌────────────┐      ┌────────────┐                        │
│       │ Return     │      │ Query      │                        │
│       │ cached     │      │ PostgreSQL │                        │
│       │ data       │      │            │                        │
│       └────────────┘      └─────┬──────┘                        │
│            │                    │                               │
│            │                    ▼                               │
│            │              ┌────────────┐                        │
│            │              │ Store in   │                        │
│            │              │ cache      │                        │
│            │              └─────┬──────┘                        │
│            │                    │                               │
│            ▼                    ▼                               │
│       ┌──────────────────────────────┐                          │
│       │       Return to client       │                          │
│       └──────────────────────────────┘                          │
│                                                                 │
│  HIT:  ~1ms   (bookshelf)                                      │
│  MISS: ~50ms  (library trip + put on bookshelf for next time)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The goal: maximize hits, minimize misses.**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  HIT RATE = Cache Hits / Total Lookups                          │
│                                                                 │
│  90% hit rate on 10,000 requests:                               │
│  • 9,000 served from Redis   (9,000 × 1ms  =   9s)             │
│  • 1,000 hit PostgreSQL      (1,000 × 50ms = 50s)              │
│                                                                 │
│  0% hit rate (no cache):                                        │
│  • 10,000 hit PostgreSQL     (10,000 × 50ms = 500s)            │
│                                                                 │
│  DB load reduced by 90%. That's the point.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 2: REDIS IN PYTHON

## 2.1 redis.asyncio — Your Async Redis Client

**The `redis` package (redis-py) includes a fully async interface.**

```python
# Installation — you may already have this from Lecture 1's CLI work
# pip install redis
# (or in your pyproject.toml / requirements.txt)
```

**Your first async connection:**

```python
import redis.asyncio as redis

async def main():
    # Connect to Redis (same server you used in CLI last lecture)
    client = redis.Redis(
        host="localhost",
        port=6379,
        db=0,
        decode_responses=True,  # ← IMPORTANT — explained below
    )

    # Same operations you did in CLI, now in Python
    await client.set("greeting", "hello from python")
    value = await client.get("greeting")
    print(value)  # "hello from python"

    # Always close when done
    await client.aclose()

asyncio.run(main())
```

> "Every Redis command you typed in the CLI last lecture is now an `await` call on the client object. Same commands, same server, async Python."

**The `decode_responses` flag:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  decode_responses                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Redis stores everything as bytes. By default, redis-py         │
│  returns raw bytes:                                             │
│                                                                 │
│  decode_responses=False (default):                              │
│  ─────────────────────────────────                              │
│  >>> await client.get("greeting")                               │
│  b"hello from python"               ← bytes, not str           │
│                                                                 │
│  decode_responses=True (what we want):                          │
│  ─────────────────────────────────────                          │
│  >>> await client.get("greeting")                               │
│  "hello from python"                ← clean string             │
│                                                                 │
│  ALWAYS set decode_responses=True for application code.         │
│  You'll be storing JSON strings — you want str, not bytes.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connecting from a URL (cleaner for configuration):**

```python
# From a URL — common when reading from environment variables
# (You'll use pydantic-settings for this in Week 15)
client = redis.from_url(
    "redis://localhost:6379/0",
    decode_responses=True,
)
```

---

## 2.2 Connection Pooling (Reuse, Don't Reconnect)

> "You learned about connection pooling for PostgreSQL back in Week 7 — `pool_size`, `max_overflow`. Same concept here: opening a new TCP connection to Redis for every operation is wasteful. Pool them."

```
┌─────────────────────────────────────────────────────────────────┐
│              WITHOUT POOLING vs WITH POOLING                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT POOLING:                                               │
│  ────────────────                                               │
│  Request 1 → Open TCP conn → Redis GET → Close TCP conn        │
│  Request 2 → Open TCP conn → Redis GET → Close TCP conn        │
│  Request 3 → Open TCP conn → Redis GET → Close TCP conn        │
│                                                                 │
│  Each TCP handshake adds ~1ms. Under load, you exhaust          │
│  file descriptors. Connection setup becomes the bottleneck.     │
│                                                                 │
│                                                                 │
│  WITH POOLING:                                                  │
│  ──────────────                                                 │
│  Pool: [conn1, conn2, conn3, conn4, conn5]                      │
│                                                                 │
│  Request 1 → Borrow conn1 → Redis GET → Return conn1           │
│  Request 2 → Borrow conn2 → Redis GET → Return conn2           │
│  Request 3 → Borrow conn1 → Redis GET → Return conn1  (reused!)│
│                                                                 │
│  Connections created ONCE, reused MANY times.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Setting up a connection pool:**

```python
import redis.asyncio as redis

# Create the pool ONCE — at application startup
pool = redis.ConnectionPool.from_url(
    "redis://localhost:6379/0",
    max_connections=20,        # Max simultaneous connections
    decode_responses=True,
)

# Create client that uses the pool
client = redis.Redis(connection_pool=pool)

# Every operation borrows a connection from the pool,
# uses it, and returns it automatically.
await client.set("key", "value")   # Borrows conn, returns it
await client.get("key")            # Borrows conn, returns it

# When app shuts down — close the pool
await client.aclose()
```

> "Here's the good news: `redis.Redis()` actually creates a connection pool internally by default. When you write `redis.Redis(host=..., port=...)`, you already get pooling. The explicit `ConnectionPool` gives you control over `max_connections` — useful when you know your concurrency level."

```
┌─────────────────────────────────────────────────────────────────┐
│                  POOL SIZING RULE OF THUMB                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  max_connections should be ≥ your peak concurrent Redis calls.  │
│                                                                 │
│  FastAPI with 4 workers, each handling ~50 concurrent requests: │
│  → max_connections = 50 per worker is a safe starting point     │
│                                                                 │
│  Too few connections → requests block waiting for a free conn   │
│  Too many connections → wasted memory, Redis max clients hit    │
│                                                                 │
│  Default in redis-py: 2**31 (effectively unlimited)             │
│  In production: set an explicit limit (10–100 typical)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 CLI → Python (The Translation Table)

> "Everything you typed in the Redis CLI last lecture has a Python equivalent. Here's the Rosetta Stone."

```
┌──────────────────────────────────┬─────────────────────────────────────────┐
│  Redis CLI (Lecture 1)           │  Python redis.asyncio (Today)           │
├──────────────────────────────────┼─────────────────────────────────────────┤
│                                  │                                         │
│  STRINGS                         │                                         │
│  SET user:1 "Alice"              │  await r.set("user:1", "Alice")         │
│  GET user:1                      │  val = await r.get("user:1")            │
│  DEL user:1                      │  await r.delete("user:1")               │
│  SETEX user:1 300 "Alice"        │  await r.set("user:1", "Alice", ex=300) │
│  SETNX user:1 "Alice"           │  await r.set("user:1", "Alice", nx=True)│
│  INCR counter                    │  await r.incr("counter")                │
│  TTL user:1                      │  ttl = await r.ttl("user:1")            │
│  EXISTS user:1                   │  exists = await r.exists("user:1")      │
│                                  │                                         │
│  HASHES                          │                                         │
│  HSET user:1 name "Alice"        │  await r.hset("user:1", "name", "Alice")│
│  HGET user:1 name                │  val = await r.hget("user:1", "name")   │
│  HGETALL user:1                  │  data = await r.hgetall("user:1")       │
│                                  │                                         │
│  KEYS / SCAN                     │                                         │
│  KEYS user:*                     │  keys = await r.keys("user:*")          │
│                                  │  (prefer SCAN in production — Week 10)  │
│                                  │                                         │
│  EXPIRY                          │                                         │
│  EXPIRE key 60                   │  await r.expire("key", 60)              │
│  PERSIST key                     │  await r.persist("key")                 │
│                                  │                                         │
└──────────────────────────────────┴─────────────────────────────────────────┘
```

**Quick example pulling it together:**

```python
import redis.asyncio as redis

async def demo_operations():
    r = redis.from_url("redis://localhost:6379/0", decode_responses=True)

    # Strings — same as CLI
    await r.set("tasks:count", "0")
    await r.incr("tasks:count")
    await r.incr("tasks:count")
    count = await r.get("tasks:count")
    print(f"Task count: {count}")  # "2"

    # Set with TTL — same as SETEX
    await r.set("session:abc123", "user_42", ex=3600)  # 1 hour
    ttl = await r.ttl("session:abc123")
    print(f"Session TTL: {ttl}s")  # 3600

    # Hash — same as HSET/HGETALL
    await r.hset("task:42", mapping={
        "title": "Design caching layer",
        "status": "in_progress",
        "assignee_id": "7",
    })
    task_data = await r.hgetall("task:42")
    print(f"Task: {task_data}")
    # {"title": "Design caching layer", "status": "in_progress", "assignee_id": "7"}

    # Check existence — same as EXISTS
    exists = await r.exists("task:42")
    print(f"Exists: {exists}")  # 1 (truthy)

    # Delete — same as DEL
    await r.delete("task:42")

    await r.aclose()
```

**Key difference to remember:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CLI returns:        Python returns:                            │
│  ──────────          ────────────────                           │
│  "OK"                True                                       │
│  (nil)               None                                       │
│  (integer) 1         1                                          │
│  "hello"             "hello"  (with decode_responses=True)      │
│  "hello"             b"hello" (without decode_responses)        │
│                                                                 │
│  GET on missing key:                                            │
│  CLI  → (nil)                                                   │
│  Python → None     ← Check for this! Don't assume data exists. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Serialization (Python Objects ↔ Redis Strings)

**This is where most beginners trip. Pay attention.**

Redis stores **strings** (or bytes). Your Python code works with **objects** — dicts, Pydantic models, objects with datetimes and UUIDs. You can't just shove a Python object into Redis.

```python
# ❌ WRONG: Trying to store a Python dict directly
task = {"id": 42, "title": "Design cache", "created_at": datetime.now()}
await r.set("task:42", task)
# TypeError: Invalid input of type: 'dict'. 
#            Convert to a string or bytes first.
```

**You must serialize (Python → string) and deserialize (string → Python).**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE SERIALIZATION ROUND-TRIP                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Python Object                         Redis                   │
│   ─────────────                         ─────                   │
│                                                                 │
│   TaskResponse(      ── serialize ──▶  '{"id": 42,             │
│     id=42,              (to JSON)       "title": "Design...",  │
│     title="Design..."                   "created_at": "2026..."│
│     created_at=...                      }'                     │
│   )                                                             │
│                                                                 │
│                      ◀── deserialize ──                         │
│                         (from JSON)                             │
│                                                                 │
│   Bookshelf analogy:                                            │
│   You can't put a real building on your shelf.                  │
│   You photograph it (serialize) and store the photo.            │
│   When you need the info, you look at the photo (deserialize). │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Option 1: Pydantic (your best friend — you already know this from Week 3):**

```python
from pydantic import BaseModel
from datetime import datetime
from uuid import UUID

class TaskResponse(BaseModel):
    id: UUID
    title: str
    status: str
    created_at: datetime
    assignee_name: str | None = None

# SERIALIZE: Pydantic model → JSON string → Redis
task = TaskResponse(
    id=UUID("a1b2c3d4-e5f6-7890-abcd-ef1234567890"),
    title="Design caching layer",
    status="in_progress",
    created_at=datetime(2026, 2, 14, 10, 30),
    assignee_name="Alice",
)

json_str: str = task.model_dump_json()
# '{"id":"a1b2c3d4-e5f6-7890-abcd-ef1234567890","title":"Design caching layer",...}'

await r.set("task:a1b2c3d4", json_str, ex=300)


# DESERIALIZE: Redis → JSON string → Pydantic model
cached: str | None = await r.get("task:a1b2c3d4")

if cached is not None:
    task = TaskResponse.model_validate_json(cached)
    # Full Pydantic model with proper types — UUID is a UUID,
    # datetime is a datetime, not strings.
    print(task.title)       # "Design caching layer"
    print(type(task.id))    # <class 'uuid.UUID'>
```

> "Pydantic's `model_dump_json()` handles UUIDs, datetimes, enums — all the types that break regular `json.dumps()`. And `model_validate_json()` converts them back. You already use these models for your API responses. Same models, now also your cache serialization format."

**Option 2: json module (for simple dicts without complex types):**

```python
import json

# ✅ Works for simple dicts
simple_data = {"count": 42, "tags": ["backend", "cache"]}
await r.set("stats", json.dumps(simple_data))

cached = await r.get("stats")
if cached:
    data = json.loads(cached)

# ❌ BREAKS with complex types
from datetime import datetime

bad_data = {"created_at": datetime.now()}
json.dumps(bad_data)
# TypeError: Object of type datetime is not JSON serializable
```

**When to use which:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Pydantic model_dump_json / model_validate_json                │
│  ───────────────────────────────────────────────                │
│  Use when: Caching API responses (TaskResponse, UserResponse)   │
│  Why: Handles UUIDs, datetimes, validation on deserialize       │
│  This will be your DEFAULT for this course.                     │
│                                                                 │
│  json.dumps / json.loads                                        │
│  ───────────────────────                                        │
│  Use when: Simple counters, config data, primitive types only   │
│  Why: Simpler, no Pydantic dependency in utility code           │
│                                                                 │
│  Redis Hashes (HSET / HGETALL)                                  │
│  ─────────────────────────────                                  │
│  Use when: You need to read/write INDIVIDUAL fields             │
│  Why: Update one field without deserializing the whole object   │
│  Limitation: All values are strings — no nested structures      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Caching a list of objects:**

```python
from pydantic import TypeAdapter

# For lists of models, use TypeAdapter
task_list_adapter = TypeAdapter(list[TaskResponse])

# SERIALIZE a list
tasks: list[TaskResponse] = [task1, task2, task3]
json_str = task_list_adapter.dump_json(tasks)
await r.set("tasks:org:acme:page:1", json_str, ex=300)

# DESERIALIZE a list
cached = await r.get("tasks:org:acme:page:1")
if cached:
    tasks = task_list_adapter.validate_json(cached)
    # list[TaskResponse] with proper types
```

---

## 2.5 Redis as a FastAPI Dependency

> "You've been injecting database sessions with `Depends()` since Week 3. Redis follows the exact same pattern."

**Step 1: Create Redis client at app startup using lifespan:**

```python
# app/core/redis.py
import redis.asyncio as redis
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan — startup and shutdown."""
    # STARTUP: Create Redis connection pool
    app.state.redis = redis.from_url(
        "redis://localhost:6379/0",  # Use settings in production
        max_connections=20,
        decode_responses=True,
    )

    yield  # Application runs here

    # SHUTDOWN: Close Redis cleanly
    await app.state.redis.aclose()
```

**Step 2: Create a dependency that provides the client:**

```python
# app/core/dependencies.py
from fastapi import Request
import redis.asyncio as redis

async def get_redis(request: Request) -> redis.Redis:
    """Dependency: provides the Redis client from app state."""
    return request.app.state.redis
```

**Step 3: Use it in your route handlers — right next to your DB session:**

```python
# app/routers/tasks.py
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.dependencies import get_db, get_redis  # Both injected

router = APIRouter(prefix="/tasks", tags=["tasks"])

@router.get("/{task_id}", response_model=TaskResponse)
async def get_task(
    task_id: int,
    db: AsyncSession = Depends(get_db),      # Database session (Week 6)
    cache: redis.Redis = Depends(get_redis),  # Redis client (today!)
):
    # Both db and cache are now available.
    # We'll use them together in Part 3.
    ...
```

**Wire it up in your FastAPI app:**

```python
# app/main.py
from fastapi import FastAPI
from app.core.redis import lifespan
from app.routers import tasks

app = FastAPI(lifespan=lifespan)
app.include_router(tasks.router)
```

**The mental model:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DEPENDENCY INJECTION LAYOUT                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your route handler now receives:                               │
│                                                                 │
│  ┌─────────────────────────────────────────────┐                │
│  │             get_task(task_id, db, cache)     │                │
│  │                     │         │       │      │                │
│  │                     │         │       │      │                │
│  │               Path  │   Depends(      │      │                │
│  │               Param │   get_db)  Depends(    │                │
│  │                     │         │  get_redis)  │                │
│  │                     ▼         ▼       ▼      │                │
│  │                ┌─────┐  ┌────────┐ ┌──────┐  │                │
│  │                │ int │  │AsyncSe-│ │Redis │  │                │
│  │                │     │  │ssion   │ │client│  │                │
│  │                └─────┘  └────────┘ └──────┘  │                │
│  └─────────────────────────────────────────────┘                │
│                                                                 │
│  Same pattern you already know. New dependency, that's all.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: THE CACHE-ASIDE PATTERN

## 3.1 The Fundamental Pattern

**Cache-aside (also called "lazy loading") is the most common caching pattern. You'll use it 90% of the time.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   CACHE-ASIDE PATTERN                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE RULE:                                                      │
│  Your application is responsible for reading from AND writing   │
│  to the cache. The cache doesn't know the database exists.      │
│  The database doesn't know the cache exists. YOUR CODE is       │
│  the coordinator.                                               │
│                                                                 │
│                                                                 │
│  THE READ FLOW:                                                 │
│                                                                 │
│     ┌──────┐   1. GET    ┌───────┐                              │
│     │ App  │ ──────────▶ │ Redis │                              │
│     │      │ ◀────────── │       │                              │
│     │      │   2. HIT?   └───────┘                              │
│     │      │       │                                            │
│     │      │   ┌───┴────┐                                       │
│     │      │  YES       NO                                      │
│     │      │   │         │                                      │
│     │      │   │    3. Query DB                                 │
│     │      │   │    ┌────┴──────┐                               │
│     │      │   │    │PostgreSQL │                               │
│     │      │   │    └────┬──────┘                               │
│     │      │   │         │                                      │
│     │      │   │    4. Store in cache                           │
│     │      │   │    ┌────┴──────┐                               │
│     │      │   │    │   Redis   │                               │
│     │      │   │    └────┬──────┘                               │
│     │      │   │         │                                      │
│     │      │   ▼         ▼                                      │
│     │      │  Return data to client                             │
│     └──────┘                                                    │
│                                                                 │
│                                                                 │
│  Bookshelf analogy:                                             │
│  1. Need a book → check your bookshelf                          │
│  2. Found it? Read it. Done.                                    │
│  3. Not found? Drive to the library. Get the book.              │
│  4. Put a copy on your bookshelf (for next time).               │
│  5. Read it. Done.                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In pseudocode (read this BEFORE the real code):**

```
function get_task(task_id):
    
    cache_key = build_key("tasks", task_id)
    
    cached = cache.GET(cache_key)
    
    if cached is not None:
        return deserialize(cached)       ← CACHE HIT: fast path
    
    task = database.SELECT(task_id)      ← CACHE MISS: slow path
    
    if task is None:
        raise NotFound
    
    cache.SET(cache_key, serialize(task), TTL=300)
    
    return task
```

> "Read this three times. This is the single most important pattern in today's lecture. Every caching decision you make for the rest of this course builds on these seven lines."

---

## 3.2 Cache Key Design (The Art of Naming)

**If serialization is HOW you store data, key design is WHERE you find it.**

A poorly designed key scheme is like an unlabeled bookshelf — you'll put things on it and never find them again, or worse, you'll find the WRONG thing.

```
┌─────────────────────────────────────────────────────────────────┐
│                BAD KEYS vs GOOD KEYS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ BAD KEYS:                                                   │
│  ──────────────                                                 │
│  "task"            → Which task? You have 10,000.               │
│  "data"            → What data? Useless.                        │
│  "42"              → 42 what? A task? A user? An org?           │
│  "user_alice"      → What if two orgs each have an Alice?       │
│                                                                 │
│                                                                 │
│  ✅ GOOD KEYS (namespaced, unambiguous):                        │
│  ──────────────────────────────────────                         │
│  "tasks:42"                    → Task with ID 42                │
│  "users:7:profile"             → User 7's profile               │
│  "orgs:acme:tasks:page:1"     → Acme org's task list, page 1   │
│  "orgs:acme:tasks:count"      → Acme org's total task count    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Three rules for key design:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  RULE 1: NAMESPACE WITH COLONS                                  │
│  ─────────────────────────────                                  │
│  Format:  {entity}:{id}:{sub-resource}                          │
│                                                                 │
│  "tasks:42"                                                     │
│  "tasks:42:comments"                                            │
│  "users:7:profile"                                              │
│  "orgs:acme:members"                                            │
│                                                                 │
│  Colons are a Redis convention — not enforced by Redis,         │
│  but universally used. Makes KEYS/SCAN patterns work:           │
│  KEYS "tasks:*"  → find all task-related keys.                  │
│                                                                 │
│                                                                 │
│  RULE 2: INCLUDE EVERY PARAMETER THAT CHANGES THE RESULT        │
│  ─────────────────────────────────────────────────────          │
│  If two different queries can produce different data,            │
│  they MUST have different cache keys.                            │
│                                                                 │
│  GET /tasks?status=done&page=2                                  │
│  → "tasks:org:acme:status:done:page:2:size:20"                 │
│                                                                 │
│  GET /tasks?status=open&page=1                                  │
│  → "tasks:org:acme:status:open:page:1:size:20"                 │
│                                                                 │
│  If you use the SAME key for both, you serve wrong data.        │
│  This is a cache CORRECTNESS bug, not a performance bug.        │
│                                                                 │
│                                                                 │
│  RULE 3: VERSION YOUR KEYS                                      │
│  ─────────────────────────                                      │
│  When your data shape changes (e.g., you add a field to         │
│  TaskResponse), old cached data has the OLD shape.              │
│                                                                 │
│  "v1:tasks:42"  → old format (title, status)                   │
│  "v2:tasks:42"  → new format (title, status, priority)         │
│                                                                 │
│  Bump the version → old keys are ignored, no stale shapes.      │
│  Simple, brutal, effective.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Build keys programmatically — don't scatter string formatting everywhere:**

```python
# app/core/cache_keys.py

CACHE_VERSION = "v1"

def task_key(task_id: int) -> str:
    """Cache key for a single task."""
    return f"{CACHE_VERSION}:tasks:{task_id}"

def task_list_key(org_id: int, status: str | None, page: int, size: int) -> str:
    """Cache key for a paginated, filtered task list."""
    status_part = status or "all"
    return f"{CACHE_VERSION}:tasks:org:{org_id}:status:{status_part}:page:{page}:size:{size}"

def user_profile_key(user_id: int) -> str:
    """Cache key for a user profile."""
    return f"{CACHE_VERSION}:users:{user_id}:profile"
```

> "Centralizing key construction in one module prevents the #1 cache bug: two parts of your code using slightly different key formats for the same data. If your `get_task` uses `tasks:{id}` but your `update_task` uses `task:{id}` (missing the `s`), you'll never invalidate correctly and you'll spend hours debugging."

---

## 3.3 Full Read Implementation

**Now let's build cache-aside into the service layer of your Task Manager.**

> "You built a `TaskRepository` in Week 6 and a service layer on top of it. We're adding a cache layer ABOVE the service, not inside the repository. The repository talks to the database. The service coordinates cache + repository."

```python
# app/services/task_service.py
import redis.asyncio as redis
from pydantic import TypeAdapter
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.task import Task
from app.schemas.task import TaskResponse
from app.repositories.task_repository import TaskRepository
from app.core.cache_keys import task_key, task_list_key

# For caching lists of tasks
task_list_adapter = TypeAdapter(list[TaskResponse])

# Default TTL: 5 minutes
DEFAULT_TTL = 300


class TaskService:
    def __init__(self, db: AsyncSession, cache: redis.Redis):
        self.repo = TaskRepository(db)
        self.cache = cache

    async def get_task(self, task_id: int) -> TaskResponse:
        """
        Get a single task — cache-aside pattern.
        
        1. Check Redis
        2. If hit → deserialize and return
        3. If miss → query PostgreSQL → cache → return
        """
        key = task_key(task_id)

        # ── STEP 1: Check cache ──
        cached: str | None = await self.cache.get(key)

        if cached is not None:
            # CACHE HIT — deserialize and return immediately
            return TaskResponse.model_validate_json(cached)

        # ── STEP 2: Cache miss — go to database ──
        task: Task | None = await self.repo.get_by_id(task_id)

        if task is None:
            raise TaskNotFoundError(task_id)

        # Convert SQLAlchemy model → Pydantic response model
        response = TaskResponse.model_validate(task)

        # ── STEP 3: Store in cache for next time ──
        await self.cache.set(
            key,
            response.model_dump_json(),
            ex=DEFAULT_TTL,
        )

        return response
```

**Using it in a route handler:**

```python
@router.get("/{task_id}", response_model=TaskResponse)
async def get_task(
    task_id: int,
    db: AsyncSession = Depends(get_db),
    cache: redis.Redis = Depends(get_redis),
):
    service = TaskService(db, cache)
    return await service.get_task(task_id)
```

**Caching a paginated list (more complex key, same pattern):**

```python
    async def list_tasks(
        self,
        org_id: int,
        status: str | None = None,
        page: int = 1,
        size: int = 20,
    ) -> list[TaskResponse]:
        """List tasks with caching — paginated and filtered."""

        key = task_list_key(org_id, status, page, size)

        # ── Check cache ──
        cached = await self.cache.get(key)
        if cached is not None:
            return task_list_adapter.validate_json(cached)

        # ── Cache miss ──
        tasks = await self.repo.list_tasks(
            org_id=org_id, status=status, page=page, size=size
        )
        response = [TaskResponse.model_validate(t) for t in tasks]

        # ── Store in cache ──
        await self.cache.set(
            key,
            task_list_adapter.dump_json(response),
            ex=DEFAULT_TTL,
        )

        return response
```

**Ask the class:**

> "Look at `list_tasks`. The key includes `org_id`, `status`, `page`, and `size`. What happens if I forget to include `status` in the key? Say a user requests `status=done` and a different user requests `status=open` — both for the same org, page, and size. Walk me through what goes wrong."

Answer: They get the same cached data. Whoever queries first determines what EVERYONE sees until the cache expires. The second user sees the WRONG data. This is a **correctness bug**, not just a performance issue.

---

## 3.4 What Happens on Writes?

**Cache-aside handles reads. What about when data CHANGES?**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE WRITE PROBLEM (PREVIEW)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Scenario:                                                      │
│  1. Task 42 is cached: {"title": "Old title", ...}             │
│  2. User updates Task 42's title to "New title" (via PUT)       │
│  3. Database now has "New title"                                │
│  4. Cache STILL has "Old title"                                 │
│  5. Next GET request → cache hit → returns "Old title" ❌       │
│                                                                 │
│  THE CACHE IS LYING.                                            │
│                                                                 │
│  Bookshelf analogy:                                             │
│  The library got a new edition of the book.                     │
│  Your bookshelf still has the old edition.                      │
│  You're reading outdated information and don't know it.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The simplest fix: DELETE the cache key on every write.**

```python
    async def update_task(
        self,
        task_id: int,
        data: TaskUpdate,
    ) -> TaskResponse:
        """Update a task and invalidate its cache."""

        # ── Update in database ──
        task = await self.repo.update(task_id, data)

        if task is None:
            raise TaskNotFoundError(task_id)

        # ── Invalidate cache ──
        await self.cache.delete(task_key(task_id))

        # The NEXT read will miss the cache, hit the DB, 
        # and re-populate the cache with fresh data.
        # That's cache-aside doing its job.

        return TaskResponse.model_validate(task)
```

> "Why DELETE instead of updating the cache with the new value? Because it's simpler and safer. If you try to update the cache, you have to worry about race conditions: what if the database write succeeds but the cache write fails? Now your cache has old data FOREVER (until TTL). With DELETE, the worst case is one extra cache miss — the next read will re-populate."

**But there's a bigger problem. Look at this carefully:**

```python
    async def update_task(self, task_id: int, data: TaskUpdate) -> TaskResponse:
        task = await self.repo.update(task_id, data)

        # We invalidate the single-task cache ✅
        await self.cache.delete(task_key(task_id))

        # But what about the LIST cache?
        # "v1:tasks:org:acme:status:all:page:1:size:20"
        # That list CONTAINS task 42 with the OLD title.
        # We haven't invalidated it. It's still lying. ❌

        return TaskResponse.model_validate(task)
```

**This is where caching gets genuinely hard. We'll solve it properly in Part 4.**

---

# PART 4: CACHE INVALIDATION

## 4.1 The Hardest Problem in Computer Science

> "There are only two hard things in Computer Science: cache invalidation, naming things, and off-by-one errors."

**That's a joke, but the first item is dead serious.**

```
┌─────────────────────────────────────────────────────────────────┐
│                WHY INVALIDATION IS HARD                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  One task update can affect MANY cache keys:                    │
│                                                                 │
│  Task 42 is updated. Which cache keys are now stale?            │
│                                                                 │
│  "v1:tasks:42"                               ← obviously       │
│  "v1:tasks:org:acme:status:all:page:1:size:20"  ← contains 42 │
│  "v1:tasks:org:acme:status:done:page:2:size:20"  ← maybe       │
│  "v1:tasks:org:acme:status:open:page:1:size:20"  ← maybe       │
│  "v1:tasks:org:acme:count"                       ← if status   │
│  "v1:users:7:assigned_tasks"                      │  changed    │
│  "v1:dashboard:org:acme:stats"                    ← definitely  │
│                                                                 │
│  How many of these can you reliably track?                      │
│  What if you miss one?                                          │
│  What if a new feature adds a new list query you forgot?        │
│                                                                 │
│  This is why people say cache invalidation is hard.             │
│  It's not the DELETE command — it's KNOWING WHAT to delete.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Time-Based Invalidation (TTL)

**The blunt instrument. Set a TTL, accept temporary staleness.**

```
┌─────────────────────────────────────────────────────────────────┐
│                TIME-BASED INVALIDATION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HOW IT WORKS:                                                  │
│  Every cached value expires after N seconds.                    │
│  After expiry, the next read triggers a cache miss              │
│  and re-fetches from the database.                              │
│                                                                 │
│  Timeline (TTL = 300 seconds / 5 minutes):                      │
│                                                                 │
│  0s        120s       240s       300s       310s                │
│  │          │          │          │          │                  │
│  Cache SET  │          │          Cache      Cache miss →       │
│  (fresh)    │          │          EXPIRES    re-fetch from DB   │
│             │          │                                        │
│             │          Data updated in DB at 240s               │
│             │          Cache is STALE from 240s-300s            │
│             │          (60 seconds of stale data)               │
│             │                                                   │
│                                                                 │
│  STALENESS WINDOW = Time between DB update and cache expiry     │
│                                                                 │
│  Worst case: data changes 1 second after cache SET              │
│  → stale for almost the entire TTL duration                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Choosing a TTL — this is NOT a technical decision alone:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   TTL GUIDELINES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Data Type              TTL        Reasoning                    │
│  ──────────────────     ─────────  ──────────────────────────   │
│  User profile           5-15 min   Changes rarely              │
│  Task details           2-5 min    Changes during work hours   │
│  Task list (paginated)  30-60 sec  New tasks added frequently  │
│  Dashboard stats        1-5 min    Exact counts not critical   │
│  Org settings           15-60 min  Almost never changes        │
│  Search results         30-60 sec  Stale search is frustrating │
│  External API data      5-60 min   You don't control freshness │
│                                                                 │
│                                                                 │
│  SHORT TTL (30s):                                               │
│  ├─ More cache misses (more DB load)                            │
│  ├─ Fresher data                                                │
│  └─ Use for: rapidly changing data, user-facing lists           │
│                                                                 │
│  LONG TTL (60min):                                              │
│  ├─ Fewer cache misses (less DB load)                           │
│  ├─ More stale data                                             │
│  └─ Use for: rarely changing data, config, reference data       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Centralize TTLs — don't scatter magic numbers:**

```python
# app/core/cache_config.py

from enum import IntEnum

class CacheTTL(IntEnum):
    """Cache TTL values in seconds. One place, all values."""

    TASK_DETAIL = 300       # 5 min — single task
    TASK_LIST = 60          # 1 min — paginated lists change more often
    USER_PROFILE = 900      # 15 min — rarely changes
    ORG_SETTINGS = 3600     # 1 hour — almost never changes
    DASHBOARD_STATS = 120   # 2 min — aggregated, doesn't need to be real-time
    EXTERNAL_API = 600      # 10 min — we don't control the source
```

```python
# Usage in TaskService:
await self.cache.set(key, data, ex=CacheTTL.TASK_DETAIL)
```

---

## 4.3 Event-Based Invalidation (Active Deletion)

**TTL alone means stale data until expiry. Event-based invalidation fixes this: when you KNOW data changed, delete the cache immediately.**

```
┌─────────────────────────────────────────────────────────────────┐
│               EVENT-BASED INVALIDATION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Timeline (Event-based + TTL = 300s):                           │
│                                                                 │
│  0s        120s   121s       240s       300s                    │
│  │          │      │          │          │                      │
│  Cache SET  │  DB updated     │          │                      │
│  (fresh)    │  → DELETE cache │          │                      │
│             │      │          │          │                      │
│             │  122s: Cache miss → fresh data from DB            │
│             │                                                   │
│                                                                 │
│  STALENESS WINDOW: ~1 second (time between DB write and        │
│  cache DELETE). Compare with TTL-only: up to 300 seconds.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Implementing it — the "invalidate on write" pattern:**

```python
# app/services/task_service.py — complete write operations

class TaskService:
    # ... __init__ and read methods from Part 3 ...

    async def _invalidate_task_caches(self, task_id: int, org_id: int) -> None:
        """
        Invalidate ALL cache keys affected by a task change.
        This is the hard part — you must know every key pattern.
        """
        # 1. Always invalidate the single-task key
        await self.cache.delete(task_key(task_id))

        # 2. Invalidate list caches for this org
        #    We can't enumerate every possible page/filter combo,
        #    so we use a pattern-based approach:
        pattern = f"{CACHE_VERSION}:tasks:org:{org_id}:*"
        async for key in self.cache.scan_iter(match=pattern, count=100):
            await self.cache.delete(key)

    async def update_task(
        self, task_id: int, data: TaskUpdate
    ) -> TaskResponse:
        task = await self.repo.update(task_id, data)
        if task is None:
            raise TaskNotFoundError(task_id)

        # Invalidate AFTER successful DB write
        await self._invalidate_task_caches(task_id, task.org_id)

        return TaskResponse.model_validate(task)

    async def delete_task(self, task_id: int, org_id: int) -> None:
        await self.repo.delete(task_id)

        await self._invalidate_task_caches(task_id, org_id)

    async def create_task(
        self, data: TaskCreate, org_id: int
    ) -> TaskResponse:
        task = await self.repo.create(data, org_id)

        # New task → list caches are stale (they don't include it)
        # No single-task cache to delete (it's new)
        pattern = f"{CACHE_VERSION}:tasks:org:{org_id}:*"
        async for key in self.cache.scan_iter(match=pattern, count=100):
            await self.cache.delete(key)

        return TaskResponse.model_validate(task)
```

> "Notice `scan_iter` instead of `KEYS`. You learned `KEYS` in the CLI lecture — it works, but it blocks Redis while scanning ALL keys. `SCAN` does it incrementally, one batch at a time, without blocking. In production with millions of keys, `KEYS *` can freeze your Redis for seconds. Always use `SCAN` in application code."

**The tradeoff of pattern-based invalidation:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SCAN-based invalidation:                                       │
│  ├─ ✅ Catches all list cache keys regardless of filter/page    │
│  ├─ ✅ Works even when new query patterns are added             │
│  ├─ ❌ SCAN has cost — O(n) over all keys with that prefix      │
│  └─ ❌ Can be slow if thousands of keys match                   │
│                                                                 │
│  Explicit key invalidation:                                     │
│  ├─ ✅ Fast — you know exactly which keys to delete             │
│  ├─ ❌ You must maintain a list of every key pattern             │
│  └─ ❌ Add a new query? Forget to add invalidation? Stale data. │
│                                                                 │
│  For most applications: use TTL as your safety net              │
│  (data is never stale for more than X minutes)                  │
│  + event-based deletion for the keys you CAN track.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 The Staleness Trade-off (A Business Decision)

**This is the part most tutorials skip. Caching is not a purely technical decision.**

> "When you cache data, you're making a promise to your users: 'The data you see might be up to X seconds old.' Is that acceptable? That depends on what the data IS."

```
┌─────────────────────────────────────────────────────────────────┐
│              THE STALENESS CONVERSATION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ASK YOUR PRODUCT TEAM (or yourself, for your project):        │
│                                                                 │
│  "If a user updates a task title, and another user sees the     │
│   old title for up to 60 seconds, is that acceptable?"          │
│                                                                 │
│  PROBABLY YES:                                                  │
│  ├─ Dashboard statistics (exact count doesn't matter)           │
│  ├─ Org member list (people don't join/leave every second)      │
│  ├─ Search results (eventual consistency is expected)           │
│  └─ Settings/configuration (almost never changes)               │
│                                                                 │
│  PROBABLY NO:                                                   │
│  ├─ Task assignment (Alice might re-do Bob's work)              │
│  ├─ Remaining inventory (overselling is a real cost)            │
│  ├─ Account balance (showing wrong money = angry users)         │
│  └─ Permissions (security-critical: stale = unauthorized)       │
│                                                                 │
│  THE ANSWER DETERMINES YOUR TTL AND INVALIDATION STRATEGY.      │
│                                                                 │
│  High staleness tolerance → Long TTL, TTL-only invalidation     │
│  Low staleness tolerance  → Short TTL + aggressive event-based  │
│  Zero staleness tolerance → Don't cache it at all.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The combined strategy for your Task Manager:**

```python
# Task detail (single task):
# → TTL 5 min + event-based delete on update/delete
# → Users see fresh data within seconds of a change
# → If event-based invalidation fails, TTL catches it in 5 min

# Task list (paginated):
# → TTL 60 sec + event-based SCAN delete on any task mutation
# → Lists refresh frequently anyway
# → 60 seconds of potential staleness is acceptable

# Dashboard stats (aggregations):
# → TTL 2 min, NO event-based invalidation
# → Stats don't need to be real-time
# → Saves complexity — just let TTL handle it

# User auth/permissions:
# → NO CACHE. Always query the database.
# → Security-critical data must be authoritative.
# → You saw this in Week 9 — get_current_user always hits DB.
```

---

# PART 5: WHEN CACHING BREAKS

## 5.1 Cache Stampede (The Thundering Herd)

**The scenario:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE CACHE STAMPEDE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A popular task is cached. 500 users/minute view it.            │
│                                                                 │
│  TTL expires.                                                   │
│                                                                 │
│  In the SAME millisecond:                                       │
│  Request 1 → cache miss → query PostgreSQL                     │
│  Request 2 → cache miss → query PostgreSQL                     │
│  Request 3 → cache miss → query PostgreSQL                     │
│  ...                                                            │
│  Request 50 → cache miss → query PostgreSQL                    │
│                                                                 │
│  ALL 50 concurrent requests see the miss.                       │
│  ALL 50 query the database.                                     │
│  ALL 50 write to cache (same data, 50 times).                  │
│  Database gets hammered. For ONE expired key.                   │
│                                                                 │
│  Bookshelf analogy:                                             │
│  Your book's "valid until" date passes.                         │
│  50 people in your house ALL drive to the library               │
│  at the same time, all asking for the same book.                │
│  49 of those trips were pointless.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize the damage:**

```
Normal operation (cache hit):              Stampede (cache expired):

  Req→ [Redis: HIT] → Response             Req→ [Redis: MISS] → [PostgreSQL]
  Req→ [Redis: HIT] → Response             Req→ [Redis: MISS] → [PostgreSQL]
  Req→ [Redis: HIT] → Response             Req→ [Redis: MISS] → [PostgreSQL]
  Req→ [Redis: HIT] → Response             Req→ [Redis: MISS] → [PostgreSQL]
  Req→ [Redis: HIT] → Response             Req→ [Redis: MISS] → [PostgreSQL]
                                            ...
  DB Load: 0%                               DB Load: SPIKE 📈💥
```

**Solution: Distributed Locking (one request rebuilds, the rest wait)**

```python
    async def get_task_with_stampede_protection(
        self,
        task_id: int,
    ) -> TaskResponse:
        key = task_key(task_id)
        lock_key = f"lock:{key}"

        # ── Step 1: Check cache (same as before) ──
        cached = await self.cache.get(key)
        if cached is not None:
            return TaskResponse.model_validate_json(cached)

        # ── Step 2: Cache miss — try to acquire rebuild lock ──
        #
        #   SET lock_key "1" NX EX 5
        #   NX = only set if key does NOT exist (you learned this in Lecture 1)
        #   EX = auto-expire the lock after 5 seconds (safety net)
        #
        acquired = await self.cache.set(lock_key, "1", nx=True, ex=5)

        if acquired:
            # I WON the lock. I'm the one who rebuilds the cache.
            try:
                task = await self.repo.get_by_id(task_id)
                if task is None:
                    raise TaskNotFoundError(task_id)

                response = TaskResponse.model_validate(task)
                await self.cache.set(
                    key,
                    response.model_dump_json(),
                    ex=CacheTTL.TASK_DETAIL,
                )
                return response
            finally:
                # Release the lock
                await self.cache.delete(lock_key)
        else:
            # Someone ELSE is rebuilding. Wait briefly, then retry.
            await asyncio.sleep(0.05)  # 50ms
            return await self.get_task_with_stampede_protection(task_id)
```

**How the lock works:**

```
┌─────────────────────────────────────────────────────────────────┐
│              STAMPEDE PREVENTION WITH LOCKING                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request 1: cache miss → SET lock NX → acquired! → query DB    │
│  Request 2: cache miss → SET lock NX → NOT acquired → wait...  │
│  Request 3: cache miss → SET lock NX → NOT acquired → wait...  │
│  Request 4: cache miss → SET lock NX → NOT acquired → wait...  │
│                                                                 │
│  Request 1 finishes: writes to cache, deletes lock              │
│                                                                 │
│  Request 2: retry → cache HIT → return ✅                       │
│  Request 3: retry → cache HIT → return ✅                       │
│  Request 4: retry → cache HIT → return ✅                       │
│                                                                 │
│  DATABASE QUERIES: 1 (not 50)                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The `NX` flag is the key (pun intended). You learned `SETNX` in the CLI lecture. It's an atomic 'set only if not exists' — perfect for a lock. The `EX 5` is a safety net: if Request 1 crashes before deleting the lock, it auto-expires after 5 seconds so the lock doesn't get stuck forever."

**When do you need stampede protection?**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ✅ USE stampede protection when:                                │
│  ├─ The key is very popular (many concurrent readers)           │
│  ├─ The rebuild query is expensive (complex JOINs, aggregates) │
│  └─ A traffic spike during cache miss would hurt the database   │
│                                                                 │
│  ❌ SKIP stampede protection when:                               │
│  ├─ The key is rarely accessed (no stampede possible)           │
│  ├─ The rebuild query is fast (database can handle duplicates)  │
│  └─ You're over-engineering (start simple, add when needed)     │
│                                                                 │
│  Start WITHOUT stampede protection. Add it when you observe     │
│  problems (or when you KNOW a key will be hot).                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Cache Penetration (The Lookup That Never Succeeds)

**What happens when users repeatedly request data that DOESN'T EXIST?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  CACHE PENETRATION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User requests task_id=999999 (doesn't exist in DB)             │
│                                                                 │
│  Request 1: cache miss → query DB → not found → return 404     │
│  Request 2: cache miss → query DB → not found → return 404     │
│  Request 3: cache miss → query DB → not found → return 404     │
│  ...                                                            │
│  Request 1000: cache miss → query DB → not found → return 404  │
│                                                                 │
│  The cache NEVER helps because there's nothing to cache.        │
│  Every request goes to the database.                            │
│                                                                 │
│  Bookshelf analogy:                                             │
│  Someone keeps asking for a book that the library doesn't have. │
│  You drive there every time, find nothing, come home empty.     │
│  If you'd just written "This book doesn't exist" on a sticky   │
│  note, you'd save every trip after the first.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Solution: cache the absence — store a sentinel value for "not found".**

```python
    # A sentinel that means "we checked, it's not in the DB"
    _NOT_FOUND_SENTINEL = "__NOT_FOUND__"
    _NOT_FOUND_TTL = 60  # Cache "not found" for only 60 seconds

    async def get_task(self, task_id: int) -> TaskResponse:
        key = task_key(task_id)
        cached = await self.cache.get(key)

        if cached is not None:
            # Check if it's our "not found" sentinel
            if cached == self._NOT_FOUND_SENTINEL:
                raise TaskNotFoundError(task_id)
            return TaskResponse.model_validate_json(cached)

        # Cache miss — query database
        task = await self.repo.get_by_id(task_id)

        if task is None:
            # Cache the ABSENCE with a short TTL
            await self.cache.set(
                key,
                self._NOT_FOUND_SENTINEL,
                ex=self._NOT_FOUND_TTL,
            )
            raise TaskNotFoundError(task_id)

        response = TaskResponse.model_validate(task)
        await self.cache.set(key, response.model_dump_json(), ex=CacheTTL.TASK_DETAIL)
        return response
```

> "Why a short TTL for the sentinel? Because the task MIGHT be created later. If a user creates task 999999 at minute 2, and we cached 'not found' for 5 minutes, they'll get 404 on their own new task for 3 minutes. 60 seconds is a reasonable compromise."

---

## 5.3 Graceful Degradation (Redis is Down, Now What?)

**Redis WILL go down at some point. Your app must survive.**

```
┌─────────────────────────────────────────────────────────────────┐
│                GRACEFUL DEGRADATION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE RULE:                                                      │
│  Cache is an OPTIMIZATION, not a REQUIREMENT.                   │
│  If Redis is down, your app should work — just slower.          │
│                                                                 │
│  Redis UP:       Request → Redis HIT  → Response (fast)        │
│  Redis UP:       Request → Redis MISS → DB → Redis SET → Resp  │
│  Redis DOWN:     Request → Redis ERROR → DB → Response (slow)  │
│                                    ↑                            │
│                             Catch the error,                    │
│                             skip the cache,                     │
│                             go straight to DB.                  │
│                                                                 │
│  Bookshelf analogy:                                             │
│  If your bookshelf falls over, you can still drive to the       │
│  library. Every single time. It's annoying and slow, but        │
│  you get the data. Your research paper doesn't stop.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Wrap EVERY cache operation in try/except:**

```python
import logging

logger = logging.getLogger(__name__)

class TaskService:
    # ...

    async def get_task(self, task_id: int) -> TaskResponse:
        key = task_key(task_id)

        # ── Try cache (but don't die if Redis is down) ──
        try:
            cached = await self.cache.get(key)
            if cached is not None:
                if cached == self._NOT_FOUND_SENTINEL:
                    raise TaskNotFoundError(task_id)
                return TaskResponse.model_validate_json(cached)
        except redis.RedisError as e:
            # Redis is down or unreachable — log and continue
            logger.warning("Redis unavailable, skipping cache: %s", e)

        # ── Database query (always works, Redis or not) ──
        task = await self.repo.get_by_id(task_id)
        if task is None:
            raise TaskNotFoundError(task_id)

        response = TaskResponse.model_validate(task)

        # ── Try to populate cache (but don't die if Redis is down) ──
        try:
            await self.cache.set(
                key, response.model_dump_json(), ex=CacheTTL.TASK_DETAIL
            )
        except redis.RedisError as e:
            logger.warning("Redis unavailable, skipping cache write: %s", e)

        return response
```

**Ask the class:**

> "Why do we wrap the cache write in a SEPARATE try/except instead of putting everything in one big block?"

Answer: Because the DB query and cache write are INDEPENDENT operations. If the DB query succeeds, we should return the data to the user regardless of whether we managed to cache it. One big try/except would swallow the DB result if only the cache write fails.

---

## 5.4 Monitoring: Is Your Cache Actually Helping?

**A cache with a 5% hit rate is just overhead. You need to know.**

```python
# app/services/cache_metrics.py

class CacheMetrics:
    """Simple in-memory cache metrics. Replace with Prometheus in production."""

    def __init__(self):
        self.hits: int = 0
        self.misses: int = 0

    def record_hit(self) -> None:
        self.hits += 1

    def record_miss(self) -> None:
        self.misses += 1

    @property
    def total(self) -> int:
        return self.hits + self.misses

    @property
    def hit_rate(self) -> float:
        if self.total == 0:
            return 0.0
        return self.hits / self.total

    def summary(self) -> dict:
        return {
            "hits": self.hits,
            "misses": self.misses,
            "total": self.total,
            "hit_rate": f"{self.hit_rate:.1%}",
        }
```

**Using it in your service:**

```python
class TaskService:
    def __init__(self, db: AsyncSession, cache: redis.Redis, metrics: CacheMetrics):
        self.repo = TaskRepository(db)
        self.cache = cache
        self.metrics = metrics

    async def get_task(self, task_id: int) -> TaskResponse:
        key = task_key(task_id)

        try:
            cached = await self.cache.get(key)
            if cached is not None:
                self.metrics.record_hit()   # ← Track hit
                return TaskResponse.model_validate_json(cached)
        except redis.RedisError:
            pass

        self.metrics.record_miss()          # ← Track miss
        # ... rest of cache-aside pattern ...
```

**Expose a health endpoint:**

```python
@router.get("/admin/cache-stats")
async def cache_stats(metrics: CacheMetrics = Depends(get_metrics)):
    return metrics.summary()
    # {"hits": 8432, "misses": 891, "total": 9323, "hit_rate": "90.4%"}
```

**What the numbers tell you:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   INTERPRETING HIT RATE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  > 90%     Excellent. Cache is working well.                    │
│  70-90%    Good. Normal for varied access patterns.             │
│  50-70%    Okay. Review your TTLs and key design.               │
│  < 50%     Problem. You're caching the wrong things, or        │
│            TTLs are too short, or access is too random.         │
│  ~ 0%      Cache is broken or misconfigured. Investigate.      │
│                                                                 │
│  Low hit rate causes:                                           │
│  ├─ TTL too short (data expires before it's requested again)   │
│  ├─ Too many unique keys (each user gets unique query params)   │
│  ├─ Over-invalidation (deleting too much on every write)       │
│  └─ Wrong data cached (caching things nobody re-requests)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│             REDIS + CACHING QUICK REFERENCE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONNECT (async):                                               │
│      import redis.asyncio as redis                              │
│      r = redis.from_url("redis://localhost:6379/0",             │
│                          decode_responses=True)                 │
│                                                                 │
│  BASIC OPERATIONS:                                              │
│      await r.set("key", "value", ex=300)  # SET with 5min TTL  │
│      val = await r.get("key")             # GET (None if miss) │
│      await r.delete("key")                # DELETE              │
│      await r.exists("key")                # Check existence     │
│                                                                 │
│  SERIALIZE (Pydantic):                                          │
│      json_str = model.model_dump_json()            # → Redis   │
│      model = MyModel.model_validate_json(json_str) # ← Redis   │
│                                                                 │
│  CACHE-ASIDE PATTERN:                                           │
│      cached = await r.get(key)                                  │
│      if cached: return deserialize(cached)       # HIT         │
│      data = await db_query()                     # MISS        │
│      await r.set(key, serialize(data), ex=TTL)   # STORE       │
│      return data                                                │
│                                                                 │
│  INVALIDATE ON WRITE:                                           │
│      await db_update(...)                                       │
│      await r.delete(cache_key)     # Delete stale cache        │
│                                                                 │
│  STAMPEDE LOCK:                                                 │
│      acquired = await r.set(lock_key, "1", nx=True, ex=5)      │
│      if acquired: rebuild cache                                │
│      else: wait and retry                                      │
│                                                                 │
│  KEY DESIGN:                                                    │
│      "{version}:{entity}:{id}:{sub}"                           │
│      "v1:tasks:42"                                              │
│      "v1:tasks:org:acme:status:done:page:1:size:20"            │
│                                                                 │
│  GOLDEN RULE:                                                   │
│      ALWAYS wrap cache ops in try/except RedisError.            │
│      Cache is optional. Database is not.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CACHING = TRADING FRESHNESS FOR SPEED                          │
│                                                                 │
│  You keep copies of expensive data in a fast place (Redis),     │
│  knowing those copies might become stale. Every caching         │
│  decision is a bet: "This data won't change in the next         │
│  N seconds, and if it does, I can tolerate the staleness."      │
│                                                                 │
│  ┌───────────┐     ┌───────────┐     ┌───────────┐              │
│  │  Client   │ ──▶ │  Redis    │ ──▶ │ PostgreSQL│              │
│  │ (request) │     │ (cache)   │     │ (truth)   │              │
│  └───────────┘     └───────────┘     └───────────┘              │
│       │                 │                  │                     │
│       │   fast (1ms)    │   slow (50ms)    │                     │
│       │   maybe stale   │   always fresh   │                     │
│       │                 │                  │                     │
│                                                                 │
│  THE BOOKSHELF ANALOGY:                                         │
│  ├─ PostgreSQL = City Library (comprehensive, slow)             │
│  ├─ Redis = Your bookshelf (fast, limited, maybe outdated)      │
│  ├─ Cache-aside = Check shelf first, library as fallback        │
│  ├─ TTL = Sticky note: "re-check library after this date"       │
│  ├─ Invalidation = Remove outdated books when you KNOW          │
│  │                 a new edition is out                          │
│  ├─ Stampede = 50 people all driving to the library at once     │
│  └─ Degradation = Shelf broke? Library still works. You're fine.│
│                                                                 │
│  REMEMBER:                                                      │
│  • Cache is an optimization, NEVER the source of truth          │
│  • Always handle Redis failure gracefully                       │
│  • Invalidation is hard — TTL is your safety net                │
│  • Measure your hit rate — a bad cache is worse than no cache   │
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
│  WEEK 10, LECTURE 3: Session & Token Storage                    │
│  └─ You'll store refresh tokens in Redis (fast lookup + TTL).   │
│     The SET/GET/DELETE + TTL patterns from today are exactly    │
│     what you'll use. Token revocation = cache invalidation.     │
│                                                                 │
│  WEEK 10 PROJECT: Add Caching Layer                             │
│  └─ You'll apply everything from this lecture:                  │
│     cache-aside on expensive queries, TTL strategy,             │
│     invalidation on writes, graceful degradation,               │
│     hit rate monitoring. All of it.                             │
│                                                                 │
│  WEEK 11: Background Jobs & Celery                              │
│  └─ Redis doubles as Celery's message broker.                   │
│     Same Redis server, different role — storing task queues     │
│     instead of cached data.                                     │
│                                                                 │
│  WEEK 12: Performance & Load Testing                            │
│  └─ You'll measure the actual performance impact of your        │
│     cache layer under load. Before/after p95 latencies.         │
│     Your cache metrics from today become your evidence.         │
│                                                                 │
│  WEEK 13-14: Capstone                                           │
│  └─ Your SaaS backend will have Redis caching as a core        │
│     infrastructure component. Everything today scales up        │
│     to a production system.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```