# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THEIR CODE IS THE CASE STUDY                                   │
│  ────────────────────────────                                   │
│  Students built a monolithic SaaS platform. They don't need     │
│  imaginary examples. Their capstone IS the starting point.      │
│  Every pattern is shown as an evolution of what they own.       │
│                                                                 │
│  TRADEOFFS OVER "BEST PRACTICES"                                │
│  ───────────────────────────────                                │
│  There is no "right" architecture. Only tradeoffs.              │
│  Every diagram has a cost column. Every pattern has a "when     │
│  NOT to use" section. This is the hardest lesson in software.   │
│                                                                 │
│  SCALE THE PROBLEM BEFORE SCALING THE SOLUTION                  │
│  ─────────────────────────────────────────────                  │
│  Students must feel the PRESSURE that creates each pattern.     │
│  No pattern is introduced without the pain it solves.           │
│                                                                 │
│  CONNECT TO PRIOR WORK                                          │
│  ─────────────────────                                          │
│  Repository pattern (Wk 6) → service boundaries                │
│  Circuit breaker (Wk 8)    → system-level resilience           │
│  Celery + Redis (Wk 11)    → message queues at scale           │
│  Redis Pub/Sub (Wk 11-12)  → event-driven architecture         │
│  Docker Compose (Wk 15)    → independent service deployment    │
│  Load balancers, CAP (Wk 16 L1) → the infrastructure under    │
│                                     these patterns              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                   ARCHITECTURE PATTERNS                         │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE MONOLITH YOU BUILT (35 min)                        │
│  ├─ 1.1 The 3am Incident (Opening Scenario)                    │
│  ├─ 1.2 Anatomy of Your Monolith                                │
│  ├─ 1.3 Why Monoliths Are Good (Seriously)                      │
│  ├─ 1.4 When Monoliths Break (The Pressure Points)              │
│  └─ 1.5 The Company Analogy                                     │
│                                                                 │
│  PART 2: MICROSERVICES — THE DECOMPOSITION (50 min)             │
│  ├─ 2.1 What Microservices Actually Are                         │
│  ├─ 2.2 Finding Service Boundaries                              │
│  ├─ 2.3 Conway's Law (The Uncomfortable Truth)                  │
│  ├─ 2.4 The Hidden Costs (What Nobody Tells You)                │
│  └─ 2.5 The Distributed Monolith Trap                           │
│                                                                 │
│  PART 3: HOW SERVICES COMMUNICATE (50 min)                      │
│  ├─ 3.1 The Communication Problem                               │
│  ├─ 3.2 Synchronous: HTTP Between Services                      │
│  ├─ 3.3 Asynchronous: Message Queues                            │
│  ├─ 3.4 Kafka vs RabbitMQ (Different Tools, Different Jobs)     │
│  ├─ 3.5 Event-Driven Architecture at Scale                      │
│  └─ 3.6 Choosing Your Communication Pattern                     │
│                                                                 │
│  PART 4: THE FRONT DOOR — API GATEWAY (25 min)                  │
│  ├─ 4.1 The Problem: Where Do I Send This Request?              │
│  ├─ 4.2 API Gateway Pattern                                     │
│  ├─ 4.3 Gateway Responsibilities                                │
│  └─ 4.4 Gateway Tradeoffs                                       │
│                                                                 │
│  PART 5: DESIGNING FOR FAILURE (40 min)                         │
│  ├─ 5.1 The Fallacies of Distributed Computing                  │
│  ├─ 5.2 Redundancy (No Single Points of Failure)                │
│  ├─ 5.3 The Bulkhead Pattern (Containing the Blast)             │
│  ├─ 5.4 Fallback Strategies                                     │
│  ├─ 5.5 Graceful Degradation (The Art of Partially Working)     │
│  └─ 5.6 Transactions Across Services: The Saga Pattern          │
│                                                                 │
│  PART 6: SERVICE MESH & THE BIG PICTURE (20 min)                │
│  ├─ 6.1 What is a Service Mesh?                                 │
│  ├─ 6.2 The Architecture Decision Framework                     │
│  └─ 6.3 The One Rule                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE MONOLITH YOU BUILT

## 1.1 The 3am Incident

**Start with a scenario. Make it personal.**

> "Your capstone project is a hit. 10,000 organizations use it daily. Your team has grown to 50 engineers. One night, at 3am, Engineer A pushes a one-line change — a typo fix in the notification email template. The CI passes. The deployment rolls out. Fifteen minutes later, your on-call phone rings."

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE 3AM INCIDENT                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  02:45  Engineer A deploys notification template fix            │
│  02:47  Deployment completes across all instances                │
│  02:51  Monitoring alert: /tasks endpoints returning 500s       │
│  02:53  Monitoring alert: /billing endpoints returning 500s     │
│  02:55  Customer reports: "Can't access anything"               │
│  02:58  Incident declared. Full outage.                         │
│  03:00  You wake up to 47 Slack notifications                   │
│                                                                 │
│  Root cause: The "template fix" deployment included another     │
│  engineer's unfinished database migration that was merged       │
│  to main 20 minutes earlier. Everything deploys together.       │
│  A notification change took down billing and task management.   │
│                                                                 │
│  Damage: 4 hours of total downtime. $200,000 in lost revenue.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "How did a *notification template fix* take down the *billing system*?"

Let them answer. They'll get to it: **Because everything lives in one application. One deployment. One process. One failure domain.**

> "So the answer is obvious, right? We should have used microservices from day one."

Pause.

> "No. Absolutely not. And understanding *why not* is the most important lesson in this entire lecture."

---

## 1.2 Anatomy of Your Monolith

**Let's look at what you actually built:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   YOUR CAPSTONE: A MONOLITH                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│            ┌───────────────────────────────────┐                │
│            │          ONE FastAPI APP           │                │
│            │          ONE PROCESS               │                │
│            │          ONE DEPLOYMENT            │                │
│            ├───────────────────────────────────┤                │
│            │                                   │                │
│            │  ┌─────────┐  ┌──────────────┐    │                │
│            │  │  Auth   │  │    Tasks     │    │                │
│            │  │ Router  │  │   Router     │    │                │
│            │  └────┬────┘  └──────┬───────┘    │                │
│            │       │              │             │                │
│            │  ┌────┴────┐  ┌──────┴───────┐    │                │
│            │  │  Auth   │  │    Task      │    │                │
│            │  │ Service │  │   Service    │    │                │
│            │  └────┬────┘  └──────┬───────┘    │                │
│            │       │              │             │                │
│            │  ┌────┴────┐  ┌──────┴───────┐    │                │
│            │  │  User   │  │    Task      │    │                │
│            │  │  Repo   │  │    Repo      │    │                │
│            │  └────┬────┘  └──────┬───────┘    │                │
│            │       │              │             │                │
│            │       └──────┬───────┘             │                │
│            │              │                     │                │
│            │     ┌────────┴─────────┐           │                │
│            │     │   ONE Database   │           │                │
│            │     │   (PostgreSQL)   │           │                │
│            │     └──────────────────┘           │                │
│            │                                   │                │
│            │  + Celery workers (same codebase) │                │
│            │  + Redis (cache + broker)          │                │
│            │  + WebSocket connections           │                │
│            │                                   │                │
│            └───────────────────────────────────┘                │
│                                                                 │
│  ALL of this deploys as ONE unit.                               │
│  ALL of this scales as ONE unit.                                │
│  ALL of this fails as ONE unit.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Look at your project structure:**

```python
# main.py — The single entry point to EVERYTHING
from fastapi import FastAPI
from app.routers import auth, tasks, organizations, notifications, billing

app = FastAPI(title="SaaS Platform")

# Every domain lives in ONE application
app.include_router(auth.router, prefix="/auth")
app.include_router(tasks.router, prefix="/tasks")
app.include_router(organizations.router, prefix="/orgs")
app.include_router(notifications.router, prefix="/notifications")
app.include_router(billing.router, prefix="/billing")

# One database connection pool shared by ALL domains
# One Redis connection shared by ALL domains
# One Celery instance processing ALL background jobs
```

> "This is a monolith. Every feature, every domain, every background job ships inside the same artifact. Your Docker image contains ALL of it."

**But notice something important — you already have boundaries.**

> "Your repository pattern from Week 6, your service layer, your separate routers — those ARE logical boundaries. They just happen to run inside one process. Remember that. It matters later."

---

## 1.3 Why Monoliths Are Good (Seriously)

**This section exists because the industry has a bias problem.**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY MONOLITHS ARE ACTUALLY GREAT                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. SIMPLE TO DEVELOP                                           │
│     ─────────────────                                           │
│     One codebase. One repo. One IDE window.                     │
│     Find all references. Refactor globally.                     │
│     You've been doing this for 15 weeks. Was it painful?        │
│                                                                 │
│  2. SIMPLE TO TEST                                              │
│     ───────────────                                             │
│     Your test suite runs against ONE application.               │
│     Integration tests call real endpoints. No network mocks     │
│     between services. Your 80%+ coverage? Easy in a monolith.   │
│                                                                 │
│  3. SIMPLE TO DEPLOY                                            │
│     ────────────────                                            │
│     One Docker image. One CI pipeline. One rollback command.    │
│     Your GitHub Actions workflow builds and ships ONE thing.    │
│                                                                 │
│  4. SIMPLE TO DEBUG                                             │
│     ────────────────                                            │
│     Stack trace goes from router → service → repo → database.  │
│     All in one process. One set of logs. One Sentry project.   │
│     No tracing requests across 8 services at 3am.              │
│                                                                 │
│  5. NO NETWORK BOUNDARY BETWEEN COMPONENTS                     │
│     ─────────────────────────────────────                       │
│     Task service calls Auth service? That's a function call.    │
│     Nanoseconds. In-process. Cannot fail due to network.        │
│     Cannot timeout. Cannot return a garbled response.           │
│                                                                 │
│  6. TRANSACTIONS ARE TRIVIAL                                    │
│     ────────────────────────                                    │
│     Remember BEGIN/COMMIT/ROLLBACK from Week 5?                 │
│     One database. One transaction wraps everything.             │
│     "Create user AND create their org" — atomic. Done.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Your capstone project took 2 weeks to build. A comparable microservices setup would take 2 months. You'd still be configuring service discovery instead of shipping features."

---

## 1.4 When Monoliths Break (The Pressure Points)

**Monoliths don't break because of technology. They break because of GROWTH.**

```
┌─────────────────────────────────────────────────────────────────┐
│             WHEN MONOLITHS START TO CRACK                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PRESSURE 1: TEAM SCALING                                       │
│  ────────────────────────                                       │
│                                                                 │
│    5 engineers?   Monolith is fine. Talk across the table.      │
│   15 engineers?   Getting crowded. Merge conflicts daily.       │
│   50 engineers?   Deployments queue up. Teams block each other. │
│  200 engineers?   Monolith is a warzone.                        │
│                                                                 │
│  The 3am incident: Engineer A deployed Engineer B's unfinished  │
│  migration. In a monolith, EVERYONE deploys EVERYTHING.         │
│                                                                 │
│                                                                 │
│  PRESSURE 2: INDEPENDENT SCALING                                │
│  ────────────────────────────                                   │
│                                                                 │
│  Your task search is CPU-heavy (full-text search).              │
│  Your notification system is I/O-heavy (sending emails).        │
│  Your auth system is called 100x more than billing.             │
│                                                                 │
│  In a monolith, you scale EVERYTHING to handle the busiest      │
│  part. 90% of your servers are overpowered for 90% of their    │
│  work.                                                          │
│                                                                 │
│                                                                 │
│  PRESSURE 3: TECHNOLOGY LOCK-IN                                 │
│  ────────────────────────────                                   │
│                                                                 │
│  Your monolith is Python/FastAPI. Turns out your search         │
│  feature would be 10x faster in Go. Your ML recommendation     │
│  engine wants to use a different Python version with GPU libs.  │
│                                                                 │
│  In a monolith: tough luck. Everything uses the same runtime.   │
│                                                                 │
│                                                                 │
│  PRESSURE 4: FAULT ISOLATION                                    │
│  ──────────────────────────                                     │
│                                                                 │
│  A memory leak in the notification renderer crashes the         │
│  process. The entire application goes down. Tasks, billing,     │
│  auth — all dead. Because one component ran out of memory.      │
│                                                                 │
│  In a monolith: one bad component kills everything.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "Looking at your capstone project — which of these four pressures would hit you FIRST if your platform grew to 10,000 users and 50 engineers?"

Let them discuss. The answer is almost always **team scaling**. Technical limits come later than people think.

---

## 1.5 The Company Analogy

**This analogy carries through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE COMPANY ANALOGY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR MONOLITH = A 5-PERSON STARTUP IN A GARAGE                │
│  ──────────────────────────────────────────────                 │
│                                                                 │
│  Everyone sits in one room.                                     │
│  Need to ask the designer something? Turn your chair.           │
│  Need to change the billing logic? Just do it.                  │
│  Need to deploy? One person pushes the button.                  │
│                                                                 │
│  Communication cost: ZERO (just talk)                           │
│  Coordination cost: ZERO (everyone sees everything)             │
│  Overhead: ZERO (no departments, no process)                    │
│                                                                 │
│  This is FAST. This is EFFICIENT.                               │
│  Startups are productive BECAUSE of this, not despite it.       │
│                                                                 │
│                                                                 │
│  But now the company grows to 200 people...                     │
│                                                                 │
│  Everyone in one room? Can't hear yourself think.               │
│  Need to ask the designer? Which of the 15 designers?           │
│  Need to change billing? File a request with 3 teams.           │
│  Need to deploy? Get in line behind 12 other teams.             │
│                                                                 │
│  You NEED departments. Not because departments are "better."    │
│  Because at this scale, the garage doesn't work anymore.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map it:**

```
Startup (Garage)              │  Software Architecture
──────────────────────────────│──────────────────────────
5 people in one room          │  Monolith (one process)
Everyone does everything      │  All code in one deployment
Shouting across the room      │  Function calls between modules
One shared whiteboard         │  One shared database
Hire more people → get louder │  More features → bigger deploys
Reorganize into departments   │  Decompose into services
Departments need email/Slack  │  Services need HTTP/queues
HR department handles hiring  │  Auth service handles identity
Each dept has own budget      │  Each service scales independently
```

> "Architecture decisions are, at their core, organizational decisions. This isn't a metaphor. It's a law. And it has a name."

---

# PART 2: MICROSERVICES — THE DECOMPOSITION

## 2.1 What Microservices Actually Are

**Strip away the hype. Here's what it means:**

```
┌─────────────────────────────────────────────────────────────────┐
│                MICROSERVICES: THE DEFINITION                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  An architectural style where a system is composed of           │
│  SMALL, INDEPENDENT services that:                              │
│                                                                 │
│  1. Own their data         (each has its own database/schema)   │
│  2. Deploy independently   (ship billing without shipping auth) │
│  3. Communicate over the   (HTTP, message queues, events)       │
│     network                                                     │
│  4. Are organized around   (not technical layers, but business  │
│     business capabilities    domains)                           │
│                                                                 │
│  KEY WORD: independently.                                       │
│  If you can't deploy Service A without also deploying           │
│  Service B, you don't have microservices.                       │
│  You have a distributed mess.                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Your capstone, reimagined:**

```
┌─────────────────────────────────────────────────────────────────┐
│           YOUR SAAS PLATFORM AS MICROSERVICES                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│   │ Auth Service │  │ Task Service │  │Notification  │         │
│   │              │  │              │  │  Service     │         │
│   │ FastAPI app  │  │ FastAPI app  │  │ FastAPI app  │         │
│   │ Own DB (users│  │ Own DB (tasks│  │ Own DB (notif│         │
│   │  sessions)   │  │  projects)   │  │  templates)  │         │
│   │ Port 8001    │  │ Port 8002    │  │ Port 8003    │         │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
│          │                 │                  │                 │
│          │      ┌──────────┴──────────┐       │                 │
│          │      │    Message Broker   │       │                 │
│          │      │   (Redis / Kafka)   │       │                 │
│          │      └─────────────────────┘       │                 │
│          │                                    │                 │
│   ┌──────┴───────┐  ┌──────────────┐  ┌──────┴───────┐         │
│   │Billing       │  │  Search      │  │  File        │         │
│   │  Service     │  │  Service     │  │  Service     │         │
│   │ FastAPI app  │  │ Go app (!)   │  │ FastAPI app  │         │
│   │ Own DB       │  │ Elasticsearch│  │ S3 storage   │         │
│   │ Port 8004    │  │ Port 8005    │  │ Port 8006    │         │
│   └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                 │
│  6 applications. 6 databases. 6 deployments. 6 CI pipelines.   │
│  6 sets of logs. 6 things that can fail independently.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Notice:** Search Service is written in Go. In microservices, each service can use the technology that best fits its problem. That's impossible in a monolith.

**Also notice:** 6 databases. Not one shared PostgreSQL. Each service **owns** its data. Task Service cannot directly query the users table. It must ASK Auth Service. This is the hardest rule to follow and the most important.

---

## 2.2 Finding Service Boundaries

**This is the hardest part of microservices. Get it wrong and you'll suffer for years.**

```
┌─────────────────────────────────────────────────────────────────┐
│               FINDING SERVICE BOUNDARIES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE QUESTION: Where do you draw the lines?                     │
│                                                                 │
│  WRONG APPROACH — Split by technical layer:                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│  │ API      │  │ Business │  │ Database │                      │
│  │ Service  │──│ Logic    │──│ Service  │                      │
│  │          │  │ Service  │  │          │                      │
│  └──────────┘  └──────────┘  └──────────┘                      │
│  ❌ Every feature change touches ALL three services.            │
│     You've gained nothing. Just added network calls.            │
│                                                                 │
│                                                                 │
│  RIGHT APPROACH — Split by business domain:                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│  │  Auth    │  │  Tasks   │  │ Billing  │                      │
│  │ (users,  │  │ (tasks,  │  │(invoices,│                      │
│  │  login,  │  │ projects,│  │ payments,│                      │
│  │  tokens) │  │  labels) │  │ plans)   │                      │
│  └──────────┘  └──────────┘  └──────────┘                      │
│  ✅ A billing change only touches the billing service.          │
│     Auth team deploys independently. Tasks team deploys         │
│     independently.                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to identify domains — the practical test:**

```
┌─────────────────────────────────────────────────────────────────┐
│            THREE TESTS FOR SERVICE BOUNDARIES                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TEST 1: THE TEAM TEST                                          │
│  ─────────────────────                                          │
│  "Can one team (3-8 people) own this service completely?"       │
│                                                                 │
│  If yes → Good boundary.                                        │
│  If the service needs 3 teams to change → Too big. Split more. │
│  If one person can maintain it part-time → Too small. Merge.    │
│                                                                 │
│                                                                 │
│  TEST 2: THE CHANGE TEST                                        │
│  ──────────────────────                                         │
│  "When I add a feature, how many services must change?"         │
│                                                                 │
│  If most features touch 1 service → Good boundaries.            │
│  If most features touch 3+ services → Bad boundaries.           │
│  Your boundaries should minimize cross-service changes.         │
│                                                                 │
│                                                                 │
│  TEST 3: THE DATA TEST                                          │
│  ─────────────────────                                          │
│  "Does this service own a coherent set of data?"                │
│                                                                 │
│  Auth owns: users, sessions, roles, permissions                 │
│  Tasks owns: tasks, projects, labels, assignments               │
│  Billing owns: invoices, subscriptions, payment methods         │
│                                                                 │
│  If two services constantly need each other's data to           │
│  function → They're probably one service.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Look at your capstone project's routers and repository modules. You already drew these boundaries instinctively. In Week 6, the repository pattern separated database logic by domain. Those repositories are proto-services. The boundary was always there."

---

## 2.3 Conway's Law (The Uncomfortable Truth)

**This is the single most important idea in software architecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      CONWAY'S LAW                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Any organization that designs a system will produce a         │
│   design whose structure is a copy of the organization's        │
│   communication structure."                                     │
│                                                — Melvin Conway  │
│                                                                 │
│  Translation:                                                   │
│  YOUR SOFTWARE ARCHITECTURE WILL MIRROR YOUR TEAM STRUCTURE.    │
│  Whether you intend it or not.                                  │
│                                                                 │
│                                                                 │
│  ONE TEAM:                                                      │
│  ┌───────────────────────────────┐                              │
│  │  Alice  Bob  Carol  Dave  Eve │ → Monolith                   │
│  └───────────────────────────────┘   (tight communication,     │
│                                       tightly coupled code)     │
│                                                                 │
│  THREE TEAMS:                                                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │Team Auth │ │Team Tasks│ │Team Bill │ → Three services        │
│  │Alice Bob │ │Carol Dave│ │Eve Frank │   (teams communicate   │
│  └──────────┘ └──────────┘ └──────────┘    through APIs, so    │
│       │            │            │           do their services)  │
│       └────────────┼────────────┘                               │
│              APIs & Messages                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why this matters:**

> "You don't choose microservices because they're technically superior. You adopt microservices because your ORGANIZATION has grown to the point where teams need autonomy. The architecture follows the org chart, not the other way around."

**The reverse also applies (Inverse Conway Maneuver):**

> "If you want a microservices architecture, organize your teams first. If you have one team and try to build microservices, that one team will end up coordinating across all services for every change. You'll get all the costs of distributed systems with none of the benefits."

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1 team + microservices  = Pain. All coordination, no benefit.  │
│  1 team + monolith       = Fast. This is your capstone.         │
│  5 teams + monolith      = Slow. Merge conflicts, deploy wars. │
│  5 teams + microservices = Right. Each team owns their domain.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 The Hidden Costs (What Nobody Tells You)

**Every conference talk hypes microservices. Nobody talks about the bill.**

```
┌─────────────────────────────────────────────────────────────────┐
│         THE REAL COST OF MICROSERVICES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MONOLITH                        MICROSERVICES                  │
│  ────────                        ──────────────                 │
│                                                                 │
│  1 repo                          6-20 repos                     │
│  1 CI pipeline                   6-20 CI pipelines              │
│  1 deployment                    6-20 deployments               │
│  1 database                      6-20 databases                 │
│  1 log stream                    6-20 log streams               │
│  Function calls (ns)             HTTP calls (ms)                │
│  Stack traces                    Distributed traces             │
│  DB transactions                 Saga patterns                  │
│  Shared types (import)           API contracts (OpenAPI)        │
│  pytest                          Contract testing + E2E         │
│  One Docker Compose              Kubernetes                     │
│                                                                 │
│  Infrastructure team: 0          Infrastructure team: 2-5       │
│                                  people JUST to keep the        │
│                                  platform running.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The cold math:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  In your monolith, calling auth from the task router:           │
│                                                                 │
│      user = await auth_service.get_current_user(token)          │
│                                                                 │
│      Time: ~0.001ms (function call)                             │
│      Can fail: No (same process)                                │
│      Needs retry logic: No                                      │
│      Needs timeout: No                                          │
│      Needs circuit breaker: No                                  │
│      Data format: Python objects                                │
│                                                                 │
│                                                                 │
│  In microservices, calling auth from the task service:           │
│                                                                 │
│      response = await client.get(                               │
│          "http://auth-service:8001/users/me",                   │
│          headers={"Authorization": f"Bearer {token}"},          │
│          timeout=5.0                                            │
│      )                                                          │
│      user = response.json()                                     │
│                                                                 │
│      Time: ~5-50ms (network hop)                                │
│      Can fail: Yes (network, timeout, service down)             │
│      Needs retry logic: Yes (Week 8)                            │
│      Needs timeout: Yes (Week 8)                                │
│      Needs circuit breaker: Yes (Week 8)                        │
│      Data format: JSON (must serialize/deserialize)             │
│                                                                 │
│  EVERY function call that crosses a service boundary            │
│  becomes a potential failure point that didn't exist before.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "In Week 8, you learned retry strategies, timeouts, and circuit breakers for external APIs. In a microservices architecture, EVERY service call is an external API call. That toolkit you built isn't optional — it's survival."

---

## 2.5 The Distributed Monolith Trap

**The worst outcome. Worse than staying with a monolith.**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE DISTRIBUTED MONOLITH                         │
│                (The Architecture Anti-Pattern)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT IT LOOKS LIKE:                                            │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                   │
│  │Service A │───▶│Service B │───▶│Service C │                   │
│  └──────────┘    └──────────┘    └──────────┘                   │
│       │               │               │                         │
│       └───────────────┼───────────────┘                         │
│                       │                                         │
│              ┌────────┴────────┐                                │
│              │  SHARED         │                                │
│              │  DATABASE       │  ← All services read/write    │
│              │                 │    the SAME tables             │
│              └─────────────────┘                                │
│                                                                 │
│  "Microservices" that:                                          │
│  ├─ Share a database (no data ownership)                        │
│  ├─ Must deploy together (coupled release cycles)              │
│  ├─ Call each other synchronously in chains (A→B→C→D)          │
│  └─ Break when any one service is down                          │
│                                                                 │
│  RESULT: All the complexity of distributed systems              │
│          + None of the benefits of independence                 │
│          + Worse than the monolith you started with             │
│                                                                 │
│                                                                 │
│  HOW TO DETECT IT:                                              │
│                                                                 │
│  "Can I deploy Service A without deploying Service B?"          │
│  If the answer is NO → Distributed monolith.                   │
│                                                                 │
│  "Can Service A function if Service B is down?"                 │
│  If the answer is always NO → Distributed monolith.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The takeaway:**

> "A well-structured monolith is BETTER than a badly-decomposed microservices system. If you split wrong — if services share databases, if they can't deploy independently, if they call each other in long synchronous chains — you've made everything harder for no gain."

> "This is why the industry consensus, from people who've done this at scale, is: **start with a monolith. Split when you feel the pain.** Not before."

---

# PART 3: HOW SERVICES COMMUNICATE

## 3.1 The Communication Problem

**The moment you split into services, you have a new problem.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE COMMUNICATION PROBLEM                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MONOLITH:                                                      │
│  ─────────                                                      │
│  task_service.py:                                               │
│      from app.services.auth import get_user  ← Direct import   │
│      user = get_user(user_id)                ← Function call   │
│                                                                 │
│  Done. No network. No serialization. No failure modes.          │
│                                                                 │
│                                                                 │
│  MICROSERVICES:                                                 │
│  ──────────────                                                 │
│  task_service can't import from auth_service.                   │
│  They're different applications. Different processes.           │
│  Maybe different servers. Maybe different continents.           │
│                                                                 │
│  HOW does Task Service get user data?                           │
│  HOW does Task Service tell Notification Service                │
│      "a task was completed, send an email"?                     │
│  HOW does Billing Service know a new org was created?           │
│                                                                 │
│  Two fundamental approaches:                                    │
│  ┌─────────────────┐    ┌──────────────────┐                    │
│  │  SYNCHRONOUS    │    │  ASYNCHRONOUS    │                    │
│  │  "Call and      │    │  "Send and       │                    │
│  │   wait"         │    │   forget"        │                    │
│  └─────────────────┘    └──────────────────┘                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Back to the company analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SYNCHRONOUS = PHONE CALL                                       │
│  ────────────────────────                                       │
│  You call the Legal department.                                 │
│  You wait on hold until they answer.                            │
│  You get your answer immediately.                               │
│  If they don't pick up → you're stuck.                          │
│                                                                 │
│  ASYNCHRONOUS = EMAIL / TICKET SYSTEM                           │
│  ────────────────────────────────────                           │
│  You email the Legal department.                                │
│  You go back to your work immediately.                          │
│  They process it when they can.                                 │
│  You get a response eventually.                                 │
│  If they're out of office → your email waits. You don't.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Synchronous: HTTP Between Services

**The simplest approach: one service calls another via REST.**

```python
# task-service/services/auth_client.py
# Task Service needs to verify a user — calls Auth Service via HTTP

import httpx
from tenacity import retry, stop_after_attempt, wait_exponential

AUTH_SERVICE_URL = "http://auth-service:8001"  # Internal Docker network

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=0.5, max=5)
)
async def get_user_from_auth(token: str) -> dict:
    """Call Auth Service to verify token and get user data.
    
    This is what used to be a function call in the monolith.
    Now it's an HTTP request across the network.
    """
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get(
            f"{AUTH_SERVICE_URL}/users/me",
            headers={"Authorization": f"Bearer {token}"}
        )
        response.raise_for_status()
        return response.json()
```

> "Look familiar? This is your Week 8 httpx code — retries, timeouts, all of it. The difference is that now the 'external API' isn't some third-party service. It's YOUR OWN AUTH SERVICE running on a different port."

**When synchronous communication works:**

```
┌─────────────────────────────────────────────────────────────────┐
│            SYNCHRONOUS COMMUNICATION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REQUEST FLOW:                                                  │
│                                                                 │
│  Client ──▶ Task Service ──▶ Auth Service                       │
│             │                │                                  │
│             │   "Verify      │  (looks up token,                │
│             │    this user"  │   returns user data)             │
│             │                │                                  │
│             │ ◀──────────────┘                                  │
│             │  {user data}                                      │
│  ◀──────────┘                                                   │
│  {task data + user info}                                        │
│                                                                 │
│                                                                 │
│  ✅ USE WHEN:                                                    │
│  ├─ You need the response RIGHT NOW to continue                 │
│  ├─ The call is fast (<100ms)                                   │
│  ├─ The caller cannot proceed without the answer                │
│  └─ Example: "Is this user authorized?" (yes/no needed now)    │
│                                                                 │
│  ❌ PROBLEMS:                                                    │
│  ├─ COUPLING: If Auth is down, Task is down too                │
│  ├─ LATENCY: Each hop adds milliseconds (they compound)        │
│  ├─ CASCADING FAILURE: A→B→C — if C is slow, A is slow         │
│  └─ CHAIN DEPTH: A→B→C→D→E = 5 points of failure              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The cascading failure problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  CASCADING FAILURE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NORMAL OPERATION:                                              │
│                                                                 │
│  Client → API Gateway → Task Service → Auth Service → DB       │
│           (5ms)          (10ms)         (15ms)        (5ms)    │
│                                                                 │
│  Total: ~35ms ✅                                                │
│                                                                 │
│                                                                 │
│  AUTH SERVICE DATABASE IS SLOW:                                  │
│                                                                 │
│  Client → API Gateway → Task Service → Auth Service → DB       │
│           (5ms)          (10ms)         (waiting...)   (30sec!) │
│                                                                 │
│  Task Service is blocked. Its thread pool fills up.             │
│  New requests to Task Service start timing out.                 │
│  API Gateway's connection pool to Task Service fills up.        │
│  ALL services behind the gateway become unreachable.            │
│                                                                 │
│  One slow database → entire platform down.                      │
│  This is WHY you learned circuit breakers in Week 8.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Asynchronous: Message Queues

**Instead of calling and waiting, drop a message and move on.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  MESSAGE QUEUE PATTERN                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  PRODUCER              BROKER               CONSUMER            │
│  (sends message)       (holds messages)      (processes later)  │
│                                                                 │
│  ┌──────────────┐    ┌───────────────┐    ┌──────────────┐      │
│  │ Task Service │    │               │    │ Notification │      │
│  │              │───▶│  ┌─┬─┬─┬─┐   │───▶│ Service      │      │
│  │ "Task done.  │    │  │ │ │ │ │   │    │              │      │
│  │  Send email."│    │  └─┴─┴─┴─┘   │    │ "Got it.     │      │
│  │              │    │   Message     │    │  Sending      │      │
│  │ (moves on    │    │   Queue       │    │  email now."  │      │
│  │  immediately)│    │               │    │              │      │
│  └──────────────┘    └───────────────┘    └──────────────┘      │
│                                                                 │
│  Task Service does NOT wait for the email to send.              │
│  Task Service does NOT know (or care) who processes this.       │
│  If Notification Service is down, messages WAIT in the queue.   │
│  When it comes back up, it processes the backlog.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "This pattern should feel familiar. Where have you seen 'producer sends work to a broker, consumer processes it later'?"

Answer: **Celery.** From Week 11. Your FastAPI app published tasks to Redis, Celery workers consumed them. Same pattern. Now zoom out: the producer and consumer are DIFFERENT SERVICES, not different processes of the same app.

```python
# task-service/events.py
# Task Service publishes an event when a task is completed.
# It does NOT call Notification Service directly.

import json
import redis.asyncio as redis

event_bus = redis.Redis(host="redis", port=6379, db=0)

async def publish_event(event_type: str, payload: dict) -> None:
    """Publish a domain event to the message broker."""
    event = {
        "type": event_type,
        "payload": payload,
        "timestamp": datetime.utcnow().isoformat()
    }
    await event_bus.publish("domain_events", json.dumps(event))


# Inside the task completion endpoint:
async def complete_task(task_id: int):
    task = await task_repo.mark_complete(task_id)
    
    # Publish event — fire and forget
    await publish_event("task.completed", {
        "task_id": task.id,
        "user_id": task.assigned_to,
        "project_id": task.project_id
    })
    
    # Return immediately. Don't wait for email/notification.
    return task
```

```python
# notification-service/consumer.py
# Notification Service listens for events and reacts.
# It has NO idea who publishes these events. It doesn't care.

async def listen_for_events():
    """Subscribe to domain events and process them."""
    pubsub = event_bus.pubsub()
    await pubsub.subscribe("domain_events")
    
    async for message in pubsub.listen():
        if message["type"] == "message":
            event = json.loads(message["data"])
            await handle_event(event)

async def handle_event(event: dict):
    match event["type"]:
        case "task.completed":
            await send_completion_email(event["payload"])
        case "user.invited":
            await send_invitation_email(event["payload"])
        case _:
            pass  # Ignore events we don't care about
```

**When asynchronous communication works:**

```
┌─────────────────────────────────────────────────────────────────┐
│           ASYNCHRONOUS COMMUNICATION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ USE WHEN:                                                    │
│  ├─ You DON'T need an immediate response                        │
│  ├─ The operation can happen eventually (seconds/minutes ok)   │
│  ├─ You want to decouple services (producer doesn't know       │
│  │   who consumes)                                              │
│  ├─ You need resilience (if consumer is down, messages wait)   │
│  └─ Examples: send email, generate report, sync to search      │
│                                                                 │
│  ❌ DOESN'T WORK WHEN:                                           │
│  ├─ You need the answer NOW ("is this user authorized?")       │
│  ├─ The client is waiting for a response                        │
│  └─ The action must be confirmed before proceeding             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Kafka vs RabbitMQ (Different Tools, Different Jobs)

> "In Week 11, you learned that Kafka and RabbitMQ exist. Now you need to understand WHEN you'd choose each."

```
┌─────────────────────────────────────────────────────────────────┐
│              RABBITMQ VS KAFKA                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RABBITMQ — "The Post Office"                                   │
│  ────────────────────────────                                   │
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                    │
│  │ Producer │──▶│ Exchange │──▶│  Queue   │──▶ Consumer        │
│  └──────────┘   │ (router) │   │          │   gets message,    │
│                 └──────────┘   │ message  │   message is       │
│                                │ removed  │   DELETED from     │
│                                │ after    │   queue.           │
│                                │ delivery │                    │
│                                └──────────┘                    │
│                                                                 │
│  • Smart broker, simple consumers                               │
│  • Message delivered to ONE consumer, then deleted              │
│  • Think: work distribution (like Celery!)                      │
│  • Great for: task queues, RPC, work that should happen ONCE   │
│                                                                 │
│                                                                 │
│  KAFKA — "The Append-Only Newspaper Archive"                    │
│  ────────────────────────────────────────────                   │
│                                                                 │
│  ┌──────────┐   ┌────────────────────────────────┐              │
│  │ Producer │──▶│ Topic: "task-events"            │              │
│  └──────────┘   │ ┌───┬───┬───┬───┬───┬───┬───┐  │              │
│                 │ │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │  │              │
│                 │ └───┴───┴───┴───┴───┴───┴───┘  │              │
│                 │            ▲           ▲        │              │
│                 │   Consumer A reads     │        │              │
│                 │   at position 3    Consumer B   │              │
│                 │                    reads at     │              │
│                 │                    position 5   │              │
│                 │                                 │              │
│                 │ Messages are NEVER deleted.     │              │
│                 │ Consumers track their position. │              │
│                 └────────────────────────────────┘              │
│                                                                 │
│  • Dumb broker (just a log), smart consumers                    │
│  • Messages persist — multiple consumers read independently    │
│  • Each consumer tracks its own position (offset)              │
│  • Great for: event streaming, audit logs, when multiple       │
│    services need the same events, when you need replay         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The practical decision:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CHOOSING BETWEEN THEM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  "Send a welcome email when user signs up"                      │
│   → RabbitMQ. One consumer, process once, delete.               │
│                                                                 │
│  "When a task is completed, update search index AND send        │
│   notification AND update analytics dashboard"                  │
│   → Kafka. Three consumers need the same event independently.  │
│                                                                 │
│  "Process uploaded files (resize images, extract text)"         │
│   → RabbitMQ. Work queue. Distribute across workers.            │
│                                                                 │
│  "Maintain a full audit trail of every mutation in the system   │
│   that can be replayed to rebuild state"                        │
│   → Kafka. Messages persist. You can replay from the start.    │
│                                                                 │
│  "I have a small team and simple needs"                         │
│   → RabbitMQ (or just Redis, like your Celery setup).          │
│                                                                 │
│  "I have 50 services that all need real-time event streams"     │
│   → Kafka. It was designed for exactly this.                    │
│                                                                 │
│                                                                 │
│  When in doubt? Start with RabbitMQ (or Redis queues).          │
│  Migrate to Kafka when you NEED fan-out or replay.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Event-Driven Architecture at Scale

> "You explored pub/sub and event-driven concepts in Week 11. Now see how they fit at the system level."

```
┌─────────────────────────────────────────────────────────────────┐
│            EVENT-DRIVEN ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TRADITIONAL (Request-Driven):                                  │
│  ──────────────────────────────                                 │
│  Task Service TELLS each service what to do:                    │
│                                                                 │
│  Task Service ──"send email"──▶ Notification Service            │
│  Task Service ──"update index"──▶ Search Service                │
│  Task Service ──"log event"──▶ Analytics Service                │
│                                                                 │
│  Task Service KNOWS about all downstream services.              │
│  Add a new service? Change Task Service code.                   │
│  This is tight coupling disguised as microservices.             │
│                                                                 │
│                                                                 │
│  EVENT-DRIVEN:                                                  │
│  ─────────────                                                  │
│  Task Service announces what HAPPENED. Doesn't care who's      │
│  listening.                                                     │
│                                                                 │
│                     ┌──────────────────┐                        │
│                     │  Event Bus       │                        │
│                     │  "task.completed"│                        │
│                     └────────┬─────────┘                        │
│                              │                                  │
│            ┌─────────────────┼─────────────────┐                │
│            │                 │                 │                │
│            ▼                 ▼                 ▼                │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│   │ Notification │  │   Search    │  │  Analytics  │          │
│   │   Service    │  │   Service   │  │   Service   │          │
│   │ (sends email)│  │(re-indexes) │  │(updates     │          │
│   └──────────────┘  └──────────────┘  │ dashboard) │          │
│                                       └──────────────┘          │
│                                                                 │
│  Task Service has ZERO knowledge of who listens.                │
│  Add a new service? Subscribe it. Zero changes to Task Service. │
│  Remove a service? Unsubscribe. Zero changes to Task Service.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key principle:**

> "In request-driven architecture, the caller says 'DO THIS.' In event-driven architecture, the publisher says 'THIS HAPPENED.' The difference is who holds the knowledge. Events invert the dependency — the publisher is ignorant of its consumers, and that ignorance is a feature."

> "Remember Week 11 when you learned about CQRS — separating read and write paths? Event-driven architecture is how that works at the system level. Write operations publish events. Read-optimized services consume those events and build their own views of the data."

---

## 3.6 Choosing Your Communication Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│          COMMUNICATION PATTERN DECISION TREE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│               "Service A needs to interact                      │
│                with Service B"                                  │
│                       │                                         │
│                       ▼                                         │
│            ┌─────────────────────┐                              │
│            │ Does A need B's     │                              │
│            │ response to         │                              │
│            │ continue?           │                              │
│            └──────────┬──────────┘                              │
│                 │           │                                   │
│                YES          NO                                  │
│                 │           │                                   │
│                 ▼           ▼                                   │
│    ┌────────────────┐  ┌────────────────────┐                   │
│    │  SYNCHRONOUS   │  │  ASYNCHRONOUS      │                   │
│    │  (HTTP/gRPC)   │  │  (Message Queue)   │                   │
│    └───────┬────────┘  └──────────┬─────────┘                   │
│            │                      │                             │
│            ▼                      ▼                             │
│    Must be FAST and         ┌──────────────────┐                │
│    service must be UP.      │ How many services │                │
│    Add circuit breaker      │ consume this?     │                │
│    and timeout.             └────────┬─────────┘                │
│                                │          │                     │
│                              ONE      MULTIPLE                  │
│                                │          │                     │
│                                ▼          ▼                     │
│                         ┌──────────┐ ┌──────────┐               │
│                         │ Work     │ │ Pub/Sub  │               │
│                         │ Queue    │ │ or Event │               │
│                         │(RabbitMQ)│ │ Stream   │               │
│                         └──────────┘ │ (Kafka)  │               │
│                                      └──────────┘               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Your SaaS platform mapped:**

```
┌─────────────────────────────────────────────────────────────────┐
│       WHICH PATTERN FOR WHICH INTERACTION?                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SYNCHRONOUS (HTTP):                                            │
│  ├─ Task Service → Auth Service: "Verify this token"           │
│  │   (Need answer NOW to authorize the request)                │
│  ├─ Gateway → Any Service: Route incoming request              │
│  │   (Client is waiting for response)                          │
│  └─ Billing Service → Auth Service: "Get org details"          │
│     (Need org data to calculate invoice)                       │
│                                                                 │
│  ASYNCHRONOUS (Message Queue / Events):                         │
│  ├─ Task completed → Notification: "Send email"                │
│  │   (User doesn't wait for email to be sent)                  │
│  ├─ User signed up → Search: "Index new user"                  │
│  │   (Search index can be seconds behind. That's fine.)        │
│  ├─ Any mutation → Audit: "Log this action"                    │
│  │   (Audit logging should never slow down the main path)      │
│  └─ Report requested → Worker: "Generate CSV"                  │
│     (Takes minutes. Client polls for result or gets notified.) │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: THE FRONT DOOR — API GATEWAY

## 4.1 The Problem: Where Do I Send This Request?

**Ask the class:**

> "In your monolith, the frontend calls ONE URL for everything: `https://api.yourapp.com`. Simple. Now imagine you've split into 6 services, each on a different port or host. Does the frontend need to know all 6 addresses?"

```
┌─────────────────────────────────────────────────────────────────┐
│              THE PROBLEM WITHOUT A GATEWAY                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Frontend must know:                                            │
│                                                                 │
│  "Login"           → https://auth.yourapp.com:8001/login       │
│  "Get my tasks"    → https://tasks.yourapp.com:8002/tasks      │
│  "My invoices"     → https://billing.yourapp.com:8004/invoices │
│  "Upload file"     → https://files.yourapp.com:8006/upload     │
│  "Search"          → https://search.yourapp.com:8005/query     │
│                                                                 │
│  Problems:                                                      │
│  ├─ Client is coupled to your internal architecture             │
│  ├─ Move a service → update every client                       │
│  ├─ CORS nightmare (5 different origins)                       │
│  ├─ Auth logic duplicated in every service                     │
│  ├─ Rate limiting duplicated in every service                  │
│  └─ Can't restructure services without breaking clients        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 API Gateway Pattern

**One front door. Routes to the right service.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│   Clients know ONE address: https://api.yourapp.com            │
│                                                                 │
│                     ┌──────────────────┐                        │
│                     │   API GATEWAY    │                        │
│                     │                  │                        │
│   Client ──────────▶│  /auth/*   ──────┼──▶ Auth Service       │
│                     │  /tasks/*  ──────┼──▶ Task Service       │
│                     │  /billing/*──────┼──▶ Billing Service    │
│                     │  /files/*  ──────┼──▶ File Service       │
│                     │  /search/* ──────┼──▶ Search Service     │
│                     │                  │                        │
│                     └──────────────────┘                        │
│                                                                 │
│   Client sends:  POST https://api.yourapp.com/tasks            │
│   Gateway routes: POST http://task-service:8002/tasks           │
│                                                                 │
│   Client has NO IDEA there are 5 services behind that gateway. │
│   You can restructure everything behind it without              │
│   changing a single client URL.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A simplified gateway in FastAPI (conceptual):**

```python
# api-gateway/main.py
# A simplified API Gateway — real ones use Kong, NGINX, AWS ALB, etc.
# Showing this in FastAPI so you see the pattern, not to suggest
# you'd build one from scratch.

from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import Response
import httpx

app = FastAPI(title="API Gateway")

# Service registry — where each service lives internally
SERVICES: dict[str, str] = {
    "auth":          "http://auth-service:8001",
    "tasks":         "http://task-service:8002",
    "notifications": "http://notification-service:8003",
    "billing":       "http://billing-service:8004",
}

@app.api_route(
    "/{service}/{path:path}",
    methods=["GET", "POST", "PUT", "PATCH", "DELETE"]
)
async def route_request(service: str, path: str, request: Request):
    """Forward incoming request to the appropriate internal service."""
    
    target_base = SERVICES.get(service)
    if not target_base:
        raise HTTPException(status_code=404, detail="Service not found")
    
    target_url = f"{target_base}/{path}"
    
    async with httpx.AsyncClient(timeout=10.0) as client:
        response = await client.request(
            method=request.method,
            url=target_url,
            headers={
                key: val for key, val in request.headers.items()
                if key.lower() != "host"
            },
            content=await request.body(),
            params=request.query_params,
        )
    
    return Response(
        content=response.content,
        status_code=response.status_code,
        headers=dict(response.headers),
    )
```

> "In production, you'd use a dedicated gateway like Kong, NGINX, AWS API Gateway, or Traefik — not a custom FastAPI app. But the pattern is exactly this: receive request, decide where it goes, forward it, return the response."

---

## 4.3 Gateway Responsibilities

**A gateway does more than just routing.**

```
┌─────────────────────────────────────────────────────────────────┐
│              GATEWAY RESPONSIBILITIES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                     ┌──────────────────────┐                    │
│                     │     API GATEWAY      │                    │
│                     ├──────────────────────┤                    │
│    Incoming    ──▶  │                      │  ──▶  Internal    │
│    Request          │  1. AUTHENTICATION   │       Service     │
│                     │     Verify JWT once  │                    │
│                     │     (not in every    │                    │
│                     │      service)        │                    │
│                     │                      │                    │
│                     │  2. RATE LIMITING    │                    │
│                     │     One place for    │                    │
│                     │     rate limits      │                    │
│                     │                      │                    │
│                     │  3. ROUTING          │                    │
│                     │     /tasks → task    │                    │
│                     │     service          │                    │
│                     │                      │                    │
│                     │  4. LOAD BALANCING   │                    │
│                     │     Distribute among │                    │
│                     │     service replicas │                    │
│                     │                      │                    │
│                     │  5. CORS HANDLING    │                    │
│                     │     One config for   │                    │
│                     │     all services     │                    │
│                     │                      │                    │
│                     │  6. LOGGING/TRACING  │                    │
│                     │     Assign request   │                    │
│                     │     ID, log at entry │                    │
│                     │                      │                    │
│                     │  7. RESPONSE CACHING │                    │
│                     │     Cache GET at the │                    │
│                     │     gateway level    │                    │
│                     │                      │                    │
│                     └──────────────────────┘                    │
│                                                                 │
│  Think of it like the middleware stack you built in your        │
│  FastAPI monolith — authentication, CORS, rate limiting,       │
│  logging. The gateway is middleware for the entire system.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Notice how the gateway's responsibilities are the same cross-cutting concerns you handled with FastAPI middleware and dependencies. CORS from Week 9. Rate limiting from Week 12. Structured logging from Week 15. Same problems, higher altitude."

---

## 4.4 Gateway Tradeoffs

```
┌─────────────────────────────────────────────────────────────────┐
│                GATEWAY TRADEOFFS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ BENEFITS:                                                    │
│  ├─ Clients see one URL (simple)                                │
│  ├─ Cross-cutting concerns in one place                         │
│  ├─ Internal services can change freely                         │
│  └─ Can aggregate responses from multiple services              │
│                                                                 │
│  ❌ COSTS:                                                       │
│  ├─ SINGLE POINT OF FAILURE (gateway dies → everything dies)   │
│  │   Must be highly available (multiple instances, Lecture 1   │
│  │   load balancers)                                           │
│  ├─ ADDED LATENCY (one extra network hop on every request)     │
│  ├─ BOTTLENECK (all traffic flows through it)                  │
│  └─ COMPLEXITY (one more thing to deploy, monitor, configure)  │
│                                                                 │
│                                                                 │
│  ⚠️  THE "GOD GATEWAY" ANTI-PATTERN:                            │
│  Putting too much logic in the gateway. Business rules,         │
│  data transformation, orchestration. The gateway should be      │
│  THIN — route, authenticate, rate limit. Nothing more.          │
│  Business logic belongs in the services.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: DESIGNING FOR FAILURE

## 5.1 The Fallacies of Distributed Computing

**Before microservices, your failure modes were limited.**

> "In your monolith, what could go wrong? The process crashes. The database goes down. That's basically it. Two things."

> "In a distributed system, the number of things that can fail increases EXPONENTIALLY. And the first step to designing for failure is understanding why."

```
┌─────────────────────────────────────────────────────────────────┐
│          THE FALLACIES OF DISTRIBUTED COMPUTING                │
│          (Peter Deutsch, 1994 — still painfully true)          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Engineers new to distributed systems assume:                   │
│                                                                 │
│  1. The network is reliable.              ← It's not.          │
│  2. Latency is zero.                      ← It's not.          │
│  3. Bandwidth is infinite.                ← It's not.          │
│  4. The network is secure.                ← It's not.          │
│  5. Topology doesn't change.              ← It does.           │
│  6. There is one administrator.           ← There isn't.       │
│  7. Transport cost is zero.               ← It's not.          │
│  8. The network is homogeneous.           ← It's not.          │
│                                                                 │
│                                                                 │
│  IN YOUR MONOLITH:                                              │
│  task_service calls auth_service → function call. Always works. │
│                                                                 │
│  IN MICROSERVICES:                                              │
│  task_service calls auth_service → network call.                │
│  What if the network drops for 50ms?                            │
│  What if auth_service takes 30 seconds to respond?              │
│  What if the response is corrupted?                             │
│  What if auth_service moved to a different IP?                  │
│  What if the connection pool is exhausted?                      │
│                                                                 │
│  EVERY call between services can fail in ways that              │
│  NEVER existed when everything was in one process.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The mindset shift:**

> "In a monolith, you design for the HAPPY PATH and handle exceptions. In a distributed system, you design for FAILURE and the happy path takes care of itself."

---

## 5.2 Redundancy (No Single Points of Failure)

```
┌─────────────────────────────────────────────────────────────────┐
│                     REDUNDANCY                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SINGLE POINT OF FAILURE (SPOF):                                │
│  ────────────────────────────────                               │
│  A component whose failure kills the entire system.             │
│                                                                 │
│  ❌ BAD — One instance of everything:                            │
│                                                                 │
│  Client → [Gateway] → [Task Service] → [Database]              │
│              ▲              ▲               ▲                   │
│              │              │               │                   │
│           If this dies,  If this dies,  If this dies,           │
│           everything     tasks don't    everything              │
│           is dead.       work.          is dead.                │
│                                                                 │
│                                                                 │
│  ✅ BETTER — Multiple instances, no single point of failure:    │
│                                                                 │
│             ┌─── [Gateway 1]                                    │
│  Client ────┤                ──┬── [Task Service 1]             │
│             └─── [Gateway 2]  │                                 │
│                               ├── [Task Service 2]              │
│                               │                                 │
│                               └── [Task Service 3]              │
│                                        │                        │
│                               ┌────────┴────────┐               │
│                               │   DB Primary    │               │
│                               │        │        │               │
│                               │   DB Replica    │               │
│                               └─────────────────┘               │
│                                                                 │
│  Gateway 1 dies → Gateway 2 takes over (load balancer)         │
│  Task Service 2 dies → requests go to 1 and 3                  │
│  DB Primary dies → replica promotes (from Lecture 1)           │
│                                                                 │
│  Nothing has a single instance. Everything has a backup.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "In Lecture 1 today, you learned about load balancers distributing traffic across replicas. THIS is why. Not just for performance — for survival."

---

## 5.3 The Bulkhead Pattern (Containing the Blast)

**Named after ship compartments that prevent one leak from sinking the whole vessel.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE BULKHEAD PATTERN                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE PROBLEM:                                                   │
│  ────────────                                                   │
│  Task Service calls Auth Service AND Notification Service.      │
│  Both use the SAME httpx connection pool.                       │
│                                                                 │
│  Notification Service is slow (overloaded email provider).      │
│  Its requests eat all connections in the shared pool.           │
│  Now Auth Service requests ALSO can't get connections.          │
│  Task Service can't verify users → EVERYTHING fails.           │
│                                                                 │
│  A problem in a NON-CRITICAL service (notifications)            │
│  killed a CRITICAL service (authentication).                    │
│                                                                 │
│                                                                 │
│  THE SOLUTION — ISOLATE RESOURCES:                              │
│  ─────────────────────────────────                              │
│                                                                 │
│  ┌──────────────────────────────────────────┐                   │
│  │           Task Service                   │                   │
│  │                                          │                   │
│  │  ┌─────────────────┐ ┌────────────────┐  │                   │
│  │  │ Auth Client     │ │ Notif Client   │  │                   │
│  │  │                 │ │                │  │                   │
│  │  │ Pool: 50 conns  │ │ Pool: 10 conns │  │                   │
│  │  │ Timeout: 2s     │ │ Timeout: 10s   │  │                   │
│  │  │ CRITICAL ⚡     │ │ NON-CRITICAL   │  │                   │
│  │  └────────┬────────┘ └───────┬────────┘  │                   │
│  │           │                  │            │                   │
│  └───────────│──────────────────│────────────┘                   │
│              │                  │                                │
│              ▼                  ▼                                │
│       Auth Service      Notification Service                    │
│                                                                 │
│  Notification pool exhausted? Only notifications fail.          │
│  Auth pool is completely separate. Auth still works.            │
│  The blast radius is CONTAINED.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In code (building on what you know from Week 8):**

```python
# task-service/clients.py
# Separate httpx clients with ISOLATED resource pools

import httpx

# CRITICAL PATH — auth verification is required for every request
auth_client = httpx.AsyncClient(
    base_url="http://auth-service:8001",
    timeout=httpx.Timeout(2.0),                    # Strict: fail fast
    limits=httpx.Limits(max_connections=50),        # Large pool: critical
)

# NON-CRITICAL PATH — notifications can fail without breaking anything
notification_client = httpx.AsyncClient(
    base_url="http://notification-service:8003",
    timeout=httpx.Timeout(10.0),                   # Lenient: can wait
    limits=httpx.Limits(max_connections=10),        # Small pool: non-critical
)

# If notification_client's 10 connections are all stuck waiting on a
# slow notification service, auth_client still has its own 50
# connections completely unaffected. The blast radius is contained.
```

> "Remember connection pooling from Week 7? Same concept. But now you're using separate pools strategically — not for performance, but for ISOLATION."

---

## 5.4 Fallback Strategies

```
┌─────────────────────────────────────────────────────────────────┐
│                  FALLBACK STRATEGIES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  When a service call fails, you have options:                   │
│                                                                 │
│                                                                 │
│  STRATEGY 1: RETURN CACHED DATA                                │
│  ──────────────────────────────                                 │
│  "Auth Service is down. But I cached this user's profile       │
│   5 minutes ago in Redis. Return the cached version."           │
│                                                                 │
│  Risk: Data might be slightly stale.                            │
│  Acceptable when: Read-only data that doesn't change often.    │
│                                                                 │
│                                                                 │
│  STRATEGY 2: RETURN DEFAULT/EMPTY                              │
│  ────────────────────────────────                               │
│  "Recommendation Service is down. Return an empty list          │
│   of recommendations instead of a 500 error."                   │
│                                                                 │
│  Risk: Reduced functionality.                                   │
│  Acceptable when: The feature is non-essential.                 │
│                                                                 │
│                                                                 │
│  STRATEGY 3: USE A SECONDARY SERVICE                           │
│  ───────────────────────────────────                            │
│  "Primary payment processor is down. Route to the backup       │
│   processor."                                                   │
│                                                                 │
│  Risk: Backup may be slower or more expensive.                  │
│  Acceptable when: The operation is business-critical.           │
│                                                                 │
│                                                                 │
│  STRATEGY 4: QUEUE FOR LATER                                   │
│  ──────────────────────────                                     │
│  "Email service is down. Put the email in a queue. Send it     │
│   when the service recovers."                                   │
│                                                                 │
│  Risk: Delayed delivery.                                        │
│  Acceptable when: Eventual delivery is fine.                    │
│                                                                 │
│                                                                 │
│  STRATEGY 5: FAIL FAST WITH CLEAR ERROR                        │
│  ──────────────────────────────────────                         │
│  "Billing Service is down. Return 503 immediately.              │
│   Don't make the user wait 30 seconds for a timeout."           │
│                                                                 │
│  This IS the circuit breaker from Week 8.                       │
│  Once you detect failure, stop trying and fail immediately.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Graceful Degradation (The Art of Partially Working)

**The goal: when parts fail, the whole system doesn't crash — it loses features gracefully.**

```
┌─────────────────────────────────────────────────────────────────┐
│                GRACEFUL DEGRADATION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FULL SYSTEM (all services healthy):                            │
│  ───────────────────────────────────                            │
│  GET /tasks/123 returns:                                        │
│  {                                                              │
│    "id": 123,                                                   │
│    "title": "Design API",                                       │
│    "assigned_to": {"id": 1, "name": "Alice", "avatar": "..."},│
│    "recommendations": ["Similar: Task 456", "Task 789"],       │
│    "activity_log": [{"user": "Bob", "action": "commented"}]    │
│  }                                                              │
│                                                                 │
│                                                                 │
│  AUTH SERVICE DOWN (critical degradation):                      │
│  ──────────────────────────────────────                         │
│  → Return 503 for authenticated routes.                         │
│    Can't serve tasks without knowing who's asking.              │
│    This is ACCEPTABLE — auth failure means "come back later."   │
│                                                                 │
│                                                                 │
│  RECOMMENDATION SERVICE DOWN (non-critical):                    │
│  ───────────────────────────────────────────                    │
│  → Return tasks WITHOUT recommendations.                        │
│  {                                                              │
│    "id": 123,                                                   │
│    "title": "Design API",                                       │
│    "assigned_to": {"id": 1, "name": "Alice", "avatar": "..."},│
│    "recommendations": [],     ← Empty, not an error            │
│    "activity_log": [{"user": "Bob", "action": "commented"}]    │
│  }                                                              │
│    The page still works. One section is empty. Nobody panics.   │
│                                                                 │
│                                                                 │
│  SEARCH SERVICE DOWN:                                           │
│  ────────────────────                                           │
│  → Disable search bar in the UI. Show "Search temporarily      │
│    unavailable." All other features work perfectly.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In code:**

```python
# task-service/routes/tasks.py
# Graceful degradation — non-critical features fail silently

async def get_task_detail(
    task_id: int,
    task_repo: TaskRepo = Depends(get_task_repo),
) -> TaskDetailResponse:
    # CRITICAL — must succeed or we return an error
    task = await task_repo.get_or_404(task_id)
    
    # NON-CRITICAL — failures return defaults, never crash the request
    recommendations = await _safe_get_recommendations(task_id)
    activity = await _safe_get_activity(task_id)
    
    return TaskDetailResponse(
        **task.model_dump(),
        recommendations=recommendations,
        activity_log=activity,
    )


async def _safe_get_recommendations(task_id: int) -> list[dict]:
    """Non-critical: returns empty list on ANY failure."""
    try:
        response = await recommendation_client.get(
            f"/recommendations/{task_id}"
        )
        response.raise_for_status()
        return response.json()
    except Exception:
        # Log it, but don't crash. Recommendations are a nice-to-have.
        logger.warning(f"Recommendation service unavailable for task {task_id}")
        return []


async def _safe_get_activity(task_id: int) -> list[dict]:
    """Non-critical: returns empty list on ANY failure."""
    try:
        response = await activity_client.get(f"/activity/{task_id}")
        response.raise_for_status()
        return response.json()
    except Exception:
        logger.warning(f"Activity service unavailable for task {task_id}")
        return []
```

**The principle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CLASSIFY EVERY DEPENDENCY AS CRITICAL OR NON-CRITICAL.         │
│                                                                 │
│  CRITICAL dependency fails    → Fail the request. Return 503.  │
│  NON-CRITICAL dependency fails → Return degraded response.     │
│                                  Log a warning. Move on.        │
│                                                                 │
│  The user sees a slightly less rich page.                       │
│  That's infinitely better than a 500 error.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.6 Transactions Across Services: The Saga Pattern

**In your monolith, you handled this easily.**

```python
# Monolith — one database transaction wraps everything.
# If any step fails, EVERYTHING rolls back. Clean.

async def create_organization(data: OrgCreate) -> Organization:
    async with db.begin():  # One transaction
        org = await org_repo.create(data)
        await billing_repo.create_subscription(org.id, plan="free")
        await notification_repo.schedule_welcome(org.owner_id)
        # If notification_repo fails → billing rolls back too.
        # ACID guarantees from Week 5. Everything or nothing.
```

**In microservices, there is no shared transaction.**

```
┌─────────────────────────────────────────────────────────────────┐
│          THE DISTRIBUTED TRANSACTION PROBLEM                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "Create Organization" now spans 3 services:                    │
│                                                                 │
│  Step 1: Org Service    → Create org         ✅ Success        │
│  Step 2: Billing Service → Create subscription ✅ Success      │
│  Step 3: Notification   → Send welcome email  ❌ FAILS         │
│                                                                 │
│  Now what?                                                      │
│                                                                 │
│  Org was created. Subscription was created. Email failed.       │
│  There's no BEGIN/COMMIT/ROLLBACK across services.              │
│  Each service has its OWN database with its OWN transactions.  │
│                                                                 │
│  You can't "undo" the org creation from the notification       │
│  service. The databases don't know about each other.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The Saga Pattern — a sequence of local transactions with compensating actions:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE SAGA PATTERN                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Instead of ONE distributed transaction,                        │
│  execute a SEQUENCE of local transactions.                      │
│  If any step fails, execute COMPENSATING ACTIONS to undo        │
│  previous steps.                                                │
│                                                                 │
│  FORWARD PATH (happy):                                          │
│  ─────────────────────                                          │
│  Step 1: Create org          ──▶ Success                       │
│  Step 2: Create subscription ──▶ Success                       │
│  Step 3: Send welcome email  ──▶ Success                       │
│  Done. ✅                                                        │
│                                                                 │
│                                                                 │
│  COMPENSATING PATH (step 3 fails):                              │
│  ─────────────────────────────────                              │
│  Step 1: Create org          ──▶ Success                       │
│  Step 2: Create subscription ──▶ Success                       │
│  Step 3: Send welcome email  ──▶ FAILURE ❌                     │
│                                                                 │
│  Compensate Step 2: Cancel subscription (undo)                  │
│  Compensate Step 1: Delete org (undo)                           │
│                                                                 │
│  Each service defines a "do" action AND an "undo" action.      │
│                                                                 │
│                                                                 │
│  STEP          │ ACTION              │ COMPENSATION             │
│  ──────────────│──────────────────── │──────────────────────    │
│  Create org    │ POST /orgs          │ DELETE /orgs/{id}        │
│  Create sub    │ POST /subscriptions │ DELETE /subscriptions/{} │
│  Send welcome  │ POST /notifications │ (none — can't unsend)   │
│                                                                 │
│                                                                 │
│  NOTE: The last step is "send email" because you CAN'T         │
│  unsend an email. Order matters. Put irreversible actions LAST. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Ask the class:**

> "Why is this harder than a database transaction?"

Let them think. Key answers: you must write compensation logic for every step, the compensation itself might fail, and there's a window where the system is in an inconsistent state. In your monolith, the database handled all of this with ROLLBACK. That's one of the costs of splitting.

> "This is the tax you pay for microservices. Every operation that used to be a single transaction becomes a saga with compensation logic. This is why you don't split until you HAVE to."

---

# PART 6: SERVICE MESH & THE BIG PICTURE

## 6.1 What is a Service Mesh?

**Brief. Awareness level. This is not something you'll implement — it's something you should know exists.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE MESH                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE PROBLEM:                                                   │
│  Every service needs: retries, timeouts, circuit breakers,     │
│  load balancing, TLS encryption, tracing, metrics.              │
│                                                                 │
│  Do you implement all of that in EVERY service? That's a       │
│  lot of duplicated infrastructure code.                         │
│                                                                 │
│                                                                 │
│  THE SOLUTION:                                                  │
│  A dedicated infrastructure layer that handles service-to-     │
│  service communication. Each service gets a "sidecar" proxy    │
│  that transparently handles the hard stuff.                     │
│                                                                 │
│                                                                 │
│  WITHOUT SERVICE MESH:                                          │
│  ┌────────────────┐                ┌────────────────┐           │
│  │  Task Service  │───── HTTP ────▶│  Auth Service  │           │
│  │  (+ retries)   │                │                │           │
│  │  (+ timeouts)  │                │                │           │
│  │  (+ TLS)       │                │                │           │
│  │  (+ tracing)   │  Your code     │                │           │
│  └────────────────┘  handles all   └────────────────┘           │
│                      of this.                                   │
│                                                                 │
│                                                                 │
│  WITH SERVICE MESH (e.g., Istio, Linkerd):                      │
│  ┌────────────────┐ ┌───────┐    ┌───────┐ ┌────────────────┐  │
│  │  Task Service  │─│ Proxy │────│ Proxy │─│  Auth Service  │  │
│  │  (just business│ │       │    │       │ │  (just business│  │
│  │   logic)       │ │handles│    │handles│ │   logic)       │  │
│  └────────────────┘ │retries│    │TLS    │ └────────────────┘  │
│                     │TLS    │    │metrics │                     │
│                     │tracing│    │routing │                     │
│                     └───────┘    └───────┘                      │
│                      Sidecar      Sidecar                       │
│                      proxy        proxy                         │
│                                                                 │
│  Your service code has ZERO awareness of retries, TLS,          │
│  tracing, or circuit breakers. The mesh handles it.             │
│                                                                 │
│  Analogy: In the company, the IT department manages the         │
│  phone system, email servers, building security, and network.   │
│  Departments just do their work without thinking about          │
│  infrastructure. The service mesh IS the IT department.         │
│                                                                 │
│                                                                 │
│  WHEN YOU NEED IT:                                              │
│  50+ services. Large team. Complex communication patterns.      │
│                                                                 │
│  WHEN YOU DON'T:                                                │
│  Under 10 services. Small team. The operational overhead        │
│  of running a mesh outweighs the benefits.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You will probably not deploy a service mesh in your first or even your fifth job. But in a system design interview, knowing it exists and what it solves shows you think about infrastructure, not just application code."

---

## 6.2 The Architecture Decision Framework

**The most valuable diagram in this lecture. Photograph it.**

```
┌─────────────────────────────────────────────────────────────────┐
│          WHICH ARCHITECTURE FOR WHICH SITUATION?                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  TEAM SIZE    USERS       ARCHITECTURE       YOU'VE BUILT IT   │
│  ─────────   ──────────  ─────────────────   ────────────────  │
│                                                                 │
│  1-5         < 10K       MONOLITH             ← Your capstone  │
│  engineers   users       Simple. Fast.          is here.        │
│                          Ship features.        And that's       │
│                          Don't overthink.      CORRECT.         │
│                                                                 │
│  5-20        10K-100K    MODULAR MONOLITH                      │
│  engineers   users       Well-defined module                    │
│                          boundaries. Shared                     │
│                          database. Could be                     │
│                          split later.                           │
│                          (Your capstone with                    │
│                          better boundaries.)                    │
│                                                                 │
│  20-100      100K-1M     MACRO-SERVICES                        │
│  engineers   users       2-5 large services                     │
│                          split by major                         │
│                          domain. Shared                         │
│                          infrastructure.                        │
│                                                                 │
│  100+        1M+         MICROSERVICES                          │
│  engineers   users       Fine-grained services.                │
│                          Full independence.                     │
│                          Service mesh. K8s.                     │
│                          Platform team.                         │
│                                                                 │
│                                                                 │
│  NOTICE: Team size is listed FIRST. Not users. Not traffic.    │
│  Architecture is primarily an ORGANIZATIONAL decision.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 The One Rule

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "If you can't build a well-structured monolith,               │
│   what makes you think you can build microservices?"            │
│                                                                 │
│                            — Simon Brown                        │
│                                                                 │
│                                                                 │
│  You built a well-structured monolith. Clean layers.            │
│  Repository pattern. Service layer. Dependency injection.       │
│  Separate routers per domain. Alembic migrations.               │
│  80%+ test coverage. Structured logging.                        │
│                                                                 │
│  THAT is the foundation. If the day comes when your             │
│  monolith needs to split, you'll know WHERE to cut              │
│  because the boundaries are already clear in your code.         │
│                                                                 │
│  Architecture is not a day-one choice.                          │
│  Architecture is an EVOLUTION in response to real pressure.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              ARCHITECTURE PATTERNS QUICK REFERENCE              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MONOLITH:                                                      │
│      One deployment. Simple. Start here.                        │
│      Split when team/scale forces you.                          │
│                                                                 │
│  MICROSERVICES:                                                 │
│      Independent services, own data, own deployments.           │
│      High cost. Need strong organizational reason.              │
│                                                                 │
│  SYNC COMMUNICATION (HTTP/gRPC):                                │
│      Use when caller needs response NOW.                        │
│      Add circuit breakers and timeouts.                         │
│                                                                 │
│  ASYNC COMMUNICATION (Message Queues):                          │
│      Use when caller can fire-and-forget.                       │
│      RabbitMQ for work queues. Kafka for event streams.         │
│                                                                 │
│  EVENT-DRIVEN:                                                  │
│      Publish "what happened," not "what to do."                 │
│      Decouples publisher from consumers completely.             │
│                                                                 │
│  API GATEWAY:                                                   │
│      One entry point for clients. Routes internally.            │
│      Handles auth, rate limiting, CORS at the edge.             │
│                                                                 │
│  DESIGNING FOR FAILURE:                                         │
│      Redundancy — no single points of failure.                  │
│      Bulkhead — isolate critical from non-critical.            │
│      Fallback — cache, defaults, queue for later.              │
│      Graceful degradation — partially work, don't crash.       │
│      Saga — compensating transactions across services.          │
│                                                                 │
│  SERVICE MESH:                                                  │
│      Infrastructure layer for service-to-service concerns.      │
│      Needed at 50+ services scale. Not before.                 │
│                                                                 │
│  THE ONE RULE:                                                  │
│      Start monolith. Structure well. Split when you must.       │
│      Architecture follows organization, not the reverse.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ARCHITECTURE = MANAGING TRADEOFFS UNDER GROWTH                │
│                                                                 │
│  Every pattern solves a problem but creates new ones.           │
│  The art is knowing which problems you'd rather have.           │
│                                                                 │
│                                                                 │
│  MONOLITH                    MICROSERVICES                      │
│  ────────                    ──────────────                     │
│  Simple to build             Complex to build                   │
│  Hard to scale teams         Easy to scale teams                │
│  One failure domain          Many failure domains               │
│  Easy transactions           Saga pattern                       │
│  Function calls (fast)       Network calls (fragile)            │
│  One deployment              Independent deployments            │
│  One tech stack              Any tech per service               │
│                                                                 │
│  Neither is "better." Each is right for a CONTEXT.              │
│                                                                 │
│                                                                 │
│  THE COMPANY ANALOGY:                                           │
│  ├─ Monolith      = Startup in a garage (fast, simple)         │
│  ├─ Microservices = Corporation with departments (scaled)      │
│  ├─ Sync comms    = Phone call (instant, blocks you)           │
│  ├─ Async comms   = Email (decoupled, eventual)                │
│  ├─ API Gateway   = Reception desk (single entry point)        │
│  ├─ Service Mesh  = IT department (infrastructure)             │
│  ├─ Circuit Breaker = "Stop calling legal, they're swamped"    │
│  ├─ Bulkhead      = Separate budgets per department            │
│  └─ Saga          = Multi-department approvals with rollback   │
│                                                                 │
│  You don't CHOOSE an architecture.                              │
│  You EVOLVE into one.                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Prior Lectures

```
┌─────────────────────────────────────────────────────────────────┐
│               WHAT YOU ALREADY KNEW                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOU LEARNED (in-process)       │ ARCHITECTURE EQUIVALENT       │
│  ───────────────────────────────│──────────────────────────     │
│  Repository pattern (Wk 6)     │ Service boundaries             │
│  Depends() injection (Wk 3)    │ Service discovery              │
│  Pydantic validation (Wk 3)    │ API contracts between services │
│  Circuit breaker (Wk 8)        │ System-level resilience        │
│  Retry + timeout (Wk 8)        │ Service communication safety   │
│  Celery + Redis (Wk 11)        │ Message queue architecture     │
│  Redis Pub/Sub (Wk 11-12)      │ Event-driven architecture      │
│  CORS middleware (Wk 9)        │ API Gateway responsibilities   │
│  FastAPI middleware (Wk 9)     │ Cross-cutting concerns at edge │
│  Docker Compose (Wk 15)       │ Multi-service deployment        │
│  Load balancers (Wk 16 L1)    │ Traffic distribution            │
│  DB transactions (Wk 5)       │ Saga pattern (the hard version) │
│  Redis cache (Wk 10)          │ Fallback / degradation strategy │
│                                │                                │
│  You've been learning architecture patterns all along.          │
│  This lecture just zoomed out and showed you the bigger map.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Connection to Upcoming

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE THIS LEADS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NEXT LECTURE (Wk 16, Lecture 3 — DSA & Interview Prep):       │
│  └─ System design interviews will ask you to design systems    │
│     using EXACTLY these patterns. "Design a task management     │
│     platform" — you'll draw monolith vs microservices,          │
│     choose sync vs async communication, add an API gateway,    │
│     and explain your tradeoffs. You've BUILT this system.      │
│     Now you can TALK about it.                                  │
│                                                                 │
│  YOUR CAPSTONE DOCUMENTATION:                                   │
│  └─ Your Architecture Decision Records should now include      │
│     "Why we chose a monolithic architecture" with the          │
│     tradeoffs from this lecture. This shows interviewers you   │
│     CHOSE a monolith deliberately, not out of ignorance.       │
│                                                                 │
│  YOUR CAREER:                                                   │
│  └─ Most backend roles work within existing architectures.     │
│     Some are monoliths being split. Some are microservices     │
│     being consolidated. Knowing BOTH sides of the tradeoff     │
│     is what separates a junior from a senior engineer.          │
│     Juniors follow patterns. Seniors choose them.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```