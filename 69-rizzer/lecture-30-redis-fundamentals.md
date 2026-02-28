# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PAIN FIRST, TOOL SECOND                                        │
│  ────────────────────────                                       │
│  Students must SEE why PostgreSQL alone isn't enough.           │
│  Show the latency. Show the redundancy. Then offer the cure.   │
│                                                                 │
│  PYTHON-MAPPED                                                  │
│  ─────────────                                                  │
│  Every Redis data type maps to a Python structure they know.    │
│  dict, list, set — Redis is Python collections on a server.    │
│                                                                 │
│  CLI BEFORE LIBRARY                                             │
│  ──────────────────                                             │
│  Learn Redis through its native CLI first. Same reason we       │
│  wrote raw SQL before SQLAlchemy in Week 5.                     │
│  Understand the tool, then abstract it.                         │
│                                                                 │
│  CONNECT TO ARCHITECTURE                                        │
│  ───────────────────────                                        │
│  Redis doesn't replace PostgreSQL. It sits BESIDE it.           │
│  Every concept ties back to the stack they've already built.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                    REDIS FUNDAMENTALS                           │
│                     (3–3.5 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE SPEED PROBLEM (35 min)                             │
│  ├─ 1.1 The Latency Gap (Demonstration)                         │
│  ├─ 1.2 The Redundancy Problem                                  │
│  ├─ 1.3 The Working Desk Analogy                                │
│  └─ 1.4 Where Redis Fits (Architecture Diagram)                 │
│                                                                 │
│  PART 2: REDIS — THE MENTAL MODEL (35 min)                      │
│  ├─ 2.1 What IS Redis?                                          │
│  ├─ 2.2 Redis with Docker (Familiar Territory)                   │
│  ├─ 2.3 redis-cli — Talking to Redis Directly                   │
│  ├─ 2.4 Keys and Values — The Foundation                        │
│  └─ 2.5 Key Naming Conventions                                  │
│                                                                 │
│  PART 3: STRINGS — MORE THAN TEXT (30 min)                      │
│  ├─ 3.1 Strings Hold Everything                                 │
│  ├─ 3.2 Atomic Counters (INCR, DECR, INCRBY)                   │
│  ├─ 3.3 Batch Operations (MSET, MGET)                           │
│  └─ 3.4 Backend Pattern: Rate Limit Counter                     │
│                                                                 │
│  PART 4: COMPOUND DATA TYPES (55 min)                           │
│  ├─ 4.1 Hashes — Objects in Redis                               │
│  ├─ 4.2 Lists — Ordered Sequences                               │
│  ├─ 4.3 Sets — Unique Collections                               │
│  └─ 4.4 Sorted Sets — Rankings and Priority                     │
│                                                                 │
│  PART 5: TTL — AUTOMATIC EXPIRATION (25 min)                    │
│  ├─ 5.1 Why Data Should Disappear                               │
│  ├─ 5.2 EXPIRE, TTL — Setting the Timer                        │
│  ├─ 5.3 SET with EX — Atomic Create + Expire                   │
│  └─ 5.4 What Redis is NOT (Misconceptions)                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE SPEED PROBLEM

## 1.1 The Latency Gap

**Start with raw numbers. Make the gap visceral.**

```
┌─────────────────────────────────────────────────────────────────┐
│              LATENCY NUMBERS EVERY DEVELOPER SHOULD KNOW        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Operation                          │  Time                     │
│  ───────────────────────────────────│──────────────             │
│  CPU register access                │  < 1 ns                   │
│  L1 cache reference                 │  ~1 ns                    │
│  RAM reference                      │  ~100 ns                  │
│  Read 1 MB from RAM                 │  ~0.25 ms                 │
│  ─── THE MEMORY WALL ──────────────│────────────               │
│  Redis GET (localhost)              │  ~0.1 ms     ◀ HERE       │
│  SSD random read                    │  ~0.1 ms                  │
│  ─── THE DISK/NETWORK WALL ───────│────────────               │
│  PostgreSQL simple query            │  ~1-5 ms     ◀ AND HERE   │
│  PostgreSQL complex JOIN            │  ~10-50 ms                │
│  Network round trip (same region)   │  ~0.5 ms                  │
│  Network round trip (cross-region)  │  ~50-150 ms               │
│                                                                 │
│                                                                 │
│  Redis is ~10-50x faster than PostgreSQL for simple lookups.    │
│  Not because PostgreSQL is bad — because RAM is faster than     │
│  disk + query parsing + planning + execution.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now make it real. Show code they recognize.**

```python
# This is your Task Manager API right now.
# Every single request does this:

@router.get("/dashboard")
async def get_dashboard(
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    start = time.perf_counter()

    # Query 1: Get user's tasks (with JOINs)
    tasks = await task_repo.get_by_user(db, current_user.id)       # ~5ms

    # Query 2: Get categories
    categories = await category_repo.get_all(db)                    # ~3ms

    # Query 3: Get organization settings
    org = await org_repo.get_settings(db, current_user.org_id)     # ~4ms

    elapsed = (time.perf_counter() - start) * 1000
    # elapsed ≈ 12ms per request
    
    return DashboardResponse(tasks=tasks, categories=categories, org=org)
```

**Now ask the class:**

> "12ms per request. That sounds fast. But how often does this endpoint get called?"

---

## 1.2 The Redundancy Problem

**12ms feels fine. Until you zoom out.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE REDUNDANCY PROBLEM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your Task Manager has 200 active users.                        │
│  Each loads their dashboard every 30 seconds.                   │
│                                                                 │
│  Requests per minute:  200 users × 2 loads/min = 400 req/min   │
│  DB queries per minute: 400 × 3 queries = 1,200 queries/min    │
│                                                                 │
│  But here's the thing:                                          │
│  ┌──────────────────────────────────────────────────┐           │
│  │  Categories change once a WEEK.                   │           │
│  │  Org settings change once a MONTH.                │           │
│  │  Most task lists don't change between refreshes.  │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
│  You're running 1,200 queries/min to get the SAME DATA.        │
│                                                                 │
│  That's PostgreSQL doing:                                       │
│  ├─ Parse the SQL           (every time)                        │
│  ├─ Plan the query          (every time)                        │
│  ├─ Execute with I/O        (every time)                        │
│  ├─ Serialize the results   (every time)                        │
│  └─ Send over the network   (every time)                        │
│                                                                 │
│  For data that HASN'T CHANGED.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Visualize the waste:**

```
                PostgreSQL's Day (without cache)
                
  Minute 1:   [query][query][query][query][query] × 240...
  Minute 2:   [query][query][query][query][query] × 240...
  Minute 3:   [query][query][query][query][query] × 240...
     ...
  
  Meanwhile, the actual category data:  ["Work", "Personal", "Urgent"]
  Changed: never.

  PostgreSQL: "You asked me this 800 times already today. 
               Same answer every time. I'm not mad, just tired."
```

> "What if we could ask PostgreSQL ONCE, remember the answer, and serve it from memory the next 799 times?"

That's what Redis does.

---

## 1.3 The Working Desk Analogy

**This analogy will carry us through the lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE WORKING DESK ANALOGY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  ┌──────────────────────┐     ┌──────────────────────┐          │
│  │                      │     │                      │          │
│  │    YOUR DESK         │     │   FILING CABINET     │          │
│  │    (Redis)           │     │   (PostgreSQL)       │          │
│  │                      │     │                      │          │
│  │  • Right in front    │     │  • Down the hall     │          │
│  │    of you            │     │  • Takes a walk      │          │
│  │  • Grab in < 1 sec   │     │  • Find drawer,      │          │
│  │  • Limited space     │     │    find folder...    │          │
│  │  • You clear it      │     │  • Massive capacity  │          │
│  │    at end of day     │     │  • Permanent storage │          │
│  │  • Messy if too full │     │  • Organized & safe  │          │
│  │                      │     │                      │          │
│  └──────────────────────┘     └──────────────────────┘          │
│                                                                 │
│                                                                 │
│  How you actually work:                                         │
│                                                                 │
│  1. First time you need a file → walk to filing cabinet         │
│  2. Bring it back to your desk → keep it handy                  │
│  3. Next time you need it → it's right there on your desk       │
│  4. File gets outdated? → Throw it away, fetch fresh copy       │
│  5. Desk too full? → Clear old stuff you haven't touched        │
│                                                                 │
│  That's EXACTLY how Redis works alongside PostgreSQL.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map it:**

```
Your Office Life           │  Your Backend
───────────────────────────│───────────────────────────
Your desk                  │  Redis (in-memory)
Filing cabinet room        │  PostgreSQL (on disk)
Pulling a file to desk     │  Caching a query result
File on desk expires/stale │  TTL expires, cache invalidated
Grabbing file from desk    │  Redis GET (~0.1ms)
Walking to filing cabinet  │  PostgreSQL query (~5ms)
Desk has limited space     │  Redis is limited by RAM
Filing cabinet is huge     │  PostgreSQL scales to terabytes
```

---

## 1.4 Where Redis Fits

**Redis does NOT replace PostgreSQL. It sits beside it.**

```
┌─────────────────────────────────────────────────────────────────┐
│              YOUR ARCHITECTURE: BEFORE AND AFTER                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE (what you have now):                                    │
│                                                                 │
│  Client ──▶ FastAPI ──▶ PostgreSQL                              │
│                           │                                     │
│                           └─ Every request hits the DB          │
│                                                                 │
│                                                                 │
│  AFTER (what you're building this week):                        │
│                                                                 │
│                         ┌─────────┐                             │
│                    ┌───▶│  Redis  │ (fast, temporary)           │
│                    │    │  (RAM)  │                              │
│                    │    └─────────┘                              │
│                    │       │                                     │
│  Client ──▶ FastAPI┤     miss? ──┐                              │
│                    │             ▼                               │
│                    │    ┌────────────┐                           │
│                    └───▶│ PostgreSQL │ (reliable, permanent)     │
│                         │  (Disk)   │                           │
│                         └────────────┘                           │
│                                                                 │
│  Flow:                                                          │
│  1. Check Redis first (fast path)                               │
│  2. If found → return immediately ("cache hit")                 │
│  3. If not found → query PostgreSQL ("cache miss")              │
│  4. Store result in Redis for next time                         │
│  5. Return to client                                            │
│                                                                 │
│  This is the "cache-aside" pattern. We'll implement it          │
│  in Lecture 2.                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The dashboard endpoint, reimagined:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  WITHOUT REDIS:                                                 │
│  GET /dashboard → PostgreSQL (12ms) → Response                  │
│  GET /dashboard → PostgreSQL (12ms) → Response                  │
│  GET /dashboard → PostgreSQL (12ms) → Response                  │
│  ... 400 times per minute ... every minute ...                  │
│                                                                 │
│                                                                 │
│  WITH REDIS:                                                    │
│  GET /dashboard → Redis HIT (0.2ms) → Response      ← fast!    │
│  GET /dashboard → Redis HIT (0.2ms) → Response      ← fast!    │
│  GET /dashboard → Redis HIT (0.2ms) → Response      ← fast!    │
│  ... 399 cache hits ...                                         │
│  GET /dashboard → Redis MISS → PostgreSQL (12ms)                │
│                   → Store in Redis → Response                   │
│                                                                 │
│  PostgreSQL goes from 1,200 queries/min to maybe 10.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "But caching is just ONE thing Redis does. Redis is a full-blown data structure server. It gives you strings, hashes, lists, sets, sorted sets — all in memory, all blazing fast. That's what we're learning today."

---

# PART 2: REDIS — THE MENTAL MODEL

## 2.1 What IS Redis?

```
┌─────────────────────────────────────────────────────────────────┐
│                      WHAT IS REDIS?                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Redis = Remote Dictionary Server                               │
│                                                                 │
│  "Remote"     → Runs as a separate server process               │
│  "Dictionary" → Stores key-value pairs (like a Python dict!)    │
│  "Server"     → Clients connect over the network                │
│                                                                 │
│                                                                 │
│  Core properties:                                               │
│                                                                 │
│  IN-MEMORY                                                      │
│  ──────────                                                     │
│  All data lives in RAM. That's why it's fast.                   │
│  Your PostgreSQL data lives on disk — fast, but not THIS fast.  │
│                                                                 │
│  KEY-VALUE                                                      │
│  ─────────                                                      │
│  Everything is: key → value.                                    │
│  No tables. No rows. No SQL. No JOINs.                          │
│  Just: "give me the value for this key."                        │
│                                                                 │
│  SINGLE-THREADED                                                │
│  ───────────────                                                │
│  One thread processes ALL commands, one at a time.              │
│  Sound familiar? Same model as Python's event loop (Week 1).   │
│  Fast because: no locks, no context switching, no contention.   │
│  Each command is ATOMIC — no half-finished operations.          │
│                                                                 │
│  DATA STRUCTURES (not just strings)                             │
│  ──────────────────────────────────                             │
│  Values aren't just text. Redis supports:                       │
│  Strings, Hashes, Lists, Sets, Sorted Sets, and more.          │
│  Think of it as Python's collections module on a server.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why single-threaded works:**

```
┌─────────────────────────────────────────────────────────────────┐
│         WHY SINGLE-THREADED ISN'T A PROBLEM                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Wait — single-threaded? That sounds slow!"                    │
│                                                                 │
│  Remember Week 1 — Python's event loop is single-threaded       │
│  too, and it handles thousands of concurrent connections.       │
│                                                                 │
│  Redis is the same idea:                                        │
│  ├─ Commands execute in MICROSECONDS (RAM access)               │
│  ├─ The bottleneck is NETWORK, not computation                  │
│  ├─ Single thread = every command is atomic (no race conditions)│
│  └─ Can handle ~100,000+ operations per second                  │
│                                                                 │
│  Compare:                                                       │
│  ├─ PostgreSQL: needs locks, MVCC, WAL writes for atomicity     │
│  └─ Redis: single thread = atomicity for free                   │
│                                                                 │
│  Remember optimistic vs pessimistic locking from Week 7?        │
│  Redis sidesteps the entire problem. INCR on a counter is       │
│  atomic by design. No "lost update" anomaly. Ever.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Redis with Docker

**You already know Docker from Week 5. Same pattern.**

```yaml
# Add to your existing docker-compose.yml
# (alongside postgres, your api, etc.)

services:
  # ... your existing services ...

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"      # Default Redis port
    volumes:
      - redis_data:/data  # Optional: persist data across restarts
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  # ... your existing volumes ...
  redis_data:
```

```bash
# Start it
docker compose up -d redis

# Verify it's running
docker compose ps
# NAME    IMAGE            STATUS          PORTS
# redis   redis:7-alpine   Up 5 seconds    0.0.0.0:6379->6379/tcp
```

**That's it. Redis is running. No configuration files, no users, no schemas, no migrations.** PostgreSQL needed `CREATE DATABASE`, `CREATE TABLE`, Alembic migrations. Redis needs... nothing. It's ready.

---

## 2.3 redis-cli — Talking to Redis Directly

**Just like `psql` for PostgreSQL, Redis has `redis-cli`.**

```bash
# Connect to Redis running in Docker
docker compose exec redis redis-cli
```

```
127.0.0.1:6379>
```

**You're in. That prompt means:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  127.0.0.1 : 6379 >                                            │
│  ─────────   ────                                               │
│  localhost    default Redis port                                │
│                                                                 │
│  Just like psql connects to PostgreSQL on port 5432,            │
│  redis-cli connects to Redis on port 6379.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**First command — make sure it's alive:**

```
127.0.0.1:6379> PING
PONG
```

**If you see PONG, Redis is alive and listening.**

```
127.0.0.1:6379> PING "are you there?"
"are you there?"
```

---

## 2.4 Keys and Values — The Foundation

**Everything in Redis is a key → value mapping.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE KEY-VALUE MODEL                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PostgreSQL thinks in TABLES:                                   │
│                                                                 │
│  ┌────────────────────────────────────┐                         │
│  │ users table                        │                         │
│  ├──────┬──────────┬─────────────────┤                          │
│  │ id   │ name     │ email           │                          │
│  ├──────┼──────────┼─────────────────┤                          │
│  │ 1    │ Alice    │ alice@mail.com  │                          │
│  │ 2    │ Bob      │ bob@mail.com    │                          │
│  └──────┴──────────┴─────────────────┘                          │
│                                                                 │
│  Redis thinks in KEYS:                                          │
│                                                                 │
│  ┌──────────────────────────┬─────────────────┐                 │
│  │ KEY                      │ VALUE            │                │
│  ├──────────────────────────┼─────────────────┤                 │
│  │ "user:1:name"            │ "Alice"          │                │
│  │ "user:1:email"           │ "alice@mail.com" │                │
│  │ "user:2:name"            │ "Bob"            │                │
│  │ "user:2:email"           │ "bob@mail.com"   │                │
│  └──────────────────────────┴─────────────────┘                 │
│                                                                 │
│  Or, using a Hash (we'll learn this in Part 4):                 │
│                                                                 │
│  ┌──────────────┬──────────────────────────────┐                │
│  │ KEY          │ VALUE (Hash)                  │               │
│  ├──────────────┼──────────────────────────────┤                │
│  │ "user:1"     │ {name: "Alice",               │               │
│  │              │  email: "alice@mail.com"}      │               │
│  │ "user:2"     │ {name: "Bob",                  │               │
│  │              │  email: "bob@mail.com"}         │               │
│  └──────────────┴──────────────────────────────┘                │
│                                                                 │
│                                                                 │
│  KEY = always a string (the "name" you look up)                 │
│  VALUE = one of several data types (string, hash, list, etc.)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The simplest operation in Redis:**

```
127.0.0.1:6379> SET greeting "hello world"
OK

127.0.0.1:6379> GET greeting
"hello world"

127.0.0.1:6379> DEL greeting
(integer) 1

127.0.0.1:6379> GET greeting
(nil)
```

**Compare to Python:**

```python
# What you just did in Redis is equivalent to:

store = {}                          # Redis server (in-memory dict)

store["greeting"] = "hello world"   # SET greeting "hello world"

store["greeting"]                   # GET greeting → "hello world"

del store["greeting"]               # DEL greeting

store.get("greeting")               # GET greeting → (nil) / None
```

**That's the entire mental model.** Redis is a giant Python dictionary running on a server, shared between all your application instances, with superpowers (data types, expiration, atomic operations).

**A few more essential key operations:**

```
127.0.0.1:6379> SET name "Alice"
OK

127.0.0.1:6379> SET age "30"
OK

127.0.0.1:6379> EXISTS name
(integer) 1

127.0.0.1:6379> EXISTS nonexistent
(integer) 0

127.0.0.1:6379> TYPE name
string

127.0.0.1:6379> DBSIZE
(integer) 2
```

**Listing keys — and a WARNING:**

```
127.0.0.1:6379> KEYS *
1) "name"
2) "age"

127.0.0.1:6379> KEYS na*
1) "name"
```

```
┌─────────────────────────────────────────────────────────────────┐
│  ⚠️  WARNING: KEYS IS DANGEROUS IN PRODUCTION                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  KEYS * scans EVERY key in the database. O(N).                  │
│                                                                 │
│  With 10 million keys, this BLOCKS Redis for seconds.           │
│  Remember: single-threaded. While KEYS is scanning,             │
│  EVERY other client is frozen. Waiting. Dead.                   │
│                                                                 │
│  ✅ Development: KEYS is fine. You have 20 keys.                │
│  ❌ Production: Use SCAN instead (cursor-based iteration).      │
│                                                                 │
│  Sound familiar? Same reason we use cursor pagination           │
│  instead of loading all rows from PostgreSQL (Week 4).          │
│                                                                 │
│  SCAN 0 MATCH "user:*" COUNT 100                                │
│  → Returns a cursor + a batch of matching keys                  │
│  → Call again with the returned cursor for the next batch       │
│  → Non-blocking: other commands execute between batches         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 Key Naming Conventions

**Redis has no tables, no schemas. Your key names ARE your schema.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 KEY NAMING CONVENTIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE COLONS AS SEPARATORS (industry standard):                  │
│                                                                 │
│  Pattern:   entity:id:field                                     │
│                                                                 │
│  ✅ Good key names:                                              │
│  ├─ "user:123"                  (a user object)                 │
│  ├─ "user:123:tasks"            (that user's task list)         │
│  ├─ "org:456:settings"          (org settings)                  │
│  ├─ "cache:dashboard:user:123"  (cached dashboard for user)    │
│  ├─ "session:abc123def"         (a session)                     │
│  ├─ "ratelimit:login:192.168.1.1" (rate limit counter)         │
│  └─ "queue:email:notifications" (a job queue)                   │
│                                                                 │
│  ❌ Bad key names:                                               │
│  ├─ "u123"          (too cryptic — what is this?)               │
│  ├─ "mydata"        (meaningless — what data?)                  │
│  ├─ "user_123_tasks" (underscores work, but colons are         │
│  │                    the Redis convention — be consistent)      │
│  └─ "a really long key name with spaces and no structure"       │
│       (wastes memory, hard to scan, no pattern matching)        │
│                                                                 │
│                                                                 │
│  WHY THIS MATTERS:                                              │
│  ├─ SCAN can match patterns: SCAN 0 MATCH "user:123:*"         │
│  ├─ Humans can read your data: you'll use redis-cli to debug    │
│  └─ Namespacing prevents collisions between features            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Think of colons as folders:**

```
cache:
├── dashboard:
│   ├── user:123     → "{tasks: [...], categories: [...]}"
│   └── user:456     → "{tasks: [...], categories: [...]}"
├── categories:
│   └── org:789      → "[{id: 1, name: 'Work'}, ...]"
└── task:
    ├── 101          → "{id: 101, title: 'Fix bug', ...}"
    └── 102          → "{id: 102, title: 'Add feature', ...}"

session:
├── abc123def        → {user_id: 1, role: "admin"}
└── xyz789ghi        → {user_id: 2, role: "member"}

ratelimit:
└── login:
    ├── 192.168.1.1  → 3  (attempts this minute)
    └── 10.0.0.55    → 1  (attempts this minute)
```

> "Redis doesn't actually have folders. But consistent colon-delimited naming gives you the same mental organization. You'll thank yourself when you're debugging at 2 AM."

---

# PART 3: STRINGS — MORE THAN TEXT

## 3.1 Strings Hold Everything

**Redis "strings" are misleading. They hold text, numbers, even serialized JSON.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  REDIS STRINGS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python equivalent:  variable = value                           │
│                                                                 │
│  A Redis string is a single value attached to a key.            │
│  Despite the name "string," it can hold:                        │
│                                                                 │
│  ├─ Text:       "hello world"                                   │
│  ├─ Numbers:    "42" (stored as string, treated as integer)     │
│  ├─ JSON:       '{"id":1,"name":"Alice"}'                       │
│  ├─ Binary:     raw bytes (images, serialized data)             │
│  └─ Anything up to 512 MB per value                             │
│                                                                 │
│  It's the simplest and most common Redis type.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Basic string operations:**

```
# Store a simple value
127.0.0.1:6379> SET user:1:name "Alice"
OK

127.0.0.1:6379> GET user:1:name
"Alice"


# Store a number (still stored as a string internally)
127.0.0.1:6379> SET user:1:login_count "0"
OK

127.0.0.1:6379> GET user:1:login_count
"0"


# Store JSON (a serialized Pydantic model, perhaps?)
127.0.0.1:6379> SET cache:task:101 '{"id":101,"title":"Fix bug","status":"open"}'
OK

127.0.0.1:6379> GET cache:task:101
"{\"id\":101,\"title\":\"Fix bug\",\"status\":\"open\"}"


# Check string length
127.0.0.1:6379> STRLEN user:1:name
(integer) 5

127.0.0.1:6379> STRLEN cache:task:101
(integer) 45
```

**SET has options you should know:**

```
# NX — Only set if key does NOT exist (create, don't overwrite)
127.0.0.1:6379> SET lock:task:101 "worker-1" NX
OK                                               ← Set (key was new)

127.0.0.1:6379> SET lock:task:101 "worker-2" NX
(nil)                                            ← NOT set (key exists)

127.0.0.1:6379> GET lock:task:101
"worker-1"                                       ← First writer wins


# XX — Only set if key DOES exist (update, don't create)
127.0.0.1:6379> SET user:999:name "Ghost" XX
(nil)                                            ← NOT set (key doesn't exist)

127.0.0.1:6379> SET user:1:name "Alice Smith" XX
OK                                               ← Updated (key existed)
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SET key value NX  →  "Create if missing" (like INSERT)         │
│  SET key value XX  →  "Update if exists"  (like UPDATE)         │
│  SET key value     →  "Overwrite always"  (like UPSERT)         │
│                                                                 │
│  NX is critical for DISTRIBUTED LOCKS.                          │
│  "Acquire the lock only if nobody else has it."                 │
│  The atomic nature of SET NX makes this safe —                  │
│  two workers racing will never BOTH succeed.                    │
│  (Single-threaded, remember?)                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Atomic Counters (INCR, DECR, INCRBY)

**This is where "strings hold numbers" becomes powerful.**

```
127.0.0.1:6379> SET pageviews:homepage 0
OK

127.0.0.1:6379> INCR pageviews:homepage
(integer) 1

127.0.0.1:6379> INCR pageviews:homepage
(integer) 2

127.0.0.1:6379> INCR pageviews:homepage
(integer) 3

127.0.0.1:6379> GET pageviews:homepage
"3"
```

**INCR is ATOMIC. This matters.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHY ATOMIC COUNTERS MATTER                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Imagine 50 users hit your homepage simultaneously.             │
│                                                                 │
│  WITHOUT atomic counter (e.g., in PostgreSQL):                  │
│                                                                 │
│  Worker A: reads count = 10                                     │
│  Worker B: reads count = 10          ← Same value!              │
│  Worker A: writes count = 11                                    │
│  Worker B: writes count = 11        ← LOST UPDATE!              │
│                                                                 │
│  You lost a page view. This is the exact race condition         │
│  we discussed with optimistic locking in Week 7.                │
│                                                                 │
│                                                                 │
│  WITH Redis INCR:                                               │
│                                                                 │
│  Worker A: INCR pageviews → 11      (atomic read + increment)  │
│  Worker B: INCR pageviews → 12      (atomic read + increment)  │
│                                                                 │
│  No race condition. No locks. No retries.                       │
│  Single-threaded execution guarantees correctness.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The full counter toolkit:**

```
127.0.0.1:6379> SET score 10
OK

127.0.0.1:6379> INCR score            # +1
(integer) 11

127.0.0.1:6379> INCRBY score 5        # +5
(integer) 16

127.0.0.1:6379> DECR score            # -1
(integer) 15

127.0.0.1:6379> DECRBY score 3        # -3
(integer) 12

127.0.0.1:6379> INCRBYFLOAT score 2.5 # +2.5 (float increment)
"14.5"
```

**What happens if the key doesn't exist?**

```
127.0.0.1:6379> EXISTS brand_new_counter
(integer) 0                            ← Key doesn't exist

127.0.0.1:6379> INCR brand_new_counter
(integer) 1                            ← Redis creates it at 0, then increments

127.0.0.1:6379> INCR brand_new_counter
(integer) 2
```

> "You don't need to initialize counters. INCR on a non-existent key treats it as 0 and increments to 1. One less thing to worry about."

**What happens if the value isn't a number?**

```
127.0.0.1:6379> SET name "Alice"
OK

127.0.0.1:6379> INCR name
(error) ERR value is not an integer or out of range
```

Redis returns clear errors. No silent corruption.

---

## 3.3 Batch Operations (MSET, MGET)

**What if you need to set or get multiple keys at once?**

```
# Set multiple keys in one command
127.0.0.1:6379> MSET user:1:name "Alice" user:1:role "admin" user:1:org "42"
OK

# Get multiple keys in one command
127.0.0.1:6379> MGET user:1:name user:1:role user:1:org
1) "Alice"
2) "admin"
3) "42"

# If a key doesn't exist, you get nil for that position
127.0.0.1:6379> MGET user:1:name user:1:phone user:1:role
1) "Alice"
2) (nil)
3) "admin"
```

**Why batch matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 NETWORK ROUND TRIPS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INDIVIDUAL COMMANDS (3 round trips):                           │
│                                                                 │
│  App ──GET user:1:name──▶ Redis ──"Alice"──▶ App     ~0.1ms    │
│  App ──GET user:1:role──▶ Redis ──"admin"──▶ App     ~0.1ms    │
│  App ──GET user:1:org───▶ Redis ──"42"────▶ App      ~0.1ms    │
│                                              Total:  ~0.3ms    │
│                                                                 │
│  BATCH COMMAND (1 round trip):                                  │
│                                                                 │
│  App ──MGET name role org──▶ Redis ──all 3──▶ App    ~0.1ms    │
│                                              Total:  ~0.1ms    │
│                                                                 │
│  3x faster. And the gap grows with more keys.                   │
│  With 100 keys: 100 round trips vs 1 round trip.               │
│                                                                 │
│  Redis itself is fast. The NETWORK is often the bottleneck.     │
│  Batch operations minimize network round trips.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Backend Pattern: Rate Limit Counter

**You learned about rate limiting from the client side in Week 8, and you discussed rate limiting login endpoints in Week 9. Here's how Redis makes it trivial to implement.**

```
┌─────────────────────────────────────────────────────────────────┐
│        PATTERN: RATE LIMIT WITH STRING COUNTER                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Problem: Limit each IP to 10 login attempts per minute.        │
│                                                                 │
│  Key design:  ratelimit:login:{ip}:{minute_timestamp}           │
│                                                                 │
│  For IP 192.168.1.50 at minute 1707000060:                      │
│                                                                 │
│  1st attempt:                                                   │
│     INCR ratelimit:login:192.168.1.50:1707000060  → 1           │
│     EXPIRE ratelimit:login:192.168.1.50:1707000060 60           │
│                                                                 │
│  2nd attempt:                                                   │
│     INCR ratelimit:login:192.168.1.50:1707000060  → 2           │
│                                                                 │
│  ... 10th attempt → 10 (at the limit)                           │
│                                                                 │
│  11th attempt:                                                  │
│     INCR → 11                                                   │
│     11 > 10 → REJECT with 429 Too Many Requests                │
│                                                                 │
│  After 60 seconds: key expires automatically. Counter resets.   │
│                                                                 │
│  No cron job. No cleanup code. Redis TTL handles it.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "We'll implement this exact pattern in Python in Lecture 2. For now, understand the building blocks: INCR gives you an atomic counter, EXPIRE makes it self-cleaning. That's rate limiting in two commands."

---

# PART 4: COMPOUND DATA TYPES

**Before we dive in, here's the map:**

```
┌─────────────────────────────────────────────────────────────────┐
│              REDIS DATA TYPES → PYTHON EQUIVALENTS              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Redis Type     Python Equivalent     Best For                  │
│  ──────────     ──────────────────    ─────────────────         │
│  String         str / int variable    Simple values, counters   │
│  Hash           dict[str, str]        Objects with fields       │
│  List           collections.deque     Queues, recent items      │
│  Set            set                   Unique items, membership  │
│  Sorted Set     (no direct equiv.)    Rankings, priority queues │
│                                                                 │
│  The key insight: you already KNOW these data structures.       │
│  Redis just puts them on a server with superpowers:             │
│  shared access, persistence, expiration, atomicity.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.1 Hashes — Objects in Redis

**A Hash is a key that holds multiple field-value pairs. Like a Python dict, but flat (one level deep, no nesting).**

```
┌─────────────────────────────────────────────────────────────────┐
│                     REDIS HASHES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python equivalent:   dict[str, str]                            │
│                                                                 │
│  String:      key → single value                                │
│  Hash:        key → { field1: value1, field2: value2, ... }     │
│                                                                 │
│                                                                 │
│  Think of a hash as ONE ROW from a PostgreSQL table:            │
│                                                                 │
│  PostgreSQL row:                                                │
│  ┌──────┬──────────┬─────────────────┬───────┐                  │
│  │ id   │ name     │ email           │ role  │                  │
│  ├──────┼──────────┼─────────────────┼───────┤                  │
│  │ 1    │ Alice    │ alice@mail.com  │ admin │                  │
│  └──────┴──────────┴─────────────────┴───────┘                  │
│                                                                 │
│  Redis hash:                                                    │
│  Key: "user:1"                                                  │
│  ┌──────────┬─────────────────┐                                 │
│  │ field    │ value            │                                │
│  ├──────────┼─────────────────┤                                 │
│  │ name     │ Alice            │                                │
│  │ email    │ alice@mail.com   │                                │
│  │ role     │ admin            │                                │
│  └──────────┴─────────────────┘                                 │
│                                                                 │
│  Same data. But in Redis, it's in RAM, and you can              │
│  read/write individual fields without fetching the whole thing. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Core hash commands:**

```
# HSET — set one or more fields
127.0.0.1:6379> HSET user:1 name "Alice" email "alice@mail.com" role "admin"
(integer) 3                    ← 3 fields created

# HGET — get one field
127.0.0.1:6379> HGET user:1 name
"Alice"

127.0.0.1:6379> HGET user:1 email
"alice@mail.com"

# HGETALL — get ALL fields and values
127.0.0.1:6379> HGETALL user:1
1) "name"
2) "Alice"
3) "email"
4) "alice@mail.com"
5) "role"
6) "admin"

# Results alternate: field, value, field, value...
```

**More hash operations:**

```
# Check if a field exists
127.0.0.1:6379> HEXISTS user:1 name
(integer) 1                    ← Yes

127.0.0.1:6379> HEXISTS user:1 phone
(integer) 0                    ← No

# Delete a field
127.0.0.1:6379> HDEL user:1 email
(integer) 1

127.0.0.1:6379> HGETALL user:1
1) "name"
2) "Alice"
3) "role"
4) "admin"

# Get just the field names or just the values
127.0.0.1:6379> HKEYS user:1
1) "name"
2) "role"

127.0.0.1:6379> HVALS user:1
1) "Alice"
2) "admin"

# Count fields
127.0.0.1:6379> HLEN user:1
(integer) 2
```

**Counters inside hashes — HINCRBY:**

```
127.0.0.1:6379> HSET user:1 login_count 0 tasks_completed 0
(integer) 2

127.0.0.1:6379> HINCRBY user:1 login_count 1
(integer) 1

127.0.0.1:6379> HINCRBY user:1 tasks_completed 5
(integer) 5

127.0.0.1:6379> HGETALL user:1
1) "name"
2) "Alice"
3) "role"
4) "admin"
5) "login_count"
6) "1"
7) "tasks_completed"
8) "5"
```

> "Atomic increment on a single field inside a hash. No need to fetch the whole object, update in Python, and write it back. One command."

**When to use a Hash vs multiple Strings:**

```
┌─────────────────────────────────────────────────────────────────┐
│           HASH VS MULTIPLE STRINGS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MULTIPLE STRINGS:                                              │
│  ├─ SET user:1:name "Alice"                                     │
│  ├─ SET user:1:email "alice@mail.com"                           │
│  └─ SET user:1:role "admin"                                     │
│  → 3 keys. Each has its own TTL. Fetching all = 3 GETs.        │
│                                                                 │
│  ONE HASH:                                                      │
│  └─ HSET user:1 name "Alice" email "alice@mail.com" role "admin"│
│  → 1 key. One TTL for the whole object. Fetch all = 1 HGETALL. │
│                                                                 │
│  USE HASH WHEN:                                                 │
│  ├─ Data represents a single entity (user, task, session)       │
│  ├─ Fields are read/written together                            │
│  ├─ You want one TTL for the whole object                       │
│  └─ You need to update individual fields independently          │
│                                                                 │
│  USE STRINGS WHEN:                                              │
│  ├─ Each value needs its own independent TTL                    │
│  ├─ Values are unrelated (not fields of an object)              │
│  └─ You only ever read one value at a time                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to your stack:** Think of a Pydantic model's `.model_dump()` output — a flat dictionary of field names to values. That's exactly what a Redis hash stores. In Lecture 2, you'll serialize Pydantic models into Redis hashes and back.

---

## 4.2 Lists — Ordered Sequences

**A Redis List is an ordered collection of strings, optimized for pushing and popping at both ends.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      REDIS LISTS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python equivalent:   collections.deque (NOT list)              │
│                                                                 │
│  Why deque, not list?                                           │
│  ├─ LPUSH/RPUSH (add to front/back): O(1)    ← like deque     │
│  ├─ LPOP/RPOP (remove from front/back): O(1) ← like deque     │
│  ├─ Access by index: O(N)                     ← unlike list     │
│  └─ Optimized for the ENDS, not the middle                      │
│                                                                 │
│  Visualize a Redis list:                                        │
│                                                                 │
│  LPUSH (left/front)                    RPUSH (right/back)       │
│       │                                      │                  │
│       ▼                                      ▼                  │
│      ┌─────┬─────┬─────┬─────┬─────┬─────┐                     │
│      │  e  │  d  │  c  │  b  │  a  │  f  │                     │
│      └─────┴─────┴─────┴─────┴─────┴─────┘                     │
│       ▲                                      ▲                  │
│       │                                      │                  │
│  LPOP (left/front)                     RPOP (right/back)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Core list commands:**

```
# RPUSH — add to the RIGHT (end/tail)
127.0.0.1:6379> RPUSH activity:user:1 "logged in"
(integer) 1                                         ← list length

127.0.0.1:6379> RPUSH activity:user:1 "created task 101"
(integer) 2

127.0.0.1:6379> RPUSH activity:user:1 "completed task 99"
(integer) 3

# LRANGE — read a range of elements (0-based index)
127.0.0.1:6379> LRANGE activity:user:1 0 -1        ← 0 to end (all)
1) "logged in"
2) "created task 101"
3) "completed task 99"

127.0.0.1:6379> LRANGE activity:user:1 0 1          ← first 2
1) "logged in"
2) "created task 101"
```

**LPUSH — add to the LEFT (front/head):**

```
# LPUSH — add to the LEFT (front)
127.0.0.1:6379> LPUSH recent:notifications:user:1 "You were mentioned in Task 101"
(integer) 1

127.0.0.1:6379> LPUSH recent:notifications:user:1 "Task 99 was completed"
(integer) 2

127.0.0.1:6379> LPUSH recent:notifications:user:1 "New comment on Task 55"
(integer) 3

# Newest first! LPUSH puts each item at the FRONT.
127.0.0.1:6379> LRANGE recent:notifications:user:1 0 -1
1) "New comment on Task 55"            ← most recent
2) "Task 99 was completed"
3) "You were mentioned in Task 101"    ← oldest
```

**LPUSH + LRANGE 0 N = "most recent N items" — very common pattern.**

**Popping elements:**

```
# LPOP — remove and return from the LEFT (front)
127.0.0.1:6379> LPOP recent:notifications:user:1
"New comment on Task 55"

# RPOP — remove and return from the RIGHT (end)
127.0.0.1:6379> RPOP recent:notifications:user:1
"You were mentioned in Task 101"

# What's left?
127.0.0.1:6379> LRANGE recent:notifications:user:1 0 -1
1) "Task 99 was completed"

# List length
127.0.0.1:6379> LLEN recent:notifications:user:1
(integer) 1
```

**The "capped list" pattern — keep only the last N items:**

```
# LTRIM — trim a list to a specified range
# Keep only the 100 most recent notifications:

127.0.0.1:6379> LPUSH recent:notifications:user:1 "new event"
127.0.0.1:6379> LTRIM recent:notifications:user:1 0 99
OK
# List is now at most 100 items. Oldest items are discarded.
```

```
┌─────────────────────────────────────────────────────────────────┐
│              LIST PATTERN: CAPPED ACTIVITY FEED                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every time something happens:                                  │
│     LPUSH activity:user:{id} "{event JSON}"                     │
│     LTRIM activity:user:{id} 0 99                               │
│                                                                 │
│  To display the feed:                                           │
│     LRANGE activity:user:{id} 0 19        ← latest 20 events   │
│                                                                 │
│  Why this is powerful:                                          │
│  ├─ Always bounded (never grows past 100)                       │
│  ├─ Always sorted by time (newest first via LPUSH)              │
│  ├─ O(1) writes (LPUSH + LTRIM on a capped list)               │
│  └─ No need for ORDER BY created_at DESC LIMIT 20              │
│                                                                 │
│  In PostgreSQL, this would be:                                  │
│     SELECT * FROM activity                                      │
│     WHERE user_id = 1                                           │
│     ORDER BY created_at DESC                                    │
│     LIMIT 20;                                                   │
│                                                                 │
│  That works, but Redis serves it from memory in ~0.1ms.         │
│  The PostgreSQL query might take 5-15ms with indexes.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Lists as queues — a preview of next week:**

```
┌─────────────────────────────────────────────────────────────────┐
│              LIST PATTERN: SIMPLE JOB QUEUE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Producer (your API):                                           │
│     RPUSH queue:emails '{"to":"bob@mail.com","type":"welcome"}' │
│     RPUSH queue:emails '{"to":"alice@mail.com","type":"reset"}' │
│                                                                 │
│  Consumer (a worker process):                                   │
│     LPOP queue:emails  → gets the oldest job first              │
│     process it...                                               │
│     LPOP queue:emails  → gets the next one                      │
│                                                                 │
│  RPUSH + LPOP = FIFO queue (First In, First Out)                │
│                                                                 │
│       RPUSH ──▶ [ job1 | job2 | job3 ] ──▶ LPOP                │
│       (enqueue)                          (dequeue)              │
│                                                                 │
│  This is exactly what Celery uses Redis for under the hood.     │
│  Next week (Week 11), you'll see the full picture.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Sets — Unique Collections

**A Redis Set is an unordered collection of unique strings.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      REDIS SETS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python equivalent:   set()                                     │
│                                                                 │
│  Properties:                                                    │
│  ├─ Unordered (no index, no position)                           │
│  ├─ Unique (duplicates silently ignored)                        │
│  ├─ O(1) add, remove, and membership check                     │
│  └─ Supports set operations: union, intersection, difference    │
│                                                                 │
│  If you know Python sets, you know Redis sets.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Core set commands:**

```
# SADD — add members to a set
127.0.0.1:6379> SADD online:users "user:1" "user:2" "user:3"
(integer) 3                                    ← 3 added

# Try adding a duplicate
127.0.0.1:6379> SADD online:users "user:1"
(integer) 0                                    ← 0 added (already exists)

# SMEMBERS — get all members
127.0.0.1:6379> SMEMBERS online:users
1) "user:1"
2) "user:2"
3) "user:3"

# SISMEMBER — check membership (O(1), very fast)
127.0.0.1:6379> SISMEMBER online:users "user:2"
(integer) 1                                    ← Yes, present

127.0.0.1:6379> SISMEMBER online:users "user:99"
(integer) 0                                    ← No

# SCARD — count members ("set cardinality")
127.0.0.1:6379> SCARD online:users
(integer) 3

# SREM — remove a member
127.0.0.1:6379> SREM online:users "user:2"
(integer) 1

127.0.0.1:6379> SMEMBERS online:users
1) "user:1"
2) "user:3"
```

**Set operations — this is where Sets become powerful:**

```
# Setup: users in two different project teams
127.0.0.1:6379> SADD project:alpha:members "alice" "bob" "carol"
(integer) 3

127.0.0.1:6379> SADD project:beta:members "bob" "dave" "carol"
(integer) 3


# SINTER — who is on BOTH teams? (intersection)
127.0.0.1:6379> SINTER project:alpha:members project:beta:members
1) "bob"
2) "carol"


# SUNION — everyone across both teams? (union)
127.0.0.1:6379> SUNION project:alpha:members project:beta:members
1) "alice"
2) "bob"
3) "carol"
4) "dave"


# SDIFF — who is on alpha but NOT beta? (difference)
127.0.0.1:6379> SDIFF project:alpha:members project:beta:members
1) "alice"
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    SET OPERATIONS VISUAL                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  project:alpha          project:beta                            │
│  ┌─────────────┐        ┌─────────────┐                         │
│  │             │        │             │                         │
│  │  alice   ┌──┼────────┼──┐  dave    │                         │
│  │          │  bob      │  │          │                         │
│  │          │  carol    │  │          │                         │
│  │          └──┼────────┼──┘          │                         │
│  │             │        │             │                         │
│  └─────────────┘        └─────────────┘                         │
│                                                                 │
│  SINTER:  { bob, carol }       (the overlap)                    │
│  SUNION:  { alice, bob, carol, dave }  (everyone)               │
│  SDIFF (alpha - beta):  { alice }  (only in alpha)              │
│                                                                 │
│  In PostgreSQL, this would be JOINs or subqueries.              │
│  In Redis, it's one command and microseconds.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Backend pattern: Unique visitors per day**

```
┌─────────────────────────────────────────────────────────────────┐
│            PATTERN: UNIQUE DAILY VISITORS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every time a user visits:                                      │
│     SADD visitors:2026-02-14 "user:123"                         │
│     SADD visitors:2026-02-14 "user:456"                         │
│     SADD visitors:2026-02-14 "user:123"  ← duplicate ignored   │
│                                                                 │
│  How many unique visitors today?                                │
│     SCARD visitors:2026-02-14  → 2                              │
│                                                                 │
│  Was user:123 active today?                                     │
│     SISMEMBER visitors:2026-02-14 "user:123"  → 1 (yes)        │
│                                                                 │
│  Set a TTL so yesterday's data auto-cleans:                     │
│     EXPIRE visitors:2026-02-14 86400   (24 hours)               │
│                                                                 │
│  Compare to PostgreSQL:                                         │
│     SELECT COUNT(DISTINCT user_id)                              │
│     FROM page_views                                             │
│     WHERE visited_at >= '2026-02-14'                            │
│     AND visited_at < '2026-02-15';                              │
│                                                                 │
│  PostgreSQL scans rows and deduplicates.                        │
│  Redis already deduplicated on insert. SCARD is O(1).           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Sorted Sets — Rankings and Priority

**Sorted Sets are Redis's most unique and powerful data type. There is no direct Python equivalent.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   REDIS SORTED SETS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python equivalent:   None (closest is a dict sorted by value)  │
│                                                                 │
│  A Sorted Set is a Set PLUS a numeric score for each member.    │
│  Members are automatically sorted by their score. Always.       │
│                                                                 │
│  ┌──────────┬──────────┐                                        │
│  │  SCORE   │  MEMBER  │                                        │
│  ├──────────┼──────────┤                                        │
│  │  100     │  alice   │  ← highest score                       │
│  │  85      │  carol   │                                        │
│  │  72      │  bob     │                                        │
│  │  45      │  dave    │  ← lowest score                        │
│  └──────────┴──────────┘                                        │
│                                                                 │
│  Properties:                                                    │
│  ├─ Unique members (like a Set — no duplicates)                 │
│  ├─ Each member has a numeric score (float)                     │
│  ├─ ALWAYS sorted by score (Redis maintains order for you)      │
│  ├─ O(log N) add and remove                                     │
│  ├─ O(log N + M) range queries (M = number of results)          │
│  └─ Get rank of any member in O(log N)                          │
│                                                                 │
│  Use cases: leaderboards, priority queues, time-series indexes, │
│  "top N" lists, scheduling by timestamp.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Core sorted set commands:**

```
# ZADD — add members with scores
127.0.0.1:6379> ZADD leaderboard 100 "alice" 85 "carol" 72 "bob" 45 "dave"
(integer) 4

# ZRANGE — get members by rank (low to high score, 0-based)
127.0.0.1:6379> ZRANGE leaderboard 0 -1 WITHSCORES
1) "dave"
2) "45"
3) "bob"
4) "72"
5) "carol"
6) "85"
7) "alice"
8) "100"

# ZREVRANGE — get members by rank (HIGH to low)
127.0.0.1:6379> ZREVRANGE leaderboard 0 -1 WITHSCORES
1) "alice"
2) "100"
3) "carol"
4) "85"
5) "bob"
6) "72"
7) "dave"
8) "45"

# Top 3 players:
127.0.0.1:6379> ZREVRANGE leaderboard 0 2 WITHSCORES
1) "alice"
2) "100"
3) "carol"
4) "85"
5) "bob"
6) "72"
```

**Individual member operations:**

```
# ZSCORE — get a member's score
127.0.0.1:6379> ZSCORE leaderboard "bob"
"72"

# ZRANK — get a member's rank (0 = lowest score)
127.0.0.1:6379> ZRANK leaderboard "bob"
(integer) 1                                     ← 2nd from the bottom

# ZREVRANK — rank from the TOP (0 = highest score)
127.0.0.1:6379> ZREVRANK leaderboard "alice"
(integer) 0                                     ← #1 position

127.0.0.1:6379> ZREVRANK leaderboard "bob"
(integer) 2                                     ← #3 position

# ZINCRBY — increment a member's score (atomic, like INCR)
127.0.0.1:6379> ZINCRBY leaderboard 15 "bob"
"87"                                            ← 72 + 15

# Bob just passed Carol! Let's check:
127.0.0.1:6379> ZREVRANGE leaderboard 0 -1 WITHSCORES
1) "alice"
2) "100"
3) "bob"
4) "87"                                         ← bob moved up!
5) "carol"
6) "85"
7) "dave"
8) "45"

# ZCARD — count members
127.0.0.1:6379> ZCARD leaderboard
(integer) 4

# ZREM — remove a member
127.0.0.1:6379> ZREM leaderboard "dave"
(integer) 1
```

**Range queries by score:**

```
# ZRANGEBYSCORE — get members within a score range
127.0.0.1:6379> ZRANGEBYSCORE leaderboard 80 100 WITHSCORES
1) "carol"
2) "85"
3) "bob"
4) "87"
5) "alice"
6) "100"

# Members with score >= 80
# Useful for: "show all users with reputation above threshold"
```

**Backend patterns with Sorted Sets:**

```
┌─────────────────────────────────────────────────────────────────┐
│       SORTED SET PATTERNS FOR BACKEND DEVELOPMENT               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PATTERN 1: LEADERBOARD                                        │
│  ────────────────────────                                       │
│  ZADD leaderboard {score} {user}                                │
│  ZINCRBY leaderboard {points} {user}     (score a point)        │
│  ZREVRANGE leaderboard 0 9 WITHSCORES    (top 10)               │
│  ZREVRANK leaderboard {user}             ("you are #5")         │
│                                                                 │
│                                                                 │
│  PATTERN 2: PRIORITY QUEUE                                      │
│  ────────────────────────                                       │
│  Score = priority (lower = more urgent)                         │
│  ZADD tasks:priority 1 "task:critical-bug"                      │
│  ZADD tasks:priority 5 "task:nice-to-have"                      │
│  ZADD tasks:priority 3 "task:user-request"                      │
│  ZRANGE tasks:priority 0 0    → "task:critical-bug" (highest    │
│                                   priority = lowest score)      │
│                                                                 │
│                                                                 │
│  PATTERN 3: TIME-SERIES INDEX                                   │
│  ────────────────────────────                                   │
│  Score = Unix timestamp                                         │
│  ZADD logins:recent 1707900000 "user:1"                         │
│  ZADD logins:recent 1707900060 "user:2"                         │
│  ZADD logins:recent 1707900120 "user:3"                         │
│                                                                 │
│  "Who logged in the last 5 minutes?"                            │
│  ZRANGEBYSCORE logins:recent {now-300} {now}                    │
│                                                                 │
│  This is how you could track your refresh tokens by             │
│  expiration time — score = expiry timestamp. Range query         │
│  gives you all tokens expiring in a window. But we'll           │
│  see an even simpler pattern with TTL in Lecture 3.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Compare the full type spectrum — where each shines:**

```
┌─────────────────────────────────────────────────────────────────┐
│          CHOOSING THE RIGHT DATA TYPE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "I need to store..."          Use...                           │
│  ──────────────────────────    ─────────────                    │
│  A single value/counter        String                           │
│  A cached JSON response        String (serialized JSON)         │
│  An object with fields         Hash                             │
│  An ordered log of events      List (LPUSH + LTRIM)             │
│  A job queue                   List (RPUSH + LPOP)              │
│  A collection of unique items  Set                              │
│  "Is X in the group?"          Set (SISMEMBER, O(1))            │
│  "What's in common?"           Set (SINTER)                     │
│  A ranked list                 Sorted Set (score = rank)        │
│  Items ordered by time         Sorted Set (score = timestamp)   │
│  "Top N" of anything           Sorted Set (ZREVRANGE 0 N)       │
│                                                                 │
│  When in doubt, start with String. It covers 70% of cases.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: TTL — AUTOMATIC EXPIRATION

## 5.1 Why Data Should Disappear

**This is the feature that makes Redis practical for caching.**

```
┌─────────────────────────────────────────────────────────────────┐
│               WHY DATA SHOULD DISAPPEAR                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Problem 1: STALE DATA                                          │
│  ─────────────────────                                          │
│  You cache the category list in Redis. Someone adds a new       │
│  category in PostgreSQL. Redis still serves the old list.       │
│  Users see stale data.                                          │
│                                                                 │
│  Solution: Set a TTL. Cache expires every 5 minutes.            │
│  Next request gets fresh data from PostgreSQL.                  │
│                                                                 │
│                                                                 │
│  Problem 2: MEMORY EXHAUSTION                                   │
│  ────────────────────────────                                   │
│  Redis lives in RAM. RAM is limited and expensive.              │
│  If you cache everything forever, you'll run out of memory.     │
│                                                                 │
│  Solution: TTL automatically frees memory when data expires.    │
│                                                                 │
│                                                                 │
│  Problem 3: SECURITY                                            │
│  ────────────────────                                           │
│  Remember refresh tokens from Week 9? A refresh token that      │
│  lives forever is a security risk. Tokens MUST expire.          │
│                                                                 │
│  Solution: Store tokens in Redis with TTL matching their        │
│  expiration. Token expired? Key is gone. Automatic.             │
│  No cleanup cron job. No stale tokens sitting in PostgreSQL.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Back to the desk analogy: you don't keep every file on your desk forever. You keep today's files. Yesterday's go back to the cabinet or the trash. TTL is the cleaning schedule."

---

## 5.2 EXPIRE, TTL — Setting the Timer

**EXPIRE sets a countdown on a key. When it reaches zero, the key vanishes.**

```
# Create a key
127.0.0.1:6379> SET session:abc123 '{"user_id":1,"role":"admin"}'
OK

# Set it to expire in 3600 seconds (1 hour)
127.0.0.1:6379> EXPIRE session:abc123 3600
(integer) 1                                    ← 1 = success

# Check remaining time
127.0.0.1:6379> TTL session:abc123
(integer) 3597                                 ← 3597 seconds left

# ... time passes ...

127.0.0.1:6379> TTL session:abc123
(integer) 3200                                 ← ticking down

# Check a key with no expiration
127.0.0.1:6379> SET permanent "I live forever"
OK

127.0.0.1:6379> TTL permanent
(integer) -1                                   ← -1 = no expiration set

# Check a key that doesn't exist
127.0.0.1:6379> TTL nonexistent
(integer) -2                                   ← -2 = key doesn't exist
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   TTL RETURN VALUES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TTL key → positive number   (seconds remaining)                │
│  TTL key → -1                (key exists, NO expiration)        │
│  TTL key → -2                (key does NOT exist)               │
│                                                                 │
│  These three cases matter in your application logic.            │
│  -1 might mean a bug: you forgot to set TTL on a cache entry.  │
│  -2 means cache miss: go to PostgreSQL.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**PEXPIRE and PTTL — millisecond precision:**

```
# PEXPIRE — expire in milliseconds
127.0.0.1:6379> SET temp "short-lived"
OK

127.0.0.1:6379> PEXPIRE temp 5000         ← 5000ms = 5 seconds
(integer) 1

127.0.0.1:6379> PTTL temp
(integer) 4823                             ← milliseconds remaining

# Useful for very short-lived items like distributed locks
```

**PERSIST — remove the expiration:**

```
127.0.0.1:6379> SET cache:config '{"feature_x":true}'
OK

127.0.0.1:6379> EXPIRE cache:config 300
(integer) 1

127.0.0.1:6379> TTL cache:config
(integer) 298

# "Actually, I want this to stay indefinitely"
127.0.0.1:6379> PERSIST cache:config
(integer) 1

127.0.0.1:6379> TTL cache:config
(integer) -1                               ← no expiration
```

---

## 5.3 SET with EX — Atomic Create + Expire

**Setting a key and its TTL separately is TWO commands. What if your app crashes between them?**

```
# TWO COMMANDS (not ideal):
127.0.0.1:6379> SET cache:user:1 '{"name":"Alice"}'
OK
# --- what if your application crashes RIGHT HERE? ---
127.0.0.1:6379> EXPIRE cache:user:1 300
# This never runs. Key exists forever. Memory leak.
```

**SET with EX does both atomically in ONE command:**

```
# ONE COMMAND (atomic — use this):
127.0.0.1:6379> SET cache:user:1 '{"name":"Alice"}' EX 300
OK
# Key is created AND expires in 300 seconds. Guaranteed.

127.0.0.1:6379> TTL cache:user:1
(integer) 298
```

**SET options for TTL:**

```
127.0.0.1:6379> SET key value EX 300      ← expire in 300 SECONDS
127.0.0.1:6379> SET key value PX 5000     ← expire in 5000 MILLISECONDS
```

**The complete pattern for caching:**

```
┌─────────────────────────────────────────────────────────────────┐
│             THE COMPLETE CACHE PATTERN                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pseudocode for what you'll write in Python in Lecture 2:       │
│                                                                 │
│  1. GET cache:categories:org:789                                │
│                                                                 │
│  2. If result is not nil → return it (CACHE HIT, ~0.1ms)       │
│                                                                 │
│  3. If result is nil → (CACHE MISS)                             │
│     a. Query PostgreSQL for categories  (~5ms)                  │
│     b. SET cache:categories:org:789 "{...}" EX 300              │
│     c. Return the data                                          │
│                                                                 │
│  Effect:                                                        │
│  ├─ First request: ~5ms (DB query + Redis write)                │
│  ├─ Next 299 seconds of requests: ~0.1ms each (Redis only)     │
│  ├─ After 300 seconds: cache expires, next request = ~5ms       │
│  └─ Cycle repeats                                               │
│                                                                 │
│  You've just reduced PostgreSQL load by ~99% for this query.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**TTL applies to ALL data types, not just strings:**

```
# Hash with TTL
127.0.0.1:6379> HSET session:abc123 user_id 1 role "admin"
(integer) 2
127.0.0.1:6379> EXPIRE session:abc123 3600
(integer) 1

# List with TTL
127.0.0.1:6379> RPUSH recent:search:user:1 "query 1" "query 2"
(integer) 2
127.0.0.1:6379> EXPIRE recent:search:user:1 86400
(integer) 1

# Set with TTL
127.0.0.1:6379> SADD visitors:2026-02-14 "user:1" "user:2"
(integer) 2
127.0.0.1:6379> EXPIRE visitors:2026-02-14 172800
(integer) 1

# Sorted Set with TTL
127.0.0.1:6379> ZADD leaderboard:weekly 100 "alice" 85 "bob"
(integer) 2
127.0.0.1:6379> EXPIRE leaderboard:weekly 604800
(integer) 1
```

> "TTL is per KEY, not per value or per field. When a hash expires, ALL its fields disappear. When a list expires, ALL its elements are gone. One timer, one key, everything inside it."

---

## 5.4 What Redis is NOT (Misconceptions)

**Now that you've seen what Redis can do, let's be clear about what it isn't.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHAT REDIS IS NOT                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  ❌ REDIS IS NOT A REPLACEMENT FOR POSTGRESQL                   │
│  ────────────────────────────────────────────                   │
│  Redis has no tables, no JOINs, no SQL, no foreign keys,       │
│  no constraints, no ACID transactions in the traditional       │
│  sense, no complex queries. PostgreSQL is your source of       │
│  truth. Redis is your fast-access layer.                       │
│                                                                 │
│  ❌ REDIS IS NOT GUARANTEED PERMANENT                           │
│  ────────────────────────────────────                           │
│  By default, Redis data can be lost if the server crashes.     │
│  Redis has persistence options (RDB snapshots, AOF logs),      │
│  but you should design your system to handle Redis being       │
│  empty. If Redis goes down, you fall back to PostgreSQL.       │
│  That's "graceful degradation" — we'll implement it in         │
│  Lecture 2.                                                     │
│                                                                 │
│  ❌ REDIS IS NOT INFINITE                                       │
│  ────────────────────────                                       │
│  Redis is limited by available RAM. A server with 16GB RAM     │
│  can store ~16GB of data. PostgreSQL with the same server      │
│  can store terabytes on disk. Be selective about what you      │
│  cache. Cache HOT data, not everything.                        │
│                                                                 │
│  ❌ REDIS IS NOT A COMPLEX QUERY ENGINE                         │
│  ──────────────────────────────────────                         │
│  "Find all users where age > 25 AND city = 'London'            │
│   ORDER BY name" — this is a PostgreSQL query.                 │
│  Redis finds things by KEY. You must know the key to get       │
│  the value. Design your key structure around your              │
│  access patterns.                                               │
│                                                                 │
│                                                                 │
│  THE GOLDEN RULE:                                               │
│  ┌──────────────────────────────────────────────────────┐       │
│  │  PostgreSQL = source of truth (reliable, queryable)  │       │
│  │  Redis = performance layer (fast, temporary, simple) │       │
│  │                                                      │       │
│  │  If Redis dies, your app should STILL WORK.          │       │
│  │  Slower, but functional.                             │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**On persistence — awareness, not deep dive:**

```
┌─────────────────────────────────────────────────────────────────┐
│          REDIS PERSISTENCE (AWARENESS ONLY)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Redis CAN save data to disk. Two mechanisms:                   │
│                                                                 │
│  RDB (Redis Database Backup)                                    │
│  ├─ Point-in-time snapshots at intervals                        │
│  ├─ "Save everything to disk every 5 minutes"                   │
│  └─ Can lose up to 5 minutes of data on crash                   │
│                                                                 │
│  AOF (Append Only File)                                         │
│  ├─ Logs every write command                                    │
│  ├─ Replays log on restart                                      │
│  └─ Can be configured for minimal data loss                     │
│                                                                 │
│  For our course, we use Redis as a CACHE. If it dies, we        │
│  rebuild from PostgreSQL. Persistence is a safety net,          │
│  not a guarantee we depend on.                                  │
│                                                                 │
│  In production: you'd typically enable both RDB + AOF.          │
│  But your application logic should never ASSUME Redis has       │
│  the data. Always have the PostgreSQL fallback.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                REDIS QUICK REFERENCE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONNECT:                                                       │
│      docker compose exec redis redis-cli                        │
│      PING → PONG                                                │
│                                                                 │
│  STRINGS:                                                       │
│      SET key value                  (create/overwrite)          │
│      SET key value EX 300           (with 5-min TTL)            │
│      SET key value NX               (only if not exists)        │
│      GET key                        (retrieve)                  │
│      DEL key                        (delete)                    │
│      INCR key                       (atomic +1)                 │
│      INCRBY key 5                   (atomic +5)                 │
│      MSET k1 v1 k2 v2              (batch set)                 │
│      MGET k1 k2                     (batch get)                 │
│                                                                 │
│  HASHES:                                                        │
│      HSET key field value           (set field)                 │
│      HGET key field                 (get one field)             │
│      HGETALL key                    (get all fields)            │
│      HDEL key field                 (delete field)              │
│      HEXISTS key field              (check field exists)        │
│      HINCRBY key field 1            (atomic field increment)    │
│                                                                 │
│  LISTS:                                                         │
│      LPUSH key value                (add to front)              │
│      RPUSH key value                (add to back)               │
│      LPOP key                       (remove from front)         │
│      RPOP key                       (remove from back)          │
│      LRANGE key 0 -1               (get all elements)          │
│      LLEN key                       (count elements)            │
│      LTRIM key 0 99                 (keep only first 100)       │
│                                                                 │
│  SETS:                                                          │
│      SADD key member                (add member)                │
│      SREM key member                (remove member)             │
│      SISMEMBER key member           (check membership)          │
│      SMEMBERS key                   (get all members)           │
│      SCARD key                      (count members)             │
│      SINTER key1 key2              (intersection)              │
│      SUNION key1 key2              (union)                     │
│                                                                 │
│  SORTED SETS:                                                   │
│      ZADD key score member          (add with score)            │
│      ZSCORE key member              (get score)                 │
│      ZRANK key member               (rank, low to high)         │
│      ZREVRANK key member            (rank, high to low)         │
│      ZRANGE key 0 -1 WITHSCORES    (all, low to high)          │
│      ZREVRANGE key 0 9 WITHSCORES  (top 10)                    │
│      ZINCRBY key 5 member           (increment score)           │
│      ZRANGEBYSCORE key min max      (filter by score)           │
│                                                                 │
│  TTL:                                                           │
│      EXPIRE key 300                 (expire in 300 seconds)     │
│      PEXPIRE key 5000               (expire in 5000 ms)         │
│      TTL key                        (seconds remaining)         │
│      PERSIST key                    (remove expiration)         │
│                                                                 │
│  KEY MANAGEMENT:                                                │
│      EXISTS key                     (does key exist?)           │
│      TYPE key                       (what type is this key?)    │
│      KEYS pattern                   (⚠ DEV ONLY — blocks)      │
│      SCAN 0 MATCH pattern COUNT 100 (production-safe scan)     │
│      DBSIZE                         (total key count)           │
│      FLUSHDB                        (⚠ delete everything)      │
│                                                                 │
│  KEY NAMING CONVENTION:                                         │
│      entity:id:field                                            │
│      cache:dashboard:user:123                                   │
│      session:abc123def                                          │
│      ratelimit:login:192.168.1.1                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  REDIS = YOUR WORKING DESK                                      │
│                                                                 │
│  Fast (in-memory), limited (RAM), temporary (TTL).              │
│  PostgreSQL is the filing cabinet: permanent, vast, slower.     │
│  You use BOTH — not one or the other.                           │
│                                                                 │
│                                                                 │
│  ┌────────────────┐      ┌────────────────┐                     │
│  │     Redis      │      │  PostgreSQL    │                     │
│  │   (The Desk)   │      │ (The Cabinet)  │                     │
│  │                │      │                │                     │
│  │ • ~0.1ms reads │      │ • ~5ms reads   │                     │
│  │ • RAM storage  │      │ • Disk storage │                     │
│  │ • Key → Value  │      │ • SQL queries  │                     │
│  │ • Data expires │      │ • Data persists│                     │
│  │ • 5 data types │      │ • Relational   │                     │
│  │ • No JOINs     │      │ • Full ACID    │                     │
│  └────────────────┘      └────────────────┘                     │
│           │                       │                             │
│           └───────────┬───────────┘                              │
│                       │                                         │
│              Your FastAPI Application                           │
│         Check Redis first, fall back to PostgreSQL              │
│                                                                 │
│                                                                 │
│  DATA TYPES — THINK IN PYTHON:                                  │
│  ├─ String     →  variable (text, number, JSON blob)            │
│  ├─ Hash       →  dict     (object with fields)                │
│  ├─ List       →  deque    (queue, feed, log)                  │
│  ├─ Set        →  set      (unique items, membership)          │
│  └─ Sorted Set →  ✨       (rankings, top-N, priority)         │
│                                                                 │
│  TTL — THE CLEANUP CREW:                                        │
│  ├─ Every cache entry MUST have a TTL                           │
│  ├─ SET key value EX seconds (atomic, prefer this)              │
│  ├─ Prevents stale data, memory leaks, security risks           │
│  └─ If Redis dies, your app still works (just slower)           │
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
│  WEEK 10, LECTURE 2: Redis in Python & Caching Patterns         │
│  ├─ redis.asyncio client (connect from FastAPI)                 │
│  ├─ Redis as a FastAPI dependency (like your DB session)        │
│  ├─ Cache-aside pattern in actual Python code                   │
│  ├─ Cache invalidation strategies (the hard part)               │
│  └─ Cache key design and stampede prevention                    │
│                                                                 │
│  WEEK 10, LECTURE 3: Session & Token Storage                    │
│  ├─ Move refresh tokens from PostgreSQL to Redis                │
│  │   (remember Week 9? TTL replaces expiry column checking)     │
│  ├─ Token revocation with DEL (instant logout)                  │
│  ├─ Distributed rate limiting with INCR + EXPIRE                │
│  └─ Graceful degradation when Redis is unavailable              │
│                                                                 │
│  WEEK 10 PROJECT: Add Caching Layer                             │
│  └─ You'll apply every data type from today:                    │
│     Strings for cached responses, Hashes for sessions,          │
│     Lists for recent activity, Sets for online users,           │
│     Sorted Sets for "trending" items.                           │
│                                                                 │
│  WEEK 11: Background Jobs                                       │
│  └─ Redis as Celery's message broker                            │
│     (remember RPUSH + LPOP = queue? Celery automates this)     │
│     Redis Pub/Sub for event broadcasting                        │
│                                                                 │
│  WEEK 12: WebSockets & Real-Time                                │
│  └─ Redis Pub/Sub for scaling WebSocket messages                │
│     across multiple server instances                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```