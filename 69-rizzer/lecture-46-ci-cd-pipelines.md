# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO FIRST, YAML LAST                                      │
│  ─────────────────────────                                      │
│  Students must SEE the deployment disaster before learning      │
│  the automation that prevents it. We'll break production.       │
│  Hypothetically.                                                │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  CI/CD is abstract infrastructure. We use a factory assembly    │
│  line analogy throughout. Every concept maps to something       │
│  physical: inspection stations, shipping departments,           │
│  factory workers.                                               │
│                                                                 │
│  BUILD THE PIPELINE ONE GATE AT A TIME                          │
│  ──────────────────────────────────────                         │
│  Don't dump a 60-line YAML file on students. Start with a      │
│  3-line workflow. Add one quality gate per step. Each gate      │
│  maps to a tool they already know.                              │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  mypy → Week 1, Lecture 1. pytest → Week 2, Lecture 2.          │
│  Docker → this week, Lecture 1. Alembic → Week 6, Lecture 3.    │
│  Git → Week 1, Lecture 4. pydantic-settings → Lecture 2.        │
│  Every CI/CD concept lands on soil already prepared.            │
│                                                                 │
│  YAML ≠ THE POINT                                               │
│  ────────────────                                               │
│  The point is the PIPELINE THINKING: automated, reproducible,   │
│  gated quality enforcement. GitHub Actions is the vehicle,      │
│  not the destination. The concepts transfer to any CI system.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                      CI/CD PIPELINES                            │
│                      (3-4 Hour Lecture)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (30 min)                                   │
│  ├─ 1.1 The Friday Night Deployment                             │
│  ├─ 1.2 The Human Checklist Failure                             │
│  └─ 1.3 The Factory Assembly Line Analogy                       │
│                                                                 │
│  PART 2: THE MENTAL MODEL (35 min)                              │
│  ├─ 2.1 CI vs CD — Two Different Machines                       │
│  ├─ 2.2 The Pipeline (The Assembly Line)                        │
│  ├─ 2.3 Events and Triggers (When the Line Starts)              │
│  └─ 2.4 Runners (Fresh Workers, Every Shift)                    │
│                                                                 │
│  PART 3: GITHUB ACTIONS SYNTAX (50 min)                         │
│  ├─ 3.1 YAML — You Already Know This (Connection to Lec 1)     │
│  ├─ 3.2 Workflow Files (The Factory Blueprints)                 │
│  ├─ 3.3 Your First Workflow                                     │
│  ├─ 3.4 Jobs and Steps (The Building Blocks)                    │
│  ├─ 3.5 Using Actions (Pre-Built Machines)                      │
│  └─ 3.6 Secrets and Variables (Connection to Lec 2)             │
│                                                                 │
│  PART 4: THE CI PIPELINE (55 min)                               │
│  ├─ 4.1 Quality Gates (The Inspection Stations)                 │
│  ├─ 4.2 Linting and Formatting with ruff                        │
│  ├─ 4.3 Type Checking with mypy (Connection to Wk 1 Lec 1)    │
│  ├─ 4.4 Testing with pytest + Coverage (Conn to Wk 2 Lec 2)   │
│  ├─ 4.5 Service Containers (PostgreSQL in CI)                   │
│  ├─ 4.6 Caching Dependencies                                    │
│  └─ 4.7 The Complete CI Pipeline                                │
│                                                                 │
│  PART 5: THE CD PIPELINE (45 min)                               │
│  ├─ 5.1 From CI to Deployment (The Shipping Department)         │
│  ├─ 5.2 Build and Push Docker Images (Connection to Lec 1)     │
│  ├─ 5.3 Database Migrations in Deployment (Conn to Wk 6)       │
│  ├─ 5.4 Environment-Specific Configuration (Conn to Lec 2)     │
│  └─ 5.5 Rolling Deployments and Rollback Strategies             │
│                                                                 │
│  PART 6: COMMON MISTAKES AND MISCONCEPTIONS (15 min)            │
│  ├─ 6.1 Committing Secrets                                      │
│  ├─ 6.2 Blocking the Event Loop in CI Tests                     │
│  ├─ 6.3 Forgetting --frozen                                     │
│  ├─ 6.4 Ignoring Failing CI                                     │
│  └─ 6.5 Running Migrations Before CI Passes                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Friday Night Deployment

**Start with a scenario. Make them cringe.**

> Imagine this. It's Friday, 6 PM. Your capstone project is due Monday. You and two teammates have been merging code all day. Time to deploy.

```
┌─────────────────────────────────────────────────────────────────┐
│                THE FRIDAY NIGHT DEPLOYMENT                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  6:00 PM — You SSH into the server.                             │
│  6:01 PM — git pull origin main                                 │
│  6:02 PM — "Hmm, did anyone run the tests?"                     │
│  6:02 PM — "I think so."                                        │
│  6:03 PM — uv sync && uv run alembic upgrade head               │
│  6:04 PM — Restart the server. Looks good.                      │
│  6:05 PM — Hit the API. 500 Internal Server Error.              │
│                                                                 │
│  6:06 PM — Check logs. TypeError on line 42 of user_service.py  │
│  6:07 PM — Teammate A pushed code with wrong type.              │
│            mypy would have caught it. Nobody ran mypy.           │
│                                                                 │
│  6:10 PM — Fix the type. Restart.                               │
│  6:11 PM — Hit the API again. 500.                              │
│  6:12 PM — Import error. Teammate B added a new dependency      │
│            but forgot to commit the lockfile.                    │
│                                                                 │
│  6:20 PM — Sync dependencies. Restart.                          │
│  6:21 PM — API works... but the /tasks endpoint returns [].     │
│  6:25 PM — Migration was wrong. Data column got dropped.        │
│  6:26 PM — No rollback plan.                                    │
│                                                                 │
│  6:30 PM — Panic.                                               │
│  8:45 PM — Finally working. Dinner is cold. Team is angry.      │
│                                                                 │
│  This was preventable. All of it.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Now ask the class:**

> "Every single failure in this story could have been caught BEFORE the code reached the server. The type error — mypy. The missing lockfile — a dependency install step. The broken migration — a test database run. Why didn't you catch them?"

Answer: **Because you relied on humans to remember to run things. Humans forget. Especially on Fridays.**

---

## 1.2 The Human Checklist Failure

**The deployment above is what happens when quality control is a mental checklist.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE HUMAN CHECKLIST                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Before merging code, you SHOULD:                               │
│                                                                 │
│  □ Run the linter                        ← "I'll do it later"  │
│  □ Check code formatting                 ← "My editor does it" │
│  □ Run type checker (mypy --strict)      ← "Takes too long"    │
│  □ Run the full test suite               ← "I ran MY tests"    │
│  □ Check test coverage                   ← "Probably fine"     │
│  □ Make sure dependencies are locked     ← "I think I did"     │
│  □ Verify the Docker image builds        ← "Works on my Mac"   │
│  □ Test database migrations              ← "What could go wrong"│
│  □ Update environment variables          ← "Which ones?"       │
│                                                                 │
│  9 steps. 3 team members. Under deadline pressure.              │
│  Someone WILL skip something.                                   │
│                                                                 │
│  FAILURE RATE: Not if. When.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The real problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "I ran the tests" ≠ "All tests passed"                         │
│  "I ran mypy"      ≠ "mypy --strict with no errors"             │
│  "I formatted"     ≠ "I formatted ALL files"                    │
│  "It works"        ≠ "It works on a clean machine"              │
│                                                                 │
│  Humans say "I checked." Machines prove it.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "The solution is not to try harder. The solution is to remove humans from the checklist entirely. Automate every check. Make the machine enforce quality. That's CI/CD."

---

## 1.3 The Factory Assembly Line Analogy

**This analogy will carry us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE FACTORY ASSEMBLY LINE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT AN ASSEMBLY LINE (Manual Deployment)                   │
│  ─────────────────────────────────────────────                  │
│                                                                 │
│  A factory worker hand-inspects each product:                   │
│                                                                 │
│    "Does it look right?" → Eyeball check (unreliable)           │
│    "Does it measure right?" → Ruler check (sometimes skipped)   │
│    "Does it work?" → Quick test (incomplete)                    │
│    "Ship it." → Hope for the best.                              │
│                                                                 │
│  Result: Defective products reach customers.                    │
│          Recalls. Refunds. Reputation damage.                   │
│                                                                 │
│                                                                 │
│  WITH AN ASSEMBLY LINE (CI/CD)                                  │
│  ──────────────────────────────                                 │
│                                                                 │
│  Every product passes through AUTOMATED INSPECTION STATIONS:    │
│                                                                 │
│    Station 1: Visual scan (laser)       → PASS or REJECT        │
│    Station 2: Measurement (sensors)     → PASS or REJECT        │
│    Station 3: X-ray (internal defects)  → PASS or REJECT        │
│    Station 4: Stress test (load/heat)   → PASS or REJECT        │
│                                                                 │
│    ALL PASS → Ship to customer.                                 │
│    ANY FAIL → Product is REJECTED. Line STOPS. Fix the issue.   │
│                                                                 │
│  Result: Only verified products ship. Consistent. Reliable.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to CI/CD:**

```
Factory                         │  CI/CD
────────────────────────────────│───────────────────────────────
Factory                         │  Your Git repository
Raw materials arrive            │  Code is pushed / PR is opened
Assembly line starts            │  Pipeline triggers
                                │
Inspection Station 1:           │  Lint check (ruff check)
  Visual scan                   │    Catches style issues, unused imports
                                │
Inspection Station 2:           │  Format check (ruff format --check)
  Measurement check             │    Catches inconsistent formatting
                                │
Inspection Station 3:           │  Type check (mypy --strict)
  X-ray scan                    │    Catches invisible type errors
                                │
Inspection Station 4:           │  Test suite (pytest)
  Stress test                   │    Catches logic bugs, regressions
                                │
ALL PASS stamp                  │  Green checkmark ✅ on the PR
ANY FAIL rejection              │  Red X ❌ — merge blocked
                                │
Shipping department             │  CD pipeline (build, deploy)
Factory floor blueprint         │  Workflow YAML file
Factory workers                 │  Runners (GitHub's servers)
Fresh shift every morning       │  Runners are ephemeral (clean VM each run)
```

> "From this point forward: inspection stations = CI. Shipping department = CD. Factory blueprint = workflow file. Workers = runners."

---

# PART 2: THE MENTAL MODEL

## 2.1 CI vs CD — Two Different Machines

**CI and CD are often said together, but they do very different things.**

```
┌─────────────────────────────────────────────────────────────────┐
│                     CI VS CD                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONTINUOUS INTEGRATION (CI)                                    │
│  ────────────────────────────                                   │
│  "Does this code meet our standards?"                           │
│                                                                 │
│  • Runs on EVERY push and pull request                          │
│  • Checks code quality (lint, format, types)                    │
│  • Runs tests (unit, integration)                               │
│  • Reports pass or fail                                         │
│  • Does NOT deploy anything                                     │
│                                                                 │
│  Factory analogy: The INSPECTION LINE.                          │
│  Products are tested. Defective ones are rejected.              │
│  Nothing ships from here.                                       │
│                                                                 │
│                                                                 │
│  CONTINUOUS DEPLOYMENT (CD)                                     │
│  ──────────────────────────                                     │
│  "Ship the verified code to users."                             │
│                                                                 │
│  • Runs ONLY after CI passes                                    │
│  • Runs ONLY on specific branches (main, production)            │
│  • Builds Docker image                                          │
│  • Pushes to container registry                                 │
│  • Runs database migrations                                     │
│  • Deploys to servers                                           │
│                                                                 │
│  Factory analogy: The SHIPPING DEPARTMENT.                      │
│  Only products that passed ALL inspections reach here.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The relationship:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│          CI must pass              CD only runs                  │
│          before CD                 for main branch               │
│              │                         │                        │
│              ▼                         ▼                        │
│  ┌───────────────────┐   ┌───────────────────────┐              │
│  │        CI         │──▶│          CD            │             │
│  │  (Quality Gates)  │   │  (Build + Deploy)      │             │
│  │                   │   │                        │             │
│  │  ✅ Lint          │   │  📦 Build Docker image │             │
│  │  ✅ Format        │   │  📤 Push to registry   │             │
│  │  ✅ Type check    │   │  🗄️  Run migrations    │             │
│  │  ✅ Tests         │   │  🚀 Deploy to server   │             │
│  └───────────────────┘   └───────────────────────┘              │
│                                                                 │
│  If ANY CI check fails → CD NEVER runs.                         │
│  Broken code never reaches production.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A note on terminology:**

> "You'll hear 'Continuous Delivery' and 'Continuous Deployment' used interchangeably. Technically, Continuous Delivery means 'code is always in a deployable state, but a human clicks the button.' Continuous Deployment means 'code deploys automatically, no human needed.' For this course, we'll set up Continuous Deployment — merge to main, and it ships."

---

## 2.2 The Pipeline (The Assembly Line)

**A pipeline is an ordered sequence of automated stages.**

```
┌─────────────────────────────────────────────────────────────────┐
│                      THE PIPELINE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Code Push                                                      │
│      │                                                          │
│      ▼                                                          │
│  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐             │
│  │  LINT  │──▶│ FORMAT │──▶│  TYPE  │──▶│  TEST  │             │
│  │        │   │        │   │ CHECK  │   │        │             │
│  └────┬───┘   └────┬───┘   └────┬───┘   └────┬───┘             │
│       │            │            │            │                  │
│      PASS?        PASS?        PASS?        PASS?               │
│      │   │        │   │        │   │        │   │               │
│     ✅   ❌       ✅   ❌       ✅   ❌       ✅   ❌              │
│      │    │        │    │        │    │        │    │            │
│      ▼   STOP     ▼   STOP     ▼   STOP     ▼   STOP           │
│    next           next          next       ┌──────┐             │
│    stage          stage         stage      │DEPLOY│             │
│                                            └──────┘             │
│                                                                 │
│  Key rule: The pipeline STOPS at the first failure.             │
│  No point testing code that doesn't even pass linting.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The pipeline is a gatekeeper, not a servant:**

> "The pipeline doesn't exist to help you deploy. It exists to PREVENT you from deploying bad code. Every stage is a gate. Your code must earn the right to pass through each one."

---

## 2.3 Events and Triggers (When the Line Starts)

**The assembly line doesn't run 24/7. It starts when raw materials arrive.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PIPELINE TRIGGERS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EVENT                         │  WHAT HAPPENS                  │
│  ──────────────────────────────│────────────────────────────     │
│                                │                                │
│  Push to any branch            │  Run CI (quality checks)       │
│  Pull request opened/updated   │  Run CI (block merge if fail)  │
│  Push to main branch           │  Run CI + CD (check + deploy)  │
│  Manual trigger (button click) │  Run whatever you want         │
│  Scheduled (cron)              │  Run nightly tests, reports    │
│  Tag created (v1.0.0)          │  Build release artifacts       │
│                                │                                │
│  Most common for backend:                                       │
│  ──────────────────────────                                     │
│  • CI runs on: push + pull_request (every code change)          │
│  • CD runs on: push to main only (only verified, merged code)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Connection to what you've learned:**

> "Remember branch protection from Week 1, Lecture 4? You learned that you can block merges until reviews are approved. CI adds another layer: you can block merges until ALL automated checks pass. No green checkmark, no merge. Even if the PR has two approvals."

```
┌─────────────────────────────────────────────────────────────────┐
│             BRANCH PROTECTION + CI                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pull Request: "Add user search endpoint"                       │
│                                                                 │
│  ☑ 2/2 reviewers approved                                       │
│  ☑ CI: lint ✅                                                   │
│  ☑ CI: format ✅                                                 │
│  ☑ CI: type check ✅                                             │
│  ☑ CI: tests (47 passed) ✅                                      │
│                                                                 │
│  [ Merge pull request ]  ← Button is GREEN. Merge allowed.      │
│                                                                 │
│                                                                 │
│  Pull Request: "Fix login bug"                                  │
│                                                                 │
│  ☑ 2/2 reviewers approved                                       │
│  ☑ CI: lint ✅                                                   │
│  ☑ CI: format ✅                                                 │
│  ☒ CI: type check ❌  (1 error)                                  │
│  ☒ CI: tests (2 failed) ❌                                       │
│                                                                 │
│  [ Merge pull request ]  ← Button is GREY. Merge BLOCKED.       │
│                                                                 │
│  Even with 2 approvals, the machine says NO.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Runners (Fresh Workers, Every Shift)

**This concept trips up every beginner. Understand it now.**

```
┌─────────────────────────────────────────────────────────────────┐
│                       RUNNERS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A "runner" is the MACHINE that executes your pipeline.         │
│                                                                 │
│  When your pipeline triggers, GitHub:                           │
│                                                                 │
│  1. Spins up a FRESH virtual machine (Ubuntu, typically)        │
│  2. The VM has NOTHING on it — no Python, no code, no deps      │
│  3. Your pipeline installs everything from scratch              │
│  4. Runs all checks                                             │
│  5. Reports results                                             │
│  6. DESTROYS the VM                                             │
│                                                                 │
│  Next run? Brand new VM. Nothing persists.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why this matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  YOUR LAPTOP                      │  CI RUNNER                  │
│  ────────────────                 │  ─────────                  │
│                                   │                             │
│  Python installed months ago      │  Python installed NOW       │
│  Dependencies installed ONCE      │  Dependencies installed NOW │
│  Old test database lying around   │  Fresh database every time  │
│  Random .env file from last week  │  Only explicit env vars     │
│  "Works on my machine" ✅         │  Works on a CLEAN machine   │
│                                   │                             │
│  Your laptop accumulates state.   │  The runner has NO state.   │
│  Tests pass because of leftover   │  Tests pass because the     │
│  config, cached data, or luck.    │  code is actually correct.  │
│                                   │                             │
└─────────────────────────────────────────────────────────────────┘
```

**Factory analogy:**

> "Think of runners as factory workers who show up fresh every shift. They don't remember what they did yesterday. They follow the blueprint on the wall, step by step, from scratch. That's the point — the blueprint must be complete. If you forgot to write a step, the worker can't magically know what to do."

**This is why CI catches bugs your laptop didn't:**

> "When a test passes on your laptop but fails in CI, the test was never really passing — it was relying on something specific to your environment. CI tells you the truth. Your laptop lies to you."

---

# PART 3: GITHUB ACTIONS SYNTAX

## 3.1 YAML — You Already Know This (Connection to Lecture 1)

**You've been reading YAML since Tuesday.**

> "Remember Docker Compose from Lecture 1? That was YAML. GitHub Actions uses the exact same format. Same indentation rules, same key-value structure. If you could read a `docker-compose.yml`, you can read a workflow file."

```yaml
# Docker Compose (Lecture 1 — you read this already)
services:
  api:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - db

# GitHub Actions (same language, different purpose)
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        run: echo "Hello from CI"
```

**YAML in 60 seconds — just what you need:**

```yaml
# Key-value pairs (like Python dict)
name: CI Pipeline
version: 3

# Nesting via indentation (like Python!)
database:
  host: localhost
  port: 5432

# Lists with dashes
fruits:
  - apple
  - banana
  - cherry

# Multi-line strings with | (pipe)
script: |
  echo "Line 1"
  echo "Line 2"
  echo "Each line runs separately"

# Booleans
enabled: true
verbose: false

# Comments with #
# This is a comment, just like Python
```

**The one YAML gotcha:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  YAML cares about INDENTATION, just like Python.                │
│  But YAML uses SPACES ONLY. Never tabs. 2-space indent.         │
│                                                                 │
│  ❌ WRONG:                                                      │
│  jobs:                                                          │
│  ··test:           ← Wrong indent level                         │
│  ····steps:                                                     │
│                                                                 │
│  ✅ CORRECT:                                                    │
│  jobs:                                                          │
│    test:           ← 2-space indent under jobs                  │
│      steps:        ← 2-space indent under test                  │
│                                                                 │
│  Your IDE will handle this. But if CI gives you a "YAML parse   │
│  error," check your indentation first.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Workflow Files (The Factory Blueprints)

**A workflow file is a YAML file that tells GitHub Actions what to do.**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHERE WORKFLOWS LIVE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  your-project/                                                  │
│  ├─ .github/                                                    │
│  │   └─ workflows/          ← GitHub looks HERE for workflows   │
│  │       ├─ ci.yml          ← CI pipeline (lint, test, etc.)    │
│  │       └─ cd.yml          ← CD pipeline (build, deploy)       │
│  ├─ app/                                                        │
│  │   ├─ main.py                                                 │
│  │   ├─ routes/                                                 │
│  │   └─ ...                                                     │
│  ├─ tests/                                                      │
│  ├─ Dockerfile                                                  │
│  ├─ docker-compose.yml                                          │
│  └─ pyproject.toml                                              │
│                                                                 │
│  Rules:                                                         │
│  • Must be in .github/workflows/ (exact path)                   │
│  • Must be .yml or .yaml extension                              │
│  • Can have multiple workflow files (they run independently)    │
│  • File name is up to you (ci.yml, test.yml, deploy.yml, etc.) │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Factory analogy:**

> "The `.github/workflows/` directory is the bulletin board on the factory floor. Each `.yml` file is a different blueprint — one for quality inspection, one for shipping. The factory workers (runners) read these blueprints and follow them exactly."

---

## 3.3 Your First Workflow

**The simplest possible pipeline:**

```yaml
# .github/workflows/hello.yml

name: Hello CI                # Name shown in GitHub UI

on: push                      # Trigger: run on every push

jobs:                         # What to do
  greet:                      # Job name (you choose)
    runs-on: ubuntu-latest    # Machine type (fresh Ubuntu VM)
    steps:                    # Ordered list of actions
      - name: Say hello       # Step label (for readability)
        run: echo "Hello from CI!"
```

**Push this to GitHub. Go to the "Actions" tab. Watch it run.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHAT HAPPENS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. You push code to GitHub                                     │
│                                                                 │
│  2. GitHub sees .github/workflows/hello.yml                     │
│     "There's a blueprint. Trigger says 'on: push.' Let's go."  │
│                                                                 │
│  3. GitHub spins up a fresh Ubuntu VM (the runner)              │
│                                                                 │
│  4. Runner reads the workflow file:                              │
│     - Job: "greet"                                              │
│     - Step 1: "Say hello" → run: echo "Hello from CI!"         │
│                                                                 │
│  5. Runner executes: echo "Hello from CI!"                      │
│     Output: Hello from CI!                                      │
│                                                                 │
│  6. All steps passed → Job passes → Workflow passes             │
│     Green checkmark ✅ appears on your commit.                   │
│                                                                 │
│  7. VM is destroyed. Gone forever.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**That's it. That's a CI pipeline.** Everything else is just adding more steps.

---

## 3.4 Jobs and Steps (The Building Blocks)

**A workflow has three levels: Workflow → Jobs → Steps.**

```
┌─────────────────────────────────────────────────────────────────┐
│               WORKFLOW → JOBS → STEPS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WORKFLOW (the whole file)                                      │
│  ├─ name, triggers, permissions                                 │
│  │                                                              │
│  ├─ JOB 1: "lint" (runs on its own runner)                      │
│  │   ├─ Step 1: Checkout code                                   │
│  │   ├─ Step 2: Install Python                                  │
│  │   └─ Step 3: Run linter                                      │
│  │                                                              │
│  ├─ JOB 2: "test" (runs on a DIFFERENT runner)                  │
│  │   ├─ Step 1: Checkout code                                   │
│  │   ├─ Step 2: Install Python                                  │
│  │   ├─ Step 3: Install dependencies                            │
│  │   └─ Step 4: Run tests                                       │
│  │                                                              │
│  └─ JOB 3: "deploy" (needs job 1 + 2 to pass first)            │
│      ├─ Step 1: Checkout code                                   │
│      ├─ Step 2: Build Docker image                              │
│      └─ Step 3: Deploy                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key rules:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  JOBS:                                                          │
│  • Run in PARALLEL by default (different VMs!)                  │
│  • Use "needs:" to create dependencies between jobs             │
│  • Each job gets a FRESH runner (no shared state)               │
│                                                                 │
│  STEPS:                                                         │
│  • Run in SEQUENCE within a job (same VM)                       │
│  • Share the same filesystem within the job                     │
│  • If a step fails, all subsequent steps are SKIPPED            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**In YAML:**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Job 1 and Job 2 run in PARALLEL (different VMs)
  lint:
    runs-on: ubuntu-latest
    steps:                       # Steps run in SEQUENCE (same VM)
      - name: Step 1
        run: echo "linting..."
      - name: Step 2
        run: echo "more linting..."

  test:
    runs-on: ubuntu-latest       # Different VM from lint!
    steps:
      - name: Step 1
        run: echo "testing..."

  # Job 3 waits for Job 1 AND Job 2
  deploy:
    needs: [lint, test]          # Runs ONLY if both pass
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: echo "deploying..."
```

**Visualize the execution:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   EXECUTION FLOW                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Time ──────────────────────────────────────────────────▶       │
│                                                                 │
│  lint:   [Step 1 → Step 2 → ✅]                                │
│                                  \                              │
│                                   ▶─── deploy: [Step 1 → ✅]   │
│                                  /                              │
│  test:   [Step 1 → ✅]─────────                                │
│                                                                 │
│  lint and test run SIMULTANEOUSLY (parallel).                   │
│  deploy waits for BOTH to finish (needs: [lint, test]).         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Using Actions (Pre-Built Machines)

**You don't build every tool from scratch. You use pre-built ones.**

> "In the factory analogy, you don't build your own X-ray machine. You buy one and install it on the line. GitHub Actions has a marketplace of pre-built 'actions' — someone else built the tool, you just plug it in."

```yaml
steps:
  # A "run" step: you write the command yourself
  - name: Say hello
    run: echo "Hello"

  # A "uses" step: you use a PRE-BUILT action
  - name: Check out code
    uses: actions/checkout@v4      # Someone else wrote this
```

**The most common actions you'll use:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  ESSENTIAL ACTIONS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  actions/checkout@v4                                            │
│  ─────────────────────                                          │
│  Clones your repository onto the runner.                        │
│  Without this, the runner has NO code.                          │
│  Almost EVERY workflow starts with this.                        │
│                                                                 │
│  astral-sh/setup-uv@v5                                          │
│  ─────────────────────                                          │
│  Installs uv on the runner.                                     │
│  Handles caching of dependencies automatically.                 │
│  Then you use "uv sync" and "uv run" like normal.              │
│                                                                 │
│  docker/login-action@v3                                         │
│  ──────────────────────                                         │
│  Logs into a container registry (Docker Hub, GHCR).             │
│  Required before pushing Docker images.                         │
│                                                                 │
│  docker/build-push-action@v6                                    │
│  ───────────────────────────                                    │
│  Builds and pushes a Docker image in one step.                  │
│  Handles multi-stage builds, layer caching.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Action versioning — the `@v4` part:**

```yaml
# The @v4 pins to a MAJOR version (like a dependency version)
- uses: actions/checkout@v4      # Latest v4.x.x (stable, safe)

# You CAN pin to exact commit (most secure, but overkill for now)
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

# ❌ DON'T use @main or @latest — these change without warning
- uses: actions/checkout@main    # Could break your pipeline tomorrow
```

---

## 3.6 Secrets and Variables (Connection to Lecture 2)

**Connection to what you've learned:**

> "In Lecture 2, you learned about secrets management with pydantic-settings: never commit secrets, use environment variables, validate them at startup. CI/CD follows the same principle — but secrets are stored in GitHub, not in `.env` files."

```
┌─────────────────────────────────────────────────────────────────┐
│                  SECRETS IN CI/CD                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  YOUR LAPTOP (Lecture 2):                                       │
│  ─────────────────────────                                      │
│  .env file → pydantic-settings reads it → app gets config       │
│                                                                 │
│  CI/CD RUNNER:                                                  │
│  ─────────────                                                  │
│  No .env file exists (runner is fresh, .env is gitignored).     │
│  Instead: GitHub Secrets → injected as environment variables    │
│           → your app reads them normally.                       │
│                                                                 │
│  HOW TO SET A SECRET:                                           │
│  GitHub repo → Settings → Secrets and variables → Actions       │
│  → New repository secret → Name: DATABASE_URL, Value: ...       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Using secrets in workflows:**

```yaml
steps:
  - name: Run tests
    env:
      # ${{ secrets.NAME }} pulls from GitHub Secrets
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
      REDIS_URL: ${{ secrets.REDIS_URL }}
      JWT_SECRET: ${{ secrets.JWT_SECRET }}
    run: uv run pytest
```

**The `${{ }}` syntax — template expressions:**

```yaml
# Think of ${{ }} like Python f-strings, but for YAML

# Access secrets
${{ secrets.DATABASE_URL }}       # A secret you stored in GitHub

# Access built-in context
${{ github.ref }}                 # "refs/heads/main"
${{ github.actor }}               # "your-username"
${{ github.repository }}          # "your-username/your-repo"

# Conditional logic
${{ github.ref == 'refs/heads/main' }}    # true or false
```

**Security rules:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ✅ Secrets are ENCRYPTED in GitHub. Masked in logs.            │
│  ✅ Only your repo's workflows can access your secrets.         │
│  ✅ Use secrets for: DATABASE_URL, API keys, JWT_SECRET,        │
│     deploy tokens, registry passwords.                          │
│                                                                 │
│  ❌ NEVER echo a secret in a run step (it will be masked,       │
│     but don't rely on it).                                      │
│  ❌ NEVER hardcode secrets in the YAML file.                    │
│  ❌ NEVER print secrets for debugging.                          │
│                                                                 │
│  GITHUB_TOKEN is special — automatically available, no setup.   │
│  It lets your workflow push images to GitHub Container Registry, │
│  create comments on PRs, and interact with the GitHub API.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: THE CI PIPELINE

## 4.1 Quality Gates (The Inspection Stations)

**Each quality gate answers a specific question about your code.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   QUALITY GATES                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GATE          TOOL               QUESTION IT ANSWERS           │
│  ─────         ────               ───────────────────           │
│                                                                 │
│  Lint          ruff check         "Does the code follow         │
│                                    quality rules? Any bugs      │
│                                    detectable by pattern?"      │
│                                                                 │
│  Format        ruff format        "Is the code formatted        │
│                                    consistently? Would every    │
│                                    developer read it the same?" │
│                                                                 │
│  Type Check    mypy --strict      "Do the types line up?        │
│                                    Will this crash at runtime   │
│                                    from a type mismatch?"       │
│                                                                 │
│  Test          pytest + coverage  "Does the code WORK?          │
│                                    Do all behaviors produce     │
│                                    correct results?"            │
│                                                                 │
│  Each gate catches a DIFFERENT class of defect.                 │
│  Together, they form a comprehensive safety net.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why this order?**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  FAST checks first. SLOW checks last.                           │
│                                                                 │
│  ruff check:       ~1 second     (pattern matching)             │
│  ruff format:      ~1 second     (syntax parsing)               │
│  mypy --strict:    ~10-30 seconds (full type analysis)          │
│  pytest:           ~30s-5 minutes (actual code execution)       │
│                                                                 │
│  If linting fails in 1 second, why wait 5 minutes for tests?    │
│  Fix the lint error first. Then worry about correctness.        │
│                                                                 │
│  Factory analogy: You check the PAINT COLOR before doing        │
│  a crash test. Why crash-test a product that's the wrong color? │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Linting and Formatting with ruff (One Tool to Rule Them All)

**ruff is a single tool that replaces an entire ecosystem.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE OLD WORLD                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Before ruff, Python projects needed MULTIPLE tools:            │
│                                                                 │
│  Flake8          → Linting (style + error detection)            │
│  isort           → Import sorting                               │
│  pyupgrade       → Modernizing old syntax                       │
│  Black           → Code formatting                              │
│  Bandit          → Security linting                             │
│  pydocstyle      → Docstring linting                            │
│  + dozens of Flake8 plugins                                     │
│                                                                 │
│  Each tool: separate install, separate config, separate run.    │
│  Your CI had 4-6 steps just for code style.                     │
│                                                                 │
│                                                                 │
│                    THE NEW WORLD                                │
│  ──────────────────────────────────                             │
│                                                                 │
│  ruff check      → Replaces Flake8, isort, pyupgrade, Bandit,  │
│                    pydocstyle, and 700+ rules from 50+ plugins  │
│                                                                 │
│  ruff format     → Replaces Black (consistent formatting)       │
│                                                                 │
│  ONE tool. ONE config. TWO commands. Done.                      │
│  And it runs 10-100x faster (written in Rust).                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What linting catches (ruff check):**

```python
# ruff check catches defects like these:

# F841: Local variable assigned but never used
def process_data(items: list[dict]) -> int:
    unused_var = 42            # ← ruff: F841
    return len(items)

# F811: Redefinition of unused name
import os                      # ← ruff: F811 (redefined below)
import os

# I001: Import block is un-sorted
import os                      # ← ruff: I001 (should be above)
import asyncio                 #    asyncio comes before os

# UP006: Use list instead of List (modern Python)
from typing import List        # ← ruff: UP006
def get_items() -> List[str]:  #    Use list[str] instead
    ...

# B006: Mutable default argument
def add_item(items: list = []):  # ← ruff: B006 (shared state bug!)
    items.append("new")
    return items
```

**What formatting catches (ruff format --check):**

```python
# ruff format enforces CONSISTENT style:

# Before formatting (inconsistent, messy):
x=1
y =2
z   =    3
long_dict = {"key1":"value1","key2":"value2","key3":"value3","key4":"value4"}

# After ruff format (consistent, readable):
x = 1
y = 2
z = 3
long_dict = {
    "key1": "value1",
    "key2": "value2",
    "key3": "value3",
    "key4": "value4",
}
```

**Two different modes:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ruff format .                                                  │
│  ─────────────                                                  │
│  MODIFIES files in place. Use during development.               │
│  "Fix the formatting for me."                                   │
│                                                                 │
│  ruff format --check .                                          │
│  ──────────────────────                                         │
│  CHECKS but does NOT modify. Use in CI.                         │
│  "Tell me if anything is wrong. Don't fix it."                  │
│  Exits with error code 1 if any file would be reformatted.     │
│  This FAILS the CI pipeline — forcing the developer to fix it. │
│                                                                 │
│  Same for linting:                                              │
│  ruff check .          → Report problems (CI mode)              │
│  ruff check --fix .    → Auto-fix safe problems (dev mode)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Configuring ruff in pyproject.toml:**

> "Your `pyproject.toml` already manages your project (uv, from Week 1). ruff reads its config from the same file. Single source of truth."

```toml
# pyproject.toml — add these sections

[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors (basic style rules)
    "F",    # pyflakes (unused imports, undefined names)
    "I",    # isort (import sorting)
    "UP",   # pyupgrade (use modern Python syntax)
    "B",    # flake8-bugbear (common bugs and design issues)
    "SIM",  # flake8-simplify (simplifiable code patterns)
    "N",    # pep8-naming (naming conventions)
]

[tool.ruff.lint.isort]
known-first-party = ["app"]    # Your package name — isort groups it correctly
```

**The CI steps for ruff:**

```yaml
# In your workflow:
steps:
  # ... checkout, install uv, install deps ...

  - name: Lint
    run: uv run ruff check .

  - name: Format check
    run: uv run ruff format --check .
```

**When these fail, the output tells you exactly what to fix:**

```
$ uv run ruff check .
app/routes/tasks.py:15:5: F841 Local variable `result` is assigned to but never used
app/services/auth.py:3:1: I001 Import block is un-sorted or un-formatted
Found 2 errors.

$ uv run ruff format --check .
Would reformat: app/routes/tasks.py
Would reformat: app/models/user.py
2 files would be reformatted, 18 files already formatted.
```

> "The developer sees this in the PR checks, opens their terminal, runs `ruff check --fix .` and `ruff format .`, commits the fix, pushes. CI re-runs. Gate passes."

---

## 4.3 Type Checking with mypy (Connection to Week 1, Lecture 1)

**Connection to what you've learned:**

> "Week 1, Lecture 1: you learned type hints. Week 1 onward: you've been adding type annotations to every function. Now mypy in CI enforces it. Every push, every PR — the machine verifies your types are correct. No more 'I forgot to run mypy.'"

```yaml
# The CI step:
- name: Type check
  run: uv run mypy . --strict
```

**What `--strict` catches that non-strict misses:**

```python
# Without --strict, mypy is LENIENT. Too lenient.
# These all pass mypy without --strict:

def process(data):              # No type annotation — mypy ignores it
    return data["key"]          # Could crash if data isn't a dict

def fetch_user(user_id):        # No return type — mypy doesn't check callers
    return {"id": user_id}

# With --strict, mypy REQUIRES annotations and checks everything:

def process(data):              # error: missing type annotation
    return data["key"]

def fetch_user(user_id):        # error: missing return type
    return {"id": user_id}

# Correct:
def process(data: dict[str, str]) -> str:
    return data["key"]

def fetch_user(user_id: int) -> dict[str, int]:
    return {"id": user_id}
```

**Configure mypy in pyproject.toml:**

```toml
# pyproject.toml

[tool.mypy]
strict = true
plugins = ["pydantic.mypy"]    # Understands Pydantic models

# If you use SQLAlchemy, you may also need:
[[tool.mypy.overrides]]
module = "celery.*"            # Some libraries lack type stubs
ignore_missing_imports = true
```

**When mypy fails in CI:**

```
$ uv run mypy . --strict
app/services/user.py:25: error: Function is missing a return type
    annotation  [no-untyped-def]
app/routes/tasks.py:42: error: Argument 1 to "get_user" has
    incompatible type "str"; expected "int"  [arg-type]
Found 2 errors in 2 files (checked 24 source files)
```

> "This is the same gate that would have caught the TypeError in the Friday night deployment scenario. The machine catches it. The merge is blocked. Production is safe."

---

## 4.4 Testing with pytest + Coverage (Connection to Week 2, Lecture 2)

**Connection to what you've learned:**

> "Week 2, Lecture 2: you learned pytest, fixtures, parametrize, mocking. You've been writing tests for 13 weeks. Now CI runs them ALL, every time, on a clean machine. If someone pushes code that breaks a test they didn't even know existed — CI catches it."

```yaml
# The CI step:
- name: Test with coverage
  run: uv run pytest --cov=app --cov-report=term-missing -q
```

**What `--cov` adds:**

```
$ uv run pytest --cov=app --cov-report=term-missing -q

47 passed in 12.34s

---------- coverage: platform linux, python 3.12.8 -----------
Name                          Stmts   Miss  Cover   Missing
─────────────────────────────────────────────────────────────
app/main.py                      12      0   100%
app/routes/tasks.py              45      3    93%   78-80
app/routes/users.py              38      0   100%
app/services/auth.py             52      8    85%   34-41
app/services/task_service.py     67      2    97%   112-113
app/models/user.py               23      0   100%
─────────────────────────────────────────────────────────────
TOTAL                           237     13    95%
```

**Reading the coverage report:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  COVERAGE REPORT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Stmts:   Total executable statements in the file              │
│  Miss:    Statements NOT executed by any test                   │
│  Cover:   Percentage of statements covered (Stmts-Miss)/Stmts  │
│  Missing: Exact line numbers not covered                        │
│                                                                 │
│  app/services/auth.py  85%  Missing: 34-41                      │
│  → Lines 34-41 in auth.py have NO test that executes them.      │
│  → Probably an error handling branch nobody tested.             │
│                                                                 │
│  GOAL: 80%+ coverage (project requirement from Week 2).         │
│  Not 100% — chasing 100% leads to worthless tests.              │
│  But 80% means most important paths are verified.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Enforcing minimum coverage in CI:**

```yaml
# Fail CI if coverage drops below 80%:
- name: Test with coverage
  run: uv run pytest --cov=app --cov-fail-under=80 -q
```

If coverage is 79%, the step exits with a non-zero code. CI fails. The developer must add tests before merging.

---

## 4.5 Service Containers (Running PostgreSQL in CI)

**Your integration tests need a real database. The runner doesn't have one.**

> "Your laptop has Docker with PostgreSQL running (from Lecture 1). The CI runner has nothing. We need to spin up a PostgreSQL container inside CI so integration tests can run against a real database."

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    # Service containers — spin up alongside the job
    services:
      postgres:
        image: postgres:16                  # Same version as production
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432                       # Accessible on localhost:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      # ... checkout, install uv, install deps ...

      - name: Run tests
        env:
          DATABASE_URL: postgresql+asyncpg://test:test@localhost:5432/test_db
        run: uv run pytest --cov=app --cov-fail-under=80 -q
```

**What's happening here:**

```
┌─────────────────────────────────────────────────────────────────┐
│                SERVICE CONTAINERS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────┐                │
│  │               CI Runner (Ubuntu VM)         │                │
│  │                                             │                │
│  │  ┌─────────────────┐  ┌──────────────────┐  │                │
│  │  │  Your pipeline  │  │  postgres:16     │  │                │
│  │  │  (checkout,     │  │  (test database) │  │                │
│  │  │   install,      │  │                  │  │                │
│  │  │   lint, test)   │──│  localhost:5432  │  │                │
│  │  │                 │  │                  │  │                │
│  │  └─────────────────┘  └──────────────────┘  │                │
│  │                                             │                │
│  └─────────────────────────────────────────────┘                │
│                                                                 │
│  The service container starts BEFORE your steps run.            │
│  The health check ensures Postgres is READY before tests start. │
│  When the job ends, the container is destroyed.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**You can add Redis the same way:**

```yaml
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: test_db
    ports:
      - 5432:5432
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5

  redis:
    image: redis:7
    ports:
      - 6379:6379
    options: >-
      --health-cmd "redis-cli ping"
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

> "This mirrors your `docker-compose.yml` from Lecture 1 — same databases, same ports, same health checks. Except here, it's automated by GitHub for every push. You never have to remember to start the database."

---

## 4.6 Caching Dependencies (Don't Rebuild the Factory Every Morning)

**Problem: installing dependencies every run is slow.**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  WITHOUT CACHING:                                               │
│                                                                 │
│  Every CI run:                                                  │
│  1. Download Python            (~10s)                           │
│  2. Download uv                (~3s)                            │
│  3. Download ALL dependencies  (~30-60s)                        │
│  4. Install them               (~15-30s)                        │
│                                                                 │
│  Total setup: ~60-100 seconds BEFORE any check runs.            │
│  And it's the SAME dependencies as last run.                    │
│                                                                 │
│                                                                 │
│  WITH CACHING:                                                  │
│                                                                 │
│  First run:                                                     │
│  1. Download + install everything (~100s)                       │
│  2. SAVE the cache                                              │
│                                                                 │
│  Subsequent runs (if lockfile unchanged):                       │
│  1. RESTORE from cache           (~5s)                          │
│  2. Skip download + install                                     │
│                                                                 │
│  Total setup: ~5-10 seconds. 10x faster.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**With uv, caching is one line:**

```yaml
- name: Install uv
  uses: astral-sh/setup-uv@v5
  with:
    enable-cache: true           # ← This one line caches everything
```

**How it works:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  astral-sh/setup-uv with enable-cache: true                     │
│                                                                 │
│  1. Checks: "Is there a saved cache matching uv.lock?"          │
│  2. If YES → Restores cached packages. Skip downloads.          │
│  3. If NO  → Downloads fresh. Saves cache for next run.         │
│                                                                 │
│  Cache key is based on uv.lock — if your dependencies           │
│  change, the cache is invalidated and rebuilt.                  │
│                                                                 │
│  This is why "uv lock" and committing uv.lock matters           │
│  (Week 1, Lecture 4). The lockfile is the cache key.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.7 The Complete CI Pipeline

**Everything together. Read it step by step.**

```yaml
# .github/workflows/ci.yml

name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      # ── SETUP ──────────────────────────────────────────────────
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install

      - name: Install dependencies
        run: uv sync --frozen

      # ── QUALITY GATES (fast → slow) ───────────────────────────
      - name: Lint
        run: uv run ruff check .

      - name: Format check
        run: uv run ruff format --check .

      - name: Type check
        run: uv run mypy . --strict

      # ── TESTS ──────────────────────────────────────────────────
      - name: Run tests
        env:
          DATABASE_URL: postgresql+asyncpg://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0
          JWT_SECRET: test-secret-not-for-production
          ENVIRONMENT: test
        run: uv run pytest --cov=app --cov-fail-under=80 -q
```

**Walk through what happens:**

```
┌─────────────────────────────────────────────────────────────────┐
│               EXECUTION TIMELINE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  0s    Developer pushes code to PR                              │
│  2s    GitHub sees workflow file, spins up runner               │
│  5s    Services starting (Postgres, Redis)                      │
│  8s    Checkout code onto runner                                │
│  12s   Install uv (from cache: instant)                         │
│  14s   Install Python                                           │
│  18s   Install dependencies (from cache: ~5s)                   │
│  20s   Lint → ruff check . (1s)                                 │
│  21s   Format check → ruff format --check . (1s)                │
│  22s   Type check → mypy . --strict (15s)                       │
│  37s   Tests → pytest (60s)                                     │
│  97s   ALL PASSED → Green checkmark ✅                           │
│                                                                 │
│  Total: ~90-120 seconds for full quality verification.          │
│  Every push. Every PR. Automatically. No humans remembering.    │
│                                                                 │
│  If ANYTHING fails at any step:                                 │
│  → Remaining steps are SKIPPED                                  │
│  → Red X ❌ appears on PR                                        │
│  → Merge is BLOCKED                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: THE CD PIPELINE

## 5.1 From CI to Deployment (The Shipping Department)

**CI verified the product. Now it's time to ship.**

```
┌─────────────────────────────────────────────────────────────────┐
│              CI → CD HANDOFF                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  Feature Branch                    main Branch                  │
│  ──────────────                    ───────────                  │
│                                                                 │
│  Push to PR ──▶ CI runs            Push to main ──▶ CI runs     │
│                 ✅ Pass                              ✅ Pass     │
│                 (no deploy)                          │           │
│                                                     ▼           │
│                                              CD runs            │
│                                              📦 Build image     │
│                                              📤 Push to registry│
│                                              🗄️  Run migrations  │
│                                              🚀 Deploy          │
│                                                                 │
│  Key distinction:                                               │
│  • Feature branches: CI only (verify quality)                   │
│  • main branch: CI + CD (verify AND deploy)                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The `needs` keyword enforces the gate:**

```yaml
jobs:
  ci:
    # ... all quality checks from section 4.7 ...

  cd:
    needs: ci                       # CD runs ONLY if CI passes
    if: github.ref == 'refs/heads/main'  # AND only on main branch
    runs-on: ubuntu-latest
    steps:
      # ... build and deploy ...
```

> "The `needs: ci` line is the connection between the inspection line and the shipping department. If the inspection line rejects the product, the shipping department never sees it."

---

## 5.2 Build and Push Docker Images (Connection to Lecture 1)

**Connection to what you've learned:**

> "In Lecture 1, you wrote a Dockerfile and built images with `docker build`. CD does the same thing — but automatically, and pushes the image to a registry where your server can pull it."

```
┌─────────────────────────────────────────────────────────────────┐
│              CONTAINER REGISTRY                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A container registry is like a PACKAGE WAREHOUSE.              │
│                                                                 │
│  CD builds your Docker image and PUSHES it to the registry.     │
│  Your production server PULLS the latest image from the same    │
│  registry and runs it.                                          │
│                                                                 │
│                                                                 │
│  ┌──────────┐     ┌────────────────┐     ┌──────────────┐       │
│  │ CD builds│────▶│  Container     │────▶│  Production  │       │
│  │ image    │push │  Registry      │pull │  Server      │       │
│  └──────────┘     │  (GHCR/Docker  │     │  runs image  │       │
│                   │   Hub)         │     └──────────────┘       │
│                   └────────────────┘                             │
│                                                                 │
│  Common registries:                                             │
│  • ghcr.io — GitHub Container Registry (free for public repos)  │
│  • Docker Hub — The default registry                            │
│  • AWS ECR — Amazon's registry                                  │
│                                                                 │
│  We'll use GHCR because it's built into GitHub — no extra       │
│  setup, authentication uses your GITHUB_TOKEN automatically.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The CD workflow for building and pushing:**

```yaml
# .github/workflows/cd.yml

name: CD

on:
  push:
    branches: [main]         # Only deploy from main

jobs:
  ci:
    # ... reuse or reference your CI job ...
    # (You can put CI and CD in the same file or separate files.
    #  Same file is simpler for small projects.)
    uses: ./.github/workflows/ci.yml   # Reuse CI workflow

  deploy:
    needs: ci
    runs-on: ubuntu-latest

    # Required for pushing to GHCR
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}    # Auto-provided!

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
```

**Why two tags?**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ghcr.io/yourname/yourproject:latest                            │
│  ────────────────────────────────────                           │
│  Always points to the most recent build.                        │
│  Your server pulls ":latest" to get the newest version.         │
│                                                                 │
│  ghcr.io/yourname/yourproject:a1b2c3d                           │
│  ────────────────────────────────────                           │
│  Tagged with the Git commit SHA.                                │
│  Unique to this exact version of the code.                      │
│  Use this for ROLLBACKS — "deploy commit a1b2c3d instead."      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Database Migrations in Deployment (Connection to Week 6, Lecture 3)

**Connection to what you've learned:**

> "Week 6, Lecture 3: you learned Alembic migrations. `alembic upgrade head` applies all pending migrations. In deployment, this must happen AFTER the image is built but BEFORE the new code starts serving traffic."

**The ordering problem:**

```
┌─────────────────────────────────────────────────────────────────┐
│             MIGRATION ORDERING                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ WRONG ORDER:                                                │
│                                                                 │
│  1. Deploy new code (expects new column "email_verified")       │
│  2. Run migrations (adds "email_verified" column)               │
│                                                                 │
│  For the seconds between step 1 and 2, your app CRASHES         │
│  because it queries a column that doesn't exist yet.            │
│                                                                 │
│                                                                 │
│  ✅ CORRECT ORDER:                                              │
│                                                                 │
│  1. Run migrations (adds "email_verified" column)               │
│  2. Deploy new code (column already exists, no crash)           │
│                                                                 │
│  The schema is updated FIRST. The code that uses it arrives     │
│  SECOND.                                                        │
│                                                                 │
│                                                                 │
│  ⚠️  THIS REQUIRES BACKWARD-COMPATIBLE MIGRATIONS:              │
│                                                                 │
│  While migration runs but BEFORE new code deploys, the OLD      │
│  code is still running. The migration must not break the old    │
│  code. Example:                                                 │
│                                                                 │
│  ✅ ADD a new column (old code ignores it)                       │
│  ✅ ADD a new table (old code doesn't query it)                  │
│  ❌ DROP a column (old code crashes trying to read it)           │
│  ❌ RENAME a column (old code uses the old name)                 │
│                                                                 │
│  For destructive changes: deploy in TWO steps.                  │
│  Step 1: Deploy code that doesn't use the old column.           │
│  Step 2: Migrate to drop the column.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Running migrations in CD:**

```yaml
deploy:
  needs: ci
  runs-on: ubuntu-latest
  steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Install uv
      uses: astral-sh/setup-uv@v5

    - name: Set up Python
      run: uv python install

    - name: Install dependencies
      run: uv sync --frozen

    # Migrations run BEFORE the new code serves traffic
    - name: Run database migrations
      env:
        DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
      run: uv run alembic upgrade head

    # THEN build and deploy the new code
    - name: Build and push Docker image
      uses: docker/build-push-action@v6
      with:
        context: .
        push: true
        tags: ghcr.io/${{ github.repository }}:latest

    - name: Deploy to platform
      env:
        DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
      run: |
        # Platform-specific command:
        # Railway:  railway up
        # Fly.io:   flyctl deploy --image ghcr.io/...
        # AWS ECS:  aws ecs update-service ...
        echo "Deploying..."
```

---

## 5.4 Environment-Specific Configuration (Connection to Lecture 2)

**Connection to what you've learned:**

> "In Lecture 2, you set up pydantic-settings to validate configuration from environment variables. Different environments need different values for those variables — but the app code is the SAME Docker image everywhere."

```
┌─────────────────────────────────────────────────────────────────┐
│           ONE IMAGE, MANY ENVIRONMENTS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The SAME Docker image runs in every environment.               │
│  What changes is the CONFIGURATION, not the code.               │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Same Docker Image                            │ │
│  │  ghcr.io/your/project:abc123                              │ │
│  └────────────────────────────────────────────────────────────┘ │
│        │                │                │                      │
│        ▼                ▼                ▼                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐               │
│  │   DEV    │    │ STAGING  │    │  PRODUCTION   │              │
│  │          │    │          │    │               │              │
│  │ DB: local│    │ DB: stg  │    │ DB: prod      │              │
│  │ DEBUG: 1 │    │ DEBUG: 0 │    │ DEBUG: 0      │              │
│  │ LOG: dbg │    │ LOG: info│    │ LOG: warn     │              │
│  └──────────┘    └──────────┘    └──────────────┘               │
│                                                                 │
│  Environment variables change. Code does not.                   │
│  This is a 12-Factor App principle (Lecture 2).                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Managing environments in GitHub Actions:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  GitHub has "Environments" — named sets of secrets and rules.   │
│                                                                 │
│  Settings → Environments → New environment                      │
│                                                                 │
│  Environment: "production"                                      │
│  ├─ Secrets:                                                    │
│  │   ├─ DATABASE_URL = postgresql://prod-db:5432/app            │
│  │   ├─ REDIS_URL = redis://prod-redis:6379/0                   │
│  │   └─ JWT_SECRET = (actual production secret)                 │
│  └─ Protection rules:                                           │
│      └─ Required reviewers: 1  (human must approve deploys)     │
│                                                                 │
│  Environment: "staging"                                         │
│  ├─ Secrets:                                                    │
│  │   ├─ DATABASE_URL = postgresql://stg-db:5432/app             │
│  │   ├─ REDIS_URL = redis://stg-redis:6379/0                    │
│  │   └─ JWT_SECRET = (staging secret)                           │
│  └─ Protection rules: (none — auto-deploy is fine)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Using environments in workflows:**

```yaml
jobs:
  deploy-staging:
    needs: ci
    runs-on: ubuntu-latest
    environment: staging          # Uses "staging" secrets
    steps:
      - name: Deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}   # Staging DB URL
        run: echo "Deploying to staging..."

  deploy-production:
    needs: deploy-staging         # Only after staging succeeds
    runs-on: ubuntu-latest
    environment: production       # Uses "production" secrets
                                  # May require manual approval!
    steps:
      - name: Deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}   # Production DB URL
        run: echo "Deploying to production..."
```

**The promotion flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Push to main                                                   │
│      │                                                          │
│      ▼                                                          │
│  CI (lint, format, types, tests)                                │
│      │ ✅                                                       │
│      ▼                                                          │
│  Deploy to STAGING (automatic)                                  │
│      │ ✅                                                       │
│      ▼                                                          │
│  Deploy to PRODUCTION (requires manual approval)                │
│      │                                                          │
│      │ ← Team lead clicks "Approve" in GitHub UI                │
│      │ ✅                                                       │
│      ▼                                                          │
│  Live! 🚀                                                       │
│                                                                 │
│  Staging is the "dress rehearsal." If it breaks there,          │
│  production is never touched.                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Rolling Deployments and Rollback Strategies

**What happens when the new version goes live?**

```
┌─────────────────────────────────────────────────────────────────┐
│                DEPLOYMENT STRATEGIES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RECREATE (simplest — what you'll use first)                    │
│  ───────────────────────────────────────────                    │
│  1. Stop old version                                            │
│  2. Start new version                                           │
│                                                                 │
│  ⚠️  DOWNTIME between stop and start.                           │
│     Acceptable for small projects, staging environments.        │
│                                                                 │
│  Time: ──[OLD OLD OLD]──[💀 DOWN 💀]──[NEW NEW NEW]──▶         │
│                                                                 │
│                                                                 │
│  ROLLING (zero-downtime)                                        │
│  ────────────────────────                                       │
│  1. Start new version alongside old version                     │
│  2. Gradually shift traffic from old → new                      │
│  3. Once all traffic on new → remove old                        │
│                                                                 │
│  ✅ NO downtime. Users don't notice.                            │
│                                                                 │
│  Time: ──[OLD OLD OLD]──────────────────────────▶               │
│                 [NEW NEW NEW NEW NEW NEW]──────▶                │
│          ↑ start new    ↑ shift traffic  ↑ remove old           │
│                                                                 │
│                                                                 │
│  BLUE-GREEN (two identical environments)                        │
│  ────────────────────────────────────────                       │
│  1. "Blue" is live (current version)                            │
│  2. Deploy "Green" with new version (not serving traffic)       │
│  3. Test Green                                                  │
│  4. Switch traffic from Blue → Green instantly                  │
│  5. Blue becomes the standby (ready for rollback)               │
│                                                                 │
│  ✅ Instant rollback — just switch back to Blue.                │
│                                                                 │
│                                                                 │
│  For this course: RECREATE is sufficient.                       │
│  Platforms like Railway and Fly.io handle rolling deploys       │
│  for you automatically.                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Rollback — when things go wrong after deploying:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ROLLBACK                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Deployment went wrong. Users are seeing errors. What now?      │
│                                                                 │
│  OPTION 1: Revert the Git commit                                │
│  ────────────────────────────────                               │
│  git revert HEAD                                                │
│  git push origin main                                           │
│  → CI runs → CD deploys the reverted code automatically         │
│  → Takes ~2-3 minutes                                           │
│                                                                 │
│  OPTION 2: Deploy a previous Docker image                       │
│  ─────────────────────────────────────────                      │
│  Remember the commit SHA tag?                                   │
│  ghcr.io/yourname/project:a1b2c3d  ← known good version        │
│  Point your server at this tag instead of :latest               │
│  → Takes ~30 seconds                                            │
│                                                                 │
│  OPTION 3: Platform rollback                                    │
│  ──────────────────────────                                     │
│  Railway / Fly.io have "rollback to previous deployment"        │
│  buttons in their UI. One click.                                │
│  → Takes ~10 seconds                                            │
│                                                                 │
│                                                                 │
│  This is why we tag images with commit SHAs.                    │
│  Every deploy is a snapshot you can return to.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Database rollback — the hard part:**

> "Rolling back CODE is easy — deploy an older image. Rolling back a DATABASE is hard. If your migration added a column and you populated it with data, `alembic downgrade` drops that column and the data is gone. This is why backward-compatible migrations are critical. If the migration only ADDs things, the old code simply ignores the new columns. No database rollback needed."

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SAFE ROLLBACK SCENARIO:                                        │
│                                                                 │
│  Migration: ALTER TABLE users ADD COLUMN bio TEXT;              │
│  New code:  Uses user.bio                                       │
│  Old code:  Doesn't know about bio (ignores it)                 │
│                                                                 │
│  Rollback code → old code runs → bio column still exists        │
│  but is unused → No crash. Data preserved. Safe.                │
│                                                                 │
│                                                                 │
│  DANGEROUS ROLLBACK SCENARIO:                                   │
│                                                                 │
│  Migration: ALTER TABLE users DROP COLUMN legacy_field;         │
│  New code:  Doesn't use legacy_field                            │
│  Old code:  Reads legacy_field → CRASH (column missing)         │
│                                                                 │
│  Rollback code → old code runs → queries legacy_field           │
│  → Column doesn't exist → 500 errors. Broken.                  │
│                                                                 │
│  Rule: Never drop or rename columns in the same deploy          │
│  as the code change. Do it in a SEPARATE, later migration       │
│  after confirming the old code is no longer running.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 6: COMMON MISTAKES AND MISCONCEPTIONS

### Mistake 1: Committing secrets to the workflow file

```yaml
# ❌ NEVER DO THIS:
- name: Deploy
  env:
    DATABASE_URL: postgresql://admin:s3cretP@ss@prod-db:5432/app
    # This is in your Git history FOREVER. Even if you delete it.

# ✅ ALWAYS use GitHub Secrets:
- name: Deploy
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

> "If you accidentally commit a secret: rotate it immediately. Change the password. Revoke the API key. Assume it's compromised. Git history is public on public repos and difficult to scrub even on private ones."

---

### Mistake 2: Using blocking calls in async tests under CI

```python
# ❌ Your tests pass locally but fail in CI with timeout errors.
# Reason: you used time.sleep() in async test code (Lecture 3, Week 1).
# The event loop is blocked, and CI runners are slower than your laptop.

async def test_with_delay():
    time.sleep(5)              # Blocks the event loop in CI!
    result = await fetch_data()
    assert result is not None

# ✅ Use asyncio.sleep() — you've known this since Week 1, Lecture 3.
async def test_with_delay():
    await asyncio.sleep(5)
    result = await fetch_data()
    assert result is not None
```

---

### Mistake 3: Forgetting `--frozen` when installing dependencies

```yaml
# ❌ WRONG: uv sync without --frozen
- name: Install dependencies
  run: uv sync
  # This might UPDATE your lockfile if pyproject.toml changed.
  # CI should NEVER modify the lockfile. It should use it as-is.

# ✅ CORRECT: --frozen means "use the lockfile exactly, fail if outdated"
- name: Install dependencies
  run: uv sync --frozen
  # If uv.lock is out of date, this FAILS — forcing the developer
  # to run "uv lock" locally and commit the updated lockfile.
```

> "CI reproduces what you committed. If the lockfile doesn't match the project definition, that's a bug in your commit, not something CI should silently fix."

---

### Mistake 4: Ignoring failing CI ("I'll fix it later")

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "CI is red but I know my code works. I'll merge anyway."       │
│                                                                 │
│  This is the pipeline equivalent of unplugging a smoke alarm    │
│  because the beeping is annoying. The alarm is telling you      │
│  something. Listen.                                             │
│                                                                 │
│  If CI is red:                                                  │
│  1. Read the error. It tells you exactly what failed.           │
│  2. Fix it locally. Push again.                                 │
│  3. If the check is WRONG (flaky test, etc.), fix the CHECK.   │
│     Don't bypass it.                                            │
│                                                                 │
│  Branch protection should make bypassing physically impossible: │
│  Settings → Branches → Branch protection rules → main:          │
│    ☑ Require status checks to pass before merging               │
│    ☑ Require branches to be up to date                          │
│                                                                 │
│  If the merge button is grey, you CANNOT ignore CI.             │
│  That's the point.                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Mistake 5: Running migrations before CI passes

```yaml
# ❌ DANGEROUS: Migration step doesn't depend on CI
jobs:
  ci:
    # ... quality checks ...

  migrate:
    runs-on: ubuntu-latest     # No "needs: ci" — runs in PARALLEL with CI!
    steps:
      - name: Run migrations
        run: uv run alembic upgrade head
        # If CI fails, this already modified the production database.
        # You've changed the schema for code that will never deploy.

# ✅ SAFE: Migration runs only after CI passes
jobs:
  ci:
    # ... quality checks ...

  deploy:
    needs: ci                  # Waits for CI to pass
    steps:
      - name: Run migrations
        run: uv run alembic upgrade head
      - name: Deploy
        run: echo "Deploying..."
```

> "Migrations change production data. They are irreversible in practice. They must NEVER run for code that hasn't passed all quality gates. The `needs` keyword is your safety net."

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                   CI/CD QUICK REFERENCE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WORKFLOW LOCATION:                                             │
│      .github/workflows/ci.yml                                   │
│                                                                 │
│  TRIGGER ON PUSH + PR:                                          │
│      on:                                                        │
│        push:                                                    │
│          branches: [main]                                       │
│        pull_request:                                            │
│          branches: [main]                                       │
│                                                                 │
│  SETUP STEPS (almost every workflow):                           │
│      - uses: actions/checkout@v4                                │
│      - uses: astral-sh/setup-uv@v5                              │
│        with:                                                    │
│          enable-cache: true                                     │
│      - run: uv python install                                   │
│      - run: uv sync --frozen                                    │
│                                                                 │
│  QUALITY GATES:                                                 │
│      uv run ruff check .             # Lint                     │
│      uv run ruff format --check .    # Format                   │
│      uv run mypy . --strict          # Type check               │
│      uv run pytest --cov=app         # Test + coverage          │
│                         --cov-fail-under=80                     │
│                                                                 │
│  SERVICE CONTAINERS (Postgres):                                 │
│      services:                                                  │
│        postgres:                                                │
│          image: postgres:16                                     │
│          env: { POSTGRES_USER: test, POSTGRES_PASSWORD: test,   │
│                 POSTGRES_DB: test_db }                           │
│          ports: ["5432:5432"]                                   │
│                                                                 │
│  JOB DEPENDENCIES:                                              │
│      deploy:                                                    │
│        needs: ci                                                │
│        if: github.ref == 'refs/heads/main'                      │
│                                                                 │
│  SECRETS:                                                       │
│      env:                                                       │
│        DATABASE_URL: ${{ secrets.DATABASE_URL }}                 │
│                                                                 │
│  PYPROJECT.TOML CONFIG:                                         │
│      [tool.ruff]                                                │
│      target-version = "py312"                                   │
│      [tool.ruff.lint]                                           │
│      select = ["E", "F", "I", "UP", "B", "SIM", "N"]           │
│      [tool.mypy]                                                │
│      strict = true                                              │
│                                                                 │
│  DEPLOYMENT ORDER:                                              │
│      1. CI passes (lint, format, types, tests)                  │
│      2. Run migrations (alembic upgrade head)                   │
│      3. Build + push Docker image                               │
│      4. Deploy new code                                         │
│                                                                 │
│  ROLLBACK:                                                      │
│      git revert HEAD && git push     # Triggers new CI+CD       │
│      OR deploy previous image tag    # Instant rollback         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CI/CD = AUTOMATED QUALITY ENFORCEMENT                          │
│                                                                 │
│  Every code change passes through an automated assembly line.   │
│  The machine checks what humans forget. The machine never       │
│  skips a step. The machine never has a bad Friday.              │
│                                                                 │
│                                                                 │
│  Code Push ──▶ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐             │
│                │ LINT │▶│FORMAT│▶│ TYPE │▶│ TEST │             │
│                └──────┘ └──────┘ └──────┘ └──┬───┘             │
│                                              │                  │
│                                         ALL PASS?               │
│                                          │     │                │
│                                         YES    NO               │
│                                          │     │                │
│                                          ▼     ▼                │
│                                       DEPLOY  BLOCK             │
│                                       🚀      ❌                │
│                                                                 │
│                                                                 │
│  THE FACTORY ASSEMBLY LINE:                                     │
│  ├─ Workflow file = The blueprint on the factory wall            │
│  ├─ Runner = Factory worker (fresh every shift, no memory)      │
│  ├─ Quality gates = Inspection stations (lint, format, type,    │
│  │   test)                                                      │
│  ├─ CD = The shipping department (build, migrate, deploy)       │
│  ├─ needs: ci = "Don't ship until inspection passes"            │
│  └─ Rollback = Product recall (deploy previous known-good       │
│      version)                                                   │
│                                                                 │
│  The pipeline doesn't make your code better.                    │
│  It PREVENTS bad code from reaching production.                 │
│  You still write the code. The machine just holds you to        │
│  your own standards.                                            │
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
│  WEEK 15, LECTURE 4 (Next lecture):                              │
│  └─ Cloud Fundamentals                                          │
│     Your CD pipeline deploys TO somewhere. You'll learn         │
│     about cloud platforms, managed services, and where          │
│     your Docker images actually run.                            │
│                                                                 │
│  WEEK 15 DELIVERABLE:                                           │
│  └─ Dockerized application with CI/CD pipeline                  │
│     You'll build the ci.yml and cd.yml for your capstone.       │
│     GitHub Actions CI (lint + type check + test + coverage)     │
│     deployed to a platform of your choice.                      │
│                                                                 │
│  WEEK 16 (System Design):                                       │
│  └─ CI/CD is a building block of production systems.            │
│     System design interviews assume you know this.              │
│     "How would you deploy this?" has a real answer now.         │
│                                                                 │
│  CAREER:                                                        │
│  └─ Every professional team uses CI/CD. Every one.              │
│     The specific tool varies (GitHub Actions, GitLab CI,        │
│     Jenkins, CircleCI) but the CONCEPTS are identical:          │
│     triggers, jobs, steps, gates, secrets, environments.        │
│     You learned the concepts. The tools are interchangeable.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```