# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ZOOM OUT, NOT UP                                               │
│  ────────────────                                               │
│  For 15 weeks, students have lived INSIDE a single server.      │
│  Today we step outside and look at the building from above.     │
│  Every concept maps back to code they've already written.       │
│                                                                 │
│  NUMBERS BEFORE THEORY                                          │
│  ─────────────────────                                          │
│  Students must CALCULATE why one server fails before we         │
│  teach them how to add more. No hand-waving.                    │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  We extend Week 1's restaurant to a RESTAURANT CHAIN.           │
│  Every system design concept maps to growing a business.        │
│                                                                 │
│  TRADEOFFS, NOT ANSWERS                                         │
│  ──────────────────────                                         │
│  System design has no "correct" solution. Only tradeoffs.       │
│  We teach students to CHOOSE and JUSTIFY, not memorize.         │
│                                                                 │
│  CONNECT EVERYTHING                                             │
│  ────────────────                                               │
│  JWT, Redis, Celery, Docker, async — students have built all    │
│  the pieces. Today they see how those pieces form a machine.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                 SYSTEM DESIGN FUNDAMENTALS                      │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE BREAKING POINT (45 min)                            │
│  ├─ 1.1 Your Capstone Under Siege                               │
│  ├─ 1.2 What is System Design?                                  │
│  ├─ 1.3 The Restaurant Chain Analogy                            │
│  ├─ 1.4 The Framework (Requirements → Estimation → Design       │
│  │       → Tradeoffs)                                           │
│  └─ 1.5 Back-of-Envelope Estimation                             │
│                                                                 │
│  PART 2: SCALING YOUR SERVERS (50 min)                          │
│  ├─ 2.1 Vertical Scaling (A Bigger Restaurant)                  │
│  ├─ 2.2 Horizontal Scaling (More Locations)                     │
│  ├─ 2.3 The State Problem (Why Horizontal Is Hard)              │
│  ├─ 2.4 Load Balancers (The Central Reservation System)         │
│  ├─ 2.5 Load Balancing Algorithms                               │
│  └─ 2.6 Health Checks (Is This Location Open?)                  │
│                                                                 │
│  PART 3: SCALING YOUR DATABASE (50 min)                         │
│  ├─ 3.1 The Database Bottleneck                                 │
│  ├─ 3.2 Read Replicas (Copies of the Recipe Book)               │
│  ├─ 3.3 Replication Lag (The Update Delay)                      │
│  ├─ 3.4 Sharding (Splitting the Kitchen)                        │
│  ├─ 3.5 Sharding Strategies                                     │
│  └─ 3.6 The Decision Framework                                  │
│                                                                 │
│  PART 4: CACHING AT EVERY LAYER (40 min)                        │
│  ├─ 4.1 The Caching Hierarchy                                   │
│  ├─ 4.2 CDN (The Menu Board Outside the Building)               │
│  ├─ 4.3 Application Cache (Your Redis Knowledge)                │
│  ├─ 4.4 Database-Level Caching                                  │
│  └─ 4.5 Cache Invalidation (The Hardest Problem)                │
│                                                                 │
│  PART 5: CAP THEOREM & DESIGN THINKING (40 min)                 │
│  ├─ 5.1 The CAP Theorem                                         │
│  ├─ 5.2 CP vs AP in Practice                                    │
│  ├─ 5.3 Everything is a Tradeoff                                │
│  └─ 5.4 Full Architecture — Putting It All Together             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE BREAKING POINT

## 1.1 Your Capstone Under Siege

**Start by putting their own project on trial. Make it personal.**

Draw the architecture of their capstone on the whiteboard:

```
┌─────────────────────────────────────────────────────────────────┐
│            YOUR CAPSTONE RIGHT NOW (Week 13-14)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                        ┌─────────┐                              │
│                        │ Client  │                              │
│                        └────┬────┘                              │
│                             │                                   │
│                             ▼                                   │
│                   ┌──────────────────┐                           │
│                   │   ONE SERVER     │                           │
│                   │  ┌────────────┐  │                           │
│                   │  │  FastAPI   │  │                           │
│                   │  │  Celery    │  │                           │
│                   │  │  Redis     │  │                           │
│                   │  └────────────┘  │                           │
│                   └────────┬─────────┘                           │
│                            │                                    │
│                            ▼                                    │
│                   ┌──────────────────┐                           │
│                   │   PostgreSQL     │                           │
│                   │  (ONE instance)  │                           │
│                   └──────────────────┘                           │
│                                                                 │
│   This is EVERYTHING on ONE machine.                            │
│   Your API, your workers, your cache, your database.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "Your capstone is deployed. It works. Your load test from Week 12 showed it handles 500 requests/min with <200ms p95. You're proud of it. Now imagine you post it on Hacker News and it goes viral. What happens?"

Pause. Let them think.

> "Let's not guess. Let's do math."

**Walk through the numbers — scale progression:**

```
┌─────────────────────────────────────────────────────────────────┐
│              YOUR CAPSTONE AT DIFFERENT SCALES                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USERS      DAU       REQ/DAY       PEAK REQ/SEC    STATUS     │
│  ──────     ───       ───────       ───────────     ──────     │
│  1,000      100       5,000         ~1              😊 Fine     │
│  10,000     1,000     50,000        ~6              😊 Fine     │
│  100,000    10,000    500,000       ~58             😐 Warm     │
│  1,000,000  100,000   5,000,000     ~580            😰 Trouble  │
│  10,000,000 1,000,000 50,000,000    ~5,800          💀 Dead     │
│                                                                 │
│  ASSUMPTIONS:                                                   │
│  ├─ 10% of total users active daily (DAU)                       │
│  ├─ Each active user makes ~50 requests/day                     │
│  ├─ Peak traffic = ~4 hours where 40% of daily traffic lands    │
│  └─ Peak RPS = (daily_requests × 0.4) / (4 × 3600)             │
│                                                                 │
│  YOUR LOAD TEST: 500 req/min = ~8 req/sec sustained             │
│                                                                 │
│  CONCLUSION: Your single server dies somewhere between           │
│              100K and 1M total users.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**But it's not just requests. What else breaks?**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT BREAKS AND WHY                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. CPU SATURATION                                              │
│     Your FastAPI process maxes out CPU.                         │
│     Requests queue up. Latency spikes. Timeouts.                │
│     Even async can't help — the event loop is overloaded.       │
│                                                                 │
│  2. DATABASE CONNECTIONS EXHAUSTED                              │
│     PostgreSQL default: max_connections = 100.                  │
│     Your connection pool from Week 6 fills up.                  │
│     New requests get "connection refused."                      │
│                                                                 │
│  3. MEMORY EXHAUSTION                                           │
│     Each WebSocket connection (Week 12) holds state.            │
│     10,000 concurrent WebSockets ≈ hundreds of MB.              │
│     Redis, Celery workers, FastAPI — all compete for RAM.       │
│                                                                 │
│  4. DISK I/O BOTTLENECK                                         │
│     PostgreSQL writes WAL logs, indexes, temp files.            │
│     Heavy queries cause disk thrashing.                         │
│     Your EXPLAIN ANALYZE from Week 7 showed fast queries —      │
│     but that was with 1,000 rows, not 500 million.              │
│                                                                 │
│  5. SINGLE POINT OF FAILURE                                     │
│     Server crashes → EVERYTHING goes down.                      │
│     Database corrupts → ALL data at risk.                       │
│     No redundancy. No fallback. No recovery.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "Your capstone isn't broken. It's *unscaled*. Every single decision you made — FastAPI, async, Redis, Celery, Docker — was a correct decision. But you designed it for one machine. System design is the discipline of making it work on *many* machines."

---

## 1.2 What is System Design?

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHAT IS SYSTEM DESIGN?                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  System design is the process of defining the ARCHITECTURE,     │
│  COMPONENTS, and DATA FLOW of a system to satisfy               │
│  requirements at SCALE.                                         │
│                                                                 │
│  It answers:                                                    │
│  ├─ How do we handle millions of users?                         │
│  ├─ How do we survive server failures?                          │
│  ├─ How do we keep responses fast as data grows?                │
│  ├─ How do we evolve the system without downtime?               │
│  └─ What do we SACRIFICE to achieve the above?                  │
│                                                                 │
│                                                                 │
│  WHAT IT IS NOT:                                                │
│  ├─ ✗ Writing code (you've done that for 15 weeks)              │
│  ├─ ✗ Choosing frameworks (FastAPI, SQLAlchemy — done)          │
│  ├─ ✗ A single "correct answer" (always tradeoffs)              │
│  └─ ✗ Only relevant at Google-scale (a 10-person startup needs  │
│       it too — just different decisions)                         │
│                                                                 │
│                                                                 │
│  WHAT IT IS:                                                    │
│  ├─ ✓ Deciding HOW components connect                           │
│  ├─ ✓ Deciding WHERE data lives                                 │
│  ├─ ✓ Deciding WHAT happens when things fail                    │
│  └─ ✓ JUSTIFYING your choices with numbers and tradeoffs        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "For the past 15 weeks, you've been an architect designing the interior of a building — the rooms, the plumbing, the wiring. System design is about designing the *city* — where do buildings go, how do roads connect them, what happens when a bridge collapses?"

---

## 1.3 The Restaurant Chain Analogy

**In Week 1, we used a single restaurant to explain async. Now we're growing a chain.**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE RESTAURANT CHAIN ANALOGY                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your capstone is a RESTAURANT.                                 │
│                                                                 │
│  Right now:                                                     │
│  ├─ One location                                                │
│  ├─ One kitchen (database)                                      │
│  ├─ A few waiters (async workers)                               │
│  ├─ One fridge for prep (Redis cache)                           │
│  ├─ One prep cook doing tasks overnight (Celery)                │
│  └─ Serves 200 customers/day comfortably                        │
│                                                                 │
│  Now imagine 50,000 customers want to eat there daily.          │
│  What do you do?                                                │
│                                                                 │
│  OPTION A: Make the restaurant BIGGER                           │
│            (Vertical Scaling)                                   │
│                                                                 │
│  OPTION B: Open MORE restaurants                                │
│            (Horizontal Scaling)                                 │
│                                                                 │
│  This analogy will carry us through the entire lecture.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The mapping:**

```
Restaurant Chain             │  Distributed System
─────────────────────────────│──────────────────────────────
One restaurant               │  Single server
Bigger restaurant            │  Vertical scaling
Multiple locations           │  Horizontal scaling
Central reservation hotline  │  Load balancer
Master recipe book (HQ)      │  Primary database
Copies at each location      │  Read replicas
Splitting by cuisine type    │  Database sharding
Menu board outside building  │  CDN
Daily specials board (front) │  Application cache (Redis)
Pre-prepped sauce bases      │  Database query cache
"Is this location open?"     │  Health checks
Every location, same menu    │  Consistency
Every location, always open  │  Availability
Phone lines between go down  │  Network partition
```

---

## 1.4 The Framework: Requirements → Estimation → Design → Tradeoffs

**Before you design anything, you need a PROCESS. This is the system design thinking framework — it's also how system design interviews work:**

```
┌─────────────────────────────────────────────────────────────────┐
│             THE SYSTEM DESIGN FRAMEWORK                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: CLARIFY REQUIREMENTS (5 min in interview)              │
│  ─────────────────────────────                                  │
│  │                                                              │
│  ├─ Functional: What does it DO?                                │
│  │  "Users create tasks, assign them, get notifications"        │
│  │                                                              │
│  └─ Non-Functional: What QUALITIES must it have?                │
│     "< 200ms latency, 99.9% uptime, 10M users"                 │
│                                                                 │
│          │                                                      │
│          ▼                                                      │
│                                                                 │
│  STEP 2: BACK-OF-ENVELOPE ESTIMATION (5 min)                    │
│  ────────────────────────────────────                           │
│  │                                                              │
│  ├─ How many users? How many requests/second?                   │
│  ├─ How much data? How much storage?                            │
│  └─ What bandwidth? What are the bottlenecks?                   │
│                                                                 │
│          │                                                      │
│          ▼                                                      │
│                                                                 │
│  STEP 3: HIGH-LEVEL DESIGN (10 min)                             │
│  ─────────────────────────                                      │
│  │                                                              │
│  ├─ Draw the main components (boxes and arrows)                 │
│  ├─ Identify the data flow                                      │
│  └─ Pick the core technologies                                  │
│                                                                 │
│          │                                                      │
│          ▼                                                      │
│                                                                 │
│  STEP 4: DETAILED DESIGN (15 min)                               │
│  ────────────────────────                                       │
│  │                                                              │
│  ├─ Zoom into critical components                               │
│  ├─ Database schema, API design, caching strategy               │
│  └─ Handle edge cases and failure modes                         │
│                                                                 │
│          │                                                      │
│          ▼                                                      │
│                                                                 │
│  STEP 5: IDENTIFY TRADEOFFS & BOTTLENECKS (5 min)               │
│  ────────────────────────────────────────                       │
│  │                                                              │
│  ├─ What breaks first under 10x load?                           │
│  ├─ What did you sacrifice? Why is that acceptable?             │
│  └─ What would you change with more time/budget?                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why this order?**

> "Notice: you don't draw a SINGLE box until Step 3. Most beginners jump straight to drawing diagrams. But without understanding the requirements and the scale, your diagram is fiction. You wouldn't design a restaurant without knowing if it serves 50 or 50,000 people per day."

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  FUNCTIONAL REQUIREMENTS         NON-FUNCTIONAL REQUIREMENTS    │
│  (What the system does)          (How well it does it)          │
│  ─────────────────────           ─────────────────────          │
│                                                                 │
│  • Users can create tasks        • 99.9% uptime                 │
│  • Users can assign tasks        • < 200ms p95 latency          │
│  • Real-time notifications       • Support 1M daily users       │
│  • Search and filter tasks       • Data must survive failures   │
│  • Export reports as CSV         • Audit trail for 7 years      │
│                                                                 │
│  These are WHAT you build.       These decide HOW you build.    │
│                                                                 │
│  You already know how to         Today is about THESE.          │
│  build all of these.                                            │
│  (You did it in your capstone.)                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.5 Back-of-Envelope Estimation

**This is the most underrated skill in system design. Let's work a real example using your capstone: the Project Management SaaS.**

**First, memorize these reference numbers:**

```
┌─────────────────────────────────────────────────────────────────┐
│               NUMBERS EVERY ENGINEER SHOULD KNOW               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TIME                                                           │
│  ────                                                           │
│  1 day ≈ 86,400 sec ≈ 100,000 sec (for quick math)             │
│  1 day peak (assume 8 busy hours) = 28,800 sec ≈ 30,000 sec    │
│                                                                 │
│  THROUGHPUT (rough single-machine numbers)                      │
│  ──────────                                                     │
│  FastAPI (async):           ~1,000 – 10,000 req/sec             │
│  PostgreSQL simple reads:   ~10,000 queries/sec                 │
│  PostgreSQL writes:         ~1,000 – 5,000 queries/sec          │
│  Redis operations:          ~100,000 ops/sec                    │
│                                                                 │
│  STORAGE                                                        │
│  ───────                                                        │
│  1 JSON API response:       ~1-5 KB                             │
│  1 database row:            ~0.5-2 KB                           │
│  1 million rows × 1KB:     ~1 GB                                │
│  1 billion rows × 1KB:     ~1 TB                                │
│                                                                 │
│  NETWORK                                                        │
│  ───────                                                        │
│  Typical server NIC:        1 Gbps ≈ 125 MB/sec                │
│  Average API response:      ~2 KB                               │
│  1 Gbps / 2KB =            ~62,000 responses/sec (not the      │
│                              bottleneck for most apps)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Worked example: Your capstone at 1M total users.**

```
┌─────────────────────────────────────────────────────────────────┐
│        ESTIMATION: PROJECT MANAGEMENT SAAS (1M USERS)          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: TRAFFIC                                                │
│  ───────────────                                                │
│  Total users:              1,000,000                            │
│  Daily active (10%):       100,000                              │
│  Actions per user per day: ~50 (page loads, creates, edits)     │
│  Total requests/day:       5,000,000                            │
│                                                                 │
│  Average RPS:  5,000,000 / 86,400 ≈ 58 req/sec                 │
│  Peak RPS (3x): ≈ 175 req/sec                                  │
│                                                                 │
│  ✅ One FastAPI server can handle this (you tested ~8 RPS       │
│     in Week 12 load test, but that was unoptimized with DB      │
│     queries. Tuned, a single server can push 500-2000 RPS).     │
│                                                                 │
│                                                                 │
│  STEP 2: STORAGE                                                │
│  ────────────────                                               │
│  Users table:              1M × 1KB = 1 GB                      │
│  Organizations:            100K × 0.5KB = 50 MB                 │
│  Tasks (avg 30/user):      30M × 2KB = 60 GB                   │
│  Audit log (10 events/     10M events/day × 0.5KB = 5 GB/day   │
│     user/day):             × 365 days = 1.8 TB/year             │
│                                                                 │
│  ⚠️  Audit log grows FAST. This will be a problem.              │
│                                                                 │
│                                                                 │
│  STEP 3: DATABASE LOAD                                          │
│  ─────────────────────                                          │
│  Read/write ratio:         ~90% reads / 10% writes              │
│  Read queries:             ~160 reads/sec at peak                │
│  Write queries:            ~18 writes/sec at peak               │
│                                                                 │
│  ✅ PostgreSQL handles this easily.                              │
│     But at 10M users? 1,750 reads/sec peak.                     │
│     With complex JOINs? That's where it breaks.                 │
│                                                                 │
│                                                                 │
│  STEP 4: WEBSOCKET CONNECTIONS                                  │
│  ──────────────────────────────                                 │
│  Concurrent users (peak):  ~30,000 (30% of DAU online at once)  │
│  Each WS connection:       ~10 KB memory                        │
│  Total WS memory:          ~300 MB                              │
│                                                                 │
│  ⚠️  30K concurrent connections is a lot for one process.        │
│     Linux default file descriptor limit: 1024.                  │
│     Need to tune OS settings AND think about scaling.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "At 1M users, we're *probably* fine on one beefy server. Tight, but fine. What about 10M users? Everything we just calculated multiplies by 10. 1,750 peak reads/sec, 300K WebSocket connections, 18 TB/year audit logs. One machine cannot do that. *That's* when system design kicks in."

> "The point isn't the exact numbers. It's the HABIT: estimate before you design. This turns 'I think we need more servers' into 'We need at least 3 app servers and 2 database replicas to handle projected peak load.' That's engineering."

---

# PART 2: SCALING YOUR SERVERS

## 2.1 Vertical Scaling (A Bigger Restaurant)

```
┌─────────────────────────────────────────────────────────────────┐
│                    VERTICAL SCALING                             │
│               "Make the machine BIGGER"                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE                          AFTER                          │
│  ──────                          ─────                          │
│  ┌─────────────┐                 ┌──────────────────────┐       │
│  │   4 CPU     │                 │   32 CPU             │       │
│  │   8 GB RAM  │    ────▶        │   128 GB RAM         │       │
│  │   100 GB SSD│                 │   2 TB NVMe SSD      │       │
│  │   $50/month │                 │   $800/month         │       │
│  └─────────────┘                 └──────────────────────┘       │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  "Knock down walls, expand the dining room, buy a bigger        │
│   oven, hire more waiters — but it's still ONE restaurant."     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Advantages and limits:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ✅ ADVANTAGES:                    ❌ LIMITS:                    │
│  ──────────────                    ──────────                   │
│  Simple — no code changes          Hardware has a ceiling       │
│  No distributed complexity         (you can't buy a 10,000-    │
│  No network hops between           core machine)               │
│    components                                                   │
│  Same deployment process           Cost grows non-linearly     │
│  Your capstone works AS-IS         (2x CPU ≠ 2x cost, often   │
│                                     more like 3-4x)            │
│                                                                 │
│                                    Still ONE failure point     │
│                                    (server dies = app dies)    │
│                                                                 │
│                                    Downtime during upgrade     │
│                                    (must stop, resize, start)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**When to use vertical scaling:**

> "Vertical scaling is your FIRST move. Always. It's the cheapest, simplest fix. Don't build a distributed system when a $200/month server solves your problem. Many successful startups run on a single beefy machine for years."

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  THE VERTICAL SCALING RULE OF THUMB:                            │
│                                                                 │
│  If your problem can be solved by throwing money at a bigger    │
│  server, DO THAT FIRST. Distributed systems are an order of     │
│  magnitude more complex. You earn that complexity with growth.  │
│                                                                 │
│  Go horizontal when:                                            │
│  ├─ The biggest available machine isn't enough                  │
│  ├─ You need redundancy (can't afford single-server downtime)   │
│  └─ Cost of scaling up exceeds cost of scaling out              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.2 Horizontal Scaling (More Locations)

```
┌─────────────────────────────────────────────────────────────────┐
│                   HORIZONTAL SCALING                            │
│               "Add MORE machines"                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE                          AFTER                          │
│  ──────                          ─────                          │
│  ┌─────────────┐                 ┌────────────┐                 │
│  │  Server 1   │                 │  Server 1  │                 │
│  │  (handles   │    ────▶        │  Server 2  │                 │
│  │   everything│                 │  Server 3  │                 │
│  │   alone)    │                 │  Server 4  │                 │
│  └─────────────┘                 └────────────┘                 │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  "Open a second location. Then a third. Then a fourth.          │
│   Same menu, same quality, multiple places serving customers."  │
│                                                                 │
│                                                                 │
│  ✅ ADVANTAGES:                    ❌ COMPLEXITY:                │
│  ──────────────                    ──────────────               │
│  Near-infinite scaling             Must coordinate between      │
│  (add more machines)               servers                      │
│                                                                 │
│  Redundancy built in               Where does data live?        │
│  (one dies, others continue)       Which server handles which   │
│                                    request?                     │
│  Cheaper at scale                                               │
│  (many small > one huge)           How do they share state?     │
│                                    How do you deploy changes?   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What your architecture looks like horizontally scaled:**

```
┌─────────────────────────────────────────────────────────────────┐
│          HORIZONTAL ARCHITECTURE (simplified)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                       ┌──────────┐                              │
│                       │ Clients  │                              │
│                       └────┬─────┘                              │
│                            │                                    │
│                            ▼                                    │
│                   ┌────────────────┐                             │
│                   │ LOAD BALANCER  │  ← NEW (we'll cover soon)  │
│                   └───┬────┬────┬──┘                             │
│                       │    │    │                                │
│              ┌────────┘    │    └────────┐                       │
│              ▼             ▼             ▼                       │
│        ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│        │ FastAPI  │ │ FastAPI  │ │ FastAPI  │                   │
│        │ Server 1 │ │ Server 2 │ │ Server 3 │                   │
│        └────┬─────┘ └────┬─────┘ └────┬─────┘                   │
│             │            │            │                          │
│             └──────┬─────┴─────┬──────┘                          │
│                    ▼           ▼                                 │
│            ┌────────────┐ ┌────────────┐                         │
│            │ PostgreSQL │ │   Redis    │  ← Shared resources    │
│            └────────────┘ └────────────┘                         │
│                                                                 │
│   THREE copies of your FastAPI app,                             │
│   sharing ONE database and ONE Redis.                           │
│   Each can handle requests independently.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to Week 15:**

> "Remember Docker Compose from Week 15? You could spin this up today. Three containers running the same FastAPI image, one Postgres container, one Redis container. The infrastructure knowledge you have already supports this."

---

## 2.3 The State Problem (Why Horizontal Is Hard)

**This is the CENTRAL difficulty of horizontal scaling. Pay attention.**

> "Opening a second restaurant sounds simple. But think: if a customer walks into Location A, places an order, then drives to Location B to pick it up — does Location B know about that order?"

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE STATE PROBLEM                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "State" = data that a server remembers between requests.       │
│                                                                 │
│  STATEFUL SERVER (your Week 3 in-memory Task Manager):          │
│  ──────────────────────────────────────────────────             │
│  tasks = {}  ← Data lives IN the server's memory               │
│                                                                 │
│  Request 1 → Server A → tasks["1"] = "Buy milk" ✅             │
│  Request 2 → Server B → tasks["1"] = ???          ❌ NOT HERE!  │
│                                                                 │
│  Server B has its OWN empty dict. It doesn't know about         │
│  the task created on Server A.                                  │
│                                                                 │
│                                                                 │
│  STATELESS SERVER (your Week 6+ database-backed app):           │
│  ─────────────────────────────────────────────────              │
│  Each request is self-contained. Server stores NOTHING locally. │
│  All persistent data lives in the database.                     │
│                                                                 │
│  Request 1 → Server A → writes to PostgreSQL     ✅             │
│  Request 2 → Server B → reads from PostgreSQL    ✅ FOUND IT!  │
│                                                                 │
│  Both servers talk to the SAME database.                        │
│  The server is just a "compute pipe" — data flows through it.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "Why did we switch from an in-memory dict in Week 3 to PostgreSQL in Week 6? Back then we said it was for persistence and data integrity. But here's the second reason you didn't know about yet: *stateless servers can be horizontally scaled.* That migration wasn't just about databases — it was about scalability."

**Connection to Week 9:**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHY YOU CHOSE JWT (AND DIDN'T KNOW IT)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SESSION-BASED AUTH (if you had chosen it):                     │
│  ──────────────────────────────────────────                     │
│                                                                 │
│  Login → Server A creates session, stores in memory             │
│  Next request → Load balancer sends to Server B                 │
│  Server B: "Who are you? I have no session for you." ❌         │
│                                                                 │
│  SOLUTIONS:                                                     │
│  ├─ Sticky sessions (always route to same server — fragile)     │
│  └─ Session store in Redis (extra infrastructure — viable)      │
│                                                                 │
│                                                                 │
│  JWT AUTH (what you built in Week 9):                            │
│  ────────────────────────────────────                           │
│                                                                 │
│  Login → Server A creates JWT, sends to client                  │
│  Next request → Client sends JWT → load balancer → Server B     │
│  Server B: Validates JWT signature. No server state needed. ✅  │
│                                                                 │
│  The token carries the identity. ANY server can verify it.      │
│  This is why JWT is the standard for horizontally scaled APIs.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The statelessness checklist for your capstone:**

```
┌─────────────────────────────────────────────────────────────────┐
│          IS YOUR CAPSTONE HORIZONTALLY SCALABLE?                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Auth via JWT (stateless — Week 9)                            │
│  ✅ Data in PostgreSQL (shared external state — Week 6)          │
│  ✅ Cache in Redis (shared external state — Week 10)             │
│  ✅ Background jobs via Celery+Redis (shared queue — Week 11)    │
│  ✅ Config in environment variables (12-factor — Week 15)        │
│                                                                 │
│  ⚠️  WebSocket connections: connection manager holds state       │
│     in memory (your ConnectionManager from Week 12).            │
│     This BREAKS with multiple servers. Solution: Redis pub/sub  │
│     (you learned this in Week 12 Lecture 2).                    │
│                                                                 │
│  ⚠️  File uploads: if you save to local disk, only that server  │
│     has the file. Solution: S3-compatible storage               │
│     (you learned about presigned URLs in Week 13).              │
│                                                                 │
│  VERDICT: Your capstone is MOSTLY ready for horizontal          │
│           scaling. The decisions you made over 15 weeks were     │
│           the right decisions. You just didn't know WHY yet.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Load Balancers (The Central Reservation System)

**If you have 3 servers, who decides which server gets which request?**

```
┌─────────────────────────────────────────────────────────────────┐
│                     LOAD BALANCER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A load balancer sits BETWEEN clients and your servers.         │
│  It distributes incoming requests across multiple servers.      │
│                                                                 │
│                                                                 │
│  WITHOUT LOAD BALANCER:             WITH LOAD BALANCER:         │
│  ──────────────────────             ────────────────────        │
│                                                                 │
│   Client ──▶ Server 1 😰 (all        Client                    │
│   Client ──▶ Server 1    traffic      Client ──▶ ┌──────┐      │
│   Client ──▶ Server 1    to one       Client ──▶ │  LB  │      │
│                server)    Client ──▶ └──┬───┘      │
│              Server 2 😴 (idle)             │   │   │           │
│              Server 3 😴 (idle)             ▼   ▼   ▼           │
│                                           S1   S2   S3         │
│                                           😊   😊   😊          │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  A central reservation hotline. You call one number.            │
│  The operator checks which location has available tables        │
│  and routes you there. You never pick the server yourself.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What a load balancer does:**

```
┌─────────────────────────────────────────────────────────────────┐
│               LOAD BALANCER RESPONSIBILITIES                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      ┌──────────────────┐                       │
│      Client ────────▶│  LOAD BALANCER   │                       │
│      Request         │                  │                       │
│                      │  1. Receive      │                       │
│                      │  2. Choose server│                       │
│                      │  3. Forward      │                       │
│                      │  4. Return       │                       │
│                      │     response     │                       │
│                      └───┬──────┬───┬───┘                       │
│                          │      │   │                           │
│                          ▼      ▼   ▼                           │
│                        S1      S2   S3                          │
│                                                                 │
│  CORE FUNCTIONS:                                                │
│  ├─ DISTRIBUTE traffic across healthy servers                   │
│  ├─ DETECT failed servers (health checks)                       │
│  ├─ REMOVE failed servers from the pool                         │
│  ├─ RE-ADD recovered servers automatically                      │
│  └─ TERMINATE SSL (handle HTTPS, pass HTTP to servers)          │
│                                                                 │
│  The client talks to ONE address. It never knows there are      │
│  multiple servers behind it. The load balancer is invisible.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Layer 4 vs Layer 7 — two types of load balancing:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TWO TYPES OF LOAD BALANCING                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER 4 (Transport Layer)                                      │
│  ─────────────────────────                                      │
│  Routes based on: IP address and port                           │
│  Knows about: TCP/UDP connections                               │
│  Cannot see: HTTP headers, URLs, cookies                        │
│  Speed: Very fast (just forwarding packets)                     │
│  Analogy: A receptionist who sends you to "the next available   │
│           restaurant" without knowing what food you want.       │
│                                                                 │
│                                                                 │
│  LAYER 7 (Application Layer)                                    │
│  ──────────────────────────                                     │
│  Routes based on: URL path, headers, cookies, request body      │
│  Knows about: HTTP content (can read your request)              │
│  Can do: Route /api/* to API servers, /static/* to CDN          │
│  Speed: Slightly slower (must parse HTTP)                       │
│  Analogy: A host who reads your reservation and sends you       │
│           to the Italian section or the sushi bar accordingly.  │
│                                                                 │
│                                                                 │
│  FOR YOUR FASTAPI APP: Layer 7 is most common.                  │
│  It lets you route by URL, inspect headers, handle              │
│  WebSocket upgrades separately from HTTP requests.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 Load Balancing Algorithms

**How does the load balancer CHOOSE which server gets the request?**

```
┌─────────────────────────────────────────────────────────────────┐
│               LOAD BALANCING ALGORITHMS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  1. ROUND ROBIN                                                 │
│  ──────────────                                                 │
│  Take turns. Request 1 → S1, Request 2 → S2,                   │
│  Request 3 → S3, Request 4 → S1, ...                            │
│                                                                 │
│  Request:  1    2    3    4    5    6                            │
│  Server:  S1   S2   S3   S1   S2   S3                           │
│                                                                 │
│  ✅ Simple, fair         ❌ Ignores server load                  │
│                             (S1 might be handling a heavy       │
│                              query while S2 is idle)            │
│                                                                 │
│  Analogy: Customers line up, alternating between                │
│           Location A, B, C regardless of how busy each is.      │
│                                                                 │
│                                                                 │
│  2. LEAST CONNECTIONS                                           │
│  ────────────────────                                           │
│  Send to the server with the fewest active connections.         │
│                                                                 │
│  S1: 12 connections                                             │
│  S2: 3 connections     ← Next request goes HERE                 │
│  S3: 8 connections                                              │
│                                                                 │
│  ✅ Adapts to actual load  ❌ Needs connection tracking          │
│  ✅ Good for varied         (slightly more complex)             │
│     request durations                                           │
│                                                                 │
│  Analogy: The reservation system checks which restaurant        │
│           has the shortest wait and sends you there.            │
│                                                                 │
│                                                                 │
│  3. IP HASH                                                     │
│  ─────────                                                      │
│  Hash the client's IP to determine server.                      │
│  Same client → same server (always).                            │
│                                                                 │
│  hash("192.168.1.1") % 3 = 1  → Always Server 2                │
│  hash("10.0.0.5") % 3 = 0     → Always Server 1                │
│                                                                 │
│  ✅ Session affinity       ❌ Uneven distribution if             │
│     (sticky sessions)        few clients send lots of traffic   │
│                                                                 │
│  Analogy: "You always go to the location closest to your        │
│           house." Predictable, but some locations get crowded.  │
│                                                                 │
│                                                                 │
│  4. WEIGHTED ROUND ROBIN                                        │
│  ───────────────────────                                        │
│  Like round robin, but some servers get MORE traffic.           │
│                                                                 │
│  S1 (weight 5): gets 5 out of every 8 requests                  │
│  S2 (weight 2): gets 2 out of every 8 requests                  │
│  S3 (weight 1): gets 1 out of every 8 requests                  │
│                                                                 │
│  ✅ When servers have different capacities                       │
│  ✅ Gradual rollout (send 10% to new version)                    │
│                                                                 │
│  Analogy: The flagship location has 100 tables.                 │
│           The new small location has 20. Route accordingly.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Which algorithm should you pick?**

```
┌─────────────────────────────────────────────────────────────────┐
│                 ALGORITHM DECISION GUIDE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Stateless API (your capstone):      → ROUND ROBIN or           │
│                                        LEAST CONNECTIONS        │
│                                                                 │
│  WebSocket connections (long-lived): → LEAST CONNECTIONS        │
│    (some servers accumulate more                                │
│     open sockets over time)                                     │
│                                                                 │
│  Mixed server sizes:                 → WEIGHTED ROUND ROBIN     │
│                                                                 │
│  Need same-client → same-server:     → IP HASH                  │
│    (only if absolutely necessary;                               │
│     prefer stateless design instead)                            │
│                                                                 │
│  DEFAULT CHOICE: LEAST CONNECTIONS.                             │
│  It adapts to real-world conditions. Start there.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.6 Health Checks (Is This Location Open?)

**A load balancer needs to know which servers are alive.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      HEALTH CHECKS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The load balancer periodically asks each server:               │
│  "Are you alive and ready to handle requests?"                  │
│                                                                 │
│                                                                 │
│  LB ──GET /health──▶ Server 1 ──▶ 200 OK ✅  (keep in pool)    │
│  LB ──GET /health──▶ Server 2 ──▶ 200 OK ✅  (keep in pool)    │
│  LB ──GET /health──▶ Server 3 ──▶ TIMEOUT ❌  (remove!)        │
│                                                                 │
│  After removal:                                                 │
│  LB routes traffic to Server 1 and 2 only.                      │
│  Periodically retries Server 3.                                 │
│  If Server 3 recovers → add it back.                            │
│                                                                 │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  The franchise manager calls each location every 30 seconds.    │
│  "Are you open? Is the kitchen working?"                        │
│  If a location doesn't answer 3 times → stop sending customers. │
│  Keep calling. When they answer again → resume sending.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to Week 15:**

> "You already built health check endpoints. Remember `/health` and `/ready` from Week 15? Now you see exactly where those get used — the load balancer hits them continuously. The `/health` endpoint you wrote isn't decorative. It's critical infrastructure."

**Two types of health checks:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  LIVENESS CHECK: /health                                        │
│  ─────────────────────                                          │
│  "Is the process running?"                                      │
│  Returns 200 if the FastAPI process is alive.                   │
│  Doesn't check dependencies.                                   │
│  If this fails → server is crashed, restart it.                 │
│                                                                 │
│                                                                 │
│  READINESS CHECK: /ready                                        │
│  ──────────────────────                                         │
│  "Can you handle requests right now?"                           │
│  Checks: database connection? Redis connection? Disk space?     │
│  If this fails → server is alive but NOT ready.                 │
│  Don't restart, don't send traffic. Wait for recovery.          │
│                                                                 │
│                                                                 │
│  WHY BOTH?                                                      │
│  ──────────                                                     │
│  Server is live but not ready: database migration running.      │
│  Don't kill the server (it's working!). Don't send it traffic   │
│  (it can't serve users yet). These are different signals.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: SCALING YOUR DATABASE

## 3.1 The Database Bottleneck

**Scaling app servers is relatively easy — they're stateless. The database is the hard part.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE DATABASE BOTTLENECK                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  You can add 10 FastAPI servers easily:                         │
│                                                                 │
│   S1  S2  S3  S4  S5  S6  S7  S8  S9  S10                      │
│    \   \   \   |   |   |   /   /   /   /                        │
│     \   \   \  |   |   |  /   /   /   /                         │
│      ▼   ▼   ▼ ▼   ▼   ▼ ▼   ▼   ▼   ▼                         │
│     ┌──────────────────────────────────────┐                    │
│     │         ONE PostgreSQL               │ ← ALL traffic     │
│     │         (sweating)          😰       │    funnels here    │
│     └──────────────────────────────────────┘                    │
│                                                                 │
│  More app servers = MORE database connections.                  │
│  10 servers × 20 connections each = 200 connections.            │
│  All reading and writing to the same PostgreSQL.                │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  You opened 10 restaurant locations. But they all share ONE     │
│  kitchen. The kitchen becomes the bottleneck. Waiters are       │
│  fast, but everyone's waiting for the same oven.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You scaled the easy part (stateless servers). But the database is the *shared state* that every server depends on. You can't just 'add more databases' the way you add more app servers — because the data must be consistent. This is the fundamental challenge of distributed systems."

**A key observation:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Most web applications have a READ-HEAVY workload:              │
│                                                                 │
│  ┌──────────────────────────────────────────┐                   │
│  │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░▓▓▓▓▓│                   │
│  │          READS (~85-95%)           WRITES│                   │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
│  Think about your capstone:                                     │
│  ├─ Loading the dashboard:       READ                           │
│  ├─ Viewing task details:        READ                           │
│  ├─ Listing team members:        READ                           │
│  ├─ Searching tasks:             READ                           │
│  ├─ Viewing audit log:           READ                           │
│  ├─ Creating a task:             WRITE                          │
│  ├─ Updating task status:        WRITE                          │
│  └─ Assigning a member:          WRITE                          │
│                                                                 │
│  Most users spend most of their time LOOKING, not CREATING.     │
│  This asymmetry is the key to database scaling strategy.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Read Replicas (Copies of the Recipe Book)

**If reads are the bottleneck, make more copies for reading.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     READ REPLICAS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                   ┌────────────────────┐                        │
│   ALL WRITES ──▶  │  PRIMARY DATABASE  │                        │
│                   │  (read + write)    │                        │
│                   └───┬──────────┬─────┘                        │
│                       │replicates│                              │
│                       │  data    │                              │
│               ┌───────▼──┐  ┌───▼────────┐                     │
│   READS ───▶  │ REPLICA 1│  │ REPLICA 2  │  ◀─── READS        │
│               │ (read    │  │ (read      │                     │
│               │  only)   │  │  only)     │                     │
│               └──────────┘  └────────────┘                     │
│                                                                 │
│  HOW IT WORKS:                                                  │
│  ├─ One PRIMARY database handles all WRITES                     │
│  ├─ REPLICAS are exact copies of the primary                    │
│  ├─ Data streams from primary → replicas automatically          │
│  ├─ Reads are distributed across replicas                       │
│  └─ If primary dies → promote a replica to primary              │
│                                                                 │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  Headquarters (primary) maintains the master recipe book.       │
│  Each location (replica) gets a copy.                           │
│  When customers ask "What's on the menu?" → any location can    │
│  answer (read from their copy).                                 │
│  When a NEW recipe is invented → it goes to HQ first (write     │
│  to primary), then gets distributed to all locations.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The impact on throughput:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  BEFORE (single database):                                      │
│  ─────────────────────────                                      │
│  All 10,000 read queries/sec → ONE database                     │
│  All 500 write queries/sec → ONE database                       │
│  Total load: 10,500 queries/sec on ONE machine                  │
│                                                                 │
│                                                                 │
│  AFTER (1 primary + 2 replicas):                                │
│  ────────────────────────────────                               │
│  500 write queries/sec → PRIMARY only                           │
│  10,000 read queries/sec → split across 2 REPLICAS              │
│    Replica 1: ~5,000 reads/sec                                  │
│    Replica 2: ~5,000 reads/sec                                  │
│                                                                 │
│  Each machine handles ~5,000-5,500 queries/sec instead of       │
│  10,500. That's a ~50% reduction per machine.                   │
│                                                                 │
│  Add more replicas → further reduce per-machine load.           │
│  Read capacity scales near-linearly with replicas.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to your SQLAlchemy code (Week 6):**

> "In practice, your repository layer from Week 6 would route queries: write operations go to the primary's connection string, read operations go to a replica's connection string. The SQLAlchemy `Session` can be configured with separate `bind` settings for reads vs. writes. Your code doesn't change much — the routing logic lives in the infrastructure layer."

```python
# Conceptual — how your Week 6 async sessions would adapt:

# Primary database (writes)
primary_engine = create_async_engine("postgresql+asyncpg://primary-host/mydb")

# Replica database (reads)
replica_engine = create_async_engine("postgresql+asyncpg://replica-host/mydb")

# In your FastAPI dependency (like your Week 6 session dependency):
async def get_read_session():
    """Use replica for read-heavy endpoints"""
    async with AsyncSession(replica_engine) as session:
        yield session

async def get_write_session():
    """Use primary for write operations"""
    async with AsyncSession(primary_engine) as session:
        yield session
```

---

## 3.3 Replication Lag (The Update Delay)

**Replicas are NOT instant. This creates real problems.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   REPLICATION LAG                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Time ──────────────────────────────────────────────▶           │
│                                                                 │
│  t=0ms     User creates a task (writes to PRIMARY)              │
│  t=1ms     PRIMARY confirms: "Task created!"                    │
│  t=2ms     User's page refreshes, reads from REPLICA            │
│  t=2ms     REPLICA hasn't received the new task yet...          │
│  t=2ms     User sees: "Where's my task?!"  😱                   │
│  t=50ms    REPLICA finally gets the data                        │
│  t=100ms   User refreshes again. "Oh, there it is."            │
│                                                                 │
│                                                                 │
│  This 50ms gap is REPLICATION LAG.                              │
│  Usually milliseconds. Can spike to seconds under heavy load.   │
│                                                                 │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  HQ invents a new recipe at 2:00 PM.                            │
│  They fax it to all locations. (Yes, fax. It's an analogy.)     │
│  Location B gets it at 2:03 PM.                                 │
│  A customer walks into Location B at 2:01 PM and orders the     │
│  new dish. "Sorry, we don't have that." But HQ says they do!   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Strategies for dealing with replication lag:**

```
┌─────────────────────────────────────────────────────────────────┐
│             HANDLING REPLICATION LAG                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. READ-YOUR-OWN-WRITES                                        │
│  ───────────────────────                                        │
│  After a write, route THAT USER's next reads to the primary     │
│  for a short window (e.g., 5 seconds). Other users still        │
│  read from replicas.                                            │
│                                                                 │
│    User writes task → next 5 sec reads go to PRIMARY            │
│    After 5 sec → back to REPLICA (lag has caught up)            │
│                                                                 │
│                                                                 │
│  2. SYNCHRONOUS REPLICATION (for critical data)                 │
│  ──────────────────────────                                     │
│  Primary waits for at least one replica to confirm before       │
│  telling the client "write successful."                         │
│                                                                 │
│    ✅ No lag (replica is always up-to-date)                      │
│    ❌ Slower writes (must wait for replica confirmation)         │
│    Use for: Financial data, auth changes, critical state        │
│                                                                 │
│                                                                 │
│  3. TOLERATE IT                                                 │
│  ─────────────                                                  │
│  For many features, a few hundred milliseconds of lag is        │
│  invisible to users. Dashboard analytics, activity feeds,       │
│  search results — all fine with slight staleness.               │
│                                                                 │
│  Not everything needs perfect consistency.                      │
│  (We'll formalize this idea in the CAP theorem section.)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Sharding (Splitting the Kitchen)

**Read replicas handle the *read* bottleneck. What about writes?**

> "You can't add write capacity with replicas. ALL writes still go to one primary. If you have 50,000 writes/sec and one PostgreSQL can handle 5,000 — replicas don't help. You need to split the data itself."

```
┌─────────────────────────────────────────────────────────────────┐
│                       SHARDING                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sharding = splitting your data across MULTIPLE databases,      │
│  each holding a SUBSET of the total data.                       │
│                                                                 │
│                                                                 │
│  BEFORE (one database, all data):                               │
│                                                                 │
│  ┌──────────────────────────────────┐                           │
│  │          PostgreSQL              │                           │
│  │  Users A-Z, Tasks 1-50M         │                           │
│  │  Organizations 1-100K            │ ← Everything, one DB     │
│  └──────────────────────────────────┘                           │
│                                                                 │
│                                                                 │
│  AFTER (sharded by organization):                               │
│                                                                 │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │   Shard 1   │ │   Shard 2   │ │   Shard 3   │               │
│  │  Orgs 1-33K │ │ Orgs 33-66K │ │ Orgs 66-100K│               │
│  │  + their    │ │  + their    │ │  + their    │               │
│  │  users,     │ │  users,     │ │  users,     │               │
│  │  tasks,     │ │  tasks,     │ │  tasks,     │               │
│  │  audit logs │ │  audit logs │ │  audit logs │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
│                                                                 │
│  Each shard handles 1/3 of the writes.                          │
│  Each shard has 1/3 of the data (smaller, faster indexes).      │
│                                                                 │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  Instead of one massive kitchen cooking everything, you         │
│  create specialized kitchens:                                   │
│  Kitchen A handles customers A-H (their orders, their tabs)     │
│  Kitchen B handles customers I-P                                │
│  Kitchen C handles customers Q-Z                                │
│  Each kitchen is smaller, faster, independent.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Sharding Strategies

**How do you decide which data goes to which shard?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SHARDING STRATEGIES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  1. KEY-BASED (HASH) SHARDING                                   │
│  ────────────────────────────                                   │
│  shard = hash(shard_key) % number_of_shards                     │
│                                                                 │
│  Example with org_id:                                           │
│  hash(org_42) % 3 = 0  → Shard 1                               │
│  hash(org_99) % 3 = 2  → Shard 3                               │
│  hash(org_71) % 3 = 1  → Shard 2                               │
│                                                                 │
│  ✅ Even distribution (if hash is good)                          │
│  ❌ Adding a shard requires rehashing (data migration)           │
│     hash(org_42) % 4 ≠ hash(org_42) % 3                        │
│     → Data must move between shards                             │
│                                                                 │
│                                                                 │
│  2. RANGE-BASED SHARDING                                        │
│  ────────────────────────                                       │
│  Shard 1: org_id 1 – 33,000                                     │
│  Shard 2: org_id 33,001 – 66,000                                │
│  Shard 3: org_id 66,001 – 100,000                               │
│                                                                 │
│  ✅ Simple to understand, easy range queries                     │
│  ❌ Hot spots: if new users get sequential IDs, Shard 3          │
│     gets ALL new writes. Older shards idle.                     │
│                                                                 │
│                                                                 │
│  3. DIRECTORY-BASED SHARDING                                    │
│  ────────────────────────────                                   │
│  A lookup table maps each key to its shard.                     │
│                                                                 │
│  org_42 → Shard 1                                               │
│  org_99 → Shard 2                                               │
│  org_71 → Shard 1   (custom assignment)                         │
│                                                                 │
│  ✅ Maximum flexibility (can rebalance individual keys)          │
│  ❌ Lookup table becomes its own bottleneck/SPOF                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The danger of cross-shard queries:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CROSS-SHARD QUERIES (THE PAIN)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHIN ONE SHARD — everything is easy:                         │
│  ──────────────────────────────────────                         │
│  "Get all tasks for org_42"                                     │
│  → org_42 is on Shard 1 → query Shard 1 → done.                │
│                                                                 │
│  ACROSS SHARDS — everything is hard:                            │
│  ────────────────────────────────────                           │
│  "Get the top 10 most active organizations across ALL orgs"     │
│  → Must query ALL shards                                        │
│  → Combine results in the application layer                     │
│  → No database-level JOIN across shards                         │
│  → Pagination becomes a nightmare                               │
│                                                                 │
│  This is why SHARD KEY CHOICE is the most important decision.   │
│  You want queries to hit ONE shard whenever possible.           │
│                                                                 │
│                                                                 │
│  FOR YOUR MULTI-TENANT CAPSTONE:                                │
│  ──────────────────────────────                                 │
│  Shard key = organization_id  ← NATURAL FIT                    │
│                                                                 │
│  Why? Your multi-tenant design (Week 13) already isolates       │
│  data by organization. Every query already filters by org.      │
│  Users, tasks, projects, audit logs — all scoped to an org.     │
│  Sharding by org_id means almost every query hits one shard.    │
│                                                                 │
│  This is not a coincidence. Multi-tenant design and sharding    │
│  strategy are deeply connected. Your architecture was already   │
│  shard-friendly and you didn't know it.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.6 The Decision Framework

**Don't shard prematurely. Follow this sequence:**

```
┌─────────────────────────────────────────────────────────────────┐
│            DATABASE SCALING DECISION SEQUENCE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  START HERE                                                     │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────┐                            │
│  │  1. OPTIMIZE QUERIES FIRST     │  ← You learned this        │
│  │     Indexes, EXPLAIN ANALYZE,  │    in Week 7.               │
│  │     fix N+1, optimize JOINs    │    Always start here.       │
│  └───────────────┬─────────────────┘                            │
│                  │ Still not enough?                             │
│                  ▼                                               │
│  ┌─────────────────────────────────┐                            │
│  │  2. VERTICAL SCALE THE DB      │  ← Bigger machine.         │
│  │     More RAM, faster disk,     │    Cheapest fix.            │
│  │     more connections           │                             │
│  └───────────────┬─────────────────┘                            │
│                  │ Still not enough?                             │
│                  ▼                                               │
│  ┌─────────────────────────────────┐                            │
│  │  3. ADD CACHING LAYER          │  ← You built this          │
│  │     Redis for hot data,        │    in Week 10.              │
│  │     reduce DB read load        │    Huge impact.             │
│  └───────────────┬─────────────────┘                            │
│                  │ Still not enough?                             │
│                  ▼                                               │
│  ┌─────────────────────────────────┐                            │
│  │  4. ADD READ REPLICAS          │  ← Handles read            │
│  │     Route reads to replicas,   │    bottleneck.              │
│  │     writes to primary          │    Moderate complexity.     │
│  └───────────────┬─────────────────┘                            │
│                  │ Still not enough?                             │
│                  ▼                                               │
│  ┌─────────────────────────────────┐                            │
│  │  5. SHARD THE DATABASE         │  ← Last resort.            │
│  │     Split data across multiple │    Highest complexity.      │
│  │     databases by shard key     │    Hard to undo.            │
│  └─────────────────────────────────┘                            │
│                                                                 │
│  MOST COMPANIES NEVER REACH STEP 5.                             │
│  Steps 1-4 handle millions of users for most applications.      │
│  Sharding is for companies with hundreds of millions of         │
│  rows and thousands of writes per second.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: CACHING AT EVERY LAYER

## 4.1 The Caching Hierarchy

**Caching isn't one thing. It's a CHAIN of caches, each closer to the user.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE CACHING HIERARCHY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│            USER (browser)                                       │
│                │                                                │
│           ┌────▼─────┐                                          │
│     ①     │ BROWSER  │  HTTP cache headers (Cache-Control)      │
│           │ CACHE    │  Static assets, some API responses       │
│           └────┬─────┘                                          │
│                │  cache miss                                    │
│           ┌────▼─────┐                                          │
│     ②     │   CDN    │  Edge servers worldwide                  │
│           │          │  Static files, public API responses      │
│           └────┬─────┘                                          │
│                │  cache miss                                    │
│           ┌────▼─────┐                                          │
│     ③     │   LOAD   │                                          │
│           │ BALANCER │  (routes to app server)                  │
│           └────┬─────┘                                          │
│                │                                                │
│           ┌────▼─────┐                                          │
│     ④     │  APP     │  Redis / in-memory cache                 │
│           │  CACHE   │  Hot data, computed results              │
│           └────┬─────┘                                          │
│                │  cache miss                                    │
│           ┌────▼─────┐                                          │
│     ⑤     │ DATABASE │  Query cache, materialized views         │
│           │  CACHE   │  Internal PostgreSQL buffer pool         │
│           └────┬─────┘                                          │
│                │  cache miss                                    │
│           ┌────▼─────┐                                          │
│     ⑥     │ DATABASE │  Actual disk read                        │
│           │  DISK    │  The source of truth                     │
│           └──────────┘                                          │
│                                                                 │
│  Each layer catches requests BEFORE they reach the next.        │
│  The more requests caught early, the less load on the DB.       │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  ① Customer remembers the menu from last visit (browser)        │
│  ② Menu board outside the building (CDN)                        │
│  ③ Host stand directs you (load balancer)                       │
│  ④ "Today's specials" board inside — waiter answers             │
│     without asking the kitchen (Redis app cache)                │
│  ⑤ Pre-prepped sauce bases in the kitchen (DB cache)            │
│  ⑥ Actually cooking from raw ingredients (DB disk read)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 CDN — Content Delivery Network

```
┌─────────────────────────────────────────────────────────────────┐
│                         CDN                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A CDN is a NETWORK OF SERVERS distributed worldwide.           │
│  They cache and serve content from a location CLOSE to the      │
│  user, instead of going all the way to your origin server.      │
│                                                                 │
│                                                                 │
│  WITHOUT CDN:                                                   │
│  ────────────                                                   │
│  User in Tokyo ─── 12,000 km ───▶ Server in Virginia           │
│  Latency: ~150ms round trip                                     │
│                                                                 │
│                                                                 │
│  WITH CDN:                                                      │
│  ──────────                                                     │
│  User in Tokyo ─── 50 km ───▶ CDN Edge in Tokyo                │
│  Latency: ~5ms round trip                                       │
│                                                                 │
│  The CDN edge has a CACHED COPY of the content.                 │
│  If it doesn't (cache miss), it fetches from your origin        │
│  server, caches it, and serves future requests locally.         │
│                                                                 │
│                                                                 │
│       User        User        User                              │
│      (Tokyo)    (London)    (São Paulo)                         │
│        │           │           │                                │
│        ▼           ▼           ▼                                │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐                          │
│   │CDN Edge │ │CDN Edge │ │CDN Edge │                          │
│   │ Tokyo   │ │ London  │ │São Paulo│                          │
│   └────┬────┘ └────┬────┘ └────┬────┘                          │
│        │           │           │                                │
│        └───────────┼───────────┘                                │
│                    │  (cache miss only)                         │
│                    ▼                                            │
│            ┌──────────────┐                                     │
│            │ Your Server  │                                     │
│            │ (Virginia)   │                                     │
│            └──────────────┘                                     │
│                                                                 │
│                                                                 │
│  WHAT CDNs CACHE WELL:                                          │
│  ├─ Static assets (images, CSS, JS, fonts)                      │
│  ├─ Public API responses that rarely change                     │
│  ├─ Documentation pages, marketing pages                        │
│  └─ Pre-rendered content                                        │
│                                                                 │
│  WHAT CDNs CANNOT CACHE:                                        │
│  ├─ User-specific data (my tasks, my dashboard)                 │
│  ├─ Real-time data (live WebSocket feeds)                       │
│  └─ Write operations (POST, PUT, DELETE)                        │
│                                                                 │
│  Connection to Week 15: Cloud providers (AWS CloudFront,        │
│  GCP Cloud CDN) offer built-in CDN. It's a config switch,      │
│  not a code change.                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Application Cache (Your Redis Knowledge)

**You've already built this layer. Let's see where it fits in the big picture.**

> "In Week 10, you added Redis caching to your project. You implemented cache-aside, TTL, and cache invalidation on write. That work wasn't a standalone feature — it was building one layer of this caching hierarchy."

```
┌─────────────────────────────────────────────────────────────────┐
│        APPLICATION CACHE IN THE BIG PICTURE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR WEEK 10 CACHING:                 ITS ROLE AT SCALE:       │
│  ──────────────────────                ──────────────────       │
│  Cache-aside pattern          →   Reduces DB reads by 80-95%    │
│  TTL-based expiration         →   Balances freshness vs load    │
│  Invalidation on write        →   Prevents stale data           │
│  Redis as FastAPI dependency  →   Shared across all servers     │
│                                                                 │
│                                                                 │
│  WHY THIS MATTERS AT SCALE:                                     │
│  ──────────────────────────                                     │
│                                                                 │
│  Without cache:   10 servers × 1,000 reads/sec = 10,000 DB     │
│                   reads/sec (DB struggles)                       │
│                                                                 │
│  With 90% hit:    10 servers × 1,000 reads/sec → 9,000 from    │
│                   Redis, 1,000 from DB (DB is relaxed)          │
│                                                                 │
│  Redis handles 100,000 ops/sec. Your database handles 10,000.  │
│  The cache absorbs 90% of the load that would crush your DB.   │
│                                                                 │
│  This is why caching comes BEFORE read replicas in the          │
│  decision framework. It's cheaper, simpler, and often enough.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What to cache at the application level (decision guide):**

```
┌─────────────────────────────────────────────────────────────────┐
│          WHAT TO CACHE (DECISION GUIDE)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    How often does it change?                     │
│                    │                                            │
│         Rarely     │             Frequently                     │
│     ◄──────────────┼──────────────────▶                         │
│     │              │                  │                         │
│     │              │                  │                         │
│  ┌──▼───────────┐  │  ┌──────────────▼──┐                      │
│  │ LONG TTL     │  │  │ SHORT TTL       │                      │
│  │ (hours/days) │  │  │ (seconds/mins)  │                      │
│  │              │  │  │                 │                       │
│  │ • App config │  │  │ • Dashboard     │                       │
│  │ • Permission │  │  │   aggregations  │                       │
│  │   matrices   │  │  │ • Search results│                       │
│  │ • Static     │  │  │ • User profile  │                       │
│  │   lookups    │  │  │   data          │                       │
│  │              │  │  │                 │                       │
│  └──────────────┘  │  └─────────────────┘                      │
│                    │                                            │
│            ┌───────▼─────────┐                                  │
│            │  DON'T CACHE    │                                  │
│            │                 │                                  │
│            │ • Write-heavy   │                                  │
│            │   data          │                                  │
│            │ • User-unique   │                                  │
│            │   one-time data │                                  │
│            │ • Security-     │                                  │
│            │   critical data │                                  │
│            │   (balances,    │                                  │
│            │    permissions  │                                  │
│            │    during       │                                  │
│            │    changes)     │                                  │
│            └─────────────────┘                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Database-Level Caching

```
┌─────────────────────────────────────────────────────────────────┐
│                 DATABASE-LEVEL CACHING                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PostgreSQL has its OWN caching — you don't control it          │
│  directly, but you should know it exists.                       │
│                                                                 │
│                                                                 │
│  1. BUFFER POOL (shared_buffers)                                │
│  ────────────────────────────────                               │
│  PostgreSQL keeps frequently accessed data pages in RAM.        │
│  When you query a row, PostgreSQL checks RAM first,             │
│  disk second. This is automatic.                                │
│                                                                 │
│  Tuning: Set shared_buffers to ~25% of server RAM.              │
│  Default is usually too low (128MB).                            │
│                                                                 │
│                                                                 │
│  2. MATERIALIZED VIEWS                                          │
│  ─────────────────────                                          │
│  A pre-computed query result stored as a "frozen table."        │
│  Your dashboard showing "tasks per project" with complex        │
│  JOINs and aggregations? Compute it once, store the result.     │
│                                                                 │
│  CREATE MATERIALIZED VIEW dashboard_stats AS                    │
│    SELECT org_id, COUNT(*) as total_tasks,                      │
│           COUNT(*) FILTER (WHERE status = 'done') as completed  │
│    FROM tasks                                                   │
│    GROUP BY org_id;                                             │
│                                                                 │
│  -- Reads: blazing fast (pre-computed)                          │
│  -- Must REFRESH periodically:                                  │
│  REFRESH MATERIALIZED VIEW CONCURRENTLY dashboard_stats;        │
│                                                                 │
│  This is like the kitchen pre-making soup stock every morning.  │
│  When a customer orders soup, the base is already done.         │
│                                                                 │
│                                                                 │
│  3. CONNECTION-LEVEL: PREPARED STATEMENTS                       │
│  ────────────────────────────────────────                       │
│  PostgreSQL caches the query plan for repeated queries.         │
│  SQLAlchemy uses prepared statements by default with asyncpg.   │
│  You've been benefiting from this since Week 6.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Cache Invalidation (The Hardest Problem)

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**You encountered this in Week 10. At scale, it gets worse.**

```
┌─────────────────────────────────────────────────────────────────┐
│               CACHE INVALIDATION AT SCALE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE PROBLEM:                                                   │
│  ────────────                                                   │
│  Data in the database changes.                                  │
│  Cached copy is now STALE (outdated).                           │
│  How do you keep caches in sync?                                │
│                                                                 │
│                                                                 │
│  SINGLE SERVER (Week 10 — what you did):                        │
│  ──────────────────────────────────────                         │
│  User updates task → delete cache key → next read refills it    │
│  Simple. One Redis, one server, one point of change.            │
│                                                                 │
│                                                                 │
│  MULTIPLE LAYERS (at scale):                                    │
│  ────────────────────────────                                   │
│  User updates task →                                            │
│    Delete from Redis?               ✅ (you control this)       │
│    Invalidate CDN edge caches?      ⚠️  (harder, API call       │
│                                         to CDN provider)        │
│    Clear browser cache?             ❌ (you can't! client-side) │
│    Update read replicas?            ⚠️  (replication lag)       │
│    Refresh materialized views?      ⚠️  (background job)       │
│                                                                 │
│  Every cache layer has its own invalidation challenge.          │
│                                                                 │
│                                                                 │
│  THE RESTAURANT ANALOGY:                                        │
│  The head chef changes a recipe. Now you must:                  │
│  ├─ Update the daily specials board (app cache)         Easy    │
│  ├─ Reprint the menus at this location (DB cache)       Medium  │
│  ├─ Update the outdoor menu board (CDN)                 Medium  │
│  ├─ Ship new menus to all other locations (replicas)    Hard    │
│  └─ Recall every menu photo a customer saved (browser)  LOL     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Practical strategies:**

```
┌─────────────────────────────────────────────────────────────────┐
│            INVALIDATION STRATEGIES (REVIEW)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. TIME-BASED (TTL) — the simplest and most robust             │
│     Set it and forget it. Data may be stale for up to TTL.      │
│     For many use cases, 30-60 seconds of staleness is fine.     │
│                                                                 │
│  2. EVENT-BASED — invalidate when data changes                  │
│     Your Week 10 pattern: write to DB → delete cache key.       │
│     More complex but keeps data fresher.                        │
│     Connection to Week 11: Celery tasks or Redis pub/sub        │
│     can broadcast invalidation events.                          │
│                                                                 │
│  3. VERSIONED KEYS — avoid invalidation entirely                │
│     Instead of cache key "user:42:profile",                     │
│     use "user:42:profile:v7"                                    │
│     When data changes, increment version → old key expires      │
│     naturally via TTL. No explicit invalidation needed.         │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  Start with TTL. Add event-based only for data that MUST        │
│  be fresh within seconds. Accept staleness where you can.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: CAP THEOREM & DESIGN THINKING

## 5.1 The CAP Theorem

**The CAP theorem defines the fundamental limits of distributed systems.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE CAP THEOREM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  In a distributed data store, you can only guarantee            │
│  TWO of these three properties simultaneously:                  │
│                                                                 │
│                    CONSISTENCY                                   │
│                        ╱╲                                       │
│                       ╱  ╲                                      │
│                      ╱    ╲                                     │
│                     ╱ PICK ╲                                    │
│                    ╱  TWO   ╲                                   │
│                   ╱          ╲                                  │
│     AVAILABILITY ╱────────────╲ PARTITION                       │
│                                 TOLERANCE                       │
│                                                                 │
│                                                                 │
│  C — CONSISTENCY                                                │
│  Every read receives the most recent write or an error.         │
│  All nodes see the same data at the same time.                  │
│                                                                 │
│  A — AVAILABILITY                                               │
│  Every request receives a response (not an error),              │
│  without guarantee that it contains the most recent write.      │
│                                                                 │
│  P — PARTITION TOLERANCE                                        │
│  The system continues to operate despite network failures       │
│  between nodes (messages are lost or delayed).                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now here's what most explanations get WRONG:**

```
┌─────────────────────────────────────────────────────────────────┐
│          THE COMMON MISCONCEPTION ABOUT CAP                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ WRONG interpretation:                                        │
│  "Pick 2 of 3. You can have CA, CP, or AP."                     │
│                                                                 │
│  ✅ CORRECT interpretation:                                      │
│  Network partitions WILL happen in any distributed system.      │
│  You can't opt out of P. Networks fail. Cables get cut.         │
│  Data centers lose connectivity.                                │
│                                                                 │
│  So the REAL choice is:                                          │
│  When a partition occurs, do you sacrifice C or A?              │
│                                                                 │
│  ┌─────────────────┐         ┌─────────────────┐               │
│  │ CP SYSTEM       │         │ AP SYSTEM       │               │
│  │                 │         │                 │               │
│  │ During network  │         │ During network  │               │
│  │ partition:      │         │ partition:      │               │
│  │                 │         │                 │               │
│  │ Refuse requests │         │ Serve requests  │               │
│  │ to guarantee    │         │ but data may be │               │
│  │ data is correct │         │ stale/outdated  │               │
│  │                 │         │                 │               │
│  │ "I'd rather     │         │ "I'd rather     │               │
│  │  give you NO    │         │  give you OLD   │               │
│  │  answer than a  │         │  data than NO   │               │
│  │  WRONG answer"  │         │  answer"        │               │
│  └─────────────────┘         └─────────────────┘               │
│                                                                 │
│  When there's NO partition (most of the time), you can          │
│  have BOTH consistency AND availability. CAP only forces         │
│  a choice during failures.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The restaurant chain analogy for CAP:**

```
┌─────────────────────────────────────────────────────────────────┐
│                CAP IN THE RESTAURANT CHAIN                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Consistency = Every location has the exact same menu           │
│  Availability = Every location is always open for business      │
│  Partition = Phone line between HQ and Location B goes down     │
│                                                                 │
│                                                                 │
│  SCENARIO: HQ removes a dish (allergen found).                  │
│            Phone to Location B is down.                         │
│                                                                 │
│                                                                 │
│  CP CHOICE:                                                     │
│  "Location B, CLOSE your kitchen until we can confirm you       │
│   have the updated menu. We can't risk serving that dish."      │
│                                                                 │
│   ✅ No customer gets the dangerous dish (consistent)            │
│   ❌ Location B loses all business until phones work (not        │
│      available)                                                  │
│                                                                 │
│                                                                 │
│  AP CHOICE:                                                     │
│  "Location B, stay open. Serve what you have. We'll update      │
│   the menu as soon as phones are back."                         │
│                                                                 │
│   ✅ Location B still serves customers (available)               │
│   ❌ Some customers might order the removed dish (inconsistent)  │
│                                                                 │
│                                                                 │
│  WHICH IS BETTER? Depends on what you're serving.               │
│  Allergen risk? CP. Minor recipe tweak? AP.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 CP vs AP in Practice

**Systems you know, classified:**

```
┌─────────────────────────────────────────────────────────────────┐
│             REAL SYSTEMS AND THEIR CAP CHOICES                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CP SYSTEMS (Consistency + Partition Tolerance)                  │
│  ──────────────────────────────────────────────                 │
│                                                                 │
│  PostgreSQL (with synchronous replication)                       │
│  ├─ Your capstone's database.                                   │
│  ├─ Refuses to confirm a write until replica has it.            │
│  └─ If replica is unreachable → write BLOCKS or FAILS.          │
│                                                                 │
│  When CP is right:                                              │
│  ├─ Financial transactions (bank transfers)                     │
│  ├─ Inventory management (can't oversell)                       │
│  ├─ User authentication (security-critical)                     │
│  └─ Anything where WRONG data is worse than NO data             │
│                                                                 │
│                                                                 │
│  AP SYSTEMS (Availability + Partition Tolerance)                 │
│  ──────────────────────────────────────────────                 │
│                                                                 │
│  DNS (Domain Name System)                                       │
│  ├─ Returns cached IP addresses even if root servers are down.  │
│  ├─ May serve stale records briefly.                            │
│  └─ Better to serve old IP than refuse to resolve entirely.     │
│                                                                 │
│  Redis (default configuration)                                  │
│  ├─ Your cache layer from Week 10.                              │
│  ├─ In a Redis cluster, reads succeed even during partition.    │
│  └─ Some reads may return stale data briefly.                   │
│                                                                 │
│  When AP is right:                                              │
│  ├─ Social media feeds (slightly stale is fine)                 │
│  ├─ Product catalogs (cached prices, periodic refresh)          │
│  ├─ Analytics dashboards (eventually consistent)                │
│  └─ Anything where STALE data is better than NO data            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to your capstone:**

```
┌─────────────────────────────────────────────────────────────────┐
│          YOUR CAPSTONE'S CAP CHOICES (YOU ALREADY MADE THEM)   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FEATURE                NEEDS           YOUR IMPLEMENTATION     │
│  ───────                ─────           ────────────────────    │
│  User login/auth        Strong C        PostgreSQL (CP)         │
│  Task creation          Strong C        PostgreSQL (CP)         │
│  Dashboard stats        Eventual C      Redis cache + TTL (AP)  │
│  WebSocket notifs       Availability    Redis pub/sub (AP)      │
│  Audit log              Strong C        PostgreSQL (CP)         │
│  Search results         Eventual C      Cached / tolerate lag   │
│                                                                 │
│  You naturally made different CAP choices for different          │
│  features without knowing the formal framework.                 │
│                                                                 │
│  Auth data goes directly to PostgreSQL (you can't risk stale    │
│  permissions). Dashboard data comes from Redis with a TTL       │
│  (a 30-second-old task count is fine for a dashboard).          │
│                                                                 │
│  SYSTEM DESIGN ISN'T PICKING CP OR AP FOR YOUR WHOLE APP.       │
│  IT'S PICKING CP OR AP PER FEATURE, per data path.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Everything is a Tradeoff

**System design is not about finding the "best" architecture. It's about choosing which problems you're willing to have.**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE TRADEOFF MATRIX                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DECISION               GAIN                COST                │
│  ────────               ────                ────                │
│                                                                 │
│  Add caching            Lower latency,      Cache invalidation  │
│  (Redis)                reduced DB load     complexity, stale   │
│                                             data risk           │
│                                                                 │
│  Add read               Read throughput,    Replication lag,     │
│  replicas               redundancy          write still limited, │
│                                             operational cost     │
│                                                                 │
│  Shard the              Write throughput,   Cross-shard queries  │
│  database               smaller indexes     hard, operational    │
│                                             nightmare            │
│                                                                 │
│  Add load               Redundancy,         New failure point,   │
│  balancer               horizontal scale    added latency (ms),  │
│                                             complexity           │
│                                                                 │
│  Use CDN                Global low latency, Cache invalidation,  │
│                         reduced bandwidth   cost per GB, only    │
│                                             helps cacheable data │
│                                                                 │
│  Background             Decoupled, async    Eventually consistent│
│  jobs (Celery)          processing          job failures need    │
│                                             monitoring, debugging│
│                                             is harder            │
│                                                                 │
│  EVERY architectural decision gives you something and costs     │
│  you something. The skill is knowing what your system can       │
│  afford to lose.                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The question to always ask:**

> "When someone proposes an architectural change, ask: 'What gets worse?' If they can't answer, they don't understand the change. Every improvement has a cost. Find it."

---

## 5.4 Full Architecture — Putting It All Together

**This is what your capstone looks like at 10M users. Every component maps to something you've built or learned:**

```
┌─────────────────────────────────────────────────────────────────┐
│          FULL PRODUCTION ARCHITECTURE (10M Users)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     CLIENTS                              │   │
│  │            (browsers, mobile apps)                       │   │
│  └────────────────────────┬─────────────────────────────────┘   │
│                           │                                     │
│                    ┌──────▼──────┐                               │
│                    │     CDN     │ ← Static assets, public data │
│                    └──────┬──────┘   (Section 4.2)              │
│                           │ (dynamic requests pass through)     │
│                    ┌──────▼──────┐                               │
│                    │    LOAD     │ ← Distributes traffic        │
│                    │  BALANCER   │   (Section 2.4)              │
│                    └──┬───┬───┬──┘                               │
│                       │   │   │                                  │
│              ┌────────┘   │   └────────┐                         │
│              ▼            ▼            ▼                          │
│        ┌──────────┐ ┌──────────┐ ┌──────────┐                    │
│        │ FastAPI  │ │ FastAPI  │ │ FastAPI  │ ← Your app ×N     │
│        │ Server 1 │ │ Server 2 │ │ Server 3 │   (Weeks 3-14)    │
│        └──┬───┬───┘ └──┬───┬───┘ └──┬───┬───┘                    │
│           │   │        │   │        │   │                        │
│     ┌─────┘   └──┬─────┘   └──┬─────┘   └────┐                  │
│     ▼            ▼            ▼               ▼                  │
│  ┌───────┐  ┌─────────┐  ┌─────────────┐  ┌────────┐           │
│  │ Redis │  │ Celery   │  │ PostgreSQL  │  │External│           │
│  │ Cache │  │ Workers  │  │             │  │  APIs  │           │
│  │       │  │          │  │  Primary    │  │        │           │
│  │Week 10│  │ Week 11  │  │  ┌──┴──┐   │  │ Week 8 │           │
│  │       │  │          │  │  R1   R2   │  │        │           │
│  └───────┘  └─────┬────┘  │ (replicas) │  └────────┘           │
│                   │       │  Week 5-7  │                        │
│                   ▼       └────────────┘                        │
│              ┌─────────┐                                        │
│              │  Redis   │ ← Broker for Celery                   │
│              │  (Queue) │   + pub/sub for WebSockets             │
│              └─────────┘   (Week 11-12)                         │
│                                                                 │
│                                                                 │
│   Everything in this diagram is something you've built,         │
│   configured, or used. System design isn't new technology —     │
│   it's arranging known pieces at a larger scale.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Walk through a single request:**

```
┌─────────────────────────────────────────────────────────────────┐
│          LIFE OF A REQUEST: "Get my tasks"                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Client sends GET /api/v1/tasks                              │
│     Header: Authorization: Bearer <JWT>                         │
│                                                                 │
│  2. CDN: "This is a dynamic, authenticated request."            │
│     → Pass through (can't cache user-specific data)             │
│                                                                 │
│  3. Load Balancer: "Server 2 has fewest connections."           │
│     → Forward to Server 2                                       │
│                                                                 │
│  4. Server 2 (FastAPI):                                         │
│     a. Validate JWT (stateless — no DB needed) [Week 9]         │
│     b. Check Redis: "tasks:user:42:page:1" → CACHE HIT ✅       │
│     c. Return cached response. Done! (< 10ms)                   │
│                                                                 │
│  4. (alternate) CACHE MISS:                                     │
│     a. Validate JWT                                             │
│     b. Check Redis → MISS                                       │
│     c. Query PostgreSQL read replica [Week 6 repository]        │
│     d. Store result in Redis with 60s TTL [Week 10]             │
│     e. Return response (< 100ms)                                │
│                                                                 │
│                                                                 │
│  LIFE OF A REQUEST: "Create a task"                             │
│  ────────────────────────────────────                           │
│                                                                 │
│  1. Client sends POST /api/v1/tasks with JSON body              │
│  2. CDN: pass through (POST = not cacheable)                    │
│  3. Load Balancer → Server 1                                    │
│  4. Server 1:                                                   │
│     a. Validate JWT [Week 9]                                    │
│     b. Validate body with Pydantic [Week 3]                     │
│     c. Write to PostgreSQL PRIMARY [Week 6]                     │
│     d. Invalidate Redis cache keys [Week 10]                    │
│     e. Dispatch Celery task: send notification [Week 11]        │
│     f. Publish WebSocket event via Redis pub/sub [Week 12]      │
│     g. Return 201 Created                                       │
│                                                                 │
│  EVERY step references something you've implemented.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              SYSTEM DESIGN QUICK REFERENCE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE FRAMEWORK:                                                 │
│      1. Clarify requirements (functional + non-functional)      │
│      2. Estimate scale (users, RPS, storage, bandwidth)         │
│      3. Draw high-level design (boxes and arrows)               │
│      4. Detail critical components                              │
│      5. Identify tradeoffs and bottlenecks                      │
│                                                                 │
│  SCALING:                                                       │
│      Vertical = bigger machine (simple, limited)                │
│      Horizontal = more machines (complex, unlimited)            │
│      Stateless servers + shared external state = scalable       │
│                                                                 │
│  LOAD BALANCING:                                                │
│      Round Robin        → simple, equal distribution            │
│      Least Connections  → best default for most apps            │
│      IP Hash            → when you need sticky sessions         │
│      Weighted           → mixed server sizes or canary deploys  │
│                                                                 │
│  DATABASE SCALING (in order):                                   │
│      1. Optimize queries + indexes                              │
│      2. Scale up the machine                                    │
│      3. Add caching (Redis)                                     │
│      4. Add read replicas                                       │
│      5. Shard (last resort)                                     │
│                                                                 │
│  CACHING LAYERS (closest to user first):                        │
│      Browser → CDN → Load Balancer → App Cache → DB Cache → DB  │
│                                                                 │
│  CAP THEOREM:                                                   │
│      Partitions are inevitable. Choose per feature:             │
│      CP = correct data or error (financial, auth)               │
│      AP = available data, maybe stale (feeds, dashboards)       │
│                                                                 │
│  GOLDEN RULES:                                                  │
│      • Measure before optimizing (Week 12 load tests)           │
│      • Prefer simple over distributed (scale up first)          │
│      • Every gain has a cost (name the tradeoff)                │
│      • Design for the common case, handle the edge case         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SYSTEM DESIGN = SCALING WHAT YOU'VE ALREADY BUILT              │
│                                                                 │
│  You spent 15 weeks building a complete backend system.         │
│  Today you learned that every piece you built has a place       │
│  in a larger architecture.                                      │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐       │
│  │                                                      │       │
│  │  WHAT YOU BUILT           WHERE IT FITS AT SCALE     │       │
│  │  ──────────────           ──────────────────────     │       │
│  │  FastAPI async (W1,3)     Handles 1000s of conns     │       │
│  │  Pydantic (W3)            Validates at every server  │       │
│  │  PostgreSQL (W5-7)        Primary + replicas         │       │
│  │  SQLAlchemy (W6)          Routes reads/writes        │       │
│  │  Alembic (W6)             Migrates all shards        │       │
│  │  JWT auth (W9)            Stateless = scalable       │       │
│  │  Redis cache (W10)        Shared cache layer         │       │
│  │  Celery (W11)             Distributed workers        │       │
│  │  WebSockets (W12)         Scale via Redis pub/sub    │       │
│  │  Load tests (W12)         Prove your design works    │       │
│  │  Docker (W15)             Identical server copies    │       │
│  │  Health checks (W15)      Load balancer integration  │       │
│  │  CI/CD (W15)              Deploy to N servers        │       │
│  │                                                      │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                 │
│  THE RESTAURANT CHAIN:                                          │
│  ├─ One restaurant → chain of restaurants                       │
│  ├─ Bigger restaurant → more locations                          │
│  ├─ Central reservations → load balancer                        │
│  ├─ Master recipe book + copies → primary + replicas            │
│  ├─ Split by cuisine → sharding                                 │
│  ├─ Menu boards → CDN + caches                                  │
│  └─ "Every location, same menu" vs "always open" → CAP         │
│                                                                 │
│  System design isn't about learning new tools.                  │
│  It's about learning to THINK about the tools you already       │
│  know — at a scale where one machine isn't enough.              │
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
│  WEEK 16, LECTURE 2 — Architecture Patterns:                    │
│  └─ Today you learned the PIECES (load balancers, replicas,     │
│     caching, sharding). Next lecture you'll learn how to        │
│     ARRANGE them: monolith vs microservices, event-driven       │
│     architecture, API gateways, designing for failure.          │
│     Today = components. Next = composition.                     │
│                                                                 │
│  WEEK 16, LECTURE 3 — DSA & Interview Prep:                     │
│  └─ The system design framework from today IS the interview     │
│     format. You'll practice applying it to common system        │
│     design problems: "Design a URL shortener", "Design a        │
│     chat app." You'll also learn how to talk about YOUR         │
│     capstone as a system design case study.                     │
│                                                                 │
│  FINAL DELIVERABLE:                                             │
│  └─ Your README's architecture diagram should now show          │
│     WHERE load balancers, caches, and replicas would go         │
│     in a production deployment — not just your Docker           │
│     Compose local setup. Show you understand the path           │
│     from your single-machine capstone to a production           │
│     system. That's what separates a student project from        │
│     professional thinking.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```