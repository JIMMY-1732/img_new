# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM FIRST, TOOL LAST                                       │
│  ────────────────────────                                       │
│  Students must see a deployment BREAK from hardcoded config     │
│  and stare at USELESS logs before we introduce solutions.       │
│  Pain is the teacher. We just provide the bandages.             │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Production operations are abstract. We use a hospital/ICU      │
│  analogy throughout. Every concept maps to patient care.        │
│                                                                 │
│  PROGRESSIVE COMPLEXITY                                         │
│  ──────────────────────                                         │
│  Hardcoded → os.environ → pydantic-settings.                    │
│  print() → logging → structlog → correlation IDs.               │
│  Each layer solves the previous layer's failure.                │
│                                                                 │
│  BUILD ON WHAT THEY KNOW                                        │
│  ────────────────────────                                       │
│  Pydantic BaseModel → BaseSettings is one import away.          │
│  FastAPI Depends() → Settings as a dependency.                  │
│  Docker Compose → Environment variables they just used.         │
│  Don't re-teach. Reference, connect, extend.                    │
│                                                                 │
│  SECURITY AS HABIT, NOT AFTERTHOUGHT                            │
│  ─────────────────────────────────────                          │
│  Secrets hygiene and log sanitization are not optional.          │
│  We teach them as reflexes, not features.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│              CONFIGURATION, SECRETS & OBSERVABILITY             │
│                      (3-4 Hour Lecture)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PROBLEM (35 min)                                   │
│  ├─ 1.1 The Deploy That Breaks (Demonstration)                  │
│  ├─ 1.2 The Invisible Crash (Demonstration)                     │
│  ├─ 1.3 Three Pillars of Production Readiness                   │
│  └─ 1.4 The Hospital Analogy                                    │
│                                                                 │
│  PART 2: CONFIGURATION (55 min)                                 │
│  ├─ 2.1 The 12-Factor App: Config in Environment                │
│  ├─ 2.2 Environment Variables (The Foundation)                  │
│  ├─ 2.3 pydantic-settings (Connection to Week 3)                │
│  ├─ 2.4 Environment-Specific Configuration                      │
│  └─ 2.5 Configuration in Docker (Connection to Lecture 1)       │
│                                                                 │
│  PART 3: SECRETS MANAGEMENT (25 min)                            │
│  ├─ 3.1 What Counts as a Secret?                                │
│  ├─ 3.2 The .env File Pattern                                   │
│  ├─ 3.3 The Golden Rule: Never Commit Secrets                   │
│  └─ 3.4 Secrets in Docker and Beyond                            │
│                                                                 │
│  PART 4: STRUCTURED LOGGING & OBSERVABILITY (55 min)            │
│  ├─ 4.1 print() is Not Logging                                  │
│  ├─ 4.2 structlog — Structured JSON Logging                     │
│  ├─ 4.3 Log Levels and When to Use Each                         │
│  ├─ 4.4 Correlation IDs (Tracing a Request Across the System)   │
│  ├─ 4.5 Logging Middleware for FastAPI                           │
│  └─ 4.6 What NOT to Log (Security!)                             │
│                                                                 │
│  PART 5: HEALTH CHECKS & ERROR TRACKING (40 min)               │
│  ├─ 5.1 Liveness vs Readiness (Two Different Questions)         │
│  ├─ 5.2 Implementing /health and /ready                         │
│  ├─ 5.3 Error Tracking with Sentry                              │
│  └─ 5.4 The Complete Picture                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PROBLEM

## 1.1 The Deploy That Breaks

**Start with a demonstration. Make them watch the failure.**

You just learned Docker in Lecture 1. You wrote a Dockerfile, a `docker-compose.yml`. Let's try to deploy the capstone app that a hypothetical (reckless) student wrote.

```python
# main.py — "It works on my machine!"
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import create_async_engine

app = FastAPI(title="Task Manager SaaS")

# Look at this. Just LOOK at it.
DATABASE_URL = "postgresql+asyncpg://admin:password123@localhost:5432/myapp"
SECRET_KEY = "super-secret-key-do-not-share"
REDIS_URL = "redis://localhost:6379"
DEBUG = True
CORS_ORIGINS = ["http://localhost:3000"]

engine = create_async_engine(DATABASE_URL)

@app.get("/")
async def root():
    return {"status": "running", "debug": DEBUG}
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password123
      POSTGRES_DB: myapp
  redis:
    image: redis:7-alpine
```

**Run it:**

```bash
docker compose up
```

**Watch it explode:**

```
api-1  | sqlalchemy.exc.OperationalError: 
api-1  |   cannot connect to host "localhost" port 5432:
api-1  |   Connection refused
api-1  | 
api-1  | Process exited with code 1
```

**Now ask the class:**

> "The database is running. Postgres is up and healthy. Why can't the API connect?"

Pause. Let them think. Someone remembers from Lecture 1: inside a Docker container, `localhost` means *that container itself*, not the host machine, not the Postgres container. The database service is reachable at hostname `db` (the service name), not `localhost`.

> "So... do we go into `main.py` and change `localhost` to `db`? What happens when we run the code locally without Docker during development? Now `db` doesn't resolve. Do we change it back? What about staging? Production? Every environment has a different database host."

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE ENVIRONMENT PROBLEM                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LOCAL DEVELOPMENT:                                             │
│    DATABASE_URL = "postgresql://admin:pass@localhost:5432/myapp" │
│                                                                 │
│  DOCKER COMPOSE:                                                │
│    DATABASE_URL = "postgresql://admin:pass@db:5432/myapp"       │
│                                       ▲▲                       │
│  STAGING:                              ││                       │
│    DATABASE_URL = "postgresql://app:X@staging-db.internal/myapp"│
│                                                                 │
│  PRODUCTION:                                                    │
│    DATABASE_URL = "postgresql://app:Y@prod-rds.aws.com/myapp"  │
│                                                                 │
│                                                                 │
│  HARDCODING = PICKING ONE AND BREAKING ALL THE OTHERS           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "Configuration that is baked into code is a lie. It tells the code WHERE things are, but that answer changes depending on where the code RUNS. Configuration must come from OUTSIDE the code."

---

## 1.2 The Invisible Crash

**Second demonstration. A different kind of pain.**

It's Monday morning. A user reports: "I can't create tasks. It just gives me an error." You check your application logs:

```
Starting server...
Connected to database
Connected to Redis
User logged in
Creating task
Error occurred
User logged in
User logged in
Creating task
Done
Creating task
Error occurred
User logged in
```

**Ask the class:**

> "From these logs, tell me: WHICH user couldn't create a task? WHEN did it happen? What was the ERROR? Was it the same error both times? Which organization were they in? What was in the request?"

Answer: **You have no idea.** These logs are useless. They are noise masquerading as information.

**Now show what STRUCTURED logs look like:**

```json
{"ts": "2026-02-14T09:14:01Z", "level": "info", "event": "task_create_started", "user_id": 42, "org_id": 7, "request_id": "a1b2c3"}
{"ts": "2026-02-14T09:14:01Z", "level": "error", "event": "task_create_failed", "user_id": 42, "org_id": 7, "request_id": "a1b2c3", "error": "UniqueViolationError", "detail": "duplicate key: title"}
{"ts": "2026-02-14T09:15:23Z", "level": "info", "event": "task_create_started", "user_id": 91, "org_id": 3, "request_id": "d4e5f6"}
{"ts": "2026-02-14T09:15:23Z", "level": "info", "event": "task_create_success", "user_id": 91, "org_id": 3, "request_id": "d4e5f6", "task_id": 1847}
```

> "Now I know: User 42 in organization 7 tried to create a task at 9:14 AM and failed because of a duplicate title constraint. User 91 in org 3 succeeded at 9:15 AM. I can filter by `request_id`, by `user_id`, by `org_id`, by `level`. I can feed this into a search tool and query it. THAT is the difference between print statements and structured logging."

---

## 1.3 Three Pillars of Production Readiness

```
┌─────────────────────────────────────────────────────────────────┐
│              THREE PILLARS OF PRODUCTION READINESS              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │              │  │              │  │              │          │
│  │  CONFIGURE   │  │   PROTECT    │  │   OBSERVE    │          │
│  │    IT        │  │     IT       │  │     IT       │          │
│  │              │  │              │  │              │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│         ▼                 ▼                 ▼                   │
│   Environment-       Secrets never    Structured logs,          │
│   driven config,     in code, never   health checks,            │
│   validated at       in git, never    error tracking,           │
│   startup            in logs          correlation IDs           │
│                                                                 │
│                                                                 │
│   "Works in ANY      "Breaches are    "When it breaks at        │
│    environment"       prevented,       3AM, you know WHAT       │
│                       not reacted to"  broke and WHY"           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Today we make your capstone project production-worthy. Not feature-complete — you already did that. Production-WORTHY. The kind of application a team can deploy, monitor, debug, and trust at 3AM without you being awake."

---

## 1.4 The Hospital Analogy

**This analogy carries us through the entire lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE HOSPITAL ANALOGY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your application in production is a PATIENT IN THE ICU.        │
│  It's alive, it's running, but it needs MONITORING and CARE.    │
│                                                                 │
│                                                                 │
│  TREATMENT PLAN = CONFIGURATION                                 │
│  ──────────────────────────────                                 │
│  Every patient gets a different dosage, different medication,   │
│  different schedule. You don't hardcode "give everyone 500mg    │
│  of ibuprofen." The treatment plan comes from OUTSIDE —         │
│  the doctor writes it, the nurse reads it, the patient gets it. │
│                                                                 │
│  CONTROLLED SUBSTANCES CABINET = SECRETS                        │
│  ────────────────────────────────────────                       │
│  Morphine isn't left on the counter. It's locked, logged,       │
│  audited. Only authorized staff access it. If it goes missing,  │
│  everyone knows immediately. API keys and passwords deserve     │
│  the same discipline.                                           │
│                                                                 │
│  MEDICAL CHART = STRUCTURED LOGS                                │
│  ────────────────────────────────                               │
│  Every action is recorded: who, what, when, why. Structured,    │
│  timestamped, tied to the patient. Not scribbled on a napkin.   │
│                                                                 │
│  PATIENT WRISTBAND = CORRELATION ID                             │
│  ──────────────────────────────────                             │
│  One ID follows the patient through every department — ER,      │
│  radiology, surgery, pharmacy. Every record links back to       │
│  that one wristband. One request ID follows a request through   │
│  every layer of your application.                               │
│                                                                 │
│  VITAL SIGNS MONITOR = HEALTH CHECKS                            │
│  ────────────────────────────────────                           │
│  Heartbeat, blood pressure, oxygen — continuously checked.      │
│  "Is the patient alive?" and "Is the patient stable enough      │
│  for surgery?" are two different questions.                     │
│                                                                 │
│  CODE BLUE ALARM = ERROR TRACKING (SENTRY)                      │
│  ─────────────────────────────────────────                      │
│  When something goes critically wrong, the RIGHT team gets      │
│  paged IMMEDIATELY. Not an email. Not a log line nobody reads.  │
│  An alarm that wakes people up.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Map the analogy to code:**

```
Hospital                    │  Production Application
────────────────────────────│──────────────────────────────
Patient in ICU              │  Your app deployed to cloud
Treatment plan              │  Configuration (Settings class)
  (dosage per patient)      │    (different per environment)
Controlled substances       │  Secrets (API keys, DB passwords)
  cabinet                   │    (never in code, never in git)
Medical chart               │  Structured logs (structlog)
  (structured, timestamped) │    (JSON, timestamped, contextual)
Patient wristband ID        │  Correlation ID (X-Correlation-ID)
  (tracks across depts)     │    (tracks across services/layers)
"Is patient alive?"         │  GET /health  (liveness)
"Ready for surgery?"        │  GET /ready   (readiness)
Code Blue alarm             │  Sentry alert (error tracking)
```

---

# PART 2: CONFIGURATION

## 2.1 The 12-Factor App: Config in Environment

**The 12-Factor App is a methodology for building modern, deployable applications. We care about one factor right now: Factor III — Config.**

```
┌─────────────────────────────────────────────────────────────────┐
│              12-FACTOR APP: FACTOR III — CONFIG                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  THE RULE:                                                      │
│  ─────────                                                      │
│  "Store config in the environment."                             │
│                                                                 │
│  Config is everything that VARIES between environments:         │
│  ├─ Database URLs                                               │
│  ├─ Redis URLs                                                  │
│  ├─ API keys for external services                              │
│  ├─ Secret keys for JWT                                         │
│  ├─ Debug mode on/off                                           │
│  ├─ CORS allowed origins                                        │
│  ├─ Log level                                                   │
│  └─ Port numbers                                                │
│                                                                 │
│  Config is NOT:                                                  │
│  ├─ Your route definitions                                      │
│  ├─ Your Pydantic model schemas                                 │
│  ├─ Your business logic                                         │
│  └─ Anything that stays the same across all environments        │
│                                                                 │
│                                                                 │
│  THE TEST:                                                      │
│  ─────────                                                      │
│  "Could you open-source this codebase RIGHT NOW without         │
│   exposing any credentials or environment-specific details?"    │
│                                                                 │
│   If NO → you have config in code. Fix it.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why the environment?**

```
┌─────────────────────────────────────────────────────────────────┐
│                WHY ENVIRONMENT VARIABLES?                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Language-agnostic  — Every OS, every language supports them │
│  ✅ Not in code        — Can't accidentally commit them         │
│  ✅ Easy to change     — No code deploy needed to rotate a key  │
│  ✅ Per-environment    — Dev, staging, prod each set their own  │
│  ✅ Docker-native      — docker-compose.yml has environment:    │
│  ✅ Cloud-native       — Every cloud platform supports them     │
│                                                                 │
│                                                                 │
│       ┌─────────┐      ┌─────────┐      ┌─────────┐           │
│       │   DEV   │      │ STAGING │      │  PROD   │           │
│       │         │      │         │      │         │           │
│       │ DB=local│      │ DB=stg  │      │ DB=rds  │           │
│       │ DEBUG=1 │      │ DEBUG=0 │      │ DEBUG=0 │           │
│       │ LOG=dbg │      │ LOG=info│      │ LOG=warn│           │
│       └────┬────┘      └────┬────┘      └────┬────┘           │
│            │                │                │                 │
│            └────────────────┼────────────────┘                 │
│                             │                                  │
│                      ┌──────▼──────┐                           │
│                      │  SAME CODE  │                           │
│                      │  (main.py)  │                           │
│                      │             │                           │
│                      │  No config  │                           │
│                      │  hardcoded  │                           │
│                      └─────────────┘                           │
│                                                                 │
│  The code is IDENTICAL across all environments.                 │
│  Only the environment variables change.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Think of it like the hospital treatment plan. The nurse (your code) follows the SAME protocol regardless of the patient. But the dosage (config) is written on the chart (environment), not tattooed on the nurse's arm."

---

## 2.2 Environment Variables (The Foundation)

**Environment variables are key-value pairs set OUTSIDE your code:**

```bash
# Setting environment variables (terminal)
export DATABASE_URL="postgresql+asyncpg://admin:pass@localhost:5432/myapp"
export SECRET_KEY="a-real-secret-key"
export DEBUG="false"
```

**Reading them in Python:**

```python
import os

# Method 1: os.environ — raises KeyError if missing
database_url = os.environ["DATABASE_URL"]

# Method 2: os.environ.get() — returns default if missing
debug = os.environ.get("DEBUG", "false")
```

**But this approach has serious problems:**

```python
# Problem 1: No type safety — everything is a string
debug = os.environ.get("DEBUG", "false")
# debug is "false" (a STRING), not False (a BOOL)

if debug:  # ← THIS IS ALWAYS TRUE! Non-empty string is truthy!
    print("Debug mode on")  # Oops. "false" is truthy.

# Problem 2: Manual type conversion is error-prone
port = int(os.environ.get("PORT", "8000"))
max_conn = int(os.environ.get("MAX_CONNECTIONS", "ten"))  # 💥 ValueError!

# Problem 3: No validation — you discover missing config at RUNTIME
# App starts, runs for hours, then crashes when it first needs
# a variable that was never set.

# Problem 4: Config scattered across files
# main.py reads DATABASE_URL
# auth.py reads SECRET_KEY
# cache.py reads REDIS_URL
# No single place to see ALL configuration
```

```
┌─────────────────────────────────────────────────────────────────┐
│                os.environ PROBLEMS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ Everything is a string (no types)                           │
│  ❌ "false" is truthy in Python                                 │
│  ❌ Manual type conversion (error-prone)                        │
│  ❌ Missing variables found at runtime, not startup             │
│  ❌ No validation (is this URL valid? is this port in range?)   │
│  ❌ Config scattered across multiple files                      │
│  ❌ No documentation of what config exists                      │
│  ❌ No defaults management                                     │
│                                                                 │
│  We need something that:                                        │
│  ✅ Reads from environment                                      │
│  ✅ Validates types automatically                               │
│  ✅ Fails FAST at startup if config is wrong                    │
│  ✅ Lives in ONE place                                          │
│  ✅ Self-documents all configuration                            │
│                                                                 │
│  Sound familiar? It should.                                     │
│  This is exactly what Pydantic does for REQUEST data.           │
│  Now we do it for CONFIGURATION.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.3 pydantic-settings (Connection to Week 3)

**Connection to what you've learned:**

> "Remember Pydantic from Week 3? `BaseModel` validates incoming request data — it parses, coerces types, and rejects bad input. `BaseSettings` does the EXACT same thing, but for environment variables. Same validators. Same Field constraints. Same mental model. Different data source."

**Install it:**

```bash
pip install pydantic-settings
```

**The transformation — from chaos to clarity:**

```python
# config.py — ONE file, ALL configuration, FULLY validated.
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field


class Settings(BaseSettings):
    """Application configuration.
    
    All values are read from environment variables.
    Variable names match field names (case-insensitive).
    """
    
    model_config = SettingsConfigDict(
        env_file=".env",              # Load from .env file (if it exists)
        env_file_encoding="utf-8",
        extra="ignore",               # Ignore extra env vars (don't crash)
    )
    
    # ── Database ──────────────────────────────────────────────
    database_url: str
    # Env var: DATABASE_URL (required — no default, must be set)
    
    # ── Redis ─────────────────────────────────────────────────
    redis_url: str = "redis://localhost:6379/0"
    # Has a default — optional to set
    
    # ── Security ──────────────────────────────────────────────
    secret_key: str
    # Required — JWT signing key (no sane default for this)
    
    # ── Application ───────────────────────────────────────────
    debug: bool = False
    # Pydantic coerces "true"/"false"/"1"/"0" → bool automatically!
    
    app_name: str = "Task Manager SaaS"
    api_v1_prefix: str = "/api/v1"
    
    # ── Server ────────────────────────────────────────────────
    host: str = "0.0.0.0"
    port: int = Field(default=8000, ge=1, le=65535)
    # Field constraints — same as Week 3! Port must be valid range.
    
    # ── Connections ───────────────────────────────────────────
    db_pool_size: int = Field(default=10, ge=1, le=100)
    db_max_overflow: int = Field(default=20, ge=0)
    
    # ── CORS ──────────────────────────────────────────────────
    cors_origins: list[str] = ["http://localhost:3000"]
    # Pydantic can parse JSON strings from env vars!
    # Set as: CORS_ORIGINS='["https://app.example.com"]'
    
    # ── Observability (we'll fill these in later this lecture) ─
    log_level: str = "INFO"
    sentry_dsn: str | None = None
    # None = Sentry disabled (good for local dev)
    
    environment: str = "development"
    # "development", "staging", "production"
```

**What just happened? Let's unpack it:**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHAT pydantic-settings DOES FOR YOU                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. READS from environment variables automatically              │
│     Field name "database_url" → reads env var DATABASE_URL      │
│     (case-insensitive matching)                                 │
│                                                                 │
│  2. COERCES types                                               │
│     debug: bool → "true", "1", "yes", "on" all become True     │
│     port: int → "8000" becomes 8000                             │
│     cors_origins: list[str] → parses JSON string to list        │
│                                                                 │
│  3. VALIDATES constraints                                       │
│     Field(ge=1, le=65535) → rejects port=99999                  │
│     str (no default) → rejects missing values                   │
│                                                                 │
│  4. FAILS FAST at startup                                       │
│     Missing DATABASE_URL? App crashes IMMEDIATELY with a        │
│     clear error — not 3 hours later on first DB query.          │
│                                                                 │
│  5. LOADS from .env file (for development convenience)          │
│     env_file=".env" → reads .env if present, env vars override  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**See the startup validation in action:**

```python
# If DATABASE_URL is not set:
settings = Settings()

# pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings
# database_url
#   Field required [type=missing, input_value={...}]

# If PORT is out of range:
# PORT=99999 python main.py

# pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings
# port
#   Input should be less than or equal to 65535 [type=less_than_equal, ...]
```

> "Your app never starts with bad config. It fails LOUDLY at the door, not silently in the middle of serving users. Like a hospital — you check the treatment plan BEFORE administering medication, not after the patient has a reaction."

**Using Settings in FastAPI (Connection to Week 3 — Depends):**

```python
# dependencies.py
from functools import lru_cache
from config import Settings


@lru_cache  # ← Create Settings only ONCE, reuse everywhere
def get_settings() -> Settings:
    return Settings()
```

```python
# main.py
from fastapi import FastAPI, Depends
from config import Settings
from dependencies import get_settings

settings = get_settings()

app = FastAPI(
    title=settings.app_name,
    debug=settings.debug,
)

# In route handlers — inject settings as a dependency
@app.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return {
        "app": settings.app_name,
        "environment": settings.environment,
        "debug": settings.debug,
    }
```

**Why `@lru_cache`?**

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHY @lru_cache?                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT @lru_cache:                                            │
│                                                                 │
│  get_settings()  →  reads .env, parses, validates  →  Settings  │
│  get_settings()  →  reads .env, parses, validates  →  Settings  │
│  get_settings()  →  reads .env, parses, validates  →  Settings  │
│  (every call does the work again)                               │
│                                                                 │
│                                                                 │
│  WITH @lru_cache:                                               │
│                                                                 │
│  get_settings()  →  reads .env, parses, validates  →  Settings  │
│  get_settings()  →  returns SAME object (cached)   →  Settings  │
│  get_settings()  →  returns SAME object (cached)   →  Settings  │
│  (work done once, result reused)                                │
│                                                                 │
│                                                                 │
│  Config doesn't change during runtime.                          │
│  Parse it once. Reuse forever.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.4 Environment-Specific Configuration

**The same Settings class works across all environments. What changes is the VALUES:**

```
┌─────────────────────────────────────────────────────────────────┐
│              ENVIRONMENT-SPECIFIC VALUES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DEVELOPMENT (.env file on your laptop):                        │
│  ─────────────────────────────────────────                      │
│  DATABASE_URL=postgresql+asyncpg://admin:pass@localhost/myapp   │
│  REDIS_URL=redis://localhost:6379/0                             │
│  SECRET_KEY=dev-only-not-real-secret                            │
│  DEBUG=true                                                     │
│  LOG_LEVEL=DEBUG                                                │
│  ENVIRONMENT=development                                       │
│                                                                 │
│                                                                 │
│  DOCKER COMPOSE (docker-compose.yml environment:):              │
│  ─────────────────────────────────────────────────              │
│  DATABASE_URL=postgresql+asyncpg://admin:pass@db:5432/myapp     │
│  REDIS_URL=redis://redis:6379/0                                 │
│  SECRET_KEY=docker-dev-secret                                   │
│  DEBUG=true                                                     │
│  LOG_LEVEL=DEBUG                                                │
│  ENVIRONMENT=development                                       │
│                                                                 │
│                                                                 │
│  PRODUCTION (set in cloud platform's config/secrets manager):   │
│  ────────────────────────────────────────────────────────       │
│  DATABASE_URL=postgresql+asyncpg://app:$ROTATED@prod-rds/myapp │
│  REDIS_URL=redis://prod-redis.cache.aws:6379/0                 │
│  SECRET_KEY=kj2h3f98a7sd6f5...  (64-char random)               │
│  DEBUG=false                                                    │
│  LOG_LEVEL=WARNING                                              │
│  ENVIRONMENT=production                                        │
│  SENTRY_DSN=https://abc123@sentry.io/456                       │
│                                                                 │
│                                                                 │
│  THE CODE DOES NOT CHANGE. Only the environment changes.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Conditional behavior based on environment:**

```python
# config.py — add computed properties
from pydantic import computed_field

class Settings(BaseSettings):
    # ... fields from before ...
    environment: str = "development"

    @computed_field
    @property
    def is_production(self) -> bool:
        return self.environment == "production"
    
    @computed_field
    @property
    def is_development(self) -> bool:
        return self.environment == "development"
```

```python
# main.py — use environment to control behavior
settings = get_settings()

app = FastAPI(
    title=settings.app_name,
    debug=settings.debug,
    docs_url="/docs" if not settings.is_production else None,
    # ↑ Disable Swagger UI in production!
    redoc_url="/redoc" if not settings.is_production else None,
)
```

> "Same code. The production deployment disables Swagger docs automatically because `ENVIRONMENT=production`. No if-else in a config file. No separate branches. The environment tells the code how to behave."

---

## 2.5 Configuration in Docker (Connection to Lecture 1)

**You learned Docker Compose yesterday. Here's how config flows into containers:**

```yaml
# docker-compose.yml — development configuration
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      # Method 1: Set values directly
      DATABASE_URL: "postgresql+asyncpg://admin:devpass@db:5432/myapp"
      REDIS_URL: "redis://redis:6379/0"
      SECRET_KEY: "docker-dev-only-secret"
      DEBUG: "true"
      LOG_LEVEL: "DEBUG"
      ENVIRONMENT: "development"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: myapp
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d myapp"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
```

**Or use a separate `.env` file for Docker Compose:**

```yaml
# docker-compose.yml — cleaner version
services:
  api:
    build: .
    ports:
      - "8000:8000"
    env_file:
      - .env.docker    # ← Load from file
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
```

```bash
# .env.docker — Docker-specific configuration
DATABASE_URL=postgresql+asyncpg://admin:devpass@db:5432/myapp
REDIS_URL=redis://redis:6379/0
SECRET_KEY=docker-dev-only-secret
DEBUG=true
LOG_LEVEL=DEBUG
ENVIRONMENT=development
```

**The priority order — when the same variable is set in multiple places:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CONFIGURATION PRIORITY (highest → lowest)          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HIGHEST PRIORITY                                               │
│  ─────────────────                                              │
│  1. Actual environment variables (set in shell, CI, cloud)      │
│     └─ export DATABASE_URL=...                                  │
│                                                                 │
│  2. .env file loaded by pydantic-settings                       │
│     └─ Only if env var is NOT already set                       │
│                                                                 │
│  3. Default values in Settings class                            │
│     └─ debug: bool = False                                      │
│  ─────────────────                                              │
│  LOWEST PRIORITY                                                │
│                                                                 │
│                                                                 │
│  This means:                                                    │
│  • Real env vars ALWAYS override .env file values               │
│  • .env file overrides class defaults                           │
│  • You can override any config without touching code            │
│                                                                 │
│  In production, you set real env vars (via cloud platform).     │
│  The .env file doesn't even exist. Defaults fill the rest.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Go back to the demo from 1.1 — the fix is now obvious:**

```python
# main.py — FIXED
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import create_async_engine
from dependencies import get_settings

settings = get_settings()

app = FastAPI(title=settings.app_name, debug=settings.debug)
engine = create_async_engine(
    settings.database_url,
    pool_size=settings.db_pool_size,
    max_overflow=settings.db_max_overflow,
)

# No hardcoded URLs. No hardcoded secrets. No hardcoded flags.
# The environment tells the code what to do.
```

> "Locally, `DATABASE_URL` points to `localhost`. In Docker, it points to `db`. In production, it points to `prod-rds.aws.com`. The code doesn't know and doesn't care. Just like the nurse follows the same protocol but reads a different treatment plan for each patient."

---

# PART 3: SECRETS MANAGEMENT

## 3.1 What Counts as a Secret?

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHAT IS A SECRET?                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  A secret is any value that, if leaked, could:                  │
│  • Give unauthorized access to your systems                     │
│  • Compromise user data                                        │
│  • Cost you money                                               │
│  • Impersonate your application                                 │
│                                                                 │
│                                                                 │
│  🔒 SECRETS (must be protected):                                │
│  ├─ Database passwords               (DB access)               │
│  ├─ JWT secret keys                   (forge any token)         │
│  ├─ API keys for external services    (billing, impersonation)  │
│  ├─ OAuth client secrets              (impersonate your app)    │
│  ├─ Sentry DSN                        (send fake errors)        │
│  ├─ SMTP/email credentials            (send email as you)       │
│  ├─ Cloud provider credentials        (spin up resources)       │
│  ├─ Encryption keys                   (decrypt user data)       │
│  └─ Webhook signing secrets           (forge webhook events)    │
│                                                                 │
│                                                                 │
│  🔓 NOT SECRETS (safe to commit):                               │
│  ├─ App name                                                    │
│  ├─ Port number                                                 │
│  ├─ Log level                                                   │
│  ├─ API version prefix                                          │
│  ├─ Pagination defaults                                         │
│  └─ Feature flags (usually)                                     │
│                                                                 │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  "If someone malicious found this value in a GitHub search,     │
│   could they do damage?"                                        │
│  If YES → it's a secret.                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 The .env File Pattern

**For local development, we use a `.env` file for convenience:**

```bash
# .env — LOCAL DEVELOPMENT ONLY
# This file is NEVER committed to git.

# Database
DATABASE_URL=postgresql+asyncpg://admin:localpass@localhost:5432/myapp

# Security
SECRET_KEY=local-dev-key-not-real
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# Redis
REDIS_URL=redis://localhost:6379/0

# External APIs
GITHUB_API_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
SENDGRID_API_KEY=SG.xxxxxxxxxxxxxxxxxxxx

# Observability
LOG_LEVEL=DEBUG
SENTRY_DSN=
ENVIRONMENT=development
```

**pydantic-settings reads this automatically** (because we set `env_file=".env"` in `SettingsConfigDict`). No extra code needed.

But the `.env` file is ONLY for development convenience. In production, secrets are set through your cloud platform's secrets manager, not files on disk.

---

## 3.3 The Golden Rule: Never Commit Secrets

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   🚨  THE GOLDEN RULE                                          │
│   ════════════════════                                          │
│                                                                 │
│   NEVER COMMIT SECRETS TO GIT. EVER.                            │
│                                                                 │
│   Not even once. Not even "temporarily."                        │
│   Not even in a private repo.                                   │
│   Not even if you "plan to remove it later."                    │
│                                                                 │
│   Git remembers EVERYTHING. A secret committed and then         │
│   deleted is still in the git history. Forever.                 │
│                                                                 │
│   Bots scan GitHub continuously for leaked secrets.             │
│   AWS keys committed to public repos are exploited within       │
│   MINUTES. Not hours. Minutes.                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Step 1: `.gitignore` — your first line of defense:**

```bash
# .gitignore — MUST include these
.env
.env.*
!.env.example
# ↑ The ! means "DO track .env.example" (it's safe, it's a template)
```

**Step 2: Provide a `.env.example` file (THIS gets committed):**

```bash
# .env.example — TEMPLATE for developers.
# Copy this to .env and fill in real values.
# This file IS committed to git. It contains NO real secrets.

DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/dbname
SECRET_KEY=generate-a-real-secret-key-here
REDIS_URL=redis://localhost:6379/0
GITHUB_API_TOKEN=your-github-token-here
SENDGRID_API_KEY=your-sendgrid-key-here
LOG_LEVEL=DEBUG
SENTRY_DSN=
ENVIRONMENT=development
```

> "`.env.example` is the treatment plan TEMPLATE. It says 'this patient needs a painkiller at dose X' but doesn't have the actual controlled substance. `.env` is the filled-in prescription. The template is public. The prescription is locked up."

**Step 3: Verify you haven't committed secrets:**

```bash
# Check if .env is tracked by git
git ls-files | grep .env

# If it shows .env — you've already committed it!
# Remove it from tracking (but keep the file):
git rm --cached .env
# Then commit the removal.
# But the secret is STILL IN HISTORY. Rotate it immediately.
```

```
┌─────────────────────────────────────────────────────────────────┐
│              IF YOU ACCIDENTALLY COMMIT A SECRET                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Don't panic. But act FAST.                                  │
│                                                                 │
│  2. ROTATE THE SECRET IMMEDIATELY.                              │
│     ├─ Generate a new database password                         │
│     ├─ Generate a new API key                                   │
│     ├─ Generate a new JWT secret                                │
│     └─ Update all environments with new values                  │
│                                                                 │
│  3. Removing the commit is NOT enough.                          │
│     Git history retains it. Anyone who cloned has it.           │
│     The old secret is COMPROMISED. Rotation is the only fix.    │
│                                                                 │
│  4. Learn from it. Set up pre-commit hooks to prevent it:       │
│     └─ Tools like detect-secrets or gitleaks scan commits       │
│        for patterns that look like secrets                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Secrets in Docker and Beyond

**In Docker Compose (development), set secrets as environment variables:**

```yaml
# docker-compose.yml — dev environment
services:
  api:
    build: .
    env_file:
      - .env.docker    # Contains secrets for local dev
```

**In production, secrets come from the platform's secrets manager:**

```
┌─────────────────────────────────────────────────────────────────┐
│              SECRETS IN PRODUCTION                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PLATFORM            │  HOW SECRETS ARE SET                     │
│  ─────────────────── │ ─────────────────────────                │
│  Railway             │  Dashboard → Variables tab               │
│  Fly.io              │  fly secrets set KEY=VALUE               │
│  AWS ECS             │  AWS Secrets Manager / SSM               │
│  Kubernetes          │  kubectl create secret                   │
│  GitHub Actions (CI) │  Repository → Settings → Secrets         │
│                                                                 │
│  In ALL cases: secrets are injected as environment variables    │
│  into the running container. Your code (pydantic-settings)      │
│  reads them the exact same way. No code changes needed.         │
│                                                                 │
│                                                                 │
│  .env file?          ← Does NOT exist in production.            │
│  pydantic-settings?  ← Reads real env vars instead. Same code.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "In the hospital analogy: during training, a student nurse might read patient records from a printed sheet (`.env` file). In the real hospital, they access the electronic health record system (secrets manager). Same workflow, same protocol — different source. And the real system has audit logs and access control that the printed sheet does not."

---

# PART 4: STRUCTURED LOGGING & OBSERVABILITY

## 4.1 print() is Not Logging

**Let's be clear about what `print()` does and does not give you:**

```python
# What most beginners write:
print("Starting server...")
print(f"User {user_id} logged in")
print("Error occurred")  # ← Helpful as a chocolate teapot
print(f"Task created: {task}")
```

Output:
```
Starting server...
User 42 logged in
Error occurred
Task created: {'id': 1, 'title': 'Fix bug'}
```

**What's wrong with this?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  WHY print() FAILS IN PRODUCTION                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ NO TIMESTAMPS                                               │
│     "Error occurred" — When? This morning? Last week?           │
│                                                                 │
│  ❌ NO SEVERITY LEVELS                                          │
│     Is "Error occurred" a WARNING or a CRITICAL failure?        │
│     Can I filter to see ONLY errors?                            │
│                                                                 │
│  ❌ NO STRUCTURE                                                │
│     "User 42 logged in" — Try parsing that with a tool.         │
│     Is 42 the user ID or the age? A regex? Fragile.             │
│                                                                 │
│  ❌ NO CONTEXT                                                  │
│     Which request caused this? Which server instance?           │
│     Which organization in our multi-tenant system?              │
│                                                                 │
│  ❌ NO SEARCHABILITY                                            │
│     "Show me all errors from user 42 in org 7 last Tuesday"    │
│     Good luck grep-ing that from print() output.               │
│                                                                 │
│  ❌ NO MACHINE READABILITY                                      │
│     Log aggregation tools (Datadog, CloudWatch, Loki) need     │
│     structured data (JSON), not human prose.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What we NEED our logs to be:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHAT GOOD LOGS LOOK LIKE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MACHINE-READABLE (JSON):                                       │
│  {                                                              │
│    "timestamp": "2026-02-14T14:23:01.456Z",                    │
│    "level": "error",                                            │
│    "event": "task_create_failed",                               │
│    "user_id": 42,                                               │
│    "org_id": 7,                                                 │
│    "request_id": "a1b2c3d4",                                   │
│    "error_type": "UniqueViolationError",                        │
│    "detail": "duplicate key: title"                             │
│  }                                                              │
│                                                                 │
│  ✅ Timestamp (when?)                                           │
│  ✅ Level (how bad?)                                            │
│  ✅ Event name (what happened?)                                 │
│  ✅ Structured context (who? which org? which request?)         │
│  ✅ Machine-parseable (JSON → tools can index, search, alert)  │
│  ✅ Human-readable too (you can still read it)                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Medical charts aren't prose essays. They're structured records. Time, vital signs, medication administered, dosage, nurse's signature. Every field has a purpose. Every field is searchable. Your application logs should be medical charts, not diary entries."

---

## 4.2 structlog — Structured JSON Logging

**structlog turns logging into structured data emission.**

```bash
pip install structlog
```

**Basic usage — immediately better than print():**

```python
import structlog

# Configure structlog (do this ONCE at app startup)
structlog.configure(
    processors=[
        structlog.processors.add_log_level,         # Add "level" field
        structlog.processors.TimeStamper(fmt="iso"), # Add ISO timestamp
        structlog.dev.ConsoleRenderer(),             # Pretty output for dev
    ],
    logger_factory=structlog.PrintLoggerFactory(),
    cache_logger_on_first_use=True,
)

# Get a logger
logger = structlog.get_logger()

# Use it
logger.info("server_started", host="0.0.0.0", port=8000)
logger.info("user_logged_in", user_id=42, org_id=7)
logger.error("task_create_failed", user_id=42, error="duplicate key")
```

**Development output (pretty, human-readable):**

```
2026-02-14T14:23:00Z [info     ] server_started       host=0.0.0.0 port=8000
2026-02-14T14:23:01Z [info     ] user_logged_in       org_id=7 user_id=42
2026-02-14T14:23:01Z [error    ] task_create_failed   error=duplicate key user_id=42
```

**Production output — swap ONE processor, get JSON:**

```python
# For production: replace ConsoleRenderer with JSONRenderer
structlog.configure(
    processors=[
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),          # ← JSON for machines
    ],
    logger_factory=structlog.PrintLoggerFactory(),
    cache_logger_on_first_use=True,
)
```

```json
{"level": "info", "timestamp": "2026-02-14T14:23:00Z", "event": "server_started", "host": "0.0.0.0", "port": 8000}
{"level": "info", "timestamp": "2026-02-14T14:23:01Z", "event": "user_logged_in", "user_id": 42, "org_id": 7}
{"level": "error", "timestamp": "2026-02-14T14:23:01Z", "event": "task_create_failed", "user_id": 42, "error": "duplicate key"}
```

> "Same code. Same logger calls. Different output format. In development, you want pretty colors and aligned text. In production, you want JSON that log aggregation tools can ingest. One config change."

**The anatomy of a structlog call:**

```python
logger.info("event_name", key1=value1, key2=value2)
#      ▲        ▲              ▲
#      │        │              │
#   log level   │          context (key-value pairs)
#            event name    (structured data — not a message string!)
#         (what happened)

# ❌ BAD: Message strings like print()
logger.info(f"User {user_id} created task {task_id}")
# This creates an unstructured string. Hard to parse.

# ✅ GOOD: Event name + structured context
logger.info("task_created", user_id=user_id, task_id=task_id)
# event, user_id, task_id are all separate, searchable fields.
```

```
┌─────────────────────────────────────────────────────────────────┐
│               structlog CALL ANATOMY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  logger.info("task_created", user_id=42, task_id=7, org_id=3)  │
│         │          │              │          │          │       │
│         │          │              └──────────┴──────────┘       │
│         │          │              context: key=value pairs      │
│         │          │              (become JSON fields)          │
│         │          │                                            │
│         │          └── event: what happened                     │
│         │              (snake_case, past tense)                 │
│         │                                                       │
│         └── level: severity                                     │
│             debug/info/warning/error/critical                   │
│                                                                 │
│                                                                 │
│  OUTPUT (JSON):                                                 │
│  {                                                              │
│    "level": "info",                                             │
│    "timestamp": "2026-02-14T14:23:01Z",                        │
│    "event": "task_created",    ← from the first argument       │
│    "user_id": 42,              ← from kwargs                   │
│    "task_id": 7,               ← from kwargs                   │
│    "org_id": 3                 ← from kwargs                   │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Bound loggers — attach context once, use everywhere:**

```python
# Without binding: repetitive, error-prone
logger.info("task_listed", user_id=42, org_id=7)
logger.info("task_created", user_id=42, org_id=7, task_id=1)
logger.info("task_assigned", user_id=42, org_id=7, task_id=1, assignee=99)
# ↑ user_id and org_id repeated every single time

# With binding: attach context once
log = logger.bind(user_id=42, org_id=7)

log.info("task_listed")
# {"event": "task_listed", "user_id": 42, "org_id": 7, ...}

log.info("task_created", task_id=1)
# {"event": "task_created", "user_id": 42, "org_id": 7, "task_id": 1, ...}

log.info("task_assigned", task_id=1, assignee=99)
# {"event": "task_assigned", "user_id": 42, "org_id": 7, "task_id": 1, "assignee": 99, ...}

# user_id and org_id are automatically included in EVERY log from this logger
```

> "A bound logger is like a pre-labelled medical chart. Once you write the patient's name and wristband ID at the top, every subsequent entry inherits that context. You don't write 'Patient John Smith' on every single line."

---

## 4.3 Log Levels and When to Use Each

```
┌─────────────────────────────────────────────────────────────────┐
│                      LOG LEVELS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LEVEL     │ WHEN TO USE                    │ EXAMPLE           │
│  ──────────│────────────────────────────────│───────────────── │
│            │                                │                   │
│  DEBUG     │ Detailed diagnostic info.      │ "SQL query         │
│            │ Only in development.           │  executed in 3ms"  │
│            │ TOO noisy for production.      │                   │
│            │                                │                   │
│  INFO      │ Normal operations.             │ "User logged in"  │
│            │ "Things are working as         │ "Task created"    │
│            │  expected."                    │ "Server started"  │
│            │                                │                   │
│  WARNING   │ Something unexpected, but      │ "API rate limit   │
│            │ the system handled it.         │  at 80% capacity" │
│            │ "This might become a problem." │ "Retry attempt 2" │
│            │                                │                   │
│  ERROR     │ Something failed, but the      │ "Task creation    │
│            │ system is still running.       │  failed: DB error"│
│            │ A specific request/operation   │ "Payment API      │
│            │ could not be completed.        │  returned 500"    │
│            │                                │                   │
│  CRITICAL  │ The system itself is in        │ "Database          │
│            │ danger. Wake someone up.       │  connection lost" │
│            │ Can't serve ANY requests.      │ "Out of disk"     │
│            │                                │                   │
└─────────────────────────────────────────────────────────────────┘
```

**The hospital mapping:**

```
┌─────────────────────────────────────────────────────────────────┐
│              LOG LEVELS AS HOSPITAL EVENTS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DEBUG    = Nurse notes: "Checked vitals at 3:00 PM. Normal."  │
│             (Routine. Only checked during detailed review.)     │
│                                                                 │
│  INFO     = Chart entry: "Administered 500mg ibuprofen."       │
│             (Normal operation. Expected. Logged for records.)   │
│                                                                 │
│  WARNING  = Flag: "Blood pressure slightly elevated."           │
│             (Not a crisis, but worth monitoring closely.)       │
│                                                                 │
│  ERROR    = Incident: "Patient had allergic reaction to med."   │
│             (Bad event, patient being treated. System intact.)  │
│                                                                 │
│  CRITICAL = Code Blue: "Cardiac arrest."                        │
│             (Everything stops. All hands on deck. NOW.)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Filtering by environment (connection to Settings):**

```python
import logging
import structlog
from config import Settings

def configure_logging(settings: Settings) -> None:
    """Configure structlog based on application settings."""
    
    # Choose renderer based on environment
    if settings.is_development:
        renderer = structlog.dev.ConsoleRenderer()  # Pretty for humans
    else:
        renderer = structlog.processors.JSONRenderer()  # JSON for machines
    
    # Convert LOG_LEVEL string to logging constant
    log_level = getattr(logging, settings.log_level.upper(), logging.INFO)
    
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,     # For correlation IDs
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),    # Stack traces
            structlog.processors.format_exc_info,        # Exception formatting
            renderer,
        ],
        wrapper_class=structlog.make_filtering_bound_logger(log_level),
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )
```

```python
# main.py — call at startup
from logging_config import configure_logging
from dependencies import get_settings

settings = get_settings()
configure_logging(settings)

# Now:
# LOG_LEVEL=DEBUG   → you see everything (development)
# LOG_LEVEL=INFO    → normal operations (staging)
# LOG_LEVEL=WARNING → only problems (production)
```

> "In development, you want to see every SQL query, every cache hit/miss, every detail. In production, that volume would drown you. The `LOG_LEVEL` setting acts as a filter — same logs in the code, but only the important ones make it through in production."

---

## 4.4 Correlation IDs (Tracing a Request Across the System)

**The problem: a single user action touches many layers.**

```
┌─────────────────────────────────────────────────────────────────┐
│            A SINGLE REQUEST TOUCHES MANY LAYERS                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User clicks "Create Task"                                      │
│       │                                                         │
│       ▼                                                         │
│  FastAPI Route Handler                                          │
│       │ └─ log: "task_create_started"                           │
│       ▼                                                         │
│  Auth Dependency (verify JWT)                                   │
│       │ └─ log: "token_validated"                               │
│       ▼                                                         │
│  Service Layer (business logic)                                 │
│       │ └─ log: "checking_permissions"                          │
│       ▼                                                         │
│  Repository Layer (database)                                    │
│       │ └─ log: "db_insert_started"                             │
│       ▼                                                         │
│  Cache Invalidation (Redis)                                     │
│       │ └─ log: "cache_invalidated"                             │
│       ▼                                                         │
│  Background Task (notification)                                 │
│         └─ log: "notification_queued"                           │
│                                                                 │
│                                                                 │
│  6 log entries. From 6 different layers. In production, these   │
│  are interleaved with logs from HUNDREDS of concurrent          │
│  requests. How do you find all 6 entries for ONE request?       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The solution: a Correlation ID — one unique ID per request.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CORRELATION ID                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every request gets a UNIQUE ID at the front door.              │
│  That ID is attached to EVERY log entry in that request.        │
│                                                                 │
│  Request arrives → ID: "req-a1b2c3d4"                           │
│                                                                 │
│  {"event": "task_create_started",  "request_id": "req-a1b2c3"} │
│  {"event": "token_validated",      "request_id": "req-a1b2c3"} │
│  {"event": "checking_permissions", "request_id": "req-a1b2c3"} │
│  {"event": "db_insert_started",    "request_id": "req-a1b2c3"} │
│  {"event": "cache_invalidated",    "request_id": "req-a1b2c3"} │
│  {"event": "notification_queued",  "request_id": "req-a1b2c3"} │
│                                                                 │
│  Now: filter by request_id = "req-a1b2c3" and you see the      │
│  ENTIRE journey of that one request through all layers.         │
│                                                                 │
│  It's the patient wristband. Radiology scans it. Pharmacy      │
│  scans it. Surgery scans it. Every department records against   │
│  the same ID. One search pulls the complete patient history.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How do we attach the ID to every log without passing it as an argument through every function call?**

The answer: Python's `contextvars`. They provide thread-safe (and async-safe) storage that is scoped to the current execution context. structlog integrates with `contextvars` natively.

```
┌─────────────────────────────────────────────────────────────────┐
│           HOW contextvars WORK (Mental Model)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Think of contextvars as an INVISIBLE BACKPACK that each        │
│  request carries. Every function in that request's call chain   │
│  can reach into the backpack and read what's inside.            │
│                                                                 │
│                                                                 │
│  Request A arrives:                                             │
│    backpack = {request_id: "aaa", user_id: 42}                 │
│       │                                                         │
│       ├─ route handler   → reads backpack → sees "aaa", 42     │
│       ├─ service layer   → reads backpack → sees "aaa", 42     │
│       └─ repository      → reads backpack → sees "aaa", 42     │
│                                                                 │
│  Request B arrives (concurrently):                              │
│    backpack = {request_id: "bbb", user_id: 99}                 │
│       │                                                         │
│       ├─ route handler   → reads backpack → sees "bbb", 99     │
│       ├─ service layer   → reads backpack → sees "bbb", 99     │
│       └─ repository      → reads backpack → sees "bbb", 99     │
│                                                                 │
│                                                                 │
│  Each request has its OWN backpack. They never mix up.          │
│  Even in async code with concurrent requests.                   │
│                                                                 │
│  structlog's merge_contextvars processor automatically          │
│  dumps the backpack contents into every log entry.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Setting up correlation IDs with structlog's contextvars:**

```python
# middleware.py
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response


class CorrelationIdMiddleware(BaseHTTPMiddleware):
    """Assign a unique correlation ID to every request."""

    async def dispatch(self, request: Request, call_next) -> Response:
        # Check if the caller already sent a correlation ID
        # (useful when services call each other — the ID propagates)
        correlation_id = request.headers.get(
            "X-Correlation-ID",
            str(uuid.uuid4()),  # Generate one if not provided
        )

        # Clear any leftover context from a previous request
        structlog.contextvars.clear_contextvars()

        # Bind the correlation ID (and other request context)
        # into the invisible backpack for this request
        structlog.contextvars.bind_contextvars(
            correlation_id=correlation_id,
            method=request.method,
            path=request.url.path,
        )

        # Process the request
        response = await call_next(request)

        # Include correlation ID in the response headers
        # so the client can reference it in bug reports
        response.headers["X-Correlation-ID"] = correlation_id

        return response
```

```python
# main.py — register the middleware
from middleware import CorrelationIdMiddleware

app = FastAPI(title=settings.app_name)
app.add_middleware(CorrelationIdMiddleware)
```

**Now every log entry in this request's context gets the correlation ID automatically:**

```python
# routes/tasks.py — no need to pass correlation_id around!
import structlog

logger = structlog.get_logger()

@router.post("/tasks", status_code=201)
async def create_task(
    task_in: TaskCreate,
    current_user: User = Depends(get_current_user),
    service: TaskService = Depends(get_task_service),
):
    # Bind user context (adds to the backpack for this request)
    structlog.contextvars.bind_contextvars(
        user_id=current_user.id,
        org_id=current_user.org_id,
    )
    
    logger.info("task_create_started", title=task_in.title)
    
    task = await service.create_task(task_in, current_user)
    
    logger.info("task_create_success", task_id=task.id)
    
    return task
```

```python
# services/task_service.py — correlation ID follows automatically
import structlog

logger = structlog.get_logger()

class TaskService:
    async def create_task(self, task_in: TaskCreate, user: User) -> Task:
        logger.debug("checking_permissions", role=user.role)
        # ↑ This log AUTOMATICALLY includes correlation_id, user_id,
        #   org_id, method, path — without us passing them here!
        
        task = await self.repo.create(task_in, user.id)
        
        logger.debug("db_insert_complete", task_id=task.id)
        
        return task
```

**The resulting logs (all entries for one request share the same correlation_id):**

```json
{"ts": "2026-02-14T14:23:01Z", "level": "info", "event": "task_create_started", "correlation_id": "a1b2c3d4", "method": "POST", "path": "/api/v1/tasks", "user_id": 42, "org_id": 7, "title": "Fix login bug"}
{"ts": "2026-02-14T14:23:01Z", "level": "debug", "event": "checking_permissions", "correlation_id": "a1b2c3d4", "method": "POST", "path": "/api/v1/tasks", "user_id": 42, "org_id": 7, "role": "member"}
{"ts": "2026-02-14T14:23:01Z", "level": "debug", "event": "db_insert_complete", "correlation_id": "a1b2c3d4", "method": "POST", "path": "/api/v1/tasks", "user_id": 42, "org_id": 7, "task_id": 1847}
{"ts": "2026-02-14T14:23:01Z", "level": "info", "event": "task_create_success", "correlation_id": "a1b2c3d4", "method": "POST", "path": "/api/v1/tasks", "user_id": 42, "org_id": 7, "task_id": 1847}
```

> "Filter by `correlation_id = a1b2c3d4` — you see the entire story of one request, across every layer, in order. In a hospital, pull up patient wristband `#4872` and you see every test, every medication, every procedure — from admission to discharge. That's what a correlation ID gives you."

---

## 4.5 Logging Middleware for FastAPI

**We already have the correlation ID middleware. Let's add request/response logging too.**

This captures timing, status codes, and provides automatic visibility into every request your API handles.

```python
# middleware.py — add to the existing file
import time
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

logger = structlog.get_logger()


class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """Log every request and response with timing."""

    async def dispatch(self, request: Request, call_next) -> Response:
        # ── Correlation ID setup ─────────────────────────────
        correlation_id = request.headers.get(
            "X-Correlation-ID", str(uuid.uuid4())
        )
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            correlation_id=correlation_id,
            method=request.method,
            path=request.url.path,
        )

        # ── Log request start ────────────────────────────────
        logger.info("request_started")

        # ── Process request and measure time ─────────────────
        start_time = time.perf_counter()
        
        try:
            response = await call_next(request)
        except Exception:
            # Unhandled exception — log it as error
            duration_ms = (time.perf_counter() - start_time) * 1000
            logger.exception("request_failed", duration_ms=round(duration_ms, 2))
            raise

        duration_ms = (time.perf_counter() - start_time) * 1000

        # ── Log request completion ───────────────────────────
        logger.info(
            "request_completed",
            status_code=response.status_code,
            duration_ms=round(duration_ms, 2),
        )

        # ── Warn on slow requests ────────────────────────────
        if duration_ms > 500:
            logger.warning(
                "slow_request",
                duration_ms=round(duration_ms, 2),
                threshold_ms=500,
            )

        response.headers["X-Correlation-ID"] = correlation_id
        return response
```

```python
# main.py — register it
app.add_middleware(RequestLoggingMiddleware)
# This replaces the separate CorrelationIdMiddleware —
# this one does both correlation IDs AND request logging.
```

**What this produces for every request:**

```json
{"ts": "...", "level": "info", "event": "request_started", "correlation_id": "a1b2", "method": "GET", "path": "/api/v1/tasks"}
{"ts": "...", "level": "info", "event": "request_completed", "correlation_id": "a1b2", "method": "GET", "path": "/api/v1/tasks", "status_code": 200, "duration_ms": 45.3}
```

And if a request is slow:

```json
{"ts": "...", "level": "warning", "event": "slow_request", "correlation_id": "c3d4", "method": "GET", "path": "/api/v1/reports", "duration_ms": 1823.7, "threshold_ms": 500}
```

> "Now you can see every request, how long it took, and whether it succeeded — without adding a single line of logging code in your route handlers. The middleware captures it all. This is like having a nurse automatically record arrival time, departure time, and outcome for every patient visit."

---

## 4.6 What NOT to Log (Security!)

**This is non-negotiable. Get this wrong and you create a security breach.**

```
┌─────────────────────────────────────────────────────────────────┐
│                 🚨 NEVER LOG THESE 🚨                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ PASSWORDS (plaintext or hashed)                             │
│     logger.info("login", password=password)        ← NEVER     │
│                                                                 │
│  ❌ API KEYS / TOKENS                                          │
│     logger.info("auth", token=jwt_token)           ← NEVER     │
│                                                                 │
│  ❌ CREDIT CARD NUMBERS                                        │
│     logger.info("payment", card="4111111111111111") ← NEVER    │
│                                                                 │
│  ❌ PERSONAL IDENTIFIABLE INFORMATION (PII)                    │
│     logger.info("user", email="alice@example.com") ← CAREFUL   │
│     logger.info("user", ssn="123-45-6789")         ← NEVER    │
│                                                                 │
│  ❌ FULL REQUEST BODIES (may contain any of the above)         │
│     logger.info("request", body=request.body())    ← NEVER    │
│                                                                 │
│  ❌ DATABASE CONNECTION STRINGS (contain passwords!)           │
│     logger.info("connected", url=DATABASE_URL)     ← NEVER    │
│                                                                 │
│                                                                 │
│  ✅ SAFE TO LOG:                                                │
│     logger.info("login_attempt", user_id=42)       ← ID only   │
│     logger.info("auth_success", token_prefix="eyJ") ← Prefix  │
│     logger.info("payment", card_last4="1111")      ← Masked   │
│     logger.info("connected", db_host="prod-rds")   ← Host only │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example — safe authentication logging:**

```python
# ❌ DANGEROUS
async def login(credentials: LoginRequest):
    logger.info("login_attempt", 
                username=credentials.username, 
                password=credentials.password)  # 💀 Password in logs!

# ✅ SAFE
async def login(credentials: LoginRequest):
    logger.info("login_attempt", username=credentials.username)
    # Only log the username, NEVER the password.
    
    user = await authenticate(credentials)
    
    if user:
        logger.info("login_success", user_id=user.id)
    else:
        logger.warning("login_failed", username=credentials.username)
        # Log failure — useful for detecting brute force attacks.
        # Still no password logged.
```

> "In a hospital, the medical chart records that morphine was administered. It does NOT record the combination to the controlled substances cabinet. Your logs record that authentication happened. They do NOT record the credentials used."

---

# PART 5: HEALTH CHECKS & ERROR TRACKING

## 5.1 Liveness vs Readiness (Two Different Questions)

**These two checks answer fundamentally different questions:**

```
┌─────────────────────────────────────────────────────────────────┐
│              LIVENESS VS READINESS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LIVENESS: "Is the process alive?"                              │
│  ──────────────────────────────────                             │
│  • Can the application respond at all?                          │
│  • Is the Python process running?                               │
│  • If NO → restart the container                                │
│                                                                 │
│  Hospital: "Does the patient have a heartbeat?"                 │
│  If no → start resuscitation (restart container)                │
│                                                                 │
│                                                                 │
│  READINESS: "Can it serve real requests?"                       │
│  ─────────────────────────────────────────                      │
│  • Can it reach the database?                                   │
│  • Can it reach Redis?                                          │
│  • Has it finished startup initialization?                      │
│  • If NO → stop sending traffic to this instance                │
│    (but don't restart — it might recover on its own)            │
│                                                                 │
│  Hospital: "Is the patient ready for surgery?"                  │
│  Having a heartbeat ≠ ready for surgery.                        │
│  Need: blood tests done, consent signed, anesthesia available.  │
│  If not ready → postpone surgery (reroute traffic),             │
│  but don't pull the plug (don't restart).                       │
│                                                                 │
│                                                                 │
│       ┌───────────────────────────────────────────┐             │
│       │                                           │             │
│       │  ALIVE?   YES ──▶  READY?   YES ──▶ ✅   │             │
│       │    │                  │               SERVE│            │
│       │   NO                 NO              TRAFFIC            │
│       │    │                  │                    │             │
│       │    ▼                  ▼                    │             │
│       │  🔄 RESTART      🚫 STOP TRAFFIC          │             │
│       │  CONTAINER       (but keep alive)         │             │
│       │                                           │             │
│       └───────────────────────────────────────────┘             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Implementing /health and /ready

**Liveness check — keep it dead simple:**

```python
# routes/health.py
from fastapi import APIRouter

router = APIRouter(tags=["health"])

@router.get("/health")
async def health():
    """Liveness probe. If this responds, the process is alive."""
    return {"status": "healthy"}
```

That's it. No database check. No Redis check. No dependencies. If FastAPI can respond to this endpoint, the process is alive. If it can't, the process is dead or frozen, and the orchestrator (Docker, Kubernetes) should restart it.

**Why no dependency checks in liveness?**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY /health IS SIMPLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  If /health checks the database, and the database is DOWN:     │
│                                                                 │
│  1. /health fails                                               │
│  2. Orchestrator thinks the APP is dead                         │
│  3. Orchestrator RESTARTS the app container                     │
│  4. New container starts... checks database... still down       │
│  5. /health fails again                                         │
│  6. Orchestrator restarts again                                 │
│  7. RESTART LOOP — your app is now flapping endlessly           │
│                                                                 │
│  The DATABASE is the problem, not the APP.                      │
│  Restarting the app doesn't fix the database.                   │
│                                                                 │
│  Liveness = "Is the APP process functioning?"                   │
│  Readiness = "Are the DEPENDENCIES available?"                  │
│                                                                 │
│  Keep them separate. Always.                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Readiness check — actually verify dependencies:**

```python
# routes/health.py
import structlog
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text
from redis.asyncio import Redis

from dependencies import get_db_session, get_redis

router = APIRouter(tags=["health"])
logger = structlog.get_logger()


@router.get("/health")
async def health():
    """Liveness probe."""
    return {"status": "healthy"}


@router.get("/ready")
async def ready(
    db: AsyncSession = Depends(get_db_session),
    redis: Redis = Depends(get_redis),
):
    """Readiness probe. Checks all critical dependencies."""
    
    checks: dict[str, str] = {}
    all_ok = True
    
    # ── Check database ───────────────────────────────────
    try:
        await db.execute(text("SELECT 1"))
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"failed: {type(e).__name__}"
        all_ok = False
        logger.error("readiness_check_failed", dependency="database", error=str(e))
    
    # ── Check Redis ──────────────────────────────────────
    try:
        await redis.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"failed: {type(e).__name__}"
        all_ok = False
        logger.error("readiness_check_failed", dependency="redis", error=str(e))
    
    # ── Return result ────────────────────────────────────
    status_code = 200 if all_ok else 503
    
    from fastapi.responses import JSONResponse
    return JSONResponse(
        status_code=status_code,
        content={
            "status": "ready" if all_ok else "not_ready",
            "checks": checks,
        },
    )
```

**When all dependencies are healthy:**

```json
// GET /ready → 200
{
    "status": "ready",
    "checks": {
        "database": "ok",
        "redis": "ok"
    }
}
```

**When Redis is down:**

```json
// GET /ready → 503
{
    "status": "not_ready",
    "checks": {
        "database": "ok",
        "redis": "failed: ConnectionError"
    }
}
```

**Connecting to Docker health checks (Connection to Lecture 1):**

```yaml
# docker-compose.yml — use YOUR endpoints for health checks
services:
  api:
    build: .
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    # ↑ Docker checks /health every 30 seconds.
    # If it fails 3 times in a row, Docker marks the container
    # as unhealthy. Orchestrators can then restart it.
```

```
┌─────────────────────────────────────────────────────────────────┐
│           HEALTH CHECKS IN THE BIGGER PICTURE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      LOAD BALANCER                              │
│                           │                                     │
│              ┌────────────┼────────────┐                        │
│              │            │            │                         │
│              ▼            ▼            ▼                         │
│         ┌────────┐  ┌────────┐  ┌────────┐                     │
│         │ App #1 │  │ App #2 │  │ App #3 │                     │
│         │ ✅ /rdy│  │ ❌ /rdy│  │ ✅ /rdy│                     │
│         └────────┘  └────────┘  └────────┘                     │
│              ▲                       ▲                          │
│              │                       │                          │
│              └───── traffic ─────────┘                          │
│                  routed only to                                 │
│                  READY instances                                │
│                                                                 │
│  App #2 has a database connection issue.                        │
│  It's ALIVE (liveness passes) but NOT READY.                   │
│  Load balancer stops sending traffic to it.                     │
│  When DB recovers, /ready returns 200 again.                   │
│  Load balancer includes it again. No restart needed.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Error Tracking with Sentry

**Structured logs tell you what happened. Health checks tell you if things are working NOW. But neither one WAKES YOU UP when something goes wrong.**

> "The vital signs monitor beeps. The nurse sees it on rounds. But if there's a cardiac arrest at 3AM, you need a Code Blue alarm that pages the entire team. Sentry is that alarm."

**What Sentry does:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHAT SENTRY PROVIDES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. AUTOMATIC EXCEPTION CAPTURE                                 │
│     Any unhandled exception → captured, grouped, reported       │
│                                                                 │
│  2. RICH CONTEXT                                                │
│     Stack trace, request data, user info, environment           │
│     Not just "Error occurred" — the full crime scene            │
│                                                                 │
│  3. DEDUPLICATION                                               │
│     Same error happening 1000 times → grouped as ONE issue      │
│     with a count, not 1000 separate alerts                      │
│                                                                 │
│  4. ALERTING                                                    │
│     New error → Slack notification, email, PagerDuty            │
│     Configurable: alert on new issues, regressions, spikes     │
│                                                                 │
│  5. PERFORMANCE MONITORING (optional)                           │
│     Traces requests end-to-end, shows slow operations           │
│     (We'll keep this basic for now)                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Setup — surprisingly simple:**

```bash
pip install sentry-sdk
```

```python
# main.py — add Sentry initialization
import sentry_sdk
from dependencies import get_settings

settings = get_settings()

# Initialize Sentry (only if DSN is configured)
if settings.sentry_dsn:
    sentry_sdk.init(
        dsn=settings.sentry_dsn,
        
        # Percentage of requests to trace for performance monitoring
        # 0.1 = 10% of requests (keep this low in production!)
        traces_sample_rate=0.1,
        
        # Tag every event with the environment
        environment=settings.environment,
        
        # FastAPI integration is auto-detected by the SDK.
        # No manual integration setup needed.
    )

app = FastAPI(title=settings.app_name)
```

**That's it. Sentry now automatically captures:**

```python
# This unhandled exception → automatically sent to Sentry
@router.get("/tasks/{task_id}")
async def get_task(task_id: int):
    task = await repo.get(task_id)
    if not task:
        raise HTTPException(status_code=404)  # ← NOT sent (handled)
    
    # But if the database is down:
    # sqlalchemy.exc.OperationalError → SENT to Sentry (unhandled!)
    return task
```

```
┌─────────────────────────────────────────────────────────────────┐
│            WHAT SENTRY CAPTURES VS IGNORES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ CAPTURED (unexpected errors):                               │
│  ├─ Database connection failures                                │
│  ├─ Unhandled exceptions in your code                           │
│  ├─ External API failures (if not caught)                       │
│  ├─ Import errors, syntax errors at runtime                     │
│  └─ Any 500 Internal Server Error                               │
│                                                                 │
│  ❌ NOT CAPTURED (expected behavior):                           │
│  ├─ HTTPException(404) — you raised it intentionally            │
│  ├─ HTTPException(401) — authentication failure (expected)      │
│  ├─ Validation errors — Pydantic doing its job                  │
│  └─ Rate limit responses — working as designed                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Adding user context (so you know WHO was affected):**

```python
# dependencies.py — enrich Sentry context in auth dependency
import sentry_sdk

async def get_current_user(...) -> User:
    # ... your existing JWT validation logic ...
    user = await get_user_from_token(token)
    
    # Tell Sentry who this user is
    # If an error occurs in this request, Sentry shows this info
    sentry_sdk.set_user({
        "id": str(user.id),
        "username": user.username,
        "org_id": str(user.org_id),
    })
    
    return user
```

**Manual exception capture (for errors you handle but still want to track):**

```python
# You handled the error gracefully, but you still want Sentry to know
async def sync_external_data():
    try:
        data = await external_api.fetch()
    except ExternalAPIError as e:
        logger.error("external_sync_failed", error=str(e))
        
        # Tell Sentry about this, even though we handled it
        sentry_sdk.capture_exception(e)
        
        # Return cached/stale data as fallback
        return await get_cached_data()
```

---

## 5.4 The Complete Picture

**Let's see how all three pillars work together in a single request lifecycle:**

```
┌─────────────────────────────────────────────────────────────────┐
│            THE FULL OBSERVABILITY STORY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BEFORE ANY REQUEST:                                            │
│  ├─ Settings validated at startup (pydantic-settings)           │
│  ├─ structlog configured (JSON in prod, pretty in dev)          │
│  ├─ Sentry initialized (if DSN provided)                       │
│  └─ Health checks available (/health, /ready)                   │
│                                                                 │
│                                                                 │
│  REQUEST LIFECYCLE:                                             │
│                                                                 │
│  Client ──▶ FastAPI                                             │
│              │                                                  │
│              ▼                                                  │
│  RequestLoggingMiddleware                                       │
│  ├─ Generate correlation_id                                     │
│  ├─ Bind to contextvars (correlation_id, method, path)          │
│  ├─ Log: "request_started"                                      │
│  ├─ Start timer                                                 │
│  │                                                              │
│  ▼ ── Route Handler ──────────────────────────────────────      │
│  │   Auth dependency → bind user_id, org_id                     │
│  │   Logger calls → all include correlation_id + user context   │
│  │   DB queries → logged with timing                            │
│  │   Redis cache → logged with hit/miss                         │
│  │                                                              │
│  │   IF UNHANDLED EXCEPTION:                                    │
│  │   ├─ Sentry captures it (with full context)                  │
│  │   ├─ Logger records it                                       │
│  │   └─ 500 returned to client                                  │
│  │                                                              │
│  ▼ ── Back to Middleware ─────────────────────────────────      │
│  ├─ Stop timer                                                  │
│  ├─ Log: "request_completed" (status_code, duration_ms)         │
│  ├─ Log: "slow_request" (if duration > threshold)               │
│  └─ Return response with X-Correlation-ID header                │
│                                                                 │
│              │                                                  │
│  Client ◀──┘                                                   │
│                                                                 │
│                                                                 │
│  MEANWHILE, CONTINUOUSLY:                                       │
│  ├─ Docker checks /health every 30s (is process alive?)         │
│  ├─ Load balancer checks /ready (can it serve traffic?)         │
│  └─ Sentry monitors for new error types (alert team)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Here's the complete startup wiring — all the pieces together:**

```python
# main.py — the full production-ready setup
import sentry_sdk
import structlog
from fastapi import FastAPI

from config import Settings
from dependencies import get_settings
from logging_config import configure_logging
from middleware import RequestLoggingMiddleware
from routes import tasks, health


# ── Load and validate ALL configuration at startup ────────
settings = get_settings()
# If ANY required env var is missing or invalid → crash NOW

# ── Configure structured logging ─────────────────────────
configure_logging(settings)
logger = structlog.get_logger()

# ── Initialize Sentry (only if configured) ───────────────
if settings.sentry_dsn:
    sentry_sdk.init(
        dsn=settings.sentry_dsn,
        traces_sample_rate=0.1 if settings.is_production else 1.0,
        environment=settings.environment,
    )
    logger.info("sentry_initialized", environment=settings.environment)

# ── Create FastAPI app ───────────────────────────────────
app = FastAPI(
    title=settings.app_name,
    debug=settings.debug,
    docs_url="/docs" if not settings.is_production else None,
    redoc_url="/redoc" if not settings.is_production else None,
)

# ── Middleware (order matters — last added runs first) ───
app.add_middleware(RequestLoggingMiddleware)

# ── Routes ───────────────────────────────────────────────
app.include_router(health.router)
app.include_router(tasks.router, prefix=settings.api_v1_prefix)

# ── Startup log ──────────────────────────────────────────
logger.info(
    "application_started",
    app_name=settings.app_name,
    environment=settings.environment,
    debug=settings.debug,
    log_level=settings.log_level,
    sentry_enabled=settings.sentry_dsn is not None,
)
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│          CONFIGURATION, SECRETS & OBSERVABILITY CHEAT SHEET     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONFIGURATION:                                                 │
│    class Settings(BaseSettings):                                │
│        model_config = SettingsConfigDict(env_file=".env")       │
│        database_url: str        # Required, from DATABASE_URL   │
│        debug: bool = False      # Optional, auto-coerced        │
│                                                                 │
│  SINGLETON SETTINGS:                                            │
│    @lru_cache                                                   │
│    def get_settings() -> Settings:                              │
│        return Settings()                                        │
│                                                                 │
│  SECRETS:                                                       │
│    .env          → local dev (in .gitignore, NEVER committed)   │
│    .env.example  → template (committed, no real values)         │
│    Production    → cloud platform secrets/env vars              │
│                                                                 │
│  STRUCTURED LOGGING:                                            │
│    logger = structlog.get_logger()                              │
│    logger.info("event_name", key=value, key2=value2)            │
│    # Event name = WHAT. Kwargs = CONTEXT.                       │
│                                                                 │
│  BOUND CONTEXT:                                                 │
│    log = logger.bind(user_id=42, org_id=7)                     │
│    log.info("task_created", task_id=1)  # user_id + org_id auto │
│                                                                 │
│  CORRELATION ID:                                                │
│    structlog.contextvars.bind_contextvars(correlation_id=uuid)  │
│    # All subsequent logs in this request include it             │
│                                                                 │
│  HEALTH CHECKS:                                                 │
│    GET /health → always 200 (process alive)                     │
│    GET /ready  → 200 if DB+Redis OK, 503 if not                │
│                                                                 │
│  SENTRY:                                                        │
│    sentry_sdk.init(dsn=..., environment=...)                    │
│    # Unhandled exceptions auto-captured                         │
│    sentry_sdk.capture_exception(e)  # Manual capture            │
│                                                                 │
│  COMMON MISTAKES:                                               │
│    ❌ Hardcoded config          → Use pydantic-settings         │
│    ❌ Secrets in git            → Use .env + .gitignore         │
│    ❌ print() for logging       → Use structlog                 │
│    ❌ Passwords in logs         → Log IDs, never credentials    │
│    ❌ /health checks database   → That's /ready's job           │
│    ❌ No correlation ID         → Can't trace requests          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  YOUR APP IN PRODUCTION = PATIENT IN THE ICU                    │
│                                                                 │
│                                                                 │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐              │
│  │CONFIGURE │      │ PROTECT  │      │ OBSERVE  │              │
│  │          │      │          │      │          │              │
│  │ pydantic │      │ .env +   │      │structlog │              │
│  │ settings │      │.gitignore│      │ + sentry │              │
│  └────┬─────┘      └────┬─────┘      └────┬─────┘              │
│       │                 │                 │                     │
│       ▼                 ▼                 ▼                     │
│  Treatment plan    Locked cabinet     Vital signs               │
│  (per patient)     (controlled        monitor + chart           │
│                     substances)       + code blue alarm         │
│                                                                 │
│                                                                 │
│  THE PRINCIPLES:                                                │
│  ├─ Config comes from OUTSIDE the code (environment)            │
│  ├─ Config is VALIDATED at startup (fail fast, not at 3AM)      │
│  ├─ Secrets are NEVER in code, NEVER in git, NEVER in logs      │
│  ├─ Logs are STRUCTURED (JSON), CONTEXTUAL (correlation ID),    │
│  │   and LEVELED (debug/info/warning/error/critical)            │
│  ├─ Health checks are SEPARATED (liveness ≠ readiness)          │
│  └─ Errors are TRACKED and ALERTED (Sentry, not log grep)      │
│                                                                 │
│                                                                 │
│  CODE CHANGES FOR FEATURES.                                     │
│  ENVIRONMENT CHANGES FOR CONFIG.                                │
│  LOGS TELL YOU WHAT HAPPENED.                                   │
│  HEALTH CHECKS TELL YOU WHAT'S HAPPENING NOW.                  │
│  SENTRY TELLS YOU WHAT WENT WRONG.                              │
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
│  WEEK 15, LECTURE 3 (CI/CD Pipelines):                          │
│  └─ Your Settings class defines what env vars the pipeline      │
│     must provide. CI tests use ENVIRONMENT=testing.             │
│     CD sets production env vars in the deployment platform.     │
│     .env.example documents what the pipeline needs to set.      │
│                                                                 │
│  WEEK 15, LECTURE 4 (Cloud Fundamentals):                       │
│  └─ Cloud platforms have dedicated secrets managers              │
│     (AWS Secrets Manager, GCP Secret Manager).                  │
│     Your pydantic-settings reads from env vars regardless       │
│     of HOW they were injected — same code, different source.    │
│                                                                 │
│  WEEK 16, LECTURE 1 (System Design):                            │
│  └─ In distributed systems, correlation IDs propagate           │
│     across services via X-Correlation-ID headers.               │
│     One user action → traced through 5 microservices.           │
│     Structured logging is the foundation of observability       │
│     at scale.                                                   │
│                                                                 │
│  WEEK 16, LECTURE 2 (Architecture Patterns):                    │
│  └─ Health checks become critical with load balancers.          │
│     /ready determines which instances receive traffic.          │
│     12-factor config enables horizontal scaling —               │
│     spin up 10 instances, each reads the same env vars.         │
│                                                                 │
│  CAPSTONE FINAL DELIVERABLE:                                    │
│  └─ Your capstone MUST have:                                    │
│     ├─ pydantic-settings configuration (no hardcoded values)    │
│     ├─ .env.example checked into git                            │
│     ├─ Structured logging with correlation IDs                  │
│     ├─ /health and /ready endpoints                             │
│     └─ Sentry integration (DSN configurable, optional in dev)   │
│     This lecture gives you EVERYTHING you need for that.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```