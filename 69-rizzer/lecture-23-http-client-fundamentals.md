# LECTURE DESIGN PHILOSOPHY

```
┌─────────────────────────────────────────────────────────────────┐
│                    PEDAGOGICAL APPROACH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FLIP THE PERSPECTIVE                                           │
│  ────────────────────                                           │
│  For 5 weeks, students RECEIVED HTTP requests. They built       │
│  the server. Now they SEND them. Same protocol, opposite side.  │
│  Every concept has a mirror image they already understand.      │
│                                                                 │
│  FEEL THE FAILURE FIRST                                         │
│  ──────────────────────                                         │
│  Students must watch their API HANG, CASCADE, and DIE because   │
│  of an external service before we teach them how to defend.     │
│  Fear is a better teacher than syntax.                          │
│                                                                 │
│  DEFENSE IN DEPTH                                               │
│  ────────────────                                               │
│  Timeouts, retries, connection pooling — each is a LAYER of     │
│  protection. We stack them one by one, building a fortress.     │
│  Each layer is useless alone, lethal in combination.            │
│                                                                 │
│  EXTEND THE ANALOGY                                             │
│  ──────────────────                                             │
│  The restaurant from Week 1 now needs SUPPLIERS (external       │
│  APIs). Every concept maps to supply chain logistics.           │
│                                                                 │
│  CONNECT TO PRIOR LECTURES                                      │
│  ─────────────────────────                                      │
│  Context managers (W1) → AsyncClient lifecycle                  │
│  Error handling (W1) → httpx exception hierarchy                │
│  Pydantic (W3) → Validating external API responses              │
│  FastAPI Depends (W3) → httpx client as dependency              │
│  Connection pooling (W7) → Same concept, HTTP layer             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## LECTURE OUTLINE

```
┌─────────────────────────────────────────────────────────────────┐
│                  HTTP CLIENT FUNDAMENTALS                       │
│                     (3.5-4 Hour Lecture)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PART 1: THE PARADIGM SHIFT (30 min)                            │
│  ├─ 1.1 Your Server Becomes a Client (Demonstration)            │
│  ├─ 1.2 The Danger Zone (Everything That Can Go Wrong)          │
│  └─ 1.3 The Supply Chain Analogy                                │
│                                                                 │
│  PART 2: HTTPX — YOUR HTTP CLIENT (30 min)                      │
│  ├─ 2.1 Why httpx? (And Why Not requests)                       │
│  ├─ 2.2 Sync vs Async in One Library                            │
│  ├─ 2.3 The Response Object (Anatomy)                           │
│  └─ 2.4 raise_for_status() — Turning Codes into Exceptions     │
│                                                                 │
│  PART 3: REQUEST PATTERNS (45 min)                              │
│  ├─ 3.1 GET Requests (Query Parameters, Headers)                │
│  ├─ 3.2 POST Requests (JSON Body)                               │
│  ├─ 3.3 Custom Headers and Authentication                       │
│  ├─ 3.4 Validating External Data with Pydantic (Week 3 Link)   │
│  └─ 3.5 Complete Example: Calling a Real API                    │
│                                                                 │
│  PART 4: TIMEOUTS — NON-NEGOTIABLE (40 min)                     │
│  ├─ 4.1 The Hanging Endpoint (Demonstration)                    │
│  ├─ 4.2 The Four Timeout Types                                  │
│  ├─ 4.3 Configuring httpx Timeouts                              │
│  ├─ 4.4 Cascading Failures (The Domino Effect)                  │
│  └─ 4.5 Choosing Timeout Values                                 │
│                                                                 │
│  PART 5: RETRY STRATEGIES (40 min)                              │
│  ├─ 5.1 Transient vs Permanent Failures                         │
│  ├─ 5.2 Naive Retry (And Why It Backfires)                      │
│  ├─ 5.3 Exponential Backoff with Jitter                         │
│  ├─ 5.4 The tenacity Library                                    │
│  └─ 5.5 Combining Retries with Timeouts                         │
│                                                                 │
│  PART 6: CONNECTION POOLING (25 min)                            │
│  ├─ 6.1 The TCP Handshake Tax                                   │
│  ├─ 6.2 AsyncClient as Connection Pool                          │
│  ├─ 6.3 Client Lifecycle (Context Managers Revisited)           │
│  └─ 6.4 AsyncClient as FastAPI Dependency                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 1: THE PARADIGM SHIFT

## 1.1 Your Server Becomes a Client

**Start with a question. Reframe everything they know.**

> "For the past 5 weeks, you've built APIs. Clients sent requests to YOUR server, and YOU sent responses back. You controlled both sides: the validation, the database, the error handling. Now here's the question: what happens when YOUR server needs data from SOMEONE ELSE'S server?"

```
┌─────────────────────────────────────────────────────────────────┐
│              THE PERSPECTIVE FLIP                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WEEKS 3–7: You were the SERVER                                 │
│                                                                 │
│   Client ───Request──▶ [YOUR API] ───Query──▶ [YOUR DB]        │
│   Client ◀──Response── [YOUR API] ◀──Result── [YOUR DB]        │
│                                                                 │
│   You controlled EVERYTHING.                                    │
│                                                                 │
│                                                                 │
│  WEEK 8: Your server is ALSO a client                           │
│                                                                 │
│   Client ──▶ [YOUR API] ──▶ [EXTERNAL API] ──▶ [THEIR DB]     │
│   Client ◀── [YOUR API] ◀── [EXTERNAL API] ◀── [THEIR DB]     │
│                                                                 │
│   You control YOUR API.                                         │
│   You control NOTHING about the external API.                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Run this demonstration. Let them watch the architecture.**

```python
# demo_middleman.py — Your API as a middleman
from fastapi import FastAPI
import httpx

app = FastAPI()

# YOUR endpoint that a client calls
@app.get("/dashboard/{city}")
async def get_dashboard(city: str):
    """
    Client asks YOUR API for a dashboard.
    YOUR API needs weather data from an EXTERNAL API to build it.
    """
    async with httpx.AsyncClient() as client:
        # YOUR server is now a CLIENT to someone else's server
        response = await client.get(
            f"https://api.weatherservice.com/v1/current?city={city}"
        )
        weather = response.json()

    return {
        "city": city,
        "weather": weather,
        "message": f"Dashboard for {city}",
    }
```

**Draw the full request chain on the board:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE REQUEST CHAIN                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Mobile App                  Your FastAPI              Weather  │
│  (client)                    (server AND client)       API      │
│  ─────────                   ──────────────────        ───────  │
│      │                              │                     │     │
│      │ ── GET /dashboard/London ──▶ │                     │     │
│      │                              │                     │     │
│      │                              │ ── GET /current ──▶ │     │
│      │                              │    ?city=London      │     │
│      │                              │                     │     │
│      │         Your API is          │    😴 Waiting...    │     │
│      │         BLOCKED here.        │                     │     │
│      │         Your CLIENT          │ ◀── 200 OK ─────── │     │
│      │         is ALSO waiting.     │    {temp: 8}        │     │
│      │                              │                     │     │
│      │ ◀── 200 OK ──────────────── │                     │     │
│      │    {city: "London",          │                     │     │
│      │     weather: {temp: 8}}      │                     │     │
│      │                              │                     │     │
│                                                                 │
│  YOUR API'S RESPONSE TIME = Your code + External API's time     │
│                                                                 │
│  You can optimize YOUR code all day.                            │
│  You cannot optimize THEIR server.                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The insight:**

> "Your API is now only as fast and as reliable as the SLOWEST, LEAST RELIABLE external service it depends on. If their API takes 30 seconds, YOUR client waits 30 seconds. If their API goes down, YOUR dashboard endpoint is broken. You've introduced a dependency you don't control."

---

## 1.2 The Danger Zone (Everything That Can Go Wrong)

**List every failure mode. Make them uncomfortable.**

```
┌─────────────────────────────────────────────────────────────────┐
│            EVERYTHING THAT CAN GO WRONG                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NETWORK FAILURES                                               │
│  ├─ DNS resolution fails (domain doesn't exist)                │
│  ├─ Connection refused (server is down)                        │
│  ├─ Connection reset (server crashed mid-response)             │
│  ├─ Network unreachable (routing broken)                       │
│  └─ SSL/TLS handshake fails (certificate issues)              │
│                                                                 │
│  TIMEOUT FAILURES                                               │
│  ├─ Connection timeout (server not responding to TCP SYN)      │
│  ├─ Read timeout (connected but response never arrives)        │
│  ├─ Write timeout (can't send request body fast enough)        │
│  └─ Pool timeout (all connections busy, can't get one)         │
│                                                                 │
│  HTTP FAILURES (server responds, but says "no")                 │
│  ├─ 400 Bad Request (you sent garbage)                         │
│  ├─ 401/403 (auth invalid or forbidden)                        │
│  ├─ 404 (endpoint doesn't exist)                               │
│  ├─ 429 Too Many Requests (rate limited!)                      │
│  ├─ 500 Internal Server Error (their bug)                      │
│  └─ 503 Service Unavailable (their server overloaded)          │
│                                                                 │
│  DATA FAILURES (server responds 200, but data is wrong)         │
│  ├─ Schema changed without notice (field renamed/removed)      │
│  ├─ Unexpected null values                                     │
│  ├─ Different data types than documented                       │
│  ├─ Truncated response (connection dropped mid-transfer)       │
│  └─ Valid JSON but wrong structure entirely                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "When you query YOUR PostgreSQL database, how often does it lie to you? Almost never — you control the schema, the data, the connection. External APIs are a different universe. They change without warning, go down without notice, rate limit without mercy, and return data that doesn't match their own documentation."

**Contrast with what they're used to:**

```
┌─────────────────────────────────────────────────────────────────┐
│           YOUR DATABASE  vs  EXTERNAL API                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your PostgreSQL               External API                     │
│  ──────────────                ────────────                     │
│  Same network (fast)           Internet (slow, variable)        │
│  You control the schema        They change whenever they want   │
│  Always available*             Down for maintenance at 3am      │
│  Millisecond responses         100ms—10s responses              │
│  No rate limits                429 if you call too often        │
│  You trust the data            You trust NOTHING                │
│  Errors are your bugs          Errors are their bugs            │
│                                                                 │
│  * Except when it isn't. But you have Alembic, backups,        │
│    connection pooling, retry logic. You need the SAME           │
│    defensive mindset for HTTP clients.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 The Supply Chain Analogy

**Extend the restaurant analogy from Week 1.**

```
┌─────────────────────────────────────────────────────────────────┐
│               THE SUPPLY CHAIN ANALOGY                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Week 1 Analogy:                                                │
│  Your restaurant serves customers. Waiters are coroutines.      │
│  The event loop is the manager.                                 │
│                                                                 │
│  Week 8 Extension:                                              │
│  Your restaurant has grown. You need INGREDIENTS from           │
│  SUPPLIERS. You can't grow your own tomatoes — you ORDER them.  │
│                                                                 │
│                                                                 │
│  Restaurant                  │  Your Backend                    │
│  ───────────────────────     │  ───────────────────────         │
│  Customers order dishes      │  Clients call your API           │
│  You need ingredients        │  You need external data          │
│  You call a supplier         │  You make an HTTP request        │
│  Phone rings...              │  TCP connection opening...       │
│  No answer after 30 sec      │  Connection timeout              │
│  Supplier says "try later"   │  HTTP 503 / 429 response        │
│  You call back in 5 min      │  Retry with backoff              │
│  Dedicated phone line to     │  Persistent connection pool      │
│    your top supplier         │    to frequently-called APIs     │
│  You DON'T wait on the       │  You DON'T block your event      │
│    phone — you serve          │    loop — you use async           │
│    other tables while         │    while awaiting the response    │
│    the order is prepared      │                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The key framing:**

> "For the rest of this lecture, think of every HTTP request as a phone call to a supplier. Suppliers are unreliable. They're slow. They put you on hold. They change their phone numbers. Your job is to build a system that serves customers even when your suppliers are having a bad day."

```
┌─────────────────────────────────────────────────────────────────┐
│               WHAT WE'LL BUILD TODAY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER 1: httpx basics        → How to make the phone call     │
│  LAYER 2: Response handling   → How to understand the answer   │
│  LAYER 3: Timeouts            → How long to wait before        │
│                                  hanging up                     │
│  LAYER 4: Retries             → When and how to call back      │
│  LAYER 5: Connection pooling  → Keep the line open for         │
│                                  frequent suppliers             │
│                                                                 │
│  Each layer protects you. Combined, they make you resilient.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 2: HTTPX — YOUR HTTP CLIENT

## 2.1 Why httpx? (And Why Not `requests`)

**Address the elephant in the room:**

> "Many of you have heard of the `requests` library. It's the most popular Python HTTP client. So why aren't we using it?"

```
┌─────────────────────────────────────────────────────────────────┐
│              requests  vs  httpx                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  requests                         httpx                         │
│  ────────                         ─────                         │
│  Sync only ❌                     Sync AND Async ✅             │
│  Blocks the event loop            Works with asyncio             │
│  HTTP/1.1 only                    HTTP/1.1 and HTTP/2            │
│  No async support at all          First-class async support      │
│  Massive ecosystem                requests-compatible API        │
│  (many know it)                   (easy to switch)               │
│                                                                 │
│  THE DEAL-BREAKER:                                              │
│  ─────────────────                                              │
│  You're writing FastAPI applications. FastAPI is ASYNC.          │
│  If you use requests.get() inside an async endpoint,            │
│  you BLOCK the event loop. Remember Week 1?                     │
│                                                                 │
│  requests.get()  =  time.sleep()  =  😴 BLOCKED                │
│  await client.get()  =  await asyncio.sleep()  =  ✅ FREE      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ❌ NEVER do this in FastAPI
from fastapi import FastAPI
import requests  # Sync-only library

app = FastAPI()

@app.get("/weather/{city}")
async def get_weather(city: str):
    # This BLOCKS the event loop! Same disaster as time.sleep()
    # Every other request to your API is frozen while this runs.
    response = requests.get(f"https://api.weather.com/{city}")
    return response.json()


# ✅ Use httpx async instead
import httpx

@app.get("/weather/{city}")
async def get_weather(city: str):
    async with httpx.AsyncClient() as client:
        # This YIELDS control to the event loop. Other requests keep flowing.
        response = await client.get(f"https://api.weather.com/{city}")
        return response.json()
```

**One more reason:**

> "You've already used httpx. In Week 4, you used `httpx.AsyncClient` to test your FastAPI endpoints. Now you're using the same tool for a different purpose — calling EXTERNAL services instead of testing your own."

---

## 2.2 Sync vs Async in One Library

**httpx gives you both. You'll use async. But know both exist.**

```python
import httpx

# ─────────────────────────────────────────
# SYNC MODE — quick scripts, one-off calls
# ─────────────────────────────────────────

# One-shot request (opens and closes connection each time)
response = httpx.get("https://api.github.com/users/octocat")
print(response.status_code)  # 200
print(response.json())       # {...}

# With a client (connection reuse)
with httpx.Client() as client:
    r1 = client.get("https://api.github.com/users/octocat")
    r2 = client.get("https://api.github.com/users/torvalds")


# ─────────────────────────────────────────
# ASYNC MODE — what you'll use in FastAPI
# ─────────────────────────────────────────

# One-shot async request
async with httpx.AsyncClient() as client:
    response = await client.get("https://api.github.com/users/octocat")
    print(response.status_code)
    print(response.json())

# Concurrent requests (the real power)
async with httpx.AsyncClient() as client:
    responses = await asyncio.gather(
        client.get("https://api.github.com/users/octocat"),
        client.get("https://api.github.com/users/torvalds"),
        client.get("https://api.github.com/users/gvanrossum"),
    )
```

**When to use which:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  USE SYNC httpx:                                                │
│  ├─ Quick scripts and notebooks                                │
│  ├─ CLI tools (like your Week 2 project, before you learned    │
│  │   async — actually, you used async there already)           │
│  └─ Anywhere outside an async context                          │
│                                                                 │
│  USE ASYNC httpx:                                               │
│  ├─ Inside FastAPI endpoints (always)                           │
│  ├─ Inside any async function                                  │
│  ├─ When making multiple HTTP calls concurrently               │
│  └─ Basically: whenever you're in async code, which is always  │
│     in this course                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "For the rest of this lecture, every example uses async mode. If you need sync, the API is identical — just remove `async`, `await`, and `AsyncClient` becomes `Client`."

---

## 2.3 The Response Object (Anatomy)

**When you make a request, httpx returns a `Response` object. Know every part.**

```python
import httpx

async with httpx.AsyncClient() as client:
    response = await client.get("https://api.github.com/users/octocat")
```

```
┌─────────────────────────────────────────────────────────────────┐
│               THE RESPONSE OBJECT                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  response.status_code          200                              │
│  ├─ The HTTP status code                                       │
│  └─ You know these from Week 3 (2xx, 4xx, 5xx families)       │
│                                                                 │
│  response.headers              {'content-type': 'application/  │
│  ├─ Response headers             json', 'x-ratelimit-          │
│  └─ Case-insensitive dict        remaining': '59', ...}        │
│                                                                 │
│  response.json()               {"login": "octocat", ...}       │
│  ├─ Parse body as JSON                                         │
│  └─ Raises if body isn't valid JSON                            │
│                                                                 │
│  response.text                 '{"login": "octocat", ...}'     │
│  └─ Raw response body as string                                │
│                                                                 │
│  response.content              b'{"login": "octocat", ...}'    │
│  └─ Raw response body as bytes                                 │
│                                                                 │
│  response.url                  URL('https://api.github...')     │
│  └─ The final URL (after any redirects)                        │
│                                                                 │
│  response.elapsed              datetime.timedelta(0, 0, 23456) │
│  └─ How long the request took                                  │
│                                                                 │
│  response.is_success           True                             │
│  └─ True if status_code is 2xx                                 │
│                                                                 │
│  response.is_client_error      False                            │
│  └─ True if status_code is 4xx                                 │
│                                                                 │
│  response.is_server_error      False                            │
│  └─ True if status_code is 5xx                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Explore it interactively:**

```python
import httpx
import asyncio

async def explore_response():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.github.com/users/octocat")

    # Status
    print(f"Status: {response.status_code}")          # 200
    print(f"Success? {response.is_success}")           # True

    # Headers (case-insensitive!)
    print(f"Content-Type: {response.headers['content-type']}")
    print(f"Rate Limit Remaining: {response.headers.get('x-ratelimit-remaining')}")

    # Body
    data = response.json()
    print(f"Username: {data['login']}")                # octocat
    print(f"Name: {data['name']}")                     # The Octocat

    # Timing
    print(f"Request took: {response.elapsed.total_seconds():.3f}s")

asyncio.run(explore_response())
```

---

## 2.4 raise_for_status() — Turning Codes into Exceptions

**You have two choices for handling bad status codes:**

```python
# APPROACH 1: Manual checking
async def manual_check(client: httpx.AsyncClient) -> dict:
    response = await client.get("https://api.example.com/data")

    if response.status_code == 404:
        raise ValueError("Resource not found")
    elif response.status_code == 429:
        raise ValueError("Rate limited")
    elif response.status_code >= 500:
        raise ValueError("Server error")
    elif not response.is_success:
        raise ValueError(f"Unexpected status: {response.status_code}")

    return response.json()


# APPROACH 2: raise_for_status() — let httpx do it
async def auto_check(client: httpx.AsyncClient) -> dict:
    response = await client.get("https://api.example.com/data")
    response.raise_for_status()  # Raises httpx.HTTPStatusError for 4xx/5xx
    return response.json()
```

**What `raise_for_status()` actually does:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 raise_for_status()                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Status Code     What Happens                                   │
│  ───────────     ────────────                                   │
│  200–299         Nothing. Returns normally.                     │
│  301,302,307     httpx follows redirects automatically.         │
│                  You won't even see these.                      │
│  400–499         Raises httpx.HTTPStatusError (client error)    │
│  500–599         Raises httpx.HTTPStatusError (server error)    │
│                                                                 │
│  The exception includes:                                        │
│  ├─ e.response         → the full Response object              │
│  ├─ e.response.status_code → the status code                   │
│  ├─ e.response.json()  → the error body (if JSON)              │
│  └─ e.request          → the original Request object           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Catching it properly:**

```python
async def fetch_user_profile(client: httpx.AsyncClient, username: str) -> dict:
    """Fetch a user profile, with proper error handling."""
    try:
        response = await client.get(
            f"https://api.github.com/users/{username}"
        )
        response.raise_for_status()
        return response.json()

    except httpx.HTTPStatusError as e:
        # Server responded, but with an error status
        if e.response.status_code == 404:
            raise UserNotFoundError(f"User '{username}' not found")
        elif e.response.status_code == 403:
            raise RateLimitError("GitHub API rate limit exceeded")
        else:
            raise ExternalAPIError(
                f"GitHub API error: {e.response.status_code}"
            )

    except httpx.RequestError as e:
        # Network-level failure: couldn't reach the server at all
        raise ExternalAPIError(f"Could not reach GitHub: {e}")
```

**Now let's see the full httpx exception hierarchy:**

```
┌─────────────────────────────────────────────────────────────────┐
│              HTTPX EXCEPTION HIERARCHY                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  httpx.HTTPError  (catch-all base)                              │
│  ├─ httpx.HTTPStatusError                                      │
│  │   └─ Raised by raise_for_status() for 4xx/5xx              │
│  │     Has: .response, .request                                │
│  │                                                             │
│  └─ httpx.RequestError  (network/transport problems)           │
│      ├─ httpx.TransportError                                   │
│      │   ├─ httpx.TimeoutException                             │
│      │   │   ├─ httpx.ConnectTimeout                           │
│      │   │   ├─ httpx.ReadTimeout                              │
│      │   │   ├─ httpx.WriteTimeout                             │
│      │   │   └─ httpx.PoolTimeout                              │
│      │   ├─ httpx.ConnectError                                 │
│      │   ├─ httpx.ReadError                                    │
│      │   └─ httpx.CloseError                                   │
│      ├─ httpx.DecodingError                                    │
│      └─ httpx.TooManyRedirects                                 │
│                                                                 │
│  REMEMBER from Week 1: Exception hierarchies let you           │
│  catch at different levels of specificity.                      │
│                                                                 │
│  catch httpx.ConnectTimeout  → only connection timeouts        │
│  catch httpx.TimeoutException → ANY timeout                    │
│  catch httpx.TransportError  → any network problem             │
│  catch httpx.RequestError    → any request failure             │
│  catch httpx.HTTPError       → absolutely anything             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "This is exactly the exception hierarchy pattern you built in Week 1 — base class → specific subclasses. Same principle, different library. Catch broadly for generic fallbacks, catch narrowly when you need specific recovery logic."

---

# PART 3: REQUEST PATTERNS

## 3.1 GET Requests (Query Parameters, Headers)

**You know what GET means from Week 3. Now you're the one sending it.**

```python
import httpx

async def demo_get_requests():
    async with httpx.AsyncClient() as client:

        # ─────────────────────────────────────────
        # Simple GET
        # ─────────────────────────────────────────
        response = await client.get("https://api.github.com/users/octocat")


        # ─────────────────────────────────────────
        # GET with query parameters
        # ─────────────────────────────────────────

        # ❌ DON'T build URLs by hand (special characters will break it)
        response = await client.get(
            "https://api.github.com/search/repositories?q=fastapi&sort=stars"
        )

        # ✅ Use the params argument (httpx handles URL encoding)
        response = await client.get(
            "https://api.github.com/search/repositories",
            params={
                "q": "fastapi",
                "sort": "stars",
                "order": "desc",
                "per_page": 10,
            }
        )
        # httpx builds: /search/repositories?q=fastapi&sort=stars&order=desc&per_page=10


        # ─────────────────────────────────────────
        # GET with custom headers
        # ─────────────────────────────────────────
        response = await client.get(
            "https://api.github.com/users/octocat",
            headers={
                "Accept": "application/vnd.github.v3+json",
                "User-Agent": "my-backend-app/1.0",
            }
        )
```

**Why `params={}` instead of string concatenation:**

```
┌─────────────────────────────────────────────────────────────────┐
│          WHY USE params= INSTEAD OF f-STRINGS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  search_term = "fast api web"                                   │
│                                                                 │
│  ❌ f-string:                                                   │
│  f"https://api.com/search?q={search_term}"                      │
│  → "https://api.com/search?q=fast api web"                      │
│     Broken! Spaces aren't valid in URLs.                        │
│                                                                 │
│  ✅ params=:                                                    │
│  client.get("https://api.com/search", params={"q": search_term})│
│  → "https://api.com/search?q=fast%20api%20web"                  │
│     httpx URL-encodes automatically.                            │
│                                                                 │
│  Also handles: &, =, ?, #, non-ASCII characters, etc.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 POST Requests (JSON Body)

**Sending data to an external API:**

```python
async def demo_post_requests():
    async with httpx.AsyncClient() as client:

        # ─────────────────────────────────────────
        # POST with JSON body (most common)
        # ─────────────────────────────────────────
        response = await client.post(
            "https://api.example.com/webhooks",
            json={
                "event": "user.created",
                "data": {"user_id": 42, "email": "alice@example.com"},
            }
        )
        # json= automatically:
        #   1. Serializes dict to JSON string
        #   2. Sets Content-Type: application/json


        # ─────────────────────────────────────────
        # POST with form data (less common, some APIs require it)
        # ─────────────────────────────────────────
        response = await client.post(
            "https://api.example.com/oauth/token",
            data={
                "grant_type": "client_credentials",
                "client_id": "my-app",
                "client_secret": "secret123",
            }
        )
        # data= automatically:
        #   1. URL-encodes the data
        #   2. Sets Content-Type: application/x-www-form-urlencoded
```

**`json=` vs `data=` vs `content=`:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SENDING DATA                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  json={"key": "value"}                                          │
│  ├─ Sends JSON body                                            │
│  ├─ Auto-sets Content-Type: application/json                   │
│  └─ USE THIS for most API calls                                │
│                                                                 │
│  data={"key": "value"}                                          │
│  ├─ Sends form-encoded body                                    │
│  ├─ Auto-sets Content-Type: application/x-www-form-urlencoded  │
│  └─ USE THIS for OAuth token endpoints, legacy forms           │
│                                                                 │
│  content=b"raw bytes"                                           │
│  ├─ Sends raw bytes                                            │
│  ├─ YOU must set Content-Type header manually                  │
│  └─ USE THIS for file uploads, binary data                     │
│                                                                 │
│  RULE: If the external API docs say "JSON body", use json=.    │
│        If they say "form data", use data=.                      │
│        When in doubt, json= is almost always right.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.3 Custom Headers and Authentication

**Most external APIs require authentication. Here are the common patterns:**

```python
async def demo_auth_patterns():
    async with httpx.AsyncClient() as client:

        # ─────────────────────────────────────────
        # Pattern 1: API Key in header
        # (Most common for third-party services)
        # ─────────────────────────────────────────
        response = await client.get(
            "https://api.openweathermap.org/data/2.5/weather",
            params={"q": "London"},
            headers={"X-API-Key": "your-api-key-here"},
        )


        # ─────────────────────────────────────────
        # Pattern 2: API Key as query parameter
        # (Some APIs do this — less secure, appears in logs)
        # ─────────────────────────────────────────
        response = await client.get(
            "https://api.openweathermap.org/data/2.5/weather",
            params={
                "q": "London",
                "appid": "your-api-key-here",  # Key in URL
            },
        )


        # ─────────────────────────────────────────
        # Pattern 3: Bearer token (OAuth2, JWT)
        # (You'll build this yourself in Week 9!)
        # ─────────────────────────────────────────
        response = await client.get(
            "https://api.github.com/user",
            headers={"Authorization": "Bearer ghp_abc123..."},
        )


        # ─────────────────────────────────────────
        # Pattern 4: Basic Auth
        # (username:password, base64-encoded)
        # ─────────────────────────────────────────
        response = await client.get(
            "https://api.example.com/private",
            auth=("username", "password"),  # httpx handles encoding
        )
```

**Set default headers on the client to avoid repetition:**

```python
# ✅ Set headers ONCE on the client, applied to EVERY request
async with httpx.AsyncClient(
    base_url="https://api.github.com",
    headers={
        "Authorization": "Bearer ghp_abc123...",
        "Accept": "application/vnd.github.v3+json",
        "User-Agent": "my-backend/1.0",
    },
) as client:
    # These requests ALL include the auth + accept + user-agent headers
    user = await client.get("/users/octocat")
    repos = await client.get("/users/octocat/repos")
    starred = await client.get("/users/octocat/starred")
```

> "Notice `base_url`. Instead of repeating `https://api.github.com` in every call, set it once. Combined with default headers, your client is now configured for a specific external API. Think of this as setting up the dedicated phone line to your top supplier."

---

## 3.4 Validating External Data with Pydantic

**This is critical. You learned Pydantic in Week 3 to validate data coming INTO your API. Now use it to validate data coming FROM external APIs.**

> "When YOUR client sends you data, Pydantic catches bad input at the door. But who validates the data coming from an external API? Nobody — unless YOU do. External APIs will return unexpected nulls, missing fields, renamed keys, and silently changed types. Pydantic is your validation layer for data you don't control."

```python
from pydantic import BaseModel, Field
from typing import Optional
import httpx

# ─────────────────────────────────────────
# Define what you EXPECT from the external API
# ─────────────────────────────────────────
class GitHubUser(BaseModel):
    """Model for GitHub's /users/{username} response.
    
    We only model fields WE CARE ABOUT.
    GitHub returns 30+ fields — we don't need them all.
    """
    login: str
    id: int
    name: Optional[str] = None       # Some users have no name set
    bio: Optional[str] = None
    public_repos: int
    followers: int
    created_at: str                   # ISO datetime string

    model_config = {"extra": "ignore"}  # Ignore fields we didn't define


class GitHubRepo(BaseModel):
    """Model for items in GitHub's repository list."""
    id: int
    name: str
    full_name: str
    description: Optional[str] = None
    stargazers_count: int = Field(ge=0)
    language: Optional[str] = None
    fork: bool


# ─────────────────────────────────────────
# Use models to validate external responses
# ─────────────────────────────────────────
async def get_github_user(
    client: httpx.AsyncClient,
    username: str,
) -> GitHubUser:
    """Fetch and VALIDATE a GitHub user profile."""
    response = await client.get(f"/users/{username}")
    response.raise_for_status()

    # raw_data is an untyped dict — anything could be in here
    raw_data: dict = response.json()

    # Pydantic validates, coerces types, and strips unknown fields
    # If GitHub changes their API and breaks our assumptions,
    # this raises ValidationError — we KNOW immediately
    return GitHubUser.model_validate(raw_data)


async def get_user_repos(
    client: httpx.AsyncClient,
    username: str,
) -> list[GitHubRepo]:
    """Fetch and validate a user's repositories."""
    response = await client.get(
        f"/users/{username}/repos",
        params={"sort": "updated", "per_page": 5},
    )
    response.raise_for_status()

    raw_list: list[dict] = response.json()
    return [GitHubRepo.model_validate(item) for item in raw_list]
```

**Why this matters:**

```
┌─────────────────────────────────────────────────────────────────┐
│           WITHOUT PYDANTIC  vs  WITH PYDANTIC                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WITHOUT:                                                       │
│  data = response.json()                                         │
│  name = data["name"]          ← KeyError if field removed!     │
│  repos = data["public_repos"] ← What if it's a string now?     │
│  ...crash happens 10 lines later in your business logic...     │
│  ...good luck debugging that.                                  │
│                                                                 │
│  WITH:                                                          │
│  user = GitHubUser.model_validate(response.json())              │
│  ├─ Missing field?    → ValidationError with EXACT field name   │
│  ├─ Wrong type?       → ValidationError with expected vs got    │
│  ├─ Extra fields?     → Silently ignored (extra="ignore")      │
│  └─ Valid data?       → Typed, validated, IDE-autocomplete      │
│                                                                 │
│  The error tells you WHAT broke and WHERE — not 10 lines later. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Treat every external API response like untrusted user input. Because that's exactly what it is — input from a system you don't control."

---

## 3.5 Complete Example: Calling a Real API

**Tying it all together — a function that does everything right:**

```python
import httpx
from pydantic import BaseModel, ValidationError
from typing import Optional


class WeatherData(BaseModel):
    city: str
    temperature_celsius: float
    description: str
    humidity: int

    model_config = {"extra": "ignore"}


class ExternalAPIError(Exception):
    """Raised when an external API call fails."""
    def __init__(self, service: str, detail: str, status_code: Optional[int] = None):
        self.service = service
        self.detail = detail
        self.status_code = status_code
        super().__init__(f"{service} error: {detail}")


async def fetch_weather(client: httpx.AsyncClient, city: str) -> WeatherData:
    """
    Fetch weather from external API.

    Demonstrates:
    - Query parameters
    - Response status checking
    - JSON parsing
    - Pydantic validation of external data
    - Structured error handling
    """
    try:
        response = await client.get(
            "https://api.openweathermap.org/data/2.5/weather",
            params={
                "q": city,
                "units": "metric",
                "appid": "YOUR_API_KEY",  # From environment in real code!
            },
        )
        response.raise_for_status()

    except httpx.HTTPStatusError as e:
        # Server responded with error status
        raise ExternalAPIError(
            service="OpenWeatherMap",
            detail=f"HTTP {e.response.status_code} for city '{city}'",
            status_code=e.response.status_code,
        )
    except httpx.RequestError as e:
        # Network failure — couldn't reach the server
        raise ExternalAPIError(
            service="OpenWeatherMap",
            detail=f"Network error: {e}",
        )

    # Parse and validate
    raw = response.json()

    try:
        return WeatherData(
            city=raw["name"],
            temperature_celsius=raw["main"]["temp"],
            description=raw["weather"][0]["description"],
            humidity=raw["main"]["humidity"],
        )
    except (KeyError, IndexError) as e:
        raise ExternalAPIError(
            service="OpenWeatherMap",
            detail=f"Unexpected response structure: missing {e}",
        )
    except ValidationError as e:
        raise ExternalAPIError(
            service="OpenWeatherMap",
            detail=f"Response validation failed: {e}",
        )
```

**Map the error handling to the exception hierarchy:**

```
┌─────────────────────────────────────────────────────────────────┐
│            ERROR HANDLING LAYERS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 1: httpx.RequestError                                    │
│  └─ Network failed. Server unreachable.                        │
│     "The supplier's phone is disconnected."                    │
│                                                                 │
│  Layer 2: httpx.HTTPStatusError (via raise_for_status)          │
│  └─ Server responded, but with an error code.                  │
│     "The supplier answered but said NO."                       │
│                                                                 │
│  Layer 3: KeyError / IndexError                                 │
│  └─ Response was 200, but the data structure changed.          │
│     "The supplier sent a box, but it's full of oranges         │
│      instead of tomatoes."                                     │
│                                                                 │
│  Layer 4: Pydantic ValidationError                              │
│  └─ Data exists but violates our expectations.                 │
│     "The supplier sent tomatoes, but they're made of plastic." │
│                                                                 │
│  ALL four layers raise ExternalAPIError with structured info.   │
│  Your FastAPI error handler catches ONE type, returns 502.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 4: TIMEOUTS — NON-NEGOTIABLE

## 4.1 The Hanging Endpoint (Demonstration)

**Run this. Watch it suffer.**

```python
# demo_no_timeout.py — The silent killer
from fastapi import FastAPI
import httpx
import asyncio

app = FastAPI()

# Simulates an external API that takes FOREVER
async def fake_slow_api():
    await asyncio.sleep(120)  # 2 full minutes
    return {"data": "finally"}

@app.get("/dashboard")
async def get_dashboard():
    async with httpx.AsyncClient() as client:
        # What happens if the external API never responds?
        response = await client.get("http://slow-api.example.com/data")
        return response.json()
```

**Now ask the class:**

> "A user hits `/dashboard`. The external API is down — not erroring, just not responding. What happens?"

Answer:

```
┌─────────────────────────────────────────────────────────────────┐
│              WITHOUT TIMEOUTS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Second:  0   5   10  15  20  25  30 ... 60 ... 120 ...        │
│           │   │   │   │   │   │   │      │      │              │
│  User:    [===============LOADING SPINNER==============...???]  │
│  Your API:[==============WAITING ON EXTERNAL API=======...???]  │
│  External:[.......crickets......not responding...............]  │
│                                                                 │
│  Your endpoint is STUCK.                                        │
│  The user gives up after 10 seconds.                            │
│  But your server is STILL waiting. That connection is held.     │
│  That async task is still alive. That memory is still used.     │
│                                                                 │
│  Now imagine 100 users hit /dashboard:                          │
│  → 100 tasks stuck waiting for an API that will never respond   │
│  → Your server's resources are being CONSUMED by doing NOTHING  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The good news:**

> "httpx has a DEFAULT timeout of 5 seconds for all operations. If you use httpx as-is, you're already somewhat protected. But 'somewhat' isn't good enough. You need to understand the timeout types and configure them intentionally."

---

## 4.2 The Four Timeout Types

**There are four distinct phases where a request can stall:**

```
┌─────────────────────────────────────────────────────────────────┐
│             THE FOUR PHASES OF AN HTTP REQUEST                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your Code                    Network              Their Server │
│  ─────────                    ───────              ──────────── │
│      │                                                  │       │
│  ①  │ ─── TCP SYN ───────────────────────────────▶ │  │       │
│      │                                              │  │       │
│  C   │     ...waiting for TCP handshake...          │  │       │
│  O   │                                              │  │       │
│  N   │ ◀── TCP SYN-ACK ─────────────────────────── │  │       │
│  N   │ ─── TCP ACK ──────────────────────────────▶ │  │       │
│  E   │                                                  │       │
│  C   │─────────────── CONNECTION ESTABLISHED ───────────│       │
│  T   │                                                  │       │
│  ②  │ ─── HTTP Request Headers ───────────────────▶    │       │
│      │ ─── HTTP Request Body ─────────────────────▶    │       │
│  W   │                                                  │       │
│  R   │     ...sending data to server...                 │       │
│  I   │                                                  │       │
│  T   │─────────────── REQUEST SENT ─────────────────────│       │
│  E   │                                                  │       │
│  ③  │                                    Server is      │       │
│      │     ...waiting for response...    processing     │       │
│  R   │                                                  │       │
│  E   │ ◀── HTTP Response Headers ───────────────────── │       │
│  A   │ ◀── HTTP Response Body ─────────────────────── │       │
│  D   │                                                  │       │
│      │─────────────── RESPONSE RECEIVED ────────────────│       │
│      │                                                  │       │
│                                                                 │
│  ④ POOL TIMEOUT: Happens BEFORE step ①                         │
│     If all connections in the pool are busy,                    │
│     waiting for one to become available.                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│              THE FOUR TIMEOUT TYPES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONNECT TIMEOUT (Phase ①)                                     │
│  ─────────────────────────                                      │
│  How long to wait for TCP connection to be established.         │
│  Fails when: Server is unreachable, firewall drops SYN,        │
│              DNS resolution hangs.                               │
│  Analogy: Phone ringing, no one picks up.                       │
│  Typical: 5 seconds                                             │
│                                                                 │
│  WRITE TIMEOUT (Phase ②)                                       │
│  ────────────────────────                                       │
│  How long to wait for request data to be sent.                  │
│  Fails when: Very large request body, slow upload speed,        │
│              server stops reading.                               │
│  Analogy: Dictating your order, but the supplier keeps          │
│           saying "hold on, I'm writing that down..."            │
│  Typical: 5 seconds (only matters for large uploads)            │
│                                                                 │
│  READ TIMEOUT (Phase ③)                                        │
│  ───────────────────────                                        │
│  How long to wait for response data to arrive.                  │
│  Fails when: Server is processing forever, response is huge     │
│              and arrives slowly, server hangs mid-response.      │
│  Analogy: Supplier says "let me check the warehouse" and        │
│           you hear nothing for 10 minutes.                      │
│  Typical: 10 seconds (most critical, most variable)             │
│                                                                 │
│  POOL TIMEOUT (Phase ④)                                        │
│  ────────────────────────                                       │
│  How long to wait for a connection from the pool.               │
│  Fails when: All connections are busy with other requests.      │
│  Analogy: All your phone lines to the supplier are in use.      │
│           You wait for one to free up.                           │
│  Typical: 5 seconds                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.3 Configuring httpx Timeouts

```python
import httpx

# ─────────────────────────────────────────
# Simple: one number for everything
# ─────────────────────────────────────────
client = httpx.AsyncClient(timeout=10.0)  # 10 seconds for all types


# ─────────────────────────────────────────
# Precise: configure each type separately
# ─────────────────────────────────────────
timeout = httpx.Timeout(
    connect=5.0,    # 5s to establish connection
    read=15.0,      # 15s to receive response (some APIs are slow)
    write=5.0,      # 5s to send request
    pool=5.0,       # 5s to acquire connection from pool
)
client = httpx.AsyncClient(timeout=timeout)


# ─────────────────────────────────────────
# Per-request override (when one call is known to be slow)
# ─────────────────────────────────────────
async with httpx.AsyncClient(timeout=5.0) as client:
    # Normal call: 5 second timeout
    fast_response = await client.get("https://api.fast.com/data")

    # This specific call hits a slow endpoint, needs more time
    slow_response = await client.get(
        "https://api.slow.com/generate-report",
        timeout=httpx.Timeout(connect=5.0, read=30.0, write=5.0, pool=5.0),
    )


# ─────────────────────────────────────────
# ❌ NEVER: disable timeouts entirely
# ─────────────────────────────────────────
client = httpx.AsyncClient(timeout=None)   # DON'T DO THIS
client = httpx.AsyncClient(timeout=300.0)  # 5 minutes? Basically no timeout.
```

**Handling timeout exceptions:**

```python
async def fetch_with_timeout(client: httpx.AsyncClient, url: str) -> dict:
    try:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()

    except httpx.ConnectTimeout:
        # Couldn't establish connection in time
        raise ExternalAPIError(
            service=url,
            detail="Connection timed out — server may be unreachable",
        )

    except httpx.ReadTimeout:
        # Connected, but response took too long
        raise ExternalAPIError(
            service=url,
            detail="Response timed out — server too slow",
        )

    except httpx.PoolTimeout:
        # All connections busy — YOUR client is overloaded
        raise ExternalAPIError(
            service=url,
            detail="Connection pool exhausted — too many concurrent requests",
        )

    except httpx.TimeoutException:
        # Catch-all for any timeout type
        raise ExternalAPIError(
            service=url,
            detail="Request timed out",
        )
```

---

## 4.4 Cascading Failures (The Domino Effect)

**This is WHY timeouts are non-negotiable. Demonstrate the cascade:**

```
┌─────────────────────────────────────────────────────────────────┐
│              THE CASCADING FAILURE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: External API becomes slow (responding in 30s)          │
│                                                                 │
│  Step 2: Your /dashboard endpoint waits 30s per request         │
│                                                                 │
│  Step 3: Users keep hitting /dashboard while it's slow          │
│          10 users = 10 async tasks stuck waiting                 │
│          100 users = 100 async tasks stuck waiting              │
│                                                                 │
│  Step 4: YOUR server's connection pool fills up                 │
│          (Remember connection pooling from Week 7?)             │
│                                                                 │
│  Step 5: New requests to /dashboard → PoolTimeout               │
│                                                                 │
│  Step 6: Even endpoints that DON'T call the external API        │
│          start failing because server resources are exhausted   │
│                                                                 │
│  Step 7: YOUR entire API is down.                               │
│          Because SOMEONE ELSE'S API got slow.                   │
│                                                                 │
│                                                                 │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐                │
│  │ External │     │ Your API │     │ All Your │                │
│  │ API slow │ ──▶ │ handlers │ ──▶ │ Clients  │                │
│  │ (30s)    │     │ stuck    │     │ timeout  │                │
│  └──────────┘     └──────────┘     └──────────┘                │
│       ONE             YOUR            EVERYONE                  │
│     failure          problem           suffers                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**With timeouts, the cascade is broken:**

```
┌─────────────────────────────────────────────────────────────────┐
│             WITH TIMEOUTS (5 second read timeout)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: External API becomes slow (responding in 30s)          │
│                                                                 │
│  Step 2: Your endpoint waits 5 seconds → ReadTimeout!           │
│                                                                 │
│  Step 3: You catch the timeout, return a helpful error:         │
│          HTTP 502 Bad Gateway: "Weather service unavailable"    │
│                                                                 │
│  Step 4: The async task is FREED after 5 seconds, not 30.       │
│          Resources are released. Other requests flow normally.  │
│                                                                 │
│  Step 5: /dashboard is degraded. But /users, /tasks, /auth     │
│          all work perfectly. Your API is PARTIALLY degraded,    │
│          not COMPLETELY down.                                   │
│                                                                 │
│  DAMAGE: One endpoint returns errors.                           │
│  WITHOUT TIMEOUTS: Entire server goes down.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Timeouts are not about handling errors gracefully. They're about SURVIVAL. A timeout says: 'I'd rather give my user a fast error than a slow one — and I refuse to let one bad supplier burn down my entire restaurant.'"

---

## 4.5 Choosing Timeout Values

```
┌─────────────────────────────────────────────────────────────────┐
│             TIMEOUT VALUE GUIDELINES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONNECT TIMEOUT: 3-5 seconds                                   │
│  ─────────────────────────────                                  │
│  If you can't establish a TCP connection in 5 seconds,          │
│  the server is either down or unreachable.                      │
│  Waiting longer won't help.                                     │
│                                                                 │
│  READ TIMEOUT: Depends on the API                               │
│  ───────────────────────────────                                │
│  Fast lookup APIs (geocoding, user profiles):  5-10 seconds     │
│  Search/query APIs (GitHub search):            10-15 seconds    │
│  Report generation APIs:                       30-60 seconds    │
│  Webhook callbacks you send:                   10 seconds       │
│                                                                 │
│  RULE: Read timeout should be slightly LONGER than the          │
│        external API's expected response time. If they say       │
│        "responses within 2 seconds", use 5 seconds.            │
│        NOT 2 seconds — leave room for variance.                 │
│                                                                 │
│  WRITE TIMEOUT: 5-10 seconds                                    │
│  ────────────────────────────                                   │
│  Only matters when sending large request bodies.                │
│  For JSON payloads (small), 5 seconds is generous.              │
│                                                                 │
│  POOL TIMEOUT: 5-10 seconds                                     │
│  ───────────────────────────                                    │
│  If you hit pool timeouts, the problem isn't the timeout        │
│  value — it's that you need a larger pool or your external      │
│  calls are too slow (fix the read timeout first).               │
│                                                                 │
│                                                                 │
│  STARTER CONFIG (use this until you have real data):            │
│                                                                 │
│  httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 5: RETRY STRATEGIES

## 5.1 Transient vs Permanent Failures

**Not all failures are created equal. Some are worth retrying. Some aren't.**

```
┌─────────────────────────────────────────────────────────────────┐
│          TRANSIENT vs PERMANENT FAILURES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TRANSIENT (temporary — retry WILL help)                        │
│  ────────────────────────────────────────                       │
│  • httpx.ConnectTimeout     Server was briefly overwhelmed     │
│  • httpx.ReadTimeout        Response was slow, might work now  │
│  • HTTP 429 Too Many Req    Rate limited, wait and retry       │
│  • HTTP 500 Internal Error  Server glitched, might recover     │
│  • HTTP 502 Bad Gateway     Upstream hiccup                    │
│  • HTTP 503 Unavailable     Server overloaded, temporary       │
│  • httpx.ConnectError       Network blip, route flap           │
│                                                                 │
│  "Supplier was on the phone with someone else. Call back."     │
│                                                                 │
│                                                                 │
│  PERMANENT (won't change no matter how many times you retry)    │
│  ──────────────────────────────────────────────────────────     │
│  • HTTP 400 Bad Request     Your request is malformed          │
│  • HTTP 401 Unauthorized    Your API key is wrong              │
│  • HTTP 403 Forbidden       You don't have permission          │
│  • HTTP 404 Not Found       Resource doesn't exist             │
│  • HTTP 422 Unprocessable   Your data is invalid               │
│  • JSON decode error        Response isn't JSON                │
│                                                                 │
│  "Supplier says you gave the wrong account number.             │
│   Calling 50 more times won't change the account number."      │
│                                                                 │
│                                                                 │
│  RULE: Only retry TRANSIENT failures.                           │
│        Retrying permanent failures wastes time and resources.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 Naive Retry (And Why It Backfires)

**First instinct: "just try again." Show why this is dangerous.**

```python
# ❌ NAIVE RETRY — looks reasonable, is actually harmful
async def fetch_naive_retry(
    client: httpx.AsyncClient,
    url: str,
    max_retries: int = 3,
) -> dict:
    for attempt in range(max_retries):
        try:
            response = await client.get(url)
            response.raise_for_status()
            return response.json()
        except (httpx.TimeoutException, httpx.HTTPStatusError):
            if attempt < max_retries - 1:
                continue  # Immediately retry!
            raise
```

**What's wrong with this?**

```
┌─────────────────────────────────────────────────────────────────┐
│             WHY NAIVE RETRY IS DANGEROUS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PROBLEM 1: RETRY STORM (The Thundering Herd)                   │
│  ─────────────────────────────────────────────                  │
│                                                                 │
│  Scenario: External API is overloaded (503 errors).             │
│                                                                 │
│  100 users call your API.                                       │
│  All 100 get 503.                                               │
│  All 100 immediately retry.                 → 200 requests     │
│  All 200 get 503.                                               │
│  All 200 immediately retry.                 → 400 requests     │
│  All 400 get 503.                                               │
│                                                                 │
│  The external API was struggling with 100 requests.             │
│  Your "fix" sent 700 requests in a few seconds.                │
│  You didn't help recovery — you PREVENTED it.                  │
│                                                                 │
│                                                                 │
│  PROBLEM 2: RETRYING NON-RETRYABLE ERRORS                       │
│  ─────────────────────────────────────────                      │
│                                                                 │
│  HTTP 401 → Your API key is invalid.                            │
│  Retry 1: Still invalid.                                        │
│  Retry 2: Still invalid.                                        │
│  Retry 3: Still invalid.                                        │
│  Wasted 3 requests. Key didn't magically become valid.          │
│                                                                 │
│                                                                 │
│  PROBLEM 3: NO DELAY = NO TIME TO RECOVER                       │
│  ────────────────────────────────────────                       │
│                                                                 │
│  If the API needs 5 seconds to recover, but you retry           │
│  instantly, you're kicking it while it's down.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.3 Exponential Backoff with Jitter

**The right way: wait longer between each retry, with randomness.**

```
┌─────────────────────────────────────────────────────────────────┐
│             EXPONENTIAL BACKOFF                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Attempt 1: Fails → Wait 1 second                              │
│  Attempt 2: Fails → Wait 2 seconds                             │
│  Attempt 3: Fails → Wait 4 seconds                             │
│  Attempt 4: Fails → Wait 8 seconds                             │
│  Attempt 5: Give up.                                            │
│                                                                 │
│  Formula: wait = min(base * 2^attempt, max_wait)                │
│                                                                 │
│  The wait doubles each time, giving the server exponentially    │
│  more breathing room to recover.                                │
│                                                                 │
│                                                                 │
│  BUT THERE'S A PROBLEM:                                         │
│  ──────────────────────                                         │
│  If 100 clients ALL use the exact same backoff timing:          │
│                                                                 │
│  Time 0: 100 requests → all fail                               │
│  Time 1: 100 retries arrive simultaneously → all fail          │
│  Time 3: 100 retries arrive simultaneously → all fail          │
│                                                                 │
│  They all retry AT THE SAME MOMENT. Still a thundering herd.   │
│                                                                 │
│                                                                 │
│  SOLUTION: ADD JITTER (randomness)                              │
│  ──────────────────────────────────                             │
│  Instead of: wait = 2^attempt                                   │
│  Use:        wait = random(0, 2^attempt)                        │
│                                                                 │
│  Now 100 clients retry at DIFFERENT times. Spread the load.    │
│                                                                 │
│  Time 0.0: 100 requests → all fail                             │
│  Time 0.3: Client A retries                                    │
│  Time 0.7: Client B retries                                    │
│  Time 0.9: Client C retries                                    │
│  ...                                                           │
│  Time 1.8: Client Z retries                                    │
│                                                                 │
│  The server sees a smooth trickle, not a spike.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Manual implementation (understand before using library):**

```python
import asyncio
import random
import httpx

RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504}

async def fetch_with_backoff(
    client: httpx.AsyncClient,
    url: str,
    max_retries: int = 4,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
) -> dict:
    """
    Fetch with exponential backoff and jitter.
    Only retries transient failures.
    """
    last_exception = None

    for attempt in range(max_retries):
        try:
            response = await client.get(url)
            response.raise_for_status()
            return response.json()

        except httpx.HTTPStatusError as e:
            status = e.response.status_code

            # Permanent failure — don't retry
            if status not in RETRYABLE_STATUS_CODES:
                raise

            last_exception = e

            # Check for Retry-After header (the server TELLS you when)
            retry_after = e.response.headers.get("Retry-After")
            if retry_after:
                delay = float(retry_after)
            else:
                # Exponential backoff with full jitter
                delay = min(base_delay * (2 ** attempt), max_delay)
                delay = random.uniform(0, delay)  # Full jitter

            print(f"Attempt {attempt + 1} failed ({status}). "
                  f"Retrying in {delay:.1f}s...")

            await asyncio.sleep(delay)

        except httpx.TimeoutException as e:
            # Timeouts are transient — retry
            last_exception = e
            delay = min(base_delay * (2 ** attempt), max_delay)
            delay = random.uniform(0, delay)

            print(f"Attempt {attempt + 1} timed out. "
                  f"Retrying in {delay:.1f}s...")

            await asyncio.sleep(delay)

        except httpx.RequestError:
            # Connection error, DNS failure, etc. — don't retry these
            raise

    # All retries exhausted
    raise last_exception
```

> "Notice we check for `Retry-After` headers. Some APIs — like GitHub — explicitly tell you how long to wait. ALWAYS respect that header. It's the supplier literally saying 'call me back in 60 seconds.'"

---

## 5.4 The tenacity Library

**Now that you understand the algorithm, use a battle-tested library.**

> "Writing retry logic by hand is error-prone and tedious. The `tenacity` library handles exponential backoff, jitter, conditional retries, logging, and more — all declaratively with a decorator."

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential_jitter,
    retry_if_exception_type,
    retry_if_result,
    before_sleep_log,
    RetryError,
)
import httpx
import logging

logger = logging.getLogger(__name__)


# ─────────────────────────────────────────
# Basic: retry on timeouts, 3 attempts, exponential backoff
# ─────────────────────────────────────────
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=1, max=10),
    retry=retry_if_exception_type(httpx.TimeoutException),
    before_sleep=before_sleep_log(logger, logging.WARNING),
)
async def fetch_reliable(client: httpx.AsyncClient, url: str) -> httpx.Response:
    response = await client.get(url)
    response.raise_for_status()
    return response
```

**Break down each parameter:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TENACITY PARAMETERS EXPLAINED                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  stop=stop_after_attempt(3)                                     │
│  └─ Give up after 3 total attempts (1 original + 2 retries)   │
│                                                                 │
│  wait=wait_exponential_jitter(initial=1, max=10)                │
│  └─ Wait 1s, then ~2s, then ~4s... up to max 10s              │
│     Jitter is built in (randomized automatically)              │
│                                                                 │
│  retry=retry_if_exception_type(httpx.TimeoutException)          │
│  └─ ONLY retry if a timeout occurs                             │
│     All other exceptions propagate immediately                  │
│                                                                 │
│  before_sleep=before_sleep_log(logger, logging.WARNING)         │
│  └─ Log a WARNING before each retry sleep                      │
│     "Retrying in 2.3s after TimeoutException..."               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**More advanced: retry on specific HTTP status codes too:**

```python
def is_retryable_status(response: httpx.Response) -> bool:
    """Return True if the response status code is retryable."""
    return response.status_code in {429, 500, 502, 503, 504}


@retry(
    stop=stop_after_attempt(4),
    wait=wait_exponential_jitter(initial=1, max=30),
    retry=(
        retry_if_exception_type((httpx.TimeoutException, httpx.ConnectError))
        | retry_if_result(is_retryable_status)
    ),
    before_sleep=before_sleep_log(logger, logging.WARNING),
)
async def fetch_robust(client: httpx.AsyncClient, url: str) -> httpx.Response:
    """
    Retries on:
    - Timeout exceptions (network too slow)
    - Connection errors (server unreachable)
    - HTTP 429/5xx responses (server struggling)

    Does NOT retry on:
    - HTTP 400/401/403/404 (our fault, not transient)
    - Pydantic validation errors (data problem, not network)
    """
    response = await client.get(url)
    return response  # Return response — tenacity checks is_retryable_status


# Usage:
async def get_external_data(client: httpx.AsyncClient) -> dict:
    try:
        response = await fetch_robust(client, "https://api.example.com/data")
        response.raise_for_status()  # Raise for any non-retryable errors that snuck through
        return response.json()
    except RetryError as e:
        # All retries exhausted — the last exception is inside
        raise ExternalAPIError(
            service="example",
            detail=f"Failed after retries: {e.last_attempt.exception()}",
        )
```

**tenacity composable pieces (mix and match):**

```
┌─────────────────────────────────────────────────────────────────┐
│              TENACITY BUILDING BLOCKS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STOP CONDITIONS:                                               │
│  ├─ stop_after_attempt(5)           Max 5 tries                │
│  ├─ stop_after_delay(30)            Max 30 seconds total       │
│  └─ stop_after_attempt(5) | stop_after_delay(30)   Either      │
│                                                                 │
│  WAIT STRATEGIES:                                               │
│  ├─ wait_fixed(2)                   Always 2 seconds           │
│  ├─ wait_exponential(min=1, max=30) 1, 2, 4, 8, ... max 30    │
│  ├─ wait_exponential_jitter(...)    Exponential + randomness   │
│  └─ wait_random(min=1, max=5)       Random between 1–5s        │
│                                                                 │
│  RETRY CONDITIONS:                                              │
│  ├─ retry_if_exception_type(...)    Retry specific exceptions  │
│  ├─ retry_if_result(func)           Retry if result matches    │
│  ├─ retry_if_not_result(func)       Retry if result doesn't   │
│  └─ Combine with | (or) and & (and)                           │
│                                                                 │
│  CALLBACKS:                                                     │
│  ├─ before_sleep=before_sleep_log(logger, level)               │
│  ├─ after=after_log(logger, level)                             │
│  └─ Custom: any callable                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 Combining Retries with Timeouts

**Retries and timeouts work TOGETHER. Get the math right.**

```
┌─────────────────────────────────────────────────────────────────┐
│          TIMEOUTS + RETRIES = TOTAL BUDGET                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Question: If your timeout is 10s and you retry 3 times        │
│  with exponential backoff, what's the WORST CASE total wait?   │
│                                                                 │
│  Attempt 1: 10s timeout → fails                                │
│  Wait: ~1s (backoff)                                           │
│  Attempt 2: 10s timeout → fails                                │
│  Wait: ~2s (backoff)                                           │
│  Attempt 3: 10s timeout → fails                                │
│                                                                 │
│  Worst case: 10 + 1 + 10 + 2 + 10 = 33 seconds                │
│                                                                 │
│  YOUR CLIENT is waiting 33 seconds for a response.             │
│  Is that acceptable for your use case?                          │
│                                                                 │
│                                                                 │
│  RULE OF THUMB:                                                 │
│  ──────────────                                                 │
│  Total budget = (timeout × attempts) + sum(backoff delays)      │
│                                                                 │
│  For a user-facing endpoint:                                    │
│  └─ Total budget should be < 15 seconds (users leave at 10s)  │
│  └─ So maybe: 3s timeout × 3 attempts + ~3s backoff = ~12s    │
│                                                                 │
│  For a background job:                                          │
│  └─ Total budget can be minutes                                │
│  └─ Maybe: 30s timeout × 5 attempts + ~60s backoff = ~3.5min  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Production example showing the full picture:**

```python
import httpx
from tenacity import (
    retry,
    stop_after_attempt,
    stop_after_delay,
    wait_exponential_jitter,
    retry_if_exception_type,
    before_sleep_log,
)
import logging

logger = logging.getLogger(__name__)

# Timeout: 5 second connect, 10 second read
CLIENT_TIMEOUT = httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0)


@retry(
    stop=(
        stop_after_attempt(3)        # Max 3 attempts
        | stop_after_delay(25)       # OR max 25 seconds total
    ),
    wait=wait_exponential_jitter(
        initial=1,                   # First wait: ~1 second
        max=8,                       # Max single wait: 8 seconds
    ),
    retry=retry_if_exception_type(
        (httpx.TimeoutException, httpx.ConnectError)
    ),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,                    # Raise the original exception, not RetryError
)
async def fetch_external_data(
    client: httpx.AsyncClient,
    url: str,
    params: dict | None = None,
) -> dict:
    """
    Reliable external API call with timeouts and retries.

    Worst case timing:
    - Attempt 1: up to 10s (read timeout)
    - Wait: ~1s
    - Attempt 2: up to 10s
    - Wait: ~2s
    - Attempt 3: up to 10s
    - Total: ~33s max, but stop_after_delay(25) caps it
    """
    response = await client.get(url, params=params)
    response.raise_for_status()
    return response.json()
```

---

# PART 6: CONNECTION POOLING

## 6.1 The TCP Handshake Tax

**Every HTTP request requires a TCP connection. Connections are expensive.**

```
┌─────────────────────────────────────────────────────────────────┐
│           THE TCP HANDSHAKE (simplified)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Your App                              External Server          │
│  ────────                              ───────────────          │
│      │                                       │                  │
│      │ ── SYN ────────────────────────────▶ │  ┐               │
│      │                                       │  │               │
│      │ ◀── SYN-ACK ─────────────────────── │  │ TCP 3-way     │
│      │                                       │  │ handshake     │
│      │ ── ACK ────────────────────────────▶ │  ┘               │
│      │                                       │                  │
│      │ ── TLS ClientHello ────────────────▶ │  ┐               │
│      │ ◀── TLS ServerHello ─────────────── │  │ TLS handshake │
│      │ ◀── Certificate ────────────────── │  │ (for HTTPS)    │
│      │ ── Key Exchange ───────────────────▶ │  │               │
│      │ ── Finished ───────────────────────▶ │  ┘               │
│      │                                       │                  │
│      │ ── GET /data ─────────────────────▶  │  Actual request  │
│      │ ◀── 200 OK ──────────────────────── │  Actual response  │
│      │                                       │                  │
│                                                                 │
│  TCP handshake:  ~1 round trip (50-200ms)                       │
│  TLS handshake:  ~2 round trips (100-400ms)                     │
│  TOTAL OVERHEAD: 150-600ms BEFORE any data is exchanged         │
│                                                                 │
│  For ONE request, this is fine.                                  │
│  For 100 requests per second, you're wasting 15-60 SECONDS     │
│  of cumulative handshake time. PER SECOND.                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "You learned about database connection pooling in Week 7 — keeping PostgreSQL connections open and reusing them instead of connecting fresh each time. Same principle, HTTP layer. The cost isn't the query, it's the handshake."

---

## 6.2 AsyncClient as Connection Pool

**httpx.AsyncClient IS a connection pool. Using it correctly matters.**

```python
# ─────────────────────────────────────────
# ❌ BAD: New client per request (new TCP+TLS handshake each time)
# ─────────────────────────────────────────
@app.get("/weather/{city}")
async def get_weather(city: str):
    async with httpx.AsyncClient() as client:  # Opens connections
        response = await client.get(f"https://api.weather.com/{city}")
        return response.json()
    # Closes ALL connections. Next request starts from scratch.


# ❌ WORSE: httpx.get() shorthand (same problem, less obvious)
@app.get("/weather/{city}")
async def get_weather(city: str):
    response = await httpx.AsyncClient().get(...)  # Don't do this
    return response.json()


# ✅ GOOD: Shared client, connections are reused
client = httpx.AsyncClient(
    base_url="https://api.weather.com",
    timeout=httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0),
)

@app.get("/weather/{city}")
async def get_weather(city: str):
    response = await client.get(f"/{city}")  # Reuses existing connection!
    return response.json()
```

**Visualize the difference:**

```
┌─────────────────────────────────────────────────────────────────┐
│       NEW CLIENT PER REQUEST vs SHARED CLIENT                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NEW CLIENT EACH TIME:                                          │
│                                                                 │
│  Req 1: [TCP+TLS handshake 200ms][GET 50ms][close]             │
│  Req 2: [TCP+TLS handshake 200ms][GET 50ms][close]             │
│  Req 3: [TCP+TLS handshake 200ms][GET 50ms][close]             │
│                                                                 │
│  Total for 3 requests: 750ms                                    │
│  Overhead: 600ms (80% wasted on handshakes!)                    │
│                                                                 │
│                                                                 │
│  SHARED CLIENT (connection reuse):                              │
│                                                                 │
│  Req 1: [TCP+TLS handshake 200ms][GET 50ms]                    │
│  Req 2: ─────────connection reused──[GET 50ms]                  │
│  Req 3: ─────────connection reused──[GET 50ms]                  │
│                                                                 │
│  Total for 3 requests: 350ms                                    │
│  Overhead: 200ms (one handshake, reused)                        │
│                                                                 │
│  With 100 requests: 250ms vs 25,000ms (100x difference!)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Configure the pool:**

```python
import httpx

# Control pool size with Limits
limits = httpx.Limits(
    max_connections=100,           # Max total open connections
    max_keepalive_connections=20,  # Max idle connections kept open
)

client = httpx.AsyncClient(
    limits=limits,
    timeout=httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0),
)
```

```
┌─────────────────────────────────────────────────────────────────┐
│              CONNECTION POOL PARAMETERS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  max_connections (default: 100)                                 │
│  ──────────────────────────────                                 │
│  Maximum number of connections that can be OPEN simultaneously. │
│  If all 100 are busy and request #101 comes in → PoolTimeout.  │
│  Higher = more concurrent outbound requests.                    │
│  Too high = you might overwhelm the external API.              │
│                                                                 │
│  max_keepalive_connections (default: 20)                        │
│  ───────────────────────────────────────                        │
│  Maximum IDLE connections kept alive in the pool.               │
│  These are ready-to-use — no handshake needed.                 │
│  Too low = frequent re-handshaking.                            │
│  Too high = holding resources for connections you don't need.  │
│                                                                 │
│  STARTER CONFIG:                                                │
│  ────────────────                                               │
│  For one external API: max_connections=20, keepalive=10         │
│  For multiple APIs:    max_connections=100, keepalive=20        │
│  Tune based on load testing (Week 12).                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.3 Client Lifecycle (Context Managers Revisited)

**The AsyncClient holds open connections. It MUST be properly closed.**

> "Remember context managers from Week 1? `async with` ensures cleanup even if exceptions occur. The AsyncClient holds TCP connections, file descriptors, and memory. If you don't close it, you leak resources."

```python
# ─────────────────────────────────────────
# For scripts: async with (auto-cleanup)
# ─────────────────────────────────────────
async def main():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    # Client is closed here. All connections released.


# ─────────────────────────────────────────
# For FastAPI: lifespan events (the correct way)
# ─────────────────────────────────────────
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # STARTUP: Create clients
    app.state.weather_client = httpx.AsyncClient(
        base_url="https://api.weather.com",
        timeout=httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0),
        headers={"X-API-Key": "your-key"},
    )
    app.state.github_client = httpx.AsyncClient(
        base_url="https://api.github.com",
        timeout=httpx.Timeout(connect=5.0, read=15.0, write=5.0, pool=5.0),
        headers={
            "Authorization": "Bearer ghp_...",
            "Accept": "application/vnd.github.v3+json",
        },
    )

    yield  # App runs here

    # SHUTDOWN: Close clients (release connections)
    await app.state.weather_client.aclose()
    await app.state.github_client.aclose()

app = FastAPI(lifespan=lifespan)
```

```
┌─────────────────────────────────────────────────────────────────┐
│              CLIENT LIFECYCLE IN FASTAPI                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Server starts                                                  │
│      │                                                          │
│      ▼                                                          │
│  lifespan: create AsyncClient instances                         │
│  ├─ TCP connection pools initialized                           │
│  └─ Stored in app.state                                        │
│      │                                                          │
│      ▼                                                          │
│  ┌─ Server running ─────────────────────────────────────┐      │
│  │  Request 1 → uses app.state.weather_client → reuse   │      │
│  │  Request 2 → uses app.state.weather_client → reuse   │      │
│  │  Request 3 → uses app.state.github_client  → reuse   │      │
│  │  ...thousands of requests, connections reused...      │      │
│  └──────────────────────────────────────────────────────┘      │
│      │                                                          │
│      ▼                                                          │
│  lifespan: aclose() all clients                                 │
│  ├─ All connections gracefully closed                          │
│  └─ Resources freed                                            │
│      │                                                          │
│      ▼                                                          │
│  Server stopped                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.4 AsyncClient as FastAPI Dependency

**Now wire it into your dependency injection system.**

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends, Request
import httpx

# ─────────────────────────────────────────
# Setup: create clients at startup
# ─────────────────────────────────────────
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.github_client = httpx.AsyncClient(
        base_url="https://api.github.com",
        timeout=httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=5.0),
        headers={
            "Authorization": "Bearer ghp_...",
            "Accept": "application/vnd.github.v3+json",
        },
        limits=httpx.Limits(max_connections=20, max_keepalive_connections=10),
    )
    yield
    await app.state.github_client.aclose()

app = FastAPI(lifespan=lifespan)


# ─────────────────────────────────────────
# Dependency: extract client from app state
# ─────────────────────────────────────────
async def get_github_client(request: Request) -> httpx.AsyncClient:
    return request.app.state.github_client


# ─────────────────────────────────────────
# Use in endpoints: clean, testable, reusable
# ─────────────────────────────────────────
@app.get("/github/{username}")
async def get_github_profile(
    username: str,
    client: httpx.AsyncClient = Depends(get_github_client),
):
    response = await client.get(f"/users/{username}")
    response.raise_for_status()
    return response.json()


@app.get("/github/{username}/repos")
async def get_github_repos(
    username: str,
    client: httpx.AsyncClient = Depends(get_github_client),
):
    response = await client.get(
        f"/users/{username}/repos",
        params={"sort": "updated", "per_page": 10},
    )
    response.raise_for_status()
    return response.json()
```

**Why this pattern works:**

```
┌─────────────────────────────────────────────────────────────────┐
│           WHY Depends(get_github_client)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. CONNECTION REUSE                                            │
│     One client, one pool, all requests share connections.       │
│                                                                 │
│  2. TESTABILITY                                                 │
│     In tests, override the dependency with a mock client:      │
│                                                                 │
│     app.dependency_overrides[get_github_client] = mock_client   │
│                                                                 │
│     You already know this pattern from Week 4 testing!          │
│                                                                 │
│  3. CONFIGURATION IN ONE PLACE                                  │
│     Timeout, headers, base URL — all configured once.          │
│     Endpoints just use the client.                             │
│                                                                 │
│  4. CLEAN SEPARATION                                            │
│     Endpoints don't know about API keys or base URLs.          │
│     The dependency handles it.                                  │
│     Same pattern as your database session dependency.           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> "Compare this to how you inject your async database session from Week 6. Same pattern: create at startup, inject via dependency, close at shutdown. The tool changes — SQLAlchemy AsyncSession vs httpx AsyncClient — but the architecture is identical."

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              HTTP CLIENT QUICK REFERENCE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INSTALL:                                                       │
│      pip install httpx tenacity                                 │
│                                                                 │
│  BASIC ASYNC REQUEST:                                           │
│      async with httpx.AsyncClient() as client:                  │
│          response = await client.get("https://...")             │
│          data = response.json()                                 │
│                                                                 │
│  WITH PARAMETERS:                                               │
│      await client.get(url, params={"key": "val"})              │
│      await client.post(url, json={"key": "val"})               │
│      await client.put(url, json={"key": "val"})                │
│      await client.patch(url, json={"key": "val"})              │
│      await client.delete(url)                                   │
│                                                                 │
│  RESPONSE HANDLING:                                             │
│      response.status_code      → int                           │
│      response.json()           → dict/list                     │
│      response.text             → str                           │
│      response.headers          → dict-like                     │
│      response.is_success       → bool (2xx)                    │
│      response.raise_for_status()  → raises HTTPStatusError     │
│                                                                 │
│  TIMEOUTS:                                                      │
│      timeout = httpx.Timeout(connect=5, read=10,               │
│                              write=5, pool=5)                   │
│      client = httpx.AsyncClient(timeout=timeout)                │
│                                                                 │
│  CONNECTION POOLING:                                            │
│      limits = httpx.Limits(max_connections=100,                 │
│                            max_keepalive_connections=20)         │
│      client = httpx.AsyncClient(limits=limits)                  │
│                                                                 │
│  EXCEPTIONS TO CATCH:                                           │
│      httpx.TimeoutException    → any timeout                   │
│      httpx.ConnectError        → can't reach server            │
│      httpx.HTTPStatusError     → 4xx/5xx (from raise_for_...)  │
│      httpx.RequestError        → any request failure           │
│                                                                 │
│  RETRY WITH TENACITY:                                           │
│      @retry(                                                    │
│          stop=stop_after_attempt(3),                            │
│          wait=wait_exponential_jitter(initial=1, max=10),       │
│          retry=retry_if_exception_type(httpx.TimeoutException), │
│      )                                                          │
│      async def reliable_fetch(...): ...                         │
│                                                                 │
│  GOLDEN RULES:                                                  │
│      ❌ Never use requests in async code                        │
│      ❌ Never disable timeouts                                  │
│      ❌ Never create AsyncClient per request                    │
│      ❌ Never retry permanent failures (4xx)                    │
│      ✅ Always use Pydantic on external responses               │
│      ✅ Always configure explicit timeouts                      │
│      ✅ Always use exponential backoff with jitter              │
│      ✅ Always share AsyncClient via dependency injection       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Summary: The Key Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  EXTERNAL HTTP = UNTRUSTED, UNRELIABLE, UNPREDICTABLE           │
│                                                                 │
│  Your API depends on external APIs you don't control.           │
│  Defend yourself in layers:                                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                CONNECTION POOLING                       │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │                 TIMEOUTS                         │   │   │
│  │  │  ┌─────────────────────────────────────────┐   │   │   │
│  │  │  │              RETRIES                     │   │   │   │
│  │  │  │  ┌─────────────────────────────────┐   │   │   │   │
│  │  │  │  │      PYDANTIC VALIDATION        │   │   │   │   │
│  │  │  │  │  ┌─────────────────────────┐   │   │   │   │   │
│  │  │  │  │  │    YOUR HTTP REQUEST    │   │   │   │   │   │
│  │  │  │  │  └─────────────────────────┘   │   │   │   │   │
│  │  │  │  └─────────────────────────────────┘   │   │   │   │
│  │  │  └─────────────────────────────────────────┘   │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  THE SUPPLY CHAIN ANALOGY:                                      │
│  ├─ Suppliers (external APIs) are unreliable                   │
│  ├─ Phone calls (HTTP requests) drop and fail                  │
│  ├─ Set a timer (timeout) — don't wait forever                 │
│  ├─ Call back if busy (retry with backoff)                     │
│  ├─ Keep dedicated lines open (connection pool)                │
│  └─ Inspect every delivery (Pydantic validation)              │
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
│  WEEK 8, LECTURE 2: External API Patterns                       │
│  └─ Rate limiting (respecting 429 + X-RateLimit headers)       │
│     Client-side rate limiting (token bucket)                    │
│     Circuit breaker pattern (stop calling dead APIs)            │
│     Webhooks (receiving events instead of polling)              │
│                                                                 │
│  WEEK 8, LECTURE 3: Data Transformation & Integration           │
│  └─ External vs internal models (never leak external schemas)  │
│     Normalizing data from different sources                     │
│     Background data refresh (don't block on every request)     │
│                                                                 │
│  WEEK 8 PROJECT: Third-Party Integration Service                │
│  └─ Integrate with 2-3 real public APIs (GitHub, weather, etc.)│
│     You'll use EVERYTHING from this lecture:                    │
│     httpx clients, timeouts, retries, connection pooling,      │
│     Pydantic validation of external data.                      │
│                                                                 │
│  WEEK 9: Authentication                                         │
│  └─ Bearer tokens you'll set on httpx headers? You built       │
│     those yourself with JWT. Full circle.                      │
│                                                                 │
│  WEEK 10: Redis Caching                                         │
│  └─ Cache external API responses to avoid repeated calls.      │
│     Why call the supplier every time when you can store        │
│     the answer for 5 minutes?                                  │
│                                                                 │
│  WEEK 12: Performance & Load Testing                            │
│  └─ Tune your timeout values and pool sizes with real data.    │
│     Today's values are educated guesses. Week 12 gives you     │
│     the tools to measure and optimize.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```