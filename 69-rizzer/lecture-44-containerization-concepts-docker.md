# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, TOOL LAST                                       │
│  ────────────────────────                                       │
│  Students must feel the deployment pain before seeing Docker.   │
│  We'll make them count every manual setup step.                 │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Containers are abstract. We use a SHIPPING CONTAINER analogy   │
│  throughout. Every concept maps to physical logistics.          │
│                                                                 │
│  CONCEPT BEFORE COMMAND                                         │
│  ─────────────────────                                          │
│  Students understand WHAT a container is (isolated process)     │
│  before they ever type `docker run`. Mental model first.        │
│                                                                 │
│  CONNECT TO 14 WEEKS OF WORK                                    │
│  ────────────────────────────                                   │
│  This lecture containerizes THEIR application — the FastAPI +   │
│  PostgreSQL + Redis + Celery stack they've built all course.    │
│  Every Dockerfile line maps to something they already know.     │
│                                                                 │
│  DEMYSTIFY, DON'T MYSTIFY                                       │
│  ─────────────────────────                                      │
│  Containers are NOT magic. They're processes with boundaries.   │
│  Students leave understanding the mechanism, not just the UI.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│              CONTAINERIZATION CONCEPTS & DOCKER                 │
│                     (3-4 Hour Lecture)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (40 min)                                   │
│  ├─ 1.1 The Setup Nightmare (Demonstration)                     │
│  ├─ 1.2 "It Works on My Machine"                                │
│  ├─ 1.3 What Virtual Environments DON'T Solve                   │
│  └─ 1.4 The Shipping Container Analogy                          │
│                                                                 │
│  PART 2: THE MENTAL MODEL (45 min)                              │
│  ├─ 2.1 What is a Container? (Process Isolation)                │
│  ├─ 2.2 Containers vs Virtual Machines                          │
│  ├─ 2.3 Images vs Containers (Blueprint vs House)               │
│  └─ 2.4 Docker Architecture (Client, Daemon, Registry)          │
│                                                                 │
│  PART 3: BUILDING IMAGES — THE DOCKERFILE (60 min)              │
│  ├─ 3.1 The Dockerfile — A Recipe                               │
│  ├─ 3.2 Your First Dockerfile (The Naive Version)               │
│  ├─ 3.3 The Layer System (Video Game Save Points)               │
│  ├─ 3.4 The Optimized Dockerfile (Layer Caching Done Right)     │
│  ├─ 3.5 Multi-Stage Builds (Ship Only What You Need)            │
│  └─ 3.6 .dockerignore (What NOT to Ship)                        │
│                                                                 │
│  PART 4: DOCKER COMPOSE — YOUR FULL STACK (50 min)              │
│  ├─ 4.1 Why Compose? (Your App is 6 Processes Now)              │
│  ├─ 4.2 docker-compose.yml — The Full Stack File                │
│  ├─ 4.3 Container Networking (The #1 Gotcha)                    │
│  ├─ 4.4 Volumes — Persistent Data and Development               │
│  └─ 4.5 Health Checks and Service Dependencies                  │
│                                                                 │
│  PART 5: COMMON MISTAKES AND PRODUCTION PATTERNS (30 min)       │
│  ├─ 5.1 The "Fat Image" Problem                                 │
│  ├─ 5.2 The Security Mistakes                                   │
│  ├─ 5.3 The Networking Confusion                                │
│  ├─ 5.4 The Data Loss Disaster                                  │
│  └─ 5.5 The 0.0.0.0 Mystery                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Setup Nightmare

**Start with a demonstration. Make them count the steps.**

> "You've spent 14 weeks building a backend. FastAPI, PostgreSQL, Redis, Celery workers, Alembic migrations, WebSockets. It runs beautifully on your machine. Now your teammate clones the repo. Here's what they have to do:"

```
┌─────────────────────────────────────────────────────────────────┐
│           SETTING UP YOUR PROJECT FROM SCRATCH                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Step 1:  Install Python 3.11+                                 │
│            └─ Mac: brew install python@3.11                     │
│            └─ Windows: download from python.org, add to PATH    │
│            └─ Linux: sudo apt install python3.11 (maybe PPA?)   │
│                                                                 │
│   Step 2:  Install PostgreSQL 16                                │
│            └─ Mac: brew install postgresql@16, brew services     │
│            └─ Windows: download installer, remember the         │
│               password you set, configure pg_hba.conf           │
│            └─ Linux: apt install, systemctl enable, createuser  │
│                                                                 │
│   Step 3:  Install Redis                                        │
│            └─ Mac: brew install redis                            │
│            └─ Windows: ...Redis doesn't officially support       │
│               Windows. Use WSL2? Maybe Memurai?                 │
│            └─ Linux: apt install redis-server                   │
│                                                                 │
│   Step 4:  Create virtual environment                           │
│            └─ python -m venv .venv                              │
│            └─ Activate it (different command per OS)             │
│                                                                 │
│   Step 5:  Install Python dependencies                          │
│            └─ pip install -r requirements.txt                   │
│            └─ Hope that no package needs a C compiler            │
│            └─ If it does: install gcc/Xcode/Visual Studio C++   │
│                                                                 │
│   Step 6:  Create the database                                  │
│            └─ psql -U postgres -c "CREATE DATABASE myapp"       │
│            └─ Wait, what's the postgres password again?          │
│                                                                 │
│   Step 7:  Set environment variables                            │
│            └─ DATABASE_URL, REDIS_URL, SECRET_KEY, ...          │
│            └─ Copy .env.example to .env, fill in values          │
│                                                                 │
│   Step 8:  Run Alembic migrations                               │
│            └─ alembic upgrade head                              │
│            └─ Hope the database URL is correct                   │
│                                                                 │
│   Step 9:  Start the API server                                 │
│            └─ uvicorn app.main:app --reload                     │
│                                                                 │
│   Step 10: Start Celery worker (separate terminal)              │
│            └─ celery -A app.celery_app worker --loglevel=info   │
│                                                                 │
│   Step 11: Start Celery Beat (another terminal)                 │
│            └─ celery -A app.celery_app beat --loglevel=info     │
│                                                                 │
│   Step 12: Maybe start Flower for monitoring                    │
│            └─ celery -A app.celery_app flower                   │
│                                                                 │
│   ───────────────────────────────────────────────               │
│   Total: 12 steps. 4 open terminals. 3 platform-specific       │
│   paths. At least 2 things WILL go wrong.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "How many of you had trouble setting up PostgreSQL in Week 5? Raise your hand."

*(Most hands go up.)*

> "Now imagine you have to write instructions so that ANYONE — on ANY operating system — can set up this exact environment. That README would be a novel. And it would be outdated within a month."

**Now show the alternative:**

```bash
# What if setup looked like this instead?
git clone https://github.com/yourteam/saas-backend.git
cd saas-backend
docker compose up
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  $ docker compose up                                            │
│                                                                 │
│  ✓ postgres:16 — running, healthy                               │
│  ✓ redis:7     — running, healthy                               │
│  ✓ api         — running on port 8000                           │
│  ✓ worker      — ready, listening for tasks                     │
│  ✓ beat        — scheduler active                               │
│  ✓ migrations  — applied 14 migrations                          │
│                                                                 │
│  All services running. Open http://localhost:8000/docs           │
│                                                                 │
│  Time elapsed: ~30 seconds                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "One command. Every service. Every dependency. Every configuration. Works on Mac, Windows, Linux. First time, every time. That's what we're learning today."

---

## 1.2 "It Works on My Machine"

**The fundamental problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│               "IT WORKS ON MY MACHINE"                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Developer A's Laptop        Developer B's Laptop              │
│   ─────────────────────       ─────────────────────             │
│   macOS 14                    Windows 11 + WSL2                 │
│   Python 3.11.7               Python 3.11.4                     │
│   PostgreSQL 16.1             PostgreSQL 15.3                   │
│   Redis 7.2                   Redis ??? (Windows?!)             │
│   openssl 3.2.0               openssl 3.1.1                     │
│   libpq 16.1                  libpq 15.3                        │
│                                                                 │
│                                                                 │
│   Production Server                                             │
│   ─────────────────────                                         │
│   Ubuntu 22.04                                                  │
│   Python 3.10.12  ← DIFFERENT MINOR VERSION                    │
│   PostgreSQL 14.9 ← TWO MAJOR VERSIONS BEHIND                  │
│   Redis 6.0       ← MISSING FEATURES YOU USED                  │
│                                                                 │
│                                                                 │
│   These THREE environments are supposed to run the              │
│   SAME application. They won't behave the same.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world symptoms:**

```
┌─────────────────────────────────────────────────────────────────┐
│               SYMPTOMS OF ENVIRONMENT DRIFT                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "It passes tests on my machine but fails in CI"               │
│  └─ Different Python patch version handles edge cases           │
│     differently.                                                │
│                                                                 │
│  "The query works locally but crashes in production"            │
│  └─ PostgreSQL 16 accepts syntax that PostgreSQL 14             │
│     doesn't.                                                    │
│                                                                 │
│  "Redis GETDEL works for me but throws an error for you"       │
│  └─ GETDEL was added in Redis 6.2. Your teammate               │
│     has Redis 6.0.                                              │
│                                                                 │
│  "The password hash doesn't verify on the server"              │
│  └─ Different bcrypt library version on the server              │
│     uses a different default rounds count.                      │
│                                                                 │
│  The problem is NOT the code.                                   │
│  The problem is the ENVIRONMENT the code runs in.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 What Virtual Environments DON'T Solve

**Connection to Week 1, Lecture 4:**

> "In Week 1, you learned about virtual environments — `venv`, `pip`, `requirements.txt`. Virtual environments solved one problem: isolating Python packages between projects. But that's only one layer of the stack."

```
┌─────────────────────────────────────────────────────────────────┐
│             WHAT venv ISOLATES vs. WHAT IT DOESN'T              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      YOUR APPLICATION                           │
│         ┌──────────────────────────────────────┐                │
│         │         Your Python Code             │                │
│         ├──────────────────────────────────────┤                │
│    ✅   │    Python Packages (pip)             │  ← venv        │
│  venv   │    fastapi, sqlalchemy, celery...    │    handles     │
│  does   ├──────────────────────────────────────┤    this        │
│  this   │    Python Interpreter (3.11)         │                │
│         ├──────────────────────────────────────┤                │
│         │    System Libraries                  │                │
│    ❌   │    libpq, openssl, gcc, libffi...    │  ← venv does  │
│  venv   ├──────────────────────────────────────┤    NOT handle  │
│  does   │    System Services                   │    any of      │
│  NOT    │    PostgreSQL, Redis, Nginx...       │    this        │
│  do     ├──────────────────────────────────────┤                │
│  this   │    Operating System                  │                │
│         │    macOS / Windows / Ubuntu 22.04    │                │
│         └──────────────────────────────────────┘                │
│                                                                 │
│  venv isolates ONE layer. Your app depends on ALL of them.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The question we need answered:**

> "What if we could package ALL of these layers — your code, your packages, the Python version, the system libraries, even parts of the OS — into a single, portable unit that runs identically everywhere?"

---

## 1.4 The Shipping Container Analogy

**This analogy carries through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│         BEFORE STANDARDIZED SHIPPING CONTAINERS (1950s)         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Loading a cargo ship in 1950:                                  │
│                                                                 │
│     🛢️ Oil barrels — need special handling                       │
│     📦 Wooden crates — fragile, different sizes                  │
│     🚗 Cars — oddly shaped, heavy                               │
│     🧺 Grain sacks — loose, need containment                    │
│     🧊 Frozen meat — needs refrigeration                        │
│                                                                 │
│  Every item loaded INDIVIDUALLY by hand.                        │
│  Every port had DIFFERENT equipment.                            │
│  Every ship had DIFFERENT storage layouts.                      │
│  Loading took DAYS. Theft was rampant.                          │
│  Damage was constant.                                           │
│                                                                 │
│  Sound familiar?                                                │
│  "Every developer has a DIFFERENT machine."                     │
│  "Every server has DIFFERENT software installed."               │
│  "Setup takes DAYS. Things break constantly."                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│        AFTER STANDARDIZED SHIPPING CONTAINERS (1960s+)          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The insight: STANDARDIZE THE BOX, NOT THE CONTENTS.            │
│                                                                 │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│     │  🛢️ Oil   │  │ 📦 Crates│  │ 🧊 Frozen│                   │
│     │  barrels  │  │ of goods │  │ meat     │                   │
│     └──────────┘  └──────────┘  └──────────┘                   │
│          │             │             │                          │
│     Same size.    Same size.    Same size.                      │
│     Same shape.   Same shape.   Same shape.                    │
│     Same corners. Same corners. Same corners.                  │
│                                                                 │
│  ANY crane can lift them.                                       │
│  ANY truck can carry them.                                      │
│  ANY ship can stack them.                                       │
│  ANY port can handle them.                                      │
│                                                                 │
│  The CONTENTS are completely different.                          │
│  The INTERFACE is completely standard.                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to software:**

```
┌─────────────────────────────────────────────────────────────────┐
│                SHIPPING → SOFTWARE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Physical World             │  Software World                   │
│  ────────────────────────── │ ──────────────────────────        │
│  Shipping container (box)   │  Docker container                 │
│  Contents (oil, grain, etc) │  Your app + dependencies          │
│  Container blueprint/spec   │  Docker image                     │
│  Bill of lading (packing    │  Dockerfile                       │
│    instructions)            │                                   │
│  Port / warehouse           │  Server / cloud platform          │
│  Container ship / truck     │  Docker runtime                   │
│  Container registry (yard)  │  Docker Hub (image storage)       │
│                             │                                   │
│  KEY INSIGHT:               │  KEY INSIGHT:                     │
│  The port doesn't care      │  The server doesn't care          │
│  what's INSIDE the box.     │  what's INSIDE the container.     │
│  It just knows how to       │  It just knows how to             │
│  handle the standard box.   │  run the standard container.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Docker didn't invent containerization — Linux had the underlying technology for years. Docker made it EASY to use, just like standardized shipping containers made global trade easy. The technology existed; Docker standardized the interface."

---

# PART 2: THE MENTAL MODEL

## 2.1 What is a Container? (Process Isolation)

**Strip away the mysticism:**

> "A container is NOT a tiny virtual computer. A container is a REGULAR PROCESS on your computer that has been given BOUNDARIES."

```
┌─────────────────────────────────────────────────────────────────┐
│              A CONTAINER IS A PROCESS WITH WALLS                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  When you run your FastAPI app normally:                        │
│                                                                 │
│     $ uvicorn app.main:app                                      │
│                                                                 │
│     This is a PROCESS on your computer.                         │
│     It can see ALL files on your disk.                          │
│     It can see ALL other running processes.                     │
│     It shares the network with everything else.                 │
│     It can use ALL available memory and CPU.                    │
│                                                                 │
│                                                                 │
│  When you run your FastAPI app in a container:                  │
│                                                                 │
│     $ docker run my-fastapi-app                                 │
│                                                                 │
│     This is STILL a process on your computer.                   │
│     But it can ONLY see files in its own filesystem.            │
│     It can ONLY see processes inside its container.             │
│     It has its OWN network interface.                           │
│     It has LIMITS on memory and CPU.                            │
│                                                                 │
│     Same process. Different boundaries.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The cubicle analogy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE OFFICE ANALOGY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT containers (regular process):                          │
│                                                                 │
│     ┌─────── Open Office Floor Plan ───────────────┐            │
│     │                                              │            │
│     │  👤 FastAPI   👤 PostgreSQL   👤 Redis        │            │
│     │                                              │            │
│     │  Everyone sees everyone.                     │            │
│     │  Everyone shares the same desk supplies.     │            │
│     │  Anyone can walk over and mess with           │            │
│     │  anyone else's work.                         │            │
│     │                                              │            │
│     └──────────────────────────────────────────────┘            │
│                                                                 │
│                                                                 │
│  WITH containers (isolated process):                            │
│                                                                 │
│     ┌──────────────────────────────────────────────┐            │
│     │                                              │            │
│     │  ┌──────────┐ ┌───────────┐ ┌──────────┐    │            │
│     │  │ 👤 API   │ │ 👤 Postgres│ │ 👤 Redis │    │            │
│     │  │          │ │           │ │          │    │            │
│     │  │ Own desk │ │ Own desk  │ │ Own desk │    │            │
│     │  │ Own files│ │ Own files │ │ Own files│    │            │
│     │  │ Own phone│ │ Own phone │ │ Own phone│    │            │
│     │  └──────────┘ └───────────┘ └──────────┘    │            │
│     │                                              │            │
│     │  Same building (same OS kernel).             │            │
│     │  But separate cubicles (isolated processes). │            │
│     │  They communicate through intercom (network).│            │
│     │                                              │            │
│     └──────────────────────────────────────────────┘            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How Linux does this (conceptual — you don't need to memorize this):**

```
┌─────────────────────────────────────────────────────────────────┐
│            THE TWO LINUX FEATURES BEHIND CONTAINERS             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NAMESPACES — "What can this process SEE?"                      │
│  ─────────────────────────────────────────                      │
│  ├─ PID namespace:   Container sees only ITS processes.         │
│  │                   Inside, your app is PID 1 (it thinks       │
│  │                   it's the only thing running).              │
│  ├─ Network ns:      Container gets its own IP, its own         │
│  │                   ports. Port 8000 inside ≠ port 8000        │
│  │                   outside.                                   │
│  ├─ Mount ns:        Container sees only ITS filesystem.        │
│  │                   Your host files are invisible.             │
│  └─ User ns:         "root" inside the container ≠ root         │
│                      on the host (for security).                │
│                                                                 │
│  CGROUPS — "How much can this process USE?"                     │
│  ──────────────────────────────────────────                     │
│  ├─ CPU limit:       "This container gets at most 2 cores"      │
│  ├─ Memory limit:    "This container gets at most 512MB"        │
│  └─ I/O limit:       "This container gets limited disk speed"   │
│                                                                 │
│  NAMESPACES = isolation (what you see)                           │
│  CGROUPS    = limitation (what you can use)                      │
│  Together   = a container                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You don't need to memorize namespaces or cgroups. But you need to understand that containers are NOT virtual machines. They're regular processes wearing blinders and a leash. That's all."

---

## 2.2 Containers vs Virtual Machines

**This distinction matters for understanding tradeoffs.**

```
┌─────────────────────────────────────────────────────────────────┐
│              VIRTUAL MACHINE ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   App A     │  │   App B     │  │   App C     │             │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤             │
│  │  Bins/Libs  │  │  Bins/Libs  │  │  Bins/Libs  │             │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤             │
│  │ Guest OS    │  │ Guest OS    │  │ Guest OS    │             │
│  │ (Ubuntu)    │  │ (Debian)    │  │ (Alpine)    │             │
│  │ ~1-2 GB     │  │ ~1-2 GB     │  │ ~200 MB     │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         └───────────┬────┴───────────┬────┘                     │
│              ┌──────┴──────┐                                    │
│              │  Hypervisor │  (VMware, VirtualBox, KVM)         │
│              ├─────────────┤                                    │
│              │  Host OS    │                                    │
│              ├─────────────┤                                    │
│              │  Hardware   │                                    │
│              └─────────────┘                                    │
│                                                                 │
│  Each VM runs a COMPLETE operating system.                      │
│  3 VMs = 3 full copies of an OS kernel.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                CONTAINER ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   App A     │  │   App B     │  │   App C     │             │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤             │
│  │  Bins/Libs  │  │  Bins/Libs  │  │  Bins/Libs  │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         └───────────┬────┴───────────┬────┘                     │
│              ┌──────┴──────┐                                    │
│              │ Container   │  (Docker Engine)                   │
│              │ Runtime     │                                    │
│              ├─────────────┤                                    │
│              │  Host OS    │  ← ONE kernel, SHARED              │
│              ├─────────────┤                                    │
│              │  Hardware   │                                    │
│              └─────────────┘                                    │
│                                                                 │
│  NO guest OS. Containers share the host kernel.                 │
│  3 containers = STILL just 1 OS kernel.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Side-by-side comparison:**

```
┌─────────────────────────────────────────────────────────────────┐
│             CONTAINERS vs VIRTUAL MACHINES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    Container           Virtual Machine           │
│  ─────────────    ───────────          ───────────────          │
│  Startup time     Milliseconds         30-60 seconds            │
│  Size             MBs (10-500)         GBs (1-20)              │
│  Overhead         Near zero            Significant (full OS)    │
│  Isolation        Process-level        Hardware-level           │
│  OS Kernel        Shared with host     Own kernel per VM        │
│  Density          100s per host        Tens per host            │
│  Boot process     Start a process      Boot entire OS           │
│  Portability      Docker image         VM image (large)         │
│                                                                 │
│  ─────────────────────────────────────────────────────          │
│                                                                 │
│  Container:  Cubicle in an office building                      │
│              (shares walls, electricity, plumbing)              │
│                                                                 │
│  VM:         Separate house on the same street                  │
│              (own walls, own plumbing, own foundation)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "For backend development, containers are almost always the right choice. VMs are for when you need strong security isolation — like running untrusted code, or when you literally need a different operating system. For running your FastAPI + Postgres + Redis stack? Containers. Every time."

---

## 2.3 Images vs Containers (Blueprint vs House)

**This distinction trips up everyone on day one.**

```
┌─────────────────────────────────────────────────────────────────┐
│               IMAGES vs CONTAINERS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IMAGE                              CONTAINER                   │
│  ─────                              ─────────                   │
│  A read-only TEMPLATE               A RUNNING INSTANCE          │
│                                     of an image                 │
│                                                                 │
│  Like a CLASS in Python             Like an OBJECT instance     │
│  Like a RECIPE                      Like the COOKED MEAL        │
│  Like a BLUEPRINT                   Like the BUILT HOUSE        │
│  Like a .py FILE                    Like a RUNNING PROCESS      │
│                                                                 │
│  You BUILD an image once.           You RUN many containers     │
│  It doesn't change.                 from one image.             │
│                                                                 │
│  Stored on disk.                    Running in memory.          │
│  Can be shared/pushed.              Lives and dies on one host. │
│                                                                 │
│                                                                 │
│         IMAGE                                                   │
│       (one blueprint)                                           │
│            │                                                    │
│      ┌─────┼─────────┐                                          │
│      │     │         │                                          │
│      ▼     ▼         ▼                                          │
│   ┌─────┐ ┌─────┐ ┌─────┐                                      │
│   │ C1  │ │ C2  │ │ C3  │  (three containers)                  │
│   │     │ │     │ │     │                                       │
│   │ dev │ │test │ │prod │  Same image, different contexts       │
│   └─────┘ └─────┘ └─────┘                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connecting to Python concepts:**

```python
# You already understand this pattern:

class FastAPIApp:          # ← Like a Docker IMAGE
    """The blueprint. Doesn't run by itself."""
    def __init__(self):
        self.db = Database()
        self.cache = Redis()

app1 = FastAPIApp()        # ← Like a CONTAINER (instance 1)
app2 = FastAPIApp()        # ← Like a CONTAINER (instance 2)
app3 = FastAPIApp()        # ← Like a CONTAINER (instance 3)

# One class, many instances.
# One image, many containers.
```

**The lifecycle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                IMAGE → CONTAINER LIFECYCLE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────┐     docker build     ┌─────────────┐           │
│  │ Dockerfile │ ──────────────────▶  │   IMAGE     │           │
│  │ (recipe)   │                      │  (template) │           │
│  └────────────┘                      └──────┬──────┘           │
│                                             │                  │
│                                      docker run               │
│                                             │                  │
│                                             ▼                  │
│                                      ┌─────────────┐          │
│                                      │  CONTAINER  │          │
│                                      │  (running)  │          │
│                                      └──────┬──────┘          │
│                                             │                  │
│                              ┌──────────────┼──────────────┐   │
│                              │              │              │   │
│                        docker stop    docker logs    docker exec│
│                              │              │              │   │
│                              ▼              ▼              ▼   │
│                        ┌──────────┐   (view output)  (run a   │
│                        │ STOPPED  │                  command   │
│                        │ container│                  inside)   │
│                        └──────────┘                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Docker Architecture (Client, Daemon, Registry)

**Three pieces work together:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  DOCKER ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR TERMINAL                  YOUR MACHINE                    │
│  ─────────────                  ────────────                    │
│  ┌──────────────┐               ┌──────────────────────┐        │
│  │ Docker CLI   │───commands───▶│ Docker Daemon        │        │
│  │              │               │ (dockerd)            │        │
│  │ docker build │               │                      │        │
│  │ docker run   │               │ Builds images        │        │
│  │ docker ps    │               │ Runs containers      │        │
│  │              │◀──responses───│ Manages networks     │        │
│  └──────────────┘               │ Manages volumes      │        │
│                                 └──────────┬───────────┘        │
│                                            │                    │
│                            docker pull / docker push            │
│                                            │                    │
│                                            ▼                    │
│                                 ┌──────────────────────┐        │
│  THE INTERNET                   │ Docker Registry      │        │
│  ────────────                   │ (Docker Hub)         │        │
│                                 │                      │        │
│                                 │ python:3.11-slim     │        │
│                                 │ postgres:16          │        │
│                                 │ redis:7-alpine       │        │
│                                 │ your-team/api:v1.2   │        │
│                                 └──────────────────────┘        │
│                                                                 │
│  CLI = what you TYPE                                            │
│  Daemon = what RUNS containers                                  │
│  Registry = where images are STORED and SHARED                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**You've already used this without knowing:**

> "Since Week 5, when you ran `docker run postgres:16`, here's what actually happened:"

```
┌─────────────────────────────────────────────────────────────────┐
│           WHAT HAPPENED IN WEEK 5                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  $ docker run postgres:16                                       │
│                                                                 │
│  Step 1: CLI sends command to Docker daemon                     │
│                                                                 │
│  Step 2: Daemon checks: "Do I have postgres:16 locally?"        │
│          └─ No → Pull from Docker Hub (registry)                │
│                                                                 │
│  Step 3: Daemon downloads the image (layers, ~150MB)            │
│          └─ Image contains: Debian Linux + PostgreSQL 16        │
│             + configuration + startup scripts                   │
│                                                                 │
│  Step 4: Daemon creates a container from the image              │
│          └─ New isolated process with its own filesystem        │
│                                                                 │
│  Step 5: Container starts                                       │
│          └─ PostgreSQL server boots inside the container        │
│          └─ Listens on port 5432 (inside the container)         │
│                                                                 │
│  You've been using containers for 10 weeks.                     │
│  Today you learn to BUILD your own.                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: BUILDING IMAGES — THE DOCKERFILE

## 3.1 The Dockerfile — A Recipe

**A Dockerfile is a text file with instructions for building an image.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE DOCKERFILE = A RECIPE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Restaurant Recipe                  Dockerfile                  │
│  ────────────────                   ──────────                  │
│  "Start with chicken breast"        FROM python:3.11-slim       │
│  "Move to cutting board"            WORKDIR /app                │
│  "Add salt and pepper"              COPY requirements.txt .     │
│  "Add garlic and olive oil"         RUN pip install -r req...   │
│  "Add the vegetables"               COPY . .                    │
│  "Oven at 375°F, 30 min"            EXPOSE 8000                 │
│  "Serve on plate"                   CMD ["uvicorn", ...]        │
│                                                                 │
│  The recipe is NOT the meal.        The Dockerfile is NOT       │
│  You EXECUTE the recipe             the image.                  │
│  to CREATE the meal.                You BUILD the Dockerfile    │
│                                     to CREATE the image.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Every Dockerfile instruction:**

```
┌─────────────────────────────────────────────────────────────────┐
│              DOCKERFILE INSTRUCTIONS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FROM      "Start from this base image"                         │
│            Every Dockerfile MUST start with FROM.               │
│            It's the foundation — usually an OS + language.      │
│                                                                 │
│  WORKDIR   "cd into this directory"                             │
│            Sets the working directory for subsequent commands.   │
│            Creates it if it doesn't exist.                      │
│                                                                 │
│  COPY      "Copy files from my machine into the image"          │
│            COPY <src on host> <dest in image>                   │
│                                                                 │
│  RUN       "Execute this shell command inside the image"        │
│            Used for installing packages, building things.       │
│            Each RUN creates a new layer.                        │
│                                                                 │
│  ENV       "Set an environment variable"                        │
│            Available during build AND at runtime.               │
│                                                                 │
│  EXPOSE    "Document which port the app listens on"             │
│            Does NOT actually publish the port.                  │
│            It's documentation for humans and tooling.           │
│                                                                 │
│  CMD       "The default command when a container starts"        │
│            Only ONE CMD per Dockerfile (last one wins).         │
│            This is what runs when you `docker run`.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Your First Dockerfile (The Naive Version)

**Let's containerize your FastAPI application. First attempt — the obvious way:**

```
Your current project structure (you've been building this since Week 3):

saas-backend/
├── app/
│   ├── __init__.py
│   ├── main.py              ← FastAPI app entry point
│   ├── models/              ← SQLAlchemy models
│   ├── routers/             ← API route handlers
│   ├── schemas/             ← Pydantic schemas
│   ├── services/            ← Business logic
│   ├── repositories/        ← Database queries
│   ├── dependencies.py      ← FastAPI dependencies
│   └── celery_app.py        ← Celery configuration
├── alembic/
│   ├── versions/            ← Migration files
│   └── env.py
├── alembic.ini
├── tests/
├── requirements.txt
├── .env                     ← Secrets (DATABASE_URL, etc.)
└── .git/
```

**First attempt — the naive Dockerfile:**

```dockerfile
# Dockerfile (v1 — naive, but it works)

FROM python:3.11

WORKDIR /app

COPY . .

RUN pip install -r requirements.txt

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Build and run it:**

```bash
# Build an image named "my-api" from the current directory
$ docker build -t my-api .

# Run a container from that image
$ docker run -p 8000:8000 my-api
```

**Ask the class:**

> "This works. Your FastAPI app is running in a container. But there are FIVE things wrong with this Dockerfile. Can you spot any?"

```
┌─────────────────────────────────────────────────────────────────┐
│             PROBLEMS WITH THE NAIVE DOCKERFILE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Problem 1: python:3.11 is HUGE (~900MB)                        │
│  └─ Includes compilers, build tools, documentation              │
│    you don't need at runtime.                                   │
│                                                                 │
│  Problem 2: COPY . . copies EVERYTHING                          │
│  └─ Includes .git/ (~50MB), .venv/, __pycache__/,              │
│    .env (YOUR SECRETS!), tests/, node_modules/...               │
│                                                                 │
│  Problem 3: Requirements installed AFTER code copy              │
│  └─ Change ONE line of code? ALL dependencies reinstall.        │
│    Every. Single. Time.                                         │
│                                                                 │
│  Problem 4: Running as root                                     │
│  └─ By default, everything runs as root inside the container.   │
│    A security vulnerability in your app = root access.          │
│                                                                 │
│  Problem 5: No .dockerignore                                    │
│  └─ The entire directory gets sent to Docker daemon as          │
│    "build context" — including files you don't need.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Let's fix these one by one. Starting with the most important concept: layers."

---

## 3.3 The Layer System (Video Game Save Points)

**This is the concept that makes Dockerfiles non-obvious.**

> "Every instruction in a Dockerfile creates a LAYER. Think of layers as VIDEO GAME SAVE POINTS."

```
┌─────────────────────────────────────────────────────────────────┐
│               DOCKERFILE LAYERS = SAVE POINTS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Dockerfile:                          Save Points:              │
│                                                                 │
│  FROM python:3.11-slim        ─────▶  💾 Save Point 1          │
│                                       (base OS + Python)        │
│                                                                 │
│  WORKDIR /app                 ─────▶  💾 Save Point 2          │
│                                       (directory created)       │
│                                                                 │
│  COPY requirements.txt .      ─────▶  💾 Save Point 3          │
│                                       (requirements file in)    │
│                                                                 │
│  RUN pip install ...          ─────▶  💾 Save Point 4          │
│                                       (all packages installed)  │
│                                                                 │
│  COPY . .                     ─────▶  💾 Save Point 5          │
│                                       (app code copied in)      │
│                                                                 │
│  CMD [...]                    ─────▶  💾 Save Point 6          │
│                                       (startup command set)     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The caching rule:**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE GOLDEN RULE OF DOCKER LAYERS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  If NOTHING changed at a layer, Docker LOADS THE SAVE.          │
│  If SOMETHING changed, Docker REPLAYS from that point.          │
│                                                                 │
│  Layer 1: FROM python:3.11-slim     → Cached ✅ (never changes)│
│  Layer 2: WORKDIR /app              → Cached ✅ (never changes)│
│  Layer 3: COPY requirements.txt .   → Cached ✅ (file same)    │
│  Layer 4: RUN pip install ...       → Cached ✅ (same input)   │
│  Layer 5: COPY . .                  → CHANGED! 🔄 (you edited  │
│                                       a .py file)              │
│  Layer 6: CMD [...]                 → REBUILD 🔄 (after change)│
│                                                                 │
│                                                                 │
│  KEY: Everything AFTER the changed layer gets rebuilt.           │
│  Everything BEFORE the changed layer uses cache.                │
│                                                                 │
│                                                                 │
│  ✅ ✅ ✅ ✅ 🔄 🔄                                              │
│  ─────────── ──────                                             │
│  cached       rebuilt                                           │
│  (seconds)    (could take minutes)                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now you see why the naive Dockerfile is slow:**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHY THE NAIVE DOCKERFILE REBUILDS EVERYTHING          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NAIVE (COPY . . BEFORE pip install):                           │
│                                                                 │
│  FROM python:3.11             → Cached ✅                       │
│  WORKDIR /app                 → Cached ✅                       │
│  COPY . .                     → CHANGED 🔄 (you edited main.py)│
│  RUN pip install -r req...    → REBUILD 🔄 (reinstalls ALL     │
│                                  packages from scratch!)        │
│                                                                 │
│  You changed ONE LINE of Python code.                           │
│  Docker reinstalls 50+ packages. Takes 2 minutes.              │
│  Every. Single. Build.                                          │
│                                                                 │
│                                                                 │
│  OPTIMIZED (requirements.txt FIRST, then code):                 │
│                                                                 │
│  FROM python:3.11-slim        → Cached ✅                       │
│  WORKDIR /app                 → Cached ✅                       │
│  COPY requirements.txt .      → Cached ✅ (file didn't change) │
│  RUN pip install -r req...    → Cached ✅ (same input!)        │
│  COPY . .                     → CHANGED 🔄 (you edited main.py)│
│                                                                 │
│  You changed ONE LINE of Python code.                           │
│  Docker reuses cached packages. Rebuild takes 2 SECONDS.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The principle:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  THINGS THAT CHANGE RARELY → early in the Dockerfile            │
│  (base image, system packages, Python dependencies)             │
│                                                                 │
│  THINGS THAT CHANGE OFTEN  → late in the Dockerfile             │
│  (your application code)                                        │
│                                                                 │
│  This way, most builds hit cache for the slow steps             │
│  and only re-do the fast steps (copying code).                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 The Optimized Dockerfile (Layer Caching Done Right)

**Version 2 — layer-optimized:**

```dockerfile
# Dockerfile (v2 — optimized layer caching)

# 1. Start from the SLIM variant (150MB instead of 900MB)
FROM python:3.11-slim

# 2. Set working directory
WORKDIR /app

# 3. Copy ONLY the dependency file first
#    This layer is cached until requirements.txt changes
COPY requirements.txt .

# 4. Install dependencies
#    This expensive step is cached as long as requirements.txt hasn't changed
#    --no-cache-dir: don't store pip's download cache (smaller image)
RUN pip install --no-cache-dir -r requirements.txt

# 5. NOW copy the rest of your code
#    This changes frequently, but it's AFTER the expensive pip install
COPY . .

# 6. Document the port (doesn't actually publish it)
EXPOSE 8000

# 7. Default command
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Build it and watch the cache in action:**

```bash
# First build — everything runs (slower)
$ docker build -t my-api .
Step 1/7 : FROM python:3.11-slim
 ---> a3b6e4c5d8f2
Step 2/7 : WORKDIR /app
 ---> Running in 8f2a3b6c...
Step 3/7 : COPY requirements.txt .
 ---> 4d5e6f7a...
Step 4/7 : RUN pip install --no-cache-dir -r requirements.txt
 ---> Running in 9a8b7c6d...     ← Takes 30-60 seconds
Installing collected packages: fastapi, uvicorn, sqlalchemy...
Step 5/7 : COPY . .
 ---> 1a2b3c4d...
Step 6/7 : EXPOSE 8000
Step 7/7 : CMD [...]

Built in 45 seconds


# Now edit app/main.py and rebuild
$ docker build -t my-api .
Step 1/7 : FROM python:3.11-slim
 ---> Using cache ✅
Step 2/7 : WORKDIR /app
 ---> Using cache ✅
Step 3/7 : COPY requirements.txt .
 ---> Using cache ✅              ← File didn't change!
Step 4/7 : RUN pip install ...
 ---> Using cache ✅              ← Dependencies cached!
Step 5/7 : COPY . .
 ---> 7e8f9a0b...                 ← Only this rebuilds
Step 6/7 : EXPOSE 8000
Step 7/7 : CMD [...]

Built in 2 seconds ⚡
```

> "Same result. 45 seconds → 2 seconds. The only difference? We put `COPY requirements.txt` BEFORE `COPY . .`. Layer order matters."

---

## 3.5 Multi-Stage Builds (Ship Only What You Need)

**The problem: build tools in production.**

```
┌─────────────────────────────────────────────────────────────────┐
│            THE BUILD ARTIFACT PROBLEM                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Some Python packages need COMPILATION during install:          │
│  ├─ bcrypt (your password hashing from Week 9)                  │
│  ├─ asyncpg (your async Postgres driver from Week 6)            │
│  ├─ uvloop (fast event loop)                                    │
│  └─ psycopg2 (alternative Postgres driver)                      │
│                                                                 │
│  To install these, you need BUILD TOOLS:                        │
│  ├─ gcc (C compiler)                                            │
│  ├─ python3-dev (Python header files)                           │
│  ├─ libpq-dev (PostgreSQL client library headers)               │
│  └─ libffi-dev (foreign function interface)                     │
│                                                                 │
│  These tools are needed to BUILD, but NOT to RUN.               │
│                                                                 │
│  Including them in your final image:                            │
│  ├─ Wastes ~200-400MB                                           │
│  ├─ Increases attack surface (compilers in production?!)        │
│  └─ Slows down deployment (larger image to transfer)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The solution: two stages.**

```
┌─────────────────────────────────────────────────────────────────┐
│               MULTI-STAGE BUILD CONCEPT                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STAGE 1: "BUILDER"                 STAGE 2: "RUNTIME"          │
│  (Build the ingredients)            (Serve the meal)            │
│                                                                 │
│  ┌──────────────────┐              ┌──────────────────┐         │
│  │ Full base image  │              │ Slim base image  │         │
│  │ + gcc, build     │              │ (no build tools) │         │
│  │   tools          │              │                  │         │
│  │                  │              │                  │         │
│  │ Install all      │              │ Copy ONLY the    │         │
│  │ Python packages  │──── copy ──▶│ installed pkgs   │         │
│  │ (compile C       │   compiled  │ from Stage 1     │         │
│  │  extensions)     │   packages  │                  │         │
│  │                  │              │ Copy app code    │         │
│  └──────────────────┘              │                  │         │
│                                    │ CMD [run app]    │         │
│  THIS STAGE IS THROWN AWAY.        └──────────────────┘         │
│  It does NOT end up in the                                      │
│  final image.                      THIS IS YOUR FINAL IMAGE.    │
│                                    Small. Clean. No compilers.  │
│                                                                 │
│  RESULT:                                                        │
│  Single-stage:  ~900MB                                          │
│  Multi-stage:   ~180MB (80% smaller!)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Version 3 — multi-stage Dockerfile:**

```dockerfile
# Dockerfile (v3 — multi-stage production build)

# ============================================================
# STAGE 1: Builder — install and compile dependencies
# ============================================================
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies (needed for compiling C extensions)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt
#                               ^^^^^^^^^^^^^^^^
#                               Install to /install instead of default location.
#                               This makes it easy to copy JUST the packages.


# ============================================================
# STAGE 2: Runtime — lean production image
# ============================================================
FROM python:3.11-slim

WORKDIR /app

# Install only RUNTIME libraries (not build tools)
# libpq5 is the runtime lib; libpq-dev was the build-time headers
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Copy installed packages from builder stage
COPY --from=builder /install /usr/local
#    ^^^^^^^^^^^^^^
#    This is the magic: grab artifacts from stage 1

# Create a non-root user for security
RUN useradd --create-home appuser
USER appuser

# Copy application code
COPY --chown=appuser:appuser . .

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**What happens during build:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MULTI-STAGE BUILD PROCESS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  $ docker build -t my-api .                                     │
│                                                                 │
│  Stage 1 (builder):                                             │
│  ├─ Pull python:3.11-slim                                       │
│  ├─ Install gcc, libpq-dev (build tools)                        │
│  ├─ pip install all packages into /install                      │
│  ├─ bcrypt compiles its C extension ✅                           │
│  ├─ asyncpg compiles its C extension ✅                          │
│  └─ Stage 1 complete (this entire image = ~500MB)               │
│                                                                 │
│  Stage 2 (runtime):                                             │
│  ├─ Pull python:3.11-slim (FRESH — no build tools!)             │
│  ├─ Install only libpq5 (runtime library, tiny)                 │
│  ├─ COPY --from=builder: grab compiled packages                 │
│  ├─ Create non-root user                                        │
│  ├─ Copy application code                                       │
│  └─ Stage 2 complete (final image = ~180MB)                     │
│                                                                 │
│  Stage 1 is DISCARDED. Only Stage 2 ships.                      │
│  gcc, libpq-dev, build artifacts — all gone.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Image size comparison:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  python:3.11                           ~920 MB  ████████████    │
│  python:3.11-slim (naive, no multi)    ~350 MB  █████           │
│  python:3.11-slim (multi-stage)        ~180 MB  ███             │
│  python:3.11-alpine (smallest, risky)  ~80 MB   █              │
│                                                                 │
│  Why not Alpine?                                                │
│  Alpine uses musl instead of glibc. Some Python packages        │
│  (e.g., pandas, numpy) break or compile slowly on Alpine.       │
│  Stick with -slim unless you have a reason.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.6 .dockerignore (What NOT to Ship)

**Just like `.gitignore` prevents files from entering git, `.dockerignore` prevents files from entering the Docker build context.**

> "When you run `docker build .`, Docker sends the ENTIRE current directory to the daemon as 'build context.' Without `.dockerignore`, you're shipping your `.git` directory, your virtual environment, your `.env` secrets file — everything."

```
# .dockerignore

# Version control (your .git can be 50-200MB)
.git
.gitignore

# Virtual environment (dependencies are installed in the image)
.venv
venv
__pycache__

# Environment files (SECRETS! Never bake into images)
.env
.env.*

# Tests (not needed in production image)
tests/

# IDE files
.vscode/
.idea/

# Docker files (prevent recursive confusion)
Dockerfile
docker-compose*.yml
.dockerignore

# Documentation
*.md
docs/

# OS files
.DS_Store
Thumbs.db
```

**Why `.env` in `.dockerignore` is CRITICAL:**

```
┌─────────────────────────────────────────────────────────────────┐
│              NEVER BAKE SECRETS INTO IMAGES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your .env file contains:                                       │
│  ├─ DATABASE_URL=postgresql://user:P@ssw0rd@localhost/db        │
│  ├─ SECRET_KEY=super-secret-jwt-signing-key                     │
│  ├─ REDIS_URL=redis://localhost:6379                            │
│  └─ SENDGRID_API_KEY=SG.xxxxx                                  │
│                                                                 │
│  If .env is COPY'd into the image:                              │
│  ├─ Anyone who pulls your image can extract the file            │
│  ├─ Even if you delete it in a later layer, it's still          │
│  │  in the earlier layer (layers are additive!)                 │
│  └─ Your production database password is now public             │
│                                                                 │
│  Instead: pass secrets as ENVIRONMENT VARIABLES at runtime.     │
│  We'll cover this properly in Lecture 2 with pydantic-settings. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: DOCKER COMPOSE — YOUR FULL STACK

## 4.1 Why Compose? (Your App is 6 Processes Now)

**Look at what you've built over 14 weeks:**

```
┌─────────────────────────────────────────────────────────────────┐
│         YOUR APPLICATION IS NOT ONE PROCESS ANYMORE             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐       │
│  │               YOUR SAAS BACKEND                      │       │
│  │                                                      │       │
│  │  ┌─────────┐  ┌──────────┐  ┌─────────────────┐     │       │
│  │  │ FastAPI │  │PostgreSQL│  │ Redis            │     │       │
│  │  │ Server  │  │ Database │  │ Cache + Broker   │     │       │
│  │  │         │  │          │  │                  │     │       │
│  │  │ (Week   │  │ (Week    │  │ (Week 10)       │     │       │
│  │  │  3-14)  │  │  5-7)    │  │                  │     │       │
│  │  └─────────┘  └──────────┘  └─────────────────┘     │       │
│  │                                                      │       │
│  │  ┌─────────┐  ┌──────────┐  ┌─────────────────┐     │       │
│  │  │ Celery  │  │ Celery   │  │ Flower          │     │       │
│  │  │ Worker  │  │ Beat     │  │ Monitor         │     │       │
│  │  │         │  │          │  │                  │     │       │
│  │  │ (Week   │  │ (Week    │  │ (Week 11)       │     │       │
│  │  │  11)    │  │  11)     │  │                  │     │       │
│  │  └─────────┘  └──────────┘  └─────────────────┘     │       │
│  │                                                      │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                 │
│  That's 6 services. Without Compose, you'd need to:             │
│                                                                 │
│  $ docker run postgres ...           (terminal 1)               │
│  $ docker run redis ...              (terminal 2)               │
│  $ docker run my-api ...             (terminal 3)               │
│  $ docker run my-api celery worker   (terminal 4)               │
│  $ docker run my-api celery beat     (terminal 5)               │
│  $ docker run my-api celery flower   (terminal 6)               │
│                                                                 │
│  And manually configure networking between them.                │
│  And manually set up volumes.                                   │
│  And manually restart them in the right order.                  │
│                                                                 │
│  Docker Compose: define ALL of this in ONE file.                │
│  Start with ONE command: docker compose up                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 docker-compose.yml — The Full Stack File

**Before we write YAML, a 30-second primer for those who haven't seen it:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   YAML IN 30 SECONDS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YAML = "YAML Ain't Markup Language"                            │
│  It's a data format, like JSON, but more human-readable.        │
│                                                                 │
│  Python dict     →  YAML                                        │
│  ────────────       ─────                                       │
│  {"name": "Alice",  name: Alice                                 │
│   "age": 30}        age: 30                                     │
│                                                                 │
│  Python list     →  YAML                                        │
│  ────────────       ─────                                       │
│  ["a", "b", "c"]    - a                                         │
│                      - b                                        │
│                      - c                                        │
│                                                                 │
│  Nesting uses INDENTATION (like Python!).                       │
│  USE SPACES, NOT TABS.                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now, your complete stack as a Compose file:**

```yaml
# docker-compose.yml — Your entire SaaS backend stack

services:
  # ── YOUR FASTAPI APPLICATION ──────────────────────────────────
  api:
    build: .                          # Build from Dockerfile in current dir
    ports:
      - "8000:8000"                   # Host port 8000 → Container port 8000
    environment:
      - DATABASE_URL=postgresql+asyncpg://appuser:apppass@postgres:5432/appdb
      #                                                   ^^^^^^^^
      #                            NOT localhost! The SERVICE NAME.
      - REDIS_URL=redis://redis:6379/0
      #                   ^^^^^
      #                   Service name, not localhost.
      - SECRET_KEY=dev-secret-key-change-in-production
    depends_on:
      postgres:
        condition: service_healthy    # Wait for Postgres to be READY
      redis:
        condition: service_healthy    # Wait for Redis to be READY
    volumes:
      - .:/app                        # Bind mount for live reload (dev only!)

  # ── POSTGRESQL DATABASE ───────────────────────────────────────
  postgres:
    image: postgres:16                # Pre-built image from Docker Hub
    environment:
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=apppass
      - POSTGRES_DB=appdb
    ports:
      - "5432:5432"                   # Expose to host (for DB tools)
    volumes:
      - postgres_data:/var/lib/postgresql/data   # Named volume for persistence
    healthcheck:
      test: ["CMD-line", "pg_isready", "-U", "appuser", "-d", "appdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  # ── REDIS ─────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine             # Alpine variant (tiny, ~30MB)
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  # ── CELERY WORKER ─────────────────────────────────────────────
  worker:
    build: .                          # Same Dockerfile as api
    command: celery -A app.celery_app worker --loglevel=info
    #        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    #        Override CMD from Dockerfile — run Celery instead of uvicorn
    environment:
      - DATABASE_URL=postgresql+asyncpg://appuser:apppass@postgres:5432/appdb
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  # ── CELERY BEAT (SCHEDULER) ───────────────────────────────────
  beat:
    build: .
    command: celery -A app.celery_app beat --loglevel=info
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      redis:
        condition: service_healthy

# ── NAMED VOLUMES (persistent storage) ─────────────────────────
volumes:
  postgres_data:        # Docker manages this volume
  redis_data:           # Data survives container restarts
```

**One command to rule them all:**

```bash
# Start everything
$ docker compose up

# Start everything in background (detached)
$ docker compose up -d

# See running services
$ docker compose ps

# View logs for a specific service
$ docker compose logs api
$ docker compose logs -f worker    # -f = follow (like tail -f)

# Run a one-off command in a service
$ docker compose exec api bash     # Open a shell in the api container
$ docker compose exec postgres psql -U appuser -d appdb  # Open psql

# Stop everything
$ docker compose down

# Stop everything AND delete volumes (DESTROYS DATA)
$ docker compose down -v
```

---

## 4.3 Container Networking (The #1 Gotcha)

**This is where students get confused. Pay close attention.**

> "When you run `docker compose up`, Docker creates a private NETWORK that all services share. Inside this network, each service is reachable BY ITS SERVICE NAME."

```
┌─────────────────────────────────────────────────────────────────┐
│       DOCKER COMPOSE CREATES A PRIVATE NETWORK                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR HOST MACHINE (macOS / Windows / Linux)                    │
│  ┌──────────────────────────────────────────────────────┐       │
│  │                                                      │       │
│  │   localhost = 127.0.0.1 = your machine               │       │
│  │                                                      │       │
│  │   ┌──── Docker Compose Network (172.18.0.0/16) ────┐ │       │
│  │   │                                                │ │       │
│  │   │  ┌──────────┐   ┌──────────┐   ┌──────────┐   │ │       │
│  │   │  │   api    │   │ postgres │   │  redis   │   │ │       │
│  │   │  │          │   │          │   │          │   │ │       │
│  │   │  │ 172.18.  │   │ 172.18.  │   │ 172.18.  │   │ │       │
│  │   │  │   0.4    │   │   0.2    │   │   0.3    │   │ │       │
│  │   │  │          │   │          │   │          │   │ │       │
│  │   │  │  :8000   │   │  :5432   │   │  :6379   │   │ │       │
│  │   │  └──────────┘   └──────────┘   └──────────┘   │ │       │
│  │   │                                                │ │       │
│  │   │  DNS INSIDE THE NETWORK:                       │ │       │
│  │   │  "postgres" → 172.18.0.2                       │ │       │
│  │   │  "redis"    → 172.18.0.3                       │ │       │
│  │   │  "api"      → 172.18.0.4                       │ │       │
│  │   │                                                │ │       │
│  │   └────────────────────────────────────────────────┘ │       │
│  │                                                      │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The critical rule:**

```
┌─────────────────────────────────────────────────────────────────┐
│                THE NETWORKING RULE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE DOCKER (how you connected in Weeks 5-14):               │
│                                                                 │
│    DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/db│
│                                                    ^^^^^^^^^    │
│                                                    your machine │
│                                                                 │
│                                                                 │
│  INSIDE DOCKER COMPOSE:                                         │
│                                                                 │
│    DATABASE_URL=postgresql+asyncpg://user:pass@postgres:5432/db │
│                                                    ^^^^^^^^     │
│                                                    service name │
│                                                                 │
│                                                                 │
│  WHY?                                                           │
│                                                                 │
│  Inside the "api" container:                                    │
│  ├─ "localhost" = the api container ITSELF (not your machine)   │
│  ├─ "postgres"  = the postgres container (Docker DNS)           │
│  ├─ "redis"     = the redis container (Docker DNS)              │
│  └─ "worker"    = the celery worker container                   │
│                                                                 │
│  Docker runs a DNS server inside the network.                   │
│  Service names are hostnames. Like a local /etc/hosts.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Port mapping — inside vs. outside:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 PORT MAPPING                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ports:                                                         │
│    - "8000:8000"                                                │
│       ^^^^  ^^^^                                                │
│       HOST  CONTAINER                                           │
│                                                                 │
│                                                                 │
│    YOUR MACHINE                    CONTAINER                    │
│    ────────────                    ─────────                    │
│                                                                 │
│    Browser requests                FastAPI listens              │
│    localhost:8000  ──────────────▶ 0.0.0.0:8000                 │
│                    port mapping                                 │
│                                                                 │
│                                                                 │
│  You CAN map to a different host port:                          │
│    - "9999:8000"                                                │
│                                                                 │
│    Browser → localhost:9999 → Container's port 8000             │
│                                                                 │
│                                                                 │
│  CONTAINER-TO-CONTAINER (inside the network):                   │
│    No port mapping needed! Services communicate directly.       │
│                                                                 │
│    api container → postgres:5432 → postgres container           │
│    (No "ports:" needed on postgres for api to reach it.         │
│     "ports:" is only for HOST access, like your DB tool.)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Here's a rule: `ports:` in docker-compose.yml exposes the service to YOUR MACHINE. Containers inside the Compose network can ALWAYS reach each other by service name and internal port. You only need `ports:` for services YOU want to access from your browser or external tools."

---

## 4.4 Volumes — Persistent Data and Development

**The ephemeral filesystem problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│          CONTAINERS ARE EPHEMERAL (THEY FORGET)                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Everything inside a container's filesystem                     │
│  DISAPPEARS when the container is removed.                      │
│                                                                 │
│  $ docker compose up                                            │
│    → PostgreSQL starts, you create tables, insert data          │
│                                                                 │
│  $ docker compose down                                          │
│    → Container removed. ALL DATA IS GONE.                       │
│                                                                 │
│  $ docker compose up                                            │
│    → Fresh PostgreSQL. Empty. No tables. No data. Nothing.      │
│                                                                 │
│  For a DATABASE, this is a disaster.                            │
│  You need data to PERSIST across container restarts.            │
│                                                                 │
│  Solution: VOLUMES.                                             │
│  A volume is storage OUTSIDE the container that gets            │
│  mounted INSIDE the container.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Two types of volumes you'll use:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   TWO TYPES OF VOLUMES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TYPE 1: NAMED VOLUME (for data persistence)                    │
│  ─────────────────────────────────────────────                  │
│                                                                 │
│  volumes:                                                       │
│    - postgres_data:/var/lib/postgresql/data                      │
│      ^^^^^^^^^^^^^                                              │
│      Docker manages this. You don't know (or care) where        │
│      it lives on your host. It just PERSISTS.                   │
│                                                                 │
│  Use for: Database data, Redis data, anything that must         │
│           survive `docker compose down` and `up`.               │
│                                                                 │
│  ┌────────────┐         ┌──────────────┐                        │
│  │ postgres   │ mounts  │ Named Volume │                        │
│  │ container  │────────▶│ postgres_data│                        │
│  │ /var/lib/  │         │              │                        │
│  │  postgresql│         │ (lives on    │                        │
│  │  /data     │         │  host disk)  │                        │
│  └────────────┘         └──────────────┘                        │
│  Container dies.        Volume survives.                        │
│                                                                 │
│                                                                 │
│  TYPE 2: BIND MOUNT (for development)                           │
│  ─────────────────────────────────────                          │
│                                                                 │
│  volumes:                                                       │
│    - .:/app                                                     │
│      ^  ^^^^                                                    │
│      │  inside container                                        │
│      current directory on HOST                                  │
│                                                                 │
│  Your PROJECT FOLDER gets mapped into the container.            │
│  Edit code on your host → container sees changes instantly.     │
│  Combined with --reload, you get live development.              │
│                                                                 │
│  Use for: DEVELOPMENT ONLY. Not production.                     │
│           Lets you edit code without rebuilding the image.       │
│                                                                 │
│  ┌────────────┐         ┌──────────────┐                        │
│  │ api        │ mounts  │ Your project │                        │
│  │ container  │────────▶│ folder on    │                        │
│  │ /app       │         │ your machine │                        │
│  │            │         │              │                        │
│  │ sees YOUR  │         │ ./app/       │                        │
│  │ live code  │         │ ./tests/     │                        │
│  └────────────┘         │ ./alembic/   │                        │
│                         └──────────────┘                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The danger of `docker compose down -v`:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      ⚠️  WARNING  ⚠️                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  $ docker compose down          ← Stops containers.             │
│                                   Named volumes SURVIVE.        │
│                                   Data is safe.                 │
│                                                                 │
│  $ docker compose down -v       ← Stops containers AND          │
│                                   DELETES named volumes.        │
│                                   YOUR DATABASE IS GONE.        │
│                                                                 │
│  The -v flag is useful for a clean reset during development.    │
│  It is CATASTROPHIC if you accidentally use it in production.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.5 Health Checks and Service Dependencies

**Connection to a pain point you've felt:**

> "Since Week 6, when you first connected FastAPI to PostgreSQL, some of you hit a race condition: the API tries to connect to the database, but PostgreSQL hasn't finished starting yet. The app crashes. You restart it, and it works because Postgres is ready by then."

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE STARTUP RACE CONDITION                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Time ──────────────────────────────────────────────▶           │
│                                                                 │
│  docker compose up:                                             │
│                                                                 │
│  postgres:  [starting.......initializing.....accepting]         │
│                                                                 │
│  api:       [starting..connect to postgres..💥 CRASH]           │
│                     ↑                                           │
│                     PostgreSQL not ready yet!                    │
│                                                                 │
│                                                                 │
│  depends_on (without health check):                             │
│  ─────────────────────────────────                              │
│  Only waits for the CONTAINER to start — NOT for the            │
│  SERVICE inside to be ready. PostgreSQL container can be         │
│  "running" but still initializing its database.                 │
│                                                                 │
│                                                                 │
│  depends_on + healthcheck:                                      │
│  ─────────────────────────                                      │
│  Waits for the health check to PASS — meaning PostgreSQL        │
│  is actually accepting connections.                             │
│                                                                 │
│  postgres:  [starting.......initializing..✅ healthy]           │
│  api:                                     [starting..connect..✅]│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Anatomy of a health check:**

```yaml
  postgres:
    image: postgres:16
    healthcheck:
      # The command to test if the service is healthy
      test: ["CMD-line", "pg_isready", "-U", "appuser", "-d", "appdb"]
      #      ^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      #      Run as      pg_isready is a PostgreSQL utility that
      #      shell cmd   returns 0 if the server is accepting connections
      
      interval: 5s      # Check every 5 seconds
      timeout: 5s       # If check takes longer than 5s, it's a failure
      retries: 5        # After 5 consecutive failures, mark as "unhealthy"
      start_period: 10s # Grace period during startup (don't count early failures)
```

```yaml
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      # redis-cli ping returns PONG if Redis is ready
      interval: 5s
      timeout: 5s
      retries: 5
```

```yaml
  api:
    build: .
    depends_on:
      postgres:
        condition: service_healthy   # ← Wait for health check to pass
      redis:
        condition: service_healthy   # ← Not just "container started"
```

**What Docker does with health checks:**

```
┌─────────────────────────────────────────────────────────────────┐
│                HEALTH CHECK STATE MACHINE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Container starts                                               │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────┐   start_period passes   ┌───────────────┐        │
│  │ starting │ ──────────────────────▶ │  health check │        │
│  └──────────┘   (no checks yet)       │  runs...      │        │
│                                       └───────┬───────┘        │
│                                          │         │           │
│                                       passes    fails          │
│                                          │         │           │
│                                          ▼         ▼           │
│                                    ┌─────────┐ ┌───────────┐   │
│                                    │ healthy │ │ unhealthy │   │
│                                    │   ✅     │ │  (retries │   │
│                                    └─────────┘ │   left?)  │   │
│                                                └───────────┘   │
│                                                                 │
│  depends_on with service_healthy waits until ✅                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: COMMON MISTAKES AND PRODUCTION PATTERNS

## 5.1 The "Fat Image" Problem

```
┌─────────────────────────────────────────────────────────────────┐
│                 FAT IMAGE MISTAKES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ MISTAKE: Using the full base image                          │
│                                                                 │
│     FROM python:3.11         ← 920MB!                           │
│     # Includes gcc, make, perl, dpkg-dev...                     │
│     # Do you need a Perl interpreter in production?             │
│                                                                 │
│  ✅ FIX: Use the slim variant                                   │
│                                                                 │
│     FROM python:3.11-slim    ← 150MB                            │
│     # Minimal Debian + Python. Nothing extra.                   │
│                                                                 │
│                                                                 │
│  ❌ MISTAKE: Installing dev dependencies in production           │
│                                                                 │
│     RUN pip install -r requirements.txt                         │
│     # requirements.txt includes pytest, ruff, mypy...           │
│                                                                 │
│  ✅ FIX: Separate requirements files                            │
│                                                                 │
│     COPY requirements.txt .                   # prod deps only  │
│     RUN pip install --no-cache-dir -r requirements.txt          │
│     # requirements-dev.txt is NOT copied                        │
│                                                                 │
│                                                                 │
│  ❌ MISTAKE: Leaving pip cache in the image                     │
│                                                                 │
│     RUN pip install -r requirements.txt       ← Caches all      │
│                                                 downloaded       │
│                                                 .whl files      │
│                                                                 │
│  ✅ FIX: --no-cache-dir                                         │
│                                                                 │
│     RUN pip install --no-cache-dir -r requirements.txt          │
│                                                                 │
│                                                                 │
│  ❌ MISTAKE: apt-get cache left in image                         │
│                                                                 │
│     RUN apt-get update && apt-get install -y libpq5             │
│                                                                 │
│  ✅ FIX: Clean up in the SAME RUN command                       │
│                                                                 │
│     RUN apt-get update \                                        │
│         && apt-get install -y --no-install-recommends libpq5 \  │
│         && rm -rf /var/lib/apt/lists/*                          │
│     # Must be same RUN — separate RUN still keeps the layer     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 The Security Mistakes

```
┌─────────────────────────────────────────────────────────────────┐
│                 SECURITY MISTAKES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ MISTAKE: Running as root                                    │
│                                                                 │
│     # If your app has a vulnerability, the attacker             │
│     # gets root access inside the container.                    │
│     # With certain misconfigurations, that can                  │
│     # escalate to the host.                                     │
│                                                                 │
│  ✅ FIX: Create and switch to a non-root user                   │
│                                                                 │
│     RUN useradd --create-home appuser                           │
│     USER appuser                                                │
│     # Everything after this runs as "appuser", not root         │
│                                                                 │
│                                                                 │
│  ❌ MISTAKE: Baking secrets into the image                      │
│                                                                 │
│     COPY .env .                                                 │
│     # Now anyone with the image has your database password      │
│     # Even if you delete it later — it's in the layer!          │
│                                                                 │
│  ✅ FIX: .dockerignore + runtime environment variables          │
│                                                                 │
│     # .dockerignore:                                            │
│     .env                                                        │
│     .env.*                                                      │
│                                                                 │
│     # Pass secrets at runtime:                                  │
│     $ docker run -e SECRET_KEY=abc123 my-api                    │
│     # Or via docker-compose.yml environment: section            │
│     # (More in Lecture 2: pydantic-settings)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 The Networking Confusion

```
┌─────────────────────────────────────────────────────────────────┐
│                NETWORKING MISTAKES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ MISTAKE: Using "localhost" in container env vars            │
│                                                                 │
│     # In docker-compose.yml:                                    │
│     environment:                                                │
│       - DATABASE_URL=...@localhost:5432/db                       │
│     # The api container tries to connect to ITSELF              │
│     # on port 5432. There's no Postgres there. 💥               │
│                                                                 │
│  ✅ FIX: Use the service name                                   │
│                                                                 │
│     environment:                                                │
│       - DATABASE_URL=...@postgres:5432/db                       │
│     # Docker DNS resolves "postgres" to the right container     │
│                                                                 │
│                                                                 │
│  ─────────────────────────────────────────────────────          │
│                                                                 │
│  ❌ MISTAKE: Thinking "ports:" is needed for containers         │
│     to talk to each other                                       │
│                                                                 │
│     # "My api can't connect to postgres!"                       │
│     # "Did you add ports: to postgres?"                         │
│     # That's not the issue.                                     │
│                                                                 │
│  ✅ FACT: Containers in the same Compose network can ALWAYS     │
│     reach each other on their internal ports.                   │
│     "ports:" only exposes to YOUR HOST MACHINE.                 │
│                                                                 │
│     You NEED ports: on postgres only if YOU want to connect     │
│     from your host (e.g., using pgAdmin or DBeaver).            │
│                                                                 │
│     In production, you likely DON'T expose database ports.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 The Data Loss Disaster

```
┌─────────────────────────────────────────────────────────────────┐
│                 DATA LOSS MISTAKES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ MISTAKE: No volume for database                             │
│                                                                 │
│     postgres:                                                   │
│       image: postgres:16                                        │
│       # No volumes section                                      │
│       # Container stops → ALL data gone                         │
│                                                                 │
│  ✅ FIX: Named volume for database data directory               │
│                                                                 │
│     postgres:                                                   │
│       image: postgres:16                                        │
│       volumes:                                                  │
│         - postgres_data:/var/lib/postgresql/data                 │
│                                                                 │
│     volumes:                                                    │
│       postgres_data:    # Declared at the top level             │
│                                                                 │
│                                                                 │
│  ❌ MISTAKE: Using bind mount for database in production        │
│                                                                 │
│     volumes:                                                    │
│       - ./data:/var/lib/postgresql/data                          │
│     # Permission issues. Performance issues on Mac/Windows.     │
│     # Docker's volume driver is optimized; bind mounts aren't.  │
│                                                                 │
│  ✅ FIX: Named volumes for data. Bind mounts for code (dev).   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 The 0.0.0.0 Mystery

```
┌─────────────────────────────────────────────────────────────────┐
│              THE 0.0.0.0 MYSTERY                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ MISTAKE: Running uvicorn with default host inside container │
│                                                                 │
│     CMD ["uvicorn", "app.main:app", "--port", "8000"]           │
│     # Default host = 127.0.0.1 (loopback)                      │
│     # uvicorn only listens on the container's internal          │
│     # loopback interface. Port mapping CANNOT reach it.         │
│                                                                 │
│     Result: Container is running. Port is mapped.               │
│     Browser shows "connection refused." You stare at the        │
│     screen for 30 minutes.                                      │
│                                                                 │
│                                                                 │
│  WHY this happens:                                              │
│                                                                 │
│     ┌──────────── Container ────────────┐                       │
│     │                                   │                       │
│     │   loopback (127.0.0.1)  ← uvicorn listens here           │
│     │                                   │                       │
│     │   eth0 (172.18.0.4)    ← port mapping connects here      │
│     │                                   │                       │
│     └───────────────────────────────────┘                       │
│                                                                 │
│     If uvicorn binds to 127.0.0.1, only processes INSIDE       │
│     the container can reach it. Port mapping uses eth0.         │
│                                                                 │
│                                                                 │
│  ✅ FIX: Bind to 0.0.0.0 (all interfaces)                      │
│                                                                 │
│     CMD ["uvicorn", "app.main:app",                             │
│          "--host", "0.0.0.0", "--port", "8000"]                 │
│                     ^^^^^^^^^                                   │
│          "Listen on ALL network interfaces"                     │
│          Now port mapping works. Browser connects.              │
│                                                                 │
│  This ONLY matters inside containers.                           │
│  On your host machine, 127.0.0.1 is fine for development.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Essential Docker Commands

```
┌─────────────────────────────────────────────────────────────────┐
│                    DOCKER COMMANDS REFERENCE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IMAGES:                                                        │
│  ───────                                                        │
│  docker build -t name .         Build image from Dockerfile     │
│  docker images                  List all local images           │
│  docker rmi name                Remove an image                 │
│                                                                 │
│  CONTAINERS:                                                    │
│  ───────────                                                    │
│  docker run name                Run a container from image      │
│  docker run -p 8000:8000 name   Run with port mapping           │
│  docker run -d name             Run in background (detached)    │
│  docker run -e KEY=VAL name     Run with environment variable   │
│  docker ps                      List running containers         │
│  docker ps -a                   List ALL containers (incl.      │
│                                   stopped)                      │
│  docker logs <id>               View container output           │
│  docker logs -f <id>            Follow logs (live tail)         │
│  docker exec -it <id> bash      Open shell inside container     │
│  docker stop <id>               Stop a container                │
│  docker rm <id>                 Remove a stopped container      │
│                                                                 │
│  COMPOSE:                                                       │
│  ────────                                                       │
│  docker compose up              Start all services              │
│  docker compose up -d           Start in background             │
│  docker compose up --build      Rebuild images then start       │
│  docker compose down            Stop and remove containers      │
│  docker compose down -v         Stop + delete volumes (⚠️)      │
│  docker compose ps              List running services           │
│  docker compose logs api        View logs for "api" service     │
│  docker compose logs -f api     Follow logs for "api"           │
│  docker compose exec api bash   Shell into "api" container      │
│  docker compose build           Rebuild all images              │
│                                                                 │
│  CLEANUP:                                                       │
│  ────────                                                       │
│  docker system prune            Remove unused data              │
│  docker system prune -a         Remove ALL unused (aggressive)  │
│  docker volume prune            Remove unused volumes           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                DOCKER QUICK REFERENCE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHAT IS A CONTAINER?                                           │
│      A process with isolation (own filesystem, network,         │
│      resource limits). NOT a VM. Shares the host kernel.        │
│                                                                 │
│  IMAGE vs CONTAINER:                                            │
│      Image = blueprint (class). Container = running             │
│      instance (object). One image → many containers.            │
│                                                                 │
│  DOCKERFILE LAYER RULE:                                         │
│      Rarely-changing steps FIRST (dependencies).                │
│      Frequently-changing steps LAST (your code).                │
│      Layers cache. Change invalidates everything after.         │
│                                                                 │
│  MULTI-STAGE BUILD:                                             │
│      Stage 1 (builder): install + compile.                      │
│      Stage 2 (runtime): COPY --from=builder. Ship this only.    │
│                                                                 │
│  NETWORKING IN COMPOSE:                                         │
│      "localhost" = the container itself, NOT your machine.       │
│      Use SERVICE NAMES: postgres, redis, api.                   │
│      "ports:" is for HOST access, not container-to-container.   │
│                                                                 │
│  VOLUMES:                                                       │
│      Named volume: for data persistence (databases).            │
│      Bind mount: for development (live code reload).            │
│                                                                 │
│  HEALTH CHECKS:                                                 │
│      depends_on alone = waits for container start.              │
│      depends_on + condition: service_healthy = waits for        │
│      the service to actually be READY.                          │
│                                                                 │
│  UVICORN IN CONTAINERS:                                         │
│      Always --host 0.0.0.0 inside a container.                  │
│                                                                 │
│  SECURITY:                                                      │
│      .dockerignore your .env files.                             │
│      USER appuser (don't run as root).                          │
│      Pass secrets as environment variables at runtime.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  DOCKER = STANDARDIZED PACKAGING FOR SOFTWARE                   │
│                                                                 │
│  Before Docker:                                                 │
│  "Here's my code. Good luck setting up Python 3.11,             │
│   PostgreSQL 16, Redis 7, and 47 system libraries."             │
│                                                                 │
│  After Docker:                                                  │
│  "Here's my image. Run it. Everything's inside."                │
│                                                                 │
│                                                                 │
│  THE THREE THINGS DOCKER GIVES YOU:                             │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐       │
│  │                                                      │       │
│  │  1. REPRODUCIBILITY                                  │       │
│  │     Same image → same behavior. Every time.          │       │
│  │     Dev = staging = production.                      │       │
│  │                                                      │       │
│  │  2. ISOLATION                                        │       │
│  │     Each service in its own container.               │       │
│  │     PostgreSQL can't mess with Redis.                │       │
│  │     A crash in the worker doesn't kill the API.      │       │
│  │                                                      │       │
│  │  3. PORTABILITY                                      │       │
│  │     Runs on your Mac. Runs on Linux CI.              │       │
│  │     Runs on AWS. Runs on your teammate's             │       │
│  │     Windows laptop. Same image. Same behavior.       │       │
│  │                                                      │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                 │
│  THE SHIPPING CONTAINER ANALOGY:                                │
│  ├─ Dockerfile      = Packing instructions                      │
│  ├─ Image           = Packed, sealed container                  │
│  ├─ Container       = Container loaded on a ship, in transit    │
│  ├─ Docker Hub      = Container storage yard                    │
│  ├─ Compose         = Logistics plan for multiple containers    │
│  └─ Volume          = Cargo that persists between shipments     │
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
│  WEEK 15, LECTURE 2 (Configuration & Observability):            │
│  └─ pydantic-settings to validate environment variables         │
│     that you pass into containers.                              │
│  └─ Structured logging (JSON logs from containers).             │
│  └─ Health check endpoints (/health, /ready) that your         │
│     Compose health checks will call.                            │
│  └─ Sentry integration for error tracking in containers.        │
│                                                                 │
│  WEEK 15, LECTURE 3 (CI/CD):                                    │
│  └─ GitHub Actions will BUILD your Docker image                 │
│     automatically on every push.                                │
│  └─ CI pipeline: lint → type check → test → build image         │
│     → push to registry.                                         │
│  └─ Alembic migrations in the deployment pipeline.              │
│                                                                 │
│  WEEK 15, LECTURE 4 (Cloud):                                    │
│  └─ Deploy your Docker image to a cloud platform                │
│     (Railway, Fly.io, or AWS).                                  │
│  └─ Managed databases vs Dockerized databases                   │
│     (spoiler: use managed in production).                       │
│                                                                 │
│  WEEK 16 (System Design):                                       │
│  └─ Containers are the UNIT of deployment in modern             │
│     architecture. Horizontal scaling = more containers.         │
│  └─ Kubernetes orchestrates containers at scale                 │
│     (awareness only — not in this course).                      │
│                                                                 │
│  CAPSTONE DELIVERABLE:                                          │
│  └─ Your entire SaaS backend, Dockerized.                       │
│     `docker compose up` is the ONLY setup step                  │
│     in your README.                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```